# PATTERN 24 — BITMASK DP
## Subset States, Held-Karp TSP, Assignment DP, Game-Theory Masks — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of bitmask DP:**

Bitmask DP problems are recognizable by a single signal that should fire instantly:

> **N ≤ 20 (sometimes ≤ 14, occasionally ≤ 24 with heavy pruning) → think bitmask.**

The moment you see a small `n` combined with words like "all subsets," "every combination," "assign each," "visit all," or "partition into groups," your first hypothesis should be:

```
dp[mask] = answer restricted to the subset of elements represented by mask
```

where `mask` is an integer whose `i`-th bit tells you whether element `i` is "used," "visited," "assigned," or "included."

That's Pattern 24 at Tier 1. The real test is in the **state design**:

- **Subsets (LC 78):** `mask` IS the answer — trivial enumeration, no DP needed. This is the "hello world."
- **Partition to K Equal Sum Subsets (LC 698):** `mask` = elements already placed into *some* completed or in-progress group. The DP hides an implicit "current group" via `sum(mask) % target`.
- **Shortest Path Visiting All Nodes (LC 847):** The state is NOT just `mask` — it's `(node, mask)`. Two graphs can share a mask but be in totally different positions. This is the leap from "bitmask enumeration" to "bitmask as one axis of a bigger state."
- **Held-Karp TSP:** `dp[mask][i]` = minimum cost to have visited exactly the cities in `mask`, currently standing at city `i`. This is THE canonical bitmask DP and every harder problem in this pattern is a variation of it.
- **Can I Win (LC 464):** The mask isn't spatial at all — it is a *history* of which numbers have been consumed in a game. The DP is over game states, and parity of `popcount(mask)` tells you whose turn it is.

**The key skill:**

Before coding, answer:
1. What does bit `i` of the mask represent — "element i used," "person i satisfied," "city i visited"? Say it out loud.
2. Does the transition need ONLY the mask, or does it need `(mask, extra_dimension)` like a current position, current player, or current group index?
3. Is `n` small enough for `O(2^n · n)` or do I need the sharper `O(2^n · n^2)` (Held-Karp) or `O(3^n)` (submask enumeration)?
4. Can I derive "whose turn," "which group," or "how far along" purely from `popcount(mask)` instead of tracking it as a separate dimension? (This is the #1 space-saving trick in this pattern.)

If you answer these correctly, the code writes itself — and it is usually short. The entire difficulty of bitmask DP lives in the state design, not the loops.

---

## SECTION 2 — BIT-TRICKS PRIMER

Every template in this document leans on a small, fixed vocabulary of bit operations. Memorize these cold — fumbling them under interview pressure is the single most common way to lose points on an otherwise-correct bitmask DP.

```cpp
// ─────────────────────────────────────────────────────────────────
// CORE BIT OPERATIONS
// ─────────────────────────────────────────────────────────────────

// Set bit i (mark element i as used/included)
mask |= (1 << i);

// Clear bit i (mark element i as unused/excluded)
mask &= ~(1 << i);

// Test bit i (is element i included?)
bool included = mask & (1 << i);

// Toggle bit i
mask ^= (1 << i);

// Count set bits — how many elements are in this mask
int count = __builtin_popcount(mask);        // for int / unsigned
int count64 = __builtin_popcountll(mask);     // for long long / unsigned long long

// Lowest set bit VALUE (isolates the rightmost 1-bit)
int lowBit = mask & (-mask);
// WHY: -mask is the two's complement negation = (~mask + 1).
// Every bit above the lowest set bit gets flipped and then flipped back
// by the +1 carry chain; only the lowest set bit survives the AND.
// Example: mask = 0b10110100 (=180)
//          -mask = 0b01001100 (two's complement)
//          mask & -mask = 0b00000100 = 4  ← the lowest set bit's value

// Index of lowest set bit (0-indexed)
int lowIdx = __builtin_ctz(mask);   // "count trailing zeros"

// Clear the lowest set bit — classic trick for iterating set bits one at a time
mask & (mask - 1);
// WHY: mask-1 flips the lowest set bit to 0 and all bits below it to 1.
// ANDing with the original mask keeps everything above unchanged and
// zeroes out the lowest set bit (the borrow-flipped bits cancel via AND).

// Iterate over all SET bits of a mask
for (int m = mask; m > 0; m &= (m - 1)) {
    int i = __builtin_ctz(m);   // current set bit's index
    // process element i
}

// Check if mask is a submask of full (mask ⊆ full)
bool isSubmask = (mask & full) == mask;
```

### Iterating all submasks of a mask — the O(3^n) technique

A recurring need: given a `mask`, enumerate every `sub` such that `sub` is a subset of `mask` (bitwise `sub & mask == sub`), including `sub == mask` and `sub == 0`.

```cpp
// Enumerate every submask of `mask`, from `mask` itself down to 0
for (int sub = mask; ; sub = (sub - 1) & mask) {
    // process sub  (sub is guaranteed to be a subset of mask's bits)
    if (sub == 0) break;   // must check AFTER processing sub==0, then break
}

// Equivalent, more common idiom (skips sub==0 explicitly, add extra `if` if needed):
for (int sub = mask; sub; sub = (sub - 1) & mask) {
    // process sub (never includes sub == 0 in this form)
}
```

**WHY `sub = (sub - 1) & mask` visits exactly the submasks, in decreasing order:**

`sub - 1` flips the lowest set bit of `sub` to 0 and sets every bit below it to 1 (standard binary decrement borrow behavior). ANDing with `mask` then does two things simultaneously:
1. Strips out any of those newly-set "borrow bits" that are not part of `mask` (so we never leave `mask`'s universe).
2. Correctly restores bits of `mask` in positions below the flipped bit, giving exactly the next-smaller submask.

**PROOF SKETCH that total submask enumeration over ALL masks costs O(3^n), not O(4^n):**

Consider a naive nested loop `for mask in [0, 2^n) { for sub submask of mask { ... } }`. It looks like `O(2^n · 2^n) = O(4^n)` work, but it is not — because most masks have far fewer than `2^n` submasks.

For a fixed universe of `n` elements, look at any single element `i`. Across all pairs `(mask, sub)` where `sub` is a submask of `mask`, element `i` can be in exactly one of **three** states:
1. Not in `mask` at all (hence not in `sub` either) — 1 way.
2. In `mask` but not in `sub` — 1 way.
3. In `mask` AND in `sub` — 1 way.

There is no fourth state ("in `sub` but not in `mask`" is impossible since `sub ⊆ mask`). Since each of the `n` elements independently has 3 choices, the total number of valid `(mask, sub)` pairs across the ENTIRE double loop is exactly `3^n`.

```
Σ (over all mask) [number of submasks of mask] = Σ (over all mask) 2^popcount(mask) = 3^n
```

(The middle equality is itself a binomial-theorem fact: `Σ_{k=0}^{n} C(n,k) · 2^k = (1+2)^n = 3^n`, since there are `C(n,k)` masks with exactly `k` bits set, each contributing `2^k` submasks.)

**Practical consequence:** any DP of the form `dp[mask] = f(dp[sub], dp[mask \ sub])` for all submasks `sub` of `mask` (SOS DP, subset-sum partitioning, "Maximum Students"-style row-pair enumeration, Parallel Courses II) runs in `O(3^n)`, which is tractable up to roughly `n ≈ 18–20`, versus `O(4^n)` which is not.

---

## SECTION 3 — TEMPLATE 1: SUBSETS VIA BITMASK (WARM-UP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 78 — Subsets
// Given a set of n distinct integers, return ALL possible subsets.
//
// KEY INSIGHT: A subset of n elements corresponds 1-to-1 with an
// n-bit number: bit i set ⟺ element i is included.
// There are exactly 2^n subsets (including empty set and full set),
// so iterating mask from 0 to 2^n - 1 enumerates every subset exactly once.
// ─────────────────────────────────────────────────────────────────

vector<vector<int>> subsets(vector<int>& nums) {
    int n = nums.size();
    vector<vector<int>> result;
    result.reserve(1 << n);

    for (int mask = 0; mask < (1 << n); mask++) {
        vector<int> subset;
        for (int i = 0; i < n; i++) {
            if (mask & (1 << i)) subset.push_back(nums[i]);
        }
        result.push_back(subset);
    }
    return result;
}
```

**FULL TRACE for nums = [1,2,3]:**

```
n=3, 2^3 = 8 masks

mask=000: {}              mask=100: {3}
mask=001: {1}             mask=101: {1,3}
mask=010: {2}             mask=110: {2,3}
mask=011: {1,2}           mask=111: {1,2,3}

8 subsets total ✓
```

**Why this matters as a foundation:** every later template in this pattern is "Subsets, but with a smarter loop order and a `dp[]` array attached to the mask instead of materializing the subset itself." Internalizing "mask ↔ subset" bijection here makes everything downstream mechanical.

---

## SECTION 4 — TEMPLATE 2: PARTITION TO K EQUAL SUM SUBSETS

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 698 — Partition to K Equal Sum Subsets
// Given nums and k, determine if nums can be partitioned into k
// non-empty subsets with equal sums.
//
// State: memo[mask] = can the elements in `mask` be exactly partitioned
//                      into some number of COMPLETE groups of size `target`,
//                      possibly with one partially-filled group remaining?
//
// KEY TRICK: we don't need a separate "which group" dimension. The
// running total of elements already placed, taken mod target, tells us
// exactly how much room is left in the group currently being filled.
//
// Transition: from mask, compute currentSum = sum of elements in mask.
//             remainder = currentSum % target  (space used in current group)
//             try adding each unused element i whose value fits
//             (remainder + nums[i] <= target); recurse into mask | (1<<i).
//
// Pruning: sort descending — placing large elements first fails fast.
// ─────────────────────────────────────────────────────────────────

bool canPartitionKSubsets(vector<int>& nums, int k) {
    int n = nums.size();
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum % k != 0) return false;
    int target = sum / k;

    sort(nums.rbegin(), nums.rend());          // pruning: try big pieces first
    if (nums[0] > target) return false;        // a single element too big — impossible

    vector<int> memo(1 << n, -1);               // -1 = unvisited, 0 = false, 1 = true

    function<bool(int)> dfs = [&](int mask) -> bool {
        if (mask == (1 << n) - 1) return true;   // every element placed
        if (memo[mask] != -1) return memo[mask];

        int currentSum = 0;
        for (int i = 0; i < n; i++)
            if (mask & (1 << i)) currentSum += nums[i];
        int remainder = currentSum % target;      // room used in the in-progress group

        for (int i = 0; i < n; i++) {
            if (mask & (1 << i)) continue;                 // already placed
            if (remainder + nums[i] > target) continue;    // would overflow current group
            if (dfs(mask | (1 << i))) return memo[mask] = 1;
        }
        return memo[mask] = 0;
    };

    return dfs(0);
}
```

**FULL TRACE for nums=[1,2,3,4], k=2 (sum=10, target=5):**

```
Sorted descending: nums = [4,3,2,1]  (indices 0..3 now refer to sorted order)

dfs(0000): currentSum=0, remainder=0
  try i=0 (val 4): 0+4=4<=5 → dfs(1000)

    dfs(1000): currentSum=4, remainder=4
      try i=1 (val 3): 4+3=7>5 SKIP
      try i=2 (val 2): 4+2=6>5 SKIP
      try i=3 (val 1): 4+1=5<=5 → dfs(1001)

        dfs(1001): currentSum=5, remainder=5%5=0   ← group 1 exactly closed!
          try i=1 (val 3): 0+3=3<=5 → dfs(1011)

            dfs(1011): currentSum=8, remainder=8%5=3
              try i=2 (val 2): 3+2=5<=5 → dfs(1111)

                dfs(1111): mask == 1111 (full) → return TRUE

              dfs(1011) = TRUE (via i=2)
          dfs(1001) = TRUE (via i=1)
      dfs(1000) = TRUE (via i=3)
  dfs(0000) = TRUE (via i=0)

Answer: TRUE
Groups found: {4,1}=5 and {3,2}=5 ✓
```

**Why the `remainder = currentSum % target` trick works without an explicit "group index" dimension:** at any point, the elements placed so far tile perfectly into `floor(currentSum / target)` COMPLETE groups plus one partial group holding `currentSum % target`. We never need to know *which* group index we're filling — only how much room remains in whichever group is currently open. This collapses what looks like a 2D state `(mask, groupIndex)` into a 1D state `(mask)`.

---

## SECTION 5 — TEMPLATE 3: SHORTEST PATH VISITING ALL NODES (BFS + BITMASK STATE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 847 — Shortest Path Visiting All Nodes
// Undirected connected graph, n <= 12 nodes. You may start at ANY node
// and revisit nodes/edges freely. Find the shortest walk that visits
// every node at least once.
//
// KEY INSIGHT: This is unweighted-edge shortest path → BFS, but the
// "position" in the search space is NOT just the current node — it's
// (current node, set of nodes visited so far). Two walks that are both
// standing at node 3 but have visited different node-sets are in
// different states and must be tracked separately.
//
// State: (node, mask)   mask's bit i = "have we visited node i yet"
// Multi-source: since we may start anywhere, push ALL (i, {i}, dist=0)
//               into the queue up front — standard "multi-source BFS."
// Goal: first time mask == (1<<n) - 1 is popped, its dist is the answer
//       (BFS guarantees first-reached == shortest).
// ─────────────────────────────────────────────────────────────────

int shortestPathLength(vector<vector<int>>& graph) {
    int n = graph.size();
    if (n == 1) return 0;
    int target = (1 << n) - 1;

    queue<tuple<int,int,int>> q;                   // (node, mask, dist)
    vector<vector<bool>> visited(n, vector<bool>(1 << n, false));

    for (int i = 0; i < n; i++) {
        q.push({i, 1 << i, 0});
        visited[i][1 << i] = true;
    }

    while (!q.empty()) {
        auto [node, mask, dist] = q.front(); q.pop();
        if (mask == target) return dist;

        for (int next : graph[node]) {
            int nextMask = mask | (1 << next);
            if (!visited[next][nextMask]) {
                visited[next][nextMask] = true;
                q.push({next, nextMask, dist + 1});
            }
        }
    }
    return -1;   // unreachable for a connected graph — should not happen
}
```

**FULL TRACE for graph = [[1,2,3],[0],[0],[0]] (star graph, center 0):**

```
n=4, target = 1111

Multi-source seed (dist 0):
  (0, 0001, 0)  (1, 0010, 0)  (2, 0100, 0)  (3, 1000, 0)

dist=1 layer (expand from each seed through its neighbors):
  (0, 0011, 1) from node1→0     (0, 0101, 1) from node2→0
  (0, 1001, 1) from node3→0     (1, 0011, 1) from node0→1
  (2, 0101, 1) from node0→2     (3, 1001, 1) from node0→3

dist=2 layer (expand dist=1 states through node 0's 3 neighbors):
  from (0,0011,1): → (2, 0111, 2)   → (3, 1011, 2)
  from (0,0101,1): → (1, 0111, 2)   → (3, 1101, 2)
  from (0,1001,1): → (1, 1011, 2)   → (2, 1101, 2)
  (states like (1,0011,2) are duplicates of already-visited (1,0011,1) — skipped)

dist=3 layer (expand dist=2 states back through 0):
  from (2,0111,2): → (0, 0111, 3)   [first time node0 reaches mask 0111]
  from (3,1011,2): → (0, 1011, 3)
  from (1,0111,2): → (0, 0111, 3)   duplicate, skipped
  from (3,1101,2): → (0, 1101, 3)
  from (1,1011,2): → (0, 1011, 3)   duplicate, skipped
  from (2,1101,2): → (0, 1101, 3)   duplicate, skipped

dist=4 layer (from (0, 0111, 3) expand to the ONE missing neighbor, node 3):
  (0,0111,3) --node3--> (3, 1111, 4)   ← mask == target!

Answer: 4
Winning walk (one instance): 1 → 0 → 2 → 0 → 3   (4 edges, visits {0,1,2,3})
```

**Why BFS (not DFS) is mandatory here:** the state space `(node, mask)` has `n · 2^n` states and every edge has weight 1. BFS explores states in non-decreasing distance order, so the FIRST time `mask == target` is dequeued, that distance is provably minimal — exactly the same guarantee as plain BFS shortest-path, just over a bigger state graph.

---

## SECTION 6 — TEMPLATE 4: HELD-KARP TSP — dp[mask][i]

This is the canonical bitmask DP. Almost every other template in this pattern is a structural variant of it.

```cpp
// ─────────────────────────────────────────────────────────────────
// Held-Karp algorithm — Traveling Salesman Problem (exact, O(2^n · n^2))
// Complete graph with n cities, dist[i][j] = cost to go from i to j.
// Find the minimum-cost tour starting and ending at city 0, visiting
// every city exactly once.
//
// State: dp[mask][i] = minimum cost to have visited EXACTLY the cities
//                       in `mask` (mask always includes city 0), ending
//                       the partial tour AT city i.
// Transition: dp[mask][v] = min over u in mask\{v} of
//                            dp[mask \ {v}][u] + dist[u][v]
//             (we arrived at v last, having just come from u)
// Base case: dp[{0}][0] = 0  (mask = 0...01, standing at city 0, cost 0)
// Answer: min over i != 0 of dp[full][i] + dist[i][0]   (close the tour)
// ─────────────────────────────────────────────────────────────────

int tsp(vector<vector<int>>& dist) {
    int n = dist.size();
    const int INF = INT_MAX / 2;
    vector<vector<int>> dp(1 << n, vector<int>(n, INF));
    dp[1][0] = 0;   // mask = {0}, standing at city 0

    for (int mask = 1; mask < (1 << n); mask++) {
        if (!(mask & 1)) continue;              // city 0 must always be visited already
        for (int u = 0; u < n; u++) {
            if (!(mask & (1 << u))) continue;   // u must be in mask
            if (dp[mask][u] == INF) continue;
            for (int v = 0; v < n; v++) {
                if (mask & (1 << v)) continue;  // v must NOT be visited yet
                int nmask = mask | (1 << v);
                dp[nmask][v] = min(dp[nmask][v], dp[mask][u] + dist[u][v]);
            }
        }
    }

    int full = (1 << n) - 1;
    int ans = INT_MAX;
    for (int u = 1; u < n; u++)
        if (dp[full][u] != INF)
            ans = min(ans, dp[full][u] + dist[u][0]);
    return ans;
}
```

**FULL TRACE for the 4-city distance matrix:**

```
dist = [[ 0,10,15,20],
        [10, 0,35,25],
        [15,35, 0,30],
        [20,25,30, 0]]

dp[0001][0] = 0

── from mask=0001 (u=0) ──
  v=1: dp[0011][1] = 0+10 = 10
  v=2: dp[0101][2] = 0+15 = 15
  v=3: dp[1001][3] = 0+20 = 20

── from mask=0011 (u=1, dp=10) ──
  v=2: dp[0111][2] = min(INF, 10+35) = 45
  v=3: dp[1011][3] = min(INF, 10+25) = 35

── from mask=0101 (u=2, dp=15) ──
  v=1: dp[0111][1] = min(INF, 15+35) = 50
  v=3: dp[1101][3] = min(INF, 15+30) = 45

── from mask=1001 (u=3, dp=20) ──
  v=1: dp[1011][1] = min(INF, 20+25) = 45
  v=2: dp[1101][2] = min(INF, 20+30) = 50

── from mask=0111 (u=1, dp=50; u=2, dp=45) ──
  u=1,v=3: dp[1111][3] = min(INF, 50+25) = 75
  u=2,v=3: dp[1111][3] = min(75,  45+30) = 75  (tie, no change)

── from mask=1011 (u=1, dp=45; u=3, dp=35) ──
  u=1,v=2: dp[1111][2] = min(INF, 45+35) = 80
  u=3,v=2: dp[1111][2] = min(80,  35+30) = 65

── from mask=1101 (u=2, dp=50; u=3, dp=45) ──
  u=2,v=1: dp[1111][1] = min(INF, 50+35) = 85
  u=3,v=1: dp[1111][1] = min(85,  45+25) = 70

Final row: dp[1111][1]=70, dp[1111][2]=65, dp[1111][3]=75

Close the tour (+ dist back to city 0):
  u=1: 70+10 = 80
  u=2: 65+15 = 80
  u=3: 75+20 = 95

Answer: min(80, 80, 95) = 80
Optimal tour: 0 → 1 → 3 → 2 → 0  = 10+25+30+15 = 80 ✓
```

**Why `O(2^n · n^2)` and why this beats brute force `O(n!)`:** brute force tries every permutation of cities — `(n-1)!` distinct tours. Held-Karp instead observes that the SPECIFIC ORDER in which a subset of cities was visited doesn't matter for future decisions — only WHICH subset has been visited and WHERE the tour currently stands. That collapses `(n-1)!` orderings down to `2^n` subsets × `n` endpoints, each transition costing `O(n)` (trying every previous endpoint) — for `n=15`, that's `~15! ≈ 1.3×10^12` vs Held-Karp's `~2^15·15^2 ≈ 7.4×10^6`, a five-orders-of-magnitude improvement.

---

## SECTION 7 — TEMPLATE 5: MINIMUM XOR SUM OF TWO ARRAYS (ASSIGNMENT DP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1879 — Minimum XOR Sum of Two Arrays
// Given nums1 and nums2 of equal length n, permute nums2 (call it P)
// to minimize sum over i of (nums1[i] XOR P[i]).
//
// KEY INSIGHT: This is a bipartite assignment problem — assign each
// index of nums2 to exactly one index of nums1. With n <= 14, bitmask
// DP over "which nums2 elements have been used" works, PROCESSING
// nums1 IN A FIXED ORDER (index by index) — the mask's popcount tells
// you exactly which nums1 index you're currently assigning.
//
// State: dp[mask] = min cost to assign nums2 elements in `mask` to the
//                    first popcount(mask) elements of nums1.
// Transition: from dp[mask], the NEXT nums1 index to fill is
//             i = popcount(mask). Try every unused j in nums2:
//             dp[mask | (1<<j)] = min(dp[mask | (1<<j)],
//                                      dp[mask] + (nums1[i] ^ nums2[j]))
// Answer: dp[(1<<n) - 1]
// ─────────────────────────────────────────────────────────────────

int minimumXORSum(vector<int>& nums1, vector<int>& nums2) {
    int n = nums1.size();
    vector<int> dp(1 << n, INT_MAX);
    dp[0] = 0;

    for (int mask = 0; mask < (1 << n); mask++) {
        if (dp[mask] == INT_MAX) continue;
        int i = __builtin_popcount(mask);     // which nums1 index we're placing next
        if (i >= n) continue;

        for (int j = 0; j < n; j++) {
            if (mask & (1 << j)) continue;    // nums2[j] already assigned
            int nmask = mask | (1 << j);
            dp[nmask] = min(dp[nmask], dp[mask] + (nums1[i] ^ nums2[j]));
        }
    }
    return dp[(1 << n) - 1];
}
```

**FULL TRACE for nums1=[1,2], nums2=[2,3]:**

```
n=2. dp[00]=0, all else INF.

mask=00, i=popcount(00)=0 (placing nums1[0]=1):
  j=0: nmask=01, dp[01] = 0 + (1^2=3) = 3
  j=1: nmask=10, dp[10] = 0 + (1^3=2) = 2

mask=01, i=popcount(01)=1 (placing nums1[1]=2), dp[01]=3:
  j=1 (j=0 used): nmask=11, dp[11] = min(INF, 3 + (2^3=1)) = 4

mask=10, i=popcount(10)=1 (placing nums1[1]=2), dp[10]=2:
  j=0 (j=1 used): nmask=11, dp[11] = min(4, 2 + (2^2=0)) = 2

Final dp[11] = 2

Answer: 2
Assignment: nums1[0]=1 ↔ nums2[1]=3 (cost 1^3=2), nums1[1]=2 ↔ nums2[0]=2 (cost 2^2=0)
Total = 2 ✓
```

**Why `popcount(mask)` replaces an explicit `i` dimension:** because we always assign nums1 indices in a FIXED order (0, 1, 2, ...), the number of nums2 elements used so far (`popcount(mask)`) IS the nums1 index being filled. This is the same trick as Section 4's `remainder` — deriving one coordinate of the state from the mask itself instead of storing it separately, cutting the DP table from `O(2^n · n)` cells conceptually tracked to just `O(2^n)`.

---

## SECTION 8 — TEMPLATE 6: MAXIMUM STUDENTS TAKING EXAM (ROW-BITMASK DP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1349 — Maximum Students Taking Exam
// m x n seat grid ('.' = usable, '#' = broken). Seat students so that
// no student can see another's answers: no student directly left/right
// of another, and no student diagonally front-left/front-right of
// another (i.e., row i, col j conflicts with row i+1, col j-1 or j+1).
// Maximize the number of seated students.
//
// State: dp[i][mask] = max students seated in rows 0..i, where `mask`
//                       is exactly which seats in row i are occupied.
// Validity of a row mask (checked once per row, independent of other rows):
//   1. mask must be a submask of that row's usable-seat mask.
//   2. mask must have no two ADJACENT bits set (no left/right neighbor).
// Compatibility between consecutive rows (prev = row i-1's mask):
//   3. mask & (prev << 1) == 0   (no student diagonally front-left)
//   4. mask & (prev >> 1) == 0   (no student diagonally front-right)
// Transition: dp[i][mask] = max over valid prev of dp[i-1][prev] + popcount(mask)
// ─────────────────────────────────────────────────────────────────

int maxStudents(vector<vector<char>>& seats) {
    int m = seats.size(), n = seats[0].size();
    vector<int> validMask(m, 0);
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (seats[i][j] == '.') validMask[i] |= (1 << j);

    auto noSelfAdjacent = [&](int mask) {
        return (mask & (mask << 1)) == 0;
    };

    vector<vector<int>> dp(m, vector<int>(1 << n, -1));

    // Row 0: no previous row to conflict with
    for (int mask = 0; mask < (1 << n); mask++) {
        if ((mask & validMask[0]) != mask) continue;   // must fit usable seats
        if (!noSelfAdjacent(mask)) continue;
        dp[0][mask] = __builtin_popcount(mask);
    }

    for (int i = 1; i < m; i++) {
        for (int mask = 0; mask < (1 << n); mask++) {
            if ((mask & validMask[i]) != mask) continue;
            if (!noSelfAdjacent(mask)) continue;

            for (int pmask = 0; pmask < (1 << n); pmask++) {
                if (dp[i-1][pmask] == -1) continue;
                if (mask & (pmask << 1)) continue;   // diagonal front-left conflict
                if (mask & (pmask >> 1)) continue;   // diagonal front-right conflict
                dp[i][mask] = max(dp[i][mask], dp[i-1][pmask] + __builtin_popcount(mask));
            }
        }
    }

    int ans = 0;
    for (int mask = 0; mask < (1 << n); mask++)
        ans = max(ans, dp[m-1][mask]);
    return ans;
}
```

**FULL TRACE for a 2×3 grid, row0 = ". # .", row1 = ". . .":**

```
n=3. validMask[0] = 0b101 (cols 0,2 usable). validMask[1] = 0b111 (all usable).

Row 0 valid masks (submasks of 101, no adjacent bits):
  mask=000: popcount 0  → dp[0][000]=0
  mask=001: popcount 1  → dp[0][001]=1
  mask=100: popcount 1  → dp[0][100]=1
  mask=101: popcount 2, check 101 & (101<<1=1010) = 0000 → OK → dp[0][101]=2

Row 1 valid masks (submasks of 111, no adjacent bits):
  000,001,010,100 valid trivially.
  011: 011 & 110 = 010 ≠ 0 → INVALID (adjacent)
  101: 101 & 1010 = 0000 → VALID, popcount 2
  110: 110 & 1100 = 0100 ≠ 0 → INVALID
  111: has adjacent bits → INVALID
  Valid row-1 masks: 000(0), 001(1), 010(1), 100(1), 101(2)

Compute dp[1][101] (best case — 2 students in row 1):
  Try pmask=000: no diagonal bits possible → candidate = dp[0][000]+2 = 0+2 = 2
  Try pmask=001: (001<<1)=010, 101&010=0 OK; (001>>1)=000, 101&000=0 OK
                 → candidate = dp[0][001]+2 = 1+2 = 3
  Try pmask=100: (100<<1)=1000, 101&1000=0 OK; (100>>1)=010, 101&010=0 OK
                 → candidate = dp[0][100]+2 = 1+2 = 3
  Try pmask=101: (101<<1)=1010, 101&1010=0 OK; (101>>1)=010, 101&010=0 OK
                 → candidate = dp[0][101]+2 = 2+2 = 4
  dp[1][101] = max(2,3,3,4) = 4

Answer: 4
(Row 0 seats cols {0,2}, row 1 seats cols {0,2} — no column shares a diagonal
 with the blocked middle seat, and columns 0/2 aren't adjacent to each other.)
```

**Why this is `O(m · 4^n)` in the worst case, not `O(m · 2^n)`:** for every row, we compare EVERY candidate mask against EVERY possible previous-row mask — that's `2^n × 2^n = 4^n` pairs per row transition. With `n ≤ 8` (LC constraint), `4^8 ≈ 65536`, trivially fast. This is a different flavor from the `3^n` submask-enumeration bound in Section 2 — here we enumerate PAIRS of independent masks (current row, previous row), not (mask, submask-of-that-same-mask) pairs.

---

## SECTION 9 — TEMPLATE 7: NUMBER OF WAYS TO WEAR DIFFERENT HATS (ITERATE OVER ITEMS, NOT PEOPLE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1434 — Number of Ways to Wear Different Hats to Each Other
// n people (n <= 10), each with a list of preferred hats (hat IDs 1..40).
// Count the number of ways to assign a DISTINCT hat to every person.
//
// CRITICAL DESIGN CHOICE: with up to 40 hats but only <=10 people, the
// mask MUST represent "which people are covered," NOT "which hats are
// used" (a 40-bit mask would blow the state space). The outer loop
// iterates over HATS (1..40); the mask tracks PEOPLE.
//
// State: dp[h][mask] = number of ways to assign hats 1..h such that
//                       exactly the people in `mask` have a hat so far.
// Transition, for hat h:
//   Option A — no one takes hat h: dp[h][mask] = dp[h-1][mask]
//   Option B — some eligible, not-yet-covered person p takes hat h:
//              dp[h][mask] += dp[h-1][mask without p]  for each such p
// Answer: dp[40][(1<<n) - 1]
// ─────────────────────────────────────────────────────────────────

int numberWays(vector<vector<int>>& hats) {
    const int MOD = 1e9 + 7;
    int n = hats.size();
    const int MAX_HAT = 40;

    vector<vector<int>> peopleForHat(MAX_HAT + 1);
    for (int p = 0; p < n; p++)
        for (int h : hats[p])
            peopleForHat[h].push_back(p);

    vector<vector<int>> dp(MAX_HAT + 1, vector<int>(1 << n, 0));
    dp[0][0] = 1;   // base case: 0 hats processed, 0 people covered, 1 way (do nothing)

    for (int h = 1; h <= MAX_HAT; h++) {
        for (int mask = 0; mask < (1 << n); mask++) {
            int ways = dp[h-1][mask];                          // Option A: skip hat h
            for (int p : peopleForHat[h]) {
                if (mask & (1 << p)) {                          // p is covered in this mask
                    ways = (ways + dp[h-1][mask ^ (1 << p)]) % MOD;   // Option B: p wears hat h
                }
            }
            dp[h][mask] = ways;
        }
    }
    return dp[MAX_HAT][(1 << n) - 1];
}
```

**FULL TRACE for a 3-person instance: person0 likes {3,4}, person1 likes {4,5}, person2 likes {5}:**

```
peopleForHat[3]=[0], peopleForHat[4]=[0,1], peopleForHat[5]=[1,2]
Hats 1,2 have no interested people → dp[h][mask]=dp[h-1][mask] unchanged (passthrough).
dp[2][000]=1, all other dp[2][mask]=0.

h=3 (peopleForHat[3]=[0]):
  mask=000: ways = dp[2][000]=1 (bit0 not set, no Option B)          → dp[3][000]=1
  mask=001: ways = dp[2][001]=0 + dp[2][000]=1 (person0 takes hat3)  → dp[3][001]=1
  all other masks: dp[2][mask]=0 and Option B also references 0     → dp[3][mask]=0

h=4 (peopleForHat[4]=[0,1]):
  mask=000: ways = dp[3][000] = 1                                             → dp[4][000]=1
  mask=001: ways = dp[3][001]=1 (skip) + dp[3][000]=1 (p0 takes hat4)         → dp[4][001]=2
  mask=010: ways = dp[3][010]=0 (skip) + dp[3][000]=1 (p1 takes hat4)         → dp[4][010]=1
  mask=011: ways = dp[3][011]=0 (skip)
                    + dp[3][011^001=010]=0 (p0 takes hat4, p1 pre-covered how? =0)
                    + dp[3][011^010=001]=1 (p1 takes hat4, p0 already covered by hat3)
                                                                                → dp[4][011]=1
  masks with bit2 set (person2, no hat4 interest, and dp[3] there is 0 anyway) → all 0

h=5 (peopleForHat[5]=[1,2]), we only need the full mask 111:
  mask=111: ways = dp[4][111]=0 (skip, passthrough)
                    + [bit1 set] dp[4][111^010=101]=0  (person2 has no hat before 5, dp[4][101]=0)
                    + [bit2 set] dp[4][111^100=011]=1  (person2 takes hat5, {0,1} already covered)
                                                                                → dp[5][111]=1

Hats 6..40: no one likes them → dp[h][111] stays 1 all the way to dp[40][111].

Answer: 1
(Forced chain: person2 MUST take hat5 (only option) → person1 can't take hat5,
 so MUST take hat4 → person0 can't take hat4, so MUST take hat3. Exactly 1 way.) ✓
```

**Why iterate hats in the outer loop instead of people:** with up to 40 distinct hat IDs but only ≤10 people, a mask over people is `2^10 = 1024` states — tiny. A mask over hats would need `2^40` states — impossible. The dimension you put "outside" the mask (iterated with a plain loop) must be the LARGE one; the dimension INSIDE the mask must be the SMALL one. Getting this backwards is Mistake #1 in Section 15.

---

## SECTION 10 — TEMPLATE 8: FIND THE SHORTEST SUPERSTRING (TSP ON OVERLAP GRAPH)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 943 — Find the Shortest Superstring
// Given words (n <= 12), find the shortest string that contains every
// word as a substring.
//
// KEY INSIGHT: reduce to TSP. Build overlap[i][j] = length of the
// longest suffix of words[i] that is also a prefix of words[j] — this
// is the "savings" from placing word j immediately after word i.
// The shortest superstring corresponds to the Hamiltonian path (not
// cycle — no need to return to start) through the overlap graph that
// MAXIMIZES total overlap, equivalently MINIMIZES total length.
//
// State: dp[mask][i] = min length of a string that (a) contains every
//                       word in `mask` as a substring, arranged via
//                       maximal pairwise overlaps, and (b) ends with
//                       word i as its suffix.
// Transition: dp[mask][i] = min over j in mask\{i} of
//             dp[mask\{i}][j] + len(words[i]) - overlap[j][i]
// ─────────────────────────────────────────────────────────────────

string shortestSuperstring(vector<string>& words) {
    int n = words.size();
    vector<vector<int>> overlap(n, vector<int>(n, 0));
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i == j) continue;
            int maxLen = min(words[i].size(), words[j].size());
            for (int k = maxLen; k > 0; k--) {
                if (words[i].substr(words[i].size() - k) == words[j].substr(0, k)) {
                    overlap[i][j] = k;
                    break;
                }
            }
        }
    }

    vector<vector<int>> dp(1 << n, vector<int>(n, -1));
    vector<vector<int>> parent(1 << n, vector<int>(n, -1));

    for (int mask = 1; mask < (1 << n); mask++) {
        for (int i = 0; i < n; i++) {
            if (!(mask & (1 << i))) continue;
            int pmask = mask ^ (1 << i);
            if (pmask == 0) { dp[mask][i] = words[i].size(); continue; }

            dp[mask][i] = -1;
            for (int j = 0; j < n; j++) {
                if (!(pmask & (1 << j))) continue;
                int cand = dp[pmask][j] + (int)words[i].size() - overlap[j][i];
                if (dp[mask][i] == -1 || cand < dp[mask][i]) {
                    dp[mask][i] = cand;
                    parent[mask][i] = j;
                }
            }
        }
    }

    int full = (1 << n) - 1, last = 0, best = INT_MAX;
    for (int i = 0; i < n; i++)
        if (dp[full][i] < best) { best = dp[full][i]; last = i; }

    // Reconstruct the ordering by walking parent pointers backward
    vector<int> order;
    int mask = full, cur = last;
    while (cur != -1) {
        order.push_back(cur);
        int pcur = parent[mask][cur];
        mask ^= (1 << cur);
        cur = pcur;
    }
    reverse(order.begin(), order.end());

    string ans = words[order[0]];
    for (size_t k = 1; k < order.size(); k++) {
        int prevIdx = order[k-1], idx = order[k];
        ans += words[idx].substr(overlap[prevIdx][idx]);
    }
    return ans;
}
```

**FULL TRACE for words = ["abc", "bcx", "xdef"]:**

```
Overlaps:  overlap[abc][bcx] = 2 ("bc")     overlap[bcx][xdef] = 1 ("x")
           all other pairs = 0

n=3. Base cases (single word):
  dp[001][0]=3(abc)  dp[010][1]=3(bcx)  dp[100][2]=4(xdef)

mask=011 {abc,bcx}:
  i=0(abc), pmask=010: dp[010][1]+3-overlap[bcx][abc](=0) = 3+3-0=6  → dp[011][0]=6
  i=1(bcx), pmask=001: dp[001][0]+3-overlap[abc][bcx](=2) = 3+3-2=4  → dp[011][1]=4

mask=101 {abc,xdef}:
  i=0(abc), pmask=100: dp[100][2]+3-overlap[xdef][abc](=0) = 4+3-0=7 → dp[101][0]=7
  i=2(xdef),pmask=001: dp[001][0]+4-overlap[abc][xdef](=0) = 3+4-0=7 → dp[101][2]=7

mask=110 {bcx,xdef}:
  i=1(bcx), pmask=100: dp[100][2]+3-overlap[xdef][bcx](=0) = 4+3-0=7 → dp[110][1]=7
  i=2(xdef),pmask=010: dp[010][1]+4-overlap[bcx][xdef](=1) = 3+4-1=6 → dp[110][2]=6

mask=111 (full):
  i=0(abc), pmask=110: try j=1: dp[110][1]+3-overlap[bcx][abc]=7+3-0=10
                        try j=2: dp[110][2]+3-overlap[xdef][abc]=6+3-0=9
                        → dp[111][0]=9, parent=2
  i=1(bcx), pmask=101: try j=0: dp[101][0]+3-overlap[abc][bcx]=7+3-2=8
                        try j=2: dp[101][2]+3-overlap[xdef][bcx]=7+3-0=10
                        → dp[111][1]=8, parent=0
  i=2(xdef),pmask=011: try j=0: dp[011][0]+4-overlap[abc][xdef]=6+4-0=10
                        try j=1: dp[011][1]+4-overlap[bcx][xdef]=4+4-1=7
                        → dp[111][2]=7, parent=1

Best ending state: min(9,8,7) = 7, achieved at i=2 (xdef), parent=1 (bcx)

Reconstruct: cur=2 → order=[2], parent[111][2]=1, mask→011
             cur=1 → order=[2,1], parent[011][1]=0, mask→001
             cur=0 → order=[2,1,0], parent[001][0]=-1, STOP
Reversed order: [0, 1, 2] = abc, bcx, xdef

Build string: ans="abc"
  append bcx.substr(overlap[abc][bcx]=2) = "x"    → ans="abcx"
  append xdef.substr(overlap[bcx][xdef]=1) = "def" → ans="abcxdef"

Answer: "abcxdef" (length 7 = 3+3+4 - 2 - 1) ✓
```

**Why Hamiltonian PATH, not CYCLE:** unlike TSP (Section 6) where the salesman must return to the start, a superstring has a definite first and last word — there's no "closing the loop" cost. This is why `shortestSuperstring`'s answer is `min over all i of dp[full][i]` directly, with no `+ dist[i][start]` term, whereas Held-Karp adds `+ dist[u][0]` to close the tour.

---

## SECTION 11 — TEMPLATE 9: MAXIMIZE SCORE AFTER N OPERATIONS (PAIR-MATCHING WITH GCD)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1799 — Maximize Score After N Operations
// nums has 2*m integers. In operation k (1-indexed, k=1..m), pick any
// two REMAINING numbers x,y and add k * gcd(x,y) to the score, then
// remove both. Maximize total score.
//
// KEY INSIGHT: only EVEN-popcount masks are meaningful states — you
// can only be "between operations" after an even number of elements
// have been consumed (each operation removes exactly 2). The operation
// number about to complete is DERIVED from popcount, not stored
// separately: if `mask` has popcount `cnt`, we are completing
// operation number `cnt / 2`.
//
// State: dp[mask] = max score achievable using exactly the elements in
//                    `mask` (popcount(mask) must be even).
// Transition: dp[mask] = max over pairs (i,j) in mask of
//             dp[mask \ {i,j}] + (popcount(mask)/2) * gcd(nums[i],nums[j])
// Answer: dp[(1<<n) - 1]
// ─────────────────────────────────────────────────────────────────

int maxScore(vector<int>& nums) {
    int n = nums.size();
    vector<int> dp(1 << n, 0);

    for (int mask = 1; mask < (1 << n); mask++) {
        int cnt = __builtin_popcount(mask);
        if (cnt % 2 != 0) continue;             // only even-sized masks are valid states
        int op = cnt / 2;                        // operation number being completed

        for (int i = 0; i < n; i++) {
            if (!(mask & (1 << i))) continue;
            for (int j = i + 1; j < n; j++) {
                if (!(mask & (1 << j))) continue;
                int pmask = mask ^ (1 << i) ^ (1 << j);
                int score = op * __gcd(nums[i], nums[j]);
                dp[mask] = max(dp[mask], dp[pmask] + score);
            }
        }
    }
    return dp[(1 << n) - 1];
}
```

**FULL TRACE for nums = [3,4,6,8]:**

```
n=4, indices: 0=3, 1=4, 2=6, 3=8
gcd pairs: gcd(3,4)=1 gcd(3,6)=3 gcd(3,8)=1 gcd(4,6)=2 gcd(4,8)=4 gcd(6,8)=2

cnt=2 masks (op=1, score = 1 * gcd):
  {0,1}=0011: dp = 1*gcd(3,4) = 1
  {0,2}=0101: dp = 1*gcd(3,6) = 3
  {0,3}=1001: dp = 1*gcd(3,8) = 1
  {1,2}=0110: dp = 1*gcd(4,6) = 2
  {1,3}=1010: dp = 1*gcd(4,8) = 4
  {2,3}=1100: dp = 1*gcd(6,8) = 2

cnt=4 mask=1111 (op=2, score = 2 * gcd):
  pair(0,1): pmask={2,3}, dp[pmask]=2, score=2*gcd(3,4)=2   → total 4
  pair(0,2): pmask={1,3}, dp[pmask]=4, score=2*gcd(3,6)=6   → total 10
  pair(0,3): pmask={1,2}, dp[pmask]=2, score=2*gcd(3,8)=2   → total 4
  pair(1,2): pmask={0,3}, dp[pmask]=1, score=2*gcd(4,6)=4   → total 5
  pair(1,3): pmask={0,2}, dp[pmask]=3, score=2*gcd(4,8)=8   → total 11  ← best
  pair(2,3): pmask={0,1}, dp[pmask]=1, score=2*gcd(6,8)=4   → total 5

dp[1111] = max(4,10,4,5,11,5) = 11

Answer: 11
(Operation 1 pairs {3,6}: 1*gcd(3,6)=3. Operation 2 pairs {4,8}: 2*gcd(4,8)=8. Total 3+8=11.) ✓
```

**Why deriving `op` from `popcount(mask)/2` matters:** the naive state would be `(operationIndex, mask)` — a 3D-feeling DP. But `operationIndex` is entirely determined by how many elements are already consumed (`popcount(mask)/2` operations have completed to REACH this mask, so the operation about to run to LEAVE it going forward is `popcount(mask)/2 + 1`... careful: in the code above we compute the score for arriving AT `mask` from a smaller `pmask`, so `op = popcount(mask)/2` is exactly the operation index that was just completed to produce `mask`. This mirrors Section 7's `popcount` trick precisely.

---

## SECTION 12 — TEMPLATE 10: CAN I WIN (GAME THEORY + MEMO ON USED-MASK)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 464 — Can I Win
// Two players alternately pick an unused integer from 1..maxChoosableInteger
// (without replacement). The player whose picks make the CUMULATIVE
// total reach >= desiredTotal wins. Determine if the first player can
// force a win with optimal play.
//
// KEY INSIGHT: the mask isn't spatial — it's a GAME-STATE fingerprint
// of which numbers have been used. Because both players play optimally
// and the rules are symmetric, dp[mask] fully determines whether the
// PLAYER-TO-MOVE from that state can force a win — we do NOT need to
// track "whose turn" as a separate flag, because popcount(mask) parity
// combined with memoizing "can the mover win" already captures it
// implicitly through the recursive minimax structure.
//
// State: memo[mask] = can the player about to move force a win, given
//                      that `mask` numbers are already used and
//                      `remaining` more total is needed?
// (remaining is threaded through the recursion, not memoized on —
//  it is a deterministic function of mask + desiredTotal, but passing
//  it explicitly avoids recomputing it from mask every call.)
// ─────────────────────────────────────────────────────────────────

bool canIWin(int maxChoosableInteger, int desiredTotal) {
    if (desiredTotal <= 0) return true;
    int sumAll = maxChoosableInteger * (maxChoosableInteger + 1) / 2;
    if (sumAll < desiredTotal) return false;      // even taking everything isn't enough
    if (maxChoosableInteger >= 31) return false;   // guard against mask overflow (see Mistake 2)

    unordered_map<int,int> memo;   // mask -> 1 (mover wins) / -1 (mover loses)

    function<bool(int,int)> dfs = [&](int mask, int remaining) -> bool {
        if (memo.count(mask)) return memo[mask] == 1;

        for (int i = 1; i <= maxChoosableInteger; i++) {
            if (mask & (1 << i)) continue;                       // already used
            // Picking i wins immediately if it reaches/exceeds remaining,
            // OR if it leaves the OPPONENT in a state where THEY cannot win.
            if (i >= remaining || !dfs(mask | (1 << i), remaining - i)) {
                memo[mask] = 1;
                return true;
            }
        }
        memo[mask] = -1;
        return false;
    };

    return dfs(0, desiredTotal);
}
```

**FULL TRACE for maxChoosableInteger=4, desiredTotal=6:**

```
sumAll = 1+2+3+4 = 10 >= 6, so a forced win is possible for someone.

dfs(0000, remaining=6):
  try i=1: 1>=6? no. recurse dfs(0010, remaining=5)   [bit for i=1 is (1<<1)=0010]

    dfs(0010, remaining=5):
      try i=2: 2>=5? no. recurse dfs(0110, remaining=3)

        dfs(0110, remaining=3):
          try i=3: 3>=3? YES → immediate win
        dfs(0110,3) = TRUE

      back in dfs(0010,5): i=2 gives !dfs(...)=!true=false, i>=remaining=false
                            → combined false, this choice does NOT win — try next i
      try i=3: 3>=5? no. recurse dfs(1010, remaining=2)

        dfs(1010, remaining=2):
          try i=2: 2>=2? YES → immediate win
        dfs(1010,2) = TRUE

      i=3 also fails to win (same reasoning) — try next i
      try i=4: 4>=5? no. recurse dfs(10010, remaining=1)

        dfs(10010, remaining=1):
          try i=2: 2>=1? YES → immediate win
        dfs(10010,1) = TRUE

      i=4 also fails to win — all options exhausted
    dfs(0010, 5) = FALSE   (the mover here, i.e. player 2, LOSES)

  back in dfs(0000,6): i=1 gives !dfs(0010,5) = !false = TRUE
                        → combined with OR: TRUE. WIN via i=1, short-circuit — stop here.

dfs(0000, 6) = TRUE

Answer: TRUE
(Player 1 opens with 1. Whatever Player 2 does next among {2,3,4}, Player 1 can
 always complete the total to >=6 on their following turn — verified by exhaustive
 game-tree search above: every one of player 2's replies leads to a state where
 the ORIGINAL first player wins.) ✓
```

**Why no explicit "whose turn" flag is needed:** `dfs(mask, remaining)` always answers "can the player about to move from THIS state force a win?" — this is symmetric regardless of whether it's logically "player 1" or "player 2" moving, because both play with the same optimal strategy function. The alternation is handled implicitly by each recursive call flipping the question to the OTHER side ("does MY opponent, moving from the resulting state, win?"), and `!dfs(...)` inverts that answer back to "so do I lose or win by making this move." This is standard game-theory DP (see Pattern 16), specialized here with a bitmask as the state key instead of an array index.

---

## SECTION 13 — COMPLEXITY TABLE

| Problem | State Space | Transition Cost | Total Time | Space |
|---------|-------------|------------------|------------|-------|
| Subsets (LC78) | `2^n` | `O(n)` to build each subset | `O(2^n · n)` | `O(2^n · n)` |
| Partition K Subsets (LC698) | `2^n` | `O(n)` (with pruning, much less in practice) | `O(2^n · n)` | `O(2^n)` |
| Shortest Path Visiting All Nodes (LC847) | `n · 2^n` | `O(deg(v))` per state | `O(n · 2^n · n)` = `O(n^2 · 2^n)` | `O(n · 2^n)` |
| Held-Karp TSP | `n · 2^n` | `O(n)` per state | `O(2^n · n^2)` | `O(n · 2^n)` |
| Min XOR Sum (LC1879) | `2^n` | `O(n)` per state | `O(2^n · n)` | `O(2^n)` |
| Max Students (LC1349) | `m · 2^n` | `O(2^n)` per state (row-pair check) | `O(m · 4^n)` | `O(2^n)` (rolling rows) |
| Different Hats (LC1434) | `40 · 2^n` | `O(n)` per state | `O(40 · 2^n · n)` | `O(40 · 2^n)` or `O(2^n)` rolling |
| Shortest Superstring (LC943) | `n · 2^n` | `O(n)` per state | `O(2^n · n^2)` | `O(n · 2^n)` |
| Maximize Score (LC1799) | `2^n` | `O(n^2)` per state (pair enumeration) | `O(2^n · n^2)` | `O(2^n)` |
| Can I Win (LC464) | `2^n` | `O(n)` per state | `O(2^n · n)` | `O(2^n)` (memo map) |
| SOS / submask DP (Parallel Courses II style) | `2^n` masks, `Σ submasks` | — | `O(3^n)` | `O(2^n)` |

**Rule of thumb for feasibility:**
- `O(2^n · n)` — comfortable up to `n ≈ 20–22`
- `O(2^n · n^2)` — comfortable up to `n ≈ 15–18`
- `O(4^n)` (mask-pair enumeration, e.g. Max Students) — comfortable up to `n ≈ 8–10`
- `O(3^n)` (submask enumeration) — comfortable up to `n ≈ 18–20`

---

## SECTION 14 — VARIANTS

1. **Count Square Submatrices ↔ Subsets:** not really related, but the "derive one field of state from popcount" trick (Sections 7, 11) generalizes to ANY assignment DP where items are processed in a fixed, predetermined order.

2. **Broken Profile DP (tiling problems, e.g., "Domino and Tromino Tiling," "Number of Ways to Tile a Region"):** a close cousin of Max Students (Section 8) — `dp[col][mask]` where `mask` represents the "broken profile" boundary between filled and unfilled cells. Same row-to-row mask transition idea, different validity/compatibility rules.

3. **SOS (Sum over Subsets) DP:** `dp[mask] = Σ over all submasks sub of f(sub)`, computed in `O(2^n · n)` using a "bit-by-bit" relaxation (NOT full `O(3^n)` submask enumeration, which SOS DP specifically improves upon):
   ```cpp
   for (int i = 0; i < n; i++)
       for (int mask = 0; mask < (1<<n); mask++)
           if (mask & (1<<i))
               dp[mask] += dp[mask ^ (1<<i)];
   ```
   This is worth knowing exists — it's the `O(2^n·n)` upgrade over the `O(3^n)` submask loop when you need EVERY mask's full submask sum, not just a one-off query.

4. **Parallel Courses II (LC1494, in the problem set below):** `dp[mask]` = min semesters to complete courses in `mask`, with a `k`-per-semester cap and prerequisite masks. The transition enumerates submasks of "eligible, not-yet-taken courses" to pick each semester's batch — a direct `O(3^n)`-flavored application (bounded further by the "pick up to k" filter).

5. **Held-Karp generalizes to any "visit all, minimize/maximize path cost" problem** — Shortest Superstring (Hamiltonian path) and TSP (Hamiltonian cycle) are the two canonical shapes; recognize which one a new problem wants (does it need to "return to start" or not?) before coding.

6. **Bitmask + BFS/0-1 BFS hybrid:** Section 5's `(node, mask)` state generalizes to weighted graphs via Dijkstra over the same state space, or to 0/1 edge weights via deque-based 0-1 BFS — same state design, different traversal engine.

---

## SECTION 15 — COMMON MISTAKES

### Mistake 1: Iterating people vs. items in the wrong dimension

```cpp
// WRONG — mask over the 40 hats, loop over people
// 2^40 states — instant TLE/MLE, completely infeasible
vector<int> dp(1 << 40, 0);   // does not even compile in reasonable memory

// CORRECT — mask over the (small) people set, loop over the (large) hat range
vector<vector<int>> dp(41, vector<int>(1 << n, 0));   // n <= 10 people
for (int h = 1; h <= 40; h++)
    for (int mask = 0; mask < (1 << n); mask++)
        // ...

// THE RULE: whichever dimension is SMALL goes inside the mask.
// Whichever dimension is LARGE stays as a plain outer loop variable.
```

---

### Mistake 2: `1 << n` integer overflow when n is large

```cpp
// WRONG — n can be up to 20 for canIWin's ARGUMENT maxChoosableInteger,
// but the mask needs maxChoosableInteger+1 bits (numbers 1..n, bit 0 unused).
// If maxChoosableInteger were, say, 31, then (1 << 31) is undefined
// behavior for a 32-bit signed int (sign bit), and 1 << 32 is UB outright.
int mask = 1 << maxChoosableInteger;   // UB / wraps to INT_MIN or worse when >=31

// CORRECT — guard the range explicitly, or use a wider type
if (maxChoosableInteger >= 31) return false;   // sumAll check already makes this moot
                                                 // for any winnable instance in practice
// OR, when genuinely needed:
long long mask = 1LL << n;   // promotes to 64-bit, safe up to n=62

// THE RULE: before writing `1 << n`, ask "what's the max n this problem allows?"
// n <= 20   → plain `int`/`1 << n` is safe (2^20 ≈ 1M, well under INT_MAX)
// n in 21..30 → still fits in int, but double-check 2^n * (per-state cost) memory
// n >= 31   → MUST use `1LL << n` / `unsigned` / reject the input as infeasible
```

---

### Mistake 3: Submask enumeration loop bug (off-by-one on termination)

```cpp
// WRONG — this loop NEVER TERMINATES when mask == 0, and even for
// mask > 0 it stops one iteration too early (skips sub == 0)
for (int sub = mask; sub > 0; sub = (sub - 1) & mask) {
    // process sub
}
// If mask=0, the loop body never runs at all (sub=0 fails sub>0), which
// is usually fine — but many solutions ALSO need to process sub==0
// (the empty submask) and this form silently skips it.

// WRONG — this version infinite-loops because (0-1) & mask on signed int
// underflows to -1, and -1 & mask == mask again (assuming mask's bits
// are a subset of an all-1s pattern), potentially cycling forever
// depending on comparison operator used.

// CORRECT — the standard do/while-style idiom that correctly includes sub==0
for (int sub = mask; ; sub = (sub - 1) & mask) {
    // process sub   (this DOES include sub == 0 as the final iteration)
    if (sub == 0) break;
}
```

---

### Mistake 4: Using `popcount` to derive turn/count when the mapping isn't actually 1-to-1

```cpp
// WRONG — Maximize Score After N Operations: assuming EVERY mask
// (not just even-popcount ones) is a valid "between operations" state
for (int mask = 1; mask < (1 << n); mask++) {
    int op = __builtin_popcount(mask) / 2;   // silently WRONG for odd popcount
    // ... proceeds to compute garbage dp[mask] for odd-popcount masks,
    // which then gets read by some other transition as if valid
}

// CORRECT — explicitly skip masks that can't correspond to a real
// "boundary between operations" (each operation consumes exactly 2 elements)
for (int mask = 1; mask < (1 << n); mask++) {
    int cnt = __builtin_popcount(mask);
    if (cnt % 2 != 0) continue;              // ODD popcount is never a valid state here
    int op = cnt / 2;
    // ...
}

// THE RULE: popcount-derived fields are only safe when you've proven
// the mapping is well-defined for EVERY mask you actually visit — audit
// which masks are reachable/meaningful before trusting the derived value.
```

---

### Mistake 5: Held-Karp — forgetting to pin the start city inside the mask

```cpp
// WRONG — allows masks that don't include city 0, producing dp[mask][i]
// entries for "partial tours" that could never actually be extended back
// to a valid full tour starting at 0, wasting time and risking wrong answers
// if the base case isn't ALSO fixed correctly
vector<vector<int>> dp(1 << n, vector<int>(n, INF));
dp[0][0] = 0;   // BUG: mask=0 means "no cities visited," but we're claiming
                // to stand AT city 0 with nobody visited — contradictory state

// CORRECT — mask must include the start city from step 1, matching where
// the "standing at" pointer claims to be
dp[1][0] = 0;   // mask = 0...01 = {city 0}, standing at city 0, cost 0
for (int mask = 1; mask < (1 << n); mask++) {
    if (!(mask & 1)) continue;   // skip masks that don't even contain city 0
    // ...
}
```

---

### Mistake 6: Assignment DP (Min XOR Sum style) — reusing the same array index twice

```cpp
// WRONG — forgetting to check whether index j (from the SECOND array)
// has already been used, allowing the same nums2[j] to be assigned to
// two different nums1[i]'s
for (int j = 0; j < n; j++) {
    int nmask = mask | (1 << j);           // BUG: no check that j is unused!
    dp[nmask] = min(dp[nmask], dp[mask] + (nums1[i] ^ nums2[j]));
}
// This silently overwrites nmask entries incorrectly and can even let
// dp[mask] == dp[nmask] (mask unchanged) if bit j was already set,
// corrupting the DP order (processing a mask "before" itself).

// CORRECT — always skip already-used indices before extending the mask
for (int j = 0; j < n; j++) {
    if (mask & (1 << j)) continue;          // nums2[j] already assigned — MUST skip
    int nmask = mask | (1 << j);
    dp[nmask] = min(dp[nmask], dp[mask] + (nums1[i] ^ nums2[j]));
}
```

---

## SECTION 16 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Subsets | 78 | mask ↔ subset bijection, foundational enumeration |
| 2 | Partition to K Equal Sum Subsets | 698 | `sum % target` trick to avoid a group-index dimension |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Shortest Path Visiting All Nodes | 847 | `(node, mask)` state, multi-source BFS |
| 4 | Minimum XOR Sum of Two Arrays | 1879 | assignment DP, `popcount` as implicit index |
| 5 | Maximum Students Taking Exam | 1349 | row-bitmask DP, validity + adjacency compatibility masks |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Number of Ways to Wear Different Hats to Each Other | 1434 | iterate LARGE dimension outside, SMALL dimension in mask |
| 7 | Find the Shortest Superstring | 943 | Hamiltonian PATH on overlap graph, reconstruction via parent pointers |
| 8 | Parallel Courses II | 1494 | submask enumeration of "eligible" courses, `O(3^n)`-flavored DP |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Maximize Score After N Operations | 1799 | pair matching, `gcd`, `popcount/2` as operation index |
| 10 | Can I Win | 464 | game-theory minimax over a used-number mask |
| 11 | Beautiful Arrangement | 526 | mask = positions filled, backtracking with divisibility pruning |

---

## SECTION 17 — PATTERN CONNECTIONS

1. **Pattern 15 (Backtracking):** Every bitmask DP started life as a backtracking solution — `dp[mask]` is literally memoized backtracking, where `mask` is the "path so far" that backtracking would otherwise re-derive from scratch on every call. Subsets (LC78), Partition K Subsets (LC698), and Beautiful Arrangement (LC526) are explicitly backtracking problems that bitmask DP either accelerates (via memoization) or simply reframes as iteration.

2. **Pattern 12 / Pattern 30 (Graph Algorithms — BFS/shortest path, and advanced graph DP):** Shortest Path Visiting All Nodes (LC847) is a direct fusion of Pattern 12's BFS machinery with a bitmask state dimension. Held-Karp TSP and Shortest Superstring are both "Hamiltonian path/cycle on a complete weighted graph" — the graph-theory framing (build a distance/overlap matrix, then solve TSP) is the bridge from raw strings/lists into a graph problem Pattern 30 already has tools for.

3. **Pattern 16 (DP / Game Theory DP):** Can I Win (LC464) is a textbook game-theory DP (two-player, optimal-play, win/lose determination) where the state representation happens to be a bitmask instead of an array index or interval `[l,r]`. The recursive minimax structure (`win iff exists a move such that the opponent, from the resulting state, loses`) is identical to Stone Game / Predict the Winner from Pattern 16 — only the state key changes.

4. **Pattern 19 (Knapsack Family):** Partition to K Equal Sum Subsets (LC698) is a bitmask-flavored generalization of subset-sum — instead of asking "does SOME subset sum to target," it asks "can the FULL set be partitioned into exactly k subsets each summing to target." The `remainder = currentSum % target` trick is a direct descendant of knapsack's "remaining capacity" state.

5. **Pattern 06 (Prefix Sums) / Pattern 04 (Monotonic Structures) — indirectly:** the row-validity precomputation in Max Students (`validMask[i]`, `noSelfAdjacent`) is conceptually a "precompute once per row" optimization in the same spirit as prefix sums — front-loading cheap-to-compute per-row facts so the expensive `O(4^n)` transition loop doesn't redo them.

---

## SECTION 18 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 464 — Can I Win

**Interviewer:** "Two players alternately pick numbers from 1 to `maxChoosableInteger` without replacement. Whoever makes the running total reach `desiredTotal` first wins. Can the first player force a win?"

**Candidate:**

*[First minute — recognize the shape]*
> "`maxChoosableInteger` is small — this screams bitmask state. The state I need is 'which numbers have been used' — that's naturally a bitmask over `1..maxChoosableInteger`. Before I dive in, let me handle the trivial cases: if `desiredTotal <= 0`, player 1 trivially wins with zero picks. If the sum of ALL numbers is less than `desiredTotal`, NOBODY can ever reach it — impossible, return false."

*[State the recursive structure]*
> "`dfs(mask, remaining)` answers: can the player about to move force a win? For each unused number `i`: if picking `i` reaches or exceeds `remaining`, that's an immediate win. Otherwise, I recurse — if the OPPONENT, from the resulting state, cannot win (`!dfs(...)`), then I win by making this move. I memoize purely on `mask`, since `remaining` is a deterministic function of `mask` and the original `desiredTotal` — I don't need it as part of the memo key."

*[Flag the overflow trap before writing code]*
> "One risk: `1 << maxChoosableInteger` — if that value gets anywhere near 31, I'm in undefined-behavior territory with a 32-bit int. The `sumAll < desiredTotal` check already filters out most large-and-unwinnable cases, but I'll add an explicit guard for safety."

*[Code — 6 minutes]*
```cpp
bool canIWin(int maxChoosableInteger, int desiredTotal) {
    if (desiredTotal <= 0) return true;
    int sumAll = maxChoosableInteger * (maxChoosableInteger + 1) / 2;
    if (sumAll < desiredTotal) return false;
    if (maxChoosableInteger >= 31) return false;

    unordered_map<int,int> memo;
    function<bool(int,int)> dfs = [&](int mask, int remaining) -> bool {
        if (memo.count(mask)) return memo[mask] == 1;
        for (int i = 1; i <= maxChoosableInteger; i++) {
            if (mask & (1 << i)) continue;
            if (i >= remaining || !dfs(mask | (1 << i), remaining - i)) {
                return memo[mask] = 1;
            }
        }
        return memo[mask] = 0;
    };
    return dfs(0, desiredTotal);
}
```

*[Follow-up]: "Why is it safe to memoize only on `mask` and not on `(mask, remaining)`?"*
> "`remaining` is fully determined by `mask`: it equals `desiredTotal` minus the sum of all numbers whose bits are set in `mask`. So `(mask, remaining)` pairs never actually vary independently — `mask` alone is a sufficient memo key, and using it directly (instead of a compound key) roughly halves my hash map overhead."

**RED FLAGS:**
- Not checking `sumAll < desiredTotal` first (leads to infinite/very deep recursion on unwinnable inputs)
- Memoizing on `remaining` alone or on `(mask, remaining)` as if they vary independently
- Missing the overflow guard on `1 << maxChoosableInteger`

---

### Interview Simulation 2: Held-Karp TSP (generic version)

**Interviewer:** "Given `n` cities and a distance matrix, find the minimum-cost tour that visits every city exactly once and returns to the start. `n <= 15`."

**Candidate:**

*[Rule out brute force immediately, justify the complexity target]*
> "Brute force over permutations is `O(n!)` — for `n=15` that's over a trillion, infeasible. The key realization: for deciding the OPTIMAL continuation of a partial tour, I don't care about the ORDER in which I visited cities so far — only WHICH SET I've visited and WHERE I currently am. That collapses the state from a full permutation down to `(mask, currentCity)` — `2^n · n` states instead of `n!` orderings."

*[State the DP precisely before coding]*
> "`dp[mask][i]` = minimum cost to have visited exactly the cities in `mask` (mask always includes city 0, my fixed start), ending at city `i`. Transition: `dp[mask][v] = min over u in mask\{v} of dp[mask\{v}][u] + dist[u][v]`. Base case: `dp[{0}][0] = 0`. Final answer closes the loop: `min over i != 0 of dp[full][i] + dist[i][0]`."

*[Code — 5 minutes, then trace a tiny 3-city example out loud to sanity check before declaring done]*

*[Interviewer]: "What if `n` were up to 22 instead of 15?"*
> "`2^22 · 22^2 ≈ 2×10^9` — too slow for Held-Karp exactly as written within a typical time limit. At that scale I'd pivot to either approximate methods (nearest-neighbor + 2-opt local search, branch-and-bound with strong pruning) or accept that exact TSP isn't tractable and the interviewer is probing whether I recognize the ceiling of this technique rather than expecting a bigger exact solution."

**RED FLAGS:**
- Jumping straight to code without stating the `dp[mask][i]` semantics first
- Forgetting `dp[1][0] = 0` uses `mask=1` (binary `...001`), not `mask=0`
- Not recognizing when `n` has grown too large for `O(2^n · n^2)` to be the right tool

---

### Interview Simulation 3: Meta-discussion — When is bitmask DP the right tool?

**Interviewer:** "How do you decide a problem needs bitmask DP instead of a different DP shape?"

**Candidate:**
> "First signal: the constraint. If `n <= 20` or so, and the problem talks about subsets, assignments, 'visit all,' or 'use every element exactly once,' bitmask DP is the leading hypothesis.

> Second, I ask what a bit represents — is it 'this element is included,' 'this city is visited,' 'this person is covered,' or 'this number has been picked in a game'? Naming this precisely up front prevents half the bugs in this pattern.

> Third — and this is the part people skip — I ask whether the mask needs an EXTRA dimension attached, like a current position (TSP, Shortest Path Visiting All Nodes) or a current row (Max Students), versus standing alone (Subsets, Can I Win). If the transition needs to know not just 'what's been used' but also 'where am I right now,' the state is `(mask, extra)`, not just `mask`.

> Fourth, I look for whether one coordinate of that 'extra' dimension can be DERIVED from `popcount(mask)` instead of tracked explicitly — this is the difference between a clean `O(2^n)` table and an unnecessarily bloated `O(2^n · n)` one, and it comes up constantly (Min XOR Sum, Maximize Score, Partition K Subsets all use this)."

**RED FLAGS:**
- Reaching for bitmask DP when `n` is large (say `n > 25`) without questioning feasibility first
- Not distinguishing "mask alone" states from "mask + position" states
- Missing the `popcount`-as-implicit-index optimization when it clearly applies

---

## SECTION 19 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why does `sub = (sub - 1) & mask` correctly enumerate all submasks of `mask` in decreasing order, and why does that loop cost `O(3^n)` in total across all masks?**

> `sub - 1` flips the lowest set bit of `sub` off and sets all lower bits on (standard binary decrement). ANDing with `mask` then clips those spuriously-set lower bits back down to only the ones actually present in `mask`, producing exactly the next-smaller submask of `mask`. The `O(3^n)` bound comes from a per-element counting argument: across ALL `(mask, sub)` pairs where `sub ⊆ mask`, each of the `n` elements independently falls into exactly one of three buckets — absent from `mask`, present in `mask` but absent from `sub`, or present in both — giving `3^n` total pairs, not `4^n`.

---

**Q2: In Held-Karp, why is the base case `dp[1][0] = 0` and not `dp[0][0] = 0`?**

> `dp[mask][i]` means "having visited exactly the cities in `mask`, currently standing at city `i`." If `mask = 0`, no cities have been visited — but claiming to be "standing at city 0" while city 0 hasn't been visited is self-contradictory. The correct base case is `mask = 1` (binary `...0001`), meaning "the set of visited cities is exactly `{0}`," matching the claim of standing at city 0.

---

**Q3: In Minimum XOR Sum (LC1879), how does `popcount(mask)` eliminate the need for a separate "which nums1 index" dimension?**

> Because `nums1` is always processed in a FIXED index order (0, 1, 2, ...) regardless of which `nums2` elements have been consumed, the count of `nums2` elements assigned so far (`popcount(mask)`) is EXACTLY equal to how many `nums1` positions have been filled — which is exactly the next `nums1` index to assign. This works only because the "left side" of the assignment is processed in a predetermined order; if `nums1` could be assigned out of order, this trick would break.

---

**Q4: Why does Maximum Students Taking Exam run in `O(m · 4^n)` rather than `O(m · 2^n)`, and why is that acceptable?**

> The DP compares EVERY candidate current-row mask against EVERY candidate previous-row mask for compatibility — that's `2^n` choices times `2^n` choices per row, i.e., `4^n` pairs, times `m` rows. This is acceptable because LC1349 constrains `n <= 8`, so `4^8 = 65536` — trivially fast even multiplied by `m` rows and a constant-time compatibility check per pair.

---

**Q5: In Can I Win, why is it safe to memoize on `mask` alone, without also storing `remaining` in the memo key?**

> `remaining` is a pure function of `mask` and the fixed `desiredTotal`: `remaining = desiredTotal - (sum of numbers whose bits are set in mask)`. Since `mask` fully determines `remaining`, storing both in the memo key would be redundant — two calls with the same `mask` always have the same `remaining`, so `mask` alone is a sufficient (and more memory-efficient) memoization key.

---

**Q6: Why must Number of Ways to Wear Different Hats iterate over HATS in the outer loop and PEOPLE in the bitmask, rather than the reverse?**

> There can be up to 40 distinct hat IDs but only up to 10 people. A mask over hats would need `2^40` states — computationally impossible. A mask over people needs only `2^10 = 1024` states — trivial. The general rule: the SMALL-cardinality dimension goes inside the bitmask; the LARGE-cardinality dimension stays as an ordinary outer loop variable that the DP iterates through one value at a time.

---

**Q7: In Maximize Score After N Operations, why does the code skip every mask with odd `popcount`, and what would go wrong if it didn't?**

> Each operation consumes exactly 2 elements, so any state reachable "between operations" must have an EVEN number of elements already consumed — an odd-popcount mask corresponds to a state mid-operation, which never actually occurs as a boundary in this problem's mechanics. If the code didn't skip odd-popcount masks, it would compute a meaningless `dp[mask]` value for those masks (using `op = popcount(mask)/2`, e.g., `popcount=3` gives `op=1`, silently wrong), and — worse — later transitions FROM a valid even mask might read a neighboring odd mask's garbage value if the pairing logic weren't restricted to `mask ^ (1<<i) ^ (1<<j)` staying even by construction. The `cnt % 2 != 0: continue` guard prevents ever computing or trusting those invalid states.

---

## SECTION 20 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. CORE BIT OPERATIONS
// ─────────────────────────────────────────────────────────────────
mask |= (1 << i);              // set bit i
mask &= ~(1 << i);              // clear bit i
bool on = mask & (1 << i);      // test bit i
mask ^= (1 << i);               // toggle bit i
int cnt = __builtin_popcount(mask);      // count set bits (int)
int cnt64 = __builtin_popcountll(mask);  // count set bits (long long)
int low = mask & (-mask);        // isolate lowest set bit's VALUE
int idx = __builtin_ctz(mask);   // index of lowest set bit
mask &= (mask - 1);              // clear the lowest set bit

// ─────────────────────────────────────────────────────────────────
// 2. ITERATE ALL SET BITS OF A MASK
// ─────────────────────────────────────────────────────────────────
for (int m = mask; m > 0; m &= (m - 1)) {
    int i = __builtin_ctz(m);
    // process element i
}

// ─────────────────────────────────────────────────────────────────
// 3. ITERATE ALL SUBMASKS OF A MASK — O(3^n) total across all masks
// ─────────────────────────────────────────────────────────────────
for (int sub = mask; ; sub = (sub - 1) & mask) {
    // process sub (includes sub == mask and sub == 0)
    if (sub == 0) break;
}

// ─────────────────────────────────────────────────────────────────
// 4. SUBSETS ENUMERATION (LC 78 style)
// ─────────────────────────────────────────────────────────────────
for (int mask = 0; mask < (1 << n); mask++)
    for (int i = 0; i < n; i++)
        if (mask & (1 << i)) /* nums[i] is in this subset */;

// ─────────────────────────────────────────────────────────────────
// 5. HELD-KARP TSP SKELETON
// ─────────────────────────────────────────────────────────────────
vector<vector<int>> dp(1<<n, vector<int>(n, INF));
dp[1][0] = 0;
for (int mask = 1; mask < (1<<n); mask++) {
    if (!(mask & 1)) continue;
    for (int u = 0; u < n; u++) {
        if (!(mask & (1<<u)) || dp[mask][u] == INF) continue;
        for (int v = 0; v < n; v++) {
            if (mask & (1<<v)) continue;
            dp[mask | (1<<v)][v] = min(dp[mask|(1<<v)][v], dp[mask][u] + dist[u][v]);
        }
    }
}
// answer: min over i!=0 of dp[full][i] + dist[i][0]

// ─────────────────────────────────────────────────────────────────
// 6. (NODE, MASK) BFS SKELETON — Shortest Path Visiting All Nodes
// ─────────────────────────────────────────────────────────────────
queue<tuple<int,int,int>> q;              // (node, mask, dist)
vector<vector<bool>> visited(n, vector<bool>(1<<n, false));
for (int i = 0; i < n; i++) { q.push({i, 1<<i, 0}); visited[i][1<<i] = true; }
while (!q.empty()) {
    auto [node, mask, dist] = q.front(); q.pop();
    if (mask == (1<<n)-1) return dist;
    for (int next : graph[node]) {
        int nm = mask | (1<<next);
        if (!visited[next][nm]) { visited[next][nm]=true; q.push({next, nm, dist+1}); }
    }
}

// ─────────────────────────────────────────────────────────────────
// 7. ASSIGNMENT DP — popcount as implicit index (LC 1879 style)
// ─────────────────────────────────────────────────────────────────
vector<int> dp(1<<n, INT_MAX); dp[0] = 0;
for (int mask = 0; mask < (1<<n); mask++) {
    if (dp[mask] == INT_MAX) continue;
    int i = __builtin_popcount(mask);          // next left-side index, derived
    if (i >= n) continue;
    for (int j = 0; j < n; j++) {
        if (mask & (1<<j)) continue;
        int nm = mask | (1<<j);
        dp[nm] = min(dp[nm], dp[mask] + cost(i, j));
    }
}

// ─────────────────────────────────────────────────────────────────
// 8. GAME-THEORY MEMO ON USED-MASK (LC 464 style)
// ─────────────────────────────────────────────────────────────────
unordered_map<int,int> memo;
function<bool(int,int)> dfs = [&](int mask, int remaining) -> bool {
    if (memo.count(mask)) return memo[mask] == 1;
    for (int i = 1; i <= maxVal; i++) {
        if (mask & (1<<i)) continue;
        if (i >= remaining || !dfs(mask | (1<<i), remaining - i))
            return memo[mask] = 1;
    }
    return memo[mask] = 0;
};

// ─────────────────────────────────────────────────────────────────
// 9. OVERFLOW / FEASIBILITY GUARDS
// ─────────────────────────────────────────────────────────────────
// n <= 20        → int, 1 << n is safe
// n in 21..30    → int is still numerically safe, but check total memory
// n >= 31        → MUST use 1LL << n or reject as infeasible
// O(2^n · n)     → n up to ~20-22
// O(2^n · n^2)   → n up to ~15-18
// O(4^n)         → n up to ~8-10
// O(3^n)         → n up to ~18-20
```

---

## SECTION 21 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Shortest Superstring and Max Students are the two that require the most careful state design. Budget 45 minutes each for those two.

2. **Practice deriving state fields from `popcount`:** after finishing Min XOR Sum (LC1879), rewrite it explicitly tracking `i` as a SEPARATE dimension (`dp[mask][i]`) and confirm you get the identical answer with strictly more memory — this cements WHY the `popcount` derivation is valid, not just that it works.

3. **Held-Karp → Shortest Superstring bridge:** solve LC943 immediately after mastering Held-Karp (Section 6). Both are Hamiltonian-path-family DPs; the only real differences are (a) path vs. cycle, and (b) the "distance" function (string-overlap savings vs. literal distances). Doing them back-to-back cements the TSP reduction pattern.

4. **Preview Pattern 15 (Backtracking) if you haven't done it yet:** Beautiful Arrangement (LC526) and Partition to K Equal Sum Subsets (LC698) are both fundamentally backtracking problems that this pattern reframes with memoization — understanding the RAW backtracking version first makes the bitmask optimization click faster.

5. **The submask-enumeration challenge:** after finishing this pattern, implement Parallel Courses II (LC1494) from scratch, paying close attention to how "eligible courses this semester" is computed as `availableAndReady = (~taken) & prereqsSatisfied`, and how you enumerate submasks of THAT restricted set (not the full universe) to try every valid "up to k courses this semester" batch. This is the hardest submask-enumeration problem in the set.

6. **Cross-check with Pattern 16 (Game Theory DP):** after Can I Win, revisit any array-indexed game-theory DP you've already solved (Stone Game, Predict the Winner) and rewrite its state as a bitmask if the "removed/used" elements aren't contiguous — this generalizes the minimax recursion beyond interval-shaped games.

---

## SECTION 22 — SIGN-OFF CRITERIA

### Tier 1 — Bitmask enumeration mastered
You solve Subsets (LC78) and can recite the mask ↔ subset bijection without hesitation. You solve Partition to K Equal Sum Subsets (LC698) and can explain the `sum % target` trick that avoids an explicit group-index dimension.

### Tier 2 — Bitmask + extra dimension understood
You solve Shortest Path Visiting All Nodes (LC847) using `(node, mask)` BFS state, correctly seeded with multi-source starts. You solve Minimum XOR Sum (LC1879) and can explain why `popcount(mask)` safely replaces an explicit index dimension.

### Tier 3 — Held-Karp and its variants solid
You solve Held-Karp TSP from scratch, correctly using `dp[1][0]=0` as the base case (not `dp[0][0]`). You solve Maximum Students Taking Exam (LC1349) using row-bitmask DP with both self-adjacency and cross-row diagonal validity checks. You solve Find the Shortest Superstring (LC943) and can explain why it needs a Hamiltonian PATH (not cycle) formulation.

### Tier 4 — Hard variants complete
You solve Number of Ways to Wear Different Hats (LC1434) and can justify WHY hats go in the outer loop and people go in the mask. You solve Maximize Score After N Operations (LC1799) and correctly skip odd-popcount masks. You solve Can I Win (LC464) using memoization keyed purely on `mask`, and can explain why `remaining` doesn't need to be part of the memo key. You solve Parallel Courses II (LC1494) using submask enumeration over eligible courses.

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 847 (Shortest Path Visiting All Nodes — the `(node, mask)` state design must be explained), the Held-Karp TSP template (dp[mask][i] semantics and base case), and LC 464 (Can I Win — the memoization-key reasoning must be explained). These three represent the three hardest conceptual challenges in this pattern.

---

*Pattern 24 Complete — Bitmask DP*
*Next: Pattern 25 — Digit DP (counting numbers with constraints, tight-bound state machines)*
