# 09 — Exception Design: When Checked Exceptions Damage an API

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

Topic 08 gave you a kitchen. This topic is about the **menu**.

The menu is what the customer sees. It is a contract. Whatever is printed on it,
every customer must now deal with — forever, on every visit.

So imagine printing this on your menu:

> *"The salmon may be unavailable because our supplier's refrigerated truck can break
> down on the A34."*

Two things go wrong.

**First, you leaked your supply chain onto the menu.** The customer does not care
which road the truck uses. Worse, when you switch suppliers next year, you have to
reprint every menu — and every customer who wrote a plan around "A34 breakdown" has
to change their plan too.

**Second, and much worse: customers stop reading it.** Every single dish now carries
a warning. Waiters start saying "yeah, ignore that bit". And on the one night the
salmon genuinely is unavailable, nobody notices, because the warning is background
noise.

That second failure — **the warning that gets ignored because it is everywhere** — is
what a swallowed `catch` block is. It is the failure drill at the end of this doc.

The menu discipline you actually want:

- Print it if the **customer has a decision to make**.
- Do not print it if their only response is "oh, okay".
- Never print your supplier's name.

Those three rules are the whole of exception API design.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

There is no construct in TypeScript, JavaScript, or NestJS that forces a **caller** to
handle or re-declare a specific failure at compile time. `catch (e)` binds `unknown`,
and a function's type says nothing at all about what it can throw. You have never had
to design around this, so there is no habit to transfer and no intuition to correct —
there is only a new decision to learn how to make.

That is why this whole topic exists as a separate doc. In your stack, "what errors can
this throw" is documentation. In Java, it is part of the type signature, it propagates
up the call stack automatically, and changing it is a breaking API change.

### The partial analogue that IS worth using: errors as values

You have seen this pattern:

```ts
type ChargeResult =
  | { kind: "ok";       transactionId: string }
  | { kind: "declined"; code: string }
  | { kind: "retryable"; retryAfterMs: number };

const result = await charge(order);
switch (result.kind) {
  case "ok": ...
  case "declined": ...
  case "retryable": ...
  // TS errors if you miss a case
}
```

**Verdict: PARTIAL analogue — and a genuinely useful one.** Java 21 can express this
almost exactly, with sealed interfaces plus records plus pattern matching (Topics
27–29):

```java
sealed interface ChargeResult permits Ok, Declined, Retryable { }
record Ok(String transactionId)     implements ChargeResult { }
record Declined(String code)        implements ChargeResult { }
record Retryable(long retryAfterMs) implements ChargeResult { }
```

The reason this matters here: **most of what people model as checked exceptions is
better modelled as a return value.** A declined card is not exceptional. It is one of
the three normal outcomes of charging a card. Exceptions are for the paths where
returning is not sensible; unions are for the paths where it is.

### NestJS exception filters — an honest analogue, later

`@ControllerAdvice` is Nest's exception filter with a different name (Topic 46). Both
turn a thrown domain object into an HTTP response in one place. The design work — a
stable domain exception hierarchy that a single handler can map — is what this topic
sets up.

### Mapping table

| You know | Java | Verdict |
|---|---|---|
| Any function may throw anything; types say nothing | Checked exceptions in the signature | **NO ANALOGUE** |
| `Result<T,E>` / neverthrow / discriminated union | sealed interface + records + `switch` | **PARTIAL** — Java's is language-level and exhaustiveness-checked |
| `HttpException` subclasses in Nest | domain exceptions + `@ControllerAdvice` | **PARTIAL** — do not put HTTP status in your domain layer |
| Nest exception filter | `@ControllerAdvice` | **HONEST ANALOGUE** (Topic 46) |
| Unhandled rejection crashes the process (Node 15+) | Uncaught exception kills only that thread | **PARTIAL** — Java's default is quieter and therefore more dangerous |

---

## What is this?

Exception **design** is three separable decisions, and people conflate them:

**1. Checked or unchecked?**
A statement about whether the *caller* is obliged to plan for this.

**2. Exception or return value?**
A statement about whether this outcome is *exceptional* or *expected*.

**3. What shape is the hierarchy?**
A statement about how many `catch` clauses your callers must write, and how your
error contract survives contact with HTTP, Kafka, and a retry loop.

### The checked-exception argument, honestly, both sides

You need to be able to argue this from either side in an interview and in a design
review. Here it is with the actual consequences, not slogans.

**The case FOR checked exceptions:**

- They are the only mechanism in mainstream Java that makes a failure mode
  *impossible to ignore by accident*. You must write something.
- They document the failure contract in a machine-checked way. A signature that says
  `throws InsufficientFundsException` is verified documentation; a Javadoc `@throws`
  is not.
- For genuinely recoverable, foreseeable conditions with a real alternative action,
  they push handling to the right place and keep it there under refactoring.
- Removing one from a signature is source-compatible for callers. Adding one is not —
  which is a real compatibility guarantee in the useful direction.

**The case AGAINST, with consequences:**

- **They do not compose through lambdas.** `Function.apply` does not declare `throws`.
  So this does not compile:
  ```java
  List<Order> orders = ids.stream()
          .map(id -> repository.load(id))   // load() throws OrderNotFoundException
          .toList();                        // COMPILE ERROR
  ```
  Every stream, `Optional.map`, `CompletableFuture.thenApply` and `Comparator` becomes
  a wrapping exercise. This is not a small annoyance; it is a structural conflict with
  every API written after Java 8.

- **They leak implementation into signatures.** `List<Order> findAll() throws
  SQLException` tells every caller you use JDBC. Move to MongoDB and every caller's
  code changes. The abstraction you spent effort building has a hole in it.

- **They propagate.** One checked exception at the bottom of the stack forces a
  `throws` clause or a `try` block in every frame above it. The usual response is
  `throws Exception` at some layer, which erases all information for everything below.

- **They get swallowed under deadline pressure.** This is the empirical one. The
  compiler demands *something*. At 5pm on a Friday, `catch (SQLException e) {
  log.warn("oops"); }` is something. Now you have a silent failure with a log line
  nobody reads. This is the most common failure mode in real Java codebases, and it is
  a direct consequence of forcing a response at a place where no good response exists.

**The verdict the industry reached, and the evidence:**

Spring made its **entire data-access hierarchy unchecked**. `DataAccessException`
extends `RuntimeException`, and Spring translates every `SQLException`,
`HibernateException` and vendor-specific error into it. That was a deliberate,
argued-for decision by people who had watched a decade of `catch (SQLException e) {}`.
Hibernate did the same. Jackson did the same for its runtime path. Kotlin removed
checked exceptions from the language entirely.

That is not proof that checked exceptions are wrong. It *is* strong evidence that the
default should be unchecked, and checked should be a deliberate exception you can
justify in one sentence.

### The decision test that actually works

> **Make it checked only if the caller has a genuine alternative action that is not
> "log it and give up".**

Apply it honestly. Say the sentence out loud. If it comes out as "the caller can log
it", "the caller can decide what to do", or "the caller should know about it", the
answer is unchecked. Those are all disguised ways of saying *there is no alternative
action*.

Real alternative actions look like: write the record to a rejects file; fall back to
the cached price; try the secondary payment provider; return 409 and let the client
retry with a new idempotency key.

### The third option people forget

Before choosing checked or unchecked, ask whether it should be an exception at all.

| Outcome | Frequency | Model as |
|---|---|---|
| Card declined | ~5% of charges | **return value** — sealed `ChargeResult` |
| Insufficient stock | common on popular SKUs | **return value**, or unchecked if it is a violated precondition |
| Idempotency key already seen | expected, by design | **return value** — it is a successful outcome |
| Payment provider returns 503 | occasional | unchecked, wrapped, retried by a policy |
| Order ID not found | caller passed a bad ID | unchecked — a defect in the caller |
| Database is down | rare | unchecked — nothing to do but fail |
| Malformed line in a 2M-row import | ~0.01% of lines | **checked** — the caller writes it to rejects and continues |

Notice that exactly one row in that table earns a checked exception, and it is the one
where the caller genuinely does something else.

---

## Why does it matter?

**1. It is the decision you cannot walk back.** A `throws` clause is part of your
signature. Adding one to a published method breaks every caller at compile time. Once
other teams depend on your service, your error contract is as permanent as your URL
paths — and error responses are an API surface with exactly the same compatibility
obligations as the success path.

**2. Swallowed exceptions are the most expensive silent bug there is.** They produce no
exception, no error metric, no failed request — only a divergence between two tables
that someone finds eleven days later from a support ticket. Your dashboard stays green
throughout. The failure drill in this doc produces that exact shape on purpose, because
recognising it is worth more than any rule you could memorise.

**3. Checked exceptions in the wrong place quietly lower your code quality.** Not
because they are ugly — because the least careful call site sets the bar. If four
places must catch something and two of them log-and-continue, you now have two silent
failure paths that you created by forcing a response where none existed.

**4. It decides whether your transactions roll back.** Spring's default rollback rule
covers unchecked exceptions only. Choosing `extends Exception` instead of `extends
RuntimeException` for a domain failure means a `@Transactional` method that throws it
**commits** (Topic 54). That is a data-corruption bug whose root cause is a decision
made in a file with no Spring in it.

---

## Syntax breakdown

New constructs only. Topic 08 covered `throws`, multi-catch, and try-with-resources.

### A domain exception base class with a stable machine-readable code

```java
public abstract class OrderflowException extends RuntimeException {

    private final ErrorCode code;

    protected OrderflowException(ErrorCode code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public ErrorCode code() { return code; }
}
```

| Bit | Why it is there |
|---|---|
| `abstract` | Nobody throws the base type. Callers catch it; they never construct it. |
| `extends RuntimeException` | The default. Every subclass is unchecked unless a subclass deliberately extends `Exception` instead. |
| `ErrorCode code` | A stable enum, separate from the message. The message is for humans and may change; the code is an API contract and may not. This is what your `@ControllerAdvice` and your clients switch on. |
| `protected` constructor | Only subclasses build these. Enforced by Topic 03's access rules. |
| Always takes a `Throwable cause` | Even if callers pass `null`. Having the parameter means nobody can *forget* it — they have to actively pass null, which shows up in review. |

### Suppressing the stack trace for control-flow exceptions

```java
public final class DuplicateOrderException extends OrderflowException {
    public DuplicateOrderException(String idempotencyKey) {
        super(ErrorCode.DUPLICATE_ORDER,
              "idempotency key already used: " + idempotencyKey,
              null,
              /* enableSuppression */ false,
              /* writableStackTrace */ false);
    }
}
```

The four-argument `RuntimeException` constructor lets you turn off stack-trace
capture. That capture is the expensive part of an exception (Topic 08).

**When this is right:** an exception thrown thousands of times per second where the
stack is always the same and nobody will ever read it — a duplicate-request rejection
on a hot endpoint.

**When it is wrong:** anywhere you might need to debug it. Losing the stack trace is
losing the evidence. This is an optimisation you apply *after* a profile shows
exception construction is hot, not before. Topic 77 is how you would know.

### Sealed result types as the alternative to throwing

```java
public sealed interface ChargeOutcome {

    record Captured(String transactionId, long amountMinor) implements ChargeOutcome { }
    record Declined(DeclineReason reason)                   implements ChargeOutcome { }
    record Retryable(String providerRef, long retryAfterMs) implements ChargeOutcome { }
}
```

```java
switch (gateway.charge(order)) {
    case ChargeOutcome.Captured c  -> orders.markPaid(order.id(), c.transactionId());
    case ChargeOutcome.Declined d  -> orders.markDeclined(order.id(), d.reason());
    case ChargeOutcome.Retryable r -> retryQueue.schedule(order.id(), r.retryAfterMs());
}
```

| Bit | What it means |
|---|---|
| `sealed interface` | The set of outcomes is closed. Nothing outside this file can add a fourth. Topic 28. |
| Nested records | Each variant is a data carrier with generated `equals`/`hashCode`. Topic 27. |
| `switch` with no `default` | Because the type is sealed, the compiler checks exhaustiveness. Add a fourth variant and **every switch in the codebase becomes a compile error** — which is exactly what you want. Topic 29. |

This is your TypeScript discriminated union, with compiler-checked exhaustiveness. It
is the closest true analogue in the language, and for expected business outcomes it is
strictly better than exceptions: no stack capture, no control-flow jump, no
possibility of it being swallowed.

---

## Example 1 — minimal

The same failure, modelled three ways. The point is to feel the cost of each at the
call site.

```java
import java.util.*;

public class ThreeWays {

    record Wallet(String userId, long balanceMinor) { }

    // --- Way 1: checked exception -------------------------------------------
    static class InsufficientFundsCheckedException extends Exception {
        InsufficientFundsCheckedException(long shortfall) {
            super("short by " + shortfall);
        }
    }

    static long debitChecked(Wallet w, long amount) throws InsufficientFundsCheckedException {
        if (w.balanceMinor() < amount) {
            throw new InsufficientFundsCheckedException(amount - w.balanceMinor());
        }
        return w.balanceMinor() - amount;
    }

    // --- Way 2: unchecked exception -----------------------------------------
    static class InsufficientFundsException extends RuntimeException {
        InsufficientFundsException(long shortfall) { super("short by " + shortfall); }
    }

    static long debitUnchecked(Wallet w, long amount) {
        if (w.balanceMinor() < amount) {
            throw new InsufficientFundsException(amount - w.balanceMinor());
        }
        return w.balanceMinor() - amount;
    }

    // --- Way 3: return value ------------------------------------------------
    sealed interface DebitResult { }
    record Debited(long newBalanceMinor)   implements DebitResult { }
    record Insufficient(long shortfallMinor) implements DebitResult { }

    static DebitResult debitResult(Wallet w, long amount) {
        return w.balanceMinor() >= amount
                ? new Debited(w.balanceMinor() - amount)
                : new Insufficient(amount - w.balanceMinor());
    }

    public static void main(String[] args) {
        List<Wallet> wallets = List.of(
                new Wallet("u1", 5_000L), new Wallet("u2", 100L));

        // Way 1 inside a stream: DOES NOT COMPILE. Try it.
        // var out = wallets.stream().map(w -> debitChecked(w, 1_000L)).toList();

        // Way 2 inside a stream: compiles, but one bad wallet aborts the whole batch.
        // var out = wallets.stream().map(w -> debitUnchecked(w, 1_000L)).toList();

        // Way 3: compiles, composes, and handles every wallet.
        List<DebitResult> results = wallets.stream()
                .map(w -> debitResult(w, 1_000L))
                .toList();

        long succeeded = results.stream().filter(r -> r instanceof Debited).count();
        System.out.println(succeeded + " of " + results.size() + " debited");
    }
}
```

Uncomment the first stream line. The compile error is the single strongest practical
argument against checked exceptions in modern Java, and it takes ten seconds to
produce.

---

## Example 2 — production scenario

The `orderflow` domain exception hierarchy — designed so that it survives contact with
a REST layer, a Kafka consumer, and a retry policy, all of which will consume it.

```java
package com.orderflow.error;

/**
 * Stable, machine-readable error identity. Clients switch on this.
 * The enum name is part of the public API contract: never rename a constant,
 * only add new ones. The HTTP mapping lives at the edge (Topic 46), NOT here.
 */
public enum ErrorCode {
    ORDER_NOT_FOUND,
    DUPLICATE_ORDER,
    INSUFFICIENT_STOCK,
    INSUFFICIENT_FUNDS,
    PAYMENT_DECLINED,
    PAYMENT_PROVIDER_UNAVAILABLE,
    INVENTORY_RESERVATION_CONFLICT,
    INVALID_ORDER_STATE
}
```

> Each `public` type below is its own file in `com.orderflow.error`; they are shown
> in one block for reading. Java allows only one public type per source file.

```java
package com.orderflow.error;

/** Base of every orderflow domain failure. Unchecked by default and by design. */
public abstract class OrderflowException extends RuntimeException {

    private final ErrorCode code;

    protected OrderflowException(ErrorCode code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public ErrorCode code() { return code; }
}

/** Marker: the operation may succeed if attempted again, unchanged. */
public interface Retryable {
    long suggestedRetryAfterMillis();
}
```

```java
package com.orderflow.error;

public final class OrderNotFoundException extends OrderflowException {
    private final long orderId;
    public OrderNotFoundException(long orderId) {
        super(ErrorCode.ORDER_NOT_FOUND, "order " + orderId + " not found", null);
        this.orderId = orderId;
    }
    public long orderId() { return orderId; }
}

public final class InsufficientStockException extends OrderflowException {
    private final String sku;
    private final int requested;
    private final int available;
    public InsufficientStockException(String sku, int requested, int available) {
        super(ErrorCode.INSUFFICIENT_STOCK,
              "sku " + sku + ": requested " + requested + ", available " + available,
              null);
        this.sku = sku; this.requested = requested; this.available = available;
    }
    public String sku()     { return sku; }
    public int requested()  { return requested; }
    public int available()  { return available; }
}

public final class PaymentProviderUnavailableException
        extends OrderflowException implements Retryable {

    private final long retryAfterMillis;

    public PaymentProviderUnavailableException(String provider, long retryAfterMillis,
                                               Throwable cause) {
        super(ErrorCode.PAYMENT_PROVIDER_UNAVAILABLE,
              "payment provider " + provider + " unavailable", cause);
        this.retryAfterMillis = retryAfterMillis;
    }

    @Override public long suggestedRetryAfterMillis() { return retryAfterMillis; }
}
```

And the one place the whole hierarchy is consumed — a single mapping table, at the
edge, in one file:

```java
package com.orderflow.api;

import com.orderflow.error.*;
import java.util.*;

/** The ONLY place ErrorCode meets HTTP. Topic 46 turns this into @ControllerAdvice. */
public final class ErrorCodeHttpMapping {

    private static final Map<ErrorCode, Integer> STATUS = Map.of(
            ErrorCode.ORDER_NOT_FOUND,               404,
            ErrorCode.DUPLICATE_ORDER,              409,
            ErrorCode.INSUFFICIENT_STOCK,           409,
            ErrorCode.INSUFFICIENT_FUNDS,           402,
            ErrorCode.PAYMENT_DECLINED,             402,
            ErrorCode.PAYMENT_PROVIDER_UNAVAILABLE, 503,
            ErrorCode.INVENTORY_RESERVATION_CONFLICT, 409,
            ErrorCode.INVALID_ORDER_STATE,          422);

    public static int statusFor(OrderflowException e) {
        return STATUS.getOrDefault(e.code(), 500);
    }

    private ErrorCodeHttpMapping() { }
}
```

Now read the design decisions, because each one is a thing you will be asked to
defend:

| Decision | Why |
|---|---|
| Everything unchecked | No caller of `OrderService.place()` has an alternative action for "provider unavailable" except retry, and retry is a **policy**, applied by an interceptor, not by every call site. |
| `ErrorCode` enum separate from the message | The message will change (someone will improve the wording). The code is a client contract. Never couple them. |
| Exception fields carry **structured** data (`sku`, `requested`, `available`) | So the API can return a useful body without regex-parsing a message string. Message parsing is a bug generator. |
| No HTTP status in the exception | The same exception is thrown in a Kafka consumer and a batch job, neither of which has an HTTP status. Coupling the domain to one transport is how you end up with `HttpException` in a scheduled job. |
| `Retryable` as a separate **interface**, not a base class | Retryability is orthogonal to which failure it is. A retry interceptor asks `if (e instanceof Retryable r)` and does not need to know about payments at all. Java has single inheritance; interfaces are how you attach an orthogonal axis. |
| No PII in messages | `"order 4829113 not found"` is fine. `"card 4111-1111-1111-1111 declined for alice@example.com"` ends up in a log aggregator with a 90-day retention and a compliance obligation you did not sign up for. |
| Every exception is `final` | Nobody subclasses your leaf types and breaks the mapping table. |

### The one checked exception in `orderflow`, and its justification

```java
package com.orderflow.settlement;

/**
 * CHECKED, deliberately.
 *
 * Alternative action, stated in one sentence: the caller writes this line to the
 * rejects file and continues processing the remaining 2 million lines.
 *
 * If that sentence ever stops being true, make this unchecked.
 */
public final class UnparseableSettlementLineException extends Exception {
    private final int lineNumber;
    public UnparseableSettlementLineException(int lineNumber, String message, Throwable cause) {
        super("line " + lineNumber + ": " + message, cause);
        this.lineNumber = lineNumber;
    }
    public int lineNumber() { return lineNumber; }
}
```

One checked exception in the entire service, with the justification written down next
to it. That is what a defensible position looks like.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — the swallowed exception

**Wrong:**
```java
public void reserveInventory(long orderId, List<OrderLine> lines) {
    for (OrderLine line : lines) {
        try {
            inventory.reserve(line.sku(), line.quantity());
        } catch (Exception e) {
            log.warn("could not reserve {}", line.sku());   // <-- the defect
        }
    }
}
```

**Exact symptom — and this is the important part, because there is no exception
anywhere:** the order is created with status `CONFIRMED`. The customer gets a
confirmation email. The warehouse pick-list is generated from
`inventory_reservations`, which has no rows for this order. The order is never
shipped. You discover it 11 days later from a customer-service ticket, or from this
query:

```sql
SELECT count(*)
  FROM orders o
  LEFT JOIN inventory_reservations r ON r.order_id = o.id
 WHERE o.status = 'CONFIRMED' AND r.order_id IS NULL;
```

The metric that would have caught it — `inventory.reservation.failures` — does not
exist, because nothing throws. The log line exists but is one of 40,000 `WARN` lines
that day.

**Root cause:** the `catch` converts a failure into a success. Control flow continues
as though the reservation happened. The caller has no way to know it did not, because
the method returns `void` and does not throw.

**Fix:** decide, explicitly, and make the decision visible.
```java
public ReservationResult reserveInventory(long orderId, List<OrderLine> lines) {
    List<String> failed = new ArrayList<>();
    for (OrderLine line : lines) {
        try {
            inventory.reserve(line.sku(), line.quantity());
        } catch (InsufficientStockException e) {
            failed.add(line.sku());                       // an expected outcome
        }
    }
    if (!failed.isEmpty()) {
        metrics.increment("inventory.reservation.partial");
        throw new PartialReservationException(orderId, failed);   // caller MUST deal with it
    }
    return ReservationResult.complete(orderId);
}
```

This is the failure drill below. You will produce the symptom on purpose.

---

### Trap 2 — `throws Exception` on an interface

**Wrong:**
```java
public interface PaymentGateway {
    Payment charge(Order order) throws Exception;
}
```

**Exact symptom:** every call site is forced into `catch (Exception e)`. Six months
later, an `InterruptedException`, a `NullPointerException` from a bug in your own
mapping code, and a genuine gateway timeout are all caught by the same block and
logged with the same message. Your payment-failure dashboard shows a spike; nobody can
tell whether it is the provider or your own NPE. Mean time to diagnosis goes from
minutes to hours, permanently.

**Root cause:** `throws Exception` is the widest possible declaration. It erases every
distinction below it and forces the widest possible catch. And because overrides may
only *narrow* the throws clause (Topic 08), every implementation inherits the damage —
you cannot fix it in one place.

**Fix:** declare nothing, and throw specific unchecked domain exceptions.
```java
public interface PaymentGateway {
    /**
     * @throws PaymentProviderUnavailableException transport or 5xx failure (Retryable)
     * @throws PaymentDeclinedException            the provider refused the charge
     */
    Payment charge(Order order);
}
```
Javadoc `@throws` on unchecked exceptions is the right way to document this: it
informs without obliging.

---

### Trap 3 — the wrapping onion

**Wrong:** each layer wraps the previous one in its own exception type.
```java
catch (SQLException e)              { throw new DaoException(e); }
catch (DaoException e)              { throw new RepositoryException(e); }
catch (RepositoryException e)       { throw new ServiceException(e); }
catch (ServiceException e)          { throw new OrderProcessingException(e); }
```

**Exact symptom:** a 200-line stack trace with four `Caused by:` sections, where the
first three carry no information the fourth did not already have. On-call engineers
scroll to the bottom every time. Worse, the outermost message is
`OrderProcessingException: processing failed`, so your alerting groups every distinct
database problem into one bucket, and your error-rate dashboard shows one line where
it should show five.

**Root cause:** wrapping is being used as a layering ritual rather than as
*translation*. A wrapper is only worth it if it **adds information** or **changes the
abstraction level** in a way a caller will act on.

**Fix:** wrap once, at the boundary where the abstraction genuinely changes, and add
context the caller does not have.
```java
catch (SQLException e) {
    throw new OrderPersistenceException(
            "saving order " + order.id() + " with " + order.lines().size() + " lines", e);
}
```
One wrapper. The order ID and line count are new information. Everything below is
still reachable through `getCause()`.

---

### Trap 4 — checked exception for an expected business outcome

**Wrong:**
```java
public Payment charge(Order o) throws CardDeclinedException { ... }
```

**Exact symptom:** grep the codebase six months later and you find four call sites.
One handles it properly. Two do `catch (CardDeclinedException e) { log.warn(...); }`.
One does `catch (CardDeclinedException e) { }` with a `// TODO` from a sprint that
ended a year ago. The business number: the payment-retry funnel shows 5% of customers
dropping at checkout with no recorded reason, because two of the four paths never
record a decline.

**Root cause:** a card decline is not exceptional — it is one of the normal outcomes
of charging a card, happening several percent of the time. Forcing a `catch` at every
call site guarantees that the least-careful call site sets the quality bar.

**Fix:** make it a return value.
```java
public ChargeOutcome charge(Order o);   // sealed: Captured | Declined | Retryable
```
Now the compiler requires every call site to consider `Declined` — via exhaustiveness
in a `switch`, not via a `catch` they can leave empty. Same enforcement, no escape
hatch.

---

### Trap 5 — the exception carries PII or an entity

**Wrong:**
```java
throw new PaymentDeclinedException("declined for " + customer + " card " + cardNumber);
```
or, subtler:
```java
throw new InvalidOrderStateException("bad order: " + order);   // order.toString() dumps everything
```

**Exact symptom:** the message goes into Logback, into your log aggregator, into a
90-day-retention index, and — if your `@ControllerAdvice` is naive — into the HTTP
response body. Two concrete costs: a PCI-DSS finding on the card number, and a GDPR
erasure request that you cannot satisfy because the customer's email is embedded in
free-text log lines across 90 days of indexes.

The subtler version is worse in a different way: `order.toString()` on a JPA entity
touches lazy associations. In a `catch` block, after the transaction has ended, that
is a `LazyInitializationException` thrown from inside your error handler — which
replaces the real exception (Topic 49).

**Root cause:** exception messages are treated as debug scratch space. They are not:
they are an output channel with the same compliance obligations as any other.

**Fix:** identifiers only, structured fields for the rest.
```java
throw new PaymentDeclinedException(order.id(), DeclineReason.INSUFFICIENT_FUNDS);
```
The order ID lets an engineer find everything else in the database. Nothing sensitive
crosses the boundary. And never interpolate an entity into a message — use its ID.

---

## Hands-on proof

Commands **you** run. I have no JVM and will not invent output.

### Setup

```bash
mkdir -p ~/java-lab/09 && cd ~/java-lab/09
java --version
```

### Proof 1 — checked exceptions do not compose through lambdas

`LambdaCompose.java`:
```java
import java.util.*;
import java.util.function.*;

public class LambdaCompose {

    static class NotFoundException extends Exception {
        NotFoundException(long id) { super("no order " + id); }
    }

    static String loadChecked(long id) throws NotFoundException {
        if (id < 0) throw new NotFoundException(id);
        return "order-" + id;
    }

    static String loadUnchecked(long id) {
        if (id < 0) throw new IllegalArgumentException("no order " + id);
        return "order-" + id;
    }

    public static void main(String[] args) {
        List<Long> ids = List.of(1L, 2L, 3L);

        // (A) uncomment this line:
        // var a = ids.stream().map(LambdaCompose::loadChecked).toList();

        var b = ids.stream().map(LambdaCompose::loadUnchecked).toList();
        System.out.println(b);
    }
}
```

```bash
javac LambdaCompose.java          # with (A) commented: compiles
# now uncomment (A) and:
javac LambdaCompose.java
```

**What to look for:** the exact compile error on line (A).

| What you see | What it means |
|---|---|
| `error: incompatible thrown types NotFoundException in method reference` (wording varies) | Expected. `Function.apply` has no `throws` clause, so no checked exception can escape a lambda body. This is a *structural* incompatibility with every post-Java-8 API. |
| It compiles | You did not uncomment the line, or you wrapped the call in a try. Check. |
| A different error mentioning `Supplier` or `Consumer` | You changed the stream operation; the shape of the error is the same for all of them. |

**How to read it:** there is no flag, no setting, no library that fixes this cleanly.
The standard workarounds are a wrapper lambda with a try/catch (verbose, at every use)
or a "sneaky throws" generic trick (which defeats the entire point of checked
exceptions). Neither is good. **That is the argument**, and you just produced it in
two commands.

### Proof 2 — the cost of a stack trace

`TraceCost.java`:
```java
public class TraceCost {

    static class WithTrace extends RuntimeException {
        WithTrace() { super("boom"); }
    }
    static class NoTrace extends RuntimeException {
        NoTrace() { super("boom", null, false, false); }
    }

    static int deep(int n, boolean withTrace) {
        if (n == 0) { throw withTrace ? new WithTrace() : new NoTrace(); }
        return deep(n - 1, withTrace);
    }

    static long run(boolean withTrace, int iterations) {
        long start = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            try { deep(50, withTrace); } catch (RuntimeException ignored) { }
        }
        return System.nanoTime() - start;
    }

    public static void main(String[] args) {
        for (int round = 0; round < 5; round++) {
            System.out.printf("round %d  withTrace=%d ns  noTrace=%d ns%n",
                    round, run(true, 100_000), run(false, 100_000));
        }
    }
}
```

```bash
java TraceCost.java
```

**What to look for:** whether the `withTrace` number is meaningfully larger, and
whether both numbers change across rounds.

| What you see | What it means |
|---|---|
| `withTrace` clearly larger, and stable after round 1 or 2 | The stack-trace capture is the dominant cost. The 50-frame depth is what makes it visible. |
| Both numbers fall sharply between round 0 and round 2 | JIT warm-up. This is exactly why the number from round 0 is meaningless. |
| The numbers are close, or noisy round to round | **Do not conclude anything.** A wall-clock loop is not a benchmark. |

> **This is a deliberately unreliable measurement.** The JIT can inline, the recursion
> may be optimised, and there is no warm-up discipline. Topic 77 (JMH) is where you
> learn to do this properly, and you will come back and redo this. What the loop is
> good for is showing you the *shape* — that trace capture is not free — not for
> giving you a number you would put in a design document.

### Proof 3 — the wrapping onion, seen

`Onion.java`:
```java
public class Onion {
    static class L1 extends RuntimeException { L1(Throwable c) { super("l1", c); } }
    static class L2 extends RuntimeException { L2(Throwable c) { super("l2", c); } }
    static class L3 extends RuntimeException { L3(Throwable c) { super("l3", c); } }

    public static void main(String[] args) {
        try {
            try {
                try {
                    throw new IllegalStateException("THE ACTUAL PROBLEM");
                } catch (RuntimeException e) { throw new L1(e); }
            } catch (RuntimeException e) { throw new L2(e); }
        } catch (RuntimeException e) {
            RuntimeException wrapped = new L3(e);
            wrapped.printStackTrace(System.out);

            Throwable root = wrapped;
            while (root.getCause() != null) root = root.getCause();
            System.out.println("ROOT CAUSE: " + root);
        }
    }
}
```

```bash
java Onion.java | tee onion.txt
wc -l onion.txt
grep -c "Caused by" onion.txt
```

**What to look for:**

| What you see | What it means |
|---|---|
| Three `Caused by:` lines and a long trace | Each wrapper added a frame set and no information. Count the lines an on-call engineer must scroll past to reach the truth. |
| `ROOT CAUSE: java.lang.IllegalStateException: THE ACTUAL PROBLEM` | The last cause in the chain is what you actually needed. Write that unwrap loop once; you will use it in a `@ControllerAdvice`. |
| Fewer `Caused by:` lines than wrappers | You dropped a cause somewhere. Find it — that is Topic 08's Trap 5. |

### Proof 4 — sealed exhaustiveness is compiler-enforced

`Exhaustive.java`:
```java
public class Exhaustive {

    sealed interface ChargeOutcome { }
    record Captured(String txId)  implements ChargeOutcome { }
    record Declined(String code)  implements ChargeOutcome { }
    record Retryable(long afterMs) implements ChargeOutcome { }

    static String describe(ChargeOutcome o) {
        return switch (o) {
            case Captured c  -> "captured " + c.txId();
            case Declined d  -> "declined " + d.code();
            case Retryable r -> "retry in " + r.afterMs();
        };
    }

    public static void main(String[] args) {
        System.out.println(describe(new Declined("51")));
    }
}
```

```bash
java Exhaustive.java
```

Now add a fourth variant — `record Reversed(String txId) implements ChargeOutcome { }`
— and recompile **without** touching `describe`.

| What you see | What it means |
|---|---|
| `error: the switch expression does not cover all possible input values` | Exactly the guarantee you get from TypeScript discriminated unions, enforced by javac. Adding an outcome forces every handler to be revisited. A `catch` block gives you nothing comparable. |
| It compiles | You left a `default` clause in. Remove it — `default` on a sealed switch throws away the entire benefit. |

**How to read it:** compare this to adding a new checked exception to a method. Both
break callers at compile time. But the sealed version breaks them at the point where
the *decision* is made, with a message naming exactly what is unhandled, and cannot be
silenced by an empty block.

---

## Failure drill [BONUS]

**What this proves:** that "log and continue" is a defect, not a style preference —
because the failure becomes invisible in every channel except a business number, days
later.

This drill runs as a plain Java program. No database, no Spring, no Docker. The
`orderflow` service does not exist until Topic 35; the *shape* of the bug does.

### Setup

```bash
mkdir -p ~/java-lab/09/drill && cd ~/java-lab/09/drill
```

`SwallowDrill.java`:
```java
import java.util.*;
import java.util.concurrent.ThreadLocalRandom;

/**
 * Simulates orderflow's order-placement path with an inventory reservation that
 * intermittently fails. Run mode "swallow" or "propagate".
 */
public class SwallowDrill {

    // ---- fake stores, standing in for two database tables --------------------
    static final Map<Long, String>      ORDERS       = new LinkedHashMap<>();  // orders
    static final Map<Long, List<String>> RESERVATIONS = new LinkedHashMap<>(); // inventory_reservations

    // ---- a "metric" nobody is publishing yet ---------------------------------
    static long reservationFailures = 0;

    static class InsufficientStockException extends RuntimeException {
        InsufficientStockException(String sku) { super("no stock for " + sku); }
    }

    /** Fails for roughly 3% of reservations — a hot SKU running out. */
    static void reserve(long orderId, String sku) {
        if (ThreadLocalRandom.current().nextInt(100) < 3) {
            throw new InsufficientStockException(sku);
        }
        RESERVATIONS.computeIfAbsent(orderId, k -> new ArrayList<>()).add(sku);
    }

    // ---- THE DEFECT ----------------------------------------------------------
    static void placeOrderSwallowing(long orderId, List<String> skus) {
        for (String sku : skus) {
            try {
                reserve(orderId, sku);
            } catch (InsufficientStockException e) {
                System.err.println("WARN  could not reserve " + sku + " for order " + orderId);
                // and we carry on. Nothing above this line will ever know.
            }
        }
        ORDERS.put(orderId, "CONFIRMED");     // <- the lie
    }

    // ---- THE FIX -------------------------------------------------------------
    static void placeOrderPropagating(long orderId, List<String> skus) {
        for (String sku : skus) {
            try {
                reserve(orderId, sku);
            } catch (InsufficientStockException e) {
                reservationFailures++;                       // an actual metric
                ORDERS.put(orderId, "RESERVATION_FAILED");   // an actual state
                throw e;                                     // an actual signal
            }
        }
        ORDERS.put(orderId, "CONFIRMED");
    }

    public static void main(String[] args) {
        String mode = args.length > 0 ? args[0] : "swallow";
        int orderCount = args.length > 1 ? Integer.parseInt(args[1]) : 2000;

        int rejectedAtApi = 0;

        for (long id = 1; id <= orderCount; id++) {
            List<String> skus = List.of("SKU-4471", "SKU-9002", "SKU-1180");
            try {
                if (mode.equals("swallow")) placeOrderSwallowing(id, skus);
                else                        placeOrderPropagating(id, skus);
            } catch (RuntimeException e) {
                rejectedAtApi++;      // the caller — an HTTP layer — returns 409
            }
        }

        // ---- the ONLY place the defect is visible: a business reconciliation ----
        long confirmed = ORDERS.values().stream().filter("CONFIRMED"::equals).count();
        long confirmedWithoutFullReservation = ORDERS.entrySet().stream()
                .filter(e -> e.getValue().equals("CONFIRMED"))
                .filter(e -> RESERVATIONS.getOrDefault(e.getKey(), List.of()).size() < 3)
                .count();

        System.out.println("mode                            : " + mode);
        System.out.println("orders confirmed                : " + confirmed);
        System.out.println("rejected at API (client saw 409): " + rejectedAtApi);
        System.out.println("reservationFailures metric      : " + reservationFailures);
        System.out.println("CONFIRMED but under-reserved    : " + confirmedWithoutFullReservation);
    }
}
```

### Run it

```bash
# 1. The defect. Send stderr to a log file, the way production does.
java SwallowDrill.java swallow 2000 2> swallow.log

# 2. Count the "alerts" nobody read.
wc -l swallow.log

# 3. The fix.
java SwallowDrill.java propagate 2000 2> propagate.log
wc -l propagate.log
```

### What to capture

Write these five numbers down for both runs:

1. `orders confirmed`
2. `rejected at API (client saw 409)`
3. `reservationFailures metric`
4. `CONFIRMED but under-reserved`
5. the line count of the `.log` file

### How to read it

| Observation | What it means |
|---|---|
| **swallow:** `CONFIRMED but under-reserved` is greater than zero | This is the entire drill. Every one of those is an order the customer was told is confirmed, that the warehouse will never pick. There is no exception, no error metric, no failed request — only a divergence between two tables. |
| **swallow:** `rejected at API` is `0` | The API returned 200 for every request, including the broken ones. Your HTTP error rate is a flat, healthy 0%. Your dashboard is green. |
| **swallow:** `reservationFailures` is `0` | The metric that would have alerted you was never incremented, because incrementing it was not the compiler's problem — swallowing was. |
| **swallow:** `swallow.log` has hundreds of lines | The warnings existed. They were emitted. Nobody read them, because they are indistinguishable from the other 40,000 WARN lines that day. **A log line is not an error signal.** |
| **propagate:** `CONFIRMED but under-reserved` is `0` | Correct by construction: an order is either fully reserved and CONFIRMED, or it is not CONFIRMED. The invariant now holds. |
| **propagate:** `rejected at API` is greater than zero | The failure is now *visible in the channel that is monitored* — the HTTP error rate. On-call finds out in minutes, not days. |
| **propagate:** `reservationFailures` matches `rejected at API` | You have a metric that tracks the real thing, and it agrees with the transport-level signal. Two independent confirmations. |
| Your numbers vary between runs | Expected — the 3% failure is random. The *sign* of each number is what matters, not the value. |

### The production version of that reconciliation query

The whole point is that in a real service, the only way to find this is a query like
the one below — which someone has to think to write, days after the damage:

```sql
SELECT o.id, o.created_at, count(r.id) AS reserved_lines, count(l.id) AS ordered_lines
  FROM orders o
  JOIN order_lines l            ON l.order_id = o.id
  LEFT JOIN inventory_reservations r ON r.order_id = o.id
 WHERE o.status = 'CONFIRMED'
 GROUP BY o.id, o.created_at
HAVING count(r.id) < count(l.id)
 ORDER BY o.created_at;
```

You already know how to write that query. What this drill teaches is that **needing to
write it is the symptom**, and the cause is one `catch` block that turned a failure
into a success three weeks earlier.

### What the fix proves

Three things, precisely:

1. **A caught exception with no rethrow is a control-flow decision, not logging.** The
   method continued and reported success. That is a lie the caller cannot detect.
2. **Logging is not error handling.** The information was emitted into a channel with
   no threshold, no alert, and a 40,000-line/day noise floor. If it is worth catching,
   it is worth a metric and a state change.
3. **The correct question is never "how do I handle this here?" but "who has an
   alternative action, and how do I get the failure to them?"** In this case: nobody
   inside the loop does, so it propagates to the API boundary, which has exactly one:
   return 409 and do not confirm the order.

### Extension, if you want the harder version

Change `placeOrderSwallowing` to swallow only 1 SKU in 500 instead of 3 in 100. Run
100,000 orders. Now the divergence is ~0.2% — small enough that nobody notices in a
weekly report, large enough to be hundreds of orders a month. That is the version that
actually happens in production, and it is the reason "we would have noticed" is not a
defence.

---

## Practice exercises

### 1 — Easy: the justification test

Take these eight failures from `orderflow`. For each, write **one sentence** naming
the caller's alternative action. Then classify: checked exception, unchecked
exception, or return value. If your sentence contains "log", "report", or "decide what
to do", it is not an alternative action — mark that one as unchecked.

1. Order ID does not exist.
2. Wallet balance is 300 and the order costs 4500.
3. The payment provider returns HTTP 503.
4. The card is declined with code 51 (insufficient funds at the bank).
5. The idempotency key on an incoming request has been seen before.
6. A JSON request body has `quantity: -3`.
7. The settlement CSV has a line with a malformed timestamp, at line 918,443 of 2M.
8. The inventory row version changed between read and write (optimistic lock).

Then: pick the two you found hardest, and write the argument for the *other* answer.

### 2 — Medium: combines Topics 05, 06, 07 and 08

Build a small `Retry` utility with this signature:

```java
public static <T> T withRetry(int maxAttempts,
                              Supplier<? extends T> action,
                              Predicate<? super RuntimeException> retryable);
```

Requirements:
- Justify each wildcard against Topic 07's PECS rule, in a comment.
- On the final failed attempt, rethrow the last exception with all earlier attempts'
  exceptions attached via `addSuppressed` (Topic 08). Prove it by printing
  `getSuppressed().length`.
- It must work with the `Retryable` marker interface from Example 2, so a caller can
  pass `e -> e instanceof Retryable`.
- Explain, in a comment, why the `Supplier` cannot throw a checked exception and what
  that forces on callers who have one. (Topic 06 will help you see why you cannot
  simply add a type parameter for the exception and have it work through erasure —
  actually you *can* declare `<T, E extends Exception>`; explain precisely what you
  gain and what still breaks.)
- Write a test proving that a non-retryable exception is thrown on the **first**
  attempt with no retries and no suppressed entries.

### 3 — Hard: production simulation on `orderflow`

**Part A — design the hierarchy.** Build the full `com.orderflow.error` package from
Example 2. Requirements:
- At least six leaf exceptions, all `final`.
- Exactly one checked exception in the whole package, with its justification sentence
  written as a Javadoc comment. If you cannot write the sentence, you have zero.
- A `Retryable` interface and at least two exceptions that implement it.
- No HTTP status codes anywhere in the package.

**Part B — the boundary.** Write a `GlobalErrorTranslator` class (this becomes your
`@ControllerAdvice` at Topic 46) that turns any `Throwable` into a record:
```java
record ApiError(int status, String code, String detail, String traceId) { }
```
Rules it must enforce:
- `OrderflowException` maps via `ErrorCode`.
- Anything else maps to 500 with code `INTERNAL` and a **generic** detail string —
  never `e.getMessage()`, because that may contain internals.
- The full exception, with cause chain, is logged once with the `traceId`.
- The `detail` field must never contain an email address, a card number, or a class
  name. Write a test that asserts this against a deliberately nasty exception message.

**Part C — run the failure drill against your own code.** Take the swallow drill
above and re-implement it using your Part A hierarchy and your Part B translator.
Produce the same table of five numbers. Then answer: which of your design decisions
made the swallowed version *harder* to write? (A good answer names at least one:
returning a value instead of `void`, a non-empty constructor, an enum code that would
have to be invented.)

**Part D — argue against yourself.** You have now built an all-unchecked hierarchy.
Write the strongest case that `InsufficientStockException` should be **checked** —
naming the specific caller and its specific alternative action. Then say what evidence
from your own drill supports or refutes it. If you conclude it should be checked,
change it, and report what broke.

---

## Interview questions

### Q1 — "What do you think of checked exceptions?"

**Mid-level answer:** "They force you to handle errors, which is good in theory, but
in practice people wrap everything in `RuntimeException`. Most teams avoid them."

**Senior answer:** "I use them rarely and deliberately, and I can say exactly when.
The test is whether the caller has a genuine alternative action that is not 'log it'.
A malformed line in a two-million-row import qualifies — the caller writes it to a
rejects file and continues. 'The database is down' does not; nobody up that stack can
do anything but fail. Three concrete problems push the default to unchecked: they
don't compose through lambdas, because `Function.apply` has no `throws` clause, so
every stream becomes a wrapping exercise; they leak implementation into signatures, so
`throws SQLException` on a repository interface tells every caller which database
you use; and under deadline pressure the compiler's demand for *something* gets
satisfied with an empty catch, which is strictly worse than no exception at all.
Spring made the whole data-access hierarchy unchecked for exactly those reasons, and
that is a decision made by people who watched a decade of swallowed `SQLException`s."

**What separates them:** having a decision *test*, three named consequences rather
than a vibe, and citing the Spring precedent as evidence rather than authority.

**Follow-up:** "Give me a case where you'd genuinely use checked." A weak answer is
"IOException". A strong one is narrow, record-shaped, and names the alternative
action.

---

### Q2 — "A junior submits `catch (Exception e) { log.warn("failed", e); }`. Walk me through the review."

**Mid-level answer:** "I'd say don't swallow exceptions — either rethrow or handle it
properly."

**Senior answer:** "I'd start with what it does to the caller, not with the rule. That
catch converts a failure into a success: the method returns normally and the caller
has no way to detect anything went wrong. So if this is inside order placement, we
create a `CONFIRMED` order with no inventory reservation. The observable consequence
is that there is *no* observable consequence — HTTP error rate stays at zero, no
metric fires, and the only evidence is a divergence between the orders table and the
reservations table that someone finds days later from a support ticket. The log line
is not a signal; it is one of forty thousand WARN lines that day. So the review
comment is: name the alternative action. If there is one, take it and record it in a
metric and a state change. If there isn't, let it propagate to a boundary that has
one. And if this genuinely is a tolerable partial failure, the method must return
something that says so — a result type, not `void`."

**What separates them:** describing the *invisibility* as the harm, distinguishing a
log from a signal, and offering three concrete resolutions rather than one rule.

**Follow-up:** "When is an empty catch block acceptable?" There are a few — a
best-effort `close()` in a cleanup path, a parse attempt in a
try-this-then-that chain — and every one of them needs a comment saying why.

---

### Q3 — "Why did Spring make `DataAccessException` unchecked?"

**Mid-level answer:** "Because checked exceptions are annoying and Spring wanted
cleaner code."

**Senior answer:** "Two reasons, and the second is the important one. First,
portability: `SQLException` is a single type carrying a vendor-specific error code, so
you cannot catch 'duplicate key' portably — you have to compare integers you looked up
in a manual. Spring translates that into a *hierarchy* —
`DuplicateKeyException`, `DeadlockLoserDataAccessException`,
`CannotAcquireLockException` — so you can catch the concept rather than the vendor
code. Second, and this is the design argument: for the overwhelming majority of
data-access failures, no caller has an alternative action. Forcing `throws
SQLException` up through the service layer buys nothing and leaks the persistence
technology into every signature. Making it unchecked means the failure travels to a
boundary that *does* have a policy — a transaction rollback, a retry, a 500 — without
polluting anything in between. Note what they did keep: the exception is still
specific, so when there *is* an alternative action, like retrying on a deadlock, you
can still catch precisely."

**What separates them:** knowing that translation into a *hierarchy* was the primary
value, and noticing that unchecked did not mean less specific.

**Follow-up:** "Does that argument apply to your own domain exceptions?"

---

### Q4 — "How do you decide between throwing an exception and returning a result type?"

**Mid-level answer:** "Exceptions for errors, return values for normal results."

**Senior answer:** "I ask how often it happens and whether the caller must branch on
it. A card decline happens on several percent of charges — it is a normal outcome of
charging a card, not an exceptional one — so it is a sealed `ChargeOutcome` with
`Captured`, `Declined` and `Retryable` variants. The compiler then forces every call
site to consider all three via switch exhaustiveness, which is the same enforcement a
checked exception claims to give but without an empty catch block as an escape hatch.
Adding a fourth outcome later breaks every switch at compile time, which is exactly
what I want. Exceptions I keep for the paths where returning is genuinely wrong: a
violated precondition, an unrecoverable infrastructure failure, or anywhere the stack
must unwind to a boundary. There is also a cost angle — an exception captures a stack
trace on construction, so a high-frequency control-flow exception is measurably
expensive, though I would confirm that with JMH before acting on it."

**What separates them:** framing it as frequency plus caller-branching rather than
error-vs-success, and knowing that sealed exhaustiveness gives the enforcement checked
exceptions were supposed to.

**Follow-up:** "Is a sealed result type always better then?" No — it forces every
caller to handle every case immediately, which is wrong when the correct behaviour is
to unwind ten frames to a boundary. Exceptions are for propagation across layers.

---

### Q5 — "What goes in an exception message, and what does not?"

**Mid-level answer:** "Enough detail to debug it — the values involved, the ID."

**Senior answer:** "Identifiers, never payloads. An exception message travels to the
log aggregator, sits in an index with a retention policy, and — if the error handler
is naive — into the HTTP response body. So a card number in a message is a PCI
finding, and an email address is a GDPR erasure request you cannot satisfy. I put the
order ID or SKU in the message and the structured detail in *fields* on the exception,
which lets the API return something useful without regex-parsing a string. I also
never interpolate an entity: `order.toString()` on a JPA entity touches lazy
associations, and doing that inside a catch block after the transaction ended throws
`LazyInitializationException` from your error handler, which replaces the real
exception. And separately from the message I keep a stable `ErrorCode` enum, because
the message will get reworded and clients must not be parsing it."

**What separates them:** treating the message as an output channel with compliance
obligations, the entity-`toString` trap, and separating a stable machine code from a
mutable human string.

**Follow-up:** "How do you enforce that in a large codebase?" They want a test that
runs the translator against nasty inputs, plus a lint rule — not "code review".

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Kotlin removed checked exceptions from the language but runs on the JVM, so it
   calls Java methods that declare them and simply ignores the declarations. What
   does that interoperability tell you about where checked exceptions are actually
   enforced?

2. Adding a checked exception to a method breaks callers at compile time. So does
   adding a variant to a sealed interface used in exhaustive switches. Are these the
   same kind of breaking change? Which one would you rather inflict on a downstream
   team, and why?

3. You made every domain exception unchecked. A new engineer now cannot tell, from a
   signature, what a method can fail with. What do you put in place instead, and is it
   as good? Be honest about the gap.

4. The failure drill showed that a swallowed exception is invisible in every monitored
   channel. Design a control that would catch this class of bug *without* relying on
   anyone remembering to write the reconciliation query. What does it cost?

5. `writableStackTrace = false` makes an exception dramatically cheaper. Why is
   defaulting to it a bad idea, even for exceptions you are confident nobody will
   debug?

6. Spring translates `SQLException` into an unchecked hierarchy. Your own service
   could translate `IOException` from an HTTP client the same way. What would you gain,
   and at what point does the translation layer cost more than it saves?

7. Your `ErrorCode` enum is a public contract, so constants can be added but never
   renamed or removed. Ten years in, you have 60 codes, eleven of which are
   deprecated. What did you get wrong at the start, and what would you do differently?

---

## Quick reference card

### The decision procedure

```
Is it an expected outcome the caller must branch on?
    -> sealed result type (Topic 28/29). Not an exception.

Does the caller have an alternative action that is not "log it"?
    -> checked exception. Write the alternative action down next to the class.

Is the JVM or environment broken?
    -> Error. Do not throw one, do not catch one.

Everything else
    -> unchecked domain exception extending your base type.
```

### Hierarchy design rules

- One abstract base per bounded context (`OrderflowException`). Nobody throws it.
- Leaf classes are `final`.
- A stable `ErrorCode` enum, separate from the message. Add constants; never rename.
- Structured fields (`sku`, `requested`, `available`), not information hidden in a
  message string.
- Orthogonal axes (retryable, client-fault-vs-server-fault) as **interfaces**, not
  base classes.
- No HTTP status, no `ResponseEntity`, no transport concept in the domain package.
- Always a `(String, Throwable)` constructor. Always pass the cause.
- Identifiers in messages. Never PII, never an entity's `toString()`.

### Smells, and what each one means

| You see | It means |
|---|---|
| `throws Exception` on an interface | Every caller must `catch (Exception)`; all distinction is lost |
| `catch (Exception e) { log.warn(...) }` | A failure has been converted into a success. Run the drill. |
| Four `Caused by:` in one trace | Wrapping as ritual. Wrap once, at a real abstraction boundary. |
| `throw new X(e.getMessage())` | The cause was dropped. The stack below is gone. |
| A checked exception with four call sites, two of which log-and-continue | It should not be checked |
| `e.printStackTrace()` | Goes to stderr, bypasses your logging config, no correlation ID |
| `catch (Throwable t)` | Catches `Error`. See Topic 08, Trap 4. |

### API-compatibility facts

| Change | Source-compatible for callers? |
|---|---|
| Adding a checked exception to a signature | **No** — callers stop compiling |
| Removing a checked exception | Yes |
| Adding an unchecked exception | Yes (silently — which is the trade) |
| Adding a variant to a sealed interface | **No** — exhaustive switches stop compiling |
| Renaming an `ErrorCode` constant | **No** — and it breaks deployed clients at runtime |

---

## When would I use this at work?

**1. The first design review of a new service.**
Someone will propose `throws BusinessException` on the service interface. This is the
moment to ask, once, "name the caller's alternative action" — and to write the
`ErrorCode` enum before there is a single endpoint. Doing it on day one costs an hour;
retrofitting a stable error contract onto a service with live clients costs a
versioning scheme.

**2. Every code review, on one specific pattern.**
`catch` blocks that do not rethrow, do not change state, and do not increment a
metric. It is greppable, it takes ten seconds per instance, and each one you catch is
a class of silent data divergence that would otherwise be found by a customer.

**3. Post-incident, when the answer is "we had no idea".**
The failure drill's shape — orders confirmed with no reservations, a green dashboard,
a support ticket eleven days later — is one of the most common post-mortem findings in
service-oriented systems. Knowing the shape means you go looking for swallowed
exceptions early instead of spending the first two hours on infrastructure.

---

## Connected topics

**Prerequisites:**
- **08 — Exceptions fundamentals**: the hierarchy, try-with-resources, suppressed
  exceptions, and the compile-time obligation this topic argues about.
- **03 — Access modifiers**: your exception package is a published API surface;
  `protected` constructors and `final` leaves are enforcement, not decoration.
- **04 — Interfaces**: `Retryable` as an orthogonal marker works because Java has
  single inheritance but multiple interface implementation.

**This unlocks:**
- **21–24 — Lambdas and Streams**: the composition failure from Proof 1, and the
  wrapper patterns people build around it.
- **26 — Optional**: modelling expected absence without an exception.
- **27–29 — Records, sealed types, pattern matching**: the result-type alternative, in
  full, with compiler-checked exhaustiveness.
- **46 — Error handling and `ProblemDetail`**: your `ErrorCodeHttpMapping` becomes a
  `@ControllerAdvice` producing RFC 9457 responses.
- **47 — Spring Data JPA**: where you meet `DataAccessException` and Spring's
  translation layer as a working example of everything argued here.
- **54 — `@Transactional`**: the sharpest consequence of the checked/unchecked
  decision — Spring rolls back on unchecked exceptions **only** by default, so a
  checked exception thrown from a transactional method **commits**. Choosing checked
  in the wrong place is a silent data-corruption bug.
- **111 — Retries and circuit breakers**: the `Retryable` marker becomes a policy.
- **118 — Metrics**: the counter the drill proved was missing.
- **133 — Incident post-mortem**: the drill's symptom shape is a recurring finding.

---

*Java baseline 21. Nothing here is version-specific at the language level. Sealed
interfaces and pattern matching in `switch` are final in Java 21, so the result-type
approach is fully available on the baseline. The checked-exception debate itself has
been running since Java 1.0 and will outlive this document; what matters is that you
can state a decision test and defend it with consequences rather than preferences.*
