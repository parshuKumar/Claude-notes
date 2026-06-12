# Java for DSA & CP — Complete Revision

> Companion to your C++ STL and JS files. Same style: every method = one line of code + a comment (what / when / gotcha / complexity).
> Good news: Java's **Collections Framework** maps closely to the C++ STL — `TreeMap`≈`map`, `HashMap`≈`unordered_map`, `PriorityQueue`≈`priority_queue`, `ArrayDeque`≈`deque`. You lose far less than in JS.
> Each section gives the C++ bridge so your existing knowledge transfers directly.

---

## 0. The big picture — Java collections vs C++ STL

| C++ STL | Java | Note |
|---|---|---|
| `vector<int>` | `ArrayList<Integer>` or `int[]` | use primitive `int[]` when size is fixed — far faster, no boxing |
| `stack` | `ArrayDeque` (as stack) | `java.util.Stack` exists but is legacy/slow — avoid |
| `queue` | `ArrayDeque` (as queue) | O(1) both ends |
| `deque` | `ArrayDeque` | the all-purpose double-ended container |
| `unordered_map` | `HashMap` | O(1) avg, unordered |
| `unordered_set` | `HashSet` | O(1) avg, unordered |
| `map` (sorted) | `TreeMap` | ✅ built-in red-black tree — JS lacks this entirely |
| `set` (sorted) | `TreeSet` | ✅ has `floor/ceiling/higher/lower` (≈ lower/upper_bound) |
| `multiset` | `TreeMap<K,Integer>` (value=count) | no direct multiset |
| `priority_queue` | `PriorityQueue` | ✅ built-in (min-heap by default — OPPOSITE of C++!) |
| `pair` | no built-in | use `int[]{a,b}`, `Map.Entry`, or a small record/class |
| `lower_bound`/`upper_bound` | `Arrays.binarySearch` / `TreeSet.ceiling` | semantics differ — see section 11 |

The three adjustments that matter most:

1. **Autoboxing.** Generic collections hold *objects* (`Integer`, not `int`). Every `int` ↔ `Integer` conversion costs time and memory. For tight loops / big N, prefer primitive `int[]`, `long[]`, `char[]` and primitive logic. This is the #1 Java-CP performance lever.

2. **`PriorityQueue` is a MIN-heap by default** — the *opposite* of C++'s max-heap default. `pq.poll()` gives the smallest. (Mnemonic: Java is "natural order = ascending = smallest first".)

3. **`==` vs `.equals()`.** `==` on objects compares *references*, not values. `Integer a = 1000, b = 1000; a == b` is **false**. Always use `.equals()` for object content, and beware the `Integer` cache (−128..127 are cached so small values *seem* to work with `==`, hiding the bug).

---

## 1. Environment & fast I/O

`Scanner` is convenient but **slow** — it TLEs on large inputs. Competitive Java uses `BufferedReader` for input and a `StringBuilder` + single `print` for output. Memorise this template.

```java
import java.util.*;
import java.io.*;

public class Main {
    public static void main(String[] args) throws IOException {
        // ---------- FAST INPUT ----------
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        int n = Integer.parseInt(br.readLine().trim());        // read a single int line
        StringTokenizer st = new StringTokenizer(br.readLine()); // split a line into tokens
        int a = Integer.parseInt(st.nextToken());              // pull next token as int
        long x = Long.parseLong(st.nextToken());               // as long
        // read an array on one line:
        int[] arr = new int[n];
        st = new StringTokenizer(br.readLine());
        for (int i = 0; i < n; i++) arr[i] = Integer.parseInt(st.nextToken());

        // ---------- FAST OUTPUT (buffer, print once) ----------
        StringBuilder sb = new StringBuilder();
        sb.append(answer).append('\n');     // collect everything...
        System.out.print(sb);               // ...print ONCE. Per-line System.out.println in a loop => TLE.
    }
}
```

```java
// Even faster input — a StreamTokenizer for pure-number problems:
StreamTokenizer in = new StreamTokenizer(new BufferedReader(new InputStreamReader(System.in)));
in.nextToken(); int v = (int) in.nval;     // reads next number into in.nval (a double)

// Scanner — ONLY for tiny inputs / quick tests (convenient but slow):
Scanner sc = new Scanner(System.in);
int k = sc.nextInt(); String line = sc.nextLine();   // nextInt leaves the newline -> a classic bug
```

The two TLE-killers: `BufferedReader` (not `Scanner`) for input, and `StringBuilder` buffered output (not `println` per line). LeetCode is different — you implement a method and `return`; no stdin needed.

---

## 2. Arrays — primitive `int[]` vs `ArrayList`

Java has two worlds: fixed-size primitive arrays (`int[]`, fast, no boxing) and the resizable `ArrayList<Integer>` (your `vector`, but boxed). Use `int[]` whenever size is known — it's the performance default in CP.

```java
// ---------- primitive arrays (fixed size, fast) ----------
int[] a = new int[n];                 // n zeros (primitives auto-init to 0 / false / null)
int[] a = {1, 2, 3};                  // literal
int[][] g = new int[n][m];            // n*m grid of zeros — one line, no shared-row trap (unlike JS)
long[] dp = new long[n];              // use long[] when values exceed ~2.1e9 (int overflow)

a.length;                             // size (a FIELD, no parentheses, no () — unlike .size())
a[i];                                 // access. Out of range -> ArrayIndexOutOfBoundsException (a real crash, not silent)
Arrays.fill(a, 7);                    // set every element to 7
Arrays.fill(a, l, r, 7);              // fill [l, r) with 7
int[] b = a.clone();                  // shallow copy (fine for primitives)
int[] b = Arrays.copyOf(a, newLen);   // copy, resized (pads with 0 or truncates)
int[] b = Arrays.copyOfRange(a, i, j);// copy of [i, j)
Arrays.sort(a);                       // ascending, O(n log n). For int[] it's a dual-pivot quicksort.
                                      //   ⚠ NO descending comparator for primitive arrays! (see gotcha)
int idx = Arrays.binarySearch(a, x);  // index of x if present (array MUST be sorted); else negative (see §11)
String s = Arrays.toString(a);        // "[1, 2, 3]" — for debugging
int[][] grid; Arrays.deepToString(grid); // for 2D debugging

// ---------- ArrayList (resizable, your vector — but boxed) ----------
List<Integer> list = new ArrayList<>();         // empty (program to the List interface)
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3));   // from values
List<Integer> list = new ArrayList<>(capacity); // pre-size to avoid regrowth (like reserve)

list.add(x);                 // append at end. O(1) amortised. (push_back)
list.add(i, x);              // insert at index i. O(n) — shifts right.
list.get(i);                 // access at i. O(1). (NOT list[i] — Java has no [] for collections)
list.set(i, x);              // overwrite index i. O(1).
list.remove(i);              // remove at INDEX i (int arg). O(n). ⚠ remove(Object) vs remove(int) trap below.
list.remove(Integer.valueOf(x)); // remove by VALUE x (must box, or it's treated as an index!)
list.size();                 // count (method, with parentheses — unlike array.length)
list.isEmpty();              // prefer over size()==0
list.contains(x);            // O(n) membership — use a Set for frequent lookups
list.indexOf(x);             // first index or -1
list.clear();
Collections.sort(list);                          // ascending
Collections.sort(list, Collections.reverseOrder());   // descending (works because it's boxed Integers)
Collections.sort(list, (p, q) -> p - q);         // custom comparator
Collections.reverse(list);
Collections.max(list); Collections.min(list);
int[] arr = list.stream().mapToInt(Integer::intValue).toArray();  // List<Integer> -> int[]

for (int x : list) {}        // enhanced for (range-based). Works on arrays and any Iterable.
for (int i = 0; i < list.size(); i++) {}   // indexed
```

The Java array traps that bite C++ users:
- **`Arrays.sort(int[])` has no descending option.** To sort descending: sort ascending then reverse, OR box to `Integer[]` and use a comparator, OR negate values. (Primitive arrays can't take a `Comparator`.)
- **`list.remove(5)` removes the element at INDEX 5**, not the value 5. To remove value 5: `list.remove(Integer.valueOf(5))`. Silent logic bug.
- `array.length` (field, no `()`) vs `list.size()` (method, with `()`) — don't mix them up.
- Out-of-range access **throws** (unlike JS's silent `undefined`) — at least the bug is loud.

---

## 3. Strings, StringBuilder & chars

Java `String` is **immutable** (like JS) — building with `+` in a loop is O(n²). Use `StringBuilder` for any incremental construction. Chars are a primitive type you can do arithmetic on.

```java
String s = "hello";
s.length();              // length (method, parentheses — unlike array.length field)
s.charAt(i);             // char at i. s[i] does NOT work in Java.
s.substring(i);          // from i to end (new string)
s.substring(i, j);       // [i, j) (new string)
s.indexOf("lo");         // first index, or -1
s.indexOf("l", from);    // search from index
s.lastIndexOf("l");      // last index
s.contains("ell");       // boolean
s.startsWith("he"); s.endsWith("lo");
s.equals(other);         // ⚠ VALUE equality — ALWAYS use this, NEVER == for strings
s.equalsIgnoreCase(o);
s.compareTo(o);          // <0, 0, >0 (lexicographic) — for sorting strings
s.toCharArray();         // -> char[] (mutable; sort/manipulate then rebuild)
s.toLowerCase(); s.toUpperCase();
s.trim(); s.strip();     // strip() is Unicode-aware (Java 11+)
s.replace('a', 'b');     // replace ALL occurrences of a char (or CharSequence)
s.split(" ");            // -> String[] (regex! "." or "|" must be escaped: split("\\."))
s.split("\\s+");         // split on runs of whitespace
String.join(",", list);  // join a collection/array with a separator
s.repeat(3);             // Java 11+
s.chars();               // IntStream of char codes (functional style)

// char arithmetic (chars are integers under the hood):
char c = s.charAt(i);
int idx = c - 'a';                 // 0..25 for lowercase -> frequency array index
char back = (char) ('a' + idx);    // int -> char (cast required)
Character.isDigit(c); Character.isLetter(c); Character.isUpperCase(c);
Character.toLowerCase(c); Character.getNumericValue(c);  // '7' -> 7

// ---------- StringBuilder: the mutable string (USE for building) ----------
StringBuilder sb = new StringBuilder();
sb.append(x);            // append anything (chars, ints, strings) — O(1) amortised
sb.append(a).append(b);  // chainable
sb.insert(i, x);         // insert at index i
sb.deleteCharAt(i);      // remove one char
sb.delete(i, j);         // remove [i, j)
sb.setCharAt(i, c);      // mutate one char in place
sb.charAt(i);            // read
sb.reverse();            // reverse in place (handy for big-number / palindrome tricks)
sb.length();
sb.toString();           // -> final String (do this ONCE at the end)

// number <-> string:
Integer.parseInt("42");  Long.parseLong("42");  Double.parseDouble("3.14");
Integer.parseInt("ff", 16);            // parse in base 16 -> 255
String.valueOf(42);  Integer.toString(42);  "" + 42;   // number -> string
Integer.toBinaryString(5);             // "101"
Integer.toString(255, 16);             // "ff" — to any base
```

Traps: `==` on strings compares references (interned literals may coincidentally match, hiding the bug) — always `.equals()`. `split` takes a **regex**, so `"a.b.c".split(".")` returns nothing useful — escape it as `split("\\.")`. Building strings with `+` in a loop is quadratic — use `StringBuilder`.

---

## 4. HashMap — the hash map (your unordered_map)

`HashMap<K,V>`: O(1) average get/put, **unordered**. The default key→value structure. Keys/values are objects (boxed).

```java
import java.util.*;

Map<String,Integer> m = new HashMap<>();      // program to the Map interface
Map<Integer,Integer> m = new HashMap<>();

m.put(k, v);                 // insert/overwrite. Returns previous value (or null). O(1) avg.
m.get(k);                    // value, or NULL if absent. O(1). ⚠ null can crash if you auto-unbox to int.
m.getOrDefault(k, 0);        // value or a default — USE THIS for frequency counting (avoids null).
m.containsKey(k);            // boolean — the safe existence test
m.containsValue(v);          // O(n) — rarely used
m.remove(k);                 // remove, returns old value
m.size(); m.isEmpty(); m.clear();

// frequency counting — the two idioms (equivalent to C++ m[x]++):
m.put(k, m.getOrDefault(k, 0) + 1);
m.merge(k, 1, Integer::sum);          // cleaner: if absent set 1, else add 1. (Java 8+)
m.putIfAbsent(k, v);                  // set only if key missing
m.computeIfAbsent(k, key -> new ArrayList<>()).add(x);  // adjacency list / grouping idiom — very common

// iterate (ORDER IS UNSPECIFIED — don't rely on it):
for (Map.Entry<String,Integer> e : m.entrySet()) {
    e.getKey(); e.getValue();         // access; e.setValue(x) to modify in place
}
for (var k : m.keySet()) {}           // keys only ('var' infers the type — Java 10+)
for (var v : m.values()) {}           // values only
m.forEach((k, v) -> {});              // functional iteration
```

Null trap: `int c = m.get(k)` throws `NullPointerException` if `k` is absent (auto-unboxing null) — use `getOrDefault`. For non-string keys, the key's class must have correct `hashCode()`/`equals()` (built-in types and records do; your own classes need them).

---

## 5. TreeMap — the SORTED map (your std::map) ⭐

This is what JS lacks entirely. `TreeMap` keeps keys **sorted** (red-black tree), all ops O(log n), and gives you `floor/ceiling/higher/lower` — Java's answer to `lower_bound`/`upper_bound`. Essential for ordered-data problems.

```java
TreeMap<Integer,Integer> tm = new TreeMap<>();              // ascending keys
TreeMap<Integer,Integer> tm = new TreeMap<>(Collections.reverseOrder());  // descending

tm.put(k, v); tm.get(k); tm.remove(k);     // same as HashMap but O(log n), keys stay sorted
tm.containsKey(k); tm.size();

// --- the ordered-navigation methods (THE reason to use TreeMap) ---
tm.firstKey();           // smallest key            O(log n)   (≈ *set.begin())
tm.lastKey();            // largest key                         (≈ *set.rbegin())
tm.ceilingKey(x);        // smallest key >= x   (≈ lower_bound)  — or null if none
tm.higherKey(x);         // smallest key >  x   (≈ upper_bound)  — strictly greater
tm.floorKey(x);          // largest key  <= x                    — or null
tm.lowerKey(x);          // largest key  <  x                    — strictly less
tm.firstEntry(); tm.lastEntry();           // the {key,value} pair, not just the key
tm.ceilingEntry(x); tm.floorEntry(x);      // entry versions
tm.pollFirstEntry();     // remove & return smallest (like a sorted-map "pop min")
tm.pollLastEntry();      // remove & return largest

// --- range views (live sub-maps, O(log n) to create) ---
tm.headMap(x);           // entries with key < x
tm.tailMap(x);           // entries with key >= x
tm.subMap(lo, hi);       // entries with lo <= key < hi
tm.keySet();             // keys in SORTED order
tm.descendingMap();      // a view in reverse order

for (var e : tm.entrySet()) {}   // iterates in SORTED key order
```

`ceilingKey`/`floorKey` returning `null` is how you detect "no such element" — always null-check before unboxing. This is your tool for "smallest element >= x in a changing set", coordinate compression with order, sweep-line problems.

---

## 6. HashSet & TreeSet — hash set and sorted set

`HashSet` = unordered, O(1) avg (your `unordered_set`). `TreeSet` = sorted, O(log n), with the same `floor/ceiling/higher/lower` navigation (your `set`).

```java
// ---------- HashSet: fast membership, unordered ----------
Set<Integer> hs = new HashSet<>();
Set<Integer> hs = new HashSet<>(Arrays.asList(1, 2, 3));
hs.add(x);               // insert (no-op if present), returns true if newly added. O(1) avg.
hs.contains(x);          // O(1) avg membership — the reason to use it over list.contains (O(n))
hs.remove(x);            // O(1) avg
hs.size(); hs.isEmpty(); hs.clear();
hs.addAll(other);        // union into hs
hs.retainAll(other);     // intersection (keep only elements also in other)
hs.removeAll(other);     // difference
for (int x : hs) {}      // unspecified order

// ---------- TreeSet: sorted, with navigation (your std::set) ⭐ ----------
TreeSet<Integer> ts = new TreeSet<>();
TreeSet<Integer> ts = new TreeSet<>(Collections.reverseOrder());
ts.add(x); ts.contains(x); ts.remove(x);   // O(log n), kept sorted
ts.first();              // smallest        (≈ *set.begin())
ts.last();               // largest         (≈ *set.rbegin())
ts.ceiling(x);           // smallest >= x   (≈ lower_bound) — null if none
ts.higher(x);            // smallest >  x   (≈ upper_bound)
ts.floor(x);             // largest  <= x   — null if none
ts.lower(x);             // largest  <  x
ts.pollFirst();          // remove & return smallest
ts.pollLast();           // remove & return largest
ts.headSet(x); ts.tailSet(x); ts.subSet(lo, hi);   // range views
ts.descendingSet();      // reverse-order view
```

`TreeSet` is how you do "ordered set with binary search on live data" — something JS can't do without a pasted structure. `ceiling`/`floor` are your `lower_bound`-style queries.

To emulate a **multiset** (Java has none): `TreeMap<Integer,Integer>` mapping value→count. Add: `map.merge(x,1,Integer::sum)`. Remove one: decrement, and `remove` the key when count hits 0. `firstKey`/`lastKey` give min/max.

---

## 7. ArrayDeque — stack, queue, AND deque

The one container for stack, queue, and deque. O(1) at both ends, no boxing of the structure itself (still boxes Integer elements). **Use this instead of the legacy `Stack` and `LinkedList`** — it's faster than both.

```java
Deque<Integer> dq = new ArrayDeque<>();

// --- as a STACK (LIFO) — use the *First methods ---
dq.push(x);              // = addFirst — push onto stack. O(1).
dq.pop();                // = removeFirst — pop, returns top. O(1). Throws if empty.
dq.peek();               // = peekFirst — look at top without removing (null if empty)

// --- as a QUEUE (FIFO) ---
dq.offer(x);             // = addLast — enqueue at back. O(1).
dq.poll();               // = removeFirst — dequeue from front, returns it (null if empty). O(1).
dq.peek();               // front element (null if empty)

// --- full deque (both ends explicitly) ---
dq.addFirst(x); dq.addLast(x);       // push to either end
dq.removeFirst(); dq.removeLast();   // pop from either end (throw if empty)
dq.peekFirst(); dq.peekLast();       // peek either end (null if empty)
dq.offerFirst(x); dq.offerLast(x);   // add, returns boolean instead of throwing

dq.size(); dq.isEmpty(); dq.clear();
for (int x : dq) {}                  // iterates front -> back
```

Two method families: the *throwing* ones (`addFirst`, `removeFirst`, `getFirst`) vs the *null/false-returning* ones (`offerFirst`, `pollFirst`, `peekFirst`). In CP, `offer`/`poll`/`peek` are safer (no exceptions). Use `ArrayDeque` for BFS (queue), DFS (stack), 0-1 BFS and sliding-window-max (deque). Never use the old `java.util.Stack` (synchronized, slow).

---

## 8. PriorityQueue — the heap (built in!) ⭐

Java has a built-in binary heap. **Default is a MIN-heap** (smallest out first) — the OPPOSITE of C++'s max-heap default. Burn that into memory.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();                      // MIN-heap (smallest first)
PriorityQueue<Integer> pq = new PriorityQueue<>(Collections.reverseOrder()); // MAX-heap
PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> b - a);       // MAX-heap via comparator

pq.offer(x);             // insert. O(log n). (also pq.add(x))
pq.poll();               // remove & return the TOP (min by default). O(log n). null if empty.
pq.peek();               // look at top without removing. O(1). null if empty.
pq.size(); pq.isEmpty(); pq.clear();
pq.remove(x);            // remove a specific value. O(n) — heap isn't built for arbitrary removal.

// custom objects / pairs (Dijkstra: {dist, node}, order by dist):
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);   // min by a[0]
pq.offer(new int[]{0, src});
while (!pq.isEmpty()) {
    int[] cur = pq.poll();
    int d = cur[0], u = cur[1];           // unpack
    // relax edges -> pq.offer(new int[]{nd, v});
}

// ⚠ comparator overflow trap: (a,b) -> a - b OVERFLOWS for large/negative ints.
// Safe for all int ranges:
PriorityQueue<Integer> safe = new PriorityQueue<>((a, b) -> Integer.compare(a, b));
PriorityQueue<int[]> safe2 = new PriorityQueue<>((a, b) -> Long.compare(a[0], b[0]));
```

The two things to remember: (1) **default is min-heap** (use `reverseOrder()` for max), and (2) `a - b` comparators **overflow** — use `Integer.compare`/`Long.compare` when values can be large or negative. Push/poll are O(log n), peek O(1), same as C++.

---

## 9. Numbers, overflow & math

Java integers are fixed-width and **overflow silently** (wrap around, no error) — exactly like C++, unlike JS. `int` is 32-bit (±2.1e9); `long` is 64-bit (±9.2e18). Pick `long` early when in doubt.

```java
int     // 32-bit signed, ~±2.1e9     -> overflows silently past Integer.MAX_VALUE
long    // 64-bit signed, ~±9.2e18    -> the CP default for sums/products that might exceed int
Integer.MAX_VALUE;  Integer.MIN_VALUE;   // ±2147483647
Long.MAX_VALUE;                          // use as "infinity" sentinel for long DP
Integer.MAX_VALUE;                       // "infinity" for int (careful: INF+INF overflows!)

long big = (long) a * b;     // ⚠ CAST BEFORE multiplying — (a*b) computes in int then overflows, THEN widens
long ok  = 1L * a * b;       // same fix: the 1L forces long arithmetic from the start

// math (java.lang.Math):
Math.abs(x); Math.max(a, b); Math.min(a, b);     // only TWO args — for arrays loop or stream
Math.pow(b, e);              // returns DOUBLE — for integer powers write fast-exponentiation, don't cast pow
Math.sqrt(x);  Math.cbrt(x);
Math.floorDiv(a, b);         // floor division (correct for negatives, unlike a/b which truncates toward 0)
Math.floorMod(a, b);         // modulo that's always non-negative — USE for "mod into [0,m)" problems
Math.ceil(x); Math.floor(x); Math.round(x);
Math.gcd?  // NONE — write your own:
long gcd(long a, long b) { return b == 0 ? a : gcd(b, a % b); }
long lcm(long a, long b) { return a / gcd(a, b) * b; }

// modular exponentiation (the standard CP utility — Java has no built-in):
long power(long base, long exp, long mod) {
    long res = 1; base %= mod;
    while (exp > 0) {
        if ((exp & 1) == 1) res = res * base % mod;
        base = base * base % mod;
        exp >>= 1;
    }
    return res;
}

// BigInteger / BigDecimal — when even long overflows (factorials, huge powers):
import java.math.BigInteger;
BigInteger a = BigInteger.valueOf(123);
BigInteger b = new BigInteger("99999999999999999999");
a.add(b); a.multiply(b); a.mod(m); a.pow(e); a.gcd(b);   // methods, not operators (no + for BigInteger)
a.compareTo(b);          // <0/0/>0
a.longValue();           // back to long (only if it fits)

// bitwise (32-bit int / 64-bit long):
a & b; a | b; a ^ b; ~a; a << k; a >> k; a >>> k;   // >>> is unsigned right shift
Integer.bitCount(x);     // number of set bits (popcount)
Long.bitCount(x);
Integer.highestOneBit(x); Integer.numberOfTrailingZeros(x);
```

The overflow trap that costs the most: `int a, b; long c = a * b;` computes `a*b` in **int** (overflows) then widens — wrong. Cast one operand first: `(long) a * b`. And `Math.pow` returns a double — never use it for exact integer powers (precision loss); write modular/fast exponentiation.

---

## 10. Sorting & comparators

`Arrays.sort` for primitives (no comparator allowed) and objects; `Collections.sort` for Lists. Comparators return `<0 / 0 / >0`. Java's object sort is **stable**; primitive sort is not (but values are indistinguishable, so it doesn't matter).

```java
// primitives — ascending only, no comparator:
Arrays.sort(arr);                          // int[]/long[]/char[] ascending, O(n log n)
Arrays.sort(arr, l, r);                    // sort sub-range [l, r)
// descending primitive: box to Integer[] + comparator, OR sort asc then reverse, OR negate.

// objects / boxed — comparators allowed:
Integer[] boxed = {3, 1, 2};
Arrays.sort(boxed, Collections.reverseOrder());          // descending
Arrays.sort(boxed, (a, b) -> a - b);                     // custom (mind overflow -> Integer.compare)
Arrays.sort(points, (a, b) -> a[0] != b[0] ? a[0] - b[0] : a[1] - b[1]);  // by [0] then [1]

Collections.sort(list);                                  // ascending
Collections.sort(list, Comparator.reverseOrder());
list.sort((a, b) -> a - b);                              // List has its own sort
list.sort(Comparator.comparingInt(o -> o.cost));         // by a field, ascending
list.sort(Comparator.comparingInt((int[] o) -> o[0])
                    .thenComparingInt(o -> o[1]));        // multi-key — the clean, overflow-safe way
list.sort(Comparator.comparingInt((P o) -> o.a).reversed());  // descending by field

// the overflow-safe comparator building blocks (PREFER these over a-b):
Comparator.comparingInt(o -> o.x);     // for int fields
Comparator.comparingLong(o -> o.x);    // for long fields
Comparator.comparingDouble(o -> o.x);
.thenComparing(...);                   // tie-breakers
.reversed();                           // flip order
```

Use `Comparator.comparingInt(...).thenComparingInt(...)` rather than hand-written `a-b` chains — it's readable and immune to subtraction overflow. Remember primitive `int[]` can't take a comparator at all.

---

## 11. Binary search — Arrays.binarySearch & the bound functions

`Arrays.binarySearch`/`Collections.binarySearch` exist but have awkward semantics (negative insertion point when absent). For `lower_bound`/`upper_bound` behaviour, either use `TreeSet.ceiling/higher` (live data) or write the functions (sorted arrays). Array must be sorted.

```java
// built-in (array MUST be sorted):
int r = Arrays.binarySearch(a, x);
// if found: r = index of x. if NOT found: r = -(insertionPoint) - 1  (a NEGATIVE number).
int insertionPoint = (r < 0) ? -(r + 1) : r;   // where x would go to stay sorted = lower_bound when absent

// hand-rolled lower/upper bound (clearer, the C++ equivalents):
static int lowerBound(int[] a, int target) {   // first index with a[i] >= target (or a.length)
    int lo = 0, hi = a.length;
    while (lo < hi) { int mid = (lo + hi) >>> 1; if (a[mid] < target) lo = mid + 1; else hi = mid; }
    return lo;
}
static int upperBound(int[] a, int target) {   // first index with a[i] > target
    int lo = 0, hi = a.length;
    while (lo < hi) { int mid = (lo + hi) >>> 1; if (a[mid] <= target) lo = mid + 1; else hi = mid; }
    return lo;
}
// (note >>> not >> for the midpoint — avoids overflow when lo+hi exceeds Integer.MAX_VALUE)

// for live ordered data use TreeSet instead of re-sorting:
ts.ceiling(x);   // lower_bound  (smallest >= x)
ts.higher(x);    // upper_bound  (smallest >  x)

// binary search on the answer (predicate-based, no array):
static int bsAnswer(int lo, int hi, java.util.function.IntPredicate ok) {
    while (lo < hi) { int mid = lo + (hi - lo) / 2; if (ok.test(mid)) hi = mid; else lo = mid + 1; }
    return lo;     // smallest value where ok() is true
}
```

Use `>>> 1` (unsigned shift) or `lo + (hi-lo)/2` for the midpoint to avoid the classic `(lo+hi)` overflow. For changing sorted data, `TreeSet`'s `ceiling`/`floor` beat re-sorting an array.

---

## 12. Pairs & custom objects

Java has no `pair`/`tuple`. Options: `int[]{a,b}` (fastest, mutable, but no type safety), `Map.Entry`, or a small `record` (Java 16+) / class. Choose by need.

```java
// int[] as a pair — the CP default (fast, works in PriorityQueue/sort with a comparator):
int[] p = {x, y};               // p[0], p[1]
pq.offer(new int[]{dist, node});

// Map.Entry as an immutable pair:
Map.Entry<Integer,Integer> e = Map.entry(x, y);   // Java 9+
e.getKey(); e.getValue();

// record — clean, immutable, auto equals/hashCode/toString (Java 16+; great as a HashMap key):
record Pair(int a, int b) {}
Pair pr = new Pair(1, 2);
pr.a(); pr.b();                 // accessors. Works correctly as a key in HashMap/HashSet (value equality!).

// classic class when you need mutability or methods:
class Node {
    int u, w;
    Node(int u, int w) { this.u = u; this.w = w; }
}

// using a pair as a HashSet/HashMap key by VALUE:
//   int[]   -> does NOT work (array equality is by reference) — encode instead: key = (long)r * COLS + c
//   record  -> works perfectly (auto value-based equals/hashCode)
//   String  -> works: r + "," + c
Set<Long> seen = new HashSet<>();
seen.add((long) r * COLS + c);     // encode 2D coordinate into one long — the fast, allocation-free way
```

Trap: `int[]` as a `HashSet`/`HashMap` key compares by **reference**, so lookups never match — same bug as JS arrays. Use a `record`, a `String` key, or encode coordinates into a single `long`.

---

## 13. Streams — concise, but know the cost

Java 8 streams give functional one-liners. Convenient and readable, but they **box and allocate** — for hot loops in CP, plain loops are faster. Use streams for setup/transform, not inner loops.

```java
import java.util.stream.*;

int[] a = IntStream.range(0, n).toArray();        // [0,1,...,n-1]  (like iota)
int[] a = IntStream.rangeClosed(1, n).toArray();  // [1..n]
int sum = Arrays.stream(arr).sum();               // sum of int[]
int max = Arrays.stream(arr).max().getAsInt();    // max (returns OptionalInt)
long s2 = Arrays.stream(arr).asLongStream().sum();// sum as long (avoid int overflow!)
int[] evens = Arrays.stream(arr).filter(x -> x % 2 == 0).toArray();
int[] doubled = Arrays.stream(arr).map(x -> x * 2).toArray();

List<Integer> list = Arrays.stream(arr).boxed().collect(Collectors.toList());  // int[] -> List<Integer>
int[] back = list.stream().mapToInt(Integer::intValue).toArray();              // List -> int[]
String joined = list.stream().map(String::valueOf).collect(Collectors.joining(","));

// grouping / counting (handy but allocation-heavy):
Map<Integer,Long> freq = Arrays.stream(arr).boxed()
    .collect(Collectors.groupingBy(x -> x, Collectors.counting()));
```

Rule: streams for readability in non-critical code; raw `for` loops with primitives for performance-critical sections. `Arrays.stream(arr).sum()` returns `int` — use `.asLongStream().sum()` to avoid overflow.

---

## 14. The C++ → Java quick translation table

```
C++ STL                          Java
-------------------------------- -----------------------------------------------
vector<int> v;                   List<Integer> v = new ArrayList<>();  OR int[] for fixed size
v.push_back(x); v.pop_back();     v.add(x);  v.remove(v.size()-1);
v[i];  v.size();                  v.get(i);  v.size()          // arrays: a[i], a.length
sort(v.begin(),v.end());          Collections.sort(v);  /  Arrays.sort(arr);
sort(...greater)                  Collections.sort(v, Collections.reverseOrder());
*max_element                      Collections.max(v) / Arrays.stream(arr).max().getAsInt()
accumulate(...,0)                 Arrays.stream(arr).sum()  (use asLongStream for long)
unordered_map<K,V>               HashMap<K,V>
m[k]++                           m.merge(k, 1, Integer::sum);
m.count(k)                       m.containsKey(k)
map<K,V> (sorted)                TreeMap<K,V>
m.lower_bound(x) (map)           tm.ceilingKey(x)            // smallest key >= x
m.upper_bound(x)                 tm.higherKey(x)
unordered_set<int>               HashSet<Integer>
set<int> (sorted)                TreeSet<Integer>
s.lower_bound(x) (set)           ts.ceiling(x);  upper -> ts.higher(x)
*s.begin() / *s.rbegin()         ts.first() / ts.last()
stack<int>                       ArrayDeque (push/pop)
queue<int>                       ArrayDeque (offer/poll)
deque<int>                       ArrayDeque (addFirst/addLast...)
priority_queue<int> (max!)       PriorityQueue<>(Collections.reverseOrder())  // Java default is MIN
priority_queue min-heap          new PriorityQueue<>()        // Java's default
pair<int,int>                    int[]{a,b} / record Pair(int a,int b) / Map.entry
long long                        long  (cast BEFORE multiply: (long)a*b)
__int128 / overflow              BigInteger
multiset                         TreeMap<K,Integer> value=count
INT_MAX                          Integer.MAX_VALUE / Long.MAX_VALUE
next_permutation                 write your own (no built-in)
```

```java
// next_permutation, since Java lacks it (in-place, returns false when it wraps to sorted):
static boolean nextPermutation(int[] a) {
    int i = a.length - 2;
    while (i >= 0 && a[i] >= a[i + 1]) i--;
    if (i < 0) return false;
    int j = a.length - 1;
    while (a[j] <= a[i]) j--;
    int t = a[i]; a[i] = a[j]; a[j] = t;            // swap pivot
    for (int l = i + 1, r = a.length - 1; l < r; l++, r--) { t = a[l]; a[l] = a[r]; a[r] = t; } // reverse tail
    return true;
}
```

### Final Java-CP checklist (the mistakes that lose points)
- Use `BufferedReader` + `StringBuilder`, never `Scanner`/per-line `println` (both TLE on big inputs).
- `int` overflows silently → use `long`; and cast BEFORE multiplying: `(long) a * b`, not `(long)(a * b)`.
- `PriorityQueue` is a MIN-heap by default — use `Collections.reverseOrder()` for max.
- Comparator `a - b` overflows for large/negative values → use `Integer.compare` / `Comparator.comparingInt`.
- `==` on objects (Integer, String) compares references → use `.equals()`. (Integer cache −128..127 hides this.)
- `m.get(k)` returns null when absent; unboxing null to `int` throws → use `getOrDefault`.
- `list.remove(int)` removes by INDEX; to remove a value use `remove(Integer.valueOf(x))`.
- `Arrays.sort(int[])` has no descending comparator → box, reverse, or negate.
- `split(".")` is a regex → escape as `split("\\.")`.
- `int[]` as a HashMap/HashSet key fails (reference equality) → use a `record`, `String`, or encode into a `long`.
- Prefer primitive `int[]`/`long[]` over `ArrayList<Integer>` in hot loops — autoboxing is the main Java slowdown.
- Midpoint `(lo+hi)/2` can overflow → use `(lo+hi) >>> 1` or `lo + (hi-lo)/2`.
