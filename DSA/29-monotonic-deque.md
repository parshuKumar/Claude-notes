# PATTERN 29 — MONOTONIC DEQUE
## Sliding Window Max/Min, Deque-Optimized DP, Prefix-Sum Deques — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The one-sentence pitch:** a monotonic deque is what you reach for the moment a monotonic-stack problem gets a **sliding window** bolted onto it.

Pattern 04 taught you the monotonic stack: elements enter from one end, get resolved (popped) when a bigger/smaller element arrives, and the answer is extracted at pop time. That works perfectly when the "history" you're comparing against is unbounded — "everything to the left of me, forever." The moment the problem says **"in the last k elements"** or **"within this window,"** the stack breaks. A stack can only remove from the top. It has no way to evict an element just because it has aged out of a window — that element might still be sitting at the bottom, buried under newer pushes, with no way to reach it without destroying everything above it.

The monotonic deque fixes this with one structural change: it can pop from **both ends**.

- **Pop from the back** — same job as the monotonic stack: throw away elements that can never again be the answer because something better just arrived. This maintains the monotonic invariant.
- **Pop from the front** — the new capability: throw away the element at the front once it has fallen outside the current window. This maintains window validity.

That's the entire idea. Everything else in this pattern — sliding window max, deque-optimized DP, prefix-sum deques — is a variation on "push, evict-from-back-for-monotonicity, evict-from-front-for-expiry, read-the-front."

**Where people get stuck at 1600:**

1. **Storing values instead of indices.** If you push `nums[i]` instead of `i`, you can never tell whether the front of the deque has expired — you don't know *where* it came from. This is the single most common bug in this pattern (see Common Mistakes).
2. **Confusing "window of fixed size k" with "window defined implicitly by DP recurrence."** LC 239 (Sliding Window Maximum) has an explicit window of size k. LC 1696 (Jump Game VI) has an *implicit* window: `dp[i]` depends on `dp[i-k..i-1]`, which is a sliding window over the **dp array**, not the input array. Once you see that the recurrence itself defines a window, deque optimization becomes obvious.
3. **Getting the pop-from-back condition backwards.** For a "max" deque you pop back elements that are `<=` the incoming element (they're now useless — dominated). For a "min" deque you pop back elements that are `>=`. Mixing these up silently breaks the invariant without crashing your code.
4. **Not connecting this to prefix sums.** LC 862 (Shortest Subarray with Sum ≥ K) looks like nothing you've seen — until you realize it's "smallest `i - j` such that `P[i] - P[j] >= K`," which is a monotonic deque over the **prefix sum array**, not the original array.

**How this differs from Pattern 04 (Monotonic Stack) at a glance:**

| | Monotonic Stack (P04) | Monotonic Deque (P29) |
|---|---|---|
| Structure | `stack<int>` (indices) | `deque<int>` (indices) |
| Ends used | One (top) | Two (front and back) |
| Back operation | Pop when new element violates order | Same — pop when new element violates order |
| Front operation | None — stack has no front access | Pop when front index has expired (fallen out of window) |
| Answer lives at | Pop time (the popped element's answer) | The front, read at query time (no pop needed to "answer") |
| Classic problem | Next Greater Element, Largest Rectangle | Sliding Window Maximum, Jump Game VI |
| Window concept | Whole array so far (unbounded) | Fixed-size or DP-implicit window (bounded) |

If you can write the four boundary templates from Pattern 04 (PSE/NSE/PGE/NGE) from memory, you already have 80% of the muscle memory for this pattern. The remaining 20% is: (a) remembering to evict from the front, and (b) recognizing when the "window" isn't literally `nums[i-k+1..i]` but something derived — a DP array, a prefix-sum array, or a `y - x` transformed array.

---

## SECTION 2 — THE MONOTONIC DEQUE TAXONOMY

### Type 1: Window Maximum
Structure: **decreasing** deque (front = largest, back = smallest of the "surviving" candidates)
Pop back: while `nums[i] >= nums[back]` — the incoming element dominates
Pop front: while `front index <= i - k` — expired
Read: `nums[dq.front()]`
Problems: Sliding Window Maximum (239), Constrained Subsequence Sum's max half (1425)

### Type 2: Window Minimum
Structure: **increasing** deque (front = smallest, back = largest of survivors)
Pop back: while `nums[i] <= nums[back]` — the incoming element dominates
Pop front: while `front index <= i - k` — expired
Read: `nums[dq.front()]`
Problems: Sliding Window Minimum, the "min" half of Longest Subarray With Abs Diff ≤ Limit (1438)

### Type 3: Deque + DP (the window is IMPLICIT, defined by the recurrence)
State: `dp[i] = nums[i] + best(dp[i-k], dp[i-k+1], ..., dp[i-1])`
The deque stores **indices into the dp array**, not the input array.
Because `dp[i]` is computed using the deque *before* `dp[i]` itself is pushed, this is a single forward pass — push happens right after computing, so the deque is always "one step ahead" ready for the next query.
Problems: Jump Game VI (1696), Constrained Subsequence Sum (1425)

### Type 4: Deque + Prefix Sum
Transform the array to prefix sums `P[0..n]`. The target condition ("subarray sum ≥ K") becomes "find `i, j` with `i > j` and `P[i] - P[j] >= K`, minimizing `i - j`."
Deque is **increasing** on `P` values (a candidate `P[j]` is useless as a left-boundary the moment a later, smaller `P[j']` appears — it can never produce a shorter *and* valid answer than `j'` would).
Pop front when `P[i] - P[front] >= K` (found an answer — record and evict, since a shorter window from this front is now impossible, and evicting one lets a later i test the next front).
Pop back when `P[i] <= P[back]` (back is dominated: shorter distance AND smaller value beats it in every future comparison).
Problems: Shortest Subarray with Sum at Least K (862)

### Type 5: Deque + Geometric Transform
Rewrite the objective so it splits into "a term depending only on `i`" plus "a term depending only on `j`," then maintain a monotonic deque on the `j`-term.
Problems: Max Value of Equation (1499) — `y_i + y_j + |x_i - x_j|` becomes `(y_j - x_j) + (y_i + x_i)` for `j < i`.

### Type 6: Two Deques (max AND min simultaneously)
When a problem needs both the running max and running min of the same window (e.g., "keep the window valid while `max - min <= limit`"), run a max-deque and a min-deque **in parallel** over the same two pointers.
Problems: Longest Continuous Subarray With Absolute Diff ≤ Limit (1438)

---

## SECTION 3 — TEMPLATE 1: SLIDING WINDOW MAXIMUM (THE FOUNDATION)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 239 — Sliding Window Maximum
// Given nums and window size k, return the max of each window of size k
// as the window slides from left to right.
//
// INVARIANT: dq holds INDICES, values at those indices are in
//            DECREASING order from front to back.
//            dq.front() is always the index of the current window's max.
//
// Time: O(n) amortized — each index pushed once, popped at most once
// Space: O(k) — deque holds at most k indices
// ─────────────────────────────────────────────────────────────────

#include <deque>
#include <vector>
using namespace std;

vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;             // stores INDICES, decreasing by value
    vector<int> result;
    int n = nums.size();

    for (int i = 0; i < n; i++) {
        // STEP 1: evict from the BACK — maintain monotonic decreasing order.
        // If nums[i] >= nums[dq.back()], the back element can NEVER be the
        // answer for any future window (nums[i] is both newer AND at least
        // as large — it dominates in every way). Throw it away.
        while (!dq.empty() && nums[dq.back()] <= nums[i]) {
            dq.pop_back();
        }
        dq.push_back(i);

        // STEP 2: evict from the FRONT — maintain window validity.
        // The window covering index i is [i-k+1, i]. If the front index
        // has fallen behind that, it's expired — no longer in the window.
        if (dq.front() <= i - k) {
            dq.pop_front();
        }

        // STEP 3: once the window is full (i >= k-1), record the answer.
        // The front of the deque IS the max — no searching needed.
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }

    return result;
}
```

**FULL STEP-BY-STEP TRACE for `nums = [1,3,-1,-3,5,3,6,7]`, `k = 3`:**

```
i=0, val=1:
  back-evict: dq empty, nothing to pop.
  push 0.                          dq = [0]           (values: [1])
  front-evict: dq.front()=0, 0 <= 0-3=-3? no.
  i < k-1=2, no output yet.

i=1, val=3:
  back-evict: nums[dq.back()=0]=1 <= 3? yes, pop.     dq = []
  push 1.                          dq = [1]           (values: [3])
  front-evict: dq.front()=1, 1 <= 1-3=-2? no.
  i < 2, no output.

i=2, val=-1:
  back-evict: nums[1]=3 <= -1? no. stop.
  push 2.                          dq = [1, 2]        (values: [3, -1])
  front-evict: dq.front()=1, 1 <= 2-3=-1? no.
  i == 2 == k-1: OUTPUT nums[dq.front()=1] = 3

i=3, val=-3:
  back-evict: nums[2]=-1 <= -3? no. stop.
  push 3.                          dq = [1, 2, 3]     (values: [3, -1, -3])
  front-evict: dq.front()=1, 1 <= 3-3=0? no.
  OUTPUT nums[1] = 3

i=4, val=5:
  back-evict: nums[3]=-3<=5 pop; nums[2]=-1<=5 pop; nums[1]=3<=5 pop.  dq=[]
  push 4.                          dq = [4]           (values: [5])
  front-evict: dq.front()=4, 4 <= 4-3=1? no.
  OUTPUT nums[4] = 5

i=5, val=3:
  back-evict: nums[4]=5 <= 3? no. stop.
  push 5.                          dq = [4, 5]        (values: [5, 3])
  front-evict: dq.front()=4, 4 <= 5-3=2? no.
  OUTPUT nums[4] = 5

i=6, val=6:
  back-evict: nums[5]=3<=6 pop; nums[4]=5<=6 pop.     dq = []
  push 6.                          dq = [6]           (values: [6])
  front-evict: dq.front()=6, 6 <= 6-3=3? no.
  OUTPUT nums[6] = 6

i=7, val=7:
  back-evict: nums[6]=6 <= 7? yes, pop.                dq = []
  push 7.                          dq = [7]           (values: [7])
  front-evict: dq.front()=7, 7 <= 7-3=4? no.
  OUTPUT nums[7] = 7

FINAL RESULT: [3, 3, 5, 5, 6, 7]   ✓ (matches LeetCode's expected output)
```

**Why the back-evict uses `<=` and not `<`:** with `<=`, equal values also get evicted — since the incoming element is at a later index and has the same value, it will remain valid in the window for at least as long as the old one, so the old duplicate is strictly useless. Using `<` instead would still be *correct* (both survive), just with a marginally larger deque in the presence of duplicates.

---

## SECTION 4 — TEMPLATE 2: SLIDING WINDOW MINIMUM

```cpp
// Sliding Window Minimum — the mirror image of Template 1.
// Flip the comparison direction: INCREASING deque, front = minimum.
// Time: O(n) amortized | Space: O(k)

vector<int> minSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;             // stores INDICES, increasing by value
    vector<int> result;
    int n = nums.size();

    for (int i = 0; i < n; i++) {
        // back-evict: throw away anything >= the incoming value —
        // it can never again be the minimum once a smaller, newer value exists.
        while (!dq.empty() && nums[dq.back()] >= nums[i]) {
            dq.pop_back();
        }
        dq.push_back(i);

        // front-evict: same expiry rule as before.
        if (dq.front() <= i - k) {
            dq.pop_front();
        }

        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }

    return result;
}
```

**MICRO-TRACE for `nums = [4,2,12,3,8,3,5,4]`, `k = 4`:**

```
i=0 val=4: push. dq=[0](4)
i=1 val=2: back-evict nums[0]=4>=2 pop. push. dq=[1](2)
i=2 val=12: back-evict nums[1]=2>=12? no. push. dq=[1,2](2,12)
i=3 val=3: back-evict nums[2]=12>=3 pop. nums[1]=2>=3? no. push. dq=[1,3](2,3)
           i==3==k-1: OUTPUT nums[1]=2
i=4 val=8: back-evict nums[3]=3>=8? no. push. dq=[1,3,4](2,3,8)
           front-evict: front=1 <= 4-4=0? no.
           OUTPUT nums[1]=2
i=5 val=3: back-evict nums[4]=8>=3 pop. nums[3]=3>=3 pop (equal, evict). push. dq=[1,5](2,3)
           front-evict: front=1 <= 5-4=1? YES, pop. dq=[5](3)
           OUTPUT nums[5]=3
i=6 val=5: back-evict nums[5]=3>=5? no. push. dq=[5,6](3,5)
           front-evict: front=5 <= 6-4=2? no.
           OUTPUT nums[5]=3
i=7 val=4: back-evict nums[6]=5>=4 pop. push. dq=[5,7](3,4)
           front-evict: front=5 <= 7-4=3? no.
           OUTPUT nums[5]=3

RESULT: [2, 2, 3, 3, 3]
```

Note the `>=` on back-evict for the min deque uses the same "duplicate goes with the newer index" convention as the max deque — consistent, easy to remember: **evict on tie, keep the newest**.

---

## SECTION 5 — TEMPLATE 3: JUMP GAME VI (DEQUE + DP, IMPLICIT WINDOW)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1696 — Jump Game VI
// Start at index 0. From index i you may jump to any index in
// [i+1, i+k]. Maximize the sum of visited values ending at index n-1.
//
// State:    dp[i] = max score to reach index i
// Transition: dp[i] = nums[i] + max(dp[i-k], dp[i-k+1], ..., dp[i-1])
//
// THE KEY REALIZATION: "max of the last k dp values" is EXACTLY the
// sliding-window-maximum problem, except the array being windowed is
// the dp array itself, which we are building AS we go. The deque
// stores indices into dp, maintained decreasing by dp VALUE.
//
// Time: O(n) amortized | Space: O(k) for deque + O(n) for dp
// ─────────────────────────────────────────────────────────────────

int maxResult(vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> dp(n);
    dp[0] = nums[0];

    deque<int> dq;          // indices into dp, decreasing by dp[] value
    dq.push_back(0);

    for (int i = 1; i < n; i++) {
        // front-evict FIRST: the window for dp[i] is [i-k, i-1].
        // Any index < i-k is out of reach from i (jump length > k).
        while (!dq.empty() && dq.front() < i - k) {
            dq.pop_front();
        }

        // READ before writing dp[i] — dq.front() holds the best
        // reachable predecessor within the last k steps.
        dp[i] = nums[i] + dp[dq.front()];

        // back-evict: any earlier index with dp <= dp[i] is now useless —
        // dp[i] is both closer (better reach) and at least as large.
        while (!dq.empty() && dp[dq.back()] <= dp[i]) {
            dq.pop_back();
        }
        dq.push_back(i);
    }

    return dp[n - 1];
}
```

**FULL TRACE for `nums = [1,-1,-2,4,-7,3]`, `k = 2`:**

```
dp[0] = 1.  dq = [0]                       (dp values: [1])

i=1:
  front-evict: dq.front()=0, 0 < 1-2=-1? no.
  dp[1] = nums[1] + dp[dq.front()=0] = -1 + 1 = 0
  back-evict: dp[dq.back()=0]=1 <= dp[1]=0? no. stop.
  push 1.   dq = [0, 1]                    (dp values: [1, 0])

i=2:
  front-evict: dq.front()=0, 0 < 2-2=0? no (not strictly less).
  dp[2] = nums[2] + dp[0] = -2 + 1 = -1
  back-evict: dp[1]=0 <= -1? no. stop.
  push 2.   dq = [0, 1, 2]                 (dp values: [1, 0, -1])

i=3:
  front-evict: dq.front()=0, 0 < 3-2=1? YES, pop.  dq = [1, 2]
                dq.front()=1, 1 < 1? no. stop.
  dp[3] = nums[3] + dp[dq.front()=1] = 4 + 0 = 4
  back-evict: dp[2]=-1<=4 pop; dp[1]=0<=4 pop.     dq = []
  push 3.   dq = [3]                       (dp values: [4])

i=4:
  front-evict: dq.front()=3, 3 < 4-2=2? no.
  dp[4] = nums[4] + dp[3] = -7 + 4 = -3
  back-evict: dp[3]=4 <= -3? no.
  push 4.   dq = [3, 4]                    (dp values: [4, -3])

i=5:
  front-evict: dq.front()=3, 3 < 5-2=3? no (not strictly less).
  dp[5] = nums[5] + dp[3] = 3 + 4 = 7
  back-evict: dp[4]=-3<=7 pop; dp[3]=4<=7 pop.     dq = []
  push 5.   dq = [5]

RESULT: dp[5] = 7   ✓ (matches LeetCode's expected output for this example)
```

**Why front-evict comes before the dp computation, but back-evict comes after:** the front-evict is a *validity* check — it must happen before you read `dq.front()`, otherwise you might read a stale, out-of-range index. The back-evict is a *maintenance* operation for future queries — it can only happen after `dp[i]` is known, since you need `dp[i]`'s value to know what to evict.

---

## SECTION 6 — TEMPLATE 4: CONSTRAINED SUBSEQUENCE SUM

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1425 — Constrained Subsequence Sum
// Choose a subsequence such that for every two consecutive chosen
// indices i < j, j - i <= k. Maximize the sum. (May choose a single
// element, and negative numbers may be worth "resetting" from.)
//
// State: dp[i] = max sum of a valid subsequence ENDING at index i
//                 (index i must be included)
// Transition: dp[i] = nums[i] + max(0, dp[i-k], ..., dp[i-1])
//             The "max(0, ...)" term says: either extend the best
//             recent subsequence, OR start fresh at i if every
//             reachable predecessor's dp is negative (not worth taking).
//
// Same deque-window machinery as Jump Game VI, with one twist: the
// value fed into the "max with predecessors" is clamped at 0 before
// use, but the RAW dp[i] (not clamped) is what gets pushed into the
// deque, because dp[i] itself might still be needed with its true
// (possibly negative) value for width bookkeeping. In practice we
// keep the deque holding indices with UNCLAMPED dp values — the
// clamping only happens at the point we compute dp[i] from the front.
//
// Answer = max over all i of dp[i] (the best subsequence might not
// end at the last index).
//
// Time: O(n) amortized | Space: O(k)
// ─────────────────────────────────────────────────────────────────

int constrainedSubsetSum(vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> dp(n);
    deque<int> dq;               // indices, decreasing by dp[] value
    int ans = INT_MIN;

    for (int i = 0; i < n; i++) {
        // front-evict: predecessor must be within the last k indices.
        while (!dq.empty() && dq.front() < i - k) {
            dq.pop_front();
        }

        // best predecessor contribution, clamped at 0 (0 = "start fresh")
        int bestPrev = dq.empty() ? 0 : max(0, dp[dq.front()]);
        dp[i] = nums[i] + bestPrev;
        ans = max(ans, dp[i]);

        // back-evict: dominated (smaller-or-equal, older) entries are useless.
        while (!dq.empty() && dp[dq.back()] <= dp[i]) {
            dq.pop_back();
        }
        dq.push_back(i);
    }

    return ans;
}
```

**FULL TRACE for `nums = [10,2,-10,5,20]`, `k = 2`:**

```
i=0: dq empty -> bestPrev=0. dp[0]=10+0=10. ans=10.
     back-evict: dq empty. push 0.        dq=[0]           (dp: [10])

i=1: front-evict: front=0, 0 < 1-2=-1? no.
     bestPrev = max(0, dp[0]=10) = 10. dp[1]=2+10=12. ans=12.
     back-evict: dp[0]=10<=12 pop.        dq=[]
     push 1.                              dq=[1]           (dp: [12])

i=2: front-evict: front=1, 1 < 2-2=0? no.
     bestPrev = max(0, dp[1]=12) = 12. dp[2]=-10+12=2. ans=12.
     back-evict: dp[1]=12<=2? no.
     push 2.                              dq=[1,2]         (dp: [12, 2])

i=3: front-evict: front=1, 1 < 3-2=1? no (not strictly less).
     bestPrev = max(0, dp[1]=12) = 12. dp[3]=5+12=17. ans=17.
     back-evict: dp[2]=2<=17 pop; dp[1]=12<=17 pop.  dq=[]
     push 3.                              dq=[3]           (dp: [17])

i=4: front-evict: front=3, 3 < 4-2=2? no.
     bestPrev = max(0, dp[3]=17) = 17. dp[4]=20+17=37. ans=37.
     back-evict: dp[3]=17<=37 pop.        dq=[]
     push 4.                              dq=[4]

RESULT: ans = 37   ✓ (matches LeetCode's expected output)
```

**On the "keeping positive" framing:** some write-ups describe this deque as "only keeping positive dp values." That's a slightly looser way of saying the same thing — since we clamp negative predecessors to 0 anyway, an index with `dp[idx] <= 0` will always lose the back-evict battle to literally any later index with `dp >= 0` (which every index has a chance of achieving, since `nums[i] + 0` is always an option). In practice the back-evict condition `dp[back] <= dp[i]` naturally flushes non-positive stale entries over time — you don't need a special case for it.

---

## SECTION 7 — TEMPLATE 5: SHORTEST SUBARRAY WITH SUM AT LEAST K

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 862 — Shortest Subarray with Sum at Least K
// nums can contain NEGATIVE numbers (this is why two-pointer sliding
// window, which requires monotonic sums, does NOT work here).
//
// TRANSFORM: build prefix sums P[0..n], P[0]=0, P[i]=P[i-1]+nums[i-1].
// A subarray nums[j..i-1] has sum P[i] - P[j]. We want the SHORTEST
// (i - j) such that P[i] - P[j] >= K, over all 0 <= j < i <= n.
//
// DEQUE INVARIANT: dq holds indices into P, with P[dq[0]] < P[dq[1]]
// < ... strictly increasing left to right (increasing deque).
//
// WHY pop the BACK when P[i] <= P[back]:
//   Suppose P[back] >= P[i], and back's index < i (back is older).
//   For any future i', if P[back] were going to satisfy
//   P[i'] - P[back] >= K, then P[i] (which is <= P[back] but at a
//   LATER index, hence a SHORTER candidate window) satisfies the sum
//   condition just as easily AND gives a shorter width. back is
//   strictly dominated — discard it.
//
// WHY pop the FRONT when P[i] - P[front] >= K:
//   This means we found a valid subarray of length i - front. Once
//   found, record it. Then discard front — could a LATER i' (i' > i)
//   ever use this SAME front for a SHORTER answer? No: i already gave
//   the shortest answer achievable using front (any i' > i only
//   makes the width i' - front LARGER). front is now useless. Evict.
//   Keep checking the new front — it might ALSO already satisfy the
//   condition with the current i (multiple evictions per i are
//   normal and are what make this O(n) amortized).
//
// Time: O(n) amortized | Space: O(n) worst case for deque
// ─────────────────────────────────────────────────────────────────

int shortestSubarray(vector<int>& nums, int k) {
    int n = nums.size();
    vector<long long> P(n + 1, 0);
    for (int i = 0; i < n; i++) P[i + 1] = P[i] + nums[i];

    deque<int> dq;              // indices into P, increasing by P[] value
    int best = INT_MAX;

    for (int i = 0; i <= n; i++) {
        // front-evict / answer-check: keep peeling while the window
        // [front, i] already satisfies the sum condition.
        while (!dq.empty() && P[i] - P[dq.front()] >= k) {
            best = min(best, i - dq.front());
            dq.pop_front();
        }

        // back-evict: maintain strictly increasing P values.
        while (!dq.empty() && P[i] <= P[dq.back()]) {
            dq.pop_back();
        }

        dq.push_back(i);
    }

    return best == INT_MAX ? -1 : best;
}
```

**FULL TRACE for `nums = [2,-1,2]`, `k = 3`:**

```
P = [0, 2, 1, 3]      (P[0]=0, P[1]=2, P[2]=2-1=1, P[3]=1+2=3)

i=0, P[0]=0:
  front-check: dq empty, skip.
  back-evict: dq empty, skip.
  push 0.                     dq = [0]              (P values: [0])

i=1, P[1]=2:
  front-check: P[1]-P[dq.front()=0] = 2-0 = 2 >= 3? no.
  back-evict: P[1]=2 <= P[dq.back()=0]=0? no.
  push 1.                     dq = [0, 1]            (P values: [0, 2])

i=2, P[2]=1:
  front-check: P[2]-P[0] = 1-0 = 1 >= 3? no.
  back-evict: P[2]=1 <= P[dq.back()=1]=2? YES, pop.   dq = [0]
              P[2]=1 <= P[dq.back()=0]=0? no. stop.
  push 2.                     dq = [0, 2]            (P values: [0, 1])

i=3, P[3]=3:
  front-check: P[3]-P[0] = 3-0 = 3 >= 3? YES!
    best = min(INF, 3-0) = 3.  pop front.             dq = [2]
    front-check again: P[3]-P[2] = 3-1 = 2 >= 3? no. stop.
  back-evict: P[3]=3 <= P[2]=1? no.
  push 3.                     dq = [2, 3]

RESULT: best = 3   ✓ (the whole array [2,-1,2] is the shortest subarray
                       with sum >= 3; matches LeetCode's expected output)
```

---

## SECTION 8 — TEMPLATE 6: LONGEST SUBARRAY WITH ABSOLUTE DIFF ≤ LIMIT (TWO DEQUES)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1438 — Longest Continuous Subarray With Absolute Diff <= Limit
// Find the longest subarray where max(subarray) - min(subarray) <= limit.
//
// APPROACH: classic two-pointer sliding window, but "is this window
// valid?" requires knowing BOTH the running max AND running min of
// the CURRENT window. Maintain two monotonic deques in parallel:
//   maxDq — decreasing, front = window max
//   minDq — increasing, front = window min
//
// When the window becomes invalid (max - min > limit), shrink from
// the left; if the element leaving the window is the current
// front of EITHER deque, pop that front too.
//
// Time: O(n) amortized — each index pushed/popped from each deque
//       at most once, so O(n) total across both deques.
// Space: O(n) worst case (strictly monotonic input)
// ─────────────────────────────────────────────────────────────────

int longestSubarray(vector<int>& nums, int limit) {
    deque<int> maxDq, minDq;    // both store INDICES
    int left = 0, best = 0;

    for (int right = 0; right < (int)nums.size(); right++) {
        // maintain maxDq: decreasing
        while (!maxDq.empty() && nums[maxDq.back()] <= nums[right]) {
            maxDq.pop_back();
        }
        maxDq.push_back(right);

        // maintain minDq: increasing
        while (!minDq.empty() && nums[minDq.back()] >= nums[right]) {
            minDq.pop_back();
        }
        minDq.push_back(right);

        // shrink the window from the left while it's invalid
        while (nums[maxDq.front()] - nums[minDq.front()] > limit) {
            if (maxDq.front() == left) maxDq.pop_front();
            if (minDq.front() == left) minDq.pop_front();
            left++;
        }

        best = max(best, right - left + 1);
    }

    return best;
}
```

**FULL TRACE for `nums = [8,2,4,7]`, `limit = 4`:**

```
left=0, best=0

right=0, val=8:
  maxDq: empty, push 0.        maxDq=[0](8)
  minDq: empty, push 0.        minDq=[0](8)
  window [0,0]: max=8, min=8, diff=0 <= 4. no shrink.
  best = max(0, 0-0+1) = 1

right=1, val=2:
  maxDq: nums[0]=8 <= 2? no. push 1.      maxDq=[0,1](8,2)
  minDq: nums[0]=8 >= 2? yes, pop.        minDq=[]
         push 1.                          minDq=[1](2)
  window [0,1]: max=nums[0]=8, min=nums[1]=2, diff=6 > 4. SHRINK:
    maxDq.front()=0==left(0)? yes, pop.   maxDq=[1](2)
    minDq.front()=1==left(0)? no.
    left=1.
    recheck: max=nums[1]=2, min=nums[1]=2, diff=0 <= 4. stop shrinking.
  best = max(1, 1-1+1) = 1

right=2, val=4:
  maxDq: nums[1]=2 <= 4? yes, pop.        maxDq=[]
         push 2.                          maxDq=[2](4)
  minDq: nums[1]=2 >= 4? no. push 2.      minDq=[1,2](2,4)
  window [1,2]: max=nums[2]=4, min=nums[1]=2, diff=2 <= 4. no shrink.
  best = max(1, 2-1+1) = 2

right=3, val=7:
  maxDq: nums[2]=4 <= 7? yes, pop.        maxDq=[]
         push 3.                          maxDq=[3](7)
  minDq: nums[2]=4 >= 7? no. push 3.      minDq=[1,2,3](2,4,7)
  window [1,3]: max=nums[3]=7, min=nums[1]=2, diff=5 > 4. SHRINK:
    maxDq.front()=3==left(1)? no.
    minDq.front()=1==left(1)? yes, pop.   minDq=[2,3](4,7)
    left=2.
    recheck: max=nums[3]=7, min=nums[2]=4, diff=3 <= 4. stop shrinking.
  best = max(2, 3-2+1) = 2

RESULT: best = 2   ✓ (matches LeetCode's expected output — e.g. [2,4] or [4,7])
```

---

## SECTION 9 — TEMPLATE 7: MAX VALUE OF EQUATION

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1499 — Max Value of Equation
// points sorted by x. For i < j with xj - xi <= k, maximize:
//     yi + yj + |xi - xj|  =  yi + yj + (xj - xi)     [since xj > xi]
//                          =  (yi - xi) + (yj + xj)
//
// SPLIT INSIGHT: the expression decomposes into a term depending
// ONLY on i (yi - xi) and a term depending ONLY on j (yj + xj).
// For fixed j, we want to maximize (yi - xi) over all valid i < j
// with xj - xi <= k, i.e., xi >= xj - k.
//
// DEQUE INVARIANT: decreasing by (y - x) value, storing indices.
// front = the best (largest yi - xi) among points still in range.
//
// Time: O(n) amortized | Space: O(n) worst case
// ─────────────────────────────────────────────────────────────────

int findMaxValueOfEquation(vector<vector<int>>& points, int k) {
    deque<int> dq;              // indices, decreasing by (y - x)
    int ans = INT_MIN;

    for (int j = 0; j < (int)points.size(); j++) {
        int xj = points[j][0], yj = points[j][1];

        // front-evict: points too far to the left are out of range.
        while (!dq.empty() && xj - points[dq.front()][0] > k) {
            dq.pop_front();
        }

        // answer using the current front (if any valid predecessor exists)
        if (!dq.empty()) {
            int i = dq.front();
            int yi_minus_xi = points[i][1] - points[i][0];
            ans = max(ans, yi_minus_xi + xj + yj);
        }

        // back-evict: dominated (smaller-or-equal y-x, and older -> will
        // expire sooner) entries can never beat the current point again.
        int yj_minus_xj = yj - xj;
        while (!dq.empty() &&
               points[dq.back()][1] - points[dq.back()][0] <= yj_minus_xj) {
            dq.pop_back();
        }
        dq.push_back(j);
    }

    return ans;
}
```

**FULL TRACE for `points = [[1,3],[2,0],[5,10],[6,-10]]`, `k = 1`:**

```
(y-x) values: idx0: 3-1=2, idx1: 0-2=-2, idx2: 10-5=5, idx3: -10-6=-16

j=0 (x=1,y=3):
  front-evict: dq empty.
  no answer (dq empty).
  back-evict: dq empty. push 0.        dq=[0]         (y-x: [2])

j=1 (x=2,y=0):
  front-evict: xj-x[front=0]=2-1=1 > 1? no.
  answer: (y-x)[0]=2 + xj+yj = 2 + 2+0 = 4.  ans=4.
  back-evict: (y-x)[0]=2 <= (y-x)[1]=-2? no.
  push 1.                               dq=[0,1]       (y-x: [2,-2])

j=2 (x=5,y=10):
  front-evict: xj-x[front=0]=5-1=4 > 1? YES, pop.  dq=[1]
               xj-x[front=1]=5-2=3 > 1? YES, pop.  dq=[]
  dq empty -> no answer this round.
  back-evict: dq empty. push 2.         dq=[2]         (y-x: [5])

j=3 (x=6,y=-10):
  front-evict: xj-x[front=2]=6-5=1 > 1? no.
  answer: (y-x)[2]=5 + xj+yj = 5 + 6+(-10) = 1.  ans=max(4,1)=4.
  back-evict: (y-x)[2]=5 <= (y-x)[3]=-16? no.
  push 3.                               dq=[2,3]

RESULT: ans = 4   ✓ (matches LeetCode's expected output, from pair (0,1):
                      3 + 0 + |1-2| = 4)
```

---

## SECTION 10 — VARIANT: MAXIMUM NUMBER OF ROBOTS WITHIN BUDGET

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 2398 — Maximum Number of Robots Within Budget
// Choose the LONGEST contiguous subarray (window) of robots such that
// cost = max(chargeTimes in window) + windowSize * sum(runningCosts
// in window)  <=  budget.
//
// APPROACH: two-pointer sliding window + monotonic max-deque for
// chargeTimes, + a running sum (prefix-sum-style, O(1) update) for
// runningCosts. Expand right; while the window's cost exceeds budget,
// shrink from the left (both the deque AND the running sum must be
// kept consistent with the current [left, right] window).
//
// Time: O(n) amortized — deque work is O(n); sum maintenance is O(1)
//       per step; each robot enters/leaves the window once.
// Space: O(n) worst case for the deque
// ─────────────────────────────────────────────────────────────────

int maximumRobots(vector<int>& chargeTimes, vector<int>& runningCosts,
                   long long budget) {
    int n = chargeTimes.size();
    deque<int> dq;               // indices, decreasing by chargeTimes[]
    long long sumRunning = 0;
    int left = 0, best = 0;

    for (int right = 0; right < n; right++) {
        // maintain max-deque for chargeTimes
        while (!dq.empty() && chargeTimes[dq.back()] <= chargeTimes[right]) {
            dq.pop_back();
        }
        dq.push_back(right);

        sumRunning += runningCosts[right];

        // shrink while the window violates budget
        while (!dq.empty() &&
               (long long)chargeTimes[dq.front()] +
               (long long)(right - left + 1) * sumRunning > budget) {
            if (dq.front() == left) dq.pop_front();
            sumRunning -= runningCosts[left];
            left++;
        }

        best = max(best, right - left + 1);
    }

    return best;
}

// NOTE: This is NOT a "shrink to restore validity then never grow past
// it" two-pointer in the classic monotonic-window-size sense — the
// window size here CAN shrink and regrow non-monotonically across
// iterations of `right`, but since `left` only ever moves forward,
// total work across the whole run is still O(n) amortized.
```

---

## SECTION 11 — WHY IT'S O(n) AMORTIZED (THE PROOF YOU SAY OUT LOUD)

Every monotonic deque algorithm in this pattern follows the same accounting argument used in Pattern 04, extended to two ends:

> "Each index is **pushed** onto the deque exactly once, when it is first processed. Each index can be **popped** at most once from the back (during monotonicity maintenance) and at most once from the front (during expiry eviction) — after it's popped from either end, it's gone forever; it is never pushed again. So across the entire run, total pushes ≤ n, total back-pops ≤ n, total front-pops ≤ n. Total deque operations ≤ 3n = O(n). The inner while loops are NOT O(n) *per outer iteration* — they are O(n) *in total*, across all outer iterations combined."

This is why a solution with a `while` loop nested inside a `for` loop is still linear, not quadratic — the crucial detail interviewers listen for is whether you can articulate *why* the nested loop doesn't multiply the complexity.

For Template 5 (Shortest Subarray, LC 862) specifically: the front-evict loop can run multiple times for a single `i` (each eviction reports an answer), but each front index is evicted at most once total across the whole algorithm, so the same accounting applies.

---

## SECTION 12 — COMPLEXITY TABLE

| Problem | Time | Space | Deque Type | Notes |
|---------|------|-------|-----------|-------|
| Sliding Window Maximum (239) | O(n) | O(k) | Decreasing (max) | Foundation template |
| Sliding Window Minimum | O(n) | O(k) | Increasing (min) | Mirror of above |
| Jump Game VI (1696) | O(n) | O(k) | Decreasing (max), on dp[] | Implicit window = DP recurrence |
| Constrained Subsequence Sum (1425) | O(n) | O(k) | Decreasing (max), clamped at 0 | dp[i] may reset to nums[i] alone |
| Shortest Subarray Sum ≥ K (862) | O(n) | O(n) worst case | Increasing, on prefix sums | Handles negative numbers |
| Longest Subarray, Abs Diff ≤ Limit (1438) | O(n) | O(n) worst case | TWO deques (max + min) | Two-pointer + parallel deques |
| Max Value of Equation (1499) | O(n) | O(n) worst case | Decreasing (max), on `y-x` | Geometric split trick |
| Max Robots Within Budget (2398) | O(n) | O(n) worst case | Decreasing (max) + running sum | Deque + prefix-sum hybrid |

All are **O(n) time**. Space is bounded by the window size `k` when the window is fixed-size (239, 1696, 1425), or by `n` in the worst case when the "window" is content-defined rather than a fixed count (862, 1438, 1499, 2398) — e.g., a strictly monotonic input array can leave every index sitting in the deque simultaneously.

---

## SECTION 13 — VARIANTS AND EXTENSIONS

**Deque-based DP with a MINIMUM instead of maximum:** flip every "decreasing/max" to "increasing/min." Example: minimum-cost variants of Jump Game VI where you want the cheapest path instead of the priciest.

**Circular arrays:** for circular sliding-window-max problems, process the array twice (length `2n`) exactly as you would extend a monotonic stack for a circular Next Greater Element — the deque naturally self-limits to the correct window via the front-evict rule, so duplicated indices beyond `n` never leak incorrect answers into the first pass.

**k varies per query (offline, monotonic k):** if `k` is non-decreasing across queries, you can process all queries with a single deque pass, since the window can only grow — never needs to "un-expire" an already-evicted front. If `k` is arbitrary per query, fall back to a sparse table / segment tree for O(1) or O(log n) range max, since a single deque pass no longer suffices.

**Deque replaced by a multiset / priority queue with lazy deletion:** in interviews, if you can't recall the deque trick under pressure, a `multiset<int>` window (insert `nums[right]`, erase `nums[left]` on slide, `*rbegin()`/`*begin()` for max/min) gives the same *correctness* in O(n log k) instead of O(n) — a valid fallback that shows you understand the problem, at a complexity cost. State this trade-off explicitly if you use it.

**Combining Type 3 (deque+DP) with Type 4 (deque+prefix-sum):** some contest problems require both simultaneously — e.g., "maximize `dp[i] = prefix[i] - prefix[j]` subject to `j` being in a sliding window AND `dp` having its own recurrence." Recognize these as two independent deque instances layered on the same loop, not one combined deque.

---

## SECTION 14 — COMMON MISTAKES

### Mistake 1: Storing values instead of indices

```cpp
// WRONG — deque of VALUES, can't detect expiry
deque<int> dq;
for (int i = 0; i < n; i++) {
    while (!dq.empty() && dq.back() <= nums[i]) dq.pop_back();
    dq.push_back(nums[i]);
    // front-evict is now IMPOSSIBLE — you have no idea which
    // original index the front value came from, so you can't
    // tell whether it has aged out of the window.
    if (i >= k - 1) result.push_back(dq.front());  // WRONG, stale value risk
}

// CORRECT — deque of INDICES, value looked up via nums[dq.front()]
deque<int> dq;
for (int i = 0; i < n; i++) {
    while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
    dq.push_back(i);
    if (dq.front() <= i - k) dq.pop_front();   // expiry check works now
    if (i >= k - 1) result.push_back(nums[dq.front()]);
}
```

This is THE defining bug of this pattern — if you remember only one rule from this document, make it this one.

---

### Mistake 2: Wrong pop condition — `>` vs `>=` around duplicates

```cpp
// WRONG (subtly) — using strict > instead of >= for back-evict on max deque
while (!dq.empty() && nums[dq.back()] < nums[i]) dq.pop_back();
// This keeps EQUAL values in the deque as separate entries. It's not
// incorrect for computing the max itself, but it silently INFLATES the
// deque size in the presence of many duplicates, which can blow past
// your assumed O(k) space bound and, in problems where deque SIZE or
// deque CONTENTS matter beyond just reading the front (like counting
// distinct maxima), gives wrong answers.

// CORRECT — evict on tie, keep only the newest of equal values
while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
```

The rule of thumb: **evict on `<=` for a max deque, evict on `>=` for a min deque** — always let the newer element win ties, since it will remain valid in the window strictly longer.

---

### Mistake 3: Forgetting to evict the expired front — checking BEFORE the window is even full

```cpp
// WRONG — checking expiry with the wrong window boundary
if (dq.front() < i - k) dq.pop_front();   // BUG: should be <=, not <
// For window size k covering indices [i-k+1, i], the valid range's
// lower bound is i-k+1. An index at i-k is ALREADY outside that range.
// Using strict < lets a stale index survive one iteration too long.

// CORRECT
if (dq.front() <= i - k) dq.pop_front();
```

Trace it manually with `k=1`: window is a single element `[i,i]`. At `i=1`, the correct check must evict index 0 (since `0 <= 1-1=0` is true). The buggy `<` version checks `0 < 0`, which is false — index 0 incorrectly survives, and `dq.front()` returns a value from OUTSIDE the size-1 window.

---

### Mistake 4: Not checking `dq.empty()` before reading `dq.front()`/`dq.back()`

```cpp
// WRONG — reads front without confirming the deque is non-empty
dp[i] = nums[i] + dp[dq.front()];   // CRASH if dq is empty (e.g., i < k
                                     // in some formulations, or after an
                                     // over-aggressive front-evict loop)

// CORRECT — guard every read
int bestPrev = dq.empty() ? 0 : dp[dq.front()];
dp[i] = nums[i] + bestPrev;
```

This shows up constantly in the deque+DP templates (1696, 1425) at `i=0`, and in deque+prefix-sum (862) whenever the deque has just been fully drained by a run of successful front-evictions.

---

### Mistake 5: Evicting from the back BEFORE checking the front in prefix-sum problems

```cpp
// WRONG order — for LC 862, doing back-evict before front-check
for (int i = 0; i <= n; i++) {
    while (!dq.empty() && P[i] <= P[dq.back()]) dq.pop_back();  // back FIRST
    while (!dq.empty() && P[i] - P[dq.front()] >= k) {          // front SECOND
        best = min(best, i - dq.front());
        dq.pop_front();
    }
    dq.push_back(i);
}
// This is subtly WRONG: if P[i] would have evicted the CURRENT back via
// the back-evict rule, and that back index was ALSO a valid front-check
// candidate (P[i] - P[back] >= k), doing back-evict first silently
// discards that index WITHOUT ever recording it as an answer — you lose
// a potential (and possibly the ONLY) valid answer for this i.

// CORRECT order — ALWAYS front-check/evict first, THEN back-evict
for (int i = 0; i <= n; i++) {
    while (!dq.empty() && P[i] - P[dq.front()] >= k) {   // front FIRST
        best = min(best, i - dq.front());
        dq.pop_front();
    }
    while (!dq.empty() && P[i] <= P[dq.back()]) dq.pop_back();  // back SECOND
    dq.push_back(i);
}
```

General rule: **whenever a single index could simultaneously be "answerable" via the front-check AND "dominated" via the back-evict rule, always resolve the front (answer-producing) logic first.** This ordering issue does not arise in the pure sliding-window-max/min templates (239, minimum) because there the front-evict is about *expiry*, not about *answer extraction* — but it is very real for 862's front-evict, which doubles as the answer-reporting step.

---

### Mistake 6: Confusing "monotonic deque" with "monotonic stack that happens to use `deque<int>`"

```cpp
// WRONG — using deque<int> as a drop-in replacement for stack<int>,
// never popping from the front, on a problem that actually needs a
// window (e.g., mistakenly applying the NGE template to LC 239)
deque<int> dq;
for (int i = 0; i < n; i++) {
    while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
    dq.push_back(i);
    // MISSING: no front-evict logic at all — this degrades silently
    // into an unbounded monotonic stack; window semantics are lost.
}

// CORRECT — front-evict is what MAKES it a deque problem, not optional
```

If your monotonic-deque solution never calls `pop_front()`, you have not written a monotonic deque solution — you've written a monotonic stack solution using the wrong container. That's a strong signal to re-read the problem for a window constraint you missed.

---

## SECTION 15 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Sliding Window Maximum | 239 | Deque of indices, decreasing, front/back eviction |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 2 | Jump Game VI | 1696 | Deque + DP, window is the DP recurrence itself |
| 3 | Constrained Subsequence Sum | 1425 | Deque + DP with clamp-at-0 ("start fresh") logic |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 4 | Shortest Subarray with Sum at Least K | 862 | Deque over prefix sums, handles negatives |
| 5 | Longest Continuous Subarray With Abs Diff ≤ Limit | 1438 | TWO parallel deques (max + min) with two pointers |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Max Value of Equation | 1499 | Algebraic split into i-term / j-term before deque design |
| 7 | Maximum Number of Robots Within Budget | 2398 | Deque + running-sum hybrid, non-monotonic window size |

---

## SECTION 16 — PATTERN CONNECTIONS

1. **Pattern 03 (Sliding Window):** the monotonic deque is the natural upgrade path the moment a sliding-window problem asks for the max/min inside the window rather than a sum or count. Two-pointer sliding window (P03) tells you WHERE the window is; the deque tells you WHAT is the extreme value inside it, both in O(n) total.

2. **Pattern 04 (Monotonic Stack):** the deque is a monotonic stack with a second exit. Every intuition you built for NGE/NSE/PGE/NSE transfers directly to the back-evict half of a deque. The only genuinely new skill is the front-evict for window expiry. If Pattern 04's Largest Rectangle in Histogram made sense to you, Sliding Window Maximum should take under 10 minutes to internalize.

3. **Pattern 17 (1D DP):** Templates 3 and 4 in this document (Jump Game VI, Constrained Subsequence Sum) are 1D DP problems whose naive transition is O(nk) — an O(n) scan for the max/min of the last k dp values, repeated for each of n states. The monotonic deque is the optimization that turns an O(nk) DP into O(n) DP, the exact same role a sparse table or segment tree plays for range-query-optimized DP, but simpler and faster when the query is specifically "max/min over a SLIDING window of the DP array."

4. **Sparse Table / Segment Tree (advanced range query):** for arbitrary (non-sliding) range max/min queries, or when window sizes are not monotonic across queries, a sparse table (O(1) query, O(n log n) build, static array) or segment tree (O(log n) query, supports updates) replaces the deque. Recognize the deque as the *specialized, faster* tool for the *sliding* case — reach for the general tool only when the window doesn't behave nicely.

---

## SECTION 17 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 239 — Sliding Window Maximum

**Interviewer:** "Given an array and window size k, return the max of every window as it slides across the array."

**Candidate:**

*[First establish why brute force fails, then pivot]*
> "Brute force recomputes the max for every window in O(k), giving O(nk) total. To get O(n), I need the max of the current window available without rescanning it every time I slide. I'll use a monotonic deque of indices, kept in decreasing order of value — the front is always the current max."

*[State both eviction rules before coding]*
> "Two eviction rules: back-evict — pop any index whose value is ≤ the incoming value, since it's now dominated by a newer, at-least-as-large element and can never be a future window's max. Front-evict — pop the front if its index has fallen outside the current window, i.e., `front <= i - k`."

*[Code — 5 minutes]*
```cpp
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;
    vector<int> result;
    for (int i = 0; i < (int)nums.size(); i++) {
        while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
        dq.push_back(i);
        if (dq.front() <= i - k) dq.pop_front();
        if (i >= k - 1) result.push_back(nums[dq.front()]);
    }
    return result;
}
```

*[Follow-up]: "Why does this stay O(n) despite the nested while loop?"*
> "Each index is pushed exactly once and popped at most once total, across the whole run — either by the back-evict rule or by the front-evict rule, never both, since once popped it's gone. Total operations are bounded by 3n, so it's O(n) amortized, not O(nk)."

**RED FLAGS:**
- Storing values instead of indices (breaks front-evict entirely)
- Using a `multiset` without acknowledging the O(n log k) cost difference — acceptable as a fallback, but state the tradeoff
- Checking `i >= k` instead of `i >= k - 1` for when to start recording output (off-by-one on window fullness)

---

### Interview Simulation 2: LC 1696 — Jump Game VI

**Interviewer:** "From index i you may jump to any index in [i+1, i+k]. Maximize the sum of visited values from index 0 to n-1."

**Candidate:**

*[Identify the DP first, THEN spot the optimization]*
> "This is `dp[i] = nums[i] + max(dp[i-k..i-1])`. Naively, computing that max is an O(k) scan per index, giving O(nk) overall. But 'max of the last k values of an array I'm building left to right' is structurally identical to sliding window maximum — except the array being windowed is `dp`, not `nums`. I'll run a decreasing monotonic deque over dp indices."

*[Explain the ordering: front-evict BEFORE using the front, back-evict AFTER computing dp[i]]*
> "For each i: first evict any deque front outside `[i-k, i-1]` — I need a valid predecessor before I can compute `dp[i]`. Then compute `dp[i] = nums[i] + dp[dq.front()]`. Only after `dp[i]` is known can I decide whether to evict dominated entries from the back and push `i`."

*[Code — 6 minutes, same as Template 3 above]*

*[Interviewer]: "What if k could be as large as n? Does this still work?"*
> "Yes — the deque naturally bounds itself to at most k+1 relevant entries at any time regardless of how large k is, because entries with dp <= a newer entry's dp are evicted from the back immediately. Worst case space is O(min(n, k))."

**RED FLAGS:**
- Attempting a naive O(nk) scan for the max without recognizing the deque optimization
- Reading `dq.front()` before performing the front-evict for the current index
- Forgetting `dp[0] = nums[0]` as the base case, or trying to push index 0 through the same loop body without a base case

---

### Interview Simulation 3: Meta-discussion — Recognizing when you need a deque instead of a stack

**Interviewer:** "How do you decide, mid-problem, that a monotonic stack isn't enough and you need a monotonic deque?"

**Candidate:**
> "I look for a boundedness constraint on how far back the comparison window reaches. A monotonic stack answers 'what's the nearest larger/smaller element considering everything I've seen so far' — unbounded history. The moment the problem adds a phrase like 'within the last k elements,' 'within a window of size k,' or has a DP recurrence that only looks back k steps — `dp[i]` depends on `dp[i-k..i-1]` — that's a bounded window, and a stack has no mechanism to expire old entries. I need the front-pop capability of a deque.

> The other signal is prefix-sum-based problems with negative numbers, like Shortest Subarray with Sum at Least K. There, the 'window' isn't a count of elements at all — it's implicitly defined by which prefix sums remain useful candidates, and both ends of the deque get evicted for different reasons: back for domination, front because a shorter answer was just found and further growth from that same front can't improve on it."

**RED FLAGS:**
- Reaching for a segment tree or sparse table by default for every range-max problem without first checking whether the window is a simple sliding one (over-engineering)
- Not being able to state, out loud, the specific condition under which the front gets evicted vs. the back

---

## SECTION 18 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: What is the single structural difference between a monotonic stack and a monotonic deque?**

> A monotonic deque can pop from BOTH ends. The back-pop maintains the monotonic invariant (same job the stack's pop does — evict dominated elements). The front-pop is new: it evicts elements that have fallen outside the current window (expired), which a stack structurally cannot do without destroying everything pushed after it.

---

**Q2: In Sliding Window Maximum, why must indices be stored instead of values?**

> Expiry detection requires knowing WHERE an element came from. `dq.front() <= i - k` is a check on the element's original position relative to the current window boundary. If only the value were stored, there would be no way to tell whether the front element is still within the last k positions or has aged out — two equal values from different positions would be indistinguishable.

---

**Q3: For Jump Game VI, why is the front-evict performed BEFORE computing dp[i], but the back-evict performed AFTER?**

> The front-evict is a validity check: `dp[i]` needs a legitimate predecessor within `[i-k, i-1]`, so stale (out-of-range) entries must be removed before reading `dq.front()`. The back-evict is a maintenance step for FUTURE queries: it decides whether `dp[i]` dominates older entries, which requires `dp[i]`'s value to already be known — impossible before it's computed.

---

**Q4: In Shortest Subarray with Sum at Least K, why do we evict the front the moment `P[i] - P[front] >= K`, rather than waiting to see if a later index gives an even shorter answer?**

> Because widening the window from the SAME front index only ever increases `i - front` (later `i` values are farther from a fixed front). Once the condition is satisfied at the current `i`, that front index has already produced its best (shortest) possible answer — any future `i' > i` paired with this same front would only be longer. The front is provably useless going forward and can be safely discarded, while we continue checking the new front against the same `i` in case multiple fronts qualify at once.

---

**Q5: Why does Longest Subarray With Abs Diff ≤ Limit need TWO deques instead of one?**

> The window's validity depends on BOTH the running maximum and the running minimum simultaneously (`max - min <= limit`). A single deque can only track one extreme. Running a max-deque and a min-deque in parallel over the same two-pointer window lets you query both extremes in O(1) at every step, while both deques independently maintain O(n) amortized total work.

---

**Q6: In Max Value of Equation, what algebraic step had to happen BEFORE any deque could be designed?**

> The equation `yi + yj + |xi - xj|` had to be rewritten as `(yi - xi) + (yj + xj)` (using `xj > xi` to drop the absolute value), splitting it into a term depending only on the earlier point `i` and a term depending only on the later point `j`. Only after this split does it become clear that maintaining a decreasing deque on the `(y - x)` term of past points, queried when processing each new `j`, solves the problem — the deque design falls out of the algebra, not the other way around.

---

**Q7: True or false: a monotonic deque's front is always read but never itself the target of the back-evict comparison, and vice versa. Explain why keeping these two operations conceptually separate matters.**

> True. Back-evict compares the INCOMING new element against existing BACK elements to decide who is dominated — it has nothing to do with window boundaries. Front-evict compares the FRONT element's stored index/position against the current position to decide if it has expired — it has nothing to do with value comparisons. Conflating them (e.g., writing one unified "pop if worse" condition) is a common source of bugs, especially in problems like LC 862 where BOTH conditions can fire for the same index in the same iteration, and the ORDER in which you check them changes correctness (see Common Mistake 5).

---

## SECTION 19 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. deque<T> OPERATIONS
// ─────────────────────────────────────────────────────────────────
#include <deque>
deque<int> dq;

dq.push_back(x);    dq.push_front(x);   // insert — O(1)
dq.pop_back();       dq.pop_front();     // remove — O(1), returns void
dq.back();            dq.front();         // peek — O(1), does NOT remove
dq.empty();  dq.size();                  // O(1)

// CRITICAL: pop_back()/pop_front() return void.
int idx = dq.back(); dq.pop_back();      // always two steps

// ─────────────────────────────────────────────────────────────────
// 2. WINDOW MAXIMUM (decreasing deque, front = max)
// ─────────────────────────────────────────────────────────────────
while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
dq.push_back(i);
if (dq.front() <= i - k) dq.pop_front();
// current window max = nums[dq.front()]

// ─────────────────────────────────────────────────────────────────
// 3. WINDOW MINIMUM (increasing deque, front = min)
// ─────────────────────────────────────────────────────────────────
while (!dq.empty() && nums[dq.back()] >= nums[i]) dq.pop_back();
dq.push_back(i);
if (dq.front() <= i - k) dq.pop_front();
// current window min = nums[dq.front()]

// ─────────────────────────────────────────────────────────────────
// 4. DEQUE + DP (implicit window = DP recurrence, LC 1696 / 1425)
// ─────────────────────────────────────────────────────────────────
// front-evict FIRST (need a valid predecessor before computing dp[i])
while (!dq.empty() && dq.front() < i - k) dq.pop_front();
dp[i] = nums[i] + /* transform of */ dp[dq.front()];
// back-evict SECOND (dp[i] now known, can compare)
while (!dq.empty() && dp[dq.back()] <= dp[i]) dq.pop_back();
dq.push_back(i);

// ─────────────────────────────────────────────────────────────────
// 5. DEQUE + PREFIX SUM (LC 862) — front-check BEFORE back-evict
// ─────────────────────────────────────────────────────────────────
while (!dq.empty() && P[i] - P[dq.front()] >= K) {
    best = min(best, i - dq.front());
    dq.pop_front();
}
while (!dq.empty() && P[i] <= P[dq.back()]) dq.pop_back();
dq.push_back(i);

// ─────────────────────────────────────────────────────────────────
// 6. TWO DEQUES IN PARALLEL (LC 1438)
// ─────────────────────────────────────────────────────────────────
while (!maxDq.empty() && nums[maxDq.back()] <= nums[right]) maxDq.pop_back();
maxDq.push_back(right);
while (!minDq.empty() && nums[minDq.back()] >= nums[right]) minDq.pop_back();
minDq.push_back(right);
while (nums[maxDq.front()] - nums[minDq.front()] > limit) {
    if (maxDq.front() == left) maxDq.pop_front();
    if (minDq.front() == left) minDq.pop_front();
    left++;
}

// ─────────────────────────────────────────────────────────────────
// 7. DECISION TABLE
// ─────────────────────────────────────────────────────────────────
// Fixed-size window, need max/min:        Template 1 / 2
// DP recurrence with a lookback of k:      Template 3 / 4
// "subarray sum >= K", negatives allowed:  Template 5 (prefix sum)
// Need BOTH max and min of same window:    Template 6 (two deques)
// Objective splits into f(i) + g(j):       Template 7 (geometric split)

// ─────────────────────────────────────────────────────────────────
// 8. THE TWO GOLDEN RULES
// ─────────────────────────────────────────────────────────────────
// (a) ALWAYS store indices, never values.
// (b) When both front-check and back-evict could apply to the same
//     step, resolve FRONT first if the front-check produces an answer
//     (LC 862 style); order doesn't matter if front-evict is pure
//     expiry with no side effect (LC 239 style) — but front-first is
//     always a safe default.
```

---

## SECTION 20 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 7 problems in order.** Template 1 (LC 239) must become completely automatic — under 8 minutes, written cold, correct on the first submission. Everything else in this pattern builds on that muscle memory.

2. **Do the "spot the implicit window" drill:** for any 1D DP problem you've solved before with an O(n²) or O(nk) transition, ask "does `dp[i]` depend on a bounded RANGE of previous dp values (`dp[i-k..i-1]`), and does that range slide forward by exactly one as `i` increases?" If yes, it is very likely deque-optimizable. Jump Game VI and Constrained Subsequence Sum are the canonical drills for this.

3. **Re-derive LC 862 from scratch without looking at the trace.** The ordering bug in Common Mistake 5 (front-check before back-evict) is subtle enough that most people get it wrong the first time even after reading the correct code. Write it, delete it, write it again a day later.

4. **Compare the deque solution against a `multiset`-based O(n log k) fallback** for LC 239. Time both mentally: understand that in an interview, offering the multiset as a first correct pass, then optimizing to the deque, is a strong signal of engineering maturity — ship correct, then ship fast.

5. **Revisit Pattern 04 (Monotonic Stack)** and specifically re-solve Largest Rectangle in Histogram. Notice that its "width" computation (`i - st.top() - 1`) plays a conceptually similar role to a deque's front-evict boundary check — both are about knowing exactly how far a candidate's influence extends.

---

## SECTION 21 — SIGN-OFF CRITERIA

### Tier 1 — Foundation mastered
You solve Sliding Window Maximum (LC 239) cold, in under 10 minutes, with indices (not values) in the deque, and can state the O(n) amortized argument out loud in under 30 seconds. You can also write Sliding Window Minimum as a pure mirror-image without re-deriving it from scratch.

### Tier 2 — Deque + DP internalized
You solve Jump Game VI (LC 1696) and correctly identify, unprompted, that the "window" is over the DP array, not the input array. You solve Constrained Subsequence Sum (LC 1425) and can explain why clamping the predecessor's contribution at 0 is necessary (starting fresh is always a legal option).

### Tier 3 — Prefix-sum and dual-deque variants solid
You solve Shortest Subarray with Sum at Least K (LC 862) with the correct front-check-before-back-evict ordering, and can produce a failing example (negative numbers present) showing why a plain two-pointer sliding window is insufficient here. You solve Longest Subarray With Abs Diff ≤ Limit (LC 1438) using two parallel deques without conflating their eviction conditions.

### Tier 4 — Contest-level variants complete
You solve Max Value of Equation (LC 1499), performing the algebraic split (`yi + yj + |xi-xj|` → `(yi-xi) + (yj+xj)`) yourself before writing any code. You solve Maximum Number of Robots Within Budget (LC 2398) and can explain why the window size is allowed to be non-monotonic across iterations while the algorithm remains O(n) amortized overall.

**Sign-off threshold:** Solve 5 of 7 problems. Mandatory: LC 239 (Sliding Window Maximum — index-based deque, both eviction rules explained), LC 1696 (Jump Game VI — implicit window recognition), and LC 862 (Shortest Subarray with Sum at Least K — front-before-back ordering). These three represent the foundation, the DP-optimization leap, and the trickiest correctness pitfall in this pattern, respectively.

---

*Pattern 29 Complete — Monotonic Deque*
*Connects back to: Pattern 03 (Sliding Window), Pattern 04 (Monotonic Stack), Pattern 17 (1D DP)*
