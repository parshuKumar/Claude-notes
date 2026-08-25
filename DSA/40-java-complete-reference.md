# JAVA — THE COMPLETE DSA REVISION REFERENCE
## Every Class, Every Idiom, Every Trap — In One Document
### Baseline: Java 17 | Modern additions marked **[Java 21]** / **[Java 16]** / **[Java 10]**
### Companion to Patterns 01–38 and the C++ Reference | Interview + Codeforces Ready

---

## HOW TO USE THIS DOCUMENT

Your primary language is C++. This is the doc you open when the problem is (or must be) in
Java and the syntax has slipped — so you never stop mid-problem to relearn `PriorityQueue`
or how `TreeMap.floorKey` works. **Every place Java differs from C++ is flagged with a
`> **vs C++:**` note**, because you think in C++ and Java's differences are exactly what trip you.

Three ways to read it:

| Situation | Where to go |
|---|---|
| **Switching to Java after months in C++** | Read the **C++→Java mental-switch box at the end of §37**, then skim Parts I–IV |
| **Mid-problem, forgot the exact API** — "how do I get floor key in a TreeMap again?" | **§33** (The "How Do I...?" Recipe Index) |
| **Something compiles but is wrong / TLEs** | **§36** (Top 30 Java-DSA Mistakes) and **§32** (Debugging & Exceptions) |

**The rule:** if you catch yourself thinking *"I know this exists in Java but not the exact
call"* — it should be in here. If it isn't, add it.

### The 8 things a C++ person MUST remember in Java (full list in §37)

1. `PriorityQueue` is a **MIN-heap** by default — the opposite of C++ `priority_queue`.
2. `==` compares **references**, not values — use `.equals()` for `Integer`/`String`.
3. No unsigned types — use `>>>` for logical right shift, `Long.compareUnsigned`, etc.
4. Use `1L << k` for any bitmask bit ≥ 31 — `1 << 31` is negative, and Java **masks** the shift count.
5. You can't sort `int[]` with a `Comparator` — box to `Integer[]` first.
6. `Arrays.sort(int[])` is **anti-quicksort-hackable** on Codeforces → shuffle or box.
7. No operator overloading — `.equals`, `.compareTo`, `BigInteger.add`, never `==`/`<`/`+`.
8. `Scanner` is too slow for CP → `BufferedReader`; on Codeforces the class must be named `Main`.

### Conventions used throughout

- Code is **Java 17** unless tagged. **[Java 21]** / **[Java 16]** / **[Java 10]** / **[Java 9]** tags note version + judge availability (Codeforces commonly runs Java 8 and Java 21; LeetCode runs Java 17).
- `// =>` comments show expected output.
- `> **Gotcha:**` blockquotes flag traps that cause real WA/TLE/RE.
- `> **vs C++:**` blockquotes flag where Java behaves differently from the C++ you already know.
- Complexity is given for every container operation and algorithm.

---

## TABLE OF CONTENTS

### PART I — CORE LANGUAGE
1. Primitives, Wrappers, Literals and Limits
2. References, Objects and null
3. Methods
4. Classes, Interfaces, Records, Enums
5. Lambdas and Functional Interfaces
6. Generics
7. Exceptions, Control Flow and Modern Syntax

### PART II — ARRAYS, STRINGS AND LISTS
8. Arrays (the primitive workhorse)
9. String and StringBuilder
10. ArrayList, LinkedList, and the List interface
11. Pair-like Types and Tuples

### PART III — MAPS, SETS AND ADAPTERS
12. HashMap and TreeMap
13. HashSet, TreeSet, LinkedHashSet
14. Deque, Stack, Queue, PriorityQueue
15. BitSet
16. Choosing the Right Structure

### PART IV — SORTING, SEARCHING, COMPARATORS AND STREAMS
17. Sorting
18. Binary Search
19. Collections and Arrays Utility Methods
20. Iterators and Iterable
21. Comparators and Streams (DSA subset)

### PART V — BIT MANIPULATION
22. Bitwise Operators and Fundamentals
23. The Complete Bit-Trick Catalog
24. Integer/Long Static Methods and Bit Containers

### PART VI — NUMBERS, BIGINTEGER AND I/O
25. Integer Arithmetic, Overflow and Safety
26. BigInteger and BigDecimal
27. Fast I/O (the make-or-break CP topic)
28. Randomness, Time and Useful Library Bits

### PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT
29. The Contest Template and Structure
30. Recursion, Stack, and JVM Gotchas for CP
31. Performance and Anti-Test Defenses
32. Debugging, Exceptions and Compiling

### PART VIII — QUICK REFERENCE AND RECIPES
33. The "How Do I...?" Recipe Index ← **the mid-problem lookup**
34. Master Complexity Table
35. n → Required Complexity Cheat Sheet
36. The Top 30 Java-DSA Mistakes That Cost You Problems
37. 60-Second Warm-Up Drill + C++→Java mental-switch box ← **start here after a break**

---

# PART I — CORE LANGUAGE

## 1. Primitives, Wrappers, Literals and Limits

### The 8 primitives

| Type | Size | Range | Default |
|---|---|---|---|
| `byte` | 8-bit | -128 .. 127 | 0 |
| `short` | 16-bit | -32,768 .. 32,767 | 0 |
| `int` | 32-bit | -2,147,483,648 .. 2,147,483,647 (~2.1e9) | 0 |
| `long` | 64-bit | ~-9.2e18 .. 9.2e18 | 0L |
| `float` | 32-bit IEEE 754 | ~7 sig digits | 0.0f |
| `double` | 64-bit IEEE 754 | ~15-17 sig digits | 0.0 |
| `char` | 16-bit **unsigned** UTF-16 code unit | 0 .. 65,535 | '\u0000' |
| `boolean` | JVM-defined (not a byte count) | true/false | false |

Sizes are fixed by the JVM spec — unlike C++, `int` is ALWAYS 32-bit and `long` is ALWAYS 64-bit on every platform. No `sizeof`, no platform-dependent width.

> **vs C++:** `char` in Java is 16-bit and unsigned (a UTF-16 code unit), not 8-bit like C++ `char`. For byte-level work use `byte` (signed, 8-bit).

### No unsigned types

Java has **no `unsigned` keyword at all** — no `unsigned int`, no `uint64_t`. Every integer type is signed. This is the sharpest early trap for a C++ programmer doing bit-manipulation or hashing problems.

Workarounds:

```java
// Treat an int's 32 bits as unsigned when printing/comparing
int x = -1; // bit pattern 0xFFFFFFFF
System.out.println(Integer.toUnsignedString(x));   // => 4294967295
long ux = Integer.toUnsignedLong(x);                // => 4294967295 (widen, safe)

// Unsigned comparison of two ints (both hold "unsigned" bit patterns)
int a = -1, b = 1;
System.out.println(Integer.compareUnsigned(a, b)); // => 1 (a is "bigger" unsigned)
System.out.println(Long.compareUnsigned(-1L, 1L)); // same idea for longs

// Unsigned right shift: >>> fills with 0s regardless of sign (C++ has no equivalent
// operator — in C++ >> on signed ints is implementation-defined/arithmetic)
int y = -8;
System.out.println(y >> 1);   // => -4  (arithmetic shift, sign-extends)
System.out.println(y >>> 1);  // => 2147483644 (logical shift, zero-fills)
```

> **Gotcha:** `>>>` does not exist for `byte`/`short` — they're promoted to `int` first, so shifting a negative `byte` with `>>>` still zero-fills 32 bits, not 8.

### int vs long — overflow

Exactly like C++: `int` arithmetic wraps silently (no exception), and intermediate expressions are computed in `int` even if you assign to a `long`.

```java
int a = 1_000_000_000;       // 1e9
int prod = a * 3;            // computed as int -> overflows silently
System.out.println(prod);    // => -1294967296  (WRONG, wrapped)

long fixed = (long) a * 3;   // cast ONE operand to long before multiplying
System.out.println(fixed);   // => 3000000000

// Classic DSA trap: n up to 1e5, sum of squares can hit 1e5 * (1e5)^2 = 1e15 -> use long
long sumSq = 0;
for (int i = 0; i < 100000; i++) sumSq += (long) i * i;
```

> **Gotcha:** `int + int` is always `int` math even when the target/return type is `long`. `long result = intA * intB;` still overflows before widening. Cast an operand: `(long) intA * intB`.

### Limits constants

```java
Integer.MAX_VALUE   // 2147483647
Integer.MIN_VALUE   // -2147483648
Long.MAX_VALUE       // 9223372036854775807
Long.MIN_VALUE       // -9223372036854775808
Double.MAX_VALUE     // ~1.8e308
```

> **Gotcha:** `-Integer.MIN_VALUE == Integer.MIN_VALUE` (overflow, same as C++ `INT_MIN`). `Math.abs(Integer.MIN_VALUE)` returns a negative number.

### Wrapper classes — autoboxing/unboxing

Every primitive has a wrapper: `Integer`, `Long`, `Double`, `Float`, `Short`, `Byte`, `Character`, `Boolean`. Wrappers are objects (needed for generics, since `List<int>` is illegal — see §6).

```java
Integer boxed = 5;        // autoboxing: int -> Integer (compiler inserts Integer.valueOf(5))
int unboxed = boxed;      // auto-unboxing: Integer -> int (compiler inserts boxed.intValue())

List<Integer> list = new ArrayList<>();
list.add(10);              // autobox
int first = list.get(0);   // auto-unbox
```

### THE INTEGER CACHE TRAP

`Integer.valueOf(int)` caches boxed values in **-128..127**. Autoboxing uses `valueOf`, so small `Integer`s that fall in this range are singletons and `==` "accidentally" works; outside the range, `==` compares references and fails.

```java
Integer a = 100, b = 100;
System.out.println(a == b);        // => true  (both from cache, same object)

Integer c = 1000, d = 1000;
System.out.println(c == d);        // => false (two distinct objects!)
System.out.println(c.equals(d));   // => true  (correct way)
System.out.println(c.intValue() == d.intValue()); // => true (unboxed, primitive compare)
```

> **Gotcha:** This bites hardest in a `HashMap<Integer,Integer>`-heavy solution where someone writes `if (map.get(k1) == map.get(k2))` expecting value comparison. It silently works for small test cases (values < 128) and fails on real input. ALWAYS use `.equals()` (or unbox to primitive first) when comparing wrapper types for value equality — never `==`.

### null-unboxing NPE

```java
Map<String, Integer> map = new HashMap<>();
// map.get("missing") returns null
int x = map.get("missing"); // auto-unbox attempts null.intValue() -> NullPointerException!

// Safe patterns:
int x1 = map.getOrDefault("missing", 0);          // preferred
Integer boxed = map.get("missing");
int x2 = (boxed != null) ? boxed : 0;
```

> **Gotcha:** This is one of the most common runtime crashes in Java DSA code — an unboxing NPE from a map lookup that "should" exist but doesn't (off-by-one in a frequency map, etc). If a bare `int x = map.get(...)` throws NPE, this is almost certainly why.

### Literals

```java
int million = 1_000_000_000;      // underscores as digit separators, purely readability
long big = 10_000_000_000L;       // L suffix REQUIRED — without it, 10000000000 is not a
                                   // valid int literal and won't compile (too big for int)
long small = 5L;                  // L suffix optional here but good habit for long literals
int hex = 0x1F;                   // 31
int bin = 0b1010;                 // 10  (binary literal, Java 7+)
int oct = 010;                    // 8  (leading 0 = octal — same trap as C++, avoid)
char c = 'A';
char newline = '\n';
double d = 3.14;
float f = 3.14f;                  // f suffix required for float literal
```

> **Gotcha:** `long x = 3000000000;` fails to compile — `3000000000` is parsed as an `int` literal first and it's out of `int` range. Must write `3000000000L`.

### `var` — local type inference [Java 10]

CF/LeetCode note: available on Java 17+ judges; not on Codeforces's older Java 8 option (use the Java 21 option on CF if you want `var`).

```java
var list = new ArrayList<Integer>();  // inferred as ArrayList<Integer>
var n = 5;                            // inferred as int
var sb = new StringBuilder();
for (var entry : map.entrySet()) { }  // fine in for-each too
```

`var` only works for **local variables with an initializer** — not for fields, parameters, or return types, and not without an initializer (`var x;` is illegal).

> **Gotcha:** `var x = 5; var y = 5L;` — x is `int`, y is `long`, inferred from the literal, not "whatever fits". Overuse `var` on ambiguous right-hand sides (`var x = getValue();`) hurts readability when scanning someone else's competitive code fast — prefer explicit types for anything non-obvious.

### `final`

```java
final int N = 100005;         // constant, cannot be reassigned (like C++ const, roughly)
final int[] arr = new int[5]; // reference is final, but arr[0] = 1 is still allowed!
arr[0] = 99;                  // OK — final only locks the reference, not the contents
```

> **vs C++:** `final` on a reference type is like `T* const p` (pointer itself const), NOT `const T*` (pointee const). The object can still be mutated.

### Casting, narrowing, widening

```java
int i = 300;
byte b = (byte) i;          // narrowing: truncates bits -> => 44 (300 % 256 = 44)
double dd = i;               // widening: implicit, no cast needed -> 300.0
int back = (int) 3.99;       // truncates toward zero -> 3 (not rounds!)
int neg = (int) -3.99;       // => -3
long l = 123456789012L;
int truncated = (int) l;     // truncates high bits — can produce garbage silently
```

> **Gotcha:** `(int) someDouble` truncates toward zero, not "rounds". Use `Math.round(x)` for rounding (returns `long` for `double` input, `int` for `float` input).

### Integer division and `%` sign

Same rules as C++11+: division truncates toward zero, and `%` result takes the sign of the **dividend**.

```java
System.out.println(7 / 2);     // => 3
System.out.println(-7 / 2);    // => -3  (truncated toward 0)
System.out.println(7 % -2);    // => 1   (sign follows dividend: 7)
System.out.println(-7 % 2);    // => -1  (sign follows dividend: -7)
System.out.println(Math.floorMod(-7, 2)); // => 1  (true mathematical mod, always >= 0)
```

> **Gotcha:** For a "true" non-negative modulo (common need when hashing / wrapping indices), use `Math.floorMod(a, b)`, not `((a % b) + b) % b` — although that manual trick also works and is common in ported C++ code.

---

## 2. References, Objects and null

### No pointers — reference semantics

Java has no pointers, no `&`, no `*`, no pointer arithmetic. Every variable of a non-primitive type is a **reference** to an object on the heap. All object arguments are passed **by value of the reference** — a copy of the reference is passed, not the object, and not a true alias to the caller's variable.

```java
static void mutate(int[] arr) {
    arr[0] = 999;          // mutates the SHARED array object -> visible to caller
}
static void reassign(int[] arr) {
    arr = new int[]{1,2,3}; // reassigns the LOCAL copy of the reference only
}                            // caller's variable is untouched

int[] a = {1, 2, 3};
mutate(a);
System.out.println(a[0]);   // => 999  (object was mutated through the shared reference)
reassign(a);
System.out.println(a[0]);   // => 999  (still 999 — reassignment inside the method didn't
                             //          propagate back, unlike C++ int*& or int&)
```

> **vs C++:** This is the #1 source of confusion coming from C++. Java's reference is like a C++ pointer that is *always passed by value* — never like `T&` (reference parameter) and never like `T*&` (reference to pointer). You can mutate what it points to, but you can NEVER make the caller's variable point somewhere else. There is no way to get "output parameter" semantics for a single primitive except: return it, use a 1-element array `int[1]`, wrap it in a mutable holder object, or use a field.

```java
// Wanting to "return" two values Java-style, since there's no std::pair<int,int>& out-param:
static void swapViaArray(int[] a, int i, int j) {
    int tmp = a[i]; a[i] = a[j]; a[j] = tmp;   // fine — mutating array contents
}
// To fake pass-by-reference for a single primitive:
static void increment(int[] box) { box[0]++; }
int[] counter = {0};
increment(counter);
System.out.println(counter[0]); // => 1
```

### null and NullPointerException

`null` is the absence of a reference — analogous to `nullptr`, but EVERY reference type variable defaults to `null` (uninitialized instance fields of object type are `null`, not garbage). Calling any method or accessing any field on a `null` reference throws `NullPointerException` (NPE) — Java's equivalent of a C++ null-pointer-dereference segfault, except it's a catchable exception, not a crash.

```java
String s = null;
System.out.println(s.length()); // throws NullPointerException

String s2 = null;
System.out.println(s2 == null);          // => true, safe (no dereference)
System.out.println("literal".equals(s2)); // => false, safe — put the non-null side first!
System.out.println(s2 != null && s2.length() > 0); // safe, short-circuit guards deref
```

> **Gotcha:** `s2.equals("literal")` throws NPE if `s2` is null; `"literal".equals(s2)` never throws. When comparing a variable that might be null against a known non-null constant/literal, put the constant on the left.

### `==` vs `.equals()` — the single most important distinction

`==` on reference types compares **object identity** (same memory address / same object), never contents. `.equals()` compares **logical/value equality**, as defined by the class (default `Object.equals` is identity-based unless overridden, e.g. by `String`, wrapper classes, records, or your own override).

```java
String a = new String("hi");
String b = new String("hi");
System.out.println(a == b);       // => false (two distinct objects)
System.out.println(a.equals(b));  // => true  (same content)

String c = "hi";   // string literal -> interned, from the "string pool"
String d = "hi";   // same literal -> JVM reuses the pooled instance
System.out.println(c == d);       // => true (both point at the same pooled literal!)
System.out.println(c == a);       // => false (a was built with `new`, not pooled)

Integer x = 200, y = 200;
System.out.println(x == y);       // => false (outside Integer cache -128..127, see §1)
System.out.println(x.equals(y));  // => true
```

> **Gotcha:** String literal `==` "accidentally" works in trivial test cases (`"abc" == "abc"` is often `true` due to interning) which teaches the WRONG habit. It silently breaks the moment a string is built via concatenation, `substring`, `new String(...)`, or read from input — always use `.equals()` for string content comparison, never `==`. Never rely on interning being active.

> **vs C++:** In C++, `==` on `std::string` compares contents (operator overloading). In Java, `==` on any object type — `String` included — is always identity comparison. There is no way to overload `==` in Java (see §4).

### `.hashCode()` / `.equals()` contract

If you override `.equals()`, you MUST also override `.hashCode()` consistently: equal objects must produce equal hash codes (the converse isn't required). This matters because `HashMap`/`HashSet` bucket entries by `hashCode()` first, then disambiguate with `.equals()` within a bucket.

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    @Override
    public int hashCode() { return Objects.hash(x, y); }
}

Set<Point> seen = new HashSet<>();
seen.add(new Point(1, 2));
System.out.println(seen.contains(new Point(1, 2))); // => true, ONLY because equals+hashCode
                                                       // are both overridden consistently
```

> **Gotcha:** If you override `equals()` but forget `hashCode()`, `HashSet`/`HashMap` will silently misbehave — `contains`/`get` return `false`/`null` for logically-equal objects because they hash to different buckets by identity hash code. Use a `record` (§4) to get both for free, or an array/list of primitives as key material via `List.of(...)`/`Arrays.asList(...)` which already implement both correctly. Raw arrays (`int[]`) as `HashMap` keys are a classic silent bug — arrays don't override `equals`/`hashCode` at all (identity-based), so two arrays with identical contents are different keys — wrap in `Arrays.asList(...)` or a `record`/`String` instead.

### Object identity, shallow vs deep copy

```java
int[] original = {1, 2, 3};
int[] alias = original;                 // same array object, NOT a copy
int[] shallowCopy = original.clone();   // new array, same primitive elements copied
int[] copyOf = Arrays.copyOf(original, original.length); // equivalent to clone() for 1D

alias[0] = 99;
System.out.println(original[0]);      // => 99 (alias shares the same backing array)
System.out.println(shallowCopy[0]);   // => 1  (independent copy, untouched)

// 2D arrays: clone() / copyOf() is shallow — inner arrays are STILL SHARED
int[][] grid = {{1,2},{3,4}};
int[][] shallow2D = grid.clone();
shallow2D[0][0] = 99;
System.out.println(grid[0][0]);       // => 99  (inner row array is shared, not copied!)

// True deep copy of a 2D array:
int[][] deep = new int[grid.length][];
for (int i = 0; i < grid.length; i++) deep[i] = grid[i].clone();
```

> **Gotcha:** `.clone()` and `Arrays.copyOf` on a 2D (or object-element) array only copy the outer array of references — the inner arrays/objects are still shared. This is a very common bug when backtracking algorithms snapshot a `grid`/`board` state.

### Immutability

`String` and all wrapper classes (`Integer`, `Long`, etc.) are immutable — every "mutating" operation returns a NEW object.

```java
String s = "hello";
s.concat(" world");             // return value DISCARDED — s is unchanged!
System.out.println(s);          // => hello
s = s.concat(" world");         // must reassign to capture the new object
System.out.println(s);          // => hello world
```

> **Gotcha:** Calling `s.toUpperCase()`, `s.trim()`, `s.replace(...)`, `s.substring(...)` etc. and ignoring the return value is a no-op — nothing on the original `s` changes. Always assign the result. This is a frequent silent bug when porting C++ in-place-mutation habits (`std::transform` on a `std::string` in place) to Java `String`.

### `instanceof` and pattern matching [Java 16]

```java
Object obj = "hello";
if (obj instanceof String) {
    String s = (String) obj;      // classic pre-16 style: check then cast
    System.out.println(s.length());
}
// Pattern matching for instanceof (Java 16+, LeetCode/CF Java 17+ or 21 both fine):
if (obj instanceof String s) {    // binds s directly, no separate cast line
    System.out.println(s.length()); // => 5
}
if (!(obj instanceof Integer i)) {
    // i is NOT in scope here (negated, so flow analysis excludes it)
} else {
    // i WOULD be in scope here if this branch existed
}
```

---

## 3. Methods

### Declaration and pass-by-value semantics

```java
static int add(int a, int b) {          // static: callable without an instance
    return a + b;
}
public int instanceAdd(int a, int b) {  // instance method, needs `new Foo().instanceAdd(...)`
    return a + b;
}
static void noReturn() { }              // void, like C++
```

All arguments — primitive or reference — are passed by value. For primitives that means a copy of the value (mutating the parameter inside the method never affects the caller). For references it means a copy of the reference (see §2 for the full explanation: you can mutate the referenced object's contents, but reassigning the parameter never propagates back).

```java
static void tryIncrement(int x) { x++; }
int n = 5;
tryIncrement(n);
System.out.println(n);   // => 5 (unchanged — primitive passed by value)
```

### Overloading and resolution order

Java picks an overload at compile time based on static argument types, in this priority order: (1) exact match without boxing/widening, (2) widening primitive conversion, (3) boxing/unboxing, (4) varargs — as a last resort.

```java
static void f(int x)        { System.out.println("int"); }
static void f(long x)       { System.out.println("long"); }
static void f(Integer x)    { System.out.println("Integer"); }
static void f(int... x)     { System.out.println("varargs"); }

f(5);        // => "int"      (exact match wins)
f(5L);       // => "long"     (exact match)
f((short)5); // => "long"     (widening short->int is possible, but short->long also works;
             //     actually short widens to int first in priority — but since f(int) exists,
             //     it resolves to "int". This line is illustrative; verify per-case.)
```

> **Gotcha:** If both a widening overload (`f(long)`) and a boxing overload (`f(Integer)`) are candidates, **widening always wins over boxing** — the compiler tries widening before it tries autoboxing. And varargs is tried dead last, after both. This surprises people who expect boxing (Integer) to be "closer" to `int` than widening to `long`.

### Varargs

```java
static int sum(int... nums) {          // becomes int[] nums inside the method
    int total = 0;
    for (int n : nums) total += n;
    return total;
}
sum();              // => 0  (empty array, legal)
sum(1, 2, 3);        // => 6
sum(new int[]{1,2}); // => 3, can also pass an array directly
```

> **Gotcha:** A varargs parameter must be LAST in the parameter list, and a class can have at most one varargs parameter. `System.out.printf` and `String.format` both rely on varargs (`Object...`), which is why mixing `int` and boxed types in format calls works.

### `static` methods

```java
class MathUtils {
    static int square(int x) { return x * x; }   // called as MathUtils.square(5), no instance
}
```

Almost all DSA helper functions in Java are `static` — there is no free-function equivalent of C++'s top-level `int square(int x)`; every method lives inside a class (see §4).

### Returning arrays / collections

```java
static int[] twoSum(int[] nums, int target) {
    Map<Integer,Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (seen.containsKey(need)) return new int[]{seen.get(need), i};
        seen.put(nums[i], i);
    }
    return new int[]{-1, -1};
}
static List<Integer> range(int n) {
    List<Integer> out = new ArrayList<>();
    for (int i = 0; i < n; i++) out.add(i);
    return out;                          // returning a reference to a heap object — safe,
}                                          // no "returning a local" danger like C++ stack refs
```

> **vs C++:** Unlike returning `&local_var` or a `vector` by reference to a stack-local in C++ (undefined behavior), returning a locally-`new`'d array/collection in Java is always safe — it lives on the heap and the GC keeps it alive as long as it's reachable.

### Recursion and THE STACK SIZE LIMIT

Default JVM thread stack size is small (commonly ~512KB–1MB depending on platform/`-Xss`), noticeably smaller than a typical C++ program's default (often 1MB–8MB, and CF's C++ judge frequently allows much deeper recursion). A naive recursive DFS on a graph/tree with ~10,000–100,000+ depth (e.g., a skewed/linked-list-shaped tree, or a long path graph) commonly throws `StackOverflowError` in Java when the equivalent C++ solution passes fine.

```java
// A deep, unbalanced recursion — this can StackOverflowError in Java well before it
// would crash equivalent C++ (e.g. n around 1e4-1e5 depending on frame size/JVM):
static int depth(TreeNode node) {
    if (node == null) return 0;
    return 1 + depth(node.right); // degenerates to a linked list for a skewed tree
}
```

**The standard CP fix**: run the solution's entry point inside a new `Thread` constructed with an explicit larger stack size, then join it.

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        // 1<<26 = 64 MB stack — generous headroom for deep recursion
        Thread t = new Thread(null, Main::solve, "main", 1 << 26);
        t.start();
        t.join();
    }

    static void solve() {
        // ... read input, run deep-recursive DFS here, print output ...
        // (do the REAL work in this method, not in the default main-thread stack)
    }
}
```

> **Gotcha:** `main(String[])` itself always runs on a fixed-size default thread — wrapping code in a try/catch around a StackOverflowError is NOT a real fix (state is usually corrupted afterward and it's fragile). The bigger-stack-thread pattern above is the idiomatic CP workaround; the more robust long-term fix is converting the recursion to an explicit-stack iterative version, but the thread trick is faster to reach for mid-contest.

### No default arguments

Java has no default parameter values (`void f(int x, int y = 0)` is illegal). Use overloading instead.

```java
static void greet(String name) { greet(name, "Hello"); }        // delegates
static void greet(String name, String greeting) {
    System.out.println(greeting + ", " + name);
}
greet("Bob");             // => Hello, Bob
greet("Bob", "Hi");        // => Hi, Bob
```

### No free functions

Every method must belong to a class (or interface). There is no Java equivalent of a C++ top-level `int helper(...)` outside any class — the closest thing is a `static` method in a utility class, called via `ClassName.method(...)` or, if in the same class as `main`, just `method(...)`.

---

## 4. Classes, Interfaces, Records, Enums

### Class basics

```java
class Point {
    int x, y;                       // fields, default 0 if primitive
    Point(int x, int y) {           // constructor — same name as class, no return type
        this.x = x;                 // `this` disambiguates field from parameter
        this.y = y;
    }
    int manhattan() { return Math.abs(x) + Math.abs(y); }
}
Point p = new Point(3, 4);          // `new` required — no stack-allocated objects in Java
System.out.println(p.manhattan()); // => 7
```

> **vs C++:** Java objects (non-primitives) are always heap-allocated via `new`; there's no stack-object/RAII equivalent, no destructors, and no `delete` — the garbage collector reclaims unreachable objects automatically.

### Access modifiers

| Modifier | Same class | Same package | Subclass (other package) | Everywhere |
|---|---|---|---|---|
| `private` | yes | no | no | no |
| (package-private, no modifier) | yes | yes | no | no |
| `protected` | yes | yes | yes | no |
| `public` | yes | yes | yes | yes |

For single-file CP solutions this rarely matters — everything is often just package-private or `public static` inside one class.

### `static` fields/methods and nested static classes

```java
class DSU {
    int[] parent, rank;
    DSU(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    int find(int x) {
        return parent[x] == x ? x : (parent[x] = find(parent[x])); // path compression
    }
    void union(int a, int b) {
        int ra = find(a), rb = find(b);
        if (ra == rb) return;
        if (rank[ra] < rank[rb]) { int t = ra; ra = rb; rb = t; }
        parent[rb] = ra;
        if (rank[ra] == rank[rb]) rank[ra]++;
    }
}

// The standard way to write a lightweight DSA helper type: a STATIC nested class
class Solution {
    static class Node {
        int val, weight;
        Node(int val, int weight) { this.val = val; this.weight = weight; }
    }
    static class Pair {
        int first, second;
        Pair(int first, int second) { this.first = first; this.second = second; }
    }
}
```

### Instance vs static nested classes — why DSA helpers should be `static`

A **non-static** (inner) nested class implicitly holds a hidden reference to its enclosing instance (`Outer.this`), so you can't instantiate it without an outer instance (`outer.new Inner()`), and it silently keeps the whole outer object alive/reachable. A **static** nested class has no such hidden link — it's just a normal class that happens to be scoped inside another for organization, and needs no outer instance to construct.

```java
class Outer {
    int x = 10;
    class Inner {                      // NON-static: holds implicit Outer.this
        int getX() { return x; }       // can access Outer's fields directly
    }
    static class StaticNested {        // static: no implicit outer reference
        int getX() { return 99; }      // CANNOT access Outer's instance fields directly
    }
}
Outer o = new Outer();
Outer.Inner inner = o.new Inner();               // needs an outer instance — awkward!
Outer.StaticNested nested = new Outer.StaticNested(); // no outer instance needed
```

> **Gotcha:** Forgetting `static` on a `Node`/`Pair`/`DSU` helper class defined inside your `Solution` class means every helper instance secretly carries a pointer to the enclosing `Solution` instance — harmless for correctness in a small CP program, but it's needless overhead and, in a long-lived program, a memory-leak risk (the helper keeps the whole outer object alive). Default to `static` for any DSA helper class unless it genuinely needs the enclosing instance's state.

### Interfaces, `default` methods, abstract classes

```java
interface Shape {
    double area();                          // abstract, no body — implementers must define it
    default String describe() {             // default method: has a body, inherited unless overridden
        return "Shape with area " + area();
    }
}
class Circle implements Shape {
    double r;
    Circle(double r) { this.r = r; }
    public double area() { return Math.PI * r * r; }
}

abstract class Animal {                     // cannot be instantiated directly
    abstract String sound();                // must be implemented by subclasses
    void speak() { System.out.println(sound()); } // concrete method, shared
}
class Dog extends Animal {
    String sound() { return "Woof"; }
}
```

### `Comparable<T>` vs `Comparator<T>`

`Comparable<T>` — the type defines its OWN natural ordering (analogous to overloading `operator<` in C++), via `compareTo`.

```java
class Person implements Comparable<Person> {
    String name; int age;
    Person(String name, int age) { this.name = name; this.age = age; }
    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age);  // ascending by age
        // contract: negative if this < other, 0 if equal, positive if this > other
    }
}
List<Person> people = new ArrayList<>(List.of(new Person("A", 30), new Person("B", 20)));
Collections.sort(people);                 // uses compareTo -> sorted by age ascending
```

`Comparator<T>` — an EXTERNAL ordering, doesn't require modifying the class, and you can define as many as you want (ascending, descending, by different fields).

```java
Comparator<Person> byName = (a, b) -> a.name.compareTo(b.name);
Comparator<Person> byAgeDesc = (a, b) -> Integer.compare(b.age, a.age);
Comparator<Person> byNameThenAge = Comparator
        .comparing((Person p) -> p.name)
        .thenComparingInt(p -> p.age);

people.sort(byAgeDesc);
people.sort(Comparator.comparingInt(p -> p.age).reversed());
```

> **vs C++:** Java has NO operator overloading whatsoever — you cannot define `<`, `==`, `+`, etc. on your own types. Everywhere C++ would use `operator<` for `sort`/`set`/`priority_queue`, Java requires `Comparable.compareTo` or an explicit `Comparator`. Everywhere C++ would use `operator==`, Java requires `.equals()`. Everywhere C++ would use `operator+` (e.g. on a custom Money/Vector type), Java requires a named method like `.add(...)`. This is a constant, real friction point when porting C++ idioms — expect to write a `Comparator` or `compareTo` for every custom sortable DSA type.

### Records — concise immutable data carriers [Java 16]

CF/LeetCode note: needs Java 16+; LeetCode (Java 17) supports it, CF's modern Java 21 judge supports it, CF's classic Java 8 option does not.

```java
record Pair(int a, int b) { }   // auto-generates: constructor, a(), b() accessors,
                                  // equals(), hashCode(), and toString() — all for free

Pair p = new Pair(3, 4);
System.out.println(p.a());          // => 3   (accessor, NOT p.a like a public field)
System.out.println(p);              // => Pair[a=3, b=4]   (toString for free)
System.out.println(p.equals(new Pair(3, 4))); // => true (value equality, for free)

// Perfect as a HashMap/HashSet key since equals+hashCode are correct out of the box:
Map<Pair, Integer> dist = new HashMap<>();
dist.put(new Pair(0, 0), 5);
System.out.println(dist.get(new Pair(0, 0))); // => 5  (works! unlike int[] as a key)

record Point(int x, int y) implements Comparable<Point> {
    public int compareTo(Point o) { return Integer.compare(x + y, o.x + o.y); }
    // records can have extra methods, static fields, and additional constructors too
    double dist() { return Math.sqrt((double)(x*x + y*y)); }
}
```

> **Gotcha:** Record fields are implicitly `final` — records are immutable by design, there's no setter. To "modify" a record you construct a new one: `p = new Pair(p.a() + 1, p.b());`. This is the single biggest upgrade over a hand-rolled static nested `Pair` class for CP — one line replaces a constructor + equals + hashCode + toString.

### Enums (brief)

```java
enum Direction { NORTH, SOUTH, EAST, WEST }
Direction d = Direction.NORTH;
switch (d) {
    case NORTH -> System.out.println("up");
    case SOUTH -> System.out.println("down");
    default -> System.out.println("sideways");
}
// Enums can carry fields/methods too:
enum Op {
    ADD { public int apply(int a, int b) { return a + b; } },
    SUB { public int apply(int a, int b) { return a - b; } };
    public abstract int apply(int a, int b);
}
```

### `toString()` for debug printing

```java
class Pt {
    int x, y;
    Pt(int x, int y) { this.x = x; this.y = y; }
    @Override
    public String toString() { return "(" + x + ", " + y + ")"; }
}
System.out.println(new Pt(1, 2));   // => (1, 2)   -- without an override this would print
                                      // something like Pt@1b6d3586 (class name + hash)
```

> **Gotcha:** Printing an object (or an array!) without a `toString()` override prints the useless default `ClassName@hexhash`. This trips people up hardest with arrays: `System.out.println(new int[]{1,2,3})` prints `[I@6d06d69c`, NOT the contents — use `Arrays.toString(arr)` / `Arrays.deepToString(arr2d)` instead (covered fully in the Collections part of this reference).

---

## 5. Lambdas and Functional Interfaces

### Lambda syntax

```java
Runnable r1 = () -> System.out.println("run");             // no args
var square = (java.util.function.IntUnaryOperator)(x -> x * x);
Comparator<Integer> cmp = (a, b) -> a - b;                  // two args, inferred types
Comparator<Integer> cmp2 = (Integer a, Integer b) -> {      // explicit types + block body
    int diff = a - b;
    return diff;
};
```

A lambda has no type of its own — it's only valid where the compiler can infer a target **functional interface** (an interface with exactly one abstract method).

### Functional interfaces (java.util.function) — the essentials

| Interface | Signature | Typical DSA use |
|---|---|---|
| `Comparator<T>` | `int compare(T a, T b)` | sorting, PriorityQueue ordering |
| `Runnable` | `void run()` | thread body (see §3 stack-size trick) |
| `Function<T,R>` | `R apply(T t)` | transform a value |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | combine two values |
| `Predicate<T>` | `boolean test(T t)` | filter condition |
| `Supplier<T>` | `T get()` | lazy value / factory (e.g. `map.computeIfAbsent(k, x -> new ArrayList<>())`) |
| `Consumer<T>` | `void accept(T t)` | side-effecting use (e.g. `forEach`) |
| `BinaryOperator<T>` | `T apply(T a, T b)` | reduce/merge (e.g. `Integer::sum`) |
| `IntUnaryOperator` | `int applyAsInt(int x)` | primitive, no boxing |
| `ToIntFunction<T>` | `int applyAsInt(T t)` | e.g. `Comparator.comparingInt(...)` |
| `IntBinaryOperator` | `int applyAsInt(int a, int b)` | e.g. combine two ints without boxing |

```java
Function<Integer, Integer> square = x -> x * x;
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
Predicate<Integer> isEven = x -> x % 2 == 0;
Supplier<List<Integer>> newList = ArrayList::new;
Consumer<Integer> print = System.out::println;
BinaryOperator<Integer> max = Integer::max;

System.out.println(square.apply(5));       // => 25
System.out.println(add.apply(2, 3));       // => 5
System.out.println(isEven.test(4));        // => true
List.of(1, 2, 3).forEach(print);           // => 1 2 3 (each on its own println)
```

> **Gotcha:** Prefer the primitive-specialized interfaces (`IntUnaryOperator`, `ToIntFunction`, `IntBinaryOperator`, `IntPredicate`, `IntSupplier`, `IntConsumer`, and their `Long`/`Double` counterparts) in hot loops — `Function<Integer,Integer>` boxes every `int` argument and return value, which is real overhead in tight DSA loops.

### Method references

```java
Comparator<Integer> cmp = Integer::compare;      // static method reference
Function<String, Integer> len = String::length;   // instance method on the argument
List<String> words = new ArrayList<>(List.of("bb", "a", "ccc"));
words.sort(Comparator.comparingInt(String::length)); // => [a, bb, ccc]

class Greeter {
    String greet(String name) { return "Hi " + name; }
    Function<String, String> asFn() { return this::greet; }  // bound instance method reference
}

Supplier<ArrayList<Integer>> factory = ArrayList::new;        // constructor reference
```

### Capturing — effectively-final only

A lambda can only capture local variables (and parameters) that are **effectively final** — never reassigned after initialization. Java has no equivalent of C++'s `[&]` capture-by-reference; captured locals are effectively copied by value at lambda-creation time (for the variable's value at that point), and you simply cannot reassign a captured local from inside the lambda.

```java
int total = 0;
// total++;                          // if this line were uncommented, total is no longer
                                       // effectively final and the lambda below won't compile
Runnable bad = () -> System.out.println(total); // fine as-is (0 never reassigned)

// Workaround #1: 1-element array as a mutable "box" (extremely common CP idiom)
int[] sum = {0};
List.of(1, 2, 3).forEach(x -> sum[0] += x);
System.out.println(sum[0]);          // => 6

// Workaround #2: an instance field (fields have no effectively-final restriction)
class Counter {
    int count = 0;
    void run() {
        Runnable inc = () -> count++;   // fine — `count` is a field, not a local
        inc.run(); inc.run();
        System.out.println(count);      // => 2
    }
}
```

> **vs C++:** C++'s `[&x]` lambda capture lets the lambda body reassign the outer `x` directly. Java flatly disallows mutating a captured local — the 1-element-array trick (or an `AtomicInteger`, or a field) is the standard workaround, and it comes up constantly in BFS/DFS lambdas that need to accumulate a result.

### Lambdas as comparators — the main DSA use

```java
int[][] intervals = {{1,3},{2,6},{8,10}};
Arrays.sort(intervals, (a, b) -> a[0] - b[0]);              // sort by start, ascending
Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0])); // safer: avoids overflow
                                                               // (a[0]-b[0] can overflow for
                                                               // extreme int values, unlike
                                                               // a safe Integer.compare)

PriorityQueue<int[]> minHeap = new PriorityQueue<>((a, b) -> a[1] - b[1]); // min by 2nd elem
```

> **Gotcha:** `(a, b) -> a - b` as a comparator is a classic overflow trap when `a`/`b` are near `Integer.MIN_VALUE`/`MAX_VALUE` (e.g. sorting values that can be negative and large in magnitude) — `a - b` can wrap around and silently break the sort. Prefer `Integer.compare(a, b)` (or `Long.compare` for longs) inside comparators.

### Why you can't easily write a recursive lambda

A lambda variable can't refer to itself in its own initializer, because at the point the lambda body is being defined, the variable isn't yet definitely assigned (`self` inside the lambda would refer to an as-yet-uninitialized local — doesn't compile as a simple assignment).

```java
// This does NOT work:
// Function<Integer,Integer> fact = n -> n <= 1 ? 1 : n * fact.apply(n - 1); // ERROR:
//     "fact" not yet initialized while defining the lambda that references it

// Workaround #1 (preferred): just write a normal recursive method — simplest and idiomatic
static int fact(int n) { return n <= 1 ? 1 : n * fact(n - 1); }

// Workaround #2: a 1-element array or field holding the functional interface, assigned
// AFTER declaration so the lambda body can close over the array/field (not the local itself)
Function<Integer,Integer>[] factHolder = new Function[1];
factHolder[0] = n -> ((int) n) <= 1 ? 1 : (int) n * (int) factHolder[0].apply((int) n - 1);
System.out.println(factHolder[0].apply(5)); // => 120
```

> **Gotcha:** In practice, nobody writes recursive lambdas in Java DSA code — just declare a normal (often `static`) recursive method. Reach for the array/field trick only if you're forced to keep something as a `Function`/`BiFunction` value (e.g. passing it around as a parameter) and truly need self-reference.

---

## 6. Generics

### Basic generic class and methods

```java
class Box<T> {
    private T value;
    Box(T value) { this.value = value; }
    T get() { return value; }
    void set(T value) { this.value = value; }
}
Box<Integer> b = new Box<>(5);      // diamond operator infers <Integer> on the right

static <T> void printAll(List<T> list) {         // generic method, own type parameter <T>
    for (T item : list) System.out.println(item);
}
static <T extends Comparable<T>> T max(List<T> list) {  // bounded type parameter
    T best = list.get(0);
    for (T item : list) if (item.compareTo(best) > 0) best = item;
    return best;
}
System.out.println(max(List.of(3, 1, 4, 1, 5)));  // => 5
```

### Wildcards — PECS (brief)

`? extends T` — read-only producer; `? super T` — write-only consumer. "**P**roducer **E**xtends, **C**onsumer **S**uper."

```java
static double sum(List<? extends Number> list) {   // can read Integers, Longs, Doubles, etc.
    double total = 0;
    for (Number n : list) total += n.doubleValue();
    return total;
}
static void addNumbers(List<? super Integer> list) { // can safely add Integers into it
    list.add(1);
    list.add(2);
}
```

DSA code rarely needs wildcards explicitly — they mostly matter for writing generic library-style utility methods; most CP code just uses concrete types like `List<Integer>`.

### Type erasure and its consequences

Generic type information exists only at compile time; at runtime all generic types are erased to their bound (usually `Object`). This has real, practical consequences for DSA code:

```java
class Box<T> {
    // T[] arr = new T[10];        // ILLEGAL — cannot create an array of a type parameter
    Object[] arr = new Object[10]; // must use Object[] and cast, or use a collection instead

    // if (value instanceof T) { } // ILLEGAL — cannot check instanceof against erased T

    // Class<T> cls = T.class;     // ILLEGAL — no T.class, the type is erased at runtime
}

// List<int> list;                 // ILLEGAL — generics work ONLY with reference types,
List<Integer> list;                // must use the boxed wrapper Integer

// You CANNOT mix a primitive array into generic collection machinery directly:
// List<int[]> is fine (int[] IS a reference/object type), but List<int> can never exist.
List<int[]> arrays = new ArrayList<>();  // OK — int[] is an object, not a primitive
arrays.add(new int[]{1, 2, 3});
```

> **Gotcha (major CP constraint):** Because there's no generic primitive collection, `ArrayList<Integer>`, `HashSet<Long>`, etc. box every single element — each `int` becomes a full `Integer` object on the heap. For huge `n` (say 1e6+ elements) this is a real, measurable performance and memory cost versus C++'s `vector<int>`, which stores primitives contiguously and unboxed. **The escape hatch**: for performance-critical large arrays, use a raw primitive array (`int[]`, `long[]`) directly instead of `ArrayList<Integer>`/`List<Long>` — manual resizing (`Arrays.copyOf` to double capacity) if the size isn't known upfront, exactly like implementing your own `vector` growth. This is a very common CP optimization when TLE (time-limit-exceeded) is caused by boxing overhead in a hot loop.

### `Comparable<T>` / `Comparator<T>` generics (cross-ref §4)

```java
class Task implements Comparable<Task> {
    int priority;
    public int compareTo(Task other) { return Integer.compare(priority, other.priority); }
}
Comparator<Task> byPriorityDesc = Comparator.<Task>comparingInt(t -> t.priority).reversed();
```

### Diamond operator

```java
List<Map<String, List<Integer>>> nested = new ArrayList<>();  // <> infers the full type
                                                                 // from the left-hand side —
                                                                 // no need to repeat it (Java 7+)
```

### Generic static helper methods

```java
static <T> void swap(T[] arr, int i, int j) {
    T tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
}
static <T> boolean allEqual(List<T> list) {
    if (list.isEmpty()) return true;
    T first = list.get(0);
    for (T item : list) if (!item.equals(first)) return false;
    return true;
}
```

---

## 7. Exceptions, Control Flow and Modern Syntax

### try/catch/finally

```java
try {
    int x = 5 / 0;                     // throws ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("caught: " + e.getMessage()); // => caught: / by zero
} finally {
    System.out.println("always runs"); // runs whether or not an exception was thrown
}

try {
    // multi-catch: handle several exception types with one block
    Object o = null;
    o.toString();
} catch (NullPointerException | ArithmeticException e) {
    System.out.println("npe or arith: " + e);
}
```

### Checked vs unchecked exceptions

Checked exceptions (subclasses of `Exception` but not `RuntimeException`, e.g. `IOException`) MUST be either caught or declared with `throws` on the enclosing method — the compiler enforces this. Unchecked exceptions (subclasses of `RuntimeException`, e.g. `NullPointerException`, `ArithmeticException`) require no such declaration.

```java
// BufferedReader.readLine() can throw IOException (checked) -> won't compile without
// handling it. The standard CP idiom is to just declare `throws IOException` on main
// and never actually catch it (let the JVM print the stack trace and exit on failure):
import java.io.*;
public class Main {
    public static void main(String[] args) throws IOException {   // <-- the CP idiom
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        String line = br.readLine();
        System.out.println(line);
    }
}
```

> **vs C++:** C++ has no checked/unchecked distinction — any exception can be thrown from anywhere without declaration. Java's checked-exception requirement is why virtually every CP Java template has `public static void main(String[] args) throws IOException` — it's the path of least resistance, not a sign of careful error handling.

### Common runtime exceptions in DSA

| Exception | Typical cause |
|---|---|
| `NullPointerException` | dereferencing/unboxing a `null` (see §1, §2) |
| `ArrayIndexOutOfBoundsException` | `arr[n]` for `arr.length == n`, off-by-one loops |
| `ArithmeticException` | `x / 0` for integers (NOT for `double` — see below) |
| `NumberFormatException` | `Integer.parseInt("abc")` or parsing malformed/empty input |
| `ClassCastException` | bad explicit cast, e.g. unchecked generic cast gone wrong |
| `ConcurrentModificationException` | mutating a collection while iterating it (below) |
| `StackOverflowError` | too-deep recursion (see §3) — an `Error`, not an `Exception` |

```java
System.out.println(1 / 0);      // throws ArithmeticException: / by zero
System.out.println(1.0 / 0);    // => Infinity (double division by zero does NOT throw!)
System.out.println(0.0 / 0);    // => NaN
Integer.parseInt("12a");        // throws NumberFormatException
```

> **Gotcha:** Integer division by zero throws; floating-point division by zero does NOT throw — it silently produces `Infinity`/`-Infinity`/`NaN`. This asymmetry matches C++ IEEE-754 float behavior but surprises people expecting a uniform "divide by zero always errors" rule.

### ConcurrentModificationException — the iteration trap

Structurally modifying (add/remove) a collection while iterating it with a for-each loop or an `Iterator` (other than via `Iterator.remove()` itself) throws `ConcurrentModificationException` — even in single-threaded code. This is a genuinely common bug.

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5));
// for (int n : nums) {
//     if (n % 2 == 0) nums.remove(Integer.valueOf(n));  // throws ConcurrentModificationException!
// }

// FIX 1: use an explicit Iterator and its own .remove()
Iterator<Integer> it = nums.iterator();
while (it.hasNext()) {
    int n = it.next();
    if (n % 2 == 0) it.remove();     // safe — the ONLY safe way to remove during iteration
}
System.out.println(nums);            // => [1, 3, 5]

// FIX 2: use removeIf (internally handles it safely)
List<Integer> nums2 = new ArrayList<>(List.of(1, 2, 3, 4, 5));
nums2.removeIf(n -> n % 2 == 0);
System.out.println(nums2);           // => [1, 3, 5]
```

> **vs C++:** Similar in spirit to C++ iterator invalidation (`vector::erase` inside a range-for), but Java actively detects and throws instead of causing silent undefined behavior — arguably safer to debug, but it still crashes at runtime instead of compiling cleanly like `it = v.erase(it)` idioms in C++.

### Enhanced for-each

```java
int[] arr = {1, 2, 3};
for (int x : arr) System.out.println(x);
List<String> words = List.of("a", "b");
for (String w : words) System.out.println(w);
Map<String,Integer> map = Map.of("a", 1, "b", 2);
for (Map.Entry<String,Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + "=" + e.getValue());
}
```

> **Gotcha:** For-each gives you a COPY of each primitive element (or a copy of each reference for objects) — mutating the loop variable itself never affects the underlying collection/array (`for (int x : arr) x++;` does nothing to `arr`). Use an index-based loop to mutate array contents in place.

### Labeled break/continue

Genuinely useful for breaking out of nested loops — Java has no `goto`, but labels give a clean equivalent for the common "break out of a nested grid/matrix double loop" pattern that C++ often handles with a `goto` or a flag variable.

```java
int[][] grid = {{1,2,3},{4,5,6},{7,8,9}};
int target = 5;
outer:
for (int i = 0; i < grid.length; i++) {
    for (int j = 0; j < grid[i].length; j++) {
        if (grid[i][j] == target) {
            System.out.println("found at " + i + "," + j); // => found at 1,1
            break outer;                 // breaks BOTH loops at once
        }
    }
}

search:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) continue search;    // continues the OUTER loop, skipping rest of inner
        System.out.println(i + "," + j);
    }
}
```

> **vs C++:** C++ has no labeled break/continue at all — the idiomatic equivalents are a `goto` (rare, frowned upon but legal) or a boolean sentinel flag checked after the inner loop. Java's labels are a genuinely cleaner, standard tool for this — use them freely for nested grid/BFS-grid searches.

### switch statement and switch expressions [Java 14]

```java
// Classic switch STATEMENT (fall-through by default — same trap as C++ switch)
int day = 3;
switch (day) {
    case 1:
    case 7:
        System.out.println("weekend");
        break;                      // required, or execution falls through to next case
    default:
        System.out.println("weekday");
}

// Modern switch EXPRESSION with arrow syntax (Java 14+) — no fall-through, returns a value
String kind = switch (day) {
    case 1, 7 -> "weekend";          // comma-separated labels, no fall-through
    case 2, 3, 4, 5, 6 -> "weekday";
    default -> "invalid";
};
System.out.println(kind);            // => weekday

// yield for a multi-statement arrow branch
int score = 85;
String grade = switch (score / 10) {
    case 10, 9 -> "A";
    case 8 -> {
        System.out.println("close to A!");
        yield "B";                   // yield produces the branch's value in a block body
    }
    default -> "C";
};
```

> **vs C++:** C++ `switch` always falls through without an explicit `break`, and can never directly produce a value as an expression. Java's older `switch` statement shares the fall-through trap (a classic bug source), but the newer arrow-form switch EXPRESSION eliminates fall-through entirely and can be used as a value — prefer arrow-form `switch` in new code.

### Ternary

```java
int a = 5, b = 3;
int max = (a > b) ? a : b;             // identical to C++
```

### Text blocks [Java 15]

CF/LeetCode note: Java 15+; useful for embedding multi-line expected output/test fixtures in scratch code, rarely needed for actual algorithm logic.

```java
String json = """
        {
          "name": "test",
          "value": 42
        }""";       // preserves internal newlines/indentation (relative to the closing """)
System.out.println(json);
```

### `assert` and why it's disabled by default

```java
int n = -1;
assert n >= 0 : "n must be non-negative, got " + n;   // compiles, but does NOTHING by default
```

`assert` statements are silently skipped at runtime unless the JVM is started with the `-ea` (enable assertions) flag. Neither Codeforces's nor LeetCode's judges enable `-ea`, so `assert` is effectively a no-op comment in submitted CP code — never rely on it for real validation or as a substitute for an actual `if (...) throw ...` check.

> **Gotcha:** Writing `assert` expecting it to catch a bug during a contest submission is a silent trap — it simply never fires unless `-ea` is explicitly passed, which judges don't do. Use an explicit `if (condition) throw new IllegalStateException(...)` (or just a debug `System.err.println`) if you actually want a runtime check to fire.

---

# PART II — ARRAYS, STRINGS AND LISTS

## 8. Arrays (the primitive workhorse)

### 8.1 Declaration, init, defaults

```java
int[] a = new int[5];              // all zeros: [0,0,0,0,0]
int[] b = {1, 2, 3};               // literal init (declaration only)
int[] c = new int[]{1, 2, 3};      // same, usable as expression/argument
int[] d;
d = new int[]{4, 5, 6};            // {1,2,3} form ONLY works at declaration
System.out.println(Arrays.toString(a)); // => [0, 0, 0, 0, 0]
```

Default values on `new type[n]`:

| Element type | Default |
|---|---|
| `int`, `long`, `short`, `byte` | `0` |
| `double`, `float` | `0.0` |
| `boolean` | `false` |
| `char` | `'\u0000'` (prints as blank/null char) |
| any object type (`String[]`, `Integer[]`, ...) | `null` |

> **Gotcha:** `Integer[] arr = new Integer[5];` gives an array of `null`, NOT `0`. Auto-unboxing a `null` element (`int x = arr[0];`) throws `NullPointerException`.

### 8.2 `.length` vs `.length()` vs `.size()` — THE recurring confusion

| Type | Syntax | Kind |
|---|---|---|
| Array (`int[]`, `String[]`, ...) | `a.length` | field, no parens |
| `String` | `s.length()` | method |
| `List`, `Map`, `Set`, `Collection` | `list.size()` | method |

```java
int[] arr = {1, 2, 3};
String s = "abc";
List<Integer> list = List.of(1, 2, 3);
System.out.println(arr.length);   // => 3   (field)
System.out.println(s.length());   // => 3   (method)
System.out.println(list.size());  // => 3   (method)
```

> **Gotcha:** Mixing these up is the single most common compile error when switching between array/String/List in Java. `arr.length()` and `s.length` both fail to compile.

### 8.3 Filling, copying

```java
int[] a = new int[6];
Arrays.fill(a, 7);                    // fill whole array
System.out.println(Arrays.toString(a)); // => [7, 7, 7, 7, 7, 7]

Arrays.fill(a, 1, 4, 9);              // fill index [1,4) only
System.out.println(Arrays.toString(a)); // => [7, 9, 9, 9, 7, 7]

int[] src = {1, 2, 3, 4, 5};
int[] cp1 = Arrays.copyOf(src, 3);       // first 3 elements
int[] cp2 = Arrays.copyOf(src, 7);       // pads with 0
int[] cp3 = Arrays.copyOfRange(src, 1, 4); // [from, to)
System.out.println(Arrays.toString(cp1)); // => [1, 2, 3]
System.out.println(Arrays.toString(cp2)); // => [1, 2, 3, 4, 5, 0, 0]
System.out.println(Arrays.toString(cp3)); // => [2, 3, 4]

// System.arraycopy — fastest, in-place, no allocation (vs C++ memmove)
int[] dst = new int[5];
System.arraycopy(src, 1, dst, 0, 3); // src[1..3] -> dst[0..2]
System.out.println(Arrays.toString(dst)); // => [2, 3, 4, 0, 0]
```

| Method | What it does | Complexity |
|---|---|---|
| `Arrays.fill(a, v)` | fill entire array with `v` | O(n) |
| `Arrays.fill(a, from, to, v)` | fill `[from, to)` with `v` | O(to-from) |
| `Arrays.copyOf(a, newLen)` | new array, truncate/pad with 0 | O(newLen) |
| `Arrays.copyOfRange(a, from, to)` | new array from `[from, to)` | O(to-from) |
| `System.arraycopy(src,sp,dst,dp,len)` | fast raw copy (like memmove, overlap-safe) | O(len) |
| `Arrays.sort(a)` | sort ascending in place | O(n log n) — see Part IV §anti-quicksort |
| `Arrays.equals(a, b)` | element-wise equality | O(n) |
| `Arrays.toString(a)` | `"[1, 2, 3]"` for 1D | O(n) |
| `Arrays.deepToString(a)` | for nested/2D arrays | O(total elements) |
| `Arrays.stream(a)` | `IntStream`/`LongStream`/`DoubleStream` (primitives) or `Stream<T>` (objects) | O(n) lazy |
| `Arrays.asList(a)` | fixed-size `List` view backed by `a` | O(1) |
| `Arrays.binarySearch(a, key)` | index or `-(insertion point)-1`, array must be sorted | O(log n) |

```java
int[] x = {3, 1, 2};
int[] y = {3, 1, 2};
System.out.println(x == y);            // => false (reference compare)
System.out.println(Arrays.equals(x,y)); // => true  (content compare)

int[][] g = {{1,2},{3,4}};
System.out.println(Arrays.deepToString(g)); // => [[1, 2], [3, 4]]
System.out.println(Arrays.toString(g));      // => [[I@<hash>, [I@<hash>] -- WRONG for 2D
```

> **vs C++:** `Arrays.toString`/`deepToString` are the closest thing to printing an array directly — C++ has no equivalent, you loop manually. `System.arraycopy` maps to `std::copy`/`memmove`.

### 8.4 `Arrays.asList` traps

```java
Integer[] boxed = {1, 2, 3};
List<Integer> view = Arrays.asList(boxed);
// view.add(4);       // throws UnsupportedOperationException (fixed-size!)
view.set(0, 99);       // OK — set/get allowed, writes through to `boxed`
System.out.println(boxed[0]); // => 99

int[] prim = {1, 2, 3};
List<int[]> weird = Arrays.asList(prim);   // List<int[]> of size 1, NOT List<Integer> of size 3!
System.out.println(weird.size()); // => 1
```

> **Gotcha:** `Arrays.asList(intArray)` does NOT autobox each `int` — varargs sees `int[]` as a single `T`, so you get `List<int[]>` containing one array. Must use `Integer[]` (boxed) or a stream to get a real `List<Integer>`.
> **Gotcha:** the list returned by `Arrays.asList` is fixed-size — `add`/`remove` throw `UnsupportedOperationException`; wrap in `new ArrayList<>(Arrays.asList(...))` for a mutable copy.

### 8.5 2D and jagged arrays

```java
int[][] g = new int[3][4];          // 3 rows x 4 cols, all 0, row-major
g[1][2] = 9;
for (int[] row : g) System.out.println(Arrays.toString(row));
// => [0, 0, 0, 0]
// => [0, 0, 9, 0]
// => [0, 0, 0, 0]

int[][] jagged = new int[3][];      // rows allocated separately -> jagged (ragged) array
jagged[0] = new int[]{1};
jagged[1] = new int[]{1, 2};
jagged[2] = new int[]{1, 2, 3};
System.out.println(Arrays.deepToString(jagged)); // => [[1], [1, 2], [1, 2, 3]]

int[][] lit = {{1,2,3},{4,5},{6}};  // literal jagged array
```

```java
// filling a 2D array needs a loop -- Arrays.fill has no 2D overload
int[][] grid = new int[3][3];
for (int[] row : grid) Arrays.fill(row, -1);
System.out.println(Arrays.deepToString(grid)); // => [[-1, -1, -1], [-1, -1, -1], [-1, -1, -1]]
```

```java
// COMMON BUG: this fills only ONE row array, aliased across all rows!
int[][] bad = new int[3][];
int[] shared = new int[3];
Arrays.fill(shared, 0);
for (int i = 0; i < 3; i++) bad[i] = shared;   // all rows point to SAME array
bad[0][0] = 99;
System.out.println(bad[1][0]); // => 99 (unintended aliasing)
```

> **Gotcha:** `new int[n][m]` allocates independent row arrays automatically — safe. But if you build rows yourself (e.g. `new int[n][]` then assigning the same array reference to multiple rows, or `Collections.nCopies`-style row sharing) you alias rows. Always allocate a fresh array per row.

### 8.6 Converting between `int[]`, `Integer[]`, `List<Integer>`

```java
int[] prim = {1, 2, 3};

// int[] -> List<Integer>  (must box, no direct method)
List<Integer> list1 = Arrays.stream(prim).boxed().collect(Collectors.toList());
List<Integer> list2 = new ArrayList<>();
for (int v : prim) list2.add(v);              // manual, also fine

// List<Integer> -> int[]
int[] back = list1.stream().mapToInt(Integer::intValue).toArray();
System.out.println(Arrays.toString(back)); // => [1, 2, 3]

// int[] -> Integer[]
Integer[] boxedArr = Arrays.stream(prim).boxed().toArray(Integer[]::new);

// Integer[] -> int[]
int[] unboxedArr = Arrays.stream(boxedArr).mapToInt(Integer::intValue).toArray();

// int[] -> IntStream -> sum/max/sorted etc (CP-handy)
int sum = Arrays.stream(prim).sum();
int max = Arrays.stream(prim).max().getAsInt();
int[] sorted = Arrays.stream(prim).sorted().toArray();
System.out.println(sum + " " + max); // => 6 3
```

| Conversion | Code |
|---|---|
| `int[]` → `List<Integer>` | `Arrays.stream(a).boxed().collect(Collectors.toList())` |
| `List<Integer>` → `int[]` | `list.stream().mapToInt(Integer::intValue).toArray()` |
| `int[]` → `Integer[]` | `Arrays.stream(a).boxed().toArray(Integer[]::new)` |
| `Integer[]` → `int[]` | `Arrays.stream(a).mapToInt(Integer::intValue).toArray()` |
| `List<Integer>` → `Integer[]` | `list.toArray(new Integer[0])` |

> **vs C++:** C++ `vector<int>` has none of this friction — the boxing bridge is pure Java tax caused by generics not supporting primitives directly.

---

## 9. String and StringBuilder

### 9.1 Immutability — the root of an O(n²) trap

Every `String` "modification" (`+`, `concat`, `replace`, `substring`) allocates a **new** `String` object; the original is untouched.

```java
String s = "";
for (int i = 0; i < 5; i++) {
    s += (char) ('a' + i);   // creates a NEW String object every iteration -- O(n) per append
}
System.out.println(s); // => abcde
// Total cost over n iterations: O(1+2+...+n) = O(n^2). BAD for large n.
```

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 5; i++) {
    sb.append((char) ('a' + i));  // amortized O(1) per append
}
System.out.println(sb.toString()); // => abcde
// Total cost: O(n). ALWAYS use StringBuilder for loop-built strings.
```

> **Gotcha:** `s += x` inside any loop over n iterations is an instant O(n²) red flag in competitive programming — swap to `StringBuilder`.
> **vs C++:** `std::string` is mutable and `+=` is amortized O(1) (like a `vector<char>`); Java's `String` is immutable, so `+=` in Java is fundamentally different (and much slower) than in C++.

### 9.2 Complete `String` method table

| Method | What it does | Complexity |
|---|---|---|
| `length()` | number of chars | O(1) |
| `charAt(i)` | char at index | O(1) |
| `substring(from)` | suffix starting at `from` | O(n) (new string) |
| `substring(from, to)` | `[from, to)` | O(to-from) |
| `indexOf(x)` / `indexOf(x, from)` | first index of char/substr, or `-1` | O(n) |
| `lastIndexOf(x)` | last index, or `-1` | O(n) |
| `contains(cs)` | substring present? | O(n) |
| `startsWith(p)` / `endsWith(p)` | prefix/suffix check | O(p) |
| `equals(o)` | content equality | O(n) |
| `equalsIgnoreCase(o)` | case-insensitive equality | O(n) |
| `compareTo(o)` | lexicographic diff (like `strcmp`) | O(n) |
| `toCharArray()` | copy into `char[]` | O(n) |
| `chars()` | `IntStream` of char codes | O(n) lazy |
| `split(regex)` | regex split → `String[]` | O(n) |
| `split(regex, limit)` | capped split | O(n) |
| `replace(a, b)` | literal replace (NOT regex) all occurrences | O(n) |
| `replaceAll(regex, b)` | regex replace all | O(n) |
| `trim()` | strip leading/trailing ASCII whitespace ≤ `' '` | O(n) |
| `strip()` **[Java 11]** | Unicode-aware trim | O(n) |
| `toLowerCase()` / `toUpperCase()` | case conversion (new string) | O(n) |
| `repeat(n)` **[Java 11]** | repeat string n times | O(n·len) |
| `isEmpty()` | `length()==0` | O(1) |
| `isBlank()` **[Java 11]** | empty or all-whitespace | O(n) |
| `String.format(fmt, args)` | printf-style formatting | O(n) |
| `String.join(sep, ...)` | join with separator | O(n) |
| `String.valueOf(x)` | any type → String | O(1)–O(n) |

> CF note: `strip`/`isBlank`/`repeat` (Java 11) are fine on CF (runs Java 8 and Java 21) only under the Java 21 judge; unavailable under the Java 8 judge. LeetCode's Java 17 has them.

```java
String s = "Hello, World!";
System.out.println(s.length());              // => 13
System.out.println(s.charAt(1));              // => e
System.out.println(s.substring(7));           // => World!
System.out.println(s.substring(7, 12));       // => World
System.out.println(s.indexOf('o'));            // => 4
System.out.println(s.indexOf('o', 5));         // => 8
System.out.println(s.indexOf('z'));            // => -1   ALWAYS check for -1!
System.out.println(s.lastIndexOf('o'));        // => 8
System.out.println(s.contains("World"));       // => true
System.out.println(s.startsWith("Hello"));     // => true
System.out.println(s.endsWith("!"));           // => true
System.out.println(s.equalsIgnoreCase("HELLO, WORLD!")); // => true
System.out.println("apple".compareTo("banana")); // => -1 (negative: "apple" < "banana")
System.out.println(s.toLowerCase());            // => hello, world!
System.out.println(s.replace(",", ";"));        // => Hello; World!
System.out.println("  hi  ".trim());            // => "hi"
System.out.println("ab".repeat(3));             // => ababab
System.out.println("".isEmpty());               // => true
System.out.println("   ".isBlank());            // => true
```

#### `split` — regex traps

```java
String csv = "a,b,,c";
System.out.println(Arrays.toString(csv.split(","))); // => [a, b, , c]

// "." is a REGEX metachar (any char) -- must escape to split on literal dot
String dots = "a.b.c";
// dots.split(".")   would split on EVERY char, giving [] (empty array-ish garbage)
System.out.println(Arrays.toString(dots.split("\\."))); // => [a, b, c]

// "|" is regex alternation -- must escape for literal pipe
String pipes = "a|b|c";
System.out.println(Arrays.toString(pipes.split("\\|"))); // => [a, b, c]

// split(" ") vs split("\\s+")
String spaced = "a   b  c";
System.out.println(Arrays.toString(spaced.split(" ")));    // => [a, , , b, , c]  (empty tokens for extra spaces)
System.out.println(Arrays.toString(spaced.split("\\s+"))); // => [a, b, c]        (collapses whitespace runs)

// trailing empty strings are DROPPED by default (limit = 0)
System.out.println(Arrays.toString("a,b,,".split(",")));      // => [a, b]        (trailing empties dropped)
System.out.println(Arrays.toString("a,b,,".split(",", -1)));  // => [a, b, , ]    (negative limit keeps all)
System.out.println(Arrays.toString("a,b,c".split(",", 2)));   // => [a, b,c]      (limit caps piece count)
```

> **Gotcha:** `split` takes a **regex**, not a literal string. `.` `|` `*` `+` `(` `)` `[` `]` `\` `^` `$` `?` all need escaping (`\\.` etc.) or use `Pattern.quote(s)`.
> **Gotcha:** leading whitespace on `" a b".split(" ")` produces a leading empty string `["", "a", "b"]`; trailing whitespace tokens are dropped unless `limit < 0`.

#### char arithmetic

```java
char c = 'd';
int idx = c - 'a';                 // => 3 (0-indexed letter position)
char back = (char) ('a' + idx);    // => 'd'
System.out.println(idx + " " + back); // => 3 d

System.out.println(Character.isDigit('7'));         // => true
System.out.println(Character.isLetter('x'));        // => true
System.out.println(Character.isLetterOrDigit('_'));  // => false
System.out.println(Character.toLowerCase('A'));     // => a
System.out.println(Character.getNumericValue('7')); // => 7 (int, works for digit chars)
System.out.println(Character.getNumericValue('a')); // => 10 (also handles a-z as 10-35, radix use)
```

#### String <-> number

```java
int i = Integer.parseInt("42");
long l = Long.parseLong("123456789012");
int hex = Integer.parseInt("1A", 16);        // => 26 (radix parse)
double d = Double.parseDouble("3.14");
String si = String.valueOf(42);              // int -> String
String sr = Integer.toString(255, 16);       // => "ff" (radix)
String bin = Integer.toBinaryString(10);     // => "1010"

try {
    Integer.parseInt("abc");
} catch (NumberFormatException e) {
    System.out.println("bad number"); // => bad number
}
```

> **Gotcha:** `Integer.parseInt` throws (unchecked) `NumberFormatException`, not returns a sentinel — unlike C++ `strtol` which sets `errno`/end-pointer. Wrap in try/catch if input may be malformed, or validate with `Character.isDigit` first.

#### `String` ↔ `char[]`, comparing strings

```java
char[] arr = "hello".toCharArray();
String back2 = new String(arr);               // char[] -> String
String back3 = String.valueOf(arr);           // same, alt spelling
System.out.println(back2 + " " + back3);      // => hello hello

String x = "abc", y = "abc";
String z = new String("abc");
System.out.println(x == y);        // => true  (string pool, interned literals)
System.out.println(x == z);        // => false (new object, NOT pooled)
System.out.println(x.equals(z));   // => true  (content compare -- ALWAYS use this)
```

> **Gotcha (the biggest Java-newbie bug):** NEVER compare `String` objects with `==` unless you specifically want reference identity. `==` sometimes "works" by luck (interned literals) then breaks in prod/CP tests. Always use `.equals()` (or `Objects.equals(a,b)` if either may be `null`).
> **vs C++:** C++ `std::string == std::string` compares content directly via `operator==`; Java's `==` on `String` is a raw reference compare, closer to C++ comparing two `char*` pointers.

#### sorting characters of a string

```java
String s = "dcba";
char[] chars = s.toCharArray();
Arrays.sort(chars);                 // sorts char[] in place, ascending
String sorted = new String(chars);
System.out.println(sorted); // => abcd
```

### 9.3 StringBuilder — complete method table

| Method | What it does | Complexity |
|---|---|---|
| `append(x)` | append any type (String, char, int, ...) — many overloads | amortized O(1) |
| `insert(i, x)` | insert at index | O(n) |
| `delete(from, to)` | remove `[from, to)` | O(n) |
| `deleteCharAt(i)` | remove one char | O(n) |
| `replace(from, to, str)` | replace range with `str` | O(n) |
| `reverse()` | reverse in place | O(n) |
| `charAt(i)` | read char | O(1) |
| `setCharAt(i, c)` | write char at index (mutates!) | O(1) |
| `length()` | current length | O(1) |
| `setLength(n)` | truncate/pad-with-`\0` | O(1)/O(n) |
| `toString()` | convert to `String` | O(n) |
| `capacity()` | current backing array size | O(1) |
| `indexOf(str)` | first index of substring | O(n) |

```java
StringBuilder sb = new StringBuilder("hello");
sb.append(" world");                     // => "hello world"
sb.append(42).append('!');               // append overloads for int, char, etc: "hello world42!"
sb.insert(5, ",");                       // "hello, world42!"
sb.deleteCharAt(sb.length() - 1);        // remove trailing '!': "hello, world42"
sb.delete(sb.length() - 2, sb.length()); // remove "42": "hello, world"
sb.setCharAt(0, 'H');                    // "Hello, world"
sb.replace(7, 12, "Java");               // "Hello, Java"
System.out.println(sb.toString());       // => Hello, Java
System.out.println(sb.length());         // => 11

StringBuilder rev = new StringBuilder("abcde");
rev.reverse();
System.out.println(rev); // => edcba   (println auto-calls toString())
```

#### palindrome check idiom

```java
static boolean isPalindrome(String s) {
    String rev = new StringBuilder(s).reverse().toString();
    return s.equals(rev);
}
System.out.println(isPalindrome("racecar")); // => true
```

> **Gotcha:** `StringBuilder` has NO `.equals()` content override — it inherits `Object.equals` (reference compare). Always `.toString()` before comparing content, or compare `sb1.toString().equals(sb2.toString())`.
> **vs C++:** `StringBuilder` ≈ mutable `std::string`; `.append()` ≈ `+=`/`push_back`; `.reverse()` ≈ `std::reverse(s.begin(), s.end())`.

### 9.4 Building output — the CP performance rule

```java
// BAD in a loop / when printing many lines: repeated System.out.println is slow (flushes)
// GOOD: accumulate into ONE StringBuilder, print once at the end
StringBuilder out = new StringBuilder();
for (int i = 0; i < 5; i++) {
    out.append(i).append(i < 4 ? " " : "\n");
}
System.out.print(out); // => 0 1 2 3 4  (single flush)
```

> **Gotcha:** In CP with heavy output (thousands of lines), calling `System.out.println` per line is slow because stdout is unbuffered by default in many judges. Build one `StringBuilder` (or wrap in `BufferedWriter`/`PrintWriter` — see Part I I/O section) and flush once.

### 9.5 `String.format` / printf specifiers

```java
System.out.println(String.format("%d + %d = %d", 2, 3, 5));   // => 2 + 3 = 5
System.out.println(String.format("%.2f", 3.14159));           // => 3.14
System.out.println(String.format("%5d", 42));                  // => "   42" (width 5, right-align)
System.out.println(String.format("%-5d|", 42));                 // => "42   |" (left-align)
System.out.println(String.format("%05d", 42));                  // => 00042 (zero-pad)
System.out.println(String.format("%s is %d", "age", 30));      // => age is 30
System.out.println(String.format("%x", 255));                   // => ff (hex)
System.out.printf("%d-%d%n", 1, 2);                              // => 1-2  (printf writes directly, no return string)
```

| Specifier | Meaning |
|---|---|
| `%d` | integer |
| `%f` | floating point (default 6 decimals) |
| `%.Nf` | float with N decimals |
| `%s` | string (any object via toString) |
| `%c` | char |
| `%x` / `%X` | hex (lower/upper) |
| `%5d` | min width 5, right-pad with spaces |
| `%-5d` | left-align width 5 |
| `%05d` | zero-pad width 5 |
| `%n` | platform newline (prefer over `\n` in format strings) |

---

## 10. ArrayList, LinkedList, and the List interface

`List<Integer>` is the interface (contract); `ArrayList<Integer>` and `LinkedList<Integer>` are implementations. Always **declare with the interface, construct with the impl**:

```java
List<Integer> list = new ArrayList<>();   // idiomatic
```

### 10.1 Declaration and init

```java
List<Integer> a = new ArrayList<>();                      // empty
List<Integer> b = new ArrayList<>(List.of(1, 2, 3));       // from another collection (copies)
List<Integer> c = new ArrayList<>(100);                    // initial CAPACITY hint, size still 0
List<Integer> d = List.of(1, 2, 3);                         // [Java 9] IMMUTABLE
List<Integer> e = new ArrayList<>(Arrays.asList(1, 2, 3));  // mutable copy from fixed-size view

System.out.println(a.size()); // => 0
System.out.println(c.size()); // => 0  (capacity != size, unlike some misconceptions)
// d.add(4); // throws UnsupportedOperationException -- List.of is IMMUTABLE
```

> CF note: `List.of` (Java 9) works fine on both CF judges (Java 8 -- NO, requires Java 9+ so fails on Java 8 judge; Java 21 judge -- yes) and on LeetCode's Java 17. Double-check the judge's Java version before relying on it.

> **Gotcha:** `Arrays.asList(...)` returns a FIXED-SIZE list (restated from §8.4) — `add`/`remove` throw. `List.of(...)` is fully IMMUTABLE — even `set` throws. Only `new ArrayList<>(...)` gives you a fully mutable list.

### 10.2 Complete method table

| Method | What it does | Complexity |
|---|---|---|
| `add(e)` | append to end | amortized O(1) |
| `add(index, e)` | insert at index, shifts right | O(n) |
| `get(index)` | read at index | O(1) ArrayList / O(n) LinkedList |
| `set(index, e)` | overwrite at index, returns old value | O(1) ArrayList / O(n) LinkedList |
| `size()` | element count | O(1) |
| `isEmpty()` | `size()==0` | O(1) |
| `remove(int index)` | remove BY INDEX, shifts left | O(n) |
| `remove(Object o)` | remove first element EQUAL to `o` | O(n) |
| `indexOf(o)` | first index of `o`, or `-1` | O(n) |
| `contains(o)` | membership test | O(n) |
| `clear()` | remove all elements | O(n) |
| `addAll(coll)` | append all elements of another collection | O(m) |
| `subList(from, to)` | VIEW (not copy!) of `[from, to)` | O(1) to create |
| `toArray()` / `toArray(T[]::new)` | copy to array | O(n) |
| `iterator()` | get an `Iterator<E>` | O(1) |
| `sort(comparator)` | in-place sort (null comparator = natural order) | O(n log n) |
| `forEach(consumer)` | apply lambda to each element | O(n) |

### 10.3 THE `remove(int)` vs `remove(Object)` overload trap

```java
List<Integer> nums = new ArrayList<>(List.of(10, 20, 30, 40));
nums.remove(2);                       // calls remove(int index) -- removes element AT index 2 (value 30)!
System.out.println(nums);             // => [10, 20, 40]   NOT what a beginner expects

List<Integer> nums2 = new ArrayList<>(List.of(10, 20, 30, 40));
nums2.remove(Integer.valueOf(2));     // calls remove(Object) -- removes the VALUE 2 (not present -> no-op)
System.out.println(nums2);            // => [10, 20, 30, 40]  unchanged, 2 was never in the list

List<Integer> nums3 = new ArrayList<>(List.of(10, 20, 30, 40));
nums3.remove(Integer.valueOf(30));    // removes the value 30 correctly
System.out.println(nums3);            // => [10, 20, 40]
```

> **Gotcha — famous bug:** for `List<Integer>`, `list.remove(2)` is ambiguous-looking but Java resolves it to the `remove(int index)` overload (exact primitive match wins over autoboxing) — it removes the element AT INDEX 2, not the value `2`. To remove the *value*, box it explicitly: `list.remove(Integer.valueOf(2))`. This does NOT happen for `List<String>` etc. since there's no int overload conflict — it's specific to `List<Integer>`/`List<Long>`/... where the value type itself is an integral wrapper.
> **vs C++:** `vector::erase` always takes an iterator/index, never a value — C++ has no equivalent ambiguity because there's no "remove by value" overload on the container itself (you'd use `std::remove` + `erase`, the erase-remove idiom).

### 10.4 Iteration patterns

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4, 5));

// indexed
for (int i = 0; i < list.size(); i++) System.out.print(list.get(i) + " ");
System.out.println(); // => 1 2 3 4 5

// for-each (read-only access, no index)
for (int v : list) System.out.print(v + " ");
System.out.println(); // => 1 2 3 4 5

// Iterator with safe removal
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    int v = it.next();
    if (v % 2 == 0) it.remove();   // SAFE removal during iteration
}
System.out.println(list); // => [1, 3, 5]
```

```java
List<Integer> bad = new ArrayList<>(List.of(1, 2, 3, 4));
try {
    for (int v : bad) {
        if (v == 2) bad.remove(Integer.valueOf(2)); // modifying list during for-each
    }
} catch (ConcurrentModificationException e) {
    System.out.println("CME!"); // => CME!
}
```

> **Gotcha:** Removing from a `List` while iterating with a for-each loop (or a raw `Iterator` without calling `it.remove()`) throws `ConcurrentModificationException`. The ONLY safe in-loop removal is `Iterator.remove()` (or build a new filtered list, or iterate backwards with indices).

### 10.5 `ArrayList` vs `LinkedList` — decision table

| Need | Use | Why |
|---|---|---|
| Random access by index (`get(i)`) | `ArrayList` | O(1) vs `LinkedList` O(n) |
| Append at end | `ArrayList` | amortized O(1), better cache locality |
| Frequent insert/remove in the MIDDLE by index | Neither really — both are O(n) (ArrayList shifts, LinkedList must first walk to index) | |
| Stack / Queue / Deque operations (push/pop both ends) | `ArrayDeque` | O(1) both ends, forward ref → Part III |
| General CP list use | `ArrayList` | almost always the right default |

> **Gotcha:** `LinkedList` is "almost never worth it" in Java CP — its only real edge is O(1) insert/delete once you already hold an `Iterator`/`ListIterator` positioned there (rare in typical problems), and it implements `Deque` — but `ArrayDeque` beats it for stack/queue use (lower memory overhead, no per-node object allocation). Default to `ArrayList`; reach for `ArrayDeque` (Part III) when you need a real deque/stack/queue.

### 10.6 `List<Integer>` <-> `int[]`

```java
List<Integer> list = List.of(5, 3, 8, 1);
int[] arr = list.stream().mapToInt(Integer::intValue).toArray();
System.out.println(Arrays.toString(arr)); // => [5, 3, 8, 1]

int[] src = {5, 3, 8, 1};
List<Integer> back = Arrays.stream(src).boxed().collect(Collectors.toList());
System.out.println(back); // => [5, 3, 8, 1]
```

### 10.7 `Collections` utility — brief (full table in Part IV)

```java
List<Integer> list = new ArrayList<>(List.of(3, 1, 4, 1, 5));

Collections.reverse(list);
System.out.println(list); // => [5, 1, 4, 1, 3]

Collections.swap(list, 0, 1);
System.out.println(list); // => [1, 5, 4, 1, 3]

Collections.fill(list, 0);
System.out.println(list); // => [0, 0, 0, 0, 0]

List<Integer> copies = Collections.nCopies(3, 9);
System.out.println(copies); // => [9, 9, 9]

int freq = Collections.frequency(List.of(1,2,2,3,2), 2);
System.out.println(freq); // => 3

List<Integer> ro = Collections.unmodifiableList(new ArrayList<>(List.of(1,2)));
// ro.add(3); // throws UnsupportedOperationException
```

> See Part IV for `Collections.sort`, `max`/`min`, `binarySearch`, `shuffle`, and the full utility table.

---

## 11. Pair-like Types and Tuples

Java has **no built-in `Pair`** for general-purpose use (unlike C++ `std::pair<A,B>`, which is in every corner of the STL — `map` entries, `priority_queue` of pairs, etc.). This is a real friction point when translating C++ CP habits to Java. Options, roughly best-to-niche:

### 11.1 (a) `record` — the modern best choice **[Java 16]**

```java
record Pair(int a, int b) {}

Pair p1 = new Pair(1, 2);
Pair p2 = new Pair(1, 2);
System.out.println(p1.a() + " " + p1.b()); // => 1 2   (accessors, NOT getA()/getB())
System.out.println(p1.equals(p2));          // => true  (auto value-equals)
System.out.println(p1);                     // => Pair[a=1, b=2] (auto toString)

Set<Pair> seen = new HashSet<>();
seen.add(p1);
System.out.println(seen.contains(p2)); // => true  (auto hashCode -- works correctly as a HashSet element)

Map<Pair, String> map = new HashMap<>();
map.put(new Pair(0, 0), "origin");
System.out.println(map.get(new Pair(0, 0))); // => origin  (works correctly as a HashMap KEY)
```

> CF note: records (Java 16) run under CF's Java 21 judge but NOT its Java 8 judge; LeetCode's Java 17 supports records fine.

Records auto-generate `equals`, `hashCode`, `toString`, and accessor methods (`a()`, `b()` — note: NOT `getA()`) based on the fields — exactly what you want for a value-type Pair used as a map key/set element.

### 11.2 (b) `int[]{a, b}` — cheap but NO value-equals

```java
int[] p1 = {1, 2};
int[] p2 = {1, 2};
System.out.println(p1.equals(p2));       // => false! arrays use IDENTITY equals
System.out.println(p1 == p2);            // => false (different objects)
System.out.println(Arrays.equals(p1, p2)); // => true (content compare, but this is a static method, not natural equals)

Set<int[]> set = new HashSet<>();
set.add(new int[]{1, 2});
System.out.println(set.contains(new int[]{1, 2})); // => false! -- BROKEN as a set element
```

> **Gotcha — critical:** `int[]` (and any array type) inherits `Object.equals`/`Object.hashCode` — reference identity, NOT content. Putting `int[]` keys into a `HashMap` or `HashSet` "compiles and runs" but silently never finds logically-equal entries. Arrays are fine for return values / sortable data (`List<int[]>` sorted by content is fine, since sorting uses a `Comparator` you supply, not `equals`) but NEVER use `int[]` as a `HashMap` key or `HashSet` element expecting value semantics.
> **vs C++:** C++ `pair<int,int>` has value semantics (`operator==`, hashable via `boost`/manual) out of the box; Java arrays deliberately don't, because arrays are just raw fixed blocks, not value types — this is the single biggest pair-related gotcha translating from C++.

### 11.3 (c) Encoding two ints into a `long` — the fast CP idiom

```java
// pack (a, b) where both fit comfortably in 32 bits, using bit-shift encoding
long packed = ((long) 5 << 32) | (7 & 0xffffffffL);
int a = (int) (packed >> 32);
int b = (int) packed;               // truncation drops upper bits, keeps lower 32
System.out.println(packed + " " + a + " " + b); // => 21474836487 5 7

// alternative: decimal-offset encoding (readable, safe if b is a known small non-negative bound)
long encoded = (long) 5 * 1_000_000 + 7;   // valid only if 0 <= b < 1_000_000
int a2 = (int) (encoded / 1_000_000);
int b2 = (int) (encoded % 1_000_000);
System.out.println(a2 + " " + b2); // => 5 7
```

```java
// used as a fast HashMap key or PriorityQueue element -- avoids boxing a Pair object per entry
Map<Long, Integer> freq = new HashMap<>();
long key = ((long) 3 << 32) | (4 & 0xffffffffL);
freq.merge(key, 1, Integer::sum);
System.out.println(freq.get(key)); // => 1
```

> Prefer this when performance matters (millions of pair-keyed map operations, or a `PriorityQueue<Long>` avoids per-element object allocation/boxing overhead vs `PriorityQueue<int[]>` or `PriorityQueue<Pair>`). The bit-shift form (`<<32 | (b & 0xffffffffL)`) handles negative `b` correctly (via the mask); the decimal form is more readable but needs a known safe bound on `b` and breaks silently if `b` is negative or exceeds the bound.

### 11.4 (d) `Map.Entry` / `AbstractMap.SimpleEntry`

```java
Map.Entry<Integer, Integer> e = new AbstractMap.SimpleEntry<>(1, 2);
System.out.println(e.getKey() + " " + e.getValue()); // => 1 2

// Map.entry(...) [Java 9] -- IMMUTABLE convenience factory
Map.Entry<Integer, Integer> e2 = Map.entry(1, 2);
System.out.println(e2); // => 1=2
```

> CF note: `Map.entry` (Java 9) needs CF's Java 21 judge or LeetCode Java 17; unavailable on CF's Java 8 judge. `AbstractMap.SimpleEntry` works everywhere (pre-Java 9).

Handy when a method already returns/expects `Map.Entry` (e.g. iterating a map), less idiomatic as a general-purpose ad-hoc pair versus a `record`.

### 11.5 (e) `int[]` for a triple

```java
int[] triple = {row, col, dist};   // common in BFS/Dijkstra grid problems
```

Same caveats as §11.2 apply (no value-equals) — fine for transient data (queue elements, sortable rows), not for map keys/set elements.

### 11.6 Sorting a `List<int[]>` by first then second

```java
List<int[]> pts = new ArrayList<>(List.of(
    new int[]{3, 1}, new int[]{1, 5}, new int[]{1, 2}
));
pts.sort((x, y) -> x[0] != y[0] ? x[0] - y[0] : x[1] - y[1]);
for (int[] p : pts) System.out.print(Arrays.toString(p) + " ");
System.out.println(); // => [1, 2] [1, 5] [3, 1]

// equivalent, more readable via Comparator chaining
pts.sort(Comparator.<int[]>comparingInt(p -> p[0]).thenComparingInt(p -> p[1]));
```

> **Gotcha:** `x[0] - y[0]` for the comparator can overflow if values approach `Integer.MIN_VALUE`/`MAX_VALUE`; prefer `Integer.compare(x[0], y[0])` for safety in CP with large/negative bounds.

### 11.7 Record as a `HashSet` element / `HashMap` key (recap)

```java
record Point(int x, int y) {}
Set<Point> visited = new HashSet<>();
visited.add(new Point(0, 0));
System.out.println(visited.contains(new Point(0, 0))); // => true (record auto value-equals+hashCode)
```

### 11.8 Decision table — which pair representation for which need

| Need | Use |
|---|---|
| HashMap key / HashSet element with correct value-equality | `record Pair(int a, int b) {}` |
| Sortable list of pairs, no map/set use | `int[]{a, b}` (with an explicit `Comparator`) |
| Performance-critical (millions of ops), single primitive key | `long` bit-packing (`(a<<32)|b`) |
| Readable code, small scale, mutable fields desired | small custom class, or `record` (records are immutable by design) |
| Interop with methods returning `Map.Entry` | `Map.Entry`/`AbstractMap.SimpleEntry` |
| Triple/quadruple of primitives, transient (e.g. BFS queue) | `int[]{a, b, c}` |
| Triple/quadruple as map key or set element | nested `record` or pack into a single `long`/`String` key |

> **vs C++ summary:** C++ `std::pair<A,B>` gives value-equality, ordering (`operator<`), hashability-with-effort, and structured bindings for free. Java has no single equivalent — `record` (Java 16+) is the closest (value-equals + hashCode + toString for free, but no default ordering — implement `Comparable` or pass a `Comparator`), everything else is a deliberate CP-specific trade-off (raw arrays for speed at the cost of losing value semantics, `long`-packing for maximum speed at the cost of readability and a bit of encoding care).

---

# PART III — MAPS, SETS AND ADAPTERS

## 12. HashMap and TreeMap

`HashMap<K,V>` = hash table, average O(1) ops, **no order** — like C++ `unordered_map`.
`TreeMap<K,V>` = red-black tree, O(log n) ops, **sorted by key** — like C++ `map`.

> **vs C++:** `HashMap` ~ `unordered_map`, `TreeMap` ~ `map`. Java has no direct equivalent of `unordered_map::reserve` tuning you'd bother with in CP — just use the no-arg constructor.

### 12.1 Common `Map` interface — method table

| Method | Does | Complexity (HashMap / TreeMap) |
|---|---|---|
| `put(k,v)` | insert or overwrite, returns old value or `null` | O(1) avg / O(log n) |
| `get(k)` | value for key or `null` if absent | O(1) avg / O(log n) |
| `getOrDefault(k,def)` | value or `def` if absent (no mutation) | O(1) avg / O(log n) |
| `containsKey(k)` | key present? | O(1) avg / O(log n) |
| `containsValue(v)` | value present? (scans) | O(n) / O(n) |
| `remove(k)` | delete, returns old value or `null` | O(1) avg / O(log n) |
| `size()` / `isEmpty()` | count / empty check | O(1) |
| `clear()` | remove all entries | O(n) |
| `keySet()` | live `Set<K>` view | O(1) to obtain |
| `values()` | live `Collection<V>` view | O(1) to obtain |
| `entrySet()` | live `Set<Map.Entry<K,V>>` view | O(1) to obtain |
| `putIfAbsent(k,v)` | insert only if key absent | O(1) avg / O(log n) |
| `merge(k,v,fn)` | combine existing value with `v` via `fn`, or insert `v` if absent | O(1) avg / O(log n) |
| `compute(k,fn)` | recompute value from `(k, oldVal)`, remove if `fn` returns `null` | O(1) avg / O(log n) |
| `computeIfAbsent(k,fn)` | insert `fn(k)` only if absent, returns value | O(1) avg / O(log n) |
| `computeIfPresent(k,fn)` | update only if present | O(1) avg / O(log n) |
| `replace(k,v)` | overwrite only if key present | O(1) avg / O(log n) |
| `forEach((k,v)->...)` | iterate via lambda | O(n) |

### 12.2 Frequency counting — the core idiom

```java
Map<Character,Integer> freq = new HashMap<>();
String s = "banana";
for (char c : s.toCharArray()) {
    freq.put(c, freq.getOrDefault(c, 0) + 1);
}
System.out.println(freq);
// => {a=3, b=1, n=2}
```

Cleaner with `merge` (avoids the get-then-put double lookup mentally, and reads better):

```java
Map<Character,Integer> freq2 = new HashMap<>();
for (char c : "banana".toCharArray()) {
    freq2.merge(c, 1, Integer::sum);   // (oldVal, 1) -> oldVal + 1, or just 1 if absent
}
System.out.println(freq2);
// => {a=3, b=1, n=2}
```

> **Gotcha:** `merge` removes the key automatically if the remapping function returns `null` — handy for "decrement and remove if zero" counters:
```java
Map<Character,Integer> cnt = new HashMap<>(Map.of('a', 1));
cnt.merge('a', -1, (old, dec) -> (old + dec == 0) ? null : old + dec);
System.out.println(cnt.containsKey('a'));
// => false
```

### 12.3 Adjacency lists — `computeIfAbsent`

The standard idiom for building `Map<Integer, List<Integer>>` graphs without null-checking:

```java
Map<Integer, List<Integer>> adj = new HashMap<>();
int[][] edges = {{1,2},{1,3},{2,3}};
for (int[] e : edges) {
    int u = e[0], v = e[1];
    adj.computeIfAbsent(u, x -> new ArrayList<>()).add(v);
    adj.computeIfAbsent(v, x -> new ArrayList<>()).add(u); // undirected
}
System.out.println(adj);
// => {1=[2, 3], 2=[1, 3], 3=[1, 2]}
```

`compute` — full read-modify-write in one call:

```java
Map<String,Integer> scores = new HashMap<>();
scores.put("alice", 10);
scores.compute("alice", (k, v) -> v == null ? 1 : v + 5);
scores.compute("bob",   (k, v) -> v == null ? 1 : v + 5);
System.out.println(scores);
// => {alice=15, bob=1}
```

### 12.4 Iteration

```java
Map<String,Integer> m = new HashMap<>();
m.put("x", 1);
m.put("y", 2);

for (Map.Entry<String,Integer> e : m.entrySet()) {
    System.out.println(e.getKey() + "=" + e.getValue());
}
// => x=1
// => y=2   (HashMap: order not guaranteed)

for (String k : m.keySet())   System.out.print(k + " ");   // => x y
System.out.println();
for (int v : m.values())      System.out.print(v + " ");   // => 1 2
```

> **Gotcha:** iterating `keySet()` and calling `get(k)` per key is a second lookup per entry — wasteful. Prefer `entrySet()` when you need both key and value.

### 12.5 THE unboxing-NPE trap

```java
Map<Integer,Integer> m = new HashMap<>();
m.put(1, 100);
// int x = m.get(2);   // throws NullPointerException! get(2) returns null, unboxing null -> NPE
Integer boxed = m.get(2);          // null, safe
int safe = m.getOrDefault(2, 0);   // 0, safe
System.out.println(boxed + " " + safe);
// => null 0
```

> **Gotcha:** `map.get(missingKey)` returns `null` for object-valued maps. Assigning that to a primitive `int`/`long`/`double`/etc. auto-unboxes and throws NPE at runtime. Always use `getOrDefault` or check `containsKey` first when the target is a primitive.

### 12.6 Autoboxing cost

`Map<Integer,Integer>` stores boxed `Integer` objects, not primitive `int`s — every `put`/`get` may allocate (values outside the cached `-128..127` range) and every key comparison is `.equals()`, not `==`. For hot loops over huge integer-keyed maps this is a real constant-factor cost versus C++'s unboxed `unordered_map<int,int>`. There's no primitive-specialized `HashMap` in the JDK (unlike, say, Eclipse Collections' `IntIntHashMap`) — for CP you generally just accept the cost, or fall back to arrays when keys are small dense integers.

### 12.7 TreeMap — navigable methods (the killer feature)

`TreeMap` implements `NavigableMap`, giving you C++ `map`'s `lower_bound`/`upper_bound` equivalents and more, directly.

| Method | Meaning |
|---|---|
| `firstKey()` / `lastKey()` | smallest / largest key (throws `NoSuchElementException` if empty) |
| `firstEntry()` / `lastEntry()` | smallest / largest entry, or `null` if empty |
| `floorKey(k)` | **largest key ≤ k**, or `null` |
| `ceilingKey(k)` | **smallest key ≥ k**, or `null` |
| `higherKey(k)` | **strictly >** k, or `null` |
| `lowerKey(k)` | **strictly <** k, or `null` |
| `floorEntry/ceilingEntry/higherEntry/lowerEntry(k)` | same, but return the `Map.Entry` |
| `pollFirstEntry()` / `pollLastEntry()` | remove + return smallest/largest entry |
| `headMap(k)` / `headMap(k, inclusive)` | view of keys `< k` (or `≤ k`) |
| `tailMap(k)` / `tailMap(k, inclusive)` | view of keys `≥ k` (or `> k`) |
| `subMap(from, to)` / `subMap(from, fromIncl, to, toIncl)` | view of keys in `[from, to)` |
| `descendingMap()` | reverse-order view |
| `descendingKeySet()` | reverse-order key view |
| `navigableKeySet()` | ascending `NavigableSet<K>` view |

```java
TreeMap<Integer,String> tm = new TreeMap<>();
tm.put(10, "a"); tm.put(20, "b"); tm.put(30, "c");

System.out.println(tm.floorKey(25));    // => 20  (largest <= 25)
System.out.println(tm.ceilingKey(25));  // => 30  (smallest >= 25)
System.out.println(tm.higherKey(20));   // => 30  (strictly > 20)
System.out.println(tm.lowerKey(20));    // => 10  (strictly < 20)
System.out.println(tm.floorKey(5));     // => null (nothing <= 5)
```

**Worked example: predecessor/successor query** (e.g. "nearest smaller/greater element already inserted" in a running stream):

```java
TreeMap<Integer,Integer> seen = new TreeMap<>(); // value -> index
int[] arr = {5, 2, 9, 4};
for (int i = 0; i < arr.length; i++) {
    Integer pred = seen.floorKey(arr[i]);   // nearest value <= arr[i] seen so far
    Integer succ = seen.ceilingKey(arr[i]); // nearest value >= arr[i] seen so far
    System.out.println(arr[i] + " -> pred=" + pred + " succ=" + succ);
    seen.put(arr[i], i);
}
// => 5 -> pred=null succ=null
// => 2 -> pred=null succ=5
// => 9 -> pred=5 succ=null
// => 4 -> pred=2 succ=5
```

### 12.8 Custom comparator TreeMap

```java
TreeMap<String,Integer> byLenDesc = new TreeMap<>(
    Comparator.comparingInt(String::length).reversed().thenComparing(Comparator.naturalOrder())
);
byLenDesc.put("a", 1);
byLenDesc.put("ccc", 3);
byLenDesc.put("bb", 2);
System.out.println(byLenDesc.keySet());
// => [ccc, bb, a]
```

> **Gotcha:** a `TreeMap`'s comparator defines *equality* for keys. Two keys that compare `0` are treated as the same key — `put` on the second overwrites the first, `containsKey` matches it. This bites hardest with `TreeSet` (see §13) but applies identically here: never use a comparator that returns `0` for genuinely-distinct keys unless you *want* that collapsing behavior.

### 12.9 `headMap` / `tailMap` / `subMap` are LIVE views

```java
TreeMap<Integer,String> tm2 = new TreeMap<>();
tm2.put(1,"a"); tm2.put(2,"b"); tm2.put(3,"c"); tm2.put(4,"d");
SortedMap<Integer,String> head = tm2.headMap(3);   // keys < 3
System.out.println(head);
// => {1=a, 2=b}
```

> **Gotcha:** these are *views* backed by the original map — mutating the view mutates the map (and structural changes to the underlying map while iterating a view throw `ConcurrentModificationException`, same rule as any collection).

### 12.10 LinkedHashMap — insertion order & LRU

`LinkedHashMap` = hash table + doubly linked list threading entries in **insertion order** (or **access order** if configured). No C++ STL equivalent — closest conceptual match is manually pairing `unordered_map` with a `list`.

```java
Map<String,Integer> lhm = new LinkedHashMap<>();
lhm.put("z", 1); lhm.put("a", 2); lhm.put("m", 3);
System.out.println(lhm.keySet());
// => [z, a, m]   (insertion order preserved, unlike HashMap)
```

**LRU cache trick** — access-order mode + `removeEldestEntry` override:

```java
class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private final int capacity;
    LRUCache(int capacity) {
        super(16, 0.75f, true); // true = access-order (get/put move entry to the end)
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity;
    }
}

LRUCache<Integer,String> cache = new LRUCache<>(2);
cache.put(1, "a");
cache.put(2, "b");
cache.get(1);          // touches 1, making 2 the least-recently-used
cache.put(3, "c");     // evicts 2 (LRU), not 1
System.out.println(cache.keySet());
// => [1, 3]
```

> **Gotcha:** the `true` third constructor arg is *access-order*; default (`false`) is *insertion-order*. This single flag plus overriding `removeEldestEntry` gives you a full O(1) LRU cache in ~10 lines — a very common "design an LRU cache" interview answer in Java.

### 12.11 HashMap with a custom-object key

Custom key types need correct, consistent `equals()` and `hashCode()` — without them, `HashMap` compares by reference identity, so two "equal" objects land in different buckets.

```java
record Point(int x, int y) {} // record auto-generates equals/hashCode/toString for free

Map<Point,String> m = new HashMap<>();
m.put(new Point(1,2), "origin-ish");
System.out.println(m.get(new Point(1,2)));
// => origin-ish   (works because record equals/hashCode compare field values)
```

```java
class BadKey {
    int x;
    BadKey(int x) { this.x = x; }
    // no equals()/hashCode() override -> identity semantics
}
Map<BadKey,String> bad = new HashMap<>();
bad.put(new BadKey(1), "v");
System.out.println(bad.get(new BadKey(1)));
// => null   (different object identity, default hashCode/equals don't match)
```

> **Gotcha:** if you override `equals()` you MUST override `hashCode()` consistently (equal objects → equal hash codes), or `HashMap`/`HashSet` silently break (lookups fail even when `equals` would say true). `record` types (Java 16+) give you both for free field-by-field — prefer them for composite keys in CP instead of hand-rolled classes or error-prone `Arrays.asList(x,y)` / string-concatenation hacks.

### 12.12 Complexity contrast & worst case

| Op | HashMap | TreeMap |
|---|---|---|
| get/put/remove | O(1) average | O(log n) guaranteed |
| get/put/remove worst case | **O(n)** (all keys collide into one bucket — treeified bucket in modern JDK caps it at O(log n) per bucket, but still no guarantee like TreeMap) | O(log n) guaranteed |
| iteration order | unspecified | ascending key order |
| min/max/predecessor/successor | not supported directly | O(log n) via navigable methods |

> **Gotcha:** classic HashMap worst-case O(n) happens with pathological hash collisions (rare in practice, and JDK 8+ treeifies buckets with ≥8 colliding entries of `Comparable` keys, capping worst case at O(log n) per bucket) — but don't rely on this; if you need guaranteed log-n behavior or sorted iteration, just use `TreeMap`.

---

## 13. HashSet, TreeSet, LinkedHashSet

`HashSet<E>` ~ C++ `unordered_set`. `TreeSet<E>` ~ C++ `set`. `LinkedHashSet<E>` = HashSet + insertion order, no direct C++ STL equivalent.

### 13.1 Common `Set` interface — method table

| Method | Does | Complexity (Hash / Tree) |
|---|---|---|
| `add(e)` | insert, returns `false` if already present | O(1) avg / O(log n) |
| `remove(e)` | delete, returns `false` if absent | O(1) avg / O(log n) |
| `contains(e)` | membership test | O(1) avg / O(log n) |
| `size()` / `isEmpty()` | count / empty check | O(1) |
| `clear()` | remove all | O(n) |
| `addAll(coll)` | union in place (set ← set ∪ coll) | O(m) avg / O(m log n) |
| `retainAll(coll)` | intersection in place | O(n) |
| `removeAll(coll)` | difference in place | O(m) avg / O(m log n) |
| `iterator()` | iterate (ascending for TreeSet) | O(n) full traversal |

```java
Set<Integer> a = new HashSet<>(List.of(1,2,3,4));
Set<Integer> b = new HashSet<>(List.of(3,4,5,6));

Set<Integer> union = new HashSet<>(a); union.addAll(b);
Set<Integer> inter = new HashSet<>(a); inter.retainAll(b);
Set<Integer> diff  = new HashSet<>(a); diff.removeAll(b);

System.out.println(union); // => [1, 2, 3, 4, 5, 6]  (order unspecified)
System.out.println(inter); // => [3, 4]
System.out.println(diff);  // => [1, 2]
```

> **Gotcha:** `addAll`/`retainAll`/`removeAll` **mutate the receiver** — always copy first (`new HashSet<>(a)`) if you need to keep `a` unchanged, exactly like you'd be careful with in-place C++ set algebra.

### 13.2 TreeSet — navigable methods

Same semantics as TreeMap's navigable key methods (§12.7) — this is the direct analog of C++ `set::lower_bound`/`upper_bound`.

| Method | Meaning |
|---|---|
| `first()` / `last()` | smallest / largest (throws if empty) |
| `floor(x)` | largest element **≤ x**, or `null` |
| `ceiling(x)` | smallest element **≥ x**, or `null` |
| `higher(x)` | smallest element **> x**, or `null` |
| `lower(x)` | largest element **< x**, or `null` |
| `pollFirst()` / `pollLast()` | remove + return smallest/largest, or `null` |
| `headSet(x)` / `headSet(x, incl)` | view of elements `< x` (or `≤ x`) |
| `tailSet(x)` / `tailSet(x, incl)` | view of elements `≥ x` (or `> x`) |
| `subSet(from, to)` | view of elements in `[from, to)` |
| `descendingSet()` | reverse-order view |
| `descendingIterator()` | iterator in reverse order |

**"Largest element ≤ x" / "smallest element > x" idioms** (always null-guard — `NavigableSet` returns `null`, it does NOT throw, unlike C++ where `lower_bound`/`upper_bound` return `end()`):

```java
TreeSet<Integer> ts = new TreeSet<>(List.of(10, 20, 30, 40));

Integer le = ts.floor(25);    // largest <= 25
Integer gt = ts.higher(30);   // strictly > 30
System.out.println(le + " " + gt);
// => 20 40

Integer none = ts.higher(40); // nothing bigger than 40
if (none == null) System.out.println("no successor");
// => no successor
```

> **vs C++:** C++ `set::lower_bound(x)` returns an iterator to the first element `>= x` (compare to `ceiling`), and `upper_bound(x)` returns first element `> x` (compare to `higher`) — but C++ signals "not found" with `end()` iterator, Java signals it with `null`. Forgetting the null check is the #1 porting bug here.

### 13.3 TreeSet as sorted running structure / sliding window

```java
// maintain a sorted window of the last k elements, query min/max/kth in O(log k)
TreeSet<Integer> window = new TreeSet<>();
int[] stream = {5, 1, 9, 3, 7};
int k = 3;
for (int i = 0; i < stream.length; i++) {
    window.add(stream[i]);
    if (window.size() > k) window.remove(stream[i - k]);
    if (window.size() == k) {
        System.out.println("window=" + window + " min=" + window.first() + " max=" + window.last());
    }
}
// => window=[1, 5, 9] min=1 max=9
// => window=[1, 3, 9] min=1 max=9
// => window=[3, 7, 9] min=3 max=9
```

### 13.4 NO multiset in Java — a real gap vs C++

C++ has `multiset<T>` (sorted, duplicates allowed, O(log n) insert/erase-one/count). **The JDK has nothing equivalent** — `Set` by contract forbids duplicates. Two standard workarounds:

**Workaround 1 — `TreeMap<T,Integer>` as a count-map** (THE Java multiset; sorted, supports insert/erase-one/erase-all/min/max):

```java
TreeMap<Integer,Integer> multiset = new TreeMap<>();

// insert
for (int v : new int[]{5, 3, 5, 1, 5, 3}) {
    multiset.merge(v, 1, Integer::sum);
}
System.out.println(multiset);
// => {1=1, 3=2, 5=3}

// erase one occurrence of 5
multiset.computeIfPresent(5, (k, c) -> c == 1 ? null : c - 1); // null removes the key
System.out.println(multiset);
// => {1=1, 3=2, 5=2}

// erase all occurrences of 3
multiset.remove(3);
System.out.println(multiset);
// => {1=1, 5=2}

// min / max (like C++ multiset::begin() / rbegin())
System.out.println(multiset.firstKey() + " " + multiset.lastKey());
// => 1 5
```

**Workaround 2 — `TreeMap` for a running median** (two-heap or balanced-tree median-maintenance; here shown via `TreeMap` count-map + tracked size, useful when you also need arbitrary rank queries):

```java
TreeMap<Integer,Integer> counts = new TreeMap<>();
List<Integer> data = List.of(5, 15, 1, 3);
List<Double> medians = new ArrayList<>();
for (int x : data) {
    counts.merge(x, 1, Integer::sum);
    // naive O(n) median scan for illustration; use a two-heap approach for O(log n) per insert
    List<Integer> sorted = new ArrayList<>();
    for (var e : counts.entrySet()) for (int i = 0; i < e.getValue(); i++) sorted.add(e.getKey());
    int n = sorted.size();
    double med = (n % 2 == 1) ? sorted.get(n/2) : (sorted.get(n/2 - 1) + sorted.get(n/2)) / 2.0;
    medians.add(med);
}
System.out.println(medians);
// => [5.0, 10.0, 5.0, 4.0]
```

> **Gotcha:** the real "running median" pattern in CP is normally a **two-PriorityQueue** solution (max-heap for the lower half, min-heap for the upper half) — see §14 — not a `TreeMap` scan. The `TreeMap` count-map is shown here specifically as the *multiset* substitute; use it when you need multiset semantics (count/erase-one/duplicates + sorted order), not as your default median algorithm.

### 13.5 LinkedHashSet — insertion order + dedup

```java
Set<Integer> lhs = new LinkedHashSet<>(List.of(5, 1, 5, 3, 1, 9));
System.out.println(lhs);
// => [5, 1, 3, 9]   (dedup'd, original insertion order preserved)
```

Useful whenever you need "unique elements, in the order first seen" — HashSet loses order, TreeSet reorders by value.

### 13.6 Custom comparator TreeSet — the dedup trap

```java
TreeSet<String> byLen = new TreeSet<>(Comparator.comparingInt(String::length));
byLen.add("cat");
byLen.add("dog"); // same length as "cat" -> comparator returns 0 -> treated as DUPLICATE, NOT added
byLen.add("apple");
System.out.println(byLen);
// => [cat, apple]
```

> **Gotcha:** a `TreeSet`'s comparator (or natural ordering via `compareTo`) is the SOLE definition of equality inside the set — `equals()`/`hashCode()` are irrelevant. If your comparator can return `0` for two objects you consider distinct, the second insert silently vanishes. Always break ties (e.g. `.thenComparing(...)`) if you don't want this collapsing behavior — this is the single most common TreeSet bug when porting from C++ `set` with a custom comparator (C++ has the identical trap, `set` also uses `Compare` for equivalence, so if you know to avoid it in C++ apply the same discipline here).

### 13.7 Set algebra recap

```java
Set<Integer> a = new HashSet<>(List.of(1,2,3));
Set<Integer> b = new HashSet<>(List.of(2,3,4));
boolean disjoint = Collections.disjoint(a, b);
System.out.println(disjoint);
// => false
```

---

## 14. Deque, Stack, Queue, PriorityQueue

### 14.1 `ArrayDeque` — THE stack AND THE queue

`ArrayDeque<E>` is a resizable circular-array double-ended queue. It is **faster than both** the legacy `Stack` class and `LinkedList` for stack/queue use (better cache locality, no per-node allocation, no synchronization overhead) — **use `ArrayDeque` for both stacks and queues by default.**

> **vs C++:** `ArrayDeque` ~ `std::deque` used as a stack/queue (or `std::stack`/`std::queue` adapters). No direct equivalent of choosing an underlying container like C++ adapters allow — `ArrayDeque` IS the container.

**As a stack** (`push`/`pop`/`peek` — pushes/pops at the head):

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
stack.push(2);
stack.push(3);
System.out.println(stack.pop());  // => 3
System.out.println(stack.peek()); // => 2
```

**As a queue** (`offer`/`poll`/`peek` — offer adds at tail, poll removes from head):

```java
Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1);
queue.offer(2);
queue.offer(3);
System.out.println(queue.poll());  // => 1
System.out.println(queue.peek());  // => 2
```

**As a true deque** (`addFirst/addLast/pollFirst/pollLast/peekFirst/peekLast`) — the monotonic-deque tool for sliding-window-maximum and similar:

```java
// sliding window maximum, window size k, storing INDICES, deque kept monotonically decreasing
int[] nums = {1, 3, -1, -3, 5, 3, 6, 7};
int k = 3;
Deque<Integer> mono = new ArrayDeque<>(); // holds indices
List<Integer> result = new ArrayList<>();
for (int i = 0; i < nums.length; i++) {
    while (!mono.isEmpty() && mono.peekFirst() <= i - k) mono.pollFirst();
    while (!mono.isEmpty() && nums[mono.peekLast()] < nums[i]) mono.pollLast();
    mono.addLast(i);
    if (i >= k - 1) result.add(nums[mono.peekFirst()]);
}
System.out.println(result);
// => [3, 3, 5, 5, 6, 7]
```

### 14.2 Why not the legacy `Stack` class

`java.util.Stack` extends `Vector`, meaning every method is `synchronized` (pointless overhead outside multithreading) and it inherits `Vector`'s odd index-based API. It works correctly but is strictly slower than `ArrayDeque` for single-threaded CP use — mentioned here so you recognize it in old code, but prefer `ArrayDeque` going forward.

### 14.3 `Queue`/`Deque` method pairs — throws vs returns-sentinel

| Operation | Throws on failure | Returns null/false on failure |
|---|---|---|
| Insert (queue, tail) | `add(e)` | `offer(e)` |
| Remove (queue, head) | `remove()` | `poll()` |
| Examine (queue, head) | `element()` | `peek()` |
| Insert (deque, head) | `addFirst(e)` | `offerFirst(e)` |
| Insert (deque, tail) | `addLast(e)` | `offerLast(e)` |
| Remove (deque, head) | `removeFirst()` | `pollFirst()` |
| Remove (deque, tail) | `removeLast()` | `pollLast()` |
| Examine (deque, head) | `getFirst()` | `peekFirst()` |
| Examine (deque, tail) | `getLast()` | `peekLast()` |

```java
Deque<Integer> dq = new ArrayDeque<>();
System.out.println(dq.poll());   // => null  (empty, poll returns null, doesn't throw)
try {
    dq.remove();                 // throws
} catch (NoSuchElementException e) {
    System.out.println("threw NoSuchElementException");
}
// => null
// => threw NoSuchElementException
```

> **Gotcha:** in CP, always prefer the `offer`/`poll`/`peek` family — they return a sentinel (`null`/`false`) instead of throwing, so you can check `if (queue.isEmpty())` OR just check the return value, without exception-handling boilerplate.

### 14.4 Null-not-allowed gotcha

```java
Deque<Integer> dq2 = new ArrayDeque<>();
try {
    dq2.add(null);
} catch (NullPointerException e) {
    System.out.println("ArrayDeque rejects null elements");
}
// => ArrayDeque rejects null elements
```

> **Gotcha:** `ArrayDeque` explicitly **forbids `null` elements** (null is used internally as the "empty" sentinel for `peek`/`poll`). `LinkedList` (which also implements `Deque`) *does* allow null — another reason `ArrayDeque` is the safer default, it fails fast instead of silently corrupting logic that checks `== null` for emptiness.

### 14.5 PriorityQueue in depth — DEFAULT IS A MIN-HEAP

> **vs C++: THIS IS THE #1 PORTING BUG.** C++ `std::priority_queue` defaults to a **max-heap** (largest on top). Java's `PriorityQueue` defaults to a **min-heap** (smallest on top). Porting C++ competitive code that relies on default max-heap behavior without flipping the comparator is the single most common Java-CP bug.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(5); pq.offer(1); pq.offer(3);
System.out.println(pq.poll()); // => 1   (smallest first! NOT 5 like C++ default)
```

**Max-heap in Java** — two ways:

```java
PriorityQueue<Integer> maxHeap1 = new PriorityQueue<>(Collections.reverseOrder());
PriorityQueue<Integer> maxHeap2 = new PriorityQueue<>((a, b) -> b - a);
maxHeap1.offer(5); maxHeap1.offer(1); maxHeap1.offer(3);
System.out.println(maxHeap1.poll()); // => 5
```

> **Gotcha:** `(a, b) -> b - a` **overflows** for large-magnitude ints (e.g. `a = Integer.MIN_VALUE`, `b` positive → `b - a` overflows and gives a wrong sign, corrupting heap order). Always use `Integer.compare(b, a)` instead — it's overflow-safe:

```java
PriorityQueue<Integer> safeMax = new PriorityQueue<>((a, b) -> Integer.compare(b, a));
safeMax.offer(Integer.MIN_VALUE);
safeMax.offer(5);
System.out.println(safeMax.peek());
// => 5   (correct; b-a form could have given the wrong order here)
```

**Custom comparator** — lambda or `Comparator.comparingInt`:

```java
record Task(String name, int priority) {}
PriorityQueue<Task> tasks = new PriorityQueue<>(Comparator.comparingInt(Task::priority));
tasks.offer(new Task("low", 5));
tasks.offer(new Task("high", 1));
System.out.println(tasks.poll().name());
// => high   (priority 1 is "smaller" -> comes out first, min-heap semantics)
```

**PQ of `int[]`** (common when you want a pair without allocating a record, e.g. Dijkstra `{dist, node}`):

```java
PriorityQueue<int[]> pqArr = new PriorityQueue<>((a, b) -> a[0] - b[0]); // compare by dist
pqArr.offer(new int[]{5, 100});
pqArr.offer(new int[]{2, 200});
System.out.println(Arrays.toString(pqArr.poll()));
// => [2, 200]
```

**PQ via encoded `long`** (pack two ints into one long to avoid array allocation entirely — a common CP micro-optimization):

```java
PriorityQueue<Long> pqLong = new PriorityQueue<>(); // natural order on the packed long
long dist = 2, node = 200;
long packed = (dist << 32) | node;
pqLong.offer(packed);
long top = pqLong.poll();
System.out.println((top >> 32) + " " + (top & 0xFFFFFFFFL));
// => 2 200
```

> **Gotcha:** packing into `long` only preserves correct ordering by the high bits if both fields are non-negative (or you offset them) — negative numbers' two's-complement bit patterns break naive shifting comparisons.

### 14.6 PQ methods & the lazy-deletion Dijkstra idiom

| Method | Does | Complexity |
|---|---|---|
| `offer(e)` / `add(e)` | insert | O(log n) |
| `poll()` | remove + return min (or `null` if empty) | O(log n) |
| `peek()` | return min without removing (or `null`) | O(1) |
| `remove(Object o)` | remove *specific* element (linear scan to find it!) | **O(n)** |
| `size()` | count | O(1) |

**No O(log n) remove-arbitrary, no decrease-key.** Unlike a textbook binary heap with decrease-key (or C++'s `indexed` structures / `pairing_heap` used in some libraries), Java's `PriorityQueue` can't efficiently update or remove an arbitrary element's priority. The standard workaround is **lazy deletion**: push new (better) entries without removing stale ones, and skip stale entries when popped.

```java
// Dijkstra with lazy deletion instead of decrease-key
int n = 4;
List<int[]>[] adj = new List[n];
for (int i = 0; i < n; i++) adj[i] = new ArrayList<>();
adj[0].add(new int[]{1, 4});
adj[0].add(new int[]{2, 1});
adj[2].add(new int[]{1, 1});
adj[1].add(new int[]{3, 1});
adj[2].add(new int[]{3, 5});

int[] dist = new int[n];
Arrays.fill(dist, Integer.MAX_VALUE);
dist[0] = 0;
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0])); // {dist, node}
pq.offer(new int[]{0, 0});

while (!pq.isEmpty()) {
    int[] cur = pq.poll();
    int d = cur[0], u = cur[1];
    if (d > dist[u]) continue; // STALE entry (a better one was already pushed) -> skip, this IS the lazy delete
    for (int[] edge : adj[u]) {
        int v = edge[0], w = edge[1];
        if (dist[u] + w < dist[v]) {
            dist[v] = dist[u] + w;
            pq.offer(new int[]{dist[v], v}); // push again instead of decrease-key
        }
    }
}
System.out.println(Arrays.toString(dist));
// => [0, 2, 1, 3]
```

### 14.7 Building a heap from a collection is O(n)

```java
List<Integer> data = List.of(5, 1, 9, 3, 7, 2);
PriorityQueue<Integer> pq2 = new PriorityQueue<>(data); // heapify, O(n), NOT O(n log n)
System.out.println(pq2.poll());
// => 1
```

> **Gotcha:** the `PriorityQueue(Collection)` constructor heapifies in O(n) — but repeatedly calling `offer` one element at a time to build the same heap is O(n log n). If you have all elements upfront, always prefer the collection constructor.

### 14.8 Complexity table

| Structure | push/offer | pop/poll | peek | random access |
|---|---|---|---|---|
| `ArrayDeque` (stack/queue use) | O(1) amortized | O(1) | O(1) | — |
| `PriorityQueue` | O(log n) | O(log n) | O(1) | — |
| `PriorityQueue` remove(Object) | — | O(n) | — | — |
| legacy `Stack` | O(1) amortized (synchronized) | O(1) (synchronized) | O(1) | O(1) (Vector-backed) |

---

## 15. BitSet

`java.util.BitSet` = a dynamically-growable bit vector.

> **vs C++:** C++ `std::bitset<N>` has a **fixed, compile-time size N** baked into the type. Java's `BitSet` **grows dynamically** at runtime (like a `vector<bool>` conceptually, but with bitset-style bulk operations) — you never need to know the size upfront. This is a genuine ergonomic win over C++ here.

### 15.1 Method table

| Method | Does | Complexity |
|---|---|---|
| `new BitSet()` / `new BitSet(nbits)` | empty bitset (optional initial capacity hint, still grows past it) | O(1) / O(nbits) |
| `set(i)` | set bit i to 1 | O(1) amortized |
| `set(i, val)` | set bit i to `val` (boolean) | O(1) amortized |
| `set(from, to)` | set bits `[from, to)` to 1 | O(range/64) |
| `clear(i)` | set bit i to 0 | O(1) |
| `clear()` | clear all bits | O(n/64) |
| `flip(i)` | toggle bit i | O(1) |
| `flip(from, to)` | toggle bits in range | O(range/64) |
| `get(i)` | value of bit i (false if beyond current size) | O(1) |
| `get(from, to)` | new `BitSet` = slice `[from, to)` | O(range/64) |
| `cardinality()` | popcount — number of set bits | O(n/64) |
| `length()` | index of highest set bit + 1 (0 if empty) | O(n/64) |
| `size()` | bits allocated internally (implementation detail, ≥ length) | O(1) |
| `isEmpty()` | no bits set? | O(n/64) |
| `and(other)` | in-place AND | O(n/64) |
| `or(other)` | in-place OR | O(n/64) |
| `xor(other)` | in-place XOR | O(n/64) |
| `andNot(other)` | in-place `this &= ~other` (set difference) | O(n/64) |
| `nextSetBit(i)` | first set bit at index ≥ i, or -1 | O(distance/64) |
| `nextClearBit(i)` | first clear bit at index ≥ i | O(distance/64) |
| `previousSetBit(i)` | last set bit at index ≤ i, or -1 | O(distance/64) |
| `stream()` **[Java 8]** | `IntStream` of set-bit indices | O(n/64) |
| `toString()` | e.g. `{0, 2, 5}` | O(n/64) |

### 15.2 Basics

```java
BitSet bs = new BitSet();
bs.set(2);
bs.set(5);
bs.set(7);
System.out.println(bs);
// => {2, 5, 7}

System.out.println(bs.cardinality()); // => 3  (popcount)
System.out.println(bs.get(5));        // => true
System.out.println(bs.get(3));        // => false
bs.flip(3);
System.out.println(bs);
// => {2, 3, 5, 7}
```

### 15.3 Bulk ops — the 64x speedup for subset DP / sieve / reachability

`and`/`or`/`xor`/`andNot` operate word-at-a-time (64 bits per machine word) instead of bit-by-bit, giving roughly a 64x constant-factor speedup over a `boolean[]` loop for large bitsets — this is the classic trick for **subset-sum DP**, **bitmask reachability**, and **bulk sieve marking**.

```java
// subset-sum reachability: dp.or(dp << w)  ==  achievable-sums bitset shifted and OR'd in
BitSet dp = new BitSet();
dp.set(0); // sum 0 is always achievable
int[] weights = {2, 3, 7};
for (int w : weights) {
    BitSet shifted = new BitSet();
    for (int i = dp.nextSetBit(0); i >= 0; i = dp.nextSetBit(i + 1)) {
        shifted.set(i + w);
    }
    dp.or(shifted); // dp |= (dp << w), the BitSet way
}
System.out.println(dp);
// => {0, 2, 3, 5, 7, 9, 10, 12}   (all achievable subset sums from {2,3,7})
```

```java
BitSet a = new BitSet(); a.set(1); a.set(2); a.set(3);
BitSet b = new BitSet(); b.set(2); b.set(3); b.set(4);

BitSet union = (BitSet) a.clone(); union.or(b);
BitSet inter = (BitSet) a.clone(); inter.and(b);
BitSet diff  = (BitSet) a.clone(); diff.andNot(b);

System.out.println(union); // => {1, 2, 3, 4}
System.out.println(inter); // => {2, 3}
System.out.println(diff);  // => {1}
```

> **Gotcha:** `and`/`or`/`xor`/`andNot` mutate the receiver in place, just like Set's `retainAll`/`addAll`/`removeAll` (§13.1) — `clone()` first if you need to preserve the original.

### 15.4 Iterate-set-bits idiom

```java
BitSet bs2 = new BitSet();
bs2.set(1); bs2.set(4); bs2.set(6);
for (int i = bs2.nextSetBit(0); i >= 0; i = bs2.nextSetBit(i + 1)) {
    System.out.print(i + " ");
}
System.out.println();
// => 1 4 6
```

This is THE idiom — `nextSetBit(i+1)` returns `-1` once no more set bits exist, terminating the loop.

### 15.5 `stream()` **[Java 8]**

```java
BitSet bs3 = new BitSet();
bs3.set(1); bs3.set(3); bs3.set(5);
int sum = bs3.stream().sum();
System.out.println(sum);
// => 9
```

### 15.6 Sieve of Eratosthenes with BitSet

```java
int n = 30;
BitSet composite = new BitSet(n + 1);
for (int i = 2; (long) i * i <= n; i++) {
    if (!composite.get(i)) {
        for (int j = i * i; j <= n; j += i) {
            composite.set(j);
        }
    }
}
List<Integer> primes = new ArrayList<>();
for (int i = 2; i <= n; i++) {
    if (!composite.get(i)) primes.add(i);
}
System.out.println(primes);
// => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

### 15.7 BitSet vs `long`-mask vs `boolean[]` — decision table

| Need | Use |
|---|---|
| ≤ 64 boolean flags, fixed known count | `long` bitmask (fastest — fits in a register, use `1L << i`, `&`, `\|`, `Long.bitCount`) |
| > 64 flags, size known/bounded but need bulk AND/OR/XOR (subset DP, sieve, reachability) | `BitSet` (word-parallel ops, dynamic size) |
| > 64 flags, need O(1) random access with no bulk bit ops, or need to store more than a single bit's worth of info conceptually per index | `boolean[]` (simplest, most predictable, sometimes fastest for pure random-access read/write with no bulk ops due to lower per-op overhead / JIT-friendliness) |
| Need a `Set<Integer>`-like API with iteration over "true" indices and dynamic growth | `BitSet` (`nextSetBit` idiom beats scanning a `boolean[]`) |
| Compile-time-fixed size, C++ codebase parity | closest Java analog is still `BitSet`, just don't rely on a fixed size |

---

## 16. Choosing the Right Structure

### 16.1 Decision table — requirement → Java structure → DSA pattern

| Requirement | Java structure | Typical DSA pattern |
|---|---|---|
| Need insertion order preserved, no dup keys | `LinkedHashMap` / `LinkedHashSet` | dedup preserving first-seen order, simple LRU (with access-order) |
| Need O(1) avg key lookup, no ordering | `HashMap` | frequency counting, memoization, adjacency list |
| Need O(1) avg membership, no ordering | `HashSet` | visited-set, dedup, set algebra |
| Need sorted iteration / range queries by key | `TreeMap` | ordered stats, calendar/interval maps |
| Need sorted iteration / range queries, unique elements | `TreeSet` | order statistics, coordinate compression scaffolding |
| Need predecessor/successor/floor/ceiling queries | `TreeMap`/`TreeSet` navigable methods | nearest-value queries, interval scheduling |
| Need duplicates + sorted + count-aware (multiset) | `TreeMap<T,Integer>` count-map | multiset simulation |
| Need LIFO access | `ArrayDeque` (as stack) | DFS iterative, monotonic stack, balanced parens |
| Need FIFO access | `ArrayDeque` (as queue) | BFS |
| Need both-end insert/remove | `ArrayDeque` (as deque) | sliding window max/min (monotonic deque), palindrome checks |
| Need fast min/max extraction, not full sort | `PriorityQueue` | Dijkstra, k-way merge, top-K, running median (two heaps) |
| Need O(1) recent-value expiry / LRU | `LinkedHashMap` (access-order) | LRU cache |
| Need dense boolean flags with bulk set ops | `BitSet` | subset-sum DP, sieve, reachability bitmasks |
| Need dense boolean flags, ≤ 64, max speed | `long` bitmask | bitmask DP over small n |
| Need array-like O(1) index + dynamic resize | `ArrayList` | general dynamic array use (see Part II) |
| Need O(1) head/tail insert with node semantics | `LinkedList` (rarely — `ArrayDeque` usually wins) | only when you truly need `List`+`Deque` combined |

### 16.2 C++ → Java container translation table

| C++ | Java | Note |
|---|---|---|
| `vector<T>` | `ArrayList<T>` | see Part II |
| `unordered_map<K,V>` | `HashMap<K,V>` | avg O(1), no order |
| `map<K,V>` | `TreeMap<K,V>` | sorted, navigable methods = lower_bound/upper_bound family |
| `unordered_set<T>` | `HashSet<T>` | avg O(1), no order |
| `set<T>` | `TreeSet<T>` | sorted, navigable methods |
| `multiset<T>` | `TreeMap<T,Integer>` (count-map) | **no built-in multiset in Java** — this is the workaround |
| `multimap<K,V>` | `Map<K, List<V>>` (e.g. `TreeMap<K,List<V>>`) | no built-in multimap either |
| `stack<T>` | `ArrayDeque<T>` (push/pop/peek) | not legacy `Stack` |
| `queue<T>` | `ArrayDeque<T>` (offer/poll/peek) | |
| `priority_queue<T>` | `PriorityQueue<T>` | **MIND THE FLIP: C++ default max-heap, Java default min-heap** |
| `deque<T>` | `ArrayDeque<T>` | |
| `bitset<N>` | `BitSet` | Java's grows dynamically, no fixed N |
| `pair<A,B>` | `record Pair<A,B>(A first, B second) {}` or encoded `long` | no built-in pair type in Java |
| `tuple<...>` | `record` with named fields | |
| `array<T,N>` | `T[]` (fixed-size Java array) | |
| `optional<T>` | `Optional<T>` **[Java 8]** | rarely used in CP hot loops |

### 16.3 Master complexity table — all structures (Parts II & III)

| Structure | Access/index | Search | Insert | Delete | Notes |
|---|---|---|---|---|---|
| `ArrayList` | O(1) | O(n) | O(1) amortized end / O(n) middle | O(n) | dynamic array |
| `LinkedList` | O(n) | O(n) | O(1) at known node | O(1) at known node | rarely wins over ArrayDeque/ArrayList in practice |
| `ArrayDeque` | — | O(n) | O(1) amortized both ends | O(1) amortized both ends | preferred stack/queue/deque |
| legacy `Stack` | O(1) (Vector-backed) | O(n) | O(1) amortized | O(1) amortized | synchronized, slower — avoid |
| `HashMap` | — | O(1) avg / O(n) worst | O(1) avg | O(1) avg | no order |
| `TreeMap` | — | O(log n) | O(log n) | O(log n) | sorted, navigable |
| `LinkedHashMap` | — | O(1) avg | O(1) avg | O(1) avg | insertion/access order |
| `HashSet` | — | O(1) avg / O(n) worst | O(1) avg | O(1) avg | no order |
| `TreeSet` | — | O(log n) | O(log n) | O(log n) | sorted, navigable |
| `LinkedHashSet` | — | O(1) avg | O(1) avg | O(1) avg | insertion order |
| `PriorityQueue` | O(1) peek only | O(n) (not min/max) | O(log n) | O(log n) poll / **O(n) remove(Object)** | min-heap by default |
| `BitSet` | O(1) get/set | O(1) get | O(1) amortized | O(1) amortized | bulk ops O(n/64) |
| Plain array `T[]` | O(1) | O(n) unsorted / O(log n) sorted+binarySearch | O(n) (fixed size, no resize) | O(n) | fastest raw baseline |

> **Gotcha recap across Parts II & III:** unboxing NPE from `Map.get`/autoboxed containers on primitives, `PriorityQueue` min-heap default, `TreeSet`/`TreeMap` comparator-defines-equality trap, `ArrayDeque` rejecting `null`, `ConcurrentModificationException` on structural changes during iteration (use an `Iterator`'s own `remove()`, or iterate a copy), and no built-in multiset/multimap/pair in the JDK — all recurring themes when porting C++ CP code to Java.

---

# PART IV — SORTING, SEARCHING, COMPARATORS AND STREAMS

## 17. Sorting

### 17.1 The single most important CP-Java sorting fact: the anti-quicksort hack

`Arrays.sort(int[])` (and other primitive-array overloads) uses a **dual-pivot quicksort**. It has no worst-case guarantee — Codeforces (and other judges) keep **adversarial anti-quicksort test data** that forces `Arrays.sort(int[])` into `O(n^2)` and TLEs you. This is the Java analogue of the classic C++ `unordered_map` hack, except here it hits the *default sort itself*, not just hashing.

`Arrays.sort(Object[])` (so `Integer[]`, `Long[]`, `List<T>` via `Collections.sort`/`List.sort`) uses **TimSort** — a merge-based, adaptive, **stable**, guaranteed `O(n log n)` algorithm. It is immune to this hack because it doesn't pick pivots off the input pattern.

| Array type | Algorithm | Worst case | Stable | CF-safe |
|---|---|---|---|---|
| `int[]`, `long[]`, `double[]`, ... (primitives) | Dual-pivot quicksort | `O(n^2)` (adversarial) | No | **NO — hackable** |
| `Integer[]`, `Long[]`, `Object[]`, `List<T>` | TimSort | `O(n log n)` | Yes | Yes |

> **Gotcha:** Never submit `Arrays.sort(int[])` unguarded on Codeforces for an array you didn't generate randomly yourself (e.g. adjacency-derived arrays, arrays built from input order). Problem setters specifically construct killer inputs for Java's dual-pivot quicksort.

**Fix (a): shuffle before sorting the primitive array.** Cheapest fix, keeps `int[]` (fast, no boxing).

```java
import java.util.*;

public class ShuffleSort {
    static void shuffleSort(int[] a) {
        Random rnd = new Random();
        for (int i = a.length - 1; i > 0; i--) {
            int j = rnd.nextInt(i + 1);
            int tmp = a[i]; a[i] = a[j]; a[j] = tmp;
        }
        Arrays.sort(a);
    }

    public static void main(String[] args) {
        int[] a = {5, 3, 4, 1, 2};
        shuffleSort(a);
        System.out.println(Arrays.toString(a)); // => [1, 2, 3, 4, 5]
    }
}
```

**Fix (b): box to `Integer[]` and let TimSort handle it.** Slightly slower / more memory (boxing overhead) but no shuffle needed and you get stability for free.

```java
import java.util.*;
import java.util.stream.*;

public class BoxedSort {
    public static void main(String[] args) {
        int[] prim = {5, 3, 4, 1, 2};
        Integer[] boxed = Arrays.stream(prim).boxed().toArray(Integer[]::new);
        Arrays.sort(boxed); // TimSort, guaranteed O(n log n), CF-safe
        System.out.println(Arrays.toString(boxed)); // => [1, 2, 3, 4, 5]

        // unbox back if you need int[] afterward
        int[] back = Arrays.stream(boxed).mapToInt(Integer::intValue).toArray();
        System.out.println(Arrays.toString(back)); // => [1, 2, 3, 4, 5]
    }
}
```

> **Rule of thumb:** if the array is small or you're already boxing (e.g. sorting with a custom comparator), just use `Integer[]`. If you need raw `int[]` performance for a huge array, shuffle first.

### 17.2 Sorting method table

| Method | Signature | What it does | Complexity |
|---|---|---|---|
| `Arrays.sort` | `sort(int[] a)` | Dual-pivot quicksort, ascending, **not stable, hackable** | `O(n log n)` avg, `O(n^2)` worst |
| `Arrays.sort` | `sort(int[] a, int from, int to)` | Sorts sub-range `[from, to)` | same as above |
| `Arrays.sort` | `sort(Integer[] a)` / `sort(Object[] a)` | TimSort, ascending (natural order), **stable** | `O(n log n)` guaranteed |
| `Arrays.sort` | `sort(T[] a, Comparator<? super T> c)` | TimSort with custom order | `O(n log n)` |
| `Collections.sort` | `sort(List<T> l)` | TimSort on a `List`, natural order | `O(n log n)` |
| `Collections.sort` | `sort(List<T> l, Comparator<? super T> c)` | TimSort on a `List`, custom order | `O(n log n)` |
| `List.sort` | `l.sort(Comparator)` (default method, Java 8+) | In-place sort of any `List` | `O(n log n)` |

### 17.3 Comparator sorting requires boxing (real friction vs C++)

In C++, `std::sort(v.begin(), v.end(), cmp)` works uniformly whether `v` is `vector<int>` or `vector<pair<int,int>>`. In Java, `Arrays.sort` with a `Comparator` is **only defined on `Object[]` / `List<T>`**, never on `int[]`, `long[]`, etc. — primitives have no comparator overload.

```java
int[] a = {5, 1, 4};
// Arrays.sort(a, (x, y) -> y - x);   // COMPILE ERROR: no such overload for int[]

Integer[] b = {5, 1, 4};
Arrays.sort(b, (x, y) -> y - x);      // OK — Object[] only
System.out.println(Arrays.toString(b)); // => [5, 4, 1]
```

> **vs C++:** `std::sort` is a template that works on any iterator range with any comparator, primitive or not. Java's primitive-array sort and comparator-based sort are two completely separate code paths — if you need custom order on numbers, you must box.

### 17.4 Descending order

```java
import java.util.*;

public class DescendingSort {
    public static void main(String[] args) {
        Integer[] a = {5, 3, 4, 1, 2};
        Arrays.sort(a, Collections.reverseOrder());
        System.out.println(Arrays.toString(a)); // => [5, 4, 3, 2, 1]

        Arrays.sort(a, Comparator.reverseOrder()); // equivalent, java.util.Comparator
        System.out.println(Arrays.toString(a)); // => [5, 4, 3, 2, 1]

        List<Integer> list = new ArrayList<>(Arrays.asList(5, 3, 4, 1, 2));
        list.sort(Collections.reverseOrder());
        System.out.println(list); // => [5, 4, 3, 2, 1]

        // primitive int[] descending: sort ascending then reverse manually (no comparator overload)
        int[] p = {5, 3, 4, 1, 2};
        Arrays.sort(p);
        for (int i = 0, j = p.length - 1; i < j; i++, j--) {
            int t = p[i]; p[i] = p[j]; p[j] = t;
        }
        System.out.println(Arrays.toString(p)); // => [5, 4, 3, 2, 1]
    }
}
```

### 17.5 Sorting 2D arrays

```java
import java.util.*;

public class Sort2D {
    public static void main(String[] args) {
        int[][] arr = {{3, 1}, {1, 2}, {2, 0}};

        // ascending by column 0 — naive subtraction (works here, but see overflow trap below)
        Arrays.sort(arr, (a, b) -> a[0] - b[0]);
        System.out.println(Arrays.deepToString(arr)); // => [[1, 2], [2, 0], [3, 1]]

        // overflow-safe version — ALWAYS prefer this
        Arrays.sort(arr, (a, b) -> Integer.compare(a[0], b[0]));
        System.out.println(Arrays.deepToString(arr)); // => [[1, 2], [2, 0], [3, 1]]
    }
}
```

> **Gotcha:** `a[0] - b[0]` overflows silently when values are near `Integer.MIN_VALUE`/`MAX_VALUE`, producing a wrong (even inconsistent) ordering. Always use `Integer.compare` / `Long.compare`. Full discussion in §21.1.

### 17.6 Sorting `List<int[]>` / `List<Pair>`

```java
import java.util.*;

public class SortListOfArrays {
    public static void main(String[] args) {
        List<int[]> list = new ArrayList<>();
        list.add(new int[]{3, 1});
        list.add(new int[]{1, 2});
        list.add(new int[]{2, 0});

        list.sort((a, b) -> Integer.compare(a[0], b[0]));
        for (int[] p : list) System.out.print(Arrays.toString(p) + " ");
        // => [1, 2] [2, 0] [3, 1]
    }
}
```

Java has no built-in `Pair` (unlike `std::pair`) — CP coders typically use `int[]` of length 2, a record (`record Pair(int a, int b) {}`), or `Map.Entry`. See Part III for details.

### 17.7 Argsort — sorting indices by value

Java has no direct "sort indices by a key array" utility; box the indices to `Integer[]` and sort with a comparator that reads the backing array.

```java
import java.util.*;

public class ArgSort {
    public static void main(String[] args) {
        int[] val = {40, 10, 30, 20};
        Integer[] idx = new Integer[val.length];
        Arrays.setAll(idx, i -> i); // idx = [0,1,2,3]

        Arrays.sort(idx, (i, j) -> Integer.compare(val[i], val[j]));
        System.out.println(Arrays.toString(idx)); // => [1, 3, 2, 0]  (order of val ascending)
    }
}
```

> **Gotcha:** you cannot capture `val` if it's non-effectively-final in the enclosing scope — arrays are fine since the *reference* is final even though contents mutate, but reassigning `val = ...` inside the same scope after the lambda breaks compilation.

### 17.8 Stability note

- Primitive-array `Arrays.sort` (dual-pivot quicksort): **NOT stable** — equal elements may be reordered. Irrelevant for bare `int[]` (no payload to lose), but matters if you're sorting parallel arrays.
- Object-array / `List` sort (TimSort): **stable** — equal elements retain relative input order. Critical when you sort by one key, then need a secondary tie-break to preserve original order (or when you do two sequential stable sorts to fake multi-key sort — see §21).

```java
import java.util.*;

public class StabilityDemo {
    record Person(String name, int age) {}
    public static void main(String[] args) {
        List<Person> people = new ArrayList<>(List.of(
            new Person("A", 30), new Person("B", 25), new Person("C", 30)
        ));
        people.sort(Comparator.comparingInt(Person::age));
        System.out.println(people);
        // => [Person[name=B, age=25], Person[name=A, age=30], Person[name=C, age=30]]
        // A before C preserved because TimSort is stable
    }
}
```

---

## 18. Binary Search

### 18.1 `Arrays.binarySearch` — return value semantics

Requires the array to be **sorted ascending** first (undefined behavior otherwise).

- If found: returns **some** index containing the key (unspecified which one if duplicates exist).
- If NOT found: returns `-(insertionPoint) - 1`, where `insertionPoint` is the index the key would be inserted at to keep the array sorted (i.e., the index of the first element greater than key, or `a.length` if key is bigger than everything).

```java
import java.util.*;

public class BinarySearchDemo {
    public static void main(String[] args) {
        int[] a = {10, 20, 20, 20, 30, 40};

        System.out.println(Arrays.binarySearch(a, 20)); // => 2 (any index in [1,3] is valid, JDK impl-defined)
        System.out.println(Arrays.binarySearch(a, 25)); // => -5 (insertion point 4 -> -4-1)
        System.out.println(Arrays.binarySearch(a, 5));  // => -1 (insertion point 0 -> -0-1)
        System.out.println(Arrays.binarySearch(a, 50)); // => -7 (insertion point 6 -> -6-1)

        // sub-range search
        System.out.println(Arrays.binarySearch(a, 1, 4, 20)); // search only a[1..4)
    }
}
```

| key | found? | insertionPoint | returned value |
|---|---|---|---|
| 20 | yes (dup) | — | some index in `[1,3]` |
| 25 | no | 4 | `-4-1 = -5` |
| 5  | no | 0 | `-0-1 = -1` |
| 50 | no | 6 (past end) | `-6-1 = -7` |

**Converting a negative result to the insertion point** (lower_bound-style):

```java
int idx = Arrays.binarySearch(a, key);
if (idx < 0) idx = -idx - 1; // idx now = first position >= key (or a.length if none)
```

`Collections.binarySearch(List<T> list, T key)` — same semantics, works on any `List` implementing `RandomAccess` efficiently (linked lists degrade to `O(n log n)` due to sequential access cost).

> **Gotcha:** if the array has duplicates, `Arrays.binarySearch` does NOT guarantee returning the first or last occurrence — it's whichever the internal probing happens to land on. Never rely on it for "find leftmost/rightmost equal element" — use hand-rolled `lowerBound`/`upperBound` instead (below).

### 18.2 No built-in lower_bound / upper_bound — a real gap vs C++

> **vs C++:** `std::lower_bound` / `std::upper_bound` are standard library one-liners. Java has **nothing equivalent** in `java.util` — you must hand-roll them (or abuse `Arrays.binarySearch`'s insertion-point trick, which only gives you a lower-bound-ish answer and breaks down with duplicates for upper-bound). Keep these as copy-paste templates.

```java
public class BoundHelpers {
    // first index i such that a[i] >= key  (== C++ lower_bound)
    static int lowerBound(int[] a, int key) {
        int lo = 0, hi = a.length; // search space [lo, hi)
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] >= key) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }

    // first index i such that a[i] > key  (== C++ upper_bound)
    static int upperBound(int[] a, int key) {
        int lo = 0, hi = a.length;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] > key) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }

    public static void main(String[] args) {
        int[] a = {10, 20, 20, 20, 30, 40};
        System.out.println(lowerBound(a, 20)); // => 1  (first index with a[i] >= 20)
        System.out.println(upperBound(a, 20)); // => 4  (first index with a[i] > 20)
        System.out.println(lowerBound(a, 25)); // => 4
        System.out.println(upperBound(a, 25)); // => 4
    }
}
```

### 18.3 Counting elements in `[l, r]`

```java
// count of a[i] with l <= a[i] <= r, a sorted ascending
static int countInRange(int[] a, int l, int r) {
    return upperBound(a, r) - lowerBound(a, l);
}
```

```java
int[] a = {10, 20, 20, 20, 30, 40};
System.out.println(countInRange(a, 20, 30)); // => 4  (three 20s + one 30)
```

### 18.4 Generic "binary search on the answer" template

The workhorse for CP: monotonic predicate `f(x)` that is `false...false true...true` (or the reverse) — find the boundary.

**`[lo, hi)` half-open form** — finds the first `x` in `[lo, hi)` where `predicate(x)` is true, assuming predicate is monotonic false→true:

```java
public class SearchOnAnswer {
    interface IntPredicate2 { boolean test(int x); }

    static int firstTrue(int lo, int hi, IntPredicate2 predicate) {
        // invariant: predicate(hi) assumed true (or hi is a sentinel "infinity")
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2; // overflow-safe midpoint
            if (predicate.test(mid)) hi = mid;
            else lo = mid + 1;
        }
        return lo; // first x with predicate(x) == true
    }

    public static void main(String[] args) {
        // example: smallest x such that x*x >= 50
        int ans = firstTrue(0, 100, x -> (long) x * x >= 50);
        System.out.println(ans); // => 8   (7*7=49 < 50, 8*8=64 >= 50)
    }
}
```

**`[lo, hi]` closed-interval predicate form** — the other common idiom, useful when hi is a real inclusive bound (e.g. max possible answer):

```java
static long maxTrue(long lo, long hi, java.util.function.LongPredicate predicate) {
    // predicate is monotonic true...true false...false over [lo, hi]; find last true
    long ans = lo - 1; // sentinel: "no valid answer found"
    while (lo <= hi) {
        long mid = lo + (hi - lo) / 2; // overflow-safe
        if (predicate.test(mid)) { ans = mid; lo = mid + 1; }
        else hi = mid - 1;
    }
    return ans;
}
```

```java
// example: largest x (1..1000) such that x*x <= 50
long ans = maxTrue(1, 1000, x -> x * x <= 50);
System.out.println(ans); // => 7
```

> **Gotcha:** `mid = (lo + hi) / 2` overflows for large `lo`/`hi` (same trap as C++). Always write `mid = lo + (hi - lo) / 2`.

### 18.5 TreeSet/TreeMap floor/ceiling as the idiomatic ordered-search alternative

For dynamic sets (insertions/deletions interleaved with queries), don't re-sort an array — use `TreeSet`/`TreeMap`, whose `floor`/`ceiling`/`lower`/`higher` give `O(log n)` ordered search directly (cross-ref Part III §Ordered Collections).

```java
import java.util.*;

public class TreeSetSearch {
    public static void main(String[] args) {
        TreeSet<Integer> ts = new TreeSet<>(List.of(10, 20, 30, 40));
        System.out.println(ts.floor(25));   // => 20  (largest <= 25)
        System.out.println(ts.ceiling(25)); // => 30  (smallest >= 25)
        System.out.println(ts.lower(20));   // => 10  (largest < 20)
        System.out.println(ts.higher(20));  // => 30  (smallest > 20)
    }
}
```

---

## 19. Collections and Arrays Utility Methods

### 19.1 `Collections` static methods

| Method | Signature | What it does | Complexity |
|---|---|---|---|
| `sort` | `sort(List<T>)` / `sort(List<T>, Comparator)` | TimSort in place | `O(n log n)` |
| `reverse` | `reverse(List<T>)` | Reverses list in place | `O(n)` |
| `shuffle` | `shuffle(List<T>)` / `shuffle(List<T>, Random)` | Randomly permutes | `O(n)` |
| `min` / `max` | `min(Collection<T>)` / with `Comparator` | Extremum by natural/custom order | `O(n)` |
| `swap` | `swap(List<T>, i, j)` | Swaps two elements | `O(1)` |
| `fill` | `fill(List<T>, T val)` | Overwrites every slot | `O(n)` |
| `nCopies` | `nCopies(int n, T o)` | Immutable list of `n` copies of `o` | `O(1)` (lazy view) |
| `frequency` | `frequency(Collection<?>, Object)` | Counts occurrences | `O(n)` |
| `binarySearch` | `binarySearch(List<T>, T key)` | Like `Arrays.binarySearch` | `O(log n)` (random access) |
| `emptyList` / `singletonList` | `emptyList()` / `singletonList(T)` | Immutable 0/1-element lists | `O(1)` |
| `unmodifiableList` | `unmodifiableList(List<T>)` | Read-only view (backing list still mutable via original ref) | `O(1)` |
| `addAll` | `addAll(Collection<T>, T...)` | Bulk-add varargs into a collection | `O(k)` |
| `rotate` | `rotate(List<T>, int dist)` | Cyclic-shifts elements | `O(n)` |
| `disjoint` | `disjoint(Collection<?>, Collection<?>)` | true if no common elements | `O(n+m)` |

```java
import java.util.*;

public class CollectionsUtilDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(List.of(5, 3, 4, 1, 2));

        Collections.sort(list);
        System.out.println(list); // => [1, 2, 3, 4, 5]

        Collections.reverse(list);
        System.out.println(list); // => [5, 4, 3, 2, 1]

        Random rnd = new Random(42); // fixed seed => reproducible shuffle
        Collections.shuffle(list, rnd);
        System.out.println(list); // => (deterministic given seed 42, e.g.) [3, 1, 5, 2, 4]

        System.out.println(Collections.min(list)); // => 1
        System.out.println(Collections.max(list)); // => 5

        Collections.swap(list, 0, 1);
        System.out.println(list); // swaps first two elements of current order

        List<Integer> filled = new ArrayList<>(List.of(0, 0, 0));
        Collections.fill(filled, 9);
        System.out.println(filled); // => [9, 9, 9]

        List<String> copies = Collections.nCopies(3, "x");
        System.out.println(copies); // => [x, x, x]

        System.out.println(Collections.frequency(List.of(1, 2, 2, 3, 2), 2)); // => 3

        List<Integer> sorted = new ArrayList<>(List.of(1, 3, 5, 7, 9));
        System.out.println(Collections.binarySearch(sorted, 5)); // => 2

        System.out.println(Collections.emptyList());       // => []
        System.out.println(Collections.singletonList(7));  // => [7]

        List<Integer> ro = Collections.unmodifiableList(sorted);
        // ro.add(1); // would throw UnsupportedOperationException

        List<Integer> dest = new ArrayList<>();
        Collections.addAll(dest, 1, 2, 3);
        System.out.println(dest); // => [1, 2, 3]

        List<Integer> rot = new ArrayList<>(List.of(1, 2, 3, 4, 5));
        Collections.rotate(rot, 2);
        System.out.println(rot); // => [4, 5, 1, 2, 3]

        System.out.println(Collections.disjoint(List.of(1, 2), List.of(3, 4))); // => true
    }
}
```

> **Gotcha:** `Collections.shuffle` without an explicit `Random` seed uses a default `Random` internally — fine for the anti-quicksort defense in §17.1, but pass a fixed-seed `Random` when you need a **reproducible** shuffle for debugging.

### 19.2 `Arrays` static methods

| Method | Signature | What it does | Complexity |
|---|---|---|---|
| `sort` | `sort(T[])` / `sort(int[])` | See §17 | `O(n log n)` |
| `fill` | `fill(int[] a, int val)` | Fills entire array | `O(n)` |
| `copyOf` | `copyOf(int[] a, int newLen)` | New array, truncated/zero-padded | `O(n)` |
| `copyOfRange` | `copyOfRange(int[] a, from, to)` | New array = slice `[from,to)` | `O(to-from)` |
| `equals` | `equals(int[] a, int[] b)` | Element-wise equality (1D) | `O(n)` |
| `deepEquals` | `deepEquals(Object[] a, Object[] b)` | Recursive equality (nested arrays) | `O(n)` total |
| `toString` | `toString(int[])` | `"[1, 2, 3]"` | `O(n)` |
| `deepToString` | `deepToString(Object[])` | Recursive stringify (2D+) | `O(n)` total |
| `asList` | `asList(T... a)` | Fixed-size `List` view backed by array | `O(1)` |
| `stream` | `stream(int[])` | `IntStream` view | `O(1)` lazy |
| `binarySearch` | see §18.1 | | `O(log n)` |
| `hashCode` | `hashCode(int[])` | Content-based hash | `O(n)` |
| `setAll` | `setAll(int[] a, IntUnaryOperator f)` | Fills `a[i] = f.apply(i)` | `O(n)` |

```java
import java.util.*;
import java.util.stream.*;

public class ArraysUtilDemo {
    public static void main(String[] args) {
        int[] a = new int[5];
        Arrays.fill(a, 7);
        System.out.println(Arrays.toString(a)); // => [7, 7, 7, 7, 7]

        int[] b = Arrays.copyOf(a, 8);
        System.out.println(Arrays.toString(b)); // => [7, 7, 7, 7, 7, 0, 0, 0]

        int[] c = Arrays.copyOfRange(a, 1, 3);
        System.out.println(Arrays.toString(c)); // => [7, 7]

        System.out.println(Arrays.equals(new int[]{1,2}, new int[]{1,2})); // => true

        int[][] m1 = {{1,2},{3,4}}, m2 = {{1,2},{3,4}};
        System.out.println(Arrays.deepEquals(m1, m2)); // => true
        System.out.println(Arrays.deepToString(m1));   // => [[1, 2], [3, 4]]

        Integer[] boxedArr = {1, 2, 3};
        List<Integer> view = Arrays.asList(boxedArr); // fixed-size, backed by boxedArr
        view.set(0, 99);
        System.out.println(boxedArr[0]); // => 99 (view writes through to the array)
        // view.add(4); // would throw UnsupportedOperationException — fixed size

        int[] arr = {3, 1, 2};
        int sum = Arrays.stream(arr).sum();
        System.out.println(sum); // => 6

        int[] iota = new int[5];
        Arrays.setAll(iota, i -> i); // iota == C++ std::iota
        System.out.println(Arrays.toString(iota)); // => [0, 1, 2, 3, 4]
    }
}
```

> **Gotcha:** `Arrays.asList(int[])` does NOT do what you expect — with a **primitive** `int[]` it produces a `List<int[]>` of size 1 (the whole array is treated as a single generic-type element), because generics can't take a primitive type argument. Only works correctly with `Integer[]`, `T[]`.

```java
int[] p = {1, 2, 3};
List<int[]> weird = Arrays.asList(p);
System.out.println(weird.size()); // => 1  (NOT 3!)
```

### 19.3 `Math` static methods

| Method | Signature | What it does | Complexity |
|---|---|---|---|
| `max` / `min` | `max(int,int)`, ... | Only **2-arg** overloads (int/long/float/double) | `O(1)` |
| `abs` | `abs(int)` | Absolute value | `O(1)` |
| `pow` | `pow(double, double)` → `double` | Power, always returns `double` | `O(1)` |
| `sqrt` | `sqrt(double)` → `double` | Square root | `O(1)` |
| `floor` / `ceil` | `floor(double)`, `ceil(double)` → `double` | Round down/up | `O(1)` |
| `round` | `round(double)` → `long` | Round to nearest | `O(1)` |
| `floorDiv` | `floorDiv(int,int)` | Signed division rounding toward -infinity | `O(1)` |
| `floorMod` | `floorMod(int,int)` | Correct signed modulo (result has divisor's sign) | `O(1)` |
| `log` | `log(double)` | Natural log | `O(1)` |
| `hypot` | `hypot(double,double)` | `sqrt(x^2+y^2)` without intermediate overflow | `O(1)` |
| `addExact`/`subtractExact`/`multiplyExact` | `(int,int)`→`int` | Throws `ArithmeticException` on overflow | `O(1)` |

```java
public class MathDemo {
    public static void main(String[] args) {
        System.out.println(Math.max(3, 7)); // => 7
        // Math.max(1, 2, 3); // COMPILE ERROR — only 2-arg overloads exist, unlike C++ max({1,2,3})
        System.out.println(Math.max(Math.max(1, 2), 3)); // => 3, must nest manually

        System.out.println(Math.abs(-5)); // => 5
        // OVERFLOW TRAP: abs(Integer.MIN_VALUE) cannot be represented as a positive int
        System.out.println(Math.abs(Integer.MIN_VALUE)); // => -2147483648 (still negative!)

        // pow returns double — imprecise for large integer powers
        System.out.println((long) Math.pow(2, 62)); // => 4611686018427387904 (may be off by rounding for big exponents)
        System.out.println(fastPow(2, 62)); // => 4611686018427387904 (exact, integer-only)

        System.out.println((int) Math.sqrt(16));      // => 4, correct
        System.out.println((int) Math.sqrt(25) );     // => 5, but for some doubles floating error rounds DOWN
        // safe pattern: probe neighbors after truncation
        int r = (int) Math.sqrt(25);
        while ((long)(r+1) * (r+1) <= 25) r++;
        while ((long) r * r > 25) r--;
        System.out.println(r); // => 5 (corrected)

        System.out.println(Math.floorDiv(-7, 3));  // => -3  (rounds toward -infinity)
        System.out.println(-7 / 3);                 // => -2  (Java's % / truncate toward 0)
        System.out.println(Math.floorMod(-7, 3));  // => 2   (CORRECT signed modulo, result >= 0 when divisor > 0)
        System.out.println(-7 % 3);                 // => -1  (Java's % keeps dividend's sign)

        System.out.println(Math.log(Math.E)); // => 1.0
        double log2of8 = Math.log(8) / Math.log(2); // change of base for log2
        System.out.println(log2of8); // => 3.0

        System.out.println(Math.hypot(3, 4)); // => 5.0

        try {
            Math.addExact(Integer.MAX_VALUE, 1); // throws
        } catch (ArithmeticException e) {
            System.out.println("overflow caught: " + e.getMessage()); // => overflow caught: integer overflow
        }
    }

    static long fastPow(long base, long exp) {
        long result = 1;
        while (exp > 0) {
            if ((exp & 1) == 1) result *= base;
            base *= base;
            exp >>= 1;
        }
        return result;
    }
}
```

> **Gotcha:** `Math.abs(Integer.MIN_VALUE)` returns `Integer.MIN_VALUE` unchanged (still negative) because `+2147483648` doesn't fit in `int`. Same trap exists in C++ with `INT_MIN`. Widen to `long` first if this value is reachable.

> **Gotcha:** Java's `%` is a **remainder** operator (result takes the sign of the dividend), same as C++'s `%` since C++11. Neither is true mathematical modulo. Use `Math.floorMod` when you need a non-negative result (e.g. circular array indexing).

> **vs C++:** `Math.max`/`Math.min` only take exactly 2 arguments (overloaded per primitive type) — there's no variadic form and no initializer-list form like C++'s `std::max({a, b, c})`. Nest calls or use `Collections.max(List.of(...))` for 3+ values.

### 19.4 `Integer` / `Long` static utilities

| Method | Signature | What it does |
|---|---|---|
| `compare` | `Integer.compare(int,int)` | -1/0/1, overflow-safe (see §21.1) |
| `max`/`min`/`sum` | `Integer.max(a,b)` etc. | Convenience wrappers, same as `Math.max` for ints |
| `parseInt` | `Integer.parseInt(String)` | String → int, throws `NumberFormatException` on bad input |
| `toBinaryString` | `Integer.toBinaryString(int)` | Unsigned binary representation |
| `bitCount` | `Integer.bitCount(int)` | Popcount (forward-ref Part V) |
| `highestOneBit` | `Integer.highestOneBit(int)` | Isolates MSB set bit |
| `numberOfLeadingZeros` / `numberOfTrailingZeros` | `(int)` | Bit-scan utilities (forward-ref Part V) |
| `MAX_VALUE` / `MIN_VALUE` | constants | `2147483647` / `-2147483648` |

```java
public class IntegerUtilDemo {
    public static void main(String[] args) {
        System.out.println(Integer.compare(3, 7));       // => -1
        System.out.println(Integer.max(3, 7));            // => 7
        System.out.println(Integer.parseInt("42"));       // => 42
        System.out.println(Integer.toBinaryString(10));   // => 1010
        System.out.println(Integer.bitCount(7));          // => 3
        System.out.println(Integer.highestOneBit(10));    // => 8
        System.out.println(Integer.numberOfLeadingZeros(1));  // => 31
        System.out.println(Integer.numberOfTrailingZeros(8)); // => 3
        System.out.println(Integer.MAX_VALUE);            // => 2147483647
        System.out.println(Long.MAX_VALUE);                // => 9223372036854775807
    }
}
```

---

## 20. Iterators and Iterable

### 20.1 `Iterator<T>` and `Iterable<T>`

```java
public interface Iterator<T> {
    boolean hasNext();
    T next();
    default void remove() { throw new UnsupportedOperationException(); }
}

public interface Iterable<T> {
    Iterator<T> iterator();
}
```

Any class implementing `Iterable<T>` can be used in a for-each loop. The for-each loop is pure syntactic sugar:

```java
for (String s : list) { System.out.println(s); }

// desugars to:
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}
```

### 20.2 `Iterator.remove()` — the ONLY safe way to remove during iteration

```java
import java.util.*;

public class SafeRemoveDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4, 5));

        // UNSAFE: throws ConcurrentModificationException
        try {
            for (int x : list) {
                if (x % 2 == 0) list.remove(Integer.valueOf(x));
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("CME caught"); // => CME caught
        }

        // SAFE: use Iterator.remove()
        List<Integer> list2 = new ArrayList<>(List.of(1, 2, 3, 4, 5));
        Iterator<Integer> it = list2.iterator();
        while (it.hasNext()) {
            int x = it.next();
            if (x % 2 == 0) it.remove();
        }
        System.out.println(list2); // => [1, 3, 5]
    }
}
```

> **Gotcha:** modifying a collection (add/remove) directly while a for-each loop or another iterator is active over it throws `ConcurrentModificationException` at the *next* `next()`/`hasNext()` call — this is Java's **fail-fast** behavior, deliberately detected via an internal `modCount`. C++ has a similar concept (iterator invalidation) but it's usually silent UB, not an exception — Java's fail-fast at least surfaces the bug loudly.

### 20.3 `ListIterator` — bidirectional iteration with mutation

```java
import java.util.*;

public class ListIteratorDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
        ListIterator<Integer> lit = list.listIterator();

        while (lit.hasNext()) {
            int x = lit.next();
            lit.set(x * 10); // replace current element
        }
        System.out.println(list); // => [10, 20, 30]

        while (lit.hasPrevious()) {
            int x = lit.previous();
            System.out.print(x + " "); // => 30 20 10
        }
        System.out.println();

        lit = list.listIterator();
        lit.next();
        lit.add(99); // inserts before the next element
        System.out.println(list); // => [10, 99, 20, 30]
    }
}
```

### 20.4 `descendingIterator` on `Deque` / `NavigableSet`

```java
import java.util.*;

public class DescendingIteratorDemo {
    public static void main(String[] args) {
        Deque<Integer> deque = new ArrayDeque<>(List.of(1, 2, 3));
        Iterator<Integer> desc = deque.descendingIterator();
        while (desc.hasNext()) System.out.print(desc.next() + " "); // => 3 2 1
        System.out.println();

        TreeSet<Integer> ts = new TreeSet<>(List.of(1, 2, 3));
        Iterator<Integer> tsDesc = ts.descendingIterator();
        while (tsDesc.hasNext()) System.out.print(tsDesc.next() + " "); // => 3 2 1
        System.out.println();
    }
}
```

### 20.5 Iterating a `Map` via `entrySet`

`Map` is not itself `Iterable` (no single natural element type — it's key/value pairs) — you get an iterator through one of its **views**: `entrySet()`, `keySet()`, or `values()`.

```java
import java.util.*;

public class MapIteratorDemo {
    public static void main(String[] args) {
        Map<String, Integer> map = new LinkedHashMap<>();
        map.put("a", 1); map.put("b", 2);

        Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<String, Integer> e = it.next();
            System.out.println(e.getKey() + "=" + e.getValue());
        }
        // => a=1
        // => b=2

        // Iterator.remove() on entrySet() removes from the underlying map too
        it = map.entrySet().iterator();
        while (it.hasNext()) {
            if (it.next().getKey().equals("a")) it.remove();
        }
        System.out.println(map); // => {b=2}
    }
}
```

### 20.6 No pointer-arithmetic iterators like C++

> **vs C++:** C++ iterators are generalized pointers — you can do `it + 5`, `it - it2` (random-access iterators), dereference with `*it`, and freely copy/compare raw positions. Java's `Iterator` is a **cursor abstraction**: only `hasNext()`/`next()`/`remove()` — no arithmetic, no dereference-without-advance, no distance calculation between two iterators. This is a deliberate simplification: it works uniformly across linked lists, hash tables, and trees where "add 5 to a pointer" wouldn't make sense. If you need random access, use the `List` interface's `get(i)` directly (only efficient for `ArrayList`, `O(n)` for `LinkedList` — cross-ref Part I).

### 20.7 `Iterable` as a method return type

Useful for exposing a custom read-only view without leaking the backing collection type:

```java
import java.util.*;

public class IterableReturnDemo {
    static Iterable<Integer> evensUpTo(int n) {
        List<Integer> evens = new ArrayList<>();
        for (int i = 0; i <= n; i += 2) evens.add(i);
        return evens; // ArrayList IS-A Iterable<Integer>
    }

    public static void main(String[] args) {
        for (int x : evensUpTo(10)) System.out.print(x + " "); // => 0 2 4 6 8 10
        System.out.println();
    }
}
```

---

## 21. Comparators and Streams (DSA subset)

### 21.1 `Comparator<T>` mental model

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T a, T b); // negative => a<b, zero => a==b, positive => a>b
}
```

Ascending sort places elements such that `compare` returns negative for "a should come before b".

### 21.2 THE SUBTRACTION OVERFLOW TRAP — the #1 comparator bug

```java
Comparator<Integer> bad = (a, b) -> a - b; // LOOKS fine, IS NOT
```

This overflows when `a` and `b` have opposite signs and large magnitude: e.g. `a = -2_000_000_000`, `b = 2_000_000_000` → `a - b` wraps around to a large **positive** `int`, incorrectly reporting `a > b` when actually `a < b`. Sorting algorithms assume a *consistent total order*; a comparator that occasionally lies produces subtly wrong sort results (not always a crash — often just wrong order, which is worse because it's silent).

```java
public class ComparatorOverflowDemo {
    public static void main(String[] args) {
        int a = -2_000_000_000, b = 2_000_000_000;
        System.out.println(a - b); // => 294967296 (WRONG SIGN — overflowed, should be very negative)
        System.out.println(Integer.compare(a, b)); // => -1 (CORRECT)
    }
}
```

> **Gotcha:** ALWAYS use `Integer.compare(a, b)` / `Long.compare(a, b)` / `Double.compare(a, b)` instead of `a - b` in comparators. This applies to lambdas, anonymous classes, and `compareTo` overrides alike. It's the single most common silent bug in Java CP code — costs nothing to avoid, costs hours to debug when it bites (usually only on large/adversarial test data, so it can pass small samples and fail on the judge).

### 21.3 Comparator builder methods

| Method | Example | What it does |
|---|---|---|
| `Comparator.comparingInt` | `comparingInt(x -> x.val)` | Builds comparator from an int-valued key extractor, overflow-safe internally |
| `Comparator.comparing` | `comparing(x -> x.name)` | Builds comparator from any `Comparable` key |
| `.thenComparing` | `cmp.thenComparing(x -> x.age)` | Tie-break — multi-key sort |
| `.reversed()` | `cmp.reversed()` | Flips the order |
| `Comparator.naturalOrder()` | — | Uses the type's own `compareTo` |
| `Comparator.reverseOrder()` | — | Descending natural order |
| `Comparator.nullsFirst` | `nullsFirst(cmp)` | Nulls sort before non-nulls (avoids `NullPointerException`) |

```java
import java.util.*;

public class ComparatorBuilderDemo {
    record Person(String name, int age) {}

    public static void main(String[] args) {
        List<Person> people = new ArrayList<>(List.of(
            new Person("Bob", 25), new Person("Alice", 30), new Person("Alice", 20)
        ));

        // single key
        people.sort(Comparator.comparingInt(Person::age));
        System.out.println(people);
        // => [Person[name=Alice, age=20], Person[name=Bob, age=25], Person[name=Alice, age=30]]

        // multi-key: name ascending, then age descending
        people.sort(Comparator.comparing(Person::name).thenComparing(Comparator.comparingInt(Person::age).reversed()));
        System.out.println(people);
        // => [Person[name=Alice, age=30], Person[name=Alice, age=20], Person[name=Bob, age=25]]

        // naturalOrder / reverseOrder on a simple list
        List<Integer> nums = new ArrayList<>(List.of(3, 1, 2));
        nums.sort(Comparator.naturalOrder());
        System.out.println(nums); // => [1, 2, 3]
        nums.sort(Comparator.reverseOrder());
        System.out.println(nums); // => [3, 2, 1]

        // nullsFirst
        List<String> withNulls = new ArrayList<>(Arrays.asList("b", null, "a"));
        withNulls.sort(Comparator.nullsFirst(Comparator.naturalOrder()));
        System.out.println(withNulls); // => [null, a, b]
    }
}
```

### 21.4 Comparator for `PriorityQueue` — min-heap-default reminder

`PriorityQueue<T>` is a **min-heap by default** (root = smallest element), the opposite of C++'s `std::priority_queue`, which is a **max-heap by default**.

```java
import java.util.*;

public class PQComparatorDemo {
    public static void main(String[] args) {
        PriorityQueue<Integer> minHeap = new PriorityQueue<>(); // default: min at top
        minHeap.addAll(List.of(5, 1, 3));
        System.out.println(minHeap.poll()); // => 1

        // max-heap: reverseOrder()
        PriorityQueue<Integer> maxHeap1 = new PriorityQueue<>(Collections.reverseOrder());
        maxHeap1.addAll(List.of(5, 1, 3));
        System.out.println(maxHeap1.poll()); // => 5

        // max-heap: explicit overflow-safe comparator
        PriorityQueue<Integer> maxHeap2 = new PriorityQueue<>((a, b) -> Integer.compare(b, a));
        maxHeap2.addAll(List.of(5, 1, 3));
        System.out.println(maxHeap2.poll()); // => 5
    }
}
```

> **vs C++:** `std::priority_queue<int>` defaults to max-heap; Java's `PriorityQueue<Integer>` defaults to min-heap. This trips up almost everyone porting C++ CP habits to Java — check the direction every time you declare one.

### 21.5 `Comparable.compareTo` vs external `Comparator`

- `Comparable<T>` (method `compareTo`) is implemented **inside** the class — defines the type's one "natural" order (like C++ `operator<`).
- `Comparator<T>` is **external** — lets you impose an order without modifying the class, and lets you have multiple different orderings for the same type.

```java
import java.util.*;

class Point implements Comparable<Point> {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
    @Override public int compareTo(Point o) { return Integer.compare(this.x, o.x); } // natural order: by x
    @Override public String toString() { return "(" + x + "," + y + ")"; }
}

public class ComparableVsComparatorDemo {
    public static void main(String[] args) {
        List<Point> pts = new ArrayList<>(List.of(new Point(3, 1), new Point(1, 2), new Point(2, 0)));

        pts.sort(null); // null Comparator => uses natural order (compareTo)
        System.out.println(pts); // => [(1,2), (2,0), (3,1)]

        // external comparator: sort by y instead, without touching the Point class
        pts.sort(Comparator.comparingInt(p -> p.y));
        System.out.println(pts); // => [(2,0), (3,1), (1,2)]
    }
}
```

### 21.6 Sort recipe table

| Goal | Recipe |
|---|---|
| Ascending, primitives | `Arrays.sort(intArr)` (mind §17.1 hack) |
| Ascending, objects | `Arrays.sort(objArr)` / `list.sort(null)` (needs `Comparable`) |
| Descending | `Arrays.sort(boxedArr, Collections.reverseOrder())` |
| By a key (ascending) | `list.sort(Comparator.comparingInt(x -> x.key))` |
| By a key (descending) | `list.sort(Comparator.comparingInt((T x) -> x.key).reversed())` |
| Multi-key (a then b) | `list.sort(Comparator.comparing(x->x.a).thenComparing(x->x.b))` |
| Custom numeric compare | ALWAYS `Integer.compare(x,y)`, never `x-y` |

### 21.7 Streams — DSA-useful subset

Streams trade a bit of speed for readability. Keep the core operations below in muscle memory.

```java
import java.util.*;
import java.util.stream.*;

public class StreamsDemo {
    public static void main(String[] args) {
        // generating a range
        int[] range = IntStream.range(0, 5).toArray();       // [0, 5) exclusive
        System.out.println(Arrays.toString(range)); // => [0, 1, 2, 3, 4]

        int[] rangeClosed = IntStream.rangeClosed(1, 5).toArray(); // [1, 5] inclusive
        System.out.println(Arrays.toString(rangeClosed)); // => [1, 2, 3, 4, 5]

        // wrapping an existing array
        int[] a = {3, 1, 4, 1, 5};
        System.out.println(Arrays.stream(a).sum());  // => 14
        System.out.println(Arrays.stream(a).max().getAsInt()); // => 5
        System.out.println(Arrays.stream(a).min().getAsInt()); // => 1

        // map / mapToObj / boxed
        int[] doubled = Arrays.stream(a).map(x -> x * 2).toArray();
        System.out.println(Arrays.toString(doubled)); // => [6, 2, 8, 2, 10]

        List<String> asStrings = Arrays.stream(a).mapToObj(x -> "n" + x).collect(Collectors.toList());
        System.out.println(asStrings); // => [n3, n1, n4, n1, n5]

        List<Integer> boxedList = Arrays.stream(a).boxed().collect(Collectors.toList());
        System.out.println(boxedList); // => [3, 1, 4, 1, 5]

        // filter
        int[] evens = Arrays.stream(a).filter(x -> x % 2 == 0).toArray();
        System.out.println(Arrays.toString(evens)); // => [4]

        // Collectors.groupingBy / counting
        Map<Integer, Long> freq = Arrays.stream(a).boxed()
                .collect(Collectors.groupingBy(x -> x, Collectors.counting()));
        System.out.println(freq); // => {1=2, 3=1, 4=1, 5=1}

        // List<Integer> -> sum via mapToInt
        List<Integer> list = List.of(1, 2, 3, 4);
        int total = list.stream().mapToInt(Integer::intValue).sum();
        System.out.println(total); // => 10

        // List<Integer> -> int[]
        int[] backToArr = list.stream().mapToInt(Integer::intValue).toArray();
        System.out.println(Arrays.toString(backToArr)); // => [1, 2, 3, 4]
    }
}
```

| Operation | Example |
|---|---|
| Range | `IntStream.range(0, n)` — `[0, n)` |
| Closed range | `IntStream.rangeClosed(1, n)` — `[1, n]` |
| Array → stream | `Arrays.stream(a)` |
| Reduce | `.sum()`, `.max()`, `.min()` (latter two return `OptionalInt`) |
| Transform (primitive) | `.map(x -> ...)` |
| Transform to object | `.mapToObj(x -> ...)` |
| Box | `.boxed()` — `IntStream` → `Stream<Integer>` |
| Filter | `.filter(x -> ...)` |
| To array | `.toArray()` |
| To list | `.collect(Collectors.toList())` |
| Group | `Collectors.groupingBy(keyFn)` / with downstream `Collectors.counting()` |
| `List<Integer>` → sum | `list.stream().mapToInt(Integer::intValue).sum()` |
| `List<Integer>` → `int[]` | `list.stream().mapToInt(Integer::intValue).toArray()` |

> **Gotcha:** `.max()`/`.min()` on an `IntStream` return `OptionalInt`, not `int` — must call `.getAsInt()` (throws `NoSuchElementException` on an empty stream, so guard with `.orElse(default)` if the stream might be empty).

> **Performance note:** streams allocate intermediate objects/lambdas and have per-element dispatch overhead. In hot CP loops (tight `O(n)`/`O(n log n)` inner loops running millions of times, e.g. inside a binary search or DP transition), prefer plain `for` loops over primitive arrays. Reach for streams when writing setup/one-off code (parsing input into a structure, building frequency maps once) where clarity matters more than the constant-factor cost.

---

# PART V — BIT MANIPULATION

## 22. Bitwise Operators and Fundamentals

### 22.1 The operator set

| Op | Name | Works on | Semantics |
|---|---|---|---|
| `&` | AND | int, long, boolean | bitwise AND |
| `\|` | OR | int, long, boolean | bitwise OR |
| `^` | XOR | int, long, boolean | bitwise XOR |
| `~` | NOT | int, long | bitwise complement (unary) |
| `<<` | left shift | int, long | shift left, fill with 0 |
| `>>` | **arithmetic** right shift | int, long | shift right, fill with **sign bit** |
| `>>>` | **logical** right shift | int, long | shift right, fill with **0** — JAVA ONLY |

> **vs C++:** C++ has no `>>>`. In C++, right-shifting a signed negative value is implementation-defined (in practice usually arithmetic on mainstream compilers); right-shifting an `unsigned` type is always logical. Java sidesteps the ambiguity by having **two explicit operators**: `>>` always sign-extends, `>>>` always zero-fills, regardless of the value's sign. Since Java has **no unsigned integer types at all** (`byte`, `short`, `char`, `int`, `long` are all signed except `char`, which is a special unsigned-16-bit case — see §22.5), `>>>` is your *only* way to get "unsigned-style" right shift behavior. This is the single biggest bit-manipulation trap moving from C++ to Java.

### 22.2 Truth table

```java
int a = 0b1100; // 12
int b = 0b1010; // 10
System.out.println(Integer.toBinaryString(a & b));  // => 1000  (8)
System.out.println(Integer.toBinaryString(a | b));  // => 1110  (14)
System.out.println(Integer.toBinaryString(a ^ b));  // => 110   (6)
System.out.println(Integer.toBinaryString(~a));     // => 11111111111111111111111111110011 (-13, all bits flipped)
```

Bit-by-bit for `a=1100`, `b=1010`:

```
a : 1 1 0 0
b : 1 0 1 0
& : 1 0 0 0   (both must be 1)
| : 1 1 1 0   (either is 1)
^ : 0 1 1 0   (exactly one is 1)
```

### 22.3 `>>` vs `>>>` — worked examples

```java
int x = -8;                          // 32-bit two's complement:
                                      // 11111111111111111111111111111000
System.out.println(x >> 1);          // => -4   (sign-extends: fills with 1s)
System.out.println(x >>> 1);         // => 2147483644  (zero-fills: fills with 0s)

System.out.println(Integer.toBinaryString(x >> 1));
// => 11111111111111111111111111111100   (top bit still 1 → stays negative)
System.out.println(Integer.toBinaryString(x >>> 1));
// => 1111111111111111111111111111100    (top bit is 0 → huge positive)
```

```
x        = 1111 1111 1111 1111 1111 1111 1111 1000   (-8)
x >> 1   = 1111 1111 1111 1111 1111 1111 1111 1100   (-4)   fill bit = old sign bit (1)
x >>> 1  = 0111 1111 1111 1111 1111 1111 1111 1100   (2147483644)  fill bit = 0
```

For **non-negative** numbers `>>` and `>>>` are identical (there's no sign bit to propagate):

```java
int y = 20;
System.out.println(y >> 2);   // => 5
System.out.println(y >>> 2);  // => 5   (same result — sign bit is 0)
```

> **Gotcha:** the classic "unsigned average without overflow" trick `(lo + hi) >>> 1` matters even for **positive** `lo, hi` in Java because `lo + hi` can overflow into a *negative* `int` — and `>>` on that overflowed negative would sign-extend and give the wrong (negative or wrong-magnitude) midpoint. `>>>` treats the overflowed bit pattern as unsigned, giving the arithmetically correct midpoint. This is idiomatic in binary search:
```java
int mid = low + ((high - low) >>> 1); // safest: avoids overflow entirely
// or, if you must add first:
int mid2 = (low + high) >>> 1;        // correct even if low+high overflows int
```

### 22.4 Operator precedence trap

`&`, `^`, `|` bind **looser** than relational/equality operators (`==`, `!=`, `<`, `<=`, `>`, `>=`). This is identical to C++ and trips up everyone at some point.

```java
int x = 5;
// WRONG — parses as x & (1 == 0), a type error in Java (won't even compile,
// since & on boolean vs int mismatches) — but conceptually the trap is real:
// if ((x & 1) == 0) is what you want; if (x & 1 == 0) is broken.
if ((x & 1) == 0) {            // correct: check even/odd
    System.out.println("even");
} else {
    System.out.println("odd"); // => odd
}
```

> **Gotcha:** In C++, `x & 1 == 0` silently compiles (evaluates `1 == 0` first, then `x & 0`) and gives a *wrong but non-crashing* result. In Java, mixing `int` and `boolean` with `&`/`==` like that is usually a **compile error**, which is actually a small safety net — but the fix is the same either language: **always parenthesize** `(x & mask) == val`, `(x | mask) != 0`, etc.

Full precedence order (high → low) relevant to bit code:
```
~  (unary)          highest
<<  >>  >>>
<  <=  >  >=
==  !=
&
^
|
&&
||
?:
=
```

### 22.5 Two's complement and `~`

Java integers are two's complement, same as C++. `~x == -x - 1`, equivalently `-x == ~x + 1`.

```java
int x = 5;
System.out.println(~x);        // => -6
System.out.println(-x);        // => -5
System.out.println(~x + 1);    // => -5   (== -x)
System.out.println(~(-x) + 1); // => 5    (== x, involution check)
```

```
 x        = 0000 0000 0000 0000 0000 0000 0000 0101   (5)
~x        = 1111 1111 1111 1111 1111 1111 1111 1010   (-6)
~x + 1    = 1111 1111 1111 1111 1111 1111 1111 1011   (-5)  == -x
```

### 22.6 Shift-count masking — the #1 Java bitmask bug

Java **masks the shift distance** instead of treating an out-of-range shift as undefined behavior. For `int` operands the shift count is taken **mod 32** (only the low 5 bits are used); for `long` operands it's **mod 64** (low 6 bits).

```java
System.out.println(1 << 32);   // => 1     !!  (32 & 31 == 0, so this is really 1 << 0)
System.out.println(1 << 33);   // => 2     !!  (33 & 31 == 1, so this is really 1 << 1)
System.out.println(1L << 64);  // => 1     !!  (64 & 63 == 0)
System.out.println(1L << 65);  // => 2     !!  (65 & 63 == 1)
```

> **vs C++:** In C++, shifting by an amount ≥ the operand's bit-width is **undefined behavior** — it might crash, might produce garbage, might "work" depending on the platform/compiler, and UBSan will flag it. Java instead gives you a **well-defined but almost certainly not what you meant** result via silent masking. This is arguably worse for buggy code because it never crashes and never gets flagged — it just silently computes the wrong bitmask, often in a loop over bit positions where you assumed `1 << i` for `i` up to 31/63 was safe up to and including the boundary.

```java
// classic bug: building a mask for bit 31 using int
int mask = 1 << 31;
System.out.println(mask);                      // => -2147483648  (Integer.MIN_VALUE — NOT a bug, this is correct!)
System.out.println(Integer.toBinaryString(mask)); // => 10000000000000000000000000000000... wait, 32 bits:
                                                 // 10000000000000000000000000000000 (trimmed to 32) = 1 followed by 31 zeros
```

```
1 << 31 = 1000 0000 0000 0000 0000 0000 0000 0000   = Integer.MIN_VALUE (negative!)
```

`1 << 31` is *correctly* computed (no masking needed, 31 < 32) but the **result is negative** because bit 31 is the sign bit for `int`. If you then compare it against a positive expected value, or print it expecting a huge positive number, you'll be confused. If you need bit 31 to behave as an unsigned magnitude, or you need bit ≥ 32 at all, switch to `long`:

```java
long maskBit31 = 1L << 31;
System.out.println(maskBit31);                 // => 2147483648   (positive, as a long)

long maskBit60 = 1L << 60;                      // fine — long has 64 bits
System.out.println(Long.toBinaryString(maskBit60));
// => 1000000000000000000000000000000000000000000000000000000000000  (61 chars, bit 60 set)
```

> **Gotcha — the rule to memorize:** for any bitmask problem where the bit index `k` can reach **31 or higher**, use `1L << k` (a `long` literal, capital `L`) not `1 << k`. Writing `1 << k` for `k == 31` gives a negative `int` (usually *not* a bug by itself, but dangerous once you add/compare/print it as if positive); writing `1 << k` for `k >= 32` silently wraps due to shift-count masking and is *always* a bug.

### 22.7 No unsigned types — consequences

Java has exactly one unsigned-flavored primitive: `char` (16-bit, `0..65535`, no sign bit). Everything else (`byte`, `short`, `int`, `long`) is signed two's complement. There is no `unsigned int`, no `uint32_t`, nothing.

Practical fallout:

```java
byte b = (byte) 0xFF;              // -1, not 255 — byte is signed 8-bit
System.out.println(b);             // => -1
System.out.println(b & 0xFF);      // => 255   (widen to int, mask off sign-extended 1s)

short s = (short) 0xFFFF;          // -1, not 65535 — short is signed 16-bit
System.out.println(s);             // => -1
System.out.println(s & 0xFFFF);    // => 65535

char c = (char) 0xFFFF;            // char IS unsigned — no sign flip
System.out.println((int) c);       // => 65535
```

> **Gotcha — sign extension on widening:** converting a `byte`/`short` to a wider type (`int`, `long`) **sign-extends** — the sign bit is replicated into the new high bits. This bites hardest when reading raw bytes (e.g. from an `InputStream`, or unpacking a byte array into an int):
```java
byte raw = (byte) 200;             // 200 doesn't fit in signed byte (-128..127) → wraps to -56
System.out.println(raw);           // => -56
int widened = raw;                 // sign-extends: -56, not 200
System.out.println(widened);       // => -56
int fixed = raw & 0xFF;            // mask to recover the "unsigned byte value"
System.out.println(fixed);         // => 200
```
> **vs C++:** in C++ you'd just declare `unsigned char`/`uint8_t` and the problem doesn't exist. In Java the idiom is always **`& 0xFF`** (byte→unsigned int), **`& 0xFFFF`** (short→unsigned int), or use the `Integer.toUnsignedLong` / `Byte.toUnsignedInt` / `Short.toUnsignedInt` helper methods (Java 8+) which do the masking for you.

```java
byte raw = (byte) 200;
System.out.println(Byte.toUnsignedInt(raw));   // => 200
```

### 22.8 Quick reference — precedence & pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Missing parens around `&`/`\|`/`^` with `==`/`<` | wrong logic or compile error | `(x & mask) == val` |
| `1 << k` for `k >= 31` | negative or wrapped-around value | `1L << k` |
| Right-shifting negative with `>>` when you wanted unsigned semantics | wrong (still-negative) result | use `>>>` |
| Treating `byte`/`short` as unsigned after widening | sign-extended garbage | `& 0xFF`, `& 0xFFFF`, or `Byte/Short.toUnsignedInt` |
| Assuming `char` is signed like `byte` | surprise: it isn't | `char` is unsigned 16-bit by design |

---

## 23. The Complete Bit-Trick Catalog

Throughout, `i` is a 0-indexed bit position (bit 0 = least significant bit).

### 23.1 Test / set / clear / toggle a single bit

```java
int x = 0b0000; // start
int i = 2;

// SET bit i
x = x | (1 << i);
System.out.println(Integer.toBinaryString(x)); // => 100

// TEST bit i (is it set?)
boolean isSet = (x & (1 << i)) != 0;
System.out.println(isSet); // => true

// CLEAR bit i
x = x & ~(1 << i);
System.out.println(Integer.toBinaryString(x)); // => 0

// TOGGLE bit i
x = x ^ (1 << i);
System.out.println(Integer.toBinaryString(x)); // => 100

// EXTRACT bit i as 0/1 (not just boolean)
int bitVal = (x >> i) & 1;
System.out.println(bitVal); // => 1
```

> **Gotcha:** for bit positions ≥ 31, every `1 << i` here must become `1L << i` and `x` must be `long`. See §22.6.

### 23.2 Odd/even and shift-based multiply/divide

```java
int n = 7;
System.out.println((n & 1) == 1 ? "odd" : "even"); // => odd
System.out.println((n & 1) == 0 ? "odd" : "even"); // => odd  (WRONG intentionally-shown mistake — don't flip the check!)
```

```java
int n2 = -7;
System.out.println(n2 & 1); // => 1   (works correctly even for negatives — two's complement LSB is still the parity bit)
```

```java
int n3 = 20;
System.out.println(n3 >> 1);   // => 10   (divide by 2, floor toward -infinity for negatives)
System.out.println(n3 << 1);   // => 40   (multiply by 2)
System.out.println(n3 << 3);   // => 160  (multiply by 8, i.e. 2^3)

int neg = -20;
System.out.println(neg >> 1);  // => -10  (arithmetic shift: rounds toward -infinity, i.e. floor division)
System.out.println(neg / 2);   // => -10  (int / for -20/2 happens to match here; NOT always true for odd negatives)

int negOdd = -7;
System.out.println(negOdd >> 1);  // => -4   (floor(-7/2) = -4)
System.out.println(negOdd / 2);   // => -3   (Java integer division truncates toward zero)
```

> **Gotcha:** `x >> 1` is **not** always the same as `x / 2` for negative odd `x` — `>>` floors, `/` truncates toward zero. `-7 >> 1 == -4` but `-7 / 2 == -3`. Same trap exists in C++.

> **vs C++:** to get *unsigned-style* "divide by 2" semantics on a bit pattern regardless of sign (rare, but shows up when treating an `int` as a raw 32-bit unsigned quantity), use `>>>` instead of `>>`. There is no such distinction available via `/`.

### 23.3 Clear lowest set bit — `x & (x - 1)`

```java
int x = 0b10110100; // 180
int cleared = x & (x - 1);
System.out.println(Integer.toBinaryString(cleared)); // => 10110000
```

```
x       = 1011 0100
x-1     = 1011 0011
x&(x-1) = 1011 0000     <- lowest set bit (the trailing "100" -> "0") is cleared
```

**Brian Kernighan popcount** (loop runs once per set bit, not per bit-width — faster than a 32/64-iteration loop when bits are sparse):

```java
int popcount(int x) {
    int count = 0;
    while (x != 0) {
        x &= (x - 1);   // clear lowest set bit
        count++;
    }
    return count;
}
System.out.println(popcount(0b10110100)); // => 4
```

> **Gotcha:** in production DSA code just call `Integer.bitCount(x)` (§24) — it's an intrinsic, faster than any hand-rolled loop. Kernighan's trick is worth knowing for interviews / when `bitCount` isn't available (e.g. some competitive judges historically, or when you need the *positions* of bits one at a time anyway).

### 23.4 Isolate lowest set bit — `x & (-x)`

```java
int x = 0b10110100; // 180
int lowestBit = x & (-x);
System.out.println(Integer.toBinaryString(lowestBit)); // => 100
```

```
x   = 1011 0100
-x  = 0100 1100    (two's complement negation: ~x + 1)
x&-x= 0000 0100    <- isolates the single lowest set bit
```

This is the standard building block for Fenwick trees / BIT (Binary Indexed Tree) traversal (`i += i & (-i)`, `i -= i & (-i)`).

**Set lowest zero bit** — `x | (x + 1)`:

```java
int y = 0b10110011; // has lowest zero bit at position 2
int setLowestZero = y | (y + 1);
System.out.println(Integer.toBinaryString(setLowestZero)); // => 10110111
```

```
y     = 1011 0011
y+1   = 1011 0100
y|y+1 = 1011 0111    <- the lowest 0 (bit 2) is now 1
```

### 23.5 Power-of-two check

```java
boolean isPowerOfTwo(int x) {
    return x > 0 && (x & (x - 1)) == 0;
}
System.out.println(isPowerOfTwo(64));  // => true
System.out.println(isPowerOfTwo(63));  // => false
System.out.println(isPowerOfTwo(0));   // => false  (0 must be excluded explicitly: 0 & -1 == 0 would wrongly pass)
```

> **Gotcha:** the `x > 0` guard is mandatory — without it, `x == 0` passes the `(x & (x-1)) == 0` test (since `0 & -1 == 0`), and negative `x` can also pass spuriously depending on bit pattern. Same trap in C++.

### 23.6 XOR swap and XOR properties

**XOR swap** (classic, but *avoid it in real code*):

```java
int a = 5, b = 9;
a ^= b;
b ^= a;
a ^= b;
System.out.println(a + " " + b); // => 9 5
```

> **Gotcha:** XOR swap **fails** if `a` and `b` are the same variable / same array index (`arr[i] ^= arr[i]; ...` zeroes it out instead of "swapping with itself"). It's also not faster than a temp-variable swap on modern hardware/JIT and is much harder to read. Prefer:
```java
int tmp = a; a = b; b = tmp;
```
> **vs C++:** identical trap and identical advice in C++ — mentioned here because interviewers still ask for it as a "trick", not because you should use it.

**XOR core properties** — `x ^ x == 0`, `x ^ 0 == x`, XOR is commutative and associative. These make it the engine behind several classic problems:

**Single Number** (every element appears twice except one):

```java
int singleNumber(int[] nums) {
    int result = 0;
    for (int n : nums) result ^= n;   // pairs cancel to 0, survivor remains
    return result;
}
System.out.println(singleNumber(new int[]{4, 1, 2, 1, 2})); // => 4
```

**Missing Number** (array of `0..n` with one missing):

```java
int missingNumber(int[] nums) {
    int result = nums.length; // pre-seed with n
    for (int i = 0; i < nums.length; i++) {
        result ^= i ^ nums[i];
    }
    return result;
}
System.out.println(missingNumber(new int[]{3, 0, 1})); // => 2
```

**Two Single Numbers** (every element appears twice except exactly two, which differ):

```java
int[] singleNumberTwo(int[] nums) {
    int xorAll = 0;
    for (int n : nums) xorAll ^= n;              // xorAll = a ^ b  (a != b)
    int diffBit = xorAll & (-xorAll);            // any bit where a and b differ (lowest such bit)
    int groupA = 0, groupB = 0;
    for (int n : nums) {
        if ((n & diffBit) != 0) groupA ^= n;
        else groupB ^= n;
    }
    return new int[]{groupA, groupB};
}
int[] r = singleNumberTwo(new int[]{1, 2, 1, 3, 2, 5});
System.out.println(r[0] + " " + r[1]); // => 3 5  (order may vary)
```

**XOR of range `[0..n]`** — closed form using `n % 4` (avoids an O(n) loop):

```java
int xorUpTo(int n) {
    // xor(0..n) cycles with period 4:
    switch (n % 4) {
        case 0: return n;
        case 1: return 1;
        case 2: return n + 1;
        default: return 0; // n % 4 == 3
    }
}
System.out.println(xorUpTo(5)); // => xor(0^1^2^3^4^5) = 1
```

```java
int xorRange(int l, int r) {           // xor of [l..r] inclusive
    return xorUpTo(r) ^ xorUpTo(l - 1);
}
System.out.println(xorRange(3, 9)); // => xor(3^4^5^6^7^8^9)
```

### 23.7 Masks and bit-field extraction

```java
int k = 5;
int mask = (1 << k) - 1;                       // low k bits all set
System.out.println(Integer.toBinaryString(mask)); // => 11111

int x = 0b110110111;
int lowFive = x & mask;                        // extract low k bits
System.out.println(Integer.toBinaryString(lowFive)); // => 10111
```

**Extract a bit-field** `[hi:lo]` (inclusive, hi >= lo):

```java
int extractField(int x, int hi, int lo) {
    int width = hi - lo + 1;
    int fieldMask = ((1 << width) - 1) << lo;
    return (x & fieldMask) >>> lo;
}
int val = 0b1101_1010;
System.out.println(Integer.toBinaryString(extractField(val, 6, 3))); // bits 6..3 => 1011
```

**Turn off all bits below `i` (keep bit i and above)**:

```java
int turnOffBelow(int x, int i) {
    return x & (~0 << i);
}
```

**Turn off all bits above `i` (keep bit i and below, i.e. keep low `i+1` bits)**:

```java
int turnOffAbove(int x, int i) {
    return x & ((1 << (i + 1)) - 1);
}
```

> **Gotcha:** `~0` is `-1`, i.e. all 1-bits (`11111111111111111111111111111111`... 32 of them for int). `~0 << i` is a common idiom for "mask with the top `32-i` bits set". Combined with the shift-masking rule (§22.6), this is safe for `i` in `0..31` for `int`.

### 23.8 Highest set bit / floor(log2) / power-of-two rounding

```java
int x = 200;
int floorLog2 = 31 - Integer.numberOfLeadingZeros(x);
System.out.println(floorLog2); // => 7   (2^7 = 128 <= 200 < 256 = 2^8)

long xl = 200L;
int floorLog2Long = 63 - Long.numberOfLeadingZeros(xl);
System.out.println(floorLog2Long); // => 7
```

**Highest / lowest one-bit as a mask** (built-ins, see §24 for full table):

```java
System.out.println(Integer.highestOneBit(200)); // => 128
System.out.println(Integer.lowestOneBit(200));  // => 8
```

**Next power of two ≥ x** (bit-smear trick — classic, works even without built-ins):

```java
int nextPowerOfTwo(int x) {
    if (x <= 1) return 1;
    x--;
    x |= x >> 1;
    x |= x >> 2;
    x |= x >> 4;
    x |= x >> 8;
    x |= x >> 16;
    return x + 1;
}
System.out.println(nextPowerOfTwo(200)); // => 256
System.out.println(nextPowerOfTwo(256)); // => 256  (already a power of two)
```

**Next power of two using built-ins** (shorter, relies on `highestOneBit`):

```java
int nextPowerOfTwoBuiltin(int x) {
    if (x <= 1) return 1;
    int h = Integer.highestOneBit(x - 1);
    return h << 1;
}
System.out.println(nextPowerOfTwoBuiltin(200)); // => 256
```

**Previous power of two ≤ x**:

```java
int prevPowerOfTwo(int x) {
    return Integer.highestOneBit(x); // 0 if x == 0
}
System.out.println(prevPowerOfTwo(200)); // => 128
```

### 23.9 Reverse bits / bytes, bit-count (preview — full table in §24)

```java
System.out.println(Integer.toBinaryString(Integer.reverse(0b1))); // => 10000000000000000000000000000000 -> trimmed leading zeros removed, actually prints:
// (Integer.reverse(1) sets only bit 31) => 1 followed by 31 zeros when printed as toBinaryString it shows "10000000000000000000000000000000"
// but toBinaryString never shows more than 32 chars for a valid 32-bit value:
System.out.println(Integer.reverse(1));            // => -2147483648  (bit 31 set => Integer.MIN_VALUE)

System.out.println(Integer.reverseBytes(0x12345678)); // => 0x78563412 in decimal: => 2018915346
System.out.println(Integer.bitCount(0b10110100));      // => 4
```

### 23.10 Subset enumeration over a bitmask

**Enumerate all non-empty submasks of `m`** in `O(popcount(m) choose ...)` amortized — total work across ALL masks `m` from `0..(1<<n)-1` is `O(3^n)` (each bit is either 0 in m, or 1-and-1-in-submask, or 1-and-0-in-submask — 3 states):

```java
int m = 0b10110; // enumerate all non-empty submasks of m
for (int s = m; s > 0; s = (s - 1) & m) {
    System.out.println(Integer.toBinaryString(s));
}
// => 10110
// => 10100
// => 10010
// => 10000
// => 00110
// => 00100
// => 00010
```

To **include the empty submask (0)**, restructure as a do-while (a plain `for` loop as above stops at `s > 0` and never visits 0):

```java
int m2 = 0b101;
int s = m2;
while (true) {
    System.out.println(Integer.toBinaryString(s)); // visits m2, then submasks..., then 0
    if (s == 0) break;
    s = (s - 1) & m2;
}
// => 101
// => 100
// => 001
// => 000
```

> **Gotcha:** the naive `for (int s = m; s >= 0; s = (s-1) & m)` is an **infinite loop** once `s` hits 0, because `(0 - 1) & m == -1 & m == m` (wraps back to `m`). Always special-case zero, either with `s > 0` (skips empty subset) or an explicit `break` after processing `s == 0` as shown above.

**Iterate all `2^n` masks** for `n` items (e.g. bitmask DP over subsets):

```java
int n = 3;
for (int mask = 0; mask < (1 << n); mask++) {
    System.out.println(Integer.toBinaryString(mask));
}
// => 0, 1, 10, 11, 100, 101, 110, 111
```

> **Gotcha:** for `n >= 31` this overflows `int` range/time budget entirely — bitmask-over-all-subsets DP is only tractable up to roughly `n <= 20` in practice (2^20 ≈ 1M), regardless of int vs long, because the *time* complexity (not just the mask's storage) is exponential.

**Iterate set bits of a mask** — two idiomatic forms:

```java
int mask = 0b10110100;

// Form 1: peel off lowest set bit each time
int m3 = mask;
while (m3 != 0) {
    int lowest = m3 & (-m3);              // isolate lowest set bit
    int bitIndex = Integer.numberOfTrailingZeros(lowest);
    System.out.println(bitIndex);         // => 2, 4, 5, 7  (in increasing order)
    m3 -= lowest;                          // equivalently: m3 &= (m3 - 1)
}
```

```java
// Form 2: numberOfTrailingZeros + clear lowest bit directly
int m4 = mask;
while (m4 != 0) {
    int bitIndex = Integer.numberOfTrailingZeros(m4);
    System.out.println(bitIndex);         // => 2, 4, 5, 7
    m4 &= (m4 - 1);                       // clear lowest set bit
}
```

Both forms are `O(popcount(mask))` — proportional to the number of set bits, not the bit width.

### 23.11 Gray code

Binary-to-Gray: `gray = n ^ (n >> 1)`. Gray code sequences differ by exactly one bit between consecutive values (used in Gray Code / bitmask-traversal problems).

```java
int toGray(int n) { return n ^ (n >> 1); }

for (int i = 0; i < 8; i++) {
    System.out.println(Integer.toBinaryString(toGray(i)));
}
// => 0, 1, 11, 10, 110, 111, 101, 100   (each differs from previous by exactly 1 bit)
```

Gray-to-binary (inverse, needs a loop since it's not a single XOR):

```java
int fromGray(int g) {
    int n = 0;
    for (; g != 0; g >>= 1) n ^= g;
    return n;
}
System.out.println(fromGray(0b111)); // => 5
```

### 23.12 Counting bits for `0..n` via DP

Classic "Counting Bits" LeetCode pattern — `dp[i] = dp[i >> 1] + (i & 1)`, i.e. the popcount of `i` equals the popcount of `i` with its last bit dropped, plus that dropped bit.

```java
int[] countBits(int n) {
    int[] dp = new int[n + 1];
    for (int i = 1; i <= n; i++) {
        dp[i] = dp[i >> 1] + (i & 1);
    }
    return dp;
}
System.out.println(Arrays.toString(countBits(7)));
// => [0, 1, 1, 2, 1, 2, 2, 3]
```

Runs in `O(n)` total vs `O(n log n)` from calling `Integer.bitCount` in a loop (though in practice `bitCount` is a JIT intrinsic and often faster in wall-clock for modest `n` — know the DP for the "derive it yourself" interview ask).

### 23.13 `long` as a 64-bit mask — the `1L <<` discipline (restated, critical)

Whenever a bitmask problem's universe size `n` can reach 31 or more items, you **must** back the mask with `long`, and every literal shift **must use `1L`**, not `1`.

```java
int n = 40; // more than 31 items -> must use long
long fullMask = (1L << n) - 1;                 // low n bits set
System.out.println(Long.bitCount(fullMask));   // => 40

long mask = 0L;
mask |= (1L << 35);          // set bit 35 — MUST be 1L, not 1 (1 << 35 would silently wrap: 35 & 31 = 3 -> 1<<3 = 8, WRONG bit!)
System.out.println(mask == (1L << 35));        // => true
System.out.println((1 << 35));                  // => 8  <-- the bug in action: int shift wrapped, wrong value entirely
```

> **Gotcha:** `1 << 35` does **not** throw or truncate to 0 — it silently computes `1 << (35 & 31) == 1 << 3 == 8`, a completely different, plausible-looking wrong answer that will pass code review and fail only on specific test cases. This is the single most common bug reported when porting a working C++ bitmask-DP solution (where `1 << 35` on a 32-bit `int` would at least be a compiler warning or UB flagged by sanitizers) to Java.

---

## 24. Integer/Long Static Methods and Bit Containers

### 24.1 Complete method reference table

All methods below exist on both `Integer` and `Long` with the obvious `int`↔`long` signature change, unless noted. Complexity is O(1) — these compile to JIT intrinsics (often a single hardware instruction, e.g. `POPCNT`, `LZCNT`, `TZCNT`) on modern JVMs.

| Method | Signature (`Integer`) | Meaning | `Long` variant |
|---|---|---|---|
| `bitCount` | `static int bitCount(int i)` | popcount — number of 1-bits | `Long.bitCount(long i)` |
| `numberOfLeadingZeros` | `static int numberOfLeadingZeros(int i)` | count of 0-bits before the highest 1-bit (32 if `i==0`) | `Long.numberOfLeadingZeros(long i)` (64 if 0) |
| `numberOfTrailingZeros` | `static int numberOfTrailingZeros(int i)` | count of 0-bits after the lowest 1-bit (32 if `i==0`) | `Long.numberOfTrailingZeros(long i)` (64 if 0) |
| `highestOneBit` | `static int highestOneBit(int i)` | isolates the highest set bit as a power-of-two value (0 if `i==0`) | `Long.highestOneBit(long i)` |
| `lowestOneBit` | `static int lowestOneBit(int i)` | isolates lowest set bit, i.e. `i & -i` (0 if `i==0`) | `Long.lowestOneBit(long i)` |
| `reverse` | `static int reverse(int i)` | reverses bit order (bit 0 ↔ bit 31) | `Long.reverse(long i)` (bit 0 ↔ bit 63) |
| `reverseBytes` | `static int reverseBytes(int i)` | reverses byte order (4 bytes) | `Long.reverseBytes(long i)` (8 bytes) |
| `rotateLeft` | `static int rotateLeft(int i, int distance)` | circular left shift | `Long.rotateLeft(long i, int distance)` |
| `rotateRight` | `static int rotateRight(int i, int distance)` | circular right shift | `Long.rotateRight(long i, int distance)` |
| `toBinaryString` | `static String toBinaryString(int i)` | base-2 string, **unsigned** representation, no leading zeros/sign | `Long.toBinaryString(long i)` |
| `toHexString` | `static String toHexString(int i)` | base-16 string, unsigned representation | `Long.toHexString(long i)` |
| `toOctalString` | `static String toOctalString(int i)` | base-8 string, unsigned representation | `Long.toOctalString(long i)` |
| `parseInt(s, radix)` | `static int parseInt(String s, int radix)` | parse string in given base, e.g. `parseInt(s, 2)` | `Long.parseLong(String s, int radix)` |
| `compareUnsigned` | `static int compareUnsigned(int a, int b)` | compares as if unsigned | `Long.compareUnsigned(long a, long b)` |
| `divideUnsigned` | `static int divideUnsigned(int a, int b)` | division treating operands as unsigned | `Long.divideUnsigned(long a, long b)` |
| `remainderUnsigned` | `static int remainderUnsigned(int a, int b)` | remainder treating operands as unsigned | `Long.remainderUnsigned(long a, long b)` |
| `toUnsignedLong` | `static long toUnsignedLong(int i)` | widen int to long as if unsigned (no sign extension) | *(N/A — long is already the widest)* |
| `toUnsignedString` | `static String toUnsignedString(int i)` | decimal string treating value as unsigned | `Long.toUnsignedString(long i)` |

### 24.2 Worked examples for each

```java
System.out.println(Integer.bitCount(0b10110100));            // => 4
System.out.println(Long.bitCount(-1L));                      // => 64  (all bits set)

System.out.println(Integer.numberOfLeadingZeros(1));          // => 31  (1 = 0...0001, 31 leading zeros)
System.out.println(Integer.numberOfLeadingZeros(0));          // => 32  (well-defined! see §24.3)
System.out.println(Integer.numberOfLeadingZeros(-1));         // => 0   (all bits set, no leading zeros)

System.out.println(Integer.numberOfTrailingZeros(8));         // => 3   (1000 -> 3 trailing zeros)
System.out.println(Integer.numberOfTrailingZeros(0));         // => 32  (well-defined!)

System.out.println(Integer.highestOneBit(200));               // => 128
System.out.println(Integer.highestOneBit(0));                 // => 0
System.out.println(Integer.lowestOneBit(200));                // => 8
System.out.println(Integer.lowestOneBit(0));                  // => 0

System.out.println(Integer.reverse(1));                       // => -2147483648  (bit 0 -> bit 31)
System.out.println(Integer.reverseBytes(0x12345678));         // => 2018915346  (== 0x78563412)

System.out.println(Integer.rotateLeft(0b1, 1));                // => 2   (0...01 -> 0...10)
System.out.println(Integer.rotateLeft(1 << 31, 1));             // => 1   (wraps around: top bit rotates to bottom)
System.out.println(Integer.rotateRight(1, 1));                  // => -2147483648  (bottom bit rotates to top)

System.out.println(Integer.toBinaryString(-1));                // => 11111111111111111111111111111111 -> actually 32 ones:
                                                                 // "11111111111111111111111111111111" (32 chars)
System.out.println(Integer.toHexString(-1));                   // => ffffffff
System.out.println(Integer.toOctalString(8));                  // => 10

System.out.println(Integer.parseInt("1011", 2));               // => 11
System.out.println(Long.parseLong("ff", 16));                  // => 255

System.out.println(Integer.compareUnsigned(-1, 1));            // => 1  (-1 as unsigned bit pattern is 4294967295 > 1)
System.out.println(Integer.compare(-1, 1));                    // => -1 (SIGNED compare: -1 < 1) -- contrast!

System.out.println(Integer.divideUnsigned(-1, 2));             // => 2147483647  (treats -1 as 4294967295u)
System.out.println(-1 / 2);                                    // => 0  (signed division, contrast!)

System.out.println(Integer.toUnsignedLong(-1));                // => 4294967295  (widened WITHOUT sign extension)
System.out.println((long) -1);                                 // => -1  (plain cast DOES sign-extend, contrast!)

System.out.println(Integer.toUnsignedString(-1));              // => 4294967295
```

> **Gotcha:** `Integer.toBinaryString`/`toHexString`/`toOctalString` always print the **unsigned bit-pattern representation** even though `int` itself is signed — `toBinaryString(-1)` gives 32 ones, not a minus sign. This is different from `String.valueOf(-1)` or `Integer.toString(-1)`, which prints `"-1"` in decimal (signed). Don't be surprised that `toBinaryString`/`toHexString`/`toOctalString` never emit a `-` sign — that's by design, since they're meant to show the raw bit layout.

> **vs C++:** none of `compareUnsigned`, `divideUnsigned`, `remainderUnsigned`, `toUnsignedLong`, `toUnsignedString` have a C++ equivalent as *methods* — in C++ you'd just reinterpret/cast to an `unsigned` type and use the normal operators, because C++ has real unsigned types. Java fakes it with these static helper methods operating on signed storage.

### 24.3 Key safety notes — zero-input behavior

| Call | C++ equivalent | C++ behavior on 0 | Java behavior on 0 |
|---|---|---|---|
| `Integer.numberOfTrailingZeros(0)` | `__builtin_ctz(0)` | **UB** — may crash or return garbage | **32** (well-defined) |
| `Long.numberOfTrailingZeros(0L)` | `__builtin_ctzll(0)` | **UB** | **64** (well-defined) |
| `Integer.numberOfLeadingZeros(0)` | `__builtin_clz(0)` | **UB** | **32** (well-defined) |
| `Long.numberOfLeadingZeros(0L)` | `__builtin_clzll(0)` | **UB** | **64** (well-defined) |
| `Integer.highestOneBit(0)` | *(no direct builtin)* | n/a | **0** |
| `Integer.lowestOneBit(0)` | `x & -x` with `x=0` | 0 (well-defined in C++ too, since `&`/unary `-` on 0 is fine) | **0** |
| `Integer.bitCount(0)` | `__builtin_popcount(0)` | 0 (well-defined) | **0** |

> **Gotcha:** Java's well-defined "32 for zero" behavior on `numberOfTrailingZeros`/`numberOfLeadingZeros` is *safer* than C++'s UB, but it is **still a value you must explicitly handle** in loops like "iterate set bits" (§23.10) — if you forget the `while (mask != 0)` guard and call `numberOfTrailingZeros(0)` expecting some sentinel like `-1`, you'll get `32`, which if used as an array index or shift amount will silently misbehave (e.g. `1 << 32 == 1` per §22.6) rather than crash — making the bug harder to spot than a C++ crash would be.

### 24.4 int/long as a subset bitmask — the DSA workhorse

Represent a subset of a universe of `n` items as the bits of an integer: bit `i` set ⟺ item `i` is in the subset. Use `int` for `n <= 30` (bit 31 is usable but awkward since it's the sign bit — many people cap at `n <= 30` defensively), `long` for `31 <= n <= 62` (again capping a couple bits below 64 defensively; up to 63 is technically fine).

```java
int n = 6; // items 0..5
int subset = 0;

// ADD item i to subset
subset |= (1 << 3);
System.out.println(Integer.toBinaryString(subset)); // => 1000

// REMOVE item i from subset
subset &= ~(1 << 3);
System.out.println(Integer.toBinaryString(subset)); // => 0

// TEST membership of item i
subset |= (1 << 2) | (1 << 4);
boolean has2 = (subset & (1 << 2)) != 0;
System.out.println(has2); // => true

// ITERATE all items currently in the subset
int s = subset;
while (s != 0) {
    int i = Integer.numberOfTrailingZeros(s);
    System.out.println(i);       // => 2, then 4
    s &= (s - 1);
}

// FULL universe mask for n items
int full = (1 << n) - 1;
System.out.println(Integer.toBinaryString(full)); // => 111111

// COMPLEMENT within the universe (items NOT in subset)
int complement = full & ~subset;
System.out.println(Integer.toBinaryString(complement)); // => 001011
```

**For `n` up to 60, switch to `long`** — same operations, `1L` discipline mandatory:

```java
int n2 = 45; // items 0..44 -> needs long
long subset2 = 0L;
subset2 |= (1L << 40);              // ADD item 40 -- must be 1L!
subset2 &= ~(1L << 40);             // REMOVE item 40
boolean has40 = (subset2 & (1L << 40)) != 0; // TEST
long full2 = (1L << n2) - 1;        // full universe mask
System.out.println(Long.bitCount(full2)); // => 45
```

### 24.5 `int`/`long` mask vs `BitSet` — one-line rule

> Use a primitive `int`/`long` bitmask when the universe size is small and fixed (`n <= ~62`, so it fits a single word) and you need it as a **DP state / hashable key / array index**; reach for `java.util.BitSet` (cross-ref Part III §15) when `n` is large/unbounded or you need dynamic resizing, because `BitSet` is not usable as a map key or DP-array index by value the way a primitive `long` is.

### 24.6 Worked bitmask-DP transition snippet (cross-ref DSA Pattern 24: Bitmask DP / TSP-style)

Classic "minimum cost to visit all nodes" (Traveling-Salesman-style) DP: `dp[mask][i]` = min cost to have visited exactly the set of nodes in `mask`, ending at node `i`.

```java
int n = 4;
int[][] cost = {
    {0, 10, 15, 20},
    {10, 0, 35, 25},
    {15, 35, 0, 30},
    {20, 25, 30, 0}
};

int INF = Integer.MAX_VALUE / 2;
int[][] dp = new int[1 << n][n];
for (int[] row : dp) Arrays.fill(row, INF);
dp[1][0] = 0; // start at node 0, mask = {0}

for (int mask = 1; mask < (1 << n); mask++) {
    for (int last = 0; last < n; last++) {
        if ((mask & (1 << last)) == 0) continue;    // last must be in mask
        if (dp[mask][last] == INF) continue;
        for (int next = 0; next < n; next++) {
            if ((mask & (1 << next)) != 0) continue; // next must NOT be in mask yet
            int newMask = mask | (1 << next);         // transition: add next to the set
            dp[newMask][next] = Math.min(dp[newMask][next], dp[mask][last] + cost[last][next]);
        }
    }
}

int best = INF;
int fullMask = (1 << n) - 1;
for (int last = 0; last < n; last++) {
    if (dp[fullMask][last] == INF) continue;
    best = Math.min(best, dp[fullMask][last] + cost[last][0]); // return to start
}
System.out.println(best); // => 80
```

> **Gotcha:** `1 << n` as the outer array dimension overflows quickly — `n <= 20` gives `2^20 ≈ 1,000,000` states, already the practical ceiling for this DP shape (states × transitions is `O(2^n * n^2)`). This is a *time* ceiling, not a `1L` vs `1` correctness issue, since `n` here indexes array size (must stay `int`-representable and small) rather than a bit position needing `long`.

### 24.7 Counting-bits DP (restated as a §24 reference item)

```java
int[] countBits(int n) {
    int[] dp = new int[n + 1];
    for (int i = 1; i <= n; i++) {
        dp[i] = dp[i >> 1] + (i & 1);   // dp[i] = popcount(i>>1) + last bit of i
    }
    return dp;
}
System.out.println(Arrays.toString(countBits(10)));
// => [0, 1, 1, 2, 1, 2, 2, 3, 1, 2, 2]
```

Equivalent one-liner using the built-in for validation/testing:

```java
for (int i = 0; i <= 10; i++) {
    System.out.print(Integer.bitCount(i) + " ");
}
// => 0 1 1 2 1 2 2 3 1 2 2
```

### 24.8 Summary cheat-sheet

| Need | Java idiom |
|---|---|
| Unsigned right shift | `x >>> k` |
| Bitmask for bit `k >= 31` | `1L << k` |
| Popcount | `Integer.bitCount(x)` / `Long.bitCount(x)` |
| floor(log2(x)) | `31 - Integer.numberOfLeadingZeros(x)` |
| Lowest set bit index | `Integer.numberOfTrailingZeros(x)` |
| Isolate lowest set bit | `x & (-x)` |
| Clear lowest set bit | `x & (x - 1)` |
| Check power of two | `x > 0 && (x & (x-1)) == 0` |
| Overflow-safe midpoint | `low + ((high - low) >>> 1)` |
| Unsigned byte value | `b & 0xFF` or `Byte.toUnsignedInt(b)` |
| Full n-bit mask | `(1 << n) - 1` (int) / `(1L << n) - 1` (long) |
| Iterate submasks of `m` | `for (int s = m; s > 0; s = (s-1) & m)` |

---

# PART VI — NUMBERS, BIGINTEGER AND I/O

## 25. Integer Arithmetic, Overflow and Safety

### 25.1 Picking a type from constraints

| Type | Range | Use when |
|---|---|---|
| `int` | ≈ -2.1e9 … 2.1e9 (±2^31) | counts/indices up to ~2×10^9, products up to that range |
| `long` | ≈ -9.2e18 … 9.2e18 (±2^63) | sums over arrays, factorials, `n*m` where either can be ~1e9, any "could this exceed 2e9?" doubt |
| `BigInteger` | unbounded | factorial of 20+, exact big products, when even `long` overflows |

Rule of thumb: if a value is a **count of up to 10^5–10^9 items multiplied together**, or a **running sum of up to 10^5 values each up to 10^9**, use `long`. When unsure, use `long` — the perf cost vs `int` is negligible on modern JVMs.

### 25.2 Overflow wraps silently — Java is well-defined, C++ is UB

> **vs C++:** In C++, signed integer overflow is **undefined behavior** — anything can happen (including compiler optimizations deleting your check). In Java, `int`/`long` overflow is **well-defined two's-complement wraparound**. It will never crash, but it silently gives a **wrong answer** — arguably worse for CP because there's no UBSan/ASan to catch it.

```java
public class Main {
    public static void main(String[] args) {
        int a = Integer.MAX_VALUE;      // 2147483647
        System.out.println(a + 1);      // => -2147483648  (wrapped, no exception)

        long b = Long.MAX_VALUE;
        System.out.println(b + 1);      // => -9223372036854775808
    }
}
```

### 25.3 `Math.*Exact` — Java's built-in overflow checker

Since these throw on overflow instead of silently wrapping, they're excellent for **debugging** a suspected overflow bug (crash immediately at the exact line instead of chasing a wrong answer 40 lines later).

```java
public class Main {
    public static void main(String[] args) throws Exception {
        System.out.println(Math.addExact(2_000_000_000, 100_000_000)); // => 2100000000
        try {
            Math.addExact(Integer.MAX_VALUE, 1);
        } catch (ArithmeticException e) {
            System.out.println("overflow caught: " + e.getMessage()); // => overflow caught: integer overflow
        }
        Math.multiplyExact(100000, 100000);     // fine (1e10 fits in... wait, doesn't fit int!)
    }
}
```

> **Gotcha:** `Math.multiplyExact(100000, 100000)` throws — `1e10` overflows `int`. This is exactly the class of bug `int*int` overflow causes silently without `Exact`. Also available: `subtractExact`, `negateExact`, `incrementExact`, `decrementExact`, `toIntExact(long)` (throws if the `long` doesn't fit in `int`).

### 25.4 Classic overflow bugs

**1. `int * int` overflows before widening to `long`** — same trap as C++:

```java
public class Main {
    public static void main(String[] args) {
        int a = 100000, b = 100000;
        long bad  = a * b;          // computed as int FIRST, then widened -> wrong
        long good = (long) a * b;   // cast BEFORE multiply -> correct
        System.out.println(bad);    // => 1410065408  (WRONG, overflowed as int)
        System.out.println(good);   // => 10000000000 (correct)
    }
}
```

**2. Binary search midpoint overflow:**

```java
int lo = 0, hi = Integer.MAX_VALUE - 1;
int midBad  = (lo + hi) / 2;        // lo+hi can overflow int
int midGood = lo + (hi - lo) / 2;   // safe
```

**3. Summing an array into an `int` when the true sum needs `long`:**

```java
int[] arr = new int[100_000];
Arrays.fill(arr, 1_000_000);
int sumBad = 0;
long sumGood = 0;
for (int x : arr) { sumBad += x; sumGood += x; }
System.out.println(sumBad);   // => wraps, wrong (overflowed int)
System.out.println(sumGood);  // => 100000000000 (correct)
```

**4. `1 << k` for large `k` — shifting an `int` literal:**

```java
System.out.println(1 << 40);    // => 256  (WRONG: int shift amount is taken mod 32!)
System.out.println(1L << 40);   // => 1099511627776 (correct: long shift, mod 64)
```

> **Gotcha:** Java shift amounts are taken **mod the operand's bit width** (mod 32 for `int`, mod 64 for `long`) — they don't overflow to 0 like you might expect, they silently wrap the *shift amount itself*. `1 << 33` equals `1 << 1` = `2`, not `0`. Same rule exists in C++ but is UB there instead of defined.

**5. `Math.abs(Integer.MIN_VALUE)` is still negative:**

```java
System.out.println(Math.abs(Integer.MIN_VALUE)); // => -2147483648
```

> **Gotcha:** `Integer.MIN_VALUE` is `-2147483648`; its positive counterpart `2147483648` doesn't fit in `int`, so `Math.abs` returns the same negative value unchanged. Use `long` (`Math.abs((long) x)`) or `Math.abs(Long.MIN_VALUE... )` has the identical issue at the `long` boundary. If `x` might be `MIN_VALUE`, widen to `long` first.

### 25.5 Division and modulo semantics

- Integer division **truncates toward zero** (same as C++11 onward).
- `%` follows the sign of the **dividend**, not the divisor (same as C++).

```java
System.out.println(-7 / 3);   // => -2   (truncated toward zero)
System.out.println(-7 % 3);   // => -1   (sign of dividend, -7)
System.out.println(7 % -3);   // => 1    (sign of dividend, 7)
```

**Java has built-in mathematically-correct (Euclidean) versions — C++ doesn't, until you hand-roll them:**

```java
System.out.println(Math.floorMod(-7, 3));  // => 2   (always same sign as divisor, non-negative here)
System.out.println(Math.floorDiv(-7, 3));  // => -3  (rounds toward negative infinity)
```

> This is a genuine Java win over C++ for CP: normalizing indices into `[0, m)` (e.g. modular hashing, circular buffers) is `Math.floorMod(x, m)` in one call instead of the usual C++ `((x % m) + m) % m` idiom.

### 25.6 No `__int128` — use `long` tricks, `Math.multiplyHigh`, or `BigInteger`

> **vs C++:** C++ (GCC/Clang) has `__int128` for cheap 128-bit intermediates (e.g. `mulmod` for numbers near `long long` range). Java has **no 128-bit primitive**. Options: `BigInteger` (simplest, slower), or `Math.multiplyHigh` **[Java 9]** to get the high 64 bits of a 64×64 multiply without allocating.

```java
import java.math.BigInteger;

public class Main {
    // mulmod via BigInteger — simple, correct, a bit slow if called millions of times
    static long mulmodBig(long a, long b, long mod) {
        return BigInteger.valueOf(a)
                .multiply(BigInteger.valueOf(b))
                .mod(BigInteger.valueOf(mod))
                .longValue();
    }

    // mulmod via Math.multiplyHigh [Java 9] — no allocation, needs unsigned 128-bit combine
    static long mulmodFast(long a, long b, long mod) {
        long high = Math.multiplyHigh(a, b);
        long low  = a * b; // low 64 bits (wraps, that's fine — it's exactly what we want)
        // combine (high:low) mod `mod` using 128-bit division emulated with Math.unsignedMultiplyHigh style logic,
        // or simplest: fall back to BigInteger for the final mod when high != 0.
        if (high == 0 && low >= 0) return Long.remainderUnsigned(low, mod);
        return BigInteger.valueOf(high).shiftLeft(64)
                .add(BigInteger.valueOf(low).and(BigInteger.valueOf(-1).shiftRight(0)).and(new BigInteger("FFFFFFFFFFFFFFFF", 16)))
                .mod(BigInteger.valueOf(mod)).longValue();
    }

    public static void main(String[] args) {
        long a = 4_000_000_000L, b = 4_000_000_000L, mod = 1_000_000_007L;
        System.out.println(mulmodBig(a, b, mod)); // => 951807488
    }
}
```

> **Practical advice:** unless the intermediate genuinely exceeds `long` range (i.e. both operands can be up to ~1e18 and you need the exact product mod something), just use `BigInteger` for the rare mulmod call — it's simple and correct. Reserve `Math.multiplyHigh` hand-rolling for when it's called in a hot loop millions of times and profiling shows `BigInteger` is the bottleneck.

### 25.7 `Integer.MAX_VALUE` as infinity — the "infinity + x overflows" trap

```java
public class Main {
    public static void main(String[] args) {
        int INF = Integer.MAX_VALUE;
        int weight = 5;
        int bad = INF + weight;          // overflows -> becomes very negative!
        System.out.println(bad);         // => -2147483644 (WRONG, looks smaller than INF)

        int SAFE_INF = Integer.MAX_VALUE / 2; // or 1_000_000_000
        int good = SAFE_INF + weight;         // no overflow, still "big"
        System.out.println(good < SAFE_INF);  // => false (correct: bigger than INF as expected)
    }
}
```

> **Gotcha:** In Dijkstra/Bellman-Ford/DP-with-infinity code, `dist[u] + w` where `dist[u] == INF` must not overflow. Either cap `INF` at `Integer.MAX_VALUE / 2` (leaves headroom for one addition) or use `long` for distances and `Long.MAX_VALUE / 2`. Never compare against `Integer.MAX_VALUE` after adding to it without this guard.

---

## 26. BigInteger and BigDecimal

> **vs C++:** C++'s standard library has **no arbitrary-precision integer type** — competitive programmers hand-roll a bignum class or use `__int128`/Python-style tricks. Java ships `BigInteger` and `BigDecimal` in `java.math` — a real advantage when a problem needs exact big-integer arithmetic (factorials, huge Fibonacci, exact combinatorics without a modulus).

### 26.1 BigInteger — construction

```java
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        BigInteger a = BigInteger.valueOf(123456789L);
        BigInteger b = new BigInteger("987654321987654321");
        BigInteger zero = BigInteger.ZERO;
        BigInteger one  = BigInteger.ONE;
        BigInteger two  = BigInteger.TWO;   // [Java 9]
        BigInteger ten  = BigInteger.TEN;

        System.out.println(a);   // => 123456789
        System.out.println(b);   // => 987654321987654321
    }
}
```

> **Gotcha:** `BigInteger` is **immutable** — every operation returns a new object. `a.add(b)` does not modify `a`; you must assign the result: `a = a.add(b)`.

### 26.2 Arithmetic operations

```java
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        BigInteger x = new BigInteger("123456789012345678901234567890");
        BigInteger y = BigInteger.valueOf(987654321);

        System.out.println(x.add(y));                 // add
        System.out.println(x.subtract(y));             // subtract
        System.out.println(x.multiply(y));              // multiply
        System.out.println(x.divide(y));                // integer division
        System.out.println(x.mod(y));                    // mod (always non-negative for positive modulus)
        System.out.println(x.pow(2));                     // pow(int exponent) — exact, exponent must be int and non-negative

        BigInteger base = BigInteger.valueOf(2), exp = BigInteger.valueOf(1000), mod = BigInteger.valueOf(1_000_000_007);
        System.out.println(base.modPow(exp, mod));         // fast modular exponentiation, huge exponent OK

        System.out.println(x.gcd(y));
        System.out.println(x.abs());
        System.out.println(x.negate());
        System.out.println(x.min(y));
        System.out.println(x.max(y));
        System.out.println(x.compareTo(y));                 // -1, 0, or 1  (NEVER use == to compare value!)
        System.out.println(x.equals(y));                     // value equality — this IS correct for BigInteger
        System.out.println(x.signum());                       // -1, 0, or 1
    }
}
```

> **Gotcha:** Use `.equals()` or `.compareTo() == 0` for value equality, **never `==`** — `==` compares object references (same trap as `String`, see Part on Strings). `.compareTo()` is also what `<`/`>` would have been; `BigInteger` has no operator overloading (Java has none at all, unlike C++).

### 26.3 Bit operations

```java
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        BigInteger x = BigInteger.valueOf(42); // 101010

        System.out.println(x.shiftLeft(3));     // => 336
        System.out.println(x.shiftRight(1));    // => 21
        System.out.println(x.and(BigInteger.valueOf(15))); // => 10
        System.out.println(x.or(BigInteger.valueOf(1)));   // => 43
        System.out.println(x.xor(BigInteger.valueOf(7)));  // => 45
        System.out.println(x.testBit(1));       // => true (bit index 1 is set)
        System.out.println(x.setBit(0));        // => 43  (returns NEW value, immutable)
        System.out.println(x.bitLength());      // => 6   (minimal two's-complement bits, excluding sign)
    }
}
```

### 26.4 `modInverse` and primality — free in Java

```java
import java.math.BigInteger;
import java.util.Random;

public class Main {
    public static void main(String[] args) {
        BigInteger a = BigInteger.valueOf(3);
        BigInteger m = BigInteger.valueOf(1_000_000_007);
        System.out.println(a.modInverse(m));         // => modular inverse of 3 mod 1e9+7, computed via extended Euclid internally

        BigInteger p = BigInteger.valueOf(1_000_000_007);
        System.out.println(p.isProbablePrime(20));    // => true  (Miller-Rabin, certainty param = # of rounds; higher = more confident)

        BigInteger next = BigInteger.valueOf(1_000_000_000).nextProbablePrime();
        System.out.println(next);                       // => 1000000007
    }
}
```

> These two — `modInverse` and `isProbablePrime` — are a real convenience: in C++ you'd hand-roll extended Euclid and Miller-Rabin. `isProbablePrime(certainty)` has false-positive probability at most `1/2^certainty`; `certainty=20`+ is effectively certain for CP purposes.

### 26.5 Conversions

```java
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        BigInteger big = new BigInteger("123456789012345678901234567890");

        System.out.println(big.intValue());       // => truncated/wraps silently if it doesn't fit (like a narrowing cast)
        System.out.println(big.longValue());       // => truncated/wraps silently if it doesn't fit
        try {
            big.intValueExact();                     // throws ArithmeticException if it doesn't fit — prefer this to catch bugs
        } catch (ArithmeticException e) {
            System.out.println("doesn't fit in int"); // => doesn't fit in int
        }
        System.out.println(big.toString(16));       // => hex string
        System.out.println(new BigInteger("ff", 16)); // => 255  (parse from radix 16)
    }
}
```

### 26.6 Performance note

> **Gotcha:** `BigInteger` operations allocate new objects and do arbitrary-precision arithmetic under the hood — **10-100x slower** than `long` arithmetic. Only reach for it when the problem genuinely needs numbers beyond `long` range (exact factorials of n≥21, huge exact products, RSA-style modpow with huge moduli). If a modulus like `1e9+7` is given, that's a strong hint to use `long` with manual mulmod, not `BigInteger`, for performance-critical inner loops.

### 26.7 BigDecimal — brief

Arbitrary-precision **decimal** (base-10, exact — unlike `double`'s binary floating point, which can't represent `0.1` exactly). Use when a problem needs exact decimal arithmetic (money, exact rounding) rather than approximate `double`.

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public class Main {
    public static void main(String[] args) {
        BigDecimal a = new BigDecimal("0.1");
        BigDecimal b = new BigDecimal("0.2");
        System.out.println(a.add(b));                          // => 0.3  (EXACT, unlike double 0.1+0.2)
        System.out.println(0.1 + 0.2);                          // => 0.30000000000000004  (double imprecision, for contrast)

        BigDecimal x = new BigDecimal("10").divide(new BigDecimal("3"), 5, RoundingMode.HALF_UP);
        System.out.println(x);                                   // => 3.33333

        BigDecimal y = new BigDecimal("2.345").setScale(2, RoundingMode.HALF_UP);
        System.out.println(y);                                    // => 2.35
    }
}
```

> **Gotcha:** Always construct `BigDecimal` from a `String`, not a `double` literal — `new BigDecimal(0.1)` captures the actual imprecise binary value of `0.1` (`0.1000000000000000055511151231257827021181583404541015625`). `new BigDecimal("0.1")` is exact. Rarely needed in CP (most CP problems use `double` + epsilon comparison, or work mod a prime), but useful to recognize for "exact decimal" problems.

### 26.8 Worked example: nCr for large numbers (exact, no modulus)

```java
import java.math.BigInteger;

public class Main {
    static BigInteger nCr(int n, int r) {
        BigInteger num = BigInteger.ONE, den = BigInteger.ONE;
        for (int i = 0; i < r; i++) {
            num = num.multiply(BigInteger.valueOf(n - i));
            den = den.multiply(BigInteger.valueOf(i + 1));
        }
        return num.divide(den);
    }

    public static void main(String[] args) {
        System.out.println(nCr(50, 25)); // => 126410606437752  (exact, no overflow, no modulus needed)
    }
}
```

### 26.9 Worked example: modPow for fast modular exponentiation

```java
import java.math.BigInteger;

public class Main {
    public static void main(String[] args) {
        // compute 2^(1e18) mod (1e9+7) — infeasible with a naive loop, instant with modPow
        BigInteger base = BigInteger.valueOf(2);
        BigInteger exp  = BigInteger.valueOf(1_000_000_000_000_000_000L);
        BigInteger mod  = BigInteger.valueOf(1_000_000_007L);
        System.out.println(base.modPow(exp, mod)); // => some value in [0, mod)
    }
}
```

> For hot loops needing modpow millions of times, hand-write it with `long` (binary exponentiation + manual mulmod) instead — `BigInteger.modPow` is convenient but allocates per call.

```java
public class Main {
    static long power(long base, long exp, long mod) {
        base %= mod;
        long result = 1;
        while (exp > 0) {
            if ((exp & 1) == 1) result = result * base % mod;
            base = base * base % mod;
            exp >>= 1;
        }
        return result;
    }

    public static void main(String[] args) {
        System.out.println(power(2, 1_000_000_000_000_000_000L, 1_000_000_007L));
    }
}
```

---

## 27. Fast I/O (the make-or-break CP topic)

> **THE RULE:** `Scanner` is convenient — parses ints/longs/doubles/words directly, no manual tokenizing — but it is **5-10x too slow** for large inputs (its regex-based parsing under the hood is expensive). On Codeforces/AtCoder with n up to 10^5-10^6+, `Scanner` routinely causes **TLE** even when your algorithm is correct. **Never use `Scanner` for large input in CP.** Use it only for tiny/interactive inputs or LeetCode (where I/O is handled for you and this doesn't matter).

### 27.1 `BufferedReader` + `StringTokenizer` — the standard fast reader

This is the default choice for 95% of CP problems: fast, and not too verbose.

```java
import java.io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

        // 1. read a single int on its own line
        int n = Integer.parseInt(br.readLine().trim());

        // 2. read n ints on ONE line, space-separated
        StringTokenizer st = new StringTokenizer(br.readLine());
        int[] arr = new int[n];
        for (int i = 0; i < n; i++) arr[i] = Integer.parseInt(st.nextToken());

        System.out.println(Arrays.toString(arr));
        br.close();
    }
}
```

> Note `throws IOException` on `main` — `BufferedReader.readLine()` is a checked exception; either declare `throws IOException` on `main` (simplest, standard in CP) or wrap in try/catch.

**Reading a 2D grid (n rows, m chars or ints each):**

```java
import java.io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer st = new StringTokenizer(br.readLine());
        int n = Integer.parseInt(st.nextToken());
        int m = Integer.parseInt(st.nextToken());

        // grid of chars, e.g. a maze
        char[][] grid = new char[n][];
        for (int i = 0; i < n; i++) grid[i] = br.readLine().toCharArray();

        // OR grid of ints, space-separated per row
        int[][] gridInt = new int[n][m];
        for (int i = 0; i < n; i++) {
            st = new StringTokenizer(br.readLine());
            for (int j = 0; j < m; j++) gridInt[i][j] = Integer.parseInt(st.nextToken());
        }
    }
}
```

**Reading until EOF (unknown number of lines):**

```java
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        String line;
        int lineCount = 0;
        while ((line = br.readLine()) != null) {
            if (line.isEmpty()) continue; // skip blank lines if the format has them
            lineCount++;
        }
        System.out.println(lineCount);
    }
}
```

**The `StringTokenizer` reuse pattern** — create one per line, don't try to hold tokens across `readLine()` calls:

```java
StringTokenizer st;
for (int i = 0; i < n; i++) {
    st = new StringTokenizer(br.readLine());   // fresh tokenizer per line
    int a = Integer.parseInt(st.nextToken());
    int b = Integer.parseInt(st.nextToken());
}
```

> **Gotcha:** if a problem's "n numbers" can be split across **multiple lines** unpredictably (rare on CF, common on some judges/ICPC-style input), `StringTokenizer` per-line breaks. Use a **streaming tokenizer** instead — read the whole input and tokenize across line boundaries — which is exactly what `StreamTokenizer` (below) or a custom byte-stream reader gives you for free.

### 27.2 `StreamTokenizer` — fastest built-in for pure-number input

Faster than `BufferedReader`+`StringTokenizer` because it tokenizes directly from the stream without building intermediate `String` objects per token, and it doesn't care about line boundaries — it just gives you the next number. Best when the input is **almost entirely numbers** (no need to distinguish words from numbers).

```java
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        StreamTokenizer in = new StreamTokenizer(new BufferedReader(new InputStreamReader(System.in)));

        in.nextToken();
        int n = (int) in.nval;

        long sum = 0;
        for (int i = 0; i < n; i++) {
            in.nextToken();
            sum += (long) in.nval;
        }
        System.out.println(sum);
    }
}
```

> **Gotcha:** `StreamTokenizer.nval` is a `double` — fine for ints and longs up to ~2^53 exactly, but for values near `Long.MAX_VALUE` precision is lost. Also by default it treats `-` as part of a word in some configs and can mishandle certain characters; for pure whitespace-separated non-negative/negative integers it's reliable, but read words/strings with it carefully (use `TT_WORD` handling) or fall back to `BufferedReader` if the input mixes text and numbers unpredictably.

### 27.3 Custom `DataInputStream` FastReader — the fastest, for extreme input sizes

Reads raw bytes directly, parses integers/longs manually with no object allocation per token. This is the "everyone copies this into their CP template" class — use it when n is huge (10^6-10^7+) and `BufferedReader` still isn't fast enough, or just use it as your default template since it's not much more code once written.

```java
import java.io.*;

public class Main {
    static class FastReader {
        private final DataInputStream in;
        private final byte[] buffer = new byte[1 << 16];
        private int bufferPointer, bytesRead;

        FastReader() { in = new DataInputStream(new BufferedInputStream(System.in, 1 << 16)); }

        private byte readByte() throws IOException {
            if (bufferPointer == bytesRead) {
                bytesRead = in.read(buffer, 0, buffer.length);
                bufferPointer = 0;
                if (bytesRead == -1) return -1; // EOF
            }
            return buffer[bufferPointer++];
        }

        int nextInt() throws IOException {
            int ret = 0;
            byte b = readByte();
            while (b <= ' ') b = readByte();            // skip whitespace/newlines
            boolean neg = (b == '-');
            if (neg) b = readByte();
            while (b >= '0' && b <= '9') {
                ret = ret * 10 + (b - '0');
                b = readByte();
            }
            return neg ? -ret : ret;
        }

        long nextLong() throws IOException {
            long ret = 0;
            byte b = readByte();
            while (b <= ' ') b = readByte();
            boolean neg = (b == '-');
            if (neg) b = readByte();
            while (b >= '0' && b <= '9') {
                ret = ret * 10 + (b - '0');
                b = readByte();
            }
            return neg ? -ret : ret;
        }

        String next() throws IOException {              // next whitespace-delimited token
            StringBuilder sb = new StringBuilder();
            byte b = readByte();
            while (b <= ' ') b = readByte();
            while (b > ' ') {
                sb.append((char) b);
                b = readByte();
            }
            return sb.toString();
        }

        String nextLine() throws IOException {
            StringBuilder sb = new StringBuilder();
            byte b = readByte();
            while (b != '\n' && b != -1) {
                if (b != '\r') sb.append((char) b);
                b = readByte();
            }
            return sb.toString();
        }

        double nextDouble() throws IOException { return Double.parseDouble(next()); }
    }

    public static void main(String[] args) throws IOException {
        FastReader fr = new FastReader();
        int n = fr.nextInt();
        long[] arr = new long[n];
        for (int i = 0; i < n; i++) arr[i] = fr.nextLong();
        long sum = 0;
        for (long x : arr) sum += x;
        System.out.println(sum);
    }
}
```

> This class is safe to paste verbatim into any submission. It handles negative numbers, mixed int/long reads, and arbitrary whitespace (spaces, tabs, newlines) between tokens.

### 27.4 Output: never `println` in a loop

`System.out` is line-buffered/auto-flushing by default in many environments, and even when it isn't, each `println` call has overhead — printing 10^5+ times in a loop is a classic **silent TLE** cause.

**Option A: build one `StringBuilder`, print once:**

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        int n = 100_000;
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n; i++) {
            sb.append(i).append('\n');   // '\n' is fine and cheaper than System.lineSeparator() here
        }
        System.out.print(sb);            // ONE call, flushed at program exit
    }
}
```

**Option B: `PrintWriter` wrapping `BufferedWriter`, with explicit `.flush()`:**

```java
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        PrintWriter pw = new PrintWriter(new BufferedWriter(new OutputStreamWriter(System.out)));

        int n = Integer.parseInt(br.readLine().trim());
        for (int i = 0; i < n; i++) {
            pw.println(i);          // buffered — cheap, not flushed to the OS each call
        }
        pw.flush();                  // MUST flush (or close) before exit, or output may be lost/truncated
    }
}
```

> **Gotcha:** if you forget `pw.flush()` (or `pw.close()`) at the end, buffered output can be **lost entirely** — the program exits before the buffer is written out. Always flush before `main` returns. Use `bw.newLine()` (platform-specific line separator) if writing through a raw `BufferedWriter`, or just append `'\n'` (Unix newline, universally accepted by judges) when building strings manually — `'\n'` is simpler and what most competitive programmers use.

**`System.out.flush()` for interactive problems** — when the judge reads your output as you produce it (interactive problems, judge-in-the-loop):

```java
System.out.print("? 5 3");
System.out.flush();     // force it out NOW so the judge sees the query immediately
// then read the judge's response...
```

> For interactive problems, either call `System.out.flush()` after every query, or construct `PrintWriter` with `autoFlush=true`: `new PrintWriter(new BufferedWriter(new OutputStreamWriter(System.out)), true)` — but note `autoFlush` only flushes on `println`/`\n`, not `print`.

### 27.5 Speed/convenience comparison

| Method | Relative speed | Convenience | Use when |
|---|---|---|---|
| `Scanner` | slowest (baseline 1x) | highest — `nextInt()`, `nextLine()`, etc., no setup | tiny input, LeetCode, interactive/quick scripts — **never for CF with large n** |
| `BufferedReader` + `StringTokenizer` | ~10-20x faster than Scanner | high — one line of boilerplate | default choice for most CP problems |
| `StreamTokenizer` | ~20-30x faster than Scanner | medium — numbers only, `nval` is `double` | pure-numeric input, want max speed with minimal code |
| Custom `DataInputStream`/byte-buffer reader | fastest (~30-50x+ faster than Scanner) | lower — need the boilerplate class (copy-paste once) | n ≥ 10^6-10^7, tight TL, or just as a standard template |

### 27.6 Formatted output: `String.format` / `printf`

```java
public class Main {
    public static void main(String[] args) {
        double pi = 3.14159265358979;
        System.out.printf("%.9f%n", pi);              // => 3.141592654   (%n is platform newline; use \n for CP)
        String s = String.format("%.2f", pi);           // => "3.14"
        System.out.println(s);

        System.out.printf("%5d|%n", 42);                 // => "   42|"  (width 5, right-aligned)
        System.out.printf("%-5d|%n", 42);                  // => "42   |"  (left-aligned)
        System.out.printf("%05d%n", 42);                    // => "00042" (zero-padded)
        System.out.printf("%x %o%n", 255, 8);                 // => "ff 10" (hex, octal)
    }
}
```

> **Gotcha:** for CP output, prefer `'\n'` inside `String.format`/`sb.append` over `%n` — `%n` produces `\r\n` on Windows judges configured that way, occasionally causing "wrong answer: extra whitespace" on strict checkers. `System.out.println` itself uses the platform line separator too, but almost all judges accept it; `%n` inside format strings is the more common footgun since people forget it isn't `\n`.

### 27.7 Reading doubles

```java
import java.io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringTokenizer st = new StringTokenizer(br.readLine());
        double x = Double.parseDouble(st.nextToken());
        double y = Double.parseDouble(st.nextToken());
        System.out.println(x + y);
    }
}
```

> `Double.parseDouble` handles standard decimal and scientific notation (`"1e-9"`, `"-3.5"`) directly — no extra work needed, same as C++'s `atof`/`stod`.

---

## 28. Randomness, Time and Useful Library Bits

### 28.1 `java.util.Random`

```java
import java.util.Random;

public class Main {
    public static void main(String[] args) {
        Random rnd = new Random(42);          // seeded -> reproducible sequence, good for debugging

        System.out.println(rnd.nextInt());          // any int, full range
        System.out.println(rnd.nextInt(100));         // int in [0, 100)
        System.out.println(rnd.nextInt(10, 20));       // [Java 17] int in [10, 20)
        System.out.println(rnd.nextLong());              // any long
        System.out.println(rnd.nextDouble());              // double in [0.0, 1.0)
        System.out.println(rnd.nextBoolean());               // true/false
    }
}
```

### 28.2 `ThreadLocalRandom` — faster, no shared-seed contention

```java
import java.util.concurrent.ThreadLocalRandom;

public class Main {
    public static void main(String[] args) {
        int x = ThreadLocalRandom.current().nextInt(0, 100); // [0, 100)
        System.out.println(x);
    }
}
```

> `ThreadLocalRandom.current()` avoids the internal CAS-based seed update `java.util.Random` does (relevant only under multithreading — irrelevant for typical single-threaded CP, but it's also just slightly faster and has a cleaner `nextInt(origin, bound)` API pre-Java 17). Either is fine for single-threaded CP.

### 28.3 Defeating the anti-quicksort hack with a random seed

> Cross-ref Part IV §17: `Arrays.sort` on primitive arrays is dual-pivot quicksort, which has known **adversarial worst-case inputs** that some CF problems specifically construct to force O(n^2) and TLE your submission. Shuffling the array with a random seed before sorting defeats this, since the adversary can't target a randomized order.

```java
import java.util.*;

public class Main {
    static void shuffle(int[] arr) {
        Random rnd = new Random();
        for (int i = arr.length - 1; i > 0; i--) {
            int j = rnd.nextInt(i + 1);
            int tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
        }
    }

    public static void main(String[] args) {
        int[] arr = {5, 3, 8, 1, 9, 2};
        shuffle(arr);
        Arrays.sort(arr);
        System.out.println(Arrays.toString(arr)); // => [1, 2, 3, 5, 8, 9]  (shuffle doesn't affect final sorted order)
    }
}
```

> **Gotcha:** shuffling doesn't change the *result* (still fully sorted), only defeats the adversarial *input pattern* that targets quicksort's pivot selection. `Collections.shuffle` works the same way for `List<Integer>` (boxed), but note boxed sorting (`Collections.sort`/`List.sort`) uses **TimSort**, which has no known quicksort-style adversarial worst case — so shuffling is specifically a primitive-array-`Arrays.sort` concern.

### 28.4 Benchmarking: `System.nanoTime()` / `System.currentTimeMillis()`

```java
public class Main {
    public static void main(String[] args) {
        long start = System.nanoTime();
        long sum = 0;
        for (int i = 0; i < 10_000_000; i++) sum += i;
        long elapsedNanos = System.nanoTime() - start;
        System.out.println(sum);
        System.out.println(elapsedNanos / 1_000_000.0 + " ms");

        long ms = System.currentTimeMillis();  // wall-clock epoch millis — coarser resolution, useful for timestamps
    }
}
```

> Use `System.nanoTime()` for **measuring elapsed time / benchmarking** (monotonic, high resolution, not affected by system clock changes). Use `System.currentTimeMillis()` for **wall-clock timestamps** (e.g. "what time is it"). Never use `currentTimeMillis()` to measure elapsed durations — it can jump if the system clock is adjusted.

### 28.5 `System.arraycopy` — fast bulk copy (restated)

```java
int[] src = {1, 2, 3, 4, 5};
int[] dst = new int[5];
System.arraycopy(src, 0, dst, 0, src.length); // src, srcPos, dst, dstPos, length
System.out.println(Arrays.toString(dst)); // => [1, 2, 3, 4, 5]
```

> Intrinsic/native bulk copy — faster than a manual loop, same idea as C++'s `memcpy`/`std::copy`. `Arrays.copyOf`/`copyOfRange` use this internally.

### 28.6 Key library classes at a glance

| Class | Package | Provides |
|---|---|---|
| `Scanner` | `java.util` | convenient but slow parsing input — avoid for large CP input |
| `BufferedReader` | `java.io` | fast buffered line reading |
| `StringTokenizer` | `java.util` | fast whitespace/delimiter tokenizing of a line |
| `StreamTokenizer` | `java.io` | fast numeric token stream, ignores line boundaries |
| `PrintWriter` | `java.io` | buffered, flushable output wrapper |
| `StringBuilder` | `java.lang` | mutable string, O(1) amortized append — build output/strings |
| `Arrays` | `java.util` | `sort`, `fill`, `copyOf`, `binarySearch`, `equals`, `toString`, `asList` for arrays |
| `Collections` | `java.util` | `sort`, `reverse`, `shuffle`, `max`/`min`, `unmodifiableX` for `List`/collections |
| `Math` | `java.lang` | `abs`, `max`, `min`, `pow`, `sqrt`, `floorDiv`, `floorMod`, `*Exact` overflow-checked ops |
| `Integer` / `Long` | `java.lang` | `parseInt`/`parseLong`, `MAX_VALUE`/`MIN_VALUE`, `toBinaryString`, `bitCount`, boxing utils |
| `BigInteger` / `BigDecimal` | `java.math` | arbitrary-precision integer / decimal arithmetic |
| `Random` | `java.util` | pseudo-random numbers, seedable |
| `PriorityQueue` | `java.util` | binary heap (min-heap by default) |
| `ArrayDeque` | `java.util` | fast stack/queue/deque, preferred over `Stack`/`LinkedList` |
| `TreeMap` / `TreeSet` | `java.util` | sorted map/set, O(log n) ops, `floor`/`ceiling`/`higher`/`lower` |
| `HashMap` / `HashSet` | `java.util` | O(1) average hash-based map/set, no ordering |
| `List` / `Map` / `Set` | `java.util` | core collection interfaces (`ArrayList`, `HashMap`, `HashSet`, etc. are implementations) |

### 28.7 The CP header and the C++ `bits/stdc++.h` contrast

> **vs C++:** C++ CP code typically starts with `#include <bits/stdc++.h>` — one header pulls in the entire standard library. Java has **no equivalent single import**; you import what you use, package by package. In practice two wildcard imports cover nearly everything needed:

```java
import java.util.*;   // collections: List, Map, Set, ArrayDeque, PriorityQueue, Arrays, Collections, Scanner, StringTokenizer, Random, ...
import java.io.*;     // BufferedReader, PrintWriter, InputStreamReader, IOException, StreamTokenizer, DataInputStream, ...

public class Main {
    public static void main(String[] args) throws IOException {
        // ...
    }
}
```

Add as needed: `import java.math.*;` (BigInteger/BigDecimal), `import java.util.stream.*;` (streams), `import java.util.concurrent.ThreadLocalRandom;`.

### 28.8 `public class Main` on Codeforces vs LeetCode's `class Solution`

> **Gotcha:** Codeforces (and most online judges that take a full program) require the **public class to be named exactly `Main`**, containing `public static void main(String[] args)` — this is the compiler entry point convention, not optional. A file `Main.java` with `public class Main { public static void main(...) { ... } }` is the standard CP skeleton.

```java
// Codeforces / AtCoder style — the whole program
import java.util.*;
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        int n = Integer.parseInt(br.readLine().trim());
        System.out.println(n * 2);
    }
}
```

```java
// LeetCode style — no main(), no I/O; you implement a method on a pre-declared class,
// the judge's harness constructs Solution and calls the method with test-case arguments.
class Solution {
    public int twoSumFirstIndex(int[] nums, int target) {
        Map<Integer, Integer> seen = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            if (seen.containsKey(target - nums[i])) return seen.get(target - nums[i]);
            seen.put(nums[i], i);
        }
        return -1;
    }
}
```

| | Codeforces / AtCoder / most judges | LeetCode |
|---|---|---|
| Class name | must be `public class Main` | `class Solution` (given, not renamed) |
| Entry point | you write `main(String[] args)` | judge calls your method directly |
| I/O | you read stdin / write stdout yourself | no I/O — method takes args, returns result |
| File | single file, `Main.java` | single file, class body only |

---

# PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT

## 29. The Contest Template and Structure

### 29.1 The full Codeforces template

On Codeforces the public class **must** be named `Main` (the judge compiles `Main.java`). This is the template to paste at the start of every contest.

```java
import java.util.*;
import java.io.*;

public class Main {
    public static void main(String[] args) {
        // Run everything inside a thread with a big stack — the default
        // JVM thread stack (~512KB-1MB) overflows on deep recursion
        // (e.g. DFS on a chain graph of 10^5 nodes). See §30 for why.
        new Thread(null, Main::run, "main", 1 << 26).start(); // 64MB stack
    }

    static FastReader in;
    static StringBuilder sb; // buffer ALL output, flush once at the end

    static void run() {
        try {
            in = new FastReader(System.in);
            sb = new StringBuilder();

            int t = in.nextInt();
            while (t-- > 0) {
                solve();
            }

            System.out.print(sb); // single flush — see §31 for why this matters
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    static void solve() throws IOException {
        int n = in.nextInt();
        long[] a = new long[n];
        for (int i = 0; i < n; i++) a[i] = in.nextLong();

        long sum = 0;
        for (long x : a) sum += x;

        sb.append(sum).append('\n');
    }

    // ---- fast input ----
    static class FastReader {
        BufferedReader br;
        StringTokenizer st;

        FastReader(InputStream is) {
            br = new BufferedReader(new InputStreamReader(is));
        }

        String next() throws IOException {
            while (st == null || !st.hasMoreTokens()) {
                st = new StringTokenizer(br.readLine());
            }
            return st.nextToken();
        }

        int nextInt() throws IOException { return Integer.parseInt(next()); }
        long nextLong() throws IOException { return Long.parseLong(next()); }
        double nextDouble() throws IOException { return Double.parseDouble(next()); }

        int[] nextIntArray(int n) throws IOException {
            int[] a = new int[n];
            for (int i = 0; i < n; i++) a[i] = nextInt();
            return a;
        }

        long[] nextLongArray(int n) throws IOException {
            long[] a = new long[n];
            for (int i = 0; i < n; i++) a[i] = nextLong();
            return a;
        }

        String nextLine() throws IOException { return br.readLine(); }
    }
}
```

### 29.2 Line-by-line breakdown

| Piece | Why it's there |
|---|---|
| `public class Main` | CF requires the public class to be named `Main`; the filename is `Main.java`. |
| `import java.util.*; import java.io.*;` | Covers `Scanner`/`ArrayList`/`Deque`/etc. and `BufferedReader`/`IOException`. Java has no wildcard-free convenience — CP code just imports everything. |
| `new Thread(null, Main::run, "main", 1<<26).start()` | Spawns a **new** thread with an explicit 64MB stack, then runs all logic there. `main()` itself returns almost immediately; the real work happens on the big-stack thread. Needed because recursion depth in graph/tree DFS on CP-sized inputs (10^4–10^6) blows the default stack → `StackOverflowError`. This is a Java-specific concern; C++ contestants rarely think about stack size because the OS default (often 1MB–8MB) plus tail-call-friendly compilers usually suffice, and when it doesn't, they just bump `ulimit -s` — Java offers no such external knob on a judge sandbox, so the fix has to be in-process. |
| `FastReader` field | Wraps `BufferedReader` + `StringTokenizer` — `Scanner` is far too slow for CP-sized input (see §31). |
| `main` isn't `throws IOException` here | Because I/O happens in `run()`; wrapping the checked exception in a `RuntimeException` keeps `main`'s signature clean for the `Thread` constructor reference. You may instead just declare `run() throws IOException` and catch inside `main`'s lambda — both are common. |
| `StringBuilder sb` flushed once | Building output with `System.out.println` per line is slow (each call may flush); appending to one `StringBuilder` and printing once at the end amortizes I/O cost. |
| `int t = in.nextInt(); while (t-- > 0) solve();` | The standard CF multi-test-case loop. For single-case problems, just delete this loop and call `solve()` once. |

### 29.3 Minimal LeetCode skeleton

LeetCode gives you the method signature and handles I/O — no template, no `Main`, no multi-test loop.

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> seen = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int need = target - nums[i];
            if (seen.containsKey(need)) {
                return new int[]{seen.get(need), i};
            }
            seen.put(nums[i], i);
        }
        return new int[]{};
    }
}
```

> **Gotcha:** LeetCode's Java judge does **not** give you a big-stack thread wrapper. If your recursive solution is deep enough to overflow the default stack (rare on LC's smaller constraints, but possible), you must either convert to iteration or spawn your own thread inside the method — LC won't do it for you like a hand-rolled CF template does.

### 29.4 No preprocessor — Java has no macros

> **vs C++:** C++ CP code leans on `#define int long long`, `#define all(x) x.begin(),x.end()`, `#define rep(i,n) for(int i=0;i<n;i++)`. Java has **no preprocessor** — no `#define`, no textual macros, no `all(x)` shorthand. Every "macro" becomes a real helper method or a `static final` constant. This is a genuine ergonomics loss; budget extra keystrokes for CP in Java.

Common tiny helpers people paste into every template:

```java
// gcd (Java 17 has no builtin — Math.gcd doesn't exist until nowhere; roll your own)
static long gcd(long a, long b) {
    return b == 0 ? a : gcd(b, a % b);
}

// sort an int[] safely (defeats the anti-quicksort hack — see §31)
static void shuffleSort(int[] a) {
    Random rnd = new Random();
    for (int i = a.length - 1; i > 0; i--) {
        int j = rnd.nextInt(i + 1);
        int tmp = a[i]; a[i] = a[j]; a[j] = tmp;
    }
    Arrays.sort(a);
}

// read a whole array in one line — folded into FastReader.nextIntArray above,
// but here as a standalone if you're using plain BufferedReader + split
static int[] readIntArray(BufferedReader br, int n) throws IOException {
    StringTokenizer st = new StringTokenizer(br.readLine());
    int[] a = new int[n];
    for (int i = 0; i < n; i++) a[i] = Integer.parseInt(st.nextToken());
    return a;
}
```

### 29.5 Conditional debug without a preprocessor

C++ uses `#ifdef LOCAL ... #endif` to strip debug prints in the submitted binary. Java has no such mechanism, so the idiom is a `static final boolean` flag — the JIT dead-code-eliminates the branch when it's `false`, so it costs nothing at runtime once compiled (and you flip it to `true` locally, `false` before submitting).

```java
static final boolean DEBUG = false; // flip to true when testing locally

static void dbg(Object... vals) {
    if (!DEBUG) return;
    System.err.println(Arrays.deepToString(vals));
}

// usage inside solve():
dbg("n =", n, "arr =", a);
```

> **Gotcha:** Print debug output to `System.err`, not `System.out` — judges only compare `stdout`, so stray debug lines on `stdout` cause a Wrong Answer even when the logic is correct. `System.err` is invisible to the checker.

---

## 30. Recursion, Stack, and JVM Gotchas for CP

### 30.1 The default stack overflows on deep recursion

The JVM's default thread stack size is small (platform-dependent, commonly 512KB–1MB, sometimes reported as ~320KB effective usable frames for recursive methods with several local variables). A DFS over a chain-shaped graph or a recursive tree traversal with depth 10^4–10^5 will throw:

```
Exception in thread "main" java.lang.StackOverflowError
	at Main.dfs(Main.java:42)
	at Main.dfs(Main.java:45)
	at Main.dfs(Main.java:45)
	... (repeats thousands of times)
```

> **vs C++:** C++ programs typically get a much larger default stack from the OS (Linux default is often 8MB), and many CF setups even raise it further. The same recursive DFS that overflows in Java's default thread runs fine in C++ without any special handling. This is one of the sharpest Java-specific CP gotchas — code that is "obviously correct" fails only because of the runtime, not the algorithm.

### 30.2 Fix #1 — run inside a thread with a bigger stack

This is the *complete*, working pattern (same as §29 but shown standalone so it's copy-pasteable into any file, not just the full template):

```java
import java.util.*;

public class Main {
    public static void main(String[] args) {
        new Thread(null, Main::run, "main", 1 << 26).start(); // 64MB stack
    }

    static int[] adj_head, adj_next, adj_to;
    static int edgeCount = 0;
    static boolean[] visited;

    static void addEdge(int u, int v) {
        adj_to[edgeCount] = v;
        adj_next[edgeCount] = adj_head[u];
        adj_head[u] = edgeCount++;
    }

    static void dfs(int u) {
        visited[u] = true;
        for (int e = adj_head[u]; e != -1; e = adj_next[e]) {
            int v = adj_to[e];
            if (!visited[v]) dfs(v); // safe now even at depth 10^5+
        }
    }

    static void run() {
        int n = 200_000;
        adj_head = new int[n];
        Arrays.fill(adj_head, -1);
        adj_next = new int[2 * n];
        adj_to = new int[2 * n];
        visited = new boolean[n];

        // build a long chain 0-1-2-...-n-1 to stress recursion depth
        for (int i = 0; i + 1 < n; i++) {
            addEdge(i, i + 1);
            addEdge(i + 1, i);
        }

        dfs(0); // would StackOverflowError on the default main thread
        System.out.println("done, visited all: " +
                java.util.stream.IntStream.range(0, n).allMatch(i -> visited[i]));
    }
}
```

`1 << 26` is 64MB; bump to `1 << 27` (128MB) if still overflowing on extreme depth. There's no downside to over-allocating — the OS reserves virtual address space, not physical memory, up front.

### 30.3 Fix #2 — convert recursion to iteration (explicit stack)

Sometimes you can't or don't want a thread wrapper (e.g. inside a LeetCode `class Solution`, where you don't control `main`). The alternative is manually simulating the call stack:

```java
static void dfsIterative(int start, List<List<Integer>> adj) {
    boolean[] visited = new boolean[adj.size()];
    Deque<Integer> stack = new ArrayDeque<>(); // ArrayDeque, NOT java.util.Stack — see §31
    stack.push(start);

    while (!stack.isEmpty()) {
        int u = stack.pop();
        if (visited[u]) continue;
        visited[u] = true;

        for (int v : adj.get(u)) {
            if (!visited[v]) stack.push(v);
        }
    }
}
```

> **Gotcha:** Naive stack-based DFS (as above) does not preserve the exact recursive pre/post-order semantics (e.g. computing subtree sizes on the way back up). For that, either push a `(node, state)` pair to fake "entering" vs "leaving" a frame, or just use the thread-stack trick and keep true recursion — it's usually less error-prone.

### 30.4 JVM warmup / JIT — the first run is slow

Java bytecode starts out interpreted; the JIT compiler only kicks in and compiles hot methods to native code after they've run enough times (typically thousands of invocations). This means:

- A tight loop's **first** iterations run much slower than its later iterations.
- Micro-benchmarking a single small run inside a submission is misleading — the real submission is one-shot, so you never benefit from a "warmed up" JIT the way a long-running server does.
- This is *why online judges give Java (and other JIT/managed languages) a time-limit multiplier* relative to C++ — commonly 2x–3x on Codeforces, sometimes stated explicitly in the problem ("time limit is multiplied by 3 for Java"). Don't assume the multiplier always fully compensates — on very tight problems it can still be a squeeze.

> **vs C++:** C++ is ahead-of-time compiled to native machine code — there's no warmup phase, no interpreter tier, no JIT compilation happening during the run. The first loop iteration is exactly as fast as the millionth.

### 30.5 Garbage collection pauses — reduce allocation in hot loops

The JVM's garbage collector periodically pauses (or runs concurrently with) your program to reclaim unused objects. In a tight CP hot loop, excessive allocation means more GC work, which can push you over the time limit even though the algorithmic complexity is correct.

Practical rules for hot loops:
- **Reuse arrays** instead of allocating a new one per iteration/test case.
- **Avoid boxing** (`Integer`, `Long`) — each boxed value is a heap object; a loop that autoboxes millions of times creates millions of short-lived objects.
- **Avoid creating objects inside inner loops** — no `new int[]{...}`, no `new Pair(...)`, no lambda capturing state, inside a loop that runs 10^7+ times.

```java
// BAD: allocates a new int[] and a new HashMap entry-ish boxed Integer every iteration
List<int[]> bad = new ArrayList<>();
for (int i = 0; i < n; i++) {
    bad.add(new int[]{i, i * i}); // n small objects on the heap
}

// GOOD: two parallel primitive arrays, zero per-iteration allocation
int[] xs = new int[n];
int[] ys = new int[n];
for (int i = 0; i < n; i++) {
    xs[i] = i;
    ys[i] = i * i;
}
```

### 30.6 The autoboxing cost of `ArrayList<Integer>` / `HashMap<Integer,Integer>`

Generics in Java cannot hold primitives — `ArrayList<int>` doesn't compile. Every `int` you put into an `ArrayList<Integer>` or use as a `HashMap<Integer,Integer>` key/value gets **autoboxed** into an `Integer` object (a heap allocation, modulo the small-integer cache for `-128..127`), and every read **unboxes** it back. At CP scale (10^5–10^7 elements) this is a real, measurable slowdown plus GC pressure.

```java
// slower: ~10^6 Integer objects allocated + boxed
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) list.add(i);

// faster: one contiguous primitive array, zero boxing
int[] arr = new int[1_000_000];
for (int i = 0; i < 1_000_000; i++) arr[i] = i;
```

| Structure | Boxed? | When to use |
|---|---|---|
| `int[]`, `long[]`, `double[]` | No | Default for hot loops, known-size data |
| `ArrayList<Integer>` | Yes | Dynamic size needed, not perf-critical, or size unknown upfront |
| `HashMap<Integer,Integer>` | Yes (both K and V) | Sparse key sets; fine unless it's the hot path |
| `int[]` used as a "map" via direct indexing | No | When keys are small dense integers — just use an array instead of a map |

> **Gotcha:** `map.get(key)` on a `HashMap<Integer,Integer>` returns a boxed `Integer`; if the key is absent it returns `null`, and assigning that to an `int` triggers unboxing of `null` → `NullPointerException`. Always check `containsKey` or use `getOrDefault(key, 0)`.

### 30.7 `-Xss` / `-Xmx` — can't set them on judges

Locally you can run `java -Xss64m -Xmx256m Main` to raise the stack size and heap cap from the command line. **On Codeforces/LeetCode you cannot pass JVM flags** — you only submit source, and the judge controls the invocation. That's precisely why the big-stack-thread trick in §29/§30.2 exists: it's the only way to get a bigger stack *from within the program itself*, since the external `-Xss` lever is unavailable.

### 30.8 Why `Arrays.asList` / streams allocate

```java
List<Integer> view = Arrays.asList(1, 2, 3); // fixed-size wrapper — allocates a wrapper object
                                              // AND boxes each int literal
int[] a = {1, 2, 3};
int sum = Arrays.stream(a).sum(); // allocates an IntStream pipeline object graph
```

`Arrays.asList(int[]...)` on a **primitive** array is a classic trap: `Arrays.asList(a)` where `a` is `int[]` produces a `List<int[]>` of size 1 (the whole array as one boxed element), not a `List<Integer>` — you need `Arrays.stream(a).boxed().collect(...)` for that, which allocates even more. Streams in general build an internal pipeline of objects and (for `IntStream`/boxed streams) can autobox — convenient, but in a hot loop over 10^7 elements, a plain `for` loop over a primitive array is faster and allocates nothing.

---

## 31. Performance and Anti-Test Defenses

### 31.1 The `Arrays.sort(int[])` anti-quicksort hack

`Arrays.sort(int[])` (and other primitive-array overloads) uses a **dual-pivot quicksort**. Codeforces problems occasionally include *adversarial test data specifically crafted to trigger quicksort's worst case* against known pivot strategies, degrading it to **O(n²)** and causing a Time Limit Exceeded — even though your algorithm is O(n log n) by construction. This is a well-known, named trap in the CF community.

> **Gotcha:** This is NOT a hypothetical — problems have literally included "hack" test cases targeting `Arrays.sort(int[])` specifically because the dual-pivot quicksort pattern is public and predictable.

**Fix 1 — shuffle before sorting** (defeats any pivot-selection adversary, since the input order is now randomized):

```java
static void shuffleSort(int[] a) {
    Random rnd = new Random();
    for (int i = a.length - 1; i > 0; i--) {
        int j = rnd.nextInt(i + 1);
        int tmp = a[i]; a[i] = a[j]; a[j] = tmp;
    }
    Arrays.sort(a);
}
```

**Fix 2 — box to `Integer[]` and sort that instead.** `Arrays.sort(Object[])` uses **TimSort** (a merge-sort variant), which is **not** vulnerable to the same quicksort-pivot attack and is guaranteed O(n log n) worst case.

```java
Integer[] boxed = Arrays.stream(a).boxed().toArray(Integer[]::new);
Arrays.sort(boxed); // TimSort — safe from the quicksort hack, but boxing costs time/memory
int[] back = Arrays.stream(boxed).mapToInt(Integer::intValue).toArray();
```

| Approach | Safe from hack? | Cost |
|---|---|---|
| `Arrays.sort(int[])` raw | No | Fastest normally, O(n²) risk on adversarial CF data |
| `shuffleSort(int[])` | Yes | One extra O(n) shuffle pass, stays primitive (fast) |
| Box to `Integer[]`, `Arrays.sort` | Yes | Boxing overhead + more memory, guaranteed O(n log n) |

> **vs C++:** `std::sort` in C++ has the *identical* underlying vulnerability (introsort falls back to quicksort for the partitioning phase) and CF has historically had the same anti-`sort` hacks against C++ too — this isn't Java being uniquely bad, but Java CPers need to know their own standard-library sort has the same soft spot and apply the same defenses (shuffle, or use a merge-sort-backed container).

### 31.2 HashMap collision attacks — the Java analogue of `unordered_map` hacks

Just as C++'s `unordered_map` can be hash-flooded on Codeforces (well-known "hack unordered_map" technique using the default hash of small integers), Java's `HashMap<Integer,...>` can *also* be adversarially collided, because `Integer.hashCode()` is simply the int value itself, which is entirely predictable to a test-setter.

**Fix — salt the key with a random offset before hashing** (XOR trick):

```java
static final long SALT = System.nanoTime() ^ (long) new Random().nextInt();

static long saltedKey(int key) {
    // spread bits so predictable input integers no longer collide predictably
    long x = key ^ SALT;
    x = (x ^ (x >>> 33)) * 0xff51afd7ed558ccdL;
    x = (x ^ (x >>> 33)) * 0xc4ceb9fe1a85ec53L;
    return x ^ (x >>> 33);
}

// usage: wrap every map key
Map<Long, Integer> map = new HashMap<>();
map.put(saltedKey(someInt), value);
```

Simpler variant if you just need to defeat *sequential/known* adversarial keys without a full mix function — encode as a `long` with a random per-run offset added:

```java
static final long OFFSET = new Random().nextInt(Integer.MAX_VALUE);
static long key(int x) { return x + OFFSET; } // shifts which buckets collide, unknown to test-setter
```

**Alternative fix — sidestep hashing entirely:**
- Use a **manual open-addressing `int[]`-based hash table** you control (predictable, no adversarial dependency on `Integer.hashCode()`).
- Or use `TreeMap<Integer,Integer>` (red-black tree, O(log n) but immune to hash-flooding since it's comparison-based, not hash-based).

```java
TreeMap<Integer, Integer> safe = new TreeMap<>(); // O(log n) ops, immune to hash collision attacks
```

> **vs C++:** The C++ fix for `unordered_map` hacking is nearly identical in spirit — either supply a custom hash with a random salt, or fall back to `std::map` (the tree-based analogue of `TreeMap`). Same threat model, same two remedies, different syntax.

### 31.3 The constant-factor reality — Java vs C++ speed

Java is typically **~1.5x–2x slower** than well-written C++ for CPU-bound numeric/array work, mostly due to: JIT warmup (§30.4), array bounds-checking on every access (C++ has none by default), and GC overhead. Judges usually compensate with a time-limit multiplier, but *not always generously enough* — treat the multiplier as a cushion, not a guarantee.

**n → complexity rule-of-thumb, adjusted for Java's constant factor** (assuming a ~2-3x TL multiplier is in effect; if unsure, assume no multiplier and be more conservative):

| n | Safe complexity in Java | Notes |
|---|---|---|
| ≤ 10 | up to O(n!) / O(2ⁿ · n) | brute force / bitmask DP fine |
| ≤ 20 | O(2ⁿ · n) | bitmask DP, subset enumeration |
| ≤ 500 | O(n³) | with care on constant factor |
| ≤ 5,000 | O(n²) | fine if inner loop is simple (primitive arrays) |
| ≤ 10⁵ | O(n log n) | sort, segment tree, DSU with path compression |
| ≤ 10⁶ | O(n log n) or tight O(n) | avoid boxing/streams in the hot path |
| ≤ 10⁷–10⁸ | O(n) only, and only with primitive arrays, no boxing, no per-element object creation | this is where Java's constant factor really bites — budget extra margin vs the equivalent C++ estimate |

### 31.4 Cache-friendliness

- Prefer **primitive arrays** (`int[]`, `long[]`) over `ArrayList<Integer>` — contiguous memory, no pointer chasing, no boxing.
- Prefer a **1D flattened array** over `int[][]` in hot code: a true 2D Java array is an array of references to separate row arrays (jagged, not contiguous), so `grid[i][j]` involves an extra pointer dereference per access versus `flat[i * cols + j]`.

```java
// jagged, one extra indirection per access
int[][] grid = new int[rows][cols];

// flattened, single contiguous block, cache-friendlier in tight loops
int[] flat = new int[rows * cols];
int val = flat[i * cols + j];
```

- Avoid `LinkedList` and `TreeMap` in hot loops when a simpler structure works — `LinkedList` has poor cache locality (node-per-element, scattered on the heap) and O(n) random access; `TreeMap` is O(log n) per op versus O(1) amortized for `HashMap`/array indexing.

### 31.5 StringBuilder over `+` in loops (restated)

```java
// BAD: string concatenation in a loop is O(n^2) total — each `+` allocates a new String
String result = "";
for (int i = 0; i < n; i++) result += a[i] + " ";

// GOOD: StringBuilder mutates an internal buffer, O(n) total
StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) sb.append(a[i]).append(' ');
String result = sb.toString();
```

### 31.6 BufferedReader / StringBuilder I/O (restated)

`Scanner` is convenient but slow (regex-based tokenizing, synchronized internals) — never use it for large CP input. `System.out.println` per line is also slow (frequent small flushes). The standard fix, already baked into the §29 template:

```java
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
StringBuilder out = new StringBuilder();
// ... read via br + StringTokenizer, write via out.append(...) ...
System.out.print(out); // one flush at the very end
```

### 31.7 Avoid `String.split` in hot loops

`String.split(regex)` compiles a regex `Pattern` **every call** (unless the pattern happens to hit a tiny internal fast-path for single-character non-special delimiters) — calling it once per line across thousands of lines adds real overhead versus a non-regex tokenizer.

```java
// SLOWER in a tight loop: regex overhead per call
String[] parts = line.split(" ");

// FASTER: StringTokenizer avoids regex entirely
StringTokenizer st = new StringTokenizer(line);
while (st.hasMoreTokens()) {
    int x = Integer.parseInt(st.nextToken());
}

// FASTEST for known-simple formats: manual index scanning, zero allocation of token strings
```

> **Gotcha:** `split(" ")` also breaks silently on multiple consecutive spaces or tabs (produces empty-string tokens), which `StringTokenizer` handles gracefully by default (it treats runs of whitespace as one delimiter).

### 31.8 Prefer `ArrayDeque` over `Stack` / `LinkedList`

```java
Deque<Integer> stack = new ArrayDeque<>(); // as a stack: push/pop
stack.push(1);
stack.pop();

Deque<Integer> queue = new ArrayDeque<>(); // as a queue: offer/poll
queue.offer(1);
queue.poll();
```

`java.util.Stack` is a legacy class built on `Vector` — every method is `synchronized` (unneeded lock overhead in single-threaded CP code). `LinkedList` allocates a node object per element and has poor cache locality. `ArrayDeque` is backed by a resizable circular array — no synchronization, no per-element node allocation, and is faster for both stack and queue use in essentially all CP scenarios.

---

## 32. Debugging, Exceptions and Compiling

### 32.1 Reading a Java stack trace

```
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "Integer.intValue()" because the return value of "java.util.Map.get(Object)" is null
	at Main.solve(Main.java:23)
	at Main.run(Main.java:14)
	at Main.main(Main.java:9)
Caused by: java.lang.ArithmeticException: / by zero
	at Main.helper(Main.java:31)
	... 3 more
```

- **Read top-down; the first line after `Exception in thread` names the exception type and (in modern Java) often a helpful message.**
- **The first `at` line is where it was thrown** — that's the line to go look at first, not the bottom of the trace.
- Each subsequent `at` line is a caller, going outward until `main`.
- **`Caused by:`** marks a *chained* exception — the true root cause is usually the deepest `Caused by:` block; the exception above it was thrown while handling/wrapping that root cause.
- `... 3 more` means the remaining frames are identical to the frames already shown above in the enclosing trace (deduplicated for readability).

### 32.2 Common exceptions decoded

| Exception | Plain-English cause |
|---|---|
| `NullPointerException` (NPE) | Called a method or accessed a field on a reference that's `null`. **[Java 14+]** "Helpful NPE messages" are enabled by default from Java 14 onward and name the exact null variable/expression (`"...because the return value of Map.get(Object) is null"`), which used to just say `NullPointerException` with no detail pre-14. |
| `ArrayIndexOutOfBoundsException` | Indexed an array (`arr[i]`) with `i < 0` or `i >= arr.length`. |
| `StringIndexOutOfBoundsException` | Same idea but for `String.charAt(i)`/`substring` — index out of `[0, length)` (or invalid substring range). |
| `ClassCastException` | Cast an object to a type it isn't actually an instance of, e.g. casting an `Object` from a raw collection to the wrong type. |
| `NumberFormatException` | `Integer.parseInt("abc")` or similar — the string isn't a valid number (also thrown on empty string, leading/trailing whitespace not trimmed, or a value too large for the target type). |
| `ArithmeticException: / by zero` | Integer division or modulo by zero (`x / 0`, `x % 0`) — note this is **only** for integer types; floating-point division by zero produces `Infinity`/`NaN`, not an exception. |
| `ConcurrentModificationException` | Structurally modified a collection (add/remove) while iterating it with a for-each loop or an `Iterator` that doesn't own the modification — see §32.3. |
| `StackOverflowError` | Recursion too deep for the current thread's stack — see §30. Note this is an `Error`, not an `Exception` (extends `Throwable` directly via `Error`). |
| `OutOfMemoryError` | Allocated more heap than the JVM was given — check for accidental huge array sizes (e.g. reading `n` wrong and allocating `new int[n]` with a garbage `n`), or genuine memory limit issues. Also an `Error`, not an `Exception`. |
| `NoSuchElementException` | Called `poll()`/`remove()`/`next()` on an empty structure where you used the **throwing** variant instead of the **safe** one — e.g. `Deque.remove()` / `Deque.element()` throw when empty, whereas `poll()` / `peek()` return `null` instead. Also thrown by `Iterator.next()` past the end, and by `Scanner.nextInt()` when input is exhausted. |

### 32.3 Common-bug checklist for DSA-Java

| Bug | What goes wrong | Fix |
|---|---|---|
| `==` vs `.equals()` on `Integer` | `Integer` caches values in `[-128, 127]`; `==` "accidentally" works for small values then silently breaks for larger ones (e.g. `200 == 200` as boxed `Integer` can be `false`). | Always use `.equals()` for boxed types; `==` is only safe for primitive `int`/`long`. |
| `==` vs `.equals()` on `String` | `==` compares references, not content — two equal-content strings from different sources (`new String(...)`, `substring`, I/O-read tokens) are often **not** `==`. | Always use `.equals()` (or `Objects.equals()` if nullable) for string content comparison. |
| `list.remove(int)` overload | `List<Integer>.remove(5)` removes the element **at index 5**, not the value `5` — because there's both a `remove(int index)` and `remove(Object o)` overload, and an `int` argument always binds to the `int` overload. | To remove *by value* on a `List<Integer>`, box explicitly: `list.remove(Integer.valueOf(5))`. |
| PriorityQueue default order | `new PriorityQueue<>()` is a **min-heap** by natural ordering — the opposite of C++'s `priority_queue` which defaults to **max-heap**. | For a max-heap: `new PriorityQueue<>(Collections.reverseOrder())` or `new PriorityQueue<>((a,b) -> b - a)`. |
| `1 << 31` overflow | `1 << 31` in `int` arithmetic overflows into `Integer.MIN_VALUE` (sign bit), not `2^31` — bit tricks assuming wider range silently wrap. | Use `1L << 31` (a `long` shift) when you need the actual value `2147483648`. |
| Comparator subtraction overflow | `(a, b) -> a - b` as a comparator looks fine but overflows (and gives a wrong sign) when `a` and `b` are far apart, e.g. `a = Integer.MIN_VALUE`, `b = 1`. | Use `Integer.compare(a, b)` (or `Long.compare`) instead of manual subtraction. |
| Unboxing NPE from `map.get` | `int x = map.get(key);` throws NPE if `key` is absent (`get` returns `null`, unboxing `null` to `int` throws). | Use `map.getOrDefault(key, 0)`, or check `containsKey` first. |
| `int[]` as a map key | Arrays don't override `equals()`/`hashCode()` — two different `int[]` objects with identical contents are neither `.equals()` nor hash the same, so they're always distinct map keys. | Encode the key as a `String` (`Arrays.toString(arr)`), a `List<Integer>` (via `Arrays.asList` — which *does* have value equality), or a manually packed `long`/`String` composite key. |
| Modifying a list while iterating (for-each) | Adding/removing from a `List`/`Set`/`Map` during a for-each loop over it throws `ConcurrentModificationException`. | Use an explicit `Iterator` and call `iterator.remove()`, or collect changes and apply after the loop, or iterate over a copy. |
| Shadowing | A local variable or parameter reuses the same name as a field (e.g. constructor parameter `n` hiding a field `n`), silently making assignments target the wrong one. | Use `this.n = n;` in constructors, or name parameters distinctly (`nParam`, `_n`). |
| Forgetting `throws IOException` | Any method calling `BufferedReader.readLine()` or similar must either catch `IOException` or declare `throws IOException` — otherwise it's a compile error. | Add `throws IOException` to the method signature (simplest for CP; no need to catch). |
| `Scanner` too slow | Using `Scanner` for large input (10^5+ tokens) causes TLE purely from I/O overhead, even with an O(n) algorithm. | Switch to `BufferedReader` + `StringTokenizer` (see §31.6). |
| `Arrays.sort(int[])` anti-quicksort hack | See §31.1 — adversarial CF data can force O(n²). | Shuffle first, or sort a boxed `Integer[]` for TimSort. |
| Static field not reset between test cases | In a multi-test-case CF problem, a `static` field (e.g. a reused `visited[]`, an accumulator, a DSU parent array) retains its value from the previous test case if not explicitly reset in `solve()`. | Re-initialize every `static` mutable field at the top of `solve()`, or size/allocate fresh inside `solve()` each call. |

```java
// static-field-not-reset example — a classic silent-wrong-answer bug
static boolean[] visited; // declared once, OUTSIDE solve()

static void solve() throws IOException {
    int n = in.nextInt();
    visited = new boolean[n]; // MUST re-allocate (or Arrays.fill(visited, false))
                              // every test case, else stale `true`s leak into the next case
    // ...
}
```

### 32.4 Compiling and running locally

```bash
javac Main.java        # compiles Main.java -> Main.class
java Main               # runs it, reading stdin as usual

# enable assertions (assert statements are OFF by default at runtime!)
java -ea Main

# raise the stack size locally (only works locally — judges ignore JVM flags you can't pass)
java -Xss64m Main

# combine flags as needed
java -ea -Xss64m -Xmx512m Main
```

> **Gotcha:** `assert` statements are **compiled in but disabled by default** at runtime unless you pass `-ea` (enable assertions) to `java`. A submission relying on `assert` for correctness (rather than just as a debugging aid) is a bug waiting to happen — asserts silently no-op in the default run mode, including (typically) on judges, so never depend on an `assert` to actually stop execution; use a real `if (...) throw ...` for anything that must always be checked.

```java
static void checkInvariant(int x) {
    assert x >= 0 : "x should never be negative, got " + x; // silently skipped without -ea
}
```

### 32.5 A tiny stress-testing harness

The classic CP workflow: write a slow-but-obviously-correct brute force, write the fast intended solution, generate random small inputs, run both, and diff — repeat until a mismatch surfaces a bug.

**Random input generator (`Gen.java`):**

```java
import java.util.*;

public class Gen {
    public static void main(String[] args) {
        long seed = Long.parseLong(args[0]); // pass a different seed each run
        Random rnd = new Random(seed);

        int n = 1 + rnd.nextInt(8); // small n to keep brute force fast
        System.out.println(n);
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n; i++) {
            sb.append(1 + rnd.nextInt(20)).append(' ');
        }
        System.out.println(sb.toString().trim());
    }
}
```

**Bash driver looping seeds, comparing `Brute` vs `Fast`:**

```bash
javac Gen.java Brute.java Fast.java

for i in $(seq 1 200); do
    java Gen $i > in.txt
    java Brute < in.txt > out_brute.txt
    java Fast  < in.txt > out_fast.txt
    if ! diff -q out_brute.txt out_fast.txt > /dev/null; then
        echo "MISMATCH on seed $i"
        cat in.txt
        echo "--- brute ---"; cat out_brute.txt
        echo "--- fast  ---"; cat out_fast.txt
        break
    fi
done
echo "done"
```

`Brute.java` and `Fast.java` are just two ordinary `class Solution`-style `Main` programs (one obviously-correct O(n²)/O(n³) approach, one the real submission logic) reading from stdin and writing to stdout in the same format, so `diff` can compare them directly.

> **Gotcha:** Keep `n` small in the generator (single digits to low tens) so the brute force finishes fast across hundreds of random seeds — the point is to find a *logic* bug via many small cases, not to performance-test.

### 32.6 The `-Xlint` flag

```bash
javac -Xlint Main.java          # all recommended warnings
javac -Xlint:unchecked Main.java # e.g. flags raw-type / unchecked generic casts
javac -Xlint:all Main.java       # everything, including deprecation, rawtypes, etc.
```

`-Xlint` surfaces compiler warnings that are silent by default — most usefully in CP, `unchecked` warnings when mixing raw types and generics (e.g. `new PriorityQueue[n]` — you can't create a generic array in Java, so people fall back to a raw `PriorityQueue[]` and get an unchecked warning) and `deprecation` warnings for old API usage. Not essential for a contest, but useful when a local test is misbehaving in a way that smells like a type-erasure issue.

```java
// classic Java generics limitation surfaced by -Xlint:unchecked
@SuppressWarnings("unchecked")
List<Integer>[] buckets = new List[n]; // generic array creation isn't allowed directly;
                                        // this raw-array-then-suppress pattern is the standard workaround
for (int i = 0; i < n; i++) buckets[i] = new ArrayList<>();
```

> **vs C++:** `std::vector<std::vector<int>> buckets(n);` needs no such workaround — C++ templates are monomorphized at compile time and don't suffer Java's type-erasure restriction on array creation. This generic-array limitation is purely a Java/JVM artifact (arrays in Java are reified/covariant, generics are erased, and the two don't mix safely), so `@SuppressWarnings("unchecked")` + raw array is the accepted idiom rather than a bug.

---

# PART VIII — QUICK REFERENCE AND RECIPES

## 33. The "How Do I...?" Recipe Index

### Sorting

| Task | Code |
|---|---|
| Sort `int[]` ascending | `Arrays.sort(a);` |
| Sort `int[]` descending | box it, then reverse comparator |
| Anti-hack shuffle before sort | shuffle first, *then* `Arrays.sort` |
| Sort 2D array by column `k` | `Arrays.sort(arr, (x,y) -> x[k]-y[k]);` |
| Sort `List<T>` by key | `list.sort(Comparator.comparingInt(T::getKey));` |
| Sort `List<T>` multi-key | `.thenComparing(...)` chain |
| Argsort (indices by value) | sort a boxed index array with a comparator that reads `a[i]` |
| Dedupe preserving order | round-trip through `LinkedHashSet` |
| Reverse an array | manual two-pointer swap |
| Reverse a `List` | `Collections.reverse(list);` |

```java
// Sort int[] descending — box, sort reverse, (optionally unbox back)
int[] a = {5, 1, 4, 2, 3};
Integer[] b = Arrays.stream(a).boxed().toArray(Integer[]::new);
Arrays.sort(b, Collections.reverseOrder());
// b is now {5,4,3,2,1}; unbox again if you need int[]:
int[] desc = Arrays.stream(b).mapToInt(Integer::intValue).toArray();

// Anti-hack shuffle — defeats adversarial worst-case input targeting
// Java's dual-pivot quicksort on Arrays.sort(int[])
Random rnd = new Random();
for (int i = a.length - 1; i > 0; i--) {
    int j = rnd.nextInt(i + 1);
    int t = a[i]; a[i] = a[j]; a[j] = t;
}
Arrays.sort(a);

// Sort 2D array by column 1, then by column 0 as tiebreak
int[][] arr = {{3,1},{1,2},{2,1}};
Arrays.sort(arr, (x, y) -> x[1] != y[1] ? Integer.compare(x[1], y[1])
                                        : Integer.compare(x[0], y[0]));

// Sort List<Person> by age asc, then name desc
list.sort(Comparator.comparingInt(Person::getAge)
                     .thenComparing(Person::getName, Comparator.reverseOrder()));

// Argsort: get indices 0..n-1 ordered by a[i]
int n = a.length;
Integer[] idx = IntStream.range(0, n).boxed().toArray(Integer[]::new);
Arrays.sort(idx, (i, j) -> Integer.compare(a[i], a[j]));

// Dedupe a List while preserving first-seen order
List<Integer> deduped = new ArrayList<>(new LinkedHashSet<>(original));

// Reverse a raw array (no built-in for primitives)
for (int i = 0, j = a.length - 1; i < j; i++, j--) {
    int t = a[i]; a[i] = a[j]; a[j] = t;
}

// Reverse a List
Collections.reverse(someList);
```

### Min / Max / Selection

| Task | Code |
|---|---|
| Max of `int[]` (stream) | `Arrays.stream(a).max().getAsInt();` |
| Max of `int[]` (loop — faster, no boxing) | manual `Math.max` accumulate |
| Min of `int[]` | `Arrays.stream(a).min().getAsInt();` |
| Index of max | linear scan tracking best index |
| Kth largest | bounded min-heap of size `k` |
| Sum without overflow | accumulate into `long` |

```java
int mxStream = Arrays.stream(a).max().getAsInt();          // simple, allocates a stream
int mxLoop = a[0];
for (int x : a) mxLoop = Math.max(mxLoop, x);               // preferred in hot paths

int maxIdx = 0;
for (int i = 1; i < a.length; i++) if (a[i] > a[maxIdx]) maxIdx = i;

// Kth largest via bounded min-heap — keep only the k biggest seen so far
int k = 3;
PriorityQueue<Integer> pq = new PriorityQueue<>();
for (int x : a) {
    pq.offer(x);
    if (pq.size() > k) pq.poll();      // evict the smallest of the current top-k
}
int kthLargest = pq.peek();

long sum = 0;                                                // int sum overflows past ~2.1e9
for (int x : a) sum += x;
```

### Maps / Frequency / Adjacency

| Task | Code |
|---|---|
| Frequency map (merge) | `map.merge(x, 1, Integer::sum);` |
| Frequency map (getOrDefault) | `map.put(x, map.getOrDefault(x,0)+1);` |
| Adjacency list build | `computeIfAbsent` |
| Iterate a map | `entrySet()`, not `keySet()`+`get()` |
| TreeMap floor / ceiling / higher / lower | 4 distinct nav methods, `null` if absent |
| Multiset via TreeMap | add / remove-one / min / max |

```java
Map<Integer, Integer> freq = new HashMap<>();
for (int x : a) freq.merge(x, 1, Integer::sum);              // preferred, one lookup
// equivalent, more verbose:
freq.put(x, freq.getOrDefault(x, 0) + 1);

// Adjacency list for a graph with n nodes, edges (u, v)
Map<Integer, List<Integer>> g = new HashMap<>();
g.computeIfAbsent(u, key -> new ArrayList<>()).add(v);
g.computeIfAbsent(v, key -> new ArrayList<>()).add(u);        // undirected: add both

// Iterate a map — entrySet avoids a second hash lookup per key
for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
    int key = e.getKey(), val = e.getValue();
}

TreeMap<Integer, Integer> tm = new TreeMap<>(freq);
Integer f = tm.floorKey(10);     // largest key <= 10, or null
Integer c = tm.ceilingKey(10);   // smallest key >= 10, or null
Integer h = tm.higherKey(10);    // smallest key > 10 (strict), or null
Integer l = tm.lowerKey(10);     // largest key < 10 (strict), or null

// Multiset via TreeMap<Integer,Integer> — counts with sorted-order access
TreeMap<Integer, Integer> ms = new TreeMap<>();
ms.merge(x, 1, Integer::sum);                                 // add one occurrence of x
ms.merge(x, -1, Integer::sum);
if (ms.get(x) == 0) ms.remove(x);                             // remove one occurrence
int msMin = ms.firstKey();
int msMax = ms.lastKey();
```

### Heaps

| Task | Code |
|---|---|
| Min-heap (default) | `new PriorityQueue<>();` |
| Max-heap | `new PriorityQueue<>(Collections.reverseOrder());` |
| PriorityQueue with comparator on objects | lambda comparator |
| Capacity hint | `new PriorityQueue<>(initialCapacity, cmp);` |

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();                       // C++ default is max — Java is min!
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Heap of int[] pairs {distance, node}, ordered by distance ascending (Dijkstra style)
PriorityQueue<int[]> pq = new PriorityQueue<>((x, y) -> Integer.compare(x[0], y[0]));
pq.offer(new int[]{0, sourceNode});

PriorityQueue<Integer> sized = new PriorityQueue<>(1024, Collections.reverseOrder());
```

### Arrays — Construction

| Task | Code |
|---|---|
| 2D array init | `new int[n][m]` — auto-zeroed |
| 2D array fill with value | loop `Arrays.fill` per row |
| 2D array of Lists | must instantiate each row |
| 3D array | `new int[a][b][c]` |
| dx/dy 4-direction | N,S,E,W arrays |
| dx/dy 8-direction | add diagonals |

```java
int[][] grid = new int[n][m];                    // every cell defaults to 0
for (int[] row : grid) Arrays.fill(row, -1);      // fill with sentinel value

@SuppressWarnings("unchecked")
List<Integer>[] adj = new List[n];
for (int i = 0; i < n; i++) adj[i] = new ArrayList<>();

int[][][] cube = new int[a][b][c];

int[] dx4 = {-1, 1, 0, 0}, dy4 = {0, 0, 1, -1};                       // N, S, E, W
int[] dx8 = {-1, -1, -1, 0, 0, 1, 1, 1};
int[] dy8 = {-1,  0,  1,-1, 1,-1, 0, 1};                              // + diagonals
```

### Strings

| Task | Code |
|---|---|
| String → int | `Integer.parseInt(s);` |
| int → String | `String.valueOf(x);` |
| String → Integer (boxed) | `Integer.valueOf(s);` |
| Split a string | `s.split(regex)` — regex trap! |
| Join a list into a string | `String.join` / stream+joining |
| char ↔ digit | `c - '0'` / `(char)('0'+d)` |
| Upper/lowercase | `.toUpperCase()` / `.toLowerCase()` |
| Reverse a string | via `StringBuilder` |
| Check palindrome | compare to its reverse |
| Substring | `[i, j)`, `j` exclusive |
| indexOf with -1 check | always check for `-1` |
| Build string efficiently | `StringBuilder`, never `+=` in a loop |

```java
int x = Integer.parseInt(s);
String s2 = String.valueOf(x);                 // or Integer.toString(x)
Integer boxed = Integer.valueOf(s);

// split() regex trap: '.', '|', '+', '*', '(', ')', '[' are regex metachars
String[] parts = "a.b.c".split("\\.");         // must escape '.' — split(".") gives []
String[] safe = "a|b|c".split(Pattern.quote("|"));
String[] ws = "a   b\tc".split("\\s+");        // collapse runs of whitespace
String[] keepEmpty = "a,,b,".split(",", -1);   // -1 keeps trailing empty tokens

String joined = String.join(",", listOfStrings);
String joinedInts = intList.stream().map(String::valueOf).collect(Collectors.joining(","));

char c = 'a' + 3;                              // NOT valid — char+int is int; cast needed:
char shifted = (char) ('a' + 3);
int digit = '7' - '0';                          // char to digit
char ch = (char) ('0' + digit);                  // digit to char

String up = s.toUpperCase(), lo = s.toLowerCase();

String reversed = new StringBuilder(s).reverse().toString();
boolean isPalindrome = s.equals(new StringBuilder(s).reverse().toString());

String sub = s.substring(2, 5);                 // chars at index 2,3,4 — end exclusive

int p = s.indexOf('x');
if (p == -1) { /* not found — ALWAYS check before using p */ }

StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) sb.append(a[i]).append(' ');   // O(1) amortized per append
String result = sb.toString();
// NEVER: String r = ""; for (...) r += x;   // O(n^2) total — reallocates every time

char[] chars = s.toCharArray();                 // faster than repeated charAt() in hot loops
```

### Combinatorics / Recursion

```java
// All permutations — in-place swap-based recursion
void permute(int[] a, int k) {
    if (k == a.length) { /* use a */ return; }
    for (int i = k; i < a.length; i++) {
        swap(a, k, i);
        permute(a, k + 1);
        swap(a, k, i);                          // backtrack
    }
}

// All subsets via bitmask — n must be small (<= ~20)
for (int mask = 0; mask < (1 << n); mask++) {
    List<Integer> subset = new ArrayList<>();
    for (int i = 0; i < n; i++)
        if ((mask & (1 << i)) != 0) subset.add(a[i]);
    // use subset
}

// All subsets via include/exclude recursion
void subsets(int i, List<Integer> cur, int[] a) {
    if (i == a.length) { /* use cur */ return; }
    subsets(i + 1, cur, a);                     // exclude a[i]
    cur.add(a[i]);
    subsets(i + 1, cur, a);                     // include a[i]
    cur.remove(cur.size() - 1);                  // backtrack
}
```

### Binary Search

| Task | Code |
|---|---|
| Insertion point via `Arrays.binarySearch` | `p>=0 ? p : -p-1` |
| Hand-rolled lowerBound | first idx with `a[idx] >= x` |
| Hand-rolled upperBound | first idx with `a[idx] > x` |
| Count values in `[l, r]` | `upperBound(r) - lowerBound(l)` |

```java
// Arrays.binarySearch: negative return encodes -(insertion point) - 1
int p = Arrays.binarySearch(a, x);              // a must already be sorted
int insertAt = (p >= 0) ? p : -p - 1;

int lowerBound(int[] a, int x) {                // first index with a[idx] >= x
    int lo = 0, hi = a.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;            // avoid (lo+hi)/2 overflow
        if (a[mid] < x) lo = mid + 1; else hi = mid;
    }
    return lo;
}

int upperBound(int[] a, int x) {                // first index with a[idx] > x
    int lo = 0, hi = a.length;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] <= x) lo = mid + 1; else hi = mid;
    }
    return lo;
}

int countInRange(int[] a, int l, int r) {       // a sorted; count of values in [l, r]
    return upperBound(a, r) - lowerBound(a, l);
}
```

### Math

| Task | Code |
|---|---|
| gcd | recursive Euclid |
| lcm | divide before multiply |
| Fast power (modpow) | iterative binary exponentiation |
| Modular add | `((a%m)+(b%m))%m` |
| Modular mul (avoid overflow) | cast to `long` first |
| Safe positive modulo | `Math.floorMod` |
| Integer sqrt (with correction) | `Math.sqrt` then nudge |
| Check power of two | `(x>0)&&(x&(x-1))==0` |
| Count set bits | `Integer.bitCount` |
| Iterate submasks | `sub=(sub-1)&mask` |
| Swap in array | manual temp var |
| Clamp | nested `Math.max`/`Math.min` |
| Random int | `Random`/`ThreadLocalRandom` |

```java
long gcd(long a, long b) { return b == 0 ? a : gcd(b, a % b); }
long lcm(long a, long b) { return a / gcd(a, b) * b; }          // divide first — avoid overflow

long modpow(long base, long exp, long mod) {
    long result = 1;
    base %= mod;
    while (exp > 0) {
        if ((exp & 1) == 1) result = result * base % mod;
        base = base * base % mod;
        exp >>= 1;
    }
    return result;
}

long modAdd = ((a % m) + (b % m)) % m;
long modMul = (long) a * b % m;                  // cast BEFORE multiplying — else int overflow

int safeMod = Math.floorMod(x, m);               // handles negative x correctly, unlike x % m

long isqrt(long x) {
    long s = (long) Math.sqrt(x);
    while (s * s > x) s--;
    while ((s + 1) * (s + 1) <= x) s++;
    return s;
}

boolean isPowerOfTwo = (x > 0) && (x & (x - 1)) == 0;
int bits = Integer.bitCount(x);                  // Long.bitCount(x) for long

for (int sub = mask; sub > 0; sub = (sub - 1) & mask) {
    // use sub — this enumerates all nonempty submasks of mask
}

int t = a[i]; a[i] = a[j]; a[j] = t;              // no std::swap equivalent for primitives

int clamped = Math.max(lo, Math.min(hi, x));

int r1 = new Random().nextInt(bound);             // [0, bound)
int r2 = ThreadLocalRandom.current().nextInt(bound);
```

### I/O and Misc

```java
long t0 = System.nanoTime();
/* work */
long elapsedMs = (System.nanoTime() - t0) / 1_000_000;

BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
StringTokenizer st = new StringTokenizer(br.readLine());
int n = Integer.parseInt(st.nextToken());
int[] arr = new int[n];
st = new StringTokenizer(br.readLine());
for (int i = 0; i < n; i++) arr[i] = Integer.parseInt(st.nextToken());

char[][] grid = new char[n][];
for (int i = 0; i < n; i++) grid[i] = br.readLine().toCharArray();

System.out.printf("%.2f%n", 3.14159);            // fixed precision

System.out.println(Arrays.toString(arr));        // 1D
System.out.println(Arrays.deepToString(grid2d));  // 2D / nested arrays

int[] fromList = list.stream().mapToInt(Integer::intValue).toArray();
List<Integer> fromArray = Arrays.stream(arr).boxed().collect(Collectors.toList());
// NOTE: Arrays.asList(arr) on a primitive int[] does NOT give List<Integer> — see §36 #14
```

---

## 34. Master Complexity Table

### Data structure operations

| Structure | Insert | Remove | Lookup / Contains | Access by index | Iterate | Notes / when it degrades |
|---|---|---|---|---|---|---|
| `int[]` (array) | O(n) — fixed size, no true insert | O(n) shift | O(n) linear scan | **O(1)** | O(n) | Size fixed at creation; "insert" means rebuild |
| `ArrayList` | O(1) amortized at end, O(n) middle/front | O(n) middle/front, O(1) last | O(n) `contains`/`indexOf` | **O(1)** | O(n) | Doubling resize → amortized O(1) `add` |
| `LinkedList` | O(1) at a known node or either end | O(1) at a known node or either end | O(n) | **O(n)** | O(n) | Never index into it in a loop — that's O(n²) total |
| `String` | — immutable, no in-place insert | — immutable | O(n) `contains`/`indexOf` | **O(1)** `charAt` | O(n) | Every concatenation allocates a new object |
| `StringBuilder` | O(1) amortized `append` | O(n) `deleteCharAt`/`delete` in middle | O(n) `indexOf` | **O(1)** `charAt` | O(n) | The correct tool for loop-built strings |
| `ArrayDeque` | **O(1)** both ends (`addFirst`/`addLast`) | **O(1)** both ends | O(n) `contains` | N/A (no index access) | O(n) | Preferred over `Stack` and `LinkedList` for stack/queue use |
| `HashMap` | **O(1)** average | **O(1)** average | **O(1)** average | N/A | O(n + capacity) | O(n) worst case under hash collisions/adversarial keys |
| `TreeMap` | O(log n) | O(log n) | O(log n) | N/A | O(n), sorted key order | Also provides floor/ceiling/higher/lower/first/last |
| `LinkedHashMap` | O(1) average | O(1) average | O(1) average | N/A | O(n), insertion (or access) order | Same hash cost as `HashMap` + linked-list bookkeeping |
| `HashSet` | O(1) average | O(1) average | O(1) average | N/A | O(n + capacity) | Backed internally by a `HashMap` |
| `TreeSet` | O(log n) | O(log n) | O(log n) | N/A | O(n), sorted | Backed internally by a `TreeMap` |
| `PriorityQueue` | O(log n) `offer` | O(log n) `poll`; **O(n)** arbitrary `remove(Object)` | O(1) `peek` | N/A | O(n), **not** sorted order | Iteration order is heap-array order, not priority order |
| `BitSet` | O(1) amortized `set` | O(1) `clear` | O(1) `get` | N/A | O(n/64) via `nextSetBit` | ~64x more compact than `boolean[]`; word-parallel bulk ops |

### Algorithm complexities

| Operation | Complexity | Notes |
|---|---|---|
| `Arrays.sort(int[])` (primitive overload) | O(n log n) average, **O(n²) worst case** | Dual-pivot quicksort; deterministic pivot choice is exploitable — shuffle first on adversarial judges |
| `Arrays.sort(Object[])` / `Collections.sort(List)` | **O(n log n) guaranteed** | TimSort — stable, no quadratic worst case |
| `Arrays.binarySearch` | O(log n) | Requires the array to already be sorted |
| `TreeMap`/`TreeSet` navigation (`floor`/`ceiling`/`higher`/`lower`/`first`/`last`) | O(log n) each | Backed by a red-black tree |
| `PriorityQueue.offer` / `.poll` | O(log n) | `.peek` is O(1) |
| `PriorityQueue.remove(Object)` | O(n) | Linear scan to locate, then O(log n) to restore heap property |
| `BigInteger.add` / `.subtract` | O(d), d = digit count | |
| `BigInteger.multiply` | O(d log d) for large operands (Java switches algorithms), O(d²) schoolbook for small | Fine for typical CP-sized big integers |
| `BigInteger.modPow` | O(log e) modular multiplications | Efficient enough for crypto-scale exponents |

**Amortized vs worst-case gotcha**: `ArrayList.add` and `HashMap` operations are amortized O(1) — a single call can occasionally be O(n) (resize/rehash), but the cost is spread over many calls. This is fine for total runtime but can matter for strict per-call time limits (rare in CP, common in real-time systems).

---

## 35. n → Required Complexity Cheat Sheet

Rule of thumb: modern judges run roughly **10⁸ simple operations/sec** as a baseline (typically calibrated for C++). **Java tends to run ~1.5–2x slower** than C++ for the same algorithm, due to JVM overhead — bounds checking on every array access, object headers, garbage collection pauses, and JIT warmup before hot loops reach native-speed compiled code. In practice this rarely matters: judges like Codeforces, AtCoder, and LeetCode typically grant Java **2–3x the C++ time limit** specifically to compensate. Don't over-optimize preemptively — but when you're already near the ceiling for your `n`, the highest-leverage fix is avoiding boxed collections (`ArrayList<Integer>`, `Integer[]`) in the hot path; use primitive arrays (`int[]`, `long[]`) instead, since autoboxing plus poor cache locality is where Java actually loses time versus C++.

| n ≤ | Required complexity | Technique | Cross-ref (pattern #) |
|---|---|---|---|
| ~10 | O(n!) or O(2ⁿ · n) | Brute-force permutations, full recursive search tree | 01 — Recursion / Backtracking |
| ~20 | O(2ⁿ) | Bitmask DP, subset enumeration | 02 — Bitmask DP |
| ~100 | O(n³) – O(n⁴) | Triple/quadruple nested loops, Floyd–Warshall all-pairs shortest path | 05 — Graph (All-Pairs Shortest Path) |
| ~500 | O(n³) | 3D DP tables, matrix-chain-style algorithms | 03 — Interval / 2D DP |
| ~5,000 | O(n²) | Nested-loop DP, pairwise comparisons, simple string DP | 03 — DP, 07 — Two Pointers (quadratic variants) |
| ~10⁵ | O(n log n) | Sorting, heap-based algorithms, binary search, balanced BST ops, segment tree build/query | 09 — Sorting/Searching, 14 — Segment Tree |
| ~10⁶ | O(n) or O(n log log n) | Single linear pass, prefix sums, sieve of Eratosthenes, one-pass DP | 06 — Prefix Sums, 20 — Sieve / Number Theory |
| ~10⁹ | O(log n), O(√n), or O(1) closed-form | Binary search on the answer, direct math formulas, fast exponentiation, gcd-based approaches | 21 — Math / Modular Arithmetic, 09 — Binary Search |

Sanity check before writing a single line: read `n` off the constraints first and pick the ceiling row. If `n ≤ 10⁵` and your planned approach is O(n²), that's ~10¹⁰ operations — too slow even with a generous Java time multiplier. Re-derive the approach before coding, not after the TLE verdict.

---

## 36. The Top 30 Java-DSA Mistakes That Cost You Problems

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | `==` on boxed `Integer`/`String` instead of `.equals()` | Silent wrong answer | Always `.equals()` for object/wrapper comparison |
| 2 | Integer cache (`-128..127`) makes `==` "work" for small values | Passes small tests, WA on larger inputs — false confidence | Never rely on cache behavior; it's an implementation detail, not a guarantee |
| 3 | `list.remove(int)` removes by **index** for `List<Integer>`, not by value | Silent wrong answer — wrong element gone | `list.remove(Integer.valueOf(x))` to remove by value |
| 4 | `PriorityQueue<>()` defaults to **min-heap** (C++ `priority_queue` defaults to max-heap) | Silent wrong answer — elements pop in reverse priority | `new PriorityQueue<>(Collections.reverseOrder())` for max-heap |
| 5 | Comparator `(a,b) -> a - b` overflows on extreme `int` values | Silent wrong answer / inconsistent ordering | `Integer.compare(a, b)` instead of subtraction |
| 6 | `1 << 31` / `1 << k` for large `k` overflows as `int` | Silent wrong answer — wrong mask/bit value | `1L << k`, use `long` when `k` can reach 31+ |
| 7 | Unboxing a `null` from `map.get(x)` (key absent) | **NPE** at runtime | `map.getOrDefault(x, 0)` or `containsKey` check first |
| 8 | `int[]` used as a `HashMap`/`HashSet` key — identity, not content, equality | Silent wrong answer — every array "different" even with same contents | Wrap as `List<Integer>`, or encode into a `String`/packed `long` key |
| 9 | `Arrays.sort(int[])` (quicksort) hit with adversarial input | **TLE** — known Codeforces anti-Java-quicksort test pattern | Shuffle the array before sorting |
| 10 | `HashMap` under adversarial hash collisions | TLE (rarer, but real on some judges) | Use `TreeMap`, or salt/randomize the hash, for untrusted input |
| 11 | `Scanner` used for large input | **TLE** — regex-based tokenizing is very slow | `BufferedReader` + `StringTokenizer` for anything nontrivial |
| 12 | `System.out.println` inside a tight loop | **TLE** — effectively unbuffered per-call cost | Accumulate into a `StringBuilder`, print once at the end |
| 13 | String `+` concatenation inside a loop | **TLE** — O(n²) total (reallocates + copies each time) | `StringBuilder.append` |
| 14 | `Arrays.asList(intArray)` on a **primitive** `int[]` | Silent wrong answer — yields `List<int[]>` of size 1, not `List<Integer>` | Box first: `Arrays.stream(intArray).boxed().collect(Collectors.toList())` |
| 15 | Modifying a collection while iterating it with for-each | **ConcurrentModificationException** | `Iterator.remove()`, iterate a copy, or `removeIf` |
| 16 | `Math.abs(Integer.MIN_VALUE)` | Silent wrong answer — still returns `Integer.MIN_VALUE` (negative!), overflow | Cast to `long` first, or special-case `MIN_VALUE` |
| 17 | `(int) Math.sqrt(x)` for exact integer roots | Silent wrong answer — floating point can be off by one | Nudge-correct with a small `while` loop after |
| 18 | `Math.pow` used for integer exponentiation | Silent wrong answer — returns `double`, loses precision for large integers | Write an integer binary-exponentiation loop |
| 19 | `%` on negative operands vs mathematical mod | Silent wrong answer — Java's `%` can return a negative result | `Math.floorMod(x, m)` when a non-negative result is required |
| 20 | `(lo + hi) / 2` in binary search | Wrong answer / rare overflow for very large bounds | `lo + (hi - lo) / 2` |
| 21 | Forgetting `throws IOException` on `main`/I/O methods | Compile error | `public static void main(String[] args) throws IOException` |
| 22 | Deep recursive DFS without adjusting stack size | **StackOverflowError** — default thread stack (~512KB–1MB) is smaller than typical CP recursion needs | Run in a `new Thread(..., biggerStackSize)`, or convert to explicit-stack iterative DFS |
| 23 | `static` fields not reset between test cases in one run | Silent wrong answer from the 2nd test case onward — stale state leaks | Reset all mutable `static` state per test case, or avoid `static` for per-test data |
| 24 | Confusing `.length` (array field) / `.length()` (String method) / `.size()` (Collection method) | Compile error | Memorize which construct uses which |
| 25 | `String.split(regex)` surprises — metacharacters, dropped trailing empties | Silent wrong answer — wrong token count/content | Escape metachars (`Pattern.quote`/`\\`); `split(regex, -1)` to keep trailing empties |
| 26 | Forgetting to `flush()` a `PrintWriter`/`BufferedWriter` | Silent wrong answer / truncated output — buffer never reaches the judge | Explicit `.flush()` (or close the writer) before the program exits |
| 27 | Public class not named `Main` (Codeforces-style judges require it) | **Compile error** | Name the public class `Main` per the judge's convention |
| 28 | Assuming a local variable auto-defaults like a field does | **Compile error** — Java requires definite assignment for locals before use | Explicitly initialize locals before use |
| 29 | Autoboxing in hot loops (`Integer` arithmetic, `List<Integer>` instead of `int[]`) | TLE at the margin — boxing/unboxing overhead compounds at 10⁶+ iterations | Use primitive arrays in performance-critical inner loops |
| 30 | Using `LinkedList` expecting fast random access | TLE — O(n) per indexed access, O(n²) if indexed repeatedly in a loop | `ArrayList` for indexed access; `ArrayDeque`/`LinkedList` only for sequential/deque ops |

```java
// #4 illustrated — the classic C++ → Java porting bug
PriorityQueue<Integer> pq = new PriorityQueue<>();     // MIN-heap in Java!
// if you meant "process largest first" (typical C++ habit), you need:
PriorityQueue<Integer> pqMax = new PriorityQueue<>(Collections.reverseOrder());

// #7 illustrated — silent NPE from unboxing
Map<String, Integer> counts = new HashMap<>();
int c = counts.get("missing");        // NPE: null auto-unboxes to int and throws
int safe = counts.getOrDefault("missing", 0);   // correct

// #14 illustrated — the Arrays.asList trap
int[] primArr = {1, 2, 3};
List<int[]> wrong = Arrays.asList(primArr);      // size 1 !! one element: the whole array
Integer[] boxedArr = {1, 2, 3};
List<Integer> right = Arrays.asList(boxedArr);   // size 3, as expected — but fixed-size (no add/remove)
```

---

## 37. 60-Second Warm-Up Drill

Run through this block mentally (or literally paste-compile it) before a contest — it exercises the ~20 constructs you'll actually type under pressure.

```java
import java.util.*;
import java.io.*;
import java.util.stream.*;

public class Main {
    // record as a pair — value class, auto equals/hashCode/toString  [Java 16]
    record Pair(int val, int idx) {}

    public static void main(String[] args) throws IOException {
        // 1. Fast reader: read n, then an int array
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        int n = Integer.parseInt(br.readLine().trim());
        StringTokenizer st = new StringTokenizer(br.readLine());
        int[] a = new int[n];
        for (int i = 0; i < n; i++) a[i] = Integer.parseInt(st.nextToken());

        // 2. Anti-hack shuffle then sort — defeats adversarial Arrays.sort(int[]) TLE
        Random rnd = new Random();
        for (int i = n - 1; i > 0; i--) {
            int j = rnd.nextInt(i + 1);
            int t = a[i]; a[i] = a[j]; a[j] = t;
        }
        Arrays.sort(a);

        // 3. HashMap frequency count via merge
        Map<Integer, Integer> freq = new HashMap<>();
        for (int x : a) freq.merge(x, 1, Integer::sum);

        // 4. TreeMap floor/ceiling — nearest value queries
        TreeMap<Integer, Integer> tm = new TreeMap<>(freq);
        Integer floorKey = tm.floorKey(a[n / 2]);     // <= target
        Integer ceilKey  = tm.ceilingKey(a[n / 2]);   // >= target

        // 5. Min-heap PriorityQueue — default is min-heap in Java (not max like C++)
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        for (int x : a) minHeap.offer(x);

        // 6. StringBuilder for output — never string-concat in a loop
        StringBuilder sb = new StringBuilder();
        sb.append("sorted: ").append(Arrays.toString(a)).append('\n');

        // 7. Bitmask subset enumeration
        int m = Math.min(n, 20);
        long subsetCount = 0;
        for (int mask = 0; mask < (1 << m); mask++) subsetCount++;
        sb.append("subsets: ").append(subsetCount).append('\n');

        // 8. Hand-rolled binary search (lowerBound: first idx with a[idx] >= target)
        int target = a[n / 2];
        int lo = 0, hi = n;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] < target) lo = mid + 1; else hi = mid;
        }
        sb.append("lowerBound(").append(target).append(") = ").append(lo).append('\n');

        // 9. 2D grid BFS with ArrayDeque + direction arrays
        int rows = 3, cols = 3;
        int[][] grid = new int[rows][cols]; // 0 = open
        int[] dx = {-1, 1, 0, 0};
        int[] dy = {0, 0, 1, -1};
        boolean[][] visited = new boolean[rows][cols];
        ArrayDeque<Pair> queue = new ArrayDeque<>();
        queue.offer(new Pair(0, 0));
        visited[0][0] = true;
        int reachable = 0;
        while (!queue.isEmpty()) {
            Pair cur = queue.poll();
            reachable++;
            for (int d = 0; d < 4; d++) {
                int nr = cur.val() + dx[d];   // reusing Pair.val()/idx() as (row, col)
                int nc = cur.idx() + dy[d];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                        && !visited[nr][nc] && grid[nr][nc] == 0) {
                    visited[nr][nc] = true;
                    queue.offer(new Pair(nr, nc));
                }
            }
        }
        sb.append("bfs reachable: ").append(reachable).append('\n');

        // 10. Print everything at once — buffered, flushed on close
        System.out.print(sb);
    }
}
```

### 10-item self-check

| If you can't write this cold... | Re-read |
|---|---|
| Fast I/O (`BufferedReader` + `StringTokenizer`) without looking it up | Part VI §27-28 (I/O) |
| Anti-hack shuffle before `Arrays.sort(int[])` | §36 mistake #9, §33 Sorting |
| `HashMap.merge` for frequency counting | §33 Maps / Part III §12 |
| `TreeMap` floor/ceiling/higher/lower | §33 Maps / Part III §12 |
| Min-heap vs max-heap `PriorityQueue` construction | §36 mistake #4, §33 Heaps |
| `StringBuilder` for building output | Part II §9, §36 mistake #13 |
| Bitmask subset enumeration syntax | §33 Combinatorics, Part V §22-24 |
| Hand-rolled `lowerBound`/`upperBound` binary search | §33 Binary Search |
| `ArrayDeque` as a BFS queue + dx/dy direction arrays | Part III §14, §33 Arrays-construction |
| `record` for a lightweight pair/tuple type | Part I §5 |

### C++ → Java mental switch (the 8 things to remember)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. PriorityQueue defaults to MIN-heap   (C++ priority_queue = MAX)   │
│ 2. Compare objects with .equals()        (== compares references)   │
│ 3. No unsigned types — use >>> for logical right shift, not >>      │
│ 4. Bit masks with shift >= 31 need 1L    (int literals overflow)    │
│ 5. Arrays.sort(Object[], cmp) needs BOXED arrays for a Comparator   │
│ 6. No operator overloading — no +, ==, < on custom/BigInteger types │
│    (use .add(), .equals(), .compareTo())                            │
│ 7. Scanner is slow — use BufferedReader+StringTokenizer for CP      │
│ 8. Codeforces wants public class Main     (not your usual filename) │
└─────────────────────────────────────────────────────────────────────┘
```

---

*Java Complete Reference — companion to DSA Patterns 01–38 and the C++ Reference (doc 39).*
*When you forget something and it is not in here, add it. This document should only grow.*
