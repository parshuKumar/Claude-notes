# PATTERN 16 — DP FUNDAMENTALS AND THE DP MINDSET
## The Meta-Pattern That Unlocks All of Phase 3
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**This is the most important document in the entire curriculum.**

Not because DP Fundamentals problems are hard. They aren't — Fibonacci, Climbing Stairs, House Robber are straightforward. The document is important because **how you think about DP determines whether you can solve ANY DP problem**, from 1D DP to bitmask DP to digit DP.

Most people fail at DP because they try to memorize solutions. A Fibonacci solution tells you nothing about Burst Balloons. A Coin Change solution doesn't help with Russian Doll Envelopes unless you understand the underlying framework.

**The DP framework that works for all 10 patterns (P16-P25):**

Before writing a single line of DP code, answer four questions:
1. **What is my state?** — What information uniquely describes a subproblem?
2. **What is my transition?** — How does the answer to the current state depend on smaller states?
3. **What are my base cases?** — What states can be answered directly (no recursion)?
4. **What is my answer?** — Which state (or combination of states) holds the final answer?

If you can answer all four clearly, writing the code is mechanical. If you can't, writing code first makes everything worse.

**The brutal truth about DP in interviews:**

Interviewers can tell within 30 seconds whether you truly understand DP or are pattern-matching. The signal is whether you can **define the state** before coding. If you say "let me just try dp[i] and see what it means," you've already failed the conceptual test.

**Time to mastery:** Pattern 16 is mostly conceptual — 3 days maximum. But the mental models built here will be tested across 10 more patterns. Return to this document after each DP pattern (17-25) to verify you still understand the fundamentals.

---

## SECTION 2 — ELI5: WHAT IS DYNAMIC PROGRAMMING?

Imagine you're climbing stairs. You can take 1 or 2 steps at a time. How many ways to reach step 10?

Brute force: try every possible sequence — but many sequences share sub-sequences. Computing "ways to reach step 9" and "ways to reach step 8" both require "ways to reach step 7." You're computing the same thing over and over.

**DP says:** Compute each subproblem ONCE, store the answer, reuse it.

That's it. DP is not a clever algorithm — it's an optimization strategy for problems that have:
1. **Overlapping subproblems** (same sub-question appears multiple times)
2. **Optimal substructure** (the answer to the big problem can be built from answers to small problems)

If both properties hold, DP gives you exponential speedup over brute force recursion.

---

## SECTION 3 — FORMALLY DEFINED

**Dynamic Programming** is a method for solving complex optimization/counting/decision problems by breaking them into simpler overlapping subproblems and storing computed results.

**Optimal Substructure:** The optimal solution to the problem contains optimal solutions to its subproblems. Example: the shortest path from A to C through B must use the shortest path from A to B AND from B to C.

**Overlapping Subproblems:** The same subproblems are solved multiple times. Example: in Fibonacci, `fib(5)` requires `fib(4)` and `fib(3)`, which both require `fib(2)` — computed twice without memoization.

**Two equivalent implementations:**

**Top-Down (Memoization):**
- Write the natural recursion
- Add a cache (`memo[]` or `unordered_map`) — before computing, check if already computed
- Return cached result if available

**Bottom-Up (Tabulation):**
- Fill a DP table starting from base cases
- Compute states in order such that all dependencies are computed before the current state
- No recursion, no stack overflow

**They are computationally equivalent** — same time complexity, same space complexity. The choice is:
- Top-down: easier to write for irregular state spaces, recursive thinking feels natural
- Bottom-up: avoids stack overflow, often has better constant factors, allows space optimization

---

## SECTION 4 — THE FOUR QUESTIONS FRAMEWORK IN DEPTH

### Question 1: What is my state?

The state is the **minimum information needed to uniquely identify a subproblem** such that its answer doesn't depend on how we got there — only on the state itself.

**Bad state definition:** "The answer for the current situation" — too vague.
**Good state definition:** `dp[i]` = minimum cost to reach step i (depends only on i, nothing else).

**State identification rules:**
- Start with the brute force recursive function signature — those parameters ARE your state
- Drop parameters that don't affect the answer (e.g., if the array doesn't change, don't include it)
- Add parameters only when the same index can have different answers depending on history

**Example — Climbing Stairs:**
- State: `dp[i]` = number of ways to reach step i
- Why not dp[i][last_step]? Because the number of ways to reach step i is the same regardless of whether we came from i-1 or i-2 (no constraint on consecutive steps). The last_step doesn't affect the count.

**Example — House Robber II (circular):**
- State: `dp[i]` = max money robbing houses 0..i with house i's choice being made
- But wait — do we need to know if house 0 was robbed? YES for the circular version.
- Solution: solve twice — once excluding house[0], once excluding house[n-1]. For each run, the state is just `dp[i]`.

**The state dimension rule of thumb:**
- 1D state: problem depends on one index/value (most 1D DP)
- 2D state: problem depends on two sequences, or position + constraint, or two quantities (LCS, knapsack, grid DP)
- 3D+ state: rare, but appears in problems like "Ones and Zeroes" (two resource constraints) or "Cherry Pickup" (two agents)

---

### Question 2: What is my transition?

The transition tells you: **given the answer to smaller states, how do I compute the current state?**

This is where the "choices at each state" mindset comes in. For each state, ask: "What was the LAST decision made before reaching this state?"

**Example — Climbing Stairs:**
- Last decision: jumped from step i-1 (took 1 step) OR jumped from step i-2 (took 2 steps)
- Transition: `dp[i] = dp[i-1] + dp[i-2]`
- WHY plus: each path ending at i-1 can extend to i, AND each path ending at i-2 can extend to i (independent, non-overlapping sets)

**Example — House Robber:**
- At house i, last decision: rob house i OR skip house i
- If rob: `dp[i] = dp[i-2] + nums[i]` (can't rob adjacent, so best from two houses back)
- If skip: `dp[i] = dp[i-1]` (best we can do from previous house)
- Transition: `dp[i] = max(dp[i-2] + nums[i], dp[i-1])`

**The "last decision" technique** always yields the correct transition because:
- We consider ALL possible last decisions
- For each, we assume the best solution to the remaining subproblem (optimal substructure)
- We take the best among all choices (max/min/sum depending on the goal)

**Common transition patterns:**
- `dp[i] = max(dp[i-1], dp[i-2] + something)` — take or skip
- `dp[i] = min(dp[j] + cost)` for j < i — optimal previous state
- `dp[i][j] = dp[i-1][j-1] + 1` if match — extend from smaller subproblem
- `dp[i] += dp[i-k]` for some k — sum over choices (unbounded knapsack)

---

### Question 3: What are my base cases?

Base cases are states that can be answered **directly without using the transition**. They anchor the recursion.

**Common base case patterns:**
- `dp[0] = 0` or `dp[0] = 1` — empty input or first element
- `dp[i][0] = i` — cost of matching with empty string (edit distance: delete all)
- `dp[0][j] = 0` — base condition for grid (no items to select)
- The "out of bounds" case — in recursion, these return a sentinel value (0, ∞, -∞)

**The base case test:** Your transition should NOT need to access `dp[-1]` or `dp[n+1]` without a guard. Base cases prevent invalid access.

**For bottom-up:** Pre-fill the dp array before the main loop. `dp[0] = 0` before iterating from 1.
**For top-down:** Add `if (state is base case) return base_value` as the first line of the recursive function.

---

### Question 4: What is my answer?

The final answer is usually one of:
- `dp[n]` or `dp[n-1]` — answer for the full input
- `max(dp[0..n])` — answer over all ending positions
- `dp[n][m]` — answer for two sequences of length n and m
- `dp[(1<<n) - 1]` — answer when all elements are processed (bitmask DP)

The answer question is often forgotten — people compute the dp table correctly but read the wrong cell.

**Example:** In Maximum Subarray (Kadane's), `dp[i]` = max subarray ending at i. The answer is `max(dp[0..n-1])` — NOT `dp[n-1]`, because the maximum subarray might not end at the last element.

---

## SECTION 5 — TOP-DOWN VS BOTTOM-UP: THE DECISION

```
Top-Down (Memoization):                Bottom-Up (Tabulation):

solve(state):                          Fill dp[0], dp[1], ..., dp[n]
  if memo[state] exists:               in order from base cases
    return memo[state]
  if base case:                        for i = base_case to n:
    return base_value                    dp[i] = f(dp[i-1], dp[i-2], ...)
  result = f(solve(substate1), ...)
  memo[state] = result                 answer = dp[n]
  return result
```

**When to choose Top-Down:**
- The state space is sparse (not all states are reachable) — memoization only computes reachable states
- The recursion is more natural than the iterative order
- State space has irregular shape (graph DP, tree DP)
- Interview: you're not sure of the exact iteration order — write recursion first, add memo

**When to choose Bottom-Up:**
- Large n (n > 10^4) — recursion stack overflow risk
- Need space optimization (rolling array) — only possible with iterative DP
- Performance matters — no function call overhead
- Clear iteration order exists (1D from left to right, 2D row by row)

**The space optimization pattern (Bottom-Up only):**

If `dp[i]` depends only on `dp[i-1]` and `dp[i-2]`, you don't need the full array:

```cpp
// Full array (O(n) space):
dp[0] = 0; dp[1] = 1;
for (int i = 2; i <= n; i++) dp[i] = dp[i-1] + dp[i-2];

// Space optimized (O(1) space):
int prev2 = 0, prev1 = 1;
for (int i = 2; i <= n; i++) {
    int curr = prev1 + prev2;
    prev2 = prev1;
    prev1 = curr;
}
```

This pattern applies whenever the transition only looks back a FIXED number of states.

---

## SECTION 6 — TEMPLATE 1: TOP-DOWN MEMOIZATION (FIBONACCI FAMILY)

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: Top-Down Memoization
// Problem: Fibonacci Number (LC 509) — baseline to understand the pattern
// ─────────────────────────────────────────────────────────────────

class Solution {
    unordered_map<int, int> memo;

    int fib(int n) {
        // Base cases
        if (n <= 1) return n;

        // Check memo BEFORE computing
        if (memo.count(n)) return memo[n];

        // Compute and STORE in memo BEFORE returning
        memo[n] = fib(n-1) + fib(n-2);
        return memo[n];
    }

public:
    int fib(int n) { return fib(n); }
};
```

**WHY checking memo before computing is essential:**

Without memoization, `fib(40)` makes ~10^9 recursive calls. With it, it makes exactly 40 unique calls (one per unique n value from 0 to 40), then returns cached results for all repeats.

**The call tree WITHOUT memo:**
```
fib(5)
├── fib(4)
│   ├── fib(3)
│   │   ├── fib(2)
│   │   │   ├── fib(1) = 1
│   │   │   └── fib(0) = 0
│   │   └── fib(1) = 1
│   └── fib(2)      ← RECOMPUTED (already done above)
│       ├── fib(1)
│       └── fib(0)
└── fib(3)           ← RECOMPUTED (already done above)
    ├── fib(2)       ← RECOMPUTED AGAIN
    └── fib(1)
```

**With memo:** fib(2) and fib(3) computed once, returned instantly for all subsequent calls.

**Using `vector<int> memo` for index-based states (faster than unordered_map):**

```cpp
class Solution {
    vector<int> memo;

    int solve(int n) {
        if (n <= 1) return n;
        if (memo[n] != -1) return memo[n];
        return memo[n] = solve(n-1) + solve(n-2);
    }

public:
    int fib(int n) {
        memo.assign(n + 1, -1);  // -1 = not computed
        return solve(n);
    }
};
```

**HOW TO ADAPT:**
- Change base cases and transition for any Fibonacci-like problem (Climbing Stairs, etc.)
- For 2D states: use `vector<vector<int>> memo(n, vector<int>(m, -1))`
- Sentinel value for "not computed": -1 works if answer is ≥ 0; use INT_MIN if answer can be negative

---

## SECTION 7 — TEMPLATE 2: BOTTOM-UP TABULATION (1D DP)

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: Bottom-Up 1D DP
// Problem: Climbing Stairs (LC 70)
// State: dp[i] = number of distinct ways to climb to step i
// Transition: dp[i] = dp[i-1] + dp[i-2]  (came from i-1 or i-2)
// Base cases: dp[1] = 1, dp[2] = 2
// Answer: dp[n]
// ─────────────────────────────────────────────────────────────────

int climbStairs(int n) {
    if (n <= 2) return n;

    vector<int> dp(n + 1);
    dp[1] = 1;  // 1 way to reach step 1: {1}
    dp[2] = 2;  // 2 ways to reach step 2: {1,1} or {2}

    for (int i = 3; i <= n; i++) {
        dp[i] = dp[i-1] + dp[i-2];
    }

    return dp[n];
}

// SPACE OPTIMIZED version:
int climbStairs(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; i++) {
        int c = a + b;
        a = b; b = c;
    }
    return b;
}
```

**FULL TRACE for n=5:**
```
State definition: dp[i] = ways to reach step i

Base cases: dp[1]=1, dp[2]=2

i=3: dp[3] = dp[2] + dp[1] = 2 + 1 = 3
  Paths: {1,1,1}, {1,2}, {2,1}

i=4: dp[4] = dp[3] + dp[2] = 3 + 2 = 5
  Paths: {1,1,1,1}, {1,1,2}, {1,2,1}, {2,1,1}, {2,2}

i=5: dp[5] = dp[4] + dp[3] = 5 + 3 = 8

Answer: 8
```

**WHY dp[i-1] + dp[i-2] and not something else:**
- To reach step i, the LAST step must have been either a 1-step (came from i-1) or a 2-step (came from i-2).
- Each distinct way to reach i-1 can be extended to i by adding a 1-step → dp[i-1] new paths.
- Each distinct way to reach i-2 can be extended to i by adding a 2-step → dp[i-2] new paths.
- These sets are disjoint (a path that last step was 1-step is distinct from one where last step was 2-step).
- Total = dp[i-1] + dp[i-2].

---

## SECTION 8 — TEMPLATE 3: THE "TAKE OR SKIP" PATTERN (HOUSE ROBBER)

This is one of the most important DP patterns. It appears everywhere: House Robber, Delete and Earn, Buy/Sell Stock, many knapsack problems.

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: Take or Skip
// Problem: House Robber (LC 198)
// State: dp[i] = max money robbing from houses 0..i
// Transition:
//   Rob house i:   dp[i] = dp[i-2] + nums[i]
//   Skip house i:  dp[i] = dp[i-1]
//   Combine:       dp[i] = max(dp[i-2] + nums[i], dp[i-1])
// Base cases: dp[0] = nums[0], dp[1] = max(nums[0], nums[1])
// Answer: dp[n-1]
// ─────────────────────────────────────────────────────────────────

int rob(vector<int>& nums) {
    int n = nums.size();
    if (n == 1) return nums[0];

    vector<int> dp(n);
    dp[0] = nums[0];
    dp[1] = max(nums[0], nums[1]);

    for (int i = 2; i < n; i++) {
        dp[i] = max(dp[i-2] + nums[i],  // rob house i
                    dp[i-1]);             // skip house i
    }

    return dp[n-1];
}

// SPACE OPTIMIZED (most common form in interviews):
int rob(vector<int>& nums) {
    int prev2 = 0, prev1 = 0;
    for (int num : nums) {
        int curr = max(prev1, prev2 + num);
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
```

**FULL TRACE for [2,7,9,3,1]:**
```
dp[0] = 2                    (only house 0 available)
dp[1] = max(2,7) = 7        (can't rob both 0 and 1, pick max)
dp[2] = max(dp[0]+9, dp[1]) = max(2+9, 7) = max(11, 7) = 11
dp[3] = max(dp[1]+3, dp[2]) = max(7+3, 11) = max(10, 11) = 11
dp[4] = max(dp[2]+1, dp[3]) = max(11+1, 11) = max(12, 11) = 12

Answer: 12 (rob houses 0, 2, 4: 2+9+1=12)
```

**WHY dp[i-2] and not dp[i-1] when robbing house i:**
Adjacent houses have alarms connected — can't rob house i AND house i-1. So if we rob house i, the BEST we can get from previous houses is dp[i-2] (best up to two houses back, definitely not adjacent to i).

**The "skip" option uses dp[i-1] NOT dp[i-2]:** When we skip house i, we include all possibilities up to house i-1 — including the option of robbing house i-1. So dp[i-1] (not dp[i-2]) is correct for the skip case.

---

## SECTION 9 — TEMPLATE 4: 2D DP (GRID PATH)

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: 2D DP on Grid
// Problem: Unique Paths (LC 62)
// State: dp[i][j] = number of unique paths from (0,0) to (i,j)
// Transition: dp[i][j] = dp[i-1][j] + dp[i][j-1]
//             (came from above OR came from left)
// Base cases: dp[i][0] = 1 for all i (only one way: go all down)
//             dp[0][j] = 1 for all j (only one way: go all right)
// Answer: dp[m-1][n-1]
// ─────────────────────────────────────────────────────────────────

int uniquePaths(int m, int n) {
    vector<vector<int>> dp(m, vector<int>(n, 1));  // fill with 1 (base cases)

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i-1][j] + dp[i][j-1];
        }
    }

    return dp[m-1][n-1];
}

// SPACE OPTIMIZED to O(n):
int uniquePaths(int m, int n) {
    vector<int> dp(n, 1);  // represents current row; initialized as top row (all 1s)
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j-1];  // dp[j] was dp[i-1][j], dp[j-1] is dp[i][j-1]
        }
    }
    return dp[n-1];
}
```

**TRACE for m=3, n=3:**
```
Initialize (all 1s since first row and col = 1):
dp = [[1,1,1],
      [1,0,0],
      [1,0,0]]

i=1, j=1: dp[1][1] = dp[0][1] + dp[1][0] = 1+1 = 2
i=1, j=2: dp[1][2] = dp[0][2] + dp[1][1] = 1+2 = 3
dp = [[1,1,1],
      [1,2,3],
      [1,0,0]]

i=2, j=1: dp[2][1] = dp[1][1] + dp[2][0] = 2+1 = 3
i=2, j=2: dp[2][2] = dp[1][2] + dp[2][1] = 3+3 = 6
dp = [[1,1,1],
      [1,2,3],
      [1,3,6]]

Answer: dp[2][2] = 6
```

**WHY we can compress 2D to 1D (rolling array):**
The transition `dp[i][j] = dp[i-1][j] + dp[i][j-1]` only uses the PREVIOUS ROW (`dp[i-1][j]`) and the CURRENT ROW to the left (`dp[i][j-1]`). After processing row i, row i-1 is never needed again. So one 1D array suffices: before the inner loop, `dp[j]` holds the row i-1 value; after `dp[j] += dp[j-1]`, it holds the row i value.

---

## SECTION 10 — TEMPLATE 5: DP WITH MEMOIZATION ON A WORD/STRING

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: String DP (Top-Down)
// Problem: Word Break (LC 139)
// State: dp[i] = can the first i characters of s be segmented?
// Transition: dp[i] = true if ANY dp[j]=true AND s[j..i-1] is in wordDict
//             for some j < i
// Base case: dp[0] = true (empty string can always be segmented)
// Answer: dp[s.size()]
// ─────────────────────────────────────────────────────────────────

bool wordBreak(string s, vector<string>& wordDict) {
    unordered_set<string> dict(wordDict.begin(), wordDict.end());
    int n = s.size();
    vector<bool> dp(n + 1, false);
    dp[0] = true;  // empty prefix is always valid

    for (int i = 1; i <= n; i++) {
        for (int j = 0; j < i; j++) {
            // If dp[j] is true AND s[j..i-1] is a valid word
            if (dp[j] && dict.count(s.substr(j, i - j))) {
                dp[i] = true;
                break;  // found one valid segmentation — no need to check more j values
            }
        }
    }

    return dp[n];
}
```

**TRACE for s="leetcode", wordDict=["leet","code"]:**
```
n=8, dp = [T,F,F,F,F,F,F,F,F]  (T=true, F=false)

i=1: j=0: dp[0]=T, s[0..0]="l" in dict? NO.
i=2: j=0: "le" in dict? NO. j=1: dp[1]=F, skip.
i=3: j=0: "lee" in dict? NO. ...
i=4: j=0: dp[0]=T, s[0..3]="leet" in dict? YES → dp[4]=T, break.

dp = [T,F,F,F,T,F,F,F,F]

i=5: j=0: "leetc" NO. j=1..3: dp=F. j=4: dp[4]=T, s[4..4]="c" NO.
i=6: j=4: dp[4]=T, s[4..5]="co" NO.
i=7: j=4: dp[4]=T, s[4..6]="cod" NO.
i=8: j=4: dp[4]=T, s[4..7]="code" in dict? YES → dp[8]=T, break.

Answer: dp[8] = true ✓
```

**WHY dp[0] = true:** The empty prefix is trivially segmentable (zero words needed). This base case enables the transition: when `j=0` and `dp[0]=true`, we check if the entire prefix `s[0..i-1]` is a valid word.

**Optimization — only check word-length substrings:** Instead of checking all j from 0 to i, only check j = i - len for each word of length len:

```cpp
for (int i = 1; i <= n; i++) {
    for (const string& word : wordDict) {
        int len = word.size();
        if (i >= len && dp[i - len] && s.substr(i - len, len) == word) {
            dp[i] = true;
            break;
        }
    }
}
```

---

## SECTION 11 — TEMPLATE 6: DP ON A CIRCULAR ARRAY (HOUSE ROBBER II)

A classic trick: solve two sub-problems and take the best.

```cpp
// LC 213 — House Robber II
// Houses arranged in a circle: first and last are adjacent.
// Can't rob both first and last.
// Trick: solve twice — once without first house, once without last house.

int robRange(vector<int>& nums, int start, int end) {
    int prev2 = 0, prev1 = 0;
    for (int i = start; i <= end; i++) {
        int curr = max(prev1, prev2 + nums[i]);
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}

int rob(vector<int>& nums) {
    int n = nums.size();
    if (n == 1) return nums[0];
    if (n == 2) return max(nums[0], nums[1]);

    // Case 1: exclude last house (rob from 0 to n-2)
    // Case 2: exclude first house (rob from 1 to n-1)
    return max(robRange(nums, 0, n-2),
               robRange(nums, 1, n-1));
}
```

**WHY two sub-problems are sufficient:**
The ONLY additional constraint over linear House Robber is: houses 0 and n-1 cannot BOTH be robbed.
- If we exclude house n-1 from consideration → optimal solution for houses 0..n-2 → may or may not include house 0, but that's fine (n-1 is excluded).
- If we exclude house 0 from consideration → optimal for houses 1..n-1 → fine.
One of these two cases contains the globally optimal solution, since in the optimal solution, at least one of house 0 or house n-1 is NOT robbed (they can't both be robbed). Taking the max covers all cases.

---

## SECTION 12 — TEMPLATE 7: KADANE'S ALGORITHM (MAXIMUM SUBARRAY)

```cpp
// LC 53 — Maximum Subarray
// State: dp[i] = maximum subarray SUM ending at index i
// Transition: dp[i] = max(nums[i], dp[i-1] + nums[i])
//             Start fresh at nums[i] OR extend previous subarray
// Base case: dp[0] = nums[0]
// Answer: max(dp[0..n-1])  ← NOT dp[n-1]

int maxSubArray(vector<int>& nums) {
    int maxSum = nums[0];
    int currentSum = nums[0];  // dp[i] — rolling variable

    for (int i = 1; i < nums.size(); i++) {
        currentSum = max(nums[i], currentSum + nums[i]);
        maxSum = max(maxSum, currentSum);
    }

    return maxSum;
}
```

**THE CRUCIAL INSIGHT — why `max(nums[i], dp[i-1] + nums[i])`:**

At each position i, we have two choices:
- **Start a new subarray at i:** The subarray is just `{nums[i]}`, with sum `nums[i]`.
- **Extend the previous subarray:** Add `nums[i]` to the best subarray ending at i-1.

We choose whichever is larger. If the previous subarray ended with a NEGATIVE total (`dp[i-1] < 0`), extending it makes things worse — we start fresh. If it ended positive, extending is beneficial.

**Equivalent reformulation:** `dp[i] = nums[i] + max(0, dp[i-1])`
This makes the logic explicit: add `nums[i]` and optionally add the positive part of the previous sum.

**TRACE for [-2,1,-3,4,-1,2,1,-5,4]:**
```
i=0: current = -2, maxSum = -2
i=1: current = max(1, -2+1) = max(1,-1) = 1, maxSum = 1
i=2: current = max(-3, 1+(-3)) = max(-3,-2) = -2, maxSum = 1
i=3: current = max(4, -2+4) = max(4,2) = 4, maxSum = 4
i=4: current = max(-1, 4+(-1)) = max(-1,3) = 3, maxSum = 4
i=5: current = max(2, 3+2) = max(2,5) = 5, maxSum = 5
i=6: current = max(1, 5+1) = max(1,6) = 6, maxSum = 6
i=7: current = max(-5, 6+(-5)) = max(-5,1) = 1, maxSum = 6
i=8: current = max(4, 1+4) = max(4,5) = 5, maxSum = 6

Answer: 6 (subarray [4,-1,2,1])
```

---

## SECTION 13 — RECOGNIZING DP VS GREEDY VS BACKTRACKING

This decision is critical and saves enormous time:

| Indicator | Likely approach |
|-----------|-----------------|
| "All possible" answers, small N (≤20) | Backtracking |
| "Optimal" (min/max/count) answer, N can be large | DP or Greedy |
| "Greedy choice property" holds (local best = global best) | Greedy |
| Exchange argument fails (local best ≠ global best) | DP |
| Overlapping subproblems + optimal substructure | DP |
| Problem involves choosing from a sequence to optimize | DP |
| Subsequences / subsets / partitions | DP |

**The greedy-vs-DP test:** Can I prove the greedy choice is always optimal by exchange argument?
- "Activity selection" (sort by end time): YES → greedy works
- "0/1 Knapsack" (pick items to maximize value under weight constraint): NO → DP needed
- "Fractional Knapsack" (can take fractions): YES → greedy works (take highest value/weight ratio first)

**The DP-vs-backtracking boundary:**
- Find if ANY solution exists → often DP
- Find ONE solution → backtracking or DP
- Find ALL solutions → backtracking (enumeration)
- Find count of solutions → DP (counting DP)

---

## SECTION 14 — THE SIX DP CATEGORIES AND THEIR KEY SIGNAL

After Pattern 16, you'll study 9 more DP patterns. Each has a key recognition signal:

| Pattern | Key Signal | Example |
|---------|-----------|---------|
| **P17: 1D DP** | "depends on i-1 or i-2" | House Robber, Decode Ways |
| **P18: 2D DP** | Two indices, grid paths | Unique Paths, Maximal Square |
| **P19: Knapsack** | "select items with capacity constraint" | Partition Equal Subset Sum |
| **P20: LCS family** | "two sequences, align them" | Edit Distance, LCS |
| **P21: LIS family** | "increasing subsequence" | Russian Doll Envelopes |
| **P22: DP on Trees** | "tree, postorder answer" | House Robber III |
| **P23: Interval DP** | "merge/remove from middle, dp[i][j]" | Burst Balloons |
| **P24: Bitmask DP** | "N ≤ 20, subsets as states" | TSP, Assignment |
| **P25: Digit DP** | "count numbers in range with property" | Count Special Integers |

When you see a DP problem, match it to one of these 9 categories first. Then apply that category's template.

---

## SECTION 15 — COMMON MISTAKES IN DP

### Mistake 1: Wrong state definition — state doesn't capture all relevant info

```cpp
// WRONG — for "Longest Increasing Subsequence with at most K replacements"
// dp[i] = length of LIS ending at index i
// But this doesn't capture how many replacements have been used!

// CORRECT — add the constraint as a dimension
// dp[i][k] = length of LIS ending at index i with exactly k replacements used
```

**HOW TO DETECT:** If two different situations can have the same "state" but require different future decisions, your state is too coarse. Add the differentiating information as a new dimension.

---

### Mistake 2: Off-by-one in state indexing (1-indexed vs 0-indexed)

```cpp
// WRONG — for Word Break, using 0-indexed dp where dp[i] = can segment s[0..i-1]
// Base case: dp[-1] = true — out of bounds!

// WRONG ATTEMPT to fix:
dp[0] = false;  // s[0..0-1] = s[0..-1] = empty string... but dp[0] means different things now

// CORRECT — use 1-indexed dp where dp[i] = can segment first i characters
// dp[0] = true (empty prefix)
// dp[i] = can segment s[0..i-1]
// Answer: dp[n]
```

**The 1-indexed shift trick:** For string/array DPs, define `dp[i]` as the answer for the first `i` elements (1-indexed). This gives a clean base case `dp[0] = true/0` for the empty case.

---

### Mistake 3: Reading wrong cell as answer

```cpp
// WRONG — Maximum Subarray
int maxSubArray(vector<int>& nums) {
    vector<int> dp(nums.size());
    dp[0] = nums[0];
    for (int i = 1; i < nums.size(); i++)
        dp[i] = max(nums[i], dp[i-1] + nums[i]);
    return dp[nums.size()-1];  // BUG: maximum might not end at last index!
}

// CORRECT — track maximum across all states
int maxSum = dp[0];
for (int i = 1; i < nums.size(); i++) {
    dp[i] = max(nums[i], dp[i-1] + nums[i]);
    maxSum = max(maxSum, dp[i]);  // update max at each step
}
return maxSum;
```

---

### Mistake 4: Iterating in wrong order (bottom-up)

```cpp
// WRONG — for 1D knapsack (each item used once), iterating forward allows reuse
for (int j = 0; j <= capacity; j++)  // WRONG direction for 0/1 knapsack
    if (j >= items[i]) dp[j] = max(dp[j], dp[j - items[i]] + value[i]);

// CORRECT — iterate backwards to prevent reuse of item i
for (int j = capacity; j >= items[i]; j--)  // backwards = item used at most once
    dp[j] = max(dp[j], dp[j - items[i]] + value[i]);

// CORRECT — iterate forwards for unbounded knapsack (can reuse items)
for (int j = items[i]; j <= capacity; j++)
    dp[j] = max(dp[j], dp[j - items[i]] + value[i]);
```

**The direction rule:** Backward = 0/1 (each item once). Forward = unbounded (items reusable). This will be covered deeply in Pattern 19, but establish the intuition now.

---

### Mistake 5: Forgetting to handle edge cases before DP

```cpp
// WRONG — House Robber on empty array
int rob(vector<int>& nums) {
    vector<int> dp(nums.size());
    dp[0] = nums[0];
    dp[1] = max(nums[0], nums[1]);  // CRASH if nums.size() == 1!
    ...
}

// CORRECT — check edge cases explicitly
int rob(vector<int>& nums) {
    if (nums.size() == 1) return nums[0];
    if (nums.size() == 2) return max(nums[0], nums[1]);
    // Now safe to set dp[0] and dp[1]
    ...
}
```

---

### Mistake 6: Incorrect memoization sentinel value

```cpp
// WRONG — using 0 as "not computed" when 0 is a valid answer
vector<int> memo(n, 0);  // BUG: "not computed" = 0 = valid answer "0 ways"

// CORRECT — use a sentinel that's NEVER a valid answer
vector<int> memo(n, -1);   // if answer is always ≥ 0
vector<int> memo(n, INT_MIN);  // if answer can be 0 but not INT_MIN
vector<bool> computed(n, false);  // explicit "computed" flag
```

---

## SECTION 16 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Fibonacci Number | 509 | Both top-down and bottom-up — baseline |
| 2 | Climbing Stairs | 70 | 1D DP, WHY dp[i] = dp[i-1]+dp[i-2] |
| 3 | Min Cost Climbing Stairs | 746 | Slight variant — min instead of count |

### CORE (solve in ≤ 20 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 4 | House Robber | 198 | Take-or-skip pattern, 2 choices per state |
| 5 | House Robber II | 213 | Circular constraint → two sub-problems |
| 6 | Unique Paths | 62 | 2D DP, space optimization to 1D |

### STRETCH (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Coin Change | 322 | Unbounded knapsack disguise, answer vs -1 |
| 8 | Longest Increasing Subsequence | 300 | O(n²) DP, state = "ending at i" |
| 9 | Word Break | 139 | String DP, 1-indexed, substr matching |

### REFERENCE

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Maximum Subarray | 53 | Kadane's = "extend or start fresh" |

---

## SECTION 17 — PATTERN CONNECTIONS

**Pattern 16 IS the foundation for P17-P25.** Every future DP pattern uses exactly the same four questions (state, transition, base cases, answer). The difference is:

- **P17 (1D DP):** Simple linear states, O(n) time
- **P18 (2D DP):** Two-index states, O(nm) time
- **P19 (Knapsack):** Items + capacity state, forward/backward iteration direction matters
- **P20 (LCS):** Two-string DP, `dp[i][j]` = best for first i chars of s1 and j chars of s2
- **P21 (LIS):** "Length of longest increasing subsequence ENDING at i" state
- **P22 (Tree DP):** State is a tree node, transition is postorder (children first)
- **P23 (Interval DP):** `dp[i][j]` = best for subarray `[i..j]`, try all split points k
- **P24 (Bitmask DP):** State includes a bitmask of which elements are used
- **P25 (Digit DP):** State includes current digit position + tight constraint + accumulator

**Connection to Pattern 15 (Backtracking):** Every DP problem can be solved by backtracking first (enumerate all solutions), but DP is the optimization when the problem has overlapping subproblems. Coin Change could be solved by backtracking (enumerate all ways to make change), but DP makes it O(amount × coins) instead of exponential.

**Connection to Pattern 05 (Binary Search):** Some problems combine binary search with DP — binary search on the answer, then verify with DP. LIS O(n log n) uses binary search as an optimization. Pattern 14's "Path With Minimum Effort" can be solved with binary search + BFS or directly with Dijkstra.

---

## SECTION 18 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 198 — House Robber

**Interviewer:** "You're a robber. Houses have money. Adjacent houses have alarms. Maximize total stolen."

**Candidate:**

*[30 seconds — apply four questions]*
> "State: dp[i] = max money robbing from houses 0..i. Transition: at house i, I either rob it (add nums[i], skip i-1, use dp[i-2]) or skip it (use dp[i-1]). Base cases: dp[0]=nums[0], dp[1]=max(nums[0],nums[1]). Answer: dp[n-1]."

*[Code — 3 minutes]*
```cpp
int rob(vector<int>& nums) {
    if (nums.size() == 1) return nums[0];
    int a = nums[0], b = max(nums[0], nums[1]);
    for (int i = 2; i < nums.size(); i++) {
        int c = max(b, a + nums[i]);
        a = b; b = c;
    }
    return b;
}
```

*[Follow-up]: "What if houses are in a circle?"*
> "The only new constraint is houses 0 and n-1 can't both be robbed. Solution: run the linear House Robber twice — once on houses 0..n-2, once on houses 1..n-1. Take the max. Both sub-problems are the same linear House Robber, which I just wrote. The optimal solution must have at least one of house 0 or house n-1 NOT robbed, so one of these two runs captures it."

**RED FLAGS:**
- Defining state as "which houses to rob" — too vague, leads to confusion
- Not articulating the two choices at each state before coding
- Writing `dp[1] = max(nums[0], nums[1])` without guarding against n=1

---

### Interview Simulation 2: LC 322 — Coin Change

**Interviewer:** "Coins of denominations [1,5,10,25]. Minimum coins to make amount = 30."

**Candidate:**

*[1 minute — four questions]*
> "State: dp[a] = minimum coins to make amount a. Transition: for each coin c, if a >= c, dp[a] = min(dp[a], dp[a-c] + 1) — use coin c as the last coin. Base case: dp[0] = 0. Answer: dp[amount] (or -1 if still INT_MAX)."

*[Identify this as unbounded knapsack before coding]*
> "This is unbounded knapsack — coins can be reused. The inner loop goes FORWARD (not backward) to allow reuse."

*[Code]*
```cpp
int coinChange(vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, amount + 1);  // sentinel > any valid answer
    dp[0] = 0;
    for (int a = 1; a <= amount; a++) {
        for (int c : coins) {
            if (a >= c) dp[a] = min(dp[a], dp[a-c] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```

*[Why `amount + 1` as sentinel?]* "The maximum coins needed is `amount` (use all 1-cent coins). So `amount + 1` is larger than any valid answer, making it a safe 'unreachable' sentinel. If dp[amount] is still `amount+1`, no valid combination exists → return -1."

**RED FLAGS:**
- Initializing with `INT_MAX` and then doing `INT_MAX + 1` → overflow
- Not considering that this is unbounded (same coin usable multiple times)
- Returning `dp[amount]` directly without checking for the sentinel

---

### Interview Simulation 3: Meta-interview — "Explain your DP thought process"

**Interviewer:** "Walk me through how you approach a DP problem you've never seen before."

**Candidate response:**

> "I follow four questions in order. First: What's my state? I ask what information uniquely determines a subproblem — usually it's the parameter(s) of the natural recursion. For linear problems it's one index; for two-sequence problems it's two indices; for subset problems it might be a bitmask.

> Second: What's my transition? I ask 'what was the last decision made before reaching this state?' Enumerating all possible last decisions gives all the ways dp[i] can be computed from smaller states. I take the min/max/sum over all those choices.

> Third: What are my base cases? These are states I can answer directly — usually empty input (dp[0] = 0 or 1), single elements, or the first row/column of a 2D table.

> Fourth: What is my answer? Often dp[n] or dp[n-1], but for problems like Maximum Subarray where the answer doesn't have to end at the last position, it's max(dp[0..n]).

> Once I can answer all four clearly, I write the code — either top-down (recursion + memo) or bottom-up (iterative table). Top-down is usually faster to write; bottom-up is better for space optimization."

**Why this response wins:** It shows a systematic framework, not memorized patterns. The interviewer is evaluating whether you can handle DP problems you haven't seen before.

---

## SECTION 19 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: What are the two properties that make a problem amenable to DP?**

> **Overlapping subproblems:** The same sub-question appears multiple times in the recursion tree. Without this, divide-and-conquer (like merge sort) suffices — DP's caching provides no benefit.
>
> **Optimal substructure:** The optimal solution to the problem is built from optimal solutions to its subproblems. Without this, even if subproblems overlap, the stored result might not be usable — the optimal sub-result might not contribute to the optimal overall result.
>
> Both must hold. If only optimal substructure holds (no overlap), use divide-and-conquer. If only overlap holds (no optimal substructure), you might need to enumerate all solutions.

---

**Q2: What's the difference between `dp[i]` and `dp[i-1]` in the House Robber recurrence `dp[i] = max(dp[i-2] + nums[i], dp[i-1])`?**

> `dp[i-2] + nums[i]`: We're robbing house i. Since we can't rob i-1 (adjacent), the best we can do from the remaining houses is `dp[i-2]` (optimal solution considering houses 0 to i-2). We add `nums[i]` for robbing house i.
>
> `dp[i-1]`: We're SKIPPING house i. The answer is simply the best we could do from houses 0 to i-1, which is `dp[i-1]`. Importantly, `dp[i-1]` might itself have robbed house i-1 or skipped it — we don't care. We just want the best up to i-1.
>
> The key: when skipping house i, we use `dp[i-1]` (best INCLUDING the option to rob house i-1), NOT `dp[i-2]` (which would unnecessarily restrict us from ever robbing house i-1).

---

**Q3: When should you prefer top-down memoization over bottom-up tabulation?**

> Prefer top-down when:
> - The state space is sparse (many states are unreachable) — memoization only computes states actually visited
> - The iteration order for bottom-up is non-obvious (requires topological analysis)
> - The recursion is natural and the iterative version would be complex
> - You're in an interview and want to write correct code quickly
>
> Prefer bottom-up when:
> - n is very large (> 10^5) — recursion risks stack overflow
> - You need space optimization (rolling array) — only possible iteratively
> - Performance is critical — no function call overhead
> - The problem has a natural left-to-right or top-to-bottom iteration order

---

**Q4: Why does `dp[i] = max(nums[i], dp[i-1] + nums[i])` correctly implement Kadane's?**

> At each position i, we're computing the maximum subarray sum that ENDS at i (required by the state definition).
>
> A subarray ending at i either:
> - Starts fresh at i: sum = `nums[i]`
> - Extends the best subarray ending at i-1: sum = `dp[i-1] + nums[i]`
>
> We take the max. If `dp[i-1] < 0`, extending is worse than starting fresh — `nums[i] > dp[i-1] + nums[i]` when `dp[i-1] < 0`.
>
> This is correct because we always consider the two exhaustive cases (start fresh vs extend), and optimal substructure guarantees that the optimal subarray ending at i must be one of these two forms.

---

**Q5: In Word Break, why is the 1-indexed dp formulation cleaner than 0-indexed?**

> With 0-indexed dp: `dp[i]` = can segment `s[0..i]`. The base case "can segment the empty string" would need `dp[-1] = true` — invalid index. We'd need special case handling.
>
> With 1-indexed dp: `dp[i]` = can segment the first i characters = `s[0..i-1]`. The base case "can segment zero characters" = `dp[0] = true` — a valid index with a natural meaning.
>
> The 1-indexed shift is a common DP trick: whenever the base case involves an "empty" object (empty prefix, empty subset, empty sequence), use 1-indexed dp so the base case maps to index 0.

---

**Q6: What is the "last decision" technique for deriving transitions, and why does it always work?**

> The technique: for any state, ask "what was the LAST decision made to arrive at this state?" Enumerate all possible last decisions. For each, the answer is: (cost of that last decision) + (optimal answer for the sub-state BEFORE that decision).
>
> It works because of optimal substructure: if we assume the sub-state before the last decision is solved optimally, then taking the best last decision gives the optimal answer for the current state. This is the formal definition of optimal substructure.
>
> Example: Coin Change. Last decision = which coin was used last. `dp[amount] = min over all coins c of (1 + dp[amount - c])`. The "1" is the cost of using coin c; `dp[amount-c]` is the optimal sub-state.

---

**Q7: If a problem asks "count the number of ways" rather than "find the minimum/maximum," does DP still apply?**

> Yes — counting DP is equally valid. The only change is in the transition: instead of `min`/`max`, you SUM over all choices.
>
> Example: Climbing Stairs (count ways) vs Min Cost Climbing Stairs (minimum cost). The state is the same, but:
> - Count: `dp[i] = dp[i-1] + dp[i-2]` (sum all ways)
> - Min cost: `dp[i] = min(dp[i-1] + cost[i-1], dp[i-2] + cost[i-2])` (take best)
>
> For counting, you often need modular arithmetic (`% 1e9+7`) to prevent overflow. This is a common pattern in contest problems: "count the number of valid arrangements modulo 10^9+7."
>
> The key property is the same — overlapping subproblems + optimal (or "complete") substructure, where "complete" means "counting all ways to reach this state is the sum of ways to reach predecessor states."

---

## SECTION 20 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// TOP-DOWN MEMOIZATION — general template
// ─────────────────────────────────────────────────────────────────
vector<int> memo(N + 1, -1);
int solve(int state) {
    if (is_base_case(state)) return base_value(state);
    if (memo[state] != -1) return memo[state];
    int result = /* f(solve(smaller_state1), solve(smaller_state2), ...) */;
    return memo[state] = result;
}

// 2D memoization:
vector<vector<int>> memo(N, vector<int>(M, -1));
int solve(int i, int j) {
    if (base case) return base_value;
    if (memo[i][j] != -1) return memo[i][j];
    return memo[i][j] = /* transition */;
}

// ─────────────────────────────────────────────────────────────────
// BOTTOM-UP — 1D template
// ─────────────────────────────────────────────────────────────────
vector<int> dp(n + 1, initial_value);
dp[0] = base_case_0;
dp[1] = base_case_1;  // if needed
for (int i = 2; i <= n; i++) {
    dp[i] = /* f(dp[i-1], dp[i-2], ...) */;
}
return dp[n];  // or max/min of dp[0..n]

// ─────────────────────────────────────────────────────────────────
// SPACE OPTIMIZATION — O(n) → O(1) (when only last k states needed)
// ─────────────────────────────────────────────────────────────────
int prev2 = base0, prev1 = base1;
for (int i = 2; i <= n; i++) {
    int curr = /* f(prev1, prev2) */;
    prev2 = prev1;
    prev1 = curr;
}
return prev1;

// ─────────────────────────────────────────────────────────────────
// FOUR QUESTIONS — fill before writing any DP
// ─────────────────────────────────────────────────────────────────
// 1. STATE:      dp[?] = what does this represent?
// 2. TRANSITION: dp[i] = f(dp[i-1], dp[i-2], ...) — LAST DECISION
// 3. BASE CASES: dp[0] = ?, dp[1] = ?
// 4. ANSWER:     dp[n]? max(dp[0..n])? dp[n][m]?

// ─────────────────────────────────────────────────────────────────
// SENTINEL VALUES — choose based on valid answer range
// ─────────────────────────────────────────────────────────────────
// Minimum problems: INT_MAX/2 (safe from overflow), "amount+1" for coin change
// Maximum problems: INT_MIN or 0 (if answer is non-negative)
// Count problems: 0 (then dp[0] = 1 for empty base case)
// Boolean: false (then dp[0] = true)

// ─────────────────────────────────────────────────────────────────
// CIRCULAR ARRAY TRICK (House Robber II pattern)
// ─────────────────────────────────────────────────────────────────
// Can't use both endpoints → solve twice:
max(solve(arr, 0, n-2),   // exclude last
    solve(arr, 1, n-1));  // exclude first
```

---

## SECTION 21 — WHAT TO DO AFTER THIS PATTERN

1. **Immediately solve (in order):** LC 509 → 70 → 746 → 198 → 213 → 62 → 322 → 300 → 139 → 53

2. **Write the four questions for EACH problem:** Don't just code — explicitly write out: state definition, transition derivation (using "last decision"), base cases, and where to read the answer. This is the practice that builds the skill.

3. **Verify you can explain each transition:** For every DP you write, be able to answer: "Why is this transition correct and not some other formula?" If you can't explain it, you memorized rather than understood.

4. **Preview Pattern 17 (1D DP):** After this pattern, the 1D DP problems (Decode Ways, Jump Game, Delete and Earn) will use exactly the same framework. The state will always be "dp[i] = something about the first i elements."

5. **The DP mindset practice (ongoing):** For any optimization/counting problem you see, immediately ask: what's the state? what's the transition? If you can answer in under 30 seconds, you've internalized the framework.

---

## SECTION 22 — SIGN-OFF CRITERIA

### Tier 1 — Framework understood
You can recite the four questions (state, transition, base cases, answer) from memory and explain why each is necessary. You know the difference between top-down and bottom-up and when to use each.

### Tier 2 — Basic patterns mechanical
You solve LC 70 (Climbing Stairs) and LC 198 (House Robber) without reference. The space-optimized version (rolling two variables) is natural. You can write `dp[i] = max(dp[i-2] + nums[i], dp[i-1])` and explain BOTH terms.

### Tier 3 — Can handle variations
You solve House Robber II (circular) by reducing to two linear sub-problems. You solve Coin Change and correctly use `amount+1` as sentinel (not INT_MAX). You solve Word Break with 1-indexed dp and explain the base case `dp[0]=true`.

### Tier 4 — Can apply framework to new problems
Given a new DP problem, you can define the state correctly on the first try (not just "let me try dp[i] and see"). You use the "last decision" technique to derive transitions. You know which of the 9 DP categories (P17-P25) the problem belongs to within 30 seconds.

**Sign-off threshold:** Solve all 10 problems. More importantly: for each problem, write the four questions BEFORE coding and verify your answers match your code. The code matters less than the conceptual clarity.

---

*Pattern 16 Complete — DP Fundamentals*
*Phase 3 (Dynamic Programming) begins here. This is the hardest phase. Budget extra time.*
*Next: Pattern 17 — 1D DP (House Robber family, Decode Ways, Jump Game, Word Break)*
