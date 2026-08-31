# 67 — Class Loading: Delegation, Laziness, and Initialization Order

## Phase: 8 — JVM Internals
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you diagnose an `orderflow` startup failure, a mid-request `NoClassDefFoundError`, and a class-initialization deadlock that shows up as a hung thread pool — from evidence, in one step, instead of by restarting and hoping. Every command here runs against the containerised service you built for the Topic 65 gate.

---

## Mechanical statement

Read this three times. The rest of the document is an elaboration of it.

> **Loading is parent-first delegation.** When a class loader is asked for a class, it
> first asks its parent, and its parent asks *its* parent, all the way up to the
> bootstrap loader. Only if every ancestor fails does the loader try to find the class
> itself. The loader that ultimately *defines* the class becomes part of that class's
> runtime identity: a class is identified by the pair **(binary name, defining loader)**,
> not by its name alone.
>
> **A class is LOADED eagerly-ish and INITIALIZED lazily.** Loading, verifying and
> preparing can happen at any point before first use. **Initialization** — running the
> `static` field initializers and `static { }` blocks, collectively the `<clinit>`
> method — happens exactly once, at the first *active use*, under a per-class lock.
>
> **An initialization failure poisons the class for the entire lifetime of the JVM.**
> The first thread to touch it gets an `ExceptionInInitializerError` wrapping the real
> cause. The class is then marked **erroneous**. Every subsequent access — from any
> thread, forever — gets a `NoClassDefFoundError: Could not initialize class ...`, which
> contains none of the information you need. There is no retry. There is no recovery
> short of a restart.

Four consequences follow directly, and you should be able to derive each one:

1. The **first** error in the log is the truth. Every error after it is an echo.
2. Putting configuration reads, network calls, or file I/O in a `static` block converts
   a recoverable runtime error into a permanent, process-wide, undiagnosable one.
3. Two loaders can define the same class name, and the resulting `ClassCastException`
   will appear to say that a class cannot be cast to itself.
4. Because `<clinit>` runs under a lock, two classes whose static initializers reference
   each other can **deadlock** on first touch from two threads — and the thread dump
   will show threads parked in `<clinit>` with no `synchronized` block anywhere in your
   code.

---

## The bridge from what you know

### The honest answer: mostly NO ANALOGUE

Node has a module system. Java has a class-loading system. They rhyme in one place and
diverge everywhere else, and the divergences are exactly where the bugs live.

**What genuinely transfers — one thing.**

A Node module body executes **once**, on first `require`/`import`, and the result is
cached. Java's `<clinit>` also runs once, on first active use, and the result is cached.
Both are lazy, both are memoised, and in both runtimes "top-level code in a module" and
"static initializer in a class" are the place where people put initialisation that
should have gone in a constructor or a factory.

Keep that intuition. It is correct.

### NO TYPESCRIPT ANALOGUE — the four things with no counterpart

**1. Parent-first delegation has no analogue.**

Node resolves a module by walking `node_modules` directories **upward** from the
requiring file. It is a filesystem search, and it is *child-first* in spirit: the
closest copy wins. The JVM does the opposite. A loader asks its **parent** first, and
the parent's answer wins. This inversion is why a jar you put on your classpath cannot
override a JDK class, and why the "just shade it into your fat jar" instinct from npm
does not do what you expect.

**2. A class's identity including its loader has no analogue — well, almost.**

Here is the one place your npm experience is genuinely useful, and it is a **PARTIAL**
analogue worth transferring carefully. You have met this bug:

```js
// two copies of the same library, at different nesting depths
const A = require('some-lib');            // node_modules/some-lib
const B = require('dep/node_modules/some-lib');
new A.Thing() instanceof B.Thing          // false. Same code. Different identity.
```

Java's version is the same *shape* with a different *cause*. In Java there is exactly
one version of an artifact on a flat classpath (Topic 32), so npm's nesting problem
cannot happen. But two **class loaders** can each define `com.orderflow.orders.Order`
from the same bytes, and those two classes are different types at runtime. The error is
worse than JavaScript's silent `false`:

```
java.lang.ClassCastException: class com.orderflow.orders.Order cannot be cast to
class com.orderflow.orders.Order (com.orderflow.orders.Order is in unnamed module of
loader 'app'; com.orderflow.orders.Order is in unnamed module of loader
org.example.PluginClassLoader @1b6d3586)
```

Read that message once and you will never misread it again: the parenthesised loader
names are the whole diagnosis.

**3. Initialization poisoning has no analogue — and Node behaves the OPPOSITE way.**

This is the single most important paragraph in the bridge, so read it slowly.

In **CommonJS**, if a module throws while it is being evaluated, Node **removes it from
the module cache**. The next `require` of that module re-executes it from scratch. A
transient failure — a config file that was not written yet, a DNS blip — can therefore
succeed on the second attempt. The system is *self-healing by default*.

In **ESM**, a module that throws during evaluation is cached in a failed state, and a
later `import` throws **the same error again**. Closer to Java — but you still get the
real error, every time.

In **Java**, neither happens. The first thread gets the real error, once. The class is
marked erroneous permanently, and every access thereafter — for the life of the process
— gets a **different, useless error** that names no cause. The system is not
self-healing, and it actively destroys the evidence.

> **Write this down:** in Node, a failed module retries or repeats its error. In Java, a
> failed class initialization is permanent and the error changes into a worse one. Your
> instinct that "it'll probably work on the retry" is not just wrong here — it is
> inverted.

**4. Explicit, pluggable loaders have no analogue.**

You cannot write a custom module resolver in Node that produces two mutually
incompatible copies of the same class with distinct identities, and then hand one to a
plugin. Java application servers, OSGi, Spring Boot's fat-jar launcher, IDE plugin
systems, and hot-reload tooling all do exactly that, deliberately. You will meet at
least one of them, and when you do, the delegation order is the first thing you check.

| You know | Java | Verdict |
|---|---|---|
| Module body runs once, cached | `<clinit>` runs once, cached | **HONEST ANALOGUE** |
| `node_modules` upward search, closest wins | Parent-first delegation, ancestor wins | **NO ANALOGUE** — the direction is inverted |
| Two nested copies of a package break `instanceof` | Two loaders defining one name break `instanceof`/casts | **PARTIAL** — same symptom, different cause, worse message |
| CJS: failed module is evicted and retried | Erroneous class is permanent | **NO ANALOGUE** — and the Java behaviour is the opposite of your instinct |
| ESM: failed module repeats the same error | Later accesses get a *different*, useless error | **NO ANALOGUE** |
| `import` is statically analysable; bundlers see the whole graph | `Class.forName(String)` is a runtime string lookup | **NO ANALOGUE** — the graph is not knowable ahead of time, which is Topic 83's whole problem |
| No such thing as a module identity crisis you can cause on purpose | Custom loaders, containers, OSGi | **NO ANALOGUE** |

---

## What is this?

Class loading is the process by which a `.class` file — a byte array, from wherever —
becomes a usable runtime type inside the JVM.

The JVM specification breaks it into three activities, and the middle one has three
sub-steps. You will use these names when you read log tags and error messages, so learn
them as vocabulary, not as trivia.

| Step | What happens | What can go wrong, and what you see |
|---|---|---|
| **Loading** | A class loader is asked for a binary name. It obtains the bytes and calls `defineClass`, which creates the runtime `Klass` metadata in metaspace and the `java.lang.Class` mirror on the heap. | Bytes not found: `ClassNotFoundException` (from a reflective/explicit lookup) or `NoClassDefFoundError` (from a linkage attempt the compiler baked in). Malformed bytes: `ClassFormatError`. Name inside the file does not match the name asked for: `NoClassDefFoundError` with a "wrong name" message. |
| **Linking → Verification** | The bytecode verifier proves every method is type-safe: the operand stack never underflows, locals hold what they claim, no jump lands mid-instruction. | `VerifyError`. Almost always means generated or rewritten bytecode (an agent — Topic 81 — or a bytecode library on the wrong class-file version). |
| **Linking → Preparation** | Static fields are allocated and set to **default values**: `0`, `0L`, `false`, `null`. Constant-valued `static final` fields get their real value here, before any code runs. | Nothing user-visible. But this is why a static field can be observed as `null` *during* initialization — see the circular-init trap. |
| **Linking → Resolution** | Symbolic references in the constant pool (`"com/orderflow/orders/Order"`) are turned into direct references. HotSpot does this **lazily**, at first use of each reference, not eagerly at load time. | `NoClassDefFoundError`, `NoSuchMethodError`, `NoSuchFieldError`, `IllegalAccessError` — the entire family of "it compiled but the runtime classpath is different" errors (Topic 32). |
| **Initialization** | `<clinit>` runs: static field initializers and `static { }` blocks, **in textual order**. Exactly once. Under a per-class lock. | `ExceptionInInitializerError` the first time; `NoClassDefFoundError: Could not initialize class X` every time after. |

The single most consequential design decision in that table is that **resolution is lazy
and initialization is lazier still.** A jar can be missing from your container image and
the service will start perfectly, pass its readiness probe, serve traffic for six
minutes, and then fail on the first request that reaches the code path that needs it.

### The three built-in loaders

```
bootstrap  (written in native code; reported as null from Java)
   ^ parent
platform   (jdk.* modules that are not core; formerly "extension")
   ^ parent
application  (your classpath / module path; also called "system")
   ^ parent
[your custom loaders, if any]
```

Ask any class where it came from:

```java
System.out.println(String.class.getClassLoader());          // null  -> bootstrap
System.out.println(java.sql.Driver.class.getClassLoader()); // platform
System.out.println(OrderService.class.getClassLoader());    // app (or Boot's launcher)
```

`null` means bootstrap. It does not mean "no loader" and it is not a bug.

---

## Why does it matter?

**1. Because the error you see is almost never the error that happened.**

A static initializer throws once, at 03:14:07, on one request. From 03:14:07 onward,
every request that touches that class logs `NoClassDefFoundError: Could not initialize
class com.orderflow.pricing.CurrencyRates`. At 40 order placements per second, that is
144,000 useless log lines per hour burying one useful one. The on-call engineer greps
for the most frequent error and finds the echo. The cause is a single line, hours back,
possibly rotated out of the log entirely.

Knowing this rule — *"Could not initialize class" means look earlier, not here* — turns
a multi-hour incident into a two-minute one. It is the highest return-on-investment fact
in Phase 8.

**2. Because laziness moves failures from deploy time to peak time.**

Your CI passed. Your container started. Your readiness probe went green. None of that
loaded the class that is missing, because nothing had called that code path yet. The
failure surfaces when a customer hits the endpoint — which, for an infrequently used
endpoint, might be during the Friday-evening peak rather than the Tuesday-morning
deploy. Topic 31's `provided` scope and Topic 32's nearest-wins resolution are the two
most common ways to arrange this for yourself.

**3. Because `<clinit>` holds a lock, and locks deadlock.**

Class initialization is specified to be thread-safe (JLS 12.4.2). The JVM takes a
per-class initialization lock. If class `A`'s static initializer touches `B`, and `B`'s
touches `A`, and two threads enter through different doors at the same moment, they
deadlock. The thread dump shows both threads in `<clinit>`. `jcmd Thread.print` will
**not** report it in the "Found one Java-level deadlock" section, because those are
monitor deadlocks and this one is not. You have to recognise it by shape.

Under the Topic 65 load profile, "two threads hit two classes for the first time
simultaneously" is not a rare event. It is what happens in the first 200 milliseconds
after your service starts taking traffic.

**4. Because class loading is a measurable share of your startup and your first-request
latency.**

`orderflow` on Spring Boot loads on the order of ten thousand classes before it serves
its first request. Each one is read, verified, and linked. That is real CPU time, real
metaspace, and a real contribution to the "why is the first minute after deploy slow"
question that Topic 74 answers from the JIT side. You can count it exactly, and you will,
in the Measurement section.

---

## Machine-level reality

### The delegation chain, as code

`ClassLoader.loadClass` has done essentially this since Java 1.2. Read it, because every
container, plugin system and fat-jar launcher you will ever debug either follows it or
deliberately breaks it:

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 1. Have I already defined or been handed this class?
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                // 2. Ask my parent FIRST. Recurses to the top of the chain.
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // Parent could not find it. That is normal, not an error.
            }
            if (c == null) {
                // 3. Only now do I look in my own sources (classpath entries, jars, network).
                c = findClass(name);          // this is the method you override
            }
        }
        if (resolve) { resolveClass(c); }
        return c;
    }
}
```

Three facts fall out of this that you will use:

- **`findClass` is the extension point, not `loadClass`.** Overriding `findClass`
  preserves delegation. Overriding `loadClass` is how you deliberately break it, which
  is what a "parent-last" web application loader does — and it is why a servlet
  container can let a webapp ship its own version of a library the container also uses.
- **The lock is per class name** (`getClassLoadingLock`), not global. That was a Java 7
  change to allow parallel-capable loaders. It matters because it means *loading* is
  concurrent, while *initializing* is serialised per class.
- **A `ClassNotFoundException` from the parent is normal control flow.** Delegation is
  implemented with exceptions. This is why class loading shows up disproportionately in
  exception-throughput profiles of application servers.

### Class identity is (name, defining loader)

Two definitions of the same binary name by two different loaders produce **two distinct
runtime types**. They are not assignable to each other. They do not share static fields.
`instanceof` is false between them. This is by design: it is what makes plugin isolation
and multi-tenant application servers possible at all.

The rule you need for diagnosis:

> A `ClassCastException` whose two type names are **identical** is always a class-loader
> problem. Read the parenthesised loader identity, not the class name.

The same rule applies to a slightly different message when it involves package-private
access: runtime packages are also identified by (package name, loader), so two classes
in the same-named package loaded by different loaders cannot access each other's
package-private members, and you get `IllegalAccessError`.

Where you will actually meet this:

| Situation | Why two loaders exist |
|---|---|
| Spring Boot fat jar | The launcher installs its own loader that can read classes from nested jars inside `BOOT-INF/lib`. The class name of that loader has changed across Boot versions — **print it, do not quote me**: `System.out.println(OrderService.class.getClassLoader())`. |
| Servlet container (Tomcat/Jetty) deployment | One loader per web application, deliberately parent-last for application classes so each app can ship its own library versions. |
| Devtools / hot restart | Boot Devtools uses a throwaway "restart" loader for your classes and a stable "base" loader for libraries, so restarting only re-loads your code. This is the most common way a developer meets this bug on their own laptop. |
| A plugin/extension system you write | You created the loader on purpose. |
| An APM or instrumentation agent | Topic 81. The agent may define helper classes into your loader or its own. |

### Laziness: exactly what triggers initialization

This is a closed list. Memorise it; it is short and it settles arguments.

**These trigger initialization of a class `C`:**

1. `new C(...)` — creating an instance.
2. Invoking a static method declared in `C`.
3. Reading or writing a static field declared in `C` — **unless** it is a `static final`
   field whose initialiser is a compile-time constant expression.
4. `Class.forName("C")` (the one-argument form; it initializes by default).
5. Initializing a subclass of `C` — superclasses are initialized first, top-down.
6. `C` being the class whose `main` method starts the JVM.
7. Certain reflective and `MethodHandle` operations, notably
   `MethodHandles.Lookup.ensureInitialized(C.class)`.

Also: when a class is initialized, any **superinterface that declares a default method**
is initialized too. Interfaces with only constants and abstract methods are not.

**These do NOT trigger initialization:**

| Non-trigger | Why it surprises people |
|---|---|
| `C[] array = new C[10];` | Creates an array *type*, not an instance of `C`. `C` is loaded but never initialized. |
| Reading `C.SOME_CONSTANT` where it is `static final int/String` with a constant initialiser | `javac` **inlines the value into the caller's constant pool**. `C` is not even loaded. This is Trap 5, and it is a silent-wrong-value bug across a library boundary. |
| `Class.forName("C", false, loader)` | The `false` is `initialize`. Explicitly loads without initializing. |
| `loader.loadClass("C")` | `loadClass` never initializes. Only `forName` does. |
| Accessing a static field *declared in a superclass* through a subclass name (`Sub.fieldFromSuper`) | Only the declaring class is initialized. `Sub` is not. |
| Declaring a field, parameter or return type of type `C` | Types in signatures are symbolic until used. |

Try to hold one sentence: **mentioning a type is free; using it is what costs.**

### Initialization, step by step, including the failure path

JLS 12.4.2 specifies this precisely. The parts you need:

1. The JVM acquires the **initialization lock** for the class.
2. It reads the class's initialization state:
   - **already initialized** → release the lock and return.
   - **erroneous** → release the lock and throw `NoClassDefFoundError`. **This is the
     poisoned state.**
   - **being initialized by *another* thread** → wait on the lock until that thread
     finishes. This is where the deadlock lives.
   - **being initialized by *this* thread** (recursive entry) → **return immediately and
     let execution continue.** This is how circular initialization silently yields
     default values instead of deadlocking.
3. Mark the class as being-initialized-by-this-thread, release the lock.
4. Initialize the superclass (and any default-method-declaring superinterfaces) first.
5. Run `<clinit>`: static field initialisers and `static { }` blocks in **textual
   source order**.
6. On success: mark initialized, notify waiters.
7. On a `Throwable` `E`:
   - if `E` is an `Error`, propagate `E` as-is;
   - otherwise wrap: `throw new ExceptionInInitializerError(E)`;
   - **mark the class erroneous, permanently**, and notify waiters.

Step 2's "erroneous" branch and step 7's "permanently" are the whole topic. There is no
API to reset it. There is no flag to disable it. A class that failed to initialize is
dead until the process restarts.

Step 2's recursive-entry branch deserves its own emphasis, because it is a *silent*
failure and silence is worse than a stack trace:

```java
class Catalog {
    static final int PAGE_SIZE = Defaults.PAGE_SIZE;   // reads Defaults
}
class Defaults {
    static { warm(); }                                  // warm() touches Catalog
    static final int PAGE_SIZE = 50;
    static void warm() { System.out.println(Catalog.PAGE_SIZE); }
}
```

If `Defaults` is initialized first, `warm()` triggers `Catalog`'s initialization, which
reads `Defaults.PAGE_SIZE` — but `Defaults` is already being-initialized-by-this-thread,
so the read proceeds against the **prepared, not yet assigned** value. `PAGE_SIZE` is
`0`. No exception. No warning. `Catalog.PAGE_SIZE` is `0` forever.

If `Catalog` is initialized first, you get `50`. **The correct value depends on which
class something happened to touch first**, which depends on request order, which depends
on load. This is the shape of the worst bug in this document.

### `<clinit>` versus `<init>`, and the order of everything

You will be asked to order these in an interview. Here is the whole picture for one
`new Sub()` where `Sub extends Base`:

```
1. (once, lazily)   Base.<clinit>     — Base's static fields + static blocks, textual order
2. (once, lazily)   Sub.<clinit>      — Sub's static fields + static blocks, textual order
3. (every instance) Base.<init> begins: super() chain runs to Object first
4.                  Base's instance field initialisers + instance { } blocks, textual order
5.                  Base's constructor body
6. (every instance) Sub's instance field initialisers + instance { } blocks, textual order
7.                  Sub's constructor body
```

The trap that lives at steps 5 and 6: if `Base`'s constructor calls an overridable
method that `Sub` overrides, `Sub`'s override runs at step 5 — **before** `Sub`'s own
fields are initialised at step 6. Your `final` field is `null`. This is not a class
loading bug, but it is in the same family of "initialisation order is not the order you
read the file in", and it is the most common way a `final` field is observed as `null`.

### The thread context class loader, and why `ServiceLoader` needs it

The bootstrap loader cannot see your classpath. That is the whole point of parent-first
delegation. So how does a JDK class like `java.sql.DriverManager` — loaded by the
platform loader — instantiate the PostgreSQL driver, which lives on your application
classpath, below it in the chain?

It cheats, via the **thread context class loader** (TCCL):

```java
ClassLoader tccl = Thread.currentThread().getContextClassLoader();
```

Every thread carries a reference to a loader that framework code can use to load
application classes "upward-blind". `ServiceLoader`, JDBC, JAXP, JNDI and most
plugin-style JDK APIs use it. Two consequences:

- **Threads you create inherit the TCCL of the creating thread.** A thread created at
  startup by a library may carry a different TCCL than a request-handling thread. This
  is why "it works in the main thread and throws `ClassNotFoundException` in my executor"
  is a real bug shape.
- **In a container or a fat jar, the TCCL is the interesting loader**, not
  `getClass().getClassLoader()`. When you write code that loads classes by name, prefer
  an explicitly passed loader; fall back to the TCCL; fall back to your own loader last.

### Class unloading and metaspace

A class can be unloaded only when its **defining class loader** becomes unreachable —
which means the loader, every class it defined, and every instance of those classes must
all be garbage. In practice this is all-or-nothing per loader.

The consequence: a **class-loader leak** retains the loader, therefore every class it
defined, therefore the metaspace those classes occupy, therefore the static field values
those classes hold on the heap. It shows up as *both* metaspace growth and heap growth,
which is why it confuses people (Topic 66 made this point; Topic 79 gives you the tool).

The classic producers: repeated deployments into a long-lived container, a
`ThreadLocal` on a pooled thread holding an instance of a class from a discarded loader,
a JDBC driver registering itself in a `DriverManager` static that outlives the webapp,
and dynamic proxy or bytecode-generation libraries creating a loader per generated class.

Bound it and observe it:

```bash
jcmd <pid> VM.metaspace
jcmd <pid> VM.classloaders
jcmd <pid> VM.classloader_stats
```

`-Xlog:class+unload=info` prints a line per unloaded class. Zero lines over a long run
with heavy dynamic class generation is a finding, not an absence of one.

### Where the JDK itself does this to you

`Integer.valueOf` (Topic 01) is implemented against a private nested class:

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer[] cache;
    static { /* reads the java.lang.Integer.IntegerCache.high property, builds the array */ }
}
```

`IntegerCache` is initialized lazily, on the first boxing operation in the JVM's life,
and it reads a system property inside its static block. That is the *exact* pattern this
document warns you about — the JDK gets away with it because the property parse cannot
realistically fail. Yours can.

---

## Example 1 — minimal

The smallest program that separates loading from initialization, and shows the poisoning.

```java
package com.orderflow.lab.classload;

/**
 * Three classes, one file, no framework.
 * Demonstrates: (a) loading is not initialization, (b) initialization is lazy,
 * (c) an initialization failure is permanent and the second error is a lie.
 */
public final class InitOrderProbe {

    public static void main(String[] args) throws Exception {

        System.out.println("--- 1. Declaring an array type does NOT initialize ---");
        Rates[] unused = new Rates[3];
        System.out.println("array created, length " + unused.length);

        System.out.println("--- 2. Reading a compile-time constant does NOT initialize ---");
        System.out.println("currency = " + Rates.DEFAULT_CURRENCY);

        System.out.println("--- 3. loadClass does NOT initialize ---");
        Class<?> c = InitOrderProbe.class.getClassLoader().loadClass(
                "com.orderflow.lab.classload.Rates");
        System.out.println("loaded: " + c.getName());

        System.out.println("--- 4. Class.forName DOES initialize ---");
        try {
            Class.forName("com.orderflow.lab.classload.Rates");
            System.out.println("initialized without error (unexpected for this probe)");
        } catch (ExceptionInInitializerError e) {
            System.out.println("FIRST TOUCH  -> " + e.getClass().getName());
            System.out.println("  real cause -> " + e.getCause());
        }

        System.out.println("--- 5. Touching it again gives a DIFFERENT, worse error ---");
        try {
            new Rates();
            System.out.println("constructed (unexpected)");
        } catch (Throwable t) {
            System.out.println("SECOND TOUCH -> " + t.getClass().getName());
            System.out.println("  message    -> " + t.getMessage());
            System.out.println("  cause      -> " + t.getCause());
        }
    }
}

final class Rates {
    /** Compile-time constant: inlined by javac into every caller. */
    static final String DEFAULT_CURRENCY = "GBP";

    /** NOT a compile-time constant: computed at runtime, so reading it initializes. */
    static final double GBP_TO_EUR;

    static {
        System.out.println("    [Rates.<clinit> is running]");
        String raw = System.getenv("ORDERFLOW_GBP_EUR");    // absent in this probe
        GBP_TO_EUR = Double.parseDouble(raw);                // throws NPE
    }
}
```

```bash
mkdir -p ~/java-lab/67 && cd ~/java-lab/67
javac -d out InitOrderProbe.java
java -cp out com.orderflow.lab.classload.InitOrderProbe
```

**WHAT TO LOOK FOR.** Not the values — the *sequence*. Four specific questions:

1. Does `[Rates.<clinit> is running]` appear before or after the "loaded:" line?
2. Does step 2 (the constant) print `GBP` without `<clinit>` running at all?
3. Is the exception type at step 4 different from the exception type at step 5?
4. At step 5, is `getMessage()` informative, and is `getCause()` null?

| What you see | What it means |
|---|---|
| `<clinit>` runs at step 4 only, not at steps 1–3 | The expected result. Loading, array-type creation and constant reads do not initialize. You have observed laziness directly. |
| Step 2 prints `GBP` and step 3 succeeds, with no `<clinit>` line | Confirms `javac` inlined the constant, and that `loadClass` does not initialize. Two separate facts, one output line each. |
| Step 4: `ExceptionInInitializerError` with a non-null cause naming `NumberFormatException`/`NullPointerException` | Correct. The **cause** is your real bug. This is the only place it is ever reported. |
| Step 5: `NoClassDefFoundError` with a message like "Could not initialize class ..." | **The point of the exercise.** Same class, same defect, completely different and much less useful error. |
| Step 5 also has a non-null `getCause()` chaining back to the original | Some JDK builds now chain the original cause onto the follow-up error. **I am not certain which JDK versions do this and will not guess.** If yours does, that is a large quality-of-life improvement and you should know it. Record the answer for your runtime. |
| Step 5 throws `ExceptionInInitializerError` again rather than `NoClassDefFoundError` | Surprising. Re-check that step 4 actually ran and did not silently succeed — the JVM only poisons the class if `<clinit>` genuinely threw. |
| The whole program runs with no errors | `ORDERFLOW_GBP_EUR` is set in your shell. `unset ORDERFLOW_GBP_EUR` and re-run. |

Now run it again with class-loading logging on, which is the tool you will use for the
rest of your career:

```bash
java -Xlog:class+load=info -cp out com.orderflow.lab.classload.InitOrderProbe \
  | grep -i rates
```

**WHAT TO LOOK FOR:** the point in the output at which `Rates` is *loaded*, relative to
the `<clinit>` print. They are different moments, and the log proves it.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised, Spring Boot fat jar |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Load mix | 70% catalogue read, 20% order read, 10% order placement, open-model arrival |
| Order placement rate | roughly 40 requests per second at baseline |
| Latency budget | `POST /orders` p99 within the recorded baseline |
| Baseline artefacts | `/docs/java/baselines/` |
| Deploy | rolling, readiness probe on `GET /actuator/health` |

### The change that causes the incident

A currency-conversion feature lands. The implementation is small, reviewed, and looks
completely idiomatic to anyone who has not read this document.

```java
package com.orderflow.pricing;

import java.io.InputStream;
import java.util.Properties;

/**
 * Static holder for FX rates. "It's just config, it never changes at runtime."
 */
public final class CurrencyRates {

    private static final Properties RATES = new Properties();

    static {
        try (InputStream in = CurrencyRates.class
                .getResourceAsStream("/fx/rates.properties")) {
            RATES.load(in);                      // NPE if the resource is absent
        } catch (Exception e) {
            throw new IllegalStateException("cannot load FX rates", e);
        }
    }

    private CurrencyRates() { }

    public static long convertMinor(long amountMinor, String from, String to) {
        double rate = Double.parseDouble(RATES.getProperty(from + "_" + to));
        return Math.round(amountMinor * rate);
    }
}
```

And it is used from exactly one place, on the order-placement path:

```java
// inside OrderService.placeOrder(...)
long chargeMinor = CurrencyRates.convertMinor(order.totalMinor(),
                                              order.currency(), wallet.currency());
```

The resource `src/main/resources/fx/rates.properties` exists in the repository. It is
excluded from the container image by a `.dockerignore` rule someone added months ago to
keep test fixtures out of the image. Nobody notices, because:

- Unit tests pass — the resource is on the test classpath.
- The Testcontainers integration tests pass — same reason.
- The image builds.
- **The service starts and the readiness probe goes green**, because nothing on the
  health-check path touches `CurrencyRates`.

### What you observe, in the order you observe it

1. Deploy completes. All pods healthy. Dashboards green for roughly the time it takes a
   single order placement to arrive — **seconds**, at 40 rps.
2. `POST /orders` starts returning 500. Catalogue reads (70% of traffic) are completely
   unaffected, so overall error rate looks like a modest single-digit percentage and does
   not trip the page immediately.
3. The application log fills at roughly 40 lines per second with:
   ```
   java.lang.NoClassDefFoundError: Could not initialize class com.orderflow.pricing.CurrencyRates
   ```
   *(illustration of the message shape, not captured output)*
4. The on-call engineer greps the log, sorts by frequency, and finds this error tens of
   thousands of times. It names a class that plainly *exists* — you can see it in the
   jar. `unzip -l app.jar | grep CurrencyRates` confirms it.
5. Someone concludes the class is missing from the image, rebuilds, redeploys. Same
   result. Twenty minutes gone.
6. Someone else suggests a class-loader problem because the class "is there but can't be
   found". Another thirty minutes gone chasing the fat-jar launcher.

**Every one of those minutes is spent because the useful error was thrown once and is
sitting hundreds of thousands of lines earlier in the log.**

### The one command that ends the incident

```bash
# Find the FIRST occurrence, not the most frequent one.
grep -n -m1 'ExceptionInInitializerError' /var/log/orderflow/app.log
```

Or, if you have structured logs, sort ascending by timestamp and take the head of the
error stream rather than aggregating by count. Your log aggregation UI defaults to
"most frequent" or "most recent". **For this failure shape, both defaults are wrong.**

The first occurrence carries the truth:

```
java.lang.ExceptionInInitializerError
  at com.orderflow.orders.OrderService.placeOrder(OrderService.java:88)
  ...
Caused by: java.lang.IllegalStateException: cannot load FX rates
  at com.orderflow.pricing.CurrencyRates.<clinit>(CurrencyRates.java:19)
Caused by: java.lang.NullPointerException: Cannot invoke
  "java.io.InputStream.read(byte[])" because "in" is null
```

*(illustration of the stack-trace shape, not captured output)*

Two things to read off it immediately:

- `<clinit>` in the frame list means **static initializer**. Whenever you see `<clinit>`
  in a stack trace, you are in this document.
- The nested `Caused by` chain ends at the real defect: the resource stream was `null`,
  so the resource is not on the **runtime** classpath.

### Confirm the resource, not the class

```bash
# Is the class in the jar? (It is. This is the misleading check.)
unzip -l orderflow.jar | grep -i CurrencyRates

# Is the RESOURCE in the jar? (This is the check that matters.)
unzip -l orderflow.jar | grep -i 'fx/rates.properties'

# What does the running JVM think? Ask it from inside.
jcmd $(pgrep -f orderflow) VM.system_properties | grep -i class.path
```

| What you see | What it means |
|---|---|
| Class present, resource absent | Confirmed. The `.dockerignore`/build exclusion is the root cause. Fix the packaging. |
| Both present, and it still fails | The resource is present but at a different path — a leading slash, a `BOOT-INF/classes/` prefix difference, or a case difference on a case-sensitive filesystem. `getResourceAsStream("/fx/...")` is loader-relative; `getResourceAsStream("fx/...")` is package-relative. Those are different lookups. |
| Both present and it fails only on some pods | Image drift. Compare image digests, not tags. |
| Error is `ExceptionInInitializerError` on every request rather than only the first | Something is creating a fresh class loader per request — a plugin mechanism, a scripting engine, or a devtools loader left enabled in production. Each new loader gets its own copy of the class and its own chance to fail. That is a *worse* bug than the one you were chasing. |
| The class initializes fine but `convertMinor` throws `NumberFormatException` | Different bug: the resource loaded but the key is missing. Note that this one is *recoverable per request* — because it happens in a normal method, not in `<clinit>`. That contrast is the whole design lesson. |

### The real fix is not "add the resource back"

Adding the file back fixes today. The design fix removes the failure mode:

```java
package com.orderflow.pricing;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;
import java.util.Map;

/**
 * Rates as a Spring bean, not a static holder.
 *
 * Three properties this gains over the static version:
 *   1. A missing/invalid configuration fails at CONTEXT STARTUP, so the readiness
 *      probe never goes green and the rolling deploy halts. Topic 43.
 *   2. The failure is reported once, with the real cause, at a moment somebody is
 *      watching.
 *   3. The class is never poisoned, so a fix does not require a full restart to
 *      become effective in a hot-reload environment.
 */
@Component
@ConfigurationProperties(prefix = "orderflow.fx")
public class CurrencyRates {

    private Map<String, Double> rates = Map.of();

    public void setRates(Map<String, Double> rates) { this.rates = Map.copyOf(rates); }

    public long convertMinor(long amountMinor, String from, String to) {
        Double rate = rates.get(from + "_" + to);
        if (rate == null) {
            throw new UnsupportedCurrencyPairException(from, to);   // Topic 09
        }
        return Math.round(amountMinor * rate);
    }
}
```

> **The rule to carry:** anything that can fail belongs in a constructor, a bean, or a
> factory — never in a `static` block. A constructor failure is a normal exception you
> can catch, retry, or surface at startup. A `<clinit>` failure is a permanent,
> process-wide, evidence-destroying outage.

And if you cannot avoid a static holder — sometimes you inherit one — at minimum make it
fail loudly at startup on purpose, so the failure happens where somebody sees it:

```java
@Component
class StaticHolderWarmup implements ApplicationRunner {
    @Override public void run(ApplicationArguments args) {
        // Force <clinit> during startup, so a failure stops the deploy rather
        // than surfacing on a customer request at peak.
        CurrencyRates.convertMinor(0L, "GBP", "GBP");
    }
}
```

That is a workaround, and you should label it as one in the code. It converts an
unpredictable production failure into a predictable deploy failure, which is a large
improvement and not a fix.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — reading `NoClassDefFoundError` as "the class is missing"

**Wrong:** you see `NoClassDefFoundError`, conclude the class is not on the classpath,
and start rebuilding the image.

**Exact symptom:** the error names a class that demonstrably exists in the artifact.
`unzip -l app.jar | grep TheClass` finds it. Rebuilding and redeploying changes nothing.
The message is usually of the form `Could not initialize class X` — but under pressure
people read the exception *type* and stop.

**Root cause:** `NoClassDefFoundError` has **two** completely different causes, and the
message distinguishes them:

| Message shape | Meaning | Where to look |
|---|---|---|
| `java.lang.NoClassDefFoundError: com/orderflow/orders/OrderService` (a bare name, slashes) | The class was present when this code was **compiled** but is not present at **runtime**. Genuine classpath problem. | Topic 31 `provided` scope, Topic 32 dependency resolution, container image contents |
| `java.lang.NoClassDefFoundError: Could not initialize class com.orderflow.orders.OrderService` | The class was found, and its **static initializer already threw**. It is poisoned. | The **first** `ExceptionInInitializerError` in the log, earlier in time |

And the third member of the family, which people conflate with both:

| `java.lang.ClassNotFoundException: com.orderflow.orders.OrderService` (dotted name) | A **reflective or explicit** lookup failed: `Class.forName`, `loader.loadClass`, a Spring bean class name in YAML, a JDBC driver name, a deserialization `resolveClass` (Topic 19). It is a *checked exception*, which tells you somebody called an API that declared it. | The string that was looked up — often it is a typo or a stale configuration value, not a missing jar |

**Fix:** a decision procedure you can run in ten seconds.

1. Does the message contain "Could not initialize class"? → Search the log for the
   **first** `ExceptionInInitializerError` or the first `<clinit>` frame. Stop looking at
   classpaths.
2. Is it `ClassNotFoundException` with a dotted name? → Find the *string* being looked
   up. It came from configuration, reflection, or a service-loader file. Check for typos
   and for the loader being used.
3. Is it a bare slashed name? → Now it is a real classpath problem. Compare compile
   classpath to runtime classpath (`mvn dependency:tree`, `unzip -l`, `jcmd
   VM.system_properties`).

---

### Trap 2 — the static initializer that reads the environment

**Wrong:**

```java
public final class PaymentGatewayClient {
    private static final String API_KEY = System.getenv("PAYMENT_API_KEY").trim();
    private static final HttpClient HTTP = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(2)).build();
    static {
        // "Fail fast" — a health check at class-init time. This is the worst version.
        if (!ping()) { throw new IllegalStateException("gateway unreachable"); }
    }
}
```

**Exact symptom:** the service starts and passes readiness. Then one of three things,
depending on the environment:

- `PAYMENT_API_KEY` unset in one namespace → `ExceptionInInitializerError` caused by
  `NullPointerException` on the first payment, then permanent `NoClassDefFoundError`.
- The gateway has a 30-second blip during startup warm-up → the class is poisoned **for
  the lifetime of the pod** by a transient failure that resolved itself seconds later.
  The pod is now permanently broken while looking healthy.
- It works in every environment except the one where a different pod happened to make the
  first payment call during a deploy.

The tell that makes this diagnosable: the failure is **sticky per pod**. Some pods serve
payments fine and others fail 100% of the time, with identical images and identical
config. Retrying the request never helps; a pod restart always does.

**Root cause:** class initialization is a one-shot, unretryable, process-wide event.
Putting anything environment-dependent, network-dependent, or time-dependent inside it
means a transient failure becomes permanent. The "fail fast" instinct is right; the
*place* is wrong.

**Fix:**

- Move it to a bean with constructor injection (Topic 39). A failing constructor fails
  the context, which fails the readiness probe, which stops the rolling deploy. That is
  fail-fast done properly.
- If it must be a static holder, use the **initialization-on-demand holder** pattern
  only for things that cannot fail (pure computation), and never for I/O.
- Never put a network call in a `static` block. Not once. Not "just a health check".

**How to find them all in your codebase, right now:**

```bash
# Static blocks that touch the environment, the filesystem, or the network.
grep -rn --include=*.java -A5 'static\s*{' src/main/java \
  | grep -E 'getenv|getProperty|getResourceAsStream|new File|HttpClient|Socket|DriverManager'
```

Every hit is a candidate incident. Triage them by whether the operation can fail.

---

### Trap 3 — class-initialization deadlock between two classes

**Wrong:**

```java
public final class OrderStatusCodes {
    static final Map<String, String> BY_CODE = new HashMap<>();
    static {
        BY_CODE.put("PLACED", PaymentStatusCodes.describe("AUTH"));   // touches the other class
        BY_CODE.put("PAID",   PaymentStatusCodes.describe("CAPTURED"));
    }
}

public final class PaymentStatusCodes {
    static final Map<String, String> BY_CODE = new HashMap<>();
    static {
        BY_CODE.put("AUTH",     OrderStatusCodes.BY_CODE.getOrDefault("PLACED", "placed"));
        BY_CODE.put("CAPTURED", "captured");
    }
    static String describe(String code) { return BY_CODE.get(code); }
}
```

**Exact symptom:** the service hangs. Not slowly — **stops**. Requests time out. CPU is
near zero. Heap is fine. GC log is quiet. Restarting fixes it, sometimes, and then it
happens again days later at a different time of day. It correlates with traffic bursts
and with cold starts, which makes it look like a capacity problem.

The diagnostic tell is in the thread dump:

```bash
jcmd $(pgrep -f orderflow) Thread.print > threads.txt
grep -n '<clinit>' threads.txt
```

You will find two (or more) threads whose stack top is a `<clinit>` frame, each waiting,
and **no "Found one Java-level deadlock" section**, because `jstack`'s deadlock detector
finds monitor and `ReentrantLock` cycles — not class-initialization-lock cycles.

**Root cause:** initialization takes a per-class lock and holds it while running
`<clinit>`. Thread A enters `OrderStatusCodes.<clinit>`, takes lock O, and needs
`PaymentStatusCodes`. Thread B simultaneously enters `PaymentStatusCodes.<clinit>`, takes
lock P, and needs `OrderStatusCodes`. Classic circular wait — with locks the JVM took on
your behalf, at points you did not write.

Note the nasty asymmetry with the *single-threaded* circular case: on one thread, the
recursive-entry rule lets execution proceed with default values (Trap 4). On two threads,
it deadlocks. **The same code is a silent wrong answer or a total hang depending on
concurrency.** Under the Topic 65 load profile, both are reachable.

**Fix:**

- Break the cycle. Static state that references other static state across classes is the
  design defect; the deadlock is the symptom.
- If the tables are genuinely coupled, put them in **one** class so there is one lock.
- Better: make them beans, or `enum` constants, or immutable `Map.of(...)` constants with
  no cross-class references.
- If you must keep it, **force the initialization order deterministically at startup**,
  single-threaded, before traffic arrives — the same `ApplicationRunner` warm-up trick as
  before, and again label it a workaround.

**How to confirm you have fixed it:** the drill in the Failure drill section
includes a reproduction. A fix that you cannot reproduce the failure against is a hope.

---

### Trap 4 — circular static initialization producing a silent zero

**Wrong:** the `Catalog` / `Defaults` pair from the Machine-level reality section, or its
much more common real-world shape:

```java
public enum OrderStatus {
    PLACED, PAID, SHIPPED;
    static final Set<OrderStatus> TERMINAL = Set.of(SHIPPED);
    // ... and elsewhere ...
}

public final class StatusPolicy {
    static final int MAX_TRANSITIONS = OrderStatus.values().length * Limits.FACTOR;
}

public final class Limits {
    static final int FACTOR = compute();      // compute() touches StatusPolicy
    static int compute() { return StatusPolicy.MAX_TRANSITIONS > 0 ? 2 : 1; }
}
```

**Exact symptom:** a constant is `0`, `null`, `false` or an empty collection **in
production but not in your tests**, and no exception is thrown anywhere. Downstream you
see a limit of zero causing every request to be rejected, or a `null` map causing an NPE
several classes away from the actual defect. The value depends on which class was touched
first, so it changes between environments, between test runs, and between a warm and a
cold JVM.

**Root cause:** step 2 of the initialization procedure. When a thread re-enters
initialization of a class it is *already initializing*, the JVM returns immediately and
lets the read proceed against the **prepared** value — the default assigned during
Preparation, before `<clinit>` ran. No error is raised because, per the specification,
nothing is wrong.

**Fix:**

- Remove the cycle. There is no configuration that makes this safe.
- Prefer `enum` singletons and `Map.of(...)`/`List.of(...)` constants with no
  cross-class dependencies.
- Where a genuinely derived constant is needed, compute it lazily in a method or via the
  holder idiom, so the read happens after both classes are fully initialized.
- Add a test that asserts the constant is non-default, and — this is the part people skip
  — **run it in a JVM that touches the classes in the opposite order.** A single test
  ordering proves nothing here.

**Detection command:**

```bash
# Ask which classes the JVM initialized, and in what order, on a real startup.
java -Xlog:class+init=info:file=init.log -jar orderflow.jar
```

**One-line version caveat:** the `class+init` tag is not available in every JDK product
build — some class-loading tags are debug-build-only. Settle it on your runtime with
`java -Xlog:help` and check whether `class+init` is listed; if not, use
`-Xlog:class+load=info` for the load order and instrument the static blocks with a print
statement for the init order.

---

### Trap 5 — the compile-time constant that never updates

**Wrong:** a shared library declares

```java
// in orderflow-common, version 1.4.0
public final class Limits {
    public static final int MAX_ORDER_LINES = 500;       // compile-time constant
}
```

and the service uses it:

```java
if (order.lines().size() > Limits.MAX_ORDER_LINES) { throw new TooManyLinesException(); }
```

The library is bumped to 1.5.0 with `MAX_ORDER_LINES = 2000`. The dependency is updated.
The service is redeployed **without recompiling**, or a module in a multi-module build is
not rebuilt because its own sources did not change.

**Exact symptom:** the limit is still 500 in production. The new jar is definitely on the
classpath — you can decompile it and see 2000. `javap -c` on the *consumer* class shows
`sipush 500` where you expected a field read. No error, no warning, no log line. Just a
value that refuses to change.

**Root cause:** `javac` inlines `static final` fields whose initialiser is a
**compile-time constant expression** (primitives and `String` literals) directly into the
caller's constant pool, per JLS 13.1. The consumer's bytecode does not contain a
reference to `Limits` at all — so the class is never loaded, the new value is never read,
and there is nothing to go wrong at runtime. It already went wrong at compile time,
months ago.

**Fix:**

- For values that might change across a library boundary, **defeat constant folding**:

```java
public final class Limits {
    // Not a compile-time constant expression: the value is read at class init.
    public static final int MAX_ORDER_LINES = Integer.parseInt("2000");
    // or, better, make it a method:
    public static int maxOrderLines() { return 2000; }
}
```

- Better still, do not put tunables in a library constant at all. Put them in
  configuration (Topic 43), where changing them does not require a recompile of anything.
- Prove which one you have with bytecode:

```bash
javap -c -p target/classes/com/orderflow/orders/OrderValidator.class | grep -n -B2 -A2 '500\|MAX_ORDER_LINES'
```

**WHAT TO LOOK FOR:** a `getstatic` instruction referencing `Limits.MAX_ORDER_LINES`
means the value is read at runtime and a library bump will take effect. A literal
`sipush 500` / `ldc` with the value baked in means it was inlined and a library bump will
not. Topic 76 is the full treatment; this one check is worth doing today.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call
it captured. What follows is the exact command, what to look for, and how to read every
plausible result — including the ones that contradict the expected answer.

### Setup

```bash
mkdir -p ~/java-lab/67/logs && cd ~/java-lab/67
java --version              # expect 21 or 25; record which
java -Xlog:help | head -40  # confirm which -Xlog tags your build supports
```

That last command matters. Some class-related tags are present in all builds
(`class+load`, `class+unload`, `class+path`) and some are not. Check before you rely on
one, rather than concluding a subsystem is inactive because a tag printed nothing.

### Proof 1 — watch classes load, lazily, on the real service

```bash
java -Xlog:class+load=info:file=logs/load.log:time,uptime -jar orderflow.jar
```

Then, while it runs:

```bash
wc -l logs/load.log                                   # classes loaded so far
grep -c 'source: jrt:' logs/load.log                  # from the JDK itself
grep -c 'BOOT-INF' logs/load.log                      # from your fat jar
grep -c 'source: __JVM_DefineClass__' logs/load.log   # generated at runtime
```

Now hit one endpoint you have never hit in this process, and count again.

**WHAT TO LOOK FOR:** the delta. Classes loaded by a single first request to a
previously-untouched endpoint is the concrete measure of "laziness" for your service.

| What you see | What it means |
|---|---|
| Several thousand classes at startup, then a few hundred more on the first request to a new endpoint | Normal for Spring Boot. The per-endpoint delta is exactly the first-request latency contribution you cannot JIT your way out of. |
| A large delta on the *second* and later requests too | Something is generating classes per request: a dynamic proxy created per call, a scripting engine, an expression-language compiler, or a serialization library building a codec per type. This is a metaspace growth risk (Topic 80) and a real latency cost. |
| Many lines with `__JVM_DefineClass__` or `Lambda$` in the name | Lambdas (Topic 21) are linked at first execution by `LambdaMetafactory`, which spins a class. Expected, and part of why the first execution of a lambda-heavy path is slow. |
| Almost no classes loaded during requests | Either you have already warmed the path, or CDS/AOT is in play (see Proof 5). |
| The log file is empty | Wrong `-Xlog` syntax, or the process is not the one you think. Test the syntax on its own: `java -Xlog:class+load=info -version` should print hundreds of lines. |

### Proof 2 — see the delegation chain and each class's loader

```java
package com.orderflow.lab.classload;

public final class LoaderFacts {
    public static void main(String[] args) {
        show(String.class);                        // JDK core
        show(java.sql.Driver.class);               // platform
        show(LoaderFacts.class);                   // yours
        show(org.postgresql.Driver.class);         // a third-party jar, if present

        ClassLoader l = LoaderFacts.class.getClassLoader();
        System.out.println("--- delegation chain upward ---");
        while (l != null) { System.out.println("  " + l); l = l.getParent(); }
        System.out.println("  null   (bootstrap)");
        System.out.println("TCCL = " + Thread.currentThread().getContextClassLoader());
    }

    static void show(Class<?> c) {
        System.out.printf("%-40s loader=%s module=%s%n",
            c.getName(), c.getClassLoader(), c.getModule().getName());
    }
}
```

**WHAT TO LOOK FOR:** four things — which classes report `null`, how many links the chain
has, whether the TCCL is the same object as your own loader, and the module names.

| What you see | What it means |
|---|---|
| `String` reports `null`, your class reports an app loader | The expected shape. `null` is bootstrap, not an error. |
| Your class's loader is not `jdk.internal.loader.ClassLoaders$AppClassLoader` but something Boot-specific | You are running from a fat jar and Boot's launcher installed its own loader so it can read nested jars. Normal. **The exact class name has changed across Spring Boot versions — record what yours prints rather than trusting any document.** |
| TCCL is a *different* object from your own loader | Normal in containers and under some launchers. Remember it when you write code that loads classes by name. |
| The chain has more than three links | Something installed a loader: devtools, an agent (Topic 81), a plugin system. Know which, before you debug anything else. |
| Module name is `null` for your classes | You are on the classpath, in the unnamed module. Expected unless you adopted JPMS (Topic 20). |

### Proof 3 — prove that two loaders make two types

```java
package com.orderflow.lab.classload;

import java.net.URL;
import java.net.URLClassLoader;
import java.nio.file.Path;

/**
 * Loads the SAME class file twice, through two loaders that do NOT delegate for it,
 * then tries to assign one to the other.
 */
public final class TwoLoaders {
    public static void main(String[] args) throws Exception {
        URL[] cp = { Path.of("out").toUri().toURL() };

        // Parent = the bootstrap-ish platform loader, so our app classes are NOT
        // visible via delegation and each child must define its own copy.
        ClassLoader parent = ClassLoader.getPlatformClassLoader();
        try (var a = new URLClassLoader("A", cp, parent);
             var b = new URLClassLoader("B", cp, parent)) {

            Class<?> ca = a.loadClass("com.orderflow.lab.classload.Widget");
            Class<?> cb = b.loadClass("com.orderflow.lab.classload.Widget");

            System.out.println("same name?     " + ca.getName().equals(cb.getName()));
            System.out.println("same Class?    " + (ca == cb));
            System.out.println("assignable?    " + ca.isAssignableFrom(cb));

            Object instance = ca.getDeclaredConstructor().newInstance();
            System.out.println("instanceof B's? " + cb.isInstance(instance));
            cb.cast(instance);      // expected to throw
        }
    }
}

class Widget { }
```

**WHAT TO LOOK FOR:** the `ClassCastException` message, and specifically the
parenthesised loader identities.

| What you see | What it means |
|---|---|
| `same name? true`, `same Class? false`, and a `ClassCastException` naming the same class twice | The expected result, and the single most valuable message shape in this document. Read the loader names in the parentheses. |
| `same Class? true` | Delegation found the class through the shared parent, so only one definition exists. Check that `out` is not also on the parent's classpath — if you launched with `-cp out`, the app loader already has it and the platform parent may reach it. Run with an empty `-cp` and point the `URLClassLoader` at the directory instead. |
| `NoClassDefFoundError` inside the child loaders | The URL is wrong. `Path.of("out").toUri().toURL()` must end in a slash for a directory; verify by printing it. |
| It works, but the exception message does **not** name the loaders | Older message format. The diagnosis is the same; you just have to find the loaders yourself with `getClassLoader()`. |

### Proof 4 — count what class loading costs your startup

```bash
# A: normal startup
java -Xlog:class+load=info:file=logs/cold.log -jar orderflow.jar &
# ... wait for the readiness endpoint to return 200, then note the time ...

wc -l logs/cold.log
```

Correlate with the application's own startup timing (Spring prints a "Started ... in Xs"
line). Then compare against a run with class-data sharing, below.

**WHAT TO LOOK FOR:** class count and startup wall time as a pair. Neither number alone
is interesting; the ratio between two configurations is.

### Proof 5 — class-data sharing, and the JDK 25 caveat

Class-data sharing (CDS) pre-parses class metadata into a memory-mappable archive, so
startup skips a large amount of parsing and verification.

```bash
# Record an archive from a real run of YOUR application.
java -XX:ArchiveClassesAtExit=logs/orderflow.jsa -jar orderflow.jar
#   ... let it start, hit a few endpoints, then shut it down cleanly ...

# Use it.
java -XX:SharedArchiveFile=logs/orderflow.jsa -jar orderflow.jar

# Confirm sharing is actually happening.
java -XX:SharedArchiveFile=logs/orderflow.jsa -Xlog:class+load=info -jar orderflow.jar \
  | grep -c 'shared objects file'
```

**WHAT TO LOOK FOR:** the count of classes reported as coming from the shared archive,
and the change in startup time.

| What you see | What it means |
|---|---|
| Many classes sourced from the shared archive, measurably faster startup | CDS is working. This is the cheapest startup win available and it costs no application changes. |
| Archive created but zero classes shared at runtime | The runtime classpath differs from the dump-time classpath. CDS requires them to match; even ordering matters. Compare both. |
| A warning that the archive was disabled | The JVM tells you why. Read it — the usual causes are a classpath mismatch, a different JDK version, or an agent (Topic 81) transforming classes. |
| No difference in startup at all | Your startup may be dominated by something else — database connection establishment, Flyway migrations, or bean creation. Class loading is a *component* of startup, not all of it. Measure before concluding. |

> **One-line version note, flagged honestly:** Project Leyden added ahead-of-time class
> loading and linking (an "AOT cache") in a JDK newer than 21, with further ergonomics
> and profiling work in the 24/25 timeframe. **I am not certain of the exact JDK for each
> piece and will not guess.** Settle it on your runtime:
> `java -XX:+PrintFlagsFinal -version | grep -i -E 'AOT|SharedArchive|CDS'` and
> `java -XX:AOTMode=help -version` (which will either explain itself or reject the flag —
> both answers are informative). Nothing else in this document depends on the answer.

### Proof 6 — find who is loading classes at runtime

```bash
jcmd <pid> VM.classloaders
jcmd <pid> VM.classloader_stats
jcmd <pid> VM.metaspace
jcmd <pid> help                # the authoritative list for YOUR JDK
```

**WHAT TO LOOK FOR:** the number of loaders, and whether it grows over time.

| What you see | What it means |
|---|---|
| A small, fixed number of loaders (bootstrap, platform, app, maybe one launcher) | Healthy. |
| Loader count growing steadily under load | A class-loader leak in progress. Each retained loader retains all its classes and their static state. This becomes Topic 79's heap-dump investigation, and it will show up as metaspace growth in `VM.metaspace`. |
| Hundreds of loaders with generated-looking names | A bytecode-generation library making one loader per class. Common with some mocking, serialization and expression libraries. Usually configurable to reuse a loader. |
| Metaspace committed growing without bound while heap is flat | Classic classloader leak signature. Note it is *not* a heap problem, so raising `-Xmx` does nothing (Topic 66, Trap 1). |

---

## Failure drill

**Mandatory.** Do not read the "how to read it" table until you have produced the failure
yourself and written down what you saw. The point is not the knowledge; it is the memory
of the moment the second error told you nothing and you knew where to look anyway.

### The assignment, restated from the master plan

> Throw from a static initializer, catch the first `ExceptionInInitializerError`, then
> touch the class again and get the misleading `NoClassDefFoundError`.

We will do it three times: standalone, then on `orderflow` under the Topic 65 load
profile, then as a concurrent deadlock.

### Part A — standalone, ninety seconds

```java
package com.orderflow.lab.classload;

public final class PoisonDrill {

    public static void main(String[] args) {
        for (int attempt = 1; attempt <= 3; attempt++) {
            try {
                System.out.println("attempt " + attempt + ": calling Rates.rate()");
                System.out.println("  got " + Rates.rate("GBP_EUR"));
            } catch (Throwable t) {
                System.out.println("  threw " + t.getClass().getName());
                System.out.println("  message = " + t.getMessage());
                System.out.println("  cause   = " + t.getCause());
            }
        }
    }
}

final class Rates {
    private static final java.util.Map<String, Double> TABLE;
    static {
        System.out.println("  [<clinit> entered]");
        throw new IllegalStateException("FX feed unavailable");
    }
    static double rate(String pair) { return TABLE.get(pair); }
}
```

```bash
javac -d out PoisonDrill.java && java -cp out com.orderflow.lab.classload.PoisonDrill
```

Write down, before reading anything: the exception type on attempt 1, the exception type
on attempts 2 and 3, and whether `[<clinit> entered]` printed once or three times.

### Part B — the same failure on `orderflow`, under load

**Step 0 — establish the control.** Re-run the Topic 65 baseline load profile unchanged.
Confirm p50/p95/p99/p999, throughput and error rate are within ±10% of the committed
values in `/docs/java/baselines/`. If they are not, stop: the gate rule applies, and a
drill against an unstable baseline proves nothing.

**Step 1 — poison a class on the order-placement path only.**

```java
package com.orderflow.lab;

/**
 * DRILL CODE. Never merge this.
 * Poisons itself on first use, but ONLY when the drill env var is set,
 * so the same image is safe to run without it.
 */
public final class FxRates {

    private static final double GBP_EUR;

    static {
        if (System.getenv("ORDERFLOW_DRILL_POISON") != null) {
            throw new IllegalStateException("drill: FX feed unavailable");
        }
        GBP_EUR = 1.17d;
    }

    private FxRates() { }

    public static long convertMinor(long minor) { return Math.round(minor * GBP_EUR); }
}
```

Call it from `OrderService.placeOrder` — the 10% slice of the Topic 65 mix — and from
nowhere else. Catalogue reads must not touch it; that asymmetry is half the lesson.

**Step 2 — run the baseline profile with the drill enabled.**

```bash
ORDERFLOW_DRILL_POISON=1 java -Xmx1200m -Xms1200m \
  -Xlog:class+load=info:file=/var/log/orderflow/class.log:time,uptime \
  -jar orderflow.jar

k6 run --out json=logs/drill.json load/baseline.js
```

**Step 3 — capture, before analysing.** Write down:

1. The **wall-clock time** of the first `ExceptionInInitializerError` in the app log.
2. The count of `ExceptionInInitializerError` over the whole run.
3. The count of `NoClassDefFoundError` over the whole run.
4. The ratio between those two counts.
5. k6's error rate for `POST /orders` and, separately, for `GET /products`.
6. Whether the readiness probe ever went red.
7. `grep -c FxRates /var/log/orderflow/class.log` — how many times the class was *loaded*.

```bash
grep -c 'ExceptionInInitializerError' /var/log/orderflow/app.log
grep -c 'NoClassDefFoundError'        /var/log/orderflow/app.log
grep -n -m1 'ExceptionInInitializerError' /var/log/orderflow/app.log
```

### How to read it

| What you see | What it means |
|---|---|
| Exactly **one** `ExceptionInInitializerError` and thousands of `NoClassDefFoundError` | **The drill has fired.** This ratio is the entire lesson. Write the sentence: "the useful error occurred once, at time T, and every error after it is an echo containing no diagnostic information." |
| `POST /orders` error rate near 100%, `GET /products` unaffected | Correct and important. A poisoned class breaks only the paths that touch it — which is why the overall error rate can look survivable while one business-critical flow is completely dead. |
| The readiness probe stayed green the whole time | Expected, and the reason this failure survives a rolling deploy. Note it as an argument for a deeper health check, and note the counter-argument: a health check that touches every class is a health check that can fail for reasons unrelated to health (Topic 121). |
| `FxRates` appears exactly **once** in `class.log` | Loading happened once. The class was not reloaded and not re-attempted. Confirms the permanence claim directly. |
| More than one `ExceptionInInitializerError`, at different times | Something is creating new class loaders — devtools, a plugin mechanism, a scripting engine. Each fresh loader gets a fresh copy of the class and a fresh chance to fail. Find the loader with `jcmd VM.classloaders`; this is a more serious finding than the drill itself. |
| Zero `NoClassDefFoundError`, only repeated `ExceptionInInitializerError` | Your framework is catching and re-wrapping, or the call site is behind a proxy that swallows and retries. Look at what sits between the controller and `FxRates` — a Spring proxy (Topic 40), a resilience library, a retry aspect. |
| The whole service falls over rather than failing one endpoint | Something on a shared path — a `@ControllerAdvice`, a filter, a metrics interceptor — also touches the class. Find it: that is a coupling you did not know you had. |
| Errors stop after a while without a restart | Genuinely surprising, and worth chasing: it means a *different* class loader started serving requests. Investigate; do not celebrate. |

### Part C — the concurrent variant (the one that hangs)

Take Trap 3's two mutually-referencing classes, add them to `orderflow`, and touch them
from two endpoints that the Topic 65 mix hits simultaneously — for example, one from the
catalogue read path and one from the order read path.

```bash
# Run the load profile from a COLD JVM so first-touch happens under concurrency.
k6 run load/baseline.js

# The moment throughput flatlines:
jcmd $(pgrep -f orderflow) Thread.print > threads.txt
grep -n '<clinit>' threads.txt
grep -n 'Found one Java-level deadlock' threads.txt
```

| What you see | What it means |
|---|---|
| Two or more threads with `<clinit>` frames, and **no** Java-level-deadlock section | **The drill has fired.** Class-initialization deadlock. The absence of the deadlock section is the diagnostic, not a reason to look elsewhere. |
| It does not reproduce | Timing. Increase the arrival rate, ensure a genuinely cold JVM (no CDS archive, no warm-up runner), and make both `<clinit>` blocks slower with a short `Thread.sleep` to widen the window. Widening a race to prove it exists is legitimate; shipping the sleep is not. |
| One thread in `<clinit>` and many threads blocked elsewhere | A single slow `<clinit>` — for instance one doing I/O — serialising every thread behind it. Not a deadlock, but the same lock, and a real latency cliff worth understanding. |
| A Java-level deadlock **is** reported | Then you have a monitor deadlock too, probably because a `<clinit>` took a `synchronized` lock. Two bugs. Fix the class cycle first. |

### What the drill proves

- The **first** error is the only one with information. Every incident procedure you write
  from now on should say "sort ascending by time, do not aggregate by frequency".
- Initialization failure is **permanent**. There is no retry, and a transient cause
  produces a permanent effect.
- Class initialization takes a lock you did not write, which means it can deadlock in
  ways `jstack`'s deadlock detector does not report.
- A readiness probe that does not exercise a code path cannot protect that code path.

Carry one sentence out of this drill:

> *"Could not initialize class X" is not a fact about X's presence. It is a pointer to an
> earlier line in the log, and the earlier line is the incident.*

---

## Measurement

### The instrument for this topic

Class loading is not measured with a stopwatch. It is measured with **counts and log
timelines**, compared against a control configuration.

The three numbers:

| Number | Command | Why you want it |
|---|---|---|
| Classes loaded at startup | `java -Xlog:class+load=info:file=load.log ... ; wc -l load.log` | The size of the work that must happen before your first request. The denominator for any CDS/AOT comparison. |
| Classes loaded *after* readiness, per endpoint | count the log before and after the first request to each endpoint | The concrete first-request latency you cannot fix with JIT warm-up (Topic 74) alone. |
| Loader count and metaspace over time | `jcmd <pid> VM.classloaders`, `jcmd <pid> VM.metaspace` | Detects the classloader leak shape early, while it is still cheap to fix. |

### The protocol

1. **Identical load profile.** The same k6 script, the same arrival rate, the same
   dataset, the same cache state as the Topic 65 baseline.
2. **Cold JVM.** Class loading is a startup and first-touch phenomenon. A warm process
   measures nothing.
3. **Change exactly one thing** — CDS on/off, AOT cache on/off, an eager-warmup runner
   on/off.
4. **Compare the first N seconds**, bucketed. Class-loading effects live in the first
   30–120 seconds and then vanish; a five-minute average hides them completely.
5. **Repeat at least three times.** Startup timing is noisy — page cache state alone can
   move it substantially.

### Deriving the startup class-loading timeline

```bash
# Classes loaded per second of uptime, from your own log.
awk -F'[][]' '/class,load/ {print int($4)}' /var/log/orderflow/class.log \
  | sort -n | uniq -c
# columns: count, uptime-second
```

**WHAT TO LOOK FOR:** the shape of the curve, not any single value.

| What you see | What it means |
|---|---|
| A tall spike in the first few seconds, then a long tail decaying toward zero | Normal. The spike is framework bootstrap; the tail is lazy loading as endpoints are touched for the first time. |
| A second spike well after readiness | An endpoint or a scheduled job touched a whole subsystem for the first time. If that endpoint is customer-facing, its first request paid for all of it. Consider a warm-up. |
| A flat, non-zero rate that never decays | Runtime class generation. Find it (Proof 6) — it is a metaspace risk and a latency cost. |
| A spike that coincides exactly with a `Metadata GC Threshold` line in your GC log | Class loading is driving metaspace growth hard enough to trigger collections. Topic 71 lists that pause cause; Topic 80 covers sizing metaspace. |

### Why a naive `System.nanoTime()` measurement is WRONG here

You will be tempted to write this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long t0 = System.nanoTime();
Class.forName("com.orderflow.pricing.CurrencyRates");
System.out.println((System.nanoTime() - t0) + " ns to load");
```

Five independent reasons it lies, and you cannot tell which one is dominating:

1. **It is a one-shot event.** Class loading happens exactly once. You cannot loop it to
   average out noise, and a single sample of anything on a modern machine has enormous
   variance. Every technique JMH uses to get a trustworthy number is unavailable to you
   here by construction.
2. **You are measuring the disk and the page cache.** The first run reads the jar from
   storage; the second reads it from the OS page cache. The difference can be an order of
   magnitude and has nothing to do with the JVM.
3. **You are measuring the interpreter.** At startup nothing is compiled. The same
   `<clinit>` executed later, warm, in compiled code, would take a different time — so
   your number does not generalise (Topic 74).
4. **Loading is transitive and you do not control the boundary.** Loading one class loads
   its superclasses, its interfaces, the types in its `<clinit>`, and whatever those
   touch. Your timer brackets an unknown quantity of work.
5. **Verification cost depends on the bytecode, not the class count.** A class with one
   enormous method verifies differently from ten small ones. Counting classes and timing
   them are two different measurements and you are conflating them.

Topic 77 is the full treatment of why hand-rolled timing is wrong in general. The
class-loading-specific point is sharper: **this event is unrepeatable inside one JVM, so
the correct instrument is a comparison between two JVM startups, not a timer inside one.**

### The one thing worth timing, and how

Startup-to-readiness, across configurations, from outside the JVM:

```bash
for i in 1 2 3; do
  /usr/bin/time -f "%e" java -XX:SharedArchiveFile=logs/orderflow.jsa -jar orderflow.jar &
  until curl -sf localhost:8080/actuator/health >/dev/null; do :; done
  # record elapsed, then shut down cleanly
done
```

Report three runs and their spread, never a single number. If the spread between
configurations is smaller than the spread within a configuration, you measured nothing —
the same discipline as every other measurement in Phase 8.

---

## Practice exercises

### 1 — Easy: build your own trigger table

Write one class `Probe` whose static block prints a marker. Then write a driver that, in
separate JVM runs (one per case, so each starts cold), performs exactly one of the
following and reports whether the marker printed:

1. `Probe[] a = new Probe[5];`
2. `System.out.println(Probe.NAME);` where `NAME` is `static final String = "x"`
3. `System.out.println(Probe.COUNT);` where `COUNT` is `static final int` computed by a
   method call
4. `Probe.staticMethod();`
5. `new Probe();`
6. `Class.forName("Probe")`
7. `Class.forName("Probe", false, loader)`
8. `loader.loadClass("Probe")`
9. `SubProbe.staticMethodDeclaredOnProbe();` where `SubProbe extends Probe`
10. `new SubProbe();`

Produce a table with two columns: the operation, and initialized yes/no. Then answer in
one sentence each:

- Which two cases surprised you, and why?
- Case 2 does not even *load* the class. Prove it with `-Xlog:class+load=info` and say
  what that implies for a library that changes a constant.
- Case 9 initializes exactly one of the two classes. Which, and why does the JLS make
  that choice?

### 2 — Medium: the audit (combines Topics 01, 09, 17, 19, 31, 32, 39, 43)

This class is on the `orderflow` order-placement path at 40 rps. It contains **six**
distinct defects. Four are from this topic; two are from earlier ones. For each: name the
topic, state the **observable** symptom on-call (not "it's bad practice" — what does the
engineer actually see in a log, a metric, or a thread dump?), and write the fix.

```java
package com.orderflow.payments;

import java.io.InputStream;
import java.util.*;

public final class PaymentRouter {

    public static final int MAX_RETRIES = 3;

    private static final Properties ROUTES = new Properties();
    private static final Map<Long, String> DECISION_CACHE = new HashMap<>();
    private static final String REGION;

    static {
        REGION = System.getenv("ORDERFLOW_REGION").toUpperCase();
        try (InputStream in = PaymentRouter.class.getResourceAsStream("/routes.properties")) {
            ROUTES.load(in);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        if (!GatewayHealth.isUp(ROUTES.getProperty("primary"))) {
            throw new IllegalStateException("primary gateway down at startup");
        }
    }

    public static String route(Long orderId, Long amountMinor) {
        String cached = DECISION_CACHE.get(orderId);
        if (cached != null) { return cached; }

        String gateway;
        try {
            Class.forName(ROUTES.getProperty("driver." + REGION));
            gateway = ROUTES.getProperty("gateway." + REGION);
        } catch (ClassNotFoundException e) {
            gateway = "fallback";
        }

        for (Long seen : DECISION_CACHE.keySet()) {
            if (seen == orderId) { return DECISION_CACHE.get(seen); }
        }

        DECISION_CACHE.put(orderId, gateway);
        return gateway;
    }
}
```

Hints, in the order to think about them: one defect makes a transient outage permanent;
one makes the whole class poisonable by a missing environment variable; one is a
retention problem that will show up in Topic 79; one is a concurrency problem that will
show up in Topic 92; one is a Topic 01 boxing comparison that passes every test with
small fixtures; and one is a constant that will not update when the library does.

For the poisoning defects specifically: state precisely which error the on-call engineer
sees on request number 2, and where in the log they must look to find the truth.

### 3 — Hard: production simulation — diagnose three failures from evidence alone

**Part A — reproduce and record the control.** Run the Topic 65 baseline with
`-Xlog:class+load=info` to a file. Confirm you are within ±10% of the committed baseline.
Record: p50/p95/p99/p999, throughput, error rate, classes loaded at startup, classes
loaded during the run, loader count, and metaspace committed at the end.

**Part B — introduce three independent failures, one at a time.** Between each, restore
the baseline and confirm you are back within ±10%.

1. **Poisoned class on one endpoint** — the `FxRates` drill from Part B above.
2. **Class-initialization deadlock** — the mutually-referencing pair from Trap 3, touched
   from two endpoints the load mix hits concurrently.
3. **Runtime class generation leak** — a code path that creates a `new URLClassLoader` per
   request and loads one class through it, retaining the loader in a static list. (This is
   an artificial stand-in for the real-world shapes: a scripting engine, an expression
   compiler, or a serialization library caching codecs badly.)

For each, record the same nine numbers plus: the count of the first-seen error type, the
count of the follow-up error type, and whether the readiness probe went red.

**Part C — the blind test.** This is the exercise that actually teaches the topic. Hand
the three log sets to someone else — or to yourself in a week — **with the labels
removed.** For each, write down:

- The single command you would run first.
- Whether the problem is a missing class, a poisoned class, a loader-identity problem, or
  a loader leak.
- The exact evidence line that decides it.
- The fix, and whether it requires a restart to take effect.

Then check against your Part B notes. **Any log you cannot diagnose from the log alone is
a gap in your reading skill, not in your class-loading knowledge.**

**Part D — the deploy-safety question.** For failure (1), design a change to the
readiness probe that would have caught it before the pod took traffic. Then argue the
other side: name at least two concrete ways a readiness probe that eagerly initializes
application classes makes your deploys *less* reliable. State which you would ship for
`orderflow` and why. (Topic 121 is the full treatment; this is the version you can answer
today.)

**Part E — argue against yourself.** You will conclude that static initializers should
never do I/O. Make the strongest possible case for a static initializer that reads
configuration — there is a real one involving startup latency and immutability. Then say
exactly what would have to be true about the configuration source for that case to win.

---

## Interview questions

### Q1 — "You see `NoClassDefFoundError` in production. Walk me through it."

**MID-LEVEL answer:** "That means a class is missing at runtime. I'd check the classpath
— maybe a dependency has `provided` scope, or the jar didn't get into the container
image. I'd compare the build output with what's actually deployed and redeploy."

**SENIOR answer:** "First I'd read the message, because `NoClassDefFoundError` has two
completely different causes and they need opposite investigations.

If the message is a bare slashed class name, it's a genuine linkage failure: the class
was present at compile time and isn't present at runtime. That's a build or packaging
problem — `provided` scope, a shading mistake, a nearest-wins resolution surprise from
Maven's flat classpath, or a jar missing from the image. `mvn dependency:tree` and
`unzip -l` settle it.

If the message says **'Could not initialize class'**, the class is present and its static
initializer already threw. The class is poisoned for the lifetime of the JVM, and **every
error I'm currently looking at is an echo containing no information.** The real exception
was an `ExceptionInInitializerError`, thrown once, earlier in time. So I'd stop grepping
by frequency and search for the *first* occurrence — sort ascending by timestamp, take
the head, not the top of an aggregation. That single change of search strategy is
usually the whole diagnosis.

And I'd distinguish both from `ClassNotFoundException`, which is a checked exception
raised by a *reflective* lookup — `Class.forName`, a driver name, a bean class name in
YAML, a `resolveClass` during deserialization. That one usually means a wrong string or
the wrong class loader, not a missing jar.

Two operational things I'd carry out of the incident. First, the failure is sticky per
pod: some pods will be permanently broken while looking healthy, because a readiness
probe that never touches the class can't see it. So 'restart the pod' will look like a
fix and won't be one, if the underlying cause is environmental. Second, the fix is
usually to get the failing work out of the static initializer entirely — into a bean or a
constructor — so it fails at context startup where somebody is watching, once, with the
real cause."

**What separates them:** the mid answer treats the exception type as the diagnosis. The
senior answer reads the *message* to branch between two opposite investigations, knows
the poisoning rule, changes the log search strategy accordingly, distinguishes the third
member of the family, and names the design fix rather than only the immediate one. The
detail that most reliably impresses is "sort ascending, don't aggregate by frequency" —
almost nobody volunteers it, and it is the step that actually ends the incident.

**Interviewer's follow-up:** *"You said the class is poisoned. Can you un-poison it?"* —
Not from within the JVM. There is no API and no flag; the erroneous state is per class
per loader and permanent. The only ways out are a restart, or arranging for a *different*
class loader to define a fresh copy — which is what a hot-reload framework does, and
which is itself a source of the loader-identity problems in the same family.

---

### Q2 — "What's the difference between `ClassNotFoundException` and `NoClassDefFoundError`?"

**MID-LEVEL answer:** "`ClassNotFoundException` is checked and happens with reflection;
`NoClassDefFoundError` is an `Error` and happens when the JVM can't find a class it needs.
Both mean the class isn't on the classpath."

**SENIOR answer:** "The type hierarchy is the smaller half of the answer.
`ClassNotFoundException` extends `ReflectiveOperationException`, so it's checked — which
tells you immediately that *somebody called an API that declared it*: `Class.forName`,
`ClassLoader.loadClass`, a `ServiceLoader`, or `ObjectInputStream.resolveClass` during
deserialization. The class name in that message came from a **string**, and strings come
from configuration, so the first thing I check is the string, not the classpath.

`NoClassDefFoundError` is an `Error`, thrown by the JVM itself during linkage. It means
the compiler baked in a reference to a type that isn't resolvable now. Classic cause:
`provided` scope, or a dependency that resolved to a different version on Maven's flat
classpath, so the code compiled against a class that isn't there at runtime.

But the case that actually matters in production is the third one, which the textbook
answer misses entirely: `NoClassDefFoundError` with the message **'Could not initialize
class'**. There the class is present and loadable — its static initializer threw, once,
and the JVM marked it erroneous permanently. Everything after that reports the useless
error while the real exception sits earlier in the log. That's the case where the
textbook answer sends you to check a classpath that is completely fine.

There's also a related pair I keep together mentally: `NoSuchMethodError` and
`NoSuchFieldError`. Same family, same root cause — compiled against one version, running
against another — and they're the runtime shape of a Maven diamond conflict."

**What separates them:** the mid answer stops at checked-versus-unchecked. The senior
answer reads the exception *type* as evidence about *who threw it* — reflection versus
linkage — knows the third case that dominates real incidents, and connects the whole
family to dependency resolution.

**Interviewer's follow-up:** *"How would `ClassNotFoundException` show up during
deserialization?"* — `ObjectInputStream` resolves class names from the stream through a
loader; in an application server or fat jar the loader it picks may not see the
application's classes. That is Topic 19 territory, and it is one of the reasons Java
serialization is painful across deployment boundaries — the object graph is a set of
class *names*, and names are only meaningful relative to a loader.

---

### Q3 — "Why is class initialization lazy, and when does it actually happen?"

**MID-LEVEL answer:** "Java loads classes when they're first used, to save memory and
startup time. Static blocks run then."

**SENIOR answer:** "There are two separate laziness questions and it's worth keeping them
apart. *Loading* — obtaining the bytes and defining the class — can happen at various
points and HotSpot resolves constant-pool references lazily too. *Initialization* —
running `<clinit>` — is specified precisely by JLS 12.4.1 and happens on first **active
use**: a `new`, a static method call, a non-constant static field access, one-argument
`Class.forName`, initializing a subclass, or being the main class.

The list of things that *don't* trigger it is the more useful half. Creating an array of
the type doesn't. `loadClass` doesn't — only `forName` does. `Class.forName(name, false,
loader)` explicitly doesn't. And reading a `static final` compile-time constant doesn't
even *load* the class, because `javac` inlines the value into the caller's constant pool —
which is why bumping a library version can leave a constant stubbornly unchanged with no
error anywhere.

Why lazy: the specification wants class initialization to have observable, deterministic
timing tied to program semantics rather than to whenever the runtime felt like loading
something. If initialization happened at load time, a class's side effects would fire at
an unpredictable moment. Laziness also means you don't pay for code paths you never
execute, which matters a lot for a framework that ships thousands of optional integrations.

The operational consequence is the one I actually care about: **a missing class or a
broken static initializer surfaces at first use, not at startup.** So a deploy can go
fully green and then break when a rarely-used endpoint gets its first request — possibly
hours later, possibly at peak. That's an argument for eagerly exercising critical paths
at startup, and an argument against putting anything fallible in a static initializer in
the first place."

**What separates them:** naming the specification, listing the non-triggers (especially
the inlined constant), explaining *why* the design is lazy rather than just asserting it,
and landing on the deploy-time consequence.

**Interviewer's follow-up:** *"Give me a case where laziness caused a production
incident."* — The compile-time-constant one is the sharpest: a shared library changes a
limit, every service picks up the new jar, and a service that wasn't recompiled keeps the
old value forever with no error. It looks like a caching bug and it is a compilation
artefact. `javap -c` on the consumer settles it in one command.

---

### Q4 — "A `ClassCastException` says class `Order` cannot be cast to class `Order`. Explain."

**MID-LEVEL answer:** "That looks like a JVM bug, or maybe two versions of the same jar
on the classpath."

**SENIOR answer:** "It's a class-loader identity problem, and it's the JVM behaving
exactly as specified. A class's runtime identity is the pair **(binary name, defining
class loader)**. Two loaders that each define `com.orderflow.orders.Order` produce two
distinct runtime types that happen to share a name. They aren't assignable, `instanceof`
is false between them, and they don't share static state.

The message actually tells you this if you read past the class names — modern JVMs print
the module and the defining loader for each side in parentheses. Those two loader
identities *are* the diagnosis.

How you get there: a servlet container with a per-webapp loader, Spring Boot Devtools with
its restart loader, an OSGi bundle wiring, a plugin system, an instrumentation agent, or a
custom loader somebody wrote to hot-reload something. It usually appears when an object
crosses a boundary it wasn't supposed to cross — cached in a static held by a parent
loader, or passed to a listener registered by a different loader.

What I'd do: print the loader for both instances — `obj.getClass().getClassLoader()` —
and walk each loader's parent chain. Then decide whether the fix is to stop the class
being defined twice (move it to a shared parent loader so delegation finds one copy), or
to stop the object crossing the boundary (communicate across it with an interface loaded
by the common ancestor, or with plain data).

There's a related error worth mentioning in the same breath: `IllegalAccessError` on
package-private access. Runtime packages are also identified by (name, loader), so two
same-named packages from different loaders can't touch each other's package-private
members even though the source looks obviously legal."

**What separates them:** knowing that identity is (name, loader) rather than name;
reading the loader names out of the message; naming the realistic mechanisms; proposing
two structurally different fixes; and connecting it to the runtime-package rule.

**Interviewer's follow-up:** *"Why would anyone want two loaders defining the same
class?"* — Isolation. It's the mechanism that lets one JVM host two applications that
depend on incompatible versions of the same library, which is exactly the problem Maven's
flat classpath cannot solve. The identity crisis is the price of the isolation, not an
accident.

---

### Q5 — "Your service hangs under load with near-zero CPU. Thread dump shows threads in `<clinit>`. What happened?"

**MID-LEVEL answer:** "Some kind of deadlock. I'd look for the synchronized blocks and
see which locks are held."

**SENIOR answer:** "That's a class-initialization deadlock, and the giveaway is that
`jstack` won't report it in its 'Found one Java-level deadlock' section — that detector
finds monitor and `ReentrantLock` cycles, and this cycle is on the JVM's per-class
initialization locks, which aren't ordinary monitors. So the absence of a deadlock report
is evidence *for* this diagnosis, not against it.

The mechanism: JLS 12.4.2 makes class initialization thread-safe by taking a per-class
lock and holding it for the whole of `<clinit>`. If class A's static initializer touches
class B and B's touches A, and two threads enter through different doors at the same
instant, each holds one lock and waits for the other. Neither can proceed. It's a
textbook circular wait with locks nobody wrote.

The sharp detail is that the *same code* behaves completely differently single-threaded.
On one thread, the recursive-entry rule lets the second initialization return immediately
and the read proceeds against the field's **prepared** default — zero, or null. So
single-threaded you get a silent wrong value, and multi-threaded you get a hang. That's
why it passes tests and fails under load, and why it's intermittent: it needs a genuinely
cold JVM and simultaneous first-touch.

Fix: break the cycle — that's a design change, not a tuning one. Collapse the coupled
static state into one class so there's one lock, or move it into beans, or make the
constants genuinely independent. As a stopgap I'd force deterministic initialization
order, single-threaded, at startup before traffic arrives — and I'd label that a
workaround in the code, because it hides the cycle rather than removing it.

To confirm the fix, I'd reproduce the hang first. A concurrency fix I can't reproduce
the failure against is a hope, not a fix."

**What separates them:** knowing that `jstack`'s deadlock detector doesn't cover class
init locks — and treating that absence as evidence; naming the JLS rule; and especially
the single-threaded-versus-multi-threaded asymmetry, which explains the test/production
divergence and is the detail that shows real experience.

**Interviewer's follow-up:** *"How would you find these before they bite?"* — Grep for
static blocks that reference other classes with static state; that pattern is a small,
enumerable set in most codebases. And in review, treat any cross-class static dependency
as a defect on sight — the cost of removing it is always lower than the cost of the 3am
hang.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Delegation is parent-first, and a loader only defines a class its ancestors could not
   find. Derive from that one fact why you cannot replace a JDK class by putting your own
   version on the application classpath — and then say what mechanism *does* let you, and
   why it exists.

2. Initialization is lazy and failure is permanent. Construct a deployment scenario in
   which a service is 100% healthy according to every metric and probe you have, and is
   nonetheless permanently unable to serve one business-critical endpoint. Then design the
   cheapest check that would have caught it, and state that check's cost.

3. On one thread, circular initialization yields default values silently. On two threads,
   it deadlocks. Explain why the JVM specification makes those two different choices
   rather than one consistent one. What would break if recursive entry blocked instead?

4. A class is unloadable only when its defining loader is unreachable. Given that, explain
   why a `ThreadLocal` left set on a pooled thread can retain not just one object but an
   entire application's worth of classes and metaspace — and why the symptom is metaspace
   growth with a flat heap graph.

5. `javac` inlines compile-time constants into callers. Argue that this is a good design
   decision. Then argue it is a bad one. Then say what you would actually change about
   the language, if anything, and what would break.

6. `Class.forName(name)` initializes; `loader.loadClass(name)` does not. You are writing a
   plugin loader for `orderflow` that discovers payment gateway implementations at
   startup. Which do you call, at what moment, and what failure mode does each choice
   create?

7. A readiness probe that exercises every code path would catch every poisoned class. Give
   three independent reasons that is nonetheless the wrong probe to ship, and say what you
   would ship instead.

---

## Quick reference card

### JVM flags — class loading

| Flag | Default | What it does | Set it? |
|---|---|---|---|
| `-Xlog:class+load=info` | off | One line per class loaded, with its source | **Yes, during any investigation.** Cheap and decisive. |
| `-Xlog:class+load=debug` | off | Adds the defining loader per class | When you suspect a loader-identity problem |
| `-Xlog:class+unload=info` | off | One line per class unloaded | When investigating a suspected loader leak — *zero* lines is a finding |
| `-Xlog:class+path=info` | off | How the JVM resolved the classpath and module path | When "the jar is definitely there" and it definitely isn't |
| `-Xlog:class+resolve=debug` | off | Constant-pool resolution events | Rarely; very verbose. Useful when chasing a `NoSuchMethodError`. |
| `-Xlog:class+init=info` | off | Initialization events — **may be debug-build-only** | Check `java -Xlog:help` first |
| `-verbose:class` | off | Legacy alias for `-Xlog:class+load=info` | Prefer the `-Xlog` form; the decorators are worth it |
| `-XX:MaxMetaspaceSize` | unlimited by default | Caps class metadata | **Yes in containers** — an unbounded metaspace is how a loader leak becomes an OOM kill instead of an `OutOfMemoryError` you can dump |
| `-XX:ArchiveClassesAtExit=<file>` | off | Records a CDS archive from a real run | Yes, as a startup optimisation |
| `-XX:SharedArchiveFile=<file>` | JDK default archive | Uses a CDS archive | Yes, once you have one |
| `-Xshare:off` / `:auto` / `:on` | `auto` | Disable/allow/require CDS | `off` as an A/B control when measuring CDS benefit |
| `-XX:+TraceClassLoading` | — | **Removed.** Old flag from the pre-`-Xlog` era. | No — if a blog post says this, the post predates JDK 9 |
| `-javaagent:<jar>` | off | Installs a `ClassFileTransformer` | Topic 81. Note it can change class-loading behaviour and defeat CDS. |

> **Version note, one line:** class loading and initialization semantics are specified by
> the JLS and JVMS and have not changed between JDK 21 and 25; what *has* moved in that
> window is ahead-of-time class loading/linking (Project Leyden's AOT cache) and possibly
> the chaining of the original cause onto the follow-up `NoClassDefFoundError`. **Settle
> both on your runtime** with `java -XX:+PrintFlagsFinal -version | grep -i -E 'AOT|CDS|SharedArchive'`
> and by running the Failure drill and printing `getCause()` — rather than trusting any
> document, including this one.

### Diagnostic commands

```bash
# What classes are loaded, from where, and by whom?
java -Xlog:class+load=info:file=class.log:time,uptime -jar orderflow.jar
jcmd <pid> VM.classloaders
jcmd <pid> VM.classloader_stats
jcmd <pid> VM.class_hierarchy com.orderflow.orders.Order

# Metaspace, per loader. The leak detector.
jcmd <pid> VM.metaspace

# What does the JVM think its classpath is?
jcmd <pid> VM.system_properties | grep -i 'class.path\|module.path'

# Class-init deadlock: look for <clinit> frames, and note the ABSENCE of a
# "Found one Java-level deadlock" section.
jcmd <pid> Thread.print > threads.txt
grep -n '<clinit>' threads.txt

# Triage a log you were handed. Sort ASCENDING; do not aggregate by frequency.
grep -n -m1 'ExceptionInInitializerError' app.log      # the truth
grep -c    'NoClassDefFoundError'         app.log      # the echoes
grep -c    'ClassNotFoundException'       app.log      # a different bug entirely

# Is a constant inlined, or read at runtime?
javap -c -p target/classes/com/orderflow/orders/OrderValidator.class | grep -n 'getstatic\|sipush\|ldc'

# What -Xlog tags does THIS build actually support?
java -Xlog:help
jcmd <pid> help
```

### The error family — a decision table

| You see | It means | First command |
|---|---|---|
| `ClassNotFoundException: com.orderflow.orders.OrderService` (dotted) | A reflective/string lookup failed | Find the string. Check config, then the loader used. |
| `NoClassDefFoundError: com/orderflow/orders/OrderService` (slashed) | Present at compile time, absent at runtime | `mvn dependency:tree`; `unzip -l app.jar` |
| `NoClassDefFoundError: Could not initialize class com.orderflow.orders.OrderService` | The class is **poisoned** | `grep -n -m1 ExceptionInInitializerError app.log` |
| `ExceptionInInitializerError` | A static initializer threw — **read the `Caused by`** | Fix the cause; move it out of `<clinit>` |
| `ClassCastException: X cannot be cast to X` | Two loaders defined X | Print `getClassLoader()` on both sides |
| `IllegalAccessError` on package-private access | Same-named package, two loaders | Same as above |
| `NoSuchMethodError` / `NoSuchFieldError` | Compiled against a different version | Topic 32: dependency resolution |
| `VerifyError` | Malformed or rewritten bytecode | An agent (Topic 81) or a bytecode library |
| `UnsupportedClassVersionError` | Class file newer than the JVM | Check `maven.compiler.release` against the runtime JDK |
| Threads parked in `<clinit>`, no deadlock section | Class-init deadlock | Break the static cycle |

### Gotchas checklist

- [ ] "Could not initialize class" means look **earlier in the log**, not at the classpath.
- [ ] Sort logs **ascending by time** for this failure shape. Frequency aggregation lies.
- [ ] Nothing fallible goes in a `static` block. Not config, not I/O, not a health check.
- [ ] A transient failure inside `<clinit>` becomes a **permanent** one.
- [ ] `loadClass` does not initialize. `Class.forName(name)` does.
- [ ] Creating an array of a type does not initialize the type.
- [ ] `static final` compile-time constants are inlined — bumping the library changes nothing.
- [ ] Identical class names in a `ClassCastException` = loader problem. Read the parentheses.
- [ ] `jstack`'s deadlock detector does not find class-init deadlocks.
- [ ] Loader count growing under load = metaspace leak in progress (Topics 79, 80).
- [ ] A readiness probe that never touches a class cannot protect it.
- [ ] In a container, cap `-XX:MaxMetaspaceSize` so a leak gives you an error you can dump, not an OOM kill.

---

## When would I use this at work?

**1. A 3am page: one endpoint returning 500s, everything else green.**
You see thousands of `NoClassDefFoundError: Could not initialize class ...`. Instead of
rebuilding the image, you run `grep -n -m1 ExceptionInInitializerError` and read the
`Caused by` chain. It names a missing resource, a null environment variable, or a
downstream that was briefly unreachable. Two minutes instead of two hours — and you also
know that restarting the pod will "fix" it, which tells you whether the cause was
environmental or transient before you decide whether to roll back.

**2. Reviewing a pull request that adds a `static { }` block.**
You ask one question: can anything in this block fail? If yes, you explain that a
transient failure becomes a permanent, process-wide outage with a useless error message,
and you ask for it to move into a bean or a constructor. This is a thirty-second review
comment that prevents a specific, expensive, recurring incident shape — and it is the
single highest-value thing this document gives you.

**3. Cutting deploy-to-first-request latency, with numbers.**
Product wants faster scale-out for a bursty traffic pattern. You count classes loaded at
startup, record startup-to-readiness across three runs, add a CDS archive, and re-measure.
You can then say "class loading was N% of our cold start, CDS removed M% of it, and the
remaining time is Flyway plus connection-pool warm-up" — which is a capacity conversation
with evidence rather than a guess. Topic 74 gives you the JIT half of the same question,
and Topic 83 gives you the radical version of the answer.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the `Integer` cache**: `IntegerCache` is a lazily-initialized nested
  class whose static block reads a system property. The JDK does the exact thing this
  document warns you about; it gets away with it because the operation cannot realistically
  fail. Yours can.
- **17 — Immutability and `final`**: `static final` has two meanings — a compile-time
  constant that gets inlined, and a runtime constant assigned in `<clinit>`. Which one you
  wrote determines whether a library bump takes effect.
- **19 — Serialization**: `ObjectInputStream` resolves classes by **name** through a
  loader, which is why deserialization across deployment boundaries produces
  `ClassNotFoundException` even when the class obviously exists somewhere in the process.
- **20 — JPMS**: the module graph is consulted before the classpath, and module readability
  is an additional access check layered on top of loader delegation. `jlink` produces a
  runtime image with a fixed, known set of loadable classes.
- **31 — Maven fundamentals**: `provided` scope means "the runtime supplies it". Getting it
  wrong is a `NoClassDefFoundError` at deploy, not a compile error.
- **32 — Dependency resolution**: Maven's flat classpath keeps exactly one version, so a
  transitive bump silently changes what a class links against. `NoSuchMethodError` and
  `NoSuchFieldError` are the runtime shape of that.
- **39 — Injection styles**: constructor injection fails fast at context startup — the
  correct place for the failure that Trap 2 puts in a static block.
- **43 — Configuration and properties**: where configuration reads belong, with validation,
  so a bad value fails the context instead of poisoning a class.
- **66 — JVM architecture**: the runtime data areas. Class metadata lives in metaspace;
  static field *values* live on the heap in the `java.lang.Class` mirror. That split is
  why a loader leak shows up in both.

**This unlocks:**

- **68 — Heap generations, TLABs and metaspace**: metaspace is where the metadata this
  document creates actually lives, and it is sized and collected differently from the heap.
- **69 — Object layout**: the class word in every object header points at the `Klass`
  metadata that class loading produced.
- **70 — GC fundamentals**: loaded classes and their static fields are **GC roots**. A
  class held by a live loader roots everything its statics reference — the mechanism behind
  the unbounded-static-cache leak.
- **71 — G1 in depth**: the `Metadata GC Threshold` pause cause in a GC log is class
  loading forcing a collection. Different problem, different fix, same log file.
- **74 — JIT and tiered compilation**: the first request is slow for two independent
  reasons — class loading and verification, and cold compiled code. Separate them before
  optimising either.
- **76 — Reading bytecode**: `javap -c` is how you settle the inlined-constant question,
  and how you see `<clinit>` and `<init>` as the real methods they are.
- **79 — Heap dumps and MAT**: a classloader leak is found by walking the retaining path
  from a `ClassLoader` instance in the dominator tree.
- **80 — Off-heap memory**: metaspace is native memory, so unbounded class loading grows
  RSS without touching heap — the "healthy heap, OOM-killed container" shape.
- **81 — Instrumentation agents**: an agent's `ClassFileTransformer` runs *between* loading
  and defining. It can rewrite your classes, change their size, defeat CDS, and introduce
  its own loaders.
- **82 — Containers and cgroups**: cap metaspace explicitly, or a loader leak becomes an
  OOM kill with no heap dump instead of an error you can diagnose.
- **83 — GraalVM native image**: closed-world analysis means class loading happens at build
  time. Everything dynamic in this document — `Class.forName`, custom loaders, reflection —
  must be declared up front or it fails at runtime. This document is a catalogue of what
  native-image gives up.
- **101 — Virtual threads**: a slow or blocked `<clinit>` on a virtual thread holds the
  initialization lock, and every other thread that needs that class waits — an
  under-appreciated way to serialise a supposedly-concurrent workload.

---

*Java baseline 21, running on JDK 25. Two things in this document are deliberately hedged
rather than asserted: whether your JDK chains the original cause onto the follow-up
`NoClassDefFoundError`, and the exact JDK in which each piece of Project Leyden's AOT
class loading landed. Both have a command in the Hands-on section that settles them on
your machine in under a minute. Nothing in this document is captured tool output; the two
stack-trace shapes are labelled illustrations with placeholder content. What has been
stable since Java 1.2 and will still be true at 3am: delegation is parent-first, identity
is (name, loader), initialization is lazy and happens once, and a failed initialization is
permanent — which is why the first error in the log is the only one worth reading.*
