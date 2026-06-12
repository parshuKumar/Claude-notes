# C++ STL — Complete DSA & CP Revision (C++17)

> Your single source of truth for STL in Data Structures, Algorithms & Competitive Programming.
> Every function = one line of code + an explanatory comment (what it does, when to use it, the gotcha) + complexity.
> Read top-to-bottom to learn; scroll the code blocks to revise.

---

## 0. CP Boilerplate & Mental Model (read once)

```cpp
#include <bits/stdc++.h>      // pulls in the ENTIRE standard library. Allowed in CP, NOT in production.
using namespace std;          // so you write 'vector' not 'std::vector'. Fine in CP, risky in big code (name clashes).

int main() {
    ios_base::sync_with_stdio(false);  // unties C++ streams from C stdio -> cin/cout become much faster.
    cin.tie(NULL);                     // stops cin from flushing cout before every read -> faster I/O.
    // WARNING: after these two lines, do NOT mix cin/cout with scanf/printf, behaviour gets unreliable.
    return 0;
}
```

How to *think* about the STL — it has 3 pieces that snap together:
- **Containers** hold data (vector, set, map, ...). Each has its own memory layout & cost profile.
- **Iterators** are "generalised pointers" that point into containers. They are the glue.
- **Algorithms** (`sort`, `find`, ...) operate on *ranges* `[begin, end)` via iterators, so the SAME algorithm works on ANY container.

The single most important convention: every range is **half-open** `[first, last)` — `first` is included, `last` is one-past-the-end and is NOT touched. `v.end()` points *past* the last element, never *at* it. Dereferencing `end()` is undefined behaviour (a classic crash/WA source).

Choosing a container (the decision you make most in CP):

| Need | Use | Why |
|---|---|---|
| Dynamic array, index access | `vector` | contiguous, cache-friendly, O(1) index |
| Push/pop both ends | `deque` | O(1) at both ends |
| Sorted unique keys, ordered traversal | `set` / `map` | balanced BST, O(log n), keeps order |
| Just fast lookup, order irrelevant | `unordered_set/map` | hash table, O(1) average |
| LIFO | `stack` | adaptor over deque |
| FIFO | `queue` | adaptor over deque |
| Get max/min repeatedly | `priority_queue` | binary heap, O(log n) push/pop |

---

## 1. Iterators — the glue

An iterator points to an element in a container. Five categories, each more capable than the last; an algorithm states the *minimum* category it needs.

- **Input/Output**: single-pass, read-only / write-only (e.g. reading from cin).
- **Forward**: can move `++` and re-read (e.g. `forward_list`).
- **Bidirectional**: `++` and `--` (e.g. `list`, `set`, `map`).
- **Random-access**: `+ n`, `- n`, `it[k]`, `<` — jump anywhere in O(1) (e.g. `vector`, `array`, `deque`, raw arrays, `string`).

Why this matters in CP: `sort` needs **random-access** iterators — that's why you can `sort(v.begin(), v.end())` a vector but **cannot** sort a `set` or `list` with `std::sort` (they have their own `.sort()`).

```cpp
vector<int> v = {10, 20, 30};

v.begin();              // iterator to FIRST element (points at 10)
v.end();                // iterator to ONE PAST last (do NOT dereference) — marks the boundary
v.rbegin();             // REVERSE begin: points at last element (30) — ++ moves backwards
v.rend();               // reverse end: one-before-first boundary
v.cbegin(); v.cend();   // const versions: read-only, can't modify through them (use when you won't write)

auto it = v.begin();    // 'auto' deduces the messy iterator type for you — always use it for iterators
*it;                    // dereference -> the element value (10). Same syntax as pointers.
++it;                   // advance to next element. Prefer ++it over it++ (no temporary copy).
it->member;             // when elements are structs/pairs: shortcut for (*it).member

// Iterator helpers from <iterator> — work on ALL iterator types, even non-random-access:
advance(it, 5);         // move it forward by 5 (backward if negative). In-place, returns nothing.
next(it, 2);            // RETURNS a copy of it advanced by 2, without changing it. Default step = 1.
prev(it, 2);            // RETURNS a copy moved back by 2. Default step = 1.
distance(a, b);         // number of steps from a to b. O(1) for random-access, O(n) otherwise.

// Range-based for: the clean way to iterate when you don't need the index/iterator.
for (int x : v) { }          // x is a COPY of each element — modifying x does nothing to v.
for (int &x : v) x *= 2;     // reference -> modifies the container in place.
for (const auto &x : v) { }  // read-only, no copy — the idiomatic choice for heavy objects (strings, vectors).
```

Gotchas that cause crashes/WA:
- **Iterator invalidation**: inserting/erasing in a `vector` can move all elements (reallocation), making old iterators/pointers dangle. After `v.push_back`, assume all iterators are invalid. `list`/`set`/`map` only invalidate the iterator to the *erased* element.
- Comparing iterators from two different containers is undefined.
- `end()` is a boundary, not an element.

---

## 2. pair & tuple — bundling values

`pair` glues two values; `tuple` glues N. Everywhere in CP: coordinates, (distance,node) in Dijkstra, returning two results.

```cpp
#include <utility>     // pair lives here (also pulled by many headers)

pair<int,string> p;              // default: p.first=0, p.second=""
pair<int,int> p2 = {3, 4};       // brace init — most common
pair<int,int> p3(3, 4);          // ctor init — same thing
auto p4 = make_pair(3, 4);       // make_pair deduces types automatically (pre-C++17 convenience)

p.first;                         // access 1st element (it's a member, NOT a function — no parentheses)
p.second;                        // access 2nd element

p2 = {5, 6};                     // reassign whole pair at once
auto [a, b] = p2;                // C++17 structured binding: a=5, b=6. Cleanest way to unpack.

// Pairs compare LEXICOGRAPHICALLY: first by .first, ties broken by .second.
// THIS is why pair<int,int> works directly in set/priority_queue/sort with no custom comparator.
p2 < p3;                         // true if p2.first<p3.first, or (equal firsts AND p2.second<p3.second)
sort(vec_of_pairs.begin(), vec_of_pairs.end());  // sorts by first, then second — for free.

// tuple — when 2 isn't enough
tuple<int,string,double> t(1, "x", 2.5);
auto t2 = make_tuple(1, "x", 2.5);     // type-deducing factory
get<0>(t);                       // access by INDEX (must be a compile-time constant). get<1>(t) = "x".
get<1>(t) = "y";                 // get returns a reference, so you can assign through it.
auto [i, s, d] = t;              // C++17 unpack all three at once.
tie(i, ignore, d) = t;           // pre-C++17 unpack; 'ignore' discards a field you don't want.
tuple_size<decltype(t)>::value; // number of elements (3), at compile time.
```

CP idiom: store graph edges as `tuple<int,int,int>` = (weight, u, v); sorting a vector of these sorts by weight first → Kruskal's MST in two lines.

---

## 3. vector — the workhorse dynamic array

Contiguous memory (like a C array but resizable). O(1) random access and push_back; the default container for almost everything. "Contiguous" is why it's cache-friendly and fast in practice even when Big-O ties with other containers.

```cpp
#include <vector>

vector<int> v;                   // empty
vector<int> v(5);                // 5 elements, each value-initialised to 0
vector<int> v(5, 9);             // 5 elements, each = 9
vector<int> v = {1, 2, 3};       // from initializer list
vector<int> v2(v);               // copy of v
vector<int> v3(v.begin(), v.end()); // from an iterator range (e.g. copy part of another container)
vector<vector<int>> grid(n, vector<int>(m, 0)); // 2D n*m grid of 0 — THE CP idiom for matrices

// --- Access ---
v[i];           // element at index i. NO bounds check -> out-of-range is UB (fast, but a footgun).
v.at(i);        // element at i WITH bounds check -> throws out_of_range. Safer, slightly slower.
v.front();      // first element (== v[0]). UB if empty.
v.back();       // last element (== v[v.size()-1]). UB if empty. Used constantly with push/pop_back.
v.data();       // raw pointer to underlying array (for C APIs / pointer arithmetic).

// --- Size & capacity ---
v.size();       // number of elements currently held. Returns size_t (UNSIGNED — see gotcha below).
v.empty();      // true if size()==0. PREFER this over size()==0, it's clearer and always O(1).
v.capacity();   // allocated slots (>= size). Grows ~2x on reallocation.
v.reserve(n);   // pre-allocate capacity for n elements. HUGE in CP: avoids repeated reallocation if you know n.
v.shrink_to_fit(); // release unused capacity back. Rarely needed in CP.
v.resize(n);    // change size to n: shrinks (drops tail) or grows (new elems = 0).
v.resize(n, x); // grow with new elements = x instead of 0.

// --- Modifiers ---
v.push_back(x);          // append at end. O(1) amortised. The workhorse.
v.emplace_back(args...); // construct element in-place at end — avoids a copy/move. Prefer for heavy objects.
v.pop_back();            // remove last element. O(1). Does NOT return it — read v.back() first if needed.
v.insert(v.begin()+i, x);        // insert x BEFORE index i. O(n) — shifts everything right.
v.insert(v.end(), x);            // insert at end (same as push_back).
v.insert(pos, cnt, x);           // insert cnt copies of x.
v.insert(pos, first, last);      // insert a whole range.
v.erase(v.begin()+i);            // remove element at index i. O(n) — shifts left. Returns iterator to next.
v.erase(first, last);            // remove a [first,last) range. O(n).
v.clear();               // remove ALL elements. size becomes 0 (capacity may stay).
v.assign(n, x);          // replace contents with n copies of x.
v.assign(first, last);   // replace contents with a range.
swap(v1, v2);            // swap two vectors in O(1) (just swaps internal pointers). Or v1.swap(v2).

// --- Common CP patterns ---
sort(v.begin(), v.end());                    // ascending
sort(v.rbegin(), v.rend());                  // descending (sort the reverse range)
reverse(v.begin(), v.end());                 // reverse in place
v.erase(unique(v.begin(), v.end()), v.end());// REMOVE DUPLICATES — but ONLY after sorting first!
int mx = *max_element(v.begin(), v.end());   // max value (returns iterator, so dereference)
int sum = accumulate(v.begin(), v.end(), 0); // sum. The '0' is the start value AND fixes the type (use 0LL for long long!)
```

Gotchas (these cost real points):
- `v.size()` is **unsigned**. `for (int i = 0; i < v.size()-1; i++)` underflows to a huge number when `v` is empty → infinite loop / crash. Cast: `(int)v.size()`.
- `unique` only removes *consecutive* duplicates — you MUST `sort` first to dedupe fully.
- `accumulate(..., 0)` overflows for large sums because `0` is `int`; use `0LL`.
- `push_back` may reallocate and invalidate all iterators/pointers/references into the vector.
- `vector<bool>` is a special bit-packed type that does NOT behave like a normal vector (you can't take `&v[i]`); use `vector<char>` if you need real bools.

---

## 4. array — fixed-size, stack-allocated

`std::array<T,N>` — a thin wrapper over a C array with STL interface. Size fixed at compile time, lives on the stack (fast), knows its own size (unlike raw arrays).

```cpp
#include <array>

array<int,5> a = {1,2,3,4,5};    // size 5 fixed at COMPILE time (N must be a constant)
array<int,5> a2{};               // all zero-initialised

a[i];           // index access, no bounds check
a.at(i);        // bounds-checked
a.front(); a.back();             // first / last
a.size();       // always returns N (compile-time constant) — unlike C arrays which "decay" and lose size
a.fill(x);      // set every element to x
a.begin(); a.end();              // full iterator support -> works with all <algorithm> functions
sort(a.begin(), a.end());        // yes, you can sort it like a vector
```

When to use over vector: size truly known and fixed (e.g. `array<int,26>` for letter frequency) — slightly faster, no heap allocation.

---

## 5. deque — double-ended queue

Like a vector but **O(1) push/pop at BOTH ends**. Stored in chunks (not one contiguous block), so still O(1) random access but slightly slower than vector. It's the default engine behind `stack` and `queue`.

```cpp
#include <deque>

deque<int> dq;                   // empty
deque<int> dq(5, 0);             // 5 zeros

dq.push_back(x);    // append at end       O(1)
dq.push_front(x);   // prepend at front    O(1)  <- the thing vector CAN'T do cheaply
dq.pop_back();      // remove last         O(1)
dq.pop_front();     // remove first        O(1)
dq.front(); dq.back();           // peek both ends
dq[i]; dq.at(i);                 // random access — O(1) but not contiguous in memory
dq.size(); dq.empty();
dq.insert(pos, x); dq.erase(pos);// O(n) in the middle (like vector)
dq.clear();
```

CP use: sliding-window maximum (the "monotonic deque" pattern), 0-1 BFS (push_front for 0-weight edges, push_back for 1-weight), any time you need a queue you can also peek/pop from the front of.

---

## 6. list & forward_list — linked lists

`list` = doubly-linked, `forward_list` = singly-linked (lighter). O(1) insert/erase ANYWHERE *given an iterator*, but NO random access (no `[i]`) and poor cache behaviour. Rarely the best choice in CP — vector usually wins despite worse Big-O — but you must know `splice`.

```cpp
#include <list>

list<int> l = {1,2,3};
l.push_back(x); l.push_front(x);   // O(1) both ends
l.pop_back();   l.pop_front();     // O(1) both ends
l.front(); l.back();
l.insert(it, x);    // insert before iterator it — O(1) (no shifting, unlike vector!)
l.erase(it);        // erase at it — O(1). Returns iterator to next.
l.size(); l.empty(); l.clear();

// list-specific power tools (the reason list exists):
l.remove(val);          // remove ALL elements equal to val. O(n).
l.remove_if(pred);      // remove all elements where pred(x) is true.
l.unique();             // remove CONSECUTIVE duplicates (sort first to dedupe fully).
l.sort();               // sort IN PLACE — must use this, NOT std::sort (no random-access iterators).
l.reverse();            // reverse in place.
l.merge(l2);            // merge two SORTED lists into l, leaving l2 empty. O(n).
l.splice(it, l2);       // MOVE all of l2 into l before it, in O(1) — no copying. l2 becomes empty.

// forward_list: singly linked, even lighter. Insert is "after" since you can't go back.
#include <forward_list>
forward_list<int> fl = {1,2,3};
fl.push_front(x);              // ONLY front operations (no push_back, no size() in C++17!)
fl.insert_after(it, x);        // insert AFTER it
fl.erase_after(it);            // erase the element AFTER it
fl.before_begin();             // special iterator so you can insert_after at the very front
```

When list actually wins: you hold iterators to many elements and splice/move them between lists frequently (e.g. LRU-cache-style structures).

---

## 7. stack — LIFO adaptor

Not a real container — an **adaptor** that restricts a deque to Last-In-First-Out. You only touch the top. Used for: DFS (iterative), expression/bracket matching, monotonic stack, undo.

```cpp
#include <stack>

stack<int> st;          // LIFO, backed by deque by default
st.push(x);             // add on top                 O(1)
st.emplace(args);       // construct on top in place   O(1)
st.top();               // PEEK top element (does NOT remove). UB if empty — always check empty() first.
st.pop();               // remove top. Returns NOTHING — read top() before popping if you need the value.
st.size(); st.empty();
```

The two-step idiom you'll write 100 times: `int x = st.top(); st.pop();` — peek then remove, because `pop()` doesn't return.

---

## 8. queue — FIFO adaptor

Adaptor restricting a deque to First-In-First-Out. The backbone of BFS.

```cpp
#include <queue>

queue<int> q;
q.push(x);              // enqueue at back            O(1)
q.emplace(args);        // construct at back in place
q.front();              // PEEK front (next to leave). UB if empty.
q.back();               // peek the most recently added.
q.pop();                // remove front. Returns NOTHING.
q.size(); q.empty();
```

BFS idiom: `q.push(start);` then `while(!q.empty()){ int u=q.front(); q.pop(); /* process neighbours */ }`.

---

## 9. priority_queue — binary heap

Always gives you the **largest** element in O(log n) (max-heap by default). The tool for Dijkstra, Prim, "k largest", merging k lists, anything greedy-by-extremum.

```cpp
#include <queue>

priority_queue<int> pq;                  // MAX-heap: top() is the LARGEST element
pq.push(x);             // insert                     O(log n)
pq.emplace(args);       // construct in place          O(log n)
pq.top();               // peek the max (does NOT remove). O(1). UB if empty.
pq.pop();               // remove the max.             O(log n). Returns nothing.
pq.size(); pq.empty();

// MIN-heap (top = SMALLEST) — three equivalent ways, memorise the first:
priority_queue<int, vector<int>, greater<int>> minpq;   // the standard min-heap declaration
priority_queue<int> pq2;  pq2.push(-x);                  // hack: push negatives, negate on read
// negate-on-read works for ints only; use greater<> for correctness with pairs/structs.

// With pairs (e.g. Dijkstra: {distance, node}) — compares by .first then .second, like always:
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> dij;  // min-heap by distance

// Custom comparator via lambda (C++17):
auto cmp = [](int a, int b){ return a > b; };            // 'return a>b' => SMALLEST on top (min-heap)
priority_queue<int, vector<int>, decltype(cmp)> cpq(cmp);
```

The comparator confusion (read twice): the comparator defines "lower priority". `less<>` (default) → the *greatest* bubbles to top (max-heap). `greater<>` → the *least* bubbles to top (min-heap). It's the **opposite** of how `sort` reads. There is NO way to peek/erase anything but the top — if you need that, use a `set` instead.

---

## 10. set & multiset — sorted, balanced BST

Keeps elements **always sorted** and (for `set`) **unique**. Backed by a red-black tree. Everything is O(log n). Use when you need order maintained, ordered traversal, or `lower_bound`/`upper_bound` on a changing collection.

```cpp
#include <set>

set<int> s;                          // ascending, unique
set<int, greater<int>> s2;           // descending order
multiset<int> ms;                    // sorted but ALLOWS duplicates

s.insert(x);            // insert. O(log n). Returns pair<iterator,bool>: bool=false if already present.
s.emplace(args);        // construct in place.
s.erase(x);             // erase BY VALUE — removes the element. For multiset removes ALL copies of x!
s.erase(it);            // erase BY ITERATOR — removes exactly that one element (use this for multiset).
s.find(x);              // iterator to x, or s.end() if absent. O(log n).
s.count(x);             // set: 0 or 1. multiset: how many copies. O(log n + count).
s.contains(x);          // C++20 only — in C++17 use s.find(x)!=s.end().

// The CP superpower — binary search on an ordered set:
s.lower_bound(x);       // iterator to FIRST element >= x  (or end() if none). O(log n).
s.upper_bound(x);       // iterator to FIRST element >  x  (strictly greater). O(log n).
s.equal_range(x);       // pair{lower_bound, upper_bound} — the whole run of x's (handy in multiset).

s.begin();              // smallest element (front of order).
s.rbegin();             // largest element — *s.rbegin() is the max. O(1).
s.size(); s.empty(); s.clear();

// CRITICAL: use the MEMBER lower_bound, not std::lower_bound, on a set:
//   s.lower_bound(x)            -> O(log n)   ✓ uses the tree
//   lower_bound(s.begin(),s.end(),x) -> O(n)  ✗ std:: version can't exploit the tree, walks linearly
```

multiset deletion gotcha (very common bug): `ms.erase(x)` deletes **every** copy of x. To delete just ONE copy: `ms.erase(ms.find(x));`.

Custom ordering for structs:
```cpp
struct Cmp { bool operator()(const Node&a, const Node&b) const { return a.cost < b.cost; } };
set<Node, Cmp> ns;       // sorted by cost. The comparator must define a strict weak ordering.
```

---

## 11. unordered_set & unordered_multiset — hash table

Same interface as `set` but **no order**, backed by a hash table → O(1) **average** lookup/insert/erase (O(n) worst case on hash collisions). Use when you only need membership/lookup and don't care about order — it's faster than `set` for that.

```cpp
#include <unordered_set>

unordered_set<int> us;
us.insert(x);           // O(1) average, O(n) worst case
us.erase(x);            // O(1) average
us.find(x);             // O(1) average; us.find(x)!=us.end() to test membership
us.count(x);            // 0 or 1
us.size(); us.empty(); us.clear();
unordered_multiset<int> ums;     // duplicates allowed, still unordered

// NO lower_bound/upper_bound — there is no order to bound. If you need those, use set.

// Bucket interface (rarely needed, but explains performance):
us.bucket_count();      // number of buckets
us.load_factor();       // size / bucket_count — high load => more collisions => slower
us.reserve(n);          // pre-size for n elements -> fewer rehashes. Speed win if n is known.
us.max_load_factor(0.25); // force more buckets (less collision) at cost of memory.
```

CP DANGER — anti-hash tests: codeforces problemsetters craft inputs that force `unordered_map/set<int>` into O(n) per operation (everything collides) → guaranteed TLE. Defences: (1) use ordered `set`/`map` if the extra log n is affordable, or (2) add a custom hash with a random seed. Custom hash:
```cpp
struct Hsh {
    size_t operator()(uint64_t x) const {
        static const uint64_t R = chrono::steady_clock::now().time_since_epoch().count();
        x ^= R; x ^= x >> 33; x *= 0xff51afd7ed558ccdULL; x ^= x >> 33;  // splitmix-style mix
        return x;
    }
};
unordered_set<int, Hsh> safe_us;   // immune to crafted anti-hash inputs
```

---

## 12. map & multimap — sorted key→value

`map` = sorted, unique keys, each mapping to a value. Backed by a red-black tree, all ops O(log n). `multimap` allows duplicate keys. Use for: frequency-by-key with order, coordinate compression, anything needing sorted key iteration.

```cpp
#include <map>

map<string,int> m;               // keys sorted ascending, unique
map<int,int, greater<int>> m2;   // keys descending
multimap<int,int> mm;            // duplicate keys allowed

// --- THE two ways to insert/read, and the trap between them ---
m["apple"] = 5;         // operator[]: if key absent, CREATES it with value 0 then assigns. O(log n).
m["apple"]++;           // works because missing key auto-creates with 0 -> perfect for FREQUENCY COUNTING.
int x = m["banana"];    // TRAP: reading a MISSING key INSERTS it (value 0) and grows the map silently!
m.at("apple");          // bounds-checked read: throws if key absent, NEVER inserts. Use to read safely.
m.insert({"k", 9});     // insert pair; does NOTHING if key already exists (no overwrite).
m.emplace("k", 9);      // construct pair in place; also no overwrite.
m.insert_or_assign("k", 9); // C++17: insert OR overwrite existing — the "just set it" you usually want.
m.try_emplace("k", 9);  // C++17: insert only if absent, and doesn't construct the value if key exists.

// --- Lookup ---
m.find(key);            // iterator to the pair, or end() if absent. The safe "does it exist" check.
m.count(key);           // map: 0/1. multimap: number of entries with that key.
m.contains(key);        // C++20 only; in C++17 use m.find(key)!=m.end().
m.lower_bound(key);     // first entry with key >= given. O(log n). (member version — uses the tree.)
m.upper_bound(key);     // first entry with key > given.
m.equal_range(key);     // {lower,upper} — all entries with this key (the way to read multimap groups).

// --- Erase / iterate ---
m.erase(key);           // erase by key. multimap: erases ALL with that key.
m.erase(it);            // erase by iterator (one entry).
m.size(); m.empty(); m.clear();

for (auto &[k, v] : m)  // C++17 structured binding — iterate in SORTED key order. Cleanest form.
    cout << k << " " << v << "\n";
// older form: for (auto &p : m)  use p.first (key) and p.second (value).
```

The `[]`-inserts-on-read trap is one of the most common silent bugs: checking `if (m[key] == 0)` *adds* `key` to the map. To merely test existence use `m.count(key)` or `m.find(key)`.

---

## 13. unordered_map & unordered_multimap — hash map

Same interface as `map`, no ordering, O(1) average. The go-to for fast key→value when order is irrelevant. **All the unordered_set warnings apply**: no lower_bound/upper_bound, and it is anti-hash-test vulnerable in CP.

```cpp
#include <unordered_map>

unordered_map<int,int> um;
um[k]++;                // frequency counting in O(1) average — same []-creates-on-access behaviour & trap
um.at(k);               // bounds-checked, never inserts
um.find(k); um.count(k);
um.insert({k,v}); um.emplace(k,v);
um.insert_or_assign(k,v); um.try_emplace(k,v);   // C++17
um.erase(k); um.size(); um.empty(); um.clear();
um.reserve(n); um.max_load_factor(0.25);          // pre-size to avoid rehashing -> speed
unordered_multimap<int,int> umm;                  // duplicate keys, unordered

// Use the same random-seed custom hash from section 11 to defend against TLE attacks:
// unordered_map<int,int,Hsh> safe_um;
```

Rule of thumb for CP: reach for ordered `map`/`set` by default (predictable, immune to anti-hash); switch to `unordered_` only when O(log n) is too slow AND you've added a safe custom hash.

---

## 14. string — the text container

`std::string` is a full STL container of `char` with random access plus text-specific tools. Behaves like a `vector<char>` you can also search and slice.

```cpp
#include <string>

string s = "hello";
string s2(5, 'a');          // "aaaaa" — n copies of a char
string s3(s, 1, 3);         // substring of s: start index 1, length 3 -> "ell"

// --- Access / size ---
s[i];                       // char at i, no bounds check
s.at(i);                    // bounds-checked
s.front(); s.back();
s.size();  s.length();      // identical — number of chars
s.empty();

// --- Build / modify ---
s += "world";               // append string or char — O(amortised). The everyday concatenation.
s.push_back('!'); s.pop_back();
s.append("xyz");
s.insert(i, "abc");         // insert at index i
s.erase(i, len);            // erase len chars from index i
s.replace(i, len, "new");   // replace the [i, i+len) chunk with "new"
s.substr(i);                // substring from index i to end
s.substr(i, len);           // substring: length len from index i. (Copies — O(len).)
s.clear();
getline(cin, s);            // read a WHOLE LINE incl. spaces (cin >> s stops at whitespace!)

// --- Search (returns index, or string::npos if not found) ---
s.find("lo");               // first occurrence index, else string::npos
s.find("lo", pos);          // search starting at pos
s.rfind("l");               // LAST occurrence
s.find_first_of("aeiou");   // first index of ANY char in the set
if (s.find("x") != string::npos) { }   // the idiom: ALWAYS compare against npos, never < 0

// --- Conversions (CP lifesavers) ---
stoi(s);   stol(s);  stoll(s);   // string -> int / long / long long
stod(s);                         // string -> double
to_string(42);                   // number -> string
s.c_str();                       // C-style const char* (for C APIs / printf)

// --- Works with all algorithms (random-access iterators) ---
sort(s.begin(), s.end());                // sort characters (anagram checks)
reverse(s.begin(), s.end());             // reverse text
count(s.begin(), s.end(), 'a');          // count a specific char
transform(s.begin(), s.end(), s.begin(), ::tolower);  // lowercase the whole string in place
```

Gotchas: `cin >> s` stops at the first whitespace (reads one word) — use `getline` for full lines, and beware a leftover `'\n'` in the buffer after `cin >> x` (consume it with `cin.ignore()` before `getline`). `find` returns the unsigned `npos` on failure — compare with `!= string::npos`, never treat it as `-1` in unsigned context.

---

## 15. <algorithm> & <numeric> — the algorithms

The payoff of iterators: these work on ANY container's `[first, last)` range. Group by purpose. `(f,l)` below means `(v.begin(), v.end())`.

```cpp
#include <algorithm>
#include <numeric>

// ---------- SORTING & ORDERING ----------
sort(f, l);                      // ascending, O(n log n). Introsort (not stable).
sort(f, l, greater<int>());      // descending
sort(f, l, [](auto&a,auto&b){ return a.x < b.x; });  // custom: 'true' means a comes BEFORE b
stable_sort(f, l);               // keeps equal elements' relative order. O(n log n). Use when ties matter.
partial_sort(f, mid, l);         // sorts only enough to get the smallest (mid-f) elements at the front.
nth_element(f, f+k, l);          // puts the element that WOULD be at index k in sorted order, there.
                                 //   Everything left is <= it, right is >= it. O(n) average. (k-th smallest!)
is_sorted(f, l);                 // bool: already sorted?
reverse(f, l);                   // reverse the range in place.
rotate(f, mid, l);               // rotate so 'mid' becomes the new first element.

// ---------- BINARY SEARCH (range MUST already be sorted!) ----------
binary_search(f, l, x);          // bool: is x present? O(log n).
lower_bound(f, l, x);            // iterator to first element >= x. Subtract f for the INDEX.
upper_bound(f, l, x);            // iterator to first element >  x.
equal_range(f, l, x);            // {lower_bound, upper_bound} — the run of x's.
// index trick:  int idx = lower_bound(f, l, x) - v.begin();   // position where x is / would go
// count of x in sorted vector:  upper_bound(...) - lower_bound(...)

// ---------- MIN / MAX ----------
*min_element(f, l);              // smallest value (returns iterator -> dereference)
*max_element(f, l);              // largest value
minmax_element(f, l);            // pair{min_it, max_it} in one pass — cheaper than two calls
min(a, b); max(a, b);            // of two values
min({a,b,c}); max({a,b,c});      // of an initializer list — handy for 3+ values
clamp(x, lo, hi);                // C++17: x bounded into [lo,hi] — returns lo/hi if out of range

// ---------- NON-MODIFYING SCANS ----------
count(f, l, x);                  // how many equal x
count_if(f, l, pred);            // how many satisfy predicate
find(f, l, x);                   // iterator to first x, else l
find_if(f, l, pred);             // iterator to first match
all_of(f, l, pred);              // true if pred holds for ALL
any_of(f, l, pred);              // true if pred holds for ANY
none_of(f, l, pred);             // true if pred holds for NONE
for_each(f, l, fn);              // apply fn to each element

// ---------- MODIFYING ----------
fill(f, l, x);                   // set every element to x
iota(f, l, start);               // <numeric>: fill with start, start+1, start+2, ... (great for index arrays)
copy(f, l, dest);                // copy range to dest iterator
unique(f, l);                    // remove CONSECUTIVE dupes; returns new logical end. Sort first, then erase tail.
remove(f, l, x);                 // shifts non-x to front, returns new end (erase-remove idiom):
                                 //   v.erase(remove(f,l,x), v.end());   // truly delete all x
replace(f, l, old, new);         // replace all 'old' with 'new'
swap(a, b);                      // swap two values/containers

// ---------- PERMUTATIONS ----------
next_permutation(f, l);          // rearrange to NEXT lexicographic permutation; false when it wraps to sorted.
                                 //   do { ... } while(next_permutation(f,l));  // iterate ALL permutations
prev_permutation(f, l);          // previous permutation

// ---------- SET OPERATIONS (inputs MUST be sorted) ----------
set_union(f1,l1, f2,l2, dest);          // A ∪ B into dest
set_intersection(f1,l1, f2,l2, dest);   // A ∩ B
set_difference(f1,l1, f2,l2, dest);     // A \ B
merge(f1,l1, f2,l2, dest);              // merge two sorted ranges into one sorted range

// ---------- HEAP (turn a vector into a binary heap manually) ----------
make_heap(f, l);     // rearrange range into a max-heap. O(n).
push_heap(f, l);     // call AFTER push_back-ing the new element, to restore heap. O(log n).
pop_heap(f, l);      // moves max to the END (then you pop_back it). O(log n).
sort_heap(f, l);     // sort a heap range ascending. O(n log n).

// ---------- <numeric> NUMERIC TOOLS ----------
accumulate(f, l, 0);             // sum. USE 0LL for long long sums or you OVERFLOW!
accumulate(f, l, 1, multiplies<int>());   // product (or any binary op)
partial_sum(f, l, dest);         // running prefix sums into dest — prefix-sum arrays in one call
adjacent_difference(f, l, dest); // dest[i] = a[i]-a[i-1] — inverse of partial_sum
inner_product(f1, l1, f2, 0);    // dot product of two ranges
gcd(a, b);                       // C++17: greatest common divisor
lcm(a, b);                       // C++17: least common multiple
```

The three rules that cause the most algorithm bugs:
1. **Binary-search & set-ops need a SORTED range** — unsorted input gives silently wrong answers, not errors.
2. **`remove`/`unique` don't actually shrink the container** — they reorder and return a new "logical end"; you must `erase` from there to `end()` to really delete. (The erase-remove idiom.)
3. **`accumulate`'s init value sets the type** — `accumulate(f,l,0)` does int math even on a `long long` vector → overflow. Pass `0LL`.

---

## Master Complexity Cheat-Sheet

```
CONTAINER          access   insert/erase   search      ordered?  notes
vector             O(1)     O(n) mid/O(1)* O(n)         insertion  *push_back O(1) amortised
                                                                   contiguous, cache-friendly — fastest in practice
deque              O(1)     O(1) ends      O(n)         insertion  O(1) push/pop BOTH ends
list               O(n)     O(1) at iter   O(n)         insertion  splice O(1); poor cache
array              O(1)     —              O(n)         —          fixed size, stack
stack/queue        —        O(1)           —            —          top/front only
priority_queue     O(1) top O(log n)       —            by prio    heap; only the extremum is reachable
set/map            O(log n) O(log n)       O(log n)     SORTED     red-black tree; has lower/upper_bound
multiset/multimap  O(log n) O(log n)       O(log n)     SORTED     duplicates; erase(val) kills ALL copies
unordered_set/map  —        O(1) avg       O(1) avg     NO         hash; O(n) worst; anti-hash-test risk
string             O(1)     O(n) mid       O(n)         —          vector<char> + text tools

ALGORITHM                        complexity
sort / stable_sort               O(n log n)
nth_element                      O(n) average      <- k-th smallest without full sort
lower_bound/upper_bound/bin_srch O(log n)          <- on SORTED range only
make_heap                        O(n)
next_permutation                 O(n) per step
accumulate / count / find        O(n)
```

### Final CP checklist (the mistakes that lose points)
- `int` overflows at ~2.1e9 → use `long long` (and `0LL` in accumulate) whenever sums/products can exceed it.
- `v.size()` is unsigned → cast to `(int)` before subtracting, or you get a huge wraparound value.
- Sort before `unique`, `binary_search`, and all set operations.
- `map[key]` and `unordered_map[key]` INSERT on read → use `.count()`/`.find()` to test existence.
- `multiset/multimap erase(value)` removes ALL copies → `erase(find(value))` for just one.
- `pop()`/`pop_back()` return nothing → read `top()`/`back()`/`front()` first.
- Prefer member `s.lower_bound(x)` over `std::lower_bound(...)` on set/map (O(log n) vs O(n)).
- Default `unordered_*` is anti-hash-vulnerable on Codeforces → use ordered containers or a random-seed custom hash.
- Add fast I/O (`sync_with_stdio(false); cin.tie(NULL);`) and don't mix with scanf/printf afterwards.
