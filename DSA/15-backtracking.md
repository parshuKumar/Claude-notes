# PATTERN 15 — BACKTRACKING
## Complete Grandmaster Guide — Subsets, Permutations, Combinations, Constraints
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**What most people get wrong about backtracking:**

They memorize three separate templates — one for subsets, one for permutations, one for combinations — and treat them as three completely different algorithms. When a problem has a twist (duplicates, constraints, pruning), they freeze.

The truth: **all backtracking is one algorithm**. You are traversing a decision tree. At each node, you make a choice, recurse, then undo that choice. The only difference between problems is:
1. **What choices are available** at each node
2. **When you record a result** (leaf node vs any node)
3. **What constraints** eliminate choices early (pruning)
4. **How you handle duplicates** (sort + skip same choice at same level)

Once you see this unified view, all backtracking problems become variations on a single theme.

**The brutal honest truth about backtracking in interviews:**

Backtracking problems are deceptively easy to write wrong and right in the same attempt. The code looks identical for correct and incorrect solutions — the bug is subtle (wrong base case, missing undo, duplicate not skipped properly). The skill is not writing the code; it's being able to **trace through your decision tree** mentally and verify correctness before submitting.

**Where backtracking appears in contests:**

Codeforces Div 2 B/C: Brute force with pruning often passes for N ≤ 20. The constraint `N ≤ 20` or `N ≤ 15` is your signal. Backtracking with good pruning can handle:
- 2^20 = ~10^6 states (fast, always passes)
- 2^25 = ~33 × 10^6 states (borderline, need tight pruning)
- 2^30 = ~10^9 states (too slow without massive pruning or bitmask DP)

---

## SECTION 2 — ELI5

Imagine you're solving a maze. At each junction, you try one path. If it leads to a dead end, you go back to the junction (backtrack) and try a different path. You keep doing this until you've explored every possible path or found what you're looking for.

**Subsets** = at each item, decide: take it or leave it.
**Permutations** = at each position, try every remaining item.
**Combinations** = at each position, try items from the remaining pool (in order, so we don't repeat).
**N-Queens** = at each row, try each column, but skip columns already used or on the same diagonal.

The "backtrack" step is crucial: after trying a choice and returning, you **undo** it completely so the next choice starts from the same state.

---

## SECTION 3 — FORMALLY DEFINED

**Backtracking** is a systematic method for generating all solutions to a combinatorial problem by incrementally building candidates and abandoning ("backtracking") candidates that cannot lead to valid solutions.

**The general recursion pattern:**

```
backtrack(state, remaining_choices):
    if is_complete(state):
        record(state)
        return
    
    for each choice in available_choices(remaining_choices):
        if is_valid(state, choice):         // pruning
            apply(state, choice)            // make choice
            backtrack(state, new_remaining) // recurse
            undo(state, choice)             // backtrack
```

**Time complexity:** O(N! × N) for permutations, O(2^N × N) for subsets/combinations in the worst case. Pruning reduces this empirically, but worst-case analysis still applies to unpruned backtracking.

**Space complexity:** O(N) call stack depth + O(N) for current path = O(N). Result storage is O(2^N × N) for subsets, O(N! × N) for permutations.

**Three key properties that make backtracking correct:**
1. **Complete:** Every valid solution is eventually found.
2. **Sound:** Every result recorded is actually valid.
3. **Termination:** The recursion terminates (choices always decrease or the depth is bounded).

---

## SECTION 4 — RECOGNITION SIGNALS

✅ **Strong signals:**
1. "Generate all subsets / combinations / permutations" — textbook backtracking
2. "Find all valid configurations" (N-queens, Sudoku) — constraint backtracking
3. "Find all possible ways to partition a string" — string backtracking
4. "Find all paths from source to destination" — path backtracking
5. N ≤ 20 with "enumerate" / "count all" / "list all" — backtracking or bitmask DP

⚠️ **Medium signals:**
6. "How many distinct ways..." with small N — might be DP or might be backtracking
7. Grid problem with "find path" or "reach all cells" — could be DFS/backtracking
8. "Can you partition into K groups?" — backtracking with pruning

❌ **FAKE signals — looks like backtracking but isn't:**
- "Count the number of ways to..." with large N → DP (backtracking would TLE)
- "Find any ONE valid assignment" with large N → greedy or constraint propagation
- "Minimum number of..." → greedy/DP, not backtracking

**The 10-second test:** Is N small (≤ 15-20)? Is the problem asking for ALL solutions? Can you draw a decision tree? If yes to all three → backtracking.

---

## SECTION 5 — THE UNIFIED BACKTRACKING FRAMEWORK

Before writing any backtracking solution, fill in these four slots:

| Slot | Question | Example (Subsets) | Example (Permutations) |
|------|----------|-------------------|------------------------|
| **State** | What are we building? | Current subset (temp vector) | Current arrangement (temp vector) |
| **Choices** | What can we add next? | Elements from index `start` onwards | Any unused element |
| **Goal** | When do we record? | At every node (any size is valid) | When `temp.size() == n` |
| **Constraint** | What do we prune? | None for basic; duplicates for Subsets II | None for basic; used[] array |

This table forces you to think before coding. Fill it out, then write the code.

---

## SECTION 6 — TEMPLATE 1: SUBSETS (POWER SET)

```cpp
// LC 78 — Subsets
// Given array of DISTINCT integers, return all possible subsets.
// Every subset size (0 to n) is valid → record at EVERY node (not just leaves)

// DECISION TREE for [1,2,3]:
//
//                    []
//           /         |        \
//         [1]        [2]       [3]
//        /   \         \
//      [1,2] [1,3]    [2,3]
//       /
//    [1,2,3]
//
// But we traverse it linearly using 'start' index:
// At depth 0 (start=0): can add 1, 2, or 3
// At depth 1 (start=1): can add 2 or 3 (if we chose 1 first)
// etc.

class Solution {
    vector<vector<int>> result;
    vector<int> temp;

    void backtrack(vector<int>& nums, int start) {
        result.push_back(temp);  // ← record at EVERY node (including empty set)

        for (int i = start; i < nums.size(); i++) {
            temp.push_back(nums[i]);       // CHOOSE: add nums[i]
            backtrack(nums, i + 1);        // EXPLORE: next elements only (i+1 prevents reuse)
            temp.pop_back();               // UNCHOOSE: remove nums[i]
        }
    }

public:
    vector<vector<int>> subsets(vector<int>& nums) {
        backtrack(nums, 0);
        return result;
    }
};
```

**TRACE through [1,2,3]:**
```
backtrack(start=0):
  record [] → result = [[]]
  i=0: temp=[1]
    backtrack(start=1):
      record [1] → result = [[], [1]]
      i=1: temp=[1,2]
        backtrack(start=2):
          record [1,2] → result = [[], [1], [1,2]]
          i=2: temp=[1,2,3]
            backtrack(start=3):
              record [1,2,3] → result = [[], [1], [1,2], [1,2,3]]
              (loop doesn't execute, start=3=nums.size())
            temp.pop_back() → temp=[1,2]
          (loop done)
        temp.pop_back() → temp=[1]
      i=2: temp=[1,3]
        backtrack(start=3):
          record [1,3] → result = [..., [1,3]]
          (loop done)
        temp.pop_back() → temp=[1]
    temp.pop_back() → temp=[]
  i=1: temp=[2]
    backtrack(start=2):
      record [2]
      i=2: temp=[2,3]
        backtrack(start=3): record [2,3]
      temp.pop_back() → temp=[2]
    temp.pop_back() → temp=[]
  i=2: temp=[3]
    backtrack(start=3): record [3]
    temp.pop_back() → temp=[]

Final result: [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
```

**WHY `start` parameter prevents duplicates:**
Without `start`, when processing from i=1 (element 2), we could go back and pick element 0 (element 1), generating [2,1] alongside [1,2]. The `start` parameter enforces that elements are always chosen in increasing index order, so each subset appears exactly once.

**WHY we record BEFORE the loop (at every node, not at leaves):**
The result includes subsets of ALL sizes (0 to n). Recording at leaves would only give us the full set. Recording before the loop means we capture: [] at root, [1] before exploring [1,2], [1,3], [1,2,3], etc.

**HOW TO ADAPT:**
- Change `result.push_back(temp)` to `if (temp.size() == k) result.push_back(temp)` → **Combinations of size K**
- Add `if (target reached) result.push_back(temp)` → **Combination Sum**

---

## SECTION 7 — TEMPLATE 2: SUBSETS II (WITH DUPLICATES)

The key challenge: given `[1,2,2]`, we should NOT produce `[2]` twice (from picking first 2, or second 2).

**The core insight:** Sort the array first. Then at each level of recursion, skip elements that are identical to the previous element at that same level. We can still pick the same VALUE in deeper recursion (representing different counts), but not the same value at the same recursion level.

```cpp
// LC 90 — Subsets II
// Contains duplicates — must not return duplicate subsets

class Solution {
    vector<vector<int>> result;
    vector<int> temp;

    void backtrack(vector<int>& nums, int start) {
        result.push_back(temp);

        for (int i = start; i < nums.size(); i++) {
            // SKIP: if nums[i] == nums[i-1] at the SAME recursion level
            // "Same level" = same start index
            // Condition: i > start (not the first element at this level)
            //            AND nums[i] == nums[i-1] (duplicate)
            if (i > start && nums[i] == nums[i-1]) continue;  // ← THE DUPLICATE SKIP

            temp.push_back(nums[i]);
            backtrack(nums, i + 1);
            temp.pop_back();
        }
    }

public:
    vector<vector<int>> subsetsWithDup(vector<int>& nums) {
        sort(nums.begin(), nums.end());  // ← MUST SORT FIRST
        backtrack(nums, 0);
        return result;
    }
};
```

**TRACE with [1,2,2] to understand the skip:**
```
sorted: [1,2,2]

backtrack(start=0):
  record []
  i=0: nums[0]=1. (i==start, no skip). temp=[1]
    backtrack(start=1):
      record [1]
      i=1: nums[1]=2. (i==start=1, no skip). temp=[1,2]
        backtrack(start=2):
          record [1,2]
          i=2: nums[2]=2. i=2 > start=2? NO (i==start). No skip. temp=[1,2,2]
            backtrack(start=3): record [1,2,2]
          temp.pop_back() → [1,2]
        temp.pop_back() → [1]
      i=2: nums[2]=2. i=2 > start=1? YES. nums[2]==nums[1]? 2==2? YES. → SKIP!
    temp.pop_back() → []
  i=1: nums[1]=2. (i=1 > start=0). nums[1]==nums[0]? 2==1? NO. No skip. temp=[2]
    backtrack(start=2):
      record [2]
      i=2: nums[2]=2. i=2 > start=2? NO. No skip. temp=[2,2]
        backtrack(start=3): record [2,2]
      temp.pop_back()
    temp.pop_back() → []
  i=2: nums[2]=2. i=2 > start=0? YES. nums[2]==nums[1]? 2==2? YES. → SKIP!

Result: [[], [1], [1,2], [1,2,2], [2], [2,2]]
Correct! No duplicate subsets.
```

**WHY `i > start` not just `i > 0`:**

Consider [1,2,2] at level 0 (start=0). When we reach i=2 (second 2), we skip because i>start AND nums[2]==nums[1]. This prevents picking "second 2" as the first element when we already generated subsets starting with "first 2".

But at level 1 (start=1, after choosing 1), when i=1 (first 2) is processed, i==start so no skip — we pick first 2. When i=2 (second 2), i=2 > start=1 AND nums[2]==nums[1] → skip. This prevents [1,2,2] from being generated twice.

**The invariant:** At any level, we pick each distinct value at most once as the "next" element. We can pick the same value again at a DEEPER level (different occurrence).

---

## SECTION 8 — TEMPLATE 3: PERMUTATIONS

```cpp
// LC 46 — Permutations (distinct integers)
// Generate all n! arrangements

class Solution {
    vector<vector<int>> result;
    vector<int> temp;
    vector<bool> used;

    void backtrack(vector<int>& nums) {
        if (temp.size() == nums.size()) {
            result.push_back(temp);  // ← record at LEAVES only (full arrangement)
            return;
        }

        for (int i = 0; i < nums.size(); i++) {
            if (used[i]) continue;  // skip already-placed elements

            used[i] = true;          // CHOOSE: place nums[i] at current position
            temp.push_back(nums[i]);
            backtrack(nums);          // EXPLORE: fill the next position
            temp.pop_back();          // UNCHOOSE
            used[i] = false;
        }
    }

public:
    vector<vector<int>> permute(vector<int>& nums) {
        used.resize(nums.size(), false);
        backtrack(nums);
        return result;
    }
};
```

**TRACE for [1,2,3] (abbreviated):**
```
backtrack(): temp=[], used=[F,F,F]
  i=0: used[0]=T, temp=[1]
    backtrack(): temp=[1], used=[T,F,F]
      i=0: skip (used)
      i=1: used[1]=T, temp=[1,2]
        backtrack(): temp=[1,2], used=[T,T,F]
          i=2: used[2]=T, temp=[1,2,3]
            backtrack(): temp=[1,2,3] == size 3 → record [1,2,3]
          used[2]=F, temp=[1,2]
        (only i=2 was valid)
      used[1]=F, temp=[1]
      i=2: used[2]=T, temp=[1,3]
        backtrack():
          i=1 available: temp=[1,3,2] → record [1,3,2]
        used[2]=F, temp=[1]
    used[0]=F, temp=[]
  i=1: used[1]=T, temp=[2]
    ... generates [2,1,3] and [2,3,1]
  i=2: used[2]=T, temp=[3]
    ... generates [3,1,2] and [3,2,1]

Total: 3! = 6 permutations
```

**KEY DIFFERENCE from subsets:**
- Subsets use `start` to prevent reusing indices at the same level
- Permutations use `used[]` to prevent reusing elements across positions
- Permutations record at LEAVES (when full arrangement is built), not at every node

**WHY the `used` array and not `start`?** In permutations, element order matters and elements from ANY index can be placed at the current position — not just those after `start`. The `used` array tracks which elements from the entire array are still available.

---

## SECTION 9 — TEMPLATE 4: PERMUTATIONS II (WITH DUPLICATES)

The classic approach of checking `used[i]` is insufficient for duplicates. A sorting trick is needed.

```cpp
// LC 47 — Permutations II (can contain duplicates)
// Key insight: if nums[i] == nums[i-1] AND nums[i-1] is NOT used (in the call stack),
// then nums[i] would generate the same subtree as nums[i-1] did → skip

class Solution {
    vector<vector<int>> result;
    vector<int> temp;
    vector<bool> used;

    void backtrack(vector<int>& nums) {
        if (temp.size() == nums.size()) {
            result.push_back(temp);
            return;
        }

        for (int i = 0; i < nums.size(); i++) {
            if (used[i]) continue;

            // DUPLICATE SKIP for permutations:
            // If nums[i] == nums[i-1] AND nums[i-1] is NOT in current path (used[i-1]=false),
            // then we're about to generate the same subtree as when we chose nums[i-1].
            // Skip to avoid duplicates.
            if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;

            used[i] = true;
            temp.push_back(nums[i]);
            backtrack(nums);
            temp.pop_back();
            used[i] = false;
        }
    }

public:
    vector<vector<int>> permuteUnique(vector<int>& nums) {
        sort(nums.begin(), nums.end());
        used.resize(nums.size(), false);
        backtrack(nums);
        return result;
    }
};
```

**WHY `!used[i-1]` and not `used[i-1]`?**

This is one of the trickiest backtracking conditions. Let's trace [1,2a,2b] (a,b distinguish the two 2s):

Case 1: We choose 2a first (used[1]=true), then 2b (i=2, used[2]=false, nums[2]==nums[1], used[1]=true). Condition `!used[i-1]` = `!true` = false → **don't skip** → allow [1,2a,2b].

Case 2: We skip 2a (used[1]=false), try 2b (i=2, used[2]=false, nums[2]==nums[1]=2a, used[1]=false). Condition `!used[i-1]` = `!false` = true → **skip** → this would produce [1,2b,...] which duplicates [1,2a,...].

**The rule in English:** "If the previous duplicate is currently NOT being used in our path, it means we already finished exploring all paths that START with that previous duplicate. Starting with the current duplicate would be redundant."

**Alternative interpretation (also valid but less common):** Always use `!used[i-1]` meaning "only pick nums[i] if nums[i-1] is currently on the path." This forces us to use the FIRST occurrence of each duplicate before the second.

---

## SECTION 10 — TEMPLATE 5: COMBINATION SUM (UNLIMITED REUSE)

```cpp
// LC 39 — Combination Sum
// Find all combinations that sum to target. EACH element can be reused.
// No duplicates in candidates.

class Solution {
    vector<vector<int>> result;
    vector<int> temp;

    void backtrack(vector<int>& candidates, int start, int target) {
        if (target == 0) {
            result.push_back(temp);  // ← exact match
            return;
        }
        if (target < 0) return;  // ← over-target, prune

        for (int i = start; i < candidates.size(); i++) {
            temp.push_back(candidates[i]);
            backtrack(candidates, i, target - candidates[i]);  // ← i, NOT i+1 (allow reuse!)
            temp.pop_back();
        }
    }

public:
    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
        sort(candidates.begin(), candidates.end());  // sort for pruning
        backtrack(candidates, 0, target);
        return result;
    }
};
```

**KEY DIFFERENCE:** `backtrack(candidates, i, ...)` uses `i` (not `i+1`) to allow the same element to be reused. The `start` parameter still prevents reordering (ensuring [2,2,3] appears but not [3,2,2]).

**IMPROVED PRUNING:** Since candidates are sorted, once `candidates[i] > target`, all subsequent candidates are also > target:

```cpp
for (int i = start; i < candidates.size(); i++) {
    if (candidates[i] > target) break;  // ← early termination since sorted
    temp.push_back(candidates[i]);
    backtrack(candidates, i, target - candidates[i]);
    temp.pop_back();
}
```

**TRACE for candidates=[2,3,6,7], target=7:**
```
backtrack(start=0, target=7):
  i=0: temp=[2], backtrack(start=0, target=5):
    i=0: temp=[2,2], backtrack(start=0, target=3):
      i=0: temp=[2,2,2], backtrack(start=0, target=1):
        i=0: candidates[0]=2 > 1 → break
      pop → [2,2]
      i=1: temp=[2,2,3], backtrack(start=1, target=0):
        target==0 → record [2,2,3]
      pop → [2,2]
      i=2: candidates[2]=6 > 3 → break
    pop → [2]
    i=1: temp=[2,3], backtrack(start=1, target=2):
      i=1: candidates[1]=3 > 2 → break
    pop → [2]
    i=2: candidates[2]=6 > 5 → break
  pop → []
  i=1: temp=[3], backtrack(start=1, target=4):
    i=1: temp=[3,3], backtrack(start=1, target=1):
      candidates[1]=3 > 1 → break
    pop → [3]
    i=2: candidates[2]=6 > 4 → break
  pop → []
  i=2: temp=[6], backtrack(start=2, target=1):
    candidates[2]=6 > 1 → break
  pop → []
  i=3: temp=[7], backtrack(start=3, target=0):
    target==0 → record [7]
  pop → []

Result: [[2,2,3], [7]]
```

---

## SECTION 11 — TEMPLATE 6: COMBINATIONS (CHOOSE K FROM N)

```cpp
// LC 77 — Combinations
// Generate all C(n,k) combinations of k numbers from [1..n]

class Solution {
    vector<vector<int>> result;
    vector<int> temp;

    void backtrack(int n, int k, int start) {
        if (temp.size() == k) {
            result.push_back(temp);
            return;
        }

        // PRUNING: remaining slots needed = k - temp.size()
        // Available elements = n - i + 1 (from i to n inclusive)
        // If available < remaining, no point trying from i onwards
        // So max useful start = n - (k - temp.size()) + 1
        for (int i = start; i <= n - (k - temp.size()) + 1; i++) {
            temp.push_back(i);
            backtrack(n, k, i + 1);
            temp.pop_back();
        }
    }

public:
    vector<vector<int>> combine(int n, int k) {
        backtrack(n, k, 1);
        return result;
    }
};
```

**THE PRUNING EXPLAINED:**
- We need `k - temp.size()` more elements.
- If we start the loop at position `i`, available elements are `i, i+1, ..., n` — that's `n - i + 1` elements.
- We need `n - i + 1 >= k - temp.size()` → `i <= n - (k - temp.size()) + 1`.
- Without this, the loop would try starting positions that can't possibly fill `k` elements.

**Example:** n=5, k=3. If temp=[1,2] (size=2), we need 1 more. Loop goes `i <= 5 - 1 + 1 = 5` → can start from any position up to 5. If temp=[1] (size=1), we need 2 more. Loop goes `i <= 5 - 2 + 1 = 4` → can't start from 5 because {5} alone can't give 2 elements.

---

## SECTION 12 — TEMPLATE 7: N-QUEENS (CONSTRAINT-BASED)

N-Queens is the canonical constraint backtracking problem. The constraint check is the core insight.

```cpp
// LC 51 — N-Queens
// Place N queens on N×N board so no two queens share row, column, or diagonal

class Solution {
    vector<vector<string>> result;
    vector<int> queenCol;  // queenCol[row] = column where queen is placed in that row
    vector<bool> colUsed, diag1Used, diag2Used;
    // diag1: top-left to bottom-right, key = row - col (constant on same diagonal)
    // diag2: top-right to bottom-left, key = row + col (constant on same diagonal)
    int n;

    void backtrack(int row) {
        if (row == n) {
            // All n queens placed — construct board
            vector<string> board(n, string(n, '.'));
            for (int r = 0; r < n; r++) board[r][queenCol[r]] = 'Q';
            result.push_back(board);
            return;
        }

        for (int col = 0; col < n; col++) {
            int d1 = row - col + n;  // offset by n to avoid negative indices
            int d2 = row + col;

            if (colUsed[col] || diag1Used[d1] || diag2Used[d2]) continue;  // CONSTRAINT CHECK

            // CHOOSE
            queenCol[row] = col;
            colUsed[col] = diag1Used[d1] = diag2Used[d2] = true;

            backtrack(row + 1);  // EXPLORE next row

            // UNCHOOSE
            colUsed[col] = diag1Used[d1] = diag2Used[d2] = false;
        }
    }

public:
    vector<vector<string>> solveNQueens(int n) {
        this->n = n;
        queenCol.resize(n);
        colUsed.resize(n, false);
        diag1Used.resize(2 * n, false);  // row-col ranges from -(n-1) to (n-1)
        diag2Used.resize(2 * n, false);  // row+col ranges from 0 to 2(n-1)
        backtrack(0);
        return result;
    }
};
```

**WHY rows are handled automatically:** We recurse one row at a time (`backtrack(row+1)`), so each recursion level corresponds to one row. No two queens can share a row — it's guaranteed by structure, not by a constraint check.

**WHY `row - col` is constant on a diagonal:**
On the main diagonal direction (↘), moving one step right (+col) and one step down (+row) keeps `row - col` constant. If two queens share a diagonal, `row1 - col1 == row2 - col2`.

**WHY `row + col` is constant on the other diagonal:**
On the anti-diagonal direction (↙), moving one step left (-col) and one step down (+row) keeps `row + col` constant.

**TRACE for n=4 (abbreviated):**
```
row=0: try col=0,1,2,3
  col=0: place queen at (0,0). colUsed[0]=T, diag1[4-0=4]=T, diag2[0+0=0]=T
    row=1: try col=0,1,2,3
      col=0: colUsed[0]=T → skip
      col=1: diag2[1+1=2]=F, diag1[1-1+4=4]=T → diag1 conflict → skip
      col=2: all free → place queen at (1,2)
        row=2: try col=0,1,2,3
          col=0: colUsed[0]=T → skip
          col=1: diag2[2+1=3]=F, diag1[2-1+4=5]=F, colUsed[1]=F → place (2,1)
            row=3: try col=0..3 — all conflict → backtrack
          col=1 fails → pop
          col=3: ... also fails → backtrack
        row=2 fails for this branch
      pop queen from (1,2)
      col=3: ... try queen at (1,3)
        row=2: col=1 works → queen at (2,1)
          row=3: col=2 works → queen at (3,2)
            row=4 == n → record! [".Q..", "...Q", "Q...", "..Q."]

... and so on for all valid configurations
```

**For n=4, there are exactly 2 solutions.**

---

## SECTION 13 — TEMPLATE 8: PALINDROME PARTITIONING

String backtracking: at each position, try all valid "cuts" and recurse on the rest.

```cpp
// LC 131 — Palindrome Partitioning
// Partition string into substrings where EACH substring is a palindrome

class Solution {
    vector<vector<string>> result;
    vector<string> temp;

    bool isPalin(const string& s, int l, int r) {
        while (l < r) if (s[l++] != s[r--]) return false;
        return true;
    }

    void backtrack(const string& s, int start) {
        if (start == s.size()) {
            result.push_back(temp);  // ← reached end: all parts are palindromes
            return;
        }

        for (int end = start; end < s.size(); end++) {
            if (isPalin(s, start, end)) {
                temp.push_back(s.substr(start, end - start + 1));  // CHOOSE
                backtrack(s, end + 1);                              // EXPLORE
                temp.pop_back();                                     // UNCHOOSE
            }
        }
    }

public:
    vector<vector<string>> partition(string s) {
        backtrack(s, 0);
        return result;
    }
};
```

**TRACE for "aab":**
```
backtrack(start=0):
  end=0: s[0..0]="a" isPalin? YES. temp=["a"]
    backtrack(start=1):
      end=1: s[1..1]="a" isPalin? YES. temp=["a","a"]
        backtrack(start=2):
          end=2: s[2..2]="b" isPalin? YES. temp=["a","a","b"]
            backtrack(start=3 == s.size()): record ["a","a","b"]
          pop → ["a","a"]
        (end loop done)
      pop → ["a"]
      end=2: s[1..2]="ab" isPalin? NO. Skip.
    pop → []
  end=1: s[0..1]="aa" isPalin? YES. temp=["aa"]
    backtrack(start=2):
      end=2: "b" isPalin? YES. temp=["aa","b"]
        backtrack(start=3): record ["aa","b"]
      pop → ["aa"]
    pop → []
  end=2: s[0..2]="aab" isPalin? NO. Skip.

Result: [["a","a","b"], ["aa","b"]]
```

**OPTIMIZATION — DP precompute palindromes:**

For large strings, calling `isPalin()` repeatedly is O(n) per call. Precompute a 2D palindrome table:

```cpp
// Before backtracking:
int n = s.size();
vector<vector<bool>> dp(n, vector<bool>(n, false));
for (int i = 0; i < n; i++) dp[i][i] = true;
for (int len = 2; len <= n; len++) {
    for (int i = 0; i <= n - len; i++) {
        int j = i + len - 1;
        dp[i][j] = (s[i] == s[j]) && (len == 2 || dp[i+1][j-1]);
    }
}
// Then: if (dp[start][end]) instead of isPalin(s, start, end)
```

This drops palindrome checking from O(n) to O(1) per call.

---

## SECTION 14 — TEMPLATE 9: WORD SEARCH (GRID BACKTRACKING)

```cpp
// LC 79 — Word Search
// Find if word exists in grid by traversing adjacent cells (no reuse)

class Solution {
    int m, n;
    string word;

    bool backtrack(vector<vector<char>>& board, int r, int c, int idx) {
        if (idx == word.size()) return true;  // ← all characters matched

        // Boundary and character check
        if (r < 0 || r >= m || c < 0 || c >= n) return false;
        if (board[r][c] != word[idx]) return false;

        // CHOOSE: mark visited by replacing cell (avoids extra visited array)
        char temp = board[r][c];
        board[r][c] = '#';  // ← in-place mark (O(1) space vs O(m*n) visited array)

        // EXPLORE all 4 directions
        bool found = backtrack(board, r+1, c, idx+1) ||
                     backtrack(board, r-1, c, idx+1) ||
                     backtrack(board, r, c+1, idx+1) ||
                     backtrack(board, r, c-1, idx+1);

        // UNCHOOSE: restore cell
        board[r][c] = temp;

        return found;
    }

public:
    bool exist(vector<vector<char>>& board, string word) {
        m = board.size(); n = board[0].size();
        this->word = word;

        for (int r = 0; r < m; r++) {
            for (int c = 0; c < n; c++) {
                if (backtrack(board, r, c, 0)) return true;
            }
        }
        return false;
    }
};
```

**THREE INSIGHTS specific to grid backtracking:**

1. **In-place marking:** `board[r][c] = '#'` and restore after recursion avoids a separate `visited` array. This is standard for grid backtracking (O(1) space for the "visited" concept).

2. **Short-circuit evaluation:** `backtrack(...) || backtrack(...)` — if the first direction finds the word, we don't explore the others. The `||` short-circuits.

3. **Boundary check before character check:** Check bounds first, then character. Reversed order causes out-of-bounds access.

**Complexity:** O(m × n × 4^L) where L = word length. For each starting cell (m×n), we explore up to 4 directions L times deep. In practice much faster due to pruning (most paths don't match the first character).

---

## SECTION 15 — TEMPLATE 10: SUDOKU SOLVER (HEAVY CONSTRAINT)

```cpp
// LC 37 — Sudoku Solver
// Fill 9x9 grid following Sudoku rules

class Solution {
    bool rowUsed[9][10] = {};   // rowUsed[r][d] = digit d used in row r
    bool colUsed[9][10] = {};   // colUsed[c][d] = digit d used in col c
    bool boxUsed[9][10] = {};   // boxUsed[b][d] = digit d used in box b
    // box index = (r/3)*3 + (c/3)

    bool solve(vector<vector<char>>& board) {
        // Find the next empty cell
        for (int r = 0; r < 9; r++) {
            for (int c = 0; c < 9; c++) {
                if (board[r][c] != '.') continue;

                int box = (r/3)*3 + (c/3);

                for (int d = 1; d <= 9; d++) {
                    if (rowUsed[r][d] || colUsed[c][d] || boxUsed[box][d]) continue;

                    // CHOOSE
                    board[r][c] = '0' + d;
                    rowUsed[r][d] = colUsed[c][d] = boxUsed[box][d] = true;

                    if (solve(board)) return true;  // ← propagate success

                    // UNCHOOSE
                    board[r][c] = '.';
                    rowUsed[r][d] = colUsed[c][d] = boxUsed[box][d] = false;
                }

                return false;  // ← no digit worked for this cell — backtrack!
            }
        }
        return true;  // ← no empty cell found — solved!
    }

public:
    void solveSudoku(vector<vector<char>>& board) {
        // Initialize constraint arrays from given clues
        for (int r = 0; r < 9; r++) {
            for (int c = 0; c < 9; c++) {
                if (board[r][c] == '.') continue;
                int d = board[r][c] - '0';
                int box = (r/3)*3 + (c/3);
                rowUsed[r][d] = colUsed[c][d] = boxUsed[box][d] = true;
            }
        }
        solve(board);
    }
};
```

**WHY `return false` inside the loop matters:**

When no digit d from 1..9 works for a cell, we must signal failure to the caller. Without `return false` at the end of the loop, the function would continue to the next cell and eventually return `true` even though the current state is invalid.

The flow is:
- Try all 9 digits for the current empty cell
- If none work → `return false` (tell caller this path is dead)
- If one works → recurse → if that recursion returns true, propagate `return true`
- If recursion returns false → undo and try next digit

---

## SECTION 16 — VARIANT: LETTER COMBINATIONS (PHONE NUMBER)

```cpp
// LC 17 — Letter Combinations of a Phone Number
// Multi-branch backtracking (not binary choices, but N-way choices)

class Solution {
    vector<string> result;
    string temp;
    vector<string> mapping = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};

    void backtrack(const string& digits, int idx) {
        if (idx == digits.size()) {
            if (!temp.empty()) result.push_back(temp);
            return;
        }

        string letters = mapping[digits[idx] - '0'];
        for (char c : letters) {
            temp += c;            // CHOOSE
            backtrack(digits, idx + 1);  // EXPLORE
            temp.pop_back();      // UNCHOOSE
        }
    }

public:
    vector<string> letterCombinations(string digits) {
        backtrack(digits, 0);
        return result;
    }
};
```

**This shows the framework scales to any branching factor.** Phone number has up to 4 branches (7→pqrs, 9→wxyz), but the structure is identical to all other backtracking.

---

## SECTION 17 — COMPLEXITY TABLE

| Problem | Time Complexity | Space (stack) | Result Size |
|---------|-----------------|---------------|-------------|
| Subsets | O(2^n × n) | O(n) | 2^n subsets |
| Subsets II (sorted, skip dup) | O(2^n × n) | O(n) | ≤ 2^n subsets |
| Permutations | O(n! × n) | O(n) | n! permutations |
| Permutations II | O(n! × n) | O(n) | ≤ n! permutations |
| Combinations C(n,k) | O(C(n,k) × k) | O(k) | C(n,k) combinations |
| Combination Sum | O(2^(target/min)) | O(target/min) | varies |
| Palindrome Partitioning | O(n × 2^n) | O(n) | varies |
| N-Queens | O(n!) | O(n) | varies |
| Sudoku | O(9^81) worst | O(81) | 1 solution |
| Word Search | O(mn × 4^L) | O(L) | bool |

**Important note on Combination Sum complexity:** If target=10 and min element=1, worst case is 2^10 = 1024 combinations, each of length up to 10 → O(10 × 2^10). With sorting and pruning, actual runtime is much lower.

**Why Sudoku worst case is O(9^81) but runs fast in practice:**
- 81 cells, each could take 9 digits = 9^81 ≈ 10^77 in theory
- But each constraint check eliminates massive subtrees immediately
- In practice, valid Sudoku puzzles are solved in microseconds

---

## SECTION 18 — COMMON MISTAKES

### Mistake 1: Forgetting to undo (missing backtrack step)

```cpp
// WRONG — no undo step, state carries over
void backtrack(vector<int>& nums, int start) {
    result.push_back(temp);
    for (int i = start; i < nums.size(); i++) {
        temp.push_back(nums[i]);
        backtrack(nums, i + 1);
        // BUG: no temp.pop_back() — temp grows forever, all subsets share state
    }
}

// CORRECT
void backtrack(vector<int>& nums, int start) {
    result.push_back(temp);
    for (int i = start; i < nums.size(); i++) {
        temp.push_back(nums[i]);
        backtrack(nums, i + 1);
        temp.pop_back();  // ← undo
    }
}
```

---

### Mistake 2: Wrong duplicate skip condition (Subsets II)

```cpp
// WRONG — using i > 0 instead of i > start
// This skips valid subsets like [2,2] from [1,2,2]
if (i > 0 && nums[i] == nums[i-1]) continue;  // BUG

// Trace: nums=[1,2,2], backtrack(start=1) (after choosing 1):
//   i=1: nums[1]=2. i=1 > 0=true. nums[1]==nums[0]? 2==1? NO → ok
//   i=2: nums[2]=2. i=2 > 0=true. nums[2]==nums[1]? 2==2? YES → SKIP!
//   But we should generate [1,2,2]! The second 2 should be allowed from start=1.

// CORRECT — using i > start
if (i > start && nums[i] == nums[i-1]) continue;
// i=2: i=2 > start=1=true. nums[2]==nums[1]? YES → SKIP only from i=2 at level start=1
// But at level start=2, i=2 == start=2, so i > start is false → NO SKIP → picks second 2
```

---

### Mistake 3: Combination Sum using `i+1` instead of `i` (no reuse)

```cpp
// WRONG — using i+1 means each element can only be used once
// This is for Combination Sum II (each used once), not Combination Sum I (unlimited reuse)
backtrack(candidates, i + 1, target - candidates[i]);  // BUG for LC 39

// Correct for LC 39 (unlimited reuse):
backtrack(candidates, i, target - candidates[i]);  // same i → can reuse

// Correct for LC 40 (each used once):
backtrack(candidates, i + 1, target - candidates[i]);  // i+1 → move forward
```

---

### Mistake 4: N-Queens wrong diagonal formula

```cpp
// WRONG — checking if queens are on same diagonal by row and column differences
// This check would need to be done against all previous queens — O(n) per check
for (int prev = 0; prev < row; prev++) {
    if (queenCol[prev] == col) return false;
    if (abs(queenCol[prev] - col) == abs(prev - row)) return false;
}
// This works but is O(n) per position. The hash set approach is O(1):

// CORRECT — hash sets for O(1) constraint check
// colUsed, diag1 (row-col), diag2 (row+col) arrays
// Update and undo these sets instead of scanning previous queens
```

---

### Mistake 5: Not sorting before applying duplicate skip

```cpp
// WRONG — applying duplicate skip on unsorted array
// [2,1,2] unsorted: when i=2 (value=2), nums[i-1]=1 ≠ 2, so no skip
// But [2] appears twice! The skip only works when equal values are adjacent.

// CORRECT — always sort before any duplicate backtracking
sort(nums.begin(), nums.end());  // ensures equal values are adjacent
// Then the i > start && nums[i] == nums[i-1] check works correctly
```

---

### Mistake 6: Recording result as reference (passes address, not copy)

```cpp
// WRONG — pushing reference or storing temp directly
result.push_back(temp);  // This is actually fine in C++ — copies temp by value ✓

// WRONG in Python-esque pseudocode thinking:
// result.append(temp)  ← in Python, this appends a reference
// result.append(list(temp)) ← correct in Python — explicit copy

// In C++, vector::push_back(vector<int>) makes a COPY by value.
// So result.push_back(temp) is CORRECT in C++.
// The bug would be: result.push_back(move(temp)) — this moves temp into result,
// leaving temp empty for subsequent use. Never use move() on temp in backtracking.
```

---

## SECTION 19 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Subsets | 78 | Basic backtracking — record at every node |
| 2 | Permutations | 46 | used[] array, record at leaves |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Combinations | 77 | start parameter, pruning upper bound |
| 4 | Combination Sum | 39 | Same index `i` for reuse |
| 5 | Subsets II | 90 | Sort + `i > start` duplicate skip |
| 6 | Permutations II | 47 | Sort + `!used[i-1]` duplicate skip |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Palindrome Partitioning | 131 | String backtracking, palindrome check |
| 8 | Word Search | 79 | Grid backtracking, in-place marking |
| 9 | N-Queens | 51 | Row-by-row, diagonal hash sets, O(1) constraint |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Sudoku Solver | 37 | Heavy constraint check, propagate true/false |
| 11 | Letter Combinations of Phone Number | 17 | Multi-branch, mapping table |

---

**Solve in this order.** LC 78 and LC 46 are mandatory — they establish the two fundamental templates. LC 90 and LC 47 are THE critical problems for duplicate handling. N-Queens is the canonical constraint problem for interviews.

---

## SECTION 20 — PATTERN CONNECTIONS

1. **Pattern 12 (BFS/DFS):** Grid backtracking (Word Search) is DFS with backtracking — the only difference is we *undo* changes when returning. DFS on a graph typically doesn't undo (it modifies a `visited` array permanently). Backtracking undoes to explore alternative paths.

2. **Pattern 14 (Shortest Path → BFS with Bitmask State):** Shortest Path Visiting All Nodes (LC 847) is BFS with bitmask state — effectively a backtracking structure but using BFS instead of DFS because we want SHORTEST path. The decision space (which nodes to visit) is the same, but BFS gives optimal.

3. **Pattern 24 (Bitmask DP):** When backtracking is too slow (N up to 20), bitmask DP is the optimization. Partition to K Equal Sum Subsets can be solved with backtracking (with pruning) OR bitmask DP. Recognizing this transition is a sign of maturity.

4. **Pattern 23 (Interval DP):** Palindrome Partitioning II (min cuts) is Pattern 15 (backtracking for all partitions) PLUS DP optimization. Backtracking gives all partitions in O(n × 2^n); DP gives the minimum in O(n²).

5. **Pattern 26 (Trie):** Word Search II (LC 212) combines Pattern 15 (grid backtracking) with Pattern 26 (Trie) — build a Trie of all words, then backtrack on the grid while pruning Trie paths. The Trie enables pruning: if no word starts with the current prefix, stop immediately.

**The backtracking mindset in harder problems:**

Many problems labeled "hard" are just backtracking with a non-obvious state space. When you see "generate all X" or "find all valid Y" and the state space is structured (grid, string, partial arrangement), backtracking is the right mental model even if the final implementation uses DP for counting.

---

## SECTION 21 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 78 → Subsets

**Interviewer:** "Given a set of distinct integers, return all possible subsets."

**Candidate:**

*[First 1 minute — note key property]*
> "Distinct integers, all subsets including empty set. This is 2^n subsets total. I'll use backtracking — at each index, I choose to include or skip the element."

*[State the two templates explicitly]*
> "I know two equivalent formulations: (1) binary choice — for each element, include or skip (generates tree of depth n); (2) start-index — loop from current index forward, pick any element as 'next'. I'll use start-index as it maps more naturally to combinations and avoids the leaf-only recording."

*[Code]*
```cpp
// [Writes Template 1 from Section 6]
```

*[Analysis]*
> "Time O(2^n × n) — 2^n subsets, each takes O(n) to copy. Space O(n) call stack."

*[Follow-up]: "What if the array has duplicates?"*
> "Sort the array, then add a skip condition: `if (i > start && nums[i] == nums[i-1]) continue`. The key is `i > start` not `i > 0` — at the first position of each level (i == start), we always proceed. At subsequent positions (i > start), we skip if equal to previous to avoid duplicates."

**RED FLAGS:**
- Not knowing why `start` prevents duplicate subsets
- Writing the subset as a leaf-only recording (missing smaller subsets)
- Forgetting `temp.pop_back()` — the single most common backtracking bug

---

### Interview Simulation 2: LC 51 → N-Queens

**Interviewer:** "Place N queens on an N×N chessboard so no two queens attack each other."

**Candidate:**

*[First 2 minutes — design the constraint check]*
> "Two queens conflict if they share a row, column, or diagonal. I'll place exactly one queen per row, so rows are handled by structure. I need to track used columns and both diagonal directions. The key insight: all cells on the same ↘ diagonal have the same `row - col` value, and all cells on the same ↙ diagonal have the same `row + col` value."

*[State the data structures explicitly]*
> "I'll use three boolean arrays: `colUsed[col]`, `diag1Used[row-col+n]` (offset for indexing), `diag2Used[row+col]`. This makes the constraint check O(1) instead of O(n)."

*[Code — 8 minutes for this problem]*
```cpp
// [Writes Template 7 from Section 12]
```

*[Complexity]*
> "Time O(n!) — we place queens row by row, and the number of valid placements decreases. Space O(n) for the constraint arrays and recursion stack."

*[Follow-up]: "What if you only need to return the COUNT of solutions, not the boards?"*
> "Remove the board construction: instead of creating and pushing the board, just increment a counter. Time and space are the same, but the constant is smaller."

**RED FLAGS:**
- Using `abs(queenCol[prev] - col) == abs(prev - row)` with an inner loop (O(n) constraint check instead of O(1))
- Getting diagonal indices wrong (forgetting to offset `row-col` which can be negative)
- Forgetting to undo `colUsed`, `diag1Used`, `diag2Used` after recursion

---

### Interview Simulation 3: LC 39 → Combination Sum (with follow-up to LC 40)

**Interviewer:** "Find all combinations of numbers that sum to target. Can reuse same number."

**Candidate:**

*[Immediately spot the "reuse" key word]*
> "The critical word is 'reuse.' In standard combination backtracking, I advance to `i+1` to avoid reusing. For this problem, I stay at `i` to allow reuse."

*[Code with pruning]*
> "I'll sort the candidates first for two reasons: (1) pruning — once candidates[i] > remaining target, break; (2) it doesn't affect correctness and makes the code cleaner."

```cpp
void backtrack(vector<int>& c, int start, int target) {
    if (target == 0) { result.push_back(temp); return; }
    for (int i = start; i < c.size() && c[i] <= target; i++) {
        temp.push_back(c[i]);
        backtrack(c, i, target - c[i]);  // ← i, not i+1
        temp.pop_back();
    }
}
```

*[Follow-up]: "Now each element can only be used once (LC 40). What changes?"*
> "Two changes: (1) use `i+1` instead of `i` in the recursive call. (2) Skip duplicates: `if (i > start && candidates[i] == candidates[i-1]) continue` — same pattern as Subsets II."

*[Why the duplicate skip is different for Subsets II vs Combination Sum II]*
> "It's actually the same condition — `i > start && same as previous`. The sort ensures duplicates are adjacent, and `i > start` ensures we don't skip the FIRST occurrence of each value at each recursion level."

**RED FLAGS:**
- Confusing LC 39 (reuse, use `i`) with LC 40 (no reuse, use `i+1`)
- Not sorting candidates (misses the pruning opportunity and also doesn't help with LC 40's duplicate skip)
- Getting the duplicate skip condition wrong for LC 40 (using `i > 0` instead of `i > start`)

---

## SECTION 22 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: In the unified backtracking template, what is the difference between "recording at every node" vs "recording only at leaves"? Which problems use each?**

> Recording at every node captures all partial states as valid results — used when any size of the constructed object is valid (Subsets: any subset is a valid answer).
>
> Recording only at leaves captures complete constructions — used when only the full-length object is valid (Permutations: only complete arrangements of all n elements; N-Queens: only full n-queen placements).
>
> Palindrome Partitioning records at leaves (when `start == s.size()`, meaning the whole string has been partitioned). Combination Sum records when `target == 0` (the exact sum is reached) — also a "leaf" condition.

---

**Q2: In Subsets II, why is `i > start && nums[i] == nums[i-1]` correct, while `i > 0 && nums[i] == nums[i-1]` is wrong?**

> `i > 0` would skip duplicates globally — at any level, skip if the previous element is the same. This is too aggressive: it would prevent picking the second `2` in `[1,2,2]` at a nested level (after already picking the first `2`).
>
> `i > start` only skips within the current recursion level (the current "slot" we're filling). At the same level, we don't want to try the same value twice (that would give duplicate subsets). But at a deeper level (after picking the first `2`), we CAN pick the second `2`.
>
> The `start` parameter represents "the beginning of our choices at this recursion level." `i > start` means "we've already tried a previous element at this level."

---

**Q3: In Permutations II, the condition is `!used[i-1]`, not `used[i-1]`. Why?**

> The condition means: "skip nums[i] if nums[i-1] is an equal-value element that is currently NOT being used."
>
> If `!used[i-1]` is true (previous equal element is free), it means we already finished exploring all permutations where nums[i-1] is placed first in this "slot." Placing nums[i] (same value) would produce identical results.
>
> If `used[i-1]` is true (previous equal element is currently on the path), then nums[i] is a genuinely different occurrence and we need to include it to form permutations like [2,2,...].
>
> Alternative mnemonic: "Always use the left-most occurrence of each duplicate first before using right-ward occurrences."

---

**Q4: Why is the in-place marking (`board[r][c] = '#'`) in Word Search better than a separate `visited[][]` array?**

> Both are correct. The in-place marking:
> - Saves O(m×n) space (no extra array)
> - Is slightly faster (one memory access vs two)
> - Is a common interview pattern worth knowing
>
> The separate `visited[][]` array:
> - Keeps the board unmodified (safer, no risk of forgetting to restore)
> - Clearer intent to readers
>
> The in-place approach requires correctly restoring `board[r][c] = temp` after recursion — forgetting this causes the character to remain '#' and future paths can't use that cell.

---

**Q5: What is the time complexity of Combination Sum and how does sorting + pruning help?**

> Without pruning: O(2^(target/min_val) × avg_combo_length) — exponential in target/min.
>
> With sorting + `if (candidates[i] > target) break`:
> - We prune the moment we exceed the target
> - In practice, this reduces the search space dramatically
> - The pruning is TIGHT for sorted arrays: once the first element exceeds target, all subsequent elements also exceed target
>
> The formal worst case is still exponential, but with tight pruning, problems with target ≤ 40 and candidates 1-9 run in milliseconds.

---

**Q6: Why does Sudoku Solver need `return false` INSIDE the digit loop, not just after?**

> The `return false` inside the `for (int d = 1; d <= 9; d++)` loop is actually at the END of the `if (board[r][c] == '.')` block, OUTSIDE the digit loop. Let me clarify:
>
> The structure is: "find empty cell, try all digits; if none work → return false."
> The `return false` after the digit loop means "this empty cell has no valid digit; current state is impossible; backtrack."
>
> If we didn't return false here, the function would continue to the NEXT empty cell. But the current cell is still `.` (empty), so the board is incomplete. Eventually we'd reach the end of the nested loops and return `true` incorrectly.
>
> The `return false` immediately after the digit loop is the core backtracking signal: "I tried all options, none worked, tell my caller to undo its last decision."

---

**Q7: When should you add a palindrome DP precomputation to Palindrome Partitioning?**

> The `isPalin()` function is O(n) per call. In `backtrack(start)`, the loop runs up to O(n) times, and each call can have O(n) depth — giving O(n²) palindrome checks total, each O(n) = O(n³) total just for checking.
>
> With DP precomputation, all `dp[i][j]` values are computed in O(n²) time and O(n²) space. Each check in backtracking becomes O(1). For strings of length n ≤ 16 (typical for this problem), the difference is negligible. For n ≤ 100-1000, precomputation is essential.
>
> The precomputation pattern: `dp[i][j] = (s[i] == s[j]) && (j - i <= 1 || dp[i+1][j-1])`. Fill by increasing length.

---

## SECTION 23 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// SUBSETS — record at every node
// ─────────────────────────────────────────────────────────────────
void backtrack(vector<int>& nums, int start) {
    result.push_back(temp);                  // record at every node
    for (int i = start; i < nums.size(); i++) {
        temp.push_back(nums[i]);
        backtrack(nums, i + 1);
        temp.pop_back();
    }
}

// ─────────────────────────────────────────────────────────────────
// SUBSETS II (duplicates) — sort first, then skip
// ─────────────────────────────────────────────────────────────────
sort(nums.begin(), nums.end());
// In loop: if (i > start && nums[i] == nums[i-1]) continue;

// ─────────────────────────────────────────────────────────────────
// PERMUTATIONS — used[] array, record at leaves
// ─────────────────────────────────────────────────────────────────
void backtrack(vector<int>& nums) {
    if (temp.size() == nums.size()) { result.push_back(temp); return; }
    for (int i = 0; i < nums.size(); i++) {
        if (used[i]) continue;
        used[i] = true; temp.push_back(nums[i]);
        backtrack(nums);
        temp.pop_back(); used[i] = false;
    }
}

// ─────────────────────────────────────────────────────────────────
// PERMUTATIONS II (duplicates) — sort + !used[i-1]
// ─────────────────────────────────────────────────────────────────
sort(nums.begin(), nums.end());
// In loop: if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;

// ─────────────────────────────────────────────────────────────────
// COMBINATION SUM (reuse) — same i
// ─────────────────────────────────────────────────────────────────
void backtrack(vector<int>& c, int start, int target) {
    if (target == 0) { result.push_back(temp); return; }
    for (int i = start; i < c.size() && c[i] <= target; i++) {
        temp.push_back(c[i]);
        backtrack(c, i, target - c[i]);    // ← i (reuse allowed)
        temp.pop_back();
    }
}

// ─────────────────────────────────────────────────────────────────
// COMBINATION SUM II (no reuse, duplicates) — i+1 + skip
// ─────────────────────────────────────────────────────────────────
// sort + i+1 + if (i > start && c[i] == c[i-1]) continue

// ─────────────────────────────────────────────────────────────────
// N-QUEENS — diagonal hash sets
// ─────────────────────────────────────────────────────────────────
// colUsed[col], diag1Used[row-col+n], diag2Used[row+col]
// Check all three before placing; undo all three after

// ─────────────────────────────────────────────────────────────────
// GRID BACKTRACKING — in-place marking
// ─────────────────────────────────────────────────────────────────
char save = board[r][c];
board[r][c] = '#';          // mark
// ... recurse ...
board[r][c] = save;          // unmark

// ─────────────────────────────────────────────────────────────────
// DECISION TABLE — 4 questions before writing any backtracking
// ─────────────────────────────────────────────────────────────────
// 1. STATE: What am I building? (temp vector, partial board, etc.)
// 2. CHOICES: What can I add? (elements from start, grid neighbors, digits 1-9)
// 3. GOAL: When do I record? (every node vs leaves)
// 4. CONSTRAINT: What do I prune? (duplicates, target exceeded, queens attacking)
```

---

## SECTION 24 — WHAT TO DO AFTER THIS PATTERN

1. **Immediately solve (in order):** LC 78 → 46 → 77 → 39 → 90 → 47 → 131 → 79 → 51 → 37 → 17

2. **Master the unified view:** After solving all 11 problems, go back and write them ALL using the same four-slot framework (State, Choices, Goal, Constraint). Verify that every solution fills these four slots identically.

3. **Connect to Bitmask DP (Pattern 24 preview):** When you finish backtracking, solve LC 698 (Partition to K Equal Sum Subsets) with both backtracking AND bitmask DP. The backtracking solution naturally reveals what the DP state should be.

4. **The "pruning practice":** Take any working backtracking solution and identify: where would adding a pruning condition reduce the search space? For N-Queens, what happens if you add "if remaining columns are fewer than remaining rows, return false"?

5. **Contest backtracking:** In Codeforces, problems asking to "enumerate all valid colorings of a graph with N ≤ 15" or "find all valid assignments of items to groups" often require backtracking with pruning. The key is identifying WHAT to prune.

---

## SECTION 25 — SIGN-OFF CRITERIA

### Tier 1 — Can implement
You can write the subset and permutation backtracking templates from scratch in under 4 minutes each, with the correct `pop_back()` undo step and correct `start` / `used[]` mechanisms.

### Tier 2 — Can apply
You solve LC 78, 46, 77, 39 cleanly. You know when to record at every node vs only at leaves. You handle the `start` parameter correctly for all combination-style problems.

### Tier 3 — Can handle duplicates
You solve LC 90 and LC 47 correctly on the first attempt. You can explain the difference between `i > start` (Subsets II) and `!used[i-1]` (Permutations II) without looking at notes. You know both conditions require sorting first.

### Tier 4 — Can apply constraints and grid backtracking
You solve N-Queens using O(1) diagonal hash sets (not O(n) scan). You solve Word Search with in-place marking. You write Sudoku Solver with correct `return false` propagation. You can apply the four-slot decision framework to any new backtracking problem.

**Sign-off threshold:** Solve 8 of 11 problems, including LC 78 (basic subsets), LC 90 (duplicates), LC 46 (permutations), and LC 51 (N-Queens). Duplicate handling and N-Queens must work on first attempt — these are the two hardest points to get right.

---

*Pattern 15 Complete — Backtracking*
*Next: Pattern 16 — DP Fundamentals and the DP Mindset*
*You've completed all of Phase 2 (Core Data Structures). The graph section (P12-P14) + backtracking (P15) are now documented. Phase 3 begins with DP — the hardest and most important section.*
