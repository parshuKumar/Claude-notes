# 19 — Serialization: Java serialization's CVE history, Jackson, schema evolution

## Phase: 1 — Core Language
## Category: CORE
## Java baseline: 21  |  Notes features from: 21
## Project spine: N/A (the `orderflow` service starts at Topic 35)

---

## ELI5 anchor

You want to send a chair to a friend.

**JSON is a photograph of the chair.** Your friend looks at the photo and builds their
own chair from their own wood, using their own instructions. If the photo shows
something they do not recognise, they ignore it or say "I do not know what that is".
Nothing in the photo can *make* your friend do anything. A photo is data.

**Java serialization is a magic spell.** You write down not just the chair's shape but
its *species* — "this is an OakChair" — and your friend's workshop looks up OakChair in
its own catalogue and runs OakChair's own assembly instructions.

Now the problem. Your friend's workshop has a catalogue of ten thousand furniture
species that came bundled with various tools they installed over the years. Some of
those species have assembly instructions that do surprising things — one of them, when
assembled, runs a shell command to check the wood grain. It was a reasonable feature
in 2007.

You do not choose which species is in the message. **The sender does.** So if a
stranger can send you a message, the stranger picks which assembly instructions your
workshop runs.

That is not a bug in one library. That is what the mechanism *is*. And it is why
"deserializing untrusted data" is a polite way of saying "letting a stranger run code
on your server".

---

## The bridge from what you know

### `JSON.parse` versus `readObject` — **NO ANALOGUE**

I want to be literal, because this is the most dangerous false analogy in the
curriculum.

```ts
const order = JSON.parse(body);   // safe by design
```

`JSON.parse` produces **only** plain objects, arrays, strings, numbers, booleans and
null. That is the entire output alphabet of the function. It does not call a
constructor. It does not look up a class. It does not run any code that the *sender*
chose. The worst a malicious payload can do is be very large, or use `__proto__` to
attempt prototype pollution — a real vulnerability class, but one that requires your
own code to then use the polluted object carelessly.

The safety is **structural**. It comes from the format having no concept of "which
code should run".

```java
ObjectInputStream in = new ObjectInputStream(socket.getInputStream());
Object order = in.readObject();   // NOT safe by design
```

`readObject` reads a **class name** out of the byte stream, loads that class from your
classpath, allocates an instance without calling its constructor, fills in its fields
from the stream, and then — if that class defines one — **invokes that class's own
private `readObject` method**. Then it does the same recursively for every object in
the graph.

So the byte stream is not data. The byte stream is a **program**, whose instructions are
"instantiate this class, then this one, then this one" and whose available operations
are every `readObject`, `readResolve`, `hashCode`, `equals`, `toString`, `compareTo`
and `finalize` on your entire classpath.

| You know | Java | Verdict |
|---|---|---|
| `JSON.parse` produces inert data | `readObject` reconstructs a live object graph and runs its code | **NO ANALOGUE** — this is why one is a parsing problem and the other is remote code execution |
| Prototype pollution via `__proto__` | Not a thing; Java has no prototype chain | **NO ANALOGUE** |
| `JSON.stringify` drops functions and cycles | Java serialization preserves cycles and object identity | **PARTIAL** — Java's is strictly more powerful, which is precisely the problem |
| A reviver function you wrote runs on each node | A `readObject` method **the sender chose** runs on each node | **NO ANALOGUE** — your reviver is your code; `readObject` is whatever class the stream names |
| `zod` / `class-transformer` validating a parsed shape | Bean Validation on a Jackson-parsed DTO | **HONEST ANALOGUE** — this part transfers cleanly (Topic 45) |
| npm package versions and breaking changes | `serialVersionUID` compatibility | **PARTIAL** — same *problem* (schema evolution), completely different mechanism and failure message |

### The one part that does transfer

Jackson-with-DTOs is genuinely close to what you already do. You parse JSON into a
declared shape, unknown fields are ignored or rejected by configuration, and validation
is a separate step. Your instincts from `zod` and `class-validator` are correct here.

The trap inside the transferable part is **polymorphic typing**: if you configure
Jackson to read a Java class name out of the JSON and instantiate it, you have
reinvented the Java serialization problem inside JSON. See Trap 5.

---

## What is this?

Three different things that all get called "serialization", with very different risk
profiles.

**1. Java built-in serialization.** `implements Serializable`, `ObjectOutputStream`,
`ObjectInputStream`. A binary format that captures a whole object graph including
private fields, cycles and object identity. Shipped in Java 1.1. Used by RMI, JMX, old
HTTP session replication, and a lot of caches.

**2. JSON via Jackson.** A text format mapping to declared Java types. This is what
`orderflow`'s REST API uses, and what Spring Boot configures for you. Safe by default;
made unsafe by specific configuration choices.

**3. Schema-first binary formats.** Protobuf, Avro, Thrift. You declare a schema in a
separate file, generate code from it, and the schema is the compatibility contract. This
is what you use for anything crossing a service boundary that you intend to evolve.

The industry direction, stated plainly:

> Java's built-in serialization is regarded — by the people who maintain the JDK — as a
> design mistake. It has been the source of a large fraction of Java's remote-code-
> execution CVEs. Its replacement path is: records for the cases that must stay in the
> platform, and schema-first formats for everything crossing a boundary.

**A precision point on deprecation, because I want to be exact.** As of Java 21,
`java.io.Serializable` is **not deprecated**. `implements Serializable` compiles without
a warning. There is no announced removal date. What has actually happened is:
serialization filtering was added (Java 9, JEP 290, backported to 8u121); context-
specific filters were added (Java 17, JEP 415); records deserialize through their
canonical constructor (Java 16+) rather than bypassing it; and the platform documents
serialization as a mechanism to avoid in new code.

Settle the deprecation question yourself rather than trusting me:

```bash
cat > SerCheck.java <<'EOF'
import java.io.Serializable;
public class SerCheck implements Serializable {
    private static final long serialVersionUID = 1L;
}
EOF
javac -Xlint:all SerCheck.java
```

| What you see | What it means |
|---|---|
| No warning at all | `Serializable` is not deprecated on your JDK. This is what I expect on 21 and 25. |
| A deprecation warning naming `Serializable` | The status changed after my knowledge cutoff. Trust your compiler, not this document, and check the JEP index at openjdk.org/jeps. |
| A `serial` lint warning on some *other* class | Normal: `-Xlint:serial` warns about `Serializable` classes with no `serialVersionUID`. That is a useful warning to leave on. |

---

## Why does it matter?

**1. It is the single largest source of critical Java CVEs.** In 2015 a talk called
"Marshalling Pickles" and a research tool called `ysoserial` demonstrated that
deserializing untrusted data with common libraries on the classpath — Apache Commons
Collections was the canonical example — yielded reliable remote code execution. What
followed was a multi-year sweep of critical CVEs across WebLogic, WebSphere, JBoss,
Jenkins, and many others. The pattern was always the same: an endpoint accepted
serialized Java objects; a "gadget chain" of ordinary library classes, invoked through
their own `readObject`/`hashCode`/`toString` methods, ended in a call that executed a
process.

I am confident about the shape and the year and the Commons Collections
`InvokerTransformer` gadget. I am **not** going to quote specific CVE numbers from
memory. If you need them for a writeup, search `nvd.nist.gov` for "deserialization"
plus the product, and cite what you find.

One clarification worth having, because it comes up: **Log4Shell was not a
deserialization bug.** It was a JNDI lookup triggered from a log message. Different
mechanism, similar lesson — data that gets *interpreted* is dangerous.

**2. `serialVersionUID` breaks deploys.** Add a field to a `Serializable` class used in
your HTTP session or your cache, deploy it, and every old serialized object in the cache
becomes unreadable with an `InvalidClassException`. If the class is in session state
replicated across a cluster, a rolling deploy takes the site down.

**3. Schema evolution during a rolling deploy is a real, weekly problem.** Old and new
instances run simultaneously for minutes. Old instances write payloads that new
instances read, and vice versa. A field rename that looks trivial in a PR takes out
half your traffic for the duration of the rollout. This applies to JSON just as much as
to Java serialization — it is a *distributed systems* problem, not a format problem.

**4. Deserialization bypasses your constructor.** For an ordinary `Serializable` class,
the JVM allocates the instance and fills the fields directly. Your validation does not
run. Your defensive copies from **Topic 17** do not happen. An object that your
constructor would have rejected as invalid can exist in your heap.

---

## Syntax breakdown

Everything in this section is genuinely new to you — there is no TypeScript
counterpart for any of it.

### Making a class serializable

```java
import java.io.Serializable;

public class OrderSnapshot implements Serializable {

    // 1. The compatibility contract. Explicit, always.
    private static final long serialVersionUID = 1L;

    private long orderId;
    private String customerEmail;

    // 2. NOT written to the stream. Reads back as null / 0 / false.
    private transient String gatewayApiKey;

    // 3. Called by the JVM when writing. Optional.
    private void writeObject(java.io.ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();          // write all non-transient fields
        // ... anything extra you want in the stream
    }

    // 4. Called by the JVM when reading. Optional. THIS IS THE DANGEROUS ONE.
    private void readObject(java.io.ObjectInputStream in)
            throws IOException, ClassNotFoundException {
        in.defaultReadObject();            // populate all non-transient fields
        // ... any code you want to run at deserialization time
    }

    // 5. Called when the stream has no data for this class in the hierarchy.
    private void readObjectNoData() throws java.io.ObjectStreamException { }

    // 6. Substitute a different object on the way OUT.
    private Object writeReplace() throws java.io.ObjectStreamException { return this; }

    // 7. Substitute a different object on the way IN. Used to preserve singletons.
    private Object readResolve() throws java.io.ObjectStreamException { return this; }
}
```

| Construct | What it is | Why it matters here |
|---|---|---|
| `implements Serializable` | A **marker interface** — it declares no methods. Its only job is to say "the JVM may serialize this". | It is opt-in, but it is inherited: any subclass of a `Serializable` class is also serializable whether you wanted that or not. |
| `serialVersionUID` | A `private static final long` that identifies the class *version*. If the value in the stream does not match the value in the loaded class, deserialization fails. | If you do not declare it, the JVM **computes** one from the class's structure — name, fields, methods, interfaces. Change almost anything and the computed value changes. That is Trap 1. |
| `transient` | "Do not write this field to the stream." | The correct place for secrets, caches, derived values, and anything not serializable. Also a silent source of nulls — Trap 3. |
| `writeObject` / `readObject` | Private callbacks the JVM invokes reflectively. Note: **private**, and yet the JVM calls them. | `readObject` runs **before any of your application code touches the object**. This is the entire security story. |
| `readResolve` | Replace the deserialized object with another. | The standard way to keep an enum-like singleton a singleton across deserialization. |
| `Externalizable` | An alternative interface where *you* write both directions explicitly. Requires a public no-arg constructor. | Faster and more controlled, but it is still Java serialization and still has the same trust problem. |

### The serialization filter — the defensive control

Added in Java 9 (JEP 290). This is the mechanism that lets you say "no" **before** the
class is loaded and before `readObject` runs.

```bash
# A JVM-wide filter, as a system property.
java -Djdk.serialFilter='com.orderflow.**;java.base/*;!*' MyApp
```

Pattern syntax, which is its own small language:

| Pattern | Meaning |
|---|---|
| `com.orderflow.orders.OrderSnapshot` | allow exactly this class |
| `com.orderflow.*` | allow classes in this package |
| `com.orderflow.**` | allow this package and all subpackages |
| `!com.evil.*` | **reject** this package (leading `!`) |
| `!*` | reject everything not already allowed. **Put this last.** |
| `maxdepth=10` | reject graphs nested deeper than 10 |
| `maxrefs=1000` | reject streams with more than 1000 back-references |
| `maxbytes=100000` | reject streams larger than this |
| `maxarray=10000` | reject arrays longer than this |

The four `max*` limits are separate from the class patterns and defend against a
different attack: a small payload that expands into an enormous object graph and
exhausts memory before any class filter matters.

You can also set a filter **per stream**, which is much better practice than a global
one:

```java
ObjectInputStream in = new ObjectInputStream(source);
in.setObjectInputFilter(
    ObjectInputFilter.Config.createFilter("com.orderflow.orders.OrderSnapshot;!*"));
Object o = in.readObject();
```

And, since Java 17 (JEP 415), a **filter factory** that can apply different filters to
different streams in the same JVM:

```java
ObjectInputFilter.Config.setSerialFilterFactory(myFactory);   // set once, at startup
```

### Jackson — the annotations that matter for evolution

```java
public record OrderPlaced(
    @JsonProperty("order_id")                long orderId,
    @JsonAlias({"customerId", "customer_id"}) long customerId,
    @JsonProperty("total_minor")             long totalMinor
) {}
```

| Annotation | What it does |
|---|---|
| `@JsonProperty("x")` | The wire name for this field. Decouples your Java name from the contract. |
| `@JsonAlias({"a","b"})` | **Additional accepted names on read.** Writes use `@JsonProperty`. This is the tool that makes a rename survivable. |
| `@JsonIgnoreProperties(ignoreUnknown = true)` | Do not fail on fields you do not recognise. |
| `@JsonIgnore` | Never write this field. The JSON equivalent of `transient`. |
| `@JsonCreator` | Name the constructor/factory to use, so validation runs. |
| `@JsonInclude(NON_NULL)` | Omit null fields from output. |

Spring Boot's default: `FAIL_ON_UNKNOWN_PROPERTIES` is **disabled**, so unknown fields
are ignored. Plain Jackson's default is the opposite. Know which one you are running.

---

## Example 1 — minimal

A class whose deserialization does something you did not ask for.

```java
import java.io.*;

public class LoyaltyToken implements Serializable {

    private static final long serialVersionUID = 1L;

    private final String customerEmail;

    public LoyaltyToken(String customerEmail) {
        if (customerEmail == null || !customerEmail.contains("@")) {
            throw new IllegalArgumentException("invalid email: " + customerEmail);
        }
        this.customerEmail = customerEmail;
        System.out.println("[constructor] validated " + customerEmail);
    }

    private void readObject(ObjectInputStream in)
            throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        System.out.println("[readObject] I ran. Nobody asked me to. email=" + customerEmail);
    }

    @Override public String toString() { return "LoyaltyToken[" + customerEmail + "]"; }
}
```

```java
import java.io.*;

public class RoundTrip {
    public static void main(String[] args) throws Exception {
        byte[] bytes;
        try (var bos = new ByteArrayOutputStream();
             var oos = new ObjectOutputStream(bos)) {
            oos.writeObject(new LoyaltyToken("ada@orderflow.test"));
            bytes = bos.toByteArray();
        }

        System.out.println("--- about to deserialize ---");
        try (var ois = new ObjectInputStream(new ByteArrayInputStream(bytes))) {
            Object o = ois.readObject();
            System.out.println("--- my code finally runs --- got " + o);
        }
    }
}
```

Run it and read the **order** of the printed lines.

- `[constructor]` prints once, during serialization.
- `[readObject]` prints **after** `--- about to deserialize ---` and **before**
  `--- my code finally runs ---`.
- `[constructor]` does **not** print a second time.

Two facts fall out of that ordering, and they are the whole topic:

1. **The constructor did not run on the way back in.** Your validation was skipped
   entirely. Any invariant you enforce in a constructor is not enforced here.
2. **Code ran before you got control.** Between calling `readObject()` and receiving
   the object, arbitrary code executed — code belonging to whichever class the *stream*
   named.

---

## Example 2 — production scenario

`orderflow` publishes an `OrderPlaced` event when an order is created. The payments
service consumes it. Both are deployed independently, several times a week.

### The version that ships and then breaks a rolling deploy

```java
package com.orderflow.orders.events;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.List;

public class OrderPlaced implements Serializable {
    // no serialVersionUID declared            <-- defect 1
    private long orderId;
    private long customerId;
    private BigDecimal total;                  // <-- defect 2: no currency
    private List<OrderLineDto> lines;
    private transient String idempotencyKey;   // <-- defect 3: silently lost
}
```

It is published to Kafka as Java-serialized bytes and consumed by the payments service.

Three failures, in the order the team meets them.

**Week 1.** A developer adds a `promotionCode` field. Deploy goes out. The payments
service — still running the old class — starts throwing:

```
java.io.InvalidClassException: com.orderflow.orders.events.OrderPlaced;
  local class incompatible: stream classdesc serialVersionUID = 8642983475902348234,
  local class serialVersionUID = 1029348572034958723
```

Every in-flight event fails. The consumer group stalls. Lag climbs. Because no
`serialVersionUID` was declared, the JVM computed one from the class structure, and
adding a field changed it.

**Week 3.** Someone fixes that by pinning `serialVersionUID = 1L` on both sides. Now
deserialization succeeds — and `idempotencyKey` arrives as `null` on every event,
because it is `transient`. Duplicate payments start appearing. There is no exception
anywhere; the only signal is a business metric.

**Week 6.** A security review notices that the payments consumer calls `readObject` on
bytes that arrive from a Kafka topic which several teams can produce to. The topic is
now an unauthenticated code-execution endpoint into the payments service.

### The corrected version

Two changes: a schema-first format for the wire, and a filter for anything that still
uses Java serialization.

```java
package com.orderflow.orders.events;

import com.fasterxml.jackson.annotation.JsonAlias;
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

/**
 * Wire contract for the order-placed event.
 *
 * Rules for changing this type, enforced in review:
 *   - Adding an optional field: always safe.
 *   - Removing a field: only after every consumer has stopped reading it.
 *   - Renaming a field: NEVER in one step. Add @JsonAlias, deploy, then rename.
 *   - Changing a field's type: never. Add a new field instead.
 */
@JsonIgnoreProperties(ignoreUnknown = true)          // forward compatible: tolerate new fields
public record OrderPlaced(

        @JsonProperty("event_version")   int eventVersion,      // explicit, not inferred
        @JsonProperty("order_id")        long orderId,
        @JsonProperty("customer_id")     long customerId,

        // Money as minor units + explicit currency (Topic 01). Never BigDecimal alone.
        @JsonProperty("total_minor")     long totalMinor,
        @JsonProperty("currency")        String currency,

        // Renamed from "dedupeKey" in v2. The alias keeps v1 producers readable.
        @JsonProperty("idempotency_key")
        @JsonAlias({"dedupeKey", "dedupe_key"})
        String idempotencyKey,

        @JsonProperty("lines")           List<OrderLineDto> lines
) {
    public OrderPlaced {
        // Runs on Jackson deserialization too, because records use the canonical
        // constructor. This is the single biggest reason to prefer records here.
        if (orderId <= 0)          throw new IllegalArgumentException("orderId");
        if (currency == null)      throw new IllegalArgumentException("currency");
        if (idempotencyKey == null) throw new IllegalArgumentException("idempotencyKey");
        lines = List.copyOf(lines);                  // Topic 17: defensive copy
    }
}
```

And for any code path that still touches `ObjectInputStream` — a legacy cache, an old
library, an HTTP session store — a filter at the JVM level as a backstop:

```bash
java -Djdk.serialFilter='maxdepth=20;maxrefs=5000;maxbytes=500000;\
com.orderflow.**;java.util.*;java.lang.*;java.time.*;!*' \
     -jar orderflow-payments.jar
```

Five decisions, each removing a specific failure:

1. **JSON, not Java serialization, on the wire.** The consumer can no longer be made to
   instantiate a class of the producer's choosing.
2. **A `record`.** Jackson deserializes records through the canonical constructor, so
   the compact-constructor validation and the `List.copyOf` actually run. An ordinary
   class would have both bypassed.
3. **`@JsonIgnoreProperties(ignoreUnknown = true)`.** Old consumers tolerate fields
   added by new producers. This is what makes "add a field" a safe deploy.
4. **`@JsonAlias` for the rename.** Both the old and new names are accepted on read
   during the rollout window.
5. **An explicit `event_version` field.** When a change genuinely cannot be made
   compatible, the consumer branches on the version rather than guessing.

> Note what is *not* here: there is no `serialVersionUID`, because there is no Java
> serialization. Removing the mechanism removes its entire failure class, which is
> always better than managing it. See **Topics 115 and 116** for the outbox and
> idempotency machinery this event feeds, and **Topic 123** for rolling-deploy
> mechanics.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — no explicit `serialVersionUID`

**Wrong:**
```java
public class OrderSnapshot implements Serializable {
    private long orderId;
    private String customerEmail;
    // no serialVersionUID
}
```

**Exact symptom:**
```
java.io.InvalidClassException: com.orderflow.orders.OrderSnapshot;
  local class incompatible: stream classdesc serialVersionUID = -4573920481093847502,
  local class serialVersionUID = 8930485720394857203
	at java.base/java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:...)
```

It appears the moment old data meets new code: during a rolling deploy, when reading a
replicated HTTP session, or when loading a cache written by the previous version.

**Root cause:** if you do not declare `serialVersionUID`, the JVM computes one by
hashing the class's structure — its name, modifiers, interfaces, fields, and even some
method signatures. Adding a field, changing a modifier, or in some cases adding a
method changes the hash. Two JVMs with structurally different versions of the class
compute different values and refuse to talk.

**Fix, and be precise about what it does and does not buy you:**

```java
private static final long serialVersionUID = 1L;
```

Declaring it **suppresses the version check**. It does not make your change compatible.
It tells the JVM "trust me, these are compatible" — and then the JVM applies its
default field-matching rules: fields present in the stream but absent from the class are
discarded; fields present in the class but absent from the stream are left at their
default (`null`/`0`/`false`).

So declaring the UID converts a **loud** failure into a **silent** one. That is usually
better, because you control the silence, but only if you also apply the compatible-change
discipline:

| Change | Compatible with a fixed UID? |
|---|---|
| Add a field | Yes. Old data leaves it at default — so it must be *optional*. |
| Remove a field | Yes. New code discards it. |
| Rename a field | **No.** It is a remove plus an add. The value is lost silently. |
| Change a field's type | **No.** Fails or corrupts. |
| Change the class name or package | **No.** |
| Add or remove a method | Yes. |
| Change a field from `transient` to non-transient | Yes for new data; old data has no value for it. |

**Also turn the compiler warning on**, so a missing UID is caught in CI:

```bash
javac -Xlint:serial -Werror src/main/java/**/*.java
```

---

### Trap 2 — a Jackson field rename during a rolling deploy

**Wrong:** a PR renames `dedupeKey` to `idempotencyKey` in an event record. It is a
one-line change, all tests pass, and it is obviously an improvement.

**Exact symptom:** during the rollout window — typically 2 to 15 minutes, while old and
new pods both run — you see one of two things depending on configuration.

If `FAIL_ON_UNKNOWN_PROPERTIES` is enabled:
```
com.fasterxml.jackson.databind.exc.UnrecognizedPropertyException:
  Unrecognized field "dedupeKey" (class com.orderflow.orders.events.OrderPlaced),
  not marked as ignorable (6 known properties: "order_id", "customer_id", ...)
```
Error rate spikes to roughly 50% and returns to zero when the rollout completes. If
nobody is watching during those minutes, it looks like a transient blip.

If unknown properties are ignored (Spring Boot's default), it is **worse**:
`idempotencyKey` arrives as `null` on every message from an old producer. No exception.
No error metric. Duplicate payments, discovered days later by reconciliation.

**Root cause:** a rolling deploy is not atomic. For the duration of the rollout, two
versions of the schema are in flight simultaneously, in both directions — old producers
to new consumers *and* new producers to old consumers. A rename is a remove plus an add,
and neither half is compatible.

**Fix — the three-deploy rename.** There is no two-deploy version of this.

1. **Deploy 1:** add `@JsonAlias({"dedupeKey"})` alongside the existing name, and keep
   *writing* the old name. Now every instance can read both spellings.
2. **Deploy 2:** switch the write side to the new name (`@JsonProperty("idempotency_key")`).
   Every instance can still read both, so both directions work.
3. **Deploy 3:** once no in-flight message can carry the old name — which for Kafka means
   after the topic retention window, not after the deploy — remove the alias.

The same shape applies to a database column rename, a protobuf field rename, and an API
field rename. Learn it once here.

**Prove it in CI** rather than in production: keep a directory of golden payloads from
every released version and assert that the current code deserializes all of them. That
test catches this class of bug in the PR, which is the only place it is cheap.

---

### Trap 3 — `transient` on a field you needed

**Wrong:**
```java
private transient String idempotencyKey;   // marked transient to "keep the payload small"
```

**Exact symptom:** no exception. The field is `null` after every round trip. What you
observe is entirely downstream: a null check that always passes, a deduplication check
that never matches, duplicate wallet debits, or an NPE thrown from somewhere completely
unrelated to serialization, hours later in the call path.

**Root cause:** `transient` means "do not write this to the stream". On read, the field
is not touched at all — and because deserialization does not run your constructor, there
is no code to give it a value. It stays at the field's default.

**Fix, in preference order:**

1. Do not mark it transient. If it needs to survive the round trip, it goes on the wire.
2. If the field is genuinely derived, recompute it in `readObject`:
   ```java
   private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
       in.defaultReadObject();
       this.derivedTotal = recomputeTotal();   // restore the invariant the constructor would have
   }
   ```
3. If the field is a secret — an API key, a token, a password — `transient` is correct,
   and the fix is that the *object holding it* should not be serialized at all.

**The review rule:** every `transient` field needs a comment saying what restores it.
If nothing restores it, the field's absence is a documented part of the contract, or the
`transient` is a bug.

---

### Trap 4 — storing serialized Java objects in a cache, session, or database column

**Wrong:**
```java
redis.set("order:" + id, serialize(order));          // Java-serialized bytes
// ... later, possibly on a different version of the code
Order order = (Order) deserialize(redis.get("order:" + id));
```

**Exact symptom:** three separate failures, arriving at different times.

- **Deploy day:** `InvalidClassException` on every cache hit written by the previous
  version. The cache effectively empties, the database takes the full read load, and
  latency spikes cluster-wide. This is Trap 1 wearing a cache costume.
- **Any day:** anyone who can write to Redis can write a serialized payload of their
  choosing, and your service will deserialize it. Redis is now a code-execution
  endpoint. This is not hypothetical — cache and session stores are a documented
  deserialization attack path.
- **Slowly:** the format is opaque. You cannot inspect the cache, cannot debug it,
  cannot migrate it, and cannot read it from any tool that is not a JVM with your exact
  classes on its classpath.

**Root cause:** Java serialization is a *transport between two JVMs running the same
code at the same instant*. Every use that violates that assumption — persistence,
caching, cross-version, cross-service — is using it outside its design.

**Fix:** store JSON, or a schema-first binary format, in the cache. You accept a small
size increase and gain: version tolerance, inspectability, no code execution on read,
and the ability to read the cache from a script.

If you inherit a codebase that already does this, the interim mitigation is a strict
allowlist filter (see the drill below) while you migrate the format.

---

### Trap 5 — Jackson polymorphic typing

**Wrong:**
```java
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
                             ObjectMapper.DefaultTyping.NON_FINAL);   // <-- the defect
```
or, on a type:
```java
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS, include = JsonTypeInfo.As.PROPERTY, property = "@class")
public abstract class PaymentInstruction { }
```

**Exact symptom:** by itself, none — it works exactly as intended. The symptom arrives
from your security scanner or your penetration test, in the form of a report naming a
Jackson deserialization CVE, or from a real incident. The JSON payload carries a
`"@class": "some.library.EvilBean"` property and Jackson dutifully instantiates it and
calls its setters.

**Root cause:** you configured Jackson to read a **Java class name out of the payload**
and instantiate it. That is precisely the property that makes Java serialization
dangerous, reimplemented in JSON. The attacker again chooses which code runs; the only
difference is that the gadget must be reachable through a constructor or setter rather
than through `readObject`.

Jackson has shipped a long series of blocklist updates for exactly this — new gadget
classes keep being found — which tells you the shape of the problem: a blocklist of
known-bad classes is a losing race.

**Fix:**

1. **Do not use default typing.** Ever. It has been discouraged by Jackson's own
   maintainers for years.
2. If you genuinely need polymorphism, use `Id.NAME` with an explicit `@JsonSubTypes`
   allowlist, so the payload chooses among *types you named*, not among every class on
   the classpath:
   ```java
   @JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "kind")
   @JsonSubTypes({
       @JsonSubTypes.Type(value = CardPayment.class,   name = "CARD"),
       @JsonSubTypes.Type(value = WalletPayment.class, name = "WALLET")
   })
   public sealed interface PaymentInstruction permits CardPayment, WalletPayment { }
   ```
3. Better still, make it a **sealed interface over records** (**Topic 28**) so the
   compiler knows the closed set and your `switch` is exhaustive.

**The general rule, which is the takeaway from this entire document:**

> Never let the *payload* decide which code runs. Let the *code* decide how to read the
> payload.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and will not print invented
output. Each proof gives the command, what to look for, and how to read every outcome.

### Setup

```bash
mkdir -p ~/java-lab/19 && cd ~/java-lab/19
java --version      # expect 21 or 25
```

### Proof 1 — the byte stream is readable, and it names classes

Write and serialize anything, then look at the bytes.

`Dump.java`:
```java
import java.io.*;
import java.util.*;

public class Dump {
    record Line(String sku, int qty) implements Serializable { }

    public static void main(String[] args) throws Exception {
        var lines = new ArrayList<Line>(List.of(new Line("SKU-4471", 2)));
        try (var oos = new ObjectOutputStream(new FileOutputStream("order.ser"))) {
            oos.writeObject(lines);
        }
        System.out.println("wrote order.ser");
    }
}
```

```bash
java Dump.java
xxd order.ser | head -20        # or: od -c order.ser | head -20
strings order.ser
```

**What to look for:** the first four bytes, and the plain-text class names.

| What you see | What it means |
|---|---|
| The file starts `ac ed 00 05` | Confirmed. `0xACED` is `STREAM_MAGIC`, `0x0005` is `STREAM_VERSION`. Any byte sequence starting with these four bytes is a Java serialization stream — this is how scanners and WAFs detect them, and how *you* can spot one in a log or a database column. |
| `java.util.ArrayList` and `Dump$Line` appear as readable ASCII in `strings` output | The whole point. **The class to instantiate is written in the payload.** Change those bytes and you change which class gets loaded. |
| Field names appear as readable text too | Also expected. The format is self-describing, which is why it is large. |

Now do the same for the base64 form, because that is how you will actually meet it —
in a cookie, an HTTP header, or a database column:

```bash
base64 -i order.ser | head -c 40 ; echo
```

**How to read it:** a base64 string beginning `rO0AB` is `0xACED0005` encoded. Seeing
`rO0AB` anywhere in a request, a cookie or a database column is a five-second
identification that someone is passing serialized Java objects around. Put that string
in your grep muscle memory.

### Proof 2 — `serialVersionUID`, computed versus declared

```bash
# Does the standalone tool still ship with your JDK?
serialver -classpath . Dump\$Line
```

I am **not** certain `serialver` is present in every modern JDK distribution — it has
been a candidate for removal for years.

| What you see | What it means |
|---|---|
| It prints a `serialVersionUID = ...L;` line | The tool is present. Use it to record the computed UID of a class before you change it. |
| `command not found` | Fine, and expected on some builds. Use the programmatic form below instead. |

The programmatic form, which always works:

`Uid.java`:
```java
import java.io.ObjectStreamClass;
import java.io.Serializable;

public class Uid {
    static class V1 implements Serializable { long orderId; }
    static class V2 implements Serializable { long orderId; String promotionCode; }

    public static void main(String[] args) {
        System.out.println("V1 uid = " + ObjectStreamClass.lookup(V1.class).getSerialVersionUID());
        System.out.println("V2 uid = " + ObjectStreamClass.lookup(V2.class).getSerialVersionUID());
    }
}
```

```bash
java Uid.java
```

**What to look for:** whether the two numbers differ.

| What you see | What it means |
|---|---|
| Two different large numbers | Confirmed: adding one field changed the computed UID. Any old serialized `V1` will now be rejected by `V2`. This is Trap 1, demonstrated in four lines. |
| The same number | You accidentally declared an explicit UID on both, or the classes are structurally identical. Re-read the source. |

Now add `private static final long serialVersionUID = 1L;` to **both** and re-run. Both
print `1`. You have just converted a loud incompatibility into a silent one — which is
exactly what the fix does, and exactly why the fix requires discipline rather than just
a line of code.

### Proof 3 — deserialization skips your constructor

Use `LoyaltyToken` from Example 1 and run `RoundTrip`.

**What to look for:** the order and count of the printed lines.

| What you see | What it means |
|---|---|
| `[constructor]` once, `[readObject]` after the "about to deserialize" line, `[constructor]` **not** repeated | The expected result. Your validation ran on the way out and not on the way back in. Every invariant your constructor enforces is unenforced for deserialized objects. |
| `[constructor]` twice | You are deserializing a **record**, or the class has a `readResolve` that constructs. Records are the exception — since Java 16 they deserialize via the canonical constructor, which is a genuine improvement and a strong reason to prefer them (Topic 27). |

**Confirm the record difference yourself**, because it is the most useful fact in this
proof. Change `LoyaltyToken` from a class to:

```java
public record LoyaltyToken(String customerEmail) implements Serializable {
    public LoyaltyToken {
        if (customerEmail == null || !customerEmail.contains("@")) {
            throw new IllegalArgumentException("invalid email");
        }
        System.out.println("[compact constructor] validated " + customerEmail);
    }
}
```

Re-run. If the compact constructor now prints on the way *in* as well, you have proven
that records fix the constructor-bypass problem. That single fact is why "use records
for anything serializable" is good advice rather than fashion.

### Proof 4 — the filter, from the outside

```bash
# Reject everything.
java -Djdk.serialFilter='!*' RoundTrip

# Allow only your own class.
java -Djdk.serialFilter='LoyaltyToken;!*' RoundTrip

# Add graph limits.
java -Djdk.serialFilter='maxdepth=5;maxrefs=100;LoyaltyToken;!*' RoundTrip
```

**What to look for:** in the rejecting run, an `InvalidClassException` and, critically,
**the absence of the `[readObject]` line**.

| What you see | What it means |
|---|---|
| `java.io.InvalidClassException: filter status: REJECTED` and no `[readObject]` output | The correct and important result. The filter ran **before** the class was loaded and before `readObject` was invoked. This is the only control that acts earlier than the attacker's code. |
| The exception appears but `[readObject]` printed first | Would mean the filter ran too late to help. If you see this, capture the exact command and JDK version — it contradicts the documented design and is worth investigating carefully. |
| Everything succeeds with `!*` | The property name is misspelled, or it is being overridden. Check `JAVA_TOOL_OPTIONS` and `_JAVA_OPTIONS`, and print the effective value with `System.getProperty("jdk.serialFilter")` at startup. |

To see the filter in use inside a running service:

```bash
jcmd <pid> VM.system_properties | grep -i serialfilter
```

### Proof 5 — Jackson tolerance settings

`JacksonTolerance.java`:
```java
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.annotation.*;

public class JacksonTolerance {
    record OrderPlacedV1(@JsonProperty("order_id") long orderId) { }

    public static void main(String[] args) throws Exception {
        String fromNewerProducer = """
            {"order_id": 4471, "promotion_code": "SUMMER25", "event_version": 2}
            """;

        ObjectMapper strict = new ObjectMapper();
        ObjectMapper tolerant = new ObjectMapper()
                .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        try {
            System.out.println("strict  : " + strict.readValue(fromNewerProducer, OrderPlacedV1.class));
        } catch (Exception e) {
            System.out.println("strict  : " + e.getClass().getSimpleName() + " - "
                    + e.getMessage().lines().findFirst().orElse(""));
        }
        System.out.println("tolerant: " + tolerant.readValue(fromNewerProducer, OrderPlacedV1.class));
    }
}
```

You need Jackson on the classpath:
```bash
mvn -q dependency:get -Dartifact=com.fasterxml.jackson.core:jackson-databind:2.17.2
CP=$(find ~/.m2/repository/com/fasterxml/jackson -name '*.jar' | tr '\n' ':')
java -cp "$CP." JacksonTolerance.java
```

**What to look for:** whether the strict mapper throws.

| What you see | What it means |
|---|---|
| strict throws `UnrecognizedPropertyException`, tolerant succeeds | Expected. This is the forward-compatibility switch. Spring Boot disables `FAIL_ON_UNKNOWN_PROPERTIES` by default; plain Jackson enables it. Know which you are running, because the failure modes are opposite. |
| Both succeed | Your Jackson version or configuration already disables the check. Confirm with `strict.isEnabled(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)`. |
| A `jackson-databind` version mismatch error | Adjust the version in the `dependency:get` line. Version choice for Jackson matters — Boot 4 standardises on Jackson 3, Boot 3 on Jackson 2, with package differences (Topic 31–32). |

---

## Failure drill

**The claim being tested:** deserializing untrusted data is not "risky parsing". It is
**letting the sender choose which code runs on your machine**, and that code runs
*before* any of your own code sees the object.

**The payload is deliberately benign** — a printed line and a file in your temp
directory. This is defensive education. You are learning to recognise and block the
mechanism, not to exploit it.

### Setup

```bash
mkdir -p ~/java-lab/19/drill && cd ~/java-lab/19/drill
```

`AuditRecord.java`:
```java
import java.io.*;
import java.nio.file.*;
import java.time.Instant;

/** An ordinary-looking Serializable class. The readObject is the whole point. */
public class AuditRecord implements Serializable {

    private static final long serialVersionUID = 1L;

    private final long orderId;
    private final String action;

    public AuditRecord(long orderId, String action) {
        this.orderId = orderId;
        this.action = action;
    }

    /**
     * The JVM calls this reflectively during readObject(), BEFORE returning the
     * object to the caller. Nothing the caller does can prevent it.
     *
     * Everything here is benign and local: one printed line, one file in the
     * system temp directory. In a real gadget chain this would be a call that
     * ends in Runtime.exec.
     */
    private void readObject(ObjectInputStream in)
            throws IOException, ClassNotFoundException {
        in.defaultReadObject();

        System.out.println(">>> [SIDE EFFECT] AuditRecord.readObject executed at " + Instant.now());

        Path marker = Paths.get(System.getProperty("java.io.tmpdir"), "orderflow-drill-marker.txt");
        Files.writeString(marker,
                "readObject ran at " + Instant.now() + " for orderId=" + orderId + System.lineSeparator(),
                StandardOpenOption.CREATE, StandardOpenOption.APPEND);

        System.out.println(">>> [SIDE EFFECT] wrote " + marker);
    }

    @Override public String toString() { return "AuditRecord[" + orderId + "," + action + "]"; }
}
```

`WritePayload.java`:
```java
import java.io.*;

public class WritePayload {
    public static void main(String[] args) throws Exception {
        try (var oos = new ObjectOutputStream(new FileOutputStream("payload.ser"))) {
            oos.writeObject(new AuditRecord(4471, "ORDER_PLACED"));
        }
        System.out.println("wrote payload.ser (" + new File("payload.ser").length() + " bytes)");
    }
}
```

`VulnerableConsumer.java`:
```java
import java.io.*;

/**
 * Stands in for any endpoint that deserializes bytes it did not create:
 * an RMI service, a JMX port, an HTTP session store, a Redis cache,
 * a Kafka consumer using a Java serialization deserializer.
 */
public class VulnerableConsumer {
    public static void main(String[] args) throws Exception {
        System.out.println("[1] consumer starting; my own code has not touched the payload yet");

        try (var ois = new ObjectInputStream(new FileInputStream("payload.ser"))) {
            System.out.println("[2] about to call readObject()");
            Object o = ois.readObject();
            System.out.println("[3] readObject() returned; NOW my code runs. Object = " + o);
        } catch (Exception e) {
            System.out.println("[!] rejected: " + e.getClass().getName());
            System.out.println("[!] message : " + e.getMessage());
        }
    }
}
```

### Step 1 — run it unprotected

```bash
rm -f "$TMPDIR/orderflow-drill-marker.txt" /tmp/orderflow-drill-marker.txt
javac AuditRecord.java WritePayload.java VulnerableConsumer.java
java WritePayload
java VulnerableConsumer
cat "${TMPDIR:-/tmp}/orderflow-drill-marker.txt"
```

**What to capture:** the complete stdout, in order, and the contents of the marker file.

**What to look for:** the position of the `[SIDE EFFECT]` lines relative to `[2]` and
`[3]`.

| What you see | What it means |
|---|---|
| `[1]`, `[2]`, `>>> [SIDE EFFECT]` ×2, `[3]` — in that order | **The result the drill exists to produce.** The side effect fired *between* your call to `readObject()` and your receipt of the object. There is no point in that sequence where your code could have inspected the payload and decided not to run it. You had no opportunity to say no. |
| `[SIDE EFFECT]` appears after `[3]` | Not possible via this path; if you see it, you called something else. Re-read your `VulnerableConsumer`. |
| The marker file exists and is timestamped | A durable, out-of-process effect of merely *reading* a byte stream. In a real gadget chain the equivalent step is `Runtime.getRuntime().exec(...)`. |
| An `InvalidClassException` before any side effect | You already have a filter set, probably via `JAVA_TOOL_OPTIONS`. Check with `echo $JAVA_TOOL_OPTIONS` and clear it for step 1. |

**Now say the conclusion out loud, in your own words, before moving on:** the class in
the payload chose which code ran. Not the consumer.

### Step 2 — extend the point: it is not *your* classes that matter

Delete `AuditRecord.class` and try again:

```bash
mv AuditRecord.class AuditRecord.class.bak
java VulnerableConsumer
mv AuditRecord.class.bak AuditRecord.class
```

| What you see | What it means |
|---|---|
| `ClassNotFoundException: AuditRecord` | The class must be on the consumer's classpath for the attack to work. **This is why gadget chains use common libraries** — Apache Commons Collections, Spring, Groovy, and so on — rather than exotic classes. The attacker does not need to ship code; they need a dangerous class you already depend on. |

That is the fact that turns this from a curiosity into a threat model. Your classpath is
your attack surface, and your transitive dependencies are part of your classpath
(**Topic 32**, **Topic 34**).

### Step 3 — block it with a filter

```bash
java -Djdk.serialFilter='!*' VulnerableConsumer
```

**What to capture:** stdout, and whether the marker file's modification time changed.

```bash
ls -l --time-style=full-iso "${TMPDIR:-/tmp}/orderflow-drill-marker.txt" 2>/dev/null \
  || stat -f '%Sm %N' "${TMPDIR:-/tmp}/orderflow-drill-marker.txt"
```

| What you see | What it means |
|---|---|
| `[1]`, `[2]`, `[!] rejected: java.io.InvalidClassException`, message containing `filter status: REJECTED` — and **no** `[SIDE EFFECT]` lines | **The result that proves the fix.** The filter is consulted with the class *description* before the class is resolved and before any of its methods run. It is the only control that acts earlier than the payload's own code. |
| The marker file's timestamp is unchanged | Confirms it: nothing executed. Compare this to step 1, where reading the file was enough to write to disk. |
| Side effects still fire | The property did not take effect. Print it at startup and check `JAVA_TOOL_OPTIONS`. |

### Step 4 — the realistic configuration: an allowlist, not a blocklist

```bash
java -Djdk.serialFilter='AuditRecord;!*'        VulnerableConsumer   # allowed
java -Djdk.serialFilter='maxdepth=3;maxrefs=20;AuditRecord;!*' VulnerableConsumer
java -Djdk.serialFilter='OtherClass;!*'         VulnerableConsumer   # rejected
```

| What you see | What it means |
|---|---|
| First run: side effects fire and `[3]` prints | Correct. `AuditRecord` is explicitly allowed, so its `readObject` runs. **Note carefully what this proves:** a filter stops *unexpected* classes; it does not make an allowed class's `readObject` safe. If your own allowed class does something dangerous, the filter will not help. |
| Third run: `filter status: REJECTED` | Correct. Everything not named is rejected by the trailing `!*`. |
| The `maxdepth`/`maxrefs` run behaves like the first | Expected for this tiny graph. Those limits defend against a different attack — a small payload that expands into an enormous graph — so lower them until you see a rejection, to learn what the message looks like before you meet it in production. |

Now do the same thing **in code**, which is what you would actually ship, because a
per-stream filter is scoped to the one place that needs it rather than to the whole JVM:

```java
try (var ois = new ObjectInputStream(new FileInputStream("payload.ser"))) {
    ois.setObjectInputFilter(
        ObjectInputFilter.Config.createFilter("AuditRecord;maxdepth=3;!*"));
    Object o = ois.readObject();
}
```

Run it with and without the `setObjectInputFilter` line and confirm you get the same
two outcomes without any command-line flag.

### What the fix proves

Write these four sentences down. They are the deliverable of the drill.

1. **The byte stream selects the code.** `readObject` on the class named *in the
   payload* ran before the consumer's own code saw anything. Not "could run" — did run,
   and you have a timestamped file proving it.
2. **Your classpath is the attack surface.** Step 2 showed the class must be present.
   That is why the exploit library is a *gadget* catalogue of common dependencies rather
   than attacker-supplied code, and why a transitive dependency you never chose is part
   of your threat model.
3. **A filter is the only control that runs early enough.** Validation after
   `readObject` returns is too late by definition. `-Djdk.serialFilter` and
   `setObjectInputFilter` act on the class description, before resolution.
4. **A filter is a mitigation, not a fix.** Step 4 showed an allowed class still runs
   its `readObject`. The actual fix is to not deserialize untrusted Java objects at all
   — use a format that has no concept of "which code to run".

### Where this lives in the real world

Run this checklist against any Java service you are responsible for:

```bash
# 1. Is anything on the classpath deserializing?
grep -rn "ObjectInputStream\|readObject\|Externalizable" --include=*.java src/

# 2. Is a filter configured anywhere?
grep -rn "jdk.serialFilter\|setObjectInputFilter\|setSerialFilterFactory" .

# 3. Are serialized objects crossing a boundary?
#    Look for base64 starting rO0AB in cookies, headers, DB columns, cache values.
grep -rn "rO0AB" logs/ 2>/dev/null
```

The realistic sources in a Spring service: HTTP session replication, an old Redis or
Hazelcast serializer, a JMX/RMI port left open, a Kafka `JavaSerializer`, and a
`@Cacheable` store configured with the JDK serializer.

---

## Practice exercises

### 1 — Easy: read the bytes and break the version

**Part A.** Serialize an `OrderSnapshot` with three fields. Dump the file with `xxd` and
identify, by byte offset: the magic number, the stream version, the class name, and the
field names. Write down what you found.

**Part B.** Add a fourth field to the class, recompile **only the reader**, and
deserialize the file you wrote earlier. Capture the full `InvalidClassException`,
including both UID values.

**Part C.** Add `private static final long serialVersionUID = 1L;` to both versions,
regenerate the payload with the three-field version, and read it with the four-field
version. What is the new field's value? What happens if you now write with the
four-field version and read with the three-field one?

**Part D.** State in two sentences what declaring the UID actually bought you, and what
new obligation it created.

### 2 — Medium: the object that should not exist (combines Topics 10, 13, 16, 17)

Build a `Product` class that is **immutable and validated** in the style of Topic 17:
`final` class, `final` fields, constructor validation rejecting a blank SKU and a
negative price, a defensively-copied `List<String> tags`, and `equals`/`hashCode` on the
SKU (Topic 13).

**Part A.** Prove the constructor's guarantees hold: show that
`new Product("", -5, null)` throws.

**Part B.** Serialize a valid `Product`. Then, using a hex editor or a small program
that patches the bytes, change the serialized price to a negative value and the SKU to
an empty string. Deserialize it. Show that you now hold a `Product` instance that
violates every one of its own invariants, and that no exception was thrown.

*(If byte patching is fiddly, achieve the same thing by temporarily giving the class a
second constructor that skips validation, serializing from that, then removing it. The
point is the same: the stream can produce a state your constructor forbids.)*

**Part C.** Put the corrupted `Product` in a `HashSet` alongside a valid one with the
same SKU. Show what happens to `contains`, `remove` and `size` (Topic 13). Then put it
in an `EnumMap`-keyed category index and describe what a downstream consumer sees
(Topic 16).

**Part D.** Fix it three ways, and rank them:
1. Add a `readObject` that re-runs validation.
2. Add `readResolve` that returns a validated replacement.
3. Convert `Product` to a `record` and rely on the canonical constructor.

For each, say what it costs and what it still fails to protect against. Then say which
one you would actually put in a PR and why.

### 3 — Hard: production simulation — surviving a rolling deploy

Simulate the `orderflow` order-placed event across a version change, with old and new
code running at the same time.

**Part A — the harness.** Write two source trees, `v1/` and `v2/`, each with its own
`OrderPlaced` type and a `Producer` and `Consumer` main class that exchange payloads
through a directory on disk (standing in for Kafka). Compile them into two separate
directories so you can run any producer against any consumer.

**Part B — the matrix.** For each of these five changes, run all four combinations
(v1→v1, v1→v2, v2→v1, v2→v2) and record the exact outcome of each — success, exception
class and message, or silently-wrong data:

1. Add an optional field.
2. Remove a field.
3. Rename a field (`dedupeKey` → `idempotencyKey`).
4. Change a field's type (`int quantity` → `long quantity`).
5. Change a field's *meaning* without changing its name or type (`total` goes from
   pounds to pence).

Run the whole matrix **twice**: once with Java serialization, once with Jackson JSON.
That is 40 runs. Build it as a script.

**Part C — read the results.** Produce a table: change × format × direction → outcome.
Then answer:
- Which changes are safe in JSON and unsafe in Java serialization?
- Which are unsafe in **both**?
- Which failures were **loud** (an exception) and which were **silent** (wrong data)?
- Which of the two failure modes would you rather have in production, and why? Justify
  it in terms of what your on-call engineer sees at 3am.

Change 5 is the one to think hardest about. No format catches it. Say what does.

**Part D — the safe rename.** Implement the three-deploy rename for change 3 in the
Jackson version, and prove it by running the full four-way matrix at each of the three
stages. Nine runs, all of which must succeed.

**Part E — lock it down.** Add a `jdk.serialFilter` to the Java-serialization consumer
that allows only your own event classes plus the JDK types they need. Get it working,
then deliberately break it — add a field of a type your filter forgot — and capture the
rejection. Write the one-line runbook entry that would let an on-call engineer diagnose
that message in thirty seconds.

**Part F — the recommendation.** Write the paragraph you would put in an architecture
decision record recommending a wire format for `orderflow` events. It must cite at least
two numbers from your matrix and must state what would change your mind.

---

## Interview questions

### Q1 — "Why is deserializing untrusted data dangerous? `JSON.parse` isn't."

**Mid-level answer:** "Because an attacker can send a malicious object and the code
that deserializes it might do something bad. You should validate input."

**Senior answer:** "Because it is not parsing — it is code execution by design.
`JSON.parse` produces only plain data; nothing in the payload can select which code
runs. Java's `ObjectInputStream` reads a *class name* out of the stream, loads that
class, allocates an instance without calling its constructor, populates the fields, and
invokes that class's own private `readObject`. So the sender chooses which methods run,
from every class on my classpath. A gadget chain strings ordinary library classes
together — Commons Collections' `InvokerTransformer` is the classic — through their
`readObject`, `hashCode` and `toString` methods until it reaches something that executes
a process. Validating after `readObject` returns is too late by definition: the code has
already run. That's why the JDK added serialization filters, which act on the class
description before resolution, and why the real answer is to use a format that has no
concept of 'which code to run'."

**What separates them:** "validate the input" versus "validation cannot help because it
runs after the exploit". Naming the ordering is the whole answer.

**Follow-up:** "So a filter fixes it?" It mitigates. An allowed class still runs its own
`readObject`, and a blocklist of known gadgets is a losing race. Say that unprompted.

---

### Q2 — "What is `serialVersionUID` and what happens if you leave it out?"

**Mid-level answer:** "It's a version number for the class. If you don't declare it,
Java generates one, and you get an exception if the versions don't match."

**Senior answer:** "It's a compatibility contract. If you don't declare it, the JVM
computes it by hashing the class's structure — name, modifiers, interfaces, fields and
some method signatures — so almost any change produces a different value and old data
fails with `InvalidClassException: local class incompatible`, which quotes both UIDs.
The important part is what declaring it actually does: it *suppresses the version
check*, so the JVM falls back to default field matching — fields in the stream but not
the class are discarded, fields in the class but not the stream keep their default
value. That converts a loud failure into a silent one. It's the right change, but only
if you also enforce the compatible-change rules: adding and removing fields is fine, a
rename or a type change is not, and a rename fails silently as a null rather than
throwing. I'd also turn on `-Xlint:serial` so a missing UID fails CI rather than
production."

**What separates them:** knowing that declaring the UID trades a loud failure for a
silent one, and treating that as a discipline rather than a fix.

**Follow-up:** "How would you catch an incompatible change before it ships?" Golden
payloads from every released version, asserted in CI.

---

### Q3 — "Walk me through renaming a field in an event schema during a rolling deploy."

**Mid-level answer:** "You'd rename it and deploy. If it's a problem you could add a
backwards-compatible alias."

**Senior answer:** "It takes three deploys, and there's no two-deploy version. The
constraint is that during a rollout, old and new instances run simultaneously and traffic
flows *both* ways — old producers to new consumers and new producers to old consumers —
so the schema must be readable by both for the whole window.
Deploy one: add `@JsonAlias` for the new name while still writing the old one, so
everything can read both. Deploy two: switch the write side to the new name; both
spellings are still readable. Deploy three: once no message carrying the old name can
still exist, remove the alias — and for Kafka that means after the topic's retention
window, not after the deploy completes.
The failure mode if you skip it depends on configuration, and the *safer* configuration
is the one that looks worse: with `FAIL_ON_UNKNOWN_PROPERTIES` on you get a visible
error spike for the rollout duration, and with it off — Spring Boot's default — the
field arrives as null and you get silently duplicated payments. I'd rather have the
spike. Same three-step shape applies to a database column rename and to a protobuf
field, so it's worth learning as a pattern rather than as a Jackson trick."

**What separates them:** knowing traffic flows both ways during a rollout, knowing
retention determines deploy three's timing, and preferring the loud failure.

**Follow-up:** "What about changing a field's *meaning* — pounds to pence — with the
same name and type?" No format catches it. Only a version field plus a consumer that
branches on it, or a new field name. This is the question that separates people who
have actually done this from people who have read about it.

---

### Q4 — "Is Java serialization deprecated?"

**Mid-level answer:** "I think so — everyone says not to use it."

**Senior answer:** "Not formally, and I'd rather be precise. As of Java 21
`java.io.Serializable` carries no `@Deprecated` annotation and `implements Serializable`
compiles without a warning; there's no announced removal date. What has actually
happened is a containment strategy: serialization filters in Java 9, context-specific
filters in Java 17, records deserializing through their canonical constructor from
Java 16 so validation finally runs, and a clear statement from the JDK team that it was
a design mistake. So the honest position is 'strongly discouraged and being fenced in,
not removed'. Practically it doesn't change my behaviour — I don't put it on a wire, I
don't persist it, and where I inherit it I put a filter on it and plan a migration. If
someone needed the current status for a compliance document I'd check the JEP index
rather than quote from memory, because this is exactly the kind of thing that moves
between releases."

**What separates them:** refusing to overstate, distinguishing "deprecated" from
"discouraged", and volunteering how to verify rather than asserting.

**Follow-up:** "Where would you still legitimately use it?" Honest answers: inside a
single JVM's lifetime where both ends are the same build — some caching and clustering
libraries — and even then with a filter. Anything else is a migration candidate.

---

### Q5 — "You inherit a service that deserializes Java objects from a Redis cache. What do you do, in what order?"

**Mid-level answer:** "Replace it with JSON."

**Senior answer:** "Eventually, yes, but not first — I'd sequence by risk and by what's
independently shippable.
Day one: establish who can write to that Redis. If it's network-reachable beyond the
service, that's an unauthenticated code-execution path into production and it's an
incident, not a backlog item. Lock the network path and rotate credentials.
Day two: add a strict allowlist filter — `setObjectInputFilter` on the specific stream
rather than a JVM-wide property, so it's scoped and reviewable — plus the `maxdepth`
and `maxbytes` limits, because a class allowlist doesn't stop a graph-expansion denial
of service. That's a small, revertible change that removes most of the risk today.
Then audit the classpath: `mvn dependency:tree` for known gadget libraries, because the
threat is proportional to what's reachable, not to what I wrote (Topic 34).
Then migrate the format, and that's the slow part — it needs a dual-read window where
the code can read both the old binary and the new JSON, and a cache warm-up plan,
because switching format cold empties the cache and puts full read load on the database.
And I'd note the third problem out loud: even without an attacker, the current setup
breaks on every deploy that changes those classes, so this is a reliability fix as well
as a security one. That usually gets it prioritised faster than the security argument
alone."

**What separates them:** sequencing by risk, distinguishing the mitigation from the fix,
knowing a cold cache switch is its own outage, and knowing which argument gets the work
scheduled.

**Follow-up:** "How do you find every deserialization point?" Grep for
`ObjectInputStream` and `readObject`, but also check framework configuration — session
replication, `@Cacheable` serializers, JMX and RMI ports, and any Kafka
`JavaSerializer`.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `readObject` is `private`, and yet the JVM calls it. What does that tell you about
   how much of Java's access control the serialization mechanism respects? Name one
   other consequence of that beyond the security issue.

2. Records deserialize through their canonical constructor; ordinary classes do not.
   Explain why the platform could make that change for records but could not
   retroactively make it for every `Serializable` class.

3. A serialization filter is consulted with the class *description* before the class is
   resolved. Why is that ordering essential rather than merely convenient? What would a
   filter that ran one step later fail to prevent?

4. `transient` produces a null with no exception. `serialVersionUID` mismatch produces
   an exception with no data loss. Which design is better, and does your answer change
   depending on whether the reader is a service or a batch job?

5. Jackson's default typing recreates Java serialization's vulnerability in JSON. Given
   that, is the vulnerability a property of the *format* or of the *configuration*?
   What does your answer imply about protobuf and Avro?

6. `JSON.parse` is safe because its output alphabet is small. State the general
   principle that makes a deserializer safe, in one sentence, without mentioning Java or
   JSON. Then apply it to YAML, and say why YAML parsers have had the same class of CVE.

7. You cannot make a rename compatible in a single deploy. Is that a limitation of
   serialization formats, or a property of distributed systems? Argue one side, then
   make the strongest case against yourself.

---

## Quick reference card

### The three formats

| | Java serialization | Jackson JSON | Protobuf / Avro |
|---|---|---|---|
| Sender picks the class | **yes** | only with default typing | no |
| RCE risk on untrusted input | **high** | low, unless default typing | very low |
| Schema evolution | fragile, manual | good with aliases + ignoreUnknown | designed for it |
| Human readable | no | yes | no |
| Size | large | medium | small |
| Cross-language | no | yes | yes |
| Runs your constructor | no (records: yes) | yes for records / `@JsonCreator` | yes (generated code) |
| Use it for | almost nothing new | REST APIs, config, events | cross-service events at volume |

### Serialization keywords

```java
implements Serializable                 // marker; inherited by subclasses
private static final long serialVersionUID = 1L;   // always declare it
transient                               // excluded from the stream; reads back as default
private void writeObject(ObjectOutputStream)       // custom write
private void readObject(ObjectInputStream)         // custom read — runs before your code
private void readObjectNoData()                    // no data in stream for this class
private Object writeReplace()           // substitute on the way out
private Object readResolve()            // substitute on the way in (singletons)
implements Externalizable               // you write both directions; needs a public no-arg ctor
```

### Filter syntax

```
com.orderflow.**        allow package and subpackages
com.orderflow.*         allow package only
!com.evil.*             reject
!*                      reject everything else — always last
maxdepth=N  maxrefs=N  maxbytes=N  maxarray=N
```

```bash
-Djdk.serialFilter='com.orderflow.**;java.base/*;maxdepth=20;!*'
```
```java
ois.setObjectInputFilter(ObjectInputFilter.Config.createFilter("com.orderflow.**;!*"));
ObjectInputFilter.Config.setSerialFilterFactory(factory);   // Java 17+, once at startup
```

### Jackson evolution toolkit

```java
@JsonProperty("wire_name")                    // decouple wire name from field name
@JsonAlias({"oldName"})                       // accept old names on read — the rename tool
@JsonIgnoreProperties(ignoreUnknown = true)   // tolerate new fields — forward compatibility
@JsonIgnore                                   // never write
@JsonCreator                                  // route through a validating constructor
@JsonTypeInfo(use = Id.NAME) + @JsonSubTypes  // safe polymorphism (never use Id.CLASS)
```

### Recognising it in the wild

- Bytes beginning `AC ED 00 05` — a Java serialization stream.
- Base64 beginning `rO0AB` — the same thing, encoded. Grep for it in cookies, headers,
  logs and database columns.

### Gotchas

- [ ] Never deserialize Java objects from anywhere you do not fully control.
- [ ] Always declare `serialVersionUID`; turn on `-Xlint:serial`.
- [ ] Never rename a field in one deploy. Three deploys, always.
- [ ] Every `transient` field needs a comment saying what restores it.
- [ ] Never use Jackson default typing. `Id.NAME` with explicit subtypes only.
- [ ] Prefer records: they deserialize through the canonical constructor, so validation
      and defensive copies actually run.
- [ ] Filters are a mitigation. Changing the format is the fix.
- [ ] Adding a field is safe only if consumers treat it as optional.

---

## When would I use this at work?

**1. Reviewing any change to an event, DTO or cache-value class.**
The question you now ask on every such PR: "what happens during the rollout window when
both versions are running?" A rename, a type change or a new required field are all
outage-shaped, and all three look like one-line diffs. Catching them in review is
minutes of work; catching them in production is an incident and a backfill.

**2. Responding to a security finding or a penetration test.**
"Java deserialization of untrusted input" appears in real reports. You will need to
answer three questions fast: is there a deserialization point, can an attacker reach it,
and what is on the classpath. Then you apply the mitigation today (a scoped filter with
graph limits) and schedule the fix (format migration with a dual-read window). Being
able to separate those two, and to say honestly that the filter is not the fix, is the
senior behaviour.

**3. Choosing a wire format at design time.**
Someone proposes storing objects in Redis, or publishing events with a Java serializer
because "it is one line". You now have the three-sentence argument: it couples both
sides to the same build, it turns every schema change into a deploy hazard, and it turns
the store into a code-execution path. That conversation takes five minutes at design
time and six months of migration later.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and wrappers**: money as `long` minor units in an event payload;
  `BigDecimal` without a currency is not a schema.
- **08 / 09 — Exceptions**: `InvalidClassException` is checked; how you surface a
  deserialization failure at a service boundary is a design decision.
- **13 — equals/hashCode contract**: a gadget chain typically travels through
  `hashCode` and `equals`, which is why they are invoked at all during deserialization.
- **17 — Immutability**: deserialization bypasses your constructor, so it bypasses your
  defensive copies and validation. This topic is where that becomes a security property
  rather than a tidiness one.
- **18 — Strings**: serialized payloads are string-heavy; the `rO0AB` recognition
  trick is base64 of the magic bytes.

**This unlocks:**
- **27 — Records**: the canonical-constructor deserialization that fixes the
  invariant-bypass problem, and the main reason to prefer records for wire types.
- **28 — Sealed types**: the safe form of polymorphic deserialization — a closed set the
  compiler knows about, rather than a class name from the payload.
- **32 / 34 — Dependency resolution and supply chain**: your classpath is the gadget
  catalogue. Triaging a deserialization CVE is exactly the reachability question Topic 34
  teaches.
- **45 — Bean Validation**: the layer that runs *after* Jackson and does the job that
  Java serialization skipped entirely.
- **46 — Error handling**: how a deserialization failure becomes an HTTP response
  without leaking the class names in your stack trace.
- **79 — Memory leaks**: a deep object graph deserialized from a large payload is a
  denial-of-service vector; `maxdepth`/`maxbytes` are the bound.
- **83 — GraalVM native image**: reflection-driven deserialization needs explicit
  reachability metadata, so this topic's mechanisms are exactly what breaks there.
- **113 / 115 / 116 — Kafka, outbox, idempotency**: where `orderflow`'s event schema
  evolution actually bites, and where the idempotency key from Trap 3 earns its keep.
- **123 — Rolling deploys**: the deploy mechanics that make Trap 2 a weekly rather than
  a theoretical concern.

---

*Java baseline 21. Serialization filters (Java 9) and context-specific filters
(Java 17) are both available on 21 and 25. Record deserialization via the canonical
constructor has been in place since Java 16. The two points I have deliberately not
asserted from memory are the deprecation status of `java.io.Serializable` — settle it
with `javac -Xlint:all` as shown above — and specific CVE numbers, which you should pull
from nvd.nist.gov if you need to cite them. Everything else here is stable and has been
for years.*
