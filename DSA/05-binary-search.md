# PATTERN 05 — BINARY SEARCH (COMPLETE)
## DSA Grandmaster Series | C++ | From 1600 → 2000+
## Tutor: Grandmaster + Principal Engineer Persona

---

## METADATA

| Field | Value |
|-------|-------|
| Pattern Number | 05 |
| Estimated Mastery Time | 5-6 days |
| Interview Frequency | ALWAYS |
| Contest Frequency | ALWAYS (every Div 2 has at least one BS problem) |
| Difficulty | FOUNDATION → ADVANCED (huge range within this single pattern) |
| Prerequisites | Arrays, basic math |
| Next Pattern | 06 — Prefix Sums and Difference Arrays |

---

## THE GRANDMASTER'S HONEST TAKE

Let me tell you what most tutorials won't say about binary search.

**The problem is not the algorithm. The problem is the off-by-one errors.**

Binary search is conceptually simple — eliminate half the search space each step. Most people learn this in week 1 and think they're done. Then they fail three interviews in a row because they wrote `lo <= hi` when they needed `lo < hi`, or `mid + 1` when they needed `mid`, and they don't know which is correct or WHY.

There are exactly **two forms** of binary search termination that matter. If you understand them deeply, every off-by-one error disappears forever. I'll teach you both, prove they're correct, and give you a decision rule for picking the right one.

**The second layer is Binary Search on the Answer.** This is where binary search goes from "known algorithm" to "superpower." You binary search on the ANSWER SPACE rather than the input array. The input might not even be sorted. You just need a monotonic predicate — a function `can(x)` that returns true for large x and false for small x (or vice versa). Suddenly you can solve "minimize the maximum", "find the smallest X such that condition holds", and countless other problems in O(n log n) that would be O(n²) otherwise.

**At the 1600 level, most people:**
- Can do classic binary search on sorted arrays
- Cannot do binary search on rotated arrays without bugs
- Do NOT recognize when a problem is secretly BS on answer
- Get confused by `lo = mid` vs `lo = mid + 1` and guess

**After this doc, you will:**
- Have two templates with PROOF of correctness
- Know exactly when to use each
- Identify BS-on-answer problems from their constraint/phrasing fingerprints
- Implement Koko Eating Bananas, Split Array Largest Sum, Median of Two Sorted Arrays with zero bugs

Contests like Codeforces Div 2 have at least one "BS on answer + greedy check" problem almost every single round. If you don't internalize this, you are capping yourself at Div 3.

---

## ELI5 — THE BOOKSHELF ANALOGY

Imagine a bookshelf with 1 million books sorted alphabetically. You want "The Count of Monte Cristo."

**Brute force:** Start from book 1, read every spine. Average 500,000 books checked.

**Binary search:** Go to the middle. "M." You need "C." It's to the left. Go to the middle of the left half. "F." Still to the right — wait, you need C, C is before F. Go left. Middle of that section: "D." C is before D. Go left. You're there. Took about 20 steps. Not 500,000.

**Why 20?** Because log₂(1,000,000) ≈ 20. Every step halves the remaining search space.

**The "answer space" twist:** Now imagine you don't have a sorted bookshelf. You have a pile of numbers and I ask: "What is the smallest batch size such that you can distribute all tasks to 3 workers in under 8 hours?"

You can't "search" a sorted array here. But here's the insight: if batch size 5 works, then batch size 6 ALSO works. If batch size 5 doesn't work, batch size 4 definitely doesn't. There's a threshold. THAT threshold is what you're binary searching for.

**The question transforms from:** "Find target in array" → "Find the boundary between FALSE and TRUE in a monotonic boolean function."

That's the entire intellectual content of advanced binary search.

---

## PATTERN FORMALLY DEFINED

### Definition 1: Classic Binary Search
Given a sorted array `a[0..n-1]` and a target `x`, find the index `i` such that `a[i] == x`, or determine it doesn't exist. Time: O(log n), Space: O(1).

### Definition 2: Lower Bound / Upper Bound
- **lower_bound(x):** Find the first index `i` such that `a[i] >= x`.
- **upper_bound(x):** Find the first index `i` such that `a[i] > x`.

If all elements are less than `x`, both return `n` (one past the end — the "not found" sentinel).

### Definition 3: Binary Search on Answer
Given a monotonic predicate `f: Z → {false, false, ..., false, true, true, ..., true}` (falses before trues, or trues before falses), find the boundary index. The answer space can be integers, floating points, or anything with a total order.

**Monotonic predicate examples:**
- `canFinish(speed)` — can Koko eat all bananas in H hours at eating speed `speed`? Monotonically true as `speed` increases.
- `canDistribute(maxPages)` — can we split a book into k parts each with ≤ maxPages pages? Monotonically true as `maxPages` increases.
- `countPairsAtLeast(dist)` — do at least m pairs of cows have distance ≥ dist? Monotonically false as `dist` increases.

---

## RECOGNITION SIGNALS

**Signal 1:** You see a sorted array and need to find a target or determine if it exists
→ Classic binary search, Template 1A (`lo <= hi`). The key test: does the comparison `nums[mid] vs target` give you a definitive direction? If you can say "target must be left of mid" or "right of mid" based purely on the value at mid, binary search applies. Sorting is the pre-condition that makes this decision valid.

**Signal 2:** "Find the first/last position of X," "find first occurrence," "find last occurrence" in a sorted array
→ Lower bound + upper bound combination, Template 1B. First occurrence = `lower_bound(target)`. Last occurrence = `upper_bound(target) - 1`. These exact phrasings are direct calls to Template 1B — recognize them immediately and don't try to do it in one pass.

**Signal 3:** The problem says "minimize the maximum" or "maximize the minimum"
→ Binary search on answer, Template 2A or 2B. These exact phrasings are the clearest possible signal. "Minimize the largest subarray sum," "maximize the minimum distance between balls" — any time you see these paired words, you're looking at a binary search on the answer space.

**Signal 4:** "What is the minimum X such that [condition]" and the condition is monotonic
→ Binary search on answer, Template 2A (find FIRST TRUE). Koko Eating Bananas: "minimum speed k such that Koko can finish in h hours." The condition is FFFTTTTT — once speed is sufficient, increasing speed keeps it sufficient. This monotonicity test is the intellectual core of the entire pattern.

**Signal 5:** "The array was sorted and then rotated at an unknown pivot"
→ Template 3 (rotated search) or Template 4 (rotated minimum). The key observation is that in any rotated sorted array, at least one half is ALWAYS fully sorted — the rotation creates exactly ONE descent. You identify the sorted half and check if the target falls within it.

**Signal 6:** "Find any peak," "find any local maximum," "find where the slope transitions from rising to falling"
→ Template 5 (peak element). The virtual -∞ boundaries guarantee at least one peak exists. You move toward the rising slope — wherever the slope rises, there must be a peak in that direction.

**Signal 7:** Constraints tell you the answer can be up to 10^9 or 10^14 — far too large to enumerate
→ Binary search on the answer space. 30 iterations cover 10^9; 47 iterations cover 10^14. Any time the answer space is too large to brute-force but you have a O(n) or O(n log n) feasibility check, binary search on the answer gets you to O(n log(answer_range)) total.

**Signal 8:** The problem has a clean "feasibility" structure — "can we do this in X time/cost/size?" and larger X always makes it easier
→ Binary search on answer. The feasibility function is the check. The monotonicity is that making X larger only helps (or only hurts), never both. Split Array Largest Sum: "can we split with maximum part sum ≤ X?" Larger X = easier = FFFFFFTTTT.

### Signals that LOOK like binary search but are NOT:

**Fake signal 1:** "Find the minimum element in an unsorted array"
→ Linear scan O(n). Binary search requires sorted data or a monotonic predicate. Without structure, you can't eliminate half the candidates with one comparison. `min_element` in O(n) is already optimal.

**Fake signal 2:** "Count subarrays with sum exactly equal to K" (with negative elements allowed)
→ Prefix sum + hash map (Pattern 06). The count doesn't change monotonically as K changes — it can go up and down. Binary search on K would give silently wrong answers.

**Fake signal 3:** "Sliding window maximum" or "maximum of every window of size k"
→ Monotonic deque (Pattern 29). The window slides forward, which binary search can't model. There's no "eliminate half" — you need every position.

**Fake signal 4:** "Find the K-th smallest element in an unsorted array"
→ QuickSelect O(n) average. Or heap O(n log k). Binary search on value + counting works (O(n log n)) but is suboptimal and harder to code correctly in interviews.

### The Monotonicity Test — Apply Before Every BS on Answer:

Before writing binary search on answer, explicitly verify:

> "If answer X satisfies the condition, does answer X+1 ALSO satisfy it (or X-1 also)?"

- Koko speed: speed 5 works → speed 6 works ✓
- Split array max sum: max sum 10 allows k splits → max sum 11 also allows k splits ✓
- Exact subarray count: 5 subarrays equal K → does 6 also? Not necessarily ✗

If you can find even ONE counterexample to monotonicity, binary search gives silently wrong answers. No error, no crash — just wrong output on edge cases.

---

## TEMPLATES IN C++

### Template 1A — Classic Binary Search (find target, `lo <= hi`)

```cpp
// Use when: you want to find EXACTLY one target value
// Terminates when lo > hi
// Loop invariant: answer (if exists) is in [lo, hi]

int binarySearch(vector<int>& nums, int target) {
    int lo = 0, hi = (int)nums.size() - 1;
    
    while (lo <= hi) {                          // INCLUSIVE range [lo, hi]
        int mid = lo + (hi - lo) / 2;           // Avoid overflow: NOT (lo + hi) / 2
        
        if (nums[mid] == target) {
            return mid;                          // Found it
        } else if (nums[mid] < target) {
            lo = mid + 1;                        // mid is too small, discard it
        } else {
            hi = mid - 1;                        // mid is too large, discard it
        }
    }
    
    return -1;                                   // Not found
}
```

**When `lo <= hi` and when to use `mid + 1` / `mid - 1`:**
- The loop maintains: the target, if it exists, is in `[lo, hi]`.
- When `nums[mid] != target`, we KNOW mid is not the answer, so we can safely exclude it.
- Hence `lo = mid + 1` (exclude mid from left side) and `hi = mid - 1` (exclude mid from right side).
- Terminates when `lo > hi` (empty range).

---

### Template 1B — Lower Bound (first position ≥ target, `lo < hi`)

```cpp
// Use when: find first index where a[i] >= target (C++ lower_bound)
// Returns index in [0, n] — returns n if no element >= target
// Loop invariant: answer is in [lo, hi]

int lowerBound(vector<int>& nums, int target) {
    int lo = 0, hi = (int)nums.size();          // hi = n, not n-1 (answer can be n)
    
    while (lo < hi) {                           // NOT lo <= hi
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] < target) {
            lo = mid + 1;                        // mid is definitely not the answer
        } else {
            hi = mid;                            // mid COULD be the answer, keep it
        }
    }
    
    return lo;                                   // lo == hi, this is the answer
}
```

**PROOF of correctness:**
- Invariant: answer is in `[lo, hi]`.
- When `nums[mid] < target`: mid cannot be the first index ≥ target, so answer is in `[mid+1, hi]`, set `lo = mid+1`.
- When `nums[mid] >= target`: mid COULD be the answer (it's ≥ target), so answer is in `[lo, mid]`, set `hi = mid`.
- Why not `hi = mid - 1`? Because mid might BE the answer. We can't discard it.
- Why `lo < hi` and not `lo <= hi`? When `lo == hi`, the range has exactly one candidate — that's our answer. We stop.
- Why does this terminate? Each iteration: either `lo` increases by at least 1 (when `lo = mid+1`), or `hi` decreases (when `hi = mid`). Since `mid = (lo+hi)/2 < hi` when `lo < hi`, `hi = mid < hi` always decreases. So `hi - lo` strictly decreases every iteration.

---

### Template 1C — Upper Bound (first position > target)

```cpp
// Returns first index where a[i] > target
// C++ equivalent of upper_bound

int upperBound(vector<int>& nums, int target) {
    int lo = 0, hi = (int)nums.size();          // hi = n
    
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] <= target) {              // NOTE: <= not <
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    
    return lo;
}
```

**The one-line difference:** Change `<` to `<=` in the condition to get upper_bound from lower_bound. This is worth memorizing as a pattern, not just a fact.

**Using lower_bound and upper_bound together:**
```cpp
// Count occurrences of target in sorted array
int countOccurrences(vector<int>& nums, int target) {
    int left  = lowerBound(nums, target);       // first index of target
    int right = upperBound(nums, target);       // first index AFTER target
    return right - left;
}

// Or use C++ STL:
auto left  = lower_bound(nums.begin(), nums.end(), target);
auto right = upper_bound(nums.begin(), nums.end(), target);
int count  = right - left;
```

---

### Template 2 — Binary Search on Answer (find minimum X where predicate is true)

```cpp
// Pattern: FFFFFFTTTTTTT — find first TRUE
// Use for: "minimize X such that condition(X) is true"
// The check function MUST be monotonic

// GENERIC TEMPLATE — fill in your problem's lo, hi, and check()
int binarySearchOnAnswer() {
    int lo = MIN_POSSIBLE_ANSWER;    // smallest value that COULD be valid
    int hi = MAX_POSSIBLE_ANSWER;    // largest value that could be valid
    
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        
        if (check(mid)) {
            hi = mid;               // mid works, but maybe something smaller works
        } else {
            lo = mid + 1;           // mid doesn't work, we need something bigger
        }
    }
    
    return lo;                      // lo == hi, this is the minimum X that works
}
```

**For "maximize X such that condition(X) is true" (TTTTTFFFF — find last TRUE):**
```cpp
int binarySearchOnAnswerMax() {
    int lo = MIN_POSSIBLE_ANSWER;
    int hi = MAX_POSSIBLE_ANSWER;
    
    while (lo < hi) {
        int mid = lo + (hi - lo + 1) / 2;  // NOTE: +1 in mid to avoid infinite loop!
        
        if (check(mid)) {
            lo = mid;               // mid works, maybe something bigger also works
        } else {
            hi = mid - 1;           // mid doesn't work, answer is smaller
        }
    }
    
    return lo;
}
```

**The +1 trap explained:**
When you do `lo = mid` (not `lo = mid + 1`), if `lo = 5, hi = 6`:
- `mid = (5+6)/2 = 5` (floor division)
- If `check(5)` is true: `lo = mid = 5`. But lo was already 5! Infinite loop!
- Fix: `mid = (lo + hi + 1) / 2 = 6`. Now `lo = 6`, which is strictly greater. Loop terminates.

**Rule of thumb:** 
- When `lo = mid + 1` is possible → use `mid = (lo + hi) / 2` (standard)
- When `lo = mid` is possible → use `mid = (lo + hi + 1) / 2` (ceiling)

---

### Template 3 — Search in Rotated Sorted Array

```cpp
// Key insight: in a rotated array, one half is ALWAYS fully sorted
// Identify which half is sorted, then determine which half to search

int searchRotated(vector<int>& nums, int target) {
    int lo = 0, hi = (int)nums.size() - 1;
    
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] == target) return mid;
        
        // Left half is sorted
        if (nums[lo] <= nums[mid]) {
            // Is target in the sorted left half?
            if (nums[lo] <= target && target < nums[mid]) {
                hi = mid - 1;           // Yes, search left
            } else {
                lo = mid + 1;           // No, search right
            }
        }
        // Right half is sorted
        else {
            // Is target in the sorted right half?
            if (nums[mid] < target && target <= nums[hi]) {
                lo = mid + 1;           // Yes, search right
            } else {
                hi = mid - 1;           // No, search left
            }
        }
    }
    
    return -1;
}
```

**Why `nums[lo] <= nums[mid]` with <=?**
- When `lo == mid` (two-element array), left half is trivially "sorted" (single element).
- The `<=` handles this edge case correctly.

**Rotated array with duplicates (LC 81):**
```cpp
// When nums[lo] == nums[mid], we can't determine which side is sorted
// Worst case O(n), but still correct

bool searchRotatedWithDuplicates(vector<int>& nums, int target) {
    int lo = 0, hi = (int)nums.size() - 1;
    
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] == target) return true;
        
        // Can't determine sorted half — shrink from both sides
        if (nums[lo] == nums[mid] && nums[mid] == nums[hi]) {
            lo++; hi--;
        }
        else if (nums[lo] <= nums[mid]) {
            if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {
            if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    
    return false;
}
```

---

### Template 4 — Find Minimum in Rotated Sorted Array

```cpp
// The minimum is at the "rotation point" — where the descent from hi to lo occurs
// Key insight: the minimum is in the UNSORTED half

int findMin(vector<int>& nums) {
    int lo = 0, hi = (int)nums.size() - 1;
    
    while (lo < hi) {                           // lo < hi, not lo <= hi
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] > nums[hi]) {
            // mid is in the left (higher) portion → minimum is to the RIGHT
            lo = mid + 1;
        } else {
            // nums[mid] <= nums[hi] → minimum is at mid or to the LEFT
            hi = mid;
        }
    }
    
    return nums[lo];                            // lo == hi, that's the minimum
}
```

**Intuition:** Compare `nums[mid]` with `nums[hi]` (not `nums[lo]`!).
- If `nums[mid] > nums[hi]`: The array "descends" from mid to the right side, meaning the rotation point (minimum) is somewhere in `[mid+1, hi]`.
- If `nums[mid] <= nums[hi]`: The right side is "normal" (increasing), so minimum is at `mid` or earlier.

---

### Template 5 — Find Peak Element

```cpp
// Peak = element greater than both neighbors
// Key insight: if nums[mid] < nums[mid+1], there's a peak to the RIGHT
// This is because nums[mid+1] is "rising" — either it's the peak or there's a
// higher peak further right (with the boundary condition nums[n] = -inf)

int findPeakElement(vector<int>& nums) {
    int lo = 0, hi = (int)nums.size() - 1;
    
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] < nums[mid + 1]) {
            // Rising slope → peak is to the right (mid is NOT the peak)
            lo = mid + 1;
        } else {
            // Falling slope → peak is at mid or to the left
            hi = mid;
        }
    }
    
    return lo;
}
```

---

### Template 6 — Binary Search on Floating Point Answer

```cpp
// Use when answer is a real number (not integer)
// Stop when range is small enough (epsilon-based termination)
// OR use a fixed number of iterations (100 iterations = ~2^-100 precision)

double binarySearchFloat() {
    double lo = 0.0, hi = 1e9;              // adjust bounds per problem
    
    // Method 1: epsilon-based (careful — can loop forever if epsilon too small)
    while (hi - lo > 1e-9) {               // precision: 9 decimal places
        double mid = (lo + hi) / 2.0;
        
        if (check(mid)) {
            hi = mid;
        } else {
            lo = mid;
        }
    }
    
    return lo;
    
    // Method 2: fixed iterations (SAFER, always terminates, sufficient precision)
    // for (int iter = 0; iter < 100; iter++) {
    //     double mid = (lo + hi) / 2.0;
    //     if (check(mid)) hi = mid;
    //     else lo = mid;
    // }
    // return lo;
}
```

---

### Template 7 — Binary Search on Answer: Koko Eating Bananas (full implementation)

```cpp
// LC 875: Koko has n piles of bananas. Eats at most k bananas/hour.
// What is the minimum k to finish all piles in h hours?

class Solution {
    // Check: can Koko finish all bananas at speed k within h hours?
    bool canFinish(vector<int>& piles, long long k, int h) {
        long long hours = 0;
        for (int p : piles) {
            hours += (p + k - 1) / k;      // ceil(p / k) without floating point
            if (hours > h) return false;    // Early termination
        }
        return hours <= h;
    }
    
public:
    int minEatingSpeed(vector<int>& piles, int h) {
        int lo = 1;                         // Minimum speed: 1 banana/hour
        int hi = *max_element(piles.begin(), piles.end());  // Max: finish biggest pile in 1 hour
        
        // Binary search for minimum k where canFinish is true
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            
            if (canFinish(piles, mid, h)) {
                hi = mid;                   // mid works, try smaller
            } else {
                lo = mid + 1;               // mid too slow, need bigger
            }
        }
        
        return lo;
    }
};
```

---

### Template 8 — Binary Search on Answer: Split Array Largest Sum (full implementation)

```cpp
// LC 410: Split array into m non-empty parts. Minimize the maximum subarray sum.
// This is the "minimize the maximum" archetype.

class Solution {
    // Check: can we split nums into at most m parts, each with sum <= maxSum?
    bool canSplit(vector<int>& nums, int m, long long maxSum) {
        int parts = 1;
        long long currentSum = 0;
        
        for (int x : nums) {
            if (x > maxSum) return false;   // Single element exceeds maxSum
            if (currentSum + x > maxSum) {
                // Start a new part
                parts++;
                currentSum = x;
                if (parts > m) return false;
            } else {
                currentSum += x;
            }
        }
        
        return true;
    }
    
public:
    int splitArray(vector<int>& nums, int m) {
        // Answer range:
        // lo = max single element (can't do better — every part has at least one element)
        // hi = total sum (put everything in one part)
        long long lo = *max_element(nums.begin(), nums.end());
        long long hi = accumulate(nums.begin(), nums.end(), 0LL);
        
        while (lo < hi) {
            long long mid = lo + (hi - lo) / 2;
            
            if (canSplit(nums, m, mid)) {
                hi = mid;               // mid works, try smaller maximum
            } else {
                lo = mid + 1;           // mid too small, need bigger maximum
            }
        }
        
        return (int)lo;
    }
};
```

---

### Template 9 — Median of Two Sorted Arrays (Binary Search on Partition)

```cpp
// LC 4: O(log(min(m, n))) solution
// Core idea: binary search on the partition point in the SMALLER array
// The partition automatically determines the partition in the larger array

class Solution {
public:
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
        // Always binary search on the smaller array
        if (nums1.size() > nums2.size()) {
            return findMedianSortedArrays(nums2, nums1);
        }
        
        int m = nums1.size(), n = nums2.size();
        int lo = 0, hi = m;                     // partition1 can be from 0 to m
        
        while (lo <= hi) {
            int partition1 = lo + (hi - lo) / 2;
            int partition2 = (m + n + 1) / 2 - partition1;  // ensures left halves have (m+n+1)/2 elements total
            
            // Elements just left and right of partitions
            int maxLeft1  = (partition1 == 0) ? INT_MIN : nums1[partition1 - 1];
            int minRight1 = (partition1 == m) ? INT_MAX : nums1[partition1];
            int maxLeft2  = (partition2 == 0) ? INT_MIN : nums2[partition2 - 1];
            int minRight2 = (partition2 == n) ? INT_MAX : nums2[partition2];
            
            if (maxLeft1 <= minRight2 && maxLeft2 <= minRight1) {
                // Found the correct partition
                if ((m + n) % 2 == 0) {
                    // Even total: average of two middle elements
                    return (max(maxLeft1, maxLeft2) + min(minRight1, minRight2)) / 2.0;
                } else {
                    // Odd total: middle element is max of left halves
                    return max(maxLeft1, maxLeft2);
                }
            } else if (maxLeft1 > minRight2) {
                hi = partition1 - 1;            // partition1 too far right, move left
            } else {
                lo = partition1 + 1;            // partition1 too far left, move right
            }
        }
        
        return -1.0; // unreachable
    }
};
```

**The deep insight:** We're looking for partition points `partition1` and `partition2` such that:
- `maxLeft1 <= minRight2` (nothing from left of A is bigger than right of B)
- `maxLeft2 <= minRight1` (nothing from left of B is bigger than right of A)

This ensures the left half of the merged array consists of exactly the right elements.

---

## VARIANTS DEEP-DIVE

### Variant 1: Find First and Last Position (LC 34)

This is lower_bound + upper_bound applied together.

```cpp
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        int n = nums.size();
        
        // Lower bound: first index where nums[i] >= target
        int lo = 0, hi = n;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] < target) lo = mid + 1;
            else hi = mid;
        }
        int first = lo;
        
        // Check if target exists at all
        if (first == n || nums[first] != target) return {-1, -1};
        
        // Upper bound: first index where nums[i] > target
        hi = n;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] <= target) lo = mid + 1;  // NOTE: <=
            else hi = mid;
        }
        int last = lo - 1;                          // last occurrence = upper_bound - 1
        
        return {first, last};
    }
};
```

**Critical note:** `last = upperBound - 1`. Upper bound gives you one past the last occurrence.

---

### Variant 2: Search Insert Position (LC 35)

This is exactly lower_bound.

```cpp
int searchInsert(vector<int>& nums, int target) {
    int lo = 0, hi = nums.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] < target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}
// Returns the index where target should be inserted to keep array sorted.
// If target exists, returns its first occurrence.
// Same as lower_bound — learn this pattern ONCE, apply everywhere.
```

---

### Variant 3: Find Minimum in Rotated Sorted Array II (with duplicates, LC 154)

```cpp
int findMin(vector<int>& nums) {
    int lo = 0, hi = nums.size() - 1;
    
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        
        if (nums[mid] > nums[hi]) {
            lo = mid + 1;
        } else if (nums[mid] < nums[hi]) {
            hi = mid;
        } else {
            // nums[mid] == nums[hi]: can't determine which side the minimum is on
            // Shrink hi by 1 (not hi = mid, because mid might = hi and we'd loop)
            hi--;
        }
    }
    
    return nums[lo];
}
```

**Worst case O(n)** when all elements are equal (e.g., [1,1,1,1,1]).

---

### Variant 4: Capacity to Ship Packages (LC 1011)

This is the same archetype as Koko but with a "consecutive allocation" check:

```cpp
class Solution {
    bool canShip(vector<int>& weights, int days, int capacity) {
        int daysNeeded = 1, currentLoad = 0;
        for (int w : weights) {
            if (w > capacity) return false;     // Single package exceeds capacity
            if (currentLoad + w > capacity) {
                daysNeeded++;
                currentLoad = w;
            } else {
                currentLoad += w;
            }
        }
        return daysNeeded <= days;
    }
    
public:
    int shipWithinDays(vector<int>& weights, int days) {
        int lo = *max_element(weights.begin(), weights.end());
        int hi = accumulate(weights.begin(), weights.end(), 0);
        
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (canShip(weights, days, mid)) hi = mid;
            else lo = mid + 1;
        }
        
        return lo;
    }
};
```

---

### Variant 5: Aggressive Cows / Magnetic Force Between Two Balls (LC 1552)

This is the "maximize the minimum" variant — the OPPOSITE of the usual minimize-the-maximum.

```cpp
// LC 1552: Place m balls in positions[]. Maximize minimum distance between any two.

class Solution {
    // Can we place m balls such that each pair has distance >= minDist?
    bool canPlace(vector<int>& positions, int m, int minDist) {
        int count = 1;                          // Place first ball at positions[0]
        int last = positions[0];
        
        for (int i = 1; i < positions.size(); i++) {
            if (positions[i] - last >= minDist) {
                count++;
                last = positions[i];
                if (count == m) return true;
            }
        }
        
        return count >= m;
    }
    
public:
    int maxDistance(vector<int>& position, int m) {
        sort(position.begin(), position.end());
        
        int lo = 1;                             // Minimum possible distance
        int hi = position.back() - position[0]; // Maximum possible distance
        
        // Find MAXIMUM minDist where placement is possible
        // Pattern: TTTTTFFFFF → find last TRUE
        while (lo < hi) {
            int mid = lo + (hi - lo + 1) / 2;  // CEILING mid (avoid infinite loop)
            
            if (canPlace(position, m, mid)) {
                lo = mid;                       // mid works, try larger
            } else {
                hi = mid - 1;
            }
        }
        
        return lo;
    }
};
```

**This is the EXACT OPPOSITE template from Koko/Split Array:**
- Koko: minimize max → `canFinish ? hi = mid : lo = mid + 1` (find first TRUE)
- Cows: maximize min → `canPlace ? lo = mid : hi = mid - 1` (find last TRUE, use ceiling mid)

Memorize both and you cover 90% of BS-on-answer problems.

---

### Variant 6: Nth Root / Square Root (Floating Point BS)

```cpp
// Find sqrt(x) to 6 decimal places
double sqrtBS(double x) {
    double lo = 0, hi = max(1.0, x);           // Note: sqrt(0.25) = 0.5 < 0.25, so lo=0
    
    for (int iter = 0; iter < 100; iter++) {   // 100 iterations = ~2^-100 precision
        double mid = (lo + hi) / 2.0;
        if (mid * mid <= x) lo = mid;
        else hi = mid;
    }
    
    return lo;
}

// Integer version (LC 69)
int mySqrt(int x) {
    if (x == 0) return 0;
    int lo = 1, hi = x;
    
    while (lo < hi) {
        long long mid = lo + (hi - lo + 1) / 2; // Ceiling mid (lo = mid pattern)
        if (mid * mid <= (long long)x) lo = mid;
        else hi = mid - 1;
    }
    
    return lo;
}
```

---

### Variant 7: Binary Search in 2D Matrix (LC 74)

```cpp
// Sorted 2D matrix: rows sorted, row[i][0] > row[i-1][last]
// Treat as flattened 1D sorted array, convert indices back

bool searchMatrix(vector<vector<int>>& matrix, int target) {
    int m = matrix.size(), n = matrix[0].size();
    int lo = 0, hi = m * n - 1;
    
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        int val = matrix[mid / n][mid % n];     // 1D index → 2D index conversion
        
        if (val == target) return true;
        else if (val < target) lo = mid + 1;
        else hi = mid - 1;
    }
    
    return false;
}
```

---

### Variant 8: C++ STL Binary Search Functions

These are worth knowing cold:

```cpp
#include <algorithm>

vector<int> a = {1, 3, 3, 5, 7, 9};

// binary_search: does target exist?
bool exists = binary_search(a.begin(), a.end(), 3);  // true

// lower_bound: iterator to first element >= target
auto it1 = lower_bound(a.begin(), a.end(), 3);       // points to index 1
int idx1 = it1 - a.begin();                          // 1

// upper_bound: iterator to first element > target
auto it2 = upper_bound(a.begin(), a.end(), 3);       // points to index 3
int idx2 = it2 - a.begin();                          // 3

// Count occurrences: upper_bound - lower_bound
int count = upper_bound(a.begin(), a.end(), 3) - lower_bound(a.begin(), a.end(), 3);  // 2

// Equal range: returns pair(lower_bound, upper_bound)
auto [lo, hi] = equal_range(a.begin(), a.end(), 3);  // [1, 3)

// On SORTED custom objects:
struct Item { int val; };
vector<Item> items = {{1}, {3}, {5}};
auto it = lower_bound(items.begin(), items.end(), Item{3},
                      [](const Item& a, const Item& b) { return a.val < b.val; });
```

**With `std::set` and `std::map` (ordered containers have built-in O(log n) methods):**
```cpp
set<int> s = {1, 3, 5, 7};

// lower_bound on set: O(log n)
auto it = s.lower_bound(4);     // points to 5
auto it2 = s.upper_bound(5);    // points to 7

// DO NOT use std::lower_bound(s.begin(), s.end(), 4) — that's O(n)!
// ALWAYS use s.lower_bound(4) for sets/maps — that's O(log n)!
```

---

## COMPLEXITY ANALYSIS

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Classic binary search | O(log n) | O(1) | Halve search space each step: log₂(n) halving steps |
| Lower bound / upper bound | O(log n) | O(1) | Same as binary search — each step reduces [lo, hi] by ~half |
| Binary search on answer | O(log(hi-lo) × f(n)) | O(1) | log₂ steps × cost of check function |
| Koko Eating Bananas | O(n log max(piles)) | O(1) | log steps × O(n) check |
| Split Array Largest Sum | O(n log(sum)) | O(1) | log steps × O(n) check |
| Search in rotated array | O(log n) | O(1) | Still halve each step, just with more case analysis |
| Find peak element | O(log n) | O(1) | Move toward peak (steepest direction), halve each step |
| Median of two sorted arrays | O(log min(m,n)) | O(1) | Binary search on smaller array partition |
| Floating point BS (100 iter) | O(100 × f(n)) | O(1) | 100 fixed iterations ≈ enough for 1e-30 precision |
| 2D matrix BS | O(log(m×n)) | O(1) | Treat as 1D sorted array |
| STL lower_bound (vector) | O(log n) | O(1) | Random access iterator — can jump to mid |
| STL lower_bound (list) | O(n) | O(1) | No random access — degrades to linear scan |
| set::lower_bound | O(log n) | O(1) | Tree structure, each step goes down one level |

### The O(log n) Intuition — Never Forget This

Why is log₂(10⁹) ≈ 30? Because 2³⁰ = 1,073,741,824 > 10⁹.

This means: even if your answer space is 10⁹, you only need 30 iterations of binary search. Every O(n log n) algorithm exploiting this will comfortably pass within 10⁸ operations.

The real question is always: "What is my check function cost?" If check is O(n), total is O(n log n). If check is O(n²), total is O(n² log n) — you might need a faster check.

---

## COMMON MISTAKES AT 1600 LEVEL

---

### Mistake 1: Integer Overflow in `mid` Calculation

**What they do:** Write `int mid = (lo + hi) / 2`.

**What goes wrong:** When doing binary search on answer with large ranges (e.g., lo = 0, hi = 10^9), `lo + hi` can overflow a 32-bit signed integer (max ~2.1 × 10^9). The result becomes negative, mid becomes negative, array access goes out of bounds or infinite loop results.

**The fix:** Always write `mid = lo + (hi - lo) / 2`. Mathematically equivalent, never overflows.

```cpp
// WRONG: can overflow when lo and hi are both large
int mid = (lo + hi) / 2;

// CORRECT: safe from overflow
int mid = lo + (hi - lo) / 2;

// For long long ranges (answer up to 10^18):
long long lo = 0, hi = 2e18;
long long mid = lo + (hi - lo) / 2;  // must use long long throughout
```

---

### Mistake 2: `lo <= hi` vs `lo < hi` — Using the Wrong Termination

**What they do:** Use `lo <= hi` with `hi = mid` for lower bound / BS on answer templates.

**What goes wrong:** When `lo == hi` and `hi = mid`:
- `mid = (lo + lo) / 2 = lo`
- `hi = mid = lo` — nothing changed
- `lo <= hi` is still true — **INFINITE LOOP**

The off-by-one only shows on specific edge cases (small arrays, target at index 0 or n-1), so it passes some tests and fails others.

**The fix:** Apply the decision rule:
- Finding **exact target**: `lo <= hi` with `lo = mid+1`, `hi = mid-1` (exclude confirmed non-answers)
- Finding **boundary** (lower bound, BS on answer): `lo < hi` with one branch using `hi = mid` (can't discard potential answers)

```cpp
// WRONG for lower bound — infinite loop when lo == hi:
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;
    if (nums[mid] < target) lo = mid + 1;
    else hi = mid;                   // lo == hi → mid == lo → hi stays lo → stuck
}

// CORRECT for lower bound:
while (lo < hi) {                    // stops when lo == hi (one candidate left)
    int mid = lo + (hi - lo) / 2;
    if (nums[mid] < target) lo = mid + 1;
    else hi = mid;                   // shrinks range while keeping candidate
}
```

---

### Mistake 3: Wrong Semantic Bounds for `lo` and `hi` in BS on Answer

**What they do:** Set `lo = 0` or `hi = 1e9` or `hi = INT_MAX` without thinking about what the answer MEANS.

**What goes wrong:**
- `lo = 0` for Koko: speed 0 means 0 bananas/hour → Koko never finishes → check returns true infinitely (or division by zero crash). lo = 0 is not a valid answer semantically.
- `lo = 0` for Split Array: no subarray of positive integers has sum 0 → check returns false for every mid → lo marches all the way to hi → answer is wrong, no error.
- `hi = INT_MAX`: wastes extra iterations and signals you didn't reason about the problem.

**The fix:** Derive lo and hi from the problem's semantics, not from "safe large numbers."

```cpp
// Koko Eating Bananas — WRONG lo:
int lo = 0;   // speed 0 is invalid — causes division by zero

// CORRECT:
int lo = 1;   // minimum useful speed
int hi = *max_element(piles.begin(), piles.end());
// hi = max pile: any higher speed still takes 1 hour per pile — useless to go higher

// Split Array Largest Sum — WRONG lo:
long long lo = 0;   // no part with positive integers has sum 0

// CORRECT:
long long lo = *max_element(nums.begin(), nums.end());
// Every part has at least one element; the largest element is the floor
long long hi = accumulate(nums.begin(), nums.end(), 0LL);
// If m=1, everything is one part — this is the ceiling
```

---

### Mistake 4: Forgetting Ceiling Mid for the "Maximize" Template

**What they do:** Use floor mid (`lo + (hi-lo)/2`) in the maximize template where `lo = mid` on true.

**What goes wrong:** When `lo = 5, hi = 6`:
- Floor mid = 5
- `check(5)` is true → `lo = mid = 5` — **nothing changed**
- `lo < hi` is still true → **INFINITE LOOP**

**The fix:** Use ceiling mid `(lo + (hi-lo+1) / 2)` in the maximize template.

```cpp
// Maximize minimum distance — WRONG (floor mid):
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;      // floor mid
    if (check(mid)) lo = mid;           // lo=5 when lo was 5: stuck forever
    else hi = mid - 1;
}

// CORRECT (ceiling mid):
while (lo < hi) {
    int mid = lo + (hi - lo + 1) / 2;  // ceiling mid
    if (check(mid)) lo = mid;           // lo=6 when lo was 5: guaranteed progress
    else hi = mid - 1;
}

// Memory aid: "Maximize → ceiling. Minimize → floor."
```

---

### Mistake 5: Comparing `nums[lo]` Instead of `nums[hi]` for Rotated Minimum

**What they do:** Write `if (nums[mid] >= nums[lo])` for finding the minimum in a rotated array.

**What goes wrong:** For a non-rotated array `[1, 2, 3, 4, 5]`:
- `nums[lo]=1 <= nums[mid]=3` is always true
- They set `lo = mid + 1` repeatedly, always moving right
- End up at index 4 (value 5) — completely wrong. The minimum is 1.

**The fix:** Compare `nums[mid]` with `nums[hi]`, not `nums[lo]`. The rotation's anomaly is relative to the RIGHT end: if `nums[mid] > nums[hi]`, the descent is to the right of mid.

```cpp
// WRONG: compare with nums[lo] — fails for non-rotated arrays
if (nums[mid] > nums[lo]) lo = mid + 1;
else hi = mid;

// CORRECT: compare with nums[hi]
if (nums[mid] > nums[hi]) lo = mid + 1;  // descent is in right half → min there
else hi = mid;                            // min is at mid or in left half
```

---

### Mistake 6: Applying Binary Search When Predicate Is Not Monotonic

**What they do:** Binary search on answer for a problem where the check function is NOT monotonic.

**What goes wrong:** Binary search requires FFFFFFTTTT or TTTTFFFFF. If the actual truth values look like FTFTFT or have any non-monotonic pattern, binary search converges to an arbitrary boundary and returns a wrong answer with no error or crash. This is the hardest mistake to debug because everything "looks correct" in the code.

**Example:** "Find the window size K such that the window contains EXACTLY sum S." For arrays with negative numbers, the window sum can go up and down as K grows — not monotonic. Binary search gives silently wrong answers.

**The fix:** Before writing any BS on answer code, explicitly verify monotonicity:

```cpp
// Before coding, ask: "If check(X) = true, is check(X+1) GUARANTEED to also be true?"
// If you can find ONE counterexample, binary search is wrong.

// WRONG: trying to BS on non-monotonic check
// "Find window size K where window has exactly sum S"
// Sum can decrease as K grows (negative numbers) → not monotonic → BS gives wrong answer

// RIGHT: test your intuition
// "If window of size K has sum >= S, does K+1 ALSO have sum >= S?"
// For non-negative arrays: YES (adding more non-negative values keeps sum >= S). Binary search OK.
// For arrays with negatives: NOT necessarily. Binary search WRONG.
```

---

## PROBLEM SET

### WARMUP — 4 problems (build the template into muscle memory)

**1. Binary Search — LC 704 — [Pure Template 1A, no tricks]**

Why this problem: This is the template. No rotation, no answer space, no tricks — just sorted array and target. If you cannot write Template 1A in 2 minutes from memory without bugs, stop everything and drill this until you can. Every other binary search problem is built on this foundation.

What to prove: `mid = lo + (hi-lo)/2` (not `(lo+hi)/2`). Both branches use `mid±1` to strictly shrink the range. Returns `-1` correctly when the loop exits without finding the target.

Target time: 2 minutes.

Edge cases: Empty array (return -1 immediately). Single element that is not target. Target smaller than all elements. Target larger than all elements.

---

**2. Search Insert Position — LC 35 — [Lower bound IS Template 1B]**

Why this problem: This problem is the definition of lower_bound. The return value is either the index of the target or the index where the target would be inserted to keep the array sorted. That is exactly what lower_bound returns. Implement it with `hi = nums.size()` (not `nums.size()-1`) to handle the "insert at the end" case.

What to prove: `hi = nums.size()` (the answer can be index n — insertion at end). `lo < hi` termination. `hi = mid` when `nums[mid] >= target` (keeping mid as a candidate).

Target time: 3 minutes.

Edge cases: Target is larger than all elements → return n. Target equals nums[0] → return 0. All duplicates: the return is the FIRST index of the target.

---

**3. Guess Number Higher or Lower — LC 374 — [BS on implicit range, no array]**

Why this problem: Builds mental flexibility for binary search on answer. There is no physical array. The "array" is the integers 1..n in sorted order. You call `guess(mid)` instead of comparing `nums[mid]`. The insight is that sorted structure can be implicit — this is exactly what happens in BS-on-answer problems where you search over a range of possible values.

Target time: 3 minutes.

Edge cases: n = 1 (only one number, return 1 immediately after checking). `guess(mid) == 0` means you found it — return `mid` immediately, don't continue searching.

Common mistake: Calling `guess` with values outside [1, n]. Your lo and hi should always stay within [1, n].

---

**4. First Bad Version — LC 278 — [Your first real BS on answer]**

Why this problem: The "array" is `[1, 2, 3, ..., n]`. The predicate is `isBadVersion(v)`, which is FFFTTTTT — false for good versions, true from the first bad version onwards. This is the BS-on-answer template in its purest form. Every Koko/Split Array/Capacity-to-Ship problem is a more complex version of this exact structure.

What to prove: `hi = mid` (NOT `hi = mid-1`) when `isBadVersion(mid)` returns true — mid might be the first bad version, so we must keep it as a candidate. `lo = mid + 1` when `isBadVersion(mid)` returns false — mid is definitely NOT the first bad version.

Target time: 5 minutes.

Edge cases: All versions are bad (n=1, version 1 is bad → return 1). The last version is the first bad one (first bad = n).

---

### CORE — 5 problems (the real pattern learning)

**5. Find First and Last Position of Element in Sorted Array — LC 34 — [Lower bound + upper bound]**

Why this problem: The single most important lower_bound application. Forces you to implement two binary searches and combine them. The `last = upperBound - 1` transformation must become automatic — if you have to derive it each time, you haven't internalized it.

What to prove: First occurrence = `lower_bound(target)`. Last occurrence = `upper_bound(target) - 1`. When target doesn't exist: `first == n` OR `nums[first] != target` → return `{-1, -1}`. Without the second check, you get wrong answers on arrays where target is not present but lower_bound returns a valid index.

Target time: 15 minutes.

Edge cases: Target not in array (return {-1,-1}). All elements equal target. Single occurrence. Target at index 0 or n-1.

Common mistake: Running upper_bound starting from index 0 when you already found `first` — it works but wastes log(n) comparisons and shows you didn't think about the optimization. Start upper_bound from `first`.

---

**6. Search in Rotated Sorted Array — LC 33 — [Identify the sorted half]**

Why this problem: Tests the single most important insight about rotated arrays: **in any rotated sorted array, at least one half is ALWAYS fully sorted**. The rotation creates exactly one "descent." That descent is either in the left half or the right half — making the other half a clean sorted sequence you can binary search in.

What to prove: `nums[lo] <= nums[mid]` (with `<=` for lo==mid edge case) → left half is sorted. Check if target is within `[nums[lo], nums[mid])` → search left, else right. The symmetric logic for the right-sorted case.

Target time: 20 minutes.

The thing to do before coding: **Trace through [4,5,6,7,0,1,2] with target=0 manually on paper.** The logic only "clicks" after a concrete trace. Don't attempt to code from pure reasoning.

Edge cases: Two-element arrays (lo=0, hi=1 → mid=0 → the `<=` in `nums[lo] <= nums[mid]` matters here). Array that was never rotated (rotation at index 0 is equivalent to no rotation — your code must handle this).

---

**7. Find Minimum in Rotated Sorted Array — LC 153 — [Compare with nums[hi], not nums[lo]]**

Why this problem: This is the source of one of the most common confusion pairs in binary search — LC 33 uses `nums[lo]` for the rotation detection, but LC 153 MUST use `nums[hi]`. Understanding WHY they're different cements the rotation array understanding.

Key insight: To find the minimum, you want to move toward the "descending" side — where the rotation dipped the values low. If `nums[mid] > nums[hi]`, the minimum is in the right half (the descent is there). If `nums[mid] <= nums[hi]`, minimum is at mid or in the left half.

What to prove: Trace `[1,2,3,4,5]` (non-rotated). With `nums[lo]`: `nums[mid]=3 > nums[lo]=1` is always true → always go right → end at 5 (WRONG). With `nums[hi]`: `nums[mid]=3 <= nums[hi]=5` → go left → correctly find 1.

Target time: 15 minutes.

Edge cases: Non-rotated array (must still return index 0). All same elements: LC 154 variant requires `hi--` when `nums[mid] == nums[hi]`.

---

**8. Find Peak Element — LC 162 — [Move toward rising slope]**

Why this problem: Binary search on an array that is NOT sorted and doesn't even have a target. You're finding a PROPERTY — local maximum. The insight: if you're on a rising slope (`nums[mid] < nums[mid+1]`), a peak must exist to the right (either the next element is a peak or the slope continues rising and eventually must peak due to the -∞ virtual boundary). This argument — "move toward the guaranteed direction" — is fundamental to many non-obvious binary searches.

Target time: 15 minutes.

Edge cases: Single element (it's always a peak). Strictly increasing array (peak is the last element). Strictly decreasing array (peak is the first element). Two adjacent peaks (any one of them is fine — problem only requires ANY peak).

---

**9. Koko Eating Bananas — LC 875 — [BS on answer, canonical minimize template]**

Why this problem: The canonical BS-on-answer problem. Every aspect of Template 2A must come from reasoning, not memory: lo = 1 (speed 0 is useless), hi = max(piles) (faster than this doesn't help), `canFinish` check uses ceiling division `(p + k - 1) / k`.

What to prove: You can derive lo, hi, and the check function independently from the problem statement. The check function sums ceiling-divided hours and compares to h. You use `lo < hi` and `hi = mid` (not `hi = mid - 1`) when check passes.

Target time: 20 minutes.

Edge cases: h = piles.size() (each hour eats exactly one pile — speed = max pile is needed). A single pile with very large value. Early termination in check when `hours > h` (prevents overflow with large piles).

---

### STRETCH — 4 problems (separate 1600 from 1800)

**10. Capacity to Ship Packages Within D Days — LC 1011 — [Consecutive allocation, same skeleton as Koko]**

Why this problem: Template 2A again, but the check function is "consecutive allocation" (packages must be shipped in order, can't reorder). This forces a fresh derivation of the check — you can't reuse Koko's `ceil(pile/speed)` directly. You greedily fill each day's capacity in order.

What makes this "stretch": Beginners try to sort the packages first (WRONG — order matters). The consecutive constraint forces you to scan left to right and start a new day only when the next package doesn't fit.

Target time: 20 minutes. If you understood Koko fully, this is 10 minutes. The extra time is for people who need to re-derive the greedy check.

---

**11. Split Array Largest Sum — LC 410 — [Correct lo = max_element, NOT 0]**

Why this problem: The hardest thing here is not the code — it's the lo initialization. `lo = 0` passes syntax checks but is semantically wrong: no subarray of positive integers can have sum 0. The check function would incorrectly claim even the first positive element exceeds maxSum=0, returning false for every mid value, so lo marches to hi without finding anything meaningful.

What to prove: You derive `lo = *max_element(nums.begin(), nums.end())` (minimum possible for any part) and `hi = accumulate(...)` (maximum: one part containing everything). Your check handles the `x > maxSum → return false` guard for the single-element-exceeds-maxSum case.

Target time: 25 minutes.

Edge cases: m = 1 (answer is sum of all). m = n (answer is max element — the tightest constraint). A single element larger than all others dominates.

---

**12. Sqrt(x) — LC 69 — [Maximize template, ceiling mid, long long guard]**

Why this problem: "Largest integer k where k² ≤ x" is a "last TRUE" problem (TTTTFFFFF). This requires Template 2B with ceiling mid. If you use floor mid, you get an infinite loop on lo=5, hi=6 when check(5)=true. You MUST cast to long long before squaring — `(long long)mid * mid` — because `mid` can be up to 46340 and `mid*mid` overflows int.

Target time: 10 minutes. If this takes more than 15 minutes, the maximize template isn't internalized yet. Re-read Template 2B, trace through lo=5, hi=6 with both floor and ceiling mid, and repeat.

---

**13. Median of Two Sorted Arrays — LC 4 — [BS on partition, hardest BS problem]**

Why this problem: Binary search on the partition POSITION, not on a value or answer. You're searching for the correct way to split both arrays such that every element in the left halves is ≤ every element in the right halves. The INT_MIN/INT_MAX sentinel values, the even/odd total length handling, and the "always BS on the smaller array" optimization all require deep understanding.

What to prove: p2 = (m+n+1)/2 - p1. The `+1` is for odd total lengths. The partition is correct when `maxL1 <= minR2` AND `maxL2 <= minR1`.

Target time: 40 minutes. Do NOT attempt this without first tracing through the example `nums1=[1,3], nums2=[2]` on paper with the partition logic.

---

### CONTEST — 4 problems (CF Div 2 C/D level)

**14. Magnetic Force Between Two Balls — LC 1552 — [Maximize minimum, ceiling mid mandatory]**

Why this is contest level: "Maximize the minimum distance" is the exact OPPOSITE of Koko — it uses Template 2B (ceiling mid, `lo = mid`). Getting the ceiling mid wrong causes a silent infinite loop that looks like TLE. You must sort `position[]` first — the greedy check places balls greedily at the earliest valid position, and greedy requires sorted order.

Target time: 25 minutes. If this takes more, the maximize template needs more drilling.

---

**15. Minimize Max Distance to Gas Station — LC 774 — [Floating point BS on answer]**

Why this is contest level: The answer is a real number. The check function computes `ceil(gap / maxDist) - 1` new stations needed per gap. Using the 100-iteration fixed approach is safer than epsilon termination — fixed iterations always terminate with ~10^-30 precision, while epsilon loops can behave poorly near floating-point representation limits.

Target time: 30 minutes.

---

**16. Find in Mountain Array — LC 1095 — [Three separate BS passes]**

Why this is contest level: Three binary searches in one problem: find peak (Template 5), then binary search ascending left side (Template 1A), then binary search descending right side (Template 1A with reversed comparisons). The mountain array is an opaque interface — you can only call `get(index)` with a limited call budget, so binary search is not just efficient but required.

Target time: 30 minutes.

---

**17. Find K-th Smallest Pair Distance — LC 719 — [BS on answer + two-pointer check]**

Why this is contest level: Two patterns combined. Binary search on the distance value d; check function counts pairs with distance ≤ d using a sorted array + two-pointer sliding window. You cannot do the count in O(1) — it requires O(n) per check. Total: O(n log n) sort + O(n log maxDist) for the BS × O(n) check = O(n log maxDist). This pattern — "BS on answer + non-trivial O(n) check" — is exactly what appears in Codeforces Div 2 D problems.

Target time: 35 minutes.

---

## PATTERN CONNECTIONS

Binary search connects to almost every other pattern in this curriculum:

| Connection | How They Combine |
|------------|-----------------|
| **Greedy + BS on Answer** | Check function is a greedy feasibility test. Koko, Split Array, Capacity to Ship are all `O(n)` greedy checks inside `O(log n)` BS. |
| **Sorting + BS** | Sort array first, then BS on it. Lower/upper bound are useless on unsorted arrays. Find K-th Smallest uses sorted order + BS on value. |
| **Two Pointers + BS** | Two pointer can replace BS for some O(n log n) → O(n) improvements. But in contests, BS on answer often uses a two-pointer greedy check. |
| **Sliding Window + BS** | Find K-th Smallest Pair Distance: BS on answer, check with sliding window counting. |
| **Prefix Sums + BS** | "Find leftmost index with prefix sum ≥ target" — classic combination. Used in Kth Smallest Number in Multiplication Table. |
| **Segment Tree / BIT + BS** | "Find first position with prefix sum ≥ x" on a Fenwick tree — O(log² n) using BS on BIT, O(log n) using the "walk down BIT" technique. |
| **DP + BS** | LIS in O(n log n) uses binary search (patience sorting). Binary search finds insertion point in the `tails` array. |

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Koko loves to eat bananas. There are n piles of bananas. The guards will be back in h hours. Find the minimum eating speed k to finish all the bananas."

**Company type:** Google phone screen, Amazon OA, Microsoft (LC 875 — extremely common)
**Expected time:** 20 minutes
**What they are testing:** Recognition of BS on answer, independent derivation of check function and bounds, ceiling division without floating point

**Red flags that get candidates rejected:**
- Brute-force loop over all possible speeds from 1 to 10^9, checking each one — O(n × max_pile) which TLEs; worse, it shows you didn't recognize the monotonicity
- Setting `lo = 0`: speed 0 means eating 0 bananas per hour, meaning Koko never finishes — division by zero in the check function
- Setting `hi = 10^9` or `hi = INT_MAX` without reasoning — shows you're copying a template without understanding what the bounds mean semantically
- Using floating point `(double)p / k` for the ceiling division — imprecise, can give wrong answers due to floating point rounding
- Not adding an early exit in the check function when `hours > h` — not technically wrong but shows you're not thinking about performance or overflow
- Spending more than 3 minutes without identifying this as binary search on answer

**What they want you to say:**
"The eating speed k ranges from 1 to max(piles). Any speed higher than max(piles) is useless — at that point, each pile still only takes 1 hour to eat. The condition 'can Koko finish in h hours at speed k' is monotonically non-decreasing as k increases — if speed 5 works, speed 6 also works. So I binary search for the minimum k where the condition is true. For the check function, I sum ceil(pile/k) for each pile. I use the ceiling formula (p + k - 1) / k to avoid floating point."

**The follow-up they WILL ask:** "What is the time complexity?" → O(n log M) where M = max(piles) ≤ 10^9. log₂(10^9) ≈ 30. Total: 30n operations.

**The trace for piles = [3, 6, 7, 11], h = 8:**
- lo=1, hi=11. mid=6. check(6): ⌈3/6⌉+⌈6/6⌉+⌈7/6⌉+⌈11/6⌉ = 1+1+2+2 = 6 ≤ 8 → true. hi=6.
- lo=1, hi=6. mid=3. check(3): ⌈3/3⌉+⌈6/3⌉+⌈7/3⌉+⌈11/3⌉ = 1+2+3+4 = 10 > 8 → false. lo=4.
- lo=4, hi=6. mid=5. check(5): ⌈3/5⌉+⌈6/5⌉+⌈7/5⌉+⌈11/5⌉ = 1+2+2+3 = 8 ≤ 8 → true. hi=5.
- lo=4, hi=5. mid=4. check(4): ⌈3/4⌉+⌈6/4⌉+⌈7/4⌉+⌈11/4⌉ = 1+2+2+3 = 8 ≤ 8 → true. hi=4.
- lo=4, hi=4. Exit. Return **4**.

---

### Q2: "Given a sorted array of integers, find the first and last position of a given target value. Your algorithm must run in O(log n) time."

**Company type:** Amazon phone screen, Microsoft, Google, Meta (LC 34 — appears in 30%+ of interviews asking sorted array questions)
**Expected time:** 15 minutes
**What they are testing:** Lower bound + upper bound combination, correct handling of "target not found," off-by-one precision

**Red flags that get candidates rejected:**
- Binary searching to find ANY occurrence, then linearly scanning left and right — O(n) worst case; the problem explicitly requires O(log n), so this is a hard failure
- Using `lo <= hi` with `hi = mid` in the lower bound implementation — creates an infinite loop when lo == hi (they often don't notice because it works for some test cases but fails on others)
- Returning `{lo, lo-1}` directly from upper_bound without the `last = ub - 1` reasoning — shows mechanical copying vs. understanding
- Forgetting to check `nums[first] != target` after computing first — leads to wrong answer when target is between two array values
- Setting `hi = nums.size() - 1` instead of `hi = nums.size()` in the lower bound — misses the case where target should be at index n (insert at end)

**What they want you to say:**
"I'll run two binary searches. The first is lower_bound — it finds the first index where nums[i] >= target. If that index is n or nums[index] != target, the target doesn't exist. The second is upper_bound — the first index where nums[i] > target. Last occurrence = upper_bound - 1. Both use the `lo < hi` template with `hi = mid` for the shrinking branch."

**The follow-up:** "What if the target appears exactly once?" → lower_bound and upper_bound differ by exactly 1. last = ub - 1 = lb. Returns {lb, lb}. Correct.

**Second follow-up:** "What if you want the count of occurrences in O(log n)?" → `upper_bound(target) - lower_bound(target)`. This is exactly what STL's `equal_range` computes.

---

### Q3: "Given an integer array nums and an integer k, split nums into k non-empty subarrays such that the maximum subarray sum is minimized. Return this minimized maximum."

**Company type:** Google onsite (L4/L5), Meta, hard Amazon rounds (LC 410 — tests senior-level thinking)
**Expected time:** 30 minutes
**What they are testing:** Recognition of "minimize the maximum" → BS on answer, correct semantic derivation of bounds, greedy feasibility check under pressure

**Red flags that get candidates rejected:**
- Jumping to DP immediately — DP O(nk) works but is harder to code correctly under time pressure and shows you went for the memorized solution instead of the elegant one; interviewers will ask "can you do better?"
- Setting `lo = 0` — a subarray with positive integers cannot have sum 0; if every single element > maxSum=0, the check function returns false for all mid values and lo marches to hi unconverged
- Setting `hi = INT_MAX` or `1e18` without reasoning — acceptable but shows you're guessing, not deriving; interviewers notice this
- Not handling the `if (x > maxSum) return false` guard inside the check — when a single element exceeds maxSum, no valid split exists for this answer; omitting it produces wrong answers silently
- Getting the part-counting logic backwards — counting "cuts" vs "parts" is an off-by-one error that passes most test cases but fails on edge inputs
- Writing DP first and then "optimizing" to binary search — shows reactive thinking; they want you to recognize the pattern upfront

**What they want you to say:**
"This is 'minimize the maximum' — the classic signal for binary search on the answer. I binary search on the maximum subarray sum. Lower bound: each part must contain at least one element, so the largest single element is the minimum possible answer. Upper bound: if k=1, we put everything in one part, giving sum(nums). The check function greedily packs elements: when adding the next element would exceed maxSum, start a new part. Count parts and verify ≤ k."

**The trace for nums = [7,2,5,10,8], k = 2:**
- lo=10, hi=32. mid=21. check(21,2): 7→9→14→[14+10=24>21, new part: parts=2, curr=10]→[10+8=18≤21]. parts=2 ≤ 2 → true. hi=21.
- lo=10, hi=21. mid=15. check(15,2): 7→9→14→[14+10=24>15, new part: parts=2, curr=10]→[10+8=18>15, new part: parts=3>2] → false. lo=16.
- lo=16, hi=21. mid=18. check(18,2): 7→9→14→[14+10=24>18, new part: parts=2, curr=10]→[10+8=18≤18]. parts=2 ≤ 2 → true. hi=18.
- lo=16, hi=18. mid=17. check(17,2): 7→9→14→[14+10=24>17, new part: parts=2, curr=10]→[10+8=18>17, new part: parts=3>2] → false. lo=18.
- lo=18, hi=18. Exit. Return **18**.

---

## MENTAL MODEL CHECKPOINT

Answer these 7 questions. If you can't answer any one of them in 30 seconds, re-read the corresponding section.

**Q1:** I'm writing lower_bound. I write `hi = mid` instead of `hi = mid - 1`. Why?
> **A:** Because `mid` might BE the answer (it satisfies `nums[mid] >= target`). We can't discard it. We set `hi = mid` to narrow the window while keeping `mid` as a candidate.

**Q2:** I have `lo = 5, hi = 6` and I'm doing "maximize minimum" (lo = mid template). What mid do I compute and why?
> **A:** `mid = lo + (hi - lo + 1) / 2 = 5 + (1 + 1) / 2 = 6`. Use CEILING mid. If I used floor mid = 5, and check(5) is true, I'd set `lo = 5` → no progress → infinite loop.

**Q3:** I'm doing BS on answer. What are the `lo` and `hi` for "Split Array Largest Sum" with m parts?
> **A:** `lo = max_element(nums)` (minimum possible for any subarray — at least one element), `hi = sum(nums)` (maximum when everything is one subarray).

**Q4:** Binary search on rotated array: why do I check `nums[lo] <= nums[mid]` with `<=` and not `<`?
> **A:** When `lo == mid` (two-element array, e.g., lo=0, hi=1 → mid=0), we need to correctly identify the left half as "sorted" (a single element is trivially sorted). The `<=` handles this edge case.

**Q5:** I see a problem: "Find the minimum number of days to wait such that you can make all flowers bloom." How do I know if it's binary searchable?
> **A:** Ask: "If waiting d days works, does waiting d+1 days also work?" Yes — more time is always at least as good. Monotonic → binary searchable.

**Q6:** What does `equal_range` return in C++ STL, and what's it equivalent to?
> **A:** Returns `{lower_bound, upper_bound}` as a pair of iterators. Equivalent to running both separately. Gives you the range `[lower, upper)` where all elements equal the target.

**Q7:** For floating-point binary search, why do I prefer 100 fixed iterations over `while (hi - lo > 1e-9)`?
> **A:** Fixed iterations always terminate and give precision 2⁻¹⁰⁰ ≈ 10⁻³⁰. The epsilon approach can loop indefinitely if the epsilon is too small for floating-point representation, and can give too little precision if epsilon is wrong for the problem's requirement.

---

## C++ STL BINARY SEARCH CHEAT SHEET

```cpp
// ============================================================
// ALL BINARY SEARCH STL FUNCTIONS — QUICK REFERENCE
// ============================================================

#include <algorithm>
#include <vector>
#include <set>
#include <map>
using namespace std;

vector<int> v = {1, 2, 3, 3, 3, 5, 7};    // SORTED (required)

// 1. Does target EXIST?
bool exists = binary_search(v.begin(), v.end(), 3);     // true

// 2. FIRST position where v[i] >= target (lower bound)
int lb = lower_bound(v.begin(), v.end(), 3) - v.begin();  // 2
int lb4 = lower_bound(v.begin(), v.end(), 4) - v.begin(); // 5 (first 5)

// 3. FIRST position where v[i] > target (upper bound)
int ub = upper_bound(v.begin(), v.end(), 3) - v.begin();  // 5

// 4. Count occurrences of target
int cnt = upper_bound(v.begin(), v.end(), 3) - lower_bound(v.begin(), v.end(), 3); // 3

// 5. Equal range: [lower_bound, upper_bound) pair
auto [lo_it, hi_it] = equal_range(v.begin(), v.end(), 3);
// lo_it at index 2, hi_it at index 5

// ============================================================
// ORDERED SET — O(log n) per operation
// ============================================================

set<int> s = {1, 3, 5, 7};

s.lower_bound(4);   // iterator to 5 — O(log n)  ← USE THIS
s.upper_bound(5);   // iterator to 7 — O(log n)

// DO NOT use std::lower_bound(s.begin(), s.end(), 4) — it's O(n) on set!

// Check if value exists in set:
bool inSet = s.count(3);         // O(log n)
bool inSet2 = s.find(3) != s.end(); // O(log n)

// ============================================================
// SORTED VECTOR — COMMON PATTERNS
// ============================================================

// Insert maintaining sorted order:
auto pos = lower_bound(v.begin(), v.end(), 4);
v.insert(pos, 4);

// Delete all occurrences of target:
auto lo = lower_bound(v.begin(), v.end(), 3);
auto hi = upper_bound(v.begin(), v.end(), 3);
v.erase(lo, hi);

// Predecessor (largest element < x):
auto it = lower_bound(v.begin(), v.end(), x);
if (it != v.begin()) --it; // *it is the predecessor

// Successor (smallest element > x):
auto it = upper_bound(v.begin(), v.end(), x);
// *it is the successor (if it != v.end())

// ============================================================
// BINARY SEARCH ON CUSTOM COMPARATOR
// ============================================================

struct Point { int x, y; };
vector<Point> pts = {{1,2}, {3,4}, {5,6}};  // sorted by x

// Find first point with x >= 4:
auto it = lower_bound(pts.begin(), pts.end(), Point{4, 0},
    [](const Point& a, const Point& b) { return a.x < b.x; });

// ============================================================
// BINARY SEARCH ON ANSWER — TEMPLATE QUICK REFERENCE
// ============================================================

// MINIMIZE: find smallest X where check(X) = true (FFFFFFTTTTT)
// lo = min_answer, hi = max_answer
// while (lo < hi) { mid = (lo+hi)/2; if check(mid) hi=mid; else lo=mid+1; }

// MAXIMIZE: find largest X where check(X) = true (TTTTTTFFFFF)
// lo = min_answer, hi = max_answer
// while (lo < hi) { mid = (lo+hi+1)/2; if check(mid) lo=mid; else hi=mid-1; }

// Ceiling division (avoid floating point):
// ceil(a / b) = (a + b - 1) / b   (works for positive a, b)
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

### Immediate next steps:
1. Complete all problems in Tier 1 and 2 WITHOUT looking at your templates. From memory.
2. For Tier 3: Koko and Split Array must be independently derivable. If you need hints, re-read Template 2 and re-derive the answer range.
3. For Tier 4: These appear in actual Codeforces Div 2 C/D. Time yourself — should be under 30 minutes each.

### What to track in REVISION-TRACKER.md:
- Any problem where you couldn't determine `lo` and `hi` → that's the key insight to revisit
- Any problem where you got off-by-one on mid (floor vs ceiling) → add to revision
- LC 4 (Median of Two Sorted Arrays) — revisit this every 2 weeks. It's hard enough to forget.

### Contest application:
In every Codeforces Div 2 contest, look at problem C or D and ask: "Can I binary search on the answer?" In my experience, 30-40% of Div 2 C problems have BS on answer as the intended solution. This single pattern is worth 100+ CF rating points if internalized.

### Connecting to future patterns:
- **Pattern 10 (Heap):** "Find K-th largest" can use heap (O(n log k)) or BS on value with binary indexed tree (O(n log² n) or O(n log n)). Know both.
- **Pattern 21 (LIS):** The O(n log n) LIS algorithm uses `lower_bound` to find insertion point in the patience sort array.
- **Pattern 27 (Segment Tree) + Pattern 28 (Fenwick Tree):** "Find first index with prefix sum ≥ x" — classic BS on Fenwick tree (called "binary lifting on BIT"). Direct O(log n), without the extra log from running two separate binary searches.
- **Pattern 29 (Monotonic Deque):** "Shortest Subarray with Sum ≥ K" — deque OR binary search on prefix sum with monotonic deque. Two approaches, same complexity.

---

## SIGN-OFF CRITERIA

You have mastered Pattern 05 when:

**Tier 1 (Non-negotiable):**
- [ ] Template 1A (exact search) from memory in under 2 minutes
- [ ] Template 1B (lower bound) from memory in under 2 minutes
- [ ] Know when to use `lo <= hi` vs `lo < hi` and WHY
- [ ] Can explain why `mid = lo + (hi - lo) / 2` instead of `(lo + hi) / 2`

**Tier 2:**
- [ ] LC 34 (First and Last Position) — solved in under 20 minutes without hints
- [ ] LC 33 (Search in Rotated) — solved without bugs on first try
- [ ] LC 875 (Koko) — independently derived the check function, lo, and hi
- [ ] LC 162 (Find Peak) — can explain WHY we move toward the rising slope

**Tier 3:**
- [ ] LC 410 (Split Array) — correctly identified lo = max_element, hi = sum
- [ ] LC 4 (Median) — understood and implemented the partition logic

**Tier 4 (Bonus — attempt before sign-off):**
- [ ] LC 1552 (Magnetic Force / Aggressive Cows) — used ceiling mid correctly

**Sign-off test problem:** I will give you one unseen binary search problem. You must:
1. State whether it's classic BS or BS on answer
2. If BS on answer: state the lo, hi, and check function
3. Write the complete C++ solution
4. State time and space complexity

Type `SIGN OFF P05` when you're ready.

---

*Pattern 05 Complete — Binary Search (Classic + On Answer + Rotated Array + Floating Point + STL)*
*Next: Pattern 06 — Prefix Sums and Difference Arrays*
