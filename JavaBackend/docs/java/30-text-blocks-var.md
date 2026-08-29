# 30 — Text Blocks, `var`, and Where Ergonomics Hurt Readability

## Phase: 2 — Modern Java
## Category: CORE
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: N/A (spine starts at Topic 35)

---

## ELI5 anchor

Two conveniences, and the same warning attached to both.

**`var` is not writing the label on the box.**

You are stacking boxes. Normally you write what is inside on the side of each one:
`Order`, `List<Payment>`, `Map<String, BigDecimal>`. With `var` you skip the label and
say "whatever is inside — you can see it when you look".

That is fine when the box is transparent: `var order = new Order()`. Anybody can see
what is in it. It is not fine when the box is opaque: `var result = service.process()`.
Now the next person has to go and find `process()`, read its signature, and come back.
You saved five keystrokes and cost them thirty seconds. Do that forty times in a file
and the file becomes unreadable.

**A text block is a sheet of paper you can write on freely.**

Instead of taping together twenty small notes with `+` and `\n`, you write the whole
thing out. The paper keeps your line breaks.

The catch: the machine decides where the left margin is by looking at your **least
indented line, including the closing marker**. Move the closing marker and the margin
moves. Which means somebody re-indenting your method — an IDE reformat, wrapping the
code in an `if` — can silently change the text your program produces.

And the thing that is genuinely missing: the paper does **not** let you drop values into
it the way a template literal does. No `${...}`. Java tried and withdrew it. You use
`formatted()` or you concatenate.

---

## The bridge from what you know

### `var` versus TypeScript inference

```ts
const order = new Order();               // inferred Order, cannot be reassigned
let total = 0;                           // inferred number, can be reassigned
const items: OrderLine[] = [];           // explicit annotation when inference isn't enough

function price(order: Order): number {   // parameters and returns MUST be annotated
  return order.total;
}
```

```java
var order = new Order();                 // inferred Order, CAN be reassigned
var total = 0;                           // inferred int, CAN be reassigned
final var items = new ArrayList<OrderLine>();   // `final var` is your `const`

long price(Order order) {                // parameters and returns MUST be explicit
    return order.total();
}
```

### The verdict

| TypeScript | Java `var` | Verdict |
|---|---|---|
| Inference on local declarations | `var` on local declarations | **HONEST ANALOGUE** — same mechanism, same benefit. |
| `const` vs `let` distinction | `final var` vs `var` | **PARTIAL** — Java has it, but `final` is opt-in and verbose, so almost nobody writes it. TypeScript's default is `const`; Java's default is mutable. |
| Inference on class fields (`class A { x = 5 }`) | **not allowed** | **NO ANALOGUE** — `var` is local-only. |
| Inference on function return types | **not allowed** | **NO ANALOGUE** — Java return types are always explicit. |
| Parameter contextual typing in callbacks | lambda params: `(var a, var b) ->` or just `(a, b) ->` | **PARTIAL** — Java lambda params were already inferable without `var`. |
| `let x;` then assign later | `var x;` — **compile error** | **NO ANALOGUE** — `var` needs an initialiser, on the same line. |
| `const x = null` | `var x = null;` — **compile error** | **NO ANALOGUE** — there is no type to infer from `null`. |

**The honest summary:** `var` is TypeScript inference with three holes cut in it —
local-only, no `const` default, and it cannot infer from `null` or a bare declaration.
Everything else transfers.

### Text blocks versus template literals

```ts
const sku = "SKU-1001";
const query = `
  select id, status
  from orders
  where sku = '${sku}'
`;                                       // interpolation, multi-line, one construct
```

```java
String sku = "SKU-1001";
String query = """
        select id, status
        from orders
        where sku = '%s'
        """.formatted(sku);              // multi-line YES, interpolation NO
```

| TypeScript | Java text block | Verdict |
|---|---|---|
| Multi-line string literal | `"""` text block | **HONEST ANALOGUE** |
| Preserved line breaks and indentation | preserved, minus "incidental" indentation | **PARTIAL** — Java strips a common left margin. TypeScript does not. |
| `${expression}` interpolation | **nothing** | **NO ANALOGUE** — see below. |
| Tagged templates (`` sql`...` ``) | **nothing** | **NO ANALOGUE** |
| `String.raw` | `\` escapes still apply; no raw mode | **PARTIAL** |

### Say it plainly: Java has no string interpolation, and it is not coming soon

Java previewed **string templates** (`STR."Hello \{name}"`) in Java 21 and again in
Java 22. In Java 23 the feature was **withdrawn** — removed entirely, not deferred with
a plan. The design team said the syntax and the processor model needed rethinking.

So on Java 21 **and** on Java 25, your options are:

```java
// 1. formatted() - the idiomatic choice for a text block
String msg = """
        Order %s for user %s totalling %d pence
        """.formatted(orderId, userId, totalMinor);

// 2. String.format - the same thing, older shape
String msg = String.format("Order %s for user %s", orderId, userId);

// 3. Concatenation - fine for two or three pieces
String msg = "Order " + orderId + " for user " + userId;

// 4. MessageFormat - when you need i18n with positional arguments
String msg = MessageFormat.format("Order {0} for user {1}", orderId, userId);
```

Do not go looking for `STR."..."` in the docs and conclude you have the wrong JDK. It
is gone. If a tutorial shows it, that tutorial is from 2023 or 2024.

---

## What is this?

Two independent Java features that share a theme: they make code shorter, and shorter
is not always clearer.

**`var`** (Java 10, final) is **local variable type inference**. You write `var`
instead of a type, and the compiler works the type out from the initialiser. It is a
**compile-time** feature only — the variable is still statically, strongly typed, and
the exact type is recorded in the class file. Nothing becomes dynamic.

**Text blocks** (Java 15, final) are multi-line string literals delimited by `"""`.
They preserve line breaks, strip a common left margin, and support the usual escapes.
They produce an ordinary `String` — there is no new type and no runtime cost.

Both are final in Java 15/10 respectively, so on a Java 21 baseline neither needs any
flag.

---

## Why does it matter?

**1. `var` is the most-argued-about feature in modern Java, and the arguments are
mostly about taste — until you learn the rule.**
Teams waste review cycles on "is `var` okay here". There is a defensible rule and it is
in this document. Learning it means you stop having the argument.

**2. `var` has three genuine traps that are not about taste at all.**
`new ArrayList<>()` inferring `ArrayList<Object>`, numeric literal inference silently
choosing `int`, and a method's return-type change silently changing your local's type.
All three produce real bugs and none of them are style opinions.

**3. Text blocks changed how Java code looks, and made a whole category of code
readable.**
SQL, JSON fixtures, GraphQL queries, HTML fragments, shell scripts, YAML. Before text
blocks these were unreadable ladders of `"..." + "\n" +`. That readability gain is
large and unambiguous.

**4. Text blocks have exactly one non-obvious rule — incidental whitespace — and it
bites through code formatting.**
An IDE reformat, a merge, or wrapping code in one more `if` can change the string your
program produces. In a test that compares JSON exactly, that is a failing test with a
diff that looks identical. In a production payload, it is worse.

**5. Java's ergonomics story has holes you should know about rather than trip over.**
No interpolation, no `const` by default, no inference outside locals. Knowing what
is *not* there stops you searching for it.

---

## Syntax breakdown

### `var` — where it is allowed

```java
// local variables with an initialiser
var order = new Order(orderId, userId);

// enhanced for loop
for (var line : order.lines()) { ... }

// classic for loop
for (var i = 0; i < lines.size(); i++) { ... }

// try-with-resources
try (var stream = Files.lines(path)) { ... }

// lambda parameters (Java 11+) - only useful to attach an annotation
list.forEach((@NonNull var item) -> process(item));

// final locals - THIS is your `const`
final var maxRetries = 3;
```

### `var` — where it is NOT allowed

```java
class Order {
    var id;                              // ERROR: fields cannot use var
}

void place(var command) { }              // ERROR: parameters cannot use var

var findOrder(OrderId id) { }            // ERROR: return types cannot use var

var x;                                   // ERROR: needs an initialiser
var y = null;                            // ERROR: nothing to infer from null
var z = { 1, 2, 3 };                     // ERROR: array initialiser needs a type
var f = () -> System.out.println("x");   // ERROR: lambda has no standalone type
var g = String::length;                  // ERROR: same reason
catch (var e) { }                        // ERROR: catch parameters cannot use var
```

| Rule | Why |
|---|---|
| Locals only | Fields and method signatures are **API**. Inferring them would mean a change to a private helper could silently change a public signature. Locals are private to one method body, so inference is contained. |
| Needs an initialiser | Nothing to infer from otherwise. |
| Not `null` | `null` has the *null type*, which you cannot declare. |
| Not a bare lambda or method reference | A lambda's type comes from its **target**; there is no target when the target is being inferred. Circular. |

**Java 25 note:** the *contextual keyword* nature of `var` is unchanged. A variable
named `var` still compiles for backward compatibility, and so does a method named
`var`. Do not do that.

### `final var` — Java's `const`

```java
var       total = 0;                 // like TypeScript's `let`
final var maxRetries = 3;            // like TypeScript's `const`
```

Your TypeScript habit is `const` by default and `let` when you must. Java's default is
mutable, and `final var` is eleven characters. Most teams do not write it, and honestly
that is a defensible choice — the cost of not having it is low for a local inside a
short method. **Be aware it exists**, use it where a local genuinely must not be
reassigned (a lock object, a computed constant, a value captured by a lambda), and do
not fight your team about it.

> Related: a local captured by a lambda or anonymous class must be **effectively final**
> anyway — assigned once and never reassigned — so the compiler already enforces the
> important half of `const` where it matters most (Topic 21).

### Text blocks — the delimiters

```java
String json = """
        {"sku": "SKU-1001", "quantity": 2}
        """;
```

| Bit of syntax | What it means |
|---|---|
| `"""` opening | Must be followed by a **line terminator**. `String s = """abc"""` does not compile. |
| The content lines | Everything until the closing delimiter. |
| `"""` closing | Its own indentation **participates** in the margin calculation. This is the whole trap. |
| Result | An ordinary `String`. There is no `TextBlock` type. |

### Incidental whitespace — the exact algorithm

This is the one rule you must actually know. It has four steps:

1. Take all **non-blank content lines**, plus the line containing the **closing
   delimiter** if that delimiter is on its own line.
2. Find the **minimum leading-whitespace count** across those lines. That is the
   *incidental* indentation.
3. Remove exactly that many leading whitespace characters from **every** line.
4. Remove all **trailing** whitespace from every line.

Worked example:

```java
String s = """
        line one
          line two
        """;
```

Lines considered: `        line one` (8 spaces), `          line two` (10 spaces), and
the closing delimiter line `        ` (8 spaces). Minimum is 8. Strip 8 from each:

```
line one
  line two
```

Note `line two` keeps its extra 2 spaces — that indentation is *essential*, not
incidental. And note the result **ends with a newline**, because there is a line break
after `line two` before the closing delimiter.

Now move only the closing delimiter to column 0:

```java
String s = """
        line one
          line two
""";
```

Minimum is now 0. Nothing is stripped:

```
        line one
          line two
```

**Same content lines. Completely different string.** That is Trap 4.

### Suppressing the trailing newline, and other escapes

```java
// \ at end of line: join with the next line, no newline emitted
String oneLine = """
        select id, status \
        from orders \
        where user_id = ?""";
// -> "select id, status from orders where user_id = ?"

// no trailing newline: put the closing delimiter on the last content line
String noNewline = """
        {"ok": true}""";

// \s : a literal space that is NOT stripped as trailing whitespace
String padded = """
        value:\s\s\s
        """;
// -> "value:   \n"

// escapes still work
String quoted = """
        He said "hello" and \\ is a backslash
        """;

// three quotes inside a text block: escape at least one
String triple = """
        The delimiter is \"""
        """;
```

| Escape | Effect |
|---|---|
| `\` at end of line | Suppress that line's newline (line continuation). |
| `\s` | A space that survives trailing-whitespace stripping. |
| `\n`, `\t`, `\"`, `\\` | Work exactly as in a normal string literal. |
| `\"""` | How you write three quotes inside a text block. |

### `formatted()` — the interpolation substitute

```java
String body = """
        {
          "orderId": "%s",
          "totalMinorUnits": %d,
          "currency": "%s"
        }
        """.formatted(orderId, totalMinor, currency);
```

`formatted(Object...)` was added in Java 15 alongside text blocks. It is exactly
`String.format(this, args)` as an instance method, so it reads left-to-right with the
template first — which is the whole point.

**Format specifier crib:** `%s` any object (calls `toString`), `%d` integral, `%f`
floating point, `%.2f` two decimal places, `%n` platform newline, `%%` a literal percent.
Getting the count or the type wrong is a **runtime** `IllegalFormatException`, not a
compile error — which is the main cost of not having real interpolation.

### `[JAVA 25]` Module import declarations — JEP 511

```java
// [JAVA 25]
import module java.base;

public class Report {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>();     // java.util.* available
        Path p = Path.of("out.txt");                // java.nio.file.* available
        Map<String, Integer> counts = new HashMap<>();
    }
}
```

`import module M;` imports every package that module `M` exports, in one line.
`java.base` covers `java.util`, `java.io`, `java.nio.file`, `java.time`,
`java.util.function`, `java.util.stream` and much more.

**The 21-compatible fallback** — write the imports:
```java
import java.util.*;
import java.nio.file.*;
import java.util.stream.*;
```

> **Honesty flag.** My information is that JEP 511 finalised module import declarations
> in JDK 25 (following previews as JEP 476 in 23 and JEP 494 in 24). I am not going to
> assert that for *your* JDK. The Hands-on proof below has the command that settles it:
> compile with no flags, then with `--enable-preview`, and read what `javac` says.

**Should you use it?** In a single-file script or a teaching example, yes — it removes
noise. In production code, be cautious: a module import pulls in a very large number of
simple names, which makes ambiguity more likely (`java.util.List` versus
`java.awt.List` is the classic) and makes it harder for a reader to know where a type
came from. This is the same readability trade as `var`, one level up. Explicit imports
are a form of documentation, and your IDE writes them for you anyway.

### `[JAVA 25]` Compact source files and instance main methods — JEP 512

```java
// [JAVA 25] - a complete, runnable program
void main() {
    IO.println("hello");
}
```

No `class`, no `public static`, no `String[] args`. Run it with `java Hello.java`.

**The 21-compatible fallback:**
```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("hello");
    }
}
```

> **Honesty flag.** My information is that this finalised in JDK 25 after previews in
> 21, 22 and 23. Same settle-it command as above. This is aimed at learning and
> scripting, not at services — a Spring Boot application still has a normal `main`
> class. Worth knowing it exists so you recognise it in an article; not worth
> restructuring anything for.

---

## Example 1 — minimal

```java
import java.util.*;

public class ErgonomicsBasics {

    public static void main(String[] args) {

        // var where the right-hand side names the type: good
        var orderIds = new ArrayList<String>();
        orderIds.add("o-1001");
        orderIds.add("o-1002");

        // final var: Java's const
        final var maxDisplay = 1;

        // var in a for-each: the element type is obvious from the collection
        for (var id : orderIds) {
            System.out.println("order " + id);
        }

        // A text block. Note the closing delimiter's indentation.
        var summary = """
                Orders processed: %d
                Showing first:    %d
                """.formatted(orderIds.size(), maxDisplay);

        System.out.print(summary);

        // Prove the incidental-whitespace rule with visible markers
        var indented = """
                alpha
                  beta
                """;
        indented.lines().forEach(line -> System.out.println("[" + line + "]"));
        System.out.println("ends with newline: " + indented.endsWith("\n"));
    }
}
```

Run it. Look at the bracketed output: `[alpha]` and `[  beta]`. The 16 spaces of
source indentation are gone; the 2 extra spaces on `beta` survive. And the string ends
with a newline. Both of those are things people get wrong from memory, so see them once.

---

## Example 2 — production scenario

`orderflow` needs three multi-line strings: a SQL query for the order-summary read
path, a JSON fixture for a contract test, and an outbound webhook payload.

### The version before text blocks

```java
public class OrderSummaryRepository {

    private static final String SUMMARY_SQL =
        "select o.id, o.status, o.placed_at, " +
        "       sum(l.quantity * l.unit_price_minor) as total_minor, " +
        "       count(l.id) as line_count " +
        "from orders o " +
        "join order_lines l on l.order_id = o.id " +
        "where o.user_id = ? " +
        "  and o.placed_at >= ? " +
        "group by o.id, o.status, o.placed_at " +
        "order by o.placed_at desc " +
        "limit ?";
}
```

Every line ends with `" +` and begins with `"`. Every line needs a trailing space or
the SQL runs together — and forgetting one produces `...o.statusfrom orders...`, which
is a runtime SQL error at a line number that tells you nothing. Nobody can read this,
so nobody reviews it, so a `where` clause bug lives here for a year.

### The text block version

```java
public class OrderSummaryRepository {

    private static final String SUMMARY_SQL = """
            select o.id,
                   o.status,
                   o.placed_at,
                   sum(l.quantity * l.unit_price_minor) as total_minor,
                   count(l.id)                          as line_count
            from orders o
            join order_lines l on l.order_id = o.id
            where o.user_id   = ?
              and o.placed_at >= ?
            group by o.id, o.status, o.placed_at
            order by o.placed_at desc
            limit ?
            """;

    private final JdbcClient jdbc;

    public List<OrderSummary> summariesFor(UserId userId, Instant since, int limit) {
        return jdbc.sql(SUMMARY_SQL)
                .param(userId.value())        // BOUND parameters, not concatenated
                .param(since)
                .param(limit)
                .query(OrderSummary.class)
                .list();
    }
}
```

You can now read it. You can paste it straight into `psql`. You can diff it and see one
changed line instead of a reflowed block.

> **The hazard text blocks introduce.** They make it *pleasant* to build SQL by
> concatenation, and that is how SQL injection gets written:
> ```java
> // NEVER DO THIS
> var sql = """
>         select * from orders where user_id = '%s'
>         """.formatted(untrustedInput);
> ```
> A text block is a **template for a static query**. Every value goes in as a bound
> parameter (`?` or `:name`), always. The only thing you may interpolate is a value you
> control completely — a validated column name from an allow-list, for example — and
> even then, prefer not to. This warning is not new (Topic 47 and any SQL course will
> repeat it), but text blocks genuinely increased how often people get it wrong,
> because the ergonomics stopped fighting them.

### The JSON fixture in a contract test

```java
class PlaceOrderContractTest {

    private static final String VALID_REQUEST = """
            {
              "userId": "11111111-1111-1111-1111-111111111111",
              "idempotencyKey": "idem-abc-123",
              "lines": [
                { "sku": "SKU-1001", "quantity": 2 },
                { "sku": "SKU-2002", "quantity": 1 }
              ]
            }
            """;

    @Test
    void acceptsAValidOrder() throws Exception {
        mockMvc.perform(post("/orders")
                        .contentType(APPLICATION_JSON)
                        .content(VALID_REQUEST))
               .andExpect(status().isCreated())
               .andExpect(jsonPath("$.status").value("PENDING"));
    }
}
```

Note the assertion style: `jsonPath` on specific fields, **not** a string comparison
against another text block. That choice is deliberate and it is Trap 5 — comparing JSON
as raw strings makes your test sensitive to whitespace, key order, and the trailing
newline, none of which are part of your contract.

### `var` in a real service method — the rule applied

```java
@Transactional
public OrderPlacedResponse place(PlaceOrderCommand command) {

    // GOOD: the right-hand side names the type
    var order = new Order(OrderId.newId(), command.userId(), Instant.now());
    var reservations = new ArrayList<InventoryReservation>();

    // GOOD: the for-each element type is obvious from the collection
    for (var line : command.lines()) {
        reservations.add(inventory.reserve(line.sku(), line.quantity()));
    }

    // BAD: what is this? OrderTotal? Money? long? BigDecimal?
    // var total = pricingService.priceOrder(command);

    // GOOD: the type IS the information here, so write it
    Money total = pricingService.priceOrder(command);

    // BAD: the reader cannot tell if this is a Payment, a PaymentResult,
    //      or a CompletableFuture<PaymentResult>. That difference is enormous.
    // var payment = paymentGateway.charge(command.userId(), total);

    // GOOD
    PaymentResult payment = paymentGateway.charge(command.userId(), total);

    return switch (payment) {                       // Topic 29
        case Approved(var attemptId, var at, var ref, var amount) -> {
            orders.save(order.markPaid(ref, amount));
            yield OrderPlacedResponse.created(order.id(), ref);
        }
        case Declined d -> {
            reservations.forEach(inventory::release);
            yield OrderPlacedResponse.declined(order.id(), d.reason());
        }
        case Failed f -> {
            reservations.forEach(inventory::release);
            yield OrderPlacedResponse.failed(order.id(), f.retryable());
        }
        case PendingAuthentication(var id, var at, var url, var expires) ->
            OrderPlacedResponse.awaiting3ds(order.id(), url, expires);
    };
}
```

Read the commented-out lines against the ones that replaced them. **That contrast is
the rule**, and the next section states it.

### The rule, stated so you can apply it without taste

> **Keep `var` when the right-hand side already names the type.**
> **Drop `var` when the type is the information the reader needs.**

| Expression | `var`? | Why |
|---|---|---|
| `var order = new Order(...)` | **yes** | `new Order` says `Order`. Repeating it adds nothing. |
| `var lines = new ArrayList<OrderLine>()` | **yes** | The type is right there. |
| `var order = orders.findById(id)` | **no** | Is it `Order` or `Optional<Order>`? That difference changes every following line. |
| `var result = service.process(cmd)` | **no** | Opaque. The reader must leave the file. |
| `var total = 0` | **careful** | Infers `int`. If it should be `long`, write `long total = 0;`. See Trap 3. |
| `var id = UUID.randomUUID()` | **yes** | The factory names the type. |
| `for (var line : order.lines())` | **yes** | The collection's element type is on the same line. |
| `try (var in = Files.newInputStream(p))` | **yes** | Factory names it, and the resource type rarely matters. |
| `var status = response.getStatusCode()` | **no** | `int`? `HttpStatus`? `HttpStatusCode`? Genuinely ambiguous. |
| `var config = builder.build()` | **no** | Builders very often return something other than their own name. |

The rule fails in one place worth naming: `var` is sometimes the only way to write a
variable whose type is **non-denotable** — an anonymous class's type, or an intersection
type from a generic method. There you have no choice, and that is fine.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `var` with the diamond operator

**Wrong:**
```java
var reservations = new ArrayList<>();
reservations.add(new InventoryReservation("SKU-1001", 2));
```

**Exact symptom:** the two lines above compile. Then, later in the method:
```java
for (var r : reservations) {
    inventory.release(r.sku());      // <-- compile error here, not above
}
```
```
error: cannot find symbol
  symbol:   method sku()
  location: variable r of type java.lang.Object
```

The error is **on a different line from the mistake**, and it names `Object` in a method
where you never wrote `Object`. People stare at `r` and cannot see the problem, because
the problem is eleven lines up.

**Root cause:** `new ArrayList<>()` needs a **target type** to infer its type argument
from. With `List<InventoryReservation> x = new ArrayList<>()`, the target is the
declared type. With `var`, there is no target — `var` is *asking* for the type. Java
falls back to the only thing it can: `ArrayList<Object>`.

`var` and `<>` cancel each other out. One of them has to name the type.

**Fix — pick one:**
```java
var reservations = new ArrayList<InventoryReservation>();     // type on the right
List<InventoryReservation> reservations = new ArrayList<>();  // type on the left
```

The first is usually preferred with `var` in the codebase, and it also gives you the
better declared type in most cases. Note it declares `ArrayList`, not `List` — see
Trap 2 for why that occasionally matters.

**Code review rule:** `var` immediately followed by `new` with an empty diamond is
always wrong. It is a one-token grep: `grep -rn "var .* = new .*<>()" --include=*.java`.

---

### Trap 2 — `var` hiding a type the reader needs

**Wrong:**
```java
var result = orderService.process(command);
if (result.isSuccessful()) {
    notify(result.getOrderId());
}
```

**Exact symptom:** no compile error, no runtime error. The symptom is **human**, and it
shows up in three places:

1. **Code review.** The reviewer cannot tell whether `process` returns an
   `OrderResult`, an `Optional<OrderResult>`, or a `CompletableFuture<OrderResult>`.
   Those are wildly different — one of them means the work has not happened yet. So the
   reviewer either opens another file or, more commonly, approves without checking.
2. **A production incident, at 2am.** You are reading this method in a diff on a phone.
   There is no IDE, no "go to definition", no type on hover. `var` costs you the answer
   exactly when it is most expensive to get.
3. **A silent semantic change.** Somebody changes `process` from returning
   `OrderResult` to returning `Optional<OrderResult>`. Every explicit declaration in the
   codebase becomes a compile error, and the author fixes each one deliberately. Every
   `var` declaration **silently changes type**. If `Optional` happens to have a method
   with the same name you were calling, you now have a bug. If it does not, you get a
   compile error somewhere downstream with a confusing message.

Point 3 is the one that is not about taste. `var` moves a signature change from a
localised compile error to a diffuse one.

**Root cause:** `var` removes information from the source. That is exactly what it is
for. The question is whether the removed information was redundant (`new Order()`) or
load-bearing (`service.process()`).

**Fix:**
```java
OrderResult result = orderService.process(command);
```

**And an easy secondary fix that helps a lot:** name the variable after its type when
you do use `var`. `var order = repo.load(id)` reads far better than
`var r = repo.load(id)`, even though both hide the same information. If you cannot pick
a name that implies the type, that is evidence you should write the type.

---

### Trap 3 — numeric literal inference

**Wrong:**
```java
var totalMinorUnits = 0;                          // inferred: int
for (var line : order.lines()) {
    totalMinorUnits += line.quantity() * line.unitPriceMinorUnits();   // long values
}
walletService.debit(userId, totalMinorUnits);
```

**Exact symptom:** it compiles. It works in every test. Then, on a large B2B order, the
wallet is debited a **negative** amount, or an amount that is wildly too small. A
customer is credited money. There is no exception anywhere.

You find it in a reconciliation break, and the number in the report looks random.

**Root cause:** two things combine.

1. `var totalMinorUnits = 0;` infers `int`, because an integer literal with no suffix is
   an `int`. Not `long`. Not `BigDecimal`.
2. `totalMinorUnits += <long expression>` is a **compound assignment**, and the Java
   Language Specification says a compound assignment includes an **implicit narrowing
   cast** back to the left-hand side's type. So `int += long` compiles silently and
   truncates. The plain form `totalMinorUnits = totalMinorUnits + longValue` would be a
   compile error — `possible lossy conversion from long to int`. The `+=` form is not.

So `var` chose the wrong width, and `+=` hid the consequence. Either alone is
survivable; together they are silent data corruption in a money path.

**Fix:**
```java
long totalMinorUnits = 0;             // write the type. This is a money variable.
```

or, if you insist on `var`, make the literal say what it is:
```java
var totalMinorUnits = 0L;             // the L suffix makes it a long
```

**The rule:** for any numeric local whose width matters — money, counts that can exceed
2.1 billion, timestamps in millis or nanos, byte counts, ids — **write the type
explicitly**. `var` is for when the type is obvious, and the width of a numeric literal
is precisely the case where it is not.

This connects straight back to Topic 01: `var price = 19.99;` infers `double`, which is
the wrong type for money regardless of `var`. `var` did not create that bug, but it made
it invisible in the source.

---

### Trap 4 — text block indentation changing under a reformat

**Wrong (or rather, fragile):**
```java
public class ReportBuilder {
    String header() {
        return """
            ORDER REPORT
            ============
""";                                    // <-- closing delimiter at column 0
    }
}
```

**Exact symptom, part 1 — right now:** the output has 12 leading spaces on every line,
because the minimum indentation across the considered lines is 0 (the closing
delimiter's line). Probably not what the author wanted, and it is easy to miss because
the source *looks* aligned.

**Exact symptom, part 2 — the one that gets you:** somebody wraps the method body in an
`if`, or the IDE reformats the file, or a merge re-indents the block. The content lines
gain 4 spaces. The closing delimiter, being at column 0, does not move. **The output
now has 16 leading spaces instead of 12.**

If that string is:
- a **JSON payload** compared as a raw string in a test → the test fails with a diff
  where the two sides look identical in the terminal.
- an **HTTP request body** → a strict downstream parser rejects it, and the error is
  "invalid character at position 0".
- **YAML** → the document's meaning changes entirely, because YAML is
  indentation-sensitive. This one can be a production outage.
- **SQL** → harmless. Whitespace does not matter to a SQL parser, which is why nobody
  notices the problem exists until they use a text block for something else.

**Root cause:** the incidental-indentation algorithm includes the **closing delimiter's
own line** in the minimum-indentation calculation. Leaving it at a different indentation
from the content makes the string's value depend on the *relative* position of two
things that code formatters treat independently.

**Fix — always align the closing delimiter with the content:**
```java
public class ReportBuilder {
    String header() {
        return """
                ORDER REPORT
                ============
                """;                    // aligned with the content
    }
}
```

Now the minimum indentation is the content's own indentation, it is stripped entirely,
and the output is `"ORDER REPORT\n============\n"` no matter how the method is
re-indented. Wrap it in three more `if`s and the string is unchanged.

**The rules, short version:**
- Closing `"""` goes on its own line, at the same indentation as the content.
- If you deliberately want leading whitespace in the output, use `\s` or an explicit
  space escape, not source indentation — so the intent is visible.
- Use `.stripIndent()` on a `String` only when you did not author the literal;
  text blocks already do it.

**Detection:** add an assertion or a startup check for strings where it matters:
```java
assert !PAYLOAD.lines().findFirst().orElse("").startsWith(" ")
        : "text block has unexpected leading whitespace";
```
Better, for YAML and JSON: parse it in a test. A parse is a stronger check than any
whitespace assertion.

---

### Trap 5 — the trailing newline, and comparing text blocks as strings

**Wrong:**
```java
@Test
void returnsTheExpectedJson() throws Exception {
    var expected = """
            {"orderId":"o-1001","status":"PENDING"}
            """;

    var actual = mockMvc.perform(get("/orders/o-1001"))
            .andReturn().getResponse().getContentAsString();

    assertThat(actual).isEqualTo(expected);
}
```

**Exact symptom:**
```
expected: "{"orderId":"o-1001","status":"PENDING"}
"
 but was: "{"orderId":"o-1001","status":"PENDING"}"
```
The two strings look **identical** in the failure output. Developers stare at this,
conclude the assertion library is broken, and add `.trim()` — which fixes this instance
and leaves the underlying fragility in place.

**Root cause — two separate causes stacked:**

1. **The trailing newline.** A text block whose closing delimiter is on its own line
   ends with `\n`. The HTTP response does not. The strings differ by one invisible
   character.
2. **The deeper problem: JSON is not a string.** Even with the newline fixed, this
   assertion is sensitive to key order, insignificant whitespace between tokens, and
   number formatting (`1999` vs `1999.0`). None of those are part of your API contract.
   The test will fail on a change that breaks nothing and pass on a change that breaks
   something (a field with the right name and the wrong meaning).

**Fix for the newline** — put the closing delimiter on the last content line:
```java
var expected = """
        {"orderId":"o-1001","status":"PENDING"}""";
```
or end the last content line with `\`:
```java
var expected = """
        {"orderId":"o-1001","status":"PENDING"}\
        """;
```

**Fix for the real problem** — compare JSON as JSON:
```java
mockMvc.perform(get("/orders/o-1001"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.orderId").value("o-1001"))
       .andExpect(jsonPath("$.status").value("PENDING"));
```
or, when you genuinely want a whole-document comparison, use a JSON-aware comparator
(`JSONAssert`, or your assertion library's JSON support) so key order and whitespace are
ignored and the failure diff is semantic.

**The general rule:** a text block is an excellent way to *write* JSON, XML or YAML into
your source. It is a bad way to *compare* them, because you are comparing a serialised
form when you care about a structure. Use the format's own comparison. This applies
equally to XML (`XmlUnit`) and YAML (parse then compare).

---

## Hands-on proof

Every command below is one **you** run. I do not have a JVM and will not print output
and claim it is real. What follows is exactly what to look for and how to read each
possible result.

### Setup

```bash
mkdir -p ~/java-lab/30 && cd ~/java-lab/30
java --version
javac --version
```

### Proof 1 — make the compiler tell you what `var` inferred

There is no `typeof`. But there is a reliable trick: **assign the variable to something
impossible and read the error message.**

`WhatType.java`:
```java
import java.util.*;

public class WhatType {
    public static void main(String[] args) {
        var a = new ArrayList<>();
        var b = new ArrayList<String>();
        var c = 0;
        var d = 19.99;
        var e = List.of("x", "y");

        int probeA = a;      // deliberate errors
        int probeB = b;
        int probeC = c;      // this one will NOT error - c is already an int
        int probeD = d;
        int probeE = e;
    }
}
```

```bash
javac WhatType.java
```

**What to look for:** each error names the inferred type on the right-hand side.

| What you see | What it means |
|---|---|
| `incompatible types: ArrayList<Object> cannot be converted to int` for `a` | **Trap 1, proved.** `var` + `<>` gives you `Object` elements. |
| `incompatible types: ArrayList<String> cannot be converted to int` for `b` | The fix works — the type argument on the right was used. |
| No error at all for `c` | `var c = 0` inferred `int`. Trap 3's starting point. |
| `incompatible types: double cannot be converted to int` for `d` | `var d = 19.99` inferred `double`, not `BigDecimal`. Topic 01's money trap, now invisible in the source. |
| `incompatible types: List<String> cannot be converted to int` for `e` | Note `List`, not `ImmutableCollections.ListN` — `List.of` is declared to return `List<E>`, so that is what is inferred. |

**Then verify at runtime**, which tells you a *different* thing:
```java
System.out.println(b.getClass().getName());     // java.util.ArrayList
System.out.println(e.getClass().getName());     // an internal ImmutableCollections class
```

| What you see | What it means |
|---|---|
| `getClass()` shows a class with no type argument | Erasure (Topic 06). The runtime class never knows `<String>`. `var` is a **compile-time** feature; the inferred type exists only in the compiler and the debug info. |
| `List.of(...)` reports an internal class name like `java.util.ImmutableCollections$ListN` | The *static* type is `List<String>`; the *runtime* class is an implementation detail you must not depend on. |

### Proof 2 — `var` is recorded in the class file, not erased away

```bash
javac -g WhatType.java                     # -g keeps local variable debug info
javap -l -p -c WhatType.class | head -60
```

**What to look for:** a `LocalVariableTable` (and possibly a
`LocalVariableTypeTable`) listing each local with its **fully resolved type
descriptor** — `Ljava/util/ArrayList;` and so on.

| What you see | What it means |
|---|---|
| Local variable entries with concrete type descriptors | `var` is purely source-level sugar. The class file records exactly the type the compiler inferred. There is nothing dynamic and no runtime cost. |
| A `LocalVariableTypeTable` with a generic signature like `Ljava/util/ArrayList<Ljava/lang/String;>;` | Generic signatures are kept as *metadata* even though the type is erased for execution (Topic 06). |
| No tables at all | You compiled without `-g`. Add it. |

**How to read the significance:** anyone who tells you `var` "makes Java dynamically
typed" is wrong, and this listing is how you show it in ten seconds.

### Proof 3 — see incidental whitespace with your own eyes

`Indent.java`:
```java
public class Indent {

    static String aligned() {
        return """
                alpha
                  beta
                """;
    }

    static String delimiterAtZero() {
        return """
                alpha
                  beta
""";
    }

    static String noTrailingNewline() {
        return """
                alpha
                  beta""";
    }

    static void show(String name, String s) {
        System.out.println("--- " + name + " (length " + s.length()
                + ", ends with newline: " + s.endsWith("\n") + ")");
        s.lines().forEach(line -> System.out.println("[" + line + "]"));
    }

    public static void main(String[] args) {
        show("aligned", aligned());
        show("delimiterAtZero", delimiterAtZero());
        show("noTrailingNewline", noTrailingNewline());
    }
}
```

```bash
java Indent.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `aligned`: `[alpha]` and `[  beta]`, ends with newline `true` | Incidental indentation fully stripped; `beta`'s extra 2 spaces are essential and survive. This is the shape you want. |
| `delimiterAtZero`: `[                alpha]` — every line carrying source indentation | **Trap 4, proved.** The closing delimiter at column 0 made the minimum indentation 0, so nothing was stripped. Note the two methods have *identical content lines*. |
| `noTrailingNewline`: ends with newline `false` | **Trap 5's first half, proved.** Closing on the last content line removes the trailing `\n`. |
| Different lengths for `aligned` and `delimiterAtZero` | The clearest single number showing they are different strings. |

**Now do the destructive experiment.** Wrap the body of `delimiterAtZero` in an `if
(true) { ... }` so the IDE re-indents the content by 4 more spaces — but leave the
closing `"""` at column 0. Re-run and compare the length. **The number changes.** That
is a code formatter altering your program's output, which is the thing to internalise.

Repeat with `aligned` (re-indent everything including the delimiter). **The number does
not change.**

### Proof 4 — confirm string templates are gone

`Templates.java`:
```java
public class Templates {
    public static void main(String[] args) {
        String name = "world";
        String greeting = STR."Hello \{name}";     // string templates
        System.out.println(greeting);
    }
}
```

```bash
javac Templates.java
javac --enable-preview --release 25 Templates.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `error: cannot find symbol  symbol: variable STR` (both with and without the flag) | Confirmed: string templates are **not present**, not even as a preview. They were previewed in 21 and 22 and **withdrawn** in 23. Use `formatted()`. |
| It compiles with `--enable-preview` | Your JDK still has them — check `java --version`. That would mean you are on 21 or 22, and you should still not use them, because code written against them will not compile on 23+. |
| `error: ... preview feature` | Same conclusion: not for production. |

**Why do this at all?** Because you will find blog posts and Stack Overflow answers from
2023–2024 showing `STR."..."`, and you need to know in ten seconds that the feature no
longer exists rather than debugging your JDK setup.

### Proof 5 — `[JAVA 25]` module imports: preview, final, or absent?

`ModuleImport.java`:
```java
import module java.base;

public class ModuleImport {
    public static void main(String[] args) {
        List<String> skus = new ArrayList<>();
        skus.add("SKU-1001");
        Map<String, Integer> counts = new HashMap<>();
        counts.put("total", skus.size());
        System.out.println(counts + " " + Path.of("out.txt"));
    }
}
```

**Step 1 — no flags:**
```bash
javac ModuleImport.java
```

| What you see | What it means |
|---|---|
| It compiles cleanly | Module import declarations are **final** on your JDK. My information said JEP 511 finalised this in 25; your compiler is the authority. |
| `error: ... is a preview feature and is disabled by default` | It is **preview** on your JDK. Do not use it in production; features can still change. Use explicit imports. |
| `error: <identifier> expected` or a syntax error at `import module` | Your JDK predates the feature entirely. Use explicit imports. |

**Step 2 — if it said preview:**
```bash
javac --release 25 --enable-preview ModuleImport.java
java  --enable-preview ModuleImport
```
Expect a preview warning and then the program running.

**Step 3 — the fallback you would actually ship on a 21 baseline:**
```java
import java.util.*;
import java.nio.file.*;
```
```bash
javac --release 21 ModuleImportFallback.java && java ModuleImportFallback
```

**Bonus, and worth doing:** add `import module java.desktop;` alongside `java.base` and
try to use `List`. Look for `error: reference to List is ambiguous` naming both
`java.util.List` and `java.awt.List`. **What it means:** module imports pull in a very
large namespace, and ambiguity is a real cost, not a theoretical one. That is the
concrete argument for explicit imports in production code.

Do the same check for compact source files (`[JAVA 25]`, JEP 512):
```java
void main() {
    IO.println("hello");
}
```
```bash
javac CompactMain.java     # or: java CompactMain.java
```
Read the error (or the absence of one) the same way as above.

---

## Practice exercises

### 1 — Easy: apply the `var` rule

For each declaration, decide `var`, `final var`, or an explicit type. Write one sentence
of justification for each — and your justification must cite the rule, not your taste.

```java
 (a) ??? customer = new Customer(id, name, email);
 (b) ??? total = 0;                                  // accumulates order line totals in pence
 (c) ??? orders = orderRepository.findByUserId(userId);
 (d) ??? sku = "SKU-1001";
 (e) ??? results = new HashMap<>();                  // maps SKU to available quantity
 (f) ??? now = Instant.now();
 (g) ??? config = ConfigBuilder.forEnvironment("prod").build();
 (h) for (??? line : order.lines())
 (i) try (??? reader = Files.newBufferedReader(path))
 (j) ??? status = httpResponse.statusCode();
 (k) ??? rate = 0.2;                                 // VAT rate applied to money
 (l) ??? handler = new PaymentHandler() { ... };     // an anonymous class
```

Two of these have a defect that is not about readability at all — find them. (One is
Trap 1, one is Trap 3. (k) has a third problem from Topic 01.)

### 2 — Medium: fix the text blocks (combines Topics 01–29)

This test class has **six** defects: three from this topic and three from earlier
topics (01, 26, 27). Find them, state the observable symptom, and rewrite the class.

```java
class OrderApiTest {

    static final String EXPECTED = """
            {"orderId":"o-1001","totalMinorUnits":1999,"status":"PENDING"}
""";

    static final String QUERY = """
            select * from orders where user_id = '%s' and status = '%s'
            """;

    record Fixture(String name, List<String> skus, Optional<String> coupon, Double total) { }

    @Test
    void placesAnOrder() throws Exception {
        var skus = new ArrayList<>();
        skus.add("SKU-1001");

        var fixture = new Fixture("basic", skus, Optional.empty(), 19.99);

        var sql = QUERY.formatted(userId, request.getParameter("status"));
        var rows = jdbcTemplate.queryForList(sql);

        var actual = mockMvc.perform(post("/orders").content(body))
                .andReturn().getResponse().getContentAsString();

        assertThat(actual).isEqualTo(EXPECTED);
    }
}
```

For each defect write one sentence starting "The engineer sees...". For the SQL one,
also say what an attacker sends.

### 3 — Hard: production simulation on `orderflow`

Build a small report generator and prove your text blocks are stable under formatting.

**Part A — the generator.** Write `OrderReportGenerator` producing three outputs from the
same `List<OrderSummary>`:
1. A plain-text report with aligned columns (use `formatted()` with width specifiers
   like `%-20s` and `%8d`).
2. A JSON document.
3. A YAML document.

Every multi-line string must be a text block. No `+` concatenation of literals anywhere.

**Part B — the formatting attack.** This is the real exercise.
1. Write a test that asserts the exact output of each of the three generators. For JSON
   and YAML, **parse** the output and assert on the structure, not the string.
2. Now deliberately break them: for each text block in turn, move the closing `"""` to
   column 0, and separately wrap the method body in `if (true) { ... }` and re-indent.
3. Record, in a table, which of the three outputs each change breaks and which it does
   not. Explain why the plain-text and SQL cases behave differently from the YAML case.
4. Fix all three so that **no formatting change can alter the output**, and re-run all
   your attacks to confirm.

**Part C — the `var` audit.** Go through your generator and classify every local as
"var is fine", "var is wrong", or "var is dangerous". For each "dangerous" one, say what
would have to change elsewhere in the codebase to turn it into a bug. Then answer: how
many did you find, and does that change your default?

**Part D — the interpolation question.** Rewrite the JSON generator three ways:
`formatted()`, `StringBuilder`, and a real JSON library (Jackson's `ObjectMapper`
writing a record — Topic 27). Compare them on: safety against injection and escaping,
readability, and what happens when a field value contains a `"` or a newline. State
which you would ship for `orderflow` and why the answer is not the same as for the SQL
case.

**Part E — `[JAVA 25]`.** Take your smallest generator class and rewrite it as a
Java 25 compact source file using `import module java.base;`. Run the settle-it commands
from Proof 5 first. Then write one paragraph: would you use either feature in
`orderflow`? Name the specific readability cost of module imports in a codebase with
200 classes, and say what would change your answer.

---

## Interview questions

### Q1 — "When should you use `var`?"

**Mid-level answer:** "When the type is obvious from the right-hand side. It makes code
less verbose."

**Senior answer:** "My rule is: keep `var` when the right-hand side already names the
type, drop it when the type **is** the information. `var order = new Order(...)` is
pure noise removal — `new Order` says `Order`. `var result = service.process(cmd)` hides
whether that is an `OrderResult`, an `Optional<OrderResult>` or a
`CompletableFuture<OrderResult>`, and those are three completely different programs.
The cost is not felt at write time; it is felt reading a diff on a phone at 2am with no
IDE. There is also a maintenance argument that is not about taste: if somebody changes
a method's return type, every explicit declaration becomes a compile error that the
author fixes deliberately, while every `var` silently re-infers — so `var` converts a
localised, obvious break into a diffuse one. And there are two `var` cases that are
outright bugs rather than style: `var list = new ArrayList<>()` infers
`ArrayList<Object>` because `var` and the diamond cancel each other out, and
`var total = 0` infers `int`, which combined with `+=`'s implicit narrowing cast will
silently truncate a `long` in a money accumulation. So for any numeric local whose width
matters I write the type."

**What separates them:** a stated rule rather than "when it's obvious", the diff-reading
argument, the return-type-change maintenance point, and knowing the two `var` cases that
are genuine bugs.

**Interviewer's follow-up:** "Does `var` affect performance or make Java dynamically
typed?" No to both — it is compile-time only, and the class file records the fully
resolved type. `javac -g` plus `javap -l` shows it in the local variable table.

---

### Q2 — "Explain a text block's incidental whitespace."

**Mid-level answer:** "It strips the leading indentation so you can indent the text
block with your code."

**Senior answer:** "Precisely: the compiler takes every non-blank content line **plus
the line containing the closing delimiter**, finds the minimum leading-whitespace count
across them, strips that many characters from every line, then strips trailing
whitespace from every line. The detail people miss is that the **closing delimiter
participates**, which means the string's value depends on the relative position of the
content and the delimiter. Leave the closing `\"\"\"` at column 0 and nothing is
stripped; then wrap the method in one more `if`, or let the IDE reformat, and the
content lines shift while the delimiter does not — so your program's output changes from
a formatting change. For SQL that is harmless because SQL ignores whitespace, which is
exactly why nobody discovers the problem until they put YAML in a text block and an
indentation shift changes the document's meaning. My rule is: closing delimiter always
on its own line, aligned with the content. And if I genuinely want leading whitespace in
the output I use `\\s`, so the intent is visible rather than being a property of the
source layout."

**What separates them:** the four-step algorithm including the delimiter's
participation, the formatter-changes-your-output failure mode, and the observation that
SQL masks the problem while YAML exposes it.

**Interviewer's follow-up:** "How do you suppress the trailing newline?" Put the closing
delimiter on the last content line, or end that line with `\`. Then usually: "why does
that matter?" Because a test comparing a text block to an HTTP response body fails on an
invisible character, and the failure output looks identical on both sides.

---

### Q3 — "Coming from TypeScript, how do you do string interpolation in Java?"

**Mid-level answer:** "You use `String.format` or concatenation. I think Java added
string templates recently."

**Senior answer:** "There is no interpolation, and I want to be exact about the history
because it comes up. String templates — `STR.\"Hello \\{name}\"` — were previewed in
Java 21 and again in 22, and then **withdrawn** in 23. Not deferred: removed. So on both
21 and 25 the options are `formatted()` on a text block, `String.format`,
concatenation, or `MessageFormat` for i18n. I default to `formatted()` because the
template reads first and the arguments follow, which is the same shape as a template
literal. The real cost of not having interpolation is that format-specifier mistakes —
wrong count, wrong type — are **runtime** `IllegalFormatException`s rather than compile
errors, so I lean on static analysis for format strings and I keep the argument count
small. And for anything structured — JSON, XML — I do not build the string at all; I
serialize a record with Jackson, because escaping is the actual requirement and
`formatted()` will happily produce invalid JSON when a value contains a quote."

**What separates them:** knowing string templates were withdrawn rather than shipped —
which is a real trap given how much 2023–2024 content shows them — and the
"format errors are runtime errors" consequence with a mitigation.

**Interviewer's follow-up:** "So what would you use to build a SQL query?" A text block
for the static query text and **bound parameters** for the values, always. Text blocks
made concatenating SQL more pleasant, and that measurably increased how often people
write injectable queries.

---

### Q4 — "What does `var list = new ArrayList<>()` infer, and why?"

**Mid-level answer:** "`ArrayList`. I'd have to check what the element type is."

**Senior answer:** "`ArrayList<Object>`, and the reason is that `var` and the diamond
cancel each other out. The diamond infers its type argument from a **target type** — in
`List<String> x = new ArrayList<>()` the target is the declared type on the left. With
`var` there is no target, because `var` is asking the right-hand side what the type is,
so inference falls back to `Object`. The nasty part is where the error appears: the
declaration and the `add` calls compile fine, and you get `cannot find symbol` naming
`Object` at the first place you use an element — often ten lines away, in a method where
you never wrote `Object`. So it reads as a mystery. The fix is that exactly one side
must name the type: `var list = new ArrayList<String>()` or
`List<String> list = new ArrayList<>()`. It is also a trivial grep, so I would add it to
a lint rule rather than relying on review."

**What separates them:** knowing the answer without checking, explaining *why* via
target typing, and describing where the error surfaces — which is what makes it hard to
diagnose in the wild.

**Interviewer's follow-up:** "Which of the two fixes do you prefer?" A reasonable answer
notes that `var list = new ArrayList<String>()` declares the local as `ArrayList`, not
`List`, which matters if you later want to reassign it to a different implementation —
so in a codebase that programs to interfaces, the explicit `List<String>` form is often
the better habit.

---

### Q5 — "Your team is arguing about `var` in code review. How do you settle it?"

**Mid-level answer:** "Agree on a team convention and put it in the style guide."

**Senior answer:** "A convention is the outcome, but I would not start there, because
'use `var` where it's obvious' is not actionable and the argument restarts every sprint.
I would separate the question into two, because they get conflated. First, the cases
that are **defects**, not preferences: `var` with an empty diamond inferring `Object`,
and `var` on a numeric local whose width matters, where `+=`'s implicit narrowing hides
a truncation. Those go into a linter — ErrorProne or an IDE inspection wired into CI —
and they stop being a review topic entirely. Second, the genuinely stylistic case, where
I would adopt a rule that can be applied without argument: keep `var` when the
right-hand side names the type, drop it when the type is the information. That is
checkable by a reviewer in one second without opening another file, which is the actual
property you want from a style rule. I would also point out that neither position is
worth much review time relative to, say, whether the transaction boundary is right —
so I would time-box the discussion, write the rule down, and move on. Style debates are
mostly a symptom of not having automated the parts that are actually decidable."

**What separates them:** splitting defects from preferences and automating the first,
choosing a rule for its *checkability* rather than its correctness, and the judgment to
name the opportunity cost of the argument itself. That last part is what a senior
interview is actually probing.

**Interviewer's follow-up:** "What if the team overrules you?" The right answer is that
consistency beats being right about `var`, you follow the team's rule, and you spend
your credibility on things that break production instead.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `var` is banned on fields, parameters and return types. Construct the argument for
   that restriction from first principles: what is different about a local variable that
   makes inference safe there and unsafe elsewhere?

2. TypeScript's default is `const` and Java's is mutable, with `final var` as the opt-in.
   Java could not have changed the default without breaking every existing program.
   Given that, was adding `var` without a `val`-style counterpart the right call? Argue
   both sides.

3. The incidental-whitespace algorithm includes the closing delimiter's line. Design an
   alternative rule that does not, and say what it would break. Why do you think the
   designers chose this one?

4. String templates were previewed twice and then withdrawn. What does that tell you
   about how to treat *any* preview feature in production code? Now generalise it: what
   is the correct policy for preview features in a service you are on-call for?

5. `var total = 0; total += someLong;` compiles and truncates, while
   `total = total + someLong;` does not compile. Both are in the JLS deliberately. What
   was the compound-assignment narrowing rule protecting, and is that trade still worth
   it today?

6. A text block containing YAML is dangerous under reformatting; one containing SQL is
   not. Name three other formats and classify each. What property of a format determines
   which side it lands on?

7. Both features in this topic make code shorter. Name a third Java feature you have met
   in Phase 2 that makes code shorter and has a comparable "this can hide something the
   reader needs" cost, and say how the mitigation differs.

---

## Quick reference card

### `var` — allowed and forbidden

| Allowed | Forbidden |
|---|---|
| local variable with initialiser | field |
| `for (var x : xs)` | method parameter |
| `for (var i = 0; ...)` | method return type |
| try-with-resources resource | constructor parameter |
| lambda parameter `(var a, var b) ->` | `catch (var e)` |
| `final var` | `var x;` (no initialiser) |
| non-denotable types (anonymous classes, intersections) | `var x = null;` |
| | `var f = () -> {}` / `var g = X::y` |
| | `var a = {1, 2, 3}` |

### The `var` decision rule

```
Does the right-hand side already name the type?
    YES -> var is fine        var order = new Order(...)
    NO  -> write the type     OrderResult r = service.process(cmd)

Is it a numeric local whose width matters (money, counts, timestamps)?
    -> ALWAYS write the type. var infers int / double.

Is it `new Something<>()` with an empty diamond?
    -> Bug. Put the type argument on the right, or the type on the left.
```

### Text blocks

```java
String s = """
        content                 <- content lines
          more content
        """;                    <- closing delimiter: ALIGN WITH CONTENT
```

| Rule | Detail |
|---|---|
| Opening `"""` | must be followed by a line terminator |
| Incidental indent | min leading whitespace of non-blank content lines **and** the closing delimiter's line |
| Trailing whitespace | stripped from every line, always |
| Trailing newline | present unless the closing delimiter is on the last content line, or that line ends with `\` |
| `\` at end of line | suppress that newline (line continuation) |
| `\s` | a space that survives trailing-whitespace stripping |
| `\n \t \" \\` | work normally |
| `\"""` | how to write three quotes inside |
| `.formatted(args)` | the interpolation substitute (Java 15+) |
| `.stripIndent()` | for strings you did **not** author as a text block |

### Interpolation options (there is no `${}`)

```java
"""
Order %s total %d
""".formatted(id, total)                       // preferred with a text block
String.format("Order %s total %d", id, total)  // same thing, older shape
"Order " + id + " total " + total              // fine for 2-3 pieces
MessageFormat.format("Order {0}", id)          // i18n / positional
objectMapper.writeValueAsString(record)        // for JSON. Always. Never build it by hand.
```

### `[JAVA 25]` features in this topic

| Feature | Status (verify on your JDK) | 21 fallback |
|---|---|---|
| Text blocks | final since **15** | — |
| `var` | final since **10** | — |
| `formatted()` | final since **15** | `String.format` |
| String templates `STR."..."` | **WITHDRAWN in 23** — do not use | `formatted()` |
| Module imports `import module java.base;` (JEP 511) | believed final in **25** — Proof 5 settles it | `import java.util.*;` etc. |
| Compact source files / instance `main` (JEP 512) | believed final in **25** — Proof 5 settles it | `public static void main(String[])` |

### Gotchas checklist

- [ ] `var x = new ArrayList<>()` infers `ArrayList<Object>`. Always a bug.
- [ ] `var total = 0` is an `int`. For money/counts/timestamps, write `long`.
- [ ] `int += long` compiles and truncates. `int = int + long` does not compile.
- [ ] `var price = 19.99` is a `double`. Money is `long` minor units or `BigDecimal`
      (Topic 01).
- [ ] Closing `"""` on its own line, aligned with the content. Always.
- [ ] A text block ends with `\n` unless you suppress it.
- [ ] Never compare JSON/XML/YAML as raw strings. Parse and compare structurally.
- [ ] Never interpolate untrusted values into SQL. Bind parameters.
- [ ] There is no string interpolation. String templates were withdrawn.
- [ ] `final var` is your `const`. It exists; almost nobody writes it.

---

## When would I use this at work?

**1. Every SQL query, JSON fixture and outbound payload you write.**
Text blocks are the single largest readability improvement in modern Java for anyone
writing a service that talks to a database and an HTTP API. A native query you can read
gets reviewed properly; one built from `" + "` does not. The habit to build now is:
text block for the static text, bound parameters for values, and never a formatted
value in a SQL string.

**2. Settling a `var` debate in a code review, in one comment, without a fight.**
"The RHS doesn't name the type here, so I'd write it out" is a rule the author can
apply themselves next time. "I don't like `var`" is not. And when you spot
`var x = new ArrayList<>()` you are not expressing a preference at all — you are
catching a bug, and saying so changes the conversation.

**3. Debugging an assertion where both strings look identical.**
A test fails, the expected and actual look character-for-character the same in the
terminal, and someone is about to add `.trim()`. You know immediately to check the
trailing newline and the closing-delimiter alignment, and then to argue that the test
should be comparing parsed JSON rather than strings at all. That is a five-minute fix
instead of an afternoon, and the second half of it prevents the next three occurrences.

---

## Connected topics

**Prerequisites:**
- **01 — Primitives and autoboxing**: why `var total = 0` inferring `int` matters, why
  `var price = 19.99` inferring `double` is wrong for money, and the compound-assignment
  narrowing rule.
- **06 — Type erasure**: why `var` is purely compile-time, and why `getClass()` on a
  `var`-declared generic tells you less than the compiler knows.
- **18 — Strings, the pool, StringBuilder**: text blocks are ordinary `String`
  constants, interned like any other literal; `formatted()` is `String.format`; and the
  concatenation-in-a-loop rule is unchanged.
- **17 — Immutability**: `final var` and effectively-final locals.
- **21 — Lambdas**: why a bare lambda cannot initialise a `var`, and why captured locals
  must be effectively final.
- **27 — Records**: serializing a record is the correct alternative to building JSON in
  a text block.
- **29 — Pattern matching**: `var` inside record patterns —
  `case Approved(var ref, var amount)` — where it is idiomatic and safe because the
  pattern already names the type.

**This unlocks:**
- **46 — Error handling and `ProblemDetail`**: error response bodies as records
  serialized by Jackson, not as text blocks.
- **47 — Spring Data JPA**: `@Query` with a text block is the standard way to write a
  non-trivial JPQL or native query. This is where you will use text blocks daily, and
  where the bound-parameter discipline is not optional.
- **58–64 — Testing**: JSON fixtures as text blocks, and why structural comparison beats
  string comparison for every structured format.
- **117 — Sagas**: saga step definitions and compensation queries are long SQL and JSON
  documents; readable ones are reviewable ones, and in code that moves money that
  matters more than usual.
- **127 — Migration planning**: `var` adoption, text-block conversion of legacy
  concatenated SQL, and the withdrawn-string-templates story as a case study in how to
  treat preview features during a JDK upgrade.

---

*Java baseline 21. `var` is final since **Java 10**; text blocks and `formatted()` since
**Java 15** — neither needs any flag on 21 or 25. **String templates (`STR."..."`) were
previewed in 21 and 22 and WITHDRAWN in 23** — they do not exist on your JDK and are not
coming back in that form; use `formatted()`. My information is that **JEP 511 (module
import declarations)** and **JEP 512 (compact source files and instance `main` methods)**
were finalised in **JDK 25**; I have flagged that as belief rather than assertion, and
Proof 5 gives you the exact commands to settle both on your own JDK. Where this document
and your compiler disagree, your compiler is right.*
