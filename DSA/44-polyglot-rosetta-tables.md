# 44 — POLYGLOT ROSETTA TABLES
## C++ → Java → Python → JavaScript, One Construct Per Row
### Keep this open while practising the 30 problems (docs 45 and 46)

---

## HOW TO USE THIS DOC

Your C++ instinct fires. You need the equivalent in the language you are interviewing in. Find the row, read across. That is the whole workflow.

- Every row is one construct in all four languages. Read left to right.
- `> **Trap:**` callouts mark the places where porting C++ habits directly produces a wrong answer or a crash.
- When a construct needs more than a line, the code block sits directly under its table.
- This doc is the translation layer. For depth on any one language, go to docs 39 to 42 and their §33 recipe index.

Baselines: C++17, Java 17, Python 3.10+, Node 18 / ES2022.

## CONTENTS

**Part A — Language Core and Containers**
1. Boilerplate and Function Signatures
2. Variables, Types and Declarations
3. Operators and Arithmetic
4. Control Flow
5. Dynamic Arrays / Lists
6. Strings
7. Hash Map and Hash Set
8. Ordered Map / Ordered Set

**Part B — Algorithms, Idioms and Traps**
9. Stack, Queue, Deque
10. Heap / Priority Queue
11. Sorting and Comparators
12. Binary Search
13. Classes, Nodes and Custom Ordering
14. Recursion, Memoization and DP
15. Bit Manipulation
16. Numbers, Overflow and Math

---

# PART A — LANGUAGE CORE AND CONTAINERS

## 1. Boilerplate and Function Signatures

### LeetCode solution shell

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Class wrapper | `class Solution { public: ... };` | `class Solution { ... }` | `class Solution:` | not needed — top-level function |
| Returns `int` | `int solve(vector<int>& nums) {}` | `public int solve(int[] nums) {}` | `def solve(self, nums: List[int]) -> int:` | `var solve = function(nums) {};` |
| Returns `int[]` / array | `vector<int> solve(vector<int>& nums) {}` | `public int[] solve(int[] nums) {}` | `def solve(self, nums: List[int]) -> List[int]:` | `var solve = function(nums) {};` |
| Returns `List<List<int>>` | `vector<vector<int>> solve(vector<int>& nums) {}` | `public List<List<Integer>> solve(int[] nums) {}` | `def solve(self, nums: List[int]) -> List[List[int]]:` | `var solve = function(nums) {};` |
| Takes a tree node | `int solve(TreeNode* root) {}` | `public int solve(TreeNode root) {}` | `def solve(self, root: Optional[TreeNode]) -> int:` | `var solve = function(root) {};` |
| Takes a linked list node | `ListNode* solve(ListNode* head) {}` | `public ListNode solve(ListNode head) {}` | `def solve(self, head: Optional[ListNode]) -> Optional[ListNode]:` | `var solve = function(head) {};` |

Full class shells (return `int[]`, e.g. Two Sum):

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        // ...
        return {};
    }
};
```

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        // ...
        return new int[]{};
    }
}
```

```python
class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        # ...
        return []
```

```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
var twoSum = function(nums, target) {
    // ...
    return [];
};
```

### Helper method / function inside the solution

| Language | How to add a private helper |
|---|---|
| C++ | extra `private:` method in the same class, or a free function above the class |
| Java | extra `private` method in the same class |
| Python | nested `def` inside the method, or another `def _helper(self, ...)` method on the class |
| JavaScript | nested inner function, or a nother `function` declared before/after `var solve = ...` |

```cpp
class Solution {
public:
    int solve(vector<int>& nums) { return helper(nums, 0); }
private:
    int helper(vector<int>& nums, int i) { /* ... */ return 0; }
};
```

```java
class Solution {
    public int solve(int[] nums) { return helper(nums, 0); }
    private int helper(int[] nums, int i) { /* ... */ return 0; }
}
```

```python
class Solution:
    def solve(self, nums: List[int]) -> int:
        def helper(i: int) -> int:
            # closes over nums, no need to pass it
            return 0
        return helper(0)
```

```js
var solve = function(nums) {
    function helper(i) {
        return 0;
    }
    return helper(0);
};
```

### Codeforces / plain stdin-stdout shell

```cpp
#include <bits/stdc++.h>
using namespace std;
int main() {
    int n;
    cin >> n;
    vector<int> a(n);
    for (int& x : a) cin >> x;
    cout << n << "\n";
    return 0;
}
```

```java
import java.util.*;
import java.io.*;
public class Main {
    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StreamTokenizer in = new StreamTokenizer(br);
        in.nextToken(); int n = (int) in.nval;
        int[] a = new int[n];
        for (int i = 0; i < n; i++) { in.nextToken(); a[i] = (int) in.nval; }
        System.out.println(n);
    }
}
```

```python
import sys
def main():
    data = sys.stdin.read().split()
    idx = 0
    n = int(data[idx]); idx += 1
    a = [int(data[idx + i]) for i in range(n)]; idx += n
    print(n)
main()
```

```js
const lines = require("fs").readFileSync("/dev/stdin", "utf8").split("\n");
let ptr = 0;
const n = parseInt(lines[ptr++], 10);
const a = lines[ptr++].split(" ").map(Number);
console.log(n);
```

> **Trap:** `cin >>` / `Scanner` are far too slow for large Codeforces inputs. C++ pairs `cin`/`cout` with reading whole tokens via `>>` (fine once `sync_with_stdio(false); cin.tie(nullptr);` is set). Java's `Scanner` is notoriously slow — always use `BufferedReader` + `StreamTokenizer` or a custom fast reader. Python's `input()` in a loop is slow for many lines — read all of `sys.stdin` at once and split. JS reading line-by-line with `readline` per call is slow — read the whole file once.

## 2. Variables, Types and Declarations

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Declare typed variable | `int x = 5;` | `int x = 5;` | `x = 5` | `let x = 5;` |
| Declare constant | `const int x = 5;` | `final int x = 5;` | `X = 5  # convention only, not enforced` | `const x = 5;` |
| 32-bit integer | `int x;` (4 bytes, ~2.1e9 max) | `int x;` (4 bytes, ~2.1e9 max) | no fixed width — all `int` are arbitrary precision | no true int — `Number` is float64 |
| 64-bit integer | `long long x;` (8 bytes, ~9.2e18 max) | `long x;` (8 bytes, ~9.2e18 max) | same `int`, grows automatically | `BigInt`: `x = 5n` (exact, arbitrary precision) |
| Unsigned integer | `unsigned int` / `uint32_t` / `uint64_t` | none — no unsigned types at all | none needed — never overflows | none — use `>>> 0` (32-bit unsigned trick) or `BigInt` |
| Type inference | `auto x = compute();` | `var x = compute();` (Java 10+, still statically typed) | no annotation ever required (dynamic) | `let x = compute();` (dynamic) |
| Cast `int` → `double` | `double d = (double)x;` | `double d = (double) x;` | `d = float(x)` | `d = x` (already float64) |
| Cast `double` → `int` (truncate) | `int i = (int)d;` | `int i = (int) d;` | `i = int(d)` | `i = Math.trunc(d)` |
| Null / absence value | `nullptr` | `null` | `None` | `undefined` (unset) / `null` (explicit empty) |
| Check for null | `if (ptr == nullptr)` | `if (obj == null)` | `if x is None:` | `if (x == null)` (catches both `null` and `undefined`) |
| String → int | `stoi(s)` | `Integer.parseInt(s)` | `int(s)` | `parseInt(s, 10)` |
| String → long | `stoll(s)` | `Long.parseLong(s)` | `int(s)` | `BigInt(s)` |
| Int → string | `to_string(x)` | `String.valueOf(x)` | `str(x)` | `String(x)` or `` `${x}` `` |
| Char → digit value | `c - '0'` | `c - '0'` | `ord(c) - ord('0')` | `c.charCodeAt(0) - 48` |
| Print / debug output | `cout << x << "\n";` | `System.out.println(x);` | `print(x)` | `console.log(x);` |

> **Trap:** C++ `int` silently wraps/UB on overflow (`INT_MAX + 1` is undefined behavior in practice wraps to `INT_MIN`) — the classic "forgot to use `long long`" bug. Java `int`/`long` wrap silently too (no exception). Python integers never overflow — arbitrary precision — so a C++ overflow bug simply cannot happen when porting to Python (but this also hides bugs where the C++ solution *depended* on wraparound). JavaScript `Number` is a float64: integers are exact only up to `2**53 - 1` (`Number.MAX_SAFE_INTEGER`); beyond that, silently lose precision — switch to `BigInt` for exact big-integer math (note `BigInt` cannot mix with `Number` in one expression without explicit conversion).

## 3. Operators and Arithmetic

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Integer division (positive operands) | `7 / 2` → `3` | `7 / 2` → `3` | `7 // 2` → `3` | `Math.trunc(7 / 2)` → `3` |
| Power | `pow(2, 10)` (returns `double`) | `Math.pow(2, 10)` (returns `double`) | `2 ** 10` → `1024` (exact int) | `2 ** 10` → `1024` |
| Increment / decrement | `++x; x++; --x;` | `++x; x++; --x;` | no `++`/`--` — use `x += 1` | `++x; x++; --x;` |
| Swap two variables | `swap(a, b);` | `int t = a; a = b; b = t;` | `a, b = b, a` | `[a, b] = [b, a];` |
| Ternary | `cond ? a : b` | `cond ? a : b` | `a if cond else b` | `cond ? a : b` |
| Min of two | `min(a, b)` | `Math.min(a, b)` | `min(a, b)` | `Math.min(a, b)` |
| Max of two | `max(a, b)` | `Math.max(a, b)` | `max(a, b)` | `Math.max(a, b)` |
| Min of a collection | `*min_element(v.begin(), v.end())` | `Collections.min(list)` | `min(lst)` | `Math.min(...arr)` |
| Max of a collection | `*max_element(v.begin(), v.end())` | `Collections.max(list)` | `max(lst)` | `Math.max(...arr)` |
| Absolute value | `abs(x)` | `Math.abs(x)` | `abs(x)` | `Math.abs(x)` |
| Compound assignment | `x += 5;` | `x += 5;` | `x += 5` | `x += 5;` |
| Chained comparison | `a < b && b < c` | `a < b && b < c` | `a < b < c` — really chains! | `a < b && b < c` |
| Equality | `a == b` (value for primitives/structs) | `a == b` (reference for objects — use `.equals()` for value equality) | `a == b` (value, calls `__eq__`) | `a === b` (value for primitives, reference for objects) |

> **Trap:** `Math.max(...arr)` / `Math.min(...arr)` in JS spreads the whole array onto the call stack — throws `RangeError: Maximum call stack size exceeded` on very large arrays (tens of thousands+). Use `arr.reduce((a,b) => Math.max(a,b))` for large inputs.
> **Trap:** Java `==` on boxed types (`Integer`, `String`) compares *reference identity*, not value — `Integer.valueOf(200) == Integer.valueOf(200)` is `false`, but small integers in `[-128, 127]` are cached so `Integer.valueOf(100) == Integer.valueOf(100)` is `true` — an easy trap. Always use `.equals()` for boxed value comparison. `int == int` (unboxed primitives) is fine and compares value.

`pow()`/`Math.pow()`/`**` all return floats and lose precision on large integer exponents — for exact integer powers (modular exponentiation, etc.) write binary exponentiation directly, same shape in every language:

```cpp
long long binpow(long long b, long long e, long long mod) {
    long long r = 1; b %= mod;
    while (e > 0) {
        if (e & 1) r = r * b % mod;
        b = b * b % mod;
        e >>= 1;
    }
    return r;
}
```

```java
static long binpow(long b, long e, long mod) {
    long r = 1; b %= mod;
    while (e > 0) {
        if ((e & 1) == 1) r = r * b % mod;
        b = b * b % mod;
        e >>= 1;
    }
    return r;
}
```

```python
def binpow(b: int, e: int, mod: int) -> int:
    return pow(b, e, mod)   # built-in 3-arg pow does fast modexp natively — no loop needed
```

```js
function binpow(b, e, mod) {           // use BigInt — Number loses precision under repeated squaring
    b = BigInt(b); e = BigInt(e); mod = BigInt(mod);
    let r = 1n;
    b %= mod;
    while (e > 0n) {
        if (e & 1n) r = (r * b) % mod;
        b = (b * b) % mod;
        e >>= 1n;
    }
    return r;
}
```

> **Trap:** Python's built-in `pow(b, e, mod)` (three-argument form) already does fast modular exponentiation natively in C — never hand-roll the loop in Python. There is no equivalent 3-argument overload in C++/Java/JS.

### Division and modulo sign — the #1 porting bug

| Expression | C++17 | Java | Python 3 | JavaScript |
|---|---|---|---|---|
| `-7 / 2` | `-3` (truncates toward 0) | `-3` (truncates toward 0) | `-7 // 2` → `-4` (floors toward −∞) | `Math.trunc(-7 / 2)` → `-3` |
| `-7 % 3` | `-1` (sign follows dividend) | `-1` (sign follows dividend) | `-7 % 3` → `2` (sign follows divisor) | `-7 % 3` → `-1` (sign follows dividend) |
| `7 % -3` | `1` | `1` | `7 % -3` → `-2` | `1` |

> **Trap:** Python's `//` and `%` are **floor**-based — the sign of the result always matches the divisor. C++, Java, and JavaScript are all **truncating** — the sign of the result always matches the dividend. Porting a C++ rolling-hash, modular-index, or negative-offset computation to Python (or the reverse) without adjusting for this is the single most common silent bug when translating between these languages.
> **Fix — truncating division/mod in Python** (to match C++/Java/JS): `int(a / b)` for division (careful with float precision on huge `a`, prefer `math.trunc(a / b)` or `-(-a // b) if (a < 0) != (b < 0) else a // b`); for mod use `a - b * int(a / b)`, or `math.fmod(a, b)` for floats.
> **Fix — floor mod in C++/Java/JS** (to match Python, e.g. for always-non-negative wraparound indices): `((a % n) + n) % n`.

## 4. Control Flow

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| For over range `[0, n)` | `for (int i = 0; i < n; i++)` | `for (int i = 0; i < n; i++)` | `for i in range(n):` | `for (let i = 0; i < n; i++)` |
| For over collection (values) | `for (int x : v)` | `for (int x : list)` | `for x in lst:` | `for (const x of arr)` |
| For with index | `for (int i = 0; i < (int)v.size(); i++)` | `for (int i = 0; i < list.size(); i++)` | `for i, x in enumerate(lst):` | `for (const [i, x] of arr.entries())` |
| For over map entries | `for (auto& [k, v] : mp)` | `for (var e : map.entrySet())` → `e.getKey()`/`e.getValue()` | `for k, v in d.items():` | `for (const [k, v] of map.entries())` |
| While | `while (cond) { ... }` | `while (cond) { ... }` | `while cond:` | `while (cond) { ... }` |
| Do-while | `do { ... } while (cond);` | `do { ... } while (cond);` | no do-while: `while True: ...; if not cond: break` | `do { ... } while (cond);` |
| Break / continue | `break; continue;` | `break; continue;` | `break` / `continue` | `break; continue;` |
| If / else if / else | `if () {} else if () {} else {}` | `if () {} else if () {} else {}` | `if:` / `elif:` / `else:` | `if () {} else if () {} else {}` |
| Switch / pattern match | `switch (x) { case 1: ...; break; }` | `switch (x) { case 1 -> ...; }` (arrow form, Java 14+) | `match x:` / `case 1:` (3.10+) | `switch (x) { case 1: ...; break; }` |

### Labeled break out of nested loops

C++ has no labels — use a flag or `goto` (rare but idiomatic in CP for deep nesting):

```cpp
bool found = false;
for (int i = 0; i < n && !found; i++) {
    for (int j = 0; j < m; j++) {
        if (grid[i][j] == target) { found = true; break; }
    }
}
// goto form:
for (int i = 0; i < n; i++)
    for (int j = 0; j < m; j++)
        if (grid[i][j] == target) goto done;
done:;
```

Java has real labels:

```java
outer:
for (int i = 0; i < n; i++) {
    for (int j = 0; j < m; j++) {
        if (grid[i][j] == target) break outer;
    }
}
```

Python has no labels — wrap in a function and `return`, or use the for/else trick:

```python
def find():
    for i in range(n):
        for j in range(m):
            if grid[i][j] == target:
                return i, j
    return None
```

JavaScript has real labels:

```js
outer:
for (let i = 0; i < n; i++) {
    for (let j = 0; j < m; j++) {
        if (grid[i][j] === target) break outer;
    }
}
```

> **Trap:** Python has no labeled `break`/`continue` at all. The cleanest port of a labeled-break loop is a small function with `return` (works for `break`; for `continue outer` semantics you instead structure the outer loop body as a function and `return` early per outer-iteration). The `for...else` construct (`else` runs only if the loop never `break`s) is a distinct, lesser-known tool — do not confuse it with a labeled continue.

### Switch / pattern match, in full

```cpp
switch (x) {
    case 1:
    case 2:
        doA();
        break;
    case 3:
        doB();
        break;
    default:
        doC();
}
```

```java
switch (x) {
    case 1, 2 -> doA();          // Java 14+ arrow form, no fallthrough, no break needed
    case 3 -> doB();
    default -> doC();
}
```

```python
match x:
    case 1 | 2:
        doA()
    case 3:
        doB()
    case _:
        doC()
```

```js
switch (x) {
    case 1:
    case 2:
        doA();
        break;
    case 3:
        doB();
        break;
    default:
        doC();
}
```

> **Trap:** C++ and JavaScript `switch` **fall through** by default — a missing `break` runs into the next `case` (a classic bug). Java's classic colon-form `switch` also falls through; the newer arrow form (`case 1 ->`) does **not** fall through and needs no `break`. Python `match` never falls through, and `case 1 | 2:` is how you OR multiple values in one branch (equivalent to stacking `case 1: case 2:` in C++/Java/JS).

## 5. Dynamic Arrays / Lists

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create empty | `vector<int> a;` | `List<Integer> a = new ArrayList<>();` | `a = []` | `const a = [];` |
| Create size `n`, filled with 0 | `vector<int> a(n, 0);` | `int[] a = new int[n];` | `a = [0] * n` | `const a = new Array(n).fill(0);` |
| Create from literal | `vector<int> a = {1, 2, 3};` | `List<Integer> a = new ArrayList<>(List.of(1, 2, 3));` | `a = [1, 2, 3]` | `const a = [1, 2, 3];` |
| Append to end | `a.push_back(x);` | `a.add(x);` | `a.append(x)` | `a.push(x);` |
| Pop from end | `a.pop_back();` | `a.remove(a.size() - 1);` | `a.pop()` | `a.pop();` |
| Get element | `a[i]` | `a.get(i)` | `a[i]` | `a[i]` |
| Set element | `a[i] = x;` | `a.set(i, x);` | `a[i] = x` | `a[i] = x;` |
| Length | `a.size()` | `a.size()` | `len(a)` | `a.length` |
| Insert at index | `a.insert(a.begin() + i, x);` | `a.add(i, x);` | `a.insert(i, x)` | `a.splice(i, 0, x);` |
| Erase at index | `a.erase(a.begin() + i);` | `a.remove(i);` | `del a[i]` | `a.splice(i, 1);` |
| Contains value | `find(a.begin(), a.end(), x) != a.end()` | `a.contains(x)` | `x in a` | `a.includes(x)` |
| Index of value (not-found) | manual `find` → compare to `a.end()` | `a.indexOf(x)` → `-1` | `a.index(x)` → raises `ValueError` | `a.indexOf(x)` → `-1` |
| Clear | `a.clear();` | `a.clear();` | `a.clear()` | `a.length = 0;` |
| Copy / clone | `vector<int> b = a;` (deep, value semantics) | `List<Integer> b = new ArrayList<>(a);` | `b = a.copy()` or `a[:]` | `const b = [...a];` or `a.slice()` |
| Concatenate | `a.insert(a.end(), b.begin(), b.end());` | `a.addAll(b);` | `c = a + b` | `const c = a.concat(b);` or `[...a, ...b]` |
| Reverse | `reverse(a.begin(), a.end());` (in place) | `Collections.reverse(a);` (in place) | `a.reverse()` (in place) or `a[::-1]` (copy) | `a.reverse();` (in place) |
| Slice / sublist | `vector<int>(a.begin()+i, a.begin()+j)` (copy) | `a.subList(i, j)` (LIVE VIEW, not a copy!) | `a[i:j]` (copy) | `a.slice(i, j)` (copy) |
| Iterate values | `for (int x : a)` | `for (int x : a)` | `for x in a:` | `for (const x of a)` |
| Sort ascending | `sort(a.begin(), a.end());` | `Collections.sort(a);` or `a.sort(null);` | `a.sort()` | `a.sort((x, y) => x - y);` |
| Sort descending | `sort(a.rbegin(), a.rend());` | `a.sort(Collections.reverseOrder());` | `a.sort(reverse=True)` | `a.sort((x, y) => y - x);` |
| Sort with custom comparator | `sort(a.begin(), a.end(), [](int x, int y){ return abs(x) < abs(y); });` | `a.sort((x, y) -> Integer.compare(Math.abs(x), Math.abs(y)));` | `a.sort(key=lambda x: abs(x))` | `a.sort((x, y) => Math.abs(x) - Math.abs(y));` |

> **Trap:** `subList` in Java is a *view* backed by the original list — mutating it mutates the parent (and structural changes to the parent invalidate it). Python's `a[i:j]` and JS's `a.slice(i, j)` always return an independent copy. If you need an independent copy in Java, wrap it: `new ArrayList<>(a.subList(i, j))`.
> **Trap — copy vs. reference when passing to a function:** C++ passes containers **by value** by default (`vector<int> f(vector<int> v)` copies the whole vector) — a real performance bug if you forget `&` (`vector<int>& v` or `const vector<int>& v`). Java arrays/`List`, Python `list`, and JavaScript `Array` are **always reference types** when passed to a function — the callee gets a reference to the *same* underlying array, so in-place mutations (`a[i] = x`, `.sort()`, `.reverse()`, `.append()`) are visible to the caller. There is no by-value array semantics at all in Java/Python/JS; only C++ needs to opt in (with `&`) or out (by copying) explicitly.

### 2D arrays

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create `n × m`, zero-filled | `vector<vector<int>> g(n, vector<int>(m, 0));` | `int[][] g = new int[n][m];` | `g = [[0] * m for _ in range(n)]` | `const g = Array.from({length: n}, () => Array(m).fill(0));` |

> **Trap — shared-row bug (hard trap):** `vector<vector<int>> g(n, vector<int>(m, 0))` is safe — C++ deep-copies the inner vector for every row. Java `new int[n][m]` is safe — the JVM allocates `n` independent row arrays. But Python **`g = [[0] * m] * n` is BROKEN**: it creates one inner list and `n` references to that *same* list, so `g[0][0] = 5` mutates every row. Always write `g = [[0] * m for _ in range(n)]`. JavaScript has the identical bug: **`Array(n).fill(Array(m))` is BROKEN** — `Array(m)` is evaluated once and all `n` slots reference the same array. Always write `Array.from({length: n}, () => Array(m).fill(0))`, since `Array.from`'s mapper function runs fresh for every index.

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create `n × m × k` 3D, zero-filled | `vector<vector<vector<int>>> g(n, vector<vector<int>>(m, vector<int>(k, 0)));` | `int[][][] g = new int[n][m][k];` | `g = [[[0]*k for _ in range(m)] for _ in range(n)]` | `const g = Array.from({length:n}, () => Array.from({length:m}, () => Array(k).fill(0)));` |
| Jagged array of empty lists (adjacency list) | `vector<vector<int>> adj(n);` | see code block below | `adj = [[] for _ in range(n)]` | `const adj = Array.from({length: n}, () => []);` |

```java
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
```

> **Trap:** Java has no one-liner for an array of `ArrayList`s without an unchecked-cast warning (`new ArrayList[n]`) — always loop-and-add as above. Never write `new ArrayList<>(Collections.nCopies(n, new ArrayList<>()))` — it has the exact same shared-reference bug as Python's `[[]] * n` (all `n` slots point at one list).
> **Trap:** Python `adj = [[]] * n` has the identical shared-list bug as the 2D zero-fill trap above — always use the list-comprehension form `[[] for _ in range(n)]`.

## 6. Strings

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create | `string s = "abc";` | `String s = "abc";` | `s = "abc"` | `const s = "abc";` |
| Length | `s.size()` | `s.length()` | `len(s)` | `s.length` |
| Index a character | `s[i]` (a `char`) | `s.charAt(i)` (a `char`) | `s[i]` (a length-1 `str` — no `char` type) | `s[i]` (a length-1 string) |
| Char arithmetic (`'a'`→0) | `c - 'a'` | `c - 'a'` | `ord(c) - ord('a')` | `c.charCodeAt(0) - 97` |
| Int → char/letter | `char(c + 'a')` | `(char) ('a' + c)` | `chr(ord('a') + c)` | `String.fromCharCode(97 + c)` |
| Substring | `s.substr(i, len)` | `s.substring(i, j)` (end exclusive) | `s[i:j]` | `s.slice(i, j)` |
| Find substring (not found) | `s.find(t)` → `string::npos` | `s.indexOf(t)` → `-1` | `s.find(t)` → `-1` | `s.indexOf(t)` → `-1` |
| Contains substring | `s.find(t) != string::npos` | `s.contains(t)` (Java 15+) | `t in s` | `s.includes(t)` |
| Split | manual loop / `stringstream` | `s.split(",")` | `s.split(",")` | `s.split(",")` |
| Join | manual loop / `ostringstream` | `String.join(",", list)` | `",".join(lst)` | `arr.join(",")` |
| Trim whitespace | manual / Boost `trim` | `s.strip()` (Java 11+) | `s.strip()` | `s.trim()` |
| Replace | `regex_replace(s, re, rep)` | `s.replace(old, new)` (literal) | `s.replace(old, new)` (literal) | `s.replaceAll(old, new)` (regex!) or `s.replace(old, new)` (first match only) |
| To upper / lower | per-char `toupper(c)`, loop the string | `s.toUpperCase()` / `s.toLowerCase()` | `s.upper()` / `s.lower()` | `s.toUpperCase()` / `s.toLowerCase()` |
| Reverse | `reverse(s.begin(), s.end());` (in place) | `new StringBuilder(s).reverse().toString();` | `s[::-1]` | `s.split("").reverse().join("")` |
| Compare lexicographically | `s < t` | `s.compareTo(t)` (neg/0/pos) | `s < t` | `s < t` |
| Sort characters | `sort(s.begin(), s.end());` (in place, mutable) | `char[] c = s.toCharArray(); Arrays.sort(c); s = new String(c);` | `s = "".join(sorted(s))` | `s = s.split("").sort().join("");` |
| String ↔ char array | `s` is already a mutable buffer of `char` | `s.toCharArray()` / `new String(chars)` | `list(s)` / `"".join(chars)` | `Array.from(s)` / `chars.join("")` |
| Iterate characters | `for (char c : s)` | `for (char c : s.toCharArray())` | `for c in s:` | `for (const c of s)` |
| Is digit / is alpha | `isdigit(c)` / `isalpha(c)` | `Character.isDigit(c)` / `Character.isLetter(c)` | `c.isdigit()` / `c.isalpha()` | `/\d/.test(c)` / `/[a-zA-Z]/.test(c)` |
| Format / interpolate | `to_string(x) + "-" + s` or `snprintf` | `String.format("%d-%s", x, s)` | `f"{x}-{s}"` | `` `${x}-${s}` `` |

C++ has no built-in `split`/`join` — the idiomatic version uses `stringstream`:

```cpp
vector<string> split(const string& s, char delim) {
    vector<string> parts;
    stringstream ss(s);
    string item;
    while (getline(ss, item, delim)) parts.push_back(item);
    return parts;
}
string join(const vector<string>& parts, const string& delim) {
    string out;
    for (size_t i = 0; i < parts.size(); i++) {
        if (i) out += delim;
        out += parts[i];
    }
    return out;
}
```

Immutability: C++ `std::string` is **mutable** — `s[i] = 'x'`, in-place `sort`, in-place `reverse` all work directly on the buffer. Java, Python, and JavaScript strings are **all immutable** — every apparent mutation (concatenation, `.replace()`, `.toUpperCase()`) allocates and returns a brand-new string.

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Efficient string building | `string s; s += piece;` (amortized O(1), mutable buffer) | `StringBuilder sb = new StringBuilder(); sb.append(piece); String s = sb.toString();` | `parts = []; parts.append(piece); s = "".join(parts)` | `const parts = []; parts.push(piece); const s = parts.join("");` |

> **Trap — O(n²) concatenation:** C++ `string += ` in a loop is fine (amortized O(1), same growth strategy as `vector::push_back`). Java and JavaScript `s += piece` in a loop is **O(n²)** — every `+=` allocates a brand-new immutable string and copies all previous content. Python `s += piece` has a fragile CPython-specific micro-optimization for exactly this pattern that you should not rely on. In all three managed languages, always accumulate into a mutable buffer (`StringBuilder` / a `list` + `"".join()` / an array + `.join("")`) and convert to the final string once at the end.

## 7. Hash Map and Hash Set

### Map

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create | `unordered_map<int,int> mp;` | `Map<Integer,Integer> mp = new HashMap<>();` | `mp = {}` | `const mp = new Map();` |
| Insert / update | `mp[k] = v;` | `mp.put(k, v);` | `mp[k] = v` | `mp.set(k, v);` |
| Get with default (no throw) | `mp.count(k) ? mp[k] : def` | `mp.getOrDefault(k, def)` | `mp.get(k, def)` | `mp.get(k) ?? def` |
| Get or throw if missing | `mp.at(k)` (throws `out_of_range`) | `mp.get(k)` (returns `null` — never throws) | `mp[k]` (raises `KeyError`) | `mp.get(k)` (returns `undefined` — never throws) |
| Contains key | `mp.count(k) > 0` / `mp.contains(k)` (C++20) | `mp.containsKey(k)` | `k in mp` | `mp.has(k)` |
| Erase key | `mp.erase(k);` | `mp.remove(k);` | `del mp[k]` | `mp.delete(k);` |
| Size | `mp.size()` | `mp.size()` | `len(mp)` | `mp.size` |
| Iterate keys | `for (auto& [k, v] : mp)` use `k` | `for (var k : mp.keySet())` | `for k in mp:` | `for (const k of mp.keys())` |
| Iterate values | (from the pair above) use `v` | `for (var v : mp.values())` | `for v in mp.values():` | `for (const v of mp.values())` |
| Iterate entries | `for (auto& [k, v] : mp)` | `for (var e : mp.entrySet())` → `e.getKey()`/`e.getValue()` | `for k, v in mp.items():` | `for (const [k, v] of mp)` |

> **Trap:** C++ `mp[k]` on a missing key **silently inserts** it with a default-constructed value (`0` for `int`) — great for counting, but means a read-only check like `if (mp[k] == target)` can accidentally grow the map. Use `.count(k)` (or `.find(k)`) for a read that must not mutate. Java/JS `.get()` on a missing key returns `null`/`undefined` and never inserts. Python `mp[k]` on a missing key **raises `KeyError`** — use `.get(k, default)` for a safe read.

Frequency-count idiom — the single most used map pattern:

```cpp
unordered_map<char,int> freq;
for (char c : s) freq[c]++;      // operator[] default-constructs 0 for missing keys
```

```java
Map<Character,Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);
// equivalently: freq.put(c, freq.getOrDefault(c, 0) + 1);
```

```python
from collections import Counter
freq = Counter(s)                 # one line replaces the whole loop
```

```js
const freq = new Map();
for (const c of s) freq.set(c, (freq.get(c) ?? 0) + 1);
```

Nested map-of-list with auto-create on first access:

```cpp
unordered_map<int, vector<int>> mp;
mp[k].push_back(x);               // auto-creates an empty vector on first access
```

```java
Map<Integer, List<Integer>> mp = new HashMap<>();
mp.computeIfAbsent(k, key -> new ArrayList<>()).add(x);
```

```python
from collections import defaultdict
mp = defaultdict(list)
mp[k].append(x)                   # auto-creates an empty list on first access
```

```js
const mp = new Map();
if (!mp.has(k)) mp.set(k, []);
mp.get(k).push(x);
// terser, with a plain object instead of Map:
const obj = {};
(obj[k] ??= []).push(x);
```

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Map keyed by a pair `(x, y)` | `map<pair<int,int>,int> mp;` (ordered; pair has `operator<`) | `Map<List<Integer>,Integer> mp = new HashMap<>();` key = `List.of(x, y)` | `mp = {}; mp[(x, y)] = v` (tuples hashable out of the box) | `const mp = new Map();` key = `` `${x},${y}` `` (string-encoded) |

> **Trap:** `unordered_map<pair<int,int>, int>` does **not compile** in plain C++ — `std::pair` has no `std::hash` specialization. Either use `map<pair<int,int>,int>` (ordered, works because `pair` has `operator<`), or supply a custom hash functor for the unordered version. Java has the analogous trap with raw arrays as keys — `int[]` uses identity `hashCode`/`equals`, so two arrays with equal contents are different keys; use `List.of(x, y)` or a `record Point(int x, int y) {}` instead. Python tuples work as dict keys with zero boilerplate (hashed by value). JavaScript `Map` keys use reference equality for objects/arrays, so a fresh `[x, y]` literal never equals another `[x, y]` literal as a key — you must encode the pair as a single string (or number) key.

Sort a map by value:

```cpp
vector<pair<int,int>> items(mp.begin(), mp.end());
sort(items.begin(), items.end(), [](auto& a, auto& b){ return a.second < b.second; });
```

```java
List<Map.Entry<Integer,Integer>> items = new ArrayList<>(mp.entrySet());
items.sort((a, b) -> a.getValue() - b.getValue());
```

```python
items = sorted(mp.items(), key=lambda kv: kv[1])
```

```js
const items = [...mp.entries()].sort((a, b) => a[1] - b[1]);
```

### Set

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create | `unordered_set<int> st;` | `Set<Integer> st = new HashSet<>();` | `st = set()` | `const st = new Set();` |
| Add | `st.insert(x);` | `st.add(x);` | `st.add(x)` | `st.add(x);` |
| Contains | `st.count(x) > 0` / `st.contains(x)` (C++20) | `st.contains(x)` | `x in st` | `st.has(x)` |
| Remove | `st.erase(x);` | `st.remove(x);` | `st.discard(x)` (no throw) | `st.delete(x);` |
| Size | `st.size()` | `st.size()` | `len(st)` | `st.size` |
| Iterate | `for (int x : st)` | `for (int x : st)` | `for x in st:` | `for (const x of st)` |
| Dedupe a list | `unordered_set<int> st(a.begin(), a.end());` | `Set<Integer> st = new HashSet<>(a);` | `st = set(a)` | `const st = new Set(a);` |
| Union | `for (x : b) a.insert(x);` | `Set<Integer> u = new HashSet<>(a); u.addAll(b);` | `a \| b` | `new Set([...a, ...b])` |
| Intersection | manual loop, or `set_intersection` on sorted ranges | `Set<Integer> i = new HashSet<>(a); i.retainAll(b);` | `a & b` | `new Set([...a].filter(x => b.has(x)))` |
| Difference | manual loop, or `set_difference` on sorted ranges | `Set<Integer> d = new HashSet<>(a); d.removeAll(b);` | `a - b` | `new Set([...a].filter(x => !b.has(x)))` |

> **Trap:** JavaScript's `Set` has no built-in union/intersection/difference on this baseline — hand-roll with spread + `filter` as above (`Set.prototype.union/intersection/difference` landed in newer engines around 2024; don't assume availability). C++ `std::unordered_set` also has no direct set-op methods; `set_union`/`set_intersection`/`set_difference` from `<algorithm>` require **sorted** ranges, so they operate on `std::set` or sorted vectors, not directly on an `unordered_set`.

## 8. Ordered Map / Ordered Set

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Create ordered map | `map<int,int> mp;` | `TreeMap<Integer,Integer> mp = new TreeMap<>();` | `from sortedcontainers import SortedDict` | none built in — hand-roll (see Trap) |
| Create ordered set | `set<int> st;` | `TreeSet<Integer> st = new TreeSet<>();` | `from sortedcontainers import SortedList` | none built in — hand-roll (see Trap) |
| Insert | `mp[k] = v;` / `st.insert(x);` | `mp.put(k, v);` / `st.add(x);` | `mp[k] = v` / `st.add(x)` | maintain a sorted array + insert at the binary-searched index |
| Largest key `<= x` (floor) | no direct method — see Trap below | `mp.floorKey(x)` / `st.floor(x)` | `sl[sl.bisect_right(x) - 1]` | manual (see helper below) |
| Smallest key `>= x` (ceiling) | `mp.lower_bound(x)` (iterator) | `mp.ceilingKey(x)` / `st.ceiling(x)` | `sl[sl.bisect_left(x)]` | manual |
| Smallest key `> x` (higher) | `mp.upper_bound(x)` | `mp.higherKey(x)` / `st.higher(x)` | `sl[sl.bisect_right(x)]` | manual |
| Largest key `< x` (lower) | no direct method — derive from `lower_bound` minus one | `mp.lowerKey(x)` / `st.lower(x)` | `sl[sl.bisect_left(x) - 1]` | manual |
| First (min) key | `mp.begin()->first` / `*st.begin()` | `mp.firstKey()` / `st.first()` | `sl[0]` | `arr[0]` |
| Last (max) key | `mp.rbegin()->first` / `*st.rbegin()` | `mp.lastKey()` / `st.last()` | `sl[-1]` | `arr[arr.length - 1]` |
| Iterate in sorted order | `for (auto& [k, v] : mp)` — always sorted by key | `for (var k : mp.keySet())` — always sorted | `for x in sl:` — always sorted | `for (const x of arr)` — sorted only because you maintained it |
| Range query `[lo, hi)` | `mp.lower_bound(lo)` … `mp.lower_bound(hi)` | `mp.subMap(lo, hi)` | `sl.irange(lo, hi, inclusive=(True, False))` | `arr.slice(lowerBound(arr, lo), lowerBound(arr, hi))` |

> **Trap:** C++ `map`/`set` have no direct `floor`/`lower` methods — only `lower_bound` (first `>= x`) and `upper_bound` (first `> x`). To get floor (`<= x`), take `upper_bound(x)` and step the iterator one back, guarding against `begin()`. Java's `TreeMap`/`TreeSet` give you `floorKey`/`floor` and `lowerKey`/`lower` directly — far less error-prone.
Deriving floor / range queries where the library has no direct method:

```cpp
// floor(x): largest key <= x
auto it = mp.upper_bound(x);
if (it == mp.begin()) { /* no floor exists */ }
else { --it; /* it->first is the floor key */ }

// range query: all keys in [lo, hi)
for (auto it = mp.lower_bound(lo); it != mp.lower_bound(hi); ++it) {
    // it->first, it->second
}
```

```java
// Java gives you these directly, including a live sub-map view:
SortedMap<Integer,Integer> range = mp.subMap(lo, hi);   // keys in [lo, hi)
Integer f = mp.floorKey(x);                              // null if none
```

> **Trap (hard):** JavaScript has **no ordered map/set anywhere** — not in the language, not in a standard library. There is no TreeMap equivalent. For floor/ceiling/range-query problems you must either (a) maintain a plain sorted array and binary-search it yourself (insert: find the index, then `arr.splice(idx, 0, x)` — O(n) per insert, acceptable for moderate n), or (b) implement/import a balanced structure (e.g. a Fenwick tree or a self-balancing BST) if you need faster inserts. Budget time to write the binary-search helper — there is no shortcut.

### Semantics table — the names collide across languages, so pin them down precisely

| Name | Language / library | Precise meaning |
|---|---|---|
| `lower_bound(x)` | C++ `map`/`set`/`<algorithm>` | iterator/pointer to the **first element `>= x`** |
| `upper_bound(x)` | C++ `map`/`set`/`<algorithm>` | iterator/pointer to the **first element `> x`** |
| `floorKey(x)` / `floor(x)` | Java `TreeMap`/`TreeSet` | the **largest key `<= x`** (`null` if none) |
| `ceilingKey(x)` / `ceiling(x)` | Java `TreeMap`/`TreeSet` | the **smallest key `>= x`** (`null` if none) |
| `higherKey(x)` / `higher(x)` | Java `TreeMap`/`TreeSet` | the **smallest key `> x`** (`null` if none) |
| `lowerKey(x)` / `lower(x)` | Java `TreeMap`/`TreeSet` | the **largest key `< x`** (`null` if none) |
| `bisect_left(a, x)` | Python `bisect` | insertion index **before** any existing `x` — index of the **first element `>= x`** |
| `bisect_right(a, x)` | Python `bisect` | insertion index **after** any existing `x` — index of the **first element `> x`** |

Cross-reference: C++ `lower_bound` ≈ Python `bisect_left` ≈ Java `ceilingKey` — all three answer "first/smallest element `>= x`". C++ `upper_bound` ≈ Python `bisect_right` ≈ Java `higherKey` — all three answer "first/smallest element `> x`". Java's `floorKey` (largest `<= x`) and `lowerKey` (largest `< x`) have no single-call C++/Python equivalent — derive them by taking `upper_bound`/`bisect_right` (or `lower_bound`/`bisect_left`) and stepping one position back, guarding the start of the container.

```python
import bisect
a = [1, 3, 5, 7, 9]              # must be kept sorted by the caller
bisect.insort(a, 4)              # insert keeping sorted order -> [1, 3, 4, 5, 7, 9]
i = bisect.bisect_left(a, 5)     # 3 -> index of first element >= 5
j = bisect.bisect_right(a, 5)    # 4 -> index of first element > 5
```

```python
from sortedcontainers import SortedList, SortedDict
sl = SortedList([5, 1, 3])
sl.add(4)                        # O(log n) insert, stays sorted
sl.bisect_left(3)                # same semantics as the bisect module
sl[0]; sl[-1]                    # min, max in O(1)
sd = SortedDict()
sd[5] = "v"
sd.peekitem(0)                   # (min key, value)
```

```js
// JavaScript has nothing built in — hand-rolled binary search on a sorted array.
function lowerBound(arr, x) {    // first index with arr[i] >= x
    let lo = 0, hi = arr.length;
    while (lo < hi) {
        const mid = (lo + hi) >> 1;
        if (arr[mid] < x) lo = mid + 1; else hi = mid;
    }
    return lo;
}
function upperBound(arr, x) {    // first index with arr[i] > x
    let lo = 0, hi = arr.length;
    while (lo < hi) {
        const mid = (lo + hi) >> 1;
        if (arr[mid] <= x) lo = mid + 1; else hi = mid;
    }
    return lo;
}
// insert x while keeping arr sorted:
const idx = lowerBound(arr, x);
arr.splice(idx, 0, x);           // O(n) per insert
// floor(x) = arr[lowerBound(arr, x+1) - 1]   (integer keys)
// ceiling(x) = arr[lowerBound(arr, x)]
```

---

# PART B — ALGORITHMS, IDIOMS AND TRAPS

## 9. Stack, Queue, Deque

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Stack type | `stack<int> st;` | `Deque<Integer> st = new ArrayDeque<>();` | `st = []` | `const st = [];` |
| Push | `st.push(x);` | `st.push(x);` | `st.append(x)` | `st.push(x);` |
| Pop | `st.pop();` (void, use `top()` first) | `st.pop();` (returns value) | `st.pop()` (returns value) | `st.pop();` (returns value) |
| Top/peek | `st.top();` | `st.peek();` | `st[-1]` | `st[st.length-1]` |
| Empty | `st.empty();` | `st.isEmpty();` | `not st` | `st.length === 0` |
| Queue type | `queue<int> q;` | `Deque<Integer> q = new ArrayDeque<>();` | `q = collections.deque();` | see Trap below |
| Push (enqueue) | `q.push(x);` | `q.offer(x);` | `q.append(x)` | `q.push(x);` |
| Pop (dequeue) | `q.pop();` (void, use `front()`) | `q.poll();` (returns value) | `q.popleft()` (returns value) | `q.shift();` — **O(n), avoid** |
| Front | `q.front();` | `q.peek();` | `q[0]` | `q[0]` |
| Deque both ends | `deque<int> dq;` | `Deque<Integer> dq = new ArrayDeque<>();` | `dq = collections.deque();` | hand-rolled `Deque` class |
| Push front/back | `dq.push_front(x); dq.push_back(x);` | `dq.addFirst(x); dq.addLast(x);` | `dq.appendleft(x); dq.append(x)` | class methods (see below) |
| Pop front/back | `dq.pop_front(); dq.pop_back();` | `dq.pollFirst(); dq.pollLast();` | `dq.popleft(); dq.pop()` | class methods (see below) |

**Complexity (amortized), all four when using the recommended container:**

| Op | push/enqueue | pop/dequeue | front/top | random access |
|---|---|---|---|---|
| Cost | O(1) | O(1) | O(1) | O(1) stack/array top only; O(n) mid-deque |

> **Trap:** Java's legacy `Stack` and `LinkedList` are synchronized (thread-safety overhead) and `Stack` extends `Vector` — slower and semantically muddled. Always use `ArrayDeque` for stack, queue, and deque in Java; it has no capacity/thread-safety tax and O(1) on both ends.

> **Trap:** Python `list.pop(0)` and `list.insert(0, x)` are **O(n)** because the whole backing array shifts. Never use a plain `list` as a queue — use `collections.deque`, which is a doubly linked list of blocks giving O(1) at both ends.

> **Trap:** JS arrays have no O(1) `shift()`/`unshift()` — both are **O(n)** (every remaining element is re-indexed). For a queue, either (a) use a head-index pointer and only physically compact occasionally, or (b) hand-roll a `Deque`.

```javascript
// JS: index-pointer queue (amortized O(1) dequeue, avoids shift())
class Queue {
  constructor() { this.buf = []; this.head = 0; }
  push(x) { this.buf.push(x); }
  shift() {
    const v = this.buf[this.head++];
    if (this.head > 1000 && this.head * 2 > this.buf.length) {
      this.buf = this.buf.slice(this.head); this.head = 0;
    }
    return v;
  }
  front() { return this.buf[this.head]; }
  get length() { return this.buf.length - this.head; }
  get isEmpty() { return this.length === 0; }
}

// JS: full deque via doubly linked list (or wrap two Queues) when both ends needed
class Deque {
  constructor() { this.items = []; } // simplest correct version; for hot loops use a linked list
  pushBack(x) { this.items.push(x); }
  pushFront(x) { this.items.unshift(x); } // O(n) — replace with linked-list nodes if this is hot
  popBack() { return this.items.pop(); }
  popFront() { return this.items.shift(); } // O(n) — same caveat
  get front() { return this.items[0]; }
  get back() { return this.items[this.items.length - 1]; }
  get length() { return this.items.length; }
}
```

## 10. Heap / Priority Queue

**Default ordering — the single biggest cross-language gotcha:**

| Language | Container | Default order |
|---|---|---|
| C++ | `priority_queue<int>` | **MAX**-heap |
| Java | `PriorityQueue<Integer>` | **MIN**-heap |
| Python | `heapq` module | **MIN**-heap only (no max mode) |
| JavaScript | *(none built in)* | N/A — must hand-roll |

> **Trap:** Porting a C++ `priority_queue<int> pq;` (max-heap) to Java as `new PriorityQueue<>()` silently flips to a **min**-heap — same code, opposite semantics, no compile error. Always check ordering explicitly when translating.

**Declare a min-heap:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `priority_queue<int, vector<int>, greater<int>> pq;` | `PriorityQueue<Integer> pq = new PriorityQueue<>();` | `h = []` (list used with `heapq`) | `const h = new MinHeap();` (see class below) |

**Declare a max-heap:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `priority_queue<int> pq;` (default) | `PriorityQueue<Integer> pq = new PriorityQueue<>(Comparator.reverseOrder());` | push `-x`, pop and negate back | `new MinHeap((a,b) => b - a)` (flip comparator) |

**Push / pop / peek / size:**

| Op | C++ | Java | Python | JavaScript (MinHeap class) |
|---|---|---|---|---|
| Push | `pq.push(x);` | `pq.offer(x);` | `heapq.heappush(h, x)` | `h.push(x);` |
| Pop | `pq.pop();` (void; use `top()` first) | `pq.poll();` (returns value) | `heapq.heappop(h)` (returns value) | `h.pop();` (returns value) |
| Peek | `pq.top();` | `pq.peek();` | `h[0]` | `h.peek();` |
| Size | `pq.size();` | `pq.size();` | `len(h)` | `h.size();` |
| Empty | `pq.empty();` | `pq.isEmpty();` | `not h` | `h.size() === 0` |

**Heapify an existing array (O(n)):**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `priority_queue<int> pq(v.begin(), v.end());` | `new PriorityQueue<>(list)` — *becomes min-heap, order NOT guaranteed to equal list order* | `heapq.heapify(arr)` (in-place, O(n)) | `MinHeap.from(arr)` (push each, O(n log n) unless you implement sift-down heapify) |

> **Trap:** `new PriorityQueue<>(collection)` in Java only guarantees heap *property*, not that iteration order matches input order — never rely on `PriorityQueue.toString()` or iteration for sorted output.

**Heap of pairs / tuples (e.g. `(distance, node)`):**

```cpp
// C++: pair orders lexicographically by default — exactly what Dijkstra wants
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>> pq;
pq.push({dist, node});
auto [d, u] = pq.top();
```

```java
// Java: array-of-two, or a small record with Comparable, via a lambda comparator
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]); // careful of overflow, prefer Integer.compare
pq.offer(new int[]{dist, node});
int[] top = pq.peek();
```

```python
# Python: heap of tuples — compares element-wise, exactly like C++ pair
import heapq
h = []
heapq.heappush(h, (dist, node))
d, node = h[0]
```

```javascript
// JS: MinHeap with a comparator on arrays/objects
const h = new MinHeap((a, b) => a[0] - b[0]);
h.push([dist, node]);
```

> **Trap (Python heap-of-tuples crash):** If the second tuple element is not comparable (e.g. a list, or a custom object without `__lt__`), Python raises `TypeError: '<' not supported` **only when two first elements tie** — this can pass all early test cases and crash later. Fix: add a **tie-break counter** (or any always-comparable field) as the second element so ties never reach the uncomparable field.

```python
import heapq, itertools
counter = itertools.count()  # unique, increasing, always comparable
h = []
heapq.heappush(h, (priority, next(counter), payload))  # payload can be anything now
```

**Max-heap via negation (Python idiom):**

```python
# Python has no max-heap — negate on push, negate back on pop
heapq.heappush(h, -x)
biggest = -heapq.heappop(h)
```

**Heap with custom comparator on objects:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| pass a comparator struct/lambda as the 3rd template arg | `new PriorityQueue<>(Comparator.comparingInt(o -> o.key))` | wrap in tuple `(o.key, o)` or give class `__lt__` | pass a comparator function to the `MinHeap` constructor |

**k-largest / k-smallest idiom (min-heap of size k):**

```python
# Keep a MIN-heap of size k while scanning — top is the kth-largest so far
import heapq
h = []
for x in nums:
    heapq.heappush(h, x)
    if len(h) > k:
        heapq.heappop(h)
return h[0]  # kth largest overall
```
Same shape in all four: min-heap capped at size `k`; push, and pop-when-oversize. (For k-*smallest*, use a max-heap capped at size k instead.)

**JS hand-rolled MinHeap — paste this whole class, it is required infrastructure:**

```javascript
class MinHeap {
  constructor(cmp = (a, b) => a - b) { this.data = []; this.cmp = cmp; }
  static from(arr, cmp) {
    const h = new MinHeap(cmp);
    h.data = arr.slice();
    for (let i = (h.data.length >> 1) - 1; i >= 0; i--) h._siftDown(i);
    return h;
  }
  size() { return this.data.length; }
  peek() { return this.data[0]; }
  push(val) {
    this.data.push(val);
    this._siftUp(this.data.length - 1);
  }
  pop() {
    const top = this.data[0];
    const last = this.data.pop();
    if (this.data.length > 0) {
      this.data[0] = last;
      this._siftDown(0);
    }
    return top;
  }
  _siftUp(i) {
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (this.cmp(this.data[i], this.data[p]) < 0) {
        [this.data[i], this.data[p]] = [this.data[p], this.data[i]];
        i = p;
      } else break;
    }
  }
  _siftDown(i) {
    const n = this.data.length;
    while (true) {
      let l = 2 * i + 1, r = 2 * i + 2, smallest = i;
      if (l < n && this.cmp(this.data[l], this.data[smallest]) < 0) smallest = l;
      if (r < n && this.cmp(this.data[r], this.data[smallest]) < 0) smallest = r;
      if (smallest === i) break;
      [this.data[i], this.data[smallest]] = [this.data[smallest], this.data[i]];
      i = smallest;
    }
  }
}
// Max-heap: new MinHeap((a, b) => b - a);
```

## 11. Sorting and Comparators

**Comparator model — fundamentally different in each language:**

| Language | Model | Signature |
|---|---|---|
| C++ | boolean **less-than** predicate | `bool cmp(const T&a, const T&b)` — true if `a` goes before `b` |
| Java | `Comparator<T>` returning **int** | negative if `a`<`b`, 0 if equal, positive if `a`>`b` |
| Python | `key=` **function** (not a comparator!) | maps each element to a sort key; escape hatch `functools.cmp_to_key` for true comparators |
| JavaScript | comparator returning a **number** | negative → `a` before `b`; positive → `b` before `a`; 0 → keep relative order (stable since ES2019) |

**Sort ascending / descending:**

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Ascending (ints) | `sort(v.begin(), v.end());` | `Arrays.sort(arr);` / `Collections.sort(list);` | `arr.sort()` / `sorted(arr)` | `arr.sort((a, b) => a - b);` |
| Descending (ints) | `sort(v.begin(), v.end(), greater<int>());` | `Arrays.sort(arr); reverse` boxed, or `Collections.sort(list, Comparator.reverseOrder());` | `arr.sort(reverse=True)` | `arr.sort((a, b) => b - a);` |

**Sort by one key:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `sort(v.begin(), v.end(), [](auto&a, auto&b){ return a.second < b.second; });` | `list.sort(Comparator.comparingInt(o -> o.value));` | `arr.sort(key=lambda o: o.value)` | `arr.sort((a, b) => a.value - b.value);` |

**Sort by multiple keys (primary then tiebreak):**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `sort(v.begin(), v.end(), [](auto&a, auto&b){ if (a.x != b.x) return a.x < b.x; return a.y < b.y; });` | `list.sort(Comparator.comparingInt((O o) -> o.x).thenComparingInt(o -> o.y));` | `arr.sort(key=lambda o: (o.x, o.y))` | `arr.sort((a, b) => a.x - b.x \|\| a.y - b.y);` |

**Sort 2D array / list of lists by a column:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `sort(v.begin(), v.end(), [](auto&a, auto&b){ return a[1] < b[1]; });` | `Arrays.sort(arr, (a, b) -> a[1] - b[1]);` | `arr.sort(key=lambda row: row[1])` | `arr.sort((a, b) => a[1] - b[1]);` |

**Sort indices by value (argsort):**

```cpp
// C++
vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int i, int j){ return v[i] < v[j]; });
```
```java
// Java — must box, use Integer[] not int[] so the comparator applies
Integer[] idx = new Integer[n];
for (int i = 0; i < n; i++) idx[i] = i;
Arrays.sort(idx, (i, j) -> v[i] - v[j]); // or Integer.compare(v[i], v[j])
```
```python
# Python
idx = sorted(range(n), key=lambda i: v[i])
```
```javascript
// JavaScript
const idx = Array.from({length: n}, (_, i) => i);
idx.sort((i, j) => v[i] - v[j]);
```

**Sort strings:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `sort(s.begin(), s.end());` (chars) | `char[] c = s.toCharArray(); Arrays.sort(c); s = new String(c);` | `''.join(sorted(s))` | `s.split('').sort().join('');` |
| Sort array of strings | `sort(words.begin(), words.end());` (lexicographic) | `Arrays.sort(words);` (lexicographic) | `words.sort()` (lexicographic) | `words.sort();` (lexicographic — this one is correct by default) |

**Stability:** C++ `sort` is **not** guaranteed stable (use `stable_sort` if needed); `std::stable_sort`, Java `Arrays.sort`/`Collections.sort` (Timsort on objects — stable; **primitive `int[]` uses dual-pivot quicksort — not stable**, irrelevant since primitives carry no payload), Python `sorted`/`list.sort` (Timsort — always stable), JavaScript `Array.prototype.sort` (stable since ES2019 / all modern engines).

> **Trap:** JS `.sort()` with **no comparator** sorts by converting elements to strings and comparing **lexicographically**, not numerically. `[10, 2, 1].sort()` → compares `"10"`, `"2"`, `"1"` as strings → `"1" < "10" < "2"` → result is `[1, 10, 2]`, not `[1, 2, 10]`. Always pass `(a, b) => a - b` for numeric sorts.

> **Trap:** Java `(a, b) -> a - b` on `Integer`/`int` comparators **overflows** when `a` and `b` have opposite signs and large magnitude (e.g. `a = -2000000000, b = 2000000000` → `a - b` overflows `int`, wraps to positive, comparator reports the wrong order). Always use `Integer.compare(a, b)` (or `Long.compare` for longs) instead of subtraction.

> **Trap:** A C++ comparator **must** define a strict-weak-ordering (irreflexive, and `!cmp(a,b)&&!cmp(b,a)` implies equivalence). A comparator that returns `true` for `cmp(a,a)`, or that is inconsistent (e.g. `return a.x <= b.x;` using `<=` instead of `<`), causes **undefined behavior** — typically a crash or infinite loop inside `std::sort`, not a clean wrong answer.

> **Trap:** Python's `sort`/`sorted` take **`key=`**, a function mapping element→sortkey — this is *not* the same as a two-argument comparator. Old-style `cmp=` comparators were removed in Python 3. To port a true pairwise comparator, wrap it: `sorted(arr, key=functools.cmp_to_key(cmp))` where `cmp(a, b)` returns negative/0/positive like Java's `Comparator`.

## 12. Binary Search

**Built-in lower_bound / upper_bound equivalents:**

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| First index with `arr[i] >= x` | `lower_bound(v.begin(), v.end(), x) - v.begin();` | no direct equivalent — decode `binarySearch` (below) or hand-roll | `bisect.bisect_left(arr, x)` | *(none built in — hand-roll, see below)* |
| First index with `arr[i] > x` | `upper_bound(v.begin(), v.end(), x) - v.begin();` | hand-roll | `bisect.bisect_right(arr, x)` | *(none — hand-roll)* |
| Exact search | `binary_search(v.begin(), v.end(), x);` (bool only) | `Arrays.binarySearch(arr, x);` | `i = bisect_left(arr, x); found = i < len(arr) and arr[i] == x` | hand-roll |

**Java `Arrays.binarySearch` decode:** returns the index of `x` if found; if **not** found, returns `-(insertionPoint) - 1`, where `insertionPoint` is where `x` would go (i.e. equivalent to `lower_bound`'s result).

```java
int r = Arrays.binarySearch(arr, x);
int insertionPoint = (r >= 0) ? r : -(r + 1); // == C++ lower_bound index either way
boolean found = r >= 0;
```

> **Trap:** `Arrays.binarySearch` on an array containing duplicates does **not** guarantee it finds the first or last occurrence — it returns *some* matching index. Don't rely on it for "leftmost equal" logic; use the hand-rolled template instead.

**Hand-rolled binary search template — overflow-safe mid, in all four:**

```cpp
// C++: first index i such that arr[i] >= x  (lower_bound semantics)
int lo = 0, hi = (int)arr.size(); // hi is EXCLUSIVE
while (lo < hi) {
    int mid = lo + (hi - lo) / 2; // overflow-safe vs (lo+hi)/2
    if (arr[mid] >= x) hi = mid;
    else lo = mid + 1;
}
return lo; // == arr.size() if none found
```

```java
// Java
int lo = 0, hi = arr.length;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (arr[mid] >= x) hi = mid;
    else lo = mid + 1;
}
return lo;
```

```python
# Python — no overflow to guard against (ints are arbitrary precision), but keep the pattern uniform
lo, hi = 0, len(arr)
while lo < hi:
    mid = lo + (hi - lo) // 2
    if arr[mid] >= x:
        hi = mid
    else:
        lo = mid + 1
return lo
```

```javascript
// JavaScript
let lo = 0, hi = arr.length;
while (lo < hi) {
  const mid = lo + ((hi - lo) >> 1); // integer mid via bitshift
  if (arr[mid] >= x) hi = mid;
  else lo = mid + 1;
}
return lo;
```

**Variants (same template, change the predicate inside):**

| Goal | Predicate to keep on the `hi = mid` side |
|---|---|
| First index `>= x` | `arr[mid] >= x` |
| First index `> x` | `arr[mid] > x` |
| Last index `<= x` | find first index `> x`, then subtract 1 |
| Last index `< x` | find first index `>= x`, then subtract 1 |
| Count of values in `[lo_v, hi_v]` | `firstIndex(hi_v + 1) - firstIndex(lo_v)` (adapt to `>=` predicate) |

**Binary search on the answer (parametric search) skeleton — identical shape in all four:**

```python
def can(mid):  # C++/Java/JS: bool/boolean can(mid) { ... }
    ...  # feasibility check for candidate answer `mid`
    return True or False

lo, hi = LOW, HIGH  # search space of *answers*, not array indices
while lo < hi:
    mid = lo + (hi - lo) // 2
    if can(mid):
        hi = mid       # shrink toward feasible region (or lo = mid depending on monotonic direction)
    else:
        lo = mid + 1
answer = lo
```
The direction of `can(mid)`'s truth (feasible region on the left vs right of the boundary) determines whether the true branch sets `hi = mid` or `lo = mid`; in the `lo = mid` case use `mid = lo + (hi - lo + 1) / 2` to avoid an infinite loop.

## 13. Classes, Nodes and Custom Ordering

**Linked-list node:**

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};
```
```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int x) { val = x; }
}
```
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```
```javascript
class ListNode {
  constructor(val = 0, next = null) {
    this.val = val;
    this.next = next;
  }
}
```

**Binary-tree node:**

```cpp
struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};
```
```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int x) { val = x; }
}
```
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```
```javascript
class TreeNode {
  constructor(val = 0, left = null, right = null) {
    this.val = val;
    this.left = left;
    this.right = right;
  }
}
```

**Graph node (adjacency-style, generic):**

```cpp
struct Node {
    int val;
    vector<Node*> neighbors;
};
```
```java
class Node {
    int val;
    List<Node> neighbors = new ArrayList<>();
    Node(int val) { this.val = val; }
}
```
```python
class Node:
    def __init__(self, val=0, neighbors=None):
        self.val = val
        self.neighbors = neighbors if neighbors is not None else []
```
```javascript
class Node {
  constructor(val = 0, neighbors = []) {
    this.val = val;
    this.neighbors = neighbors;
  }
}
```

**General struct/record (immutable-ish data holder):**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `struct Point { int x, y; };` | `record Point(int x, int y) {}` (Java 17 record — free `equals`/`hashCode`/`toString`) | `@dataclass\nclass Point:\n    x: int\n    y: int` | `class Point { constructor(x, y) { this.x = x; this.y = y; } }` or plain `{x, y}` object |

**Making an object sortable / heap-able:**

| Language | Mechanism |
|---|---|
| C++ | define `bool operator<(const T& other) const { ... }` (used by `sort`, `set`, `priority_queue` default) |
| Java | implement `Comparable<T>` (`compareTo`) for a natural order, or pass a `Comparator<T>` at the call site |
| Python | define `__lt__(self, other)` (enough for `sort`/`heapq`), or use `@dataclass(order=True)` to auto-generate all six comparisons from field order |
| JavaScript | **no operator overloading at all** — objects are only ever ordered via an explicit comparator function passed to `.sort()` or the heap class |

```cpp
struct Job { int deadline, profit;
    bool operator<(const Job& o) const { return deadline < o.deadline; } };
```
```java
class Job implements Comparable<Job> {
    int deadline, profit;
    public int compareTo(Job o) { return Integer.compare(deadline, o.deadline); }
}
```
```python
from dataclasses import dataclass, field

@dataclass(order=True)
class Job:
    deadline: int
    profit: int = field(compare=False)  # excluded from ordering
```
```javascript
class Job {
  constructor(deadline, profit) { this.deadline = deadline; this.profit = profit; }
}
jobs.sort((a, b) => a.deadline - b.deadline); // ordering lives at the call site, never on the class
```

**Making an object usable as a hash key (in a set/map):**

| Language | Mechanism |
|---|---|
| C++ | provide a custom `hash<T>` specialization + `operator==`, or use a `map`/`set` (tree-based, needs only `operator<`) instead of `unordered_map` |
| Java | override both `equals()` **and** `hashCode()` consistently (or use a `record`, which generates both) |
| Python | define `__eq__` and `__hash__` together (defining `__eq__` alone sets `__hash__` to `None`), or use an immutable tuple / `@dataclass(frozen=True)` |
| JavaScript | objects are compared by **reference** in `Map`/`Set`, never by value — use a deterministic string key (e.g. `` `${x},${y}` ``) as the map key instead of the object itself |

> **Trap:** In Java, overriding `equals()` without overriding `hashCode()` (or vice versa) breaks `HashMap`/`HashSet` silently — lookups fail even for "equal" objects, since they land in different hash buckets.

> **Trap:** In Python, defining `__eq__` on a class automatically sets `__hash__` to `None`, making instances unhashable — you must explicitly define `__hash__` too (or freeze the dataclass) if you want to put instances in a `set` or use them as `dict` keys.

**Constructor syntax / field access / methods / static members, side by side:**

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Constructor | `Point(int x, int y) : x(x), y(y) {}` | `Point(int x, int y) { this.x = x; this.y = y; }` | `def __init__(self, x, y): self.x, self.y = x, y` | `constructor(x, y) { this.x = x; this.y = y; }` |
| Field access | `p.x` | `p.x` | `p.x` | `p.x` |
| Method | `int sum() const { return x + y; }` | `int sum() { return x + y; }` | `def sum(self): return self.x + self.y` | `sum() { return this.x + this.y; }` |
| Static member | `static int count;` (defined outside class too) | `static int count;` | `count = 0` (class body, shared unless shadowed) | `static count = 0;` |
| Static method | `static int f(int x);` | `static int f(int x) { ... }` | `@staticmethod\ndef f(x): ...` | `static f(x) { ... }` |

## 14. Recursion, Memoization and DP

**Recursion depth limits and workarounds:**

| Language | Practical limit | Workaround |
|---|---|---|
| C++ | Very large (megabytes of stack, tens of thousands to ~1M frames depending on frame size) | Rarely an issue for LeetCode-scale input; if it is, convert to an explicit stack/iterative form |
| Java | ~10,000 frames on the default thread stack | Run the recursion on a thread with a larger stack |
| Python | 1000 by default (`sys.getrecursionlimit()`) | `sys.setrecursionlimit(1_000_000)` **and** run on a thread with a bigger OS stack, since the interpreter still needs real stack space |
| JavaScript (V8/Node) | ~10,000–15,000 frames, engine-dependent | No official fix — rewrite as iterative / explicit stack; `--stack-size` exists but is fragile and not always permitted on judges |

```java
// Java: run deep recursion on a thread with a larger stack (default is often 512KB-1MB)
Thread t = new Thread(null, () -> {
    // deep recursive call here
    solve(root);
}, "deep-recursion", 1 << 26); // 64MB stack
t.start();
t.join();
```

```python
# Python: raise the limit AND use a real thread with a bigger stack
import sys, threading

sys.setrecursionlimit(1_000_000)
threading.stack_size(64 * 1024 * 1024)  # 64MB

result = []
def run():
    result.append(solve(root))

t = threading.Thread(target=run)
t.start()
t.join()
```

**Memoization container:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `unordered_map<int,int> memo;` or `vector<int> memo(n, -1);` | `Map<Integer,Integer> memo = new HashMap<>();` or `int[] memo = new int[n]; Arrays.fill(memo, -1);` | `@lru_cache(None)` decorator on the function (zero boilerplate), or `memo = {}` | `const memo = new Map();` or a plain object `{}` |

```python
# Python's one-liner superpower — no manual cache management at all
from functools import lru_cache

@lru_cache(maxsize=None)  # or @cache in 3.9+
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)
```

> **Trap:** `@lru_cache` requires all arguments to be **hashable** (no lists/dicts as args — pass tuples instead), and it caches across calls for the lifetime of the process/function object, so clear it (`fib.cache_clear()`) between independent test cases if reusing the same decorated function.

**1D DP array with an INFINITY sentinel:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `vector<int> dp(n, INT_MAX);` | `int[] dp = new int[n]; Arrays.fill(dp, Integer.MAX_VALUE);` | `dp = [float('inf')] * n` | `const dp = new Array(n).fill(Infinity);` |

> **Trap:** `dp[i] = dp[j] + weight` where `dp[j]` is still the sentinel **overflows** in C++ (`INT_MAX + weight` is UB / wraps) and Java (`Integer.MAX_VALUE + weight` silently wraps to a large negative number) — both can corrupt a `min()` comparison into picking the "infinite" branch as smallest. Python's `float('inf') + weight` stays `inf` (safe), and JS's `Infinity + weight` also stays `Infinity` (safe). **Fix for C++/Java:** guard with an explicit `if (dp[j] == SENTINEL) continue;` before adding, or use a sentinel far from the type's max (e.g. `INT_MAX / 2`) so one addition can't wrap.

**2D DP table creation:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `vector<vector<int>> dp(m, vector<int>(n, 0));` | `int[][] dp = new int[m][n];` (auto-zeroed) | `dp = [[0]*n for _ in range(m)]` | `const dp = Array.from({length: m}, () => new Array(n).fill(0));` |

> **Trap (shared-row):** `[[0]*n]*m` in Python and `Array(m).fill(new Array(n).fill(0))` in JavaScript both create **one** inner array/list referenced by every row — mutating `dp[0][0]` silently changes `dp[1][0]`, `dp[2][0]`, etc. Always build each row independently: Python's list-comprehension form above, or JS's `Array.from({length: m}, () => ...)` form above (never `.fill()` with a shared array reference).

**Backtracking PATH COPY — the single biggest cross-language trap in this whole document:**

| Language | What happens with `results.add(path)` | Fix |
|---|---|---|
| C++ | `push_back(path)` on a `vector<vector<int>>` **copies** the vector automatically (value semantics) | Nothing needed — `results.push_back(path);` is already safe |
| Java | `results.add(path)` stores a **reference** — every later mutation of `path` retroactively changes everything already added | `results.add(new ArrayList<>(path));` |
| Python | `results.append(path)` stores a **reference** to the same list object | `results.append(path[:])` or `results.append(list(path))` |
| JavaScript | `results.push(path)` stores a **reference** to the same array | `results.push([...path]);` or `results.push(path.slice());` |

```cpp
// C++ backtracking skeleton
void backtrack(vector<int>& path, /* state */) {
    if (isSolution(path)) {
        results.push_back(path); // COPIES automatically — safe
        return;
    }
    for (int choice : choices) {
        path.push_back(choice);
        backtrack(path, /* updated state */);
        path.pop_back(); // undo
    }
}
```

```java
// Java backtracking skeleton
void backtrack(List<Integer> path /*, state */) {
    if (isSolution(path)) {
        results.add(new ArrayList<>(path)); // MUST copy — reference escapes otherwise
        return;
    }
    for (int choice : choices) {
        path.add(choice);
        backtrack(path /*, updated state */);
        path.remove(path.size() - 1); // undo
    }
}
```

```python
# Python backtracking skeleton
def backtrack(path):
    if is_solution(path):
        results.append(path[:])  # MUST copy — reference escapes otherwise
        return
    for choice in choices:
        path.append(choice)
        backtrack(path)
        path.pop()  # undo
```

```javascript
// JavaScript backtracking skeleton
function backtrack(path /*, state */) {
  if (isSolution(path)) {
    results.push([...path]); // MUST copy — reference escapes otherwise
    return;
  }
  for (const choice of choices) {
    path.push(choice);
    backtrack(path /*, updated state */);
    path.pop(); // undo
  }
}
```

## 15. Bit Manipulation

**Test / set / clear / toggle a bit:**

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Test bit `i` | `(x >> i) & 1` | `(x >> i) & 1` | `(x >> i) & 1` | `(x >> i) & 1` |
| Set bit `i` | `x \|= (1 << i);` | `x \|= (1 << i);` | `x \|= (1 << i)` | `x \|= (1 << i);` |
| Clear bit `i` | `x &= ~(1 << i);` | `x &= ~(1 << i);` | `x &= ~(1 << i)` | `x &= ~(1 << i);` |
| Toggle bit `i` | `x ^= (1 << i);` | `x ^= (1 << i);` | `x ^= (1 << i)` | `x ^= (1 << i);` |
| Check power of two | `x > 0 && (x & (x-1)) == 0` | `x > 0 && (x & (x-1)) == 0` | `x > 0 and (x & (x-1)) == 0` | `x > 0 && (x & (x-1)) === 0` |

**Count set bits (popcount):**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `__builtin_popcount(x)` (unsigned int); `__builtin_popcountll(x)` for 64-bit | `Integer.bitCount(x)`; `Long.bitCount(x)` for 64-bit | `x.bit_count()` (3.10+) or `bin(x).count('1')` (older) | hand-rolled: `x.toString(2).split('0').join('').length` (slow) or Brian Kernighan loop (fast) |

```javascript
// JS: fast popcount without string conversion (Brian Kernighan's algorithm)
function popcount(x) {
  let count = 0;
  while (x !== 0) {
    x &= x - 1; // clears the lowest set bit
    count++;
  }
  return count;
}
```

**Highest set bit / log2:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `31 - __builtin_clz(x)` (undefined if `x==0`) | `31 - Integer.numberOfLeadingZeros(x)` | `x.bit_length() - 1` | `31 - Math.clz32(x)` |

**XOR tricks:**

| Trick | Expression (same in all four, adjust syntax) |
|---|---|
| Swap without temp | `a ^= b; b ^= a; a ^= b;` |
| Find single non-duplicate in array where others pair up | `reduce((acc, x) => acc ^ x, 0)` |
| Check if two ints differ only by bit `i` | `((a ^ b) >> i) & 1` |

**Iterate all subsets of a bitmask `mask`:**

```cpp
// C++ — classic "iterate submasks" loop
for (int sub = mask; ; sub = (sub - 1) & mask) {
    // use sub
    if (sub == 0) break;
}
```
```java
for (int sub = mask; ; sub = (sub - 1) & mask) {
    // use sub
    if (sub == 0) break;
}
```
```python
sub = mask
while True:
    # use sub
    if sub == 0:
        break
    sub = (sub - 1) & mask
```
```javascript
for (let sub = mask; ; sub = (sub - 1) & mask) {
  // use sub
  if (sub === 0) break;
}
```

**Shift semantics — arithmetic vs logical, and width:**

| Language | `>>` on negative int | `>>>` / unsigned shift |
|---|---|---|
| C++ | implementation-defined pre-C++20 for signed types; arithmetic in practice on all mainstream compilers; C++20 mandates arithmetic | none built in — cast to unsigned type for logical shift |
| Java | `>>` is arithmetic (sign-extending); `>>>` is logical (zero-filling) — Java is one of the few languages with **both** | `>>>` |
| Python | ints are arbitrary-precision and always non-negative-after-shift in effect (no fixed-width sign bit) — `>>` on a negative int floors toward negative infinity, there's no separate logical variant | none needed |
| JavaScript | `>>` is arithmetic on the **32-bit signed** representation; `>>>` is logical (zero-filling), also 32-bit | `>>>` |

**Hard traps table — MASK WIDTH (this differs more than any other topic):**

| Language | Problem | Fix |
|---|---|---|
| C++ | `1 << 31` on `int` is **undefined behavior** (shifting into the sign bit of a signed type) | Use `1LL << 31` (or `1u << 31` if truly bit-pattern semantics are wanted) to force a wider/unsigned type |
| Java | `1 << 31` evaluates to `Integer.MIN_VALUE` (a **negative** int) — silently wrong if you expected a positive bit value | Use `1L << 31` to get a `long`, or work in `long` throughout when masks can exceed 30 bits |
| JavaScript | All bitwise operators (`&`, `\|`, `^`, `~`, `<<`, `>>`, `>>>`) coerce operands to **32-bit signed** integers first — any mask logic beyond bit 31 silently truncates/misbehaves, and `~x` flips into negative territory unexpectedly | For masks wider than 31 bits, switch to `BigInt` (`1n << 40n`) — note `BigInt` has **no** `>>>` operator and cannot mix with `Number` in the same expression without explicit conversion |
| Python | Ints are unbounded, so masks of any width just work — but `~x` is **not** a fixed-width flip, it's the mathematical `~x == -(x + 1)` (e.g. `~5 == -6`), which surprises anyone expecting a same-width complement | If a fixed-width complement is needed (e.g. simulate 32-bit `~x`), mask explicitly: `(~x) & 0xFFFFFFFF` |

> **Trap:** Because JS truncates every bitwise op to 32 bits, a DP or bitmask-subset problem with **more than 31 items** (state space `2^n` needing `n > 31`) cannot use native bitwise operators in JS at all — reach for `BigInt` (and its own, slower, non-32-bit-truncating operators) or redesign around multiple 32-bit words.

## 16. Numbers, Overflow and Math

**Integer max/min constants and overflow behaviour:**

| Language | Max/min constants | Overflow behaviour |
|---|---|---|
| C++ | `INT_MAX`/`INT_MIN` (`<climits>`), `INT64_MAX` or `numeric_limits<long long>::max()` | Signed overflow is **undefined behavior** — may wrap, may not, may be optimized away entirely by the compiler |
| Java | `Integer.MAX_VALUE`/`MIN_VALUE`, `Long.MAX_VALUE`/`MIN_VALUE` | Silent **wraparound** (two's complement), well-defined, no exception, no warning |
| Python | none needed — ints are arbitrary precision | **Never overflows**; grows memory as needed |
| JavaScript | `Number.MAX_SAFE_INTEGER` / `MIN_SAFE_INTEGER` (`±2^53 - 1`) | Beyond that range, doubles silently **lose precision** (not a crash, not a wrap — just wrong bits) |

**Big integers:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| No native bignum; `__int128` gives ~38 decimal digits (still fixed-width, GCC/Clang extension) or hand-roll/library for true bignum | `java.math.BigInteger` (immutable, full operator-style methods: `.add`, `.multiply`, `.mod`, `.modPow`) | Native — plain `int` is already arbitrary precision | `BigInt` (`123n` literal, or `BigInt(123)`) — cannot mix with `Number` in one expression |

**Modular multiply `(a * b) % m`:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| Widen before multiplying: `(long long)a * b % m;` (if `a,b` up to `1e9` and `m` up to `1e9`, product up to `~1e18` fits in `int64`; for products near `int64` overflow, use `__int128` or `__int128_t`) | Widen: `(long) a * b % m;` (product must fit in `long`; for very large `m` use `BigInteger` or `Math.multiplyHigh`) | `(a * b) % m` — just works, no widening needed | Product can exceed `2^53` and silently lose precision — use `BigInt`: `(BigInt(a) * BigInt(b)) % BigInt(m)`, then convert back with `Number(...)` if it fits |

**Modular exponentiation `a^b mod m`:**

```cpp
// C++ — hand-rolled fast exponentiation
long long modpow(long long a, long long b, long long m) {
    long long res = 1 % m;
    a %= m;
    while (b > 0) {
        if (b & 1) res = res * a % m;
        a = a * a % m;
        b >>= 1;
    }
    return res;
}
```
```java
// Java — BigInteger.modPow (preferred), or hand-roll identically to C++ using long
BigInteger.valueOf(a).modPow(BigInteger.valueOf(b), BigInteger.valueOf(m)).longValue();
```
```python
# Python — built in, three-argument pow (fast, and never overflows)
pow(a, b, m)
```
```javascript
// JavaScript — hand-roll with BigInt (no built-in modPow)
function modpow(a, b, m) {
  a = BigInt(a) % BigInt(m); b = BigInt(b); m = BigInt(m);
  let res = 1n;
  while (b > 0n) {
    if (b & 1n) res = (res * a) % m;
    a = (a * a) % m;
    b >>= 1n;
  }
  return res;
}
```

**Modular inverse (when `m` is prime, via Fermat's little theorem):**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `modpow(a, m-2, m)` (hand-rolled above) | `BigInteger.valueOf(a).modInverse(BigInteger.valueOf(m))` (works for any `m` coprime to `a`, not just prime) | `pow(a, m-2, m)` (prime `m`) or `pow(a, -1, m)` (3.8+, any coprime `m`) | `modpow(a, m - 2n, m)` (hand-rolled, `BigInt` version above) |

**gcd / lcm:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `std::gcd(a, b)`, `std::lcm(a, b)` (`<numeric>`, C++17) | no built-in — hand-roll Euclid: `while (b != 0) { int t = b; b = a % b; a = t; }` | `math.gcd(a, b)`; `math.lcm(a, b)` (3.9+) | no built-in — hand-roll: `function gcd(a,b){ return b === 0 ? a : gcd(b, a % b); }` |
| lcm formula (when not built in): | `a / gcd(a, b) * b` — divide **before** multiplying to reduce overflow risk |

**Integer square root (with off-by-one correction):**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `(long long)sqrtl(x)` then correct: `while (r*r > x) r--; while ((r+1)*(r+1) <= x) r++;` | `(long)Math.sqrt(x)` then apply the same correction loop | `math.isqrt(x)` (3.8+) — **exact**, no correction needed | `Math.floor(Math.sqrt(x))` then apply the same correction loop |

> **Trap:** `sqrt`/`Math.sqrt` are floating-point functions — for large `x` (beyond `2^52` or so) the double result can be off by one due to rounding, so **always** apply the correction loop above in C++/Java/JS before trusting an integer square root; Python's `math.isqrt` avoids this entirely by computing exactly with integers.

**nCr (binomial coefficient), typically with a modulus:**

```python
# Python — precompute factorials and modular inverses, then O(1) per query
MOD = 10**9 + 7
N = 200005
fact = [1] * N
for i in range(1, N):
    fact[i] = fact[i-1] * i % MOD
inv_fact = [1] * N
inv_fact[N-1] = pow(fact[N-1], MOD - 2, MOD)
for i in range(N - 2, -1, -1):
    inv_fact[i] = inv_fact[i+1] * (i+1) % MOD

def nCr(n, r):
    if r < 0 or r > n:
        return 0
    return fact[n] * inv_fact[r] % MOD * inv_fact[n-r] % MOD
```
Same precompute-factorials-and-inverse-factorials shape applies verbatim in C++ (`vector<long long>`) and Java (`long[]`), swapping `pow(x, MOD-2, MOD)` for the hand-rolled `modpow` from above. In JS, every multiplication in the precompute must go through `BigInt` or stay under `2^53` per step (safe here since `MOD ~ 1e9` and products are reduced every step, staying under `~1e18`... which **already exceeds** `2^53 ≈ 9e15`) — so JS nCr precompute **must** use `BigInt` throughout, not plain `Number`.

**Floating point comparison with epsilon:**

| C++ | Java | Python | JavaScript |
|---|---|---|---|
| `fabs(a - b) < 1e-9` | `Math.abs(a - b) < 1e-9` | `abs(a - b) < 1e-9` or `math.isclose(a, b)` | `Math.abs(a - b) < 1e-9` |

**Rounding:**

| Task | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Round to nearest int | `round(x)` (ties away from zero) | `Math.round(x)` (ties round **up**, i.e. toward `+inf`, even for negatives — differs from C++!) | `round(x)` (banker's rounding — ties to **even**) | `Math.round(x)` (ties round toward `+inf`, same quirk as Java) |
| Floor / ceil | `floor(x)`, `ceil(x)` | `Math.floor(x)`, `Math.ceil(x)` | `math.floor(x)`, `math.ceil(x)` | `Math.floor(x)`, `Math.ceil(x)` |

> **Trap:** The four languages round **ties differently** — C++ `round` goes away from zero, Java/JS `Math.round` goes toward `+infinity` (so `Math.round(-0.5)` is `-0`, not `-1`), and Python's `round` uses banker's rounding (round-half-to-even: `round(0.5) == 0`, `round(1.5) == 2`). Never assume a rounding result ports unchanged — check the tie case explicitly when translating.

**Summary trap table — the same arithmetic expression across languages:**

| Expression | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| `a * b` where `a, b ~ 1e9` (fits `int32` individually) | **overflows** `int` (UB) — must widen to `long long` | **overflows** `int`, silently wraps — must widen to `long` | always correct — arbitrary precision | loses precision silently past `2^53` if the true product exceeds it — must use `BigInt` for guaranteed correctness |
| `2^62` computed via repeated doubling | fine if using `long long`/`int64_t` from the start | fine if using `long` from the start | always fine | **silently wrong** — doubles cannot represent large integers exactly past `2^53` |
| Deeply nested modular arithmetic (`(a*b + c*d) % m`) | needs `long long` for every intermediate product | needs `long` for every intermediate product | just works | needs `BigInt` for every intermediate product once `m` and the operands are large |

The general rule when porting: an expression that is "just correct" in Python (and usually fine in C++/Java **only if you remembered to widen the type**) is the one most likely to silently misbehave in JavaScript, and the one most likely to **crash-via-UB or wrap-via-silent-overflow** in C++/Java if you forgot to widen.

---

*Doc 44 — Polyglot Rosetta Tables. Companion to doc 43 (plan), 45 and 46 (the 30 solutions), and 39–42 (deep per-language references).*
