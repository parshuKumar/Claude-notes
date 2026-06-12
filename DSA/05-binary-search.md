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

### REAL Binary Search Signals (when you SHOULD use it):

| Signal | Example Problem Phrasing |
|--------|--------------------------|
| Sorted array + find target | "Search in sorted array" |
| Sorted array + find boundary | "Find first/last position of X" |
| Find minimum satisfying value | "Minimize the maximum..." |
| Find maximum satisfying value | "Maximize the minimum..." |
| "What is the smallest X such that..." | Koko, capacity to ship |
| Constraint: n ≤ 10^9 but need O(n log n) | Answer must be O(log n) search |
| Monotonic property in problem | "Larger X always works if smaller X works" |
| "K equal parts", "split array" with optimization | Split Array Largest Sum, Painters Partition |
| Rotated sorted array | "array was sorted then rotated at unknown pivot" |
| Peak element | "Find any peak, no global sort needed" |

### FAKE Binary Search Signals (when people misapply it):

| Fake Signal | Why It's Wrong |
|-------------|----------------|
| "Find element in array" | If unsorted and no property, hash map is O(1) |
| "Minimum of array" | Not binary searchable if not monotonic |
| "K-th smallest overall" | Quickselect is O(n) average, not BS directly |
| Counting problem with no monotonicity | Prefix sum might be right tool |
| "Sorted array" alone | If you're searching for ALL occurrences, you also need upper_bound |

### The Monotonicity Test (Ask Yourself This):

If you think a problem is binary search on answer, ask:
> "If answer X is valid, is answer X+1 ALSO valid (or X-1 also valid)?"

If YES → binary search on answer.
If the problem is "is this possible?" and it flips from NO to YES at exactly one threshold → binary search.

If the validity function looks like: NO NO NO YES NO NO YES → **NOT binary searchable** (it's not monotonic).

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

### Mistake 1: Integer Overflow in `mid` Calculation

```cpp
// WRONG: can overflow when lo and hi are both large integers
int mid = (lo + hi) / 2;

// RIGHT: safe from overflow
int mid = lo + (hi - lo) / 2;
```

**When does this happen?** When `hi = 2*10^9` (for binary search on answer with large range) and `lo` is also large, `lo + hi` overflows 32-bit int (max ~2.1×10⁹). Use `long long` or the safe formula.

```cpp
// Comprehensive safe version for BS on answer with large ranges:
long long lo = 0, hi = 2e18;
while (lo < hi) {
    long long mid = lo + (hi - lo) / 2;
    // ...
}
```

---

### Mistake 2: `lo <= hi` vs `lo < hi` — Choosing the Wrong Termination

```cpp
// WRONG for lower_bound (will miss the answer or loop forever):
int lo = 0, hi = n - 1;      // hi = n-1 instead of n
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;
    if (nums[mid] < target) lo = mid + 1;
    else hi = mid;             // hi can become lo, then lo <= hi still true
    // If lo == hi and nums[lo] is the answer, we continue... then lo < hi is false...
    // Actually this just doesn't converge correctly for lower_bound
}
// The issue: when lo == hi and hi = mid, nothing changes. Infinite loop.
```

**Decision rule:**
- Use `lo <= hi` with `lo = mid+1` and `hi = mid-1` → for finding EXACT target (return when found, -1 if not)
- Use `lo < hi` with `lo = mid+1` or `hi = mid` → for finding BOUNDARY (lower/upper bound, BS on answer)

---

### Mistake 3: Wrong `hi` Initialization for Binary Search on Answer

```cpp
// Koko Eating Bananas — WRONG hi:
int hi = 1e9;   // Too large (technically fine, just slower by constant)
int hi = 10;    // Too small! Missing valid answers!

// RIGHT: hi = max possible value that makes semantic sense
int hi = *max_element(piles.begin(), piles.end()); 
// Because if speed = max pile size, you finish each pile in 1 hour.
// Speed can never usefully exceed max pile size.
```

```cpp
// Split Array Largest Sum — WRONG lo:
int lo = 0;     // Wrong! A part can't have sum 0 if there are positive elements

// RIGHT:
int lo = *max_element(nums.begin(), nums.end());
// Because each part must contain at least one element,
// and the maximum is a lower bound for the answer.
```

**Rule:** Think about what `lo` and `hi` represent SEMANTICALLY, not just syntactically. `lo` should be the smallest value that could possibly be the answer. `hi` should be the largest.

---

### Mistake 4: Forgetting Ceiling Mid for "Last TRUE" Problems

```cpp
// Maximize minimum distance — WRONG:
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;  // floor mid
    if (check(mid)) lo = mid;      // lo = mid with floor mid → infinite loop when hi = lo + 1
    else hi = mid - 1;
}

// RIGHT:
while (lo < hi) {
    int mid = lo + (hi - lo + 1) / 2;  // ceiling mid
    if (check(mid)) lo = mid;
    else hi = mid - 1;
}
```

**The `lo = hi - 1` deadlock:**
- `lo = 5, hi = 6`
- Floor `mid = 5`, `check(5)` is true, so `lo = mid = 5`. Nothing changed. Infinite loop.
- Ceiling `mid = 6`, `check(6)` is true, so `lo = mid = 6`. Loop terminates.

---

### Mistake 5: Not Accounting for "No Solution" Cases

```cpp
// Binary search on answer always finds SOME answer in [lo, hi].
// You must separately verify the answer makes sense:

int kokoSpeed = binarySearch(piles, h);

// But what if h < piles.size()? (fewer hours than piles — impossible)
// BS still returns something. Add validation:
if (h < piles.size()) return -1; // or handle per problem
```

Similarly for Split Array: if `m > n` (more parts than elements), the answer is the sum of all elements (each element is its own part, but we have leftover empty parts — the problem guarantees m ≤ n, but always check constraints).

---

### Mistake 6: Applying BS When Predicate Isn't Monotonic

```cpp
// WRONG: trying to BS on something non-monotonic
// "Find the window size K such that the window has the EXACT sum S"
// This is NOT monotonic — window sums go up AND down as K changes.

// RIGHT for this: sliding window or prefix sums, not binary search.
```

**Mental check before writing BS on answer:** "Draw" the predicate function on paper. Does it look like `FFFFFFTTTT` or `TTTTFFFF`? If it has any `FTFT` pattern, binary search will give wrong answers silently.

---

## PROBLEM SET

### Tier 1: Warmup (Get the templates into muscle memory)

| # | Problem | LC # | Key Learning | Est. Time |
|---|---------|------|--------------|-----------|
| 1 | Binary Search | 704 | Pure binary search, Template 1A | 10 min |
| 2 | Search Insert Position | 35 | Lower bound = Template 1B | 10 min |
| 3 | Guess Number Higher or Lower | 374 | BS Template 1A on abstract "array" | 10 min |
| 4 | First Bad Version | 278 | BS on answer: first TRUE in FFFTTT | 15 min |

**Signs of mastery on Tier 1:** All 4 problems in under 40 total minutes, no off-by-one errors.

---

### Tier 2: Core (The problems every interviewer asks)

| # | Problem | LC # | Key Learning | Est. Time |
|---|---------|------|--------------|-----------|
| 5 | Find First and Last Position | 34 | Lower + upper bound combined | 20 min |
| 6 | Search in Rotated Sorted Array | 33 | Which half is sorted? | 25 min |
| 7 | Find Minimum in Rotated Sorted Array | 153 | Compare mid with hi, not lo | 20 min |
| 8 | Find Peak Element | 162 | BS on non-sorted, move toward peak | 20 min |
| 9 | Koko Eating Bananas | 875 | BS on answer: minimize speed | 25 min |

**Signs of mastery on Tier 2:** No peeking at templates. Correct first try on 33 and 153.

---

### Tier 3: Stretch (Separate 1600 from 1800)

| # | Problem | LC # | Key Learning | Est. Time |
|---|---------|------|--------------|-----------|
| 10 | Capacity to Ship Packages | 1011 | BS on answer: consecutive allocation check | 30 min |
| 11 | Split Array Largest Sum | 410 | BS on answer: minimize the maximum, correct lo | 35 min |
| 12 | Median of Two Sorted Arrays | 4 | BS on partition — hardest BS problem | 45 min |
| 13 | Sqrt(x) | 69 | Integer sqrt via "last TRUE" template | 15 min |

---

### Tier 4: Contest (What appears in Codeforces Div 2 C/D)

| # | Problem | LC # | Key Learning | Est. Time |
|---|---------|------|--------------|-----------|
| 14 | Magnetic Force Between Two Balls | 1552 | Maximize minimum: "last TRUE" template | 30 min |
| 15 | Minimize Max Distance to Gas Station | 774 | Floating point BS | 35 min |
| 16 | Find in Mountain Array | 1095 | Peak first, then two BS passes | 35 min |
| 17 | Find K-th Smallest Pair Distance | 719 | BS on answer + sliding window check | 40 min |

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

### Simulation 1: Koko Eating Bananas (Google phone screen style)

**Interviewer:** "Koko loves to eat bananas. There are n piles of bananas. The i-th pile has piles[i] bananas. The guards have gone and will come back in h hours. Koko can decide her bananas-per-hour eating speed k. Each hour, she chooses some pile of bananas and eats k bananas from that pile. If the pile has less than k bananas, she eats all of them and will not eat any more bananas during this hour. Koko likes to eat slowly but still wants to finish eating all the bananas before the guards come back. Return the minimum integer k such that she can eat all the bananas within h hours."

**Green flags (what good candidates say):**
- "Let me think about what values k can take. It ranges from 1 to max(piles). The answer is monotonic — if k works, k+1 works. So I can binary search."
- States the invariant: "I'll binary search on k, with check function = can she finish in h hours at speed k."
- `ceil(p / k) = (p + k - 1) / k` without floating point.
- lo = 1, hi = max(piles) — and EXPLAINS why (not 1e9 blindly).

**Red flags:**
- Starts with a brute force loop over all k from 1 to 1e9 without recognizing binary search.
- Uses `lo = 0` (speed 0 means 0 bananas/hour, can never finish — invalid lo).
- Uses floating point for ceiling division.
- Forgets early termination in check function.

**Follow-up:** "What if h < piles.size()? What if all piles have 1 banana?" Handle edge cases.

---

### Simulation 2: Find First and Last Position (Amazon phone screen style)

**Interviewer:** "Given a sorted array of integers, find the first and last position of a target value. Must be O(log n)."

**Green flags:**
- Immediately identifies this as two binary searches (lower_bound + upper_bound).
- Explains `lo < hi` termination and why `hi = mid` instead of `hi = mid - 1`.
- Handles edge cases: target not in array (first == n or nums[first] != target → return -1, -1).
- `last = upperBound(target) - 1` reasoning.

**Red flags:**
- Uses `lo <= hi` template for lower bound (might work but reveals shaky understanding).
- Does a binary search to find ANY occurrence, then linearly scans left and right (O(n) worst case).
- Doesn't handle the "target not found" edge case.

**Follow-up:** "What if there are multiple targets and you want the SECOND occurrence?" (Return lower_bound + 1, verify it's still target.)

---

### Simulation 3: Hard Problem (Split Array Largest Sum — Google onsite)

**Interviewer:** "Given an integer array `nums` and an integer `k`, split `nums` into `k` non-empty subarrays such that the largest sum of any subarray is minimized. Return this minimized largest sum."

**Green flags:**
- Immediately says "minimize the maximum → binary search on the answer."
- States the answer range: lo = max(nums), hi = sum(nums).
- Writes the check function greedily: greedily pack elements into parts until adding next exceeds maxSum — if we need more than k parts, return false.
- Uses `lo < hi` template, assigns `hi = mid` when check passes.

**Red flags:**
- Tries DP first (DP works but is harder to code correctly under pressure).
- Sets `lo = 0` (explains a conceptual gap — a subarray with positive integers can't have sum 0).
- Forgets the `x > maxSum` check inside the greedy checker (a single element might exceed maxSum).

**The real test:** Can they derive the answer range [max_element, sum] from first principles, or do they just "know" it?

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
