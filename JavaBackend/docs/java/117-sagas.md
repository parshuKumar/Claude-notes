# 117 — Sagas and Compensations

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21. Spring, Kafka client and Hibernate versions come from the Spring Boot BOM. Do not pin them by hand and do not quote a version number from this document.
## Project spine: decompose `orderflow`'s place-order flow into an orchestrated saga — reserve inventory, debit wallet, authorise payment, confirm order — with a compensation for each step and a deadline on every one.

---

## Before anything else — what is and is not in this document

**I have no JVM, no Postgres and no Kafka. Nothing in this document is captured tool
output.**

You will **not** find here:

- a captured `psql` session, a real `EXPLAIN` plan, or a real row count,
- a real stack trace, a real thread dump, or a real Kafka consumer-lag figure,
- a saga duration, a compensation rate, or any number presented as measured,
- "3% of sagas ended in `COMPENSATION_FAILED`" or anything shaped like a result.

You get instead, everywhere:

- **the exact SQL, code, command or config**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see → what it means"** table covering plausible outcomes, including the ones
  that mean your experiment did not reproduce the failure rather than that your saga is
  correct.

### The labelled exceptions

Twice below I show the **shape** of a state-table row and of a log line. Every value is a
placeholder — `<saga-id>`, `<step>`, `<n>`, `<ts>`. Each carries the inline label:

> *illustration of the format, not captured output*

As in Topic 116, I state the Postgres SQLState for a unique violation as **`23505`**,
because it is stable and it is the string you branch on. Confirm it yourself.

### Spec-level facts I state plainly, each with a confirming command

1. **A saga's intermediate states are visible to other transactions.** Confirm: Proof 1 —
   query the order table from a second session while a saga is mid-flight.
2. **`SELECT ... FOR UPDATE SKIP LOCKED` lets N workers claim disjoint rows without
   blocking each other.** Confirm: Proof 2, two `psql` sessions.
3. **Each saga step commits independently; there is no distributed transaction and no
   rollback across steps.** Confirm: Proof 3 — kill the orchestrator mid-saga and observe
   that completed steps stay committed.
4. **A compensation is an ordinary operation that can fail like any other.** Confirm: the
   failure drill, which makes one fail deliberately.

### THE RULE

> **If your own output disagrees with anything here, YOUR OUTPUT IS THE TRUTH.** Especially
> for anything about your message broker's redelivery behaviour, which is configuration, not
> physics.

### What this document assumes

**Topic 116 is a hard prerequisite.** Every saga step is retried, and every compensation is
retried. If your steps are not idempotent, a saga turns one duplicate into many. Do not read
this document as a way to avoid doing Topic 116; it is the layer on top.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **A saga is a sequence of local transactions, each with a compensating transaction that
> semantically undoes it.**
>
> **It is NOT atomic. Intermediate states are observable by everyone else in the system,
> and a compensation can itself fail.**

Three consequences you must hold throughout:

1. **"Undo" is a business decision, not a technical one.** You cannot un-charge a card; you
   issue a refund. You cannot un-send an email; you send a correction. The compensation is a
   *new* operation with its own semantics, its own failure modes, and its own customer
   impact.
2. **Someone will observe the middle.** An order with inventory reserved and no payment taken
   is visible to the catalogue, the customer, a report and the warehouse. **What that state
   means to each of them is a product question you must answer before writing the code.**
3. **The compensation path is real production code and the least-tested code you own.** It
   runs rarely, under stress, on the unhappy path — exactly the profile of broken code.

---

## The bridge from what you know

### You know the pattern. Here is what is different in Java, and it is mostly the plumbing.

You have designed sagas. You know orchestration versus choreography, you know compensations,
you know why two-phase commit is not on the table across service boundaries. None of that is
being re-taught.

What is different in Java and Spring is **where the state lives, what enforces the
transaction boundary, and which primitive claims work safely** — and there are four specific
things worth naming.

**1. Each step's effect and each step's state update must be in the same local
transaction.** This is Topic 116's rule again, and it is the single most common structural
bug in a hand-rolled saga:

```java
@Transactional
public void executeStep(UUID sagaId) {
    inventoryService.reserve(...);        // the effect
    sagaRepo.markStepCompleted(sagaId, "RESERVE_INVENTORY");   // the state
    // Both commit, or neither does. If these were separate transactions, a crash
    // between them leaves you either re-running a completed step or skipping one.
}
```

**2. `@Transactional` is proxy-based (Topics 40, 54), so "the same transaction" is a fact
about the call path.** A saga orchestrator that calls its own step methods internally gets no
transaction at all on those calls. This bites people who put the whole saga in one class.

**3. Your work-claiming primitive is `SELECT ... FOR UPDATE SKIP LOCKED`.** Multiple pods run
the same orchestrator. Without `SKIP LOCKED` they either serialise on the same row or process
it twice. You met this in Topic 115's outbox relay; it is the same tool for the same reason.

**4. There is no `try`/`catch` that unwinds.** Your Node instinct might be a `try` block whose
`catch` calls the undos. **That works only inside one process, for the life of one process.** A
saga must survive a `kill -9` between any two steps, so the "what have I done so far" list is a
set of rows, not a stack frame.

### The instinct that will hurt you: treating a saga as a distributed transaction

The phrase "we use sagas instead of distributed transactions" invites a false equivalence.
They are not two implementations of the same guarantee.

| | Distributed transaction (2PC) | Saga |
|---|---|---|
| Atomicity | Yes | **No** |
| Isolation | Yes | **No** — intermediate states are visible |
| Intermediate states observable | No | **Yes, by everyone** |
| Failure of one participant | Everything rolls back | Completed steps stay committed; you compensate |
| Availability during a participant outage | **Blocks** — the coordinator holds locks | Continues; the saga waits or compensates |
| "Undo" | Free, by the engine | **Code you write, that can fail** |

**Sagas trade atomicity and isolation for availability.** That is the whole deal, and if the
business cannot tolerate an observable intermediate state, a saga is the wrong pattern and
you need to redraw your service boundaries so the operation is one local transaction.

### The one genuinely new idea: the pivot step

This is the piece most people have not internalised, and it is what separates a designed
saga from a hopeful one. Steps come in three kinds:

| Kind | Property | Example in `orderflow` |
|---|---|---|
| **Compensatable** | Can be semantically undone | Reserve inventory (release it); debit wallet (credit it back) |
| **Pivot** | The point of no return. Once it succeeds, the saga **must** go forward. | Authorise payment with the external gateway |
| **Retriable** | Cannot fail permanently; retry until it succeeds | Mark the order confirmed; send the confirmation email |

**The design rule: all compensatable steps first, then the pivot, then only retriable steps.**
If a step after the pivot can fail permanently the saga is designed wrong — there is nowhere to
go but forward.

**Verdict: STRONG ANALOGUE for the pattern; you know it. NO ANALOGUE for the durable-state
plumbing, and a specifically misleading instinct that a `try`/`catch` can express a saga.**

---

## What is this?

Two ways to run the same sequence. The technical difference is small; the **observability**
difference is large and is usually what decides it.

### Choreography

Each service listens for events and emits its own. There is no coordinator.

```
order-service      -> OrderCreated
inventory-service  -> (on OrderCreated) reserve -> InventoryReserved
wallet-service     -> (on InventoryReserved) debit -> WalletDebited
payment-service    -> (on WalletDebited) authorise -> PaymentAuthorised
order-service      -> (on PaymentAuthorised) confirm
```

Compensations are also events: `PaymentFailed` triggers `wallet-service` to credit back,
which emits `WalletCredited`, which triggers `inventory-service` to release.

### Orchestration

One component owns the sequence and calls each participant.

```
OrderSagaOrchestrator:
   1. reserve inventory   -> on failure: fail the saga (nothing to compensate)
   2. debit wallet        -> on failure: compensate 1
   3. authorise payment   -> on failure: compensate 2, then 1        [PIVOT]
   4. confirm order       -> retriable; must eventually succeed
```

### The comparison that matters

| | Choreography | Orchestration |
|---|---|---|
| Coupling | Loose — services know events, not each other | The orchestrator knows every participant |
| Where the flow is written down | **Nowhere.** It is emergent from N subscriptions. | One class, one state machine |
| Adding a step | Add a subscriber. Easy. | Change the orchestrator. Slightly harder. |
| **Answering "where is order 4417 right now?"** | **Join logs across N services by correlation ID and hope** | **`SELECT * FROM saga_instance WHERE order_id = 4417`** |
| Answering "how many sagas are stuck?" | Effectively unanswerable without a tracing system | One query |
| Cyclic dependency risk | Real — A's event triggers B which triggers A | Structurally impossible |
| Testing the whole flow | Needs every service, or a lot of mocked brokers | Unit-testable against stubbed participants |
| Single point of failure | None | The orchestrator — mitigated by making it stateless over a state table |

**The mechanical statement about observability, which is the honest deciding factor:**

> **Choreography's flow exists only as the sum of its subscriptions, so its current state is
> not a queryable fact. Orchestration keeps the flow in one place and the state in a table,
> so "where is this saga and why is it stuck" is a `SELECT`.**

**For anything touching money, orchestrate.** During an incident someone will ask "what is
the state of this order and what happens next", and with choreography the honest answer is
"let me correlate some logs", which is not an answer at 2am. That, not coupling theory, is
why `orderflow` orchestrates.

### `[BOOT 3.x DELTA]`

Nothing structural changes on 3.x. Two notes:

- The pattern here is hand-rolled deliberately: a state table, a scheduled claimer with
  `SKIP LOCKED`, and the outbox from Topic 115. Frameworks exist (Axon, Eventuate, Temporal,
  Spring State Machine with a persister). **Learn the hand-rolled version first** so you can
  evaluate what a framework is doing for you — Topic 126's build/buy question — and so you
  can debug it when it stalls.
- Spring's `@Scheduled` and `TaskScheduler` behave the same on both lines; on Boot 3.2+ and
  on 4.x you can put the scheduler on virtual threads (`spring.threads.virtual.enabled`,
  Topic 101), which matters if steps block on network calls.

---

## Why does it matter?

**1. It is where correctness goes to die quietly.** A saga that gets the happy path right and
the compensation path wrong looks perfect in every test and every demo. The failure appears
during a downstream outage — the exact moment you can least afford a second bug — and it
manifests as inconsistent business state rather than an exception.

**2. The intermediate states are a product surface.** "Order 4417 is `PENDING` with inventory
reserved" is a state a customer sees, a warehouse acts on, and a report counts. **If nobody has
decided what it means, you have shipped a product question as a bug** — the part engineers most
often skip and product most often discovers in production.

**3. It is the assembly point for six other topics.** A saga needs idempotency (116), an
outbox (115), retries with backoff and a circuit breaker (111), Kafka semantics (113–114),
correct transaction boundaries (54–55), metrics and tracing (118–119). It is where you find out
which of those you only half-implemented.

---

## Machine-level reality

### 1. Where the saga's state lives

A saga is durable state plus a loop. The state is rows.

```sql
CREATE TABLE saga_instance (
    id              uuid        PRIMARY KEY,
    saga_type       text        NOT NULL,          -- 'PLACE_ORDER'
    order_id        bigint      NOT NULL,
    state           text        NOT NULL,          -- see the state machine below
    current_step    text,
    attempt         int         NOT NULL DEFAULT 0,
    deadline_at     timestamptz NOT NULL,          -- NEVER nullable. See Trap 2.
    last_error      text,
    payload         jsonb       NOT NULL,          -- the inputs each step needs
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_saga_claimable ON saga_instance (state, deadline_at)
    WHERE state IN ('RUNNING', 'COMPENSATING');

CREATE TABLE saga_step (
    saga_id       uuid        NOT NULL REFERENCES saga_instance(id),
    step          text        NOT NULL,
    state         text        NOT NULL,            -- COMPLETED | COMPENSATED | FAILED
    attempt       int         NOT NULL DEFAULT 0,
    completed_at  timestamptz,
    compensated_at timestamptz,
    PRIMARY KEY (saga_id, step)
);
```

Three decisions in that DDL:

| Decision | Why |
|---|---|
| `deadline_at` is `NOT NULL` | A step with no deadline can hang forever (Trap 2). Making the column nullable makes forgetting it possible. |
| `saga_step` has a composite primary key | It is Topic 116's `processed_event` table in disguise: `(saga_id, step)` uniqueness means a step cannot be recorded twice. |
| The partial index covers only in-flight states | The claimer query only ever looks at `RUNNING`/`COMPENSATING`. A full index over a table with millions of `COMPLETED` sagas is wasted. |

**The state machine:**

```
        STARTED
           |
           v
        RUNNING  --step fails-->  COMPENSATING
           |                           |
     all steps ok               all compensations ok
           |                           |
           v                           v
        COMPLETED                  COMPENSATED

   COMPENSATING --a compensation fails--> COMPENSATION_FAILED   [human required]
   RUNNING      --deadline passed-------> COMPENSATING (or COMPENSATION_FAILED)
```

**`COMPENSATION_FAILED` is a real, first-class state and it must page someone.** A saga that
cannot undo itself is inconsistent business state, and designing it away — retrying forever, or
marking it `COMPENSATED` optimistically — is how a reconciliation nightmare starts.

### 2. The transaction boundary of one step

This is the rule that makes the whole thing resumable:

> **Each step's effect and its `saga_step` row commit in ONE local transaction.**

```java
@Transactional
void runReserveInventory(SagaInstance saga) {
    // (a) claim the step -- Topic 116's idempotency, at the saga layer
    int claimed = sagaStepRepo.tryClaim(saga.id(), "RESERVE_INVENTORY");
    if (claimed == 0) return;                  // already done; a retry arrived

    // (b) the effect
    inventoryService.reserve(saga.orderId(), saga.lines());

    // (c) advance the saga
    sagaRepo.advance(saga.id(), "DEBIT_WALLET", Instant.now().plus(STEP_TIMEOUT));
    // ONE commit covers (a), (b) and (c).
}
```

**What each of the three failure windows would cost if you split them:**

| Split | Crash consequence |
|---|---|
| (a) committed separately before (b) | Step marked done, effect never happened. **Silent skip.** The saga proceeds on a false premise. |
| (b) committed separately before (a) | Effect happened, not recorded. **The retry does it again** — and only Topic 116's idempotency inside `reserve` saves you. |
| (c) separate from (a)+(b) | Step done, saga still pointing at it. The retry re-enters the step, `tryClaim` returns 0, and it advances. **Actually safe** — because (a) made the step idempotent. |

The third row is worth pausing on: **the step claim is what makes an imperfect boundary
survivable.** That is why idempotency is a prerequisite rather than a nicety.

### 3. Claiming work safely: `SELECT ... FOR UPDATE SKIP LOCKED`

Several orchestrator pods run at once. Each must pick up sagas nobody else is working on.

```sql
UPDATE saga_instance
   SET state = state, updated_at = now(), attempt = attempt + 1
 WHERE id IN (
       SELECT id FROM saga_instance
        WHERE state IN ('RUNNING', 'COMPENSATING')
          AND deadline_at <= now()
        ORDER BY deadline_at
        FOR UPDATE SKIP LOCKED
        LIMIT 20
 )
RETURNING *;
```

**What each clause is doing:**

| Clause | Purpose |
|---|---|
| `FOR UPDATE` | Row locks, so another pod cannot take the same saga |
| **`SKIP LOCKED`** | **Do not wait on rows someone else locked — skip them.** Without this, pods serialise on the head of the queue and your throughput is one pod's worth. |
| `LIMIT 20` | A batch. Bounded work per tick; also bounds transaction length (Topic 55). |
| `deadline_at <= now()` | The claimer is *also* the timeout reaper. One mechanism, not two. |
| `ORDER BY deadline_at` | Oldest first, so nothing starves. |

**`SKIP LOCKED` is the same primitive as Topic 115's outbox relay** — recognising these as one
pattern, a durable work queue in Postgres, is worth more than either topic alone.

### 4. Why compensations are harder than steps

A compensation is not a rollback. Four properties it must have, each of which is a design
obligation:

| Property | Why | What breaks without it |
|---|---|---|
| **Idempotent** | It will be retried | Double refund |
| **Semantically valid** | You cannot un-charge; you refund | A "reversal" that leaves an inconsistent ledger |
| **Ordered in reverse** | Undo step 3, then 2, then 1 | Releasing inventory before crediting the wallet can expose a state where the customer is charged for nothing |
| **Able to fail visibly** | It is a real operation | Silent inconsistency; nobody knows |

**And the asymmetry that catches people:** a step can *fail* and that is fine — you
compensate. **A compensation failing has no fallback.** There is no compensation for a
compensation. All you can do is retry with backoff, and then escalate to a human. That is
why `COMPENSATION_FAILED` exists as a terminal state and why it pages.

### 5. The isolation you gave up, and the countermeasures

A saga has no isolation. Other transactions see intermediate states. Three anomalies and
their standard countermeasures, stated in the vocabulary an interviewer will use:

| Anomaly | In `orderflow` | Countermeasure |
|---|---|---|
| **Lost update** | The customer cancels while the saga is mid-flight; the cancel is overwritten by the saga's next step | **Semantic lock**: `PENDING` is a lock. Reject cancellation, or make the saga check state before each write. |
| **Dirty read** | A report counts a `PENDING` order as revenue; the saga then compensates | **State-aware reads**: reports filter on terminal states only. This is a contract, and it must be documented. |
| **Fuzzy / non-repeatable read** | Inventory read at step 1 differs by step 3 | **Commutative updates**: `reserved = reserved + n` rather than `reserved = <value read earlier>` |

**The semantic lock is the most useful of these and the least implemented.** An `Order` in
`PENDING` announces "a saga owns me", and every other code path touching an order must respect
that — otherwise your saga's guarantees are fiction. **That is a codebase-wide invariant, not a
saga-module concern**, and it is worth an explicit test.

---

## Example 1 — minimal

The smallest saga that shows all three concepts: local transactions, reverse-order
compensation, and a pivot.

### 1a — the step definition

```java
public interface SagaStep {
    String name();
    void execute(SagaContext ctx);       // the forward action
    void compensate(SagaContext ctx);    // the semantic undo
    default boolean isPivot() { return false; }
}
```

```java
@Component
class ReserveInventoryStep implements SagaStep {

    private final InventoryService inventory;

    public String name() { return "RESERVE_INVENTORY"; }

    public void execute(SagaContext ctx) {
        inventory.reserve(ctx.orderId(), ctx.lines());
    }

    public void compensate(SagaContext ctx) {
        // NOT a rollback: a new operation with its own semantics.
        inventory.release(ctx.orderId(), ctx.lines());
    }
}

@Component
class AuthorisePaymentStep implements SagaStep {

    public String name() { return "AUTHORISE_PAYMENT"; }
    public boolean isPivot() { return true; }        // <-- after this, forward only

    public void execute(SagaContext ctx) {
        gateway.authorise(ctx.orderId(), ctx.amount(), ctx.idempotencyKey());
    }

    public void compensate(SagaContext ctx) {
        throw new UnsupportedOperationException(
                "AUTHORISE_PAYMENT is the pivot; it is never compensated");
    }
}
```

**The `UnsupportedOperationException` is deliberate** — an executable statement that this step
must never be compensated. If it is ever thrown, the step ordering is wrong, and you want to
know loudly in a test rather than quietly in production.

### 1b — the orchestrator loop

```java
void advance(SagaInstance saga) {
    List<SagaStep> steps = definition.stepsFor(saga.sagaType());

    if (saga.state() == RUNNING) {
        SagaStep step = steps.get(saga.currentIndex());
        try {
            runStepTransactionally(saga, step);            // effect + state, one commit
        } catch (RetryableException e) {
            scheduleRetry(saga, e);                        // backoff; deadline unchanged
        } catch (Exception e) {
            if (step.isPivot() || pivotAlreadyPassed(saga)) {
                // Past the pivot there is nowhere to go but forward.
                escalate(saga, "failure after pivot", e);
            } else {
                beginCompensation(saga, e);
            }
        }
    } else if (saga.state() == COMPENSATING) {
        // REVERSE order, completed steps only.
        for (SagaStep step : reversedCompletedSteps(saga, steps)) {
            try {
                compensateTransactionally(saga, step);
            } catch (Exception e) {
                markCompensationFailed(saga, step, e);     // terminal. Pages a human.
                return;
            }
        }
        sagaRepo.markState(saga.id(), COMPENSATED);
    }
}
```

Three things this tiny loop already gets right, and that hand-rolled sagas routinely get
wrong:

1. **Compensation iterates only *completed* steps, in reverse.** Compensating a step that
   never ran is a bug that produces phantom refunds.
2. **A failure past the pivot escalates rather than compensating.** There is no valid undo.
3. **A failed compensation is terminal and visible.** It does not loop forever and it does
   not get swallowed.

### 1c — what makes it resumable

Nothing in `advance` assumes it is the same process that ran the previous step. The saga is
loaded from a row, one step is attempted, and the row is updated. **Kill the process at any
point and the next claimer picks it up from the row.** That property is the entire reason the
state is in Postgres rather than in a field.

---

## Example 2 — production scenario (on the project spine)

### The flow and its failure semantics, decided before any code

| # | Step | Kind | Compensation | If the compensation fails |
|---|---|---|---|---|
| 1 | Reserve inventory | Compensatable | Release the reservation | Retry; then escalate — stock is held against nothing |
| 2 | Debit wallet | Compensatable | Credit the wallet back | Retry; then escalate — **customer is out of money** |
| 3 | Authorise payment | **PIVOT** | — never | n/a |
| 4 | Confirm order | Retriable | — never | Retry forever; alert on age |
| 5 | Publish `order.confirmed` | Retriable, via the outbox | — never | The relay retries (Topic 115) |

**Why step 3 is the pivot:** once the external gateway has authorised, the money is
committed at the network. You can refund, but a refund is a *new* business transaction with
its own settlement timeline and its own customer communication — it is not an undo, and
modelling it as one produces ledgers that do not reconcile.

**Why step 2 comes before step 3:** the compensatable steps must precede the pivot. Debiting
an internal wallet is reversible with a credit; authorising an external card is not. Getting
this order wrong is the most consequential design error in the whole flow.

### The orchestrator

```java
@Service
public class PlaceOrderSaga {

    private static final Duration STEP_TIMEOUT = Duration.ofSeconds(30);
    private static final int MAX_ATTEMPTS = 5;

    private final SagaInstanceRepository sagas;
    private final SagaStepRepository steps;
    private final Map<String, SagaStep> registry;
    private final MeterRegistry metrics;

    /** Called by POST /api/orders, inside the request's transaction. */
    @Transactional
    public UUID start(Order order, String idempotencyKey) {
        SagaInstance saga = SagaInstance.starting(
                order.id(), idempotencyKey, Instant.now().plus(STEP_TIMEOUT));
        sagas.insert(saga);
        // The order row, the idempotency record (Topic 116) and the saga row commit
        // together. There is no window in which the order exists without its saga.
        return saga.id();
    }

    /** Called by the claimer. Exactly one step attempt per invocation. */
    public void advance(UUID sagaId) {
        SagaInstance saga = sagas.require(sagaId);
        MDC.put("sagaId", sagaId.toString());          // Topic 120
        try {
            switch (saga.state()) {
                case RUNNING       -> attemptNextStep(saga);
                case COMPENSATING  -> attemptNextCompensation(saga);
                default            -> { /* terminal; nothing to do */ }
            }
        } finally {
            MDC.remove("sagaId");
        }
    }
}
```

### One step attempt

```java
private void attemptNextStep(SagaInstance saga) {
    SagaStep step = registry.get(saga.currentStep());
    Timer.Sample sample = Timer.start(metrics);
    try {
        executeStepInTransaction(saga, step);
        sample.stop(metrics.timer("orderflow.saga.step",
                "step", step.name(), "outcome", "ok"));

    } catch (TransientFailure e) {
        sample.stop(metrics.timer("orderflow.saga.step",
                "step", step.name(), "outcome", "retry"));
        if (saga.attempt() + 1 >= MAX_ATTEMPTS) {
            beginCompensation(saga, "max attempts exhausted on " + step.name(), e);
        } else {
            // Exponential backoff WITH jitter. Topic 111 -- without jitter, every
            // saga that failed during the same outage retries in lockstep.
            sagas.scheduleRetry(saga.id(), backoffWithJitter(saga.attempt() + 1));
        }

    } catch (Exception e) {
        sample.stop(metrics.timer("orderflow.saga.step",
                "step", step.name(), "outcome", "failed"));
        if (step.isPivot() || saga.pivotPassed()) {
            escalate(saga, step, e);      // no compensation exists past the pivot
        } else {
            beginCompensation(saga, "step " + step.name() + " failed", e);
        }
    }
}

@Transactional
void executeStepInTransaction(SagaInstance saga, SagaStep step) {
    int claimed = steps.tryClaim(saga.id(), step.name());     // ON CONFLICT DO NOTHING
    if (claimed == 0) {                                       // already ran; a retry arrived
        sagas.advanceTo(saga.id(), nextStepAfter(step), deadline());
        return;
    }
    step.execute(SagaContext.of(saga));
    steps.markCompleted(saga.id(), step.name());
    sagas.advanceTo(saga.id(), nextStepAfter(step), deadline());
    // ONE commit: the step claim, the effect, the step record and the saga's position.
}
```

**Note `TransientFailure` versus `Exception`.** A wallet timeout is transient — retry. A
validation failure is permanent — compensate now. **Retrying a permanent failure five times adds
minutes to every failed order**, so during an outage the compensation storm arrives late and all
at once. Classifying failures is a design decision, not an implementation detail.

### One compensation attempt

```java
@Transactional
void compensateStepInTransaction(SagaInstance saga, SagaStep step) {
    int claimed = steps.tryClaimCompensation(saga.id(), step.name());
    if (claimed == 0) return;                     // already compensated
    step.compensate(SagaContext.of(saga));
    steps.markCompensated(saga.id(), step.name());
}

private void attemptNextCompensation(SagaInstance saga) {
    Optional<SagaStep> next = nextCompensatableStep(saga);   // reverse order, completed only
    if (next.isEmpty()) {
        sagas.markState(saga.id(), COMPENSATED);
        metrics.counter("orderflow.saga.outcome", "outcome", "compensated").increment();
        return;
    }
    SagaStep step = next.get();
    try {
        compensateStepInTransaction(saga, step);
        sagas.touch(saga.id(), deadline());
    } catch (Exception e) {
        if (saga.attempt() + 1 >= MAX_COMPENSATION_ATTEMPTS) {
            // TERMINAL. There is no compensation for a compensation.
            sagas.markState(saga.id(), COMPENSATION_FAILED, e.getMessage());
            metrics.counter("orderflow.saga.outcome",
                    "outcome", "compensation_failed", "step", step.name()).increment();
            log.error("saga {} COMPENSATION FAILED at step {} -- manual intervention required",
                    saga.id(), step.name(), e);
        } else {
            sagas.scheduleRetry(saga.id(), backoffWithJitter(saga.attempt() + 1));
        }
    }
}
```

**`MAX_COMPENSATION_ATTEMPTS` should exceed `MAX_ATTEMPTS`.** A failed step costs a
compensation; a failed compensation costs a human. Try harder before giving up — but **do give
up**: an infinite retry loop is an unbounded queue (Topic 90) hiding a broken system.

### The claimer

```java
@Component
class SagaClaimer {

    @Scheduled(fixedDelayString = "${orderflow.saga.poll-interval:PT1S}")
    void claimAndAdvance() {
        List<UUID> claimed = sagas.claimDue(BATCH_SIZE);     // SKIP LOCKED, see below
        for (UUID id : claimed) {
            try {
                saga.advance(id);
            } catch (Exception e) {
                // One saga must never stop the batch.
                log.error("saga {} advance threw", id, e);
            }
        }
    }
}
```

```java
public interface SagaInstanceRepository extends Repository<SagaInstance, UUID> {

    @Modifying
    @Query(value = """
        UPDATE saga_instance
           SET attempt = attempt + 1,
               deadline_at = now() + interval '30 seconds',
               updated_at = now()
         WHERE id IN (
               SELECT id FROM saga_instance
                WHERE state IN ('RUNNING', 'COMPENSATING')
                  AND deadline_at <= now()
                ORDER BY deadline_at
                FOR UPDATE SKIP LOCKED
                LIMIT :batchSize
         )
        RETURNING id
        """, nativeQuery = true)
    List<UUID> claimDue(@Param("batchSize") int batchSize);
}
```

**Extending `deadline_at` inside the claim is the lease.** If this pod dies mid-step the
deadline expires and another claims it. **The lease must exceed the longest plausible step**, or
a slow step is claimed twice concurrently — and then your only protection is the step's
idempotency, which you have from Topic 116. Defence in depth, not accident.

### The two states a human has to care about

```sql
-- Sagas that could not undo themselves. Should be zero. Pages.
SELECT id, order_id, current_step, last_error, updated_at
  FROM saga_instance
 WHERE state = 'COMPENSATION_FAILED';

-- Sagas that are old and still in flight. Alerts.
SELECT id, order_id, state, current_step, attempt, age(now(), created_at) AS age
  FROM saga_instance
 WHERE state IN ('RUNNING', 'COMPENSATING')
   AND created_at < now() - interval '10 minutes'
 ORDER BY created_at;
```

> *illustration of the format, not captured output*
> ```
> <saga-id> | <order-id> | COMPENSATING | DEBIT_WALLET | <n> | <duration>
> ```

**These two queries are the operational deliverable of the whole topic.** A saga
implementation without them is not finished, however elegant the orchestrator.

### What the customer sees in the middle

This is the part that is not code, and it is the part that must be decided:

| Saga state | Order status shown | Inventory | Wallet | What the customer sees |
|---|---|---|---|---|
| `RUNNING`, step 1–2 | `PENDING` | Reserved | Maybe debited | "Processing your order" |
| `RUNNING`, step 3–4 | `PENDING` | Reserved | Debited | "Processing your order" |
| `COMPENSATING` | `CANCELLING` | Being released | Being credited | "We couldn't complete this order; your balance will be restored" |
| `COMPENSATED` | `CANCELLED` | Released | Credited | "Order cancelled, balance restored" |
| `COMPENSATION_FAILED` | `CANCELLING` | Unknown | **Possibly still debited** | **Must be handled by a person.** Do not show a resolved state. |

**Write this table before the code.** It is a product decision, and engineers who skip it
invent the answer at 2am. Note the last row: showing `CANCELLED` while the wallet is still
debited is a lie the customer will discover before you do.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the compensation itself fails, and nothing handles it

**Wrong approach.** Compensations are treated as infallible cleanup:

```java
catch (Exception e) {
    compensateAll(saga);              // assumed to work
    saga.markState(COMPENSATED);      // <-- unconditionally
}
```

Or the subtler variant: compensations are retried forever with no terminal state.

**Exact symptom.** Two shapes, both bad:

- **The unconditional version:** sagas marked `COMPENSATED` whose wallet was never credited.
  Discovered by a customer, or by a monthly reconciliation. **The database asserts a state
  that is false**, which is the worst possible failure because it defeats your own
  investigation.
- **The infinite-retry version:** a saga retrying a compensation every second for days.
  Visible as a flat, non-decreasing count of `COMPENSATING` sagas, growing scheduler load,
  and a log full of the same error. No alert fires because no state is terminal.

**Root cause.** A compensation is an ordinary distributed operation — it calls a service, it
can time out, the service can reject it, the row it wants can be locked. **There is no
compensation for a compensation**, so the failure has no automatic resolution. Treating it as
cleanup rather than as work denies that.

**Fix.**

```java
if (attempt >= MAX_COMPENSATION_ATTEMPTS) {
    sagas.markState(saga.id(), COMPENSATION_FAILED, e.getMessage());
    metrics.counter("orderflow.saga.outcome",
            "outcome", "compensation_failed", "step", step.name()).increment();
    // This counter has a PAGE on it, not a dashboard tile.
}
```

Plus: a bounded retry with jitter before that; a runbook entry naming exactly what a human
must check (wallet balance, inventory reservation, gateway state); and — because compensation
code is the least-executed code you own — **a test that forces every compensation to fail.**

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| `COMPENSATION_FAILED` count above zero | Working as designed: a real inconsistency is now visible instead of hidden. Investigate the saga. |
| `COMPENSATED` sagas whose wallet was never credited | The unconditional-marking bug. Reconcile now, then fix the code. |
| `COMPENSATING` count flat and non-decreasing | Infinite retry. No terminal state. |
| `COMPENSATION_FAILED` always zero, forever | Either genuinely reliable, or the state is unreachable. **Force one in a test and confirm it appears.** A state you have never observed is not implemented. |

### Trap 2 — a saga with no timeout, so a stuck step blocks forever

**Wrong approach.** Steps are driven purely by responses: call the participant, wait for the
answer, advance. No deadline column, or a nullable one that nobody sets.

**Exact symptom.** During a downstream slowdown, orders accumulate in `PENDING`. Inventory is
reserved against them, so the catalogue shows out-of-stock for items that are physically
available. Wallets are debited. **Nothing errors.** The service is healthy by every probe
(Topic 121) because no request is failing — the sagas are simply not moving. The first signal
is a support ticket, or an inventory report that does not match the warehouse.

The variant that is even harder to see: the orchestrator crashed *between* steps. The saga
row says `RUNNING`, but no process is working on it and no process ever will, because work is
only triggered by an incoming response that will never arrive.

**Root cause.** A saga driven by responses has no liveness property. **Nothing in the system
is responsible for noticing that nothing happened.** In a single process a stuck call at
least holds a thread you can see in a dump; here it holds a row nobody is looking at.

**Fix.** Make time the driver, not responses:

1. `deadline_at NOT NULL` on every saga, set on every advance.
2. A claimer that polls for `deadline_at <= now()` — so the timeout reaper and the work
   dispatcher are **the same mechanism**, and there is no second thing to forget.
3. A per-step timeout derived from the participant's own SLO, not a global constant.
4. A `SagaStuck` alert on in-flight age (the second query in Example 2).

```java
// Every write to the saga row sets a new deadline. There is no code path that leaves
// deadline_at in the past without the claimer picking it up.
sagas.advanceTo(saga.id(), nextStep, Instant.now().plus(timeoutFor(nextStep)));
```

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Stuck-saga alert fires during a downstream outage | Working. You know within minutes rather than hours. |
| Sagas in `RUNNING` older than any step timeout | The claimer is not running, or its query does not match their state. Check the scheduler and the partial index. |
| `attempt` climbing on many sagas at once | The claimer is working and the participant is down. Correct behaviour; go look at the participant. |
| Inventory reserved far above open orders | A population of abandoned sagas from before the deadline existed. Write a one-off reconciliation. |
| Every saga times out at exactly the deadline | The step is never completing at all — check the participant call, not the saga. |

### Trap 3 — a non-idempotent compensation

**Wrong approach.** The compensation is written as a plain business operation:

```java
public void compensate(SagaContext ctx) {
    walletService.credit(ctx.userId(), ctx.amount());     // no key, no claim
}
```

**Exact symptom.** Wallets credited more than once for the same failed order. Correlates with
retries and with orchestrator restarts — so it looks intermittent and infrastructure-shaped
rather than like a code bug. The ledger has two credits and one debit for one order.
**Nothing throws**, which is Topic 116's signature exactly.

**Root cause.** Compensations are retried at least as often as steps: on transient failure,
on lease expiry, and after an orchestrator crash between the credit and the `markCompensated`
update. **A compensation that is not idempotent turns a retry into a second refund.**

**Fix.** The same mechanism as everywhere else — claim first, in the same transaction:

```java
@Transactional
void compensateStepInTransaction(SagaInstance saga, SagaStep step) {
    int claimed = steps.tryClaimCompensation(saga.id(), step.name());
    if (claimed == 0) return;
    step.compensate(SagaContext.of(saga));
    steps.markCompensated(saga.id(), step.name());
}
```

And push a key down into the participant, so it is idempotent independently of the
orchestrator:

```java
walletService.credit(ctx.userId(), ctx.amount(),
        "compensate:" + saga.id() + ":DEBIT_WALLET");    // deterministic key
```

**The key must be deterministic**, derived from the saga ID and the step name — not a fresh
UUID, which would be new on every retry and therefore inert (Topic 116, Trap 2).

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Ledger: one debit, one credit per compensated saga | Correct. |
| Two credits for one saga | Non-idempotent compensation. Check the claim and the key. |
| Credits with no matching debit | A compensation ran for a step that never completed. Your reverse iteration is including non-completed steps. |
| Duplicate credits only after a deploy | An orchestrator restart mid-compensation. The claim is missing or in a separate transaction. |

### Trap 4 — choreography with no correlation, so nobody can answer "where is it?"

**Wrong approach.** Five services, five subscriptions, no orchestrator, no saga ID on the
events, no state table. It is elegant, decoupled, and passes all tests.

**Exact symptom.** During an incident someone asks: "order 4417 is stuck in `PENDING`, what
happened?" The honest answer requires reading five services' logs and correlating by order
ID and timestamp. Worse questions have no answer at all: *how many orders are currently
stuck? which step are they stuck on? is this getting better or worse?* Those require a
cross-service aggregation nobody built. And the cyclic case — a compensation event that
retriggers a forward step — appears as an infinite event loop that nobody can localise
because no single service sees the cycle.

**Root cause.** **The flow exists only as the sum of N subscriptions. It is written down
nowhere, so its state is not a queryable fact.** Choreography's coupling advantage is real;
its observability cost is also real and is usually larger.

**Fix — three options, in order of cost:**

1. **Orchestrate.** For money flows, this is the answer. One state table, one query.
2. **Keep choreography, add a saga log.** A dedicated consumer subscribes to every event and
   writes a row per saga per step. You get the observability without the coupling — at the
   cost of a component that must be kept in sync with every event schema.
3. **At minimum**, put a saga ID on every event, propagate the trace context (Topic 119), and
   accept that "where is it" is a trace lookup rather than a query.

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| You can answer "where is order 4417" with one query | Orchestrated, or you built a saga log. |
| The answer needs log correlation across services | Choreography without a saga log. Workable in calm conditions; not at 2am. |
| The same event reappearing repeatedly across services | A choreography cycle. Structurally impossible under orchestration. |
| Nobody can say how many sagas are in flight | **This is the finding.** It is an operational gap, not a preference. |

### Trap 5 — the saga state committed separately from the step's effect

**Wrong approach.** The orchestrator owns the state and the participant owns the effect, so
they naturally end up in two transactions:

```java
inventoryService.reserve(orderId, lines);       // transaction 1, in the participant
sagaRepo.markStepCompleted(sagaId, step);       // transaction 2, in the orchestrator
```

**Exact symptom.** After a crash or a network blip between the two:

- **Effect first:** the step is re-run on the next claim. Saved only by the participant's own
  idempotency — and if the participant is not idempotent, inventory is reserved twice for one
  order, which is Topic 116's consumer race arriving through the saga.
- **State first** (if you reverse them): the step is marked done and never ran. **The saga
  proceeds on a false premise**, and the failure surfaces two steps later as something
  incomprehensible — "payment authorised for an order with no reservation".

**Root cause.** Two commits, one window. It is Topic 115's dual write again.

**Fix.** Where the participant is in-process (as `orderflow`'s inventory and wallet are), put
the effect and the `saga_step` row in **one** local transaction — the
`executeStepInTransaction` in Example 2. Where the participant is genuinely remote, you
cannot; so:

- make the remote call idempotent with a deterministic key derived from
  `(sagaId, stepName)`, and
- record the step **after** the call, accepting that a retry re-calls a step that already
  succeeded — which is harmless precisely because it is idempotent.

**That is the honest answer for a remote participant: you cannot close the window, so you make
re-entry safe.** Anyone claiming otherwise without a distributed transaction is mistaken.

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| Steps marked completed with no corresponding effect | State committed before the effect. The saga is lying to itself. |
| Effects duplicated on retry | Effect committed before the state, and the participant is not idempotent. |
| `saga_step` rows for steps that never ran | Same as the first row. Query for it — it should be zero. |
| A remote step re-called on retry, with no duplicate effect | Correct behaviour for a remote participant with a deterministic key. |

---

## Hands-on proof

### Proof 1 — intermediate states really are visible

Start a saga and pause it before the pivot (a breakpoint, or a deliberately slow stub). From
a second `psql` session:

```sql
SELECT o.id, o.status, i.reserved, w.balance
  FROM orders o
  JOIN inventory i ON i.product_id = <product>
  JOIN wallet    w ON w.user_id    = o.user_id
 WHERE o.id = <order>;
```

**WHAT TO LOOK FOR:** whether the reservation and the balance change are visible while the
saga is still mid-flight.

| What you see | What it means |
|---|---|
| Reserved incremented, balance reduced, order still `PENDING` | **The mechanical statement, demonstrated.** No isolation. Everyone sees the middle. |
| Nothing visible until the saga finishes | You have accidentally wrapped the whole saga in one transaction. That is not a saga — and it will not survive a crash or a remote participant. |
| Order `PENDING` but nothing else changed | The saga has not reached step 1, or the step is not committing. |

Then run whatever report or catalogue query the business uses and see how it counts this order.
**Proof 1 is how you find out whether anyone has made that product decision.**

### Proof 2 — `SKIP LOCKED`, in two sessions

```sql
-- Session 1                                  -- Session 2
BEGIN;
SELECT id FROM saga_instance
 WHERE state = 'RUNNING'
 ORDER BY deadline_at
 FOR UPDATE SKIP LOCKED LIMIT 5;   -- (1)

                                              BEGIN;
                                              SELECT id FROM saga_instance
                                               WHERE state = 'RUNNING'
                                               ORDER BY deadline_at
                                               FOR UPDATE SKIP LOCKED LIMIT 5;   -- (2)
```

**WHAT TO LOOK FOR:** the two id sets.

| What you see | What it means |
|---|---|
| Disjoint sets, neither session blocks | **`SKIP LOCKED` working.** This is what lets N pods share one queue. |
| Session 2 blocks | You omitted `SKIP LOCKED`. Pods will serialise on the head of the queue. |
| Identical id sets | You omitted `FOR UPDATE`. Both pods will process the same sagas. |
| Session 2 returns fewer rows than expected | Correct — it skipped the locked ones and there were not five more. |

### Proof 3 — the saga survives a `kill -9`

```bash
# 1. Start a saga and let it complete step 1.
curl -s -X POST http://localhost:8080/api/orders \
  -H 'Idempotency-Key: proof-3' -H 'Content-Type: application/json' \
  -d '{"userId":1,"lines":[{"sku":"...","qty":1}]}'

# 2. Confirm step 1 committed.
psql -c "SELECT step, state FROM saga_step WHERE saga_id = '<id>';"

# 3. Kill the process hard, mid-saga.
kill -9 $(pgrep -f orderflow.jar)

# 4. Restart and wait one claimer interval.
java -jar orderflow.jar &

# 5. Read the state.
psql -c "SELECT state, current_step, attempt FROM saga_instance WHERE id = '<id>';"
```

**WHAT TO LOOK FOR:** whether the saga resumes at the right step.

| What you see | What it means |
|---|---|
| Step 1 still `COMPLETED`; saga resumed at step 2 | **Correct.** Committed steps stay committed; the row drove the resume. |
| Saga still `RUNNING` at step 1 after several intervals | The claimer is not picking it up. Check `deadline_at` and the state filter. |
| Step 1 executed again | The step claim is missing or in a separate transaction (Trap 5). |
| Saga gone entirely | It was created in a transaction that rolled back — check that `start()` shares the request's transaction. |

### Proof 4 — the unique constraint on `saga_step`

```sql
BEGIN;
INSERT INTO saga_step (saga_id, step, state) VALUES ('<id>', 'DEBIT_WALLET', 'COMPLETED');
INSERT INTO saga_step (saga_id, step, state) VALUES ('<id>', 'DEBIT_WALLET', 'COMPLETED');
```

**WHAT TO LOOK FOR:** SQLState **`23505`** on the second insert.

| What you see | What it means |
|---|---|
| `23505` | The composite primary key is doing its job — a step cannot be recorded twice. |
| Both succeed | No primary key or a surrogate one. **Your step claim is not a claim**; add the composite key. |
| "current transaction is aborted" on a following statement | Topic 116's abort semantics. Use `ON CONFLICT DO NOTHING` in the claim. |

### Proof 5 — a step and its record are in the same transaction

Do not read the code and believe it. Print it:

```java
log.info("step={} txn={} readOnly={}",
        step.name(),
        TransactionSynchronizationManager.getCurrentTransactionName(),
        TransactionSynchronizationManager.isCurrentTransactionReadOnly());
```

Log the same line from inside the participant's method.

| What you see | What it means |
|---|---|
| The same transaction name in both | One transaction. Correct. |
| Different names | Two transactions — the participant is `REQUIRES_NEW`, or the call goes through a proxy boundary you did not intend (Topic 40). |
| `null` in one of them | That code is running with no transaction at all. Self-invocation, or a missing `@Transactional`. |
| Two names *and* `hikaricp.connections.pending` climbing under load | `REQUIRES_NEW` holding two connections per step. Topic 109. |

---

## Failure drill

**Break it like this:** make a compensation fail deliberately, with no terminal state and no
timeout. **Capture:** money missing from a customer's wallet with the saga reporting success,
and a second population of sagas stuck forever. **The fix proves:** `COMPENSATION_FAILED`
must be a terminal, alerting state, and a deadline must drive the saga rather than responses.

### Part A — the compensation that fails silently

**Step 1.** Make the wallet credit fail deterministically:

```java
@Component
class WalletCreditFaultInjector {
    @Value("${orderflow.chaos.fail-wallet-credit:false}")
    private boolean failCredit;

    void maybeFail() {
        if (failCredit) throw new WalletServiceUnavailable("injected fault");
    }
}
```

**Step 2.** Introduce the bug — mark `COMPENSATED` unconditionally:

```java
// BROKEN ON PURPOSE
try {
    compensateAll(saga);
} catch (Exception e) {
    log.warn("compensation failed, continuing", e);      // <-- the whole bug
}
sagas.markState(saga.id(), COMPENSATED);
```

**Step 3.** Force a step-3 failure so the saga must compensate steps 2 and 1:

```bash
export ORDERFLOW_CHAOS_FAIL_PAYMENT=true
export ORDERFLOW_CHAOS_FAIL_WALLET_CREDIT=true
```

**Step 4.** Record the wallet balance, place an order, wait, and read everything:

```sql
SELECT user_id, balance FROM wallet WHERE user_id = 1;

SELECT id, state, current_step, last_error FROM saga_instance WHERE order_id = <n>;
SELECT step, state FROM saga_step WHERE saga_id = '<id>';
SELECT id, status FROM orders WHERE id = <n>;
SELECT product_id, reserved FROM inventory WHERE product_id = <p>;
```

### What to capture, before reading on

Write down, on paper:

1. The wallet balance before and after.
2. The saga's final `state`.
3. Each `saga_step` row's `state`.
4. The order's status, and the inventory reservation.
5. Whether anything **paged**. (Nothing did. That is the point.)

### How to read it

| What you see | What it means |
|---|---|
| Saga `COMPENSATED`, balance still reduced | **The full failure.** The database asserts a state that is false — the worst outcome, because it defeats your own investigation. |
| Order `CANCELLED`, inventory released, wallet still debited | Partial compensation. The customer paid for a cancelled order. |
| Only a `WARN` in the logs | Correct for the broken build, and the reason it survives to production: warnings are not alerts. |
| Saga `COMPENSATING` forever | You did not introduce the unconditional marking — a different bug, and a strictly better one, because at least it is visible. |
| Everything fine | The fault injection did not fire. Check the properties are actually bound (`/actuator/configprops`). |

### Part B — no timeout, so a stuck saga is invisible

**Step 1.** Remove the deadline: make the claimer poll only on `state = 'RUNNING'` with no
`deadline_at` predicate, and stop setting `deadline_at` on advance.

**Step 2.** Make the inventory service hang rather than fail:

```java
if (chaos.hangInventory()) { Thread.sleep(Duration.ofHours(1)); }
```

**Step 3.** Place ten orders. Then, while they hang, check the health endpoints:

```bash
curl -s localhost:8080/actuator/health/readiness
curl -s localhost:8080/actuator/health/liveness
psql -c "SELECT state, count(*) FROM saga_instance GROUP BY state;"
psql -c "SELECT product_id, reserved FROM inventory WHERE product_id = <p>;"
```

| What you see | What it means |
|---|---|
| Both probes report healthy; ten sagas in `RUNNING`; inventory reserved | **The full failure.** The service is "up" and doing nothing. Topic 121's point about what a probe actually asserts. |
| Catalogue shows the product out of stock | The business impact of a stuck saga, visible to customers. |
| No alert of any kind | Because no request failed and no exception was thrown. **Only a time-based check finds this.** |
| Sagas eventually advance | You did not fully remove the deadline. Check the claimer query. |

### Part C — fix both and re-run

1. Restore `deadline_at NOT NULL`, set on every advance, and the claimer's
   `deadline_at <= now()` predicate.
2. Restore `COMPENSATION_FAILED` as a terminal state with a counter and an alert.
3. Add the stuck-saga age query as an alert.
4. Re-run Parts A and B.

| What you see | What it means |
|---|---|
| Part A: saga in `COMPENSATION_FAILED`, counter incremented, page fires | **Fixed.** The inconsistency is visible and owned. |
| Part B: sagas move to `COMPENSATING` after the deadline; stuck-saga alert fires | **Fixed.** Time drives the saga. |
| Part A: saga retries the compensation a few times first | Correct — bounded retry with jitter before escalation. |
| Part B: alert fires but sagas never leave `COMPENSATING` | The compensation is also failing. That is Part A's path; both fixes are needed. |

### What the drill proves

1. **A saga that cannot undo itself is a business inconsistency, and the only correct
   handling is to make it loud.** Every design that hides it makes the eventual reconciliation
   harder.
2. **A saga with no timeout has no liveness property.** Nothing in the system is responsible
   for noticing that nothing happened, and every health probe you own will report success.
3. **The compensation path is the least-executed code you own.** If you have never seen it
   run, you do not know that it works — which is why forcing it is a drill and not a thought
   experiment.

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` loop is the WRONG way to measure any of this.** It measures
> JIT warm-up, dead-code elimination and ambient noise. Topic 77 (JMH) is where you learn to
> do it properly.

And the sharper version for this topic: **JMH is the wrong tool entirely.** A saga's duration
is dominated by network calls, retry backoff and claimer poll intervals. Nothing here is a
CPU-bound repeatable operation. **The instrument is the Topic 65 harness plus the state table
itself**, and reaching for a microbenchmark is a diagnostic error.

### The instrument for each claim

| Claim | Instrument | What invalidates it |
|---|---|---|
| "Sagas complete" | `orderflow.saga.outcome` counter by outcome | Counting starts, not terminal states |
| "Compensations work" | A test that forces every compensation to fail, plus the drill | Never having observed one |
| "Nothing is stuck" | The in-flight age query, alerting | A dashboard nobody looks at |
| "Steps are idempotent" | Topic 116's duplicate counters per participant | Testing single-threaded |
| "The claimer keeps up" | Claim latency: `now() - deadline_at` at claim time | Measuring only successful sagas |
| "Compensation is rare" | Ratio of `compensated` to `completed` outcomes | Measuring only at 1x arrival rate |
| "Saga overhead is acceptable" | End-to-end p99 of `POST /api/orders` vs the Topic 65 baseline | Timing the orchestrator rather than the order |

### The four instruments worth having permanently

```java
// 1. Terminal outcomes. The single most important saga metric.
metrics.counter("orderflow.saga.outcome", "type", "PLACE_ORDER",
                "outcome", "completed|compensated|compensation_failed").increment();

// 2. Per-step duration and outcome. Finds the slow or flaky participant.
metrics.timer("orderflow.saga.step", "step", stepName,
              "outcome", "ok|retry|failed");

// 3. In-flight gauge, by state. The saturation signal.
metrics.gauge("orderflow.saga.inflight",
              Tags.of("state", "RUNNING"), sagas, r -> r.countByState(RUNNING));

// 4. Oldest in-flight saga age. The single best stuck detector.
metrics.gauge("orderflow.saga.oldest_inflight_seconds", sagas, r -> r.oldestInflightAge());
```

**Cardinality (Topic 118): tag by saga type, step and outcome. NEVER by saga ID or order
ID.** One series per saga will take down Prometheus long before it helps you.

| What you see | What it means |
|---|---|
| `compensation_failed` above zero | Real inconsistency. Page. Investigate the individual sagas. |
| `oldest_inflight_seconds` climbing steadily | A participant is down or a step is hanging. **This is the metric to alert on**, ahead of all the others. |
| `compensated` rate rising with arrival rate | Contention: inventory or wallet conflicts increase under load. Expected up to a point — decide what the point is. |
| `inflight` growing without bound | Sagas start faster than they finish. Little's Law (Topic 90) applied to sagas. |
| Step timer p99 far above the participant's own p99 | The gap is queueing at the claimer, not the participant. Raise batch size or poll frequency. |
| All metrics healthy but customers report stuck orders | Your terminal states are being reached optimistically — the Trap 1 bug. |

### The reconciliation queries — run them on a schedule

```sql
-- (a) Should be zero: a compensated saga whose wallet was never credited.
SELECT s.id FROM saga_instance s
 WHERE s.state = 'COMPENSATED'
   AND EXISTS (SELECT 1 FROM saga_step st
                WHERE st.saga_id = s.id AND st.step = 'DEBIT_WALLET'
                  AND st.state <> 'COMPENSATED');

-- (b) Should be zero: inventory reserved against no in-flight or confirmed order.
SELECT i.product_id, i.reserved FROM inventory i
 WHERE i.reserved > (SELECT coalesce(sum(ol.qty), 0)
                       FROM order_line ol JOIN orders o ON o.id = ol.order_id
                      WHERE ol.product_id = i.product_id
                        AND o.status IN ('PENDING', 'CONFIRMED'));

-- (c) Should be zero: a step recorded COMPLETED for a saga that never reached it.
SELECT st.saga_id, st.step FROM saga_step st
  JOIN saga_instance s ON s.id = st.saga_id
 WHERE st.state = 'COMPLETED' AND s.state = 'STARTED';
```

**These run against production data, not fixtures.** A saga implementation is only as
trustworthy as the invariants you check on real rows.

### Comparing against the Topic 65 baseline

Run the harness at matched arrival rates, before and after introducing the saga, and compare
end-to-end `POST /api/orders` p50/p95/p99 — **not** the orchestrator's internal timings.

| What you see | What it means |
|---|---|
| p99 up by roughly one claimer poll interval | Expected. The saga is asynchronous; the response returns after `start()`, and completion follows. Shorten the interval if it matters. |
| p99 unchanged, completion latency up | Correct design: the client is not waiting for the saga. Make sure your SLO measures the right thing. |
| p99 up by much more than the poll interval | The `start()` transaction grew, or the saga insert is contending. Check the index. |
| Compensation rate rising with load | Contention between concurrent sagas on inventory or wallet rows (Topic 52). |
| Everything bounded by `hikaricp.connections.pending` | The claimer's transactions are competing with request traffic for the pool. Give it its own pool, or reduce the batch (Topic 109). |

---

## Practice exercises

### 1 — easy: read the state machine

Using `orderflow`'s existing place-order flow, produce a one-page table with a row per step:

1. The step name.
2. Compensatable, pivot, or retriable — and **why**.
3. The compensation, in one sentence of business language, not code.
4. What the customer sees while that step is in flight.
5. What happens if the compensation fails.

**Done when:** you can defend the position of the pivot, and every compensation is described
in words a product manager would accept.

### 2 — medium: implement and break it

1. Implement the state table, `saga_step` with the composite key, and the `SKIP LOCKED`
   claimer.
2. Put each step's effect and its `saga_step` row in one transaction; prove it with Proof 5.
3. Run the failure drill, Parts A and B, and capture the outputs.
4. Fix both, re-run, and capture again.
5. Add the four metrics and the three reconciliation queries.
6. Write a test that forces **every** compensation to fail and asserts the saga reaches
   `COMPENSATION_FAILED`.

**Done when:** step 6 passes, the reconciliation queries return zero, and you have the
before/after captures from step 3 and 4 side by side.

### 3 — hard: the design note, plus chaos

Write a two-page design note for `orderflow`'s place-order saga, in the shape Topic 131
teaches, containing:

1. **The step table** from exercise 1, including the pivot justification.
2. **The observable-intermediate-states table** — what each state means to the customer, the
   catalogue, the warehouse and the finance report. **Get someone non-engineering to read
   this section**; if they cannot answer "what does `CANCELLING` mean to a customer", it is
   not finished.
3. **The choreography-versus-orchestration decision**, with the observability argument stated
   as the deciding factor and the coupling cost acknowledged honestly.
4. **The failure matrix:** for each step, what happens on transient failure, permanent
   failure, timeout, and orchestrator crash — twelve cells minimum.
5. **The runbook** for `COMPENSATION_FAILED`: exactly what a human checks, in what order,
   and what they are authorised to do.
6. **A chaos test** killing the orchestrator at four points — mid-step-1, between steps 1 and
   2, mid-compensation, and between two compensations — with the resulting state recorded for
   each.

**Marking rubric:**

| Criterion | Fail | Pass | Strong |
|---|---|---|---|
| Pivot identification | Absent | Named | Justified, with the consequence of misplacing it |
| Intermediate states | Not considered | Tabulated | Reviewed by a non-engineer |
| Compensation failure | "Retry" | Terminal state + alert | Plus a runbook a stranger could follow |
| Timeouts | Global constant | Per-step | Derived from each participant's SLO |
| Idempotency | Assumed | Implemented | Deterministic keys, proven by a concurrent test |
| Chaos test | Not run | Four kill points | Plus the state captured and explained for each |
| Orchestration choice | Asserted | Argued | Argued on observability, with the coupling cost conceded |

---

## Interview questions

### Q1 — "We need to place an order across four services. Why not a distributed transaction?"

**MID-LEVEL ANSWER.** "2PC doesn't scale, so we use a saga instead." A slogan, and it invites
the follow-up that exposes the gap.

**SENIOR ANSWER.**

> "Two reasons, and the second is the one that matters. Practically, 2PC needs every
> participant to support it — an external payment gateway will not, so the question is
> usually settled before it is asked. Structurally, 2PC's coordinator holds locks across the
> whole transaction, so any participant being slow or down blocks everyone. That is the
> availability cost, and for an order flow it is not acceptable.
>
> So a saga: a sequence of local transactions, each with a compensating transaction. But I
> would be precise about what I am giving up rather than saying 'instead of', because they
> are not two implementations of the same guarantee. A saga is **not atomic** and has **no
> isolation**. Intermediate states are visible to everyone — the catalogue, the customer, a
> finance report. And a compensation is a new business operation that can itself fail;
> there is no compensation for a compensation.
>
> That means before writing any code I need the business to tell me what an order with
> inventory reserved and no payment *means*, and what a compensated order looks like to a
> customer. If the answer is 'that state must never be visible', a saga is the wrong pattern
> and I should redraw the service boundaries so the operation is one local transaction."

**WHAT SEPARATES THEM.** Naming both costs (atomicity and isolation) rather than just
"eventual consistency"; treating the intermediate state as a product question with a named
owner; and being willing to conclude that a saga is the wrong answer, which is the strongest
signal in the whole reply.

**FOLLOW-UP: "Where would you redraw the boundary?"**

> "If inventory and orders must be atomic, they belong in one service and one database. The
> saga would then span order-plus-inventory and payment, which is a two-step saga with a
> single compensatable step before the pivot — much easier to reason about. Reducing the
> number of steps before the pivot is generally the highest-value simplification available."

### Q2 — "What happens when a compensation fails?"

**MID-LEVEL ANSWER.** "We retry it." True and incomplete, and the incompleteness is the whole
question.

**SENIOR ANSWER.**

> "Retry with bounded attempts and jittered backoff first — most compensation failures are
> transient, and without jitter every saga that failed during the same outage retries in
> lockstep and hits the recovering service simultaneously.
>
> But bounded, because there is no compensation for a compensation. When the attempts are
> exhausted the saga goes to a terminal `COMPENSATION_FAILED` state, increments a counter
> that pages, and stops. That is deliberately loud: the business state is genuinely
> inconsistent — the customer may be debited for an order that will not ship — and the only
> honest handling is a human with a runbook.
>
> The failure mode I would specifically look for in a review is marking the saga
> `COMPENSATED` regardless of whether the compensation succeeded. That is worse than doing
> nothing, because the database now asserts something false and defeats your own
> reconciliation.
>
> Two other things. Compensations must be idempotent, with a key derived from the saga ID and
> the step name, or a retry becomes a second refund. And I would keep reconciliation queries
> running against production — for instance, compensated sagas whose wallet was never
> credited should be a permanent zero."

**WHAT SEPARATES THEM.** Bounded retry with a named reason for jitter; a terminal state that
pages; naming the optimistic-marking bug as the thing to look for; idempotent compensations
with deterministic keys; and the reconciliation query, which is the sign of someone who has
operated this.

**FOLLOW-UP: "What does the human actually do?"**

> "The runbook names the invariant to check in each system — wallet balance versus the
> ledger, the inventory reservation, the gateway's view of the authorisation — and what they
> are authorised to fix directly versus what needs finance. If I cannot write that runbook,
> I have not finished designing the saga, because I have created a state with no owner."

### Q3 — "Choreography or orchestration?"

**MID-LEVEL ANSWER.** "Choreography is more decoupled, so it's better for microservices."
A textbook answer that ignores the operational reality.

**SENIOR ANSWER.**

> "For anything touching money, orchestration, and the reason is observability rather than
> coupling.
>
> With choreography the flow is not written down anywhere — it is emergent from N
> subscriptions. So during an incident, 'where is order 4417 and what happens next' requires
> correlating logs across five services, and 'how many orders are stuck right now' is
> effectively unanswerable without building an aggregation. With orchestration it is one row
> in one table and a `SELECT`.
>
> Choreography's advantages are real: adding a participant is a new subscriber rather than an
> orchestrator change, and there is no central component to fail. But the orchestrator is not
> much of a single point of failure if it is stateless over a state table — any pod can claim
> any saga with `SELECT ... FOR UPDATE SKIP LOCKED`. So the coupling advantage is real and the
> availability advantage is mostly not.
>
> I would use choreography for genuinely fire-and-forget fan-out — notifications, analytics —
> where nobody will ever ask 'where is it'. And if a team has already chosen choreography for
> a money flow, the pragmatic fix is a saga log: one consumer subscribing to every event and
> writing a row per saga per step. You get the queryable state without changing the
> architecture."

**WHAT SEPARATES THEM.** Making observability the decision criterion; dismantling the
single-point-of-failure objection with a specific mechanism; giving a real case for
choreography; and offering a migration path rather than a verdict.

### Q4 — "How do you stop a saga getting stuck?"

**MID-LEVEL ANSWER.** "Add timeouts to the service calls." Necessary, insufficient, and it
misses the crash case entirely.

**SENIOR ANSWER.**

> "Call timeouts are necessary but they only cover the case where the orchestrator is alive
> and waiting. The harder case is the orchestrator crashing between steps — then no call is
> outstanding, nothing will time out, and the saga simply sits there. Nothing in the system is
> responsible for noticing that nothing happened.
>
> So the saga has to be driven by time, not by responses. Every saga row has a non-nullable
> `deadline_at`, reset on every advance. A scheduled claimer polls for rows where the deadline
> has passed and claims them with `SELECT ... FOR UPDATE SKIP LOCKED` so multiple pods share
> the queue without blocking each other or double-processing. Claiming extends the deadline,
> so it is a lease: if that pod dies, the lease expires and someone else takes over.
>
> The lease has to be longer than the longest plausible step, or a slow step gets claimed
> twice concurrently — and then my only protection is the step's idempotency, which I have
> anyway, but I would rather not rely on it.
>
> And the detection layer: a gauge of the oldest in-flight saga age, with an alert. That is
> the single best stuck-saga detector, because it catches every cause at once — a hanging
> participant, a dead claimer, a saga in a state the claimer's query does not match. It is
> also the case where every health probe reports green while nothing is progressing, so it is
> the only signal that would fire at all."

**WHAT SEPARATES THEM.** Distinguishing the hang from the crash; making the timeout reaper and
the work dispatcher one mechanism; the lease-longer-than-the-step reasoning; and choosing
oldest-in-flight-age as the detector because it is cause-agnostic.

### Q5 — "Your saga reserves inventory and debits a wallet, then payment fails. Walk me through what happens."

**MID-LEVEL ANSWER.** "We roll back the inventory and the wallet." The word "roll back" is
the tell — there is nothing to roll back.

**SENIOR ANSWER.**

> "Nothing rolls back — both of those committed. I run compensations, in reverse order:
> credit the wallet, then release the inventory. Reverse order matters, because if I release
> inventory first there is a window where the customer is debited for an order with nothing
> reserved, and that is the state most likely to be observed by a report or a support agent.
>
> Each compensation is a local transaction claimed against `(saga_id, step)` with
> `ON CONFLICT DO NOTHING`, so a retry after a crash does not credit twice. The wallet credit
> also carries an idempotency key derived from the saga ID and the step name, so the wallet
> service is independently safe.
>
> If a compensation fails, bounded retry with jitter, then `COMPENSATION_FAILED` and a page.
>
> The part I would raise unprompted: payment being the *pivot* means the ordering here is
> load-bearing. Everything compensatable has to come before it. If someone later adds a step
> after the payment authorisation that can fail permanently, the saga is broken — there would
> be nothing to do but escalate. So the step ordering is an invariant I would enforce with a
> test, not a comment.
>
> And the customer sees `CANCELLING`, then `CANCELLED` with the balance restored. Which is a
> state someone in product had to define, and I would want that written down before I shipped
> it."

**WHAT SEPARATES THEM.** Rejecting the word "roll back"; justifying reverse order with the
specific bad state it avoids; idempotency at two levels; and volunteering the pivot invariant
and the product decision — both of which show design ownership rather than implementation.

---

## Mental model checkpoint

Answer without scrolling up.

1. **State what a saga is and what it is not, in two sentences.**
   A sequence of local transactions, each with a compensating transaction that semantically
   undoes it. It is not atomic: intermediate states are observable, and a compensation can
   itself fail.

2. **What is a pivot step, and what rule governs where it sits?**
   The point of no return — after it succeeds the saga must go forward. All compensatable
   steps come before it; only retriable steps come after.

3. **What must share one local transaction, and why?**
   A step's effect and its `saga_step` record. Otherwise a crash between them either
   re-runs a completed step or skips one that was never run.

4. **Why `SELECT ... FOR UPDATE SKIP LOCKED` rather than `FOR UPDATE`?**
   Multiple pods share one work queue. `SKIP LOCKED` lets each claim disjoint rows without
   waiting on the other's locks; plain `FOR UPDATE` serialises them at the head of the queue.

5. **What happens when a compensation fails, and what must never happen?**
   Bounded retry with jitter, then a terminal `COMPENSATION_FAILED` that pages. What must
   never happen is marking the saga `COMPENSATED` regardless — the database would assert
   something false.

6. **Why does a saga need a deadline even if every service call has a timeout?**
   Call timeouts only cover a live orchestrator waiting on a response. A crash between steps
   leaves no outstanding call, so nothing times out. Time must drive the saga.

7. **What is the honest deciding factor between choreography and orchestration?**
   Observability. Choreography's flow is emergent from N subscriptions and its state is not
   queryable; orchestration keeps it in a table, so "where is this saga" is a `SELECT`.

---

## Quick reference card

### The rule

```
Saga = local transactions + compensations. NOT atomic. NO isolation.
Intermediate states are visible to everyone.
A compensation is a new operation that can fail. There is no compensation for one.
```

### Step kinds and ordering

```
compensatable ... compensatable ... [PIVOT] ... retriable ... retriable
                                       ^ after here, forward only
If a step after the pivot can fail permanently, the saga is designed wrong.
```

### The transaction boundary

```java
@Transactional
void step() {
    tryClaim(sagaId, stepName);   // ON CONFLICT DO NOTHING  -- Topic 116
    effect();                     // the actual work
    markCompleted(sagaId, step);  // the record
    advanceSaga(nextStep, deadline);
}   // ONE commit
```

### The claimer

```sql
UPDATE saga_instance SET attempt = attempt + 1, deadline_at = now() + <lease>
 WHERE id IN (SELECT id FROM saga_instance
               WHERE state IN ('RUNNING','COMPENSATING') AND deadline_at <= now()
               ORDER BY deadline_at FOR UPDATE SKIP LOCKED LIMIT :n)
RETURNING id;
-- lease > longest plausible step, or a slow step is claimed twice.
```

### States

```
STARTED -> RUNNING -> COMPLETED
              |
              v
        COMPENSATING -> COMPENSATED
              |
              v
        COMPENSATION_FAILED   <-- terminal, PAGES, needs a runbook
```

### Metrics (tag by type/step/outcome, NEVER by saga or order id)

```
orderflow.saga.outcome{outcome=completed|compensated|compensation_failed}
orderflow.saga.step{step,outcome=ok|retry|failed}          timer
orderflow.saga.inflight{state}                             gauge
orderflow.saga.oldest_inflight_seconds                     gauge  <-- alert on this one
```

### Things that do not work

```
try/catch as a saga           -> does not survive kill -9
Response-driven, no deadline  -> a crash between steps hangs forever, silently
Mark COMPENSATED regardless   -> the DB asserts something false
Non-idempotent compensation   -> double refund on retry
Compensating in forward order -> exposes "charged with nothing reserved"
Compensating a non-completed step -> phantom refunds
FOR UPDATE without SKIP LOCKED-> pods serialise on the head of the queue
Fresh UUID as a retry key     -> new every attempt; inert (Topic 116)
```

### Gotchas checklist

```
[ ] Pivot identified, and every compensatable step precedes it
[ ] deadline_at NOT NULL, reset on every advance
[ ] Claimer uses FOR UPDATE SKIP LOCKED, and the lease exceeds the longest step
[ ] Step effect + saga_step row in ONE transaction (proven with Proof 5)
[ ] Compensations idempotent, keyed on (sagaId, stepName)
[ ] Compensations iterate COMPLETED steps only, in REVERSE
[ ] COMPENSATION_FAILED is terminal, counted, paged, and has a runbook
[ ] A test forces every compensation to fail
[ ] Observable-intermediate-states table written and read by product
[ ] Reconciliation queries scheduled and alerting
```

---

## When would I use this at work?

**1. Any time an operation spans two datastores or two services.**

The question is not "should we use a saga" — it is **"what does the intermediate state mean,
and can the business tolerate it being visible?"** If it cannot, the answer is not a better
saga; it is redrawing the boundary so the operation is one local transaction. Being able to
propose that instead of building the saga is often the highest-value contribution available.

**2. Reviewing a saga implementation.**

Five mechanical questions that take five minutes: Where is the pivot, and does everything
compensatable precede it? Is every step's effect in the same transaction as its record? What
happens when a compensation fails — is there a terminal state? Is there a deadline on every
step, driven by time rather than responses? Has anyone ever seen the compensation path run?
The last one is usually "no", and it is usually where the bug is.

**3. During an incident involving inconsistent business state.**

`SELECT state, count(*) FROM saga_instance GROUP BY state` and the oldest-in-flight query
tell you within a minute whether you are looking at stuck sagas, failed compensations, or
something else entirely. **That is the payoff for orchestrating**, and it is the argument to
make when someone proposes choreography for a money flow.

---

## Connected topics

**Prerequisites:**

- **116 — Idempotency:** **hard prerequisite.** Every step and every compensation is retried.
  A saga on non-idempotent operations turns one duplicate into many.
- **115 — The outbox pattern:** how a saga step publishes an event atomically with its
  effect, and why the relay is at-least-once.
- **54–55 — `@Transactional` and isolation:** the local-transaction boundary that makes each
  step atomic, and why a transaction must not span a network call.
- **111 — Resilience4j:** bounded retry with jitter, and the circuit breaker that stops a
  saga hammering a dead participant.
- **113–114 — Kafka:** the delivery semantics behind choreography, and why redelivery is
  structural.
- **52 — Hibernate locking:** the contention on inventory and wallet rows that makes
  compensations more common under load.

**Also relevant:**

- **09 — Exception API design:** classifying transient from permanent failures is what decides
  retry versus compensate, and it is a design decision made in the exception hierarchy.
- **28 — Sealed types:** the saga state machine and step kinds as a sealed hierarchy gives
  compile-checked exhaustiveness when a state is added.
- **90 / 93 — Pool sizing and bounded queues:** an unbounded retry loop is an unbounded queue;
  Little's Law applies to in-flight sagas as much as to requests.
- **101 / 107 — Virtual threads:** the claimer's steps block on network calls, so virtual
  threads suit it — but they remove the concurrency ceiling, so bound the batch deliberately.
- **109 — HikariCP:** the claimer competes with request traffic for connections. Give it its
  own pool or bound the batch.
- **121 — Actuator and k8s probes:** the drill's Part B is the case where every probe reports
  green while nothing progresses.
- **126 — Build, buy or adopt:** whether to hand-roll this or adopt Temporal, Axon or
  Eventuate. Learn the hand-rolled version first so the evaluation is informed.

**This unlocks:**

- **118 — Metrics:** the four saga instruments, and the cardinality rule that forbids tagging
  by saga ID.
- **119 — Tracing:** a saga spans processes and time; linking spans of steps minutes apart is
  the hard case for trace context.
- **120 — MDC:** the saga ID belongs on every log line the orchestrator emits.- **124 — The readiness gate:** a service with stuck sagas and no stuck-saga alert is not
  production-ready, whatever its probes report.
- **131 — Design-doc authorship:** the hard exercise here is a design note; Topic 131 teaches
  the form properly.

---

*Java baseline 21, Spring Boot 4.1, Postgres, Kafka. Spring, Hibernate and Kafka client
versions come from the Boot BOM; do not pin them by hand. SQLState `23505` for a unique
violation is stable and is the value to branch on, but confirm it yourself. `SELECT ... FOR
UPDATE SKIP LOCKED` semantics are stable Postgres behaviour; your broker's redelivery
behaviour is configuration, not physics, so verify it rather than assuming. The saga
implementation here is deliberately hand-rolled so the mechanism is visible — Temporal, Axon,
Eventuate and Spring State Machine all exist, and Topic 126 is where you decide whether to
adopt one; learn this version first so that decision is informed rather than defensive. The
structural fact worth committing to memory is the mechanical statement: a saga is a sequence
of local transactions with compensations, it is not atomic, intermediate states are
observable, and a compensation can itself fail.*
