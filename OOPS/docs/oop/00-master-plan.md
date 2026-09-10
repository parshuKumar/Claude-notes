# OOP Traps + LLD Coding — Master Plan

> **Status:** Plan drafted. Diagnostic issued, **not yet graded.**
> Nothing is taught until the diagnostic is answered and `START` is typed.

---

## 0. Configuration

| Setting | Value |
|---|---|
| **Primary language** | **C++** (all docs, all BUILD exercises default here) |
| **Secondary language** | **Java** — *deferred by explicit decision* |
| C++ toolchain | Apple clang 21.0.0 (clang-2100.1.1.101), `-std=c++20 -Wall -Wextra`, target arm64-apple-darwin25.3.0 |
| Java toolchain | **None installed.** `/usr/bin/java` is the macOS stub. |
| Doc root | `/Users/backend/parshuramPers/Claude-notes/OOPS/docs/oop/` |

### The Java pass (agreed deferral)

Java is **not** interleaved into the C++ docs. Every trap doc gets a
`## Behavior — Java` section that is written as a **stub** now:

```
## Behavior — Java
DEFERRED — Java pass. Divergence predicted: [SAME / DIVERGES ⚠️]. See §2.
```

When every C++ doc is complete, a single **Java pass** runs:
1. Install a JDK (`brew install openjdk@21`) so the HARD ACCURACY RULE holds.
2. Fill in every deferred section with real executed output.
3. Build `98-java-syntax-round.md` — the syntax-round sheet.

Until then, **no Java output is claimed anywhere.** Any Java code that
appears before the Java pass is labelled **NOT EXECUTED — unverified**.

---

## 1. Tiered trap list

Ordered by tier, not category. Work top-down.

### TIER 1 — asked constantly, high consequence. Full doc.

| # | Trap | The confusion it causes |
|---|---|---|
| 01 | Virtual call from constructor | You expect the derived override; you get the base version, because the vptr is rebuilt at each level and the derived object does not exist yet. |
| 02 | Virtual call from destructor | Same mechanism running backwards; a pure virtual call here is a hard crash, not a silent fallback. |
| 03 | Construction / destruction order | Bases before members before body; destruction exactly reversed — and multiple inheritance orders by *declaration in the base list*, not by init-list order. |
| 04 | Member init order = declaration order | The member-initializer list is a set of *values*, not a *sequence*; a member initialized from another member reads garbage if declared first. |
| 05 | Object slicing | Copying a `Derived` into a `Base` by value silently discards the derived part and the vptr; the copy is a genuine `Base`, not a broken `Derived`. |
| 06 | Why polymorphism needs pointers/references | Value semantics resolve the type at compile time; dispatch is not "turned off", the object literally is not there. |
| 07 | Non-virtual destructor + polymorphic delete | `delete base_ptr` on a derived object is UB — you get partial destruction, leaked members, and no diagnostic at runtime. |
| 08 | Name hiding | A derived member named `f` hides **every** base overload of `f`, including ones with different arity. Not overriding — hiding. |
| 09 | Override vs overload vs hide | A `const` mismatch, a reference mismatch, or a different parameter type makes a new function that hides the virtual instead of overriding it. `override` is the only defence. |
| 10 | Default args + virtual dispatch | Default arguments bind **statically** from the pointer's type; the function body dispatches **dynamically**. You get the derived body with the base's default. |
| 11 | Static vs dynamic binding: the resolution table | Which constructs trigger which — virtual vs non-virtual, value vs pointer vs reference, qualified call, default arg, static member, name lookup. |
| 12 | Rule of 0 / 3 / 5 | A raw pointer member with a compiler-generated copy gives you aliasing then a double free — and the crash is at scope exit, far from the bug. |
| 13 | `vector<Base>` vs `vector<unique_ptr<Base>>` | A container of base-by-value slices every element on insertion. The canonical modern-C++ mistake. |
| 14 | Ownership & lifetime in polymorphic hierarchies | Who deletes: `unique_ptr` owner, `shared_ptr` shared owner, raw pointer observer. Getting this wrong is the #1 LLD-round C++ failure. |
| 15 | Exceptions from constructors | Fully-constructed bases and members are destroyed; the object's **own** destructor never runs, because the object never existed. |
| 16 | Diamond problem & virtual inheritance | With `virtual` bases, the *most-derived* class calls the shared base's constructor — intermediate classes' calls to it are ignored. |
| 17 | Liskov violations that compile fine | `Square : Rectangle`, `Stack : Vector`. The compiler is happy; the caller's invariant breaks at runtime. Directly graded in Track B. |
| 18 | Composition vs inheritance — a decision procedure | Not a slogan. A checklist that produces the same answer under interview pressure every time. |

### TIER 2 — asked sometimes. Compressed doc.

| # | Trap | The confusion it causes |
|---|---|---|
| 19 | Pure virtual destructor needs a definition | `virtual ~B() = 0;` compiles but fails to link — the derived destructor still calls it. |
| 20 | Non-virtual method through a base handle | Resolves by the handle's static type. Redefinition in the derived class is invisible. |
| 21 | Covariant return types | Allowed only for pointers/references to classes related by the same hierarchy, and the derived class must be complete. |
| 22 | vtable / vptr mechanics and object layout | Where the vptr sits, what the vtable holds, the size cost, and why the first virtual function is what makes the object polymorphic. |
| 23 | `final` and devirtualization | `final` on a class or an override lets the compiler resolve a virtual call statically. When it actually fires. |
| 24 | Overload resolution: conversions and defaults | Exact match beats promotion beats standard conversion beats user-defined conversion beats ellipsis; ambiguity is an error, not a coin-flip. |
| 25 | Move semantics + inheritance | A derived move constructor that forgets `std::move(base)` silently copies the base subobject; a user-declared destructor kills implicit moves. |
| 26 | `*this` chaining and self-assignment | Returning `*this` vs `this`; why copy-and-swap makes self-assignment safety free. |
| 27 | Copy elision / RVO changes observable copy counts | C++17 guaranteed elision means the copy constructor you instrumented does not run. Trap questions count constructor prints. |
| 28 | Private and protected inheritance | Not is-a. It is "implemented-in-terms-of" with the base interface sealed off, and conversion to base is blocked outside the class. |
| 29 | `protected` across sibling classes | You can access a protected member only through your **own** type, not through a sibling's. C++ and Java differ here. |
| 30 | Private virtual functions (NVI / Template Method) | A private virtual **can** be overridden by a derived class it cannot call. Access control and virtual dispatch are independent. |
| 31 | Abstract base classes & interface segregation in C++ | Pure virtual = interface; what "interface" costs in C++ and where to place the boundary. |
| 32 | Static members: shared state and init order | Non-local static init order across translation units is unspecified; `inline` statics and `constexpr` change the rules. |
| 33 | Singleton traps | Naive lazy init is a data race; double-checked locking needs the memory model; the Meyers singleton is thread-safe since C++11 and is the answer. |
| 34 | `operator==` / `operator<=>` consistency | Comparing across a hierarchy through a base reference compares only the base slice; `<=>` and `==` must stay consistent. |
| 35 | Throwing from a destructor | Destructors are implicitly `noexcept`; throwing during stack unwinding calls `std::terminate`. |
| 36 | RAII vs manual cleanup | Why RAII is correct under exceptions and early returns and `try/finally`-style code is not. |
| 37 | `const` correctness and `mutable` | `const` member functions, logical vs bitwise constness, and why `const` does not imply thread-safe by itself. |
| 38 | Dependency injection vs hard-coded construction | A class that `new`s its collaborators cannot be tested or extended. The Track B rubric checks this every time. |

### TIER 3 — senior / language-specialist. One paragraph each, single appendix file.

`90-appendix-tier3.md`

| # | Trap |
|---|---|
| 39 | Multiple-inheritance object layout, pointer adjustment, thunks |
| 40 | `dynamic_cast` cost, cross-casts, behaviour with RTTI disabled |
| 41 | `typeid` on references vs pointers vs values |
| 42 | Virtual inheritance layout (vbptr) and its runtime cost |
| 43 | Empty base optimization |
| 44 | CRTP / static polymorphism vs virtual dispatch |
| 45 | Strict aliasing and type punning through base pointers |
| 46 | Why there are no virtual template member functions |
| 47 | `std::function` and type erasure as a polymorphism alternative |
| 48 | `shared_ptr` aliasing ctor, `enable_shared_from_this`, `weak_ptr` cycles |
| 49 | Destruction order of static objects at exit, across TUs |
| 50 | `noexcept` and inheritance — an override cannot widen it |
| 51 | Access control does not affect virtual dispatch |
| 52 | Inheriting constructors (`using Base::Base`) and what it does not inherit |
| 53 | `= default` / `= delete` interacting with the rule of five |
| 54 | Slicing when catching an exception by value |

**Cut as definitional, not tricky:** what encapsulation/inheritance/
polymorphism/abstraction *are*; "abstract class vs interface" as a
vocabulary question (kept only as trap 31, the placement decision);
access-specifier tables.

---

## 2. Divergence map — C++ vs Java

The highest-value table here. Same-looking code, different behaviour.
**Verified column is empty until the Java pass** — nothing is claimed
that has not been executed.

| Trap | C++ behaviour | Java behaviour (predicted, unverified) | Root cause |
|---|---|---|---|
| 01 Virtual call in ctor | Base's version runs | **DIVERGES ⚠️** Derived override runs, sees uninitialized fields (`0`/`null`) | C++ rebuilds the vptr per level for type safety; Java installs the final vtable at allocation and initializes fields after `super()` |
| 02 Virtual call in dtor | Base's version runs; pure virtual = crash | **DIVERGES ⚠️** No destructors; `finalize`/`Cleaner` is a different problem entirely | Deterministic destruction vs GC |
| 05 Object slicing | Silently slices | **DIVERGES ⚠️** Cannot happen — all objects are reference-typed | Value semantics vs reference semantics |
| 07 Non-virtual dtor | UB, partial destruction | N/A — no destructors, GC reclaims the whole object | — |
| 08 Name hiding | Derived name hides **all** base overloads | **DIVERGES ⚠️** Base overloads stay visible and participate in resolution | C++ does name lookup then overload resolution; Java merges inherited overloads into the candidate set |
| 10 Default args + virtual | Static default + dynamic body | **DIVERGES ⚠️** No default arguments; overloads emulate them and bind statically anyway | Language feature absent |
| 12/13 Copy semantics | Rule of 3/5, deep copy needed | **DIVERGES ⚠️** Reference assignment aliases; `Cloneable` is broken; copy constructors are the fix | — |
| 16 Diamond | Duplicate base unless `virtual` inheritance | **DIVERGES ⚠️** No state MI; default-method diamond requires explicit `X.super.m()` | Single implementation inheritance |
| 29 `protected` siblings | Access only through your own type | **DIVERGES ⚠️** Also package-visible, which C++ has no equivalent of | Package as an access axis |
| 32/33 Static init | Init order fiasco across TUs | **DIVERGES ⚠️** JLS guarantees class-init order and thread safety | Defined class-loading model |
| 33 Singleton | Meyers singleton | **DIVERGES ⚠️** `enum` singleton; DCL needs `volatile` |Memory model differences |
| 34 Equality | `operator==`, `<=>` | **DIVERGES ⚠️** `==` is identity; `equals`/`hashCode` contract; `Integer` cache; string interning | Operator overloading vs library contract |
| — Arrays | `Base[]` slices by value | **DIVERGES ⚠️** Covariant arrays → `ArrayStoreException` at runtime | — |
| — Generics | Templates monomorphize | **DIVERGES ⚠️** Type erasure, no generic arrays, unchecked warnings | Compile-time codegen vs erasure |
| — Exceptions | No checked exceptions; dtor throw = terminate | **DIVERGES ⚠️** An override cannot broaden checked exceptions | Checked-exception model |
| 36 Cleanup | RAII | **DIVERGES ⚠️** try-with-resources / `AutoCloseable` | — |

---

## 3. Track B — build list

Ordered by increasing difficulty. Each codeable in the stated budget.

| # | System | Budget | Traps exercised |
|---|---|---|---|
| B1 | Vending machine | 40 min | 11, 14, 17, 18, 31 — state as polymorphism vs enum+switch |
| B2 | Parking lot | 40 min | 05, 13, 14, 17, 18, 28 — the canonical inheritance-vs-composition test |
| B3 | LRU cache | 35 min | 12, 14, 26, 37 — ownership of intrusive nodes, no leaks |
| B4 | Rate limiter | 40 min | 18, 31, 33, 37, 38 — strategy + shared mutable state |
| B5 | Splitwise / expense sharing | 45 min | 05, 34, 37, 38 — entity identity, value objects, equality |
| B6 | Logging framework | 40 min | 09, 14, 18, 30, 31, 38 — chain of responsibility, NVI |
| B7 | Notification service | 40 min | 14, 18, 31, 38 — observer, DI, interface segregation |
| B8 | Elevator system | 45 min | 11, 14, 17, 33, 37 — state machine + concurrency touch |
| B9 | Tic-tac-toe → chess | 50 min | 01, 05, 09, 13, 17, 21 — deep hierarchy, maximum Liskov pressure |
| B10 | Library / booking system | 50 min | 12, 14, 15, 34, 37, 38 — full entity model, lifetime, transactions |

---

## 4. Interleave schedule

Traps are re-tested inside code you wrote yourself — that is where
transfer happens. Each BUILD is followed by a **trap autopsy** of your
own code.

| Block | Content |
|---|---|
| **Block 1** | Traps 01 → 04 (construction & init order) |
| → | **BUILD B1 — Vending machine** + autopsy |
| **Block 2** | Traps 05 → 09 (slicing, dispatch, hiding) |
| → | **BUILD B2 — Parking lot** + autopsy |
| **Block 3** | Traps 10 → 14 (binding table, copying, ownership) |
| → | **BUILD B3 — LRU cache**, **BUILD B4 — Rate limiter** + autopsies |
| **Block 4** | Traps 15 → 18 (exceptions, diamond, LSP, composition) |
| → | **BUILD B5 — Splitwise**, **BUILD B6 — Logging** + autopsies |
| **Block 5** | Traps 19 → 28 (TIER 2, compressed) |
| → | **BUILD B7 — Notification service** + autopsy |
| **Block 6** | Traps 29 → 38 (TIER 2, compressed) |
| → | **BUILD B8 — Elevator** + autopsy |
| **Block 7** | TIER 3 appendix (traps 39 → 54) |
| → | **BUILD B9 — Chess**, **BUILD B10 — Library system** + autopsies |
| **Block 8** | **Java pass** — install JDK, fill every deferred section, syntax-round sheet |
| **Block 9** | **Revision sheet** finalized + timed mock |

---

## 5. File list

```
docs/oop/
  00-master-plan.md                     <- this file
  00b-diagnostic.md                     <- STEP 0, 15 questions
  01-virtual-call-in-constructor.md
  02-virtual-call-in-destructor.md
  03-construction-destruction-order.md
  04-member-init-declaration-order.md
  05-object-slicing.md
  06-polymorphism-needs-pointers-refs.md
  07-non-virtual-destructor.md
  08-name-hiding.md
  09-override-overload-hide.md
  10-default-args-virtual-dispatch.md
  11-static-vs-dynamic-binding-table.md
  12-rule-of-zero-three-five.md
  13-containers-slicing-unique-ptr.md
  14-ownership-lifetime-polymorphic.md
  15-exceptions-from-constructors.md
  16-diamond-virtual-inheritance.md
  17-liskov-violations-that-compile.md
  18-composition-vs-inheritance.md
  19-38-*.md                            <- TIER 2, one file each, same numbering
  90-appendix-tier3.md                  <- traps 39-54, one paragraph each
  95-build-01-vending-machine.md
  95-build-02-parking-lot.md
  95-build-03-lru-cache.md
  95-build-04-rate-limiter.md
  95-build-05-splitwise.md
  95-build-06-logging-framework.md
  95-build-07-notification-service.md
  95-build-08-elevator.md
  95-build-09-chess.md
  95-build-10-library-system.md
  97-divergence-map.md                  <- expanded §2, written in the Java pass
  98-java-syntax-round.md               <- Java pass only
  99-revision-sheet-1hr.md              <- see §6
```

---

## 6. The 1-hour revision sheet — `99-revision-sheet-1hr.md`

A single file that lets the whole curriculum be revised in **60 minutes**
the night before an interview. It is **not** a summary of the docs — it is
a recall instrument. Built **incrementally**: after each block finishes,
that block's rows are appended, so it is never a big end-of-project task.

Hard budget, enforced by section:

| Section | Time | Content |
|---|---|---|
| **A. Trap one-liners** | 12 min | All 54 traps, one line each: *trap → resolution*. Pulled verbatim from each doc's "Quick reference card". Scannable table, no prose. |
| **B. The divergence table** | 6 min | §2, condensed to one screen. The single highest-yield page. |
| **C. The binding resolution table** | 5 min | One table: construct × what determines the call. Covers traps 08–11, 20, 24, 51 at once. |
| **D. Order-of-events cheat card** | 5 min | Construction order, destruction order, init-list vs declaration order, virtual-base order, what runs when a ctor throws. ASCII, one card. |
| **E. Ownership decision card** | 4 min | `unique_ptr` / `shared_ptr` / `weak_ptr` / raw observer / value — pick one in 5 seconds. |
| **F. "The answer that gets you hired"** | 10 min | The 18 TIER-1 spoken answers, verbatim, each under 90 seconds. Read aloud, do not skim. |
| **G. LLD skeleton** | 8 min | The reusable 40-minute attack plan: clarify → nouns → boundaries → interfaces → code → extend. Plus the 6 clarifying questions that work on any prompt. |
| **H. Rubric self-check** | 4 min | The 8 Track B rubric lines, as questions to ask of your own code. |
| **I. Red-flag list** | 6 min | Every trap **you personally walked into** during Track B autopsies. Grows over the curriculum. The most valuable section in the file because it is yours. |

Rules for the file:
- Nothing in it that is not directly recitable or directly actionable.
- No code blocks longer than 6 lines.
- Every claim traceable to a doc number so you can drill down.
- Section I is never edited down — it is the personalized part.

---

## 7. Progress tracker

### Setup
- [x] Toolchain verified — Apple clang 21, `-std=c++20 -Wall -Wextra`
- [x] Doc root created
- [x] Master plan written
- [ ] Diagnostic answered
- [ ] Diagnostic graded → SKIP/SKIM/FULL assigned
- [ ] `START` given

### TIER 1
- [x] 01 Virtual call in constructor
- [ ] 02 Virtual call in destructor
- [ ] 03 Construction / destruction order
- [ ] 04 Member init = declaration order
- [ ] **B1 Vending machine** + autopsy
- [ ] 05 Object slicing
- [ ] 06 Polymorphism needs pointers/refs
- [ ] 07 Non-virtual destructor
- [ ] 08 Name hiding
- [ ] 09 Override vs overload vs hide
- [ ] **B2 Parking lot** + autopsy
- [ ] 10 Default args + virtual dispatch
- [ ] 11 Static vs dynamic binding table
- [ ] 12 Rule of 0/3/5
- [ ] 13 Containers, slicing, `unique_ptr`
- [ ] 14 Ownership & lifetime
- [ ] **B3 LRU cache** + autopsy
- [ ] **B4 Rate limiter** + autopsy
- [ ] 15 Exceptions from constructors
- [ ] 16 Diamond & virtual inheritance
- [ ] 17 Liskov violations that compile
- [ ] 18 Composition vs inheritance
- [ ] **B5 Splitwise** + autopsy
- [ ] **B6 Logging framework** + autopsy

### TIER 2
- [ ] 19–28
- [ ] **B7 Notification service** + autopsy
- [ ] 29–38
- [ ] **B8 Elevator** + autopsy

### TIER 3
- [ ] 90 Appendix (39–54)
- [ ] **B9 Chess** + autopsy
- [ ] **B10 Library system** + autopsy

### Closing
- [ ] Java pass — JDK install, all deferred sections filled
- [ ] 98 Java syntax-round sheet
- [ ] 97 Divergence map expanded
- [ ] 99 Revision sheet finalized (sections A–I)
- [ ] Timed mock: 1 trap round + 1 LLD round

---

## 8. Diagnostic results

**PENDING — diagnostic issued, not yet answered.**

Once graded, every trap in §1 gets a tag and the plan is rewritten
to skip what is already known:

- **SKIP** — demonstrated correct, including the mechanism. No doc written.
- **SKIM** — right answer, wrong or missing mechanism. Compressed doc:
  mechanism + "answer that gets you hired" + checkpoint only.
- **FULL** — wrong answer, or right answer for the wrong reason. Full doc.

| Q | Trap(s) probed | Answer | Verdict |
|---|---|---|---|
| 1 | 01, 02 — virtual call in ctor and dtor | | |
| 2 | 05, 06, 13 — slicing by value and in containers | | |
| 3 | 04 — member init = declaration order | | |
| 4 | 08 — name hiding | | |
| 5 | 10 — default args + virtual dispatch | | |
| 6 | 07, 03 — non-virtual dtor, destruction order | | |
| 7 | 16 — virtual inheritance, who calls the base ctor | | |
| 8 | 09 — const mismatch hides instead of overrides | | |
| 9 | 15 — exception from a constructor | | |
| 10 | 32, 33 — static init order, Meyers singleton | | |
| 11 | 30 — private virtual, NVI | | |
| 12 | 12 — rule of three, shallow copy | | |
| 13 | 17 — Liskov | | |
| 14 | 18 — inheritance for orthogonal dimensions | | |
| 15 | 31, 38 — interface segregation | | |

---

## 9. Session persistence

Files do not survive between chat sessions.

- End of every session: this file is re-output **in full** for local saving.
- Start of every session: re-upload it. Where we stopped is restated in
  two lines and work resumes. Completed docs are never regenerated.
- No plan uploaded → the plan is requested before any teaching.

Local copy lives at:
`/Users/backend/parshuramPers/Claude-notes/OOPS/docs/oop/00-master-plan.md`

---

## 10. Commands

`START` · `NEXT` · `HINT` · `PROGRESS` · `REDO [id]` · `DRILL [topic]` · `BUILD`
