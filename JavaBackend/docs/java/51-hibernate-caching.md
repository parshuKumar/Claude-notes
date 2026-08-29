# 51 — Hibernate IV — first-level, second-level and query cache; invalidation traps

## Phase: 5 — Spring Boot & Persistence
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: the `orderflow` product catalogue — `GET /products/{id}` and `GET /products?sku=…`, which are 70% of the Topic 65 traffic mix. This topic decides whether that traffic reaches Postgres at all, and what it costs you in correctness when it does not.

---

## Mechanical statement

There are three caches and they are not variations on one idea. They have different
scopes, different keys, different contents and different invalidation rules.

**L1 — the first-level cache — *is* the persistence context.** It is per
`EntityManager`, therefore per transaction. It is not optional and cannot be disabled.
Its key is `(entity class, identifier)` and its value is a live managed object plus the
loaded-state snapshot from Topic 48. It guarantees **object identity**: two lookups of
order 42 in one transaction return the same Java object. When the transaction ends, it
is discarded entirely.

**L2 — the second-level cache — is shared across sessions, per `SessionFactory`.** Its
key is also `(entity class, identifier)`, but its value is **dehydrated state**: an
array of column values, not an object. It is organised into **regions** — one per
entity type by default — and each region has a **concurrency strategy** that defines
what happens when a read and a write overlap. It is correct **only if every writer goes
through Hibernate.** A native `UPDATE`, another service, a Flyway migration, or a DBA
with `psql` all write without telling it, and it will serve the old value forever.

**The query cache is a third thing and behaves differently again.** It does not cache
rows. It caches, for a given query string plus bind parameters, **the list of
identifiers the query returned**, plus a timestamp. On a hit it then resolves every one
of those identifiers — through L2 if the entity is cached there, through the database
if not. It is invalidated by **any** write to **any** table the query touched, tracked
in a separate `UpdateTimestampsCache` region. On a write-heavy table it therefore
misses every time, while still paying the full cost of maintaining itself.

---

## The bridge from what you know

### L1 has a partial analogue. L2 has none.

**L1**: you have met this. Prisma's interactive transactions do not de-duplicate reads,
but TypeORM's `EntityManager` and every unit-of-work ORM has some form of identity map.
More importantly you have written it by hand: the `Map<string, Order>` you keep inside
a request handler so you do not fetch the same row twice. That is L1. **Verdict:
PARTIAL analogue — same idea, but Java's version also holds a snapshot for dirty
checking, which yours did not.**

**L2**: there is no Prisma or TypeORM equivalent that is on by default and transparent.
The closest thing in your world is a Redis cache you wrote yourself:

```ts
const cached = await redis.get(`product:${id}`);
if (cached) return JSON.parse(cached);
const product = await prisma.product.findUnique({ where: { id } });
await redis.set(`product:${id}`, JSON.stringify(product), 'EX', 300);
return product;
```

Compare the properties honestly:

| | Your Redis cache-aside | Hibernate L2 |
|---|---|---|
| Visible at the call site | yes — you wrote the `redis.get` | **no** — `em.find()` looks identical either way |
| You choose what is cached | yes, per call | no, per entity class via an annotation |
| Invalidation | you write it, and you know you wrote it | Hibernate does it — **only for writes it performed** |
| A native `UPDATE` elsewhere | you know your cache is now stale, because you wrote the cache | silently stale, forever, with no signal |
| Serialization format | your JSON | dehydrated column array, provider-specific |
| Cross-process | yes, Redis is shared | **only if you configured a distributed provider** |

**Verdict: NO ANALOGUE.** The mechanism is familiar. The *invisibility* is not, and the
invisibility is the entire risk. A cache you wrote is a cache you remember exists. A
cache that is an annotation on an entity class is a cache the person debugging a stale
price at 2 am does not know is there.

### The thing you will reach for instead is Topic 110, and it is a different thing

Spring's `@Cacheable` over Redis (Topic 110) caches **method return values keyed by
arguments**. Hibernate L2 caches **rows keyed by primary key**. They solve different
problems and they compose badly:

- `@Cacheable` on a method that returns an entity caches a *detached* object graph,
  including whatever lazy proxies it had at cache time — which throw
  `LazyInitializationException` on the next hit.
- `@Cacheable` outside `@Transactional` caches values that may still roll back. That is
  Topic 41's drill.

**Rule of thumb to carry forward:** L2 for rows that are read by id and rarely written.
`@Cacheable`/Redis for computed results and DTOs. Never both on the same data without a
written reason.

### The one thing that genuinely transfers

Every cache correctness question you already know from Redis applies here unchanged:
**stampede on expiry, stale-after-write, unbounded key growth, and "who else writes to
this data".** You are not learning caching. You are learning where Hibernate hides the
answers to those four questions.

---

## What is this?

### L1 — the persistence context

```java
@Transactional(readOnly = true)
public void demonstrateL1() {
    Product a = em.find(Product.class, 42L);   // SELECT
    Product b = em.find(Product.class, 42L);   // no SELECT
    assert a == b;                             // same object, guaranteed
}
```

Two `find` calls, one statement. This is not an optimisation you enabled; it is what a
persistence context **is**. Consequences worth internalising:

- **It is transaction-scoped.** A second HTTP request gets a new `EntityManager` and a
  new, empty L1. L1 gives you nothing across requests.
- **It only helps lookups by identifier.** A JPQL query always goes to the database. It
  then *populates* L1, and — important — if a returned row is already managed in L1,
  Hibernate **returns the existing managed instance and discards the freshly-read
  column values.** Your query results can therefore be "stale" relative to the database
  within one transaction. That is the identity guarantee doing its job, and it
  surprises people.
- **You cannot turn it off.** `em.clear()` empties it; `em.detach(x)` removes one
  entry. There is no "disable L1".
- **It grows.** Every entity you load stays until the transaction ends, with its
  snapshot. This is Topic 53's memory problem.

### L2 — the shared second-level cache

Off by default. Requires three things: a provider on the classpath, configuration, and
per-entity opt-in.

```xml
<!-- Boot manages the versions; do not pin them yourself -->
<dependency>
  <groupId>org.hibernate.orm</groupId>
  <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
  <groupId>org.ehcache</groupId>
  <artifactId>ehcache</artifactId>
  <classifier>jakarta</classifier>
</dependency>
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region:
            factory_class: jcache
        javax:
          cache:
            uri: classpath:ehcache.xml
```

> **Genuine uncertainty, stated plainly:** the JCache configuration property prefix
> moved from `hibernate.javax.cache.*` to `hibernate.jakarta.cache.*` during the
> Jakarta namespace migration, and the Ehcache artifact requires a `jakarta` classifier
> on recent versions. **Do not trust the exact keys above.** Confirm them for your
> version with:
>
> ```bash
> mvn dependency:tree | grep -E 'hibernate-core|hibernate-jcache|ehcache'
> ```
>
> then start the app with `logging.level.org.hibernate.cache=DEBUG` and check the
> region factory actually initialised. If your regions never appear in
> `Statistics.getSecondLevelCacheRegionNames()`, the property name is wrong — that is
> the symptom, and it is silent otherwise.

Opt in per entity:

```java
package com.orderflow.catalog;

import jakarta.persistence.*;
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Cacheable                                                    // jakarta.persistence
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE,           // org.hibernate.annotations
       region = "com.orderflow.catalog.Product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(name = "product_seq", sequenceName = "product_seq", allocationSize = 50)
    private Long id;

    @NaturalId
    private String sku;

    private String name;
    private long priceMinor;          // money as long minor units — Topic 01
}
```

**Both annotations are needed** in the usual configuration: `jakarta.persistence.Cacheable`
is the JPA-standard opt-in, `org.hibernate.annotations.Cache` supplies the concurrency
strategy and region. Which one is consulted depends on
`hibernate.cache.use_minimal_puts` / the shared-cache mode; the safe habit is to write
both.

**What L2 stores is not your object.** It stores a *dehydrated* representation: the
column values, with to-one associations reduced to their foreign key. On a hit,
Hibernate reconstructs a new entity instance and puts it in L1. So an L2 hit still
costs you object construction and an L1 insertion — it saves the round trip, not the
hydration.

**What L2 does not store:** collections. `Product` being cached says nothing about any
`@OneToMany` on it. That is Trap 3, and it is the most common misunderstanding.

### The query cache

```yaml
spring.jpa.properties.hibernate.cache.use_query_cache: true
```

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
@Query("select p from Product p where p.sku = :sku")
Optional<Product> findBySku(@Param("sku") String sku);
```

The cache entry is:

```
key   = (query string, bind parameter values, ...)
value = ([id1, id2, id3, ...], timestampWhenCached)
```

**Identifiers only.** On a hit, Hibernate takes each identifier and resolves it — L1
first, then L2, then the database. So the query cache without entity L2 caching turns
one query into N primary-key lookups. It can make things *slower*.

Invalidation is by **query space** — the set of tables the query touched. Hibernate
maintains an `UpdateTimestampsCache` region mapping table name → timestamp of last
write. On a query-cache lookup it compares the cached result's timestamp against the
timestamps of every space the query touched. Any write to any of those tables since
caching means a miss.

**Read that again in terms of `orderflow`:** a query-cached `select p from Product p
where p.category = ?` is invalidated by *any* insert, update or delete on `product` —
including one to a product in a different category. There is no row-level granularity.

---

## Why does it matter?

**1. A stale price is a financial incident, not a bug report.**
If `orderflow` serves a cached `priceMinor` of 1999 while the database says 2499, you
undercharge every order that lands during the stale window. The window with an
uninvalidated L2 is **unbounded** — until the entry is evicted or the process restarts.
Nobody gets paged. Finance finds it at month-end reconciliation.

**2. L2 is the highest-leverage and highest-risk change available on the read path.**
70% of `orderflow` traffic is catalogue reads, mostly by id, against 100k products with
2% of them hot. That is the textbook L2 case: high read/write ratio, small hot set, key
is the primary key. It can take most of that traffic off Postgres entirely. And it can
also serve wrong data for a week if you get invalidation wrong.

**3. It is where "we enabled caching" stops being a sentence and becomes a design.**
The interview signal is not whether you can name L1, L2 and the query cache. It is
whether you ask "who else writes to this table" before enabling anything.

---

## Machine-level reality

### Where each cache physically lives

| | L1 | L2 (local provider) | L2 (distributed provider) | Query cache |
|---|---|---|---|---|
| Lives in | the `EntityManager` object | JVM heap or off-heap, per process | a remote grid or a Redis-like store | same store as L2 |
| Lifetime | one transaction | the `SessionFactory` / process | the cluster | the `SessionFactory` |
| Cost of a hit | a `HashMap` lookup — nanoseconds | heap: a map lookup + entity reconstruction. off-heap: plus a deserialize | **a network round trip** | a lookup, then N entity resolutions |
| Shared across pods | no | **no** | yes | as per L2 |
| Survives restart | no | no (unless disk-persistent) | yes | no |

The row that catches people is "shared across pods: **no**" for a local provider.
Ehcache heap tier is per-JVM. Run three replicas of `orderflow` and you have three
independent caches. A write on pod 1 invalidates pod 1's entry. Pods 2 and 3 keep
serving the old value until their own entries expire. **This is not a bug in Ehcache.
It is what you asked for.**

### The concurrency strategies, mechanically

This is the part people cargo-cult. Here is what each one actually does when a read and
a write race.

**`READ_ONLY`** — the entry is written once, on load, and never updated. Hibernate
**throws** if you attempt to update a `READ_ONLY` cached entity. Deletion is permitted
(the entry is removed at commit). No locking is needed because there is nothing to
race with.
*Use for:* genuinely immutable reference data — country codes, tax bands, currency
definitions. In `orderflow`: `Currency`, `TaxRate` versions.
*Cost:* zero. This is the only strategy with no correctness caveat.

**`NONSTRICT_READ_WRITE`** — no locking at all. On update, Hibernate **removes** the
cache entry **after** the transaction completes. Between the database commit and the
cache removal there is a window in which a concurrent reader gets the old value. The
window is short but real, and two concurrent writers can leave a stale entry behind.
*Use for:* data where a brief stale read is harmless and concurrent updates to the same
row are genuinely rare.
*Cost:* a stale window you cannot bound precisely. Do not use for prices, balances, or
anything a transaction reads and then acts on.

**`READ_WRITE`** — the one you will use for mutable business data. It implements a
**soft lock**. On update, before the database commit, Hibernate replaces the cache
entry with a soft-lock marker. While that marker is present, every reader **bypasses
the cache and goes to the database**. After commit, the entry is updated (or removed)
and the lock lifted. If the transaction rolls back, the lock is lifted and the entry
removed.
This gives you read-committed semantics through the cache. The cost is that a hot row
under frequent writes spends much of its life soft-locked, so its hit rate collapses
toward zero while you keep paying the maintenance cost.
*Use for:* `Product` in `orderflow` — read constantly, written occasionally.

**`TRANSACTIONAL`** — the cache itself participates in the JTA transaction as an XA
resource. Full transactional isolation between cache and database. Requires a
transactional cache provider (Infinispan in that mode) and a JTA transaction manager.
*Use for:* almost never, in a Spring Boot service using a single `DataSource`. If you
are reaching for this, question the requirement first.

**The decision table:**

| Data | Strategy |
|---|---|
| Immutable reference data | `READ_ONLY` |
| Mutable, brief staleness acceptable, low write concurrency | `NONSTRICT_READ_WRITE` |
| Mutable, correctness matters, this is your default | `READ_WRITE` |
| You have JTA and a genuine requirement | `TRANSACTIONAL` |

### Region granularity

A **region** is a named partition of the cache with its own size limit and eviction
policy. By default each entity class gets its own region named after the fully-
qualified class name, and each cached collection gets a region named
`com.orderflow.orders.Order.lines`.

This matters because **eviction is per region.** Sizing `Product` at 20,000 entries when
you have 100,000 products and a 2% hot set is a deliberate, correct decision — the hot
set fits, the cold tail misses, and you have bounded the memory. Sizing it at 100,000
"to be safe" costs you heap for entries that are read once a month.

```xml
<!-- ehcache.xml — illustration of the shape, not a version-pinned config -->
<cache alias="com.orderflow.catalog.Product">
  <expiry><ttl unit="minutes">30</ttl></expiry>
  <heap unit="entries">20000</heap>
</cache>
```

**A TTL is a correctness backstop, not a policy.** If your invalidation is correct you
do not need one. Set one anyway, sized to the longest stale window you could tolerate
if invalidation silently broke — because Trap 1 says it will.

### Memory: what a cached entity actually costs

An L2 entry for `Product` holds the dehydrated column values plus provider overhead
(key object, entry wrapper, expiry metadata, LRU/LFU linkage). For a small entity that
is easily 3–5× the raw column bytes. 20,000 products at, say, 200 bytes of columns is
not 4 MB; budget several times that and **measure it** with a heap histogram rather
than arithmetic.

An off-heap tier trades that heap for a serialize/deserialize on every access. That is
usually the right trade for a large cache — it keeps the data out of the GC's reach
entirely, which is Topic 71–75's concern — but it is not free and it makes an L2 hit
substantially more expensive than a `HashMap` lookup.

---

## Example 1 — minimal

Prove L1 exists and that it is transaction-scoped. Two methods, one difference.

```java
package com.orderflow.catalog;

import jakarta.persistence.EntityManager;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CacheDemoService {

    private final EntityManager em;

    CacheDemoService(EntityManager em) { this.em = em; }

    /** One statement. Two finds. L1 serves the second. */
    @Transactional(readOnly = true)
    public boolean sameTransaction(Long productId) {
        Product a = em.find(Product.class, productId);
        Product b = em.find(Product.class, productId);
        return a == b;                         // true — identity guarantee
    }

    /** Two statements — unless L2 is on, in which case one. */
    @Transactional(readOnly = true)
    public boolean acrossClear(Long productId) {
        Product a = em.find(Product.class, productId);
        em.clear();                            // L1 emptied; L2 untouched
        Product b = em.find(Product.class, productId);
        return a == b;                         // false — different instances
    }
}
```

Run each with `generate_statistics = true` and read the counters:

| Method | L2 off | L2 on |
|---|---|---|
| `sameTransaction` | 1 statement, 0 cache activity | 1 statement (or 0 if already cached), 1 L2 miss or hit |
| `acrossClear` | 2 statements | 1 statement, then 1 **L2 hit** |

**That single difference — the second `find` after `em.clear()` — is the whole
observable definition of L2.** L1 was emptied; something else answered.

And note the return value: `acrossClear` returns `false` even on an L2 hit, because L2
stores state, not objects. Hibernate built a new `Product` instance from the cached
columns. **An L2 hit does not give you object identity. Only L1 does.**

---

## Example 2 — production scenario (on the project spine)

### The situation

`orderflow` catalogue reads are 70% of the Topic 65 traffic mix. Concretely:

- 100,000 products; measured access is skewed — 2,000 products ("hot") serve ~60% of
  reads.
- `GET /products/{id}` and `GET /products?sku=…` are both primary-key-shaped lookups
  (`sku` is a `@NaturalId`).
- **Writes:** roughly 200 price changes per day through the admin API (through
  Hibernate), **plus a nightly repricing job** that a data team runs as a native
  `UPDATE` across ~30,000 rows.
- SLO: catalogue read p99 < 40 ms. Current p99 is 55 ms, dominated by a single indexed
  Postgres lookup plus network.
- `orderflow` runs **3 replicas** behind a load balancer.

That combination — huge read/write ratio, small hot set, keyed by primary key — is
exactly what L2 is for. And three of the details above are landmines.

### Step 1 — cache the entity, with the natural id

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
@NaturalIdCache                                  // caches sku -> id, so lookups by sku hit L2
public class Product {

    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    private Long id;

    @NaturalId
    private String sku;

    private String name;
    private long priceMinor;
}
```

`@NaturalIdCache` is the piece most people miss. Without it, `findBySku` is a query —
it goes to the database every time, even though the resulting `Product` is sitting in
L2. With it, Hibernate caches `sku → id` and then resolves the id through L2. Two
cache lookups, zero statements.

Use the natural-id API so Hibernate actually consults that cache:

```java
@Transactional(readOnly = true)
public Optional<Product> bySku(String sku) {
    return em.unwrap(Session.class)
             .byNaturalId(Product.class)
             .using("sku", sku)
             .loadOptional();
}
```

A Spring Data derived `findBySku` does **not** use the natural-id cache. It generates a
JPQL query. This is a real and non-obvious distinction.

### Step 2 — size the region to the hot set, not the table

```xml
<cache alias="com.orderflow.catalog.Product">
  <expiry><ttl unit="minutes">15</ttl></expiry>
  <heap unit="entries">5000</heap>
</cache>
```

5,000 entries covers the 2,000 hot products with headroom for churn in the warm tail.
The 98,000 cold products miss, go to Postgres, and — because of LRU eviction — do not
push the hot set out. Expected hit rate: comfortably above 60%, which is what the
access skew predicts.

Caching all 100,000 would raise the hit rate by a few points and multiply the memory by
20. **Cache the hot set, not the table.** This is the single most transferable sizing
instinct in the topic.

The 15-minute TTL is not the invalidation mechanism — `READ_WRITE` handles that. It is
the backstop for the case where invalidation silently fails, which is exactly what Step
3 is about.

### Step 3 — deal with the nightly native `UPDATE`, which will otherwise ruin everything

The repricing job does this:

```sql
UPDATE product SET price_minor = price_minor * 105 / 100 WHERE category_id = 7;
```

Hibernate did not perform that write. It has no idea it happened. **Every cached
product in category 7 now serves the old price, until the TTL expires or the pod
restarts.** With no TTL: forever.

Three fixes, in order of preference:

**(a) Make the job go through Hibernate's bulk-update path and declare the spaces.**
A JPQL bulk update *does* invalidate the affected entity regions:

```java
@Modifying(clearAutomatically = true)
@Query("update Product p set p.priceMinor = p.priceMinor * 105 / 100 where p.category.id = :cat")
int reprice(@Param("cat") Long categoryId);
```

If the statement must be native, tell Hibernate which tables it touched:

```java
@Transactional
public int repriceNative(Long categoryId) {
    return em.createNativeQuery(
                 "update product set price_minor = price_minor * 105 / 100 where category_id = ?")
             .unwrap(org.hibernate.query.NativeQuery.class)
             .addSynchronizedEntityClass(Product.class)   // <-- declares the query space
             .setParameter(1, categoryId)
             .executeUpdate();
}
```

`addSynchronizedEntityClass` is the API that makes a native write participate in cache
invalidation. **If you take one API away from this document, take that one.** It is
also the thing to grep for when auditing a codebase that uses both L2 and native SQL.

**(b) Evict explicitly after the job.**

```java
sessionFactory.getCache().evictEntityData(Product.class);   // whole region
sessionFactory.getCache().evictEntityData(Product.class, productId);  // one entry
entityManagerFactory.getCache().evict(Product.class);       // JPA-standard equivalent
```

> The Hibernate `Cache` API method names were reworked in the 5.3 era —
> `evictEntity`/`evictEntityRegion` became `evictEntityData`. **Confirm on your version
> by letting your IDE complete `sessionFactory.getCache().`** rather than trusting this
> list. The JPA-standard `entityManagerFactory.getCache().evict(Class)` is stable
> across versions and is the safer thing to write.

**(c) If the writer is a different process entirely** — a data team's Python job, a DBA,
a replication stream — then explicit eviction is impossible, and you must either:
- accept a bounded stale window and set the TTL to it (a business decision, written
  down); or
- move to a distributed cache with an invalidation channel and publish an invalidation
  event from the writer; or
- **do not cache that entity.**

Option (c)-third is a legitimate engineering answer and you should be willing to give
it. "We don't cache `Product` because we don't control every writer" is a better
outcome than a cache that is right most of the time.

### Step 4 — the three-replica problem

Ehcache heap is per-JVM. With 3 replicas:

- An admin price change on pod 1 goes through Hibernate. Pod 1's L2 entry is updated.
- **Pods 2 and 3 are untouched.** They serve the old price for up to the TTL — 15
  minutes.

Options:

| Option | Stale window | Cost |
|---|---|---|
| Accept it, TTL 15 min | ≤ 15 min, on 2 of 3 pods | free; must be a written business decision |
| Shorten TTL to 60 s | ≤ 60 s | hit rate drops, Postgres load rises |
| Distributed provider (Infinispan/Hazelcast) | ~ms | a network hop on every L2 access; an ops dependency |
| Publish a price-change event; each pod evicts on receipt | ~ms | you now own an invalidation channel (Topic 113 onward) |

For `orderflow` catalogue prices, **60-second TTL plus accept it** is the honest answer
at 3 replicas — and you write down that the price a customer sees may be up to 60
seconds old, and you check that the *order placement* path reads the price from the
database inside the transaction rather than from the cache. That last clause is the one
that turns a cache decision into a correct system: **display can be stale; charging
cannot.**

### What this achieves, stated as a claim you will verify

- Catalogue reads reaching Postgres drop by roughly the hit rate.
- The claim is falsifiable via `getSecondLevelCacheHitCount()` / `MissCount()` and via
  Postgres `pg_stat_statements` `calls` on the product lookup.
- The latency claim belongs to the Topic 65 baseline, not to this document.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — L2 enabled while something writes with native SQL: stale forever

**Wrong:** `@Cache` on `Product`, plus a nightly `UPDATE product SET price_minor = …`
run as a native query or by an external job.

**Exact symptom — and it is the most easily missed set of symptoms in this document:**

- `GET /products/{id}` returns the old price. `psql` shows the new one.
- **Restarting the pod fixes it.** This is the fingerprint. Anything that a restart
  fixes and nothing else does is a process-local cache.
- Different replicas return different prices for the same product, depending on which
  one had the entry.
- No error, no log line, no metric moves. `getSecondLevelCacheHitCount()` goes *up* —
  the cache is working perfectly. It is serving exactly what it was told to store.
- Finance finds it at reconciliation, days later.

**Root cause:** Hibernate invalidates L2 only for writes **it performed**. A native
`UPDATE` bypasses the entity lifecycle entirely; there is no flush, no entity state
transition, no cache eviction. Hibernate cannot know. The database has no callback to
tell it.

**Fix:**

1. `addSynchronizedEntityClass(Product.class)` on every native mutation, so Hibernate
   knows the query space.
2. Explicit `evict` after any bulk operation you control.
3. A TTL as a bounded backstop for what you do not control.
4. An architectural rule, enforced in review: **every writer to a cached table either
   goes through Hibernate or publishes an invalidation.** Write it in the module's
   README next to the `@Cache` annotation.

**And the honest fallback:** if you cannot enumerate the writers to a table, do not
enable L2 on it.

---

### Trap 2 — the query cache on a write-heavy table

**Wrong:**

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
@Query("select o from Order o where o.customerRef = :ref order by o.placedAt desc")
List<Order> findRecentByCustomer(@Param("ref") String ref);
```

**Exact symptom:**

- `getQueryCacheHitCount()` is near zero while `getQueryCachePutCount()` is roughly
  equal to the number of requests. **A hit rate near zero with a put rate near 100% is
  the signature.**
- Latency is slightly *worse* than with the query cache off.
- Memory in the query-cache region grows and then thrashes against eviction.

**Root cause:** the query cache is invalidated by **query space**, which means table.
`orderflow` inserts orders continuously — the order-placement path is 10% of the
traffic mix. Every single insert into `orders` bumps that table's timestamp in the
`UpdateTimestampsCache`, invalidating **every** cached result set that touched
`orders`, regardless of customer.

So each request caches a result, and the next order placed anywhere in the system
invalidates it. You pay the put cost on every request and collect the hit almost never.

**Fix:**

- **Do not query-cache anything on a table with a meaningful write rate.** The query
  cache is for near-static lookup tables: tax bands, shipping zones, feature
  configuration.
- If you want per-customer caching of a computed result, that is `@Cacheable` over
  Redis with an explicit key and an explicit eviction on order placement — Topic 110.
  Different tool, explicit key, explicit invalidation, and you can reason about it.
- Never enable `use_query_cache` globally "to see if it helps". It is opt-in per query
  for a reason; enable it per query, with a measured hit rate justifying each one.

**The rule:** the query cache's hit rate is bounded above by the *quiet time* on the
tables it touches. If that is measured in seconds, the ceiling is zero.

---

### Trap 3 — L2 does not cache collections, and caching them creates a new N+1

**Wrong:**

```java
@Entity
@Cacheable @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private Set<OrderLine> lines;                     // NOT cached
}
```

Assumption: "`Order` is cached, so `order.getLines()` is served from cache."

**Exact symptom:** `getSecondLevelCacheHitCount()` rises nicely, but
`getPrepareStatementCount()` barely moves. The SQL log shows the order lookup gone and
`select … from order_line where order_id = ?` still present, once per order — Topic
50's N+1, now hiding behind a cache hit rate that makes the dashboard look healthy.

**Root cause:** an entity's L2 entry contains its **scalar columns and its to-one
foreign keys**. Collections are separate cache regions and require their own
annotation.

**The "fix" that creates a worse problem:**

```java
@OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)    // now the collection is cached
private Set<OrderLine> lines;
```

**Exact symptom of the fix:** the collection region now has a high hit rate, and the
statement count is *still* high — now it is `select … from order_line where id = ?`,
once per line.

**Root cause of that:** a cached collection stores **the identifiers of its elements**,
exactly like the query cache. On a hit, Hibernate resolves each element id through L1 →
L2 → database. If `OrderLine` itself is not cached, every element is a database
lookup. You have replaced one statement per order with five statements per order.

**The actual fix:** cache the collection **and** the element entity, or cache neither.

```java
@Entity
@Cacheable @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class OrderLine { ... }
```

**And the better answer for `orderflow`:** do not cache `Order` or `OrderLine` at all.
One million orders, five million lines, each read approximately once by its owner and
then never again. Read/write ratio near 1. Hit rate would be near zero and the memory
cost real. **Cache `Product`, which is read constantly. Do not cache `Order`, which is
read once.** The read/write ratio and the reuse rate are the criteria, not "is it
slow".

---

### Trap 4 — `READ_ONLY` on data that turns out to be mutable

**Wrong:**

```java
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class TaxRate { private BigDecimal rate; }
```

Six months later, someone adds an admin screen to edit a tax rate.

**Exact symptom:** an exception at flush time, from the cache layer, complaining that a
read-only cached entity cannot be updated. The stack trace points into Hibernate's
cache code and does not obviously say "your annotation is wrong". The developer's first
instinct is to remove the `@Cache` annotation entirely — losing the caching that was
correct for the other 99.99% of accesses.

**Root cause:** `READ_ONLY` is a promise you made to Hibernate. It is enforced.

**Fix:** change the strategy to `READ_WRITE`. Not remove the cache.

**The design lesson underneath:** `READ_ONLY` says "this data does not change *in this
application's lifetime*". Tax rates change. Country lists change. Almost nothing is
truly immutable, and versioned reference data (a new row per rate change, never an
update) is usually the better model — then `READ_ONLY` is honest, and Topic 52's
optimistic-locking problem disappears too.

**Verify this behaviour yourself** rather than trusting my description of the exception
type: write a two-line test that loads a `READ_ONLY` cached entity and mutates it, and
record the exact exception class and message on your version.

---

### Trap 5 — measuring the cache with the wrong counter

**Wrong:** "The cache is working — the hit count is 400,000."

**Exact symptom:** a hit count that grows monotonically, cited in a PR as evidence, with
no denominator. Meanwhile Postgres load did not change.

**Root cause:** a raw hit count is meaningless. 400,000 hits against 4,000,000 misses is
a 9% hit rate and the cache is costing you more than it saves. The number you need is
`hits / (hits + misses)`, per region, over a bounded window.

Worse: `getSecondLevelCacheHitCount()` is a **global** counter across all regions. A
90% hit rate on a tiny `Currency` region can mask a 3% hit rate on `Product`, which is
the one that matters.

**Fix:** measure per region, as a ratio, after a `clear()`:

```java
Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
CacheRegionStatistics r = s.getCacheRegionStatistics("com.orderflow.catalog.Product");
long hits   = r.getHitCount();
long misses = r.getMissCount();
double rate = hits / (double) (hits + misses);
long inMemory = r.getElementCountInMemory();
```

> `getCacheRegionStatistics(String)` replaced the older
> `getSecondLevelCacheStatistics(String)`. **Confirm which one your version exposes**
> via IDE completion on `Statistics.` — both names have existed, one is deprecated, and
> which is which depends on your Hibernate line.

Then decide with a threshold you set **in advance**: below 50% hit rate on a region,
the cache is not paying for its memory and its correctness risk. Remove it or resize
it. Writing that threshold down before you measure is what stops the measurement from
becoming a justification.

---

## Hands-on proof

No output from me. Settings you apply, readings you take, and how to interpret each
outcome.

### Instrument setup

`application-cachediag.yml`:

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        generate_statistics: true
        cache:
          use_second_level_cache: true
          use_query_cache: false          # opt in per query, never globally
          region.factory_class: jcache
          use_structured_entries: true    # human-readable entries, for debugging only
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.cache: DEBUG
    org.hibernate.stat: DEBUG
```

`use_structured_entries: true` stores cache entries in a readable map form instead of
the packed representation. It costs memory and speed. **Diagnostic profile only** —
it exists so you can dump a region and see what is in it.

### Proof 1 — L2 is actually initialised

Start the app and print the regions:

```java
@Component
class CacheRegionReporter implements ApplicationRunner {
    private final EntityManagerFactory emf;
    CacheRegionReporter(EntityManagerFactory emf) { this.emf = emf; }

    @Override public void run(ApplicationArguments args) {
        Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
        System.out.println("statistics enabled: " + s.isStatisticsEnabled());
        for (String region : s.getSecondLevelCacheRegionNames()) {
            System.out.println("region: " + region);
        }
    }
}
```

| What you see | What it means |
|---|---|
| `statistics enabled: true` and your entity regions listed | L2 is configured and your entities opted in. Proceed. |
| `statistics enabled: true`, **no regions** | The provider did not initialise or no entity is annotated. Check `hibernate-jcache` is on the classpath and the JCache URI property name is right for your version. |
| `statistics enabled: false` | Wrong profile. `generate_statistics` is not on. |
| Startup fails with a region-factory error | Good — a loud failure. Read it; it names the missing provider. |

**This is the check people skip**, and it is why "we enabled L2" is so often false. A
misconfigured L2 fails **silently and completely**: every read just goes to the
database, exactly as before, and every counter reads zero. Zero counters are
indistinguishable from "no traffic".

### Proof 2 — an L2 hit, isolated from L1

```java
@Test
void secondLevelCacheServesTheSecondTransaction() {
    Statistics s = emf.unwrap(SessionFactory.class).getStatistics();

    // Transaction 1 — cold
    s.clear();
    tx(() -> em.find(Product.class, productId));
    long missesAfterFirst = s.getSecondLevelCacheMissCount();
    long putsAfterFirst   = s.getSecondLevelCachePutCount();
    long stmtsAfterFirst  = s.getPrepareStatementCount();

    // Transaction 2 — new EntityManager, therefore empty L1
    s.clear();
    tx(() -> em.find(Product.class, productId));
    System.out.println("tx1: misses=" + missesAfterFirst
                     + " puts=" + putsAfterFirst + " stmts=" + stmtsAfterFirst);
    System.out.println("tx2: hits=" + s.getSecondLevelCacheHitCount()
                     + " stmts=" + s.getPrepareStatementCount());
}
```

The **two separate transactions** are the point. One transaction proves L1, which you
already had. Two prove L2.

| What you see | What it means |
|---|---|
| tx1: 1 miss, 1 put, ≥1 statement. tx2: 1 hit, **0 statements** | L2 working. This is the reading you want. |
| tx2: 1 hit, **1 statement** | Something else in the same transaction hit the database. Read the SQL log to find what — often a lazy association. |
| tx2: 0 hits, 1 miss, 1 statement | The put did not happen or the entry was evicted. Check the region size and TTL, and whether tx1 rolled back. |
| tx1 and tx2 both: 0 hits, 0 misses, 0 puts | L2 is not active for this entity. Back to Proof 1. |

### Proof 3 — see the query cache's two-step

Enable `use_query_cache`, mark one query cacheable, run it twice, and compare:

| Counter | First run | Second run | Interpretation |
|---|---|---|---|
| `getQueryCacheMissCount()` | 1 | 0 | the id list was not cached, then was |
| `getQueryCachePutCount()` | 1 | 0 | |
| `getQueryCacheHitCount()` | 0 | 1 | |
| `getPrepareStatementCount()` | 1 | **N or 0** | **this is the number that teaches you the mechanism** |

If the entity is **not** L2-cached, the second run's statement count is roughly the
number of rows the query returned — the query cache gave you identifiers and Hibernate
went to the database for each one. If the entity **is** L2-cached, it is 0.

**That single comparison is the clearest possible demonstration that the query cache
caches identifiers, not rows.** Run it once and you will never misremember it.

### Proof 4 — confirm from the database side

```sql
SELECT pg_stat_statements_reset();
-- drive N catalogue reads through the load generator
SELECT calls, rows, mean_exec_time, query
FROM pg_stat_statements
WHERE query ILIKE '%from product%'
ORDER BY calls DESC;
```

**How to read it:** `calls` should be approximately `reads × (1 − hitRate)`. If you
drove 10,000 reads and see ~9,500 calls, your hit rate is 5%, whatever the Hibernate
counter says about absolute hits. The database is the honest witness — it counts what
actually arrived, including statements from code paths you forgot about.

---

## Failure drill

**Mandatory.** Master plan, Topic 51: *enable L2 on `Product`, update a row with a
native query, read through the cache, observe the stale price.*

This drill produces the single most valuable habit in this topic: the reflex to ask
"who else writes to this table" before typing `@Cache`.

### Setup

Real Postgres via Testcontainers (Topic 61). H2 is fine for the mechanics here but you
want the same setup you use everywhere else.

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("cachediag")
class StaleCacheDrillTest {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17");

    @Autowired EntityManagerFactory emf;
    @Autowired TransactionTemplate tx;
    @Autowired ProductRepository products;
    @Autowired JdbcTemplate jdbc;

    Long productId;

    @BeforeEach
    void seed() {
        productId = tx.execute(s -> products.save(
                new Product("SKU-4471", "Aluminium Widget", 1999L)).getId());
        emf.getCache().evictAll();
        emf.unwrap(SessionFactory.class).getStatistics().clear();
    }
}
```

`Product` must carry `@Cacheable` and `@Cache(usage = READ_WRITE)`. Verify with Proof 1
before continuing — a drill against a cache that is not on teaches nothing.

### Step 1 — warm the cache

```java
long warm = tx.execute(s -> products.findById(productId).orElseThrow().getPriceMinor());
// expect 1999
```

Record `getSecondLevelCachePutCount()`. **It must be ≥ 1.** If it is 0, the entry was
never cached and everything below is meaningless. Stop and fix that first.

### Step 2 — write behind Hibernate's back

```java
tx.executeWithoutResult(s ->
    jdbc.update("update product set price_minor = ? where id = ?", 2499L, productId));
```

`JdbcTemplate` shares the transaction and the connection with Hibernate but is entirely
outside the entity lifecycle. This is precisely what a data team's repricing job does,
and precisely what a `@Modifying(nativeQuery = true)` repository method does.

Confirm the database really changed:

```java
Long fromDb = jdbc.queryForObject(
        "select price_minor from product where id = ?", Long.class, productId);
System.out.println("database says: " + fromDb);   // expect 2499
```

### Step 3 — read through the cache and observe the lie

```java
long viaCache = tx.execute(s -> products.findById(productId).orElseThrow().getPriceMinor());
System.out.println("application says: " + viaCache);
System.out.println("L2 hits: " + stats.getSecondLevelCacheHitCount());
```

**How to read every possible outcome:**

| What you see | What it means | What to do |
|---|---|---|
| app says **1999**, db says 2499, L2 hits ≥ 1 | **The drill succeeded.** You have reproduced silent staleness. This is the whole point. | continue to step 4 |
| app says 2499 | The cache was bypassed or evicted. Most likely: the entity was still in L1 from the same transaction, the region evicted, or `@Cacheable` is missing. Check the put count from step 1 and that steps 2 and 3 are in **separate** transactions. | fix, re-run |
| app says 1999 but L2 hits = 0 | You read a stale L1 entry, not L2. Steps must be in separate transactions, or call `em.clear()`. | fix, re-run |
| an exception about a read-only entity | you used `READ_ONLY`. Switch to `READ_WRITE`. | fix, re-run |

**Sit with the successful outcome for a moment.** There was no error. No log line. The
cache hit counter went *up* — by its own metrics the cache is performing beautifully.
The application returned a wrong price with complete confidence, and would keep doing so
until the entry expired or the process restarted. **This is the failure mode that gets
past every test suite that does not specifically look for it.**

### Step 4 — fix it three ways and confirm each

**Fix A — declare the query space on the native write:**

```java
tx.executeWithoutResult(s ->
    em.createNativeQuery("update product set price_minor = ? where id = ?")
      .unwrap(org.hibernate.query.NativeQuery.class)
      .addSynchronizedEntityClass(Product.class)
      .setParameter(1, 2499L)
      .setParameter(2, productId)
      .executeUpdate());
```

Re-run step 3. Expect **2499** and an L2 **miss** (the entry was invalidated, so the
read went to the database and re-cached).

**Fix B — explicit eviction:**

```java
emf.getCache().evict(Product.class, productId);
```

Re-run step 3. Expect 2499, one miss, one put.

**Fix C — a bulk JPQL update instead of native SQL:**

```java
@Modifying(clearAutomatically = true)
@Query("update Product p set p.priceMinor = :price where p.id = :id")
int reprice(@Param("price") long price, @Param("id") Long id);
```

Re-run step 3. Expect 2499.

> **Confirm Fix C's behaviour on your version rather than assuming it.** Bulk JPQL
> updates invalidate the affected entity regions, but whether a given Hibernate version
> evicts the specific rows or the whole region has varied. The test above tells you
> which, in ten seconds. If you get 1999 back from Fix C, your version does not
> auto-invalidate for this case and you need Fix A or B.

### Step 5 — the regression barrier

```java
@Test
@DisplayName("a native write to a cached table must not leave the L2 cache stale")
void nativeWriteInvalidatesCache() {
    tx.execute(s -> products.findById(productId).orElseThrow());          // warm

    tx.executeWithoutResult(s -> repriceNative(productId, 2499L));        // the guarded API

    long viaCache = tx.execute(s -> products.findById(productId).orElseThrow().getPriceMinor());
    assertThat(viaCache)
        .as("cache must reflect the native write")
        .isEqualTo(2499L);
}
```

Then go further, because the real risk is a *future* native write that nobody guards.
Write an ArchUnit rule:

```java
@ArchTest
static final ArchRule nativeWritesToCachedTablesAreGuarded =
    noMethods().that().areAnnotatedWith(Modifying.class)
        .should().beAnnotatedWith(Query.class)   // refine: reject nativeQuery = true
        .because("native writes bypass L2 invalidation — use JPQL or declare query spaces");
```

That rule needs sharpening for your codebase, and the point is the *intent*: the
barrier must catch code that does not exist yet.

### What the fix proves

It proves a **boundary condition**: the set of writers to a cached table is now closed,
and every member of that set participates in invalidation. That is a statement about
the system's architecture, and it is the only kind of statement that makes an L2 cache
safe. A hit-rate number tells you the cache is fast. This test tells you it is correct,
and correctness is the part that has no other witness.

---

## Measurement

### Counters

```java
Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
```

| Counter | Use it to |
|---|---|
| `getSecondLevelCacheHitCount()` / `MissCount()` | compute a hit **ratio** — never quote the raw hit count |
| `getSecondLevelCachePutCount()` | confirm entries are actually being stored; 0 puts means L2 is not active |
| `getCacheRegionStatistics(region)` | **per-region** hits/misses/`getElementCountInMemory()` — the only useful granularity |
| `getSecondLevelCacheRegionNames()` | confirm which entities are cached at all |
| `getQueryCacheHitCount()` / `MissCount()` / `PutCount()` | high puts + near-zero hits = Trap 2 |
| `getUpdateTimestampsCacheHitCount()` / `MissCount()` | the query cache's invalidation bookkeeping |
| `getPrepareStatementCount()` | **the cross-check.** A high L2 hit rate with an unchanged statement count means you cached the wrong thing (Trap 3). |
| `getEntityLoadCount()` | rises on cache misses; compare against statement count to spot collection-element resolution |

**The two-counter discipline:** never report a cache result with one counter. Report
`(hit ratio, statement count)` together. Hit ratio without statement count is how Trap
3 survives a review.

### Database-side

```sql
SELECT pg_stat_statements_reset();
-- run the scenario
SELECT calls, rows, mean_exec_time, query FROM pg_stat_statements ORDER BY calls DESC;
```

`calls` on the cached entity's lookup is the ground truth for how much traffic the
cache actually removed. It counts statements from every code path, including the ones
you did not know existed. When Hibernate's counters and `pg_stat_statements` disagree,
Postgres is right.

### Memory

```bash
jcmd <pid> GC.class_histogram | head -40
```

Look for the provider's entry/wrapper classes and your entity's field types. Then take
a heap dump and measure the retained size of the cache manager. **Do not estimate cache
memory arithmetically** — the provider's per-entry overhead dominates for small
entities and you will be wrong by 3–5×. Topics 79 and 80 do this properly.

### Why `System.nanoTime()` around a cached read is meaningless here

Specifically and additionally wrong for caching, on top of the four general reasons:

1. **No warmup, and the cache *is* the warmup.** The first call is a miss by
   construction. Timing "the first call" times the uncached path and tells you nothing
   about steady state.
2. **One sample of a bimodal distribution.** A cached read has two populations — hits
   (microseconds) and misses (milliseconds) — separated by orders of magnitude. The
   *mean* of a bimodal distribution describes neither mode. You need the hit ratio and
   both modes separately, which is exactly what the counters give you and a stopwatch
   does not.
3. **Cold JIT and dead-code elimination**, as always: the JIT may eliminate work whose
   result you discard, and the first hundreds of invocations run interpreted.
4. **The measurement changes the thing measured.** Running the same read in a tight
   loop gives a 100% hit rate that no production access pattern will reproduce. Your
   benchmark reports the best case and calls it the case.

The honest instruments: **counters** for the mechanism, **Topic 77 (JMH)** for anything
below the service boundary, **Topic 65 (the load baseline)** for the endpoint — with a
realistic access distribution including the cold tail, and a cache warmed the way
production warms it. **Topic 118 (Micrometer)** for a continuous production hit-ratio
gauge, which is the only number that will tell you when your access pattern shifts.

---

## Practice exercises

### 1 — easy: separate the three caches by observation

Seed 10 products. With `generate_statistics = true` and `org.hibernate.SQL=DEBUG`,
write four scenarios and record `getPrepareStatementCount()`,
`getSecondLevelCacheHitCount()`, `getSecondLevelCacheMissCount()` and
`getQueryCacheHitCount()` for each:

- **(a)** `em.find(Product, 1)` twice in **one** transaction, L2 **off**
- **(b)** `em.find(Product, 1)` in **two** transactions, L2 **off**
- **(c)** `em.find(Product, 1)` in **two** transactions, L2 **on**
- **(d)** a JPQL `select p from Product p where p.priceMinor > 1000` twice in two
  transactions, L2 **on**, query cache **off**

Then answer: **why does (d) hit the database both times even though every returned
`Product` is in L2?** Your answer must use the words "identifier" and "query". This is
the single most important sentence in the topic, and writing it yourself is worth more
than reading it.

### 2 — medium: combines Topics 13, 17, 41 and 50

**(a)** Cache `Product` with `READ_WRITE`. Load one, mutate `name` in a transaction,
commit. In a second transaction, read it again. Which value comes back, and which
counter proves whether L2 or the database answered?

**(b)** Now give `Product` an `equals`/`hashCode` based on `sku`. Load the same product
in two separate transactions with L2 on, and check `a == b` and `a.equals(b)`. Explain
both results in terms of what L2 stores (Topic 13, Topic 17 — and note that L2 storing
*state* rather than *objects* is exactly what makes it safe to share across threads).

**(c)** Put `@Cacheable` (Spring's, from Topic 110/41) on a service method that returns
a `Product` **entity**, with `@Transactional` on the same method. Roll the transaction
back by throwing. Then call the method again. What comes back? Now reverse the aspect
order with `@Order` and repeat. Write down which ordering is correct and why — this is
Topic 41's drill re-run against a real cache.

**(d)** Enable L2 on `Order` **and** on `Order.lines` but **not** on `OrderLine`. Load
20 orders in a second transaction and record `getPrepareStatementCount()` alongside the
region hit counts. Explain the number using Trap 3, then state the two-line change that
fixes it and the one-line argument for why you should instead cache none of them.

### 3 — hard: production simulation, advancing the spine

Bring the `orderflow` catalogue to a defensible caching configuration.

**Part A — realistic access.** Seed 100,000 products. Build a read generator whose key
distribution is Zipfian: 2,000 products serve ~60% of reads, with a long cold tail.
A uniform generator makes every cache size look equally good and will teach you the
wrong lesson.

**Part B — the sizing curve.** Run at region sizes 500 / 2,000 / 5,000 / 20,000 /
100,000 entries. For each, record the per-region hit ratio, `getPrepareStatementCount()`
for the run, and the retained heap of the cache manager from a heap dump. Plot hit ratio
against memory. **Identify the knee** and state the region size you would ship, with
the sentence you would put in the PR description.

**Part C — the invalidation matrix.** Write a repricing job in four variants:
(1) native `UPDATE` unguarded, (2) native with `addSynchronizedEntityClass`,
(3) bulk JPQL `@Modifying`, (4) entity-by-entity through the persistence context.
For each, measure: statements issued, stale reads observed afterwards, and wall-clock
for repricing 30,000 rows. Build the table. **Variant 4 will be correct and slow;
variant 1 will be fast and wrong.** Write two sentences on which you would ship and
what you would need to be true.

**Part D — the multi-replica reality.** Run two instances of `orderflow` against the
same Postgres with a **local** L2. Update a price through instance 1's admin API. Poll
`GET /products/{id}` on instance 2 and record how long the stale value persists. Then
set the TTL to 60 s and repeat. Write the sentence you would put in the API
documentation about price freshness — the one a customer-facing team would read.

**Part E — the boundary that must not be cached.** Confirm that order placement reads
the price inside its transaction from the database, not from L2. Write the test that
fails if someone later "optimises" it to read through the cache. Explain in two
sentences why the display path and the charging path have different freshness
requirements, and why that difference is a business rule rather than a technical one.

**Part F — argue against yourself.** You now have a working L2. Make the strongest case
for deleting it and using Redis with explicit `@Cacheable` (Topic 110) instead. Then
make the strongest case against that. State which you would actually ship at 3
replicas, and what would change your mind at 30.

---

## Interview questions

### Q1 — "Walk me through Hibernate's caches."

**Mid-level answer:** "There's a first-level cache per session, a second-level cache
shared across sessions that you enable with a provider like Ehcache, and a query cache
for query results."

**Senior answer:** "Three caches with genuinely different semantics.

L1 *is* the persistence context — per `EntityManager`, so per transaction. Not
optional. Keyed by class and id, holding a live managed object plus the dirty-checking
snapshot. It gives object identity within a transaction, which also means a query can
return a row from the database and Hibernate will hand you the already-managed instance
instead, discarding the fresh column values.

L2 is per `SessionFactory`, keyed the same way, but it stores *dehydrated state* — an
array of column values with to-ones as foreign keys, not an object graph. Organised in
regions, one per entity by default, each with a concurrency strategy. The critical
property: it is correct only if every writer goes through Hibernate. A native `UPDATE`
leaves it stale indefinitely with no signal at all.

The query cache is a different thing again. It caches, per query plus parameters, the
*list of identifiers*, then resolves each one through L1, L2 or the database. So the
query cache without entity caching turns one query into N key lookups and can be
slower. And it invalidates by query space — any write to any table the query touched.
On a table with a real write rate, its hit rate ceiling is zero."

**What separates them:** the phrase "dehydrated state", the identity-map consequence for
query results, and — the one that actually decides the interview — "correct only if
every writer goes through Hibernate", volunteered without being asked.

**Follow-up:** *"What does L2 do with a `@OneToMany`?"* Nothing, unless the collection
is separately annotated — and then it caches element **identifiers**, so you need the
element entity cached too or you have built a cache-backed N+1.

---

### Q2 — "We enabled the second-level cache and now we're serving stale prices. What happened?"

**Mid-level answer:** "The cache TTL is too long, or invalidation isn't working. I'd
shorten the TTL."

**Senior answer:** "First question: who else writes to that table? Hibernate invalidates
L2 only for writes it performed. A native `UPDATE`, a `@Modifying(nativeQuery = true)`
method, a Flyway data migration, another service, or a DBA in `psql` all bypass the
entity lifecycle, so there is no eviction and the entry is stale until it expires or
the process restarts.

The fingerprint is diagnostic: **if restarting the pod fixes it and nothing else does,
it's a process-local cache.** A second fingerprint is different replicas returning
different values for the same id.

Fixes, in order: declare the query space on native writes with
`addSynchronizedEntityClass`, or prefer a bulk JPQL update which does invalidate; add
explicit `evict` after any bulk operation you own; set a TTL as a bounded backstop for
what you don't own. And if I cannot enumerate the writers to that table, I wouldn't
cache that entity at all — a slower correct read beats a fast wrong one, especially for
a price.

Shortening the TTL is a mitigation, not a fix. It reduces the size of the wrong window
without closing it, and it silently trades hit rate for correctness without anyone
deciding that consciously."

**What separates them:** leading with "who else writes to this table", giving the
restart fingerprint, and explicitly naming "don't cache it" as an acceptable engineering
outcome.

**Follow-up:** *"You have three replicas with a local cache. Now what?"* They want:
local caches are per-JVM, so a write on one pod leaves the other two stale for the TTL;
options are a shorter TTL, a distributed provider (which adds a network hop to every
access and an ops dependency), or an invalidation event. And the systems answer: the
*display* path may be stale, the *charging* path must read inside the transaction.

---

### Q3 — "When would you use the query cache?"

**Mid-level answer:** "When the same query runs a lot with the same parameters."

**Senior answer:** "Rarely, and only for near-static lookup tables — tax bands, shipping
zones, feature configuration.

Two reasons. First, it caches identifiers, not rows, so it only helps if the entities
themselves are in L2; otherwise a hit costs you N primary-key lookups and you've made
things worse. Second, and this is the killer, invalidation is by query space, meaning
table, with no row-level granularity. Any insert, update or delete on any table the
query touched invalidates every cached result for it. In our order service, orders are
inserted continuously, so a query-cached lookup on `orders` is invalidated within
milliseconds. You pay the put on every request and collect the hit almost never.

The signature is `getQueryCachePutCount()` roughly equal to the request count with
`getQueryCacheHitCount()` near zero — you're paying maintenance for no benefit.

If I want per-customer result caching with real invalidation control, that's Spring
`@Cacheable` over Redis with an explicit key and explicit eviction on order placement.
Different tool, and the key point is that its invalidation is *mine* — I can reason
about it and I can test it."

**What separates them:** the identifiers-not-rows mechanism, the query-space
granularity, the measurable signature, and knowing when to reach for a different tool.

**Follow-up:** *"How would you decide, quantitatively, whether a given query is worth
caching?"* Hit ratio ceiling ≈ the fraction of time the touched tables are quiet
between invalidations. Measure the write rate on those tables first; the caching
decision follows from it.

---

### Q4 — "What's the difference between `READ_WRITE` and `NONSTRICT_READ_WRITE`?"

**Mid-level answer:** "`READ_WRITE` is safer, `NONSTRICT_READ_WRITE` is faster."

**Senior answer:** "Different mechanisms, not different speed settings.

`READ_WRITE` uses a soft lock. Before the database commit, Hibernate replaces the entry
with a lock marker; while that marker is present every reader bypasses the cache and
goes to the database. After commit the entry is updated or removed and the lock is
lifted; on rollback the lock is lifted and the entry removed. That gives you
read-committed semantics through the cache.

`NONSTRICT_READ_WRITE` does no locking. It removes the entry *after* transaction
completion, so between the commit and the removal, concurrent readers get the old
value. The window is short but it exists and you cannot bound it precisely, and two
concurrent writers can leave a stale entry behind entirely.

So `NONSTRICT` is appropriate where a brief stale read is genuinely harmless *and*
concurrent updates to the same row are rare — a product description, a display name.
Never for a price or a balance, and specifically never for anything a transaction reads
and then makes a decision on. `READ_WRITE` is my default for mutable business data.

And `READ_ONLY` for genuinely immutable reference data — it throws if you try to update,
which is a feature, though six months later somebody adds an admin screen and gets that
exception. Versioned reference data — a new row per change rather than an update —
makes `READ_ONLY` honest and solves the locking problem too."

**What separates them:** describing the soft lock as a mechanism, naming the exact stale
window in `NONSTRICT`, and the design observation about versioned reference data.

**Follow-up:** *"Does `READ_WRITE` give you repeatable read across transactions?"* No.
It gives read-committed through the cache. Cross-transaction repeatability is what
`@Version` is for — Topic 52.

---

### Q5 — "Your L2 hit rate is 85% and the database load didn't drop. Explain."

**Mid-level answer:** "Maybe the queries aren't using the cache — the cache only works
for lookups by id."

**Senior answer:** "That's part of it, and the general shape is right: **the hit rate is
measured on the wrong population.** Four candidates, and I'd distinguish them with two
counters together — the per-region hit ratio and `getPrepareStatementCount()`.

One: the hits are in the wrong region. `getSecondLevelCacheHitCount()` is global, so a
99% hit rate on a tiny `Currency` region masks a 3% rate on `Product`. I'd look at
`getCacheRegionStatistics` per region.

Two: I cached the entity but the traffic is queries, not id lookups. A JPQL `where sku
= ?` always goes to the database unless I've used `@NaturalId` with `@NaturalIdCache`
and the natural-id load API — and a Spring Data derived `findBySku` does *not* use that
API, it generates a query.

Three: I cached the parent but the traffic reads a collection. Collections need their
own `@Cache`, and then they cache element identifiers, so the element entity needs
caching too — otherwise the hit rate looks great and the statement count is unchanged.

Four: the load is writes, or something else entirely, and the read path was never the
bottleneck.

The cross-check that settles it in one step is `pg_stat_statements`: `calls` should
have dropped by the hit rate. If it didn't, the cache isn't intercepting the traffic
that matters, whatever Hibernate's counters say."

**What separates them:** enumerating candidates with a distinguishing measurement rather
than guessing one, knowing the global-counter trap, the `@NaturalIdCache`/derived-query
subtlety, and going to the database as the arbiter.

**Follow-up:** *"How would you catch this before shipping?"* Assert statement count, not
hit rate, in a test — Topic 50's technique applied to caching. Hit rate is a
performance metric; statement count is a structural claim.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. L1 stores live objects; L2 stores dehydrated column values. Name two capabilities L1
   has that L2 structurally cannot have, and one capability L2 has that L1 cannot.
   Connect your answer to why L2 is safe to share across threads and L1 is not.

2. The query cache stores identifiers rather than rows. Design the alternative:
   what would break if it cached full rows instead? Consider two transactions reading
   the same row through two different cached queries. Then say whether Hibernate's
   choice is the right one.

3. Query-cache invalidation is per table, not per row. What would row-level
   invalidation require Hibernate to track, and why is that intractable for an
   arbitrary `WHERE` clause? Give an example query where row-level invalidation is
   provably impossible without re-running the query.

4. `READ_WRITE`'s soft lock makes readers bypass the cache during a write. Under a
   workload where a row is written every 100 ms and read 10,000 times per second, what
   is the effective hit rate for that row, and is the cache still worth its memory?
   What would you do instead?

5. Enabling L2 on `Product` in `orderflow` is clearly right. Enabling it on `Order` is
   clearly wrong. Write the general rule that distinguishes them, in one sentence, using
   only properties you could measure before deciding.

6. You have three replicas and a local L2 with a 15-minute TTL. Someone proposes moving
   to a distributed cache "so the data is always consistent". What have they actually
   bought, what have they paid, and which of the two failures — a 15-minute stale price
   or a cache round trip on every read — is worse for `orderflow`? Does your answer
   change if the entity is `TaxRate` instead of `Product`?

7. Your Redis cache-aside code in Node made every cache read visible at the call site.
   Hibernate's L2 makes it invisible. Name one concrete debugging situation made harder
   by the invisibility, and one situation genuinely made better by it. Then say which
   property you would want if you were designing the ORM.

---

## Quick reference card

### Enable

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
        cache:
          use_second_level_cache: true
          use_query_cache: false            # opt in per query only
          region.factory_class: jcache
        # JCache provider config key: prefix moved javax -> jakarta. Verify on your version.
logging.level.org.hibernate.cache: DEBUG
```

### Annotate

```java
@Entity
@Cacheable                                              // jakarta.persistence
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)     // org.hibernate.annotations
@NaturalIdCache                                         // caches naturalId -> id
public class Product {
    @NaturalId private String sku;

    @OneToMany(mappedBy = "product")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // collections need their OWN @Cache
    private Set<Review> reviews;                        // ...and Review must be cached too
}
```

### Strategies

| Strategy | Mechanism | Use for |
|---|---|---|
| `READ_ONLY` | never updated; throws on update | immutable reference data |
| `NONSTRICT_READ_WRITE` | evict after commit; real stale window | rarely-written, staleness harmless |
| `READ_WRITE` | soft lock; readers bypass during write | **default for mutable data** |
| `TRANSACTIONAL` | XA-participating cache | JTA only; almost never |

### Invalidate

```java
emf.getCache().evict(Product.class, id);       // JPA-standard — prefer this
emf.getCache().evict(Product.class);           // whole region
emf.getCache().evictAll();

// native write: declare the query space or it will NOT invalidate
em.createNativeQuery("update product set price_minor = ? where id = ?")
  .unwrap(org.hibernate.query.NativeQuery.class)
  .addSynchronizedEntityClass(Product.class);
```

### Per-request cache control

```java
// JPA-standard hints
em.setProperty("jakarta.persistence.cache.retrieveMode", CacheRetrieveMode.BYPASS);
em.setProperty("jakarta.persistence.cache.storeMode",   CacheStoreMode.REFRESH);

// Hibernate-native
em.unwrap(Session.class).setCacheMode(CacheMode.IGNORE);
```

### Measure

```java
Statistics s = emf.unwrap(SessionFactory.class).getStatistics();
s.getSecondLevelCacheRegionNames();            // is L2 even on?
CacheRegionStatistics r = s.getCacheRegionStatistics("com.orderflow.catalog.Product");
double rate = r.getHitCount() / (double)(r.getHitCount() + r.getMissCount());
s.getPrepareStatementCount();                  // ALWAYS report this alongside the rate
```

### Gotchas checklist

- [ ] Hibernate invalidates only writes **it** performed. Native SQL leaves L2 stale forever.
- [ ] "Restart fixes it" is the fingerprint of a process-local cache serving stale data.
- [ ] A local L2 is per-JVM. N replicas means N independent caches.
- [ ] Collections need their own `@Cache`, and then the element entity needs one too.
- [ ] The query cache stores **identifiers**, then resolves each one.
- [ ] The query cache invalidates by **table**. Write-heavy table ⇒ hit rate ceiling of zero.
- [ ] `findBySku` (derived query) does not use `@NaturalIdCache`. `byNaturalId()` does.
- [ ] Report hit **ratio**, per region, alongside statement count. Never a raw hit count.
- [ ] Cache the hot set, not the table.
- [ ] Display may be stale. Charging reads inside the transaction, from the database.
- [ ] `use_structured_entries` is a debugging setting. Not for production.
- [ ] JCache property prefix and `Cache` API method names moved across versions. Verify.

---

## When would I use this at work?

**1. A "the price on the site is wrong" ticket.**
Your first question is not "which service" — it is "does restarting a pod fix it?" If
yes, you have a process-local cache with broken invalidation, and you are looking for a
native write. That diagnostic takes thirty seconds and skips an afternoon of
speculation. It is the highest-value thing in this document.

**2. Reviewing a PR that adds `@Cache` to an entity.**
The review comment is one question: *"who else writes to this table, and how do they
invalidate?"* If the author cannot list every writer, the annotation does not merge.
That question also usefully surfaces the data team's nightly job, which nobody in the
service repo knew about.

**3. Cutting database load before a traffic event.**
Catalogue reads are 70% of the mix. L2 on `Product` sized to the hot set can remove
most of them. But the work is not the annotation — it is enumerating writers, choosing
the strategy, sizing the region against a Zipfian access curve, deciding the freshness
contract with the product team, and putting a statement-count assertion in CI. That
whole sequence, done in a week, is a senior deliverable. The annotation alone is a
future incident.

---

## Connected topics

**Prerequisites:**
- **13 — equals/hashCode contract**: L2 stores state, not objects, so an L2 hit gives a
  *new* instance. Any code relying on `==` across transactions was already broken.
- **17 — Immutability and safe publication**: why storing dehydrated state rather than
  live objects is what makes L2 safe to share across threads.
- **41 — AOP ordering**: `@Cacheable` outside `@Transactional` caches values that may
  roll back. That drill is directly relevant here.
- **48 — Persistence context**: L1 *is* the persistence context. This topic is the
  cross-transaction extension of that one.
- **49 — Lazy proxies**: what happens when you cache an object graph containing
  uninitialised proxies (nothing good — and it is why `@Cacheable` on entities is a
  trap).
- **50 — N+1**: the statement counter is the cross-check that catches a cache which
  looks healthy and removes no work.

**This unlocks / is used by:**
- **52 — Locking**: `READ_WRITE`'s soft lock and `@Version` solve overlapping problems
  at different layers. Optimistic locking is what gives you cross-transaction
  repeatability, which no cache strategy does.
- **54–55 — `@Transactional` and isolation**: cache visibility is defined relative to
  transaction boundaries; get the boundary wrong and every strategy above is wrong.
- **61 — Testcontainers**: the staleness drill needs real Postgres and a real second
  transaction.
- **65 — Load baseline**: cache benefit is a load-test result, measured with a realistic
  access distribution including the cold tail. A uniform generator will lie to you.
- **79–80 — Heap analysis**: sizing a cache region correctly means measuring retained
  heap, not estimating it.
- **110 — Spring Cache and Redis**: the explicit, application-level alternative, with
  stampede and key-design problems that L2 does not have and invalidation control that
  L2 does not give you.
- **118 — Micrometer**: hit ratio as a continuous production gauge — the only way to
  notice when the access pattern shifts under you.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0,
`jakarta.persistence.*`. Cache provider artifacts, the JCache property prefix, and
several `Statistics` and `Cache` API method names have changed across Hibernate major
versions; every place this document names one, it also gives the command or the
two-line test that confirms it on your version. Prefer the confirmation. Start with
`mvn dependency:tree | grep hibernate` and `logging.level.org.hibernate.cache=DEBUG`.*
