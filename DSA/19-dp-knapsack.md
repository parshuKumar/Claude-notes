# PATTERN 19 — KNAPSACK FAMILY
## 0/1, Unbounded, Subset-Sum, Count-Ways, 2D Knapsack — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**Knapsack is not one pattern. It is a state-transition SKELETON that gets reused across ~15 different-looking LeetCode problems.**

The 0/1 knapsack itself — "n items, each with weight and value, maximize value under capacity W" — is rarely asked directly anymore. Every interviewer knows it's a textbook problem. What actually gets asked is a **disguise**: a problem that has nothing to do with weights or values on the surface, but reduces to exactly the same `dp[capacity]` recurrence once you see it.

**The three disguises that separate 1800-rated candidates from 2000-rated ones:**

1. **Partition Equal Subset Sum (LC 416)** — no "items" or "values" mentioned at all. You must notice: "split into two equal-sum subsets" = "does a subset exist that sums to `total/2`" = 0/1 knapsack with **capacity = total/2** and **every item's value equals its weight**.

2. **Target Sum (LC 494)** — looks like a plain-vanilla backtracking/DFS problem ("assign + or − to each number"). The insight that turns O(2^n) into O(n·target) is an **algebraic transform**: if `P` = sum of numbers assigned `+` and `N` = sum assigned `−`, then `P − N = target` and `P + N = totalSum`. Solving gives `P = (target + totalSum) / 2`. Now it's subset-sum counting — 0/1 knapsack.

3. **Tallest Billboard (LC 956)** — the hardest of the family. There is no "capacity" at all in the problem statement. The trick is to knapsack on the **difference between two piles** rather than on a sum, with the state being `dp[diff] = max achievable height of the SHORTER pile`. Nobody derives this cold in an interview without having seen "knapsack on difference" before.

**The single most consequential bug in this entire pattern:**

When you space-optimize a 2D knapsack DP to 1D, the direction of the capacity loop (`for w = W downto weight[i]` vs `for w = weight[i] to W`) silently switches the semantics between **0/1 knapsack** (each item used once) and **unbounded knapsack** (each item used infinitely). Get the direction backward and your code still compiles, still runs, and gives a plausible-looking WRONG answer. This is the #1 knapsack bug at every rating level, including grandmasters who are tired or rushing.

**Second most consequential bug:** swapping the two loop nests in a "count ways" knapsack silently switches between **combinations** (order doesn't matter — Coin Change II) and **permutations** (order matters — Combination Sum IV). Both compile. Both look reasonable. Only one is correct for the problem asked.

**The key skill this pattern builds:**

Before writing a single line of knapsack code, answer:
1. Is each item usable **once** (0/1) or **unlimited times** (unbounded)?
2. Am I optimizing a **value** (max/min) or **counting configurations** (number of ways)?
3. If counting: does order matter (permutations) or not (combinations)?
4. Is there a **non-obvious transform** hiding a knapsack inside a problem that doesn't mention weights/values at all?

Get these four answers right and the loop skeleton writes itself — because there are only four skeletons in the entire family, and every problem in this document maps onto exactly one of them.

---

## SECTION 2 — THE KNAPSACK TAXONOMY

### Type 1: 0/1 Knapsack (each item used at most once)
State: `dp[i][w]` = best value using first `i` items with capacity `w`
Transition: `dp[i][w] = max(dp[i-1][w], dp[i-1][w - wt[i]] + val[i])`
Space-optimized loop direction: capacity **descending** (`W downto wt[i]`)
Problems: 0/1 Knapsack, Partition Equal Subset Sum, Target Sum, Last Stone Weight II, Ones and Zeroes

### Type 2: Unbounded Knapsack (each item used unlimited times)
State: `dp[w]` = best value/count achievable with capacity exactly/at-most `w`
Transition: `dp[w] = best(dp[w], dp[w - wt[i]] + val[i])`
Space-optimized loop direction: capacity **ascending** (`wt[i] to W`) — this is what allows reuse
Problems: Coin Change, Coin Change II, Combination Sum IV, Perfect Squares, Minimum Cost to Fill Given Weight

### Type 3: Bounded Knapsack (each item used up to `k[i]` times)
State: same as 0/1, but each item is either (a) exploded into `k[i]` copies and run through 0/1 knapsack, or (b) split into binary-power groups (`1, 2, 4, ..., k[i] - 2^m+1` copies) to get from O(n·W·k) to O(n·W·log k).
Rare at this rating band — mentioned for completeness, not templated in depth here.

### Type 4: Subset-Sum (boolean reachability — a degenerate 0/1 knapsack)
State: `dp[w] = true/false` — can capacity `w` be reached exactly using a subset of items?
Transition: `dp[w] = dp[w] || dp[w - wt[i]]`
Problems: Partition Equal Subset Sum, Last Stone Weight II (via reachable-sums), Target Sum (boolean version)

### Type 5: Count-Ways Knapsack (counting, not optimizing)
State: `dp[w] = number of ways to reach capacity w`
Transition: `dp[w] += dp[w - wt[i]]`
**The loop-order fork:**
- Items outer, capacity inner → **combinations** (Coin Change II)
- Capacity outer, items inner → **permutations** (Combination Sum IV)
This is the single most interview-tested subtlety in the whole pattern (Section 8).

### Type 6: The Disguise (algebraic transform reveals a knapsack)
No weights/values in the problem statement. You must derive them.
- Target Sum: `P - N = target`, `P + N = total` → `P = (target+total)/2` → subset-sum count
- Partition Equal Subset Sum: "two equal halves" → subset-sum reachability at `total/2`
- Tallest Billboard: no sum target at all → knapsack keyed on **difference**, not sum
- Profitable Schemes: adds a **second dimension** (group size) on top of the profit dimension → 3D knapsack

### Type 7: 2D Knapsack (two independent capacity constraints)
State: `dp[i][j]` = best value using at most `i` of resource A and `j` of resource B
Transition: `dp[i][j] = max(dp[i][j], dp[i - costA][j - costB] + 1)`
Problems: Ones and Zeroes (LC 474), Profitable Schemes (LC 879, adds a 3rd dimension)

### Type 8: Space Optimization Rules (memorize this table — it IS the pattern)

| Knapsack type | Capacity loop direction | Why |
|---|---|---|
| 0/1 (1 use per item) | **descending** (W → wt[i]) | prevents reading an already-updated (this row's) cell — each item contributes at most once |
| Unbounded (∞ uses) | **ascending** (wt[i] → W) | lets `dp[w]` read a value that already includes the current item — enables reuse |
| Count ways, combinations | items outer, capacity inner (ascending) | fixes the order coins are considered, collapsing permutations into one combination |
| Count ways, permutations | capacity outer, items inner (ascending) | re-considers every item at every capacity, so every ordering is counted separately |

---

## SECTION 3 — TEMPLATE 1: 0/1 KNAPSACK (2D FOUNDATION)

```cpp
// ─────────────────────────────────────────────────────────────────
// Classic 0/1 Knapsack
// n items, each with weight wt[i] and value val[i]. Capacity W.
// Each item can be taken at most once. Maximize total value.
//
// State:      dp[i][w] = max value achievable using the first i items
//                        with capacity exactly-or-at-most w
// Transition: dp[i][w] = max(
//                 dp[i-1][w],                              // skip item i
//                 dp[i-1][w - wt[i-1]] + val[i-1]           // take item i (if it fits)
//             )
// Base case:  dp[0][w] = 0 for all w (no items → 0 value)
// Answer:     dp[n][W]
// ─────────────────────────────────────────────────────────────────

long long knapsack01(vector<long long>& wt, vector<long long>& val, long long W) {
    int n = wt.size();
    // dp[i][w]: rows = items considered so far, cols = capacity
    vector<vector<long long>> dp(n + 1, vector<long long>(W + 1, 0));

    for (int i = 1; i <= n; i++) {
        for (long long w = 0; w <= W; w++) {
            // Option 1: don't take item i (always valid)
            dp[i][w] = dp[i-1][w];

            // Option 2: take item i (only if it fits in capacity w)
            if (wt[i-1] <= w) {
                dp[i][w] = max(dp[i][w], dp[i-1][w - wt[i-1]] + val[i-1]);
            }
        }
    }

    return dp[n][W];
}
```

**FULL TRACE for wt=[1,3,4,5], val=[1,4,5,7], W=7 (classic textbook instance):**
```
dp[0][*] = [0,0,0,0,0,0,0,0]   (no items, all capacities → 0)

i=1 (item wt=1,val=1): dp[1][w] = max(dp[0][w], dp[0][w-1]+1) for w>=1
  dp[1] = [0,1,1,1,1,1,1,1]

i=2 (item wt=3,val=4): dp[2][w] = max(dp[1][w], dp[1][w-3]+4) for w>=3
  w=0..2 unchanged: [0,1,1]
  w=3: max(dp[1][3]=1, dp[1][0]+4=4) = 4
  w=4: max(dp[1][4]=1, dp[1][1]+4=5) = 5
  w=5: max(dp[1][5]=1, dp[1][2]+4=5) = 5
  w=6: max(dp[1][6]=1, dp[1][3]+4=5) = 5
  w=7: max(dp[1][7]=1, dp[1][4]+4=5) = 5
  dp[2] = [0,1,1,4,5,5,5,5]

i=3 (item wt=4,val=5): dp[3][w] = max(dp[2][w], dp[2][w-4]+5) for w>=4
  w=0..3 unchanged: [0,1,1,4]
  w=4: max(dp[2][4]=5, dp[2][0]+5=5) = 5
  w=5: max(dp[2][5]=5, dp[2][1]+5=6) = 6
  w=6: max(dp[2][6]=5, dp[2][2]+5=6) = 6
  w=7: max(dp[2][7]=5, dp[2][3]+5=9) = 9
  dp[3] = [0,1,1,4,5,6,6,9]

i=4 (item wt=5,val=7): dp[4][w] = max(dp[3][w], dp[3][w-5]+7) for w>=5
  w=0..4 unchanged: [0,1,1,4,5]
  w=5: max(dp[3][5]=6, dp[3][0]+7=7) = 7
  w=6: max(dp[3][6]=6, dp[3][1]+7=8) = 8
  w=7: max(dp[3][7]=9, dp[3][2]+7=8) = 9
  dp[4] = [0,1,1,4,5,7,8,9]

Answer: dp[4][7] = 9  (take item 2 (wt3,val4) + item 3 (wt4,val5): weight 7, value 9) ✓
```

**Reconstructing the chosen items** (a common follow-up): walk backward from `dp[n][W]`. If `dp[i][w] != dp[i-1][w]`, item `i` was taken — subtract `wt[i-1]` from `w`, move to row `i-1`. Otherwise move to row `i-1` at the same `w`.

---

## SECTION 4 — TEMPLATE 2: 0/1 KNAPSACK SPACE-OPTIMIZED (1D BACKWARD ITERATION)

This is where the single most common knapsack bug lives. Read the loop direction argument carefully — it recurs in every 0/1 knapsack disguise in this document.

```cpp
// ─────────────────────────────────────────────────────────────────
// Same problem, O(W) space instead of O(nW).
//
// KEY INSIGHT: dp[i][w] only ever reads dp[i-1][...] — the PREVIOUS row.
// If we collapse to one array dp[w] and iterate w DESCENDING, then when
// we compute dp[w] we read dp[w - wt[i-1]], which — because w - wt[i-1]
// is SMALLER than w and hasn't been touched yet this iteration — still
// holds the value from the previous item (row i-1). That's exactly what
// the 2D recurrence needs.
//
// If we iterated w ASCENDING instead, dp[w - wt[i-1]] would already have
// been updated THIS iteration (i.e., it would include item i-1 again),
// silently turning this into an UNBOUNDED knapsack. Same code, different
// bug, wrong answer that still looks plausible.
// ─────────────────────────────────────────────────────────────────

long long knapsack01_1D(vector<long long>& wt, vector<long long>& val, long long W) {
    int n = wt.size();
    vector<long long> dp(W + 1, 0);

    for (int i = 0; i < n; i++) {
        // MUST go from W down to wt[i] — descending — for 0/1 semantics
        for (long long w = W; w >= wt[i]; w--) {
            dp[w] = max(dp[w], dp[w - wt[i]] + val[i]);
        }
    }

    return dp[W];
}
```

**TRACE for wt=[1,3,4,5], val=[1,4,5,7], W=7 (same instance as Section 3):**
```
dp = [0,0,0,0,0,0,0,0]

item0 (wt=1,val=1): w=7..1 (descending)
  w=7: dp[7]=max(0,dp[6]+1)=max(0,0+1)=1
  w=6: dp[6]=max(0,dp[5]+1)=1
  w=5: dp[5]=max(0,dp[4]+1)=1
  w=4: dp[4]=max(0,dp[3]+1)=1
  w=3: dp[3]=max(0,dp[2]+1)=1
  w=2: dp[2]=max(0,dp[1]+1)=1
  w=1: dp[1]=max(0,dp[0]+1)=1
  dp = [0,1,1,1,1,1,1,1]   ← matches dp[1] row from Section 3 exactly

item1 (wt=3,val=4): w=7..3
  w=7: dp[7]=max(1, dp[4]+4=1+4=5)=5
  w=6: dp[6]=max(1, dp[3]+4=1+4=5)=5
  w=5: dp[5]=max(1, dp[2]+4=1+4=5)=5
  w=4: dp[4]=max(1, dp[1]+4=1+4=5)=5
  w=3: dp[3]=max(1, dp[0]+4=0+4=4)=4
  dp = [0,1,1,4,5,5,5,5]   ← matches dp[2] row from Section 3 exactly

item2 (wt=4,val=5): w=7..4
  w=7: dp[7]=max(5, dp[3]+5=4+5=9)=9
  w=6: dp[6]=max(5, dp[2]+5=1+5=6)=6
  w=5: dp[5]=max(5, dp[1]+5=1+5=6)=6
  w=4: dp[4]=max(5, dp[0]+5=0+5=5)=5
  dp = [0,1,1,4,5,6,6,9]   ← matches dp[3] row from Section 3 exactly

item3 (wt=5,val=7): w=7..5
  w=7: dp[7]=max(9, dp[2]+7=1+7=8)=9
  w=6: dp[6]=max(6, dp[1]+7=1+7=8)=8
  w=5: dp[5]=max(6, dp[0]+7=0+7=7)=7
  dp = [0,1,1,4,5,7,8,9]   ← matches dp[4] row from Section 3 exactly

Answer: dp[7] = 9 ✓  (identical result, O(W) space)
```

**Proof this is safe:** at the moment `dp[w]` is updated with item `i`, every index `< w` still holds its pre-item-`i` value because we haven't visited it yet this pass (we're going downward). So `dp[w - wt[i]]` is guaranteed to be the value from *before* item `i` was considered — exactly `dp[i-1][w-wt[i]]` in 2D terms.

---

## SECTION 5 — TEMPLATE 3: UNBOUNDED KNAPSACK (COIN CHANGE — MIN COINS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 322 — Coin Change
// Given coin denominations (unlimited supply of each) and a target
// amount, find the minimum number of coins to make that amount.
// Return -1 if impossible.
//
// State:      dp[a] = minimum coins to make amount a
// Transition: dp[a] = min(dp[a], dp[a - coin] + 1) for every coin
// Base case:  dp[0] = 0 (zero coins needed for amount 0)
// Answer:     dp[amount], or -1 if it's still INF
//
// LOOP DIRECTION: ascending. Because each coin has UNLIMITED supply,
// dp[a - coin] is ALLOWED to already include this same coin type —
// that's exactly what "unbounded" means. Ascending order guarantees
// dp[a - coin] may have already been updated with coin THIS pass,
// which is required to build amounts using many copies of one coin.
// ─────────────────────────────────────────────────────────────────

int coinChange(vector<int>& coins, int amount) {
    const int INF = INT_MAX / 2;
    vector<int> dp(amount + 1, INF);
    dp[0] = 0;

    for (int coin : coins) {
        // MUST go ascending — coin can be reused within the same pass
        for (int a = coin; a <= amount; a++) {
            dp[a] = min(dp[a], dp[a - coin] + 1);
        }
    }

    return dp[amount] >= INF ? -1 : dp[amount];
}
```

**TRACE for coins=[1,2,5], amount=11:**
```
dp[0..11] = [0,INF,INF,INF,INF,INF,INF,INF,INF,INF,INF,INF]

coin=1: a=1..11, dp[a]=min(dp[a], dp[a-1]+1)
  Every amount reachable using only 1-coins: dp = [0,1,2,3,4,5,6,7,8,9,10,11]

coin=2: a=2..11, dp[a]=min(dp[a], dp[a-2]+1)
  a=2: min(2, dp[0]+1=1)=1
  a=3: min(3, dp[1]+1=2)=2
  a=4: min(4, dp[2]+1=2)=2
  a=5: min(5, dp[3]+1=3)=3
  a=6: min(6, dp[4]+1=3)=3
  a=7: min(7, dp[5]+1=4)=4
  a=8: min(8, dp[6]+1=4)=4
  a=9: min(9, dp[7]+1=5)=5
  a=10: min(10, dp[8]+1=5)=5
  a=11: min(11, dp[9]+1=6)=6
  dp = [0,1,1,2,2,3,3,4,4,5,5,6]

coin=5: a=5..11, dp[a]=min(dp[a], dp[a-5]+1)
  a=5: min(3, dp[0]+1=1)=1
  a=6: min(3, dp[1]+1=2)=2
  a=7: min(4, dp[2]+1=2)=2
  a=8: min(4, dp[3]+1=3)=3
  a=9: min(5, dp[4]+1=3)=3
  a=10: min(5, dp[5]+1=2)=2
  a=11: min(6, dp[6]+1=3)=3
  dp = [0,1,1,2,2,1,2,2,3,3,2,3]

Answer: dp[11] = 3  (5 + 5 + 1) ✓
```

**Contrast with 0/1 in one sentence:** if this were 0/1 (each coin usable once), the inner loop would run `a = amount downto coin`, and the answer for amount=11 with only one of each {1,2,5} would be impossible to hit exactly with 3 coins — you'd need a completely different, larger coin set.

---

## SECTION 6 — TEMPLATE 4: SUBSET SUM / PARTITION EQUAL SUBSET SUM

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 416 — Partition Equal Subset Sum
// Can the array be split into two subsets with equal sum?
//
// THE DISGUISE: nothing here says "knapsack." The reduction:
//   - If total sum is odd, impossible (two equal integer halves can't
//     sum to an odd total). Return false immediately.
//   - Otherwise target = total / 2. Question becomes: "does a SUBSET
//     of nums sum to EXACTLY target?" — classic 0/1 subset-sum.
//   - Each number is an "item" whose weight AND value are both itself;
//     we don't maximize value, we just ask reachability.
//
// State:      dp[s] = true if some subset of items considered so far
//                     sums to exactly s
// Transition: dp[s] = dp[s] || dp[s - nums[i]]   (0/1 — take or skip)
// Base case:  dp[0] = true (empty subset sums to 0)
// Answer:     dp[target]
//
// LOOP DIRECTION: descending — this is 0/1 (each number used once).
// ─────────────────────────────────────────────────────────────────

bool canPartition(vector<int>& nums) {
    long long total = 0;
    for (int x : nums) total += x;
    if (total % 2 != 0) return false;   // odd total → can't split evenly

    long long target = total / 2;
    vector<bool> dp(target + 1, false);
    dp[0] = true;

    for (int num : nums) {
        // Descending — 0/1 knapsack: each number used at most once
        for (long long s = target; s >= num; s--) {
            dp[s] = dp[s] || dp[s - num];
        }
    }

    return dp[target];
}
```

**TRACE for nums=[1,5,11,5], total=22, target=11:**
```
dp[0..11] = [T,F,F,F,F,F,F,F,F,F,F,F]

num=1: s=11..1, dp[s] |= dp[s-1]
  Only dp[1] |= dp[0]=T → dp[1]=T. (dp[s-1] is F for all other s at this point)
  dp = [T,T,F,F,F,F,F,F,F,F,F,F]

num=5: s=11..5, dp[s] |= dp[s-5]
  s=11: dp[11] |= dp[6]=F → F
  s=10: dp[10] |= dp[5]=F → F
  s=9:  dp[9]  |= dp[4]=F → F
  s=8:  dp[8]  |= dp[3]=F → F
  s=7:  dp[7]  |= dp[2]=F → F
  s=6:  dp[6]  |= dp[1]=T → T
  s=5:  dp[5]  |= dp[0]=T → T
  dp = [T,T,F,F,F,T,T,F,F,F,F,F]

num=11: s=11..11, dp[11] |= dp[0]=T → T
  dp = [T,T,F,F,F,T,T,F,F,F,F,T]   ← already true! (subset {11})

num=5 (second copy): s=11..5, dp[s] |= dp[s-5]
  s=11: dp[11] |= dp[6]=T → T (no change, already T)
  s=10: dp[10] |= dp[5]=T → T
  s=9:  dp[9]  |= dp[4]=F → F
  s=8:  dp[8]  |= dp[3]=F → F
  s=7:  dp[7]  |= dp[2]=F → F
  s=6:  dp[6]  |= dp[1]=T → T (no change)
  s=5:  dp[5]  |= dp[0]=T → T (no change)
  dp = [T,T,F,F,F,T,T,F,F,F,T,T]

Answer: dp[11] = true → return true  ({11} or {1,5,5} both sum to 11) ✓
```

**Why boolean OR instead of max/sum:** subset-sum only asks "is it reachable," not "what's the best value." The recurrence collapses from `max(...)` to `||` because there's no value to optimize — reachability is binary.

---

## SECTION 7 — TEMPLATE 5: COUNT WAYS — COIN CHANGE II (COMBINATIONS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 518 — Coin Change II
// Count the NUMBER OF DISTINCT COMBINATIONS of coins (unlimited supply
// of each) that sum to amount. Order does NOT matter: {1,2,2} and
// {2,1,2} are the SAME combination and must be counted ONCE.
//
// State:      dp[a] = number of ways to make amount a using coins
//                     considered SO FAR (this is the key difference
//                     from a plain unbounded knapsack — "so far"
//                     freezes the set of coin types we're allowed
//                     to combine, which is exactly what prevents
//                     double-counting reorderings)
// Transition: dp[a] += dp[a - coin]
// Base case:  dp[0] = 1 (one way to make amount 0: use no coins)
// Answer:     dp[amount]
//
// LOOP ORDER — THE CRITICAL DECISION:
//   OUTER loop = coins, INNER loop = amount (ascending)
//   This processes ONE coin type fully (all amounts) before moving to
//   the next coin type. Because coin type is fixed in the outer loop,
//   any way counted is built by "deciding how many of THIS coin to use,
//   then how many of the NEXT coin, etc." — coins are always considered
//   in a fixed relative order → no two orderings of the same multiset
//   get counted separately → COMBINATIONS.
// ─────────────────────────────────────────────────────────────────

int change(int amount, vector<int>& coins) {
    vector<long long> dp(amount + 1, 0);
    dp[0] = 1;

    for (int coin : coins) {              // OUTER: coins
        for (int a = coin; a <= amount; a++) {   // INNER: amount, ascending (unbounded)
            dp[a] += dp[a - coin];
        }
    }

    return (int)dp[amount];
}
```

**TRACE for amount=5, coins=[1,2,5]:**
```
dp[0..5] = [1,0,0,0,0,0]

coin=1 (outer): a=1..5 ascending
  dp[1] += dp[0] = 0+1 = 1
  dp[2] += dp[1] = 0+1 = 1
  dp[3] += dp[2] = 0+1 = 1
  dp[4] += dp[3] = 0+1 = 1
  dp[5] += dp[4] = 0+1 = 1
  dp = [1,1,1,1,1,1]   (only way so far: all 1s)

coin=2 (outer): a=2..5 ascending
  dp[2] += dp[0] = 1+1 = 2   (ways: 1+1, 2)
  dp[3] += dp[1] = 1+1 = 2   (ways: 1+1+1, 1+2)
  dp[4] += dp[2] = 1+2 = 3   (ways: 1×4, 1+1+2, 2+2)
  dp[5] += dp[3] = 1+2 = 3   (ways: 1×5, 1+1+1+2, 1+2+2)
  dp = [1,1,2,2,3,3]

coin=5 (outer): a=5..5
  dp[5] += dp[0] = 3+1 = 4   (adds the way: {5})
  dp = [1,1,2,2,3,4]

Answer: dp[5] = 4
  The 4 combinations: {5}, {1,1,1,1,1}, {1,2,2}, {1,1,1,2} ✓
```

---

## SECTION 8 — TEMPLATE 6: COMBINATION SUM IV (PERMUTATIONS) — LOOP ORDER DEEP DIVE

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 377 — Combination Sum IV
// Count the number of ORDERED SEQUENCES of numbers (unlimited supply
// of each, order MATTERS) that sum to target. Despite the name
// "Combination," this problem counts PERMUTATIONS: (1,2,1) and (2,1,1)
// are counted as two DIFFERENT results.
//
// State:      dp[a] = number of ordered sequences summing to a
// Transition: dp[a] += dp[a - num] for every num in nums
// Base case:  dp[0] = 1 (one empty sequence sums to 0)
// Answer:     dp[target]
//
// LOOP ORDER — THE OPPOSITE OF SECTION 7:
//   OUTER loop = amount (ascending), INNER loop = nums
//   For EVERY amount, we re-try EVERY number as a possible "last
//   element appended." Because the choice of number is re-evaluated
//   at every amount independently (not frozen by an outer coin loop),
//   sequences like (1,then 2) and (2,then 1) are built via DIFFERENT
//   paths through the DP and both get counted → PERMUTATIONS.
// ─────────────────────────────────────────────────────────────────

int combinationSum4(vector<int>& nums, int target) {
    vector<long long> dp(target + 1, 0);
    dp[0] = 1;

    for (int a = 1; a <= target; a++) {         // OUTER: amount
        for (int num : nums) {                   // INNER: every number, every time
            if (num <= a) {
                dp[a] += dp[a - num];
            }
        }
    }

    return (int)dp[target];
}
```

**TRACE for nums=[1,2,3], target=4:**
```
dp[0] = 1

a=1: try num=1: dp[1] += dp[0] = 1   (num=2,3 too big)
  dp[1] = 1

a=2: try num=1: dp[2] += dp[1] = 1
     try num=2: dp[2] += dp[0] = 1+1 = 2
  dp[2] = 2   (sequences: (1,1), (2))

a=3: try num=1: dp[3] += dp[2] = 2
     try num=2: dp[3] += dp[1] = 2+1 = 3
     try num=3: dp[3] += dp[0] = 3+1 = 4
  dp[3] = 4   (sequences: (1,1,1), (1,2), (2,1), (3))

a=4: try num=1: dp[4] += dp[3] = 4
     try num=2: dp[4] += dp[2] = 4+2 = 6
     try num=3: dp[4] += dp[1] = 6+1 = 7
  dp[4] = 7

Answer: dp[4] = 7
  Sequences: (1,1,1,1), (1,1,2), (1,2,1), (2,1,1), (2,2), (1,3), (3,1) ✓
```

**SIDE-BY-SIDE — the exact same two lines of code, swapped, produce different answers on the exact same input:**

```cpp
// COMBINATIONS (Coin Change II) — coins outer, amount inner
for (int coin : coins)
    for (int a = coin; a <= amount; a++)
        dp[a] += dp[a - coin];

// PERMUTATIONS (Combination Sum IV) — amount outer, items inner
for (int a = 1; a <= target; a++)
    for (int num : nums)
        if (num <= a) dp[a] += dp[a - num];
```

Run both on `coins/nums = [1,2,5]`, `amount/target = 5`: the combinations version gives **4**, the permutations version gives **9** (many more — every reordering of {1,1,1,2}, {1,2,2} etc. is counted separately, plus the pure-5 way). Same recurrence shape, opposite loop nesting, structurally different meaning.

---

## SECTION 9 — TEMPLATE 7: TARGET SUM (THE S1 − S2 = TARGET TRANSFORM)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 494 — Target Sum
// Assign a '+' or '−' sign to each number in nums so the expression
// evaluates to target. Count the number of ways to do this.
//
// NAIVE APPROACH: O(2^n) DFS trying both signs at every index. Works
// for small n, times out for n up to 20 with large sums.
//
// THE ALGEBRAIC TRANSFORM (the entire skill being tested here):
//   Let P = sum of numbers assigned '+', N = sum of numbers assigned '−'.
//   Then:            P - N = target
//   Also trivially:  P + N = totalSum   (every number gets exactly one sign)
//   Adding these two equations: 2P = target + totalSum
//                                P = (target + totalSum) / 2
//
//   So the question "how many ways to sign the array to hit target"
//   becomes "how many SUBSETS of nums sum to exactly P" — a pure
//   0/1 knapsack COUNT problem (Section 7's recurrence, but 0/1
//   loop direction from Section 4/6, since each number is used once).
//
// EDGE CASES (interview-critical — miss these and it's a silent wrong
// answer, not a crash):
//   1. If (target + totalSum) is ODD, no integer P exists → return 0.
//   2. If (target + totalSum) < 0, or P would be negative, or P > totalSum
//      → impossible → return 0. (target can exceed totalSum in magnitude.)
//
// State:      dp[s] = number of subsets summing to exactly s
// Transition: dp[s] += dp[s - num]     (0/1: descending capacity loop)
// Base case:  dp[0] = 1
// Answer:     dp[P]
// ─────────────────────────────────────────────────────────────────

int findTargetSumWays(vector<int>& nums, int target) {
    long long totalSum = 0;
    for (int x : nums) totalSum += x;

    long long need = (long long)target + totalSum;
    // Must be non-negative and even, and P must not exceed totalSum
    if (need < 0 || need % 2 != 0) return 0;
    long long P = need / 2;
    if (P > totalSum) return 0;

    vector<long long> dp(P + 1, 0);
    dp[0] = 1;

    for (int num : nums) {
        // 0/1 knapsack — each number signed exactly once — descending
        for (long long s = P; s >= num; s--) {
            dp[s] += dp[s - num];
        }
    }

    return (int)dp[P];
}
```

**TRACE for nums=[1,1,1,1,1], target=3:**
```
totalSum = 5. need = 3 + 5 = 8 (even, good). P = 4.

dp[0..4] = [1,0,0,0,0]

item=1 (first): s=4..1, dp[s] += dp[s-1]
  dp[1] += dp[0]=1 → dp[1]=1
  dp[2..4] += dp[1..3]=0 → unchanged
  dp = [1,1,0,0,0]

item=1 (second): s=4..1
  dp[4] += dp[3]=0 → 0
  dp[3] += dp[2]=0 → 0
  dp[2] += dp[1]=1 → 1
  dp[1] += dp[0]=1 → 2
  dp = [1,2,1,0,0]

item=1 (third): s=4..1
  dp[4] += dp[3]=0 → 0
  dp[3] += dp[2]=1 → 1
  dp[2] += dp[1]=2 → 3
  dp[1] += dp[0]=1 → 3
  dp = [1,3,3,1,0]

item=1 (fourth): s=4..1
  dp[4] += dp[3]=1 → 1
  dp[3] += dp[2]=3 → 4
  dp[2] += dp[1]=3 → 6
  dp[1] += dp[0]=1 → 4
  dp = [1,4,6,4,1]

item=1 (fifth): s=4..1
  dp[4] += dp[3]=4 → 5
  dp[3] += dp[2]=6 → 10
  dp[2] += dp[1]=4 → 10
  dp[1] += dp[0]=1 → 5
  dp = [1,5,10,10,5]

Answer: dp[4] = 5
  (verified against LC's known answer of 5 for this exact input) ✓
```

**Why this is a 0/1 knapsack, not unbounded:** each of the five `1`s in `nums` is a physically distinct array element — you assign it a sign exactly once. Even though they share the same value, they are five separate "items," each usable once. Loop direction must be descending.

---

## SECTION 10 — TEMPLATE 8: ONES AND ZEROES (2D KNAPSACK)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 474 — Ones and Zeroes
// Given strings of '0's and '1's, and a budget of m zeros and n ones,
// find the maximum number of strings you can select such that the
// TOTAL zeros used across selected strings <= m AND TOTAL ones used
// <= n. Each string used at most once (0/1).
//
// THE KEY INSIGHT: this is 0/1 knapsack with TWO capacity dimensions
// instead of one. "Weight" is now a PAIR (zeros used, ones used), and
// "value" is always 1 (we're maximizing COUNT of strings, not a
// weighted value). The 1D-backward trick from Section 4 generalizes:
// both capacity dimensions must be iterated DESCENDING for the same
// reason — each string is one "item," used at most once.
//
// State:      dp[i][j] = max strings selectable using at most i zeros
//                        and at most j ones
// Transition: dp[i][j] = max(dp[i][j], dp[i - zeros][j - ones] + 1)
//             for the current string's (zeros, ones) cost
// Base case:  dp[i][j] = 0 for all i, j (no strings selected)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

int findMaxForm(vector<string>& strs, int m, int n) {
    // dp[i][j]: budget of i zeros, j ones remaining
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (const string& s : strs) {
        int zeros = 0, ones = 0;
        for (char c : s) (c == '0') ? zeros++ : ones++;

        // 2D 0/1 knapsack — BOTH dimensions descending
        for (int i = m; i >= zeros; i--) {
            for (int j = n; j >= ones; j--) {
                dp[i][j] = max(dp[i][j], dp[i - zeros][j - ones] + 1);
            }
        }
    }

    return dp[m][n];
}
```

**TRACE for strs=["10","0","1"], m=1, n=1:**
```
dp (2x2, rows=zeros budget 0..1, cols=ones budget 0..1):
  [0,0]
  [0,0]

string "10": zeros=1, ones=1
  i=1..1, j=1..1:
    dp[1][1] = max(dp[1][1]=0, dp[0][0]+1=1) = 1
  dp:
    [0,0]
    [0,1]

string "0": zeros=1, ones=0
  i=1..1, j=1..0:
    j=1: dp[1][1] = max(dp[1][1]=1, dp[0][1]+1=0+1=1) = 1  (no change)
    j=0: dp[1][0] = max(dp[1][0]=0, dp[0][0]+1=0+1=1) = 1
  dp:
    [0,0]
    [1,1]

string "1": zeros=0, ones=1
  i=1..0, j=1..1:
    i=1,j=1: dp[1][1] = max(dp[1][1]=1, dp[1][0]+1=1+1=2) = 2
    i=0,j=1: dp[0][1] = max(dp[0][1]=0, dp[0][0]+1=0+1=1) = 1
  dp:
    [0,1]
    [1,2]

Answer: dp[1][1] = 2   (select "0" and "1": uses exactly 1 zero, 1 one) ✓
```

**Why not just two separate 1D knapsacks (one for zeros, one for ones)?** Because the constraint is joint — a single string consumes BOTH budgets simultaneously. Treating them independently would double-count budget usage. The 2D state is necessary precisely because the two resources are consumed together, not separately.

---

## SECTION 11 — TEMPLATE 9: TALLEST BILLBOARD (KNAPSACK ON DIFFERENCE)

The hardest problem in this pattern. Nobody derives the state definition from scratch without prior exposure — internalize it here.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 956 — Tallest Billboard
// Given rod lengths, split them into two disjoint groups (a rod can
// also be left unused) such that both groups' sums are EQUAL, and
// that common sum is MAXIMIZED. Return the max equal height (0 if
// no non-trivial solution beats the empty answer).
//
// WHY THIS ISN'T A NORMAL KNAPSACK: there is no fixed "capacity" in
// the problem. We're not filling a container to a target — we're
// building TWO piles simultaneously and want them equal and tall.
//
// THE KEY INSIGHT — knapsack keyed on the DIFFERENCE:
//   Track dp[diff] = the maximum height of the SHORTER pile, given
//   that (taller pile height - shorter pile height) == diff.
//   dp[0] at the end = the tallest achievable EQUAL height, because
//   diff=0 means both piles are the same height, and we tracked the
//   max shorter-pile height along the way (which equals the taller
//   pile height when diff=0).
//
// For each rod r, and each existing state (diff=d, shortHeight=s),
// there are three choices:
//   1. Skip the rod entirely — state unchanged.
//   2. Add r to the TALLER pile — new diff = d + r, short unchanged.
//   3. Add r to the SHORTER pile — the short pile grows by r. Whichever
//      pile is now shorter becomes the new "short," and the gap
//      shrinks or flips:
//        newShort = s + min(d, r)
//        newDiff  = |d - r|
//      (Derivation: tall = s + d. After adding r to short: new short
//       candidate = s + r. Compare against tall = s + d. The smaller
//       of the two becomes the new short pile height, and the
//       absolute difference is the new gap.)
//
// State:      dp[diff] = max height of the shorter pile
// Transition: as above, for every rod, for every existing diff state
// Base case:  dp[0] = 0 (both piles empty, equal, height 0)
// Answer:     dp[0] after processing all rods
// ─────────────────────────────────────────────────────────────────

int tallestBillboard(vector<int>& rods) {
    // dp: diff -> max height of the shorter pile for that diff
    unordered_map<long long, long long> dp;
    dp[0] = 0;

    for (int r : rods) {
        unordered_map<long long, long long> next = dp;   // option 1: skip r

        for (auto& [d, shortH] : dp) {
            // option 2: add r to the taller pile
            long long newDiff2 = d + r;
            next[newDiff2] = max(next.count(newDiff2) ? next[newDiff2] : -1, shortH);

            // option 3: add r to the shorter pile
            long long newShort = shortH + min<long long>(d, r);
            long long newDiff3 = llabs(d - r);
            next[newDiff3] = max(next.count(newDiff3) ? next[newDiff3] : -1, newShort);
        }

        dp = next;
    }

    return (int)dp[0];
}
```

**TRACE for rods=[1,2,3,6] (expected answer: 6):**
```
dp = {0: 0}

r=1:
  from (d=0, short=0):
    taller:  diff=1, cand=0 → dp[1]=0
    shorter: newShort=0+min(0,1)=0, diff=1 → dp[1]=max(0,0)=0
  dp = {0:0, 1:0}

r=2:
  from (d=0, short=0):
    taller:  diff=2, cand=0 → dp[2]=0
    shorter: newShort=0+min(0,2)=0, diff=2 → dp[2]=max(0,0)=0
  from (d=1, short=0):
    taller:  diff=3, cand=0 → dp[3]=0
    shorter: newShort=0+min(1,2)=1, diff=1 → dp[1]=max(0,1)=1
  dp = {0:0, 1:1, 2:0, 3:0}

r=3:
  from (d=0, short=0):   taller diff=3 → dp[3]=max(0,0)=0
                          shorter newShort=0+min(0,3)=0, diff=3 → dp[3]=max(0,0)=0
  from (d=1, short=1):   taller diff=4 → dp[4]=1
                          shorter newShort=1+min(1,3)=2, diff=2 → dp[2]=max(0,2)=2
  from (d=2, short=0):   taller diff=5 → dp[5]=0
                          shorter newShort=0+min(2,3)=2, diff=1 → dp[1]=max(1,2)=2
  from (d=3, short=0):   taller diff=6 → dp[6]=0
                          shorter newShort=0+min(3,3)=3, diff=0 → dp[0]=max(0,3)=3
  dp = {0:3, 1:2, 2:2, 3:0, 4:1, 5:0, 6:0}

r=6:
  from (d=0, short=3):   taller diff=6 → dp[6]=max(0,3)=3
                          shorter newShort=3+min(0,6)=3, diff=6 → dp[6]=max(3,3)=3
  from (d=1, short=2):   taller diff=7 → dp[7]=2
                          shorter newShort=2+min(1,6)=3, diff=5 → dp[5]=max(0,3)=3
  from (d=2, short=2):   taller diff=8 → dp[8]=2
                          shorter newShort=2+min(2,6)=4, diff=4 → dp[4]=max(1,4)=4
  from (d=3, short=0):   taller diff=9 → dp[9]=0
                          shorter newShort=0+min(3,6)=3, diff=3 → dp[3]=max(0,3)=3
  from (d=4, short=1):   taller diff=10 → dp[10]=1
                          shorter newShort=1+min(4,6)=5, diff=2 → dp[2]=max(2,5)=5
  from (d=5, short=0):   taller diff=11 → dp[11]=0
                          shorter newShort=0+min(5,6)=5, diff=1 → dp[1]=max(2,5)=5
  from (d=6, short=0):   taller diff=12 → dp[12]=0
                          shorter newShort=0+min(6,6)=6, diff=0 → dp[0]=max(3,6)=6

Final: dp[0] = 6

Verification: rods {1,2,3} sum to 6, rod {6} alone sums to 6 — two equal
piles of height 6, using every rod. Matches LC's known answer. ✓
```

**Why `unordered_map` instead of a `vector`?** `diff` can range up to the sum of all rods (tens of thousands), but only a small fraction of possible diffs are ever actually reachable — a hash map avoids allocating a huge mostly-empty array. For tighter bounds, a `vector` offset by the max possible sum also works and is faster in practice; the map is shown here for clarity of the sparse-state idea.

---

## SECTION 12 — COMPLEXITY TABLE

| Problem | Time | Space | Optimized Space | Notes |
|---|---|---|---|---|
| 0/1 Knapsack | O(nW) | O(nW) | O(W) — 1D descending | Foundation |
| Coin Change (min coins) | O(n·amount) | O(amount) | O(amount) | Unbounded, ascending |
| Coin Change II (count) | O(n·amount) | O(amount) | O(amount) | Combinations — coins outer |
| Combination Sum IV | O(target·n) | O(target) | O(target) | Permutations — amount outer |
| Partition Equal Subset Sum | O(n·sum) | O(sum) | O(sum) | Boolean 0/1 subset-sum |
| Target Sum | O(n·P) | O(P) | O(P) | 0/1 count-ways, P=(target+sum)/2 |
| Last Stone Weight II | O(n·sum) | O(sum) | O(sum) | 0/1 subset-sum, minimize diff |
| Ones and Zeroes | O(L·m·n) | O(mn) | O(mn) | 2D 0/1, both dims descending |
| Perfect Squares | O(n·√n) | O(n) | O(n) | Unbounded, "coins" = perfect squares |
| Profitable Schemes | O(crimes·n·minProfit) | O(n·minProfit) | same | 3D 0/1 (group + profit dims) |
| Tallest Billboard | O(n·D) amortized | O(D) | O(D) — D = achievable diffs | Knapsack on difference, sparse map |

`n` = number of items/strings/rods; `W`/`amount`/`sum`/`P`/`target` = capacity-like bound; `L` = number of strings (Ones and Zeroes); `D` = number of distinct reachable differences (Tallest Billboard, ≤ sum of rods but usually far smaller).

---

## SECTION 13 — VARIANTS

### Perfect Squares (LC 279) — unbounded knapsack with generated items
The "coins" aren't given — you generate them: every perfect square `≤ n` is a reusable "coin." Otherwise identical to Coin Change's min-coins template (Section 5).
```cpp
int numSquares(int n) {
    vector<int> dp(n + 1, INT_MAX);
    dp[0] = 0;
    for (int i = 1; i <= n; i++)
        for (int j = 1; j * j <= i; j++)          // "coin" = j*j
            dp[i] = min(dp[i], dp[i - j*j] + 1);
    return dp[n];
}
```
Trace check for n=12: dp builds up to dp[12]=3 (4+4+4), matching the known answer.

### Last Stone Weight II (LC 1049) — minimize the leftover, not maximize a value
Reduces to: partition into two piles minimizing `|sumA - sumB|`. Compute the maximum achievable subset sum `S ≤ total/2` via subset-sum reachability (Section 6's exact template), then answer is `total - 2*S`.
```cpp
int lastStoneWeightII(vector<int>& stones) {
    long long total = accumulate(stones.begin(), stones.end(), 0LL);
    long long target = total / 2;
    vector<bool> dp(target + 1, false);
    dp[0] = true;
    for (long long s : stones)
        for (long long w = target; w >= s; w--)
            dp[w] = dp[w] || dp[w - s];
    long long best = target;
    while (!dp[best]) best--;      // walk down to the largest reachable sum
    return (int)(total - 2 * best);
}
```

### Profitable Schemes (LC 879) — 3D knapsack (adds a group-size dimension)
Now every "item" (a crime) costs BOTH a number of group members AND yields a profit, and we cap profit contribution at `minProfit` (anything beyond the threshold still counts as "profitable enough" — profit dimension is clamped, not summed unboundedly).
```cpp
int profitableSchemes(int n, int minProfit, vector<int>& group, vector<int>& profit) {
    const int MOD = 1e9 + 7;
    int m = group.size();
    // dp[j][p] = number of schemes using exactly j members, profit capped at p
    vector<vector<long long>> dp(n + 1, vector<long long>(minProfit + 1, 0));
    for (int j = 0; j <= n; j++) dp[j][0] = 1;   // 0 profit needed: empty scheme always works

    for (int i = 0; i < m; i++) {
        int g = group[i], p = profit[i];
        for (int j = n; j >= g; j--) {
            for (int pr = minProfit; pr >= 0; pr--) {
                int prevProfit = max(0, pr - p);   // clamp: profit beyond minProfit merges into the "0+" bucket
                dp[j][pr] = (dp[j][pr] + dp[j - g][prevProfit]) % MOD;
            }
        }
    }
    return (int)dp[n][minProfit];
}
```
This is a direct 3D generalization of Ones and Zeroes: two capacity dimensions (`members`, `profit`) both iterated descending because each crime is 0/1 (committed at most once).

### Minimum Cost to Fill Given Weight (classic contest knapsack)
Given item types each with a `(weight, cost)`, unlimited supply of each, find the minimum cost to fill a bag to EXACTLY capacity `W` (not "at most" — exactly). Structurally identical to Coin Change's min-coins template (Section 5) with `cost` playing the role of "+1 per coin."
```cpp
long long minCostToFill(vector<long long>& wt, vector<long long>& cost, long long W) {
    const long long INF = LLONG_MAX / 2;
    vector<long long> dp(W + 1, INF);
    dp[0] = 0;
    for (size_t i = 0; i < wt.size(); i++)
        for (long long w = wt[i]; w <= W; w++)      // unbounded — ascending
            dp[w] = min(dp[w], dp[w - wt[i]] + cost[i]);
    return dp[W] >= INF ? -1 : dp[W];
}
```

---

## SECTION 14 — COMMON MISTAKES

### Mistake 1: Wrong loop direction for 1D 0/1 knapsack (ascending instead of descending)

```cpp
// WRONG — ascending capacity loop on a 0/1 knapsack
for (int i = 0; i < n; i++)
    for (int w = wt[i]; w <= W; w++)             // BUG: ascending
        dp[w] = max(dp[w], dp[w - wt[i]] + val[i]);
// dp[w - wt[i]] may ALREADY include item i (updated earlier THIS pass),
// silently allowing item i to be used multiple times — this computes
// the UNBOUNDED knapsack answer, not the 0/1 answer. On Partition Equal
// Subset Sum, this can make an actually-unsplittable array appear
// splittable, because a single number gets "reused."

// CORRECT — descending
for (int i = 0; i < n; i++)
    for (int w = W; w >= wt[i]; w--)             // each item used once
        dp[w] = max(dp[w], dp[w - wt[i]] + val[i]);
```

---

### Mistake 2: Wrong loop direction for unbounded knapsack (descending instead of ascending)

```cpp
// WRONG — descending capacity loop on Coin Change
for (int coin : coins)
    for (int a = amount; a >= coin; a--)          // BUG: descending
        dp[a] = min(dp[a], dp[a - coin] + 1);
// dp[a - coin] is guaranteed to be the value BEFORE this coin was
// considered (nothing below `a` has been touched yet this pass), so
// this coin can only ever be used ONCE per amount — this silently
// computes the 0/1 knapsack answer. Coin Change with coins=[1,2,5],
// amount=11 would fail to find the true minimum (3 coins: 5+5+1)
// if coins could only be used once each.

// CORRECT — ascending
for (int coin : coins)
    for (int a = coin; a <= amount; a++)          // coin reusable
        dp[a] = min(dp[a], dp[a - coin] + 1);
```

---

### Mistake 3: Combinations vs permutations — swapped loop nesting

```cpp
// WRONG — trying to count COMBINATIONS (Coin Change II) with the
// PERMUTATIONS loop order
for (int a = 1; a <= amount; a++)
    for (int coin : coins)
        if (coin <= a) dp[a] += dp[a - coin];
// This counts {1,2} and {2,1} as different ways to reach 3 — but
// Coin Change II wants combinations (order-independent), so the
// answer comes out too LARGE. For coins=[1,2,5], amount=5, this
// gives 9 instead of the correct 4.

// CORRECT for combinations — coins outer
for (int coin : coins)
    for (int a = coin; a <= amount; a++)
        dp[a] += dp[a - coin];

// And the reverse mistake: using the COMBINATIONS loop order for
// Combination Sum IV (which wants permutations) UNDER-counts —
// it would only find 4 ways for target=4 with nums=[1,2,3] instead
// of the correct 7, because it never considers "2 then 1" as
// different from "1 then 2."
```

---

### Mistake 4: Target Sum — forgetting the parity / feasibility check

```cpp
// WRONG — computing P without checking (target + totalSum) is even
// or non-negative
long long P = (target + totalSum) / 2;   // integer division silently
                                          // truncates on odd sums!
// If target + totalSum is odd, there's no integer solution for P —
// the "obvious" answer is 0 ways, but truncating division produces
// a plausible-looking P and the DP happily returns a WRONG nonzero
// count for an input that should return 0.

// CORRECT
long long need = (long long)target + totalSum;
if (need < 0 || need % 2 != 0) return 0;   // explicit guard BEFORE dividing
long long P = need / 2;
if (P > totalSum) return 0;                // target magnitude exceeds what's achievable
```

---

### Mistake 5: Integer overflow in count-ways / cost accumulation

```cpp
// WRONG — using int for dp values that can grow combinatorially
vector<int> dp(amount + 1, 0);
dp[a] += dp[a - coin];
// For large amount/coin-count combinations (or Combination Sum IV
// with small nums and large target — the permutation count grows
// roughly exponentially before capping at target), dp values can
// exceed INT_MAX and silently wrap to negative or garbage values.

// CORRECT — use long long for accumulating counts (and mod if the
// problem specifies one, e.g. Profitable Schemes uses 1e9+7)
vector<long long> dp(amount + 1, 0);
dp[a] += dp[a - coin];
// If a modulus is specified: dp[a] = (dp[a] + dp[a-coin]) % MOD;
```

---

### Mistake 6: 2D knapsack — only iterating one dimension descending

```cpp
// WRONG — Ones and Zeroes with only the outer dimension descending
for (int i = m; i >= zeros; i--)
    for (int j = 0; j <= n; j++)                 // BUG: ascending!
        dp[i][j] = max(dp[i][j], dp[i - zeros][j - ones] + 1);
// The inner (ones) dimension being ascending means dp[i][j - ones]
// may already include the CURRENT string (updated earlier this same
// pass, at a smaller j). This lets a single string be counted twice
// within one iteration over strs — effectively half-unbounded,
// half-0/1, and wrong in both directions at once.

// CORRECT — BOTH dimensions descending, since each string is 0/1
for (int i = m; i >= zeros; i--)
    for (int j = n; j >= ones; j--)
        dp[i][j] = max(dp[i][j], dp[i - zeros][j - ones] + 1);
```

---

## SECTION 15 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Partition Equal Subset Sum | 416 | Recognizing the 0/1 disguise: split → subset-sum at total/2 |
| 2 | Coin Change | 322 | Unbounded knapsack, ascending loop, min-coins recurrence |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Coin Change II | 518 | Unbounded count-ways, combinations loop order (coins outer) |
| 4 | Target Sum | 494 | The P − N = target algebraic transform, parity/feasibility guards |
| 5 | Last Stone Weight II | 1049 | 0/1 subset-sum reachability, minimize |totalA − totalB| |
| 6 | Ones and Zeroes | 474 | 2D 0/1 knapsack, both dimensions descending |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Combination Sum IV | 377 | Permutation loop order (amount outer) — contrast with #3 |
| 8 | Perfect Squares | 279 | Unbounded variant with generated "coin" set (perfect squares) |
| 9 | Profitable Schemes | 879 | 3D knapsack — group-size dimension + clamped profit dimension |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Minimum Cost to Fill Given Weight | (GFG classic) | Unbounded min-cost knapsack, exact-capacity variant |
| 11 | Tallest Billboard | 956 | Knapsack keyed on DIFFERENCE, not sum — hardest problem in the pattern |

---

## SECTION 16 — PATTERN CONNECTIONS

1. **Pattern 17 (1D DP):** Coin Change's min-coins recurrence is structurally identical to the "take or skip" 1D DP templates from Pattern 17 — the only new idea is the OUTER loop over item TYPES, which Pattern 17's single-pass problems never needed (there was only ever one sequence to scan). Once you've internalized "state = dp[i]" from Pattern 17, knapsack just adds "and now there's a family of items, each contributing its own transition."

2. **Pattern 18 (2D DP):** Ones and Zeroes (Section 10) uses EXACTLY the same 2D table shape as Unique Paths (Pattern 18, Section 3) — `dp[i][j]` with two independent dimensions. The only difference is semantic: Unique Paths' dimensions are grid coordinates; Ones and Zeroes' dimensions are two independent resource budgets. Recognizing "two independent constraints → 2D DP table" is the shared skill.

3. **Pattern 16 (DP Fundamentals):** Every template in this document is built from the four DP questions (state, transition, base case, answer) taught in Pattern 16. The knapsack-specific addition is a FIFTH question that Pattern 16 doesn't need to ask: "in what direction do I iterate capacity when space-optimizing, and does that direction change the problem's semantics?" No other DP family in this curriculum has a correctness-critical loop DIRECTION the way knapsack does.

4. **Pattern 21 (Longest Increasing Subsequence, upcoming):** LIS's O(n log n) patience-sorting solution uses a similar "greedy pile" mental model to Tallest Billboard's "two piles" — though the mechanics differ completely, both problems reward the insight that tracking an auxiliary quantity (LIS: smallest tail per length; Billboard: max short-height per diff) collapses an exponential search into a poly-time DP.

5. **Pattern 25 (Bitmask DP, upcoming):** Profitable Schemes' 3D state (`dp[j][p]`) is a preview of the "extra dimension per constraint" idea that Bitmask DP takes to its extreme — tracking a whole SET of constraints (as a bitmask) instead of just one or two scalar budgets.

---

## SECTION 17 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 416 — Partition Equal Subset Sum

**Interviewer:** "Given an array of positive integers, determine if it can be partitioned into two subsets with equal sum."

**Candidate:**

*[First 30 seconds — spot the disguise before touching code]*
> "This doesn't mention weights or values, but I recognize it as subset-sum. If the total sum is odd, it's immediately impossible — two equal integer halves can't add to an odd number. Otherwise I need to know: does a subset exist summing to exactly `total / 2`? If yes, the complement automatically sums to the other half."

*[State the DP before coding]*
> "This is a 0/1 knapsack — boolean reachability instead of value maximization. `dp[s] = true` if some subset sums to `s`. Transition: `dp[s] |= dp[s - num]`. Since each number is used at most once, I iterate the capacity loop DESCENDING."

*[Code — 5 minutes]*
```cpp
bool canPartition(vector<int>& nums) {
    long long total = accumulate(nums.begin(), nums.end(), 0LL);
    if (total % 2) return false;
    long long target = total / 2;
    vector<bool> dp(target + 1, false);
    dp[0] = true;
    for (int num : nums)
        for (long long s = target; s >= num; s--)
            dp[s] = dp[s] || dp[s - num];
    return dp[target];
}
```

*[Interviewer]: "What if the loop were ascending instead?"*
> "Then a single number could be counted toward the target sum multiple times — for example `nums = [3]`, target would need `3` reachable using one `3`, but with ascending iteration a smaller earlier-updated cell could let `3` be 'used again' at a higher capacity in the same pass. It would silently compute reachability under an UNBOUNDED assumption, which is wrong here since each number physically exists once in the array."

**RED FLAGS:**
- Not checking for odd total sum before doing any DP work
- Ascending capacity loop (turns 0/1 into unbounded)
- Confusing this with the VALUE-maximizing knapsack and returning a number instead of a boolean

---

### Interview Simulation 2: LC 494 — Target Sum

**Interviewer:** "You have an array of non-negative integers and a target. Assign each number a `+` or `−` sign so the total equals target. Count the number of ways."

**Candidate:**

*[Naive approach first, then the pivot]*
> "Brute force is O(2^n) — DFS trying both signs at each index, which times out around n=20 for large sums. There's an algebraic reduction that turns this into a knapsack count problem."

*[Derive the transform out loud — this IS the interview]*
> "Split the array into a 'positive' subset P and 'negative' subset N. Then `P - N = target` and `P + N = totalSum` since every element is in exactly one group. Adding those: `2P = target + totalSum`, so `P = (target + totalSum) / 2`. Now the question becomes: how many subsets of `nums` sum to exactly `P`? That's a 0/1 count-ways knapsack."

*[State the guards before coding — this is what separates a clean pass from a partial credit]*
> "Before computing `P`, I need three checks: `target + totalSum` must be non-negative, it must be EVEN (P has to be an integer), and `P` can't exceed `totalSum` (can't select a subset larger than everything). Any of these failing means zero ways, not a crash — I return 0 explicitly."

*[Code — 6 minutes]*
```cpp
int findTargetSumWays(vector<int>& nums, int target) {
    long long total = accumulate(nums.begin(), nums.end(), 0LL);
    long long need = (long long)target + total;
    if (need < 0 || need % 2 != 0) return 0;
    long long P = need / 2;
    if (P > total) return 0;

    vector<long long> dp(P + 1, 0);
    dp[0] = 1;
    for (int num : nums)
        for (long long s = P; s >= num; s--)
            dp[s] += dp[s - num];
    return (int)dp[P];
}
```

*[Follow-up]: "Why 0/1 and not unbounded — aren't all the numbers just 'items'?"*
> "Each array element is a physically distinct position that gets exactly one sign — it can't be 'reused.' Even if two elements share the same VALUE, they're separate items. That's what makes it 0/1, and why the capacity loop must run descending."

**RED FLAGS:**
- Jumping straight to code without deriving `P = (target+totalSum)/2` out loud
- Forgetting the parity/negativity/overflow guards (silent wrong answer, not a crash — hard to catch in review)
- Using `int` for `dp` when `total` can be large enough to matter, or not using `long long` for the `need`/`P` computation (int overflow when target and totalSum are both near their max magnitudes)

---

### Interview Simulation 3: Meta-discussion — Combinations vs Permutations Loop Order

**Interviewer:** "I give you two nearly identical problems: Coin Change II (count combinations) and Combination Sum IV (count permutations, despite the name). Same recurrence shape: `dp[a] += dp[a - x]`. Why does loop order alone distinguish them?"

**Candidate:**
> "Think about what 'freezing' an outer loop variable means. In Coin Change II, coins are the outer loop — I fully process coin type `1` (updating `dp` for every amount it can reach), THEN move to coin type `2`, and so on. Because coin `2` is only ever considered AFTER coin `1` is 'locked in,' any way I build always uses coins in a canonical order — never 'first some 2s then a 1' as a distinct way from 'first a 1 then some 2s.' They collapse into one combination.

> In Combination Sum IV, amount is the outer loop. At EVERY amount, I re-try every number as a candidate 'most recently appended element.' So building the sequence `(1, 2)` and `(2, 1)` are genuinely different execution paths through the DP table — `dp[3]` gets a contribution from `num=1` reading `dp[2]` (which itself was built ending in various numbers) AND from `num=2` reading `dp[1]`. Both paths survive independently, so both orderings get counted.

> The mental shortcut I use: **'coins outer' locks in a relative order across coin TYPES → combinations. 'Amount outer, items inner' lets EVERY position independently choose ANY item → permutations.**"

*[Interviewer]: "Prove it with the smallest possible counterexample."*
> "coins/nums = [1,2], target = 3. Combinations: only {1,1,1} and {1,2} — answer 2. Permutations: {1,1,1}, {1,2}, {2,1} — answer 3. The extra one, `(2,1)`, only appears when amount is outer, because at `a=3` we try `num=1` (reading `dp[2]`, which already accounts for the sequence ending at 2) — this path corresponds to appending a 1 after having reached 2 via a 2, i.e., the sequence `(2,1)`."

**RED FLAGS:**
- Describing the difference only as "swap the loops" without explaining the CAUSAL mechanism (what "freezing" an outer variable actually does to the search space)
- Being unable to produce a concrete counterexample on demand
- Assuming the fix is "just try both and see which matches expected output" — this is a testable, derivable fact, not something to brute-force in an interview

---

## SECTION 18 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why does 0/1 knapsack's space-optimized loop go capacity-DESCENDING, while unbounded knapsack's goes capacity-ASCENDING?**

> In 0/1 knapsack, each item is used at most once, so `dp[w]` after processing item `i` must only ever read values that do NOT already include item `i`. Iterating `w` from high to low guarantees that when we compute `dp[w]`, the cell `dp[w - wt[i]]` (which is strictly smaller than `w`) hasn't been touched yet this pass — it still holds the "before item `i`" value, exactly matching the 2D recurrence `dp[i-1][w - wt[i]]`.
>
> In unbounded knapsack, we WANT item `i` to be reusable within the same pass. Iterating `w` ascending means `dp[w - wt[i]]` may already have been updated earlier in this same pass (at a smaller `w`), which means it may already include item `i`. That's exactly what allows building an amount using multiple copies of the same coin — `dp[w]` can "build on top of" a `dp[w - wt[i]]` that itself used item `i`.

---

**Q2: In Coin Change II vs Combination Sum IV, both use `dp[a] += dp[a - x]`. What EXACTLY changes between them, and why does it produce combinations vs permutations?**

> Only the loop NESTING changes — Coin Change II has coins as the OUTER loop (amount inner, ascending); Combination Sum IV has amount as the OUTER loop (items inner).
>
> In Coin Change II, because coin type is fixed for an entire pass over all amounts, every "way" counted is implicitly built by deciding "how many of coin type 1, then how many of coin type 2, ..." in a FIXED coin-type order. Two orderings of the same multiset of coins (e.g., using a 2 before a 1, vs. a 1 before a 2) are the same combination and get merged into one count.
>
> In Combination Sum IV, at every amount, EVERY item is re-tried as a candidate "next element to append." This means the DP naturally distinguishes `(1, 2)` (built by appending 2 after reaching amount 1) from `(2, 1)` (built by appending 1 after reaching amount 2) — they're genuinely different paths through the table, so both get counted.

---

**Q3: In Target Sum, why must you check `(target + totalSum) % 2 != 0` BEFORE computing `P`, rather than just computing `P` and letting the DP naturally return 0 for an unreachable target?**

> Integer division in C++ truncates. If `target + totalSum` is odd, `(target + totalSum) / 2` silently produces a truncated (wrong) integer `P` instead of signaling "no valid P exists." The DP would then run against this WRONG `P` and could return a plausible-looking nonzero count for an input that should mathematically return exactly 0 ways.
>
> The explicit guard (`if (need % 2 != 0) return 0;`) converts a silent wrong-answer bug into a correct, explicit short-circuit — this is not just a style preference, it changes correctness.

---

**Q4: In Ones and Zeroes, why do BOTH dimensions of the 2D DP need to iterate descending, not just one of them?**

> Each string is a single 0/1 item that consumes BOTH budgets (zeros and ones) simultaneously — it's one indivisible unit of "weight" spread across two dimensions. If only one dimension iterated descending and the other ascending, the ascending dimension would allow the SAME string to be "reused" within a single pass along that dimension (reading a cell that itself already includes the current string), which is exactly the 0/1-vs-unbounded bug from Q1, just localized to one axis instead of the whole capacity space.
>
> Both dimensions must independently satisfy the "haven't been touched yet this pass" invariant, which requires both to go from high to low.

---

**Q5: Why is Tallest Billboard's state `dp[diff] = max shorter-pile height`, rather than something more "obvious" like `dp[sum] = ...`?**

> There's no target sum in this problem — the goal is to make two DYNAMICALLY built piles equal and as tall as possible, not to hit a fixed capacity. If we tried to track `dp[sum]`, we'd have no way to know how that sum splits between the two piles, which is the actual constraint we care about (equal split).
>
> Tracking the GAP between the two piles (`diff`) directly encodes the constraint we're trying to satisfy — we want `diff == 0` at the end. And tracking the shorter pile's height (rather than, say, the taller one, or the sum) lets us reconstruct both pile heights at any point: `tall = short + diff`. When `diff` returns to 0, `short` IS the answer — both piles are provably equal at that height, and we've maximized it because we always store the BEST (max) short-height seen for each diff.

---

**Q6: A candidate proposes solving Partition Equal Subset Sum with 2D `dp[i][s]` (no space optimization) instead of the 1D descending version. Is this correct? What's the tradeoff?**

> Yes, it's correct — the 1D version is a space optimization OF the 2D version, not a different algorithm. `dp[i][s] = dp[i-1][s] || dp[i-1][s - nums[i-1]]` is a perfectly valid 0/1 subset-sum DP.
>
> The tradeoff is purely space: O(n · sum) for the 2D table vs O(sum) for the 1D rolling version. Time complexity is identical, O(n · sum), in both cases. In an interview, starting with the 2D version to establish correctness, then space-optimizing as a follow-up, is a completely legitimate strategy — and often the safer one under time pressure, since the loop-direction subtlety (Q1) is much easier to get wrong when writing 1D code directly than when deriving it FROM a working 2D version.

---

**Q7: What is the single litmus-test question you should ask yourself before writing ANY knapsack-shaped DP, and why does it prevent the most common bug in this entire pattern?**

> "Can each item be used more than once?" If the answer is NO (0/1), the capacity loop must go descending when space-optimized. If YES (unbounded), it must go ascending.
>
> This single question prevents the #1 most common bug in the whole knapsack family: writing syntactically valid, plausible-looking code with the WRONG loop direction for the problem's actual reuse semantics. The bug produces no crash, no obviously wrong output shape — just a silently incorrect numeric answer, which is exactly the kind of bug that's easy to ship and hard to catch without deliberately asking this question first.

---

## SECTION 19 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. 0/1 KNAPSACK — VALUE MAXIMIZATION (1D, descending)
// ─────────────────────────────────────────────────────────────────
vector<long long> dp(W + 1, 0);
for (int i = 0; i < n; i++)
    for (long long w = W; w >= wt[i]; w--)          // DESCENDING — 0/1
        dp[w] = max(dp[w], dp[w - wt[i]] + val[i]);
// answer = dp[W]

// ─────────────────────────────────────────────────────────────────
// 2. UNBOUNDED KNAPSACK — MIN COST / MIN COINS (1D, ascending)
// ─────────────────────────────────────────────────────────────────
vector<int> dp(amount + 1, INT_MAX / 2);
dp[0] = 0;
for (int coin : coins)
    for (int a = coin; a <= amount; a++)            // ASCENDING — unbounded
        dp[a] = min(dp[a], dp[a - coin] + 1);
// answer = dp[amount] (or -1 if still INF)

// ─────────────────────────────────────────────────────────────────
// 3. SUBSET SUM / REACHABILITY (0/1, boolean, descending)
// ─────────────────────────────────────────────────────────────────
vector<bool> dp(target + 1, false);
dp[0] = true;
for (int num : nums)
    for (long long s = target; s >= num; s--)
        dp[s] = dp[s] || dp[s - num];
// answer = dp[target]

// ─────────────────────────────────────────────────────────────────
// 4. COUNT WAYS — COMBINATIONS (unbounded, coins OUTER)
// ─────────────────────────────────────────────────────────────────
vector<long long> dp(amount + 1, 0);
dp[0] = 1;
for (int coin : coins)
    for (int a = coin; a <= amount; a++)
        dp[a] += dp[a - coin];
// answer = dp[amount]     — order-independent multisets

// ─────────────────────────────────────────────────────────────────
// 5. COUNT WAYS — PERMUTATIONS (unbounded, amount OUTER)
// ─────────────────────────────────────────────────────────────────
vector<long long> dp(target + 1, 0);
dp[0] = 1;
for (int a = 1; a <= target; a++)
    for (int num : nums)
        if (num <= a) dp[a] += dp[a - num];
// answer = dp[target]     — order-dependent sequences

// ─────────────────────────────────────────────────────────────────
// 6. TARGET SUM TRANSFORM (P - N = target, P + N = total)
// ─────────────────────────────────────────────────────────────────
// P = (target + total) / 2  — GUARD before dividing:
//   if ((target + total) < 0 || (target + total) % 2 != 0) return 0;
//   if (P > total) return 0;
// Then run template #3 (subset-sum COUNT, not boolean) with capacity P:
//   dp[0] = 1; for num: for s = P downto num: dp[s] += dp[s - num];

// ─────────────────────────────────────────────────────────────────
// 7. 2D KNAPSACK (Ones and Zeroes — BOTH dims descending)
// ─────────────────────────────────────────────────────────────────
vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
for (auto& item : items) {
    auto [costA, costB] = cost(item);
    for (int i = m; i >= costA; i--)
        for (int j = n; j >= costB; j--)
            dp[i][j] = max(dp[i][j], dp[i - costA][j - costB] + 1);
}
// answer = dp[m][n]

// ─────────────────────────────────────────────────────────────────
// 8. KNAPSACK ON DIFFERENCE (Tallest Billboard)
// ─────────────────────────────────────────────────────────────────
unordered_map<long long, long long> dp; dp[0] = 0;
for (int r : rods) {
    auto next = dp;                                  // skip r
    for (auto& [d, shortH] : dp) {
        next[d + r] = max(next.count(d+r)?next[d+r]:-1, shortH);          // r → taller pile
        long long ns = shortH + min<long long>(d, r), nd = llabs(d - r);
        next[nd] = max(next.count(nd)?next[nd]:-1, ns);                    // r → shorter pile
    }
    dp = next;
}
// answer = dp[0]

// ─────────────────────────────────────────────────────────────────
// 9. THE ONE QUESTION TO ASK BEFORE WRITING ANY KNAPSACK LOOP
// ─────────────────────────────────────────────────────────────────
// "Can this item be reused?"
//   NO  (0/1)       → capacity loop DESCENDING when 1D
//   YES (unbounded) → capacity loop ASCENDING when 1D
// "Am I counting configurations, and does ORDER matter?"
//   NO  (combinations) → items OUTER, capacity INNER
//   YES (permutations) → capacity OUTER, items INNER
```

---

## SECTION 20 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Target Sum and Tallest Billboard are the two that require a genuine "aha" transform rather than template application — budget 45+ minutes for Tallest Billboard specifically.

2. **Drill the loop-direction reflex deliberately:** for every knapsack problem you solve from now on, before writing the capacity loop, say out loud "each item usable once or unlimited times?" and "descending or ascending?" Do this even when you're confident — this is the single highest-leverage habit in the whole pattern, because the bug it prevents is silent.

3. **Combination Sum IV immediately after Coin Change II:** solve them back-to-back, then write out — from memory, no looking back — the exact one-sentence reason their loop orders differ. If you can't do this crisply, re-read Section 8's interview simulation.

4. **The Target Sum → Partition Equal Subset Sum → Last Stone Weight II triangle:** these three problems are all "0/1 subset-sum in a trench coat." After solving all three, write down what's identical (the DP recurrence) and what's different (what quantity is being derived from the raw input to become the DP's target/capacity).

5. **Preview Pattern 20 (LCS Family) and Pattern 21 (LIS):** both continue the "state = dp[i] or dp[i][j], but the transition encodes a domain-specific insight" theme. Knapsack's lesson — "the loop direction and nesting order are themselves part of the algorithm's correctness, not just its performance" — will resurface in subtler forms throughout the rest of the DP curriculum (e.g., LIS's O(n log n) patience-sorting variant, Pattern 21).

6. **Contest note:** knapsack disguises (Partition Equal Subset Sum-style "notice the reduction" problems) are extremely common as Codeforces Div 2 C/D problems. Being able to spot "this is secretly subset-sum" or "this is secretly count-ways-with-order" within the first 2 minutes of reading a problem is a direct rating driver in the 1600-2000 band.

---

## SECTION 21 — SIGN-OFF CRITERIA

### Tier 1 — Core loop directions internalized
You write the 1D 0/1 knapsack (descending) and 1D unbounded knapsack (ascending) from memory in under 2 minutes each, and can state — without hesitation — WHY the directions differ.

### Tier 2 — Disguises recognized on sight
You solve Partition Equal Subset Sum and Coin Change cleanly. Given a new problem statement with no mention of "weight" or "value," you can identify within 2 minutes whether it's secretly a knapsack, and if so, what the capacity and items actually are.

### Tier 3 — Count-ways loop order mastered
You solve BOTH Coin Change II and Combination Sum IV, and can explain — using a concrete 3-element counterexample — exactly why swapping their loop nests changes the answer. You solve Target Sum including all three feasibility guards (parity, non-negativity, upper bound).

### Tier 4 — Hard variants and multi-dimensional knapsack solid
You solve Ones and Zeroes with both dimensions correctly descending. You solve Tallest Billboard using the knapsack-on-difference state and can explain the three-way transition (skip / taller / shorter) from first principles, not from memorized code.

**Sign-off threshold:** Solve 9 of 11 problems. Mandatory: LC 416 (Partition Equal Subset Sum — the foundational disguise), LC 518 AND LC 377 together (Coin Change II + Combination Sum IV — the loop-order distinction is non-negotiable), and LC 956 (Tallest Billboard — the hardest conceptual leap in the pattern). If you can derive all four without hints, you own the Knapsack family.

---

*Pattern 19 Complete — Knapsack Family*
*Next: Pattern 20 — LCS Family (Longest Common Subsequence, Edit Distance, Distinct Subsequences — two-sequence DP taken to completion)*
