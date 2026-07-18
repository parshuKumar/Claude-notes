# PATTERN 23 — INTERVAL DP
## Split-Point DP, Burst/Merge Problems, Matrix Chain, Palindrome Partitioning — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of interval DP:**

Every problem in this pattern shares one shape: `dp[i][j]` is the answer for the **subrange `[i..j]`** of an array or string — not "first i elements" (Pattern 16/17 territory), not "path to cell (i,j)" (Pattern 18 territory), but a genuine *contiguous slice* that shrinks and grows as you choose where to split it.

That single fact creates three real difficulties:

1. **The transition looks innocent but has a hidden dependency graph.** `dp[i][j] = min over k in (i,j) of dp[i][k] + dp[k][j] + cost(i,k,j)` reads like a one-liner. But `dp[i][k]` and `dp[k][j]` are BOTH narrower intervals than `[i,j]`, and `k` ranges over every possible split point — this is an O(n) inner loop inside an O(n²) outer loop, giving O(n³) total. Miss this and you'll either loop in a broken order (Section 3) or write an exponential brute force that "coincidentally" passes on tiny test cases.

2. **"Which is LAST" beats "which is FIRST."** In Burst Balloons, Matrix Chain, Polygon Triangulation — the natural instinct is to ask "what do I do first?" That question makes the two resulting subproblems *dependent* on each other (bursting balloon A first changes balloon B's neighbors). The correct question is "what happens LAST in this interval?" That question makes the two subproblems *independent*, because everything else is already gone/decided by the time the last operation happens. This reframe is the single most important idea in this entire pattern.

3. **The loop order is not optional — it's a correctness requirement.** You must fill `dp[i][j]` for every interval of length `L` before any interval of length `L+1` that depends on it. There is exactly one degree of freedom in how you achieve this (by length ascending is the cleanest; a decreasing-i/increasing-j sweep also works for adjacent-dependency recurrences), and getting it wrong silently reads uninitialized/stale values instead of crashing — which makes it a brutal bug to catch in a 45-minute interview.

**The key skill:**

Before coding any interval DP problem, answer:
1. What does `dp[i][j]` mean — inclusive of both endpoints (Matrix Chain, LPS, Stone Game), or are `i,j` untouched BOUNDARIES with the real content strictly between them (Burst Balloons, Cut Stick)?
2. What is the LAST thing that happens inside `[i,j]` — a burst, a cut, a merge, a triangle, a print stroke? Enumerate all choices for that last event as your split/decision variable `k`.
3. Does the transition need only ADJACENT intervals (`dp[i+1][j]`, `dp[i][j-1]`) or ANY split point `k` (`dp[i][k] + dp[k][j]`)? The former is O(n²), the latter is O(n³).

If you answer these three correctly, the nested-loop skeleton is always the same.

---

## SECTION 2 — THE INTERVAL DP TAXONOMY

### Type 1: Split-Point Enumeration (the archetype)
State: `dp[i][j]` = optimal cost/value to fully resolve `[i..j]`
Transition: `dp[i][j] = opt over k in (i,j) of dp[i][k] + dp[k][j] + cost(i,k,j)`
Problems: Matrix Chain Multiplication, Minimum Score Triangulation of Polygon

### Type 2: Burst / Remove-From-Middle
State: `dp[i][j]` = max/min value from removing everything STRICTLY BETWEEN boundary markers `i` and `j`
Key insight: think about the LAST element removed in `(i,j)`, not the first
Problems: Burst Balloons, Minimum Cost to Cut a Stick

### Type 3: Matrix-Chain-Style Cost Aggregation
State: `dp[i][j]` = min cost to merge/combine a chain of items `i..j` into one
Transition: identical shape to Type 1, but the "cost" term comes from a K-way merge or associative combination rule
Problems: Minimum Cost to Merge Stones (K-way generalization)

### Type 4: Palindrome Partitioning
State: `dp[i][j]` = palindrome-related property of substring `s[i..j]` (is-palindrome, longest palindromic subsequence, or a secondary 1D DP built on top of a precomputed interval table)
Transition: depends on `s[i] == s[j]` plus the inner interval `dp[i+1][j-1]`
Problems: Longest Palindromic Subsequence, Palindrome Partitioning II, Strange Printer

### Type 5: Game Theory Intervals
State: `dp[i][j]` = best achievable SCORE DIFFERENCE (current player minus opponent) over the subarray `[i..j]`
Transition: `dp[i][j] = max(a[i] - dp[i+1][j], a[j] - dp[i][j-1])` — taking an end flips whose turn dp represents
Problems: Stone Game and its many variants (Predict the Winner, Stone Game II/III)

### Type 6: State-Space Search With Interval Collapse (the edge case)
State: not a clean `dp[i][j]` table — a memoized recursion over board/hand configurations, but the core sub-operation ("remove a contiguous run, merge the boundaries that touch") is the same idea as Type 2
Problems: Zuma Game

---

## SECTION 3 — CRITICAL: THE LOOP ORDER (WHY LENGTH-ASCENDING IS MANDATORY)

Every interval DP transition needs `dp[i][j]` to be computed only AFTER every strictly narrower interval it depends on. Concretely, for the two dominant recurrence shapes in this pattern:

```
Shape A (adjacent-only):  dp[i][j] = f(dp[i+1][j], dp[i][j-1], dp[i+1][j-1])
Shape B (any split point): dp[i][j] = opt over k in (i,j) of dp[i][k] + dp[k][j] + cost(...)
```

In BOTH shapes, every term on the right-hand side is an interval strictly shorter than `[i,j]`. That is the ONLY guarantee you have — you do NOT know whether the shorter interval sits in an earlier row or a later row of a naive 2D loop.

**THE WRONG ORDER — standard row-major nested loop:**

```cpp
// WRONG — looks like ordinary 2D DP, but the dependency graph is different
for (int i = 0; i < n; i++) {
    for (int j = i + 1; j < n; j++) {
        dp[i][j] = f(dp[i+1][j], dp[i][j-1], dp[i+1][j-1]);
        //              ^^^^^^^ row i+1 has NOT been touched yet —
        //              the outer loop is still sitting at row i!
        //              This reads GARBAGE (or a stale default value).
    }
}
```

**CONCRETE FAILURE TRACE, n = 4, computing LPS-style `dp[i][j] = f(dp[i+1][j], ...)`:**

```
Outer loop i=0:
  j=1: needs dp[1][1] (fine, base case) ✓
  j=2: needs dp[1][2] ← UNCOMPUTED (row 1 hasn't run yet). Reads 0 (garbage).
  j=3: needs dp[1][3] ← UNCOMPUTED. Reads 0 (garbage).

Outer loop i=1 (runs AFTER i=0, too late — dp[0][2] and dp[0][3]
already used the wrong dp[1][2], dp[1][3] and can never be recomputed):
  j=2: dp[1][2] computed NOW — but dp[0][2] already consumed the old value!
  j=3: dp[1][3] computed NOW — same problem for dp[0][3].

RESULT: dp[0][2] and dp[0][3] are silently wrong. No crash. No exception.
Just an answer that's off by some amount on specific inputs — the worst
kind of bug because small test cases (n ≤ 2) don't expose it at all.
```

**THE CORRECT ORDER — by increasing interval length:**

```cpp
for (int len = 2; len <= n; len++) {           // ALL length-1 intervals done as base case
    for (int i = 0; i + len - 1 < n; i++) {
        int j = i + len - 1;
        // dp[i+1][j]   has length (len-1) < len   → already computed
        // dp[i][j-1]   has length (len-1) < len   → already computed
        // dp[i+1][j-1] has length (len-2) < len   → already computed
        // dp[i][k], dp[k+1][j] for any split k     → both < len     → already computed
        dp[i][j] = /* transition */;
    }
}
```

**WHY length-ascending is the ONLY loop shape that works for BOTH recurrence types (A and B) without extra reasoning:** regardless of where the split point `k` lands, `dp[i][k]` and `dp[k][j]` are always strictly shorter than `dp[i][j]`. Iterating by length guarantees "shorter is always done first," full stop — no need to separately verify a row/column sweep direction for every new problem.

**A valid ALTERNATIVE (worth recognizing, not preferring):** `for (int i = n-1; i >= 0; i--) for (int j = i+1; j < n; j++)` also works, because by the time you reach a given `i`, every row `i+1..n-1` is FULLY finished (all columns), and within row `i` you sweep `j` left to right so `dp[i][j-1]` is ready. This handles Shape A and Shape B correctly too, but it is easy to get subtly wrong when adapted to problems with extra boundary offsets (Burst Balloons' sentinels, Merge Stones' K layers) — **length-ascending is the recommended default for this entire pattern** because it makes the "is this interval ready?" question a single, uniform comparison (`length < len`).

---

## SECTION 4 — TEMPLATE 1: LONGEST PALINDROMIC SUBSEQUENCE (INTERVAL FORM)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 516 — Longest Palindromic Subsequence
// Given string s, find the length of the longest palindromic
// subsequence (not necessarily contiguous).
//
// State:    dp[i][j] = length of the longest palindromic subsequence
//                       within s[i..j] (inclusive both ends)
// Transition:
//   if s[i] == s[j]: dp[i][j] = dp[i+1][j-1] + 2   (both ends extend it)
//   else:            dp[i][j] = max(dp[i+1][j], dp[i][j-1])
// Base case: dp[i][i] = 1 (a single char is a palindrome of length 1)
//            dp[i+1][j-1] treated as 0 when i+1 > j-1 (len == 2 case)
// Answer: dp[0][n-1]
// ─────────────────────────────────────────────────────────────────

int longestPalindromeSubseq(string s) {
    int n = s.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = 0; i < n; i++) dp[i][i] = 1;      // length-1 base case

    for (int len = 2; len <= n; len++) {            // ascending length — MANDATORY
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            if (s[i] == s[j]) {
                int inner = (len == 2) ? 0 : dp[i+1][j-1];
                dp[i][j] = inner + 2;
            } else {
                dp[i][j] = max(dp[i+1][j], dp[i][j-1]);
            }
        }
    }
    return dp[0][n-1];
}
```

**FULL TRACE for s = "bbbab" (expected answer: 4, subsequence "bbbb"):**
```
Indices: 0=b 1=b 2=b 3=a 4=b
Base:    dp[i][i] = 1 for all i

len=2:
  (0,1) b,b equal → dp=2      (1,2) b,b equal → dp=2
  (2,3) b,a differ → max(dp[3][3]=1, dp[2][2]=1) = 1
  (3,4) a,b differ → max(dp[4][4]=1, dp[3][3]=1) = 1

len=3:
  (0,2) b,b equal → dp[1][1]+2 = 1+2 = 3
  (1,3) b,a differ → max(dp[2][3]=1, dp[1][2]=2) = 2
  (2,4) b,b equal → dp[3][3]+2 = 1+2 = 3

len=4:
  (0,3) b,a differ → max(dp[1][3]=2, dp[0][2]=3) = 3
  (1,4) b,b equal → dp[2][3]+2 = 1+2 = 3

len=5:
  (0,4) b,b equal → dp[1][3]+2 = 2+2 = 4

Answer: dp[0][4] = 4 ✓
```

---

## SECTION 5 — TEMPLATE 2: PALINDROME PARTITIONING II

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 132 — Palindrome Partitioning II
// Minimum number of cuts so every resulting substring of s is a
// palindrome.
//
// STEP 1 — interval DP to precompute isPalin[i][j] (length ascending):
//   isPalin[i][j] = (s[i]==s[j]) && (len==2 || isPalin[i+1][j-1])
//
// STEP 2 — 1D DP BUILT ON TOP of the interval table:
//   cuts[i] = minimum cuts needed for s[0..i]
//   cuts[i] = 0                                  if isPalin[0][i]
//   cuts[i] = min over j in [0,i) with isPalin[j+1][i] of cuts[j] + 1
// Answer: cuts[n-1]
//
// THIS IS A COMMON PATTERN: an interval DP table used as a fast
// O(1)-lookup oracle ("is this substring a palindrome?") feeding a
// SEPARATE, simpler 1D DP. Don't assume every palindrome problem
// needs 2D DP for the FINAL answer — often only the ORACLE is 2D.
// ─────────────────────────────────────────────────────────────────

int minCut(string s) {
    int n = s.size();
    vector<vector<bool>> isPalin(n, vector<bool>(n, false));
    for (int i = 0; i < n; i++) isPalin[i][i] = true;

    for (int len = 2; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            if (s[i] == s[j] && (len == 2 || isPalin[i+1][j-1]))
                isPalin[i][j] = true;
        }
    }

    vector<int> cuts(n, INT_MAX);
    for (int i = 0; i < n; i++) {
        if (isPalin[0][i]) { cuts[i] = 0; continue; }
        for (int j = 0; j < i; j++) {
            if (isPalin[j+1][i] && cuts[j] != INT_MAX)
                cuts[i] = min(cuts[i], cuts[j] + 1);
        }
    }
    return cuts[n-1];
}
```

**TRACE for s = "aab" (expected answer: 1, partition "aa"|"b"):**
```
isPalin base: all dp[i][i] = true

len=2: (0,1) a,a equal → isPalin[0][1] = true
       (1,2) a,b differ → isPalin[1][2] = false

len=3: (0,2) a,b differ → isPalin[0][2] = false

cuts[0]: isPalin[0][0]=true → cuts[0] = 0
cuts[1]: isPalin[0][1]=true → cuts[1] = 0
cuts[2]: isPalin[0][2]=false
         j=0: isPalin[1][2]=false → skip
         j=1: isPalin[2][2]=true → cuts[1]+1 = 0+1 = 1 → cuts[2] = 1

Answer: cuts[2] = 1 ✓
```

---

## SECTION 6 — TEMPLATE 3: BURST BALLOONS (THE CLASSIC)

This is the problem that defines "which is LAST, not FIRST" thinking.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 312 — Burst Balloons
// n balloons in a row, nums[i] on each. Bursting balloon i earns
// nums[left] * nums[i] * nums[right], where left/right are i's
// CURRENT neighbors AT THE MOMENT OF BURSTING (already-burst
// balloons don't count as neighbors). Maximize total coins from
// bursting ALL balloons.
//
// WRONG QUESTION: "which balloon do I burst FIRST?" — bursting
// balloon k first changes the neighbor sets for every other choice,
// so the two resulting sub-problems (left of k, right of k) are NOT
// independent. This leads to an unusable, order-dependent recursion.
//
// RIGHT QUESTION: "which balloon do I burst LAST within [i,j]?"
// If balloon k is the LAST one burst in the OPEN interval (i,j),
// then at the instant it bursts, every other balloon strictly
// between i and j is already gone — so k's neighbors are EXACTLY
// the boundary balloons i and j. This makes the two sub-problems,
// "burst everything in (i,k)" and "burst everything in (k,j)",
// completely INDEPENDENT of each other and of the order within them.
//
// Add sentinel balloons of value 1 at both ends so index 0 and n+1
// always exist as safe boundary walls: a = [1, nums[0..n-1], 1]
//
// State: dp[i][j] = max coins from bursting ALL balloons STRICTLY
//                    BETWEEN i and j (i, j themselves are NEVER
//                    burst — they are the boundary walls, still alive)
// Transition: dp[i][j] = max over k in (i,j) of
//               dp[i][k] + dp[k][j] + a[i]*a[k]*a[j]
// Base case: dp[i][j] = 0 when j == i+1 (no balloon strictly between)
// Answer: dp[0][n+1]
// ─────────────────────────────────────────────────────────────────

int maxCoins(vector<int>& nums) {
    int n = nums.size();
    vector<int> a(n + 2, 1);              // a[0] = a[n+1] = 1 (sentinels)
    for (int i = 0; i < n; i++) a[i + 1] = nums[i];

    vector<vector<int>> dp(n + 2, vector<int>(n + 2, 0));

    for (int gap = 2; gap <= n + 1; gap++) {        // gap = j - i, ascending
        for (int i = 0; i + gap <= n + 1; i++) {
            int j = i + gap;
            for (int k = i + 1; k < j; k++) {        // k = LAST balloon burst in (i,j)
                int coins = dp[i][k] + dp[k][j] + a[i] * a[k] * a[j];
                dp[i][j] = max(dp[i][j], coins);
            }
        }
    }
    return dp[0][n + 1];
}
```

**FULL TRACE for nums = [3,1,5,8] (expected answer: 167):**
```
a = [1, 3, 1, 5, 8, 1]   (indices 0..5, a[0] and a[5] are sentinels)

gap=2:
  dp[0][2]: k=1 → dp[0][1]+dp[1][2]+1*3*1 = 0+0+3   = 3
  dp[1][3]: k=2 → dp[1][2]+dp[2][3]+3*1*5 = 0+0+15  = 15
  dp[2][4]: k=3 → dp[2][3]+dp[3][4]+1*5*8 = 0+0+40  = 40
  dp[3][5]: k=4 → dp[3][4]+dp[4][5]+5*8*1 = 0+0+40  = 40

gap=3:
  dp[0][3]: k=1 → 0+15+1*3*5=30 ; k=2 → 3+0+1*1*5=8   → max=30
  dp[1][4]: k=2 → 0+40+3*1*8=64 ; k=3 → 15+0+3*5*8=135 → max=135
  dp[2][5]: k=3 → 0+40+1*5*1=45 ; k=4 → 40+0+1*8*1=48  → max=48

gap=4:
  dp[0][4]: k=1 → 0+135+1*3*8=159
            k=2 → 3+40+1*1*8=51
            k=3 → 30+0+1*5*8=70
            max = 159
  dp[1][5]: k=2 → 0+48+3*1*1=51
            k=3 → 15+40+3*5*1=70
            k=4 → 135+0+3*8*1=159
            max = 159

gap=5:
  dp[0][5]: k=1 → 0+159+1*3*1=162
            k=2 → 3+48+1*1*1=52
            k=3 → 30+40+1*5*1=75
            k=4 → 159+0+1*8*1=167   ← burst balloon at k=4 (value 8) LAST
            max = 167

Answer: dp[0][5] = 167 ✓
```

**WHY the sentinels matter:** without `a[0]=1` and `a[n+1]=1`, the boundary terms `a[i]*a[k]*a[j]` for `i=0` or `j=n+1` would need special-casing every single time. The sentinel trick lets the SAME formula handle the true edges of the array as if they were ordinary (never-bursting) balloons.

---

## SECTION 7 — TEMPLATE 4: MINIMUM COST TO CUT A STICK

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1547 — Minimum Cost to Cut a Stick
// Stick of length n. cuts[] gives positions to cut (any order).
// Cutting a segment costs its CURRENT length. Minimize total cost
// to perform all cuts.
//
// Add boundary points 0 and n to cuts, sort everything — this turns
// the problem into the SAME shape as Matrix Chain Multiplication:
// choosing which cut point divides a segment, one split at a time.
//
// State: dp[i][j] = min cost to perform every cut strictly between
//                    sorted point index i and index j (points[i] and
//                    points[j] are the segment's current physical
//                    endpoints, already-decided boundaries)
// Transition: dp[i][j] = min over k in (i,j) of
//               dp[i][k] + dp[k][j] + (points[j] - points[i])
//             (cutting at points[k] on the current segment costs the
//              FULL length of the segment [points[i], points[j]] —
//              that length doesn't shrink until the cut actually
//              happens, so it's added once per split, at every level)
// Base case: dp[i][j] = 0 when j == i+1 (no cut point strictly between)
// Answer: dp[0][m-1] where m = cuts.size() + 2 (points array size)
// ─────────────────────────────────────────────────────────────────

int minCost(int n, vector<int>& cuts) {
    vector<int> points = cuts;
    points.push_back(0);
    points.push_back(n);
    sort(points.begin(), points.end());
    int m = points.size();

    vector<vector<int>> dp(m, vector<int>(m, 0));

    for (int gap = 2; gap < m; gap++) {
        for (int i = 0; i + gap < m; i++) {
            int j = i + gap;
            int best = INT_MAX;
            for (int k = i + 1; k < j; k++) {
                int cost = dp[i][k] + dp[k][j] + (points[j] - points[i]);
                best = min(best, cost);
            }
            dp[i][j] = best;
        }
    }
    return dp[0][m - 1];
}
```

**TRACE for n = 7, cuts = [1,3,4,5] (expected answer: 16):**
```
points (sorted, with 0 and 7 added): [0, 1, 3, 4, 5, 7]   m=6

gap=2:
  dp[0][2] (seg 0-3): k=1 → 0+0+(3-0)=3           → 3
  dp[1][3] (seg 1-4): k=2 → 0+0+(4-1)=3           → 3
  dp[2][4] (seg 3-5): k=3 → 0+0+(5-3)=2           → 2
  dp[3][5] (seg 4-7): k=4 → 0+0+(7-4)=3           → 3

gap=3:
  dp[0][3] (seg 0-4): k=1 → 0+3+4=7 ; k=2 → 3+0+4=7        → 7
  dp[1][4] (seg 1-5): k=2 → 0+2+4=6 ; k=3 → 3+0+4=7        → 6
  dp[2][5] (seg 3-7): k=3 → 0+3+4=7 ; k=4 → 2+0+4=6        → 6

gap=4:
  dp[0][4] (seg 0-5): k=1 → 0+6+5=11 ; k=2 → 3+2+5=10 ; k=3 → 7+0+5=12   → 10
  dp[1][5] (seg 1-7): k=2 → 0+6+6=12 ; k=3 → 3+3+6=12 ; k=4 → 6+0+6=12   → 12

gap=5:
  dp[0][5] (seg 0-7): k=1 → 0+12+7=19
                       k=2 → 3+6+7=16   ← cut at points[2]=3 first
                       k=3 → 7+3+7=17
                       k=4 → 10+0+7=17
                       min = 16

Answer: dp[0][5] = 16 ✓
```

---

## SECTION 8 — TEMPLATE 5: MATRIX CHAIN MULTIPLICATION (THE ARCHETYPE)

```cpp
// ─────────────────────────────────────────────────────────────────
// CLASSIC — Matrix Chain Multiplication
// Matrices M1..Mn, matrix i has dimensions p[i-1] x p[i]. Find the
// parenthesization that minimizes total scalar multiplications for
// computing M1 x M2 x ... x Mn. (Every other interval DP problem in
// this pattern is a variation on this exact recurrence shape.)
//
// State: dp[i][j] = min scalar mults to compute the product of
//                    matrices i through j (1-indexed)
// Transition: dp[i][j] = min over k in [i, j-1] of
//               dp[i][k] + dp[k+1][j] + p[i-1]*p[k]*p[j]
//             (k = index of the LAST matrix in the left group —
//              equivalently, the split point of the LAST
//              multiplication performed: combine "product of i..k"
//              with "product of k+1..j")
// Base case: dp[i][i] = 0 (a single matrix needs no multiplication)
// Answer: dp[1][n]
// ─────────────────────────────────────────────────────────────────

int matrixChainOrder(vector<int>& p) {     // p.size() == n + 1
    int n = p.size() - 1;
    vector<vector<int>> dp(n + 1, vector<int>(n + 1, 0));

    for (int len = 2; len <= n; len++) {
        for (int i = 1; i + len - 1 <= n; i++) {
            int j = i + len - 1;
            dp[i][j] = INT_MAX;
            for (int k = i; k < j; k++) {
                int cost = dp[i][k] + dp[k+1][j] + p[i-1] * p[k] * p[j];
                dp[i][j] = min(dp[i][j], cost);
            }
        }
    }
    return dp[1][n];
}
```

**TRACE for p = [1,2,3,4] (matrices A1:1x2, A2:2x3, A3:3x4, expected answer: 18):**
```
len=2:
  dp[1][2]: k=1 → dp[1][1]+dp[2][2]+p0*p1*p2 = 0+0+1*2*3 = 6
  dp[2][3]: k=2 → dp[2][2]+dp[3][3]+p1*p2*p3 = 0+0+2*3*4 = 24

len=3:
  dp[1][3]: k=1 → dp[1][1]+dp[2][3]+p0*p1*p3 = 0+24+1*2*4 = 32
            k=2 → dp[1][2]+dp[3][3]+p0*p2*p3 = 6+0+1*3*4  = 18
            min = 18   ← parenthesization (A1*A2)*A3

Answer: dp[1][3] = 18 ✓
```

**WHY this is "the archetype":** Burst Balloons, Min Score Triangulation, and Min Cost to Cut a Stick are ALL Matrix Chain Multiplication with the cost function `p[i-1]*p[k]*p[j]` swapped for a different "cost of combining at split k" formula. Once you can derive this recurrence from scratch, every other Type-1/Type-2 problem in this pattern is a five-minute adaptation.

---

## SECTION 9 — TEMPLATE 6: STONE GAME (GAME THEORY INTERVAL DP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 877 — Stone Game
// Two players alternately take a stone from either END of a pile
// (stones[]). Both play optimally, each maximizing THEIR OWN total.
// Does the FIRST player end with strictly more stones?
//
// State: dp[i][j] = the maximum SCORE DIFFERENCE (current player's
//                    total minus the opponent's total) that the
//                    player ABOUT TO MOVE can guarantee, restricted
//                    to stones[i..j]
// Transition: the mover picks stones[i] or stones[j]. Whichever they
//   pick, the OPPONENT becomes "the player about to move" on the
//   remaining sub-array, and whatever the opponent nets there is a
//   LOSS from the current mover's perspective — hence the minus sign:
//   dp[i][j] = max(stones[i] - dp[i+1][j], stones[j] - dp[i][j-1])
// Base case: dp[i][i] = stones[i] (one stone left — forced take)
// Answer: dp[0][n-1] > 0  → first player wins
// ─────────────────────────────────────────────────────────────────

bool stoneGame(vector<int>& stones) {
    int n = stones.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = 0; i < n; i++) dp[i][i] = stones[i];

    for (int len = 2; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            dp[i][j] = max(stones[i] - dp[i+1][j], stones[j] - dp[i][j-1]);
        }
    }
    return dp[0][n-1] > 0;
}
```

**TRACE for stones = [5,3,4,5] (expected answer: true, margin 1):**
```
Base: dp[i][i] = [5, 3, 4, 5]

len=2:
  dp[0][1] = max(5-dp[1][1], 3-dp[0][0]) = max(5-3, 3-5) = max(2,-2) = 2
  dp[1][2] = max(3-dp[2][2], 4-dp[1][1]) = max(3-4, 4-3) = max(-1,1) = 1
  dp[2][3] = max(4-dp[3][3], 5-dp[2][2]) = max(4-5, 5-4) = max(-1,1) = 1

len=3:
  dp[0][2] = max(5-dp[1][2], 4-dp[0][1]) = max(5-1, 4-2) = max(4,2) = 4
  dp[1][3] = max(3-dp[2][3], 5-dp[1][2]) = max(3-1, 5-1) = max(2,4) = 4

len=4:
  dp[0][3] = max(5-dp[1][3], 5-dp[0][2]) = max(5-4, 5-4) = max(1,1) = 1

Answer: dp[0][3] = 1 > 0 → true
(total = 17 stones; first player nets 9, second nets 8 — margin 1 ✓)
```

**WHY the sign flips:** `dp[i][j]` always means "the player whose TURN it is right now, on interval `[i,j]`." After they take `stones[i]`, the SAME formula applied to `[i+1,j]` computes the best result for whoever moves next — which is the OPPONENT. So `dp[i+1][j]` is the opponent's advantage, and must be subtracted to convert it into "my advantage."

---

## SECTION 10 — TEMPLATE 7: STRANGE PRINTER

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 664 — Strange Printer
// The printer prints a sequence of the SAME character starting from
// any position, and every new print stroke fully overwrites whatever
// was there before. Find the minimum number of turns to reproduce s.
//
// State: dp[i][j] = min turns to print s[i..j] on a blank sequence
// Transition (default, worst case): dp[i][j] = dp[i][j-1] + 1
//   (print s[i..j-1] optimally, then s[j] needs its OWN new turn)
// IMPROVEMENT: if s[k] == s[j] for some k in [i, j-1], the stroke
//   that lays down s[k] can be "stretched" all the way to position j
//   for FREE — anything printed on TOP of that stretch in between
//   (positions k+1..j-1) simply overwrites, and what's underneath
//   still matches s[j]'s color once uncovered. So:
//   dp[i][j] = min(dp[i][j], dp[i][k] + dp[k+1][j-1])
//   (dp[k+1][j-1] = 0 when k+1 > j-1, i.e. k == j-1, adjacent match)
// Base case: dp[i][i] = 1
// Answer: dp[0][n-1]
// ─────────────────────────────────────────────────────────────────

int strangePrinter(string s) {
    int n = s.size();
    if (n == 0) return 0;
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = 0; i < n; i++) dp[i][i] = 1;

    for (int len = 2; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            dp[i][j] = dp[i][j-1] + 1;                 // default upper bound
            for (int k = i; k < j; k++) {
                if (s[k] == s[j]) {
                    int inner = (k + 1 <= j - 1) ? dp[k+1][j-1] : 0;
                    dp[i][j] = min(dp[i][j], dp[i][k] + inner);
                }
            }
        }
    }
    return dp[0][n-1];
}
```

**TRACE for s = "aba" (expected answer: 2):**
```
Base: dp[0][0]=dp[1][1]=dp[2][2]=1

len=2:
  dp[0][1] "ab": default dp[0][0]+1=2. k=0: s[0]='a'!=s[1]='b'. dp[0][1]=2
  dp[1][2] "ba": default dp[1][1]+1=2. k=1: s[1]='b'!=s[2]='a'. dp[1][2]=2

len=3:
  dp[0][2] "aba": default dp[0][1]+1 = 3
    k=0: s[0]='a'==s[2]='a' → inner=dp[1][1]=1 → candidate=dp[0][0]+1=1+1=2
    k=1: s[1]='b'!=s[2]='a' → skip
    dp[0][2] = min(3, 2) = 2

Answer: dp[0][2] = 2 ✓
(print "aaa" in turn 1, then "b" at the middle in turn 2)
```

---

## SECTION 11 — TEMPLATE 8: MINIMUM SCORE TRIANGULATION OF POLYGON

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1039 — Minimum Score Triangulation of Polygon
// Convex polygon, vertices values[0..n-1] given in order. Triangulate
// using non-crossing diagonals. A triangle's score = product of its
// three vertex values. Minimize the sum of all triangle scores.
//
// This is Matrix Chain Multiplication with a different cost formula.
// dp[i][j] represents the fan of vertices i..j (edge i-j is the base
// that closes the shape into a polygon). Choosing vertex k as the
// THIRD point of the triangle sharing edge (i,j) is the LAST triangle
// placed — everything else splits cleanly into (i,k) and (k,j).
//
// State: dp[i][j] = min score to triangulate vertices i..j
// Transition: dp[i][j] = min over k in (i,j) of
//               dp[i][k] + dp[k][j] + values[i]*values[k]*values[j]
// Base case: dp[i][j] = 0 when j <= i+1 (fewer than 3 vertices)
// Answer: dp[0][n-1]
// ─────────────────────────────────────────────────────────────────

int minScoreTriangulation(vector<int>& values) {
    int n = values.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));

    for (int len = 2; len < n; len++) {          // need >= 3 vertices: len >= 2
        for (int i = 0; i + len < n; i++) {
            int j = i + len;
            dp[i][j] = INT_MAX;
            for (int k = i + 1; k < j; k++) {
                int score = dp[i][k] + dp[k][j] + values[i]*values[k]*values[j];
                dp[i][j] = min(dp[i][j], score);
            }
        }
    }
    return dp[0][n-1];
}
```

**TRACE for values = [3,7,4,5] (expected answer: 144):**
```
len=2:
  dp[0][2]: k=1 → dp[0][1]+dp[1][2]+3*7*4 = 0+0+84  = 84
  dp[1][3]: k=2 → dp[1][2]+dp[2][3]+7*4*5 = 0+0+140 = 140

len=3:
  dp[0][3]: k=1 → dp[0][1]+dp[1][3]+3*7*5 = 0+140+105 = 245
            k=2 → dp[0][2]+dp[2][3]+3*4*5 = 84+0+60   = 144
            min = 144

Answer: dp[0][3] = 144 ✓
(triangles: (0,2,3) score 3*4*5=60, and (0,1,2) score 3*7*4=84 → 144)
```

---

## SECTION 12 — TEMPLATE 9: MINIMUM COST TO MERGE STONES (K-WAY MERGE — THE HARDEST)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1000 — Minimum Cost to Merge Stones
// Merge EXACTLY K consecutive piles into one in a single move; the
// cost of that move equals the sum of the merged piles. Return the
// minimum cost to merge everything into one pile, or -1.
//
// FEASIBILITY: each move reduces the pile count by (K-1). Starting
// from n piles, 1 pile is reachable iff (n-1) % (K-1) == 0.
//
// The extra twist versus a plain Matrix Chain shape: we need a THIRD
// dimension, m = "how many piles is this interval currently split
// into," because K piles must accumulate before they can collapse
// into 1 in a single move.
//
// State: dp[i][j][m] = min cost to merge stones[i..j] into EXACTLY m
//                       piles (1 <= m <= K)
// Transition (m > 1): dp[i][j][m] = min over k in [i,j) of
//               dp[i][k][1] + dp[k+1][j][m-1]
//             (collapse the LEFT part down to exactly 1 pile, leave
//              the RIGHT part split into m-1 piles; sweeping k covers
//              every way the m piles could be distributed)
// Transition (m == 1): only legal once dp[i][j][K] is known — merge
//               those K piles in ONE move, costing the full range sum:
//               dp[i][j][1] = dp[i][j][K] + rangeSum(i,j)
//             (valid only if (j-i) % (K-1) == 0, i.e. K piles is
//              actually reachable inside this exact interval)
// Base case: dp[i][i][1] = 0
// Answer: dp[0][n-1][1]
// LOOP ORDER: by increasing length; within a length, fill every
//             m in [2,K] BEFORE m == 1 (m==1 depends on m==K)
// ─────────────────────────────────────────────────────────────────

int mergeStones(vector<int>& stones, int K) {
    int n = stones.size();
    if ((n - 1) % (K - 1) != 0) return -1;

    vector<int> prefix(n + 1, 0);
    for (int i = 0; i < n; i++) prefix[i+1] = prefix[i] + stones[i];
    auto rangeSum = [&](int i, int j) { return prefix[j+1] - prefix[i]; };

    const int INF = INT_MAX / 2;
    vector<vector<vector<int>>> dp(
        n, vector<vector<int>>(n, vector<int>(K + 1, 0)));

    for (int len = 2; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            for (int m = 2; m <= K; m++) {
                dp[i][j][m] = INF;
                for (int k = i; k < j; k++) {
                    dp[i][j][m] = min(dp[i][j][m],
                                       dp[i][k][1] + dp[k+1][j][m-1]);
                }
            }
            if ((len - 1) % (K - 1) == 0)
                dp[i][j][1] = dp[i][j][K] + rangeSum(i, j);
            else
                dp[i][j][1] = INF;      // K piles not reachable here yet
        }
    }
    return dp[0][n-1][1] >= INF ? -1 : dp[0][n-1][1];
}
```

**FULL TRACE for stones = [3,2,4,1], K = 2 (expected answer: 20):**
```
With K=2, m ranges over {1,2} only, and EVERY interval qualifies for
the m==1 collapse (since K-1=1 divides everything). This reduces to
the plain Matrix-Chain shape — merge stones pairwise.

prefix = [0,3,5,9,10]   rangeSum(i,j) = prefix[j+1]-prefix[i]

len=2:
  dp[0][1][2]=dp[0][0][1]+dp[1][1][1]=0 → dp[0][1][1]=0+rangeSum(0,1)=5
  dp[1][2][2]=0 → dp[1][2][1]=0+rangeSum(1,2)=6
  dp[2][3][2]=0 → dp[2][3][1]=0+rangeSum(2,3)=5

len=3:
  dp[0][2][2]: k=0→0+dp[1][2][1]=6 ; k=1→dp[0][1][1]+0=5   → min=5
               dp[0][2][1] = 5 + rangeSum(0,2) = 5+9 = 14
  dp[1][3][2]: k=1→0+dp[2][3][1]=5 ; k=2→dp[1][2][1]+0=6   → min=5
               dp[1][3][1] = 5 + rangeSum(1,3) = 5+7 = 12

len=4:
  dp[0][3][2]: k=0→0+dp[1][3][1]=12
               k=1→dp[0][1][1]+dp[2][3][1]=5+5=10
               k=2→dp[0][2][1]+0=14
               min = 10
  dp[0][3][1] = 10 + rangeSum(0,3) = 10 + 10 = 20

Answer: dp[0][3][1] = 20 ✓
```

**WHY this is "the hardest":** every OTHER problem in this pattern has a single interval-value state. Merge Stones needs a THIRD axis (pile count `m`) because the physical constraint "exactly K piles merge per move" doesn't fit into a binary split — you're accumulating toward a threshold before you're allowed to collapse. Recognizing that `m` must be tracked (and that `m==1` depends on `m==K` within the SAME interval) is the make-or-break insight.

---

## SECTION 13 — TEMPLATE 10: ZUMA GAME (INTERVAL-COLLAPSE AS SEARCH, NOT A CLEAN TABLE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 488 — Zuma Game
// Board of colored balls in a row; hand of colored balls. Insert one
// hand ball anywhere in the board per move. Whenever 3+ CONSECUTIVE
// balls of the same color form, they vanish immediately (and this
// can chain — a new run can form once neighbors close the gap).
// Find the minimum hand balls needed to clear the board, or -1.
//
// WHY IT BELONGS IN THIS PATTERN: the core sub-operation — "remove a
// contiguous run and let its former left/right neighbors become
// adjacent" — is EXACTLY the Burst-Balloons idea of collapsing an
// interval and merging its boundaries. Zuma generalizes that idea to
// a SEARCH over WHERE to insert each hand ball, so it can't be solved
// with a clean length-ascending dp[i][j] table — the "array" itself
// mutates in shape at every step. It's solved with memoized recursion
// over (board, hand) STRING states instead. Included here as the
// cautionary edge case: recognize when an interval-elimination
// problem does NOT reduce to the O(n^3) template.
// ─────────────────────────────────────────────────────────────────

string collapse(string s) {                 // remove all runs of 3+, chain
    bool changed = true;
    while (changed) {
        changed = false;
        int n = s.size();
        for (int i = 0; i < n; ) {
            int j = i;
            while (j < n && s[j] == s[i]) j++;
            if (j - i >= 3) {
                s = s.substr(0, i) + s.substr(j);
                changed = true;
                break;                       // restart scan after a removal
            }
            i = j;
        }
    }
    return s;
}

int solve(string board, string hand, unordered_map<string,int>& memo) {
    if (board.empty()) return 0;
    if (hand.empty()) return INT_MAX;
    string key = board + "#" + hand;
    auto it = memo.find(key);
    if (it != memo.end()) return it->second;

    int best = INT_MAX;
    for (int i = 0; i < (int)hand.size(); i++) {
        if (i > 0 && hand[i] == hand[i-1]) continue;  // skip duplicate choices
        for (int j = 0; j <= (int)board.size(); j++) {
            string nextHand  = hand.substr(0, i) + hand.substr(i + 1);
            string nextBoard = collapse(board.substr(0, j) + hand[i] + board.substr(j));
            int sub = solve(nextBoard, nextHand, memo);
            if (sub != INT_MAX) best = min(best, sub + 1);
        }
    }
    return memo[key] = best;
}

int findMinStep(string board, string hand) {
    sort(hand.begin(), hand.end());          // canonical order for memo key
    unordered_map<string, int> memo;
    int result = solve(board, hand, memo);
    return result == INT_MAX ? -1 : result;
}
```

**CONCEPTUAL WALKTHROUGH for board = "G", hand = "GGGGG" (expected answer: 2):**
```
Board "G", hand "GGGGG" (sorted).
Move 1: insert a 'G' adjacent to the existing 'G' → board becomes "GG".
        No run of 3+ yet, no collapse. Hand used so far: 1.
Move 2: insert another 'G' adjacent → board becomes "GGG".
        Run of 3 identical → collapses to "" (empty board). Hand used: 2.

Board is empty after 2 hand balls. Answer: 2 ✓
```

The recursion explores EVERY insertion position for every distinct next hand color and memoizes on the `(board, hand)` string pair so overlapping states (reached via different insertion orders that land on the same board/hand configuration) are not recomputed — the same amortization idea as an interval DP table, just keyed by strings instead of `(i,j)` integers.

---

## SECTION 14 — COMPLEXITY TABLE

| Problem | Time | Space | Notes |
|---------|------|-------|-------|
| Longest Palindromic Subsequence | O(n²) | O(n²) | Adjacent-only recurrence, no split loop |
| Palindrome Partitioning II | O(n²) | O(n²) | Interval oracle + separate O(n²) 1D DP |
| Burst Balloons | O(n³) | O(n²) | Split-point loop, sentinel padding |
| Minimum Cost to Cut a Stick | O(m³) | O(m²) | m = cuts.size()+2, same shape as MCM |
| Matrix Chain Multiplication | O(n³) | O(n²) | The archetype for all split-point problems |
| Stone Game | O(n²) | O(n²) | Adjacent-only recurrence (no split loop) |
| Strange Printer | O(n³) | O(n²) | Split loop for the "stretch" improvement |
| Min Score Triangulation of Polygon | O(n³) | O(n²) | Identical shape to MCM |
| Minimum Cost to Merge Stones | O(n³·K) | O(n²·K) | Extra pile-count dimension m |
| Zuma Game | Exponential (memoized) | O(states seen) | Not a clean interval table — string-keyed search |
| Allocate Mailboxes | O(n³) | O(n²) | Precompute interval cost + K-partition 1D DP |

**Pattern-wide takeaway:** whenever the transition needs to enumerate a SPLIT POINT `k` between `i` and `j` (Type 1/2/3), the complexity is O(n³). Whenever the transition only touches ADJACENT sub-intervals `dp[i+1][j]`, `dp[i][j-1]`, `dp[i+1][j-1]` (Type 4/5's simplest members, like LPS and Stone Game), it drops to O(n²) — no inner `k` loop needed.

---

## SECTION 15 — VARIANTS

**Count Palindromic Substrings (LC 647):** Same `isPalin[i][j]` interval table as Palindrome Partitioning II's Step 1, but the answer is simply the COUNT of `true` cells rather than feeding a secondary 1D DP.

**Predict the Winner (LC 486) / Stone Game variants (II, III, VII):** All reuse the Stone Game score-difference recurrence `dp[i][j] = max(a[i]-dp[i+1][j], a[j]-dp[i][j-1])`; the harder variants (Stone Game II/III) add a move-count or k-choice dimension similar in spirit to Merge Stones' extra `m` axis.

**Allocate Mailboxes (LC 1478):** A different flavor — combine a precomputed interval COST table with a 1D partition DP:
```cpp
// cost[i][j] = min total distance serving houses i..j with ONE mailbox
//              (optimal mailbox location = median of houses[i..j])
// dp[k][i] = min cost using k mailboxes for the first i houses
// dp[k][i] = min over j < i of dp[k-1][j] + cost[j][i-1]
```
This shows interval DP doesn't always stand alone — it frequently supplies a `cost(i,j)` oracle consumed by an outer partition/knapsack-style DP (Pattern 16/19 territory).

**Remove Boxes (LC 546):** A harder Strange-Printer-style problem that needs a THIRD dimension (count of same-colored boxes attached to the right of `j`) — `dp[i][j][k]` — because the score depends on how many extra same-color boxes were "saved up" from earlier removals, not just the raw interval content.

---

## SECTION 16 — COMMON MISTAKES

### Mistake 1: Wrong loop order — not iterating by length

```cpp
// WRONG — row-major order; dp[i+1][...] isn't ready when row i runs
for (int i = 0; i < n; i++)
    for (int j = i+1; j < n; j++)
        dp[i][j] = max(dp[i+1][j], dp[i][j-1]);   // dp[i+1][j] is stale/garbage

// CORRECT — iterate by increasing interval length
for (int len = 2; len <= n; len++)
    for (int i = 0; i + len - 1 < n; i++) {
        int j = i + len - 1;
        dp[i][j] = max(dp[i+1][j], dp[i][j-1]);   // both strictly shorter, ready
    }
```

---

### Mistake 2: Off-by-one on the split-point range `k`

```cpp
// WRONG — k allowed to equal j (or start below i+1), causing
// dp[i][j] to read itself or an interval of the SAME length
for (int k = i; k <= j; k++)                       // BUG: k==j means
    dp[i][j] = min(dp[i][j], dp[i][k] + dp[k][j] + cost); // dp[k][j]=dp[j][j], fine,
                                                     // but dp[i][k]=dp[i][j] when k==j
                                                     // → self-reference, undefined order

// CORRECT — k strictly between i and j (exclusive on both ends)
for (int k = i + 1; k < j; k++)
    dp[i][j] = min(dp[i][j], dp[i][k] + dp[k][j] + cost);
```
For Matrix-Chain-style problems (endpoints INCLUSIVE), the correct range is `k` in `[i, j-1]` because `dp[i][k]` and `dp[k+1][j]` must never overlap at a shared index — always re-derive the range from "do the two halves cover `[i,j]` exactly once, with no gap and no overlap?"

---

### Mistake 3: Forgetting boundary sentinels in Burst Balloons

```cpp
// WRONG — no sentinels; every boundary access needs special-casing
int maxCoins(vector<int>& nums) {
    int n = nums.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int gap = 2; gap < n; gap++)
        for (int i = 0; i + gap < n; i++) {
            int j = i + gap;
            for (int k = i+1; k < j; k++) {
                int left  = (i == -1) ? 1 : nums[i];   // this branch never
                int right = (j == n) ? 1 : nums[j];    // triggers — i,j never
                dp[i][j] = max(dp[i][j], ...);          // reach those "virtual" values
            }
        }
    // The formula silently uses nums[i] and nums[j] as if they were walls,
    // but they're REAL balloons in this indexing — the answer double-counts
    // or under-counts the true edge balloons.
}

// CORRECT — pad with sentinel value 1 at both ends FIRST
vector<int> a(n + 2, 1);
for (int i = 0; i < n; i++) a[i+1] = nums[i];
// now a[0] and a[n+1] are genuine "never-burst" walls, and the same
// formula a[i]*a[k]*a[j] works uniformly for every gap, including the
// outermost dp[0][n+1]
```

---

### Mistake 4: "Which is FIRST" instead of "which is LAST"

```cpp
// WRONG mental model — trying to decide the FIRST balloon to burst
// int firstBurst = ...;
// This makes the neighbor sets of every OTHER balloon depend on which
// balloon burst first — the two resulting halves are NOT independent,
// so no valid dp[i][j] = dp[i][k] + dp[k][j] decomposition exists.

// CORRECT mental model — decide the LAST balloon burst in (i,j)
// At the moment k bursts LAST, everything else in (i,j) is already
// gone, so k's only possible neighbors are the fixed walls i and j.
// This makes (i,k) and (k,j) genuinely independent sub-problems.
dp[i][j] = max over k in (i,j) of dp[i][k] + dp[k][j] + a[i]*a[k]*a[j];
```
This "last, not first" reframe applies identically to Matrix Chain (last multiplication), Polygon Triangulation (last triangle placed), and Min Cost to Cut a Stick (conceptually: last cut = the split point `k` chosen at the OUTER level of recursion, even though physically all cuts within a segment can happen in any order — cost only depends on segment length at cut time, not on ordering).

---

### Mistake 5: Confusing inclusive vs. exclusive interval semantics

```cpp
// Matrix Chain / LPS / Stone Game: dp[i][j] is INCLUSIVE of both
// endpoints — i and j are REAL elements that participate.
//   dp[i][i] = single element base case.

// Burst Balloons / Cut Stick: dp[i][j] is EXCLUSIVE of the middle —
// i and j are BOUNDARY WALLS that are never touched; only the
// content STRICTLY BETWEEN them is being resolved.
//   dp[i][i+1] = 0 (no content between adjacent walls) is the base case,
//   NOT dp[i][i].

// WRONG — copy-pasting an inclusive base case into an exclusive-style
// problem (or vice versa)
for (int i = 0; i < n; i++) dp[i][i] = 1;   // meaningless for Burst Balloons —
                                              // dp[i][i] is never even queried
                                              // (gap starts at 2, i.e. j=i+1)

// CORRECT — match the base case to which semantic this problem uses.
// Always write out, in one sentence, "dp[i][j] means ___" before coding.
```

---

### Mistake 6: Forgetting the length-2 base case for `dp[i+1][j-1]`

```cpp
// WRONG — always dereferencing dp[i+1][j-1] even when i+1 > j-1
if (s[i] == s[j])
    dp[i][j] = dp[i+1][j-1] + 2;    // when len==2, i+1==j and j-1==i,
                                      // so i+1 > j-1 is FALSE here — actually
                                      // i+1 == j-1+1, meaning we'd read
                                      // dp[j][i] — an UNINITIALIZED cell
                                      // above the diagonal (never computed,
                                      // silently returns 0 from default-init,
                                      // which HAPPENS to be correct here but
                                      // is undefined behavior in general and
                                      // WILL be wrong for boolean isPalin
                                      // arrays or non-zero-default value types)

// CORRECT — explicit guard for the length-2 (and length-1) edge cases
int inner = (len <= 2) ? 0 : dp[i+1][j-1];
dp[i][j] = (s[i] == s[j]) ? inner + 2 : max(dp[i+1][j], dp[i][j-1]);
```

---

## SECTION 17 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Longest Palindromic Subsequence | 516 | Interval form of a two-sequence idea, adjacent-only recurrence |
| 2 | Palindrome Partitioning II | 132 | Interval oracle table feeding a separate 1D DP |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Burst Balloons | 312 | "Last burst, not first" reframe, sentinel padding |
| 4 | Minimum Cost to Cut a Stick | 1547 | MCM shape with boundary points inserted |
| 5 | Matrix Chain Multiplication | (classic) | The archetype split-point recurrence |
| 6 | Stone Game | 877 | Game-theory score-difference DP |

### STRETCH (solve in ≤ 45 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Strange Printer | 664 | "Stretch a stroke" improvement over adjacent recurrence |
| 8 | Minimum Score Triangulation of Polygon | 1039 | MCM shape applied to geometry |
| 9 | Minimum Cost to Merge Stones | 1000 | K-way merge, extra pile-count dimension |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Zuma Game | 488 | Recognizing when interval-collapse needs search, not a table |
| 11 | Allocate Mailboxes | 1478 | Interval cost oracle + K-partition outer DP |

---

## SECTION 18 — PATTERN CONNECTIONS

1. **Pattern 18 (2D DP and Grid DP):** Both patterns fill a 2D table, but the semantics differ fundamentally. Pattern 18's `dp[i][j]` means "best answer reaching/using position (i,j)" with a fixed forward or backward fill direction. Pattern 23's `dp[i][j]` means "best answer for the SUBRANGE [i,j]," filled by increasing LENGTH rather than by row or column — a genuinely different iteration discipline that trips up engineers who over-generalize Pattern 18's "just go top-to-bottom" instinct.

2. **Pattern 20 (LCS Family):** Longest Palindromic Subsequence has two equivalent formulations: the LCS-style two-sequence form (`LCS(s, reverse(s))`, Pattern 20's technique) and the direct interval form shown here (`dp[i][j]` over `s[i..j]`). Seeing both formulations side by side is the single best exercise for internalizing when a problem is "secretly" a two-sequence DP versus a genuine interval DP — the interval form is strictly more general (it also handles Palindrome Partitioning II's oracle, which the LCS form cannot express directly).

3. **Pattern 16 (DP Fundamentals and the DP Mindset):** Pattern 16 teaches the meta-skill of choosing "what does dp[state] mean, and in what order must I fill it?" Interval DP is the pattern where that second question — the FILL ORDER — is hardest to get right on instinct, because the natural row-major loop from earlier patterns actively produces a plausible-looking but WRONG answer (Section 3) rather than crashing. It is the best pattern to stress-test whether Pattern 16's dependency-order discipline actually stuck.

---

## SECTION 19 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 312 — Burst Balloons

**Interviewer:** "Given balloons with values, bursting balloon i earns `left * nums[i] * right` where left/right are its CURRENT neighbors. Maximize total coins from bursting all balloons."

**Candidate:**

*[First minute — reject the naive framing out loud]*
> "My first instinct is to think about which balloon to burst FIRST, maybe greedily bursting the smallest neighbors first. But that's wrong — bursting one balloon changes the neighbor sets for everyone else, so the sub-problems after a 'first burst' choice aren't independent. I need a different decomposition."

*[State the reframe before coding]*
> "The trick is to think about which balloon bursts LAST within some range `[i,j]`. If balloon k is the last one standing between walls i and j, then when it finally bursts, everything else between i and j is already gone — so its neighbors are GUARANTEED to be exactly i and j. That makes the two halves, `(i,k)` and `(k,j)`, fully independent sub-problems."

*[Mention the sentinel trick]*
> "I'll pad the array with a 1 on each end so the true boundary balloons don't need special-casing — `dp[0][n+1]` becomes the final answer using the exact same formula as every interior cell."

*[Code — 6 minutes]*
```cpp
int maxCoins(vector<int>& nums) {
    int n = nums.size();
    vector<int> a(n+2, 1);
    for (int i = 0; i < n; i++) a[i+1] = nums[i];
    vector<vector<int>> dp(n+2, vector<int>(n+2, 0));
    for (int gap = 2; gap <= n+1; gap++)
        for (int i = 0; i + gap <= n+1; i++) {
            int j = i + gap;
            for (int k = i+1; k < j; k++)
                dp[i][j] = max(dp[i][j], dp[i][k]+dp[k][j]+a[i]*a[k]*a[j]);
        }
    return dp[0][n+1];
}
```

*[Follow-up]: "Why must the outer loop be by `gap` and not by `i`?"*
> "`dp[i][j]` depends on `dp[i][k]` and `dp[k][j]` for an ARBITRARY split point k — both are strictly narrower intervals, but I don't know in advance whether the narrower interval sits in an earlier row or later row of a naive i-then-j sweep. Iterating by increasing gap (interval length) is the one loop order that guarantees every dependency is already computed, regardless of where it falls in the 2D table."

**RED FLAGS:**
- Trying to reason about "first balloon burst" instead of "last balloon burst"
- Forgetting the sentinel padding, then hand-writing boundary special cases that are easy to get wrong
- Using a row-major loop order (`for i, for j`) instead of by-gap/by-length

---

### Interview Simulation 2: Matrix Chain Multiplication (generalization discussion)

**Interviewer:** "Given matrix dimensions, find the minimum scalar multiplications to compute their product. Then tell me how this generalizes."

**Candidate:**

*[State the recurrence]*
> "`dp[i][j]` = min cost to multiply matrices i through j. For any split point k, computing the product of i..k, then k+1..j, then multiplying those two results costs `dp[i][k] + dp[k+1][j] + p[i-1]*p[k]*p[j]`. I take the min over all valid k."

*[Code — 5 minutes]*
```cpp
int matrixChainOrder(vector<int>& p) {
    int n = p.size() - 1;
    vector<vector<int>> dp(n+1, vector<int>(n+1, 0));
    for (int len = 2; len <= n; len++)
        for (int i = 1; i + len - 1 <= n; i++) {
            int j = i + len - 1;
            dp[i][j] = INT_MAX;
            for (int k = i; k < j; k++)
                dp[i][j] = min(dp[i][j], dp[i][k]+dp[k+1][j]+p[i-1]*p[k]*p[j]);
        }
    return dp[1][n];
}
```

*[Interviewer]: "How does this relate to Burst Balloons or Polygon Triangulation?"*
> "All three share the exact same skeleton — split-point enumeration over `dp[i][k] + dp[k][j] + cost(i,k,j)`. The only thing that changes is the cost formula: `p[i-1]*p[k]*p[j]` for matrix dimensions, `values[i]*values[k]*values[j]` for triangle scores, `a[i]*a[k]*a[j]` for balloon products. Once I recognize a problem fits 'pick a split point that becomes the LAST combining operation, cost depends on the three boundary values,' I reuse this exact template."

*[Interviewer]: "What if I asked you to also RECONSTRUCT the optimal parenthesization, not just the cost?"*
> "Store a `split[i][j]` table alongside `dp[i][j]`, recording which `k` achieved the minimum. Then recursively print `(recurse(i,split[i][j]) recurse(split[i][j]+1,j))` — same reconstruction technique as any DP with a 'best choice' table."

**RED FLAGS:**
- Not recognizing that Burst Balloons and Triangulation are the SAME recurrence shape
- Off-by-one in the `k` range — `[i, j-1]` here because endpoints are inclusive matrices, versus `(i,j)` exclusive in Burst Balloons
- No plan for reconstructing the actual parenthesization when asked

---

### Interview Simulation 3: Meta-discussion — recognizing interval DP versus other 2D DP

**Interviewer:** "How do you tell, in the first 60 seconds of reading a problem, whether it's interval DP versus some other 2D DP pattern?"

**Candidate:**
> "Three signals. First: does the problem talk about operating on a contiguous SUBRANGE of an array or string that shrinks as you make choices — bursting, cutting, merging, partitioning? That's a strong interval-DP smell.

> Second: is there a natural 'combine two adjacent pieces into one' structure — like matrix multiplication being associative, or two triangulated fan-halves sharing an edge? If the problem is fundamentally about picking WHERE to split something into two independently-solvable pieces, interval DP.

> Third — and this is the test that catches false positives — does the transition need to enumerate every possible split point `k`, or does it only ever look at the two immediately adjacent smaller intervals (`dp[i+1][j]`, `dp[i][j-1]`)? If it's the latter, I still call it interval DP (LPS and Stone Game are examples), but it degrades to O(n²), not O(n³), and I should not force an unnecessary O(n) inner loop where a direct adjacent-cell recurrence suffices.

> The failure mode to avoid: seeing 'two-dimensional table' and reflexively reaching for Pattern 18's forward/backward row-fill logic. Interval DP's fill order is BY LENGTH, which is a different discipline entirely — mixing them up produces a plausible-looking wrong answer, not a crash."

**RED FLAGS:**
- Not distinguishing O(n²) adjacent-only interval DP from O(n³) split-point interval DP — defaulting to an unnecessary k-loop
- Applying Pattern 18's row/column fill order out of habit instead of by-length

---

## SECTION 20 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why must interval DP be filled in order of increasing interval LENGTH (or an equivalent decreasing-i sweep), rather than row-major (i then j) order?**

> `dp[i][j]`'s transition depends on strictly shorter intervals — either the two adjacent ones (`dp[i+1][j]`, `dp[i][j-1]`) or any split (`dp[i][k]`, `dp[k][j]` for `k` between `i` and `j`). A row-major loop processes `dp[i][j]` before `dp[i+1][j]` has been computed (since the outer loop is still stuck at row `i`), silently reading a stale or default value instead of the correct answer. Iterating by length guarantees every dependency — regardless of which row or column it lives in — is finished before it's needed, because ALL length-`L` intervals are done before ANY length-`(L+1)` interval starts.

---

**Q2: In Burst Balloons, why do we think about the LAST balloon burst in `(i,j)` instead of the FIRST?**

> If we fix which balloon bursts FIRST, every other balloon's neighbor set becomes dependent on that choice — the two "halves" of the interval aren't independent, so there's no clean `dp[i][k] + dp[k][j]` decomposition. If we instead fix which balloon `k` bursts LAST in `(i,j)`, then at the moment it bursts, every other balloon strictly between `i` and `j` is ALREADY gone — so `k`'s neighbors are guaranteed to be exactly the boundary balloons `i` and `j`, regardless of what order the other balloons burst in. This makes `(i,k)` and `(k,j)` genuinely independent sub-problems that can be solved separately and summed.

---

**Q3: What is the difference in `dp[i][j]` semantics between Matrix Chain Multiplication and Burst Balloons?**

> Matrix Chain: `dp[i][j]` is INCLUSIVE — `i` and `j` are real matrices that participate in the product being computed. Base case is `dp[i][i] = 0` (single matrix).
> Burst Balloons: `dp[i][j]` is EXCLUSIVE of its own endpoints — `i` and `j` are boundary WALLS (padded sentinels or already-decided neighbors) that are NEVER themselves burst within this sub-problem; only the content strictly between them is resolved. Base case is `dp[i][i+1] = 0` (no balloons strictly between adjacent walls).
> Mixing these up — e.g., writing `dp[i][i]=1` for Burst Balloons — produces a base case that's never even queried (since the gap/length loop starts at 2) and signals a misunderstanding of the state.

---

**Q4: In Minimum Cost to Merge Stones, why is a third dimension `m` (pile count) needed, when Matrix Chain Multiplication doesn't need one?**

> Matrix Chain always combines exactly TWO groups at a time (any split `k` divides `[i,j]` into two pieces, and a single multiplication merges them). Merge Stones requires accumulating EXACTLY K piles before a single move can collapse them into 1 — an interval might currently be split into 2, 3, ..., or K piles, and only the K-piles state is eligible for the "one move, one collapse" cost. Without tracking `m`, there's no way to express "this interval currently sits at 3 piles, waiting to reach K before it can merge in one step." The extra dimension encodes an accumulation THRESHOLD that Matrix Chain's binary-split structure doesn't have.

---

**Q5: Why does Stone Game's recurrence `dp[i][j] = max(stones[i] - dp[i+1][j], stones[j] - dp[i][j-1])` need a MINUS sign, when Matrix Chain's recurrence only ever ADDS sub-results?**

> `dp[i][j]` always represents the advantage of the player whose TURN it currently is. After the current player takes `stones[i]`, applying the same `dp` definition to `[i+1,j]` computes the advantage of whoever moves NEXT — which is the OPPONENT, not the current player. From the current player's perspective, the opponent's advantage is a loss, so it must be subtracted, not added. Matrix Chain has no adversarial "whose turn is it" concept — both sub-results are contributions toward the SAME shared objective (total multiplication cost), so they're simply summed.

---

**Q6: Why can't Zuma Game be solved with a clean `dp[i][j]` table the way Burst Balloons can?**

> Burst Balloons operates on a FIXED array — the set of possible sub-intervals `[i,j]` is known in advance (there are exactly `O(n^2)` of them), so a table indexed by `(i,j)` covers every reachable state. Zuma Game's "array" (the board) actively changes SHAPE at every move (an insertion can happen at any of `O(len)` positions, and each insertion can trigger a cascading collapse that removes an unpredictable, position-dependent chunk). The reachable board configurations aren't a clean, enumerable set of contiguous sub-ranges of ONE fixed original array — they're an open-ended set of DERIVED strings. So the state space must be memoized by the actual STRING content (`board + hand`) rather than by two integer indices into a fixed original array.

---

**Q7: A problem gives you `dp[i][j] = f(dp[i+1][j], dp[i][j-1])` with no split-point loop at all (like LPS or Stone Game). Is this still "interval DP," and what's its complexity?**

> Yes — it's still interval DP because the state is "the answer for subrange `[i,j]`," which is the defining characteristic of this pattern, and it's still filled by increasing length for the same correctness reason (Q1). But because the transition only ever touches the two IMMEDIATELY adjacent shorter intervals rather than enumerating an arbitrary split point `k`, there's no `O(n)` inner loop — the complexity is `O(n^2)` states x `O(1)` transition = `O(n^2)`, not `O(n^3)`. Forcing an unnecessary split-point loop onto a problem that only needs adjacent-cell lookups is a common over-engineering mistake.

---

## SECTION 21 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. THE UNIVERSAL SKELETON (memorize this shape)
// ─────────────────────────────────────────────────────────────────
for (int len = 2; len <= n; len++) {              // by increasing length — MANDATORY
    for (int i = 0; i + len - 1 < n; i++) {
        int j = i + len - 1;
        // EITHER adjacent-only (O(n^2) total):
        //   dp[i][j] = f(dp[i+1][j], dp[i][j-1], dp[i+1][j-1]);
        // OR split-point enumeration (O(n^3) total):
        //   for (int k = i+1; k < j; k++)         // range depends on inclusive/exclusive!
        //       dp[i][j] = opt(dp[i][j], dp[i][k] + dp[k][j] + cost(i,k,j));
    }
}

// ─────────────────────────────────────────────────────────────────
// 2. LONGEST PALINDROMIC SUBSEQUENCE / PALINDROME ORACLE
// ─────────────────────────────────────────────────────────────────
dp[i][j] = (s[i]==s[j]) ? (len<=2 ? 0 : dp[i+1][j-1]) + 2
                         : max(dp[i+1][j], dp[i][j-1]);
// isPalin[i][j] = (s[i]==s[j]) && (len==2 || isPalin[i+1][j-1]);

// ─────────────────────────────────────────────────────────────────
// 3. BURST BALLOONS (sentinel padding + "last burst" reframe)
// ─────────────────────────────────────────────────────────────────
vector<int> a(n+2, 1);
for (int i = 0; i < n; i++) a[i+1] = nums[i];
for (int gap = 2; gap <= n+1; gap++)
    for (int i = 0; i + gap <= n+1; i++) {
        int j = i + gap;
        for (int k = i+1; k < j; k++)
            dp[i][j] = max(dp[i][j], dp[i][k]+dp[k][j]+a[i]*a[k]*a[j]);
    }
// answer = dp[0][n+1]

// ─────────────────────────────────────────────────────────────────
// 4. MATRIX CHAIN MULTIPLICATION (the archetype cost formula)
// ─────────────────────────────────────────────────────────────────
for (int k = i; k < j; k++)
    dp[i][j] = min(dp[i][j], dp[i][k]+dp[k+1][j]+p[i-1]*p[k]*p[j]);
// answer = dp[1][n]     (1-indexed, endpoints inclusive)

// ─────────────────────────────────────────────────────────────────
// 5. STONE GAME (score-difference game theory)
// ─────────────────────────────────────────────────────────────────
dp[i][j] = max(stones[i] - dp[i+1][j], stones[j] - dp[i][j-1]);
// answer: dp[0][n-1] > 0  → first player wins by that margin

// ─────────────────────────────────────────────────────────────────
// 6. MIN COST TO CUT A STICK / MIN SCORE TRIANGULATION (MCM variants)
// ─────────────────────────────────────────────────────────────────
// Cut Stick:      dp[i][j] = min_k dp[i][k]+dp[k][j]+(points[j]-points[i]);
// Triangulation:  dp[i][j] = min_k dp[i][k]+dp[k][j]+values[i]*values[k]*values[j];

// ─────────────────────────────────────────────────────────────────
// 7. MERGE STONES (extra pile-count dimension m)
// ─────────────────────────────────────────────────────────────────
// dp[i][j][m>1] = min_k dp[i][k][1] + dp[k+1][j][m-1];
// dp[i][j][1]   = dp[i][j][K] + rangeSum(i,j)   IF (len-1)%(K-1)==0

// ─────────────────────────────────────────────────────────────────
// 8. DECISION TABLE — WHICH SHAPE DOES THIS PROBLEM NEED?
// ─────────────────────────────────────────────────────────────────
// Adjacent-only, O(n^2):
//   "does the interval's answer depend only on removing/using ONE END
//    at a time?" — LPS, Stone Game
// Split-point, O(n^3):
//   "does something get inserted/combined/removed at an ARBITRARY
//    interior position?" — Burst Balloons, MCM, Triangulation, Cut Stick
// Extra dimension needed:
//   "is there an accumulation THRESHOLD before an operation is legal?"
//   — Merge Stones (need K piles), Remove Boxes (need matching color count)
// Not a clean table at all:
//   "does the 'array' itself change SHAPE unpredictably at each step?"
//   — Zuma Game (string-keyed memoized search instead)
```

---

## SECTION 22 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Burst Balloons and Minimum Cost to Merge Stones are the two that require genuine new insight (the "last, not first" reframe, and the extra pile-count dimension respectively). Budget 45–60 minutes each on first attempt.

2. **Practice deriving the loop order from scratch, not from memory.** For every new interval DP problem, explicitly write the sentence "`dp[i][j]` depends on ___, which is always a strictly ___ interval" before writing any loop. This habit is what prevents the silent row-major bug from Section 3.

3. **Matrix Chain Multiplication → Burst Balloons → Polygon Triangulation → Cut Stick:** solve these four back-to-back. They are the SAME recurrence with a different cost formula plugged in. Doing all four in one sitting is the fastest way to internalize the split-point template permanently.

4. **After Stone Game, try Predict the Winner (LC 486) and Stone Game II (LC 1140)** without looking anything up. LC 486 is a direct application; LC 1140 adds a "how many stones can I take" choice dimension, previewing the "extra dimension" idea that Merge Stones needs in full force.

5. **The reconstruction challenge:** after Matrix Chain Multiplication, extend your solution to print the actual OPTIMAL PARENTHESIZATION (not just the cost) by storing a `split[i][j]` table. This reconstruction technique transfers directly to printing the optimal triangulation or the optimal cut order.

---

## SECTION 23 — SIGN-OFF CRITERIA

### Tier 1 — Interval DP fundamentals mastered
You solve Longest Palindromic Subsequence and Palindrome Partitioning II cleanly. You can state, from memory, why the fill order must be by increasing length, and can produce the concrete failure trace from Section 3 on request.

### Tier 2 — Split-point recurrence internalized
You solve Matrix Chain Multiplication and Minimum Score Triangulation of Polygon without looking up the recurrence, and can explain why they're "the same problem" with a different cost formula. You correctly derive the `k` range (inclusive vs. exclusive) for a new problem without trial and error.

### Tier 3 — The "last, not first" reframe is second nature
You solve Burst Balloons and Minimum Cost to Cut a Stick, including the sentinel-padding trick, and can explain in one sentence why "last burst" makes the sub-problems independent while "first burst" does not. You solve Stone Game and correctly explain the sign flip in the score-difference recurrence.

### Tier 4 — Hard variants complete
You solve Minimum Cost to Merge Stones, correctly identifying why the extra pile-count dimension `m` is necessary and correctly deriving the `(len-1) % (K-1) == 0` feasibility guard. You solve Strange Printer with the "stretch a stroke" improvement. You can explain why Zuma Game does NOT reduce to a clean `dp[i][j]` table, in contrast to every other problem in this pattern.

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 312 (Burst Balloons — the "last, not first" reasoning must be explained unprompted), the Matrix Chain Multiplication archetype (the split-point recurrence must be derivable from scratch), and LC 877 (Stone Game — the score-difference sign flip must be explained). These three represent the three hardest conceptual challenges in this pattern.

---

*Pattern 23 Complete — Interval DP*
*Next: Pattern 24 — Bitmask DP (subset states, the Traveling Salesman shape, and profile DP)*
