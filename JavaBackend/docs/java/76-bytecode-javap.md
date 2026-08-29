# 76 — Reading Bytecode with `javap -c`

## Phase: 8 — JVM Internals
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: settles "what did the compiler actually emit" arguments about `orderflow` code — boxing in the pricing path, string building in the reconciliation export, bridge methods on the `PriceRule` hierarchy, and whether a lambda allocated. It diagnoses nothing about runtime; that is Topics 77–79.

---

## R0 — READ THIS BEFORE ANY OTHER LINE IN THIS DOCUMENT

**I do not have a JVM. Nothing in this document is captured `javap` output, and I will
never present anything as if it were.**

What I give you instead, everywhere:

- the **exact command with exact flags**,
- **what to look for** in the result,
- a **"what you see" → "what it means"** table covering the plausible outcomes.

I will name specific bytecode instructions — `invokestatic`, `invokevirtual`,
`invokeinterface`, `invokespecial`, `invokedynamic`, `checkcast`, `getfield`,
`aload_0`, `ldc` and their close relatives — and tell you what each one does. I am
confident about those. I will tell you the **order** to expect them in. I will not
paste a disassembly listing and call it real.

> ### THE RULE, and it is absolute:
>
> **If your `javap` listing differs from my description, YOUR LISTING IS THE TRUTH.**
>
> Not mine. Not a blog post's. Not Stack Overflow's. The disassembly of the class file
> your `javac` produced on your machine is the only authority in this topic, and the
> whole point of learning `javap` is that you stop needing to trust anyone — including
> me — about what the compiler did.
>
> When your output disagrees with this document, do not "fix" your output. Read it,
> and if you want, paste it and I will read it with you.

There are exactly three places where I state something as a plain fact rather than as
"go look":

1. **The JVM is a stack machine.** Operands are pushed onto and popped off an operand
   stack; there are no named general-purpose registers in the bytecode model.
2. **There are five invoke instructions**, and both **lambdas** and **string
   concatenation with `+`** compile to the fifth, `invokedynamic`.
3. **`javac` performs almost no optimisation.**

Each of those is spec-level, each has a command in this document that verifies it on
your machine, and you should run that command rather than believe me.

---

## Mechanical statement

Read this twice. Everything else elaborates it.

> **The JVM is a stack machine. Operands are pushed and popped, not held in named
> registers.** Every method gets two things when it runs: an **operand stack** (a
> scratch pad — instructions consume their inputs from the top and push their result
> back) and a **local variable array** (numbered slots — `this` is slot 0 in an
> instance method, then the parameters, then your locals).
>
> **There are exactly five invoke instructions:** `invokestatic`, `invokespecial`,
> `invokevirtual`, `invokeinterface`, `invokedynamic`. The first four resolve their
> target eagerly, by name and descriptor, at first execution of the call site. The
> fifth resolves it by **running a bootstrap method** the first time the instruction
> executes; that bootstrap returns a `CallSite`, the call site is linked to it, and
> every subsequent execution goes straight through the linked target.
>
> **Both lambdas and string concatenation compile to `invokedynamic`.**
>
> And: **`javac` does almost nothing.** It desugars the language into the JVM's
> instruction set and stops. It does not inline, does not eliminate dead code, does
> not hoist loop invariants, does not do escape analysis. All of that happens later,
> at runtime, in C1 and C2.
>
> Therefore: **`javap -c` tells you what the LANGUAGE did. It tells you nothing about
> what RUNS.** Two different questions. Two different tools.

That last paragraph is the main idea of this document. If you take one thing from
Topic 76, take the sentence "two different questions, two different tools", because
the single most common way engineers misuse `javap` is to answer a performance
question with it.

---

## The bridge from what you know

### There is NO TYPESCRIPT ANALOGUE for this. Say it out loud.

You have never, as a routine debugging step, disassembled compiled output to
understand what your language did. There is nothing in your toolchain that maps onto
`javap -c`, and the near-misses are near-misses in ways that matter.

Let me kill the three candidates you will reach for.

**Candidate 1: reading `tsc` output.**

```ts
const totals = lines.map(l => l.unitPriceMinor * l.quantity);
```

```js
// tsc --target es2022 output — near-identical
const totals = lines.map(l => l.unitPriceMinor * l.quantity);
```

TypeScript's compiler output is *the same language you wrote*, minus types. Down-
levelling to ES5 does real desugaring (`async`/`await` into a state machine,
generators, class fields), and reading that output is genuinely instructive — but it
is still JavaScript. You read it to understand a transform. You have never had to
read it to understand what an operation **costs in the machine's own instruction
set**, because there is no such stable instruction set to read.

**Verdict: NOT an analogue.** Same activity in spirit, completely different depth.

**Candidate 2: `node --print-bytecode` / V8's Ignition bytecode.**

This is closer, and it is worth knowing it exists. V8 compiles JS to Ignition
bytecode, which is also a register machine (not a stack machine — a real difference).

But you have never used it, and here is the honest reason: **it is not stable, not
specified, and not a contract.** V8's bytecode changes between Chrome versions
without notice. There is no `--print-bytecode` output you would put in a pull request
comment as evidence, because the next V8 release may make it wrong.

Java's bytecode is the opposite: it is **specified** by the JVM Specification, it is
**the distribution format** — the `.class` file you ship *is* the bytecode — and it is
binary-compatible across decades. A `.class` file compiled in 2005 still runs on JDK
25. That stability is what makes reading it a legitimate engineering technique rather
than a curiosity.

**Verdict: NOT an analogue.** The mechanism rhymes; the stability and the status do
not.

**Candidate 3: source maps.**

Source maps map generated code back to source. `javap -l` shows a `LineNumberTable`
that does the same job. But you use source maps *passively* — the debugger reads
them, you never read the mapping yourself. Nobody opens a `.map` file.

**Verdict: NOT an analogue.**

### The honest statement of the gap

Here is the thing to internalise, because it explains why this skill exists in Java
and does not exist in your current stack:

| | TypeScript / Node | Java |
|---|---|---|
| What you ship | source, or source-shaped JS | **the compiled bytecode itself** |
| Is the compiled form specified? | no | **yes, by the JVM Specification** |
| Does the compiler optimise? | minimally (tsc), heavily (bundlers/terser) | **almost not at all** |
| Where does optimisation happen? | build time (terser) + runtime (V8) | **entirely at runtime (C1/C2)** |
| Can you read the compiled form as a contract? | no | **yes** |
| Do you, routinely? | no | occasionally, and precisely |

The fourth row is the load-bearing one. In your world, if you want to know what a
construct costs, you look at the bundled output *and* you know the bundler already
optimised it, so the output is close to what runs. In Java those are **completely
separate stages**: `javac` desugars and stops; C2 optimises, at runtime, based on a
profile that does not exist yet at compile time.

So Java gives you a clean split you have never had:

- **What did the language do to my code?** → `javap -c`. Deterministic, reproducible,
  the same on every machine with the same `javac`.
- **What actually executes at speed?** → `-XX:+PrintCompilation`, `-XX:+PrintInlining`,
  a profiler (Topic 78), a JMH benchmark (Topic 77). Non-deterministic, workload-
  dependent, different on every run.

Using the first tool to answer the second question is the trap this whole document is
built around, and it is Trap 1 below.

### The one thing that does transfer

You already know that **a language feature is a transform, not a primitive.** You know
`async`/`await` becomes a state machine, that `for...of` becomes an iterator protocol
call, that optional chaining becomes a series of null checks, that a class field
becomes an assignment in the constructor. You already think "what does this desugar
to?"

**Keep that instinct exactly as it is.** `javap -c` is the tool that answers it in
Java, and the answers are more surprising than TypeScript's because Java's syntax
hides more: `+` on strings, `for (X x : xs)`, autoboxing, generics, try-with-resources,
`switch` on strings, varargs, and lambdas are all desugarings you cannot see in source.

**Verdict on the instinct: HONEST ANALOGUE. Verdict on the tooling: NO ANALOGUE.**

---

## What is this?

`javap` is the **class file disassembler** shipped with every JDK. It reads a
`.class` file and prints what is inside it in human-readable form.

You already used it once, in Topic 01, to see `Integer.valueOf` and `Integer.intValue`
appear in code where you wrote neither. This topic is that skill, properly.

### The three levels of `javap`

```bash
javap            OrderService.class     # public API only — signatures, no bodies
javap -p         OrderService.class     # + private and package-private members
javap -c -p      OrderService.class     # + the disassembled bytecode of every method
javap -v -p      OrderService.class     # + constant pool, attributes, flags, stack map
```

You will live in `javap -c -p`. You reach for `-v` when you need the constant pool or
the `BootstrapMethods` attribute — which is exactly when you are chasing an
`invokedynamic`.

### What a `.class` file actually contains

A `.class` file is a binary structure with a fixed layout:

```
magic number 0xCAFEBABE
minor / major version        <- which javac target produced it
constant pool                <- ALL names, strings, numbers, and symbolic references
access flags                 <- public, final, super, interface, abstract, synthetic...
this class / super class     <- indices into the constant pool
interfaces
fields[]                     <- name, descriptor, flags, attributes
methods[]                    <- name, descriptor, flags, and a Code attribute
attributes[]                 <- SourceFile, BootstrapMethods, InnerClasses, Signature,
                                RuntimeVisibleAnnotations, NestHost, PermittedSubclasses,
                                Record, ...
```

Three of those deserve immediate attention because they explain behaviour you have
already met in earlier topics:

- **The constant pool** is where every name lives. There are no inline strings in
  bytecode; instructions reference pool entries by index (`#7`, `#23`). This is why
  `javap -c` output is full of `#` numbers with a `// ...` comment resolving them.
- **`RuntimeVisibleAnnotations`** is where `@Transactional` physically lives. Topic 40
  told you an annotation is inert metadata that something else must read reflectively.
  Now you can *see* it: it is a run of bytes in an attribute. It has no code.
- **`Signature`** is where the generic type information survives erasure. Topic 06
  told you `List<String>` erases to `List`. It does — in the *descriptor*. The full
  generic signature is retained in a separate attribute for reflection and for the
  compiler when it reads your class as a library. Erasure is about what the **verifier
  and the runtime** see, not about total deletion.

### The instruction categories you need

You do not need all ~200 opcodes. You need these families:

| Family | Examples | What they do |
|---|---|---|
| **Load a local onto the stack** | `aload_0`, `aload_1`, `iload_2`, `lload`, `dload` | push local variable slot N. `a`=reference, `i`=int, `l`=long, `d`=double, `f`=float |
| **Store from the stack into a local** | `astore_1`, `istore_2` | pop and write into slot N |
| **Push a constant** | `ldc`, `ldc_w`, `bipush`, `sipush`, `iconst_0`..`iconst_5`, `aconst_null` | push a literal. `ldc` pulls from the **constant pool** |
| **Field access** | `getfield`, `putfield`, `getstatic`, `putstatic` | instance / static field read and write |
| **Method invocation** | `invokestatic`, `invokespecial`, `invokevirtual`, `invokeinterface`, `invokedynamic` | the five. See below |
| **Object creation** | `new`, `dup`, `newarray`, `anewarray`, `multianewarray` | `new` allocates *uninitialised*; a constructor call follows |
| **Type operations** | `checkcast`, `instanceof` | the casts erasure forces the compiler to insert |
| **Arithmetic** | `iadd`, `isub`, `imul`, `ladd`, `iinc` | operate on the top of the stack |
| **Control flow** | `goto`, `ifeq`, `ifnull`, `if_icmpge`, `if_acmpne`, `tableswitch`, `lookupswitch` | branches, by offset |
| **Return** | `return`, `ireturn`, `areturn`, `lreturn` | typed returns; `return` is void |
| **Stack shuffling** | `dup`, `pop`, `swap`, `dup_x1` | rearrange the operand stack |
| **Exceptions** | `athrow` + the **exception table** (not an instruction) | `try`/`catch` is a *table*, not a jump |

The prefix letter is the type: `a` reference, `i` int, `l` long, `f` float, `d`
double. Once you see that, half the instruction set stops being cryptic.

### The five invoke instructions — the core of the topic

| Instruction | Used for | How the target is found |
|---|---|---|
| `invokestatic` | `static` methods | by class + name + descriptor. No receiver on the stack. |
| `invokespecial` | constructors (`<init>`), `private` methods, `super.method()` | **non-virtual** — the exact method named is called, no dynamic dispatch |
| `invokevirtual` | ordinary instance methods on a **class** type | virtual dispatch on the receiver's runtime class |
| `invokeinterface` | instance methods on an **interface** type | dispatch through the receiver's interface method table |
| `invokedynamic` | **lambdas, method references, string concat, record `equals`/`hashCode`/`toString`, pattern-matching switch** | a **bootstrap method** runs once, returns a `CallSite`; the site is then linked |

Two immediate payoffs from that table:

1. **`invokespecial` for `private` methods explains Topic 40.** A `private` method is
   not virtually dispatched, which is one of the two independent reasons a
   `private @Transactional` method can never be advised by a CGLIB subclass.
2. **`invokeinterface` vs `invokevirtual` is decided by the STATIC type at the call
   site**, not the runtime type. `List<Order> l = new ArrayList<>(); l.size();` emits
   `invokeinterface`. `ArrayList<Order> l = new ArrayList<>(); l.size();` emits
   `invokevirtual`. Same object, different instruction. This is one of the small facts
   that makes bytecode reading click.

---

## Why does it matter?

**1. It ends arguments with evidence instead of authority.**

"Does `+` in a loop allocate a `StringBuilder` per iteration?" "Does the enhanced
`for` loop allocate an iterator?" "Does this `Map<Long, Long>` counter box?" "Is that
lambda capturing?" Every one of those is a five-second `javap` question, and every one
of them is routinely answered wrong in code review by people quoting advice from
2011. You will be the person who runs the command.

**2. It makes four earlier topics concrete rather than memorised.**

- **Topic 01, boxing:** you saw `Integer.valueOf`/`intValue` appear. Now you can find
  every boxing site in a method mechanically.
- **Topic 06, erasure and bridge methods:** a bridge method is invisible in source and
  obvious in `javap`. It has `ACC_BRIDGE` and `ACC_SYNTHETIC` flags and its body is
  `checkcast` + delegate. Reading one converts "bridge methods exist" into "I know
  exactly what they do".
- **Topic 18, string concat:** `+` compiles to `invokedynamic` bound to
  `StringConcatFactory`, not to a `StringBuilder` chain, since Java 9. Half the advice
  on the internet predates that.
- **Topic 21, lambdas:** a lambda is an `invokedynamic` linked by `LambdaMetafactory`,
  plus a synthetic method holding the body — **not** an anonymous class. The drill in
  this document proves it with a directory listing.

**3. It is the only honest way to answer "did the compiler do X?"**

`javac`'s behaviour is deterministic and reproducible. Two engineers running `javap -c`
on the same class file get identical output. That property makes bytecode a *fact* in
a design discussion, in a way that a benchmark number never is.

**4. Bytecode size is a real input to a real runtime decision.**

This is the one place where `javap` legitimately touches performance, and it is
indirect: **HotSpot's inlining heuristics use the callee's bytecode size in bytes.**
`javap -v` prints each method's `Code` length. A method that is too large will not be
inlined, and a method that is not inlined blocks escape analysis in its caller
(Topic 75). So bytecode size is an *input*, not a measurement. The Measurement section
below makes that precise, including how to check the actual thresholds on your JVM
rather than trusting numbers from a blog.

**5. It is a senior interview differentiator, in a specific way.**

Not because interviewers ask you to read a listing. Because when they ask "is
`StringBuilder` faster than `+`?", the answer that lands is: "For a single expression,
no — `+` compiles to an `invokedynamic` bound to `StringConcatFactory` since Java 9,
and the JDK's strategy generates a MethodHandle chain that is usually at least as
good as a hand-written builder. Inside a loop it's a different question, because
you get a fresh concatenation per iteration. I'd confirm which shape the compiler
emitted with `javap -c`, and then measure whether it matters with JMH — those are two
different questions." That answer contains a mechanism, a version boundary, a
verification command, and a refusal to conflate the two questions.

---

## Machine-level reality

This section is what makes Topic 76 a differentiator. Everything in it is checkable
with a command, and where I am uncertain I say so in one line and give you the
settling command.

### The operand stack, concretely

Every method invocation creates a **frame** on the calling thread's stack. The frame
holds:

- a **local variable array** — fixed size, computed by `javac`, printed by `javap -v`
  as `locals=N`;
- an **operand stack** — fixed maximum depth, also computed by `javac`, printed as
  `stack=N`;
- a reference to the class's constant pool.

Take `int total = price * quantity;` inside an instance method where `price` is in
slot 1 and `quantity` in slot 2. The instruction sequence to look for is:

```
iload_1          push the int in slot 1        ->  stack: [price]
iload_2          push the int in slot 2        ->  stack: [price, quantity]
imul             pop two, multiply, push       ->  stack: [product]
istore_3         pop and store into slot 3     ->  stack: []
```

*(That block is what to look for, not captured output.)*

Four instructions, no register names anywhere. That is the stack machine.

**Why does this design exist?** Because a stack machine's instruction encoding does
not need to name registers, so the instruction stream is compact and — critically —
**portable across CPU architectures with completely different register files.** The
`.class` file was designed to be shipped over a 1995 network connection and run
anywhere. The JIT's job at runtime is precisely to undo this: map the operand stack
back onto real machine registers. So the stack machine is a *transport encoding*, not
a description of how the code executes.

That single sentence is worth holding on to, because it is the deep reason `javap -c`
tells you nothing about runtime cost. **The stack you are reading does not exist at
runtime in compiled code.** C2 has already turned it into registers.

### Local variable slots — and the `this` rule

Slots are numbered from 0:

- **In an instance method, slot 0 is `this`.** That is why you see `aload_0` at the
  start of almost every instance method: it is pushing `this` so a `getfield` or an
  `invokevirtual` can consume it.
- **In a `static` method, slot 0 is the first parameter.** No `this`.
- Parameters occupy the next slots in declaration order.
- `long` and `double` take **two** slots each. So a method `f(long a, int b)` has `a`
  in slots 0–1 and `b` in slot 2 (static) or 1–2 and 3 (instance).
- Local variables are assigned slots after the parameters, and **slots are reused**
  once a variable goes out of scope.

> `aload_0` at the top of a method body is the single most common instruction in Java
> bytecode, and it means "load `this`". Learn to read past it.

**Do local variable *names* survive?** Only if compiled with `-g` or
`-g:vars`. Maven's compiler plugin enables debug info by default, so in a normal build
they do. `javap -l` shows the `LocalVariableTable` and `LineNumberTable`. If you see
slot numbers with no names, the class was compiled without debug info — that is Trap 4.

### The constant pool

Bytecode never contains a class name, method name, string literal, or large number
inline. It contains an **index** into the constant pool, which holds typed entries:

| Entry kind | Holds |
|---|---|
| `CONSTANT_Utf8` | the raw characters of every name and descriptor |
| `CONSTANT_Class` | an index to a Utf8 holding a class's internal name |
| `CONSTANT_NameAndType` | a name index + a descriptor index |
| `CONSTANT_Fieldref` / `Methodref` / `InterfaceMethodref` | class index + NameAndType index |
| `CONSTANT_String` | an index to a Utf8 — what `ldc` pushes for a string literal |
| `CONSTANT_Integer` / `Long` / `Float` / `Double` | numeric literals too large for `bipush`/`sipush` |
| `CONSTANT_MethodHandle` / `MethodType` | the raw material for `invokedynamic` |
| `CONSTANT_InvokeDynamic` | a **bootstrap-method index** + NameAndType |
| `CONSTANT_Dynamic` | a lazily computed *constant* (`condy`) — used by, among others, some record and pattern-matching machinery |

Run `javap -v` and you will see the whole pool numbered `#1` upward, then every
instruction referencing it.

**Two things follow immediately, and both are practically useful:**

1. **String literals are `ldc` of a `CONSTANT_String` entry, and identical literals in
   the same class share one pool entry.** That is the class-file half of the string
   pool story from Topic 18. The runtime half — `String.intern()` and the JVM's shared
   pool — is what makes literals from *different* classes share too.
2. **`static final` compile-time constants are inlined into every call site.** If
   `Config.MAX_LINES` is `public static final int MAX_LINES = 500;`, then a class using
   it emits `sipush 500`, **not** a `getstatic`. This is a JLS-level rule about
   compile-time constant expressions, and it has a very real consequence: **changing
   the constant and recompiling only that class does not update the callers.** They
   still hold the old value baked into their own constant pools until they are
   recompiled. This is a genuine production incident shape ("we changed the constant
   and deployed, and half the system still used the old value"), and `javap -c` on the
   *caller* is how you prove it in thirty seconds.

   To check: compile a caller, then `javap -c Caller.class | grep -i 500`. If you see
   the literal instead of a `getstatic` to the constant's owner, it was inlined.

### `invokedynamic`: bootstrap and call-site linkage

This is the mechanism behind lambdas, string concatenation, record methods, and
pattern-matching switches. It deserves the detail.

An `invokedynamic` instruction in the code stream carries:

- an index into the class's **`BootstrapMethods` attribute**, which names a **bootstrap
  method** (a static method, referenced as a `MethodHandle`) plus its **static
  arguments**;
- a **name** and a **descriptor** — the "invoked name" and "invoked type".

**First execution of that instruction:**

1. The JVM resolves the bootstrap method handle.
2. It calls it, passing a `MethodHandles.Lookup` (with the caller's access rights),
   the invoked name (a `String`), the invoked type (a `MethodType`), and the static
   arguments from the attribute.
3. The bootstrap returns a **`CallSite`** object holding a `MethodHandle` — the target.
4. The JVM **permanently links the instruction to that call site.**

**Every subsequent execution** jumps straight to the linked target. For a
`ConstantCallSite` — which is what `LambdaMetafactory` returns — the target can never
change, so the JIT can treat it as a constant and inline straight through it.

That last sentence is the entire performance story of `invokedynamic`, and it is the
opposite of most people's intuition. **"Dynamic" describes the linkage, which happens
once. It does not describe the dispatch, which after linking is as static as anything
else.** Someone who reads "invokedynamic" and thinks "reflection, slow" has the model
exactly backwards.

**The two bootstrap methods you will meet constantly:**

| Construct | Bootstrap method | What you will see in `javap -v` |
|---|---|---|
| lambda, method reference | `java.lang.invoke.LambdaMetafactory.metafactory` (or `altMetafactory` for serializable / marker-interface lambdas) | a `BootstrapMethods` entry naming `LambdaMetafactory`, with three static args: the erased method type, a `MethodHandle` to the implementation method, and the instantiated method type |
| `"a" + b + "c"` | `java.lang.invoke.StringConcatFactory.makeConcatWithConstants` | a `BootstrapMethods` entry naming `StringConcatFactory`, with a **recipe string** as a static argument — literal text with placeholder markers for the dynamic arguments |
| record `equals`/`hashCode`/`toString` | `java.lang.runtime.ObjectMethods.bootstrap` | a `BootstrapMethods` entry naming `ObjectMethods`, with the component names and getter handles |
| `switch` with type patterns | `java.lang.runtime.SwitchBootstraps.typeSwitch` | a `BootstrapMethods` entry naming `SwitchBootstraps` |

> **One line of honest uncertainty:** the exact *number and order* of static arguments
> in a `BootstrapMethods` entry, and whether a given `javac` emits `metafactory` or
> `altMetafactory` for a particular lambda, are implementation details that have
> shifted across JDK releases. **Settling command:** `javap -v -p YourClass.class`
> and read the `BootstrapMethods` section verbatim. That output is authoritative for
> your compiler; my description is a guide to what you are looking at.

**Why was `invokedynamic` used for lambdas rather than generating classes?** Because
generating a class per lambda at compile time would have frozen the implementation
strategy into every `.class` file ever shipped. With `invokedynamic`, the class file
says only "link this call site using `LambdaMetafactory`" and the *JDK* decides how —
currently by spinning a hidden class at link time, potentially by something else in
future — without recompiling a single application. That is a deliberate binary-
compatibility decision, and it is the same reason string concatenation moved to indy
in Java 9: the JDK can change the concat strategy for every program on earth by
changing `StringConcatFactory`.

### What `javac` DOES do — the complete short list

I have said `javac` does almost nothing. "Almost" is doing work. Here is the honest
list of transformations it performs, so you know what to attribute to it:

**Desugarings (structural — the language becomes JVM instructions):**

| Source construct | What to look for in `javap -c` |
|---|---|
| autoboxing / unboxing | `invokestatic Integer.valueOf` / `invokevirtual Integer.intValue` |
| enhanced `for` over an `Iterable` | `invokeinterface Iterable.iterator`, then `hasNext`/`next` in a loop, then a `checkcast` |
| enhanced `for` over an array | `arraylength`, an int index local, `aaload`/`iaload` — **no iterator, no allocation** |
| `+` on strings | `invokedynamic makeConcatWithConstants` |
| lambda / method reference | `invokedynamic` + a synthetic `lambda$...` method (for lambdas with a body) |
| generics | `checkcast` at every point a type variable is read; **bridge methods** on the class |
| varargs call | `anewarray` + `dup`/`aastore` per element, then the invoke |
| try-with-resources | nested exception table entries, a `close()` call, and `Throwable.addSuppressed` |
| `switch` on `String` | **two** switches: a `lookupswitch` on `hashCode()`, then `String.equals` to disambiguate, then a `tableswitch` on the resulting index |
| `switch` on `enum` | a synthetic `$SwitchMap$...` int array in a synthetic class, indexed by `ordinal()` |
| inner (non-static) class | a synthetic `this$0` field holding the enclosing instance |
| `assert` | a synthetic `$assertionsDisabled` static boolean guarding the check |
| record | generated accessors, plus `equals`/`hashCode`/`toString` via `invokedynamic` to `ObjectMethods` |
| string switch / pattern switch (21+) | `invokedynamic` to `SwitchBootstraps` for type patterns |

**Actual optimisations (there are basically three):**

1. **Constant folding of compile-time constant expressions.** `2 * 60 * 60` becomes
   `sipush 7200`. `"order-" + "flow"` becomes a single `ldc "orderflow"`. This is not
   an optimisation `javac` chose — the JLS *requires* it, because compile-time
   constants have language semantics (they can be `case` labels, they are inlined into
   callers, they make `if (DEBUG)` blocks legally unreachable).
2. **Inlining of compile-time constants across classes**, as described above.
3. **Removal of code made unreachable by a compile-time-constant condition.** `if
   (false) { ... }` produces no bytecode for the body — this is the mechanism behind
   the classic conditional-compilation idiom.

**That is the list.** No method inlining. No common subexpression elimination. No loop
unrolling. No strength reduction. No dead store elimination. No escape analysis. No
devirtualisation.

**Verify it yourself in ten seconds:** write a `private` method called exactly once
from another method in the same class, compile, and `javap -c -p`. You will still see
an `invokespecial` to it. `javac` did not inline it, even though it trivially could.
C2 will, at runtime, when the caller gets hot.

### And what `javac` does NOT do — where the real optimisation happens

At runtime, HotSpot's tiered compiler transforms this code beyond recognition:

- **inlining** — the enabling optimisation; everything below depends on it,
- **devirtualisation** — a `invokeinterface` that has only ever seen one receiver
  class becomes a direct call with a guard,
- **escape analysis and scalar replacement** — a `StringBuilder` that never escapes can
  become nothing at all; its fields become registers (Topic 75),
- **dead code elimination** — including of your entire benchmark loop (Topic 77),
- **loop unrolling, range-check elimination, vectorisation**,
- **lock elision** on non-escaping objects.

So: an allocation visible in `javap -c` **may not happen at runtime**, and a virtual
call visible in `javap -c` **may compile to a direct jump or to nothing**. This is not
a theoretical caveat. It is the normal case for hot code.

**Which is why the following sentence is the whole document:**

> `javap -c` proves what the compiler **emitted**. It proves nothing about what the
> JVM **executes**. To learn what executes: `-XX:+PrintCompilation`,
> `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`, an allocation profile
> (Topic 78), or a JMH benchmark (Topic 77).

---

## Example 1 — minimal

Two methods, one construct each. The point is to build the habit of predicting before
looking.

`Minimal.java`:

```java
public class Minimal {

    private final long unitPriceMinor;

    public Minimal(long unitPriceMinor) {
        this.unitPriceMinor = unitPriceMinor;
    }

    // (a) plain arithmetic on a field and a parameter
    public long lineTotal(int quantity) {
        return unitPriceMinor * quantity;
    }

    // (b) the same value, but boxed through a wrapper
    public Long lineTotalBoxed(Integer quantity) {
        return unitPriceMinor * quantity;
    }
}
```

```bash
javac Minimal.java
javap -c -p Minimal.class
```

**What to look for in `lineTotal`, in this order:**

1. `aload_0` — push `this`.
2. `getfield` referencing `unitPriceMinor` with descriptor `J` (`J` is the descriptor
   letter for `long`).
3. `iload_1` — push the `int` parameter from slot 1.
4. `i2l` — convert int to long (widening, so the multiply operands match).
5. `lmul` — long multiply.
6. `lreturn`.

Six instructions. No allocation. No method call except the implicit field read.

**What to look for in `lineTotalBoxed` — the same expression, and count the
difference:**

1. `aload_0`, `getfield unitPriceMinor`.
2. `aload_1` — push the `Integer` **reference**.
3. `invokevirtual java/lang/Integer.intValue` — **unboxing**. You wrote no such call.
4. `i2l`, `lmul` — the same arithmetic.
5. `invokestatic java/lang/Long.valueOf` — **boxing** the result into the `Long`
   return type. You wrote no such call either.
6. `areturn` — returning a *reference* now, not a primitive.

**How to read the difference.** You wrote the same expression twice. The second
version contains two method calls you never typed, one of which (`Long.valueOf`) is a
potential **allocation** — a `Long` outside the −128..127 cache is a fresh object
(Topic 01). That is what "autoboxing is convenient" costs, made visible.

| What you see | What it means |
|---|---|
| `Integer.intValue` and `Long.valueOf` in `lineTotalBoxed`, absent from `lineTotal` | Expected. The compiler inserted the conversions. This is the mechanical proof of Topic 01. |
| No `intValue`/`valueOf` anywhere | You disassembled the wrong class, or your source does not match. Re-run `javac`. |
| `Long.valueOf` present in **both** methods | Check the return type of `lineTotal` — if you wrote `Long` rather than `long`, you have boxed it too. |
| `getfield` where you expected `getstatic` | `unitPriceMinor` is an instance field, so `getfield` is correct. `getstatic` would mean you made it `static`. |

**Now the discipline this document is really teaching.** Having seen two extra calls,
what have you learned about performance?

**Nothing.** You have learned what the *compiler emitted*. Whether `Long.valueOf`
allocates at runtime depends on the value; whether the allocation survives depends on
escape analysis; whether any of it matters depends on how often the method runs and
what else the request does. Those are Topics 75, 77 and 78. `javap` has finished its
job at the word "emitted".

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow`, at the Topic 65 baseline:

- **100k products, 1M orders, 5M order lines** in Postgres.
- The k6 mix is 70% catalogue read / 20% order read / 10% order placement, with
  recorded p50/p95/p99 in `/docs/java/baselines/`.
- Separately from the request path, there is a **nightly reconciliation export** that
  streams every order line from the last 24 hours and writes a one-line description
  per row to a file for the finance team. On a peak day that is roughly 400k lines.
- A pull request has arrived. The reviewer's comment is:

  > "`+=` in a loop is a classic Java performance bug — this allocates a new
  > `StringBuilder` on every iteration. Please use a single `StringBuilder`."

You need to decide whether that comment is correct. **`javap -c` answers exactly half
of the question, and this example is about knowing which half.**

### The code under review

```java
package com.orderflow.reconciliation;

import java.util.List;
import java.util.function.Predicate;

public class ReconciliationExporter {

    /** Builds one export line per order line. Called once per order, ~400k/night. */
    public String describe(Order order, List<OrderLine> lines) {

        String out = "";
        for (OrderLine line : lines) {
            out += order.reference() + "," + line.sku() + "," + line.quantity() + "\n";
        }
        return out;
    }

    /** Filters the lines that need manual review. */
    public List<OrderLine> needsReview(List<OrderLine> lines, long thresholdMinor) {
        Predicate<OrderLine> expensive = l -> l.unitPriceMinor() > thresholdMinor;
        return lines.stream().filter(expensive).toList();
    }
}
```

Three separate desugarings are hiding in twelve lines, and `javap` finds all three.

```bash
javac -d out --release 21 $(find src -name '*.java')
javap -c -p -v out/com/orderflow/reconciliation/ReconciliationExporter.class > exporter.txt
less exporter.txt
```

### What to look for — construct 1: the enhanced `for` loop

Inside `describe`, before the loop body:

1. `aload_2` (the `lines` parameter) then `invokeinterface java/util/List.iterator`.
2. `astore` into a fresh slot — the iterator local you never declared.
3. The loop head: `aload` that slot, `invokeinterface java/util/Iterator.hasNext`,
   then an `ifeq` branching past the loop when false.
4. `aload` the iterator, `invokeinterface java/util/Iterator.next`, then a
   **`checkcast` to `OrderLine`** — because `next()` erases to returning `Object`
   (Topic 06), and the compiler must insert the cast you never wrote.
5. `astore` into the `line` slot.

**How to read it:** the enhanced `for` over a `List` allocates **one** iterator per
call and inserts **one** `checkcast` per element. Over an *array* it would allocate
nothing and there would be no `checkcast` at all — worth knowing when you are deciding
what shape a hot internal API should take.

### What to look for — construct 2: the string concatenation

Inside the loop body, **this is the reviewer's claim, and here is where it is settled**:

- **One `invokedynamic`**, whose call name is `makeConcatWithConstants`.
- **No `new java/lang/StringBuilder`**, no `invokevirtual StringBuilder.append`, no
  `StringBuilder.toString`.
- In the `BootstrapMethods` section of `javap -v`, a bootstrap entry naming
  `java/lang/invoke/StringConcatFactory.makeConcatWithConstants`, with a **recipe
  string** static argument containing your literal `,` and `\n` text and placeholder
  markers for the dynamic values.

| What you see | What it means |
|---|---|
| One `invokedynamic makeConcatWithConstants` per concatenation expression, no `StringBuilder` | The Java 9+ behaviour. The reviewer's mental model is a Java 8 model. **Say so with the listing, not with an opinion.** |
| `new StringBuilder` / `append` / `toString` chain | You compiled with `--release 8`, or with `-XDstringConcat=inline`, or you are on a compiler configured for the old strategy. Check your `--release` flag. |
| **Two** indy sites in one iteration | Normal if the expression is split across an operator the compiler cannot fold into one recipe. Read the recipe strings in `BootstrapMethods` to see how it partitioned. |
| An indy site **outside** the loop as well | That is the `""` initialiser or a constant-folded literal. Compare offsets against the loop's branch target. |

**But now read the loop as a whole**, and notice what `javap` *does* prove:

`out += ...` is `out = out + ...`. So each iteration concatenates the **entire
accumulated string so far** with the new fragment. The indy call's arguments include
`out`. That means each iteration produces a **new `String` whose length grows with the
iteration count** — the classic O(n²) accumulation, copying the whole buffer every
time.

**This is the real defect, and it is not the one the reviewer named.** The reviewer
said "a `StringBuilder` per iteration". `javap` shows there is no `StringBuilder` at
all. The actual problem is quadratic character copying, which is worse and is not
fixed by knowing anything about `StringBuilder` allocation.

With 400k lines and a per-order line count that can reach the tens, the accumulated
string in a large order becomes a genuinely large `char[]`/`byte[]` — and at some size
it becomes a **humongous allocation** in G1 (Topic 71) and a candidate for a heap
dump investigation (Topic 79).

### What to look for — construct 3: the lambda

Inside `needsReview`:

- **One `invokedynamic`** whose invoked name is `test` and whose invoked type returns
  `java/util/function/Predicate`. Note the descriptor: the lambda **captures**
  `thresholdMinor`, so the indy's descriptor takes a `long` **argument**. A
  non-capturing lambda's indy takes no arguments.
- In `BootstrapMethods`: an entry naming
  `java/lang/invoke/LambdaMetafactory.metafactory`.
- In the class's method list under `javap -p`: a **synthetic private static method**
  named something like `lambda$needsReview$0` with the lambda's body in it. Its name
  and index are compiler-chosen; do not depend on the exact string.
- **No extra `.class` file**, which is the drill below.

| What you see | What it means |
|---|---|
| `invokedynamic` + a `lambda$...` synthetic method, and no `$1.class` on disk | The normal Java 8+ lambda shape. This is the mechanical proof for Topic 21. |
| The indy descriptor **takes arguments** | The lambda **captures**. Captured values are passed to the metafactory-produced factory, so a fresh instance is produced per evaluation. |
| The indy descriptor takes **no** arguments | Non-capturing. The current JDK implementation links this to a call site that returns a **cached singleton**. Verify by identity, not by belief — see Hands-on Proof 5. |
| `new ...$1` / `invokespecial ...$1.<init>` | You wrote an anonymous class, not a lambda. A separate class file exists. |

### The fix, and how `javap` proves it landed

```java
    public String describe(Order order, List<OrderLine> lines) {
        StringBuilder out = new StringBuilder(lines.size() * 48);   // sized, not grown
        String ref = order.reference();                             // hoisted
        for (OrderLine line : lines) {
            out.append(ref).append(',')
               .append(line.sku()).append(',')
               .append(line.quantity()).append('\n');
        }
        return out.toString();
    }
```

Re-run `javap -c -p`. **What to look for, and what each observation licenses you to
say:**

| Observation | What it proves | What it does **not** prove |
|---|---|---|
| **No `invokedynamic` inside the loop body** | The per-iteration concatenation is gone from the emitted code | that the program got faster |
| One `new java/lang/StringBuilder` **before** the loop branch target | Exactly one builder is allocated per call | that the allocation happens at runtime — escape analysis may remove it (Topic 75) |
| `invokevirtual StringBuilder.append` inside the loop, returning `StringBuilder` | The fluent chain is a chain of virtual calls on the same object | anything about their cost; C2 will almost certainly inline them |
| `getfield`/`invokevirtual order.reference()` moved **before** the loop | The hoist landed | that it mattered — C2 hoists loop-invariant calls itself when it can prove purity |

### The honest conclusion to write in the PR

> "The comment is based on a pre-Java-9 model. `javap -c` shows no `StringBuilder`
> anywhere in the original — `+` compiles to an `invokedynamic` bound to
> `StringConcatFactory`. But there **is** a real bug: `out +=` re-concatenates the
> accumulated string each iteration, which is quadratic in the number of lines. That
> is worth fixing on algorithmic grounds regardless of allocation.
>
> I have **not** measured whether this matters for the nightly export. Before I claim
> a speedup I will run it under a profiler (Topic 78) against the Topic 65 dataset,
> because the export is dominated by JDBC streaming and file I/O, and it is entirely
> possible that string building is 0.5% of its wall clock. `javap` told us what the
> compiler did; it cannot tell us what the job spends its time on."

**That paragraph is the deliverable of this topic.** It separates the two questions,
answers the one bytecode can answer, and explicitly refuses to answer the other one
without the right tool.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — reading `javap -c` and concluding something about runtime performance

**This is the trap. The other four are footnotes to it.**

**Wrong:**

```java
// Reviewer: "Version A is faster — javap shows 4 fewer instructions."

// Version A
public long total(List<OrderLine> lines) {
    long t = 0;
    for (int i = 0; i < lines.size(); i++) t += lines.get(i).amountMinor();
    return t;
}

// Version B
public long total(List<OrderLine> lines) {
    return lines.stream().mapToLong(OrderLine::amountMinor).sum();
}
```

Version A's `javap` listing is short and flat: an index local, a compare, a
`invokeinterface get`, a `checkcast`, an `invokeinterface amountMinor`, an `ladd`, an
`iinc`, a `goto`. Version B's is three instructions and two `invokedynamic` sites,
because all the work is inside the JDK's stream machinery, which `javap` on *your*
class cannot see at all.

**Exact symptom:** you merge Version A on the strength of the bytecode, redeploy, and
the p99 on `GET /orders` in the Topic 65 baseline is **unchanged within noise**. When
someone asks for the number that justified the change, you do not have one. Worse
variant: you rewrite thirty methods this way, and the diff is now large enough that a
genuine regression hides inside it.

**Root cause — three independent errors compounding:**

1. **Bytecode instruction count is not a cost model.** One `invokeinterface` may cost
   more than twenty arithmetic instructions, or nothing at all if it inlines and the
   result is folded.
2. **`javap` on your class shows only your class.** Version B's real work is inside
   `java.util.stream`, in code you did not disassemble. You compared a whole
   implementation against a call.
3. **The operand stack you read does not exist at runtime.** C2 turns it into
   registers, then inlines, then unrolls, then eliminates. What you read is a transport
   encoding.

**Fix — the two-tool rule, and the exact sequence:**

```bash
# 1. javap answers ONLY: what did the language do?
javap -c -p out/com/orderflow/orders/OrderTotals.class

# 2. Did the method get compiled, and did the callee inline?
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+PrintInlining \
     -cp out com.orderflow.Main 2>&1 | grep -i ordertotals

# 3. Is it actually hot? (Topic 78)
#    Attach async-profiler to the running orderflow container under k6 load.

# 4. Is A actually faster than B? (Topic 77)
#    A JMH benchmark with @Fork(3), a Blackhole, and both variants as separate
#    @Benchmark methods — and report the error bars, not the means.
```

**The rule to memorise:** *`javap` answers "what did the language do". A profiler
answers "where does the time go". JMH answers "is A faster than B". Using the first
tool for the second or third question is the mistake, and it is the most common
mistake made with this tool.*

---

### Trap 2 — counting bytecode instructions as a proxy for cost

**Wrong:** a team adopts a code-review guideline: "prefer the form with fewer
bytecodes."

**Exact symptom, in two flavours, both observable:**

- *Flavour A:* the "optimised" version measures **slower** in JMH. Concretely: someone
  replaces an enhanced `for` over an `ArrayList` with an index loop to remove the
  iterator allocation, and the JMH result is within the error bar or slightly worse —
  because C2 was already scalar-replacing the iterator (Topic 75), while the index
  loop's `get(i)` reintroduces a bounds check the iterator form had let it eliminate.
- *Flavour B:* the team splits a large method into small ones to "reduce bytecode",
  and throughput drops. `-XX:+PrintInlining` shows `callee is too large` or the caller
  hitting `MaxInlineLevel`, so a chain that used to inline into one compiled unit no
  longer does.

**Root cause:** bytecode count models the *interpreter*, and the interpreter runs your
code only until it gets hot. Once C2 compiles a method, the bytecode's shape matters
in exactly one way — **as an input to the inlining heuristic** — and not as a proxy
for the number of machine instructions executed.

**Fix:** replace the guideline with two real ones.

1. "Prefer the form that reads correctly. Measure with JMH before claiming either is
   faster."
2. "If a method is genuinely on a hot path, check its `Code` length with `javap -v`
   and check whether it inlines with `-XX:+PrintInlining` — that is the one place
   bytecode size legitimately predicts a runtime decision." (See Measurement below.)

---

### Trap 3 — using `javap` without `-p` and concluding a member does not exist

**Wrong:**

```bash
javap OrderService.class | grep -i validate
# no output
# "There's no validate method. The annotation must be on something else."
```

**Exact symptom:** you spend twenty minutes convinced a method was deleted or renamed,
and it is sitting right there as `private`. In the lambda case, the symptom is
sharper and stranger: you disassemble a class containing lambdas, see the
`invokedynamic` sites, and **cannot find the method bodies anywhere**, because
`lambda$foo$0` is `private static synthetic` and plain `javap` hides it. You conclude
the bodies "live in the JDK", which is wrong and will mislead you for months.

**Root cause:** bare `javap` prints only members the *access flags* say are `public`
(and `protected`, depending on invocation). Compiler-generated members —
`lambda$...`, bridge methods, `access$N` accessors for outer-class private access,
`$SwitchMap$...`, `this$0` — are almost all `private` and/or `synthetic`.

**Fix:** **use `-p` always.** Make it muscle memory:

```bash
javap -c -p Whatever.class
```

There is no downside. There is never a reason to hide private members from yourself
when you are disassembling deliberately.

---

### Trap 4 — reading a class file compiled by a different `javac` or `--release` and concluding the language changed

**Wrong:** you disassemble a class from a dependency jar, see `new StringBuilder` and
`append`, and write in a design doc: "Java still compiles `+` to `StringBuilder`;
the `invokedynamic` claim is wrong."

**Exact symptom:** two engineers disassemble what they believe is "the same code" and
get different listings, and the argument becomes unresolvable because neither knows
why. Second symptom: your listing has no local variable names and `javap -l` shows an
empty `LocalVariableTable`, so you cannot map slots to variables and start guessing.

**Root cause — three distinct versioning axes, each of which changes the output:**

1. **The `javac` that produced the class file.** A jar built in 2016 with JDK 8's
   compiler contains JDK 8's desugarings *forever*. Bytecode is the distribution
   format — it does not get recompiled when you upgrade your JDK.
2. **The `--release` / `-source`/`-target` used.** Compiling with `--release 8` on JDK
   25 deliberately emits Java 8-era constructs, including `StringBuilder`-based concat.
3. **Debug info.** Without `-g`/`-g:vars`, there is no `LocalVariableTable`, so slots
   have no names. Maven and Gradle enable debug by default; a hand-rolled `javac`
   invocation may not.

**Fix — always establish provenance before reasoning about a listing:**

```bash
# What class-file version is this, and therefore which javac era emitted it?
javap -v Whatever.class | head -5      # look for 'major version:'
# 52 = Java 8, 61 = Java 17, 65 = Java 21, 69 = Java 25
# (If your file's number is not in that list, do NOT guess — 'major version N'
#  maps to Java (N - 44). Check the JVMS class-file version table for your JDK.)

# And when compiling for a lab, be explicit and keep names:
javac -g --release 21 -d out $(find src -name '*.java')
```

**The rule:** a `javap` listing is only evidence about **the compiler that produced
that file**. Quote the class-file version alongside the listing, always.

---

### Trap 5 — assuming an anonymous class and a lambda are "the same thing with different syntax"

**Wrong:**

```java
// "These are identical; the lambda is just shorter."
Predicate<OrderLine> a = new Predicate<>() {
    public boolean test(OrderLine l) { return l.quantity() > 0; }
};
Predicate<OrderLine> b = l -> l.quantity() > 0;
```

**Exact symptom, three observable ones:**

1. `ls *.class` shows an extra file — `Whatever$1.class` — for the anonymous version
   and **no extra file** for the lambda. On a codebase with thousands of small
   callbacks, that is thousands of class files, all of which must be **loaded,
   verified and stored in metaspace** at startup (Topics 67, 68). It is a measurable
   startup and metaspace cost, and it is the reason it is a design decision and not a
   style one.
2. `System.identityHashCode` on the same **non-capturing** lambda expression evaluated
   twice returns the **same** value; on an anonymous class it returns two different
   values, because `new` allocates unconditionally, every time.
3. An anonymous class captures the enclosing instance if it references any instance
   member — via a synthetic `this$0` field — which turns a "small callback" into a
   **strong reference to the whole enclosing object**. That is one of the four classic
   leak shapes in Topic 79, and it does not apply to a lambda unless the lambda itself
   references `this`.

**Root cause:** an anonymous class is a **compile-time** artefact — a real class,
compiled into a real file, instantiated with `new`/`invokespecial`. A lambda is a
**link-time** artefact — an `invokedynamic` whose bootstrap decides at runtime how to
materialise the function object, currently by spinning a hidden class per call site,
with non-capturing instances cached.

**Fix:** stop treating them as interchangeable. Use a lambda unless you need something
a lambda cannot do (multiple methods, state, a name, `this` referring to the callback
itself). And **prove the difference yourself** — that is the failure drill below.

---

## Hands-on proof

Every command below is one **you** run. I have no JVM. What follows is the exact
source, the exact command, what to look for, and how to read each plausible result.

### Setup

```bash
mkdir -p ~/java-lab/76 && cd ~/java-lab/76
java --version        # expect 21 or 25
javac --version       # MUST match your expectation — this is Trap 4
javap --help | head -20
```

> Note the second and third commands. Establishing which compiler you are using is
> step zero of every bytecode investigation, not an afterthought.

### Proof 1 — the stack machine, and `this` in slot 0

`Stack1.java`:

```java
public class Stack1 {
    private int quantity = 3;
    private static int staticQuantity = 3;

    public int instanceDouble(int n) { return quantity * n; }
    public static int staticDouble(int n) { return staticQuantity * n; }
}
```

```bash
javac -g Stack1.java
javap -c -p -v Stack1.class
```

**What to look for:**

- In `instanceDouble`: the method starts with **`aload_0`** (push `this`), then
  `getfield`. The parameter `n` is loaded with **`iload_1`** — slot 1, because slot 0
  is taken.
- In `staticDouble`: **no `aload_0`**. The field read is **`getstatic`**. The parameter
  is loaded with **`iload_0`** — slot 0, because there is no `this`.
- In `javap -v`, each method's `Code` attribute header shows `stack=`, `locals=` and
  `args_size=`. Compare `args_size` between the two: the instance method counts `this`.

| What you see | What it means |
|---|---|
| `aload_0` + `getfield` in the instance method; `getstatic` in the static one | Expected. You have directly observed the `this`-in-slot-0 rule. |
| `iload_1` (instance) vs `iload_0` (static) for the same parameter name | The slot shift caused by `this`. This is the single most useful fact for reading any listing. |
| `args_size=2` for `instanceDouble(int)` | Correct — `this` plus one parameter. If this confuses you later, come back to this line. |
| `getfield` in the static method | Impossible; you have disassembled the wrong method. Check for a typo in the source. |

### Proof 2 — the five invoke instructions, all in one class

`Invokes.java`:

```java
import java.util.List;
import java.util.ArrayList;
import java.util.function.Supplier;

public class Invokes {
    private String secret() { return "s"; }          // private -> invokespecial
    public  String open()   { return "o"; }          // public   -> invokevirtual
    static  String util()   { return "u"; }          // static   -> invokestatic

    public void all() {
        secret();                                     // invokespecial
        open();                                       // invokevirtual
        util();                                       // invokestatic

        List<String> viaInterface = new ArrayList<>();
        viaInterface.size();                          // invokeinterface (static type = List)

        ArrayList<String> viaClass = new ArrayList<>();
        viaClass.size();                              // invokevirtual  (static type = ArrayList)

        Supplier<String> s = () -> "lambda";          // invokedynamic
        s.get();                                      // invokeinterface
        String joined = "a" + open() + "b";           // invokedynamic (StringConcatFactory)
        System.out.println(joined);
    }
}
```

```bash
javac -g Invokes.java
javap -c -p Invokes.class | grep -n "invoke"
```

**What to look for:** every one of the five instructions, and specifically the pair
`viaInterface.size()` → `invokeinterface` versus `viaClass.size()` → `invokevirtual`
on **the same runtime object**.

| What you see | What it means |
|---|---|
| All five instruction names appear | Expected. Keep this file; it is your reference class. |
| `invokeinterface List.size` and `invokevirtual ArrayList.size` on the same object | **The dispatch instruction is chosen from the STATIC type at the call site.** This is a fact many working Java engineers do not know. |
| `invokespecial` on `secret()` | Private methods are non-virtual. This is half of why a `private @Transactional` method can never be advised (Topic 40). |
| Two `invokedynamic` sites | One for the lambda, one for the string concat. Confirm which is which by their invoked names (`get` vs `makeConcatWithConstants`) in the `// ` comment. |
| Only **one** `invokedynamic` | Your `--release` is 8, so `+` used `StringBuilder`. Recompile with `--release 21`. |
| `invokevirtual` where you expected `invokeinterface` on a `List` | The compiler resolved through a class type — check whether you declared `ArrayList` rather than `List`. |

### Proof 3 — the bootstrap methods, verbatim

```bash
javap -v -p Invokes.class | sed -n '/BootstrapMethods/,$p'
```

**What to look for:** a numbered list of bootstrap entries. For each, the method
handle name and the static arguments.

| What you see | What it means |
|---|---|
| An entry naming `LambdaMetafactory.metafactory` | The lambda's linkage. The static arguments are the erased signature, a handle to the implementation method, and the instantiated signature. |
| An entry naming `StringConcatFactory.makeConcatWithConstants` with a **recipe string** | String concatenation. The recipe contains your literal text with placeholders. Read it — it tells you exactly how the compiler partitioned the expression. |
| An entry naming `LambdaMetafactory.altMetafactory` | Used for serializable lambdas and lambdas with extra marker interfaces. Not an error. |
| Nothing at all | Your class contains no `invokedynamic`. Check your `--release`, and check you disassembled the right class. |

> **Uncertainty, stated once:** I will not tell you how many static arguments each
> entry has or their exact order, because that has varied across JDK releases.
> **Settling command: the one above.** Read your own output.

### Proof 4 — every desugaring, in one pass

`Desugar.java`:

```java
import java.util.List;
import java.util.Map;

public class Desugar {

    enum Status { NEW, PAID, SHIPPED }

    // boxing
    long boxed(Map<Long, Long> counts, long sku) { return counts.get(sku); }

    // enhanced for over a List
    int sumList(List<Integer> xs) { int t = 0; for (int x : xs) t += x; return t; }

    // enhanced for over an array
    int sumArray(int[] xs)        { int t = 0; for (int x : xs) t += x; return t; }

    // varargs
    String join(String... parts)  { return String.join("-", parts); }

    // switch on String
    int codeOf(String s) {
        return switch (s) { case "NEW" -> 1; case "PAID" -> 2; default -> 0; };
    }

    // switch on enum
    int codeOf(Status s) {
        switch (s) { case NEW: return 1; case PAID: return 2; default: return 0; }
    }

    // try-with-resources
    String read(java.io.Reader r) throws java.io.IOException {
        try (var br = new java.io.BufferedReader(r)) { return br.readLine(); }
    }
}
```

```bash
javac -g Desugar.java
javap -c -p Desugar.class > desugar.txt
ls *.class          # note: the enum produces its own file; a synthetic switch-map class may too
```

**What to look for, method by method:**

| Method | Look for | What it teaches |
|---|---|---|
| `boxed` | `invokeinterface Map.get`, then `checkcast java/lang/Long`, then `invokevirtual Long.longValue` | Erasure forces the cast; unboxing forces the call. Two inserted operations, zero written. |
| `sumList` | `Iterable.iterator`, `hasNext`, `next`, `checkcast Integer`, `Integer.intValue` | An iterator **allocation** plus a cast plus an unbox, per element. |
| `sumArray` | `arraylength`, an int index, `iaload` — **no iterator, no checkcast, no unbox** | The array form is structurally cheaper *in emitted code*. Whether it is faster is still a JMH question. |
| `join` | `anewarray java/lang/String`, `dup`, `aastore` per argument, then the invoke | **A varargs call allocates an array at every call site.** This is why `String.format` in a hot logging path is more expensive than it looks. |
| `codeOf(String)` | Two switches: a `lookupswitch` on `hashCode()`, `String.equals` calls, then a `tableswitch` | The string switch is a hash-then-verify structure, not magic. |
| `codeOf(Status)` | `getstatic` of a synthetic `$SwitchMap$...` int array, then `ordinal()`, then `tableswitch` | The switch-map indirection exists so recompiling the enum does not break the switch's class. |
| `read` | An **exception table** with multiple entries, a `close()` call on both paths, and `Throwable.addSuppressed` | try-with-resources is a substantial desugaring. This is where `getSuppressed()` (Topic 08) comes from. |

| What you see | What it means |
|---|---|
| All of the above | Expected. You now have a reference file for every common desugaring. Keep `desugar.txt`. |
| No `checkcast` in `sumList` | Check that the parameter is `List<Integer>` and not `List<int>`-shaped or raw — a raw `List` also produces a `checkcast`, so this would be surprising. |
| `sumArray` contains an iterator | You disassembled `sumList` by mistake; the methods have similar bodies. |
| No `$SwitchMap` class or field | Some compilers place the switch map differently, or the switch was compiled to an `invokedynamic` if it is a pattern switch. Read the actual instruction rather than looking for the field. |

### Proof 5 — non-capturing lambdas are cached; capturing ones are not

This is a **runtime** proof, not a bytecode one, and it is the correct tool for the
question. Note the switch of tools; that is the lesson.

`LambdaIdentity.java`:

```java
import java.util.function.Predicate;

public class LambdaIdentity {
    static Predicate<Integer> nonCapturing()      { return x -> x > 0; }
    static Predicate<Integer> capturing(int bound) { return x -> x > bound; }

    public static void main(String[] args) {
        System.out.println("nonCapturing #1 = " + System.identityHashCode(nonCapturing()));
        System.out.println("nonCapturing #2 = " + System.identityHashCode(nonCapturing()));
        System.out.println("capturing(5) #1 = " + System.identityHashCode(capturing(5)));
        System.out.println("capturing(5) #2 = " + System.identityHashCode(capturing(5)));
    }
}
```

```bash
java LambdaIdentity.java
```

| What you see | What it means |
|---|---|
| The two `nonCapturing` lines match; the two `capturing` lines differ | The expected behaviour on current HotSpot. The non-capturing lambda's call site was linked to a constant that returns one shared instance; the capturing one produces a fresh object per evaluation. |
| All four differ | Your JDK's `LambdaMetafactory` is not caching. **This is allowed** — it is an implementation choice, not a spec guarantee. Trust your output over my description, and note it: your code must not depend on lambda identity either way. |
| All four match | Would be surprising for the capturing case; re-check that `bound` is really captured and not a compile-time constant the compiler folded. |

> **Stated uncertainty:** instance caching for non-capturing lambdas is current
> `LambdaMetafactory` behaviour, not a specified guarantee. Never write code that
> depends on lambda identity — including as `Map` keys, in `==` comparisons, or as
> listener-removal tokens. That last one is a real leak shape in Topic 79: you cannot
> `removeListener(x -> ...)` because you do not have the same object back.

### Proof 6 — the compile-time constant inlining trap

```bash
mkdir -p constants && cd constants
cat > Limits.java <<'EOF'
public class Limits { public static final int MAX_LINES = 500; }
EOF
cat > Caller.java <<'EOF'
public class Caller { public int max() { return Limits.MAX_LINES; } }
EOF
javac Limits.java Caller.java
javap -c Caller.class
```

**What to look for:** in `max()`, a **`sipush 500`** (or `ldc`), **not** a `getstatic`
referencing `Limits.MAX_LINES`.

Now prove the consequence:

```bash
sed -i.bak 's/500/900/' Limits.java
javac Limits.java            # recompile ONLY Limits
javap -c Caller.class        # Caller still says 500
java -cp . -e 2>/dev/null; java Caller.java 2>/dev/null   # or write a main that prints it
```

| What you see | What it means |
|---|---|
| `sipush 500` in `Caller`, and still `500` after recompiling only `Limits` | **The constant was baked into the caller's constant pool.** This is a real deployment hazard: partial rebuilds silently keep the old value. |
| A `getstatic` to `Limits.MAX_LINES` instead | The field is not a compile-time constant — check it is both `static` **and** `final` **and** initialised with a constant expression. A `static final` initialised from a method call is not a compile-time constant. |

**Why this matters operationally:** any build system that does incremental compilation
without full dependency tracking can produce a jar where callers hold stale constants.
If you ever see "we changed the constant, deployed, and it did not take effect", this
is the first thing to check, and `javap -c` on the caller settles it immediately.

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the output yourself and
written down what you saw.

### The claim you are going to break

> "A lambda is just shorthand for an anonymous inner class."

You have almost certainly said this, or nodded at it. It is wrong at the class-file
level, at the linkage level and at the allocation level, and one directory listing
kills it.

### Setup

`~/java-lab/76/drill/LambdaVsAnon.java`:

```java
package drill;

import java.util.function.Predicate;

public class LambdaVsAnon {

    /** An anonymous class implementing the same interface. */
    static Predicate<Integer> anonymous() {
        return new Predicate<Integer>() {
            @Override public boolean test(Integer q) { return q > 0; }
        };
    }

    /** A lambda implementing the same interface, same body. */
    static Predicate<Integer> lambda() {
        return q -> q > 0;
    }

    /** A capturing lambda, for contrast. */
    static Predicate<Integer> capturingLambda(int bound) {
        return q -> q > bound;
    }

    /** A capturing anonymous class, for contrast. */
    static Predicate<Integer> capturingAnonymous(int bound) {
        return new Predicate<Integer>() {
            @Override public boolean test(Integer q) { return q > bound; }
        };
    }

    public static void main(String[] args) {
        System.out.println("anon  #1 = " + System.identityHashCode(anonymous()));
        System.out.println("anon  #2 = " + System.identityHashCode(anonymous()));
        System.out.println("lam   #1 = " + System.identityHashCode(lambda()));
        System.out.println("lam   #2 = " + System.identityHashCode(lambda()));
        System.out.println("cap   #1 = " + System.identityHashCode(capturingLambda(5)));
        System.out.println("cap   #2 = " + System.identityHashCode(capturingLambda(5)));
    }
}
```

### Step 1 — compile and count the class files

```bash
cd ~/java-lab/76
javac -g -d out drill/LambdaVsAnon.java
ls -l out/drill/
ls out/drill/*.class | wc -l
```

**Write down the exact filenames.** This is the headline result of the drill.

| What you see | What it means |
|---|---|
| `LambdaVsAnon.class`, `LambdaVsAnon$1.class`, `LambdaVsAnon$2.class` — **three files** | Expected. The **two anonymous classes** each produced their own class file. The **two lambdas produced none.** This is the whole drill in one `ls`. |
| Only `LambdaVsAnon.class` | You removed the anonymous classes, or you are looking at the wrong directory. Anonymous classes always produce a file. |
| Four or more files | Check for other nested types you added. Each anonymous class gets its own `$N`. |

**Now say the consequence out loud, because this is why it matters and not trivia:**
every anonymous class is a separate class file that must be **read, verified, linked
and stored in metaspace** at first use (Topics 67, 68). In a codebase with three
thousand small callbacks, that is three thousand class-loading events at startup, and
metaspace to hold them all. Lambdas move that cost to link time and share
implementation machinery, which is a measurable startup difference on a service that
must pass a Kubernetes readiness probe quickly (Topic 121).

### Step 2 — disassemble and compare the two shapes

```bash
javap -c -p out/drill/LambdaVsAnon.class > main.txt
javap -c -p out/drill/'LambdaVsAnon$1.class' > anon1.txt
less main.txt
less anon1.txt
```

**What to look for in `main.txt`:**

In `anonymous()`:

1. `new drill/LambdaVsAnon$1` — allocate an uninitialised instance of the generated
   class.
2. `dup` — duplicate the reference (one copy for the constructor, one to return).
3. `invokespecial drill/LambdaVsAnon$1.<init>` — run the constructor.
4. `areturn`.

In `lambda()`:

1. **`invokedynamic`** with invoked name `test` and a return type of
   `java/util/function/Predicate`, taking **no arguments**.
2. `areturn`.

In `capturingLambda(int)`:

1. `iload_0` — push the captured `bound`.
2. **`invokedynamic`** whose descriptor now **takes an `int`** and returns
   `Predicate`.
3. `areturn`.

And in the method list (this is why `-p` is mandatory): **synthetic private static
methods** holding the lambda bodies, named `lambda$lambda$0`-ish and
`lambda$capturingLambda$1`-ish. Exact names are compiler-chosen; do not depend on them.

**What to look for in `anon1.txt`:**

- A real class declaration: `final class drill.LambdaVsAnon$1 implements
  java.util.function.Predicate`.
- A constructor `<init>`.
- A method `test(java.lang.Integer)` — your body.
- **A second `test` taking `java.lang.Object`** — the **bridge method** (Topic 06),
  flagged `ACC_BRIDGE, ACC_SYNTHETIC`, whose body is `checkcast java/lang/Integer`
  followed by an `invokevirtual` to the real `test`. Confirm with:

```bash
javap -v -p out/drill/'LambdaVsAnon$1.class' | grep -A3 -i "test"
# look for 'flags:' lines containing ACC_BRIDGE and ACC_SYNTHETIC
```

| What you see | What it means |
|---|---|
| `new`/`dup`/`invokespecial` for the anon; a single `invokedynamic` for the lambda | The mechanical difference. One is a compile-time class instantiation; the other is a link-time call site. |
| The lambda's indy takes **no** args; the capturing one takes an `int` | **Captured values are arguments to the indy.** That is precisely why a capturing lambda cannot be a cached singleton. |
| Two `test` methods in `$1.class`, one flagged `ACC_BRIDGE` | Erasure's bridge method, in the flesh. `Predicate`'s erased signature is `test(Object)`; yours is `test(Integer)`; the bridge casts and delegates. |
| No `lambda$...` methods in `main.txt` | You forgot `-p`. That is Trap 3. |
| A `this$0` field in `$1.class` | Your anonymous class captured the enclosing instance. In a `static` method it should not — check whether you accidentally made the methods non-static. This field is the leak shape from Topic 79. |

### Step 3 — run it and read the identities

```bash
java -cp out drill.LambdaVsAnon
```

| What you see | What it means |
|---|---|
| `anon #1` ≠ `anon #2` | Every `new` allocates. Unconditionally, always. |
| `lam #1` == `lam #2` | The non-capturing lambda's call site returns a **cached instance**. Two evaluations, one object, zero allocations after the first. |
| `cap #1` ≠ `cap #2` | The capturing lambda allocates per evaluation, because the captured value is an argument. |
| `lam #1` ≠ `lam #2` | Allowed. Caching is an implementation choice. Record what you saw and note that your code must not depend on it either way. |

### Step 4 — write down the four-line conclusion

Before reading on, write these four lines in your own words:

1. How many class files did the anonymous version produce, and the lambda version?
2. Which instruction created the anonymous instance, and which one created the lambda?
3. Which lambda allocated per call, and what in the bytecode predicted that?
4. What did the bridge method in `$1.class` do, and why does it exist?

### What the drill proves

**Anonymous class:** a class, compiled at build time, into a file, on disk, loaded and
verified at runtime, instantiated with `new` every single time, plus a bridge method,
plus — if it touches an instance member — a `this$0` reference to the whole enclosing
object.

**Lambda:** no class file, an `invokedynamic` linked once by `LambdaMetafactory`, a
synthetic private static method holding the body, and — for the non-capturing case —
one shared instance for the life of the JVM.

**They are not the same construct with different syntax.** They differ in class count,
in allocation behaviour, in what they capture, and in when the implementation is
decided.

And the final, obligatory sentence: **none of that told you which is faster.** The
allocation difference is real in the emitted code; whether it costs anything in your
service depends on whether the site is hot and whether escape analysis removes it
anyway. That is Topic 77.

---

## Measurement

`javap` is not a measurement tool. This section is about the one place where a
bytecode-level fact legitimately feeds a runtime decision, and about how to hand off
to the tools that do measure.

### The one legitimate performance use of `javap`: bytecode size and inlining

HotSpot decides whether to inline a callee partly from its **bytecode size in bytes**.
`javap -v` prints that number for every method, as the length of the `Code` attribute.

```bash
javap -v -p out/com/orderflow/orders/OrderTotals.class | grep -B4 "stack=" | grep -E "Code:|stack=|^  [a-zA-Z]"
```

More usefully, extract every method's code length:

```bash
javap -c -p -v out/com/orderflow/orders/OrderTotals.class \
  | awk '/^  [a-zA-Z].*\(/ {m=$0} /stack=/ {print $0 "   <- " m}'
```

**Why the size matters.** Roughly:

- A **small** method is inlined aggressively wherever it is called.
- A **larger** method is inlined only if the call site is **hot**, and only up to a
  larger size limit.
- Above that limit it is **not inlined at all**, no matter how hot.
- Inlining also stops at a maximum **depth**.

The three flags that control this are `-XX:MaxInlineSize`, `-XX:FreqInlineSize` and
`-XX:MaxInlineLevel`.

> **I am not going to quote their default values as facts.** They have changed across
> HotSpot versions and can differ by platform. **Settling command:**
>
> ```bash
> java -XX:+PrintFlagsFinal -version | grep -E "MaxInlineSize|FreqInlineSize|MaxInlineLevel|InlineSmallCode"
> ```
>
> Run that on **your** JDK 25 and use those numbers. That command is worth more than
> any number I could write here, because it is right by construction.

**And the consequence, which is the real point:** a method too large to inline blocks
**escape analysis** in its caller (Topic 75). Escape analysis needs to see the whole
lifetime of an allocation; if the allocation is passed into a method that did not
inline, the compiler must assume it escapes. So "this method is 400 bytes of bytecode"
can be the root cause of an allocation you cannot otherwise explain in an allocation
profile.

That is a genuine `javap` → runtime chain, and it is the only one. Note its shape:
`javap` supplies an **input** to a decision made elsewhere, and you still verify the
decision with `-XX:+PrintInlining`.

### Confirming what actually happened: the handoff commands

```bash
# Which methods got compiled, at which tier, and were any deoptimised?
java -XX:+PrintCompilation -cp out com.orderflow.Main 2>&1 | tee compilation.log

# Which call sites inlined, and why not when they did not?
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     -cp out com.orderflow.Main 2>&1 | tee inlining.log
grep -iE "too large|not inlined|hot method too big|callee is too large" inlining.log
```

**How to read `PrintInlining`:** it prints a tree of call sites with a verdict per
site. What to look for:

| Verdict text you may see | What it means |
|---|---|
| `inline (hot)` | The call site was hot and the callee was inlined. What you want on a hot path. |
| `too big` / `callee is too large` | Bytecode size exceeded the limit. **This is where `javap -v`'s `Code` length becomes actionable** — shrink the method or move the cold part out. |
| `not inlineable` | Often a native method, or one the compiler cannot see through. |
| `no static binding` / bimorphic / megamorphic notes | The call site sees several receiver types, so devirtualisation failed. Relates to profile pollution, Topic 74. |
| `recursive` / `inlining too deep` | `MaxInlineLevel` reached. |

> **One line of uncertainty:** the exact wording of these verdicts varies by JDK
> version and is not a stable API. Grep case-insensitively for `inline`, `too`, and
> `not`, and read what your JVM prints rather than matching my strings.

### Rules for measuring anything in this topic

1. **Never quote an instruction count as a cost.** If you catch yourself writing
   "that's 12 bytecodes versus 8", stop and open Topic 77.
2. **Always state which compiler produced the class file.** `javap -v | head -5`.
3. **A `javap` observation is a hypothesis, not a result.** "The compiler emits an
   allocation here" is a hypothesis about runtime behaviour. Confirm it with an
   allocation profile (async-profiler `-e alloc`, Topic 78) before acting.
4. **Confirm removal, not just presence.** If you refactor to eliminate an allocation,
   the proof is an allocation profile showing it gone under the Topic 65 load — not a
   `javap` listing showing the `new` is gone from source-level emitted code.
5. **Do not compare bytecode across `--release` levels** and call it a language change.

### What is safe in production

`javap` is a **static, offline** tool. It reads files. It never touches a running JVM,
never attaches, never perturbs anything. You can run it against a production jar on
your laptop with zero risk:

```bash
unzip -o orderflow.jar 'BOOT-INF/classes/com/orderflow/orders/*' -d /tmp/of
javap -c -p /tmp/of/BOOT-INF/classes/com/orderflow/orders/OrderTotals.class
```

That is a meaningful property. It means "what did the compiler emit for the code that
is actually deployed" is always answerable, from the artefact, with no access to the
running system and no incident-time risk. Very few of your Phase 8 tools have that
property; JFR (Topic 78) is the other one and it comes with caveats.

---

## Practice exercises

### 1 — Easy: build your own desugaring reference sheet

Take `Desugar.java` from Proof 4. For **each** of its eight methods, and before running
anything:

1. Write down, in plain English, what you predict the compiler will emit — specifically
   whether it will insert any **method call you did not write**, any **allocation** you
   did not write, or any **cast** you did not write.
2. Run `javap -c -p` and check.
3. Produce a table: construct → inserted calls → inserted allocations → inserted casts.

Then answer two questions in writing:

- **(a)** Which construct surprised you most, and what wrong belief did it correct?
- **(b)** `sumArray` emits no iterator, no `checkcast` and no unboxing, while `sumList`
  emits all three. Does that make `sumArray` faster? Answer in **exactly two
  sentences**: one about what the bytecode proves, one about what would have to be
  true for the difference to show up in a measurement.

### 2 — Medium: the audit (combines Topics 01, 06, 18, 21, 40)

Here is a class from `orderflow`'s pricing module. It contains **six** constructs that
emit something you did not write. Find all six, name the topic each belongs to, state
what `javap -c -p` would show, and then — separately and explicitly — state whether
each one is a **performance concern**, a **correctness concern**, or **neither**.

```java
package com.orderflow.pricing;

import java.util.*;
import java.util.function.Function;

public class PriceCalculator implements Comparator<Product> {

    private static final Map<Long, Integer> discountBySku = new HashMap<>();
    private final String currency;

    public PriceCalculator(String currency) { this.currency = currency; }

    @Override
    public int compare(Product a, Product b) {
        return Long.compare(a.priceMinor(), b.priceMinor());
    }

    public String auditLine(Order order) {
        String line = "";
        for (OrderLine ol : order.lines()) {
            line += order.reference() + "|" + ol.sku() + "|" + ol.quantity() + ";";
        }
        return line;
    }

    public long discountedTotal(Order order) {
        long total = 0;
        for (OrderLine ol : order.lines()) {
            Integer pct = discountBySku.get(ol.sku());
            total += ol.amountMinor() * (100 - pct) / 100;
        }
        return total;
    }

    public Function<Product, String> labeller() {
        return p -> p.name() + " (" + currency + ")";
    }
}
```

Hints, in the order you should think about them:

- One of the six is a **bridge method**, and it is not in any method body — it is a
  *member of the class*. `javap -p -v` and look at the flags.
- One of the six is an **unboxing NPE waiting to happen** (Topic 01) and is a
  correctness bug, not a performance one. Say what the exception message will name.
- One of the six is the **quadratic accumulation** from Example 2.
- One of the six is a **capturing** lambda. Say what it captures and what the indy's
  descriptor will therefore contain — and note the second-order consequence of what it
  captures for object lifetime (Topic 79).
- Two of the six are ordinary desugarings with no consequence at all. Naming them as
  harmless is part of the exercise; a senior engineer's value is as much in what they
  *do not* flag.

Finally: **which single one of the six would you actually change**, and what evidence
would you require before claiming the change improved anything?

### 3 — Hard: production simulation on the `orderflow` baseline

You are settling a real dispute with evidence, and the deliverable is a written
decision, not a patch.

**Part A — establish provenance.** For the deployed `orderflow` jar:

```bash
unzip -l target/orderflow.jar | head -30
unzip -o target/orderflow.jar 'BOOT-INF/classes/com/orderflow/**' -d /tmp/of
javap -v /tmp/of/BOOT-INF/classes/com/orderflow/orders/OrderService.class | head -5
```

Record the class-file major version. Then check **three dependency jars** the same way
(pick Spring, Hibernate and one small utility). Report the range of class-file versions
present in your deployed artefact and explain, in two sentences, why they differ and
why that is normal.

**Part B — find every boxing site in the hot path.** The Topic 65 mix is 70% catalogue
read. Identify the classes on the `GET /products` path. For each, disassemble and count
occurrences of `Integer.valueOf`, `Long.valueOf`, `intValue`, `longValue`:

```bash
for f in $(find /tmp/of/BOOT-INF/classes/com/orderflow/catalog -name '*.class'); do
  n=$(javap -c -p "$f" | grep -cE "java/lang/(Integer|Long|Double)\.(valueOf|intValue|longValue|doubleValue)")
  [ "$n" -gt 0 ] && echo "$n  $f"
done | sort -rn
```

Produce the ranked list. **Then stop and write one paragraph explaining why this list
is NOT a list of things to fix.** Name at least two reasons drawn from Topic 75.

**Part C — the two-tool comparison, and this is the core of the exercise.** Take the
top method from Part B's list. Now answer the same question with the *other* tool:

1. Run `orderflow` at the Topic 65 baseline load under k6.
2. Attach async-profiler in **allocation** mode (Topic 78) for 60 seconds.
3. Find where that method ranks in the allocation profile.

Produce a two-column table: *rank by boxing sites in bytecode* versus *rank by actual
allocation under load*. They will not match. **Explain every discrepancy** using one
of: the method is not hot; escape analysis removed the allocation; the values are in
the `Integer` cache; the allocation is dominated by something else entirely (Hibernate
entity snapshots, JDBC result buffers, Jackson serialization).

**Part D — the decision.** Pick exactly one change worth making, or argue that none
is. Write it as a PR description containing:

- the `javap` evidence (what the compiler emits),
- the profile evidence (what actually allocates under load),
- the p50/p95/p99 from `/docs/java/baselines/` as the current state,
- a falsifiable prediction: "if this change matters, metric X moves by at least Y".

Then **make the change and check the prediction.** If the prediction fails, the correct
deliverable is a PR that says so and closes itself. That outcome is worth more than a
successful optimisation, and writing it down is the exercise.

**Part E — argue against yourself.** You have just spent a day with `javap`. Make the
strongest possible case that a senior engineer on `orderflow` should **never** open
`javap`, and that everything in Parts B–D should have started with a profiler. Then
state the specific class of question for which that case fails — the questions where
only bytecode can answer.

---

## Interview questions

### Q1 — "Is `StringBuilder` faster than `+` for string concatenation?"

**Mid-level answer:** "Yes, you should always use `StringBuilder` because `+` creates a
new `String` object each time."

**Senior answer:** "It depends on where the `+` is, and the answer changed in Java 9.
For a **single expression** — `a + b + c` — `javac` compiles it to one `invokedynamic`
bound to `StringConcatFactory.makeConcatWithConstants`, with a recipe string holding
the literal parts. The JDK then generates a `MethodHandle` chain at link time, which is
typically at least as good as a hand-written builder and can be better, because it can
size the result buffer exactly. So for a single expression, hand-rolling a
`StringBuilder` is usually neutral-to-worse and definitely less readable.

Inside a **loop** it is a different question, and the shape of the problem is not what
people usually say. `s += x` in a loop is `s = s + x`, so each iteration concatenates
the entire accumulated string — that is quadratic character copying, and it is a real
algorithmic bug regardless of what the compiler emits. Hoisting a `StringBuilder`
outside the loop fixes the algorithm, not the allocation.

I would confirm which shape the compiler emitted with `javap -c` — one indy per
expression, or a builder chain if someone is compiling with `--release 8`. And I would
confirm whether it *matters* with JMH, because those are two separate questions.
`javap` tells me what the language did; it tells me nothing about what runs, because
`javac` does essentially no optimisation and C2 does all of it."

**What separates them:** three things. Naming `invokedynamic` and
`StringConcatFactory` with the version boundary. Reframing the loop problem as
**quadratic copying** rather than allocation — which is the correct diagnosis and a
strictly better one. And explicitly refusing to answer the performance half without
measuring.

**Follow-up the interviewer asks:** "Why did they move it to `invokedynamic` at all?"
The answer they want: so the JDK can change the concatenation strategy for every
already-compiled program on earth without recompilation, because the class file only
records "link this with `StringConcatFactory`". Binary compatibility, not speed.

---

### Q2 — "Is a lambda just an anonymous inner class with nicer syntax?"

**Mid-level answer:** "Basically yes — the compiler turns it into an anonymous class
that implements the functional interface."

**Senior answer:** "No, and it is easy to prove. Compile a class with one lambda and
one anonymous class and run `ls *.class`: the anonymous class produces its own
`$1.class` file; the lambda produces none. In the disassembly, the anonymous class is
`new` / `dup` / `invokespecial <init>`, while the lambda is a single `invokedynamic`
whose bootstrap is `LambdaMetafactory.metafactory`, plus a private static synthetic
method holding the body.

Three consequences that matter in production. First, class count: a codebase with
thousands of small callbacks pays thousands of class-loading and verification events at
startup and holds them all in metaspace — lambdas move that to link time and share
machinery. Second, allocation: a **non-capturing** lambda's call site links to a
constant that returns a cached singleton, so repeated evaluation allocates nothing,
whereas `new` always allocates; a **capturing** lambda takes the captured values as
arguments to the `invokedynamic`, so it does allocate per evaluation — and that is
visible in the indy's descriptor. Third, and this one is a leak shape: an anonymous
inner class in a non-static context gets a synthetic `this$0` field holding the whole
enclosing instance, so a small callback can retain a large object graph. A lambda only
captures what it actually references.

The caching of non-capturing instances is current `LambdaMetafactory` behaviour and not
a spec guarantee, so I would never write code that depends on lambda identity —
including trying to remove a listener by passing an equivalent lambda, which silently
does nothing."

**What separates them:** the `ls *.class` proof is the tell. It is concrete, takes five
seconds, and shows the candidate has actually done it. Then the capturing/non-capturing
allocation distinction, and the `this$0` retention point, which connects to memory
leaks and almost nobody volunteers.

**Follow-up:** "When would you deliberately choose an anonymous class over a lambda?"
When you need more than one method, need instance state, need a name in stack traces
for a long-lived object, or need `this` to refer to the callback itself rather than the
enclosing instance.

---

### Q3 — "You read `javap -c` and there are four extra method calls the source doesn't have. What have you learned about performance?"

**Mid-level answer:** "That the method is doing more work than it looks like, so it is
probably slower than expected."

**Senior answer:** "Nothing. I have learned what `javac` **emitted**, which is a
different fact from what the JVM **executes**.

`javac` does essentially no optimisation — it desugars the language into JVM
instructions and stops. Every real optimisation happens at runtime in C1 and C2:
inlining, devirtualisation, escape analysis and scalar replacement, dead-code
elimination, loop unrolling. So an allocation I can see in the bytecode may never
happen at runtime because escape analysis scalar-replaced it, and a virtual call I can
see may compile to a direct jump or vanish entirely into an inlined body.

What I *have* learned is a set of hypotheses worth testing: 'this path boxes', 'this
path allocates an iterator per call', 'this call site is an interface dispatch'. To
turn any of those into a fact about runtime I need different tools —
`-XX:+PrintInlining` to see whether the callee inlined, an allocation profile from
async-profiler to see whether the allocation actually occurred under load, and JMH if I
want to compare two variants.

There is exactly one place bytecode legitimately predicts a runtime decision: **the
method's bytecode size is an input to HotSpot's inlining heuristic**, and a method too
large to inline blocks escape analysis in its caller. So `javap -v`'s `Code` length is
worth checking on a genuinely hot path — but I would still confirm the inlining
decision with `PrintInlining` rather than inferring it from the size."

**What separates them:** the one-word answer "nothing" is the correct answer and almost
nobody gives it. Then the *reason* — the compile-time/runtime split — and then the one
honest exception, which shows the candidate is drawing a real boundary rather than
reciting "always measure".

**Follow-up:** "So when is `javap` the right tool?" For questions about the **language**:
did this box, is there a bridge method here, what does this desugar to, is this lambda
capturing, did the compiler inline that constant into my caller. Deterministic
questions with deterministic answers.

---

### Q4 — "What is `invokedynamic` and why should I care?"

**Mid-level answer:** "It's the instruction used for lambdas. It does dynamic dispatch
at runtime, so it's a bit slower than a normal call."

**Senior answer:** "That last part is backwards, and the misunderstanding is common
enough to be worth correcting carefully.

`invokedynamic` defers **linkage**, not dispatch. The instruction names a **bootstrap
method** in the class's `BootstrapMethods` attribute. The first time that instruction
executes, the JVM calls the bootstrap, which returns a `CallSite` holding a
`MethodHandle`, and the instruction is **permanently linked** to it. Every subsequent
execution goes straight to the linked target. `LambdaMetafactory` returns a
`ConstantCallSite`, whose target can never change — so C2 treats it as a constant and
inlines straight through it. After linking, it is as static as anything else in the
program. 'Dynamic' describes a one-time linkage step, not per-call work.

Four things use it that I meet regularly: lambdas and method references via
`LambdaMetafactory`; string concatenation via `StringConcatFactory` since Java 9;
record `equals`/`hashCode`/`toString` via `ObjectMethods`; and pattern-matching
`switch` via `SwitchBootstraps`.

Why I care: it is a **binary compatibility mechanism**. The class file records only
'link this site with this bootstrap'. That means the JDK can completely change how
lambdas are materialised or how strings are concatenated, and every already-compiled
program on earth picks up the new strategy without being recompiled. That is why it was
chosen over generating a class per lambda at compile time, which would have frozen a
2014 implementation decision into every jar ever built.

The one honest caveat is **startup**: bootstrap methods run at first execution, so a
program with many distinct indy sites pays linkage cost during warm-up. That is a real
consideration for a service with a tight readiness deadline, and it is one of the
things AppCDS and AOT caching target."

**What separates them:** correcting the "dynamic means slow" model with the actual
mechanism, naming the four users, and then giving the *design* reason — binary
compatibility — rather than a performance reason. The startup caveat at the end shows
the candidate knows the genuine cost rather than defending the feature.

**Follow-up:** "How would you see the bootstrap for a specific lambda?"
`javap -v -p TheClass.class` and read the `BootstrapMethods` section.

---

### Q5 — "A junior engineer sends you a PR that rewrites twenty methods to reduce bytecode instruction count. What do you say?"

**Mid-level answer:** "I'd ask them to benchmark it first to prove it's actually
faster."

**Senior answer:** "I'd say two things, in order, and the first one is about the model,
not the process.

The model: bytecode instruction count is a cost model for the **interpreter**, and the
interpreter only runs your code until it gets hot. Once C2 compiles a method, the
operand stack is gone — it has been mapped to registers — and the code has been
inlined, unrolled, devirtualised and had dead code removed. Instruction count does not
survive that transformation in any usable form. There are concrete cases where the
'optimised' version is slower: removing an iterator in favour of an index loop can
reintroduce bounds checks that the iterator form let the compiler eliminate, and
splitting a method into small pieces can push a call chain past `MaxInlineLevel` so it
stops inlining altogether.

The process: a twenty-method refactor with no measurement is a risk with no upside. I
would ask for the p50/p95/p99 from our load baseline before and after — we have those
recorded — and a JMH benchmark with at least three forks and error bars for any
specific claim about one method. If neither moves outside the noise, the correct
outcome is to close the PR, and I would say that up front so it does not feel like
moving the goalposts.

Then the coaching part, because the instinct is a good one pointed at the wrong target:
the *right* use of `javap` here is to find things the compiler emitted that we did not
intend — a boxing site on a genuinely hot path, a varargs array allocated per log call,
a quadratic string accumulation. Those are hypotheses. Then we confirm with an
allocation profile under our real load before changing anything. And there is exactly
one case where bytecode size itself matters: a hot method too large to inline, which
also blocks escape analysis in its caller — but I would verify that with
`-XX:+PrintInlining`, not by counting instructions."

**What separates them:** the mid answer is procedurally correct and adds nothing. The
senior answer explains *why* the model is wrong with two concrete counter-examples,
sets the falsification criterion **before** the work, and redirects the engineer's
energy at a valid use of the same tool. That last move — redirect rather than reject —
is what a staff-level reviewer does.

**Follow-up:** "What if they say the JIT might not kick in for cold paths, so bytecode
count matters there?" A genuinely good point that deserves a genuine answer: yes, for
code that runs a handful of times — startup, one-shot CLI work, serverless cold
starts — the interpreter's cost model is closer to reality. But that is an argument for
optimising *startup*, which is measured with startup time, not with instruction counts,
and is addressed by AppCDS/AOT rather than by micro-refactoring twenty methods.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The JVM is a stack machine; most physical CPUs are register machines; V8's Ignition
   bytecode is a register machine. Derive, from the fact that a `.class` file was
   designed to be **shipped and run anywhere**, why a stack encoding was the right
   choice in 1995 — and then argue whether it would still be the right choice for a
   format designed today.

2. `javac` performs almost no optimisation, by deliberate design. Name one concrete
   thing that would get **worse** if `javac` aggressively optimised — think about what
   C2 knows at runtime that `javac` cannot know at compile time.

3. A `private` method compiles to `invokespecial`, which is non-virtual. Connect that
   single fact to Topic 40's rule that a `private @Transactional` method can never be
   advised — and then explain why that is *two* independent reasons, not one.

4. `invokeinterface` versus `invokevirtual` is chosen from the **static** type at the
   call site, on the same runtime object. Given that, does declaring a variable as
   `ArrayList` rather than `List` make your program faster? Argue both sides, then say
   what you would need to observe to settle it.

5. A compile-time `static final` constant is inlined into every caller's constant pool.
   Design a deployment process for `orderflow` that makes the resulting stale-constant
   hazard impossible. Then say what that process costs, and whether you would actually
   adopt it.

6. Lambdas were implemented with `invokedynamic` specifically so the JDK could change
   the strategy later without recompiling anything. Name the cost of that decision that
   an application pays, and the situation in which that cost is large enough to
   matter.

7. You can read `javap` output from a production jar with zero risk to the running
   system, because it is a static file-reading tool. Every other diagnostic in Phase 8
   touches a live JVM. Given that asymmetry, write down the ordered list of questions
   you would try to answer with `javap` **first** during an incident, before attaching
   anything to the running process — and the point at which you stop and attach.

---

## Quick reference card

### Commands

| Command | What it gives you |
|---|---|
| `javap Foo.class` | public signatures only. Almost never what you want. |
| `javap -p Foo.class` | **+ private, package-private, and synthetic members.** Use `-p` always. |
| `javap -c -p Foo.class` | **+ disassembled method bodies.** Your default. |
| `javap -v -p Foo.class` | **+ constant pool, `BootstrapMethods`, access flags, `Code` lengths, stack map.** For `invokedynamic` and inlining-size questions. |
| `javap -l -p Foo.class` | `LineNumberTable` and `LocalVariableTable` (needs `-g` at compile time) |
| `javap -s -p Foo.class` | internal type descriptors (`(Ljava/lang/String;I)J`) |
| `javap -v Foo.class \| head -5` | **class-file major version — establish provenance first, always** |
| `javap -c -p -cp app.jar com.orderflow.Foo` | disassemble straight out of a jar, no unzip needed |
| `javap -c -p -m your.module com.orderflow.Foo` | disassemble from a module (JPMS, Topic 20) |

### Compile-side flags that change what you will see

| Flag | Effect |
|---|---|
| `-g` | keep full debug info — **local variable names and line numbers** |
| `-g:none` | strip it. Slots have no names; `javap -l` shows nothing useful |
| `--release 21` | emit for Java 21 (indy string concat, modern desugarings) |
| `--release 8` | emit Java 8-era shapes (**`StringBuilder` concat**) — a common source of confusing listings |
| `-parameters` | keep real parameter names in `MethodParameters` (matters for Spring/Jackson binding, not just for reading) |

### Runtime flags for the *other* question

```bash
-XX:+PrintCompilation                                   # what got compiled, at which tier
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining        # which call sites inlined, and why not
-XX:+PrintFlagsFinal -version | grep -i inline           # YOUR JVM's inlining thresholds
-XX:-DoEscapeAnalysis                                    # A/B control for Topic 75
```

### How to read a `javap -c` listing — the columns

```
<offset>: <mnemonic>   <operands>    // <constant pool resolution>
```

- **`<offset>`** — byte offset of the instruction within the method's code array.
  Branch targets are these offsets, which is how you find loop boundaries: a `goto`
  backwards to a smaller offset is the bottom of a loop.
- **`<mnemonic>`** — the opcode. First letter is usually the type: `a` reference,
  `i` int, `l` long, `f` float, `d` double.
- **`<operands>`** — either an immediate value, or a `#n` constant-pool index.
- **`// comment`** — `javap` resolving the `#n` for you. This is where you read class
  and method names.

Above each method body, `javap -v` prints `stack=`, `locals=`, `args_size=`. `locals`
is the size of the local variable array; `args_size` counts `this` for instance
methods.

### Instruction cheat sheet

```
aload_0            push 'this' (instance methods) / first ref param (static)
iload_N / istore_N int local read / write, slot N
ldc  #n            push a constant-pool constant (String, int, Class, MethodHandle)
bipush / sipush    push a small int immediate (byte / short range)
getfield / putfield        instance field read / write   (consumes the objectref)
getstatic / putstatic      static field read / write
new / dup / invokespecial  the three-instruction allocate-and-construct sequence
checkcast          the cast erasure forced the compiler to insert
anewarray + aastore        a varargs call site building its array
invokestatic       static method
invokespecial      constructor, private method, super.method()
invokevirtual      instance method, class-typed receiver
invokeinterface    instance method, interface-typed receiver
invokedynamic      lambda / method ref / string concat / record methods / pattern switch
athrow             throw; try/catch itself is an EXCEPTION TABLE, not an instruction
```

### Desugaring lookup

| See this | It came from |
|---|---|
| `Integer.valueOf` / `intValue` | autoboxing / unboxing (01) |
| `checkcast` after a generic read | erasure (06) |
| a second method with `ACC_BRIDGE, ACC_SYNTHETIC` | a bridge method (06) |
| `invokedynamic ... makeConcatWithConstants` | `+` on strings (18) |
| `invokedynamic` + `lambda$...` synthetic method | a lambda or method reference (21) |
| `iterator` / `hasNext` / `next` / `checkcast` | enhanced `for` over an `Iterable` |
| `arraylength` + `iaload`, no iterator | enhanced `for` over an array |
| `anewarray` + `aastore` at a call site | a varargs call |
| `lookupswitch` on `hashCode`, then `equals` | `switch` on `String` |
| a `$SwitchMap$...` static int array | `switch` on `enum` |
| an exception table + `addSuppressed` | try-with-resources (08) |
| a synthetic `this$0` field | a non-static inner / anonymous class — **retention risk, Topic 79** |
| `RuntimeVisibleAnnotations` attribute | your annotations, inert, as data (40) |

### Gotchas checklist

- [ ] **Your listing is the truth.** Not any document's, including this one.
- [ ] Always `-p`. Synthetic and private members are where the interesting things are.
- [ ] Check the class-file major version before reasoning about any listing.
- [ ] `javap` answers "what did the language do", never "what runs".
- [ ] Instruction count is not a cost model.
- [ ] `javac` does no inlining, no escape analysis, no dead-code elimination.
- [ ] An allocation in bytecode may not happen at runtime (Topic 75).
- [ ] Bytecode size **is** an input to inlining — the one legitimate perf link.
- [ ] Never depend on lambda identity or on synthetic method names.
- [ ] `static final` constants are inlined into callers; partial rebuilds go stale.
- [ ] `javap` is a static, offline, zero-risk tool. It never touches a running JVM.

---

## When would I use this at work?

**1. Settling a code-review argument in sixty seconds.**
Someone claims `+` in a loop creates a `StringBuilder` per iteration and blocks the PR.
You run `javap -c` and paste the relevant lines: one `invokedynamic`, no
`StringBuilder`, and here is the `BootstrapMethods` entry naming `StringConcatFactory`.
The discussion ends, with evidence, and — critically — you also name the real defect,
which is the quadratic accumulation nobody mentioned. Reviews that end in evidence
rather than seniority are how a team's technical culture actually improves.

**2. Diagnosing a stale constant after a partial deploy.**
"We changed `MAX_BATCH_SIZE` from 500 to 900, deployed, and the batch size is still
500." Everyone starts looking at configuration and caches. You `javap -c` the *caller*
class from the deployed jar, see `sipush 500` baked into its constant pool, and know
within a minute that the caller was not recompiled. This class of bug is invisible to
every other tool you own, because there is no configuration, no cache and no log line —
the wrong value is literally in the binary.

**3. Understanding an unfamiliar framework instead of guessing at it.**
You are debugging why a Spring bean is not being advised, or why Jackson cannot bind a
constructor parameter, or why a Hibernate entity behaves strangely. `javap -p -v` on
the actual deployed class shows you the annotations that are physically present, the
`MethodParameters` attribute (or its absence — that is the `-parameters` flag, and its
absence is a real Jackson/Spring binding failure mode), the bridge methods, and whether
a class is `final`. That is *ground truth about the artefact*, obtained without
attaching to anything, which makes it usable during an incident when attaching a
profiler to production is a conversation you do not have time for.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing:** `Integer.valueOf` / `intValue` were your first bytecode reading. This
  topic generalises it.
- **06 — Type erasure and bridge methods:** `checkcast` and `ACC_BRIDGE` are the
  mechanical evidence for everything that topic claimed.
- **18 — Strings:** `+` compiles to `invokedynamic` bound to `StringConcatFactory`. The
  `BootstrapMethods` recipe string is where you see how the expression was partitioned.
- **21 — Lambdas:** `invokedynamic` + `LambdaMetafactory`, one shared instance for
  non-capturing lambdas, no class file. The drill in this document is that topic's
  proof.
- **40 — Proxying:** `invokespecial` for `private` methods is half the reason a private
  `@Transactional` method can never be advised. Annotations as inert
  `RuntimeVisibleAnnotations` bytes is the other half of that topic's mechanism.
- **66 — JVM architecture:** frames, the per-thread stack, the method area. This topic
  is the frame's contents, in detail.
- **67 — Class loading:** every anonymous class in your codebase is a class-loading
  event. The drill's `ls *.class` is that cost, counted.
- **74 — JIT tiered compilation:** the reason `javap` cannot answer performance
  questions. Read them adjacently.
- **75 — Escape analysis and inlining:** why an allocation visible in bytecode may not
  happen, and why bytecode **size** is the one legitimate performance link.

**This unlocks:**

- **77 — JMH:** the tool for the question `javap` cannot answer. Every "is A faster
  than B" in this document forwards to it.
- **78 — Profiling:** where the emitted code actually spends time, and where synthetic
  `lambda$...` frames show up in a flame graph — you will recognise them now.
- **79 — Memory leaks:** the synthetic `this$0` field on a non-static inner class is one
  of the four classic leak shapes, and you can now *see* it in a class file.
- **80 — Off-heap and FFM:** the FFM API is built on `MethodHandle`s and `invokedynamic`
  machinery; downcall stubs are linked the same way lambdas are.
- **81 — Instrumentation agents:** an agent's `ClassFileTransformer` rewrites exactly
  the bytes you have been reading. `javap` on a dumped transformed class is how you see
  what an APM agent did to your code.
- **83 — GraalVM native image:** closed-world analysis must resolve every
  `invokedynamic` at build time, which is precisely why lambdas, proxies and reflection
  need reachability metadata there.
- **96 — False sharing:** field layout in the class file versus the JVM's actual
  reordering at runtime — another case where the class file is not the final word.
- **101 — Virtual threads:** continuations and stack unmounting operate on frames, which
  are the structure this topic describes.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted, each with a settling command given inline: the exact number
and order of static arguments in a `BootstrapMethods` entry (`javap -v -p`), the default
values of `MaxInlineSize` / `FreqInlineSize` / `MaxInlineLevel` on your JVM
(`java -XX:+PrintFlagsFinal -version | grep -i inline`), and the exact wording of
`PrintInlining` verdicts (grep case-insensitively and read your own output). Instance
caching for non-capturing lambdas is current implementation behaviour and not a spec
guarantee — never depend on lambda identity. Everything else here — the stack machine,
the five invoke instructions, `invokedynamic` bootstrap-and-link, the desugaring table,
and the fact that `javac` performs essentially no optimisation — is spec-level and has
been stable for many releases. And above all: **when your listing disagrees with this
document, your listing is right.***
