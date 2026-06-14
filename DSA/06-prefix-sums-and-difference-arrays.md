# PATTERN 06 — PREFIX SUMS AND DIFFERENCE ARRAYS
## DSA Grandmaster Series | C++ | From 1600 → 2000+
## Tutor: Grandmaster + Principal Engineer Persona

---

## METADATA

| Field | Value |
|-------|-------|
| Pattern Number | 06 |
| Estimated Mastery Time | 3-4 days |
| Interview Frequency | OFTEN |
| Contest Frequency | ALWAYS (appears in nearly every CF Div 2 A/B/C) |
| Difficulty | FOUNDATION |
| Prerequisites | Arrays, basic hash maps |
| Next Pattern | 07 — Linked Lists |

---

## THE GRANDMASTER'S HONEST TAKE

Prefix sums are the most underrated pattern in competitive programming.

Every intermediate programmer knows Two Sum. Every intermediate programmer has heard of sliding window. But prefix sums — and more importantly their powerful variants — get dismissed as "too easy" and not drilled deeply enough. Then they fail a contest problem that needed prefix XOR, or an interview problem that needed 2D prefix sums, and wonder why.

**Let me tell you what actually separates 1600 from 1800 on this topic:**

A 1600-rated programmer knows: `prefix[i] = prefix[i-1] + arr[i-1]` and can compute subarray sums in O(1).

A 1800-rated programmer knows:
1. **Prefix sum + hash map** to count subarrays with a target sum in O(n) — the `prefixCount[0] = 1` initialization is the key insight that most people miss
2. **Prefix sum modular arithmetic** for "divisible by K" problems
3. **Prefix XOR** for XOR range queries
4. **2D prefix sums** for matrix region sums with the inclusion-exclusion formula
5. **Difference arrays** for O(1) range updates
6. **Difference arrays on 2D matrices** for rectangle range updates

If you treat prefix sums as "just the easy warmup" and skip straight to binary search and DP, you'll find yourself unable to implement the check functions for many binary-search-on-answer problems — because the check function IS a prefix sum + hash map computation.

**The secret:** prefix sums don't exist in isolation. They're the FOUNDATION for:
- Sliding window (the "sum in a window" check is secretly prefix sums)
- Binary search (check function for "how many subarrays have sum ≥ X")
- DP optimization (some DP transitions are really prefix sum queries)
- Segment trees (at their core, they're prefix sums with update capability)

Grind this pattern for 3 focused days and it will pay dividends in every other pattern.

---

## ELI5 — THE RUNNING TOTAL ANALOGY

**Prefix Sums:**
Imagine you're running a marathon and checkpoints record the total distance covered so far:
- Checkpoint 0: 0 km
- Checkpoint 1: 3 km
- Checkpoint 2: 8 km
- Checkpoint 3: 15 km
- Checkpoint 4: 21 km

"What was the distance run between checkpoint 2 and checkpoint 4?" Without prefix sums: you'd add up all legs between 2 and 4. With prefix sums: `21 - 8 = 13 km`. One subtraction, O(1).

**Difference Arrays:**
Now imagine you want to MODIFY the marathon — add 5 km to every leg between checkpoints 2 and 4.
- Without difference arrays: update 2 values individually (checkpoints 3 and 4 shift). Fine for 2 values.
- With n=10^6 checkpoints and 10^5 updates, each touching ranges of 10^5 positions: that's 10^10 operations. Not fine.
- Difference array: record "+5 at position 2, -5 at position 5." Apply all updates at the end with one scan. O(1) per update, O(n) to materialize.

**The Hash Map Twist:**
"How many subarrays sum to exactly 7?"
- Brute force: check all O(n²) pairs.
- Insight: if `prefix[j] - prefix[i] = 7`, then subarray `[i+1..j]` sums to 7.
- Rearranging: `prefix[i] = prefix[j] - 7`.
- So as you compute `prefix[j]`, look up "how many times have we seen `prefix[j] - 7` before?" in a hash map.
- This turns O(n²) into O(n).

That's the entire intellectual depth of the prefix sum + hash map combo. Everything else is variations on these three ideas.

---

## PATTERN FORMALLY DEFINED

### Definition 1: 1D Prefix Sum
Given array `a[0..n-1]`, define `prefix[i] = a[0] + a[1] + ... + a[i-1]` (1-indexed, with `prefix[0] = 0`).

Then: `sum(l, r) = prefix[r+1] - prefix[l]` (0-indexed range `[l, r]`).

**Size:** `prefix` has size `n+1`. `prefix[0] = 0`, `prefix[i] = prefix[i-1] + a[i-1]` for `i` from 1 to n.

### Definition 2: Prefix Sum + Hash Map (Count Subarrays)
"Count subarrays with sum = target":
- As we compute running prefix sum `curr`, ask: "how many previous prefix sums equal `curr - target`?"
- Use a hash map `count` where `count[prefix_sum]` = number of times this prefix sum has been seen.
- Initialize `count[0] = 1` (the empty prefix — before the first element, prefix sum is 0).

### Definition 3: Difference Array
Given array `a[0..n-1]`, define `diff[i] = a[i] - a[i-1]` (with `diff[0] = a[0]`).

Then: `a[i] = diff[0] + diff[1] + ... + diff[i]` (prefix sum of diff = original array).

**Range update [l, r] += val:** `diff[l] += val`, `diff[r+1] -= val`. Then apply prefix sum to recover the updated array.

### Definition 4: 2D Prefix Sum
Given matrix `A[m][n]`, define `P[i][j]` = sum of all elements in the rectangle `[0..i-1][0..j-1]`.

Then: `P[i][j] = A[i-1][j-1] + P[i-1][j] + P[i][j-1] - P[i-1][j-1]`

Sum of rectangle `(r1, c1)` to `(r2, c2)` (0-indexed):
`P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]`

---

## RECOGNITION SIGNALS

**Signal 1:** The problem asks for the sum (or XOR, or some aggregate) of a contiguous subarray, with multiple queries on a static array
→ Build a prefix sum array once in O(n), then answer each query in O(1). The key indicator is "multiple queries" with "no updates between queries." If the array is static and you're querying ranges, prefix sums are almost always the right tool.

**Signal 2:** "Count subarrays with sum = K" or "count subarrays with sum divisible by K" or "count subarrays with exactly M odd numbers"
→ Prefix sum + hash map. These exact phrasings mean you need to count pairs (i, j) where `prefix[j] - prefix[i] = K`. For each j, you query the hash map for `prefix[j] - K`. The `prefixCount[0] = 1` initialization is NON-NEGOTIABLE — it handles subarrays that start at index 0.

**Signal 3:** "Range update — add value V to all elements in range [l, r] — then read the final array"
→ Difference array. If the operations are N range updates followed by a single read of the entire array, difference array does this in O(N + n) total instead of O(N × n). The sentinel at index n+1 absorbs the "undo" for updates ending at the last element.

**Signal 4:** The problem involves a 2D matrix with "sum of elements in rectangle (r1,c1) to (r2,c2)"
→ 2D prefix sum. Build the 2D prefix array in O(m×n), then each rectangle query is O(1) inclusion-exclusion. This is the 2D generalization of the 1D range sum query.

**Signal 5:** "Subarray sum equals K" with the array containing NEGATIVE numbers
→ You cannot use sliding window (negative numbers break the monotonicity). You MUST use prefix sum + hash map. This is the critical distinction that separates the two approaches — when the interviewer says "the array may contain negative numbers," they are telling you sliding window won't work.

**Signal 6:** The problem has "longest subarray" or "minimum length subarray" with some sum condition
→ Prefix sum + hash map tracking FIRST occurrence of each prefix value (not count). For longest: `maxLen = max(maxLen, i - firstSeen[prefix])`. For minimum: needs a deque or different approach.

**Signal 7:** "Product of array except self" or "sum of elements to the left/right"
→ Prefix product (or sum) from the left combined with a suffix sweep from the right. Build once from each direction, combine in O(n) total.

**Signal 8:** The problem says "at most K" subarrays or "at most K" aggregate — and the array is non-negative
→ Binary search on the count using prefix sums, or sliding window (valid for non-negative). "Exactly K" = "at most K" minus "at most K-1." This transformation is extremely common in advanced prefix sum problems.

### Signals that LOOK like prefix sums but are NOT:

**Fake signal 1:** "Range sum query with point updates" (e.g., update nums[i] = x, then query range sum)
→ Prefix sum breaks the moment you update an element — you'd need to rebuild the entire prefix array in O(n). Use a Fenwick tree (Pattern 28) for O(log n) updates and queries.

**Fake signal 2:** "Maximum subarray sum of EXACTLY length K"
→ Sliding window (fixed window size). Prefix sum also works here (`prefix[i+k] - prefix[i]`) but sliding window is slightly cleaner. Both are O(n).

**Fake signal 3:** "Range updates AND range queries simultaneously"
→ Difference array only handles range updates + full final read. For simultaneous range updates and range queries, you need lazy segment tree (Pattern 27).

**Fake signal 4:** "Maximum subarray sum" (without subarray length constraint)
→ Kadane's algorithm O(n). Not a prefix sum problem. You CAN express Kadane's as "maximum over all j of (prefix[j] - min prefix seen so far)" but coding it as Kadane's is much cleaner.

### The 3-Question Test — Apply Before Writing Any Code:

1. "Does the answer depend on a CONTIGUOUS range's aggregate (sum/XOR/product) on a STATIC array?" → Prefix aggregate (build once, query O(1))
2. "Do I need to COUNT subarrays with a sum property?" → Prefix sum + hash map (`prefixCount[0] = 1`, look up BEFORE updating)
3. "Are there range UPDATES with a delayed final SINGLE read?" → Difference array (O(updates + n) total)

---

## TEMPLATES IN C++

### Template 1 — 1D Prefix Sum (build and query)

```cpp
// Build prefix sum array
// prefix[i] = sum of first i elements (a[0] through a[i-1])
// prefix[0] = 0 (empty prefix)

vector<int> buildPrefix(vector<int>& a) {
    int n = a.size();
    vector<int> prefix(n + 1, 0);          // Size n+1, initialized to 0
    for (int i = 1; i <= n; i++) {
        prefix[i] = prefix[i-1] + a[i-1];  // 1-indexed prefix, 0-indexed array
    }
    return prefix;
}

// Query: sum of a[l..r] (0-indexed, inclusive)
int rangeSum(vector<int>& prefix, int l, int r) {
    return prefix[r+1] - prefix[l];
}

// Example usage:
// a = [3, 5, 2, 8, 1]
// prefix = [0, 3, 8, 10, 18, 19]
// rangeSum(prefix, 1, 3) = prefix[4] - prefix[1] = 18 - 3 = 15 (= 5+2+8)
```

**Why `prefix[0] = 0`?** So that rangeSum starting from index 0 works: `rangeSum(prefix, 0, r) = prefix[r+1] - prefix[0] = prefix[r+1]`. Without this sentinel, you'd need a special case.

---

### Template 2 — Prefix Sum + Hash Map (count subarrays with sum = target)

```cpp
// LC 560: Count subarrays with sum exactly = target
// Key insight: sum(l..r) = prefix[r+1] - prefix[l] = target
// → prefix[l] = prefix[r+1] - target
// → as we process element r, count how many l's had prefix[l] = curr - target

int subarraySum(vector<int>& nums, int target) {
    unordered_map<int, int> prefixCount;
    prefixCount[0] = 1;     // CRITICAL: empty prefix has sum 0, seen once
                            // This handles subarrays starting from index 0
    
    int curr = 0, result = 0;
    for (int x : nums) {
        curr += x;                              // curr = prefix sum up to current element
        
        // How many previous prefix sums equal (curr - target)?
        // Each such prefix gives a valid subarray ending here
        result += prefixCount[curr - target];   // If not found, returns 0 (unordered_map default)
        
        prefixCount[curr]++;                    // Record this prefix sum
    }
    
    return result;
}

// THE INITIALIZATION: prefixCount[0] = 1
// Without it: a subarray [0..j] with sum = target would be missed
// because we'd look for prefix[i] = curr - target = target - target = 0
// and find 0 instead of 1.
// Example: nums = [3, 2, 1], target = 6
// At j=2: curr = 6, curr - target = 0, prefixCount[0] = 1 → result += 1 ✓
// Without init: prefixCount[0] = 0 → misses the entire array
```

**Why `unordered_map` default returns 0:** In C++, accessing a missing key in `unordered_map<int,int>` via `[]` operator creates it with value 0. This is convenient — `prefixCount[curr - target]` returns 0 when that prefix sum has never been seen, adding nothing to result.

---

### Template 3 — Prefix Sum + Hash Map (modular arithmetic, subarrays divisible by K)

```cpp
// LC 523: Check if any subarray has sum divisible by K
// LC 974: Count subarrays with sum divisible by K
// Key insight: if prefix[r] ≡ prefix[l] (mod K), then sum(l..r) is divisible by K

int subarraysDivByK(vector<int>& nums, int k) {
    unordered_map<int, int> modCount;
    modCount[0] = 1;        // Empty prefix has sum 0, which is divisible by k
    
    int curr = 0, result = 0;
    for (int x : nums) {
        curr = ((curr + x) % k + k) % k;   // CRITICAL: +k to handle negative numbers
                                            // In C++, (-3) % 5 = -3, not 2
        
        result += modCount[curr];           // Count previous indices with same mod
        modCount[curr]++;
    }
    
    return result;
}

// THE NEGATIVE MOD FIX: ((val % k) + k) % k
// C++ computes (-3) % 5 = -3 (implementation-defined, but typically negative)
// We need the mathematical modulo: -3 mod 5 = 2
// ((−3 % 5) + 5) % 5 = (−3 + 5) % 5 = 2 % 5 = 2 ✓
```

---

### Template 4 — Prefix XOR (count subarrays with XOR = target)

```cpp
// XOR has a beautiful property: prefix_xor[l..r] = prefix_xor[0..r] XOR prefix_xor[0..l-1]
// So: if prefix_xor[r] XOR prefix_xor[l-1] = target
//     then prefix_xor[l-1] = prefix_xor[r] XOR target
// (Because x XOR x = 0, and x XOR 0 = x)

int countSubarraysWithXOR(vector<int>& nums, int target) {
    unordered_map<int, int> xorCount;
    xorCount[0] = 1;        // Same as prefix sum: empty prefix has XOR = 0
    
    int curr = 0, result = 0;
    for (int x : nums) {
        curr ^= x;                          // Running XOR prefix
        result += xorCount[curr ^ target];  // Look for prefix_xor that, when XORed with curr, gives target
        xorCount[curr]++;
    }
    
    return result;
}

// XOR PROPERTIES THAT MAKE THIS WORK:
// a XOR a = 0
// a XOR 0 = a
// XOR is associative and commutative
// So: curr XOR (curr XOR target) = target ✓
// Which means: we look for xorCount[curr XOR target]
```

**When to use Prefix XOR instead of Prefix Sum:**
- Any problem asking about subarray XOR queries
- Checking if subarray XOR equals 0 (check for duplicate prefix XOR)
- Maximum XOR of subarray (combine with Trie — see Pattern 26)

---

### Template 5 — 2D Prefix Sum (build and query)

```cpp
// For m×n matrix, build (m+1)×(n+1) prefix sum matrix
// P[i][j] = sum of rectangle from (0,0) to (i-1,j-1)
// P[0][*] = P[*][0] = 0 (sentinel row and column)

class NumMatrix {
    vector<vector<int>> P;
    
public:
    NumMatrix(vector<vector<int>>& matrix) {
        int m = matrix.size(), n = matrix[0].size();
        P.assign(m + 1, vector<int>(n + 1, 0));
        
        for (int i = 1; i <= m; i++) {
            for (int j = 1; j <= n; j++) {
                P[i][j] = matrix[i-1][j-1]     // Current cell
                         + P[i-1][j]            // Sum above
                         + P[i][j-1]            // Sum to the left
                         - P[i-1][j-1];         // Subtract double-counted top-left
            }
        }
    }
    
    // Query: sum of rectangle (r1,c1) to (r2,c2) — 0-indexed
    int sumRegion(int r1, int c1, int r2, int c2) {
        return P[r2+1][c2+1]    // Full rectangle to (r2,c2)
             - P[r1][c2+1]      // Remove top strip
             - P[r2+1][c1]      // Remove left strip
             + P[r1][c1];       // Add back top-left (subtracted twice)
    }
};

// VISUAL: The inclusion-exclusion for sumRegion
//
//  +-------+-------+
//  |       |       |
//  |  TL   |  T    |
//  |       |       |
//  +-------+-------+
//  |       |  ///  |
//  |  L    | query |
//  |       |  ///  |
//  +-------+-------+
//
// sum(query) = P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]
//            = Full - T - L + TL
// (TL was subtracted twice in T and L, so add it back once)
```

---

### Template 6 — Difference Array (range updates)

```cpp
// Range update: add val to all elements in [l, r]
// Then recover the updated array in O(n)

class DifferenceArray {
    vector<int> diff;
    int n;
    
public:
    // Build difference array from initial array
    DifferenceArray(vector<int>& a) {
        n = a.size();
        diff.resize(n + 1, 0);              // Extra slot to handle r+1 = n safely
        diff[0] = a[0];
        for (int i = 1; i < n; i++) {
            diff[i] = a[i] - a[i-1];
        }
    }
    
    // Build difference array from all-zeros array (if starting from scratch)
    DifferenceArray(int n) : n(n), diff(n + 1, 0) {}
    
    // Range update: a[l..r] += val in O(1)
    void update(int l, int r, int val) {
        diff[l] += val;
        diff[r + 1] -= val;                 // r+1 might equal n — that's why size is n+1
    }
    
    // Recover the final array in O(n)
    vector<int> result() {
        vector<int> a(n);
        a[0] = diff[0];
        for (int i = 1; i < n; i++) {
            a[i] = a[i-1] + diff[i];        // Prefix sum of diff = original array
        }
        return a;
    }
};

// Example:
// Initial: [0, 0, 0, 0, 0]
// update(1, 3, +5): diff = [0, 5, 0, 0, -5, 0]
// update(0, 2, +3): diff = [3, 5, 0, -3, -5, 0]
// result: a[0]=3, a[1]=3+5=8, a[2]=8+0=8, a[3]=8-3=5, a[4]=5-5=0
// = [3, 8, 8, 5, 0]
// Verify: index 0: +3 → 3; index 1: +5+3 → 8; index 2: +5+3 → 8; index 3: +5 → 5; index 4: 0 → 0 ✓
```

---

### Template 7 — Difference Array: Corporate Flight Bookings (full problem)

```cpp
// LC 1109: n seats on flights 1..n. Bookings: [first, last, seats].
// Add seats to all flights [first, last]. Return total seats per flight.

vector<int> corpFlightBookings(vector<vector<int>>& bookings, int n) {
    vector<int> diff(n + 1, 0);            // 1-indexed flights, extra slot
    
    for (auto& b : bookings) {
        int first = b[0], last = b[1], seats = b[2];
        diff[first] += seats;              // 1-indexed
        if (last + 1 <= n) diff[last + 1] -= seats;
        // or: diff[last + 1] -= seats; (the n+1 slot absorbs excess)
    }
    
    // Recover: prefix sum of diff = answer
    vector<int> answer(n);
    answer[0] = diff[1];                   // Convert 1-indexed diff to 0-indexed answer
    for (int i = 1; i < n; i++) {
        answer[i] = answer[i-1] + diff[i+1];
    }
    
    return answer;
}
```

---

### Template 8 — Prefix Sum on Transformed Array (Count Nice Subarrays)

```cpp
// LC 1248: Count subarrays with exactly k odd numbers
// Transform: odd → 1, even → 0. Then count subarrays with sum = k.
// This is the "transform then prefix sum" technique.

int numberOfSubarrays(vector<int>& nums, int k) {
    // Transform: replace each number with its parity
    // Then run standard "count subarrays with sum = k"
    
    unordered_map<int, int> prefixCount;
    prefixCount[0] = 1;
    
    int curr = 0, result = 0;
    for (int x : nums) {
        curr += (x % 2);                    // 1 if odd, 0 if even
        result += prefixCount[curr - k];
        prefixCount[curr]++;
    }
    
    return result;
}

// THE TRANSFORM INSIGHT:
// Many problems that ask "count subarrays with property X" can be solved by:
// 1. Transform array so that "count of X in subarray" = sum of subarray
// 2. Run prefix sum + hash map for "count subarrays with sum = target"
//
// Other transforms:
// - "k zeros": 0→1, non-zero→0
// - "k elements divisible by d": divisible→1, otherwise→0
// - "balanced parentheses": '('→+1, ')'→-1, look for sum=0
```

---

### Template 9 — Prefix Sum for "At Most K" → "Exactly K" (Alternative to Sliding Window)

```cpp
// For problems where atMost(K) is easier to compute than exactly(K):
// exactly(K) = atMost(K) - atMost(K-1)

// This is covered in depth in Pattern 03 (Sliding Window)
// But prefix sums can also implement atMost:

// Example: count subarrays with at most K distinct characters
int atMost(string& s, int k) {
    unordered_map<char, int> freq;
    int left = 0, result = 0;
    for (int right = 0; right < s.size(); right++) {
        freq[s[right]]++;
        while (freq.size() > k) {
            if (--freq[s[left]] == 0) freq.erase(s[left]);
            left++;
        }
        result += right - left + 1;     // All subarrays ending at 'right' with ≤ k distinct
    }
    return result;
}

// LC 992: Subarrays with exactly K different integers
int subarraysWithKDistinct(vector<int>& nums, int k) {
    return atMost(nums, k) - atMost(nums, k - 1);
}
```

---

### Template 10 — 2D Difference Array (rectangle range updates)

```cpp
// Range update: add val to all cells in rectangle (r1,c1) to (r2,c2)
// Then materialize the final matrix in O(m*n)

void rectangle2DUpdate(vector<vector<int>>& diff, int r1, int c1, int r2, int c2, int val) {
    diff[r1][c1]         += val;
    diff[r1][c2 + 1]     -= val;
    diff[r2 + 1][c1]     -= val;
    diff[r2 + 1][c2 + 1] += val;
}

vector<vector<int>> materialize2D(vector<vector<int>>& diff, int m, int n) {
    // First apply prefix sum row-wise
    for (int i = 0; i < m + 1; i++) {
        for (int j = 1; j < n + 1; j++) {
            diff[i][j] += diff[i][j-1];
        }
    }
    // Then apply prefix sum column-wise
    for (int j = 0; j < n + 1; j++) {
        for (int i = 1; i < m + 1; i++) {
            diff[i][j] += diff[i-1][j];
        }
    }
    // Extract result (ignoring the extra row/column)
    vector<vector<int>> result(m, vector<int>(n));
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            result[i][j] = diff[i][j];
    return result;
}

// Usage: LC 2132, or problems with many rectangle additions followed by one read
// Each rectangle update is O(1), final materialize is O(m*n)
```

---

### Template 11 — Prefix Product (with zeros)

```cpp
// LC 238: Product of Array Except Self
// Can't divide by zero, so we compute prefix and suffix products

vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> answer(n, 1);
    
    // Forward pass: answer[i] = product of all elements to the LEFT of i
    int prefix = 1;
    for (int i = 0; i < n; i++) {
        answer[i] = prefix;
        prefix *= nums[i];
    }
    
    // Backward pass: multiply by product of all elements to the RIGHT of i
    int suffix = 1;
    for (int i = n - 1; i >= 0; i--) {
        answer[i] *= suffix;
        suffix *= nums[i];
    }
    
    return answer;
}

// ZERO HANDLING:
// If there are two or more zeros, all products = 0.
// If there is exactly one zero at index k:
//   - answer[k] = product of all non-zero elements
//   - answer[i] = 0 for all i ≠ k
// The prefix/suffix product approach handles all cases automatically.
```

---

## VARIANTS DEEP-DIVE

### Variant 1: Maximum Subarray Sum with Prefix Sum Perspective

Kadane's algorithm is actually a prefix sum optimization in disguise.

```cpp
// Maximum subarray sum = max over all j of (prefix[j+1] - min(prefix[0..j]))
// As we scan left to right, maintain the minimum prefix sum seen so far.

int maxSubArray(vector<int>& nums) {
    int maxSum = INT_MIN;
    int minPrefix = 0;       // minimum prefix sum seen so far (prefix[0] = 0)
    int curr = 0;
    
    for (int x : nums) {
        curr += x;                          // curr = prefix sum up to here
        maxSum = max(maxSum, curr - minPrefix); // curr - minPrefix = best subarray ending here
        minPrefix = min(minPrefix, curr);   // update minimum prefix seen
    }
    
    return maxSum;
}

// This is EQUIVALENT to Kadane's, just viewed through the prefix sum lens.
// Kadane's standard form: dp[i] = max(nums[i], dp[i-1] + nums[i])
// Both give O(n) time, O(1) space.
```

---

### Variant 2: Subarray Sum Equals K with Negative Numbers

The prefix sum + hash map approach works WITH negative numbers — no modification needed. This is a key advantage over sliding window (which requires non-negative elements for the two-pointer to work).

```cpp
// Works for nums with negative numbers, zeros, positive numbers:
int subarraySum(vector<int>& nums, int target) {
    unordered_map<int, int> prefixCount;
    prefixCount[0] = 1;
    int curr = 0, result = 0;
    for (int x : nums) {
        curr += x;
        result += prefixCount[curr - target];
        prefixCount[curr]++;
    }
    return result;
}

// Example: nums = [-1, -1, 1], target = -1
// curr = -1: look for prefixCount[-1 - (-1)] = prefixCount[0] = 1 → result = 1
// curr = -2: look for prefixCount[-2 - (-1)] = prefixCount[-1] = 1 → result = 2
// curr = -1: look for prefixCount[-1 - (-1)] = prefixCount[0] = 1 → result = 3
// Answer: 3 (subarrays [-1], [-1], [-1,-1,1])
```

**Contrast with sliding window:** Sliding window only works when you can guarantee that extending the window always increases the sum (non-negative elements). Prefix sum + hash map has no such restriction.

---

### Variant 3: Longest Subarray with Sum = 0 (or = K)

```cpp
// Instead of counting subarrays, find the LONGEST one
// Change: instead of counting occurrences, record FIRST occurrence of each prefix sum

int longestSubarrayWithSumZero(vector<int>& nums) {
    unordered_map<int, int> firstSeen;
    firstSeen[0] = -1;       // Empty prefix at index -1 (before the array starts)
    
    int curr = 0, maxLen = 0;
    for (int i = 0; i < nums.size(); i++) {
        curr += nums[i];
        
        if (firstSeen.count(curr)) {
            // Subarray from firstSeen[curr]+1 to i has sum 0
            maxLen = max(maxLen, i - firstSeen[curr]);
        } else {
            firstSeen[curr] = i;    // Record FIRST occurrence only
        }
    }
    
    return maxLen;
}

// WHY "first occurrence" and not count?
// To MAXIMIZE length, we want the earliest starting index.
// If prefix[i] = prefix[j] with i < j, the subarray [i+1..j] has length j-i.
// For maximum length, we want minimum i, so record first occurrence only.
// For COUNT, we want all occurrences, so we count every time.
```

---

### Variant 4: Minimum Operations Using Difference Array (Range Addition)

```cpp
// LC 370: Range Addition
// You're given n operations (i, j, val): add val to all elements in [i, j].
// Return the final array.

vector<int> getModifiedArray(int length, vector<vector<int>>& updates) {
    vector<int> diff(length + 1, 0);
    
    for (auto& u : updates) {
        diff[u[0]] += u[2];
        diff[u[1] + 1] -= u[2];
    }
    
    // Prefix sum to recover final array
    for (int i = 1; i < length; i++) {
        diff[i] += diff[i-1];
    }
    
    diff.pop_back();    // Remove the extra sentinel element
    return diff;
}

// Time: O(k + n) where k = number of updates, n = array size
// Space: O(n)
// vs brute force: O(k * n) for k updates each touching O(n) elements
```

---

### Variant 5: Continuous Subarray Sum (Divisible by K, with Edge Cases)

```cpp
// LC 523: Does a continuous subarray of length >= 2 sum to a multiple of k?

bool checkSubarraySum(vector<int>& nums, int k) {
    // prefix[j] - prefix[i] divisible by k
    // ⟺ prefix[j] % k == prefix[i] % k
    // We need j - i >= 2 (subarray length >= 2)
    
    unordered_map<int, int> firstSeen;
    firstSeen[0] = -1;      // Empty prefix seen at index -1
    
    int curr = 0;
    for (int i = 0; i < nums.size(); i++) {
        curr = (curr + nums[i]) % k;
        
        if (firstSeen.count(curr)) {
            if (i - firstSeen[curr] >= 2) return true;  // Length >= 2
            // Don't update firstSeen[curr] — keep the earliest occurrence
        } else {
            firstSeen[curr] = i;
        }
    }
    
    return false;
}

// CRITICAL EDGE CASES:
// 1. k = 0: Can't do modulo by 0. Special handling required.
//    (Subarray sum divisible by 0 means sum = 0)
// 2. Length requirement >= 2: That's why we store firstSeen[0] = -1 not 0.
//    If prefix[i] % k = 0 at i=0 and i=1, then i - firstSeen[mod] = 1 - (-1) = 2 >= 2 ✓
// 3. Multiple zeros: nums=[0,0,0], k=anything. Subarray [0,0] sums to 0. Answer: true.
```

---

### Variant 6: Pivot Index (Sum Left = Sum Right)

```cpp
// LC 724: Find index where sum of elements to its left = sum to its right

int pivotIndex(vector<int>& nums) {
    int total = accumulate(nums.begin(), nums.end(), 0);
    int leftSum = 0;
    
    for (int i = 0; i < nums.size(); i++) {
        int rightSum = total - leftSum - nums[i];  // Sum to the right of i
        
        if (leftSum == rightSum) return i;
        
        leftSum += nums[i];
    }
    
    return -1;
}

// The single-pass insight: don't build prefix array explicitly.
// Track running leftSum. rightSum = total - leftSum - nums[i].
// No need for a prefix array here — one scan with running totals is enough.
```

---

### Variant 7: Subarray Sum Closest to Zero / Maximum Length Subarray Sum ≤ K

These require combining prefix sums with sorted structures (not just hash maps):

```cpp
// Maximum length subarray with sum <= K (for arrays with negative numbers)
// This needs sorted prefix sums + binary search (or merge sort)
// Simple approach for positive numbers only: sliding window (Pattern 03)

// For negative numbers: O(n log n) approach
int maxSubarrayLen(vector<int>& nums, int k) {
    int n = nums.size();
    vector<pair<int,int>> prefixes; // {prefix_sum, index}
    prefixes.push_back({0, -1});
    
    int curr = 0, maxLen = 0;
    for (int i = 0; i < n; i++) {
        curr += nums[i];
        // Find the minimum prefix sum j such that curr - prefix[j] <= k
        // i.e., prefix[j] >= curr - k
        // Binary search on sorted prefixes for first value >= curr - k
        int target = curr - k;
        auto it = lower_bound(prefixes.begin(), prefixes.end(), make_pair(target, INT_MIN));
        if (it != prefixes.end()) {
            maxLen = max(maxLen, i - it->second);
        }
        // Insert curr into sorted position
        auto pos = lower_bound(prefixes.begin(), prefixes.end(), make_pair(curr, i));
        prefixes.insert(pos, {curr, i});
    }
    
    return maxLen;
}
// Note: this O(n^2) worst case due to vector insert — use order statistics tree for O(n log n)
// In practice for this specific LC problem (positive only), sliding window is better
```

---

## COMPLEXITY ANALYSIS

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Build 1D prefix sum | O(n) | O(n) | Single pass through array |
| 1D range sum query | O(1) | O(1) | Single subtraction |
| Prefix sum + hash map (count subarrays) | O(n) | O(n) | Single pass, O(1) per hash map lookup average |
| Build 2D prefix sum | O(m×n) | O(m×n) | Two nested loops |
| 2D range sum query | O(1) | O(1) | Four array accesses + arithmetic |
| Difference array: single update | O(1) | O(1) | Two array writes |
| Difference array: materialize | O(n) | O(n) | Single prefix sum pass |
| k range updates + 1 read | O(k + n) | O(n) | k×O(1) updates + O(n) materialize |
| 2D difference array: single update | O(1) | O(1) | Four array writes |
| 2D difference array: materialize | O(m×n) | O(m×n) | Two passes (row + column prefix sum) |
| Prefix XOR (subarray XOR queries) | O(n) | O(n) | Same structure as prefix sum |

### When Prefix Sum Beats Brute Force By Orders of Magnitude

| Problem | Brute Force | With Prefix Sum |
|---------|-------------|-----------------|
| Count subarrays with sum K | O(n²) | O(n) |
| Range sum queries (Q queries) | O(Q×n) | O(n + Q) |
| k range updates + 1 read | O(k×n) | O(k + n) |
| 2D region sum (Q queries) | O(Q×m×n) | O(m×n + Q) |

The difference array win is most dramatic: if k = 10^5 updates each on range of 10^5 elements, brute force is 10^10 operations (TLE in any language), while difference array is 2×10^5 operations (instant).

---

## COMMON MISTAKES AT 1600 LEVEL

---

### Mistake 1: Forgetting `prefixCount[0] = 1`

**What they do:** Start the hash map empty and begin the scan loop directly.

**What goes wrong:** When `curr == target` at some position j (meaning the subarray from index 0 to j-1 sums to target), the lookup `prefixCount[curr - target] = prefixCount[0]` returns 0 instead of 1. You miss every subarray that starts at index 0. Your code passes most test cases but fails on inputs like `nums=[3,1,2], k=3` where the expected answer is 2 but you return 1.

**The fix:** Always initialize `prefixCount[0] = 1` before the loop. Read it three ways until it's automatic: (1) "empty prefix has sum 0," (2) "there is one way to take a prefix of sum 0 — take nothing," (3) "this handles subarrays starting at index 0."

```cpp
// WRONG — fails on subarrays starting at index 0:
unordered_map<int, int> prefixCount;
// MISSING: prefixCount[0] = 1
int curr = 0, result = 0;
for (int x : nums) {
    curr += x;
    result += prefixCount[curr - target];  // returns 0 when curr == target
    prefixCount[curr]++;
}

// CORRECT:
unordered_map<int, int> prefixCount;
prefixCount[0] = 1;                        // NON-NEGOTIABLE
int curr = 0, result = 0;
for (int x : nums) {
    curr += x;
    result += prefixCount[curr - target];  // now correctly counts when curr == target
    prefixCount[curr]++;
}
```

---

### Mistake 2: Wrong Update Order — Updating the Map BEFORE Querying

**What they do:** Write `prefixCount[curr]++` before `result += prefixCount[curr - target]` (the two lines in the opposite order).

**What goes wrong:** When `curr - target == curr` (i.e., target == 0), the count includes the current position as a "previous" prefix, creating phantom subarrays of length 0. More generally, the current prefix sum can match itself, generating false positives. The bug is subtle — it only manifests when target = 0 or when specific prefix sums collide.

**The fix:** Always look up FIRST, then update.

```cpp
// WRONG for COUNT problems — update before lookup:
for (int x : nums) {
    curr += x;
    prefixCount[curr]++;                      // update first
    result += prefixCount[curr - target];     // lookup second — can count the current prefix as "previous"
}

// CORRECT for COUNT — lookup first, then update:
for (int x : nums) {
    curr += x;
    result += prefixCount[curr - target];     // lookup first (current prefix not yet in map)
    prefixCount[curr]++;                      // then record current
}

// CORRECT for LONGEST — record first occurrence only:
for (int i = 0; i < n; i++) {
    curr += nums[i];
    if (!firstSeen.count(curr)) firstSeen[curr] = i;   // record ONLY if first time
    if (firstSeen.count(curr - target))
        maxLen = max(maxLen, i - firstSeen[curr - target]);
}
```

---

### Mistake 3: Negative Modulo in Prefix Sum Mod K Problems

**What they do:** Write `curr = (curr + x) % k` directly without handling negative values.

**What goes wrong:** In C++, the `%` operator truncates toward zero: `-3 % 5 = -3` (NOT 2). When the running prefix sum becomes negative (due to negative array elements), you get negative keys in your hash map. These negative keys never match the positive expected values, causing silent wrong answers with no crash or error.

**The fix:** Always normalize to [0, k-1] using `((x % k) + k) % k`.

```cpp
// WRONG: negative modulo causes wrong keys in hash map
for (int x : nums) {
    curr = (curr + x) % k;               // if x=-3, k=5, curr=1: (1-3)%5 = -2 (WRONG, should be 3)
    result += modCount[curr];
    modCount[curr]++;
}

// CORRECT: always normalize to positive remainder
for (int x : nums) {
    curr = ((curr + x) % k + k) % k;    // always in [0, k-1], even for negative x
    result += modCount[curr];
    modCount[curr]++;
}
// Why does this work? (a % k + k) % k:
// If a % k >= 0: (non-negative + k) % k = non-negative (the +k adds at most k, %k removes it)
// If a % k < 0: (negative + k) % k gives the positive equivalent
```

---

### Mistake 4: Off-by-One in 2D Prefix Sum Formula

**What they do:** Write the query formula using raw matrix coordinates without the +1 offset, or mix up which coordinates are row/column in the 4-term inclusion-exclusion.

**What goes wrong:** For a query (r1=0, c1=0, r2=1, c2=1) on a 2×2 matrix, using `P[r2][c2] - P[r1-1][c2] - P[r2][c1-1] + P[r1-1][c1-1]` with 0-indexed inputs accesses `P[-1][c2]` → undefined behavior. Or the signs are right but the indices are wrong — you get plausible but incorrect sums.

**The fix:** Use 1-indexed prefix array. The input (0-indexed) gets +1 in the prefix array access.

```cpp
// WRONG: assuming 1-indexed input but receiving 0-indexed matrix coordinates
int sumRegion(int r1, int c1, int r2, int c2) {
    return P[r2][c2] - P[r1-1][c2] - P[r2][c1-1] + P[r1-1][c1-1];
    // When r1=0: accesses P[-1][c2] → undefined behavior!
}

// CORRECT: 0-indexed input, 1-indexed prefix array
int sumRegion(int r1, int c1, int r2, int c2) {
    return P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1];
    // Full rect - top strip - left strip + top-left corner (subtracted twice)
}
// Mnemonic: Full minus Top minus Left plus Corner
```

---

### Mistake 5: Not Allocating the Extra Slot for Difference Array

**What they do:** Declare `vector<int> diff(n)` for a difference array.

**What goes wrong:** For a range update ending at index `r = n-1`, the code does `diff[r+1] -= val` which accesses `diff[n]` — out of bounds. This causes undefined behavior (sometimes a crash, sometimes a silent wrong answer that varies by run).

**The fix:** Always allocate `n+1` for difference arrays. The extra slot at index n absorbs the "undo" for updates ending at the last element.

```cpp
// WRONG: accessing diff[n] when r = n-1
vector<int> diff(n);
diff[l] += val;
diff[r + 1] -= val;   // r = n-1 → diff[n] → OUT OF BOUNDS

// CORRECT: one extra slot
vector<int> diff(n + 1, 0);   // size n+1
diff[l] += val;
diff[r + 1] -= val;   // r = n-1 → diff[n] → valid access
```

---

### Mistake 6: Using Prefix Sum When the Array Has Updates

**What they do:** Build a prefix sum array in setup, then later modify the original array and continue querying the prefix array.

**What goes wrong:** Prefix sums are STATIC. If `nums[2]` changes from 5 to 10, the prefix sum array still reflects the old value 5 at all indices >= 2. All subsequent range queries that include index 2 return wrong values — no crash, no warning, purely silent wrong answers.

**The fix:** Match the data structure to the update pattern.

```cpp
// WRONG: building prefix sum, then updating original array
vector<int> prefix(n+1);
// ... build prefix ...
nums[2] = 10;                        // prefix array is now stale at all indices >= 3
cout << query(prefix, 0, 5);         // WRONG: still uses old value at index 2

// CORRECT choices:
// Point updates + range queries → Fenwick tree (Pattern 28) or segment tree (Pattern 27)
// Range updates + single final read → difference array (this pattern)
// Range updates + range queries → lazy segment tree (Pattern 27)
// No updates → prefix sum (this pattern)
```

---

## PROBLEM SET

### WARMUP — 4 problems (build the foundation)

**1. Running Sum of 1d Array — LC 1480 — [Build prefix sum in place]**

Why this problem: The most direct possible application of prefix sums. You're literally building `prefix[i] = prefix[i-1] + nums[i]` with no hash map, no queries, no tricks. If you can't do this in 2 minutes, you don't have the pattern in your hands yet.

What to prove: You understand that `prefix[0] = nums[0]` (no separate "empty prefix" slot needed here since you return the modified nums). The transformation is in-place — no extra array required.

Target time: 2 minutes.

Edge cases: Single element (return as-is). Already all zeros (running sum is all zeros).

---

**2. Range Sum Query — Immutable — LC 303 — [Build once, query O(1)]**

Why this problem: The canonical "build once, query many" problem. Forces you to understand that prefix[i] represents the sum of the FIRST i elements (not the first i-1), making the range query formula `prefix[r+1] - prefix[l]`. This 1-indexed prefix array design with `prefix[0] = 0` eliminates all off-by-one errors.

What to prove: The constructor builds the prefix array. `sumRange(l, r)` returns `prefix[r+1] - prefix[l]`. The +1 offset and the empty prefix at index 0 are the key insight here.

Target time: 10 minutes.

Edge cases: Query covering the full array (l=0, r=n-1). Single element query (l==r).

---

**3. Find Pivot Index — LC 724 — [Prefix sum without building an array]**

Why this problem: Instead of building the full prefix array, you compute `leftSum` on the fly and derive `rightSum = totalSum - leftSum - nums[i]`. This is the "space-optimized" prefix sum pattern — recognizing that you only need the current prefix sum plus the total.

What to prove: `rightSum = totalSum - leftSum - nums[i]` (not `totalSum - leftSum`). If they're equal, `i` is the pivot. You must NOT include `nums[i]` in either side.

Target time: 10 minutes.

Edge cases: Pivot at index 0 (leftSum = 0, rightSum = totalSum - nums[0]). Pivot at last index. No pivot exists (return -1).

---

**4. Minimum Value to Get Positive Step by Step Sum — LC 1413 — [Running prefix sum, track minimum]**

Why this problem: The answer is `1 - minPrefixSum` (clamped to 1 if minPrefixSum >= 0). Trains you to track the minimum running sum as you scan left to right, which is a fundamental sub-operation in harder prefix sum problems.

Target time: 10 minutes.

Edge cases: All positive numbers (minPrefixSum is always > 0, answer is 1). Large negative numbers in the array.

---

### CORE — 5 problems (the real pattern learning)

**5. Subarray Sum Equals K — LC 560 — [prefixCount[0]=1, look up BEFORE updating]**

Why this problem: The single most important prefix sum problem. It appears in ~30% of Meta and Google interviews asking about subarrays. The two principles are non-negotiable: (1) `prefixCount[0] = 1` — must initialize before the loop — (2) look up `prefixCount[curr - k]` BEFORE updating `prefixCount[curr]++`.

What to prove: You can explain `prefixCount[0] = 1` three ways — "empty prefix," "subarrays starting at 0," "baseline for when prefix itself equals k." You can explain why negative numbers make sliding window impossible here: as you expand the window, the sum doesn't monotonically change, so you can't shrink from one side to fix an overshoot.

Target time: 20 minutes.

Edge cases: k = 0 (count subarrays with sum 0). Negative numbers in array. Single-element subarray equaling k.

Common mistake: Updating `prefixCount` BEFORE querying. This counts the current element as a "previous" prefix sum, adding phantom subarrays of length 0.

---

**6. Continuous Subarray Sum — LC 523 — [Prefix mod + first-seen index, length ≥ 2 edge case]**

Why this problem: Extends the prefix sum hash map pattern to MODULAR arithmetic. Two prefix sums at positions i and j have the same mod-k remainder if and only if the subarray `[i+1..j]` has sum divisible by k. You need to track FIRST occurrence (not count) because you need the longest/valid subarray.

What to prove: `prefixMod[0] = -1` (not 0 or missing) to handle subarrays starting at index 0 with the length ≥ 2 constraint. The length check `i - firstSeen[mod] >= 2`.

Target time: 25 minutes.

Edge cases: k = 1 (every subarray sum is divisible by 1 — need length ≥ 2, so answer is true if n ≥ 2). Two adjacent zeros. Negative numbers require the `((curr % k) + k) % k` fix.

---

**7. Range Sum Query 2D — Immutable — LC 304 — [2D prefix sum, inclusion-exclusion]**

Why this problem: The 2D generalization. The build formula adds a row, adds a column, subtracts the overlap (which was added twice), then adds the single cell. The query formula is inclusion-exclusion in reverse. Being able to draw this on a whiteboard is an interview skill.

What to prove: Build: `P[i][j] = mat[i-1][j-1] + P[i-1][j] + P[i][j-1] - P[i-1][j-1]`. Query (0-indexed input, 1-indexed prefix): `P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]`.

Target time: 25 minutes.

Edge cases: Single cell query (r1==r2, c1==c2). Full matrix query. The 1-indexed prefix array with extra row and column of zeros is the cleanest design — all boundary checks disappear.

---

**8. Product of Array Except Self — LC 238 — [Left prefix product × right suffix product]**

Why this problem: Classic "no division" constraint forces you to use prefix and suffix aggregates. Left sweep builds the prefix products; right sweep builds the suffix products and combines them into the answer. Space complexity: O(1) extra if you reuse the result array for the left pass.

What to prove: Left pass: `res[i] = product of nums[0..i-1]`. Right pass: multiply `res[i]` by `rightProduct` and update `rightProduct *= nums[i]`. The pattern of "left accumulation × right accumulation" appears in many other problems.

Target time: 20 minutes.

Edge cases: Zeros in the array (products involving the zero position are 0 except potentially the position at the zero itself). Multiple zeros.

---

**9. Count Number of Nice Subarrays — LC 1248 — [Parity transform + prefix sum hash map]**

Why this problem: Teaches you to transform the problem before applying prefix sums. Replace each number with its parity (0 for even, 1 for odd). Now "exactly k odd numbers in subarray" = "sum of transformed subarray equals k." Then the exact same `prefixCount[0] = 1` template from LC 560 applies directly. This transformation trick is reusable in many problems.

Target time: 20 minutes.

Edge cases: k = 0 (count subarrays with all even numbers). All numbers are odd. Single element equal to k=1.

---

### STRETCH — 4 problems (separate 1600 from 1800)

**10. Subarray Sums Divisible by K — LC 974 — [Prefix mod, negative mod fix]**

Why this problem: The negative modulo is the trap. In C++, `-3 % 5 = -3` (NOT 2). Without the fix `((curr % k) + k) % k`, your hash map will have negative keys that never match, silently producing wrong answers. This is one of the most common C++ gotchas in contest problems.

What to prove: You understand WHY `-3 % 5 = -3` in C++ (truncation toward zero), and WHY the fix `((x % k) + k) % k` always gives the correct positive remainder in [0, k-1].

Target time: 25 minutes.

Common mistake: Using `(curr + x) % k` directly. For x = -3, k = 5, curr = 1: `(1 + (-3)) % 5 = (-2) % 5 = -2`. Should be 3. Wrong key → wrong count.

---

**11. Maximum Subarray Sum with One Deletion — LC 1186 — [Prefix Kadane + Suffix Kadane]**

Why this problem: Combines prefix sum with DP. For each position i, the optimal subarray using one deletion ending at position i is: max of (no deletion, ending at i) OR (prefix ending at i-1 + suffix starting at i+1 using the element at i as the deleted one). Build forward-Kadane array and backward-Kadane array, then combine.

Target time: 30 minutes.

Edge cases: All negative (must keep at least one element — the "one deletion" means you can delete ONE element, not that you MUST). Single element.

---

**12. Range Addition — LC 370 — [Difference array, the canonical example]**

Why this problem: This IS the difference array template. Each update `[l, r, val]` becomes `diff[l] += val; diff[r+1] -= val`. The final array is the prefix sum of the difference array. Understanding this makes Corporate Flight Bookings and similar problems trivial.

Target time: 20 minutes. If this takes more than 15 minutes, re-read the difference array template until the "add at start, subtract at end+1" pattern is mechanical.

---

**13. Corporate Flight Bookings — LC 1109 — [Difference array, 1-indexed input]**

Why this problem: Identical structure to Range Addition but with 1-indexed flights. The main trap is index confusion — flights are 1-indexed (1 to n), so `diff[first-1] += seats` and `diff[last] -= seats` when your array is 0-indexed. Alternatively, use a size n+2 array with 1-indexed indexing throughout.

Target time: 20 minutes. If you solved LC 370, this is 5 minutes of index adjustment.

---

### CONTEST — 4 problems

**14. Number of Sub-arrays of Size K and Average ≥ Threshold — LC 1343 — [Prefix sum for fixed window sum check]**

Why this is contest level: Trains the "fixed window sum via prefix sum" pattern. For a window of size k, sum = `prefix[i+k] - prefix[i]`. If sum ≥ k × threshold, count it. Direct application of prefix sum on a fixed-size window, which is the building block for harder variable-window problems.

Target time: 15 minutes.

---

**15. Minimum Operations to Reduce X to Zero — LC 1658 — [Complement: find subarray with sum = total - x]**

Why this is contest level: You can't directly binary search or prefix-sum for "remove elements from both ends." The insight: removing elements summing to x from the ends is equivalent to keeping a middle subarray with sum = `total - x`. Now it's a sliding window problem (max length subarray with sum = total - x). The transformation from "remove from ends" to "keep middle" is a common contest trick.

Target time: 25 minutes.

---

**16. Ways to Split Array into Three Parts with Equal Sum — LC 1013 — [Two-pass prefix sum trick]**

Why this is contest level: First pass finds all positions where prefix sum = total/3. Second pass counts valid pairs. The subtlety: the third part must be non-empty, so the second split index must be strictly less than n-1. Off-by-one errors on the index range are the trap.

Target time: 20 minutes.

---

**17. Maximum Sum of Two Non-Overlapping Subarrays — LC 1031 — [Prefix max of subarray sums]**

Why this is contest level: For each split point between the two subarrays, you need "max subarray sum of length L ending before the split" and "max subarray sum of length M starting at or after the split." Pre-compute these in two passes using prefix sums + rolling maximum. Two orderings (L before M, M before L) must both be checked. This pre-computation + greedy combination pattern appears frequently in CF Div 2 C problems.

Target time: 30 minutes.

---

## PATTERN CONNECTIONS

| Connection | How They Combine |
|------------|-----------------|
| **Prefix Sum + Hash Map + Sliding Window** | For arrays with non-negative numbers, sliding window is often cleaner. For arrays with negatives, prefix sum + hash map is the only O(n) approach. Know when to use which. |
| **Prefix Sum + Binary Search** | "Count subarrays with sum ≥ K" can use sorted prefix sums + binary search for O(n log n). This pattern appears in harder problems. |
| **Prefix Sum + Monotonic Stack** | Some DP transitions require "minimum prefix sum in a range" which can use monotonic stack for amortized O(1). |
| **Difference Array + Greedy** | Range scheduling problems: use diff array to track overlapping intervals, then greedy on the resulting counts. |
| **2D Prefix Sum + DP** | "Count subrectangles with sum ≤ K" needs 2D prefix sum for O(1) queries inside an O(m²×n²) or better DP. |
| **Prefix XOR + Trie** | Maximum XOR of any subarray: prefix XOR + Bit Trie gives O(n log max_val). See Pattern 26. |
| **Prefix Sum as Fenwick Tree Basis** | A Fenwick tree is essentially a prefix sum structure that supports O(log n) updates. Mentally, BIT = prefix sum with update capability. |
| **Prefix Sum + Segment Tree** | Segment trees answer range aggregate queries (min, max, sum, XOR) in O(log n) — they generalize prefix sums to dynamic arrays. |

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given an array of integers nums and an integer k, return the total number of continuous subarrays whose sum equals k. The array may contain negative numbers."

**Company type:** Meta, Google phone screen (LC 560 — appears in ~30% of Meta onsite rounds)
**Expected time:** 20 minutes
**What they are testing:** Recognition that sliding window fails with negatives, prefix sum + hash map pattern, `prefixCount[0] = 1` initialization, correct update order

**Red flags that get candidates rejected:**
- Immediately jumping to sliding window and coding it — it flat-out doesn't work when the array has negative numbers; the interviewer will let you finish and then say "what about negative numbers?" wasting 10+ minutes
- Implementing O(n²) brute force and calling it done — shows you don't know the O(n) approach
- Forgetting `prefixCount[0] = 1` — this is the most common mistake; your code passes most test cases but fails on subarrays that start at index 0 (e.g., nums=[1,2,3], k=3 — expects 2 but returns 1)
- Updating `prefixCount[curr]++` BEFORE `result += prefixCount[curr - k]` — this counts the current prefix sum as a "previous" occurrence, adding phantom subarrays of length 0
- Not being able to explain WHY `prefixCount[0] = 1` even after writing it

**What they want you to say:**
"Since the array can have negative numbers, sliding window won't work — the sum doesn't change monotonically as we expand the window. I'll use prefix sums with a hash map. For each position j, I need to count how many previous positions i have `prefix[i] = prefix[j] - k`, meaning the subarray `[i..j-1]` sums to k. I preload `prefixCount[0] = 1` to account for subarrays starting at index 0 — it represents the empty prefix. I look up `prefixCount[curr - k]` first, then update the map."

**The follow-up:** "What if we want the LONGEST subarray with sum = k instead of the count?"
→ Change from counting occurrences to tracking FIRST occurrence. Use `firstSeen[prefix] = i` (only set if not already present). Then `maxLen = max(maxLen, i - firstSeen[curr - k])`.

**The trace for nums = [1, 1, 1], k = 2:**
- prefixCount = {0:1}, curr=0, result=0
- x=1: curr=1. Lookup prefixCount[1-2] = prefixCount[-1] = 0. result=0. prefixCount = {0:1, 1:1}
- x=1: curr=2. Lookup prefixCount[2-2] = prefixCount[0] = 1. result=1. prefixCount = {0:1, 1:1, 2:1}
- x=1: curr=3. Lookup prefixCount[3-2] = prefixCount[1] = 1. result=2. prefixCount = {0:1, 1:1, 2:1, 3:1}
- Return **2** (subarrays [1,1] at index 0-1 and index 1-2).

---

### Q2: "Design a class NumMatrix. On construction, take a 2D matrix. Implement sumRegion(r1,c1,r2,c2) that returns the sum of elements in the rectangle from (r1,c1) to (r2,c2). Assume many calls to sumRegion."

**Company type:** Amazon onsite, Google (LC 304 — tests 2D extension of a known technique)
**Expected time:** 25 minutes
**What they are testing:** 2D prefix sum construction, correct inclusion-exclusion query formula, clean constructor design

**Red flags that get candidates rejected:**
- Re-summing the rectangle on every query — O(m×n) per query — the problem says "many calls," making this TLE and showing you didn't read the constraints
- Writing the build formula incorrectly — the most common error is forgetting to subtract P[i-1][j-1] (the top-left corner that was subtracted twice)
- Writing the query formula with wrong signs — this is pure memorization failure; every sign error here is preventable by re-deriving inclusion-exclusion
- Confusing 0-indexed matrix with 1-indexed prefix array — using `matrix[i][j]` when you should be using `matrix[i-1][j-1]` in the build, or messing up the query's +1 offsets
- Not allocating the extra row and column of zeros — accessing P[-1][j] or P[i][-1] causes undefined behavior

**What they want you to say:**
"I'll build a 2D prefix sum in the constructor. P[i][j] stores the sum of all elements in the rectangle from (0,0) to (i-1,j-1). The build recurrence is: P[i][j] = matrix[i-1][j-1] + P[i-1][j] + P[i][j-1] - P[i-1][j-1]. Each query is O(1) inclusion-exclusion: P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]. The extra row and column of zeros eliminates all boundary checks."

**The follow-up:** "What if the matrix has updates?" → This breaks prefix sums entirely — you'd need to rebuild in O(m×n). Use a 2D Fenwick tree for O(log m × log n) per update and query.

**The whiteboard they'll ask you to draw:**
```
Full rectangle (0,0 to r2,c2)
MINUS top strip (0,0 to r1-1,c2)
MINUS left strip (0,0 to r2,c1-1)
PLUS top-left corner (0,0 to r1-1,c1-1)  ← added back because it was subtracted twice
```

---

### Q3: "There are n flights numbered 1 to n. You're given an array bookings where bookings[i] = [first_i, last_i, seats_i]. The i-th booking reserves seats_i seats on every flight from first_i to last_i. Return an array of length n, the total seats reserved for each flight."

**Company type:** Google, Microsoft (LC 1109 — commonly used for difference array pattern recognition)
**Expected time:** 20 minutes
**What they are testing:** Difference array recognition ("range update, read final array"), correct 1-indexed handling, O(k + n) approach vs. O(k × n) brute force

**Red flags that get candidates rejected:**
- Iterating over each booking and updating every flight in the range — O(k × n) which TLEs for large inputs; the entire point of difference arrays is to avoid this inner loop
- Not allocating `diff[n+2]` (one extra slot at both ends) — accessing `diff[n+1]` (when last=n) causes out-of-bounds in a size-n+1 array
- Mixing 0-indexed and 1-indexed — the flights are 1-indexed (1 to n); writing `diff[first] += seats` when you mean flight `first` means different things depending on your indexing
- Forgetting to run the prefix sum pass over the difference array to get the actual flight counts
- Building the final answer array with the wrong range — iterating 0 to n-1 but the flights are 1 to n

**What they want you to say:**
"This is a classic difference array problem — each booking is a range update, and we read the entire array once at the end. I allocate `diff[n+2]` (size n+2 for 1-indexed flights plus one sentinel slot). For each booking [first, last, seats]: `diff[first] += seats; diff[last+1] -= seats`. Then I take the prefix sum of diff to get the actual reservations per flight. Total time: O(k + n)."

**The follow-up:** "What if we also need to answer 'how many total seats are reserved on flights from a to b?' after building?" → Now we need range queries ON the flight reservation array — use a Fenwick tree or segment tree on top of the difference array result.

---

## MENTAL MODEL CHECKPOINT

**Q1:** What is `prefixCount[0] = 1` representing, and what breaks without it?
> **A:** It represents the empty prefix (sum = 0), which exists exactly once before the array starts. Without it, any subarray starting at index 0 with sum = target is missed, because looking up `prefixCount[curr - target]` when `curr = target` returns 0 instead of 1.

**Q2:** I have an array with negative numbers. Someone says "use sliding window for subarray sum = K." Is this right?
> **A:** No. Sliding window for subarray sum relies on the monotonicity that adding elements increases the sum. With negatives, adding an element can decrease the sum, so you can't use two-pointer shrinking logic. Use prefix sum + hash map instead (O(n), handles all integers).

**Q3:** Write the 2D prefix sum query formula from memory.
> **A:** `P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]`. Mnemonic: "Full minus top-strip minus left-strip plus top-left corner."

**Q4:** I have 10^5 range update operations and one final read. What data structure do I use?
> **A:** Difference array. Each update is O(1) (two writes). Final materialization is O(n). Total: O(k + n). Segment tree would be overkill — difference array is simpler and equally fast here.

**Q5:** What's wrong with `curr = (curr + x) % k` when x can be negative?
> **A:** In C++, the `%` operator for negative numbers returns a negative result (e.g., -3 % 5 = -3). We need the mathematical modulo which is always non-negative. Use `((curr + x) % k + k) % k` to normalize to [0, k-1].

**Q6:** For "longest subarray with sum = k," should I record first occurrence or last occurrence in my hash map? Why?
> **A:** First occurrence. We want to maximize `j - i` where prefix[i] = prefix[j] - k. To maximize `j - i` for a fixed `j`, we want to minimize `i`. Minimum `i` means earliest occurrence. Once we record `firstSeen[prefix]`, we never update it.

**Q7:** Prefix sum handles "range sum query with no updates." If I now need both "range sum query" AND "point update," what do I use?
> **A:** Fenwick tree (BIT) — O(log n) update + O(log n) query. Or segment tree — same complexity but more flexible (supports more operations like range min/max). Prefix sum alone can't handle updates.

---

## C++ STL CHEAT SHEET FOR THIS PATTERN

```cpp
#include <numeric>
#include <unordered_map>
#include <vector>
using namespace std;

// ============================================================
// QUICK PREFIX SUM BUILD
// ============================================================

vector<int> a = {1, 2, 3, 4, 5};

// Method 1: Manual build (explicit, preferred for clarity)
vector<int> prefix(a.size() + 1, 0);
for (int i = 1; i <= a.size(); i++)
    prefix[i] = prefix[i-1] + a[i-1];

// Method 2: std::partial_sum (exclusive scan for [l,r] query)
// NOTE: This gives prefix[i] = a[0] + a[1] + ... + a[i] (1-indexed, no leading 0)
vector<int> ps(a.size());
partial_sum(a.begin(), a.end(), ps.begin());
// ps = [1, 3, 6, 10, 15]
// Sum of [l, r]: ps[r] - (l > 0 ? ps[l-1] : 0)

// Method 3: std::exclusive_scan (C++17, gives the leading-0 version)
vector<int> ps2(a.size() + 1, 0);
exclusive_scan(a.begin(), a.end(), ps2.begin() + 1, 0);
// ps2 = [0, 1, 3, 6, 10, 15]
// Sum of [l, r]: ps2[r+1] - ps2[l]

// ============================================================
// TOTAL SUM
// ============================================================

int total = accumulate(a.begin(), a.end(), 0);
long long total_ll = accumulate(a.begin(), a.end(), 0LL); // Use 0LL for large arrays!

// ============================================================
// COMMON PREFIX SUM + HASH MAP PATTERN
// ============================================================

// Count subarrays with sum = target:
auto countSubarrays = [](vector<int>& nums, int target) {
    unordered_map<int, int> cnt;
    cnt[0] = 1;
    int curr = 0, result = 0;
    for (int x : nums) {
        curr += x;
        result += cnt[curr - target];  // default 0 if not found
        cnt[curr]++;
    }
    return result;
};

// Longest subarray with sum = target:
auto longestSubarray = [](vector<int>& nums, int target) {
    unordered_map<int, int> first;
    first[0] = -1;
    int curr = 0, maxLen = 0;
    for (int i = 0; i < nums.size(); i++) {
        curr += nums[i];
        if (first.count(curr - target))
            maxLen = max(maxLen, i - first[curr - target]);
        if (!first.count(curr))
            first[curr] = i;  // Only record first occurrence
    }
    return maxLen;
};

// ============================================================
// 2D PREFIX SUM — INLINE BUILD
// ============================================================
// Given matrix (m x n), build P (m+1 x n+1):

auto build2D = [](vector<vector<int>>& mat) {
    int m = mat.size(), n = mat[0].size();
    vector<vector<int>> P(m+1, vector<int>(n+1, 0));
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            P[i][j] = mat[i-1][j-1] + P[i-1][j] + P[i][j-1] - P[i-1][j-1];
    return P;
};

// Query: sum of [r1,c1] to [r2,c2] (0-indexed):
// P[r2+1][c2+1] - P[r1][c2+1] - P[r2+1][c1] + P[r1][c1]

// ============================================================
// DIFFERENCE ARRAY — QUICK INLINE
// ============================================================

auto rangeAdd = [](int n, vector<vector<int>>& ops) -> vector<int> {
    vector<int> diff(n + 1, 0);
    for (auto& op : ops) {
        diff[op[0]] += op[2];
        diff[op[1] + 1] -= op[2];
    }
    for (int i = 1; i < n; i++) diff[i] += diff[i-1];
    diff.pop_back();
    return diff;
};

// ============================================================
// NEGATIVE MODULO FIX
// ============================================================

// Safe modulo that always returns [0, k-1]:
int safeMod(int val, int k) {
    return ((val % k) + k) % k;
}

// ============================================================
// OVERFLOW WARNING
// ============================================================

// If nums[i] can be large (e.g., up to 10^4) and n up to 2×10^4:
// max prefix sum ≈ 10^4 × 2×10^4 = 2×10^8 — fits in int (max ~2.1×10^9)
// If nums[i] up to 10^9 and n up to 10^5:
// max prefix sum ≈ 10^9 × 10^5 = 10^14 — use long long!

vector<long long> prefix_ll(n + 1, 0LL);
long long curr_ll = 0;
unordered_map<long long, int> cnt_ll;
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

### Immediate next steps:
1. LC 560 (Subarray Sum Equals K) — if this takes you more than 15 minutes, re-read Template 2 and re-derive why `prefixCount[0] = 1` is needed.
2. LC 304 (Range Sum Query 2D) — implement the 2D prefix sum from scratch, no looking at templates. Draw the inclusion-exclusion on paper.
3. LC 1109 (Corporate Flight Bookings) — implement difference array from scratch in under 10 minutes.

### What to add to REVISION-TRACKER.md:
- Any problem where you forgot `prefixCount[0] = 1` → add both LC 560 and whatever problem reminded you
- Any 2D prefix sum off-by-one error → add LC 304 for revision
- LC 523 if you got the modulo wrong

### Connection to upcoming patterns:
- **Pattern 07 (Linked Lists):** No direct connection, but prefix sum thinking helps with "sum of two numbers stored as linked lists."
- **Pattern 10 (Heap):** Combining prefix sums with heap — "find the subarray of size k with maximum sum" is a classic sliding window, but heap + prefix sum for top-k subarray sums is a useful variant.
- **Pattern 12 (Graphs):** "Number of nodes within distance k" in a graph uses BFS, not prefix sums. But 1D/2D grid problems combine prefix sums with graph traversal.
- **Pattern 27 (Segment Tree) + Pattern 28 (Fenwick Tree):** These are the DYNAMIC version of prefix sums. Whenever you want range sum with updates, you'll use these. The mental model from prefix sums carries over directly.
- **Pattern 16-25 (DP):** Many DP transition optimizations boil down to answering "what is the minimum/maximum prefix sum in some range?" — a query that segment trees and monotonic deques can answer efficiently.

---

## SIGN-OFF CRITERIA

You have mastered Pattern 06 when:

**Tier 1 (Non-negotiable):**
- [ ] Can build 1D prefix sum and answer range queries from memory in under 2 minutes
- [ ] Can explain WHY `prefixCount[0] = 1` is needed (not just that it is)
- [ ] Know the 2D prefix sum build formula and query formula from memory
- [ ] Know when to use difference array vs prefix sum (update vs query)

**Tier 2:**
- [ ] LC 560 (Subarray Sum Equals K) — solved in under 15 minutes without hints
- [ ] LC 304 (Range Sum Query 2D) — correct on first try, correct formula
- [ ] LC 238 (Product of Array Except Self) — prefix × suffix, no division, O(1) extra space

**Tier 3:**
- [ ] LC 974 (Subarray Sums Divisible by K) — handled negative modulo correctly
- [ ] LC 1109 (Corporate Flight Bookings) — difference array, solved in under 10 minutes

**Sign-off test:** I'll give you one unseen problem from this pattern. You must:
1. Identify which sub-pattern applies (prefix sum + hashmap / 2D prefix sum / difference array / prefix XOR)
2. State whether `prefixCount[0] = 1` is needed and why
3. Write a complete C++ solution
4. State time and space complexity

Type `SIGN OFF P06` when ready.

---

*Pattern 06 Complete — Prefix Sums and Difference Arrays (1D, 2D, Hash Map, XOR, Difference)*
*Next: Pattern 07 — Linked Lists*
