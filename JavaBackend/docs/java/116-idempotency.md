# 116 — Idempotent Consumers and Idempotency Keys

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21. Spring, Hibernate and driver versions come from the Spring Boot BOM. Do not pin them by hand and do not quote a version number from this document.
## Project spine: an `Idempotency-Key` header on `POST /api/orders` and `POST /api/payments`, plus a `processed_event` table making the `orderflow` inventory and notification consumers idempotent.

---

## Before anything else — what is and is not in this document

**I have no JVM, no Postgres and no Kafka. Nothing in this document is captured tool
output.**

You will **not** find here:

- a captured `psql` session, a real `EXPLAIN` plan, or a real row count,
- a real stack trace, a real SQLState printed by a real driver, or a real exception message,
- a throughput, latency or duplicate-rate figure presented as measured,
- "the race window was 4 ms" or anything shaped like a measurement.

You get instead, everywhere:

- **the exact SQL, code, command or config**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see → what it means"** table covering the plausible outcomes, including the
  ones that mean your experiment did not reproduce the race rather than that your code is
  correct.

### The labelled exceptions

Twice below I show the **shape** of an exception chain and of a log line, so you can
recognise them. Every value is a placeholder — `<Class>`, `<n>`, `<constraint-name>`,
`<key>` — each carrying the inline label:

> *illustration of the format, not captured output*

**One exception to the no-values rule, and it is deliberate:** I state the Postgres SQLState
for a unique-constraint violation as **`23505`**. That is a value from the SQL standard's
class 23 (integrity constraint violation), it is stable, and it is the string you will
actually branch on. Confirm it yourself with the two-session experiment in Hands-on proof.

### Spec-level facts I state plainly, each with a confirming command

1. **A `SELECT` followed by an `INSERT` is two operations and races.** Confirm: the
   two-session `psql` experiment in Proof 1, which reproduces it deterministically.
2. **A `UNIQUE` constraint makes the database the arbiter: exactly one of N concurrent
   inserters of the same key succeeds.** Confirm: Proof 2.
3. **In Postgres, a second inserter of a duplicate key BLOCKS while the first transaction
   is still open, then fails with `23505` if the first commits, or succeeds if it rolls
   back.** Confirm: Proof 2, watching the second session hang.
4. **A constraint violation aborts the current transaction; you cannot simply catch it and
   carry on in the same transaction.** Confirm: Proof 3 — catch it, then try another
   statement, and read the error.

### THE RULE

> **If your own output disagrees with anything here, YOUR OUTPUT IS THE TRUTH.** Especially
> for the concurrent-`ON CONFLICT` behaviour, which has subtleties across Postgres versions.
> Run the two-session experiment. It takes ninety seconds and it is the entire foundation of
> this topic.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **An idempotency key only works if the uniqueness check and the effect are in the same
> atomic unit.**
>
> **A `SELECT` then an `INSERT` is two operations, and it races: two concurrent requests
> with the same key both see "not found" and both proceed.**
>
> **A `UNIQUE` constraint makes the database the arbiter. Exactly one inserter wins; the
> loser gets SQLState `23505` and must treat that as "someone else is handling this key",
> not as an error.**

And the corollary that catches even people who get the constraint right:

> **The idempotency record must be written in the SAME transaction as the effect. If the
> record commits and the effect does not, you have permanently suppressed a legitimate
> retry. If the effect commits and the record does not, you will do the effect twice.**

---

## The bridge from what you know

### You know the pattern. Skip the theory; the failures are the content.

You have designed idempotent endpoints. You know Stripe's `Idempotency-Key` header. You know
at-least-once delivery means consumers must dedupe. None of that is being re-taught.

What is different in Java is **where the race actually lives** and **what the correct
primitive is**, and there are three specific things your Node experience will not have
prepared you for.

**1. In Node you probably did this with Redis `SET NX`.**

```javascript
const acquired = await redis.set(`idem:${key}`, '1', 'NX', 'EX', 86400);
if (!acquired) return res.status(409).send();
await chargeWallet(...);
```

That is a genuinely atomic check-and-set, and it is a reasonable design. **But notice what
it cannot do: it cannot be in the same transaction as the wallet debit.** Redis and Postgres
are two systems. If the process dies between `SET NX` and the commit, the key exists and the
effect does not — the retry is suppressed and the order is silently lost. **This is Topic
115's dual-write problem wearing a different hat**, and it is why the Java answer puts the
key in the same database as the effect.

**2. Java gives you the constraint violation as an exception, and the exception poisons the
transaction.**

`INSERT` a duplicate key through Hibernate and you get an exception. Catching it feels like
handling it. **It is not**, because in Postgres a failed statement aborts the transaction —
every subsequent statement fails until you roll back or return to a `SAVEPOINT`. Spring's
transaction manager marks the transaction rollback-only. So the naive
`try { insert } catch (DuplicateKeyException e) { return existing(); }` **does not work**
inside a transaction, and this is the single most common way a correct-looking Java
implementation is wrong. There is nothing analogous in your Node/Postgres experience unless
you used explicit savepoints.

**3. `@Transactional` is proxy-based (Topics 40, 54), so "the same transaction" is a
question about the call path, not about the source file.** An idempotency helper called from
the same class does not start a new transaction; called through the proxy with
`REQUIRES_NEW` it takes a *second connection* while holding the first, which is Topic 109's
pool deadlock. Getting this right requires knowing exactly where transaction boundaries sit.

### The in-memory version you will be tempted by, and why it is a lie

This is Topic 92's check-then-act at the API layer, and if you have read Topic 92 you have
already seen the shape:

```java
// WRONG on three separate axes.
private final ConcurrentHashMap<String, Boolean> seen = new ConcurrentHashMap<>();

if (seen.containsKey(key)) return alreadyDone();     // check
seen.put(key, true);                                 // ...then act
walletService.debit(userId, amount);
```

1. **It races.** Two threads both pass `containsKey`. `putIfAbsent` fixes *that* one, but
   not the next two.
2. **It does not survive a restart.** Deploy during a retry storm and every key is forgotten.
3. **It is per-instance.** With three pods behind a load balancer, a retry routed to a
   different pod sees nothing.

**And it fails worse the more concurrency you have.** On platform threads a 200-thread pool
caps how many requests can interleave. Switch to virtual threads (Topics 101, 107) and the
real concurrency rises sharply — which is exactly why Topic 107's measurement section tells
you to count duplicate suppressions at every arrival rate. **A thread cap was hiding this
bug for you.**

**Verdict: STRONG ANALOGUE for the pattern. NO ANALOGUE for the transaction-poisoning
behaviour of a caught constraint violation, and a specifically misleading instinct that a
separate atomic store (Redis) is good enough.**

---

## What is this?

Two related mechanisms with different clients and different failure modes. Do not conflate
them.

| | **Idempotency key** (API layer) | **Processed-event table** (consumer layer) |
|---|---|---|
| Who supplies the identifier | **The client**, in a header | The producer, as an event ID in the message |
| Transport | HTTP `POST` | Kafka |
| Why duplicates happen | Client retry after a timeout, a proxy retry, a double-click | At-least-once delivery: rebalance, redelivery after a failed commit (Topic 113), an outbox relay that published twice (Topic 115) |
| Scope of the key | Per user, per endpoint | Per consumer group, per event |
| What a duplicate must return | **The original response** — same status, same body | Nothing; silently skip |
| Where the record lives | `idempotency_key` table, same DB as the effect | `processed_event` table, same DB as the effect |
| Retention | Hours to days; a client will not retry a week later | Longer; a redelivery can follow a long outage |

**The unifying rule, and the reason they are one topic:**

> **In both cases, a uniqueness constraint on the identifier, inserted in the same transaction
> as the effect, is what makes the operation idempotent. Everything else is ergonomics.**

### What "idempotent" must actually mean to a client

Three different behaviours all get called idempotent. **Suppressed** — the duplicate does
nothing and returns a generic 409: safe, and frequently useless, because the client that
timed out still does not know whether the original succeeded. **Replayed** — the duplicate
returns the original response, byte for byte, with the original status: this is what clients
need, and why the response is stored in the record. **Recomputed** — the duplicate re-runs
the work and happens to produce the same result: not idempotency, luck, and it breaks the
moment the work has a side effect or reads changed state.

**Aim for replayed.** `POST /api/orders` returning `201` the first time must return the same
order — and the same `201`, or a documented `200` — on a retry. Otherwise the client retries
forever, or gives up on an order that exists.

### `[BOOT 3.x DELTA]`

The mechanism is identical on 3.x. Spring's translation of SQLState `23505` to
`DuplicateKeyException` (a `DataIntegrityViolationException` subclass) is long-standing and
unchanged, so the code here works on both lines. `ProblemDetail` (RFC 9457, Topic 46) also
exists from Framework 6, so the error bodies below port unchanged — but on a 2.x codebase
you will be hand-rolling them.

---

## Why does it matter?

**1. Money.** `POST /api/payments` charging a wallet twice is a customer-visible financial
error and, at any scale, a regulatory conversation. Duplicate submission is not exotic: a
client timeout firing while the server is still committing produces exactly this, and your
p99 tail guarantees it happens.

**2. At-least-once is not a choice you get to make.** Topic 113 established that Kafka
consumers redeliver on rebalance. Topic 115 established that an outbox relay is at-least-once
by construction — it publishes, then marks published, and a crash between those publishes
again. **You cannot remove duplicates from the transport. You can only make the consumer not
care.** Idempotency is the price of the outbox; if you skipped it you did not finish Topic 115.

**3. The naive fix looks correct and passes tests.** A `SELECT`-then-`INSERT` check passes
every single-threaded test you will write. It fails only under concurrency, only sometimes,
and produces a *duplicate side effect* rather than an exception — so nothing alerts. The first
evidence is a customer complaint. This is Topic 92's lesson at a different layer, and the
reason the concurrency trace comes before any code below.

---

## Machine-level reality

### 1. What a `UNIQUE` constraint actually is

A `UNIQUE` constraint in Postgres is implemented by a **unique B-tree index**. When you
`INSERT`, the engine:

1. inserts the heap tuple,
2. attempts to insert the index entry,
3. finds an existing entry with the same key, and — this is the important part — **checks
   the visibility and transaction status of the row that entry points at.**

Step 3 is where the concurrency semantics live:

| State of the conflicting row | What happens to your `INSERT` |
|---|---|
| Committed | Immediate error: SQLState **`23505`**, unique violation |
| Inserted by a transaction still open | **Your `INSERT` BLOCKS**, waiting on that transaction's ID |
| ...and that transaction commits | You then get `23505` |
| ...and that transaction rolls back | Your `INSERT` **succeeds** |
| Deleted by a committed transaction | Your `INSERT` succeeds |

**The blocking row is the one that makes this work as a mutual-exclusion primitive.** Two
concurrent requests with the same key: the first inserts and holds; the second blocks until
the first's fate is decided, then either fails (first succeeded) or proceeds (first rolled
back). **That is exactly the semantics you want, and you get it without writing a lock.**

**Verify this yourself.** It is Proof 2 and it takes two `psql` windows. Do not take my word
for it.

### 2. What a violation costs

Not free, and worth knowing before designing a hot path around it. A failed insert leaves a
**dead heap tuple** that must be vacuumed, so high duplicate rates cause bloat. It **consumes
a sequence value** if you used `bigserial`/`IDENTITY` — sequences are not transactional, so
gaps appear (harmless; people file bugs about it). It **aborts the transaction**, which is the
expensive one, covered next. And in Java the driver builds a `PSQLException`, Spring
translates it, and `fillInStackTrace` walks a deep Spring/Hibernate stack — measurable at a
high duplicate rate, and visible in a profile (Topic 78).

**The design consequence:** if duplicates are *rare* (a client retry after a timeout — the
normal case), let the constraint throw and handle it. If duplicates are *common* (a consumer
reprocessing a replayed topic), prefer `INSERT ... ON CONFLICT DO NOTHING`, which returns a
row count instead of throwing and does not abort anything.

### 3. The aborted transaction — the Java-specific trap

In Postgres, **any** statement error aborts the current transaction. Subsequent statements
fail with "current transaction is aborted, commands ignored until end of transaction block".

In Spring, the consequences compound:

- Hibernate's `SessionImpl` marks itself for rollback when a `ConstraintViolationException`
  escapes flush; the `EntityManager` is in an unusable state.
- `TransactionAspectSupport` marks the transaction **rollback-only**. Even if your `catch`
  block returns normally, the commit at the proxy boundary throws
  `UnexpectedRollbackException`.
- So this **does not work**:

```java
@Transactional
public OrderResponse place(String key, OrderRequest req) {
    try {
        idempotencyRepo.save(new IdempotencyRecord(key));   // may violate the constraint
    } catch (DataIntegrityViolationException e) {
        return replayExisting(key);      // <-- transaction is ALREADY doomed.
    }                                    //     This SELECT fails, or the commit throws.
    return doTheWork(req);
}
```

> *illustration of the format, not captured output*
> ```
> org.springframework.dao.DataIntegrityViolationException: could not execute statement
>     [ERROR: duplicate key value violates unique constraint "<constraint-name>"
>      Detail: Key (user_id, endpoint, key)=(<n>, <endpoint>, <key>) already exists.]
>     ...
> Caused by: org.hibernate.exception.ConstraintViolationException: could not execute statement
> Caused by: org.postgresql.util.PSQLException: ERROR: duplicate key value violates ...
> ```
>
> Note the three-layer chain: Spring's translated exception, Hibernate's, the driver's.
> **The SQLState you branch on lives on the innermost one.**

**The three ways out, and when each is right:**

| Approach | How | When |
|---|---|---|
| **`ON CONFLICT DO NOTHING`** | Native query returning the affected row count. **Never throws, never aborts.** | Default. Especially for consumers with high duplicate rates. |
| **Let it abort and retry the whole request** | Do not catch. Map the violation at the controller boundary to a replay path in a *fresh* transaction. | When you want the simplest code and duplicates are rare. |
| **`SAVEPOINT`** | `Propagation.NESTED` (JDBC savepoints), or explicit savepoint SQL. | When you must continue the same transaction. Adds round trips; use sparingly. |

**`ON CONFLICT DO NOTHING` is the default answer**, and the reason is precisely that it
converts an exception into a number.

### 4. Where the idempotency record must live relative to the effect

This is the corollary of the mechanical statement, and it is the part people get wrong after
they get the constraint right.

There are three placements. Only one is correct.

| Placement | Crash between record and effect | Verdict |
|---|---|---|
| **Record committed BEFORE the effect** (separate transaction) | Key exists, effect never happened. **The client's retry is suppressed forever.** The order is silently lost. | **WRONG — silent data loss** |
| **Record committed AFTER the effect** (separate transaction) | Effect happened, no key. **The retry does it again.** Double charge. | **WRONG — the bug you were fixing** |
| **Record and effect in the SAME transaction** | Both commit or neither does. A retry either replays a completed record or does the work. | **CORRECT** |

**Say this in an interview and you separate yourself immediately** — most people stop at "use
a unique constraint" and never state where the record lives.

**And it is why Redis is not sufficient alone.** `SET NX` plus a Postgres commit is a dual
write with a window between. Redis is a fine *optimisation* — a fast pre-check — but the
authority must be the row in the same transaction as the effect.

### 5. Why the record needs states, not just existence

A single row meaning "seen" is not enough, because there are three situations: **not seen**
(do the work); **seen and finished** (replay the stored response); and **seen and still
running** — the original is in flight right now, so you cannot replay (no response yet) and
must not act (that is the double charge).

So the record carries a state: `IN_PROGRESS`, inserted at the start, and `COMPLETED`, updated
with the response — **both in the same transaction as the effect.** Because they are one
transaction, `IN_PROGRESS` is only ever visible to *other* transactions while the original is
still running, so a concurrent duplicate blocks on the insert (§1) rather than reading a
half-state. That is why the state machine is simpler than it looks.

**The one case that needs care:** the process crashes mid-transaction. Postgres rolls it back,
the row disappears, and a retry proceeds cleanly. **That is the correct outcome, and a direct
consequence of putting the record in the same transaction.** A design that committed
`IN_PROGRESS` separately would leave a stale row needing a lease and a reaper — real
complexity you avoid by not splitting the transaction.

---

## Concurrency trace

**Read this before any correct code.** The whole topic exists because this interleaving is
possible, and the trace is the artefact to reproduce in your head under interview pressure.

### The naive implementation being traced

```java
// WRONG. This is the code the trace below executes.
@Transactional
public PaymentResponse pay(String idempotencyKey, PaymentRequest req) {

    // Step 1: CHECK
    Optional<IdempotencyRecord> existing = idempotencyRepo.findByKey(idempotencyKey);
    if (existing.isPresent()) {
        return replay(existing.get());
    }

    // Step 2: ACT
    walletService.debit(req.userId(), req.amount());
    Payment payment = paymentRepo.save(new Payment(req));
    idempotencyRepo.save(new IdempotencyRecord(idempotencyKey, payment));
    return PaymentResponse.of(payment);
}
```

The client posts `£40` with `Idempotency-Key: pay-8f2c`. The response is slow, the client's
HTTP timeout fires, and the client retries with **the same key**. Both requests are now in
flight.

### The interleaving

Request A is the original. Request B is the client's retry with the same key. Read the
Step column top to bottom; each row is one moment in time.

| Step | Request A (original) | Request B (client retry, same key) | State / outcome |
|---|---|---|---|
| 1 | `SELECT` key `pay-8f2c` → not found | — | key absent |
| 2 | — | `SELECT` key `pay-8f2c` → not found | key absent |
| 3 | debits wallet £40 | — | balance −40 |
| 4 | — | debits wallet £40 | **customer charged twice** |
| 5 | `INSERT` payment row | — | 2 payment rows will exist |
| 6 | — | `INSERT` payment row | 2 payment rows |
| 7 | `INSERT` idempotency key | — | key row (A) |
| 8 | — | `INSERT` idempotency key | **`23505`** or a second row — see below |
| 9 | COMMIT → `201` | — | A's effects durable |
| 10 | — | COMMIT or rollback | see the four outcomes below |

**Steps 1 and 2 are the entire bug.** Both requests read "not found", because a `SELECT`
takes no lock that would prevent the other from reading the same absence. Under Postgres
READ COMMITTED — `orderflow`'s isolation level (Topic 55) — a row that does not exist yet is
simply not there for either reader. **There is nothing to lock.**

**What happens at step 8 depends on whether you have the constraint, and the four outcomes
are all instructive:**

| Situation at step 8 | Outcome | Verdict |
|---|---|---|
| **No unique constraint** | B inserts a second key row happily. Both commit. | **Double charge, permanent, silent. Two payments, two key rows, one angry customer.** |
| **Unique constraint present, A already committed** | B gets `23505` immediately | B's transaction aborts and rolls back — **the wallet debit at step 4 is undone.** Correct outcome, arrived at by accident. |
| **Unique constraint present, A still open** | B **blocks** at step 8 until A commits, then gets `23505` | Same as above. The block is the constraint doing mutual exclusion for you. |
| **Unique constraint present, `catch` inside the transaction** | B catches `DataIntegrityViolationException` and tries to `SELECT` the existing record | **Fails.** The transaction is already aborted (Machine-level reality §3). B gets `UnexpectedRollbackException` at the commit boundary. |

Read the second and third rows carefully, because they contain the fix in embryo: **the
unique constraint alone already prevents the double charge**, by aborting B's whole
transaction — including the wallet debit. What it does *not* do is give B a useful answer. B
gets a 500, the client retries again, and the design work remaining is turning that abort
into a **replay of A's response**.

### The same race in the consumer, for completeness

`order.placed` is delivered twice to the inventory consumer — a rebalance redelivered it
(Topic 113), or the outbox relay published twice (Topic 115).

| Step | Consumer instance 1 | Consumer instance 2 | State / outcome |
|---|---|---|---|
| 1 | `SELECT` event `evt-3d91` in `processed_event` → not found | — | not processed |
| 2 | — | `SELECT` event `evt-3d91` → not found | not processed |
| 3 | `UPDATE inventory SET reserved = reserved + 3` | — | reserved +3 |
| 4 | — | `UPDATE inventory SET reserved = reserved + 3` | **reserved +6 for one order** |
| 5 | `INSERT processed_event` | — | row (1) |
| 6 | — | `INSERT processed_event` | `23505`, or a duplicate row |
| 7 | commit; commit offset | rollback or commit | **stock is oversold** |

**Same shape, different blast radius.** Step 4 is Topic 52's oversell arriving through a
completely different door: not two orders competing for stock, but *one* order counted
twice. Your `@Version` optimistic locking does not help — both consumers are applying a
legitimate-looking increment.

**Now, and only now, the code.**

---

## Example 1 — minimal

### 1a — the schema

```sql
CREATE TABLE idempotency_key (
    id              bigserial   PRIMARY KEY,
    user_id         bigint      NOT NULL,
    endpoint        text        NOT NULL,
    idem_key        text        NOT NULL,
    request_hash    text        NOT NULL,
    state           text        NOT NULL,          -- IN_PROGRESS | COMPLETED
    response_status int,
    response_body   jsonb,
    created_at      timestamptz NOT NULL DEFAULT now(),
    completed_at    timestamptz,
    CONSTRAINT uq_idempotency_key UNIQUE (user_id, endpoint, idem_key)
);

CREATE INDEX idx_idempotency_created_at ON idempotency_key (created_at);
```

Four decisions in that DDL, each of which someone gets wrong:

| Decision | Why |
|---|---|
| Key scoped by `(user_id, endpoint, idem_key)` | A client's key is unique to *that client*. Global uniqueness lets user X's key collide with user Y's — a cross-tenant denial of service, and arguably an information leak. Scoping by endpoint stops a key reused across `/orders` and `/payments` from aliasing. |
| `request_hash` stored | Detects a client reusing a key for a *different* body. Without it you replay the wrong response — worse than a duplicate. |
| `response_status` and `response_body` stored | Replay needs the original answer. A bare 409 is not useful to a client that timed out. |
| `created_at` indexed | You must delete old keys. Without the index, the cleanup job scans the table. |

### 1b — the correct insert: `ON CONFLICT DO NOTHING`

```java
public interface IdempotencyRepository extends Repository<IdempotencyRecord, Long> {

    @Modifying
    @Query(value = """
        INSERT INTO idempotency_key
            (user_id, endpoint, idem_key, request_hash, state, created_at)
        VALUES (:userId, :endpoint, :key, :hash, 'IN_PROGRESS', now())
        ON CONFLICT (user_id, endpoint, idem_key) DO NOTHING
        """, nativeQuery = true)
    int tryClaim(@Param("userId") long userId,
                 @Param("endpoint") String endpoint,
                 @Param("key") String key,
                 @Param("hash") String hash);
}
```

**`tryClaim` returns 1 if you won the key and 0 if someone else already has it. It never
throws and never aborts the transaction.** That is the whole reason to use it.

**Read the return value as a claim, not as a check:**

```java
int claimed = idempotencyRepo.tryClaim(userId, "/api/payments", key, hash);
if (claimed == 1) {
    // We own this key. Do the work in THIS transaction.
} else {
    // Someone else owns it. Either they finished (replay) or they are running (409).
}
```

The difference from `SELECT`-then-`INSERT` is not stylistic. **`tryClaim` is one atomic
statement.** There is no window between deciding and acting, because deciding *is* acting.

### 1c — the consumer version, which is even simpler

```sql
CREATE TABLE processed_event (
    consumer     text        NOT NULL,
    event_id     uuid        NOT NULL,
    processed_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (consumer, event_id)
);
```

```java
@Transactional
public void handle(OrderPlaced event) {
    int claimed = processedEventRepo.tryClaim("inventory-consumer", event.eventId());
    if (claimed == 0) {
        duplicateCounter.increment();      // Topic 118: this is a METRIC, not a log line
        return;                            // already processed; nothing to do
    }
    inventoryService.reserve(event.orderId(), event.lines());
    // Both the claim and the reservation commit together, or neither does.
}
```

**Nine lines, and it is correct.** The two things that make it correct:

1. The claim is one atomic statement.
2. The claim and the effect are in **one** `@Transactional` — so the commit is all-or-nothing.

**The `consumer` column matters** because the same event goes to several consumer groups.
The inventory consumer and the notification consumer must each process it once, and a
single-column primary key on `event_id` would let whichever arrived first suppress the other.

---

## Example 2 — production scenario (on the project spine)

### The API filter

Idempotency for `orderflow` sits in one place: a filter that wraps the handler, so the
business code does not repeat the pattern per endpoint.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 50)   // after security, before the controller
public class IdempotencyFilter extends OncePerRequestFilter {

    private static final String HEADER = "Idempotency-Key";
    private static final Set<String> GUARDED =
            Set.of("/api/orders", "/api/payments");

    private final IdempotencyService service;
    private final ObjectMapper mapper;

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        return !"POST".equals(request.getMethod())
                || !GUARDED.contains(request.getRequestURI());
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        String key = request.getHeader(HEADER);
        if (key == null || key.isBlank()) {
            // A guarded endpoint REQUIRES the header. Rejecting is the correct,
            // and the unpopular, choice -- see Trap 2.
            writeProblem(response, HttpStatus.BAD_REQUEST,
                    "Missing Idempotency-Key header");
            return;
        }
        if (key.length() > 255) {
            writeProblem(response, HttpStatus.BAD_REQUEST, "Idempotency-Key too long");
            return;
        }

        // Cache the body so we can hash it AND still let the controller read it.
        ContentCachingRequestWrapper wrapped = new ContentCachingRequestWrapper(request);
        byte[] body = wrapped.getInputStream().readAllBytes();
        String hash = sha256Hex(body);

        long userId = currentUserId();
        String endpoint = request.getRequestURI();

        IdempotencyOutcome outcome = service.claimOrLookup(userId, endpoint, key, hash);

        switch (outcome) {
            case Claimed ignored -> {
                CachingResponseWrapper out = new CachingResponseWrapper(response);
                chain.doFilter(wrapped, out);
                // Persist the response INSIDE the same transaction as the effect --
                // see IdempotencyService below for how that boundary is arranged.
                service.complete(userId, endpoint, key,
                        out.getStatus(), out.getCapturedBody());
                out.copyBodyToResponse();
            }
            case Replay replay -> {
                response.setStatus(replay.status());
                response.setHeader("Idempotent-Replay", "true");
                response.getOutputStream().write(replay.body());
            }
            case InProgress ignored -> {
                response.setHeader("Retry-After", "1");
                writeProblem(response, HttpStatus.CONFLICT,
                        "A request with this Idempotency-Key is still being processed");
            }
            case KeyReuseMismatch ignored ->
                writeProblem(response, HttpStatus.UNPROCESSABLE_ENTITY,
                        "This Idempotency-Key was used with a different request body");
        }
    }
}
```

`IdempotencyOutcome` is a sealed interface over records (Topic 28), so the `switch` is
exhaustive at compile time — add a fifth outcome later and every call site becomes a compile
error rather than a silent fall-through.

```java
public sealed interface IdempotencyOutcome {
    record Claimed()                              implements IdempotencyOutcome {}
    record Replay(int status, byte[] body)        implements IdempotencyOutcome {}
    record InProgress()                           implements IdempotencyOutcome {}
    record KeyReuseMismatch()                     implements IdempotencyOutcome {}
}
```

### The transaction boundary — the part that is actually hard

A filter runs **outside** any transaction. The controller's `@Transactional` service method
is where the effect commits. So "claim and effect in the same transaction" requires the
claim to happen *inside* that method, not in the filter.

**Two designs. Know both and know why one is chosen.**

**Design 1 — claim in the service, filter only replays.** The filter's `claimOrLookup` does
a plain read to catch the obvious replay case; the *authoritative* claim happens inside the
transactional service method.

```java
@Service
public class PaymentService {

    @Transactional
    public PaymentResponse pay(IdempotencyContext idem, PaymentRequest req) {

        // 1. CLAIM -- atomic, first statement, same transaction as everything below.
        int claimed = idempotencyRepo.tryClaim(
                idem.userId(), idem.endpoint(), idem.key(), idem.requestHash());
        if (claimed == 0) {
            // Someone else owns the key. Do NOT proceed. The caller decides
            // whether that is a replay or an in-progress conflict.
            throw new IdempotencyKeyHeld(idem.key());
        }

        // 2. THE EFFECT -- same transaction.
        walletService.debit(req.userId(), req.amount());
        Payment payment = paymentRepo.save(Payment.from(req));

        // 3. COMPLETE -- same transaction. The response is durable with the effect.
        PaymentResponse body = PaymentResponse.of(payment);
        idempotencyRepo.markCompleted(
                idem.userId(), idem.endpoint(), idem.key(),
                201, mapper.writeValueAsBytes(body));

        return body;
        // COMMIT: claim, wallet debit, payment row and stored response, atomically.
    }
}
```

**This is the design `orderflow` uses.** Everything that matters — the claim, the wallet
debit, the payment row, and the stored response — commits or does not commit as one unit.
There is no window.

**Design 2 — a separate `REQUIRES_NEW` transaction for the claim.** Tempting, because it
lets the filter own the whole pattern. **Reject it**, for two reasons you should be able to
state:

1. It reintroduces the dual-write window. The claim commits separately from the effect, so a
   crash between them either suppresses a legitimate retry or permits a double charge —
   whichever order you choose (Machine-level reality §4).
2. `REQUIRES_NEW` takes a **second connection while holding the first** (Topics 54, 109).
   At pool size N with N concurrent requests, that is a pool deadlock.

**Say both reasons in an interview.** The correctness one is the important one; the pool one
shows you have read Topic 109.

### The lookup path

```java
@Transactional(readOnly = true)
public IdempotencyOutcome claimOrLookup(long userId, String endpoint,
                                        String key, String hash) {
    return idempotencyRepo.find(userId, endpoint, key)
            .<IdempotencyOutcome>map(record -> {
                if (!record.requestHash().equals(hash)) {
                    return new IdempotencyOutcome.KeyReuseMismatch();
                }
                return switch (record.state()) {
                    case COMPLETED   -> new IdempotencyOutcome.Replay(
                                            record.responseStatus(), record.responseBody());
                    case IN_PROGRESS -> new IdempotencyOutcome.InProgress();
                };
            })
            .orElseGet(IdempotencyOutcome.Claimed::new);
}
```

**Name the honest weakness rather than hiding it:** this read is *advisory*. Between this
`SELECT` and the service's `tryClaim`, another request can claim the key — the check-then-act
race again. That is acceptable here, and **only** because the authoritative claim downstream
is atomic: if `tryClaim` returns 0, the service throws `IdempotencyKeyHeld` and the filter
converts it to a replay or a 409. The advisory read is a latency optimisation, not a
correctness mechanism.

**The distinction to internalise: a racy read is acceptable when it is a hint; it is never
acceptable when it is the decision.**

### The consumers

```java
@Component
public class InventoryEventConsumer {

    @KafkaListener(topics = "order.placed", groupId = "inventory-consumer")
    @Transactional
    public void onOrderPlaced(OrderPlaced event, Acknowledgment ack) {
        int claimed = processedEventRepo.tryClaim("inventory-consumer", event.eventId());
        if (claimed == 0) {
            duplicates.increment();
            ack.acknowledge();          // Acknowledge! It IS handled. Not acking replays forever.
            return;
        }
        inventoryService.reserve(event.orderId(), event.lines());
        ack.acknowledge();
    }
}
```

Three details that are easy to get wrong:

| Detail | Why |
|---|---|
| Acknowledge on the duplicate path | A duplicate **is** handled. Failing to ack makes the consumer redeliver it forever and lag climbs (Topic 113). |
| `event.eventId()` is the **producer's** ID | Not the Kafka offset, and not a hash of the payload. The outbox row's ID (Topic 115) is the natural choice: it is generated once, in the same transaction as the order, and survives republication by the relay. |
| The notification consumer uses its own `consumer` value | Same event, different group, must process independently. |

**The notification consumer has a subtlety the inventory consumer does not:** its effect —
sending an email — is *not* in the database, so it cannot be in the transaction. The
`processed_event` insert commits, and then the email is sent. A crash in between loses the
email permanently.

**The honest answer, and this is a design decision you must be able to defend:** for a
notification, that is usually acceptable, and the alternative (send first, then record) risks
sending twice. **Choose the failure you can live with and write it down.** If you cannot live
with either, the notification itself needs an outbox (Topic 115), which is the same problem
one layer down. Do not pretend the transaction covers the email.

### Retention

```sql
-- Runs nightly. Keys older than the client's plausible retry horizon are dead weight.
DELETE FROM idempotency_key
 WHERE created_at < now() - interval '7 days';

-- processed_event is kept longer: a redelivery can follow a long outage.
DELETE FROM processed_event
 WHERE processed_at < now() - interval '30 days';
```

**The retention window is a correctness parameter, not housekeeping.** Delete a key while a
client might still retry with it and you reopen the double-charge window. Derive it from the
client's documented retry policy, put the derivation in a comment, and delete in batches so
the transaction does not hold locks for minutes (Topic 55).

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — check-then-act on an idempotency key

**Wrong approach.** `SELECT` the key, and `INSERT` if absent. The code in the Concurrency
trace.

**Exact symptom.** Two payment rows for one logical payment, a wallet balance reduced twice,
and **no error anywhere**. In the logs, two requests seconds apart with the same
`Idempotency-Key`, both returning `201`. Discovered by reconciliation or by the customer.
Reproduces only under concurrency, so every unit test passes.

**Root cause.** A `SELECT` takes no lock on a row that does not exist. Under READ COMMITTED
both readers see absence. The check and the act are two statements with a window between
them. **This is Topic 92's `ConcurrentHashMap` race at the API layer, with money attached.**

**Fix.** Make the check and the act one statement, and let the database arbitrate:

```java
int claimed = idempotencyRepo.tryClaim(userId, endpoint, key, hash);
if (claimed == 0) { /* someone else owns it */ }
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Duplicate-suppression counter increments under concurrent load | Working. The constraint is arbitrating. |
| Two payment rows with the same key | The constraint is missing or scoped wrong. `\d idempotency_key` in `psql` and check it exists. |
| `23505` reaching your error handler as a 500 | You are letting the raw violation escape. Convert it to a replay or a 409. |
| Counter stays at zero under a load test | **Your test is not actually concurrent.** See Proof 4 — a loop is not concurrency. |
| Duplicates appear only above the old thread-pool concurrency | The thread cap was masking the race. Topic 107's measurement section predicts exactly this. |

### Trap 2 — a server-generated idempotency key

**Wrong approach.** "We'll generate the key ourselves so clients don't have to."

```java
String key = UUID.randomUUID().toString();       // generated per REQUEST
```

Or the subtler version: derive it from the request body.

**Exact symptom.** Duplicates still occur, at exactly the same rate as before. The
idempotency table fills with rows that are never matched. Someone concludes "idempotency
doesn't work" and removes it.

**Root cause.** **The key must identify the client's *intent*, and only the client knows
when two requests are the same intent.** A server-generated UUID is new on every request,
including the retry — so the retry claims a fresh key and does the work again. The
idempotency table is a write-only log.

The body-hash variant is worse in a different direction: two *legitimately distinct* orders
with identical contents (same customer buying the same item twice, deliberately) hash the
same and the second is wrongly suppressed. **You have converted a double-charge bug into a
lost-order bug**, which is harder to detect.

**Fix.** The client supplies the key; the same key on the retry.

```
POST /api/payments
Idempotency-Key: 9f2c4a1e-...        <-- generated ONCE by the client, per intent
```

And on a guarded endpoint, **require it.** Rejecting the request with a 400 is unpopular and
correct: an optional idempotency key is idempotency that does not work, and it will be
discovered during an incident.

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Every row in `idempotency_key` has a distinct key, none ever matched | Server-generated keys. The whole table is inert. |
| Clients send the header but the key changes on retry | The client regenerates per HTTP attempt rather than per intent. **This is a client bug and it is your job to document it**, prominently, with an example. |
| Legitimate repeat orders rejected as duplicates | Body-hash keys. Revert to a client-supplied key. |
| A key reused for a different body | The `request_hash` check is doing its job. Return 422; do not replay the wrong response. |

### Trap 3 — the idempotency record in a different transaction from the effect

**Wrong approach.** The tidy layering instinct — the idempotency concern is
infrastructure, so it gets its own transaction:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordKey(String key) { ... }        // commits on its own
```

**Exact symptom — and it differs by ordering, which is the tell:**

- **Record first:** a customer reports that their order "vanished". The idempotency row
  exists, no order does. Their retry returned a replay of a response for work that never
  happened. **Silent, permanent data loss**, and the idempotency table is the evidence
  nobody thinks to look at.
- **Effect first:** the original double-charge symptom, at a lower rate — only when a crash
  lands in the window.
- **Either way**, under load: `hikaricp.connections.pending` climbs and eventually every
  request times out, because each request holds two connections (Topic 109).

**Root cause.** Two transactions means two commits means a window. It is Topic 115's
dual-write failure with the outbox row replaced by an idempotency row, and it has the same
answer: **make it one transaction.**

**Fix.** Claim, effect and completion in one `@Transactional` method (Example 2, Design 1).
The filter does an advisory read and maps outcomes; it never writes.

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| `idempotency_key` rows with no corresponding order or payment | Record committed before the effect. Data loss. Count these — it is a reconciliation query you should run permanently. |
| Effects with no key row | Record committed after the effect. Duplicates are possible. |
| `hikaricp.connections.pending` rising with request rate | `REQUIRES_NEW` is holding two connections per request. Topic 109. |
| `UnexpectedRollbackException` at the controller boundary | You caught a constraint violation inside the transaction. Machine-level reality §3. |

**And the permanent check:** a scheduled query counting `COMPLETED` keys with no matching
effect row (Measurement, query (a)). Any nonzero result means a key committed without its
effect — a transaction-boundary bug that no test will find.

### Trap 4 — catching the constraint violation inside the transaction

**Wrong approach.** The constraint is in place, the exception is caught, the code looks
defensive:

```java
@Transactional
public PaymentResponse pay(String key, PaymentRequest req) {
    try {
        idempotencyRepo.save(new IdempotencyRecord(key));
    } catch (DataIntegrityViolationException e) {
        return replayExisting(key);           // <-- transaction already doomed
    }
    ...
}
```

**Exact symptom.** Not a duplicate charge — a *confusing 500*. The `catch` runs, the replay
`SELECT` either fails with "current transaction is aborted" or appears to succeed and then
the commit throws `UnexpectedRollbackException`. The stack trace names
`TransactionAspectSupport`, not your code, so the reported bug is "intermittent 500 on
retries" and nobody connects it to idempotency.

**Root cause.** A statement error aborts the Postgres transaction; Hibernate's session is
unusable; Spring marks the transaction rollback-only. **Catching an exception does not undo
the abort.** This is the single most common way a *correct-looking* Java implementation is
wrong, and there is no equivalent in your Node experience.

**Fix — in order of preference:**

```java
// 1. Best: never throw. ON CONFLICT DO NOTHING returns a count.
int claimed = idempotencyRepo.tryClaim(...);

// 2. Acceptable: let it abort, handle the replay OUTSIDE, in a new transaction.
//    The controller advice catches IdempotencyKeyHeld and calls a read-only lookup.

// 3. Only if you must continue this transaction: a savepoint.
//    Propagation.NESTED uses JDBC savepoints. Costs round trips. Use sparingly.
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| `UnexpectedRollbackException` from the proxy boundary | You caught something inside the transaction and the transaction was already rollback-only. |
| "current transaction is aborted, commands ignored until end of transaction block" | The Postgres-level statement of the same fact. |
| It works in an H2 test and fails on Postgres | H2's abort semantics differ. **This is why Topic 61's Testcontainers rule exists**: test against the real engine. |
| No exception at all with `ON CONFLICT DO NOTHING` | Correct. That is the point of the form. |

### Trap 5 — an in-memory or Redis-only dedupe cache

**Wrong approach.** A `ConcurrentHashMap` or a Redis `SET NX` as the sole authority.

**Exact symptom.** Three failures with three distinct signatures. **After a deploy:** a burst
of duplicates, because the in-memory map was lost — correlating with restart times, which
makes it look like a deploy bug. **With more than one pod:** a steady duplicate rate roughly
`(instances − 1) / instances`, because a retry routed elsewhere sees nothing. **With Redis:**
rare duplicates, or rare suppressed-but-not-done operations, concentrated around crashes.

**Root cause.** In-memory: the state is neither shared nor durable. Redis: it is shared and
durable but **not in the same transaction as the effect**, so it is Trap 3 across two systems
— where you cannot even reach for a savepoint. This is Topic 115's dual write again.

**Fix.** The authority is a row in the same database as the effect, in the same transaction.
Redis, if you want it, is a read-through pre-check only, and a stale or missing entry must be
harmless:

```java
// Optional optimisation. NEVER the authority.
if (redis.exists("idem:" + key)) {
    return replayFromDatabase(key);      // still read the real answer from Postgres
}
```

| What you see | What it means |
|---|---|
| Duplicate rate spikes at deploy times | In-memory state, lost on restart. |
| Duplicate rate roughly `(n−1)/n` with n instances | Per-instance state. Scale-out made it worse — that is the giveaway. |
| Duplicates only around crashes | The Redis/Postgres dual-write window. |
| Duplicate rate unchanged after adding Redis | Redis is a cache in front of a still-broken check. The race is downstream of it. |

---

## Hands-on proof

Every claim reduces to one of five observations. Three need only two `psql` windows.

### Proof 1 — reproduce the check-then-act race by hand

Two `psql` sessions against `orderflow`'s database. Session 1 on the left, session 2 on the
right; run them in the numbered order.

```sql
-- Session 1                              -- Session 2
BEGIN;
SELECT * FROM idempotency_key
 WHERE idem_key = 'proof-1';              -- (1) returns 0 rows

                                          BEGIN;
                                          SELECT * FROM idempotency_key
                                           WHERE idem_key = 'proof-1';   -- (2) also 0 rows

UPDATE wallet SET balance = balance - 40
 WHERE user_id = 1;                       -- (3)

                                          UPDATE wallet SET balance = balance - 40
                                           WHERE user_id = 1;            -- (4) BLOCKS on the row lock
COMMIT;                                   -- (5)
                                          -- (4) now proceeds: balance -80
                                          COMMIT;                        -- (6)
```

**WHAT TO LOOK FOR:** the wallet balance after step 6.

| What you see | What it means |
|---|---|
| Balance reduced by 80 | **The race, reproduced deterministically.** Both sessions read "not found" and both debited. Note that the row lock at step 4 did not help — it serialised the debits, it did not prevent the second one. |
| Session 2 blocks at step 4 and then proceeds | Correct and instructive: row locks serialise, they do not deduplicate. |
| Balance reduced by 40 | You have a constraint or trigger you did not know about. Find it — `\d wallet`. |

**The lesson to write down:** *locking the row you are about to modify does not protect you,
because the thing you needed to lock was a row that did not exist yet.*

### Proof 2 — the constraint arbitrates, and the loser blocks

```sql
-- Session 1                              -- Session 2
BEGIN;
INSERT INTO idempotency_key
  (user_id, endpoint, idem_key, request_hash, state)
VALUES (1, '/api/payments', 'proof-2', 'h', 'IN_PROGRESS');   -- (1) succeeds

                                          BEGIN;
                                          INSERT INTO idempotency_key
                                            (user_id, endpoint, idem_key, request_hash, state)
                                          VALUES (1, '/api/payments', 'proof-2', 'h', 'IN_PROGRESS');
                                          -- (2) BLOCKS. Watch it hang.
COMMIT;                                   -- (3)
                                          -- (2) now fails: SQLSTATE 23505
```

**WHAT TO LOOK FOR:** session 2 hanging at step 2, then the SQLState after step 3.

| What you see | What it means |
|---|---|
| Session 2 hangs, then errors with `23505` | **The constraint is doing mutual exclusion for you.** This is the mechanism the whole design rests on. |
| Session 2 errors immediately without hanging | Session 1 had already committed. Re-run with the ordering above. |
| Session 1 rolled back instead: session 2 **succeeds** | Also correct, and important. The loser gets to proceed if the winner failed — which is why a crashed request does not permanently block its own retry. |
| No error at all | The unique constraint does not exist. `\d idempotency_key`. |

Then repeat with `ON CONFLICT DO NOTHING` on both inserts and observe the difference:
session 2 does not error; it reports zero rows affected.

> **Verify this one rather than trusting it:** the precise blocking behaviour of
> `ON CONFLICT DO NOTHING` against a *concurrent uncommitted* insert has version-dependent
> subtleties. Run it on your Postgres version and record what you see. The plain-constraint
> behaviour above is the stable foundation; `ON CONFLICT` is the ergonomic wrapper.

### Proof 3 — the aborted transaction

```sql
BEGIN;
INSERT INTO idempotency_key
  (user_id, endpoint, idem_key, request_hash, state)
VALUES (1, '/api/payments', 'proof-2', 'h', 'IN_PROGRESS');   -- fails: 23505
SELECT 1;                                                     -- now watch THIS
```

**WHAT TO LOOK FOR:** the error on the `SELECT 1`.

| What you see | What it means |
|---|---|
| "current transaction is aborted, commands ignored until end of transaction block" | **The fact behind Trap 4**, at the SQL level. Your Java `catch` block cannot escape this. |
| `SELECT 1` succeeds | You are not in an explicit transaction — autocommit wrapped each statement. Add `BEGIN;`. |

Then repeat with a savepoint and see it work:

```sql
BEGIN;
SAVEPOINT s1;
INSERT ... ;                 -- fails
ROLLBACK TO SAVEPOINT s1;
SELECT 1;                    -- now succeeds
```

**That is exactly what `Propagation.NESTED` does under the hood**, and seeing it makes the
propagation mode concrete rather than a word in Topic 54.

### Proof 4 — a real concurrent test, in Java

A loop is not concurrency. This is:

```java
@Test
void concurrentRequestsWithTheSameKeyChargeOnce() throws Exception {
    int n = 32;
    String key = UUID.randomUUID().toString();
    BigDecimal before = walletRepo.balanceOf(USER_ID);

    // A barrier makes them start together. Without it they queue and the race never happens.
    CountDownLatch start = new CountDownLatch(1);
    List<Callable<Integer>> tasks = IntStream.range(0, n)
            .mapToObj(i -> (Callable<Integer>) () -> {
                start.await();
                return postPayment(key, new BigDecimal("40.00")).getStatusCode().value();
            })
            .toList();

    try (var pool = Executors.newVirtualThreadPerTaskExecutor()) {
        List<Future<Integer>> futures = tasks.stream().map(pool::submit).toList();
        start.countDown();
        List<Integer> statuses = new ArrayList<>();
        for (Future<Integer> f : futures) statuses.add(f.get());

        assertThat(walletRepo.balanceOf(USER_ID))
                .isEqualByComparingTo(before.subtract(new BigDecimal("40.00")));
        assertThat(paymentRepo.countByIdempotencyKey(key)).isEqualTo(1);
        assertThat(statuses).allMatch(s -> s == 201 || s == 200 || s == 409);
    }
}
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Balance down by exactly 40; one payment row | Correct. |
| Balance down by a multiple of 40 | The race is live. Check the constraint exists and that the claim is the first statement in the transaction. |
| All 32 return 201 | No suppression at all. The key is not being honoured — check the filter's `shouldNotFilter`. |
| A mix of 201, 409 and 200 | Expected and correct: one winner, some concurrent duplicates rejected as in-progress, some later ones replayed. |
| A 500 among them | Almost certainly Trap 4 — a caught violation inside the transaction. |
| Always exactly one 201 even with the constraint removed | **Your test is not concurrent.** Check the barrier, and use Testcontainers (Topic 61) rather than H2. |

**Run this test once with the constraint dropped and confirm it fails.** A test that passes
for the wrong reason is worse than no test.

---

## Failure drill

**Break it like this:** send two concurrent requests with the same `Idempotency-Key` at a
check-then-act implementation. **Capture:** a double wallet debit and two payment rows.
**The fix proves:** a unique constraint plus a same-transaction record is what makes the
database the arbiter.

### Part A — build the broken version deliberately

**Step 1.** Drop the constraint:

```sql
ALTER TABLE idempotency_key DROP CONSTRAINT uq_idempotency_key;
```

**Step 2.** Replace the claim with a check-then-act, exactly as in the Concurrency trace:

```java
@Transactional
public PaymentResponse pay(IdempotencyContext idem, PaymentRequest req) {
    if (idempotencyRepo.find(idem.userId(), idem.endpoint(), idem.key()).isPresent()) {
        return replay(idem);
    }
    walletService.debit(req.userId(), req.amount());
    Payment payment = paymentRepo.save(Payment.from(req));
    idempotencyRepo.insertPlain(idem, 201, mapper.writeValueAsBytes(payment));
    return PaymentResponse.of(payment);
}
```

**Step 3.** Record the starting balance:

```sql
SELECT user_id, balance FROM wallet WHERE user_id = 1;
```

**Step 4.** Fire two concurrent requests with the same key. Two shells, `&`, so they overlap:

```bash
KEY=drill-116-$(date +%s)
for i in 1 2; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:8080/api/payments \
    -H "Content-Type: application/json" \
    -H "Idempotency-Key: $KEY" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"userId":1,"orderId":"...","amount":"40.00"}' &
done
wait
```

Two `curl`s may not overlap reliably. If they do not, use the Proof 4 harness with 32 threads
and a barrier — **and note that needing 32 threads is itself the lesson**: the window is
narrow, which is why it survives review and testing and appears in production.

**Step 5.** Capture the damage:

```sql
SELECT user_id, balance FROM wallet WHERE user_id = 1;
SELECT id, amount, created_at FROM payment WHERE idempotency_key = 'drill-116-...';
SELECT count(*) FROM idempotency_key WHERE idem_key = 'drill-116-...';
```

### What to capture, before reading on

Write down, on paper:

1. Both HTTP status codes.
2. The wallet balance before and after.
3. The number of `payment` rows.
4. The number of `idempotency_key` rows.
5. Whether **anything** appeared in the error log. (It did not. Write that down — the silence
   is the point.)

### How to read it

| What you see | What it means |
|---|---|
| Two 201s, balance down 80, two payment rows, two key rows | **The full failure.** Both requests passed the check, both acted, and the "idempotency" table faithfully recorded both. |
| Two 201s, balance down 80, two payment rows, **one** key row | You dropped the constraint but a partial index or trigger still deduped the key row. The money is still wrong — which shows the key row is not the thing that matters. |
| One 201, one 409, balance down 40 | The requests did not overlap. Use the barrier harness. |
| Balance down 80 but only one payment row | The payment insert has its own uniqueness — but the wallet was still debited twice, so the effect is **not** protected by it. Partial protection is the most dangerous kind. |

### Part B — fix it and re-run

**Step 1.** Restore the constraint:

```sql
ALTER TABLE idempotency_key
  ADD CONSTRAINT uq_idempotency_key UNIQUE (user_id, endpoint, idem_key);
```

**Step 2.** Replace the check-then-act with the atomic claim as the **first statement** of
the transactional method (Example 2, Design 1).

**Step 3.** Re-run steps 3–5 of Part A with a new key.

| What you see | What it means |
|---|---|
| One 201 and one 409 (or 200 replay); balance down 40; one payment row | **Fixed.** The database arbitrated. |
| One 201 and one 500 | The loser's constraint violation is escaping. Convert `IdempotencyKeyHeld` to a 409 in `@ControllerAdvice`. |
| Both 409 | Both lost, which means neither claimed — you are claiming in a separate transaction that rolled back. Check the boundary. |
| Balance still down 80 | The claim is not in the same transaction as the debit. **Print the transaction name** with `TransactionSynchronizationManager.getCurrentTransactionName()` at both points and compare. |

### Part C — the consumer half

**Step 1.** Publish the same `order.placed` event twice with the same `eventId` (or replay
the partition by resetting the consumer group offset).

**Step 2.** Watch `inventory.reserved` for the order.

| What you see | What it means |
|---|---|
| Reserved incremented once; duplicate counter incremented once | Correct. |
| Reserved incremented twice | The `processed_event` claim is missing, in a separate transaction, or keyed on the offset rather than the producer's event ID. |
| Consumer lag climbing after the duplicate | You are not acknowledging on the duplicate path. It redelivers forever. |
| The notification consumer also skipped | Check the `consumer` column — a single-column PK on `event_id` lets one group suppress another. |

### What the drill proves

Three things, from experience rather than reading:

1. **The failure is silent.** No exception, no log, no alert. Only money.
2. **A row lock on the thing you are modifying does not help**, because the thing you needed
   to lock did not exist yet. Only a uniqueness constraint on the *key* does.
3. **The constraint alone prevents the double charge but produces a 500.** Turning that into
   a useful replay is the remaining design work, and it is why the record stores the
   response.

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` loop is the WRONG way to measure any of this.** It measures
> JIT warm-up, dead-code elimination and ambient noise. Topic 77 (JMH) is where you learn to
> do it properly.

And a sharper version for this topic: **the thing you are measuring is not latency, it is a
count.** The question "does this race" is answered by a counter under concurrent load, not by
a timer. Timing the happy path tells you nothing about correctness, and the instinct to reach
for a stopwatch here is itself a diagnostic error.

### The instrument for each claim

| Claim | Instrument | What invalidates it |
|---|---|---|
| "Duplicates are suppressed" | `orderflow.idempotency.duplicate` counter under injected retries | Measuring at 1x arrival rate only |
| "The effect happens exactly once" | Reconciliation query: payments per idempotency key | Counting key rows instead of effect rows |
| "The record is in the same transaction" | The Part B drill; plus `getCurrentTransactionName()` logged at both points | Reading the code and believing it |
| "Consumers are idempotent" | Replay a partition; compare `inventory.reserved` before and after | Testing with one consumer instance |
| "The constraint exists in production" | `\d idempotency_key`, in production, not in a migration file | A migration that was written but never applied |
| "Idempotency costs us little" | p99 of the guarded endpoints with and without the filter, at matched rates | A microbenchmark of the insert |
| "Retention is safe" | Oldest key age vs the client's documented retry window | Deleting on a schedule nobody derived |

### The three numbers worth graphing permanently

```java
// 1. Duplicates suppressed -- proves the mechanism is live.
Counter.builder("orderflow.idempotency.duplicate")
       .tag("endpoint", endpoint).tag("outcome", outcome).register(registry);

// 2. In-progress conflicts -- a latency signal in disguise.
Counter.builder("orderflow.idempotency.in_progress")
       .tag("endpoint", endpoint).register(registry);

// 3. Consumer duplicates -- proves at-least-once is being absorbed.
Counter.builder("orderflow.consumer.duplicate")
       .tag("consumer", consumerName).register(registry);
```

**Cardinality warning (Topic 118): tag by endpoint and outcome, never by key or user.** One
series per idempotency key takes down Prometheus faster than it helps you — exactly the
mistake Topic 118's drill is about.

| What you see | What it means |
|---|---|
| Duplicate counter at zero, permanently | Either no client retries (unlikely) or the mechanism is inert. Inject a retry and confirm it moves. **A counter you have never seen move is not evidence.** |
| `in_progress` >> `replay` | Clients retry before the original finishes. Compare their retry timeout to your p99 — that is the real problem. |
| Consumer duplicates spiking | A rebalance storm (Topic 113) or an outbox relay republishing (Topic 115). The counter is working; go look upstream. |
| Duplicates rising after a Loom migration | Higher real concurrency exposing a race the thread cap was hiding. **Topic 107 predicts this exactly.** |

### The reconciliation queries — run them on a schedule

```sql
-- (a) Should always be zero: a completed key with no effect.
SELECT count(*) FROM idempotency_key k
 WHERE k.state = 'COMPLETED' AND k.endpoint = '/api/payments'
   AND NOT EXISTS (SELECT 1 FROM payment p WHERE p.idempotency_key_id = k.id);

-- (b) Should always be zero: more than one effect per key.
SELECT k.id, count(p.id) FROM idempotency_key k
 JOIN payment p ON p.idempotency_key_id = k.id
 GROUP BY k.id HAVING count(p.id) > 1;

-- (c) Should always be zero: an effect with no key at all.
SELECT count(*) FROM payment p
 WHERE p.idempotency_key_id IS NULL AND p.created_at > now() - interval '1 day';
```

**These three queries are the honest test of the whole design**, run against production data
rather than a fixture. Alert on any of them being nonzero.

### Cost, measured properly

Idempotency is not free: one extra `INSERT` per guarded request, one extra `UPDATE` to store
the response, and a growing table. Measure it as a p99 delta on the guarded endpoints at
matched arrival rates against the Topic 65 baseline — **not** as a microbenchmark of the
insert, which tells you the insert is fast and nothing about whether the extra round trip
matters at your rate. A delta that grows with load means the extra statements are contending
(the index, or pool pressure — Topic 109); a delta that grows over weeks means your retention
job is not running.
---

## Practice exercises

### 1 — easy: reproduce and fix

1. Run Proof 1 in two `psql` sessions and record the wallet balance.
2. Run Proof 2; record where session 2 blocks and what SQLState it gets. Then Proof 3, and
   record the error on `SELECT 1`.
3. Add the unique constraint and rerun Proof 1's Java equivalent (Proof 4).

**Done when:** you can state, without looking, what a `SELECT` locks when the row does not
exist (nothing), and what a duplicate `INSERT` does while the first transaction is open
(blocks).

### 2 — medium: make `POST /api/orders` idempotent end to end

1. Add the `idempotency_key` table with the scoped unique constraint.
2. Add the filter, the sealed `IdempotencyOutcome`, and the `tryClaim` claim as the first
   statement of the transactional method.
3. Store and replay the response body and status.
4. Reject a key reused with a different body with 422.
5. Write the Proof 4 concurrent test with 32 virtual threads and a barrier.
6. **Drop the constraint and confirm the test fails.** Restore it.
7. Add the duplicate counter and run the Topic 65 harness with 5% injected retries.

**Done when:** the concurrent test passes with the constraint and fails without it, and the
counter moves under load. Step 6 is the one people skip; it is the one that proves the test.

### 3 — hard: idempotency across the whole flow, plus the reconciliation

Make the full `orderflow` place-order path idempotent, end to end:

1. `POST /api/orders` with a client `Idempotency-Key`.
2. The outbox row (Topic 115) written in the same transaction, carrying an `eventId`.
3. The inventory consumer deduping on `(consumer, event_id)`.
4. The notification consumer deduping independently, **with a written decision about the
   non-transactional email side effect** and which failure you chose to accept.
5. The three reconciliation queries, scheduled, alerting on nonzero.
6. A chaos test: `kill -9` the service at three points — between the claim and the wallet
   debit, between the debit and the commit, and between the commit and the Kafka publish —
   and record the state of the wallet, the order, the outbox and the key after each.

**Write a one-page note** answering: for each kill point, what does a client retry with the
same key do, and is that correct? Include the case where the client retries **after** the
retention window has deleted the key.

**Marking rubric:**

| Criterion | Fail | Pass | Strong |
|---|---|---|---|
| Claim atomicity | `SELECT`-then-`INSERT` | `ON CONFLICT DO NOTHING` | Plus a test that fails without the constraint |
| Transaction boundary | Separate transactions | One transaction | Proven with `getCurrentTransactionName()` logging in the drill |
| Replay | 409 only | Original status and body replayed | Plus request-hash mismatch → 422 |
| Consumers | Deduped on offset | Deduped on producer event ID | Per-consumer-group key, with the notification trade-off written down |
| Reconciliation | None | Queries written | Scheduled and alerting |
| Chaos test | Not run | Three kill points | Plus the post-retention-window case, and retention derived rather than picked |

---

## Interview questions

### Q1 — "How do you make a POST endpoint idempotent?"

**MID-LEVEL ANSWER.** "The client sends an `Idempotency-Key` header, we check whether we've
seen it, and if we have, we return the previous response." Correct pattern, and the word
"check" hides the entire bug.

**SENIOR ANSWER.**

> "The client supplies the key, because only the client knows when two requests are the same
> intent — a server-generated key is new on the retry, so it does nothing.
>
> The important part is that the check and the effect have to be one atomic unit. A `SELECT`
> for the key followed by an `INSERT` races: under READ COMMITTED, two concurrent requests
> both see the key absent, because a row that does not exist cannot be locked. Both proceed
> and you charge twice. So I put a unique constraint on `(user_id, endpoint, key)` and claim
> it with a single `INSERT ... ON CONFLICT DO NOTHING`, which returns a row count instead of
> throwing. If I get 1, I own the key; if I get 0, someone else does.
>
> And the claim has to be in the **same transaction** as the effect. If the key commits first
> and we then crash, the retry is suppressed and the order is silently lost — which is worse
> than a duplicate, because nobody detects it. If the effect commits first, we are back to
> double charges. One transaction, all of it.
>
> Then the record stores the response status and body, so a retry replays the original answer
> rather than a bare 409 — a client that timed out needs to know what happened. And I store a
> hash of the request body, so if a client reuses a key with different content I return 422
> instead of replaying the wrong response."

**WHAT SEPARATES THEM.** Four things, and an interviewer is listening for all of them:

1. **Naming the race explicitly**, including *why* a `SELECT` cannot lock a nonexistent row.
2. **The constraint as the arbiter**, with `ON CONFLICT DO NOTHING` rather than a caught
   exception.
3. **The same-transaction requirement, with both failure directions named.** This is the
   discriminating one — most candidates stop at the constraint.
4. **Replay rather than suppression**, because the client needs an answer.

**FOLLOW-UP: "What if the retry arrives while the original is still running?"**

> "The second `INSERT` blocks on the first transaction and then fails, so I return 409 with
> `Retry-After`. I do not want to make the client wait on a database lock for an unbounded
> time. And it is worth watching that rate — if in-progress conflicts dominate replays, the
> client's retry timeout is shorter than our p99, which is a latency problem showing up as an
> idempotency metric."

### Q2 — "Why can't you just catch the duplicate-key exception?"

**MID-LEVEL ANSWER.** "You can — catch `DataIntegrityViolationException` and return the
existing record." This is the most common wrong answer, and it looks right.

**SENIOR ANSWER.**

> "Not inside the same transaction, and this catches people who otherwise have the design
> right.
>
> In Postgres, any statement error aborts the transaction. Every subsequent statement fails
> with 'current transaction is aborted'. On top of that, Hibernate's session is unusable
> after a `ConstraintViolationException` escapes flush, and Spring marks the transaction
> rollback-only — so even if my `catch` block returns normally, the commit at the proxy
> boundary throws `UnexpectedRollbackException`. The symptom is an intermittent 500 whose
> stack trace names `TransactionAspectSupport` rather than my code, so nobody connects it to
> idempotency.
>
> There are three ways out. Best is not to throw at all: `ON CONFLICT DO NOTHING` returns a
> count. Second, let it abort and handle the replay outside, in a fresh transaction — which
> is what a `@ControllerAdvice` mapping does naturally. Third, a savepoint —
> `Propagation.NESTED` uses JDBC savepoints — if I genuinely must continue the same
> transaction, at the cost of extra round trips.
>
> One practical note: this works differently on H2, so a test suite on H2 will pass and
> production will fail. It is one of the better arguments for Testcontainers."

**WHAT SEPARATES THEM.** Knowing that the abort is a database-level fact rather than a
framework quirk; knowing the three escape routes and ranking them; and the H2-versus-Postgres
observation, which shows the person has actually shipped this.

### Q3 — "Your consumer processes a Kafka message twice. Walk me through the fix."

**MID-LEVEL ANSWER.** "Store the message ID and check it before processing." Same
check-then-act, same race, now distributed across consumer instances.

**SENIOR ANSWER.**

> "First, accept that duplicates are structural, not a bug to fix upstream. Kafka redelivers
> on rebalance if the offset was not committed, and if we publish through an outbox the relay
> is at-least-once by construction — it publishes and then marks published, and a crash
> between those republishes. So the consumer has to be idempotent; there is nowhere else to
> put it.
>
> A `processed_event` table with a primary key on `(consumer, event_id)`. Claim it with
> `ON CONFLICT DO NOTHING` as the first statement, and do the effect in the same transaction.
> If the claim returns 0, increment a duplicate counter and acknowledge — acknowledging
> matters, because a duplicate *is* handled and not acking makes it redeliver forever.
>
> Two details people get wrong. The ID must be the **producer's** event ID, not the Kafka
> offset — an offset changes if the topic is recreated or if the same logical event is
> republished, and then dedupe silently stops working. The outbox row's ID is the natural
> choice: generated once, in the same transaction as the order. And the key must include the
> consumer group, because the same event goes to inventory and to notifications and they must
> each process it once; a single-column key on `event_id` lets whichever arrives first
> suppress the other.
>
> The one case the transaction cannot cover is a non-database effect — sending an email.
> There I have to choose: record then send, and risk losing an email on a crash; or send then
> record, and risk sending twice. For a notification I would take the lost email and write
> the decision down. If that is not acceptable, the notification needs its own outbox, which
> is the same problem one layer down."

**WHAT SEPARATES THEM.** Treating duplicates as structural; producer ID over offset;
per-consumer-group scoping; acknowledging the duplicate; and — the strongest signal — naming
the non-transactional side effect as an explicit, documented trade-off rather than pretending
the transaction covers it.

### Q4 — "We store the idempotency key in Redis with SET NX. Is that enough?"

**MID-LEVEL ANSWER.** "Yes, `SET NX` is atomic." True and insufficient, and the candidate has
answered a different question.

**SENIOR ANSWER.**

> "`SET NX` is genuinely atomic, so it fixes the check-then-act race. But it is a different
> system from the database holding the effect, so it is a dual write — the same problem the
> outbox pattern exists to solve.
>
> Concretely: we `SET NX`, then start the wallet debit, then the process dies. Redis has the
> key, Postgres has nothing. The client's retry is suppressed and the payment is silently
> lost. Reverse the order and the crash window produces a double charge instead. Either way
> there is a window, and it does not close by being careful — it closes only by making the
> key and the effect commit together.
>
> So the authority has to be a row in the same database as the effect, in the same
> transaction. Redis is fine as an optimisation in front of that — a fast pre-check that
> avoids a Postgres round trip for obvious duplicates — but it can never be the decision, and
> a stale or missing Redis entry has to be harmless.
>
> The exception is if the effect itself is in Redis. Then Redis is the database and the
> reasoning is the same, just with a different engine."

**WHAT SEPARATES THEM.** Recognising it as the dual-write problem rather than an atomicity
question; naming both crash orderings and their distinct symptoms; and demoting Redis to an
optimisation with a stated safety property, rather than either accepting or rejecting it
wholesale.

### Q5 — "How long do you keep idempotency keys, and what happens when you delete one?"

**MID-LEVEL ANSWER.** "We clean up old ones on a schedule, maybe 24 hours." A number with no
derivation, and no thought about the consequence.

**SENIOR ANSWER.**

> "The retention window is a correctness parameter, not housekeeping, and I would derive it
> rather than pick it.
>
> Deleting a key reopens the window: if a client retries with a key we have forgotten, we
> treat it as new and do the work again. So the window has to exceed the longest plausible
> gap between a client's original request and its last retry. That comes from the client's
> documented retry policy — how many attempts, what backoff, what ceiling — plus a margin for
> a client that queues requests through an outage. For a mobile client that retries on next
> app launch, that could be days.
>
> The two tables get different windows. `idempotency_key` is bounded by client retry
> behaviour, so days. `processed_event` is bounded by how far back a Kafka topic can be
> replayed or a consumer group reset, so longer — and I would size it against the topic's
> retention rather than guess.
>
> Operationally: index `created_at`, delete in batches so the transaction does not hold locks
> for minutes, and monitor the oldest surviving row so a failed cleanup job is visible before
> the table becomes a problem. And I would put the derivation in a comment next to the
> interval, because the next person will otherwise 'tidy it up' to something shorter."

**WHAT SEPARATES THEM.** Treating retention as correctness; deriving the number from the
client's behaviour; giving the two tables different windows for different reasons; and the
operational detail about batching and monitoring, which is what someone who has run this at
size says.

---

## Mental model checkpoint

Answer without scrolling up.

1. **Why does `SELECT`-then-`INSERT` race, precisely?**
   A `SELECT` for a row that does not exist takes no lock — there is nothing to lock. Under
   READ COMMITTED two concurrent readers both see absence, both proceed.

2. **What does a `UNIQUE` constraint give you that a row lock does not?**
   Mutual exclusion on a key that has no row yet. Exactly one inserter wins; concurrent
   losers block until the winner's fate is decided, then fail with `23505` — or succeed if
   the winner rolled back.

3. **Where must the idempotency record live relative to the effect, and what are both failure
   modes?**
   Same transaction. Record first → the retry is suppressed and the work is silently lost.
   Effect first → the retry does the work again.

4. **Why can't you catch `DataIntegrityViolationException` and continue?**
   The statement error aborted the Postgres transaction and Spring marked it rollback-only.
   Subsequent statements fail; the commit throws `UnexpectedRollbackException`. Use
   `ON CONFLICT DO NOTHING`, a fresh transaction, or a savepoint.

5. **Why must the client generate the key?**
   Only the client knows when two requests are the same intent. A server-generated key is new
   on the retry. A body-hash key wrongly merges two legitimately identical requests.

6. **What identifier does a consumer dedupe on, and what must the key include?**
   The producer's event ID — not the Kafka offset — and the key must include the consumer
   group, so one group cannot suppress another.

7. **What should a duplicate return, and why is 409 usually not enough?**
   The original response, same status and body. A client that timed out does not know whether
   the original succeeded, so a bare 409 leaves it unable to proceed.

---

## Quick reference card

### The rule

```
Idempotency = uniqueness check AND effect in ONE atomic unit.
SELECT then INSERT   -> two operations -> races -> double charge.
UNIQUE constraint    -> the database arbitrates -> exactly one winner.
Same transaction     -> or you trade a double charge for silent data loss.
```

### The claim, in one statement

```sql
INSERT INTO idempotency_key (user_id, endpoint, idem_key, request_hash, state)
VALUES (:u, :e, :k, :h, 'IN_PROGRESS')
ON CONFLICT (user_id, endpoint, idem_key) DO NOTHING;
-- returns 1 = you own it, 0 = someone else does. Never throws. Never aborts.
```

### The two tables

```sql
UNIQUE (user_id, endpoint, idem_key)     -- API: scoped per user AND endpoint
PRIMARY KEY (consumer, event_id)         -- consumers: scoped per consumer group
```

### Outcomes to return

```
claimed         -> do the work, store status+body, commit it all together
COMPLETED       -> replay the stored status and body ("Idempotent-Replay: true")
IN_PROGRESS     -> 409 + Retry-After
hash mismatch   -> 422 (key reused for a different body)
missing header  -> 400 on a guarded endpoint. Required, not optional.
```

### SQLState and exception chain

```
23505  unique_violation
  -> org.postgresql.util.PSQLException
  -> org.hibernate.exception.ConstraintViolationException
  -> org.springframework.dao.DuplicateKeyException (extends DataIntegrityViolationException)
The SQLState lives on the INNERMOST exception.
```

### Things that do not work

```
ConcurrentHashMap dedupe      -> lost on restart, per-instance
Redis SET NX alone            -> dual write; a crash window remains
Server-generated / body-hash  -> inert on retry; or merges distinct requests
catch(DataIntegrityViolation) -> transaction already aborted
REQUIRES_NEW for the record   -> dual write + a second connection (Topic 109)
Dedupe on the Kafka offset    -> breaks on republish or topic recreation
```

### Gotchas checklist

```
[ ] Unique constraint EXISTS in production, verified with \d, not with a migration file
[ ] The claim is the FIRST statement of the transactional method
[ ] The claim, the effect and the stored response are in ONE transaction
[ ] The response status and body are stored; the request hash is stored and checked
[ ] Consumers ack on the duplicate path; the consumer key includes the consumer group
[ ] Retention derived from the client retry policy, with the derivation written down
[ ] Duplicate counters tagged by endpoint/outcome, NEVER by key or user (Topic 118)
[ ] The concurrent test FAILS when the constraint is dropped
```

---

## When would I use this at work?

**1. The first PR on any endpoint that moves money or creates an external obligation.**

`POST /payments`, `POST /orders`, `POST /refunds`, anything calling a payment gateway. The
review question is never "is this idempotent" — it is **"where exactly is the uniqueness
check, and is it in the same transaction as the effect?"** Thirty seconds, and it catches the
bug that costs the most.

**2. Immediately after adopting the outbox pattern.**

Topic 115's relay is at-least-once by construction. Shipping the outbox without idempotent
consumers converts a rare lost-event bug into a routine duplicate-processing bug — usually
worse, because it happens constantly. **They are one piece of work, not two.**

**3. When investigating a "double charge" or "duplicate order" report.**

You now have a diagnostic order. Does the endpoint require the header? Is the constraint
actually present in production? Is the claim atomic or a `SELECT`-then-`INSERT`? Is it in the
same transaction as the effect? Four questions, each with a command, and one of them is almost
always the answer — in ten minutes rather than a day.

---

## Connected topics

**Prerequisites:**

- **92 — Concurrent collections:** **this document is Topic 92's check-then-act race at the
  API layer**, with money attached. Read that drill first if the race is not yet instinctive.
- **52 — Hibernate locking:** why a row lock on the entity you are about to modify does not
  protect you when the row you needed to lock does not exist.
- **54 — `@Transactional` semantics:** proxy boundaries, propagation, and why
  `REQUIRES_NEW` for the idempotency record is the wrong instinct.
- **55 — Isolation and the connection pool:** READ COMMITTED is why both readers see absence,
  and a transaction holds its connection for its whole life.
- **113–114 — Kafka consumer groups and delivery semantics:** where the duplicates come from
  and why at-least-once is not negotiable.
- **115 — The outbox pattern:** the relay is at-least-once by construction. **This topic is
  the other half of that one**; shipping the outbox without it is incomplete.

**Also relevant:**

- **28 — Sealed types:** `IdempotencyOutcome` as a sealed interface over records gives
  compile-checked exhaustiveness when a fifth outcome is added.
- **46 — `ProblemDetail`:** the 409, 422 and 400 bodies are an API surface with the same
  compatibility obligations as the success path.
- **61 — Testcontainers:** the abort-on-constraint-violation behaviour differs on H2 — one of
  the clearest cases for testing against the real engine.
- **65 — The load baseline:** the harness for the duplicate-rate measurements.
- **101 / 107 — Virtual threads and the Loom decision:** higher real concurrency surfaces
  races that a bounded thread pool was hiding. Count duplicates at every arrival rate.
- **109 — HikariCP:** why `REQUIRES_NEW` for the idempotency record is a pool deadlock as
  well as a correctness bug.
- **118 — Metrics:** the duplicate counters, and the cardinality rule that forbids tagging
  by key.

**This unlocks:**

- **117 — Sagas and compensations:** every saga step must be idempotent, because every step
  can be retried. This topic is the precondition for that one.
- **119 — Tracing:** correlating a retry with its original request across services requires
  both the trace context and the idempotency key on the span.
- **124 — The readiness gate:** a service whose idempotency constraint is missing in production
  is not ready, whatever its health endpoint reports. A startup assertion is cheap insurance.

---

*Java baseline 21, Spring Boot 4.1, Postgres. Driver, Hibernate and Spring versions come from
the Boot BOM; do not pin them by hand. SQLState `23505` for a unique violation is stable across
Postgres versions and is the value to branch on, but confirm it in your own two-session
experiment rather than trusting this document. The one behaviour explicitly flagged as
version-sensitive is `INSERT ... ON CONFLICT DO NOTHING` against a concurrent **uncommitted**
duplicate — verify it on your version; the plain unique-constraint behaviour it wraps is the
stable foundation. The structural fact worth committing to memory is the mechanical statement:
an idempotency key only works if the uniqueness check and the effect are in the same atomic
unit, and a `SELECT` then an `INSERT` is two operations.*
