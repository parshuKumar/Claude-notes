# PATTERN 27 — SEGMENT TREE
## Range Queries, Lazy Propagation, Segment Tree on Values — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**Why this pattern is different from everything before it:**

Segment tree is not a trick you spot in 10 seconds. It is a **data structure you build from scratch**, every single time, unless your language ships one (C++ doesn't — you write it by hand in every contest). That changes the skill being tested. The interviewer or judge isn't asking "can you notice the pattern," they're asking "can you correctly implement a recursive tree over an array, with O(log n) query and update, without off-by-one errors, in under 15 minutes."

**The honest difficulty curve:**

- **Tier 1 (point update, range query):** Build, update a single index, query a range sum/min/max. This is mechanical once you've written it 3-4 times. The only real danger is array sizing (`4*n`) and midpoint computation.
- **Tier 2 (range update, point query):** A "difference array on a tree" — push a delta to a range, read a single point. Easier than it sounds, and a good stepping stone to lazy propagation.
- **Tier 3 (range update, range query — LAZY PROPAGATION):** This is where most people's segment tree knowledge actually lives or dies. You must defer work at a node, store it as a "lazy" pending operation, and push it down to children **only when you're forced to recurse into them**. Get the order wrong — push after recursing instead of before, forget to multiply by subtree size, forget to clear the lazy flag — and you get silently wrong answers that pass small tests and fail at scale.
- **Tier 4 (segment tree on values / coordinate compression):** The segment tree's index axis is no longer "array position" — it's "compressed value rank." This unlocks an enormous class of problems: counting inversions, counting range sums, computing skyline areas, LIS with a distance constraint. This is the single biggest unlock in this pattern, and it's what separates 1600 from 2000+ contestants.

**What actually separates a correct segment tree from a broken one:**

1. Sizing: `4*n` for the safe recursive array-based tree (a tight bound is closer to `2*2^⌈log2 n⌉`, but `4*n` is always safe and standard).
2. Push_down **before** recursing into children on a partial-overlap update or query — never after.
3. Lazy values must be **cleared** once pushed (or explicitly overwritten for assignment-lazy), and must be **combined** (added), never overwritten, for additive lazy.
4. Midpoint via `l + (r - l) / 2`, never `(l + r) / 2` — the latter overflows for large `l, r` in C++ `int`.
5. The identity element for merges must match the operation: `0` for sum, `INT_MAX` for min, `INT_MIN` for max, `1` for product (rare).

**The honest time investment:** budget a full weekend to internalize lazy propagation cold — not because it's conceptually deep, but because your fingers need to type it correctly without thinking, the way you type a for-loop. Once that's automatic, this pattern becomes one of your highest-leverage tools in competitive programming: O(log n) per operation on a structure that can represent almost any associative range query.

---

## SECTION 2 — THE SEGMENT TREE TAXONOMY

### Type 1: Point Update + Range Query
State: `tree[node]` = merge (sum/min/max/gcd/...) of the range `[l, r]` it represents.
Update: change **one** array position, propagate the change up to the root.
Query: combine O(log n) canonical nodes that exactly cover `[ql, qr]`.
Problems: Range Sum Query — Mutable (LC 307), Range Minimum Query, general RMQ.

### Type 2: Range Update + Point Query
State: same tree, but updates touch an entire subrange.
Technique: store the update as a **lazy delta at the topmost fully-covered node** and only resolve it (sum the path from root to leaf) when a point is read. No push-down needed if you only ever read points — you can just walk root→leaf and accumulate lazy values along the way.
Problems: range-add + single-point read, difference-array-on-a-tree problems.

### Type 3: Range Update + Range Query (LAZY PROPAGATION)
State: tree stores aggregate; a parallel `lazy[]` array stores "pending operation not yet applied to children."
Technique: apply the update/query optimistically at the highest node that's fully covered; **defer** pushing to children until a future operation needs to descend past that node.
Problems: range-add + range-sum, range-assign + range-max, most "real" segment tree problems in contests.

### Type 4: Merge Sort Tree
State: `tree[node]` = a **sorted vector** of all array elements in `[l, r]`, not a scalar.
Technique: build like a merge sort (merge two sorted children into a sorted parent). Query = binary search inside O(log n) canonical nodes → O(log² n) per query.
Problems: "count elements ≤ x in range [l, r]", offline range-rank queries.

### Type 5: Coordinate Compression + Segment Tree
State: the segment tree's index axis represents **compressed values** (ranks), not array positions — or represents a **huge coordinate range** (like `[-10^9, 10^9]`) compressed down to only the coordinates that actually appear.
Technique: collect all coordinates that will ever be touched, sort + dedupe, binary-search to map real coordinate → compressed index, build a segment tree over `[0, m-1]`.
Problems: Falling Squares (LC 699), Count of Smaller Numbers After Self (LC 315), Count of Range Sum (LC 327), Rectangle Area II (LC 850), LIS II (LC 2407).

---

## SECTION 3 — TEMPLATE 1: RECURSIVE SEGMENT TREE — RANGE SUM (FOUNDATION)

This is the template every other template in this document builds on. Internalize it completely before moving on.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 307 — Range Sum Query - Mutable
// Support: update(idx, val) — set arr[idx] = val
//          query(l, r)      — sum of arr[l..r] inclusive
//
// Representation: array-based binary tree.
//   Root is at index 1. Node `node` has children `2*node` and `2*node+1`.
//   tree[node] covers array range [l, r] (tracked via recursion, not stored).
//   Size 4*n is a SAFE upper bound on array slots needed (proof: the tree
//   has depth ceil(log2 n) + 1; a tight bound is 2*2^(depth), which is
//   always ≤ 4n for n ≥ 1).
// ─────────────────────────────────────────────────────────────────

class SegTreeSum {
public:
    vector<long long> tree;   // 1-indexed segment tree, size 4*n
    int n;

    SegTreeSum(vector<int>& arr) {
        n = arr.size();
        tree.assign(4 * n, 0);
        build(arr, 1, 0, n - 1);
    }

    void build(vector<int>& arr, int node, int l, int r) {
        if (l == r) {
            tree[node] = arr[l];          // leaf: exactly one array element
            return;
        }
        int mid = l + (r - l) / 2;        // never (l+r)/2 — overflow risk
        build(arr, 2 * node,     l,     mid);
        build(arr, 2 * node + 1, mid + 1, r);
        tree[node] = tree[2 * node] + tree[2 * node + 1];   // merge = sum
    }

    // Point update: arr[idx] = val
    void update(int node, int l, int r, int idx, int val) {
        if (l == r) {
            tree[node] = val;
            return;
        }
        int mid = l + (r - l) / 2;
        if (idx <= mid) update(2 * node,     l,     mid, idx, val);
        else             update(2 * node + 1, mid + 1, r,   idx, val);
        tree[node] = tree[2 * node] + tree[2 * node + 1];   // pull up
    }
    void update(int idx, int val) { update(1, 0, n - 1, idx, val); }

    // Range query: sum of arr[ql..qr]
    long long query(int node, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return 0;              // no overlap — identity for sum
        if (ql <= l && r <= qr) return tree[node];    // total overlap — return cached value
        int mid = l + (r - l) / 2;                    // partial overlap — recurse both sides
        return query(2 * node, l, mid, ql, qr)
             + query(2 * node + 1, mid + 1, r, ql, qr);
    }
    long long query(int ql, int qr) { return query(1, 0, n - 1, ql, qr); }
};
```

**The three overlap cases — memorize this decision tree, it appears in every template below:**

```
Given current node covers [l, r], query wants [ql, qr]:

1. NO OVERLAP:      qr < l  OR  r < ql        → return identity (0 for sum)
2. TOTAL OVERLAP:    ql <= l  AND  r <= qr      → return tree[node] directly (O(1))
3. PARTIAL OVERLAP:  otherwise                  → recurse into both children, combine
```

This is why query is O(log n): at each depth, at most O(1) nodes are "partial" (the boundary nodes), and the rest resolve in O(1) via case 2.

---

### FULL BUILD + QUERY TRACE

Array: `[2, 5, 1, 4, 9, 3]` (indices 0..5, n = 6).

**Build trace** (node : [l,r] → value):

```
build(1, 0, 5)  mid = 2
├── build(2, 0, 2)  mid = 1
│   ├── build(4, 0, 1)  mid = 0
│   │   ├── build(8, 0, 0)  → leaf, tree[8] = 2
│   │   └── build(9, 1, 1)  → leaf, tree[9] = 5
│   │   tree[4] = tree[8] + tree[9] = 2 + 5 = 7
│   └── build(5, 2, 2)  → leaf, tree[5] = 1
│   tree[2] = tree[4] + tree[5] = 7 + 1 = 8
└── build(3, 3, 5)  mid = 4
    ├── build(6, 3, 4)  mid = 3
    │   ├── build(12, 3, 3) → leaf, tree[12] = 4
    │   └── build(13, 4, 4) → leaf, tree[13] = 9
    │   tree[6] = tree[12] + tree[13] = 4 + 9 = 13
    └── build(7, 5, 5)  → leaf, tree[7] = 3
    tree[3] = tree[6] + tree[7] = 13 + 3 = 16

tree[1] = tree[2] + tree[3] = 8 + 16 = 24   (= 2+5+1+4+9+3 ✓)
```

**Query trace** for `query(1, 4)` — expected answer: `5+1+4+9 = 19`:

```
query(node=1, [0,5], want [1,4])
  Partial overlap (0<1). mid=2.
  ├── query(node=2, [0,2], want [1,4])
  │     Partial overlap (0<1). mid=1.
  │     ├── query(node=4, [0,1], want [1,4])
  │     │     Partial overlap (0<1). mid=0.
  │     │     ├── query(node=8, [0,0], want [1,4]) → r=0 < ql=1 → NO OVERLAP → 0
  │     │     └── query(node=9, [1,1], want [1,4]) → 1<=1<=4 → TOTAL OVERLAP → tree[9]=5
  │     │     returns 0 + 5 = 5
  │     └── query(node=5, [2,2], want [1,4]) → 1<=2<=4 → TOTAL OVERLAP → tree[5]=1
  │     returns 5 + 1 = 6
  └── query(node=3, [3,5], want [1,4])
        Partial overlap (5>4). mid=4.
        ├── query(node=6, [3,4], want [1,4]) → 1<=3 and 4<=4 → TOTAL OVERLAP → tree[6]=13
        └── query(node=7, [5,5], want [1,4]) → l=5 > qr=4 → NO OVERLAP → 0
        returns 13 + 0 = 13
  returns 6 + 13 = 19  ✓
```

Only 4 leaves were ever "fully resolved" via case 2 (nodes 9, 5, 6 — three canonical nodes) — this is the O(log n) canonical decomposition in action. Total function calls: 9, for an array of size 6; the call count is always O(log n) canonical nodes plus O(log n) path nodes on each side ≈ O(log n) total, not O(n).

---

## SECTION 4 — TEMPLATE 2: RANGE MIN / RANGE MAX VARIANTS

The entire segment tree machinery is unchanged. Only two lines differ: **the merge operation** and **the identity element** used for "no overlap."

```cpp
// ─────────────────────────────────────────────────────────────────
// RANGE MINIMUM QUERY — only 2 lines change vs. the sum version
// ─────────────────────────────────────────────────────────────────
class SegTreeMin {
public:
    vector<int> tree;
    int n;
    SegTreeMin(vector<int>& arr) {
        n = arr.size();
        tree.assign(4 * n, INT_MAX);         // default = identity for min
        build(arr, 1, 0, n - 1);
    }
    void build(vector<int>& arr, int node, int l, int r) {
        if (l == r) { tree[node] = arr[l]; return; }
        int mid = l + (r - l) / 2;
        build(arr, 2*node, l, mid);
        build(arr, 2*node+1, mid+1, r);
        tree[node] = min(tree[2*node], tree[2*node+1]);      // CHANGED: min not +
    }
    void update(int node, int l, int r, int idx, int val) {
        if (l == r) { tree[node] = val; return; }
        int mid = l + (r - l) / 2;
        if (idx <= mid) update(2*node, l, mid, idx, val);
        else update(2*node+1, mid+1, r, idx, val);
        tree[node] = min(tree[2*node], tree[2*node+1]);      // CHANGED
    }
    void update(int idx, int val) { update(1, 0, n-1, idx, val); }
    int query(int node, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return INT_MAX;                // CHANGED: identity for min
        if (ql <= l && r <= qr) return tree[node];
        int mid = l + (r - l) / 2;
        return min(query(2*node, l, mid, ql, qr), query(2*node+1, mid+1, r, ql, qr));
    }
    int query(int ql, int qr) { return query(1, 0, n-1, ql, qr); }
};

// RANGE MAXIMUM QUERY — identical, but:
//   default/identity = INT_MIN
//   merge = max(...) instead of min(...)
```

**The general lesson:** any operation that is **associative** (order of combination doesn't matter for grouping) can plug into this skeleton — sum, min, max, gcd, bitwise AND/OR/XOR, product (with overflow care). The identity element is whatever value `x` satisfies `merge(x, y) == y` for all `y`: `0` for `+`, `INT_MAX` for `min`, `INT_MIN` for `max`, `0` for `xor`/`or`, `-1` (all bits set) for `and`.

---

## SECTION 5 — TEMPLATE 3: ITERATIVE SEGMENT TREE (BOTTOM-UP, SIZE 2N)

The recursive tree is easiest to reason about and easiest to extend with lazy propagation. But for **pure point-update / range-query** problems with no lazy propagation needed, an iterative bottom-up tree (popularized by Al.Cash's Codeforces blog "Efficient and Easy Segment Trees") is shorter, faster in practice (no recursion overhead), and uses only `2*n` space instead of `4*n`.

```cpp
// ─────────────────────────────────────────────────────────────────
// ITERATIVE SEGMENT TREE — range sum, point update
// Leaves live at indices [n, 2n-1]. Internal nodes at [1, n-1].
// tree[i]'s children are tree[2i] and tree[2i+1] — SAME indexing
// convention as a binary heap, but built bottom-up instead of top-down.
// Queries use HALF-OPEN ranges [l, r) — this is the standard convention
// for this style and avoids +1/-1 fencepost bugs.
// ─────────────────────────────────────────────────────────────────
struct IterativeSegTree {
    int n;
    vector<long long> tree;

    IterativeSegTree(vector<int>& arr) {
        n = arr.size();
        tree.assign(2 * n, 0);
        for (int i = 0; i < n; i++) tree[n + i] = arr[i];        // leaves
        for (int i = n - 1; i >= 1; i--)                         // build internal nodes
            tree[i] = tree[2*i] + tree[2*i+1];
    }

    void update(int pos, int val) {
        pos += n;                       // map array index → leaf index
        tree[pos] = val;
        for (pos /= 2; pos >= 1; pos /= 2)
            tree[pos] = tree[2*pos] + tree[2*pos+1];             // walk up to root
    }

    // Sum of arr[l..r-1] — HALF-OPEN interval [l, r)
    long long query(int l, int r) {
        long long res = 0;
        for (l += n, r += n; l < r; l /= 2, r /= 2) {
            if (l & 1) res += tree[l++];   // l is a right child → it's a canonical node, take it, move past it
            if (r & 1) res += tree[--r];   // r is a right child → step back, take it as canonical node
        }
        return res;
    }
};
```

**Why this works (the intuition, not just the mechanics):** as `l` and `r` climb toward the root one level at a time, whenever `l` is a right child of its parent, that means its LEFT sibling is entirely outside `[l, r)` (to the left) — so `l` itself is a maximal canonical block, we take it and move to `l+1`, then go up. Symmetrically for `r`. This is the same "canonical decomposition into O(log n) blocks" idea as the recursive version, just derived bottom-up via binary index arithmetic instead of top-down recursion.

**Iterative vs. recursive — when to use which:**

| | Recursive (4N) | Iterative (2N) |
|---|---|---|
| Lazy propagation | Natural, standard | Awkward, rarely done (needs extra bookkeeping) |
| Code length | Longer | ~15 lines |
| Constant factor | Slower (recursion + function calls) | Faster (tight loop, no recursion) |
| Range convention | Inclusive `[l, r]` — matches problem statements | Half-open `[l, r)` — needs care at call sites |
| Best for | Any problem, especially with lazy | Simple point-update/range-query only |

**TRACE** for `arr = [1, 3, 5, 7, 9, 11]` (n=6):

```
Leaves (tree[6..11]): [1, 3, 5, 7, 9, 11]
i=5: tree[5] = tree[10]+tree[11] =  9+11 = 20
i=4: tree[4] = tree[8]+tree[9]   =  5+ 7 = 12
i=3: tree[3] = tree[6]+tree[7]   =  1+ 3 =  4
i=2: tree[2] = tree[4]+tree[5]   = 12+20 = 32
i=1: tree[1] = tree[2]+tree[3]   = 32+ 4 = 36   (total = 1+3+5+7+9+11 = 36 ✓)

query(1, 4)  — sum of indices {1,2,3} = 3+5+7 = 15
  l = 1+6 = 7,  r = 4+6 = 10
  iter 1: l=7,r=10, l<r
    l&1=1 → res += tree[7]=3,  l=8
    r&1=0 → no change
    l/=2 → 4,  r/=2 → 5
  iter 2: l=4,r=5, l<r
    l&1=0 → no change
    r&1=1 → res += tree[--r=4] = tree[4] = 12,  r=4
    l/=2 → 2,  r/=2 → 2
  iter 3: l=2,r=2 → loop ends (l < r is false)

res = 3 + 12 = 15  ✓
```

---

## SECTION 6 — TEMPLATE 4: LAZY PROPAGATION — RANGE ADD + RANGE SUM

This is the core of Tier 3. Read this section slowly. Almost every hard segment tree problem is a variation of this exact skeleton.

**The core idea:** when an update covers a node's range *completely*, don't recurse into its children — just update this node's aggregate directly, and record "children owe this update" in a `lazy[]` slot. Only when a **future** operation is forced to look *inside* this node's children (because it only partially overlaps) do you finally push the pending update down one level — and immediately mark it applied at this node.

```cpp
// ─────────────────────────────────────────────────────────────────
// LAZY PROPAGATION — range add, range sum
// updateRange(l, r, val): add `val` to every element in arr[l..r]
// query(l, r):            sum of arr[l..r]
// ─────────────────────────────────────────────────────────────────
struct LazySegTree {
    int n;
    vector<long long> tree, lazy;

    LazySegTree(vector<int>& arr) {
        n = arr.size();
        tree.assign(4 * n, 0);
        lazy.assign(4 * n, 0);          // 0 = "no pending add" (safe because 0 is the additive identity)
        build(arr, 1, 0, n - 1);
    }

    void build(vector<int>& arr, int node, int l, int r) {
        if (l == r) { tree[node] = arr[l]; return; }
        int mid = l + (r - l) / 2;
        build(arr, 2*node, l, mid);
        build(arr, 2*node+1, mid+1, r);
        tree[node] = tree[2*node] + tree[2*node+1];
    }

    // Push this node's pending lazy value down to BOTH children, then clear it here.
    // MUST be called before recursing into children whenever this node has a
    // partial-overlap update or query in flight.
    void push_down(int node, int l, int r) {
        if (lazy[node] == 0) return;                 // nothing pending — cheap early exit
        int mid = l + (r - l) / 2;
        int leftLen  = mid - l + 1;
        int rightLen = r - mid;

        // Apply the pending add to each child's STORED SUM — must scale by
        // subtree size, since "add v to every element" changes the sum by v*count.
        tree[2*node]     += lazy[node] * leftLen;
        tree[2*node + 1] += lazy[node] * rightLen;

        // Accumulate (not overwrite!) — a child might already have its own
        // pending lazy from an earlier update that hasn't been pushed further down yet.
        lazy[2*node]     += lazy[node];
        lazy[2*node + 1] += lazy[node];

        lazy[node] = 0;                                // fully delegated to children — clear it
    }

    void updateRange(int node, int l, int r, int ql, int qr, int val) {
        if (qr < l || r < ql) return;                              // no overlap
        if (ql <= l && r <= qr) {                                  // total overlap — stop here, defer
            tree[node] += (long long)val * (r - l + 1);
            lazy[node] += val;
            return;
        }
        push_down(node, l, r);                                     // partial overlap — MUST push before recursing
        int mid = l + (r - l) / 2;
        updateRange(2*node, l, mid, ql, qr, val);
        updateRange(2*node+1, mid+1, r, ql, qr, val);
        tree[node] = tree[2*node] + tree[2*node+1];                // pull up AFTER children are finalized
    }
    void updateRange(int ql, int qr, int val) { updateRange(1, 0, n-1, ql, qr, val); }

    long long query(int node, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) return tree[node];
        push_down(node, l, r);                                     // MUST push before descending
        int mid = l + (r - l) / 2;
        return query(2*node, l, mid, ql, qr) + query(2*node+1, mid+1, r, ql, qr);
    }
    long long query(int ql, int qr) { return query(1, 0, n-1, ql, qr); }
};
```

**The golden rule, stated once, precisely:** *any time you are about to recurse into a node's children — whether for an update or a query — call `push_down` on that node first.* No exceptions. The only place you never call `push_down` is on a leaf (it has no children) or on a node that resolves fully via case 2 (no recursion happens, so nothing needs pushing).

---

### FULL LAZY PROPAGATION TRACE

Array: `[1, 2, 3, 4, 5, 6, 7, 8]` (n=8, indices 0..7). Perform `updateRange(2, 5, +10)` (add 10 to indices 2..5), then `query(3, 4)`.

**Tree layout** (perfectly balanced since n=8): node1:[0,7], node2:[0,3], node3:[4,7], node4:[0,1], node5:[2,3], node6:[4,5], node7:[6,7], leaves node8..node15 = indices 0..7.

**Step 1 — `updateRange(2, 5, +10)`:**

```
updateRange(node1, [0,7], want [2,5]) — partial overlap. push_down(node1): lazy=0, no-op. mid=3.
├── updateRange(node2, [0,3], want [2,5]) — partial overlap. push_down(node2): no-op. mid=1.
│   ├── updateRange(node4, [0,1], want [2,5]) — r=1 < ql=2 → NO OVERLAP, return
│   └── updateRange(node5, [2,3], want [2,5]) — TOTAL OVERLAP (2<=2, 3<=5)
│         tree[5] += 10*(3-2+1) = 10*2 = 20  →  tree[5] = (3+4)+20 = 27
│         lazy[5] += 10  →  lazy[5] = 10          ★ STOPS HERE — does not recurse into leaves 10,11
│   tree[2] = tree[4]+tree[5] = 3 + 27 = 30
└── updateRange(node3, [4,7], want [2,5]) — partial overlap. push_down(node3): no-op. mid=5.
    ├── updateRange(node6, [4,5], want [2,5]) — TOTAL OVERLAP (2<=4, 5<=5)
    │     tree[6] += 10*(5-4+1) = 10*2 = 20  →  tree[6] = (5+6)+20 = 31
    │     lazy[6] += 10  →  lazy[6] = 10          ★ STOPS HERE — does not recurse into leaves 12,13
    └── updateRange(node7, [6,7], want [2,5]) — qr=5 < l=6 → NO OVERLAP, return
    tree[3] = tree[6]+tree[7] = 31 + 15 = 46

tree[1] = tree[2]+tree[3] = 30 + 46 = 76
```

Check: original sum 36, plus `10 × 4` indices (2,3,4,5) = `36 + 40 = 76`. ✓

**Critical state after the update:** `node5` (covers indices [2,3]) and `node6` (covers indices [4,5]) are each **internal nodes with `lazy = 10` pending** — their own `tree[]` value is already correct, but their *children* (the actual leaves for indices 2,3,4,5) have stale values. This is the entire point of laziness: we did O(log n) work, not O(range length) work.

**Step 2 — `query(3, 4)`:** expected `arr[3] + arr[4] = (4+10) + (5+10) = 14 + 15 = 29`.

```
query(node1, [0,7], want [3,4]) — partial. push_down(node1): lazy=0, no-op. mid=3.
├── query(node2, [0,3], want [3,4]) — partial (0<3). push_down(node2): lazy=0, no-op. mid=1.
│   ├── query(node4, [0,1], want [3,4]) — r=1 < ql=3 → NO OVERLAP → 0
│   └── query(node5, [2,3], want [3,4]) — partial (2<3, but 3<=3<=4 ⇒ not total: ql<=l fails, 3<=2 false)
│         ★ push_down(node5, [2,3]): lazy[5]=10 ≠ 0!
│             mid=2, leftLen=1, rightLen=1
│             tree[10] += 10*1 = 10  →  tree[10] = 3+10 = 13   (index 2 = 3+10 ✓)
│             tree[11] += 10*1 = 10  →  tree[11] = 4+10 = 14   (index 3 = 4+10 ✓)
│             lazy[5] = 0   (cleared — now applied to children)
│         mid=2.
│         ├── query(node10, [2,2], want [3,4]) — r=2 < ql=3 → NO OVERLAP → 0
│         └── query(node11, [3,3], want [3,4]) — TOTAL OVERLAP → tree[11] = 14
│         returns 0 + 14 = 14
│   returns 0 + 14 = 14
└── query(node3, [4,7], want [3,4]) — partial. push_down(node3): lazy=0, no-op. mid=5.
    ├── query(node6, [4,5], want [3,4]) — partial (5>4, not total: r<=qr fails)
    │     ★ push_down(node6, [4,5]): lazy[6]=10 ≠ 0!
    │         mid=4, leftLen=1, rightLen=1
    │         tree[12] += 10*1 = 10  →  tree[12] = 5+10 = 15   (index 4 = 5+10 ✓)
    │         tree[13] += 10*1 = 10  →  tree[13] = 6+10 = 16   (index 5 = 6+10 ✓)
    │         lazy[6] = 0
    │     mid=4.
    │     ├── query(node12, [4,4], want [3,4]) — TOTAL OVERLAP → tree[12] = 15
    │     └── query(node13, [5,5], want [3,4]) — l=5 > qr=4 → NO OVERLAP → 0
    │     returns 15 + 0 = 15
    └── query(node7, [6,7], want [3,4]) — l=6 > qr=4 → NO OVERLAP → 0
    returns 15 + 0 = 15

TOTAL = 14 + 15 = 29  ✓
```

This trace shows the two moments — at `node5` and `node6` — where a pending lazy value from Step 1 was finally resolved because a later query needed to see inside those subtrees. Everywhere else, the tree either resolved in O(1) (total overlap on a node with correct aggregate) or bailed out immediately (no overlap).

---

## SECTION 7 — TEMPLATE 5: RANGE-ASSIGN LAZY VARIANT

Range-add lazy **accumulates** (two pending adds combine into one bigger add). Range-**assign** lazy is different: a later assignment completely **overwrites** whatever was pending before, because "set to 7, then set to 12" is the same as just "set to 12" — the earlier operation is irrelevant.

The other subtlety: for range-add, `0` is a safe "no pending update" sentinel, because adding 0 is a no-op. For range-**assign**, `0` might be a *legitimate* value to assign — so you need an explicit boolean flag instead of overloading a sentinel value.

```cpp
// ─────────────────────────────────────────────────────────────────
// RANGE ASSIGN + RANGE SUM
// updateRange(l, r, val): set every element in arr[l..r] to val
// query(l, r):            sum of arr[l..r]
// ─────────────────────────────────────────────────────────────────
struct RangeAssignSegTree {
    int n;
    vector<long long> tree, lazy;
    vector<bool> hasLazy;            // explicit flag — 0 is a valid assign value!

    RangeAssignSegTree(vector<int>& arr) {
        n = arr.size();
        tree.assign(4*n, 0);
        lazy.assign(4*n, 0);
        hasLazy.assign(4*n, false);
        build(arr, 1, 0, n-1);
    }

    void build(vector<int>& arr, int node, int l, int r) {
        if (l == r) { tree[node] = arr[l]; return; }
        int mid = l + (r - l) / 2;
        build(arr, 2*node, l, mid);
        build(arr, 2*node+1, mid+1, r);
        tree[node] = tree[2*node] + tree[2*node+1];
    }

    void push_down(int node, int l, int r) {
        if (!hasLazy[node]) return;
        int mid = l + (r - l) / 2;
        int leftLen = mid - l + 1, rightLen = r - mid;

        tree[2*node]     = lazy[node] * leftLen;     // OVERWRITE (assign), not +=
        tree[2*node + 1] = lazy[node] * rightLen;

        lazy[2*node] = lazy[2*node + 1] = lazy[node];  // OVERWRITE child's pending value too —
        hasLazy[2*node] = hasLazy[2*node + 1] = true;  // this assign supersedes anything queued there

        hasLazy[node] = false;
    }

    void updateRange(int node, int l, int r, int ql, int qr, int val) {
        if (qr < l || r < ql) return;
        if (ql <= l && r <= qr) {
            tree[node] = (long long)val * (r - l + 1);
            lazy[node] = val;
            hasLazy[node] = true;
            return;
        }
        push_down(node, l, r);
        int mid = l + (r - l) / 2;
        updateRange(2*node, l, mid, ql, qr, val);
        updateRange(2*node+1, mid+1, r, ql, qr, val);
        tree[node] = tree[2*node] + tree[2*node+1];
    }
    void updateRange(int ql, int qr, int val) { updateRange(1, 0, n-1, ql, qr, val); }

    long long query(int node, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) return tree[node];
        push_down(node, l, r);
        int mid = l + (r - l) / 2;
        return query(2*node, l, mid, ql, qr) + query(2*node+1, mid+1, r, ql, qr);
    }
    long long query(int ql, int qr) { return query(1, 0, n-1, ql, qr); }
};
```

**Assign vs. add — the one-line rule:** in `push_down`, additive lazy uses `+=` on the child's lazy slot (combine pending operations); assignment lazy uses `=` on the child's lazy slot (the newer assignment discards the older one). Mixing these up — using `+=` for an assign-lazy tree — is one of the most common contest bugs (see Common Mistake 3 below).

If a problem needs **both** range-add and range-assign simultaneously (rare but it happens — e.g. Codeforces "Chtholly Tree" style problems), the convention is: assignment always takes priority and clears any pending add, because "set to X" makes any earlier "add Y" irrelevant.

---

## SECTION 8 — TEMPLATE 6: COORDINATE COMPRESSION + SEGMENT TREE (FALLING SQUARES)

Coordinate compression lets you build a segment tree over an axis with values up to `10^9` by first shrinking it down to only the O(n) coordinates that actually matter.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 699 — Falling Squares
// Squares drop at positions[i] = [left, size]. Each square falls straight
// down and rests on top of whatever is directly beneath it (or the floor).
// After each square lands, report the height of the TALLEST stack so far.
//
// KEY IDEA: coordinates can be up to 10^9, so we can't index a segment
// tree by raw x-coordinate. Instead, compress: collect every square's
// left edge and right edge (left + size), sort + dedupe, and build the
// segment tree over these O(n) compressed indices.
//
// Once a square lands, the ENTIRE range it covers becomes one flat height
// (until a future square partially overlaps and raises part of it) —
// so this is a RANGE-ASSIGN, RANGE-MAX segment tree.
// ─────────────────────────────────────────────────────────────────

vector<int> fallingSquares(vector<vector<int>>& positions) {
    // Step 1: gather + compress all coordinates that matter
    vector<int> xs;
    for (auto& p : positions) {
        xs.push_back(p[0]);           // left edge
        xs.push_back(p[0] + p[1]);    // right edge (exclusive)
    }
    sort(xs.begin(), xs.end());
    xs.erase(unique(xs.begin(), xs.end()), xs.end());
    int m = xs.size();
    auto idx = [&](int x) {
        return (int)(lower_bound(xs.begin(), xs.end(), x) - xs.begin());
    };

    // Step 2: segment tree over compressed indices [0, m-1]
    //         supports range-assign + range-max
    vector<int> tree(4 * m, 0), lazy(4 * m, -1);   // -1 = no pending assign (heights are >= 0)

    function<void(int,int,int)> push_down = [&](int node, int l, int r) {
        if (lazy[node] == -1) return;
        tree[2*node] = tree[2*node+1] = lazy[node];
        lazy[2*node] = lazy[2*node+1] = lazy[node];
        lazy[node] = -1;
    };
    function<void(int,int,int,int,int,int)> update = [&](int node, int l, int r, int ql, int qr, int val) {
        if (qr < l || r < ql) return;
        if (ql <= l && r <= qr) { tree[node] = val; lazy[node] = val; return; }
        push_down(node, l, r);
        int mid = l + (r - l) / 2;
        update(2*node, l, mid, ql, qr, val);
        update(2*node+1, mid+1, r, ql, qr, val);
        tree[node] = max(tree[2*node], tree[2*node+1]);
    };
    function<int(int,int,int,int,int)> query = [&](int node, int l, int r, int ql, int qr) -> int {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) return tree[node];
        push_down(node, l, r);
        int mid = l + (r - l) / 2;
        return max(query(2*node, l, mid, ql, qr), query(2*node+1, mid+1, r, ql, qr));
    };

    vector<int> result;
    int curMax = 0;
    for (auto& p : positions) {
        int left = idx(p[0]);
        int right = idx(p[0] + p[1]) - 1;   // compressed index of the LAST covered slot
        // (right edge is exclusive on the real number line, so the covered
        //  compressed range is [left, right_compressed_index - 1])

        int base = query(1, 0, m - 1, left, right);   // tallest stack currently under this square
        int top  = base + p[1];                        // this square rests on top of that
        update(1, 0, m - 1, left, right, top);          // the whole footprint becomes flat height `top`

        curMax = max(curMax, top);
        result.push_back(curMax);
    }
    return result;
}
```

**Why `right = idx(p[0]+p[1]) - 1`:** the compressed coordinate array contains *points*, not ranges. A square spanning `[left, left+size)` on the real line covers compressed indices from `idx(left)` up to (but not including) `idx(left+size)`. Since our segment tree query/update takes an **inclusive** `[ql, qr]`, we subtract 1 from the right endpoint's compressed index.

**Trace** for `positions = [[1,2],[2,3],[6,1]]`:

```
Coordinates seen: 1, 3(=1+2), 2, 5(=2+3), 6, 7(=6+1)
Sorted+deduped xs = [1, 2, 3, 5, 6, 7]   (m=6, indices 0..5)

Square 1: [1,2] → left=idx(1)=0, right=idx(3)-1=2-1=1
  base = query(0,1) = 0 (empty). top = 0+2 = 2. update(0,1,2). curMax=2. result=[2]

Square 2: [2,3] → left=idx(2)=1, right=idx(5)-1=3-1=2
  base = query(1,2): index1 has height 2 (from square1's update covering [0,1]), index2 has height 0
         → max(2,0) = 2. top = 2+3 = 5. update(1,2,5). curMax=5. result=[2,5]

Square 3: [6,1] → left=idx(6)=4, right=idx(7)-1=5-1=4
  base = query(4,4) = 0 (untouched). top = 0+1 = 1. update(4,4,1). curMax stays 5. result=[2,5,5]
```

Matches the known LC 699 expected output `[2, 5, 5]` for this exact input. ✓

---

## SECTION 9 — TEMPLATE 7: SEGMENT TREE ON VALUES — COUNT OF SMALLER NUMBERS AFTER SELF (LC 315)

This is the template that unlocks the entire "inversion counting" family. The segment tree's index axis is the **compressed value**, not the array position. Each leaf tracks "how many times has this value been inserted so far."

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 315 — Count of Smaller Numbers After Self
// For each nums[i], count how many elements to its RIGHT are smaller than it.
//
// STRATEGY: process the array RIGHT TO LEFT. Maintain a "frequency
// segment tree" over compressed value ranks. Before inserting nums[i],
// query "how many already-inserted values have rank < rank(nums[i])" —
// that count IS the answer for index i, because everything already
// inserted is to the right of i (we're going right to left).
// ─────────────────────────────────────────────────────────────────

vector<int> countSmaller(vector<int>& nums) {
    // Step 1: compress values to dense ranks
    vector<int> sortedVals(nums.begin(), nums.end());
    sort(sortedVals.begin(), sortedVals.end());
    sortedVals.erase(unique(sortedVals.begin(), sortedVals.end()), sortedVals.end());
    int m = sortedVals.size();
    auto rank = [&](int x) {
        return (int)(lower_bound(sortedVals.begin(), sortedVals.end(), x) - sortedVals.begin());
    };

    // Step 2: frequency segment tree over ranks [0, m-1]
    vector<int> tree(4 * m, 0);
    function<void(int,int,int,int)> update = [&](int node, int l, int r, int pos) {
        if (l == r) { tree[node]++; return; }
        int mid = l + (r - l) / 2;
        if (pos <= mid) update(2*node, l, mid, pos);
        else update(2*node+1, mid+1, r, pos);
        tree[node] = tree[2*node] + tree[2*node+1];
    };
    function<int(int,int,int,int,int)> query = [&](int node, int l, int r, int ql, int qr) -> int {
        if (qr < l || r < ql || ql > qr) return 0;
        if (ql <= l && r <= qr) return tree[node];
        int mid = l + (r - l) / 2;
        return query(2*node, l, mid, ql, qr) + query(2*node+1, mid+1, r, ql, qr);
    };

    int n = nums.size();
    vector<int> result(n);
    for (int i = n - 1; i >= 0; i--) {
        int r = rank(nums[i]);
        result[i] = (r == 0) ? 0 : query(1, 0, m - 1, 0, r - 1);   // count of strictly smaller ranks
        update(1, 0, m - 1, r);                                     // now insert nums[i] itself
    }
    return result;
}
```

**Trace** for `nums = [5, 2, 6, 1]`:

```
Sorted unique values: [1, 2, 5, 6]  →  ranks: 1→0, 2→1, 5→2, 6→3

i=3, val=1, rank=0: r==0 → result[3] = 0.               insert rank0.  counts: {0:1}
i=2, val=6, rank=3: query(0,2) = count of ranks{0,1,2} = 1 (only rank0)  → result[2] = 1.  insert rank3.  counts: {0:1, 3:1}
i=1, val=2, rank=1: query(0,0) = count of rank{0} = 1                    → result[1] = 1.  insert rank1.  counts: {0:1, 1:1, 3:1}
i=0, val=5, rank=2: query(0,1) = count of ranks{0,1} = 1+1 = 2           → result[0] = 2.  insert rank2.

result = [2, 1, 1, 0]
```

This matches the well-known expected output for `[5,2,6,1]` → `[2,1,1,0]`. ✓ (Read: to the right of `5` there are two smaller elements `2,1`; to the right of `2` there is one smaller `1`; to the right of `6` there is one smaller `1`; to the right of `1` there are none.)

---

## SECTION 10 — TEMPLATE 8: SEGMENT TREE ON VALUES — COUNT OF RANGE SUM (LC 327)

A harder variant of the same idea: instead of compressing raw array values, we compress **prefix sums**, and instead of counting "smaller," we count "how many earlier prefix sums fall inside a computed range."

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 327 — Count of Range Sum
// Count the number of (i, j) pairs with i < j such that
//   lower <= sum(nums[i..j-1]) <= upper
//
// Let prefix[k] = sum(nums[0..k-1]), prefix[0] = 0.
// sum(nums[i..j-1]) = prefix[j] - prefix[i].
// We need: lower <= prefix[j] - prefix[i] <= upper
//       ⇔  prefix[j] - upper <= prefix[i] <= prefix[j] - lower
//
// Process j = 1..n in order. Before processing j, all prefix[i] for i<j
// have already been inserted into the segment tree. For each j, query how
// many inserted prefix[i] values fall in [prefix[j]-upper, prefix[j]-lower],
// then insert prefix[j] itself.
// ─────────────────────────────────────────────────────────────────

int countRangeSum(vector<int>& nums, int lower, int upper) {
    int n = nums.size();
    vector<long long> prefix(n + 1, 0);
    for (int i = 0; i < n; i++) prefix[i+1] = prefix[i] + nums[i];

    // Compress ALL prefix sum values (these are the only values we'll ever query/insert)
    vector<long long> sortedVals(prefix.begin(), prefix.end());
    sort(sortedVals.begin(), sortedVals.end());
    sortedVals.erase(unique(sortedVals.begin(), sortedVals.end()), sortedVals.end());
    int m = sortedVals.size();
    auto rank = [&](long long x) {
        return (int)(lower_bound(sortedVals.begin(), sortedVals.end(), x) - sortedVals.begin());
    };

    vector<int> tree(4 * m, 0);
    function<void(int,int,int,int)> update = [&](int node, int l, int r, int pos) {
        if (l == r) { tree[node]++; return; }
        int mid = l + (r - l) / 2;
        if (pos <= mid) update(2*node, l, mid, pos);
        else update(2*node+1, mid+1, r, pos);
        tree[node] = tree[2*node] + tree[2*node+1];
    };
    function<int(int,int,int,int,int)> query = [&](int node, int l, int r, int ql, int qr) -> int {
        if (qr < l || r < ql || ql > qr) return 0;
        if (ql <= l && r <= qr) return tree[node];
        int mid = l + (r - l) / 2;
        return query(2*node, l, mid, ql, qr) + query(2*node+1, mid+1, r, ql, qr);
    };

    int count = 0;
    update(1, 0, m - 1, rank(prefix[0]));            // insert prefix[0] before processing j=1..n
    for (int j = 1; j <= n; j++) {
        long long loBound = prefix[j] - upper;
        long long hiBound = prefix[j] - lower;
        // Map [loBound, hiBound] to compressed rank range that actually exists in sortedVals
        int lo = (int)(lower_bound(sortedVals.begin(), sortedVals.end(), loBound) - sortedVals.begin());
        int hi = (int)(upper_bound(sortedVals.begin(), sortedVals.end(), hiBound) - sortedVals.begin()) - 1;
        if (lo <= hi) count += query(1, 0, m - 1, lo, hi);
        update(1, 0, m - 1, rank(prefix[j]));
    }
    return count;
}
```

**Validation** for `nums = [-2, 5, -1]`, `lower = -2`, `upper = 2` (a well-known example, expected answer `3`):

```
prefix = [0, -2, 3, 2]

All (i, j) pairs, i < j, checking lower<=prefix[j]-prefix[i]<=upper:
  (0,1): prefix[1]-prefix[0] = -2-0  = -2   → in [-2,2] ✓
  (0,2): prefix[2]-prefix[0] =  3-0  =  3   → NOT in range
  (0,3): prefix[3]-prefix[0] =  2-0  =  2   → in [-2,2] ✓
  (1,2): prefix[2]-prefix[1] =  3-(-2)= 5   → NOT in range
  (1,3): prefix[3]-prefix[1] =  2-(-2)= 4   → NOT in range
  (2,3): prefix[3]-prefix[2] =  2-3  = -1   → in [-2,2] ✓

Total = 3 matches — corresponds to subarrays [-2], [-1], [-2,5,-1] having sums -2, -1, 2, all within [-2,2].
```

The segment tree mechanically finds these same 3 pairs by, at each `j`, querying the compressed-rank range `[prefix[j]-upper, prefix[j]-lower]` against everything inserted so far — exactly the same logic, just O(log n) per query instead of O(n) brute force.

**Overflow note:** prefix sums can exceed `int` range even if `nums[i]` fits in `int` — always use `long long` for prefix sums in this family of problems.

---

## SECTION 11 — COMPLEXITY TABLE

| Operation / Variant | Build | Update | Query | Space | Notes |
|---|---|---|---|---|---|
| Recursive point-update, range-sum/min/max | O(n) | O(log n) | O(log n) | O(n) — array of size 4n | Foundation template |
| Iterative bottom-up (2N) | O(n) | O(log n) | O(log n) | O(n) — array of size 2n | No lazy propagation support |
| Lazy propagation (range-add, range-sum) | O(n) | O(log n) | O(log n) | O(n) — two arrays of size 4n | push_down is O(1) amortized per call |
| Range-assign lazy | O(n) | O(log n) | O(log n) | O(n) — three arrays (tree, lazy, hasLazy) | |
| Coordinate compression + seg tree | O(n log n) — sort dominates | O(log n) | O(log n) | O(n) | Compression itself is O(n log n) one-time cost |
| Segment tree on values (315, 327) | O(n log n) — sort dominates | O(log n) per insert | O(log n) per query | O(n) | n insertions + n queries → O(n log n) total |
| Merge sort tree | O(n log n) | — (usually static) | O(log² n) | O(n log n) | Each node stores a sorted vector |
| Persistent segment tree | O(n) initial | O(log n) per version | O(log n) per version | O(n log n) total across all versions | Each update creates O(log n) new nodes |

**Why every core operation is O(log n):** the tree has depth `⌈log₂ n⌉`. A single point update touches exactly one node per level (root-to-leaf path). A range query/update touches at most O(1) "boundary" nodes per level (the nodes where the query range partially overlaps) plus O(1) "canonical" nodes that resolve instantly — so the total work across all levels is O(log n), not O(n).

---

## SECTION 12 — VARIANTS (BRIEF)

**Persistent Segment Tree:** instead of modifying nodes in place, every update creates **new** nodes only along the root-to-leaf path that changed (O(log n) new nodes), while reusing every untouched subtree from the previous version. This gives you access to *every historical version* of the array simultaneously — "what was the sum of `[l,r]` after the 5th update?" is answerable in O(log n) by just querying the root pointer saved at version 5. Used for problems like "K-th smallest in a range" (persistent segment tree of prefix frequency counts, one version per array prefix) and offline/online range-rank queries where you can't just process everything in one pass. The core mechanic: `update` returns a **new root** instead of mutating; children pointers are shared (not copied) unless they lie on the changed path.

**Merge Sort Tree:** each node stores a **sorted copy** of its range instead of a single aggregate value, built the same way merge sort builds a sorted array (merge two sorted children). A range query decomposes into O(log n) canonical nodes as usual, but now each node answers "how many elements ≤ x" via binary search in O(log(node size)), giving O(log² n) per query. Useful for "count values ≤ x in range [l, r]" when you can't afford full coordinate-compression-plus-offline-processing, or when queries arrive online and can't be reordered. Costs O(n log n) space instead of O(n) — the sorted-vector storage is the price of the extra flexibility.

**Dynamic / Sparse Segment Tree:** when the value range is enormous (e.g. `[0, 10^9]` calendar dates in My Calendar III, LC 732) but the number of actual updates is small, don't pre-allocate `4*n` — allocate tree nodes **lazily**, only when a path is actually visited, using explicit left/right child pointers (or indices into a dynamically-growing array) instead of the `2*node`/`2*node+1` convention. Each of the O(q) operations creates at most O(log(range)) new nodes, so total space is O(q log(range)) instead of O(range). This is the standard alternative to coordinate compression when queries arrive online (you don't know all the coordinates up front) — My Calendar I/II/III are the canonical examples.

---

## SECTION 13 — COMMON MISTAKES

### Mistake 1: Array sizing — 2N vs 4N

```cpp
// WRONG — sizing the recursive tree array as 2*n
vector<long long> tree(2 * n);   // BUG: this is the size for the ITERATIVE bottom-up
                                  // tree convention, not the recursive top-down one!
build(1, 0, n-1);  // will write out of bounds for non-power-of-2 n

// CORRECT — recursive top-down tree needs 4*n
vector<long long> tree(4 * n);
// (2*n is only safe for the ITERATIVE bottom-up layout from Section 5,
//  which uses a completely different indexing scheme — leaves at [n, 2n-1].
//  Do NOT mix the two conventions.)
```

### Mistake 2: Forgetting push_down before recursing

```cpp
// WRONG — recurses into children without resolving this node's pending lazy first
long long query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return 0;
    if (ql <= l && r <= qr) return tree[node];
    int mid = l + (r - l) / 2;
    // BUG: never called push_down(node, l, r) — children still hold STALE
    // values if this node has a pending lazy add/assign from an earlier update!
    return query(2*node, l, mid, ql, qr) + query(2*node+1, mid+1, r, ql, qr);
}

// CORRECT
long long query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return 0;
    if (ql <= l && r <= qr) return tree[node];
    push_down(node, l, r);          // resolve pending lazy BEFORE trusting children's values
    int mid = l + (r - l) / 2;
    return query(2*node, l, mid, ql, qr) + query(2*node+1, mid+1, r, ql, qr);
}
```

### Mistake 3: Lazy not cleared (or wrongly combined for assign-type lazy)

```cpp
// WRONG — additive lazy that forgets to clear itself after pushing
void push_down(int node, int l, int r) {
    int mid = l + (r - l) / 2;
    tree[2*node]   += lazy[node] * (mid - l + 1);
    tree[2*node+1] += lazy[node] * (r - mid);
    lazy[2*node]   += lazy[node];
    lazy[2*node+1] += lazy[node];
    // BUG: forgot `lazy[node] = 0;` — this node's pending add will be
    // APPLIED AGAIN the next time push_down runs on it, double-counting.
}

// WRONG (the other direction) — assign-type lazy using += instead of = for the child
void push_down(int node, int l, int r) {
    if (!hasLazy[node]) return;
    lazy[2*node] += lazy[node];   // BUG: should be `=`. Assignment must OVERWRITE
                                    // the child's pending value, not add to it —
                                    // "set to 5" after "set to 3" means "set to 5", not "set to 8".
    hasLazy[2*node] = true;
    hasLazy[node] = false;
}

// CORRECT (additive lazy)
void push_down(int node, int l, int r) {
    int mid = l + (r - l) / 2;
    tree[2*node]   += lazy[node] * (mid - l + 1);
    tree[2*node+1] += lazy[node] * (r - mid);
    lazy[2*node]   += lazy[node];
    lazy[2*node+1] += lazy[node];
    lazy[node] = 0;                 // MUST clear — it has now been fully delegated
}
```

### Mistake 4: Midpoint overflow

```cpp
// WRONG — classic overflow bug (also a famous binary-search bug)
int mid = (l + r) / 2;
// If l and r are both close to INT_MAX (rare for array indices, but common
// when the segment tree's index axis represents raw COORDINATES up to 10^9,
// e.g. dynamic segment trees for My Calendar III without compression),
// l + r can overflow a 32-bit int and wrap to a negative number.

// CORRECT — always compute midpoint this way, no exceptions
int mid = l + (r - l) / 2;
```

### Mistake 5: Wrong merge identity (0 for sum, INT_MAX for min, INT_MIN for max)

```cpp
// WRONG — using 0 as the "no overlap" identity for a MIN segment tree
int query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return 0;   // BUG: 0 might be SMALLER than the true
                                        // minimum of the queried range, corrupting
                                        // the min() combine at the caller.
    ...
    return min(query(left...), query(right...));
}

// CORRECT — identity must satisfy merge(identity, x) == x for ALL x
// sum:  identity = 0
// min:  identity = INT_MAX  (or LLONG_MAX for long long trees)
// max:  identity = INT_MIN  (or LLONG_MIN)
// gcd:  identity = 0        (gcd(0, x) == x)
// xor:  identity = 0
// and:  identity = -1       (all bits set — the identity for bitwise AND)
```

### Mistake 6: Pulling the parent's aggregate up BEFORE both children are finalized

```cpp
// WRONG — computes the parent's new value using a stale child
void updateRange(int node, int l, int r, int ql, int qr, int val) {
    if (qr < l || r < ql) return;
    if (ql <= l && r <= qr) { tree[node] += val*(r-l+1); lazy[node]+=val; return; }
    push_down(node, l, r);
    int mid = l + (r - l) / 2;
    updateRange(2*node, l, mid, ql, qr, val);
    tree[node] = tree[2*node] + tree[2*node+1];   // BUG: tree[2*node+1] hasn't
                                                     // been updated yet — this
                                                     // reads a STALE right-child value!
    updateRange(2*node+1, mid+1, r, ql, qr, val);
}

// CORRECT — recurse into BOTH children first, THEN pull up
void updateRange(int node, int l, int r, int ql, int qr, int val) {
    if (qr < l || r < ql) return;
    if (ql <= l && r <= qr) { tree[node] += val*(r-l+1); lazy[node]+=val; return; }
    push_down(node, l, r);
    int mid = l + (r - l) / 2;
    updateRange(2*node, l, mid, ql, qr, val);
    updateRange(2*node+1, mid+1, r, ql, qr, val);
    tree[node] = tree[2*node] + tree[2*node+1];   // pull up only after BOTH children are final
}
```

---

## SECTION 14 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Range Sum Query - Mutable | 307 | Foundation recursive segment tree, point update + range sum |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 2 | Count of Smaller Numbers After Self | 315 | Segment tree on compressed VALUES, right-to-left insertion |
| 3 | Count of Range Sum | 327 | Segment tree on compressed PREFIX SUMS, range-count query |
| 4 | Falling Squares | 699 | Coordinate compression + range-assign + range-max lazy |
| 5 | My Calendar I / II / III | 729 / 731 / 732 | Dynamic/sparse segment tree OR offline compression; progressively harder overlap-count constraints |

### STRETCH (solve in ≤ 40 minutes)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Reverse Pairs | 493 | Segment tree (or merge sort tree) on values, `nums[i] > 2*nums[j]` distance constraint |
| 7 | The Skyline Problem | 218 | Coordinate compression + segment tree range-max, sweep-line reconstruction |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 8 | Rectangle Area II | 850 | Coordinate compression + sweep line + segment tree tracking "covered length" |
| 9 | Longest Increasing Subsequence II | 2407 | Segment tree range-max query + point update over the VALUE axis, with a `[val-k, val-1]` window constraint |
| 10 | Range Module | 715 | Lazy range-assign segment tree over compressed/dynamic coordinates, tracking covered/uncovered intervals |

---

## SECTION 15 — PATTERN CONNECTIONS

1. **Pattern 28 (Fenwick Tree / BIT):** For pure point-update + prefix-sum-query problems (exactly what LC 315 and LC 327 need), a Fenwick tree is a simpler, shorter, faster alternative to a segment tree — same O(log n) complexity, roughly a third of the code, no recursion. The trade-off: Fenwick trees are naturally suited to prefix aggregates (sum, XOR) and are awkward for range-min/max or lazy propagation. Rule of thumb: if you only ever need "point update, prefix/range sum query," reach for Fenwick first; the moment you need range MIN/MAX, or range update + range query with lazy propagation, or a non-invertible operation, segment tree is the right (and often only) tool.

2. **Pattern 06 (Prefix Sums):** A segment tree with only point updates and range-sum queries is, conceptually, a "prefix sum array that supports fast updates." Plain prefix sums answer range-sum queries in O(1) but need O(n) to update after a single element changes; a segment tree trades that O(1) query for O(log n) query in exchange for O(log n) updates instead of O(n). Recognize the trade-off explicitly in interviews: "if the array were static, I'd use prefix sums; since it's mutable, I need a segment tree (or Fenwick tree)."

3. **Pattern 21 (LIS Family):** LIS II (LC 2407, in this problem set) is a direct generalization of the O(n log n) patience-sorting LIS algorithm from Pattern 21. Classic LIS uses binary search over a `tails[]` array because the "find the best previous dp value" step is a simple monotonic search; LIS II adds a distance constraint (`nums[j] - k <= nums[i] <= nums[j]`) that a simple binary search can't handle, but a segment tree range-max query over the value axis handles it directly in O(log(maxVal)) per element.

4. **Pattern 04 (Monotonic Stack):** The Skyline Problem (LC 218) and Largest Rectangle in Histogram are both "sweep across x-axis, track a running extremum" problems. Monotonic stack solves it in O(n) when the sweep only needs LIFO extremum tracking; segment tree solves the same class of problem in O(n log n) when the update/query pattern is more general (arbitrary interval overlaps, not just a simple stack push/pop order) — Skyline is often first attempted with a segment tree before students learn the more specialized O(n log n) multiset/sweep solution.

5. **Pattern 18 (2D DP / Grid):** Rectangle Area II (LC 850) is a 1D sweep-line problem that, if attempted as a direct 2D grid coordinate-compression DP, becomes an O(n²) coordinate grid; a segment tree sweep collapses one dimension to O(log n) per event, turning it into O(n log n) overall. Recognizing "this 2D geometry problem is really a 1D sweep plus a segment tree tracking coverage" is a key skill transfer from grid-DP thinking to sweep-line thinking.

---

## SECTION 16 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 307 — Range Sum Query - Mutable

**Interviewer:** "Design a data structure that supports updating a single array element and querying the sum of any subrange, both efficiently."

**Candidate:**

*[First minute — rule out the naive options]*
> "If updates are rare and queries are frequent, prefix sums give O(1) query but O(n) update, since one changed element shifts every downstream prefix sum. If queries are rare and updates frequent, no precomputation, O(n) query O(1) update. Since the problem needs both to be fast, I need something better than either extreme — a segment tree gives O(log n) for both."

*[State the invariant before coding]*
> "Each node in the tree represents a range `[l, r]` and stores the sum of that range. A leaf is a single array position. An internal node's value is always the sum of its two children — that invariant must hold after every update."

*[Code — 6 minutes, the Section 3 template]*

*[Interviewer]: "Why size the array 4*n instead of exactly 2*n?"*
> "2n is the exact size needed only when n is a power of two, in the bottom-up iterative layout. For the recursive top-down layout, when n isn't a power of two, the tree can be unbalanced enough that 2n isn't a safe bound — 4n is the standard safe over-allocation that always works regardless of n's factorization, and the constant-factor waste is irrelevant."

*[Interviewer]: "What if I need range MINIMUM instead of range sum — how much code changes?"*
> "Two lines: the merge in `build`/`update` becomes `min(...)` instead of `+`, and the 'no overlap' return value in `query` becomes `INT_MAX` instead of `0`, since that's the identity element for min. Everything else — the three-case overlap logic, the recursion structure — is unchanged."

**RED FLAGS:**
- Reaching for a segment tree without first stating why prefix sums fail (interviewer wants to hear the trade-off reasoning, not just "I know this structure")
- Using `(l + r) / 2` for the midpoint
- Not being able to state the invariant ("parent = merge of children") when asked

---

### Interview Simulation 2: LC 699 — Falling Squares

**Interviewer:** "Squares fall onto a number line and stack on whatever's beneath them. After each square lands, report the current tallest stack. Coordinates can be up to 10^9."

**Candidate:**

*[Identify the core difficulty first]*
> "Two things make this hard: coordinates go up to 10^9, so I can't index an array or segment tree by raw x-coordinate directly. And after a square lands, the height across its footprint becomes flat — so I need range-assign, not range-add."

*[State the compression strategy]*
> "Since there are only n squares, there are at most 2n distinct coordinates that ever matter — each square's left edge and right edge. I compress those into ranks 0..m-1, and build the segment tree over that compressed range instead of the raw coordinate range."

*[State the per-square algorithm]*
> "For each square: query the current max height across its compressed footprint — call it `base`. The square's new top is `base + size`. Then I range-assign that entire footprint to `base + size`, since after landing, that whole span is flat. Track a running max across all squares processed so far for the answer array."

*[Interviewer]: "Walk through why range-ASSIGN, not range-ADD, is correct here."*
> "Range-add would be wrong because it accumulates — if two squares later overlap the same slice again, add would keep piling arbitrary deltas instead of reflecting the true flat height. Assign correctly models 'this whole footprint now has exactly this height,' which is physically what happens when a square lands and creates a flat surface."

*[Interviewer]: "Why `right = idx(left+size) - 1` and not `idx(left+size)` directly?"*
> "The right edge is exclusive on the real number line — a square from x=1 with size=2 covers `[1,3)`, i.e. real coordinates 1 and 2, not 3. The compressed index of coordinate 3 marks the START of whatever comes after this square, so the last compressed slot this square actually covers is one less than that."

**RED FLAGS:**
- Trying to build a segment tree over the raw `[0, 10^9]` coordinate range without compression (immediate memory/TLE failure)
- Using range-add instead of range-assign
- Off-by-one on the right-edge compression (a very common bug in this specific problem)

---

### Interview Simulation 3: Meta-discussion — When do you reach for a segment tree vs. simpler tools?

**Interviewer:** "You could solve a lot of range problems with prefix sums, a Fenwick tree, or a segment tree. How do you decide?"

**Candidate:**
> "I ask three questions in order:
>
> First — is the array static (no updates after the initial build)? If yes, prefix sums give O(1) query with zero ongoing update cost. No tree needed at all.
>
> Second — if updates exist, is the operation invertible and does the tree only need point-update + prefix/range aggregate queries (sum, XOR)? If yes, a Fenwick tree is simpler and faster in practice — same asymptotic complexity as a segment tree but roughly a third the code and no recursion overhead.
>
> Third — do I need any of: range MIN/MAX (non-invertible, Fenwick can't do it cleanly), range UPDATE + range QUERY (needs lazy propagation), or a non-trivial merge like 'sorted list of elements in this range' (merge sort tree)? Any of those pushes me to a full segment tree, because that's the only structure in this family flexible enough to support them.
>
> There's a fourth axis too — is the value/coordinate range huge (like 10^9) with only a few actual updates? Then either coordinate-compress everything up front (if I know all queries in advance — offline), or use a dynamic/sparse segment tree that only allocates nodes as visited (if queries arrive online and I can't precompute the coordinate set)."

**RED FLAGS:**
- Defaulting to segment tree for every range problem without considering that prefix sums or Fenwick might be simpler and sufficient
- Not knowing the offline-compression vs. online-dynamic-tree distinction when coordinates are unbounded

---

## SECTION 17 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why is the safe array size for a recursive segment tree `4*n` instead of `2*n`?**

> The recursive tree has depth `⌈log₂ n⌉ + 1`. If `n` is a power of two, a tightly packed tree needs exactly `2n` slots (same as the iterative bottom-up layout). But when `n` is NOT a power of two, the recursive `build` function still descends using `mid = l + (r-l)/2`, which can produce an unbalanced tree shape where some leaves are one level deeper than others. In the worst case, the highest node index used can approach `4n`. `4n` is the standard, always-safe bound that avoids doing a tight analysis for every `n` — the constant-factor memory waste (2x in most cases) is irrelevant next to the guarantee of never indexing out of bounds.

---

**Q2: In lazy propagation, why must `push_down` happen BEFORE recursing into children, never after?**

> `push_down` resolves a node's pending operation by applying it to the children's stored aggregate AND passing the pending flag down to them. If you recurse into children first and call `push_down` afterward, you'd be reading/writing the children's values while they're still stale (missing the pending update from this node) — corrupting the query result or the update. The invariant that must hold at all times is: "a node's `tree[]` value is always correct for its own range; a node's CHILDREN's `tree[]` values are correct only if the node's `lazy[]` is empty." The instant you're about to look inside a node's children (partial-overlap case), you must first restore that invariant for the children by pushing down.

---

**Q3: Why does additive lazy use `+=` when pushing to a child's lazy slot, while assignment lazy uses `=`?**

> Additive lazy represents "add this delta." Two pending adds compose: "add 3, then add 5" = "add 8" — so combining requires summing (`+=`). Assignment lazy represents "set to this value." Two pending assigns do NOT compose additively: "set to 3, then set to 5" is just "set to 5" — the earlier assignment is completely irrelevant once a later one arrives, so the child's pending value must be **overwritten** (`=`), not summed. Using `+=` for assignment-lazy would corrupt the semantics entirely (you'd get nonsensical accumulated "assigned" values that don't correspond to any real state).

---

**Q4: What are the three overlap cases in a segment tree query/update, and what happens in each?**

> 1. **No overlap** (`qr < l || r < ql`): return the identity element immediately (0 for sum, INT_MAX for min, etc.) for query; do nothing for update.
> 2. **Total overlap** (`ql <= l && r <= qr`): the current node's range is entirely inside the query/update range — return `tree[node]` directly for query, or update `tree[node]` and set `lazy[node]` directly for update, without recursing further.
> 3. **Partial overlap** (neither of the above): `push_down` (if this is a lazy tree) and recurse into both children, then combine their results (query) or pull the parent's value up from the now-updated children (update).
>
> This three-way split is why every operation is O(log n): at each depth of the recursion, at most 2 nodes are in the "partial overlap" state (the two boundary nodes on each side of the query range), so the total work across `O(log n)` depths is `O(log n)`.

---

**Q5: For Falling Squares (LC 699), why is coordinate compression necessary, and what exactly gets compressed?**

> Coordinates in the problem can be up to `10^9`, but there are only `n` squares, meaning at most `2n` coordinates (each square's left edge and right edge) that will ever be queried or updated. A segment tree indexed by raw coordinate would need `O(10^9)` space — infeasible. Instead, we collect exactly those `2n` coordinates, sort and dedupe them into an array of `m ≤ 2n` values, and map every real coordinate to its rank (position) in that sorted array via binary search. The segment tree is then built over the small range `[0, m-1]`, and every square's footprint is translated to a compressed index range before querying/updating.

---

**Q6: In "Count of Smaller Numbers After Self" (LC 315), why process the array right-to-left instead of left-to-right?**

> The goal is: for each `nums[i]`, count elements to its RIGHT that are smaller. If we process right-to-left and maintain a frequency segment tree of "values inserted so far," then at the moment we're about to process `nums[i]`, everything already in the tree came from indices `> i` — i.e., everything to the right. So querying "how many inserted values have rank < rank(nums[i])" at that exact moment directly answers the question for index `i`. Processing left-to-right would instead naturally answer "how many elements to the LEFT are smaller" (a different, though related, problem — that's the mirror version used in counting inversions).

---

**Q7: When would you choose a Fenwick tree over a segment tree, and when is a segment tree unavoidable?**

> Choose Fenwick (BIT) when: the operation is invertible or you only need prefix aggregates (sum, XOR), updates are point updates, and you don't need range MIN/MAX or lazy propagation. Fenwick trees are shorter, faster in practice, and easier to get right under time pressure — this is most of LC 315/327-style "point update, prefix query" problems.
>
> Segment tree is unavoidable when: you need range MIN or MAX (not invertible — you can't "subtract" a min the way you can subtract a sum), you need range UPDATE combined with range QUERY (requires lazy propagation — Fenwick's "range update, range query" tricks exist but are far more fragile and limited to sum-like operations), or you need a non-scalar merge (like "sorted list of this range," i.e. a merge sort tree) or a persistent multi-version structure. In short: Fenwick for "sum/XOR + point update + prefix query," segment tree for everything else in this family.

---

## SECTION 18 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. THE THREE-CASE OVERLAP SKELETON (memorize this shape)
// ─────────────────────────────────────────────────────────────────
T query(int node, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) return IDENTITY;          // no overlap
    if (ql <= l && r <= qr) return tree[node];       // total overlap
    push_down(node, l, r);                            // (only if lazy tree) partial overlap
    int mid = l + (r - l) / 2;                        // NEVER (l+r)/2
    return MERGE(query(2*node, l, mid, ql, qr),
                 query(2*node+1, mid+1, r, ql, qr));
}

// ─────────────────────────────────────────────────────────────────
// 2. SIZING
// ─────────────────────────────────────────────────────────────────
// Recursive top-down (1-indexed, node -> 2*node, 2*node+1):  tree.assign(4*n, ...)
// Iterative bottom-up (leaves at [n, 2n-1]):                  tree.assign(2*n, ...)
// NEVER mix these two indexing conventions in one structure.

// ─────────────────────────────────────────────────────────────────
// 3. IDENTITY ELEMENTS
// ─────────────────────────────────────────────────────────────────
// sum:  0            min: INT_MAX / LLONG_MAX      max: INT_MIN / LLONG_MIN
// gcd:  0             xor: 0                         and: -1 (all bits set)

// ─────────────────────────────────────────────────────────────────
// 4. LAZY PROPAGATION SKELETON (range-add + range-sum)
// ─────────────────────────────────────────────────────────────────
void push_down(int node, int l, int r) {
    if (lazy[node] == 0) return;
    int mid = l + (r - l) / 2;
    tree[2*node]   += lazy[node] * (mid - l + 1);
    tree[2*node+1] += lazy[node] * (r - mid);
    lazy[2*node]   += lazy[node];      // ACCUMULATE for additive lazy
    lazy[2*node+1] += lazy[node];
    lazy[node] = 0;                     // ALWAYS clear after pushing
}
void updateRange(int node, int l, int r, int ql, int qr, int val) {
    if (qr < l || r < ql) return;
    if (ql <= l && r <= qr) {
        tree[node] += (long long)val * (r - l + 1);
        lazy[node] += val;
        return;
    }
    push_down(node, l, r);              // BEFORE recursing
    int mid = l + (r - l) / 2;
    updateRange(2*node, l, mid, ql, qr, val);
    updateRange(2*node+1, mid+1, r, ql, qr, val);
    tree[node] = tree[2*node] + tree[2*node+1];   // AFTER both children updated
}

// ─────────────────────────────────────────────────────────────────
// 5. RANGE-ASSIGN LAZY DIFFERENCE (overwrite, not accumulate)
// ─────────────────────────────────────────────────────────────────
// push_down: tree[child] = lazy[node] * len;  lazy[child] = lazy[node];  (=, not +=)
// Needs an explicit `hasLazy` boolean flag array — 0 may be a valid assign value.

// ─────────────────────────────────────────────────────────────────
// 6. COORDINATE COMPRESSION BOILERPLATE
// ─────────────────────────────────────────────────────────────────
vector<int> xs = /* collect every coordinate that will be touched */;
sort(xs.begin(), xs.end());
xs.erase(unique(xs.begin(), xs.end()), xs.end());
auto idx = [&](int x) { return lower_bound(xs.begin(), xs.end(), x) - xs.begin(); };
// Build segment tree over [0, xs.size()-1] using idx(x) instead of raw x.

// ─────────────────────────────────────────────────────────────────
// 7. SEGMENT TREE ON VALUES (inversion counting family)
// ─────────────────────────────────────────────────────────────────
// 1. Compress values (or prefix sums) to dense ranks.
// 2. Segment tree over [0, m-1] storing FREQUENCY counts (starts all 0).
// 3. Process elements in the order the problem's "count pairs" logic needs
//    (often right-to-left for "smaller to the right" style problems).
// 4. For each element: query the rank-range that answers this element's
//    contribution, THEN insert (point-update ++) this element's own rank.

// ─────────────────────────────────────────────────────────────────
// 8. ITERATIVE BOTTOM-UP (no lazy — point update, range query only)
// ─────────────────────────────────────────────────────────────────
for (int i = 0; i < n; i++) tree[n+i] = arr[i];
for (int i = n-1; i >= 1; i--) tree[i] = tree[2*i] + tree[2*i+1];
// query [l, r) half-open:
long long q(int l, int r) {
    long long res = 0;
    for (l += n, r += n; l < r; l /= 2, r /= 2) {
        if (l & 1) res += tree[l++];
        if (r & 1) res += tree[--r];
    }
    return res;
}
```

---

## SECTION 19 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 10 problems in order.** Falling Squares and My Calendar III are the two that require the most careful thinking about compression vs. dynamic allocation. Budget 45 minutes each for those two.

2. **Rebuild the lazy propagation template from memory, twice, on different days.** The single highest-leverage drill for this pattern is: close this document, implement range-add + range-sum lazy propagation from scratch, then compare line-by-line against Section 6. Do this until you make zero mistakes.

3. **Reimplement LC 315 with a Fenwick tree** after finishing it with a segment tree. This cements the Pattern 27 ↔ Pattern 28 connection and shows you directly how much shorter the Fenwick solution is for this specific problem shape.

4. **Convert one range-add lazy tree into a range-assign lazy tree, and vice versa,** without looking at Section 6/7. This is the fastest way to internalize the `+=` vs `=` distinction from Mental Model Checkpoint Q3.

5. **Preview Pattern 28 (Fenwick Tree):** once segment tree is solid, Fenwick tree will feel almost trivial — it's a compressed, non-recursive special case optimized for the sum/prefix-query scenario. Understanding segment tree deeply first makes Fenwick's bit-manipulation tricks (`i & (-i)`) much easier to trust rather than memorize blindly.

---

## SECTION 20 — SIGN-OFF CRITERIA

### Tier 1 — Foundation mastered
You implement the recursive point-update/range-sum segment tree (Section 3) from scratch, correctly, in under 10 minutes, without referencing this document. You can explain the three overlap cases and why each is necessary for O(log n) complexity.

### Tier 2 — Variants understood
You adapt the foundation template to range-min and range-max by changing only the merge and identity (Section 4). You can implement the iterative bottom-up tree (Section 5) and explain when it's preferable to the recursive version, and when it isn't (no lazy propagation support).

### Tier 3 — Lazy propagation solid
You implement range-add + range-sum lazy propagation (Section 6) from scratch, correctly, including `push_down` in the right place, in under 20 minutes. You can trace through a range-update-then-query sequence by hand and correctly predict which nodes end up with pending lazy values. You understand and can explain the `+=` vs `=` distinction between additive and assignment lazy (Section 7).

### Tier 4 — Segment tree on values fluent
You solve Count of Smaller Numbers After Self (LC 315) and Count of Range Sum (LC 327) using the "compress values, build a frequency segment tree, query before inserting" pattern, and can explain why the processing order (right-to-left, or prefix-sum order) matters. You solve Falling Squares (LC 699) with correct coordinate compression and range-assign lazy, including the `right = idx(...) - 1` boundary handling.

### Tier 5 — Contest-ready
You solve at least 8 of the 10 problems, including the Contest tier (Rectangle Area II, LIS II, Range Module). You can articulate, unprompted, when to reach for a Fenwick tree instead of a full segment tree (Pattern Connection 1), and when a huge coordinate range demands a dynamic/sparse segment tree instead of offline compression (My Calendar III).

**Sign-off threshold:** Solve 7 of 10 problems. Mandatory: LC 307 (foundation — must be typo-free from memory), LC 315 or LC 327 (segment tree on values — pick one, must fully explain the compression + right-to-left/ordered-insertion logic), and LC 699 (coordinate compression + lazy range-assign — the canonical "everything comes together" problem in this pattern).

---

## REFERENCES

- CP-Algorithms — Segment Tree: https://cp-algorithms.com/data_structures/segment_tree.html
- Al.Cash, "Efficient and easy segment trees" (Codeforces blog — the canonical iterative bottom-up template): https://codeforces.com/blog/entry/18051

---

*Pattern 27 Complete — Segment Tree*
*Next: Pattern 28 — Fenwick Tree (Binary Indexed Tree) — the compressed, non-recursive alternative for point-update/prefix-query problems*
