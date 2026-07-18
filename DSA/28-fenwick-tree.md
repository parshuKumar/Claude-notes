# PATTERN 28 — FENWICK TREE (BINARY INDEXED TREE)
## Point Update, Range Query, Difference Tricks, 2D BIT, Order Statistics — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The entire idea, in ten lines:**

```cpp
vector<long long> bit(n + 1, 0);                 // 1-indexed, size n+1

void update(int i, long long delta) {            // add delta at position i
    for (; i <= n; i += i & (-i)) bit[i] += delta;
}

long long query(int i) {                          // sum of [1..i]
    long long s = 0;
    for (; i > 0; i -= i & (-i)) s += bit[i];
    return s;
}
```

That's it. No node structs, no recursion, no left/mid/right splitting, no lazy propagation. A Fenwick Tree (Binary Indexed Tree, "BIT") is a flat array that answers **point update + prefix query** in O(log n), built on a single bit trick: `i & (-i)`.

**Why a BIT and not a segment tree?**

Segment trees are strictly more general — they handle range min/max, range assignment, lazy propagation, non-invertible operations. But that generality costs you: a node struct or 4n-sized array, recursive build/update/query, and lazy-propagation bookkeeping that is a notorious bug magnet.

A BIT wins specifically when:
- The operation is **invertible** (has a well-defined inverse) — sum, XOR, product-mod-p. NOT min/max (you can't "un-min" a value).
- You need **point update + prefix/range query** — no range assignment, no range min/max.
- You want **5x less code**, a **smaller constant factor**, and an **iterative** implementation with zero recursion overhead.

If you need range-min-query with point updates, or lazy range-assignment, reach for Pattern 27 (Segment Tree) instead. If you need dynamic prefix sums, inversions, order statistics, or frequency counting — BIT is the right tool, every time.

**The core mental model:** A BIT is "prefix sums, but updatable." A plain prefix-sum array (Pattern 06) gives O(1) range query but O(n) update (shifting all sums downstream). A BIT gives O(log n) for both, by storing partial sums cleverly instead of full prefix sums.

---

## SECTION 2 — THE CORE INSIGHT: LOW-BIT DECOMPOSITION

### The bit trick

`i & (-i)` isolates the **lowest set bit** of `i` (its value, not its position). In two's complement, `-i` is `~i + 1`. Flipping all bits and adding 1 means every bit below the lowest set bit of `i` becomes 1 in `-i`, and ANDing with `i` cancels everything except that lowest set bit.

```
i = 12 = 0b1100
-i     = 0b...110100   (two's complement)
i & -i = 0b0100 = 4    ← lowest set bit, as a VALUE
```

| i (binary) | i & (-i) | meaning |
|---|---|---|
| 6 = 0b0110 | 2 | lowest set bit is the "2s" bit |
| 8 = 0b1000 | 8 | lowest set bit IS the whole number (power of 2) |
| 12 = 0b1100 | 4 | lowest set bit is the "4s" bit |
| 7 = 0b0111 | 1 | odd numbers always give 1 |

### What `tree[i]` actually stores

`tree[i]` stores the sum of a range of the original array: **`(i - lowbit(i), i]`** — i.e., `lowbit(i)` elements ending at position `i`.

```
i=1 (lowbit=1): tree[1] = sum(a[1..1])          — 1 element
i=2 (lowbit=2): tree[2] = sum(a[1..2])          — 2 elements
i=3 (lowbit=1): tree[3] = sum(a[3..3])          — 1 element
i=4 (lowbit=4): tree[4] = sum(a[1..4])          — 4 elements
i=5 (lowbit=1): tree[5] = sum(a[5..5])          — 1 element
i=6 (lowbit=2): tree[6] = sum(a[5..6])          — 2 elements
i=7 (lowbit=1): tree[7] = sum(a[7..7])          — 1 element
i=8 (lowbit=8): tree[8] = sum(a[1..8])          — 8 elements
```

### Diagram for n = 8

```
Index:        1     2     3     4     5     6     7     8
Coverage:   [1,1] [1,2] [3,3] [1,4] [5,5] [5,6] [7,7] [1,8]
Size:         1     2     1     4     1     2     1     8

Tree structure (indentation = "nested inside"):

tree[8] ─────────────────────────── covers [1,8]
  ├── tree[4] ──────── covers [1,4]
  │     ├── tree[2] ── covers [1,2]
  │     │     └── tree[1] covers [1,1]
  │     └── tree[3] covers [3,3]
  └── tree[6] ──────── covers [5,6]
        ├── tree[5] covers [5,5]
        └── tree[7] covers [7,7]
```

### Two directions, two purposes

**UPDATE (propagate to parents):** `i += i & (-i)` — moves to the next range that CONTAINS position `i`. This walks UP the "contains" chain in O(log n) steps.

```
update(3): 3 → 3+lowbit(3)=3+1=4 → 4+lowbit(4)=4+4=8 → 8+8=16 (>8, stop)
Touches: tree[3], tree[4], tree[8]
Check: tree[3]=[3,3] ✓ contains 3.  tree[4]=[1,4] ✓ contains 3.  tree[8]=[1,8] ✓ contains 3.
```

**QUERY (walk down, collecting disjoint blocks):** `i -= i & (-i)` — moves to the range immediately BEFORE the current block. This walks DOWN in O(log n) steps, decomposing `[1..i]` into at most `log2(n)` disjoint ranges.

```
query(7): 7 → 7-lowbit(7)=7-1=6 → 6-lowbit(6)=6-2=4 → 4-4=0 (stop)
Touches: tree[7], tree[6], tree[4]
Ranges:  [7,7] + [5,6] + [1,4]  =  [1,7]  ✓ exactly covers 1..7, no overlap, no gap
```

**Why O(log n)?** Every step in `update` sets one more bit of `i` to 1 (bounded by log n bits total). Every step in `query` clears the lowest set bit of `i` (also bounded by log n bits). Both loops terminate in at most `⌊log2 n⌋ + 1` iterations.

---

## SECTION 3 — TEMPLATE 1: BASIC BIT (POINT UPDATE + PREFIX QUERY)

```cpp
// ─────────────────────────────────────────────────────────────────
// The canonical Fenwick Tree — 1-indexed, point update, prefix query.
// ─────────────────────────────────────────────────────────────────
class Fenwick {
public:
    int n;
    vector<long long> tree;

    Fenwick(int n) : n(n), tree(n + 1, 0) {}

    // Add `delta` to position i (1-indexed)
    void update(int i, long long delta) {
        for (; i <= n; i += i & (-i))
            tree[i] += delta;
    }

    // Sum of a[1..i] (1-indexed, inclusive)
    long long query(int i) {
        long long s = 0;
        for (; i > 0; i -= i & (-i))
            s += tree[i];
        return s;
    }

    // Build from an existing array in O(n) instead of O(n log n)
    // (n calls to update would be O(n log n); this is the direct O(n) build)
    void build(vector<long long>& a) {                 // a is 1-indexed, size n+1
        for (int i = 1; i <= n; i++) {
            tree[i] += a[i];
            int parent = i + (i & (-i));
            if (parent <= n) tree[parent] += tree[i];
        }
    }
};
```

**FULL TRACE — build + update + query.**

Array (1-indexed): `a = [_, 3, 2, -1, 6, 5, 4, -3, 3]` for positions 1..8.

Building via 8 individual `update` calls:

```
update(1, 3):  i=1: tree[1]+=3=3   i=2: tree[2]+=3=3   i=4: tree[4]+=3=3   i=8: tree[8]+=3=3   i=16>8 stop
update(2, 2):  i=2: tree[2]+=2=5   i=4: tree[4]+=2=5   i=8: tree[8]+=2=5   i=16>8 stop
update(3,-1):  i=3: tree[3]+=-1=-1 i=4: tree[4]+=-1=4  i=8: tree[8]+=-1=4  i=16>8 stop
update(4, 6):  i=4: tree[4]+=6=10  i=8: tree[8]+=6=10  i=16>8 stop
update(5, 5):  i=5: tree[5]+=5=5   i=6: tree[6]+=5=5   i=8: tree[8]+=5=15  i=16>8 stop
update(6, 4):  i=6: tree[6]+=4=9   i=8: tree[8]+=4=19  i=16>8 stop
update(7,-3):  i=7: tree[7]+=-3=-3 i=8: tree[8]+=-3=16 i=16>8 stop
update(8, 3):  i=8: tree[8]+=3=19  i=16>8 stop

FINAL tree = [_, 3, 5, -1, 10, 5, 9, -3, 19]   (indices 1..8)
```

**Sanity check:** `tree[4]` should equal `a[1]+a[2]+a[3]+a[4] = 3+2-1+6 = 10` ✓. `tree[8]` should equal the sum of all 8 elements `= 3+2-1+6+5+4-3+3 = 19` ✓. `tree[6]` should equal `a[5]+a[6] = 5+4 = 9` ✓.

**`query(7)` — sum of a[1..7]:**

```
i=7: s += tree[7] = -3           → s = -3
i=7-1=6: s += tree[6] = 9        → s = 6
i=6-2=4: s += tree[4] = 10       → s = 16
i=4-4=0: stop

query(7) = 16
Check: 3+2-1+6+5+4-3 = 16  ✓
```

**Point update — change a[3] from -1 to 3 (delta = +4):**

```
update(3, +4):
  i=3: tree[3] += 4 → -1+4 = 3
  i=4: tree[4] += 4 → 10+4 = 14
  i=8: tree[8] += 4 → 19+4 = 23
  i=16>8 stop

New query(7):
  i=7: s += tree[7] = -3         → s = -3
  i=6: s += tree[6] = 9          → s = 6
  i=4: s += tree[4] = 14         → s = 20
  i=0: stop

query(7) = 20
Check: new a[1..7] = 3+2+3+6+5+4-3 = 20  ✓  (old sum 16 + delta 4 = 20  ✓)
```

Both `update` and `query` are O(log n). No rebuild needed after the point update — that's the entire value proposition over a plain prefix-sum array.

---

## SECTION 4 — TEMPLATE 2: RANGE SUM VIA prefix(r) − prefix(l−1)

The basic BIT only gives prefix sums `[1..i]`. Arbitrary range sums `[l..r]` fall out immediately by subtraction — exactly like 1D prefix sums (Pattern 06), except now the underlying array is mutable.

```cpp
// Sum of a[l..r], 1-indexed, inclusive on both ends
long long rangeSum(Fenwick& bit, int l, int r) {
    if (l > r) return 0;
    return bit.query(r) - bit.query(l - 1);
}
```

**Why `l - 1` and not `l`:** `query(r)` = sum(1..r). `query(l-1)` = sum(1..l-1) — everything strictly BEFORE the range starts. Subtracting removes exactly the prefix `[1, l-1]`, leaving `[l, r]`.

**Trace using the tree from Section 3** (after the point update, `a = [_,3,2,3,6,5,4,-3,3]`):

```
rangeSum(3, 6) = query(6) - query(2)

query(6): i=6: tree[6]=9 → s=9;  i=6-2=4: tree[4]=14 → s=23;  i=0 stop.  query(6)=23
query(2): i=2: tree[2]=5 → s=5;  i=0 stop.  query(2)=5

rangeSum(3,6) = 23 - 5 = 18
Check: a[3]+a[4]+a[5]+a[6] = 3+6+5+4 = 18  ✓
```

**Edge case:** `rangeSum(1, r)` needs `query(0)`, which correctly returns 0 (the loop `for (; i > 0; ...)` never executes when `i = 0`). No special-casing required.

---

## SECTION 5 — TEMPLATE 3: RANGE UPDATE + POINT QUERY (DIFFERENCE BIT)

**Problem:** Instead of updating one element, you need to add `v` to every element in `[l, r]` — and then ask for the value at a single index. Doing this with the basic BIT would take O(r-l+1) individual updates. The fix: maintain a BIT over the **difference array**, not the original array.

```cpp
// d[i] = a[i] - a[i-1]  (difference array, d[1] = a[1])
// A range-add on a[l..r] becomes just TWO point-updates on d:
//     d[l]   += v      (this value and everything after shifts up by v)
//     d[r+1] -= v      (cancel the shift beyond r)
// a[i] itself is recovered as prefix_d(i) = d[1] + d[2] + ... + d[i]

class DiffFenwick {
public:
    int n;
    vector<long long> tree;

    DiffFenwick(int n) : n(n), tree(n + 1, 0) {}

    void add(int i, long long delta) {
        for (; i <= n; i += i & (-i)) tree[i] += delta;
    }

    void rangeUpdate(int l, int r, long long v) {
        add(l, v);
        if (r + 1 <= n) add(r + 1, -v);
    }

    long long pointQuery(int i) {                 // returns current a[i]
        long long s = 0;
        for (; i > 0; i -= i & (-i)) s += tree[i];
        return s;
    }
};
```

**FULL TRACE.** Start with `a = [0,0,0,0,0]` (n=5, 1-indexed). Apply `rangeUpdate(2, 4, +3)` — add 3 to a[2..4].

```
rangeUpdate(2,4,3):
  add(2, +3):  i=2: tree[2]+=3=3   i=4: tree[4]+=3=3   i=8>5 stop
  add(5, -3):  i=5: tree[5]+=-3=-3 i=6>5 stop

tree = [_, 0, 3, 0, 3, -3]

pointQuery(1): i=1: tree[1]=0 → s=0                          a[1] = 0  ✓
pointQuery(2): i=2: tree[2]=3 → s=3                          a[2] = 3  ✓
pointQuery(3): i=3: tree[3]=0 → s=0; i=2: tree[2]=3 → s=3    a[3] = 3  ✓
pointQuery(4): i=4: tree[4]=3 → s=3                          a[4] = 3  ✓
pointQuery(5): i=5: tree[5]=-3 → s=-3; i=4: tree[4]=3 → s=0  a[5] = 0  ✓

Resulting array: [0, 3, 3, 3, 0]  — exactly a[2..4] bumped by 3, a[1] and a[5] untouched.
```

**Why this works:** `d[l] += v` says "starting at l, every subsequent prefix sum is v higher." `d[r+1] -= v` cancels that bump the moment we go past r. `pointQuery(i)` is literally `prefix_d(i)`, which reconstructs `a[i]` exactly.

---

## SECTION 6 — TEMPLATE 4: RANGE UPDATE + RANGE QUERY (TWO BITs)

**Problem:** Combine both — range-add AND range-sum, both in O(log n). One difference BIT alone only gives point queries. The trick: maintain **two** BITs and derive prefix sums algebraically.

**Derivation.** Let `d[i] = a[i] - a[i-1]` as before. Then:

```
S(i) = Σ_{k=1}^{i} a[k]
     = Σ_{k=1}^{i} Σ_{j=1}^{k} d[j]              (since a[k] = Σ_{j≤k} d[j])
     = Σ_{j=1}^{i} d[j] · (i - j + 1)             (d[j] appears in a[j], a[j+1], ..., a[i] → (i-j+1) times)
     = (i+1) · Σ_{j=1}^{i} d[j]  −  Σ_{j=1}^{i} d[j]·j
     = (i+1) · B1.prefix(i)  −  B2.prefix(i)
```

where `B1` stores `d[j]` (same as Section 5) and `B2` stores `d[j] · j`.

```cpp
struct RangeBIT {
    int n;
    vector<long long> B1, B2;          // B1: raw diffs, B2: diffs weighted by index

    RangeBIT(int n) : n(n), B1(n + 1, 0), B2(n + 1, 0) {}

    void add(vector<long long>& t, int i, long long v) {
        for (; i <= n; i += i & (-i)) t[i] += v;
    }
    long long sum(vector<long long>& t, int i) {
        long long s = 0;
        for (; i > 0; i -= i & (-i)) s += t[i];
        return s;
    }

    // Add v to a[l..r], 1-indexed inclusive
    void rangeUpdate(int l, int r, long long v) {
        add(B1, l, v);            add(B1, r + 1, -v);
        add(B2, l, v * l);        add(B2, r + 1, -v * (r + 1));
    }

    long long prefixSum(int i) {
        return (long long)(i + 1) * sum(B1, i) - sum(B2, i);
    }

    long long rangeSum(int l, int r) {
        if (l > r) return 0;
        return prefixSum(r) - prefixSum(l - 1);
    }
};
```

**TRACE** — same operation as Section 5: `a = [0,0,0,0,0]`, n=5, `rangeUpdate(2,4,+3)`:

```
add(B1, 2, 3), add(B1, 5, -3)           → B1 identical to Section 5's tree
add(B2, 2, 3*2=6), add(B2, 5, -3*5=-15) → B2 tracks the weighted diffs

prefixSum(1) = 2 * B1.sum(1) - B2.sum(1) = 2*0 - 0 = 0            S(1) = 0   ✓
prefixSum(2) = 3 * B1.sum(2) - B2.sum(2) = 3*3 - 6 = 3             S(2) = 3   ✓ (a[1]+a[2]=0+3)
prefixSum(3) = 4 * B1.sum(3) - B2.sum(3) = 4*3 - 6 = 6             S(3) = 6   ✓ (0+3+3)
prefixSum(4) = 5 * B1.sum(4) - B2.sum(4) = 5*3 - 6 = 9             S(4) = 9   ✓ (0+3+3+3)
prefixSum(5) = 6 * B1.sum(5) - B2.sum(5) = 6*0 - (6-15) = 0-(-9)=9 S(5) = 9   ✓ (a[5]=0, total stays 9)

rangeSum(2,4) = prefixSum(4) - prefixSum(1) = 9 - 0 = 9
Check: a[2]+a[3]+a[4] = 3+3+3 = 9  ✓
```

Both `rangeUpdate` and `rangeSum` are O(log n) — twice the constant of a plain BIT, still asymptotically identical.

---

## SECTION 7 — TEMPLATE 5: 2D FENWICK TREE

A 2D BIT extends the same idea to a grid: `tree[i][j]` aggregates a rectangular block, and both dimensions independently walk via `+= x&(-x)` (update) or `-= x&(-x)` (query).

```cpp
class Fenwick2D {
public:
    int rows, cols;
    vector<vector<long long>> tree;

    Fenwick2D(int r, int c) : rows(r), cols(c), tree(r + 1, vector<long long>(c + 1, 0)) {}

    // Add delta at (r, c), both 1-indexed
    void update(int r, int c, long long delta) {
        for (int i = r; i <= rows; i += i & (-i))
            for (int j = c; j <= cols; j += j & (-j))
                tree[i][j] += delta;
    }

    // Sum of rectangle [1,1] .. [r,c]
    long long query(int r, int c) {
        long long s = 0;
        for (int i = r; i > 0; i -= i & (-i))
            for (int j = c; j > 0; j -= j & (-j))
                s += tree[i][j];
        return s;
    }

    // Sum of rectangle [r1,c1] .. [r2,c2] via inclusion-exclusion
    long long rangeSum(int r1, int c1, int r2, int c2) {
        return query(r2, c2) - query(r1 - 1, c2) - query(r2, c1 - 1) + query(r1 - 1, c1 - 1);
    }
};
```

**Mini-trace** for a 2×2 grid (1-indexed rows/cols 1..2), starting from all zeros, `update(1, 1, 5)`:

```
update(1,1,5): i=1(<=2): j=1(<=2): tree[1][1]+=5=5;  j=2: tree[1][2]+=5=5
               i=2(<=2): j=1: tree[2][1]+=5=5;        j=2: tree[2][2]+=5=5

query(2,2) = walk i=2→0, j=2→0: tree[2][2]=5 → total 5
rangeSum(1,1,1,1) = query(1,1) - query(0,1) - query(1,0) + query(0,0) = 5-0-0+0 = 5  ✓
```

**Complexity:** O(log(rows) · log(cols)) per update/query, O(rows · cols) space — the direct 2D analogue of the 1D case.

---

## SECTION 8 — TEMPLATE 6: COORDINATE COMPRESSION + BIT FOR ORDER STATISTICS

Most "count smaller/larger elements" problems need a BIT indexed by **value**, not by array position. But values can be arbitrarily large (up to 10^9) or negative — far too large to allocate a BIT array over directly. **Coordinate compression** maps values to dense ranks `1..m` first.

```cpp
// Standard compression recipe
vector<int> compress(vector<int>& nums) {
    vector<int> sorted(nums);
    sort(sorted.begin(), sorted.end());
    sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
    return sorted;                      // sorted[k] = the (k+1)-th smallest distinct value
}

int rankOf(vector<int>& sorted, int val) {
    // 1-indexed rank — required for BIT correctness (Section 2: index 0 breaks the loop)
    return lower_bound(sorted.begin(), sorted.end(), val) - sorted.begin() + 1;
}
```

Once every value has a dense 1-indexed rank, a BIT over ranks answers "how many inserted elements have rank ≤ k" (a count-BIT), which is exactly an **order-statistics tree**: point-insert a value, prefix-query how many values ≤ x have been inserted so far.

**Bonus: finding the k-th smallest inserted element in O(log n)** — the BIT binary-lifting trick, the direct analogue of binary search on a segment tree.

```cpp
// tree = a Fenwick tree of 0/1 counts (or general counts) over ranks 1..n
// Returns the 1-indexed rank of the k-th smallest CURRENTLY INSERTED value
int findKth(vector<long long>& tree, int n, long long k) {
    int pos = 0;
    int logn = 0;
    while ((1 << (logn + 1)) <= n) logn++;      // highest power of 2 that fits

    for (int pw = 1 << logn; pw > 0; pw >>= 1) {
        if (pos + pw <= n && tree[pos + pw] < k) {
            pos += pw;
            k -= tree[pos];
        }
    }
    return pos + 1;                              // 1-indexed rank of the k-th smallest
}
```

**Why it works:** This walks the same implicit tree structure as `update`, but greedily takes the largest jump `pw` such that the block `tree[pos+pw]` still doesn't reach `k` — mirroring binary search over the BIT's internal partial sums, in O(log n) instead of O(log² n) (which a naive "binary search + query" would cost).

This template underlies every application in Section 9: compress once, then treat the BIT as a live, insertable multiset that answers rank/count queries in O(log n).

---

## SECTION 9 — APPLICATIONS (FULL WORKED CODE)

### 9.1 — LC 315: Count of Smaller Numbers After Self

For each index `i`, count how many elements to its RIGHT are smaller than `nums[i]`. Compress values, then sweep **right to left**, inserting into the BIT as you go — so a query at index `i` only sees elements already to the right.

```cpp
class Solution {
public:
    vector<int> tree;
    int n;

    void update(int i) {
        for (; i <= n; i += i & (-i)) tree[i]++;
    }
    int query(int i) {
        int s = 0;
        for (; i > 0; i -= i & (-i)) s += tree[i];
        return s;
    }

    vector<int> countSmaller(vector<int>& nums) {
        vector<int> sorted(nums);
        sort(sorted.begin(), sorted.end());
        sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
        n = sorted.size();
        tree.assign(n + 1, 0);

        int m = nums.size();
        vector<int> ans(m);
        for (int i = m - 1; i >= 0; i--) {
            int rank = lower_bound(sorted.begin(), sorted.end(), nums[i]) - sorted.begin() + 1;
            ans[i] = query(rank - 1);      // count of already-inserted values strictly smaller
            update(rank);
        }
        return ans;
    }
};
```

**TRACE** for `nums = [5, 2, 6, 1]`. Compressed: `sorted = [1, 2, 5, 6]`, n=4.

```
i=3 (val=1, rank=1): query(0)=0            ans[3]=0    update(1): tree=[_,1,1,0,1]
i=2 (val=6, rank=4): query(3)=1            ans[2]=1    update(4): tree=[_,1,1,0,2]
i=1 (val=2, rank=2): query(1)=1            ans[1]=1    update(2): tree=[_,1,2,0,3]
i=0 (val=5, rank=3): query(2)=2            ans[0]=2    update(3)

Final: ans = [2, 1, 1, 0]   ✓ (matches LeetCode's expected output)
```

### 9.2 — LC 493: Reverse Pairs

Count pairs `(i, j)` with `i < j` and `nums[i] > 2 * nums[j]`. Same right-to-left sweep, but the threshold `nums[i] / 2.0` doesn't land on a compressed rank directly — use a small binary search over the sorted compressed values to find the count boundary, avoiding floating point entirely.

```cpp
class Solution {
public:
    vector<int> tree;
    int n;

    void update(int i) {
        for (; i <= n; i += i & (-i)) tree[i]++;
    }
    int query(int i) {
        int s = 0;
        for (; i > 0; i -= i & (-i)) s += tree[i];
        return s;
    }

    int reversePairs(vector<int>& nums) {
        vector<long long> sorted(nums.begin(), nums.end());
        sort(sorted.begin(), sorted.end());
        sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
        n = sorted.size();
        tree.assign(n + 1, 0);

        int count = 0;
        for (int i = (int)nums.size() - 1; i >= 0; i--) {
            long long target = nums[i];
            // find count of sorted[k] with 2*sorted[k] < target  (via index, no floating point)
            int lo = 0, hi = n;
            while (lo < hi) {
                int mid = lo + (hi - lo) / 2;
                if (2LL * sorted[mid] < target) lo = mid + 1;
                else hi = mid;
            }
            count += query(lo);            // ranks 1..lo satisfy the condition
            int rank = lower_bound(sorted.begin(), sorted.end(), (long long)nums[i]) - sorted.begin() + 1;
            update(rank);
        }
        return count;
    }
};
```

**TRACE** for `nums = [1, 3, 2, 3, 1]` (expected answer: 2). Compressed: `[1, 2, 3]`, n=3.

```
i=4 (val=1): lo=0 (no value satisfies 2v<1)         query(0)=0   count=0   update(rank=1)
i=3 (val=3): lo=1 (only val=1: 2*1=2<3)             query(1)=1   count=1   update(rank=3)
i=2 (val=2): lo=0 (2*1=2<2 is false)                query(0)=0   count=1   update(rank=2)
i=1 (val=3): lo=1 (val=1: 2<3 true; val=2: 4<3 false) query(1)=1  count=2   update(rank=3)
i=0 (val=1): lo=0                                    query(0)=0   count=2   update(rank=1)

Final count = 2  ✓
```

### 9.3 — LC 1649: Create Sorted Array through Instructions

Insert `instructions[i]` one at a time. Cost for each insertion = `min(count of already-inserted elements strictly less, count of already-inserted elements strictly greater)`. Sum all costs mod 1e9+7.

```cpp
class Solution {
public:
    vector<int> tree;
    int n;

    void update(int i) {
        for (; i <= n; i += i & (-i)) tree[i]++;
    }
    int query(int i) {
        int s = 0;
        for (; i > 0; i -= i & (-i)) s += tree[i];
        return s;
    }

    int createSortedArray(vector<int>& instructions) {
        const int MOD = 1e9 + 7;
        n = *max_element(instructions.begin(), instructions.end());
        tree.assign(n + 1, 0);

        long long cost = 0;
        for (int i = 0; i < (int)instructions.size(); i++) {
            int val = instructions[i];
            int less = query(val - 1);         // strictly smaller, already inserted
            int leq  = query(val);              // less-or-equal, already inserted
            int greater = i - leq;               // (total inserted so far = i) minus (less-or-equal)
            cost += min(less, greater);
            update(val);
        }
        return (int)(cost % MOD);
    }
};
```

Values fit directly as BIT indices here (`instructions[i] <= nums.length <= 10^5`), so no compression is strictly needed — but for arbitrary-range values, compress first exactly as in 9.1.

**TRACE** for `instructions = [1, 5, 6, 2]` (expected cost: 1):

```
i=0, val=1: less=0, leq=0, greater=0-0=0     cost += min(0,0)=0     → total 0
i=1, val=5: less=1, leq=1, greater=1-1=0     cost += min(1,0)=0     → total 0
i=2, val=6: less=2, leq=2, greater=2-2=0     cost += min(2,0)=0     → total 0
i=3, val=2: less=1, leq=1, greater=3-1=2     cost += min(1,2)=1     → total 1

Final cost = 1  ✓
```

### 9.4 — LC 2179: Count Good Triplets in an Array (Two BITs)

`nums1` and `nums2` are both permutations of `0..n-1`. Count triples `i<j<k` (positions in `nums1`) whose relative order is preserved in `nums2`. Map each value to its position in `nums2`, then the problem reduces to: **count strictly increasing triples** in the mapped array `conv[]`. Two BITs handle the two halves of each triple independently.

```cpp
class Solution {
public:
    long long countGoodTriplets(vector<int>& nums1, vector<int>& nums2) {
        int n = nums1.size();
        vector<int> pos(n);
        for (int i = 0; i < n; i++) pos[nums2[i]] = i;

        vector<int> conv(n);
        for (int i = 0; i < n; i++) conv[i] = pos[nums1[i]] + 1;   // 1-indexed rank in [1..n]

        auto add = [](vector<long long>& t, int i, int n) {
            for (; i <= n; i += i & (-i)) t[i]++;
        };
        auto sum = [](vector<long long>& t, int i) {
            long long s = 0;
            for (; i > 0; i -= i & (-i)) s += t[i];
            return s;
        };

        vector<long long> left(n), right(n);

        // Pass 1 (left to right): left[j] = count of i<j with conv[i] < conv[j]
        vector<long long> bit1(n + 1, 0);
        for (int j = 0; j < n; j++) {
            left[j] = sum(bit1, conv[j] - 1);
            add(bit1, conv[j], n);
        }

        // Pass 2 (right to left): right[j] = count of k>j with conv[k] > conv[j]
        vector<long long> bit2(n + 1, 0);
        for (int j = n - 1; j >= 0; j--) {
            long long insertedSoFar = n - 1 - j;              // elements k>j already processed
            long long notGreater = sum(bit2, conv[j]);         // among them, count with conv[k] <= conv[j]
            right[j] = insertedSoFar - notGreater;
            add(bit2, conv[j], n);
        }

        long long ans = 0;
        for (int j = 0; j < n; j++) ans += left[j] * right[j];
        return ans;
    }
};
```

**TRACE** for `nums1 = [2,0,1,3]`, `nums2 = [0,1,2,3]` (expected answer: 1):

```
pos: pos[0]=0, pos[1]=1, pos[2]=2, pos[3]=3    (nums2 is the identity here)
conv = [pos[2]+1, pos[0]+1, pos[1]+1, pos[3]+1] = [3, 1, 2, 4]

Pass 1 (left[]):
  j=0, conv=3: left[0]=sum(bit1,2)=0             insert rank 3
  j=1, conv=1: left[1]=sum(bit1,0)=0             insert rank 1
  j=2, conv=2: left[2]=sum(bit1,1)=1             insert rank 2
  j=3, conv=4: left[3]=sum(bit1,3)=3             insert rank 4
  left = [0, 0, 1, 3]

Pass 2 (right[]):
  j=3, conv=4: insertedSoFar=0, notGreater=0     right[3]=0    insert rank4
  j=2, conv=2: insertedSoFar=1, notGreater=0     right[2]=1    insert rank2
  j=1, conv=1: insertedSoFar=2, notGreater=0     right[1]=2    insert rank1
  j=0, conv=3: insertedSoFar=3, notGreater=2     right[0]=1    insert rank3
  right = [1, 2, 1, 0]

ans = 0*1 + 0*2 + 1*1 + 3*0 = 1  ✓
```

### 9.5 — LC 308: Range Sum Query 2D — Mutable

Directly instantiate the Section 7 template as a `NumMatrix` class.

```cpp
class NumMatrix {
public:
    int m, n;
    vector<vector<long long>> tree, mat;

    NumMatrix(vector<vector<int>>& matrix) {
        m = matrix.size(); n = matrix[0].size();
        tree.assign(m + 1, vector<long long>(n + 1, 0));
        mat.assign(m, vector<long long>(n, 0));
        for (int i = 0; i < m; i++)
            for (int j = 0; j < n; j++)
                update(i, j, matrix[i][j]);
    }

    void updatePoint(int r, int c, long long delta) {           // r, c are 0-indexed
        for (int i = r + 1; i <= m; i += i & (-i))
            for (int j = c + 1; j <= n; j += j & (-j))
                tree[i][j] += delta;
    }

    void update(int row, int col, int val) {
        long long delta = val - mat[row][col];
        mat[row][col] = val;
        updatePoint(row, col, delta);
    }

    long long prefix(int r, int c) {           // sum of [0,0]..[r-1,c-1], 0-indexed exclusive
        long long s = 0;
        for (int i = r; i > 0; i -= i & (-i))
            for (int j = c; j > 0; j -= j & (-j))
                s += tree[i][j];
        return s;
    }

    int sumRegion(int row1, int col1, int row2, int col2) {
        return (int)(prefix(row2 + 1, col2 + 1) - prefix(row1, col2 + 1)
                    - prefix(row2 + 1, col1) + prefix(row1, col1));
    }
};
```

**Mini-check:** `matrix=[[1]]`, `sumRegion(0,0,0,0)` → constructor calls `update(0,0,1)` → `tree[1][1]=1`. `sumRegion` = `prefix(1,1) - prefix(0,1) - prefix(1,0) + prefix(0,0) = 1-0-0+0 = 1` ✓.

**Why `update()` computes a delta first:** a BIT stores partial aggregates, not raw values — writing `mat[val]` directly into the tree would corrupt every downstream partial sum. You must always update by the DIFFERENCE between new and old value (see Common Mistake 5).

### 9.6 — Counting Inversions (General Utility)

Literally 9.1 with the per-index answers summed into one number — the textbook use case that motivates every "count smaller/larger" BIT pattern above.

```cpp
long long countInversions(vector<int>& arr) {
    vector<int> sorted(arr);
    sort(sorted.begin(), sorted.end());
    sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
    int n = sorted.size();
    vector<int> tree(n + 1, 0);

    auto update = [&](int i) {
        for (; i <= n; i += i & (-i)) tree[i]++;
    };
    auto query = [&](int i) {
        int s = 0;
        for (; i > 0; i -= i & (-i)) s += tree[i];
        return s;
    };

    long long inversions = 0;
    for (int i = (int)arr.size() - 1; i >= 0; i--) {
        int rank = lower_bound(sorted.begin(), sorted.end(), arr[i]) - sorted.begin() + 1;
        inversions += query(rank - 1);       // elements to the right that are smaller than arr[i]
        update(rank);
    }
    return inversions;
}
```

For `arr = [5, 2, 6, 1]`, this returns `2+1+1+0 = 4` — the same per-index values traced in 9.1, summed.

---

## SECTION 10 — COMPLEXITY TABLE

| Operation | Time | Space | Notes |
|---|---|---|---|
| Point update (basic BIT) | O(log n) | O(n) | `i += i&(-i)` |
| Prefix query (basic BIT) | O(log n) | — | `i -= i&(-i)` |
| Range sum `[l,r]` | O(log n) | — | `query(r) - query(l-1)` |
| Build from array (n calls) | O(n log n) | O(n) | naive |
| Build from array (direct) | O(n) | O(n) | push partial sums to parent once |
| Range update, point query | O(log n) each | O(n) | difference BIT |
| Range update, range query | O(log n) each | O(n) | two BITs |
| 2D point update / query | O(log r · log c) | O(r·c) | nested loops |
| 2D range sum | O(log r · log c) | — | inclusion-exclusion, 4 queries |
| Coordinate compression | O(n log n) | O(n) | sort + unique, one-time cost |
| findKth (order statistics) | O(log n) | — | BIT binary lifting |
| Counting inversions | O(n log n) | O(n) | n queries + n updates on compressed ranks |

---

## SECTION 11 — VARIANTS

1. **BIT for XOR instead of sum:** replace `+=` with `^=` everywhere. XOR is its own inverse, so `query(i)` gives the running XOR of `a[1..i]`, and range-XOR is `query(r) ^ query(l-1)`.

2. **BIT for prefix max (restricted):** works ONLY when values are non-decreasing over time (you never need to "undo" a max). `update` takes `tree[i] = max(tree[i], val)`; there's no general inverse, so range-min/max queries with arbitrary updates require a segment tree instead (Pattern 27).

3. **BIT on Euler Tour (subtree sums on trees):** flatten a tree via DFS in/out times, then a subtree-sum-update / path-to-root-query becomes a 1D BIT range-update + point-query — the same difference-BIT trick from Section 5, applied to tree structure (connects to Pattern 22).

4. **Persistent / offline BIT:** for problems with many queries about "state at time t," process offline sorted by time and roll a single BIT forward, snapshotting answers as you go — avoids persistent data structures entirely for a large class of problems.

5. **BIT binary lifting for k-th order statistic:** Section 8's `findKth` — turns a BIT of counts into a mergeable order-statistics structure, competitive with a balanced BST for "insert value, find k-th smallest" workloads.

6. **Merge-sort-based inversion counting:** an O(n log n) alternative to the BIT approach that needs no coordinate compression — useful context, but the BIT approach generalizes better to the LC 315/493/1649/2179 family above.

---

## SECTION 12 — COMMON MISTAKES

### Mistake 1: 0-indexed instead of 1-indexed

```cpp
// WRONG — starting the loop at index 0
for (int i = idx; i < n; i += i & (-i))    // idx=0 → 0 & (-0) = 0 → INFINITE LOOP
    tree[i] += delta;

// CORRECT — always shift to 1-indexed before touching the BIT
update(idx + 1, delta);
// and remember: query(rank - 1) requires rank to already be 1-indexed
```
`0 & (-0) == 0`, so a loop starting at 0 never advances — either an infinite loop or (if bounded by `i <= n`) a silent, permanently-stuck update.

### Mistake 2: Forgetting coordinate compression

```cpp
// WRONG — using raw values (up to 1e9, possibly negative) directly as BIT indices
vector<int> tree(*max_element(nums.begin(), nums.end()) + 1);   // MLE, or crash on negatives

// CORRECT — compress to dense 1-indexed ranks first
vector<int> sorted(nums);
sort(sorted.begin(), sorted.end());
sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
int rank = lower_bound(sorted.begin(), sorted.end(), val) - sorted.begin() + 1;
```

### Mistake 3: Integer overflow

```cpp
// WRONG — int accumulator with up to 1e5 elements each up to 1e9
vector<int> tree(n + 1, 0);          // sums can reach ~1e14 — overflows int

// CORRECT — use long long for tree storage and returned sums
vector<long long> tree(n + 1, 0);
```

### Mistake 4: Off-by-one in `prefix(l-1)`

```cpp
// WRONG — using query(l) instead of query(l-1), silently dropping element l
long long rangeSum = query(r) - query(l);       // excludes a[l] from the range!

// CORRECT
long long rangeSum = query(r) - query(l - 1);   // l-1 correctly INCLUDES a[l]
// query(0) returns 0 automatically — no special-casing needed for l=1
```

### Mistake 5: Updating with `=` instead of a delta

```cpp
// WRONG — treating update(i, newVal) as "set position i to newVal"
void update(int i, long long newVal) {
    for (; i <= n; i += i & (-i)) tree[i] = newVal;   // corrupts every partial sum touching i
}

// CORRECT — a BIT stores AGGREGATES, not raw values. Compute the delta first.
long long delta = newVal - currentValue[idx];         // track raw values separately
currentValue[idx] = newVal;
update(idx + 1, delta);
```
This is exactly why `NumMatrix::update()` in Section 9.5 keeps a separate `mat[]` array — the BIT itself can never be queried for "the raw value at position i" without another O(log n) point-query trick.

### Mistake 6: BIT sized off by one

```cpp
// WRONG — sizing the tree to exactly n when 1-indexed positions go up to n
vector<long long> tree(n);           // valid indices 0..n-1, but update writes tree[n] → OOB

// CORRECT — always allocate n+1 slots for a 1-indexed BIT over n elements
vector<long long> tree(n + 1, 0);
```

---

## SECTION 13 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Range Sum Query - Mutable | 307 | Basic BIT: point update + prefix query |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 2 | Count of Smaller Numbers After Self | 315 | Compress values, sweep right-to-left |
| 3 | Reverse Pairs | 493 | Compress + binary search for threshold rank |
| 4 | Create Sorted Array through Instructions | 1649 | less/greater via prefix query, left-to-right sweep |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 5 | Count Good Triplets in an Array | 2179 | Two BITs, position remapping |
| 6 | Range Sum Query 2D - Mutable | 308 | 2D BIT, inclusion-exclusion |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Global and Local Inversions | 775 | Inversion-counting insight without brute force |
| 8 | Longest Increasing Subsequence | 300 | BIT-based O(n log n) LIS via prefix-max |

---

## SECTION 14 — PATTERN CONNECTIONS

1. **Pattern 27 (Segment Tree):** A BIT is a specialized, lower-constant-factor segment tree for **invertible** point-update/range-query operations (sum, XOR). Reach for a segment tree instead the moment you need range-min/max, range-assignment, or lazy propagation — operations without a clean inverse.

2. **Pattern 21 (LIS):** The O(n log n) patience-sorting LIS (binary search on tails) has a direct BIT-based twin: compress values, then LIS length ending at index i = `1 + BIT.prefixMax(rank[i] - 1)`, updating a prefix-max BIT as you sweep left to right (Problem 8 above).

3. **Pattern 06 (Prefix Sums):** A BIT is literally "prefix sums that support O(log n) updates." Static prefix sums (Pattern 06) give O(1) query but O(n) rebuild after any change — a BIT trades a small O(log n) query cost for O(log n) updates, which is the right trade the moment the array is mutable.

4. **Pattern 22 (DP on Trees / Euler Tour):** Flattening a tree via DFS in/out timestamps turns "add to a subtree, query a path to root" into a 1D range-update problem — solved with exactly the difference-BIT technique from Section 5.

5. **Pattern 04 (Monotonic Stack) — contrast, not overlap:** counting inversions can also be done with a merge-sort-style divide and conquer, or in specific structured cases with a monotonic stack; the BIT approach is the most general and reusable across the whole "count smaller/larger to one side" problem family (LC 315, 493, 1649, 2179 all share the same skeleton).

---

## SECTION 15 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 307 — Range Sum Query Mutable

**Interviewer:** "Design a data structure that supports updating an array element and querying the sum of a range, both efficiently."

**Candidate:**

*[First minute — rule out the naive options]*
> "A plain array gives O(1) update but O(n) range sum. A prefix-sum array flips that: O(1) range sum but O(n) update, since one change shifts every downstream prefix. I need something that's O(log n) for BOTH — that's exactly what a Fenwick Tree gives, since the operation (sum) is invertible."

*[State the core trick before coding]*
> "The key structure: `tree[i]` stores the sum of `lowbit(i)` elements ending at i, where `lowbit(i) = i & (-i)`. Update walks UP by adding the lowbit; query walks DOWN by subtracting it. Both terminate in O(log n) steps because each step flips one bit of the index."

*[Code — 5 minutes]*
```cpp
class NumArray {
    vector<int> nums;
    vector<long long> tree;
    int n;
    void update(int i, long long delta) {
        for (; i <= n; i += i & (-i)) tree[i] += delta;
    }
    long long query(int i) {
        long long s = 0;
        for (; i > 0; i -= i & (-i)) s += tree[i];
        return s;
    }
public:
    NumArray(vector<int>& a) : nums(a), n(a.size()), tree(a.size() + 1, 0) {
        for (int i = 0; i < n; i++) update(i + 1, a[i]);
    }
    void update(int index, int val) {
        long long delta = val - nums[index];
        nums[index] = val;
        update(index + 1, delta);
    }
    int sumRange(int left, int right) {
        return query(right + 1) - query(left);
    }
};
```

*[Interviewer]: "Why do you keep a separate `nums` array instead of reading values back out of `tree`?"*
> "The BIT stores partial aggregates, not raw values — there's no O(1) way to recover `a[i]` from `tree[]` alone. Keeping `nums[]` lets me compute the delta (`newVal - oldVal`) needed for a correct point update, since the BIT only understands additive deltas."

**RED FLAGS:**
- Forgetting to shift indices by +1 (0-indexed input, 1-indexed BIT)
- Calling `update(index, val)` as if it sets the value directly instead of computing a delta
- Off-by-one in `sumRange`: using `query(right)` instead of `query(right+1)`, or `query(left-1)` instead of `query(left)` after the +1 shift

---

### Interview Simulation 2: LC 315 — Count of Smaller Numbers After Self

**Interviewer:** "For each element, count how many elements to its right are strictly smaller."

**Candidate:**

*[Identify the brute force and its cost]*
> "Brute force is O(n²) — for each i, scan everything to the right. To beat that, I want to process elements in an order where 'everything currently in my data structure' is exactly 'everything to the right of me' — that means sweeping right to left, inserting as I go, and querying BEFORE I insert the current element."

*[Explain why compression is required first]*
> "Values can be up to 1e4 in magnitude and can repeat, but what matters is RELATIVE order, not absolute value. I compress to dense ranks first — sort, dedupe, binary search each value's position. Then a count-BIT over ranks answers 'how many inserted values have rank < mine' in O(log n)."

*[Code — 6 minutes, same as Section 9.1]*

*[Interviewer]: "Walk through why you query BEFORE updating, not after."*
> "If I update first, the current element counts itself in its own query — I'd never want that since I'm asking about elements strictly to the right, and by construction the current element hasn't been 'placed' yet when the query runs. Query captures the state of 'everything inserted so far,' i.e., everything with a larger original index, which is exactly the right side."

**RED FLAGS:**
- Sweeping left to right (would count elements to the LEFT, not the right)
- Using `query(rank)` instead of `query(rank - 1)` — this would count itself and any earlier tied duplicate as "smaller," inflating the answer
- Not deduplicating during compression, causing rank collisions to silently double-count

---

### Interview Simulation 3: Meta-discussion — BIT vs Segment Tree

**Interviewer:** "When would you choose a Fenwick Tree over a segment tree, and vice versa?"

**Candidate:**
> "I check two things: is the operation invertible, and do I need anything beyond point-update/range-query?

> Sum and XOR are invertible — subtraction undoes addition, XOR undoes itself. For these, a BIT is strictly better: 5-10 lines, no recursion, roughly half the constant factor of a segment tree, and it trivially extends to 2D by nesting the loop.

> Min and max are NOT invertible — you can't 'un-min' a removed value without re-scanning the remaining elements. For range-min/max, or anything needing range assignment with lazy propagation, I go straight to a segment tree (Pattern 27); trying to force a BIT there either doesn't work or requires enough extra bookkeeping that the segment tree ends up simpler.

> There's a middle ground too: BITs can support range-update/range-query via the two-BIT trick (Section 6), matching a lazy segment tree's capability for sum queries specifically — but the moment the query type generalizes past 'sum,' the segment tree wins on flexibility even if the BIT would still win on constant factor for the sum case alone."

**RED FLAGS:**
- Claiming BITs can do range-min/max with simple modifications (they structurally cannot, without extra machinery beyond a plain BIT)
- Not knowing the two-BIT range-update/range-query trick exists (defaults to "BIT can't do range updates" — false)
- Choosing a segment tree by default without articulating why the BIT's invertibility requirement rules it out or in

---

## SECTION 16 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: What does `x & (-x)` compute, and why does it isolate the lowest set bit?**

> In two's complement, `-x` equals `~x + 1`. Flipping every bit of `x` and adding 1 causes every bit below `x`'s lowest set bit to become 1 in `-x` (since they were all 0 in `x`, flipped to 1, and the `+1` doesn't propagate past them), while the lowest set bit itself and everything above it get inverted differently. ANDing `x` with `-x` cancels every bit except that lowest set bit, leaving its VALUE (not position) — e.g., `12 = 0b1100` gives `12 & (-12) = 4 = 0b0100`.

---

**Q2: Why must Fenwick trees be 1-indexed — what specifically breaks at index 0?**

> `0 & (-0) = 0`. If `update` or `query` ever operates on index 0, the loop condition `i += i & (-i)` (or `i -= i & (-i)`) never changes `i` — it's either an infinite loop (unbounded) or a permanently stuck no-op (bounded by `i <= n` but `i` never advances past 0). Every real position must be shifted to start at 1, and `query(0)` is used deliberately as the sentinel "empty prefix, sum = 0" — its loop body never executes since `i > 0` is false immediately.

---

**Q3: `update()` moves `i += lowbit(i)`; `query()` moves `i -= lowbit(i)`. Why opposite directions?**

> `update` needs to touch every block that CONTAINS the modified position — those blocks get larger and more inclusive as you move up, which happens by adding the lowbit (extending to the next larger containing range). `query` needs to decompose `[1..i]` into disjoint blocks that exactly tile it without overlap — subtracting the lowbit moves to the block immediately preceding the current one, walking DOWN through progressively smaller/earlier ranges until reaching 0. One walks toward larger containing intervals (ancestors); the other walks toward a disjoint tiling (siblings-to-the-left).

---

**Q4: Derive how to support range-update + point-query using a "difference BIT."**

> Define `d[i] = a[i] - a[i-1]`. Then `a[i] = d[1] + d[2] + ... + d[i]` — a prefix sum of `d`. Adding `v` to every element in `[l, r]` only changes TWO entries of `d`: `d[l] += v` (the bump starts here and propagates to every later prefix sum) and `d[r+1] -= v` (cancels the bump immediately after r). A BIT over `d` supporting prefix-sum queries then answers `pointQuery(i) = prefix_d(i) = a[i]` directly. Two O(log n) updates replace an O(r-l+1) loop.

---

**Q5: Derive the two-BIT formula for range-update + range-query: why `(i+1)·B1.prefix(i) − B2.prefix(i)`?**

> Starting from `S(i) = Σ_{k=1}^{i} a[k]` and substituting `a[k] = Σ_{j=1}^{k} d[j]`, swap the order of summation: each `d[j]` contributes to every `a[k]` with `k ≥ j`, i.e., it appears `(i - j + 1)` times in the double sum for `k` up to `i`. So `S(i) = Σ_{j=1}^{i} d[j]·(i-j+1) = (i+1)·Σd[j] − Σ(d[j]·j)`. The first sum is a plain BIT over `d[j]` (`B1`); the second is a BIT over `d[j]·j` (`B2`). A range-update `[l,r] += v` still only touches two positions in each BIT: `B1` gets `+v` at l and `−v` at r+1 (as before); `B2` gets `+v·l` at l and `−v·(r+1)` at r+1, matching the weight `d[j]·j` at each of those two positions.

---

**Q6: What's the time/space complexity of a 2D BIT for an n×m grid, and why?**

> O(log n · log m) per update or query — the outer loop over rows contributes a factor of `log n` steps (each moving by the row's lowbit), and for EACH of those steps, the inner loop over columns contributes another `log m` steps. Space is O(n·m), same as storing the original grid once, since `tree` is allocated as a full `(n+1) × (m+1)` array. This generalizes cleanly to k dimensions as O(log^k) time, O(product of dimensions) space, though k > 2 is rare in practice.

---

**Q7: Why is coordinate compression necessary before using a BIT for order-statistics problems like Count of Smaller Numbers After Self?**

> A BIT's array size must match the RANGE of indices you index into, not the number of elements. If values range up to 1e9 (or are negative), allocating a BIT sized to that range is either impossible (negative indices) or wasteful (MLE for a mostly-empty array). Since these problems only care about RELATIVE order — "how many elements are smaller" — compression maps every distinct value to a dense rank in `[1, m]` where `m` = number of distinct values ≤ n. This preserves all order relationships needed for the queries while making the BIT's size proportional to the input size, not the value range.

---

## SECTION 17 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. BASIC BIT — point update, prefix query
// ─────────────────────────────────────────────────────────────────
vector<long long> tree(n + 1, 0);
void update(int i, long long delta) { for (; i <= n; i += i & (-i)) tree[i] += delta; }
long long query(int i) { long long s = 0; for (; i > 0; i -= i & (-i)) s += tree[i]; return s; }

// ─────────────────────────────────────────────────────────────────
// 2. RANGE SUM
// ─────────────────────────────────────────────────────────────────
long long rangeSum(int l, int r) { return query(r) - query(l - 1); }   // never forget l-1

// ─────────────────────────────────────────────────────────────────
// 3. DIFFERENCE BIT — range update, point query
// ─────────────────────────────────────────────────────────────────
void rangeUpdate(int l, int r, long long v) { update(l, v); update(r + 1, -v); }
long long pointQuery(int i) { return query(i); }        // over the diff BIT

// ─────────────────────────────────────────────────────────────────
// 4. TWO-BIT — range update, range query
// ─────────────────────────────────────────────────────────────────
void rangeUpdate(int l, int r, long long v) {
    add(B1, l, v);        add(B1, r + 1, -v);
    add(B2, l, v * l);    add(B2, r + 1, -v * (r + 1));
}
long long prefixSum(int i) { return (long long)(i + 1) * sum(B1, i) - sum(B2, i); }

// ─────────────────────────────────────────────────────────────────
// 5. 2D BIT
// ─────────────────────────────────────────────────────────────────
void update(int r, int c, long long d) {
    for (int i = r; i <= rows; i += i & (-i))
        for (int j = c; j <= cols; j += j & (-j)) tree[i][j] += d;
}
long long query(int r, int c) {
    long long s = 0;
    for (int i = r; i > 0; i -= i & (-i))
        for (int j = c; j > 0; j -= j & (-j)) s += tree[i][j];
    return s;
}

// ─────────────────────────────────────────────────────────────────
// 6. COORDINATE COMPRESSION
// ─────────────────────────────────────────────────────────────────
vector<int> sorted(nums);
sort(sorted.begin(), sorted.end());
sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
int rank = lower_bound(sorted.begin(), sorted.end(), val) - sorted.begin() + 1;   // 1-indexed

// ─────────────────────────────────────────────────────────────────
// 7. COUNT SMALLER TO THE RIGHT (sweep right to left, query before insert)
// ─────────────────────────────────────────────────────────────────
for (int i = n - 1; i >= 0; i--) {
    ans[i] = query(rank[i] - 1);      // count of smaller values inserted so far
    update(rank[i]);
}

// ─────────────────────────────────────────────────────────────────
// 8. DECISION TABLE
// ─────────────────────────────────────────────────────────────────
// Need point update + prefix/range SUM (or XOR)         → basic BIT
// Need range update + point query                       → difference BIT
// Need range update + range query                       → two BITs
// Need 2D point/range sum                                → 2D BIT
// Need "count smaller/larger" or k-th order statistic    → compress + BIT
// Need range MIN/MAX, or lazy range-assignment           → segment tree instead (Pattern 27)
```

---

## SECTION 18 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 8 problems in order.** LC 307 first (mechanical warmup), then the compression-heavy trio (315, 493, 1649), then the harder combinations (2179, 308), then the two contest problems under a strict 20-minute clock.

2. **Re-derive the two-BIT formula from scratch** (Section 6) without looking at the notes. If you can reconstruct `S(i) = (i+1)·B1.prefix(i) − B2.prefix(i)` from the difference-array definition alone, you understand the technique rather than having memorized it.

3. **Implement `findKth` (Section 8) and test it against a `multiset`.** For random insert/find-kth sequences, confirm the BIT binary-lifting answer matches `std::next(multiset.begin(), k-1)`. This cements the "BIT as order-statistics tree" mental model.

4. **Cross-train against Pattern 27 (Segment Tree):** implement LC 307 with BOTH a BIT and a segment tree, and time them. The constant-factor gap should be obvious — this is the practical argument for reaching for a BIT first whenever the operation is invertible.

5. **Preview Pattern 21 (LIS):** re-solve LC 300 with the BIT prefix-max technique (Problem 8 above) after having solved it with binary search on tails. Compare the two O(n log n) approaches — same complexity, different underlying structure.

---

## SECTION 19 — SIGN-OFF CRITERIA

### Tier 1 — Basic BIT mastered
You implement `update`/`query` from memory in under 2 minutes, correctly 1-indexed, and can explain why `x & (-x)` isolates the lowest set bit. You solve LC 307 cleanly.

### Tier 2 — Order statistics fluent
You solve LC 315 and 493 without hesitating on the coordinate compression step, and you can explain — without looking — why the sweep direction (right-to-left) and query-before-insert ordering are both load-bearing.

### Tier 3 — Difference tricks solid
You derive the difference-BIT (range update + point query) and the two-BIT formula (range update + range query) on a blank page, including the `(i+1)·B1.prefix(i) − B2.prefix(i)` algebra, not just the code.

### Tier 4 — 2D and combinations complete
You solve LC 308 (2D BIT) and LC 2179 (two BITs, position remapping) cleanly. You can articulate precisely when a BIT is the wrong tool (range min/max, lazy assignment) and reach for a segment tree instead without hesitation.

**Sign-off threshold:** Solve 6 of 8 problems. Mandatory: LC 307 (the foundational template), LC 315 (coordinate compression + order statistics — the single most reused sub-skill in this pattern), and LC 308 (2D BIT — proof you can generalize the 1D trick).

---

*Pattern 28 Complete — Fenwick Tree (Binary Indexed Tree)*
*Next: Pattern 29 — Union-Find Advanced (Weighted DSU, Small-to-Large Merging, Offline Connectivity)*
