# PYTHON — THE COMPLETE DSA REVISION REFERENCE
## Every Idiom, Every Battery, Every Trap — In One Document
### Baseline: Python 3.10+ (LeetCode runtime) | CPython & PyPy notes for Codeforces
### Companion to Patterns 01–38 and the C++/Java/JS References | Interview + CP Ready

---

## HOW TO USE THIS DOCUMENT

Your primary language is C++ (you also know Java and JS). Python is the *easiest* of the four to
write and the *hardest* to make fast — arbitrary-precision ints mean you never fight overflow, but
CPython's speed and the recursion limit are what bite. So **every place Python differs from
C++/Java/JS is flagged with a `> **vs C++:**` note**, and the CP survival topics (fast I/O, PyPy,
recursion, the `%`/`//` sign rules) get real space.

Three ways to read it:

| Situation | Where to go |
|---|---|
| **Coming to Python from C++/Java/JS** | Read the **C++/Java/JS→Python mental-switch box at the end of §37**, then skim Parts I–IV |
| **Mid-problem, forgot the API** — "how do I get a max-heap in Python again?" | **§33** (Recipe Index); heapq/deque/sortedcontainers live in **§14–§15** |
| **Something is wrong or TLEs** | **§36** (Top 30 Python-DSA Mistakes) and **§32** (Debugging & Exceptions) |

**The rule:** if you catch yourself thinking *"I know this in C++ but not the Python way"* — it
should be in here. If it isn't, add it.

### The 11 things a C++/Java/JS person MUST remember in Python (full list in §37)

1. **`int` is unbounded — no overflow, ever.** No `long long`, no `1L<<k`, no 2^53 limit, and
   `(a*b) % m` is always exact. The one place Python is strictly easier.
2. **`//` floors toward −∞** (`-7 // 2 == -4`) and **`%` follows the divisor's sign**
   (`-7 % 3 == 2`) — the *opposite* of C++/Java/JS. A porting landmine.
3. **`[[0]*m]*n` shares one row** — use `[[0]*m for _ in range(n)]`.
4. **Mutable default args persist** across calls — `def f(x, acc=[])` is a bug; use `acc=None`.
5. **`heapq` is a min-heap only** (and a module of functions, not a class) — negate for a max-heap.
6. **No built-in TreeMap/TreeSet** — use `sortedcontainers` (or `bisect` on a sorted list).
7. **Recursion limit is 1000** — deep DFS needs `sys.setrecursionlimit` + a big-stack thread, or go iterative.
8. **CPython is slow** — select **PyPy** on Codeforces, push work into C-level builtins.
9. **`sort` takes `key=`, not a comparator** — use `functools.cmp_to_key` when you truly need one.
10. **`is` is not `==`** (identity vs value; small ints are cached, so `is` "accidentally" works).
11. **Empty set is `set()`, not `{}`** (`{}` is an empty dict).

### Conventions used throughout

- Code runs on **CPython 3.10+** unless tagged (**[3.8]**, **[3.10]**, …) with an availability note (Codeforces offers CPython 3.x + PyPy 3.x; LeetCode runs CPython 3.11).
- `# =>` comments show expected output.
- `> **Gotcha:**` flags traps that cause real WA/TLE/RE.
- `> **vs C++:**` flags where Python behaves differently from the C++/Java/JS you already know.
- Complexity is given for every container operation and algorithm.

---

## TABLE OF CONTENTS

### PART I — CORE LANGUAGE
1. Numbers and Arbitrary Precision
2. Types, Truthiness and Identity
3. Variables, Scope and Closures
4. Functions
5. The Data Model — Classes and Dunder Methods
6. Comprehensions, Generators and Iterators
7. Control Flow and Pythonic Idioms

### PART II — SEQUENCES, STRINGS AND SLICING
8. list — The Workhorse
9. tuple and Sequence Packing
10. str — Immutable Strings
11. array module, bytes, and Fixed-Type Sequences

### PART III — DICTS, SETS AND SPECIALIZED CONTAINERS
12. dict and the collections Mappings
13. set and frozenset
14. deque, heapq — Stack, Queue, Deque, Priority Queue
15. sortedcontainers — The TreeMap/TreeSet Replacement
16. Choosing the Right Container

### PART IV — SORTING, SEARCHING, ITERTOOLS AND MATH
17. Sorting
18. bisect — Binary Search Built In
19. itertools and functools
20. Comparators, Keys and Multi-key Ordering
21. Math and Number Theory Stdlib

### PART V — BIT MANIPULATION
22. Bitwise Operators and the Arbitrary-Precision Reality
23. The Complete Bit-Trick Catalog
24. int Bit Methods, Wide Masks, and Byte Conversion

### PART VI — NUMBERS, PRECISION AND I/O
25. Integer Arithmetic — The No-Overflow World
26. Floats, Decimal and Fractions
27. Fast I/O (the make-or-break CP topic for Python)
28. Randomness, Time and the Runtime

### PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT
29. The Contest Template and Structure
30. Recursion and the Recursion Limit
31. Performance — Why Python Is Slow and How to Cope
32. Debugging, Exceptions and Running

### PART VIII — QUICK REFERENCE AND RECIPES
33. The "How Do I...?" Recipe Index ← **the mid-problem lookup**
34. Master Complexity Table
35. n → Required Complexity Cheat Sheet
36. The Top 30 Python-DSA Mistakes That Cost You Problems
37. 60-Second Warm-Up Drill + C++/Java/JS→Python mental-switch box ← **start here after a break**

---

# PART I — CORE LANGUAGE

## 1. Numbers and Arbitrary Precision

### 1.1 `int` is arbitrary precision — no overflow, ever

Python `int` auto-promotes to as many bits as needed. There is no `long long`, no `1L<<k` trick, no silent wraparound, no `2^53` float-safe-integer ceiling to worry about. This is the single biggest daily relief coming from C++/Java/JS.

```python
print(2**200)
# => 1606938044258990275541962092341162602522202993782792835301376

def factorial(n):
    r = 1
    for i in range(2, n + 1):
        r *= i
    return r

print(factorial(100))
# => 93326215443944152681699238856266700490715968264381621468592963895217599993229915608941463976156518286253697920827223758251185210916864000000000000000000000000
```

> **vs C++:** In C++/Java you'd reach for `__int128`, `long long` with overflow checks, or a bignum library for this. In Python it's just `int`. Never write manual overflow guards or mod-early tricks purely to avoid overflow in Python — only mod-early because the problem *asks* for `% MOD`.

### 1.2 `float` is a 64-bit double — same precision traps as everywhere else

```python
print(0.1 + 0.2)            # => 0.30000000000000004
print(0.1 + 0.2 == 0.3)     # => False

import math
print(math.isclose(0.1 + 0.2, 0.3))          # => True
print(math.isclose(1e10, 1e10 + 1, rel_tol=1e-9))  # => True
```

> **Gotcha:** `float` gives you the exact same IEEE-754 rounding headaches as C++/Java/JS `double`. Arbitrary precision is an `int`-only superpower.

### 1.3 `//` floor division vs `/` true division vs `int()`/`math.trunc` truncation

`/` ALWAYS returns a `float` (even `4 / 2 == 2.0`). `//` is **floor division** — it rounds toward **negative infinity**, not toward zero.

```python
print(7 / 2)     # => 3.5   (always float)
print(7 // 2)    # => 3
print(-7 // 2)   # => -4   (floors toward -inf!)
print(7 // -2)   # => -4
print(-7 // -2)  # => 3

import math
print(int(-7 / 2))     # => -3  (truncates toward zero)
print(math.trunc(-7 / 2))  # => -3
```

> **vs C++:** In C++/Java/JS, integer division truncates toward zero: `-7 / 2 == -3`. In Python, `-7 // 2 == -4`. This is a real porting bug source — if you translate a C++ formula that relies on truncation (e.g. binary-search midpoints on negative ranges, or grid math), `//` will give a DIFFERENT answer. If you need C-style truncating division in Python, use `int(a / b)` or `math.trunc(a / b)` (careful with huge ints losing float precision — for big ints use `a // b` then adjust, or `-(-a // b)` tricks for ceiling).

### 1.4 `%` modulo follows the divisor's sign (opposite of C++/Java/JS)

```python
print(-7 % 3)    # => 2    (Python: sign of divisor)
print(7 % -3)    # => -2
print(7 % 3)     # => 1
print(-7 % -3)   # => -1
```

> **vs C++:** In C++/Java/JS, `%` follows the sign of the **dividend**: `-7 % 3 == -1` in C++, but `2` in Python. This trips up ported code constantly. The upshot: in Python, for **positive** modulus `m`, `a % m` is ALWAYS in `[0, m)` — you do **not** need the classic `((a % m) + m) % m` normalization trick that's required in C++/Java/JS to force a non-negative result. Python's `%` already gives you a non-negative remainder for positive `m`. Writing `((a % m) + m) % m` in Python is harmless but redundant.

```python
MOD = 10**9 + 7
a = -15
print(a % MOD)              # => 999999992   (already correct, non-negative)
print(((a % MOD) + MOD) % MOD)  # => 999999992   (same — the extra wrap is a no-op here)
```

### 1.5 `divmod`, `pow` (and the 3-arg modular exponentiation superpower)

```python
q, r = divmod(17, 5)
print(q, r)   # => 3 2

print(pow(2, 10))          # => 1024
print(2 ** 10)              # => 1024 (same thing)

# THE killer feature: modular exponentiation built in, O(log exp), no manual fast-pow needed
MOD = 10**9 + 7
print(pow(2, 1_000_000, MOD))   # fast, exact — instant
print(pow(3, -1, 7))            # => 5   modular inverse! (base, -1, mod) when gcd(base,mod)==1
```

> **vs C++:** In C++/Java you hand-roll `fast_pow(base, exp, mod)` with a `while` loop and repeated squaring. In Python, `pow(base, exp, mod)` does it natively in C — always prefer it over writing your own loop, and over `(base**exp) % mod` (which builds a HUGE intermediate integer first — slow and memory-heavy for large exponents).

### 1.6 `abs`, `round` (banker's rounding trap), `min`/`max`

```python
print(abs(-5), abs(5))   # => 5 5

# round() uses "round half to even" (banker's rounding) — NOT "round half up"!
print(round(0.5))   # => 0   (rounds to even)
print(round(1.5))   # => 2
print(round(2.5))   # => 2   (rounds to even, NOT 3!)
print(round(2.567, 2))   # => 2.57  (ndigits form behaves more normally, but still binary-float-based)

print(min(3, 1, 4))            # => 1   (variadic)
print(max([3, 1, 4]))          # => 4   (single iterable)
print(min([], default=-1))     # => -1  (avoids ValueError on empty)
print(max(['bb', 'a', 'ccc'], key=len))  # => 'ccc'
```

> **Gotcha:** `round(2.5) == 2`, not 3. This is deliberate ("banker's rounding" to reduce cumulative bias) but is a classic surprise if you expect C-style round-half-up. If you need round-half-up, do `math.floor(x + 0.5)` for positives, or use `Decimal` with an explicit rounding mode (see Part VI).

> **vs C++:** `min`/`max` without arguments on an empty sequence raise `ValueError`, unlike C++ where you'd just have undefined/empty range behavior — always check emptiness or pass `default=`.

### 1.7 Complex numbers (brief)

```python
z = 3 + 4j
print(z.real, z.imag, abs(z))   # => 3.0 4.0 5.0
```
Rarely needed in DSA/CP except geometry-adjacent problems (rotations, FFT-like tricks). `abs(z)` gives magnitude for free.

### 1.8 `Decimal` / `Fraction` — forward reference

For exact decimal arithmetic (money, no float drift) or exact rational arithmetic (fractions that must never lose precision), see **Part VI (`decimal.Decimal`, `fractions.Fraction`)**.

### 1.9 Infinity as a sentinel

```python
import math
INF = float('inf')
best = float('-inf')
print(math.inf, -math.inf)   # => inf -inf
print(INF > 10**18)          # => True — safe sentinel for "unreached" in Dijkstra/DP
print(float('nan') == float('nan'))  # => False (NaN never equals anything, even itself)
```

> Use `float('inf')` / `math.inf` as your "unvisited"/"unreachable" sentinel in Dijkstra, BFS distances, DP tables — it compares correctly against any real number and needs no artificial `INT_MAX` guess.

### 1.10 Base conversions and bit utilities

```python
print(int('101', 2))     # => 5      parse binary string
print(int('ff', 16))     # => 255
print(int('0b101', 0))   # => 5      base 0 = auto-detect from prefix

print(bin(10))    # => '0b1010'
print(hex(255))   # => '0xff'
print(oct(8))      # => '0o10'

print(bin(10)[2:])   # => '1010'   strip the '0b' prefix

print((10).bit_length())   # => 4    minimal bits to represent (excluding sign)
print((0).bit_length())    # => 0
print((255).bit_length())  # => 8

# [3.10] population count (number of set bits) — built in, no manual Brian Kernighan loop
print((0b10110).bit_count())   # => 3   [3.10]+
```

> **[3.10]** `int.bit_count()` — before 3.10 use `bin(x).count('1')` or `gmpy2`/manual Kernighan's bit trick. Codeforces CPython/PyPy both ship 3.10+ as of recent judges; LeetCode runs 3.11, so `bit_count()` is safe there.

---

## 2. Types, Truthiness and Identity

### 2.1 Dynamic typing, `type()` vs `isinstance()`

```python
x = 5
x = "now a string"   # legal — rebinding, not a type error

print(type(5) == int)        # works but fragile with subclasses
print(isinstance(5, int))    # => True — PREFER this
print(isinstance(True, int)) # => True!  bool is a subclass of int (see gotcha below)
```

> Prefer `isinstance()` over `type() ==` — it respects inheritance and is the idiomatic check, e.g. `isinstance(x, (int, float))` for "is numeric".

### 2.2 `None` — the null

```python
x = None
print(x is None)     # => True   — CORRECT way to check
print(x == None)      # => True but flagged by linters — use `is`
```

> **vs C++:** `None` is Python's `nullptr`/`null`/`undefined`. Always compare with `is None` / `is not None`, never `==`, because `==` can be overridden by `__eq__` and give surprising results for custom objects.

### 2.3 `is` (identity) vs `==` (value) — and the small-int cache trap

```python
a = [1, 2]
b = [1, 2]
print(a == b)   # => True   (value equality)
print(a is b)   # => False  (different objects in memory)

c = a
print(a is c)   # => True   (same object)

# SMALL INT CACHE: CPython pre-allocates and reuses int objects in [-5, 256]
x = 100
y = 100
print(x is y)    # => True   ("works" by accident — implementation detail!)

x = 1000
y = 1000
print(x is y)    # => False  (usually — NOT guaranteed, don't rely on it either way)
```

> **Gotcha:** Never use `is` for value comparison of numbers/strings. `is` for small ints "happens" to work due to CPython's internal caching of `-5..256`, which makes bugs LOOK correct in toy tests, then break the moment values leave that range. Always use `==` for value comparisons; reserve `is` for `None`, sentinels, and genuine identity checks.

### 2.4 Truthiness — what's falsy

Falsy: `0`, `0.0`, `0j`, `''`, `[]`, `()`, `{}`, `set()`, `None`, `False`. Everything else — including `'0'` (non-empty string!), `[0]`, `-1` — is truthy.

```python
if not []:
    print("empty list is falsy")   # => prints
if not {}:
    print("empty dict is falsy")   # => prints
if '0':
    print("non-empty string '0' is truthy")  # => prints (classic trap)

arr = []
if not arr:
    print("array is empty")   # idiomatic emptiness check
```

> **vs C++:** In C++ there's no implicit "container is empty" truthiness — you write `arr.empty()`. In Python `if not arr:` IS the idiom for "is this list/dict/set/string empty", and it's genuinely nicer — use it instead of `if len(arr) == 0:`.

### 2.5 `and` / `or` return a VALUE, not a bool

```python
print(3 and 5)     # => 5   (returns second operand if first is truthy)
print(0 and 5)     # => 0   (short-circuits, returns first falsy)
print(0 or 5)      # => 5   (returns first truthy, or last operand)
print('' or 'default')  # => 'default'

x = None
val = x or 10          # common "default value" idiom
print(val)              # => 10
```

> **Gotcha:** `x = val or default` breaks if `val` is a legitimate falsy value you want to keep, e.g. `val = 0` or `val = ''`. `0 or 10` gives `10`, silently discarding the real `0`. Use `val if val is not None else default` when 0/''/[] are valid values, and `or` only when any falsy value is genuinely meant to be replaced.

### 2.6 Chained comparisons — Pythonic bounds checking

```python
i = 5
n = 10
print(0 <= i < n)     # => True   equivalent to (0 <= i) and (i < n)

x = 7
print(1 < x < 5 < 100)   # chains any number of comparisons
```

> A genuine nicety over C++/Java/JS, where `0 <= i < n` either doesn't compile as intended or silently does the wrong thing (`(0<=i) < n` evaluates the boolean as 0/1 then compares to n). In Python this is a real, correct chained AND — use it constantly for array-bounds checks in grid/matrix problems.

### 2.7 Ternary expression

```python
x = 5
label = "even" if x % 2 == 0 else "odd"
print(label)   # => odd
```

### 2.8 `bool` and explicit conversions

```python
print(bool(0), bool(1), bool(''), bool('a'))  # => False True False True
print(int('42'))     # => 42
print(int(3.9))       # => 3    (truncates toward zero)
print(str(42))         # => '42'
print(float('3.14'))   # => 3.14
print(list("abc"))      # => ['a', 'b', 'c']
```

### 2.9 Container equality is deep value equality

```python
print([1, 2, 3] == [1, 2, 3])           # => True
print([1, [2, 3]] == [1, [2, 3]])        # => True  (recursive)
print({'a': 1} == {'a': 1})               # => True
print({1, 2} == {2, 1})                    # => True  (sets ignore order)
print((1, 2) == (1, 2))                     # => True
```

> **vs C++/Java/JS:** In C++, comparing `vector`/`array` with `==` does compare element-wise (like Python) — but in Java, `==` on arrays/collections is REFERENCE equality (`arr1 == arr2` is almost always false unless same object; you need `.equals()` or `Arrays.equals()`), and in JS, `[1,2] === [1,2]` is `false` (reference equality) — you'd need `JSON.stringify` or a deep-equal helper. Python's `==` doing deep structural equality on lists/dicts/sets/tuples out of the box is a genuine relief — no `.equals()`, no `Arrays.equals()`, no manual loop needed for "are these two arrays the same content".

---

## 3. Variables, Scope and Closures

### 3.1 LEGB scope resolution

Name lookup order: **L**ocal → **E**nclosing (outer function) → **G**lobal (module) → **B**uiltin.

```python
x = "global"

def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)   # => local
    inner()
    print(x)        # => enclosing

outer()
print(x)             # => global
```

### 3.2 `global` and `nonlocal` — when you MUST use them

Python functions can **read** outer variables freely, but **assigning** to a name inside a function makes it local by default (unless declared `global`/`nonlocal`). This bites DFS/backtracking code that tries to maintain a running counter or best-answer via a plain outer variable.

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count      # REQUIRED — without this, `count += 1` raises UnboundLocalError
        count += 1
        return count
    return increment

c = make_counter()
print(c(), c(), c())   # => 1 2 3
```

Classic DFS pattern — global-ish accumulator via `nonlocal`:

```python
def count_paths(grid):
    n, m = len(grid), len(grid[0])
    total = 0
    def dfs(r, c):
        nonlocal total
        if r == n - 1 and c == m - 1:
            total += 1
            return
        if r + 1 < n:
            dfs(r + 1, c)
        if c + 1 < m:
            dfs(r, c + 1)
    dfs(0, 0)
    return total

print(count_paths([[0,0],[0,0]]))  # => 2
```

The forgotten-`nonlocal` bug:

```python
def broken_counter():
    count = 0
    def increment():
        count += 1   # BUG: this creates a NEW local `count`, shadows outer one
        return count
    try:
        increment()
    except UnboundLocalError as e:
        print("boom:", e)   # => boom: cannot access local variable 'count' ...

broken_counter()
```

For module-level (global) variables, use `global` instead of `nonlocal`:

```python
total = 0
def add(x):
    global total
    total += x

add(5)
print(total)   # => 5
```

> **Gotcha:** Merely *reading* an outer variable never needs `global`/`nonlocal` — only *rebinding* it (any `=`, `+=`, etc.) does. Mutating a mutable object in place (`outer_list.append(x)`) does NOT need `nonlocal` either, because you're not rebinding the name, just mutating what it points to — this is a common source of confusion.

### 3.3 No block scope — loop/if variables leak out

```python
for i in range(5):
    pass
print(i)   # => 4   — `i` is still visible after the loop!

if True:
    y = 10
print(y)    # => 10  — no block scope for `if` either
```

> **vs C++/Java:** In C++/Java, a variable declared in a `for(...)` or `if(...)` block is scoped to that block and doesn't exist outside it. In Python, `for`/`if`/`while`/`with` do NOT introduce a new scope — only `def`/`class`/lambda do. This means a loop variable is readable (and reusable, dangerously) after the loop ends. Watch for accidentally reusing a leaked loop variable name later in the same function.

### 3.4 Closures — and the late-binding trap in loops

Closures capture *variables by reference to the enclosing scope*, not by value at definition time. In a loop, all closures created in that loop share the SAME variable, so by the time they're called, the loop variable has its FINAL value.

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])   # => [2, 2, 2]   NOT [0, 1, 2] !!

# FIX: bind the value via a default argument, evaluated at lambda-DEFINITION time
funcs_fixed = [lambda i=i: i for i in range(3)]
print([f() for f in funcs_fixed])   # => [0, 1, 2]
```

Same trap with regular nested functions built in a loop:

```python
def make_adders_broken():
    adders = []
    for i in range(3):
        def add(x):
            return x + i          # captures `i` by reference
        adders.append(add)
    return adders

fns = make_adders_broken()
print([f(10) for f in fns])   # => [13, 13, 13]  — all use the final i == 2

def make_adders_fixed():
    adders = []
    for i in range(3):
        def add(x, i=i):        # default arg captures current value
            return x + i
        adders.append(add)
    return adders

print([f(10) for f in make_adders_fixed()])   # => [10, 11, 12]
```

> **Gotcha:** This is THE classic Python closures-in-a-loop bug. Whenever building a list of lambdas/functions inside a `for` loop that reference the loop variable, default-arg-capture it: `lambda i=i: ...`.

### 3.5 THE mutable default argument trap (#1 Python footgun)

Default argument values are evaluated **once**, at function-definition time, not per-call. A mutable default (list/dict/set) is therefore **shared and persists across every call** that doesn't pass its own.

```python
def append_bad(x, acc=[]):
    acc.append(x)
    return acc

print(append_bad(1))   # => [1]
print(append_bad(2))   # => [1, 2]   !!! — same list object reused, not fresh
print(append_bad(3))   # => [1, 2, 3]   still growing
```

This bites recursive helper functions that accumulate a result via a default list parameter:

```python
def collect_paths_bad(n, path=[], out=[]):   # BOTH defaults are shared traps
    if n == 0:
        out.append(list(path))   # must copy! path itself keeps mutating
        return
    path.append(n)
    collect_paths_bad(n - 1, path, out)
    path.pop()

# FIX: use None as sentinel default, create fresh mutable inside the function
def append_good(x, acc=None):
    if acc is None:
        acc = []
    acc.append(x)
    return acc

print(append_good(1))   # => [1]
print(append_good(2))   # => [2]   — fresh list every call, as expected
```

> **Gotcha:** This is the #1 Python footgun and shows up constantly in recursive DSA helpers (`def dfs(node, path=[], result=[])`). ALWAYS use `None` as the default for a mutable parameter and initialize inside the function body: `if acc is None: acc = []`.

### 3.6 Variable unpacking scope and augmented assignment

```python
a, b = 1, 2
a, b = b, a          # swap, no temp needed (see Part VII, tuple unpacking)
print(a, b)           # => 2 1

x = [1, 2, 3]
y = x
y += [4]              # += on a list MUTATES in place (list.__iadd__)
print(x)               # => [1, 2, 3, 4]   x changed too! y and x are the same object

s = "ab"
t = s
t += "c"               # += on a str creates a NEW string (str is immutable)
print(s)                 # => "ab"   s unchanged
```

> **vs C++:** `+=` behavior differs by whether the type is mutable. For immutable types (`int`, `str`, `tuple`) `+=` rebinds the name to a new object — safe, no aliasing surprise. For mutable types (`list`, `dict`, `set`) `+=` calls the in-place dunder (`__iadd__`) and mutates the shared object — if another variable aliases the same list, it sees the change too. This is closer to C++ reference semantics than most Python behavior, so it stands out.

---

## 4. Functions

### 4.1 `def`, positional/keyword args, defaults

```python
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

print(greet("Bob"))                  # => Hello, Bob!
print(greet("Bob", "Hi"))             # => Hi, Bob!
print(greet(name="Bob", greeting="Yo"))  # => Yo, Bob!
print(greet(greeting="Hey", name="Ann"))  # keyword args can be reordered => Hey, Ann!
```

### 4.2 `*args` / `**kwargs`, keyword-only, positional-only

```python
def f(a, b, *args, **kwargs):
    print(a, b, args, kwargs)

f(1, 2, 3, 4, x=5, y=6)   # => 1 2 (3, 4) {'x': 5, 'y': 6}

def g(a, b, *, c):    # `*` forces c to be KEYWORD-ONLY
    return a + b + c

print(g(1, 2, c=3))   # => 6
# g(1, 2, 3) would raise TypeError — c must be passed by keyword

def h(a, b, /, c):    # [3.8] `/` forces a, b to be POSITIONAL-ONLY
    return a + b + c

print(h(1, 2, c=3))       # => 6
print(h(1, 2, 3))          # => 6
# h(a=1, b=2, c=3) would raise TypeError — a, b can't be passed by keyword
```

> **[3.8]** positional-only `/` syntax — Codeforces/LeetCode judges (3.10+/3.11) fully support it; rare in DSA code but shows up in library signatures.

### 4.3 Argument unpacking at the call site

```python
def add3(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add3(*nums))         # => 6   list unpacked as positional args

kw = {'a': 1, 'b': 2, 'c': 3}
print(add3(**kw))           # => 6   dict unpacked as keyword args

print(max(*[3, 1, 4, 1, 5]))  # => 5  common trick: unpack into variadic builtins
```

### 4.4 Lambda — expression-only anonymous functions

```python
square = lambda x: x * x
print(square(5))   # => 25

pairs = [(1, 'b'), (3, 'a'), (2, 'c')]
pairs.sort(key=lambda p: p[1])
print(pairs)   # => [(3, 'a'), (1, 'b'), (2, 'c')]

# multiple sort keys
words = ['bb', 'a', 'ccc', 'dd']
words.sort(key=lambda w: (len(w), w))
print(words)   # => ['a', 'bb', 'dd', 'ccc']
```

> **Gotcha:** `lambda` bodies must be a single EXPRESSION — no statements (`=` assignment, `if`/`for` as statements, `return`, multi-line blocks). For anything beyond a one-liner, use a real `def`. The loop late-binding trap from §3.4 applies to lambdas exactly the same way.

### 4.5 Functions are first-class

```python
def apply_twice(f, x):
    return f(f(x))

print(apply_twice(lambda x: x * 2, 3))   # => 12

ops = {'+': lambda a, b: a + b, '-': lambda a, b: a - b}
print(ops['+'](3, 4))   # => 7
```

### 4.6 Decorators, and the DSA-critical `@lru_cache` / `@cache`

A decorator wraps a function with extra behavior: `@dec` above `def f` is sugar for `f = dec(f)`.

```python
def trace(fn):
    def wrapper(*args, **kwargs):
        print(f"calling {fn.__name__}{args}")
        return fn(*args, **kwargs)
    return wrapper

@trace
def add(a, b):
    return a + b

add(2, 3)   # => prints "calling add(2, 3)", returns 5
```

**THE Python DP superpower** — `functools.lru_cache`/`functools.cache` turns any recursive function into top-down memoized DP with a single line, no manual memo dict:

```python
from functools import lru_cache, cache

@lru_cache(maxsize=None)     # unbounded cache
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(50))   # => 12586269025   instant, thanks to memoization

@cache   # [3.9]+ — shorthand for lru_cache(maxsize=None), preferred when available
def climb_stairs(n):
    if n <= 2:
        return n
    return climb_stairs(n - 1) + climb_stairs(n - 2)

print(climb_stairs(30))   # => 832040
```

> **[3.9]** `@functools.cache` is sugar for `@lru_cache(maxsize=None)`, added in 3.9 — both work on LeetCode (3.11) and modern Codeforces. Use `@cache` when available; fall back to `@lru_cache(maxsize=None)` on older judges.

> **Gotcha:** Cached function arguments must be **hashable** — no `list`/`dict`/`set` arguments. If your recursive state includes a list (e.g. a partial path, a mutable visited array), convert it to a `tuple` (or `frozenset`) before calling, or restructure state as indices/bitmask instead.

```python
@cache
def dp(i, mask):     # mask: int bitmask instead of a mutable visited set — hashable
    ...
```

> Also remember `lru_cache` state persists across calls in the SAME process — in competitive programming with multiple test cases in one run, either design the cache to be test-case-independent, or call `fib.cache_clear()` between test cases if state must reset.

`functools.reduce` (brief):

```python
from functools import reduce
print(reduce(lambda acc, x: acc + x, [1, 2, 3, 4], 0))   # => 10  (like a fold/accumulate)
print(reduce(lambda a, b: a * b, [1, 2, 3, 4]))            # => 24  (no init: uses first elem)
```
Usually `sum()`/`math.prod()` are clearer for arithmetic; reach for `reduce` only for custom combining logic.

### 4.7 Returning multiple values

```python
def minmax(xs):
    return min(xs), max(xs)   # actually returns a tuple

lo, hi = minmax([3, 1, 4, 1, 5])
print(lo, hi)   # => 1 5

result = minmax([3, 1, 4])
print(result)    # => (1, 4)   a plain tuple if not unpacked
```

### 4.8 Recursion and the default recursion limit

```python
import sys
print(sys.getrecursionlimit())   # => 1000 (default)
```
Deep recursion (deep trees, DFS on long chains, recursive DP without memo) can hit `RecursionError` well before 10^5 calls. See **Part VII** for `sys.setrecursionlimit(...)` and when to convert recursion to an explicit stack instead.

### 4.9 No function overloading — last `def` wins

```python
def f(x):
    return x + 1

def f(x, y):     # silently REPLACES the first f — no overload resolution
    return x + y

# f(5) would now raise TypeError: missing argument y
print(f(5, 6))   # => 11
```

> **vs C++/Java:** Unlike C++/Java where multiple `def`s with different signatures coexist as overloads, Python has ONE name per scope — the second `def f` completely overwrites the first. Emulate overloading with default arguments, `*args`, `isinstance` checks inside one function, or (rarely needed in DSA) `functools.singledispatch`.

### 4.10 Nested functions for helpers

```python
def solve(nums):
    def valid(i, j):
        return 0 <= i < len(nums) and 0 <= j < len(nums)
    total = 0
    for i in range(len(nums)):
        for j in range(len(nums)):
            if valid(i, j):
                total += nums[i]
    return total

print(solve([1, 2, 3]))   # => 18
```
Idiomatic in DSA for a `dfs`/`bfs`/`check` helper scoped to one problem's `solve`/`class Solution` method — avoids polluting outer namespace and gets closure access to shared state (grid, memo, n) for free.

---

## 5. The Data Model — Classes and Dunder Methods

### 5.1 Basics: `__init__`, `self`, attributes, methods

```python
class Point:
    def __init__(self, x, y):
        self.x = x            # instance attribute
        self.y = y

    def dist_from_origin(self):
        return (self.x ** 2 + self.y ** 2) ** 0.5

p = Point(3, 4)
print(p.dist_from_origin())   # => 5.0
```

### 5.2 `@staticmethod`, `@classmethod`, `@property`

```python
class Rect:
    count = 0                 # class attribute, shared across instances

    def __init__(self, w, h):
        self.w, self.h = w, h
        Rect.count += 1

    @property
    def area(self):            # accessed like an attribute, no ()
        return self.w * self.h

    @staticmethod
    def unit():                 # no self/cls — just namespaced under Rect
        return Rect(1, 1)

    @classmethod
    def square(cls, side):       # receives the class, common for alt constructors
        return cls(side, side)

r = Rect(3, 4)
print(r.area)          # => 12   (no parens — it's a property)
print(Rect.count)       # => 1
sq = Rect.square(5)
print(sq.area)           # => 25
```

### 5.3 Dunder methods that matter for DSA

**`__lt__` — makes objects sortable AND usable in `heapq`.** `heapq` and `sorted`/`.sort()` only need `<` (`__lt__`); they never call `__eq__`/`__gt__` internally.

```python
import heapq

class Task:
    def __init__(self, priority, name):
        self.priority = priority
        self.name = name
    def __lt__(self, other):
        return self.priority < other.priority
    def __repr__(self):
        return f"Task({self.priority}, {self.name!r})"

heap = []
heapq.heappush(heap, Task(3, "low"))
heapq.heappush(heap, Task(1, "high"))
heapq.heappush(heap, Task(2, "mid"))
print(heapq.heappop(heap))   # => Task(1, 'high')
```

**`__eq__` + `__hash__` — to use objects as dict keys / set members.**

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __eq__(self, other):
        return isinstance(other, Point) and self.x == other.x and self.y == other.y
    def __hash__(self):
        return hash((self.x, self.y))
    def __repr__(self):
        return f"Point({self.x}, {self.y})"

s = {Point(1, 2), Point(1, 2), Point(3, 4)}
print(len(s))   # => 2   duplicates collapse thanks to __eq__/__hash__
```

> **Gotcha:** Defining `__eq__` **without** `__hash__` makes instances **unhashable** — Python sets `__hash__ = None` automatically whenever a class defines `__eq__` but not `__hash__`, because the default identity-based hash would now be inconsistent with the new equality. That object then can't go in a `set` or be a `dict` key, raising `TypeError: unhashable type`. Always define both together (or neither) when you need dict/set membership.

**`__repr__` / `__str__` — debug printing.**

```python
class Node:
    def __init__(self, val):
        self.val = val
    def __repr__(self):
        return f"Node({self.val})"

print(Node(5))          # => Node(5)   (uses __repr__ when __str__ absent)
print([Node(1), Node(2)])  # => [Node(1), Node(2)]  — containers always use __repr__ for elements
```
`__repr__` is used by `print()` on containers of objects, debuggers, and REPL echo — always define it for custom classes used in DSA (linked-list nodes, tree nodes, graph edges) so debugging is readable instead of `<__main__.Node object at 0x...>`.

**`__len__`, `__getitem__`/`__setitem__`, `__contains__` — make an object act like a built-in container.**

```python
class Deque2D:
    def __init__(self, rows, cols):
        self.data = [[0] * cols for _ in range(rows)]
    def __getitem__(self, idx):
        return self.data[idx]
    def __setitem__(self, idx, val):
        self.data[idx] = val
    def __len__(self):
        return len(self.data)
    def __contains__(self, val):
        return any(val in row for row in self.data)

g = Deque2D(2, 2)
g[0][1] = 5
print(g[0])          # => [0, 5]
print(len(g))          # => 2
print(5 in g)            # => True
```

**`__iter__`/`__next__` — make an object iterable (usable in `for`).**

```python
class Countdown:
    def __init__(self, n):
        self.n = n
    def __iter__(self):
        return self
    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1

for x in Countdown(3):
    print(x, end=' ')   # => 3 2 1
```

**`__call__` — make an instance callable like a function.**

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor
    def __call__(self, x):
        return x * self.factor

double = Multiplier(2)
print(double(21))   # => 42
```

### 5.4 Operator overloading via dunders

```python
class Vec2:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __add__(self, other):
        return Vec2(self.x + other.x, self.y + other.y)
    def __repr__(self):
        return f"Vec2({self.x}, {self.y})"

print(Vec2(1, 2) + Vec2(3, 4))   # => Vec2(4, 6)
```

Common dunders: `__add__` (`+`), `__sub__` (`-`), `__mul__` (`*`), `__eq__` (`==`), `__lt__`/`__le__`/`__gt__`/`__ge__` (comparisons), `__len__` (`len()`), `__bool__` (truthiness).

> **vs Java/JS:** Java and JS have NO operator overloading at all — `+` on custom objects either doesn't compile (Java) or falls back to string concatenation / `NaN` (JS). C++ has operator overloading via `operator+` etc., closer to Python's model. Python's dunder-method approach means every operator (`+`, `<`, `==`, `[]`, `()`, `in`, `len()`) is customizable and is exactly how built-in types (`list`, `dict`, `str`) implement themselves too — there's no special "built-in magic" separate from what you can also write.

### 5.5 `@dataclass` — concise data-holding classes **[3.7]**

Auto-generates `__init__`, `__repr__`, `__eq__` from type-annotated fields — eliminates boilerplate for CP "struct"-style classes.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p1 = Point(1, 2)
p2 = Point(1, 2)
print(p1)            # => Point(x=1, y=2)     auto __repr__
print(p1 == p2)        # => True                auto __eq__ (value-based!)
```

`@dataclass(order=True)` auto-generates `__lt__`/`__le__`/`__gt__`/`__ge__` by comparing fields **in declaration order** — perfect for heap elements without hand-writing `__lt__`:

```python
from dataclasses import dataclass, field

@dataclass(order=True)
class Task:
    priority: int
    name: str = field(compare=False)   # exclude `name` from comparisons

import heapq
heap = []
heapq.heappush(heap, Task(3, "low"))
heapq.heappush(heap, Task(1, "high"))
heapq.heappush(heap, Task(2, "mid"))
print(heapq.heappop(heap))   # => Task(priority=1, name='high')
```

> **[3.7]** `@dataclass` requires 3.7+; both `order=True` comparisons and `field()` are available. Safe on both LeetCode (3.11) and Codeforces (3.10+).

### 5.6 `__slots__` — memory/speed trick for CP

By default every instance carries a `__dict__` for arbitrary attribute storage. `__slots__` declares a fixed attribute set, saving memory and speeding up attribute access — matters when creating millions of small objects (e.g. graph nodes in a huge BFS).

```python
class FastPoint:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x, self.y = x, y

p = FastPoint(1, 2)
# p.z = 3   # would raise AttributeError — no __dict__, no arbitrary new attrs
```

### 5.7 Inheritance and `super()` (brief)

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        return f"{self.name} makes a sound"

class Dog(Animal):
    def speak(self):
        base = super().speak()
        return f"{base}, specifically barks"

print(Dog("Rex").speak())   # => Rex makes a sound, specifically barks
```
Rare in raw DSA solving beyond custom exceptions or a small class hierarchy (e.g. `TreeNode` subclasses); more relevant for larger tooling.

---

## 6. Comprehensions, Generators and Iterators

### 6.1 List comprehension — the default loop

```python
xs = [1, 2, 3, 4, 5]
doubled = [x * 2 for x in xs]
print(doubled)   # => [2, 4, 6, 8, 10]

evens = [x for x in xs if x % 2 == 0]
print(evens)      # => [2, 4]

pairs = [(i, j) for i in range(2) for j in range(2)]
print(pairs)        # => [(0, 0), (0, 1), (1, 0), (1, 1)]

flat = [x for row in [[1,2],[3,4]] for x in row]
print(flat)           # => [1, 2, 3, 4]
```
Prefer comprehensions over manual `for` + `.append()` loops — faster (bytecode-optimized) and more idiomatic; use everywhere a transformed/filtered list is being built.

### 6.2 2D initialization — the correct way, and the shared-row bug

```python
n, m = 3, 4
grid = [[0] * m for _ in range(n)]   # CORRECT — n independent row objects
grid[0][0] = 1
print(grid)   # => [[1, 0, 0, 0], [0, 0, 0, 0], [0, 0, 0, 0]]

bad_grid = [[0] * m] * n     # BUG — n references to the SAME row object!
bad_grid[0][0] = 1
print(bad_grid)   # => [[1, 0, 0, 0], [1, 0, 0, 0], [1, 0, 0, 0]]   all rows changed!
```

> **Gotcha (critical):** `[[0]*m] * n` uses Python's list `*` repetition, which creates `n` references to the SAME inner list object — mutating one row mutates all of them. This is one of the most common DP/grid-initialization bugs in Python. ALWAYS use `[[0] * m for _ in range(n)]` (a comprehension, which calls `[0]*m` fresh `n` times) for 2D array init. The same trap applies to `[[] for _ in range(n)]` vs `[[]] * n` for adjacency lists.

### 6.3 Dict and set comprehensions

```python
squares = {x: x * x for x in range(5)}
print(squares)   # => {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

freq = {}
word = "banana"
freq = {c: word.count(c) for c in set(word)}
print(freq)   # => {'b': 1, 'a': 3, 'n': 2}   (order may vary by set iteration)

uniq_lens = {len(w) for w in ["a", "bb", "cc", "ddd"]}
print(uniq_lens)   # => {1, 2, 3}
```

### 6.4 Generator expressions — lazy, memory-efficient

```python
gen = (x * x for x in range(10**8))   # builds NOTHING yet — O(1) memory
print(sum(x for x in range(1, 6)))      # => 15  — no intermediate list built
print(any(x > 100 for x in range(50)))    # => False, short-circuits on first True
print(max((len(w) for w in ["a", "bbb", "cc"])))  # => 3
```

> Use a generator expression `(...)` instead of a list comprehension `[...]` whenever the result is immediately consumed once by `sum`/`any`/`all`/`max`/`min`/`sorted`/a `for` loop and doesn't need to be stored — saves building a full list in memory. Drop the parens when it's the sole argument to a call: `sum(x for x in xs)` (no double parens needed).

### 6.5 `yield` and generator functions

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

for x in countdown(3):
    print(x, end=' ')   # => 3 2 1

def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)   # delegates to sub-generator
        else:
            yield item

print(list(flatten([1, [2, 3, [4, 5]], 6])))   # => [1, 2, 3, 4, 5, 6]
```
A generator function pauses at each `yield` and resumes on the next `next()` call — useful for lazily producing infinite/large sequences (e.g. streaming candidate values in a search) without materializing them all.

### 6.6 The iterator protocol: `iter()`, `next()`, `StopIteration`

```python
xs = [10, 20, 30]
it = iter(xs)
print(next(it))   # => 10
print(next(it))    # => 20
print(next(it))     # => 30
print(next(it, 'done'))  # => 'done'   (default avoids StopIteration)
# next(it) again (no default) would raise StopIteration
```
`for x in xs:` is sugar for repeatedly calling `iter(xs)` then `next()` until `StopIteration`.

### 6.7 `enumerate`, `zip` (and `zip(*matrix)` transpose), `reversed`, `sorted`

```python
xs = ['a', 'b', 'c']
for i, v in enumerate(xs):
    print(i, v)   # => 0 a / 1 b / 2 c
for i, v in enumerate(xs, start=1):   # custom start index
    print(i, v)     # => 1 a / 2 b / 3 c

a, b = [1, 2, 3], ['x', 'y', 'z']
print(list(zip(a, b)))   # => [(1, 'x'), (2, 'y'), (3, 'z')]   stops at shorter

matrix = [[1, 2, 3], [4, 5, 6]]
transposed = list(zip(*matrix))
print(transposed)   # => [(1, 4), (2, 5), (3, 6)]   slick transpose idiom

print(list(reversed([1, 2, 3])))   # => [3, 2, 1]
print(sorted([3, 1, 2]))              # => [1, 2, 3]   (new list, original untouched)
```

> **vs C++/Java:** No manual `for (int i = 0; i < xs.size(); i++)` index bookkeeping needed for "index + value together" (`enumerate`) or "walk two arrays in lockstep" (`zip`) — both are direct, allocation-light idioms baked into the language.

### 6.8 `range` — lazy, with step

```python
print(list(range(5)))          # => [0, 1, 2, 3, 4]
print(list(range(2, 10, 3)))    # => [2, 5, 8]
print(list(range(10, 0, -2)))    # => [10, 8, 6, 4, 2]
print(len(range(0, 100)))          # => 100   O(1), doesn't materialize
```
`range` is a lazy, O(1)-memory sequence object (not a list) — safe to write `range(10**18)` without it trying to allocate anything.

### 6.9 `map`/`filter` — iterators; comprehensions are often clearer

```python
xs = [1, 2, 3, 4]
print(list(map(lambda x: x * x, xs)))     # => [1, 4, 9, 16]
print(list(filter(lambda x: x % 2 == 0, xs)))  # => [2, 4]

# common CP idiom: parse a line of space-separated ints
nums = list(map(int, input().split()))   # forward-ref Part VII for input()
```
`map`/`filter` return lazy iterators (must wrap in `list(...)` to materialize); a comprehension `[x*x for x in xs]` is usually equally fast and more readable — prefer it except for the very common `map(int, ...)` input-parsing idiom.

### 6.10 Walrus operator `:=` **[3.8]**

Assignment as an expression — assigns AND returns the value in one go. Useful inside comprehensions (avoid recomputation) and `while` loop conditions.

```python
data = [1, 2, 3, 4, 5]
# avoid computing y = f(x) twice
results = [y for x in data if (y := x * x) > 5]
print(results)   # => [9, 16, 25]

# classic use: read-until-empty loop
# while (line := input()):
#     process(line)

n = 10
squares_under_50 = []
i = 0
while (sq := i * i) < 50:
    squares_under_50.append(sq)
    i += 1
print(squares_under_50)   # => [0, 1, 4, 9, 16, 25, 36, 49]
```

> **[3.8]** `:=` requires 3.8+ — safe on both LeetCode and modern Codeforces judges.

---

## 7. Control Flow and Pythonic Idioms

### 7.1 `if` / `elif` / `else`

No parentheses, colon + indentation define the block. No `switch` before 3.10 (use `if/elif` or a dict dispatch; see `match` below).

```python
x = 5
if x < 0:
    sign = -1
elif x == 0:
    sign = 0
else:
    sign = 1
print(sign)   # => 1

# dict dispatch replaces switch for value->action
op = {'+': lambda a, b: a + b, '-': lambda a, b: a - b}
print(op['+'](2, 3))   # => 5
```

### 7.2 `for` over iterables (not C-style indices)

Iterate the elements directly; reach for `range`/`enumerate` only when you truly need the index.

```python
xs = [10, 20, 30]
for v in xs:            # value directly — the default
    pass
for i, v in enumerate(xs):   # index + value when you need both
    pass
for i in range(len(xs)):     # only when you need the index alone
    pass
```

> **vs C++:** the default `for` is a range-for over values, like C++ `for (auto x : v)`. A raw index loop is the exception in Python, not the norm.

### 7.3 `for-else` and `while-else` (else runs if NO `break`)

The `else` clause on a loop runs only if the loop finished without hitting `break`. Genuinely useful for search loops — no separate "found" flag needed.

```python
def has_prime_factor_upto(n, limit):
    for d in range(2, limit + 1):
        if n % d == 0:
            print(f"divisible by {d}")
            break
    else:
        # runs ONLY if the loop never broke
        print("no small factor found")

has_prime_factor_upto(17, 4)   # => "no small factor found"
has_prime_factor_upto(15, 4)   # => "divisible by 3"
```

> **Gotcha:** the `else` binds to the LOOP, not to any `if` inside it. Reads oddly at first; think of it as "no-break".

### 7.4 `break` / `continue`

Standard. Python has no labeled break (unlike Java/JS) — to break out of nested loops, either `break` out of a flag-checked inner loop, factor the loops into a function and `return`, or use a sentinel.

```python
# breaking nested loops via a helper + return (the clean Python way)
def find_pair(grid, target):
    for r, row in enumerate(grid):
        for c, val in enumerate(row):
            if val == target:
                return (r, c)     # returns straight out of both loops
    return None

print(find_pair([[1, 2], [3, 4]], 3))   # => (1, 0)
```

> **vs C++/Java/JS:** no `goto` and no `break outer:` label. A function + `return` is the idiomatic replacement for breaking out of nested loops.

### 7.5 Tuple unpacking everywhere

The most Pythonic idiom — parallel assignment, swap without a temp, splitting head/tail.

```python
a, b = 1, 2
a, b = b, a               # SWAP without temp — evaluates RHS tuple first
print(a, b)               # => 2 1

first, *rest = [1, 2, 3, 4]
print(first, rest)        # => 1 [2, 3, 4]

first, *mid, last = [1, 2, 3, 4, 5]
print(first, mid, last)   # => 1 [2, 3, 4] 5

# nested unpacking (e.g. iterating edges with weights)
for u, (v, w) in [(0, (1, 5)), (1, (2, 3))]:
    pass

# unpacking in a for loop over enumerate/zip
for i, (x, y) in enumerate(zip([1, 2], [3, 4])):
    pass
```

> **Gotcha:** the starred target (`*rest`) always collects a **list**, even when empty (`a, *rest = [1]` gives `rest == []`).

### 7.6 EAFP vs LBYL

Python culture prefers **EAFP** ("Easier to Ask Forgiveness than Permission") — try the operation, catch the exception — over **LBYL** ("Look Before You Leap") pre-checks. In DSA it is often a wash; use whichever is clearer, but know that `dict.get`/`defaultdict` usually beat a `try/except KeyError`.

```python
# LBYL
if key in d:
    v = d[key]
else:
    v = 0

# EAFP
try:
    v = d[key]
except KeyError:
    v = 0

# Pythonic (best for this case): just use .get
v = d.get(key, 0)
```

### 7.7 `match` / `case` — structural pattern matching **[3.10]**

A real structural match (not just a C `switch`) — matches values, sequences, and can bind variables. Handy but rarely essential in CP; plain `if/elif` is fine.

```python
def describe(point):
    match point:
        case (0, 0):
            return "origin"
        case (0, y):
            return f"on Y axis at {y}"
        case (x, 0):
            return f"on X axis at {x}"
        case (x, y):
            return f"point ({x}, {y})"
        case _:
            return "not a 2D point"

print(describe((0, 5)))   # => "on Y axis at 5"
print(describe((3, 4)))   # => "point (3, 4)"
```

> **[3.10]** `match` requires Python 3.10+ — available on LeetCode (3.11) and modern Codeforces; avoid it if a judge pins an older Python.

### 7.8 Ternary and container-`*` idioms

```python
mx = a if a > b else b            # ternary: <val> if <cond> else <val>

row = [0] * n                     # fast fill of an immutable
line = '=' * 40                   # repeat a string
grid = [[0] * m for _ in range(n)]  # CORRECT 2D init (see §6 for the [[0]*m]*n bug)
```

### 7.9 `in` membership — and its complexity trap

```python
x in some_list    # O(n) — scans the list
x in some_set     # O(1) average — hash lookup
x in some_dict    # O(1) average — checks keys
sub in some_str   # O(n*m) substring search
```

> **Gotcha (TLE):** `x in list` inside a loop is a classic O(n^2) blowup. If you repeatedly test membership, build a `set` first. This is one of the most common Python-DSA performance bugs.

### 7.10 One-liner reducers over comprehensions

`sum`/`any`/`all`/`min`/`max` consume a generator expression directly — no intermediate list, clean and fast (C-level).

```python
nums = [3, 1, 4, 1, 5, 9]
print(sum(x for x in nums if x % 2))       # => 18 (sum of odds)
print(any(x > 8 for x in nums))            # => True
print(all(x > 0 for x in nums))            # => True
print(max(nums))                           # => 9
print(min(nums, default=0))                # => 1 (default avoids ValueError on empty)
print(max(range(len(nums)), key=lambda i: nums[i]))  # => 5 (argmax index)
```

> **Gotcha:** `max([])`/`min([])` raise `ValueError` on an empty iterable — pass `default=` to be safe.

---

# PART II — SEQUENCES, STRINGS AND SLICING

## 8. list — The Workhorse

`list` is Python's dynamic array — like C++ `std::vector`, Java `ArrayList`, or JS `Array`. Unlike C++/Java, a single `list` can hold **mixed types** (it stores references/pointers, not inline values).

### Creation

```python
a = []                      # empty list
b = [1, 2, 3]                # literal
c = [0] * 5                  # => [0, 0, 0, 0, 0]  fast fill (O(n), but see 2D trap below)
d = list(range(5))           # => [0, 1, 2, 3, 4]
e = list("abc")              # => ['a', 'b', 'c']  iterable -> list
f = [x * x for x in range(5)]  # => [0, 1, 4, 9, 16]  comprehension
print(a, b, c, d, e, f)
```

> **vs C++:** no fixed size/type at creation. `[0]*n` is the idiomatic `vector<int> v(n, 0)` equivalent — but see the 2D-list aliasing bug below, `[0]*n` for a flat list is safe (ints are immutable), the danger is only with `[[...]]*n`.

### Complete Method Table

| Method | What it does | Mutates? | Complexity |
|---|---|---|---|
| `a.append(x)` | add `x` to end | yes | O(1) amortized |
| `a.pop()` | remove & return last | yes | O(1) |
| `a.pop(i)` | remove & return index `i` | yes | O(n) (shifts) |
| `a.insert(i, x)` | insert `x` before index `i` | yes | O(n) (shifts) |
| `a.remove(x)` | remove **first** occurrence of `x` (ValueError if absent) | yes | O(n) |
| `a.index(x[, start[, end]])` | index of first occurrence (ValueError if absent) | no | O(n) |
| `a.count(x)` | number of occurrences of `x` | no | O(n) |
| `a.extend(iterable)` / `a += iterable` | append all items from iterable | yes | O(k) |
| `a.sort(key=None, reverse=False)` | sort in place | yes | O(n log n) |
| `a.reverse()` | reverse in place | yes | O(n) |
| `a.clear()` | remove all elements | yes | O(n) |
| `a.copy()` | shallow copy | no | O(n) |
| `x in a` | membership test | no | O(n) |
| `len(a)` | number of elements | no | O(1) |
| `a[i]` | get/set element | get: no, set: yes | O(1) |
| `min(a)` / `max(a)` | smallest / largest | no | O(n) |
| `sum(a)` | sum of elements | no | O(n) |

```python
a = [3, 1, 2]
a.append(4)        # a => [3, 1, 2, 4]
last = a.pop()      # last => 4, a => [3, 1, 2]
a.insert(1, 9)      # a => [3, 9, 1, 2]
a.remove(9)         # a => [3, 1, 2]   removes FIRST match, not by index
print(a.index(1))   # => 1
print(a.count(1))   # => 1
```

> **Gotcha:** `list.remove(x)` takes a *value*, not an index (unlike `pop(i)`). Calling `remove` on a missing value raises `ValueError` — wrap in `if x in a:` or `try/except` if uncertain.

### Indexing — including NEGATIVE indices

```python
a = [10, 20, 30, 40, 50]
print(a[0])    # => 10
print(a[-1])   # => 50   last element
print(a[-2])   # => 40   second-to-last
```

> **vs C++/Java/JS:** none of those support negative indexing natively (C++ `v[-1]` is UB / out-of-bounds; JS `a[-1]` is `undefined`). Python's negative index is a first-class feature — huge for "last k" patterns without computing `len(a)-1`.

### Slicing in depth

`a[start:stop:step]` — `start` inclusive, `stop` exclusive, `step` optional (default 1). Out-of-range indices are clamped silently (no IndexError, unlike single-index access).

```python
a = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
print(a[2:5])     # => [2, 3, 4]
print(a[:3])      # => [0, 1, 2]        omit start => from beginning
print(a[7:])      # => [7, 8, 9]        omit stop => to end
print(a[::2])     # => [0, 2, 4, 6, 8]  every 2nd element
print(a[::-1])    # => [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]   REVERSE (the idiom)
print(a[5:1:-1])  # => [5, 4, 3, 2]     negative step, start>stop
print(a[100:200]) # => []               out-of-range -> empty, no error
```

> **Gotcha:** a slice always creates a **new list** (shallow copy) — `a[:]` is O(n), not O(1). For read-only iteration over a sub-range without copying, prefer `itertools.islice` (Part III) or just index arithmetic in a loop.

**Slice assignment** — can change length, unlike C++ array assignment:

```python
a = [1, 2, 3, 4, 5]
a[1:3] = [8, 9, 10]     # replaces 2 elements with 3
print(a)                 # => [1, 8, 9, 10, 4, 5]

a = [1, 2, 3, 4, 5]
a[:] = [7, 7, 7]          # in-place replace of ALL contents
print(a)                  # => [7, 7, 7]   same list object, contents replaced

a = [1, 2, 3]
del a[1:]                 # slice deletion
print(a)                  # => [1]
```

> **Gotcha:** `a = [...]` REBINDS the name `a` to a new list object. `a[:] = [...]` mutates the existing object in place — matters if another variable/reference points to the same list (e.g. passed into a function).

### 2D lists — the shared-row bug

```python
n, m = 3, 3

# CORRECT — each row is an independent list
grid = [[0] * m for _ in range(n)]
grid[0][0] = 1
print(grid)   # => [[1, 0, 0], [0, 0, 0], [0, 0, 0]]

# BUG — all rows are the SAME list object (reference repeated n times)
bad = [[0] * m] * n
bad[0][0] = 1
print(bad)    # => [[1, 0, 0], [1, 0, 0], [1, 0, 0]]   <-- every row changed!
```

> **Gotcha:** `[[0]*m]*n` repeats the *same inner list reference* `n` times. Mutating one row mutates all of them. This is one of the single most common Python DSA bugs (grid/DP init). ALWAYS use `[[0]*m for _ in range(n)]` for a mutable 2D grid.

```python
grid = [[1, 2, 3], [4, 5, 6]]
print(grid[1][2])          # => 6           row 1, col 2
print(list(zip(*grid)))    # => [(1, 4), (2, 5), (3, 6)]   transpose (tuples)
transposed = [list(row) for row in zip(*grid)]
print(transposed)          # => [[1, 4], [2, 5], [3, 6]]
```

> `zip(*grid)` unpacks each row as a separate argument to `zip`, pairing up elements column-wise — the standard Python matrix-transpose idiom.

### list as stack vs list as queue

```python
stack = []
stack.append(1); stack.append(2); stack.append(3)
print(stack.pop())   # => 3   LIFO, O(1) — list is a natural stack
```

> **Gotcha:** `list.pop(0)` (dequeue from front) is O(n) — it shifts every remaining element left. Using a `list` as a FIFO queue in a hot loop silently degrades to O(n^2). Use `collections.deque` instead (see Part III) for O(1) `popleft()`/`appendleft()`.

### Copying — shallow vs deep

```python
import copy

a = [1, 2, 3]
b = a.copy()      # shallow copy, same as a[:] or list(a)
b.append(4)
print(a, b)        # => [1, 2, 3] [1, 2, 3, 4]   independent top-level lists

# the nested-reference trap:
grid = [[1, 2], [3, 4]]
shallow = grid.copy()          # copies OUTER list only
shallow[0][0] = 99
print(grid)                    # => [[99, 2], [3, 4]]   inner lists still SHARED!

deep = copy.deepcopy(grid)
deep[0][0] = -1
print(grid)                    # => [[99, 2], [3, 4]]   unaffected — true independent copy
```

> **Gotcha:** `a.copy()`, `a[:]`, and `list(a)` are all *shallow* — for a list of lists (or list of dicts/objects), the inner containers are still shared references. Use `copy.deepcopy` for true independence, or rebuild manually (`[row[:] for row in grid]` is a cheap deep-enough copy for 2D of primitives).

### Unpacking

```python
a, b, *rest = [1, 2, 3, 4, 5]
print(a, b, rest)     # => 1 2 [3, 4, 5]

first, *middle, last = [1, 2, 3, 4, 5]
print(first, middle, last)   # => 1 [2, 3, 4] 5
```

> **vs C++/Java:** no direct equivalent to `*rest` star-unpacking — closest is manual slicing (`v[0]`, `v[1]`, `vector<int>(v.begin()+2, v.end())`).

### sorted() vs .sort()

```python
a = [3, 1, 2]
b = sorted(a)          # NEW list, a unchanged
print(a, b)             # => [3, 1, 2] [1, 2, 3]

a.sort()                # in-place, returns None
print(a)                 # => [1, 2, 3]

c = sorted(a, reverse=True)   # => [3, 2, 1]
print(c)
```

> **Gotcha:** `a.sort()` returns `None` — `a = a.sort()` silently sets `a` to `None`. Use `sorted(a)` when you need an expression / new list. Both accept `key=` and `reverse=`.

Both `list.sort()` and `sorted()` are **stable** — equal elements keep their relative input order. This matters for multi-pass sorts (sort by secondary key first, then by primary key — equal primaries retain their secondary order):

```python
data = [("b", 2), ("a", 2), ("a", 1)]
data.sort(key=lambda t: t[0])              # stable sort by name only
print(data)   # => [('a', 2), ('a', 1), ('b', 2)]   'a' items keep original relative order
```

> **vs C++:** `std::sort` is NOT guaranteed stable (use `std::stable_sort` explicitly for that guarantee). Python's `sort`/`sorted` are ALWAYS stable — one less thing to worry about, and it's what makes the "sort by tuple key" pattern (Section 9) reliable for tie-breaking.

### Comprehension as idiomatic transform/filter

```python
nums = [1, 2, 3, 4, 5, 6]
squares = [x * x for x in nums]              # => [1, 4, 9, 16, 25, 36]
evens = [x for x in nums if x % 2 == 0]       # => [2, 4, 6]
pairs = [(x, y) for x in range(2) for y in range(2)]  # => [(0,0),(0,1),(1,0),(1,1)]
print(squares, evens, pairs)
```

### Membership and enumerate/zip

```python
a = [10, 20, 30]
print(20 in a)          # => True   O(n) linear scan — use a set for O(1) repeated lookups

for i, v in enumerate(a):
    print(i, v)          # => 0 10 / 1 20 / 2 30

b = ["a", "b", "c"]
for v, w in zip(a, b):
    print(v, w)           # => 10 a / 20 b / 30 c
```

> **Gotcha:** `x in list` is O(n) per check. If you're checking membership repeatedly in a loop (e.g. "have I seen this number"), convert to `set(a)` first — turns O(n*k) into O(n+k).

### del vs remove vs pop — the three ways to delete

```python
a = [10, 20, 30, 40]
del a[1]          # delete by INDEX, no return value
print(a)            # => [10, 30, 40]

a = [10, 20, 30, 40]
a.remove(30)        # delete by VALUE (first match), no return value
print(a)              # => [10, 20, 40]

a = [10, 20, 30, 40]
x = a.pop(1)          # delete by INDEX, RETURNS the removed value
print(x, a)             # => 20 [10, 30, 40]
```

> **Gotcha:** three different tools for "delete an element" — `del a[i]` (index, statement, no return), `a.remove(v)` (value, no return), `a.pop(i)` (index, returns value). Mixing them up is a common source of confusion when porting from C++ `erase`.

### list equality and repetition

```python
a = [1, 2, 3]
b = [1, 2, 3]
print(a == b)     # => True    element-wise value comparison (not identity)
print(a is b)      # => False   different objects

print([1, 2] + [3, 4])   # => [1, 2, 3, 4]   concatenation
print([0, 1] * 3)          # => [0, 1, 0, 1, 0, 1]   repetition (flat data only — see 2D trap)
```

> **vs Java/JS:** `==` on Python lists compares contents recursively (like Java's `List.equals`, unlike JS `===` which is reference-only for arrays). Use `is` only when you actually need identity comparison.

### Nested comprehension with condition (filter + transform + flatten)

```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat_even = [x for row in matrix for x in row if x % 2 == 0]
print(flat_even)     # => [2, 4, 6, 8]

# conditional expression (ternary) INSIDE a comprehension:
labels = ["even" if x % 2 == 0 else "odd" for x in range(5)]
print(labels)          # => ['even', 'odd', 'even', 'odd', 'even']
```

> **Gotcha:** the ternary `x if cond else y` sits BEFORE the `for`; the filter `if cond` (no `else`) sits AFTER the `for`. Mixing these two `if` forms up is a frequent syntax confusion.

### Two-pointer / sliding-window pattern (idiomatic use of list indexing)

```python
def two_sum_sorted(a, target):
    lo, hi = 0, len(a) - 1
    while lo < hi:
        s = a[lo] + a[hi]
        if s == target:
            return lo, hi
        elif s < target:
            lo += 1
        else:
            hi -= 1
    return -1, -1

print(two_sum_sorted([1, 3, 5, 7, 9], 12))   # => (2, 4)
```

> Standard two-pointer skeleton — plain index arithmetic on a `list`, no special syntax needed; shown here since it's the most common `list`-indexing pattern in DSA interview/CP problems.

---

## 9. tuple and Sequence Packing

`tuple` is an **immutable** sequence — and critically, **hashable** (as long as its elements are hashable), which means it can be used as a `dict` key or `set` member. This is the idiomatic way to represent a coordinate/pair in Python.

```python
seen = set()
seen.add((2, 3))
print((2, 3) in seen)   # => True

dp = {}
dp[(0, 0)] = 1
print(dp[(0, 0)])        # => 1
```

> **vs C++/JS:** C++ needs `std::pair<int,int>` (or a custom hash for `unordered_set<pair<int,int>>`, painful — `pair` has no default hash). JS has no tuple type at all — you're forced to stringify keys (`` `${r},${c}` ``) which is slower and more error-prone. Python's tuple-as-key is a genuine, load-bearing advantage for grid/graph problems — use it liberally.

### Creation

```python
t1 = (1, 2)
t2 = 1, 2              # parens optional — same as t1
t3 = (1,)               # single-element tuple — TRAILING COMMA REQUIRED
t4 = (1)                 # NOT a tuple! just int 1, parens are just grouping
t5 = tuple([1, 2, 3])    # from iterable
t6 = ()                  # empty tuple
print(t1, t2, t3, type(t4), t5, t6)
# => (1, 2) (1, 2) (1,) <class 'int'> (1, 2, 3) ()
```

> **Gotcha:** `(1)` is just the integer `1` in parentheses — parentheses alone don't make a tuple. You NEED the trailing comma: `(1,)`. This trips up everyone coming from C++/Java where a single-element container has an obvious syntax.

### Indexing/slicing like list, but no mutation

```python
t = (10, 20, 30, 40)
print(t[1])       # => 20
print(t[-1])      # => 40
print(t[1:3])     # => (20, 30)   slicing a tuple returns a tuple
# t[0] = 99        # TypeError: 'tuple' object does not support item assignment
```

### Unpacking and multiple return values

```python
def divmod_like(a, b):
    return a // b, a % b   # returns a tuple (a, b) implicitly

q, r = divmod_like(17, 5)
print(q, r)                 # => 3 2

point = (3, 4)
x, y = point
print(x, y)                  # => 3 4
```

> **vs C++/Java:** returning multiple values needs `std::pair`/`std::tuple` with `std::tie` in C++, or a wrapper class in Java. Python's implicit tuple-return + unpack is native and idiomatic — used constantly in DSA code (`return left, right`, `for i, v in enumerate(a)`).

### namedtuple / typing.NamedTuple — lightweight struct

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
print(p.x, p.y)      # => 3 4
print(p)              # => Point(x=3, y=4)
print(p[0])            # => 3   still index-accessible, still a tuple

from typing import NamedTuple

class Point2(NamedTuple):
    x: int
    y: int

p2 = Point2(1, 2)
print(p2.x, p2.y)     # => 1 2
```

> Use when field names improve readability (e.g. `Interval(start, end)`) over a plain `(start, end)` tuple. Both are still immutable and hashable, so they work as dict keys / heap elements just like plain tuples.

### Tuples compare LEXICOGRAPHICALLY — critical for DSA

```python
print((1, 2) < (1, 3))    # => True    compare element by element
print((1, 5) < (2, 0))    # => True    first element decides
print((1, 2) < (1, 2, 0)) # => True    shorter is "less" if it's a prefix

pairs = [(3, 'c'), (1, 'b'), (1, 'a')]
print(sorted(pairs))       # => [(1, 'a'), (1, 'b'), (3, 'c')]
```

> This is WHY `sorted(list_of_tuples)` naturally sorts by the first element, breaking ties on the second, then third, etc. — exactly like `operator<` on `std::pair`/`std::tuple` in C++. It's also why a **min-heap of tuples works as a priority queue with tie-breaking**:

```python
import heapq

h = []
heapq.heappush(h, (5, "task_e"))
heapq.heappush(h, (1, "task_a"))
heapq.heappush(h, (1, "task_b"))   # ties broken by 2nd element
heapq.heappop(h)   # => (1, 'task_a')   lowest priority first, then lexicographic tiebreak
```

> **Gotcha:** if the 2nd tuple element isn't comparable (e.g. a custom object without `__lt__`) and a tie occurs on the first element, `heapq` will try to compare the 2nd elements and raise `TypeError`. Fix by adding a unique tiebreaker (e.g. insertion counter) as the 2nd tuple field: `(priority, counter, item)`.

### Multi-key sort via tuple key

```python
class Person:
    def __init__(self, name, age):
        self.name, self.age = name, age

people = [Person("Bob", 25), Person("Amy", 30), Person("Amy", 25)]
people.sort(key=lambda p: (p.name, -p.age))   # name asc, age desc within same name
print([(p.name, p.age) for p in people])
# => [('Amy', 30), ('Amy', 25), ('Bob', 25)]
```

> `key=lambda x: (x.a, -x.b)` is the standard multi-key sort pattern — negate a numeric field to reverse its sort order independently of the others (no equivalent single-line trick for strings; use `reverse=` per-field only via multiple stable sorts if needed).

### tuple vs list — when to use which

| | `tuple` | `list` |
|---|---|---|
| Mutable | No | Yes |
| Hashable (dict key / set member) | Yes (if elements are) | No |
| Typical use | Fixed record, coordinate, dict key, function return, heap element | Growing/shrinking collection |
| Memory | Slightly smaller | Slightly larger (over-allocates for growth) |

### Packing/unpacking in loops

```python
pairs = [(1, "a"), (2, "b"), (3, "c")]
for i, (a, b) in enumerate(pairs):
    print(i, a, b)
# => 0 1 a / 1 2 b / 2 3 c

for a, b in pairs:              # direct unpack without enumerate
    print(a, b)
```

### tuple concatenation, repetition, and zip's return type

```python
print((1, 2) + (3, 4))     # => (1, 2, 3, 4)    concatenation makes a new tuple
print((0, 1) * 3)            # => (0, 1, 0, 1, 0, 1)

pairs = list(zip([1, 2, 3], ['a', 'b', 'c']))
print(pairs)                  # => [(1, 'a'), (2, 'b'), (3, 'c')]
print(type(pairs[0]))          # => <class 'tuple'>
```

> `zip()` always yields `tuple`s (even when zipping just two lists) — that's why `dict(zip(keys, values))` works directly: `dict()` accepts an iterable of 2-tuples.

```python
d = dict(zip(["a", "b", "c"], [1, 2, 3]))
print(d)   # => {'a': 1, 'b': 2, 'c': 3}
```

### Single-argument function call gotcha

```python
def f(*args):
    print(args)

f(1, 2, 3)     # => (1, 2, 3)     *args is collected as a tuple
f(1)             # => (1,)          even one arg becomes a 1-tuple
```

> `*args` inside a function signature always packs into a `tuple`, reinforcing that a tuple is Python's default "bag of positional values" — same object type used for multiple-return-unpack.

### Tuple as an immutable default / sentinel

```python
def solve(nums, memo_key=()):
    # a mutable default (memo_key=[]) would be a shared-across-calls bug (see Part IV)
    ...
```

> **Gotcha (forward-ref):** never use a mutable default argument (`def f(a=[])`) — the same list object is reused across calls. `()` (an empty tuple) is a safe immutable default when you need an empty "no keys yet" placeholder; covered in full in Part IV (functions/closures).

---

## 10. str — Immutable Strings

Python strings are **immutable** — every operation (`+`, slicing, `.replace()`, etc.) returns a brand-new string; nothing mutates the original.

> **vs C++:** `std::string` is mutable (`s[i] = 'x'` works, `s += c` in a loop is amortized O(1)). **vs Java/JS:** both are also immutable, same trap applies there too — this is NOT a Python-only surprise, but it bites C++ developers hardest.

```python
s = "hello"
# s[0] = 'H'          # TypeError: 'str' object does not support item assignment
s2 = s.replace("h", "H")   # new string
print(s, s2)                # => hello Hello
```

### The O(n^2) trap and the join idiom

```python
# BAD — O(n^2): each += creates a new string, copying everything so far
result = ""
for c in "hello world":
    if c != " ":
        result += c            # re-copies entire result each time
print(result)   # => helloworld

# GOOD — O(n): build a list, join once at the end
parts = []
for c in "hello world":
    if c != " ":
        parts.append(c)
result = "".join(parts)         # single O(n) join
print(result)   # => helloworld
```

> **Gotcha:** `s += c` in a loop is the single most common Python performance bug in string-heavy DSA code — CPython has a minor optimization for some patterns, but never rely on it. THE idiom: accumulate into a `list`, `''.join(...)` once at the end.

### No char type — ord/chr for char arithmetic

Indexing a string gives back a length-1 `str`, not a distinct `char` type (unlike C++/Java `char`).

```python
s = "abc"
print(s[0], type(s[0]))     # => a <class 'str'>

print(ord('a'))              # => 97
print(chr(97))                # => a
print(ord('c') - ord('a'))     # => 2      common "letter to index" trick
print(chr(ord('a') + 2))        # => c      "index to letter" trick
```

### Complete Method Table

| Method | What it does | Mutates? | Complexity |
|---|---|---|---|
| `len(s)` | length — a FUNCTION, not `.length` | no | O(1) |
| `s[i]` | char at index (length-1 str) | no | O(1) |
| `s[i:j:k]` | slice | no | O(k) new string |
| `s.find(sub)` | index of first match, **-1** if absent | no | O(n*m) |
| `s.rfind(sub)` | index of last match, -1 if absent | no | O(n*m) |
| `s.index(sub)` | like find but **raises ValueError** if absent | no | O(n*m) |
| `s.count(sub)` | non-overlapping occurrences | no | O(n) |
| `s.split(sep=None, maxsplit=-1)` | split into list; no-arg splits on ANY whitespace & drops empties | no | O(n) |
| `s.rsplit(sep, maxsplit)` | split from the right, limited count | no | O(n) |
| `s.splitlines()` | split on line boundaries | no | O(n) |
| `sep.join(iterable)` | join strings with `sep` between — STR method taking an iterable | no | O(n) |
| `s.strip(chars=None)` | remove leading+trailing whitespace (or given chars) | no | O(n) |
| `s.lstrip()` / `s.rstrip()` | strip left / right only | no | O(n) |
| `s.replace(old, new, count=-1)` | replace occurrences, ALL by default | no | O(n) |
| `s.startswith(prefix)` | prefix check; `prefix` may be a TUPLE | no | O(k) |
| `s.endswith(suffix)` | suffix check; may be a TUPLE | no | O(k) |
| `s.lower()` / `s.upper()` | case conversion | no | O(n) |
| `s.swapcase()` | swap case of every letter | no | O(n) |
| `s.title()` / `s.capitalize()` | title-case / capitalize first letter | no | O(n) |
| `s.isdigit()` | all chars are digits (non-empty) | no | O(n) |
| `s.isalpha()` | all chars are letters | no | O(n) |
| `s.isalnum()` | all chars are letters/digits | no | O(n) |
| `s.isspace()` | all chars are whitespace | no | O(n) |
| `s.islower()` / `s.isupper()` | case checks | no | O(n) |
| `s.zfill(width)` | left-pad with `'0'` to width (keeps sign) | no | O(n) |
| `s.ljust(width, fill=' ')` / `s.rjust(...)` / `s.center(...)` | pad to width | no | O(n) |
| `s.format(...)` | old-style templating | no | O(n) |
| `s.encode(enc='utf-8')` | str -> bytes | no | O(n) |

```python
s = "  Hello, World!  "
print(len(s))                    # => 17
print(s.strip())                  # => 'Hello, World!'
print(s.lower())                   # => '  hello, world!  '
print("Hello".find("l"))            # => 2
print("Hello".find("z"))             # => -1
# print("Hello".index("z"))          # ValueError: substring not found
print("banana".count("a"))            # => 3
print("a,b,,c".split(","))             # => ['a', 'b', '', 'c']   keeps empties with sep
print("  a  b  c  ".split())            # => ['a', 'b', 'c']       no-arg drops empties
print("a.b.c".rsplit(".", 1))            # => ['a.b', 'c']         split from right, limit 1
print("-".join(["a", "b", "c"]))          # => 'a-b-c'
print("Hello World".replace("o", "0"))     # => 'Hell0 W0rld'   ALL occurrences by default
print("file.tar.gz".startswith(("file", "data")))  # => True   tuple of options
print("report.csv".endswith((".csv", ".tsv")))       # => True
print("42".zfill(5))                                  # => '00042'
print("42".rjust(5, "0"))                               # => '00042'   equivalent here
print("hi".center(6, "*"))                               # => '**hi**'
```

> **Gotcha:** `sep.join(iterable)` is a method OF the separator string, called WITH the iterable — backwards from what JS (`arr.join(sep)`) or intuition suggests. `"-".join(["a","b"])`, not `["a","b"].join("-")`.

> **Gotcha:** `s.replace(old, new)` replaces ALL occurrences by default — unlike JS `String.replace(str, new)` which replaces only the first match (JS needs `replaceAll` or a global regex for all).

> **Gotcha:** `find`/`rfind` return `-1` on failure; `index`/`rindex` raise `ValueError`. Pick based on whether "not found" is an expected case (use find) or a bug (use index, let it raise loudly).

### f-strings

```python
x, pi = 42, 3.14159265
name = "Ann"
print(f"{name} has {x} points")        # => Ann has 42 points
print(f"{pi:.2f}")                       # => 3.14        2 decimal places
print(f"{x:03d}")                         # => 042         zero-padded width 3
print(f"{x:>6}")                           # => '    42'    right-align width 6
print(f"{x:<6}|")                           # => '42    |'   left-align width 6
print(f"{x:,}")                              # => 42          (try with 1000000 => '1,000,000')
```

f-string debug spec **[3.8]** shows expression text and value:

```python
val = 7
print(f"{val=}")     # => val=7
```

Expressions can appear inside `{}`:

```python
a, b = 3, 4
print(f"{a} + {b} = {a + b}")   # => 3 + 4 = 7
```

### string <-> list, string <-> number

```python
chars = list("hello")          # => ['h', 'e', 'l', 'l', 'o']
back = "".join(chars)           # => 'hello'
words = "a b c".split()          # => ['a', 'b', 'c']

n = int("42")                     # => 42
h = int("2a", 16)                  # => 42   base-16 parse
f = float("3.14")                   # => 3.14
s = str(42)                          # => '42'
print(chars, back, words, n, h, f, s)
```

> **Gotcha:** `int("3.0")` raises `ValueError` — `int()` won't parse a decimal-point string directly; go through `float()` first: `int(float("3.0"))`.

### Comparison — lexicographic by code point

```python
print("apple" < "banana")   # => True
print("Zebra" < "apple")     # => True   'Z' (90) < 'a' (97) in ASCII — case matters!
print("abc" < "abcd")         # => True   prefix is "less"
```

> **vs Java:** no `.compareTo()` needed — `<`, `>`, `==` work directly on strings and compare by Unicode code point, same rule as tuple comparison above.

### Sorting characters, multiplication, membership

```python
print("".join(sorted("dcba")))   # => 'abcd'   sort chars of a string
print("ab" * 3)                    # => 'ababab'
print("ell" in "hello")             # => True    clean substring check (no .contains() needed)
```

### translate/maketrans (brief)

```python
table = str.maketrans("abc", "xyz")   # map a->x, b->y, c->z
print("aabbcc".translate(table))       # => 'xxyyzz'

remove_vowels = str.maketrans("", "", "aeiou")
print("hello world".translate(remove_vowels))   # => 'hll wrld'
```

> Fast for bulk single-character substitution/deletion — O(n), single pass, faster than chained `.replace()` calls.

### Building output efficiently — the CP rule

```python
# CP idiom: accumulate parts in a list, join once
out = []
for i in range(5):
    out.append(str(i))
print(" ".join(out))   # => '0 1 2 3 4'
```

> Same rule as above: never grow a string with `+=` in a tight loop. This applies directly to competitive-programming output formatting (building large multi-line answers before a single `print`/`sys.stdout.write`, forward-ref Part VI).

### Raw strings and multi-line strings

```python
path = r"C:\new\test"       # raw string — backslashes NOT escape sequences
print(path)                   # => C:\new\test

regex = r"\d+\.\d+"            # common for regex patterns (Part VI) — avoids double-escaping
print(regex)                     # => \d+\.\d+

multi = """line one
line two"""
print(multi)   # => line one
               #    line two
```

> **Gotcha:** without the `r` prefix, `"C:\new\test"` would try to interpret `\n` and `\t` as escapes (newline, tab) — a classic Windows-path bug. Always use raw strings for regex patterns and filesystem paths with backslashes.

### String identity / interning (brief, know it exists)

```python
a = "hello"
b = "hello"
print(a is b)     # => True (usually) — CPython interns short literal strings

c = "".join(["h", "e", "l", "l", "o"])
print(a is c)       # => False — built at runtime, not interned automatically
print(a == c)         # => True — value equality still holds
```

> **Gotcha:** NEVER rely on `is` for string comparison — interning is a CPython implementation detail, not a language guarantee (and doesn't apply to runtime-built strings, or on other implementations). Always use `==` for string equality, exactly like Java (never use `==` for Java string content either) — unlike C++ `std::string operator==` which is always value comparison and has no identity subtlety at all.

### Case-insensitive comparison and additional format() examples

```python
print("Hello".lower() == "hello".lower())   # => True   standard case-insensitive check

print("{} + {} = {}".format(2, 3, 5))         # => '2 + 3 = 5'    old-style .format()
print("{0} {1} {0}".format("a", "b"))           # => 'a b a'       positional index reuse
print("{name} is {age}".format(name="Al", age=9))  # => 'Al is 9'  keyword args
```

> f-strings (shown above) are preferred in modern code (faster, more readable); `.format()` still appears in older codebases and template strings built from a non-literal format string at runtime (f-strings can't do that — the format string must be a literal).

### String as an iterable — direct looping

```python
for ch in "abc":
    print(ch)   # => a / b / c

print(sum(1 for ch in "hello" if ch in "aeiou"))   # => 2   vowel count via generator expr
```

> A `str` is directly iterable char-by-char — no `.toCharArray()` (Java) or manual index loop (C++) needed for a simple scan.

### Composite DSA pattern — group anagrams (ties str + tuple + sort together)

```python
words = ["eat", "tea", "tan", "ate", "nat", "bat"]
groups = {}
for w in words:
    key = "".join(sorted(w))       # canonical form: sorted chars as a string key
    groups.setdefault(key, []).append(w)

print(sorted(groups.values()))
# => [['bat'], ['eat', 'tea', 'ate'], ['tan', 'nat']]
```

> This single pattern chains together: `sorted(w)` returning a list of chars, `"".join(...)` turning it back into a hashable `str` key (tuples work as keys too — `tuple(sorted(w))` is equally valid and slightly faster since it skips the join), and `dict.setdefault` for group-by. A very common composite in string-heavy DSA problems.

---

## 11. array module, bytes, and Fixed-Type Sequences

### array.array — compact typed array

`array.array` stores a single primitive C type contiguously (like a C++ `int[]`/`vector<int>` in raw memory layout) — far more memory-compact than a `list` of `int` objects (each Python `int` in a list is a full boxed object with overhead; a plain `list` of a million ints can be 4-8x the memory of an `array`).

```python
from array import array

a = array('i', [1, 2, 3, 4])   # typecode 'i' = signed int
a.append(5)
print(a)              # => array('i', [1, 2, 3, 4, 5])
print(a[0], len(a))    # => 1 5
print(list(a))          # => [1, 2, 3, 4, 5]   convert to list when needed
```

Common typecodes:

| Typecode | C type | Python type | Min size (bytes) |
|---|---|---|---|
| `'b'` / `'B'` | signed/unsigned char | int | 1 |
| `'i'` / `'I'` | signed/unsigned int | int | 2 |
| `'l'` / `'L'` | signed/unsigned long | int | 4 |
| `'q'` / `'Q'` | signed/unsigned long long | int | 8 |
| `'f'` | float | float | 4 |
| `'d'` | double | float | 8 |

> **Use when:** you have millions of numeric values and are hitting a memory limit (some judges cap memory tightly), or need `array.array` for interop with `struct`/binary I/O. **Otherwise:** plain `list` is fine and far more common in DSA/CP solutions — `array` gives up mixed-type flexibility and most list convenience methods (no `.sort()` with `key=`, limited comprehension ergonomics) for a memory/cache win that rarely matters at typical CP input sizes (10^5-10^7).

### bytes and bytearray

```python
b = b"hello"                 # bytes literal — IMMUTABLE
print(b[0])                   # => 104   indexing bytes gives an INT, not a length-1 bytes!
print(list(b))                 # => [104, 101, 108, 108, 111]

ba = bytearray(b"hello")        # MUTABLE version
ba[0] = 72                       # ord('H')
print(ba)                         # => bytearray(b'Hello')
print(ba.decode())                 # => 'Hello'
```

> **Gotcha:** indexing `bytes`/`bytearray` returns an `int` (0-255), NOT a length-1 `bytes`/`str` — this differs from indexing a `str`, which returns a length-1 `str`. Easy to trip on when porting string logic to byte logic.

**Fast I/O use case** (forward-referenced fully in Part VI — I/O & performance):

```python
import sys
data = sys.stdin.buffer.read().split()   # reads raw bytes, splits on whitespace
nums = list(map(int, data))               # int() parses bytes objects directly too
```

> `sys.stdin.buffer` (a `BufferedReader`, yields `bytes`) skips text-decoding overhead and is the standard trick for fast bulk input in CPython on Codeforces-scale input (10^6+ tokens), where `input()` per line is too slow.

### memoryview (brief)

```python
data = bytearray(b"hello world")
mv = memoryview(data)
print(mv[0:5].tobytes())   # => b'hello'   slice WITHOUT copying the underlying buffer
```

> Lets you slice/window into a large `bytes`/`bytearray`/`array` buffer without copying — relevant for large binary-parsing tasks; rarely needed in typical algorithmic DSA problems, mentioned for completeness.

### struct module (brief) — packing binary data

```python
import struct

packed = struct.pack('ii', 10, 20)     # pack two C ints into bytes
print(packed)                            # => b'\n\x00\x00\x00\x14\x00\x00\x00'
print(struct.unpack('ii', packed))         # => (10, 20)
```

> Rarely needed for DSA/CP — relevant when a problem's I/O format is literally raw binary, or when interfacing with `array`/`bytes` at the byte-layout level. Mentioned for completeness alongside `array`/`bytes`/`memoryview`.

### str vs bytes — the Python 3 distinction (brief)

```python
s = "héllo"
b = s.encode("utf-8")     # str -> bytes
print(b)                   # => b'h\xc3\xa9llo'
s2 = b.decode("utf-8")      # bytes -> str
print(s2)                    # => héllo
```

> Python 3 strictly separates text (`str`, Unicode code points) from binary data (`bytes`). Mixing them (`"a" + b"b"`) raises `TypeError`. Most DSA/CP problems stay entirely in `str`/`int`; this distinction mainly matters for fast I/O (reading raw bytes, then `int()`/`decode()` as needed) and any problem touching encodings/checksums directly.

### The tuple-as-key pattern — summary

Recap from Section 9, listed here for the decision table below: for any "coordinate" or "composite key" need in a `dict`/`set` (visited cells, memoization keys, edges as `(u, v)`), reach for a plain `tuple` — it's hashable, comparable, and needs zero setup, unlike C++ (`pair` needs a custom hash for `unordered_set`/`unordered_map`) or JS (string-concatenation keys).

```python
visited = set()
def dfs(r, c):
    if (r, c) in visited:
        return
    visited.add((r, c))
    # ... explore neighbors ...

memo = {}
def solve(i, j):
    if (i, j) in memo:
        return memo[(i, j)]
    # ... compute ...
    memo[(i, j)] = result
    return result
```

### Decision table — which sequence type to reach for

| Need | Use | Why |
|---|---|---|
| General growable collection, mixed ops | `list` | default choice, richest method set |
| Fixed-size record / dict key / set member / heap element | `tuple` | immutable + hashable + lexicographic compare |
| FIFO queue, need O(1) push/pop at both ends | `collections.deque` *(Part III)* | `list.pop(0)` is O(n); deque is O(1) both ends |
| Millions of homogeneous numeric values, memory-critical | `array.array` | compact C-level storage, less overhead than boxed `list[int]` |
| Text data, char/substring ops | `str` | immutable, rich text methods, code-point comparison |
| Raw byte data / fast bulk I/O | `bytes` / `bytearray` | avoids text decode overhead, `sys.stdin.buffer` reads bytes |
| Need O(1) membership/lookup | `set` / `dict` *(Part III)* | `list`/`tuple` membership is O(n) |

> **Rule of thumb for CP/DSA:** default to `list` for anything mutable and `tuple` for anything used as a key or returned/compared as a fixed group. Reach for `array`/`bytes`/`memoryview` only when a memory or raw-I/O constraint specifically demands it — they are the exception, not the starting point.

---

# PART III — DICTS, SETS AND SPECIALIZED CONTAINERS

## 12. dict and the collections Mappings

`dict` is Python's hash map. Since **3.7 it is guaranteed insertion-ordered** (CPython 3.6 had this as an implementation detail; 3.7+ makes it a language guarantee) — this is like Java's `LinkedHashMap`, and unlike C++'s `unordered_map` (no order guarantee) or `std::map` (sorted, not insertion order).

> **vs C++/Java:** `dict` ≈ `unordered_map` in average complexity (O(1) lookup/insert/delete) but with insertion order preserved like `LinkedHashMap`. There is no built-in `TreeMap`/`std::map` equivalent — see §15 (sortedcontainers).

### Creation

```python
d1 = {}                              # empty dict (NOT a set!)
d2 = {"a": 1, "b": 2}
d3 = dict(a=1, b=2)                  # kwargs -> keys (keys must be valid identifiers)
d4 = dict(zip(["a", "b"], [1, 2]))   # from two parallel lists
d5 = {k: k * k for k in range(4)}    # dict comprehension
d6 = dict.fromkeys(["a", "b", "c"], 0)  # all keys -> same default value
print(d3, d4, d5, d6)
# => {'a': 1, 'b': 2} {'a': 1, 'b': 2} {0: 0, 1: 1, 2: 4, 3: 9} {'a': 0, 'b': 0, 'c': 0}
```

> **Gotcha:** `dict.fromkeys(keys, [])` gives every key the **same** list object (mutating one mutates all). Use a dict/defaultdict comprehension `{k: [] for k in keys}` if the value is mutable.

### Complete method table

| Method | What it does | Complexity |
|---|---|---|
| `d[k]` | read/write; **raises `KeyError`** if `k` missing on read | O(1) avg |
| `d.get(k, default=None)` | safe read, never raises | O(1) avg |
| `d.setdefault(k, default)` | return `d[k]` if present, else insert `default` and return it | O(1) avg |
| `d.pop(k[, default])` | remove `k`, return its value; raises `KeyError` if missing and no default given | O(1) avg |
| `d.popitem()` | remove and return **last-inserted** `(k, v)` pair (LIFO order) | O(1) |
| `d.keys()` | view of keys (live, set-like) | O(1) to create |
| `d.values()` | view of values (live) | O(1) to create |
| `d.items()` | view of `(k, v)` pairs (live) | O(1) to create |
| `d.update(other)` | merge in-place from dict/iterable of pairs/kwargs | O(len(other)) |
| `k in d` | membership test on **keys** | O(1) avg |
| `d.clear()` | remove all items | O(n) |
| `d.copy()` | shallow copy | O(n) |
| `len(d)` | number of entries | O(1) |
| `d.keys() & other.keys()` | set-style ops on key views | O(min(len)) |

### Iteration

```python
d = {"a": 1, "b": 2, "c": 3}
for k in d:                 # iterates keys, insertion order
    pass
for k, v in d.items():
    pass
for v in d.values():
    pass
print(list(d), list(d.items()))
# => ['a', 'b', 'c'] [('a', 1), ('b', 2), ('c', 3)]
```

> **Gotcha:** Never add/remove keys while iterating over `d` (or `d.keys()`) directly — raises `RuntimeError: dictionary changed size during iteration`. Iterate over `list(d.items())` if you need to mutate.

### KeyError vs `.get()` — the core trap

```python
d = {"a": 1}
# d["z"]                # KeyError: 'z'  -- DON'T do this for optional keys
print(d.get("z"))        # => None
print(d.get("z", 0))     # => 0
```

### Frequency-count idiom

```python
s = "abracadabra"
freq = {}
for ch in s:
    freq[ch] = freq.get(ch, 0) + 1
print(freq)
# => {'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1}
```

### `collections.defaultdict`

Auto-creates a default value on first access of a missing key — no more `if k not in d` boilerplate. THE tool for counting and adjacency lists.

```python
from collections import defaultdict

# 1) counting
cnt = defaultdict(int)
for ch in "banana":
    cnt[ch] += 1
print(dict(cnt))
# => {'b': 1, 'a': 3, 'n': 2}

# 2) adjacency list (graph edges)
g = defaultdict(list)
edges = [(0, 1), (0, 2), (1, 2)]
for u, v in edges:
    g[u].append(v)
    g[v].append(u)
print(dict(g))
# => {0: [1, 2], 1: [0, 2], 2: [0, 1]}

# 3) grouping into sets (dedupe neighbors)
groups = defaultdict(set)
groups["even"].add(2)
groups["even"].add(2)
groups["odd"].add(3)
print({k: sorted(v) for k, v in groups.items()})
# => {'even': [2], 'odd': [3]}
```

> **Gotcha:** Merely **reading** `d[k]` on a `defaultdict` (even inside `if d[k] == 0:`) **creates** the key with the default value if it wasn't present. This silently bloats the dict and breaks `k in d` checks written afterward expecting `k` to be absent. Use `k in d` or `d.get(k)` for a pure existence check when you don't want the side effect. Use a plain `dict` + `.get()` if you need to distinguish "never touched" from "touched, default value."

```python
from collections import defaultdict
d = defaultdict(int)
print(len(d))          # => 0
_ = d["x"]              # just reading — no assignment!
print(len(d), "x" in d) # => 1 True   <- "x" now exists with value 0
```

### `collections.Counter`

A `dict` subclass specialized for counting. Missing keys return `0` (not `KeyError`) without turning into `defaultdict`'s creation side-effect on write, though reading also doesn't insert.

```python
from collections import Counter

c = Counter("mississippi")
print(c)
# => Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})

print(c["i"], c["z"])          # missing key -> 0, no KeyError, no insertion
# => 4 0

print(c.most_common(2))        # top-2 by count, ties broken by insertion order
# => [('i', 4), ('s', 4)]

c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
print(c1 + c2)                  # elementwise add
# => Counter({'a': 4, 'b': 3})
print(c1 - c2)                  # elementwise subtract, DROPS zero/negative counts
# => Counter({'a': 2})
print(c1 & c2)                  # min per key (multiset intersection)
# => Counter({'a': 1, 'b': 1})
print(c1 | c2)                  # max per key (multiset union)
# => Counter({'a': 3, 'b': 2})

print(list(Counter(a=2, b=1).elements()))  # expand back to a multiset
# => ['a', 'a', 'b']

# anagram check via subtraction (empty result == anagram)
def is_anagram(a: str, b: str) -> bool:
    return not (Counter(a) - Counter(b)) and not (Counter(b) - Counter(a))

print(is_anagram("listen", "silent"), is_anagram("abc", "abd"))
# => True False
```

> **Gotcha:** `Counter.__sub__` (`-`) and `&`/`|` **discard non-positive counts** — `Counter(a=1) - Counter(a=5)` is `Counter()`, not `Counter(a=-4)`. Use `c1.subtract(c2)` (in-place, keeps negatives) if you need signed deltas.

```python
c = Counter(a=1)
c.subtract(Counter(a=5))
print(c)
# => Counter({'a': -4})
```

### `collections.OrderedDict`

Largely historical since plain `dict` is ordered (3.7+), but still useful for `move_to_end` + `popitem(last=False)` semantics that read intent clearly, and equality comparison that (unlike plain dict) considers order — plus the classic **LRU cache** sketch (though `functools.lru_cache`/`cache` covers most memoization needs directly).

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.od = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.od:
            return -1
        self.od.move_to_end(key)       # mark as recently used
        return self.od[key]

    def put(self, key: int, value: int) -> None:
        if key in self.od:
            self.od.move_to_end(key)
        self.od[key] = value
        if len(self.od) > self.cap:
            self.od.popitem(last=False)  # evict least-recently-used (oldest)

lru = LRUCache(2)
lru.put(1, 1); lru.put(2, 2)
print(lru.get(1))   # => 1   (1 is now most-recent)
lru.put(3, 3)        # evicts 2 (least-recently-used)
print(lru.get(2))   # => -1
print(lru.get(3))   # => 3
```

> **vs C++/Java:** plain `dict` with `d.pop(k); d[k] = v` (delete-then-reinsert to move to end) also works for LRU since 3.7, but `OrderedDict.move_to_end(k, last=True/False)` is O(1) and more explicit — it can also move to the **front** (`last=False`), which plain dict cannot do without a full rebuild.

### dict as adjacency list / memo table / visited map

```python
# memo table for DP, tuple keys
dp = {}
def fib_memo(n):
    if n in dp:
        return dp[n]
    if n <= 1:
        return n
    dp[n] = fib_memo(n - 1) + fib_memo(n - 2)
    return dp[n]
print(fib_memo(10))
# => 55

# grid DP with (i, j) tuple keys
grid_dp = {}
grid_dp[(0, 0)] = 1
grid_dp[(1, 2)] = grid_dp.get((0, 2), 0) + grid_dp.get((1, 1), 0)
print(grid_dp)
# => {(0, 0): 1, (1, 2): 0}
```

> **Gotcha:** Tuple keys must contain only hashable elements. `(i, j)` of ints works; a `[i, j]` list does **not** (`TypeError: unhashable type: 'list'`).

### Merging dicts

```python
a = {"x": 1, "y": 2}
b = {"y": 20, "z": 3}
merged = {**a, **b}            # b's values win on key collision
print(merged)
# => {'x': 1, 'y': 20, 'z': 3}

merged2 = a | b                # 3.9+, same semantics, more readable
print(merged2)
# => {'x': 1, 'y': 20, 'z': 3}
a |= b                          # 3.9+, in-place update
print(a)
# => {'x': 1, 'y': 20, 'z': 3}
```

---

## 13. set and frozenset

`set` is an unordered collection of unique, hashable elements — same average complexity profile as `unordered_set` in C++ / `HashSet` in Java.

### Creation

```python
s1 = set()             # empty set — {} is a DICT, this is the classic trap
s2 = {1, 2, 3}          # non-empty literal is fine
s3 = set([1, 2, 2, 3])  # from iterable, dedupes
s4 = {x * x for x in range(5)}  # set comprehension
print(s1, s2, s3, s4)
# => set() {1, 2, 3} {1, 2, 3} {0, 1, 4, 9, 16}
```

> **Gotcha:** `{}` is **always** an empty `dict`, never an empty set. `type({})` is `dict`. Use `set()` for an empty set.

### Complete method table

| Method | What it does | Complexity |
|---|---|---|
| `s.add(x)` | insert `x` | O(1) avg |
| `s.remove(x)` | delete `x`; **raises `KeyError`** if absent | O(1) avg |
| `s.discard(x)` | delete `x` if present; **no error** if absent | O(1) avg |
| `s.pop()` | remove and return an **arbitrary** element | O(1) |
| `x in s` | membership test | O(1) avg |
| `s.clear()` | remove all elements | O(n) |
| `s.copy()` | shallow copy | O(n) |
| `len(s)` | element count | O(1) |

### Set algebra — built into the language

Unlike JavaScript (which has no built-in set-algebra operators pre-2024's proposal), Python's `set` has full algebra both as operators and named methods, plus in-place variants.

| Operation | Operator | Method | In-place operator |
|---|---|---|---|
| Union | `a \| b` | `a.union(b)` | `a \|= b` |
| Intersection | `a & b` | `a.intersection(b)` | `a &= b` |
| Difference | `a - b` | `a.difference(b)` | `a -= b` |
| Symmetric difference | `a ^ b` | `a.symmetric_difference(b)` | `a ^= b` |
| Subset test | `a <= b` | `a.issubset(b)` | — |
| Proper subset | `a < b` | — | — |
| Superset test | `a >= b` | `a.issuperset(b)` | — |
| Disjoint test | — | `a.isdisjoint(b)` | — |

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
print(a | b)   # => {1, 2, 3, 4, 5, 6}
print(a & b)   # => {3, 4}
print(a - b)   # => {1, 2}
print(a ^ b)   # => {1, 2, 5, 6}
print({1, 2} <= a)   # => True   (subset)
print(a >= {1, 2})   # => True   (superset)
print(a.isdisjoint({100, 200}))  # => True

a |= {10}      # in-place union
print(a)
# => {1, 2, 3, 4, 10}
```

> **vs C++/JS:** C++ has no built-in set algebra either (you'd write `std::set_union` with iterators into a new container); JS `Set` has *no* built-in union/intersection at all pre-ES2025's `Set.prototype.union` etc. Python's operator-level set algebra is a genuine ergonomic edge — use it freely for problems like "common elements", "symmetric difference of two arrays."

### O(1) membership — the `visited` idiom

```python
def has_path_no_cycle(edges, start, target):
    seen = set()
    stack = [start]
    while stack:
        node = stack.pop()
        if node == target:
            return True
        if node in seen:      # O(1) — replaces an O(n) list scan
            continue
        seen.add(node)
        stack.extend(edges.get(node, []))
    return False

print(has_path_no_cycle({0: [1], 1: [2], 2: []}, 0, 2))
# => True
```

### Dedupe

```python
lst = [3, 1, 2, 3, 1, 4]
deduped_unordered = list(set(lst))       # order NOT preserved
print(sorted(deduped_unordered))
# => [1, 2, 3, 4]

deduped_ordered = list(dict.fromkeys(lst))  # first-seen order preserved
print(deduped_ordered)
# => [3, 1, 2, 4]
```

> **Gotcha:** `set(lst)` silently discards duplicate order information. For order-preserving dedupe, `dict.fromkeys(lst)` (keys are ordered, O(n)) beats a manual "seen set + list" loop for terseness, though the manual loop is equally valid and often clearer in DSA code.

### frozenset — immutable, hashable

`set` is unhashable (can't be a dict key or set element) because it's mutable. `frozenset` is the immutable counterpart — hashable, so it can nest inside another set or be a dict key. Useful for memoizing over "visited combination of items" (subset-sum / bitmask-like state where the state is naturally a set).

```python
visited_states = set()
state1 = frozenset({1, 2, 3})
visited_states.add(state1)
print(frozenset({3, 2, 1}) in visited_states)   # order doesn't matter for sets
# => True

# frozenset as a dict key, e.g. caching results per unordered pair
cache = {}
cache[frozenset({"A", "B"})] = 42
print(cache[frozenset({"B", "A"})])
# => 42
```

### Coordinate-set idiom — tuples are hashable

```python
# grid BFS/DFS visited set of (row, col) — the standard pattern
seen = set()
seen.add((0, 0))
seen.add((1, 2))
print((0, 0) in seen, (5, 5) in seen)
# => True False
```

> **vs JS:** JS's `Set`/`Map` compare objects by reference, so `[0,0]` as a "coordinate key" never matches another literally-equal array — you're forced to stringify (`"0,0"`). Python tuples are value-hashable, so `(0, 0) == (0, 0)` and both hash identically — no stringification needed. This is a genuine Python win, use it.

### set vs dict vs list — when to reach for which

| Need | Container |
|---|---|
| Just "have I seen this?" | `set` |
| Seen-with-associated-value (count, parent, distance) | `dict` |
| Ordered sequence, duplicates allowed, index access | `list` |
| Both order AND uniqueness AND O(1) lookup | `dict` (keys) or `dict.fromkeys` |

---

## 14. deque, heapq — Stack, Queue, Deque, Priority Queue

### Stack — just use `list`

```python
stack = []
stack.append(1)
stack.append(2)
stack.append(3)
print(stack.pop())   # LIFO, O(1) amortized
# => 3
print(stack)
# => [1, 2]
```

### Queue — `list.pop(0)` is O(n); use `deque`

```python
from collections import deque

q = [1, 2, 3]
q.pop(0)   # O(n) -- shifts every remaining element. AVOID for queues.
```

> **Gotcha:** `list.pop(0)` and `list.insert(0, x)` are both O(n) because every other element must shift. A `list` is a poor queue. Always use `collections.deque` for FIFO.

```python
from collections import deque

def bfs(graph, start):
    order = []
    seen = {start}
    q = deque([start])          # O(1) append/popleft
    while q:
        node = q.popleft()      # O(1) dequeue — the key operation
        order.append(node)
        for nb in graph.get(node, []):
            if nb not in seen:
                seen.add(nb)
                q.append(nb)
    return order

g = {0: [1, 2], 1: [3], 2: [3], 3: []}
print(bfs(g, 0))
# => [0, 1, 2, 3]
```

### deque in full

| Method | What it does | Complexity |
|---|---|---|
| `append(x)` | push to right end | O(1) |
| `appendleft(x)` | push to left end | O(1) |
| `pop()` | pop from right end | O(1) |
| `popleft()` | pop from left end | O(1) |
| `extend(iterable)` | append many to right | O(k) |
| `extendleft(iterable)` | append many to left (**reversed** order) | O(k) |
| `rotate(n)` | rotate right by `n` (negative = left) | O(k) |
| `d[i]` | indexed access | **O(n)** — not O(1) like list! |
| `deque(maxlen=k)` | bounded deque, auto-evicts opposite end when full | — |

```python
from collections import deque

d = deque([1, 2, 3])
d.appendleft(0)
d.append(4)
print(d)
# => deque([0, 1, 2, 3, 4])

d.rotate(1)     # right rotate by 1
print(d)
# => deque([4, 0, 1, 2, 3])
d.rotate(-2)    # left rotate by 2
print(d)
# => deque([1, 2, 3, 4, 0])

d.extendleft([9, 8, 7])   # each pushed to left in turn -> reversed relative order
print(d)
# => deque([7, 8, 9, 1, 2, 3, 4, 0])
```

> **Gotcha:** `extendleft([9, 8, 7])` does NOT produce `[9, 8, 7, ...]` at the front — each element is appended-left one at a time, so the input order is **reversed** in the result. If you want `[9, 8, 7, ...]` at front, `extendleft(reversed([9, 8, 7]))`.

> **Gotcha:** `d[i]` on a deque walks from whichever end is closer — O(n), NOT O(1) like `list[i]`. Don't use `deque` when you need random-access indexing in a hot loop; use `list` (or both a deque for the ends and a list if you truly need both).

**Sliding window with `maxlen`:**

```python
from collections import deque

window = deque(maxlen=3)
for x in [1, 2, 3, 4, 5]:
    window.append(x)             # oldest auto-evicted once full
    print(list(window))
# => [1]
# => [1, 2]
# => [1, 2, 3]
# => [2, 3, 4]
# => [3, 4, 5]
```

**Monotonic deque** (e.g. sliding-window maximum — LC 239):

```python
from collections import deque

def max_sliding_window(nums, k):
    dq = deque()   # stores INDICES, values kept decreasing left->right
    out = []
    for i, x in enumerate(nums):
        while dq and nums[dq[-1]] <= x:
            dq.pop()               # drop smaller values from the back
        dq.append(i)
        if dq[0] <= i - k:
            dq.popleft()           # drop index that fell out of window
        if i >= k - 1:
            out.append(nums[dq[0]])
    return out

print(max_sliding_window([1, 3, -1, -3, 5, 3, 6, 7], 3))
# => [3, 3, 5, 5, 6, 7]
```

### heapq — priority queue, MIN-HEAP ONLY

`heapq` is a **module of free functions operating on a plain `list`**, not a class with a `.push()`/`.pop()` API — a different shape from C++'s `priority_queue` or Java's `PriorityQueue`. It is always a **min-heap**; there is no `heapq.MaxHeap`.

> **vs C++/Java:** C++'s `std::priority_queue` defaults to max-heap; Java's `PriorityQueue` defaults to min-heap but takes a `Comparator`. Python's `heapq` is min-heap only, full stop — negate values for a max-heap (see below). There's also no O(log n) "decrease-key" — see lazy deletion below.

| Function | What it does | Complexity |
|---|---|---|
| `heapq.heappush(h, x)` | push `x` onto heap `h` | O(log n) |
| `heapq.heappop(h)` | pop and return smallest | O(log n) |
| `h[0]` | peek smallest without popping | O(1) |
| `heapq.heapify(lst)` | turn a list into a heap **in-place** | O(n) |
| `heapq.heappushpop(h, x)` | push then pop-smallest (one op, faster than separate calls) | O(log n) |
| `heapq.heapreplace(h, x)` | pop-smallest then push `x` (**heap can be temporarily "wrong order" — assumes non-empty**) | O(log n) |
| `heapq.nlargest(k, iterable, key=None)` | k largest, sorted descending | O(n log k) |
| `heapq.nsmallest(k, iterable, key=None)` | k smallest, sorted ascending | O(n log k) |

```python
import heapq

h = [5, 1, 8, 3]
heapq.heapify(h)              # O(n), in-place
print(h)
# => [1, 3, 8, 5]
heapq.heappush(h, 0)
print(h[0])                    # peek min
# => 0
print(heapq.heappop(h))
# => 0
print(heapq.heappop(h))
# => 1

nums = [7, 2, 9, 1, 5, 6]
print(heapq.nlargest(3, nums))
# => [9, 7, 6]
print(heapq.nsmallest(2, nums))
# => [1, 2]

people = [("bob", 25), ("amy", 19), ("cid", 40)]
print(heapq.nsmallest(2, people, key=lambda p: p[1]))
# => [('amy', 19), ('bob', 25)]
```

**Max-heap via negation — THE idiom:**

```python
import heapq

max_heap = []
for x in [3, 1, 4, 1, 5, 9]:
    heapq.heappush(max_heap, -x)     # push negated
print(-max_heap[0])                   # peek max: negate back
# => 9
print(-heapq.heappop(max_heap))       # pop max: negate back
# => 9
```

For objects/tuples where you want "largest priority first," negate the **key** field, or push `(-priority, item)`:

```python
import heapq

max_heap = []
tasks = [("low", 1), ("high", 9), ("mid", 5)]
for name, priority in tasks:
    heapq.heappush(max_heap, (-priority, name))
while max_heap:
    neg_p, name = heapq.heappop(max_heap)
    print(name, -neg_p)
# => high 9
# => mid 5
# => low 1
```

### Heap of tuples — the tie-break trap

A tuple heap sorts lexicographically: first element compared first, and **only if tied** does it compare the second element (then third, ...). This is exactly what you want for `(distance, node)` in Dijkstra — but it becomes a crash source when the second element is a non-comparable object (e.g. a custom class without `__lt__`, or a dict) and two first-elements tie.

```python
import heapq

# fine: second element is an int, comparable
h = [(2, "b"), (1, "a"), (2, "a")]
heapq.heapify(h)
print(heapq.heappop(h))
# => (1, 'a')

# CRASHES if two priorities tie and payload is not comparable:
class Node:
    pass
# h2 = [(1, Node()), (1, Node())]
# heapq.heapify(h2)   # TypeError: '<' not supported between instances of 'Node' and 'Node'
```

> **Gotcha:** Fix ties by inserting a monotonically increasing counter as the tie-breaker so the comparison never reaches the payload: `(priority, counter, item)`. This also makes the heap **stable** (FIFO among equal priorities), which plain tuple comparison does not guarantee once payloads *are* comparable but you don't want them driving order.

```python
import heapq
import itertools

counter = itertools.count()   # unique, increasing tie-breaker
h = []
heapq.heappush(h, (1, next(counter), {"id": 1}))
heapq.heappush(h, (1, next(counter), {"id": 2}))   # same priority, dict payload
print(heapq.heappop(h))
# => (1, 0, {'id': 1})
```

### Dijkstra with heapq

```python
import heapq

def dijkstra(graph, src):
    # graph: {node: [(neighbor, weight), ...]}
    dist = {src: 0}
    pq = [(0, src)]              # (distance, node)
    visited = set()
    while pq:
        d, u = heapq.heappop(pq)
        if u in visited:          # lazy deletion: skip stale entries
            continue
        visited.add(u)
        for v, w in graph.get(u, []):
            nd = d + w
            if nd < dist.get(v, float("inf")):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))   # push new, don't touch old entry
    return dist

graph = {
    "A": [("B", 1), ("C", 4)],
    "B": [("C", 2), ("D", 5)],
    "C": [("D", 1)],
    "D": [],
}
print(dijkstra(graph, "A"))
# => {'A': 0, 'B': 1, 'C': 3, 'D': 4}
```

> **Gotcha — lazy deletion, no decrease-key:** `heapq` has no way to decrease an existing entry's priority in place (unlike a textbook binary heap with `decrease-key`, or `std::priority_queue` where you'd also typically not have it either — but some languages' heap libs do). The standard workaround, shown above, is **lazy deletion**: push a fresh, better entry when you find a shorter distance, leave the stale one in the heap, and skip stale entries (`if u in visited: continue`) when popped. This keeps the heap correct at the cost of extra (harmless) stale entries — O(E log E) total instead of O(E log V).

### Complexity table — deque & heapq

| Container | Push/append | Pop end | Pop front | Peek min/max | Index access |
|---|---|---|---|---|---|
| `list` (as stack) | O(1) amortized | O(1) amortized | O(n) | — | O(1) |
| `deque` | O(1) both ends | O(1) both ends | O(1) | — | O(n) |
| `heapq` (list-backed) | O(log n) push | O(log n) pop-min | — | O(1) via `h[0]` | O(n) (unordered) |

---

## 15. sortedcontainers — The TreeMap/TreeSet Replacement

Python's standard library has **no balanced-BST / ordered-map container** — no `std::map`, no `TreeMap`, no `TreeSet`. (JavaScript has the same gap.) `dict` is hash-based and unordered by key; `list` + `bisect` gives sorted order but O(n) insertion. The de facto answer used in competitive programming and interviews is the third-party **`sortedcontainers`** library.

> **Availability:** `sortedcontainers` is **preinstalled on LeetCode** (`from sortedcontainers import SortedList, SortedDict, SortedSet` works with no setup). On **Codeforces**, it is available for CPython submissions on recent judges but is *not guaranteed on every judge/PyPy configuration* — check before relying on it in a live contest, and know the bisect/Fenwick fallback (below) in case it isn't.

```python
from sortedcontainers import SortedList, SortedDict, SortedSet
```

### SortedList — multiset + ordered array + order-statistics, all in one

Internally a list of small sorted sublists (a load-balanced structure), giving O(log n) for add/remove/index/rank operations — unlike a plain `list`, where `bisect.insort` is O(log n) to *find* the spot but O(n) to *insert* (due to shifting).

| Method | What it does | Complexity |
|---|---|---|
| `sl.add(x)` | insert `x`, keeps sorted, **allows duplicates** (multiset) | O(log n) |
| `sl.remove(x)` | remove one occurrence of `x`; raises `ValueError` if absent | O(log n) |
| `sl.discard(x)` | remove one occurrence if present, else no-op | O(log n) |
| `x in sl` | membership | O(log n) |
| `sl[i]` | i-th smallest element (supports negative indices, slicing) | O(log n) |
| `sl.pop(i=-1)` | remove and return element at index `i` | O(log n) |
| `sl.bisect_left(x)` | count of elements `< x` (insertion point, leftmost) | O(log n) |
| `sl.bisect_right(x)` | count of elements `<= x` (insertion point, rightmost) | O(log n) |
| `sl.count(x)` | number of occurrences of `x` | O(log n) |
| `sl.irange(lo, hi)` | iterator over elements in `[lo, hi]` (bounds optional/inclusive flags) | O(log n + k) |
| `sl[0]`, `sl[-1]` | min, max | O(log n) (O(1) for the very ends in practice) |
| `len(sl)` | size | O(1) |

```python
from sortedcontainers import SortedList

sl = SortedList([5, 1, 8, 3])
print(sl)
# => SortedList([1, 3, 5, 8])

sl.add(4)
print(sl)
# => SortedList([1, 3, 4, 5, 8])

print(sl[0], sl[-1])       # min, max
# => 1 8
print(sl[2])                # k-th smallest (0-indexed) -> 3rd smallest
# => 4

sl.remove(3)                 # remove BY VALUE, not by index
print(sl)
# => SortedList([1, 4, 5, 8])

print(sl.bisect_left(5))    # count of elements strictly < 5
# => 2
print(sl.bisect_right(5))   # count of elements <= 5
# => 3

print(list(sl.irange(4, 8)))  # inclusive range query
# => [4, 5, 8]
```

**Count of elements `< x`:**

```python
from sortedcontainers import SortedList
sl = SortedList([1, 3, 3, 5, 7])
print(sl.bisect_left(5))    # how many elements are strictly less than 5
# => 3
```

**k-th smallest (order statistic):**

```python
from sortedcontainers import SortedList
sl = SortedList([9, 2, 7, 4, 1])
k = 2   # 0-indexed -> 3rd smallest
print(sl[k])
# => 4
```

**Predecessor / successor of x (x need not be present):**

```python
from sortedcontainers import SortedList
sl = SortedList([1, 4, 7, 10])

def predecessor(sl, x):
    i = sl.bisect_left(x) - 1
    return sl[i] if i >= 0 else None

def successor(sl, x):
    i = sl.bisect_right(x)
    return sl[i] if i < len(sl) else None

print(predecessor(sl, 8), successor(sl, 8))
# => 7 10
```

**Running median with two SortedLists (or one, using index math):**

```python
from sortedcontainers import SortedList

class RunningMedian:
    def __init__(self):
        self.sl = SortedList()

    def add(self, x):
        self.sl.add(x)

    def median(self):
        n = len(self.sl)
        mid = n // 2
        if n % 2:
            return self.sl[mid]
        return (self.sl[mid - 1] + self.sl[mid]) / 2

rm = RunningMedian()
for x in [5, 15, 1, 3]:
    rm.add(x)
    print(rm.median())
# => 5
# => 10.0
# => 5
# => 4.0
```

> **vs C++/Java:** this single `SortedList` replaces the "two-heap running median" trick you'd otherwise need, and also does the job of `multiset` with `.count()`/order-statistics that C++'s `std::multiset` + manual iterator-walking (or a Policy-Tree / order-statistics tree extension) provides.

### SortedDict — the TreeMap floor/ceiling equivalent

Keeps keys sorted; values are looked up by key like a normal dict, but you get sorted-key iteration, `peekitem`, and bisect-on-keys for floor/ceiling.

| Method | What it does | Complexity |
|---|---|---|
| `sd[k] = v` | insert/update | O(log n) |
| `sd[k]` | read (KeyError if absent) | O(log n) |
| `sd.peekitem(i=-1)` | `(key, value)` at sorted index `i` (default last = max key) | O(log n) |
| `sd.keys()` / `.values()` / `.items()` | sorted views, support indexing | O(1) to get view |
| `sd.bisect_left(k)` / `.bisect_right(k)` | index of `k` among sorted keys | O(log n) |
| `sd.irange(lo, hi)` | keys in `[lo, hi]` | O(log n + k) |

```python
from sortedcontainers import SortedDict

sd = SortedDict({3: "c", 1: "a", 2: "b"})
print(sd)
# => SortedDict({1: 'a', 2: 'b', 3: 'c'})

print(sd.peekitem(0))    # smallest key
# => (1, 'a')
print(sd.peekitem(-1))   # largest key
# => (3, 'c')

sd[5] = "e"
print(list(sd.items()))
# => [(1, 'a'), (2, 'b'), (3, 'c'), (5, 'e')]

# floor(x): largest key <= x
def floor_key(sd, x):
    i = sd.bisect_right(x) - 1
    return sd.keys()[i] if i >= 0 else None

# ceiling(x): smallest key >= x
def ceiling_key(sd, x):
    i = sd.bisect_left(x)
    return sd.keys()[i] if i < len(sd) else None

print(floor_key(sd, 4), ceiling_key(sd, 4))
# => 3 5
```

### SortedSet — ordered, unique

Combines `set`'s uniqueness with `SortedList`'s ordering/bisect — the closest thing to `std::set` / `TreeSet`.

```python
from sortedcontainers import SortedSet

ss = SortedSet([5, 1, 3, 1, 5])
print(ss)
# => SortedSet([1, 3, 5])

ss.add(2)
print(ss)
# => SortedSet([1, 2, 3, 5])
print(ss.bisect_left(3))     # rank of 3 (count of elements < 3)
# => 2
print(2 in ss)
# => True
ss.discard(2)
print(ss)
# => SortedSet([1, 3, 5])
```

### Complexity table

| Container | add/insert | remove | index access `[i]` | contains | min/max | rank / bisect |
|---|---|---|---|---|---|---|
| `SortedList` | O(log n) | O(log n) | O(log n) | O(log n) | O(log n)* | O(log n) |
| `SortedDict` | O(log n) | O(log n) | O(log n) via `.keys()[i]` | O(log n) | O(log n)* | O(log n) |
| `SortedSet` | O(log n) | O(log n) | O(log n) | O(log n) | O(log n)* | O(log n) |

\* practically near O(1) for the very first/last element due to the internal block structure, but formally O(log n).

### Fallback when sortedcontainers is unavailable

If the judge doesn't have `sortedcontainers`:
- **Order statistics / range-sum with updates** → build a **Fenwick tree (BIT)** or **segment tree** over coordinate-compressed values (cross-ref DSA Patterns §27/§28).
- **Occasional inserts, mostly reads** → keep a plain `list`, use `bisect.insort(lst, x)` for insertion (O(log n) to find position, **O(n)** to shift — acceptable if insert count is small) and `bisect.bisect_left`/`bisect_right` for queries.

```python
import bisect

lst = [1, 3, 5, 7]
bisect.insort(lst, 4)      # O(n) insert (shifts elements), but sorted position found in O(log n)
print(lst)
# => [1, 3, 4, 5, 7]
print(bisect.bisect_left(lst, 4))   # rank / count-less-than
# => 2
```

---

## 16. Choosing the Right Container

### Decision table

| Requirement | Python container | Typical DSA pattern |
|---|---|---|
| Order doesn't matter, need O(1) lookup by key | `dict` | hashing, memoization, adjacency list |
| Just presence/absence, O(1) | `set` | visited set, dedupe, "seen" check |
| Need to count occurrences | `Counter` (or `defaultdict(int)`) | frequency map, anagram, sliding window |
| Group items by a key, auto-create buckets | `defaultdict(list)` / `defaultdict(set)` | adjacency list, grouping/bucketing |
| Ordered sequence, index access, duplicates | `list` | arrays, stacks, DP tables |
| FIFO queue, O(1) both ends | `deque` | BFS, sliding window, monotonic deque |
| Get min/max repeatedly, O(log n) insert | `heapq` | Dijkstra, k-th largest, merge k lists, greedy scheduling |
| Sorted order maintained, O(log n) insert/delete/rank | `SortedList` | order statistics, running median, count-inversions |
| Sorted **keys** with value lookup, floor/ceiling | `SortedDict` | TreeMap-style range queries |
| Unique + sorted, O(log n) ops | `SortedSet` | TreeSet-style range/rank queries |
| Immutable, hashable composite key | `tuple` / `frozenset` | dict keys, memo state, visited-state-set elements |
| Need arbitrary-precision bit manipulation | `int` (Python ints are bignum) | bitmask DP, subset enumeration |

### C++/Java → Python container translation

| C++ / Java | Python equivalent |
|---|---|
| `std::vector` / `ArrayList` | `list` |
| `std::unordered_map` / `HashMap` | `dict` |
| `std::map` / `TreeMap` | `SortedDict` (or `dict` + manually sorted keys if read-only) |
| `std::unordered_set` / `HashSet` | `set` |
| `std::set` / `TreeSet` | `SortedSet` (or sorted `list` + `bisect`) |
| `std::multiset` | `Counter` (frequency-only) or `SortedList` (needs order too) |
| `std::stack` | `list` (`append`/`pop`) |
| `std::queue` | `deque` (`append`/`popleft`) |
| `std::priority_queue` (max by default) / `PriorityQueue` (min by default) | `heapq` — **always min-heap**, negate values for max |
| `std::deque` | `deque` |
| `std::bitset<N>` | plain `int` used as a bitmask — Python ints are **arbitrary precision**, no fixed-width overflow |
| `std::pair` / records | `tuple` |

> **Gotcha:** Python `int` never overflows (auto-promotes to bignum), so a bitmask over 10,000 bits is just `1 << 10000` — no `bitset<N>` size declaration needed, but also no automatic wraparound semantics; mask explicitly with `& ((1 << n) - 1)` if you rely on fixed width.

### Master complexity table — all containers, Parts II & III

| Container | Access by index/key | Search/contains | Insert | Delete | Ordered? |
|---|---|---|---|---|---|
| `list` | O(1) | O(n) | O(1) amortized append / O(n) middle | O(n) | insertion order |
| `tuple` | O(1) | O(n) | — (immutable) | — | insertion order |
| `str` | O(1) | O(n) | — (immutable) | — | insertion order |
| `dict` | O(1) avg | O(1) avg (`in`) | O(1) avg | O(1) avg | insertion order (3.7+) |
| `defaultdict` | O(1) avg (creates on read!) | O(1) avg | O(1) avg | O(1) avg | insertion order |
| `Counter` | O(1) avg (0 if missing) | O(1) avg | O(1) avg | O(1) avg | insertion order |
| `set` | — | O(1) avg | O(1) avg | O(1) avg (`remove`/`discard`) | unordered |
| `frozenset` | — | O(1) avg | — (immutable) | — | unordered |
| `deque` | O(n) middle, O(1) ends | O(n) | O(1) both ends | O(1) both ends | insertion order |
| `heapq` (on list) | O(1) peek min only | O(n) | O(log n) | O(log n) pop-min | heap order only |
| `SortedList` | O(log n) | O(log n) | O(log n) | O(log n) | fully sorted |
| `SortedDict` | O(log n) | O(log n) | O(log n) | O(log n) | sorted by key |
| `SortedSet` | O(log n) | O(log n) | O(log n) | O(log n) | fully sorted |

---

# PART IV — SORTING, SEARCHING, ITERTOOLS AND MATH

## 17. Sorting

| Operation | Signature | Returns | Works on |
|---|---|---|---|
| `sorted(iterable, key=None, reverse=False)` | builtin function | **new** `list` | any iterable (list, tuple, str, dict, set, generator) |
| `list.sort(key=None, reverse=False)` | list method | `None` (in place) | `list` only |

> **Gotcha:** `x = a.sort()` sets `x = None`. `sort()` mutates and returns nothing; `sorted()` returns the result. Chain `sorted(...)` if you need the value, never `.sort()`.

Use `sorted()` when you need a new collection or are sorting a non-list (tuple, generator, dict keys). Use `list.sort()` when you already own the list and want to avoid the O(n) copy — cheaper in CP for large arrays.

### `key=` — Python's comparator model

> **vs C++/JS/Java:** C++ (`std::sort` with a comparator lambda `bool cmp(a,b)`), Java (`Comparator<T>`), and JS (`Array.prototype.sort((a,b)=>...)`) all sort by a **pairwise comparison function**. Python instead sorts by a **key function**: `key(x)` is computed once per element, and elements are ordered by the *natural* ordering of the keys. This is O(n) key computations instead of O(n log n) comparator calls — faster, and it forces you to think in terms of "what value do I want to sort by" rather than "how do I compare two elements."

```python
words = ["banana", "kiwi", "apple"]
print(sorted(words, key=len))
# => ['kiwi', 'apple', 'banana']

nums = [-5, 3, -1, 2]
print(sorted(nums, key=abs))
# => [-1, 2, 3, -5]

pairs = [(3, 'z'), (1, 'a'), (2, 'm')]
print(sorted(pairs, key=lambda x: x[1]))
# => [(1, 'a'), (2, 'm'), (3, 'z')]
```

`reverse=True` reverses the final ordering (NOT the key comparison) — it is not the same as negating a numeric key when there are ties, because it also reverses tie order for non-stable-sensitive cases (stability still holds: equal elements keep original relative order even under `reverse=True`).

```python
print(sorted([3, 1, 2], reverse=True))
# => [3, 2, 1]
```

### Stability (Timsort)

Python's `sorted`/`.sort()` use **Timsort**, which is **guaranteed stable**: elements that compare equal under the key retain their original relative order. This lets you do correct **multi-pass sorts** (sort by secondary key first, then primary key — see below).

```python
data = [("a", 2), ("b", 1), ("c", 2), ("d", 1)]
print(sorted(data, key=lambda x: x[1]))
# => [('b', 1), ('d', 1), ('a', 2), ('c', 2)]
```

> **vs C++:** `std::sort` is **not** guaranteed stable (use `std::stable_sort` explicitly). Python's `sorted`/`.sort()` are always stable — one less thing to worry about.

### Multi-key sort

The idiomatic way is a **tuple key** — Python compares tuples lexicographically for free.

```python
class Person:
    def __init__(self, name, age, score):
        self.name, self.age, self.score = name, age, score
    def __repr__(self):
        return f"{self.name}({self.age},{self.score})"

people = [Person("Bob", 30, 90), Person("Amy", 25, 95), Person("Cy", 25, 80)]

# age ascending, score descending: negate the numeric key
result = sorted(people, key=lambda p: (p.age, -p.score))
print(result)
# => [Amy(25,95), Cy(25,80), Bob(30,90)]
```

**Negation trick** only works for numeric keys. For a **string** key you can't negate it — use one of:
1. Two separate sort criteria expressed differently (reverse the whole thing if ALL keys need reversing).
2. **Two-pass stable sort**: sort by the secondary key with the correct direction first, then stable-sort by the primary key (Timsort's stability preserves the secondary ordering within primary-key ties).

```python
students = [("Bob", "A"), ("Amy", "B"), ("Cy", "A")]
# want: grade ascending, name DESCENDING (name is a string — can't negate)
step1 = sorted(students, key=lambda s: s[0], reverse=True)  # name desc
step2 = sorted(step1, key=lambda s: s[1])                   # grade asc, stable keeps name-desc order
print(step2)
# => [('Cy', 'A'), ('Bob', 'A'), ('Amy', 'B')]
```

### `operator.itemgetter` / `attrgetter` — faster than lambda

A CP micro-optimization: `operator.itemgetter`/`attrgetter` avoid a Python-level function call per comparison and are noticeably faster than an equivalent `lambda` on large inputs.

```python
from operator import itemgetter, attrgetter

pairs = [(1, 'z'), (3, 'a'), (2, 'm')]
print(sorted(pairs, key=itemgetter(1)))
# => [(3, 'a'), (2, 'm'), (1, 'z')]

print(sorted(people, key=attrgetter('age', 'score')))  # multi-key: pass multiple field names
# => [Cy(25,80), Amy(25,95), Bob(30,90)]
```

### `functools.cmp_to_key` — the comparator escape hatch

When ordering genuinely can't be expressed as "extract a key and sort naturally" — e.g. the ordering depends on a pairwise relationship between the two elements themselves — fall back to a C++-style comparator via `cmp_to_key`.

```python
from functools import cmp_to_key

def largest_number(nums):
    """LeetCode 179: arrange nums to form the largest concatenated number."""
    strs = list(map(str, nums))
    def cmp(a, b):
        if a + b > b + a:
            return -1   # a should come before b
        elif a + b < b + a:
            return 1
        return 0
    strs.sort(key=cmp_to_key(cmp))
    return str(int("".join(strs)))

print(largest_number([3, 30, 34, 5, 9]))
# => 9534330
```

> **Gotcha:** `cmp(a, b)` must return negative/zero/positive (like C's `strcmp` / Java's `Comparator.compare`), not a bool. `cmp_to_key` is slower than a `key=` function (O(n log n) actual comparator calls vs O(n) key computations) — only use it when `key=` truly can't express the ordering.

### Sorting tuples, argsort, sorting a dict by value

Tuples sort **lexicographically by default** — no key needed:

```python
print(sorted([(1, 2), (1, 1), (0, 5)]))
# => [(0, 5), (1, 1), (1, 2)]
```

**Argsort** (sort indices by the values they point to) — extremely common in CP:

```python
a = [40, 10, 30, 20]
idx = sorted(range(len(a)), key=lambda i: a[i])
print(idx)
# => [1, 3, 2, 0]
```

Sorting a `dict` by value:

```python
d = {"a": 3, "b": 1, "c": 2}
print(sorted(d.items(), key=lambda kv: kv[1]))
# => [('b', 1), ('c', 2), ('a', 3)]
```

### Pitfalls

```python
# Mixing incomparable key types raises TypeError — Python 3 removed the
# "everything is comparable to everything" fallback that Python 2 had.
try:
    sorted([1, "a", 2])
except TypeError as e:
    print(type(e).__name__)
# => TypeError

# sorted() accepts any iterable, including a generator — but .sort() only
# exists on list, so a generator must be materialized first.
gen = (x * x for x in range(4))
print(sorted(gen))
# => [0, 1, 4, 9]
```

> **Gotcha:** unlike JS's `Array.sort()` (which coerces everything to strings if no comparator is given) or loosely-typed comparisons in some languages, Python raises `TypeError` the moment it tries to compare two elements of incompatible types. Keep sort keys homogeneous in type.

### Timsort complexity

| Case | Time |
|---|---|
| Worst case | O(n log n) |
| Best case (already sorted / reverse-sorted) | O(n) |
| Average | O(n log n) |
| Space | O(n) |

Timsort is **adaptive**: it detects existing runs of ordered data and merges them, so nearly-sorted input sorts close to O(n). This matters in CP when repeatedly re-sorting a mostly-stable array (e.g. after a small perturbation).

---

## 18. `bisect` — Binary Search Built In

> **vs JS:** JavaScript has no built-in binary search / lower-bound at all — you hand-roll it every time. C++ has `std::lower_bound`/`std::upper_bound` (require a sorted range + iterators). Python's `bisect` module gives you both directly on a sorted `list`.

| `bisect` function | Meaning | C++ equivalent |
|---|---|---|
| `bisect_left(a, x)` | first index `i` such that `a[i] >= x` | `std::lower_bound` |
| `bisect_right(a, x)` / `bisect(a, x)` | first index `i` such that `a[i] > x` | `std::upper_bound` |
| `insort_left(a, x)` | insert `x` keeping `a` sorted, before equal elements | — |
| `insort_right(a, x)` / `insort(a, x)` | insert `x` keeping `a` sorted, after equal elements | — |

### Worked table (array WITH duplicates)

`a = [1, 3, 3, 3, 5, 7]`  (indices 0..5)

| `x` | `bisect_left(a, x)` | `bisect_right(a, x)` |
|---|---|---|
| 0 | 0 | 0 |
| 1 | 0 | 1 |
| 3 | 1 | 4 |
| 4 | 4 | 4 |
| 5 | 4 | 5 |
| 8 | 6 | 6 |

```python
import bisect

a = [1, 3, 3, 3, 5, 7]
for x in [0, 1, 3, 4, 5, 8]:
    print(x, bisect.bisect_left(a, x), bisect.bisect_right(a, x))
# => 0 0 0
# => 1 0 1
# => 3 1 4
# => 4 4 4
# => 5 4 5
# => 8 6 6
```

### `insort_left` / `insort_right`

```python
import bisect
a = [1, 3, 3, 5]
bisect.insort_left(a, 3)
print(a)
# => [1, 3, 3, 3, 5]
```

> **Gotcha:** `insort` is O(n) per call (the list shift), even though the search itself is O(log n) — do NOT insort in a loop over n elements expecting O(n log n); that's O(n^2). Use a `SortedList` (see Part III) for repeated inserts into a dynamic sorted sequence.

### `key=` parameter [3.10+]

```python
import bisect
data = [(1, 'a'), (3, 'b'), (5, 'c')]
i = bisect.bisect_left(data, 3, key=lambda t: t[0])   # 3.10+
print(i)
# => 1
```

> **Gotcha:** Pre-3.10, `bisect` has no `key=`. Either bisect on a parallel list of extracted keys (`keys = [t[0] for t in data]; bisect_left(keys, 3)`), or bisect directly on tuples (see below).

### Idioms

**Count of elements in `[lo, hi]` (inclusive)**:

```python
a = [1, 3, 3, 3, 5, 7]
def count_range(a, lo, hi):
    return bisect.bisect_right(a, hi) - bisect.bisect_left(a, lo)
print(count_range(a, 2, 5))
# => 4
```

**Floor (largest `<= x`) / Ceiling (smallest `>= x`)** — always guard the boundary:

```python
def floor(a, x):
    i = bisect.bisect_right(a, x) - 1
    return a[i] if i >= 0 else None      # guard: x smaller than everything

def ceiling(a, x):
    i = bisect.bisect_left(a, x)
    return a[i] if i < len(a) else None  # guard: x larger than everything

a = [1, 3, 3, 5, 7]
print(floor(a, 4), ceiling(a, 4))
# => 3 5
print(floor(a, 0), ceiling(a, 8))
# => None None
```

**Predecessor/successor** (strictly `< x` / strictly `> x`) use the same shape but with `bisect_left(a, x) - 1` (predecessor, strict) and `bisect_right(a, x)` (successor, strict) respectively.

**Binary search on the answer** — `bisect` itself only works on a materialized `list`; when the search space is a monotonic predicate over a numeric range (not backed by a list), hand-roll it:

```python
def min_valid(lo, hi, predicate):
    """Smallest x in [lo, hi] with predicate(x) True, assuming predicate is
    False,False,...,False,True,True,...,True (monotonic)."""
    while lo < hi:
        mid = (lo + hi) // 2
        if predicate(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo

print(min_valid(0, 100, lambda x: x * x >= 50))
# => 8
```

> **Gotcha:** `mid = (lo + hi) // 2` never overflows in Python (arbitrary precision ints) — unlike C++/Java where `(lo+hi)/2` can overflow `int` and requires `lo + (hi-lo)/2`. One less thing to worry about.

**Searching a sorted list of tuples** — bisect compares tuples lexicographically; bisect against a partial tuple to search by the first field only:

```python
data = [(1, 'a'), (3, 'b'), (5, 'c')]
i = bisect.bisect_left(data, (3,))    # shorter tuple with equal prefix sorts "less"
print(i, data[i])
# => 1 (3, 'b')
```

**Exact-match search** (does `x` exist in `a`, and where) — `bisect` finds insertion points, not membership, so always verify:

```python
def find_exact(a, x):
    i = bisect.bisect_left(a, x)
    if i < len(a) and a[i] == x:
        return i
    return -1

a = [1, 3, 3, 5, 7]
print(find_exact(a, 5), find_exact(a, 4))
# => 3 -1
```

> **Gotcha:** `bisect_left(a, x)` returns a valid index (0..len(a)) even when `x` is not in `a` — it never raises or returns a sentinel by itself. Always check `i < len(a) and a[i] == x` before trusting it as a "found" index.

> Cross-ref Part III: for a **dynamically changing** sorted sequence (frequent inserts/deletes, not just point queries) use `sortedcontainers.SortedList`, which keeps insert/delete at O(sqrt n) amortized / O(log n) typical rather than `bisect.insort`'s O(n).

---

## 19. `itertools` and `functools`

### `itertools` — combinatorics and lazy iterators

| Function | Produces | Notes |
|---|---|---|
| `product(a, b, ...)` | cartesian product | `repeat=n` for `product(a, repeat=n)` (n-tuples over `a`, e.g. base-n digits) |
| `permutations(xs, r=None)` | all r-length orderings, no repeats | `r=None` → full-length permutations |
| `combinations(xs, r)` | r-length subsets, order-independent, no repeats | indices, not values, are what stay increasing |
| `combinations_with_replacement(xs, r)` | r-length subsets, repeats allowed | |
| `accumulate(xs, func=operator.add, *, initial=None)` | running fold (prefix sums by default) | `initial=` is [3.8+] |
| `chain(a, b, ...)` | flatten several iterables into one | |
| `chain.from_iterable(iterable_of_iterables)` | same, but takes ONE iterable of iterables | |
| `groupby(iterable, key=None)` | group **consecutive** equal (by key) elements | input MUST be pre-sorted by that key, or you get many small groups |
| `count(start=0, step=1)` | infinite arithmetic sequence | pair with `islice` |
| `cycle(xs)` | infinite repeat of `xs` | pair with `islice` |
| `repeat(x, n)` | `x` repeated n times (or forever if `n` omitted) | |
| `islice(it, [start,] stop[, step])` | slice an iterator (works on infinite iterators) | |
| `takewhile(pred, xs)` | elements until `pred` first False, then stop | |
| `dropwhile(pred, xs)` | skip elements while `pred` True, then yield the rest | |
| `pairwise(xs)` [3.10+] | consecutive `(x[i], x[i+1])` pairs | great for diffs |
| `starmap(func, iterable_of_tuples)` | `func(*args)` for each tuple | like `map` but unpacks args |
| `zip_longest(a, b, fillvalue=None)` | `zip`, padded to the longest iterable | |

```python
from itertools import product, permutations, combinations, combinations_with_replacement

print(list(product([1, 2], ['a', 'b'])))
# => [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

print(list(product([0, 1], repeat=3)))   # all 3-bit binary strings
# => [(0, 0, 0), (0, 0, 1), (0, 1, 0), (0, 1, 1), (1, 0, 0), (1, 0, 1), (1, 1, 0), (1, 1, 1)]

print(list(permutations([1, 2, 3], 2)))
# => [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

print(list(combinations([1, 2, 3], 2)))
# => [(1, 2), (1, 3), (2, 3)]

print(list(combinations_with_replacement([1, 2, 3], 2)))
# => [(1, 1), (1, 2), (1, 3), (2, 2), (2, 3), (3, 3)]
```

```python
from itertools import accumulate

a = [1, 2, 3, 4]
print(list(accumulate(a)))              # prefix sums
# => [1, 3, 6, 10]
print(list(accumulate(a, max)))         # running max
# => [1, 2, 3, 4]
print(list(accumulate(a, initial=100))) # 3.8+: prepend a seed value
# => [100, 101, 103, 106, 110]
```

```python
from itertools import chain

print(list(chain([1, 2], [3, 4], [5])))
# => [1, 2, 3, 4, 5]
print(list(chain.from_iterable([[1, 2], [3], [4, 5]])))
# => [1, 2, 3, 4, 5]
```

```python
from itertools import groupby

data = [1, 1, 2, 2, 3, 1]   # NOT sorted — the trailing 1 forms its OWN group
print([(k, list(g)) for k, g in groupby(data)])
# => [(1, [1, 1]), (2, [2, 2]), (3, [3]), (1, [1])]

data_sorted = sorted(data)
print([(k, list(g)) for k, g in groupby(data_sorted)])
# => [(1, [1, 1, 1]), (2, [2, 2]), (3, [3])]
```

> **Gotcha:** `groupby` only merges **consecutive** matching elements — it is NOT a "group all equal elements" operation like SQL `GROUP BY`. Sort by the key first if you want true grouping. Also, each group `g` is a **lazy iterator that is invalidated** once you advance to the next group — materialize it with `list(g)` immediately if you need to keep it.

```python
from itertools import count, cycle, repeat, islice

print(list(islice(count(10, 2), 5)))       # infinite: 10,12,14,... take first 5
# => [10, 12, 14, 16, 18]
print(list(islice(cycle([1, 2, 3]), 7)))   # infinite repeat, take first 7
# => [1, 2, 3, 1, 2, 3, 1]
print(list(repeat('x', 3)))
# => ['x', 'x', 'x']

print(list(islice(range(20), 5, 15, 3)))   # start, stop, step on an iterator
# => [5, 8, 11, 14]
```

```python
from itertools import takewhile, dropwhile

a = [1, 2, 3, 10, 4, 5]
print(list(takewhile(lambda x: x < 5, a)))
# => [1, 2, 3]
print(list(dropwhile(lambda x: x < 5, a)))
# => [10, 4, 5]
```

> **Gotcha:** `takewhile`/`dropwhile` stop testing at the FIRST element where the predicate flips — they do not filter the whole sequence like `filter` does. `takewhile` above stops at `10` even though `4` and `5` also satisfy `< 5`.

```python
from itertools import pairwise   # 3.10+

a = [1, 3, 6, 10]
print(list(pairwise(a)))
# => [(1, 3), (3, 6), (6, 10)]
diffs = [y - x for x, y in pairwise(a)]
print(diffs)
# => [2, 3, 4]
```

> Pre-3.10 equivalent: `zip(a, a[1:])`.

```python
from itertools import starmap, zip_longest

print(list(starmap(pow, [(2, 3), (3, 2)])))
# => [8, 9]
print(list(zip_longest([1, 2, 3], ['a', 'b'], fillvalue='?')))
# => [(1, 'a'), (2, 'b'), (3, '?')]
```

### Iterators are single-use

Every `itertools` function (and `map`/`filter`/`zip`/`enumerate`/generators) returns a **lazy iterator that is exhausted after one pass**. This is a common source of "empty result" bugs coming from C++/Java, where a range/collection can be iterated repeatedly for free.

```python
from itertools import permutations
perms = permutations([1, 2, 3], 2)
print(len(list(perms)))   # first pass consumes the iterator
# => 6
print(len(list(perms)))   # second pass: already exhausted
# => 0
```

> **Gotcha:** if you need to scan the same combinatorial sequence more than once, materialize it into a `list` first, or regenerate it by calling the itertools function again. `itertools.tee(it, n)` can split ONE iterator into `n` independent ones if you must fan out without regenerating — but each `tee` branch still buffers internally, so it's not free for huge sequences.

### `functools`

**`lru_cache` / `cache`** — the standard DSA memoization tool for top-down DP:

```python
from functools import lru_cache, cache

@lru_cache(maxsize=None)          # maxsize=None => unbounded cache (default for DP)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)
print(fib(30))
# => 832040

@cache                            # 3.9+: shorthand for lru_cache(maxsize=None)
def fact(n):
    return 1 if n == 0 else n * fact(n - 1)
print(fact(10))
# => 3628800
```

> **Gotcha:** all arguments to a cached function must be **hashable** (no `list`/`dict`/`set` args — use `tuple`s instead). `lru_cache` is per-function-object; recursive helper functions defined inside a class method should usually be `@staticmethod` or module-level to cache correctly across calls. Recreating the function (e.g. redefining it inside a loop) resets the cache.

**`reduce(func, iterable, initializer=None)`** — fold left-to-right:

```python
from functools import reduce
import math

xs = [4, 6, 8]
print(reduce(lambda a, b: a ^ b, xs))   # XOR-all (e.g. "find the single number")
# => 10
print(reduce(math.gcd, [12, 18, 24]))   # GCD of a list
# => 6
```

> **vs C++/Java:** `reduce` is Python's `std::accumulate`(C++) / `Stream.reduce`(Java) equivalent — folds are not a builtin operator like they sometimes feel in functional languages; you must `import functools`.

**`cmp_to_key`** — restated from §17: wraps a `(a, b) -> neg/0/pos` comparator into a `key=`-compatible callable for `sorted`/`.sort()`/`heapq` contexts that need one. Use only when a plain key function can't express the ordering.

**`partial(func, *args, **kwargs)`** — pre-bind some arguments:

```python
from functools import partial
add = lambda a, b: a + b
add5 = partial(add, 5)
print(add5(10))
# => 15
```

### Builtins as functional tools

`sum`, `min`, `max`, `any`, `all`, `sorted`, `map`, `filter`, `zip`, `enumerate`, `reversed` all take iterables directly (no explicit loop needed), and several accept `key=`/`default=`:

```python
words = ["kiwi", "banana", "fig"]
print(max(words, key=len))
# => banana
print(min(words, key=len))
# => fig
print(sum([1, 2, 3], 100))          # start= param: sum begins at 100
# => 106
print(any(x > 5 for x in [1, 2, 6]))
# => True
print(all(x > 0 for x in [1, 2, 6]))
# => True
print(max([], default=-1))          # default= avoids ValueError on empty input
# => -1
```

> **Gotcha:** `max([])` / `min([])` on an empty sequence raise `ValueError: max() arg is an empty sequence` — always pass `default=` in CP code where the input could be empty.

---

## 20. Comparators, Keys and Multi-key Ordering

### The mental model

> **vs C++/JS/Java:** those languages sort by a **comparator**: a function `(a, b) -> {negative, 0, positive}` (or a boolean "does a come before b" in JS) that is called O(n log n) times during the sort. Python sorts by a **key function**: `key(x)` is called exactly once per element (O(n) total), producing a value; the final order is the *natural* ascending order of those values. This is why Python's sort is generally faster in practice for equivalent logic, and why "how do I compare two things" (C++/Java/JS mindset) should be reframed as "what value do I want to sort by" in Python.

- Default: **ascending**.
- `reverse=True`: flips the WHOLE final ordering (stability is preserved either way).
- Mixed ascending/descending on numeric fields: **negate** the field you want descending inside the tuple key.
- When no key expresses the ordering (relational/pairwise logic) → escape hatch is `functools.cmp_to_key`.

```python
from functools import cmp_to_key

def cmp(a, b):
    if a + b > b + a:
        return -1
    elif a + b < b + a:
        return 1
    return 0
# sorted(strs, key=cmp_to_key(cmp))   -- restated from §17/§19
```

### Speed: `operator.itemgetter` / `attrgetter`

Prefer these over an equivalent `lambda` in hot sort paths — restated from §17. `itemgetter(i)` for sequences/dicts by index/key, `attrgetter('field')` for objects.

### Making custom objects orderable

So that `sorted`/`heapq`/`min`/`max` work directly on instances without a `key=`:

```python
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __lt__(self, other):
        return (self.x, self.y) < (other.x, other.y)
    def __repr__(self):
        return f"({self.x},{self.y})"

pts = [Point(2, 1), Point(1, 5), Point(1, 2)]
print(sorted(pts))
# => [(1,2), (1,5), (2,1)]
```

Or, more declaratively (cross-ref Part I), `@dataclass(order=True)` auto-generates `__lt__`/`__le__`/`__gt__`/`__ge__` from the field order — exclude non-comparison fields with `field(compare=False)`:

```python
from dataclasses import dataclass, field

@dataclass(order=True)
class Task:
    priority: int
    name: str = field(compare=False)   # excluded from comparisons

tasks = [Task(3, "c"), Task(1, "a"), Task(2, "b")]
print(sorted(tasks))
# => [Task(priority=1, name='a'), Task(priority=2, name='b'), Task(priority=3, name='c')]
```

> **Gotcha:** `heapq` only needs `__lt__` (it never calls `__eq__`/`__gt__`). Defining only `__lt__` (not `functools.total_ordering` or all six dunders) is enough for both `sorted` and `heapq`.

If you need the object to also support `<=`, `>`, `>=` directly (not just `sorted`/`heapq`), either define all of them yourself or use `@functools.total_ordering`, which fills in the rest from `__eq__` + one of `__lt__`/`__le__`/`__gt__`/`__ge__`:

```python
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, major, minor):
        self.major, self.minor = major, minor
    def __eq__(self, other):
        return (self.major, self.minor) == (other.major, other.minor)
    def __lt__(self, other):
        return (self.major, self.minor) < (other.major, other.minor)

v1, v2 = Version(1, 2), Version(1, 5)
print(v1 < v2, v1 <= v2, v1 > v2, v1 >= v2)
# => True True False False
```

### `heapq` tie-break tuple pattern (cross-ref Part III)

When pushing objects that AREN'T comparable (or you want to avoid the cost/risk of comparing them on ties), push `(priority, tiebreak_counter, obj)` — the monotonically increasing counter guarantees no two tuples ever compare equal past the first two fields, so `obj` itself is never compared:

```python
import heapq, itertools

counter = itertools.count()
heap = []
heapq.heappush(heap, (5, next(counter), "task-A"))
heapq.heappush(heap, (5, next(counter), "task-B"))
heapq.heappush(heap, (1, next(counter), "task-C"))
print(heapq.heappop(heap))
# => (1, 2, 'task-C')
print(heapq.heappop(heap))
# => (5, 0, 'task-A')
```

### Summary table

| Goal | Idiom |
|---|---|
| Sort ascending | `sorted(a)` |
| Sort descending | `sorted(a, reverse=True)` |
| Sort by a derived value | `sorted(a, key=f)` |
| Sort by multiple fields, all same direction | `sorted(a, key=lambda x: (x.k1, x.k2))` (+`reverse=True` if all descending) |
| Sort by multiple fields, mixed direction (numeric) | `sorted(a, key=lambda x: (x.k1, -x.k2))` |
| Sort by multiple fields, mixed direction (string field) | two-pass stable sort (§17) |
| Sort with true pairwise/relational logic | `sorted(a, key=cmp_to_key(cmp))` |
| argmin / argmax | `min(range(n), key=lambda i: a[i])` / `max(...)` |

```python
a = [40, 10, 30, 20]
print(max(range(len(a)), key=lambda i: a[i]))  # argmax
# => 0
print(min(range(len(a)), key=lambda i: a[i]))  # argmin
# => 1
```

---

## 21. Math and Number Theory Stdlib

| Function | Signature | Notes |
|---|---|---|
| `math.gcd(*xs)` | GCD | [3.9+] variadic (2+ args); pre-3.9 only `gcd(a, b)` |
| `math.lcm(*xs)` | LCM | [3.9+] |
| `math.isqrt(n)` | exact integer floor sqrt | no float rounding error — the correct way for large `n` |
| `math.comb(n, k)` | nCr, exact int | [3.8+] |
| `math.perm(n, k=None)` | nPr, exact int | [3.8+] |
| `math.factorial(n)` | n! | exact int |
| `math.floor(x)` / `ceil(x)` / `trunc(x)` | int rounding | `trunc` truncates toward 0 |
| `math.inf`, `-math.inf`, `math.nan` | float constants | `+inf`, `-inf`, NaN |
| `math.isclose(a, b, rel_tol=1e-9)` | float equality | the correct way to compare floats |
| `math.log(x, base)` / `log2` / `log10` | logarithms | |
| `math.sqrt(x)` | float sqrt | returns `float`, may lose precision for huge ints — prefer `isqrt` |
| `math.pow(x, y)` | float power | ALWAYS returns `float`, even for int args |
| `math.hypot(x, y, ...)` | Euclidean norm | |
| `math.dist(p, q)` | Euclidean distance between points (tuples) | |
| `math.prod(xs, start=1)` | product of iterable | [3.8+] |
| `pow(a, b, m)` | builtin, modular exponentiation | O(log b), no need to hand-roll fast-pow |
| `pow(a, -1, m)` | builtin, modular inverse | [3.8+], requires `gcd(a, m) == 1` |
| `divmod(a, b)` | `(a // b, a % b)` in one call | |
| `abs(x)` | absolute value | works on int/float/complex |

### `isqrt` vs `sqrt` — the exact-vs-float trap

```python
import math
n = 50
print(math.isqrt(n))          # exact integer floor sqrt — always correct
# => 7
print(int(math.sqrt(n)))      # float sqrt then truncate — can be off-by-one for large n
# => 7
```

> **Gotcha:** For large `n` (beyond `float`'s 53-bit mantissa precision, roughly `n > 2**53`), `math.sqrt(n)` can round incorrectly and `int(math.sqrt(n))` can be off by one. `math.isqrt` works on arbitrary-precision ints and is always exact — always prefer it for "is n a perfect square" or integer sqrt in CP.

```python
import math
print(math.isqrt(n) ** 2 == n)   # perfect-square check, exact
```

### `comb` / `perm` / `factorial`

```python
import math
print(math.comb(5, 2))     # nCr, exact — no manual factorial formula needed
# => 10
print(math.perm(5, 2))     # nPr, exact
# => 20
print(math.factorial(5))
# => 120
```

> **vs C++/Java:** no stdlib `nCr`/`nPr` exists in C++ or Java — you hand-roll factorials or Pascal's triangle. Python's `math.comb`/`math.perm` [3.8+] give you exact big-int results directly; watch out that they are exact but NOT modular — for `nCr mod p` with huge `n`, you still need a precomputed-factorial + modular-inverse approach (see below), since `math.comb` computes the full (possibly huge) integer first.

### floor / ceil / trunc, inf/nan, isclose

```python
import math
print(math.floor(3.7), math.ceil(3.2), math.trunc(-3.7))
# => 3 4 -3

print(math.inf, -math.inf, math.nan)
# => inf -inf nan

print(math.isclose(0.1 + 0.2, 0.3))
# => True
print(0.1 + 0.2 == 0.3)
# => False
```

> **Gotcha:** never compare floats with `==` — binary floating point can't represent most decimals exactly. Use `math.isclose(a, b, rel_tol=..., abs_tol=...)`.

### log / sqrt / pow / hypot / dist / prod

```python
import math
print(math.log(8, 2), math.log2(8), math.log10(1000))
# => 3.0 3.0 3.0
print(math.sqrt(16))
# => 4.0
print(math.pow(2, 10))    # ALWAYS float, even though inputs are ints
# => 1024.0
print(2 ** 10)             # int ** int stays an exact int — prefer this in CP
# => 1024
print(math.hypot(3, 4))
# => 5.0
print(math.dist((0, 0), (3, 4)))
# => 5.0
print(math.prod([1, 2, 3, 4]))   # 3.8+
# => 24
```

> **Gotcha:** `math.pow` returns `float` unconditionally — for exact integer exponentiation (e.g. `2**60`, which exceeds float precision) always use the `**` operator or the 3-arg `pow(a, b, m)`, never `math.pow`.

### `pow(a, b, m)` and `pow(a, -1, m)` — CP essentials built in

```python
print(pow(2, 10, 1000))    # modular exponentiation, O(log b) — no hand-rolled fast-pow needed
# => 24
print(pow(3, -1, 7))       # modular inverse, [3.8+] — huge CP win, no extended-Euclid needed
# => 5
print((3 * 5) % 7)         # verify: 3 * inverse(3) ≡ 1 (mod 7)
# => 1
```

> **Gotcha:** `pow(a, -1, m)` requires `math.gcd(a, m) == 1` (a must be invertible mod m), else it raises `ValueError`. Needs Python **3.8+**; before that, compute the inverse via `pow(a, m-2, m)` (Fermat's little theorem, only valid when `m` is prime) or extended Euclid.

### `divmod`, `abs`

```python
print(divmod(17, 5))
# => (3, 2)
print(abs(-7), abs(3 + 4j))
# => 7 5.0
```

### Modular arithmetic patterns

```python
MOD = 10**9 + 7
a, b = 123456789, 987654321
print((a * b) % MOD)     # never overflows — Python ints are arbitrary precision
# => 259106859
```

> **vs C++/Java/JS:** in C++/Java, `int64` multiplication of two large values can silently overflow before the `% MOD` is applied, forcing `__int128`/`long` tricks or a mulmod helper; JS numbers lose integer precision past 2^53 and need `BigInt`. In Python, **there is no overflow** — `a * b` is always exact regardless of magnitude, so `(a * b) % MOD` is always safe as written. This is one of the biggest quality-of-life differences doing CP in Python.

**GCD/LCM idioms:**

```python
import math
from functools import reduce
print(math.gcd(12, 18, 24))              # 3.9+ variadic
# => 6
print(reduce(math.gcd, [12, 18, 24]))    # pre-3.9 fallback / explicit fold
# => 6
print(math.lcm(4, 6))
# => 12
```

**`nCr mod p`** (for large `n` where `math.comb` followed by `% p` would compute a huge exact int first — still correct, but wasteful/slow for many queries): precompute factorials and use the modular inverse via Fermat's little theorem when `p` is prime:

```python
MOD = 10**9 + 7
N = 200
fact = [1] * (N + 1)
for i in range(1, N + 1):
    fact[i] = fact[i - 1] * i % MOD

def nCr_mod(n, r, mod=MOD):
    if r < 0 or r > n:
        return 0
    return fact[n] * pow(fact[r], mod - 2, mod) * pow(fact[n - r], mod - 2, mod) % mod

print(nCr_mod(10, 3))
# => 120
```

> `pow(fact[r], mod - 2, mod)` is Fermat's little theorem (`x^(p-1) ≡ 1 mod p` for prime `p`, so `x^(p-2)` is `x`'s inverse) — this only works when `mod` is **prime**. For a non-prime modulus, use `pow(x, -1, mod)` [3.8+] instead (works whenever `gcd(x, mod) == 1`, prime or not).

### `%` sign behavior (a recurring CP trap)

```python
print(-7 % 3)      # Python: result has the SIGN OF THE DIVISOR, always non-negative here
# => 2
print(7 % -3)
# => -2
print(math.fmod(-7, 3))   # C-style fmod: sign of the DIVIDEND, matches C++/Java/JS `%`
# => -1.0
```

> **vs C++/Java/JS:** in C++, Java, and JS, `%` is a remainder operator whose result takes the sign of the **dividend** (so `-7 % 3 == -1` in those languages). Python's `%` is a true **modulo** whose result takes the sign of the **divisor**, so `-7 % 3 == 2`. This is usually what you WANT for CP (e.g. wrapping array indices with `i % n` always lands in `[0, n)` even for negative `i`), but it is a frequent source of off-by-sign bugs when porting a formula from C++. Use `math.fmod` if you specifically need C-style truncating remainder semantics.

### Forward references

- `decimal.Decimal` — arbitrary-precision fixed-point decimals (avoids float rounding entirely) — see Part VI.
- `fractions.Fraction` — exact rational arithmetic — see Part VI.

### `sum` with `start`

```python
print(sum([1, 2, 3], 100))   # start= adds a base value (also works for non-numeric with __add__)
# => 106
```

---

# PART V — BIT MANIPULATION

## 22. Bitwise Operators and the Arbitrary-Precision Reality

### 22.1 The operators

| Op | Name | Example | Result | Notes |
|----|------|---------|--------|-------|
| `&` | AND | `6 & 3` | `2` | `0b110 & 0b011 = 0b010` |
| `\|` | OR | `6 \| 3` | `7` | `0b110 \| 0b011 = 0b111` |
| `^` | XOR | `6 ^ 3` | `5` | `0b110 ^ 0b011 = 0b101` |
| `~` | NOT | `~6` | `-7` | `~x == -(x+1)`, see §22.3 |
| `<<` | left shift | `6 << 1` | `12` | grows the int, never overflows |
| `>>` | right shift | `6 >> 1` | `3` | arithmetic (sign-preserving) shift |

Truth table for a single bit pair `(a, b)`:

| a | b | a&b | a\|b | a^b |
|---|---|-----|------|-----|
| 0 | 0 | 0 | 0 | 0 |
| 0 | 1 | 0 | 1 | 1 |
| 1 | 0 | 0 | 1 | 1 |
| 1 | 1 | 1 | 1 | 0 |

Worked example with `bin()`:

```python
a, b = 0b1100, 0b1010
print(bin(a), bin(b))          # => 0b1100 0b1010
print(bin(a & b))              # => 0b1000
print(bin(a | b))              # => 0b1110
print(bin(a ^ b))              # => 0b110
print(bin(a << 2))             # => 0b110000  (multiply by 4)
print(bin(a >> 2))             # => 0b11      (divide by 4, floor)
```

### 22.2 THE Python bit model — read this before anything else

Python `int` is **arbitrary precision**. There is no `int32`/`int64`, no fixed word size, and — critically —
**no unsigned integer type at all**. Every int is conceptually a signed value in an *infinite* two's-complement
representation:

- A non-negative int is `...0000` followed by its bits (infinitely many leading zeros, conceptually).
- A negative int is `...1111` followed by its bits (infinitely many leading ones, conceptually) — i.e. `-1` is
  "all bits set", forever, in both directions.

This is fundamentally different from every other mainstream language:

> **vs C++:** `int`/`long` are fixed-width (32/64-bit) and signed overflow is UB; `unsigned` types wrap
> modulo `2^N` and support `>>` as a *logical* (zero-fill) shift.
> **vs Java:** `int`/`long` are fixed 32/64-bit two's-complement with defined wraparound; Java has `>>>` for
> unsigned (logical) right shift because it needs one.
> **vs JS:** bitwise ops (`&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`) coerce operands to **32-bit signed** (or
> unsigned for `>>>`) before operating, then convert back to `Number` — so JS bit tricks silently truncate
> at 32 bits even though `Number` itself is a 64-bit float.
>
> Python has **none** of these limits — no wraparound, no fixed width, no overflow. That's a relief (you
> never accidentally overflow a running XOR / mask) but it is also a *difference* you must account for:
> problems that explicitly want 32-bit wraparound semantics (LeetCode "Reverse Bits", "Sum of Two Integers",
> "Divide Two Integers") need you to *simulate* it manually — see §22.5.

### 22.3 `~x == -(x + 1)` — THE #1 Python bit gotcha

Bitwise NOT flips every bit of the infinite two's-complement representation. Flipping `...0000_0101` (5) gives
`...1111_1010`, which in two's complement is `-6`. In general:

```python
x = 5
print(~x)          # => -6
print(~x == -(x + 1))   # => True

print(~0)           # => -1
print(~-1)          # => 0
print(~-6)           # => 5   (NOT is its own inverse)
```

The `bin()` output on a negative number is deceptive — Python prints a **minus sign plus the magnitude's
binary**, NOT a two's-complement bit pattern:

```python
print(bin(~5))      # => -0b110   (this is "-(0b110)", i.e. -6 — NOT a bit pattern you can read bits off of)
print(bin(5))        # => 0b101
```

> **Gotcha:** `bin(-6)` does **not** show you infinite leading 1s — it shows `-0b110`, the sign plus magnitude
> of 6. If you need to *see* the two's-complement bit pattern of a negative number (e.g. for a "reverse bits"
> style problem), you must mask to a fixed width first: `bin(-6 & 0xFF)` → `'0b11111010'` (8-bit view).

### 22.4 Negative numbers behave as infinitely sign-extended

Because of the model above, `&`, `|`, `^`, and `>>` on negative operands act as if there were infinitely many
1-bits (for negatives) or 0-bits (for non-negatives) extending to the left forever.

```python
print(bin(-1))        # => -0b1        (but conceptually: ...1111, all ones)
print(-1 & 0b1010)      # => 10   (every low bit of -1 reads as 1, so it's a no-op AND)
print(-1 | 0)           # => -1
print(-1 >> 1)           # => -1   (arithmetic shift: sign-extends forever, never becomes 0)
print(-1 >> 100)         # => -1   (still -1, no matter how far you shift)
print(-4 >> 1)           # => -2   (floor division by 2)
print(-3 >> 1)           # => -2   (NOT -1: floor(-3/2) = -2, not truncation toward zero)
```

> **Gotcha:** `>>` on a negative int is **floor** shifting (matches `//`), not truncating division. `-3 >> 1
> == -2`, whereas in C++/Java, `-3 >> 1` on a fixed-width signed int is implementation-historically also
> arithmetic-shift-floor (`-2`) for those languages too — but the point to remember for Python specifically is
> there's no bit-pattern floor to hit; it just keeps producing `-1` for an all-negative shift instead of ever
> settling to `0` the way an unsigned language would with `>>>`.

### 22.5 `<<` never overflows — Python's superpower for wide masks

```python
x = 1 << 1000              # a 1001-bit integer. Just works.
print(x.bit_length())      # => 1001
print(x >> 999)             # => 2
```

> **vs C++/Java/JS:** `1 << 1000` is undefined behavior / a compile error / silently wrong in C++
> (shift amount ≥ width is UB), overflows in Java (ints/longs wrap at 32/64 bits), and in JS coerces to a
> 32-bit shift amount (`1000 & 31 == 40 & 31 == 8`, so `1 << 1000 === 256`, wildly wrong).
> In Python, `<<` simply grows the integer — there is no width to exceed. This is *huge* for bitmask-DP
> and "bitset" tricks over more than 63 items (see §24.3).

Because Python ints can't overflow, there is also **no `>>>` operator and none is needed** — since there's no
fixed width and no unsigned type, "logical shift" and "arithmetic shift" would be indistinguishable for
non-negatives, and for negatives Python simply doesn't offer an unsigned interpretation at all.

### 22.6 Simulating fixed-width unsigned behavior when a problem demands it

Some LeetCode/CP problems are specified in terms of 32-bit (or 64-bit) two's-complement wraparound — "Reverse
Bits", "Number of 1 Bits" (edge case with negative input in other languages), "Sum of Two Integers without
`+`", "Divide Two Integers". Since Python has no such wraparound natively, mask manually:

```python
MASK32 = 0xFFFFFFFF          # (1 << 32) - 1
SIGN32 = 0x80000000          # 1 << 31

def to_uint32(x: int) -> int:
    return x & MASK32

def to_int32(x: int) -> int:
    x &= MASK32
    return x - (1 << 32) if x & SIGN32 else x

# Example: simulate 32-bit overflow addition
def add_32bit(a: int, b: int) -> int:
    return to_int32(a + b)

print(to_uint32(-1))          # => 4294967295   (all 32 bits set, interpreted unsigned)
print(to_int32(0xFFFFFFFF))    # => -1           (all bits set, interpreted signed)
print(add_32bit(2**31 - 1, 1))  # => -2147483648  (classic signed overflow wraparound)
```

The idiom in two lines: **mask with `& 0xFFFFFFFF` to force a 32-bit unsigned view; then, if you need the
signed interpretation, subtract `1 << 32` when the sign bit (`0x80000000`) is set.** Same pattern with
`0xFFFFFFFFFFFFFFFF` / `1 << 63` for 64-bit.

### 22.7 Operator precedence trap

`&`, `|`, `^` bind **looser** than comparison operators (`==`, `<`, `>`, etc.) in Python — the opposite of what
your C++/Java/JS muscle memory expects for `if x & 1 == 0`. Comparisons happen first, so this is a silent bug:

```python
x = 4
# WRONG — parses as: x & (1 == 0)  →  x & False  →  x & 0  →  0  (falsy, looks "even", coincidentally works here)
print(bool(x & 1 == 0))     # => False   (accidentally "correct" for x=4, but for the WRONG reason)

x = 5
print(bool(x & 1 == 0))     # => False   (x=5 is odd, so this SHOULD be False — but again for the wrong reason:
                              #            it parsed as x & (1==0) = x & False = 0 = False, not "5 & 1 == 0")

# CORRECT — always parenthesize the bitwise sub-expression:
print((x & 1) == 0)         # => False  (correct, for the right reason)
```

> **Gotcha:** `&`/`|`/`^` precedence is lower than `==`/`<`/`>`/`in`/`is`, and also lower than `+`/`-`. Always
> wrap the bitwise expression in parens when mixing with comparisons: `if (x & mask) == target:`. This bites
> everyone coming from C++/Java/JS, where `&` binds tighter than `==` is *also* true there too, actually — so
> the trap isn't relative precedence direction differing, it's that Python programmers (especially from a
> C-family background) simply forget it applies in Python exactly as in C, and skip the parens out of habit
> from higher-precedence contexts like `if (x & 1)`. Parenthesize on sight, every time.

---

## 23. The Complete Bit-Trick Catalog

Throughout, `i` is a 0-indexed bit position (bit 0 = LSB = value `1`).

### 23.1 Test / set / clear / toggle / extract a single bit

```python
x = 0b1010          # 10

# test bit i (is it 1?)
def test_bit(x, i):
    return (x >> i) & 1

print(test_bit(x, 1))     # => 1   (bit 1 is set)
print(test_bit(x, 0))     # => 0   (bit 0 is clear)

# set bit i (force to 1)
def set_bit(x, i):
    return x | (1 << i)

print(bin(set_bit(x, 0)))  # => 0b1011

# clear bit i (force to 0)
def clear_bit(x, i):
    return x & ~(1 << i)

print(bin(clear_bit(x, 1)))  # => 0b1000

# toggle bit i (flip it)
def toggle_bit(x, i):
    return x ^ (1 << i)

print(bin(toggle_bit(x, 0)))  # => 0b1011
print(bin(toggle_bit(x, 1)))  # => 0b1000
```

### 23.2 Odd/even, divide/multiply by powers of two

```python
x = 13
print(x & 1)          # => 1   (odd; 0 means even)
print((-4) & 1)         # => 0   (even; works for negatives too, thanks to sign-extension model)

print(x >> 1)           # => 6   (floor(13 / 2))
print(x << 2)           # => 52  (13 * 4)
print(-13 >> 1)          # => -7  (floor(-13 / 2) = -6.5 -> -7, NOT truncation toward zero!)
```

> **Gotcha:** `x >> k` is `x // (1 << k)` (floor division), not truncating division. For negative `x` this
> differs from C++'s "round toward zero" integer division and from a naive `x / 2**k`. If you need
> truncation-toward-zero semantics for negatives, use `int(x / 2**k)` or `-(-x >> k)` explicitly.

### 23.3 `x & (x - 1)` — clear the lowest set bit

Subtracting 1 flips the lowest set bit to 0 and every bit below it to 1; ANDing with the original clears the
lowest set bit and leaves everything else untouched.

```python
x = 0b10110000
print(bin(x & (x - 1)))    # => 0b10100000   (lowest set bit, position 4, cleared)
```

**Brian Kernighan's popcount** — repeatedly clear the lowest set bit and count iterations (runs in
`O(popcount)` instead of `O(bit_length)`):

```python
def popcount_bk(x: int) -> int:
    count = 0
    while x:
        x &= x - 1
        count += 1
    return count

print(popcount_bk(0b10110000))  # => 3
```

(In modern Python just use `x.bit_count()` — §24.1 — but Brian Kernighan's loop is worth knowing for
interviews/CP judges without 3.10, and it generalizes to "iterate set bits" — see §23.13.)

### 23.4 `x & (-x)` — isolate the lowest set bit

`-x` is `~x + 1` (two's complement negation), which flips every bit below the lowest set bit and keeps that
bit and everything above flipped too; ANDing with `x` leaves only the lowest set bit standing.

```python
x = 0b10110000
print(bin(x & -x))     # => 0b10000    (16 — just the lowest set bit, as a power of two)
```

This works identically for arbitrarily large `x` — no width limit:

```python
big = (1 << 200) | (1 << 5)
print((big & -big) == (1 << 5))   # => True
```

### 23.5 Power-of-two check

A power of two has exactly one bit set, so clearing its lowest set bit yields zero:

```python
def is_power_of_two(x: int) -> bool:
    return x > 0 and (x & (x - 1)) == 0

print(is_power_of_two(64))   # => True
print(is_power_of_two(63))   # => False
print(is_power_of_two(0))    # => False  (0 has no set bit at all — must exclude explicitly)
```

> **Gotcha:** the `x > 0` guard is required. Without it, `x = 0` passes `(0 & -1) == 0` as `True` incorrectly,
> and negative `x` (infinite leading 1-bits) would also misbehave.

### 23.6 XOR swap and XOR properties

XOR swap (no temp variable — mostly a curiosity in Python since tuple-swap `a, b = b, a` is idiomatic and
faster; know it for C-style problems / interview trivia):

```python
a, b = 5, 9
a ^= b
b ^= a
a ^= b
print(a, b)     # => 9 5
```

Core XOR identities: `x ^ x == 0`, `x ^ 0 == x`, XOR is commutative and associative. These power a whole
family of tricks:

**Single Number** — every element appears twice except one; XOR of all elements leaves only the odd one:

```python
def single_number(nums: list[int]) -> int:
    result = 0
    for n in nums:
        result ^= n
    return result

print(single_number([4, 1, 2, 1, 2]))   # => 4
```

**Missing Number** — array of `0..n` with one missing; XOR the full range against the array:

```python
def missing_number(nums: list[int]) -> int:
    n = len(nums)
    result = n
    for i, v in enumerate(nums):
        result ^= i ^ v
    return result

print(missing_number([3, 0, 1]))   # => 2
```

**Two Single Numbers** ("Single Number III") — every element appears twice except *two*; XOR everything to get
`a ^ b`, then split the array by any bit where `a` and `b` differ (the lowest set bit of `a ^ b` is guaranteed
to be such a bit):

```python
def two_single_numbers(nums: list[int]) -> tuple[int, int]:
    xor_all = 0
    for n in nums:
        xor_all ^= n
    diff_bit = xor_all & -xor_all           # lowest set bit where a, b differ
    a = b = 0
    for n in nums:
        if n & diff_bit:
            a ^= n
        else:
            b ^= n
    return a, b

print(two_single_numbers([1, 2, 1, 3, 2, 5]))   # => (3, 5)
```

**XOR of range `[0, n]`** — closed form using `n % 4`, avoids an `O(n)` loop:

```python
def xor_0_to_n(n: int) -> int:
    r = n % 4
    if r == 0:
        return n
    if r == 1:
        return 1
    if r == 2:
        return n + 1
    return 0   # r == 3

print(xor_0_to_n(5))    # => 1   (0^1^2^3^4^5 = 1)

def xor_range(lo: int, hi: int) -> int:
    """XOR of [lo, hi] inclusive."""
    return xor_0_to_n(hi) ^ xor_0_to_n(lo - 1)

print(xor_range(3, 5))   # => 3   (3^4^5 = 3)
```

### 23.7 Masks `(1 << k) - 1`

A mask of the low `k` bits set:

```python
k = 5
mask = (1 << k) - 1
print(bin(mask))    # => 0b11111

print(0b110110 & mask)   # => 0b10110  (keep only the low 5 bits)
```

> **vs C++/JS:** in C++, `(1 << 31) - 1` or `(1LL << 63) - 1` are the *widest* masks you can build with native
> types before hitting UB/overflow; in JS, `(1 << k) - 1` breaks down once `k >= 31` because `<<` coerces to
> 32-bit. In Python, `k` can be **any size** — `(1 << 500) - 1` is a perfectly good 500-bit mask, built and
> used exactly like a small one. This removes an entire class of "which width do I need" bookkeeping.

### 23.8 Turn on/off bits below/above `i`; extract a bit-field

```python
i = 4
x = 0b11111111

# clear all bits below i (keep bit i and above)
below_cleared = x & ~((1 << i) - 1)
print(bin(below_cleared))     # => 0b11110000

# clear all bits from i upward (keep bits strictly below i)
above_cleared = x & ((1 << i) - 1)
print(bin(above_cleared))     # => 0b1111

# set all bits below i
turn_on_below = x | ((1 << i) - 1)
print(bin(turn_on_below))     # => 0b11111111  (already all set here; illustrative)

# extract bit-field [lo, hi] inclusive (hi >= lo), right-aligned
def extract_field(x: int, lo: int, hi: int) -> int:
    width = hi - lo + 1
    return (x >> lo) & ((1 << width) - 1)

print(bin(extract_field(0b110101100, 2, 5)))   # => 0b1011  (bits 2..5 of 0b110101100)
```

### 23.9 Highest/lowest set bit position, floor(log2), next/prev power of two

```python
x = 0b0101_0000    # 80

# highest set bit position == floor(log2(x)) for x > 0
def highest_set_bit_pos(x: int) -> int:
    return x.bit_length() - 1

print(highest_set_bit_pos(80))   # => 6   (2**6 = 64 <= 80 < 128 = 2**7)

# lowest set bit position
def lowest_set_bit_pos(x: int) -> int:
    return (x & -x).bit_length() - 1

print(lowest_set_bit_pos(80))    # => 4   (80 = 0b1010000, lowest set bit is bit 4)

# next power of two >= x (x > 0)
def next_pow2(x: int) -> int:
    if x & (x - 1) == 0:
        return x        # already a power of two (or x could be 0, treat separately if needed)
    return 1 << x.bit_length()

print(next_pow2(80))    # => 128
print(next_pow2(64))    # => 64   (already a power of two)

# previous (largest) power of two <= x (x > 0)
def prev_pow2(x: int) -> int:
    return 1 << (x.bit_length() - 1)

print(prev_pow2(80))    # => 64
```

`int.bit_length()` is the clean built-in replacement for C's `__builtin_clz`/`clzll` — no separate
"count leading zeros" intrinsic needed since Python has no fixed width to count zeros *from*.

### 23.10 Count set bits (popcount)

```python
x = 0b10110110

# 3.10+ built-in — the fast, correct way
print(x.bit_count())          # => 5

# portable fallback — works on any Python 3.x, any CPython/PyPy version
print(bin(x).count('1'))       # => 5
```

> **Gotcha:** `int.bit_count()` requires **Python 3.10+**. Older Codeforces judges / some legacy graders may
> still run 3.8, where this raises `AttributeError`. When unsure of the judge version, use
> `bin(x).count('1')` — slightly slower (string build) but works everywhere, including negative `x` treated by
> magnitude (`bin(-5).count('1')` counts bits in `5`, not the infinite-1s representation — usually not what
> you want for negatives, so popcount is normally only meaningful for `x >= 0`).

### 23.11 Reverse bits

Python ints have no fixed width, so "reverse the bits" only makes sense once you pick a width (commonly 32,
matching the LeetCode "Reverse Bits" problem):

```python
def reverse_bits_32(x: int) -> int:
    return int(format(x, '032b')[::-1], 2)

print(reverse_bits_32(0b00000010100101000001111010011100))
# => 964176192  (0b00111001111000010100101000000100)

# loop version (no string building) — also useful when width isn't a round number
def reverse_bits_loop(x: int, width: int = 32) -> int:
    result = 0
    for _ in range(width):
        result = (result << 1) | (x & 1)
        x >>= 1
    return result

print(reverse_bits_loop(0b1011, width=8))   # => 0b11010000 => 208
```

### 23.12 Subset enumeration over a bitmask

**Enumerate all subsets of a given mask `m`** (a classic, non-obvious idiom — runs in `O(3^n)` total across all
masks of an `n`-bit universe, since each of the `n` bits is either "not in `m`" (1 way), "in `m` and in the
subset" (1 way), or "in `m` but not in the subset" (1 way) — hence `3^n`, not `4^n`):

```python
def subsets_of_mask(m: int):
    sub = m
    while True:
        yield sub
        if sub == 0:
            break
        sub = (sub - 1) & m

print(list(subsets_of_mask(0b101)))   # => [5, 4, 1, 0]   (bits: {0,2}, {2}, {0}, {})
```

> **Gotcha:** the empty subset (`0`) IS included and the loop must terminate right after yielding it — if you
> instead write `while sub: ... sub = (sub-1) & m`, you silently skip the empty subset. Decide up front
> whether your problem wants it (often it does, e.g. "sum over all subsets including empty").

**Iterate all masks over `n` items** (the full power set, `2^n` masks):

```python
n = 3
for m in range(1 << n):
    print(bin(m))
# => 0b0, 0b1, 0b10, 0b11, 0b100, 0b101, 0b110, 0b111
```

**Iterate set bits of a mask** (extract each `1`-bit's position, low to high), two equivalent styles:

```python
m = 0b10110

# style 1: peel off the lowest set bit each time (Brian Kernighan-style)
positions = []
mm = m
while mm:
    b = mm & -mm                  # isolate lowest set bit
    positions.append(b.bit_length() - 1)
    mm ^= b                       # clear it (equivalent to mm &= mm - 1)
print(positions)    # => [1, 2, 4]

# style 2: test every candidate position directly
positions2 = [i for i in range(m.bit_length()) if (m >> i) & 1]
print(positions2)   # => [1, 2, 4]
```

> **Performance note:** for small/medium `n` (say up to a few thousand), a single big int as a bitmask is fast
> and memory-tight (CPython packs bits densely — 30 bits per internal "digit"). For very large `n` (millions)
> where you mostly need *membership tests and iteration* rather than dense bit-twiddling, a `set` or a
> `list`/`bytearray` of booleans can outperform a giant int, because operations like `(mask - 1) & mask` on a
> million-bit Python int still cost `O(n / 30)` machine words per op — linear in size, same asymptotics as a
> byte array, but with more per-operation overhead from Python's bigint machinery. Profile if it matters; see
> the decision table in §24.4.

### 23.13 Counting-bits DP

The classic "for every `i` in `0..n`, compute popcount(i)" done in `O(n)` total via DP, reusing already-computed
smaller results — `dp[i] = dp[i >> 1] + (i & 1)` (drop the lowest bit via right shift, add back whether that
dropped bit was 1):

```python
def count_bits(n: int) -> list[int]:
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i >> 1] + (i & 1)
    return dp

print(count_bits(7))   # => [0, 1, 1, 2, 1, 2, 2, 3]
```

(This is LeetCode "Counting Bits" — trivially replaceable by `[x.bit_count() for x in range(n+1)]` on 3.10+,
but the DP recurrence itself is a reusable pattern worth knowing since it generalizes to other per-bit DPs.)

---

## 24. `int` Bit Methods, Wide Masks, and Byte Conversion

### 24.1 The `int` method reference

| Method / builtin | Meaning | Example | Result | Version |
|---|---|---|---|---|
| `x.bit_length()` | position of highest set bit + 1 (bits needed to represent \|x\|, excluding sign) | `(13).bit_length()` | `4` | all |
| `x.bit_count()` | population count (number of set bits, by magnitude) | `(13).bit_count()` | `3` | **3.10+** |
| `x.to_bytes(length, byteorder, *, signed=False)` | pack int into a fixed-length `bytes` object | `(1024).to_bytes(2, 'big')` | `b'\x04\x00'` | all |
| `int.from_bytes(bytes, byteorder, *, signed=False)` | unpack `bytes` back into an int | `int.from_bytes(b'\x04\x00', 'big')` | `1024` | all |
| `bin(x)` | binary string, prefixed `'0b'` (or `'-0b'`) | `bin(10)` | `'0b1010'` | all |
| `hex(x)` | hex string, prefixed `'0x'` | `hex(255)` | `'0xff'` | all |
| `oct(x)` | octal string, prefixed `'0o'` | `oct(8)` | `'0o10'` | all |
| `format(x, 'b')` | binary string, no prefix | `format(10, 'b')` | `'1010'` | all |
| `format(x, '08b')` | binary, zero-padded to width 8 | `format(10, '08b')` | `'00001010'` | all |
| `int(s, 2)` | parse a binary string | `int('1010', 2)` | `10` | all |

```python
print((13).bit_length())     # => 4     (13 = 0b1101, needs 4 bits)
print((0).bit_length())       # => 0     (special case: zero needs zero bits)
print((-13).bit_length())     # => 4     (bit_length ignores sign, uses magnitude)

print((13).bit_count())       # => 3     (0b1101 has three 1-bits)   [3.10+]
```

**Log2 via `bit_length`:** `x.bit_length() - 1` gives `floor(log2(x))` for `x > 0` — exact integer arithmetic,
no floating-point `math.log2` precision worries (which can be off-by-one near powers of two due to float
rounding):

```python
import math
x = 2**53 + 1
print(x.bit_length() - 1)     # => 53   (exact)
print(int(math.log2(x)))       # => 53 or 54 depending on float rounding — RISKY near large powers of two
```

**Stripping the `'0b'` prefix and zero-padding:**

```python
x = 10
print(bin(x))          # => 0b1010
print(bin(x)[2:])       # => 1010          (slice off the prefix)
print(format(x, 'b'))    # => 1010          (cleaner — no slicing needed)
print(format(x, '08b'))  # => 00001010      (zero-padded to 8 bits — handy for fixed-width display)
print(f'{x:08b}')        # => 00001010      (f-string form, same thing)
print(int('1010', 2))    # => 10            (parse back)
print(int('ff', 16))      # => 255
print(int('0b1010', 2))   # => 10           (int() also accepts the 0b/0x/0o prefix directly)
```

> **Gotcha:** `bin()`/`format()` on a **negative** number does NOT give a two's-complement bit string — it
> gives a minus sign followed by the binary of the magnitude (`bin(-10)` → `'-0b1010'`, `format(-10, '08b')` →
> `'-0001010'`, note the field width counts the sign character too). If you need an actual fixed-width
> two's-complement bit string for a negative number, mask first: `format(-10 & 0xFF, '08b')` → `'11110110'`.

### 24.2 Byte conversion (brief)

Useful for hashing, checksum-style problems, or interfacing with binary formats:

```python
n = 1024
b = n.to_bytes(2, byteorder='big')       # 2 bytes, most-significant first
print(b)                                  # => b'\x04\x00'
print(int.from_bytes(b, byteorder='big'))  # => 1024

b_le = n.to_bytes(2, byteorder='little')
print(b_le)                               # => b'\x00\x04'

# signed values need signed=True or you'll get an OverflowError / wrong value
neg = (-1).to_bytes(2, byteorder='big', signed=True)
print(neg)                                 # => b'\xff\xff'
print(int.from_bytes(neg, byteorder='big', signed=True))  # => -1
```

> **Gotcha:** `to_bytes` raises `OverflowError` if the value doesn't fit in `length` bytes (or is negative
> without `signed=True`). This is one of the few places Python *does* enforce a width — because `bytes` itself
> is fixed-width, even though `int` is not.

### 24.3 A Python int as an ARBITRARY-WIDTH bitmask — the superpower

Because `int` has no width ceiling, a single Python int can represent a subset of **any** number of items —
50, 500, 50,000 — with the same `&`/`|`/`^`/`<<`/`>>` toolkit you'd use for a tiny mask. No `BigInt` ceremony
(JS), no `std::bitset<N>` template-parameter-at-compile-time restriction or manual `vector<uint64_t>`
multi-word bitset (C++), no `BitSet` object overhead (Java) — just an `int`.

**Subset of 50 items**, using bit `i` to mean "item `i` is in the set":

```python
n = 50
subset = 0

def add_item(mask: int, i: int) -> int:
    return mask | (1 << i)

def remove_item(mask: int, i: int) -> int:
    return mask & ~(1 << i)

def has_item(mask: int, i: int) -> bool:
    return (mask >> i) & 1 == 1

subset = add_item(subset, 3)
subset = add_item(subset, 47)
subset = add_item(subset, 12)

print(has_item(subset, 47))    # => True
print(has_item(subset, 10))    # => False
print(subset.bit_count())       # => 3     (works fine at width 50 — no special-casing needed)  [3.10+]

subset = remove_item(subset, 12)
items_present = [i for i in range(n) if has_item(subset, i)]
print(items_present)           # => [3, 47]
```

> **vs C++/Java/JS:** the equivalent in C++ needs `std::bitset<50>` (fixed at compile time) or manual
> multi-word handling for a runtime-variable width; in Java, `java.util.BitSet`; in JS, either an array-backed
> structure or `BigInt` (which loses native `Number` bitwise-op speed and requires `1n << 47n` `BigInt`
> literal syntax throughout). Python just uses a plain `int` — no import, no template parameter, no separate
> type.

**Knapsack-feasibility bitset trick: `dp |= dp << w`.** For 0/1 subset-sum-style feasibility (which totals are
achievable from a list of weights), represent the set of achievable sums as **one bit per possible sum** in a
single big int, and fold in each weight with a shift-and-OR — this is exactly the "C++ `bitset` trick" for
speeding up knapsack from `O(n * W)` per-cell work down to `O(n * W / 64)` word-parallel work, except in Python
the "word width" isn't fixed at 64 — the interpreter packs and shifts however many machine words the current
value needs, and you get the same asymptotic win for free:

```python
def subset_sums_reachable(weights: list[int]) -> int:
    """Return a bitmask where bit k is 1 iff sum k is achievable using a subset of weights."""
    dp = 1                      # bit 0 set: the empty subset achieves sum 0
    for w in weights:
        dp |= dp << w           # every sum previously achievable is still achievable (dp),
                                  # AND every previously achievable sum + w becomes achievable (dp << w)
    return dp

weights = [3, 5, 7]
reachable = subset_sums_reachable(weights)
achievable_sums = [k for k in range(reachable.bit_length()) if (reachable >> k) & 1]
print(achievable_sums)     # => [0, 3, 5, 7, 8, 10, 12, 15]

target = 12
print(bool((reachable >> target) & 1))   # => True   ("can we make 12?" answered in O(1) bit test)
```

Why this is fast: `dp << w` and `dp |= ...` are single CPython bigint operations that internally operate word
by word (30 bits per internal digit on most builds), so for a target sum up to `W`, each `dp |= dp << w` costs
roughly `O(W / 30)` — the same "divide by machine word size" speedup you'd hand-roll with a `vector<uint64_t>`
bitset in C++, except Python gives it to you for free with ordinary integer operators, and — because there's
no fixed width — it automatically extends to `W` values beyond 64 bits with zero extra code.

> **Gotcha:** this bitset trick only tells you **feasibility** (can sum `k` be reached?), not which items were
> used or how many ways. If you need count-of-ways or item-reconstruction, you need the classic DP array (or
> a parallel structure), not just the bitmask.

### 24.4 Decision table: int-mask vs `set` vs `list`/array of bools vs `bitarray`

| Structure | Best for | Complexity per op | Notes |
|---|---|---|---|
| **int bitmask** | small-to-medium fixed universe (`n` up to a few thousand); bitmask-DP; subset enumeration; "is X compatible with Y" via `&`; feasibility bitsets (`dp \|= dp << w`) | `O(n / 30)` amortized per bitwise op (word-packed) | No import; supports `&`/`\|`/`^`/shifts directly; hashable (usable as dict key / in a `set` of states) — ideal for bitmask-DP state |
| **`set`** | sparse membership, arbitrary (non-contiguous, non-integer) elements, when you need fast add/remove/contains without caring about bit order | `O(1)` avg per op | No positional/ordering semantics; more memory overhead per element than a packed bitmask; not usable with `&`/`\|` bit tricks directly (though `set` does support `&`, `\|`, `^`, `-` as *set* operations — different semantics, same symbols) |
| **`list`/`bytearray` of bools** | large dense universe (`n` in the hundreds of thousands+) needing per-element flags with O(1) random access/mutation and no need for whole-mask bitwise ops | `O(1)` per element access; `O(n)` for a bulk merge | `bytearray` is far more memory-efficient than `list[bool]` (1 byte vs ~28 bytes per Python `bool` object reference in a list); no built-in "AND two masks" without a loop or `zip` |
| **`bitarray` (3rd-party lib)** | very large bit vectors (millions+) needing both dense storage AND fast vectorized bitwise ops, when a plain Python int becomes unwieldy or you want C-speed operations with explicit fixed width | near-C speed, `O(n/64)`-ish per op | Not in the stdlib — unavailable on most online judges (LeetCode/Codeforces); use only when you control the environment (e.g. local scripts, "pip install bitarray") |

**Rule of thumb:** default to the plain int bitmask for anything DSA/CP-sized (`n <= ~64` for classic bitmask-DP,
up to low thousands for enumeration-heavy tricks) — it needs no import and plugs straight into `&`/`|`/`^`/`<<`/`>>`
and dict/set hashing. Reach for `set` when elements aren't small dense integers. Reach for `bytearray`/`array`
when `n` is huge and you only need O(1) point access, not whole-mask bit ops. `bitarray` is a specialist
choice outside contest/interview settings.

### 24.5 Worked bitmask-DP snippet (small `n`)

Classic "visit all cities" (Travelling Salesman DP) skeleton — state is `(mask, last)` where `mask` is the set
of visited cities as a bitmask (see the broader DSA reference, Pattern 24, for the full family of bitmask-DP
problems: TSP, assignment problem, "minimum XOR subset", etc.):

```python
def tsp_min_cost(dist: list[list[int]]) -> int:
    n = len(dist)
    FULL = (1 << n) - 1
    # dp[mask][last] = min cost to have visited exactly `mask`, ending at `last`, starting from city 0
    dp = [[float('inf')] * n for _ in range(1 << n)]
    dp[1][0] = 0   # only city 0 visited, currently at city 0

    for mask in range(1 << n):
        for last in range(n):
            if dp[mask][last] == float('inf'):
                continue
            if not (mask >> last) & 1:
                continue                    # `last` must actually be in `mask`
            for nxt in range(n):
                if (mask >> nxt) & 1:
                    continue                 # already visited
                new_mask = mask | (1 << nxt)
                new_cost = dp[mask][last] + dist[last][nxt]
                if new_cost < dp[new_mask][nxt]:
                    dp[new_mask][nxt] = new_cost

    return min(dp[FULL][last] + dist[last][0] for last in range(n))

dist = [
    [0, 10, 15, 20],
    [10, 0, 35, 25],
    [15, 35, 0, 30],
    [20, 25, 30, 0],
]
print(tsp_min_cost(dist))   # => 80
```

### 24.6 Counting-bits DP (int-method version, cross-reference to §23.13)

With `bit_count()` available (3.10+), "Counting Bits" collapses to a one-liner, but keep the `dp[i] = dp[i>>1] +
(i&1)` recurrence from §23.13 in your back pocket for judges without 3.10, and because the recurrence pattern
itself (derive `f(i)` from `f(i >> 1)` and `i & 1`) generalizes to other per-bit DPs (e.g. digit-DP-style bit
DPs) where no single built-in exists:

```python
def count_bits_builtin(n: int) -> list[int]:
    return [i.bit_count() for i in range(n + 1)]     # [3.10+]

def count_bits_dp(n: int) -> list[int]:
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i >> 1] + (i & 1)
    return dp

print(count_bits_builtin(7))   # => [0, 1, 1, 2, 1, 2, 2, 3]
print(count_bits_dp(7))         # => [0, 1, 1, 2, 1, 2, 2, 3]
```

---

# PART VI — NUMBERS, PRECISION AND I/O

## 25. Integer Arithmetic — The No-Overflow World

Python's `int` is **arbitrary precision** (a.k.a. bignum). There is no fixed bit width, no wraparound, no silent overflow. This is arguably the single biggest quality-of-life win for CP in Python vs C++/Java/JS.

```python
x = 2 ** 200
print(x)
# => 1606938044258990275541962092341162602522202993782792835301376

fact = 1
for i in range(1, 31):
    fact *= i
print(fact)
# => 265252859812191058636308480000000

big = 10 ** 100          # a googol, exact
print(len(str(big)))
# => 101

a = 123456789123456789
b = 987654321987654321
print(a * b)              # exact — would overflow int64 in C++/Java
# => 121932631137021795226185032733622297921
```

> **vs C++/Java:** `long long` (C++) / `long` (Java) top out at ~9.2×10^18 (`2^63 - 1`). Multiply two ~10^9 numbers and you're already near overflow; multiply two ~10^18 numbers and you overflow silently (undefined behavior in C++, wraps in Java). In Python this simply never happens — `int` grows as needed.
> **vs JS:** `Number` is a 64-bit float — safe integers only up to `2^53 - 1` (~9×10^15). Beyond that you silently lose precision (`9007199254740993 === 9007199254740992` in JS is effectively true for many ops). JS added `BigInt` (`10n`) as an opt-in fix, but you must remember to use it and can't mix `BigInt` with `Number` directly. Python's `int` IS the bigint, always, with no separate type or syntax.

### Modular arithmetic is trivial

Because `int` never overflows, `(a * b) % m` is **always exact**, no matter how large `a` and `b` are — no need to cast to a wider type first.

```python
MOD = 10**9 + 7
a = 999999999999999999999
b = 888888888888888888888
print((a * b) % MOD)
# => 279938862
```

> **vs C++:** you must manually widen: `(long long)a * b % MOD`, and even `long long * long long` can overflow if `a, b` are already near `1e18` — forcing `__int128` or careful bounding. In Java, same problem, same `long` widening dance, and no 128-bit fallback (you'd reach for `BigInteger`, which is slow). In JS, you'd need `BigInt` literals (`10n`) throughout, and mixing with regular numbers throws a `TypeError`. Python: just multiply.

### Built-in modpow and modular inverse

```python
# pow(base, exp, mod) — fast modular exponentiation, built in, O(log exp)
print(pow(3, 200, 10**9 + 7))
# => 655181418

# pow(base, -1, mod) [Python 3.8+] — modular inverse (base must be coprime with mod)
inv = pow(3, -1, 10**9 + 7)
print(inv, (3 * inv) % (10**9 + 7))
# => 333333336 1
```

> **Gotcha:** `pow(a, -1, m)` raises `ValueError: base is not invertible for the given modulus` if `gcd(a, m) != 1`. In C++/Java you'd hand-roll extended Euclid or Fermat's little theorem (`pow(a, m-2, m)` when `m` is prime) — Python gives you both `pow(a, m-2, m)` AND the direct inverse form for free.

### Floor division `//` and modulo `%`

```python
print(-7 // 2)     # floors toward -inf, NOT truncation
# => -4
print(7 // -2)
# => -4
print(-7 % 3)      # result has the SAME SIGN as the divisor
# => 2
print(7 % -3)
# => -2
```

| Expression | Python | C++/Java/JS (truncating division) |
|---|---|---|
| `-7 // 2` | `-4` (floors) | `-3` (truncates toward 0) |
| `-7 % 3` | `2` (sign of divisor) | `-1` (sign of dividend) |
| `7 % -3` | `-2` (sign of divisor) | `1` (sign of dividend) |

> **Gotcha (restated, important):** Python's `//` floors toward `-inf`; C++/Java/JS `/` on integers truncates toward `0`. These differ whenever operands have different signs. If you port a C++ formula involving negative integer division, do NOT assume `//` behaves the same — verify with a small example.
> **vs C++/Java/JS:** in those languages `%` can return a negative result (sign follows the *dividend*), so CP code there routinely does `((a % m) + m) % m` to force a non-negative remainder. In Python, for a **positive** modulus `m`, `%` already returns a non-negative result — that defensive trick is unnecessary. `-7 % 3` is already `2`, not `-1`.
> **Edge case:** if the modulus itself is negative, Python's `%` still follows the divisor's sign, so `7 % -3 == -2` — a *negative* result. This is rare in CP (moduli are conventionally positive) but worth knowing if you ever compute with a negative `m`.

`divmod` returns both at once:

```python
q, r = divmod(17, 5)
print(q, r)
# => 3 2
q, r = divmod(-17, 5)
print(q, r)
# => -4 3
```

### `round` — banker's rounding trap

```python
print(round(2.5))   # NOT 3 — rounds to nearest EVEN on ties
# => 2
print(round(3.5))
# => 4
print(round(0.5))
# => 0
print(round(2.345, 2))   # float imprecision can also bite here
# => 2.35   (or possibly 2.35-ish; see §26 for float caveats)
```

> **Gotcha:** Python 3's `round()` uses "round half to even" (banker's rounding), not "round half away from zero" like most people expect from school math / C's `round()`. `round(0.5) == 0` and `round(1.5) == 2`. If you need consistent away-from-zero rounding, do it manually: `math.floor(x + 0.5)` for positive `x`, or use `decimal.Decimal` with an explicit rounding mode (see §26).

### Performance note (rare)

Big-int arithmetic is `O(digits)` or worse (multiplication is superlinear for huge numbers), so genuinely huge intermediate values (thousands of digits, e.g. unbounded factorials or repeated squaring without a modulus) can slow you down. In the vast majority of CP problems numbers stay small enough (fit in a few dozen digits) that this is irrelevant — but if a solution TLEs and you're computing something like `2**(10**7)` without a modulus, that's the culprit; add a `% MOD` to keep numbers small.

### Conversions

```python
print(int("101", 2))     # base-2 string -> int
# => 5
print(int("ff", 16))
# => 255
print(int("0x1A", 0))    # base 0 = auto-detect prefix (0x/0o/0b)
# => 26
print(int(7.9))          # truncates toward zero, like C++ (int)x
# => 7
print(int(-7.9))
# => -7
```

> **Gotcha:** `int(float)` truncates toward zero (same as C++ `(int)x` / Java `(int)x`), which is DIFFERENT from `//` which floors. `int(-7.9) == -7` but `-7.9 // 1 == -8.0`. Don't conflate the two.

---

## 26. Floats, Decimal and Fractions

`float` in Python is a 64-bit IEEE-754 double — the same representation as C++ `double`, Java `double`, and JS `Number`. All the usual binary-floating-point traps apply.

```python
print(0.1 + 0.2)
# => 0.30000000000000004
print(0.1 + 0.2 == 0.3)
# => False
```

> **Gotcha:** never compare floats with `==`. Use `math.isclose` or an explicit epsilon.

```python
import math
a, b = 0.1 + 0.2, 0.3
print(math.isclose(a, b))                         # default rel_tol=1e-9
# => True
print(math.isclose(a, b, rel_tol=0, abs_tol=1e-9)) # absolute tolerance form
# => True
print(abs(a - b) < 1e-9)                            # manual epsilon
# => True
```

### Infinity and NaN

```python
import math
print(float('inf'), float('-inf'), float('nan'))
# => inf -inf nan
print(math.inf, -math.inf, math.nan)
# => inf -inf nan
print(math.isnan(math.nan), math.isinf(math.inf))
# => True True
print(float('nan') == float('nan'))   # NaN never equals itself
# => False

best = float('-inf')
for v in [3, -5, 9, 1]:
    best = max(best, v)
print(best)
# => 9
```

> Common CP use: `float('inf')` / `math.inf` as a sentinel "unreached" distance in Dijkstra/BFS-with-weights instead of `INT_MAX` — and since Python ints are unbounded you can't accidentally overflow a sentinel by adding to it, but `math.inf` is still cleaner and comparisons behave as expected (`math.inf + 1 == math.inf`).

### Formatting

```python
x = 3.14159265
print(f"{x:.6f}")     # fixed 6 decimal places
# => 3.141593
print(f"{x:.2f}")
# => 3.14
print(round(x, 2))    # round() ALSO works on floats; returns a float
# => 3.14
print(format(x, ".3f"))
# => 3.142
print(f"{1234567:,}")     # thousands separator
# => 1,234,567
print(f"{0.5:.0%}")       # percentage formatting
# => 50%
```

### When to avoid floats in DSA

Floats accumulate rounding error and can never exactly represent most decimal fractions. In CP, avoid floats whenever exactness matters:

- **Fractions / ratios:** compare `a/b` vs `c/d` via cross-multiplication `a*d` vs `c*b` (integer, exact) instead of dividing.
- **Geometry:** prefer integer coordinates and integer cross/dot products over floating angles/distances when possible.
- **Probability / rational results:** use `fractions.Fraction` (below) instead of `float`.
- **Money:** use integer cents, or `decimal.Decimal`.

```python
# BAD: float comparison risk
a, b, c, d = 1, 3, 2, 6
print(a / b == c / d)          # works here, but risky in general
# => True

# GOOD: cross-multiplication, exact for integers
print(a * d == c * b)
# => True
```

### `fractions.Fraction` — exact rationals

```python
from fractions import Fraction

f1 = Fraction(1, 3)
f2 = Fraction(1, 6)
print(f1 + f2)
# => 1/2
print(f1 + f2 == Fraction(1, 2))
# => True
print(Fraction(4, 8))              # auto-reduces (gcd division)
# => 1/2
print(Fraction("0.25"))            # parses decimal strings exactly
# => 1/4
print(float(Fraction(1, 3)))       # convert to float when you finally need one
# => 0.3333333333333333
f3 = Fraction(2, 4) * Fraction(3, 5)
print(f3, f3.numerator, f3.denominator)
# => 3/10 3 10
```

> This is a genuine Python convenience others lack out of the box: exact rational arithmetic with automatic reduction, usable directly in geometry, probability, and combinatorics problems where floats would silently drift. C++ has no standard-library equivalent (you'd write your own `struct Frac` with `__gcd`); Java has none in the JDK either (third-party libs only).

### `decimal.Decimal` — arbitrary-precision base-10

```python
from decimal import Decimal, getcontext

print(Decimal('0.1') + Decimal('0.2'))    # EXACT in base 10, unlike float
# => 0.3
print(Decimal('0.1') + Decimal('0.2') == Decimal('0.3'))
# => True

getcontext().prec = 50                     # set precision (significant digits)
print(Decimal(1) / Decimal(3))
# => 0.33333333333333333333333333333333333333333333333333
```

> **Gotcha:** always construct `Decimal` from a **string** (`Decimal('0.1')`), not from a `float` (`Decimal(0.1)`), or you inherit the float's binary imprecision baked in (`Decimal(0.1)` prints `0.1000000000000000055511151231257827021181583404541015625`).
> Use `Decimal` for money / exact-decimal problems where digit-for-digit base-10 exactness matters. It's slower than `float`, so don't reach for it in hot numeric loops unless the problem demands decimal exactness. `Fraction` is usually the better fit for pure math/CP (ratios, probabilities); `Decimal` is the better fit when the problem is phrased in decimal digits (currency, decimal rounding rules).

### Integer square root — exact, no float error

```python
import math
print(math.isqrt(10**18))     # exact integer floor(sqrt(n)), no float rounding risk
# => 1000000000
print(math.isqrt(99))
# => 9

n = 10**18 + 1
r = math.isqrt(n)
print(r * r <= n < (r + 1) ** 2)   # verify perfect-square boundary exactly
# => True
```

> **Gotcha:** `int(math.sqrt(n))` can be off by one for large `n` because `math.sqrt` converts through `float` (53 bits of precision — breaks down past ~2^53). `math.isqrt` (3.8+) works purely on integers and is always exact. Always prefer `math.isqrt` over `int(sqrt(n))` in CP when `n` can exceed ~10^15.

### General rule

Prefer `int` (with `//`, `%`, cross-multiplication) or `Fraction` over `float` whenever a problem's correctness hinges on exactness — comparisons, equality checks, accumulation over many steps. Reach for `float` only when the problem is inherently approximate (allowed error `1e-6`, geometry with irrational results, etc.), and even then, verify with `math.isclose`, not `==`.

---

## 27. Fast I/O (the make-or-break CP topic for Python)

**The rule:** the friendly `input()` and `print()` are SLOW. On CPython, for problems reading/writing 10^5–10^6+ lines, plain `input()`/`print()` can single-handedly cause TLE even when your algorithm is optimal. Fast I/O is not optional in Python CP — it's often the difference between AC and TLE on identical logic.

### (a) The standard fast input: redefine `input`

```python
import sys
input = sys.stdin.readline

n = int(input())
a = list(map(int, input().split()))
print(n, a)
# => 3 [1, 2, 3]     (given stdin: "3\n1 2 3\n")
```

> **Why it's faster:** `sys.stdin.readline` reads a line via a lower-level buffered call, avoiding the overhead of the interactive-friendly machinery behind builtin `input()` (which also handles prompt strings, readline hooks, etc.).
> **Gotcha:** `sys.stdin.readline` keeps the **trailing newline** (`'\n'`), unlike builtin `input()` which strips it. This is usually harmless because `int(...)`, `.split()`, and `float(...)` all ignore trailing whitespace — but if you read a raw string line (e.g. a word/name), call `.strip()` (or `.rstrip('\n')`) explicitly:

```python
import sys
input = sys.stdin.readline
s = input()
print(repr(s))          # trailing '\n' still present
# => 'hello\n'
s = input().strip()
print(repr(s))
# => 'hello'
```

Reading `n` lines each with one integer or a list:

```python
import sys
input = sys.stdin.readline

n = int(input())
values = [int(input()) for _ in range(n)]        # n lines, one int each
# => e.g. [4, 7, 2] for stdin "3\n4\n7\n2\n"

m = int(input())
rows = [list(map(int, input().split())) for _ in range(m)]  # m lines of ints
```

### (b) Reading ALL input at once — maximum speed

For the largest inputs, one bulk read beats even `readline` in a loop, because it minimizes the number of Python-level I/O calls.

```python
import sys
data = sys.stdin.buffer.read().split()   # bytes, split on whitespace — FASTEST
# data is a list of bytes objects, e.g. [b'3', b'1', b'2', b'3']
idx = 0
n = int(data[idx]); idx += 1
a = [int(x) for x in data[idx:idx + n]]; idx += n
print(n, a)
# => 3 [1, 2, 3]
```

`sys.stdin.buffer.read()` reads raw **bytes** (no text decoding overhead) — faster than `sys.stdin.read()` which reads and decodes as `str`. `int(b'123')` works directly on bytes, so decoding to `str` is usually unnecessary.

```python
import sys
data = sys.stdin.read().split()    # str version — slightly slower than .buffer.read()
```

The fastest common competitive pattern — a token iterator:

```python
import sys

it = iter(sys.stdin.buffer.read().split())
def ni():
    return int(next(it))
def ns():
    return next(it).decode()

n = ni()
a = [ni() for _ in range(n)]
print(n, a)
# => 3 [1, 2, 3]     (given stdin: "3\n1 2 3\n")
```

Or the ultra-compact one-liner style seen in many fast CP templates:

```python
import sys
input = iter(sys.stdin.buffer.read().split()).__next__
ni = lambda: int(input())

n = ni()
a = [ni() for _ in range(n)]
print(n, a)
# => 3 [1, 2, 3]
```

> **Gotcha:** once you tokenize the entire input with `.split()`, you lose line structure — all whitespace (spaces AND newlines) becomes token separators. This is fine (and usually preferable) for grid/array-heavy problems where you just need "the next n tokens" regardless of line breaks, but wrong if blank lines or line boundaries are semantically meaningful in the input format (rare in CP, but check the problem statement).

### (c) Output: batch it, don't `print` in a loop

```python
# SLOW: print() flushes / has overhead on every call, O(n) syscalls-ish overhead
for x in range(5):
    print(x)

# FAST: build strings, join once, print once
results = [str(x) for x in range(5)]
print('\n'.join(results))
# => 0
#    1
#    2
#    3
#    4

# ALSO FAST: sys.stdout.write — needs strings and explicit '\n'
import sys
out = []
for x in range(5):
    out.append(str(x))
sys.stdout.write('\n'.join(out) + '\n')
# => 0
#    1
#    2
#    3
#    4
```

> **Gotcha:** `sys.stdout.write` does NOT auto-convert or auto-append newlines like `print` does — you must pass a `str` (not an `int`) and add `'\n'` yourself where needed. `sys.stdout.write(5)` raises `TypeError`.

A common idiom: accumulate output lines in a list throughout the whole solve, then flush once at the end:

```python
import sys
out = []
for case in range(3):
    out.append(f"Case {case+1}: {case * case}")
sys.stdout.write('\n'.join(out) + '\n')
# => Case 1: 0
#    Case 2: 1
#    Case 3: 4
```

### Comparison table

| Method | Relative speed | When to use |
|---|---|---|
| `input()` (builtin) | Slowest | Small inputs (< ~10^4 lines), quick scripts, when clarity matters more |
| `sys.stdin.readline` (`input = ...`) | Fast | Default fast-I/O for most CP problems — one-line change |
| `sys.stdin.buffer.read().split()` + token iterator | Fastest | Very large inputs (10^5–10^7 tokens), tight TLEs |
| `print()` in a loop | Slow | Never for large output — batch it |
| `'\n'.join(...)` + one `print`/`sys.stdout.write` | Fast | Default fast-output pattern |

### `map(int, ...)` idiom

```python
a, b, c = map(int, input().split())     # unpack exactly 3 ints from one line
print(a, b, c)
# => 1 2 3    (given stdin "1 2 3\n")

nums = list(map(int, input().split()))  # variable-length list of ints from one line
```

> **Gotcha:** `map(...)` returns a lazy iterator, not a list — `a, b, c = map(int, ...)` works (unpacking consumes it), but if you need to reuse it multiple times or index it, wrap in `list(...)` first. Unpacking into the wrong number of names raises `ValueError: not enough values to unpack` / `too many values to unpack`.

### Reading a grid

```python
import sys
input = sys.stdin.readline

# grid of characters (e.g. maze/board), n rows
n = int(input())
grid = [input().strip() for _ in range(n)]     # list[str], each row a string
# grid[i][j] indexes character at row i, col j

# grid of integers, n rows x m cols
n, m = map(int, input().split())
mat = [list(map(int, input().split())) for _ in range(n)]
```

### Reading until EOF (unknown number of lines)

```python
import sys
total = 0
for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    total += int(line)
print(total)
# => sums every integer given, one per line, until input ends
```

`for line in sys.stdin` is itself reasonably fast (buffered iteration) and is the idiomatic way to consume input when the line count isn't given up front.

### CPython vs PyPy — the crucial CP fact

- **CPython** (the reference interpreter — what "Python" means by default, and what LeetCode always runs) is roughly **30–100x slower than C++** for raw computational loops. This gap is why fast I/O and vectorized/stdlib operations (§ elsewhere) matter so much — every avoidable Python-level loop iteration costs real time.
- **PyPy** is a JIT (just-in-time compiling) implementation of Python. For pure-Python loop-heavy code, PyPy is typically **5–30x faster than CPython**, often closing most of the gap to C++. On Codeforces, when a language selector is available, **choose PyPy 3** for tight-TLE problems — it is very often the difference between AC and TLE for an otherwise-correct O(n log n) or O(n) Python solution.
- **Nuance:** PyPy's JIT warms up over many iterations of "hot" pure-Python code — it shines on tight loops. It is sometimes **not** faster (occasionally slower to start, or no better) for code dominated by:
  - big-integer-heavy arithmetic (bignum ops are already implemented in C in CPython, so there's less for the JIT to speed up),
  - heavy use of C-extension-backed libraries (`numpy`, etc. — the compute already happens in C, not interpreted Python),
  - very short-running programs (JIT warm-up overhead may not pay off).
- **Practical rule of thumb for Codeforces:** default to PyPy 3 unless the problem statement or your solution leans on things PyPy handles worse (rare) or the judge doesn't offer PyPy for that problem. For LeetCode, there is no PyPy option — you're always on CPython 3.11, so fast I/O habits and algorithmic efficiency matter even more there.

| | CPython | PyPy |
|---|---|---|
| What it is | Reference interpreter (bytecode, no JIT) | JIT-compiling alternative interpreter |
| Speed vs C++ | ~30–100x slower | Often much closer (5–30x faster than CPython) |
| Codeforces | Available as "Python 3" | Available as "PyPy 3" — prefer for tight TLE |
| LeetCode | The only option (3.11) | Not available |
| Best at | Anything (baseline) | Long pure-Python loops, recursion-heavy code |
| Weaker at | — | JIT warm-up on short programs; no extra gain over C-extension-heavy code (numpy etc.) |

### LeetCode vs Codeforces I/O model

LeetCode has **no stdin/stdout** — you implement a method on a given `class Solution`, and the judge calls it directly with arguments, reading your `return` value. All the fast-I/O material above (`sys.stdin`, batching prints) is irrelevant on LeetCode; it matters for Codeforces/AtCoder/etc., which read from stdin and expect stdout.

```python
# LeetCode skeleton — no I/O code at all, just implement the method
class Solution:
    def twoSum(self, nums: list[int], target: int) -> list[int]:
        seen = {}
        for i, x in enumerate(nums):
            if target - x in seen:
                return [seen[target - x], i]
            seen[x] = i
        return []
```

```python
# Codeforces skeleton — read from stdin, write to stdout, manage I/O yourself
import sys
input = sys.stdin.readline

def solve():
    n = int(input())
    a = list(map(int, input().split()))
    print(sum(a))

t = int(input())
for _ in range(t):
    solve()
```

---

## 28. Randomness, Time and the Runtime

### `random` module

```python
import random

random.seed(42)                    # reproducible sequence — set once, for debugging a failing test
print(random.randint(1, 6))        # INCLUSIVE on BOTH ends — a trap vs other langs
# => 6

print(random.randrange(0, 10))     # like range(): [0, 10) — EXCLUSIVE upper, like most langs' rand
# => 1

print(random.random())             # float in [0.0, 1.0)
# => 0.6394267984578837 (some float in [0,1))

lst = [1, 2, 3, 4, 5]
random.shuffle(lst)                # shuffles IN PLACE, returns None
print(lst)
# => [3, 1, 4, 5, 2]   (order will vary; deterministic here only because seeded above)

print(random.choice([10, 20, 30]))     # one random element
# => 20
print(random.sample([1, 2, 3, 4, 5], 3))  # k distinct elements, no repeats, new list
# => [4, 1, 5]
```

> **Gotcha:** `random.randint(a, b)` is **inclusive of both `a` and `b`** — different from `random.randrange(a, b)` (exclusive upper, like `range`) and different from most other languages' "randint"-style functions which are commonly exclusive on one end. Double-check which one you're calling; mixing them up silently shifts your bounds by one.
> **Gotcha:** `random.shuffle(lst)` shuffles **in place** and returns `None` — `lst = random.shuffle(lst)` sets `lst` to `None`. Same "mutates and returns `None`" family as `.sort()`, `.append()`, etc.

**CP uses:**
- **Shuffling to defend against adversarial worst cases:** e.g. quicksort-style algorithms, or hash-based structures under adversarial test data — `random.shuffle(arr)` before processing can turn a worst-case O(n²) into expected O(n log n) when the judge's test data is crafted against a naive deterministic approach.
- **Seeding to reproduce a failing test:** `random.seed(1234)` when writing a stress-test/brute-force checker locally, so a failing random case can be replayed exactly instead of chasing a new random failure each run.

### Timing

```python
import time

start = time.perf_counter()     # monotonic, highest resolution — preferred for benchmarking
# ... work ...
elapsed = time.perf_counter() - start
print(f"{elapsed:.6f}s")
# => 0.000012s  (varies)

t0 = time.time()                # wall-clock time (epoch seconds) — fine too, less precise guarantee
```

> Use `time.perf_counter()` over `time.time()` for benchmarking: it's guaranteed monotonic (never goes backward, e.g. from system clock adjustments) and has the best available resolution. `time.time()` is for "what time is it" (epoch timestamps), not for measuring short intervals precisely.

### `sys.stdin` / `sys.stdout` / `sys.stderr`

```python
import sys
print("debug: n =", 5, file=sys.stderr)   # goes to stderr — does NOT corrupt stdout answer
print(42)                                  # the actual answer, on stdout
# stdout => 42
# stderr => debug: n = 5
```

> A handy CP trick: judges compare only stdout against expected output, so debug prints sent to `sys.stderr` (via `file=sys.stderr`) are visible to you when running locally but never interfere with grading — leave them in during development without fear of a wrong-answer verdict from stray debug output.

### `if __name__ == '__main__':` guard

```python
def main():
    import sys
    input = sys.stdin.readline
    n = int(input())
    print(n * 2)

if __name__ == '__main__':
    main()
```

> **Why CP solutions often wrap logic in `main()`:** in CPython, **local variable access is faster than global variable access** (locals live in a fast array slot resolved at compile time; globals go through a dict lookup each access). A solve loop written at module (global) scope pays dict-lookup overhead on every variable access; the same code inside a `def main(): ...` function uses locals throughout and is measurably faster in tight loops. This is a real micro-optimization, not just style — worth doing when a solution is borderline on time. (Function-call overhead and closures are covered in Part VII; this note is a forward reference.)
> The `if __name__ == '__main__':` guard itself matters more for importable scripts/tests than single-file CP submissions, but wrapping in `main()` and calling it unconditionally (`main()`) at file end is equally common and sufficient for a CP submission — the guard is a good habit either way.

### Standard CP import header

```python
import sys
from collections import deque, defaultdict, Counter
import heapq
from bisect import bisect_left, bisect_right, insort
from functools import lru_cache, cmp_to_key
import math
from itertools import permutations, combinations, product, accumulate, groupby

input = sys.stdin.readline
```

### Key stdlib modules for DSA — quick reference

| Module | Provides |
|---|---|
| `sys` | `stdin`/`stdout`/`stderr`, `setrecursionlimit`, `maxsize`, fast I/O |
| `collections` | `deque` (O(1) ends), `defaultdict`, `Counter`, `OrderedDict`, `namedtuple` |
| `heapq` | Binary min-heap functions (`heappush`, `heappop`, `heapify`, `nlargest`/`nsmallest`) |
| `bisect` | Binary search on sorted sequences (`bisect_left`/`right`, `insort`) |
| `functools` | `lru_cache`/`cache` (memoization), `reduce`, `cmp_to_key` (custom sort comparators) |
| `math` | `isqrt`, `gcd`, `lcm` (3.9+), `factorial`, `comb`, `perm`, `inf`, `isclose`, trig/log |
| `itertools` | `permutations`, `combinations`, `product`, `accumulate`, `groupby`, `chain` |
| `random` | `randint`, `shuffle`, `choice`, `sample`, `seed` |
| `time` | `perf_counter`, `time` — benchmarking |
| `string` | `ascii_lowercase`, `digits`, etc. — character set constants |
| `decimal` | `Decimal` — exact base-10 arbitrary precision |
| `fractions` | `Fraction` — exact rational arithmetic |
| `re` | Regular expressions for string-parsing-heavy problems |

---

# PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT

## 29. The Contest Template and Structure

### 29.1 The full Codeforces template

This is a single paste-ready block. Every piece is explained below it.

```python
import sys
from collections import deque, defaultdict, Counter
import heapq
import bisect
from functools import lru_cache
import math
from itertools import accumulate, permutations, combinations, product

input = sys.stdin.buffer.readline  # fast line-at-a-time input, returns bytes

DEBUG = False


def dprint(*args, **kwargs):
    if DEBUG:
        print(*args, file=sys.stderr, **kwargs)


def solve():
    n = int(input())
    a = list(map(int, input().split()))
    # ... solve one test case, append results to `out` ...
    out.append(str(sum(a)))


def main():
    global out
    out = []
    t = int(input())
    for _ in range(t):
        solve()
    sys.stdout.write('\n'.join(out) + '\n')


if __name__ == '__main__':
    main()
```

> **Gotcha:** `input = sys.stdin.buffer.readline` shadows the builtin `input()` and returns **bytes**, not `str`. `int(b"5\n")` works fine (int() accepts bytes), but `b.split()` splits bytes into a list of bytes objects, and `map(int, ...)` still works. If you need actual strings (e.g. reading a word), decode: `input().decode()`. If that trips you up, use `input = sys.stdin.readline` instead (text mode, slightly slower but returns `str`).

### 29.2 Line-by-line breakdown

| Line | Why |
|---|---|
| `import sys` | needed for fast I/O, stderr, recursion limit |
| `from collections import deque, defaultdict, Counter` | O(1) queue pops, auto-vivifying dict, frequency counting — used in nearly every problem |
| `import heapq` | binary heap (priority queue) — Dijkstra, k-th smallest, greedy merging |
| `import bisect` | binary search on sorted sequences |
| `from functools import lru_cache` | one-line memoization for recursive DP |
| `import math` | `gcd`, `isqrt`, `inf`, `log2`, `comb`, `factorial` |
| `import itertools` bits | `accumulate` (prefix sums in C), `permutations`/`combinations`/`product` for brute force / small-n enumeration |
| `input = sys.stdin.buffer.readline` | reading via `input()` builtin is line-buffered and slow for 10^5+ lines; `sys.stdin.buffer.readline` is much faster (see §31) |
| `out = []` + `'\n'.join(out)` | building one big string and writing once beats calling `print()` per line (see §31) |
| `def main(): ...` | **wrapping logic in a function makes all its variables LOCAL.** CPython bytecode accesses locals via `LOAD_FAST` (array index into the frame) but globals via `LOAD_GLOBAL` (a dict lookup, with a fallback to builtins). `LOAD_FAST` is measurably faster — in tight loops touching a variable millions of times, this alone can save 10-30%. This is a real, benchmarkable CPython effect, not folklore. |
| `if __name__ == '__main__': main()` | lets the file be imported without side effects (irrelevant on CF, good habit anyway; also required if you use the threading trick in §30) |
| `for _ in range(int(input())): solve()` | standard "T test cases" loop — read T once, call `solve()` T times |

> **vs C++:** in C++ global vs local variable access speed differences are usually optimized away by the compiler; in interpreted CPython the `LOAD_FAST` vs `LOAD_GLOBAL` distinction is real and visible at runtime. This is a Python-specific micro-optimization with no real C++/Java/JS analogue (JIT'd JS/Java engines typically optimize this away too).

### 29.3 Minimal LeetCode skeleton

LeetCode never wants you to read input or print output — you implement a method on `Solution` and the harness drives it.

```python
from typing import List, Optional


class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        seen = {}
        for i, x in enumerate(nums):
            if target - x in seen:
                return [seen[target - x], i]
            seen[x] = i
        return []
```

> **Gotcha:** no `main()`, no `input()`, no `sys.stdout` needed. `self` is required as the first parameter even though you rarely use it — Python has no implicit `this` (unlike Java/JS methods, closer to how C++ requires nothing since methods are compiler-implicit but Python makes it an explicit textual parameter).

### 29.4 No macros — use tiny helper lambdas instead

Python has no preprocessor and no macros (same situation as Java/JS, unlike C++'s `#define`/templates for competitive shortcuts). The idiomatic substitute is one-letter helper lambdas defined once at the top:

```python
import sys
input = sys.stdin.readline

ni = lambda: int(input())                      # read one int
nl = lambda: list(map(int, input().split()))    # read a line of ints
ns = lambda: input().split()                    # read a line of tokens (str)
nss = lambda: input().strip()                   # read one whitespace-trimmed string
n2 = lambda: map(int, input().split())          # lazy version of nl (no list())

# usage
n = ni()
a = nl()
x, y = n2()
```

> **Gotcha:** `map()` returns a lazy iterator. `x, y = n2()` works via unpacking (it consumes exactly 2 items), but `n2()` printed directly shows `<map object at 0x...>`, not the values — wrap in `list(...)` to materialize.

### 29.5 Debug helper gated behind a flag

```python
import sys

DEBUG = False  # flip to True locally; NEVER leave True when submitting


def dprint(*args, **kwargs):
    if DEBUG:
        print(*args, file=sys.stderr, **kwargs)


dprint("state:", {"n": 5, "arr": [1, 2, 3]})
```

`file=sys.stderr` matters: on Codeforces/LeetCode, stdout is graded — anything printed there that isn't your real answer causes a Wrong Answer / Presentation Error. stderr is never checked, so it's safe for scratch logging even in a submitted-but-DEBUG-False file (the `if DEBUG` guard means it costs nothing and prints nothing when off, but even if you forgot to flip it, stderr output does not corrupt stdout-based judging).

### 29.6 A reusable token-reader class (alternative to per-line `input()`)

For problems where input isn't cleanly one-value-per-line (numbers wrapped across lines, mixed formats), read the entire stdin once and hand out tokens one at a time — this sidesteps line-boundary assumptions entirely and is also the fastest input pattern (§31.8):

```python
import sys


class Reader:
    def __init__(self):
        self._it = iter(sys.stdin.buffer.read().split())

    def s(self) -> str:
        return next(self._it).decode()

    def i(self) -> int:
        return int(next(self._it))

    def f(self) -> float:
        return float(next(self._it))


rd = Reader()
n = rd.i()
arr = [rd.i() for _ in range(n)]
```

> **Gotcha:** this reads and tokenizes the **whole input up front**, which is fine (and fastest) for CF-sized inputs (typically well under a few hundred MB), but means you can't interleave "read some, print a prompt, read more" — irrelevant for CP judges (no interactivity), but worth knowing if you ever port this pattern to an interactive-protocol problem, where you must flush output and read incrementally instead.

### 29.7 Import placement

If you use `sortedcontainers.SortedList`, `heapq`, `deque`, or anything else, import it **once, at the top of the file**, exactly as you would in a normal Python project. Unlike JS/Node CP setups (which sometimes require manually inlining a library's source because there's no npm at judge time), Codeforces' Python judge already has the full standard library, and `sortedcontainers` is available on Codeforces (check "Choose your compiler" language list — it's bundled with PyPy 3 / CPython 3 on CF). So:

```python
from sortedcontainers import SortedList, SortedDict, SortedSet
```

is a single line, no vendoring needed — a real batteries-included advantage over browser/Node CP setups.

> **vs C++:** in C++ you'd `#include <bits/stdc++.h>` or hand-roll a policy-tree ordered set; in Python `sortedcontainers.SortedList` gives you an O(log n) insert/erase/index sorted structure out of the box (not in the stdlib itself, but present on CF/most judges — LeetCode also has it available).

---

## 30. Recursion and the Recursion Limit

### 30.1 The default limit, and why it bites in CP

```python
import sys
print(sys.getrecursionlimit())  # 1000 by default
```

Python's recursion limit protects the C stack from overflowing (each Python call frame consumes real C stack space, unlike languages with a huge dedicated call stack or tail-call optimization). **1000 is tiny for CP.** A recursive DFS over a graph with 10^5 nodes that happens to be a long path (worst case: a linked-list-shaped tree/graph) recurses 10^5 deep and blows the limit almost immediately:

```python
import sys
sys.setrecursionlimit(1000)  # default-ish, shown explicitly

def dfs(u, adj, visited):
    visited.add(u)
    for v in adj[u]:
        if v not in visited:
            dfs(v, adj, visited)

# adj is a "chain": 0-1-2-3-...-N-1  (a path graph)
N = 5000
adj = defaultdict_style_adjacency = {i: [i+1] for i in range(N-1)}
adj[N-1] = []
# dfs(0, adj, set())  -> RecursionError: maximum recursion depth exceeded
```

> **vs C++:** C++'s implicit call stack is typically 1 MB (MSVC) to 8 MB (Linux default `ulimit -s`), and each frame is small, so unguided recursion to depth 10^5-10^6 often "just works" (though it's still fragile and judge-dependent). Python's *default* limit (1000) is a **software** limit set far below what the OS stack could handle — Python is being deliberately conservative, unlike C++ which has no built-in recursion-depth guard at all (you just segfault).

### 30.2 Fix 1 — raise the limit (with the segfault caveat)

```python
import sys
sys.setrecursionlimit(10**6)  # or 2 * 10**5 + 100 as a tighter/safer bound
```

> **Gotcha:** raising the *Python* recursion limit does **not** raise the underlying OS/C stack size. If you set the limit to 10^6 and actually recurse that deep, you can **segfault** (hard crash, not a catchable Python exception) because you've exhausted the real C stack before Python's own counter says "stop." This is the single most common gotcha with this fix — it *works* for moderate depths (a few times 10^5) but is not a universal safety net.

### 30.3 Fix 2 — the threading trick (raise the actual stack size)

The robust companion to `setrecursionlimit` is to run your recursive code in a new thread that you explicitly give a big C stack (analogous to spawning a thread with a large stack size in Java to work around its own `StackOverflowError`).

```python
import sys
import threading


def main():
    # ... your actual recursive solution here ...
    n = int(input())
    parent = [0] + list(map(int, input().split()))
    sys.setrecursionlimit(1 << 25)

    def dfs(u):
        depth = 1
        for v in children[u]:
            depth = max(depth, 1 + dfs(v))
        return depth
    # ... call dfs, print result ...


if __name__ == '__main__':
    sys.setrecursionlimit(1 << 25)          # ~33 million, generous
    threading.stack_size(1 << 27)           # 128 MB C stack for the new thread
    t = threading.Thread(target=main)
    t.start()
    t.join()
```

This is **the** standard Python-CP pattern for "I have a genuinely deep recursive DFS and no time to rewrite it iteratively." The main thread's stack size can't be changed on most platforms after start, so you launch a *new* thread with `threading.stack_size(...)` set beforehand, and do all real work inside it.

> **Gotcha:** `threading.stack_size()` must be called **before** `Thread(...).start()`, and it affects threads created *after* the call, not the main thread. Also note it's measured in bytes (`1 << 27` = 128 MiB).

> **vs Java:** identical idea to `new Thread(null, runnable, "main", 1 << 27).start()` in Java to dodge `StackOverflowError` — same root cause (bounded per-thread stack), same fix (bigger stack on a fresh thread).

### 30.4 Fix 3 — convert to iterative with an explicit stack (most robust)

This avoids the recursion-limit problem entirely and is the fix of choice on **PyPy**, where the threading trick is less reliable and function-call overhead differences make iterative code comparatively even more attractive.

**Recursive DFS (reference):**

```python
def dfs_recursive(start, adj):
    visited = {start}
    def go(u):
        for v in adj[u]:
            if v not in visited:
                visited.add(v)
                go(v)
    go(start)
    return visited
```

**Iterative DFS with an explicit stack (equivalent, no recursion):**

```python
def dfs_iterative(start, adj):
    visited = {start}
    stack = [start]
    while stack:
        u = stack.pop()
        for v in adj[u]:
            if v not in visited:
                visited.add(v)
                stack.append(v)
    return visited
```

> **Gotcha:** this iterative version visits nodes in a different *order* than the naive recursive version (LIFO pop order vs Python's left-to-right recursive descent) — fine if you only need reachability/components, but if the exact traversal order matters (e.g. Euler tour, pre/post timestamps), you need the state-marker version below.

**Iterative post-order (needed for tree DP / subtree aggregation) using a visited/color marker:**

```python
def dfs_postorder(start, adj):
    order = []
    stack = [(start, False)]  # (node, processed_children_flag)
    visited = {start}
    while stack:
        u, processed = stack.pop()
        if processed:
            order.append(u)          # post-order visit: children already done
            continue
        stack.append((u, True))      # re-push self to process AFTER children
        for v in adj[u]:
            if v not in visited:
                visited.add(v)
                stack.append((v, False))
    return order
```

This "push self back with a `processed=True` marker" trick is the standard way to simulate the "do work after recursive calls return" part of a recursive post-order traversal without an actual call stack. It generalizes: `(node, state)` where `state` can be an index into "which child to visit next" for in-order-style traversals too.

### 30.5 `lru_cache` and recursion depth

`@lru_cache` (memoized recursion) is convenient for top-down DP, but it is **still real recursion** — it does not avoid the recursion limit or the C-stack cost, it only avoids recomputation.

```python
from functools import lru_cache
import sys

sys.setrecursionlimit(10**6)

@lru_cache(maxsize=None)
def fib(n):
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)
```

If your memoized recursion's *dependency chain* is deep (e.g. `fib(n)` needs `fib(n-1)` needs `fib(n-2)` ... down to `fib(0)`, all uncached on first call), you still pay the full recursion depth on that first descent — memoization avoids exponential blowup, not linear stack depth. For deep chains, either raise the limit/thread stack (§30.2-30.3) or convert to a bottom-up iterative DP table.

### 30.6 When recursion is fine vs when you must go iterative

| Situation | Verdict |
|---|---|
| Depth < ~900 (well under default 1000) | plain recursion is fine, no changes needed |
| Depth up to a few × 10^4, with `setrecursionlimit` raised | usually fine, but test the worst case (a skewed/path-like tree, not just a balanced one) |
| Depth ~10^5-10^6 (e.g. worst-case skewed tree, long chain graph) | use the threading trick, or better, go iterative |
| Depth unbounded / adversarial input on CF | **always** go iterative or use the threading trick — CF setters routinely include chain/star worst cases specifically to break naive recursion |
| Tree DP needing post-order aggregation, deep tree | iterative post-order with explicit stack (§30.4) — most robust, no stack-size guessing |
| PyPy submission | prefer iterative; the threading trick still works on PyPy but PyPy's own C-stack/GC interaction with very deep recursion is less predictable than CPython's |

> **Gotcha:** a balanced binary tree of 10^5 nodes only has depth ~17 (log2), which is completely safe — it's specifically **skewed** trees, linked-list-shaped graphs, and long simple paths that create 10^5-deep recursion. Don't assume "n is only 10^5" means recursion is safe; check the *shape*, not just the size.

### 30.7 PyPy note

PyPy still enforces a Python-level recursion limit and still consumes real C-stack per frame, just like CPython — `sys.setrecursionlimit` and the threading-with-bigger-stack trick (§30.3) both still apply verbatim on PyPy. What differs is the constant factor: PyPy's JIT makes plain function calls (including recursive ones) cheaper per call than CPython's, so a given recursion depth is somewhat less likely to be the bottleneck — but it does **not** raise the depth at which you get a `RecursionError` or risk a stack overflow, so don't skip §30.2-30.4 just because you're submitting to PyPy. When in doubt, the iterative-with-explicit-stack rewrite (§30.4) is the option that works identically well (and is often fastest) on both CPython and PyPy.

---

## 31. Performance — Why Python Is Slow and How to Cope

### 31.1 The blunt truth

CPython executes bytecode in a tree-walking interpreter loop with dynamic typing checks on every operation — roughly **30-100x slower than optimized C++** for equivalent raw loop-heavy code, and noticeably slower than JIT'd JS (V8) or JIT'd/AOT Java for the same reason (no JIT in stock CPython). Some Codeforces problems with tight time limits (1-2s) and O(n) or O(n log n) algorithms over n = 10^6-10^7 are **not passable in vanilla CPython even with a correct, optimal algorithm** — they require either PyPy or dropping to C-level operations for the hot loop. This is a real, structural limitation, not a skill issue — plan for it rather than fighting it every contest.

### 31.2 Rule (a): choose PyPy for tight loops

On Codeforces, submit under **PyPy 3** (not CPython) whenever your solution has a tight, unavoidable per-element Python-level loop (simulation, heavy DP, brute-force-ish constant factors). PyPy's JIT compiles hot loops to machine code and commonly gives **10-50x** speedup over CPython with **zero code changes**.

> **Gotcha — the big-int caveat:** PyPy's JIT does not meaningfully speed up **arbitrary-precision big-integer arithmetic** (numbers exceeding machine word size) — those still go through the same slow bignum path as CPython, since JIT-compiling variable-width integer math to fixed machine instructions isn't generally possible. If your bottleneck is huge-integer modular exponentiation or bignum multiplication rather than loop overhead, switching to PyPy buys you much less. Also: some libraries and exact floating-point/`decimal` edge-case behaviors can differ subtly between CPython and PyPy — test if precision matters.

### 31.3 Rule (b): push work into C-level builtins

Every builtin function, comprehension, and stdlib call listed below runs its inner loop in **C**, bypassing the Python bytecode interpreter almost entirely. Prefer them over hand-written `for` loops doing the same thing.

| Slow (Python-level loop) | Fast (C-level builtin) |
|---|---|
| `s = 0`<br>`for x in arr: s += x` | `s = sum(arr)` |
| `res = []`<br>`for x in arr: res.append(x*2)` | `res = [x*2 for x in arr]` (list comp is still bytecode but avoids `.append` overhead + is optimized specially) or `res = list(map(lambda x: x*2, arr))` |
| `s = ""`<br>`for w in words: s += w` | `s = ''.join(words)` |
| manual bubble/insertion sort | `arr.sort()` / `sorted(arr)` (Timsort, C implementation) |
| `for i in range(len(arr)):`<br>`  if arr[i] > best: best = arr[i]` | `best = max(arr)` |
| custom comparator loop for "sort by key" | `sorted(arr, key=lambda x: (...))` — never hand-roll a comparator loop; `key=` is C-level, and Python 3 has no `cmp=` (use `functools.cmp_to_key` only if you truly need a comparator) |
| `for x in arr:`<br>`  if x == target: found = True` | `target in arr` (still O(n) but C-level scan) or better, `target in a_set` for O(1) |
| manual prefix sums via loop | `list(itertools.accumulate(arr))` |
| manual gcd loop | `math.gcd(a, b)` |
| manual "count occurrences" loop | `collections.Counter(arr)` |
| nested loop for cartesian product | `itertools.product(a, b)` |

> **vs C++/Java:** in C++ a hand-written `for` loop *is* already compiled machine code, so there's no "use a builtin instead" trick needed — the loop and the "builtin" (`std::accumulate`, etc.) perform similarly. In Python the *only* way to get near-C speed for a bulk operation is to hand it to a function that internally loops in C — this is a Python-specific idiom with no real C++ analogue (and only a partial JS analogue via typed arrays/array methods, which JIT well but aren't guaranteed C-speed).

### 31.4 Rule (c): use locals, cache attribute/method lookups

```python
def process(arr):
    # slow: arr.append is looked up (attribute resolution) on every call
    out = []
    for x in arr:
        if x % 2 == 0:
            out.append(x * x)
    return out


def process_fast(arr):
    # fast: resolve the bound method ONCE, call the cached local
    out = []
    ap = out.append          # cache method lookup as a local
    for x in arr:
        if x % 2 == 0:
            ap(x * x)
    return out
```

Wrapping your whole solution in `main()` (§29) makes every variable inside it a fast local (`LOAD_FAST`) instead of a global (`LOAD_GLOBAL`, a dict lookup with builtins fallback). Combined with caching hot method references (`ap = out.append`, `push = heapq.heappush`) before a tight loop, this removes repeated attribute-resolution overhead that CPython would otherwise redo on every single iteration.

### 31.5 Rule (d): avoid repeated attribute lookups / dots in loops

```python
# slower: `self.graph[u]` and `.append` re-resolved every iteration
for u in range(n):
    for v in self.graph[u]:
        self.result.append(v)

# faster: hoist lookups out of the hot loop
graph = self.graph
result = self.result
ap = result.append
for u in range(n):
    for v in graph[u]:
        ap(v)
```

Every `.` in a loop body is a dictionary lookup in CPython (attributes live in `__dict__` or slots resolved at runtime) — hoisting it once outside the loop and reusing a local name avoids paying that cost n times.

### 31.6 Rule (e): flat arrays over list-of-lists for huge numeric data

```python
import array

# list-of-lists: each row is a separate Python list object (pointer chasing, more memory)
grid = [[0] * m for _ in range(n)]

# flat list: one contiguous block, index as r*m + c
flat = [0] * (n * m)
flat[r * m + c] = 5

# array.array: even more compact for huge homogeneous numeric data (C-level storage,
# no per-element Python int boxing overhead like a plain list has)
a = array.array('i', [0]) * (n * m)     # 'i' = signed int typecode
```

> **Gotcha:** a plain Python `list` stores **pointers to boxed int objects**, not raw ints — even `[0, 1, 2]` is three separate heap objects referenced by a list of pointers. `array.array` and `bytes`/`bytearray` store raw machine values contiguously, which matters for cache locality and memory when n is 10^6+. For most CP problems a plain list is fine; reach for `array`/`bytes` only when memory or cache-locality is the actual bottleneck (e.g. huge sieve, huge DP table of small values).

### 31.7 Rule (f): `lru_cache` overhead vs a manual table

`@lru_cache` is convenient but has real per-call overhead (hashing the argument tuple, dict lookup in the cache, LRU bookkeeping). For hot inner-loop DP with many calls, a manual `dict` or pre-sized `list`/2D array is often meaningfully faster:

```python
from functools import lru_cache

# convenient, some overhead per call
@lru_cache(maxsize=None)
def f(i, j):
    ...

# faster for very hot paths: manual memo table, no function-call/hash overhead
memo = {}
def f_manual(i, j):
    key = (i, j)
    if key in memo:
        return memo[key]
    ...
    memo[key] = result
    return result

# fastest when state space is small & dense: flat list DP table, no hashing at all
dp = [[-1] * (M + 1) for _ in range(N + 1)]
def f_table(i, j):
    if dp[i][j] != -1:
        return dp[i][j]
    ...
    dp[i][j] = result
    return dp[i][j]
```

Rule of thumb: `lru_cache` first (fastest to write, usually fine); if it TLEs, drop to a manual dict; if the state space is small/dense and indexable by small ints, use a flat/2D array table (no hashing at all — indexing beats hashing).

### 31.8 Rule (g): batch I/O

Never call `input()`/`print()` once per line for 10^5+ lines — read all input at once and write all output at once. Full detail is in Part VI (fast I/O); the short version:

```python
import sys
data = sys.stdin.buffer.read().split()   # read EVERYTHING once, split into tokens
it = iter(data)
n = int(next(it))
arr = [int(next(it)) for _ in range(n)]

out = []
# ... out.append(...) instead of print(...) ...
sys.stdout.write('\n'.join(out) + '\n')  # ONE write call
```

### 31.9 Rule (h): build strings via `join`, not `+=`

```python
# O(n^2) worst case: each += may reallocate and copy the whole string so far
s = ""
for w in words:
    s += w

# O(n): join builds the result in one pass
s = ''.join(words)
```

> **Gotcha:** CPython has a specific optimization that can make `s += w` in a tight loop closer to amortized O(n) *in some cases* (in-place resize when refcount == 1), but this is a CPython implementation detail you should never rely on — it doesn't apply on PyPy, and it doesn't always trigger even on CPython. Always use `''.join(...)` for building strings from many pieces; it's guaranteed fast everywhere.

### 31.10 Rule (i): `set`/`dict` for membership, not `list`

```python
big_list = list(range(10**6))
big_set = set(big_list)

x in big_list   # O(n) linear scan
x in big_set    # O(1) average — hash lookup
```

This single swap turns an accidental O(n^2) algorithm into O(n) — one of the most common "mystery TLE" causes in Python CP.

### 31.11 Complexity budget, adjusted for CPython

C++ rule-of-thumb tables (~10^8-10^9 simple ops/sec) do **not** transfer directly to Python. CPython does roughly **10^6-10^7 simple bytecode operations per second** — often cited as ~10-50x fewer "operations you can afford" than C++ for the same time limit, though the exact ratio depends heavily on whether the hot path hits C builtins or not.

| n | Complexity that's typically SAFE in CPython (1-2s TL) | Notes |
|---|---|---|
| ≤ 10 | anything, even O(n!) | brute force fine |
| ≤ 20 | O(2^n) | bitmask DP |
| ≤ 500 | O(n^3) | if simple inner ops |
| ≤ 5,000 | O(n^2) | borderline; use builtins in inner loop if possible |
| ≤ 10^5 | O(n log n) | standard sort/heap/BIT algorithms — fine |
| ≤ 10^6 | O(n) with **simple** per-element work, or O(n log n) if per-element work is truly trivial and uses builtins | risky with pure Python loops; strongly consider PyPy |
| ≤ 10^7-10^8 | only if the hot loop is essentially a single C-level builtin call (`sum`, `sorted`, numpy-style bulk op) | pure per-element Python loops will very likely TLE on CPython even at O(n) |

> **Gotcha:** an O(n^2) algorithm with n = 10^4 (10^8 total operations) is a classic "passes easily in C++, TLEs in CPython, passes in PyPy" boundary case. Know this pattern — when you see it, either switch judge language to PyPy, vectorize with builtins, or find an O(n log n) algorithm.

### 31.12 A concrete before/after

A frequent pattern: summing pairwise products or filtering-then-transforming a large array. Compare a naive loop against the builtin-heavy version:

```python
import time

n = 2 * 10**6
arr = list(range(n))

# --- naive Python loop ---
t0 = time.perf_counter()
total = 0
for x in arr:
    if x % 2 == 0:
        total += x * x
t1 = time.perf_counter()

# --- builtin/comprehension version ---
t2 = time.perf_counter()
total2 = sum(x * x for x in arr if x % 2 == 0)
t3 = time.perf_counter()

# total == total2 always; t3 - t2 is typically noticeably smaller than t1 - t0
# (generator expr avoids intermediate list; sum() runs its accumulation loop in C)
```

The two computations are equivalent, but the second avoids Python-level `total += ...` bytecode dispatch on every element by letting `sum()`'s C loop drive the iteration; only the *predicate/transform* stays in Python bytecode, and even that runs faster inside a generator expression than as an explicit `for` body with manual accumulation.

### 31.13 `numpy`, briefly

`numpy` is available on Codeforces and LeetCode and can give large speedups for **bulk numeric array operations** (vectorized add/multiply/compare over huge arrays) by pushing the entire loop into C/SIMD, well beyond what plain builtins achieve:

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5], dtype=np.int64)
b = a * 2 + 1          # vectorized: whole array processed in C, no Python-level loop at all
s = int(a.sum())       # cast back to a plain Python int for output/comparisons
```

> **Gotcha:** numpy has real overhead for *small* arrays (array creation, dtype dispatch) and its own set of surprises — fixed-width integer overflow (`np.int64` wraps instead of Python's arbitrary-precision `int`), and mixing `numpy` scalars with plain Python ints/floats in comparisons or as dict keys can behave subtly differently (`np.int64` is hashable and usually compares equal to `int`, but printing it shows `5` while `repr()` shows `np.int64(5)` in newer numpy — don't `print()` raw numpy scalars directly if the judge is strict about output format; cast with `int(x)` first). For most CP problems, plain lists + builtins (§31.3) are simpler and sufficient; reach for numpy specifically when you have a genuinely large, uniform numeric array and a bulk elementwise/matrix operation to perform.

### 31.14 When to just accept you need PyPy or C++

If you've already: chosen an optimal-complexity algorithm, batched I/O, pushed inner loops into builtins/comprehensions, avoided repeated attribute lookups, and used local variables — and it *still* TLEs on CPython — that is the signal to **switch the submission language to PyPy** (free, zero rewrite) before considering a full rewrite in C++. Reach for an actual C++ rewrite only when PyPy also fails (rare, but happens on the most constant-factor-sensitive problems, or ones needing heavy bignum work where PyPy's JIT can't help per §31.2).

---

## 32. Debugging, Exceptions and Running

### 32.1 Reading a traceback

Python tracebacks read **bottom-up**: the **last line** is the exception type and message (the actual error); the frames above it show the call chain with the **most recently called frame printed LAST**, right above the error line (i.e., top of the printed frames = where execution *started*, bottom of the printed frames = where it actually *crashed*).

```
Traceback (most recent call last):
  File "sol.py", line 12, in <module>
    main()
  File "sol.py", line 9, in main
    result = solve(arr)
  File "sol.py", line 4, in solve
    return arr[10]
IndexError: list index out of range
```

Read order: start at the bottom (`IndexError: list index out of range` — the actual problem), then look at the frame directly above it (`line 4, in solve` — exactly where it happened), then walk upward through the call chain (`solve` was called from `main`, `main` was called from module level) if you need to understand *how* execution got there.

> **vs C++/Java:** conceptually the same "most recent call last" convention as a Java stack trace read top-down (Java prints innermost frame *first*/top) — Python's ordering is the opposite visual direction (innermost/crash frame is at the **bottom**), which trips people coming from Java. C++ has no built-in equivalent at all unless you're using a debugger or a crash-handler library.

### 32.2 Common exceptions, decoded

| Exception | Plain-English meaning | Typical DSA cause |
|---|---|---|
| `IndexError` | list/string index outside valid range | off-by-one; `arr[n]` when valid indices are `0..n-1`; empty list access `arr[0]` |
| `KeyError` | dict lookup for a key that doesn't exist | `d[k]` on missing `k` — use `d.get(k, default)` or `defaultdict` to avoid |
| `TypeError` | operation applied to the wrong type | `'int' object is not subscriptable` (`x[0]` where `x` is an `int`, not a list); `unhashable type: 'list'` (using a `list` as a dict key or set element — lists are mutable so unhashable; convert to `tuple(list_)` first) |
| `ValueError` | right type, but an invalid *value* | `int('abc')` (not a valid numeral); `not enough values to unpack` (`a, b = [1]` when the right side has fewer/more items than the left) |
| `AttributeError` | calling a method/attribute that doesn't exist on that object | `None.append(...)` (forgot a function returns `None`, e.g. `arr = arr.sort()` — `list.sort()` sorts in place and returns `None`) |
| `RecursionError` | recursion depth exceeded the limit | deep DFS without raising `sys.setrecursionlimit` / going iterative (§30) |
| `ZeroDivisionError` | division or modulo by zero | `x % 0`, `x // 0`, `x / 0` — guard the denominator |
| `StopIteration` | `next()` called on an exhausted iterator with no default | manual `next(it)` past the end without a sentinel; usually only surfaces if you're hand-rolling iterator consumption (e.g. the token-reader pattern in §29/§31.8) |
| `IndentationError` / `TabError` | inconsistent indentation, or mixed tabs and spaces | pasting code from two sources with different indent styles — a Python-only class of bug since indentation is syntactically significant (no braces to fall back on) |
| `UnboundLocalError` | assigned to a name inside a function that Python therefore treats as local for the *entire* function body, but you read it before that assignment (or meant to modify the outer/global one) | see §32.4 for a worked example |

### 32.3 `unhashable type: 'list'` — a classic

```python
seen = set()
path = [1, 2]
seen.add(path)          # TypeError: unhashable type: 'list'

seen.add(tuple(path))   # fine — tuples are hashable (immutable)
```

This bites constantly in grid/BFS-visited-state problems where "state" is naturally a small list (e.g. `[row, col, direction]`) — always convert to a `tuple` before using it as a `set` element or `dict` key.

### 32.4 `UnboundLocalError` — the scope trap

```python
counter = 0

def increment():
    counter += 1   # UnboundLocalError: local variable 'counter' referenced before assignment
    return counter

increment()
```

Because `counter` is **assigned to** anywhere inside `increment`, Python decides at *compile time* that `counter` is a local variable for the whole function body — so the read on the right-hand side of `+=` (which happens before the assignment completes) refers to a local that has no value yet, not the outer/global one. Fix with `global` (module-level) or `nonlocal` (enclosing-function scope):

```python
counter = 0

def increment():
    global counter
    counter += 1
    return counter
```

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count       # needed: `count` lives in the enclosing function, not global
        count += 1
        return count
    return increment
```

> **vs C++/Java/JS:** Python decides "is this name local?" for an **entire function body at compile time** based on whether it's ever assigned anywhere in that body — not line-by-line like C++/Java block scoping, and not hoisting-with-`var` like old JS. This whole-function-scope rule is what makes the trap subtle: the error fires on a *read* that textually precedes the *write*, even though both are "the same line" in a `+=`.

### 32.5 Common-bug checklist for DSA Python

| Bug | Example | Fix |
|---|---|---|
| Mutable default argument | `def f(acc=[]): acc.append(1)` — `acc` persists across calls! | `def f(acc=None): acc = acc or []` |
| Shared rows via `*` | `grid = [[0]*m]*n` — all `n` rows are the **same** list object | `grid = [[0]*m for _ in range(n)]` |
| `is` vs `==` for values | `a is b` checks identity, not equality; small-int caching (-5..256) makes this *sometimes* work by accident for ints, then silently break for bigger numbers or strings | use `==` for value comparison, `is` only for `None`/singleton checks |
| `%` / `//` sign semantics | Python's `%` and `//` follow **floor division** (result has the sign of the divisor); C++'s `%`/`/` truncate toward zero (sign of the dividend) — `-7 % 3` is `2` in Python, `-1` in C++ | know the difference; use `math.fmod`-style manual truncation if you need C++ semantics |
| Forgetting `nonlocal` | inner closure tries to mutate an outer function's local, silently creates a new local instead (or raises `UnboundLocalError` if also read) | add `nonlocal x` |
| Late-binding closures in loops | `fns = [lambda: i for i in range(3)]` — all three lambdas return `2` (they share the *same* `i`, evaluated at call time, not creation time) | `fns = [lambda i=i: i for i in range(3)]` (default-arg trick captures the value at creation time) |
| `set()` vs `{}` | `{}` is an **empty dict**, not an empty set | use `set()` for an empty set; `{1, 2, 3}` non-empty set literal is fine |
| List as dict key | `d[[1,2]] = 3` | convert to tuple: `d[(1,2)] = 3` |
| Mutating a container while iterating it | `for k in d: del d[k]` → `RuntimeError: dictionary changed size during iteration` | iterate over `list(d.keys())` (a snapshot) if you need to mutate during the loop |
| `input()` per line at scale → TLE | reading 10^5+ lines with plain `input()` in a loop | batch-read with `sys.stdin` (Part VI, §31.8) |
| Recursion limit → `RecursionError` | deep DFS on a skewed tree/chain graph | §30: raise limit + threading trick, or go iterative |
| Integer `/` gives `float` | `5 / 2 == 2.5`, not `2` | use `//` for integer division: `5 // 2 == 2` |
| `sorted`/`.sort()` need `key=`, not `cmp` | Python 3 removed the `cmp=` parameter entirely (Python 2 had it) | use `key=lambda x: ...`, or `functools.cmp_to_key(cmp_fn)` if you truly need pairwise comparison logic |
| Off-by-one in `bisect` | `bisect_left` vs `bisect_right` give different insertion points for equal elements | `bisect_left`: leftmost valid insert point (first position `>=` x); `bisect_right`/`bisect`: rightmost (first position `> `x) — pick based on whether you want "first" or "last" among duplicates |
| Shallow copy of nested list | `b = a.copy()` or `b = a[:]` only copies the **outer** list; `b[0]` is still the *same* inner list object as `a[0]` | `import copy; b = copy.deepcopy(a)`, or rebuild: `b = [row[:] for row in a]` |
| `range` upper bound exclusive | `range(0, n)` yields `0..n-1`, never `n` | remember to add 1 for an inclusive upper bound: `range(0, n+1)` |
| `random.randint` upper bound **inclusive** | `random.randint(0, n)` can return `n` itself — unlike `range(0, n)` which excludes `n` | double check which of your loop bounds are inclusive; this specific inconsistency (`randint` inclusive vs `range` exclusive) is a well-known Python wart |

> **vs C++:** the `%`/`//` sign-semantics difference is the one that silently produces *wrong answers* rather than crashes — it's worth memorizing cold: Python's `%` always has the same sign as the divisor (`-7 % 3 == 2`, `7 % -3 == -2`), C++'s `%` has the same sign as the dividend (`-7 % 3 == -1` in C++). If you're porting a C++ modular-arithmetic formula, verify this explicitly rather than assuming it transfers.

### 32.6 Running locally

```bash
python3 sol.py < input.txt          # feed a file as stdin, exactly like a judge would
python3 sol.py < input.txt > out.txt
echo "5
1 2 3 4 5" | python3 sol.py         # pipe input directly
python3 -O sol.py < input.txt       # -O strips assert statements and __debug__ blocks (rarely needed for CP, but good to know)
```

### 32.7 `assert` for sanity checks

```python
def solve(n, arr):
    assert len(arr) == n, f"expected {n} elements, got {len(arr)}"
    ...
```

`assert` statements are stripped entirely when running with `python -O` (optimized mode) — never rely on an `assert` to perform required validation or side effects in submitted code; use it only as a development-time sanity check.

### 32.8 `pdb` / `breakpoint()` — the built-in debugger

```python
def solve(arr):
    total = 0
    for x in arr:
        breakpoint()   # [Python 3.7+] drops into pdb here; (Pdb) prompt: n=next, s=step, p x=print x, c=continue
        total += x
    return total
```

At the `(Pdb)` prompt: `n` (next line), `s` (step into), `c` (continue), `p expr` (print an expression), `l` (list surrounding source), `q` (quit). For CP this is usually overkill compared to `dprint(...)` (§29.5), but it's the right tool when a bug needs interactive inspection rather than a printed trace.

### 32.9 A tiny stress-testing harness

When a solution passes examples but fails on a hidden test, generate random small inputs, run a slow-but-obviously-correct brute force against your fast solution, and diff the outputs until you find a mismatch.

```python
# gen.py — random small test generator
import random
import sys

n = random.randint(1, 8)
print(n)
print(' '.join(str(random.randint(1, 10)) for _ in range(n)))
```

```python
# brute.py — obviously correct, possibly slow (e.g. O(n^2) or O(2^n))
import sys
data = sys.stdin.read().split()
n = int(data[0])
arr = list(map(int, data[1:1 + n]))
print(max(a + b for i, a in enumerate(arr) for b in arr[i+1:]))
```

```python
# fast.py — your optimized submission candidate
import sys
data = sys.stdin.read().split()
n = int(data[0])
arr = list(map(int, data[1:1 + n]))
arr.sort()
print(arr[-1] + arr[-2])
```

```bash
# stress.sh — loop: generate, run both, diff, stop on first mismatch
for i in $(seq 1 200); do
    python3 gen.py > in.txt
    python3 brute.py < in.txt > out_brute.txt
    python3 fast.py < in.txt > out_fast.txt
    if ! diff -q out_brute.txt out_fast.txt > /dev/null; then
        echo "MISMATCH on iteration $i"
        echo "--- input ---"; cat in.txt
        echo "--- brute ---"; cat out_brute.txt
        echo "--- fast ---"; cat out_fast.txt
        break
    fi
done
echo "done"
```

Run with `bash stress.sh`. The moment it prints `MISMATCH`, `in.txt` is a minimal-ish failing case you can reason about directly — far faster than staring at hidden test WA with no visibility.

### 32.10 Printing to stderr for debug output

```python
import sys

print("debug: n =", n, file=sys.stderr)   # never touches stdout / the graded answer stream
```

Combine this with the `DEBUG` flag pattern from §29.5 so debug prints are a single toggle away from silent, without needing to delete or comment out print statements before submitting.

### 32.11 `try`/`except` patterns worth knowing for CP

```python
# reading until EOF when the number of lines isn't given up front
import sys
for line in sys.stdin:
    line = line.strip()
    if not line:
        continue
    # process line

# or, catching an explicit EOF from a token reader
it = iter(sys.stdin.buffer.read().split())
try:
    while True:
        x = int(next(it))
        # process x
except StopIteration:
    pass
```

```python
from contextlib import suppress

d = {}
with suppress(KeyError):
    del d["missing"]   # no-op instead of crashing, when "might not exist" is expected
```

> **Gotcha:** don't reach for broad `except Exception:` blocks to "make errors go away" in CP code — it hides real bugs (an `IndexError` from a genuine off-by-one looks identical to an expected boundary case once swallowed) and a silently wrong answer is far harder to debug than a crash with a traceback. Catch the *specific* exception you expect (`KeyError`, `StopIteration`, `ValueError`), never a bare `except:`.

### 32.12 Quick sanity pass before submitting

- Removed/disabled `DEBUG = True` and any stray `print(...)` to stdout.
- Recursion: checked worst-case input *shape* (chain/skewed), not just size, against §30.6.
- I/O: reading via `sys.stdin`, writing via one `sys.stdout.write` / `'\n'.join`, not per-line `input()`/`print()` at scale (§31.8).
- Integer division intent verified: `//` vs `/` used correctly for the problem's semantics.
- Any list-as-dict-key or list-as-set-element converted to `tuple`.
- Nested-list initialization used a comprehension (`[[0]*m for _ in range(n)]`), not `[[0]*m]*n`.
- If PyPy is available for the judge and the loop is tight, submitted under PyPy rather than CPython.

---

# PART VIII — QUICK REFERENCE AND RECIPES

## 33. The "How Do I...?" Recipe Index

### Sorting

| Task | Python |
|---|---|
| Sort ascending | `a.sort()` / `sorted(a)` |
| Sort descending | `a.sort(reverse=True)` / `sorted(a, reverse=True)` |
| Sort by key | `a.sort(key=lambda x: x.val)` |
| Sort by multi-key (a asc, b desc) | `a.sort(key=lambda x: (x.a, -x.b))` |
| Multi-key, b not negatable (e.g. str) | `sorted(a, key=lambda x: (x.a, x.b), reverse=True)` won't mix directions — sort stably in passes, last key first: `a.sort(key=lambda x: x.b, reverse=True); a.sort(key=lambda x: x.a)` |
| Sort with a comparator | `from functools import cmp_to_key; a.sort(key=cmp_to_key(cmp))` where `cmp(x,y)` returns neg/0/pos |
| Argsort (indices that would sort a) | `idx = sorted(range(len(a)), key=lambda i: a[i])` |
| Sort dict by value | `sorted(d.items(), key=lambda kv: kv[1])` |
| Sort dict by value desc, top-k | `heapq.nlargest(k, d.items(), key=lambda kv: kv[1])` |
| Dedupe, preserve order | `list(dict.fromkeys(a))` |
| Dedupe, sorted order | `sorted(set(a))` |
| Reverse a list (new) | `a[::-1]` |
| Reverse a list (in place) | `a.reverse()` |
| Reverse a string | `s[::-1]` (str has no `.reverse()`) |
| Check sorted | `all(a[i] <= a[i+1] for i in range(len(a)-1))` |

### Max / Min / Aggregation

| Task | Python |
|---|---|
| Max / min of iterable | `max(a)` / `min(a)` |
| Max by key | `max(a, key=lambda x: x.val)` |
| Argmax (index of max) | `max(range(len(a)), key=lambda i: a[i])` |
| Max with default (empty-safe) | `max(a, default=None)` |
| Kth largest | `heapq.nlargest(k, a)[-1]` |
| Kth smallest | `heapq.nsmallest(k, a)[-1]` |
| Top-k largest | `heapq.nlargest(k, a)` |
| Sum (never overflows) | `sum(a)` |
| Sum with start (avoid float default) | `sum(a, 0)` |
| Product | `math.prod(a)` |
| Frequency count | `from collections import Counter; c = Counter(a)` |
| Most common element | `c.most_common(1)[0]` |
| Running / prefix sums | `list(itertools.accumulate(a))` |
| Running max | `list(itertools.accumulate(a, max))` |

### Dicts / Graphs

| Task | Python |
|---|---|
| Adjacency list | `g = defaultdict(list); g[u].append(v)` |
| Iterate dict items | `for k, v in d.items():` |
| Iterate keys only | `for k in d:` (or `d.keys()`) |
| Iterate values only | `for v in d.values():` |
| Get with default (no KeyError) | `d.get(k, default)` |
| Increment-or-init | `d[k] = d.get(k, 0) + 1` or `Counter`/`defaultdict(int)` |
| Invert a dict | `{v: k for k, v in d.items()}` |
| Merge two dicts (3.9+) | `d1 | d2` (right wins on conflict) |
| Merge in place | `d1.update(d2)` |

### Sorted structure floor/ceiling

| Task | Python |
|---|---|
| Floor (largest ≤ x) in sorted list `a` | `i = bisect.bisect_right(a, x) - 1; floor = a[i] if i >= 0 else None` |
| Ceiling (smallest ≥ x) in sorted list `a` | `i = bisect.bisect_left(a, x); ceil = a[i] if i < len(a) else None` |
| Floor/ceil with `SortedList` | `sl.irange(maximum=x, reverse=True)` first item / `sl.irange(minimum=x)` first item |
| Floor/ceil key in `SortedDict` | same via `sd.keys()` as a `SortedKeysView` + `irange` |

### Heaps

| Task | Python |
|---|---|
| Min-heap | `import heapq; h=[]; heapq.heappush(h,x); heapq.heappop(h)` |
| Max-heap | push negated: `heapq.heappush(h, -x)`; pop and negate back |
| Heapify existing list | `heapq.heapify(a)` — O(n), in place |
| Heap of tuples, tie-break on 2nd elem | if payload not comparable, add a counter: `heapq.heappush(h, (priority, counter, item)); counter += 1` |
| Peek min | `h[0]` (don't pop) |
| Push-then-pop efficiently | `heapq.heappushpop(h, x)` (faster than push+pop) |

### 2D / 3D / Grids

| Task | Python |
|---|---|
| 2D list init (correct) | `[[0]*m for _ in range(n)]` |
| 2D list init (BUGGY — shared rows) | `[[0]*m]*n` ← all rows are the *same* list object, mutating one mutates all |
| 3D list init | `[[[0]*p for _ in range(m)] for _ in range(n)]` |
| 4-directional deltas | `for dr, dc in ((0,1),(0,-1),(1,0),(-1,0)):` |
| 8-directional deltas | `for dr in (-1,0,1):\n    for dc in (-1,0,1):\n        if dr or dc:` |
| In-bounds check | `0 <= nr < rows and 0 <= nc < cols` |
| Transpose grid | `list(zip(*grid))` (rows become tuples) |
| Flatten 2D list | `[x for row in g for x in row]` |
| Rotate grid 90° CW | `list(zip(*grid[::-1]))` |

### Strings

| Task | Python |
|---|---|
| str → int | `int(s)` |
| str → int, given base | `int(s, 16)` / `int(s, 2)` |
| int → str | `str(n)` |
| Split on whitespace | `s.split()` |
| Split on char | `s.split(',')` |
| Split at most k times | `s.split(',', k)` |
| Join list of str | `''.join(parts)` |
| Join with separator | `','.join(parts)` |
| char → code point | `ord(c)` |
| code point → char | `chr(n)` |
| letter → 0-indexed (a=0) | `ord(c) - ord('a')` |
| Upper / lower | `s.upper()` / `s.lower()` |
| Strip whitespace | `s.strip()` (also `.lstrip()`, `.rstrip()`) |
| Palindrome check | `s == s[::-1]` |
| Substring check | `sub in s` |
| Find index or -1 (no exception) | `s.find(sub)` (use `.index(sub)` if you want a raised error) |
| Count occurrences | `s.count(sub)` |
| Replace | `s.replace(old, new)` |
| Build string efficiently (avoid `+=` in loop) | `parts = []; ...; parts.append(x); ''.join(parts)` |
| Check alpha/digit/alnum | `s.isalpha()` / `s.isdigit()` / `s.isalnum()` |
| Pad with zeros | `s.zfill(width)` |
| Format with width/precision | `f"{x:5.2f}"` |

### Combinatorics / itertools

| Task | Python |
|---|---|
| All permutations | `itertools.permutations(a)` |
| Permutations of length r | `itertools.permutations(a, r)` |
| All combinations (r) | `itertools.combinations(a, r)` |
| Combinations with replacement | `itertools.combinations_with_replacement(a, r)` |
| Cartesian product | `itertools.product(a, b)` |
| Cartesian power (self × self × ...) | `itertools.product(a, repeat=k)` |
| All subsets via bitmask | `for mask in range(1 << n):` |
| All subsets via itertools | `chain.from_iterable(combinations(a, r) for r in range(len(a)+1))` |
| Pairwise (3.10+) | `itertools.pairwise(a)` → consecutive `(a[i], a[i+1])` |

### Binary search / bisect

| Task | Python |
|---|---|
| Lower bound (first idx with `a[i] >= x`) | `bisect.bisect_left(a, x)` |
| Upper bound (first idx with `a[i] > x`) | `bisect.bisect_right(a, x)` (== `bisect.bisect`) |
| Insert keeping sorted | `bisect.insort(a, x)` (O(n) due to shift, but bisect part is O(log n)) |
| Count of x in sorted a | `bisect_right(a, x) - bisect_left(a, x)` |
| Count values in `[l, r]` inclusive | `bisect_right(a, r) - bisect_left(a, l)` |
| Binary search on answer | write `ok(x) -> bool` monotonic, then bisect manually or `bisect_left(range(lo,hi), True, key=ok)` (3.10+ `key=`) |

### Math / Modular

| Task | Python |
|---|---|
| gcd / lcm | `math.gcd(a, b)` / `math.lcm(a, b)` (3.9+) |
| gcd/lcm of a list | `math.gcd(*a)` / `math.lcm(*a)` |
| Fast modpow | `pow(a, b, m)` — built-in, C-fast |
| Modular inverse (m prime or coprime) | `pow(a, -1, m)` (3.8+) |
| Modular multiply | `(a * b) % m` — Python ints never overflow, no need for mulmod tricks |
| Safe modulo | `x % m` already returns `[0, m)` for positive `m`, even if `x` is negative |
| Integer sqrt (floor) | `math.isqrt(n)` — exact, no float error |
| Check perfect square | `r = math.isqrt(n); r*r == n` |
| nCr / nPr | `math.comb(n, r)` / `math.perm(n, r)` (3.8+) |
| Factorial | `math.factorial(n)` |
| Check power of two | `x > 0 and (x & (x-1)) == 0` |
| Count set bits | `x.bit_count()` (3.10+) or `bin(x).count('1')` |
| Iterate submasks of `mask` | `sub = mask;\nwhile True:\n    ...\n    if sub == 0: break\n    sub = (sub - 1) & mask` |

### Misc / plumbing

| Task | Python |
|---|---|
| Swap variables | `a, b = b, a` |
| Clamp x to `[lo, hi]` | `max(lo, min(hi, x))` |
| Random int, INCLUSIVE both ends | `random.randint(lo, hi)` |
| Random int, exclusive upper | `random.randrange(lo, hi)` |
| Shuffle a list in place | `random.shuffle(a)` |
| Measure runtime | `import time; t0=time.perf_counter(); ...; print(time.perf_counter()-t0)` |
| Read all stdin fast | `import sys; data = sys.stdin.buffer.read().split()` |
| Read a grid of chars | `grid = [line.strip() for line in sys.stdin]` or `[input() for _ in range(n)]` |
| Read a grid of ints | `grid = [list(map(int, input().split())) for _ in range(n)]` |
| Print all output at once (fast) | `print('\n'.join(map(str, res)))` |
| Print a list space-separated | `print(*a)` |
| Print a list, one per line | `print(*a, sep='\n')` |
| Memoize a function | `from functools import cache; @cache\ndef f(n): ...` |
| Memoize with size limit | `@lru_cache(maxsize=None)` |

### Input / Output patterns

| Task | Python |
|---|---|
| Read one int | `n = int(input())` |
| Read one line of ints | `a = list(map(int, input().split()))` |
| Read n lines of ints (one int per line) | `a = [int(input()) for _ in range(n)]` |
| Read n lines, each a list of ints | `rows = [list(map(int, input().split())) for _ in range(n)]` |
| Read until EOF | `for line in sys.stdin:` or `while (line := sys.stdin.readline()):` |
| Read two ints on one line | `a, b = map(int, input().split())` |
| Read a single line fast | `sys.stdin.readline().strip()` |
| Tokenize whole input once, consume with an iterator | `it = iter(sys.stdin.buffer.read().split()); nxt = lambda: next(it); x = int(nxt())` |
| Print without trailing newline | `print(x, end='')` |
| Print with custom separator | `print(a, b, c, sep=', ')` |
| Print float with fixed precision | `print(f"{x:.6f}")` |
| Print in scientific notation | `print(f"{x:.3e}")` |
| Print with thousands separator | `print(f"{n:,}")` |
| Print a 2D grid, one row per line | `print('\n'.join(' '.join(map(str, row)) for row in grid))` |
| Flush output immediately (interactive problems) | `print(x, flush=True)` |

### Two Pointers / Sliding Window

| Task | Python |
|---|---|
| Opposite-end two pointers | `lo, hi = 0, len(a) - 1;\nwhile lo < hi:\n    ...\n    lo += 1; hi -= 1` |
| Fixed-size sliding window sum | `window = sum(a[:k]);\nfor i in range(k, len(a)):\n    window += a[i] - a[i-k]` |
| Variable-size shrinking window | `lo = 0;\nfor hi in range(len(a)):\n    # expand with a[hi]\n    while not ok(lo, hi):\n        lo += 1  # shrink` |
| Fast/slow pointer (cycle detection) | `slow = fast = head;\nwhile fast and fast.next:\n    slow = slow.next; fast = fast.next.next\n    if slow is fast: cycle_found` |

### Common checks & transforms

| Task | Python |
|---|---|
| Anagram check | `Counter(s1) == Counter(s2)` (or `sorted(s1) == sorted(s2)`) |
| Is subsequence of | `it = iter(s); all(c in it for c in sub)` |
| All unique characters | `len(set(s)) == len(s)` |
| Rotate list left by k | `a[k:] + a[:k]` |
| Rotate list right by k | `a[-k:] + a[:-k]` (guard `k == 0` separately — `a[-0:]` is the whole list) |
| Rotation check (`s2` is a rotation of `s1`) | `len(s1) == len(s2) and s2 in s1 + s1` |
| Matrix multiply (dense, small) | `C = [[sum(A[i][k]*B[k][j] for k in range(len(B))) for j in range(len(B[0]))] for i in range(len(A))]` |
| Count inversions (data structure route) | BIT/merge-sort — see Part VII pattern 17, no O(n log n) one-liner |
| Convert list of digits to int | `int(''.join(map(str, digits)))` |
| Convert int to list of digits | `[int(c) for c in str(n)]` |
| Base conversion (to base b, b ≤ 36) | manual loop: `divmod(n, b)` repeatedly; no built-in beyond base 2/8/16 (`bin`/`oct`/`hex`) |
| Binary string → int | `int(s, 2)` |
| Int → binary string, fixed width | `format(n, '08b')` or `f"{n:08b}"` |

### Iterators / generators

| Task | Python |
|---|---|
| Index + value together | `for i, x in enumerate(a):` |
| Index from a given start | `for i, x in enumerate(a, start=1):` |
| Walk two lists together | `for x, y in zip(a, b):` |
| Walk lists of unequal length, pad with fill | `itertools.zip_longest(a, b, fillvalue=0)` |
| Iterate in reverse | `for x in reversed(a):` |
| Short-circuit "any match" | `any(cond(x) for x in a)` |
| Short-circuit "all match" | `all(cond(x) for x in a)` |
| Next item with a default (no StopIteration) | `next(it, default)` |
| Chain multiple iterables | `itertools.chain(a, b, c)` |
| Lazy vs eager memory | list comprehension `[f(x) for x in a]` builds the whole list now O(n) memory; generator `(f(x) for x in a)` is lazy O(1) memory but single-pass only |

### Graph / DP quick recipes

```python
# BFS shortest distance (unweighted) from src
def bfs(graph, src):
    dist = {src: 0}
    dq = deque([src])
    while dq:
        u = dq.popleft()
        for v in graph[u]:
            if v not in dist:
                dist[v] = dist[u] + 1
                dq.append(v)
    return dist

# Dijkstra (weighted, non-negative)
def dijkstra(graph, src):               # graph[u] = [(v, w), ...]
    dist = {src: 0}
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist.get(u, float('inf')):
            continue
        for v, w in graph[u]:
            nd = d + w
            if nd < dist.get(v, float('inf')):
                dist[v] = nd
                heapq.heappush(pq, (nd, v))
    return dist

# Topological sort (Kahn's algorithm)
def topo_sort(n, graph):                # graph[u] = [v, ...], nodes 0..n-1
    indeg = [0] * n
    for u in range(n):
        for v in graph[u]:
            indeg[v] += 1
    dq = deque(u for u in range(n) if indeg[u] == 0)
    order = []
    while dq:
        u = dq.popleft()
        order.append(u)
        for v in graph[u]:
            indeg[v] -= 1
            if indeg[v] == 0:
                dq.append(v)
    return order if len(order) == n else None   # None => cycle

# Union-Find / DSU with path compression + union by rank
class DSU:
    def __init__(self, n):
        self.p = list(range(n))
        self.r = [0] * n
    def find(self, x):
        while self.p[x] != x:
            self.p[x] = self.p[self.p[x]]        # path halving
            x = self.p[x]
        return x
    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False
        if self.r[ra] < self.r[rb]:
            ra, rb = rb, ra
        self.p[rb] = ra
        if self.r[ra] == self.r[rb]:
            self.r[ra] += 1
        return True

# LIS length in O(n log n) via bisect (patience sorting)
def lis_length(a):
    tails = []
    for x in a:
        i = bisect_left(tails, x)
        if i == len(tails):
            tails.append(x)
        else:
            tails[i] = x
    return len(tails)

# 0/1 knapsack DP, O(n * capacity), rolling 1D array
def knapsack(weights, values, cap):
    dp = [0] * (cap + 1)
    for w, v in zip(weights, values):
        for c in range(cap, w - 1, -1):   # iterate DOWN to keep it 0/1 (not unbounded)
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[cap]

# Edit distance DP, O(n*m)
def edit_distance(s1, s2):
    n, m = len(s1), len(s2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    for i in range(n + 1):
        dp[i][0] = i
    for j in range(m + 1):
        dp[0][j] = j
    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
    return dp[n][m]
```

### Choosing a container — decision cheat sheet

| Need | Container |
|---|---|
| FIFO queue (BFS) | `deque` |
| LIFO stack | `list` (`.append`/`.pop`) |
| O(1) average membership test | `set` |
| Frequency / multiset counting | `Counter` |
| Repeatedly extract the min (or max) | `heapq` (negate for max) |
| Order statistics: floor/ceil/rank, sorted iteration with mutation | `sortedcontainers.SortedList` |
| Sorted map with floor/ceil on keys | `sortedcontainers.SortedDict` |
| Insertion-order key→value mapping | `dict` (ordered since 3.7) |
| Auto-vivifying nested structure (trie, adjacency list) | `defaultdict(list)` / `defaultdict(dict)` / recursive `defaultdict` |
| Immutable, hashable sequence (dict key, set element) | `tuple` |
| Fixed-type numeric buffer, memory-tight | `array.array` (rare in CP; list is usually fine) |

## 34. Master Complexity Table

### Containers

| Container | Index/Access | Insert end | Insert front/middle | Delete end | Delete front/middle | `x in c` | Iterate | Notes / Python caveat |
|---|---|---|---|---|---|---|---|---|
| `list` | O(1) | O(1)* amortized `.append` | O(n) `.insert(0,x)` | O(1) `.pop()` | O(n) `.pop(0)` | O(n) | O(n) | `.pop(0)` and `.insert(0,x)` are O(n) — use `deque` for queue behavior |
| `tuple` | O(1) | — immutable | — | — | — | O(n) | O(n) | hashable if elements are; use as dict key / set element |
| `str` | O(1) | — immutable | — | — | — | O(n) substr search (C-optimized) | O(n) | building via `+=` in a loop is O(n²) total — use list+join |
| `dict` | O(1) avg key lookup | O(1) avg `d[k]=v` | — | O(1) avg `del d[k]` | — | O(1) avg (`in` checks keys) | O(n), insertion order preserved (3.7+) | worst-case O(n) on hash collisions (adversarial, rare in CP) |
| `defaultdict` | same as dict | same | — | same | — | O(1) avg | O(n) | accessing missing key via `[]` **creates** it — can silently bloat / change `len` |
| `Counter` | O(1) avg | O(1) avg | — | O(1) avg | — | O(1) avg | O(n) | missing key via `[]` returns 0 without creating entry (unlike defaultdict) |
| `set` | — | O(1) avg `.add` | — | O(1) avg `.remove`/`.discard` | — | O(1) avg | O(n) | `.remove` raises KeyError if absent; `.discard` doesn't |
| `frozenset` | — | immutable | — | — | — | O(1) avg | O(n) | hashable → usable as dict key / set element |
| `deque` | O(1) ends, O(n) middle | O(1) `.append`/`.appendleft` | O(1) at ends | O(1) `.pop`/`.popleft` | O(1) at ends | O(n) | O(n) | the correct queue/stack for both ends; indexing middle is O(n) unlike list |
| `heapq` (list-based) | O(1) peek `h[0]` | O(log n) `heappush` | — | O(log n) `heappop` (min only) | — | O(n) | O(n) | no O(log n) arbitrary delete/decrease-key built in; only a binary min-heap |
| `SortedList` | O(log n) | O(log n) `.add` | — | O(log n) `.remove` | — | O(log n) | O(n) | third-party (`sortedcontainers`); real order stats: `.bisect_left`, `[i]` O(log n)-ish |
| `SortedDict` | O(log n) by key | O(log n) | — | O(log n) | — | O(log n) | O(n) sorted by key | closest Python has to `std::map`/TreeMap |
| `SortedSet` | O(log n) | O(log n) | — | O(log n) | — | O(log n) | O(n) sorted | closest Python has to `std::set` |
| `array` (`array.array`) | O(1) | O(1) amortized append | O(n) | O(1) pop end | O(n) | O(n) | O(n) | typed, compact, but arithmetic still boxes to Python ints one at a time — no numeric speedup over list for CP loops |

*list `.append` is amortized O(1); occasional O(n) resize.

### Function / algorithm complexities

| Function | Complexity | Notes |
|---|---|---|
| `list.sort()` / `sorted()` | O(n log n) | Timsort; stable; exploits existing runs → close to O(n) on nearly-sorted input |
| `bisect_left`/`bisect_right` | O(log n) | requires already-sorted sequence |
| `heapq.heappush`/`heappop` | O(log n) | |
| `heapq.heapify` | O(n) | not O(n log n) |
| `heapq.nlargest(k, a)`/`nsmallest` | O(n log k) | better than `sorted(a)[-k:]` when k ≪ n |
| `Counter(a)` | O(n) | |
| `Counter.most_common()` | O(n log n) | full sort; `most_common(k)` is O(n log k) |
| `in` on list/tuple/str | O(n) | linear scan / substring search |
| `in` on dict/set | O(1) avg | hash lookup |
| `itertools.permutations(a)` | O(n!) items, O(n) to produce each | total n! · n |
| `itertools.combinations(a, r)` | O(C(n,r)) items | |
| `dict.fromkeys` dedupe | O(n) | |
| `min`/`max`/`sum` | O(n) | |
| `str.join` | O(total length) | |
| `str +=` in a loop | O(n) per op → O(n²) total | strings are immutable, each `+=` copies |

### List method complexities (the ones that surprise C++/Java devs)

| Method | Complexity | Notes |
|---|---|---|
| `a.append(x)` | O(1) amortized | occasional O(n) resize when capacity doubles |
| `a.pop()` | O(1) | removes/returns last element |
| `a.pop(0)` | O(n) | shifts every remaining element down — use `deque.popleft()` instead |
| `a.insert(0, x)` | O(n) | shifts every element up — use `deque.appendleft()` instead |
| `a.index(x)` | O(n) | linear scan for first match; raises `ValueError` if absent |
| `a.remove(x)` | O(n) | linear scan to find `x`, then O(n) shift |
| `a.count(x)` | O(n) | linear scan |
| `a[i:j] = other` (slice assignment) | O(n) | can insert/delete/replace a run in one call |
| `a in b` (sublist membership via `in`) | O(n·m) worst case | `str in str` is optimized in C; `list in list` is not the same fast path |
| `list(a)` (copy) | O(n) | shallow copy |
| `a[:]` (slice copy) | O(n) | shallow copy, same cost as `list(a)` |

### Iteration helper complexities

| Helper | Complexity | Notes |
|---|---|---|
| `enumerate(a)` | O(1) to create, O(n) to exhaust | lazy, no extra list built |
| `zip(a, b)` | O(1) to create, O(min(len)) to exhaust | stops at shortest iterable |
| `reversed(a)` | O(1) to create (for sequences), O(n) to exhaust | needs `__reversed__`/`__len__`+`__getitem__`; not for arbitrary generators |
| `itertools.chain(a, b)` | O(1) to create, O(total) to exhaust | avoids materializing `a + b` |
| dict `.keys()`/`.values()`/`.items()` | O(1) to create | live views, not copies — reflect later mutation, and disallow structural mutation mid-iteration |
| `deque.rotate(k)` | O(k) | rotates in place; O(min(k, n-k)) internally |

## 35. n → Required Complexity Cheat Sheet

CPython pure-Python loop throughput is roughly **10⁶–10⁷ simple operations/sec** — much slower than C++'s ~10⁸–10⁹. Budget accordingly: a technique that's "fine" in C++ at a given complexity may **TLE in CPython** at the same n. When possible, push the hot loop into C-level builtins (`sum`, `sorted`, `map`, list/set comprehensions, `itertools`, `bisect`, `heapq`, NumPy) rather than a manual `for`.

| n (input size) | Required complexity | Technique | DSA pattern ref |
|---|---|---|---|
| n ≤ 10 | O(n!), O(2ⁿ·n) | brute-force permutations, full subset search + check | 24 (backtracking) |
| n ≤ 20 | O(2ⁿ) | bitmask DP — Python `int` mask has **no width limit**, so masks past 63 bits still work natively, no `long long` tricks needed | 21 (bitmask DP) |
| n ≤ 100 | O(n³)–O(n⁴) | Floyd-Warshall, cubic DP, brute triple loop | 15 (graphs), 20 (DP) |
| n ≤ 500 | O(n³) | still cubic DP/matrix ops, watch the constant | 20 |
| n ≤ 5,000 | O(n²) | double loop DP, simple pairwise checks | 20 |
| n ≤ 10⁵ | O(n log n) | sort-based, binary search, heap, segment tree, BIT | 06 (bin search), 12 (heaps), 17 (segment tree/BIT) |
| n ≤ 10⁶ | O(n) or O(n log n) with tiny constant | sieve, two pointers, prefix sums, single-pass hashing | 04 (two pointers), 05 (prefix sums) |
| n ≤ 10⁹ | O(log n), O(√n), or closed-form math | binary search on value, fast modpow, gcd, number theory | 06, 26 (math/number theory) |

**Python-specific notes:**
- An O(n²) loop with n = 10⁴ (10⁸ inner iterations) is borderline-to-TLE in CPython (~10–60s) but often passes in **PyPy** (JIT-compiled, near-C speed) — select PyPy3 on Codeforces/AtCoder for tight numeric loops.
- Vectorize where possible: `sum(...)`, comprehensions, `map`, and `itertools` run their inner loop in C, often 10-50x faster than an equivalent explicit `for`.
- Python's arbitrary-precision `int` means "big number" subproblems (factorials of 1000, products that overflow 64-bit) that need `BigInteger`/manual bignum arithmetic in C++/Java are **free** in Python — no special handling required.
- Recursion has overhead beyond C++ function calls; deep recursive DFS at n = 10⁵ may need iterative conversion or `sys.setrecursionlimit` + a thread with a larger stack (Part VII §30) even when the *complexity* is fine.

### Rough CPython throughput benchmarks (order-of-magnitude, single core)

| Operation | Approx throughput | Note |
|---|---|---|
| Bare arithmetic in a `for` loop | ~3-5 × 10⁷ /sec | interpreter dispatch overhead dominates |
| Function call (Python-level) | ~10⁷ calls/sec | each call has real overhead — avoid deep call chains in hot loops |
| `list.append` in a loop | ~2 × 10⁷ /sec | still faster to build via comprehension |
| List comprehension vs equivalent `for`+`append` | ~1.5-2x faster | comprehension body runs in a specialized bytecode loop |
| `dict`/`set` get or set | ~10⁷ /sec | hash + probe, C-implemented |
| String concatenation via `+=` in a loop | degrades to effectively O(n) per iteration | avoid entirely — use list + `''.join` |
| Built-ins (`sum`, `sorted`, `map`, `min`, `max`) over a list | near C speed | inner loop is C, not bytecode — always prefer these over manual loops |

### CPython vs PyPy speedup (typical, problem-dependent)

| Workload | Typical PyPy speedup over CPython |
|---|---|
| Tight numeric loops, manual arithmetic | 10-100x |
| Heavy recursion / function-call-bound code | 5-30x |
| String-heavy or regex-heavy code | modest, 2-5x (already partly C-backed) |
| Code dominated by NumPy/C-extension calls | little to none — the C extension is already fast; PyPy doesn't speed up the C part |

## 36. The Top 30 Python-DSA Mistakes That Cost You Problems

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | Mutable default argument: `def f(x=[]):` | silent wrong answer across calls (list persists between calls) | `def f(x=None):\n    if x is None: x = []` |
| 2 | `[[0]*m]*n` for a 2D grid | silent wrong answer — all rows are the same object, one mutation changes every row | `[[0]*m for _ in range(n)]` |
| 3 | `//` assumed to truncate toward zero | wrong answer on negative operands (`-7 // 2 == -4`, not `-3`) | know it **floors**; for trunc-toward-zero use `int(a / b)` or `math.trunc(a / b)` |
| 4 | `%` assumed to follow dividend's sign like C++/Java/JS | wrong answer on negative operands (`-7 % 3 == 2` in Python, `-1` in C++) | this is usually what you *want* for CP (non-negative mod); be aware it differs |
| 5 | `is` used instead of `==` for value equality | intermittent silent bugs — works for small cached ints (-5..256) then breaks for larger ones | always use `==` for value comparison; reserve `is` for `None`/identity |
| 6 | Forgetting `nonlocal`/`global` when reassigning an outer variable in a nested function | `UnboundLocalError` | add `nonlocal x` (closure) or `global x` (module scope) before reassigning |
| 7 | Late-binding closures in a loop (`lambda: i` inside `for i in range(n)`) | all closures see the final value of `i` | default-arg capture: `lambda i=i: i` |
| 8 | `{}` used for an empty set | `TypeError` or silent wrong type — `{}` is an empty **dict** | `set()` for an empty set |
| 9 | Using a `list` as a dict key or set element | `TypeError: unhashable type: 'list'` | use `tuple(x)` instead |
| 10 | Modifying a dict/list while iterating over it | `RuntimeError: dictionary changed size during iteration` or skipped elements | iterate over a copy: `for k in list(d):` / build a new collection |
| 11 | `input()`/`print()` called once per line in a big loop | TLE — each call has real overhead in CPython | batch: `sys.stdin.buffer.read().split()` for input, `'\n'.join(...)` + one `print` for output |
| 12 | Deep recursion hitting the default 1000-frame limit | `RecursionError: maximum recursion depth exceeded` | `sys.setrecursionlimit(1<<20)` **and** run in a thread with a bigger stack for very deep DFS |
| 13 | Assuming `sorted`/`list.sort` take a `cmp=` comparator like old Python 2 | `TypeError: sort() takes no positional arguments` (or just wrong — no such kwarg in 3) | Python 3 only has `key=`; wrap a comparator with `functools.cmp_to_key` |
| 14 | Treating `heapq` as a max-heap | wrong element popped — `heapq` is **min-only** | negate values on push/pop for a max-heap |
| 15 | Heap of tuples crashing when the 2nd element isn't comparable (e.g. custom objects, or ties on 1st element) | `TypeError: '<' not supported between instances of ...` | push `(priority, tie_breaker_counter, item)` so ties never compare `item` |
| 16 | Assuming `random.randint(a, b)` excludes `b` | off-by-one — it's **inclusive** on both ends, unlike `random.randrange` | use `random.randrange(a, b)` if you want exclusive upper bound |
| 17 | `int('3.5')` to parse a numeric string that might be a float | `ValueError: invalid literal for int()` | `int(float(s))` if fractional strings are possible |
| 18 | `x in some_list` inside a hot loop | TLE — O(n) per check, O(n²) or worse overall | convert to `set` first: `x in some_set` is O(1) avg |
| 19 | `new = old` or `new = old[:]` on a **nested** list, then mutating | shallow copy — inner lists still shared, mutating `new[0][0]` also changes `old[0][0]` | `copy.deepcopy(old)` or rebuild: `[row[:] for row in old]` |
| 20 | Off-by-one on `range(n)` (assuming inclusive upper bound) | loop runs one short — `range(n)` is `0..n-1` | `range(n+1)` if you need `0..n` inclusive |
| 21 | `/` used where integer division is intended | wrong type (float) propagating into indices → `TypeError: list indices must be integers` | use `//` for integer division |
| 22 | `round(x)` expected to always round half up | surprising results on `.5` values — Python uses **banker's rounding** (round-half-to-even): `round(0.5)==0`, `round(1.5)==2` | for CP "round half up" semantics, use `math.floor(x + 0.5)` (careful with negatives) or work in integers |
| 23 | Misremembering `~x` as just "flip sign" | wrong bit arithmetic — `~x == -(x + 1)` (two's-complement-style, arbitrary precision) | know the identity explicitly; test with small values before trusting |
| 24 | Not stripping the newline from `sys.stdin.readline()` | trailing `'\n'` corrupts string comparisons / `int()` parse in some cases | `.strip()` or `.rstrip('\n')` after `readline()` |
| 25 | Submitting plain CPython on a judge where the intended complexity needs PyPy speed | TLE despite "correct" complexity | select PyPy3 on Codeforces/AtCoder when available for tight loops |
| 26 | Comparing floats with `==` | silent wrong answer from accumulated floating-point error | compare with a tolerance: `abs(a - b) < 1e-9`, or avoid floats (use `Fraction`/ints) entirely |
| 27 | Off-by-one with `bisect_left` vs `bisect_right` | wrong insertion point / wrong count in range queries | `bisect_left` = first index where value could go before duplicates (lower bound); `bisect_right` = after duplicates (upper bound) — pick deliberately |
| 28 | Reading with `defaultdict` when you actually need "does this key exist" | key count / `len(d)` inflated because a lookup via `[]` auto-creates missing keys | use `.get(k)` or `k in d` for read-only checks; reserve `d[k]` writes for when you intend to insert |
| 29 | Mixing tabs and spaces for indentation | `TabError`/`IndentationError`, sometimes only on some lines | use spaces consistently (4 spaces is standard); configure editor to insert spaces for Tab key |
| 30 | Assuming `a, b = b, a` needs a temp variable / gets it wrong under multiple simultaneous swaps | none if done right, but manual temp-var swaps in translated C++ code sometimes introduce bugs (`a=b; b=a` overwrites `a` before use) | trust the tuple-unpack swap — the right side is fully evaluated before assignment |

### Illustrations of the highest-impact mistakes

```python
# #1 Mutable default argument — the [] is created ONCE, at def time
def bad(x=[]):
    x.append(1)
    return x
bad(); bad()                      # [1] then [1, 1] — NOT [1] twice
def good(x=None):
    if x is None:
        x = []
    x.append(1)
    return x

# #2 Shared rows — mutating one row mutates all of them
grid_bad = [[0] * 3] * 3
grid_bad[0][0] = 9                # every row's first element is now 9
grid_good = [[0] * 3 for _ in range(3)]
grid_good[0][0] = 9               # only row 0 changes

# #3 / #4 Floor division and sign-following modulo
-7 // 2                           # -4 (floors toward -inf, not -3)
-7 % 3                            # 2  (result takes the sign of the divisor)

# #5 is vs == — small ints are cached, larger ones are not
a, b = 200, 200
a is b                            # True by accident (CPython caches -5..256)
a, b = 1000, 1000
a is b                            # False — do not rely on this, use ==

# #7 Late-binding closure — all lambdas capture the SAME variable i
fns_bad = [lambda: i for i in range(3)]
[f() for f in fns_bad]            # [2, 2, 2], not [0, 1, 2]
fns_good = [lambda i=i: i for i in range(3)]
[f() for f in fns_good]           # [0, 1, 2]

# #14 heapq is min-only — negate for a max-heap
import heapq
h = []
for x in (3, 1, 4, 1, 5):
    heapq.heappush(h, -x)
-heapq.heappop(h)                 # 5, the max

# #15 Heap of tuples — add a tie-breaker so payloads never get compared
counter = 0
pq = []
heapq.heappush(pq, (priority, counter, payload)); counter += 1  # avoids TypeError on ties

# #18 O(n) membership in a hot loop — TLE risk
seen_list = [1, 2, 3]             # x in seen_list is O(n)
seen_set = {1, 2, 3}              # x in seen_set is O(1) avg — use this in loops

# #19 Shallow copy of nested structures
old = [[1, 2], [3, 4]]
new = old[:]                      # shallow: new[0] IS old[0]
new[0][0] = 99                    # old[0][0] is now 99 too — bug
import copy
safe = copy.deepcopy(old)         # or [row[:] for row in old]

# #26 Never compare floats with ==
0.1 + 0.2 == 0.3                  # False — floating point representation error
abs((0.1 + 0.2) - 0.3) < 1e-9     # True — compare with a tolerance instead
```

## 37. 60-Second Warm-Up Drill + Language-Switch Box

Run this mentally (or literally paste and read) before a contest after weeks away from Python. Each line is one construct you'll need within the first 10 minutes of almost any CP problem.

```python
import sys, heapq
from collections import Counter, deque
from functools import cache
from bisect import bisect_left

data = sys.stdin.buffer.read().split()          # fast whole-input read
n = int(data[0])
a = list(map(int, data[1:n+1]))                  # list comprehension via map

freq = Counter(a)                                # O(n) frequency table
visited = set()                                  # O(1) membership set for graph/grid visits

h = []                                           # min-heap
for x in a:
    heapq.heappush(h, x)
smallest = heapq.heappop(h)
max_heap = []
for x in a:
    heapq.heappush(max_heap, -x)                 # max-heap via negation
biggest = -heapq.heappop(max_heap)

grid = [[0] * n for _ in range(n)]               # 2D init — NOT [[0]*n]*n

mask = 0
for i in range(n):                               # bitmask subset loop
    mask |= (1 << i)

pos = bisect_left(a, 5)                          # binary search: first idx with a[i] >= 5

# BFS on a grid using a direction-tuple loop
dq = deque([(0, 0)])
seen = {(0, 0)}
DIRS = ((0, 1), (0, -1), (1, 0), (-1, 0))
while dq:
    r, c = dq.popleft()
    for dr, dc in DIRS:
        nr, nc = r + dr, c + dc
        if 0 <= nr < n and 0 <= nc < n and (nr, nc) not in seen:
            seen.add((nr, nc))
            dq.append((nr, nc))

letters = ''.join(chr(ord('a') + i) for i in range(5))   # build string via ord/join

for k, v in freq.items():                        # dict.get / items iteration
    _ = freq.get(k, 0)

ranked = sorted(a, key=lambda x: (-freq[x], x))   # sorted with a composite key

@cache                                            # memoized DP
def fib(k: int) -> int:
    return k if k < 2 else fib(k - 1) + fib(k - 2)
```

What each line proves you still remember:

| Line(s) | Construct |
|---|---|
| fast stdin read | `sys.stdin.buffer.read().split()` batch parse |
| `list(map(int, ...))` | list-from-map idiom |
| `Counter(a)` | frequency table in one call |
| `visited = set()` | O(1) membership container |
| heap push/pop ×2 | min-heap direct, max-heap via negation |
| `[[0]*n for _ in range(n)]` | correct 2D init (no shared rows) |
| bitmask loop | subset/bitmask construction |
| `bisect_left` | binary search lower bound |
| `deque` + `DIRS` tuple loop | BFS scaffold, direction-delta idiom |
| `''.join(chr(ord(...)))` | char-code string building |
| `freq.get(k, 0)` | safe dict read |
| `sorted(..., key=lambda x: (...))` | multi-key sort |
| `@cache def fib` | one-line memoized recursion |

### Self-check — if you can't write this cold, re-read...

| Can you write, unaided... | Re-read |
|---|---|
| A 2D DP table init without the shared-row bug | §33 grid table, §36 mistake #2 |
| A min-heap and a max-heap from the same `heapq` module | §33 heaps, Part III §14 |
| `bisect_left` vs `bisect_right` and know which is which | §33 binary search, Part IV §18 |
| A `defaultdict(list)` adjacency list from an edge list | §33 dicts/graphs, Part III §12 |
| A closure that correctly captures a loop variable | §36 mistake #7, Part I §3 |
| `@cache`/`@lru_cache` on a recursive function, and when it breaks (mutable args) | Part I §4 |
| Fast I/O pattern (`sys.stdin` read-all + join-print) that won't TLE | §33 misc, Part VI §27 |
| The difference between `//`/`%` in Python vs C++/Java/JS | §36 mistakes #3, #4 |
| `functools.cmp_to_key` to sort with a legacy comparator | §33 sorting, Part IV §17 |
| Raising the recursion limit + running deep DFS safely | §36 mistake #12, Part VII §30 |

### C++ / Java / JS → Python mental-switch box

The ~10 things to actively remember when you've been away from Python:

| Coming from C++/Java/JS, remember... |
|---|
| Ints are **unbounded** — no overflow, no `long long`/`BigInteger`/BigInt needed. A genuine relief; stop pre-checking overflow. |
| `//` **floors** (rounds toward -∞), `%` **follows the divisor's sign** — both differ from C++/Java/JS truncation-toward-zero semantics on negative operands. |
| `[[0]*m]*n` is a **shared-row bug**, not a 2D array — always use a list comprehension for nested mutable structures. |
| Mutable default arguments (`def f(x=[])`) **persist across calls** — there's no per-call fresh copy like you'd assume. |
| `heapq` is **min-only** — negate values for a max-heap; there's no `std::priority_queue<T, vector<T>, greater<T>>` toggle. |
| No built-in balanced BST / `TreeMap` / `TreeSet` — use the third-party `sortedcontainers` (`SortedList`/`SortedDict`/`SortedSet`) for O(log n) order-statistics. |
| Default recursion limit is **1000 frames** — raise it (`sys.setrecursionlimit`) and use a thread with a bigger stack for deep DFS, or convert to iterative. |
| **CPython is slow** for raw loops (~10⁶–10⁷ ops/sec) — push work into built-ins/`itertools`, or switch to PyPy on judges that allow it. |
| `sorted`/`.sort` take `key=`, not a comparator — wrap any comparator function with `functools.cmp_to_key`. |
| `is` checks **identity**, not value equality — use `==` for values; `is` only for `None`/singleton checks (and never rely on small-int/string caching). |
| Empty set is `set()`, **not** `{}` — `{}` is an empty dict. |

---

*Python Complete Reference — companion to DSA Patterns 01–38 and the C++ (doc 39), Java (doc 40), and JS (doc 41) references.*
*When you forget something and it is not in here, add it. This document should only grow.*
