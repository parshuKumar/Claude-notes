# 43 — POLYGLOT SYNTAX TRANSFER — THE PLAN

## 1. The Premise

You are not here to learn algorithms. You have ~1600 LeetCode rating in C++ and every pattern in Docs 01-38 already lives in your head. What breaks you in a Java/Python/JS interview is not "how do I solve this" — it is "how do I *write* a hash map iteration / a comparator / a queue with both-end pops" without a 20-second stall while your brain transliterates from C++.

This curriculum exists to eliminate that stall. It is 30 LeetCode problems, chosen for **syntax coverage, not algorithmic novelty**. You have almost certainly already solved every one of these (or its twin) while working Patterns 01-38 in C++. That is deliberate: if the logic is already memorized, 100% of your working memory during practice goes to syntax, not to re-deriving the approach. You are not solving new problems — you are re-typing solved problems in unfamiliar clothing until the clothing stops being unfamiliar.

**Explicit non-goals:**
- Not learning a new algorithm or data structure concept.
- Not optimizing for "cleverest" solution — the boring, obvious solution is correct if it exercises the target syntax.
- Not building intuition for problem-solving — that intuition already exists.

**The coverage claim.** There are ~37 syntax surfaces that genuinely differ across C++ / Java / Python / JavaScript — things like "how do I create a 2D array without a shared-reference bug," "how does a min-heap vs max-heap default," "what's the null value called and how does it print." Section 2 enumerates them. The 30 problems in Section 3 hit **every single one of the 37 at least twice**, with ~2.9x average redundancy (108 total surface-hits / 37 surfaces) — so a mistake or a slow recall on your first exposure gets corrected on the second, or third.

**Reuse claim.** Cross-reference against Patterns 01-38: 27 of these 30 problems (90%) already appear somewhere in those pattern docs' problem sets. The 3 exceptions (My Calendar I, Serialize/Deserialize Binary Tree, Implement Trie) were added purely because no other problem in the set naturally drills ordered-map-floor/ceiling, tree-to-string serialization, or a trie's nested-map class design — they are syntax-motivated additions, not algorithm gaps.

**What "done" looks like.** After finishing all 30 in a target language, you should be able to open any LeetCode medium in that language and write a first-draft solution without opening a reference doc, even if the algorithm is unfamiliar in this curriculum — because the *scaffolding* (function signature, containers, iteration, string ops) is now reflexive.

**How to read this document.** Sections 1-4 are read once, before you start — they're the argument for why this works and in what order to run it. Sections 5-6 are the operating manual you glance at during the first few sessions until the loop is automatic. Section 7 you touch exactly twice per language (once to read it, once to sit it). Section 8 is a living document — open it every session, check boxes, log gaps. Section 9 is a lookup table for when you forget which doc has what.

---

## 2. The 37 Syntax Surfaces

Why 37, not a rounder number: this is an enumeration, not a target. It is every construct the author could identify, working problem-by-problem through Patterns 01-38, where the four languages genuinely diverge — not merely differ in keyword spelling (`push_back` vs `append` vs `push` is a spelling difference you learn in five minutes; the shared-reference trap in row 3 is a *semantic* difference that produces silently wrong output). If Doc 44's Rosetta tables surface a 38th construct during their own writing pass, treat it as an addendum to this list and re-check the coverage matrix in §3 rather than ignoring it.

| # | Surface | What differs across C++ / Java / Python / JS |
|---|---|---|
| 1 | Function signature / boilerplate | Free function vs `public static` in a class vs bare `def` vs arrow/function keyword; return type placement; no class wrapper needed in Py/JS, mandatory in Java |
| 2 | Dynamic array create/append/index/length | `vector<int> v; v.push_back(x); v.size()` vs `ArrayList<Integer>`/`.add()`/`.size()` vs `list.append()`/`len()` vs `[]`/`.push()`/`.length` |
| 3 | 2D array creation + shared-reference trap | `vector<vector<int>>(n, vector<int>(m))` is safe; `new int[n][m]` in Java is safe; `[[0]*m]*n` in Python **aliases every row**; `Array(n).fill(Array(m).fill(0))` in JS **aliases every row** too |
| 4 | Array fill/init | `fill(v.begin(), v.end(), x)` vs `Arrays.fill(a, x)` vs `[x]*n` vs `new Array(n).fill(x)` |
| 5 | Sort default | `sort()` on primitives ascending everywhere, but Python `sorted(list_of_tuples)` does lexicographic multi-key for free; JS `.sort()` **defaults to string/lexicographic**, `[10,2,1].sort()` → `[1,10,2]` |
| 6 | Sort with 1-key comparator | `sort(v.begin(), v.end(), cmp)` vs `Collections.sort(list, Comparator...)` / `list.sort(Comparator.comparingInt(...))` vs `list.sort(key=lambda x: ...)` vs `arr.sort((a,b) => a - b)` |
| 7 | Sort multi-key / by column | Comparator chaining `.thenComparing()` vs Python tuple keys `key=lambda x: (x[0], -x[1])` vs JS manual `(a,b) => a[0]-b[0] || b[1]-a[1]` |
| 8 | Hash map put/get-default/contains/iterate | `unordered_map`, `m[k]++` auto-inits vs `HashMap.getOrDefault/putIfAbsent` vs `dict.get(k,0)` / `defaultdict` vs `Map.get`/`has`/`Object` gotchas |
| 9 | Nested containers (map of list, map of set) | `unordered_map<int,vector<int>>` needs no init; Java needs `computeIfAbsent(k, x -> new ArrayList<>())`; Python `defaultdict(list)`; JS `Map` + manual `if (!map.has(k)) map.set(k, [])` |
| 10 | Hash set | `unordered_set` vs `HashSet<>` vs `set()` vs `new Set()` — add/contains/remove naming all differ |
| 11 | String index + char arithmetic | `s[i] - 'a'` works in C++/Java (chars are integer-like); Python has no char type, `ord(s[i]) - ord('a')`; JS `s.charCodeAt(i) - 97` |
| 12 | Efficient string building | `stringstream`/`+=` amortized in C++; Java `StringBuilder` mandatory (String is immutable, `+=` in a loop is O(n²)); Python `''.join(list)`; JS array `.join('')` or `+=` (V8 optimizes it, but join is idiomatic) |
| 13 | Split/join/trim/substring/reverse | `substr`/manual reverse in C++; Java `split`/`trim`/`substring`/`new StringBuilder(s).reverse()`; Python `split()`/`strip()`/slicing `s[::-1]`; JS `split`/`trim`/`slice`/`.split('').reverse().join('')` |
| 14 | Stack | `stack<T>`/push/pop/top vs `Deque<T>` as stack (never `Stack` class) vs `list.append/pop()` vs `[].push()/.pop()` |
| 15 | Queue/deque both ends | `deque<T>` push_front/back, pop_front/back vs Java `Deque` `offerFirst/Last`, `pollFirst/Last` vs Python `collections.deque` `appendleft/append`, `popleft/pop` vs JS **no built-in deque** — `.shift()/.unshift()` are O(n) |
| 16 | Heap min/max + comparator | `priority_queue` max-default, min needs `greater<>` vs Java `PriorityQueue` **min-default**, max needs reversed comparator vs Python `heapq` **min-only**, max via negation vs JS **no built-in heap at all** |
| 17 | Ordered map floor/ceiling | C++ `map`/`lower_bound`/`upper_bound` vs Java `TreeMap`/`floorKey`/`ceilingKey` vs Python **no ordered map**, use `sortedcontainers` or `bisect` on sorted list vs JS **nothing built-in**, roll your own or use a sorted array + binary search |
| 18 | Binary search built-in + hand-rolled + mid overflow | `lower_bound`/`upper_bound` vs Java `Arrays.binarySearch` (returns negative on miss) vs Python `bisect_left/right` vs JS **no built-in**, hand-roll always; mid-overflow (`(l+r)/2`) is a real bug in Java/C++, impossible in Python (bigints) and JS (doubles, but 2^53 cap) |
| 19 | Class/struct definition | `struct`/`class` with public fields vs Java mandatory class + getters or public fields vs Python `class` + `__init__` (or `@dataclass`) vs JS `class` or plain object literal |
| 20 | Custom object ordering | operator overload `bool operator<` vs Java `Comparable<T>`/`compareTo` vs Python `__lt__` or `functools.total_ordering` vs JS no operator overload, comparator function only |
| 21 | Linked list pointers + null | `Node* next = nullptr` vs Java `Node next = null` (reference, no explicit pointer syntax) vs Python `self.next = None` vs JS `this.next = null` |
| 22 | Tree recursion returning values | Return type must be declared (`int`/`TreeNode*`) in C++/Java; Python/JS return anything, type is a comment at best |
| 23 | Adjacency list construction | `vector<vector<int>>` or `unordered_map<int, vector<int>>` vs Java `List<List<Integer>>` with `computeIfAbsent` boilerplate vs Python `defaultdict(list)` one-liner vs JS `Map` + array-push boilerplate |
| 24 | Grid traversal + direction arrays + visited | `int dr[] = {0,0,1,-1}` vs Java `int[] dr = {...}` vs Python `dr = [0,0,1,-1]` (any type mix allowed) vs JS array of arrays for `[dr,dc]` pairs; visited as 2D bool array vs `Set` of encoded `r*cols+c` keys |
| 25 | Recursion + memoization container | `unordered_map<int,int> memo` passed by reference vs Java `HashMap` field or param vs Python `@lru_cache` decorator (zero boilerplate) vs JS `Map` passed through params or closure |
| 26 | 1D dp with infinity sentinel | `INT_MAX` vs Java `Integer.MAX_VALUE` vs Python `float('inf')` (no overflow, ever) vs JS `Infinity` |
| 27 | 2D dp table | `vector<vector<int>>(n+1, vector<int>(m+1, 0))` vs Java nested `new int[n+1][m+1]` (auto-zeroed) vs Python list-comprehension `[[0]*(m+1) for _ in range(n+1)]` (must avoid the `*` alias trap) vs JS nested `Array.from` |
| 28 | Backtracking path copy semantics | `res.push_back(path)` copies the vector in C++; Java `new ArrayList<>(path)` required or you get a mutated shared reference; Python `path[:]` or `list(path)` required; JS `[...path]` required — **forgetting the copy is the #1 backtracking bug in Java/Py/JS** |
| 29 | Bit ops XOR/shift/popcount/mask width | `__builtin_popcount` vs Java `Integer.bitCount` vs Python `bin(x).count('1')` (also arbitrary-width ints, no overflow) vs JS `>>>` unsigned shift needed because JS bitwise ops coerce to **32-bit signed** |
| 30 | Integer overflow + division + modulo semantics | C++/Java 32/64-bit wraparound is real; Python ints are arbitrary precision, no overflow; JS numbers are doubles (safe int only to 2^53) but `\|0`/bitwise forces 32-bit; negative modulo differs: C++/Java `-7 % 3 == -1`, Python `-7 % 3 == 2`, JS `-7 % 3 == -1` |
| 31 | Tuple/pair create/return/unpack/as-key | `pair<int,int>`/`{a,b}`/`.first/.second` vs Java **no tuple**, use array or a small class vs Python native `(a,b)` tuple, unpacking, and hashable-as-dict-key vs JS array `[a,b]` or object, not hashable as a Map key by value |
| 32 | Iterate with index | `for (int i = 0; i < n; i++)` everywhere in C++/Java vs Python `enumerate(arr)` idiom vs JS `for (const [i, v] of arr.entries())` or classic `for` |
| 33 | Container copy semantics on pass | C++ pass-by-value copies unless `&`; Java/JS objects and arrays are always passed by reference (mutations leak); Python same — everything is a reference except immutables |
| 34 | Null/None/undefined | `nullptr` vs Java `null` vs Python `None` (and falsy-but-not-null-check pitfalls) vs JS **two** absence values, `null` and `undefined`, plus `undefined == null` but `!== ` |
| 35 | Matrix in-place mutation + swap | `swap(a[i], a[j])` vs Java `int tmp = a[i]; a[i]=a[j]; a[j]=tmp;` (no built-in swap) vs Python `a[i], a[j] = a[j], a[i]` (native tuple-swap) vs JS `[a[i], a[j]] = [a[j], a[i]]` (destructuring swap) |
| 36 | Design class with internal state | Constructor + private fields + methods look almost identical across all four, but Java needs full getters/access modifiers ceremony; Python convention-only privacy (`_x`); JS `#x` true-private fields (ES2022) or convention |
| 37 | Generics / type declarations | C++ templates vs Java `<T>` generics + boxing (`Integer` vs `int`) traps vs Python fully dynamic, type hints optional and unenforced vs JS fully dynamic, no types at all (TypeScript not in scope here) |

---

## 3. The 30 Problems

Legend for the **Surfaces** column: numbers refer to §2. **Core-15** marks the 15 problems the user's block list flagged `(core)` — if you are short on time, these 15 alone still cover all 37 surfaces (verify yourself as a sanity exercise; the full matrix below uses all 30).

### Block 1 — Containers and hashing

*Focus: the constructs you'll type in the first 60 seconds of nearly every problem — hash map, dynamic array, tuple-as-key.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 1 | 1 | Two Sum | 1, 2, 8, 31, 32 | Yes | The single most common "first problem in a new language" — establishes function-signature reflex and map get/put |
| 2 | 49 | Group Anagrams | 1, 5, 8, 9, 11, 31 | Yes | Forces a map-of-list and a tuple/sorted-string-as-key, the two hardest "nested container" idioms |
| 3 | 347 | Top K Frequent Elements | 2, 6, 8, 16, 20 | Yes | First heap exposure, plus a custom-ordering decision (bucket vs comparator) |
| 4 | 3 | Longest Substring Without Repeating Characters | 10, 11, 32, 34 | No | Sliding window over a hash set — cements set add/contains/remove naming differences |

### Block 2 — Strings

*Focus: the operations C++ does with pointer arithmetic and every other language does with a named method — split, join, reverse, trim.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 5 | 125 | Valid Palindrome | 1, 11, 13, 34 | No | Char-arithmetic and two-pointer string scanning without a container's safety net |
| 6 | 151 | Reverse Words in a String | 2, 12, 13 | Yes | The canonical split/trim/join/reverse gauntlet — every language handles this differently |
| 7 | 344 | Reverse String | 2, 13, 35 | No | In-place mutation and swap on a char array — trivial logic, maximal syntax exposure |

### Block 3 — Linked list and class design

*Focus: null semantics and the ceremony (or lack of it) around defining a class with a constructor.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 8 | 206 | Reverse Linked List | 19, 21, 34 | Yes | Simplest possible node class + null-check reflex, in isolation from any other complexity |
| 9 | 21 | Merge Two Sorted Lists | 1, 19, 21 | No | Two node pointers moving in lockstep — a second, harder null-handling rep |
| 10 | 146 | LRU Cache | 8, 9, 19, 36, 37 | Yes | The heaviest single design problem in the set — full class-with-internal-state plus a map-of-nodes |

### Block 4 — Trees and recursion

*Focus: recursive functions that return a value, and what "no explicit return type" changes about how you write them.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 11 | 104 | Maximum Depth of Binary Tree | 1, 19, 22 | No | The smallest possible tree-recursion-returning-a-value rep — isolate this before adding complexity |
| 12 | 102 | Binary Tree Level Order Traversal | 2, 9, 15 | Yes | Queue-driven BFS — your first real deque exercise, plus building a list-of-lists result |
| 13 | 297 | Serialize and Deserialize Binary Tree | 12, 13, 14, 22, 34 | No | Round-trips a tree through a string — combines string-building with tree recursion, added specifically for this syntax combo |

### Block 5 — Graphs and grids

*Focus: adjacency construction and visited-tracking, where every language's "obvious" approach differs.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 14 | 200 | Number of Islands | 3, 14, 24 | Yes | The only pure grid problem in the set — direction arrays and a 2D visited structure |
| 15 | 207 | Course Schedule | 9, 10, 15, 23 | Yes | Adjacency-list-from-edge-list construction, the single most-repeated graph boilerplate in interviews |
| 16 | 133 | Clone Graph | 8, 19, 23, 25, 33, 34 | No | Recursion with a visited map doubling as a memo — heaviest reference-semantics rep in the set |

### Block 6 — Sorting / heap / search / ordered map

*Focus: min vs max defaults and the built-ins each language does or doesn't give you for free.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 17 | 56 | Merge Intervals | 2, 3, 5, 7 | Yes | Sort-by-column on an array of arrays — the multi-key sort rep every interview eventually needs |
| 18 | 973 | K Closest Points to Origin | 6, 7, 16, 20 | No | Heap with a custom comparator over compound objects — pairs naturally with #3's simpler heap use |
| 19 | 704 | Binary Search | 17, 18, 30 | Yes | Hand-rolled binary search forces the mid-overflow question in every language, even ones immune to it |
| 20 | 729 | My Calendar I | 17, 18, 19, 36 | No | Ordered-map floor/ceiling in its most natural form — added specifically because nothing else in the set needs it |

### Block 7 — DP and backtracking

*Focus: sentinel values for "infinity," table initialization, and the single most common backtracking bug (forgetting to copy the path).*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 21 | 70 | Climbing Stairs | 25, 26 | No | The simplest possible memoized recursion — establishes the memo-container idiom before DP gets harder |
| 22 | 322 | Coin Change | 4, 26, 27, 30 | Yes | Infinity-sentinel 1D dp, with a 2D alternate formulation available for extra reps |
| 23 | 72 | Edit Distance | 3, 4, 27 | Yes | The canonical 2D dp table — every language's table-init idiom, side by side |
| 24 | 139 | Word Break | 10, 13, 26, 28 | No | 1D dp with substring slicing at each step — a lighter backtracking-adjacent rep |
| 25 | 78 | Subsets | 2, 28, 33 | Yes | The purest backtracking-path-copy bug generator in the entire set — get this one exactly right |

### Block 8 — Bit / math / matrix / design

*Focus: the corner cases (overflow, 32-bit truncation, unsigned shifts) each language handles invisibly or not at all.*

| # | LC | Problem | Surfaces drilled | Core-15? | Why this problem |
|---|---|---|---|---|---|
| 26 | 136 | Single Number | 29 | No | XOR in isolation — smallest possible bit-op rep |
| 27 | 191 | Number of 1 Bits | 29 | Yes | Shift + popcount — the second bit-op rep, different operation from #26 |
| 28 | 50 | Pow(x, n) | 1, 29, 30 | No | Fast exponentiation mixes bit-shift logic with overflow/negative-exponent division semantics |
| 29 | 48 | Rotate Image | 3, 24, 35 | No | In-place matrix mutation with a transpose+reverse trick — swap syntax under real constraints |
| 30 | 208 | Implement Trie (Prefix Tree) | 9, 19, 36, 37 | No | A design class whose internal state is itself a nested container — closes the loop back to Block 1 and Block 3 |

### Coverage matrix — every surface, ≥2 hits, proven

| # | Surface | Covering problems (#) | Count |
|---|---|---|---|
| 1 | Function signature/boilerplate | 1, 2, 5, 9, 11, 28 | 6 |
| 2 | Dynamic array | 1, 3, 6, 7, 12, 17, 25 | 7 |
| 3 | 2D array + shared-ref trap | 14, 17, 23, 29 | 4 |
| 4 | Array fill/init | 22, 23 | 2 |
| 5 | Sort default | 2, 17 | 2 |
| 6 | Sort 1-key comparator | 3, 18 | 2 |
| 7 | Sort multi-key/by column | 17, 18 | 2 |
| 8 | Hash map | 1, 2, 3, 10, 16 | 5 |
| 9 | Nested containers | 2, 10, 12, 15, 30 | 5 |
| 10 | Hash set | 4, 15, 24 | 3 |
| 11 | String index + char arithmetic | 2, 4, 5 | 3 |
| 12 | Efficient string building | 6, 13 | 2 |
| 13 | Split/join/trim/substring/reverse | 5, 6, 7, 13, 24 | 5 |
| 14 | Stack | 13, 14 | 2 |
| 15 | Queue/deque both ends | 12, 15 | 2 |
| 16 | Heap min/max + comparator | 3, 18 | 2 |
| 17 | Ordered map floor/ceiling | 19, 20 | 2 |
| 18 | Binary search (built-in/hand-rolled/mid overflow) | 19, 20 | 2 |
| 19 | Class/struct definition | 8, 9, 10, 11, 16, 20, 30 | 7 |
| 20 | Custom object ordering | 3, 18 | 2 |
| 21 | Linked list pointers + null | 8, 9 | 2 |
| 22 | Tree recursion returning values | 11, 13 | 2 |
| 23 | Adjacency list construction | 15, 16 | 2 |
| 24 | Grid traversal + direction arrays + visited | 14, 29 | 2 |
| 25 | Recursion + memoization container | 16, 21 | 2 |
| 26 | 1D dp + infinity sentinel | 21, 22, 24 | 3 |
| 27 | 2D dp table | 22, 23 | 2 |
| 28 | Backtracking path copy semantics | 24, 25 | 2 |
| 29 | Bit ops (XOR/shift/popcount/mask) | 26, 27, 28 | 3 |
| 30 | Integer overflow/division/modulo | 19, 22, 28 | 3 |
| 31 | Tuple/pair create/return/unpack/as-key | 1, 2 | 2 |
| 32 | Iterate with index | 1, 4 | 2 |
| 33 | Container copy semantics on pass | 16, 25 | 2 |
| 34 | Null/None/undefined | 4, 5, 8, 13, 16 | 5 |
| 35 | Matrix in-place mutation + swap | 7, 29 | 2 |
| 36 | Design class with internal state | 10, 20, 30 | 3 |
| 37 | Generics/type declarations | 10, 30 | 2 |

**Total surface-hits: 108. Surfaces: 37. Average redundancy: 108 / 37 ≈ 2.9x.** Every row is ≥2 — the claim holds. Note the two deliberate cross-drills: #19 Binary Search and #20 My Calendar I each drill the *other's* primary surface (17 and 18) via their natural alternate implementations (sorted-list-with-binary-search vs balanced-ordered-map) — solving both problems means you've implicitly practiced both approaches for both problems.

**The Core-15, spelled out** (for when you're triaging time against a looming interview date): #1 Two Sum, #2 Group Anagrams, #3 Top K Frequent Elements, #6 Reverse Words in a String, #8 Reverse Linked List, #10 LRU Cache, #12 Level Order Traversal, #14 Number of Islands, #15 Course Schedule, #17 Merge Intervals, #19 Binary Search, #22 Coin Change, #23 Edit Distance, #25 Subsets, #27 Number of 1 Bits. These 15 alone touch 34 of the 37 surfaces at least once — everything except #22 (tree recursion returning values), #25 (recursion + memoization container), and #35 (matrix in-place mutation + swap), each of which lives only in a non-core problem (#11/#13, #16/#21, #7/#29 respectively). Translation: if you truly have no time, the core-15 gets you nearly all the way, but you must still tack on at least one of each missing pair (e.g. #11, #16, #7) to avoid a total blind spot. The full 30 is what gets every surface to ≥2x and closes the gap to the "never look anything up" bar in §1.

---

## 4. Language Order and Why

**Order: Python → Java → JavaScript.**

- **Python first.** Fastest to *write* under interview time pressure once syntax is loaded — least ceremony, native tuples/slicing/comprehensions do half your work. It is also the language most syntactically distant from C++, so it forces the biggest mental context-switch first, while you have the most energy. Highest speed ROI per hour invested.
- **Java second.** Verbose but mechanically closest to C++ (static types, explicit classes, compiled-ish mental model) — an easier second step after Python's distance. Also the dominant choice in Indian product-company interview loops: Amazon, Microsoft, and most Bangalore/Hyderabad/Pune unicorns explicitly allow or default to Java. Highest interview-frequency ROI.
- **JavaScript third, conditionally.** Only invest here if you're targeting frontend or full-stack roles where the interviewer expects JS. Otherwise Python + Java cover the overwhelming majority of DSA rounds you'll face.

**Rough interview-language expectation by role**, as a sanity check on whether to bother with JS at all:

| Role target | Typical DSA round language | This curriculum's relevance |
|---|---|---|
| Backend / SDE, Indian product companies (Amazon, Microsoft, Flipkart, etc.) | Java (default), Python (allowed) | Python + Java = full coverage |
| Backend / SDE, US-HQ or remote-first companies | Any of C++/Java/Python (candidate's choice) | Stick to C++, no polyglot drilling needed unless the JD says otherwise |
| Full-stack / frontend-leaning | JavaScript or TypeScript, occasionally candidate's choice | Add the JS pass |
| Early-stage startups | Often "language of your choice," sometimes explicitly the stack's language | Check the JD; default to Python if unspecified |

**Do one language fully before starting the next.** Interleaving destroys muscle memory: syntax recall is a motor-skill-like process, not a lookup table. Switching languages mid-formation means you build a blended, unstable model of "what a for-loop looks like" instead of a clean, fast, language-tagged reflex. Finish all 30 in Python, pass the exam, take a day off if needed, then start Java from problem 1 again.

**Relative verbosity and typing load**, ranked, so you can budget your practice hour accordingly — this is felt weight per problem, not a formal metric:

| Rank | Language | Typing load vs C++ | Why |
|---|---|---|---|
| 1 (lightest) | Python | ~0.5x | No types, no braces, no semicolons, comprehensions collapse 3-line loops into 1 |
| 2 | JavaScript | ~0.7x | No types, but still braces/semicolons and more manual container boilerplate than Python |
| 3 | C++ (anchor) | 1x | Baseline — your fluent language |
| 4 (heaviest) | Java | ~1.4x | Types on every declaration, mandatory class wrapper, generic diamond operators, checked-exception ceremony on some I/O paths |

This is also why Python is first (fastest feedback loop while still building recall) and Java is deliberately budgeted more wall-clock time per problem in §6 even though the algorithm is identical.

**Per-language "what will bite you coming from C++":**

| Language | Gotcha | Detail |
|---|---|---|
| Python | `//` vs `/` | `//` is floor division, not comment. `/` always returns float. |
| Python | `%` on negatives | `-7 % 3 == 2`, not `-1` — different sign convention than C++ |
| Python | No overflow | Never worry about int wraparound — but also means LeetCode "overflow" tricks (e.g. Pow, bit masks) need explicit masking if you want to simulate 32-bit behavior |
| Python | `heapq` is min-only | Max-heap needs negation or a wrapper — no `greater<>` equivalent |
| Python | Recursion limit | Default ~1000; deep recursion (e.g. skewed tree, long linked list) needs `sys.setrecursionlimit` |
| Python | Mutable default args | `def f(x, memo={})` shares `memo` across calls — a classic silent bug |
| Java | Verbosity | Every variable typed, every class boilerplate — budget 2x the typing time of Python |
| Java | Boxing | `Integer` vs `int`; `==` on boxed `Integer` compares references above value -128..127 — use `.equals()` or unbox |
| Java | `PriorityQueue` is min-default | Opposite instinct from C++ `priority_queue` (max-default) — easy to get backwards under pressure |
| Java | No unsigned types | No `unsigned int`; bit tricks relying on unsigned shift need `>>>` |
| Java | `String` immutability | `+=` in a loop is O(n²) — must use `StringBuilder` |
| JavaScript | `.sort()` default is lexicographic | `[10, 2, 1].sort()` → `[1, 10, 2]` — always pass a comparator for numbers |
| JavaScript | No heap, no TreeMap | Nothing built-in for either — hand-roll a binary heap array or fake with sorted array + binary search |
| JavaScript | Bitwise ops are 32-bit signed | `x >> 1` behaves like Java int shift, but numbers themselves are doubles — mixing these mental models causes bugs |
| JavaScript | 2^53 safe-integer ceiling | Silent precision loss above `Number.MAX_SAFE_INTEGER` — matters for any problem with large sums or `10^18`-scale constraints |
| JavaScript | `null` vs `undefined` | Two absence values; `==` treats them equal, `===` doesn't; uninitialized object fields are `undefined`, not `null` |

---

## 5. The Practice Protocol

The loop, per problem, in the **target** language (say Python, on your first pass):

1. **Write the C++ solution from memory first, ~2 minutes, as the anchor.**
   Why: this confirms the algorithm is genuinely automatic before you spend effort on syntax, and gives you a concrete reference to translate *from* rather than trying to recall algorithm + syntax simultaneously. If you can't produce the C++ in 2 minutes, stop — the problem isn't syntax-ready yet, go re-read the relevant Pattern doc.

2. **Translate line by line into the target, with references CLOSED.**
   Why: the entire point is forcing recall, not recognition. Recognition (reading Doc 44 and nodding) does not build muscle memory; production under a blank page does. Close Docs 39-46 and the browser tab.

3. **Open Doc 44 only when stuck, only for that construct.**
   Why: this is not "no reference ever" — it's "reference is a scalpel, not a crutch." Look up exactly the one Rosetta row you need (e.g. "Python heap max-heap idiom"), close it again, keep writing. Never read ahead in Doc 44 "just to see what's coming."

4. **Submit on LeetCode until Accepted.**
   Why: compiling and passing hidden test cases is the actual test — not "does this look right to me." Syntax bugs (off-by-one from 0-indexed slicing, a wrong default sort, a mutated shared list) are exactly what LeetCode's judge catches and your eyes don't.

5. **Log every lookup into your personal gap list** (template in §8).
   Why: a lookup you don't log is a lookup you'll make again in an interview. The gap list turns vague unease ("I never remember X") into a ranked, attackable list — review it before your final exam.

6. **Redo the problem cold, 3 days later, no notes, timed.**
   Why: this is the actual spaced-repetition test of whether the syntax stuck or whether you just pattern-matched your way through step 2. If a lookup from your gap list resurfaces on the redo, that construct isn't learned yet — drill it in isolation (write it 5 times in a scratch file) before moving on.

**Hard rule: never copy-paste from Doc 44, 45, or 46. Type every character.** Copy-paste produces zero motor memory. The value of this entire curriculum collapses if you paste.

**Common failure modes per step, and the fix:**

| Step | Failure mode | Fix |
|---|---|---|
| 1 | You can't produce the C++ in 2 minutes | The problem is a Patterns 01-38 gap, not a syntax gap — go drill the algorithm first, come back later |
| 2 | You keep peeking at Doc 44 before even trying to recall | You're building a recognition habit, not a recall habit — force yourself to write *something*, even if wrong, before opening anything |
| 3 | You read three rows of Doc 44 "while you're in there" | Scope creep turns a scalpel into a crutch — cover the rest of the page with your hand if you have to |
| 4 | You get Accepted but can't explain why a fix worked | Log it in the gap list anyway — "worked but don't know why" is itself a gap |
| 5 | You skip logging because "it was a small lookup" | Small lookups are exactly the ones that resurface in a live interview — log everything, prune later |
| 6 | The 3-day redo feels identical to the first pass | You may be recalling the *specific solution*, not the *general construct* — vary something trivial (variable names, problem order) to break memorized-sequence recall |

---

## 6. Schedule

| Parameter | Value |
|---|---|
| Problems | 30 |
| Time per problem (translate + submit) | 15–25 min |
| Total time per language | ~10 hours |
| At 1 hr/day | ~2 weeks per language |
| Intensive weekend | ~2 days per language |
| Full curriculum (Python + Java, skip JS) | ~4 weeks at 1 hr/day, or ~4 days intensive |

**Time allocation within one problem's 15-25 minute slot** (protocol steps map to §5):

| Step | Activity | Budget |
|---|---|---|
| 1 | C++ anchor from memory | 2 min |
| 2 | Line-by-line translation, references closed | 8-12 min |
| 3 | Doc 44 lookups (as needed, scalpel not crutch) | 2-5 min total |
| 4 | Submit, read failing test cases, fix, resubmit | 3-5 min |
| 5 | Log gap-list entries | 1 min |
| — | **Total** | **15-25 min** |

If a single problem blows past 30 minutes on your first pass, that's a signal the algorithm itself isn't as automatic as you thought — flag it and revisit the relevant Pattern doc after the session, don't burn the whole slot fighting syntax for something you're also unsure how to solve.

### 14-day day-by-day plan (1 language)

| Day | Problems (#) | Notes |
|---|---|---|
| 1 | 1, 2, 3 | Block 1 — hashing fundamentals; expect heaviest lookup volume today, that's normal |
| 2 | 4, 5, 6 | Finish Block 1, start Block 2 strings |
| 3 | 7, 8, 9 | Finish Block 2, start linked lists |
| 4 | 10, 11 | LRU Cache is the heaviest single problem so far — give it the full slot |
| 5 | 12, 13 | Tree traversal + serialization |
| 6 | 14, 15 | Grid + graph BFS/DFS |
| 7 | 16, 17 | Clone Graph, then switch gears into sorting |
| 8 | 18, 19, 20 | Heap, binary search, ordered-map — do 19 and 20 back to back, they cross-drill each other |
| 9 | 21, 22 | DP warm-up then Coin Change |
| 10 | 23, 24 | Edit Distance (hardest DP here), Word Break |
| 11 | 25, 26, 27 | Backtracking, then two quick bit-op problems |
| 12 | 28, 29 | Pow(x,n), Rotate Image |
| 13 | 30 + gap-list review | Trie, then re-drill your worst 3 gap-list entries from the whole run |
| 14 | Final Exam (§7) | 5 unseen mediums, no references, 30 min each |

### 5-day sprint variant (6 problems/day)

| Day | Problems (#) |
|---|---|
| 1 | 1–6 |
| 2 | 7–12 |
| 3 | 13–18 |
| 4 | 19–24 |
| 5 | 25–30 |
| +1 | Final Exam |

Sprint variant trades spacing (worse retention) for speed — use it only if you're under a hard interview deadline. The 14-day version produces more durable recall because of the natural day-boundaries acting as mini spaced-repetition gaps.

---

## 7. The Final Exam

5 **unseen** LeetCode mediums, written directly in the target language, references CLOSED, 30 minutes each, back-to-back or spread across a day.

| # | LC | Problem | Why it's a good exam pick |
|---|---|---|---|
| 1 | 54 | Spiral Matrix | Fresh matrix traversal — no direction-array crutch memorized from #14/#29, forces you to derive bounds live |
| 2 | 253 | Meeting Rooms II | Fresh interval + heap combo — different shape from Merge Intervals (#17) and K Closest Points (#18) |
| 3 | 494 | Target Sum | Fresh DP/backtracking hybrid — tests whether dp-table syntax (§27) and backtracking-copy syntax (§28) both generalize |
| 4 | 199 | Binary Tree Right Side View | Fresh tree + BFS/level tracking — different output shape from Level Order Traversal (#12) |
| 5 | 1046 | Last Stone Weight | Fresh heap usage — tests whether max-heap idiom (§16) is truly reflexive, not memorized to one specific problem shape |

**Pass criteria:** Accepted on first or second submission, for all 5, with **zero syntax lookups** (algorithm thinking time doesn't count against you — only reference-doc opens do). If you fail this bar:
- 1-2 failures: identify the specific surface that broke, drill it in isolation for 15 minutes, retake just those problems.
- 3+ failures: you moved through §6 too fast. Go back to your gap list, re-run the 3-day-cold redo on every problem tied to your worst 5 gap-list entries, then retake the full exam with 5 *different* unseen mediums.

**If a specific exam problem fails, redrill this way:**

| Exam # | If it broke you, redrill | Because it shares |
|---|---|---|
| 1 Spiral Matrix | #14 Number of Islands, #29 Rotate Image | Grid indexing, direction/bounds logic, in-place matrix mutation |
| 2 Meeting Rooms II | #17 Merge Intervals, #18 K Closest Points | Interval sorting, heap-with-comparator |
| 3 Target Sum | #22 Coin Change, #23 Edit Distance, #25 Subsets | dp-table init, backtracking path-copy |
| 4 Right Side View | #12 Level Order Traversal, #11 Maximum Depth | BFS-by-level, tree recursion return values |
| 5 Last Stone Weight | #3 Top K Frequent, #18 K Closest Points | Heap construction and comparator direction |

Do not treat the exam as pass/fail trivia — the table above exists precisely so a failure routes you back to a specific practice problem instead of a vague "study more."

Do not reuse these 5 exam problems for practice beforehand — that defeats the "unseen" purpose. If you've already solved one of these 5 in the past (check your LeetCode submission history), swap it for another medium of the same topic.

---

## 8. Progress Tracker

**Block completion at a glance** — check a block only once every problem in it has both 1st-pass and 3-day-redo checked, for that language:

| Block | Problems | Python | Java | JS |
|---|---|---|---|---|
| 1 Containers and hashing | 1-4 | [ ] | [ ] | [ ] |
| 2 Strings | 5-7 | [ ] | [ ] | [ ] |
| 3 Linked list and class design | 8-10 | [ ] | [ ] | [ ] |
| 4 Trees and recursion | 11-13 | [ ] | [ ] | [ ] |
| 5 Graphs and grids | 14-16 | [ ] | [ ] | [ ] |
| 6 Sorting/heap/search/ordered map | 17-20 | [ ] | [ ] | [ ] |
| 7 DP and backtracking | 21-25 | [ ] | [ ] | [ ] |
| 8 Bit/math/matrix/design | 26-30 | [ ] | [ ] | [ ] |
| — | Final Exam passed | [ ] | [ ] | [ ] |

Two passes per problem per language: **1st** = first translation (step 1-4 of the protocol), **3d** = the 3-day-cold redo (step 6). Check off as you go.

| # | Problem | Python 1st | Python 3d | Java 1st | Java 3d | JS 1st | JS 3d |
|---|---|---|---|---|---|---|---|
| 1 | Two Sum | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 2 | Group Anagrams | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 3 | Top K Frequent Elements | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 4 | Longest Substring Without Repeating Characters | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 5 | Valid Palindrome | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 6 | Reverse Words in a String | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 7 | Reverse String | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 8 | Reverse Linked List | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 9 | Merge Two Sorted Lists | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 10 | LRU Cache | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 11 | Maximum Depth of Binary Tree | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 12 | Binary Tree Level Order Traversal | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 13 | Serialize and Deserialize Binary Tree | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 14 | Number of Islands | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 15 | Course Schedule | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 16 | Clone Graph | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 17 | Merge Intervals | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 18 | K Closest Points to Origin | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 19 | Binary Search | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 20 | My Calendar I | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 21 | Climbing Stairs | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 22 | Coin Change | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 23 | Edit Distance | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 24 | Word Break | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 25 | Subsets | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 26 | Single Number | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 27 | Number of 1 Bits | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 28 | Pow(x, n) | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 29 | Rotate Image | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |
| 30 | Implement Trie (Prefix Tree) | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |

### Personal gap list

Log every reference-doc lookup here, immediately, before you forget it happened. Review this before the final exam and before every 3-day-cold redo.

| Construct (be specific, e.g. "Java max-heap comparator syntax") | Language | Times looked up |
|---|---|---|
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |

---

## 9. How These Docs Fit Together

| Doc | Role | When to use |
|---|---|---|
| 43 (this doc) | The plan — why, what, in what order, how to measure done | Read once up front; revisit §6/§8 to track progress |
| 44 | Rosetta translation tables (all 37 surfaces, side-by-side C++/Java/Python/JS) | Keep OPEN in a side pane while practicing — this is your step-3 lookup target |
| 45 | Worked solutions, problems 1-13, all 4 languages | Only after you've submitted your own attempt — check your translation against it, don't read it first |
| 46 | Worked solutions, problems 14-30, all 4 languages | Same rule as Doc 45 |
| 39 | C++ deep reference (~7000 lines, 8 parts, 37 sections) | Anchor-language lookup if your C++ recall itself is rusty on a specific construct |
| 40 | Java deep reference | When Doc 44's one-line Rosetta entry isn't enough — jump to Doc 40 §33 (recipe index) for the fuller "how do I...?" writeup |
| 41 | JavaScript deep reference | Same, for JS gaps Doc 44 doesn't resolve |
| 42 | Python deep reference | Same, for Python gaps Doc 44 doesn't resolve |

**Escalation path when stuck:** Doc 44 (one-line Rosetta answer) → that language's deep reference §33 "How Do I...?" recipe index (Docs 39/40/41/42) → that language's full deep reference if the recipe index doesn't cover it. Never skip straight to Doc 45/46's worked solution as your first lookup — that shows you the whole answer, not just the syntax construct you're missing, and defeats the purpose of the protocol in §5.

**First-time reading order, start to finish:**
1. §1 (why) → §2 (what) → §4 (which language, in what order) — one sitting, no coding yet.
2. §5 (the loop) → §6 (pick your schedule variant and commit to it on a calendar).
3. Open Doc 44 in a side pane. Open a fresh LeetCode tab. Open §8's tracker.
4. Start problem #1 in your first target language, following the §5 loop exactly.
5. Don't open §3's coverage matrix again until you're deciding whether to do the core-15 or the full 30 — it's a reference table, not a reading requirement.
6. Open §7 only when §8's block-completion table for a language is fully checked.

**Before you start a language, confirm:**
- [ ] You've read §1, §2, §4 for this language's row in the gotcha table.
- [ ] Doc 44 is reachable (bookmarked, downloaded, or open).
- [ ] Your LeetCode account is logged in and set to the target language by default, to avoid losing time re-selecting it every submission.
- [ ] The progress tracker in §8 is either printed, copied into a personal note, or otherwise something you'll actually update — a tracker nobody checks is not a tracker.

**Before you sit the final exam for a language, confirm:**
- [ ] Every row in §8's per-problem tracker for that language has both 1st and 3d checked.
- [ ] Your gap list has no entry with a lookup count that's still climbing — a construct you looked up on problem 28 the same way you looked it up on problem 3 has not actually been learned, redrill it now, not during the exam.

That's the whole plan. Nothing here is clever — the cleverness was already spent on Patterns 01-38. This is 30 problems, 4 languages, one loop, run without shortcuts, until the shortcut you were taking (mentally translating from C++ during a live interview) is no longer necessary.
