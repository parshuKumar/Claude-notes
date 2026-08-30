# 81 — `java.lang.instrument` Agents: How APM and Spring Actually Work

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: explains how an APM or OpenTelemetry agent adds spans to `orderflow` code you never touched — and measures what that costs in startup time, in metaspace, and in the JIT's inlining decisions against the recorded Topic 65 baseline.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> A `-javaagent` registers a **`ClassFileTransformer`**. The JVM invokes that
> transformer with the **raw bytes of every class, before the class is defined**, and
> the transformer may **return different bytes**. The JVM then defines the class from
> whatever came back.
>
> That is how an APM adds spans to code you never touched. There is no annotation, no
> interface, no wrapper object, and no cooperation from your source. The bytes that run
> are not the bytes `javac` produced.
>
> And because rewriting a method **makes its bytecode bigger**, the same mechanism can
> push a method past the JIT's inlining size threshold. A method that was inlined into
> its caller yesterday may not be inlined today. **The observer changed the system it
> was observing**, and the only thing that changed in your deploy was a flag.

Two consequences follow immediately, and they are the whole document.

1. **Instrumentation is not a wrapper.** Unlike a Spring proxy (Topic 40), which sits
   *outside* your object and is bypassed by `this.method()`, an agent rewrites the
   method *body*. Self-invocation is instrumented. Private methods are instrumented.
   `final` classes are instrumented. Third-party library internals are instrumented.
2. **Instrumentation is not free and not neutral.** It costs startup time (a
   transformer runs per class, and a Spring Boot app loads thousands), it costs
   metaspace and code cache (Topic 80's RSS equation), and it perturbs the compiler
   (Topics 74 and 75).

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE — and this one is important to say out loud

There is no general equivalent of `java.lang.instrument` in Node. Not a weaker one, not
an awkward one — the capability does not exist.

**What you cannot do in Node:**

- You cannot hand V8 the source or bytecode of a function that is about to be compiled
  and give back different source or bytecode.
- You cannot rewrite the body of a function that has already been compiled and is
  currently on a call stack.
- You cannot instrument a function that was never exported, was captured in a closure,
  or was called internally by the module that defined it.

**What people offer as the analogue, and exactly how far it goes:**

```ts
// The Node OTel auto-instrumentation approach, simplified.
// require-in-the-middle / import-in-the-middle: hook the MODULE BOUNDARY.
const hook = require('require-in-the-middle');

hook(['pg'], (exports, name) => {
  const originalQuery = exports.Client.prototype.query;
  exports.Client.prototype.query = function (...args) {
    const span = tracer.startSpan('pg.query');
    try {
      return originalQuery.apply(this, args);
    } finally {
      span.end();
    }
  };
  return exports;
});
```

This is **monkey-patching at a module boundary**. It is genuinely useful, and it is how
OpenTelemetry's Node auto-instrumentation works. It is also **partial at best**, in four
specific ways:

| What you want to instrument | Java agent | Node module hook |
|---|---|---|
| A public method reached through an exported object | yes | yes |
| A method called by its own class internally (`this.helper()`) | **yes** — the body is rewritten | **no** — the internal call site was resolved before you patched, or never went through the export |
| A private/non-exported function | **yes** | **no** — you cannot reach it |
| A function already compiled and running | **yes** — `retransformClasses` | **no** |
| A `final` class or a `static` method | **yes** | n/a — but the closure/const-binding equivalent blocks you |
| Code inlined by the compiler at build time (esbuild, Rollup) | **yes** — the JVM inlines at *runtime*, after your transform | **no** — the boundary you wanted was erased at build |

That last row is worth pausing on. In Node, bundling **destroys** the module boundaries
your instrumentation depends on. Once esbuild has inlined `pg` into your bundle, the
`require('pg')` interception point does not exist any more. This is a known, real,
frequently-hit limitation of Node auto-instrumentation. In Java, the JVM's inlining
happens *after* your transformer has already run, so the agent always gets first
refusal on the bytes.

**Verdict: NO ANALOGUE in the general case.** Module hooks are a partial, boundary-only
approximation. Say exactly that in an interview — "Node's auto-instrumentation patches
exports; a Java agent rewrites bytecode before class definition" — and you have
demonstrated that you understand both.

### The one honest half-analogue: a build-time transform

The closest true analogue is not a runtime hook at all. It is a **compiler plugin**: a
Babel plugin, a TypeScript transformer, an SWC plugin. Those genuinely rewrite code
before it runs.

**Where it still differs:** a Babel plugin runs at build time, over source you have. An
agent runs at **class-load time**, over bytes it has never seen, including bytes from
jars you do not own and did not compile. You can instrument Hibernate's internals
without a copy of Hibernate's source, and without rebuilding anything.

### The Java-side thing this is NOT: Spring's proxies

You already met the other mechanism, in Topic 40. Keep them apart — the difference is
the single most useful thing in this document.

| | Spring proxy (Topic 40) | Instrumentation agent (this topic) |
|---|---|---|
| What exists at runtime | a **different object** wrapping yours | **your class**, with rewritten method bodies |
| When it is created | bean post-processing, after the class is loaded | during class **definition**, before any instance exists |
| Reaches self-invocation (`this.m()`) | **no** — the trap you drilled in Topic 40 | **yes** |
| Reaches `private` / `final` / `static` methods | **no** | **yes** |
| Reaches third-party library code | only if you wrapped it as a bean | **yes** |
| Reaches `java.*` classes | no | yes (with care; see boot classpath below) |
| Requires the JVM to be started with a flag | no | **yes** — `-javaagent` |
| Visible in `getClass().getName()` | **yes** — `$$SpringCGLIB$$` / `$Proxy12` | **no** — the class name is unchanged, which is why this is hard to notice |

That last row is why the observer effect in this document is so easy to miss. A proxy
announces itself in the class name. An agent leaves no trace in any name you can print.

---

## What is this?

`java.lang.instrument` is a small package, in the `java.instrument` module, with three
things that matter:

1. **`ClassFileTransformer`** — a single-method interface that receives class bytes and
   returns class bytes.
2. **`Instrumentation`** — the handle the JVM gives your agent, letting you register
   transformers, force re-transformation of already-loaded classes, and ask a few
   questions the JVM otherwise will not answer.
3. **A manifest contract** — a jar becomes an agent by declaring `Premain-Class` (or
   `Agent-Class`) in its `META-INF/MANIFEST.MF`.

### The interface, in full

```java
package java.lang.instrument;

public interface ClassFileTransformer {

    default byte[] transform(Module module,
                             ClassLoader loader,
                             String className,             // internal form: com/orderflow/orders/OrderService
                             Class<?> classBeingRedefined, // null on first load
                             ProtectionDomain protectionDomain,
                             byte[] classfileBuffer) throws IllegalClassFormatException {
        return null;   // null means "I did not change anything"
    }
}
```

Four details in that signature carry real weight:

- **`className` is in internal form** — slashes, not dots, and `null` for a lambda
  proxy class or other synthetically-defined class with no name yet. Matching on
  `com.orderflow.orders.OrderService` will silently never match anything. This is the
  single most common first-agent bug.
- **`classBeingRedefined` is `null` on first load** and non-null on a retransform. It is
  how a transformer knows which of the two situations it is in.
- **`classfileBuffer` is the bytes as they stand right now** — which may already have
  been rewritten by an earlier transformer. You are a link in a chain, not the only
  actor.
- **Returning `null` means "no change"**, and it is not the same as returning the
  input array. Returning `null` is the fast path; the JVM skips the redefinition
  machinery entirely.

### The two entry points

**`premain` — the one you should use.**

```java
public static void premain(String agentArgs, Instrumentation inst) { }
```

The JVM calls this after the VM is initialised but **before the application's `main`
method is invoked**, and before your application classes are loaded. Manifest:

```
Premain-Class: com.orderflow.agent.TimingAgent
Can-Retransform-Classes: true
```

Launched with:

```
java -javaagent:/opt/agents/timing-agent.jar=arg1,arg2 -jar orderflow.jar
```

**`agentmain` — dynamic attach, after the JVM is already running.**

```java
public static void agentmain(String agentArgs, Instrumentation inst) { }
```

Attached from a second process via `com.sun.tools.attach.VirtualMachine.attach(pid)` and
`loadAgent(path)`. Manifest:

```
Agent-Class: com.orderflow.agent.TimingAgent
Can-Retransform-Classes: true
```

`[JAVA 21+]` **JEP 451 changed the default posture on dynamic attach.** From JDK 21, the
JVM prints a warning when an agent is loaded dynamically into a running VM, and the
stated direction is that this will be disallowed by default in a future release. You opt
in explicitly with `-XX:+EnableDynamicAgentLoading`.

> **Honest uncertainty, one line:** I am confident of the warning and of the flag name
> on 21+, and confident that JEP 451 is "prepare to disallow" rather than "disallow"; I
> am **not** certain which release, if any, has flipped the default by the JDK build in
> front of you. **Settle it:** run `java -XX:+PrintFlagsFinal -version | grep -i
> EnableDynamicAgentLoading` and read the default and origin, and try one dynamic attach
> on your actual JDK and read the warning text.

Why this matters operationally: **`-javaagent` at launch and dynamic attach are not
equivalent**, and the difference is not just a warning. See the retransformation limits
below.

### A third entry point you will meet in tooling

`Launcher-Agent-Class` (JDK 9+) lets an *executable jar* carry its own agent, invoked
before the jar's main class:

```
Launcher-Agent-Class: com.orderflow.agent.TimingAgent
```

Its method is `agentmain(String, Instrumentation)`. This is how some self-instrumenting
tools ship as a single runnable jar.

### What actually uses this in your stack

| Tool | Uses `java.lang.instrument`? | Notes |
|---|---|---|
| OpenTelemetry Java agent | **yes** | ByteBuddy-based; instruments hundreds of libraries. Topic 119. |
| Datadog / New Relic / Dynatrace / Elastic APM | **yes** | Same shape; ByteBuddy or ASM underneath. |
| JaCoCo (coverage) | **yes** | Inserts probes into every branch. This is why coverage runs are slower. |
| Mockito inline mock maker | **yes** | Uses an agent to instrument `final` classes and `static` methods. |
| JOL (`getObjectSize`) | **optionally** | `Instrumentation.getObjectSize` is the authoritative size (Topic 69). |
| Spring Framework, ordinary use | **no** | Proxies, not agents. Topic 40. |
| Spring load-time weaving | **yes, opt-in** | `spring-instrument.jar` + `@EnableLoadTimeWeaving`. |
| AspectJ LTW | **yes** | `aspectjweaver.jar` as a `-javaagent`, driven by `META-INF/aop.xml`. |
| Hibernate bytecode enhancement | **usually not** | Normally a **build-time** Maven/Gradle plugin. A runtime agent is possible but rare. |
| JFR | **partly** | The JDK instruments `jdk.jfr.Event` subclasses at load time to generate their commit code. |

**Read the Spring row twice.** The title of this topic says "how APM *and Spring* actually
work", and the honest answer is that they work by **two different mechanisms**. Spring's
default is proxies. Agents are the opt-in escape hatch Spring offers for the cases proxies
cannot reach — which is exactly the set of cases in the comparison table above.

---

## Why does it matter?

**1. It is the answer to "how does the APM know about my code?"**

If you cannot answer that, every observability conversation you have is faith-based. The
answer is one sentence: *it rewrites the bytecode of the libraries it recognises, at class
load, before the class is defined.*

**2. It is a real, measurable observer effect, and it is blamed on the wrong thing.**

A release goes out. p99 on `POST /orders` is worse. The diff contains no code changes to
the order path. What changed was a Helm value adding `-javaagent`. Nobody looks at the
Helm value, because "adding tracing" is not a code change. This trap has cost real teams
real weeks, and it is Trap 1 below.

**3. It is a term in the container memory budget you built in Topic 80.**

An agent adds classes (metaspace), generated classes (metaspace), and more compiled code
(code cache). Both are native, both are outside the heap, and both are in the RSS equation
that gets your pod OOMKilled. "We added tracing and now the pod OOMKills" is a coherent
sentence.

**4. It explains a category of bug that is otherwise inexplicable.**

"It works in my IDE but not in the container." "It works with the profiler attached but
not without." "The stack trace has frames from a class that is not in our repo." All three
are agent behaviour, and all three are unfalsifiable if you do not know agents exist.

**5. It is the mechanism behind tools you already rely on.**

Mockito's ability to mock a `final` class. JaCoCo's branch coverage. Your APM. If you want
to reason about *why* your test suite is slow, or why coverage numbers change under a
profiler, this is the machinery.

---

## Machine-level reality

Everything here is checkable. Where I am not certain, I say so in one line and give the
command.

### The lifecycle, in exact order

This is the sequence for `-javaagent`, and the ordering is what makes agents powerful and
awkward at the same time.

```
1.  JVM process starts. Native VM initialisation.
2.  Bootstrap / platform class loaders come up. Core java.* classes load.
3.  For each -javaagent, IN COMMAND-LINE ORDER:
      a. the agent jar is added to the SYSTEM class path
      b. the Premain-Class is loaded BY THE SYSTEM CLASS LOADER
      c. premain(String, Instrumentation) is invoked, on the main thread
      d. whatever transformers it registered are now live
    -- your premain runs to completion before the next agent's premain --
4.  The application's main class is loaded.
      -> every transformer registered in step 3 sees these bytes
5.  main(String[]) is invoked.
6.  Every subsequent class definition, for the life of the JVM, passes through
    every registered transformer.
```

Four things fall out of that ordering, all of which bite people:

- **Your agent's classes are loaded before your application's.** An agent cannot
  reference application classes at `premain` time — they do not exist yet, and touching
  them forces them to load *before* your transformers are ready, so they load
  un-instrumented. This is why real agents are shy about what they touch early.
- **Everything in `premain` is on the startup critical path.** It is synchronous, on the
  main thread, before `main`. A slow `premain` is pure added startup latency, and it is
  invisible to Spring's own "Started in N seconds" number. That gap is how you measure it
  (see Measurement).
- **Agent order is command-line order.** Two `-javaagent` flags means agent 1's
  transformers are registered first, and see the original bytes; agent 2's transformers
  see agent 1's output. Reorder the flags and you get different bytes.
- **Classes loaded in step 2 have already been defined.** If you want to instrument
  `java.util.HashMap`, `premain` is too late for the plain path — you must
  `retransformClasses` it. Which brings us to the limits.

### The `Instrumentation` handle: what you can and cannot do

```java
public interface Instrumentation {
    void addTransformer(ClassFileTransformer t, boolean canRetransform);
    boolean removeTransformer(ClassFileTransformer t);

    void retransformClasses(Class<?>... classes);      // re-run transformers on loaded classes
    void redefineClasses(ClassDefinition... defs);     // supply replacement bytes directly

    boolean isRetransformClassesSupported();
    boolean isRedefineClassesSupported();
    boolean isModifiableClass(Class<?> theClass);

    Class<?>[] getAllLoadedClasses();
    Class<?>[] getInitiatedClasses(ClassLoader loader);

    long getObjectSize(Object objectToSize);           // the JOL trick, Topic 69

    void appendToBootstrapClassLoaderSearch(JarFile jarfile);
    void appendToSystemClassLoaderSearch(JarFile jarfile);

    boolean isNativeMethodPrefixSupported();
    void setNativeMethodPrefix(ClassFileTransformer t, String prefix);
}
```

**The hard limits on `retransformClasses` and `redefineClasses` are the most important
facts in this section.** You may change:

- method **bodies**
- the **constant pool**
- class **attributes**

You may **not**:

- add or remove **methods**
- add or remove **fields**
- change method **signatures** or modifiers
- change the class **hierarchy** (superclass, implemented interfaces)

Attempting any of those throws `UnsupportedOperationException`.

**Why you must care:** many instrumentation strategies want to add a field — a span
handle, a start timestamp, a context object attached to a request. On the `premain` path
that is legal, because the class has not been defined yet and the transformer can emit
whatever class file it likes. On the **retransform** path it is illegal.

So: **an agent attached dynamically to a running JVM can be strictly less capable than the
same agent passed with `-javaagent` at launch.** ByteBuddy and the big APM agents work
around this with side tables (a `WeakConcurrentMap` keyed by the instrumented object) at a
cost in lookup time and in memory. If you have ever read "for full instrumentation, attach
the agent at startup" in an APM's docs, that sentence is this paragraph.

### Transformer ordering, and why it is not simply "registration order"

Transformers are invoked in registration order. The subtlety is that there are **two
registration lists**: transformers registered with `canRetransform=false` and those
registered with `canRetransform=true`. The documented behaviour is that the
retransformation-incapable transformers run first (in their registration order), then the
retransformation-capable ones (in theirs) — and on a **retransformation**, only the capable
ones run, starting again from the original class file bytes rather than from a previously
transformed state.

> **Honest uncertainty, one line:** I am confident about the two-list split and about
> retransformation restarting from the original bytes; I would not stake a production
> decision on my recollection of edge cases in the ordering. **Settle it in five minutes:**
> register two transformers that each log `className` and `classfileBuffer.length`, one
> capable and one not, register them in both orders, and read your own log. The Hands-on
> section has the harness.

The practical rule that survives regardless: **when two agents are attached, the second
one on the command line sees bytes the first one already rewrote.** That is why "reordering
our `-javaagent` flags fixed it" is a real sentence that real people say.

### What a transformer actually does to a method

You almost never write raw bytes. Two libraries own this space:

**ASM** — a low-level visitor API over the class file format. You implement
`ClassVisitor`/`MethodVisitor` and emit opcodes yourself. Maximum control, maximum
opportunity to produce a class file the verifier rejects.

**ByteBuddy** — a high-level DSL that generates the bytecode for you, and what most modern
agents (including the OpenTelemetry Java agent) actually use. Two modes matter:

- **`Advice`** — your advice code is **inlined into the target method** as bytecode. There
  is no extra call, no extra object. This is what agents use on hot paths.
- **`MethodDelegation`** — the target method's body is replaced with a call out to your
  handler. Simpler and more flexible; more overhead, and an extra frame in every stack
  trace.

Here is the shape of what `Advice` does, expressed as equivalent Java. **This is the
mechanism to have in your head for the rest of the document.**

```java
// What you wrote:
public Reservation reserve(long productId, int quantity) {
    Inventory inv = inventoryRepository.findForUpdate(productId);
    inv.decrement(quantity);
    return new Reservation(productId, quantity);
}

// What the agent's Advice makes the JVM define (conceptually — it is bytecode,
// not source, and there is no separate method):
public Reservation reserve(long productId, int quantity) {
    // --- injected @Advice.OnMethodEnter ---
    Span span = tracer.spanBuilder("InventoryService.reserve").startSpan();
    Scope scope = span.makeCurrent();
    long t0 = System.nanoTime();
    Throwable thrown = null;
    // --- end injected ---
    try {
        Inventory inv = inventoryRepository.findForUpdate(productId);
        inv.decrement(quantity);
        return new Reservation(productId, quantity);
    } catch (Throwable t) {
        thrown = t;                       // injected
        throw t;                          // injected
    } finally {
        // --- injected @Advice.OnMethodExit ---
        if (thrown != null) span.recordException(thrown);
        span.setAttribute("duration_ns", System.nanoTime() - t0);
        scope.close();
        span.end();
        // --- end injected ---
    }
}
```

**Now count what changed about this method as an object of the JIT's attention:**

1. Its **bytecode size** grew — by the advice body, plus the try/catch/finally scaffolding.
2. It gained an **exception handler table entry**, where it may have had none.
3. It gained **calls** to `tracer`, `span`, `scope` — call sites that must themselves be
   inlined, or the whole thing gets expensive.
4. It gained **new local variables**, so its stack frame is larger.
5. Objects that did not exist before (`Span`, `Scope`) now exist, and they **escape** —
   they are handed to a tracer, stored in a thread-local context. Topic 75's escape
   analysis has strictly less to work with.

Every one of those is a JIT input. Point 1 is the one that produces Trap 1.

### The inlining connection — this is the payload

From Topic 75 you know: **escape analysis, scalar replacement and lock elision all depend
on inlining happening first.** And inlining stops at size limits.

HotSpot's two size thresholds:

| Flag | What it gates | Commonly-cited default |
|---|---|---|
| `-XX:MaxInlineSize` | max bytecode size of a **cold** callee to inline it | 35 bytes |
| `-XX:FreqInlineSize` | max bytecode size of a **hot** callee to inline it | 325 bytes |
| `-XX:MaxInlineLevel` | max depth of the inlining tree | 15 on recent JDKs (was 9 historically) |
| `-XX:InlineSmallCode` | do not inline into a callee whose *compiled* code already exceeds this | platform-dependent |

> **Honest uncertainty, one line:** 35 and 325 are the long-standing HotSpot defaults and
> I am reasonably confident they still hold, but defaults do move between releases and
> platforms and I will not have you tune from my memory. **Settle it:**
> `java -XX:+PrintFlagsFinal -version | grep -iE "MaxInlineSize|FreqInlineSize|MaxInlineLevel|InlineSmallCode"`
> and, on the live process, `jcmd <pid> VM.flags -all | grep -i inline`.

Here is the mechanism in one paragraph, which is what you say in an interview:

> `InventoryService.reserve` was a small method — comfortably under `FreqInlineSize`. C2
> inlined it into `OrderService.placeOrder`, and having inlined it, could see that the
> `Reservation` object never escaped and scalar-replace it. The agent's advice added
> bytecode to `reserve`. It is now over the threshold. C2 stops inlining it. That single
> change removes the inlining, which removes the escape analysis, which reintroduces the
> allocation, which raises the allocation rate, which increases young-collection frequency
> (Topic 68), which shows up as a p99 regression on a release that contained no code
> changes.

That is a **chain of five mechanisms**, all of which you have already studied, triggered by
adding a flag. It is not a large effect on every method — most methods are nowhere near a
threshold. It is a large effect on the specific method that happened to be sitting just
under one, and on a hot path that is exactly where it hurts.

**This is not a reason not to use an APM.** It is a reason to measure the agent as a change,
because it *is* a change.

### The other costs, named and attributable

| Cost | Where it lands | How you see it |
|---|---|---|
| `premain` execution | startup, before `main` | the gap between Spring's "Started in N" and "process running for M" |
| Per-class transformer evaluation | class-loading time, every class | `-Xlog:class+load` count × per-class matcher cost |
| Agent's own classes | **metaspace** (native, Topic 80) | NMT `Class` category delta |
| ByteBuddy-generated classes | **metaspace** | NMT `Class`, and `jcmd <pid> VM.classloader_stats` |
| Larger method bodies → more compiled code | **code cache** (native) | NMT `Code` category delta |
| More compilation work | CPU during warm-up | `-XX:+PrintCompilation` volume; `Compiler` in NMT |
| Lost inlining / lost scalar replacement | steady-state throughput and allocation rate | `-XX:+PrintInlining` diff; allocation profile |
| Span objects and context maps | heap and allocation rate | async-profiler alloc mode (Topic 78) |
| CDS/AppCDS effectiveness | startup | a transformed class cannot be used from the shared archive |

That last row deserves a hedge. **A class whose bytes an agent modified cannot be served
from a class-data-sharing archive** — the archive holds a pre-parsed form of the *original*
bytes, and if the transformer changes them, the JVM must fall back to the normal path for
that class. I am confident of the principle. I am **not** confident of how much of your
archive a given agent invalidates, because that depends entirely on how broadly the agent's
matchers fire.

> **Settle it:** build the AppCDS archive from Topic 122, then run with and without the
> agent and compare `-Xlog:class+load` lines marked `shared objects file` against those
> loaded from the jar. The ratio is your answer, on your app, with your agent.

This is the direct collision between Topic 122's cheap startup win and this topic's cost,
and it is worth knowing before someone tells you "AppCDS didn't help".

### Two things that silently swallow your agent

**1. A transformer that throws is ignored.** If `transform` throws, the JVM discards the
exception and defines the class from the **untransformed** bytes. Your agent does nothing,
reports nothing, and logs nothing. The class loads fine. This is by specification, and it
is the reason "the agent isn't instrumenting anything" is such a common and such an
infuriating bug.

**Always wrap your transformer body in `try { ... } catch (Throwable t) { log it; return
null; }`.** Not for correctness — for visibility.

**2. A transformer that triggers class loading can deadlock or recurse.** If your
transformer calls into a class that is not yet loaded, loading that class re-enters the
transformer chain. Real agents keep a thread-local re-entrancy guard and pre-load
everything they need in `premain`.

### Instrumenting `java.*` classes

To instrument bootstrap classes (`java.util.concurrent.ThreadPoolExecutor`, say), the
helper classes your injected code calls **must also be visible to the bootstrap class
loader** — otherwise the injected call site references a class the bootstrap loader cannot
resolve, and you get `NoClassDefFoundError` from inside the JDK.

That is what these two are for:

```
Boot-Class-Path: agent-bootstrap.jar          # manifest attribute
```
```java
inst.appendToBootstrapClassLoaderSearch(new JarFile("agent-bootstrap.jar"));
```

It is also why real agents ship a small "bootstrap" jar of shared types separate from the
main agent jar. If you have ever unzipped an APM agent jar and wondered what the nested
jars were for, that is the answer.

### `-XX:+TraceClassLoading`-adjacent: how to see it happening

The single most convincing five seconds in this topic:

```bash
java -Xlog:class+load=info -javaagent:/opt/agents/otel.jar -jar orderflow.jar \
  | grep -c "source:"
```

Run it with and without the agent. The **class count goes up**, because the agent has
loaded its own classes and generated new ones. That number is your metaspace story.

---

## Example 1 — minimal

Forty lines that demonstrate the mechanical statement with **no third-party dependency at
all**. This agent changes nothing; it just proves it is called for every class, before the
class is defined, with the raw bytes.

**`ClassLoadObserverAgent.java`:**

```java
package com.orderflow.agent;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.concurrent.atomic.AtomicLong;

public final class ClassLoadObserverAgent {

    private static final AtomicLong COUNT = new AtomicLong();
    private static final AtomicLong BYTES = new AtomicLong();

    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("[agent] premain running. args=" + agentArgs);
        System.out.println("[agent] retransform supported = " + inst.isRetransformClassesSupported());
        System.out.println("[agent] classes already loaded  = " + inst.getAllLoadedClasses().length);

        final String prefix = (agentArgs == null || agentArgs.isBlank())
                ? "com/orderflow/" : agentArgs.replace('.', '/');

        inst.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(Module module,
                                    ClassLoader loader,
                                    String className,
                                    Class<?> classBeingRedefined,
                                    ProtectionDomain pd,
                                    byte[] classfileBuffer) {
                try {
                    // className is INTERNAL FORM: slashes, not dots. And it can be null.
                    if (className == null || !className.startsWith(prefix)) {
                        return null;                 // null == "no change", the fast path
                    }
                    long n = COUNT.incrementAndGet();
                    long b = BYTES.addAndGet(classfileBuffer.length);
                    System.out.printf("[agent] #%d %s  bytes=%d  redefining=%s  loader=%s  running_total=%d%n",
                            n, className, classfileBuffer.length,
                            classBeingRedefined != null,
                            loader == null ? "bootstrap" : loader.getClass().getSimpleName(),
                            b);
                    return null;                     // we observe, we do not rewrite
                } catch (Throwable t) {
                    // MANDATORY. A thrown exception here is SILENTLY swallowed by the JVM.
                    System.err.println("[agent] transformer failed for " + className);
                    t.printStackTrace();
                    return null;
                }
            }
        }, true);   // true == canRetransform

        Runtime.getRuntime().addShutdownHook(new Thread(() ->
                System.out.printf("[agent] saw %d classes, %d bytes of class file%n",
                        COUNT.get(), BYTES.get())));
    }
}
```

**Build it. The manifest is the whole trick:**

```bash
mkdir -p ~/java-lab/81/agent/src/com/orderflow/agent ~/java-lab/81/agent/META-INF
cd ~/java-lab/81/agent

cat > META-INF/MANIFEST.MF <<'EOF'
Manifest-Version: 1.0
Premain-Class: com.orderflow.agent.ClassLoadObserverAgent
Can-Retransform-Classes: true
Can-Redefine-Classes: true
EOF

javac -d out src/com/orderflow/agent/ClassLoadObserverAgent.java
jar cfm observer-agent.jar META-INF/MANIFEST.MF -C out .
```

**A target that is deliberately trivial:**

```java
package com.orderflow.demo;

public class Target {
    public static void main(String[] args) {
        System.out.println("[app] main entered");
        System.out.println("[app] 3 + 4 = " + new Adder().add(3, 4));
    }
}
```
```java
package com.orderflow.demo;

public class Adder {
    public int add(int a, int b) { return a + b; }
}
```

**Run it:**

```bash
java -javaagent:observer-agent.jar=com/orderflow/ -cp out-demo com.orderflow.demo.Target
```

| What you see | What it means |
|---|---|
| All `[agent] premain running` output appears **before** `[app] main entered` | The lifecycle, proven. `premain` completes before `main` is invoked. |
| `[agent] #1 com/orderflow/demo/Target` appears **before** `[app] main entered` | The transformer ran on the main class **before it was defined**. This is the mechanical statement as an observation. |
| `[agent] #2 com/orderflow/demo/Adder` appears **after** the first `println` | Class loading is **lazy** (Topic 67). `Adder` was not loaded until first use. |
| `classes already loaded` is a substantial number | Those are the JDK's own classes, already defined before your agent existed. **You cannot transform them on the normal path** — only via `retransformClasses`. |
| `loader=AppClassLoader` for your classes | Delegation, from Topic 67. Change the prefix to `java/util/` and re-run: you see **nothing**, because those were loaded in step 2 of the lifecycle. |
| Nothing at all is printed by the agent | Your prefix has dots in it. `className` is internal form. This is the classic first-agent bug. |

**Now make it a real transformer.** Change the return from `null` to `classfileBuffer`
(the same bytes) and re-run. Behaviour is identical, but the JVM now goes through the full
redefinition path for every matched class instead of the fast path. **That difference —
`null` versus "the same array" — is a real cost on a codebase with thousands of classes**,
and it is the first thing to check when someone's homegrown agent is slow.

### Step two: actually rewrite something, with ByteBuddy

```xml
<dependency>
  <groupId>net.bytebuddy</groupId>
  <artifactId>byte-buddy</artifactId>
</dependency>
<dependency>
  <groupId>net.bytebuddy</groupId>
  <artifactId>byte-buddy-agent</artifactId>
</dependency>
```

```java
package com.orderflow.agent;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.asm.Advice;
import net.bytebuddy.matcher.ElementMatchers;

import java.lang.instrument.Instrumentation;

public final class TimingAgent {

    public static void premain(String args, Instrumentation inst) {
        new AgentBuilder.Default()
            .type(ElementMatchers.nameStartsWith("com.orderflow.inventory"))
            .transform((builder, type, loader, module, pd) ->
                builder.visit(Advice.to(TimingAdvice.class)
                        .on(ElementMatchers.isMethod()
                            .and(ElementMatchers.isPublic()))))
            .installOn(inst);
    }

    public static final class TimingAdvice {

        @Advice.OnMethodEnter
        static long enter() {
            return System.nanoTime();
        }

        @Advice.OnMethodExit(onThrowable = Throwable.class)
        static void exit(@Advice.Origin String method,
                         @Advice.Enter long start,
                         @Advice.Thrown Throwable thrown) {
            long elapsedNanos = System.nanoTime() - start;
            System.out.printf("[timing] %s took %d ns%s%n",
                    method, elapsedNanos, thrown == null ? "" : " (threw)");
        }
    }
}
```

Manifest gains one line, because ByteBuddy needs to be on the agent's classpath:

```
Premain-Class: com.orderflow.agent.TimingAgent
Can-Retransform-Classes: true
```

and the agent must be built as a shaded/uber jar containing ByteBuddy, or ByteBuddy must be
on the boot classpath. **Shading is the norm for exactly this reason** — an agent that puts
its own dependencies on the application classpath will collide with the application's
versions (Topic 32's nearest-wins problem, at runtime, with no build to warn you). Every
production APM agent ships fully shaded.

**What to look for:** `[timing]` lines for methods you never annotated, in a class you did
not modify, including methods called via `this.` from inside the same class.

**That last clause is the payoff.** Take Topic 40's self-invocation drill — the
`@Transactional` method called from a sibling method that silently did nothing. Instrument
the same class with this agent. **The timing advice fires on the self-invoked call.** Same
codebase, same call, and the two mechanisms disagree, because one wraps the object and the
other rewrites the method.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the recorded Topic 65 baseline:

- **Container:** `--memory=2g`, `--cpus=2`; Kubernetes `limits.memory: 2Gi`, `limits.cpu: 2`.
- **Dataset:** 100k products, 1M orders, 5M order lines, with a few hot products.
- **Load:** the recorded k6 mix — 70% catalogue read, 20% order read, 10% order placement —
  open-model arrival rate.
- **Recorded baseline:** p50/p95/p99/p999 and throughput per endpoint, committed to
  `/docs/java/baselines/`, re-runnable within ±10%.
- **JVM flags at baseline:** `-Xmx1g`, G1, `-XX:MaxDirectMemorySize` and
  `-XX:MaxMetaspaceSize` set from Topic 80's work, NMT summary on.

### The change that ships

The platform team is rolling out distributed tracing. The change to `orderflow` is a Helm
values edit, reviewed by nobody on the `orderflow` team because it touches no application
code:

```yaml
# values.yaml
env:
  - name: JAVA_TOOL_OPTIONS
    value: >-
      -javaagent:/otel/opentelemetry-javaagent.jar
      -Dotel.service.name=orderflow
      -Dotel.traces.exporter=otlp
      -Dotel.exporter.otlp.endpoint=http://collector:4317
```

That is the entire diff. No Java files changed. No dependency changed. The application jar
is byte-for-byte the one that produced the recorded baseline.

### What the agent does to the order path

`orderflow`'s hot path for `POST /orders` runs through, roughly:

```java
package com.orderflow.orders;

@Service
public class OrderService {

    private final InventoryService inventory;
    private final WalletService wallets;
    private final PaymentGateway payments;
    private final OrderRepository orders;

    @Transactional
    public OrderResult placeOrder(PlaceOrderCommand cmd) {
        Reservation reservation = inventory.reserve(cmd.productId(), cmd.quantity());
        Money total = priceOf(cmd);                        // small, private, hot
        WalletDebit debit = wallets.debit(cmd.userId(), total);
        Payment payment = payments.authorize(cmd.userId(), total);
        Order order = orders.save(Order.from(cmd, reservation, debit, payment));
        return OrderResult.accepted(order.getId());
    }

    private Money priceOf(PlaceOrderCommand cmd) {         // ~20 bytes of bytecode
        return catalogue.priceOf(cmd.productId()).multipliedBy(cmd.quantity());
    }
}
```

The OTel agent's matchers fire on: the Spring MVC entry point, `@Transactional` boundaries,
the JDBC driver's `PreparedStatement.execute*`, HikariCP's `getConnection`, the HTTP client
used by `PaymentGateway`, and — depending on version and configuration — Spring `@Service`
methods.

Every one of those methods gains: a span start, a scope open, a try/catch/finally, a scope
close, a span end. Every one grows in bytecode size. Every one gains an exception handler.
And span/scope objects are created per call and handed to a thread-local context — which
means they **escape**, and Topic 75's scalar replacement cannot touch them.

### What you observe, in the order you actually meet it

1. **Nobody notices for two days.** Tracing works. Dashboards look great. Everyone is
   pleased.
2. **The weekly load-test job fails its gate.** The Topic 65 regression run reports p99 on
   `POST /orders` outside the ±10% band. p50 barely moves. **p99 moves and p50 does not** —
   remember that shape.
3. **Startup time is up.** The deployment's readiness probe now takes noticeably longer to
   pass. Nobody has a number for it, because nobody was recording startup time. (Topic 122
   is why you should have been.)
4. **Metaspace is up.** The NMT `Class` category delta from your Topic 80 baseline has
   grown. If `-XX:MaxMetaspaceSize` was set tight from that work, you may now be near it.
5. **The `orderflow` team looks at the wrong week's commits.** The application diff for
   that period is a copy change and a test fix. Neither can explain a p99 regression on
   order placement. Somebody says "flaky load test" and re-runs it. It fails again.

### The correct diagnosis, in the order you should do it

**Step 1 — establish that the agent is the variable.** Do not theorise. Re-run the
recorded Topic 65 scenario twice, identical in every respect except the `-javaagent` flag.

```bash
# arm A — the exact baseline configuration
docker run --rm --memory=2g --cpus=2 \
  -e JAVA_TOOL_OPTIONS="-Xmx1g -XX:NativeMemoryTracking=summary" \
  -p 8080:8080 orderflow:baseline

# arm B — identical, plus the agent, and nothing else
docker run --rm --memory=2g --cpus=2 \
  -e JAVA_TOOL_OPTIONS="-Xmx1g -XX:NativeMemoryTracking=summary \
     -javaagent:/otel/opentelemetry-javaagent.jar -Dotel.traces.exporter=none" \
  -p 8080:8080 orderflow:baseline
```

Note `-Dotel.traces.exporter=none` in arm B. **This separates the instrumentation cost from
the export cost.** If arm B is slow with the exporter off, the cost is bytecode and
allocation, not network. If arm B is fine with the exporter off and slow with it on, your
problem is the exporter's batching or the collector, which is a completely different fix.
This one flag turns "the agent is slow" into two separable hypotheses.

**Step 2 — decompose the startup regression.** Spring Boot prints two numbers:

```
Started OrderflowApplication in <n> seconds (process running for <m>)
```

*Illustration of the format, not captured output.*

- `<n>` is Spring's own context refresh and bean instantiation.
- `<m>` is JVM uptime — which **includes** JVM init, all agent `premain` calls, and all
  class loading before Spring started.
- **`<m> − <n>` is the pre-Spring cost, and the agent's `premain` lives entirely inside
  it.** Compare that difference between arm A and arm B and you have attributed the agent's
  startup cost without any special tooling.

**Step 3 — look for changed inlining on the specific methods that regressed.** This is the
part almost nobody does, and it is what makes the diagnosis definitive rather than
plausible.

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     -Xmx1g -jar orderflow.jar > inlining-baseline.txt 2>&1

java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     -Xmx1g -javaagent:/otel/opentelemetry-javaagent.jar \
     -jar orderflow.jar > inlining-agent.txt 2>&1
```

Run identical load against both — **the compiler only makes decisions about code it has
seen run**, so an un-warmed process tells you nothing. Then:

```bash
grep -E "com\.orderflow\.(orders|inventory|wallet)" inlining-baseline.txt \
  | sed 's/@ [0-9]*//' | sort -u > a.txt
grep -E "com\.orderflow\.(orders|inventory|wallet)" inlining-agent.txt \
  | sed 's/@ [0-9]*//' | sort -u > b.txt
diff a.txt b.txt
```

*Illustration of the format, not captured output.* `PrintInlining` lines look like this
shape — I am showing you the **columns and the reason strings**, with `<n>` where a number
would be:

```
   @ <bci>   com.orderflow.orders.OrderService::priceOf (<n> bytes)   inline (hot)
   @ <bci>   com.orderflow.inventory.InventoryService::reserve (<n> bytes)   inline (hot)
   @ <bci>   com.orderflow.inventory.InventoryService::reserve (<n> bytes)   callee is too large
   @ <bci>   com.orderflow.orders.OrderService::placeOrder (<n> bytes)   hot method too big
   @ <bci>   io.opentelemetry.javaagent.shaded...::onEnter (<n> bytes)   inline (hot)
```

**How to read those columns, which is the actual skill:**

| Field | Meaning | What to do with it |
|---|---|---|
| `@ <bci>` | bytecode index of the call site in the caller | ignore for this purpose; it is how you locate the call in `javap -c` output (Topic 76) |
| indentation | inlining **depth** — each level is one nested inline | deep trees are where `MaxInlineLevel` bites |
| `(<n> bytes)` | the **callee's bytecode size** | **this is the number the agent changed.** Compare it between the two runs for the same method. |
| `inline (hot)` | it was inlined, and the call site was hot | good |
| `callee is too large` | callee bytecode exceeded `MaxInlineSize`/`FreqInlineSize` | **the smoking gun.** Present in arm B, absent in arm A, for the same method. |
| `hot method too big` | the **caller** is already too large to accept more inlining | a knock-on effect: the agent inflated the caller, so unrelated callees stop being inlined |
| `not inlineable` | native, or otherwise ineligible | usually uninteresting |
| `too big` | size-based refusal in a cold context | check whether the call site is genuinely cold |

**The finding you are looking for is one line moving from `inline (hot)` to `callee is too
large`, for a method on the order path.** That is the observer effect, named, in the
compiler's own words.

**Step 4 — confirm the downstream consequence.** Lost inlining is not itself a latency
number. Prove the chain with an allocation profile (Topic 78, async-profiler in `alloc`
mode) run against both arms. If a value object that did not appear in arm A's allocation
profile appears in arm B's, you have the full chain: bigger method → not inlined → escape
analysis lost → allocation reappears → higher allocation rate → more young collections →
worse p99.

### The fix, in four layers

**Layer 0 — decide whether it is worth fixing.** Say this out loud before touching
anything: *distributed tracing has real value, and a p99 regression on the order path has
real cost.* Topic 129 is where you put a number on both. Do not silently delete the agent
because it made a graph worse.

**Layer 1 — narrow the agent's matchers.** The single highest-leverage change. Most agents
instrument far more than you need by default.

```
-Dotel.instrumentation.common.default-enabled=false
-Dotel.instrumentation.spring-webmvc.enabled=true
-Dotel.instrumentation.jdbc.enabled=true
-Dotel.instrumentation.hikaricp.enabled=true
```

> **Honest uncertainty, one line:** the property names and the granularity of these
> switches vary by agent and by version. **Settle it:** read your agent's own documentation
> for the version you have deployed, and verify empirically by diffing
> `-Xlog:class+load` counts and `PrintInlining` output before and after. Do not trust my
> property names.

This turns "instrument everything" into "instrument the four boundaries we actually put on
a dashboard", which is usually what the team wanted anyway.

**Layer 2 — if a specific hot method is the victim, make it smaller.** If
`InventoryService.reserve` is 20 bytes under the threshold before instrumentation, it is
fragile regardless of the agent. Extract the cold part into a separate method, so the hot
part stays small and inlinable.

```java
// Before: one method that does the fast path and the contended slow path.
public Reservation reserve(long productId, int quantity) {
    Inventory inv = inventoryRepository.findForUpdate(productId);
    if (inv.available() < quantity) {
        auditLog.record(productId, quantity, inv.available());
        metrics.counter("inventory.insufficient").increment();
        throw new InsufficientInventoryException(productId, quantity, inv.available());
    }
    inv.decrement(quantity);
    return new Reservation(productId, quantity);
}

// After: the hot path is small; the failure path is a separate, non-inlined method.
public Reservation reserve(long productId, int quantity) {
    Inventory inv = inventoryRepository.findForUpdate(productId);
    if (inv.available() < quantity) {
        return failReservation(productId, quantity, inv.available());   // cold, out of line
    }
    inv.decrement(quantity);
    return new Reservation(productId, quantity);
}

private Reservation failReservation(long productId, int quantity, int available) {
    auditLog.record(productId, quantity, available);
    metrics.counter("inventory.insufficient").increment();
    throw new InsufficientInventoryException(productId, quantity, available);
}
```

This is a standard technique and it is **good practice independent of agents** — it keeps
hot bytecode small so that C2 has room. Do not do it speculatively. Do it when
`PrintInlining` names the method.

**Layer 3 — raise the threshold, only with evidence and only as a last resort.**

```
-XX:FreqInlineSize=<larger value>
```

This is a global change with global consequences: more inlining means larger compiled
methods, a larger code cache (native memory, Topic 80), longer compilation times, and
possibly *worse* instruction-cache behaviour. **Never do this first.** Do it only when
`PrintInlining` names a specific method, the method genuinely cannot be shrunk, and you
re-run the full Topic 65 baseline afterwards to prove it helped rather than moved the
problem.

**Layer 4 — re-run the baseline and record the new numbers.** Whatever you settle on
becomes the new recorded baseline, with a note in `/docs/java/baselines/` saying *this
baseline includes the OTel agent with matchers narrowed to four instrumentations*. A
baseline that does not name its configuration is not a baseline.

### The budget line this produces

The output of this exercise is not "the agent is bad". It is a line in a table you can
defend:

| Item | Cost | Measured how |
|---|---|---|
| Startup | agent `premain` + extra class loading | `<m> − <n>` from Spring's startup line, both arms |
| Metaspace | agent classes + generated classes | NMT `Class` delta, both arms |
| Code cache | larger compiled methods | NMT `Code` delta, both arms |
| p50 | usually small | Topic 65 scenario, both arms |
| p99 | the number that moved | Topic 65 scenario, both arms |
| Allocation rate | span/scope objects + lost scalar replacement | async-profiler alloc, both arms |
| Value delivered | end-to-end traces across HTTP → Kafka → consumer | Topic 119 |

**That table is the deliverable.** "Tracing costs us X on p99 and Y megabytes of metaspace,
and here is what it buys" is a senior engineer's sentence. "The agent made it slow" is not.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — blaming a release for a slowdown when the only change was attaching an agent

**Wrong:** treating a `-javaagent` flag as a non-change because it touches no application
code.

**Exact symptom, precisely:**

- The Topic 65 regression run reports **p99 outside the ±10% gate** on `POST /orders`.
- **p50 is essentially unchanged.** p95 moves a little. p99 and p999 move most.
- `git log` for the period contains no change to the order path. The application jar's
  checksum is identical to the one that produced the baseline.
- CPU utilisation at the same request rate is **higher**, not lower — the service is doing
  more work per request, not waiting more.
- An allocation profile (Topic 78, alloc mode) shows object types on the order path that
  the baseline profile did not contain — including short-lived value objects that were
  previously scalar-replaced.
- `-XX:+PrintInlining` shows at least one order-path method that reads `inline (hot)` in
  the baseline run and `callee is too large` in the current run.
- GC logs show **more frequent young collections** at the same request rate, with
  unchanged pause durations.

**Root cause:** the agent's `ClassFileTransformer` rewrote the method bodies on the order
path. The added advice bytecode pushed at least one hot method past `FreqInlineSize`. C2
stopped inlining it. Without the inline, escape analysis could no longer prove the
`Reservation` value object non-escaping, so scalar replacement stopped (Topic 75). The
allocation reappeared, allocation rate rose, young-collection frequency rose, and the tail
widened. Separately, the span and scope objects genuinely escape into a thread-local
context, so they are real allocations that no compiler can remove.

**Why the team blames the release:** because the timeline correlates with a deploy, and
because "we turned on tracing" is filed mentally as configuration, not as a change to the
executed code. It is a change to the executed code. It is the *most literal possible*
change to the executed code.

**Fix — in this order:**

1. **A/B the flag, nothing else.** Same image, same data, same k6 script, agent on and
   agent off. If the delta is not reproducible, stop; it is not the agent.
2. **Separate instrumentation cost from export cost** with the exporter disabled
   (`-Dotel.traces.exporter=none` or equivalent). Two different problems, two different
   owners.
3. **Diff `-XX:+PrintInlining`** between the two arms, filtered to your packages. Find the
   method whose reason string changed.
4. **Narrow the agent's matchers** to the boundaries you actually need.
5. **Shrink the named hot method** by extracting its cold path.
6. **Re-run the Topic 65 baseline** and record the result, whichever way it goes.

**The rule to carry:** *an agent is a code change delivered by a flag. Review it like a
code change and measure it like a code change.*

---

### Trap 2 — an agent adds seconds to startup, and Spring gets the blame

**Wrong:** reading "Started OrderflowApplication in <n> seconds" as the startup time.

**Exact symptom, precisely:**

- The Kubernetes readiness probe's `initialDelaySeconds` is no longer sufficient. Pods are
  marked unready, then restarted by an over-eager liveness probe (Topic 121's failure
  mode, triggered by this topic's cause).
- Rolling deploys take materially longer, and a rollback takes longer too — which is worse,
  because rollbacks happen during incidents.
- **Spring's own "Started in `<n>` seconds" number has barely moved.** The number that moved
  is `(process running for <m>)`.
- `-Xlog:class+load=info | wc -l` returns a substantially larger class count with the agent
  attached.
- `jcmd <pid> VM.classloader_stats` shows classes under a loader you do not recognise —
  the agent's, and ByteBuddy's generated-class loader.
- If AppCDS is in use (Topic 122), `-Xlog:class+load` shows **fewer** classes annotated as
  coming from the shared archive than it did before.

**Root cause:** three costs stack, and none of them are inside Spring's timer.

1. `premain` runs before `main`. It loads the agent's classes, builds its matchers, and (for
   ByteBuddy) initialises a type pool. All of it is synchronous, on the main thread, before
   Spring exists.
2. Every class definition thereafter passes through the transformer chain. A Spring Boot
   app with Hibernate defines thousands of classes. Even a fast per-class matcher, times
   thousands, is a real number.
3. Transformed classes cannot be served from the CDS/AppCDS archive, so a startup
   optimisation you already paid for is partially defeated.

**Fix:**

1. **Measure the split first.** `<m> − <n>` is pre-Spring; `<n>` is Spring. Attack the
   bigger one. This is Topic 122's discipline applied here.
2. **Narrow the matchers.** Fewer types considered means fewer matcher evaluations across
   thousands of classes.
3. **Check whether the agent supports a "muzzle"/prefilter mode** that skips whole
   classloaders or package prefixes cheaply. Most production agents do.
4. **Re-measure AppCDS effectiveness with the agent attached** rather than assuming your
   earlier archive still helps.
5. **Raise `initialDelaySeconds`** as a stopgap, and say out loud that it is a stopgap.

**The rule to carry:** *Spring's startup number does not include the JVM's startup. The gap
between the two is where agents live.*

---

### Trap 3 — the transformer throws, and the JVM says nothing at all

**Wrong:** assuming a broken agent produces an error.

**Exact symptom, precisely:**

- The agent's `premain` output appears. It clearly ran.
- **No spans appear.** No timing output. No instrumentation whatsoever, for some classes or
  for all of them.
- The application **starts and works perfectly**. No exception, no warning, no log line.
- `getClass().getName()` on the supposedly-instrumented object is completely normal —
  because it always is, agent or no agent. You cannot tell by looking.
- If you add your own `catch (Throwable)` inside the transformer, you suddenly see a
  `NoClassDefFoundError`, a ByteBuddy `IllegalStateException`, or a
  `VerifyError`-adjacent failure that was there all along.

**Root cause:** the `java.lang.instrument` specification says an exception thrown from
`transform` is **ignored**, and the class is defined from the original bytes. This is
deliberate — a broken agent must not be able to prevent an application from running — and it
means a broken agent is **completely silent**.

There is a second, related cause with the same symptom: the transformer returned bytes that
failed **bytecode verification**. In that case you usually *do* get a `VerifyError`, but it
surfaces at the point of class definition, deep in a stack trace that names your
application class, not the agent — so it gets misattributed.

**Fix:**

1. **Never write a transformer without a `try { ... } catch (Throwable t)` around the whole
   body**, logging and returning `null`. This is not optional.
2. **Turn on your bytecode library's diagnostics.** ByteBuddy's `AgentBuilder` has
   listeners; wire them:
   ```java
   new AgentBuilder.Default()
       .with(AgentBuilder.Listener.StreamWriting.toSystemError().withErrorsOnly())
       .with(AgentBuilder.InstallationListener.StreamWriting.toSystemError())
       // ... matchers and transforms ...
       .installOn(inst);
   ```
3. **Verify what actually got defined**, rather than trusting the agent. Dump the
   transformed bytes to disk from inside the transformer and run `javap -c` on them
   (Topic 76). If the advice is not in the bytecode, the transform did not happen.
4. **Check the class-name form.** Internal form, with slashes. This is the cause a
   depressing fraction of the time.
5. For third-party agents, **check their debug flag** (most have one) rather than guessing.

**The rule to carry:** *a silent agent is the expected failure mode, not an unusual one.
Design for visibility or you will have none.*

---

### Trap 4 — two agents, and the order on the command line changes the behaviour

**Wrong:** treating `-javaagent` flags as an unordered set.

**Exact symptom, precisely:**

- Spans are **duplicated** — a method appears twice in the trace, nested inside itself —
  or a span is **missing** entirely.
- A profiler's flame graph shows frames belonging to the *other* agent's shaded packages
  inside your application's methods.
- Behaviour differs between environments where the flags happen to be assembled in a
  different order — for example, `JAVA_TOOL_OPTIONS` from a ConfigMap concatenated with
  flags from a Dockerfile `ENTRYPOINT`.
- Removing either agent "fixes" it, which sends everyone down the wrong path, because both
  agents are individually fine.
- Startup cost is **more than the sum** of the two agents measured separately.

**Root cause:** agents' `premain` methods run in command-line order, and their transformers
are registered in that order. **The second agent's transformer receives the bytes the first
one produced.** Agent 2 therefore instruments agent 1's injected code, or matches on a
method shape that agent 1 has already changed, or (with a broad matcher) instruments agent
1's own classes.

A related failure in the same family: an agent that is **not shaded** puts its dependencies
on the application classpath, where Maven's flat nearest-wins resolution (Topic 32) never
saw them, producing a `NoSuchMethodError` from a library the application never declared.

**Fix:**

1. **Pin the order explicitly and document why.** Instrumentation agents generally go
   before profiling agents, so the profiler sees the code that will actually run — but
   verify against your specific agents rather than trusting a rule of thumb.
2. **Assemble flags in exactly one place.** Do not concatenate `JAVA_TOOL_OPTIONS` from
   multiple sources; make one file or one Helm value authoritative.
3. **Exclude each agent's packages from the others' matchers.** Every serious agent
   supports an exclusion list; use it.
4. **Prefer one agent.** Two general-purpose bytecode-rewriting agents in the same JVM is a
   configuration to justify, not a default.
5. **Verify with `-Xlog:class+load`** that both agents' classes are present, and with a
   dump of the transformed bytes that the instrumentation is what you expect.

**The rule to carry:** *`-javaagent` is an ordered pipeline over the same bytes, not a set
of independent plugins.*

---

### Trap 5 — attaching the agent to a running JVM and getting different behaviour

**Wrong:** assuming `-javaagent` at launch and a dynamic attach are equivalent.

**Exact symptom, precisely:**

- On JDK 21+, the attach prints a **warning** about dynamic agent loading, which the person
  doing it during an incident ignores.
- Instrumentation is **partial**. Framework entry points get spans; internals do not. Some
  libraries are instrumented and others silently are not.
- Classes already loaded before the attach are **not instrumented at all** unless the agent
  explicitly retransforms them — and the agent may not, because retransformation is
  expensive and it does not know which classes matter.
- An agent that adds fields on the load path throws `UnsupportedOperationException` on the
  retransform path, and the agent falls back to a slower side-table strategy — so you get
  instrumentation, but with a different performance profile than the launch-time path.
- Traces from the dynamically-attached pod differ in **shape** from traces from
  launch-time pods, which makes the trace data internally inconsistent and hard to reason
  about.

**Root cause:** two independent restrictions.

1. **Timing.** By the time you attach, most classes are already defined. `premain` gets
   first refusal on every class; `agentmain` gets whatever is left plus whatever it chooses
   to retransform.
2. **Capability.** `retransformClasses` may change method bodies, the constant pool and
   attributes. It may **not** add or remove fields or methods, change signatures, or change
   the hierarchy. An instrumentation strategy that adds a field works at load time and is
   illegal at retransform time.

**Fix:**

1. **Attach at launch in every environment that produces numbers you compare.** Baselines
   are only comparable if the configuration is.
2. **Reserve dynamic attach for genuine investigation** — a heap dump, a thread dump, a
   short profiling session — not for permanent instrumentation.
3. **Know your JDK's posture** on dynamic loading:
   `java -XX:+PrintFlagsFinal -version | grep -i EnableDynamicAgentLoading`, and read the
   warning text on an actual attach.
4. **If you must attach dynamically, drive the retransformation explicitly** and record
   which classes were retransformed, so you know what your data covers.
5. **Never compare a dynamically-attached run to a launch-time baseline** and call the
   difference a code change.

**The rule to carry:** *`premain` sees everything before it exists; `agentmain` negotiates
with what already does.*

---

## Hands-on proof

Every command below is one **you** run. I have no JVM, so I print no output and claim
nothing as captured. What follows is the exact command, what to look for, and how to read
each result you might get.

### Setup

```bash
mkdir -p ~/java-lab/81 && cd ~/java-lab/81
java --version           # note 21 or 25; it changes the dynamic-attach story
```

### Proof 0 — does your JDK warn on dynamic attach?

```bash
java -XX:+PrintFlagsFinal -version | grep -i EnableDynamicAgentLoading
```

| What you see | What it means |
|---|---|
| A `bool EnableDynamicAgentLoading = true {...}` line | Dynamic attach is permitted; expect a warning on use. |
| Origin shown as `default` | Nobody has overridden it in your environment. |
| No output at all | The flag does not exist on this JDK — you are on a release before JEP 451 landed, or a non-HotSpot VM. |

Then attach something trivial and **read the warning text**, because you will see it in a
production log one day and you should recognise it instantly.

### Proof 1 — the agent runs before `main`

Build and run Example 1's `ClassLoadObserverAgent`.

| What you see | What it means |
|---|---|
| All agent output precedes `[app] main entered` | `premain` completes before `main`. The lifecycle, proven. |
| The main class appears in the transformer log before the app prints anything | The transformer ran **before the class was defined**. This is the mechanical statement. |
| `classes already loaded` is a large number | Those are pre-agent JDK classes. They are only reachable via `retransformClasses`. |

### Proof 2 — lazy loading, demonstrated by the agent

Add a class that is only touched in a branch that does not execute. Re-run.

| What you see | What it means |
|---|---|
| The unused class never appears in the transformer log | Class loading is lazy (Topic 67). An agent only sees classes that are actually used. |
| It appears after you make the branch execute | Confirms the same fact from the other direction. |

**Why this matters for an APM:** an agent cannot instrument what never loads, and it
instruments things *the first time they are used*, which is inside your request path.
Cold-start instrumentation cost is paid by the first request that touches each class.

### Proof 3 — `null` versus the same array

Change the transformer's `return null` to `return classfileBuffer`, keeping everything else
identical. Run against a class-heavy application (`orderflow` itself is ideal).

| What you see | What it means |
|---|---|
| Startup is measurably slower with `return classfileBuffer` | Returning non-null forces the full class-redefinition path even though nothing changed. **`null` is a meaningful optimisation**, not a style choice. |
| No difference | Your class count is too small for the effect to show. Retry against `orderflow`. |

### Proof 4 — see the rewritten bytecode with your own eyes

Dump what the transformer produces, then decompile it. This is the step that converts
belief into knowledge.

```java
// inside the transformer, after computing the new bytes
java.nio.file.Path out = java.nio.file.Path.of("/tmp/dump",
        className.replace('/', '.') + ".class");
java.nio.file.Files.createDirectories(out.getParent());
java.nio.file.Files.write(out, transformedBytes);
```

```bash
javap -c -p /tmp/dump/com.orderflow.inventory.InventoryService.class | less
```

| What you see | What it means |
|---|---|
| Extra opcodes at the start and end of the method, plus an exception table entry | **This is the advice, inlined.** Compare against `javap -c` of the original class file from the jar. |
| An `Exception table:` section that the original did not have | The `try/finally` the advice needs. This is part of why the method grew. |
| A larger `Code:` attribute size than the original | **The number that pushes a method past the inlining threshold.** Write both numbers down. |
| Identical output to the original | The transform did not fire. Go back to Trap 3. |

### Proof 5 — the inlining decision, before and after

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:-BackgroundCompilation \
     -cp out com.orderflow.demo.Bench > no-agent.txt 2>&1

java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:-BackgroundCompilation \
     -javaagent:timing-agent.jar \
     -cp out com.orderflow.demo.Bench > with-agent.txt 2>&1

diff <(grep com.orderflow no-agent.txt | sed 's/@ [0-9]*//' | sort -u) \
     <(grep com.orderflow with-agent.txt | sed 's/@ [0-9]*//' | sort -u)
```

`-XX:-BackgroundCompilation` makes compilation synchronous, which makes the output far more
deterministic and worth the slowdown for this purpose.

| What you see | What it means |
|---|---|
| A method's reason string changes from `inline (hot)` to `callee is too large` | **The observer effect, in the compiler's own words.** This is the finding. |
| The `(<n> bytes)` figure for the same method is larger in the agent run | The direct measurement of what the transform did. |
| A *caller* gains `hot method too big` | Second-order effect: the caller was inflated, so unrelated callees stopped inlining. Note which ones. |
| No difference at all | Your target methods are not near a threshold. That is a **legitimate and common result** — say so honestly. Try a method whose size you can tune deliberately (see the drill). |
| Wildly different output run to run | Compilation is non-deterministic under load. Use `-XX:-BackgroundCompilation`, warm properly, and compare several runs. |

### Proof 6 — the agent's footprint in native memory

```bash
# both arms started with -XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory baseline
# ... run identical load ...
jcmd <pid> VM.native_memory summary.diff
jcmd <pid> VM.classloader_stats
```

| What you see | What it means |
|---|---|
| A larger `Class` committed figure in the agent arm | Agent classes plus generated classes. Metaspace is native (Topic 68) and counts against the container limit (Topic 80). |
| A larger `Code` committed figure in the agent arm | Bigger methods compile to more machine code. |
| An unfamiliar classloader in `VM.classloader_stats` with many classes | ByteBuddy's generated-class loader. Growth here over time is a **classloader leak**, and it is a real agent failure mode. |
| `Command executed with 'summary.diff' but NMT is off` | NMT must be on at JVM start. Redeploy. Topic 80 said this too. |

### Proof 7 — proving an agent reaches where a proxy cannot

This is the demonstration that makes the Topic 40 comparison concrete.

```java
package com.orderflow.demo;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SelfInvocationDemo {

    public void outer() {
        System.out.println("outer, proxy class = " + this.getClass().getName());
        inner();                      // self-invocation: bypasses the Spring proxy
    }

    @Transactional
    public void inner() {
        System.out.println("inner");
    }
}
```

Run it with your ByteBuddy timing agent matching `com.orderflow.demo`.

| What you see | What it means |
|---|---|
| `[timing]` fires for **both** `outer` and `inner` | The agent rewrote both method bodies. Self-invocation is irrelevant to it. |
| The Spring transaction still does **not** apply to `inner` | The proxy is still bypassed. **Two mechanisms, two different answers, same call.** |
| `this.getClass().getName()` contains `$$SpringCGLIB$$` | The proxy announces itself. The agent does not — the class name is unchanged. |

Carry one sentence out of this proof: *a proxy wraps the object; an agent rewrites the
method. Only one of them cares how you called it.*

---

## Failure drill

**Mandatory.** Do not read the interpretation tables until you have produced the numbers
yourself. The point is not the knowledge — it is the memory of watching a p99 move because
of a flag, and then finding the reason in the compiler's own output.

### The scenario, exactly as assigned

Attach an APM/OTel agent to `orderflow`. Re-run the recorded Topic 65 baseline. Compare
startup time and p99 against the recorded numbers. Then look for changed inlining with
`-XX:+PrintInlining`.

### Part A — a standalone rig, to learn the tools

Do this first, on a program you fully control, so that when you meet the real thing you are
reading output rather than learning to read output.

**`InliningVictim.java`** — a method deliberately sized so that a small addition pushes it
over the edge:

```java
package com.orderflow.demo;

public class InliningVictim {

    private long acc;

    // Deliberately near the inlining threshold. Add or remove a line to move it.
    public long step(long x) {
        long a = x * 31;
        long b = a ^ (a >>> 7);
        long c = b + (b << 3);
        long d = c ^ (c >>> 11);
        return d + (d << 5);
    }

    public long run(int iterations) {
        for (int i = 0; i < iterations; i++) {
            acc += step(i);
        }
        return acc;
    }

    public static void main(String[] args) {
        InliningVictim v = new InliningVictim();
        // Crude warm-up. This is NOT a benchmark - see the Measurement section.
        for (int round = 0; round < 20; round++) {
            v.run(1_000_000);
        }
        System.out.println("acc = " + v.acc);
    }
}
```

**Establish the baseline inlining decision:**

```bash
javac -d out InliningVictim.java
javap -c -p out/com/orderflow/demo/InliningVictim.class | grep -A2 "long step"
# note the Code attribute size of step()

java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -XX:-BackgroundCompilation \
     -cp out com.orderflow.demo.InliningVictim 2>&1 | grep -i "InliningVictim::step"
```

**Now attach the timing agent from Example 1 and repeat.**

### Part B — what to write down before reading on

Six numbers and one sentence, in a notes file, not in your head.

1. The bytecode size of `step` **without** the agent (from `javap -c`).
2. The bytecode size of `step` **with** the agent (dump the transformed bytes, Proof 4).
3. `MaxInlineSize` and `FreqInlineSize` on your JVM (`PrintFlagsFinal`).
4. The `PrintInlining` reason string for `step` without the agent.
5. The `PrintInlining` reason string for `step` with the agent.
6. Whether `run` gained a `hot method too big` line.
7. One sentence: *why did the size change, in terms of what the advice added?*

### Part C — how to read it

| What you see | What it means |
|---|---|
| `step` is `inline (hot)` without the agent and `callee is too large` with it | **The drill has fired.** You have reproduced the entire mechanism in a program you can read end to end. |
| `step` is inlined in both cases | Your advice is small or your method is far from the threshold. **Grow the advice** (add a `String.format` to the exit advice) or **grow the method** (add two more mixing lines) until you cross. Crossing it deliberately is the lesson. |
| `step` is `too large` in **both** cases | The method was already over. Shrink it and start again from a state where it inlines. |
| `run` gains `hot method too big` | A second-order effect: the caller was inflated by the advice too, so it stopped accepting inlines. Note this — it is the version of the effect that hits methods you did not instrument. |
| The output is different on every run | Compilation is non-deterministic. Use `-XX:-BackgroundCompilation`, warm longer, and compare three runs before concluding anything. |
| `PrintInlining` prints nothing for your method | It never got hot. Raise the iteration count. C2 says nothing about code it never compiled. |

### Part D — the real drill, on `orderflow` under the Topic 65 baseline

This is the version that counts, because it produces numbers you compare against a recorded
baseline rather than against an intuition.

**1. Confirm the baseline is still valid.** Before touching anything, re-run the recorded
Topic 65 scenario against the unmodified image and check it lands within ±10%. **If it does
not, stop.** You have a drifted environment and every number you take today is worthless.
This is the gate rule from Topic 65 and it exists for exactly this moment.

**2. Record the pre-agent facts.**

```bash
docker run --rm --memory=2g --cpus=2 \
  -e JAVA_TOOL_OPTIONS="-Xmx1g -XX:NativeMemoryTracking=summary \
     -Xlog:class+load=info:file=/tmp/classload-A.log \
     -Xlog:gc:file=/tmp/gc-A.log:time,uptime" \
  -p 8080:8080 orderflow:baseline
```

Capture, before load starts:

- Spring's startup line: both `<n>` and `<m>` from `Started ... in <n> seconds (process running for <m>)`.
- `wc -l /tmp/classload-A.log` — the class count.
- `jcmd 1 VM.native_memory summary` — the `Class` and `Code` committed figures.

Then run the **recorded k6 scenario, unmodified**, and capture p50/p95/p99/p999 and
throughput per endpoint.

**3. Add the agent and change nothing else.**

```bash
docker run --rm --memory=2g --cpus=2 \
  -e JAVA_TOOL_OPTIONS="-Xmx1g -XX:NativeMemoryTracking=summary \
     -javaagent:/otel/opentelemetry-javaagent.jar \
     -Dotel.service.name=orderflow -Dotel.traces.exporter=none \
     -Xlog:class+load=info:file=/tmp/classload-B.log \
     -Xlog:gc:file=/tmp/gc-B.log:time,uptime" \
  -p 8080:8080 orderflow:baseline
```

**The exporter is off on purpose.** You are measuring instrumentation cost, not network
cost. Do a third run later with the exporter on, and the difference between run 2 and run 3
is your exporter cost — a separate line item with a separate owner.

**4. Capture the same facts, then the inlining evidence.** Restart both arms with
`-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`, run a **short warm-up under the same
load** (the compiler must have seen the code run), and diff the filtered output as in
Proof 5.

**What to capture and compare against `/docs/java/baselines/`:**

| Metric | Where it comes from | What you are looking for |
|---|---|---|
| Startup, Spring's `<n>` | the startup log line | small change; Spring's own work barely differs |
| Startup, `<m> − <n>` | the same line | **the agent's `premain` and pre-main class loading.** This is where the seconds are. |
| Class count | `wc -l` of the class-load logs | higher with the agent; the metaspace story in one number |
| NMT `Class` committed | `VM.native_memory summary` | higher with the agent |
| NMT `Code` committed | same | higher with the agent |
| p50 per endpoint | k6 summary | usually within noise. **Say so if it is.** |
| p99 / p999 on `POST /orders` | k6 summary | the number most likely to move, and the one your gate cares about |
| Throughput at fixed arrival rate | k6 summary | if it dropped, CPU per request rose |
| Young-collection frequency | `gc-A.log` vs `gc-B.log` | more frequent at the same rate means a higher allocation rate |
| `PrintInlining` diff | the two filtered files | **at least one order-path method whose reason string changed** |
| Allocation profile | async-profiler alloc mode (Topic 78) | object types present in arm B and absent in arm A |

### Part E — what counts as having completed the drill

You have finished when you can write these four sentences with your own numbers in them:

1. *The agent cost `<m> − <n>` additional pre-Spring startup, and `<k>` additional loaded
   classes.*
2. *p50 changed by X%; p99 changed by Y%. The gate says Z.*
3. *Method `com.orderflow.<something>` went from `inline (hot)` to `callee is too large`,
   and its bytecode grew from A to B bytes.*  **(Or: no inlining decision changed on the
   order path — which is an equally valid and equally publishable finding.)*
4. *Therefore the observed p99 change is / is not explained by lost inlining, and the
   remaining explanation is <span allocation | exporter | GC frequency | noise>.*

**If your honest answer to 3 is "nothing changed", say that.** A drill that produces a null
result and proves it carefully is worth more than one that produces a dramatic result you
cannot attribute. The skill being trained is attribution, not drama.

### Part F — the fix, and the re-run

Narrow the agent's matchers to the boundaries you actually put on a dashboard. Re-run the
identical scenario. Compare against **both** the original baseline and the wide-matcher run.

**What the fix proves — and this is the point of the drill, not the fix:**

- The **same** agent, the **same** load, the **same** container limits, and a different
  cost, because the only variable was how much code was rewritten. Instrumentation cost is
  a **dial**, not a constant.
- The traces you actually use are still there. You gave up spans nobody looked at.
- You can now write the cost/benefit line in Topic 129's model with measured numbers on
  both sides.

Carry one sentence out of this drill: *the agent did not slow down my code; it changed my
code, and the compiler made a different decision about the code it was given.*

---

## Measurement

### The instrument for each claim

Every claim in this document maps to an instrument that could falsify it. That mapping is
the difference between engineering and folklore.

| Claim | Instrument that makes it falsifiable |
|---|---|
| "The transformer runs before the class is defined" | Example 1's agent: transformer log line precedes any output from the class |
| "The agent rewrote this method" | dump the transformed bytes from the transformer, then `javap -c` (Topic 76) |
| "The method got bigger" | the `Code:` attribute size in `javap -c`, original jar vs dumped bytes |
| "It is no longer inlined" | `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`, diffed between arms |
| "Escape analysis stopped" | async-profiler `alloc` mode: the object type appears in one arm and not the other |
| "The allocation rate went up" | `-Xlog:gc` young-collection frequency at a fixed arrival rate |
| "It cost startup time" | `<m> − <n>` from Spring's startup line, both arms |
| "It cost metaspace / code cache" | `jcmd VM.native_memory baseline` then `summary.diff` (Topic 80) |
| "It generated classes that are leaking" | `jcmd <pid> VM.classloader_stats` sampled over hours |
| "It cost p99" | **the recorded Topic 65 p50/p95/p99, re-run identically** |
| "The exporter, not the instrumentation, is the cost" | a third arm with the exporter enabled |

That last-but-one row is the one people skip. **An instrumentation change that costs you p99
is not free observability; it is a trade you made without measuring.** The Topic 65 baseline
exists precisely so that every Phase 8 change can be checked against it.

### Startup-time decomposition — the four slices

You cannot fix a startup problem you have not decomposed. There are four slices, and each
has a different instrument and a different fix.

| Slice | What happens | How to measure it | What fixes it |
|---|---|---|---|
| **1. JVM init** | VM creation, heap reservation, core JDK classes, **every agent's `premain`** | `<m> − <n>` from Spring's startup line; subtract a run with no agent to isolate the agent | fewer agents, narrower matchers, smaller heap pre-touch |
| **2. Class loading** | finding, verifying, defining application and library classes — **every one passing through every transformer** | `-Xlog:class+load=info` line count; `-Xlog:class+load` entries marked as coming from the shared archive | AppCDS / the JDK 25 AOT cache (Topic 122), fewer dependencies, narrower agent matchers |
| **3. Context refresh** | Spring reads bean definitions, evaluates `@Conditional`s, runs `BeanFactoryPostProcessor`s | Boot's `--debug` condition-evaluation report; `spring.main.lazy-initialization=true` as a **diagnostic** A/B | fewer auto-configurations, explicit `@Import` over broad scanning |
| **4. Bean instantiation** | constructing singletons, building the EntityManagerFactory, warming pools | `BufferingApplicationStartup` + `/actuator/startup`, sorted by duration | lazy init for genuinely cold beans, defer pool warm-up, fix the one slow `@PostConstruct` |

Wiring slice 4 up is three lines and it is worth doing permanently:

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(OrderflowApplication.class);
    app.setApplicationStartup(new BufferingApplicationStartup(4096));
    app.run(args);
}
```
```
management.endpoints.web.exposure.include=startup
```
```bash
curl -s localhost:8080/actuator/startup | jq '.timeline.events
  | sort_by(.duration) | reverse | .[0:20]
  | map({name: .startupStep.name, tags: .startupStep.tags, duration})'
```

Step names include `spring.beans.instantiate`, `spring.context.beans.post-process`,
`spring.context.config-classes.parse`. The tags name the bean. **Sorting by duration and
reading the top twenty is the whole technique**, and it answers "which bean is slow" in one
command instead of one afternoon.

**The rule that matters:** *slices 1 and 2 are where an agent lives; slices 3 and 4 are
where Spring lives.* Attacking Spring for a slice-1 regression is how teams spend a month
on the wrong 200 milliseconds — which is Topic 122's warning and, in the next document,
Topic 83's most common bad decision.

### The standing rule: a naive `System.nanoTime()` measurement is wrong

You will be tempted to answer "how much does the agent cost per call?" like this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    inventoryService.reserve(productId, 1);
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/op");
```

Four independent reasons this lies, and you cannot tell which one is lying:

1. **Dead-code elimination.** If the result is unused, C2 may prove the call has no
   observable effect and delete it. You measure an empty loop.
2. **Warm-up state.** The loop begins interpreted, is on-stack-replaced mid-flight, and
   ends in C2 code. Your average blends three execution modes in a ratio decided by the
   loop count you happened to pick. **This is the specific way an agent benchmark lies
   worst**, because the agent's own classes also need warming, so the two arms warm at
   different rates.
3. **Profile pollution across arms.** Run both arms in one JVM and the first arm's profile
   affects the second's compilation (Topic 74). The order of your two measurements changes
   the result.
4. **The measurement is inside the instrumented region.** With the agent attached,
   `System.nanoTime()` calls in your loop may themselves sit next to injected advice, and
   the advice's own `System.nanoTime()` calls are counted in your elapsed time. **You are
   measuring the ruler.**

**Topic 77 is the full treatment.** The correct shape is a JMH harness with
`@State(Scope.Benchmark)`, `Blackhole.consume`, explicit warm-up iterations, and at least
`@Fork(3)` — and for *this* question specifically, the two arms must be **separate JVMs**,
because the agent is a JVM-level configuration and cannot be toggled inside a fork.

And the honest framing to lead with, before any benchmark: **per-call overhead is the wrong
question.** The number that matters is the end-to-end p99 of a real endpoint under the
recorded Topic 65 load, because that is where lost inlining, extra allocation, GC frequency
and exporter batching all compose. A per-call microbenchmark of an instrumented method
measures one of those four and silently omits the rest.

### What to graph in production, permanently

| Metric | Why |
|---|---|
| `jvm_memory_used_bytes{area="nonheap"}` | metaspace + code cache; the agent's footprint lives here |
| `jvm_classes_loaded_classes` | a **steadily rising** count with the agent attached is a generated-class leak |
| `jvm_memory_used_bytes{id="Metaspace"}` | separately, so you can see class growth without the code cache masking it |
| `jvm_compilation_time_ms_total` | more and larger methods means more compiler work |
| Startup duration, as an explicit metric | emit it from an `ApplicationReadyEvent` listener; you cannot regress what you do not record |
| p99 per endpoint, from your own service | the number the gate is about |
| The agent's own exporter queue/drop counters | most agents expose them; a full queue is a silent data-loss mode |

**Put `jvm_classes_loaded_classes` on a dashboard with a long time window.** A slow upward
ramp over days is the signature of an agent generating classes it never releases, and it
ends as a metaspace exhaustion at 3am with a stack trace nobody recognises.

---

## Practice exercises

### 1 — Easy: write an agent that answers a question you actually have

Write a `premain` agent that, at JVM shutdown, prints:

- the total number of classes it saw defined,
- the total bytes of class file it saw,
- the top ten classloaders by number of classes defined under them,
- the ten largest individual class files by byte size.

Run it against `orderflow`. Then run it against `orderflow` **plus** the OTel agent, with
the OTel agent listed **second** on the command line.

**Questions to answer in writing:**

1. How many classes does `orderflow` define, and how many does the OTel agent add?
2. Which classloaders appear only in the second run, and what are they?
3. Did any class file get **bigger** between the two runs? Which, and by how much? (Your
   agent is first on the command line, so it sees the original bytes — put your agent
   **second** as well, in a third run, and compare. **The difference between run 2 and run 3
   is the entire content of Trap 4**, demonstrated by you.)
4. What does the largest class file in your application turn out to be, and is that a
   surprise?

**Why this exercise:** it makes agent ordering concrete rather than theoretical, and it
gives you a permanently useful diagnostic tool that fits in one file.

### 2 — Medium: the observer-effect audit (combines Topics 40, 67, 68, 74, 75, 76, 80)

Take one hot method from `orderflow` — `InventoryService.reserve` is the natural choice —
and build a complete before/after profile of what an agent does to it. Produce a one-page
document with these sections:

**A. Bytecode.** `javap -c -p` on the class from the jar, and on the transformed bytes
dumped from a transformer. Report the `Code:` attribute size for the method in both, and
list every structural difference (new locals, new exception-table entries, new call sites).
*(Topic 76.)*

**B. Class loading.** When does this class load, relative to `main`? Prove it with Example
1's observer agent. What loads it? *(Topic 67.)*

**C. Compilation.** `-XX:+PrintCompilation` and `-XX:+PrintInlining` for both arms, warmed
identically. Report the inlining decision for the method, for its caller, and for one
callee. *(Topics 74 and 75.)*

**D. Allocation.** async-profiler in `alloc` mode for both arms. List every object type
present in one and not the other, and classify each as "the agent's own object" (span,
scope, context) or "an object that stopped being scalar-replaced". *(Topics 68 and 75.)*

**E. Native footprint.** NMT `summary.diff` for both arms: `Class` and `Code` deltas.
Express the agent's cost as a line in Topic 80's RSS equation, and state whether it fits
inside the 2 GB container budget you built there. *(Topic 80.)*

**F. Proxies versus agents.** The same method is also inside a Spring proxy. Print
`getClass().getName()` and say, in two sentences, which of the two mechanisms is
responsible for which observed behaviour, and what happens on a self-invocation. *(Topic
40.)*

**G. The verdict.** One paragraph: given all of the above, would you keep this
instrumentation on this method in production? What would you change first if the answer
were no?

**Success criterion:** section G is defensible to someone who disagrees with you, because
sections A–F contain measurements rather than assertions.

### 3 — Hard: production simulation against the `orderflow` baseline

You are on call. The following is the situation, and you have the recorded Topic 65
baseline and nothing else.

**The situation.** At 09:00 a platform-wide change rolled out tracing to every service via
a shared Helm chart. `orderflow`'s image did not change. At 14:00 the weekly load-test job
fails its gate: p99 on `POST /orders` is outside the ±10% band. At 15:00 a second alert
fires: two pods have restarted with `OOMKilled`, exit 137. At 15:30 someone in the incident
channel proposes reverting yesterday's application release, which contained a copy change
and a test fix.

**Build the following, with evidence at every step.**

1. **A one-paragraph hypothesis** written *before* you run anything, naming what you think
   changed and what would falsify it.
2. **A reproduction** of both symptoms locally, with `--memory=2g --cpus=2` and the
   recorded k6 scenario. Both symptoms — the p99 regression and the OOMKill — from the same
   configuration.
3. **An attribution of the OOMKill** using Topic 80's method: NMT baseline and
   `summary.diff` across the two arms. Say which category grew and by how much. State
   explicitly whether the OOMKill is caused by the agent's metaspace and code-cache
   footprint exceeding a `-XX:MaxMetaspaceSize` you set in Topic 80, or by something else.
   **A wrong-but-evidenced answer beats a right-but-unevidenced one here.**
4. **An attribution of the p99 regression** using this document's method: a `PrintInlining`
   diff naming a specific method, plus an allocation-profile diff, plus GC young-collection
   frequency at fixed arrival rate. If the inlining diff shows nothing, say so and pursue
   the remaining explanations.
5. **A separation of instrumentation cost from export cost** using three arms: no agent,
   agent with exporter off, agent with exporter on.
6. **A decision, with three options costed.** (a) Revert the tracing rollout for
   `orderflow`. (b) Keep it and narrow the matchers. (c) Keep it and raise the memory limit
   and the inlining threshold. For each: what it costs, what it risks, and what you would
   measure to confirm it worked.
7. **A written response to the revert proposal**, three sentences maximum, that explains why
   yesterday's application release cannot be the cause — in terms a product manager will
   accept.
8. **The new recorded baseline** in `/docs/java/baselines/`, explicitly naming the agent,
   its version, and its matcher configuration.

**The trap in this exercise:** two symptoms appeared at once and it is very tempting to
assume one cause. They may have one cause. They may not — the OOMKill could be the agent's
metaspace, or it could be an unrelated direct-buffer growth (Topic 80) that the extra
metaspace merely pushed over the edge. **Investigate them separately and only then argue
that they are connected.** Assuming a single root cause for two simultaneous symptoms is one
of the most common and most expensive mistakes in incident response.

---

## Interview questions

### Q1 — "We attached an APM agent and now the service is slower. Is that real, or is the team imagining it?"

**Mid-level answer:** "APM agents add overhead — every method call has to record a span, so
some slowdown is expected. We could sample less, or only instrument the important
endpoints."

**Senior answer:** "It is real, it is measurable, and there are at least three distinct
mechanisms, which need to be separated before we fix anything.

First, the direct cost: the agent registers a `ClassFileTransformer` and rewrites bytecode
at class load, so instrumented methods genuinely do more work — start a span, open a scope,
close them in a `finally`. Those spans are real objects that escape into a thread-local
context, so they are real allocations.

Second, and the one people miss: **rewriting a method makes its bytecode bigger, and
HotSpot's inlining decision is size-gated.** A hot method that was under `FreqInlineSize`
may now be over it. Once C2 stops inlining it, escape analysis on the caller has less to
work with, so objects that were previously scalar-replaced start being allocated again.
That is a second-order allocation-rate increase in code the agent did not touch. It shows up
as p99 moving while p50 barely does.

Third, the exporter: batching, queueing and network I/O to the collector, which is a
completely different problem with a different owner.

So the investigation is: A/B the flag alone against the recorded load-test baseline; run a
third arm with the exporter disabled to split instrumentation cost from export cost; and
diff `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` between arms, filtered to our
packages, looking for a method whose reason string moved from `inline (hot)` to `callee is
too large`. That last one turns 'the agent is slow' into 'this named method stopped being
inlined', which is actionable — we can narrow the matchers, or split the method's cold path
out to keep the hot path small.

And I would say clearly that this is not an argument against tracing. It is an argument for
treating a `-javaagent` flag as a code change, because that is literally what it is."

**What separates them:** the mid answer knows there is overhead and reaches for a knob. The
senior answer names **three separable mechanisms**, identifies the **non-obvious** one, gives
a **falsifiable procedure** for each, and — the part that gets people hired — refuses to
turn a measurement problem into a values argument about whether observability is worth it.

**Follow-up the interviewer asks:** "You said p99 moved and p50 did not. Why that shape?"
(Because inlining loss and extra allocation raise the *variance* more than the median: the
median request may never touch the affected path or may not coincide with a young
collection, while the tail is exactly the requests that hit a GC pause or a de-optimised
call site. A uniform per-call overhead would shift p50 too — so p99-only movement is
evidence for a GC/compilation mechanism rather than a constant tax.)

---

### Q2 — "How does an APM agent add tracing to code we never wrote and never annotated?"

**Mid-level answer:** "It uses reflection and proxies to wrap the framework's classes, or it
hooks into Spring's lifecycle."

**Senior answer:** "It rewrites bytecode. Concretely: the agent jar declares a
`Premain-Class` in its manifest, and `-javaagent` makes the JVM call that class's `premain`
method before `main`, handing it an `Instrumentation` object. The agent registers a
`ClassFileTransformer`. From that point on, **every class the JVM is about to define is
passed to the transformer as a raw `byte[]`, before definition, and the transformer may
return different bytes.** Most agents use ByteBuddy on top of that, in `Advice` mode, which
inlines the instrumentation bytecode directly into the target method — so there is no wrapper
object and no extra frame.

The important consequence is what that reaches. A Spring proxy is a *different object* that
wraps yours, so it is bypassed by self-invocation and cannot advise `private`, `static` or
`final` methods. An agent rewrites the method *body*, so it reaches all of those, plus
third-party library internals, plus JDK classes if it puts its helper types on the bootstrap
classpath. That is why the APM can trace the JDBC driver and the connection pool without
anyone wrapping them as beans.

The limits are worth knowing too. `retransformClasses` on an already-loaded class may change
method bodies but may **not** add fields or methods or change the hierarchy — so an agent
attached dynamically to a running JVM can be strictly less capable than the same agent passed
at launch. That is why the vendor docs say to attach at startup, and it is why I would never
compare a dynamically-attached run to a launch-time baseline."

**What separates them:** the mid answer describes a mechanism Java does have (proxies) but
that is the *wrong* one here. The senior answer names the exact API, the exact lifecycle
hook, and — decisively — explains **what the mechanism can reach that the other one cannot**,
plus the retransformation limits that explain vendor guidance.

**Follow-up the interviewer asks:** "If it rewrites the class, why doesn't
`getClass().getName()` show anything unusual?" (Because the class name is part of the class
file and the transformer does not change it. Unlike a CGLIB proxy, which is a generated
*subclass* with a generated name, an agent redefines the same class. The absence of a
visible marker is exactly why this is hard to notice in production.)

---

### Q3 — "Our agent isn't instrumenting anything. Where do you start?"

**Mid-level answer:** "Check the `-javaagent` path is right and the jar is in the image.
Then turn up the agent's log level."

**Senior answer:** "Both of those, and then the specific thing about this API that makes it
fail silently: **if a `ClassFileTransformer` throws, the JVM ignores the exception and
defines the class from the original bytes.** No log line, no warning, the application starts
and works. A completely broken agent is indistinguishable from a working one from the
outside. So the first move is to make the failure visible rather than to guess at causes.

My order would be:

One, confirm the agent's `premain` ran at all — a `println` or the agent's own startup
banner. If it did not, the problem is the flag, the path, or the manifest's `Premain-Class`
attribute.

Two, confirm the transformer is being *called*: log `className` for everything. If it is
called but never matches, check the class name form — `transform` receives **internal form
with slashes**, not dotted names, and matching on the dotted name silently never matches.
That is the single most common cause.

Three, wrap the transformer body in `catch (Throwable)` and log. For a third-party agent,
turn on its listener/debug output — ByteBuddy's `AgentBuilder` has a listener that reports
transformation errors, and it is off by default.

Four, verify what was actually defined rather than trusting anyone: dump the transformed
bytes from the transformer and run `javap -c` on them. If the advice is not in the bytecode,
the transform did not happen; if it is, the problem is downstream.

Five, check ordering and classloaders — whether another agent is running first, and whether
the classes we want are loaded by a classloader the agent's matcher excludes. Framework
classloaders in application servers and in some Spring Boot launcher configurations are a
frequent cause.

And a timing check: were the target classes already loaded before the agent registered? If
we attached dynamically, they were, and nothing gets instrumented without an explicit
retransformation."

**What separates them:** the mid answer debugs the deployment. The senior answer knows the
API's **specified silent-failure behaviour**, which reframes the whole problem from "find the
bug" to "install visibility first", and produces a ranked procedure whose first step is to
make the system able to tell you what is wrong.

**Follow-up the interviewer asks:** "You said it fails silently by design. Why would the
spec choose that?" (Because a bug in an optional observability agent must never prevent an
application from starting. It is a deliberate availability-over-diagnosability trade, and the
cost of that trade is exactly the debugging pain we just discussed.)

---

### Q4 — "Should we run our profiler and our APM agent at the same time in production?"

**Mid-level answer:** "It would use more resources, but it should work. I'd try it in
staging first."

**Senior answer:** "It can work and it is sometimes the right call, but it is a decision with
specific mechanics, not a resource question.

`-javaagent` flags are processed in **command-line order**, and transformers are invoked in
registration order — so the second agent receives the bytes the first one produced. That has
three concrete consequences. The second agent may instrument the first agent's injected code,
producing frames or spans that describe the instrumentation rather than the application. The
combined bytecode growth is additive, so the inlining-threshold risk we discussed compounds.
And if either agent is not fully shaded, its dependencies land on the application classpath
and can produce a `NoSuchMethodError` from a library we never declared — a resolution problem
at runtime, with no build to warn us.

Practically: I would pin the order explicitly, assemble the JVM flags in exactly one place
rather than concatenating `JAVA_TOOL_OPTIONS` from a ConfigMap and a Dockerfile, and add each
agent's package prefix to the other's exclusion list. Then I would A/B against the recorded
load-test baseline with each agent alone and both together, because 'both together' is not
guaranteed to cost the sum of the parts.

There is also a cheaper answer worth putting on the table. **JFR is built into the JVM**, has
low enough overhead to leave on continuously, and is not a second bytecode-rewriting agent —
so 'APM agent plus JFR' is a materially safer combination than 'APM agent plus a second
instrumenting profiler'. For continuous production profiling I would start there, and reserve
async-profiler for a bounded, deliberate investigation window."

**What separates them:** the mid answer treats agents as independent plugins. The senior
answer knows they form an **ordered pipeline over the same bytes**, names the three concrete
failure modes, and then proposes an alternative that avoids the problem class entirely.

**Follow-up the interviewer asks:** "You said reordering can change behaviour. Give me a
concrete example." (Agent A adds a span around every `@Transactional` method. Agent B matches
on method size or on a specific bytecode shape. Run A first and B sees the enlarged method
and may skip it; run B first and it sees the original. Same flags, different set, different
instrumentation.)

---

### Q5 — "How much does instrumentation cost us? Give me a number."

**Mid-level answer:** "Agents typically add a few percent overhead. I can benchmark the
instrumented method with a timing loop and tell you the per-call cost."

**Senior answer:** "Per-call cost is the wrong number, and a timing loop cannot produce it
anyway — with the agent attached, the `System.nanoTime()` calls in the loop sit next to the
injected advice's own timing calls, so you are partly measuring the ruler, and the two arms
warm up at different rates because the agent's classes need compiling too.

The number the business cares about is **end-to-end p99 on a real endpoint at the recorded
load**, because that is where the four separate costs compose: the advice's own work, the
allocation of span and scope objects, the second-order allocation from lost inlining, and the
exporter's batching. A microbenchmark measures the first and silently omits the other three.

So I would give you a table rather than a number, from three arms of the recorded Topic 65
scenario — no agent, agent with exporter off, agent with exporter on — with rows for startup
time split into pre-Spring and Spring, loaded class count, metaspace and code-cache committed
from NMT, p50/p95/p99 per endpoint, throughput at fixed arrival rate, and young-collection
frequency. Each row is a cost with a mechanism attached, which means each one has a different
possible fix.

And I would put the benefit column next to it: what did we actually resolve faster because we
had traces? That comparison is the decision, and it belongs in the capacity and cost model
rather than in a performance argument."

**What separates them:** the mid answer offers a plausible number produced by a broken
method. The senior answer **rejects the question's framing with a specific technical reason**,
substitutes a measurable one, names the experimental design, and puts the benefit on the same
page as the cost.

**Follow-up the interviewer asks:** "If I insisted on a per-call number, how would you get
one honestly?" (JMH, two separate JVM forks because the agent is a JVM-level configuration
and cannot be toggled inside a fork, `@Fork(3)` minimum so profile pollution shows as
variance, `Blackhole.consume` on the result, and an explicit statement that the number
excludes GC and exporter effects — which are most of the real cost.)

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A Spring CGLIB proxy is bypassed by `this.method()`. An agent's advice is not. Derive
   both facts from **where each mechanism puts its code**, without recalling either from the
   text. Then name one thing a proxy can do that an agent cannot.

2. `retransformClasses` may change method bodies but may not add fields. Construct the
   argument for why the JVM must forbid adding a field to a class that already has live
   instances. What would break, concretely, and where?

3. An agent's transformer runs for every class definition. Your application defines a few
   thousand classes. Estimate — in mechanism, not in milliseconds — where the per-class cost
   actually goes, and name the single cheapest filter an agent could apply to reduce it by
   an order of magnitude.

4. Adding advice to a method makes it larger, which can stop it being inlined. Construct the
   **opposite** case: a plausible scenario in which agent instrumentation makes a program
   *faster*. (There is more than one answer; at least one involves the compiler making a
   different but better decision, and at least one involves the agent replacing something
   slower.)

5. Transformed classes cannot be served from a CDS archive. Topic 122 will tell you AppCDS
   is a cheap startup win. Reconcile those two statements into a single recommendation for
   `orderflow`, and name the measurement that decides it.

6. An agent generates classes at runtime, into its own classloader. Topic 79 taught you that
   a Java leak is unintended reachability. Describe the exact shape of an agent-induced
   metaspace leak: what holds what, why the GC cannot reclaim it, and which single `jcmd`
   command would show it growing.

7. You have three agents you would like to run: an APM, a security agent that blocks
   dangerous reflective calls, and a coverage agent. Order them on the command line and
   defend the order. Then name the observation that would prove your ordering wrong.

---

## Quick reference card

### JVM flags

| Flag | What it does | When to set it |
|---|---|---|
| `-javaagent:<jar>[=<args>]` | loads an agent at startup; calls `premain` before `main` | the **only** way to get full instrumentation |
| `-XX:+EnableDynamicAgentLoading` | permits dynamic attach without the JEP 451 warning | only when you must attach at runtime; verify the default on your JDK |
| `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` | prints every inlining decision with its reason | **the tool of this document.** Diff between arms. |
| `-XX:-BackgroundCompilation` | makes compilation synchronous | when you need `PrintInlining` output to be reproducible |
| `-XX:+PrintCompilation` | prints each method as it is compiled | volume comparison between arms; deopt hunting (Topic 74) |
| `-XX:MaxInlineSize=<n>` | size limit for inlining a **cold** callee | almost never; understand it, do not tune it blind |
| `-XX:FreqInlineSize=<n>` | size limit for inlining a **hot** callee | last resort, with `PrintInlining` evidence and a baseline re-run |
| `-XX:MaxInlineLevel=<n>` | inlining tree depth limit | rarely; deep call chains only |
| `-Xlog:class+load=info` | logs every class definition and its source | class-count and CDS-effectiveness comparisons |
| `-Xlog:class+unload=info` | logs class unloading | classloader-leak investigation |
| `-XX:NativeMemoryTracking=summary` | per-category native accounting | **always** when measuring an agent; metaspace and code cache are native (Topic 80) |
| `-XX:MaxMetaspaceSize=<n>m` | bounds metaspace | always in a container; agents grow it |
| `-XX:ReservedCodeCacheSize=<n>m` | bounds the JIT code cache | when NMT shows `Code` growing with the agent |
| `-XX:+PrintFlagsFinal` | dumps every flag's value and origin | settling any default in this document |
| `-Dotel.traces.exporter=none` | agent instruments but does not export | **splits instrumentation cost from export cost.** Verify the property for your agent. |

### Diagnostic commands

| Command | What it answers |
|---|---|
| `jcmd <pid> VM.flags -all` | every flag's effective value and origin, on the live process |
| `jcmd <pid> VM.command_line` | the exact command line, including every `-javaagent` and their order |
| `jcmd <pid> VM.classloader_stats` | classes per classloader — finds the agent's generated-class loader |
| `jcmd <pid> VM.class_hierarchy` | the loaded class hierarchy; confirms generated types exist |
| `jcmd <pid> GC.class_histogram` | instance counts by class; agent types show up here |
| `jcmd <pid> VM.native_memory summary` | metaspace and code cache committed, right now |
| `jcmd <pid> VM.native_memory summary.diff` | **what changed** since a baseline — the incident-solving command |
| `javap -c -p <file>.class` | the bytecode actually present, original or dumped-transformed |
| `javap -v <file>.class \| grep -A3 "Code:"` | the `Code` attribute size — the inlining-threshold input |
| `unzip -l agent.jar \| head -40` | is the agent shaded, and does it carry nested bootstrap jars |
| `unzip -p agent.jar META-INF/MANIFEST.MF` | `Premain-Class`, `Agent-Class`, `Can-Retransform-Classes`, `Boot-Class-Path` |
| `curl -s localhost:8080/actuator/startup \| jq ...` | per-bean startup durations (needs `BufferingApplicationStartup`) |
| `java -XX:+PrintFlagsFinal -version \| grep -i inline` | the inlining thresholds on **your** JVM |
| `java -XX:+PrintFlagsFinal -version \| grep -i EnableDynamicAgentLoading` | your JDK's dynamic-attach posture |

### The agent manifest

```
Premain-Class:            com.orderflow.agent.TimingAgent    # -javaagent entry point
Agent-Class:              com.orderflow.agent.TimingAgent    # dynamic-attach entry point
Launcher-Agent-Class:     com.orderflow.agent.TimingAgent    # executable-jar self-agent (9+)
Can-Redefine-Classes:     true
Can-Retransform-Classes:  true
Can-Set-Native-Method-Prefix: true
Boot-Class-Path:          agent-bootstrap.jar
```

### The API surface worth memorising

```java
// entry points
public static void premain(String agentArgs, Instrumentation inst)
public static void agentmain(String agentArgs, Instrumentation inst)

// the transformer
byte[] transform(Module m, ClassLoader l, String internalName,
                 Class<?> beingRedefined, ProtectionDomain pd, byte[] bytes)
// return null == no change (fast path); non-null == redefine from these bytes
// a THROWN exception is SILENTLY IGNORED and the original bytes are used

// the handle
inst.addTransformer(t, /* canRetransform */ true);
inst.retransformClasses(SomeClass.class);        // bodies only: no new fields/methods
inst.getAllLoadedClasses();
inst.getObjectSize(obj);                          // Topic 69
inst.appendToBootstrapClassLoaderSearch(jarFile); // needed to instrument java.*
```

### Gotchas checklist

- [ ] `className` in `transform` is **internal form** (slashes) and can be `null`.
- [ ] A thrown exception in `transform` is **silently swallowed**; always catch and log.
- [ ] Return `null` for "no change" — returning the same array forces the slow path.
- [ ] `premain` runs **before** `main` and is pure startup latency, on the main thread.
- [ ] Agent order is **command-line order**; agent 2 sees agent 1's bytes.
- [ ] `retransformClasses` cannot add fields or methods or change the hierarchy.
- [ ] Dynamic attach ≠ `-javaagent`: less coverage, possibly a different strategy, and a
      JEP 451 warning on 21+.
- [ ] Agents rewrite the method body, so **self-invocation is instrumented** — unlike a
      Spring proxy (Topic 40).
- [ ] Agent classes go in **metaspace**, which is native and in Topic 80's RSS equation.
- [ ] Generated classes that are never unloaded are a **metaspace leak**; watch
      `jvm_classes_loaded_classes`.
- [ ] Bigger methods can fall out of inlining range — diff `PrintInlining`, do not assume.
- [ ] A transformed class **cannot** be served from a CDS/AppCDS archive.
- [ ] An unshaded agent puts its dependencies on your classpath (Topic 32, at runtime).
- [ ] To instrument `java.*`, helper classes must be on the **bootstrap** classpath.
- [ ] Compare arms against the **recorded Topic 65 baseline**, not against each other's
      vibes.

---

## When would I use this at work?

**1. The p99 regression with an empty diff.**
The load-test gate fails. The application diff for that week contains a copy change. Instead
of bisecting commits that cannot possibly be the cause, you ask "what changed about the
runtime?", find a `-javaagent` in a Helm value, A/B it against the recorded baseline with the
exporter disabled, and diff `PrintInlining`. You arrive at "`InventoryService.reserve`
stopped being inlined because the advice grew it past the threshold" in an afternoon instead
of a fortnight, and — this is the part that matters — you arrive with evidence, so the
conversation about whether to keep tracing is a trade-off discussion rather than an argument.

**2. Reviewing a proposal to add a second agent.**
Someone wants a security agent alongside the APM. You ask four questions: what order will
they be in and who guarantees it, is each one shaded, what is each one's exclusion list for
the other's packages, and what does the combined configuration do to the recorded baseline?
None of those questions are hostile, all of them are answerable, and asking them before the
rollout prevents an incident whose root cause would otherwise take a week to find because
"both agents work fine individually" is true and misleading.

**3. Building a diagnostic nobody else has.**
An agent is roughly a hundred lines and it can answer questions no other tool will. Which
classes load, in what order, from which loader, and how large are they. Which of our methods
are large enough to be at inlining risk. Which classes are being generated at runtime and by
whom. You do not ship it — you keep it in a scratch repo and reach for it when a mystery has
the shape of "something is happening at class-load time". Being the person who can write that
in an hour, rather than waiting for a vendor tool to expose the number, is a genuine
capability difference — and it feeds straight into Topic 122's startup work and Topic 129's
cost model.

---

## Connected topics

**Prerequisites:**

- **40 — proxying, JDK vs CGLIB, and the self-invocation trap.** The other mechanism. An
  agent rewrites the method; a proxy wraps the object. Everything confusing about "why did
  my annotation not fire" resolves once you know which of the two is in play, and this
  document's Proof 7 puts them side by side on the same call.
- **65 — the load-test gate.** Every measurement here is a delta against the recorded
  p50/p95/p99 and the recorded container limits. "The agent made it slower" is not a finding
  without a baseline; "p99 moved 18% against the recorded baseline, outside the ±10% gate"
  is.
- **67 — class loading.** Agents live entirely inside the class-definition path. Lazy
  loading, delegation, and which loader defines what are the mechanics an agent operates on,
  and the reason `premain` cannot touch application classes.
- **68 — memory areas.** The agent's own classes and its generated classes go in
  **metaspace**, which is native, outside the heap, and unbounded by default.
- **74 — tiered compilation, profiling, deoptimization.** The compiler decides based on a
  profile of code it has seen run. An agent changes the code it sees, so it can change the
  decision — and warm-up state is why a naive before/after comparison is meaningless.
- **75 — inlining, escape analysis, scalar replacement.** **The direct mechanism of this
  document's headline trap.** Inlining is size-gated; instrumentation increases size;
  escape analysis depends on inlining having happened. `-XX:+PrintInlining` is the shared
  instrument.
- **76 — reading bytecode with `javap -c`.** How you prove an agent did what you think. Dump
  the transformed bytes and read them. The `Code` attribute size is the number that decides
  the inlining question.
- **77 — JMH.** Any per-call claim about instrumentation overhead must come from a proper
  harness, in separate forks, because the agent is a JVM-level configuration.
- **78 — profiling and flame graphs.** The allocation profile is how you confirm that lost
  inlining actually produced allocations. Also: async-profiler is itself an agent, which is
  why "profile it to find out why the profiler slowed it down" is circular.
- **79 — memory leaks and heap dumps.** An agent that generates classes into a loader that
  is never released is a metaspace leak with exactly the reachability shape you learned
  there.
- **80 — off-heap memory and NMT.** The direct predecessor. Metaspace and the code cache are
  two terms in that document's RSS equation, and this document is the story of a change that
  grows both. `summary.diff` is the shared instrument.

**This unlocks:**

- **82 — JVM tuning and container awareness.** The agent's metaspace and code-cache
  footprint has to fit inside a container budget, and the next document is where you build
  that budget properly — including the CPU-count cascade that decides how many compiler
  threads are available to do the extra compilation an agent creates.
- **83 — GraalVM native-image.** Native image performs **closed-world** analysis at build
  time, which means a runtime bytecode-rewriting agent has nowhere to stand. Everything in
  this document stops being possible. That is one of the concrete, under-discussed costs of
  going native, and it is a large one for a team that depends on APM auto-instrumentation.
- **101 — virtual threads.** Agents instrument thread-related JDK classes, and advice that
  holds a monitor or blocks can interact with pinning. Context propagation across a
  virtual-thread boundary is agent territory too.
- **119 — OpenTelemetry and context propagation.** The consumer of this document. The agent
  is *how* auto-instrumentation happens; Topic 119 is *what it produces* and where it drops
  context — every one of those drops is a boundary the transformer's matchers did not cover.
- **122 — layered jars, AppCDS and startup.** The direct collision. AppCDS is the cheap
  startup win; a transforming agent partially defeats it, and the startup decomposition in
  this document's Measurement section is the shared technique.
- **129 — capacity, cost and latency budgets.** The measured cost table from this document's
  Example 2 is a direct input. "What does observability cost us per request, and what does it
  buy" is a capacity-model line, not a performance opinion.

---

*Java baseline 21, running on JDK 25. Four things in this document are deliberately hedged
rather than asserted: the exact defaults of `MaxInlineSize`, `FreqInlineSize` and
`MaxInlineLevel` on your JDK build; the precise ordering guarantees between
retransformation-capable and retransformation-incapable transformers; your JDK's current
default for `EnableDynamicAgentLoading` under JEP 451; and the property names your specific
APM agent uses to narrow its matchers or disable its exporter. Each has a command or a
five-minute experiment attached that settles it on your machine. Everything else — that a
`ClassFileTransformer` receives the raw bytes of every class before it is defined and may
return different bytes, that a thrown exception in a transformer is silently ignored, that
retransformation cannot add fields, that agent order is command-line order, and that
rewriting a method makes it bigger and therefore a different candidate for inlining — is
specified behaviour and will still be true the next time a p99 moves on a release with an
empty diff.*
