# 08 — Exceptions: Checked vs Unchecked, the Hierarchy, and try-with-resources

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

You are running a restaurant kitchen. Three completely different kinds of problem can
happen, and confusing them is how kitchens burn down.

**1. The building is on fire.**
Nobody plates a dish. Nobody "handles" this. You evacuate. Trying to carry on cooking
is not brave, it is stupid.
Java calls this an **`Error`**.

**2. You have run out of salmon.**
This is a normal, foreseeable event in a restaurant. The waiter *has* a plan: offer
the sea bass. This is worth telling the waiter about, in advance, in writing, because
the waiter genuinely has an alternative action.
Java calls this a **checked exception**.

**3. The chef put salt in the crème brûlée.**
There is no recovery. You do not write "the dessert may be salted" on the menu. You
throw it away, and then you fix the recipe. It is a defect, not an event.
Java calls this an **unchecked exception** (a `RuntimeException`).

And a fourth thing, which is not a problem but a habit:

**4. Whatever happens, the gas gets turned off.**
Dish served, dish burnt, building on fire — the gas gets turned off. In Java that is
`finally`, and the modern version that does it for you is **try-with-resources**.

Carry these four through the whole doc. Almost every exception mistake you will make
is putting a problem in the wrong one of the first three boxes.

---

## The bridge from what you know

### What transfers cleanly

`try` / `catch` / `finally` and `throw` work the way you expect. So does a custom
error type:

```ts
class PaymentDeclined extends Error {
  constructor(readonly code: string) { super(`declined: ${code}`); }
}
try { charge(); }
catch (e) { if (e instanceof PaymentDeclined) retryWithAnotherCard(); }
finally { releaseLock(); }
```

```java
class PaymentDeclinedException extends RuntimeException {
    private final String code;
    PaymentDeclinedException(String code) { super("declined: " + code); this.code = code; }
}
try { charge(); }
catch (PaymentDeclinedException e) { retryWithAnotherCard(); }
finally { releaseLock(); }
```

Two immediate differences worth noting, both small:
- Java dispatches on the exception **type** in the `catch` clause itself. There is no
  `instanceof` ladder inside one catch block. You can have several `catch` clauses,
  tried top to bottom.
- Java can only throw `Throwable` subclasses. You cannot `throw "oops"` or
  `throw {code: 42}`. Every thrown thing has a stack trace.

### The partial analogue: `using` / `Symbol.dispose`

TypeScript 5.2 added explicit resource management:

```ts
{
  using conn = await pool.connect();   // conn[Symbol.asyncDispose]() at scope exit
  await conn.query("...");
}
```

That is the same idea as Java's try-with-resources: a scope-bound, automatic cleanup
hook. **Verdict: PARTIAL analogue.** The mechanism matches. What TypeScript does not
have is the *suppressed exception* concept — the case where cleanup itself fails
while an exception is already in flight, and you need both. Java has an explicit
answer for that, and it is one of the better bits of the language. We get to it below.

### What does not transfer at all: checked exceptions

> **NO TYPESCRIPT ANALOGUE.**
>
> TypeScript has no way to declare, in a function's type, which errors it can throw.
> `catch (e)` gives you `unknown` and that is the end of the type system's
> involvement. There is no compiler obligation on the *caller* of a function to do
> anything about its failures.
>
> Java has exactly that obligation. If a method declares `throws IOException`, every
> caller must either catch it or declare it too — and the program **will not
> compile** otherwise. This is a compile-time contract about failure, and it has no
> counterpart anywhere in your current stack. It is the single most distinctive
> feature of Java error handling, and Topic 09 is entirely about whether it was a
> good idea.

The nearest thing in your world is a hand-rolled `Result<T, E>` type (neverthrow,
fp-ts `Either`, a tRPC error union). Those are opt-in and enforced only where you use
them. Java's is language-level and unavoidable. Do not think of them as the same
thing; think of `Result` as a design pattern and checked exceptions as a language
rule you cannot escape.

### The mapping table

| You know | Java | Verdict |
|---|---|---|
| `throw new Error("...")` | `throw new RuntimeException("...")` | **HONEST ANALOGUE** |
| `catch (e) { if (e instanceof X) }` | `catch (X e)` clauses | **PARTIAL** — dispatch moves into the language |
| `finally` | `finally` | **HONEST ANALOGUE** |
| `using` / `Symbol.dispose` | try-with-resources | **PARTIAL** — Java also models *suppressed* exceptions |
| Unhandled promise rejection warning | Uncaught exception kills the thread | **PARTIAL** — a Java thread dies silently unless you install a handler |
| `e.cause` (ES2022) | `getCause()` / `initCause` | **HONEST ANALOGUE** |
| — | Checked exceptions | **NO ANALOGUE** |
| — | `Error` vs `Exception` split | **NO ANALOGUE** — JS has no "the runtime is broken" category |
| `throw "a string"` | impossible | — Java only throws `Throwable` |

---

## What is this?

### The hierarchy, and what each level *means*

```
Object
 └── Throwable                     the only thing you can throw or catch
      ├── Error                    the JVM or environment is compromised
      │    ├── OutOfMemoryError
      │    ├── StackOverflowError
      │    ├── NoClassDefFoundError
      │    └── ExceptionInInitializerError
      └── Exception                a condition an application might handle
           ├── IOException              CHECKED
           ├── SQLException             CHECKED
           ├── InterruptedException     CHECKED
           └── RuntimeException         UNCHECKED — a programming defect
                ├── NullPointerException
                ├── IllegalArgumentException
                ├── IllegalStateException
                ├── ClassCastException
                ├── ArithmeticException
                └── ConcurrentModificationException
```

The rule the compiler applies is mechanical and has exactly one sentence:

> **Everything that is not an `Error` and not a `RuntimeException` is checked, and
> the compiler forces you to catch it or declare it.**

But the *meaning* is the part that matters, and it is a contract, not a mechanism:

| Category | The contract it asserts | What you should do |
|---|---|---|
| `Error` | The JVM or environment is broken. Your code did not cause it and cannot fix it. | Do not catch. Let it kill the thread and let the process die or be restarted. |
| Checked `Exception` | Something outside your program failed, and the **caller has a plausible alternative action**. | Catch it where the alternative action lives — often several frames up. |
| `RuntimeException` | Your program has a defect. A precondition was violated. | Do not catch it locally. Let it reach a top-level handler that logs it and returns a 500. Then fix the code. |

Almost every bad exception decision in a Java codebase is a category error against
that table.

### Stack traces

Every `Throwable` captures the call stack at **construction** time, not at throw
time. That capture is the expensive part of an exception — walking the stack and
recording frames. It is why exceptions are not a control-flow mechanism, and why
`Throwable` has a constructor that lets you turn the capture off (Topic 09).

### Causes

An exception can carry another exception as its **cause**, forming a chain:

```java
catch (SQLException e) {
    throw new OrderPersistenceException("could not save order " + id, e);
    //                                                              ^^^ the cause
}
```

Printing that gives your message, your stack, then `Caused by: java.sql.SQLException:
...` and *its* stack. Dropping the cause is one of the highest-cost, lowest-effort
mistakes in this topic — you throw away the only evidence of what actually happened.

### Suppressed exceptions — the thing try-with-resources exists for

Here is the problem, written the old way:

```java
Connection conn = dataSource.getConnection();
try {
    conn.createStatement().execute("INSERT ...");   // throws SQLException A
} finally {
    conn.close();                                   // ALSO throws SQLException B
}
```

Exception A is in flight. Then `finally` runs, and `close()` throws B. **B replaces
A.** The original failure — the one you actually needed — is gone, permanently, with
no record. Your logs show a connection-close error and nothing about the failed
insert.

try-with-resources fixes this by keeping both: A propagates, and B is attached to A
as a **suppressed** exception, retrievable with `getSuppressed()` and printed by
`printStackTrace()` under a `Suppressed:` heading.

---

## Why does it matter?

Four things, all of which cost real money and none of which are stylistic.

**1. Lost evidence.** The single most expensive exception bug is not a crash — it is a
crash whose cause was overwritten. A hand-rolled `finally { close(); }` throws away the
exception you needed, and you spend hours debugging a connection pool that was never
the problem. try-with-resources exists to prevent exactly that, and knowing why is the
difference between using it reflexively and using it when you remember.

**2. Leaked handles.** A connection, file or socket that is not closed on every path is
a cumulative failure. Nothing goes wrong for hours. Then the pool is exhausted, and
every endpoint fails at once — including ones that touch no database. The code that
caused it ran four hours before the outage.

**3. Wrong category, wrong outcome.** Putting a failure in the wrong box has concrete
consequences. Catching an `Error` turns a clean crash into slow corruption. Swallowing
an `InterruptedException` makes every pod take an extra 30 seconds to shut down and
kills in-flight orders mid-transaction. And — the one that costs the most in Spring —
a **checked** exception thrown from a `@Transactional` method **commits** by default
(Topic 54). One classification decision, made in Phase 1, silently corrupts data in
Phase 5.

**4. Diagnosis speed.** Every production incident starts with a stack trace. Knowing
that the deepest `Caused by:` is usually the truth, that a `Suppressed:` block means a
cleanup also failed, and that an empty trace means the JIT elided it, turns a 20-line
wall of text into a five-second read. That is minutes versus hours, at 3am, repeatedly.

---

## Syntax breakdown

New constructs only. Topics 01–06 covered types, generics and classes.

### `throws` on a method

```java
public Order load(long id) throws OrderNotFoundException, IOException {
```

| Bit | What it means |
|---|---|
| `throws` | Part of the **signature**, not the body. It is a promise to callers about what can come out. |
| Multiple types | Comma-separated. Callers must handle each, or declare each. |
| Declaring an unchecked type | Legal, and purely documentation. The compiler does not enforce it on callers. Occasionally useful; often noise. |
| Overriding | An override may declare **fewer or narrower** checked exceptions than the method it overrides, never more. This is contravariance applied to failure, and it is why an interface with `throws Exception` poisons every implementation. |

### Multi-catch

```java
try {
    gateway.charge(order);
} catch (SocketTimeoutException | SSLHandshakeException e) {
    metrics.increment("payment.transport_failure");
    throw new PaymentUnavailableException("gateway transport failure", e);
}
```

| Bit | What it means |
|---|---|
| `A \| B` | One block for several unrelated types. The types must not be subtypes of one another — `IOException \| SocketTimeoutException` is a compile error. |
| `e`'s static type | The nearest common supertype (`IOException` here). |
| `e` is implicitly final | You cannot reassign it inside the block. |

### Catch ordering

```java
try { ... }
catch (FileNotFoundException e) { ... }   // must come FIRST
catch (IOException e) { ... }             // more general, comes after
```

Reverse the order and you get `error: exception FileNotFoundException has already
been caught` — a compile error, not a silent shadow. Good language design.

### try-with-resources

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(SQL)) {

    ps.setLong(1, orderId);
    ps.execute();

}   // ps.close() then conn.close() — reverse order, guaranteed, even on exception
```

| Bit | What it means |
|---|---|
| The resources go in `( )` | Each must implement `AutoCloseable` (or `Closeable`, which extends it and narrows `close()` to `throws IOException`). |
| Multiple resources | Separated by `;`. Closed in **reverse** declaration order — so a statement closes before the connection it came from. |
| No `finally` needed | The compiler generates it, including the suppressed-exception plumbing. |
| Effectively-final variables | Since Java 9 you can write `try (existingResource) { }` if the variable is final or effectively final. Before that you had to redeclare it. |
| Resource scope | The variables are only visible inside the `try` block. |
| It still takes `catch`/`finally` | `try (…) { } catch (SQLException e) { } finally { }` is legal and common. |

### `getSuppressed`

```java
catch (Exception primary) {
    for (Throwable suppressed : primary.getSuppressed()) {
        log.warn("also failed during cleanup: {}", suppressed.toString());
    }
    throw primary;
}
```

Returns an array (empty, never null). You rarely read it by hand — the default
`printStackTrace` already prints it — but knowing it exists is what turns
"why does my log show a close error?" into a five-second diagnosis.

### Precise rethrow (Java 7+)

```java
public void run() throws SocketTimeoutException, SSLHandshakeException {
    try {
        gateway.charge(order);
    } catch (Exception e) {          // catches broadly...
        metrics.increment("charge.failed");
        throw e;                     // ...but the compiler knows only two types can reach here
    }
}
```

The compiler analyses what the `try` block can actually throw and lets you declare
the precise set, even though you caught `Exception`. Before Java 7 you had to declare
`throws Exception` and lose all precision. Small feature, large ergonomic payoff.

---

## Example 1 — minimal

The point of this one is to *see* a suppressed exception, because you will meet the
symptom before you meet the concept.

```java
public class SuppressedDemo {

    static class FlakyResource implements AutoCloseable {
        private final String name;
        FlakyResource(String name) { this.name = name; }

        void work() {
            throw new IllegalStateException("work failed in " + name);
        }

        @Override
        public void close() {
            throw new IllegalStateException("close failed in " + name);
        }
    }

    public static void main(String[] args) {
        try (FlakyResource r = new FlakyResource("A")) {
            r.work();
        } catch (Exception primary) {
            System.out.println("primary  : " + primary.getMessage());
            for (Throwable s : primary.getSuppressed()) {
                System.out.println("suppressed: " + s.getMessage());
            }
        }
    }
}
```

Two failures happened. You keep both, and the one you keep as *primary* is the one
that actually broke your logic. Now write the same thing with a hand-rolled
`finally { r.close(); }` and observe that the `work failed` message is gone
completely. That deletion is what try-with-resources exists to prevent.

---

## Example 2 — production scenario

`orderflow` settles a batch of payments overnight. For each payment it must:

1. read a signed settlement file from the payment provider (an `InputStream`),
2. write matched rows to the database (a JDBC `Connection`),
3. write unmatched rows to a rejects file (a `Writer`),
4. and under no circumstances leave a connection or file handle open, because the
   job runs every night for years and handle leaks are cumulative.

There are three resources, two of which can fail on close, and a business rule about
what counts as recoverable.

```java
package com.orderflow.settlement;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.sql.*;
import java.util.*;
import javax.sql.DataSource;

/** Thrown when a whole settlement run cannot proceed. Unchecked: no caller can retry it usefully. */
public class SettlementRunFailedException extends RuntimeException {
    public SettlementRunFailedException(String message, Throwable cause) { super(message, cause); }
}

/** Thrown for ONE bad line. Checked: the caller has a real alternative — write it to rejects. */
public class UnparseableSettlementLineException extends Exception {
    private final int lineNumber;
    public UnparseableSettlementLineException(int lineNumber, String message, Throwable cause) {
        super("line " + lineNumber + ": " + message, cause);
        this.lineNumber = lineNumber;
    }
    public int lineNumber() { return lineNumber; }
}

public final class SettlementImporter {

    private static final String MATCH_SQL = """
            UPDATE payments
               SET status = 'SETTLED', settled_at = ?
             WHERE provider_reference = ? AND status = 'PENDING'
            """;

    private final DataSource dataSource;

    public SettlementImporter(DataSource dataSource) { this.dataSource = dataSource; }

    public SettlementResult importFile(Path settlementFile, Path rejectsFile) {

        int matched = 0, rejected = 0;

        try (BufferedReader in = Files.newBufferedReader(settlementFile, StandardCharsets.UTF_8);
             BufferedWriter rejects = Files.newBufferedWriter(rejectsFile, StandardCharsets.UTF_8);
             Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(MATCH_SQL)) {

            conn.setAutoCommit(false);

            String line;
            int lineNumber = 0;
            while ((line = in.readLine()) != null) {
                lineNumber++;
                try {
                    SettlementLine parsed = SettlementLine.parse(lineNumber, line);
                    ps.setTimestamp(1, Timestamp.from(parsed.settledAt()));
                    ps.setString(2, parsed.providerReference());
                    matched += ps.executeUpdate();

                } catch (UnparseableSettlementLineException bad) {
                    // A CHECKED exception, caught exactly where the alternative action is.
                    rejects.write(bad.lineNumber() + "," + line + "," + bad.getMessage());
                    rejects.newLine();
                    rejected++;
                }
            }
            conn.commit();
            return new SettlementResult(matched, rejected);

        } catch (SQLException e) {
            // Cannot recover a whole run. Wrap with the cause and let it kill the job.
            throw new SettlementRunFailedException(
                    "settlement run failed for " + settlementFile, e);

        } catch (IOException e) {
            throw new SettlementRunFailedException(
                    "could not read " + settlementFile + " or write " + rejectsFile, e);
        }
    }

    public record SettlementResult(int matched, int rejected) { }
}
```

Read what each decision buys:

| Decision | Why |
|---|---|
| Four resources in one `try (...)` | All four close, in reverse order, on every path including a `SQLException` mid-loop. There is no path that leaks a connection. |
| `UnparseableSettlementLineException` is **checked** | The caller genuinely has an alternative: write the line to rejects and continue. That is the exact test for "should this be checked?" |
| It is caught **inside** the loop | Because that is where the alternative action lives. Catching it outside the loop would abort the run on one bad line. |
| `SettlementRunFailedException` is **unchecked** | Nobody up the stack can do anything but fail the job and alert. Forcing every caller to declare `throws` would buy nothing. |
| Both wrappers pass the cause `e` | Without it, the log shows "settlement run failed" and no SQLState, no constraint name, nothing. |
| `setAutoCommit(false)` inside the try | If we throw before `commit()`, the connection is closed without committing, which rolls back. Correct by construction. |

And what is deliberately **not** here: there is no `catch (Exception e) { log.error(...) }`.
Nothing is swallowed. That is Topic 09's subject and its failure drill.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `return` inside `finally` eats the exception

**Wrong:**
```java
boolean chargeCustomer(Order order) {
    try {
        gateway.charge(order);
        return true;
    } finally {
        auditLog.record(order.id());
        return false;          // <-- the defect
    }
}
```

**Exact symptom:** the method **never throws** and always returns `false`. If
`gateway.charge` throws a `PaymentDeclinedException`, that exception is discarded
silently — no log line, no stack trace, no metric. The observable evidence is a
business number: orders sitting in `PENDING` forever with no corresponding failure in
the payment-error dashboard, because no error was ever raised.

**Root cause:** a `return` (or a `throw`, or a `break`) in a `finally` block
**completes the block abruptly**, which discards any in-flight exception and any
pending return value. The language spec says so explicitly. javac warns about it
under `-Xlint:finally`.

**Fix:** never put `return`, `break`, `continue` or `throw` in a `finally`. Put the
side effect there and nothing else.
```java
boolean chargeCustomer(Order order) {
    try {
        gateway.charge(order);
        return true;
    } finally {
        auditLog.record(order.id());
    }
}
```

---

### Trap 2 — hand-rolled close loses the real exception

**Wrong:**
```java
Connection conn = null;
try {
    conn = dataSource.getConnection();
    conn.createStatement().execute(sql);
} finally {
    if (conn != null) conn.close();   // this can throw too
}
```

**Exact symptom:** your on-call engineer sees
```
java.sql.SQLException: Connection is closed
	at com.zaxxer.hikari.pool.ProxyConnection.close(...)
	at com.orderflow.settlement.SettlementImporter.importFile(SettlementImporter.java:88)
```
and nothing at all about the constraint violation that actually caused the failure.
Hours are spent debugging the connection pool.

**Root cause:** the exception from `close()` replaces the in-flight exception from the
body. Only one exception can propagate from a frame, and the last one wins.

**Fix:** try-with-resources. The body's exception propagates; the close exception is
attached via `addSuppressed` and printed under `Suppressed:`. You keep both, in the
right priority order, with zero extra code.

---

### Trap 3 — swallowing `InterruptedException`

**Wrong:**
```java
try {
    Thread.sleep(backoffMillis);
} catch (InterruptedException e) {
    // nothing to do here
}
```

**Exact symptom:** on shutdown, `ExecutorService.shutdownNow()` returns but
`awaitTermination` times out. In Kubernetes the pod ignores SIGTERM, sits for the
full `terminationGracePeriodSeconds` (30s by default), and is then SIGKILLed. The
observable metric is deploy time: every rolling update takes 30 seconds per pod
longer than it should, and in-flight orders are killed mid-transaction rather than
draining.

**Root cause:** catching `InterruptedException` **clears the thread's interrupt
flag**. That flag is the only signal telling the rest of the stack "stop what you are
doing". You caught the message and threw it away, so nothing above you ever learns
that cancellation was requested.

**Fix:** either propagate it, or restore the flag.
```java
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // put the signal back
    throw new SettlementRunFailedException("interrupted during backoff", e);
}
```
This is Topic 89's territory in full; the rule ("never swallow an interrupt") is worth
installing now.

---

### Trap 4 — catching `Throwable` or `Error`

**Wrong:**
```java
try {
    processBatch(orders);
} catch (Throwable t) {
    log.error("batch failed", t);
    continueWithNextBatch();
}
```

**Exact symptom:** an `OutOfMemoryError` is caught and the loop continues. The JVM is
now in an undefined state — a thread may have died halfway through updating shared
data. What you see, thirty minutes later, is an unrelated `NullPointerException` in a
completely different subsystem, or a `HashMap` with a wrong `size()`, or a thread that
never wakes up. The heap dump you eventually take points at the wrong thing, because
the real event is thirty minutes and one log line back.

**Root cause:** `Error` means "the JVM or environment is compromised". Catching it
converts a clean, diagnosable crash into a slow, undiagnosable corruption.

**Fix:** catch `Exception`, never `Throwable`. If you genuinely need a top-level
safety net (an executor's uncaught-exception handler, a request filter), catch
`Exception` and let `Error` through. If you must catch `Throwable` at the absolute
process boundary, log it and **exit**, do not continue.

---

### Trap 5 — losing the cause

**Wrong:**
```java
catch (SQLException e) {
    throw new OrderPersistenceException("could not save order");
}
```

**Exact symptom:** a stack trace with exactly one frame set and no `Caused by:`
section. You know the save failed. You do not know whether it was a unique-constraint
violation, a deadlock, a timeout, or the database being down — and those four have
four completely different fixes and four different escalation paths.

**Root cause:** `OrderPersistenceException` has a `(String)` constructor and you used
it. The original `SQLException`, containing the SQLState, the vendor error code and
the constraint name, is now unreachable and garbage.

**Fix:** always give your exceptions a `(String, Throwable)` constructor and always
pass the cause.
```java
catch (SQLException e) {
    throw new OrderPersistenceException("could not save order " + order.id(), e);
}
```

**Review rule:** any `throw new` inside a `catch` block that does not mention the
caught variable is a defect. It is trivially greppable, and worth a lint rule.

---

## Hands-on proof

Everything below is a command **you** run. I do not have a JVM. Each proof states the
file, the command, what to look for, and how to read each outcome.

### Setup

```bash
mkdir -p ~/java-lab/08 && cd ~/java-lab/08
java --version
```

### Proof 1 — suppressed exceptions exist, and `finally` destroys them

`Suppression.java`:
```java
public class Suppression {

    static class Res implements AutoCloseable {
        public void work()  { throw new IllegalStateException("WORK FAILED"); }
        public void close() { throw new IllegalStateException("CLOSE FAILED"); }
    }

    static void modern() {
        try (Res r = new Res()) { r.work(); }
    }

    static void legacy() {
        Res r = new Res();
        try { r.work(); }
        finally { r.close(); }
    }

    public static void main(String[] args) {
        String mode = args.length > 0 ? args[0] : "modern";
        try {
            if (mode.equals("modern")) modern(); else legacy();
        } catch (Throwable t) {
            System.out.println("caught      : " + t.getMessage());
            System.out.println("suppressed n: " + t.getSuppressed().length);
            t.printStackTrace(System.out);
        }
    }
}
```

```bash
java Suppression.java modern
java Suppression.java legacy
```

**What to look for:** the value of `caught` and `suppressed n` in each run.

| What you see | What it means |
|---|---|
| modern: `caught = WORK FAILED`, `suppressed n = 1`, and a `Suppressed:` block in the trace | Correct. The real failure propagated; the cleanup failure was preserved beside it. |
| legacy: `caught = CLOSE FAILED`, `suppressed n = 0` | Correct, and it is the whole argument. `WORK FAILED` no longer exists anywhere in the universe. |
| modern shows `suppressed n = 0` | Your `close()` is not throwing — check you did not accidentally catch inside it. |
| Both show the same thing | You passed the wrong argument; check `args[0]`. |

**How to read it:** the two runs differ by nothing except which construct you used to
close a resource. One keeps the evidence and one deletes it. That is the entire case
for try-with-resources, made in two commands.

### Proof 2 — the compiler enforces checked exceptions

`CheckedProof.java`:
```java
import java.io.*;
import java.nio.file.*;

public class CheckedProof {
    static String readIt() {
        return Files.readString(Path.of("settlement.csv"));   // throws IOException
    }
    public static void main(String[] args) {
        System.out.println(readIt());
    }
}
```

```bash
javac CheckedProof.java
```

**What to look for:** compilation must FAIL.

| What you see | What it means |
|---|---|
| `error: unreported exception IOException; must be caught or declared to be thrown` | Expected. This is the compile-time obligation that has no TypeScript equivalent. Look at it properly — it is the defining feature of Java error handling. |
| It compiles | You caught it, or you added `throws IOException`. Both are the two legal answers, and the compiler accepted one. |

Now add `throws IOException` to `readIt` and recompile. It fails again — at `main`
this time. **The obligation propagates.** Keep adding `throws` until you reach `main`,
which is allowed to declare it. That propagation is exactly what Topic 09 argues about.

### Proof 3 — unchecked exceptions carry no obligation

Change `Files.readString` to `Integer.parseInt(args[0])` (which throws the *unchecked*
`NumberFormatException`) and recompile. It compiles with no complaint at all, and
fails at runtime when you pass a non-number.

| What you see | What it means |
|---|---|
| Clean compile, runtime `NumberFormatException` | The unchecked category. The compiler assumes a bad argument is your defect and does not force the caller to plan for it. |

Two files, one difference: which superclass the exception extends. That is the whole
mechanism.

### Proof 4 — the disappearing stack trace

This one catches people out at 2am and is worth seeing on purpose.

`FastThrow.java`:
```java
public class FastThrow {
    static int depth(String s) { return s.length(); }   // NPE when s is null

    public static void main(String[] args) {
        for (int i = 0; i < 200_000; i++) {
            try { depth(null); }
            catch (NullPointerException e) {
                if (i % 50_000 == 0) {
                    System.out.println(i + " frames=" + e.getStackTrace().length
                                         + " msg=" + e.getMessage());
                }
            }
        }
    }
}
```

```bash
java FastThrow.java
java -XX:-OmitStackTraceInFastThrow FastThrow.java
```

**What to look for:** whether `frames` drops to `0` partway through the first run.

| What you see | What it means |
|---|---|
| `frames` starts non-zero, later becomes `0`, and `msg` becomes `null` | The JIT replaced the repeatedly-thrown implicit exception with a preallocated, stack-trace-less instance. This is `OmitStackTraceInFastThrow`, on by default. It is why a hot production NPE sometimes has an empty trace while the same code in a debugger has a full one. |
| `frames` stays non-zero in the second run | The flag disabled the optimisation. This is the flag you set temporarily on a service to recover a trace you cannot reproduce locally. |
| `frames` never drops in either run | The loop was not hot enough or the JIT chose not to. Raise the iteration count. This optimisation is a JIT heuristic, not a guarantee. |

> **Uncertainty, stated honestly:** whether and when this kicks in is a JIT heuristic
> and has varied across HotSpot versions; it applies to implicit exceptions (NPE,
> ArithmeticException, ArrayIndexOutOfBounds, ClassCast) at hot call sites, not to
> exceptions you construct yourself. Settle what your JVM does with:
> `java -XX:+PrintFlagsFinal -version | grep -i OmitStackTrace`

### Proof 5 — close ordering

`CloseOrder.java`:
```java
public class CloseOrder {
    record Res(String name) implements AutoCloseable {
        public void close() { System.out.println("close " + name); }
    }
    public static void main(String[] args) {
        try (Res a = new Res("A"); Res b = new Res("B"); Res c = new Res("C")) {
            System.out.println("body");
        }
    }
}
```

```bash
java CloseOrder.java
```

**What to look for:** the order of the three `close` lines relative to `body`.

| What you see | What it means |
|---|---|
| `body`, then `close C`, `close B`, `close A` | Reverse declaration order, as specified. This is why a `PreparedStatement` declared after its `Connection` closes first — the dependency order is correct by construction. |
| Any other order | You are not on a standard JDK, or you edited the declaration order. |

### Proof 6 — see what the compiler generates

```bash
javac CloseOrder.java
javap -c CloseOrder.class
```

**What to look for** in `main`: an exception table with several entries, calls to
`Throwable.addSuppressed`, and duplicated close logic on the normal and exceptional
paths.

**How to read it:** you wrote one `try (…)`. The bytecode contains a full
`try`/`catch`/`finally` skeleton plus suppression plumbing you never typed. That is
the boilerplate try-with-resources deletes — and the reason hand-writing it correctly
is genuinely hard. Reading bytecode properly is Topic 76; today you are only
counting.

---

## Practice exercises

### 1 — Easy: classify

For each of these twelve situations, say which of the three categories it belongs in
(`Error`, checked, unchecked), and give the exact Java type you would throw. Then, for
the checked ones only, name the **alternative action** the caller has — if you cannot
name one, it should not be checked.

1. A settlement CSV file does not exist on disk.
2. `orderId` is negative.
3. The payment gateway returns HTTP 503.
4. The card was declined.
5. The JVM cannot allocate a 2 GB array.
6. A `Map` lookup returned null and was unboxed.
7. A config property `orderflow.gateway.url` is missing at startup.
8. A recursive order-tree walk hits a cycle.
9. A JSON payload has `quantity: "many"`.
10. Two threads modified an `ArrayList` at once.
11. A static initializer threw.
12. A wallet debit would take the balance below zero.

For at least three of them, argue that a reasonable engineer could put it in a
different category than you did, and say what would change their mind.

### 2 — Medium: combines Topics 01, 03, 05 and 06

Write a small `ConfigLoader` with this API:

```java
public final class ConfigLoader {
    public ConfigLoader(Path file) throws ConfigUnreadableException { ... }
    public <T> T get(String key, Class<T> type) { ... }
    public int getInt(String key) { ... }
}
```

Requirements:
- `ConfigUnreadableException` is **checked**. Justify that in a comment naming the
  caller's alternative action.
- `get` throws an **unchecked** `MissingConfigException` naming the key, because a
  missing key at read time is a deployment defect, not an event.
- `getInt` must not throw `NullPointerException` from unboxing (Topic 01). Prove it
  by writing a test that asks for an absent key and asserting on the exception type
  and message.
- `get(String, Class<T>)` uses a class literal as a type token — explain in a comment
  why the signature cannot be `<T> T get(String key)` and infer `T`. That is Topic 06.
- Use try-with-resources for the file read, and use package-private visibility
  (Topic 03) for anything that is not part of the API.

Then: deliberately break `close()` on your reader so it throws, trigger a parse
failure in the body, and print `getSuppressed()` to prove both exceptions survived.

### 3 — Hard: production simulation on `orderflow`

Build a resilient nightly settlement importer, then attack it.

**Part A — build.** Implement `SettlementImporter` from Example 2 against real files
and an embedded database (H2 in-memory is fine; no Docker needed at this stage).
Handle at minimum: file missing, file present but unreadable line, DB constraint
violation on one row, DB unavailable entirely.

**Part B — the resource-leak test.** Write a test harness that runs `importFile`
10,000 times against a DataSource limited to **2** connections, where 30% of runs hit
a `SQLException` mid-file. If any path leaks a connection, run 3 will hang.

```bash
java -cp . SettlementLeakHarness 10000
```
Capture the outcome:
- If it completes: your try-with-resources covers every path. Now deliberately
  rewrite one resource as a manual `finally { conn.close(); }` with a `return` inside
  the try, and confirm the harness now hangs. Record how many iterations it took.
- If it hangs: get a thread dump with `jcmd <pid> Thread.print` and find the thread
  parked in `getConnection`. Note the stack — you will see this exact shape again in
  Topic 109.

**Part C — the exception-shape review.** For every exception your importer can throw,
fill in this table and defend each row:

| Exception | Checked or unchecked | Caller's alternative action | What the on-call engineer sees |
|---|---|---|---|

Any row where the "alternative action" column says "log it" means the exception
should not be checked. Fix those.

**Part D — argue against yourself.** You made
`UnparseableSettlementLineException` checked. A colleague says every exception in the
codebase should be unchecked because Spring does that for data access. Write the
counter-argument, then write the strongest version of *their* argument, and say what
evidence would settle it. This is the setup for Topic 09.

---

## Interview questions

### Q1 — "What is the difference between a checked and an unchecked exception, and how do you decide?"

**Mid-level answer:** "Checked exceptions must be caught or declared; unchecked ones
don't. Checked extends `Exception`, unchecked extends `RuntimeException`."

**Senior answer:** "That is the mechanism. The contract is the useful part: the
hierarchy encodes recoverability. `Error` means the JVM or environment is
compromised and I should not catch it at all. Checked means something external
failed and the **caller has a plausible alternative action** — that phrase is my
actual decision test. Unchecked means a programming defect: a precondition was
violated, and the right response is to fail fast and fix the code, not to handle it
locally. If I cannot name the caller's alternative action in one sentence, the
exception should be unchecked, because a checked exception whose only handler is
`log.warn` has made every caller worse and bought nothing."

**What separates them:** having a *decision procedure* rather than a definition, and
treating `Error` as a third category with its own rule rather than lumping it with
exceptions.

**Follow-up:** "Give me an example where you would genuinely choose checked." The
best answers are narrow and file-shaped or protocol-shaped — a parse failure on one
record where the alternative is a rejects file, not "IOException".

---

### Q2 — "Why does try-with-resources exist? `finally` already worked."

**Mid-level answer:** "It closes resources automatically so you don't forget, and it's
less code."

**Senior answer:** "The boilerplate is the smaller half. The real reason is that the
naive `finally { close(); }` pattern **loses the original exception**: if the body
throws A and `close()` throws B, B propagates and A is gone forever. Your logs then
show a connection-close error and nothing about the constraint violation that
actually failed. try-with-resources keeps A as the primary and attaches B via
`addSuppressed`, which you can read with `getSuppressed()` and which
`printStackTrace` prints under a `Suppressed:` heading. It also closes multiple
resources in reverse declaration order, which is what makes a statement close before
its connection. Writing that by hand correctly, for three resources, is about
twenty lines nobody gets right."

**What separates them:** naming suppressed exceptions unprompted. That is the tell
for someone who has actually debugged a lost stack trace.

**Follow-up:** "Show me the `finally` version and tell me exactly which line loses
the exception."

---

### Q3 — "You see `catch (InterruptedException e) { }` in a PR. What do you say?"

**Mid-level answer:** "You should log it, not swallow it."

**Senior answer:** "Logging is still wrong. Catching `InterruptedException` **clears
the thread's interrupt flag**, which is the only mechanism by which cancellation
propagates. Once cleared, nothing above this frame knows shutdown was requested. In
a service that means `shutdownNow()` does not stop the work, `awaitTermination` times
out, and in Kubernetes the pod sits until the grace period expires and is then
SIGKILLed mid-transaction — you can literally see it as extra seconds per pod on
every deploy. The two correct responses are: rethrow it as-is if the signature
allows, or restore the flag with `Thread.currentThread().interrupt()` and then throw
something the caller can act on. Never neither."

**What separates them:** knowing that the flag is state, not just a message, and
being able to name the observable production consequence.

**Follow-up:** "What if the method signature can't declare it and you're inside a
`Runnable`?" — restore the flag and return; the pool will observe it.

---

### Q4 — "This NPE in production has an empty stack trace. What happened?"

**Mid-level answer:** "Something stripped it, maybe the logging framework."

**Senior answer:** "Most likely `OmitStackTraceInFastThrow`. When the JIT sees the
same implicit exception — NPE, `ArithmeticException`, array index, class cast —
thrown repeatedly at the same hot call site, it replaces it with a preallocated
instance that has no stack trace and no message. It is on by default. That is why it
reproduces with a full trace locally and comes back empty under load. To recover it,
restart the affected instance with `-XX:-OmitStackTraceInFastThrow`, accept the small
cost, and get one real trace. Then fix the null. I would also check whether we are
throwing that NPE thousands of times a second, because if the JIT noticed, it is a
hot path and the exception is being used as control flow somewhere."

**What separates them:** knowing the specific optimisation, the flag, *and* drawing
the second inference — an empty trace is itself evidence about throughput.

**Follow-up:** "Would you leave that flag on permanently?" No: it exists because
stack-trace capture is expensive; the right fix is not to throw there.

---

### Q5 — "Why should you never catch `Throwable`?"

**Mid-level answer:** "Because it catches `Error` too, and errors are serious."

**Senior answer:** "Because `Error` means the JVM or environment is compromised and
continuing produces undiagnosable damage. Catch an `OutOfMemoryError` in a batch loop
and you keep running with a thread that may have died halfway through mutating shared
state — what you then observe, thirty minutes later, is an unrelated NPE or a
`HashMap` whose `size()` disagrees with its contents. The real event is thirty minutes
and one swallowed log line back, and the heap dump you take points at the wrong
thing. The rule is: catch `Exception` at boundaries, let `Error` through, and if you
must have a process-level net, log and exit rather than continue. The one nuance is
`StackOverflowError`, which is sometimes genuinely recoverable — and I would still
rather fix the recursion."

**What separates them:** describing the *shape of the damage* rather than asserting
severity, and volunteering the one honest edge case.

**Follow-up:** "What about a top-level handler in a thread pool?" They want to hear
`Thread.UncaughtExceptionHandler` and an awareness that an uncaught exception in a
plain `Thread` kills it silently.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `Error` and `RuntimeException` are both unchecked, but they sit in different
   branches of the hierarchy. Why did the designers not simply make one
   `UncheckedException` category? What does the split let you write that a single
   category would not?

2. An overriding method may declare *fewer* checked exceptions than the method it
   overrides, never more. Why is that direction the safe one? Connect it to variance
   from Topic 07.

3. A stack trace is captured when the exception is **constructed**, not when it is
   thrown. Name a situation where those two points differ, and say what the trace
   would show.

4. TypeScript's `catch (e)` gives you `unknown`. Java's gives you a typed variable.
   Which language's design makes it easier to write a *correct* handler, and which
   makes it easier to write a *convenient* one? Are those the same question?

5. try-with-resources closes in reverse declaration order. Construct a case where
   forward order would be correct and reverse is wrong. Does one exist? If not, why
   not?

6. Suppose Java had no `finally` at all, only try-with-resources. What would become
   impossible to write, and would you miss it?

7. You are told "exceptions are slow, so we return error codes on the hot path." What
   specifically is slow about an exception, which of those costs can be removed, and
   what would you need to measure before agreeing?

---

## Quick reference card

### The decision table

| Situation | Category | Type to throw |
|---|---|---|
| Caller passed nonsense | unchecked | `IllegalArgumentException` |
| Object is in the wrong state for this call | unchecked | `IllegalStateException` |
| A required reference was null | unchecked | `NullPointerException` / `Objects.requireNonNull` |
| Feature genuinely not implemented yet | unchecked | `UnsupportedOperationException` |
| External resource failed, caller has an alternative | **checked** | your own, extending `Exception` |
| External resource failed, caller can only give up | unchecked | your own, extending `RuntimeException` |
| JVM or environment is broken | `Error` | do not throw or catch these |

### Syntax

```java
throws A, B                         // signature-level promise; enforced only for checked
catch (A | B e)                     // multi-catch; e is implicitly final
try (Res r = ...; Res2 s = ...) {}  // closes s then r, always
try (existingFinalVar) {}           // Java 9+; variable must be effectively final
e.getCause()                        // the wrapped exception
e.getSuppressed()                   // Throwable[]; never null
Thread.currentThread().interrupt()  // restore the flag you just cleared
Objects.requireNonNull(x, "msg")    // fail fast with a useful message
```

### Rules with no exceptions

- Never `return`, `break`, `continue` or `throw` inside `finally`.
- Never swallow `InterruptedException`. Rethrow or restore the flag.
- Never catch `Throwable` or `Error` and continue.
- Always pass the cause when wrapping. A `throw new` in a `catch` that ignores the
  caught variable is a defect.
- Close resources with try-with-resources, not `finally`.
- Catch subclasses before superclasses (the compiler enforces this).
- An empty `catch` block needs a comment explaining why, or it is a bug (Topic 09).

### Useful flags

```bash
-XX:-OmitStackTraceInFastThrow   # recover stack traces on hot implicit exceptions
javac -Xlint:finally             # warns about abrupt completion in finally
javac -Xlint:try                 # warns about unused try-with-resources variables
javac -Xdiags:verbose            # fuller compiler diagnostics
```

---

## When would I use this at work?

**1. Every time you touch a resource.**
A JDBC connection, a file, an HTTP response body, a Kafka producer. try-with-resources
is the default and there is no situation in day-to-day service code where hand-rolled
`finally { close(); }` is better. This one habit prevents an entire class of slow
production failure — handle exhaustion — that is miserable to diagnose because it
appears hours after the code that caused it.

**2. Reading an incident stack trace.**
The three questions you ask are: what is the primary exception, what is the deepest
`Caused by:`, and is there a `Suppressed:` block. Knowing that the deepest cause is
usually the truth and the outermost message is usually your own wrapper text turns a
20-line trace into a 5-second read.

**3. Designing a service interface other teams call.**
Every `throws` clause you write is a permanent obligation you impose on every caller,
forever, including ones that do not exist yet. Deciding checked-vs-unchecked at that
moment is a genuine API design decision with a cost you cannot walk back without a
breaking change. Topic 09 is that decision in full.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and autoboxing**: `NullPointerException` from unboxing is the
  single most common unchecked exception you will meet, and its message shape is a
  diagnosis tool.
- **03 — Access modifiers**: exception classes are part of your published API surface,
  so their visibility is a deliberate choice.
- **04 — Interfaces**: a `throws` clause on an interface method binds every
  implementation forever.

**This unlocks:**
- **09 — Exception API design**: the whole checked-exception argument, the `orderflow`
  domain hierarchy, and the failure drill for swallowed exceptions.
- **10 — Collections**: `ConcurrentModificationException` and
  `UnsupportedOperationException` are the two you will meet first.
- **21–24 — Lambdas and Streams**: checked exceptions do not compose through
  functional interfaces, which is one of Topic 09's strongest arguments.
- **26 — Optional**: the alternative to throwing for an expected absence.
- **28 — Sealed types**: errors-as-values, the other way to model failure.
- **46 — Error handling and `ProblemDetail`**: where your exception hierarchy meets
  HTTP status codes, in one place.
- **54 — `@Transactional`**: Spring rolls back on unchecked exceptions **only** by
  default. A checked exception thrown from a transactional method commits. That is
  the single most expensive consequence of this topic's classification decision.
- **67 — Class loading**: `ExceptionInInitializerError` and the `NoClassDefFoundError`
  that follows it and lies to you.
- **89 / 98 — Interruption and thread pools**: Trap 3 in full.

---

*Java baseline 21. The exception hierarchy and try-with-resources have been stable
since Java 7 and 9 respectively; nothing here changed in 21 or 25. Helpful NPE
messages became default in Java 15, so on any modern JDK you get the "Cannot invoke
X because Y is null" form. The `OmitStackTraceInFastThrow` behaviour is a JIT
heuristic that has varied across HotSpot versions — verify on your JVM with
`java -XX:+PrintFlagsFinal -version | grep -i OmitStackTrace` rather than assuming.*
