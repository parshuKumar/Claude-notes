# 00 — Java Deep Mastery: Master Plan (v2)

> **Re-read this file before generating any topic doc.** Numbering, terminology,
> the project spine, and the drill assignments are all defined here and must stay
> consistent across sessions.

---

## A. STACK VERIFICATION (performed 2026-08-20, from primary sources)

§0.1 required me to check whether Spring Boot 4 / Spring Framework 7 are GA
before Topic 01. They are. Here is the verified ground truth:

| Thing | Status as of 2026-08-20 | Source |
|---|---|---|
| Spring Framework 7.0 | **GA — 13 Nov 2025**. JDK 17 baseline, JDK 25 recommended. Jakarta EE 11 baseline. Kotlin 2.2, GraalVM 25. | spring.io blog |
| Spring Boot 4.0.0 | **GA — 20 Nov 2025** | spring.io blog |
| Spring Boot 4.1.0 | **GA — 10 Jun 2026** (current feature line) | spring.io blog |
| Spring Boot 3.5.x | **OSS support ENDED June 2026** (3.5.16 was the last OSS release). Commercial support only. | spring.io support policy |
| Java 21 | LTS, Sept 2023 | openjdk.org |
| Java 25 | **LTS, GA 16 Sept 2025** — current LTS | openjdk.org / Oracle |
| Java 26 | GA 17 Mar 2026, **non-LTS** | openjdk.org |

### The delta you need to decide on

Your brief pinned **Spring Boot 3.x / Framework 6.x**. That target is now
**out of OSS support**. This is not a style preference — it means no free CVE
patches. The honest recommendation is to move the target.

**Boot 3.x → Boot 4.x, the parts that actually change how you write code:**

1. **Codebase modularisation.** Boot 4 splits into many smaller, focused jars.
   Starters still exist, but the artifact you depend on for a given feature may
   have moved (e.g. MongoDB health indicators moved from `spring-boot-data-mongodb`
   to `spring-boot-mongodb`, with package renames). Affects Phase 3 and 5 docs.
2. **Jackson 3 is standard; Jackson 2 support is deprecated.** Package and
   configuration differences. Affects Phase 1 (serialization) and Phase 5.
3. **JSpecify null-safety annotations portfolio-wide.** This is the closest Java
   gets to your `strictNullChecks`, and it is now a first-class Spring concern.
   Affects Phase 2 (Optional) and Phase 5.
4. **HTTP Service Clients auto-configured** — annotated Java interfaces as HTTP
   clients, the successor story to Feign/`RestTemplate`. Affects Phase 11.
5. **API versioning** built into MVC/WebFlux (`spring.mvc.apiversion.*`).
6. **`JmsClient`**, **`RestTestClient`**, an **OpenTelemetry starter**, Redis
   master/replica auto-config, Gradle 9 support.
7. **Spring Security 7.0**, **Spring Data 2025.1** ride along — Security 7 changes
   filter-chain configuration idioms. Affects Topics 56–57.
8. `javax` → `jakarta` is **already done** in Boot 3; Boot 4 does not redo it.
   You still need to know it cold — it is the single most-asked migration
   question in interviews, and it is Topic 127.

**Your options — reply with one, or just type START to accept the default:**

| Option | Stack | Why you'd pick it |
|---|---|---|
| **A (default, recommended)** | Java 21 language baseline, **run on JDK 25**, Spring Boot 4.1 / Framework 7.0, Jakarta EE 11 | Supported, current, and what a 2026 greenfield service is built on. Interviewers at good companies are running this. |
| **B** | Same as A but Boot 3.5.x / Framework 6.2 | Only if your target employers are demonstrably on 3.x and you want the interview surface to match exactly. Cost: you learn an EOL line, and I have to caveat every Phase 5/11 doc. |
| **C** | A, plus a standing "Boot 3.x delta" callout box in every Spring topic | Best of both, ~10% more doc length. Pick this if you're job-hunting into a market that still runs 3.x — which, realistically, most of it does. |

**I recommend C.** The industry lags GA by 18–36 months; you will interview at
companies on 3.5 and possibly 2.7. Learning 4.x as the mental model with an
explicit "on 3.x this is different because X" box costs little and makes you the
person in the room who knows *why* the difference exists.

### Version-labelling convention used in every doc

- Unlabelled code = **compiles and runs on Java 21**.
- `[JAVA 25]` = requires 25; the 21-compatible fallback is shown immediately after.
- `[LEGACY — still asked]` = Java 8-era idiom, not current, but interviewers ask it.
- `[BOOT 3.x DELTA]` = how this differs on the EOL 3.x line (present if you pick C).

---

## B. PHASES OVERVIEW AND WHY THIS ORDER

The conventional Java syllabus teaches concurrency at position 4 and JVM
internals at position 9 or never. That order produces people who can recite
`volatile` and have never seen a GC log. This order is different, deliberately:

| # | Phase | Topics | Why it sits here |
|---|---|---|---|
| 1 | Core Language | 01–20 | Nothing else is expressible without the type system, generics, and collections. Erasure and the `equals`/`hashCode` contract are load-bearing for everything downstream. |
| 2 | Modern Java | 21–30 | **Before concurrency, not after.** `CompletableFuture` composition is unteachable without lambdas and functional interfaces. Sealed types + pattern matching are also the honest bridge from your TS discriminated unions, and that bridge is worth building early. |
| 3 | Build & Supply Chain | 31–34 | You cannot run a Spring app you cannot build, and Maven's flat nearest-wins resolution will confuse you badly coming from npm. Short phase, high leverage. |
| 4 | Spring Core | 35–41 | **The project spine starts here.** Container mechanics before Boot magic — otherwise auto-configuration is unfalsifiable belief. |
| 5 | Spring Boot & Persistence | 42–57 | The bulk of day-one senior Java work. Hibernate gets six topics because persistence-context semantics are where most production Java bugs actually live. |
| 6 | Testing | 58–64 | After the service exists, so tests test something real. Mutation testing here rather than later because it changes how you write the tests in Phases 8–11. |
| 7 | **GATE — service under load** | 65 | Hard gate. Everything after this profiles *your* running system. |
| 8 | JVM Internals | 66–83 | **After** a load-testable service. Reading a heap dump with nothing to dump is memorisation. |
| 9 | Concurrency | 84–102 | After JVM internals, because the JMM is a statement about caches, barriers and the compiler's reordering licence — all of which you meet in Phase 8. Virtual-thread pinning is also a JIT/safepoint story. |
| 10 | Reactive & Async at Scale | 103–108 | After concurrency, so "backpressure" is a scheduling fact and not a metaphor. This is the phase closest to what you already know from Node. |
| 11 | Distributed & Production | 109–124 | Your system-design knowledge transfers; what's new is Java's specific failure shapes (pool-vs-pool deadlock, MDC across threads, rebalance storms). |
| 12 | **Principal track** | 125–135 | Judgment, not APIs. Phases 1–11 make you a strong Senior. This phase is a different axis entirely. |

### Honest pacing (assumption: ~10 hrs/week, and you complete the exercises)

The assumption matters. These numbers are for *doing the drills*, not reading the
docs. Reading-only is roughly 40% of this and produces roughly 20% of the
retention.

| Phase | Topics | Weeks @10h/wk | Notes |
|---|---|---|---|
| 1 | 20 | 5–7 | Fast — you know OOP. Slow points: erasure/variance (05–07), HashMap internals (12). |
| 2 | 10 | 3–4 | Fastest phase. Streams ≈ your array methods; the surprises are laziness and parallel-stream cost. |
| 3 | 4 | 1–2 | Short. Dependency resolution is the only real cognitive load. |
| 4 | 7 | 2–3 | Proxying (40) is worth two sessions on its own. |
| 5 | 16 | 7–10 | Longest CORE phase. Hibernate (48–53) is where people plateau; budget for it. |
| 6 | 7 | 2–3 | |
| 7 | 1 | 1–2 | Infrastructure work. Don't rush it — everything downstream depends on it. |
| 8 | 18 | 9–13 | The hardest phase. Drills dominate the time, not reading. |
| 9 | 19 | 9–13 | Comparable difficulty. jcstress runs are slow; plan around them. |
| 10 | 6 | 3–4 | Eased by your RxJS/Promise background. |
| 11 | 16 | 6–9 | Your system-design background cuts this substantially. |
| 12 | 11 | 6–10 | Gated by artefact production and review cycles, not by reading. Cannot be rushed. |
| **Total** | **135** | **54–81** | **≈ 13–19 months.** If someone told you six months, they were selling something. |

**Fastest honest path to "clears senior Java loops":** Phases 1–9 plus Topics
109, 118, 119, 121 ≈ 40–58 weeks. Phase 10–11 completeness and Phase 12 are the
difference between passing a loop and being the person they promote.

---

## C. THE PROJECT SPINE

**System:** `orderflow` — an order-and-payment service.
Domain: users, products, inventory, orders, payments, wallets. No `foo`, no `bar`.

**Starts:** Topic 35 (Phase 4).
**Must be load-testable:** by Topic 65 (Phase 7 gate).
**Every topic from 35 onward** either adds a capability to it or is used to
diagnose it. The per-topic `Spine:` line below states which.

### Capability at each phase gate

| Gate | The service can | The service **cannot** |
|---|---|---|
| End of Phase 4 (T41) | Wire beans, run in-memory, place an order against a stubbed inventory | Persist anything, serve HTTP under load, survive a restart |
| End of Phase 5 (T57) | Full REST API, Postgres persistence, transactional order placement with wallet debit, JWT auth, optimistic locking on inventory | Prove any of it correct; survive concurrency it hasn't been tested for |
| End of Phase 6 (T64) | Prove behaviour with slice tests, Testcontainers Postgres/Kafka, mutation-tested critical paths | Handle production load; you have no numbers |
| **Phase 7 gate (T65)** | Run containerised with a realistic dataset (≥1M orders, ≥100k products) under k6/Gatling load, with recorded baseline p50/p95/p99 | Survive GC pressure, thread exhaustion, or connection-pool contention — you'll discover exactly this in Phases 8–9 |
| End of Phase 8 (T83) | Be profiled, heap-dumped, GC-tuned; you know its allocation rate and where its CPU goes | Handle concurrent inventory contention correctly |
| End of Phase 9 (T102) | Serve thread-per-request on virtual threads, with correct lock ordering, bounded pools, and jcstress-verified critical sections | Backpressure a slow downstream |
| End of Phase 10 (T108) | Offer a WebFlux path for the payment-callback fan-in with real backpressure | Survive a downstream outage or a Kafka rebalance |
| End of Phase 11 (T124) | Circuit-break, outbox-publish, idempotently consume, trace end-to-end, expose RED metrics, pass k8s readiness honestly | — this is a production-shaped service |
| Phase 12 (T135) | — | The artefacts (RFC, migration plan, capacity model, postmortem) are the deliverable, not the code |

---

## D. RETENTION CADENCE (mandatory, §5)

- **Every 5 topics** (i.e. after 05, 10, 15, …): cumulative retrieval checkpoint.
  Questions drawn from **all** prior topics, weighted toward the oldest.
  Questions only. Answers withheld until you attempt them.
- **Every phase gate**: integration exercise combining ≥3 topics from that phase
  applied to `orderflow`, plus one cross-phase question.
- **Spaced return**: at every phase gate I re-ask two questions from **two phases
  back**. Weak answers get named as weak and pointed at the specific doc.

---

## E. CURRICULUM — 135 TOPICS

Legend: `FOUNDATION` · `CORE` · `DIFFERENTIATOR` · `ELITE`
Every file lives at `/docs/java/NN-slug.md`.
`Drill:` marks a deliberate-breakage exercise (mandatory for DIFFERENTIATOR/ELITE
per §4; `[BONUS]` ones are high-value CORE drills I'm adding anyway).

---

### PHASE 1 — Core Language `01–20`

**01 · Primitives, wrappers, autoboxing, and the Integer cache** `FOUNDATION` → `01-primitives-wrappers-autoboxing.md`
- **Mastery:** you predict which of two `Integer` `==` comparisons is true without running it, you know boxing is an *allocation* and can say what that does to allocation rate in a hot loop, and you reach for a primitive-specialised structure before someone profiles it for you.
- **Mid → Senior:** "autoboxing converts int to Integer automatically" → "boxing allocates unless it hits the −128..127 cache, which is why `==` on boxed values is a latent bug and why `Map<Long,Long>` counters are an allocation-rate problem — but I'd confirm with JMH and an allocation profile before rewriting anything."

**02 · Nominal vs structural typing, and how it reshapes interface design** `FOUNDATION` → `02-nominal-vs-structural-typing.md`
- **Mastery:** you stop writing TypeScript-shaped "just needs these fields" contracts and can explain why Java forces you to name a type, and what that costs (adapter boilerplate) and buys (compile-time identity, no accidental compatibility).
- **Mid → Senior:** "Java checks types by name" → "structural typing lets a type satisfy a contract by accident; nominal typing makes conformance a deliberate declaration, which is why Java codebases have adapters where TS has none, and why `implements` is a design commitment rather than a shape assertion."

**03 · Access modifiers, packages, and encapsulation boundaries** `FOUNDATION` → `03-access-modifiers-packages.md`
- **Mastery:** you use package-private deliberately as a module boundary, not as "I forgot to write public".
- **Mid → Senior:** "public, private, protected, default" → "package-private is the only real intra-artifact boundary before JPMS; I use it to keep implementation classes out of a published API surface, because everything public is a support obligation."

**04 · Interfaces vs abstract classes; default, static, private methods; diamond resolution** `FOUNDATION` → `04-interfaces-abstract-classes.md`
- **Mastery:** you can state the exact resolution order when a class inherits the same default method from two interfaces, and you know why `default` was added (interface evolution without breaking implementors) rather than treating it as multiple inheritance.
- **Mid → Senior:** "interfaces can have default methods now" → "`default` exists so `Collection` could gain `stream()` without breaking every implementor on earth; class wins over interface, most-specific interface wins, and ambiguity is a compile error you resolve with `Interface.super.method()`."

**05 · Generics I — declaration, bounded types, generic methods** `CORE` → `05-generics-declaration-bounds.md`
- **Mastery:** you write a generic method with a recursive bound (`<T extends Comparable<T>>`) without copying it from Stack Overflow, and you know when a bound belongs on the class vs the method.
- **Mid → Senior:** "generics give you type safety" → "the bound is the contract; putting it on the method rather than the class keeps the type variable's scope minimal, which is what makes the API usable without the caller naming types."

**06 · Generics II — type erasure, bridge methods, reifiable types** `CORE` → `06-type-erasure.md`
- **Mastery:** you can explain why `List<String>` and `List<Integer>` are the same class at runtime, why you cannot write `new T[]`, why `instanceof List<String>` doesn't compile, and what a bridge method is doing in `javap` output.
- **Mid → Senior:** "generics are erased at runtime" → "erasure was a source-and-binary-compatibility decision for Java 5; the compiler inserts casts and synthesises bridge methods to preserve polymorphism after erasure, which is why generic arrays are unsound and why heap pollution warnings exist."

**07 · Generics III — variance, PECS, wildcard capture** `CORE` → `07-variance-pecs-wildcards.md`
- **Mastery:** you reach for `? extends`/`? super` from the call-site's needs, not from a mnemonic, and you can explain why `List<String>` is not a `List<Object>` while `String[]` *is* an `Object[]` (and why that array behaviour is a design mistake).
- **Mid → Senior:** "Producer Extends, Consumer Super" → "Java generics are invariant because erasure gives no runtime type check; arrays are covariant and pay for it with `ArrayStoreException`. Wildcards restore the variance you need at the use site, and capture is what lets the compiler reason about the unnamed type."

**08 · Exceptions — checked vs unchecked, hierarchy, try-with-resources** `CORE` → `08-exceptions-fundamentals.md`
- **Mastery:** you know what `Throwable`/`Error`/`Exception`/`RuntimeException` each mean *as a contract*, you use try-with-resources reflexively, and you can explain suppressed exceptions and why they exist.
- **Mid → Senior:** "checked exceptions must be caught or declared" → "the hierarchy encodes recoverability: `Error` means the JVM is compromised, checked means the caller can plausibly act, unchecked means a programming defect. try-with-resources exists because the naive finally-close pattern loses the original exception — that's what `getSuppressed()` recovers."

**09 · Exception design — when checked exceptions damage an API** `CORE` → `09-exception-api-design.md`
- **Mastery:** you can argue both sides of the checked-exception debate with real consequences, and you design a domain exception hierarchy for `orderflow` that survives contact with a REST layer.
- **Mid → Senior:** "we wrap checked exceptions in runtime exceptions" → "checked exceptions don't compose through lambdas, they leak implementation into signatures, and they get swallowed under deadline pressure. Spring made the whole data-access hierarchy unchecked for exactly that reason. I use checked only where the caller has a genuine alternative action."
- **Drill [BONUS]:** swallow an exception in a `catch` block, then find it in production only via a missing side-effect — proves why "log and continue" is a defect.

**10 · Collections Framework — interfaces, contracts, iteration, fail-fast** `FOUNDATION` → `10-collections-framework.md`
- **Mastery:** you pick a collection from its contract (ordering, duplicates, null policy, iteration cost), and you know why `ConcurrentModificationException` is best-effort rather than a guarantee.
- **Mid → Senior:** "use ArrayList for lists" → "the interface is the contract and the implementation is the performance decision; fail-fast is a debugging aid backed by a non-volatile modCount, so it detects most bugs and guarantees nothing."

**11 · List implementations — ArrayList, LinkedList, ArrayDeque, and cache reality** `CORE` → `11-list-implementations.md`
- **Mastery:** you can explain why `LinkedList` loses to `ArrayList` even for middle insertion at realistic sizes, in terms of cache lines and pointer chasing rather than Big-O.
- **Mid → Senior:** "LinkedList is O(1) insertion" → "O(1) after you've paid O(n) to walk there, and each hop is a likely cache miss. `ArrayDeque` beats both for queue/stack use. Big-O ranks algorithms; memory hierarchy decides the wall clock."

**12 · HashMap internals — hashing, spreading, buckets, resize, treeification** `CORE` → `12-hashmap-internals.md`
- **Mastery:** you can walk the `get` path from `hash()` spreading through bucket index to node traversal, explain why the table is a power of two, why the spread XORs the high bits down, and what happens at the 8-node treeify / 6-node untreeify thresholds.
- **Mid → Senior:** "HashMap uses buckets and hashCode" → "index is `(n-1) & hash` so the table must be a power of two, which means only the low bits matter — hence the high-bit XOR spread. Java 8 treeifies a bucket at 8 nodes to bound worst-case collision attacks at O(log n), and resize at load factor 0.75 rehashes by splitting each bin into lo/hi lists without recomputing hashes."

**13 · The equals/hashCode contract and silent HashMap corruption** `CORE` → `13-equals-hashcode-contract.md`
- **Mastery:** you can produce, on demand, a class whose instances vanish from a `HashSet`, and explain exactly which clause of the contract you violated.
- **Mid → Senior:** "equal objects must have equal hash codes" → "breaking it doesn't throw — it silently loses data, because lookup goes to the wrong bucket. The nastier variant is a mutable key: mutate a field after insertion and the entry is unreachable but still retained, which is both a correctness bug and a leak."
- **Drill [BONUS]:** put a `Product` in a `HashSet`, mutate its `sku`, then fail to find it and fail to remove it. Show `size()` still counts it.

**14 · Comparable vs Comparator; TreeMap, TreeSet, NavigableMap** `CORE` → `14-comparable-comparator-treemap.md`
- **Mastery:** you know why a `Comparator` inconsistent with `equals` breaks `TreeMap` but not `HashMap`, and you use `Comparator.comparing().thenComparing().reversed()` fluently without mis-ordering `reversed()`.
- **Mid → Senior:** "Comparable is natural order, Comparator is custom" → "`TreeMap` defines equality by `compareTo() == 0`, not `equals()`. If those disagree your map violates the `Map` contract in ways that only show up in the sorted structure. Also: comparator must be a total order, or `Arrays.sort` throws 'Comparison method violates its general contract'."

**15 · LinkedHashMap and an LRU cache** `CORE` → `15-linkedhashmap-lru.md`
- **Mastery:** you build a bounded LRU in six lines with `accessOrder=true` and `removeEldestEntry`, and you can say why that cache is not safe to share across threads and what you'd use instead.
- **Mid → Senior:** "LinkedHashMap preserves insertion order" → "it also supports access order, which makes it a one-class LRU. In production I'd use Caffeine — because eviction policy, TTL, stats and concurrency are the actual requirements, and W-TinyLFU beats LRU on real access patterns."

**16 · Sets, EnumMap, EnumSet, and choosing the right collection** `CORE` → `16-sets-enum-collections.md`
- **Mastery:** you use `EnumSet`/`EnumMap` reflexively for enum keys and can explain the bitvector/array-indexed representations that make them beat `HashSet`/`HashMap` outright.
- **Mid → Senior:** "EnumSet is a set of enums" → "it's a long bitvector for ≤64 constants, so contains/add are bit ops with zero hashing and zero allocation per entry; `EnumMap` is a plain array indexed by ordinal. There is no reason to use `HashMap<OrderStatus, X>`."

**17 · Immutability — `final` semantics, defensive copying, safe publication** `CORE` → `17-immutability-final-safe-publication.md`
- **Mastery:** you know `final` is shallow, that a `final` field has *memory-model* meaning (a freeze at construction end) and not just "can't reassign", and you defensively copy at both the constructor and the getter.
- **Mid → Senior:** "final means it can't be changed" → "`final` prevents reassignment of the reference, not mutation of the target. Its real power is JMM: correctly-constructed final fields are guaranteed visible to other threads without synchronisation — which is the entire basis of safe publication. This is Topic 88, and it has no TypeScript equivalent."

**18 · Strings — immutability, the pool, `intern()`, StringBuilder, compact strings** `CORE` → `18-strings-pool-stringbuilder.md`
- **Mastery:** you can explain why `new String("x") != "x"`, when `intern()` helps and when it just moves a leak, and what the compiler already does with `+` in a loop versus outside one.
- **Mid → Senior:** "use StringBuilder instead of +" → "javac compiles `+` to `StringConcatFactory` invokedynamic since 9; the problem isn't `+`, it's `+` *inside a loop*, which allocates a new builder per iteration. `intern()` moves strings to a native-backed pool and can hurt: you trade heap for a lookup and a GC-root-ish retention."

**19 · Serialization — Java serialization's CVE history, Jackson, schema evolution** `CORE` → `19-serialization-and-its-hazards.md`
- **Mastery:** you can explain *why* `readObject` on untrusted input is remote code execution and not merely "unsafe", why `serialVersionUID` is a compatibility contract, and you default to a schema-evolving format for anything crossing a service boundary.
- **Mid → Senior:** "don't deserialize untrusted data" → "Java serialization reconstructs arbitrary object graphs by invoking `readObject` on classes you didn't choose; gadget chains in common libraries turn that into RCE, which is why the JDK added serialization filters and why the platform is walking away from it. For anything cross-service I use protobuf/Avro because I need *schema evolution*, not just encoding."
- **Drill:** write a class with a `readObject` that has a side effect, deserialize it, observe the side effect fire before any of your code runs. Then enable a serialization filter and watch it be rejected.

**20 · JPMS modules and `jlink`** `CORE` → `20-jpms-modules-jlink.md`
- **Mastery:** you know why JPMS exists (strong encapsulation + a reliable configuration, not "smaller jars"), why almost nobody uses it for applications, and why `jlink` still matters for runtime images.
- **Mid → Senior:** "modules are Java 9's module system" → "JPMS gives compile- and run-time strong encapsulation the classpath never had, but it's all-or-nothing across a dependency graph, which is why adoption stalled outside the JDK itself. I'd use it for a library with a real API surface, and `jlink` for a minimal container runtime image."

---

### PHASE 2 — Modern Java `21–30`

**21 · Lambdas and functional interfaces** `CORE` → `21-lambdas-functional-interfaces.md`
- **Mastery:** you know a lambda is not an anonymous class (it's an `invokedynamic` call site linked by `LambdaMetafactory`), you know which lambdas allocate and which are cached, and you name the right interface from `Function`/`Predicate`/`Supplier`/`Consumer`/`BiFunction` without looking.
- **Mid → Senior:** "lambdas are shorthand for anonymous classes" → "they compile to `invokedynamic`, not to an inner class, so a non-capturing lambda is a singleton and a capturing one allocates per evaluation. That distinction matters in a hot path, and it's visible in `javap -c`."

**22 · Method references — all four forms** `CORE` → `22-method-references.md`
- **Mastery:** you can name which of the four forms `String::length` is (unbound instance) versus `order::total` (bound), and know why the unbound form shifts the receiver into the first parameter.
- **Mid → Senior:** "`::` is shorthand for a lambda" → "there are four forms and the unbound-instance form is the one that confuses people, because the receiver becomes an implicit first argument. Also `new` references let you pass a constructor as a `Supplier`/`Function`, which is how collectors get their containers."

**23 · Streams I — pipeline structure, laziness, intermediate vs terminal** `CORE` → `23-streams-pipeline-laziness.md`
- **Mastery:** you can state what has and hasn't executed at the point a stream is built but not terminated, explain short-circuiting, and know why a stream cannot be reused.
- **Mid → Senior:** "streams are like array methods" → "your array methods are eager and materialise an intermediate array per step; a stream is a lazily-fused pipeline with a single pass and short-circuiting. That's why `findFirst` after `map` doesn't map the whole collection, and it's the main reason streams aren't just prettier loops."

**24 · Streams II — collectors, grouping, custom collectors, teeing** `CORE` → `24-collectors.md`
- **Mastery:** you write a custom `Collector` (supplier/accumulator/combiner/finisher) and can say what the combiner is for and why it's only called under parallelism.
- **Mid → Senior:** "use `Collectors.toList()`" → "the four functions of a `Collector` are a mutable-reduction contract; the combiner only runs in parallel, which is why a broken combiner is a bug that only appears under load. `groupingBy` with a downstream collector replaces most of the loop code people still write."

**25 · Parallel streams — when they help and when they actively hurt** `DIFFERENTIATOR` → `25-parallel-streams.md`
- **Mastery:** you can name the three preconditions (splittable source, sufficient N × per-element cost, no shared mutable state / associative reducer) and you know a blocking task on the common ForkJoinPool degrades unrelated code in the same JVM.
- **Mid → Senior:** "add `.parallel()` for speed" → "it uses the *common* pool, sized to cores−1 and shared with everything in the JVM. Blocking in it starves unrelated parallel work — including some library internals. And `LinkedList` splits terribly while `ArrayList`/arrays split perfectly. I'd only reach for it with a CPU-bound, associative reduction over a well-splitting source, and I'd measure."
- **Drill:** submit a blocking sleep to the common pool, then measure an unrelated parallel stream's latency collapse in the same JVM. Fix by supplying a dedicated pool.

**26 · Optional — correct use, and why `get()` is null with extra steps** `CORE` → `26-optional.md`
- **Mastery:** you never call `get()`, never use `Optional` as a field or parameter, and can explain why it was designed as a return-type-only construct.
- **Mid → Senior:** "Optional avoids NPEs" → "it's a *return type* that makes absence part of the signature. As a field it's an extra allocation and breaks serialization; as a parameter it just moves the null check. `orElseGet` over `orElse` when the fallback is expensive, and `map`/`flatMap` rather than `isPresent`/`get` — otherwise it's an `if` with ceremony."

**27 · Records — semantics, equals/hashCode, compact constructors, serialization** `CORE` → `27-records.md`
- **Mastery:** you know records give you a *shallowly* immutable carrier with generated `equals`/`hashCode`/`toString`, that validation goes in the compact constructor, and that a record holding a `List` is not immutable unless you copy.
- **Mid → Senior:** "records are data classes" → "they're nominal tuples with a canonical constructor and structural equality. Shallow immutability is the trap: a record wrapping a mutable list is mutable. They also serialize via the canonical constructor, which finally makes deserialization respect invariants."

**28 · Sealed types vs TypeScript discriminated unions** `CORE` → `28-sealed-types.md`
- **Mastery:** you model `PaymentResult` as a sealed interface over records and rely on compiler exhaustiveness — and can explain what sealing buys that an enum or an abstract class doesn't.
- **Mid → Senior:** "sealed limits which classes can extend it" → "sealed + records + pattern matching is Java's algebraic data type. The value is exhaustiveness: adding a new `PaymentResult` variant becomes a compile error at every switch, which is exactly what your TS discriminated unions do. This is the closest true analogue in the whole language."

**29 · Pattern matching — `instanceof`, `switch`, record patterns, exhaustiveness** `CORE` → `29-pattern-matching.md`
- **Mastery:** you deconstruct nested records in a switch, use guarded patterns (`when`), and know why the compiler requires a `default` unless the type is sealed.
- **Mid → Senior:** "pattern matching avoids casts" → "it moves type-narrowing into the language the way TS's control-flow narrowing does, and combined with sealed types it gives compile-checked exhaustiveness. Dominance ordering matters: a more general pattern before a specific one is a compile error, not a silent shadow." `[JAVA 25]` primitive patterns noted with the 21 fallback.

**30 · Text blocks, `var`, and where ergonomics hurt readability** `CORE` → `30-text-blocks-var.md`
- **Mastery:** you use `var` where the initialiser already names the type and refuse it where it hides a type the reader needs — and you can articulate the rule rather than following taste.
- **Mid → Senior:** "`var` is like TS type inference" → "`var` is local-only and has no `let`/`const` distinction — `final var` is the const. The readability rule is: keep it when the right-hand side names the type (`var order = new Order()`), drop it when the type is the information (`var result = service.process()`)."

---

### PHASE 3 — Build & Supply Chain `31–34`

**31 · Maven — POM, scopes, lifecycle phases, multi-module** `CORE` → `31-maven-fundamentals.md`
- **Mastery:** you can say what `mvn package` actually runs in order, what `provided` vs `runtime` vs `test` scope changes about the classpath, and why the reactor build order is a graph and not the order in `<modules>`.
- **Mid → Senior:** "`mvn clean install` builds it" → "`install` writes to your local repo, which hides broken releases — CI should use `verify`. Scopes are classpath membership decisions: `provided` means the container supplies it at runtime, and getting that wrong is a `NoClassDefFoundError` at deploy, not at build."

**32 · Dependency resolution — nearest-wins vs npm's nested tree** `CORE` → `32-dependency-resolution-boms.md`
- **Mastery:** you read `mvn dependency:tree` output fluently, you know Maven puts exactly **one** version of an artifact on a flat classpath (unlike npm's per-package nesting), and you use a BOM rather than pinning versions individually.
- **Mid → Senior:** "there's a version conflict" → "npm nests, so two versions coexist; Maven flattens with nearest-wins, so a transitive bump silently changes behaviour with no error. That's why `dependencyManagement`/BOM exists and why `NoSuchMethodError` at runtime is almost always a resolution problem, not a code problem."
- **Drill [BONUS]:** force a diamond conflict, produce a `NoSuchMethodError` at runtime with a clean compile, then resolve it with `dependencyManagement`.

**33 · Gradle — and when a team should choose it** `CORE` → `33-gradle.md`
- **Mastery:** you can state the honest trade (build speed, incrementality, configuration cache vs. a programmable build that drifts) and read a Kotlin-DSL build file without the plugin docs open.
- **Mid → Senior:** "Gradle is faster" → "it's faster because of incremental tasks, the build cache, and the configuration cache — and it costs you a build that is a program, which drifts. For a large multi-module repo the speed wins; for a single service, Maven's declarative rigidity is a feature."

**34 · Supply chain — reproducible builds, SBOM, CVE triage** `CORE` → `34-supply-chain-sbom-cve.md`
- **Mastery:** you can triage a reported transitive CVE by determining whether the vulnerable *code path* is reachable, rather than reflexively bumping and breaking things.
- **Mid → Senior:** "we run dependency scanning" → "a scanner reports presence, not exploitability. The triage is: is the vulnerable class on our classpath, is the affected method reachable from our entry points, and is the input attacker-controlled. Then bump, override the managed version, or document the exception — with an expiry date."

---

### PHASE 4 — Spring Core `35–41` — **project spine begins**

**35 · `ApplicationContext` as an IoC container** `CORE` → `35-application-context.md`
- **Mastery:** you can describe the two-phase startup (bean *definitions* registered, then singletons instantiated) and why that separation is what makes `@Bean` overriding, conditionals and proxying possible at all.
- **Mid → Senior:** "it's Spring's DI container, like Nest's" → "Nest builds a module graph from explicit imports; Spring registers `BeanDefinition` metadata first and only then instantiates, which is why `BeanFactoryPostProcessor` can rewrite definitions before anything exists. That phase split is the whole reason Boot's auto-configuration can back off conditionally."
- **Spine:** `orderflow` skeleton — `OrderService`, `InventoryService`, `WalletService` wired by a context, in-memory stubs.

**36 · Bean definition, registration, and component scanning vs Nest modules** `CORE` → `36-bean-definition-component-scan.md`
- **Mastery:** you know exactly which packages get scanned and why (`@SpringBootApplication`'s package and below), and you can explain the maintainability cost of implicit scanning versus Nest's explicit `imports`.
- **Mid → Senior:** "`@Component` makes it a bean" → "scanning is classpath-driven and implicit, so the dependency graph isn't visible in any file — the opposite of a Nest module. That's convenient until you need to know why a bean exists; then `@Bean` in an explicit `@Configuration` is worth the verbosity."
- **Spine:** package structure for `orderflow` — `catalog`, `inventory`, `orders`, `payments`, `wallet`.

**37 · Bean lifecycle, callbacks, and `BeanPostProcessor`** `CORE` → `37-bean-lifecycle.md`
- **Mastery:** you can order instantiation → populate → aware-callbacks → `BeanPostProcessor` before-init → `@PostConstruct` → `afterPropertiesSet` → post-init (where proxies are created) → destroy, and say which step wraps your bean in a proxy.
- **Mid → Senior:** "`@PostConstruct` runs after injection" → "the ordering matters because `BeanPostProcessor.postProcessAfterInitialization` is where AOP proxies get created. That's why a `@Transactional` method called from `@PostConstruct` isn't transactional — the proxy doesn't exist yet from that bean's own point of view."
- **Spine:** warm-up hook that preloads the product catalogue cache at startup.

**38 · Bean scopes vs Nest provider scopes; scoped proxies** `CORE` → `38-bean-scopes.md`
- **Mastery:** you can explain what actually goes wrong when a singleton depends on a request-scoped bean, and what `ScopedProxyMode` inserts to fix it.
- **Mid → Senior:** "there's singleton, prototype, request scope" → "injecting a shorter-scoped bean into a longer-scoped one captures one instance forever. Spring fixes it with a scoped proxy that resolves the real instance per call — Nest solves the same problem by bubbling REQUEST scope up the injection chain, which has a different cost profile."
- **Spine:** request-scoped correlation/tenant context on the order API.

**39 · Injection styles — constructor vs field vs setter; `@Qualifier`, `@Primary`, circular deps** `CORE` → `39-dependency-injection-styles.md`
- **Mastery:** you can give the three concrete reasons constructor injection won (final fields, testability without a container, immediate failure on a missing dependency) and know that a circular dependency is a design smell Spring merely *hides* for singletons.
- **Mid → Senior:** "use constructor injection, it's best practice" → "field injection lets you construct an invalid object and hides that the class has nine dependencies. Constructor injection makes the god-class visible and makes fields `final`, which gives you safe publication for free. Circular deps then fail loudly instead of being papered over by early references."
- **Spine:** refactor all `orderflow` services to constructor injection; introduce a `PaymentGateway` interface with two implementations selected by `@Qualifier`.

**40 · Proxying mechanics — JDK dynamic proxies vs CGLIB, and the self-invocation trap** `DIFFERENTIATOR` → `40-proxying-jdk-cglib-self-invocation.md`
- **Mechanical statement:** Spring hands callers a *different object* than the one you wrote. A JDK proxy implements the same interfaces and dispatches to an `InvocationHandler`; a CGLIB proxy is a runtime-generated subclass that overrides non-final methods. Either way, a call from `this` inside your bean goes straight to your own method and never touches the proxy.
- **Mastery:** you can predict whether a given bean gets a JDK or CGLIB proxy, why `final` and `private` methods are un-advisable, and diagnose "my `@Transactional`/`@Cacheable`/`@Async` did nothing" in under a minute.
- **Mid → Senior:** "Spring uses proxies for AOP" → "interface-based → JDK proxy; class-based → CGLIB subclass, so `final` classes and `final`/`private`/`static` methods can't be advised. And since advice lives on the proxy, `this.method()` bypasses it entirely — that's the self-invocation trap, and the fixes are self-injection, an injected `ObjectProvider`, extracting a collaborator, or AspectJ load-time weaving."
- **Spine:** none directly — this topic is the prerequisite that makes Topics 54–55 debuggable.
- **Drill:** annotate an internal method `@Transactional`, call it from a sibling method in the same class, write a row, throw — observe the row is **still there**. Then print `order.getClass().getName()` and see `$$SpringCGLIB$$`.

**41 · AOP — aspects, pointcuts, advice, ordering** `DIFFERENTIATOR` → `41-aop-aspects-pointcuts.md`
- **Mechanical statement:** an aspect is not a language feature; it is a `MethodInterceptor` in a chain that the proxy walks before delegating to your target, ordered by `@Order`.
- **Mastery:** you write a pointcut expression that matches what you meant (not "everything in `com.orderflow`"), and you can explain why aspect ordering matters when transactions and caching both apply.
- **Mid → Senior:** "aspects handle cross-cutting concerns" → "it's an interceptor chain on a proxy — the same shape as a Nest interceptor, except Nest wires it at the route boundary so self-calls aren't a problem. Order matters: if `@Cacheable` runs outside `@Transactional`, you can cache a value that later rolls back."
- **Spine:** audit-logging aspect over all `orderflow` command methods; timing aspect feeding Topic 118's metrics.
- **Drill:** put `@Cacheable` outside `@Transactional`, roll back the transaction, then serve the rolled-back value from cache. Fix by ordering.

---

### PHASE 5 — Spring Boot & Persistence `42–57`

**42 · Auto-configuration mechanics — `@Conditional` and `AutoConfiguration.imports`** `DIFFERENTIATOR` → `42-auto-configuration-mechanics.md`
- **Mechanical statement:** Boot reads `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` from every jar, registers each listed class as a *conditional* configuration, and evaluates `@ConditionalOnClass`/`OnMissingBean`/`OnProperty` against the current classpath and bean registry. Nothing is magic; it is a registry plus a predicate.
- **Mastery:** you can run the app with `--debug` and read the condition-evaluation report to say why a bean did or did not appear, and you write your own auto-configuration with a correct `@ConditionalOnMissingBean` back-off.
- **Mid → Senior:** "Boot configures things automatically" → "`AutoConfiguration.imports` lists candidates; conditions decide. `@ConditionalOnMissingBean` is what makes user beans win, and ordering (`@AutoConfigureBefore/After`) is what makes that deterministic. When something's wrong I read the condition-evaluation report, not the docs." `[BOOT 3.x DELTA]` file location differs from `spring.factories` (2.x).
- **Spine:** custom `orderflow-pricing-starter` auto-configuration, so you've written one, not just consumed one.

**43 · Configuration, profiles, `@ConfigurationProperties`, and property precedence** `CORE` → `43-configuration-profiles-properties.md`
- **Mastery:** you can recite the precedence order well enough to know why your env var beat your `application.yml`, and you bind typed config with validation rather than scattering `@Value`.
- **Mid → Senior:** "profiles switch config per environment" → "there's a defined ~15-level precedence chain with command line above env vars above profile-specific files. `@ConfigurationProperties` gives you a validated, relaxed-bound, typed object; `@Value` gives you a string and a runtime surprise."
- **Spine:** `orderflow` config for local/docker/load profiles — the load profile is what Topic 65 uses.

**44 · `@RestController`, request mapping, content negotiation** `CORE` → `44-rest-controllers.md`
- **Mastery:** you can describe the request path from `DispatcherServlet` through `HandlerMapping` → `HandlerAdapter` → argument resolvers → return-value handlers, and you know where your `@RequestBody` deserialization actually happens.
- **Mid → Senior:** "`@RestController` handles HTTP" → "it's `@Controller` + `@ResponseBody`; the real machinery is `DispatcherServlet`'s front-controller loop. Knowing where argument resolvers and message converters sit is what lets me add a custom one instead of parsing JSON by hand in a controller."
- **Spine:** `orderflow` REST API — products, inventory, orders, payments, wallet.

**45 · Bean Validation vs class-validator** `CORE` → `45-bean-validation.md`
- **Mastery:** you use validation groups for create-vs-update, write a cross-field custom constraint, and know why `@Valid` on a service method needs `@Validated` on the class.
- **Mid → Senior:** "add `@NotNull` to the DTO" → "controller-level `@Valid` is bean-graph validation; method validation on a service needs `@Validated`, which is proxy-based — so it inherits every constraint from Topic 40. Groups let one DTO serve create and update without two classes."
- **Spine:** request DTO validation across the whole `orderflow` API.

**46 · `@ControllerAdvice`, error handling, and RFC 9457 `ProblemDetail`** `CORE` → `46-error-handling-problem-detail.md`
- **Mastery:** you map the domain exception hierarchy from Topic 09 onto HTTP status codes in one place, and you never leak a stack trace or an entity into a response body.
- **Mid → Senior:** "`@ControllerAdvice` catches exceptions globally" → "same role as a Nest exception filter, but it's also where I enforce that the error contract is stable and machine-readable — `ProblemDetail` per RFC 9457 rather than an ad-hoc `{message}`. Error responses are an API surface with the same compatibility obligations as the success path."
- **Spine:** uniform `ProblemDetail` error contract for `orderflow`.

**47 · Spring Data JPA — repository generation, derived queries, `@Query`, projections** `CORE` → `47-spring-data-jpa.md`
- **Mastery:** you know a repository interface becomes a proxy backed by `SimpleJpaRepository`, when a derived query name becomes unreadable and should be `@Query`, and why you should return a projection rather than an entity from a read path.
- **Mid → Senior:** "Spring Data generates the query from the method name" → "it's a proxy over `SimpleJpaRepository`, parsed at startup — so a bad method name fails at context load, not at call time. Derived queries stop paying off around three predicates. And returning entities from read endpoints drags the persistence context and lazy proxies into your serialization layer; interface projections avoid that."
- **Spine:** `orderflow` repositories over Postgres, replacing all in-memory stubs.

**48 · Hibernate I — persistence context, entity states, dirty checking, flush modes** `DIFFERENTIATOR` → `48-hibernate-persistence-context.md`
- **Mechanical statement:** the persistence context is a first-level cache **plus a snapshot** of every loaded entity's field values. At flush, Hibernate diffs each managed entity against its snapshot and emits UPDATEs for what changed. You never call `save()` for an already-managed entity; the diff does it.
- **Mastery:** you can place any entity in transient/managed/detached/removed and say what the next flush will do, and you can explain why a setter inside a transaction writes to the database with no `save()` call anywhere.
- **Mid → Senior:** "the entity is saved when I call save()" → "for a managed entity `save()` is a no-op — dirty checking at flush does the write. That snapshot doubles the memory per loaded entity, which is why loading 100k entities to update one field is a design error. `FlushMode.AUTO` also flushes before queries that touch dirty tables, which is where surprise write ordering comes from."
- **Spine:** `Order`, `OrderLine`, `Product`, `Inventory`, `Wallet`, `Payment` entities.
- **Drill:** mutate a managed entity, never call `save()`, commit, observe the UPDATE in SQL logs. Then detach it first and observe nothing happens.

**49 · Hibernate II — associations, lazy vs eager, `LazyInitializationException`, DTO boundaries** `DIFFERENTIATOR` → `49-hibernate-associations-lazy.md`
- **Mechanical statement:** a lazy `@ManyToOne` is a runtime-generated subclass proxy holding only the identifier; touching any other field triggers a SELECT — and if the session is closed, an exception instead.
- **Mastery:** you default every association to `LAZY`, you never serialize an entity graph directly, and you can explain why `FetchType.EAGER` on a `@ManyToOne` is a latent N+1 rather than an optimisation.
- **Mid → Senior:** "set fetch to LAZY to avoid loading everything" → "`@ManyToOne` defaults to EAGER, which silently joins on every query including ones that don't need it. LAZY everywhere plus explicit fetch joins per use case is the only scheme that scales. `LazyInitializationException` isn't a Hibernate bug — it's the boundary telling me my transaction ended before my serialization began, and `open-session-in-view` hides that at the cost of holding a connection through view rendering."
- **Drill:** return an entity with a lazy collection from a controller, get the exception; 'fix' it with `open-session-in-view`, then show the connection held for the whole request in Topic 109's pool metrics.

**50 · Hibernate III — N+1: detection and the real fixes** `DIFFERENTIATOR` → `50-hibernate-n-plus-one.md`
- **Mastery:** you can detect N+1 by *counting queries in an assertion*, not by reading logs, and you know when to use a fetch join vs an entity graph vs `@BatchSize` vs a projection — including the fetch-join-plus-pagination trap.
- **Mid → Senior:** "N+1 means one query per row" → "I assert on query count in a test so it can't regress. The fix depends: fetch join for one collection, `@BatchSize`/`subselect` for several, entity graph when I want it declarative per use case, and a projection when I never needed the entities. A fetch join with `setMaxResults` makes Hibernate paginate in memory — it warns and you must not ignore it."
- **Spine:** `GET /orders` with lines, products and payment status — the endpoint Topic 65 hammers.
- **Drill:** turn on SQL logging and a Hibernate `Statistics` query counter, load 100 orders, count the queries, fix, re-count. The fix is only proven by the counter.

**51 · Hibernate IV — first-level, second-level and query cache; invalidation traps** `DIFFERENTIATOR` → `51-hibernate-caching.md`
- **Mastery:** you can say which cache holds what, why the second-level cache is per-entity-per-region and doesn't cache associations by default, and why any direct SQL write invalidates your assumptions.
- **Mid → Senior:** "we enabled the second-level cache" → "L1 is per-persistence-context and non-negotiable; L2 is cross-session and only safe if *every* writer goes through Hibernate. A batch job doing native SQL makes L2 serve stale rows silently. The query cache is worse — it caches identifiers and still hits L2 per row, and it invalidates on any write to the involved tables."
- **Drill:** enable L2 on `Product`, update a row with a native query, read through the cache, observe the stale price.

**52 · Hibernate V — optimistic `@Version`, pessimistic locking, lost updates** `DIFFERENTIATOR` → `52-hibernate-locking.md`
- **Mastery:** you can reproduce a lost update, then prevent it with `@Version`, and articulate when pessimistic locking is worth the throughput loss (short critical section, high contention, no retry budget).
- **Mid → Senior:** "add `@Version` for concurrency" → "optimistic assumes conflicts are rare and converts them into `OptimisticLockException` you must retry; pessimistic takes a row lock and serialises writers, trading throughput for no retries. For inventory decrement under a flash sale, a single atomic conditional UPDATE beats both."
- **Spine:** inventory decrement and wallet debit — the two contended writes in `orderflow`.
- **Drill:** two concurrent order placements against one unit of stock. Observe oversell. Fix three ways (`@Version` + retry, `PESSIMISTIC_WRITE`, atomic conditional UPDATE) and compare throughput under Topic 65's load generator.

**53 · Hibernate VI — write performance: batching and the IDENTITY trap** `DIFFERENTIATOR` → `53-hibernate-batching-writes.md`
- **Mastery:** you know `GenerationType.IDENTITY` disables JDBC batch inserts entirely (Hibernate needs the key immediately), and you can configure `batch_size` + `order_inserts` + `order_updates` and prove the batching with statement counts.
- **Mid → Senior:** "we set hibernate.jdbc.batch_size" → "which does nothing if the ID generator is IDENTITY, because Hibernate must round-trip per row to get the key. SEQUENCE with a pooled optimiser is what makes batching possible. And you must clear the persistence context periodically or the snapshot memory grows linearly."
- **Drill:** insert 50k order lines with IDENTITY, count statements; switch to SEQUENCE + pooled + batching, re-count and re-time.

**54 · `@Transactional` I — proxy semantics, propagation, rollback rules, read-only** `DIFFERENTIATOR` → `54-transactional-semantics.md`
- **Mechanical statement:** `@Transactional` is Topic 40's proxy plus a `PlatformTransactionManager`. The proxy begins a transaction, invokes your method, and commits or rolls back. It only ever sees calls that arrive *through* it.
- **Mastery:** you can predict the outcome of any nesting of `REQUIRED`/`REQUIRES_NEW`/`NESTED`/`SUPPORTS`, know rollback defaults to unchecked exceptions only, and know `readOnly=true` is a hint that also sets a Hibernate flush mode.
- **Mid → Senior:** "`@Transactional` makes it atomic" → "it makes it atomic *if the call comes through the proxy*. Default rollback is on `RuntimeException`/`Error` only — a checked exception commits, which is the single most common silent data bug in Spring codebases. `REQUIRES_NEW` takes a second connection from the pool while holding the first, which is a deadlock generator at pool size N."
- **Spine:** transactional order placement — reserve inventory, debit wallet, create payment, publish event.
- **Drill:** self-invocation trap (per §4). Then throw a *checked* exception from a `@Transactional` method and watch the write commit anyway.

**55 · `@Transactional` II — isolation and the transaction-to-connection-pool interaction** `DIFFERENTIATOR` → `55-isolation-and-connection-pool.md`
- **Mastery:** you can map each isolation level to the anomaly it prevents *and* the Postgres implementation reality (Postgres has no dirty read; REPEATABLE READ is snapshot isolation), and you know a transaction holds its connection for its entire lifetime — including any HTTP call you foolishly put inside it.
- **Mid → Senior:** "we use READ_COMMITTED" → "the anomaly table is the theory; what matters in production is that a transaction pins a pool connection for its whole duration. An external API call inside a transaction converts a 200 ms downstream blip into pool exhaustion and a full outage. Transaction boundaries should be the narrowest thing that must be atomic."
- **Drill:** put a 2-second HTTP call inside a `@Transactional` method, run Topic 65's load, watch HikariCP time out and every endpoint fail — including ones that touch no database.

**56 · Spring Security I — filter chain, authentication, `SecurityContext`** `CORE` → `56-spring-security-filter-chain.md`
- **Mastery:** you can name the order of the key filters, explain that authorization happens in a filter *before* your controller, and know `SecurityContextHolder` is `ThreadLocal`-backed — which is why it doesn't propagate to `@Async` or a new thread.
- **Mid → Senior:** "Spring Security handles auth" → "it's a servlet filter chain, so it runs before `DispatcherServlet` — quite unlike a Nest guard, which runs inside the framework's request pipeline. The `ThreadLocal` `SecurityContext` is the detail that bites: it doesn't cross thread boundaries without an explicit strategy, which matters for Topics 91 and 101."
- **Spine:** authenticated `orderflow` API. `[BOOT 3.x DELTA]` Security 6 vs 7 config idioms differ.

**57 · Spring Security II — authorization, JWT, OAuth2 resource server, CSRF, session fixation** `CORE` → `57-spring-security-authorization-jwt.md`
- **Mastery:** you can explain why a stateless JWT API doesn't need CSRF protection but a cookie-session one absolutely does, and why JWT revocation is a genuinely hard problem rather than an oversight.
- **Mid → Senior:** "we use JWT so it's stateless" → "stateless means you cannot revoke. The mitigations are short expiry plus refresh rotation, or a denylist — which reintroduces state. CSRF applies to ambient credentials (cookies), not to an `Authorization` header, so the standard advice to 'disable CSRF for APIs' is correct only for header-based auth."
- **Spine:** role-based authorization on `orderflow` — customer vs admin vs internal service.

---

### PHASE 6 — Testing `58–64`

**58 · JUnit 5 vs Jest — lifecycle, parameterized tests, extensions** `CORE` → `58-junit5.md`
- **Mastery:** you use `@ParameterizedTest` with a `@MethodSource` instead of copy-pasted cases, and know why JUnit creates a new test instance per method by default (and what `@TestInstance(PER_CLASS)` changes).
- **Mid → Senior:** "JUnit is Java's Jest" → "structurally similar, but there's no module mocking — you can't intercept an import, so testability is a *design* property enforced by injection. The extension model is also the integration point Spring, Testcontainers and Mockito all hook into."

**59 · Mockito — stubbing vs verification, and when mocking is the wrong tool** `CORE` → `59-mockito.md`
- **Mastery:** you can articulate when a mock encodes an implementation detail and makes refactoring impossible, and you prefer a real object or a fake for anything you own.
- **Mid → Senior:** "we mock the repository" → "stubbing controls inputs; verification asserts on interactions and couples the test to the implementation. Every `verify` is a design assertion. I mock what I don't own and can't run — and use Testcontainers for the database rather than mocking a repository whose real behaviour (flush timing, constraint violations) is exactly what I need to test."

**60 · The Spring test context — `@SpringBootTest` vs slices, and context caching** `CORE` → `60-spring-test-slices.md`
- **Mastery:** you can explain why a single `@MockBean` in one test class doubles your suite runtime (it forks the cached context), and you use `@WebMvcTest`/`@DataJpaTest` deliberately.
- **Mid → Senior:** "we use `@SpringBootTest` everywhere" → "the test context is cached by its full configuration key; any customisation — a `@MockBean`, a property override, an active profile — creates a new context and pays full startup. That's usually the entire explanation for a 12-minute suite."
- **Spine:** `orderflow` slice tests: web layer, persistence layer, security.

**61 · Testcontainers — real Postgres, Kafka, Redis in tests** `CORE` → `61-testcontainers.md`
- **Mastery:** you run one shared container per suite (singleton or reuse), not one per class, and you can explain why an H2-based test suite gives you false confidence about Postgres.
- **Mid → Senior:** "we test against H2, it's faster" → "H2 has different SQL semantics, different locking, different constraint behaviour and no `SKIP LOCKED`. Every bug it misses is one that only appears in production. Testcontainers with a singleton container and `@ServiceConnection` costs seconds, not minutes."
- **Spine:** all `orderflow` persistence tests against real Postgres; Kafka tests from Topic 113 onward.

**62 · Contract testing — Spring Cloud Contract / Pact** `CORE` → `62-contract-testing.md`
- **Mastery:** you can explain what a contract test proves that neither unit nor E2E tests do, and who owns the contract in a consumer-driven model.
- **Mid → Senior:** "we have integration tests" → "integration tests prove *your* service works against a stub you wrote — which is exactly the assumption that breaks. A contract test makes the stub and the provider's verification the same artefact, so a provider change fails the provider's build, not the consumer's production."

**63 · Mutation testing with PIT — the honest measure of a suite** `DIFFERENTIATOR` → `63-mutation-testing-pit.md`
- **Mechanical statement:** PIT modifies your bytecode (flip a conditional, remove a call, change a return) and re-runs the tests that cover the mutated line. A mutant that survives is a line your suite executes but does not verify.
- **Mastery:** you stop quoting line coverage and can point to specific surviving mutants in `orderflow`'s pricing and inventory logic.
- **Mid → Senior:** "we have 85% coverage" → "coverage measures execution, not assertion. Mutation score measures whether the suite would notice a defect. I run PIT on the modules where a silent bug costs money — pricing, inventory, wallet — and ignore it on glue code."
- **Drill:** run PIT on the order-total calculation. Find a surviving boundary mutant. Write the test that kills it.

**64 · Property-based testing (jqwik)** `DIFFERENTIATOR` → `64-property-based-testing.md`
- **Mechanical statement:** the framework generates inputs from a domain, and on failure *shrinks* toward the smallest input that still fails — which is what makes the counterexample readable.
- **Mastery:** you can state an invariant of `orderflow` as a property (wallet balance never negative; order total equals the sum of lines after any sequence of valid mutations) rather than as a list of examples.
- **Mid → Senior:** "we test edge cases" → "you test the edge cases you thought of. A property states the invariant and lets the generator hunt; shrinking then hands you a two-element counterexample instead of a 400-element one. It's the right tool where the input space is combinatorial — discount stacking, partial refunds, retry ordering."

---

### PHASE 7 — GATE `65`

**65 · GATE — `orderflow` running under load with recorded baselines** `CORE` → `65-gate-load-testing-baseline.md`
- **Mastery:** you have a reproducible load test, a realistic dataset, and numbers you trust — and you can explain why a single-number average latency is useless and what p99 actually measures.
- **Mid → Senior:** "we load tested it, it did 2000 rps" → "at what concurrency, what latency percentile, against what dataset size, with what cache state, and was the load generator itself saturated? Coordinated omission means most naive load tests under-report tail latency by an order of magnitude."
- **Deliverables (all mandatory before Phase 8):**
  1. `docker compose` stack: `orderflow` + Postgres + Redis (+ Kafka, stubbed until Topic 113).
  2. Seed dataset: ≥100k products, ≥1M orders, ≥5M order lines, realistic cardinality and skew (a few hot products).
  3. Load generator: **k6** (recommended — scriptable, low overhead) or Gatling. Open-model arrival rate, not a fixed VU loop, to avoid coordinated omission.
  4. Scenario mix: 70% catalogue read, 20% order read, 10% order placement.
  5. Recorded baseline: p50/p95/p99/p999 + throughput + error rate per endpoint, committed to `/docs/java/baselines/`.
  6. JVM flags recorded: heap size, collector, container limits.
- **Gate rule:** if you cannot re-run the baseline and get within ±10%, Phase 8 is not startable. Everything after this profiles *this*.

---

### PHASE 8 — JVM Internals `66–83`

Every topic here is `DIFFERENTIATOR` or `ELITE`, so: no ELI5 (R11 is replaced by a
mechanical statement), machine-level reality (R13), measurement discipline (R15),
and a failure drill (§4). All drills run against the Topic 65 baseline.

**66 · JVM architecture vs V8 — class loader, runtime data areas, execution engine** `DIFFERENTIATOR` → `66-jvm-architecture.md`
- **Mechanical:** a `.class` file is verified, linked and resolved into method-area metadata; frames live on a per-thread stack; objects live on a shared heap; the execution engine starts interpreting and promotes hot methods to compiled code.
- **Mastery:** you can say what is per-thread and what is shared, and why that split is the origin of every concurrency problem in Phase 9.
- **Mid → Senior:** "the JVM runs bytecode" → "V8 gives you one heap, one thread and an opaque GC. The JVM gives you a shared heap across preemptively-scheduled threads, a pluggable collector, a verifier, and a tiered compiler — which is why Java has a memory model at all and JavaScript essentially doesn't."

**67 · Class loading — delegation, laziness, initialization order** `DIFFERENTIATOR` → `67-class-loading.md`
- **Mechanical:** loading is parent-first delegation; a class is *initialized* (static blocks run) lazily on first active use, and initialization failure poisons the class for the JVM's lifetime.
- **Mastery:** you can distinguish `ClassNotFoundException` (reflective lookup failed) from `NoClassDefFoundError` (it was there at compile time, or its initializer already failed) and use that distinction to diagnose in one step.
- **Mid → Senior:** "the class wasn't on the classpath" → "`NoClassDefFoundError` after a successful earlier load usually means a static initializer threw the *first* time; the real exception is in the earlier log line and every subsequent access reports the useless error."
- **Drill:** throw from a static initializer, catch the first `ExceptionInInitializerError`, then touch the class again and get the misleading `NoClassDefFoundError`.

**68 · Memory areas — generations, eden/survivor, TLABs, promotion, metaspace** `DIFFERENTIATOR` → `68-heap-generations-tlab.md`
- **Mechanical:** allocation is a pointer bump in a thread-local allocation buffer inside eden; that's why Java allocation is ~10 instructions, not a free-list search. Objects surviving enough copies get promoted to old gen; class metadata lives in native metaspace, not the heap.
- **Mastery:** you can compute allocation rate from GC logs and predict young-collection frequency from eden size, and you know why "object allocation is expensive" is false and "object *retention* is expensive" is true.
- **Mid → Senior:** "objects go on the heap" → "they go in a TLAB in eden via pointer bump, which is nearly free. The cost is survival: anything that lives past a couple of young collections gets copied repeatedly and then promoted, and promotion rate — not allocation rate — is what drives old-gen pressure and full GCs."
- **Drill:** allocate short-lived vs long-lived objects at the same rate under identical flags, compare promotion in `-Xlog:gc*`.

**69 · Object layout — headers, alignment, compressed oops, measuring real size** `ELITE` → `69-object-layout-compressed-oops.md`
- **Mechanical:** an object is a mark word + class word (+ array length), then fields reordered by the JVM for alignment, padded to 8 bytes. Compressed oops store 32-bit shifted references below ~32 GB, which is why a 31 GB heap can hold more objects than a 33 GB one.
- **Mastery:** you can predict an object's size and verify it with JOL, and you know the 32 GB compressed-oops cliff as a *capacity planning* fact.
- **Mid → Senior:** "an Integer is 4 bytes" → "an `Integer` is ~16 bytes: 12-byte header plus a 4-byte int, 8-byte aligned. That's a 4× overhead versus `int`, which is the real cost of `List<Integer>` versus `int[]`. And crossing 32 GB of heap disables compressed oops, so you lose capacity by adding memory."
- **Measure:** JOL (`org.openjdk.jol`) `ClassLayout.parseInstance(x).toPrintable()`; `jcmd <pid> VM.flags | grep UseCompressedOops`.

**70 · GC fundamentals — reachability, roots, allocation math, throughput vs latency** `DIFFERENTIATOR` → `70-gc-fundamentals.md`
- **Mechanical:** collectors trace from GC roots (stacks, statics, JNI, thread-locals). Unreachable is not "unused" — a live root chain to a stale object is a leak.
- **Mastery:** you can express a workload as allocation rate (MB/s) and live-set size, and derive from those two numbers which collector and heap size the service needs.
- **Mid → Senior:** "GC cleans up unused objects" → "GC cost is a function of the *live set*, not the garbage. Doubling allocation of short-lived objects barely costs anything; doubling the live set costs on every cycle. That's why the tuning conversation starts with allocation rate and live-set size, not with flags."
- **Spine:** measure `orderflow`'s allocation rate and live set at baseline load.

**71 · G1 in depth — regions, humongous allocations, concurrent cycle, reading `-Xlog:gc*`** `ELITE` → `71-g1-in-depth.md`
- **Mechanical:** G1 divides the heap into equal regions with per-region remembered sets, collects a chosen *collection set* to meet a pause target, and treats any allocation ≥ half a region as **humongous** — allocated in contiguous old-gen regions, never in eden, and historically only reclaimed at a concurrent cycle.
- **Mastery:** you can read a GC log and name the pause cause, spot humongous allocation from `-Xlog:gc+heap`, and know that raising `MaxGCPauseMillis` shrinks the collection set rather than magically making pauses shorter.
- **Mid → Senior:** "we set MaxGCPauseMillis=200" → "that's a *goal*; G1 meets it by collecting fewer regions per pause, which can push you into evacuation failure and a full GC. I'd look at the log first: to-space exhaustion, humongous allocation, or a concurrent cycle losing the race are three different problems with three different fixes."
- **Drill (§4):** drive `orderflow` allocation rate up and allocate large `byte[]` payloads sized just over half the G1 region size. Capture `-Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=gc.log:time,uptime,level,tags`. Identify the multi-second pause's cause. Fix by region sizing or by not allocating the humongous object.
- **Measure:** `-Xlog:gc*`; `jcmd <pid> GC.heap_info`; `-XX:+PrintFlagsFinal -version | grep G1HeapRegionSize`.

**72 · Low-pause collectors — ZGC (generational), Shenandoah, and choosing one** `ELITE` → `72-zgc-shenandoah.md`
- **Mechanical:** ZGC uses coloured pointers and load barriers to relocate objects concurrently with the application; pauses become independent of heap size, paid for in throughput and barrier cost. Generational ZGC (default from JDK 23) adds a young generation back because most objects still die young.
- **Mastery:** you can state the actual trade (pause time vs throughput vs footprint) with numbers from *your* service, not from a vendor slide, and you know low pause does not mean low latency if the bottleneck is elsewhere.
- **Mid → Senior:** "ZGC has sub-millisecond pauses" → "true, and it costs throughput via load barriers and needs more headroom. If my p99 is dominated by a 300 ms downstream call, moving from G1's 40 ms pauses to ZGC's 0.5 ms buys nothing. Collector choice follows a latency budget, not a preference."
- **Drill:** run the Topic 65 baseline under G1, then `-XX:+UseZGC`. Compare p99, p999 and throughput. Explain any case where ZGC is *worse*.

**73 · Safepoints and time-to-safepoint** `ELITE` → `73-safepoints-ttsp.md`
- **Mechanical:** many JVM operations require all threads stopped at a safepoint. Compiled code polls at method returns and non-counted loop back-edges; a **counted `int` loop** may have its poll elided, so one thread can delay every other thread for an unbounded time. That delay is time-to-safepoint and it is invisible in GC pause time.
- **Mastery:** you know that a "GC pause" reported as 5 ms can hide a 900 ms TTSP, and you can enable the flags that separate the two.
- **Mid → Senior:** "our GC pauses are fine" → "GC pause and stop-the-world duration are different numbers. `-Xlog:safepoint` shows the time to *reach* the safepoint separately. Long TTSP usually means a counted loop over a large array, or a slow `System.arraycopy`, or swapped-out pages — and it also affects non-GC safepoint operations like thread dumps and biased-lock revocation."
- **Drill:** run a long counted `int` loop over a large array in one thread while another thread triggers frequent GCs. Capture `-Xlog:safepoint*` and identify TTSP dominating the pause. Change the loop counter to `long` and re-measure.

**74 · JIT I — interpreter, C1/C2 tiered compilation, profiling, deoptimization, warm-up** `ELITE` → `74-jit-tiered-compilation.md`
- **Mechanical:** methods start interpreted; invocation and back-edge counters promote them through C1 (fast, instrumented) to C2 (slow, aggressive). C2 speculates on the collected profile — monomorphic call sites, untaken branches, class hierarchy — and installs an *uncommon trap* that deoptimizes back to the interpreter if the speculation is violated.
- **Mastery:** you can explain why the first thousand requests after deploy are slow, why a benchmark that runs each case once is meaningless, and how a polluted profile makes previously-fast code permanently slower.
- **Mid → Senior:** "the JIT optimises hot code" → "it optimises based on the profile it observed. If a call site sees one implementation for a million calls and then a second one, C2 deoptimizes and recompiles bimorphic — so a rarely-used code path can permanently degrade a hot one. That's profile pollution, and it's why microbenchmarks that share a JVM across cases lie."
- **Drill:** make a call site monomorphic, warm it, then introduce a second implementation. Capture `-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization` and find the deopt. Measure the before/after with JMH forks.

**75 · JIT II — inlining, escape analysis, scalar replacement, lock elision** `ELITE` → `75-escape-analysis-inlining.md`
- **Mechanical:** escape analysis proves an object cannot be observed outside its allocating method; C2 then *scalar-replaces* it — the object is never allocated, its fields become registers. Lock elision removes monitors on non-escaping objects. All of this depends on inlining first, and inlining stops at bytecode-size and depth limits.
- **Mastery:** you know an allocation-heavy method can have zero allocations in compiled code, you know exactly what defeats it (the reference escaping to a field, a call the compiler couldn't inline, `Blackhole` in a benchmark), and you never claim it happened without proof.
- **Mid → Senior:** "the JIT removes unnecessary allocations" → "only for provably non-escaping objects, and only after inlining succeeds. A 400-byte method that doesn't get inlined kills escape analysis for its caller. I confirm with `-XX:+PrintInlining` and an allocation profile — not by asserting it."
- **Drill:** write a method allocating a `Point`-like value object; verify zero allocation with async-profiler's alloc mode; then store the reference in a static field and watch allocation reappear.
- **Measure:** `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining`; `-XX:-DoEscapeAnalysis` as the A/B control.

**76 · Reading bytecode with `javap -c`** `DIFFERENTIATOR` → `76-bytecode-javap.md`
- **Mechanical:** the JVM is a stack machine. Five invoke forms: `invokestatic`, `invokespecial`, `invokevirtual`, `invokeinterface`, `invokedynamic` — and lambdas and string concatenation both compile to the last one.
- **Mastery:** you can look at `javap -c` for a lambda and a string concat and explain what's actually there, and you use bytecode to settle arguments rather than to show off.
- **Mid → Senior:** "the compiler optimises that" → "javac does almost nothing; it emits nearly-literal bytecode and leaves optimisation to C2. So `javap -c` tells you what the *language* did — desugaring, autoboxing, bridge methods, indy — and tells you nothing about what runs. Two different questions, two different tools."
- **Note:** where I'm not certain of exact opcode output, I will say so and give you the `javap` command rather than guessing (§0.2).

**77 · JMH — why every naive benchmark is wrong** `ELITE` → `77-jmh.md`
- **Mechanical:** a `System.nanoTime()` loop measures dead-code elimination, constant folding, on-stack replacement and cold-JIT state — in unknown proportions. JMH forks a fresh JVM, warms up, uses `Blackhole` to consume results, and `@State` to defeat constant folding.
- **Mastery:** you can name the four ways a naive benchmark lies and you write a correct harness including forks, warmup iterations, and a baseline measurement.
- **Mid → Senior:** "I timed it in a loop and it took 3 ns" → "3 ns is roughly the cost of nothing, which is what a dead-code-eliminated loop measures. JMH exists because the JVM optimises away exactly the work you're trying to time. And a single fork hides profile pollution — I use at least 3."
- **Drill:** write the naive benchmark, get an absurd number, then port it to JMH and get the real one. Explain the gap.

**78 · Profiling — async-profiler, JFR, flame graphs, wall-clock vs CPU** `ELITE` → `78-profiling-flamegraphs.md`
- **Mechanical:** most Java profilers sample at safepoints, which biases toward safepoint-adjacent code. async-profiler uses `AsyncGetCallTrace`/perf events and is not safepoint-biased. CPU profiling shows where cycles go; wall-clock profiling shows where *time* goes — and for an I/O-bound service those are completely different answers.
- **Mastery:** you can look at a flame graph of `orderflow` under load and name the top three costs, and you know to use wall-clock mode when the service is blocked rather than busy.
- **Mid → Senior:** "the profiler says `HashMap.get` is hot" → "safepoint bias makes that a suspicious result. And for a service that spends 80% of its time waiting on Postgres, a CPU profile is answering the wrong question — I'd start with wall-clock, and use JFR in production because its overhead is low enough to leave on."
- **Spine:** profile `orderflow` at the Topic 65 baseline; produce a flame graph and a written top-3.

**79 · Memory leaks — finding them from a heap dump** `ELITE` → `79-memory-leaks-heap-dumps.md`
- **Mechanical:** a Java leak is unintended reachability. MAT's *dominator tree* answers "if this object were freed, how much would go with it" — which is the question you actually have, unlike a flat histogram.
- **Mastery:** you can go from `OutOfMemoryError` to a named retaining path in under fifteen minutes, and you know the four classic shapes: unbounded static collection, unremoved listener, `ThreadLocal` on a pooled thread, and non-static inner class capturing its enclosing instance.
- **Mid → Senior:** "we're leaking memory" → "Java doesn't leak, it retains. I take a dump on OOM, open the dominator tree, find the largest retained set, and walk the path-to-GC-root. The `ThreadLocal`-on-a-pooled-thread case is the nastiest: the thread outlives the request, so the entry is reachable through the thread until the pool recycles — which never happens."
- **Drill (§4):** add an unbounded `static Map<String, Order>` cache to `orderflow`, run Topic 65's load with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps -Xmx512m`. Open in Eclipse MAT, find the dominator, name the retaining path. **Second drill:** put a request context in a `ThreadLocal` and never remove it, on a fixed thread pool — show the retention grows to pool-size × context-size and stops, which is why it's often mistaken for "not a leak".

**80 · Off-heap memory — `DirectByteBuffer`, the FFM API, native memory tracking** `ELITE` → `80-off-heap-ffm.md`
- **Mechanical:** a `DirectByteBuffer`'s native memory is freed by a `Cleaner` that runs only when the *small* Java wrapper is collected — so native memory can grow while heap usage looks fine, and the process gets OOM-killed with a healthy heap.
- **Mastery:** you can explain why an RSS-vs-heap gap is a native-memory question, and you can use NMT to attribute it. `[JAVA 25]` FFM (`java.lang.foreign`) is final — `Arena` gives deterministic lifetimes that `DirectByteBuffer` never had.
- **Mid → Senior:** "heap is at 40%, why did the container get OOM-killed" → "RSS is heap + metaspace + code cache + thread stacks + direct buffers + GC structures + native libraries. I'd turn on `-XX:NativeMemoryTracking=summary` and diff `jcmd VM.native_memory`. Netty and any NIO-heavy path are the usual direct-buffer suspects, and `-XX:MaxDirectMemorySize` is the bound most people never set."
- **Drill:** allocate direct buffers in a loop without releasing references; watch RSS grow while heap stays flat; attribute it with NMT.

**81 · `java.lang.instrument` agents — how APM and Spring actually work** `ELITE` → `81-instrumentation-agents.md`
- **Mechanical:** a `-javaagent` gets a `ClassFileTransformer` invoked before each class is defined, so it can rewrite bytecode. That is how Datadog/New Relic/OpenTelemetry auto-instrumentation adds spans to code you never touched.
- **Mastery:** you can explain why an APM agent adds startup time and can perturb JIT inlining decisions, and why agent-instrumented code sometimes behaves differently under profiling.
- **Mid → Senior:** "we attach the APM agent" → "it rewrites bytecode at class-load, which means it changes method sizes and can push methods past the inlining threshold. That's a real observer effect — worth knowing before you conclude the code got slower after a release that only added the agent."

**82 · JVM tuning and container awareness** `ELITE` → `82-jvm-tuning-containers.md`
- **Mechanical:** modern JVMs read cgroup limits for `availableProcessors()` and default heap sizing, but the default `MaxRAMPercentage` (25%) is wrong for a container that exists to run one JVM, and CPU *quota* vs *shares* changes the core count the JVM sees — which sizes GC threads, the common ForkJoinPool, and the virtual-thread scheduler.
- **Mastery:** you set heap from measured live-set + headroom, not from a rule of thumb, and you know which flags matter (`MaxRAMPercentage`, `ActiveProcessorCount`, collector selection) and which are cargo cult.
- **Mid → Senior:** "we set `-Xmx` to the container limit" → "then the first metaspace or direct-buffer growth gets you OOM-killed, because heap isn't the whole footprint. And a CPU quota of 0.5 cores makes `availableProcessors()` return 1, which silently makes every parallel stream serial and every `ForkJoinPool.commonPool()` single-threaded."
- **Drill (§4):** run `orderflow` in a container with `--cpus=0.5 --memory=512m` and no JVM flags. Observe `Runtime.availableProcessors()`, the chosen collector, and the OOMKill. Then fix with explicit flags and re-run the baseline.
- **Measure:** `jcmd <pid> VM.flags -all`; `-XX:+PrintFlagsFinal -version`.

**83 · GraalVM native-image / AOT — what you gain, what you lose** `ELITE` → `83-graalvm-native-image.md`
- **Mechanical:** native-image does closed-world static analysis at build time and emits a native binary. Anything determined at runtime — reflection, dynamic proxies, resource loading, `Class.forName` — must be declared in configuration or it fails at runtime, not at build time.
- **Mastery:** you can state the honest trade (≈50 ms startup and low RSS vs. no JIT peak throughput, long builds, reflection config, weaker observability) and name the workloads where it wins (serverless, CLI, scale-to-zero) and where it loses (long-running high-throughput services). `[JAVA 25]` compare against project Leyden / AOT cache, which gets much of the startup win without the closed-world cost.
- **Mid → Senior:** "native image makes Java fast" → "it makes Java *start* fast and use less memory. It gives up C2's profile-guided peak throughput, so a long-running service is typically slower at steady state. For `orderflow` running 24/7 behind a load balancer, AppCDS or the AOT cache is the better trade."
- **Drill:** build `orderflow` as a native image. Find the first reflection failure at runtime. Fix with reachability metadata. Compare startup, RSS and steady-state throughput against the JVM baseline.

---

### PHASE 9 — Concurrency `84–102`

Every topic here carries R14 (a two-column thread-by-thread interleaving trace of
the bug **before** any correct code is shown) and, where applicable, R16 (a
`jcstress` test rather than a claim of thread-safety).

**84 · Threads and lifecycle vs the Node event loop** `DIFFERENTIATOR` → `84-threads-vs-event-loop.md`
- **Mechanical:** a Java thread is an OS thread scheduled preemptively. The scheduler can suspend it **between any two bytecodes**, including in the middle of a `long` write on a 32-bit VM or between reading and writing a counter.
- **NO TYPESCRIPT ANALOGUE** for the preemption. Your event loop guarantees a callback runs to completion; that guarantee is the reason JS has almost no data races and Java has an entire memory model.
- **Mastery:** you stop reasoning about "when does my code get interrupted" and start reasoning about "what state is shared and who may observe it mid-update".
- **Mid → Senior:** "Java threads are like worker_threads" → "workers have isolated heaps and communicate by copying — that's message passing. Java threads share one heap, so any object reachable from two threads is a correctness question. That single difference generates everything in this phase."

**85 · `synchronized`, intrinsic monitors, and lock inflation** `DIFFERENTIATOR` → `85-synchronized-monitors.md`
- **Mechanical:** every object has a mark word that can encode lock state. Uncontended locking historically used a thin/stack lock via CAS on the mark word; contention *inflates* to a fat monitor with an OS-level wait queue. Biased locking was removed in JDK 15+ — do not repeat 2015 advice about it.
- **Mastery:** you know `synchronized` provides mutual exclusion **and** a happens-before edge, that uncontended locks are cheap and contended ones are expensive for a specific reason (park/unpark syscalls), and you can measure contention rather than guess.
- **Mid → Senior:** "synchronized is slow" → "uncontended it's a CAS on the mark word — nanoseconds. Contended it inflates to a monitor with real parking, and that's what costs. So the question is never 'is synchronized slow', it's 'what is my contention rate', which JFR's monitor-blocked events answer directly."
- **Drill:** put a `synchronized` block on the `orderflow` inventory decrement, run the Topic 65 load, capture JFR monitor-blocked events, and quantify the contention.

**86 · The Java Memory Model I — happens-before, visibility, reordering** `ELITE` → `86-jmm-happens-before.md`
- **Mechanical:** the JMM is a contract about which writes a read is permitted to see. Absent a happens-before edge, the compiler, the JIT and the CPU may all reorder, and a thread may read an arbitrarily stale value **forever** — not "eventually".
- **NO TYPESCRIPT ANALOGUE.** A single-threaded runtime with run-to-completion semantics has nothing to say about visibility. Comparing this to the event loop would install a wrong model that costs more to remove than the analogy is worth.
- **Mastery:** you can list the happens-before edges (program order, monitor unlock→lock, volatile write→read, `Thread.start`, `Thread.join`, final-field freeze) and use them to *prove* a program correct rather than test it and hope.
- **Mid → Senior:** "another thread might not see the update immediately" → "it might not see it *ever*. 'Eventually' isn't in the spec — without a happens-before edge, hoisting the read out of a loop is a legal compilation. The fix isn't a sleep or a retry, it's establishing the edge."
- **Drill (§4):** the classic non-`volatile` stop flag. Write it, run with `-Xint` (terminates) and then normally with a warmed loop (hangs). That gap **is** the memory model.

**87 · The Java Memory Model II — `volatile`, and how it maps to CPU barriers** `ELITE` → `87-volatile-and-memory-barriers.md`
- **Mechanical:** a volatile write emits a `StoreStore` barrier before and a `StoreLoad` after (on x86, effectively a locked instruction / `mfence`); a volatile read emits `LoadLoad`/`LoadStore` after. This gives visibility and ordering. It gives **no atomicity** for compound operations.
- **Mastery:** you can say precisely why `volatile int i; i++` is still a race, and why `volatile` is the right tool for a flag or a safely-published reference and the wrong tool for a counter.
- **Mid → Senior:** "volatile makes it thread-safe" → "volatile guarantees visibility and ordering, not atomicity. `i++` is read-modify-write — three operations, interleavable at each boundary. On x86 the write is roughly a locked store, which is why volatile writes are meaningfully more expensive than plain ones and volatile reads are nearly free."
- **Drill + jcstress (R16):** write the interleaving trace for `volatile` `i++` losing an increment, then encode it as a jcstress test and get the actual observed-outcomes table.

**88 · `final` field semantics and safe publication** `ELITE` → `88-final-fields-safe-publication.md`
- **Mechanical:** the end of a constructor with `final` fields carries a freeze action: any thread that sees the reference is guaranteed to see the correctly-initialised final fields, with no synchronisation. Non-final fields have no such guarantee — another thread can observe a partially-constructed object.
- **Mastery:** you can write the unsafe-publication interleaving from memory and name the five safe publication idioms (static initializer, volatile/AtomicReference, final field, guarded by a lock, safely-published-into a concurrent collection).
- **Mid → Senior:** "the object is created before it's used" → "not from another thread's perspective. Without a final field or a happens-before edge, a reader can see a non-null reference to an object whose fields are still default values. That's why the double-checked locking idiom needs `volatile`, and why making the fields `final` fixes a whole class of publication bugs for free."
- **Drill:** unsafe publication of a partially-constructed `Order` via a plain field, verified with jcstress.

**89 · `wait`/`notify`, guarded blocks, spurious wakeups** `DIFFERENTIATOR` → `89-wait-notify.md`
- **Mechanical:** `wait()` atomically releases the monitor and parks; on wake it must re-acquire. The JLS explicitly permits **spurious wakeups**, so the condition must be re-checked in a `while` loop — an `if` is a bug.
- **Mastery:** you can write a correct bounded buffer with `wait`/`notify` and then explain why you would never ship it when `BlockingQueue` exists.
- **Mid → Senior:** "we use wait/notify for coordination" → "with `if` instead of `while` it's broken by spec, and `notify` rather than `notifyAll` can wake the wrong waiter and deadlock a mixed producer/consumer set. It's worth understanding because `BlockingQueue` and `Condition` are built on it — not because you should write it."

**90 · `ExecutorService` — pool sizing, queue choice, rejection, shutdown** `ELITE` → `90-executors-pool-sizing.md`
- **Mechanical:** `ThreadPoolExecutor` only grows past `corePoolSize` when the **queue is full**. With an unbounded `LinkedBlockingQueue` (what `newFixedThreadPool` uses), `maximumPoolSize` is dead configuration and the queue absorbs unbounded work — converting a throughput problem into a latency-and-memory problem.
- **Mastery:** you size a pool from Little's Law (`threads ≈ throughput × latency`, adjusted for CPU vs I/O bound), you always bound the queue, you choose the rejection policy deliberately, and you shut down with `shutdown` → `awaitTermination` → `shutdownNow`.
- **Mid → Senior:** "we use a fixed thread pool of 200" → "where did 200 come from? For an I/O-bound task at 50 ms latency and a 1000 rps target, Little's Law says ~50 in flight. And `newFixedThreadPool` has an unbounded queue, so under overload latency climbs without bound and nothing sheds — `CallerRunsPolicy` on a bounded queue at least applies backpressure to the caller."
- **Spine:** the `orderflow` async payment-notification pool.
- **Drill (§4):** unbounded queue under Topic 65 overload → watch queue depth and p99 climb together while throughput stays flat, then OOM. Fix with a bounded queue + rejection policy and show latency stabilise and errors appear (which is the correct behaviour).

**91 · `CompletableFuture` vs JS Promises — and where the analogy breaks** `DIFFERENTIATOR` → `91-completablefuture.md`
- **PARTIAL ANALOGUE.** `thenApply`/`thenCompose` ≈ `then`; `exceptionally` ≈ `catch`. Where it breaks: (1) *which thread* runs the continuation depends on `Async` variants and the supplied executor — with no executor, a completed future runs the callback on the *calling* thread; (2) there is no `await`, so `join()` blocks a real thread; (3) exceptions wrap in `CompletionException`, so `instanceof` checks on the cause, not the exception.
- **Mastery:** you always pass an explicit executor, you never `join()` inside a continuation, and you can explain why an un-handled exceptional completion is silently swallowed.
- **Mid → Senior:** "CompletableFuture is Java's Promise" → "the shape matches, the threading doesn't. Default `*Async` variants use the common ForkJoinPool, which is Topic 25's shared, cores−1, non-blocking-only pool. Doing JDBC in it starves everything else in the JVM. And a future nobody calls `join`/`get` on discards its exception entirely."
- **Spine:** parallel fan-out for the order-detail endpoint (product + inventory + payment status).
- **Drill:** run blocking JDBC on the common pool under load, observe unrelated parallel work stall; fix with a dedicated bounded executor.

**92 · `ConcurrentHashMap` internals and `CopyOnWriteArrayList`** `ELITE` → `92-concurrent-collections.md`
- **Mechanical:** Java 8+ CHM has no segments. Reads are lock-free (volatile reads of `Node.val`/`next`); writes CAS an empty bin or `synchronized` on the first node of a non-empty bin. `size()` is an approximation via a striped counter. `COWAL` copies the entire backing array on every write — O(n) per write, zero-cost lock-free reads.
- **Mastery:** you know that thread-safe methods do **not** compose into thread-safe sequences, and you reach for `compute`/`merge`/`putIfAbsent` instead of check-then-act.
- **Mid → Senior:** "ConcurrentHashMap is thread-safe so we're fine" → "each *operation* is atomic; `if (!map.containsKey(k)) map.put(k, v)` is two operations and races. `computeIfAbsent` is the atomic form — though it must not re-enter the same map, or you get a bin deadlock. And `COWAL` is only correct where reads massively dominate writes; a write-heavy listener list is quadratic."
- **Drill:** check-then-act race on a CHM-backed idempotency cache: two concurrent requests both pass the check and both charge the wallet. Trace it, then fix with `putIfAbsent`.

**93 · `BlockingQueue` and producer-consumer** `DIFFERENTIATOR` → `93-blocking-queues.md`
- **Mechanical:** `ArrayBlockingQueue` is bounded with a single lock; `LinkedBlockingQueue` uses separate put/take locks (higher throughput, unbounded by default); `SynchronousQueue` has no capacity and hands off directly.
- **Mastery:** you choose bounded by default and can explain what the bound *is* — it's your backpressure policy, expressed as a number.
- **Mid → Senior:** "we use a LinkedBlockingQueue" → "unbounded by default, which means the queue is your OOM. The capacity is a design decision: it's the amount of work you're willing to buffer before you push back on the producer, and it should be derived from a latency budget."

**94 · Explicit locks — `ReentrantLock`, `ReadWriteLock`, `StampedLock`** `ELITE` → `94-explicit-locks.md`
- **Mechanical:** all three sit on `AbstractQueuedSynchronizer` — a volatile `state` int plus a CLH wait queue. `StampedLock` adds an optimistic read that returns a stamp you *validate* after reading; it is not reentrant and does not support conditions.
- **Mastery:** you can name what explicit locks buy over `synchronized` (tryLock with timeout, interruptibility, fairness, multiple conditions, non-block-structured locking) and you know that `synchronized` is the right default until one of those is needed — with the Loom exception from Topic 101.
- **Mid → Senior:** "ReentrantLock is faster than synchronized" → "not since lock inflation improved; that's dated advice. It buys `tryLock(timeout)` — which is how you avoid deadlock rather than diagnose it — plus interruptibility and multiple `Condition`s. Since JDK 21, it also matters because a virtual thread blocking on `ReentrantLock` unmounts, whereas one blocking inside `synchronized` used to pin its carrier."
- **Drill (§4):** two-lock ordering deadlock between the wallet and inventory services. Capture `jcmd <pid> Thread.print`, read the "Found one Java-level deadlock" section, name the two threads and two monitors, fix by global lock ordering. Then fix a second time with `tryLock` + timeout and compare the failure mode.

**95 · Atomics, CAS at the instruction level, ABA, `LongAdder`** `ELITE` → `95-cas-atomics-longadder.md`
- **Mechanical:** `AtomicInteger.incrementAndGet` compiles to a `lock cmpxchg` retry loop (or `lock xadd` for plain add) on x86. Under high contention the retry loop burns CPU and cache-line ownership ping-pongs between cores. `LongAdder` gives each contending thread its own cell, summing only on read.
- **Mastery:** you can explain why a CAS counter's throughput *decreases* as you add cores under contention, and you know `LongAdder` trades read cost and memory for write scalability.
- **Mid → Senior:** "atomics are lock-free so they scale" → "lock-free means no thread is blocked, not that it scales. A contended CAS is a cache-line ping-pong: each core must own the line exclusively, so throughput collapses with core count. `LongAdder` is striped and scales; ABA matters when you're CAS-ing a pointer that can be recycled, and `AtomicStampedReference` is the answer there."
- **Drill:** `AtomicLong` vs `LongAdder` as `orderflow`'s request counter under 64 threads, measured with JMH `@Threads`. Explain the crossover point.

**96 · False sharing, cache lines, `@Contended`** `ELITE` → `96-false-sharing.md`
- **Mechanical:** cache coherence operates on 64-byte lines. Two independent variables in the same line make every write by one core invalidate the other core's copy — the variables are logically independent and physically contended.
- **Mastery:** you can construct the effect, measure it, fix it with padding or `@Contended` (which needs `-XX:-RestrictContended`), and recognise it as a *possible* explanation for a scalability wall rather than the default one.
- **Mid → Senior:** "the counters are independent so there's no contention" → "logically, yes; physically, if they're adjacent fields they share a cache line and every write invalidates the other core's copy. This is exactly why `LongAdder`'s `Cell` is `@Contended`. It's measurable with JMH and with perf's cache-miss counters — and it's over-diagnosed, so I'd prove it before padding anything."

**97 · Coordination primitives — `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`** `DIFFERENTIATOR` → `97-coordination-primitives.md`
- **Mechanical:** all AQS-based. A latch is one-shot and counts down; a barrier is reusable and counts up to a party count; a semaphore is a permit counter; a phaser is a dynamic-party, multi-phase barrier.
- **Mastery:** you pick by lifecycle (one-shot vs reusable) and you use a `Semaphore` as a bulkhead — a concurrency limiter — rather than reaching for a thread pool for that job.
- **Mid → Senior:** "we use a CountDownLatch to wait for startup" → "fine, it's one-shot. If you need to re-synchronise each round, a latch can't and a `CyclicBarrier` can. And a `Semaphore` is the primitive behind bulkheading — limiting in-flight calls to a fragile downstream is a permit count, not a thread count, which matters a lot once virtual threads are involved."

**98 · Bug taxonomy — races, deadlock, livelock, starvation, thread leaks** `ELITE` → `98-concurrency-bug-taxonomy.md`
- **Mechanical:** deadlock requires all four Coffman conditions simultaneously — mutual exclusion, hold-and-wait, no preemption, circular wait. You break deadlock by removing exactly one, and lock ordering removes circular wait.
- **Mastery:** given a symptom (hung threads / high CPU with no progress / one starved consumer / growing thread count), you can name the class of bug and the diagnostic that confirms it, in one step.
- **Mid → Senior:** "it deadlocked" → "let's separate it: a thread dump showing BLOCKED threads in a cycle is deadlock; RUNNABLE threads at 100% CPU with no progress is livelock; one thread never scheduled is starvation; a thread count that only grows is a leak from an executor nobody shuts down. Four symptoms, four fixes, and the thread dump distinguishes them."
- **Drill:** create a thread leak by constructing an `ExecutorService` per request. Watch thread count grow in JFR until `OutOfMemoryError: unable to create native thread`.

**99 · `jcstress` — actually testing concurrency correctness** `ELITE` → `99-jcstress.md`
- **Mechanical:** jcstress runs a tiny pair of actors millions of times across JIT modes and interleavings, and reports the *frequency of each observed outcome* against outcomes you declared ACCEPTABLE / ACCEPTABLE_INTERESTING / FORBIDDEN. It finds outcomes a unit test never will.
- **Mastery:** you write a jcstress test for a real `orderflow` invariant and read the outcome table, including "0 occurrences doesn't mean impossible".
- **Mid → Senior:** "we tested it with 100 threads in a loop and it passed" → "that tests your machine, your JIT state and your luck. jcstress deliberately hunts interleavings and reports outcome frequencies — and even then, a forbidden outcome with zero observations means 'not observed here', not 'proven impossible'. The proof is the happens-before argument; jcstress is how you catch the argument being wrong."

**100 · `ForkJoinPool` and work stealing** `DIFFERENTIATOR` → `100-forkjoinpool.md`
- **Mechanical:** each worker owns a deque, pushes and pops its own tasks LIFO (cache-friendly), and steals from the *tail* of another worker's deque when idle. It is designed for CPU-bound, recursively-splitting, non-blocking tasks — and `ManagedBlocker` is the escape hatch for the cases that must block.
- **Mastery:** you know the common pool is shared JVM-wide, is sized `cores − 1`, and that blocking in it is the root cause behind Topics 25 and 91's drills.
- **Mid → Senior:** "ForkJoinPool is just a thread pool" → "it's a work-stealing pool built for divide-and-conquer: LIFO locally for cache locality, FIFO stealing to grab the biggest remaining chunk. Blocking tasks break the model because a parked worker can't steal — that's what `ManagedBlocker` compensates for, and it's why the virtual-thread scheduler is a separate FJP."

**101 · Virtual threads (Loom) — carriers, scheduling, pinning** `ELITE` → `101-virtual-threads.md`
- **Mechanical:** a virtual thread's stack lives on the heap as a continuation. On a blocking call it *unmounts* — the continuation is copied off the carrier and the carrier picks up other work. It cannot unmount while inside a `synchronized` block or a native frame; that is **pinning**, and with a default scheduler of `cores` carriers, a handful of pinned threads starves the entire JVM. (JDK 24+ removes most `synchronized` pinning — verify on your JDK before assuming either way.)
- **Mastery:** you can state exactly what virtual threads make faster (concurrency at high blocking-I/O counts) and what they do **not** (CPU-bound work, downstream capacity, per-request latency), and you know pooling them is an anti-pattern.
- **Mid → Senior:** "virtual threads make everything faster" → "they raise the ceiling on *concurrent blocked operations* from thousands to millions. They don't add CPU and they don't make the database faster — if your bottleneck is a 20-connection Hikari pool, you've just moved the queue. And they must not be pooled: they're cheap to create, and pooling reintroduces the limit you removed. Pinning is the real trap, and `jdk.tracePinnedThreads` / the JFR pinned-event is how you find it."
- **Spine:** switch `orderflow` to `spring.threads.virtual.enabled=true` and re-run the Topic 65 baseline.
- **Drill (§4):** hold a `synchronized` block across the payment-gateway call under load. Observe carrier starvation and throughput collapse with idle CPU. Confirm with `-Djdk.tracePinnedThreads=full` (or the JFR `VirtualThreadPinned` event). Fix with `ReentrantLock`, re-measure.

**102 · Structured concurrency and scoped values** `ELITE` → `102-structured-concurrency-scoped-values.md`
- **Mechanical:** a `StructuredTaskScope` binds subtask lifetimes to a lexical block: the scope cannot exit until every fork completes or is cancelled, so there are no orphan tasks and errors propagate to the parent. `ScopedValue` is an immutable, inheritance-aware replacement for `ThreadLocal` that doesn't leak on pooled or virtual threads. `[JAVA 25]` both are finalised/preview-advanced — I'll state the exact status for your JDK and give the API-shape caveat rather than guessing.
- **Mastery:** you can express the order-detail fan-out as a scope with a shutdown-on-failure policy and explain what it fixes over `CompletableFuture.allOf` (cancellation, error propagation, no leaked tasks).
- **Mid → Senior:** "we use `allOf` to wait for both calls" → "if one fails, the other keeps running and you've leaked work — `allOf` has no cancellation semantics. A structured scope makes the subtask lifetime a lexical property, so failure cancels siblings and the stack trace actually shows the parent. `ScopedValue` then carries request context across those forks without the `ThreadLocal`-on-pooled-thread leak from Topic 79."
- **Spine:** rewrite Topic 91's fan-out with structured concurrency; propagate the tenant context via `ScopedValue`.

---

### PHASE 10 — Reactive & Async at Scale `103–108`

**103 · NIO, epoll, and Netty's event loop** `DIFFERENTIATOR` → `103-nio-netty-event-loop.md`
- **HONEST ANALOGUE — the closest one in the whole curriculum.** Netty's `EventLoopGroup` is structurally libuv: a small set of threads each running an `epoll_wait` loop over registered channels, dispatching readiness events to handlers. If you understand Node's event loop you already understand this.
- **Where it differs:** Netty runs *N* event loops (one per core) rather than one, so handler code must be thread-safe with respect to other loops, and blocking inside a handler blocks every connection assigned to that loop — the same rule as Node, with N times the blast radius shape.
- **Mastery:** you can explain readiness-based (epoll) vs completion-based (io_uring / IOCP) I/O and why a blocking call in a channel handler is catastrophic.
- **Mid → Senior:** "Netty is async so it's fast" → "it's fast because a few threads multiplex many sockets via epoll, avoiding a thread-per-connection stack cost. The discipline is absolute: any blocking work in a handler must go to a separate `EventExecutorGroup`, or you stall every connection on that loop."

**104 · Reactor I — `Mono`/`Flux`, operators, assembly vs subscription time** `DIFFERENTIATOR` → `104-reactor-mono-flux.md`
- **HONEST ANALOGUE:** `Flux` ≈ RxJS `Observable`, `Mono` ≈ `Observable` of 0–1. Operators, `map`/`flatMap`/`switchMap` all carry over.
- **Where it differs from Promises specifically:** a `Mono` is **cold and lazy** — nothing happens until `subscribe()`. A Promise is eager and starts on creation. Building a `Mono` and not subscribing is a silent no-op, and that is the single most common bug for people arriving from Promises.
- **Mastery:** you can distinguish assembly time from subscription time and explain why a `Mono` returned from a controller is subscribed by the framework, not by you.
- **Mid → Senior:** "Mono is like a Promise" → "cold vs hot is the difference that matters. A Promise runs whether or not you await it; a `Mono` does nothing until subscribed, so a fire-and-forget `.map(...)` without `subscribe()` silently does nothing. And `flatMap` is concurrent-with-interleaving while `concatMap` preserves order — a distinction Promises don't force you to make."

**105 · Reactor II — backpressure, `request(n)`, overflow strategies** `DIFFERENTIATOR` → `105-backpressure.md`
- **NO TYPESCRIPT ANALOGUE.** Promises have no concept of a consumer signalling how much it can accept; RxJS largely doesn't either. Reactive Streams' `request(n)` is a demand signal flowing *upstream*, which is the entire point of the specification.
- **Mastery:** you can identify where backpressure is lost (any `onBackpressureBuffer` with no bound, any bridge from a push source) and choose an overflow strategy as a product decision (buffer / drop / latest / error).
- **Mid → Senior:** "we use reactive so it handles load" → "only where demand actually propagates. The moment you bridge a push source — a Kafka listener, an SSE feed — without translating it to demand, you have an unbounded buffer and an OOM. The overflow strategy is a business decision: dropping price ticks is fine, dropping payment callbacks is not."
- **Spine:** payment-callback ingestion for `orderflow` as a `Flux` with an explicit, bounded overflow strategy.
- **Drill (§4):** bridge an unbounded push source into a `Flux` with a slow consumer. Watch heap grow until OOM. Apply `onBackpressureBuffer(n, DROP_OLDEST)` and show the tradeoff explicitly.

**106 · WebFlux vs MVC — threading models and when each wins** `DIFFERENTIATOR` → `106-webflux-vs-mvc.md`
- **Mastery:** you know WebFlux's benefit is *thread economy under high concurrency with blocking-free I/O*, that one blocking JDBC call in a WebFlux handler destroys it, and that R2DBC is a real and costly commitment.
- **Mid → Senior:** "WebFlux is faster than MVC" → "at the same throughput it isn't; it uses fewer threads. That only converts to a win when thread count is the constraint — tens of thousands of concurrent slow connections. And it's all-or-nothing: one blocking call on an event loop and you're worse than MVC, with harder debugging."
- **Spine:** a WebFlux module for `orderflow`'s callback ingestion, kept separate from the MVC core, so you can benchmark both.

**107 · The Loom-vs-reactive decision** `ELITE` → `107-loom-vs-reactive.md`
- **Mechanical:** both solve thread-per-request thread cost. Loom keeps the imperative programming model and moves the cost to heap-allocated continuations. Reactive changes the programming model and gets composable backpressure as a genuine extra.
- **Mastery:** you can defend either answer against pushback with the actual decision criteria, and you know the honest residual: **Loom does not give you backpressure**, and that is the one thing reactive still uniquely provides.
- **Mid → Senior:** "virtual threads make reactive obsolete" → "for the thread-economy motivation, largely yes — and you get readable stack traces and working debuggers back. What Loom does not give you is backpressure or stream composition. So: high-concurrency request/response → virtual threads; streaming with demand propagation and a slow consumer → reactive. And the migration cost of an existing WebFlux codebase is usually the deciding factor, not the technical merits."
- **Deliverable:** a written recommendation for `orderflow` with the benchmark data from Topics 101, 105 and 106 behind it. This is a Phase 12 rehearsal.

**108 · Debugging reactive — stack traces, checkpoints, context propagation** `DIFFERENTIATOR` → `108-reactive-debugging.md`
- **Mechanical:** a reactive stack trace shows the *subscription* stack, not the assembly stack, so it tells you where it ran and not where you wrote it. `Hooks.onOperatorDebug()` reconstructs assembly information at heavy cost; `checkpoint()` does it cheaply at chosen points.
- **Mastery:** you can propagate MDC/trace context through a reactive chain via `Context`/`ContextView` and know `ThreadLocal` cannot work here — the chain hops threads between operators.
- **Mid → Senior:** "the stack trace is useless" → "it's the subscription stack; assembly info is gone unless you add `checkpoint()`. Same root cause as losing MDC: `ThreadLocal` is meaningless when the chain moves between event-loop threads, which is why Reactor has its own `Context` and why Micrometer's context-propagation library exists."
- **Drill:** lose the correlation ID across a `flatMap` boundary, prove it in logs, restore it with `ContextView` + `ContextSnapshot`.

---

### PHASE 11 — Distributed Systems & Production `109–124`

**109 · HikariCP sizing and the pool-vs-thread-pool deadlock** `ELITE` → `109-hikaricp-pool-deadlock.md`
- **Mechanical:** a request thread holding one connection and requesting a second (`REQUIRES_NEW`, a nested `@Transactional`, a parallel sub-query) can deadlock the pool: with N connections and N threads each holding one and waiting for another, nobody can proceed. The classic safe bound is `pool ≥ threads × (simultaneous_connections_per_thread − 1) + 1`.
- **Mastery:** you size the pool from database capacity and Little's Law rather than "more is better", and you know a bigger pool usually makes latency *worse* because the contention just moves into Postgres.
- **Mid → Senior:** "we raised the pool to 100 to fix timeouts" → "Postgres does a process per connection; 100 connections against 8 cores means context-switch thrashing and worse latency for everyone. The right pool size is usually close to `cores × 2`. Timeouts under load are almost always a transaction holding a connection too long — Topic 55's HTTP-call-inside-a-transaction — not a pool that's too small."
- **Spine:** tune `orderflow`'s pool against the Topic 65 baseline; record the latency curve vs pool size.
- **Drill (§4):** set pool size 10, run 10 concurrent requests that each open a nested `REQUIRES_NEW` transaction. Capture the thread dump, show every thread parked in `HikariPool.getConnection`, explain the cycle.

**110 · Spring Cache and Redis — key design, stampede, invalidation** `CORE` → `110-spring-cache-redis.md`
- **Mastery:** you can name the three cache failure modes (stampede/dogpile, stale-after-write, unbounded key growth) and the mitigations, and you know `@Cacheable` is proxy-based and therefore carries every caveat of Topic 40.
- **Mid → Senior:** "we added `@Cacheable`" → "which caches on the proxy, so self-invocation skips it, and it caches exceptions never and nulls only if you say so. The bigger question is the miss storm: when a hot product's entry expires, N concurrent requests all hit Postgres. That's a lock-on-miss or probabilistic-early-expiry problem, and the TTL alone doesn't solve it."
- **Spine:** cache the `orderflow` product catalogue; measure hit rate and the p99 change at the Topic 65 baseline.

**111 · Resilience4j — circuit breakers, bulkheads, retries with jitter, rate limiting** `DIFFERENTIATOR` → `111-resilience4j.md`
- **Mastery:** you know retries *amplify* an overload without jitter and a concurrency budget, you order the decorators correctly (retry outside circuit breaker, bulkhead innermost), and you only retry idempotent operations.
- **Mid → Senior:** "we added retries for reliability" → "retries convert a partial outage into a full one — three retries is a 4× load multiplier on an already-failing dependency. Exponential backoff **with jitter** plus a circuit breaker plus a bulkhead is the minimum set, and the decorator order matters: retrying *inside* the breaker means the breaker never sees the failures."
- **Spine:** wrap the `orderflow` payment gateway; prove the breaker opens under an injected outage.
- **Drill (§4):** stub the payment gateway to fail, retry without jitter from 200 concurrent requests, and observe the synchronised retry storm. Add jitter + breaker + bulkhead and re-measure.

**112 · Spring Cloud — discovery, config, gateway** `CORE` → `112-spring-cloud.md`
- **Mastery:** you can say what each component buys you *in a Kubernetes environment where the platform already does service discovery*, and decline the ones that duplicate it.
- **Mid → Senior:** "we use Eureka for service discovery" → "on Kubernetes, DNS and Services already do that; adding Eureka is a second source of truth to keep consistent. Spring Cloud Config is worth more, and a gateway is worth it when you need request-level logic the ingress can't express. Adopt per-component, not as a suite."

**113 · Kafka I — consumer groups, partitions, rebalancing, offsets** `DIFFERENTIATOR` → `113-kafka-consumer-groups.md`
- **Mechanical:** a partition is the unit of parallelism *and* of ordering; one partition maps to at most one consumer in a group. A consumer that exceeds `max.poll.interval.ms` is presumed dead, triggering a rebalance — during which the whole group stops consuming.
- **Mastery:** you can explain a rebalance storm from first principles and diagnose it from consumer lag plus rebalance-rate metrics.
- **Mid → Senior:** "we added consumers to increase throughput" → "beyond the partition count that does nothing — extra consumers sit idle. And if per-message processing is slow, exceeding `max.poll.interval.ms` triggers a rebalance, which pauses everyone and often causes the next timeout: a rebalance storm. The fixes are smaller `max.poll.records`, cooperative-sticky assignment, or moving work off the poll thread."
- **Spine:** `orderflow` order-placed events; inventory and notification consumers.
- **Drill:** make the consumer slower than `max.poll.interval.ms`, observe the rebalance loop and lag growth in consumer metrics.

**114 · Kafka II — delivery semantics: at-least-once vs exactly-once** `ELITE` → `114-kafka-delivery-semantics.md`
- **Mechanical:** "exactly-once" in Kafka means exactly-once *within the Kafka boundary* — an idempotent producer plus transactional read-process-write with `read_committed`. The moment the side effect is a Postgres row or an HTTP call, that guarantee does not extend to it.
- **Mastery:** you can state precisely why end-to-end exactly-once with an external side effect is impossible without idempotency at the sink, and you design for at-least-once + idempotent consumers by default.
- **Mid → Senior:** "we enabled exactly-once semantics" → "which covers Kafka-to-Kafka. Your consumer writes to Postgres and calls a payment API — neither is in that transaction. So the real design is at-least-once delivery with an idempotency key at every sink. Exactly-once is a property of the *effect*, not of the transport."

**115 · Dual-write failure and the transactional outbox** `ELITE` → `115-outbox-pattern.md`
- **Mechanical:** committing to Postgres and then publishing to Kafka is two non-atomic operations; a crash between them loses the event permanently, and reordering them loses the data. The outbox makes the event a row in the *same* transaction, published afterwards by a relay (poller or CDC).
- **Mastery:** you can draw the exact failure window in the naive version, implement the outbox in `orderflow`, and explain why the relay must assume at-least-once.
- **Mid → Senior:** "we publish the event after committing" → "there's a window between commit and publish where a crash loses the event silently, and a `@TransactionalEventListener(AFTER_COMMIT)` has exactly the same window. The outbox makes the event part of the transaction; a relay (polling with `SKIP LOCKED`, or Debezium CDC) publishes it. The relay is at-least-once by construction, so consumers must be idempotent — Topic 116."
- **Spine:** `orderflow` order-placed events move to an outbox table with a `SKIP LOCKED` relay.
- **Drill (§4):** `kill -9` the service between the DB commit and the Kafka publish. Show the order exists with no event. Implement the outbox and repeat the kill — show the event is eventually published.

**116 · Idempotent consumers and idempotency keys** `DIFFERENTIATOR` → `116-idempotency.md`
- **Mastery:** you can implement an idempotency key correctly — including the concurrent-duplicate case, where two requests with the same key arrive simultaneously and a naive check-then-act charges twice (Topic 92's race, at the API layer).
- **Mid → Senior:** "we check if we've seen the message ID" → "check-then-act races under concurrency. The correct form is a unique constraint on the key inserted in the same transaction as the effect, so the database arbitrates. And the key must be supplied by the *client* for API idempotency — a server-generated one doesn't survive a client retry."
- **Spine:** `Idempotency-Key` header on `POST /orders` and `POST /payments`.

**117 · Sagas and compensations** `DIFFERENTIATOR` → `117-sagas.md`
- **Mastery:** you can decompose `orderflow`'s place-order flow into a saga with compensations, and you can name what a saga gives up: atomicity, in exchange for availability, leaving observable intermediate states.
- **Mid → Senior:** "we use a saga instead of a distributed transaction" → "a saga is not atomic — it's a sequence of local transactions with compensations, so intermediate states are visible and compensations can themselves fail. That means the business has to define what a compensated order looks like to a customer. Choreography is loosely coupled and hard to observe; orchestration is observable and centralises the flow. For payments I'd orchestrate."

**118 · Micrometer metrics, RED/USE, and cardinality traps** `DIFFERENTIATOR` → `118-metrics-micrometer.md`
- **Mechanical:** a Prometheus time series is created per unique label-set. Putting an order ID or a user ID in a tag creates one series per value, and the scrape target's memory grows without bound — this kills the monitoring system, not the app.
- **Mastery:** you instrument RED (rate/errors/duration) for every endpoint and USE (utilisation/saturation/errors) for every pool, and you can explain why a histogram is required for meaningful percentiles and why averaging percentiles across instances is invalid.
- **Mid → Senior:** "we track average response time" → "an average hides the tail that the SLO is about. I want a histogram so percentiles are computable server-side — and you cannot average p99s across pods, you have to aggregate the buckets. And the fastest way to take down Prometheus is a high-cardinality tag."
- **Spine:** RED metrics on all `orderflow` endpoints, USE on Hikari and the executors from Topic 90.
- **Drill:** tag a metric with the order ID, run the Topic 65 load, watch series count and scrape memory explode.

**119 · Distributed tracing with OpenTelemetry and context propagation** `DIFFERENTIATOR` → `119-tracing-otel.md`
- **Mechanical:** trace context lives in a `ThreadLocal` and is injected into outbound headers (W3C `traceparent`). Any thread hop that isn't context-aware — `@Async`, a raw executor, a Reactor operator, a virtual thread not created by the instrumented path — drops it silently.
- **Mastery:** you can trace an `orderflow` request end-to-end across HTTP → Kafka → consumer, and you know how to propagate context across every async boundary you introduced in Phases 9–10.
- **Mid → Senior:** "we added the OTel agent, we have tracing" → "you have tracing for the paths the agent instruments. Any hand-rolled executor loses context, and Kafka propagation needs the context in message headers because the consumer runs in a different process entirely. Sampling strategy matters too: head sampling at 1% means you'll never have a trace for the incident you care about — tail sampling on errors and slow requests is what you actually want."
- **Drill (§4):** submit work to a plain `ExecutorService` from a traced request; watch the child span become a new trace. Fix with context propagation, then repeat across a virtual-thread boundary.

**120 · SLF4J/Logback, structured logging, MDC** `CORE` → `120-logging-mdc.md`
- **HONEST ANALOGUE:** SLF4J ≈ a logging facade you already know from Pino/Winston; MDC ≈ AsyncLocalStorage-backed request context. The difference: MDC is `ThreadLocal`-backed and does **not** propagate across threads, whereas `AsyncLocalStorage` follows the async context automatically.
- **Mastery:** you log structured JSON with a correlation ID, you use parameterised logging (`log.debug("order {}", id)`) rather than concatenation, and you know MDC leaks on pooled threads if not cleared.
- **Mid → Senior:** "we log the request ID" → "via MDC, which is a `ThreadLocal` — so it silently vanishes across `@Async` and must be explicitly cleared on pooled threads or the next request inherits the previous one's ID. That's a correctness bug in your logs *and* Topic 79's leak shape."
- **Spine:** structured JSON logging with correlation IDs across all `orderflow` paths.

**121 · Actuator — health, readiness, liveness, and Kubernetes semantics** `CORE` → `121-actuator-k8s-probes.md`
- **Mastery:** you know liveness failure means **restart** and readiness failure means **remove from load balancing**, and that putting a database check in the liveness probe turns a database blip into a cluster-wide restart cascade.
- **Mid → Senior:** "we point both probes at `/actuator/health`" → "then a slow database restarts every pod simultaneously and you've amplified a dependency blip into an outage. Liveness should check only 'is this JVM wedged'. Readiness checks dependencies. And readiness must go false *before* shutdown begins, or you drop in-flight requests during a rolling deploy."
- **Drill:** point liveness at a DB-dependent health group, take Postgres down, watch every pod restart in a loop.

**122 · Dockerising Spring Boot — layered jars, AppCDS, container defaults, startup** `DIFFERENTIATOR` → `122-docker-spring-boot-startup.md`
- **Mechanical:** a fat jar in one Docker layer invalidates the whole layer on every code change; layered jars separate rarely-changing dependencies from your classes. AppCDS memory-maps a pre-parsed class archive, cutting class-loading time — usually the largest slice of Spring startup.
- **Mastery:** you can decompose startup time (JVM init → class loading → context refresh → bean instantiation) and attack the largest slice with evidence.
- **Mid → Senior:** "startup takes 12 seconds" → "measure the split first. If it's class loading, AppCDS or the JDK 25 AOT cache helps. If it's bean instantiation, lazy initialisation or fewer auto-configurations helps. If it's a connection-pool warm-up blocking the context, that's a config problem. Reaching for native-image before measuring is how teams spend a month for the wrong 200 ms."
- **Spine:** `orderflow` image built with layered jars + AppCDS; startup time recorded before and after.

**123 · Configuration, secrets, graceful shutdown, and rolling deploys** `CORE` → `123-config-secrets-graceful-shutdown.md`
- **Mastery:** you implement graceful shutdown correctly — readiness false → drain in-flight → close pools → exit — and you know the default `SIGTERM` grace period interacts with your longest request.
- **Mid → Senior:** "Spring Boot handles shutdown" → "`server.shutdown=graceful` gives you a drain phase, but it must start *after* readiness goes false or the load balancer keeps sending traffic, and the grace period must exceed your p99 request duration or you kill in-flight orders. Kafka consumers need their own drain, and an outbox relay needs to finish its batch."
- **Spine:** zero-dropped-request rolling deploy of `orderflow`, proven under Topic 65's load.

**124 · GATE — production-readiness review of `orderflow`** `DIFFERENTIATOR` → `124-gate-production-readiness.md`
- **Deliverable:** a written production-readiness review covering: SLOs and current attainment, capacity headroom at baseline, failure modes and their mitigations, the runbook for each of Phases 8–11's drills, observability coverage gaps, and the top three risks with owners. This is the artefact a senior engineer is expected to produce and the rehearsal for Phase 12.

---

### PHASE 12 — PRINCIPAL TRACK `125–135`

Every topic here is `ELITE` and is taught as **a decision under ambiguity with an
artefact you must produce**. I review the artefact the way a staff-level reviewer
would: attacking the weakest assumption, not the prose. There are no exercises in
the Phase 1–11 sense; there is a deliverable and an adversarial review.

**125 · Reading the source — OpenJDK, Spring, JEPs** → `125-reading-the-source.md`
- **Artefact:** a written analysis of one unreleased/preview JEP, with a position on whether your organisation should adopt it and what would change your mind.
- **Mastery:** you resolve a behavioural question by reading `ConcurrentHashMap` or `AbstractAutowireCapableBeanFactory` rather than by searching for a blog post, and you can form a defensible opinion on an unreleased feature.
- **Mid → Senior/Principal:** "the docs say..." → "the docs are ambiguous here, so I read `TransactionAspectSupport`; the behaviour comes from *this* branch, and it changed in this commit for this reason."

**126 · Technical strategy — build vs buy vs adopt** → `126-build-buy-adopt.md`
- **Artefact:** a five-year total-cost evaluation of one framework choice for `orderflow`, including exit cost.
- **Mastery:** you evaluate on maintenance cost, governance, exit cost and team capability — not on a benchmark or GitHub stars.
- **Senior → Principal:** "library X is faster in benchmarks" → "X is faster on a synthetic benchmark that doesn't match our access pattern; it has one maintainer, no LTS policy, and no migration path off it. Y is 15% slower and I can hire people who know it. The 15% costs us two instances; the wrong bet costs us a quarter."

**127 · Migration planning I — Java 8→21/25, Boot 2→3→4** → `127-migration-java-and-boot.md`
- **Artefact:** a sequenced migration plan for a hypothetical 400k-line Boot 2.7/Java 8 estate, with rollback points and a stall-prevention mechanism.
- **Mastery:** you sequence by risk and by unblocking (JDK upgrade first on the old framework; `javax`→`jakarta` as a mechanical pass with OpenRewrite; then the framework), and you can say what makes a migration stall at 60% and how to prevent it structurally.
- **Senior → Principal:** "we'll upgrade everything in Q3" → "we upgrade the JDK first while staying on Boot 2 — that's independently valuable and independently revertible. Then `jakarta` mechanically via OpenRewrite. The stall risk is that teams stop when the pain stops; so the plan has a deprecation deadline on the old baseline in CI, not a request for cooperation."

**128 · Migration planning II — monolith → services, and the strangler** → `128-migration-monolith-to-services.md`
- **Artefact:** a strangler plan for extracting `orderflow`'s payments domain, including the data-ownership cut and the rollback plan at each step.
- **Mastery:** you know the hard part is the data boundary, not the code boundary, and you can articulate when *not* to split.
- **Senior → Principal:** "we'll extract payments into its own service" → "the code split is a week; the data split is a quarter, because payments and orders currently join. Step one is enforcing the boundary *inside* the monolith with no shared tables — if we can't do that, extracting will just give us a distributed monolith with network latency in the middle."

**129 · Capacity, cost, and latency budgets** → `129-capacity-cost-latency-budgets.md`
- **Artefact:** a first-principles capacity and cost-per-request model for `orderflow`, calibrated against the Topic 65 baseline, with the break-even point for a proposed optimisation.
- **Mastery:** you can size a JVM fleet from measured allocation rate, live-set size, connection-pool limits and latency budget — and decide whether an optimisation pays for the engineering time.
- **Senior → Principal:** "we need more instances" → "at 800 rps per instance we're pool-bound, not CPU-bound, and Postgres is the real ceiling at 4000 rps. Scaling out past six instances buys nothing and costs $X/month. The read replica costs $Y and moves the ceiling to 12000. Here's the break-even."

**130 · SLO design and error budgets** → `130-slo-error-budgets.md`
- **Artefact:** an SLO document for `orderflow` with SLIs, targets, error budget policy, and a written negotiation position for the product conversation.
- **Mastery:** you can derive an SLO from user-visible behaviour rather than from what's easy to measure, and you can defend "99.9% is the right target and 99.99% is not" with cost.
- **Senior → Principal:** "we target 99.99% uptime" → "on what SLI, measured where, over what window? Availability measured at the load balancer isn't what the user experiences. And the extra nine costs multi-region and on-call load — the question for product is whether the revenue at risk justifies that, and here's the number."

**131 · RFC / design-doc authorship** → `131-design-doc-authorship.md`
- **Artefact:** **you write** a design doc for a significant `orderflow` change. I review it adversarially: I attack the weakest assumption, the unstated alternative, and the missing failure mode — not the prose.
- **Mastery:** your doc states the decision, the alternatives *with the reason each was rejected*, the reversibility, and the specific thing that would prove you wrong.
- **Senior → Principal:** a doc that explains the chosen design → a doc that makes the *decision* legible: what we're trading, what we'd need to see to reverse it, and who is affected.

**132 · Engineering standards and rollout without stalling delivery** → `132-engineering-standards.md`
- **Artefact:** a code-review guideline for a Java team, plus a plan to introduce a static-analysis gate (ErrorProne/NullAway/SpotBugs) on a legacy codebase without a three-month freeze.
- **Mastery:** you introduce gates on *new and changed* code first with a ratcheting baseline, and you can deprecate an internal API across teams with a migration path and a deadline rather than an announcement.
- **Senior → Principal:** "we should enable NullAway" → "on new code only, with the existing violations baselined and the baseline ratcheting down. A gate that fails 4000 existing warnings gets turned off in a week, and then we've spent our credibility."

**133 · Incident command and blameless postmortems** → `133-incident-postmortem.md`
- **Artefact:** a full blameless postmortem for one of your Phase 8/9 drills — treated as a real incident — with a timeline, contributing factors, and an action-item chain where each item has an owner, a due date, and a stated preventive effect.
- **Mastery:** you write contributing factors rather than a root cause, and every action item prevents a *class* of incident rather than the specific instance.
- **Senior → Principal:** "root cause: the developer forgot to bound the queue" → "contributing factors: the executor factory defaults to unbounded; we had no saturation alert; the load test didn't cover overload. Actions: a wrapper that requires an explicit bound, a USE dashboard for every pool, an overload scenario in the load suite. Blaming the developer prevents nothing."

**134 · Technical mentorship and influence without authority** → `134-influence-without-authority.md`
- **Artefact:** a written plan to get one technical decision adopted by three teams that don't report to you, including how you'd handle the team that says no.
- **Mastery:** you make adoption cheaper than non-adoption (a migration tool, a working reference implementation, a paved road) rather than relying on argument or mandate.
- **Senior → Principal:** "I proposed it in the architecture forum" → "I built the reference implementation and migrated one team's service myself, so the cost of adopting is reviewing a PR. The team that said no had a real constraint I'd missed — that changed the design, and it's why the other two trusted it."

**135 · Principal-level interview simulation** → `135-principal-interview-simulation.md`
- **Format:** open-ended architecture discussion with adversarial pushback; "tell me about a technical decision you reversed"; and a deep-dive defence of one of your own Phase 12 artefacts under sustained hostile questioning.
- **Mastery:** you hold a position under pressure *and* change it when the pushback is right — and you can tell the difference in real time.

---

## F. TYPESCRIPT / NODE / NESTJS → JAVA TRANSLATION TABLE

Third column is ruthless, per §1.7. `HONEST ANALOGUE` means you can transfer your
intuition directly. `PARTIAL` means transfer it and then unlearn something
specific. `NO ANALOGUE` means there is nothing to transfer and pretending
otherwise installs a wrong model.

### Language & type system

| You know | Java | Verdict |
|---|---|---|
| Structural interfaces (`{ id: string }` matches by shape) | `interface` matched by name | **PARTIAL** — the mechanism is different enough that your design habits must change. Java needs adapters where TS needs nothing. |
| TS types fully erased at compile time | Generic type erasure | **HONEST ANALOGUE** — the single closest match in the language. Both erase; Java differs by keeping erased signatures in bytecode and inserting casts and bridge methods. |
| `readonly` / `as const` | `final` | **PARTIAL** — `readonly` is compile-time only and shallow; `final` is also shallow but has *memory-model* meaning (safe publication) that TS has no concept of. |
| `strictNullChecks`, `string \| null` | `Optional` + JSpecify `@Nullable` | **PARTIAL** — no compiler enforcement in stock Java. JSpecify + NullAway approximates it; Boot 4 adopts JSpecify portfolio-wide. |
| Discriminated unions + exhaustive `switch` | `sealed interface` + records + pattern matching | **HONEST ANALOGUE** — near-exact, including compiler-checked exhaustiveness. |
| `unknown` / `any` | `Object` | **PARTIAL** — `Object` is `unknown` with a cast; there is no `any` escape hatch, and no structural narrowing. |
| Decorators (`@Injectable()` executes at class definition) | Annotations | **PARTIAL** — annotations are *inert metadata*. They do literally nothing without a processor, a proxy, or reflection reading them. TS decorators run code. |
| Prototype chain | Class hierarchy + vtable dispatch | **PARTIAL** — you cannot monkey-patch, and dispatch is resolved statically to a vtable slot. |
| `Symbol`, index signatures, mapped/conditional types | — | **NO ANALOGUE** |
| Exceptions are always unchecked | Checked exceptions | **NO ANALOGUE** — a compile-time obligation with no TS counterpart, and one that actively shapes API design. |
| Template literals | Text blocks + `formatted()` | **PARTIAL** — no interpolation; Java's string templates were withdrawn, so it's `formatted()` or concatenation. |

### Runtime & concurrency

| You know | Java | Verdict |
|---|---|---|
| Event loop, run-to-completion callbacks | — | **NO ANALOGUE.** Java threads are preempted by the OS between any two bytecodes. The guarantee your entire mental model rests on does not exist. |
| Single-threaded ⇒ no data races | Java Memory Model | **NO ANALOGUE.** Happens-before, visibility and reordering exist because the heap is shared and preemption is real. Do not map this onto the event loop. |
| `worker_threads` | Java threads | **PARTIAL** — workers have isolated heaps and message passing. Java threads share one heap. That one difference generates all of Phase 9. |
| `SharedArrayBuffer` + `Atomics` | Shared heap + `volatile` / `VarHandle` | **HONEST ANALOGUE** — the one place JS has a genuine memory model, and it works the same way for the same reasons. |
| `Promise<T>` / `async`-`await` | `CompletableFuture<T>` | **PARTIAL** — shape matches, threading doesn't: continuations need an explicit executor, `join()` blocks a real thread, exceptions wrap in `CompletionException`, and an unobserved failure is silently dropped. |
| `Promise` is eager | `Mono` is cold and lazy | **PARTIAL — and this is the bug you will write.** No `subscribe()` means nothing runs. |
| RxJS `Observable` | Reactor `Flux` | **HONEST ANALOGUE** — plus backpressure, which RxJS mostly lacks. |
| — | Backpressure / `request(n)` | **NO ANALOGUE** — a demand signal travelling upstream has no Promise or RxJS equivalent. |
| `AsyncLocalStorage` | `ThreadLocal` / MDC / `ScopedValue` | **PARTIAL** — `AsyncLocalStorage` follows the async context automatically; `ThreadLocal` does not cross thread boundaries and leaks on pooled threads. |
| libuv event loop | Netty `EventLoopGroup` | **HONEST ANALOGUE** — same epoll loop, except N loops instead of one. |
| `cluster` module, one process per core | One JVM, many threads | **NO ANALOGUE** — Java scales up inside one process, so shared state is the norm rather than the exception. |
| V8 GC (opaque, untunable) | Pluggable collectors, GC logs, heap dumps | **NO ANALOGUE** — you gain observability and control you have never had, and the obligation to use them. |
| `--max-old-space-size` | `-Xmx` / `MaxRAMPercentage` + a dozen others | **PARTIAL** — and heap is only part of the JVM's footprint (Topic 80). |

### Frameworks & tooling

| You know | Java | Verdict |
|---|---|---|
| Nest DI container | `ApplicationContext` | **HONEST ANALOGUE** — same job, two-phase startup being the notable extra. |
| Nest modules + explicit `imports` | `@Configuration` + component scanning | **PARTIAL** — scanning is implicit and classpath-wide; there is no enforced module boundary without JPMS. |
| Nest provider scopes | Bean scopes + scoped proxies | **PARTIAL** — Nest bubbles REQUEST scope up the chain; Spring injects a proxy. Different cost, same problem. |
| Nest interceptors | Spring AOP advice | **PARTIAL** — Spring's is proxy-based, so self-invocation silently bypasses it. Nest wires at the route boundary, so that trap doesn't exist for you today. |
| Nest guards | Security filter chain + `@PreAuthorize` | **PARTIAL** — filters run *before* the framework's dispatcher, not inside it. |
| Nest exception filters | `@ControllerAdvice` | **HONEST ANALOGUE** |
| `class-validator` | Jakarta Bean Validation | **HONEST ANALOGUE** |
| TypeORM / Prisma | JPA / Hibernate | **PARTIAL** — the persistence context, dirty checking and flush ordering have no Prisma equivalent. This is the largest single conceptual gap in Phase 5. |
| Prisma migrations | Flyway / Liquibase | **HONEST ANALOGUE** |
| `package.json` scripts | Maven lifecycle phases | **PARTIAL** — phases are a fixed ordered model, not arbitrary scripts. |
| npm nested resolution (two versions coexist) | Maven flat classpath, nearest-wins | **NO ANALOGUE** — exactly one version per artifact. Diamond conflicts are real and produce `NoSuchMethodError` at runtime with a clean compile. |
| `npm audit` | OWASP dependency-check / Snyk / SBOM | **HONEST ANALOGUE** |
| Jest | JUnit 5 + Mockito | **PARTIAL** — no module mocking. You cannot intercept an import, so testability is a design property. |
| `ts-node` / nodemon hot reload | Devtools restart / JRebel | **PARTIAL** — class redefinition is limited; a restart is usually the honest answer. |
| Pino / Winston | SLF4J + Logback | **HONEST ANALOGUE** |
| `console.time` | **Never do this in Java** — use JMH | **NO ANALOGUE** — the JIT will optimise away the thing you're timing. Topic 77. |

---

## G. FAILURE-DRILL MAP

Mandatory for every `DIFFERENTIATOR`/`ELITE` topic (§4); `[BONUS]` drills are
high-value additions on `CORE` topics. All drills from Topic 65 onward run
against the running `orderflow` service under load.

| Topic | Break it like this | Capture | The fix proves |
|---|---|---|---|
| 09 `[BONUS]` | Swallow an exception in a `catch` | Missing side effect only | Why "log and continue" is a defect, not a style |
| 13 `[BONUS]` | Mutate a key after insertion into a `HashSet` | `contains` false, `size` unchanged | The contract is a data-integrity guarantee |
| 19 | Deserialize a class with a side-effecting `readObject` | The side effect firing | Untrusted deserialization is code execution |
| 25 | Block inside `ForkJoinPool.commonPool()` | Unrelated parallel-stream latency | The common pool is JVM-global shared state |
| 32 `[BONUS]` | Force a diamond version conflict | `NoSuchMethodError` at runtime | Nearest-wins is silent; BOMs are not optional |
| 40 | `@Transactional` via self-invocation | Committed row after a throw; `$$SpringCGLIB$$` in the class name | Advice lives on the proxy, not on `this` |
| 41 | `@Cacheable` outside `@Transactional` | Rolled-back value served from cache | Aspect ordering is a correctness concern |
| 48 | Mutate a managed entity, never call `save()` | The UPDATE in SQL logs | Dirty checking, not `save()`, writes |
| 49 | Serialize a lazy association after session close | `LazyInitializationException`, then a held connection | The boundary is real; OSIV hides it at a cost |
| 50 | Load 100 orders with lines | Hibernate `Statistics` query count | Only a counter proves an N+1 fix |
| 51 | Native-SQL update behind an L2 cache | Stale read | L2 is only safe if every writer goes through Hibernate |
| 52 | Two concurrent orders, one unit of stock | Oversell | Optimistic vs pessimistic vs atomic UPDATE, with throughput numbers |
| 53 | 50k inserts with `IDENTITY` | Statement count | `IDENTITY` disables JDBC batching outright |
| 54 | Throw a **checked** exception from `@Transactional` | The write commits | Default rollback covers unchecked only |
| 55 | 2-second HTTP call inside a transaction | HikariCP timeouts across all endpoints | A transaction pins a connection for its lifetime |
| 63 | Run PIT on order-total logic | A surviving boundary mutant | Coverage ≠ verification |
| 67 | Throw from a static initializer | `ExceptionInInitializerError` then `NoClassDefFoundError` | The second error is a lie; the first is the truth |
| 68 | Same allocation rate, different lifetimes | Promotion in `-Xlog:gc*` | Retention costs, allocation doesn't |
| 71 | Humongous `byte[]` + high allocation rate | `-Xlog:gc*,gc+humongous=debug` | Naming a pause cause from evidence |
| 72 | Same load under G1 vs ZGC | p99/p999 + throughput | Collector choice follows a latency budget |
| 73 | Counted `int` loop over a huge array during GC | `-Xlog:safepoint*` | TTSP hides inside "GC pause" |
| 74 | Pollute a monomorphic call site | `-XX:+PrintCompilation`, `TraceDeoptimization` | Profile pollution permanently degrades hot code |
| 75 | Let a non-escaping object escape to a static | Allocation profile before/after | Scalar replacement is real and fragile |
| 77 | Naive `nanoTime` loop, then JMH | Both numbers | The gap *is* the lesson |
| 78 | Profile `orderflow` at baseline | Flame graph, CPU vs wall-clock | The two modes answer different questions |
| 79 | Unbounded `static Map` cache under load | Heap dump → MAT dominator tree | Reachability, not allocation, is the leak |
| 79b | `ThreadLocal` never removed on a fixed pool | Bounded-but-permanent retention | The subtlest leak shape in Java |
| 80 | Direct buffers with no released references | RSS vs heap; `jcmd VM.native_memory` | Heap is not the footprint |
| 82 | `--cpus=0.5 --memory=512m`, no JVM flags | `availableProcessors()`, collector, OOMKill | Container awareness is not automatic enough |
| 83 | Native-image build of `orderflow` | Runtime reflection failure | Closed-world analysis moves errors to runtime |
| 85 | `synchronized` inventory decrement under load | JFR monitor-blocked events | Contention is measurable, not guessable |
| 86 | Non-`volatile` stop flag, warmed loop | Hang under JIT, terminates under `-Xint` | The memory model, demonstrated |
| 87 | `volatile i++` | jcstress outcome table | Visibility ≠ atomicity |
| 88 | Publish a partially-constructed `Order` via a plain field | jcstress | Safe publication is not optional |
| 90 | Unbounded queue under overload | Queue depth + p99 + OOM | Bounded queue + rejection policy is backpressure |
| 91 | Blocking JDBC on the common pool | Unrelated work stalls | Always pass an explicit executor |
| 92 | Check-then-act on a CHM idempotency cache | Double wallet charge | Atomic operations don't compose |
| 94 | Wallet/inventory lock-ordering deadlock | `jcmd <pid> Thread.print` deadlock section | Global lock ordering breaks circular wait |
| 95 | `AtomicLong` vs `LongAdder` at 64 threads | JMH `@Threads` | Lock-free ≠ scalable |
| 96 | Two counters in one cache line | JMH + perf counters | Physical contention without logical sharing |
| 98 | `new ExecutorService` per request | Thread count → `unable to create native thread` | Thread leaks are OOM with a different message |
| 101 | `synchronized` across the gateway call, virtual threads | `jdk.tracePinnedThreads` / JFR pinned event | Pinning starves carriers; `ReentrantLock` doesn't |
| 105 | Unbounded push source into a slow `Flux` | Heap growth → OOM | Overflow strategy is a product decision |
| 108 | Correlation ID across a `flatMap` | Logs before/after | `ThreadLocal` cannot survive operator thread hops |
| 109 | Pool of 10, ten nested `REQUIRES_NEW` | Thread dump: all parked in `getConnection` | The pool-vs-thread-pool deadlock |
| 111 | 200 concurrent retries, no jitter | Synchronised retry storm | Retries amplify outages |
| 113 | Consumer slower than `max.poll.interval.ms` | Rebalance rate + lag | Rebalance storms are self-inflicted |
| 115 | `kill -9` between DB commit and Kafka publish | Order exists, event lost | The outbox closes the window |
| 118 | Order ID as a metric tag | Series count, scrape memory | Cardinality kills the monitoring system |
| 119 | Plain `ExecutorService` in a traced request | Orphaned trace | Context propagation is explicit work |
| 121 | Liveness probe checking the database | Cluster-wide restart loop | Liveness and readiness are different questions |

---

## H. FILE LIST & PROGRESS TRACKER

All files live in `/docs/java/`. Tick as each topic's three exercises pass.

### Phase 1 — Core Language
- [ ] `01-primitives-wrappers-autoboxing.md`
- [ ] `02-nominal-vs-structural-typing.md`
- [ ] `03-access-modifiers-packages.md`
- [ ] `04-interfaces-abstract-classes.md`
- [ ] `05-generics-declaration-bounds.md`
- [ ] **REVIEW 1** — cumulative checkpoint (01–05)
- [ ] `06-type-erasure.md`
- [ ] `07-variance-pecs-wildcards.md`
- [ ] `08-exceptions-fundamentals.md`
- [ ] `09-exception-api-design.md`
- [ ] `10-collections-framework.md`
- [ ] **REVIEW 2** — cumulative checkpoint (01–10)
- [ ] `11-list-implementations.md`
- [ ] `12-hashmap-internals.md`
- [ ] `13-equals-hashcode-contract.md`
- [ ] `14-comparable-comparator-treemap.md`
- [ ] `15-linkedhashmap-lru.md`
- [ ] **REVIEW 3** — cumulative checkpoint (01–15)
- [ ] `16-sets-enum-collections.md`
- [ ] `17-immutability-final-safe-publication.md`
- [ ] `18-strings-pool-stringbuilder.md`
- [ ] `19-serialization-and-its-hazards.md`
- [ ] `20-jpms-modules-jlink.md`
- [ ] **REVIEW 4** + **PHASE 1 GATE**

### Phase 2 — Modern Java
- [ ] `21-lambdas-functional-interfaces.md`
- [ ] `22-method-references.md`
- [ ] `23-streams-pipeline-laziness.md`
- [ ] `24-collectors.md`
- [ ] `25-parallel-streams.md`
- [ ] **REVIEW 5** — cumulative checkpoint (01–25)
- [ ] `26-optional.md`
- [ ] `27-records.md`
- [ ] `28-sealed-types.md`
- [ ] `29-pattern-matching.md`
- [ ] `30-text-blocks-var.md`
- [ ] **REVIEW 6** + **PHASE 2 GATE**

### Phase 3 — Build & Supply Chain
- [ ] `31-maven-fundamentals.md`
- [ ] `32-dependency-resolution-boms.md`
- [ ] `33-gradle.md`
- [ ] `34-supply-chain-sbom-cve.md`
- [ ] **PHASE 3 GATE**

### Phase 4 — Spring Core (spine begins)
- [ ] `35-application-context.md`
- [ ] **REVIEW 7** — cumulative checkpoint (01–35)
- [ ] `36-bean-definition-component-scan.md`
- [ ] `37-bean-lifecycle.md`
- [ ] `38-bean-scopes.md`
- [ ] `39-dependency-injection-styles.md`
- [ ] `40-proxying-jdk-cglib-self-invocation.md`
- [ ] **REVIEW 8** — cumulative checkpoint (01–40)
- [ ] `41-aop-aspects-pointcuts.md`
- [ ] **PHASE 4 GATE**

### Phase 5 — Spring Boot & Persistence
- [ ] `42-auto-configuration-mechanics.md`
- [ ] `43-configuration-profiles-properties.md`
- [ ] `44-rest-controllers.md`
- [ ] `45-bean-validation.md`
- [ ] **REVIEW 9** — cumulative checkpoint (01–45)
- [ ] `46-error-handling-problem-detail.md`
- [ ] `47-spring-data-jpa.md`
- [ ] `48-hibernate-persistence-context.md`
- [ ] `49-hibernate-associations-lazy.md`
- [ ] `50-hibernate-n-plus-one.md`
- [ ] **REVIEW 10** — cumulative checkpoint (01–50)
- [ ] `51-hibernate-caching.md`
- [ ] `52-hibernate-locking.md`
- [ ] `53-hibernate-batching-writes.md`
- [ ] `54-transactional-semantics.md`
- [ ] `55-isolation-and-connection-pool.md`
- [ ] **REVIEW 11** — cumulative checkpoint (01–55)
- [ ] `56-spring-security-filter-chain.md`
- [ ] `57-spring-security-authorization-jwt.md`
- [ ] **PHASE 5 GATE**

### Phase 6 — Testing
- [ ] `58-junit5.md`
- [ ] `59-mockito.md`
- [ ] `60-spring-test-slices.md`
- [ ] **REVIEW 12** — cumulative checkpoint (01–60)
- [ ] `61-testcontainers.md`
- [ ] `62-contract-testing.md`
- [ ] `63-mutation-testing-pit.md`
- [ ] `64-property-based-testing.md`
- [ ] **PHASE 6 GATE**

### Phase 7 — GATE
- [ ] `65-gate-load-testing-baseline.md`
- [ ] **REVIEW 13** — cumulative checkpoint (01–65)
- [ ] **BASELINE COMMITTED** to `/docs/java/baselines/` — hard gate for Phase 8

### Phase 8 — JVM Internals
- [ ] `66-jvm-architecture.md`
- [ ] `67-class-loading.md`
- [ ] `68-heap-generations-tlab.md`
- [ ] `69-object-layout-compressed-oops.md`
- [ ] `70-gc-fundamentals.md`
- [ ] **REVIEW 14** — cumulative checkpoint (01–70)
- [ ] `71-g1-in-depth.md`
- [ ] `72-zgc-shenandoah.md`
- [ ] `73-safepoints-ttsp.md`
- [ ] `74-jit-tiered-compilation.md`
- [ ] `75-escape-analysis-inlining.md`
- [ ] **REVIEW 15** — cumulative checkpoint (01–75)
- [ ] `76-bytecode-javap.md`
- [ ] `77-jmh.md`
- [ ] `78-profiling-flamegraphs.md`
- [ ] `79-memory-leaks-heap-dumps.md`
- [ ] `80-off-heap-ffm.md`
- [ ] **REVIEW 16** — cumulative checkpoint (01–80)
- [ ] `81-instrumentation-agents.md`
- [ ] `82-jvm-tuning-containers.md`
- [ ] `83-graalvm-native-image.md`
- [ ] **PHASE 8 GATE**

### Phase 9 — Concurrency
- [ ] `84-threads-vs-event-loop.md`
- [ ] `85-synchronized-monitors.md`
- [ ] **REVIEW 17** — cumulative checkpoint (01–85)
- [ ] `86-jmm-happens-before.md`
- [ ] `87-volatile-and-memory-barriers.md`
- [ ] `88-final-fields-safe-publication.md`
- [ ] `89-wait-notify.md`
- [ ] `90-executors-pool-sizing.md`
- [ ] **REVIEW 18** — cumulative checkpoint (01–90)
- [ ] `91-completablefuture.md`
- [ ] `92-concurrent-collections.md`
- [ ] `93-blocking-queues.md`
- [ ] `94-explicit-locks.md`
- [ ] `95-cas-atomics-longadder.md`
- [ ] **REVIEW 19** — cumulative checkpoint (01–95)
- [ ] `96-false-sharing.md`
- [ ] `97-coordination-primitives.md`
- [ ] `98-concurrency-bug-taxonomy.md`
- [ ] `99-jcstress.md`
- [ ] `100-forkjoinpool.md`
- [ ] **REVIEW 20** — cumulative checkpoint (01–100)
- [ ] `101-virtual-threads.md`
- [ ] `102-structured-concurrency-scoped-values.md`
- [ ] **PHASE 9 GATE**

### Phase 10 — Reactive & Async at Scale
- [ ] `103-nio-netty-event-loop.md`
- [ ] `104-reactor-mono-flux.md`
- [ ] `105-backpressure.md`
- [ ] **REVIEW 21** — cumulative checkpoint (01–105)
- [ ] `106-webflux-vs-mvc.md`
- [ ] `107-loom-vs-reactive.md`
- [ ] `108-reactive-debugging.md`
- [ ] **PHASE 10 GATE**

### Phase 11 — Distributed & Production
- [ ] `109-hikaricp-pool-deadlock.md`
- [ ] `110-spring-cache-redis.md`
- [ ] **REVIEW 22** — cumulative checkpoint (01–110)
- [ ] `111-resilience4j.md`
- [ ] `112-spring-cloud.md`
- [ ] `113-kafka-consumer-groups.md`
- [ ] `114-kafka-delivery-semantics.md`
- [ ] `115-outbox-pattern.md`
- [ ] **REVIEW 23** — cumulative checkpoint (01–115)
- [ ] `116-idempotency.md`
- [ ] `117-sagas.md`
- [ ] `118-metrics-micrometer.md`
- [ ] `119-tracing-otel.md`
- [ ] `120-logging-mdc.md`
- [ ] **REVIEW 24** — cumulative checkpoint (01–120)
- [ ] `121-actuator-k8s-probes.md`
- [ ] `122-docker-spring-boot-startup.md`
- [ ] `123-config-secrets-graceful-shutdown.md`
- [ ] `124-gate-production-readiness.md`
- [ ] **PHASE 11 GATE**

### Phase 12 — Principal Track
- [ ] `125-reading-the-source.md`
- [ ] **REVIEW 25** — cumulative checkpoint (01–125)
- [ ] `126-build-buy-adopt.md`
- [ ] `127-migration-java-and-boot.md`
- [ ] `128-migration-monolith-to-services.md`
- [ ] `129-capacity-cost-latency-budgets.md`
- [ ] `130-slo-error-budgets.md`
- [ ] **REVIEW 26** — cumulative checkpoint (01–130)
- [ ] `131-design-doc-authorship.md`
- [ ] `132-engineering-standards.md`
- [ ] `133-incident-postmortem.md`
- [ ] `134-influence-without-authority.md`
- [ ] `135-principal-interview-simulation.md`
- [ ] **FINAL GATE** — principal-level interview simulation

---

## I. WHAT THIS PLAN DELIBERATELY DOES NOT CONTAIN

Per §0.3, and stated so neither of us drifts:

- **No DSA, no algorithm drills, no LeetCode.** Complexity appears only where it's
  intrinsic to a Java implementation (HashMap treeification, `ArrayDeque` growth).
- **No LLD, no SOLID, no design patterns, no HLD fundamentals.** I will *reference*
  them to connect ideas — "this is the same decision you make when choosing a
  strategy pattern" — and will not re-teach them.
- **No company-tier interview mapping.** I don't have reliable data on which
  company asks what, and inventing it would be worse than useless. The seniority
  rubric per topic is what replaces it.
- **No fabricated tool output.** Every GC log, thread dump, heap dump, flame graph
  and benchmark number in this curriculum comes from a command **you** run. Where
  I'm uncertain about version-specific behaviour, I will say so in one line and
  give you the command that settles it.

---

## J. GENERATION STATUS (resume point)

**95 of 135 topic docs written — 165,000+ lines.** Generation was interrupted twice by
session rate limits, never by a content problem. Every file on disk is a complete
document; agents save each file before starting the next, so nothing is a partial.

### Not yet written (40)

| Phase | Missing |
|---|---|
| 5 — Boot & Persistence | 57 |
| 8 — JVM Internals | 69, 70, 74, 75, 77, 78, 79, 83 |
| 9 — Concurrency | 86, 87, 88, 91, 92, 93, 96, 97, 98, 101, 102 |
| 10 — Reactive | 105, 106, 107, 108 |
| 11 — Distributed & Production | 111, 112, 113, 116, 117, 118, 119, 122, 123, 124 |
| 12 — Principal Track | 127, 128, 129, 132, 133, 134, 135 |

**Highest priority on resume:** 77 (JMH) — roughly fifteen other topics forward-reference
it as the measurement authority; and 86–88 (the Java Memory Model), which 92–102 all
build on.

### Verification performed
- **Template compliance:** all 95 pass — every required section, correct order for tier.
- **R14 concurrency traces:** every written Phase 9 doc carries the two-column
  thread-by-thread interleaving trace before any correct code.
- **No fabricated tool output.** Three GC/safepoint log lines flagged by sweep were
  inspected and are all correctly labelled *"illustration of the format, not captured
  output"* — the permitted teaching exception.
- **Topic 65 (the gate) contains zero invented performance numbers** — 14 explicit
  blank-template labels; its results tables are generated at runtime by the k6
  `handleSummary` function from the learner's own run.
- **No placeholder names** — all use the `orderflow` domain.

### Note on length
Docs run 1,150–3,089 lines against an original 700–950 target. Every agent
independently reported the same cause: the mandatory section set (5–6 traps with
observable symptoms, 5 four-part interview questions, 3 exercises, 5–7 checkpoint
questions, plus per-tier mechanical statement / machine-level reality / failure drill /
measurement) does not compress into 950 lines. Nothing is padded.

### To resume
Regenerate the missing topics above. Restate in full to any agent writing them: the
anti-fabrication rule, R14 (the two-column interleaving trace) for every Phase 9 topic,
and the version-honesty requirements — `StructuredTaskScope`'s API changed across
preview rounds, and `synchronized` pinning was largely removed in JDK 24+, so Topic
101's drill must be informative under both outcomes.
