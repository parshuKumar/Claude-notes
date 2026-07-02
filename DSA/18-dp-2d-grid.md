# PATTERN 18 — 2D DP AND GRID DP
## Grid Paths, Maximal Square, Reverse DP, Two-Agent DP — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of 2D DP:**

Most 2D DP problems look straightforward — "move from top-left to bottom-right." You define `dp[i][j]`, write a nested loop, and you're done. That's Pattern 18 at Tier 1.

The real test is in the **non-obvious variants:**

- **Dungeon Game (LC 174):** The direction is reversed. You must go from bottom-right to top-left because the minimum HP needed at each cell depends on what comes AFTER it, not before. Forward DP fails.
- **Cherry Pickup (LC 741):** Two agents traverse the same grid simultaneously. Modeling them as two separate 1D DPs is wrong — they interact (same cell can only be collected once). You need a shared state.
- **Maximal Square (LC 221):** Not a path problem at all. `dp[i][j]` is the side length of the largest square ending at (i,j) — a completely different semantic than path problems.

**The key skill:**

Before coding, answer:
1. Does my transition go forward (left-to-right, top-to-bottom) or backward?
2. Is this a "path ending at (i,j)" or "best using first i rows and j columns"?
3. Can I compress to O(min(m,n)) space using a rolling array?

If you answer these correctly, the code writes itself.

---

## SECTION 2 — THE 2D DP TAXONOMY

### Type 1: Grid Path DP
State: `dp[i][j]` = answer for paths FROM (0,0) TO (i,j)
Direction: Forward (fill top-to-bottom, left-to-right)
Transition: `dp[i][j] = f(dp[i-1][j], dp[i][j-1])`
Problems: Unique Paths, Unique Paths II, Min Path Sum, Minimum Falling Path Sum

### Type 2: Grid Property DP
State: `dp[i][j]` = a structural property OF the region ending at (i,j)
Direction: Forward
Transition: depends on neighbors — often `min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]) + 1`
Problems: Maximal Square, Count Square Submatrices

### Type 3: Triangle DP
State: `dp[i][j]` = minimum/maximum cost to reach row i, column j of a triangle
Direction: Bottom-up (last row → first row) OR forward with 1D optimization
Problems: Triangle, Pascal's Triangle

### Type 4: Reverse DP (answer depends on future state)
State: `dp[i][j]` = minimum resource needed AT (i,j) to successfully reach the end
Direction: Backward (fill bottom-right to top-left)
Transition: `dp[i][j] = max(1, min(dp[i+1][j], dp[i][j+1]) - grid[i][j])`
Problems: Dungeon Game

### Type 5: Two-Sequence DP (strings/arrays as dimensions)
State: `dp[i][j]` = answer for first i chars of s1 and first j chars of s2
Direction: Forward
Transition: depends on match/mismatch at (i,j)
Problems: LCS (Pattern 20), Edit Distance (Pattern 20), Interleaving String

### Type 6: Two-Agent DP
State: `dp[t][i][j]` = answer after t steps with agent 1 at row i, agent 2 at row j
Key insight: if total steps = t and agent 1 is at row i, agent 2 is at row j = t - i
Reduces to `dp[t][i]` or `dp[i][j]` (step is implicit)
Problems: Cherry Pickup, Grid with Two Agents

---

## SECTION 3 — TEMPLATE 1: UNIQUE PATHS (FOUNDATION 2D DP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 62 — Unique Paths
// m×n grid. Start at (0,0). End at (m-1,n-1). Move only right or down.
// Count all unique paths.
//
// State:    dp[i][j] = number of unique paths from (0,0) to (i,j)
// Transition: dp[i][j] = dp[i-1][j] + dp[i][j-1]
//             (came from above OR came from left)
// Base cases: dp[0][j] = 1 for all j (only way: go all right)
//             dp[i][0] = 1 for all i (only way: go all down)
// Answer: dp[m-1][n-1]
// ─────────────────────────────────────────────────────────────────

int uniquePaths(int m, int n) {
    // Initialize with 1s — first row and col are base cases
    vector<vector<int>> dp(m, vector<int>(n, 1));

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i-1][j] + dp[i][j-1];
        }
    }

    return dp[m-1][n-1];
}

// SPACE OPTIMIZED to O(n) — rolling array
int uniquePaths(int m, int n) {
    vector<int> dp(n, 1);  // represents current row; all 1s for first row

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j-1];  // dp[j] = (old dp[j] = above) + (dp[j-1] = left)
        }
    }

    return dp[n-1];
}
```

**FULL TRACE for m=3, n=3:**
```
Initialize dp (all 1s):
  [1, 1, 1]
  [1, 0, 0]
  [1, 0, 0]

i=1, j=1: dp[1][1] = dp[0][1] + dp[1][0] = 1+1 = 2
i=1, j=2: dp[1][2] = dp[0][2] + dp[1][1] = 1+2 = 3

  [1, 1, 1]
  [1, 2, 3]
  [1, 0, 0]

i=2, j=1: dp[2][1] = dp[1][1] + dp[2][0] = 2+1 = 3
i=2, j=2: dp[2][2] = dp[1][2] + dp[2][1] = 3+3 = 6

  [1, 1, 1]
  [1, 2, 3]
  [1, 3, 6]

Answer: 6 ✓
```

**Space optimization proof:** In the rolling array:
- Before inner loop at row i: `dp[j]` holds the value from row i-1 (= dp[i-1][j])
- `dp[j-1]` just got updated in the same inner loop iteration = dp[i][j-1]
- After `dp[j] += dp[j-1]`: dp[j] = dp[i-1][j] + dp[i][j-1] = correct 2D value
- Process left to right so dp[j-1] is always the current row's value when needed

---

## SECTION 4 — TEMPLATE 2: UNIQUE PATHS WITH OBSTACLES

```cpp
// LC 63 — Unique Paths II
// Same as Unique Paths but some cells are blocked (obstacleGrid[i][j] == 1).
// An obstacle means dp[i][j] = 0 (no paths through this cell).

int uniquePathsWithObstacles(vector<vector<int>>& grid) {
    int m = grid.size(), n = grid[0].size();

    // If start or end is blocked, answer is 0
    if (grid[0][0] == 1 || grid[m-1][n-1] == 1) return 0;

    vector<vector<int>> dp(m, vector<int>(n, 0));
    dp[0][0] = 1;

    // Fill first column
    for (int i = 1; i < m; i++)
        dp[i][0] = (grid[i][0] == 0) ? dp[i-1][0] : 0;

    // Fill first row
    for (int j = 1; j < n; j++)
        dp[0][j] = (grid[0][j] == 0) ? dp[0][j-1] : 0;

    // Fill rest
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            if (grid[i][j] == 1) {
                dp[i][j] = 0;  // obstacle — no paths
            } else {
                dp[i][j] = dp[i-1][j] + dp[i][j-1];
            }
        }
    }

    return dp[m-1][n-1];
}
```

**KEY EDGE CASE:** When filling the first row/column with obstacles, once a blocked cell is encountered, ALL subsequent cells in that row/column also have 0 paths (no way around the obstacle on first row/col). Example:

```
First row: [0, 0, 1, 0]
dp[0][0]=1, dp[0][1]=1, dp[0][2]=0 (blocked!), dp[0][3]=0 (no path past blocked cell)
```

This is handled correctly by the code: `dp[0][3] = dp[0][2] = 0`.

---

## SECTION 5 — TEMPLATE 3: MINIMUM PATH SUM

```cpp
// LC 64 — Minimum Path Sum
// Find the path (right or down only) that minimizes the sum of grid values.
//
// State:    dp[i][j] = minimum cost to reach (i,j) from (0,0)
// Transition: dp[i][j] = grid[i][j] + min(dp[i-1][j], dp[i][j-1])
// Base cases: dp[0][0] = grid[0][0]
//             dp[0][j] = dp[0][j-1] + grid[0][j]  (only path: go right)
//             dp[i][0] = dp[i-1][0] + grid[i][0]  (only path: go down)
// Answer: dp[m-1][n-1]

int minPathSum(vector<vector<int>>& grid) {
    int m = grid.size(), n = grid[0].size();

    // Modify grid in-place (treat as DP table) — saves O(mn) space
    // Or use separate dp array if in-place modification is not allowed

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (i == 0 && j == 0) continue;  // base case: already has grid[0][0]
            else if (i == 0) grid[i][j] += grid[i][j-1];   // first row
            else if (j == 0) grid[i][j] += grid[i-1][j];   // first column
            else grid[i][j] += min(grid[i-1][j], grid[i][j-1]);
        }
    }

    return grid[m-1][n-1];
}
```

**TRACE for grid=[[1,3,1],[1,5,1],[4,2,1]]:**
```
After in-place DP:
  [1,   3,  1]    → [1,   4,  5]
  [1,   5,  1]    → [2,   7,  6]
  [4,   2,  1]    → [6,   8,  7]

dp[0][0]=1
dp[0][1]=1+3=4, dp[0][2]=4+1=5
dp[1][0]=1+1=2
dp[1][1]=5+min(4,2)=5+2=7, dp[1][2]=1+min(5,7)=1+5=6
dp[2][0]=4+2=6
dp[2][1]=2+min(7,6)=2+6=8, dp[2][2]=1+min(6,8)=1+6=7

Answer: 7
Path: (0,0)→(0,1)→(0,2)→(1,2)→(2,2) = 1+3+1+1+1 = 7
```

**WHY in-place modification is risky in real code:** If the function is called multiple times (or grid is needed later), modifying in-place corrupts the input. Use a separate `dp` array in production/interview code unless you note the assumption.

---

## SECTION 6 — TEMPLATE 4: TRIANGLE DP

```cpp
// LC 120 — Triangle
// Given a triangular array, find the minimum path sum from top to bottom.
// Each step can go to adjacent cell in next row.
//
// Bottom-up approach: Start from the last row, compute upward.
// State: dp[j] = minimum sum to reach bottom from this cell
//         (we modify in-place or use the last row as initial dp)
// Transition: dp[j] = triangle[i][j] + min(dp[j], dp[j+1])
//             (going down-left or down-right)
// Answer: dp[0]

int minimumTotal(vector<vector<int>>& tri) {
    int n = tri.size();
    // Start with the last row as our DP
    vector<int> dp(tri.back());  // dp = last row values

    // Process from second-to-last row upward
    for (int i = n - 2; i >= 0; i--) {
        for (int j = 0; j <= i; j++) {
            dp[j] = tri[i][j] + min(dp[j], dp[j+1]);
            // dp[j] was dp from row i+1, position j (go right)
            // dp[j+1] was dp from row i+1, position j+1 (go left)
        }
    }

    return dp[0];
}
```

**TRACE for [[2],[3,4],[6,5,7],[4,1,8,3]]:**
```
n=4
Initial dp (last row): [4, 1, 8, 3]

i=2 (row [6,5,7]):
  j=0: dp[0] = 6 + min(dp[0],dp[1]) = 6 + min(4,1) = 7
  j=1: dp[1] = 5 + min(dp[1],dp[2]) = 5 + min(1,8) = 6
  j=2: dp[2] = 7 + min(dp[2],dp[3]) = 7 + min(8,3) = 10
  dp = [7, 6, 10, 3]

i=1 (row [3,4]):
  j=0: dp[0] = 3 + min(dp[0],dp[1]) = 3 + min(7,6) = 9
  j=1: dp[1] = 4 + min(dp[1],dp[2]) = 4 + min(6,10) = 10
  dp = [9, 10, 10, 3]

i=0 (row [2]):
  j=0: dp[0] = 2 + min(dp[0],dp[1]) = 2 + min(9,10) = 11
  dp = [11, 10, 10, 3]

Answer: dp[0] = 11
Path: 2→3→5→1 = 11 ✓
```

**WHY bottom-up is elegant here:** Working upward, each cell accumulates the minimum cost of the best path going downward from it. When we reach the apex (row 0), dp[0] = minimum total path. No 2D array needed — the 1D dp array is sufficient because we process rows from bottom to top, left to right.

**Alternative (top-down with memoization):**
```cpp
int n;
vector<vector<int>>* t;
vector<vector<int>> memo;

int solve(int row, int col) {
    if (row == n-1) return (*t)[row][col];
    if (memo[row][col] != INT_MAX) return memo[row][col];
    return memo[row][col] = (*t)[row][col] +
           min(solve(row+1, col), solve(row+1, col+1));
}
```

---

## SECTION 7 — TEMPLATE 5: MAXIMAL SQUARE (THE KEY INSIGHT)

This is a classic problem that requires a non-obvious state definition.

```cpp
// LC 221 — Maximal Square
// Find the largest square containing only '1's in a binary matrix.
//
// State:    dp[i][j] = side length of the largest square
//                      whose BOTTOM-RIGHT corner is at (i,j)
//
// KEY INSIGHT: If matrix[i][j] == '1', then:
//   dp[i][j] = min(dp[i-1][j],      ← square above
//                  dp[i][j-1],      ← square to the left
//                  dp[i-1][j-1])    ← square diagonally up-left
//              + 1
//
// WHY? For a square of side k to have bottom-right at (i,j):
//   - Square above (i-1,j) must have side ≥ k-1 (supplies the top rows)
//   - Square left (i,j-1) must have side ≥ k-1 (supplies the left cols)
//   - Square diagonal (i-1,j-1) must have side ≥ k-1 (ensures overlap is valid)
//   The minimum of the three limits the actual achievable square.
//
// Base cases: dp[0][j] = matrix[0][j] - '0' (top row: at most 1×1)
//             dp[i][0] = matrix[i][0] - '0' (left col: at most 1×1)

int maximalSquare(vector<vector<char>>& matrix) {
    int m = matrix.size(), n = matrix[0].size();
    int maxSide = 0;

    vector<vector<int>> dp(m, vector<int>(n, 0));

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (matrix[i][j] == '0') {
                dp[i][j] = 0;
            } else if (i == 0 || j == 0) {
                dp[i][j] = 1;  // base case: edge cells can only form 1×1 squares
            } else {
                dp[i][j] = min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]}) + 1;
            }
            maxSide = max(maxSide, dp[i][j]);
        }
    }

    return maxSide * maxSide;  // area = side²
}
```

**VISUAL PROOF of the min() formula:**

```
Consider this 4×4 binary matrix:
  1 1 1 1
  1 1 1 1
  1 1 1 1

At (2,3) [bottom-right]:
  dp[1][3] (above) = 2  ← the 2×2 square ending at (1,3)
  dp[2][2] (left)  = 3  ← the 3×3 square ending at (2,2)
  dp[1][2] (diag)  = 2  ← the 2×2 square ending at (1,2)

  dp[2][3] = min(2, 3, 2) + 1 = 3
  ← There IS a 3×3 square with bottom-right at (2,3):
    rows 0-2, cols 1-3 ✓

WHY min? Suppose dp[i-1][j]=2, dp[i][j-1]=3, dp[i-1][j-1]=2.
  The above square contributes 2 rows.
  The left square contributes 2 columns (limited by diagonal also being 2).
  We can only form a 3×3 if ALL three neighbors have dp ≥ 2 (sufficient to cover 3×3 minus current cell).
  Minimum = bottleneck = maximum achievable size.
```

**TRACE for a 3×3 all-'1' matrix:**
```
dp:
[1, 1, 1]
[1, 2, 2]
[1, 2, 3]

(2,2): min(dp[1][2]=2, dp[2][1]=2, dp[1][1]=2)+1 = 3
maxSide=3, area=9 ✓
```

**Space optimization:** Only need current row and previous row:
```cpp
vector<int> prev(n, 0), curr(n, 0);
for (int i = 0; i < m; i++) {
    for (int j = 0; j < n; j++) {
        if (matrix[i][j]=='0' || i==0 || j==0)
            curr[j] = (matrix[i][j]-'0');
        else
            curr[j] = min({prev[j], curr[j-1], prev[j-1]}) + 1;
        maxSide = max(maxSide, curr[j]);
    }
    swap(prev, curr);
    fill(curr.begin(), curr.end(), 0);
}
```

---

## SECTION 8 — TEMPLATE 6: DUNGEON GAME (REVERSE DP)

This is the prototypical "reverse DP" problem. Forward DP fails here.

```cpp
// LC 174 — Dungeon Game
// Knight starts at (0,0), must reach (m-1,n-1). Can only move right or down.
// Each cell has a value (positive = health gained, negative = health lost).
// Knight dies if HP drops to 0 at any point (need HP ≥ 1 always).
// Find minimum initial HP.
//
// WHY forward DP fails:
//   If we define dp[i][j] = min initial HP to reach (i,j), the path from (i,j)
//   to the destination still matters for calculating required HP at (i,j).
//   The "required HP" depends on future cells, not past cells.
//
// CORRECT state: dp[i][j] = minimum HP needed WHEN ENTERING cell (i,j)
//                            to survive all cells from (i,j) to (m-1,n-1)
//
// Transition (backward):
//   After leaving (i,j), go to (i+1,j) or (i,j+1).
//   We need: HP at (i,j) + dungeon[i][j] >= min(dp[i+1][j], dp[i][j+1])
//   → HP at (i,j) >= min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j]
//   → dp[i][j] = max(1, min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j])
//
//   The max(1, ...) ensures HP is always at least 1 (knight can't start dead).
//
// Base case: dp[m-1][n-1] = max(1, 1 - dungeon[m-1][n-1])
//            (need at least 1 HP after absorbing the last cell)
// Answer: dp[0][0]

int calculateMinimumHP(vector<vector<int>>& dungeon) {
    int m = dungeon.size(), n = dungeon[0].size();
    // Sentinel: dp[m][j] = dp[i][n] = INT_MAX (out of bounds = infinite HP needed)
    // Use INT_MAX/2 to avoid overflow
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, INT_MAX / 2));

    // Base case: just past the destination
    dp[m][n-1] = 1;  // trick: set dp[m][n-1]=1 so dp[m-1][n-1] computes correctly
    dp[m-1][n] = 1;

    for (int i = m - 1; i >= 0; i--) {
        for (int j = n - 1; j >= 0; j--) {
            int minHPNeeded = min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j];
            dp[i][j] = max(1, minHPNeeded);
        }
    }

    return dp[0][0];
}
```

**TRACE for dungeon=[[-2,-3,3],[-5,-10,1],[10,30,-5]]:**
```
m=3, n=3

Initialize: dp is 4×4, all INT_MAX/2
dp[3][2]=1, dp[2][3]=1

i=2, j=2: min(dp[3][2], dp[2][3]) - dungeon[2][2] = min(1,1) - (-5) = 1+5 = 6
           dp[2][2] = max(1, 6) = 6

i=2, j=1: min(dp[3][1], dp[2][2]) - dungeon[2][1] = min(INT_MAX/2, 6) - 30 = 6-30 = -24
           dp[2][1] = max(1, -24) = 1

i=2, j=0: min(dp[3][0], dp[2][1]) - dungeon[2][0] = min(INT_MAX/2, 1) - 10 = 1-10 = -9
           dp[2][0] = max(1, -9) = 1

i=1, j=2: min(dp[2][2], dp[1][3]) - dungeon[1][2] = min(6, INT_MAX/2) - 1 = 5
           dp[1][2] = max(1, 5) = 5

i=1, j=1: min(dp[2][1], dp[1][2]) - dungeon[1][1] = min(1, 5) - (-10) = 1+10 = 11
           dp[1][1] = max(1, 11) = 11

i=1, j=0: min(dp[2][0], dp[1][1]) - dungeon[1][0] = min(1,11) - (-5) = 1+5 = 6
           dp[1][0] = max(1, 6) = 6

i=0, j=2: min(dp[1][2], dp[0][3]) - dungeon[0][2] = min(5, INT_MAX/2) - 3 = 2
           dp[0][2] = max(1, 2) = 2

i=0, j=1: min(dp[1][1], dp[0][2]) - dungeon[0][1] = min(11, 2) - (-3) = 2+3 = 5
           dp[0][1] = max(1, 5) = 5

i=0, j=0: min(dp[1][0], dp[0][1]) - dungeon[0][0] = min(6, 5) - (-2) = 5+2 = 7
           dp[0][0] = max(1, 7) = 7

Answer: 7
```

**The sentinels `dp[m][n-1]=1` and `dp[m-1][n]=1`:** These represent "just past the end" — the boundary condition. When processing (m-1, n-1), we need `min(dp[m][n-1], dp[m-1][n])` = `min(1, 1)` = 1. So `dp[m-1][n-1] = max(1, 1 - dungeon[m-1][n-1])`. This correctly handles the last cell.

---

## SECTION 9 — TEMPLATE 7: INTERLEAVING STRING (TWO-SEQUENCE 2D DP)

```cpp
// LC 97 — Interleaving String
// Can s3 be formed by interleaving s1 and s2?
//
// State: dp[i][j] = can s3[0..i+j-1] be formed by interleaving s1[0..i-1] and s2[0..j-1]?
//        (using first i chars of s1 and first j chars of s2)
//
// Transition:
//   dp[i][j] = true if:
//     (s1[i-1] == s3[i+j-1] AND dp[i-1][j])  ← use s1's i-th char
//     OR
//     (s2[j-1] == s3[i+j-1] AND dp[i][j-1])  ← use s2's j-th char
//
// Base case: dp[0][0] = true (empty strings can form empty s3)
//            dp[i][0] = s1[0..i-1] == s3[0..i-1] (only s1 contributes)
//            dp[0][j] = s2[0..j-1] == s3[0..j-1] (only s2 contributes)
// Answer: dp[s1.size()][s2.size()]

bool isInterleave(string s1, string s2, string s3) {
    int m = s1.size(), n = s2.size();
    if (m + n != s3.size()) return false;

    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;

    // Base case: only s1 used
    for (int i = 1; i <= m; i++)
        dp[i][0] = dp[i-1][0] && (s1[i-1] == s3[i-1]);

    // Base case: only s2 used
    for (int j = 1; j <= n; j++)
        dp[0][j] = dp[0][j-1] && (s2[j-1] == s3[j-1]);

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            dp[i][j] = (s1[i-1] == s3[i+j-1] && dp[i-1][j]) ||
                       (s2[j-1] == s3[i+j-1] && dp[i][j-1]);
        }
    }

    return dp[m][n];
}
```

**WHY the index `s3[i+j-1]`?** When we've used i chars from s1 and j chars from s2, the next char of s3 to match is at position `i + j - 1` (0-indexed). This is the key alignment insight for interleaving problems.

**TRACE for s1="aab", s2="axy", s3="aaxaby":**
```
m=3, n=3

dp[0][0]=T
dp[1][0]=T&&(s1[0]='a'==s3[0]='a')=T
dp[2][0]=T&&(s1[1]='a'==s3[1]='a')=T
dp[3][0]=T&&(s1[2]='b'==s3[2]='x')=F

dp[0][1]=T&&(s2[0]='a'==s3[0]='a')=T
dp[0][2]=T&&(s2[1]='x'==s3[1]='a')=F
dp[0][3]=F (dp[0][2]=F already)

dp[1][1]: s1[0]='a'==s3[1]='a'→dp[0][1]=T → TRUE; or s2[0]='a'==s3[1]='a'→dp[1][0]=T → TRUE. dp[1][1]=T
dp[1][2]: s1[0]='a'==s3[2]='x'→F; s2[1]='x'==s3[2]='x'→dp[1][1]=T → TRUE. dp[1][2]=T
dp[1][3]: s1[0]='a'==s3[3]='a'→dp[0][3]=F; s2[2]='y'==s3[3]='a'→F. dp[1][3]=F

dp[2][1]: s1[1]='a'==s3[2]='x'→F; s2[0]='a'==s3[2]='x'→F. dp[2][1]=F
dp[2][2]: s1[1]='a'==s3[3]='a'→dp[1][2]=T → TRUE. dp[2][2]=T
dp[2][3]: s1[1]='a'==s3[4]='b'→F; s2[2]='y'==s3[4]='b'→F. dp[2][3]=F

dp[3][1]: s1[2]='b'==s3[3]='a'→F; s2[0]='a'==s3[3]='a'→dp[3][0]=F. dp[3][1]=F
dp[3][2]: s1[2]='b'==s3[4]='b'→dp[2][2]=T → TRUE. dp[3][2]=T
dp[3][3]: s1[2]='b'==s3[5]='y'→F; s2[2]='y'==s3[5]='y'→dp[3][2]=T → TRUE. dp[3][3]=T

Answer: dp[3][3] = true ✓
```

---

## SECTION 10 — TEMPLATE 8: MINIMUM FALLING PATH SUM (GRID WITH DIAGONAL MOVEMENT)

```cpp
// LC 931 — Minimum Falling Path Sum
// n×n matrix. Starting from any cell in row 0, choose one element per row.
// Each step can go to the cell directly below, or diagonally (below-left, below-right).
// Minimize total sum of chosen cells.
//
// State: dp[i][j] = min sum to reach (i,j) from any cell in row 0
// Transition: dp[i][j] = matrix[i][j] + min of three cells above:
//             min(dp[i-1][j-1], dp[i-1][j], dp[i-1][j+1])
//             (with boundary checks)
// Base cases: dp[0][j] = matrix[0][j]
// Answer: min(dp[n-1][0..n-1])

int minFallingPathSum(vector<vector<int>>& matrix) {
    int n = matrix.size();

    // Process in-place
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < n; j++) {
            int best = matrix[i-1][j];  // directly above
            if (j > 0) best = min(best, matrix[i-1][j-1]);   // diag-left
            if (j < n-1) best = min(best, matrix[i-1][j+1]); // diag-right
            matrix[i][j] += best;
        }
    }

    return *min_element(matrix[n-1].begin(), matrix[n-1].end());
}
```

**TRACE for [[2,1,3],[6,5,4],[7,8,9]]:**
```
Row 0: [2, 1, 3]  (base case)

i=1:
  j=0: above=2. no j-1. j+1=matrix[0][1]=1. best=min(2,1)=1. dp[1][0]=6+1=7
  j=1: above=1. j-1=2. j+1=3. best=min(1,2,3)=1. dp[1][1]=5+1=6
  j=2: above=3. j-1=1. no j+1. best=min(3,1)=1. dp[1][2]=4+1=5

Row 1: [7, 6, 5]

i=2:
  j=0: above=7. j+1=6. best=min(7,6)=6. dp[2][0]=7+6=13
  j=1: above=6. j-1=7. j+1=5. best=min(6,7,5)=5. dp[2][1]=8+5=13
  j=2: above=5. j-1=6. best=min(5,6)=5. dp[2][2]=9+5=14

Row 2: [13, 13, 14]

Answer: min(13,13,14) = 13
Path: (0,2)→(1,2)→(2,1) = 3+4+8 = 15... Wait.
Actually: (0,1)→(1,2)→(2,1) = 1+4+8 = 13. Yes! ✓
```

---

## SECTION 11 — TEMPLATE 9: CHERRY PICKUP (TWO SIMULTANEOUS AGENTS)

This is one of the hardest 2D DP problems. The key insight is treating two agents moving simultaneously as a SINGLE state.

```cpp
// LC 741 — Cherry Pickup
// Grid with cherries (1), empty (0), or thorns (-1).
// Start at (0,0). Collect all possible cherries going to (n-1,n-1),
// then return back to (0,0). (Equivalent: two agents both go from (0,0) to (n-1,n-1).)
//
// WRONG approach: solve forward trip then backward trip separately.
// Greedy doesn't work — best forward path might block the best backward path.
//
// CORRECT KEY INSIGHT: model two agents BOTH going from (0,0) to (n-1,n-1)
// simultaneously. After t steps:
//   Agent 1 is at (r1, c1) where r1+c1 = t
//   Agent 2 is at (r2, c2) where r2+c2 = t
// So both agents have taken the same number of steps t.
// We only need to track r1 and r2 (since c = t - r for each).
//
// State: dp[t][r1][r2] = max cherries collected when:
//          both agents have taken t steps,
//          agent 1 is at row r1 (col c1 = t-r1),
//          agent 2 is at row r2 (col c2 = t-r2)
//
// Optimization: dp[r1][r2] (t is implicit, updated in outer loop)

int cherryPickup(vector<vector<int>>& grid) {
    int n = grid.size();
    // dp[r1][r2] = max cherries with agent1 at row r1, agent2 at row r2
    // both have taken the same number of steps (implicit)
    const int NEG_INF = INT_MIN / 2;
    vector<vector<int>> dp(n, vector<int>(n, NEG_INF));
    dp[0][0] = grid[0][0];

    for (int t = 1; t <= 2 * (n-1); t++) {
        vector<vector<int>> ndp(n, vector<int>(n, NEG_INF));

        // r1 ranges from max(0, t-(n-1)) to min(n-1, t)
        for (int r1 = max(0, t-(n-1)); r1 <= min(n-1, t); r1++) {
            int c1 = t - r1;
            if (c1 < 0 || c1 >= n) continue;
            if (grid[r1][c1] == -1) continue;  // thorn — agent 1 can't be here

            for (int r2 = r1; r2 <= min(n-1, t); r2++) {  // r2 >= r1 to avoid duplicate states
                int c2 = t - r2;
                if (c2 < 0 || c2 >= n) continue;
                if (grid[r2][c2] == -1) continue;  // thorn — agent 2 can't be here

                // Cherries collected at this step
                int cherries = grid[r1][c1];
                if (r1 != r2) cherries += grid[r2][c2];  // different cells: add both
                // (if same cell, cherries already taken — only count once)

                // Previous states: both agents came from either above or left
                int best = NEG_INF;
                for (int pr1 : {r1-1, r1}) {       // agent 1: came from above or left
                    for (int pr2 : {r2-1, r2}) {   // agent 2: came from above or left
                        int pc1 = t - 1 - pr1;
                        int pc2 = t - 1 - pr2;
                        if (pr1 >= 0 && pc1 >= 0 && pc1 < n &&
                            pr2 >= 0 && pc2 >= 0 && pc2 < n) {
                            if (dp[pr1][pr2] != NEG_INF) {
                                best = max(best, dp[pr1][pr2]);
                            }
                        }
                    }
                }

                if (best != NEG_INF) {
                    ndp[r1][r2] = max(ndp[r1][r2], best + cherries);
                }
            }
        }
        dp = ndp;
    }

    return max(0, dp[n-1][n-1]);
}
```

**THE KEY INSIGHT visualized:**

```
Two agents, both taking t steps:
t=0: Both at (0,0). dp[0][0] = grid[0][0].
t=1: Agent 1 can be at (0,1) or (1,0).
     Agent 2 can be at (0,1) or (1,0).
     States: (r1=0,r2=0)[→(0,1),(0,1)], (r1=0,r2=1)[→(0,1),(1,0)], (r1=1,r2=1)[→(1,0),(1,0)]

WHY r2 >= r1 (symmetry reduction):
  State (r1=0, r2=1) means: agent1 at (0,1), agent2 at (1,0).
  State (r1=1, r2=0) means: agent1 at (1,0), agent2 at (0,1).
  These are EQUIVALENT (agents are interchangeable).
  By requiring r2 >= r1, we halve the state space.
```

---

## SECTION 12 — COMPLEXITY TABLE

| Problem | Time | Space | Optimized Space | Notes |
|---------|------|-------|-----------------|-------|
| Unique Paths | O(mn) | O(mn) | O(n) — rolling row | Pure path DP |
| Unique Paths II | O(mn) | O(mn) | O(n) — rolling row | Obstacles: dp=0 |
| Min Path Sum | O(mn) | O(mn) | O(n) or in-place | Standard path |
| Triangle | O(n²) | O(n²) | O(n) — bottom row | Reduce upward |
| Maximal Square | O(mn) | O(mn) | O(n) — 2 rows | Side length DP |
| Dungeon Game | O(mn) | O(mn) | O(n) — backward fill | Reverse DP |
| Interleaving String | O(mn) | O(mn) | O(n) — rolling | Two-sequence |
| Min Falling Path Sum | O(n²) | O(n²) | O(1) in-place | Answer = min of last row |
| Cherry Pickup | O(n³) | O(n²) | O(n²) — 2 layers | Hard — two agents |
| Maximal Rectangle | O(mn) | O(n) | O(n) | Uses histogram per row |

---

## SECTION 13 — VARIANT: MAXIMAL RECTANGLE (HARDEST 2D DP COMBINATION)

```cpp
// LC 85 — Maximal Rectangle
// Find the largest rectangle of 1s in a binary matrix.
//
// Key insight: For each row, compute the "height" array:
//   height[j] = number of consecutive 1s in column j ending at current row
// Then apply Largest Rectangle in Histogram (Pattern 04) on each row.
//
// The histogram approach from Pattern 04 + height array from DP = maximal rectangle.

int maximalRectangle(vector<vector<char>>& matrix) {
    if (matrix.empty()) return 0;
    int m = matrix.size(), n = matrix[0].size();
    int maxArea = 0;
    vector<int> heights(n, 0);

    for (int i = 0; i < m; i++) {
        // Update heights
        for (int j = 0; j < n; j++) {
            heights[j] = (matrix[i][j] == '1') ? heights[j] + 1 : 0;
        }

        // Largest Rectangle in Histogram (monotonic stack from Pattern 04)
        stack<int> stk;
        for (int j = 0; j <= n; j++) {
            int h = (j == n) ? 0 : heights[j];  // sentinel 0 at end
            while (!stk.empty() && heights[stk.top()] > h) {
                int height = heights[stk.top()]; stk.pop();
                int width = stk.empty() ? j : j - stk.top() - 1;
                maxArea = max(maxArea, height * width);
            }
            stk.push(j);
        }
    }

    return maxArea;
}
```

**WHY this connection works:** Each row of the matrix defines a histogram where `heights[j]` is the number of consecutive 1s above (and including) the current row in column j. The largest rectangle of 1s that has its BOTTOM in the current row is exactly the largest rectangle in this histogram.

By iterating over all rows, we ensure every possible bottom row is considered. The maximum over all rows is the answer.

---

## SECTION 14 — COMMON MISTAKES

### Mistake 1: Wrong direction — forward DP on Dungeon Game

```cpp
// WRONG — forward DP for Dungeon Game
vector<vector<int>> dp(m, vector<int>(n, 0));
dp[0][0] = max(1, 1 - dungeon[0][0]);
// This tries to compute "min HP to arrive at (i,j)"
// But the minimum HP needed at any cell depends on future cells!
// A cell with +100 health might come after a cell with -1000 health —
// you need to know about the +100 when deciding HP for the -1000 cell.

// CORRECT — backward DP
// Start from (m-1,n-1) and work backward to (0,0)
// dp[i][j] = min HP needed ENTERING (i,j) to survive to the end
```

---

### Mistake 2: Maximal Square — wrong formula variants

```cpp
// WRONG — using max instead of min
dp[i][j] = max({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]}) + 1;
// This would expand the square even when one side doesn't support it.
// A square of side k requires ALL three neighbors to have side ≥ k-1.
// The MINIMUM is the bottleneck.

// WRONG — forgetting the diagonal
dp[i][j] = min(dp[i-1][j], dp[i][j-1]) + 1;
// Without dp[i-1][j-1], we might think a 3×3 exists when the diagonal
// corner doesn't have a 2×2 square. The diagonal ensures the "overlap" is valid.

// CORRECT
dp[i][j] = min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]}) + 1;
```

---

### Mistake 3: Triangle — wrong loop direction for bottom-up

```cpp
// WRONG — processing top to bottom with 1D array
vector<int> dp(tri[0]);
for (int i = 1; i < n; i++) {
    for (int j = 0; j <= i; j++) {  // BUG: processing left to right
        dp[j] = tri[i][j] + min(dp[j], dp[j+1]);
        // When j > 0, dp[j] was already updated from dp[j-1]!
        // The "old dp[j]" (from row i-1) was overwritten.
    }
}

// CORRECT — process right to left OR use bottom-up (last row → first row)
// Bottom-up naturally avoids this since we only look forward (dp[j] and dp[j+1]):
for (int i = n-2; i >= 0; i--)
    for (int j = 0; j <= i; j++)
        dp[j] = tri[i][j] + min(dp[j], dp[j+1]);
```

---

### Mistake 4: Cherry Pickup — solving forward + backward separately

```cpp
// WRONG — greedy: solve forward trip optimally, then backward trip
// on the remaining grid after removing collected cherries
// This doesn't work because the optimal forward path might block
// a better backward path that together would collect more cherries.

// EXAMPLE WHERE GREEDY FAILS:
// Grid: [[1,1,1,0,0],
//         [0,0,1,0,1],
//         [1,0,0,0,1],
//         [0,0,0,0,1],
//         [0,0,1,1,1]]
// Best forward path might miss cherries in row 2 col 0
// that the backward path would otherwise catch.

// CORRECT: Two simultaneous agents from (0,0) to (n-1,n-1).
// Use the shared state dp[r1][r2] per step t.
```

---

### Mistake 5: Interleaving String — wrong index for s3

```cpp
// WRONG — using wrong index for s3
if (s1[i-1] == s3[i-1] && dp[i-1][j])   // BUG: should be s3[i+j-1]
    dp[i][j] = true;

// When i chars of s1 and j chars of s2 have been used,
// the NEXT char of s3 to match is at position i+j-1 (0-indexed).
// Using i-1 ignores the j characters from s2 that precede this position.

// CORRECT
if (s1[i-1] == s3[i+j-1] && dp[i-1][j])
    dp[i][j] = true;
```

---

### Mistake 6: Space optimization breaking the transition

```cpp
// WRONG — space optimizing Unique Paths with obstacles by reusing dp in-place
// in a way that uses already-updated values
vector<int> dp(n, 0);
dp[0] = 1;
for (int i = 0; i < m; i++) {
    for (int j = 0; j < n; j++) {
        if (grid[i][j] == 1) { dp[j] = 0; continue; }
        if (j > 0) dp[j] += dp[j-1];
        // BUG: dp[j-1] is CURRENT ROW (just updated), not previous row
        // But for paths problems, dp[j-1] SHOULD be the current row!
        // (This is actually fine for Unique Paths, but for Min Path Sum
        // or problems where you need BOTH above and left from SAME row,
        // you need to keep track of previous dp[j-1])
    }
}
// For Unique Paths: dp[j-1] = current row left = CORRECT (paths from left)
//                   dp[j] before update = previous row = CORRECT (paths from above)
// This works! But for Min Path Sum, need separate prev_row tracking
// since min(above, left) needs both independently.
```

---

## SECTION 15 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Unique Paths | 62 | Basic 2D DP, space optimization to 1D |
| 2 | Unique Paths II | 63 | Handle obstacles, first row/col edge cases |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Minimum Path Sum | 64 | Path DP with costs, in-place modification |
| 4 | Triangle | 120 | Bottom-up 1D DP, process upward |
| 5 | Maximal Square | 221 | Side length DP, WHY min(top, left, diag) |
| 6 | Dungeon Game | 174 | Reverse DP, max(1, ...) sentinel |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Interleaving String | 97 | Two-sequence DP, index s3[i+j-1] |
| 8 | Minimum Falling Path Sum | 931 | Diagonal movement, boundary checks |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Longest Common Subsequence | 1143 | Two-sequence DP (preview of Pattern 20) |
| 10 | Cherry Pickup | 741 | Two simultaneous agents, shared state |
| 11 | Maximal Rectangle | 85 | Height array + histogram (P04 + P18 combo) |

---

## SECTION 16 — PATTERN CONNECTIONS

1. **Pattern 17 (1D DP):** Every 2D DP can be space-optimized to a 1D rolling array when the transition only uses the previous row. Pattern 17's space optimization technique (two variables) generalizes to "previous row" in Pattern 18.

2. **Pattern 04 (Monotonic Stack):** Maximal Rectangle (LC 85) directly combines Largest Rectangle in Histogram (Pattern 04) with the height array accumulation from DP. This is the clearest example of combining two separate patterns.

3. **Pattern 20 (LCS Family):** Longest Common Subsequence, Edit Distance, and Interleaving String are all two-sequence 2D DP problems. Pattern 18 introduces Interleaving String; Pattern 20 covers the full LCS family with reconstruction and all variants.

4. **Pattern 19 (Knapsack):** The 2D knapsack (Ones and Zeroes, LC 474) uses exactly the same 2D DP structure as Unique Paths — state `dp[i][j]` with two dimensions representing two resources. The connection is: "path DP" and "knapsack DP" both use 2D tables; the only difference is what the dimensions represent.

5. **Pattern 22 (DP on Trees):** Trees are essentially hierarchical grids. The "bottom-up" approach in Triangle DP (process last row upward) directly mirrors the "postorder DP" on trees (process leaves upward to root). Both accumulate information from children to parents.

6. **Pattern 06 (Prefix Sums):** 2D prefix sums (LC 304) can answer "sum of any rectangle in O(1)" and is the foundation for some advanced 2D DP optimizations. Range Sum Query 2D from Pattern 06 is a tool that appears inside DP solutions for problems like Max Sum of Rectangle No Larger Than K.

---

## SECTION 17 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 221 — Maximal Square

**Interviewer:** "Find the largest square containing only 1s in a binary matrix."

**Candidate:**

*[First minute — state the non-obvious insight]*
> "The naive brute force is O(mn × min(m,n)²) — for each cell, try to expand a square. DP reduces this. The state: dp[i][j] = side length of the largest square whose bottom-right corner is at (i,j)."

*[Explain the min formula before coding]*
> "For a k×k square with bottom-right at (i,j), we need: a (k-1)×(k-1) or larger square above, to the left, AND diagonally. The minimum of these three is the bottleneck. So dp[i][j] = min(top, left, diagonal) + 1."

*[Code — 4 minutes]*
```cpp
int maximalSquare(vector<vector<char>>& matrix) {
    int m = matrix.size(), n = matrix[0].size(), maxSide = 0;
    vector<vector<int>> dp(m, vector<int>(n, 0));
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (matrix[i][j] == '0') continue;
            dp[i][j] = (i==0||j==0) ? 1 :
                       min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]}) + 1;
            maxSide = max(maxSide, dp[i][j]);
        }
    }
    return maxSide * maxSide;
}
```

*[Follow-up]: "What if we want to COUNT all square submatrices, not just the largest?"*
> "Observation: dp[i][j] represents not just the largest square ending at (i,j), but also dp[i][j]-1 smaller squares of side dp[i][j]-1, dp[i][j]-2, ..., 1. So the count of all square submatrices = sum of all dp[i][j] values."

**RED FLAGS:**
- Using max() instead of min() in the transition
- Forgetting the diagonal neighbor (only using top and left)
- Returning maxSide instead of maxSide * maxSide (area vs side)

---

### Interview Simulation 2: LC 174 — Dungeon Game

**Interviewer:** "Knight starts at (0,0) with some HP, must survive to (m-1,n-1). Cells add or subtract HP. Knight dies if HP ≤ 0 at any time. Find minimum initial HP."

**Candidate:**

*[Critical: identify why forward DP fails first]*
> "My first instinct is forward DP: dp[i][j] = min HP to reach (i,j). But this doesn't work. Consider a cell with +1000 health at the end: if I know that, I need less HP earlier. The required HP at each cell depends on FUTURE cells. So I must go backward."

*[State the backward DP clearly]*
> "Backward DP: dp[i][j] = minimum HP needed WHEN ENTERING cell (i,j) to survive everything from (i,j) to the end. Transition: `dp[i][j] = max(1, min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j])`. The max(1,...) ensures HP is at least 1."

*[Code — 6 minutes]*
```cpp
int calculateMinimumHP(vector<vector<int>>& dungeon) {
    int m = dungeon.size(), n = dungeon[0].size();
    vector<vector<int>> dp(m+1, vector<int>(n+1, INT_MAX/2));
    dp[m][n-1] = dp[m-1][n] = 1;
    for (int i=m-1; i>=0; i--)
        for (int j=n-1; j>=0; j--)
            dp[i][j] = max(1, min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j]);
    return dp[0][0];
}
```

*[Interviewer]: "Why is the sentinel dp[m][n-1] = dp[m-1][n] = 1 and not 0?"*
> "At the last cell (m-1,n-1), the minimum HP needed is max(1, 1 - dungeon[m-1][n-1]). For this to compute correctly using the transition `max(1, min(dp[m][n-1], dp[m-1][n]) - dungeon[m-1][n-1])`, we need `min(1, 1) = 1`. This gives the correct result. Using 0 as sentinels would make the formula `max(1, min(0,0) - dungeon[m-1][n-1]) = max(1, -dungeon[m-1][n-1])` — wrong when dungeon value is negative."

**RED FLAGS:**
- Attempting forward DP without recognizing why it fails
- Not including the max(1, ...) — allowing HP of 0 or negative
- Wrong sentinels at the boundary

---

### Interview Simulation 3: Meta-discussion — When is 2D DP the right tool?

**Interviewer:** "How do you recognize that a problem needs 2D DP versus 1D DP?"

**Candidate:**
> "I look for two independent dimensions in the state. If the answer at position (i,j) depends ONLY on the position, and there are two independent indices, it's likely 2D DP.

> For grid problems: the two dimensions are row and column. Natural.

> For two-sequence problems (LCS, Edit Distance, Interleaving): dimension 1 = how many chars of string 1 consumed, dimension 2 = how many chars of string 2 consumed.

> For two-constraint problems (Knapsack with weight and count): dimension 1 = weight used, dimension 2 = count used.

> The signal for 2D DP over 1D DP: the transition touches two independent previous states — like `dp[i-1][j]` AND `dp[i][j-1]` (both needed, neither is reducible to a single index). If the transition only uses `dp[i-1]` and `dp[i-2]` along a single dimension, it's 1D DP."

**RED FLAGS:**
- Not being able to articulate WHY two dimensions are needed
- Confusing "2D grid problem" with "2D DP" — not all grid problems need 2D DP (some only need 1D or even O(1) DP)

---

## SECTION 18 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: In Maximal Square, why is `min(top, left, diagonal) + 1` the correct formula?**

> For a square of side k to have its bottom-right corner at (i,j):
> - The cell at (i-1, j) must be the bottom-right of a square of side ≥ k-1 (this provides the k-1 rows above row i, spanning k columns ending at j)
> - The cell at (i, j-1) must be the bottom-right of a square of side ≥ k-1 (this provides the k-1 columns to the left of j, spanning k rows ending at i)
> - The cell at (i-1, j-1) must be the bottom-right of a square of side ≥ k-1 (this ensures the upper-left corner region is filled)
>
> All three constraints must hold simultaneously. The minimum of the three is the maximum k-1 we can guarantee for ALL three, giving the maximum achievable k = min + 1.

---

**Q2: Dungeon Game requires backward DP. Give a concrete example where forward DP fails.**

> Grid: `[[-3, 5]]`. One row, two columns. Knight must go right.
>
> Forward DP attempt: `dp[0][0] = max(1, 1 - (-3)) = 4`. "Need 4 HP to survive cell (0,0)."
> Then: `dp[0][1] = ?`. We know we enter (0,1) with HP = 4 + (-3) = 1 HP. Adding +5 gives us 6 HP. But the REQUIRED initial HP is just 4?
>
> Wait, actually let me construct a failing example: `[[1, -3]]`. 
> Forward: dp[0][0] = max(1, 1-1) = 1. "Need 1 HP to survive (0,0), and gain 1 HP."
> Enter (0,1) with 1+1=2 HP. After -3: 2-3 = -1 ≤ 0. DEAD.
>
> The actual minimum starting HP is max(1, min(2)-1) = max(1, -1) going backward = actually dp[0][1]=max(1,1-(-3))=4. dp[0][0]=max(1, 4-1)=3.
> So starting with 3 HP: enter (0,0) with 3 HP, gain 1 → 4 HP. Enter (0,1) with 4 HP, lose 3 → 1 HP. ALIVE.
> Forward DP gives "1 HP is enough" — wrong. Backward DP gives 3 — correct.

---

**Q3: In the rolling array optimization for Unique Paths, what does dp[j] represent before and after the inner loop iteration?**

> Before processing row i, inner loop position j: `dp[j]` = number of paths from (0,0) to (i-1, j) = "paths from above."
> After `dp[j] += dp[j-1]`: `dp[j]` = number of paths from (0,0) to (i, j).
>
> `dp[j-1]` at this point is already `dp[i][j-1]` (updated in the same row iteration), which is the "paths from left." So the addition `dp[j] += dp[j-1]` correctly computes `dp[i][j] = dp[i-1][j] + dp[i][j-1]`.
>
> The left-to-right order is crucial: `dp[j-1]` must be updated BEFORE `dp[j]` so that the current row's left value is used.

---

**Q4: Why does Cherry Pickup use the state dp[r1][r2] with an implicit step t, instead of tracking r1, c1, r2, c2 all explicitly?**

> Both agents take the same number of steps t. After t steps:
> - Agent 1 is at (r1, c1) where r1 + c1 = t (constraint: only moves right or down)
> - Agent 2 is at (r2, c2) where r2 + c2 = t (same constraint)
>
> So given r1 and t, c1 = t - r1 is determined. Similarly c2 = t - r2.
> We don't need to track c1 and c2 separately — they're implicit in r1, r2, and t.
> This reduces the state from 4D (r1, c1, r2, c2) to 3D (t, r1, r2), and by iterating t in the outer loop and keeping dp[r1][r2] as a 2D array, to 2D with an outer loop.

---

**Q5: For Triangle bottom-up DP, why must we process each row's inner loop left-to-right (not right-to-left)?**

> When computing `dp[j] = tri[i][j] + min(dp[j], dp[j+1])`:
> - `dp[j]` (before update) = best path from (i+1, j) going down — this is the "go right" option
> - `dp[j+1]` = best path from (i+1, j+1) going down — this is the "go left" option (in triangle indexing)
>
> After updating dp[j], it holds the best path from (i, j) going down.
> When computing dp[j+1] next, we need the OLD dp[j+1] (from row i+1) and dp[j+2].
> dp[j+1] HASN'T been updated yet (left-to-right means j+1 is processed after j).
> So `dp[j+1]` is still from row i+1 — correct.
>
> Processing right-to-left would update dp[j+1] before dp[j], and when computing dp[j], dp[j+1] would already be the current-row (row i) value — wrong.

---

**Q6: In Interleaving String, what does dp[i][j] = true mean exactly?**

> `dp[i][j] = true` means: s3[0..i+j-1] (the first i+j characters of s3) can be formed by interleaving s1[0..i-1] (first i chars) and s2[0..j-1] (first j chars).
>
> When i=3 and j=2: we've "consumed" 3 chars from s1 and 2 chars from s2, and successfully matched the first 5 chars of s3. The specific character at s3 that we're matching next would be at index i+j = 5.
>
> dp[0][0] = true: empty s1 and empty s2 form empty s3 — trivially true.

---

**Q7: When can you always apply the rolling array optimization to reduce a 2D DP to 1D?**

> The rolling array optimization works when: `dp[i][j]` depends only on cells in row i-1 (and possibly cells in the same row i to the left, i.e., dp[i][j-1]).
>
> Specifically: if `dp[i][j] = f(dp[i-1][j], dp[i-1][j-1], dp[i][j-1])` with possibly some terms, and row i+1, i+2, ... are NOT needed when computing row i, then one 1D array suffices.
>
> It does NOT work when: the transition requires cells from 3+ rows simultaneously (rare but possible), or when processing in reverse requires forward-looking cells.
>
> For Maximal Square: dp[i][j] uses dp[i-1][j], dp[i][j-1], dp[i-1][j-1]. The dp[i-1][j-1] term (diagonal) requires careful handling with a temporary variable `prev` to hold the "about-to-be-overwritten" dp[i-1][j-1] value before it's updated to dp[i][j-1].

---

## SECTION 19 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. GRID PATH DP (Unique Paths / Min Path Sum)
// ─────────────────────────────────────────────────────────────────
// Forward, top-left to bottom-right
// dp[i][j] = f(dp[i-1][j], dp[i][j-1])
// Base: first row = all 1s (or sum of row), first col = all 1s (or sum of col)
// Space optimize: 1D rolling array, left-to-right inner loop

vector<int> dp(n, 1);   // initialize for Unique Paths
for (int i = 1; i < m; i++)
    for (int j = 1; j < n; j++)
        dp[j] += dp[j-1];

// ─────────────────────────────────────────────────────────────────
// 2. MAXIMAL SQUARE
// ─────────────────────────────────────────────────────────────────
dp[i][j] = (matrix[i][j]=='0') ? 0 :
           (i==0||j==0) ? 1 :
           min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]}) + 1;
// Answer = maxSide * maxSide

// ─────────────────────────────────────────────────────────────────
// 3. TRIANGLE (bottom-up 1D)
// ─────────────────────────────────────────────────────────────────
vector<int> dp(tri.back());
for (int i=n-2; i>=0; i--)
    for (int j=0; j<=i; j++)
        dp[j] = tri[i][j] + min(dp[j], dp[j+1]);
return dp[0];

// ─────────────────────────────────────────────────────────────────
// 4. DUNGEON GAME (reverse DP)
// ─────────────────────────────────────────────────────────────────
// Sentinels at boundary
dp[m][n-1] = dp[m-1][n] = 1;
// Process backward
for (int i=m-1; i>=0; i--)
    for (int j=n-1; j>=0; j--)
        dp[i][j] = max(1, min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j]);
return dp[0][0];

// ─────────────────────────────────────────────────────────────────
// 5. TWO-SEQUENCE DP (Interleaving String)
// ─────────────────────────────────────────────────────────────────
// Index into s3 is always i+j-1 (when i chars of s1 and j chars of s2 used)
dp[i][j] = (s1[i-1]==s3[i+j-1] && dp[i-1][j]) ||
            (s2[j-1]==s3[i+j-1] && dp[i][j-1]);

// ─────────────────────────────────────────────────────────────────
// 6. HEIGHT ARRAY FOR MAXIMAL RECTANGLE
// ─────────────────────────────────────────────────────────────────
for (int j=0; j<n; j++)
    heights[j] = (matrix[i][j]=='1') ? heights[j]+1 : 0;
// Then apply largest rectangle in histogram (Pattern 04)

// ─────────────────────────────────────────────────────────────────
// 7. REVERSE DP PATTERN
// ─────────────────────────────────────────────────────────────────
// Use when: answer at (i,j) depends on future cells (right or below)
// Direction: bottom-right → top-left
// Sentinels: set dp at boundary to 1 or desired initial value

// ─────────────────────────────────────────────────────────────────
// 8. DECISION TABLE FOR 2D DP DIRECTION
// ─────────────────────────────────────────────────────────────────
// Forward (top-left → bottom-right):
//   "best path FROM (0,0) TO (i,j)" — Unique Paths, Min Path Sum
// Backward (bottom-right → top-left):
//   "best from (i,j) TO (m-1,n-1)" — Dungeon Game
// Bottom-up 1D (last-row → first-row):
//   "accumulate from bottom" — Triangle
```

---

## SECTION 20 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Dungeon Game and Cherry Pickup are the two that require non-trivial insight. Budget 45 minutes each.

2. **Practice the reverse DP mindset:** After finishing Dungeon Game, try to solve it with forward DP and explicitly demonstrate why it fails. Understanding failure modes deepens understanding of the correct approach.

3. **Maximal Square → Count Square Submatrices (LC 1277):** These two problems have the same dp[i][j] = side length DP table. The only difference is the answer: Maximal Square returns max² while Count Square Submatrices returns sum of dp. Solve 1277 immediately after 221 to cement this.

4. **Preview Pattern 20 (LCS):** Interleaving String (LC 97, in this pattern) is the warmup for the two-sequence 2D DP family. Pattern 20 covers LCS, Edit Distance, Distinct Subsequences — all use `dp[i][j]` for two sequences.

5. **The 2D space optimization challenge:** After finishing the pattern, space-optimize Maximal Square to O(n) space using the "diagonal rollover" technique (track the `prev` variable before overwriting). This is a common interview follow-up.

---

## SECTION 21 — SIGN-OFF CRITERIA

### Tier 1 — Grid path mastered
You solve Unique Paths (with and without obstacles) and Min Path Sum cleanly, space-optimized to O(n). You can explain the rolling array compression in under 30 seconds.

### Tier 2 — Non-path variants understood
You solve Maximal Square using the min(top, left, diag)+1 formula and can explain the geometric proof. You solve Triangle using bottom-up 1D DP without looking up the code.

### Tier 3 — Non-trivial patterns solid
You solve Dungeon Game (backward DP) and correctly identify why forward DP fails. You solve Interleaving String with the correct `s3[i+j-1]` index. You solve Minimum Falling Path Sum with boundary guards.

### Tier 4 — Hard variants complete
You solve Cherry Pickup using the two-simultaneous-agent state. You solve Maximal Rectangle by combining height arrays with Pattern 04's histogram approach. You can explain the state space reduction in Cherry Pickup (why r2 ≥ r1).

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 221 (Maximal Square — the min formula must be explained), LC 174 (Dungeon Game — backward DP reasoning), and LC 97 (Interleaving String — two-sequence DP). These three represent the three hardest conceptual challenges in this pattern.

---

*Pattern 18 Complete — 2D DP and Grid DP*
*Next: Pattern 19 — Knapsack Family (0/1, Unbounded, 2D Knapsack — the DP disguises)*
