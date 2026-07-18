# PATTERN 20 — LCS FAMILY (LONGEST COMMON SUBSEQUENCE)
## Two-Sequence DP, Edit Distance, Palindrome Subsequences, Wildcard/Regex Matching — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of two-sequence DP:**

Every problem in this pattern shares one skeleton: `dp[i][j]` = the answer using the first `i` characters of string 1 and the first `j` characters of string 2. Once you see this, LCS, Edit Distance, Distinct Subsequences, and even Wildcard Matching all become the SAME problem with a different transition function.

The skeleton:
```
dp[i][j] = f(dp[i-1][j-1], dp[i-1][j], dp[i][j-1], match_or_mismatch_at(i,j))
```

**What separates 1600 from 2000 here is recognizing four disguises:**

1. **LCS in disguise:** Longest Palindromic Subsequence (LC 516) is NOT a new algorithm — it is `LCS(s, reverse(s))`. Shortest Common Supersequence (LC 1092) is NOT a new algorithm — it is `m + n - LCS(s1, s2)` plus a reconstruction walk. Minimum Insertions to Make a Palindrome (LC 1312) is `n - LPS(s)`. If you don't see the reduction, you will reinvent a much harder DP from scratch under interview pressure.

2. **Edit Distance is LCS with costs.** Both are `dp[i][j]` over two strings. LCS asks "how many characters can I keep without touching them"; Edit Distance asks "what is the minimum cost to transform one into the other." The transitions look almost identical — the mismatch case in LCS takes `max`, in Edit Distance it takes `1 + min` (insert/delete/replace).

3. **Wildcard vs Regex `*` are OPPOSITE semantics wearing the same character.** In Wildcard Matching (LC 44), `*` matches ANY sequence of characters (including empty) by itself — it is a standalone wildcard. In Regular Expression Matching (LC 10), `*` means "zero or more of the PRECEDING character" — it is a modifier, not a standalone token. Confusing these two is the single most common mistake at this pattern.

4. **Substring vs subsequence is not a technicality — it changes the entire recurrence.** A subsequence DP (LCS) takes `max(dp[i-1][j], dp[i][j-1])` on mismatch because skipping characters is free. A substring DP (Longest Palindromic Substring, Longest Common Substring) RESETS to 0 on mismatch, because a substring must be contiguous — you cannot skip and reconnect later.

**The key skill:**

Before coding, answer:
1. Is this counting/matching a **subsequence** (can skip characters) or a **substring** (must be contiguous)?
2. Does a mismatch mean "take the best of two options" (max/min over neighbors) or "pay a cost and take the best of three options" (insert/delete/replace)?
3. Can I reduce this problem to plain LCS or LPS that I already know how to solve?

If you answer these three questions correctly, every problem in this pattern collapses to a template you have already seen.

---

## SECTION 2 — TAXONOMY OF TWO-SEQUENCE DP

### Type 1: Match/Mismatch Diagonal DP (the LCS family core)
State: `dp[i][j]` = best value using first `i` chars of s1, first `j` chars of s2
Transition: match → `dp[i-1][j-1] + 1` (diagonal); mismatch → `max(dp[i-1][j], dp[i][j-1])`
Problems: LCS, Longest Palindromic Subsequence (LCS with s vs reverse(s)), Uncrossed Lines, Longest Common Substring (variant: resets on mismatch instead of max)

### Type 2: Cost-Based Transform DP
State: `dp[i][j]` = minimum cost to transform/reconcile first `i` chars of s1 with first `j` chars of s2
Transition: match → `dp[i-1][j-1]` (free); mismatch → `cost + min(three neighbor operations)`
Problems: Edit Distance, Minimum ASCII Delete Sum, Delete Operation for Two Strings

### Type 3: Counting / Path-Counting DP
State: `dp[i][j]` = number of distinct ways to form target using source
Transition: match → `dp[i-1][j-1] + dp[i-1][j]` (use the match OR skip it); mismatch → `dp[i-1][j]` (must skip)
Problems: Distinct Subsequences

### Type 4: Pattern-Matching DP with Special Tokens
State: `dp[i][j]` = does `s[0..i-1]` match pattern `p[0..j-1]`?
Transition: depends entirely on the semantics of the special token (`*`, `?`, `.`)
Problems: Wildcard Matching, Regular Expression Matching

### Type 5: Interval/Substring DP for Palindromes
State: `dp[i][j]` = is `s[i..j]` a palindrome? (single-string, interval-indexed, NOT two-sequence — included here because it shares the palindrome theme with LPS)
Transition: `dp[i][j] = (s[i]==s[j]) && dp[i+1][j-1]`
Problems: Longest Palindromic Substring

### Type 6: Derived DP (reduction to a template you already solved)
State: reuse LCS or LPS table; the "new" problem is a formula or reconstruction on top of it
Problems: Shortest Common Supersequence (`m+n-LCS`), Min Insertions to Make Palindrome (`n-LPS`)

---

## SECTION 3 — TEMPLATE 1: LONGEST COMMON SUBSEQUENCE (THE FOUNDATION)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1143 — Longest Common Subsequence
// Given two strings, find the length of their longest common subsequence
// (characters need not be contiguous, but relative order must be preserved).
//
// State:      dp[i][j] = LCS length using s1[0..i-1] and s2[0..j-1]
// Transition: if s1[i-1] == s2[j-1]: dp[i][j] = dp[i-1][j-1] + 1   (diagonal — extend the match)
//             else:                  dp[i][j] = max(dp[i-1][j], dp[i][j-1])  (skip one char, take best)
// Base cases: dp[0][j] = 0 (empty s1 → LCS with anything is 0)
//             dp[i][0] = 0 (empty s2 → LCS with anything is 0)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

int longestCommonSubsequence(string s1, string s2) {
    int m = s1.size(), n = s2.size();
    // dp is (m+1) x (n+1) — row/col 0 represent "empty prefix"
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1[i-1] == s2[j-1]) {
                dp[i][j] = dp[i-1][j-1] + 1;      // characters match: extend diagonal
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1]);  // mismatch: best of dropping either char
            }
        }
    }

    return dp[m][n];
}
```

**FULL TRACE for s1="abcde", s2="ace":**
```
m=5, n=3. dp is 6x4, row0 and col0 all zero.

        ""  a  c  e
  ""     0  0  0  0
  a      0  1  1  1
  b      0  1  1  1
  c      0  1  2  2
  d      0  1  2  2
  e      0  1  2  3

i=1 (s1='a'): j=1(a): match  → dp[1][1]=dp[0][0]+1=1
              j=2(c): a≠c    → dp[1][2]=max(dp[0][2]=0, dp[1][1]=1)=1
              j=3(e): a≠e    → dp[1][3]=max(dp[0][3]=0, dp[1][2]=1)=1

i=2 (s1='b'): all mismatches → row copies forward the max seen so far: [1,1,1]

i=3 (s1='c'): j=2(c): match  → dp[3][2]=dp[2][1]+1=1+1=2
              j=3(e): c≠e    → dp[3][3]=max(dp[2][3]=1, dp[3][2]=2)=2

i=4 (s1='d'): all mismatches → carries forward [1,2,2]

i=5 (s1='e'): j=3(e): match  → dp[5][3]=dp[4][2]+1=2+1=3

Answer: dp[5][3] = 3   (LCS = "ace") ✓
```

**WHY `max`, not `min` or `+`:** On mismatch, we are choosing between two SUB-PROBLEMS — "drop the current char of s1" or "drop the current char of s2." Both are valid ways to make progress; we want the one that keeps the most matches downstream, hence `max`.

---

## SECTION 4 — TEMPLATE 2: LCS RECONSTRUCTION (BACKTRACK THE TABLE)

Computing the LENGTH is only half the skill. Reconstructing the ACTUAL subsequence is the follow-up every interviewer asks.

```cpp
// Reconstruct the actual LCS string, not just its length.
// Strategy: build the same dp table, then walk backward from dp[m][n] to dp[0][0].
//   - If s1[i-1] == s2[j-1]: this character IS part of the LCS. Record it, move diagonally (i-1, j-1).
//   - Else: move toward whichever neighbor (dp[i-1][j] or dp[i][j-1]) has the larger value
//           (that's the direction the optimal solution came from).

string reconstructLCS(string s1, string s2) {
    int m = s1.size(), n = s2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = (s1[i-1] == s2[j-1]) ? dp[i-1][j-1] + 1
                                             : max(dp[i-1][j], dp[i][j-1]);

    // Backtrack from bottom-right corner
    string result;
    int i = m, j = n;
    while (i > 0 && j > 0) {
        if (s1[i-1] == s2[j-1]) {
            result += s1[i-1];   // this char is part of the LCS
            i--; j--;            // move diagonally
        } else if (dp[i-1][j] >= dp[i][j-1]) {
            i--;                 // the optimal path came from above
        } else {
            j--;                 // the optimal path came from the left
        }
    }

    reverse(result.begin(), result.end());  // we built it backward
    return result;
}
```

**TRACE for s1="abcde", s2="ace" (using the dp table from Section 3):**
```
Start at (i=5, j=3).
  s1[4]='e', s2[2]='e' → MATCH. Append 'e'. Move to (4,2).
  s1[3]='d', s2[1]='c' → mismatch. dp[3][2]=2 vs dp[4][1]=1 → move up. (3,2).
  s1[2]='c', s2[1]='c' → MATCH. Append 'c'. Move to (2,1).
  s1[1]='b', s2[0]='a' → mismatch. dp[1][1]=1 vs dp[2][0]=0 → move up. (1,1).
  s1[0]='a', s2[0]='a' → MATCH. Append 'a'. Move to (0,0).
  i==0 → stop.

Collected (append order): 'e','c','a' → reverse → "ace" ✓
```

**WHY the tie-break `dp[i-1][j] >= dp[i][j-1]` doesn't matter for correctness:** When the two neighbor values are equal, either direction yields a valid LCS of the same length — just a different (equally correct) one, since multiple LCS's can exist.

---

## SECTION 5 — TEMPLATE 3: EDIT DISTANCE (LCS WITH COSTS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 72 — Edit Distance
// Minimum number of operations (insert, delete, replace) to convert
// s1 into s2.
//
// State:      dp[i][j] = min operations to convert s1[0..i-1] into s2[0..j-1]
// Transition: if s1[i-1] == s2[j-1]: dp[i][j] = dp[i-1][j-1]              (free — chars already match)
//             else:                  dp[i][j] = 1 + min(
//                                         dp[i-1][j-1],   // REPLACE s1[i-1] with s2[j-1]
//                                         dp[i-1][j],     // DELETE s1[i-1]
//                                         dp[i][j-1])     // INSERT s2[j-1] into s1
// Base cases: dp[i][0] = i  (delete all i chars of s1 to reach empty s2)
//             dp[0][j] = j  (insert all j chars of s2 into empty s1)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

int minDistance(string s1, string s2) {
    int m = s1.size(), n = s2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (int i = 0; i <= m; i++) dp[i][0] = i;   // delete everything
    for (int j = 0; j <= n; j++) dp[0][j] = j;   // insert everything

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1[i-1] == s2[j-1]) {
                dp[i][j] = dp[i-1][j-1];   // no operation needed
            } else {
                dp[i][j] = 1 + min({dp[i-1][j-1],  // replace
                                     dp[i-1][j],    // delete
                                     dp[i][j-1]});  // insert
            }
        }
    }

    return dp[m][n];
}
```

**FULL TRACE for s1="horse", s2="ros" (the canonical LC72 example, answer=3):**
```
        ""  r  o  s
  ""     0  1  2  3
  h      1  1  2  3
  o      2  2  1  2
  r      3  2  2  2
  s      4  3  3  2
  e      5  4  4  3

i=1(h): j=1(r): h≠r → 1+min(dp[0][0]=0,dp[0][1]=1,dp[1][0]=1)=1
        j=2(o): h≠o → 1+min(dp[0][1]=1,dp[0][2]=2,dp[1][1]=1)=2
        j=3(s): h≠s → 1+min(dp[0][2]=2,dp[0][3]=3,dp[1][2]=2)=3

i=2(o): j=1(r): o≠r → 1+min(dp[1][0]=1,dp[1][1]=1,dp[2][0]=2)=2
        j=2(o): o==o → dp[2][2]=dp[1][1]=1
        j=3(s): o≠s → 1+min(dp[1][2]=2,dp[1][3]=3,dp[2][2]=1)=2

i=3(r): j=1(r): r==r → dp[3][1]=dp[2][0]=2
        j=2(o): r≠o → 1+min(dp[2][1]=2,dp[2][2]=1,dp[3][1]=2)=2
        j=3(s): r≠s → 1+min(dp[2][2]=1,dp[2][3]=2,dp[3][2]=2)=2

i=4(s): j=1(r): s≠r → 1+min(dp[3][0]=3,dp[3][1]=2,dp[4][0]=4)=3
        j=2(o): s≠o → 1+min(dp[3][1]=2,dp[3][2]=2,dp[4][1]=3)=3
        j=3(s): s==s → dp[4][3]=dp[3][2]=2

i=5(e): j=1(r): e≠r → 1+min(dp[4][0]=4,dp[4][1]=3,dp[5][0]=5)=4
        j=2(o): e≠o → 1+min(dp[4][1]=3,dp[4][2]=3,dp[5][1]=4)=4
        j=3(s): e≠s → 1+min(dp[4][2]=3,dp[4][3]=2,dp[5][2]=4)=3

Answer: dp[5][3] = 3
Ops: horse → rorse (replace h→r) → rose (delete r) → ros (delete e) = 3 ops ✓
```

**WHY three neighbors instead of two:** LCS only ever "skips" a character (moving to `dp[i-1][j]` or `dp[i][j-1]`), because skipping is free. Edit Distance additionally allows "pay 1 to make them match right now" — that's the `dp[i-1][j-1]` replace option, which LCS never needs because LCS has no concept of forcing a match.

---

## SECTION 6 — TEMPLATE 4: LONGEST PALINDROMIC SUBSEQUENCE (LCS IN DISGUISE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 516 — Longest Palindromic Subsequence
// Find the length of the longest subsequence of s that is a palindrome.
//
// KEY INSIGHT: A palindromic subsequence of s is, by definition, a sequence
// that reads the same forward and backward. That is EXACTLY a common
// subsequence between s and reverse(s) — because any subsequence that
// appears in both s (left-to-right) and reverse(s) (which is s read
// right-to-left) must be palindromic.
//
// LPS(s) = LCS(s, reverse(s))
//
// This is a REDUCTION, not a new algorithm — reuse Template 1 verbatim.
// ─────────────────────────────────────────────────────────────────

int longestPalindromeSubseq(string s) {
    string rev = s;
    reverse(rev.begin(), rev.end());
    return longestCommonSubsequence(s, rev);  // exact same function from Section 3
}
```

**TRACE for s="bbbab" (expected answer: 4, from "bbbb"):**
```
s1 = "bbbab", s2 = reverse(s1) = "babbb"

Full LCS dp table (6x6):
        ""  b  a  b  b  b
  ""     0  0  0  0  0  0
  b      0  1  1  1  1  1
  b      0  1  1  2  2  2
  b      0  1  1  2  3  3
  a      0  1  2  2  3  3
  b      0  1  2  3  3  4

Row-by-row (s1 index i, s2="b,a,b,b,b"):
i=1 (b): matches s2[0]='b' → dp[1][1]=1; then all later s2 chars ≠ 'b' pattern continues via max-carry
i=2 (b): j=1 match→dp=dp[1][0]+1=1; j=3(b) match→dp[2][3]=dp[1][2]+1=1+1=2; j=4,5(b) match too→2,2
i=3 (b): j=1 match→1; j=3 match→dp[2][2]+1=1+1=2; j=4 match→dp[2][3]+1=2+1=3; j=5 match→dp[2][4]+1=2+1=3
i=4 (a): j=2(a) match → dp[4][2]=dp[3][1]+1=1+1=2; other columns carry max forward via mismatch rule
i=5 (b): j=1 match→1; j=3 match→dp[4][2]+1=2+1=3; j=4 match→dp[4][3]+1=2+1=3; j=5 match→dp[4][4]+1=3+1=4

Answer: dp[5][5] = 4  ✓  (the palindrome "bbbb")
```

**WHY this reduction is exact (not approximate):** Every character used in the LCS of `s` and `reverse(s)` was picked at some position `p` reading forward AND at some mirrored position reading backward — which is precisely the definition of a palindrome: symmetric around its center. No extra correctness argument is needed beyond "palindromic subsequence = self-and-mirror common subsequence."

---

## SECTION 7 — TEMPLATE 5: SHORTEST COMMON SUPERSEQUENCE (RECONSTRUCTION FROM LCS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1092 — Shortest Common Supersequence
// Find the shortest string that has BOTH s1 and s2 as subsequences.
//
// KEY INSIGHT: |SCS| = |s1| + |s2| - |LCS(s1,s2)|
//   Every character NOT in the LCS must appear in the SCS exactly once
//   (from whichever string it belongs to). Every character IN the LCS
//   is shared — it only needs to appear once in the SCS, saving a slot.
//
// Reconstruction: walk the LCS dp table backward like Template 2, but
// instead of only recording matched characters, ALSO record the
// unmatched characters we skip over (they must appear in the SCS).
// ─────────────────────────────────────────────────────────────────

string shortestCommonSupersequence(string s1, string s2) {
    int m = s1.size(), n = s2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = (s1[i-1] == s2[j-1]) ? dp[i-1][j-1] + 1
                                             : max(dp[i-1][j], dp[i][j-1]);

    string result;
    int i = m, j = n;
    while (i > 0 && j > 0) {
        if (s1[i-1] == s2[j-1]) {
            result += s1[i-1];   // shared char — appears ONCE in SCS
            i--; j--;
        } else if (dp[i-1][j] >= dp[i][j-1]) {
            result += s1[i-1];   // s1's char is not shared — must include it
            i--;
        } else {
            result += s2[j-1];   // s2's char is not shared — must include it
            j--;
        }
    }
    // Append any leftover prefix (one of the strings may be exhausted first)
    while (i > 0) { result += s1[i-1]; i--; }
    while (j > 0) { result += s2[j-1]; j--; }

    reverse(result.begin(), result.end());
    return result;
}
```

**TRACE for s1="abac", s2="cab" (expected: "cabac", length 5):**
```
LCS dp table (5x4):
        ""  c  a  b
  ""     0  0  0  0
  a      0  0  1  1
  b      0  0  1  2
  a      0  0  1  2
  c      0  1  1  2

LCS length = dp[4][3] = 2  ("ab")
SCS length = |s1|+|s2|-LCS = 4+3-2 = 5

Backtrack from (i=4,j=3):
  s1[3]='c', s2[2]='b' → mismatch. dp[3][3]=2 ≥ dp[4][2]=1 → take s1[3]='c'. Append 'c'. (3,3).
  s1[2]='a', s2[2]='b' → mismatch. dp[2][3]=2 ≥ dp[3][2]=1 → take s1[2]='a'. Append 'a'. (2,3).
  s1[1]='b', s2[2]='b' → MATCH. Append 'b' (shared). (1,2).
  s1[0]='a', s2[1]='a' → MATCH. Append 'a' (shared). (0,1).
  i==0 → flush remaining s2: append s2[0]='c'. (0,0).

Collected (append order): c,a,b,a,c → reverse → "cabac"
Length = 5 ✓, and "abac" ⊆ "cabac" (indices 1,2,3,4), "cab" ⊆ "cabac" (indices 0,1,3) ✓
```

**WHY this always produces a valid supersequence:** At every step we either consume ONE shared character (advancing both pointers, contributing 1 char to the result) or ONE private character (advancing one pointer, contributing 1 char). Every character of both strings is eventually emitted in order, and shared characters are emitted exactly once — satisfying both subsequence containments simultaneously.

---

## SECTION 8 — TEMPLATE 6: DISTINCT SUBSEQUENCES (COUNTING PATHS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 115 — Distinct Subsequences
// Count the number of distinct ways s (source) can form t (target)
// as a subsequence.
//
// State:      dp[i][j] = number of distinct subsequences of s[0..i-1]
//                          that equal t[0..j-1]
// Transition: if s[i-1] == t[j-1]:
//                 dp[i][j] = dp[i-1][j-1]   ← USE s[i-1] to match t[j-1]
//                          + dp[i-1][j]     ← SKIP s[i-1] (don't use it, even though it could match)
//             else:
//                 dp[i][j] = dp[i-1][j]     ← s[i-1] can't help; must skip it
//
// Base cases: dp[i][0] = 1 for all i  (empty target is a subsequence of anything, exactly 1 way)
//             dp[0][j] = 0 for j > 0  (nonempty target can't come from empty source)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

int numDistinct(string s, string t) {
    int m = s.size(), n = t.size();
    // Use long long / unsigned to avoid overflow on long strings — counts grow combinatorially
    vector<vector<unsigned long long>> dp(m + 1, vector<unsigned long long>(n + 1, 0));

    for (int i = 0; i <= m; i++) dp[i][0] = 1;   // empty target: exactly 1 way (choose nothing)

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            dp[i][j] = dp[i-1][j];               // always allowed: skip s[i-1]
            if (s[i-1] == t[j-1]) {
                dp[i][j] += dp[i-1][j-1];         // additionally allowed: match s[i-1] to t[j-1]
            }
        }
    }

    return (int)dp[m][n];
}
```

**FULL TRACE for s="babgbag", t="bag" (classic example, expected answer: 5):**
```
        ""  b  a  g
  ""     1  0  0  0
  b      1  1  0  0
  a      1  1  1  0
  b      1  2  1  0
  g      1  2  1  1
  b      1  3  1  1
  a      1  3  4  1
  g      1  3  4  5

i=1(b): j=1(b) match → dp[1][1]=dp[0][0]+dp[0][1]=1+0=1
i=2(a): j=2(a) match → dp[2][2]=dp[1][1]+dp[1][2]=1+0=1
i=3(b): j=1(b) match → dp[3][1]=dp[2][0]+dp[2][1]=1+1=2
i=4(g): j=3(g) match → dp[4][3]=dp[3][2]+dp[3][3]=1+0=1
i=5(b): j=1(b) match → dp[5][1]=dp[4][0]+dp[4][1]=1+2=3
i=6(a): j=2(a) match → dp[6][2]=dp[5][1]+dp[5][2]=3+1=4
i=7(g): j=3(g) match → dp[7][3]=dp[6][2]+dp[6][3]=4+1=5

Answer: dp[7][3] = 5
The 5 ways: ba(b)(g)ag → positions {1,2,4},{1,2,7}? etc. — 5 distinct index-subsets spell "bag". ✓
```

**WHY `dp[i][j] = dp[i-1][j] + dp[i-1][j-1]` on match (not `max`, not `min`):** This is a COUNTING problem, not an optimization problem. Every distinct way to form `t[0..j-1]` either (a) does NOT use `s[i-1]` at all — that count is `dp[i-1][j]` — or (b) DOES use `s[i-1]` to match `t[j-1]` — that count is `dp[i-1][j-1]`. These are disjoint sets of ways, so we ADD them.

---

## SECTION 9 — TEMPLATE 7: MINIMUM ASCII DELETE SUM (WEIGHTED LCS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 712 — Minimum ASCII Delete Sum for Two Strings
// Find the minimum sum of ASCII values of characters that must be
// deleted from s1 and/or s2 to make the two strings equal.
//
// KEY INSIGHT: this is Edit Distance restricted to ONLY delete
// operations, with a WEIGHTED cost (ASCII value) instead of unit cost.
// Equivalently: it is "weighted anti-LCS" — every character not in the
// common subsequence must be deleted (paid for) from whichever side
// it's on.
//
// State:      dp[i][j] = min ASCII sum of deletions to equalize
//                         s1[0..i-1] and s2[0..j-1]
// Transition: if s1[i-1] == s2[j-1]: dp[i][j] = dp[i-1][j-1]        (no deletion needed — free)
//             else: dp[i][j] = min(dp[i-1][j] + ascii(s1[i-1]),     // delete from s1
//                                   dp[i][j-1] + ascii(s2[j-1]))    // delete from s2
// Base cases: dp[i][0] = sum of ascii(s1[0..i-1])  (delete all of s1)
//             dp[0][j] = sum of ascii(s2[0..j-1])  (delete all of s2)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

int minimumDeleteSum(string s1, string s2) {
    int m = s1.size(), n = s2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));

    for (int i = 1; i <= m; i++) dp[i][0] = dp[i-1][0] + s1[i-1];
    for (int j = 1; j <= n; j++) dp[0][j] = dp[0][j-1] + s2[j-1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1[i-1] == s2[j-1]) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = min(dp[i-1][j] + (int)s1[i-1],
                                dp[i][j-1] + (int)s2[j-1]);
            }
        }
    }

    return dp[m][n];
}
```

**TRACE for s1="sea", s2="eat" (expected answer: 231):**
```
ascii: s=115, e=101, a=97, t=116

        ""    e    a    t
  ""     0  101  198  314
  s     115  216  313  429
  e     216  115  212  328
  a     313  212  115  231

dp[1][0]=115 (delete 's')
dp[2][0]=115+101=216 (delete 's','e')
dp[3][0]=216+97=313 (delete 's','e','a')
dp[0][1]=101, dp[0][2]=101+97=198, dp[0][3]=198+116=314

i=1(s): j=1(e): s≠e → min(dp[0][1]+115=216, dp[1][0]+101=216)=216
        j=2(a): s≠a → min(dp[0][2]+115=313, dp[1][1]+97=313)=313
        j=3(t): s≠t → min(dp[0][3]+115=429, dp[1][2]+116=429)=429

i=2(e): j=1(e): match → dp[2][1]=dp[1][0]=115
        j=2(a): e≠a → min(dp[1][2]+101=414, dp[2][1]+97=212)=212
        j=3(t): e≠t → min(dp[1][3]+101=530, dp[2][2]+116=328)=328

i=3(a): j=1(e): a≠e → min(dp[2][1]+97=212, dp[3][0]+101=414)=212
        j=2(a): match → dp[3][2]=dp[2][1]=115
        j=3(t): a≠t → min(dp[2][3]+97=425, dp[3][2]+116=231)=231

Answer: dp[3][3] = 231
Deleted: 's'(115) from "sea", 't'(116) from "eat" → 115+116=231 ✓ (both become "ea")
```

**WHY this is NOT the same code as plain LCS with a sum:** You cannot compute `(sum of all ascii) - 2*ascii(LCS)` naively for the general case in one clean pass without care, because the WEIGHTED delete-sum DP directly encodes "pay to delete the character now" at each mismatch step — it is cleaner and more general to write the dedicated recurrence above than to bolt weights onto the unweighted LCS length formula.

---

## SECTION 10 — TEMPLATE 8: WILDCARD MATCHING (`*` = ANY SEQUENCE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 44 — Wildcard Matching
// '?' matches any SINGLE character.
// '*' matches any SEQUENCE of characters, INCLUDING THE EMPTY SEQUENCE.
//     ('*' here is a standalone token — it does NOT modify a preceding char)
//
// State:      dp[i][j] = does s[0..i-1] match pattern p[0..j-1]?
// Transition:
//   if p[j-1] == '*':
//       dp[i][j] = dp[i-1][j]      // '*' consumes one more char of s (matches ≥1 char so far)
//                | dp[i][j-1]      // '*' matches the EMPTY sequence (consume none)
//   elif p[j-1] == '?' || s[i-1] == p[j-1]:
//       dp[i][j] = dp[i-1][j-1]    // single-char match, advance both
//   else:
//       dp[i][j] = false
//
// Base cases: dp[0][0] = true (empty matches empty)
//             dp[0][j] = dp[0][j-1] && p[j-1]=='*'   (only a run of leading '*'s can match empty s)
//             dp[i][0] = false for i>0  ('*' free' can't come from nothing on pattern side)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

bool isMatch_wildcard(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;

    // Leading stars can all match the empty string
    for (int j = 1; j <= n; j++)
        dp[0][j] = dp[0][j-1] && (p[j-1] == '*');

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') {
                dp[i][j] = dp[i-1][j] || dp[i][j-1];
            } else if (p[j-1] == '?' || s[i-1] == p[j-1]) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = false;
            }
        }
    }

    return dp[m][n];
}
```

**TRACE for s="adceb", p="\*a\*b" (expected: true):**
```
p = *, a, *, b   (4 chars).  s = a, d, c, e, b   (5 chars)

Row0: dp[0][0]=T. dp[0][1]: p[0]='*' → T.  dp[0][2]: p[1]='a' ≠'*' → F.
      dp[0][3]: dp[0][2]=F → F.  dp[0][4]: F.
Row0 = [T, T, F, F, F]

i=1 (s='a'): j=1('*'): dp=dp[0][1]||dp[1][0]=T||F=T
             j=2('a'): match → dp=dp[0][1]=T
             j=3('*'): dp=dp[0][3]||dp[1][2]=F||T=T
             j=4('b'): a≠b → F
Row1 = [F, T, T, T, F]

i=2 (s='d'): j=1('*'): dp=dp[1][1]||dp[2][0]=T||F=T
             j=2('a'): d≠a → F
             j=3('*'): dp=dp[1][3]||dp[2][2]=T||F=T
             j=4('b'): d≠b → F
Row2 = [F, T, F, T, F]

i=3 (s='c'): same shape as row2 (c doesn't match 'a' or 'b') → [F, T, F, T, F]
i=4 (s='e'): same shape → [F, T, F, T, F]

i=5 (s='b'): j=1('*'): dp=dp[4][1]||dp[5][0]=T
             j=2('a'): b≠a → F
             j=3('*'): dp=dp[4][3]||dp[5][2]=T||F=T
             j=4('b'): match → dp=dp[4][3]=T

Answer: dp[5][4] = true ✓  ("adceb" matches "*a*b": leading * absorbs nothing,
                              'a' matches 'a', middle * absorbs "dce", 'b' matches 'b')
```

---

## SECTION 11 — TEMPLATE 9: REGULAR EXPRESSION MATCHING (`*` = ZERO-OR-MORE OF PRECEDING)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 10 — Regular Expression Matching
// '.' matches any SINGLE character.
// '*' matches ZERO OR MORE occurrences of the character immediately
//     PRECEDING it. '*' is a MODIFIER, never appears alone at p[0]
//     validly without a preceding char, and NEVER matches by itself.
//
// State:      dp[i][j] = does s[0..i-1] match pattern p[0..j-1]?
// Transition:
//   if p[j-1] == '*':
//       // '*' applies to p[j-2] (the character before it)
//       dp[i][j] = dp[i][j-2]                                  // ZERO occurrences: skip "x*" entirely
//                | (matches(s[i-1], p[j-2]) && dp[i-1][j])      // ONE+ occurrences: consume one s char,
//                                                                //   stay on the same "x*" (it can repeat)
//   else:
//       dp[i][j] = matches(s[i-1], p[j-1]) && dp[i-1][j-1]
//
//   where matches(c, pc) = (pc == '.' || c == pc)
//
// Base cases: dp[0][0] = true
//             dp[0][j] = j>=2 && p[j-1]=='*' && dp[0][j-2]   (only "x*y*z*..." patterns match empty s)
// Answer:     dp[m][n]
// ─────────────────────────────────────────────────────────────────

bool isMatch_regex(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;

    for (int j = 1; j <= n; j++) {
        if (p[j-1] == '*' && j >= 2) {
            dp[0][j] = dp[0][j-2];   // "x*" can vanish (zero occurrences), so empty pattern prefix works
        }
    }

    auto matches = [&](int i, int j) {
        // i, j are 1-indexed positions being compared: s[i-1] vs p[j-1]
        return p[j-1] == '.' || s[i-1] == p[j-1];
    };

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (p[j-1] == '*') {
                // p[j-2] is the character '*' applies to (guaranteed valid input: j>=2)
                bool zeroOcc = dp[i][j-2];
                bool oneOrMoreOcc = matches(i, j-1) && dp[i-1][j];
                dp[i][j] = zeroOcc || oneOrMoreOcc;
            } else {
                dp[i][j] = matches(i, j) && dp[i-1][j-1];
            }
        }
    }

    return dp[m][n];
}
```

**TRACE for s="aab", p="c\*a\*b" (expected: true):**
```
p = c, *, a, *, b   (5 chars).  s = a, a, b   (3 chars)

Row0: dp[0][0]=T
  j=1(c): not '*' → F
  j=2(*, applies to 'c'): dp[0][2]=dp[0][0]=T   (zero 'c's)
  j=3(a): not '*' → F
  j=4(*, applies to 'a'): dp[0][4]=dp[0][2]=T   (zero 'a's, after zero 'c's)
  j=5(b): not '*' → F
Row0 = [T, F, T, F, T, F]

i=1 (s='a'): j=1('c'): a≠c → F
             j=2('*' on 'c'): zeroOcc=dp[1][0]=F; oneOrMore=matches(a,c)&&dp[0][2] = F&&T = F → dp[1][2]=F
             j=3('a'): match → dp[1][3]=dp[0][2]=T
             j=4('*' on 'a'): zeroOcc=dp[1][2]=F; oneOrMore=matches(a,a)&&dp[0][4]=T&&T=T → dp[1][4]=T
             j=5('b'): a≠b → F
Row1 = [F, F, F, T, T, F]

i=2 (s='a'): j=1('c'): F
             j=2('*' on 'c'): zeroOcc=dp[2][0]=F; oneOrMore=matches(a,c)&&dp[1][2]=F&&F=F → F
             j=3('a'): match → dp[2][3]=dp[1][2]=F
             j=4('*' on 'a'): zeroOcc=dp[2][2]=F; oneOrMore=matches(a,a)&&dp[1][4]=T&&T=T → dp[2][4]=T
             j=5('b'): a≠b → F
Row2 = [F, F, F, F, T, F]

i=3 (s='b'): j=1('c'): F
             j=2('*' on 'c'): zeroOcc=dp[3][0]=F; oneOrMore=matches(b,c)&&dp[2][2]=F&&F=F → F
             j=3('a'): b≠a → F
             j=4('*' on 'a'): zeroOcc=dp[3][2]=F; oneOrMore=matches(b,a)&&dp[2][4]=F&&T=F → F
             j=5('b'): match → dp[3][5]=dp[2][4]=T

Answer: dp[3][5] = true ✓  ("aab" matches "c*a*b": zero 'c's, two 'a's, one 'b')
```

**WHY `dp[i][j-2]` (zero occurrences) must ALWAYS be checked first, even when a match is possible:** The "one-or-more" branch can only fire from `dp[i-1][j]`, which requires `i >= 1`. If `s` is empty at this point (i=0) but the pattern has a trailing `x*`, only the zero-occurrence path (`dp[0][j-2]`) can be true. Forgetting the zero-occurrence branch is the single most common bug in this problem — it causes patterns like `"a*"` to fail against `""`.

---

## SECTION 12 — TEMPLATE 10: LONGEST PALINDROMIC SUBSTRING (SUBSTRING, NOT SUBSEQUENCE)

This problem looks like a cousin of LPS (Section 6) but is a DIFFERENT problem type: contiguous substring, not subsequence. It gets a Type-5 interval DP (single string, no second sequence) — included here because confusing it with LPS is a top interview trap.

### Approach A: Expand Around Center — O(n²) time, O(1) space

```cpp
// LC 5 — Longest Palindromic Substring
// A palindrome has a center. Odd-length palindromes center on ONE character;
// even-length palindromes center on a GAP BETWEEN two characters.
// There are 2n-1 possible centers (n single chars + n-1 gaps).
// Expand outward from each center while characters match.

pair<int,int> expandAroundCenter(const string& s, int left, int right) {
    // Expand while in bounds and characters match
    while (left >= 0 && right < (int)s.size() && s[left] == s[right]) {
        left--; right++;
    }
    // Loop over-expanded by one step on each side — return the valid bounds
    return {left + 1, right - 1};
}

string longestPalindrome_expand(string s) {
    if (s.empty()) return "";
    int bestStart = 0, bestLen = 1;

    for (int center = 0; center < (int)s.size(); center++) {
        // Odd-length palindrome: center is a single character
        auto [l1, r1] = expandAroundCenter(s, center, center);
        if (r1 - l1 + 1 > bestLen) { bestStart = l1; bestLen = r1 - l1 + 1; }

        // Even-length palindrome: center is the gap between center and center+1
        auto [l2, r2] = expandAroundCenter(s, center, center + 1);
        if (r2 - l2 + 1 > bestLen) { bestStart = l2; bestLen = r2 - l2 + 1; }
    }

    return s.substr(bestStart, bestLen);
}
```

### Approach B: DP on Substrings — O(n²) time, O(n²) space

```cpp
// State:      dp[i][j] = true if s[i..j] (inclusive, contiguous) is a palindrome
// Transition: dp[i][j] = (s[i] == s[j]) && (j - i < 2 || dp[i+1][j-1])
//             (j-i<2 means length ≤ 2: single char or two equal adjacent chars — always valid
//              once the outer characters match, no inner check needed)
// CRITICAL: must fill by INCREASING LENGTH (or i decreasing / j increasing),
// because dp[i][j] depends on dp[i+1][j-1] — a SHORTER, INNER interval.

string longestPalindrome_dp(string s) {
    int n = s.size();
    if (n == 0) return "";
    vector<vector<bool>> dp(n, vector<bool>(n, false));
    int bestStart = 0, bestLen = 1;

    // Fill by increasing substring length
    for (int len = 1; len <= n; len++) {
        for (int i = 0; i + len - 1 < n; i++) {
            int j = i + len - 1;
            if (s[i] == s[j] && (len < 3 || dp[i+1][j-1])) {
                dp[i][j] = true;
                if (len > bestLen) { bestStart = i; bestLen = len; }
            }
        }
    }

    return s.substr(bestStart, bestLen);
}
```

**TRACE (DP version) for s="babad":**
```
s = b(0) a(1) b(2) a(3) d(4)

len=1: dp[i][i]=true for all i (trivial palindromes of length 1)

len=2: dp[i][i+1] = (s[i]==s[i+1])
  (0,1): b≠a → F   (1,2): a≠b → F   (2,3): b≠a → F   (3,4): a≠d → F
  (no length-2 palindromes)

len=3: dp[i][i+2] = (s[i]==s[i+2]) && true (len<3 is false, but the "inner" is a single
                     char which is trivially a palindrome — handled by dp[i+1][j-1]=dp[i+1][i+1]=true)
  (0,2): s[0]=b, s[2]=b → equal, dp[1][1]=true → dp[0][2]=TRUE → "bab"
  (1,3): s[1]=a, s[3]=a → equal, dp[2][2]=true → dp[1][3]=TRUE → "aba"
  (2,4): s[2]=b, s[4]=d → not equal → F

len=4: (0,3): s[0]=b,s[3]=a → F.  (1,4): s[1]=a,s[4]=d → F.

len=5: (0,4): s[0]=b,s[4]=d → F.

Best length found = 3, first found at (0,2) → "bab" (or "aba" — both are valid answers) ✓
```

**WHY the fill order matters here (unlike LCS):** `dp[i][j]` reads `dp[i+1][j-1]` — a strictly SMALLER, NESTED interval. If you filled by increasing `i` with `j` free (like a normal 2D DP), you could read `dp[i+1][j-1]` before it's computed. Filling by increasing LENGTH guarantees every inner interval is already resolved.

---

## SECTION 13 — TEMPLATE 11: MINIMUM INSERTION STEPS TO MAKE A PALINDROME (LPS IN DISGUISE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1312 — Minimum Insertion Steps to Make a String Palindrome
// Find the minimum number of characters to INSERT anywhere in s to
// make the whole string a palindrome.
//
// KEY INSIGHT: Every character NOT part of the longest palindromic
// subsequence (LPS) needs a "mirror partner" inserted for it.
// The characters that ARE in the LPS already form a palindromic
// skeleton and need no insertions.
//
// minInsertions(s) = |s| - LPS(s)
//
// This reuses Template 4 (LCS(s, reverse(s))) with zero new code.
// ─────────────────────────────────────────────────────────────────

int minInsertions(string s) {
    int n = s.size();
    int lps = longestPalindromeSubseq(s);  // exact function from Section 6
    return n - lps;
}
```

**TRACE for s="mbadm" (expected answer: 2):**
```
s1 = "mbadm", s2 = reverse = "mdabm"

LCS dp table (6x6):
        ""  m  d  a  b  m
  ""     0  0  0  0  0  0
  m      0  1  1  1  1  1
  b      0  1  1  1  2  2
  a      0  1  1  2  2  2
  d      0  1  2  2  2  2
  m      0  1  2  2  2  3

Answer: LPS = dp[5][5] = 3   (the palindromic subsequence "mdm" or "aba"-shape works out to length 3)
n - LPS = 5 - 3 = 2

Verify: "mbadm" → insert 'a' after 'd' and mirror... concretely, "mbadm" needs 2 insertions,
e.g. → "mbdadbm" is NOT minimal; the standard minimal result is "madbadam"-style reasoning
confirms exactly 2 insertions suffice (matches LC1312's published answer for this input). ✓
```

**WHY this reduction holds:** Take the LPS of `s`. Every character OUTSIDE the LPS is "unpaired" — for the whole string to become a palindrome, each unpaired character needs a mirrored partner inserted on the opposite side. There are exactly `n - LPS` unpaired characters, hence `n - LPS` insertions, and this is provably optimal (you can never need fewer, because the LPS is already the largest set of characters that can serve as their own palindrome skeleton for free).

---

## SECTION 14 — COMPLEXITY TABLE

| Problem | Time | Space | Optimized Space | Notes |
|---------|------|-------|------------------|-------|
| LCS | O(mn) | O(mn) | O(min(m,n)) rolling row | Foundation template |
| LCS Reconstruction | O(mn) | O(mn) | Not reducible (need full table to backtrack) | Backtrack after fill |
| Edit Distance | O(mn) | O(mn) | O(min(m,n)) rolling row | 3-way min, +1 cost |
| Longest Palindromic Subsequence | O(n²) | O(n²) | O(n) rolling row | = LCS(s, reverse(s)) |
| Shortest Common Supersequence | O(mn) | O(mn) | Not reducible (need reconstruction) | = m+n-LCS + backtrack |
| Distinct Subsequences | O(mn) | O(mn) | O(n) rolling row | Counting, use unsigned/long long |
| Min ASCII Delete Sum | O(mn) | O(mn) | O(n) rolling row | Weighted LCS variant |
| Wildcard Matching | O(mn) | O(mn) | O(n) rolling row | '*' = any sequence |
| Regular Expression Matching | O(mn) | O(mn) | O(n) rolling row (careful: needs j-2) | '*' = 0+ of preceding char |
| Longest Palindromic Substring (expand) | O(n²) | O(1) | Already optimal | Fastest in practice |
| Longest Palindromic Substring (DP) | O(n²) | O(n²) | O(n) with careful order (rare) | Generalizes to interval DP |
| Min Insertions for Palindrome | O(n²) | O(n²) | O(n) rolling row | = n - LPS(s) |

---

## SECTION 15 — VARIANTS

1. **Longest Common Substring** (contiguous, not subsequence): `dp[i][j]` = length of common substring ENDING exactly at `s1[i-1]` and `s2[j-1]`. On match: `dp[i][j] = dp[i-1][j-1] + 1`. On mismatch: `dp[i][j] = 0` (RESETS — cannot skip and reconnect, unlike LCS's `max`). Answer = `max` over all cells, not `dp[m][n]`.

2. **Uncrossed Lines (LC 1035):** Literally identical code to LCS — draw non-crossing lines connecting equal values between two arrays. The "non-crossing" constraint is exactly the "preserve relative order" constraint of a common subsequence.

3. **Delete Operation for Two Strings (LC 583):** `minDeletions = m + n - 2*LCS(s1,s2)` — delete every character not in the LCS from both sides.

4. **Longest Repeating Subsequence:** LCS of a string with itself, with the constraint `i != j` at matching positions (to avoid trivially "matching" a character with itself at the same index).

5. **Palindrome Partitioning II (LC 132):** Uses the same `dp[i][j]` = "is s[i..j] a palindrome" table from Template 10 as a precomputed lookup, then runs a second 1D DP for minimum cuts — a two-stage combination of Pattern 20 (Type 5) and Pattern 17 (1D DP).

6. **Regex/Wildcard with `+` or `{m,n}` quantifiers:** Real regex engines extend the `*` transition with more quantifiers; the core `dp[i][j]` skeleton is unchanged — only the transition function for the quantifier character grows more cases.

---

## SECTION 16 — COMMON MISTAKES

### Mistake 1: Substring vs Subsequence confusion

```cpp
// WRONG — treating Longest Common SUBSTRING like LCS (using max on mismatch)
if (s1[i-1] == s2[j-1]) dp[i][j] = dp[i-1][j-1] + 1;
else dp[i][j] = max(dp[i-1][j], dp[i][j-1]);   // BUG for substring problems!
// This computes LCS (subsequence), not longest common SUBSTRING.
// A substring must be contiguous — once characters mismatch, the run is broken,
// it cannot "skip" and continue elsewhere.

// CORRECT — substring resets to 0 on mismatch
if (s1[i-1] == s2[j-1]) dp[i][j] = dp[i-1][j-1] + 1;
else dp[i][j] = 0;                              // reset — no partial credit
// Answer = max over ALL cells, not just dp[m][n], since the best substring
// can end anywhere in the table.
```

---

### Mistake 2: Off-by-one in table dimensions

```cpp
// WRONG — sizing dp as m x n instead of (m+1) x (n+1)
vector<vector<int>> dp(m, vector<int>(n, 0));
for (int i = 1; i < m; i++)
    for (int j = 1; j < n; j++)
        dp[i][j] = (s1[i] == s2[j]) ? dp[i-1][j-1] + 1 : max(dp[i-1][j], dp[i][j-1]);
// BUG: this silently DROPS the last character of both strings (loop bounds are
// i < m, j < n, and dp[0][*]/dp[*][0] represent REAL characters, not "empty prefix").
// There is no clean base case for "empty string" — off-by-one errors compound.

// CORRECT — always size (m+1) x (n+1); row/col 0 = "empty prefix" sentinel
vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
for (int i = 1; i <= m; i++)
    for (int j = 1; j <= n; j++)
        dp[i][j] = (s1[i-1] == s2[j-1]) ? dp[i-1][j-1] + 1 : max(dp[i-1][j], dp[i][j-1]);
// Now dp[0][j] and dp[i][0] cleanly represent "0 characters used" with no special-casing.
```

---

### Mistake 3: Regex `*` — forgetting the zero-occurrence branch

```cpp
// WRONG — only handling "one or more" occurrences of the preceding char
if (p[j-1] == '*') {
    dp[i][j] = matches(i, j-1) && dp[i-1][j];    // BUG: missing zero-occurrence case!
}
// This fails on patterns like p="a*" matching s="" — there IS a valid match
// (zero 'a's), but this code never checks dp[i][j-2].

// CORRECT — always OR in the zero-occurrence branch
if (p[j-1] == '*') {
    dp[i][j] = dp[i][j-2] ||                       // zero occurrences: skip "x*" entirely
                (matches(i, j-1) && dp[i-1][j]);    // one+ occurrences: consume a char, stay on "x*"
}
```

---

### Mistake 4: Wildcard `*` vs Regex `*` — treating them the same

```cpp
// WRONG — applying the regex '*' transition (zero-or-more of PRECEDING char)
// to a Wildcard Matching problem
if (p[j-1] == '*') {
    dp[i][j] = dp[i][j-2] || (matches(i, j-1) && dp[i-1][j]);   // BUG for LC44!
}
// In Wildcard Matching, '*' is a STANDALONE token that matches ANY sequence.
// It does NOT modify "the preceding character" — there might not even BE one
// (e.g. pattern = "*" alone, matching everything). p[j-2] is meaningless here.

// CORRECT — wildcard '*' transition: it matches either "one more char of s" or "nothing"
if (p[j-1] == '*') {
    dp[i][j] = dp[i-1][j] ||   // '*' absorbs s[i-1] (matches ≥1 char so far), keep trying with more of s
                dp[i][j-1];    // '*' absorbs nothing (matches empty), move past it in the pattern
}
// The two problems' '*' transitions are NOT interchangeable. Confusing LC10 and
// LC44 solutions is the #1 reported mistake at this pattern in interviews.
```

---

### Mistake 5: Edit Distance — forgetting the "free" case when characters already match

```cpp
// WRONG — always paying a cost, even when characters already match
dp[i][j] = 1 + min({dp[i-1][j-1], dp[i-1][j], dp[i][j-1]});   // BUG: no match check!
// This overcounts operations — if s1[i-1]==s2[j-1], no operation is needed at all,
// yet this code always adds 1.

// CORRECT — check for a match first; only pay cost on mismatch
if (s1[i-1] == s2[j-1]) {
    dp[i][j] = dp[i-1][j-1];                                  // free — already equal
} else {
    dp[i][j] = 1 + min({dp[i-1][j-1], dp[i-1][j], dp[i][j-1]});
}
```

---

### Mistake 6: Reducing to LPS/LCS but forgetting to actually reverse the string

```cpp
// WRONG — passing s and s to LCS instead of s and reverse(s)
int lps(string s) {
    return longestCommonSubsequence(s, s);    // BUG: LCS(s,s) is just |s| — trivially the whole string!
}
// Any string is trivially its own longest common subsequence with itself.
// This does NOT compute the longest PALINDROMIC subsequence — it silently
// returns n every time, which is wrong for any non-palindromic input.

// CORRECT — must reverse one copy before calling LCS
int lps(string s) {
    string rev = s;
    reverse(rev.begin(), rev.end());
    return longestCommonSubsequence(s, rev);   // LCS(s, reverse(s)) — the actual reduction
}
```

---

## SECTION 17 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Longest Common Subsequence | 1143 | Classic diagonal-match, max-on-mismatch template |
| 2 | Edit Distance | 72 | LCS with costs — 3-way min, free-match base case |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Longest Palindromic Subsequence | 516 | LCS with reversed string reduction |
| 4 | Shortest Common Supersequence | 1092 | m+n-LCS formula + reconstruction/backtrack |
| 5 | Distinct Subsequences | 115 | Counting DP — sum, not max, on match |
| 6 | Minimum ASCII Delete Sum | 712 | Weighted LCS — cost-based transitions |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Interleaving String | 97 | Two-sequence DP (see Pattern 18 for full template) |
| 8 | Wildcard Matching | 44 | '*' = any sequence — standalone token semantics |
| 9 | Regular Expression Matching | 10 | '*' = zero-or-more of preceding char — modifier semantics |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Longest Palindromic Substring | 5 | Interval DP / expand-around-center — substring not subsequence |
| 11 | Minimum Insertion Steps to Make Palindrome | 1312 | n - LPS(s) reduction |

---

## SECTION 18 — PATTERN CONNECTIONS

1. **Pattern 18 (2D DP and Grid DP):** Interleaving String (LC 97) was introduced in Pattern 18 as the entry point into two-sequence DP. Pattern 20 is the full expansion of that idea — every template here uses the same `dp[i][j]` = "first i of s1, first j of s2" skeleton that Pattern 18 first established.

2. **Pattern 21 (LIS — Longest Increasing Subsequence):** LIS is the SINGLE-sequence cousin of LCS. In fact, LIS of an array `a` can be computed as `LCS(a, sorted_unique(a))` — a direct reduction from Pattern 21 back into Pattern 20's LCS template. Recognizing this connection turns a seemingly new problem into a template you already know.

3. **Pattern 23 (Interval DP):** Longest Palindromic Substring's `dp[i][j] = (s[i]==s[j]) && dp[i+1][j-1]` is a genuine interval DP — filled by increasing LENGTH, not by row/column. This is the same fill-order discipline (`dp[i][j]` depends on a strictly smaller, NESTED sub-interval) that governs every problem in Pattern 23 (Matrix Chain Multiplication, Burst Balloons, Palindrome Partitioning). Template 10 in this pattern is a preview of Pattern 23's core technique.

4. **Pattern 19 (Knapsack Family):** Distinct Subsequences (LC 115) shares its "sum of two disjoint choices" transition (`dp[i][j] = dp[i-1][j] + dp[i-1][j-1]` on match) with 0/1 Knapsack counting variants (e.g., Target Sum, Combination Sum IV) — both are counting DPs where a match/inclusion ADDS rather than takes `max`.

---

## SECTION 19 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 72 — Edit Distance (a Google favorite)

**Interviewer:** "Given two words, find the minimum number of operations (insert, delete, replace) to convert word1 into word2."

**Candidate:**

*[First minute — establish the state before touching code]*
> "I'll define `dp[i][j]` as the minimum edit operations to convert the first `i` characters of word1 into the first `j` characters of word2. This is a two-sequence DP — the two dimensions are 'how much of word1 have I consumed' and 'how much of word2 have I produced.'"

*[Explain the transition, including WHY three options]*
> "At each cell, if the current characters match, no operation is needed — `dp[i][j] = dp[i-1][j-1]`, free. If they don't match, I have exactly three choices, and I take the cheapest: replace word1's char with word2's char (`dp[i-1][j-1]`), delete word1's char (`dp[i-1][j]`), or insert word2's char into word1 (`dp[i][j-1]`). Each costs 1 plus whatever the sub-problem costs."

*[State base cases explicitly]*
> "`dp[i][0] = i` — turning `i` characters into nothing requires `i` deletions. `dp[0][j] = j` — turning nothing into `j` characters requires `j` insertions."

*[Code — 5 minutes]*
```cpp
int minDistance(string word1, string word2) {
    int m = word1.size(), n = word2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1));
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i][j] = (word1[i-1] == word2[j-1])
                     ? dp[i-1][j-1]
                     : 1 + min({dp[i-1][j-1], dp[i-1][j], dp[i][j-1]});
    return dp[m][n];
}
```

*[Follow-up]: "Can you reduce the space to O(n)?"*
> "Yes — each row only depends on the previous row and the current row's left neighbor. I can use two 1D arrays (`prev`, `curr`), or even one array with a saved `diag` variable, since `dp[i-1][j-1]` gets overwritten before I read it if I'm not careful — same trap as Maximal Square's diagonal term from Pattern 18."

**RED FLAGS:**
- Forgetting the free "match" case and always paying cost 1
- Confusing insert/delete direction (which string gets the operation)
- Not stating base cases before coding
- Being unable to explain the O(n) space reduction when asked

---

### Interview Simulation 2: LC 10 — Regular Expression Matching

**Interviewer:** "Implement regex matching supporting `.` and `*`, where `*` means zero or more of the preceding element."

**Candidate:**

*[Critical: clarify semantics before coding — this problem is a trap for people who assume wildcard semantics]*
> "Before I code, I want to nail the semantics: `*` here is NOT a standalone wildcard like in shell globs — it's a MODIFIER attached to the character immediately before it in the pattern. So in `a*`, the `*` says 'zero or more a's,' not 'zero or more of anything.' This means `*` never appears as `p[0]` in a valid pattern without a preceding char, and every `*` transition must look one character back in the pattern, at `p[j-2]`."

*[State the transition with both branches]*
> "When `p[j-1]` is `*`: two sub-cases. Zero occurrences — I skip 'x*' entirely and check `dp[i][j-2]`. One-or-more occurrences — if the current s character matches `p[j-2]` (the char '*' modifies), I consume one s character and stay on the same 'x*' pattern position, checking `dp[i-1][j]`. I OR these two together."

*[Code — 6 minutes]*
```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m+1, vector<bool>(n+1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n; j++)
        if (p[j-1] == '*' && j >= 2) dp[0][j] = dp[0][j-2];

    auto match = [&](int i, int j) { return p[j-1]=='.' || s[i-1]==p[j-1]; };

    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (p[j-1] == '*')
                dp[i][j] = dp[i][j-2] || (match(i, j-1) && dp[i-1][j]);
            else
                dp[i][j] = match(i, j) && dp[i-1][j-1];
    return dp[m][n];
}
```

*[Interviewer]: "What's the difference between this and Wildcard Matching (LC 44)?"*
> "In LC 44, `*` is a free-standing token that matches any sequence by itself — its transition is `dp[i-1][j] || dp[i][j-1]`, no look-back at a preceding character. Here, `*` always modifies the char at `p[j-2]`; the transition explicitly looks back one more position in the pattern. They look like the same character but encode completely different grammars — I have to keep the two transition formulas mentally separate."

**RED FLAGS:**
- Writing the Wildcard Matching transition instead (conflating the two problems)
- Forgetting the zero-occurrence branch (`dp[i][j-2]`)
- Not handling the base row (`dp[0][j]`) for patterns like `"a*b*c*"` matching empty string
- Off-by-one when indexing `p[j-2]` — must guard `j >= 2`, though input is guaranteed valid so `*` never leads

---

## SECTION 20 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why does LCS use `max(dp[i-1][j], dp[i][j-1])` on mismatch, while Longest Common Substring uses `0` on mismatch?**

> LCS is a SUBSEQUENCE problem — characters can be skipped freely, so on a mismatch we simply "drop" one character from either string and keep the best subsequence found so far via the two neighbors. Longest Common Substring requires CONTIGUITY — a run of matching characters is broken the instant a mismatch occurs, and there is no way to "skip and reconnect" later while staying contiguous. So the running length must reset to 0, and the final answer is the max value seen anywhere in the table (not just the bottom-right corner).

---

**Q2: Give the one-line reduction for Longest Palindromic Subsequence, and explain WHY it's exact.**

> `LPS(s) = LCS(s, reverse(s))`. Any subsequence that appears in both `s` (read forward) and `reverse(s)` (which is `s` read backward) must, by construction, read the same forwards and backwards — which is exactly the definition of a palindrome. The reduction is exact, not approximate, because "common to s and its reverse" and "palindromic subsequence of s" are logically equivalent statements.

---

**Q3: In Regular Expression Matching, why must you check `dp[i][j-2]` (zero occurrences) even when the current characters DO match?**

> Because "zero occurrences" and "one-or-more occurrences" are two INDEPENDENT ways a `x*` pattern segment can be satisfied, and you must OR them together — checking only one branch misses valid matches from the other. Even if `s[i-1]` matches `p[j-2]` right now, the OPTIMAL match might still come from having matched zero occurrences of `x*` earlier and letting the REST of the pattern absorb `s[i-1]` differently. Skipping this check causes valid patterns like `"a*"` vs `""` to be misjudged, and more subtly, causes incorrect results whenever a greedy "always consume when possible" choice isn't actually optimal.

---

**Q4: What is the exact difference between the `*` transition in Wildcard Matching (LC 44) and Regular Expression Matching (LC 10)? Write both transitions from memory.**

> Wildcard (`*` = any sequence, standalone token):
> `dp[i][j] = dp[i-1][j] || dp[i][j-1]`
> (either '*' absorbs one more character of `s`, or '*' matches empty and we move past it in the pattern)
>
> Regex (`*` = zero-or-more of the PRECEDING pattern character):
> `dp[i][j] = dp[i][j-2] || (matches(s[i-1], p[j-2]) && dp[i-1][j])`
> (either skip "x*" entirely — zero occurrences — or consume one `s` char that matches `p[j-2]` and stay on the same "x*")
>
> The key difference: Wildcard's `*` never looks back at another pattern character; Regex's `*` always refers to `p[j-2]`.

---

**Q5: Why does Distinct Subsequences ADD `dp[i-1][j-1]` and `dp[i-1][j]` on a match, while LCS takes `dp[i-1][j-1] + 1` (no addition of the mismatch branch)?**

> Distinct Subsequences is a COUNTING problem — it asks "in how many DISTINCT ways" can `t` be formed. When `s[i-1] == t[j-1]`, there are two mutually exclusive, non-overlapping strategies: use `s[i-1]` to match `t[j-1]` (contributing `dp[i-1][j-1]` ways), or deliberately NOT use `s[i-1]` even though it could match (contributing `dp[i-1][j]` ways, same as a mismatch would). These are disjoint sets of subsequences, so their counts are ADDED.
>
> LCS is an OPTIMIZATION problem — it asks for the single best (longest) subsequence, not a count of all of them. On a match, there is exactly one clearly-optimal move (extend the match diagonally); there's no reason to also consider "not using this match," because using a match can never make the LCS shorter. So LCS only takes the diagonal value plus one, with no addition or max needed at a match cell.

---

**Q6: For Shortest Common Supersequence, why is the formula `|s1| + |s2| - |LCS(s1,s2)|` and not something involving Edit Distance?**

> The SCS must contain every character of both `s1` and `s2` at least once (as subsequences). Characters that are part of the LCS can be "shared" — placed once in the SCS and counted toward both strings' subsequence requirement simultaneously. Every character NOT in the LCS is "private" to one string and must appear once, contributing its own slot. Total length = (private chars of s1) + (private chars of s2) + (shared chars, counted once) = `(|s1| - LCS) + (|s2| - LCS) + LCS = |s1| + |s2| - LCS`.
>
> Edit Distance answers a different question ("minimum operations to transform s1 INTO s2," which can overwrite characters) — SCS never destroys characters, it only adds the minimum necessary, so LCS (which measures "what can be kept for free") is the right building block, not Edit Distance (which measures "cost to force equality").

---

**Q7: You're given a brand-new two-string problem you've never seen. What are the first three questions you ask yourself to figure out which template applies?**

> 1. **Subsequence or substring?** Does the answer require contiguous characters (interval DP, resets on mismatch) or can characters be skipped (diagonal DP, carries forward on mismatch via max/min/sum)?
> 2. **Optimize, count, or decide (bool)?** Optimization problems use `max`/`min` on mismatch; counting problems use `+` (sum of disjoint cases) on match; decision problems use `||`/`&&` (boolean OR/AND of sub-cases).
> 3. **Is this secretly a known template applied to a transformed input?** Check specifically: is one "string" actually the REVERSE of the other (→ palindrome family)? Is the "second sequence" actually a formula derived from the first (→ LIS-as-LCS)? Recognizing a disguised reduction turns 40 minutes of fresh design into 5 minutes of applying a template you already trust.

---

## SECTION 21 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. LCS — THE FOUNDATION TEMPLATE
// ─────────────────────────────────────────────────────────────────
// dp[i][j] = LCS length of s1[0..i-1], s2[0..j-1]
dp[i][j] = (s1[i-1]==s2[j-1]) ? dp[i-1][j-1]+1
                               : max(dp[i-1][j], dp[i][j-1]);

// ─────────────────────────────────────────────────────────────────
// 2. LCS RECONSTRUCTION — backtrack from (m,n)
// ─────────────────────────────────────────────────────────────────
// if match: record char, move diagonally (i--, j--)
// else: move toward the larger neighbor (dp[i-1][j] vs dp[i][j-1])
// remember to reverse() the result at the end

// ─────────────────────────────────────────────────────────────────
// 3. EDIT DISTANCE — LCS with cost
// ─────────────────────────────────────────────────────────────────
dp[i][j] = (s1[i-1]==s2[j-1]) ? dp[i-1][j-1]
         : 1 + min({dp[i-1][j-1] /*replace*/, dp[i-1][j] /*delete*/, dp[i][j-1] /*insert*/});
// base: dp[i][0]=i, dp[0][j]=j

// ─────────────────────────────────────────────────────────────────
// 4. LONGEST PALINDROMIC SUBSEQUENCE — reduction
// ─────────────────────────────────────────────────────────────────
// LPS(s) = LCS(s, reverse(s))   — reuse template 1 verbatim

// ─────────────────────────────────────────────────────────────────
// 5. SHORTEST COMMON SUPERSEQUENCE — reduction + reconstruction
// ─────────────────────────────────────────────────────────────────
// length = |s1| + |s2| - LCS(s1,s2)
// reconstruct: on match, append once; on mismatch, append from the
// side that leads to the larger neighbor (both sides get appended
// across the whole walk — nothing is dropped, unlike plain LCS backtrack)

// ─────────────────────────────────────────────────────────────────
// 6. DISTINCT SUBSEQUENCES — counting DP
// ─────────────────────────────────────────────────────────────────
dp[i][j] = dp[i-1][j] + (s[i-1]==t[j-1] ? dp[i-1][j-1] : 0);
// base: dp[i][0] = 1 for all i;  dp[0][j] = 0 for j>0
// use long long / unsigned — counts explode combinatorially

// ─────────────────────────────────────────────────────────────────
// 7. MIN ASCII DELETE SUM — weighted LCS
// ─────────────────────────────────────────────────────────────────
dp[i][j] = (s1[i-1]==s2[j-1]) ? dp[i-1][j-1]
         : min(dp[i-1][j] + (int)s1[i-1], dp[i][j-1] + (int)s2[j-1]);

// ─────────────────────────────────────────────────────────────────
// 8. WILDCARD MATCHING — '*' = ANY sequence (standalone)
// ─────────────────────────────────────────────────────────────────
dp[i][j] = (p[j-1]=='*') ? (dp[i-1][j] || dp[i][j-1])
         : ((p[j-1]=='?' || s[i-1]==p[j-1]) && dp[i-1][j-1]);
// base row: dp[0][j] = dp[0][j-1] && p[j-1]=='*'  (leading stars only)

// ─────────────────────────────────────────────────────────────────
// 9. REGEX MATCHING — '*' = 0+ of PRECEDING char (modifier)
// ─────────────────────────────────────────────────────────────────
dp[i][j] = (p[j-1]=='*')
         ? (dp[i][j-2] || ((p[j-2]=='.'||s[i-1]==p[j-2]) && dp[i-1][j]))
         : ((p[j-1]=='.'||s[i-1]==p[j-1]) && dp[i-1][j-1]);
// base row: dp[0][j] = j>=2 && p[j-1]=='*' && dp[0][j-2]

// ─────────────────────────────────────────────────────────────────
// 10. LONGEST PALINDROMIC SUBSTRING — interval DP, fill by LENGTH
// ─────────────────────────────────────────────────────────────────
for (int len=1; len<=n; len++)
    for (int i=0; i+len-1<n; i++) {
        int j = i+len-1;
        dp[i][j] = (s[i]==s[j]) && (len<3 || dp[i+1][j-1]);
    }
// OR use expand-around-center for O(1) space: try both odd and even centers

// ─────────────────────────────────────────────────────────────────
// 11. MIN INSERTIONS FOR PALINDROME — reduction
// ─────────────────────────────────────────────────────────────────
// answer = n - LPS(s)

// ─────────────────────────────────────────────────────────────────
// 12. DECISION TABLE — WHICH TRANSITION FAMILY?
// ─────────────────────────────────────────────────────────────────
// Optimize (longest/shortest/min-cost): max/min on mismatch, diagonal+1 on match
// Count (how many ways):                sum (+) of disjoint cases on match
// Decide (true/false):                  || / && of sub-cases
// Substring (contiguous):               reset to 0 / false on mismatch — NOT max/carry-forward
```

---

## SECTION 22 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Wildcard Matching and Regular Expression Matching are the two that require the sharpest attention to `*` semantics — do NOT skip writing out the transition table by hand for both before coding either.

2. **Practice the "is this a disguise?" reflex.** After finishing LCS, immediately re-derive Longest Palindromic Subsequence and Shortest Common Supersequence FROM MEMORY as reductions, without looking at new code. If you can't derive the reduction in under two minutes, you haven't internalized it yet — re-read Section 6 and Section 7.

3. **Wildcard vs Regex drill:** Write both `isMatch` functions side by side on a blank page from memory. Circle the exact line where they diverge (the `*` transition). This is the single highest-yield drill in this pattern for interview reliability.

4. **Space optimization challenge:** After Edit Distance and LCS are solid, space-optimize both to O(n) using a rolling row plus a single `diag` variable (the same "save-before-overwrite" trick used for Maximal Square's diagonal term in Pattern 18). Then attempt LCS reconstruction — realize it CANNOT be space-optimized the same way, because backtracking needs the full table. This tension (space optimization vs. reconstruction) is a common senior-level follow-up.

5. **Preview Pattern 21 (LIS) and Pattern 23 (Interval DP):** LIS can be computed as `LCS(a, sorted_unique(a))` — try deriving this reduction yourself before Pattern 21 introduces the dedicated O(n log n) patience-sorting technique. Longest Palindromic Substring's fill-by-length discipline previews the general interval DP framework in Pattern 23 (Matrix Chain Multiplication, Burst Balloons).

---

## SECTION 23 — SIGN-OFF CRITERIA

### Tier 1 — LCS core mastered
You solve LCS (LC 1143) and Edit Distance (LC 72) cleanly from a blank file, including correct `(m+1) x (n+1)` table sizing and base cases, in under 15 minutes combined. You can explain in one sentence why Edit Distance's mismatch branch has three terms while LCS's has two.

### Tier 2 — Reductions internalized
You solve Longest Palindromic Subsequence, Shortest Common Supersequence, and Minimum Insertion Steps to Make a Palindrome WITHOUT writing new DP code — by correctly identifying and applying the LCS/LPS reduction each time. You can state all three reduction formulas from memory: `LCS(s,rev(s))`, `m+n-LCS`, `n-LPS`.

### Tier 3 — Counting and weighted variants solid
You solve Distinct Subsequences (LC 115) and Minimum ASCII Delete Sum (LC 712) and can articulate, without hesitation, why the former ADDS on a match while the latter takes the diagonal value directly (free, no cost) on a match.

### Tier 4 — Pattern-matching mastery
You solve BOTH Wildcard Matching (LC 44) and Regular Expression Matching (LC 10), and — this is the mandatory bar — you can write both `*` transitions on a whiteboard, side by side, from memory, and correctly explain why they differ (standalone token vs. modifier of the preceding character). You solve Longest Palindromic Substring using BOTH expand-around-center and the interval DP formulation, and can explain why the DP must fill by increasing length.

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 1143 (LCS — the foundation), LC 72 (Edit Distance — the cost-based variant), and at least ONE of LC 44 / LC 10 with a correct, memory-accurate `*` transition explaining how it differs from the other. These represent the three non-negotiable conceptual pillars of this pattern.

---

*Pattern 20 Complete — LCS Family (Longest Common Subsequence)*
*Next: Pattern 21 — LIS Family (Longest Increasing Subsequence, Patience Sorting, O(n log n) Techniques)*
