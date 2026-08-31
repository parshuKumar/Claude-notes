# 116 — Idempotent Consumers and Idempotency Keys

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: an `Idempotency-Key` header on `POST /api/orders` and `POST /api/payments`, and a `processed_event` table for the `inventory` and `notifications` consumers. Both are enforced by a **UNIQUE constraint written in the same transaction as the effect**, so the database — not your code — arbitrates the race.

---

## Mechanical statement

Read this twice. Everything else is an elaboration of it.

> **An idempotency key only works if the uniqueness check and the effect are in the
> same atomic unit.**
>
> A `SELECT` followed by an `INSERT` is **two operations**. Between them, another
> request can execute the same `SELECT`, see the same "not seen before", and
> proceed. Both requests pass the check. Both charge the card. The window is
> microseconds and at 200 concurrent requests it is hit constantly.
>
> No amount of application-level cleverness closes it. Not a `ConcurrentHashMap`
> (Topic 92 — that is exactly the drill you already ran). Not a Redis `GET` then
> `SET`. Not `synchronized`, because there are eight pods. Not a distributed lock,
> because a distributed lock is itself a check-then-act with a lease that can
> expire mid-operation.
>
> **A UNIQUE constraint makes the database the arbiter.** Two concurrent `INSERT`s
> of the same key: the database serialises them on the unique index, one commits,
> the other receives SQLSTATE **23505**. There is no window, because the check and
> the write are the same operation, executed once, by the one component that can
> order them.
>
> And the row must be inserted **in the same transaction as the effect**. Insert it
> in a separate transaction and you have a new failure: the key is recorded, the
> effect fails, and the retry is told "already processed" for work that never
> happened.

Two corollaries, stated now because they are the parts people skip:

> **The key must come from the client.** A server-generated key is a different value
> on every attempt, so it never matches, so it does nothing. The client generates
> the key once per logical operation and re-sends the *same* key on every retry.

> **Idempotency is a property of the EFFECT, not of the transport.** Kafka's
> exactly-once covers Kafka (Topic 114). Your Postgres write and your payment call
> are outside it. The sink must be idempotent or nothing is.

---

## The bridge from what you know

You know what idempotency is and why at-least-once delivery requires it. You have
implemented `Idempotency-Key` headers. So this section goes straight to what is
different in Java, Spring and Postgres — and one of these will surprise you.

### Difference 1 — a constraint violation poisons the whole transaction

This is the one that catches everyone coming from Node, and it is the reason
"just catch the duplicate-key error and carry on" does not work.

In Postgres, **any error aborts the current transaction.** After a `23505`, every
subsequent statement on that connection fails with SQLSTATE `25P02`:
*"current transaction is aborted, commands ignored until end of transaction block"*.

So this, which looks entirely reasonable, does not work:

```java
@Transactional
public OrderResponse place(String key, PlaceOrderCommand cmd) {
    try {
        idempotencyKeys.insert(key);
    } catch (DuplicateKeyException e) {
        return loadExistingResponse(key);      // <-- this SELECT will FAIL
    }
    ...
}
```

Two independent things go wrong:

1. **At the database level**, the transaction is aborted. `loadExistingResponse`
   issues a `SELECT` on an aborted transaction and gets `25P02`.
2. **At the Hibernate/Spring level**, worse: a `PersistenceException` during flush
   marks the transaction **rollback-only**. Even if you catch the exception and do
   nothing further, the outer proxy's `commit()` throws
   `UnexpectedRollbackException`. Your catch block ran; your rollback happened
   anyway.

In Node with a driver that does not use a persistence context, catching the
duplicate-key error and continuing often does work, which is precisely why this is a
bridge problem rather than a general one.

**The Java/Postgres answer is `INSERT ... ON CONFLICT DO NOTHING`**, which does not
raise an error at all, so nothing is aborted and nothing is marked rollback-only.
The whole design below is built around that.

### Difference 2 — `@Transactional` boundaries decide correctness here, not just performance

Topic 54 taught you that `@Transactional` only applies through the proxy and that
`REQUIRES_NEW` takes a second connection. In this topic those facts become
correctness facts:

- The idempotency row must be in **the same transaction** as the effect. That means
  the same `EntityManager`, the same connection, the same commit. A
  `REQUIRES_NEW` on the idempotency insert breaks it (Trap 3).
- Which means the idempotency logic **cannot be a filter or an interceptor that
  commits before the controller runs**, which is the natural place to put it and is
  wrong.
- And a self-invoked helper method loses the transaction entirely (Topic 40).

### Difference 3 — `ConcurrentHashMap` is the wrong tool and you already proved it

Topic 92's drill was exactly this race: a check-then-act on a `ConcurrentHashMap`
idempotency cache producing a double wallet charge. The fix there was
`computeIfAbsent` / `putIfAbsent` — an atomic operation.

That fix is correct **within one JVM** and useless here, because `orderflow` runs 8
pods. The lesson generalises: **atomicity must be provided by the component that all
participants share.** With one JVM that is the CHM. With eight pods and one
database, that is the database.

Redis can also be that component (`SET key value NX`), and it is a legitimate choice
for a pure deduplication check. It is the wrong choice when the effect is a Postgres
write, because then the check and the effect are in two different systems and you are
back to two operations with a window between them — plus a new question about what
happens if Redis loses the key after the effect committed.

### Difference 4 — the API case and the consumer case are the same mechanism, different keys

| | API idempotency | Consumer idempotency |
|---|---|---|
| Key source | **The client**, in the `Idempotency-Key` header | The **producer**, as an event id in the message |
| Key scope | `(endpoint, key)` | `(consumer_group, event_id)` |
| Why duplicates happen | Client retried after a timeout; a load balancer or gateway retried (Topic 112) | At-least-once delivery: the outbox relay republished (Topic 115), or a rebalance rolled back offsets (Topic 113) |
| What the caller needs back | **The original response**, byte-identical | Nothing — just do not apply the effect twice |
| Storage | An `idempotency_key` table with the stored response | A `processed_event` table, key only |

The consumer case is strictly simpler because nobody is waiting for a reply. Do not
over-build it.

---

## What is this?

### The contract you are implementing

`Idempotency-Key` on unsafe methods is a widely used convention (Stripe popularised
it; there is an IETF draft standardising the header). The contract:

1. The client generates a key — a UUID is fine — **once per logical operation**.
2. The client sends it on the first attempt and on **every retry of that same
   operation**.
3. The server, for a given `(endpoint, key)`:
   - if unseen: perform the operation, store the key **and** the response, return it;
   - if seen and complete: return the **stored** response, without re-performing
     anything;
   - if seen and still in progress: return `409 Conflict` (or block briefly);
   - if seen with a **different request body**: return `422` — the client has reused
     a key for a different operation, which is a client bug you must not paper over.
4. Keys expire. 24 hours is a common choice; state yours.

Point 3's last clause is the one people omit and it matters: without a body
fingerprint, a client that reuses a key for a different order gets the *first*
order's response and believes the second order succeeded.

### The table

```sql
CREATE TABLE idempotency_key (
    scope                text        NOT NULL,   -- 'POST /api/orders'
    idem_key             text        NOT NULL,   -- the client's key
    request_fingerprint  text        NOT NULL,   -- SHA-256 of the canonical body
    status               text        NOT NULL,   -- 'IN_PROGRESS' | 'COMPLETED'
    response_status      int,
    response_body        jsonb,
    resource_id          text,
    principal            text        NOT NULL,   -- who owns this key
    created_at           timestamptz NOT NULL DEFAULT now(),
    completed_at         timestamptz,
    expires_at           timestamptz NOT NULL,

    CONSTRAINT idempotency_key_pk PRIMARY KEY (scope, idem_key)
);

CREATE INDEX idempotency_key_expiry_idx ON idempotency_key (expires_at);
```

Design notes, each of which is a decision:

- **`PRIMARY KEY (scope, idem_key)`** — the primary key *is* the constraint. A
  primary key is a unique index, so there is no separate index to maintain. Scoping
  by endpoint means the same key on `POST /api/orders` and `POST /api/payments` are
  different keys, which is what a client expects.
- **`principal`** — store who the key belongs to and **check it**. Without this,
  client A can guess client B's key and read B's stored response. It is a genuine
  information-disclosure hole and it is invisible in testing.
- **`request_fingerprint`** — a hash of the canonicalised request body. Enables the
  `422` in point 3.
- **`expires_at`** — this table grows at the rate of your write traffic forever
  unless something deletes from it. See the retention discussion in Example 2.

And for consumers, something much smaller:

```sql
CREATE TABLE processed_event (
    consumer_group text        NOT NULL,
    event_id       text        NOT NULL,
    processed_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (consumer_group, event_id)
);
```

**`consumer_group` is in the key and that is not optional.** `orderflow` has two
groups reading the same topic. Keying on `event_id` alone means `inventory`
processing an event marks it processed for `notifications` too, and the customer
never gets an email. This is a real bug with a silent signature.

### The three-way outcome

Every idempotent operation has three outcomes, and your code must produce all three:

| Outcome | Meaning | Response |
|---|---|---|
| **First** | Key inserted, effect applied | The real response, `201`/`200` |
| **Replay** | Key exists, `status = COMPLETED` | The **stored** response, byte-identical, `+ Idempotency-Replayed: true` |
| **In flight** | Key exists, `status = IN_PROGRESS` | `409 Conflict` with `Retry-After` |

The in-flight case is the one that is usually missing, and it is exactly what a
retry storm (Topic 111) produces: the client's second attempt arrives while the
first is still running.

---

## Why does it matter?

**1. Because the failure is money.** A double charge on `POST /api/payments` is a
customer contacting support, a refund, and a chargeback. It is one of very few bugs
in this curriculum with a direct, itemisable financial cost.

**2. Because everything upstream of it is at-least-once, by design.** Topic 115's
outbox relay republishes after a crash. Topic 113's rebalance rolls offsets back.
Topic 111's retry re-sends after a timeout. Topic 112's gateway may retry. Every one
of those is a correct design decision that *produces duplicates on purpose*, having
traded silent loss for visible duplication. **The idempotent sink is the other half
of that trade.** Without it you did not make the system safer; you moved the bug.

**3. Because the naive implementation passes every test you would think to write.**
`SELECT` then `INSERT` is correct under any single-threaded test, any
`@SpringBootTest` that calls the method twice in sequence, and any manual `curl`.
It fails only under genuine concurrency, which means it fails in production and
nowhere else.

**4. Because this is Topic 92's race with a bigger blast radius.** You already
proved that check-then-act on a `ConcurrentHashMap` double-charges a wallet. This is
the same race at the API boundary, across 8 pods, where the atomic primitive has to
be the database.

---

## Machine-level reality

### What a UNIQUE constraint actually is

In Postgres, a `UNIQUE` constraint (and a `PRIMARY KEY`) is implemented as a
**unique B-tree index**. There is no separate "constraint checker"; the index *is*
the enforcement.

An `INSERT` into a table with a unique index does, roughly:

1. Write the new tuple into a heap page.
2. For each unique index, attempt to insert the index entry.
3. During that index insert, scan the leaf page for an equal key. If one is found,
   check the visibility of the tuple it points to.
4. If a **committed live** tuple with that key exists → raise `23505`.
5. If the conflicting tuple belongs to an **in-progress** transaction → **block**,
   waiting on that transaction's id.
6. When the other transaction ends: if it committed, raise `23505`; if it rolled
   back, proceed.

**Step 5 is the whole mechanism and it is why this works.** Two concurrent requests
with the same key: the second one *blocks inside the `INSERT`* until the first
commits or rolls back. The database is serialising them. Your application code did
nothing, held no lock, and cannot get it wrong.

Two consequences you must design for:

- **The blocking duration equals the first transaction's remaining lifetime.** If
  request A holds its transaction open for two seconds (because it is calling a
  payment gateway inside it — Topic 55), request B blocks for two seconds holding a
  Hikari connection. **The idempotency mechanism amplifies the cost of a long
  transaction.** Keep the transaction short; this is now a correctness-adjacent
  reason, not just a performance one.
- **If A rolls back, B proceeds and does the work.** This is correct: the operation
  did not happen, so the retry should perform it. It also means a key is not "used
  up" by a failed attempt, which is exactly the property Trap 3 destroys.

### What a violation costs

A failed `INSERT` is not free, and the costs are worth knowing because at scale they
show up.

| Cost | Detail |
|---|---|
| A **dead heap tuple** | The row was written to the heap before the index check failed. It is now dead and must be vacuumed. |
| **WAL** | The heap insert was WAL-logged before the failure. |
| **Index bloat** | Speculative index entries may be left behind for vacuum. |
| **The transaction is ABORTED** | SQLSTATE `25P02` on every subsequent statement. This is the expensive one — it costs you a round trip and a retry of everything. |
| **In Hibernate**, the persistence context is in an undefined state | The `PersistenceException` marks the transaction rollback-only. Recovery within the same transaction is not possible. |

**`INSERT ... ON CONFLICT DO NOTHING` avoids all of the transaction-level costs.**
It still does a speculative insertion (so a small amount of the bloat cost remains),
but it does not raise an error, so nothing is aborted and nothing is marked
rollback-only. `RETURNING` then tells you which branch you are on:

```sql
INSERT INTO idempotency_key (scope, idem_key, request_fingerprint, status, principal, expires_at)
VALUES (:scope, :key, :fingerprint, 'IN_PROGRESS', :principal, now() + interval '24 hours')
ON CONFLICT (scope, idem_key) DO NOTHING
RETURNING idem_key;
```

- **One row returned** → you won the race. Proceed with the effect.
- **Zero rows returned** → someone else has the key. Read their row and branch on
  its `status`. **The transaction is still healthy**, so you can.

This is the single most important line of SQL in the topic.

### Where each piece of state lives

| State | Where | Who arbitrates |
|---|---|---|
| The idempotency key row | Postgres, unique B-tree index | **Postgres**, by blocking on the index |
| The effect (order, payment) | Postgres, same transaction | The same commit |
| Consumer offsets | Kafka `__consumer_offsets` | The group coordinator — **not** in your transaction (Topic 114) |
| The client's key | The client's memory / retry state | The client. If it loses it, idempotency is gone. |

Row three is why consumer idempotency is a database concern and not a Kafka one. The
offset commit and your Postgres write are two systems; there is no shared
transaction. So the guarantee has to be at the sink.

### The SQLSTATE and its Spring translation

| Layer | Value |
|---|---|
| Postgres SQLSTATE | `23505` — `unique_violation` |
| JDBC | `SQLIntegrityConstraintViolationException`, `getSQLState() == "23505"` |
| Spring translation | `DuplicateKeyException`, a subclass of `DataIntegrityViolationException` |
| Postgres SQLSTATE after the abort | `25P02` — `in_failed_sql_transaction` |

Spring's `SQLExceptionTranslator` performs that mapping, which is why the Spring
data-access hierarchy is unchecked and portable (Topic 09). But note: **catching
`DuplicateKeyException` tells you a duplicate happened; it does not give you back a
usable transaction.** Knowing the SQLSTATE is for diagnosis and metrics, not for
control flow. Control flow uses `ON CONFLICT`.

---

## Example 1 — minimal

The smallest correct thing: one endpoint, one table, one `ON CONFLICT`.

```sql
CREATE TABLE lab_idempotency (
    idem_key    text        PRIMARY KEY,
    status      text        NOT NULL,
    resource_id text,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

```java
@Repository
public class IdempotencyRepository {

    private final JdbcClient jdbc;   // Spring's JdbcClient; JdbcTemplate works too

    /** @return true if WE claimed the key; false if someone else already holds it. */
    public boolean tryClaim(String key) {
        return jdbc.sql("""
                INSERT INTO lab_idempotency (idem_key, status)
                VALUES (:key, 'IN_PROGRESS')
                ON CONFLICT (idem_key) DO NOTHING
                RETURNING idem_key
                """)
            .param("key", key)
            .query(String.class)
            .optional()
            .isPresent();          // one row = we won; zero rows = someone else has it
    }

    public Optional<LabRecord> find(String key) {
        return jdbc.sql("SELECT idem_key, status, resource_id FROM lab_idempotency WHERE idem_key = :key")
                   .param("key", key)
                   .query(LabRecord.class)
                   .optional();
    }

    public void complete(String key, String resourceId) {
        jdbc.sql("UPDATE lab_idempotency SET status='COMPLETED', resource_id=:id WHERE idem_key=:key")
            .param("id", resourceId).param("key", key)
            .update();
    }
}
```

```java
@Service
public class LabOrderService {

    private final IdempotencyRepository keys;
    private final OrderRepository orders;

    /**
     * ONE transaction. The key claim, the effect, and the completion all commit
     * together or roll back together. That is the entire design.
     */
    @Transactional
    public LabResult place(String idempotencyKey, PlaceOrderCommand cmd) {

        if (!keys.tryClaim(idempotencyKey)) {
            // Someone else holds the key. Read their row -- the transaction is
            // still HEALTHY because ON CONFLICT did not raise an error.
            var existing = keys.find(idempotencyKey).orElseThrow();
            return switch (existing.status()) {
                case "COMPLETED"   -> LabResult.replay(existing.resourceId());
                case "IN_PROGRESS" -> LabResult.inFlight();
                default -> throw new IllegalStateException("unknown status " + existing.status());
            };
        }

        // We won the race. Do the work.
        var order = orders.save(Order.from(cmd));
        keys.complete(idempotencyKey, order.id().value());
        return LabResult.created(order.id().value());
    }
}
```

```java
@RestController
class LabOrderController {

    @PostMapping("/lab/orders")
    ResponseEntity<?> place(@RequestHeader("Idempotency-Key") String key,
                            @RequestBody @Valid PlaceOrderCommand cmd) {
        var result = service.place(key, cmd);
        return switch (result.kind()) {
            case CREATED  -> ResponseEntity.status(201).body(Map.of("orderId", result.resourceId()));
            case REPLAY   -> ResponseEntity.status(200)
                                 .header("Idempotency-Replayed", "true")
                                 .body(Map.of("orderId", result.resourceId()));
            case IN_FLIGHT-> ResponseEntity.status(409).header("Retry-After", "1")
                                 .body(Map.of("detail", "A request with this key is in progress"));
        };
    }
}
```

### Run it

```bash
KEY=$(uuidgen)

# 1. First call.
curl -i -X POST localhost:8080/lab/orders \
  -H "Idempotency-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"productSku":"SKU-1001","quantity":1}'

# 2. Same key again -- a client retry.
curl -i -X POST localhost:8080/lab/orders \
  -H "Idempotency-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"productSku":"SKU-1001","quantity":1}'

# 3. 50 concurrent requests with the SAME key. This is the real test.
seq 50 | xargs -P 50 -I{} curl -s -o /dev/null -w '%{http_code}\n' \
  -X POST localhost:8080/lab/orders \
  -H "Idempotency-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"productSku":"SKU-1001","quantity":1}' | sort | uniq -c

# 4. Count what was actually created.
psql -c "SELECT count(*) FROM orders WHERE sku='SKU-1001';"
psql -c "SELECT idem_key, status, resource_id FROM lab_idempotency;"
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Step 1 returns `201`; step 2 returns `200` with `Idempotency-Replayed: true` | The replay path works for a sequential retry. This is the easy half. |
| Step 3's tally is one `201` and forty-nine `200`/`409` | **Correct.** Exactly one request did the work; the database serialised the rest. |
| Step 3's tally shows more than one `201` | **The race is live.** Your claim is not atomic — check that you used `ON CONFLICT` and not a `SELECT` first. |
| Step 4 shows exactly one order row | The effect happened once. This is the assertion that matters. |
| Step 4 shows several order rows | Same as above: the check-then-act race. |
| Some `409`s in step 3 | Expected under concurrency — those arrived while the winner's transaction was still open and the winner had already committed the claim... which cannot happen in one transaction. If you see many `409`s here, your claim is in a *separate* transaction. That is Trap 3. |

That last row is subtle and worth pausing on. With a single transaction, a losing
request **blocks** inside the `INSERT` until the winner commits, then reads
`COMPLETED` and returns `200`. You should see very few `409`s. Lots of `409`s means
the claim committed separately from the effect.

---

## Example 2 — production scenario (on the project spine)

### The constraints

- Topic 65 baseline: 70/20/10 mix; **10% of arrival rate is `POST /api/orders`**, and
  each order produces one payment. So the idempotency table receives writes at the
  order-placement rate, twice (orders and payments).
- 8 pods. `maximum-pool-size: 16` each (Topic 109).
- Postgres with 1M orders, 5M order lines already.
- The payment provider is external and slow enough to matter (Topic 111).
- The outbox relay is at-least-once (Topic 115).
- `orderflow.order-placed` has 12 partitions; two consumer groups (Topic 113).

### The API side — `POST /api/orders`

```sql
CREATE TABLE idempotency_key (
    scope               text        NOT NULL,
    idem_key            text        NOT NULL,
    request_fingerprint text        NOT NULL,
    principal           text        NOT NULL,
    status              text        NOT NULL,
    response_status     int,
    response_body       jsonb,
    resource_id         text,
    created_at          timestamptz NOT NULL DEFAULT now(),
    completed_at        timestamptz,
    expires_at          timestamptz NOT NULL,
    CONSTRAINT idempotency_key_pk PRIMARY KEY (scope, idem_key)
) PARTITION BY RANGE (created_at);

-- Daily partitions. DROP TABLE reclaims space instantly with no vacuum --
-- the same reasoning as the outbox table in Topic 115.
CREATE TABLE idempotency_key_2026_08_31 PARTITION OF idempotency_key
    FOR VALUES FROM ('2026-08-31') TO ('2026-09-01');
```

Partitioning by day is not premature here. At the baseline order rate this table
receives one row per order placement, forever. A `DELETE ... WHERE expires_at < now()`
job creates dead tuples that autovacuum must reclaim, on a table that is being
inserted into continuously — which is exactly the workload autovacuum handles worst.
`DROP TABLE` on a day partition is instant and generates no garbage.

```java
public record IdempotencyClaim(boolean won, IdempotencyRecord existing) { }

@Repository
public class IdempotencyKeyRepository {

    private final JdbcClient jdbc;

    public IdempotencyClaim claim(String scope, String key, String fingerprint,
                                  String principal, Duration ttl) {
        Optional<String> inserted = jdbc.sql("""
                INSERT INTO idempotency_key
                    (scope, idem_key, request_fingerprint, principal, status, expires_at)
                VALUES (:scope, :key, :fp, :principal, 'IN_PROGRESS', now() + :ttl)
                ON CONFLICT (scope, idem_key) DO NOTHING
                RETURNING idem_key
                """)
            .param("scope", scope).param("key", key).param("fp", fingerprint)
            .param("principal", principal)
            .param("ttl", ttl)
            .query(String.class).optional();

        if (inserted.isPresent()) return new IdempotencyClaim(true, null);

        var existing = jdbc.sql("""
                SELECT scope, idem_key, request_fingerprint, principal, status,
                       response_status, response_body, resource_id
                FROM idempotency_key WHERE scope = :scope AND idem_key = :key
                """)
            .param("scope", scope).param("key", key)
            .query(IdempotencyRecord.class).single();

        return new IdempotencyClaim(false, existing);
    }

    public void complete(String scope, String key, int status, String body, String resourceId) {
        jdbc.sql("""
                UPDATE idempotency_key
                   SET status='COMPLETED', response_status=:st,
                       response_body=CAST(:body AS jsonb), resource_id=:rid,
                       completed_at=now()
                 WHERE scope=:scope AND idem_key=:key
                """)
            .param("st", status).param("body", body).param("rid", resourceId)
            .param("scope", scope).param("key", key)
            .update();
    }
}
```

```java
@Service
public class OrderPlacementService {

    private static final String SCOPE = "POST /api/orders";
    private static final Duration TTL = Duration.ofHours(24);

    /**
     * ONE transaction wraps: the claim, the order insert, the inventory
     * reservation, the outbox row (Topic 115) and the completion.
     *
     * The payment gateway call is deliberately NOT here -- Topic 55/111.
     * It happens after this transaction commits, in the saga (Topic 117).
     */
    @Transactional
    public PlacementOutcome place(String idemKey, String principal, PlaceOrderCommand cmd) {

        String fingerprint = fingerprint(cmd);
        var claim = keys.claim(SCOPE, idemKey, fingerprint, principal, TTL);

        if (!claim.won()) {
            var existing = claim.existing();

            // SECURITY: a key belongs to the principal that created it.
            if (!existing.principal().equals(principal)) {
                throw new IdempotencyKeyOwnershipException();     // -> 403
            }
            // A key reused for a DIFFERENT body is a client bug. Do not serve
            // the first response for the second request.
            if (!existing.requestFingerprint().equals(fingerprint)) {
                throw new IdempotencyKeyReusedException();        // -> 422
            }
            return switch (existing.status()) {
                case "COMPLETED"    -> PlacementOutcome.replay(existing.responseStatus(),
                                                               existing.responseBody());
                case "IN_PROGRESS"  -> PlacementOutcome.inFlight();
                default -> throw new IllegalStateException(existing.status());
            };
        }

        // We hold the key. Everything below is in the SAME transaction.
        var order = orders.save(Order.pending(cmd, principal));
        inventory.reserve(cmd.lines());                       // Topic 52
        outbox.append(OrderPlacedEvent.from(order));          // Topic 115

        var body = json.write(new OrderCreatedResponse(order.id(), order.status()));
        keys.complete(SCOPE, idemKey, 201, body, order.id().value());

        return PlacementOutcome.created(201, body, order.id());
    }

    /** Canonical, stable, order-independent. A raw string hash of the JSON is NOT stable. */
    private String fingerprint(PlaceOrderCommand cmd) {
        var canonical = cmd.lines().stream()
                .sorted(Comparator.comparing(OrderLineCommand::sku))
                .map(l -> l.sku() + ":" + l.quantity())
                .collect(Collectors.joining("|"));
        return HexFormat.of().formatHex(
                MessageDigest.getInstance("SHA-256").digest(canonical.getBytes(UTF_8)));
    }
}
```

Three details that are easy to get wrong:

**The fingerprint must be canonical.** Hashing the raw request bytes means a client
that reorders JSON fields or changes whitespace on a retry gets a spurious `422`.
Build the fingerprint from the parsed, sorted, normalised domain values.

**The stored response must be the response.** Storing `resource_id` alone and
rebuilding the response on replay means the replay can differ from the original if
the code changed between them. Store the serialised body.

**The payment call is not in this transaction.** It is the single most important
structural decision on this page, and it comes from Topic 55: a transaction pins a
Hikari connection for its whole life. An external call inside it holds a connection
across the network round trip, and — because losing requests **block on the unique
index** until the winner commits — it holds *their* connections too. The idempotency
mechanism turns one long transaction into several. Keep it short.

### The controller

```java
@RestController
@RequestMapping("/api/orders")
class OrderController {

    @PostMapping
    ResponseEntity<?> place(@RequestHeader(value = "Idempotency-Key", required = false) String key,
                            @AuthenticationPrincipal Jwt jwt,
                            @RequestBody @Valid PlaceOrderCommand cmd) {

        if (key == null || key.isBlank()) {
            // A hard requirement, not a nicety. Without a key we cannot make this safe.
            var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
            pd.setTitle("Idempotency-Key required");
            pd.setDetail("POST /api/orders requires an Idempotency-Key header.");
            return ResponseEntity.badRequest().body(pd);
        }

        var outcome = placement.place(key, jwt.getSubject(), cmd);

        return switch (outcome.kind()) {
            case CREATED  -> ResponseEntity.status(201)
                                .location(URI.create("/api/orders/" + outcome.resourceId()))
                                .body(outcome.body());
            case REPLAY   -> ResponseEntity.status(outcome.status())
                                .header("Idempotency-Replayed", "true")
                                .body(outcome.body());
            case IN_FLIGHT-> ResponseEntity.status(409)
                                .header("Retry-After", "1")
                                .body(problem(409, "Request in progress",
                                              "A request with this Idempotency-Key is in progress."));
        };
    }
}
```

**Requiring the header is a deliberate API design choice.** The alternative —
accepting requests without a key and doing nothing — means the safety property is
optional and most clients will not opt in. Make it required, document it, and return
a `ProblemDetail` (Topic 46) that says exactly what to do.

### `POST /api/payments` and the provider's key

The payment endpoint has the same structure plus one addition: it must pass an
idempotency key **onward** to the provider (Topic 111's Trap 3).

```java
@Transactional
public PaymentOutcome pay(String idemKey, String principal, PayCommand cmd) {
    var claim = keys.claim("POST /api/payments", idemKey, fingerprint(cmd), principal, TTL);
    if (!claim.won()) { /* replay / in-flight, as above */ }

    // The key we send the PROVIDER is derived from OUR order, not from the client's
    // header and not from a fresh UUID. It must be identical across every retry
    // Resilience4j makes, and across a process restart and a reconciliation run.
    var providerKey = "orderflow-charge-" + cmd.orderId().value();

    var payment = payments.save(Payment.pending(cmd, providerKey));
    keys.complete("POST /api/payments", idemKey, 202, json.write(...), payment.id().value());
    return PaymentOutcome.accepted(payment.id());
    // The actual charge happens AFTER commit, in the saga. Topic 117.
}
```

Two different keys, two different purposes: the client's key deduplicates *inbound*
requests; the derived provider key deduplicates *outbound* charges. Conflating them
is a common mistake — the client's key is not stable across a reconciliation job that
retries a charge days later, and the provider's key must be.

### The consumer side

```sql
CREATE TABLE processed_event (
    consumer_group text        NOT NULL,
    event_id       text        NOT NULL,
    processed_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (consumer_group, event_id)
) PARTITION BY RANGE (processed_at);
```

```java
@Component
class InventoryConsumer {

    private static final String GROUP = "inventory";

    @KafkaListener(topics = "orderflow.order-placed", groupId = GROUP)
    @Transactional                    // ONE transaction: the dedup row AND the effect
    public void onOrderPlaced(OrderPlacedEvent event) {

        boolean firstTime = jdbc.sql("""
                INSERT INTO processed_event (consumer_group, event_id)
                VALUES (:g, :id)
                ON CONFLICT (consumer_group, event_id) DO NOTHING
                RETURNING event_id
                """)
            .param("g", GROUP).param("id", event.eventId())
            .query(String.class).optional().isPresent();

        if (!firstTime) {
            duplicateCounter.increment();     // MEASURE this -- see Measurement
            log.debug("duplicate_event_skipped group={} event_id={}", GROUP, event.eventId());
            return;                            // commit; the offset advances
        }

        inventory.applyReservation(event.orderId(), event.lines());
    }
}
```

Note `@Transactional` on the listener. The dedup row and the inventory update commit
together. If the inventory update throws, the dedup row rolls back too, so the
redelivery will process it — which is exactly right.

**And note what is still not atomic:** the Kafka offset commit. If the transaction
commits and the pod dies before the offset is committed, Kafka redelivers, and the
dedup row makes the redelivery a no-op. **That is the whole design working.** The
offset does not need to be in the transaction, because the sink is idempotent. This
is Topic 114's point made concrete.

**The `event_id` must come from the producer.** Topic 115's outbox row has an id;
put it in the message (a header or a field). Generating an id in the consumer is
Trap 2 in a different costume: a new id per delivery matches nothing.

### `[BOOT 3.x DELTA]`

| Concern | Boot 3.x | Boot 4.1 | Note |
|---|---|---|---|
| `JdbcClient` | Available from Boot 3.2 | Same | On earlier versions use `NamedParameterJdbcTemplate` |
| Exception translation | `DuplicateKeyException` | Same | Unchanged for years |
| `ProblemDetail` | Boot 3.0+ | Same, plus API versioning | Topic 46 |
| Jackson | Jackson 2 | **Jackson 3 standard** | Affects how you serialise the stored `response_body` |
| `@MockBean` in tests | `@MockBean` (deprecated 3.4+) | `@MockitoBean` | Only affects test code |

---
