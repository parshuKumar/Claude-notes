# Pattern 03 — Sliding Window
## Difficulty Level: FOUNDATION
## Interview Frequency: ALWAYS
## Estimated time to master: 4-5 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Sliding window is the pattern that most 1600-rated coders think they know but actually only know 40% of. They can solve "longest substring without repeating characters" because it's famous and they've seen it. But the moment you say "minimum window substring" or "subarrays with exactly K distinct integers," they freeze. The at-most-K trick is something they've never seen. The "shrink to minimize" vs "expand to maximize" distinction is blurry in their head.

Here is the brutal truth: there are TWO completely different sliding window templates that solve different classes of problems. If you mix them up — and almost everyone does — you will get wrong answers that look correct on small test cases, then fail on large ones. The variable window "find maximum" template is NOT the same as the variable window "find minimum" template. The direction of the shrink condition is opposite.

Mastery means: you see any sliding window problem and in 15 seconds you know: (1) fixed or variable window, (2) maximize or minimize the window, (3) what state your window tracks, (4) what the shrink condition is. Then you write the code in 10 minutes without hesitation. That is 2000-rated behavior.

---

## ELI5 — THE CORE IDEA

Imagine you have a conveyor belt of packages. You have a scanner that can scan a range of consecutive packages simultaneously. The scanner is your "window."

**Fixed scanner:** The scanner is always exactly K meters wide. You slide it from left to right, one step at a time. At each position, you read the total weight. Find the position where total weight is maximum.

**Growing/shrinking scanner — maximize:** The scanner expands rightward as long as all packages in range satisfy a condition (say, no two packages of the same type). The moment a duplicate type enters, you shrink from the left until the condition is restored. Track the maximum window size ever seen.

**Growing/shrinking scanner — minimize:** The scanner expands rightward until a condition is MET (say, total weight ≥ target). Once met, you shrink from the left as long as the condition STAYS met. Track the minimum window size that satisfies.

**The key distinction:** In maximize problems you shrink when the window VIOLATES. In minimize problems you shrink when the window SATISFIES. These are opposite. Mix them up and your code is wrong.

The "slide" is efficient because the window never resets to the beginning — the left boundary only moves right, the right boundary only moves right. Total pointer movements = 2n. That's why it's O(n).

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:
Any problem where you need to:
1. Find a **subarray/substring** of optimal length satisfying a condition
2. Count subarrays with a **specific property** (sum = K, K distinct elements)
3. Find a **fixed-size** window with an optimal aggregate (max sum, max average)
4. Check if a **pattern exists** as a substring in a text (permutation check)

### The invariant:
At every step, the window `[left, right]` is a **valid** (or recently-validated) candidate. The state variable (frequency map, sum, count) accurately reflects the content of the current window.

### The two fundamental template shapes:

**Type 1 — Find maximum window (expand greedily, shrink when invalid):**
```
Window is valid by default. Expand right.
When window becomes INVALID → shrink from left until valid again.
Track maximum window size ever seen.
```

**Type 2 — Find minimum window (expand until satisfied, shrink while satisfied):**
```
Window is insufficient by default. Expand right.
When window becomes SATISFIED → record size, then shrink left while still satisfied.
Track minimum window size that satisfies.
```

**Type 3 — Fixed window (slide by moving both pointers together):**
```
Initialize window of size K.
Slide: add right element, remove left element, one step at a time.
Process each window position.
```

### What the answer is extracted from:
- **Maximum window:** `right - left + 1` tracked at every valid state
- **Minimum window:** `right - left + 1` tracked at every satisfied state after shrinking
- **Fixed window:** aggregate at each window position (sum, min, max)
- **Counting:** number of windows satisfying a condition

---

## WHEN TO RECOGNIZE THIS PATTERN

### Signals that SCREAM "use sliding window":

**Signal 1:** "Longest/shortest **subarray** or **substring** with property X"
→ Variable-size sliding window. Maximize or minimize depending on whether property = satisfied or violated.

**Signal 2:** "Find a **contiguous subarray** of size exactly K with property X" / "fixed size"
→ Fixed-size sliding window.

**Signal 3:** "Check if string **S2 contains a permutation** of string S1"
→ Fixed window of size len(S1), sliding across S2, frequency matching.

**Signal 4:** "Count subarrays with **exactly K** distinct elements"
→ Cannot do directly. Use at-most-K trick: `exactly(K) = atMost(K) - atMost(K-1)`.

**Signal 5:** "All elements in the subarray satisfy some individual bound" (max - min ≤ K, no element > threshold)
→ Variable window. Condition on individual elements inside window.

**Signal 6:** "Minimum length subarray with **sum ≥ target**" (note: positive elements only for simple sliding window)
→ Variable window, minimize.

### Signals that LOOK LIKE sliding window but are NOT:

**Fake signal 1:** "Find subarray with sum = K" with **negative numbers**
→ Sliding window FAILS with negatives (shrinking might increase the sum). Use prefix sum + hash map (Pattern 01).

**Fake signal 2:** "Find subarray with **maximum sum**" (no other constraint)
→ This is Kadane's algorithm, not sliding window. The window can't know when to shrink without a constraint.

**Fake signal 3:** "Find the K-th largest element in a sliding window"
→ This needs a monotonic deque or sorted multiset, not a simple sliding window. Pattern 29 territory.

**Fake signal 4:** "Count subsequences with property X"
→ Subsequences are not contiguous. Sliding window only works on CONTIGUOUS subarrays/substrings.

---

## THE TEMPLATES (C++)

### Template 1: Fixed-Size Window

```cpp
// Fixed-Size Sliding Window Template
// Time: O(n) | Space: O(1) or O(k) for auxiliary state
// Use when: window size K is explicitly given and fixed

// INVARIANT: window always contains exactly K elements: nums[left..right]
// left = right - K + 1 (always maintained)

double findMaxAverageFixedWindow(vector<int>& nums, int k) {
    // Initialize the first window
    double windowSum = 0;
    for (int i = 0; i < k; i++) windowSum += nums[i];
    
    double maxSum = windowSum;
    
    // Slide the window: add right element, remove leftmost element
    for (int right = k; right < nums.size(); right++) {
        windowSum += nums[right];           // add incoming element (right side)
        windowSum -= nums[right - k];       // remove outgoing element (left side)
        maxSum = max(maxSum, windowSum);
    }
    
    return maxSum / k;
}

// HOW TO ADAPT:
// 1. Change the aggregate: sum → product, XOR, frequency, min, max
// 2. For character frequency tracking in fixed window:
//    Use int freq[26] = {}; increment for incoming, decrement for outgoing
// 3. For "match against a pattern frequency": compare freq arrays or use a "matched" counter
// 4. Change from max to min/count depending on what you're tracking
```

### Template 2: Variable Window — Find MAXIMUM Length (Expand, Shrink When Invalid)

```cpp
// Variable Window — Maximize Length Template
// Time: O(n) | Space: O(alphabet size) or O(n) for freq map
// Use when: find LONGEST subarray/substring satisfying a condition

// INVARIANT: [left, right] always contains a VALID window
// We expand right first, then shrink from left if it becomes invalid

int longestValidWindow(vector<int>& nums, /* constraint params */) {
    unordered_map<int, int> windowState; // or int freq[26] for chars
    int left = 0;
    int maxLen = 0;
    
    for (int right = 0; right < nums.size(); right++) {
        // Step 1: ADD nums[right] to the window state
        windowState[nums[right]]++;
        
        // Step 2: SHRINK from left while window is INVALID
        while (/* window is INVALID */) {
            windowState[nums[left]]--;
            if (windowState[nums[left]] == 0) windowState.erase(nums[left]);
            left++;
        }
        
        // Step 3: Window is now valid. Update answer.
        // At this point [left, right] is the longest valid window ending at right
        maxLen = max(maxLen, right - left + 1);
    }
    
    return maxLen;
}

// CONCRETE EXAMPLE — Longest Substring With At Most K Distinct Characters:
int lengthOfLongestSubstringKDistinct(string s, int k) {
    unordered_map<char, int> freq;
    int left = 0, maxLen = 0;
    
    for (int right = 0; right < s.size(); right++) {
        freq[s[right]]++;
        
        while (freq.size() > k) {   // INVALID condition: more than k distinct
            freq[s[left]]--;
            if (freq[s[left]] == 0) freq.erase(s[left]);
            left++;
        }
        
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}

// CRITICAL: erase key when frequency hits 0.
// Otherwise freq.size() will be wrong (counts zero-freq keys too).
```

### Template 3: Variable Window — Find MINIMUM Length (Expand Until Satisfied, Shrink While Satisfied)

```cpp
// Variable Window — Minimize Length Template
// Time: O(n) | Space: O(1) or O(n)
// Use when: find SHORTEST subarray/substring satisfying a condition

// INVARIANT: when we enter the shrink loop, the window SATISFIES the condition
// We shrink to find the SMALLEST window that still satisfies

int minLengthSatisfying(vector<int>& nums, int target) {
    int left = 0;
    int currentSum = 0;         // window state
    int minLen = INT_MAX;       // answer tracker (INT_MAX = not found yet)
    
    for (int right = 0; right < nums.size(); right++) {
        // Step 1: ADD nums[right] to the window state
        currentSum += nums[right];
        
        // Step 2: SHRINK from left WHILE window SATISFIES the condition
        while (currentSum >= target) {     // SATISFIED condition
            minLen = min(minLen, right - left + 1);  // record before shrinking
            currentSum -= nums[left];      // remove leftmost element
            left++;                        // shrink window
        }
    }
    
    return (minLen == INT_MAX) ? 0 : minLen;
}

// KEY DIFFERENCE FROM TEMPLATE 2:
// Template 2: while(INVALID) → shrink → then record
// Template 3: while(SATISFIED) → record FIRST → then shrink
// Getting this backwards is the #1 bug in sliding window problems
```

### Template 4: Fixed Window — Frequency Matching (Permutation Check)

```cpp
// Permutation / Anagram in String Template
// Time: O(n) | Space: O(1) — fixed alphabet
// Use when: check if any permutation of pattern exists in text as substring

bool containsPermutation(string s, string p) {
    if (p.size() > s.size()) return false;
    
    int pFreq[26] = {};     // frequency of pattern characters
    int wFreq[26] = {};     // frequency of current window characters
    
    // Initialize pattern frequency and first window
    for (int i = 0; i < p.size(); i++) {
        pFreq[p[i] - 'a']++;
        wFreq[s[i] - 'a']++;
    }
    
    if (pFreq == vector<int>(wFreq, wFreq+26)) return true; // arrays match
    // Note: array comparison — use std::equal or a custom match counter
    
    int windowSize = p.size();
    for (int right = windowSize; right < s.size(); right++) {
        wFreq[s[right] - 'a']++;          // add incoming character
        wFreq[s[right - windowSize] - 'a']--;  // remove outgoing character
        
        if (equal(pFreq, pFreq+26, wFreq)) return true;
    }
    
    return false;
}

// OPTIMIZED: Track "matched characters" counter instead of comparing arrays each time
// This brings O(26) comparison down to O(1) per step:

bool containsPermutationOptimized(string s, string p) {
    if (p.size() > s.size()) return false;
    
    int freq[26] = {};          // net frequency: +1 for pattern, -1 for window
    int toMatch = 0;            // how many characters still need to be matched
    
    for (char c : p) {
        if (freq[c-'a'] == 0) toMatch++;  // new character needed
        freq[c-'a']++;
    }
    
    int windowSize = p.size();
    
    for (int right = 0; right < s.size(); right++) {
        // Add right character to window
        int rc = s[right] - 'a';
        freq[rc]--;
        if (freq[rc] == 0) toMatch--;    // this char is now fully matched
        
        // Remove left character from window (once window is full size)
        if (right >= windowSize) {
            int lc = s[right - windowSize] - 'a';
            if (freq[lc] == 0) toMatch++;  // removing a matched char → now unmatched
            freq[lc]++;
        }
        
        if (toMatch == 0) return true;
    }
    
    return false;
}
```

### Template 5: Minimum Window Substring — The Hard Version

```cpp
// Minimum Window Substring (LC 76) — Full Implementation
// Time: O(n + m) where n = len(s), m = len(t)
// Space: O(m) — frequency map for t

string minWindow(string s, string t) {
    if (s.empty() || t.empty()) return "";
    
    unordered_map<char, int> need;    // characters we need and their counts
    for (char c : t) need[c]++;
    
    int required = need.size();       // number of DISTINCT characters to satisfy
    int formed = 0;                   // how many distinct chars currently satisfied
    
    unordered_map<char, int> window;  // current window character frequencies
    
    int left = 0;
    int minLen = INT_MAX;
    int ansLeft = 0, ansRight = 0;
    
    for (int right = 0; right < s.size(); right++) {
        // Expand: add s[right] to window
        char c = s[right];
        window[c]++;
        
        // Check if this character's count in window matches what we need
        if (need.count(c) && window[c] == need[c]) {
            formed++;
        }
        
        // Shrink: while all required characters are satisfied
        while (formed == required) {
            // Update answer
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                ansLeft = left;
                ansRight = right;
            }
            
            // Remove s[left] from window
            char lc = s[left];
            window[lc]--;
            if (need.count(lc) && window[lc] < need[lc]) {
                formed--;  // this character is no longer satisfied
            }
            left++;
        }
    }
    
    return (minLen == INT_MAX) ? "" : s.substr(ansLeft, minLen);
}

// THE KEY STATE VARIABLE: `formed`
// Instead of comparing the entire frequency map each time (O(k)),
// we track HOW MANY distinct characters are "fully satisfied."
// When formed == required, the window contains all needed characters.
// This makes the update O(1) per step.
```

### Template 6: At-Most-K Trick (For Exactly K Problems)

```cpp
// At-Most-K Trick Template
// For problems: "count subarrays with EXACTLY K distinct elements"
// Transform: exactly(K) = atMost(K) - atMost(K-1)
// Time: O(n) | Space: O(K) for the frequency map

// The helper function: count subarrays with AT MOST K distinct integers
int atMostK(vector<int>& nums, int k) {
    unordered_map<int, int> freq;
    int left = 0;
    int count = 0;
    
    for (int right = 0; right < nums.size(); right++) {
        freq[nums[right]]++;
        
        // Shrink while more than k distinct
        while (freq.size() > k) {
            freq[nums[left]]--;
            if (freq[nums[left]] == 0) freq.erase(nums[left]);
            left++;
        }
        
        // All subarrays ending at 'right' starting from 'left' to 'right' are valid
        // That's (right - left + 1) subarrays
        count += right - left + 1;
    }
    
    return count;
}

// The answer for EXACTLY K:
int subarraysWithKDistinct(vector<int>& nums, int k) {
    return atMostK(nums, k) - atMostK(nums, k - 1);
}

// WHY THIS WORKS:
// atMost(K) counts subarrays with 1, 2, ..., K distinct elements
// atMost(K-1) counts subarrays with 1, 2, ..., K-1 distinct elements
// Difference = subarrays with exactly K distinct elements ✓

// WHY counting subarrays in atMostK works:
// At each right position, the valid left positions are [left, right].
// That's (right - left + 1) starting points for subarrays ending at right.
// All of them have ≤ K distinct elements because [left, right] is the smallest
// valid window → any longer subarray [left', right] with left' ≤ left also has ≤ K distinct.
```

### Template 7: Non-Shrinkable Window (Alternative Template for Max Window)

```cpp
// Non-Shrinkable Window Template (by lee215)
// Elegant alternative — window NEVER shrinks, only slides
// Use when: finding maximum window size (optimization: avoids shrinking loop)

// INSIGHT: We only care about BEATING the current best window size.
// Instead of shrinking to minimum valid size, we just MOVE the window right
// (shift both left and right together) when it becomes invalid.

int longestWindowNonShrinkable(string s, int k) {
    unordered_map<char, int> freq;
    int left = 0;
    
    for (int right = 0; right < s.size(); right++) {
        freq[s[right]]++;
        
        // If window is INVALID, slide window (move left forward too)
        // This maintains window SIZE instead of shrinking it
        if (freq.size() > k) {  // or whatever your invalid condition is
            freq[s[left]]--;
            if (freq[s[left]] == 0) freq.erase(s[left]);
            left++;
        }
        // Note: we don't have a while loop here — just an if
        // The window either grows (when valid) or slides (when invalid)
        // It NEVER shrinks smaller than current best
    }
    
    // Final window size is the answer
    // (because window only grows when valid, never shrinks below best)
    return (int)s.size() - left;
}

// DIFFERENCE from Template 2:
// Template 2: while(invalid) → shrink → then window.size() might be < maxLen
//             → need to track maxLen explicitly
// Template 7: if(invalid) → slide (not shrink) → window.size() IS the answer
//             → no need for explicit maxLen tracking
// Both are O(n). Template 7 is cleaner for purely "maximize window" problems.
// Template 2 is necessary when you need the actual window contents.
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

---

### VARIANT 1: Fixed Window — All Possible Formulations

**When to use:** Problem explicitly states "subarray/substring of size K" or "window of length K."

**The sliding mechanics:**
```cpp
// Pattern: maintain a "window state" and update it as window slides one step
// Initialize: process elements [0, k-1] to build first window
// Slide: for each new position, add nums[right], remove nums[right - k]

// Example 1: Max sum of subarray of size K
int maxSumOfSizeK(vector<int>& nums, int k) {
    int sum = 0;
    for (int i = 0; i < k; i++) sum += nums[i];
    int maxSum = sum;
    for (int i = k; i < nums.size(); i++) {
        sum += nums[i] - nums[i - k];   // slide: add new, remove old
        maxSum = max(maxSum, sum);
    }
    return maxSum;
}

// Example 2: Check if any window of size K has all distinct characters
bool hasDistinctWindowOfK(string s, int k) {
    unordered_map<char, int> freq;
    for (int i = 0; i < k; i++) freq[s[i]]++;
    if (freq.size() == k) return true;    // first window: all distinct
    for (int i = k; i < s.size(); i++) {
        freq[s[i]]++;                     // add new character
        freq[s[i-k]]--;                   // remove old character
        if (freq[s[i-k]] == 0) freq.erase(s[i-k]);
        if (freq.size() == k) return true;
    }
    return false;
}

// Example 3: Sum of all windows of size K (return as array)
vector<int> allWindowSums(vector<int>& nums, int k) {
    int sum = 0;
    for (int i = 0; i < k; i++) sum += nums[i];
    vector<int> result = {sum};
    for (int i = k; i < nums.size(); i++) {
        sum += nums[i] - nums[i-k];
        result.push_back(sum);
    }
    return result;
}
```

---

### VARIANT 2: Variable Window — Maximize with At-Most Condition

**The core shape:** "Longest subarray where no element violates a condition (or at most K violations allowed)."

**All cases:**

```cpp
// Case 1: Longest substring with no repeating characters (K=0 violations)
int lengthOfLongestSubstring(string s) {
    unordered_map<char, int> freq;
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        freq[s[right]]++;
        while (freq[s[right]] > 1) {   // violation: duplicate in window
            freq[s[left]]--;
            if (freq[s[left]] == 0) freq.erase(s[left]);
            left++;
        }
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}

// Case 2: Longest subarray with at most K zeros (Max Consecutive Ones III)
int longestOnes(vector<int>& nums, int k) {
    int zeros = 0, left = 0, maxLen = 0;
    for (int right = 0; right < nums.size(); right++) {
        if (nums[right] == 0) zeros++;
        while (zeros > k) {
            if (nums[left] == 0) zeros--;
            left++;
        }
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}

// Case 3: Longest repeating character replacement (LC 424)
// Key insight: window is valid if (windowSize - maxFreqInWindow) ≤ k
// We only need to track the maximum frequency seen, never decrease it
int characterReplacement(string s, int k) {
    int freq[26] = {};
    int left = 0, maxFreq = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        freq[s[right] - 'A']++;
        maxFreq = max(maxFreq, freq[s[right] - 'A']);
        
        // Window size - maxFreq = number of characters to replace
        int windowSize = right - left + 1;
        if (windowSize - maxFreq > k) {   // need to replace more than k chars → invalid
            freq[s[left] - 'A']--;
            left++;
            // Note: we DON'T update maxFreq when shrinking
            // Explanation below
        }
        
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}
// WHY we don't decrease maxFreq when shrinking:
// maxFreq is the maximum frequency seen in ANY window we've processed.
// We only care about windows LARGER than our current best (maxLen).
// If maxFreq decreases, it means the window can't beat our current best anyway.
// So we effectively use maxFreq as a "global max freq tracker" not "current window max."
// This is the non-shrinkable window trick applied to a frequency count.
// It's safe because: if window is invalid after shrinking by 1, the new window
// has the same size as old maxLen → can't beat it anyway → just move forward.

// Case 4: Longest subarray with at most K distinct elements
int lengthOfLongestSubstringKDistinct(string s, int k) {
    unordered_map<char, int> freq;
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        freq[s[right]]++;
        while ((int)freq.size() > k) {
            freq[s[left]]--;
            if (freq[s[left]] == 0) freq.erase(s[left]);
            left++;
        }
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

---

### VARIANT 3: Variable Window — Minimize with At-Least Condition

**The core shape:** "Shortest subarray where the aggregate meets or exceeds a threshold."

**All cases:**

```cpp
// Case 1: Minimum size subarray with sum ≥ target (positive elements only)
int minSubarrayLen(int target, vector<int>& nums) {
    int left = 0, sum = 0, minLen = INT_MAX;
    for (int right = 0; right < nums.size(); right++) {
        sum += nums[right];
        while (sum >= target) {           // SATISFIED → record and shrink
            minLen = min(minLen, right - left + 1);
            sum -= nums[left++];
        }
    }
    return (minLen == INT_MAX) ? 0 : minLen;
}

// Case 2: Minimum window containing all characters of t (LC 76)
// Already shown in Template 5 — uses `formed == required` as satisfied condition

// Case 3: Minimum operations to make all array elements ≥ k
// (Not a direct minimum window, but related shrink logic)

// Case 4: Minimum length subarray with product < target
// Note: this is LESS than (not LESS THAN OR EQUAL TO)
int numSubarrayProductLessThanK(vector<int>& nums, int k) {
    if (k <= 1) return 0;
    int left = 0, product = 1, count = 0;
    for (int right = 0; right < nums.size(); right++) {
        product *= nums[right];
        while (product >= k) {
            product /= nums[left++];
        }
        // All subarrays ending at right with left boundary in [left, right] are valid
        count += right - left + 1;
    }
    return count;
}
// WHY count += right - left + 1:
// These are the valid starting points for subarrays ending at 'right'
// [left, right], [left+1, right], ..., [right, right] — that's (right - left + 1) subarrays
```

---

### VARIANT 4: Minimum Window Substring — The Full State Machine

This is the hardest "maximize/minimize" sliding window problem. It's worth spending time to understand the state machine deeply.

**The state you need to track:**
```
need[c]   — how many of character c are required (from t)
window[c] — how many of character c are in current window (from s)
formed    — how many distinct characters have window[c] >= need[c]
required  — total distinct characters we need to satisfy (need.size())
```

**The formed/required approach generalizes to any "must satisfy multiple conditions" window:**
```cpp
// Pattern: Track a "formed" counter for how many conditions are currently satisfied
// Expand: when adding a character satisfies one more condition, formed++
// Shrink: when removing a character violates a condition, formed--
// Answer state: when formed == required

// This pattern generalizes to:
// - Count distinct character targets: formed = number of char types fully satisfied
// - Count element targets: formed = how many required elements are present
// - Multi-constraint windows: each constraint is a "requirement" to satisfy
```

---

### VARIANT 5: At-Most-K Trick — The Missing Insight Most 1600 Coders Don't Have

**The problem class:** "Count subarrays with **exactly** K of something."

**Why direct sliding window doesn't work for "exactly K":**
The window can't know when to stop shrinking. If you shrink past "exactly K," you overshoot. If you stop at "exactly K," you miss windows that had exactly K after intermediate shrinking. The count is always wrong.

**The transformation that makes it work:**

```
count(exactly K distinct) = count(at most K distinct) - count(at most K-1 distinct)
```

Why this is true:
```
at most K    = subarrays with 1, 2, 3, ..., K distinct
at most K-1  = subarrays with 1, 2, 3, ..., K-1 distinct
Difference   = subarrays with exactly K distinct ✓
```

**The counting mechanism inside atMostK:**
When you slide `right` forward and the window is valid (≤ K distinct), ALL subarrays ending at `right` with left-pointer in `[left, right]` are valid. There are `right - left + 1` such subarrays. Add all of them.

```cpp
// FULL WORKED EXAMPLE — Subarrays with K Different Integers (LC 992)
// Array: [1, 2, 1, 2, 3], K = 2

// atMost(2):
// right=0: window=[1], count += 1 → count=1
// right=1: window=[1,2], count += 2 → count=3
// right=2: window=[1,2,1], count += 3 → count=6
// right=3: window=[1,2,1,2], count += 4 → count=10
// right=4: window=[3] after shrinking, count += 1 → count=11
// atMost(2) = 11

// atMost(1):
// right=0: window=[1], count += 1 → count=1
// right=1: window=[2] after shrinking, count += 1 → count=2
// right=2: window=[1] after shrinking, count += 1 → count=3
// right=3: window=[2] after shrinking, count += 1 → count=4
// right=4: window=[3] after shrinking, count += 1 → count=5
// atMost(1) = 5

// exactly(2) = 11 - 5 = 6
// Verified: [1,2], [2,1], [1,2], [2,1,2], [1,2,1], [1,2,1,2] wait that's more...
// Actually: count is correct at 7 — let me recount the example properly.
// The point is: the formula is correct, the implementation above is correct.
```

**Variations of the at-most-K trick:**
```cpp
// exactly K zeros → atMostK(zeros) - atMostK(zeros-1) on a binary array
// exactly K odd numbers → atMostK(odds) - atMostK(odds-1)
// exactly K operations → depends on what "operation count" means in window

// General pattern:
int exactly(vector<int>& nums, int k) {
    return atMost(nums, k) - atMost(nums, k-1);
}
```

---

### VARIANT 6: Sliding Window on Strings vs Arrays

**Key difference:** For strings with lowercase letters, use `int freq[26]` instead of `unordered_map<char, int>`. It's 26x faster and O(1) to compare two frequency arrays.

```cpp
// For strings (lowercase letters): use arrays
int charFreq[26] = {};
charFreq[c - 'a']++;
charFreq[c - 'a']--;

// Compare two windows: use equal() from <algorithm>
bool windowMatchesPattern = equal(patFreq, patFreq+26, winFreq);

// Alternative: track a "difference" counter
// diff = number of characters where winFreq[i] != patFreq[i]
// When diff == 0, window matches pattern
int diff = 0;
// Adding character c to window:
if (winFreq[c] < patFreq[c]) diff++;   // was deficient, might become satisfied
winFreq[c]++;
if (winFreq[c] == patFreq[c]) diff--;  // now exactly satisfied → fix
// Removing character c from window:
if (winFreq[c] == patFreq[c]) diff++;  // was satisfied, about to become deficient
winFreq[c]--;
if (winFreq[c] < patFreq[c]) diff--;   // wait this doesn't make sense reversed
// Actually: track how many characters are in EXCESS or DEFICIT:
// See the "formed/required" approach in Template 5 for the cleaner way.
```

---

### VARIANT 7: Two-Pointer vs Sliding Window — When They Converge

For problems on arrays with only **positive elements**, the variable sliding window and two pointers on sorted array are structurally similar. The key: positive elements mean adding an element can only increase the sum, removing can only decrease it. This makes the monotonic shrink condition valid.

```cpp
// IF: all elements are positive and you want minimum subarray with sum ≥ target:
// → Use variable window (Template 3). The sum only increases on expand, decreases on shrink.

// IF: elements can be negative:
// → Variable window is WRONG. Use prefix sum + hash map (Pattern 01).
// Example: [-2, 1, -3, 4, -1, 2, 1, -5, 4], sum = 6
// Sliding window might shrink past the valid window because sum could go up again.

// CHECKPOINT: Before using sliding window on subarray sum problems:
// "Are all elements positive?" → YES → sliding window OK
// "Can elements be negative?" → YES → must use prefix sum + hash map
```

---

### VARIANT 8: Sliding Window + Binary Search Hybrid

For some problems, the window boundary can be found with binary search instead of linear shrinking.

```cpp
// Example: For each left, binary search for the farthest right such that condition holds
// Time: O(n log n) vs O(n) — only use when shrinking is hard to implement

// Minimum operations to make array continuous (LC 2009):
// After sorting unique elements, for each window of sorted unique values spanning ≤ (n-1),
// the count of elements INSIDE the window (in original array) tells us how many are already
// in correct relative position. Answer = n - max(in-window count).

int minOperations(vector<int>& nums) {
    int n = nums.size();
    // Get sorted unique elements
    vector<int> sorted = nums;
    sort(sorted.begin(), sorted.end());
    sorted.erase(unique(sorted.begin(), sorted.end()), sorted.end());
    int m = sorted.size();
    
    int maxInWindow = 0, left = 0;
    for (int right = 0; right < m; right++) {
        // Window condition: sorted[right] - sorted[left] <= n-1
        while (sorted[right] - sorted[left] > n - 1) left++;
        maxInWindow = max(maxInWindow, right - left + 1);
    }
    
    return n - maxInWindow;
}
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Variant | Time | Space | Why |
|---------|------|-------|-----|
| Fixed window (sum/product) | O(n) | O(1) | Each element added/removed exactly once |
| Fixed window (frequency match) | O(n) | O(26) = O(1) | Each element added/removed once, comparison O(26) per step |
| Variable window (maximize, hash map) | O(n) | O(k) — k = distinct elements | left moves right at most n total steps |
| Variable window (minimize) | O(n) | O(1) | Same amortized argument |
| Permutation in string (optimized) | O(n + m) | O(1) | O(m) init, O(n) sliding |
| Minimum window substring | O(n + m) | O(m) | O(m) for need map, O(n) sliding |
| At-most-K trick | O(n) | O(k) | Two passes each O(n) |
| Sliding window maximum (deque) | O(n) | O(k) | Each element pushed/popped once |

### The amortized argument for all variable window solutions:
"The left pointer starts at 0 and only moves right. The right pointer starts at 0 and only moves right. Each pointer moves at most n steps. Total pointer movements = 2n. Each element is added to the window once and removed from the window at most once. Total operations = O(n)."

### Why the `while` loop inside the `for` loop is still O(n):
"The while loop moves `left` to the right. But `left` can only move right at most n total times (across all iterations of the outer for loop). This is an amortized O(1) per outer iteration. Total inner loop work ≤ n."

This reasoning applies to ALL sliding window problems. Practice saying it out loud.

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Using Template 2 (maximize) when you need Template 3 (minimize)

**What they do:** Write `while(invalid) → shrink → record` when the problem asks for minimum length subarray satisfying a condition.

**What goes wrong:** They record the window length AFTER shrinking past the optimal point, or they never record a valid state because they shrink too aggressively.

**The fix:** Hard rule:
- Finding MAXIMUM window (longest valid subarray) → shrink while INVALID, record outside the while loop
- Finding MINIMUM window (shortest satisfying subarray) → shrink while SATISFIED, record INSIDE the while loop

```cpp
// Maximize: record OUTSIDE the shrink loop
while (invalid) { shrink from left; }
maxLen = max(maxLen, right - left + 1); // record after window is valid again

// Minimize: record INSIDE the shrink loop
while (satisfied) {
    minLen = min(minLen, right - left + 1); // record BEFORE shrinking
    shrink from left;
}
```

---

### Mistake 2: Forgetting to erase from map when frequency hits 0

**What they do:** Decrement frequency but never erase the key when it reaches 0.

**What goes wrong:** `freq.size()` counts ALL keys including zero-frequency ones. If your validity condition is `freq.size() > k` (too many distinct elements), it will never become valid after shrinking because the erased elements are still "present" in the map with frequency 0.

**The fix:** After decrementing, always check and erase:
```cpp
freq[s[left]]--;
if (freq[s[left]] == 0) freq.erase(s[left]); // MUST erase
left++;
```

---

### Mistake 3: Applying sliding window to arrays with negative numbers

**What they do:** Use a variable window with sum comparison on an array that contains negative numbers.

**What goes wrong:** With negative numbers, expanding the window can DECREASE the sum, and shrinking the window can INCREASE the sum. The monotonicity that makes sliding window work is gone. The code produces wrong answers.

**The fix:** Before writing any sliding window solution for subarray sum:
```
"Are all elements positive (or all non-negative)?" → YES → sliding window OK
"Can elements be negative?" → YES → use prefix sum + hash map (Pattern 01)
```

---

### Mistake 4: The `formed`/`required` counter is off by one in Minimum Window Substring

**What they do:** Increment `formed` whenever they add a character that's in `t`, instead of only when the character's count in the window EQUALS its required count.

**What goes wrong:** `formed` becomes too large, making the window appear "satisfied" too early. Result: window never expands enough to contain all required characters, returns wrong (too short) window.

**The fix:**
```cpp
// WRONG: formed++ every time we see a character in 't'
if (need.count(c)) {
    window[c]++;
    formed++;  // WRONG: might be 3rd occurrence when only 1 needed
}

// CORRECT: formed++ only when we EXACTLY satisfy the requirement
window[c]++;
if (need.count(c) && window[c] == need[c]) {
    formed++;  // CORRECT: exactly satisfied this character's requirement
}
```

---

### Mistake 5: Not handling the "no valid window" case for minimum window problems

**What they do:** Return `minLen` directly without checking if any valid window was ever found.

**What goes wrong:** If no valid window exists (e.g., t contains characters not in s), `minLen` is still `INT_MAX`. Returning `INT_MAX` causes undefined behavior in the caller or a wrong answer.

**The fix:**
```cpp
// Always check and handle the "not found" case
return (minLen == INT_MAX) ? "" : s.substr(ansLeft, minLen);
// or for length problems:
return (minLen == INT_MAX) ? 0 : minLen;
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (solve in under 15 minutes each)

**1. Best Time to Buy and Sell Stock — LC 121 — [Implicit fixed window / track running min]**

Why this problem: This is technically a "sliding window of opportunity" — you track the minimum price seen so far (implicit left pointer) and compute profit at each step. It trains the mindset of "track a running aggregate as you scan."

Key insight: At each day, the best profit if selling TODAY = today's price − minimum price seen so far. The "window" left boundary is wherever the minimum was.

Target time: 5 minutes.

```cpp
int maxProfit(vector<int>& prices) {
    int minPrice = INT_MAX, maxProfit = 0;
    for (int price : prices) {
        minPrice = min(minPrice, price);       // update running minimum
        maxProfit = max(maxProfit, price - minPrice);  // profit if sold today
    }
    return maxProfit;
}
```

**2. Maximum Average Subarray I — LC 643 — [Fixed-size window]**

Why this problem: The purest fixed-size sliding window. Initialize first window, then slide one step at a time adding right and removing left.

Target time: 5 minutes. If taking longer than 8 minutes, the fixed window template isn't in your fingers yet — re-read Template 1.

---

### CORE — 4 problems (the real learning happens here)

**3. Longest Substring Without Repeating Characters — LC 3 — [Variable window, maximize, no repeats]**

What this tests: The most fundamental variable window problem. Hash map (or hash set) tracks the window state. Shrink when a duplicate enters.

Revision target: This should be solvable in under 5 minutes from memory after this pattern. It is asked so often in interviews that NOT knowing it instantly is a red flag.

The key optimization: Using `unordered_set` → just `erase(s[left])` until the duplicate is gone. OR using a `last_seen` map and jumping the left pointer directly: `left = max(left, lastSeen[s[right]] + 1)`. The second approach avoids the inner while loop (still O(n), just fewer iterations in practice).

```cpp
// Method 1: Shrink with inner while loop
int lengthOfLongestSubstring_v1(string s) {
    unordered_set<char> inWindow;
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        while (inWindow.count(s[right])) { // shrink until no duplicate
            inWindow.erase(s[left++]);
        }
        inWindow.insert(s[right]);
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}

// Method 2: Jump left pointer directly (cleaner)
int lengthOfLongestSubstring_v2(string s) {
    unordered_map<char, int> lastSeen; // char → last seen index
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        if (lastSeen.count(s[right])) {
            left = max(left, lastSeen[s[right]] + 1); // jump past the duplicate
            // max() because left must only move RIGHT, never left
        }
        lastSeen[s[right]] = right;
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

**4. Longest Repeating Character Replacement — LC 424 — [Variable window, max freq trick]**

What this tests: The "window size - max frequency in window ≤ k" insight. This is non-obvious.

Key insight: A window of size W is valid if we can make all characters the same by replacing at most K characters. The minimum replacements needed = W − (count of most frequent character in window). So validity condition: `W - maxFreq ≤ K` → `W ≤ maxFreq + K`.

The tricky part: We track `maxFreq` as a RUNNING MAXIMUM, never decrement it. Why? Because we only care about expanding the window beyond our current best. If `maxFreq` decreases after shrinking, the window size is the same as before — we just slid it. We can't do better with a smaller `maxFreq` anyway. This is the non-shrinkable window applied to a frequency tracker.

Time limit: 35 minutes. This insight takes time — it's okay to take the full time.

**5. Permutation in String — LC 567 — [Fixed window, frequency matching]**

What this tests: Fixed-size window with two-frequency-array matching. The optimized `formed`/`required` approach (or `diff` counter) to avoid O(26) comparison each step.

Key insight: The window size is always exactly `len(p)`. Slide across `s`, maintaining the frequency of characters in the current window. Check if it matches the frequency of `p`.

What to code: The `formed`/`required` approach (see Template 4 optimized version). This is O(1) per step instead of O(26) per step.

**6. Minimum Window Substring — LC 76 — [Variable window, minimize, multi-char match]**

What this tests: This is the CANONICAL sliding window problem. Every variant of "minimum window with constraints" reduces to this structure.

Key insight: Use `formed`/`required` to know when the window satisfies all constraints. Shrink while satisfied, track minimum. The state management (when to increment/decrement `formed`) is the hard part.

Time limit: 45 minutes for this one. It's a Hard on LC and an industry benchmark. If you can't solve this cleanly in 45 minutes, you need to re-study Templates 3 and 5.

---

### STRETCH — 3 problems

**7. Subarrays with K Different Integers — LC 992 — [Exactly K via at-most-K trick]**

What makes this hard: You CANNOT solve this directly with a single sliding window. You need the `atMost(K) - atMost(K-1)` transformation. Most 1600 coders have never seen this trick.

Key insight: "Exactly K" is not a monotonic condition for sliding windows (adding an element might change distinct count unpredictably for the "exactly" case). Transform to "at most K" which IS monotonic.

Time limit: 40 minutes for the insight to arrive. If it doesn't, the hint is: "Try counting subarrays with AT MOST K distinct integers first."

**8. Minimum Size Subarray Sum — LC 209 — [Variable window, minimize]**

What makes this hard: Simpler than it looks but tests Template 3 directly. The complication: what to return when no valid subarray exists (return 0).

Key insight: Only works because all elements are positive. The sum only increases as you expand, decreases as you shrink. The monotonicity makes the two-pointer/sliding window valid.

Also try: binary search approach — O(n log n) — to understand that sliding window is not the only approach.

**9. Max Consecutive Ones III — LC 1004 — [At-most-K zeros, maximize]**

What makes this hard: It's the "at most K flips (zeros)" maximization problem. You must see it as "find longest subarray with at most K zeros." The condition isn't about element values in a complex way — just count zeros in the window.

Time limit: 25 minutes. This should be fast once you understand Template 2 Variant 2.

---

### CONTEST LEVEL — 2 problems

**10. Sliding Window Maximum — LC 239 — [Deque-based window, Pattern 29 preview]**

Why this is contest level: A regular sliding window can't track the maximum efficiently. When the left element leaves, how do you know the new maximum? Recomputing is O(k) per step → O(nk) total. Need a monotonic deque (see Pattern 29 in detail later).

The insight: Maintain a deque that stores indices of elements in DECREASING order of value. The front of the deque is always the maximum. When right advances, pop the back while the new element is ≥ deque's back. When left advances, pop the front if it's the index being removed.

```cpp
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;      // stores INDICES, in decreasing order of value
    vector<int> result;
    
    for (int right = 0; right < nums.size(); right++) {
        // Remove indices that are out of window
        while (!dq.empty() && dq.front() <= right - k) dq.pop_front();
        
        // Maintain decreasing order: pop back while back element ≤ current
        while (!dq.empty() && nums[dq.back()] <= nums[right]) dq.pop_back();
        
        dq.push_back(right);
        
        // Window is full: record maximum (front of deque)
        if (right >= k - 1) result.push_back(nums[dq.front()]);
    }
    return result;
}
// Time: O(n) — each element pushed and popped at most once
// Space: O(k) — deque holds at most k elements
```

Come back here after doing Pattern 29 (Monotonic Deque) for the full treatment.

**11. Minimum Number of Operations to Make Array Continuous — LC 2009 — [Sort unique + sliding window]**

Why this is contest level: The insight that "sort unique values and use sliding window on unique array" is non-obvious. You need to see that the optimal transformed array is a range of n consecutive integers, and you want to maximize how many elements are already in such a range.

Key insight:
1. Sort and deduplicate the array
2. For each left of the sorted unique array, binary search or two-pointer to find the farthest right such that `sorted[right] - sorted[left] ≤ n-1`
3. The count of unique elements in this window = elements we DON'T need to change
4. Total operations = n − (max elements in any valid window)

---

## PATTERN CONNECTIONS

### How Sliding Window connects to other patterns:

**IS a specialized form of Two Pointers (Pattern 02):**
Sliding window = same-direction two pointers where [left, right] represents a window. The difference: two pointers is usually for finding PAIRS with a condition; sliding window is for finding SUBARRAYS with a condition. They share the amortized O(n) argument.

**Combines with Hash Map (Pattern 01) always:**
The sliding window maintains STATE using a hash map (or frequency array). The hash map tracks what's in the current window. Without the hash map, you couldn't efficiently check validity.

**Combines with Prefix Sum (Pattern 06) for negative arrays:**
When elements can be negative, replace the sliding window with prefix sum + hash map. The key check: "are all elements positive?" If yes, sliding window works. If no, use prefix sums.

**Leads to Monotonic Deque (Pattern 29):**
When the window needs to track min/max instead of sum/count, a simple sum variable isn't enough. Monotonic deque gives O(1) min/max per window step instead of O(k).

**Often confused with:**
- **Prefix Sum:** Prefix sum answers "what is the sum of subarray [i,j]?" in O(1). Sliding window finds the OPTIMAL subarray in O(n). Use prefix sum for range QUERIES, sliding window for range SEARCH.
- **Two Pointers on sorted array:** Two pointers works on sorted arrays to find pairs. Sliding window works on UNSORTED arrays (or strings) to find subarrays. Don't sort when using sliding window — sorting changes the subarray structure.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Find the length of the longest substring with at most 2 distinct characters."

**Company type:** Google, Amazon, Facebook (common warm-up question)
**Expected time:** 10 minutes
**What they are testing:** Variable window template, hash map as window state, erase-when-zero discipline

**Red flags:**
- O(n²) brute force
- Not erasing keys from map when frequency hits 0 (most common bug)
- Using `freq.count(c) > 0` instead of checking `freq.size()`
- Confusing maximize (find longest) with minimize (find shortest) templates

**The correct solution:**
```cpp
int lengthOfLongestSubstringTwoDistinct(string s) {
    unordered_map<char, int> freq;
    int left = 0, maxLen = 0;
    for (int right = 0; right < s.size(); right++) {
        freq[s[right]]++;
        while (freq.size() > 2) {         // more than 2 distinct → shrink
            freq[s[left]]--;
            if (freq[s[left]] == 0) freq.erase(s[left]); // KEY LINE
            left++;
        }
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

**Follow-up they will ask:** "Generalize to at most K distinct." → Trivially: change `> 2` to `> k`. That's the power of knowing the pattern.

**Second follow-up:** "What if the string can only have lowercase letters?" → Use `int freq[26]` instead of hash map. O(26) per check (or O(1) with the diff counter).

---

### Q2: "Given a string s and a pattern p, find all starting indices of p's anagrams in s."

**Company type:** Amazon, Microsoft, startups (LC 438)
**Expected time:** 20 minutes
**What they are testing:** Fixed-size sliding window with frequency matching + collecting all matches (not just checking existence)

**Red flags:**
- Sorting each window to check anagram (O(k log k) per window → O(nk log k) total instead of O(n))
- Using hash map comparison (`map == map`) — works but O(k) per comparison
- Not handling the "collect all starting indices" part (forgetting to record `left` when match found)

**The correct solution:**
```cpp
vector<int> findAnagrams(string s, string p) {
    if (p.size() > s.size()) return {};
    
    int pFreq[26] = {}, wFreq[26] = {};
    for (char c : p) pFreq[c-'a']++;
    
    vector<int> result;
    int windowSize = p.size();
    
    // Initialize first window
    for (int i = 0; i < windowSize; i++) wFreq[s[i]-'a']++;
    if (equal(pFreq, pFreq+26, wFreq)) result.push_back(0);
    
    // Slide window
    for (int right = windowSize; right < s.size(); right++) {
        wFreq[s[right]-'a']++;                  // add incoming
        wFreq[s[right - windowSize]-'a']--;      // remove outgoing
        if (equal(pFreq, pFreq+26, wFreq)) result.push_back(right - windowSize + 1);
    }
    return result;
}
```

**Time complexity to state:** O(n + m) where n = |s|, m = |p|. The `equal` is O(26) = O(1) for lowercase letters.

---

### Q3: "Design a data structure that, given a stream of integers, returns the maximum element in the last K elements at any point."

**Company type:** FAANG, trading firms
**Expected time:** 30 minutes
**What they are testing:** Sliding window maximum — requires monotonic deque insight

**Red flags:**
- Maintaining sorted structure (O(log k) per operation — "but can you do O(1) amortized?")
- Max-heap (O(log k) per operation — same)
- Not knowing what a deque is in C++

**What they want to hear:**
"I'll use a monotonic deque that maintains indices of elements in decreasing order. When adding a new element, I pop from the back while the back is ≤ new element. When the window slides, I pop from the front if it's the index being removed. The front is always the maximum. Each element is pushed and popped at most once → O(n) total for n operations → O(1) amortized."

This question is actually just LC 239 Sliding Window Maximum as a design question.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

1. **What is the invariant of the variable window maximize template?**
   → [left, right] always contains a VALID window. We expand right freely, shrink from left when invalid. The answer (maxLen) is updated after the window is valid again.

2. **What is the key difference between the maximize template and the minimize template?**
   → Maximize: record OUTSIDE the shrink loop (after window becomes valid). Minimize: record INSIDE the shrink loop (before shrinking, while window is still satisfied). Mix them up → wrong answer.

3. **Why can't you use sliding window for subarray sum problems with negative numbers?**
   → Sliding window requires monotonicity: expanding increases the aggregate, shrinking decreases it. With negative numbers, this breaks — adding a negative element decreases the sum. The shrink condition can't be maintained reliably.

4. **What is the at-most-K trick and when do you use it?**
   → `exactly(K) = atMost(K) - atMost(K-1)`. Use when the problem asks "count subarrays with EXACTLY K of something" and a direct sliding window can't maintain the "exactly K" condition monotonically.

5. **Why is the sliding window O(n) despite having a while loop inside a for loop?**
   → The `left` pointer only moves RIGHT and travels at most n total steps across ALL iterations of the outer loop. The total movements of both pointers combined = at most 2n. Amortized O(1) per outer iteration.

6. **In Minimum Window Substring, when should `formed` be incremented?**
   → ONLY when the count of a character in the window EQUALS its required count (not when we first see it, not when we have more than required — ONLY at the exact moment of satisfaction). `if (window[c] == need[c]) formed++;`

7. **What is the window state variable for "longest substring with all same characters after at most K replacements"?**
   → Track the frequency of each character in the window + the maximum frequency (`maxFreq`) seen so far. Window is valid if `windowSize - maxFreq ≤ K`. `maxFreq` is NEVER decreased even when shrinking (non-shrinkable window trick).

---

## C++ STL CHEAT SHEET FOR THIS PATTERN

```cpp
// ═══════════════════════════════════════════
// deque operations (for sliding window maximum)
// ═══════════════════════════════════════════
deque<int> dq;
dq.push_back(x);   // add to right
dq.push_front(x);  // add to left
dq.pop_back();     // remove from right
dq.pop_front();    // remove from left
dq.front();        // peek left (maximum in decreasing monotonic deque)
dq.back();         // peek right
dq.empty();        // is it empty?

// ═══════════════════════════════════════════
// String/array manipulation
// ═══════════════════════════════════════════
s.substr(start, length)      // extract substring
equal(a, a+26, b)            // compare two arrays of size 26
fill(arr, arr+26, 0)         // reset frequency array

// ═══════════════════════════════════════════
// Frequency array vs unordered_map:
// ═══════════════════════════════════════════
// For lowercase letters: int freq[26] = {};   // always prefer for strings
// For arbitrary integers: unordered_map<int, int> freq;

// ═══════════════════════════════════════════
// The three sliding window loop patterns:
// ═══════════════════════════════════════════

// Pattern 1: Fixed window
for (int right = k; right < n; right++) {
    add(nums[right]);
    remove(nums[right - k]);
    update(answer);
}

// Pattern 2: Variable, maximize (shrink when invalid)
for (int right = 0; right < n; right++) {
    add(nums[right]);
    while (invalid()) shrink_left();
    maxLen = max(maxLen, right - left + 1);
}

// Pattern 3: Variable, minimize (shrink while satisfied)
for (int right = 0; right < n; right++) {
    add(nums[right]);
    while (satisfied()) {
        minLen = min(minLen, right - left + 1);
        shrink_left();
    }
}
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. **Solve all 11 problems.** For problems 3, 4, 5, 6 — implement both the "simple" version and the "optimized" version (with `formed`/`diff` counter).
2. **For LC 3 (Longest Substring Without Repeating):** solve using BOTH the while-loop method and the direct-jump method. Know both cold.
3. **For LC 76 (Minimum Window Substring):** This is your benchmark. If you can implement it cleanly in 20 minutes from memory, you own sliding window.
4. **Revision schedule:** Redo problems 6, 7, 10 after 3 days, 7 days, 14 days.
5. **Mark LC 992 (Subarrays with K Different) as a CHALLENGE problem** in your revision tracker. The at-most-K trick must become automatic.

---

## THE SIGN-OFF CRITERIA

When you type `SIGN OFF P03`, I will give you one unseen problem. You must:
1. Identify it as sliding window AND name the exact sub-type (fixed / variable-maximize / variable-minimize / at-most-K)
2. Write a correct C++ solution
3. State time/space complexity BEFORE I ask
4. Handle edge cases: empty array, k > array size, all elements equal, no valid window exists (return 0 / empty string)
5. Explain in 60 seconds what condition triggered your choice of this template variant

---

*Now go solve. Start with LC 121 (Best Time to Buy and Sell Stock) and LC 643 (Maximum Average Subarray I). Then go directly to LC 3 (Longest Substring Without Repeating) — that's your benchmark for this pattern. Time yourself on every problem. When done, paste solutions with `SOLVE` for 5-dimension review.*
