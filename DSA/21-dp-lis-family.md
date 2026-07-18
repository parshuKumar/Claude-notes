# PATTERN 21 — LIS FAMILY (LONGEST INCREASING SUBSEQUENCE)
## Patience Sorting, Binary Search on Tails, Counting, and 2D Dominance DP
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**Why this pattern gates 1800+ candidates:**

Every candidate can write the O(n²) LIS solution: `dp[i]` = length of the longest increasing subsequence ending at index `i`, with `dp[i] = 1 + max(dp[j])` over all `j < i` with `nums[j] < nums[i]`. That is Pattern 21 at Tier 1 — solid, correct, and completely insufficient once `n` reaches `10^5`.

The line that separates a 1600 solver from a 2000+ solver is this:

> **The O(n log n) "patience sorting" trick does NOT compute `dp[i]` faster. It abandons `dp[i]` entirely and tracks a completely different quantity: the smallest possible tail value for every achievable subsequence length, updated online with binary search.**

This is a genuine leap, not an optimization of the same idea:

- The O(n²) DP asks: *"what is the best subsequence ENDING HERE?"*
- The O(n log n) method asks: *"for each length L, what is the smallest value that could possibly end a length-L increasing subsequence, using only what I've seen so far?"*

These two questions have the same final answer (the max length), but the second one lets you answer "can I extend length L?" in O(log n) instead of O(n), because the "smallest tail per length" array is *provably sorted* — enabling binary search.

**Three traps that separate candidates further:**

1. **`lower_bound` vs `upper_bound`.** Get this backwards and your "strictly increasing" solution silently becomes "non-decreasing" (or vice versa). This single character bug is the most common LIS interview failure.
2. **The tails array is NOT a valid subsequence.** Candidates who print `tails` as "the LIS" are wrong — `tails` only tracks lengths and minimal tail values, never a coherent path. Reconstruction needs parent pointers.
3. **LIS-shaped problems that are secretly 2D dominance DP** (Russian Doll Envelopes, Stacking Cuboids). The trick is to sort away one dimension so that a *correct tiebreak* forces the remaining dimension to behave like classic 1D LIS. Get the tiebreak wrong and you silently allow invalid "nesting."

**The key skill:** before coding any LIS variant, answer:
1. Do I need the LENGTH only, or do I need to RECONSTRUCT the actual subsequence?
2. Is the relation strictly increasing (`<`) or non-decreasing (`<=`)? This decides `lower_bound` vs `upper_bound`.
3. Is this actually a multi-dimensional dominance problem hiding behind a 1D LIS surface? If so, what sort + tiebreak reduces it to 1D?

If you can answer these three questions in the first two minutes, the rest of the code is mechanical.

---

## SECTION 2 — THE LIS TAXONOMY

### Type 1: Classic End-Based DP (O(n²))
State: `dp[i]` = length of longest increasing subsequence ENDING at index `i`
Transition: `dp[i] = 1 + max(dp[j])` for all `j < i` with `nums[j] < nums[i]`
Answer: `max(dp[i])` over all `i` (NOT `dp[n-1]`)
Problems: LIS (foundation), Russian Doll Envelopes (brute force fallback), Cuboids (this IS the final algorithm — see Type 5)

### Type 2: Patience Sorting / Binary Search on Tails (O(n log n))
State: `tails[k]` = smallest possible tail value of any increasing subsequence of length `k+1` found so far
Transition: for each `x`, binary search `tails` for the insertion point and either extend or replace
Answer: `tails.size()`
Problems: LIS, Increasing Triplet Subsequence (specialized to length 3), Russian Doll Envelopes (after sort), core building block for many variants

### Type 3: Reconstruction (parent pointers)
State: same as Type 1 or Type 2, PLUS a `parent[]` array recording which index extended which
Direction: build forward, walk backward from the best-length index to reconstruct
Problems: any "return the actual subsequence" variant

### Type 4: Counting Variants
State: `length[i]` = LIS length ending at `i`, `count[i]` = number of LIS of that length ending at `i`
Key subtlety: `count[i]` only accumulates from `j` where `length[j] + 1 == length[i]` (equal, not just any extension)
Problems: Number of LIS (LC 673)

### Type 5: 2D / 3D Dominance DP (sort-then-LIS)
State: sort objects along one or more dimensions with a careful tiebreak so that a valid "nesting"/"stacking" relation becomes a single increasing-sequence condition (or an O(n²) full-dominance DP when 3 dimensions are involved)
Problems: Russian Doll Envelopes (2D → 1D via tiebreak, then true LIS), Maximum Height by Stacking Cuboids (3D → O(n²) dominance DP, NOT reducible to O(n log n) LIS)

### Type 6: Difference-Keyed DP (hashmap replaces binary search)
State: `dp[i][diff]` = length of the longest arithmetic subsequence ending at `i` with common difference `diff`
Why not O(n log n): the "key" is not a single sortable value but a pair (index, difference) — there's no monotonic tail array here, so we fall back to O(n²) with a hashmap per index
Problems: Longest Arithmetic Subsequence (LC 1027)

### Type 7: Alphabet/Bucket-Keyed DP (bounded value LIS)
State: `dp[c]` = longest "ideal" subsequence ending in character `c`, for `c` in a small fixed alphabet (26 letters)
Key insight: because the alphabet is tiny and fixed, we don't need binary search OR an O(n²) nested loop over indices — we do O(n × alphabetSize) by keeping one dp slot per possible last-character value
Problems: Longest Ideal Subsequence (LC 2370)

### Type 8: LIS + Repair DP (mutate elements to force strict increase)
State: `dp[value]` = minimum number of replacement operations needed so far, keyed by the value of the last element used, where replacement values come from a second sorted array
Problems: Make Array Strictly Increasing (LC 1187), Minimum Operations to Make Array Increasing (LC 1827 — simpler single-array variant)

---

## SECTION 3 — TEMPLATE 1: O(n²) LIS (FOUNDATION)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 300 — Longest Increasing Subsequence (O(n²) baseline)
//
// State:      dp[i] = length of the longest strictly increasing
//                      subsequence that ENDS at index i (nums[i] is
//                      the last element of that subsequence)
// Transition: dp[i] = 1 + max(dp[j]) for all j < i with nums[j] < nums[i]
//             (if no such j exists, dp[i] = 1 — the element alone)
// Base case:  dp[i] = 1 for all i (every element is a subsequence of length 1)
// Answer:     max(dp[i]) over ALL i — the LIS does not have to end at n-1
// ─────────────────────────────────────────────────────────────────

int lengthOfLIS_On2(vector<int>& nums) {
    int n = nums.size();
    if (n == 0) return 0;

    vector<int> dp(n, 1);   // every single element is a subsequence of length 1
    int ans = 1;

    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {           // strictly increasing condition
                dp[i] = max(dp[i], dp[j] + 1);
            }
        }
        ans = max(ans, dp[i]);   // track global max — LIS may end anywhere
    }
    return ans;
}
```

**FULL TRACE for nums = [10, 9, 2, 5, 3, 7, 101, 18]:**
```
dp[0] = 1                                          (just "10")

i=1 (9):  j=0: 10<9? no.                            dp[1]=1
i=2 (2):  j=0: 10<2? no.  j=1: 9<2? no.              dp[2]=1
i=3 (5):  j=0: no. j=1: no. j=2: 2<5 yes → dp[2]+1=2 dp[3]=2
i=4 (3):  j=2: 2<3 yes → 2. j=3: 5<3? no.            dp[4]=2
i=5 (7):  j=2: 2<7 → 2. j=3: 5<7 → dp[3]+1=3.
          j=4: 3<7 → dp[4]+1=3.                      dp[5]=3
i=6 (101): all of 10,9,2,5,3,7 are < 101.
           best predecessor is dp[5]=3 → 3+1=4       dp[6]=4
i=7 (18): candidates with nums[j]<18: 10,9,2,5,3,7 (not 101)
          best predecessor is dp[5]=3 → 3+1=4        dp[7]=4

dp = [1, 1, 1, 2, 2, 3, 4, 4]
ans = max(dp) = 4
LIS (one valid instance): [2, 3, 7, 101] or [2, 5, 7, 101] — length 4 ✓
```

**Complexity:** O(n²) time, O(n) space. This is your safety net — always write this first if you're unsure, verify against small cases, THEN upgrade to O(n log n).

---

## SECTION 4 — TEMPLATE 2: O(n log n) LIS VIA BINARY SEARCH ON TAILS

This is the core skill of Pattern 21. Read the proof carefully — it is asked in nearly every strong LIS interview.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 300 — Longest Increasing Subsequence (O(n log n) — patience sorting)
//
// State:      tails[k] = the SMALLEST possible value that can end an
//                         increasing subsequence of length (k+1),
//                         considering only elements processed so far
// Invariant:  tails is always sorted in strictly increasing order
// Transition: for each new value x, binary search tails for the first
//             position pos with tails[pos] >= x (lower_bound):
//               - if pos == tails.size(): x extends the longest chain
//                 found so far → push_back(x)
//               - else: x can end a BETTER (smaller-tailed) subsequence
//                 of length (pos+1) → overwrite tails[pos] = x
// Answer:     tails.size()  (NOT the contents of tails!)
// ─────────────────────────────────────────────────────────────────

int lengthOfLIS(vector<int>& nums) {
    vector<int> tails;   // tails[k] = min tail value for length k+1

    for (int x : nums) {
        // first index with tails[idx] >= x  (STRICTLY increasing LIS → lower_bound)
        auto it = lower_bound(tails.begin(), tails.end(), x);

        if (it == tails.end()) {
            tails.push_back(x);   // x is bigger than every tail → new longest length
        } else {
            *it = x;               // x improves (shrinks) an existing tail
        }
    }

    return tails.size();
}
```

### WHY the tails array is always sorted (invariant proof)

Claim: at every point in the loop, `tails` is strictly increasing.

Base case: `tails` starts empty — trivially sorted.

Inductive step: suppose `tails` is sorted before processing `x`. Let `pos = lower_bound(tails, x)` — the first index with `tails[pos] >= x`. Two cases:

- **`pos == tails.size()`:** every existing value is `< x`, so appending `x` at the end keeps the array sorted (new last element is bigger than the old last element, since old last `< x`).
- **`pos < tails.size()`:** we know `tails[pos-1] < x` (else `pos` would not be the FIRST index `>= x`) and `tails[pos] >= x`. We overwrite `tails[pos] = x`. Now `tails[pos-1] < x = tails[pos]` (still sorted on the left) and `x <= old tails[pos] <= tails[pos+1]` (still sorted on the right, since `x` only shrinks the value, never grows it).

So the invariant is preserved in both cases. QED.

### WHY the binary search position is semantically correct

Claim: at every point, `tails[k]` equals the smallest possible tail value achievable by SOME real increasing subsequence of length `k+1`, built only from elements processed so far.

- **Case `pos == tails.size()` (append):** Since `tails` is sorted and `pos` is past the end, `x` is strictly greater than `tails.back()` (the smallest known tail for the current max length `L = tails.size()`). Since there EXISTS a real subsequence of length `L` ending at value `tails.back() < x`, we can legally append `x` to it, forming a real subsequence of length `L+1` ending at `x`. Because `x` is the value we just discovered, and no shorter processing could produce an even smaller tail for length `L+1` at this point in the scan, `x` is exactly the smallest known tail for length `L+1`. Push it.

- **Case `pos < tails.size()` (replace):** We have `tails[pos-1] < x <= tails[pos]` (or `pos == 0`, trivial). By the invariant, there is a real subsequence of length `pos` ending at value `tails[pos-1]`. Appending `x` to it (legal since `tails[pos-1] < x`) produces a real subsequence of length `pos+1` ending at `x`. Since `x <= tails[pos]` (the previously smallest known tail of length `pos+1`), `x` is now AT LEAST AS GOOD a tail for length `pos+1`. Overwriting is safe and strictly improves (or preserves) optimality for all future extensions.

- **Why we never need to shrink `tails.size()`:** replacing a value never removes a subsequence — it just finds an EQUALLY long subsequence with a smaller (more extendable) tail. The true LIS length can only be discovered by growth (the append case), and every prefix processed so far has `tails.size()` exactly equal to the true LIS length restricted to that prefix — by induction, since every append is justified by a real constructible subsequence (shown above), and no length is ever overcounted (each `tails[k]` requires demonstrating an actual chain of length `k+1`, which we just did).

### FULL TRACE for nums = [10, 9, 2, 5, 3, 7, 101, 18]

```
tails = []

x=10: lower_bound([], 10) = end (empty)      → push.     tails = [10]
x=9:  lower_bound([10], 9) = index 0 (10>=9) → replace.  tails = [9]
x=2:  lower_bound([9], 2)  = index 0 (9>=2)  → replace.  tails = [2]
x=5:  lower_bound([2], 5)  = end (2<5)       → push.     tails = [2, 5]
x=3:  lower_bound([2,5], 3) = index 1 (5>=3) → replace.  tails = [2, 3]
x=7:  lower_bound([2,3], 7) = end            → push.     tails = [2, 3, 7]
x=101: lower_bound([2,3,7], 101) = end       → push.     tails = [2, 3, 7, 101]
x=18: lower_bound([2,3,7,101], 18) = index 3 (101>=18) → replace.
      tails = [2, 3, 7, 18]

Final tails.size() = 4  ✓ (matches the O(n²) answer)
```

Note that `tails = [2, 3, 7, 18]` is NOT the actual LIS found in Section 3 (`[2, 3, 7, 101]` or `[2, 5, 7, 101]`) — it just happens to also be a valid length-4 increasing subsequence here by coincidence of this example. In general, **the tails array is not a valid subsequence of the input at all** — its entries can come from positions that could never appear together in one real increasing run. Treat it purely as a length/bound tracker.

**Complexity:** O(n log n) time (n binary searches, each O(log n)), O(n) space.

---

## SECTION 5 — TEMPLATE 3: LIS RECONSTRUCTION (PARENT POINTERS)

### Simple version — O(n²) with parent[]

```cpp
// Reconstruct an actual LIS (not just its length) using the O(n²) DP.
// parent[i] = index of the element that directly precedes nums[i]
//             in the best subsequence ending at i (-1 if none).

vector<int> reconstructLIS_On2(vector<int>& nums) {
    int n = nums.size();
    vector<int> dp(n, 1), parent(n, -1);
    int bestLen = 1, bestIdx = 0;

    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i] && dp[j] + 1 > dp[i]) {
                dp[i] = dp[j] + 1;
                parent[i] = j;      // record who we extended from
            }
        }
        if (dp[i] > bestLen) { bestLen = dp[i]; bestIdx = i; }
    }

    vector<int> seq;
    for (int cur = bestIdx; cur != -1; cur = parent[cur])
        seq.push_back(nums[cur]);
    reverse(seq.begin(), seq.end());
    return seq;
}
```

### Fast version — O(n log n) with parent[] alongside tails

The trick: keep a SEPARATE array `tailsIdx` that stores the ORIGINAL INDEX of each tail value (not just the value). Every time we place an element at position `pos` in `tailsIdx`, its parent is `tailsIdx[pos-1]` — the index currently occupying the previous slot.

```cpp
vector<int> reconstructLIS_fast(vector<int>& nums) {
    int n = nums.size();
    vector<int> tailsVal;        // sorted values, same role as before
    vector<int> tailsIdx;        // tailsIdx[k] = original index of that tail value
    vector<int> parent(n, -1);   // parent[i] = index that precedes i in ITS chain

    for (int i = 0; i < n; i++) {
        int x = nums[i];
        auto it = lower_bound(tailsVal.begin(), tailsVal.end(), x);
        int pos = it - tailsVal.begin();

        // Whoever occupies slot (pos-1) right now is our predecessor,
        // because tailsIdx[pos-1] ends a valid chain of length pos
        // with a value strictly less than x.
        if (pos > 0) parent[i] = tailsIdx[pos - 1];

        if (it == tailsVal.end()) {
            tailsVal.push_back(x);
            tailsIdx.push_back(i);
        } else {
            *it = x;
            tailsIdx[pos] = i;
        }
    }

    vector<int> seq;
    for (int cur = tailsIdx.back(); cur != -1; cur = parent[cur])
        seq.push_back(nums[cur]);
    reverse(seq.begin(), seq.end());
    return seq;
}
```

**FULL TRACE for nums = [10, 9, 2, 5, 3, 7, 101, 18]:**
```
i=0 x=10: pos=0 (tailsVal empty) → parent[0]=-1.  push.  tailsVal=[10]  tailsIdx=[0]
i=1 x=9:  pos=0 (lower_bound on [10]) → parent[1]=-1.  replace idx0.
          tailsVal=[9]   tailsIdx=[1]
i=2 x=2:  pos=0 → parent[2]=-1.  replace idx0.
          tailsVal=[2]   tailsIdx=[2]
i=3 x=5:  pos=1 (end, 2<5) → parent[3]=tailsIdx[0]=2.  push.
          tailsVal=[2,5]  tailsIdx=[2,3]
i=4 x=3:  pos=1 (lower_bound on [2,5] for 3) → parent[4]=tailsIdx[0]=2.  replace idx1.
          tailsVal=[2,3]  tailsIdx=[2,4]
i=5 x=7:  pos=2 (end) → parent[5]=tailsIdx[1]=4.  push.
          tailsVal=[2,3,7]  tailsIdx=[2,4,5]
i=6 x=101: pos=3 (end) → parent[6]=tailsIdx[2]=5.  push.
          tailsVal=[2,3,7,101]  tailsIdx=[2,4,5,6]
i=7 x=18: pos=3 (lower_bound on [2,3,7,101] for 18) → parent[7]=tailsIdx[2]=5.  replace idx3.
          tailsVal=[2,3,7,18]  tailsIdx=[2,4,5,7]

Walk back from tailsIdx.back() = 7:
  7 (nums=18) → parent[7]=5 (nums=7) → parent[5]=4 (nums=3) → parent[4]=2 (nums=2) → parent[2]=-1

seq reversed = [2, 3, 7, 18]  — length 4, strictly increasing, a VALID
subsequence of the original array (indices 2,4,5,7). ✓
```

**Why this works:** `tailsIdx[pos-1]` is guaranteed, at the moment of use, to end a real chain of length `pos` with value `< x` — that's exactly the invariant proven in Section 4. So chaining `parent` pointers through the historical occupants of each slot always produces a valid path, even though the CURRENT contents of `tailsVal`/`tailsIdx` get overwritten later.

---

## SECTION 6 — TEMPLATE 4: NUMBER OF LIS (LC 673) — COUNTING VARIANT

This is the trickiest "core" problem in the pattern. The subtlety: you need TWO parallel arrays, and the count only propagates when lengths tie exactly.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 673 — Number of Longest Increasing Subsequence
//
// State:  length[i] = length of the LIS ending at index i
//         count[i]  = NUMBER OF distinct LIS of that length ending at i
//
// Transition (for each j < i with nums[j] < nums[i]):
//   if length[j] + 1  > length[i]:
//       // j opens a strictly LONGER chain — RESET, don't accumulate
//       length[i] = length[j] + 1
//       count[i]  = count[j]
//   else if length[j] + 1 == length[i]:
//       // j offers an ALTERNATE way to reach the SAME best length — accumulate
//       count[i] += count[j]
//   // if length[j] + 1 < length[i]: j is irrelevant, ignore
//
// Answer: sum of count[i] over all i where length[i] == global max length
// ─────────────────────────────────────────────────────────────────

int findNumberOfLIS(vector<int>& nums) {
    int n = nums.size();
    if (n == 0) return 0;

    vector<int> length(n, 1), count(n, 1);
    int maxLen = 1;

    for (int i = 0; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                if (length[j] + 1 > length[i]) {
                    length[i] = length[j] + 1;
                    count[i] = count[j];       // RESET — discard old count
                } else if (length[j] + 1 == length[i]) {
                    count[i] += count[j];      // ACCUMULATE — tie in length
                }
                // else: length[j]+1 < length[i] → j can't help, skip
            }
        }
        maxLen = max(maxLen, length[i]);
    }

    int result = 0;
    for (int i = 0; i < n; i++)
        if (length[i] == maxLen) result += count[i];
    return result;
}
```

### THE TRICKY PART, explained

If you write `count[i] += count[j]` unconditionally whenever `nums[j] < nums[i]`, you will overcount, because `j` might only support a SHORTER chain than the current best `length[i]` — its count is irrelevant noise that doesn't correspond to any maximum-length path through `i`.

Symmetrically, if you write `count[i] = count[j]` (overwrite) even when `length[j] + 1 == length[i]` (a tie), you will UNDERCOUNT, because you'd be discarding the count contributed by a different, equally-valid predecessor found earlier in the `j` loop.

The fix is the three-way branch: **strictly longer → reset and copy; exactly tied → accumulate; shorter → ignore.**

**FULL TRACE for nums = [1, 3, 5, 4, 7]** (LC 673's own example, answer = 2):
```
length = [1,1,1,1,1]   count = [1,1,1,1,1]   initially

i=0 (val=1): no j to compare.                       length[0]=1  count[0]=1

i=1 (val=3): j=0: nums[0]=1<3.
             length[0]+1=2 > length[1]=1 → RESET.    length[1]=2  count[1]=count[0]=1

i=2 (val=5): j=0: 1<5. length[0]+1=2 > length[2]=1 → RESET. length[2]=2 count[2]=1
             j=1: 3<5. length[1]+1=3 > length[2]=2 → RESET. length[2]=3 count[2]=count[1]=1

i=3 (val=4): j=0: 1<4. length[0]+1=2 > length[3]=1 → RESET. length[3]=2 count[3]=1
             j=1: 3<4. length[1]+1=3 > length[3]=2 → RESET. length[3]=3 count[3]=count[1]=1
             j=2: 5<4? no — skip.

i=4 (val=7): j=0: 1<7. length[0]+1=2 > length[4]=1 → RESET. length[4]=2 count[4]=1
             j=1: 3<7. length[1]+1=3 > length[4]=2 → RESET. length[4]=3 count[4]=count[1]=1
             j=2: 5<7. length[2]+1=4 > length[4]=3 → RESET. length[4]=4 count[4]=count[2]=1
             j=3: 4<7. length[3]+1=4 == length[4]=4 → ACCUMULATE.
                       count[4] += count[3] = 1 + 1 = 2

Final:  length = [1, 2, 3, 3, 4]     count = [1, 1, 1, 1, 2]
maxLen = 4.  Sum of count[i] where length[i]==4:  only i=4 → count[4]=2.

Answer = 2 ✓
(The two LIS of length 4 are [1,3,5,7] and [1,3,4,7] — exactly what count[4]=2 captures.)
```

**Why O(n log n) does NOT trivially extend here:** the tails array only tracks ONE minimal tail per length — it throws away all the other ways to reach that length. Counting requires knowing about EVERY predecessor that achieves a tied optimal length, which the collapsed tails representation destroys. A true O(n log n) counting solution exists using Fenwick trees/segment trees keyed by value (see Pattern Connections), but the O(n²) two-array method is the expected interview solution.

---

## SECTION 7 — TEMPLATE 5: RUSSIAN DOLL ENVELOPES (LC 354) — 2D → 1D REDUCTION

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 354 — Russian Doll Envelopes
// envelopes[i] = [width_i, height_i]. Envelope A fits inside envelope B
// iff width_A < width_B AND height_A < height_B (STRICT in both).
// Find the maximum number of envelopes you can nest (Russian doll chain).
//
// STRATEGY: sort by width ASCENDING; for envelopes with EQUAL width,
// sort by height DESCENDING. Then run standard O(n log n) LIS on the
// height sequence alone.
// ─────────────────────────────────────────────────────────────────

int maxEnvelopes(vector<vector<int>>& envelopes) {
    sort(envelopes.begin(), envelopes.end(),
         [](const vector<int>& a, const vector<int>& b) {
             if (a[0] == b[0]) return a[1] > b[1];   // same width → height DESCENDING
             return a[0] < b[0];                      // width ASCENDING
         });

    vector<int> tails;   // classic LIS on heights only
    for (auto& e : envelopes) {
        int h = e[1];
        auto it = lower_bound(tails.begin(), tails.end(), h);
        if (it == tails.end()) tails.push_back(h);
        else *it = h;
    }
    return tails.size();
}
```

### WHY the height-descending tiebreak is mandatory

After sorting by width ascending, running plain LIS on the height sequence works ONLY IF two envelopes sharing the same width can never both appear in the resulting increasing-height subsequence. Nesting requires STRICTLY greater width AND height — two envelopes with the SAME width can never nest inside each other, no matter their heights.

- **If we sorted same-width groups by height ASCENDING:** within a width-tied block, heights would appear in increasing order. The LIS algorithm on heights would happily treat consecutive elements of that block as a valid increasing run (since it only looks at height values, blind to width), incorrectly "nesting" two same-width envelopes. This inflates the answer.

- **If we sort same-width groups by height DESCENDING:** within a width-tied block, heights appear in *decreasing* order as we scan left to right. The LIS-on-heights algorithm can therefore select AT MOST ONE envelope from each width-tied block into any increasing run (since a decreasing run inside the block can't contribute more than one element to an increasing subsequence). This forces the "true nesting" semantics: to include two envelopes from the same block, you'd need a run with two DIFFERENT heights (impossible for a chain built on equal widths), so the algorithm naturally rejects that.

**FULL TRACE for envelopes = [[5,4],[6,4],[6,7],[2,3]]:**
```
Sort by width asc, tie → height desc:
  width 2: [2,3]
  width 5: [5,4]
  width 6: two entries (6,4) and (6,7) → sort desc by height → (6,7) THEN (6,4)

Sorted order: [[2,3], [5,4], [6,7], [6,4]]
Height sequence extracted: [3, 4, 7, 4]

Run LIS on [3, 4, 7, 4]:
  tails = []
  h=3: push.                              tails = [3]
  h=4: lower_bound([3],4)=end → push.     tails = [3, 4]
  h=7: lower_bound([3,4],7)=end → push.   tails = [3, 4, 7]
  h=4: lower_bound([3,4,7],4)=index1 (value 4 >= 4) → replace tails[1]=4 (no visible change)
                                           tails = [3, 4, 7]

tails.size() = 3  ✓ (matches the known LC 354 answer of 3:
  [2,3] → [5,4] → [6,7])
```

Note how the second `(6,4)` — appearing right after `(6,7)` in sorted order — was correctly PREVENTED from extending the chain to length 4: `lower_bound` found it belongs at the SAME slot as the existing `4`, not past `7`. This is the tiebreak doing its job: two same-width envelopes cannot both be "nested."

**Complexity:** O(n log n) — O(n log n) sort + O(n log n) LIS.

---

## SECTION 8 — TEMPLATE 6: LONGEST ARITHMETIC SUBSEQUENCE (LC 1027)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1027 — Longest Arithmetic Subsequence
// Find the length of the longest subsequence that forms an arithmetic
// progression (constant difference between consecutive elements).
//
// WHY this is NOT reducible to O(n log n) LIS: "increasing" here means
// "matches a SPECIFIC difference," not "is numerically larger." There
// is no single sortable tail value — the state must be keyed by BOTH
// the ending index AND the difference used to reach it.
//
// State: dp[i][diff] = length of the longest arithmetic subsequence
//                       ENDING at index i with common difference `diff`
// Transition: for j < i, diff = nums[i]-nums[j]:
//   dp[i][diff] = (dp[j].count(diff) ? dp[j][diff] : 1) + 1
//                 // if j itself continues a chain with this diff, extend it;
//                 // otherwise (i,j) alone forms a length-2 AP — extend from 1
// Answer: max over all dp[i][diff]
// ─────────────────────────────────────────────────────────────────

int longestArithSeqLength(vector<int>& nums) {
    int n = nums.size();
    vector<unordered_map<int,int>> dp(n);   // dp[i][diff] -> length
    int ans = 1;

    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            int diff = nums[i] - nums[j];
            int prevLen = dp[j].count(diff) ? dp[j][diff] : 1;
            dp[i][diff] = max(dp[i][diff], prevLen + 1);
            ans = max(ans, dp[i][diff]);
        }
    }
    return ans;
}
```

**FULL TRACE for nums = [3, 6, 9, 12]:**
```
dp[0] = {}

i=1, j=0: diff=6-3=3.  prevLen = dp[0] has no key 3 → 1.
          dp[1][3] = max(0, 1+1) = 2.   ans=2

i=2, j=0: diff=9-3=6.  prevLen=1 (dp[0] empty).  dp[2][6]=2.   ans=2
i=2, j=1: diff=9-6=3.  dp[1] HAS key 3 → prevLen=dp[1][3]=2.
          dp[2][3] = max(0, 2+1) = 3.   ans=3

i=3, j=0: diff=12-3=9.  prevLen=1.  dp[3][9]=2
i=3, j=1: diff=12-6=6.  dp[1] has no key 6 → prevLen=1.  dp[3][6]=2
i=3, j=2: diff=12-9=3.  dp[2] HAS key 3 → prevLen=dp[2][3]=3.
          dp[3][3] = max(0, 3+1) = 4.   ans=4

Final answer = 4  (the entire array [3,6,9,12] is one AP with diff=3) ✓
```

**Complexity:** O(n²) time (n² pairs, each an O(1) amortized hashmap op), O(n²) space worst case (each `dp[i]` can hold up to `i` distinct differences).

---

## SECTION 9 — TEMPLATE 7: MAKE ARRAY STRICTLY INCREASING (LC 1187)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1187 — Make Array Strictly Increasing
// You may replace any element of arr1 with any element of arr2 (each
// arr2 element usable any number of times, but each replacement costs
// 1 operation). Find the MINIMUM number of replacements needed to make
// arr1 strictly increasing, or -1 if impossible.
//
// This is "LIS logic in reverse": instead of finding what to KEEP, we
// track, for every possible "value of the previous element," the
// minimum operations spent so far. At each new arr1 element we have
// TWO choices: keep it (if valid) or replace it with the cheapest
// arr2 candidate greater than the previous element.
//
// State: dp[prevValue] = minimum operations used so far, GIVEN that
//                         the last placed value is `prevValue`
// Transition per element x = arr1[i], for every (prev, ops) in dp:
//   Option 1 (KEEP):    if x > prev:  candidate (x, ops)
//   Option 2 (REPLACE): let r = smallest arr2 value > prev (if any):
//                        candidate (r, ops+1)
//   ndp[key] = min over all candidates landing on that key
// Answer: min value in the final dp map (after processing all elements)
// ─────────────────────────────────────────────────────────────────

int makeArrayIncreasing(vector<int>& arr1, vector<int>& arr2) {
    sort(arr2.begin(), arr2.end());
    arr2.erase(unique(arr2.begin(), arr2.end()), arr2.end());

    map<long long, int> dp;                 // prevValue -> min ops
    dp[LLONG_MIN] = 0;                       // sentinel: nothing placed yet

    auto relax = [](map<long long,int>& m, long long key, int val) {
        auto it = m.find(key);
        if (it == m.end() || it->second > val) m[key] = val;
    };

    for (int x : arr1) {
        map<long long, int> ndp;

        for (auto& [prev, ops] : dp) {
            // Option 1: keep x unchanged
            if (x > prev) relax(ndp, x, ops);

            // Option 2: replace x with the smallest arr2 value strictly > prev
            auto it2 = upper_bound(arr2.begin(), arr2.end(), prev);
            if (it2 != arr2.end()) {
                relax(ndp, *it2, ops + 1);
            }
        }

        dp = ndp;
        if (dp.empty()) return -1;   // no valid choice extends any state — dead end
    }

    int ans = INT_MAX;
    for (auto& [v, ops] : dp) ans = min(ans, ops);
    return ans;
}
```

**FULL TRACE for arr1 = [1,5,3,6,7], arr2 = [1,2,3,4]** (arr2 already sorted+deduped):
```
dp = { LLONG_MIN: 0 }

x=1:  (prev=LLONG_MIN, ops=0)
  keep: 1 > LLONG_MIN → ndp[1] = 0
  replace: upper_bound(arr2, LLONG_MIN) = arr2[0] = 1 → relax ndp[1] with ops+1=1
           ndp[1] already 0 (better) → stays 0
  ndp = {1: 0}

x=5:  (prev=1, ops=0)
  keep: 5 > 1 → ndp[5] = 0
  replace: upper_bound(arr2, 1) = 2 → relax ndp[2] with 1
  ndp = {2: 1, 5: 0}

x=3:  (prev=2, ops=1):
    keep: 3 > 2 → ndp[3] = 1
    replace: upper_bound(arr2, 2) = 3 → relax ndp[3] with ops+1=2 (worse, ignored)
  (prev=5, ops=0):
    keep: 3 > 5? no.
    replace: upper_bound(arr2, 5) = end (no arr2 value > 5) → no candidate
  ndp = {3: 1}

x=6:  (prev=3, ops=1):
    keep: 6 > 3 → ndp[6] = 1
    replace: upper_bound(arr2, 3) = 4 → relax ndp[4] with 2
  ndp = {4: 2, 6: 1}

x=7:  (prev=4, ops=2):
    keep: 7 > 4 → ndp[7] = 2
    replace: upper_bound(arr2, 4) = end → no candidate
  (prev=6, ops=1):
    keep: 7 > 6 → relax ndp[7] with 1  (better than 2) → ndp[7] = 1
    replace: upper_bound(arr2, 6) = end → no candidate
  ndp = {7: 1}

Final dp = {7: 1}.  Answer = 1  ✓ (matches LC 1187 example 1's expected output of 1)
```

**Connection to LIS:** this is structurally the SAME "keyed-by-last-value, minimize a cost" pattern as Section 4's tails array — except here the state carries an explicit cost (`ops`) instead of implicit length, and there are TWO transition options per step instead of one binary-search step. It's a strict generalization of the LIS idea.

---

## SECTION 10 — TEMPLATE 8: LONGEST IDEAL SUBSEQUENCE (LC 2370)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 2370 — Longest Ideal Subsequence
// Given string s and integer k, an "ideal" subsequence has, for every
// pair of ADJACENT characters in the subsequence, an absolute alphabet
// distance <= k. Find the length of the longest ideal subsequence.
//
// KEY INSIGHT: the "predecessor condition" (|c1 - c2| <= k) does NOT
// depend on POSITION at all — only on the CHARACTER VALUE. Since the
// alphabet has only 26 letters, we don't need an O(n²) loop over prior
// INDICES, nor a sorted tails array — we keep one dp slot PER LETTER
// and scan a small fixed window [c-k, c+k] over that 26-slot array.
//
// State: dp[c] = length of the longest ideal subsequence found so far
//                that ENDS in character c (c = 0..25)
// Transition, for each character ch = s[i], c = ch - 'a':
//   dp[c] = 1 + max(dp[p]) for p in [max(0,c-k), min(25,c+k)]
// Answer: max(dp[0..25])
// ─────────────────────────────────────────────────────────────────

int longestIdealString(string s, int k) {
    vector<int> dp(26, 0);
    int ans = 0;

    for (char ch : s) {
        int c = ch - 'a';
        int best = 0;
        for (int prev = max(0, c - k); prev <= min(25, c + k); prev++)
            best = max(best, dp[prev]);
        dp[c] = best + 1;
        ans = max(ans, dp[c]);
    }
    return ans;
}
```

**FULL TRACE for s = "acfgqj", k = 4:**
```
dp[0..25] all start at 0.

'a' (c=0):  window [max(0,-4), min(25,4)] = [0,4].  max(dp[0..4]) = 0.
            dp[0] = 1.                                          ans=1
'c' (c=2):  window [0,6].  max(dp[0..6]) = dp[0] = 1.
            dp[2] = 2.                                          ans=2
'f' (c=5):  window [1,9].  max(dp[1..9]) = dp[2] = 2.
            dp[5] = 3.                                          ans=3
'g' (c=6):  window [2,10]. max(dp[2..10]) = dp[5] = 3.
            dp[6] = 4.                                          ans=4
'q' (c=16): window [12,20]. max(dp[12..20]) = 0 (all untouched).
            dp[16] = 1.                                         ans stays 4
'j' (c=9):  window [5,13]. max(dp[5..13]) = dp[6] = 4.
            dp[9] = 5.                                          ans=5

Final answer = 5.
Verification: the subsequence "a","c","f","g","j" (skip 'q') has
consecutive alphabet gaps |a-c|=2, |c-f|=3, |f-g|=1, |g-j|=3 —
all <= k=4. A valid ideal subsequence of length 5, confirming dp[9]=5.
```

**Complexity:** O(n × 26) time, O(26) space — the tiny fixed alphabet is what makes this tractable without sorting or binary search; it's a bucketed variant of the LIS "keyed by last value" idea, where the key space is small enough to scan directly instead of binary-searching.

---

## SECTION 11 — TEMPLATE 9: MAXIMUM HEIGHT BY STACKING CUBOIDS (LC 1691)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1691 — Maximum Height by Stacking Cuboids
// Each cuboid has 3 dimensions and may be freely ROTATED (any of its
// 3 dimensions can be "height," the other two become width/length).
// Cuboid A can be placed directly on top of cuboid B iff, after both
// are oriented, width_A <= width_B, length_A <= length_B, height_A <= height_B
// (non-strict — equal dimensions ARE allowed to stack, unlike Russian Doll).
// Maximize total stack height.
//
// STEP 1 (per-cuboid normalization): for EACH cuboid, sort its own 3
// dimensions ascending. This is a well-known provably-optimal choice:
// among all 6 orientations of a single cuboid, using the sorted
// [min, mid, max] orientation as (width, length, height) never hurts —
// it maximizes the chance of being placed on top of / underneath others,
// because it is the "most dominated / most dominating" arrangement
// simultaneously in the base-area sense.
//
// STEP 2 (global sort): sort ALL cuboids lexicographically ascending.
// This guarantees that if cuboid j can ever be a valid BASE for cuboid i,
// then j appears before i in the sorted order (a necessary, not
// sufficient, precondition — we still verify full 3D dominance).
//
// STEP 3: run an O(n²) DOMINANCE DP (this is the 3D generalization of
// LIS — NOT reducible to O(n log n) because dominance in 3 dimensions
// has no total order / no single sortable key).
//
// State: dp[i] = max total height of a stack with cuboid i ON TOP
// Transition: dp[i] = cuboids[i][2] + max(dp[j]) for all j < i where
//             cuboids[j] dominates-or-equals cuboids[i] in ALL 3 dims
// Answer: max(dp[i])
// ─────────────────────────────────────────────────────────────────

int maxHeight(vector<vector<int>>& cuboids) {
    for (auto& c : cuboids) sort(c.begin(), c.end());   // normalize each cuboid
    sort(cuboids.begin(), cuboids.end());                // lexicographic global sort

    int n = cuboids.size();
    vector<int> dp(n);
    int ans = 0;

    for (int i = 0; i < n; i++) {
        dp[i] = cuboids[i][2];   // base case: this cuboid alone (its own "height" dim)
        for (int j = 0; j < i; j++) {
            if (cuboids[j][0] <= cuboids[i][0] &&
                cuboids[j][1] <= cuboids[i][1] &&
                cuboids[j][2] <= cuboids[i][2]) {
                dp[i] = max(dp[i], dp[j] + cuboids[i][2]);
            }
        }
        ans = max(ans, dp[i]);
    }
    return ans;
}
```

**FULL TRACE for cuboids = [[50,45,20], [95,37,53], [45,23,12]]:**
```
STEP 1 — normalize each cuboid (sort its 3 dims ascending):
  [50,45,20] → [20,45,50]
  [95,37,53] → [37,53,95]
  [45,23,12] → [12,23,45]

STEP 2 — sort all cuboids lexicographically ascending:
  [12,23,45], [20,45,50], [37,53,95]

STEP 3 — O(n²) dominance DP:

dp[0]: cuboids[0]=[12,23,45].  base dp[0] = 45.

dp[1]: cuboids[1]=[20,45,50].  base dp[1] = 50.
  j=0: [12,23,45] <= [20,45,50] component-wise? 12<=20 ✓ 23<=45 ✓ 45<=50 ✓ → dominates.
       dp[1] = max(50, dp[0]+50) = max(50, 45+50) = 95

dp[2]: cuboids[2]=[37,53,95].  base dp[2] = 95.
  j=0: [12,23,45] <= [37,53,95]? 12<=37 ✓ 23<=53 ✓ 45<=95 ✓ → dominates.
       dp[2] = max(95, dp[0]+95) = max(95, 45+95) = 140
  j=1: [20,45,50] <= [37,53,95]? 20<=37 ✓ 45<=53 ✓ 50<=95 ✓ → dominates.
       dp[2] = max(140, dp[1]+95) = max(140, 95+95) = 190

Final dp = [45, 95, 190].  ans = 190  ✓ (matches the known LC 1691 answer of 190,
achieved by stacking all three cuboids: 45 + (95-45=50... ) — concretely,
height = 45 (bottom) + 50 (middle) + 95 (top) = 190.)
```

**WHY this is O(n²), not O(n log n):** LIS-style binary search relies on a SINGLE totally-ordered key so that "can I extend?" reduces to one comparison. Here, "dominance" requires ALL THREE dimensions to compare favorably simultaneously — there is no single number that captures 3D dominance, so binary search cannot prune the candidate predecessors. The lexicographic sort is only a NECESSARY filter (it guarantees we never need `j > i`), not a sufficient one — we must still verify all 3 coordinates explicitly for every pair.

---

## SECTION 12 — COMPLEXITY TABLE

| Problem | Time | Space | Technique | Notes |
|---------|------|-------|-----------|-------|
| LIS (O(n²)) | O(n²) | O(n) | End-based DP | Foundation, always correct fallback |
| LIS (O(n log n)) | O(n log n) | O(n) | Patience sorting + binary search | `tails` is not a real subsequence |
| LIS reconstruction | O(n log n) | O(n) | Tails + parent[] via historical index array | Real subsequence recoverable |
| Number of LIS | O(n²) | O(n) | length[] + count[] two-array DP | O(n log n) exists via Fenwick tree (advanced) |
| Russian Doll Envelopes | O(n log n) | O(n) | Sort (asc width, desc height tie) + LIS | Tiebreak is the entire trick |
| Longest Arithmetic Subsequence | O(n²) | O(n²) worst case | dp[i][diff] hashmap | No sortable single key — hashmap replaces binary search |
| Make Array Strictly Increasing | O(n log(A) · log(B)) | O(n) | Map keyed by prev value, 2 transitions | A = len(arr1), B = distinct len(arr2) |
| Longest Ideal Subsequence | O(n × 26) | O(26) | Bucketed dp[char] | Small fixed alphabet replaces binary search |
| Maximum Height Stacking Cuboids | O(n²) | O(n) | Sort + 3D dominance DP | Not reducible below O(n²) |
| Increasing Triplet Subsequence | O(n) | O(1) | Track 2 running minimums (LIS length capped at 3) | Specialized greedy, not general LIS |
| Maximum Length of Pair Chain | O(n log n) | O(1) | Greedy sort by end (interval scheduling) | NOT a DP — see Pattern Connections |

---

## SECTION 13 — VARIANTS

**Non-decreasing LIS (duplicates allowed):** replace `lower_bound` with `upper_bound` in the tails-array template. See Section 14, Mistake 1, for the exact reasoning and a full trace of the difference.

**Longest Decreasing Subsequence (LDS):** either (a) reverse the array and run LIS, or (b) negate every value and run LIS on the negated array. Both work because decreasing-in-original-order is equivalent to increasing-in-reversed-order or increasing-in-negated-values.

**Bitonic subsequence (increase then decrease):** compute `LIS_left[i]` = LIS ending at `i` scanning left-to-right, and `LDS_right[i]` = longest decreasing subsequence starting at `i` scanning right-to-left (equivalently, LIS on the reversed suffix). Answer for a peak at `i` is `LIS_left[i] + LDS_right[i] - 1` (subtract 1 to avoid double-counting `nums[i]`). Take the max over all `i`.

**Minimum number of decreasing subsequences to cover the array (patience sorting piles / Dilworth's theorem):** the number of PILES created by the patience-sorting card game (not the tails-array optimization, but the literal pile-building simulation) equals the LENGTH of the LIS. This is a direct instance of **Dilworth's theorem**: in any partial order, the minimum number of chains needed to cover a set equals the maximum antichain size. Here, "chains" = decreasing runs, "antichain" = increasing subsequence.

**Count of ALL increasing subsequences (not just longest, modulo some number):** a completely different DP — `dp[i]` = number of increasing subsequences ending at `i` = `1 + sum(dp[j])` for `nums[j] < nums[i]`. Do not confuse this with LC 673 (Number of LIS), which only counts subsequences achieving the MAXIMUM length.

**LIS with a Fenwick Tree / BIT (for Number of LIS in O(n log n)):** coordinate-compress the values, then maintain two Fenwick trees — one storing max length ending at each value, one storing the sum of counts for that max length — supporting O(log n) prefix-max and prefix-sum queries as you scan left to right. This upgrades Section 6 from O(n²) to O(n log n). See Pattern Connections.

---

## SECTION 14 — COMMON MISTAKES

### Mistake 1: `lower_bound` vs `upper_bound` — strictly increasing vs non-decreasing

```cpp
// WRONG — using lower_bound when duplicates should be ALLOWED
// (i.e., you want the longest NON-DECREASING subsequence, nums[j] <= nums[i])
auto it = lower_bound(tails.begin(), tails.end(), x);
// lower_bound finds the first tails[idx] >= x, which REPLACES equal values
// instead of extending past them — this silently computes STRICTLY
// increasing LIS even though you intended to allow ties.

// CORRECT for non-decreasing (duplicates allowed):
auto it = upper_bound(tails.begin(), tails.end(), x);
// upper_bound finds the first tails[idx] > x, so an EQUAL value x is
// treated as "extendable" (x can sit right after an equal tail), correctly
// allowing runs like [2, 2, 3] to count as a non-decreasing subsequence of length 3.

// CORRECT for strictly increasing (the LC 300 default):
auto it = lower_bound(tails.begin(), tails.end(), x);
// This is what Section 4 uses — verified correct above.
```
**Quick trace to see the difference, nums = [2, 2]:**
- With `lower_bound` (strict): x=2 → push (tails=[2]). x=2 → lower_bound finds index 0 (2>=2) → REPLACE. tails stays `[2]`. Length = 1 (correct: no strictly-increasing pair exists in `[2,2]`).
- With `upper_bound` (non-decreasing): x=2 → push (tails=[2]). x=2 → upper_bound finds index 1 (end, since no tail is `> 2`) → PUSH. tails becomes `[2,2]`. Length = 2 (correct: `[2,2]` IS a valid non-decreasing subsequence).

---

### Mistake 2: Wrong tiebreak in Russian Doll Envelopes

```cpp
// WRONG — sorting same-width envelopes by height ASCENDING
sort(envelopes.begin(), envelopes.end(),
     [](auto& a, auto& b) {
         if (a[0] == b[0]) return a[1] < b[1];   // BUG: ascending tiebreak
         return a[0] < b[0];
     });
// Two envelopes with the SAME width can now appear as an increasing
// run in the height-only LIS pass, falsely suggesting one nests inside
// the other — but equal width can NEVER nest (nesting requires STRICT
// inequality in both dimensions).

// CORRECT — same-width envelopes sorted by height DESCENDING
sort(envelopes.begin(), envelopes.end(),
     [](auto& a, auto& b) {
         if (a[0] == b[0]) return a[1] > b[1];   // descending tiebreak
         return a[0] < b[0];
     });
// Now same-width blocks appear in decreasing height order, so the
// height-only LIS pass can select AT MOST ONE envelope per width-tied
// block — correctly modeling "equal width never nests."
```

---

### Mistake 3: Count double-counting in Number of LIS

```cpp
// WRONG — accumulating count unconditionally whenever nums[j] < nums[i]
for (int j = 0; j < i; j++) {
    if (nums[j] < nums[i]) {
        length[i] = max(length[i], length[j] + 1);
        count[i] += count[j];   // BUG: adds count[j] even when j only
                                  // supports a SHORTER chain than the
                                  // eventual length[i] — pollutes the count
                                  // with subsequences that are NOT actually
                                  // of maximum length ending at i
    }
}

// CORRECT — the three-way branch from Section 6
for (int j = 0; j < i; j++) {
    if (nums[j] < nums[i]) {
        if (length[j] + 1 > length[i]) {
            length[i] = length[j] + 1;
            count[i] = count[j];        // RESET on strictly better length
        } else if (length[j] + 1 == length[i]) {
            count[i] += count[j];       // ACCUMULATE only on exact tie
        }
    }
}
```

---

### Mistake 4: Treating the `tails` array as the actual LIS

```cpp
// WRONG — printing tails as if it were a real subsequence
vector<int> tails;
for (int x : nums) { /* ... build tails ... */ }
return tails;   // BUG: tails values may never have appeared together
                 // in any single real increasing run of the input array!

// CORRECT — tails.size() gives the LENGTH only. To get the actual
// subsequence, use the parent-pointer reconstruction from Section 5
// (track tailsIdx alongside tailsVal, and walk parent[] backward).
```

---

### Mistake 5: Returning `dp[n-1]` instead of `max(dp)` in the O(n²) template

```cpp
// WRONG — assuming the LIS must end at the last array element
vector<int> dp(n, 1);
for (int i = 1; i < n; i++)
    for (int j = 0; j < i; j++)
        if (nums[j] < nums[i]) dp[i] = max(dp[i], dp[j]+1);
return dp[n-1];   // BUG: the true LIS may end at ANY index, not n-1
                   // e.g. nums=[5,4,3,2,1,100] — LIS=[1,100] doesn't
                   // end at the array's conceptual "peak" but dp[n-1]
                   // (index of value 100) happens to be right here by luck;
                   // change the array to nums=[9,1,2,3,10,0] and dp[n-1]
                   // (ending at value 0, dp=1) is very wrong vs the true
                   // answer 4 ([1,2,3,10]).

// CORRECT
int ans = 1;
for (int i = 0; i < n; i++) ans = max(ans, dp[i]);
return ans;
```

---

### Mistake 6: Forgetting per-object normalization before a dominance sort (Cuboids)

```cpp
// WRONG — sorting cuboids globally without first sorting EACH cuboid's
// own 3 dimensions ascending
sort(cuboids.begin(), cuboids.end());
// Without per-cuboid normalization, a cuboid physically identical to
// another (just listed in a different rotation, e.g. [20,45,50] vs
// [50,20,45]) will not compare correctly component-wise against other
// cuboids, and the O(n²) dominance check (cuboids[j][k] <= cuboids[i][k]
// for all k) will incorrectly reject valid stackings that require an
// implicit rotation.

// CORRECT — normalize FIRST, then sort globally
for (auto& c : cuboids) sort(c.begin(), c.end());   // fix orientation per cuboid
sort(cuboids.begin(), cuboids.end());                // THEN sort globally
```

---

## SECTION 15 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Longest Increasing Subsequence | 300 | O(n²) baseline, then O(n log n) tails + `lower_bound` |
| 2 | Longest String Chain | 1048 | LIS-shaped DP over a custom "predecessor" relation (word minus one char), sort by length first |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Number of Longest Increasing Subsequence | 673 | Two-array (length + count) DP, the reset-vs-accumulate branch |
| 4 | Increasing Triplet Subsequence | 334 | O(1) space, two running minimums — a capped-length LIS special case |
| 5 | Maximum Length of Pair Chain | 646 | Recognize this is GREEDY (sort by end), not DP — see Pattern Connections |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Russian Doll Envelopes | 354 | Sort tiebreak proof (width asc, height desc) + LIS |
| 7 | Longest Arithmetic Subsequence | 1027 | dp[i][diff] hashmap, no single sortable key |
| 8 | Make Array Strictly Increasing | 1187 | LIS + optional replacement DP, map keyed by previous value |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Minimum Operations to Make the Array Increasing | 1827 | Simpler single-array greedy repair — contrast with #8 |
| 10 | Longest Ideal Subsequence | 2370 | Bucketed dp[char], fixed small alphabet |
| 11 | Maximum Height by Stacking Cuboids | 1691 | Per-object normalization + global sort + O(n²) 3D dominance DP |

---

## SECTION 16 — PATTERN CONNECTIONS

1. **Pattern 20 (LCS Family):** LCS is a TWO-sequence 2D DP (`dp[i][j]` over prefixes of two strings), while LIS is a ONE-sequence DP (`dp[i]` over a single array). They are often confused because both produce "longest compatible chain" answers, but LIS additionally admits the O(n log n) binary-search speedup that LCS fundamentally cannot (LCS has no analogous monotonic-tail structure — two independent sequences don't collapse to a single sortable key). Interestingly, **LIS of the "match positions" array IS how you solve LCS of a permutation in O(n log n)** — a classic reduction worth knowing: if both strings are permutations of `1..n`, map each value to its position in the second string, then LCS(s1, s2) equals LIS of that position-mapped array.

2. **Pattern 27/28 (Segment Tree / Fenwick Tree / BIT):** Several LIS variants that are O(n²) in the interview-friendly form have an O(n log n) upgrade using a Fenwick tree keyed by (coordinate-compressed) value, supporting range-max or range-sum queries instead of linear scans over `j < i`. The Number of LIS problem (Section 6) is the textbook example: maintain two BITs — one for max `length` in a value prefix, one for the corresponding summed `count` — turning the O(n²) double loop into O(n log n). The same technique generalizes 2D dominance problems (like a Fenwick-tree version of Russian Doll Envelopes with online updates).

3. **Pattern 34 (Greedy):** Maximum Length of Pair Chain (LC 646) LOOKS like an LIS variant (find the longest chain of pairs where each pair's start exceeds the previous pair's end) but is optimally solved with GREEDY interval scheduling (sort by end value, always pick the next chain-compatible pair) in O(n log n) with NO dynamic programming table at all — it is structurally identical to the classic "maximum number of non-overlapping intervals" greedy. Recognizing "this LIS-shaped problem is secretly greedy, not DP" is itself an important skill this pattern reinforces by contrast.

4. **Pattern 17 (1D DP):** The O(n²) LIS template (Section 3) is the direct extension of the "single running array, backward-looking transition" 1D DP style — the only difference from most Pattern 17 problems is that the transition scans ALL prior indices `j < i` rather than a fixed-size window (`i-1`, `i-2`), which is exactly why LIS needs O(n²) (or a cleverer O(n log n) structure) instead of Pattern 17's typical O(n).

5. **Pattern 04 (Monotonic Stack):** The `tails` array in Section 4 is not a stack (we binary search and mutate in the MIDDLE, not just push/pop the top), but it shares the deep invariant "always sorted" with monotonic stacks — both structures exploit an ordering invariant to avoid re-scanning already-resolved elements. Understanding WHY monotonic stacks stay sorted builds the same muscle needed to prove the tails invariant.

---

## SECTION 17 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 300 — Longest Increasing Subsequence

**Interviewer:** "Find the length of the longest strictly increasing subsequence."

**Candidate:**

*[First minute — state the O(n²) baseline, then flag the real target]*
> "Brute force is O(2ⁿ) — try every subset. The standard DP is O(n²): `dp[i]` = LIS length ending at index `i`, transition `dp[i] = 1 + max(dp[j])` over `j < i` with `nums[j] < nums[i]`. Given constraints likely go up to `10^5`, I should target O(n log n) instead."

*[State the O(n log n) insight BEFORE coding]*
> "The trick is patience sorting: maintain an array `tails` where `tails[k]` is the smallest tail value achievable by a length-`(k+1)` increasing subsequence found so far. This array is provably always sorted, so for each new value I binary search for where it belongs — `lower_bound` for the first position `>= x`. If it's past the end, I extend the longest chain; otherwise I replace that slot with a smaller, more useful tail. The final `tails.size()` is the answer."

*[Code — 3 minutes]*
```cpp
int lengthOfLIS(vector<int>& nums) {
    vector<int> tails;
    for (int x : nums) {
        auto it = lower_bound(tails.begin(), tails.end(), x);
        if (it == tails.end()) tails.push_back(x);
        else *it = x;
    }
    return tails.size();
}
```

*[Follow-up]: "Why is `lower_bound` correct here and not `upper_bound`?"*
> "Because the subsequence must be STRICTLY increasing. `lower_bound` finds the first tail `>= x`, so an equal value gets REPLACED, not extended past — correctly rejecting `x` as a valid successor to an equal predecessor. If duplicates were allowed (non-decreasing), I'd switch to `upper_bound`, which only rejects strictly-greater tails, letting an equal value extend the chain."

*[Follow-up]: "Can you reconstruct the actual subsequence, not just the length?"*
> "Yes — I need to track, alongside `tails`, the ORIGINAL INDEX occupying each slot (`tailsIdx`), and a `parent[i]` array. Every time I place index `i` at slot `pos`, `parent[i] = tailsIdx[pos-1]` — whoever currently occupies the previous slot is a valid predecessor of length `pos`. Walking `parent` backward from the last slot's index reconstructs a real subsequence."

**RED FLAGS:**
- Jumping straight to code without stating the tails invariant
- Using `upper_bound` for a strictly-increasing requirement (or vice versa) without justification
- Claiming `tails` itself IS the answer subsequence

---

### Interview Simulation 2: LC 354 — Russian Doll Envelopes

**Interviewer:** "Given envelopes `[width, height]`, find the maximum number that can nest inside each other. Nesting requires strictly smaller width AND height."

**Candidate:**

*[Recognize the 2D → 1D reduction]*
> "This looks like 2D LIS, but I can't binary-search on two independent dimensions directly. The standard trick: sort by width ascending, then run LIS on the height sequence alone. If I sort correctly, an increasing run of heights automatically corresponds to increasing widths too — because the width dimension is already sorted."

*[Critical: the tiebreak]*
> "The subtlety is what happens with EQUAL widths. Nesting needs STRICT inequality in both dimensions, so two same-width envelopes can never nest, no matter their heights. If I sort same-width groups by height ASCENDING, the height-only LIS pass would wrongly treat them as an increasing run — falsely nesting equal-width envelopes. The fix: sort same-width groups by height DESCENDING. Then within a width-tied block, heights appear in decreasing order, so LIS can select at most one envelope from that block — correctly enforcing 'equal width never nests'."

*[Code — 4 minutes]*
```cpp
int maxEnvelopes(vector<vector<int>>& envelopes) {
    sort(envelopes.begin(), envelopes.end(), [](auto& a, auto& b) {
        if (a[0] == b[0]) return a[1] > b[1];
        return a[0] < b[0];
    });
    vector<int> tails;
    for (auto& e : envelopes) {
        auto it = lower_bound(tails.begin(), tails.end(), e[1]);
        if (it == tails.end()) tails.push_back(e[1]);
        else *it = e[1];
    }
    return tails.size();
}
```

*[Interviewer]: "Walk me through why `[[6,4],[6,7]]` sorted this way doesn't overcount."*
> "Sorted, `(6,7)` comes before `(6,4)` because of the descending height tiebreak. When we process height `7`, it likely extends the current tails array. When we then process height `4`, `lower_bound` finds it belongs somewhere in the MIDDLE or at the START of tails — not past the `7` — so it can't be appended after 7 to form a fake length-+1 extension. It just replaces an earlier, larger-or-equal tail value, which doesn't inflate the count."

**RED FLAGS:**
- Sorting both dimensions ascending without discussing the tiebreak at all
- Not being able to construct a concrete failing example for the wrong tiebreak
- Applying O(n²) LIS directly on 2D pairs without the sort reduction (technically correct but misses the intended O(n log n) target)

---

### Interview Simulation 3: Meta-discussion — When is a problem "actually LIS" vs "LIS-shaped but really something else"?

**Interviewer:** "You've now seen LIS, Number of LIS, Russian Doll Envelopes, and Stacking Cuboids. How do you quickly tell whether a new problem reduces to O(n log n) LIS, needs O(n²) DP, or isn't LIS/DP at all?"

**Candidate:**
> "Three checks, in order:
>
> **Check 1 — Is there a single, sortable, totally-ordered key that fully captures 'can X follow Y'?** If yes (like plain LIS, or Russian Doll after the width/height sort reduction), O(n log n) via binary search on tails is available. If the compatibility condition needs MULTIPLE independent dimensions simultaneously (like Stacking Cuboids' 3D dominance, or Longest Arithmetic Subsequence's difference-keyed matching), there's no single sortable key, and I fall back to O(n²) DP with a hashmap or explicit multi-dimension check per pair.
>
> **Check 2 — Do I need the count of solutions, or a reconstruction, not just the max length?** Both of these typically FORCE you off the pure tails-array representation, because it collapses information (only the minimal tail survives). Counting needs `length[]` + `count[]` two-array DP (or a Fenwick tree upgrade); reconstruction needs parent pointers tracked alongside whatever structure you use.
>
> **Check 3 — Is the 'chain' relation actually about SCHEDULING (non-overlapping intervals) rather than value comparisons?** If pairs/intervals are compatible based on one ending before another starts, that's classic GREEDY interval scheduling (sort by end, pick greedily) — like Max Length of Pair Chain — NOT LIS, even though the problem statement looks superficially identical to LIS on pairs. Greedy is O(n log n) with no DP table at all, and is strictly simpler to code and reason about once recognized."

**RED FLAGS:**
- Defaulting to "just do O(n²) DP" for every problem without checking if O(n log n) or even greedy applies
- Not recognizing that "Number of LIS" and "count ALL increasing subsequences" are different problems with different DP formulations
- Missing that Max Length of Pair Chain is greedy, wasting interview time building an unnecessary DP

---

## SECTION 18 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why does the `tails` array give the correct LIS LENGTH even though it is not itself a valid real subsequence of the input?**

> Every value ever WRITTEN into `tails` is justified, at the moment of writing, by an explicit real increasing subsequence constructed from previously processed elements (proven by induction in Section 4: the append case chains onto the previous best tail, and the replace case chains onto `tails[pos-1]`). So `tails.size()` at any moment equals the TRUE maximum achievable length using elements processed so far — even though later overwrites erase the specific values that justified earlier entries, the LENGTH claim survives because each growth step was independently validated when it happened. The array's CONTENTS drift away from any single coherent subsequence, but its SIZE never lies about the achievable length.

---

**Q2: Prove that overwriting `tails[pos]` with a smaller value can never make the algorithm undercount later.**

> Overwriting only ever REPLACES a value with a smaller-or-equal one at the SAME length slot (`pos+1`). A smaller tail value is strictly more useful for future extensions (more values can legally follow a smaller number than a larger one). So the invariant "smallest possible tail per length" can only IMPROVE (or stay the same) — it never removes the possibility of reaching any length that was reachable before the overwrite. Since every future decision only cares about "is `x` greater than the tail at slot `k`," and we've only made that tail smaller-or-equal, every future append/replace decision is at least as permissive as it would have been with the old, larger value. No achievable length is ever lost by this replacement.

---

**Q3: Why does strictly-increasing LIS require `lower_bound` while non-decreasing LIS requires `upper_bound`?**

> `lower_bound(x)` finds the first tail `>= x`. Using it means: if a tail EQUALS `x`, we treat `x` as "not strictly better" and overwrite that slot rather than extending past it — correctly enforcing that `x` cannot directly follow an EQUAL predecessor in a strictly increasing chain.
>
> `upper_bound(x)` finds the first tail `> x`. Using it means: if a tail EQUALS `x`, we do NOT touch that slot — instead `x` gets placed at the NEXT slot (or appended), because an equal predecessor IS allowed to be immediately followed by another equal value in a non-decreasing chain. This correctly lets runs of duplicates count as valid extensions.
>
> The one-character swap between these two functions is the entire difference between solving "strictly increasing" and "non-decreasing" LIS.

---

**Q4: In Number of LIS, why do we need BOTH `length[]` and `count[]` arrays — why can't `count[]` alone work?**

> `count[i]` on its own has no way to know WHICH predecessors `j` are actually relevant — i.e., which `j` achieve the maximum possible chain length ending at `i`. Without `length[]`, we cannot distinguish "j supports a length-3 chain that happens to be shorter than i's true best of length-5" (irrelevant, should be ignored) from "j supports a length-4 chain that ties i's best of length-5 minus one" (relevant, should be accumulated). `length[]` is what lets us apply the exact reset-vs-accumulate rule from Section 6: reset `count[i]` when a STRICTLY longer chain is found through `j` (discarding stale counts from worse predecessors), and accumulate only on an EXACT length tie. Neither array alone carries enough information — they must be updated together, in the same loop, off each other's current values.

---

**Q5: In Russian Doll Envelopes, why sort width ascending + height DESCENDING (not ascending) for tied widths?**

> Because nesting demands STRICT inequality in width. Two envelopes sharing the same width can NEVER nest inside each other, regardless of their heights. If tied-width envelopes were sorted by height ascending, the subsequent LIS-on-heights pass (which only looks at height values, blind to width) would see an increasing run of heights within that block and incorrectly count it as a valid nesting chain. Sorting tied-width blocks by height DESCENDING forces heights to appear in decreasing order within each block, so the increasing-subsequence algorithm can select AT MOST ONE envelope from any single width-tied group — which is exactly the correct constraint, since a valid nesting chain can include at most one envelope of any given width.

---

**Q6: Why is Maximum Height by Stacking Cuboids O(n²) instead of O(n log n), even though it looks just like Russian Doll Envelopes with an extra dimension?**

> Russian Doll Envelopes reduces from 2D to 1D because a clever sort (with the right tiebreak) makes ONE remaining dimension (height) fully capture the nesting condition via a single sortable/binary-searchable key. With THREE dimensions, no sort order can collapse the problem to a single comparable key — "cuboid A can sit under cuboid B" requires checking ALL THREE dimensions independently (width, length, AND height), and there is no total order that guarantees "if A comes before B in sorted order, A dominates B in all three dimensions." The lexicographic sort in Section 11 is only a NECESSARY precondition (ensures we never need to look at `j > i`), not a SUFFICIENT one — we must still explicitly verify 3-way dominance for every candidate pair, which is fundamentally an O(n²) check. Binary search cannot prune this because there's no monotonic structure across all three dimensions simultaneously.

---

**Q7: How does patience sorting connect to Dilworth's theorem, and what does that connection tell us?**

> The literal patience-sorting card game (dealing cards into piles, starting a new pile only when a card can't go on top of any existing pile, always placing on the leftmost valid pile) produces a NUMBER OF PILES that exactly equals the LENGTH of the longest increasing subsequence. Each pile, read top to bottom, is a DECREASING sequence (by the placement rule). So the minimum number of decreasing sequences needed to partition the whole array equals the number of piles, which equals the LIS length. This is a direct instance of Dilworth's theorem: in the partial order "i comes before j AND value_i < value_j," the minimum number of CHAINS (here: decreasing runs, which are ANTICHAINS in this specific partial order since no two elements in a decreasing run satisfy the increasing relation) needed to cover the set equals the size of the maximum ANTICHAIN (here: the LIS itself, since it's the largest set of mutually "comparable" — i.e., increasing — elements). This theorem is the deep reason the greedy binary-search construction in Section 4 works at all — it's not a coincidence, it's a combinatorial guarantee.

---

## SECTION 19 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. O(n²) LIS BASELINE
// ─────────────────────────────────────────────────────────────────
vector<int> dp(n, 1);
for (int i = 1; i < n; i++)
    for (int j = 0; j < i; j++)
        if (nums[j] < nums[i]) dp[i] = max(dp[i], dp[j] + 1);
int ans = *max_element(dp.begin(), dp.end());   // NOT dp[n-1]!

// ─────────────────────────────────────────────────────────────────
// 2. O(n log n) LIS — patience sorting on tails
// ─────────────────────────────────────────────────────────────────
vector<int> tails;
for (int x : nums) {
    auto it = lower_bound(tails.begin(), tails.end(), x);  // STRICT
    // auto it = upper_bound(tails.begin(), tails.end(), x); // NON-DECREASING
    if (it == tails.end()) tails.push_back(x);
    else *it = x;
}
return tails.size();   // tails ITSELF is not a valid subsequence

// ─────────────────────────────────────────────────────────────────
// 3. LIS RECONSTRUCTION — track index + parent alongside tails
// ─────────────────────────────────────────────────────────────────
vector<int> tailsVal, tailsIdx, parent(n, -1);
for (int i = 0; i < n; i++) {
    auto it = lower_bound(tailsVal.begin(), tailsVal.end(), nums[i]);
    int pos = it - tailsVal.begin();
    if (pos > 0) parent[i] = tailsIdx[pos - 1];
    if (it == tailsVal.end()) { tailsVal.push_back(nums[i]); tailsIdx.push_back(i); }
    else { *it = nums[i]; tailsIdx[pos] = i; }
}
// walk parent[] backward from tailsIdx.back() to recover the sequence

// ─────────────────────────────────────────────────────────────────
// 4. NUMBER OF LIS — the reset-vs-accumulate branch
// ─────────────────────────────────────────────────────────────────
vector<int> length(n, 1), count(n, 1);
for (int i = 0; i < n; i++) {
    for (int j = 0; j < i; j++) {
        if (nums[j] < nums[i]) {
            if (length[j] + 1 > length[i]) { length[i] = length[j] + 1; count[i] = count[j]; }
            else if (length[j] + 1 == length[i]) count[i] += count[j];
        }
    }
}
// answer = sum of count[i] where length[i] == max(length)

// ─────────────────────────────────────────────────────────────────
// 5. RUSSIAN DOLL ENVELOPES — sort tiebreak + LIS
// ─────────────────────────────────────────────────────────────────
sort(envelopes.begin(), envelopes.end(), [](auto& a, auto& b) {
    if (a[0] == b[0]) return a[1] > b[1];   // tie: height DESCENDING
    return a[0] < b[0];                      // width ascending
});
// then run template 2 (tails) on the height sequence

// ─────────────────────────────────────────────────────────────────
// 6. DIFFERENCE-KEYED DP — Longest Arithmetic Subsequence
// ─────────────────────────────────────────────────────────────────
vector<unordered_map<int,int>> dp(n);
for (int i = 1; i < n; i++)
    for (int j = 0; j < i; j++) {
        int diff = nums[i] - nums[j];
        int prevLen = dp[j].count(diff) ? dp[j][diff] : 1;
        dp[i][diff] = max(dp[i][diff], prevLen + 1);
    }

// ─────────────────────────────────────────────────────────────────
// 7. BUCKETED DP — small fixed key space (Longest Ideal Subsequence)
// ─────────────────────────────────────────────────────────────────
vector<int> dp(26, 0);
for (char ch : s) {
    int c = ch - 'a', best = 0;
    for (int p = max(0, c - k); p <= min(25, c + k); p++) best = max(best, dp[p]);
    dp[c] = best + 1;
}

// ─────────────────────────────────────────────────────────────────
// 8. 3D DOMINANCE DP — Stacking Cuboids
// ─────────────────────────────────────────────────────────────────
for (auto& c : cuboids) sort(c.begin(), c.end());   // normalize orientation
sort(cuboids.begin(), cuboids.end());                // global lexicographic sort
vector<int> dp(n);
for (int i = 0; i < n; i++) {
    dp[i] = cuboids[i][2];
    for (int j = 0; j < i; j++)
        if (cuboids[j][0] <= cuboids[i][0] && cuboids[j][1] <= cuboids[i][1] &&
            cuboids[j][2] <= cuboids[i][2])
            dp[i] = max(dp[i], dp[j] + cuboids[i][2]);
}

// ─────────────────────────────────────────────────────────────────
// 9. DECISION TABLE
// ─────────────────────────────────────────────────────────────────
// Single sortable key, length only          → O(n log n) tails (lower_bound)
// Single sortable key, duplicates allowed    → O(n log n) tails (upper_bound)
// Single sortable key, need actual sequence  → tails + tailsIdx + parent[]
// Single sortable key, need COUNT of LIS     → O(n²) length[]+count[] (or Fenwick for O(n log n))
// Compatibility needs 2 dims, one sortable   → sort + tiebreak, reduce to 1D tails
// Compatibility needs 3+ dims                → O(n²) explicit dominance DP
// Compatibility keyed by a DIFFERENCE        → O(n²) hashmap per index
// Key space is small & fixed (alphabet)      → O(n × |alphabet|) bucketed dp
// "Chain" is really about interval scheduling → GREEDY, not DP at all
```

---

## SECTION 20 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Russian Doll Envelopes and Stacking Cuboids are the two that require the deepest sort/tiebreak reasoning. Budget 40-45 minutes each.

2. **Force yourself to derive the `lower_bound` vs `upper_bound` proof from scratch** on a blank page, with a concrete 3-element example, before looking at Section 14's Mistake 1 again. This single distinction is asked disproportionately often relative to its apparent triviality.

3. **Number of LIS → Fenwick tree upgrade.** After solving LC 673 with the O(n²) two-array method, attempt the O(n log n) version using two Fenwick trees keyed by coordinate-compressed value (Pattern Connections #2). This is a strong signal of 2000+ readiness.

4. **Preview Pattern 27/28 (Segment Tree / BIT):** several LIS variants become dramatically faster with a Fenwick tree keyed by value instead of a linear scan over prior indices — this pattern is the most natural motivating example for WHY you'd want that data structure.

5. **The reconstruction challenge:** after finishing the pattern, implement LIS reconstruction (Section 5) for the O(n log n) version WITHOUT looking at the template — this is the most commonly assigned interview follow-up on top of plain LIS, and it separates "memorized the trick" from "understands the invariant."

6. **Contrast exercise:** solve LC 646 (Maximum Length of Pair Chain) BOTH as a DP (treating it like generic LIS on pairs, O(n²) or O(n log n)) AND as the intended greedy (O(n log n), sort by end). Time both. This cements the meta-skill from Interview Simulation 3 — recognizing when DP is unnecessary.

---

## SECTION 21 — SIGN-OFF CRITERIA

### Tier 1 — LIS foundation mastered
You solve LC 300 both as O(n²) DP and O(n log n) patience sorting, cleanly, without referencing notes. You can state the `tails[k]` invariant from memory and explain why `max(dp)` (not `dp[n-1]`) is the answer in the O(n²) version.

### Tier 2 — Binary search subtleties understood
You correctly choose `lower_bound` vs `upper_bound` for strictly-increasing vs non-decreasing variants without hesitation, and can construct a 2-element example proving the difference on demand. You can reconstruct an actual LIS (not just its length) using parent pointers, for both the O(n²) and O(n log n) versions.

### Tier 3 — Core variants solid
You solve Number of LIS (LC 673) with the correct reset-vs-accumulate branch and can explain why `count[]` needs `length[]` alongside it. You correctly identify Maximum Length of Pair Chain (LC 646) as greedy, not DP, and solve it in O(n log n) with no DP table.

### Tier 4 — 2D/3D reductions and hard variants complete
You solve Russian Doll Envelopes (LC 354) with the correct width-ascending/height-descending tiebreak and can prove why the tiebreak is necessary. You solve Stacking Cuboids (LC 1691) and can explain precisely why it's stuck at O(n²) despite superficially resembling Russian Doll Envelopes. You solve Longest Arithmetic Subsequence (LC 1027) and Longest Ideal Subsequence (LC 2370), correctly identifying why neither reduces to binary search on a single sortable key.

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 300 (both complexity classes — the `tails` invariant proof must be explainable), LC 673 (Number of LIS — the reset-vs-accumulate logic must be correct), and LC 354 (Russian Doll Envelopes — the tiebreak reasoning must be explicit and provable, not just memorized). These three represent the three deepest conceptual pillars of this pattern: the core binary-search proof, the counting subtlety, and the dimensionality-reduction trick.

---

*Pattern 21 Complete — LIS Family (Longest Increasing Subsequence)*
*Next: Pattern 22 — DP on Trees (postorder accumulation, tree diameter, rerooting technique)*
