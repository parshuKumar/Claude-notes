# PATTERN 38 — HARD PROBLEM DECOMPOSITION
## Constraint Reading, Insight Extraction, and Meeting in the Middle — The Grandmaster Meta-Skill
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**This pattern is different from the other 37.**

Patterns 1–37 each gave you a recognizable shape: two pointers, a monotonic stack, a segment tree, a DP over subsets. You saw a problem, matched it to a shape, and executed. That matching skill is necessary but it caps out around 1900–2000 rated. Above that line, problems stop announcing their shape. A 2100-rated problem does not say "use a monotonic stack" anywhere in the statement — it says something like "you are given an array, find the number of ways to..." and the monotonic stack is buried three transformations deep, disguised as something else entirely.

**What actually separates a 1600 solver from a 2000+ solver is not more patterns memorized. It is the ability to decompose an unfamiliar problem into pieces that ARE patterns you know**, even when the problem is deliberately dressed up to hide the seams.

Here is the honest mental process of a strong competitive programmer or interview candidate facing a problem they have never seen before, in the order it actually happens — not the sanitized order you see in editorials:

1. **Read the constraints before the story.** The flavor text ("a robot is exploring a dungeon...") is noise. The constraints (`1 <= n <= 2*10^5`, `1 <= grid[i][j] <= 10^9`) are the actual specification. They tell you, almost mechanically, what time complexity is expected — and therefore what family of algorithms is even in play.

2. **Ask: is this ONE hard idea, or TWO medium ideas stapled together?** The overwhelming majority of "hard" problems on LeetCode and in Div1 C/D on Codeforces are not hard because of one impossible insight. They are hard because they require you to correctly compose two or three techniques you already know, each individually medium-difficulty, in the right order, with the right glue between them. Maximal Rectangle (Pattern 18) is DP-computed heights + monotonic-stack histogram. Trapping Rain Water II (this pattern) is priority-queue Dijkstra-style expansion + the boundary-shrinks-inward idea from 1D trapping rain water. Once you see the two known pieces, the "hard" problem becomes two medium problems you already know how to write.

3. **Look for the class of insight, not the specific trick.** There are maybe 15–20 *classes* of "aha" insight that recur across thousands of hard problems (Section 3 catalogs them). "Count subarrays with property ≤ K by subtracting two 'at most' counts." "Binary search on the answer when the question is minimize-the-maximum." "Two heaps to maintain a running median." When you get stuck, you are not searching for a needle in an infinite haystack — you are pattern-matching against this much smaller catalog of insight *classes*, then adapting the specific class to the specific problem.

4. **If nothing clicks in 5–10 minutes, change your frame, not your effort.** Grinding harder on the same mental model rarely works. Reversing the problem (Dungeon Game direction reversal), transforming the input (Russian Doll Envelopes → LIS), or binary-searching on the answer are all *frame changes* — they don't require new information, just looking at the same problem through a different lens.

5. **Never write code before you can state the invariant in one sentence.** "dp[i][j] = minimum HP needed entering cell (i,j) to survive to the end" is one sentence. If you cannot produce that sentence, you do not yet understand the problem well enough to code it, no matter how hard 2000-rated the problem is. This is the single biggest tell that separates candidates who "eventually get there by trial and error" from candidates who solve it cleanly in 20 minutes.

**The goal of this capstone pattern** is not to teach you three more algorithms. Patterns 1–37 already gave you the algorithmic vocabulary. This pattern teaches you the **decomposition process** — the thing you do in the first five minutes, before you write a line of code, that determines whether the next thirty minutes are productive or wasted.

---

## SECTION 2 — THE DECOMPOSITION TOOLKIT

Seven concrete moves. Apply them roughly in this order when facing an unfamiliar hard problem. Most problems only need two or three of these before the structure becomes visible.

### Move 1 — Read constraints first to infer target complexity

Constraints are a coded message from the problem setter telling you the intended algorithm. Internalize this table so it becomes reflexive:

| Constraint on n (or value range) | Target complexity | What it signals |
|---|---|---|
| `n <= 10–12` | `O(2^n · n)` or `O(n!)` | brute force over permutations, bitmask DP over full state |
| `n <= 20–22` | `O(2^n)` or `O(2^n · n)` | bitmask DP (Hamiltonian path / TSP / assignment) |
| `n <= 40` | `O(2^(n/2))` | **meeting in the middle** (Section 5) — 2^n is too slow, 2^(n/2) is fine |
| `n <= 100–500` | `O(n^3)` or `O(n^4)` | triple/quadruple nested DP, Floyd–Warshall, interval DP |
| `n <= 2,000–5,000` | `O(n^2)` or `O(n^2 log n)` | pairwise DP, two-pointer-squared, naive segment tree build |
| `n <= 10^5` | `O(n log n)` | sort + two pointers, heap, binary search, segment/Fenwick tree |
| `n <= 10^6–10^7` | `O(n)` or `O(n log n)` | linear scan, sieve, two pointers, monotonic stack/deque |
| value range `<= 10^9` | `O(log(range))` | **binary search on the answer** |
| value range `<= 10^18` | `O(log(range))` or `O(1)` | binary search, matrix exponentiation, closed-form math |

The discipline: before you think about *how* to solve a problem, compute what complexity you're being asked to hit. If `n <= 2*10^5` and you're reaching for an `O(n^2)` idea, stop — that idea is wrong regardless of how correct its logic seems, because it cannot pass within the time limit. This single habit eliminates entire branches of wasted exploration.

### Move 2 — Identify the sub-structure: "this is X combined with Y"

Hard problems are rarely atomic. Train yourself to say out loud: *"this problem is [known technique A] applied to [known technique B]'s output."*

- **Maximal Rectangle (LC 85)** = per-row height accumulation (DP-flavored) **+** Largest Rectangle in Histogram (monotonic stack, Pattern 04).
- **Trapping Rain Water II (LC 407)** = the "boundary can only be limited by its weakest point" idea from 1D Trapping Rain Water **+** Dijkstra-style priority-queue frontier expansion (Pattern on graphs).
- **Sliding Window Median** = sliding window (Pattern 03) **+** two-heap running median (Pattern catalog entry in Section 3).
- **Word Ladder II** = BFS for shortest distance **+** DFS/backtracking to reconstruct all shortest paths.
- **Course Schedule IV** = topological sort **+** reachability propagation (transitive closure via DP over the topological order).

Once you can name the two known pieces, you write two pieces of code you already know how to write, connected by one new piece of glue logic. The glue logic is almost always the actual "hard" part — and it is usually much smaller than either piece alone.

### Move 3 — Reduce to a known problem (transform the input)

Sometimes the fastest path is not to invent new logic but to **transform the input so that a problem you already know how to solve applies directly.**

- **Russian Doll Envelopes (LC 354)**: sort envelopes by width ascending, and by height **descending** when widths tie (the descending tiebreak prevents same-width envelopes from nesting into each other in the LIS pass). Then the answer is exactly **Longest Increasing Subsequence** on the height sequence, solved with patience sorting in `O(n log n)`.
- **Longest Increasing Subsequence itself**, via patience sorting, is a reduction: maintain `tails[k]` = smallest possible tail of an increasing subsequence of length `k+1`; each new element does a binary search into `tails` and replaces (or extends). LIS becomes "binary search insertion into a maintained array," a problem you already know.
- **Longest Common Subsequence of a permutation with itself sorted** reduces to LIS in `O(n log n)` instead of `O(n^2)` LCS DP, because for permutations, LCS(A, sorted(A)) = LIS(position-mapped A).

The transformation move is powerful precisely because it lets you **borrow** an `O(n log n)` algorithm you already trust, instead of inventing a new `O(n log n)` algorithm from scratch under time pressure.

```cpp
// Patience-sorting LIS — the reduction target for Russian Doll Envelopes,
// and the O(n log n) tool you "borrow" whenever a problem reduces to LIS.
int lengthOfLIS(vector<int>& nums) {
    vector<int> tails;   // tails[k] = smallest tail of an increasing subsequence of length k+1
    for (int x : nums) {
        auto it = lower_bound(tails.begin(), tails.end(), x);
        if (it == tails.end()) tails.push_back(x);
        else *it = x;
    }
    return tails.size();
}

// Russian Doll Envelopes reduction: sort by width ascending, height
// DESCENDING on ties (so same-width envelopes can never nest into
// each other during the LIS pass), then LIS on the height sequence.
int maxEnvelopes(vector<vector<int>>& envelopes) {
    sort(envelopes.begin(), envelopes.end(), [](auto& a, auto& b) {
        if (a[0] != b[0]) return a[0] < b[0];
        return a[1] > b[1];   // descending height on width ties
    });
    vector<int> heights;
    for (auto& e : envelopes) heights.push_back(e[1]);
    return lengthOfLIS(heights);
}
```

### Move 4 — Reverse the problem / think backwards

If the forward direction produces a state that depends on information you don't have yet (typically: "the best choice here depends on what happens *later*"), try defining the DP or the traversal in the opposite direction.

- **Dungeon Game (LC 174, Pattern 18)**: forward DP fails because "minimum HP needed to reach (i,j)" depends on the path taken *after* (i,j) — a huge health pack later can retroactively reduce the HP you needed earlier. Defining `dp[i][j]` = minimum HP needed **entering** (i,j) to survive everything from (i,j) onward, computed backward from the destination, fixes this instantly.
- **Task Scheduling with deadlines**: processing greedily from the *latest* deadline backward, always picking the highest-value task that still fits, is often cleaner than forward greedy with a priority queue of "regret."
- **Deleting to make an array good**: sometimes it's easier to ask "what is the maximum I can KEEP" (find the complement) rather than "what is the minimum I must remove."

The tell that you need this move: you keep writing a transition and finding that it needs `dp[i+1]` or `dp[j+1]` (something you haven't computed yet) to determine `dp[i]`. That's the signal to flip your iteration direction.

### Move 5 — Binary search on the answer

Whenever the problem says (in disguise or explicitly) **"minimize the maximum"** or **"maximize the minimum,"** and you notice the feasibility of a candidate answer is *monotonic* (if X is achievable, every value greater than X is also achievable, or vice versa), binary search on the answer directly.

Template:

```cpp
// Minimize the maximum: find smallest X such that check(X) is true
long long lo = minPossible, hi = maxPossible;
while (lo < hi) {
    long long mid = lo + (hi - lo) / 2;
    if (check(mid)) hi = mid;      // mid is achievable — try smaller
    else lo = mid + 1;             // mid too small — need larger
}
return lo;
```

`check(mid)` is usually an `O(n)` or `O(n log n)` greedy feasibility test — the "medium" half of the problem. Recognizing that you can binary search converts a search over an intractably large space of *answers* into `O(log(range))` calls to a simple, well-understood feasibility check. This single move unlocks: **Split Array Largest Sum**, **Koko Eating Bananas**, **Capacity To Ship Packages Within D Days**, **Aggressive Cows** (classic Codeforces/SPOJ), **Minimize Max Distance to Gas Station**, **Median of Two Sorted Arrays** (binary search on the *partition*, Section 4).

### Move 6 — Exploit monotonicity, symmetry, invariants

- **Monotonicity** unlocks two pointers (both pointers move only forward, never backward, because the search space shrinks monotonically as one pointer advances) and monotonic stacks/deques (elements are popped once and never revisited).
- **Symmetry** halves state spaces. In Cherry Pickup (Pattern 18), agent 1 at row `r1` and agent 2 at row `r2` is equivalent to swapping the two agents — so enforcing `r2 >= r1` cuts the state space in half without losing correctness.
- **Invariants** let you maintain a running quantity incrementally instead of recomputing from scratch. A running sum, a running max-heap top, a running count of distinct elements in a window — each of these is an invariant maintained in `O(1)` or `O(log n)` per step instead of recomputed in `O(n)`.

The question to ask explicitly: *"as I move one pointer/step forward, does some quantity only increase or only decrease? Can I avoid recomputing it from scratch?"*

### Move 7 — Change of variables / state

Rewriting the problem in terms of a different but equivalent quantity often turns an intractable problem into a textbook one.

- **Prefix sums** turn "sum of subarray [l, r]" into `prefix[r+1] - prefix[l]`, an `O(1)` lookup — the foundation of Pattern 06 and half the problems in Section 3's catalog.
- **Difference of two quantities**: "exactly K distinct elements" = `atMost(K) - atMost(K-1)` (Subarrays with K Different Integers, LC 992). Counting "exactly" is often much harder than counting "at most," so decompose exactly into a subtraction of two at-most computations.
- **XOR prefix**: subarray XOR equals `prefixXor[r+1] ^ prefixXor[l]`, mirroring the sum-prefix trick but for XOR — this reframes "find subarray with XOR = K" as "find two prefix XORs that differ by K," solvable with a hashmap in `O(n)`.
- **Maximum Sum Circular Subarray**: total sum minus the *minimum* subarray sum, when the minimum subarray is strictly interior, gives the best circular (wraparound) answer — reframing "wraparound max" as "total minus interior min" turns a fiddly circular-array problem into two applications of Kadane's algorithm.

---

## SECTION 3 — THE "AHA INSIGHT" CATALOG

This is the compressed reference you build up over hundreds of problems. When stuck, scan this table before scanning your memory at random — it is a much smaller, much more targeted search space.

| Insight class | One-line description | Problems it unlocks |
|---|---|---|
| At-most(K) subtraction | "exactly K" = atMost(K) − atMost(K−1) | Subarrays with K Distinct Integers (992), Fruit Into Baskets (904), Count Subarrays With Median K |
| Two heaps for median | max-heap for lower half, min-heap for upper half, rebalance to keep sizes within 1 | Find Median from Data Stream (295), Sliding Window Median (480), IPO (502) |
| Monotonic stack for next-greater/smaller | maintain indices in increasing/decreasing value order, pop when violated | Daily Temperatures (739), Largest Rectangle in Histogram (84), Trapping Rain Water (42), Maximal Rectangle (85) |
| Meeting in the middle | split n≤40 items into two halves, brute force each half, combine with sort+binary-search or two-pointer | Closest Subsequence Sum (1755), Partition Array Into Two Arrays to Minimize Sum Difference (1755-adjacent), 4-SUM count variants |
| 0-1 BFS with deque | edges of weight 0 pushed to front, weight 1 pushed to back — gives shortest path without a heap | Minimum Cost to Make at Least One Valid Path (1368), Shortest Path with Alternating Colors |
| Sort then two-pointer | sorting kills one dimension of freedom, two pointers exploit the resulting monotonicity | 3Sum (15), Container With Most Water (11), Boats to Save People (881), Two Sum II |
| Binary search on the answer | monotonic feasibility ⇒ search the answer space, not the solution space | Split Array Largest Sum (410), Koko Eating Bananas (875), Capacity To Ship Packages (1011) |
| Binary search on a partition | find the split point where left-max ≤ right-min across two sorted structures | Median of Two Sorted Arrays (4), Kth Smallest in Two Sorted Arrays |
| Prefix-sum-as-hashmap-key | store seen prefix sums in a map, look up `prefix - K` | Subarray Sum Equals K (560), Continuous Subarray Sum (523), Subarray XOR queries |
| Reverse DP / reverse traversal | answer at a state depends on future states, not past — invert the direction | Dungeon Game (174), some interval scheduling with deadlines |
| Difference array for range updates | `diff[l] += v; diff[r+1] -= v;` then prefix-sum once at the end | Car Pooling (1094), Range Addition (370), Corporate Flight Bookings |
| Height array + histogram | reduce a 2D "largest rectangle of 1s" to N applications of a 1D histogram problem | Maximal Rectangle (85) |
| Kadane on a transformed array | total − min(interior subarray) captures the circular/complement case | Maximum Sum Circular Subarray (918) |
| Coordinate compression + Fenwick/BIT | compress large/sparse values to ranks, then count inversions or range sums | Count of Smaller Numbers After Self (315), Reverse Pairs (493) |
| Greedy + priority queue (regret-based) | take everything, then "undo" the worst choice if a constraint is violated | Task Scheduler variants, IPO (502), Minimum Cost to Hire K Workers (857) |
| Priority-queue frontier expansion (Dijkstra-flavored) | always expand the globally weakest/cheapest boundary cell next | Trapping Rain Water II (407), Swim in Rising Water (778), Path With Minimum Effort (1631) |

**How to use this table under time pressure:** when you're stuck, don't re-read the problem for the fifth time — scan this table's left column and ask "does any of these descriptions match a sub-piece of my problem?" More often than not, one row will click, and that click is 80% of the remaining work.

**Two of the denser rows, in code, to make the abstraction concrete:**

```cpp
// At-most(K) subtraction — LC 992, Subarrays with K Distinct Integers.
// "Exactly K distinct" is hard to count directly with a sliding window,
// because shrinking the window on a distinct-count violation is ambiguous
// (which end shrinks, and by how much, isn't monotonic in an obvious way).
// "At most K distinct" IS a clean sliding window: exactly K = atMost(K) - atMost(K-1).
int atMostKDistinct(vector<int>& nums, int k) {
    unordered_map<int,int> count;
    int left = 0, result = 0;
    for (int right = 0; right < nums.size(); right++) {
        count[nums[right]]++;
        while (count.size() > k) {
            count[nums[left]]--;
            if (count[nums[left]] == 0) count.erase(nums[left]);
            left++;
        }
        result += right - left + 1;   // every window ending at `right` with <=k distinct
    }
    return result;
}

int subarraysWithKDistinct(vector<int>& nums, int k) {
    return atMostKDistinct(nums, k) - atMostKDistinct(nums, k - 1);
}
```

```cpp
// Two heaps for a running median — LC 295, Find Median from Data Stream.
// Maintain the invariant: maxHeap (lower half) size == minHeap (upper half)
// size, or exactly one more — so the median is always at a heap top.
priority_queue<int> lo;                              // max-heap: lower half
priority_queue<int, vector<int>, greater<int>> hi;   // min-heap: upper half

void addNum(int num) {
    lo.push(num);
    hi.push(lo.top()); lo.pop();          // push the max of lower half up to hi
    if (hi.size() > lo.size()) {
        lo.push(hi.top()); hi.pop();      // rebalance back if hi grew too large
    }
}

double findMedian() {
    if (lo.size() > hi.size()) return lo.top();
    return (lo.top() + hi.top()) / 2.0;
}
```

---

## SECTION 4 — THREE FULLY WORKED HARD-PROBLEM DECOMPOSITIONS

These three problems are chosen because each exemplifies a *different* toolkit move from Section 2: priority-queue frontier expansion, binary search on a partition, and binary search on the answer. Work through the constraint reading and false starts as carefully as the final code — that's where the actual decomposition skill lives.

---

### 4A — Trapping Rain Water II (LC 407)

**Problem statement:** Given an `m x n` matrix of non-negative integers representing the height of each unit cell in a 2D elevation map, compute the volume of water it can trap after raining.

**Constraint reading:** `1 <= m, n <= 200`, so `m*n <= 40,000`. Heights up to `2*10^4`. A total cell count of 40,000 means we can afford `O(mn log(mn))` — roughly 40,000 × 16 ≈ 640,000 operations, trivially fast. This rules out anything cubic in `mn`, but a heap-based approach is comfortably in budget.

**False start 1 — direct extension of 1D two-pointer:** In 1D Trapping Rain Water, you track `leftMax` and `rightMax` and move the pointer on the side with the smaller max inward. In 2D, "left" and "right" don't exist — every cell has up to 4 neighbors, and the boundary is now the entire perimeter of the grid, not two endpoints. Two pointers don't generalize to a 2D frontier with more than 2 directions of travel.

**False start 2 — row-by-row then column-by-column 1D trapping:** Applying 1D trapping water independently per row, then independently per column, and taking some combination, fails because water trapped at a cell depends on the *minimum* enclosing boundary in **all directions simultaneously**, not the max of two independent 1D answers. A cell can be blocked from leaking in every direction except one, and that one leak determines the water level — a purely row-wise or column-wise view cannot see this.

**The decomposition:** The correct generalization of "the water level at any point is bounded by the weakest link of its enclosing wall" is: **process cells in increasing order of the current known boundary height, expanding a frontier inward from the perimeter, exactly like Dijkstra's algorithm expands the closest unvisited node first.**

- Push every **perimeter cell** into a min-heap keyed by height — the perimeter is the initial "wall," since water cannot be held outside the grid.
- Repeatedly pop the cell with the **smallest wall height** — this is the weakest point of the current known boundary, so it is the one that determines how much water its unvisited interior neighbors can hold.
- For each unvisited neighbor: water trapped there = `max(0, currentWallHeight - neighborHeight)`. Then push the neighbor into the heap with height `max(currentWallHeight, neighborHeight)` — the new wall height propagates inward, "flooding up" if the interior is even lower, or staying at the neighbor's own height if it's taller than the current wall.

This is *precisely* the "priority-queue frontier expansion" row from Section 3's catalog, combined with the 1D trapping-rain-water idea that **water level = min of the enclosing walls, not the max**.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 407 — Trapping Rain Water II
// State: process cells by increasing known-boundary height, like Dijkstra.
// Every perimeter cell starts as a "wall" of its own height.
// Popping the globally smallest wall guarantees it is the bottleneck
// for any unvisited interior cell it borders.
// ─────────────────────────────────────────────────────────────────

int trapRainWater(vector<vector<int>>& heightMap) {
    int m = heightMap.size(), n = heightMap[0].size();
    if (m < 3 || n < 3) return 0;  // no interior cell can hold water

    vector<vector<bool>> visited(m, vector<bool>(n, false));
    // min-heap of (height, row, col)
    priority_queue<tuple<int,int,int>,
                    vector<tuple<int,int,int>>,
                    greater<tuple<int,int,int>>> pq;

    // Seed the heap with the entire perimeter — the initial wall.
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (i == 0 || i == m - 1 || j == 0 || j == n - 1) {
                pq.push({heightMap[i][j], i, j});
                visited[i][j] = true;
            }
        }
    }

    int water = 0;
    int dx[4] = {0, 0, 1, -1}, dy[4] = {1, -1, 0, 0};

    while (!pq.empty()) {
        auto [h, x, y] = pq.top();
        pq.pop();

        for (int d = 0; d < 4; d++) {
            int nx = x + dx[d], ny = y + dy[d];
            if (nx < 0 || nx >= m || ny < 0 || ny >= n || visited[nx][ny]) continue;
            visited[nx][ny] = true;

            // Water trapped at (nx,ny) is bounded by the CURRENT wall height h,
            // which is guaranteed to be the minimum enclosing wall so far
            // because we always pop the smallest height first.
            water += max(0, h - heightMap[nx][ny]);

            // Propagate the wall inward: it can only get taller
            // (or stay the same, if the interior cell is naturally taller).
            pq.push({max(h, heightMap[nx][ny]), nx, ny});
        }
    }

    return water;
}
```

**Why "always pop the minimum" is the crux of correctness:** Suppose we instead popped an arbitrary boundary cell. We might expand a tall boundary cell first and record water for its low interior neighbor using the tall cell's height — but that low neighbor might actually be exposed to a *lower* wall through a different, not-yet-processed path, meaning the true trapped water is less than we recorded. By always popping the globally smallest wall height, we guarantee every unvisited cell we touch is being bounded by the tightest constraint currently known — exactly the same greedy-correctness argument that makes Dijkstra's algorithm correct (always finalize the closest unvisited node).

**Complexity:** `O(mn log(mn))` — every cell is pushed and popped from the heap exactly once, each heap operation is `O(log(mn))`.

---

### 4B — Median of Two Sorted Arrays (LC 4)

**Problem statement:** Given two sorted arrays `nums1` (size `m`) and `nums2` (size `n`), find the median of the two arrays combined, in `O(log(m+n))` time.

**Constraint reading:** The `O(log(m+n))` requirement is stated explicitly — this is the rare problem that tells you the target complexity directly instead of making you infer it from `m, n <= 1000`. Merging the two arrays (`O(m+n)`) is a correct but too-slow solution; the log requirement is a hard signal to **binary search on something**.

**False start 1 — binary search for the median value directly:** You might try to binary search over the *value* of the median (say, between the min and max of both arrays) and count how many elements are ≤ mid. This can work but is fiddly with duplicate values and doesn't map cleanly onto "the k-th smallest element," which is really what a median is.

**False start 2 — merge-like two-pointer walk to the middle:** Walking pointers through both arrays simultaneously to the `(m+n)/2`-th element is `O(m+n)`, not `O(log(m+n))`. It's a correct, simple, interview-safe fallback — but it fails the stated complexity requirement, and more importantly it fails to generalize the insight needed for the follow-up ("what if arrays are huge and this needs to be sub-linear?").

**The decomposition:** The median splits the *combined, sorted* array into a left half and a right half of (near-)equal size, where **every element in the left half is ≤ every element in the right half**. We don't need to actually merge anything — we just need to find, for each array, how many elements it contributes to the "left half," such that the totals add up correctly and the boundary condition holds. This reframes the problem as **binary search on a partition index**, not binary search on a value — a distinct and less obvious application of Move 5 (binary search) than "minimize the maximum."

- Always binary search on the **smaller** array (call it `A`, size `m`) — this bounds the binary search range to `O(log m)` and guarantees the partition point `j` in the other array `B` is always in valid range, since `j = half − i` and `i` never exceeds `m`.
- For a candidate partition `i` in `A` (meaning `A[0..i-1]` goes left, `A[i..]` goes right), the required partition in `B` is `j = (m+n+1)/2 - i`, so that the left half has exactly `⌈(m+n)/2⌉` elements total.
- Check the boundary condition: `A[i-1] <= B[j]` AND `B[j-1] <= A[i]` (using `-∞`/`+∞` sentinels at the edges). If both hold, the partition is correct — compute the median directly from the four boundary values. If `A[i-1] > B[j]`, `i` is too large — search smaller `i`. Otherwise, `i` is too small — search larger `i`.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 4 — Median of Two Sorted Arrays
// Binary search on the PARTITION INDEX of the smaller array, not on
// a value. We never merge — we only need the four numbers immediately
// straddling the partition to determine the median.
// ─────────────────────────────────────────────────────────────────

double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
    // Always binary search the smaller array — bounds the search to O(log(min(m,n)))
    // and guarantees j = half - i stays within [0, n].
    if (nums1.size() > nums2.size()) return findMedianSortedArrays(nums2, nums1);

    int m = nums1.size(), n = nums2.size();
    int lo = 0, hi = m;
    int half = (m + n + 1) / 2;  // size of the LEFT half (handles odd total correctly)

    while (lo <= hi) {
        int i = lo + (hi - lo) / 2;   // elements of nums1 in the left half
        int j = half - i;             // elements of nums2 in the left half

        // Sentinels for out-of-range partitions
        int Aleft  = (i == 0) ? INT_MIN : nums1[i - 1];
        int Aright = (i == m) ? INT_MAX : nums1[i];
        int Bleft  = (j == 0) ? INT_MIN : nums2[j - 1];
        int Bright = (j == n) ? INT_MAX : nums2[j];

        if (Aleft <= Bright && Bleft <= Aright) {
            // Correct partition found.
            if ((m + n) % 2 == 1) {
                return max(Aleft, Bleft);  // odd total: median is the max of the left half
            }
            return (max(Aleft, Bleft) + min(Aright, Bright)) / 2.0;
        } else if (Aleft > Bright) {
            hi = i - 1;   // too many elements of nums1 in the left half — shrink i
        } else {
            lo = i + 1;   // too few elements of nums1 in the left half — grow i
        }
    }

    throw invalid_argument("input arrays must be sorted");  // unreachable if inputs valid
}
```

**Trace for nums1 = [1,3], nums2 = [2]:**
```
m=2, n=1. half = (2+1+1)/2 = 2.
lo=0, hi=2.

i = 1 (mid of [0,2]): j = 2 - 1 = 1
  Aleft = nums1[0] = 1, Aright = nums1[1] = 3
  Bleft = nums2[0] = 2, Bright = INT_MAX (j == n)
  Check: Aleft(1) <= Bright(INF) ✓   Bleft(2) <= Aright(3) ✓
  Valid partition!

Total length = 3 (odd) → median = max(Aleft, Bleft) = max(1, 2) = 2  ✓
(Merged array is [1,2,3], median = 2 — correct.)
```

**Why binary search on the smaller array specifically:** if we searched on the larger array `B`, the corresponding `i = half - j` in the smaller array `A` could go negative or exceed `m` for candidate `j` values, requiring extra bounds-checking. Searching the smaller array guarantees `j` always stays in `[0, n]` by construction, since `half <= (m+n+1)/2 <= m` is not guaranteed — but by choosing `A` as the smaller array, `i` ranges over `[0, m]`, and `j = half - i` is provably always within `[0, n]` because `half <= m + n` and `half - m <= n` (since `half <= (m+n+1)/2 <= n + (n+1)/2 <= n` when `m <= n`... more directly: this is exactly why the algorithm swaps to guarantee `m <= n` up front). This is a small but important correctness detail that a Grandmaster states explicitly rather than hand-waving.

**Complexity:** `O(log(min(m, n)))`.

---

### 4C — Split Array Largest Sum (LC 410)

**Problem statement:** Given an array `nums` and an integer `k`, split `nums` into `k` non-empty contiguous subarrays to **minimize the largest sum** among the `k` subarrays.

**Constraint reading:** `1 <= nums.length <= 1000`, `nums[i]` up to `10^6`, `1 <= k <= min(50, nums.length)`. A DP over "first i elements split into j pieces" is `O(nk)` states with an `O(n)` (or `O(log n)` with a monotonic-deque/convex-hull optimization) transition — feasible but fiddly. The phrase **"minimize the largest sum"** is the textbook signature of Move 5: minimize-the-maximum, monotonic feasibility, binary search on the answer.

**False start 1 — DP over (index, pieces used):** `dp[i][j]` = minimum possible "largest sum" when the first `i` elements are split into `j` pieces. This is correct: `dp[i][j] = min over p < i of max(dp[p][j-1], sum(p+1..i))`. It works in `O(n^2 k)` naively, which at `n=1000, k=50` is `5*10^7` — borderline but passable. It is NOT wrong, but it is more code, more edge cases (prefix sums, nested loops with three indices), and strictly dominated by the binary search approach in both simplicity and asymptotic complexity.

**False start 2 — greedy without verification:** Greedily filling each of the `k` pieces as full as possible without a target cap produces an arbitrary, usually unbalanced split — there is no greedy rule for "which split minimizes the max" without first fixing a candidate max and checking feasibility against it.

**The decomposition:** The key monotonicity: if a candidate maximum sum `X` allows splitting `nums` into `<= k` pieces (using the greedy rule "keep adding to the current piece until adding the next element would exceed `X`, then start a new piece"), then **every value greater than X is also feasible** (a looser cap can only merge pieces, never require more of them). This monotonic feasibility is exactly the precondition for Move 5. So:

- Binary search `X` in `[max(nums), sum(nums)]` — `X` can never be smaller than the largest single element (that element alone forms a piece with sum `X`), and never needs to be larger than the sum of everything (`k=1` case).
- `check(X)`: greedily walk the array, accumulating a running sum into the current piece; whenever adding the next element would exceed `X`, close the current piece and start a new one with that element. Count pieces used; feasible if `pieces <= k`.
- This greedy `check` is provably optimal for "minimum pieces needed to keep every piece sum `<= X`" — greedily cramming as much as possible into each piece before closing it never does worse than closing a piece early, because closing early only ever *increases* the pieces needed for the remainder.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 410 — Split Array Largest Sum
// Binary search on the ANSWER (the largest-sum cap X).
// check(X) = can we split nums into <= k pieces so every piece sum <= X?
// Monotonic: if X works, X+1 also works (never need MORE pieces for a looser cap).
// ─────────────────────────────────────────────────────────────────

bool canSplit(const vector<int>& nums, int k, long long maxSum) {
    int pieces = 1;
    long long cur = 0;
    for (int x : nums) {
        if (cur + x > maxSum) {
            pieces++;
            cur = x;
            if (pieces > k) return false;   // early exit — already infeasible
        } else {
            cur += x;
        }
    }
    return true;
}

int splitArray(vector<int>& nums, int k) {
    long long lo = *max_element(nums.begin(), nums.end());  // cap can't be < largest element
    long long hi = accumulate(nums.begin(), nums.end(), 0LL); // cap never needs to exceed total sum

    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (canSplit(nums, k, mid)) {
            hi = mid;       // mid is feasible — try to do better (search smaller)
        } else {
            lo = mid + 1;   // mid too tight — need a larger cap
        }
    }
    return (int)lo;
}
```

**Trace for nums = [7,2,5,10,8], k = 2:**
```
lo = max(nums) = 10, hi = sum(nums) = 32

mid = 21: check(21): 7+2+5=14, +10=24>21 → close piece1={7,2,5}=14, piece2 starts=10
          10+8=18<=21 → piece2={10,8}=18. pieces=2. feasible (2<=2). hi=21.

mid = (10+21)/2 = 15: check(15): 7+2=9,+5=14,+10=24>15 → close piece1={7,2,5}=14, piece2=10
          10+8=18>15 → close piece2={10}=10, piece3=8. pieces=3 > k=2. infeasible. lo=16.

mid = (16+21)/2 = 18: check(18): 7+2+5=14,+10=24>18 → piece1={7,2,5}=14, piece2=10
          10+8=18<=18 → piece2={10,8}=18. pieces=2. feasible. hi=18.

mid = (16+18)/2 = 17: check(17): 7+2+5=14,+10=24>17 → piece1=14, piece2=10
          10+8=18>17 → close piece2={10}=10, piece3=8. pieces=3>2. infeasible. lo=17.

lo == hi == 18? lo=17, hi=18. mid=(17+18)/2=17 (already checked, infeasible) lo=18.
lo == hi == 18. Return 18.

Verify: split [7,2,5]=14 and [10,8]=18. max=18. Can we do better? [7,2,5,10]=24 too big for one piece
alone at X<24, and any other 2-way split has a piece >= 18. Answer 18 is correct.
```

**Complexity:** `check` is `O(n)`; binary search does `O(log(sum(nums) - max(nums)))` iterations. Total: `O(n log(sum))`, dramatically better than the `O(n^2 k)` DP, and far simpler to implement correctly under interview time pressure.

---

## SECTION 5 — MEETING IN THE MIDDLE (n ≤ 40 SUBSET PROBLEMS)

**When it applies:** the problem wants something over **all subsets** of an array (subset sums, subset XORs, subset products under a constraint), and `n` is too large for `O(2^n)` (say `n > 22`) but small enough that `O(2^(n/2))` is comfortably fast — the classic signal is `n <= 40`. `2^40 ≈ 10^12` is far too slow, but `2^20 ≈ 10^6` is trivial.

**The decomposition:** Split the `n` elements into two halves of size `~n/2` each. Brute-force **all** `2^(n/2)` subset sums (or XORs, or whatever aggregate the problem needs) of each half independently — this is fast because `2^(n/2)` is small. Then **combine** the two halves' results — typically by sorting one half's results and binary-searching (or two-pointering) against the other half — to answer the actual question in `O(2^(n/2) log(2^(n/2)))`, without ever materializing the full `2^n` combinations.

This is not "cheating" the exponential barrier — it fundamentally respects it. `2^(n/2) · 2^(n/2) = 2^n`, so we haven't beaten the information-theoretic size of the subset space; we've simply avoided *generating* the full `2^n` set by generating its two square-root-sized factors and combining them algorithmically rather than enumerating their full cross product.

```cpp
// ─────────────────────────────────────────────────────────────────
// Meeting in the middle — Subset Sum Exists for n <= 40
// Question: does some subset of nums sum to exactly `target`?
// ─────────────────────────────────────────────────────────────────

bool subsetSumExists(vector<long long>& nums, long long target) {
    int n = nums.size();
    int half1 = n / 2, half2 = n - half1;

    // Enumerate all subset sums of each half — O(2^(n/2)) each.
    vector<long long> sumsA, sumsB;
    for (int mask = 0; mask < (1 << half1); mask++) {
        long long s = 0;
        for (int i = 0; i < half1; i++)
            if (mask & (1 << i)) s += nums[i];
        sumsA.push_back(s);
    }
    for (int mask = 0; mask < (1 << half2); mask++) {
        long long s = 0;
        for (int i = 0; i < half2; i++)
            if (mask & (1 << i)) s += nums[half1 + i];
        sumsB.push_back(s);
    }

    // Combine: for every sum a in A, does target - a exist in B?
    sort(sumsB.begin(), sumsB.end());
    for (long long a : sumsA) {
        long long need = target - a;
        if (binary_search(sumsB.begin(), sumsB.end(), need)) return true;
    }
    return false;
}
```

**Extension — closest subsequence sum (LC 1755):** instead of an exact match, you want the subset sum *closest to* a target. Same split, but combine with a **two-pointer sweep** over both sorted halves (start one pointer at the beginning of `sumsA`, the other at the end of `sumsB`, and walk them toward each other, tracking the closest total seen) instead of binary search — this covers the whole combined search space in `O(2^(n/2))` after the sort, without needing a separate binary search per element.

**Complexity:** `O(2^(n/2) · (n/2))` to generate each half's subset sums, `O(2^(n/2) log(2^(n/2)))` to sort, `O(2^(n/2) log(2^(n/2)))` to combine via binary search (or `O(2^(n/2))` via two pointers). At `n=40`: `2^20 ≈ 10^6`, comfortably fast, versus `2^40 ≈ 10^12` for brute force — the difference between "instant" and "does not finish in the age of the universe."

**Recognizing this pattern in disguise:** watch for `n <= 40`, `n <= 36`, or explicit phrasing like "array split into two halves" as strong constraint-reading signals (Move 1) that this technique — not bitmask DP, not brute force — is intended.

---

## SECTION 6 — HOW TO PRACTICE THIS SKILL DELIBERATELY

Decomposition is a skill trained by **honest struggle followed by disciplined post-mortem**, not by reading more editorials passively. A concrete regimen:

1. **Run virtual contests, not just untimed problem sets.** Pick a past Codeforces Div. 2 (or Div. 1+2 combined) round and do a *virtual participation* — same time limit, no internet, no editorial, submit as you go. Time pressure forces you to actually use the decomposition toolkit (Section 2) instead of slowly circling toward a solution with unlimited time. Untimed grinding trains recognition of problems you've seen before; timed virtual contests train the actual decomposition reflex.

2. **Upsolve everything you didn't finish, within 24–48 hours, editorial-free first.** After the contest, before reading any editorial, spend another honest attempt (30–60 minutes) on every problem you didn't solve. The goal is not speed here — it's giving your brain a second, less time-pressured pass to find the insight on its own. Most insights that "click the next day" were reachable during the contest too; the difference is stress and fatigue, and a second honest attempt trains you to reach them faster next time.

3. **Editorial discipline: read only after an honest attempt, and read for the insight, not the code (see Section 8).** Reading an editorial before attempting is the single most damaging habit for this skill — it replaces "can I decompose this" with "can I recognize this from having read it," which does not transfer to novel problems.

4. **Re-implement from scratch after reading the editorial — don't copy.** Close the editorial, and write the solution yourself from the one-sentence insight you extracted. If you can't, you didn't actually understand the insight; go back and re-read until you can close the tab and still produce correct code.

5. **Keep a personal insight log.** After every hard problem (solved or upsolved), add one line to a running log: `problem name → insight class (from Section 3's catalog) → one-sentence restatement in your own words`. Periodically re-scan this log. Over hundreds of entries, this log becomes a personalized, denser version of Section 3 — the single highest-leverage artifact you can build for pushing past 2000.

6. **Redo old hard problems blind, weeks later, no notes.** Spaced repetition applies to algorithms exactly as it applies to vocabulary. Pick 3–5 problems from a month ago that gave you real trouble, and resolve them cold. If the insight doesn't come back within 10–15 minutes, that's a signal the insight was never truly internalized — just recognized in the moment — and needs to go back into active rotation.

7. **Mix contest problems with interview-style problems deliberately.** Codeforces trains raw algorithmic decomposition under time pressure with adversarial test data. LeetCode Hard / interview-style problems train articulating the decomposition *out loud*, in real time, to another person, while writing clean production-quality code — a distinct and equally important skill for the 2000+ interview bar.

---

## SECTION 7 — THE "STUCK LADDER"

An ordered list of moves to try, in order, when you are completely stuck on a hard problem with no idea where to start. Each rung costs more time than the last — don't skip to the bottom prematurely, but don't grind indefinitely on a rung that isn't working either.

1. **Re-read the problem statement literally, word by word.** Check for a misread constraint, an edge case in the examples you glossed over, or a subtlety in "exactly" vs. "at most" vs. "at least." A surprising fraction of "I'm stuck" is actually "I solved a slightly different problem than the one asked."

2. **Re-read the constraints and recompute the target complexity (Move 1).** If your current mental approach doesn't match the target complexity, that mismatch alone tells you your current approach is the wrong branch to keep pursuing — stop refining it and change direction.

3. **Work a tiny concrete example by hand** (`n = 3` or `n = 4`). Write out the brute-force answer for that tiny case manually. Patterns that are invisible in the abstract often become obvious staring at 3–4 concrete numbers.

4. **State the brute force explicitly, and name where the redundant work is.** "Brute force is O(n^2) because for every pair (i,j) I recompute X from scratch — but X only changes by one element as j increases." Naming the redundant recomputation is frequently 90% of the way to the optimization (usually a prefix structure, a sliding window, or a monotonic stack).

5. **Ask: is this two known patterns glued together (Move 2)?** Explicitly try naming candidate pairs from your pattern vocabulary (Patterns 1–37) even if the fit feels forced. Forcing the naming exercise surfaces partial matches you'd otherwise dismiss.

6. **Ask: does reversing the problem help (Move 4)?** Try defining the DP/traversal in the opposite direction, or ask "what if I solve for the complement of what's asked" (minimum removals ↔ maximum kept, earliest ↔ latest).

7. **Ask: is this "minimize the max" or "maximize the min" in disguise (Move 5)?** Even if the problem doesn't use those exact words, check whether a candidate answer's feasibility is monotonic — if so, binary search on the answer.

8. **Ask: can I fix one variable and solve for the rest (two-pointer / meeting in the middle)?** If there are two interacting choices, try holding one fixed and asking what's optimal for the other — this often reveals a two-pointer or precompute-and-binary-search structure.

9. **Simplify: drop one constraint, solve the easier version, then add the constraint back.** If the problem has an extra twist (e.g., "at most K removals allowed"), solve the K=0 version first. The clean-version solution usually reveals exactly where the constraint needs to be threaded in.

10. **Look explicitly for an invariant or monotonic quantity you haven't used yet (Move 6).** Ask: "as I scan left to right, is there something that only grows or only shrinks?"

11. **Step away for at least 20–30 minutes.** Incubation is real — a documented cognitive phenomenon, not an excuse. Many insights surface specifically *because* you stopped consciously working the problem.

12. **As a last resort before the editorial, peek only at the problem's topic tag** (e.g., Codeforces' hidden tags, or a LeetCode "similar problems" list) — not the solution. A single word like "binary search" or "two pointers" is often enough to unstick you while still leaving the actual derivation to you.

13. **Read the editorial, but only for the ONE insight (Section 8) — then close it and re-derive everything else yourself.**

---

## SECTION 8 — READING EDITORIALS LIKE A GRANDMASTER

Most people read an editorial the wrong way: top to bottom, absorbing the code, and walking away having learned almost nothing transferable. A Grandmaster reads an editorial to extract exactly **one thing** — the core insight — and treats everything else as noise to be reconstructed independently.

**The discipline:**

1. **Read only the first paragraph or two — stop before the code appears.** Editorials are almost always structured as: restate the problem → state the key insight → derive the algorithm → give code. The key insight is nearly always contained in the first 20% of the text. Stop there.

2. **Extract the insight as a single sentence in your own words.** Not "they use a priority queue and BFS" (that's *what*, not *why*) — instead: "the water level at any interior cell is bounded by the weakest point of its enclosing wall, and processing walls from weakest to strongest guarantees correctness, exactly like Dijkstra's shortest-path proof." That sentence is portable — it will fire the next time you see *any* problem with a "weakest boundary determines the interior" structure, not just this exact problem.

3. **Close the editorial. Do not look at the code.** The code is the least valuable part of an editorial for skill-building — it's mechanical translation of the insight, and you already know how to write code. Copying it teaches syntax, not decomposition.

4. **Re-derive the full solution from your one-sentence insight, including the complexity analysis.** If you can't get from the sentence back to working code, the sentence wasn't precise enough — go back, re-read a little further, and refine it. This step is where the actual learning happens, not the reading.

5. **Only after your own implementation passes, glance at the official code** — purely to check for a cleaner formulation, a sharper edge-case handling trick, or an idiom you didn't know (e.g., using `INT_MIN/2` sentinels instead of special-casing boundaries). Never copy; only compare.

6. **File the insight into your personal log (Section 6, point 5), tagged with its Section 3 insight class.** This is what compounds over time — after 200–300 problems logged this way, you will start recognizing insight classes in brand-new problems within the first sixty seconds of reading them, which is functionally what "being rated 2100+" means.

**The trap to avoid:** reading an editorial and thinking "oh, that's obvious in hindsight" is *not* the same as having been able to produce it yourself. The hindsight bias is exactly why the re-derivation step (point 4) is non-negotiable — it's the only step that actually tests whether you internalized the transferable insight or just recognized a plausible-sounding explanation.

---

## SECTION 9 — COMMON DECOMPOSITION MISTAKES

These are mistakes made *before* any code is written — the approach itself is wrong, not the implementation. Each is drawn from the exact problems worked in Section 4, because that's where the failure modes are sharpest.

---

### Mistake 1 — Skipping the constraint read, guessing at complexity

```cpp
// WRONG mental model for Trapping Rain Water II (m, n <= 200):
// "I'll try every possible water level and flood-fill to check trapped regions."
// For each of up to 2*10^4 possible height levels, a flood fill over 40,000 cells
// is O(20000 * 40000) = 8*10^8 — too slow, and doesn't even model the physics
// correctly (water at different regions can settle at different levels).
```

> The constraint `m, n <= 200` (so `mn <= 40,000`) was screaming `O(mn log(mn))` from the start. A candidate who reads constraints first never explores the flood-fill-per-level idea — they immediately reach for the priority-queue frontier expansion, because `mn log(mn) ≈ 640,000` is the only idea in the "obviously fast enough" bucket that also respects the 2D adjacency structure.

---

### Mistake 2 — Confusing "binary search on a value" with "binary search on a partition index" with "binary search on the answer"

These three are different tools that novices frequently conflate:

```cpp
// Binary search ON A VALUE (e.g., "does X exist in this sorted array?")
// — searches over the DOMAIN of the data itself.

// Binary search ON THE ANSWER (Split Array Largest Sum)
// — the "value" being searched is not in the input at all; it's a
// hypothetical answer, tested via a separate feasibility function.
while (lo < hi) {
    long long mid = lo + (hi - lo) / 2;
    if (canSplit(nums, k, mid)) hi = mid; else lo = mid + 1;
}

// Binary search ON A PARTITION INDEX (Median of Two Sorted Arrays)
// — the "value" being searched is an INDEX into one array, and the
// check validates a boundary CONDITION between two arrays, not a
// single feasibility predicate.
while (lo <= hi) {
    int i = lo + (hi - lo) / 2;
    // ... check Aleft <= Bright && Bleft <= Aright ...
}
```

> Using a "binary search on a value" mental model for Median of Two Sorted Arrays leads people to binary search over the *numeric range* of the arrays and count elements ≤ mid — which technically can be made to work, but breaks silently on duplicate values (the count can plateau across multiple candidate values, and picking the median from a count alone, without also identifying WHICH element attains it, is genuinely fiddly). The partition-index formulation sidesteps this entirely because it never reasons about counts of a value — only about the sorted structure of indices.

---

### Mistake 3 — False generalization from 1D to 2D (Trapping Rain Water)

```cpp
// WRONG — trying to extend the 1D two-pointer approach to 2D by running it per-row:
for (int row = 0; row < m; row++) {
    // treat heightMap[row] as a 1D trapping-rain-water instance
    water += trap1D(heightMap[row]);
}
// FAILS: a cell can be perfectly enclosed within its row (looks like it
// traps water in the 1D sense) but actually leaks out through an adjacent
// row with a lower wall — 2D water finds the path of least resistance in
// ALL directions, not just left-right within one row.
```

> The fix isn't a smarter per-row algorithm — it's recognizing that "leaks out the weakest direction" is fundamentally a graph-frontier problem (Move 2: identify the sub-structure), not a row-independent one. Whenever a 1D solution's correctness argument secretly relies on "there are only two directions to leak," and the 2D version has four (or more) directions, that argument doesn't transfer — a structurally different tool (priority-queue expansion) is needed, not a per-axis reapplication of the old tool.

---

### Mistake 4 — Off-by-one errors in "binary search on the answer" boundaries

```cpp
// WRONG — using lo <= hi with hi = mid - 1 for a "find the minimum feasible X" search:
while (lo <= hi) {
    long long mid = lo + (hi - lo) / 2;
    if (canSplit(nums, k, mid)) hi = mid - 1;   // BUG: may skip past the true answer
    else lo = mid + 1;
}
return lo;   // off-by-one risk: is lo guaranteed to be feasible here?

// CORRECT — lo < hi with hi = mid (mid stays a CANDIDATE, never eliminated):
while (lo < hi) {
    long long mid = lo + (hi - lo) / 2;
    if (canSplit(nums, k, mid)) hi = mid;       // mid might BE the answer — keep it in range
    else lo = mid + 1;
}
return lo;   // loop invariant: lo == hi == the smallest feasible value
```

> The rule of thumb: when searching for the smallest value satisfying a monotonic predicate, never do `hi = mid - 1` on the "feasible" branch — that discards `mid` even though it might be the exact answer. Use `hi = mid` (keeping `mid` as a live candidate) paired with `lo < hi` as the loop condition, so the loop naturally converges to `lo == hi == answer` without a final off-by-one correction.

---

### Mistake 5 — Reaching for meeting-in-the-middle (or bitmask DP) at the wrong n

```cpp
// WRONG — over-engineering: using meeting-in-the-middle for n = 18
// Bitmask DP (O(2^n * n)) is simpler to write and just as fast here —
// 2^18 * 18 ≈ 4.7M operations. Meeting in the middle adds complexity
// (splitting, sorting, combining) for no speed benefit at this n.

// WRONG — the opposite mistake: using bitmask DP for n = 35
// 2^35 ≈ 3.4 * 10^10 — will not finish. This is exactly where
// meeting-in-the-middle (2^17 * 2 ≈ 260,000) is REQUIRED, not optional.
```

> Both directions of this mistake come from not doing Move 1 carefully. `n <= 22` is (usually) bitmask-DP territory; `n <= 40` is (usually) meeting-in-the-middle territory. Conflating the two thresholds either wastes implementation effort on unnecessary complexity, or produces a solution that confidently TLEs.

---

### Mistake 6 — Extracting the code from an editorial instead of the insight

Covered in depth in Section 8, but worth restating as a decomposition-stage mistake: copying editorial code (even after fully understanding it in the moment) without re-deriving it from a one-sentence insight produces a false sense of mastery. The test is always the same — close the editorial, and see if the one sentence alone is enough to rebuild the solution. If it isn't, the sentence was imprecise, and imprecise insights don't transfer to the next, differently-dressed problem in the same class.

---

## SECTION 10 — INTERVIEW SIMULATION: SPLIT ARRAY LARGEST SUM

**Interviewer:** "Given an array of positive integers and an integer k, split the array into k non-empty contiguous subarrays to minimize the largest sum among the k subarrays. What's your approach?"

**Candidate:**

*[First thirty seconds — restate and look for the monotonic-feasibility signal]*
> "The phrase 'minimize the largest sum' is a strong signal for binary searching on the answer, rather than trying to construct the optimal split directly. Before I commit to that, let me check the monotonicity: if some cap X lets me split the array into at most k pieces, does a larger cap X+1 also work? Yes — a looser cap can only merge pieces, never force more of them. So feasibility is monotonic in X, which means binary search on X is valid."

*[State the two-part decomposition before coding]*
> "This splits into two pieces I already know how to write: a binary search over the answer range `[max(nums), sum(nums)]` — the cap can never be smaller than the single largest element, since that element must sit alone in some piece if the cap forces it; and never needs to exceed the total sum, since a cap that large trivially puts everything in one piece. And a linear greedy feasibility check: given a cap X, walk the array once, greedily packing each piece as full as possible before starting a new one, and count how many pieces that requires."

*[Interviewer]: "Why is the greedy feasibility check optimal — how do you know greedily packing each piece full doesn't waste a better split elsewhere?"*
> "For a FIXED cap X, greedily packing each piece as full as possible before closing it is provably optimal for minimizing the NUMBER of pieces needed. Closing a piece earlier than necessary can only ever require an equal or greater number of total pieces for the remaining elements — it never helps, because leaving unused capacity in a piece is strictly wasted room. So the greedy check gives the true minimum piece count for that cap, which is exactly what we need to compare against k."

*[Code — 6 minutes]*
```cpp
bool canSplit(const vector<int>& nums, int k, long long maxSum) {
    int pieces = 1;
    long long cur = 0;
    for (int x : nums) {
        if (cur + x > maxSum) {
            pieces++;
            cur = x;
            if (pieces > k) return false;
        } else {
            cur += x;
        }
    }
    return true;
}

int splitArray(vector<int>& nums, int k) {
    long long lo = *max_element(nums.begin(), nums.end());
    long long hi = accumulate(nums.begin(), nums.end(), 0LL);
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (canSplit(nums, k, mid)) hi = mid;
        else lo = mid + 1;
    }
    return (int)lo;
}
```

*[Interviewer]: "What's the overall complexity, and how would this change if k could be up to n and nums.length up to 10^6?"*
> "`check` is O(n), and binary search does O(log(sum(nums) - max(nums))) iterations, so overall O(n log(sum)). At n = 10^6 with values up to, say, 10^6, sum can be up to 10^12, so log(sum) is around 40 — giving roughly 4*10^7 operations, comfortably fast. The DP alternative (`dp[i][j]` over index and pieces used) would be O(n^2 k) in its naive form, which is completely infeasible at this scale — this is exactly the case where recognizing the binary-search-on-the-answer reduction isn't just cleaner, it's the only approach that survives the larger constraints."

**RED FLAGS:**
- Reaching for the `dp[i][j]` formulation first without checking whether `n` and `k` make it feasible, when the "minimize the maximum" phrasing was already a clear signal for binary search.
- Using `hi = mid - 1` in the feasible branch (Mistake 4, Section 9) — silently drops the true answer from the search range in edge cases.
- Not being able to justify why the greedy feasibility check is optimal when asked — treating it as "obviously correct" instead of articulating the exchange argument (closing early never helps).

---

## SECTION 11 — TECHNIQUE AND COMPLEXITY SUMMARY

A consolidated reference of every problem worked in this pattern, matched to its decomposition move and complexity — the kind of table worth keeping open while doing timed practice.

| Problem | Decomposition move used | Time | Space | Core insight class (Section 3) |
|---|---|---|---|---|
| Trapping Rain Water II (407) | Identify sub-structure (Move 2) | O(mn log(mn)) | O(mn) | Priority-queue frontier expansion |
| Median of Two Sorted Arrays (4) | Binary search — on a partition (Move 5, variant) | O(log(min(m,n))) | O(1) | Binary search on a partition |
| Split Array Largest Sum (410) | Binary search on the answer (Move 5) | O(n log(sum)) | O(1) | Binary search on the answer |
| Meeting-in-the-middle subset sum | Change of variables / state split (Move 7) | O(2^(n/2) · n) | O(2^(n/2)) | Meeting in the middle |
| Maximal Rectangle (85, Pattern 18) | Reduce to a known problem (Move 3) | O(mn) | O(n) | Height array + histogram |
| Dungeon Game (174, Pattern 18) | Reverse the problem (Move 4) | O(mn) | O(mn) → O(n) | Reverse DP / reverse traversal |
| Russian Doll Envelopes (354) | Reduce to a known problem (Move 3) | O(n log n) | O(n) | Sort + LIS reduction |
| Subarrays with K Distinct Integers (992) | Change of variables (Move 7) | O(n) | O(k) | At-most(K) subtraction |

**How to read this table during practice:** when you finish a hard problem, add a row. Over time this becomes a cross-reference between "surface problem" and "underlying move" — the exact translation layer that lets you look at problem #400 in your practice history and immediately recall "this is the same move as problem #47," even when the two problems share no surface-level resemblance at all.

---

## SECTION 12 — PROBLEM SET

These are graded not by raw LeetCode difficulty label but by how many decomposition moves they require and how well-hidden the sub-structure is. Time yourself honestly — the point of this pattern is the process under pressure, not the final answer.

### WARMUP (solve in ≤ 20 minutes each — single, clearly-signaled move)

| # | Problem | Key move |
|---|---------|----------|
| 1 | Koko Eating Bananas (875) | Binary search on the answer — minimize the maximum eating speed |
| 2 | Capacity To Ship Packages Within D Days (1011) | Binary search on the answer — same shape as Split Array Largest Sum |
| 3 | Subarray Sum Equals K (560) | Prefix-sum-as-hashmap-key |

### CORE (solve in ≤ 40 minutes each — two moves combined)

| # | Problem | Key move |
|---|---------|----------|
| 4 | Split Array Largest Sum (410) | Binary search on the answer + greedy feasibility check |
| 5 | Median of Two Sorted Arrays (4) | Binary search on a partition |
| 6 | Russian Doll Envelopes (354) | Sort + reduce to LIS (Move 3) |
| 7 | Subarrays with K Distinct Integers (992) | At-most(K) subtraction (Move 7) |
| 8 | Maximal Rectangle (85, Pattern 18) | Height array + histogram (Move 2) |

### STRETCH (solve in ≤ 60 minutes each — the sub-structure is deliberately obscured)

| # | Problem | Key move |
|---|---------|----------|
| 9 | Trapping Rain Water II (407) | Priority-queue frontier expansion (Move 2 + graph traversal) |
| 10 | Swim in Rising Water (778) | Priority-queue frontier expansion, OR binary search on the answer + BFS/DSU feasibility — solve it BOTH ways |
| 11 | Sliding Window Median (480) | Two heaps + lazy deletion under a sliding window |
| 12 | Find Median from Data Stream (295) | Two heaps for a running median |

### CONTEST (attempt under a real timer — expect at least one false start)

| # | Problem | Key move |
|---|---------|----------|
| 13 | Closest Subsequence Sum (1755) | Meeting in the middle + two-pointer combine (Section 5 extension) |
| 14 | Ones and Zeroes (474, Pattern 19 preview) | 2D knapsack — recognize the "two resources = two dimensions" reduction |
| 15 | Kth Smallest in a Sorted Matrix (378) | Binary search on the answer, over VALUES, combined with a per-row counting pass |
| 16 | Minimum Cost to Make at Least One Valid Path in a Grid (1368) | 0-1 BFS with deque |

**Discipline for this problem set specifically:** for every problem, before writing code, write the one-sentence decomposition ("this is binary search on the answer, with a greedy O(n) feasibility check") in a comment at the top of your solution file. If you cannot write that sentence confidently, you are not ready to code yet — go back to Section 2 and Section 7.

---

## SECTION 13 — PATTERN CONNECTIONS

This capstone pattern doesn't introduce new primitives — it is the connective tissue across everything from Pattern 1 onward. Naming the connections explicitly is itself a decomposition exercise.

1. **Patterns 1–3 (Two Pointers, Sliding Window):** the "sort then two-pointer" and "at-most(K) subtraction" insight classes (Section 3) are these patterns' techniques, redeployed as *sub-routines* inside harder problems rather than as the whole solution. Recognizing a two-pointer opportunity buried inside a larger problem is Move 2 (identify the sub-structure) applied to Pattern 1's own vocabulary.

2. **Pattern 04 (Monotonic Stack):** Maximal Rectangle and Trapping Rain Water are the two clearest examples in this document of a monotonic-stack or priority-queue idea being the "known piece" that a hard problem decomposes into. Every time Section 3's catalog says "monotonic stack for next-greater," that's Pattern 04's core template being reused as a component, not reinvented.

3. **Pattern 06 (Prefix Sums) and Pattern 09-ish range-update patterns:** the "change of variables" move (Move 7) and the "difference array" catalog row are direct extensions of prefix sums into range-update territory — the same `O(1)` query-after-`O(n)`-preprocessing idea, just applied to updates instead of queries.

4. **Pattern 17–19 (1D/2D DP, Knapsack):** Dungeon Game's reverse-DP is Move 4 in this pattern's vocabulary — this document simply generalizes what Pattern 18 taught as a specific technique into a *reflex to check for* on any new problem where a forward transition needs unavailable future information.

5. **Binary search patterns (wherever introduced earlier in the curriculum):** this pattern's Section 2 Move 5 and Section 4B/4C generalize "binary search on a sorted array" into "binary search on an abstract, monotonic answer space" and "binary search on a partition index across two structures" — two applications far less obvious than searching a single sorted array, and exactly the kind of generalization that separates 1600-level binary search usage from 2000-level usage.

6. **Meeting in the middle (Section 5) as a bridge to Pattern 21-ish bitmask DP:** the constraint-reading table in Move 1 is the single artifact that ties bitmask DP (`n <= 20`) and meeting-in-the-middle (`n <= 40`) together as two solutions to the same underlying problem shape (exponential state spaces), differentiated only by which `n` threshold you're facing.

7. **Every pattern, via the insight catalog (Section 3):** the deepest claim of this capstone is that Patterns 1 through 37 are not a list to memorize sequentially — they are entries in exactly the kind of catalog Section 3 models. The skill this document teaches is the meta-skill of maintaining, querying, and extending that catalog for the rest of your problem-solving life, long after this curriculum ends.

---

## MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading the responses.

---

**Q1: Given `n <= 18` in a problem about visiting all of a set of "cities" with some additional constraint, what complexity class does this suggest, and what technique?**

> `n <= 18` gives `2^18 ≈ 262,144` — squarely in bitmask-DP territory (`O(2^n · n)` or `O(2^n · n^2)` is comfortably fast: `262144 × 18 ≈ 4.7M`, or even `× 18^2 ≈ 85M`, both fine). This is the classic signature of a **Traveling-Salesman-style bitmask DP**: `dp[mask][last]` = best answer having visited exactly the cities in `mask`, currently at city `last`. Note `n <= 18` (not `n <= 40`) rules out meeting-in-the-middle as the primary technique here — that's reserved for problems where the aggregate being computed (like subset sum) doesn't require tracking a "current position," only a running total.

---

**Q2: In Trapping Rain Water II, why must we always pop the minimum-height cell from the priority queue, not just any boundary cell?**

> Popping the minimum guarantees that when we compute water for a newly visited interior cell using the popped cell's height as the enclosing wall, that height truly is the tightest (lowest) constraint discovered so far along any path to that interior cell. If we popped an arbitrary (possibly taller) boundary cell first, we might record water using a wall that's higher than the true minimum enclosing wall reachable through a different, not-yet-processed direction — overestimating trapped water. This is the identical greedy-correctness argument that makes Dijkstra's algorithm correct: always finalize the globally closest (here: lowest) unvisited node next.

---

**Q3: In Median of Two Sorted Arrays, why do we binary search on the smaller array, and what invariant does the partition maintain?**

> Binary searching the smaller array (size `m <= n`) bounds the search space to `O(log m)` iterations and — critically — guarantees that the corresponding partition index `j = half - i` in the larger array always stays within `[0, n]` for every `i` in `[0, m]`, avoiding extra bounds-juggling. The invariant maintained at a valid partition is: every element in the combined "left half" (`nums1[0..i-1]` and `nums2[0..j-1]`) is `<=` every element in the combined "right half" (`nums1[i..]` and `nums2[j..]`), and the left half has exactly `⌈(m+n)/2⌉` elements — which makes the median directly readable from the four boundary values without ever merging the arrays.

---

**Q4: Why does Split Array Largest Sum's "feasibility check" run in `O(n)`, and why does that make binary search on the answer `O(n log(sum))` overall?**

> `check(X)` does a single linear pass over the array, greedily accumulating into the current piece and starting a new piece whenever the running sum would exceed `X` — each element is visited exactly once, so it's `O(n)`. Binary search over the range `[max(nums), sum(nums)]` performs `O(log(sum(nums) - max(nums)))` iterations, each calling `check` once. Total: `O(n log(sum))` — dramatically cheaper than the `O(n^2 k)` DP alternative, and asymptotically independent of `k` entirely.

---

**Q5: Meeting in the middle reduces `O(2^n)` to `O(2^(n/2) log(2^(n/2)))`. Why is this not "cheating" — what fundamental limit does it still respect?**

> It doesn't beat the information-theoretic size of the subset space — there are still `2^n` total subsets, and `2^(n/2) · 2^(n/2) = 2^n` accounts for all of them combinatorially. What changes is that we never *materialize* or *enumerate* the full cross product; we generate the two square-root-sized factor sets independently (each cheap, `O(2^(n/2))`) and then **combine** them algorithmically — via sorting and binary search, or a two-pointer sweep — which costs only `O(2^(n/2) log(2^(n/2)))` instead of `O(2^n)`. It's the same trick as factoring `10^12` operations into `10^6 × 10^6` and doing the two `10^6`-sized halves separately plus a cheap `O(10^6 log(10^6))` merge, instead of one `10^12`-sized brute force.

---

**Q6: When an editorial's solution "clicks" immediately but your unaided attempt didn't find it, what does that gap tell you to add to your insight catalog?**

> It tells you that you have a *recognition* gap, not necessarily a *reasoning* gap — you can follow the logic once shown, but couldn't generate the frame yourself under the problem's specific dressing. The fix is not "read more editorials" (that trains recognition of specific problems) — it's to extract the general insight class (Section 3), restate it abstractly, and actively practice *searching* for that class's signature (specific constraint ranges, specific phrasings like "minimize the maximum") in future unrelated problems, so the next time that signature appears, you reach for the class proactively instead of needing it demonstrated.

---

**Q7: What's the difference between "I couldn't solve this" and "I didn't try the right lens"? How do you tell after the fact which happened?**

> "I couldn't solve this" implies you correctly identified the right technique/lens but failed to execute it (a bug, a wrong invariant, an off-by-one) — the fix is more careful implementation practice. "I didn't try the right lens" means you spent your time exploring an approach that was never going to work, and the correct approach (binary search on the answer, reversing the DP direction, meeting in the middle) never entered your consideration set at all — the fix is deliberately drilling the decomposition toolkit (Section 2) and the stuck ladder (Section 7) so the *menu* of lenses is more complete next time. You can tell which happened after the fact by checking: did you ever, even briefly, consider the technique the editorial used and reject it for a specific (even if wrong) reason? If yes, it was an execution gap. If the technique never crossed your mind at all, it was a lens gap — and lens gaps are exactly what this pattern is designed to close.

---

## WHAT TO DO AFTER THIS PATTERN

1. **Solve all three fully-worked problems from Section 4 cold, from scratch, without looking back at this document.** Then compare your code to the versions here — not to copy, but to check whether your invariant statements (the one-sentence description of each `dp`/binary-search state) match in spirit.

2. **Do the meeting-in-the-middle extension yourself:** implement the closest-subsequence-sum variant (LC 1755) using the two-pointer combine step described in Section 5, instead of the exact-match binary-search version given in the code.

3. **Build your personal insight log now, not "eventually."** Start with the 16 rows of Section 3's catalog as seed entries, and add one new row every time you solve (or upsolve) a hard problem for the next month. Revisit it weekly.

4. **Run one virtual contest this week** under real time pressure, applying Section 2's toolkit explicitly — literally say the move names to yourself ("constraint check," "sub-structure," "reverse it," "binary search the answer") as you read each problem, until the habit is automatic rather than deliberate.

5. **Pick 3 hard problems you solved more than a month ago** (from any earlier pattern in this curriculum) and re-solve them blind, cold, with no notes, to test whether the insight has actually stuck or only felt understood in the moment.

---

## FINAL SIGN-OFF — CAPSTONE REFLECTION

**This is Pattern 38 — the last pattern in the 1600 → 2000 curriculum.**

Look back at where this started. Pattern 1 was almost certainly something foundational — two pointers, or basic sliding window — a shape simple enough that recognizing it was most of the battle. By the time you reached Pattern 18 (2D DP), the shapes had gotten subtler: reverse DP, two-agent state compression, DP semantics that weren't "path from A to B" at all but "structural property of a region." By Pattern 38, the premise has flipped entirely: there is no new *shape* to learn. What you've been building, pattern by pattern, problem by problem, trace by trace, is a **library of shapes large enough that most hard problems are now visibly combinations of shapes you already own** — and a decomposition process (Section 2), an insight catalog (Section 3), and a recovery procedure for when you're stuck (Section 7) that let you find the right combination under pressure.

**That is what 2000+ actually means.** Not "knows more algorithms than a 1600." A 1600 and a 2100 competitor often know an overlapping set of named algorithms. The difference is that the 2100 competitor, faced with a problem statement that has deliberately obscured which algorithms apply, can still — reliably, under a two-hour clock, with wrong turns and false starts along the way — find the decomposition. That reliability is a trained skill, not a talent, and it is trained by exactly the loop in Section 6: honest struggle, disciplined post-mortem, insight extraction, re-derivation, spaced repetition.

**What to do next, concretely:**

- **Contest cadence:** aim for one rated contest a week, minimum — Codeforces Div. 2/3 rounds, or an equivalent weekly cadence on whatever platform you're rated on. Rating is noisy week to week; trust the trend over a 3–6 month window, not any single contest. Skipped weeks compound into skill atrophy faster than most people expect — the decomposition reflex, like any reflex, decays without use.

- **Revision cadence:** once a month, pull 5–10 of your hardest solved problems from the preceding months and re-solve them cold, no notes. This is not busywork — it's the direct test of whether Section 6's insight log is actually converting into durable pattern recognition or just short-term familiarity.

- **Mock interviews, if the 2000+ goal is interview-driven rather than contest-driven:** run at least one mock hard-problem interview a week, with a real person if possible, where you narrate the Section 2 decomposition process out loud in real time — constraint reading first, sub-structure naming second, code last. Interviewers at this level are evaluating the *process* at least as much as the final code; narrating it is a trained skill in its own right, distinct from solving silently.

- **Give back what you've built.** Once your insight log has a few hundred entries, teaching it to someone else — writing your own capstone notes, mentoring someone earlier in their 1600→2000 climb — is one of the fastest ways to find the gaps in your own understanding. Explaining Section 4's Dijkstra-correctness argument for Trapping Rain Water II to someone who's never seen it will surface, immediately, whether you actually internalized *why* the minimum-first pop is correct, or just remembered that it is.

The 37 patterns before this one gave you the vocabulary. This one gives you the grammar — how to compose that vocabulary into sentences you've never written before, under a clock, on a problem you've never seen. That is the entire game above 2000. Go play it.

---

*Pattern 38 Complete — Hard Problem Decomposition*
*Curriculum Complete — 1600 → 2000+*
