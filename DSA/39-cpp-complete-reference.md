# C++ — THE COMPLETE DSA REVISION REFERENCE
## Every Function, Every Idiom, Every Trap — In One Document
### Baseline: C++17 | Modern additions marked **[C++20]** / **[C++23]**
### Companion to Patterns 01–38 | Interview + Codeforces Ready

---

## HOW TO USE THIS DOCUMENT

You already know C++. The problem is that after months away from DSA, the *recall* goes
— you know a thing exists, you just can't produce it under a timer. This document exists so
that **you never have to stop mid-problem to go learn something.**

Three ways to read it:

| Situation | Where to go |
|---|---|
| **Coming back after a long break** — want to reload the language | Read **§37** (60-Second Warm-Up Drill) first, then skim Parts I–IV |
| **Mid-problem, forgot the exact syntax** — "how do I sort by second element again?" | Jump straight to **§33** (The "How Do I...?" Recipe Index) |
| **Debugging something that makes no sense** | **§36** (Top 30 C++ Mistakes) and **§32** (Debugging, UB and Compiling) |

**The rule:** if you catch yourself thinking *"I've seen this construct before but I don't
remember how it works"* — that thing should be in here. If it isn't, add it.

### Conventions used throughout

- Code is **C++17** unless tagged. **[C++20]** / **[C++23]** tags include a note on judge availability.
- `// =>` comments show expected output.
- `> **Gotcha:**` blockquotes flag traps that cause real WA/TLE/RE, not theoretical concerns.
- Complexity is given for every container operation and algorithm.

---

## TABLE OF CONTENTS

### PART I — CORE LANGUAGE
1. Types, Literals and Limits
2. References and Pointers
3. Functions
4. Structs and Classes
5. Lambdas and Callables
6. Templates and Type Deduction
7. Move Semantics and Value Categories

### PART II — STL CONTAINERS (SEQUENCE)
8. vector — The Workhorse
9. string
10. array, deque, list, forward_list
11. pair, tuple and Structured Bindings

### PART III — STL CONTAINERS (ASSOCIATIVE + ADAPTERS)
12. map and unordered_map
13. set, unordered_set, multiset
14. Container Adapters: stack, queue, priority_queue
15. bitset
16. Choosing the Right Container

### PART IV — ALGORITHMS AND ITERATORS
17. Sorting
18. Binary Search Family
19. The Rest of `<algorithm>` and `<numeric>`
20. Iterators
21. Comparators — The Complete Mental Model

### PART V — BIT MANIPULATION
22. Bitwise Operators and Fundamentals
23. The Complete Bit-Trick Catalog
24. Compiler Intrinsics, `<bit>`, and Bit Containers

### PART VI — NUMERICS, I/O AND THE MATH TOOLKIT
25. Integer Arithmetic, Overflow and Safety
26. Floating Point
27. Input and Output
28. Randomness, Time and Utility Headers

### PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT
29. The Contest Template and Macros
30. PBDS — The Hidden Ordered Set
31. Performance, Hacking and Anti-Test Defenses
32. Debugging, UB and Compiling

### PART VIII — QUICK REFERENCE AND RECIPES
33. The "How Do I...?" Recipe Index ← **the mid-problem lookup**
34. Master Complexity Table
35. n → Required Complexity Cheat Sheet
36. The Top 30 C++ Mistakes That Cost You Problems
37. 60-Second Warm-Up Drill ← **start here after a break**

---

# PART I — CORE LANGUAGE

## 1. Types, Literals and Limits

### 1.1 Fundamental types and sizes

Sizes below are for typical 64-bit judges/compilers (GCC on Linux, x86-64). Not guaranteed by the standard, but reliable in practice on Codeforces/LeetCode/local GCC.

| Type | Typical size | Range (approx) |
|---|---|---|
| `bool` | 1 byte | `true`/`false` |
| `char` | 1 byte | -128..127 (signedness of plain `char` is impl-defined) |
| `short` | 2 bytes | ±3.2×10^4 |
| `int` | 4 bytes | ±2.1×10^9 (~-2^31..2^31-1) |
| `unsigned int` | 4 bytes | 0..4.29×10^9 |
| `long` | 8 bytes on Linux/macOS, 4 on Windows (MSVC) | avoid, use `long long` |
| `long long` | 8 bytes | ±9.2×10^18 (~-2^63..2^63-1) |
| `unsigned long long` | 8 bytes | 0..1.8×10^19 |
| `float` | 4 bytes | ~7 decimal digits precision |
| `double` | 8 bytes | ~15-16 decimal digits precision |
| `long double` | 8/16 bytes (impl-defined) | rarely needed in DSA |

### 1.2 `int` vs `long long` vs `unsigned` — the #1 DSA bug

**10^9 fits in `int`, but 10^9 * 10^9 does NOT.** `int` max is ~2.147×10^9. Any product, sum-of-many-terms, or prefix sum that can exceed ~2×10^9 must be `long long`.

```cpp
int a = 1000000000; // 1e9, fits in int
int b = 1000000000;
int c = a * b;       // OVERFLOW: UB, wraps to garbage
// => c is some negative/garbage number, NOT 1e18

long long c2 = 1LL * a * b; // correct: promote BEFORE multiplying
// => 1000000000000000000
```

> **Gotcha:** `long long c3 = a * b;` is STILL WRONG. The multiplication `a * b` happens in `int` first (both operands are `int`), overflows, and only the (already garbage) result is converted to `long long`. You must cast/promote an operand *before* the operation: `1LL * a * b` or `(long long)a * b`.

Common overflow triggers in DSA: sum of array of up to 10^5 elements each up to 10^9 (max sum ~10^14 → needs `long long`), `n*(n+1)/2` for n up to 10^9, adjacency matrix / DP table indices multiplied together, `mid = (lo+hi)/2` when `lo+hi` overflows (use `lo + (hi-lo)/2` instead).

```cpp
int lo = 1, hi = INT_MAX - 1;
int mid = (lo + hi) / 2;       // lo+hi can overflow int
int mid2 = lo + (hi - lo) / 2; // safe, always use this form
```

### 1.3 Fixed-width types

`<cstdint>` gives exact-width types — useful when size matters (bit tricks, hashing, packing).

```cpp
#include <cstdint>
int8_t   a;  // exactly 1 byte, signed   [-128, 127]
uint8_t  b;  // exactly 1 byte, unsigned [0, 255]
int32_t  c;  // exactly 4 bytes signed  (same as int usually)
int64_t  d;  // exactly 8 bytes signed  (same as long long usually)
uint64_t e;  // exactly 8 bytes unsigned
```

In competitive programming, `int` and `long long` are used almost exclusively over `int32_t`/`int64_t` (same width, less typing), but `int64_t`/`uint64_t` show up in bit-manipulation-heavy problems for clarity.

### 1.4 `INT_MAX` / limits

```cpp
#include <climits>   // INT_MAX, INT_MIN, LLONG_MAX, LLONG_MIN, CHAR_BIT
#include <limits>    // std::numeric_limits<T>

int a = INT_MAX;              // 2147483647
int b = INT_MIN;              // -2147483648
long long c = LLONG_MAX;      // 9223372036854775807
long long d = LLONG_MIN;

// Preferred in generic/templated code:
auto e = std::numeric_limits<int>::max();
auto f = std::numeric_limits<long long>::max();
double inf = std::numeric_limits<double>::infinity();
```

Common DSA idiom — initialize a "no answer yet" sentinel:
```cpp
long long best = LLONG_MIN;      // for maximization
long long worst = LLONG_MAX;     // for minimization
// or, avoiding overflow-on-add risk with sentinels:
const long long INF = 4e18;      // safely below LLONG_MAX, room to add
```

> **Gotcha:** Using `LLONG_MAX` directly as a "infinity" and then adding to it (`dist[u] + w` where `dist[u] == LLONG_MAX`) overflows and silently wraps to a negative number, corrupting Dijkstra/Bellman-Ford. Use a large-but-safe sentinel like `1e18` or `4e18` (well under `9.2e18`), or explicitly check `if (dist[u] == INF) continue;` before adding.

### 1.5 `size_t` and the signed/unsigned comparison trap

`v.size()` returns `size_t`, which is **unsigned** (usually `unsigned long` / 8 bytes on 64-bit). This is the classic infinite-loop / wraparound bug.

```cpp
vector<int> v; // empty
for (size_t i = 0; i < v.size() - 1; i++) {
    // v.size() is 0 (unsigned), 0 - 1 wraps to SIZE_MAX (~1.8e19)
    // loop condition i < SIZE_MAX is basically always true
    // => this either loops "forever" or crashes on v[i] out-of-bounds
}
```

```cpp
// Safe patterns:
for (size_t i = 0; i + 1 < v.size(); i++) { /* ok: no subtraction from size() */ }
for (int i = 0; i < (int)v.size() - 1; i++) { /* ok: cast to signed first */ }
if (!v.empty() && v.size() - 1 > 0) { /* guard first */ }
```

> **Gotcha:** Comparing signed and unsigned directly is dangerous: `int i = -1; if (i < v.size())` — `i` gets converted to unsigned, becomes huge, so `-1 < v.size()` is **false**. This is a very common source of "why does my loop not run / run forever" bugs. Prefer looping with a plain signed `int`/`long long` index and casting `v.size()` to signed when comparing, especially when the index can be negative or derived from subtraction.

```cpp
int i = -1;
if (i < (int)v.size()) { /* correct comparison, both signed */ }
```

### 1.6 char / bool

```cpp
char c = 'A';
int code = c;              // implicit promotion, 'A' -> 65
bool flag = true;
int x = flag;              // true -> 1, false -> 0
cout << (int)'a' << "\n";  // => 97
cout << (char)(97 + 1) << "\n"; // => b

// char arithmetic for alphabet indexing (very common in DSA)
string s = "hello";
int idx = s[0] - 'a';      // => 7 ('h' - 'a')
vector<int> freq(26, 0);
for (char ch : s) freq[ch - 'a']++;
```

> **Gotcha:** plain `char` may be signed or unsigned depending on platform. Doing `int((unsigned char)c)` is the safe way to convert a `char` that might hold a byte ≥128 (e.g. reading raw bytes) without getting a negative int.

### 1.7 Literals

| Literal | Meaning |
|---|---|
| `1e9` | `double`, 1000000000.0 |
| `1e9L` | `long double` |
| `1LL` | `long long` literal, forces `long long` arithmetic |
| `1ULL` | `unsigned long long` literal |
| `0x2A` | hex, = 42 |
| `0b101010` | binary **[C++14]** — GCC/Clang support as extension even in C++11, but standard since C++14; fine on virtually every modern judge |
| `1'000'000'000` | digit separator (C++14), same as `1000000000`, purely for readability |
| `042` | **octal** (leading zero) = 34 decimal — easy accidental trap |

```cpp
long long big = 1e9;        // 1e9 is a double; converting to long long is fine here (exact)
long long huge = 1e18;      // also fine, 1e18 is exactly representable as double... but risky near limits
long long safe = 1'000'000'000'000'000'000LL; // prefer integer literal for exact big constants

int oops = 010; // = 8 (octal!), NOT ten — classic gotcha with zero-padded literals
```

> **Gotcha:** `1e18` as a `double` literal loses precision beyond ~15-17 significant digits; comparing it for exact equality with a computed `long long` can fail. For exact big integers, always use an integer literal with `LL`/`ULL` suffix, not scientific notation.

### 1.8 `auto` and when it bites

```cpp
auto x = 5;        // int
auto y = 5.0;       // double
auto& r = v[0];     // reference to element, good — avoids copy
auto v2 = v;        // COPIES the whole container (v2 is vector<int>, a full copy)
for (auto x : v) {} // copies each element — fine for int, expensive for vector<vector<int>>
for (auto& x : v) {} // reference, no copy — prefer for large elements
for (const auto& x : v) {} // reference, read-only — safest default
```

> **Gotcha:** `auto` deduced from `vector<bool>`'s `operator[]` is **not** `bool&` — it's a proxy object (`vector<bool>::reference`) because `vector<bool>` is bit-packed. `auto& b = vec_bool[0];` can dangle or behave unexpectedly. Prefer `bool b = vec_bool[0];` (by value) when reading, and avoid taking references into a `vector<bool>`.

```cpp
vector<bool> vb = {true, false};
auto b = vb[0];       // b is a proxy, not bool — mostly works but avoid relying on its type
bool b2 = vb[0];      // safe, explicit bool
```

### 1.9 `const` vs `constexpr`

```cpp
const int n = 5;             // value fixed at runtime-or-compile-time, cannot be reassigned
constexpr int m = 5;         // MUST be computable at compile time
int arr1[n];                 // ok in practice with most compilers if n is const and initialized with a literal
int arr2[m];                 // guaranteed ok — m is a true compile-time constant

constexpr int square(int x) { return x * x; } // can run at compile time
int arr3[square(4)];          // arr3[16], valid — square(4) evaluated at compile time
```

Use `constexpr` for array bounds, template arguments, and anything that must be known at compile time (e.g. `bitset<constexprValue>`). Use plain `const` for read-only runtime values (e.g. a `const int n = cin >> n_input;`-derived bound — which actually can't be constexpr since it's runtime input).

### 1.10 Casting

```cpp
double d = 7.9;
int i1 = (int)d;              // C-style cast, => 7 (truncates toward zero)
int i2 = static_cast<int>(d); // preferred in C++, same result, => 7, but compiler-checked

int a = 7, b = 2;
double res = (double)a / b;   // => 3.5, cast BEFORE division to get floating result
double wrong = a / b;         // => 2.0 (integer division 7/2=3, THEN converted to double 3.0... 
                               //    actually a/b is int division = 3, then widened to 3.0)
```

> **Gotcha:** `int / int` is **integer division** (truncates toward zero in C++11+), even if the result is assigned to a `double`. `double res = a / b;` computes `a/b` in `int` first, THEN converts — it does NOT give you a floating-point division. You must cast an operand: `(double)a / b` or `1.0 * a / b`.

```cpp
cout << 7 / 2 << "\n";    // => 3
cout << -7 / 2 << "\n";   // => -3 (truncation toward zero, C++11+)
cout << 7 % 2 << "\n";    // => 1
cout << -7 % 2 << "\n";   // => -1 (sign follows dividend in C++11+)
cout << 7 % -2 << "\n";   // => 1
```

> **Gotcha (negative modulo):** C++'s `%` result has the same sign as the *dividend* (left operand), not always positive like in Python. `-7 % 3` is `-1` in C++ but `2` in Python. For a true mathematical mod (always non-negative), use `((x % m) + m) % m`.

```cpp
int x = -7, m = 3;
int true_mod = ((x % m) + m) % m; // => 2
```

Integer promotion: operands smaller than `int` (`char`, `short`, `bool`) are promoted to `int` before arithmetic. Mixed signed/unsigned ops promote the signed operand to unsigned (source of the `size_t` trap above).

---

## 2. References and Pointers

### 2.1 Reference vs pointer vs value

| | Value | Reference `T&` | Pointer `T*` |
|---|---|---|---|
| Can be null/unbound | no | no (must bind on init) | yes (`nullptr`) |
| Can be reseated | n/a | no (bound once) | yes (reassign) |
| Syntax to access | `x` | `x` (same as value) | `*p` or `p->member` |
| Copies on pass to function | yes | no | no (pointer itself copied, cheaply) |
| Needs `new`/arithmetic | no | no | can support pointer arithmetic |

```cpp
int a = 10;
int& ref = a;      // ref is an alias for a, must init immediately
int* ptr = &a;     // ptr holds address of a
ref = 20;          // a is now 20
*ptr = 30;         // a is now 30
cout << a << " " << ref << " " << *ptr << "\n"; // => 30 30 30
```

### 2.2 `T&` vs `const T&` vs `T&&`

```cpp
void byValue(vector<int> v);       // copies entire vector — O(n), avoid for large containers
void byRef(vector<int>& v);        // no copy, CAN modify caller's vector
void byConstRef(const vector<int>& v); // no copy, CANNOT modify — the default choice for read-only params
void byRvalueRef(vector<int>&& v); // binds to temporaries/moved-from objects, used for move semantics
```

**Rule of thumb for DSA function signatures:**

| Situation | Signature |
|---|---|
| Read-only, large object (vector, string, struct) | `const T&` |
| Need to modify caller's object in place | `T&` |
| Small trivial type (int, double, char, pair<int,int> even) | `T` (by value; cheap to copy, ref adds indirection overhead) |
| Taking ownership / sink parameter | `T` by value + `std::move` inside, or `T&&` |

```cpp
long long sum(const vector<int>& v) {   // no copy of v, read-only
    long long s = 0;
    for (int x : v) s += x;
    return s;
}
void doubleAll(vector<int>& v) {        // modifies caller's vector directly
    for (int& x : v) x *= 2;
}
```

> **Gotcha:** `vector<int> v` by value silently copies the whole vector every call — with recursion (e.g. passing a `vector<vector<int>>` grid through DFS by value) this turns an O(n) algorithm into O(n²) or worse and is a classic silent TLE. Always default to `const T&` for read access, `T&` for in-place mutation.

### 2.3 Pointer arithmetic

```cpp
int arr[5] = {10, 20, 30, 40, 50};
int* p = arr;        // arr decays to pointer to arr[0]
cout << *p << "\n";      // => 10
cout << *(p + 2) << "\n"; // => 30
p++;                  // now points to arr[1]
cout << *p << "\n";      // => 20
cout << p[2] << "\n";    // => 40 (pointer indexing == array indexing)
```

### 2.4 `nullptr`

```cpp
int* p = nullptr;         // preferred over NULL or 0 in modern C++
if (p == nullptr) { /* safe check */ }
if (!p) { /* equivalent, common idiom */ }
// TreeNode* root = nullptr; is the standard "empty tree" representation on LeetCode
```

> **Gotcha:** Dereferencing a `nullptr` (`*p` or `p->x`) is undefined behavior — usually a segfault, but not guaranteed to crash immediately, which can mask bugs in small test cases.

### 2.5 Dangling references / pointers

```cpp
int& bad() {
    int local = 42;
    return local;      // returning ref to a local — local is destroyed on return
}                       // UB: caller gets a reference to destroyed memory
// int& r = bad(); cout << r; // garbage or crash, compilers usually WARN about this
```

```cpp
vector<int> v = {1, 2, 3};
int& r = v[0];
v.push_back(4);   // may trigger reallocation if capacity exceeded
// r may now be DANGLING — it pointed into the old buffer, which was freed
cout << r << "\n"; // UB: could print garbage, or "work" by luck
```

> **Gotcha:** Any reference/pointer/iterator into a `vector` is invalidated by `push_back`, `insert`, `resize`, etc. if reallocation happens (or always for `insert`/`erase` at/before that point, regardless of reallocation). Never hold onto `&v[i]` or an iterator across a mutating call — re-fetch it, or use indices instead of references/iterators across mutations.

### 2.6 `*`, `->`, `.`

```cpp
struct Node { int val; Node* next; };
Node n{5, nullptr};
Node* p = &n;

cout << (*p).val << "\n";  // => 5, dereference then member access
cout << p->val << "\n";    // => 5, shorthand for (*p).val — ALWAYS use -> for pointers
cout << n.val << "\n";     // => 5, direct member access on the object itself (not a pointer)
```

### 2.7 Arrays decaying to pointers

```cpp
void printArr(int* arr, int n) {   // array parameter decays to pointer — size info is LOST
    // sizeof(arr) here is sizeof(int*) (8), NOT the array's byte size
    for (int i = 0; i < n; i++) cout << arr[i] << " ";
}
int a[5] = {1,2,3,4,5};
cout << sizeof(a) << "\n";       // => 20 (5 * 4 bytes), a is a real array here
printArr(a, 5);                  // must pass size explicitly, decays inside the function
```

> **Gotcha:** `sizeof(arr) / sizeof(arr[0])` to compute array length only works on the *original* array variable, never on a pointer parameter that received a decayed array — that gives `sizeof(int*)/sizeof(int)` = 2, not the real length.

### 2.8 `new` / `delete` — and why you rarely need them

```cpp
int* p = new int(5);   // heap-allocate a single int
delete p;               // must manually free, or you leak memory
int* arr = new int[10]; // heap array
delete[] arr;            // must use delete[] for arrays, not delete
```

In DSA/CP, prefer `vector`, `string`, stack-allocated arrays, or smart pointers over raw `new`/`delete` — manual memory management is a common source of leaks/double-frees and rarely buys anything except for building linked structures (`ListNode`, `TreeNode`) where LeetCode's API forces raw pointers.

> **Gotcha:** Mismatching `new`/`delete` vs `new[]`/`delete[]`, forgetting to `delete`, or double-`delete`-ing the same pointer are all UB. In a CP contest you generally never call `delete` at all (process exits and OS reclaims memory) — but in interview settings, be ready to explain ownership/cleanup semantics.

### 2.9 Smart pointers (brief)

```cpp
#include <memory>
unique_ptr<int> up = make_unique<int>(5); // sole owner, auto-freed at scope end, not copyable
shared_ptr<int> sp = make_shared<int>(5); // ref-counted, freed when last owner goes away, copyable

struct TreeNode {
    int val;
    unique_ptr<TreeNode> left, right; // ownership-clear tree — auto-cleans on destruction
    TreeNode(int v) : val(v) {}
};
```

In practice: rarely used in CP (raw pointers / indices into arrays are faster and simpler); occasionally asked about in interviews for "how would you design a tree/linked-list class safely." LeetCode's own `TreeNode`/`ListNode` definitions always use raw pointers — match that convention unless told otherwise.

### 2.10 Raw `TreeNode*` / `ListNode*` (LeetCode convention)

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

// classic reverse-linked-list using raw pointers, no smart pointers, no leaks-that-matter
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) {
        ListNode* nxt = head->next;
        head->next = prev;
        prev = head;
        head = nxt;
    }
    return prev;
}
```

---

## 3. Functions

### 3.1 Declaration vs definition

```cpp
int add(int a, int b);          // declaration (prototype) — goes in header or before use

int main() {
    cout << add(2, 3) << "\n";  // => 5, ok even though defined below (declared above)
}

int add(int a, int b) {          // definition
    return a + b;
}
```

### 3.2 Passing conventions

| Parameter type | Copies? | Can modify caller's object? | Use for |
|---|---|---|---|
| `T` (value) | yes | no | small/trivial types (`int`, `double`, `pair<int,int>`, small structs) |
| `T&` | no | yes | need in-place mutation of caller's object |
| `const T&` | no | no | large read-only objects (`vector`, `string`, `map`) — the DEFAULT for big params |
| `T*` | no (pointer copied) | yes (via deref) | optional param (can be `nullptr`), C-style APIs, linked structures |
| `T&&` | no | yes (moves from it) | sink parameters / move-optimized APIs |

```cpp
void modify(int& x) { x++; }               // caller's variable changes
void noModify(int x) { x++; }              // local copy only, caller unaffected
void bigReadOnly(const vector<int>& v) {}  // no copy, cannot mutate v
void bigMutate(vector<int>& v) { v.push_back(1); } // no copy, mutates caller's vector
```

### 3.3 Default arguments

```cpp
void greet(string name, string greeting = "Hello") {
    cout << greeting << ", " << name << "!\n";
}
greet("Alice");            // => Hello, Alice!
greet("Bob", "Hi");        // => Hi, Bob!
```

> **Gotcha:** Default arguments are resolved at compile time based on the static (declared) type/signature seen at the call site — they don't participate in virtual dispatch overrides. Rarely relevant in CP but bites in polymorphic interview designs.

### 3.4 Overloading

```cpp
int max2(int a, int b) { return a > b ? a : b; }
double max2(double a, double b) { return a > b ? a : b; }
long long max2(long long a, long long b) { return a > b ? a : b; }

max2(3, 5);        // calls int version
max2(3.0, 5.0);     // calls double version
max2(3LL, 5LL);     // calls long long version
```

> **Gotcha:** `max2(3, 5.0)` is ambiguous-ish or picks a promoted overload depending on what's available — mixing types in an overloaded call can silently pick a surprising overload. Prefer `std::max<long long>(a, b)` or explicit casts when types differ, rather than relying on overload resolution to "figure it out."

### 3.5 Return by value + RVO

```cpp
vector<int> buildVec(int n) {
    vector<int> result;
    for (int i = 0; i < n; i++) result.push_back(i * i);
    return result;   // Return Value Optimization / mandatory copy elision (C++17):
                      // no copy happens here, compiler constructs 'result' directly
                      // in the caller's storage. Safe & fast to return big containers.
}
vector<int> v = buildVec(1000000); // no expensive copy, guaranteed since C++17 for this pattern
```

Since C++17, this specific case (returning a prvalue / named local by value) is guaranteed copy-elided in most forms — safe to return large `vector`/`string` by value freely; it's the idiomatic style in CP.

### 3.6 `inline`

```cpp
inline int square(int x) { return x * x; } // hint to compiler to avoid call overhead;
// modern compilers inline small hot functions automatically regardless — rarely needed
// explicitly in CP; matters more to avoid ODR violations for functions defined in headers.
```

### 3.7 Recursion and stack depth

Each recursive call uses stack frame space (locals + saved registers + return address). Default stack size varies: Codeforces/judges often give 256MB **heap** but a much smaller **stack** (commonly 8MB on Linux by default, sometimes raised by the judge). A rough rule of thumb: naive recursive DFS with a handful of `int` locals can handle roughly **10^4–10^5** depth before stack overflow, though this varies heavily by frame size and judge configuration.

```cpp
// DFS on a skewed tree (e.g. a linked-list-shaped tree with 10^5 nodes) can stack-overflow:
void dfs(TreeNode* node) {
    if (!node) return;
    dfs(node->left);
    dfs(node->right);
}
// For a tree that degenerates into a chain of 10^5 nodes, this may crash (SIGSEGV / RE on judge).
```

> **Gotcha:** Recursive DFS on trees/graphs that can be a long chain (skewed BST, linked-list-like graph) is a classic silent stack overflow — it passes on random/balanced test data but crashes on adversarial deep/degenerate input. Fixes: convert to an explicit iterative stack-based DFS, or increase stack size if the judge allows it (not possible on LeetCode; sometimes possible locally via `ulimit -s` or a thread with a custom stack size in a CP contest environment).

```cpp
// Iterative DFS avoids recursion depth limits entirely:
void dfsIterative(TreeNode* root) {
    if (!root) return;
    stack<TreeNode*> st;
    st.push(root);
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        // process node
        if (node->right) st.push(node->right);
        if (node->left) st.push(node->left);
    }
}
```

### 3.8 `static` local variables — the multi-test-case trap

```cpp
int counter() {
    static int calls = 0;  // initialized ONCE, persists across all future calls
    calls++;
    return calls;
}
cout << counter() << " " << counter() << " " << counter() << "\n"; // => 1 2 3
```

> **Gotcha:** In CP problems with **multiple test cases** (`int T; cin >> T; while(T--) solve();`), a `static` local (or a global) inside/used by `solve()` that's meant to be "reset per test case" will silently carry state from the previous test case, producing wrong answers only on test 2+. Always explicitly reset such state at the top of `solve()`, or avoid `static`/globals for per-test-case data, or re-declare non-static locals each call (which naturally reinitialize).

```cpp
// BUGGY across multiple test cases:
void solve() {
    static vector<int> memo(100001, -1); // persists WRONG values from previous test case
    // ...
}
// FIX: either clear it each call...
void solve() {
    static vector<int> memo(100001, -1);
    fill(memo.begin(), memo.end(), -1); // explicit reset
    // ...
}
// ...or just don't make it static (re-declared and freshly initialized every call):
void solve() {
    vector<int> memo(100001, -1); // fresh every call, but re-allocates each time (minor cost)
}
```

### 3.9 Global variables in CP

Globals are extremely common in competitive programming for adjacency lists, `dist[]`/`visited[]` arrays, and memoization tables — mainly to avoid passing them through every recursive call, and because global arrays are zero-initialized automatically (unlike locals).

```cpp
const int MAXN = 200005;
vector<int> adj[MAXN];   // global adjacency list, zero-init'd (empty vectors) automatically
bool visited[MAXN];      // global bool array — zero-initialized to false automatically
int dist_[MAXN];         // global int array — zero-initialized to 0 automatically (NOT -1!)
```

> **Gotcha:** Global arrays are zero-initialized, which is convenient for `visited[]` (false = not visited) but a trap for `dist[]`/`parent[]` where you actually want `-1` or `INF` as the "unset" sentinel — zero looks like a valid value (e.g. distance 0, or node index 0) and causes subtle bugs. Explicitly `fill()`/loop-reset such arrays, and remember to reset them between multiple test cases in the same run (same trap as 3.8).

### 3.10 Forward declarations

```cpp
class B; // forward declaration — lets A reference B* / B& before B is fully defined

class A {
    B* bptr; // ok: pointer/reference to incomplete type is allowed
};

class B {
    A a; // now B can use A by value since A is fully defined above
};
```

Used to break circular dependencies between mutually-referencing classes (e.g. `Graph`/`Node` designs in interview questions).

### 3.11 Trailing return type

```cpp
auto add(int a, int b) -> int {   // equivalent to: int add(int a, int b)
    return a + b;
}
// Mainly useful when the return type depends on the parameters via decltype:
template<typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {
    return a * b;
}
multiply(3, 4.5); // => 13.5, return type deduced as double
```

In C++14+, `auto multiply(T a, U b) { return a * b; }` (no trailing type needed) usually suffices for function templates since return type deduction was generalized — trailing return type is mostly seen in older code or when the deduced type genuinely needs `decltype` on the parameters.

---

## 4. Structs and Classes

### 4.1 `struct` vs `class`

The **only** difference: default member access is `public` in `struct`, `private` in `class`. Everything else (inheritance, templates, virtual functions) is identical.

```cpp
struct S { int x; };       // x is public by default
class C { int x; };        // x is private by default

S s; s.x = 5;               // ok
C c; // c.x = 5;            // ERROR: x is private
```

Convention in DSA: use `struct` for simple data bundles (`Point`, `Edge`, `Node`) with public fields; use `class` when you want encapsulation (DSU, SegmentTree with hidden internals).

### 4.2 Member init lists

```cpp
struct Point {
    int x, y;
    Point(int x_, int y_) : x(x_), y(y_) {}  // init list — preferred
    // vs:
    // Point(int x_, int y_) { x = x_; y = y_; } // assignment in body — works but
    //   for const/reference members, or members needing non-default construction,
    //   this does NOT compile / is less efficient (default-construct then assign).
};

struct Edge {
    const int weight; // const member MUST use init list — cannot be assigned in body
    Edge(int w) : weight(w) {}
};
```

> **Gotcha:** Members are initialized in the **order they're declared in the class**, not the order they appear in the init list. If one member's init depends on another, list order doesn't help — declaration order rules, and mismatched order can trigger a compiler warning (`-Wreorder`) and possibly use an uninitialized value.

### 4.3 Constructors and destructor

```cpp
struct Box {
    int w, h;
    Box() : w(0), h(0) {}                 // default constructor
    Box(int w_, int h_) : w(w_), h(h_) {} // parameterized constructor
    Box(const Box& other) : w(other.w), h(other.h) {  // copy constructor
        cout << "copied\n";
    }
    ~Box() { /* cleanup, e.g. release resources — empty here, nothing to free */ }
};

Box b1;               // default ctor, w=0 h=0
Box b2(3, 4);          // parameterized
Box b3 = b2;           // copy ctor => prints "copied"
```

### 4.4 Rule of 0 / 3 / 5 (brief)

- **Rule of 0**: if your class only holds standard types (`vector`, `string`, `int`, smart pointers), don't write ANY of destructor/copy/move — the compiler-generated defaults are correct. This is the goal for almost all DSA code.
- **Rule of 3**: if you manually manage a resource (raw pointer via `new`), and you write ONE of {destructor, copy constructor, copy assignment}, you almost always need all three (else double-free / leak bugs).
- **Rule of 5**: same as Rule of 3, plus move constructor and move assignment operator, for performance with C++11-style move semantics.

```cpp
// Rule of 0 in action — this is all you need 99% of the time in DSA:
struct Graph {
    vector<vector<int>> adj; // vector manages its own memory correctly
    // no destructor, no copy/move ctors needed — defaults are correct and efficient
};
```

### 4.5 `this`

```cpp
struct Counter {
    int count;
    Counter(int count) { this->count = count; } // 'this->' disambiguates member from parameter
    Counter& increment() { count++; return *this; } // return *this enables chaining
};
Counter c(0);
c.increment().increment().increment();
cout << c.count << "\n"; // => 3
```

### 4.6 Access specifiers

```cpp
class BankAccount {
public:
    BankAccount(double balance) : balance_(balance) {}
    double getBalance() const { return balance_; } // const method: doesn't modify object
    void deposit(double amt) { balance_ += amt; }
private:
    double balance_; // hidden from outside — only accessible via public methods
protected:
    // accessible to this class and derived classes, not outside code
};
```

### 4.7 Nested structs

```cpp
struct Graph {
    struct Edge {
        int to, weight;
    };
    vector<vector<Edge>> adj;
};
Graph::Edge e{3, 10}; // access nested type via Graph::Edge
```

### 4.8 DSA-flavored examples

**A `Node` for a linked list / tree (see also section 2.10 for LeetCode's version):**
```cpp
struct Node {
    int val;
    Node* next;
    Node(int v) : val(v), next(nullptr) {}
};
```

**Disjoint Set Union (DSU / Union-Find) as a class:**
```cpp
class DSU {
public:
    DSU(int n) : parent(n), rank_(n, 0) {
        iota(parent.begin(), parent.end(), 0); // parent[i] = i
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]); // path compression
        return parent[x];
    }
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (rank_[a] < rank_[b]) swap(a, b);
        parent[b] = a;
        if (rank_[a] == rank_[b]) rank_[a]++;
        return true;
    }
private:
    vector<int> parent, rank_;
};

DSU dsu(5);
dsu.unite(0, 1);
cout << (dsu.find(0) == dsu.find(1)) << "\n"; // => 1 (true)
```

**A `Point` with `operator<` for sorting:**
```cpp
struct Point {
    int x, y;
    bool operator<(const Point& other) const {
        if (x != other.x) return x < other.x;
        return y < other.y; // tie-break on y
    }
};
vector<Point> pts = {{2,1}, {1,5}, {1,2}};
sort(pts.begin(), pts.end());
// => sorted by x then y: (1,2), (1,5), (2,1)
```

### 4.9 Operator overloading in depth

```cpp
struct Frac {
    long long num, den;

    bool operator<(const Frac& o) const {              // for sort()/set/map ordering
        return num * o.den < o.num * den;               // cross-multiply (careful: overflow risk with huge values)
    }
    bool operator==(const Frac& o) const {              // for == comparisons, unordered_set/map equality
        return num * o.den == o.num * den;
    }
    Frac operator+(const Frac& o) const {               // arithmetic operator, returns new object
        return {num * o.den + o.num * den, den * o.den};
    }
    friend ostream& operator<<(ostream& os, const Frac& f) { // debug printing with cout <<
        return os << f.num << "/" << f.den;
    }
};

Frac a{1, 2}, b{1, 3};
cout << (a + b) << "\n"; // => 5/6
cout << (a < b) << "\n"; // => 0 (false, since 1/2 > 1/3)
```

**`operator()` for functors (custom comparator objects):**
```cpp
struct CompareByAbs {
    bool operator()(int a, int b) const {
        return abs(a) < abs(b);
    }
};
vector<int> v = {-5, 3, -1, 4};
sort(v.begin(), v.end(), CompareByAbs()); // => {-1, 3, 4, -5}

// Functors are also used as custom hash/eq for unordered_map/set (see STL part),
// and as the comparator template argument for priority_queue:
priority_queue<int, vector<int>, greater<int>> minHeap; // greater<int> is a built-in functor
```

**`operator<<` for debug printing (very handy for quick `cerr` debugging):**
```cpp
struct Pair3 { int a, b, c; };
ostream& operator<<(ostream& os, const Pair3& p) {
    return os << "(" << p.a << "," << p.b << "," << p.c << ")";
}
Pair3 p{1,2,3};
cerr << p << "\n"; // => (1,2,3)   -- handy for debug output to stderr, doesn't affect judged stdout
```

**[C++20] spaceship operator `<=>`:** generates `<`, `<=`, `>`, `>=` (and often `==`) automatically from a single three-way comparison. Not available pre-C++20 — most CP judges (Codeforces, AtCoder) now support C++20 via GNU G++ 20+, but double-check the judge/interview environment before relying on it; **not usable if the environment is locked to C++17**.
```cpp
// [C++20] — only if your judge/compiler explicitly supports C++20
struct Point20 {
    int x, y;
    auto operator<=>(const Point20&) const = default; // auto-generates all 6 comparisons
};
```

### 4.10 Static members

```cpp
struct IdGenerator {
    static int nextId;                 // declared here, defined once outside the class
    int id;
    IdGenerator() : id(nextId++) {}    // shared counter across ALL instances
};
int IdGenerator::nextId = 0;           // required out-of-class definition (pre-C++17)
// (C++17 allows `static inline int nextId = 0;` directly inside the class, avoiding the
//  separate out-of-class definition line — convenient for header-only / single-file CP code)

IdGenerator a, b, c;
cout << a.id << " " << b.id << " " << c.id << "\n"; // => 0 1 2
```

### 4.11 `map` key vs `unordered_map` key — ordering vs hashing

```cpp
struct Point { int x, y; };

// map<Point, int> needs an ORDERING: operator< or a custom comparator
struct PointLess {
    bool operator()(const Point& a, const Point& b) const {
        return a.x != b.x ? a.x < b.x : a.y < b.y;
    }
};
map<Point, int, PointLess> orderedMap; // works: comparator defines a strict weak ordering

// unordered_map<Point, int> needs a HASH function + an equality function instead —
// there's no default hash for a user struct. This requires specializing std::hash<Point>
// or providing a custom hasher functor — covered in detail in the STL/containers part
// (hash combining, avoiding collisions, why pair<int,int> also lacks a default hash).
```

**Rule of thumb:** `map`/`set` need `operator<` (or a comparator) — easy to write inline as shown in 4.9. `unordered_map`/`unordered_set` need a hash function — more ceremony, deferred to the STL part of this doc.

---

## 5. Lambdas and Callables

### 5.1 Full syntax

```cpp
[capture](params) -> returnType { body }
//   ^ capture: what outer variables are visible inside
//              (params): like a normal function's parameter list
//   -> returnType: optional, usually deducible; required in some multi-return-statement cases
```

```cpp
auto add = [](int a, int b) -> int { return a + b; };
cout << add(2, 3) << "\n"; // => 5

auto add2 = [](int a, int b) { return a + b; }; // return type deduced as int, usually fine
```

### 5.2 Capture modes

| Capture | Meaning |
|---|---|
| `[]` | captures nothing — lambda can only use its own params/locals |
| `[=]` | captures ALL used outer variables **by value** (copy) |
| `[&]` | captures ALL used outer variables **by reference** |
| `[x]` | captures only `x`, by value |
| `[&x]` | captures only `x`, by reference |
| `[this]` | captures the enclosing object's `this` pointer (inside a member function) |
| `[x = expr]` | **[C++14]** init-capture: creates a new variable `x` initialized from `expr` |

```cpp
int threshold = 10;
vector<int> v = {5, 15, 8, 20};

auto byValue = [threshold](int x) { return x > threshold; };
auto byRef   = [&threshold](int x) { return x > threshold; };
auto all_ref = [&](int x) { return x > threshold; };       // captures threshold by ref implicitly

int count = count_if(v.begin(), v.end(), byValue); // => 2 (15, 20)

// [C++14] init-capture — great for moving a value INTO a lambda, or renaming:
auto lam = [y = threshold * 2](int x) { return x > y; }; // y = 20, independent of threshold now
```

> **Gotcha:** `[=]`/`[&]` "capture everything used" is convenient but can silently capture more than intended (e.g. accidentally capturing `this` inside a member function, or capturing a large object by value with `[=]`, causing a hidden copy). Prefer explicit captures (`[x, &y]`) in non-trivial code for clarity, though `[&]` is extremely common and idiomatic in short CP lambdas.

> **Gotcha:** A `[&]`-capturing lambda that outlives its enclosing scope (e.g. stored and called later, after the captured variables have gone out of scope) leads to dangling references — same danger as section 2.5. Fine for lambdas used and discarded immediately within the same scope (the overwhelming majority of CP usage), dangerous if the lambda escapes (stored in a `std::function` and called later from elsewhere).

### 5.3 Mutable lambdas

```cpp
int counter = 0;
auto inc = [counter]() mutable { // 'mutable' allows modifying the BY-VALUE captured copy
    counter++;
    return counter;
};
cout << inc() << " " << inc() << " " << inc() << "\n"; // => 1 2 3 (modifies lambda's own copy)
cout << counter << "\n"; // => 0 (outer variable is untouched — capture was by value)
```

Without `mutable`, a lambda's `operator()` is `const` by default, so it cannot modify by-value captures at all (compile error).

### 5.4 When return type must be explicit

```cpp
// Multiple return statements with different (or ambiguous) types need explicit -> returnType:
auto f = [](bool flag) -> double {
    if (flag) return 1;    // int
    else return 2.5;       // double — without explicit ->double, this may not compile/deduce cleanly
};
```

### 5.5 Recursive lambdas

A lambda can't naturally call itself by name (it has no name inside its own body during definition). Three common patterns:

**1. `std::function` (most common in CP, has some overhead):**
```cpp
#include <functional>
function<int(int)> fact = [&](int n) -> int {
    return n <= 1 ? 1 : n * fact(n - 1);
};
cout << fact(5) << "\n"; // => 120

// The idiom used constantly for DFS in CP:
vector<vector<int>> adj(5);
vector<bool> visited(5, false);
function<void(int)> dfs = [&](int u) {
    visited[u] = true;
    for (int v : adj[u]) if (!visited[v]) dfs(v);
};
dfs(0);
```

> **Gotcha (perf):** `std::function` incurs type-erasure overhead — an indirect call (roughly like a virtual call) and possibly a heap allocation for large captures. In tight-loop / deep-recursion DSA code (e.g. DFS over 10^5+ nodes called millions of times), this overhead is measurable. A named regular function (or a lambda captured by `auto&` via the "y-combinator"/self-parameter trick below) is faster.

**2. `auto self` trick (pass the lambda to itself as a parameter, C++14 generic lambda):**
```cpp
auto dfsImpl = [&](auto&& self, int u) -> void {
    visited[u] = true;
    for (int v : adj[u]) if (!visited[v]) self(self, v);
};
dfsImpl(dfsImpl, 0); // call by passing itself explicitly
```
Faster than `std::function` (no type erasure), but the call-site syntax is awkward (must pass itself every time).

**3. [C++23] Deducing `this`** — lets a lambda take itself as an explicit object parameter cleanly:
```cpp
// [C++23] — check availability; NOT available in C++17/20, very few judges support C++23 yet
auto dfsImpl = [&](this auto&& self, int u) -> void {
    visited[u] = true;
    for (int v : adj[u]) if (!visited[v]) self(v); // clean call, no need to pass self explicitly
};
dfsImpl(0);
```

**Practical recommendation for CP:** use `std::function<void(int)> dfs = [&](int u){...};` — the ergonomics win outweighs the perf cost for most problems; switch to a plain named recursive function (section 3.7) if DFS is the bottleneck and profiling shows `std::function` overhead matters.

### 5.6 `std::function`

```cpp
#include <functional>
function<int(int, int)> op;
op = [](int a, int b) { return a + b; };
cout << op(2, 3) << "\n"; // => 5
op = [](int a, int b) { return a * b; };
cout << op(2, 3) << "\n"; // => 6

// Storing a vector of callbacks/strategies:
vector<function<int(int)>> transforms = {
    [](int x) { return x + 1; },
    [](int x) { return x * 2; },
    [](int x) { return x * x; },
};
for (auto& f : transforms) cout << f(5) << " "; // => 6 10 25
```

### 5.7 Function pointers

```cpp
int square(int x) { return x * x; }
int (*fp)(int) = square;        // fp is a pointer to a function taking int, returning int
cout << fp(4) << "\n";           // => 16

int applyTwice(int (*f)(int), int x) { return f(f(x)); }
cout << applyTwice(square, 3) << "\n"; // => 81 (3^2=9, 9^2=81)
```

A capture-less lambda (`[](int x){...}`) implicitly converts to a function pointer of matching signature — but a lambda with captures does NOT (captures need storage a bare function pointer can't hold).

### 5.8 Functors / callable structs

```cpp
struct Adder {
    int base;
    Adder(int b) : base(b) {}
    int operator()(int x) const { return x + base; } // makes the struct "callable" like a function
};
Adder add5(5);
cout << add5(10) << "\n"; // => 15
// Functors can hold state cleanly and are used heavily as sort/heap comparators (section 4.9).
```

### 5.9 Passing comparators as lambdas

```cpp
vector<pair<int,int>> intervals = {{3,4},{1,2},{5,6}};
sort(intervals.begin(), intervals.end(),
     [](const pair<int,int>& a, const pair<int,int>& b) {
         return a.first < b.first; // sort by start of interval
     });
// => {{1,2},{3,4},{5,6}}

// Custom priority_queue with a lambda comparator needs decltype since lambda types are unique:
auto cmp = [](int a, int b) { return a > b; }; // min-heap behavior
priority_queue<int, vector<int>, decltype(cmp)> pq(cmp);
pq.push(5); pq.push(1); pq.push(3);
cout << pq.top() << "\n"; // => 1
```

---

## 6. Templates and Type Deduction

### 6.1 Function templates

```cpp
template<typename T>
T maxOf(T a, T b) {
    return a > b ? a : b;
}
cout << maxOf(3, 7) << "\n";       // => 7, T deduced as int
cout << maxOf(3.5, 2.1) << "\n";   // => 3.5, T deduced as double
cout << maxOf<double>(3, 7) << "\n"; // => 7, explicit T=double forces both args to double
```

### 6.2 Class templates

```cpp
template<typename T>
class Stack {
public:
    void push(T x) { data.push_back(x); }
    void pop() { data.pop_back(); }
    T top() const { return data.back(); }
    bool empty() const { return data.empty(); }
private:
    vector<T> data;
};
Stack<int> s;
s.push(5); s.push(10);
cout << s.top() << "\n"; // => 10
```

### 6.3 Template argument deduction

```cpp
template<typename T>
void printPair(pair<T,T> p) { cout << p.first << "," << p.second << "\n"; }

printPair(make_pair(1, 2));      // T deduced as int
// printPair(make_pair(1, 2.0)); // ERROR: can't deduce single T from (int, double) — mismatched types
```

### 6.4 `auto` deduction rules vs `decltype`

```cpp
int x = 5;
int& rx = x;
const int cx = 5;

auto a1 = x;   // int (auto strips references and top-level const)
auto a2 = rx;  // int (NOT int&, reference stripped)
auto a3 = cx;  // int (NOT const int, const stripped)
auto& a4 = cx; // const int& (explicit & preserves reference, const propagates)

decltype(x) d1;   // int — decltype preserves the EXACT declared type, no stripping
decltype(rx) d2 = x; // int& — reference preserved
decltype(cx) d3 = 5; // const int — const preserved
decltype((x)) d4 = x; // int& — note: decltype((expr)) with extra parens on an lvalue gives a reference!
```

> **Gotcha:** `decltype(x)` vs `decltype((x))` differ: the former gives the declared type of the variable, the latter (parenthesized expression) gives a reference type because a parenthesized variable name is an lvalue expression. Easy to trip over in generic/template code.

### 6.5 `decltype(auto)`

```cpp
int x = 5;
int& getRef() { return x; }

auto f1() { return getRef(); }             // return type 'auto' -> int (reference stripped, returns a copy)
decltype(auto) f2() { return getRef(); }   // return type preserves int& (reference kept)
```

Used when you need a wrapper function to perfectly forward the exact return type (reference-ness included) of another function — rare in DSA, more common in generic library code.

### 6.6 Structured bindings **[C++17]** — safe to use, this is baseline

```cpp
pair<int, string> p = {1, "one"};
auto [num, name] = p;
cout << num << " " << name << "\n"; // => 1 one

map<string, int> m = {{"a", 1}, {"b", 2}};
for (const auto& [key, value] : m) {   // extremely common idiom for iterating maps
    cout << key << "=" << value << " ";
}
// => a=1 b=2  (map iterates in sorted key order)

tuple<int, int, int> t = {1, 2, 3};
auto [a, b, c] = t;
cout << a + b + c << "\n"; // => 6

int arr2[2] = {10, 20};
auto [lo, hi] = arr2; // structured bindings also work on fixed-size arrays
```

> **Gotcha:** `auto [key, value]` in a range-for over a `map` COPIES each key/value pair by default (since `auto`, not `const auto&`). For large values or hot loops, use `const auto& [key, value] : m` to avoid copies, exactly like the `const T&` rule in section 2.2.

### 6.7 `using` vs `typedef`

```cpp
typedef unsigned long long ull;         // old-style, C-compatible
using ull2 = unsigned long long;        // modern, C++11+, reads left-to-right, preferred

typedef vector<vector<int>> Matrix;      // works for simple aliases
using Matrix2 = vector<vector<int>>;     // same effect, preferred style

// 'using' can alias TEMPLATES, which typedef cannot do at all:
template<typename T>
using Vec2D = vector<vector<T>>;
Vec2D<int> grid(3, vector<int>(3, 0)); // Vec2D<int> == vector<vector<int>>

using pii = pair<int,int>;    // extremely common CP shorthand
using vi  = vector<int>;
using vvi = vector<vector<int>>;
using ll  = long long;
```

### 6.8 Variadic templates (brief)

```cpp
template<typename T, typename... Args>
T sumAll(T first, Args... rest) {
    if constexpr (sizeof...(rest) == 0) return first;
    else return first + sumAll(rest...);
}
cout << sumAll(1, 2, 3, 4) << "\n"; // => 10
```

Enough variadic-template awareness to READ standard library code that uses it — you rarely write your own in CP. More practically relevant: understanding brace-init-list overloads like:

```cpp
cout << min({3, 1, 4, 1, 5}) << "\n"; // => 1, min/max accept an initializer_list<T>, NOT variadic args
cout << max({3, 1, 4, 1, 5}) << "\n"; // => 5
// min({...}) / max({...}) use an initializer_list<T> overload, not variadic templates —
// but they LOOK similar and are used just as often in CP for "min/max of several values."
```

### 6.9 `if constexpr` **[C++17]** — baseline, safe to use

```cpp
template<typename T>
void printValue(T x) {
    if constexpr (std::is_same_v<T, string>) {
        cout << "\"" << x << "\"\n";     // only compiled for T=string
    } else {
        cout << x << "\n";                // only compiled for other T
    }
}
printValue(5);        // => 5
printValue(string("hi")); // => "hi"
```

`if constexpr` discards the untaken branch entirely at compile time (unlike a regular `if`, both branches of which must still compile for all types) — essential for template code with genuinely type-dependent logic, like the variadic `sumAll` above.

### 6.10 When templates are worth it in CP

**Worth it:** a generic, reusable segment tree / Fenwick tree that works over `int`, `long long`, or a custom monoid struct (sum, min, max, gcd) — write once, instantiate as needed:
```cpp
template<typename T, typename Merge>
class SegTree {
public:
    SegTree(int n, T identity, Merge merge) : n(n), id(identity), merge(merge), tree(2*n, identity) {}
    void update(int i, T val) {
        i += n; tree[i] = val;
        for (i /= 2; i >= 1; i /= 2) tree[i] = merge(tree[2*i], tree[2*i+1]);
    }
    T query(int l, int r) { // [l, r)
        T resL = id, resR = id;
        for (l += n, r += n; l < r; l /= 2, r /= 2) {
            if (l & 1) resL = merge(resL, tree[l++]);
            if (r & 1) resR = merge(tree[--r], resR);
        }
        return merge(resL, resR);
    }
private:
    int n; T id; Merge merge; vector<T> tree;
};
auto sumTree = SegTree<long long, plus<long long>>(8, 0LL, plus<long long>());
```

**Overkill:** wrapping a single one-off DFS or a problem-specific DP in a template "for generality" — in a 30-90 minute contest, write the concrete typed version directly; templatizing single-use code costs debugging time for no payoff. Save templates for genuinely reusable data structures you'll paste across multiple problems (segment tree, DSU, sparse table).

---

## 7. Move Semantics and Value Categories

### 7.1 lvalue vs rvalue, in plain terms

| | lvalue | rvalue |
|---|---|---|
| Has a name / persistent address | yes | no (temporary) |
| Can appear on left of `=` | yes | generally no |
| Example | `x`, `v[0]`, `*p` | `5`, `x + 1`, `foo()` (returning by value), `std::move(x)` |

```cpp
int x = 5;        // x is an lvalue, 5 is an rvalue (used to initialize x)
int y = x + 1;     // x + 1 is an rvalue (a temporary)
int& lref = x;      // ok: lvalue reference binds to lvalue
// int& bad = 5;    // ERROR: lvalue reference cannot bind to an rvalue
const int& cref = 5; // ok: const lvalue reference CAN bind to an rvalue (extends its lifetime)
int&& rref = 5;      // ok: rvalue reference binds to an rvalue
```

### 7.2 `T&&` — rvalue reference

```cpp
void f(int& x)  { cout << "lvalue ref\n"; }
void f(int&& x) { cout << "rvalue ref\n"; }

int a = 5;
f(a);            // => lvalue ref (a is a named variable, an lvalue)
f(5);            // => rvalue ref (5 is a temporary)
f(std::move(a)); // => rvalue ref (move CASTS a to an rvalue reference, even though a is still an lvalue variable)
```

### 7.3 `std::move` — what it actually does

`std::move` does **NOT** move anything by itself. It's purely a **cast** — it converts its argument to an rvalue reference (`T&&`), which then makes rvalue-reference overloads (like move constructors) eligible to be selected by overload resolution. The actual "moving" (stealing internal pointers/buffers) happens inside the move constructor/assignment that gets called as a result.

```cpp
vector<int> a = {1, 2, 3, 4, 5};
vector<int> b = std::move(a);
// b now owns the internal buffer that 'a' used to own — O(1), no element copy.
// 'a' is left in a valid-but-UNSPECIFIED state: usually empty for vector/string,
// but you must NOT assume anything about its contents — only that it's safe to
// destroy or reassign.
cout << b.size() << "\n"; // => 5
cout << a.size() << "\n"; // => 0 (typical for vector, but not guaranteed by the standard)
```

> **Gotcha:** After `std::move(a)`, using `a` for anything other than reassigning it or letting it be destroyed is a bug waiting to happen — its state is unspecified (not necessarily empty, not necessarily unchanged). Never read from a moved-from object expecting its old value.

### 7.4 Copy vs move constructor

```cpp
struct Buffer {
    vector<int> data;
    Buffer(vector<int> d) : data(std::move(d)) {} // move the incoming param into the member
    // Copy ctor (compiler-generated default): deep-copies data — O(n)
    // Move ctor (compiler-generated default): steals data's internal pointer — O(1)
};

Buffer b1({1,2,3});             // vector temporary is moved into Buffer's ctor param, then moved into data
Buffer b2 = b1;                  // COPY ctor: b2.data is a full independent copy of b1.data
Buffer b3 = std::move(b1);       // MOVE ctor: b3.data steals b1.data's buffer, b1.data becomes empty
```

### 7.5 `push_back` vs `emplace_back`, copy vs move

```cpp
vector<string> v;
string s = "hello world, a somewhat long string";

v.push_back(s);              // COPIES s (s is an lvalue, push_back's lvalue-ref overload copies)
v.push_back(std::move(s));    // MOVES s's buffer into the vector — O(1), s is now unspecified/empty
v.push_back("literal");       // constructs a temporary string, then MOVES it in (temporary is an rvalue)
v.emplace_back("literal");    // constructs the string DIRECTLY in-place inside the vector —
                                // no temporary, no move at all — generally the most efficient option

struct Point3 { int x, y, z; };
vector<Point3> pts;
pts.push_back(Point3{1,2,3});      // constructs a temporary Point3, then move/copies it in
pts.emplace_back(1, 2, 3);          // constructs the Point3 directly in the vector's storage, no temp
```

> **Gotcha:** `emplace_back` isn't always strictly better — it forwards its arguments to the element's constructor, so `emplace_back(x)` where `x` is an *existing* object of the element type still copies `x` (it constructs the new element via the element's copy constructor from `x`) unless you also `std::move` it in: `emplace_back(std::move(x))`. The real win of `emplace_back` over `push_back` is avoiding the extra TEMPORARY when constructing a new object in place from raw arguments, as in the `Point3` example above.

### 7.6 `std::swap`

```cpp
int a = 1, b = 2;
std::swap(a, b); // => a=2, b=1
// Internally implemented via 3 moves (temp = move(a); a = move(b); b = move(temp);)
// for movable types — O(1) for vector/string, not a deep copy.
vector<int> v1 = {1,2,3}, v2 = {4,5,6,7};
swap(v1, v2); // O(1): just swaps internal pointers/sizes/capacities, no element-wise work
```

### 7.7 Copy elision / RVO — why returning big containers is fine

```cpp
vector<int> makeRange(int n) {
    vector<int> result;
    result.reserve(n);
    for (int i = 0; i < n; i++) result.push_back(i);
    return result; // Since C++17: for this pattern (returning a local by value), copy elision
                    // is GUARANTEED by the standard — no copy AND no move even happens;
                    // 'result' is constructed directly in the caller's destination storage.
}
vector<int> big = makeRange(1000000); // no O(n) copy, essentially free
```

This is why the idiomatic DSA style is "just return the vector/string by value" instead of taking an output parameter (`void makeRange(int n, vector<int>& out)`) — the latter is C-style and unnecessary in modern C++.

### 7.8 Where this actually matters for DSA performance

```cpp
// SLOW: passing/returning vector<vector<int>> BY VALUE inside recursion — real TLE cause
long long solve(vector<vector<int>> grid, int i, int j) { // COPIES the whole grid every call!
    // ... recursive calls to solve(grid, i+1, j) etc. each copy the grid again ...
    // For an NxN grid called recursively, this can blow up to O(N^2 * calls) — classic TLE.
}

// FAST: pass by const reference, no copies at all
long long solveFast(const vector<vector<int>>& grid, int i, int j) {
    // ... recursive calls to solveFast(grid, i+1, j) — grid itself is never copied ...
}
```

> **Gotcha:** This exact mistake — a recursive/backtracking function taking a large `vector`/`vector<vector<int>>`/`string` BY VALUE — is one of the most common real-world TLE causes in interviews and contests. It "works" on small examples and times out only on larger inputs, making it easy to miss in testing. Always default recursive helper parameters to `const T&` for anything bigger than a primitive (see section 2.2 / 3.2), and only pass by value when you specifically need a per-call independent copy (e.g. backtracking where you build up a `path` and must not have siblings see each other's mutations — even then, prefer mutate-then-undo on a single shared reference over copying).

```cpp
// Preferred backtracking idiom: mutate a SHARED vector/string, then undo — no copying at all
void backtrack(vector<int>& path, /* ... */) {
    path.push_back(choice);
    backtrack(path, /* ... */);
    path.pop_back(); // undo — 'path' is reused across all branches, never copied
}
```

---

# PART II — STL CONTAINERS (SEQUENCE)

## 8. vector — The Workhorse

### 8.1 Declaration and init forms

```cpp
#include <vector>
using namespace std;

vector<int> v1;                     // empty
vector<int> v2(5);                  // size 5, all 0
vector<int> v3(5, 7);               // size 5, all 7
vector<int> v4{1, 2, 3};            // init-list: {1,2,3}
vector<int> v5 = {1, 2, 3};         // same as above
vector<int> v6(v4);                 // copy of v4
vector<int> v7(v4.begin(), v4.end()); // from range/iterators
int arr[] = {9, 8, 7};
vector<int> v8(arr, arr + 3);       // from raw array
vector<int> v9(v4.begin()+1, v4.end()); // sub-range copy: {2,3}
// v2 => 0 0 0 0 0
// v3 => 7 7 7 7 7
// v4 => 1 2 3
```

> **Gotcha:** `vector<int> v(5)` is size 5 filled with zero, NOT an empty vector with capacity 5. Confusing this with `reserve` is a classic bug — `v.reserve(5)` keeps `size()==0`.

> **Gotcha:** `vector<int> v(10, 1)` vs `vector<int> v{10, 1}` — the first is ten 1's, the second is a 2-element vector `{10, 1}` (init-list constructor wins when braces are used). Always double check `()` vs `{}` when the two arguments could be `(count, value)`.

### 8.2 Complete method table

| Method | What it does | Complexity |
|---|---|---|
| `size()` | number of elements | O(1) |
| `empty()` | true if size==0 | O(1) |
| `push_back(x)` | append copy/move of x at end | amortized O(1) |
| `emplace_back(args...)` | construct element in-place at end (no temp copy) | amortized O(1) |
| `pop_back()` | remove last element (no return) | O(1) |
| `front()` | reference to first element | O(1) |
| `back()` | reference to last element | O(1) |
| `at(i)` | bounds-checked access, throws `std::out_of_range` | O(1) |
| `operator[](i)` | unchecked access, UB if out of range | O(1) |
| `begin()/end()` | forward iterators | O(1) |
| `rbegin()/rend()` | reverse iterators | O(1) |
| `clear()` | remove all elements, size→0 (capacity unchanged) | O(n) |
| `resize(n)` | grow/shrink to size n, new elems value-initialized | O(\|Δn\|) |
| `resize(n, val)` | grow/shrink, new elems = val | O(\|Δn\|) |
| `reserve(n)` | pre-allocate capacity ≥ n, size unchanged | O(n) (one-time) |
| `capacity()` | current allocated storage | O(1) |
| `shrink_to_fit()` | request capacity reduced to size (non-binding) | O(n) |
| `insert(it, x)` | insert x before it | O(n) |
| `insert(it, count, x)` | insert count copies before it | O(n) |
| `erase(it)` | remove element at it, return iterator to next | O(n) |
| `erase(first,last)` | remove range | O(n) |
| `assign(n, val)` | replace contents with n copies of val | O(n) |
| `assign(first,last)` | replace contents with a range | O(n) |
| `swap(other)` | swap contents (pointers), O(1) not O(n) | O(1) |
| `data()` | pointer to underlying contiguous array | O(1) |

```cpp
vector<int> v{1,2,3};
v.push_back(4);          // v = {1,2,3,4}
v.emplace_back(5);       // v = {1,2,3,4,5}  (no copy for POD types, matters for structs)
v.pop_back();            // v = {1,2,3,4}
cout << v.front() << " " << v.back() << "\n"; // => 1 4
cout << v.at(1) << "\n"; // => 2
try { v.at(100); } catch (const out_of_range& e) { cout << "caught\n"; } // => caught
v.resize(2);              // v = {1,2}
v.resize(4, 9);           // v = {1,2,9,9}
cout << v.size() << " " << v.capacity() << "\n"; // size=4, capacity>=4 (impl-defined)
v.clear();                 // v = {}, size 0
cout << v.empty() << "\n"; // => 1
```

### 8.3 Iteration

```cpp
vector<int> v{10, 20, 30};

for (size_t i = 0; i < v.size(); i++) cout << v[i] << " ";     // => 10 20 30
cout << "\n";

for (auto x : v) cout << x << " ";       // copies each element (fine for int)
cout << "\n";                             // => 10 20 30

for (auto& x : v) x *= 2;                 // reference: mutates original
for (auto x : v) cout << x << " ";        // => 20 40 60
cout << "\n";

for (const auto& x : v) cout << x << " "; // no copy, read-only — preferred for structs/strings
cout << "\n";                              // => 20 40 60
```

> **Gotcha:** `for (auto x : v)` copies each element. For `vector<string>` or `vector<vector<int>>` this is a silent O(n) or worse copy per iteration — use `const auto&` (read-only) or `auto&` (mutate) instead of `auto`.

### 8.4 2D and 3D vectors

```cpp
int n = 3, m = 4;
vector<vector<int>> g(n, vector<int>(m, 0));   // n x m grid, all 0
g[1][2] = 5;
for (auto& row : g) { for (int x : row) cout << x << " "; cout << "\n"; }
// => 0 0 0 0
// => 0 0 5 0
// => 0 0 0 0

// 3D: n x m x k cube
vector<vector<vector<int>>> cube(2, vector<vector<int>>(2, vector<int>(2, -1)));
cube[0][1][0] = 7;
```

`vector<vector<int>>` allocates n+1 separate heap blocks (one per row + the outer vector), so rows are NOT contiguous — poor cache locality vs a single flat buffer. For hot loops (DP grids, adjacency matrices), prefer a flat 1D vector with manual indexing:

```cpp
int n = 3, m = 4;
vector<int> flat(n * m, 0);           // single contiguous block
auto idx = [m](int i, int j) { return i * m + j; };
flat[idx(1, 2)] = 5;
cout << flat[idx(1, 2)] << "\n"; // => 5
```

### 8.5 Erase-remove idiom

`erase(single_iterator)` only removes one element and shifts the rest — it does NOT remove all matching values by itself. `remove` partitions matching elements to the end and returns the new logical end; you must then `erase` the tail.

```cpp
#include <algorithm>
vector<int> v{1, 2, 3, 2, 4, 2};
v.erase(remove(v.begin(), v.end(), 2), v.end());
for (int x : v) cout << x << " "; // => 1 3 4
cout << "\n";

// remove_if with a predicate
vector<int> v2{1, 2, 3, 4, 5, 6};
v2.erase(remove_if(v2.begin(), v2.end(), [](int x){ return x % 2 == 0; }), v2.end());
for (int x : v2) cout << x << " "; // => 1 3 5
cout << "\n";
```

**[C++20]** `std::erase`/`std::erase_if` — free functions in `<vector>`, available GCC 10+/Clang 12+/MSVC 19.28+ with `-std=c++20`:

```cpp
// C++20 only
// vector<int> v{1,2,3,2,4,2};
// erase(v, 2);                 // removes all 2's directly, no remove() needed
// erase_if(v, [](int x){ return x % 2 == 0; });
```

### 8.6 Erasing while iterating — correct pattern

```cpp
vector<int> v{1, 2, 3, 4, 5};
for (auto it = v.begin(); it != v.end(); ) {
    if (*it % 2 == 0) it = v.erase(it);   // erase returns iterator to the NEXT element
    else ++it;                             // only advance when NOT erasing
}
for (int x : v) cout << x << " "; // => 1 3 5
cout << "\n";
```

> **Gotcha:** `for (auto it = v.begin(); it != v.end(); ++it) if (cond) v.erase(it);` is a classic bug — after `erase`, the old `it` is invalidated, and `++it` on it is undefined behavior. Always capture erase's return value.

### 8.7 Iterator invalidation rules

| Operation | What gets invalidated |
|---|---|
| `push_back` / `emplace_back` (no realloc) | only `end()` iterator invalidated |
| `push_back` / `emplace_back` (causes realloc) | ALL iterators, pointers, references invalidated |
| `insert` | if realloc: everything; if no realloc: iterators from insertion point to end |
| `erase` | iterators/pointers/references at or after the erase point (the erased element and everything after) |
| `reserve` | if it changes capacity: everything; if capacity already sufficient: nothing |
| `clear` | all iterators except possibly `begin()==end()` |

```cpp
vector<int> v{1,2,3};
int* p = &v[0];
for (int i = 0; i < 100; i++) v.push_back(i);  // very likely reallocates at some point
// p is now a DANGLING pointer — using *p here is undefined behavior
```

> **Gotcha:** Never hold raw pointers/references/iterators into a vector across a `push_back`/`insert`/`resize` unless you've `reserve`'d enough capacity up front and know no realloc will happen. This is one of the most common real-world vector bugs (e.g., passing `&v[i]` into a function that then grows `v`).

### 8.8 reserve for performance

```cpp
vector<int> v;
v.reserve(1'000'000);           // one allocation up front
for (int i = 0; i < 1'000'000; i++) v.push_back(i);
// without reserve: repeated reallocations (~O(log n) reallocs, each copying/moving all elements)
// amortized push_back is still O(1) either way, but reserve avoids the constant-factor overhead
```

### 8.9 `vector<bool>` — the trap

`vector<bool>` is a specialization that packs bits (1 bit per bool, not 1 byte) — it is NOT a real container in the STL sense.

```cpp
vector<bool> vb{true, false, true};
// &vb[0];              // COMPILE ERROR or nonsense — no contiguous bool array exists
// vb[0] returns a PROXY object (std::vector<bool>::reference), not a real bool&
auto x = vb[0];          // x is a proxy/temporary, NOT a bool — surprising in generic code
bool b = vb[0];          // fine — implicit conversion to bool
for (bool v : vb) cout << v << " "; // => 1 0 1
cout << "\n";
```

> **Gotcha:** You cannot take `&vb[0]` or pass `vb.data()` to a C API expecting `bool*`. `for (auto& x : vb)` does NOT compile the way you'd expect (no real reference to bind) — use `for (bool x : vb)` or `for (auto x : vb)` instead.

**Fix:** if you need a packed bitset, use `vector<bool>` intentionally and accept the caveats; if you need normal container semantics (real references, contiguous memory, `&v[0]` for C APIs), use `vector<char>` or `deque<bool>` instead:

```cpp
vector<char> flags(10, 0);   // acts like a normal container, 1 byte per element
flags[3] = 1;
char* p = &flags[0];          // valid — vector<char> is a normal specialization
```

### 8.10 Comparing vectors

```cpp
vector<int> a{1, 2, 3}, b{1, 2, 4};
cout << (a == b) << "\n";  // => 0  (element-wise equality)
cout << (a < b) << "\n";   // => 1  (lexicographic: first differing element decides, 3 < 4)
vector<int> c{1, 2};
cout << (c < a) << "\n";   // => 1  (shorter prefix is "less" if all matching elements equal)
```

### 8.11 Passing vectors to functions

```cpp
void byValue(vector<int> v) { v.push_back(99); }        // copies entire vector — O(n), caller unaffected
void byRef(vector<int>& v) { v.push_back(99); }          // no copy, caller's vector mutated
void byConstRef(const vector<int>& v) { /* read only */ } // no copy, cannot mutate — preferred for read-only params

int main() {
    vector<int> v{1, 2, 3};
    byConstRef(v);          // pass by const& to avoid copy when just reading — standard DSA practice
    byRef(v);
    cout << v.size() << "\n"; // => 4
}
```

> **Gotcha:** Passing a large `vector<vector<int>>` by value into a recursive function (common in naive backtracking code) silently copies the whole grid on every call — pass by reference unless you specifically need an isolated copy.

---

## 9. string

### 9.1 Init forms

```cpp
#include <string>
using namespace std;

string s1;                       // empty
string s2("hello");              // from literal
string s3 = "hello";             // same
string s4(5, 'x');               // "xxxxx"
string s5(s2);                   // copy
string s6(s2, 1, 3);              // substring ctor: from index1, len3 => "ell"
string s7(s2.begin(), s2.end());  // from iterator range
// s6 => ell
```

### 9.2 Complete method table

| Method | What it does | Complexity |
|---|---|---|
| `size()` / `length()` | number of chars (identical) | O(1) |
| `empty()` | true if size==0 | O(1) |
| `push_back(c)` | append one char | amortized O(1) |
| `pop_back()` | remove last char | O(1) |
| `append(s)` / `+=` | append string/char | O(len appended), amortized |
| `substr(pos, len)` | copy of substring starting at pos, len chars (or to end) | O(len) |
| `find(s, pos=0)` | first index of s at/after pos, or `npos` | O(n·m) worst |
| `rfind(s, pos=npos)` | last index of s at/before pos, or `npos` | O(n·m) worst |
| `find_first_of(chars)` | first index of any char in `chars` | O(n) |
| `find_last_of(chars)` | last index of any char in `chars` | O(n) |
| `replace(pos, len, s)` | replace range with s | O(n) |
| `insert(pos, s)` | insert s at pos | O(n) |
| `erase(pos, len)` | remove len chars at pos | O(n) |
| `compare(s)` | 3-way lexicographic compare, <0/0/>0 | O(n) |
| `c_str()` | null-terminated `const char*` | O(1) |
| `front()` / `back()` | first / last char reference | O(1) |
| `at(i)` | bounds-checked access | O(1) |
| `resize(n)` / `resize(n,c)` | grow/shrink | O(\|Δn\|) |
| `clear()` | empty the string | O(n) |
| `starts_with(s)` **[C++20]** | prefix check | O(m) |
| `ends_with(s)` **[C++20]** | suffix check | O(m) |

```cpp
string s = "hello world";
cout << s.size() << "\n";              // => 11
s.push_back('!');                       // "hello world!"
s.pop_back();                           // "hello world"
s += " again";                          // "hello world again"
cout << s.substr(0, 5) << "\n";         // => hello
cout << s.substr(6) << "\n";            // => world again

size_t pos = s.find("world");
if (pos != string::npos) cout << "found at " << pos << "\n"; // => found at 6
cout << (s.find("xyz") == string::npos) << "\n"; // => 1

cout << s.find_first_of("aeiou") << "\n"; // => 1 (the 'e' in hello)

s.replace(0, 5, "HELLO");                // "HELLO world again"
cout << s << "\n";                        // => HELLO world again
s.erase(0, 6);                             // "world again"
cout << s << "\n";                          // => world again

string a = "abc", b = "abd";
cout << a.compare(b) << "\n";  // => negative (a < b)

cout << s.front() << " " << s.back() << "\n"; // => w n
```

> **Gotcha:** `find` returns `size_t` (unsigned). ALWAYS compare against `string::npos`, never `< 0` — `pos < 0` is always false for unsigned types and silently breaks the "not found" check.

### 9.3 Indexing and iteration

```cpp
string s = "abc";
for (size_t i = 0; i < s.size(); i++) cout << s[i]; // => abc
cout << "\n";
for (char c : s) cout << c;                          // => abc
cout << "\n";
for (char& c : s) c = toupper(c);                    // mutate in place
cout << s << "\n";                                   // => ABC
```

### 9.4 Char arithmetic

```cpp
#include <cctype>
char c = 'e';
cout << (c - 'a') << "\n";        // => 4  (0-indexed alphabet position)
int d = 7;
char digitChar = '0' + d;
cout << digitChar << "\n";        // => 7  (char '7')

cout << isalpha('a') << " " << isalpha('1') << "\n"; // => 1 0 (nonzero/0)
cout << isdigit('5') << " " << isdigit('x') << "\n"; // => 1 0
cout << (char)tolower('A') << (char)toupper('z') << "\n"; // => aZ
```

> **Gotcha:** `<cctype>` functions take/return `int` and expect values representable as `unsigned char` or EOF. Passing a plain (signed) `char` with a negative value (possible for extended/non-ASCII chars) is undefined behavior — cast to `unsigned char` first if input isn't guaranteed ASCII: `isalpha((unsigned char)c)`.

### 9.5 String ↔ number conversion

```cpp
int i = stoi("42");
long long ll = stoll("123456789012");
double d = stod("3.14");
string s = to_string(42);          // "42"
string s2 = to_string(3.14);       // "3.140000" (default 6 decimal digits)

cout << i << " " << ll << " " << d << " " << s << "\n"; // => 42 123456789012 3.14 42

try { stoi("abc"); } catch (const invalid_argument& e) { cout << "invalid\n"; } // => invalid
try { stoi("99999999999999999999"); } catch (const out_of_range& e) { cout << "overflow\n"; } // => overflow

cout << stoi("  42abc") << "\n";  // => 42  (leading whitespace skipped, trailing garbage ignored)
```

> **Gotcha:** `stoi` stops parsing at the first non-digit character and does NOT error if there's trailing garbage — `stoi("42abc")` returns `42` silently. Only a fully non-numeric prefix throws `invalid_argument`; a too-large numeric value throws `out_of_range`.

### 9.6 Building strings efficiently

```cpp
string s;
s.reserve(10000);              // pre-allocate to avoid repeated reallocation
for (int i = 0; i < 10000; i++) s += 'a';   // O(1) amortized per append => O(n) total

// BAD: O(n^2) pattern
string bad;
for (int i = 0; i < 10000; i++) bad = bad + "a";   // each `+` builds a NEW temporary string, copying all of `bad`
```

> **Gotcha:** `s = s + "x"` in a loop is O(n) PER iteration (builds a new string each time) → O(n²) total. `s += "x"` mutates in place and is amortized O(1) per call. Always prefer `+=` in loops; use `reserve` if final size is known.

### 9.7 Comparison, reversing, sorting chars

```cpp
string a = "apple", b = "banana";
cout << (a < b) << "\n";   // => 1  (lexicographic char comparison)

reverse(a.begin(), a.end());
cout << a << "\n";          // => elppa

string s = "dcba";
sort(s.begin(), s.end());
cout << s << "\n";           // => abcd
sort(s.rbegin(), s.rend());
cout << s << "\n";           // => dcba
```

### 9.8 getline and the `>>`/getline mixing trap

```cpp
int n;
cin >> n;                 // reads "3", leaves trailing '\n' in the stream
cin.ignore();              // consume the leftover newline
string line;
getline(cin, line);        // now reads the actual next line correctly
```

> **Gotcha:** `cin >> n` leaves the trailing newline in the input buffer. A subsequent `getline(cin, line)` will read an EMPTY line (just the leftover `\n`) unless you `cin.ignore()` (or `cin.ignore(numeric_limits<streamsize>::max(), '\n')`) first.

### 9.9 Splitting a string by delimiter

No built-in `split` exists in the STL. Two standard approaches:

```cpp
#include <sstream>
// Approach 1: istringstream (splits on whitespace by default)
string s = "the quick brown fox";
istringstream iss(s);
string word;
vector<string> words;
while (iss >> word) words.push_back(word);
for (auto& w : words) cout << w << "|"; // => the|quick|brown|fox|
cout << "\n";

// Approach 2: manual loop with a custom delimiter (e.g. comma)
string csv = "a,bb,ccc,";
vector<string> tokens;
string cur;
for (char c : csv) {
    if (c == ',') { tokens.push_back(cur); cur.clear(); }
    else cur += c;
}
tokens.push_back(cur);   // push whatever remains after the last delimiter (may be empty)
for (auto& t : tokens) cout << "[" << t << "]"; // => [a][bb][ccc][]
cout << "\n";
```

### 9.10 stringstream for parsing and building

```cpp
#include <sstream>
stringstream ss;
ss << "Result: " << 42 << " " << 3.14;
cout << ss.str() << "\n"; // => Result: 42 3.14

stringstream parse("10 20 30");
int a, b, c;
parse >> a >> b >> c;
cout << a + b + c << "\n"; // => 60
```

### 9.11 Raw string literals

```cpp
string path = R"(C:\Users\name\file.txt)";  // no need to escape backslashes
cout << path << "\n";  // => C:\Users\name\file.txt
string regexLike = R"(\d+\.\d+)";
cout << regexLike << "\n"; // => \d+\.\d+
```

### 9.12 `string_view` **[C++17]**

```cpp
#include <string_view>
void printIt(string_view sv) { cout << sv << "\n"; }  // no copy, works with string, literal, or char*

string s = "hello world";
printIt(s);                 // no copy of s made
printIt("literal");          // no temporary string constructed
string_view sv = s.substr(0, 5); // WRONG in intent — s.substr returns a temporary string (owns memory)
string_view sv2(s.c_str(), 5);    // views into s's buffer directly, no copy — correct usage
cout << sv2 << "\n"; // => hello
```

`string_view` is a non-owning "(pointer, length)" view — it avoids copies when a function only needs to READ string data, especially for substrings (a `string_view` substring is O(1); a `string::substr` is O(n) because it allocates a new string).

> **Gotcha (dangling):** `string_view` never extends the lifetime of the data it views. Returning a `string_view` into a local `string`, or taking a view of a temporary (`string_view sv = (s1 + s2);`), leaves `sv` dangling the moment the owner is destroyed — using it afterward is undefined behavior.

```cpp
string_view danger() {
    string local = "temp";
    return local;   // BUG: local is destroyed at function exit, returned view dangles
}
```

---

## 10. array, deque, list, forward_list

### 10.1 `std::array<T, N>`

Fixed-size, stack-allocated (no heap), size is a compile-time template parameter — prefer it over `vector` when the size is known at compile time and small (avoids heap allocation overhead, better cache behavior).

```cpp
#include <array>
array<int, 5> a{1, 2, 3, 4, 5};
a.fill(0);                       // all elements set to 0
cout << a.size() << "\n";        // => 5  (fixed, always 5, part of the type)
for (int x : a) cout << x << " "; // => 0 0 0 0 0
cout << "\n";

sort(a.begin(), a.end());         // works with std algorithms just like vector
cout << a.front() << " " << a.back() << "\n"; // => 0 0
```

### 10.2 Raw C arrays

```cpp
int a[100];                       // uninitialized (garbage) if local/automatic storage
int b[100] = {};                  // zero-initialized
memset(a, -1, sizeof a);           // sets every BYTE to 0xFF => every int becomes -1 (all bits set)
// memset(a, 1, sizeof a);         // WRONG for "set to 1": sets every BYTE to 0x01
                                    // => each int becomes 0x01010101 = 16843009, NOT 1
cout << a[0] << "\n";               // => -1
cout << sizeof(a) / sizeof(a[0]) << "\n"; // => 100  (classic "array length" trick)

int grid[3][4] = {};               // multidimensional, contiguous memory (row-major)
grid[1][2] = 9;

void takesArray(int arr[]) { /* arr is really int*, sizeof(arr) here would be 8, not 400! */ }
cout << sizeof(a) << "\n";  // => 400 (100 * 4 bytes) when `a` is the real array in this scope
```

> **Gotcha:** `memset` operates byte-by-byte, so it only "just works" for setting all bits to `0` or all bits to `1` (i.e., `-1` for signed ints). Any other value (`memset(a, 1, ...)`, `memset(a, 5, ...)`) produces a garbage repeated-byte pattern, not the integer value you intended. Use `fill(a, a+n, value)` or a loop for arbitrary values.

> **Gotcha:** arrays decay to pointers when passed to functions — `sizeof` inside a function that received `int arr[]` gives the pointer size (8 on 64-bit), not the array's byte size. The `sizeof(a)/sizeof(a[0])` trick only works in the scope where the array was actually declared.

### 10.3 `deque` — double-ended queue

```cpp
#include <deque>
deque<int> dq{1, 2, 3};
dq.push_back(4);     // {1,2,3,4}
dq.push_front(0);    // {0,1,2,3,4}
dq.pop_back();        // {0,1,2,3}
dq.pop_front();        // {1,2,3}
cout << dq[1] << "\n"; // => 2  (random access supported, O(1))
for (int x : dq) cout << x << " "; // => 1 2 3
cout << "\n";
```

`push_front`/`pop_front` are O(1) (vector's are O(n) since everything must shift) — this makes `deque` the natural choice for:
- **BFS queues** (push_back to enqueue, pop_front to dequeue, both O(1))
- **Monotonic deque** (sliding window maximum/minimum): push/pop from both ends as the window slides

```cpp
// Sliding window maximum sketch using a monotonic deque of INDICES
vector<int> nums{1, 3, -1, -3, 5, 3, 6, 7};
int k = 3;
deque<int> dq;              // stores indices, values decreasing front-to-back
vector<int> ans;
for (int i = 0; i < (int)nums.size(); i++) {
    while (!dq.empty() && dq.front() <= i - k) dq.pop_front();       // drop out-of-window
    while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back(); // maintain decreasing
    dq.push_back(i);
    if (i >= k - 1) ans.push_back(nums[dq.front()]);
}
for (int x : ans) cout << x << " "; // => 3 3 5 5 6 7
cout << "\n";
```

Downside: `deque` is implemented as a sequence of fixed-size chunks (not one contiguous block), so it has worse cache locality than `vector` and higher per-element memory overhead — don't default to it when `vector` suffices.

### 10.4 `list` — doubly-linked list

```cpp
#include <list>
list<int> l{1, 2, 3};
l.push_back(4); l.push_front(0);         // {0,1,2,3,4}
auto it = find(l.begin(), l.end(), 2);
l.insert(it, 99);                          // insert before the found element: {0,1,99,2,3,4}
l.erase(it);                                // O(1) given the iterator — no shifting needed
for (int x : l) cout << x << " ";           // => 0 1 99 3 4
cout << "\n";
```

Key property: inserting/erasing given an iterator is O(1) (just pointer relinking, no shifting), and — crucially — existing iterators/references to OTHER elements stay valid after insert/erase (unlike `vector`, where any insert/erase can invalidate iterators). This is exactly why `list` shows up in LRU cache implementations: you can hold a `list<>::iterator` per key in a hash map and splice/move nodes in O(1) without invalidating other iterators.

```cpp
// LRU cache sketch: list<{key,val}> for recency order + unordered_map<key, iterator> for O(1) lookup
#include <unordered_map>
struct LRUCache {
    int cap;
    list<pair<int,int>> order;                                   // front = most recent
    unordered_map<int, list<pair<int,int>>::iterator> loc;
    LRUCache(int capacity) : cap(capacity) {}

    int get(int key) {
        if (!loc.count(key)) return -1;
        auto it = loc[key];
        order.splice(order.begin(), order, it);  // move node to front, O(1), no invalidation
        return it->second;
    }
    void put(int key, int value) {
        if (loc.count(key)) {
            loc[key]->second = value;
            order.splice(order.begin(), order, loc[key]);
            return;
        }
        if ((int)order.size() == cap) {
            int lastKey = order.back().first;
            order.pop_back();
            loc.erase(lastKey);
        }
        order.emplace_front(key, value);
        loc[key] = order.begin();
    }
};

int main() {
    LRUCache c(2);
    c.put(1, 1); c.put(2, 2);
    cout << c.get(1) << "\n";  // => 1  (1 is now most recent)
    c.put(3, 3);                 // evicts 2 (least recently used)
    cout << c.get(2) << "\n";   // => -1
}
```

`splice` moves nodes between (or within) lists in O(1) by relinking pointers — no copying, no reallocation, and the moved node's iterator stays valid throughout.

### 10.5 `forward_list` (brief)

Singly-linked list, `<forward_list>`. Only forward iteration (no `.back()`, no `rbegin()`, no `.size()` even — checking size is O(n) if you write it yourself, hence it's omitted from the API). Slightly lower memory overhead than `list` (one pointer per node instead of two). Rarely used in competitive programming — reach for `list` unless memory is extremely tight and you never need backward traversal.

```cpp
#include <forward_list>
forward_list<int> fl{1, 2, 3};
fl.push_front(0);                 // only front insertion is O(1) directly
for (int x : fl) cout << x << " "; // => 0 1 2 3
cout << "\n";
```

### 10.6 Decision table — which sequence container to pick

| Need | Container |
|---|---|
| Default choice, random access, push/pop at back | `vector` |
| Fixed compile-time size, no heap allocation | `array` |
| Push/pop at BOTH ends, still need random access (BFS, sliding window) | `deque` |
| Frequent insert/erase in the MIDDLE given an iterator, iterator stability required (LRU, splicing) | `list` |
| Same as `list` but memory is extremely tight, no backward traversal needed | `forward_list` |
| Contiguous memory required for C-API interop | `vector` (or `array`) |
| Need C-array semantics with a size mismatch trap avoided | `array` (has `.size()`, unlike raw arrays after decay) |

---

## 11. pair, tuple and Structured Bindings

### 11.1 `pair`

```cpp
#include <utility>
pair<int, string> p1 = make_pair(1, "one");
pair<int, string> p2{2, "two"};          // brace init, same effect
pair<int, string> p3 = {3, "three"};      // also fine

cout << p1.first << " " << p1.second << "\n"; // => 1 one
p1.first = 10;                                  // members are directly mutable
cout << p1.first << "\n";                        // => 10
```

Comparison is lexicographic: compares `.first` first; if equal, compares `.second`.

```cpp
pair<int,int> a{1, 5}, b{1, 3}, c{2, 0};
cout << (a < b) << "\n";  // => 0  (firsts equal: 1==1, then 5 < 3 is false)
cout << (b < a) << "\n";  // => 1  (firsts equal, 3 < 5 true)
cout << (a < c) << "\n";  // => 1  (1 < 2, second never checked)
```

This is exactly why `pair<int,int>` is the standard trick for **Dijkstra** (`pair<dist, node>` in a min-heap sorts by distance automatically) and for **sorting by a key while carrying a payload**:

```cpp
// Dijkstra-style min-heap: smallest distance popped first
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
pq.push({5, 1});   // {dist, node}
pq.push({2, 2});
pq.push({8, 3});
cout << pq.top().first << " " << pq.top().second << "\n"; // => 2 2
```

```cpp
// Sorting objects by a key, carrying the original value along
vector<pair<int,int>> v{{3, 100}, {1, 200}, {2, 300}};
sort(v.begin(), v.end());   // sorts by .first, then .second — no comparator needed
for (auto& p : v) cout << "(" << p.first << "," << p.second << ") ";
// => (1,200) (2,300) (3,100)
cout << "\n";
```

### 11.2 `tuple`

```cpp
#include <tuple>
tuple<int, string, double> t1 = make_tuple(1, "hi", 3.14);
tuple<int, string, double> t2{2, "yo", 2.5};

cout << get<0>(t1) << " " << get<1>(t1) << " " << get<2>(t1) << "\n"; // => 1 hi 3.14

int a; string s; double d;
tie(a, s, d) = t1;                       // unpack into existing variables
cout << a << " " << s << " " << d << "\n"; // => 1 hi 3.14

tie(a, ignore, d) = t2;                   // ignore skips a field you don't need
cout << a << " " << d << "\n";              // => 2 2.5
```

`tuple` comparison is lexicographic, element by element, same rule as `pair`:

```cpp
tuple<int,int,int> x{1, 2, 3}, y{1, 2, 4};
cout << (x < y) << "\n"; // => 1
```

### 11.3 Structured bindings **[C++17]**

```cpp
pair<int, string> p{1, "one"};
auto [id, name] = p;             // decompose directly — no .first/.second needed
cout << id << " " << name << "\n"; // => 1 one

tuple<int, int, int> t{1, 2, 3};
auto [x, y, z] = t;
cout << x + y + z << "\n"; // => 6

unordered_map<string, int> mp{{"a", 1}, {"b", 2}};
for (auto& [k, v] : mp) cout << k << "=" << v << " "; // order not guaranteed for unordered_map
cout << "\n";
// => a=1 b=2   (or b=2 a=1 — unordered)

map<string, int> ordered{{"a", 1}, {"b", 2}};
for (auto& [k, v] : ordered) cout << k << "=" << v << " "; // map IS ordered by key
cout << "\n"; // => a=1 b=2
```

### 11.4 pair/tuple as map keys and in priority_queue

```cpp
map<pair<int,int>, int> cellValue;      // pair works as a key: needs operator< (built-in, lexicographic)
cellValue[{0, 0}] = 5;
cellValue[{1, 2}] = 9;
cout << cellValue[{1, 2}] << "\n"; // => 9

set<tuple<int,int,int>> triples;         // tuple also has built-in lexicographic operator<
triples.insert({1, 2, 3});
triples.insert({1, 2, 2});
for (auto& [a, b, c] : triples) cout << "(" << a << "," << b << "," << c << ") ";
cout << "\n"; // => (1,2,2) (1,2,3)   (sorted lexicographically)

priority_queue<pair<int,int>> maxHeap;   // max-heap by default, compares pair lexicographically
maxHeap.push({3, 1}); maxHeap.push({5, 2}); maxHeap.push({5, 0});
cout << maxHeap.top().first << " " << maxHeap.top().second << "\n"; // => 5 2 (tie on first, larger second wins)
```

### 11.5 Sorting `vector<pair<int,int>>` by the SECOND element

Default `pair` comparison sorts by `.first` first — to sort by `.second` instead, supply a lambda comparator:

```cpp
vector<pair<int,int>> v{{1, 30}, {2, 10}, {3, 20}};
sort(v.begin(), v.end(), [](const pair<int,int>& a, const pair<int,int>& b) {
    return a.second < b.second;
});
for (auto& p : v) cout << "(" << p.first << "," << p.second << ") ";
cout << "\n"; // => (2,10) (3,20) (1,30)
```

### 11.6 `tie` for concise multi-key `operator<`

Writing a manual multi-field comparator is error-prone (easy to forget a tie-break). `tie` builds a temporary tuple of references and reuses tuple's lexicographic comparison:

```cpp
struct Person {
    int age;
    string name;
    bool operator<(const Person& o) const {
        return tie(age, name) < tie(o.age, o.name);   // compares age first, then name — one line, no bugs
    }
};

vector<Person> people{{30, "Bob"}, {25, "Alice"}, {30, "Anna"}};
sort(people.begin(), people.end());
for (auto& p : people) cout << p.age << ":" << p.name << " ";
cout << "\n"; // => 25:Alice 30:Anna 30:Bob
```

> **Gotcha:** Manually chaining `if (a.x != b.x) return a.x < b.x; return a.y < b.y;` for 3+ fields gets error-prone fast (easy to compare the wrong field or forget a branch). `tie(a1,a2,a3) < tie(b1,b2,b3)` gives the same lexicographic multi-key comparison in one line and is the idiomatic competitive-programming pattern.

---

# PART III — STL CONTAINERS (ASSOCIATIVE + ADAPTERS)

## 12. map and unordered_map

`map<K,V>` = balanced BST (red-black tree), keys sorted, `O(log n)` ops.
`unordered_map<K,V>` = hash table, keys arbitrary order, `O(1)` average ops.

### 12.1 Complete method table

| Method | What it does | `map` complexity | `unordered_map` complexity |
|---|---|---|---|
| `mp[k]` | access, **default-inserts** if absent | O(log n) | O(1) avg |
| `mp.at(k)` | access, **throws** `out_of_range` if absent | O(log n) | O(1) avg |
| `mp.insert({k,v})` | insert if key absent, no-op if present | O(log n) | O(1) avg |
| `mp.insert_or_assign(k,v)` **[C++17]** | insert or overwrite | O(log n) | O(1) avg |
| `mp.emplace(k,v)` | construct in place if absent | O(log n) | O(1) avg |
| `mp.try_emplace(k, args...)` **[C++17]** | insert if absent, does NOT construct value if key exists (avoids wasted construction) | O(log n) | O(1) avg |
| `mp.erase(k)` | erase by key, returns count erased (0/1) | O(log n) | O(1) avg |
| `mp.erase(it)` | erase by iterator | O(1) amortized | O(1) |
| `mp.erase(first,last)` | erase range | O(range) | O(range) |
| `mp.find(k)` | returns iterator or `end()` | O(log n) | O(1) avg |
| `mp.count(k)` | 0 or 1 (unique keys) | O(log n) | O(1) avg |
| `mp.contains(k)` **[C++20]** | bool, cleanest existence check | O(log n) | O(1) avg |
| `mp.size()` / `mp.empty()` / `mp.clear()` | — | O(1)/O(1)/O(n) | O(1)/O(1)/O(n) |
| `mp.lower_bound(k)` | first elem with key `>= k` | O(log n) | **not available** |
| `mp.upper_bound(k)` | first elem with key `> k` | O(log n) | **not available** |
| `mp.equal_range(k)` | `{lower_bound, upper_bound}` pair | O(log n) | exists, but ≤1 element (unique keys) — rarely useful |
| `mp.begin()/end()` | smallest → largest (map) / arbitrary (unordered) | — | — |
| `mp.rbegin()/rend()` | largest → smallest | O(1) to get | **not available** on `unordered_map` |

> `unordered_map` genuinely has NO `lower_bound`/`upper_bound` — there's no ordering to bound against. If you need "smallest key ≥ x", you need `map`, period.

### 12.2 THE operator[] INSERTION TRAP

`mp[k]` **default-constructs and inserts** `k` if it doesn't already exist — even a bare read does this.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    map<string,int> mp;
    mp["a"] = 1;
    if (mp["ghost"] == 0) {}   // "ghost" now INSERTED with value 0!
    cout << mp.size();        // => 2   (not 1 — "ghost" silently added)
}
```

This is the single most common `map` bug. Consequences:
- Silently grows the map (memory, and wrong `size()`/`empty()`).
- Breaks any logic doing `count`-style checks afterward (`mp.count("ghost")` is now 1).
- In a loop over "does this key exist", using `mp[k]` for a read poisons the map for later iterations.

**Rule:** use `mp[k]` ONLY when you intend to insert-or-mutate. For read-only existence/lookup use `find`, `count`, or `contains`:

```cpp
map<string,int> mp{{"a",1}};
if (mp.find("ghost") != mp.end()) { /* exists */ }
if (mp.count("ghost")) { /* exists, count is 0 or 1 for map */ }
if (mp.contains("ghost")) { /* C++20, cleanest */ }   // does NOT insert
cout << mp.size();   // => 1  (unaffected)
```

> **Gotcha:** `at()` throws if missing — safe for reads that must not insert, but slower to write (`try { mp.at(k) } catch(...) {}` is ugly). Prefer `find`/`contains` over `at()` in try/catch for hot loops.

### 12.3 Iteration, structured bindings, ordering

```cpp
map<string,int> mp{{"b",2},{"a",1},{"c",3}};
for (auto& [k, v] : mp) cout << k << ":" << v << " ";
// => a:1 b:2 c:3     (map: ALWAYS sorted by key)

unordered_map<string,int> ump{{"b",2},{"a",1},{"c",3}};
for (auto& [k, v] : ump) cout << k << ":" << v << " ";
// => arbitrary order, e.g. c:3 a:1 b:2 — DO NOT rely on it
```

`auto& [k, v]` lets you mutate `v` in place (reference); use `auto [k, v]` (by value) if you don't need to modify or if `mp` is const.

### 12.4 Frequency counting idiom

```cpp
vector<int> a{4,2,4,4,2,7};
unordered_map<int,int> freq;
for (int x : a) ++freq[x];     // operator[] default-inits int to 0, then ++
for (auto& [k,v] : freq) cout << k << "->" << v << " ";
// => 4->3 2->2 7->1   (order arbitrary)
```

This is the ONE place `operator[]`'s auto-insert is exactly what you want.

### 12.5 map as an ordered structure — predecessor/successor, min/max

```cpp
map<int,int> mp{{10,1},{20,2},{30,3}};

// max key
cout << mp.rbegin()->first;              // => 30
// min key
cout << mp.begin()->first;               // => 10

// successor: smallest key >= 25
auto it = mp.lower_bound(25);
cout << (it != mp.end() ? it->first : -1);   // => 30

// predecessor: largest key <= 25
auto it2 = mp.upper_bound(25);            // first key > 25
if (it2 == mp.begin()) { /* no predecessor */ }
else { --it2; cout << it2->first; }       // => 20
```

Using `map<K, V>` (or `set<K>`) with `lower_bound`/`upper_bound` gives you an **ordered map/set** for range/predecessor queries — a poor man's balanced BST, useful in sweep-line, interval-merging, and "closest value" problems.

### 12.6 Custom comparator for map

```cpp
struct Cmp {
    bool operator()(const int& a, const int& b) const { return a > b; } // descending
};
map<int,int,Cmp> mp;      // sorted descending by key
mp[3]=1; mp[1]=2; mp[5]=3;
for (auto& [k,v] : mp) cout << k << " ";  // => 5 3 1
```

### 12.7 Custom hash for unordered_map with pair/struct keys

`unordered_map<pair<int,int>, int>` does **NOT compile** — `std::hash<pair<int,int>>` is not specialized in the standard library. You must supply a hash functor.

```cpp
#include <bits/stdc++.h>
using namespace std;

struct PairHash {
    size_t operator()(const pair<int,int>& p) const {
        // combine both hashes; shifting one avoids trivial collisions on symmetric pairs
        return hash<long long>()(((long long)p.first << 32) ^ (unsigned int)p.second);
    }
};

int main() {
    unordered_map<pair<int,int>, int, PairHash> mp;
    mp[{1,2}] = 100;
    cout << mp[{1,2}];   // => 100
}
```

For a custom struct key, also need `operator==`:

```cpp
struct Point {
    int x, y;
    bool operator==(const Point& o) const { return x==o.x && y==o.y; }
};
struct PointHash {
    size_t operator()(const Point& p) const {
        return hash<int>()(p.x) ^ (hash<int>()(p.y) << 1);
    }
};
unordered_map<Point, int, PointHash> mp;
mp[{1,2}] = 42;
cout << mp[{1,2}];   // => 42
```

> **Gotcha:** if you forget `operator==` on the struct key, compile error about no match for `operator==` — hashing alone isn't enough, the table needs equality for collision resolution.

> A quick trick many competitive programmers use for `pair<int,int>` keys: `map<pair<int,int>,int>` works out of the box (uses lexicographic `<` from `pair`), no custom code needed — trade `O(log n)` for zero boilerplate.

### 12.8 multimap (brief)

Allows duplicate keys; no `operator[]` (ambiguous which value to return).

```cpp
multimap<int,string> mm;
mm.insert({1,"a"}); mm.insert({1,"b"}); mm.insert({2,"c"});
auto [lo, hi] = mm.equal_range(1);
for (auto it = lo; it != hi; ++it) cout << it->second << " ";  // => a b
```

### 12.9 Complexity contrast & when unordered_map degrades

| Operation | `map` | `unordered_map` |
|---|---|---|
| find/insert/erase | O(log n) guaranteed | O(1) average, **O(n) worst case** |
| ordered iteration | yes, free | no |
| memory overhead | tree nodes, pointers | buckets + load factor overhead |

`unordered_map` degrades to O(n) per op when:
- Many hash collisions (adversarial input, or poor custom hash — e.g. `PairHash` above that doesn't spread bits well).
- Not reserving capacity → repeated rehashing (`mp.reserve(n)` upfront avoids this).
- Using it as `unordered_map<int,int>` in a codeforces judge with adversarial anti-hash tests exploiting the default `std::hash<int>` (identity function) — mitigate with a custom hash that mixes bits (e.g. splitmix64), or just use `map` if n is small enough that O(log n) doesn't matter.

```cpp
// anti-hash-test-safe custom hash for int keys, common CP idiom
struct SafeHash {
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t seed = chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + seed);
    }
};
// unordered_map<int,int,SafeHash> mp;
```

---

## 13. set, unordered_set, multiset

### 13.1 Complete method table

| Method | What it does | `set`/`multiset` | `unordered_set` |
|---|---|---|---|
| `s.insert(x)` | insert (unique for set, always for multiset) | O(log n) | O(1) avg |
| `s.erase(x)` | erase by **value** — ALL copies in multiset | O(log n + k) | O(1) avg |
| `s.erase(it)` | erase by iterator — exactly one element | O(1) amortized | O(1) |
| `s.find(x)` | iterator to element or `end()` | O(log n) | O(1) avg |
| `s.count(x)` | number of matches (0/1 set, ≥0 multiset) | O(log n + k) | O(1) avg |
| `s.contains(x)` **[C++20]** | bool existence check | O(log n) | O(1) avg |
| `s.lower_bound(x)` | first elem `>= x` | O(log n) | **not available** |
| `s.upper_bound(x)` | first elem `> x` | O(log n) | **not available** |
| `s.begin()/rbegin()` | min / max element | O(1) | begin only, no order |
| `s.size()/empty()/clear()` | — | O(1)/O(1)/O(n) | same |

### 13.2 set as a sorted structure — min, max, predecessor/successor

```cpp
set<int> s{5, 1, 9, 3};
cout << *s.begin();       // => 1   (min)
cout << *s.rbegin();      // => 9   (max)
```

**Idiom: largest element `<= x`** (predecessor-or-equal):

```cpp
set<int> s{1, 4, 7, 10};
int x = 8;
auto it = s.upper_bound(x);   // first element STRICTLY greater than x -> 10
if (it == s.begin()) {
    // no element <= x at all
} else {
    --it;                      // step back one -> largest element <= x
    cout << *it;               // => 7
}
```

Why `upper_bound` and not `lower_bound` here: `lower_bound(x)` gives first `>= x`, which if `x` itself is present points AT `x` — stepping back from that would skip `x`. `upper_bound(x)` always points strictly past `x`, so decrementing correctly lands on `x` if present, else the true predecessor.

**Idiom: smallest element `>= x`** (successor-or-equal): just `s.lower_bound(x)` directly, no decrement, guard against `end()`.

```cpp
auto it2 = s.lower_bound(5);
cout << (it2 != s.end() ? *it2 : -1);   // => 7
```

> **Gotcha:** always guard `it == s.begin()` before `--it` — decrementing `begin()` is undefined behavior.

### 13.3 Custom comparator — the deduplication trap

```cpp
struct CmpAbs {
    bool operator()(int a, int b) const { return abs(a) < abs(b); }
};
set<int, CmpAbs> s;
s.insert(5);
s.insert(-5);      // abs(-5) == abs(5) -> comparator says "equivalent" -> NOT inserted
cout << s.size();  // => 1   (NOT 2 — -5 was silently dropped as a "duplicate")
```

A `set`'s notion of equality is `!cmp(a,b) && !cmp(b,a)` ("equivalence"), NOT `operator==`. Any comparator that returns `false` both ways for genuinely different values will merge them into one slot. This is a very common bug when sets are keyed on a derived property (e.g. sorting points by distance) instead of full identity.

### 13.4 multiset — duplicates allowed, the classic erase bug

```cpp
multiset<int> ms{1, 2, 2, 2, 3};

ms.erase(2);                 // erases ALL copies of 2
cout << ms.size();           // => 2   (only 1 and 3 remain)
```

vs.

```cpp
multiset<int> ms2{1, 2, 2, 2, 3};
ms2.erase(ms2.find(2));      // erases exactly ONE copy (the iterator form)
cout << ms2.size();          // => 4
```

> **Gotcha:** `ms.erase(value)` (value overload) removes every matching element and returns the count removed. `ms.erase(iterator)` removes exactly the one pointed-to element. Mixing these up (calling the value overload when you meant "remove just this one") silently deletes far more than intended — a classic multiset bug in sliding-window / running-median problems.

**multiset as running median / sliding window:**

```cpp
multiset<int> window;
vector<int> a{1,3,-1,-3,5,3,6,7};
int k = 3;
vector<int> mediansOrMax;
for (int i = 0; i < (int)a.size(); i++) {
    window.insert(a[i]);
    if (window.size() > (size_t)k) window.erase(window.find(a[i-k])); // remove exact element leaving window
    if (window.size() == (size_t)k) {
        auto it = window.begin(); advance(it, k/2);
        mediansOrMax.push_back(*it);   // median-ish / order-statistic
    }
}
cout << mediansOrMax.back();  // => 6  (median of last window {5,6,7})
```

### 13.5 unordered_set + custom hash

Same story as `unordered_map`: `unordered_set<pair<int,int>>` needs a hash functor (reuse `PairHash` from §12.7).

```cpp
unordered_set<int> us{3, 1, 4, 1, 5};   // duplicate 1 collapses
cout << us.size();   // => 4
```

### 13.6 set operations vs `<algorithm>`

`set`/`multiset` don't have built-in union/intersection member functions — use the sorted-range algorithms `set_union`, `set_intersection`, `set_difference`, `set_symmetric_difference` from `<algorithm>` (they work on any sorted range, including `set`'s natural iteration order). See Part on `<algorithm>` for full signatures and snippets.

---

## 14. Container Adapters: stack, queue, priority_queue

Adapters wrap an underlying container (default `deque` for `stack`/`queue`, `vector` for `priority_queue`) and expose a restricted interface. None support iteration or random access.

### 14.1 stack (LIFO)

| Method | What it does | Complexity |
|---|---|---|
| `st.push(x)` | push on top | O(1) amortized |
| `st.pop()` | remove top — **returns void** | O(1) |
| `st.top()` | reference to top element | O(1) |
| `st.empty()` / `st.size()` | — | O(1) |
| `st.emplace(args...)` | construct in place on top | O(1) amortized |

```cpp
stack<int> st;
st.push(1); st.push(2); st.push(3);
cout << st.top();   // => 3
st.pop();            // pop() returns nothing — you MUST call top() first if you need the value
cout << st.top();   // => 2
```

> **Gotcha:** `st.pop()` does not return the popped value (unlike Python's `.pop()`). Pattern: `int x = st.top(); st.pop();`

### 14.2 queue (FIFO)

| Method | What it does | Complexity |
|---|---|---|
| `q.push(x)` | enqueue at back | O(1) amortized |
| `q.pop()` | dequeue from front — **returns void** | O(1) |
| `q.front()` / `q.back()` | reference to front/back | O(1) |
| `q.empty()` / `q.size()` | — | O(1) |

```cpp
queue<int> q;
q.push(1); q.push(2); q.push(3);
cout << q.front() << " " << q.back();   // => 1 3
q.pop();
cout << q.front();                       // => 2
```

### 14.3 priority_queue — in depth

Default is a **MAX-heap**: `top()` gives the largest element.

| Method | What it does | Complexity |
|---|---|---|
| `pq.push(x)` / `pq.emplace(args...)` | insert, re-heapify | O(log n) |
| `pq.pop()` | remove top — **returns void** | O(log n) |
| `pq.top()` | reference to max (or min, if configured) | O(1) |
| `pq.empty()` / `pq.size()` | — | O(1) |

```cpp
priority_queue<int> pq;          // max-heap by default
pq.push(3); pq.push(1); pq.push(4); pq.push(1); pq.push(5);
cout << pq.top();   // => 5
pq.pop();
cout << pq.top();   // => 4
```

#### Three ways to get a MIN-heap

```cpp
// 1. greater<int> as Compare
priority_queue<int, vector<int>, greater<int>> minpq1;
minpq1.push(3); minpq1.push(1); minpq1.push(4);
cout << minpq1.top();   // => 1

// 2. negate values, use default max-heap, negate back on read
priority_queue<int> minpq2;
minpq2.push(-3); minpq2.push(-1); minpq2.push(-4);
cout << -minpq2.top();  // => 1

// 3. custom comparator (struct) — see below
```

#### THE COMPARATOR INVERSION RULE

`sort(v.begin(), v.end(), greater<int>())` sorts **descending** — largest first. But `priority_queue<int, vector<int>, greater<int>>` gives a **min-heap** — smallest on top. Same `greater<int>`, opposite-feeling result. Why?

- `sort`'s comparator directly answers "should `a` come before `b`?" — `greater<int>{}(a,b)` is `a > b`, so bigger elements are placed first → descending order, front element is the max.
- `priority_queue`'s comparator answers a different question: "does `a` have **lower priority** than `b`?" (i.e., `cmp(a,b)` true means `a` should sit further from the top than `b`). The heap always keeps the "highest priority" (the one that loses fewest comparisons) at `top()`. With `less<int>` (default), `cmp(a,b) = a<b` means smaller elements have lower priority → **largest ends up on top** (max-heap). Swap to `greater<int>`: now `cmp(a,b) = a>b` means larger elements have lower priority → **smallest ends up on top** (min-heap).

In short: **for `priority_queue`, the comparator defines "lower priority than", and `top()` is whichever element is never "lower priority" than anything else** — it's the inverse framing of `sort`'s "comes before".

```cpp
// custom comparator via struct
struct Cmp {
    bool operator()(int a, int b) const { return a > b; }  // a "lower priority" if a > b => min-heap
};
priority_queue<int, vector<int>, Cmp> pq1;
pq1.push(5); pq1.push(2); pq1.push(8);
cout << pq1.top();   // => 2

// custom comparator via lambda (decltype form — constantly forgotten syntax)
auto cmp = [](int a, int b) { return a > b; };   // same rule: min-heap
priority_queue<int, vector<int>, decltype(cmp)> pq2(cmp);   // MUST pass cmp instance to ctor
pq2.push(5); pq2.push(2); pq2.push(8);
cout << pq2.top();   // => 2
```

> **Gotcha:** with the lambda form you MUST pass the lambda instance to the constructor — `priority_queue<int, vector<int>, decltype(cmp)> pq2;` (no `(cmp)`) fails to compile in C++17 because a captureless lambda's type has no default constructor accessible in this context unless the lambda is stateless AND you're on C++20 (which added default-constructibility for captureless lambdas' closure types). Always pass `(cmp)` explicitly to be safe on C++17.

#### priority_queue of pair / struct

```cpp
// max-heap of pair<int,int> by default: compares .first then .second lexicographically
priority_queue<pair<int,int>> pq;
pq.push({3,1}); pq.push({1,5}); pq.push({3,0});
cout << pq.top().first << "," << pq.top().second;  // => 3,1  (3 ties broken by larger second)

// min-heap of pair by distance, custom comparator struct — typical Dijkstra pattern
struct Cmp {
    bool operator()(const pair<int,int>& a, const pair<int,int>& b) const {
        return a.first > b.first;   // compare only distance, ascending on top
    }
};
priority_queue<pair<int,int>, vector<pair<int,int>>, Cmp> dijkstraPQ;
dijkstraPQ.push({5, 2}); dijkstraPQ.push({1, 0}); dijkstraPQ.push({3, 1});
cout << dijkstraPQ.top().first;   // => 1
```

#### No iteration, no random access, no decrease-key — lazy deletion

`priority_queue` cannot iterate, index, or update/remove an arbitrary element (no decrease-key like a Fibonacci heap). Dijkstra's algorithm needs "decrease distance of node X" — the standard workaround is **lazy deletion**: push a new `(newDist, node)` pair without removing the stale one, and when popping, skip entries that are outdated.

```cpp
// Dijkstra-style lazy deletion sketch
vector<int> dist(n, INT_MAX);
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;  // min-heap by first
dist[src] = 0;
pq.push({0, src});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;   // stale entry from an earlier, worse push — skip it
    for (auto& [v, w] : adj[u]) {
        if (dist[u] + w < dist[v]) {
            dist[v] = dist[u] + w;
            pq.push({dist[v], v});   // push new better entry; old stale one remains, will be skipped later
        }
    }
}
```

#### Complexity table

| Operation | Complexity |
|---|---|
| push / pop | O(log n) |
| top | O(1) |
| build from vector (via range constructor) | O(n) |
| iteration / random access / decrease-key | **not supported** |

---

## 15. bitset

`bitset<N>` — fixed-size, **compile-time** `N`, packed bits, extremely fast bitwise ops.

### 15.1 Complete operation table

| Operation | What it does | Complexity |
|---|---|---|
| `bitset<N> b;` | all bits 0 | O(N/64) |
| `bitset<N> b(ulong_val)` | init from integer | O(N/64) |
| `bitset<N> b(string)` | init from `"1010"`-style string | O(N) |
| `b.set()` | set all bits to 1 | O(N/64) |
| `b.set(i)` / `b.set(i, val)` | set bit `i` to 1 / to `val` | O(1) |
| `b.reset()` / `b.reset(i)` | clear all / clear bit `i` | O(N/64) / O(1) |
| `b.flip()` / `b.flip(i)` | toggle all / toggle bit `i` | O(N/64) / O(1) |
| `b.test(i)` | bool value of bit `i` (bounds-checked) | O(1) |
| `b[i]` | access bit `i` (no bounds check, returns proxy) | O(1) |
| `b.count()` | number of set bits | O(N/64) |
| `b.any()` | true if any bit set | O(N/64) |
| `b.none()` | true if no bit set | O(N/64) |
| `b.all()` | true if all bits set | O(N/64) |
| `b.size()` | returns `N` (compile-time constant) | O(1) |
| `b.to_string()` | string of `'0'`/`'1'` | O(N) |
| `b.to_ulong()` / `b.to_ullong()` | convert to integer (throws `overflow_error` if doesn't fit) | O(N/64) |
| `b << k` / `b >> k` | shift (new bits are 0) | O(N/64) |
| `b & c`, `b \| c`, `b ^ c`, `~b` | bitwise ops between two same-size bitsets | O(N/64) |

```cpp
bitset<8> b("1011");        // right-aligned: 00001011
cout << b;                  // => 00001011
cout << b.count();          // => 3
b.set(4);                   // 00011011
cout << b.to_string();      // => 00011011
cout << b.any() << b.none() << b.all();  // => 100
bitset<8> c(5);             // from int: 00000101
cout << (b & c);            // => 00000001
```

### 15.2 Why 64x speedup — knapsack / reachability / sieve

A `bitset<N>` packs `N` bits into `N/64` machine words; bitwise AND/OR/XOR/shift on the whole bitset happen `64` bits at a time per CPU instruction — turning an O(N) inner loop into O(N/64).

**Knapsack feasibility** (which sums are achievable):

```cpp
// classic 0/1 subset-sum feasibility via bitset, capacity W
int W = 20;
vector<int> weights{2, 3, 7};
bitset<21> dp;
dp[0] = 1;                       // sum 0 is achievable
for (int w : weights) dp |= dp << w;   // for every achievable sum s, s+w becomes achievable
cout << dp.count();              // => number of distinct achievable sums (including 0)
cout << dp[12];                  // => 1  (2+3+7=12 achievable)
```

This replaces an O(N*W) DP inner loop with O(N*W/64) — the single most common bitset use case in CP.

**Reachability / transitive closure** — each node's adjacency row as a `bitset<N>`; OR-ing rows computes reachable-set unions in O(N/64) instead of O(N).

**Sieve of Eratosthenes** — a `bitset<N>` for `isComposite` flags cuts memory 8x vs `vector<bool>`-per-byte... — actually `vector<bool>` is already bit-packed (see 15.4); the win here is mainly cache-friendliness and the same shift/AND tricks for segmented sieves.

### 15.3 Iterating set bits — `_Find_first()` / `_Find_next()`

GCC/libstdc++ extension (not portable ISO C++, flag it): iterate only over set bit positions in O(popcount) instead of O(N).

```cpp
// GCC extension — not standard C++, works with g++/libstdc++ (fine for Codeforces/most judges)
bitset<16> b("0000000000101010");
for (size_t i = b._Find_first(); i != b.size(); i = b._Find_next(i)) {
    cout << i << " ";
}
// => 1 3 5
```

Portable alternative: loop `for (int i = 0; i < N; i++) if (b[i]) ...` (O(N)), or maintain a separate popcount-driven loop with `__builtin_ctzll` on the underlying words if you need portability + speed.

### 15.4 bitset vs vector<bool> vs raw bitmask int — decision table

| Need | Use | Why |
|---|---|---|
| Size known at compile time, ≤ ~10^7 bits, need fast bitwise DP/shift tricks | `bitset<N>` | fastest bitwise ops, fixed-size, `<<`/`>>`/`&`/`\|` all O(N/64) |
| Size known only at runtime | `vector<bool>` | bit-packed like bitset but dynamically sized; **no** fast whole-container shift/AND (must loop or use `.data()` hacks) |
| ≤ 64 flags (e.g. subset masks in bitmask DP, visited-set for ≤20 items) | raw `int`/`long long`/`uint64_t` | single machine word, fits in a DP state directly, usable as array index (`dp[mask]`) |
| Need to hash/compare/use as a map key compactly | `bitset<N>` (has `operator==`, hashable via `std::hash<bitset<N>>`) or raw int (trivially hashable) | `vector<bool>` is awkward as a key |

> **Gotcha:** `vector<bool>` is a notorious specialization — it's NOT a real `vector<T>` (no contiguous `bool*` storage, `&v[0]` doesn't work, passing to APIs expecting `vector<T>&` breaks). For DSA, prefer `bitset<N>` when N is fixed, or `vector<char>`/`vector<int8_t>` when you need genuine per-element addressability and runtime sizing.

---

## 16. Choosing the Right Container

### 16.1 Decision table (requirement → container → DSA pattern)

| Requirement | Container | DSA pattern it maps to |
|---|---|---|
| Fast O(1) average key lookup, no ordering needed | `unordered_map` / `unordered_set` | hashing, frequency counting, memoization cache |
| Guaranteed O(log n) lookup + sorted iteration | `map` / `set` | ordered map, BIT/segment-tree alternative for small n |
| Need min/max fast, repeatedly, with updates | `priority_queue` | heap — Dijkstra, Huffman, k-way merge, top-k |
| Need min AND max fast, or arbitrary predecessor/successor | `set` / `multiset` (or two heaps) | order-statistics, running median, sliding-window max/min (with monotonic deque as a lighter alt) |
| Need duplicates allowed with sorted order | `multiset` / `multimap` | multiset order-statistics, interval scheduling with ties |
| Need insert/erase at BOTH ends, O(1) | `deque` | sliding window, monotonic deque (max/min in window) |
| LIFO access only | `stack` | DFS, expression evaluation, parenthesis matching, monotonic stack (next greater element) |
| FIFO access only | `queue` | BFS |
| Fixed small universe of flags/subsets, need bitwise DP | `bitset<N>` / raw int bitmask | subset-sum DP, bitmask DP (`dp[mask]`), reachability |
| Need contiguous memory, random access index, cache-friendly scans | `vector` | default general-purpose sequence container, DP tables |
| Need custom key hashing (pair/struct) | `unordered_map`/`unordered_set` + custom hash functor | graph coordinate visited-sets, compressed state hashing |
| Need range queries (sum/min over subrange) with updates | none of these directly — segment tree / Fenwick (BIT) | out of scope here — see algorithms part |
| Pair with a value, need existence check without insertion side-effects | `map`/`unordered_map` via `find`/`count`/`contains` | avoid the `operator[]` insertion trap (§12.2) |

### 16.2 Master complexity table — all containers (Parts II + III)

| Container | Access | Search | Insert | Erase | Ordered? | Notes |
|---|---|---|---|---|---|---|
| `vector` | O(1) | O(n) | O(1) amortized at back, O(n) elsewhere | O(n) (O(1) at back) | insertion order | contiguous, cache-friendly, may reallocate |
| `deque` | O(1) | O(n) | O(1) at both ends, O(n) middle | O(n) (O(1) at ends) | insertion order | no reallocation of existing elements on push_front/back |
| `list` (doubly linked) | O(n) | O(n) | O(1) given iterator | O(1) given iterator | insertion order | no random access; stable iterators |
| `forward_list` | O(n) | O(n) | O(1) given iterator | O(1) given iterator | insertion order | singly linked, minimal overhead |
| `stack` (adapter) | top only O(1) | — | O(1) amortized | O(1) (pop) | LIFO | no iteration |
| `queue` (adapter) | front/back O(1) | — | O(1) amortized | O(1) (pop) | FIFO | no iteration |
| `priority_queue` (adapter) | top O(1) | — | O(log n) | O(log n) (pop) | heap order (max or min) | no iteration, no decrease-key |
| `map` | — | O(log n) | O(log n) | O(log n) | sorted by key | red-black tree, ordered iteration |
| `set` | — | O(log n) | O(log n) | O(log n) | sorted | unique elements |
| `multimap` / `multiset` | — | O(log n + k) | O(log n) | O(log n + k) for value, O(1) for iterator | sorted | duplicates allowed |
| `unordered_map` | — | O(1) avg / O(n) worst | O(1) avg / O(n) worst | O(1) avg / O(n) worst | none | hash table, needs custom hash for pair/struct keys |
| `unordered_set` | — | O(1) avg / O(n) worst | O(1) avg / O(n) worst | O(1) avg / O(n) worst | none | same caveats as unordered_map |
| `bitset<N>` | O(1) per bit | O(N/64) via count/any | O(1) per bit set | O(1) per bit reset | index order | compile-time size, fastest bitwise ops |

> **Rule of thumb:** default to `vector` for sequences, `unordered_map`/`unordered_set` for lookups (fall back to `map`/`set` if you need order or face anti-hash tests), `priority_queue` for repeated min/max extraction, and `deque`/monotonic deque for sliding-window extremes. Reach for `bitset` only when the universe size is fixed and you're doing bitwise-parallelizable DP.

---

# PART IV — ALGORITHMS AND ITERATORS

## 17. Sorting

| Function | What it does | Complexity |
|---|---|---|
| `sort(first,last)` | Sorts range ascending, unstable | O(n log n) avg/worst (introsort) |
| `sort(first,last,cmp)` | Sorts with custom comparator | O(n log n) |
| `stable_sort(first,last[,cmp])` | Sorts, preserves relative order of equal elements | O(n log n) always (needs extra memory; O(n log²n) if memory alloc fails) |
| `partial_sort(first,mid,last[,cmp])` | First `[first,mid)` end up sorted (smallest `mid-first` elements), rest unspecified order | O(n log k), k = mid-first |
| `partial_sort_copy(first,last,d_first,d_last[,cmp])` | Like partial_sort but writes into another range | O(n log k) |
| `nth_element(first,nth,last[,cmp])` | Places element that *would* be at `nth` in sorted order there; everything before ≤ it, after ≥ it (each side unsorted) | O(n) average |
| `is_sorted(first,last[,cmp])` | Checks if range is sorted | O(n) |
| `is_sorted_until(first,last[,cmp])` | Returns iterator to first out-of-order element | O(n) |

`sort` is **introsort**: quicksort that falls back to heapsort if recursion gets too deep (avoids O(n²) worst case), then switches to insertion sort for small subranges. Unstable — equal elements may be reordered.

### Basic ascending/descending

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{5,2,8,1,9,3};
    sort(v.begin(), v.end());
    for (int x : v) cout << x << " "; // => 1 2 3 5 8 9
    cout << "\n";

    sort(v.begin(), v.end(), greater<int>());
    for (int x : v) cout << x << " "; // => 9 8 5 3 2 1
    cout << "\n";

    sort(v.rbegin(), v.rend()); // reverse iterators -> descending
    for (int x : v) cout << x << " "; // => 9 8 5 3 2 1
    cout << "\n";

    sort(v.begin(), v.end(), [](int a, int b){ return a > b; }); // lambda, descending
    for (int x : v) cout << x << " "; // => 9 8 5 3 2 1
}
```

### `nth_element` — O(n) kth largest/smallest

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{7,2,9,4,1,8,5};
    // 3rd smallest (0-indexed position 2)
    nth_element(v.begin(), v.begin()+2, v.end());
    cout << v[2] << "\n"; // => 4

    // kth largest: sort descending conceptually
    vector<int> w{7,2,9,4,1,8,5};
    int k = 3; // 3rd largest
    nth_element(w.begin(), w.begin()+k-1, w.end(), greater<int>());
    cout << w[k-1] << "\n"; // => 7
}
```
> **Gotcha:** `nth_element` only guarantees the pivot position is correct and partitions around it — the two sides are NOT individually sorted. Don't assume `v[0..1]` are sorted after the call above.

### Sorting a subrange

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{9,8,7,6,5,4,3};
    sort(v.begin()+2, v.begin()+5); // sort indices [2,5)
    for (int x : v) cout << x << " "; // => 9 8 5 6 7 4 3
}
```

### Custom comparators & strict weak ordering

A comparator `cmp(a,b)` must return `true` iff "a strictly precedes b". `std::sort` requires it to define a **strict weak ordering**:

1. **Irreflexive**: `cmp(a,a)` must be `false`.
2. **Asymmetric**: if `cmp(a,b)` is true, `cmp(b,a)` must be false.
3. **Transitive**: `cmp(a,b) && cmp(b,c)` ⇒ `cmp(a,c)`.
4. **Transitivity of equivalence**: if a,b are "equivalent" (`!cmp(a,b) && !cmp(b,a)`) and b,c are equivalent, then a,c must be equivalent too.

> **Gotcha:** Writing `return a <= b;` instead of `return a < b;` breaks rule 1 — `cmp(a,a)` becomes `true`. This is classic **undefined behavior**: with libstdc++/libc++'s introsort, this can manifest as infinite loops, heap corruption, or an actual **segfault** (out-of-bounds access during partitioning), especially on larger inputs or with `nth_element`/`partial_sort`. It may "work" on tiny test inputs and crash only on real judge data — always double check `<=`/`>=` in comparators.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{3,1,2};
    // WRONG (commented out — UB, may crash):
    // sort(v.begin(), v.end(), [](int a,int b){ return a <= b; });

    // CORRECT: strict comparison
    sort(v.begin(), v.end(), [](int a,int b){ return a < b; });
    for (int x : v) cout << x << " "; // => 1 2 3
}
```

### Sorting `vector<pair>` and structs, multi-key with `tie`

```cpp
#include <bits/stdc++.h>
using namespace std;
struct Person { string name; int age; double gpa; };
int main(){
    vector<pair<int,int>> pv{{2,5},{1,9},{2,1},{1,3}};
    sort(pv.begin(), pv.end()); // lexicographic by default: first, then second
    for (auto& p : pv) cout << "(" << p.first << "," << p.second << ") ";
    cout << "\n"; // => (1,3) (1,9) (2,1) (2,5)

    vector<Person> people{{"Bob",30,3.5},{"Amy",30,3.9},{"Cy",25,3.0}};
    // sort by age asc, then gpa desc, then name asc
    sort(people.begin(), people.end(), [](const Person&a, const Person&b){
        return tie(a.age, b.gpa, a.name) < tie(b.age, a.gpa, b.name);
    });
    for (auto& p : people) cout << p.name << " ";
    cout << "\n"; // => Cy Amy Bob
}
```
`tie` builds a `tuple<const T&...>` and compares lexicographically — this is the idiomatic way to do multi-key sort without writing nested if/else chains. Note the deliberate sign flip on the descending key (`b.gpa` paired with `a.gpa`).

### Sorting indices by value ("argsort")

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{40,10,30,20};
    vector<int> idx(v.size());
    iota(idx.begin(), idx.end(), 0); // idx = {0,1,2,3}
    sort(idx.begin(), idx.end(), [&](int a, int b){
        return v[a] < v[b]; // capture v by reference, compare by its values
    });
    for (int i : idx) cout << i << " "; // => 1 3 2 0  (indices of sorted v)
    cout << "\n";
    for (int i : idx) cout << v[i] << " "; // => 10 20 30 40
}
```
This "argsort" idiom (sort a permutation of indices using a lambda that indexes into the original array) is essential when you need the original positions after sorting by value.

### Sorting strings

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<string> words{"banana","Apple","cherry","apple"};
    sort(words.begin(), words.end()); // default: lexicographic, case-sensitive (ASCII: uppercase < lowercase)
    for (auto& w : words) cout << w << " "; // => Apple apple banana cherry
    cout << "\n";

    // sort by length, then lexicographically
    sort(words.begin(), words.end(), [](const string&a, const string&b){
        if (a.size() != b.size()) return a.size() < b.size();
        return a < b;
    });
    for (auto& w : words) cout << w << " "; // => Apple apple banana cherry
}
```

### Stability — when it matters

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<pair<int,char>> v{{1,'a'},{2,'b'},{1,'c'},{2,'d'}};
    stable_sort(v.begin(), v.end(), [](auto&x, auto&y){ return x.first < y.first; });
    for (auto&p : v) cout << "(" << p.first << p.second << ") ";
    // => (1a) (1c) (2b) (2d)   -- 'a' before 'c', 'b' before 'd' preserved
}
```
Use `stable_sort` when you need to preserve original relative order among equal keys (e.g., sorting by one key after already having sorted/grouped by another). `sort` gives no such guarantee — equal elements may end up in any order.

---

## 18. Binary Search Family

All of these require the range to be **partitioned** with respect to the predicate (for the plain overloads: sorted ascending). `lower_bound`/`upper_bound`/`equal_range` work correctly on arrays with duplicates and return *boundary* iterators, not "found" indices.

Sample array (0-indexed, **with duplicates**): `a = {1, 3, 3, 3, 5, 7, 9}`, indices `0..6`.

| Call | Meaning | Result on `a` |
|---|---|---|
| `binary_search(a.begin(),a.end(),3)` | Does 3 exist? | `true` |
| `binary_search(a.begin(),a.end(),4)` | Does 4 exist? | `false` |
| `lower_bound(a.begin(),a.end(),3)` | First position where 3 *could* be inserted keeping order = first index with value ≥ 3 | iterator to index **1** (value 3) |
| `upper_bound(a.begin(),a.end(),3)` | First index with value > 3 | iterator to index **4** (value 5) |
| `equal_range(a.begin(),a.end(),3)` | `{lower_bound, upper_bound}` pair | `{it(1), it(4)}` |
| `lower_bound(...,4)` | First index with value ≥ 4 | iterator to index **4** (value 5) — 4 not present |
| `upper_bound(...,4)` | First index with value > 4 | iterator to index **4** (value 5) — same as lower_bound since 4 absent |
| `lower_bound(...,0)` | First index with value ≥ 0 | iterator to index **0** (begin) |
| `lower_bound(...,10)` | First index with value ≥ 10 | `a.end()` (index 7, one-past-end) |

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> a{1,3,3,3,5,7,9};
    auto lo = lower_bound(a.begin(), a.end(), 3);
    auto hi = upper_bound(a.begin(), a.end(), 3);
    cout << (lo - a.begin()) << " " << (hi - a.begin()) << "\n"; // => 1 4
    cout << "count of 3 = " << (hi - lo) << "\n"; // => 3

    bool found = binary_search(a.begin(), a.end(), 4);
    cout << found << "\n"; // => 0

    auto it10 = lower_bound(a.begin(), a.end(), 10);
    cout << (it10 == a.end()) << "\n"; // => 1
}
```

### iterator → index, and the `end()` guard

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> a{1,3,3,3,5,7,9};
    int target = 5;
    auto it = lower_bound(a.begin(), a.end(), target);
    if (it != a.end() && *it == target) {
        int idx = it - a.begin(); // pointer/iterator arithmetic -> index
        cout << "found at " << idx << "\n"; // => found at 4
    } else {
        cout << "not found\n";
    }
}
```
> **Gotcha:** `*a.end()` is undefined behavior — always check `it != end()` before dereferencing. `lower_bound` never returns "not found" as a sentinel other than `end()` (or the correct insertion point), so you MUST additionally compare `*it == target` to distinguish "found" from "would insert here".

### Member functions on `set`/`map` vs `std::` free functions

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    set<int> s{1,3,5,7,9};

    // CORRECT: O(log n) — uses the tree structure
    auto it1 = s.lower_bound(5);
    cout << *it1 << "\n"; // => 5

    // WRONG habit: O(n) — std::lower_bound only knows RandomAccessIterator tricks
    // for jump-size; on a set's bidirectional iterators it degrades to
    // linear advancement per step, effectively O(n) despite looking like binary search.
    auto it2 = lower_bound(s.begin(), s.end(), 5);
    cout << *it2 << "\n"; // => 5 (same answer, MUCH slower asymptotically)
}
```
> **Gotcha (classic perf trap):** `set`/`map`/`multiset`/`multimap` iterators are only **bidirectional**, not random-access. `std::lower_bound` computes midpoints via `advance(it, n/2)`, which on a bidirectional iterator means walking one node at a time — turning the "binary search" into O(n) per call, O(n log n) or worse overall. **Always use the container's own `.lower_bound()`/`.upper_bound()`/`.equal_range()`/`.find()` member functions on `set`/`map`** — those are true O(log n) tree descents. This bug silently passes small tests and TLEs on large ones.

### Counting occurrences

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> a{1,3,3,3,5,7,9};
    int cnt = upper_bound(a.begin(),a.end(),3) - lower_bound(a.begin(),a.end(),3);
    cout << cnt << "\n"; // => 3
}
```

### Predecessor / successor idioms

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    set<int> s{1,3,5,7,9};
    int x = 6;

    // successor: smallest element >= x
    auto succ = s.lower_bound(x);
    if (succ != s.end()) cout << "succ=" << *succ << "\n"; // => succ=7

    // predecessor: largest element < x
    auto pred = s.lower_bound(x);
    if (pred != s.begin()) { --pred; cout << "pred=" << *pred << "\n"; } // => pred=5
    else cout << "no predecessor\n";
}
```

### `lower_bound` with custom comparator, and on a descending array

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    // descending array: {9,7,5,3,3,3,1}
    vector<int> d{9,7,5,3,3,3,1};
    // must supply greater<int>() so the algorithm knows the ordering it's searching under
    auto it = lower_bound(d.begin(), d.end(), 5, greater<int>());
    cout << (it - d.begin()) << "\n"; // => 2 (first position where 5 "belongs" under descending order)

    // custom comparator on struct: find first person with age >= 30
    struct P { string name; int age; };
    vector<P> people{{"A",20},{"B",25},{"C",30},{"D",35}};
    auto it2 = lower_bound(people.begin(), people.end(), 30,
                           [](const P& p, int age){ return p.age < age; });
    cout << it2->name << "\n"; // => C
}
```
> **Gotcha:** The comparator passed to `lower_bound` must express the SAME ordering the range is actually sorted under. Using default `<` on a descending array (or omitting the comparator) gives silently wrong results, not a crash — the trickiest kind of bug.

### Hand-rolled binary search templates

```cpp
#include <bits/stdc++.h>
using namespace std;

// [lo, hi) half-open form: finds first index where pred(idx) is true,
// assuming pred is false...false,true...true (monotonic)
int firstTrue_halfOpen(int lo, int hi, function<bool(int)> pred) {
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2; // avoids overflow vs (lo+hi)/2
        if (pred(mid)) hi = mid;
        else lo = mid + 1;
    }
    return lo; // == hi; first true, or hi if none
}

// [lo, hi] closed form
int firstTrue_closed(int lo, int hi, function<bool(int)> pred) {
    int ans = hi + 1; // sentinel: "not found"
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (pred(mid)) { ans = mid; hi = mid - 1; }
        else lo = mid + 1;
    }
    return ans;
}

int main(){
    vector<int> a{1,3,3,3,5,7,9};
    // first index with a[idx] >= 5, using half-open form over [0, a.size())
    int i1 = firstTrue_halfOpen(0, (int)a.size(), [&](int idx){ return a[idx] >= 5; });
    cout << i1 << "\n"; // => 4

    int i2 = firstTrue_closed(0, (int)a.size()-1, [&](int idx){ return a[idx] >= 5; });
    cout << i2 << "\n"; // => 4
}
```

### Binary search on the answer — C++ mechanics only

Same template as above, but `pred` checks feasibility of a numeric answer rather than indexing an array (e.g., "can we finish within `mid` days?"). The C++ shape is identical: monotonic predicate + `[lo,hi]` or `[lo,hi)` narrowing loop. For the algorithmic *pattern* (how to design the predicate, proving monotonicity, typical problem shapes), see **Pattern 05 — Binary Search on the Answer** elsewhere in this reference.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    // toy example: smallest x such that x*x >= 50
    long long lo = 0, hi = 100000;
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (mid * mid >= 50) hi = mid;
        else lo = mid + 1;
    }
    cout << lo << "\n"; // => 8   (8*8=64>=50, 7*7=49<50)
}
```

### `partition_point`

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,1,1,2,2,3,3,3,3}; // partitioned: true-block then false-block by pred
    // find first element that is NOT < 3, i.e., first index where "< 3" becomes false
    auto it = partition_point(v.begin(), v.end(), [](int x){ return x < 3; });
    cout << (it - v.begin()) << "\n"; // => 5
}
```
`partition_point(first,last,pred)` is the generalization of `lower_bound`: given a range already partitioned so `pred` is true for a prefix and false after, it binary-searches for the boundary in O(log n). Equivalent to `lower_bound` when `pred(x) = x < target`.

---

## 19. The Rest of `<algorithm>` and `<numeric>`

| Function | What it does | Complexity |
|---|---|---|
| `min(a,b)`, `max(a,b)` | Pairwise min/max | O(1) |
| `min({a,b,c,...})`, `max({...})` | Min/max over an init-list, any count of args | O(n) in list size |
| `minmax(a,b)` | Returns `pair<T,T>{min,max}` | O(1) |
| `min_element/max_element(first,last[,cmp])` | Iterator to min/max in range | O(n) |
| `minmax_element(first,last[,cmp])` | `pair` of iterators `{min_it,max_it}` | O(n) |
| `count(first,last,val)` | Count elements == val | O(n) |
| `count_if(first,last,pred)` | Count elements satisfying pred | O(n) |
| `find(first,last,val)` | Iterator to first == val, else `last` | O(n) |
| `find_if / find_if_not(first,last,pred)` | Iterator to first (not) satisfying pred | O(n) |
| `adjacent_find(first,last[,cmp])` | Iterator to first pair of adjacent equal elements | O(n) |
| `search(first,last,s_first,s_last)` | Find subrange (subsequence) occurrence | O(n·m) naive |
| `all_of/any_of/none_of(first,last,pred)` | Predicate over whole range | O(n) |
| `for_each(first,last,fn)` | Apply fn to each element | O(n) |
| `copy(first,last,d_first)` | Copy range | O(n) |
| `copy_if(first,last,d_first,pred)` | Copy matching elements | O(n) |
| `fill(first,last,val)` / `fill_n(first,n,val)` | Assign val to range | O(n) |
| `generate(first,last,fn)` / `generate_n` | Assign `fn()` result to each | O(n) |
| `transform(first,last,d_first,fn)` | Apply fn, write results (1-range) | O(n) |
| `transform(first1,last1,first2,d_first,fn)` | Apply binary fn over two ranges | O(n) |
| `remove(first,last,val)` / `remove_if(first,last,pred)` | Shift non-matching to front, return new logical end (does NOT resize container) | O(n) |
| `unique(first,last[,cmp])` | Removes *consecutive* duplicates, returns new logical end | O(n) |
| `reverse(first,last)` | Reverse in place | O(n) |
| `rotate(first,mid,last)` | Rotate so `mid` becomes new first | O(n) |
| `swap(a,b)` / `iter_swap(it1,it2)` | Swap values | O(1) |
| `next_permutation(first,last[,cmp])` / `prev_permutation` | Rearrange to next/prev permutation, return false if wrapped | O(n) |
| `accumulate(first,last,init[,op])` | Fold/reduce with `+` (or custom op) | O(n) |
| `partial_sum(first,last,d_first[,op])` | Prefix sums | O(n) |
| `inner_product(first1,last1,first2,init[,op1,op2])` | Dot-product-style fold over two ranges | O(n) |
| `iota(first,last,start)` | Fill with `start, start+1, start+2, ...` | O(n) |
| `gcd(a,b)` / `lcm(a,b)` **[C++17]** | Greatest common divisor / least common multiple | O(log min(a,b)) / O(log) |
| `set_union/intersection/difference(...)` | Set ops on **sorted** ranges, write to output | O(n+m) |
| `merge(first1,last1,first2,last2,d_first)` | Merge two sorted ranges | O(n+m) |
| `includes(first1,last1,first2,last2)` | Is range2 a subset of sorted range1? | O(n+m) |
| `make_heap/push_heap/pop_heap/sort_heap/is_heap` | Manual heap ops on a vector (max-heap by default) | O(n)/O(log n)/O(log n)/O(n log n)/O(n) |
| `clamp(v,lo,hi)` **[C++17]** | Clamp v into `[lo,hi]` | O(1) |
| `shuffle(first,last,rng)` | Random shuffle (needs a URNG like `mt19937`) | O(n) |
| `equal(first1,last1,first2[,cmp])` | Elementwise equality of two ranges | O(n) |
| `lexicographical_compare(first1,last1,first2,last2)` | Dictionary-order comparison of two ranges | O(n) |
| `distance(first,last)` | Number of steps between iterators | O(n) generic, O(1) random-access |
| `advance(it,n)` | Move iterator by n steps | O(n) generic, O(1) random-access |
| `next(it,n=1)` / `prev(it,n=1)` | Return advanced/receded copy (non-mutating) | same as advance |

### min/max family

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    cout << min(3,7) << " " << max(3,7) << "\n"; // => 3 7
    cout << min({5,2,9,1,7}) << " " << max({5,2,9,1,7}) << "\n"; // => 1 9

    auto [mn, mx] = minmax(4, 9);
    cout << mn << " " << mx << "\n"; // => 4 9

    vector<int> v{5,2,9,1,7};
    auto itMin = min_element(v.begin(), v.end());
    auto itMax = max_element(v.begin(), v.end());
    cout << *itMin << " " << *itMax << "\n"; // => 1 9

    auto [itMn, itMx] = minmax_element(v.begin(), v.end());
    cout << *itMn << " " << *itMx << "\n"; // => 1 9
}
```

### count / find family

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,2,3,2,4};
    cout << count(v.begin(), v.end(), 2) << "\n"; // => 3
    cout << count_if(v.begin(), v.end(), [](int x){ return x%2==0; }) << "\n"; // => 4

    auto it = find(v.begin(), v.end(), 3);
    cout << (it != v.end() ? *it : -1) << "\n"; // => 3

    auto it2 = find_if(v.begin(), v.end(), [](int x){ return x > 2; });
    cout << *it2 << "\n"; // => 3

    auto it3 = find_if_not(v.begin(), v.end(), [](int x){ return x < 3; });
    cout << *it3 << "\n"; // => 3

    vector<int> w{1,2,2,3,4};
    auto it4 = adjacent_find(w.begin(), w.end());
    cout << (it4 - w.begin()) << "\n"; // => 1 (index of first of the adjacent-equal pair)
}
```

### search, all_of/any_of/none_of, for_each

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,4,5,6};
    vector<int> pat{3,4,5};
    auto it = search(v.begin(), v.end(), pat.begin(), pat.end());
    cout << (it - v.begin()) << "\n"; // => 2

    cout << all_of(v.begin(), v.end(), [](int x){ return x > 0; }) << "\n"; // => 1
    cout << any_of(v.begin(), v.end(), [](int x){ return x == 10; }) << "\n"; // => 0
    cout << none_of(v.begin(), v.end(), [](int x){ return x < 0; }) << "\n"; // => 1

    for_each(v.begin(), v.end(), [](int x){ cout << x*x << " "; });
    cout << "\n"; // => 1 4 9 16 25 36
}
```

### copy / copy_if / back_inserter

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> src{1,2,3,4,5};
    vector<int> dst(5);
    copy(src.begin(), src.end(), dst.begin());
    for (int x : dst) cout << x << " "; cout << "\n"; // => 1 2 3 4 5

    vector<int> evens;
    copy_if(src.begin(), src.end(), back_inserter(evens), [](int x){ return x%2==0; });
    for (int x : evens) cout << x << " "; cout << "\n"; // => 2 4
}
```
> **Gotcha:** `copy`/`copy_if`/`transform` into `dst.begin()` require `dst` to already have enough space (no auto-growth). Use `back_inserter(dst)` to append-and-grow instead, or `dst.resize(n)` beforehand.

### fill, generate, transform

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v(5);
    fill(v.begin(), v.end(), 7);
    for (int x : v) cout << x << " "; cout << "\n"; // => 7 7 7 7 7

    fill_n(v.begin(), 3, 9);
    for (int x : v) cout << x << " "; cout << "\n"; // => 9 9 9 7 7

    int counter = 0;
    generate(v.begin(), v.end(), [&counter](){ return counter++; });
    for (int x : v) cout << x << " "; cout << "\n"; // => 0 1 2 3 4

    vector<int> out(v.size());
    transform(v.begin(), v.end(), out.begin(), [](int x){ return x*x; });
    for (int x : out) cout << x << " "; cout << "\n"; // => 0 1 4 9 16

    vector<int> a{1,2,3}, b{10,20,30}, sum(3);
    transform(a.begin(), a.end(), b.begin(), sum.begin(), plus<int>());
    for (int x : sum) cout << x << " "; cout << "\n"; // => 11 22 33
}
```

### remove/remove_if — the erase-remove idiom (restated)

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,2,4,2,5};
    auto newEnd = remove(v.begin(), v.end(), 2); // does NOT shrink the vector!
    cout << v.size() << "\n"; // => 7  (still 7 — remove only shuffles elements)
    v.erase(newEnd, v.end());  // actually shrinks
    for (int x : v) cout << x << " "; cout << "\n"; // => 1 3 4 5

    vector<int> w{1,2,3,4,5,6};
    w.erase(remove_if(w.begin(), w.end(), [](int x){ return x%2==0; }), w.end());
    for (int x : w) cout << x << " "; cout << "\n"; // => 1 3 5
}
```
> **Gotcha:** `remove`/`remove_if` never change container *size* — they move "kept" elements to the front and leave the tail in a valid-but-unspecified state, returning an iterator marking the new logical end. You MUST follow with `.erase(newEnd, container.end())` to actually shrink it. Forgetting the `.erase()` is one of the most common STL bugs.

### unique — needs sorted input to fully dedupe

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,1,2,3,3,3,1,1}; // NOT sorted globally
    auto newEnd = unique(v.begin(), v.end()); // only removes CONSECUTIVE dups
    v.erase(newEnd, v.end());
    for (int x : v) cout << x << " "; cout << "\n"; // => 1 2 3 1  (the trailing 1s survive as a separate group!)

    vector<int> w{3,1,1,2,3,3,1};
    sort(w.begin(), w.end());                 // {1,1,1,2,3,3,3}
    w.erase(unique(w.begin(), w.end()), w.end());
    for (int x : w) cout << x << " "; cout << "\n"; // => 1 2 3
}
```
> **Gotcha (classic bug):** `unique` only collapses **consecutive** equal runs. To dedupe an entire container down to a set of distinct values, you must `sort` first, then `unique` + `erase`. Calling `unique` on unsorted data silently gives wrong (partial) dedup — a very common competitive-programming bug (`{1,2,1}` → `unique` gives `{1,2,1}` unchanged since no two adjacent elements match).

### reverse, rotate, swap/iter_swap

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,4,5};
    reverse(v.begin(), v.end());
    for (int x : v) cout << x << " "; cout << "\n"; // => 5 4 3 2 1

    vector<int> w{1,2,3,4,5};
    rotate(w.begin(), w.begin()+2, w.end()); // new first = old index 2
    for (int x : w) cout << x << " "; cout << "\n"; // => 3 4 5 1 2

    int a=1, b=2;
    swap(a,b);
    cout << a << " " << b << "\n"; // => 2 1

    vector<int> u{10,20,30};
    iter_swap(u.begin(), u.begin()+2);
    for (int x : u) cout << x << " "; cout << "\n"; // => 30 20 10
}
```

### next_permutation / prev_permutation — full generation loop

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3};
    sort(v.begin(), v.end()); // MUST start sorted (smallest permutation) to enumerate ALL of them
    do {
        for (int x : v) cout << x;
        cout << " ";
    } while (next_permutation(v.begin(), v.end()));
    cout << "\n";
    // => 123 132 213 231 312 321
}
```
`next_permutation` rearranges to the lexicographically next permutation and returns `false` (wrapping to the smallest) when the current one is the last. To enumerate every permutation, the range must **start sorted ascending** — otherwise you'll only see a subset (from the current permutation onward until it wraps).

### accumulate — the init-type overflow trap

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v(100000, 100000); // sum = 10,000,000,000 -> overflows int32

    // WRONG: init is `0` (an int) -> accumulation done in int -> overflow/UB
    int badSum = accumulate(v.begin(), v.end(), 0);
    cout << badSum << "\n"; // => some garbage/wrapped value, NOT 10000000000

    // CORRECT: pass 0LL so accumulation is done in long long
    long long goodSum = accumulate(v.begin(), v.end(), 0LL);
    cout << goodSum << "\n"; // => 10000000000

    // custom op: product
    vector<int> nums{1,2,3,4};
    long long prod = accumulate(nums.begin(), nums.end(), 1LL, multiplies<long long>());
    cout << prod << "\n"; // => 24
}
```
> **Gotcha:** `accumulate`'s return/accumulation type is deduced from the **init value's type**, not the container's element type. Passing `0` for a sum that overflows `int` is a silent, extremely common bug — always pass `0LL` (or an explicitly-typed variable) when the sum could exceed ~2×10^9.

### partial_sum, inner_product, iota

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,4};
    vector<int> ps(4);
    partial_sum(v.begin(), v.end(), ps.begin());
    for (int x : ps) cout << x << " "; cout << "\n"; // => 1 3 6 10

    vector<int> a{1,2,3}, b{4,5,6};
    long long dot = inner_product(a.begin(), a.end(), b.begin(), 0LL);
    cout << dot << "\n"; // => 32   (1*4+2*5+3*6)

    vector<int> ids(5);
    iota(ids.begin(), ids.end(), 10);
    for (int x : ids) cout << x << " "; cout << "\n"; // => 10 11 12 13 14
}
```

### gcd/lcm **[C++17]** — `<numeric>`

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    cout << gcd(12, 18) << " " << lcm(4, 6) << "\n"; // => 6 12
}
```

### set operations on sorted ranges

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> a{1,2,3,4,5}, b{3,4,5,6,7}; // both MUST be sorted
    vector<int> out;

    out.clear();
    set_union(a.begin(),a.end(), b.begin(),b.end(), back_inserter(out));
    for (int x : out) cout << x << " "; cout << "\n"; // => 1 2 3 4 5 6 7

    out.clear();
    set_intersection(a.begin(),a.end(), b.begin(),b.end(), back_inserter(out));
    for (int x : out) cout << x << " "; cout << "\n"; // => 3 4 5

    out.clear();
    set_difference(a.begin(),a.end(), b.begin(),b.end(), back_inserter(out));
    for (int x : out) cout << x << " "; cout << "\n"; // => 1 2

    out.clear();
    merge(a.begin(),a.end(), b.begin(),b.end(), back_inserter(out));
    for (int x : out) cout << x << " "; cout << "\n"; // => 1 2 3 3 4 4 5 5 6 7

    cout << includes(a.begin(),a.end(), vector<int>{2,3}.begin(), vector<int>{2,3}.end()) << "\n"; // => 1
}
```
> **Gotcha:** All of `set_union/intersection/difference/merge/includes` **require sorted input ranges** — on unsorted data they silently produce wrong (garbage) output, no error/crash.

### heap functions

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> h{3,1,4,1,5,9,2,6};
    make_heap(h.begin(), h.end()); // O(n), max-heap by default
    cout << h.front() << "\n"; // => 9 (largest at front)
    cout << is_heap(h.begin(), h.end()) << "\n"; // => 1

    h.push_back(10);
    push_heap(h.begin(), h.end()); // re-heapify after appending at back
    cout << h.front() << "\n"; // => 10

    pop_heap(h.begin(), h.end()); // moves max to h.back(), rest still a heap
    cout << h.back() << "\n"; // => 10
    h.pop_back(); // actually remove it

    sort_heap(h.begin(), h.end()); // consumes heap property, sorts ascending
    for (int x : h) cout << x << " "; cout << "\n"; // => 1 1 2 3 4 5 6 9
}
```

### clamp **[C++17]**

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    cout << clamp(15, 0, 10) << " " << clamp(-5, 0, 10) << " " << clamp(5, 0, 10) << "\n";
    // => 10 0 5
}
```

### shuffle + mt19937

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,4,5};
    mt19937 rng(12345); // seeded for reproducibility
    shuffle(v.begin(), v.end(), rng);
    for (int x : v) cout << x << " "; cout << "\n"; // => (some fixed permutation for seed 12345)
}
```
> **Gotcha:** `random_shuffle` was removed in C++17 — use `shuffle` with an explicit URNG (`mt19937`), never plain `rand()`-based shuffling for anything you care about being unbiased.

### equal, lexicographical_compare

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> a{1,2,3}, b{1,2,3}, c{1,2,4};
    cout << equal(a.begin(),a.end(), b.begin()) << "\n"; // => 1
    cout << equal(a.begin(),a.end(), c.begin()) << "\n"; // => 0

    cout << lexicographical_compare(a.begin(),a.end(), c.begin(),c.end()) << "\n"; // => 1 (a < c)
}
```

### distance/advance/next/prev

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    list<int> lst{1,2,3,4,5};
    auto it = lst.begin();
    advance(it, 3); // mutates it in place
    cout << *it << "\n"; // => 4

    auto it2 = next(lst.begin(), 2); // returns a NEW iterator, doesn't mutate lst.begin()
    cout << *it2 << "\n"; // => 3

    auto it3 = prev(lst.end(), 1);
    cout << *it3 << "\n"; // => 5

    cout << distance(lst.begin(), lst.end()) << "\n"; // => 5
}
```

---

## 20. Iterators

### Categories and which containers provide them

| Category | Capabilities | Provided by |
|---|---|---|
| Input | Single-pass, read-only, `++`, `*` | `istream_iterator`, streams |
| Output | Single-pass, write-only, `++`, `*` | `ostream_iterator`, `back_inserter` |
| Forward | Multi-pass, read/write, `++` only | `forward_list` |
| Bidirectional | Forward + `--` | `list`, `set`, `map`, `multiset`, `multimap` |
| Random-access | Bidirectional + `+n`, `-n`, `[]`, `<`, full arithmetic | `vector`, `deque`, `array`, raw pointers/C-arrays, `string` |

`unordered_set`/`unordered_map`/`unordered_multiset`/`unordered_multimap` provide **forward** iterators only (no `--`).

> **Gotcha:** `std::sort` requires **random-access** iterators (it needs O(1) `it + n` to do quicksort partitioning/pivot jumps). `list` only offers bidirectional iterators, so `sort(lst.begin(), lst.end())` **does not compile**. Use the container's own `lst.sort()` method instead (a merge-sort adapted for linked lists, stable, O(n log n), works via pointer relinking without random access).

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    list<int> lst{5,3,1,4,2};
    // sort(lst.begin(), lst.end()); // COMPILE ERROR: list iterators aren't random-access
    lst.sort(); // correct: member function
    for (int x : lst) cout << x << " "; cout << "\n"; // => 1 2 3 4 5

    lst.sort(greater<int>()); // member sort also accepts a comparator
    for (int x : lst) cout << x << " "; cout << "\n"; // => 5 4 3 2 1
}
```

### begin/end, cbegin/cend, rbegin/rend

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3};
    for (auto it = v.cbegin(); it != v.cend(); ++it) cout << *it << " "; // const_iterator, can't modify
    cout << "\n"; // => 1 2 3

    for (auto it = v.rbegin(); it != v.rend(); ++it) cout << *it << " "; // reverse traversal
    cout << "\n"; // => 3 2 1
}
```

### Iterator arithmetic legality per category

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,4,5};
    auto it = v.begin();
    it += 2;         // OK: random-access supports +=
    cout << *it << "\n"; // => 3
    cout << (v.end() - v.begin()) << "\n"; // => 5, random-access supports subtraction

    list<int> lst{1,2,3};
    auto lit = lst.begin();
    ++lit;            // OK: all iterators support ++
    // lit += 2;       // COMPILE ERROR: list iterators aren't random-access
    advance(lit, 1);   // correct generic way to move a non-random-access iterator by n
    cout << *lit << "\n"; // => 3
}
```

### Inserters: `back_inserter`, `inserter`

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> src{1,2,3};
    vector<int> dst;
    copy(src.begin(), src.end(), back_inserter(dst)); // appends via push_back
    for (int x : dst) cout << x << " "; cout << "\n"; // => 1 2 3

    set<int> s{5,6};
    copy(src.begin(), src.end(), inserter(s, s.begin())); // inserts via s.insert(); position hint ignored for set
    for (int x : s) cout << x << " "; cout << "\n"; // => 1 2 3 5 6
}
```

### Iterator invalidation — complete table per container

| Container | Operation | What gets invalidated |
|---|---|---|
| `vector` | `push_back`/`insert` (no realloc, capacity sufficient) | Iterators/refs to elements **at or after** insertion point invalidated; elements before are fine |
| `vector` | `push_back`/`insert` (triggers realloc) | **ALL** iterators, pointers, and references invalidated |
| `vector` | `pop_back` | Iterator/reference to the removed (last) element invalidated; `end()` invalidated |
| `vector` | `erase(it)` | Iterators/refs at or after erased position invalidated; `end()` invalidated |
| `vector` | `reserve`/`resize` causing realloc | All iterators/pointers/references invalidated |
| `vector` | `clear()` | All iterators/pointers/references invalidated |
| `deque` | `push_back`/`push_front` | Iterators may ALL be invalidated (implementation-defined); **references** to existing elements usually remain valid |
| `deque` | `insert` (middle) | All iterators invalidated; refs to elements not moved may survive |
| `deque` | `pop_back`/`pop_front` | Only iterators/refs to the removed element invalidated |
| `list` | any insert | Nothing invalidated (no reallocation, node-based) |
| `list` | `erase(it)` | Only the iterator to the erased element invalidated; all others remain valid |
| `map`/`set` | `insert` | Nothing invalidated (node-based tree, no rehash/relocation) |
| `map`/`set` | `erase(it)` | Only the iterator to the erased element invalidated; all others remain valid |
| `unordered_map`/`unordered_set` | `insert` (no rehash) | Iterators invalidated, but references/pointers to existing elements remain valid |
| `unordered_map`/`unordered_set` | `insert` (triggers rehash) | **ALL iterators** invalidated; references/pointers to existing elements remain valid |
| `unordered_map`/`unordered_set` | `erase(it)` | Only the iterator to the erased element invalidated |

> **Gotcha (the #1 iterator-invalidation bug):** erasing from a `vector`/`deque` while iterating with a plain for-loop that also does `++it` afterward:
```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3,4,5};
    // WRONG pattern (would skip elements or, if erase returns end unexpectedly, UB on ++it beyond end):
    // for (auto it = v.begin(); it != v.end(); ++it) { if (*it % 2 == 0) v.erase(it); }

    // CORRECT: use erase's return value (points to the element after the erased one)
    for (auto it = v.begin(); it != v.end(); ) {
        if (*it % 2 == 0) it = v.erase(it);
        else ++it;
    }
    for (int x : v) cout << x << " "; cout << "\n"; // => 1 3 5
}
```
For `map`/`set`, the classic safe-erase-while-iterating idiom:
```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    set<int> s{1,2,3,4,5};
    for (auto it = s.begin(); it != s.end(); ) {
        if (*it % 2 == 0) it = s.erase(it); // erase returns iterator to next element
        else ++it;
    }
    for (int x : s) cout << x << " "; cout << "\n"; // => 1 3 5
}
```

### The half-open `[first,last)` convention

Every standard range is `[first, last)` — `last` is **one past the end**, never dereferenced. This is why `end()` is a valid iterator to *hold* (for comparison) but never valid to *dereference*:

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{1,2,3};
    for (auto it = v.begin(); it != v.end(); ++it) cout << *it << " ";
    cout << "\n"; // => 1 2 3
    // *v.end(); // UB — one-past-the-end is not a valid element
}
```
This convention is why empty ranges are naturally `first == last`, and why sizes are `last - first` with no off-by-one adjustment.

### **[C++20]** Ranges brief practical intro

Available since C++20 (`<ranges>`). **Judge-availability note:** many online judges (older Codeforces custom-test compilers, some ICPC-style judges) still default to C++17 or an older GCC without full ranges support — verify the judge's C++ version/compiler flags before relying on this in a contest.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{5,3,8,1,9,2,7};

    // ranges::sort — no need to pass .begin()/.end() separately
    ranges::sort(v);
    for (int x : v) cout << x << " "; cout << "\n"; // => 1 2 3 5 7 8 9

    // views::filter + views::transform — lazy, composable pipelines
    auto result = v
        | views::filter([](int x){ return x % 2 == 1; })
        | views::transform([](int x){ return x * x; });
    for (int x : result) cout << x << " "; cout << "\n"; // => 1 9 25 49 81
}
```
Requires `#include <ranges>` (pulled in by `bits/stdc++.h` on recent GCC) and `-std=c++20`. Prefer the classic `<algorithm>` forms shown above for portability across judges unless you've confirmed C++20 support.

---

## 21. Comparators — The Complete Mental Model

### What a comparator means

`cmp(a, b) == true` means **"a comes strictly before b"** in whatever ordering you're imposing — nothing more, nothing less. It is NOT "a is less than or equal to b", and it is NOT "a should be placed above/below b in a heap" (that's a derived consequence, not the definition).

### Strict weak ordering rules (restated with the `<=` trap)

1. **Irreflexive**: `cmp(x,x)` is always `false`.
2. **Asymmetric**: `cmp(a,b) == true` implies `cmp(b,a) == false`.
3. **Transitive**: `cmp(a,b) && cmp(b,c)` implies `cmp(a,c)`.

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    // ILLEGAL comparator: uses <=, violates irreflexivity (cmp(x,x) == true)
    auto badCmp = [](int a, int b){ return a <= b; };
    cout << badCmp(5,5) << "\n"; // => 1  <-- violates rule 1, this is the smoking gun

    // LEGAL comparator
    auto goodCmp = [](int a, int b){ return a < b; };
    cout << goodCmp(5,5) << "\n"; // => 0
}
```
> **Gotcha:** Passing `badCmp` to `sort`, `set`, `priority_queue`, or `stable_sort` is undefined behavior. In practice with libstdc++'s introsort this frequently manifests as **out-of-bounds reads/writes during partitioning — a real segfault** — once the input is large enough to trigger the quicksort/introsort code paths (small inputs that only hit insertion-sort fallback may "work by luck"). Never use `<=`/`>=` in a strict comparator; always use strict `<`/`>`.

### Why the SAME comparator flips behavior: `sort` vs `priority_queue`

`sort(first,last,cmp)` with `cmp = less<int>()` (the default) produces **ascending** order — the "smallest" element (per cmp) ends up first.

`priority_queue<int, vector<int>, cmp>` is a **max-heap under `cmp`** — it puts the element that is "greatest according to cmp" on top. With `cmp = less<int>()` (the default for `priority_queue`), that means the numerically **largest** value is on top — this matches intuition. But swap in `greater<int>()` for both, and the *direction* each container produces flips **relative to each other's default**:

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    vector<int> v{5,1,4,2,8};

    // sort with greater<int>(): descending order, largest FIRST in the sequence
    vector<int> s1 = v;
    sort(s1.begin(), s1.end(), greater<int>());
    cout << "sort w/ greater: ";
    for (int x : s1) cout << x << " "; cout << "\n"; // => 8 5 4 2 1

    // priority_queue with greater<int>(): this makes it a MIN-heap
    // (top() gives the SMALLEST element, opposite of default max-heap)
    priority_queue<int, vector<int>, greater<int>> pq(v.begin(), v.end());
    cout << "pq top w/ greater: " << pq.top() << "\n"; // => 1

    // priority_queue with default less<int>(): MAX-heap, top() gives LARGEST
    priority_queue<int> pq2(v.begin(), v.end());
    cout << "pq top w/ default less: " << pq2.top() << "\n"; // => 8
}
```
The mental model that resolves the apparent contradiction: `priority_queue`'s `top()` always returns "the element that `cmp` would rank as *not less than* everything else" — i.e., the **maximum under `cmp`**. With `cmp = greater<int>()`, "maximum under greater-than" is the numerically *smallest* value — hence `greater<int>()` turns `priority_queue` into a **min-heap**, while it turns `sort` into **descending order**. Both are internally consistent with "cmp(a,b) means a comes before b / a is less-ranked than b" — but "before" in a sorted sequence (front = least) vs "less-ranked" in a heap (top = greatest-ranked, i.e., NOT the "before" side) point in visually opposite directions for a human reading the output.

### Comparator as lambda vs functor struct vs `operator<` vs `greater<>`/`less<>`

```cpp
#include <bits/stdc++.h>
using namespace std;

// 1. functor struct
struct CmpFunctor {
    bool operator()(int a, int b) const { return a > b; } // descending
};

// 2. free-standing struct with operator< overloaded for a custom type (used implicitly by default less<>)
struct Point { int x, y; };
bool operator<(const Point& a, const Point& b) { return tie(a.x,a.y) < tie(b.x,b.y); }

int main(){
    vector<int> v{3,1,2};

    // lambda
    sort(v.begin(), v.end(), [](int a, int b){ return a > b; });
    for (int x : v) cout << x << " "; cout << "\n"; // => 3 2 1

    // functor struct
    sort(v.begin(), v.end(), CmpFunctor());
    for (int x : v) cout << x << " "; cout << "\n"; // => 3 2 1

    // built-in greater<>/less<>
    sort(v.begin(), v.end(), greater<int>());
    for (int x : v) cout << x << " "; cout << "\n"; // => 3 2 1

    // custom type relying on operator< (no comparator argument needed)
    vector<Point> pts{{2,1},{1,5},{1,2}};
    sort(pts.begin(), pts.end()); // uses operator< defined above
    for (auto& p : pts) cout << "(" << p.x << "," << p.y << ") "; cout << "\n";
    // => (1,2) (1,5) (2,1)
}
```

### The THREE DIFFERENT SYNTAXES — comparator placement

| Container | Comparator goes where | Example |
|---|---|---|
| `sort` / `stable_sort` / etc. (free algorithm) | Extra **function argument** | `sort(v.begin(), v.end(), greater<int>())` |
| `set<T,Cmp>` / `map<K,V,Cmp>` / `multiset` / `multimap` | **Template type parameter** (part of the container's type) | `set<int, greater<int>> s;` |
| `priority_queue<T,Container,Cmp>` | **Template type parameter**, AND container type must be specified too, AND `Cmp` needs a default-constructible instance (or pass one to the ctor) | `priority_queue<int, vector<int>, greater<int>> pq;` |

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    // 1) sort: comparator as an ARGUMENT
    vector<int> v{3,1,2};
    sort(v.begin(), v.end(), greater<int>());
    for (int x : v) cout << x << " "; cout << "\n"; // => 3 2 1

    // 2) set/map: comparator as a TYPE PARAMETER (third template arg)
    set<int, greater<int>> s{3,1,2};
    for (int x : s) cout << x << " "; cout << "\n"; // => 3 2 1

    // 3) priority_queue: TYPE PARAMETER, and needs the underlying container type spelled out too
    priority_queue<int, vector<int>, greater<int>> pq;
    pq.push(3); pq.push(1); pq.push(2);
    while (!pq.empty()) { cout << pq.top() << " "; pq.pop(); } cout << "\n"; // => 1 2 3

    // custom lambda comparator with priority_queue needs decltype + ctor argument
    // (lambdas have no default constructor pre-C++20 stateless-lambda default ctor support
    //  in a portable way, so pass the instance explicitly)
    auto cmp = [](int a, int b){ return a > b; }; // min-heap ordering
    priority_queue<int, vector<int>, decltype(cmp)> pq2(cmp);
    pq2.push(5); pq2.push(1); pq2.push(3);
    while (!pq2.empty()) { cout << pq2.top() << " "; pq2.pop(); } cout << "\n"; // => 1 3 5
}
```
> **Gotcha:** Mixing these up is a constant source of compile errors — passing a comparator as a function *argument* to `set`'s constructor works (`set<int> s(v.begin(), v.end(), greater<int>())` is NOT how you'd do it — you'd need the TYPE to already declare `greater<int>` as its third template parameter), while trying to pass a comparator as a *template parameter* to `sort` doesn't compile (`sort` has no template comparator slot — it's purely a runtime function argument).

### Transparent comparators `less<>` **[C++14]**

```cpp
#include <bits/stdc++.h>
using namespace std;
int main(){
    // less<> (no template type arg) is "transparent" -- enables heterogeneous lookup,
    // e.g. searching a set<string> with a string_view/const char* without constructing a temporary string
    set<string, less<>> s{"apple","banana","cherry"};
    auto it = s.find("banana"); // works fine even without transparent cmp
    cout << (it != s.end()) << "\n"; // => 1

    // The real benefit: with less<>, s.find("banana") using a string_view/const char*
    // argument avoids an implicit temporary std::string allocation for the lookup key.
    string_view key = "cherry";
    auto it2 = s.find(key); // no temporary std::string constructed thanks to transparent less<>
    cout << (it2 != s.end()) << "\n"; // => 1
}
```
`less<>`/`greater<>` (empty angle brackets) are **transparent** comparators available since C++14 — they enable `.find()`/`.lower_bound()`/etc. to accept any comparable type without forcing a conversion to the container's exact key type first, saving allocations in hot lookup loops.

### Multi-key comparison with `tie` (comparator context)

```cpp
#include <bits/stdc++.h>
using namespace std;
struct Task { int priority; int deadline; string name; };
int main(){
    vector<Task> tasks{{2,10,"A"},{1,5,"B"},{2,3,"C"},{1,5,"D"}};
    // sort by priority asc, then deadline asc, then name asc
    sort(tasks.begin(), tasks.end(), [](const Task&a, const Task&b){
        return tie(a.priority, a.deadline, a.name) < tie(b.priority, b.deadline, b.name);
    });
    for (auto& t : tasks) cout << t.name << " ";
    cout << "\n"; // => B D C A
}
```
`tie` builds a temporary `tuple<const T&...>` whose `operator<` compares lexicographically field-by-field — the standard idiom for multi-key sort/comparator logic without hand-writing cascaded if/else chains (and it automatically satisfies strict weak ordering as long as each individual field's `<` does).

---

# PART V — BIT MANIPULATION

## 22. Bitwise Operators and Fundamentals

### 22.1 The operators

| Op | Name | Arity | Example | Notes |
|---|---|---|---|---|
| `&` | AND | binary | `a & b` | 1 iff both bits 1 |
| `\|` | OR | binary | `a \| b` | 1 iff either bit 1 |
| `^` | XOR | binary | `a ^ b` | 1 iff bits differ |
| `~` | NOT (complement) | unary | `~a` | flips every bit |
| `<<` | left shift | binary | `a << k` | multiply by 2^k (if no overflow) |
| `>>` | right shift | binary | `a >> k` | divide by 2^k, floor toward -inf for signed (arithmetic shift) |

Truth table for the three binary bitwise ops (per bit pair):

| a | b | a&b | a\|b | a^b |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 |
| 0 | 1 | 0 | 1 | 1 |
| 1 | 0 | 0 | 1 | 1 |
| 1 | 1 | 1 | 1 | 0 |

Worked example, `a = 0b1100` (12), `b = 0b1010` (10):

```
  1100   a
& 1010   b
------
  1000   a & b = 8

  1100
| 1010
------
  1110   a | b = 14

  1100
^ 1010
------
  0110   a ^ b = 6
```

```cpp
#include <iostream>
int main() {
    int a = 0b1100, b = 0b1010;
    std::cout << (a & b) << " " << (a | b) << " " << (a ^ b) << " " << (~a) << "\n";
}
// => 8 14 6 -13
```

`~a` on a 12 (`...00001100`) flips every one of the (typically 32) bits, giving `...11110011`, which as a two's-complement `int` reads as `-13`. This is the general identity `~x == -x - 1` (see below) — `~` is almost never what a beginner expects on a signed type, because it touches **all** bits, not just the "meaningful" low ones.

### 22.2 Precedence trap — the #1 bitwise bug

`&`, `|`, `^` bind **looser** than relational/equality operators (`==`, `!=`, `<`, `>`, `<=`, `>=`). This is a historical wart in C's grammar carried into C++. It means an expression like:

```cpp
if (x & 1 == 0)   // BUG: parses as x & (1 == 0)
```

is parsed as `x & (1 == 0)`, i.e. `x & 0`, which is always `0` (falsy) — **not** "is x even". The compiler (GCC/Clang) will actually warn about this:

```
warning: & has lower precedence than ==; == will be evaluated first [-Wparentheses]
```

```cpp
#include <iostream>
int main() {
    int x = 4; // even
    std::cout << (x & 1 == 0) << "\n";   // WRONG: parses as x & (1==0) == x & 0
    std::cout << ((x & 1) == 0) << "\n"; // RIGHT: (x & 1) computed first
}
// => 0
// 1
```

> **Gotcha:** Full precedence order (loose → tight) relevant here: `==`/`!=`  >  `&`  >  `^`  >  `|`  >  `&&`  >  `||`. **Always** parenthesize bitwise ops when mixing with comparisons: `if ((x & mask) == 0)`, `while ((x & 1) != 0)`, `if ((a ^ b) < c)`. This is the single most common silent bug in competitive-programming bit code — it compiles, runs, and gives wrong answers, often with no warning if `-Wparentheses` isn't on.

### 22.3 Two's complement

C++ (since C++20 mandated; universally true in practice before that too) represents signed integers in two's complement. To negate `x`: flip all bits and add 1.

Why `-x == ~x + 1`: bitwise complement satisfies `~x == -x - 1` (flipping every bit of `x` is the same as computing `-1 - x`, since `x + ~x` is all 1-bits, i.e. `-1`). Add 1 to both sides: `~x + 1 == -x`.

```cpp
#include <bitset>
#include <cstdint>
#include <iostream>
int main() {
    int8_t x = 5;
    int8_t neg = -x;
    std::cout << std::bitset<8>((uint8_t)x)   << "\n"; // 00000101
    std::cout << std::bitset<8>((uint8_t)~x)  << "\n"; // ~x = -x-1 = -6
    std::cout << std::bitset<8>((uint8_t)neg) << "\n"; // -x == ~x+1
}
// => 00000101
// 11111010
// 11111011
```

`11111010` (bit pattern of `~5`) is `-6`; adding 1 gives `11111011` = `-5`, matching `-x`.

Negative numbers in bits: `-1` is **all one-bits** at any width (this is why `~0 == -1`):

```cpp
#include <bitset>
#include <iostream>
int main() {
    int x = -1;
    std::cout << std::bitset<32>((unsigned)x) << "\n";
}
// => 11111111111111111111111111111111
```

More generally, a negative number's bit pattern is "large positive unsigned value" — this is exactly what lets `-1` double as an all-ones mask, and why casting a negative `int` to `unsigned` is a common (and legal, well-defined) trick to get its raw bits.

### 22.4 Shift traps

**`1 << 31` on a 32-bit signed `int` is undefined behavior** — shifting a `1` into the sign bit of a *signed* type is UB in C++ (the C++20 wording tightened left-shift to be well-defined as long as the result fits in the unsigned analog and reinterprets back, but shifting into/past the sign bit of a signed operand where the mathematical result doesn't fit remains UB pre-C++20 and is a portability foot-gun regardless — assume it is undefined and never rely on it). Fix by shifting an **unsigned** or **wide enough** literal:

```cpp
#include <iostream>
int main() {
    // int bad = 1 << 31;      // UB — DO NOT DO THIS
    unsigned int a = 1u << 31;  // fine: unsigned, no sign bit semantics
    long long   b = 1LL << 31;  // fine: fits comfortably in 64 bits
    std::cout << a << " " << b << "\n";
}
// => 2147483648 2147483648
```

**Shifting by ≥ the operand's bit-width is UB**, e.g. `int x = 1; x << 32;` on a 32-bit `int` — the shift amount must be in `[0, bit-width)`. This bites when a loop variable `i` runs up to (and including) `sizeof(T)*8` or when `n` in `1 << n` is attacker/input-controlled and unchecked.

**Right-shift on negative numbers is an *arithmetic* shift** on every mainstream compiler (sign-extends, filling with the sign bit) — this was only *implementation-defined* before C++20 and became guaranteed arithmetic-shift behavior in C++20. In practice (GCC/Clang/MSVC, always) you can rely on it, but know the term:

```cpp
#include <bitset>
#include <iostream>
int main() {
    int x = -8;
    std::cout << (x >> 1) << "\n"; // arithmetic shift: sign-extends -> -4
    std::cout << std::bitset<32>((unsigned)(x >> 1)) << "\n";
}
// => -4
// 11111111111111111111111111111100
```

> **Gotcha:** if you want a *logical* (zero-filling) right shift regardless of sign, cast to `unsigned` first: `(unsigned)x >> k`.

### 22.5 Signed vs. unsigned for bit work

- Prefer **unsigned** types (`unsigned`, `unsigned long long`) for masks, popcount inputs, and anything that intentionally uses the sign bit as data — shifting/complementing signed values invites UB as above.
- But watch comparisons: mixing signed and unsigned in `<`/`>` silently converts the signed value to unsigned (a negative number becomes huge), a classic loop-bound bug (`for (int i = n - 1; i >= 0; i--)` where `n` is `size_t`-derived and `n - 1` underflows if `n == 0`).
- For DP masks / subset enumeration, `int` is fine up to `n <= 30`; beyond that use `long long`/`unsigned long long`.

### 22.6 Type sizes (typical LP64 / competitive-judge Linux, 64-bit)

| Type | Size | Bits | Range highlight |
|---|---|---|---|
| `bool` | 1 | 8 (1 used) | 0/1 |
| `char` | 1 | 8 | -128..127 or 0..255 |
| `int` | 4 | 32 | ±2.1×10^9 |
| `unsigned int` | 4 | 32 | 0..4.3×10^9 |
| `long` | 8 (Linux/Mac) / 4 (Win) | 64/32 | platform-dependent — don't rely on it |
| `long long` | 8 | 64 | ±9.2×10^18 |
| `unsigned long long` | 8 | 64 | 0..1.8×10^19 |

### 22.7 `1LL << k` for k ≥ 31 — the most common bitmask bug

The bug: writing `1 << k` where `k` can reach 31+ (e.g. building a mask for up to 60 items, or computing `2^k` for large `k`). `1` is `int` (32-bit); the shift either overflows a signed type (UB) or silently truncates once `k >= 32` conceptually. Fix: force the literal to be 64-bit **before** shifting by using the `LL` suffix.

```cpp
#include <iostream>
int main() {
    int n = 40;
    // long long buggy = 1 << n;  // UB: shift count >= 32 on a 32-bit int
    long long ok = 1LL << n;       // correct: 1 is now a 64-bit literal
    std::cout << ok << "\n";
}
// => 1099511627776
```

> **Gotcha:** `1 << n` is a 32-bit computation *regardless of what you assign it to* — assigning the (already broken) result to a `long long` does not retroactively fix it. The `LL` must be on the literal being shifted, not just on the destination variable.

---

## 23. The Complete Bit-Trick Catalog

### 23.1 Test / set / clear / toggle bit `i`

```cpp
#include <iostream>
int main() {
    int x = 0b1010; // 10
    int i = 1;
    bool test  = x & (1 << i);   // test bit i
    int  set   = x | (1 << i);   // set bit i
    int  clear = x & ~(1 << i);  // clear bit i
    int  toggle= x ^ (1 << i);   // toggle bit i
    std::cout << test << " " << set << " " << clear << " " << toggle << "\n";
}
// => 1 10 8 8
```
```
x      = 1010
1<<1   = 0010
test:  1010 & 0010 = 0010 (nonzero -> true)
set:   1010 | 0010 = 1010  (already set, unchanged = 10)
clear: 1010 & 1101 = 1000  (=8)
toggle:1010 ^ 0010 = 1000  (=8, was set so it clears)
```

Extract bit `i` as 0/1 (not just truthy): `int bit = (x >> i) & 1;`

### 23.2 `x & 1` odd/even, `x >> 1` halve, shifts as ×/÷ powers of 2

```cpp
#include <iostream>
int main() {
    int x = 7;
    std::cout << (x & 1) << "\n";  // 1 => odd
    std::cout << (x >> 1) << "\n"; // 3 => x/2 (floor)
    std::cout << (x << 2) << "\n"; // 28 => x*4
}
// => 1
// 3
// 28
```

`x << k` == `x * 2^k`; `x >> k` == `floor(x / 2^k)` for non-negative `x` (arithmetic shift for negatives, still floors toward -infinity).

### 23.3 `x & (x-1)` clears the lowest set bit — Brian Kernighan popcount

```cpp
#include <iostream>
int popcount(unsigned x) {
    int c = 0;
    while (x) { x &= (x - 1); c++; } // each iteration removes exactly one 1-bit
    return c;
}
int main() {
    unsigned x = 0b10110100; // 180
    std::cout << (x & (x - 1)) << "\n"; // 176 = lowest set bit (bit2, value 4) cleared
    std::cout << popcount(x) << "\n";   // 4
}
// => 176
// 4
```
```
x   = 10110100
x-1 = 10110011
x&(x-1) = 10110000 (176)  <- lowest 1-bit at position 2 turned off
```

Runs in `O(popcount(x))` — faster than the naive `O(bit-width)` loop-over-i when few bits are set. This is the classic building block for "process only the set bits" loops.

### 23.4 `x & (-x)` isolates the lowest set bit; `x | (x+1)` sets the lowest clear bit

```cpp
#include <iostream>
int main() {
    int x = 0b10110100; // 180
    std::cout << (x & (-x)) << "\n"; // 4 => isolates lowest set bit (bit 2)

    int y = 0b10110011; // 179
    std::cout << (y | (y + 1)) << "\n"; // 183 => sets the lowest 0-bit
}
// => 4
// 183
```
```
y    = 10110011  (lowest clear bit is bit 2)
y+1  = 10110100
y|(y+1) = 10110111 (183)
```

`x & (-x)` is the "isolate lowest set bit" trick used everywhere (Fenwick/BIT trees, subset DP transitions, `__builtin_ctz` alternative).

### 23.5 Power-of-two check

```cpp
#include <iostream>
bool isPowerOfTwo(unsigned x) {
    return x && !(x & (x - 1)); // exactly one bit set, and x != 0
}
int main() {
    std::cout << isPowerOfTwo(16) << " " << isPowerOfTwo(18) << " " << isPowerOfTwo(0) << "\n";
}
// => 1 0 0
```

The `x &&` guard is essential: `0 & (0-1)` is `0 & 0xFFFFFFFF == 0`, which would wrongly say "0 is a power of two" without it.

### 23.6 XOR swap (and why not to use it)

```cpp
#include <iostream>
int main() {
    int a = 5, b = 9;
    a ^= b; b ^= a; a ^= b;
    std::cout << a << " " << b << "\n";
}
// => 9 5
```

> **Gotcha:** breaks silently if `a` and `b` alias the same memory location (e.g. `swap_xor(arr[i], arr[i])` zeroes the element: `a^=a` gives 0 at step 1, and everything cascades to 0). It's also not faster than `std::swap` on modern compilers/pipelines (breaks instruction-level parallelism via a 3-step dependency chain, and prevents vectorization), and it's a readability tax. **Use `std::swap` (or structured bindings / `std::exchange`) in real code — this is folklore, not a recommended practice.**

### 23.7 XOR properties → the entire "find the odd one out" family

Core identities: `x ^ x == 0`, `x ^ 0 == x`, XOR is commutative and associative (order doesn't matter, can freely regroup).

**Single non-repeating element** (everything else appears twice) — XOR of all elements cancels pairs, leaving the singleton:

```cpp
#include <vector>
#include <iostream>
int singleNumber(std::vector<int>& nums) {
    int r = 0;
    for (int x : nums) r ^= x;
    return r;
}
int main() {
    std::vector<int> v = {4, 1, 2, 1, 2};
    std::cout << singleNumber(v) << "\n";
}
// => 4
```

**Two non-repeating elements** (everything else appears twice, exactly two singletons `a`,`b`) — XOR of all gives `a^b`; pick any bit where `a` and `b` differ (the lowest set bit of `a^b`) to split the array into two groups, one containing `a`, the other `b`:

```cpp
#include <vector>
#include <iostream>
std::pair<int,int> singleNumberTwo(std::vector<int>& nums) {
    int xorAll = 0;
    for (int x : nums) xorAll ^= x;
    int diff = xorAll & (-xorAll);      // a bit where a and b differ
    int a = 0;
    for (int x : nums) if (x & diff) a ^= x;
    int b = xorAll ^ a;
    return {a, b};
}
int main() {
    std::vector<int> v = {1, 2, 1, 3, 2, 5};
    auto [a, b] = singleNumberTwo(v);
    std::cout << a << " " << b << "\n";
}
// => 3 5
```

**Missing number** in `0..n` (array of size `n` holds `n` distinct values from `0..n` with exactly one missing) — XOR indices `0..n-1` and values together with `n`; every present index/value pair cancels, leaving the missing value:

```cpp
#include <vector>
#include <iostream>
int missingNumber(std::vector<int>& nums) {
    int n = nums.size();
    int r = n;
    for (int i = 0; i < n; i++) r ^= i ^ nums[i];
    return r;
}
int main() {
    std::vector<int> v = {3, 0, 1}; // missing 2, from range 0..3
    std::cout << missingNumber(v) << "\n";
}
// => 2
```

**XOR of range `[0..n]`, closed form** — no need to loop; depends only on `n % 4`:

```cpp
#include <iostream>
int xorUpTo(int n) { // 0 ^ 1 ^ 2 ^ ... ^ n
    switch (n % 4) {
        case 0: return n;
        case 1: return 1;
        case 2: return n + 1;
        default: return 0; // n % 4 == 3
    }
}
int main() {
    std::cout << xorUpTo(5) << "\n"; // 0^1^2^3^4^5
}
// => 1
```

Use `xorUpTo(r) ^ xorUpTo(l-1)` to get XOR of an arbitrary range `[l, r]` in O(1).

### 23.8 Masks: extracting and clearing bit ranges

| Recipe | Expression |
|---|---|
| Low `k` bits set (mask of width `k`) | `(1 << k) - 1` |
| Clear bits below `i` (keep bit `i` and above) | `x & ~((1 << i) - 1)` |
| Set bits below `i` | `x \| ((1 << i) - 1)` |
| Keep only bits below `i` (clear `i` and above) | `x & ((1 << i) - 1)` |
| Extract field `[lo, hi]` inclusive | `(x >> lo) & ((1 << (hi - lo + 1)) - 1)` |

```cpp
#include <bitset>
#include <iostream>
int main() {
    int k = 5;
    int mask = (1 << k) - 1;
    std::cout << std::bitset<8>(mask) << "\n"; // 00011111
}
// => 00011111
```

```cpp
#include <bitset>
#include <iostream>
int main() {
    int x = 0b11011010; // 218
    int i = 3;
    std::cout << std::bitset<8>(x & ~((1 << i) - 1)) << "\n"; // clear low 3 bits
    std::cout << std::bitset<8>(x | ((1 << i) - 1)) << "\n";  // set low 3 bits
    std::cout << std::bitset<8>(x & ((1 << i) - 1)) << "\n";  // keep only low 3 bits

    int lo = 2, hi = 5;
    int field = (x >> lo) & ((1 << (hi - lo + 1)) - 1); // bits [2..5]
    std::cout << std::bitset<4>(field) << "\n";
}
// => 11011000
// 11011111
// 00000010
// 0110
```

### 23.9 Branchless abs / sign / min / max (curiosities — clarity usually wins)

```cpp
#include <iostream>
int main() {
    int x = -37;
    int mask = x >> 31;              // all 1s if negative, all 0s if non-negative
    int absX = (x + mask) ^ mask;    // branchless abs
    std::cout << absX << "\n";

    int a = 12, b = 7;
    int signBit = ((a - b) >> 31) & 1;                       // 0 if a>=b, 1 if a<b
    int minAB = b + ((a - b) & ((a - b) >> 31));             // branchless min(a,b)
    int maxAB = a - ((a - b) & ((a - b) >> 31));             // branchless max(a,b)
    std::cout << signBit << " " << minAB << " " << maxAB << "\n";
}
// => 37
// 0 7 12
```

> **Gotcha:** these rely on arithmetic right-shift of negatives (guaranteed since C++20, universal in practice before) and are only worthwhile if profiling shows the branch is actually a bottleneck — `std::abs`/`std::min`/`std::max` are just as fast on modern compilers (which emit `cmov`) and vastly more readable. Know them for reading old code / interview trivia, don't reach for them by default.

### 23.10 Highest set bit / floor(log2) / next & previous power of two

```cpp
#include <iostream>
int main() {
    unsigned x = 37; // 100101
    int floorLog2 = 31 - __builtin_clz(x); // position of highest set bit
    std::cout << floorLog2 << "\n";
}
// => 5
```

```cpp
#include <iostream>
unsigned nextPowerOfTwo(unsigned x) {
    if (x <= 1) return 1;
    return 1u << (32 - __builtin_clz(x - 1));
}
unsigned prevPowerOfTwo(unsigned x) {
    return 1u << (31 - __builtin_clz(x));
}
int main() {
    std::cout << nextPowerOfTwo(37) << " " << prevPowerOfTwo(37) << "\n";
}
// => 64 32
```

> **Gotcha:** `__builtin_clz(0)` is undefined — both helpers above implicitly assume `x != 0` for `prevPowerOfTwo`; guard the caller if `x` can be 0. See §24 for the safe C++20 `<bit>` replacements (`bit_width`, `bit_ceil`, `bit_floor`) that handle edges cleanly.

### 23.11 Reverse bits

```cpp
#include <cstdint>
#include <iostream>
uint32_t reverseBits(uint32_t x) {
    uint32_t r = 0;
    for (int i = 0; i < 32; i++) {
        r = (r << 1) | (x & 1);
        x >>= 1;
    }
    return r;
}
int main() {
    std::cout << reverseBits(1u) << "\n"; // bit 0 -> becomes bit 31
}
// => 2147483648
```

### 23.12 Detect alternating bits

Bits alternate (`...0101...` or `...1010...`) iff `n ^ (n >> 1)` is all 1s, i.e. `x & (x+1) == 0`:

```cpp
#include <iostream>
bool hasAlternatingBits(unsigned n) {
    unsigned x = n ^ (n >> 1);
    return !(x & (x + 1)); // x must be of the form 0..0111..1
}
int main() {
    std::cout << hasAlternatingBits(5) << " " << hasAlternatingBits(7) << "\n";
}
// => 1 0
```
```
n=5=101, n>>1=010, xor=111 (all ones up to width) -> alternating: true
n=7=111, n>>1=011, xor=100 -> not all ones -> false
```

### 23.13 Subset (submask) enumeration — the O(3^n) trick

Enumerate every submask `s` of a mask `m` (including `m` itself and the empty set `0`) in decreasing order:

```cpp
#include <iostream>
int main() {
    int m = 0b1010; // enumerate every submask of {bit1, bit3}
    int s = m;
    while (true) {
        std::cout << s << " ";
        if (s == 0) break;
        s = (s - 1) & m; // next-lower submask of m
    }
    std::cout << "\n";
}
// => 10 8 2 0
```

> **Gotcha:** the idiomatic `for (int s = m; s; s = (s - 1) & m)` **stops as soon as `s` becomes 0 and never runs the body with `s == 0`**, silently skipping the empty subset. If the empty submask matters to your problem (it usually does for "sum over all subsets" style DP), handle it explicitly — either restructure as a `do { ... } while (s != m ...)`-style loop, or (simplest) special-case `s == 0` outside the loop, as done above with the `while(true)`/`break` form.

**Why the total work over all masks is O(3^n):** enumerating `(mask, submask)` pairs for every `mask` from `0` to `2^n - 1`, each of the `n` bits of `n`-bit universe independently falls into exactly one of 3 states across the pair: (a) not in `mask` at all (so also not in `submask`), (b) in `mask` but not in `submask`, (c) in both `mask` and `submask`. That's `3^n` total `(mask, submask)` combinations summed over everything — the classic "sum of subsets of subsets" bound, used to justify SOS-DP / subset-convolution style algorithms as feasible up to roughly `n ~= 20`.

**Iterate all masks** (not just submasks of one fixed `m`) — every subset of an `n`-element universe:

```cpp
int n = 5;
for (int mask = 0; mask < (1 << n); mask++) {
    // process mask
}
```

**Iterate set bits of a mask** — two standard idioms:

```cpp
#include <iostream>
int main() {
    int m = 0b101101;
    // idiom 1: peel off the lowest set bit each time (O(popcount) iterations)
    int t = m;
    while (t) {
        int b = t & (-t);          // isolate lowest set bit
        int idx = __builtin_ctz(b); // its index
        std::cout << idx << " ";
        t -= b;                     // equivalent to t &= (t - 1)
    }
    std::cout << "\n";

    // idiom 2: loop over every bit position, test membership (O(width) iterations)
    for (int i = 0; i < 6; i++) {
        if (m & (1 << i)) std::cout << i << " ";
    }
    std::cout << "\n";
}
// => 0 2 3 5
// 0 2 3 5
```

Idiom 1 is better when the mask is sparse (few bits set relative to width); idiom 2 is simpler and fine when width is small (`n <= ~20-30`, typical bitmask-DP universes).

### 23.14 Gray code

Binary-reflected Gray code: successive values differ by exactly one bit. Formula: `gray(i) = i ^ (i >> 1)`.

```cpp
#include <iostream>
int main() {
    for (int i = 0; i < 8; i++) {
        int gray = i ^ (i >> 1);
        std::cout << gray << " ";
    }
    std::cout << "\n";
}
// => 0 1 3 2 6 7 5 4
```

### 23.15 Counting bits for `0..n` via DP

`dp[i] = dp[i >> 1] + (i & 1)` — the popcount of `i` equals the popcount of `i` with its lowest bit stripped, plus that lowest bit itself. O(n) total instead of O(n log n) from calling popcount independently per value.

```cpp
#include <vector>
#include <iostream>
int main() {
    int n = 5;
    std::vector<int> dp(n + 1);
    for (int i = 1; i <= n; i++) dp[i] = dp[i >> 1] + (i & 1);
    for (int v : dp) std::cout << v << " ";
    std::cout << "\n";
}
// => 0 1 1 2 1 2
```

---

## 24. Compiler Intrinsics, `<bit>`, and Bit Containers

### 24.1 GCC/Clang `__builtin_*` family

All operate on the **unsigned** (or unsigned-`long long`) argument's bit pattern. Every one has an `ll`-suffixed overload for `unsigned long long`.

| Intrinsic | Meaning | `ll` variant | Complexity | UB / notes |
|---|---|---|---|---|
| `__builtin_popcount(x)` | number of set bits | `__builtin_popcountll` | O(1) (hardware `POPCNT` when available, else O(log w)) | none for `x==0` (returns 0) |
| `__builtin_clz(x)` | count leading zeros (from MSB) | `__builtin_clzll` | O(1) | **UB if `x == 0`** |
| `__builtin_ctz(x)` | count trailing zeros (from LSB) | `__builtin_ctzll` | O(1) | **UB if `x == 0`** |
| `__builtin_ffs(x)` | 1-based index of lowest set bit, or `0` if `x==0` | `__builtin_ffsll` | O(1) | safe for `x==0` (only one in this table that is) |
| `__builtin_parity(x)` | parity of popcount: `1` if odd number of set bits | `__builtin_parityll` | O(1) | none for `x==0` (returns 0) |

```cpp
#include <iostream>
int main() {
    unsigned x = 0b10110100; // 180
    std::cout << __builtin_popcount(x) << "\n"; // 4
    std::cout << __builtin_clz(x)      << "\n"; // leading zeros in 32-bit width
    std::cout << __builtin_ctz(x)      << "\n"; // trailing zeros
    std::cout << __builtin_ffs(x)      << "\n"; // 1-based index of lowest set bit
    std::cout << __builtin_parity(x)   << "\n"; // popcount % 2
}
// => 4
// 24
// 2
// 3
// 0
```

`ll` variants for values beyond 32 bits:

```cpp
#include <iostream>
int main() {
    long long big = 1LL << 40;
    std::cout << __builtin_popcountll(big) << "\n"; // 1
    std::cout << __builtin_ctzll(big)      << "\n"; // 40
}
// => 1
// 40
```

> **Gotcha 1 (UB on zero):** `__builtin_clz(0)` and `__builtin_ctz(0)` are **undefined behavior** — on x86 they typically compile to `bsr`/`bsf`/`lzcnt`/`tzcnt`, some of which are themselves undefined/implementation-defined at `0` at the hardware level. **Always guard**: `if (x != 0) { ... __builtin_ctz(x) ... }`.
>
> **Gotcha 2 (silent truncation):** passing a `long long` to the **non**-`ll` overload implicitly converts it to `unsigned` (32-bit), silently discarding the high 32 bits — no warning, just a wrong answer:
> ```cpp
> #include <iostream>
> int main() {
>     long long big = 1LL << 40; // bit 40 — outside the low 32 bits entirely
>     std::cout << __builtin_popcount((unsigned)big) << "\n"; // truncated -> 0 bits visible
>     std::cout << __builtin_popcountll(big)          << "\n"; // correct
> }
> // => 0
> // 1
> ```
> Always match the intrinsic's width to the operand's actual type: `long long`/`unsigned long long` → the `ll` suffix, always.

**MSVC portability note:** `__builtin_*` are GCC/Clang extensions — MSVC does not provide them. MSVC's equivalents are `__popcnt`/`__popcnt64`, `_BitScanForward`/`_BitScanReverse` (return a bool + write the index via out-parameter, different calling convention entirely). For code that must build under MSVC too, either `#ifdef _MSC_VER` and branch to the MSVC intrinsics, or — far simpler — target **C++20 `<bit>`** below, which is portable across all three major compilers.

### 24.2 **[C++20]** `<bit>` — the portable replacement

Available with `-std=c++20` (GCC 10+/Clang 10+/MSVC 19.26+, roughly). Works directly on unsigned integer types, no leading-underscore extension magic, and most of it is `constexpr`.

| `__builtin_*` (GCC/Clang) | `std::` (`<bit>`, C++20) | Notes |
|---|---|---|
| `__builtin_popcount(x)` | `std::popcount(x)` | unsigned types only |
| `__builtin_clz(x)` | `std::countl_zero(x)` | |
| `__builtin_ctz(x)` | `std::countr_zero(x)` | |
| `x && !(x & (x-1))` | `std::has_single_bit(x)` | power-of-two check, no manual `&&` guard needed |
| `31 - __builtin_clz(x) + 1` (roughly) | `std::bit_width(x)` | minimal bits needed to represent `x` (0 for `x==0`) |
| hand-rolled `nextPowerOfTwo(x)` | `std::bit_ceil(x)` | smallest power of two `>= x` |
| hand-rolled `prevPowerOfTwo(x)` | `std::bit_floor(x)` | largest power of two `<= x` (0 if `x==0`) |
| manual rotate via shifts+OR | `std::rotl(x, s)` / `std::rotr(x, s)` | rotate left/right by `s` bits |
| `reinterpret_cast` punning (UB!) | `std::bit_cast<T>(x)` | safe, `constexpr`-friendly type punning |

```cpp
// Compile with -std=c++20
#include <bit>
#include <iostream>
int main() {
    unsigned x = 180;
    std::cout << std::popcount(x)        << "\n"; // 4
    std::cout << std::countl_zero(x)     << "\n"; // 24
    std::cout << std::countr_zero(x)     << "\n"; // 2
    std::cout << std::has_single_bit(64u)<< "\n"; // 1 (power of two)
    std::cout << std::bit_width(x)       << "\n"; // 8 (bits needed to represent 180)
    std::cout << std::bit_ceil(37u)      << "\n"; // 64 (next pow2 >= 37)
    std::cout << std::bit_floor(37u)     << "\n"; // 32 (prev pow2 <= 37)
    std::cout << std::rotl(x, 2)         << "\n"; // rotate left by 2
}
// => 4
// 24
// 2
// 1
// 8
// 64
// 32
// 720
```

`std::popcount`/`countl_zero`/`countr_zero` are **well-defined for `x == 0`** (unlike their `__builtin_` counterparts) — `countl_zero(0)` and `countr_zero(0)` both return the type's full bit width, no guard needed. This is a meaningful safety upgrade, not just a syntax rename.

`std::bit_cast` replaces the classic (and UB) `*reinterpret_cast<uint32_t*>(&f)` float-bit-pattern trick:

```cpp
// Compile with -std=c++20
#include <bit>
#include <cstdint>
#include <iostream>
int main() {
    float f = 1.0f;
    uint32_t bits = std::bit_cast<uint32_t>(f);
    std::cout << std::hex << bits << std::dec << "\n"; // IEEE-754 bit pattern of 1.0f
}
// => 3f800000
```

Both types in `bit_cast<To>(from)` must be the same size and trivially copyable — the compiler enforces this at compile time, no runtime aliasing violation possible (unlike the old `reinterpret_cast` pun, which is UB under strict aliasing).

### 24.3 `int`/`long long` as a subset bitmask — the DSA workhorse

For `n <= ~20-30`, representing a subset of `{0, 1, ..., n-1}` as the bits of an `int`/`unsigned`/`long long` is the default choice for bitmask DP (TSP, assignment problem, "count ways to partition" problems) because it's O(1) to copy, compare, and hash as a DP array index or `map`/`unordered_map` key.

```cpp
#include <iostream>
int main() {
    int n = 5;
    int mask = 0;
    mask |= (1 << 2);                 // add element 2 to the set
    mask |= (1 << 4);                 // add element 4
    std::cout << ((mask >> 2) & 1) << "\n"; // test membership of 2 -> 1
    mask &= ~(1 << 2);                // remove element 2
    std::cout << ((mask >> 2) & 1) << "\n"; // -> 0
    for (int i = 0; i < n; i++)
        if (mask & (1 << i)) std::cout << i << " ";
    std::cout << "\n";
}
// => 1
// 0
// 4
```

### 24.4 bitset vs. int mask vs. `vector<bool>` — one-line rule

- **`int`/`long long` mask**: `n` fixed and small (≲ 62), needs to be copied/compared/hashed cheaply (e.g. as a DP state or map key) — the default for bitmask DP.
- **`std::bitset<N>`**: `N` fixed at compile time and can be large (thousands+), you need fast bulk bitwise ops (`&`, `|`, `count()`) over a big fixed-size set — full details (construction, `to_ulong`, `to_string`, hashing caveats) live in the containers part, not repeated here.
- **`std::vector<bool>`**: size only known at runtime, memory-packed storage matters more than per-access speed — but avoid it in hot inner loops (proxy-reference overhead); details in the containers part.

### 24.5 Worked DSA example: counting-bits DP + bitmask-DP transition

Counting-bits DP (`dp[i] = dp[i>>1] + (i&1)`, LeetCode 338) was already shown in §23.15 — it's the standard warm-up for "think in bits, not loops."

A fuller bitmask-DP state transition (the pattern referenced as bitmask DP in **Pattern 24** of the DP-patterns notes) — Traveling Salesman via `dp[mask][u]` = minimum cost to have visited exactly the cities in `mask`, ending at city `u`:

```cpp
#include <vector>
#include <iostream>
#include <climits>
#include <algorithm>
int main() {
    int n = 4;
    std::vector<std::vector<int>> dist = {
        {0, 10, 15, 20},
        {10, 0, 35, 25},
        {15, 35, 0, 30},
        {20, 25, 30, 0}
    };
    int FULL = (1 << n) - 1;
    std::vector<std::vector<int>> dp(1 << n, std::vector<int>(n, INT_MAX / 2));
    dp[1][0] = 0; // mask = {city 0}, start and end at city 0

    for (int mask = 1; mask <= FULL; mask++) {
        for (int u = 0; u < n; u++) {
            if (!(mask & (1 << u)) || dp[mask][u] == INT_MAX / 2) continue;
            for (int v = 0; v < n; v++) {
                if (mask & (1 << v)) continue;        // v already visited
                int nmask = mask | (1 << v);           // add v to the visited set
                dp[nmask][v] = std::min(dp[nmask][v], dp[mask][u] + dist[u][v]);
            }
        }
    }
    int best = INT_MAX;
    for (int u = 1; u < n; u++)
        best = std::min(best, dp[FULL][u] + dist[u][0]); // close the tour back to city 0
    std::cout << best << "\n";
}
// => 80
```

The bit operations here are exactly §23's catalog in action: `mask & (1<<u)` (membership test), `mask | (1<<v)` (add to subset), and `mask <= FULL` / `mask == (1<<n)-1` (iterate-all-masks, full-set sentinel) — this is the shape every bitmask-DP problem takes once the underlying subset representation clicks.

---

# PART VI — NUMERICS, I/O AND THE MATH TOOLKIT

## 25. Integer Arithmetic, Overflow and Safety

### 25.1 Type ranges — pick from constraints

| Type | Size (typical LP64) | Min | Max | Notes |
|---|---|---|---|---|
| `int` | 4 bytes | -2,147,483,648 | 2,147,483,647 (~2.1e9) | default integer type |
| `unsigned int` | 4 bytes | 0 | 4,294,967,295 (~4.3e9) | wraps, doesn't error |
| `long` | 8 bytes (Linux/macOS), 4 bytes (Windows/MSVC) | — | — | **avoid, platform-dependent** |
| `long long` | 8 bytes | ~-9.22e18 | ~9.22e18 | use this for "big" |
| `unsigned long long` | 8 bytes | 0 | ~1.84e19 | |
| `__int128` (GCC/Clang) | 16 bytes | ~-1.7e38 | ~1.7e38 | no `cin`/`cout` support |

Rule of thumb for competitive programming: if any intermediate value or the answer can exceed ~2×10^9, use `long long` for that variable — including loop counters that get multiplied.

```cpp
#include <iostream>
#include <climits>
using namespace std;
int main() {
    cout << INT_MAX << " " << INT_MIN << "\n";
    cout << LLONG_MAX << " " << LLONG_MIN << "\n";
    cout << UINT_MAX << "\n";
    cout << ULLONG_MAX << "\n";
}
// => 2147483647 -2147483648
// => 9223372036854775807 -9223372036854775808
// => 4294967295
// => 18446744073709551615
```

### 25.2 Signed overflow is UNDEFINED BEHAVIOR

Signed integer overflow is **not** wraparound in the standard's eyes — it's UB. The compiler is legally allowed to assume it never happens, and optimizers exploit this (e.g. `if (a + 1 > a)` can be optimized to always `true` even though it's false when `a == INT_MAX`). Unsigned overflow, by contrast, is well-defined modular wraparound.

```cpp
#include <iostream>
using namespace std;
int main() {
    int a = 2147483647;      // INT_MAX
    int b = a + 1;            // UB in the standard; in practice often wraps to INT_MIN
    cout << b << "\n";        // => -2147483648 (typical, but NOT guaranteed by the standard)

    unsigned int u = 4294967295u; // UINT_MAX
    unsigned int v = u + 1;        // well-defined: wraps to 0
    cout << v << "\n";              // => 0
}
```

> **Gotcha:** Never rely on signed overflow "wrapping" for correctness — with `-O2`/`-O3` the compiler may eliminate your overflow check entirely because it assumes overflow can't happen. Use a wider type or `unsigned` (with explicit modular reasoning) instead.

### 25.3 The classic overflow bugs

**1. Multiplying two `int`s before assigning to `long long`** — the multiplication happens in `int` first, THEN gets converted:

```cpp
#include <iostream>
using namespace std;
int main() {
    int a = 100000, b = 100000;
    long long bad = a * b;              // overflow happens in int BEFORE assignment
    long long good = (long long)a * b;  // cast ONE operand -> whole expr promoted to long long
    cout << bad << "\n";   // => garbage / UB (commonly 1410065408 due to 32-bit truncation)
    cout << good << "\n";  // => 10000000000
}
```

> **Gotcha:** `(long long)(a * b)` is WRONG — the cast applies after the overflow already happened. Cast an operand INSIDE the multiplication: `(long long)a * b`.

**2. Binary-search midpoint overflow** — `lo + hi` can exceed `int` range when both are large:

```cpp
int lo = 1'500'000'000, hi = 2'000'000'000;
int mid_bad  = (lo + hi) / 2;         // lo+hi overflows int (UB)
int mid_good = lo + (hi - lo) / 2;    // hi-lo always fits, safe
```

**3. `accumulate` with a wrong-type initial value**:

```cpp
#include <numeric>
#include <vector>
#include <iostream>
using namespace std;
int main() {
    vector<int> v(100000, 100000); // sum = 10,000,000,000 > INT_MAX
    long long bad  = accumulate(v.begin(), v.end(), 0);      // 0 is int -> accumulates as int, overflows
    long long good = accumulate(v.begin(), v.end(), 0LL);    // 0LL is long long -> accumulates as long long
    cout << bad << "\n";   // => garbage (wrapped 32-bit value)
    cout << good << "\n";  // => 10000000000
}
```

> **Gotcha:** The type of the *initial value* passed to `accumulate` determines the accumulator's type, not the container's element type. Always pass `0LL` (or `0.0` for floating sums) when a plain `0` could silently overflow.

**4. `1 << k` for large `k`**:

```cpp
int k = 40;
long long bad  = 1 << k;        // UB: 1 is int, shift amount >= 32 is UB for 32-bit int; also overflow
long long good = 1LL << k;      // 1LL is long long, safe up to k=62
// cout << bad;   // undefined — do not do this
cout << (1LL << 40) << "\n";    // => 1099511627776
```

### 25.4 Division and modulo semantics

Integer division truncates toward zero (C++11 onward standardized this). `%` (remainder) satisfies `(a/b)*b + a%b == a`, so its sign follows the **dividend**, not the divisor.

```cpp
#include <iostream>
using namespace std;
int main() {
    cout << 7 / 3 << " " << 7 % 3 << "\n";     // =>  2  1
    cout << -7 / 3 << " " << -7 % 3 << "\n";   // => -2 -1   (truncation toward zero)
    cout << 7 / -3 << " " << 7 % -3 << "\n";   // => -2  1
    cout << -7 / -3 << " " << -7 % -3 << "\n"; // =>  2 -1
}
```

**Safe positive modulo idiom** (needed constantly for hashing / cyclic indices with possibly-negative operands):

```cpp
long long safe_mod(long long a, long long m) {
    return ((a % m) + m) % m;
}
// safe_mod(-7, 3) => 2   (mathematically correct remainder, always in [0, m))
```

> **Gotcha:** `a % m` alone can be negative if `a` is negative. Array-index-by-modulo code (`arr[i % n]`) breaks silently if `i` can go negative — wrap with `safe_mod`.

### 25.5 `__int128` (GCC/Clang extension)

For intermediate products that can exceed even `long long` (e.g. multiplying two `long long`s, or modular multiplication with mod ~1e18), use `__int128`. It has no `iostream` support — write your own print helper.

```cpp
#include <iostream>
using namespace std;

void print_i128(__int128 x) {
    if (x < 0) { cout << '-'; x = -x; }
    if (x > 9) print_i128(x / 10);
    cout << (int)(x % 10);
}

int main() {
    __int128 a = 1e18, b = 1e18;
    __int128 prod = a * b;           // way beyond long long range (~1e36)
    print_i128(prod);
    cout << "\n";                    // => 1000000000000000000000000000000000000
    print_i128(-prod);
    cout << "\n";                    // => -1000000000000000000000000000000000000
}
```

> **Gotcha:** `__int128` is a **GCC/Clang extension**, not standard C++ — unavailable on MSVC. Fine for Codeforces/LeetCode (GCC-based judges), risky for "portable" code.

### 25.6 `abs`, `llabs`, `fabs` and the truncation trap

| Function | Header | Argument type | Return type |
|---|---|---|---|
| `abs(int)` | `<cstdlib>` | `int` | `int` |
| `labs(long)` | `<cstdlib>` | `long` | `long` |
| `llabs(long long)` | `<cstdlib>` | `long long` | `long long` |
| `abs(double)` / `fabs` | `<cmath>` | floating | floating |
| `std::abs` (overloaded) | `<cmath>` | any of the above | matches input |

```cpp
#include <cstdlib>
#include <cmath>
#include <iostream>
using namespace std;
int main() {
    long long big = -9223372036854775807LL - 1; // LLONG_MIN
    // int bad = abs((int)big);       // truncates to int FIRST -> wrong/UB
    long long good = llabs(big + 1);  // use the ll-specific overload
    cout << good << "\n";              // => 9223372036854775807

    cout << abs(-5) << "\n";      // => 5   (int overload)
    cout << fabs(-5.5) << "\n";   // => 5.5 (double)
}
```

> **Gotcha:** Calling `abs()` on a `long long` in some contexts (older C headers, or after `using namespace std;` collides with C's `int abs(int)` from `<cstdlib>`) silently truncates to `int` first. Prefer `llabs()` for `long long`, or `<cmath>`'s overloaded `std::abs`. Also: `abs(LLONG_MIN)` overflows — there's no positive representation of `-LLONG_MIN` in `long long`.

### 25.7 `<cmath>` essentials and integer-safety pitfalls

```cpp
#include <cmath>
#include <iostream>
using namespace std;
int main() {
    cout << sqrt(16.0) << "\n";   // => 4
    cout << pow(2.0, 10) << "\n"; // => 1024
    cout << floor(3.7) << "\n";   // => 3
    cout << ceil(3.2) << "\n";    // => 4
    cout << round(3.5) << "\n";   // => 4
    cout << log(exp(1.0)) << "\n"; // => 1
    cout << log2(1024.0) << "\n";  // => 10
}
```

**Why `sqrt`/`pow` are dangerous on integers:** both operate in floating point. FP imprecision means `(int)sqrt(x)` can be off by one for perfect squares or near-boundary values.

```cpp
#include <cmath>
#include <iostream>
using namespace std;
int main() {
    long long x = 1000000000000LL;      // 10^12, whose exact sqrt is 1,000,000
    long long r = (long long)sqrt((double)x);
    cout << r << "\n"; // => may print 999999 on some platforms due to FP rounding, instead of 1000000
}
```

**Correct integer-sqrt idiom** — compute with `sqrt`, then correct with a small loop:

```cpp
long long isqrt(long long x) {
    long long r = (long long)sqrtl((long double)x); // long double: fewer rounding issues
    while (r > 0 && r * r > x) r--;   // step down if overshot
    while ((r + 1) * (r + 1) <= x) r++; // step up if undershot
    return r;
}
// isqrt(1000000000000LL) => 1000000  (exact, corrected)
```

**Integer power via fast exponentiation** (avoid `pow` for integer results — it returns `double` and loses precision for large exponents):

```cpp
long long ipow(long long base, long long exp, long long mod = 0) {
    long long result = 1;
    base %= (mod ? mod : LLONG_MAX); // keep base bounded if mod given
    while (exp > 0) {
        if (exp & 1) {
            result = mod ? (result * base) % mod : result * base;
        }
        base = mod ? (base * base) % mod : base * base;
        exp >>= 1;
    }
    return result;
}
// ipow(2, 62) => 4611686018427387904
// ipow(3, 10, 1000000007) => 59049
```

> **Gotcha:** `pow(2, 62)` returns a `double`; doubles only have ~15-17 significant decimal digits, so large integer results silently lose precision. Always use fast-exponentiation for integer/modular powers.

### 25.8 `gcd`/`lcm` **[C++17]**

Available in `<numeric>` since C++17. Older code (and many CP templates) still uses `__gcd` from `<algorithm>` (GCC extension, pre-dates the standard version).

```cpp
#include <numeric>   // std::gcd, std::lcm — C++17
#include <algorithm> // __gcd — GCC extension, works in any standard mode
#include <iostream>
using namespace std;
int main() {
    cout << gcd(12, 18) << "\n";   // => 6      [C++17]
    cout << lcm(4, 6) << "\n";     // => 12     [C++17]
    cout << __gcd(12, 18) << "\n"; // => 6      (GCC extension, pre-C++17 friendly)
}
```

| Feature | Header | Since | Portability |
|---|---|---|---|
| `std::gcd`, `std::lcm` | `<numeric>` | C++17 | standard, portable |
| `__gcd` | `<algorithm>` | always (libstdc++) | **libstdc++ only** — see gotcha |

> **Gotcha:** `__gcd` is a **libstdc++ (GNU) extension**, not a language feature — it is fine on
> Codeforces and LeetCode (both use GCC), but it is NOT portable. On **libc++** (Apple clang, the
> default toolchain on macOS) there is a different, internal `std::__gcd` that requires *unsigned*
> types. With `using namespace std;` in scope, `__gcd(12, 18)` then resolves to that internal
> template and **fails to compile** with a `static assertion failed ... !is_signed<int>::value`
> error pointing deep inside `<numeric>` — a genuinely baffling error message if you don't know
> the cause. If you compile locally on a Mac, use `std::gcd`/`std::lcm` (C++17) and reserve
> `__gcd` for contest submissions only.

---

## 26. Floating Point

### 26.1 IEEE754 basics

| Type | Size | Decimal digits of precision | Typical use |
|---|---|---|---|
| `float` | 4 bytes | ~6-7 | rarely used in CP (too imprecise) |
| `double` | 8 bytes | ~15-17 | default choice |
| `long double` | 8, 12, or 16 bytes (platform-dependent: 8 on MSVC, 80-bit extended on x86 GCC/Clang) | ~18-19 (x86 GCC) | extra precision margin in CP |

```cpp
#include <iostream>
#include <limits>
using namespace std;
int main() {
    cout << numeric_limits<float>::digits10 << "\n";       // => 6
    cout << numeric_limits<double>::digits10 << "\n";      // => 15
    cout << numeric_limits<long double>::digits10 << "\n"; // => 18 (on x86 Linux/GCC; 15 on MSVC)
}
```

### 26.2 Never compare floats with `==`

```cpp
#include <iostream>
using namespace std;
int main() {
    double a = 0.1 + 0.2;
    cout << (a == 0.3) << "\n";        // => 0  (false! 0.1+0.2 = 0.30000000000000004...)
    cout.precision(20);
    cout << a << "\n";                  // => 0.30000000000000004441
}
```

**Epsilon comparison** — absolute vs relative:

```cpp
#include <cmath>
#include <algorithm>
using namespace std;

// Absolute epsilon: good when magnitudes are small/bounded (e.g. coordinates in [-1000, 1000])
bool almostEqualAbs(double a, double b, double eps = 1e-9) {
    return fabs(a - b) < eps;
}

// Relative epsilon: good when magnitudes vary wildly (1e-3 vs 1e9)
bool almostEqualRel(double a, double b, double relEps = 1e-9) {
    double diff = fabs(a - b);
    double largest = max(fabs(a), fabs(b));
    return diff <= largest * relEps;
}
```

| Choose | When |
|---|---|
| Absolute epsilon | Values are known to be in a bounded, moderate range |
| Relative epsilon | Values span many orders of magnitude |
| Combined (`diff < eps_abs + eps_rel*max(|a|,|b|)`) | General-purpose, robust default |

> **Gotcha:** near zero, relative comparison breaks down (dividing by ~0), and near huge values, absolute comparison is too strict. Libraries like Google Test's `ASSERT_NEAR` mix both; for CP, a single well-chosen absolute epsilon (`1e-6` to `1e-9` depending on how many operations accumulated error) is usually enough.

### 26.3 Accumulation error

```cpp
#include <iostream>
using namespace std;
int main() {
    double sum = 0.0;
    for (int i = 0; i < 100000; i++) sum += 0.1;
    cout.precision(17);
    cout << sum << "\n";          // => 10000.000000064305 (NOT exactly 10000 due to repeated rounding)
}
```

Repeated addition of numbers that aren't exactly representable in binary (like `0.1`) accumulates rounding error. For sums of many terms, prefer **Kahan summation** or reorder to sum small values first, or better: avoid floats entirely (see 26.5).

### 26.4 `INFINITY`, `NAN`, `isnan`, `isinf`

```cpp
#include <cmath>
#include <iostream>
using namespace std;
int main() {
    double inf = INFINITY;
    double nan = NAN;
    cout << inf << " " << -inf << "\n";        // => inf -inf
    cout << (nan == nan) << "\n";                // => 0   (NaN never equals itself!)
    cout << isnan(nan) << " " << isinf(inf) << "\n"; // => 1 1
    cout << (1.0 / 0.0) << "\n";                  // => inf (no exception thrown for doubles)
}
```

> **Gotcha:** `nan == nan` is always `false` by IEEE754 definition — the only correct way to test for NaN is `isnan(x)`, never `x == x` negated by hand incorrectly or `x != x` (though `x != x` does technically work, `isnan` is clearer).

### 26.5 Printing with fixed precision

```cpp
#include <iostream>
#include <iomanip>
using namespace std;
int main() {
    double pi = 3.14159265358979;
    cout << fixed << setprecision(9) << pi << "\n"; // => 3.141592654
    cout << setprecision(2) << pi << "\n";            // => 3.14   (fixed still active)
    cout.unsetf(ios::fixed);
    cout << setprecision(4) << pi << "\n";            // => 3.142  (default sig-fig mode)
}
```

### 26.6 When to avoid floating point entirely in DSA

Many CP problems that "look" like they need floats can be solved exactly with integers/rationals — always prefer this when possible, since it sidesteps epsilon bugs entirely.

**Comparing fractions via cross-multiplication** (avoid `a/b < c/d` as doubles):

```cpp
#include <iostream>
using namespace std;
// compares a/b vs c/d assuming b, d > 0 -- exact, no floating point
bool fracLess(long long a, long long b, long long c, long long d) {
    return a * d < c * b; // cross-multiply; use __int128 if a,b,c,d can be large
}
int main() {
    cout << fracLess(1, 3, 1, 2) << "\n"; // 1/3 < 1/2 => 1
}
```

**Integer-coordinate geometry** — use cross products instead of angles/slopes:

```cpp
#include <iostream>
using namespace std;
struct Pt { long long x, y; };

// cross product of (b-a) x (c-a); sign tells orientation, zero-error-free
long long cross(Pt a, Pt b, Pt c) {
    return (b.x - a.x) * (c.y - a.y) - (b.y - a.y) * (c.x - a.x);
}
int main() {
    Pt a{0,0}, b{4,0}, c{4,4};
    cout << cross(a,b,c) << "\n"; // => 16 (positive => counter-clockwise turn)
}
```

| Instead of | Use |
|---|---|
| Slope `(y2-y1)/(x2-x1)` | Cross product for orientation/collinearity |
| Fraction comparison as `double` | Cross-multiplication (watch overflow, use `long long`/`__int128`) |
| Distance comparisons via `sqrt` | Compare squared distances (avoids `sqrt` entirely) |
| Angle sorting | `atan2` only if unavoidable; otherwise cross-product-based comparators |

### 26.7 `long double` in CP

`long double` (80-bit extended precision on x86 GCC/Clang, giving ~18-19 significant digits) is a cheap way to reduce — not eliminate — floating error margin in geometry/probability problems, at negligible performance cost on most judges.

```cpp
#include <iostream>
using namespace std;
int main() {
    long double x = 1.0L / 3.0L;
    cout.precision(20);
    cout << x << "\n"; // => 0.3333333333333333315 (more digits held than double, still not exact)
}
```

> **Gotcha:** `long double` is NOT guaranteed extended precision — on MSVC it's identical to `double` (64-bit). Don't rely on the extra precision for portable code; it's a "free" upgrade on GCC/Clang judges (Codeforces, most online judges) but not universal.

---

## 27. Input and Output

### 27.1 `cin`/`cout` basics and reading until EOF

```cpp
#include <iostream>
using namespace std;
int main() {
    int a, b;
    cin >> a >> b;              // whitespace/newline-delimited, skips leading whitespace
    cout << a + b << "\n";
}
// input: 3 4
// => 7
```

Reading an unknown-length stream until end-of-file:

```cpp
#include <iostream>
using namespace std;
int main() {
    int x, sum = 0;
    while (cin >> x) sum += x;  // cin>>x fails (returns falsy) at EOF or on bad input
    cout << sum << "\n";
}
// input: 1 2 3 4 5  (then EOF, e.g. Ctrl-D)
// => 15
```

### 27.2 `getline` and the `>>`/`getline` mixing trap

`operator>>` leaves the trailing `'\n'` in the buffer. A subsequent `getline` immediately reads that leftover empty line instead of the next real line.

```cpp
#include <iostream>
#include <string>
using namespace std;
int main() {
    int n;
    string line;
    cin >> n;             // reads "3", leaves "\n" in the buffer
    getline(cin, line);    // BUG: reads the leftover empty "" before the real line
    cout << "[" << line << "]\n"; // => [] (empty, not the intended line!)
}
// input:
// 3
// hello world
```

**The exact fix** — consume the leftover newline first:

```cpp
#include <iostream>
#include <string>
#include <limits>
using namespace std;
int main() {
    int n;
    string line;
    cin >> n;
    cin.ignore(numeric_limits<streamsize>::max(), '\n'); // discard rest of current line incl. '\n'
    getline(cin, line);
    cout << "[" << line << "]\n"; // => [hello world]
}
```

`cin.ignore(numeric_limits<streamsize>::max(), '\n')` discards everything up to and including the next `'\n'` (the `max()` count means "no practical limit" — plain `cin.ignore()` only skips ONE character by default).

### 27.3 Reading n then n values

```cpp
#include <iostream>
#include <vector>
using namespace std;
int main() {
    int n; cin >> n;
    vector<int> v(n);
    for (int i = 0; i < n; i++) cin >> v[i];
    for (int x : v) cout << x << " ";
    cout << "\n";
}
// input: 4 10 20 30 40
// => 10 20 30 40
```

### 27.4 Reading a 2D grid

**Grid of chars** (no spaces between characters — read whole rows as strings):

```cpp
#include <iostream>
#include <vector>
#include <string>
using namespace std;
int main() {
    int rows, cols; cin >> rows >> cols;
    vector<string> grid(rows);
    for (int i = 0; i < rows; i++) cin >> grid[i]; // >> on string reads one whitespace-delimited token
    cout << grid[0][0] << grid[rows-1][cols-1] << "\n";
}
// input:
// 3 3
// abc
// def
// ghi
// => ai
```

**Grid of ints** (space-separated):

```cpp
#include <iostream>
#include <vector>
using namespace std;
int main() {
    int rows, cols; cin >> rows >> cols;
    vector<vector<int>> grid(rows, vector<int>(cols));
    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            cin >> grid[i][j];
    cout << grid[1][1] << "\n";
}
// input:
// 2 2
// 1 2
// 3 4
// => 4
```

### 27.5 `cin.fail()` and clearing state

Once a stream extraction fails (e.g. reading letters into an `int`), the stream enters a failed state and ALL subsequent reads silently no-op until cleared.

```cpp
#include <iostream>
using namespace std;
int main() {
    int x;
    cin >> x; // input "abc" -- fails, x may be set to 0 (C++11+) and failbit is set
    if (cin.fail()) {
        cout << "read failed\n";              // => read failed
        cin.clear();                            // reset the error flags
        cin.ignore(1000, '\n');                 // discard the bad token/rest of line
    }
}
// input: abc
```

> **Gotcha:** `cin.clear()` only resets the stream's error FLAGS — it does NOT discard the bad characters still sitting in the buffer. You must also `cin.ignore(...)` (or otherwise consume) the offending input, or the next read will fail again on the same garbage.

### 27.6 Fast I/O

```cpp
#include <iostream>
using namespace std;
int main() {
    ios_base::sync_with_stdio(false); // decouple C++ streams from C stdio (stdout/stdin/printf/scanf)
    cin.tie(nullptr);                  // untie cin from cout: cout is no longer auto-flushed before each cin
    // ... rest of program using cin/cout normally, now much faster
}
```

| Call | What it does |
|---|---|
| `ios_base::sync_with_stdio(false)` | Stops C++ streams from staying byte-for-byte synchronized with C's `stdio` (`printf`/`scanf`), enabling internal buffering optimizations |
| `cin.tie(nullptr)` | Un-links `cin` from `cout`; normally `cin >>` implicitly flushes `cout` first (so prompts show before input) — untying removes that automatic flush for speed |

> **Gotcha:** After `sync_with_stdio(false)`, you must NOT mix `printf`/`scanf` with `cin`/`cout` in the same program — their buffers are no longer synchronized and output can interleave in the wrong order or get corrupted. Pick one family and stick to it.

### 27.7 Why `endl` is slow

```cpp
cout << "line1" << endl;  // outputs "line1\n" AND flushes the output buffer (forces a syscall)
cout << "line2" << "\n";  // outputs "line2\n", no flush -- buffered, batched, faster
```

For programs with many lines of output, `endl` in a loop can be a real performance bottleneck (thousands of flush syscalls). Use `'\n'` and let the buffer flush naturally on program exit (or once at the very end if you need to guarantee visibility).

### 27.8 `<iomanip>` manipulators

```cpp
#include <iostream>
#include <iomanip>
using namespace std;
int main() {
    cout << fixed << setprecision(3) << 3.14159 << "\n"; // => 3.142
    cout << setw(6) << setfill('0') << 42 << "\n";         // => 000042
    cout << boolalpha << true << " " << false << "\n";      // => true false
    cout << hex << 255 << " " << dec << 255 << " " << oct << 8 << "\n"; // => ff 255 10
}
```

| Manipulator | Effect |
|---|---|
| `setprecision(n)` | number of digits (significant, or after decimal if `fixed`) |
| `fixed` | force fixed-point notation |
| `setw(n)` | minimum field width for the NEXT output only (not sticky) |
| `setfill(c)` | fill character used to pad `setw` |
| `boolalpha` | print `bool` as `true`/`false` instead of `1`/`0` |
| `hex` / `dec` / `oct` | base for integer input/output (sticky until changed) |

> **Gotcha:** `setw` applies only to the very next inserted value — it resets after each use, unlike `fixed`/`hex`/`boolalpha` which stay in effect until explicitly changed.

### 27.9 `printf`/`scanf` format specifiers

| Specifier | Type | Example |
|---|---|---|
| `%d` | `int` | `printf("%d", 42)` |
| `%lld` | `long long` | `printf("%lld", 123456789012LL)` |
| `%u` | `unsigned int` | `printf("%u", 4000000000u)` |
| `%llu` | `unsigned long long` | `printf("%llu", x)` |
| `%f` | `float` (printf promotes to double; scanf needs `%f` for float) | `printf("%f", 3.14f)` |
| `%lf` | `double` — required for `scanf` (printf accepts `%f` for double too) | `scanf("%lf", &d)` |
| `%c` | `char` | `printf("%c", 'A')` |
| `%s` | `char*` (C-string) | `printf("%s", "hi")` |
| `%p` | pointer | `printf("%p", (void*)&x)` |
| `%x` | hex (lowercase) | `printf("%x", 255)` => `ff` |

```cpp
#include <cstdio>
int main() {
    int a = 5; long long b = 10000000000LL; unsigned u = 4000000000u;
    double d = 3.14159; char c = 'Z'; const char* s = "hi";
    printf("%d %lld %u %f %c %s %x\n", a, b, u, d, c, s, 255);
    // => 5 10000000000 4000000000 3.141590 Z hi ff

    double x;
    scanf("%lf", &x); // NOTE: scanf needs %lf for double, unlike printf which tolerates %f
}
```

> **Gotcha:** For `scanf` into a `double`, you MUST use `%lf` — using `%f` writes only 4 bytes and corrupts the `double`'s memory (UB). For `printf`, `%f` and `%lf` are equivalent for a `double` argument because `float` args are promoted to `double` in variadic calls — but this equivalence does NOT hold for `scanf`, which writes directly into the pointed-to memory with no promotion.

### 27.10 Flushing for interactive problems

In interactive problems (judge reads your output and responds), buffered output must be explicitly flushed after each query, or the judge may hang waiting for input that's sitting unflushed in your buffer.

```cpp
#include <iostream>
using namespace std;
int main() {
    int guess = 50;
    cout << guess << endl;   // endl flushes -- correct for interactive I/O
    // OR: cout << guess << "\n" << flush;
    // OR: cout << guess << "\n"; cout.flush();
    int response; cin >> response; // now safe to block waiting for judge's reply
}
```

> **Gotcha:** This is the ONE context where `endl`'s flush is a feature, not a bug — in interactive problems, skipping the flush (using bare `"\n"` without an explicit `flush`) can cause your program to hang forever, since the judge never sees your query.

### 27.11 A getchar-based fast reader

For inputs with millions of integers where even `sync_with_stdio(false)` isn't fast enough, a raw `getchar`-based reader is the standard CP fallback.

```cpp
#include <cstdio>
#include <iostream>
using namespace std;

inline int fastReadInt() {
    int x = 0, sign = 1;
    int c = getchar_unlocked(); // getchar_unlocked: unlocked (non-thread-safe) getchar, faster
    while (c < '0' || c > '9') {
        if (c == '-') sign = -1;
        c = getchar_unlocked();
    }
    while (c >= '0' && c <= '9') {
        x = x * 10 + (c - '0');
        c = getchar_unlocked();
    }
    return x * sign;
}

int main() {
    int n = fastReadInt();
    long long sum = 0;
    for (int i = 0; i < n; i++) sum += fastReadInt();
    cout << sum << "\n";
}
// input: 3 -1 2 3
// => 4
```

> **Gotcha:** `getchar_unlocked` is POSIX/GCC-specific (not standard C++, unavailable on MSVC) — use plain `getchar` for portability at a small speed cost. Also this reader assumes integers only, no leading `+`, and stops correctly only when non-digit/non-`-` characters (including EOF as `-1`) are encountered.

---

## 28. Randomness, Time and Utility Headers

### 28.1 `<random>` done properly

`rand()` (paired with `srand(time(0))`) is inadequate for two reasons: (1) its range is tiny (`RAND_MAX` is often only 32767, e.g. on MSVC), and (2) its statistical quality is poor (low-order bits cycle predictably on many implementations). Prefer `<random>`'s Mersenne Twister engines.

```cpp
#include <random>
#include <chrono>
#include <iostream>
using namespace std;

int main() {
    // Seed with a high-resolution time-based value -- NOT a fixed constant
    mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());

    uniform_int_distribution<int> dist(1, 6); // inclusive [1, 6], like a die
    for (int i = 0; i < 5; i++) cout << dist(rng) << " ";
    cout << "\n"; // => 5 numbers in [1,6], different each run
}
```

64-bit variant for larger ranges:

```cpp
#include <random>
#include <chrono>
#include <iostream>
using namespace std;
int main() {
    mt19937_64 rng64(chrono::steady_clock::now().time_since_epoch().count());
    uniform_int_distribution<long long> dist(1, 1000000000000LL);
    cout << dist(rng64) << "\n"; // => some value in [1, 1e12]
}
```

> **Gotcha — fixed seeds are hackable on Codeforces:** If you seed with a fixed constant (e.g. `mt19937 rng(12345);` or worse, no seed at all which defaults to a fixed value), adversarial test setters — and other competitors — can PRECOMPUTE which "random" values your program will produce, then craft inputs that defeat your randomized algorithm (e.g. randomized pivot quickselect, treap priorities). This is a well-known Codeforces "hack" vector. Always seed from `chrono` (or combine with `random_device` where available) so your sequence is unpredictable per-submission.

```cpp
// A commonly recommended CP seeding idiom, combining random_device with time:
mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());
// Some templates additionally XOR in random_device{}() for extra entropy:
mt19937 rng2(chrono::steady_clock::now().time_since_epoch().count() ^ random_device{}());
```

### 28.2 `uniform_int_distribution` and `shuffle`

```cpp
#include <random>
#include <algorithm>
#include <vector>
#include <chrono>
#include <iostream>
using namespace std;
int main() {
    mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());

    vector<int> v = {1, 2, 3, 4, 5};
    shuffle(v.begin(), v.end(), rng); // Fisher-Yates shuffle using the given engine
    for (int x : v) cout << x << " ";
    cout << "\n"; // => a random permutation of 1..5, e.g. "3 1 5 2 4"
}
```

> **Gotcha:** `std::random_shuffle` (pre-C++17, using `rand()` internally) was REMOVED in C++17. Always use `std::shuffle` with an explicit engine like `mt19937`.

### 28.3 `<chrono>` for benchmarking

```cpp
#include <chrono>
#include <iostream>
using namespace std;
using namespace std::chrono;

int main() {
    auto start = steady_clock::now();

    long long sum = 0;
    for (int i = 0; i < 100000000; i++) sum += i;

    auto end = steady_clock::now();
    auto ms = duration_cast<milliseconds>(end - start).count();
    cout << "sum=" << sum << " elapsed=" << ms << "ms\n";
    // => sum=4999999950000000 elapsed=<some number>ms (varies by machine)
}
```

`steady_clock` is monotonic (never goes backward, unaffected by system clock adjustments) — always prefer it over `system_clock` for measuring elapsed time/durations. Use `system_clock` only when you need wall-clock/calendar time.

### 28.4 `<bits/stdc++.h>`

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    vector<int> v = {3,1,2};
    sort(v.begin(), v.end());
    cout << v[0] << "\n"; // => 1
}
```

`<bits/stdc++.h>` is a single umbrella header, internal to the GNU C++ standard library (libstdc++), that transitively includes nearly the entire standard library (`<vector>`, `<algorithm>`, `<string>`, `<cmath>`, ...). It exists purely as a GCC implementation convenience.

| Pros | Cons |
|---|---|
| One include, saves typing in contests | **Not part of the C++ standard** — undefined behavior to rely on it existing |
| Fine on Codeforces, AtCoder, most LeetCode-style judges (all GCC-based) | Fails to compile on MSVC and Apple Clang (no such header shipped) |
| Slightly slower compile (parses the whole stdlib every time) | Signals "contest code", not appropriate for production/portable projects |

> **Gotcha:** If you write code on a Mac using the Apple Clang toolchain (the macOS default `g++`/`clang++` is actually Clang), `#include <bits/stdc++.h>` will fail with "file not found" — Apple's libc++ doesn't ship it. Either install GCC via Homebrew, or just include the specific standard headers you need, which is also better practice outside of contests.

### 28.5 Standard headers reference table

| Header | Provides |
|---|---|
| `<vector>` | `std::vector` (dynamic array) |
| `<string>` | `std::string`, string ops |
| `<algorithm>` | `sort`, `find`, `binary_search`, `reverse`, `min`/`max`, `__gcd`, etc. |
| `<numeric>` | `accumulate`, `iota`, `gcd`/`lcm` (C++17), `partial_sum`, `inner_product` |
| `<unordered_map>` | `std::unordered_map`, `std::unordered_multimap` (hash table) |
| `<queue>` | `std::queue`, `std::priority_queue` |
| `<stack>` | `std::stack` |
| `<set>` | `std::set`, `std::multiset` (ordered, red-black tree) |
| `<map>` | `std::map`, `std::multimap` (ordered, red-black tree) |
| `<cmath>` | `sqrt`, `pow`, `floor`, `ceil`, `abs` (floating), trig, `log`/`log2`/`exp` |
| `<climits>` | `INT_MAX`, `INT_MIN`, `LLONG_MAX`, `UINT_MAX`, etc. (C-style limit macros) |
| `<cstring>` | C string functions: `memset`, `memcpy`, `strlen`, `strcmp` |
| `<iomanip>` | `setprecision`, `fixed`, `setw`, `setfill`, `boolalpha`, `hex`/`dec`/`oct` |
| `<sstream>` | `std::stringstream`, `istringstream`, `ostringstream` (string-based streams) |
| `<bitset>` | `std::bitset<N>` (fixed-size bit array) |
| `<random>` | `mt19937`, `uniform_int_distribution`, `shuffle`-compatible engines |
| `<chrono>` | `steady_clock`, `system_clock`, `duration_cast`, timing utilities |
| `<functional>` | `std::function`, `std::greater`/`less`, `std::bind`, hashing functors |
| `<utility>` | `std::pair`, `std::swap`, `std::move`, `std::forward` |
| `<tuple>` | `std::tuple`, `std::tie`, `std::make_tuple`, structured-binding support |

```cpp
// Minimal, portable equivalent of a typical <bits/stdc++.h>-based CP template:
#include <vector>
#include <string>
#include <algorithm>
#include <numeric>
#include <unordered_map>
#include <map>
#include <set>
#include <queue>
#include <stack>
#include <cmath>
#include <climits>
#include <cstring>
#include <iomanip>
#include <sstream>
#include <bitset>
#include <random>
#include <chrono>
#include <functional>
#include <utility>
#include <tuple>
#include <iostream>
using namespace std;

int main() {
    cout << "portable across GCC, Clang, and MSVC\n";
    // => portable across GCC, Clang, and MSVC
}
```

---

# PART VII — THE COMPETITIVE PROGRAMMING TOOLKIT

## 29. The Contest Template and Macros

### 29.1 The full copy-paste template

```cpp
#include <bits/stdc++.h>
using namespace std;

typedef long long ll;
typedef unsigned long long ull;
typedef long double ld;
typedef pair<int,int> pii;
typedef pair<ll,ll> pll;
typedef vector<int> vi;
typedef vector<ll> vl;
typedef vector<pii> vpii;

#define all(x) (x).begin(), (x).end()
#define rall(x) (x).rbegin(), (x).rend()
#define sz(x) (int)(x).size()
#define pb push_back
#define eb emplace_back
#define F first
#define S second
#define rep(i, a, b) for (int i = (a); i < (b); ++i)
#define rrep(i, a, b) for (int i = (a); i >= (b); --i)

const int MOD = 1e9 + 7;
const ll INF = 4e18;

#ifdef LOCAL
#include "debug.h"
#else
#define dbg(...)
#endif

void solve() {
    int n;
    cin >> n;
    vi a(n);
    for (auto &x : a) cin >> x;

    // ---- solution logic goes here ----

    cout << "\n";
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int t = 1;
    cin >> t;
    while (t--) {
        solve();
    }
    return 0;
}
```

### 29.2 Line-by-line breakdown

| Piece | What it does | Why |
|---|---|---|
| `#include <bits/stdc++.h>` | Pulls in almost the entire STL in one header | Saves time hunting includes; GCC-only, not portable, fine for CF/LeetCode judges (both use GCC/Clang with it whitelisted) |
| `using namespace std;` | Drops the `std::` prefix everywhere | Shortens code; risky in large real projects, fine in a single ~100-line file |
| `typedef long long ll;` | Renames `long long` to `ll` | `int` overflows past ~2.1×10⁹; most CF sums need 64-bit |
| `pii`, `vi`, `vpii` | Short names for common container types | Less visual noise when declaring `vector<pair<int,int>> v;` a dozen times |
| `all(x)` → `(x).begin(), (x).end()` | Expands to the begin/end pair | `sort(all(v))` instead of `sort(v.begin(), v.end())` |
| `sz(x)` | Cast `.size()` to `int` | `.size()` returns `size_t` (unsigned); mixing it with `int` in comparisons/subtraction is a classic underflow bug (`sz(x)` sidesteps that) |
| `rep(i,a,b)` | `for (int i=a; i<b; ++i)` | Terser loops; half-open `[a,b)` convention matches STL |
| `pb`, `eb`, `F`, `S` | `push_back`, `emplace_back`, `.first`, `.second` | Keystroke savings that add up over a 2-hour contest |
| `MOD`, `INF` | Common constants | `1e9+7` is the standard modulus; `INF = 4e18` is safely below `LLONG_MAX` (~9.2×10¹⁸) so `INF + INF` doesn't overflow |
| `#ifdef LOCAL ... dbg(...)` | Debug macro that vanishes on submission | See §29.5 |
| `void solve()` | One test case's logic | Isolates per-case state so nothing leaks between test cases in the `while(t--)` loop |
| `ios_base::sync_with_stdio(false); cin.tie(NULL);` | Detaches C++ streams from C `stdio` and un-links `cin` from auto-flushing `cout` | Can turn a 2-second TLE into a 200ms pass on large I/O; **never** mix `scanf/printf` with `cin/cout` afterward if you do this |
| `int t=1; cin>>t;` | Reads test count, defaults to 1 if omitted | Matches both "single test" (LeetCode-style local testing) and "T test cases" (CF) problems with one template |

> **Gotcha:** `sync_with_stdio(false)` + `cin.tie(NULL)` breaks interleaved `cin`/`scanf` or `cout`/`printf` output ordering. Pick one I/O family per program.

> **Gotcha:** `#include <bits/stdc++.h>` does not exist on MSVC/Clang-on-macOS-with-libc++ by default (it's a GCC libstdc++ convenience header). Codeforces and most CP judges use GCC, so it's safe there; some interview platforms use Clang and will reject it — include specific headers (`<vector>`, `<algorithm>`, ...) for those.

### 29.3 The macro debate — FOR contests, AGAINST interviews

| | Contests (CF/AtCoder) | Interviews (LeetCode-style, live coding) |
|---|---|---|
| Time pressure | Extreme — every keystroke counts across 5-8 problems | Low — one problem, 30-45 min, judged on clarity too |
| Reader | Only you, ever | An interviewer reading over your shoulder |
| Macros (`rep`, `pb`, `all`) | **Use them.** Muscle memory outweighs readability cost | **Avoid them.** They signal "I memorized a template" not "I can reason about code," and force the interviewer to mentally expand every macro |
| `using namespace std;` | Fine | Mildly frowned upon in production-style interviews; fine for LeetCode's isolated function scope |
| Naming | `a`, `b`, `n`, `dp` — terse is normal | Prefer descriptive names (`nums`, `left`, `right`, `dp`) — you're demonstrating communication, not just correctness |
| Verdict | Optimize for **speed of writing** | Optimize for **speed of reading** |

**Rule of thumb:** keep two mental templates. Paste the terse one into `sol.cpp` for a CF round; write clean, macro-free, well-named code in a LeetCode/interview tab.

### 29.4 Minimal LeetCode skeleton (macro-free)

```cpp
class Solution {
public:
    int myFunction(vector<int>& nums, int target) {
        int n = nums.size();
        // clean, no macros, descriptive names — this is what an interviewer sees
        for (int i = 0; i < n; ++i) {
            if (nums[i] == target) return i;
        }
        return -1;
    }
};
```

### 29.5 The debug macro — prints `name = value`, compiles to nothing on submission

```cpp
#ifdef LOCAL
template <typename T>
void __print(const T &x) { cerr << x; }
void __print(const string &x) { cerr << '"' << x << '"'; }
void __print(char x) { cerr << "'" << x << "'"; }
void __print(bool x) { cerr << (x ? "true" : "false"); }

template <typename T, typename V>
void __print(const pair<T, V> &x) {
    cerr << '{'; __print(x.first); cerr << ", "; __print(x.second); cerr << '}';
}
template <typename T>
void __print(const vector<T> &x) {
    cerr << '[';
    for (size_t i = 0; i < x.size(); ++i) { if (i) cerr << ", "; __print(x[i]); }
    cerr << ']';
}

void _dbg() { cerr << "]\n"; }
template <typename T, typename... Rest>
void _dbg(T t, Rest... rest) {
    __print(t);
    if (sizeof...(rest)) cerr << ", ";
    _dbg(rest...);
}
#define dbg(...) cerr << "[" << #__VA_ARGS__ << "] = ["; _dbg(__VA_ARGS__)
#else
#define dbg(...)
#endif
```

Usage:

```cpp
int x = 5;
vector<int> v = {1, 2, 3};
dbg(x, v);
// LOCAL build prints:  [x, v] = [5, [1, 2, 3]]
// Submitted build: dbg(...) expands to nothing — zero runtime cost, no compile error even if unused
```

> **Gotcha:** the `#__VA_ARGS__` stringize trick reprints your exact source text (`x, v`) as the label — extremely useful for locating *which* `dbg` call fired when you have twenty of them. Define `LOCAL` only via your local build command (`g++ -DLOCAL ...`); never define it in a file that gets submitted.

---

## 30. PBDS — The Hidden Ordered Set

GCC ships **Policy-Based Data Structures** in libstdc++ as an extension. They are not part of standard C++ but are available (and allowed) on Codeforces, AtCoder, and most CP judges compiled with g++.

### 30.1 Includes and the `ordered_set` typedef

```cpp
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>
using namespace __gnu_pbds;

typedef tree<
    int,                          // key type
    null_type,                    // mapped type (null_type = "no value", i.e. a set not a map)
    less<int>,                    // comparator (use less_equal<int> for multiset behavior, see 30.4)
    rb_tree_tag,                  // underlying structure: red-black tree
    tree_order_statistics_node_update  // policy enabling order statistics
> ordered_set;
```

This behaves like `std::set<int>` (insert, erase, find, iteration, all O(log n)) **plus two superpowers**.

### 30.2 The two superpowers

| Method | Meaning | Complexity |
|---|---|---|
| `order_of_key(x)` | Count of elements **strictly less than** `x` currently in the set | O(log n) |
| `find_by_order(k)` | Iterator to the `k`-th smallest element (0-indexed) | O(log n) |

```cpp
ordered_set s;
s.insert(1); s.insert(5); s.insert(3); s.insert(9);
// sorted contents: 1 3 5 9

cout << s.order_of_key(5) << "\n";   // 2  (1 and 3 are < 5)
cout << s.order_of_key(4) << "\n";   // 2  (works even if 4 isn't present)
cout << *s.find_by_order(0) << "\n"; // 1  (smallest)
cout << *s.find_by_order(2) << "\n"; // 5  (3rd smallest, 0-indexed)
```

> **Gotcha:** `find_by_order(k)` for `k >= s.size()` returns `s.end()` — dereferencing it is UB. Bounds-check like any other iterator.

### 30.3 Worked example 1 — counting inversions

An inversion is a pair `(i, j)` with `i < j` but `a[i] > a[j]`. Insert elements left to right; before inserting `a[i]`, the count of already-inserted elements **greater than** `a[i]` is `(elements so far) - order_of_key(a[i])`.

```cpp
long long countInversions(vector<int>& a) {
    ordered_set s;
    long long inversions = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        inversions += i - s.order_of_key(a[i]); // # already-inserted elements > a[i]
        s.insert(a[i]);
    }
    return inversions;
}
```

(For dense value ranges a Fenwick/BIT is usually preferred for raw speed — see Pattern 27/28 in this reference — but PBDS is faster to *write* under contest pressure.)

### 30.4 Worked example 2 — k-th order statistic with insert/erase

```cpp
ordered_set s;
for (int x : {7, 2, 9, 4, 1}) s.insert(x);
// sorted: 1 2 4 7 9
cout << *s.find_by_order(2) << "\n"; // 4  -> 3rd smallest live element
s.erase(4);
cout << *s.find_by_order(2) << "\n"; // 7  -> order statistics update automatically after erase
```

### 30.5 Supporting duplicates

`tree` is built on a strict order and, like `std::set`, silently drops duplicate keys with `less<int>`. Two fixes:

**Option A — pair with a unique tiebreaker (recommended, always safe):**

```cpp
typedef tree<pair<int,int>, null_type, less<pair<int,int>>,
             rb_tree_tag, tree_order_statistics_node_update> ordered_multiset;

ordered_multiset ms;
ms.insert({5, 0});  // second component = insertion index, guarantees uniqueness
ms.insert({5, 1});
ms.insert({3, 2});
// order_of_key({5, 0}) counts everything strictly less than (5,0)
// -> to count elements <= x, query order_of_key({x, INT_MAX})
cout << ms.order_of_key({5, INT_MAX}) << "\n"; // counts all elements with key <= 5
```

**Option B — `less_equal` comparator (works, but erase is broken):**

```cpp
typedef tree<int, null_type, less_equal<int>,
             rb_tree_tag, tree_order_statistics_node_update> ordered_multiset_le;
```

`less_equal` lets `insert` accept duplicates, but calling `.erase(x)` directly can corrupt the tree/erase the wrong duplicate. The safe erase pattern is:

```cpp
ordered_multiset_le t;
t.insert(5); t.insert(5); t.insert(3);
// to erase ONE copy of 5:
int k = t.order_of_key(5);          // index of first 5
t.erase(t.find_by_order(k));        // erase by iterator, not by key
```

> **Gotcha:** prefer Option A in real contests. `less_equal` erase footguns have cost people rounds; the pair-with-index trick is strictly safer for the same asymptotic cost.

### 30.6 `gp_hash_table` — a faster `unordered_map`

```cpp
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/hash_policy.hpp>
using namespace __gnu_pbds;

gp_hash_table<int, int> table; // ~2-3x faster than std::unordered_map in practice
table[5] = 10;
if (table.find(5) != table.end()) cout << table[5] << "\n";
```

It supports the same interface (`[]`, `find`, `erase`) as `unordered_map` but with open addressing instead of chaining — better cache locality, less pointer-chasing. It is **just as hackable** as `unordered_map` with the default hash (see §31.1), so pair it with `custom_hash` too if the keys are adversary-controlled.

### 30.7 The critical caveat

> **Gotcha: PBDS is a GCC libstdc++ extension. It compiles on Codeforces, AtCoder, and most CP judges (all GCC-based) but will NOT compile on Clang-only or MSVC-only environments — including many technical interview platforms and LeetCode's C++ compiler in some configurations.** Never rely on it in an interview. The portable, judge-agnostic alternative for order statistics is a **Fenwick tree (BIT) over compressed coordinates** or a **segment tree with counts** — see Patterns 27 and 28 in this reference for the manual O(log n) rank/select implementation that works everywhere.

---

## 31. Performance, Hacking and Anti-Test Defenses

### 31.1 Why `unordered_map` gets hacked on Codeforces

Codeforces problems are sometimes followed by **hacks**: other competitors submit adversarial inputs against your accepted solution during the challenge phase. GCC's `unordered_map`/`unordered_set` use a fixed, publicly-known hash function for integer keys (essentially identity for `int`). An attacker who knows this can construct `n` keys that all collide into the same bucket, degrading every operation from average O(1) to worst-case **O(n)** — turning an O(n) solution into O(n²) and causing a TLE, purely through adversarial input, with correct-looking code.

**The fix — randomize the hash per program run** with a splitmix64-seeded custom hash:

```cpp
#include <chrono>
struct custom_hash {
    // http://xorshift.di.unimi.it/splitmix64.c
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t FIXED_RANDOM =
            chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + FIXED_RANDOM);
    }
};
```

Plug it in as the (fourth) hash template parameter:

```cpp
unordered_map<long long, int, custom_hash> safe_map;
unordered_set<int, custom_hash> safe_set;
gp_hash_table<int, int, custom_hash> safe_gp; // also applies to PBDS hash tables
```

For a `pair<int,int>` key, combine both fields before hashing:

```cpp
struct pair_hash {
    size_t operator()(const pair<int,int>& p) const {
        static const uint64_t FIXED_RANDOM =
            chrono::steady_clock::now().time_since_epoch().count();
        uint64_t x = (uint64_t(p.first) << 32) ^ uint32_t(p.second);
        return custom_hash::splitmix64(x + FIXED_RANDOM); // reuse splitmix64 if custom_hash is visible
    }
};
```

> **Gotcha:** `chrono::steady_clock::now()` gives a per-run seed, not a per-map seed — this is intentional and sufficient, since the attacker doesn't know the seed at submission time.

### 31.2 `reserve()` and `max_load_factor()`

```cpp
unordered_map<int, int, custom_hash> m;
m.reserve(1 << 20);          // pre-size buckets, avoids repeated rehashing as it grows
m.max_load_factor(0.25);     // trade memory for fewer collisions/rehashes (default is 1.0)
```

Rehashing (which happens automatically as load factor grows) is O(n) each time it triggers; reserving upfront avoids paying that cost log(n) times over.

### 31.3 When `map` beats `unordered_map`

| Situation | Winner | Why |
|---|---|---|
| n is small (≲ 100-1000) | `map` | Constant factors dominate; `unordered_map`'s hashing overhead isn't amortized away |
| Need sorted iteration / range queries (`lower_bound`, `upper_bound`) | `map` | `unordered_map` has no order |
| Keys are adversarial and you forgot `custom_hash` | `map` | O(log n) worst case is guaranteed, no hack possible |
| Cache behavior matters, keys are scattered pointers/strings | `map` (surprisingly) | `unordered_map`'s bucket + chain layout can thrash cache as badly as a tree in practice, while offering no ordering benefit |
| Pure O(1)-average lookups on large n, trusted/random keys | `unordered_map` | Wins once n is large enough to amortize hashing cost |

### 31.4 Constant-factor rules of thumb

Rough budget: **~10⁸ simple operations per second** for a naive, unoptimized inner loop (array access + arithmetic); can reach ~10⁹ for trivial vectorized loops under `-O2`.

| n | Time limit ~1-2s → feasible complexity |
|---|---|
| n ≤ 10 | O(n!), O(2ⁿ · n) — brute force / permutations |
| n ≤ 20-22 | O(2ⁿ), O(2ⁿ · n) — bitmask DP |
| n ≤ 500 | O(n³) |
| n ≤ 2,000-5,000 | O(n²) |
| n ≤ 10⁵ | O(n log n), O(n √n) |
| n ≤ 10⁶ | O(n log n), O(n) with small constant |
| n ≤ 10⁷-10⁸ | O(n) with a *very* tight constant (simple loops, no allocation, no function-call overhead) |

> **Gotcha:** these numbers assume `-O2`/`-O3` and simple operations (int arithmetic, array indexing). Anything with divisions, `sqrt`, dynamic allocation, or virtual calls inside the innermost loop can be 5-20x slower — measure, don't just count big-O.

### 31.5 Cache-friendliness

```cpp
// BAD: vector<vector<int>> — each row is a separate heap allocation,
// rows can be scattered anywhere in memory -> cache misses on every row jump
vector<vector<int>> grid(n, vector<int>(m));

// GOOD: flat 1D array with manual indexing — one contiguous allocation
vector<int> grid(n * m);
auto at = [&](int r, int c) -> int& { return grid[r * m + c]; };
```

- Prefer `vector`/arrays over `list`, `set`, `map` in hot loops — trees and linked lists scatter nodes across the heap; every `->next` or tree-node dereference is a likely cache miss.
- Iterate arrays in the order they're laid out in memory (row-major for a flattened 2D grid) — sequential access lets the prefetcher hide latency.
- Struct-of-arrays over array-of-structs when only some fields are hot: if you only touch `.value` in a tight loop, storing `vector<int> values` separately beats `vector<Node>` where each `Node` also drags unused fields through cache.

### 31.6 Avoiding repeated allocation in loops

```cpp
// BAD: reallocates a new vector every iteration
for (int i = 0; i < q; ++i) {
    vector<int> tmp;          // heap alloc + dealloc, q times
    // ... fill tmp ...
}

// GOOD: allocate once, clear (keeps capacity) inside the loop
vector<int> tmp;
tmp.reserve(maxSize);
for (int i = 0; i < q; ++i) {
    tmp.clear();               // does NOT release capacity
    // ... fill tmp ...
}
```

`clear()` destroys elements but keeps the underlying buffer, so the next fill reuses already-allocated memory — no repeated `malloc`/`free` churn.

### 31.7 Pass by `const&`, `reserve` on vectors, `\n` over `endl`

```cpp
// Pass large objects by const reference, not by value (avoids a full copy per call)
long long sum(const vector<int>& v) {
    long long s = 0;
    for (int x : v) s += x;
    return s;
}

vector<int> v;
v.reserve(n); // pre-allocate capacity, avoids O(log n) reallocations as it grows via push_back

cout << "result\n";   // fast: just writes a newline character
cout << "result" << endl; // slow if repeated: endl also FLUSHES the stream every call
```

Flushing forces an OS-level I/O sync; in a loop printing thousands of lines, `endl` vs `\n` alone can be the difference between AC and TLE.

### 31.8 The `-O2` note

Codeforces (and most judges) compile your submission with `-O2` (check the problem's compiler settings, but this is the default for the standard GCC options). **Your local unoptimized build (plain `g++ file.cpp -o file`) can be 3-10x slower** than what actually runs on the judge. Two consequences:

1. Always benchmark locally with `-O2` to get a realistic sense of whether you'll pass.
2. Don't panic if a solution feels borderline slow un-optimized locally — re-test with `-O2` before concluding it needs a better algorithm.

```bash
g++ -std=c++17 -O2 -o sol sol.cpp
```

### 31.9 Pragmas — use sparingly, understand the risk

```cpp
#pragma GCC optimize("O2")
#pragma GCC optimize("Ofast")
#pragma GCC target("avx2")
```

- `#pragma GCC optimize(...)` forces aggressive optimization flags from *inside* the source file — useful when a judge doesn't let you pick compiler flags, or you want stronger flags than the judge's default (e.g., `Ofast` enables fast-math, which is **not IEEE-754 safe** — can silently change floating point results).
- `#pragma GCC target("avx2")` enables SIMD instruction sets not guaranteed present on the judge's actual CPU — if the judge's hardware doesn't support AVX2 (rare on modern CF judges, but a real risk on unfamiliar/embedded judges), this can cause an illegal-instruction **runtime error**, not a compile error.

> **Gotcha:** these pragmas are a "last resort squeeze," not a default habit. On Codeforces they're common and generally safe (judge hardware is known/stable); on unknown judges or interviews, leave them out — the risk of a silent correctness change (`Ofast`) or a crash (unsupported `target`) outweighs the speedup.

---

## 32. Debugging, UB and Compiling

### 32.1 Compiler flags and a ready-to-use local compile command

```bash
g++ -std=c++17 -O2 -Wall -Wextra -Wshadow -fsanitize=address,undefined -D_GLIBCXX_DEBUG -g -o sol sol.cpp
./sol < input.txt
```

| Flag | Catches / does |
|---|---|
| `-std=c++17` | Pins the language standard so behavior matches the judge |
| `-O2` | Matches judge optimization level (see §31.8) — but see gotcha below about combining with sanitizers |
| `-Wall -Wextra` | Broad warning set: unused variables, sign-compare mismatches, missing return, etc. |
| `-Wshadow` | Warns when an inner-scope variable shadows an outer one (classic bug: reusing loop variable `i` in a nested lambda/loop and silently shadowing the outer `i`) |
| `-fsanitize=address,undefined` | Instruments the binary to catch memory errors (ASan) and undefined behavior (UBSan) at runtime — see §32.2 |
| `-D_GLIBCXX_DEBUG` | Enables libstdc++'s debug mode: bounds-checked STL containers/iterators — see §32.3 |
| `-g` | Embeds debug symbols so `gdb` can show line numbers and variable names |

> **Gotcha:** `-O2` together with sanitizers/`_GLIBCXX_DEBUG` is unusual — sanitizers are normally run with `-O0`/`-O1` for clearer stack traces, and `_GLIBCXX_DEBUG` changes container ABI in ways that can conflict with an optimized/release-mode build in rare cases. In practice, keep **two separate local commands**: a "correctness" build (sanitizers + `_GLIBCXX_DEBUG`, no `-O2` or `-O1`) for finding bugs, and a "speed" build (`-O2` only, no sanitizers) for timing. Don't submit either — submit the plain, flag-free source; the judge applies its own flags.

### 32.2 AddressSanitizer / UndefinedBehaviorSanitizer — what each catches

**ASan (Address Sanitizer)** — memory errors:

```cpp
// out-of-bounds (heap)
int* a = new int[5];
a[5] = 10;              // ASan: heap-buffer-overflow
delete[] a;

// use-after-free
int* p = new int(5);
delete p;
cout << *p;              // ASan: heap-use-after-free

// stack out-of-bounds
void f() {
    int arr[3];
    arr[3] = 1;           // ASan: stack-buffer-overflow
}
```

**UBSan (Undefined Behavior Sanitizer)** — language-level UB:

```cpp
// signed integer overflow
int x = INT_MAX;
x = x + 1;                // UBSan: signed integer overflow

// null pointer dereference
int* p = nullptr;
cout << *p;                // UBSan: null pointer dereference (or SEGV, depending on platform)

// invalid shift
int y = 1;
int z = y << 32;           // UBSan: shift exponent 32 is too large for 32-bit type 'int'
```

Both attach to the running program and abort with a readable stack trace pointing at the exact line — vastly faster than guessing from a silent Wrong Answer.

### 32.3 `_GLIBCXX_DEBUG` — STL bounds/iterator checking

```cpp
#define _GLIBCXX_DEBUG
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> v = {1, 2, 3};
    cout << v[5];              // plain operator[]: UB, MAY silently return garbage without this flag
    // with _GLIBCXX_DEBUG: aborts immediately with
    // "Error: attempt to subscript container with out-of-bounds index 5, but container only holds 3 elements."
}
```

It also catches invalidated-iterator use (e.g., dereferencing an iterator after the vector it points into was resized) and comparing iterators from two different containers — bugs ASan/UBSan don't always catch cleanly since they're a *logical* STL contract violation, not a raw memory violation.

> **Gotcha:** `_GLIBCXX_DEBUG` makes STL operations significantly slower (often 5-10x) — never leave it defined in a submission or a timing benchmark.

### 32.4 `assert`

```cpp
#include <cassert>

int divide(int a, int b) {
    assert(b != 0);           // aborts with a message + file/line if b == 0, when NDEBUG is not defined
    return a / b;
}
```

`assert` is a cheap, zero-setup way to encode "this should never happen" invariants while developing; it becomes a no-op automatically when `NDEBUG` is defined (which `-O2`-style release builds often do), so it costs nothing on the judge.

### 32.5 Common UB checklist for DSA

| # | UB source | Example | Fix |
|---|---|---|---|
| 1 | Out-of-bounds `v[i]` | `v[v.size()]` | `[]` does **not** bounds-check; `.at(i)` throws `std::out_of_range` instead — use `.at()` while debugging |
| 2 | Signed integer overflow | `int a = 2e9; a += a;` | Use `long long` for any sum/product that might exceed ~2.1×10⁹ |
| 3 | Over-shifting | `1 << 32` for a 32-bit `int` | Shift amount must be `< sizeof(type)*8`; use `1LL << 32` for `long long` |
| 4 | Uninitialized variables | `int x; cout << x;` | Always initialize (`int x = 0;`); reading indeterminate values is UB, not just "garbage but harmless" |
| 5 | Dereferencing `end()` | `*v.end()` | `end()` is a past-the-end sentinel, never a valid element — check `it != v.end()` before `*it` |
| 6 | Invalidated iterators | Erasing from a `vector` while iterating with a stale iterator held from before the erase | Reassign the iterator from the `erase()` return value, or iterate with indices |
| 7 | Returning a reference to a local | `int& f() { int x = 5; return x; }` | The local is destroyed when `f` returns; the reference dangles. Return by value instead |
| 8 | Comparator violating strict weak ordering | `sort(v.begin(), v.end(), [](int a, int b){ return a <= b; });` | Must be strict (`<`, never `<=`); a non-strict comparator can crash `sort` or corrupt `set`/`map` silently |
| 9 | `__builtin_clz(0)` | `__builtin_clz(0)` | Undefined for input 0 — guard with `if (x) __builtin_clz(x);` or add 1 if you need a defined result at 0 |

```cpp
// #6 correctly:
for (auto it = v.begin(); it != v.end(); ) {
    if (shouldRemove(*it)) it = v.erase(it);  // erase() returns the next valid iterator
    else ++it;
}
```

### 32.6 Decoding compiler errors — plain-English translations

| Cryptic symptom | What's actually wrong | Fix |
|---|---|---|
| Page-long template error mentioning `std::_Rb_tree`, `operator<`, or similar when using `set<MyStruct>` / `map<MyStruct, ...>` / `priority_queue<MyStruct>` | `MyStruct` has no `operator<` (or the wrong signature) — these containers need strict weak ordering to sort/balance | Define `bool operator<(const MyStruct& o) const { ... }` or pass a custom comparator |
| Long error mentioning `std::hash<MyStruct>` when using `unordered_map<MyStruct, ...>` / `unordered_set<MyStruct>` | No `std::hash` specialization exists for your custom key type | Either specialize `std::hash<MyStruct>` or pass a custom hash functor as the template's 3rd/4th parameter |
| `error: no match for 'operator[]' ... (operand types are 'const std::map<...>' and 'int')` | You called `myMap[key]` on a `map` that's `const` (e.g., passed as `const map<...>&` to a function) — `operator[]` inserts a default-constructed value on missing key, which isn't allowed on a const object | Use `.at(key)` (throws if missing) or `.find(key)` (returns `end()` if missing) instead of `[]` |
| `error: invalid use of incomplete type` | You're using a type (calling a method, `sizeof`, dereferencing) that's only forward-declared, not fully defined at that point — common with self-referential structs (`struct Node { Node child; };` instead of `Node* child;`) | Include the full definition before use, or use a pointer/reference for self-referential members |
| Ambiguous call / unexpected behavior involving a variable named `count`, `size`, `data`, or `begin` | `using namespace std;` pulled `std::count`, `std::size`, etc. into scope, and your local variable of the same name shadows or collides with the algorithm | Rename the variable (`cnt`, `sz`, `arr`) or avoid `using namespace std;` in files that also `#include <algorithm>`/`<numeric>` |
| Baffling error only after `#include <cmath>` (often via `bits/stdc++.h`) when you have a variable named `y1` | `<cmath>` declares a global function `y1(double)` (Bessel function of the second kind, order 1) via `<math.h>`; naming a local `y1` can collide/shadow depending on compiler and cause obscure errors | Never name a variable `y0`, `y1`, `j0`, `j1`, or `tolower`/`toupper` as plain identifiers — rename to `y1_` or similar |

### 32.7 gdb quick commands

```
gdb ./sol                 # start gdb on the compiled binary
run < input.txt            # run with stdin redirected from a file
break solve                # set a breakpoint at function 'solve'
break file.cpp:42          # set a breakpoint at a specific line
next                        # step over (don't enter function calls)
step                         # step into function calls
continue                    # resume until next breakpoint / crash / exit
bt                            # backtrace — show the call stack at a crash or breakpoint
print x                     # print the value of variable x
print v[3]                 # print a specific vector element (after including STL pretty-printers, or just via .at())
watch x                    # break whenever x's value changes
list                          # show source around the current line
quit
```

### 32.8 Stress testing — brute force vs optimized, diffed on random input

Three files plus a driving bash loop:

```cpp
// gen.cpp — random test generator, takes a seed as argv[1]
#include <bits/stdc++.h>
using namespace std;
int main(int argc, char** argv) {
    srand(atoi(argv[1]));
    int n = rand() % 10 + 1;          // small n to make brute force feasible
    cout << 1 << "\n";                 // 1 test case
    cout << n << "\n";
    for (int i = 0; i < n; ++i) cout << (rand() % 20) << " \n"[i == n-1];
    return 0;
}
```

```cpp
// brute.cpp — obviously-correct O(n^2) or worse reference solution
#include <bits/stdc++.h>
using namespace std;
int main() {
    int t; cin >> t;
    while (t--) {
        int n; cin >> n;
        vector<int> a(n);
        for (auto &x : a) cin >> x;
        // ... naive/obviously-correct logic ...
        cout << /* brute answer */ "\n";
    }
}
```

```cpp
// sol.cpp — the fast solution under suspicion, compiled normally
```

```bash
#!/usr/bin/env bash
# stress.sh
set -e
g++ -std=c++17 -O2 -o gen gen.cpp
g++ -std=c++17 -O2 -o brute brute.cpp
g++ -std=c++17 -O2 -Wall -Wextra -fsanitize=address,undefined -o sol sol.cpp

for i in $(seq 1 1000); do
    ./gen "$i" > input.txt
    ./sol   < input.txt > out1.txt
    ./brute < input.txt > out2.txt
    if ! diff -q out1.txt out2.txt > /dev/null; then
        echo "MISMATCH on seed $i"
        echo "--- input ---";  cat input.txt
        echo "--- sol ---";    cat out1.txt
        echo "--- brute ---";  cat out2.txt
        break
    fi
done
echo "done"
```

```bash
chmod +x stress.sh
./stress.sh
```

The loop generates a fresh small random case each iteration (seed = loop counter, so failures are reproducible — rerun `./gen 37 > input.txt` to recreate the exact failing case), runs both solutions, and stops at the first seed where outputs diverge — printing the offending input immediately for manual inspection.

> **Gotcha:** keep `gen.cpp` producing **small** n (≤ 10-20) so `brute.cpp`'s worse complexity stays fast enough to run a thousand iterations in seconds; use a separate large-n generator (no brute-force comparison, just checking `sol` doesn't crash/TLE) for performance stress testing.

---

# PART VIII — QUICK REFERENCE AND RECIPES

## 33. The "How Do I...?" Recipe Index

### 33.1 Sorting

| Task | Code |
|---|---|
| Sort ascending | `sort(v.begin(), v.end());` |
| Sort descending | `sort(v.begin(), v.end(), greater<int>());` |
| Sort descending (alt) | `sort(v.rbegin(), v.rend());` |
| Sort by 2nd element of pairs | `sort(v.begin(), v.end(), [](auto&a, auto&b){ return a.second < b.second; });` |
| Sort pairs by 2nd desc, then 1st asc (tie-break) | `sort(v.begin(), v.end(), [](auto&a, auto&b){ if(a.second!=b.second) return a.second>b.second; return a.first<b.first; });` |
| Sort struct by multiple keys | `sort(v.begin(), v.end(), [](const P&a, const P&b){ if(a.x!=b.x) return a.x<b.x; if(a.y!=b.y) return a.y<b.y; return a.z<b.z; });` |
| Argsort (indices by value) | `vector<int> idx(n); iota(idx.begin(),idx.end(),0); sort(idx.begin(),idx.end(),[&](int i,int j){ return a[i]<a[j]; });` |
| Sort only a range | `sort(v.begin()+l, v.begin()+r+1);` |
| Sort and dedupe a vector | `sort(v.begin(),v.end()); v.erase(unique(v.begin(),v.end()), v.end());` |
| Reverse a vector | `reverse(v.begin(), v.end());` |
| Rotate left by k | `rotate(v.begin(), v.begin()+k, v.end());` |
| Rotate right by k | `rotate(v.begin(), v.end()-k, v.end());` |
| Check if sorted | `is_sorted(v.begin(), v.end());` |
| Partial sort (top k smallest) | `partial_sort(v.begin(), v.begin()+k, v.end());` |
| Stable sort (preserve equal order) | `stable_sort(v.begin(), v.end(), cmp);` |
| Sort a vector of vectors by inner size | `sort(v.begin(), v.end(), [](auto&a, auto&b){ return a.size() < b.size(); });` |
| Sort indices, then apply permutation to a vector | `vector<int> res(n); for (int i=0;i<n;i++) res[i] = a[idx[i]];` |
| Merge two already-sorted vectors | `vector<int> out; merge(a.begin(),a.end(), b.begin(),b.end(), back_inserter(out));` |
| Check if sorted descending | `is_sorted(v.begin(), v.end(), greater<int>());` |

### 33.2 Extremes & Kth Element

| Task | Code |
|---|---|
| Max element (value) | `*max_element(v.begin(), v.end());` |
| Min element (value) | `*min_element(v.begin(), v.end());` |
| Index of max element | `max_element(v.begin(), v.end()) - v.begin();` |
| Index of min element | `min_element(v.begin(), v.end()) - v.begin();` |
| Both min & max in one pass | `auto [mn, mx] = minmax_element(v.begin(), v.end());` |
| Kth largest via nth_element (partial reorder, O(n) avg) | `nth_element(v.begin(), v.begin()+k-1, v.end(), greater<int>()); int kth = v[k-1];` |
| Kth smallest via nth_element | `nth_element(v.begin(), v.begin()+k-1, v.end()); int kth = v[k-1];` |
| Kth largest via min-heap of size k | `priority_queue<int, vector<int>, greater<int>> pq; for (int x: v){ pq.push(x); if ((int)pq.size()>k) pq.pop(); } int kth = pq.top();` |
| Kth smallest via max-heap of size k | `priority_queue<int> pq; for (int x: v){ pq.push(x); if ((int)pq.size()>k) pq.pop(); } int kth = pq.top();` |
| Second largest distinct value | sort+dedupe then index `[n-2]`, or track two running maxima in one pass |
| Median of a vector (sorted) | `n%2 ? v[n/2] : (v[n/2-1]+v[n/2])/2.0;` |

### 33.3 Sum / Counting / Existence

| Task | Code |
|---|---|
| Sum without overflow | `long long sum = accumulate(v.begin(), v.end(), 0LL);` |
| Count occurrences of x | `count(v.begin(), v.end(), x);` |
| Count matching predicate | `count_if(v.begin(), v.end(), [](int x){ return x%2==0; });` |
| Exists in vector (unsorted) | `find(v.begin(), v.end(), x) != v.end();` |
| Exists in sorted vector | `binary_search(v.begin(), v.end(), x);` |
| Exists in set/map (never use `mp[k]` to check — it inserts) | `s.count(x);` / `mp.count(k);` |
| Build frequency map | `unordered_map<int,int> freq; for (int x: v) freq[x]++;` |
| Frequency of chars in string | `array<int,26> f{}; for (char c: s) f[c-'a']++;` |
| Remove an element from a vector by value (all copies) | `v.erase(remove(v.begin(), v.end(), x), v.end());` |
| Remove an element from a vector by index | `v.erase(v.begin() + idx);` |
| Insert an element at a position | `v.insert(v.begin() + idx, x);` |
| Dedupe preserving original order (unsorted) | `unordered_set<int> seen; vector<int> out; for (int x: v) if (seen.insert(x).second) out.push_back(x);` |
| Two vectors equal (same elements, ignore order) | sort copies of both, then `==` |

### 33.4 Map / Set Navigation

| Task | Code |
|---|---|
| Iterate a map | `for (auto& [k, val] : mp) { ... }` |
| Iterate keys only | `for (auto& [k, val] : mp) use(k);` |
| Map's max key | `mp.rbegin()->first;` |
| Map's min key | `mp.begin()->first;` |
| Predecessor (largest < x) in a set | `auto it = s.lower_bound(x); if (it == s.begin()) /* none */; else { --it; /* *it is predecessor */ }` |
| Successor (smallest > x) in a set | `auto it = s.upper_bound(x); if (it != s.end()) /* *it is successor */;` |
| Smallest >= x in a set | `auto it = s.lower_bound(x);` |
| Erase ONE occurrence from a multiset | `ms.erase(ms.find(x));  // ms.erase(x) removes ALL copies` |
| Erase one occurrence from multiset by count check | `if (auto it = ms.find(x); it != ms.end()) ms.erase(it);` |
| Get value or default from map | `auto it = mp.find(k); int val = (it != mp.end()) ? it->second : 0;` |
| Count occurrences of a key in a multiset/multimap | `ms.count(x);  // O(log n + count)` |
| Get range of equal keys in a multimap | `auto [first, last] = mm.equal_range(k); for (auto it = first; it != last; ++it) ...` |
| Erase while iterating a map/set safely | `for (auto it = mp.begin(); it != mp.end(); ) { if (shouldErase(*it)) it = mp.erase(it); else ++it; }` |
| Check if map is empty before `.begin()`/`.rbegin()` access | `if (!mp.empty()) { ... }` — calling on an empty map is UB |

### 33.5 Heaps / Priority Queue

| Task | Code |
|---|---|
| Declare a min-heap of int | `priority_queue<int, vector<int>, greater<int>> pq;` |
| Max-heap (default) | `priority_queue<int> pq;` |
| Custom comparator — struct/functor | `struct Cmp { bool operator()(const P&a, const P&b) const { return a.second > b.second; } }; priority_queue<P, vector<P>, Cmp> pq;` |
| Custom comparator — lambda | `auto cmp = [](const P&a, const P&b){ return a.second > b.second; }; priority_queue<P, vector<P>, decltype(cmp)> pq(cmp);` |
| Pop the top and use it | `auto top = pq.top(); pq.pop();` |
| Heapify existing vector in place | `make_heap(v.begin(), v.end());` |

### 33.6 2D / 3D Grids

| Task | Code |
|---|---|
| 2D vector init (n x m, value 0) | `vector<vector<int>> g(n, vector<int>(m, 0));` |
| 3D vector init | `vector<vector<vector<int>>> g(n, vector<vector<int>>(m, vector<int>(k, 0)));` |
| Flatten (r, c) to 1D index | `int id = r * m + c;` |
| Unflatten 1D index to (r, c) | `int r = id / m, c = id % m;` |
| 4-directional deltas (up/down/left/right) | `int dx[] = {0,0,1,-1}, dy[] = {1,-1,0,0};` |
| 8-directional deltas (incl. diagonals) | `int dx[] = {-1,-1,-1,0,0,1,1,1}, dy[] = {-1,0,1,-1,1,-1,0,1};` |
| Bounds check | `bool ok = (nr>=0 && nr<n && nc>=0 && nc<m);` |
| Loop all neighbors | `for (int d=0; d<4; d++){ int nr=r+dx[d], nc=c+dy[d]; if (ok) ... }` |
| Fill a 2D vector with a specific value | `vector<vector<int>> g(n, vector<int>(m, -1));` |
| Transpose a matrix | `vector<vector<int>> t(m, vector<int>(n)); for(int i=0;i<n;i++) for(int j=0;j<m;j++) t[j][i]=g[i][j];` |
| Copy/clone a 2D vector | `auto g2 = g;  // deep copy for vector<vector<T>>` |
| Reset a 2D vector to all zero without reallocating | `for (auto& row : g) fill(row.begin(), row.end(), 0);` |

### 33.7 Strings

| Task | Code |
|---|---|
| String to int | `int x = stoi(s);` |
| String to long long | `long long x = stoll(s);` |
| Int to string | `string s = to_string(x);` |
| Split string by delimiter | `stringstream ss(s); string tok; vector<string> out; while (getline(ss, tok, ',')) out.push_back(tok);` |
| Split by whitespace | `stringstream ss(s); string tok; while (ss >> tok) out.push_back(tok);` |
| Join vector<string> with delimiter | `string res; for (int i=0;i<(int)v.size();i++){ if(i) res += ","; res += v[i]; }` |
| Char to digit | `int d = c - '0';` |
| Digit to char | `char c = '0' + d;` |
| Char to lowercase letter index (0-25) | `int idx = c - 'a';` |
| Uppercase a string | `transform(s.begin(), s.end(), s.begin(), ::toupper);` |
| Lowercase a string | `transform(s.begin(), s.end(), s.begin(), ::tolower);` |
| Palindrome check | `string r = s; reverse(r.begin(), r.end()); bool pal = (s == r);` |
| Reverse a string | `reverse(s.begin(), s.end());` |
| Substring (start, length) | `string sub = s.substr(pos, len);` |
| Substring to end | `string sub = s.substr(pos);` |
| Find substring, check existence | `if (s.find(t) != string::npos) { ... }` |
| Find last occurrence | `s.rfind(t);` |
| Build a string efficiently (avoid repeated concat) | `string res; res.reserve(n); for (...) res += c;` |
| Remove all spaces | `s.erase(remove(s.begin(), s.end(), ' '), s.end());` |
| Compare strings case-insensitively | lowercase both, then `==` |
| Check if char is a letter/digit | `isalpha(c);` / `isdigit(c);` / `isalnum(c);` |
| Count vowels/specific chars | `count_if(s.begin(), s.end(), [](char c){ return string("aeiou").find(c)!=string::npos; });` |
| Anagram check | sort copies of both strings, compare equal; or compare 26-length frequency arrays |
| Convert `vector<char>` to `string` | `string s(v.begin(), v.end());` |
| Convert `string` to `vector<char>` | `vector<char> v(s.begin(), s.end());` |
| Pad a number with leading zeros | `ostringstream o; o << setw(width) << setfill('0') << x; string s = o.str();` |
| Repeat a string n times | `string rep; for (int i=0;i<n;i++) rep += s;`  (or `string(n, ch)` for a single char) |

### 33.8 Combinatorics / Enumeration

| Task | Code |
|---|---|
| All permutations of a vector | `sort(v.begin(), v.end()); do { /* use v */ } while (next_permutation(v.begin(), v.end()));` |
| All subsets via bitmask | `for (int mask=0; mask<(1<<n); mask++) { for (int i=0;i<n;i++) if (mask & (1<<i)) /* i is in subset */; }` |
| All subsets via recursion | `function<void(int)> go = [&](int i){ if (i==n) { /* use cur */ return; } cur.push_back(a[i]); go(i+1); cur.pop_back(); go(i+1); }; go(0);` |
| Next combination of k elements (indices) | use bitmask trick: `int t = mask \| (mask-1); mask = (t+1) \| (((~t & -~t) - 1) >> (__builtin_ctz(mask)+1));` |
| All permutations of size k out of n (nPk), recursive | `vector<bool> used(n,false); function<void(int)> go=[&](int cnt){ if(cnt==k){ /* use cur */ return; } for(int i=0;i<n;i++) if(!used[i]){ used[i]=true; cur.push_back(a[i]); go(cnt+1); cur.pop_back(); used[i]=false; } }; go(0);` |
| All k-combinations of n indices, recursive | `function<void(int,int)> go=[&](int start,int cnt){ if(cnt==k){ /* use cur */ return; } for(int i=start;i<n;i++){ cur.push_back(i); go(i+1,cnt+1); cur.pop_back(); } }; go(0,0);` |
| Number of set bits equal to k, enumerate those masks only | `for (int mask=0; mask<(1<<n); mask++) if (__builtin_popcount(mask)==k) { /* use mask */ }` |
| Generate all permutations recursively (when `next_permutation` isn't a fit, e.g. need early pruning) | `function<void()> go=[&](){ if(cur.size()==n){ /* use cur */ return; } for(int i=0;i<n;i++) if(!used[i]){ used[i]=true; cur.push_back(a[i]); go(); cur.pop_back(); used[i]=false; } }; go();` |

### 33.9 Binary Search Recipes

| Task | Code |
|---|---|
| First index with value >= x | `lower_bound(v.begin(), v.end(), x) - v.begin();` |
| First index with value > x | `upper_bound(v.begin(), v.end(), x) - v.begin();` |
| Last index with value <= x | `(upper_bound(v.begin(), v.end(), x) - v.begin()) - 1;` |
| Last index with value < x | `(lower_bound(v.begin(), v.end(), x) - v.begin()) - 1;` |
| Count elements in [l, r] (sorted vector) | `upper_bound(v.begin(),v.end(),r) - lower_bound(v.begin(),v.end(),l);` |
| Binary search on the answer (monotonic predicate) | `int lo=0, hi=MAXV; while (lo<hi){ int mid=lo+(hi-lo)/2; if (ok(mid)) hi=mid; else lo=mid+1; }` |
| Binary search on the answer, descending predicate (find largest x where ok(x)) | `int lo=0, hi=MAXV; while (lo<hi){ int mid=lo+(hi-lo+1)/2; if (ok(mid)) lo=mid; else hi=mid-1; }` |
| First and last position of x in one call | `auto [first, last] = equal_range(v.begin(), v.end(), x); // last - first == count of x` |
| Manual binary search template (no STL) | `int lo=0, hi=n-1, ans=-1; while(lo<=hi){ int mid=lo+(hi-lo)/2; if(v[mid]==x){ans=mid;break;} else if(v[mid]<x) lo=mid+1; else hi=mid-1; }` |

### 33.10 Math

| Task | Code |
|---|---|
| GCD | `std::gcd(a, b);` [C++17] `<numeric>`  (contest shorthand: `__gcd(a,b)` — GCC only, see §25.9) |
| LCM | `std::lcm(a, b);` [C++17]  (or `a / std::gcd(a,b) * b;` — divide first to avoid overflow) |
| Fast power (iterative binary exponentiation) | `long long fastpow(long long b, long long e){ long long r=1; while(e){ if(e&1) r*=b; b*=b; e>>=1; } return r; }` |
| Modular fast power | `long long modpow(long long b, long long e, long long m){ b%=m; long long r=1; while(e){ if(e&1) r=r*b%m; b=b*b%m; e>>=1; } return r; }` |
| Modular add | `long long s = ((a % mod) + (b % mod)) % mod;` |
| Modular multiply | `long long p = ((a % mod) * (b % mod)) % mod;` |
| Safe positive modulo (a can be negative) | `long long m = ((a % mod) + mod) % mod;` |
| Integer sqrt (careful with fp error) | `long long r = (long long)sqrtl((long double)x); while (r*r > x) r--; while ((r+1)*(r+1) <= x) r++;` |
| Power-of-two check | `bool p2 = x > 0 && (x & (x - 1)) == 0;` |
| Count set bits | `__builtin_popcount(x);` (use `__builtin_popcountll` for 64-bit) |
| Iterate all submasks of `mask` | `for (int sub = mask; sub; sub = (sub - 1) & mask) { /* use sub */ }` |
| Lowest set bit (value) | `x & (-x);` |
| Clear the lowest set bit | `x & (x - 1);` |
| Set bit i | `x \| (1 << i);` |
| Clear bit i | `x & ~(1 << i);` |
| Toggle bit i | `x ^ (1 << i);` |
| Check if bit i is set | `(x >> i) & 1;` |
| Index of lowest set bit | `__builtin_ctz(x);` (guard x != 0) |
| Index of highest set bit | `31 - __builtin_clz(x);` (guard x != 0, for 32-bit int) |
| Iterate submasks including 0 | append explicit `/* handle sub == 0 */` after loop since loop stops at 0 |
| Sieve of Eratosthenes up to N | `vector<bool> is_composite(N+1, false); for (int i=2;(long long)i*i<=N;i++) if (!is_composite[i]) for (int j=i*i;j<=N;j+=i) is_composite[j]=true;` |
| Factorial + modular inverse precompute (for nCr mod p) | `vector<long long> fact(N+1); fact[0]=1; for(int i=1;i<=N;i++) fact[i]=fact[i-1]*i%MOD; long long invN = modpow(fact[N], MOD-2, MOD); /* build invfact downward similarly */` |
| nCr mod p (with fact/invfact precomputed) | `long long nCr(int n,int r){ if(r<0\|\|r>n) return 0; return fact[n]*invfact[r]%MOD*invfact[n-r]%MOD; }` |
| Modular inverse via Fermat's little theorem (p prime) | `long long inv(long long a, long long p){ return modpow(a, p-2, p); }` |
| Check primality (trial division) | `bool isPrime(long long x){ if (x<2) return false; for (long long i=2;i*i<=x;i++) if (x%i==0) return false; return true; }` |
| Prefix sum array (1D) | `vector<long long> pre(n+1, 0); for (int i=0;i<n;i++) pre[i+1] = pre[i] + a[i]; // range sum [l,r] = pre[r+1]-pre[l]` |
| Difference array (range-add, point-query after) | `vector<long long> diff(n+1, 0); diff[l] += val; diff[r+1] -= val; /* then prefix-sum diff to get final array */` |
| Pascal's triangle for small nCr (no mod needed) | `vector<vector<long long>> C(n+1, vector<long long>(n+1,0)); for(int i=0;i<=n;i++){ C[i][0]=1; for(int j=1;j<=i;j++) C[i][j]=C[i-1][j-1]+C[i-1][j]; }` |
| Custom hash for `pair<int,int>` key in `unordered_map` (fixes mistake #5) | `struct PairHash { size_t operator()(const pair<int,int>& p) const { return hash<long long>()(((long long)p.first<<32) ^ (unsigned int)p.second); } }; unordered_map<pair<int,int>, int, PairHash> mp;` |

### 33.11 Classic Loop / Traversal Skeletons

| Task | Code |
|---|---|
| Two pointers, opposite ends (e.g. pair sum in sorted array) | `int l=0, r=n-1; while (l<r){ int s=a[l]+a[r]; if (s==target) {/*found*/ break;} else if (s<target) l++; else r--; }` |
| Two pointers, same direction (fast/slow) | `int slow=0; for (int fast=0; fast<n; fast++){ if (keep(a[fast])) a[slow++]=a[fast]; }` |
| Sliding window, variable size (shrink while invalid) | `int l=0; for (int r=0; r<n; r++){ /* add a[r] to window */ while (invalid()) { /* remove a[l] */ l++; } /* window [l,r] is valid, use it */ }` |
| Recursive DFS on adjacency list | `vector<bool> visited(n,false); function<void(int)> dfs=[&](int u){ visited[u]=true; for (int v: adj[u]) if (!visited[v]) dfs(v); }; dfs(0);` |
| Iterative DFS with explicit stack (avoids stack overflow on deep graphs) | `vector<bool> visited(n,false); stack<int> st; st.push(0); while(!st.empty()){ int u=st.top(); st.pop(); if(visited[u]) continue; visited[u]=true; for(int v: adj[u]) if(!visited[v]) st.push(v); }` |
| Backtracking template | `function<void(int)> bt=[&](int i){ if (i==n) { /* record solution */ return; } for (int choice : choices) { apply(choice); bt(i+1); undo(choice); } }; bt(0);` |

### 33.12 Misc Utilities

| Task | Code |
|---|---|
| Swap two values | `swap(a, b);` |
| Clamp value to range [C++17] | `int c = clamp(x, lo, hi);` |
| Random integer in [lo, hi] | `mt19937 rng(chrono::steady_clock::now().time_since_epoch().count()); uniform_int_distribution<int> dist(lo, hi); int r = dist(rng);` |
| Measure runtime of a block | `auto t0 = chrono::high_resolution_clock::now(); /* work */ auto t1 = chrono::high_resolution_clock::now(); double ms = chrono::duration<double, milli>(t1-t0).count();` |
| Read tokens until EOF | `int x; while (cin >> x) { /* use x */ }` |
| Read a full line until EOF | `string line; while (getline(cin, line)) { /* use line */ }` |
| Read an n x m grid | `vector<vector<int>> g(n, vector<int>(m)); for (int i=0;i<n;i++) for (int j=0;j<m;j++) cin >> g[i][j];` |
| Read a grid of chars (n rows) | `vector<string> g(n); for (int i=0;i<n;i++) cin >> g[i];` |
| Fixed-precision decimal print | `cout << fixed << setprecision(2) << x << "\n";` |
| Print a vector space-separated | `for (int i=0;i<(int)v.size();i++) cout << v[i] << " \n"[i+1==(int)v.size()];` |
| Print a vector via iterator | `copy(v.begin(), v.end(), ostream_iterator<int>(cout, " "));` |
| Fast I/O boilerplate | `ios_base::sync_with_stdio(false); cin.tie(nullptr);` |
| Compare two doubles for "equality" | `fabs(a - b) < 1e-9;` |
| Ternary min/max without `<algorithm>` | `int m = (a < b) ? a : b;` |
| Convert between signed/unsigned safely | `int x = static_cast<int>(u);` (cast explicitly, never rely on implicit conversion in comparisons) |
| Suppress unused-variable warning | `(void)x;` |

### 33.13 Data-Structure Skeletons

| Task | Code |
|---|---|
| DSU (Union-Find) — declare + init | `vector<int> parent(n), rnk(n, 0); iota(parent.begin(), parent.end(), 0);` |
| DSU — find with path compression | `int find(int x){ return parent[x]==x ? x : parent[x]=find(parent[x]); }` |
| DSU — union by rank | `void unite(int a,int b){ a=find(a); b=find(b); if(a==b) return; if(rnk[a]<rnk[b]) swap(a,b); parent[b]=a; if(rnk[a]==rnk[b]) rnk[a]++; }` |
| Adjacency list — declare + add undirected edge | `vector<vector<int>> adj(n); adj[u].push_back(v); adj[v].push_back(u);` |
| Adjacency list — weighted edge | `vector<vector<pair<int,int>>> adj(n); adj[u].push_back({v, w});` |
| 2D prefix sum (submatrix sum queries) | `vector<vector<long long>> pre(n+1, vector<long long>(m+1,0)); for(int i=0;i<n;i++) for(int j=0;j<m;j++) pre[i+1][j+1] = g[i][j] + pre[i][j+1] + pre[i+1][j] - pre[i][j]; // sum(r1,c1,r2,c2) = pre[r2+1][c2+1]-pre[r1][c2+1]-pre[r2+1][c1]+pre[r1][c1]` |
| Sliding window maximum (monotonic deque) | `deque<int> dq; for(int i=0;i<n;i++){ while(!dq.empty() && a[dq.back()]<=a[i]) dq.pop_back(); dq.push_back(i); if(dq.front()<=i-k) dq.pop_front(); if(i>=k-1) result.push_back(a[dq.front()]); }` |
| Fenwick tree (BIT) — point update + prefix query | `vector<long long> bit(n+1,0); void upd(int i,long long v){ for(++i; i<=n; i+=i&-i) bit[i]+=v; } long long qry(int i){ long long s=0; for(++i; i>0; i-=i&-i) s+=bit[i]; return s; }` |

### 33.14 Numeric Limits & Type Ranges (`<climits>` / `<cfloat>`)

| Type | Max | Min | Notes |
|---|---|---|---|
| `int` (32-bit) | `INT_MAX` ≈ 2.147×10^9 | `INT_MIN` ≈ -2.147×10^9 | overflow around ~2.1×10^9 — the #1 WA cause |
| `unsigned int` | `UINT_MAX` ≈ 4.295×10^9 | 0 | wraps silently, no overflow signal |
| `long long` (64-bit) | `LLONG_MAX` ≈ 9.223×10^18 | `LLONG_MIN` | safe for almost all CP sums/products up to ~9.2×10^18 |
| `unsigned long long` | `ULLONG_MAX` ≈ 1.8×10^19 | 0 | |
| `double` | ~1.8×10^308 | precision ~15-17 significant digits | fine for most floating math; watch precision loss on large sums |
| `long double` | platform-dependent, often 80-bit extended | precision ~18-19 digits on x86 | slightly safer for tight geometry/precision problems |
| Generic max-value idiom | `numeric_limits<int>::max();` / `numeric_limits<long long>::max();` | | preferred over macros in generic/templated code |
| Generic min-value idiom | `numeric_limits<int>::min();` | | |
| Epsilon for double comparisons | `numeric_limits<double>::epsilon();` (~2.2×10^-16) | | in practice use a coarser custom epsilon like `1e-9` |

### 33.15 Container Conversions

| Task | Code |
|---|---|
| `vector<int>` → `set<int>` (sorted, deduped) | `set<int> s(v.begin(), v.end());` |
| `set<int>` → `vector<int>` | `vector<int> v(s.begin(), s.end());` |
| `vector<pair<K,V>>` → `map<K,V>` | `map<K,V> mp(v.begin(), v.end());` |
| `map<K,V>` → `vector<pair<K,V>>` | `vector<pair<K,V>> v(mp.begin(), mp.end());` |
| `array<T,N>` → `vector<T>` | `vector<T> v(arr.begin(), arr.end());` |
| C-style array → `vector<T>` | `int arr[] = {1,2,3}; vector<int> v(arr, arr + size(arr));` |
| `vector<int>` → `string` (of digit chars, small values 0-9) | `string s; for (int x: v) s += ('0' + x);` |
| `initializer_list` → `vector` | `vector<int> v = {1, 2, 3};` (implicit) |

### 33.16 Container Choice — 5-Second Recap

(Full rationale lives in Part III §16; this is the fast-lookup version.)

| Need | Reach for |
|---|---|
| Random access + append at back | `vector` |
| Append/remove at both ends | `deque` |
| Frequent insert/erase in the middle, no random access needed | `list` |
| Sorted unique keys, need ordered iteration or predecessor/successor | `set` / `map` |
| Sorted, duplicates allowed | `multiset` / `multimap` |
| Fastest average lookup, don't care about order | `unordered_set` / `unordered_map` |
| Always need the current min/max, not full order | `priority_queue` |
| Strict LIFO / FIFO | `stack` / `queue` |
| Fixed-size dense boolean flags with fast bitwise ops | `bitset<N>` |

### 33.17 Headers Cheat Sheet (when `<bits/stdc++.h>` isn't available)

| Need | Header |
|---|---|
| `vector`, `array` | `<vector>`, `<array>` |
| `string` | `<string>` |
| `sort`, `find`, `reverse`, `next_permutation`, etc. | `<algorithm>` |
| `accumulate`, `iota`, `gcd`, `lcm` | `<numeric>` |
| `map`, `set`, `multiset`, `multimap` | `<map>`, `<set>` |
| `unordered_map`, `unordered_set` | `<unordered_map>`, `<unordered_set>` |
| `queue`, `priority_queue`, `stack` | `<queue>`, `<stack>` |
| `deque` | `<deque>` |
| `list`, `forward_list` | `<list>`, `<forward_list>` |
| `bitset` | `<bitset>` |
| `pair`, `tuple`, `swap` | `<utility>`, `<tuple>` |
| `INT_MAX`, `LLONG_MAX`, etc. | `<climits>` |
| `numeric_limits<T>::max()` | `<limits>` |
| `sqrt`, `pow`, `fabs`, `ceil`, `floor` | `<cmath>` |
| `mt19937`, `uniform_int_distribution` | `<random>` |
| `high_resolution_clock` (timing) | `<chrono>` |
| `stringstream`, `istringstream`, `ostringstream` | `<sstream>` |
| `setprecision`, `setw`, `setfill`, `fixed` | `<iomanip>` |
| `function<...>` (for recursive lambdas) | `<functional>` |

### 33.18 I/O Templates

| Task | Code |
|---|---|
| Multiple test cases loop | `int t; cin >> t; while (t--) { /* solve one case */ }` |
| Read n then a vector of n ints | `int n; cin >> n; vector<int> a(n); for (auto& x : a) cin >> x;` |
| Read n then m edges (u, v) | `int n, m; cin >> n >> m; vector<vector<int>> adj(n); for (int i=0;i<m;i++){ int u,v; cin >> u >> v; adj[u].push_back(v); adj[v].push_back(u); }` |
| Read a vector of pairs | `vector<pair<int,int>> v(n); for (auto& [a,b] : v) cin >> a >> b;` |
| Print "YES"/"NO" per test case | `cout << (ok ? "YES" : "NO") << "\n";` |
| Read an unknown number of ints from one line | `string line; getline(cin, line); stringstream ss(line); int x; vector<int> v; while (ss >> x) v.push_back(x);` |
| Simulate decrease-key with a priority_queue (lazy deletion) | `priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq; unordered_map<int,int> best; pq.push({dist, node}); /* on pop, skip if dist > best[node] (stale entry) */` |
| Custom `operator<` on a struct so `sort`/`set`/`priority_queue` work with no comparator | `struct P { int x, y; bool operator<(const P& o) const { return x!=o.x ? x<o.x : y<o.y; } };` |

---

## 34. Master Complexity Table

### 34.1 Containers

| Container | Insert | Erase | Lookup | Access | Iterate | Notes / when it degrades |
|---|---|---|---|---|---|---|
| `vector` | back O(1) amortized; middle O(n) | back O(1); middle O(n) | O(n) (`find`) | O(1) random | O(n) | Reallocation on growth invalidates iterators/pointers; `reserve()` avoids it |
| `deque` | front/back O(1); middle O(n) | front/back O(1); middle O(n) | O(n) | O(1) random | O(n) | Not contiguous memory; slightly higher constant than vector |
| `list` | O(1) given iterator | O(1) given iterator | O(n) | no random access | O(n) | Splice O(1); heavy pointer/cache overhead; use only when mid-list insert/erase dominates |
| `string` | back O(1) amortized; middle O(n) | O(n) | O(n) (`find`) | O(1) | O(n) | SSO (~15-22 chars) avoids heap alloc for short strings |
| `array<T,N>` | n/a (fixed size) | n/a | O(n) | O(1) | O(n) | Size fixed at compile time; no allocation |
| `set` / `map` (RB-tree) | O(log n) | O(log n) | O(log n) | n/a | O(n), sorted order | Iterators stable across insert/erase (except erased element itself) |
| `multiset` / `multimap` | O(log n) | O(log n) per element / O(log n + k) for `erase(key)` (k = count) | O(log n) via `find` | n/a | O(n), sorted | `erase(key)` removes ALL copies — use `erase(find(key))` for one |
| `unordered_set`/`unordered_map` | avg O(1), worst O(n) | avg O(1), worst O(n) | avg O(1), worst O(n) | n/a | O(n), unordered | Degrades to O(n) with many hash collisions (adversarial input against default hash); no ordering guarantee |
| `priority_queue` | O(log n) | (pop) O(log n) | top O(1) only | n/a | not iterable in order | Backed by vector + heap algorithms; no arbitrary erase/decrease-key |
| `stack` (adapter) | O(1) | O(1) | top O(1) only | n/a | not iterable | Default underlying container `deque` |
| `queue` (adapter) | O(1) | O(1) | front/back O(1) only | n/a | not iterable | Default underlying container `deque` |
| `bitset<N>` | — | — | O(1) test/set/reset | O(1) `[i]` | O(N/64) per whole-set op | Fixed size at compile time; extremely fast bitwise ops (`&`,`\|`,`^`,`~`,shifts) |
| `unordered_multiset`/`unordered_multimap` | avg O(1), worst O(n) | avg O(1) per element, worst O(n) | avg O(1), worst O(n) | n/a | O(n), unordered | Same collision caveats as `unordered_map`; rarely needed — prefer `unordered_map<key,int>` counting when possible |
| `forward_list` | O(1) given iterator (insert-after only) | O(1) given iterator | O(n) | no random access | O(n), forward only | Singly-linked, lower memory overhead than `list`; rarely used in CP |

### 34.1b Extra Operations Worth Memorizing

| Container | Operation | Complexity | Notes |
|---|---|---|---|
| `vector` | `push_back` / `emplace_back` | O(1) amortized | `emplace_back` constructs in place, avoids a copy/move |
| `vector` | `resize(n)` | O(n) | value-initializes new elements |
| `vector` | `reserve(n)` | O(n) (one-time) | pre-allocates capacity, avoids repeated reallocation — always call before a known-size loop of `push_back` |
| `vector` | `front()` / `back()` | O(1) | UB if empty — check `!v.empty()` first |
| `deque` | `push_front` / `push_back` | O(1) amortized | main reason to pick deque over vector |
| `list` | `splice` (move nodes between lists) | O(1) | no copying of elements, just pointer relinking |
| `string` | `+=` (append) | O(1) amortized | like vector push_back |
| `set`/`map` | `count(k)` | O(log n) | returns 0 or 1 (no duplicates) |
| `multiset`/`multimap` | `count(k)` | O(log n + m) where m = matches | can be slow if many duplicates of k exist |
| `unordered_map` | `reserve(n)` / `max_load_factor` | O(n) / O(1) | pre-sizing buckets avoids rehash storms, helps avoid worst-case degradation |
| `priority_queue` | `emplace` | O(log n) | constructs in place |
| all associative containers | `emplace` vs `insert` | same complexity | `emplace` can skip a temporary object construction |

### 34.2 `<algorithm>` / `<numeric>` Complexities

| Algorithm | Complexity | Notes |
|---|---|---|
| `sort` | O(n log n) | introsort (quicksort + heapsort fallback + insertion sort for small n); not stable |
| `stable_sort` | O(n log n) | stable; uses extra O(n) memory (merge sort) if available |
| `nth_element` | O(n) average | partial reorder only around the nth position; not fully sorted |
| `partial_sort` | O(n log k) | sorts only the first k elements fully |
| `lower_bound` / `upper_bound` | O(log n) on random-access iterators; O(n) on `list`/non-random-access | On `set`/`map` use the member function `s.lower_bound(x)` — O(log n) |
| `binary_search` | O(log n) | requires sorted range |
| `next_permutation` / `prev_permutation` | O(n) worst case per call, O(n!) total for all permutations | requires starting from sorted order to enumerate all |
| `accumulate` | O(n) | watch the init value's type (int vs long long) |
| `unique` | O(n) | only removes *consecutive* duplicates — sort first for full dedupe |
| `remove` / `remove_if` | O(n) | does NOT shrink the container — pair with `erase` |
| `reverse` | O(n) | |
| `rotate` | O(n) | |
| `merge` | O(n + m) | merges two sorted ranges |
| `set_union` / `set_intersection` / `set_difference` | O(n + m) | on sorted ranges |
| `count` / `count_if` | O(n) | |
| `find` / `find_if` | O(n) | linear scan |
| `max_element` / `min_element` | O(n) | |
| `accumulate` with custom op | O(n) | e.g. product via `multiplies<>()` |
| `iota` | O(n) | fills range with increasing sequence |
| `__gcd` | O(log(min(a,b))) | Euclidean algorithm |
| `partial_sum` | O(n) | writes running prefix sums to output range |
| `adjacent_difference` | O(n) | writes `a[i]-a[i-1]` to output range |
| `is_permutation` | O(n log n) or O(n^2) depending on overload | checks if one range is a permutation of another |
| `fill` / `fill_n` | O(n) | assigns a value across a range |
| `swap` (containers) | O(1) | swaps internal pointers/state, not element-by-element |
| `distance` | O(1) random-access iterators; O(n) otherwise (e.g. `list`) | be careful using it on non-random-access iterators in a hot loop |
| `binary_search` on `set`/`map` via free function | O(n) — wrong tool | use the member `.find()`/`.lower_bound()` instead, O(log n) |

### 34.3 Iterator / Reference / Pointer Invalidation Rules

The #1 source of mystery crashes (see §36 mistake #13). Know exactly when a stored iterator/reference/pointer goes stale.

| Container | Operation | Invalidates |
|---|---|---|
| `vector` | `push_back` / `insert` beyond capacity | ALL iterators, pointers, references (reallocation) |
| `vector` | `push_back` within capacity | only the end iterator; existing element refs/pointers stay valid |
| `vector` | `erase(it)` | iterators/refs/pointers at or after the erased position |
| `vector` | `resize` (shrinking) | iterators/refs to removed elements |
| `deque` | `push_back` / `push_front` | iterators (but NOT references/pointers to existing elements) |
| `deque` | `insert` / `erase` (middle) | all iterators, references, pointers |
| `list` | any insert | nothing — all existing iterators/refs/pointers stay valid |
| `list` | `erase(it)` | only the iterator to the erased element itself |
| `set` / `map` | `insert` | nothing — existing iterators stay valid |
| `set` / `map` | `erase(it)` | only the iterator to the erased element itself |
| `unordered_set` / `unordered_map` | `insert` causing rehash | ALL iterators (but references/pointers to elements stay valid) |
| `unordered_set` / `unordered_map` | `erase(it)` | only the iterator to the erased element itself |
| `string` | `push_back` / `append` beyond capacity | ALL iterators, pointers, references (like vector) |

Rule of thumb: **contiguous containers (`vector`, `string`, `deque` for refs) are the dangerous ones** — re-fetch iterators after any operation that might reallocate, or switch to storing indices instead of iterators/pointers when in doubt.

---

## 35. n → Required Complexity Cheat Sheet

Rule of thumb: judges allow roughly **10^8 simple operations/second** (up to ~10^9 for trivial loop bodies), with typical time limits of **1-2 seconds**. Read the constraint on `n` first — it tells you the algorithm before you've even read the problem statement.

| Judge / context | Typical time limit | Notes |
|---|---|---|
| Codeforces | 1-2 s (some 3-4 s for heavier problems) | C++ usually gets the base limit; interpreted languages often get 2-3x |
| LeetCode | ~1-10 s depending on problem, hidden exact budget | Generally forgiving; O(n^2) with n~1e4 is usually fine |
| ICPC / onsite judges | 1-5 s | Often stricter on memory too — watch array sizes |
| Interview ("is this efficient enough") | no hard clock, but state the complexity | Say the Big-O out loud and justify it against the given constraints |

Budget math: `operations ≈ complexity(n) evaluated at max n`. If that number clears ~10^8-10^9, the approach is too slow for a 1-2s limit — drop to a cheaper technique from the table below before coding.

| n (max) | Required complexity | Technique it implies | Pattern reference |
|---|---|---|---|
| n ≤ 10 | O(n!) or O(2^n · n) | Brute force permutations / full backtracking | 15-backtracking |
| n ≤ 20 | O(2^n) | Bitmask enumeration, bitmask DP (subsets as state) | 24-dp-bitmask, 15-backtracking |
| n ≤ 40 | O(2^(n/2)) | Meet-in-the-middle: split into two halves, enumerate each, combine with sorting/binary search | 15-backtracking, 05-binary-search |
| n ≤ 100 | O(n^4) or O(n^3) | DP over intervals, Floyd-Warshall, brute-force triple loops | 23-dp-interval, 18-dp-2d-grid, 14-graphs-shortest-path |
| n ≤ 500 | O(n^3) | Floyd-Warshall, cubic DP, matrix multiplication | 14-graphs-shortest-path, 23-dp-interval |
| n ≤ 5000 | O(n^2) | 2D DP tables, O(n^2) LIS/LCS, brute-force pair scanning | 18-dp-2d-grid, 20-dp-lcs-family, 21-dp-lis-family |
| n ≤ 1e5 | O(n log n) | Sorting, binary search, heaps, segment tree / Fenwick tree, DSU with log factor | 05-binary-search, 10-heap-and-priority-queue, 27-segment-tree, 28-fenwick-tree, 13-graphs-union-find |
| n ≤ 1e6 | O(n) or O(n log n) with small constant | Prefix sums, two pointers, sliding window, linear-time DP, single BFS/DFS pass | 02-two-pointers, 03-sliding-window, 06-prefix-sums-and-difference-arrays, 17-dp-1d, 12-graphs-bfs-dfs |
| n ≤ 1e9 (value, not count) | O(log n), O(sqrt n), or O(1) math | Binary search on the answer, fast exponentiation, number theory / closed-form math | 05-binary-search, 35-math-and-number-theory |

Common constants worth having memorized:

| Constant | Value | Use |
|---|---|---|
| `MOD` (common) | `1000000007` (1e9+7) | default modulus for counting problems |
| `MOD` (NTT-friendly) | `998244353` | used when the problem needs number-theoretic transform / specific primitive roots |
| `INF` (int-safe sentinel) | `1e9` or `INT_MAX/2` | large sentinel that won't overflow when added to itself |
| `INF` (long long-safe sentinel) | `1e18` or `LLONG_MAX/2` | same idea at 64-bit scale |
| `EPS` | `1e-9` (sometimes `1e-6` for looser precision problems) | floating-point equality threshold |

Extra guardrails:
- Multiple test cases (`t` up to 1e4-1e5) multiply the bound — check the *sum* of `n` across test cases, not per-case `n`, before picking complexity.
- `n, m ≤ 1e3` with a grid usually means O(n·m) or O(n·m·log) is intended (BFS/DFS per cell, or DP over the grid) — see 09-trees-bfs / 12-graphs-bfs-dfs / 18-dp-2d-grid.
- If two different constraints appear (e.g., n ≤ 2000 but q ≤ 1e5 queries), the intended solution usually precomputes O(n log n) or O(n^2) once, then answers each query in O(log n) or O(1) — segment tree / Fenwick tree / sparse table territory (27, 28).

---

## 36. The Top 30 C++ Mistakes That Cost You Problems

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | `int` overflow (e.g. `n*n` with n ~ 1e5) | WA / silent wrong answer | Use `long long` for anything that can exceed ~2·10^9; multiply as `1LL * a * b` |
| 2 | `(lo + hi) / 2` overflows for large indices | WA / crash on large ranges | `lo + (hi - lo) / 2` |
| 3 | `accumulate(v.begin(), v.end(), 0)` with `int` init on a sum that overflows | WA / silent wrong answer | `accumulate(v.begin(), v.end(), 0LL)` |
| 4 | `if (mp[k] > 0)` to check existence | Silent wrong answer / bloated map, wrong size() | Use `mp.count(k)` or `mp.find(k) != mp.end()` |
| 5 | `unordered_map<pair<int,int>, T>` — no default hash for pair | Compile error | Provide a custom hash functor, or use `map` instead |
| 6 | `unordered_map`/`unordered_set` with adversarial input (Codeforces hacks) | TLE | Use a custom hash (splitmix64-based) or seed with `chrono` time, or just use `map` |
| 7 | Comparator using `<=` instead of `<` (not strict weak ordering) | RE (crash / UB) or infinite loop in `sort` | Comparator must return false for equal elements — use `<` not `<=` |
| 8 | `priority_queue` comparator logic inverted (built min-heap thinking it's `<`) | Silent wrong answer (pops max instead of min or vice versa) | Remember `greater<T>` = min-heap; a custom `operator()` returning `a > b` reverses ordering |
| 9 | `multiset.erase(val)` removes ALL copies of `val` | Silent wrong answer (too many elements removed) | `ms.erase(ms.find(val))` to remove exactly one |
| 10 | `1 << 31` on `int` (shifting into/past sign bit) | UB / silent wrong answer | Use `1LL << 31` or `1U << 31`, and `long long` for the result |
| 11 | `__builtin_clz(0)` / `__builtin_ctz(0)` | UB (undefined result) | Guard with `if (x != 0)` before calling |
| 12 | `vector<bool>` is a bitset-like proxy, not a real container of bool | Compile errors with `&`, references, `auto&`; subtle bugs | Use `vector<char>` or `bitset<N>` when you need real bool semantics/references |
| 13 | Keeping a reference/pointer/iterator into a `vector` across a `push_back` | Crash / corrupted data (dangling reference after reallocation) | Re-fetch the reference after modification, or use indices instead of iterators |
| 14 | Mixing `cin >> x` and `getline(cin, line)` without consuming the trailing newline | Silent wrong answer (getline reads empty line) | Call `cin.ignore()` after the last `>>` before a `getline` |
| 15 | Using `endl` in a tight loop | TLE (flushes stream every call) | Use `"\n"` instead; flush only when necessary |
| 16 | `%` on negative numbers gives negative or implementation-varying result | Silent wrong answer | `((a % m) + m) % m` for a guaranteed non-negative result |
| 17 | `(int)sqrt(x)` off-by-one due to floating point error | Silent wrong answer near perfect squares | Compute with `sqrtl`, then verify/adjust: `while(r*r>x) r--; while((r+1)*(r+1)<=x) r++;` |
| 18 | `pow(x, y)` for integers — returns `double`, loses precision for large values | Silent wrong answer | Write an integer fast-power function instead of `pow` |
| 19 | Passing a large `vector`/`string` by value into a function | TLE (copies every call, esp. in recursion) | Pass by `const&` (or `&` if mutating) |
| 20 | `v.size() - 1` when `v` is empty (`size()` returns `size_t`, unsigned) | Crash / huge loop (wraps to SIZE_MAX) | Check `if (!v.empty())` first, or cast to a signed type before subtracting |
| 21 | Assuming `v[i]` throws on out-of-bounds | Silent memory corruption / crash later, not at the bug site | Use `v.at(i)` while debugging to get a real exception at the fault |
| 22 | Calling `unique()` without sorting first | Silent wrong answer (only removes consecutive dupes) | `sort(v.begin(), v.end())` before `unique` |
| 23 | Calling `remove()`/`remove_if()` and expecting the container to shrink | Silent wrong answer (stale trailing elements remain, size unchanged) | Always pair with `erase`: `v.erase(remove(...), v.end())` |
| 24 | Deep/unbounded recursion (e.g. DFS on a 1e6-node chain) | Crash (stack overflow), often no clear error message | Convert to iterative DFS with an explicit stack, or increase stack size if the judge allows |
| 25 | `lower_bound`/`upper_bound` directly on a `set`/`map` iterator via the free function | Silent O(n) instead of O(log n) → possible TLE | Use the member function `s.lower_bound(x)`, not `std::lower_bound(s.begin(), s.end(), x)` |
| 26 | `static` local variables persisting across multiple test cases | Silent wrong answer from test case 2 onward | Avoid `static` for per-test-case state, or explicitly reset it each test case |
| 27 | Uninitialized local variables (esp. in arrays/structs) | Silent wrong answer, non-reproducible / "works on my machine" | Always zero/initialize (`= {}`, `= 0`) explicitly; don't rely on it being zero |
| 28 | Returning a reference or pointer to a local (stack) variable | Crash / garbage values (UB) | Return by value, or use a `static`/heap-allocated/caller-owned object |
| 29 | Forgetting to reset global arrays/adjacency lists/visited flags between test cases | Silent wrong answer from test case 2 onward | Clear/reset all globals at the start (or end) of each test case loop |
| 30 | `abs()` on the wrong type (int overload truncates a `long long` argument) | Silent wrong answer for large values | Use `llabs()` / `std::abs` with correct overload, or cast explicitly to `long long` first |

### 36.1 Honorable Mentions (a few more that still bite)

| Mistake | Symptom | Fix |
|---|---|---|
| Comparing `float`/`double` with `==` | Silent wrong answer near boundary cases | Compare with an epsilon: `fabs(a-b) < 1e-9` |
| Recursion without a memo table in what should be DP | TLE (exponential blowup) | Add memoization (`vector`/`unordered_map` cache) or convert to bottom-up |
| Declaring large arrays/vectors inside a per-test-case function | TLE (repeated allocation) or stack overflow (large local array) | Declare once outside the loop and clear/resize, or use global/static storage sized to the max bound |
| Forgetting `break`/`return` after finding the answer in a search loop | Wrong answer (keeps overwriting with a later, worse candidate) | Add explicit `break` or early `return` once satisfied |
| Assuming `map`/`set` `.erase(it)` returns the same iterator | Compile error or wrong iterator used afterward | `.erase(it)` returns the NEXT valid iterator — reassign: `it = mp.erase(it);` |
| Integer division where floating-point was intended | Silent wrong answer (truncated result) | Cast one operand to `double` first: `(double)a / b` |
| Comparing `size_t` (unsigned) directly against a negative `int` | Silent wrong answer (int is converted to a huge unsigned value) | Cast explicitly to a common signed type before comparing |

---

## 37. 60-Second Warm-Up Drill

Read this top to bottom before a contest or interview. Each line demonstrates one thing you're about to need.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios_base::sync_with_stdio(false); cin.tie(nullptr);   // fast I/O boilerplate

    // 1. vector init + fill
    vector<int> v = {5, 3, 8, 1, 9, 2};                    // brace-init a vector

    // 2. sort with a lambda comparator (descending)
    sort(v.begin(), v.end(), [](int a, int b){ return a > b; });

    // 3. binary search building blocks (need ascending order first)
    sort(v.begin(), v.end());
    int idx = lower_bound(v.begin(), v.end(), 5) - v.begin(); // first index with value >= 5

    // 4. map + structured bindings iteration
    map<string, int> mp = {{"a", 1}, {"b", 2}};
    for (auto& [key, val] : mp) { /* key, val available directly */ }

    // 5. set + member lower_bound (O(log n), NOT std::lower_bound)
    set<int> s = {10, 20, 30, 40};
    auto it = s.lower_bound(25);                            // -> points to 30

    // 6. min-heap priority_queue
    priority_queue<int, vector<int>, greater<int>> pq;
    for (int x : v) pq.push(x);
    int smallest = pq.top();                                 // O(1) peek

    // 7. string manipulation: build, reverse, substring, find
    string str = "hello world";
    reverse(str.begin(), str.end());
    string sub = str.substr(0, 5);
    bool found = str.find("world") != string::npos;          // npos check pattern

    // 8. bitmask loop over all subsets of {0,...,n-1}
    int n = 4;
    for (int mask = 0; mask < (1 << n); mask++) {
        for (int i = 0; i < n; i++) {
            if (mask & (1 << i)) { /* i is in this subset */ }
        }
    }

    // 9. binary search on the answer (monotonic predicate ok(x))
    auto ok = [](int x){ return x * x >= 50; };               // example predicate
    int lo = 0, hi = 100;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;                         // overflow-safe midpoint
        if (ok(mid)) hi = mid; else lo = mid + 1;
    }

    // 10. grid BFS skeleton: direction arrays + queue + visited
    int rows = 5, cols = 5;
    vector<vector<int>> grid(rows, vector<int>(cols, 0));     // 2D vector init
    vector<vector<bool>> visited(rows, vector<bool>(cols, false));
    int dx[] = {0, 0, 1, -1}, dy[] = {1, -1, 0, 0};           // 4-directional deltas
    queue<pair<int,int>> bfsq;
    bfsq.push({0, 0});
    visited[0][0] = true;
    while (!bfsq.empty()) {
        auto [r, c] = bfsq.front(); bfsq.pop();                // structured binding on pop
        for (int d = 0; d < 4; d++) {
            int nr = r + dx[d], nc = c + dy[d];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && !visited[nr][nc]) {
                visited[nr][nc] = true;
                bfsq.push({nr, nc});
            }
        }
    }

    return 0;
}
```

### Self-check — if you can't write these from memory, re-read the listed section:

| # | Can you write it cold? | Re-read |
|---|---|---|
| 1 | Sort a vector of structs by 2+ keys with a lambda | §33.1, §17 (Algorithms & iterators) |
| 2 | Declare and use a min-heap priority_queue | §33.5, §14 (Container adapters: stack/queue/priority_queue) |
| 3 | `lower_bound`/`upper_bound` on both a vector and a `set` (and know why they differ) | §33.9, §34.2, §18 (Binary search family), §13 (set/multiset) |
| 4 | Build a frequency map and iterate it with structured bindings | §33.3-33.4, §12 (map/unordered_map) |
| 5 | Enumerate all subsets of n items via bitmask | §33.8, §22-24 (Bit manipulation) |
| 6 | Binary search on the answer with an overflow-safe midpoint | §33.9, §18 (Binary search family), §25 (overflow-safe midpoint); DSA Pattern 05 |
| 7 | Write a 4-directional grid BFS from scratch with visited tracking | §37 drill, §14 (queue), §33 (direction arrays); DSA Patterns 09/12 |
| 8 | Explain why `multiset.erase(val)` is dangerous and the safe alternative | §33.4, §36 mistake #9, §13 (Set/multiset) |
| 9 | Name the complexity of every operation on `vector`, `set`, `unordered_map`, `priority_queue` without looking | §34.1 |
| 10 | Read `n ≤ 1e5` (or `1e9`, or `20`) and immediately name the intended complexity class | §35 |

---

*C++ Complete Reference — companion to DSA Patterns 01–38*
*When you forget something and it is not in here, add it. This document should only grow.*
