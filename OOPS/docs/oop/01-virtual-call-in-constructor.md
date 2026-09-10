# 01 — Virtual Call From a Constructor

## Tier: 1 | C++ vs Java: DIVERGES ⚠️ (Java pass deferred — see §Divergence verdict)

> **Toolchain for every executed block in this file:**
> Apple clang 21.0.0 (clang-2100.1.1.101), target `arm64-apple-darwin25.3.0`,
> compiled with `clang++ -std=c++20 -Wall -Wextra`.
> Every output shown is real stdout from running the program.

---

## The question, verbatim

> "What does this print, and why?"
>
> ```cpp
> struct Shape {
>     Shape() { draw(); }
>     virtual void draw() const { std::cout << "Shape\n"; }
> };
> struct Circle : Shape {
>     void draw() const override { std::cout << "Circle\n"; }
> };
> int main() { Circle c; }
> ```
>
> Follow-up, asked immediately after: *"Now make `draw()` pure virtual. What happens?"*

---

## The trap

The confident wrong answer is **"Circle"**.

It feels correct because it *is* the correct general rule. `draw()` is
virtual, the object being constructed really is a `Circle`, and every other
place in the program — `c.draw()`, `shapePtr->draw()`, `shapeRef.draw()` —
genuinely does dispatch to `Circle::draw`. Candidates reason "virtual means
the most-derived override wins," which is true everywhere except during
construction and destruction.

The second half of the trap: **the compiler does not warn you.** I checked —
not with `-Wall -Wextra`, and not even with `-Weverything`:

```
$ clang++ -std=c++20 -Weverything -Wno-c++98-compat -Wno-padded \
          -Wno-weak-vtables -Wno-unsafe-buffer-usage -fsyntax-only a.cpp
warning: include location '/usr/local/include' is unsafe for cross-compilation [-Wpoison-system-directories]
1 warning generated.
```

That is the only warning. Nothing about the virtual call. The **pure**
virtual variant *is* diagnosed (shown below), which misleads people into
believing "no warning ⇒ no problem" for the ordinary case.

---

## The mechanism

This is the whole point of the doc. Memorising "base version wins" gets you
one question; the mechanism gets you every variant.

**A C++ object is built one layer at a time, and its dynamic type changes as
it is built.**

Constructing a `Circle` runs, in order:

1. `Shape`'s constructor.
2. `Circle`'s member initializers (`radius`).
3. `Circle`'s constructor body.

At step 1, the `Circle` part does not exist yet. `radius` is raw memory.
If a virtual call dispatched to `Circle::draw` at that moment, it would read
an uninitialized `radius`. C++ refuses to allow that, and it enforces the
refusal *mechanically*: **each constructor sets the vptr to its own class's
vtable before running its body.**

So the vptr is rewritten at every level on the way down:

```
Shape::Shape() runs  →  vptr points at Shape's vtable   →  draw() resolves to Shape::draw
Circle::Circle() runs →  vptr points at Circle's vtable  →  draw() resolves to Circle::draw
```

The dispatch is still *fully dynamic* — it is a real vtable lookup, not the
compiler quietly rewriting your call into a static one. It is just that the
vtable it looks in is the one for the class whose constructor is currently
running. **The object's dynamic type during `Shape::Shape()` is literally
`Shape`, not `Circle`.**

Three consequences fall straight out of this, and they are what variants test:

- It does not matter **how** the virtual is reached. A constructor calling a
  non-virtual helper that calls the virtual behaves identically — the vptr
  is what decides, not the call syntax.
- It does not matter if the override is `final`, or the class is `final`.
  `final` enables devirtualization *when the static type is known to be the
  most-derived one*; inside `Shape::Shape()` the object is not yet a `Circle`,
  so there is nothing to devirtualize to.
- If the function is **pure** virtual, there is no `Shape::draw` to land on.
  The vtable slot holds a poison entry, and calling it aborts the process.

### Proof 1 — the dynamic type really does change

`typeid(*this)` is a runtime query on the vptr. Watch it change:

```cpp
#include <iostream>
#include <typeinfo>
struct A { A() { std::cout << "A ctor sees: " << typeid(*this).name() << "\n"; } virtual ~A() = default; };
struct B : A { B() { std::cout << "B ctor sees: " << typeid(*this).name() << "\n"; } };
struct C : B { C() { std::cout << "C ctor sees: " << typeid(*this).name() << "\n"; } };
int main() { C c; std::cout << "main sees:  " << typeid(c).name() << "\n"; }
```

```
A ctor sees: 1A
B ctor sees: 1B
C ctor sees: 1C
main sees:  1C
```

(`1A` is the Itanium-ABI mangled name for `A`.) Three different dynamic types
for one object, at three moments in its life.

### Proof 2 — watch the vptr itself get rewritten

The vptr is the first word of a polymorphic object on this ABI. Reading it
directly is not portable, but it makes the mechanism visible:

```cpp
#include <iostream>
struct A {
    A() { show("A ctor"); }
    virtual void f() {}
    virtual ~A() = default;
    void show(const char* where) const {
        const void* const* vp = reinterpret_cast<const void* const*>(this);
        std::cout << where << ": vptr = " << *vp << "\n";
    }
};
struct B : A { B() { show("B ctor"); } void f() override {} };
struct C : B { C() { show("C ctor"); } void f() override {} };
int main() { C c; c.show("main  "); }
```

```
A ctor: vptr = 0x104470140
B ctor: vptr = 0x104470118
C ctor: vptr = 0x1044700b0
main  : vptr = 0x1044700b0
```

Three distinct vtable addresses during construction. The value installed by
`C`'s constructor is the one the object keeps. *(Addresses vary per run under
ASLR; the pattern — three different values, last one equals main's — does not.)*

---

## Behavior — C++

### The base case

```cpp
#include <iostream>
#include <string>

struct Shape {
    Shape() {
        std::cout << "Shape ctor -> ";
        draw();                       // virtual call from a constructor
    }
    virtual void draw() const { std::cout << "drawing a generic Shape\n"; }
    virtual ~Shape() = default;
};

struct Circle : Shape {
    int radius;
    Circle(int r) : radius(r) {
        std::cout << "Circle ctor -> ";
        draw();
    }
    void draw() const override {
        std::cout << "drawing a Circle of radius " << radius << "\n";
    }
};

int main() {
    Circle c(7);
    std::cout << "--- fully constructed ---\n";
    c.draw();
    Shape* s = &c;
    s->draw();
}
```

**Executed output** (no compiler diagnostics at all):

```
Shape ctor -> drawing a generic Shape
Circle ctor -> drawing a Circle of radius 7
--- fully constructed ---
drawing a Circle of radius 7
drawing a Circle of radius 7
```

Line 1 is the trap. Lines 3–4 prove dispatch is working perfectly once the
object exists — which is exactly why the behaviour feels inconsistent until
you know the mechanism.

### It is not about the call syntax

Routing the call through a non-virtual helper changes nothing:

```cpp
#include <iostream>
struct Base {
    Base() { helper(); }                                   // non-virtual helper...
    void helper() { std::cout << "helper -> "; log(); }    // ...which calls a virtual
    virtual void log() { std::cout << "Base::log\n"; }
    virtual ~Base() = default;
};
struct Derived : Base { void log() override { std::cout << "Derived::log\n"; } };
int main() { Derived d; }
```

```
helper -> Base::log
```

The vptr decides, not the shape of the call site. This matters in real code,
where the virtual is usually three frames below the constructor and nobody
notices.

### The pure virtual variant — the follow-up question

```cpp
#include <iostream>
struct Base {
    Base() { std::cout << "Base ctor calling pure virtual...\n" << std::flush; init(); }
    virtual void init() = 0;          // pure virtual
    virtual ~Base() = default;
};
struct Derived : Base { void init() override { std::cout << "Derived::init\n"; } };
int main() { Derived d; std::cout << "never reached\n"; }
```

**Compiler diagnostic — this one it does catch:**

```
d.cpp:3:80: warning: call to pure virtual member function 'init' has undefined behavior;
    overrides of 'init' in subclasses are not available in the constructor of 'Base'
    [-Wcall-to-pure-virtual-from-ctor-dtor]
```

**Executed output:**

```
Base ctor calling pure virtual...
libc++abi: Pure virtual function called!
[exit 134]
```

Exit 134 = SIGABRT. This is **undefined behaviour** per the standard
([class.cdtor]/4); the abort with that message is what *this* implementation
chose to do, and it is the friendliest possible outcome, not a guaranteed one.
The Itanium ABI fills the vtable slot with `__cxa_pure_virtual`, which prints
and aborts.

### `final` does not save you

```cpp
struct Base {
    Base() { std::cout << "Base ctor -> "; log(); }
    virtual void log() const { std::cout << "Base::log\n"; }
    virtual ~Base() = default;
};
struct Derived final : Base {
    void log() const final { std::cout << "Derived::log\n"; }
};
int main() { Derived d; }
```

```
Base ctor -> Base::log
```

Both the class and the override are `final`, and it still prints `Base::log`.
Devirtualization needs a known most-derived type; during `Base::Base()` the
object is not yet one.

### The fix that survives review

Do not call virtuals during construction. If a derived class must contribute
behaviour at creation time, finish construction first, then dispatch — a
factory makes that structural rather than a convention people forget:

```cpp
#include <iostream>
#include <memory>
#include <utility>

struct Shape {
    virtual void draw() const = 0;
    virtual ~Shape() = default;
protected:
    Shape() = default;                 // only a factory can build one
    virtual void onCreated() { draw(); }
public:
    // Construction finishes BEFORE any virtual call is made.
    template <class T, class... Args>
    static std::unique_ptr<T> create(Args&&... args) {
        auto p = std::unique_ptr<T>(new T(std::forward<Args>(args)...));
        p->onCreated();                // object is complete here -> real dispatch
        return p;
    }
};

struct Circle : Shape {
    int radius;
    explicit Circle(int r) : radius(r) {}
    void draw() const override { std::cout << "Circle r=" << radius << "\n"; }
};

int main() {
    auto c = Shape::create<Circle>(7);
    std::cout << "--- later ---\n";
    c->draw();
}
```

```
Circle r=7
--- later ---
Circle r=7
```

The first line is now the derived override — because `new T(...)` has already
returned. Two cheaper fixes, preferred when they apply:

- **Make it non-virtual and pass the data in.** If the base needs a value
  from the derived class, take it as a constructor parameter, not a virtual
  call. `Shape(std::string name)` beats `virtual std::string name() const`.
- **Pass the behaviour in as a strategy.** A `std::function` or a policy
  object handed to the base constructor is fully formed before the base
  constructor runs, so calling it is safe.

---

## Behavior — Java

**DEFERRED — Java pass.** Predicted divergence: **DIVERGES ⚠️**.
No Java output is claimed until a JDK is installed and the code is executed.
See the master plan §0 for the deferred-Java protocol.

---

## Divergence verdict

**DIVERGES ⚠️ — and this is the flagship divergence of the whole curriculum.**
The reasoning below is language-design analysis, not an output claim.

C++ and Java both face the same problem: a base constructor may call a method
the derived class has overridden, and the derived class's fields are not yet
initialized. They pick **opposite** resolutions:

- **C++ changes the object's type as it is built.** The vptr is rewritten by
  every constructor, so a virtual call during base construction resolves to
  the base. You never reach a derived override that could read an
  uninitialized derived member — the language makes the unsafe dispatch
  impossible. The cost is the surprise, plus a hard abort if the function is
  pure virtual.
- **Java installs the final method table when the object is allocated.** An
  object's dynamic type is fixed from birth, so a call from a superclass
  constructor reaches the *subclass* override — which then runs before the
  subclass's field initializers and constructor body. The cost is that the
  override observes fields at their default values (`0`, `false`, `null`),
  and a `final` field can be observed *changing*.

Same trap question, opposite answers, and — critically — **opposite failure
modes**. C++ gives you the wrong *method*. Java gives you the right method
with the wrong *data*, which is harder to spot and hits production more often.

The shared lesson is identical in both: **never call an overridable method
from a constructor.** Both language communities converged on that rule from
different directions.

*(Concrete Java code, real `javac` warnings, and executed output land here in
the Java pass.)*

---

## Memory / object layout

Constructing `Circle c(7)` — one object, three states:

```
STEP 1 — memory allocated, nothing initialized
      +--------------------+
 this |  vptr    ????????  |  <- garbage
      |  radius  ????????  |  <- garbage
      +--------------------+

STEP 2 — Shape::Shape() begins: vptr := &Shape_vtable, THEN body runs
      +--------------------+                  Shape vtable
 this |  vptr  -----------------------------> +------------------+
      |  radius  ????????  |  still garbage   | ~Shape()         |
      +--------------------+                  | Shape::draw      |  <-- draw() lands HERE
                                              +------------------+
      dynamic type == Shape.  draw() -> "drawing a generic Shape"
      radius is untouched, which is EXACTLY what this rule protects.

STEP 3 — Circle's mem-init list: radius := 7
      +--------------------+
 this |  vptr  ---------------> Shape vtable   (vptr not yet updated)
      |  radius     7      |
      +--------------------+

STEP 4 — Circle::Circle() body: vptr := &Circle_vtable, THEN body runs
      +--------------------+                  Circle vtable
 this |  vptr  -----------------------------> +------------------+
      |  radius     7      |                  | ~Circle()        |
      +--------------------+                  | Circle::draw     |  <-- draw() lands HERE
                                              +------------------+
      dynamic type == Circle.  draw() -> "drawing a Circle of radius 7"

STEP 5 — construction complete. vptr stays at Circle's vtable for the
         object's whole life (until destruction walks it back down).
```

The vptr rewrite in steps 2 and 4 is why `typeid(*this)` printed `1A`, `1B`,
`1C` and why the three raw vptr values differed. Destruction runs this diagram
in reverse — which is trap 02.

---

## The answer that gets you hired

> "It prints `Shape`, not `Circle`.
>
> The general rule is that a virtual call dispatches to the most-derived
> override, and that holds everywhere — except during construction and
> destruction. The mechanism is that **each constructor installs its own
> class's vtable pointer before running its body**, so while `Shape`'s
> constructor is executing the object's dynamic type genuinely *is* `Shape` —
> `typeid(*this)` would say `Shape`. The dispatch is still a real vtable
> lookup; it just looks in `Shape`'s vtable.
>
> That is a deliberate safety guarantee, not an accident. If it dispatched to
> `Circle::draw`, that function could read `radius`, which hasn't been
> initialized yet. C++ makes the unsafe call impossible.
>
> Two things follow. If `draw()` were **pure** virtual there'd be no base
> implementation to land on — that's undefined behaviour, and in practice
> libc++ aborts with 'Pure virtual function called'. And it doesn't matter how
> the call is routed: through a non-virtual helper, or with `final` on the
> override — the vptr decides, so it still lands on `Shape`.
>
> The practical rule is: never call an overridable function from a constructor
> or destructor. If a derived class must contribute at creation time, use a
> factory that constructs the object fully and then calls the hook, or pass the
> value in as a constructor parameter.
>
> Worth adding — **Java does the opposite**: it installs the final vtable at
> allocation, so the *subclass* override runs and sees fields at their default
> values. Wrong method in C++, right method with wrong data in Java. Same
> rule either way."

---

## Likely follow-up

**Q: "Does the compiler warn you?"**

No — not for the ordinary virtual case. I verified that `-Wall -Wextra`
produces nothing, and neither does `-Weverything`. It *does* warn for the
pure-virtual form: `-Wcall-to-pure-virtual-from-ctor-dtor`. The asymmetry is
itself a trap, because it trains people to trust silence. Static analysers
catch the general case — clang-tidy's
`clang-analyzer-optin.cplusplus.VirtualCall` is the check to name.

**Q: "So how *do* you let a derived class customise creation?"**

Three options, in order of preference: pass the varying data as a constructor
parameter; pass a strategy object/`std::function` the base can call; or a
static factory that does `new T(...)` and *then* calls the virtual hook on the
completed object. The last is shown above. Two-phase init as a bare convention
("remember to call `init()`") is the weak version — it relies on every caller
being disciplined, and a factory makes it structural.

**Q: "Is a virtual call in the member-initializer list any different?"**

No, and it's slightly worse — see checkpoint 1. `Base() : tag(compute())` runs
before the base's own body, and still resolves to the base's version.

---

## Practice — predict before running

Write your predicted stdout for all three **before** compiling.

### Exercise 1 — easy variation

```cpp
#include <iostream>
struct Employee {
    Employee() { std::cout << "Employee ctor -> "; role(); }
    virtual void role() const { std::cout << "Employee\n"; }
    virtual ~Employee() = default;
};
struct Manager : Employee {
    Manager() { std::cout << "Manager ctor -> "; role(); }
    void role() const override { std::cout << "Manager\n"; }
};
struct Director : Manager {
    Director() { std::cout << "Director ctor -> "; role(); }
    void role() const override { std::cout << "Director\n"; }
};
int main() { Director d; std::cout << "--- done ---\n"; d.role(); }
```

### Exercise 2 — medium variation

Now with a member object and a destructor. Predict every line, in order.

```cpp
#include <iostream>
struct Badge {
    Badge()  { std::cout << "Badge ctor\n"; }
    ~Badge() { std::cout << "Badge dtor\n"; }
};
struct Employee {
    Employee()          { std::cout << "Employee ctor -> "; role(); }
    virtual ~Employee() { std::cout << "Employee dtor -> "; role(); }
    virtual void role() const { std::cout << "Employee\n"; }
};
struct Manager : Employee {
    Badge b;
    Manager()  { std::cout << "Manager ctor -> "; role(); }
    ~Manager() override { std::cout << "Manager dtor -> "; role(); }
    void role() const override { std::cout << "Manager\n"; }
};
int main() { Manager m; }
```

### Exercise 3 — combines trap 01 with trap 05 (object slicing)

The copy constructor is a constructor too. Predict all seven lines.

```cpp
#include <iostream>
struct Account {
    Account() { std::cout << "Account ctor -> "; kind(); }
    Account(const Account&) { std::cout << "Account copy ctor -> "; kind(); }
    virtual void kind() const { std::cout << "Account\n"; }
    virtual ~Account() = default;
};
struct SavingsAccount : Account {
    SavingsAccount() { std::cout << "Savings ctor -> "; kind(); }
    void kind() const override { std::cout << "SavingsAccount\n"; }
};
void audit(Account a) { std::cout << "audit -> "; a.kind(); }
int main() {
    SavingsAccount s;
    std::cout << "--- copy ---\n";
    audit(s);
    std::cout << "--- ref ---\n";
    const Account& r = s;
    r.kind();
}
```

Hint on what makes this one hard: there are **two different reasons** a line
prints `Account` instead of `SavingsAccount`, and they are not the same reason.

---

## Related traps

All of these share one root cause: **an object's dynamic type is not constant —
it is built up layer by layer and torn down layer by layer, and the vptr is the
thing that changes.**

| Trap | Shared root cause |
|---|---|
| **02 — virtual call from a destructor** | Same vptr rewriting, running in reverse. The pure-virtual crash is more likely there, because base *and* derived slots can be gone. |
| **03 — construction / destruction order** | The order *is* the mechanism. This trap is a direct consequence of it. |
| **04 — member init = declaration order** | Same theme: the initializer list is not a sequence you control. Both traps punish assuming source order equals runtime order. |
| **15 — exceptions from constructors** | Also decided by "how much of the object exists right now". A ctor that throws destroys only the layers already built. |
| **05 — object slicing** | The other way the vptr surprises you: copying by value gives the copy the *base's* vtable permanently. Exercise 3 combines them. |
| **23 — `final` and devirtualization** | Explains precisely why `final` cannot rescue this case. |

---

## Mental model checkpoint

Five rapid-fire variants. Answer all five before scrolling.

**1.** Virtual call in the member-initializer list, not the body:
```cpp
struct Base {
    int tag;
    Base() : tag(compute()) { std::cout << "tag=" << tag << "\n"; }
    virtual int compute() const { return 1; }
    virtual ~Base() = default;
};
struct Derived : Base { int compute() const override { return 2; } };
int main() { Derived d; }
```

**2.** Force the downcast by hand — does that beat the mechanism?
```cpp
struct Derived;
struct Base {
    Base();
    virtual void log() const { std::cout << "Base::log\n"; }
    virtual ~Base() = default;
};
struct Derived : Base { void log() const override { std::cout << "Derived::log\n"; } };
Base::Base() { static_cast<const Derived*>(this)->log(); }
int main() { Derived d; }
```

**3.** Both calls, from the *derived* constructor:
```cpp
struct Derived : Base {
    Derived() { Base::log(); log(); }
    void log() const override { std::cout << "Derived::log\n"; }
};
```

**4.** A delegating constructor:
```cpp
struct Derived : Base {
    Derived() : Derived(0) { std::cout << "delegating body -> "; log(); }
    Derived(int)          { std::cout << "target body -> ";     log(); }
    void log() const override { std::cout << "Derived::log\n"; }
};
int main() { Derived d; }
```

**5.** `Derived` is `final` and so is the override — does it devirtualize?
(Code in "Behavior — C++" above.)

---
---
---

### Checkpoint answers — all executed, `clang++ -std=c++20 -Wall -Wextra`

**1.** `tag=1`
The mem-init list is still inside `Base`'s construction, so the vptr is
`Base`'s. Slightly worse than the body case: the wrong value is now *stored*
in a member and outlives the constructor.

**2.** `Base::log`
The cast changes the *static* type only. Dispatch reads the vptr, which is
still `Base`'s. This is **undefined behaviour** — [class.cdtor]/3 forbids
converting `this` to a derived type before that derived class's construction
starts — and `Base::log` is simply what this compiler did. Do not rely on it.
The lesson: you cannot cast your way past the mechanism, because the mechanism
is data, not type information.

**3.** `Base::log` then `Derived::log`
Two different reasons for two lines. `Base::log()` is a *qualified* call — it
suppresses virtual dispatch entirely and always calls `Base`'s version. The
bare `log()` is a normal virtual call, and by the time `Derived`'s body runs
the vptr is already `Derived`'s. Only line 1 is unusual, and it is unusual for
a reason unrelated to this trap.

**4.**
```
target body -> Derived::log
delegating body -> Derived::log
```
Both `Derived::log`. Delegating constructors are *not* a base-class layer —
the target `Derived(int)` is itself a `Derived` constructor, so the vptr is
already `Derived`'s when its body runs. Nothing is suppressed. Answering
`Base::log` here means you have pattern-matched "constructor ⇒ base version"
instead of learning the mechanism.

**5.** `Base ctor -> Base::log`
`final` lets a compiler devirtualize when it knows the most-derived type. Inside
`Base::Base()` the object is not yet a `Derived`, so there is nothing to
devirtualize to. `final` is an optimization enabler, not a dispatch override.

---

## Quick reference card

> **Trap:** a virtual call inside a constructor dispatches to the *current
> constructor's* class, not the most-derived override — because every
> constructor installs its own vtable pointer before running its body.
> **Resolution:** never call an overridable function from a constructor; pure
> virtual there is UB and aborts; `final`, helper functions, and manual
> downcasts do not change it; use a factory or pass the data/strategy in.
