# JavaScript for DSA & CP — Complete Revision

> Companion to your C++ STL file. Same style: every method = one line of code + a comment (what / when / gotcha / complexity).
> JS has **no STL**. This file shows how people who actually do DSA/CP in JS handle every structure — including the missing ones (heap, ordered map/set) which you **paste by hand** because judges rarely allow npm packages.
> Mental bridge from C++ is given for each section so your existing knowledge transfers.

---

## 0. The big picture — what JS gives you vs C++ STL

| You want (C++) | In JS you use | Important difference |
|---|---|---|
| `vector` | `Array` | dynamic, but it's also your stack/queue/deque — one type does all |
| `stack` | `Array` (`push`/`pop`) | no separate type |
| `queue` | `Array` (`push`/`shift`) | `shift` is **O(n)** — a real CP trap; use a pointer or deque class |
| `unordered_map` | `Map` or plain `{}` | **insertion-ordered**, not hashed-unordered; `Map` is the correct choice |
| `unordered_set` | `Set` | insertion-ordered |
| `map` (sorted keys) | ❌ none built-in | paste an ordered structure, or sort keys manually |
| `set` (sorted) | ❌ none built-in | same |
| `priority_queue` / heap | ❌ none built-in | **paste a MinHeap class** (section 8) — every JS CPer keeps one |
| `pair` / `tuple` | `[a, b]` array or `{a, b}` object | no dedicated type; arrays compare by reference, not lexicographically (gotcha) |

The single biggest adjustment: **JS `Map`/`Set` are NOT sorted.** They iterate in *insertion order*. There is no built-in balanced-BST / `lower_bound`. When a problem needs sorted-set behaviour, JS users either (a) collect into an array and sort, or (b) paste a structure. That's the defining constraint of CP in JS — internalise it now.

Second adjustment: **numbers are 64-bit floats.** Integer math is exact only up to 2^53 − 1 (`Number.MAX_SAFE_INTEGER`). Past that you silently lose precision — use `BigInt` (section 9). C++'s `long long` (up to ~9.2e18) has no direct safe equivalent in a normal `Number`.

---

## 1. Environment & fast I/O

You need this scaffolding before solving anything on a judge. Browser is for experimenting; Node is what judges (Codeforces, LeetCode-local, AtCoder) run.

```js
// ---------- NODE.JS: read all of stdin, then solve (the standard CP template) ----------
const data = require('fs').readFileSync('/dev/stdin', 'utf8');  // read ENTIRE input at once — far faster than line-by-line
const lines = data.split('\n');                                 // split into lines
let idx = 0;                                                    // a cursor you advance as you consume tokens
const nextLine = () => lines[idx++];                            // grab next line
const nums = () => lines[idx++].split(' ').map(Number);         // line -> array of numbers (e.g. "3 4 5")
const num  = () => Number(lines[idx++]);                        // single number line

const out = [];                          // BUFFER output — don't console.log in a loop (each call is slow I/O)
out.push(answer);                        // collect everything...
console.log(out.join('\n'));             // ...print ONCE at the end. This alone fixes many TLEs.

// ---------- parsing a token stream that ignores line boundaries ----------
const tok = data.split(/\s+/).filter(Boolean);  // all whitespace-separated tokens in one flat array
let p = 0;
const next = () => tok[p++];             // read next token regardless of newlines (robust for messy input)

// ---------- BROWSER / general JS ----------
console.log(x);                          // prints to devtools console
prompt("enter:");                        // blocking input dialog (toy use only)
// In the browser there is no stdin; you test by calling your function with literal arrays.
```

Why `readFileSync('/dev/stdin')` + buffered output: per-line reads and per-line `console.log` each carry syscall overhead; reading/writing in one shot is the difference between AC and TLE on large inputs. This is the #1 thing JS CPers get wrong first.

LeetCode note: you don't read stdin there — you implement a given function signature and `return` the answer. The I/O template above is for true competitive judges.

---

## 2. Array — the everything-container

JS `Array` is your `vector`, `stack`, `queue`, and `deque` rolled into one. Dynamically sized, holds mixed types (but keep them homogeneous for speed). O(1) index access and push/pop at the **end**.

```js
let a = [];                       // empty
let a = [1, 2, 3];                // literal
let a = new Array(5);             // length 5, all EMPTY holes (not 0!) — avoid, holes behave weirdly
let a = new Array(5).fill(0);     // length 5 of zeros — the correct "vector<int> v(5,0)"
let a = Array.from({length: n}, (_, i) => i);   // [0,1,...,n-1] — like iota
let g = Array.from({length: n}, () => new Array(m).fill(0)); // n*m grid — DON'T use .fill([]) (shares one row!)

// --- size & access ---
a.length;          // number of elements. ASSIGNABLE: a.length=2 truncates; a.length=0 clears.
a[i];              // index access, O(1). Out of range -> undefined (no error, silent — a trap).
a[a.length - 1];   // last element (no .back()).
a.at(-1);          // last element, negative-index friendly (ES2022). a.at(-2) = second last.

// --- end operations: O(1), use these for a STACK ---
a.push(x);         // append, returns new length. O(1). (stack push)
a.push(x, y, z);   // push multiple at once
a.pop();           // remove & RETURN last (unlike C++ pop which returns nothing!). O(1). (stack pop)

// --- front operations: O(n) because everything reindexes — the QUEUE TRAP ---
a.unshift(x);      // prepend. O(n) — shifts all elements. Avoid in loops.
a.shift();         // remove & return first. O(n). Using this as a queue dequeue => O(n²) total => TLE.
                   //   For a real queue use the pointer trick (section 7) or a deque class.

// --- middle / slicing ---
a.slice(i, j);     // COPY of [i, j) — does not modify a. Negative indices allowed. O(j-i).
a.splice(i, k);    // REMOVE k elements from index i, returns removed. MUTATES a. O(n).
a.splice(i, k, x); // remove k then insert x at i (insert/replace in the middle). O(n).
a.concat(b);       // returns new array a followed by b (doesn't mutate).
[...a, ...b];      // spread: same as concat, common modern style.

// --- search ---
a.indexOf(x);          // first index of x (=== match), or -1. O(n).
a.lastIndexOf(x);      // last index. O(n).
a.includes(x);         // boolean membership. O(n). (For frequent lookups use a Set instead — O(1).)
a.find(fn);            // first ELEMENT where fn(x) true, else undefined.
a.findIndex(fn);       // first INDEX where fn true, else -1.

// --- iterate (don't fight these; they're idiomatic and fast enough) ---
for (let i = 0; i < a.length; i++) {}   // classic; use when you need the index or to mutate
for (const x of a) {}                   // values (like range-based for) — preferred for read
a.forEach((x, i) => {});                // callback per element (can't break out of it — gotcha)

// --- transform (return NEW arrays; chainable) ---
a.map(x => x * 2);             // transform each -> new array (like std::transform)
a.filter(x => x > 0);          // keep matches -> new array (like copy_if)
a.reduce((acc, x) => acc + x, 0);   // fold to one value; '0' is init (like accumulate). 0 also fixes type.
a.some(fn);                    // any_of
a.every(fn);                   // all_of
a.flat();                      // flatten one level of nested arrays
a.flatMap(fn);                 // map then flat(1)

// --- ordering (MUTATE in place) ---
a.sort((x, y) => x - y);       // ASCENDING numeric. MUST pass comparator — default sort is LEXICOGRAPHIC!
a.sort((x, y) => y - x);       // descending
a.reverse();                   // reverse in place
a.fill(v);                     // set all elements to v
a.fill(v, i, j);               // fill [i, j) with v
```

The three array traps that bite C++ users:
- **`a.sort()` with no comparator sorts as STRINGS.** `[10, 2, 1].sort()` → `[1, 10, 2]`. Always pass `(x,y)=>x-y` for numbers.
- **`shift`/`unshift` are O(n).** A queue built on `push`/`shift` is O(n²) overall → TLE. Use section 7.
- **`pop` returns the element** (C++ `pop_back` returns nothing). And `a[i]` out of range gives `undefined`, not a crash — bugs stay silent.

---

## 3. String — immutable text

JS strings are **immutable** — every "modification" makes a new string. This is the biggest difference from C++ `std::string`. Building a string with `+=` in a loop is O(n²); push chars/parts into an **array** and `join('')` at the end.

```js
let s = "hello";
let s = 'hello';           // single/double quotes identical
let s = `val=${x}`;        // template literal: interpolation with ${}

// --- access & size ---
s.length;          // number of chars (property, not method)
s[i];              // char at i (read-only). s[i]='x' SILENTLY does nothing — strings are immutable.
s.charAt(i);       // same as s[i] but returns '' (not undefined) when out of range
s.charCodeAt(i);   // numeric code of char (e.g. 'a' -> 97). For frequency arrays: s.charCodeAt(i)-97.
String.fromCharCode(97);   // 'a' — reverse of charCodeAt

// --- search (return index or -1; or boolean) ---
s.indexOf("lo");        // first index, else -1
s.lastIndexOf("l");     // last index
s.includes("ell");      // boolean contains
s.startsWith("he"); s.endsWith("lo");   // boolean prefix/suffix
s.match(/[0-9]+/g);     // regex matches -> array or null
s.search(/l/);          // regex first-match index

// --- slice / extract (all return NEW strings) ---
s.slice(1, 3);          // [1,3) -> "el". Negative indices allowed: s.slice(-2) = last 2 chars.
s.substring(1, 3);      // similar but no negatives, and swaps args if start>end (prefer slice)
s.substr(i, len);       // start i, length len (legacy — avoid, but you'll see it)

// --- transform (NEW strings) ---
s.toUpperCase(); s.toLowerCase();
s.trim();               // strip leading/trailing whitespace (trimStart/trimEnd for one side)
s.replace("a", "b");    // replace FIRST occurrence (string) ...
s.replaceAll("a", "b"); // ... or ALL (ES2021). With /a/g regex, replace also does all.
s.repeat(3);            // "abc" -> "abcabcabc"
s.padStart(5, '0');     // left-pad to length 5 with '0' (e.g. "42" -> "00042")

// --- split / join: the bridge between strings and arrays ---
s.split('');            // -> array of chars (so you can sort/manipulate, then rejoin)
s.split(' ');           // -> array of words
s.split('\n');          // -> lines
arr.join('');           // array of chars/parts -> one string. USE THIS to build strings efficiently.

// --- the efficient "mutable string" pattern ---
let chars = s.split('');           // to a mutable array
chars[0] = 'H';                    // edit freely
chars.sort();                      // sort chars (anagram checks)
chars.reverse();                   // reverse
let result = chars.join('');       // back to a string — O(n) once, instead of O(n²) with +=

// --- number <-> string ---
Number("42"); parseInt("42", 10); parseFloat("3.14");   // string -> number (always pass radix 10 to parseInt)
(42).toString();  String(42);  `${42}`;                  // number -> string
(255).toString(2);   // -> "11111111" : convert to binary string (any base 2..36)
```

Key traps: strings are **immutable** (`s[i]='x'` does nothing; rebuild via array+join), and `+=` in a loop is quadratic. `parseInt("08")` is fine now but always pass the radix `10` to be safe. To count chars, an array indexed by `charCodeAt(i)-97` beats a Map for the 26 lowercase letters.

---

## 4. Map — the hash map (your unordered_map)

JS `Map` is the proper key→value structure: any type of key, **O(1) average** get/set, and it remembers **insertion order** when you iterate (NOT sorted — that's the difference from C++ `map`, and it has no `lower_bound`). Prefer it over plain `{}` objects for CP.

```js
let m = new Map();                       // empty
let m = new Map([["a", 1], ["b", 2]]);   // from array of [key,value] pairs

m.set(k, v);            // insert or overwrite. Returns the Map (so chainable: m.set(a,1).set(b,2)). O(1) avg.
m.get(k);               // value, or undefined if absent. O(1). Does NOT auto-insert (unlike C++ map[]).
m.has(k);               // boolean — the safe existence test (use before get to distinguish "0" from "absent")
m.delete(k);            // remove key, returns true if it existed
m.size;                 // number of entries (property, not a method — no parentheses)
m.clear();              // remove all

// frequency counting idiom (the JS equivalent of m[x]++):
m.set(x, (m.get(x) || 0) + 1);   // "|| 0" handles the first-time-undefined case

// iterate (in INSERTION order):
for (const [k, v] of m) {}       // destructure each entry — cleanest
for (const k of m.keys()) {}     // keys only
for (const v of m.values()) {}   // values only
[...m.keys()];                   // keys as an array (then sort if you need order!)
[...m.entries()];                // [[k,v],...] as an array

// to get SORTED-key behaviour (since Map won't): collect keys and sort
const sortedKeys = [...m.keys()].sort((a,b) => a - b);   // then iterate these manually
```

`Map` vs plain object `{}`: use `Map` when keys are non-strings (numbers, arrays-as-strings), when you insert/delete a lot, or need `.size` and ordered iteration. Objects coerce all keys to strings (`obj[1]` and `obj["1"]` collide) and carry prototype keys — subtle bugs. For pure speed with small integer keys, a plain array used as a map (`cnt[x]++`) beats both.

```js
// plain object as a quick map (keys become strings!):
let o = {};
o[k] = (o[k] || 0) + 1;      // frequency count
if (k in o) {}               // existence (also catches inherited keys — minor risk)
o.hasOwnProperty(k);         // safer existence test
delete o[k];                 // remove
Object.keys(o); Object.values(o); Object.entries(o);   // -> arrays
```

---

## 5. Set — the hash set (your unordered_set)

Unique values, **O(1) average** membership, **insertion-ordered** iteration, no sorting, no `lower_bound`.

```js
let s = new Set();               // empty
let s = new Set([1, 2, 2, 3]);   // from array -> {1,2,3} (dedupes automatically)

s.add(x);          // insert (no-op if present). Chainable. O(1) avg.
s.has(x);          // boolean membership. O(1) — THE reason to use Set over array.includes (which is O(n)).
s.delete(x);       // remove, returns true if existed
s.size;            // count (property)
s.clear();

for (const x of s) {}          // iterate in insertion order
[...s];                        // Set -> array (then sort if needed)
[...new Set(arr)];             // DEDUPE an array in one line — extremely common idiom

// set operations (no built-ins in older JS; do them manually or with ES2024 methods where available):
const inter = [...a].filter(x => b.has(x));   // intersection a ∩ b
const uni   = new Set([...a, ...b]);          // union
const diff  = [...a].filter(x => !b.has(x));  // difference a \ b
```

Use `Set` for visited-tracking in BFS/DFS, dedup, and any "have I seen this" check. For small integer ranges, a boolean/`Uint8Array` indexed by value is faster than a `Set`.

---

## 6. Stack — just an array

No dedicated type; an `Array` with `push`/`pop` IS a stack, and both are O(1). This part of JS is actually simpler than C++.

```js
let st = [];
st.push(x);            // push, O(1)
st.pop();              // pop AND returns top (C++ needs top() then pop() — JS gives it in one). O(1).
st[st.length - 1];     // peek without removing
st.at(-1);             // peek (ES2022)
st.length === 0;       // empty check
```

DFS, bracket matching, monotonic stack — identical logic to your C++ patterns, just `arr.push/pop`.

---

## 7. Queue & Deque — the O(n) shift trap and the fixes

There is **no built-in queue**. `arr.shift()` (dequeue) is **O(n)** because every element reindexes — a queue built that way is O(n²) and TLEs on big inputs. Two accepted fixes: a pointer-based queue (simplest) or a deque class (when you need both ends, e.g. 0-1 BFS / sliding window).

```js
// ---------- FIX 1: pointer queue — never shift(), just move a head index ----------
class Queue {
  constructor() { this.q = []; this.head = 0; }
  push(x) { this.q.push(x); }                  // enqueue, O(1)
  pop()   { return this.q[this.head++]; }      // dequeue, O(1) — advance head instead of shifting
  front() { return this.q[this.head]; }        // peek
  get size() { return this.q.length - this.head; }
  isEmpty() { return this.size === 0; }
}
// (memory grows since old slots aren't reclaimed; fine for one CP run. Reset by reslicing if paranoid.)

// quick inline version for BFS without a class:
const q = [start]; let h = 0;
while (h < q.length) { const u = q[h++]; /* push neighbours with q.push(...) */ }
//        ^ 'h < q.length' replaces !q.empty(); 'q[h++]' replaces front()+pop(). O(1) per node.

// ---------- FIX 2: Deque (double-ended), O(1) both ends — for 0-1 BFS, sliding-window-max ----------
class Deque {
  constructor() { this.map = new Map(); this.lo = 0; this.hi = 0; }  // index range [lo, hi)
  pushBack(x)  { this.map.set(this.hi++, x); }
  pushFront(x) { this.map.set(--this.lo, x); }
  popBack()    { const x = this.map.get(--this.hi); this.map.delete(this.hi); return x; }
  popFront()   { const x = this.map.get(this.lo); this.map.delete(this.lo++); return x; }
  front() { return this.map.get(this.lo); }
  back()  { return this.map.get(this.hi - 1); }
  get size() { return this.hi - this.lo; }
  isEmpty() { return this.size === 0; }
}
```

Rule: any BFS in JS must use the pointer queue or `h < q.length` loop, never `shift()`. This is the most common JS-CP TLE.

---

## 8. Heap / Priority Queue — paste this class (no built-in)

JS has **no `priority_queue`**. Every JS competitive programmer keeps a binary-heap class and pastes it in. Below is a complete MinHeap with a comparator (turn it into a max-heap by flipping the comparator). This is the single most important "missing piece" — memorise its interface.

```js
class Heap {
  // comparator(a,b) < 0 means a has HIGHER priority (comes out first).
  // default: min-heap (smallest out first). For max-heap pass (a,b)=>b-a.
  constructor(cmp = (a, b) => a - b) { this.h = []; this.cmp = cmp; }

  size() { return this.h.length; }
  isEmpty() { return this.h.length === 0; }
  peek() { return this.h[0]; }                       // smallest (or top priority). O(1). undefined if empty.

  push(x) {                                          // insert. O(log n).
    this.h.push(x);
    let i = this.h.length - 1;
    while (i > 0) {                                  // bubble up
      const p = (i - 1) >> 1;
      if (this.cmp(this.h[i], this.h[p]) < 0) { [this.h[i], this.h[p]] = [this.h[p], this.h[i]]; i = p; }
      else break;
    }
  }

  pop() {                                            // remove & return top. O(log n).
    const top = this.h[0], last = this.h.pop();
    if (this.h.length) {
      this.h[0] = last;
      let i = 0, n = this.h.length;                  // sift down
      while (true) {
        let l = 2*i+1, r = 2*i+2, best = i;
        if (l < n && this.cmp(this.h[l], this.h[best]) < 0) best = l;
        if (r < n && this.cmp(this.h[r], this.h[best]) < 0) best = r;
        if (best === i) break;
        [this.h[i], this.h[best]] = [this.h[best], this.h[i]]; i = best;
      }
    }
    return top;
  }
}

// usage:
const minpq = new Heap();                 // min-heap of numbers
const maxpq = new Heap((a, b) => b - a);  // max-heap
// Dijkstra: store [dist, node] and compare by dist:
const pq = new Heap((a, b) => a[0] - b[0]);
pq.push([0, src]);
while (!pq.isEmpty()) { const [d, u] = pq.pop(); /* relax edges, pq.push([nd, v]) */ }
```

This replaces `priority_queue`. Push/pop are O(log n), peek O(1) — same costs as C++. For pairs/structs, the comparator does what C++'s `greater<>`/custom comparator did.

---

## 9. Numbers, BigInt & math — the precision gap

JS `Number` is a 64-bit float. **Integers are exact only up to 2^53 − 1.** C++ `long long` (≈9.2e18) has no safe `Number` equivalent — past 2^53 you silently lose precision. Use `BigInt` when products/sums exceed that.

```js
Number.MAX_SAFE_INTEGER;       // 9007199254740991 (2^53 - 1) — above this, integer math LIES
Number.isSafeInteger(x);       // check before trusting integer arithmetic
Number.MAX_VALUE;              // ~1.8e308 (float ceiling, but not integer-exact)
Infinity; -Infinity;           // use as "no bound yet" sentinels (like INT_MAX). Math works with them.

// ---------- BigInt: arbitrary-precision integers (when you'd reach for long long / __int128) ----------
let big = 123n;                // the 'n' suffix makes a BigInt literal
let big = BigInt(x);           // from a number/string
big + 4n;  big * big;          // arithmetic — but you CANNOT mix BigInt and Number (3n + 4 throws!)
big % mod;                     // modular arithmetic works; great for "answer mod 1e9+7" problems
Number(big);                   // back to Number (only safe if it fits in 2^53)
// trade-off: BigInt is much slower than Number. Use only when values genuinely exceed 2^53.

// ---------- Math (your <cmath> / <algorithm> min-max) ----------
Math.floor(x); Math.ceil(x); Math.round(x); Math.trunc(x);  // trunc cuts toward zero (careful w/ negatives)
Math.abs(x); Math.sign(x);
Math.max(a, b, c);  Math.min(a, b, c);          // variadic — NOT array-aware directly:
Math.max(...arr);                               // spread an array in (max_element). Beware: huge arrays overflow the call stack — loop instead.
Math.pow(b, e);  b ** e;                         // power (** is the operator form)
Math.sqrt(x); Math.cbrt(x);
Math.log2(x); Math.log10(x); Math.log(x);        // log(x) is natural log
Math.hypot(x, y);                                // sqrt(x²+y²) without overflow

// integer division & gcd (no built-ins — write them):
const div = Math.floor(a / b);                   // integer division (for positives)
const gcd = (a, b) => b === 0 ? a : gcd(b, a % b);
const lcm = (a, b) => a / gcd(a, b) * b;

// bitwise ops are 32-BIT signed (a JS gotcha): (1 << 31) goes negative; >>> is unsigned shift.
a & b; a | b; a ^ b; ~a; a << k; a >> k; a >>> k;
```

The trap that costs hours: a sum/product exceeding 2^53 gives a wrong-but-no-error answer. If a problem says "answer fits in 64-bit" and you're not modding, reach for `BigInt`.

---

## 10. Sorting deep-dive — the comparator rules

Sorting in JS is one comparator function; getting it wrong is the most common silent WA. The comparator returns a **number**: negative → `a` first, positive → `b` first, 0 → keep order.

```js
arr.sort((a, b) => a - b);            // numbers ASCENDING. (default string sort would break this!)
arr.sort((a, b) => b - a);            // numbers descending
arr.sort();                           // ⚠ LEXICOGRAPHIC (string) order — only correct for strings
strs.sort();                          // strings ascending (default is fine for strings)
strs.sort((a, b) => a.localeCompare(b)); // locale-aware string sort

// sort objects / pairs by a field, then tie-break (the C++ "sort by first then second"):
pairs.sort((a, b) => a[0] - b[0] || a[1] - b[1]);   // by [0], ties broken by [1]. '||' chains keys.
items.sort((a, b) => a.cost - b.cost || a.id - b.id);

// stability: Array.prototype.sort IS stable (guaranteed since ES2019) — equal elements keep order.

// careful with strings-as-numbers and big values:
arr.sort((a, b) => a - b);            // fails for BigInt (a-b is BigInt, sort needs Number) -> use:
bigArr.sort((a, b) => (a < b ? -1 : a > b ? 1 : 0));   // comparison form, works for ANY comparable type
```

`arr.sort((a,b)=>a-b)` is the line you'll write most. The comparison form `(a<b?-1:a>b?1:0)` is the universal fallback that works for BigInt, strings, and mixed cases where subtraction is wrong.

---

## 11. Binary search — no built-in lower_bound

JS has no `lower_bound`/`upper_bound`/`binary_search`. Paste these. They're the backbone of any "search on sorted array" problem and the workaround for JS's missing ordered set.

```js
// lower_bound: first index with arr[i] >= target (or arr.length if none). Array must be SORTED.
function lowerBound(arr, target) {
  let lo = 0, hi = arr.length;                  // search in [lo, hi)
  while (lo < hi) {
    const mid = (lo + hi) >> 1;
    if (arr[mid] < target) lo = mid + 1;        // mid too small -> go right
    else hi = mid;                              // arr[mid] >= target -> candidate, shrink right bound
  }
  return lo;
}

// upper_bound: first index with arr[i] > target.
function upperBound(arr, target) {
  let lo = 0, hi = arr.length;
  while (lo < hi) {
    const mid = (lo + hi) >> 1;
    if (arr[mid] <= target) lo = mid + 1;       // only change vs lowerBound: <= instead of <
    else hi = mid;
  }
  return lo;
}

// uses:
const present = arr[lowerBound(arr, x)] === x;        // membership test (binary_search)
const countX  = upperBound(arr, x) - lowerBound(arr, x);   // how many equal x
const idx     = lowerBound(arr, x);                   // insertion position to keep sorted

// "binary search on the answer" — the CP pattern that needs no array, just a predicate:
function bsAnswer(lo, hi, ok) {        // smallest value in [lo,hi] where ok(value) is true
  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2);   // use Math.floor, not >>, if values can exceed 2^31
    if (ok(mid)) hi = mid; else lo = mid + 1;
  }
  return lo;
}
```

Since JS has no ordered set, **a sorted array + these two functions** is how you emulate `set::lower_bound`-style queries — as long as the data isn't changing mid-query (inserting into a sorted array is O(n)). For changing ordered data, you'd paste a balanced-BST/Fenwick/segment structure (beyond this core file; see section 13 pointers).

---

## 12. Pairs, tuples & the reference-equality trap

JS has no `pair`/`tuple`. You use arrays `[a, b]` or objects `{x, y}`. The trap: arrays/objects compare by **reference**, not value — and they don't auto-compare lexicographically the way C++ `pair` does.

```js
let p = [x, y];                  // "pair": access p[0], p[1]
let [a, b] = p;                  // destructure (like structured bindings)
let q = {first: x, second: y};   // object form, access q.first

// ⚠ arrays/objects are NOT value-equal:
[1,2] === [1,2];                 // FALSE — different references. Never compare arrays with ===.
JSON.stringify(a) === JSON.stringify(b);   // crude value-equality (works, slow; fine for small data)

// ⚠ can't use an array as a Map/Set key by value:
const seen = new Set();
seen.add([1,2]);  seen.has([1,2]);    // has() is FALSE — different reference each time!
// FIX: serialize the key:
seen.add(`${x},${y}`);  seen.has(`${x},${y}`);    // string key works by value
// or encode coordinates as one number when bounded:  key = x * COLS + y;

// sorting pairs lexicographically (C++ does this for free; JS needs the comparator from section 10):
pairs.sort((a, b) => a[0] - b[0] || a[1] - b[1]);
```

The visited-set-of-coordinates bug is the classic: `set.add([r,c])` never matches on lookup because each `[r,c]` is a new object. Always serialize (`` `${r},${c}` `` or `r*COLS+c`) when an array/pair must be a key.

---

## 13. What's genuinely missing & how JS CPers cope

JS's standard library stops short of C++'s. Honest summary of the gaps and the community-standard workarounds:

- **No ordered map / ordered set (balanced BST).** Workarounds: (a) sorted array + the binary-search functions in section 11 when data is static; (b) paste a Fenwick tree (BIT) for prefix sums / order-statistics, or a segment tree, when data changes; (c) on environments that allow it, the npm package `sorted-btree` or `js-sdsl` (the latter ports much of the STL: `OrderedMap`, `OrderedSet`, `PriorityQueue`, `Deque`). Most judges DON'T allow npm, so (a)/(b) are the real CP answer.
- **No priority_queue.** Paste the `Heap` class (section 8). (LeetCode's editor allows no imports either — paste it.)
- **No native int64.** Use `BigInt` (slower) when exceeding 2^53.
- **No O(1) front-pop.** Use the pointer queue / deque (section 7).
- **No multiset.** Emulate with a `Map<value, count>` (and binary-indexed structures if you need order).

Libraries by environment:
- **Codeforces / AtCoder / most judges:** no external packages — paste your own (heap, BIT, segtree, DSU). Keep a personal template file.
- **LeetCode:** function-signature style, no imports — paste classes inside or above your solution.
- **Node project / interview where install is allowed:** `npm i js-sdsl` gives `PriorityQueue`, `OrderedMap`, `OrderedSet`, `Deque` with near-STL APIs; `heap-js` for just heaps; `sorted-btree` for sorted maps.

```js
// Disjoint Set Union (Union-Find) — paste-ready, appears in countless CP problems:
class DSU {
  constructor(n) { this.p = Array.from({length: n}, (_, i) => i); this.r = new Array(n).fill(0); }
  find(x) { return this.p[x] === x ? x : (this.p[x] = this.find(this.p[x])); }  // path compression
  union(a, b) {
    a = this.find(a); b = this.find(b);
    if (a === b) return false;                       // already connected
    if (this.r[a] < this.r[b]) [a, b] = [b, a];      // union by rank
    this.p[b] = a; if (this.r[a] === this.r[b]) this.r[a]++;
    return true;
  }
}

// Fenwick / Binary Indexed Tree — prefix sums with point updates, both O(log n):
class BIT {
  constructor(n) { this.n = n; this.t = new Array(n + 1).fill(0); }
  update(i, v) { for (++i; i <= this.n; i += i & -i) this.t[i] += v; }   // add v at index i (0-based)
  query(i) { let s = 0; for (++i; i > 0; i -= i & -i) s += this.t[i]; return s; }  // prefix sum [0..i]
  range(l, r) { return this.query(r) - (l ? this.query(l - 1) : 0); }
}
```

Keep `Heap`, `Queue`/`Deque`, `DSU`, and `BIT` in a personal snippet file — those four cover the vast majority of what the C++ STL gave you for free.

---

## 14. The C++ → JS quick translation table

```
C++ STL                         JavaScript
------------------------------- --------------------------------------------------
vector<int> v;                  let v = [];
v.push_back(x); v.pop_back();    v.push(x); v.pop();           // pop RETURNS the value
v.size();  v.back();             v.length;  v.at(-1);
v[i];                            v[i]                          // out-of-range -> undefined, not UB
sort(v.begin(),v.end());         v.sort((a,b)=>a-b);           // MUST pass comparator for numbers
reverse(...)                     v.reverse();
*max_element(...)                Math.max(...v);               // (loop for very large v)
accumulate(...,0)                v.reduce((a,x)=>a+x,0);
unordered_map<K,V> m;            const m = new Map();
m[k]++;                          m.set(k,(m.get(k)||0)+1);
m.count(k);                      m.has(k);
m[k] (read)                      m.get(k)                      // get does NOT auto-insert
unordered_set<int> s;            const s = new Set();
s.insert(x); s.count(x);         s.add(x); s.has(x);
stack<int> st;                   let st = [];  st.push/pop
queue<int> q;                    pointer queue (section 7)     // NEVER arr.shift() in a loop
deque                            Deque class (section 7)
priority_queue                   Heap class (section 8)        // paste it
map<K,V> (sorted)                sorted array + lowerBound, or js-sdsl OrderedMap
set (sorted) / lower_bound       sorted array + lowerBound (section 11)
pair<int,int> p; p.first         let p=[a,b]; p[0]             // arrays compare by REFERENCE
long long                        Number up to 2^53, else BigInt
INT_MAX                          Infinity (or Number.MAX_SAFE_INTEGER)
next_permutation                 write your own (no built-in)
```

### Final JS-CP checklist (the mistakes that lose points)
- `arr.sort()` defaults to STRING order → always `(a,b)=>a-b` for numbers.
- `arr.shift()`/`unshift()` are O(n) → use a pointer queue for BFS, never shift in a loop.
- Building strings with `+=` in a loop is O(n²) → push into an array, `join('')` once.
- `Map`/`Set` are insertion-ordered, NOT sorted, and have no `lower_bound` → sort keys or paste a structure.
- Arrays/objects compare by reference → serialize coordinates (`` `${r},${c}` `` or `r*COLS+c`) for Set/Map keys.
- Integer math is exact only to 2^53 → use `BigInt` beyond that (and never mix BigInt with Number).
- Bitwise ops are 32-bit signed → `1<<31` is negative; use `>>>` for unsigned, or arithmetic for big shifts.
- Buffer output (`out.push(...)`, `console.log(out.join('\n'))`) — per-line `console.log` causes TLE.
- Read all stdin at once (`readFileSync('/dev/stdin')`) on judges, not line-by-line.
- `Array(n).fill([])` shares ONE inner array across rows → build grids with `Array.from`.
- `pop`/`shift` RETURN the element (unlike C++); `arr[i]` out of range is `undefined`, not a crash — bugs stay silent.
