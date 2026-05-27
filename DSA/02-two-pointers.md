# Pattern 02 — Two Pointers
## Difficulty Level: FOUNDATION
## Interview Frequency: ALWAYS
## Estimated time to master: 3-4 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Two Pointers is the pattern that teaches you to think about **what is always true** — the invariant — rather than just iterating and checking. Every two-pointer solution maintains a contractual guarantee about the relationship between the left and right pointer positions. If you cannot state that guarantee, you don't understand your own code.

Most 1600-rated coders can solve Two Sum II with two pointers. They "got lucky" — they saw someone else do it. But when I put them in front of Trapping Rain Water or 3Sum with duplicates, they freeze. Why? Because they never understood WHY the pointer moves the way it does. They memorized the direction without internalizing the invariant.

Mastery means this: for any two-pointer problem, you can finish the sentence "At all times, the left pointer points to ___ and the right pointer points to ___, and together they guarantee ___." Once you can fill in that blank for any variant, the code practically writes itself.

---

## ELI5 — THE CORE IDEA

Imagine you're searching for two people in a theater where seats are arranged in one row, sorted by height from left to right (shortest on left, tallest on right).

You want to find two people whose combined height equals exactly 180cm.

**Bad approach:** Try every pair. O(n²). Exhausting.

**Smart approach:** Start with one person from the left end and one from the right end. Add their heights:
- Too short? The left person is too short — swap them for someone taller → move left pointer RIGHT
- Too tall? The right person is too tall — swap them for someone shorter → move right pointer LEFT
- Exactly right? Found it!

**Why does this work?** Because the array is sorted, moving the left pointer right can ONLY increase the sum. Moving the right pointer left can ONLY decrease the sum. At every step, you are making a GUARANTEED progress — you eliminate at least one candidate. You never need to re-examine a pair you already eliminated.

That is the two-pointer invariant: **sorted order + directional movement = guaranteed elimination.**

The deeper version: the same logic applies to any problem where you can prove that moving one pointer in a direction eliminates impossible candidates. The "sorting" is just the most common way to establish that proof.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:
Any problem where you need to:
1. **Find a pair/triplet/subarray satisfying a condition** in O(n) instead of O(n²)
2. **Process elements from both ends simultaneously** (palindrome check, partition)
3. **Track two positions in an array** that move in a coordinated way
4. **In-place modification** while reading (remove duplicates, move zeros)

### The invariant (THIS is what you must always know):
A two-pointer solution maintains an invariant — a statement that is **always true** at the start of each iteration. The invariant tells you:
- What the left pointer represents
- What the right pointer represents
- What you can conclude about elements OUTSIDE the window [left, right]

### What triggers pointer movement:
- **Opposite-direction pointers:** Move the pointer that makes progress toward the target. The sorted order guarantees this eliminates impossible cases.
- **Same-direction pointers:** The "write" pointer only advances when a valid element is found. The "read" pointer always advances.
- **Fast-slow pointers:** Fast advances 2x, slow advances 1x. If there's a cycle, they meet; if not, fast reaches null first.

### What the answer is extracted from:
Either the pointer positions themselves (indices, window size), or accumulated data while pointers move (count of valid states, max/min of something tracked).

---

## WHEN TO RECOGNIZE THIS PATTERN

### Signals that SCREAM "use two pointers":

**Signal 1:** "Find a pair in a sorted array with sum = target"
→ Left-right pointers. The sorted order makes binary elimination possible.

**Signal 2:** "Remove duplicates in-place from sorted array" / "Move zeros to end"
→ Same-direction write pointer. One pointer reads, one pointer writes.

**Signal 3:** "Is this string a palindrome?" / "Reverse in-place"
→ Opposite-direction pointers meeting in the middle.

**Signal 4:** "Find triplets / quadruplets summing to target"
→ Sort + fix one element + two-pointer on remaining array.

**Signal 5:** "Find cycle in linked list" / "Middle of linked list"
→ Fast-slow pointers (Floyd's algorithm).

**Signal 6:** "Partition array in-place" (Dutch national flag, QuickSort partition)
→ Write pointer + read pointer.

**Signal 7:** Constraint is "in-place, O(1) extra space" on an array problem
→ Strong hint to use two pointers (no extra hash map allowed).

### Signals that LOOK LIKE two pointers but are NOT:

**Fake signal 1:** "Find a pair in an UNSORTED array"
→ Use hash map (Pattern 01) instead. Two pointers needs sorted order. Sorting + two pointers = O(n log n) vs hash map = O(n).

**Fake signal 2:** "Find longest subarray with some property"
→ Probably sliding window (Pattern 03). Two pointers is for fixed-size matching, not variable-size maximization.

**Fake signal 3:** "Find pair in sorted array where one element is fixed and other is in a different array"
→ Binary search might be cleaner than two pointers.

---

## THE TEMPLATES (C++)

### Template 1: Opposite-Direction Pointers (Left-Right)

```cpp
// Opposite Direction Two Pointer Template
// Precondition: array is sorted (or can be sorted)
// Time: O(n) after sorting | Space: O(1)

// INVARIANT: 
//   - Everything to the LEFT of left pointer has been ruled out (sum too small OR already processed)
//   - Everything to the RIGHT of right pointer has been ruled out (sum too large OR already processed)
//   - The answer, if it exists, lies within [left, right]

bool twoSumSorted(vector<int>& nums, int target) {
    int left = 0;
    int right = (int)nums.size() - 1;
    
    while (left < right) {                    // NEVER let left == right (would use same element twice)
        int currentSum = nums[left] + nums[right];
        
        if (currentSum == target) {
            return true;                       // Found it
        } else if (currentSum < target) {
            left++;                            // Sum too small → need bigger element → move left RIGHT
        } else {
            right--;                           // Sum too large → need smaller element → move right LEFT
        }
    }
    
    return false;
}

// HOW TO ADAPT:
// 1. Change stopping condition: left < right (no overlap) vs left <= right (allow overlap)
// 2. Change what you do when condition is met: return, count, collect all pairs
// 3. Change the "move" logic based on what shrinks the search space
// 4. For "count pairs": when currentSum == target, count += (right - left) might be needed
//    (if all elements between left and right form valid pairs — e.g., two sum count)
```

### Template 2: Same-Direction Pointers (Read/Write Pattern)

```cpp
// Same Direction Two Pointer Template (Read/Write)
// Used for: remove duplicates, move zeros, filter in-place
// Time: O(n) | Space: O(1)

// INVARIANT:
//   - Everything at indices [0, write) satisfies the desired property
//   - read pointer has processed everything at [0, read)
//   - elements at [write, read) are "garbage" (will be overwritten)

int removeDuplicates(vector<int>& nums) {
    if (nums.empty()) return 0;
    
    int write = 1;                        // next position to write valid element
                                          // (first element is always valid)
    
    for (int read = 1; read < nums.size(); read++) {
        // Condition: should we keep nums[read]?
        if (nums[read] != nums[write - 1]) {  // it's different from last written
            nums[write] = nums[read];          // write it
            write++;                            // advance write pointer
        }
        // If same as last: skip (read advances but write stays)
    }
    
    return write; // array length of valid portion
}

// HOW TO ADAPT:
// 1. Change the "keep" condition:
//    - Remove duplicates: nums[read] != nums[write-1]
//    - Remove element val: nums[read] != val
//    - Keep at most 2 duplicates: write < 2 || nums[read] != nums[write-2]
//    - Move zeros to end: nums[read] != 0
// 2. The write pointer only moves when we KEEP an element
// 3. read always moves forward — it reads every element exactly once
```

### Template 3: Three Pointers — 3Sum Pattern

```cpp
// Three Pointer Template (Fix one, two-pointer the rest)
// Precondition: sort the array first
// Time: O(n²) | Space: O(1) extra (O(n) or O(k) for result)

// INVARIANT:
//   - Outer loop fixes nums[i] as the first element
//   - Inner left-right search finds the pair summing to -nums[i]
//   - Duplicate skipping ensures no duplicate triplets

vector<vector<int>> threeSum(vector<int>& nums) {
    sort(nums.begin(), nums.end()); // MUST sort first
    vector<vector<int>> result;
    
    for (int i = 0; i < (int)nums.size() - 2; i++) {
        // Skip duplicate values for the fixed element
        if (i > 0 && nums[i] == nums[i - 1]) continue;
        
        // Early termination: if smallest possible sum > 0, no solution
        if (nums[i] > 0) break;
        
        int left = i + 1;
        int right = (int)nums.size() - 1;
        int target = -nums[i];
        
        while (left < right) {
            int sum = nums[left] + nums[right];
            
            if (sum == target) {
                result.push_back({nums[i], nums[left], nums[right]});
                
                // Skip duplicates for left pointer
                while (left < right && nums[left] == nums[left + 1]) left++;
                // Skip duplicates for right pointer
                while (left < right && nums[right] == nums[right - 1]) right--;
                
                left++;
                right--;
            } else if (sum < target) {
                left++;
            } else {
                right--;
            }
        }
    }
    
    return result;
}

// HOW TO ADAPT FOR 4SUM:
// - Add another outer loop for nums[j]
// - Inner two pointers find pair summing to target - nums[i] - nums[j]
// - Two levels of duplicate skipping
// - Time: O(n³)

// HOW TO ADAPT FOR "COUNT triplets":
// - Instead of collecting triplets, count valid (left, right) pairs
// - When sum < target: all indices (left, left+1, ..., right-1) with right form valid pairs
//   → count += right - left, then left++
```

### Template 4: Fast-Slow Pointers (Floyd's Algorithm)

```cpp
// Fast-Slow Pointer Template (Cycle Detection)
// Used for: cycle in linked list, middle of linked list, cycle entry point
// Time: O(n) | Space: O(1)

// PHASE 1: Detect if cycle exists
// INVARIANT: If cycle exists, fast and slow WILL meet inside the cycle.
// If no cycle, fast reaches null first.

bool hasCycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;          // move 1 step
        fast = fast->next->next;    // move 2 steps
        
        if (slow == fast) return true; // they met → cycle exists
    }
    
    return false; // fast reached null → no cycle
}

// PHASE 2: Find the entry point of the cycle
// PROOF: Let distance to cycle entry = F, cycle length = C, meeting point = H steps into cycle
// When they meet: slow traveled F + H, fast traveled F + H + C (one full extra loop)
// Since fast travels 2x: 2(F + H) = F + H + C → F = C - H
// So: reset slow to head, keep fast at meeting point, both move 1 step at a time
// They meet at the cycle entry!

ListNode* detectCycleEntry(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    // Phase 1: find meeting point
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;
    }
    
    if (fast == nullptr || fast->next == nullptr) return nullptr; // no cycle
    
    // Phase 2: find entry point
    slow = head; // reset slow to head
    // fast stays at meeting point
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next; // both move 1 step now
    }
    
    return slow; // this is the cycle entry
}

// FINDING MIDDLE OF LINKED LIST:
// When fast reaches end, slow is at middle
ListNode* findMiddle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast->next != nullptr && fast->next->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
    }
    
    return slow; // slow is now at middle
    // For odd length [1,2,3,4,5]: slow = 3 (exact middle)
    // For even length [1,2,3,4]: slow = 2 (first of two middles)
    // Adjust: use fast != nullptr && fast->next != nullptr to get second middle for even
}
```

### Template 5: Dutch National Flag (Three-Way Partition)

```cpp
// Dutch National Flag Template (Three-Way Partition)
// Partitions array into [< pivot | = pivot | > pivot]
// Used for: sort colors, partition with three categories
// Time: O(n) | Space: O(1)

// INVARIANT at all times:
//   - [0, low)       → elements LESS than pivot (region 0)
//   - [low, mid)     → elements EQUAL to pivot (region 1)  ← mid pointer is here
//   - [mid, high]    → UNKNOWN (not yet processed)
//   - (high, n-1]    → elements GREATER than pivot (region 2)

void sortColors(vector<int>& nums) {
    int low = 0;        // boundary of "0" region (exclusive)
    int mid = 0;        // current element being examined
    int high = (int)nums.size() - 1; // boundary of "2" region (exclusive from right)
    
    while (mid <= high) {
        if (nums[mid] == 0) {
            swap(nums[low], nums[mid]);
            low++;
            mid++;
            // Why increment both? Because [low, mid) had only 1s before,
            // swapping brought a 0 to low position (good),
            // and whatever was at low was a 1 (goes to mid position, still in 1-region)
        } else if (nums[mid] == 1) {
            mid++;
            // It's a 1 → already in correct region, just move forward
        } else { // nums[mid] == 2
            swap(nums[mid], nums[high]);
            high--;
            // Why NOT increment mid? Because nums[high] was unknown,
            // we need to examine what came to position mid (it's now unknown)
        }
    }
}

// HOW TO ADAPT:
// 1. Change 0/1/2 to your three categories
// 2. For two-way partition (just 0 and non-0): use write pointer (Template 2)
// 3. For K-way partition: usually sort or count-then-fill
```

### Template 6: Two Pointers with State Tracking (Trapping Rain Water)

```cpp
// Two Pointers with Tracked Maximums
// Used when: answer at each position depends on max seen from both sides
// Time: O(n) | Space: O(1)

// Key insight: water at position i = min(maxLeft[i], maxRight[i]) - height[i]
// Instead of precomputing all maxLeft and maxRight arrays (O(n) space),
// we use two pointers and prove we can compute locally.

// INVARIANT:
//   left pointer: maxLeft is correct for everything to its left
//   right pointer: maxRight is correct for everything to its right
//   We process whichever side has the SMALLER maximum
//   because that side's water level is determined (bounded by smaller max)

int trap(vector<int>& height) {
    int left = 0;
    int right = (int)height.size() - 1;
    int maxLeft = 0;    // max height seen from left so far
    int maxRight = 0;   // max height seen from right so far
    int water = 0;
    
    while (left < right) {
        if (height[left] <= height[right]) {
            // Left side is the bottleneck
            // Water at left is determined by maxLeft (right side is definitely >= maxLeft)
            if (height[left] >= maxLeft) {
                maxLeft = height[left]; // new max, no water here (it would overflow)
            } else {
                water += maxLeft - height[left]; // trapped water at this position
            }
            left++;
        } else {
            // Right side is the bottleneck
            if (height[right] >= maxRight) {
                maxRight = height[right];
            } else {
                water += maxRight - height[right];
            }
            right--;
        }
    }
    
    return water;
}

// WHY we process the side with SMALLER max:
// If maxLeft < maxRight:
//   - For position left: water level = min(maxLeft, something ≥ maxLeft) = maxLeft
//   - We KNOW the water at left without looking right. Process it now.
// If maxRight < maxLeft: symmetric argument for right side.
```

### Template 7: Two Pointers for Minimum Window (Greedy with Two Arrays)

```cpp
// Boats to Save People type: greedy matching with two pointers
// Used when: pair lightest with heaviest if possible, otherwise heaviest alone
// Time: O(n log n) — sort dominates | Space: O(1)

int numRescueBoats(vector<int>& people, int limit) {
    sort(people.begin(), people.end());
    int left = 0;
    int right = (int)people.size() - 1;
    int boats = 0;
    
    while (left <= right) {
        if (people[left] + people[right] <= limit) {
            // Lightest and heaviest fit together
            left++;
            right--;
        } else {
            // Heaviest alone (can't pair with anyone remaining)
            right--;
        }
        boats++;
    }
    
    return boats;
}

// WHY this greedy is correct (exchange argument):
// If the heaviest person CAN pair with someone, pairing them with the LIGHTEST
// is at least as good as pairing them with anyone else.
// Proof: suppose we pair heavy H with medium M, and light L goes alone.
//   - boats used = 2
//   - Alternative: pair H with L, M alone → boats used = 2 (same)
//   - Alternative: pair H with L, M with someone else → potentially 2 or fewer boats (better or equal)
// So always try heaviest with lightest first.
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

---

### VARIANT 1: Opposite-Direction on Sorted Array

**The core idea:** Sort the array. Left and right start at ends. At each step, the sum/product/difference tells you which pointer to move. Move the pointer that makes the current metric approach the target.

**When you have it:** Sorted array (or you can sort) + find pair/triplet with a numeric property.

**The invariant spelled out:** At every step, all elements to the LEFT of `left` are too small to form any valid pair with anything in `[left, right]`. All elements to the RIGHT of `right` are too large. The valid answer (if any) is in `[left, right]`.

**All the cases:**

```cpp
// Case 1: Find pair with sum == target (return indices or values)
// → Move left right if sum < target, right left if sum > target

// Case 2: Count pairs with sum < target
// When sum < target: ALL pairs (left, left+1), (left, left+2), ..., (left, right) are valid
// → count += right - left; left++;   [This makes it O(n) not O(n²)!]
int countPairsLessThan(vector<int>& nums, int target) {
    sort(nums.begin(), nums.end());
    int count = 0, left = 0, right = nums.size() - 1;
    while (left < right) {
        if (nums[left] + nums[right] < target) {
            count += right - left; // right can pair with left, left+1, ..., right-1
            left++;
        } else {
            right--;
        }
    }
    return count;
}

// Case 3: Find pair that maximizes/minimizes the sum (sorted)
// → After sorting, max sum = last two, min sum = first two. Trivial.
// → But "maximize product with constraint" needs two pointers:
//   e.g., maximize nums[i] * nums[j] where i < j and nums[i] + nums[j] ≤ target
//   Sort, then two pointers

// Case 4: All pairs summing to target (not just one)
// → When found: collect, then skip ALL duplicates of both pointers before moving
while (left < right && nums[left] == nums[left+1]) left++;
while (left < right && nums[right] == nums[right-1]) right--;
left++; right--;

// Case 5: Closest pair to target (no exact match needed)
// → Track minimum |sum - target| across all moves
int closestPair(vector<int>& nums, int target) {
    sort(nums.begin(), nums.end());
    int closest = INT_MAX, left = 0, right = nums.size() - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (abs(sum - target) < abs(closest - target)) closest = sum;
        if (sum < target) left++;
        else if (sum > target) right--;
        else return sum; // exact match
    }
    return closest;
}
```

---

### VARIANT 2: 3Sum and 4Sum — Extending with Fixed Outer Loops

**The 3Sum pattern** is just: fix one element + two-pointer on the rest.

**CRITICAL: Duplicate skipping.** This is where 80% of candidates fail. There are THREE places you must skip duplicates:

```cpp
// 1. The outer loop: skip if nums[i] == nums[i-1]
//    → prevents fixing the same value twice as the first element
if (i > 0 && nums[i] == nums[i-1]) continue;

// 2. After finding a valid triplet: skip duplicates for BOTH left and right
//    → prevents recording same triplet multiple times
while (left < right && nums[left] == nums[left+1]) left++;
while (left < right && nums[right] == nums[right-1]) right--;
left++; right--;
```

**WHY you skip BEFORE moving in the outer loop (not `continue` after):**
```cpp
// WRONG:
for (int i = 0; i < n - 2; i++) {
    if (nums[i] == nums[i+1]) continue; // WRONG: compares with NEXT, not PREVIOUS
    // This skips the LAST occurrence, not the duplicates
}

// CORRECT:
for (int i = 0; i < n - 2; i++) {
    if (i > 0 && nums[i] == nums[i-1]) continue; // compare with PREVIOUS
    // This skips the 2nd, 3rd, ... duplicate (keeps only FIRST)
}
```

**4Sum pattern:**
```cpp
vector<vector<int>> fourSum(vector<int>& nums, int target) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> result;
    int n = nums.size();
    
    for (int i = 0; i < n - 3; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue; // skip outer duplicates
        
        // Early termination optimizations:
        if ((long long)nums[i] + nums[i+1] + nums[i+2] + nums[i+3] > target) break; // min sum already > target
        if ((long long)nums[i] + nums[n-3] + nums[n-2] + nums[n-1] < target) continue; // max sum < target
        
        for (int j = i + 1; j < n - 2; j++) {
            if (j > i + 1 && nums[j] == nums[j-1]) continue; // skip inner duplicates
            
            // Same early termination for inner loop
            if ((long long)nums[i] + nums[j] + nums[j+1] + nums[j+2] > target) break;
            if ((long long)nums[i] + nums[j] + nums[n-2] + nums[n-1] < target) continue;
            
            int left = j + 1, right = n - 1;
            long long remain = (long long)target - nums[i] - nums[j];
            
            while (left < right) {
                long long sum = (long long)nums[left] + nums[right];
                if (sum == remain) {
                    result.push_back({nums[i], nums[j], nums[left], nums[right]});
                    while (left < right && nums[left] == nums[left+1]) left++;
                    while (left < right && nums[right] == nums[right-1]) right--;
                    left++; right--;
                } else if (sum < remain) left++;
                else right--;
            }
        }
    }
    return result;
}

// IMPORTANT: Use long long for 4Sum! nums[i] can be up to 10^9, 
// four of them summed can overflow int.
```

---

### VARIANT 3: Same-Direction Pointers (Write Pointer Pattern)

**The core idea:** One pointer (`read`) scans every element. One pointer (`write`) marks where the next valid element should go. When `read` finds a valid element, copy to `write` and advance both. When invalid, only advance `read`.

**The invariant:** `[0, write)` always contains the valid prefix of the answer.

**All the variations:**

```cpp
// Remove element val in-place (LC 27):
int removeElement(vector<int>& nums, int val) {
    int write = 0;
    for (int read = 0; read < nums.size(); read++) {
        if (nums[read] != val) {          // keep condition: NOT the value to remove
            nums[write++] = nums[read];
        }
    }
    return write;
}

// Move zeros to end, maintain relative order (LC 283):
void moveZeroes(vector<int>& nums) {
    int write = 0;
    for (int read = 0; read < nums.size(); read++) {
        if (nums[read] != 0) {             // keep condition: not zero
            nums[write++] = nums[read];
        }
    }
    while (write < nums.size()) nums[write++] = 0; // fill rest with zeros
}

// Remove duplicates - keep at most 1 (LC 26):
int removeDuplicates(vector<int>& nums) {
    int write = 0;
    for (int read = 0; read < nums.size(); read++) {
        if (write == 0 || nums[read] != nums[write - 1]) { // new unique element
            nums[write++] = nums[read];
        }
    }
    return write;
}

// Remove duplicates - keep at most 2 (LC 80):
int removeDuplicates2(vector<int>& nums) {
    int write = 0;
    for (int read = 0; read < nums.size(); read++) {
        // Keep if: fewer than 2 elements written OR differs from element 2 positions back
        if (write < 2 || nums[read] != nums[write - 2]) {
            nums[write++] = nums[read];
        }
    }
    return write;
}
// GENERALIZATION: Keep at most K duplicates → condition: write < k || nums[read] != nums[write - k]
```

---

### VARIANT 4: Fast-Slow Pointers (Floyd's Cycle Detection)

**The core idea:** Two pointers move at different speeds. In a cyclic structure, the faster one laps the slower one. In a linear structure, the faster one reaches the end first.

**The key facts you MUST know cold:**

```
1. Cycle detection: fast(2x) + slow(1x) → they meet IFF cycle exists
2. Meeting point distance: F = C - H (F = head to cycle start, H = distance into cycle at meeting)
3. Finding cycle start: reset slow to head, both move 1x → meet at cycle start
4. Middle of list: fast reaches end when slow is at middle
5. Kth from end: advance fast K steps first, then move both 1x → slow is at (n-K)th
```

**Finding Kth node from end:**
```cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    // Use a dummy to handle removing the head
    ListNode dummy(0);
    dummy.next = head;
    
    ListNode* fast = &dummy;
    ListNode* slow = &dummy;
    
    // Advance fast n+1 steps (n+1 so slow stops at node BEFORE the target)
    for (int i = 0; i <= n; i++) fast = fast->next;
    
    // Move both until fast reaches null
    while (fast != nullptr) {
        slow = slow->next;
        fast = fast->next;
    }
    
    // slow is now at the node before the one to remove
    slow->next = slow->next->next;
    
    return dummy.next;
}
```

**Happy Number problem (fast-slow on abstract cycle):**
```cpp
// Fast-slow pointers don't need a linked list — any structure that can cycle works
bool isHappy(int n) {
    auto digitSquareSum = [](int x) {
        int sum = 0;
        while (x > 0) { int d = x % 10; sum += d * d; x /= 10; }
        return sum;
    };
    
    int slow = n, fast = n;
    do {
        slow = digitSquareSum(slow);           // 1 step
        fast = digitSquareSum(digitSquareSum(fast)); // 2 steps
    } while (slow != fast);
    
    return slow == 1; // if they meet at 1, it's happy; otherwise stuck in cycle
}
```

---

### VARIANT 5: Container With Most Water — The "Move Lesser Side" Proof

This problem is the best example of **why** the two-pointer invariant works. Many candidates solve it but cannot explain it. Interviewers love asking "why do you move the shorter side?"

**The proof:**

```
Suppose height[left] <= height[right].
Current area = height[left] * (right - left)

If we move the RIGHT pointer left:
  New area = min(height[left], height[right-1]) * (right - 1 - left)
  Since min(height[left], height[right-1]) ≤ height[left]  (bounded by left)
  And (right - 1 - left) < (right - left)                  (smaller width)
  → New area ≤ old area
  → Moving right pointer CANNOT improve the answer when left is the bottleneck!
  → We can safely eliminate all (left, right), (left, right-1), ..., (left, left+1)
  → So we move LEFT instead
```

**The code:**
```cpp
int maxArea(vector<int>& height) {
    int left = 0, right = (int)height.size() - 1;
    int maxWater = 0;
    
    while (left < right) {
        int water = min(height[left], height[right]) * (right - left);
        maxWater = max(maxWater, water);
        
        // Move the SHORTER side — moving the taller side can only decrease water
        if (height[left] <= height[right]) {
            left++;
        } else {
            right--;
        }
    }
    
    return maxWater;
}
```

**When an interviewer asks "why?": state the proof above in 30 seconds. This is what separates 2000+ coders.**

---

### VARIANT 6: Trapping Rain Water — Two Approaches

This is THE classic hard two-pointer problem. You must know BOTH approaches.

**Approach 1: Precomputed prefix/suffix max arrays** — easy to understand but O(n) space:
```cpp
int trap_v1(vector<int>& height) {
    int n = height.size();
    vector<int> maxLeft(n), maxRight(n);
    
    maxLeft[0] = height[0];
    for (int i = 1; i < n; i++) maxLeft[i] = max(maxLeft[i-1], height[i]);
    
    maxRight[n-1] = height[n-1];
    for (int i = n-2; i >= 0; i--) maxRight[i] = max(maxRight[i+1], height[i]);
    
    int water = 0;
    for (int i = 0; i < n; i++) {
        water += min(maxLeft[i], maxRight[i]) - height[i];
    }
    return water;
}
```

**Approach 2: Two pointers O(1) space** (see Template 6 above). Use this in interviews for better impression.

**The key insight for two-pointer approach:**
"I don't need to know BOTH max values to compute water at the current position. If I know maxLeft and maxRight values, and one of them is smaller, water at that side is determined ENTIRELY by the smaller max — the other side doesn't matter."

---

### VARIANT 7: Palindrome Check — Simplest Opposite-Direction Use

```cpp
bool isPalindrome(string s) {
    int left = 0, right = (int)s.size() - 1;
    
    while (left < right) {
        // Skip non-alphanumeric characters (for LC 125)
        while (left < right && !isalnum(s[left])) left++;
        while (left < right && !isalnum(s[right])) right--;
        
        if (tolower(s[left]) != tolower(s[right])) return false;
        left++;
        right--;
    }
    
    return true;
}

// In-place reverse of array (basis for many problems):
void reverseArray(vector<int>& arr, int left, int right) {
    while (left < right) {
        swap(arr[left++], arr[right--]);
    }
}
```

---

### VARIANT 8: Two Pointers on Two Different Arrays

**Merge two sorted arrays, intersection, union:**

```cpp
// Merge two sorted arrays into one (classic):
vector<int> mergeSorted(vector<int>& a, vector<int>& b) {
    vector<int> result;
    int i = 0, j = 0;
    
    while (i < a.size() && j < b.size()) {
        if (a[i] <= b[j]) result.push_back(a[i++]);
        else result.push_back(b[j++]);
    }
    while (i < a.size()) result.push_back(a[i++]);
    while (j < b.size()) result.push_back(b[j++]);
    
    return result;
}

// Intersection of two sorted arrays:
vector<int> intersect(vector<int>& a, vector<int>& b) {
    vector<int> result;
    int i = 0, j = 0;
    
    while (i < a.size() && j < b.size()) {
        if (a[i] == b[j]) {
            result.push_back(a[i]);
            i++; j++;
        } else if (a[i] < b[j]) i++;
        else j++;
    }
    return result;
}

// Interval intersection (LC 986):
// Two pointers on two sorted interval lists
vector<vector<int>> intervalIntersection(vector<vector<int>>& A, vector<vector<int>>& B) {
    vector<vector<int>> result;
    int i = 0, j = 0;
    
    while (i < A.size() && j < B.size()) {
        int lo = max(A[i][0], B[j][0]);
        int hi = min(A[i][1], B[j][1]);
        
        if (lo <= hi) result.push_back({lo, hi}); // overlapping region
        
        // Advance the interval that ends earlier
        if (A[i][1] < B[j][1]) i++;
        else j++;
    }
    return result;
}
```

---

### VARIANT 9: Shortest Unsorted Subarray — Finding Boundaries

```cpp
// LC 581: Find shortest subarray to sort so entire array is sorted
// Approach: find leftmost and rightmost elements that are "out of order"

int findUnsortedSubarray(vector<int>& nums) {
    int n = nums.size();
    int left = -1, right = -2; // Default: already sorted (length 0)
    int maxSeen = nums[0], minSeen = nums[n-1];
    
    // Forward pass: find rightmost element smaller than max seen so far
    // (this element must be moved right)
    for (int i = 1; i < n; i++) {
        maxSeen = max(maxSeen, nums[i]);
        if (nums[i] < maxSeen) right = i; // this element is out of place
    }
    
    // Backward pass: find leftmost element larger than min seen from right
    // (this element must be moved left)
    for (int i = n - 2; i >= 0; i--) {
        minSeen = min(minSeen, nums[i]);
        if (nums[i] > minSeen) left = i; // this element is out of place
    }
    
    return right - left + 1;
}
// INVARIANT: right marks the rightmost boundary of the unsorted subarray
// left marks the leftmost boundary
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Variant | Time | Space | Why |
|---------|------|-------|-----|
| Opposite-direction (Two Sum II) | O(n) after sort | O(1) | Each pointer moves ≤ n times total |
| Sort + two pointers | O(n log n) | O(1) | Sort dominates |
| 3Sum | O(n²) | O(1) extra | Outer O(n) × inner O(n) two-pointer |
| 4Sum | O(n³) | O(1) extra | Two outer × one inner two-pointer |
| Same-direction (remove dups) | O(n) | O(1) | Read scans each element once |
| Dutch national flag | O(n) | O(1) | mid pointer scans each element once |
| Fast-slow (cycle detect) | O(n) | O(1) | Fast travels at most 2n steps |
| Fast-slow (middle) | O(n) | O(1) | n/2 steps for slow |
| Trapping Rain Water | O(n) | O(1) | Each element processed once |
| Container With Most Water | O(n) | O(1) | Each element eliminated once |
| Two sorted arrays (merge) | O(n + m) | O(n + m) | Each element visited once |

### The amortized argument for all two-pointer solutions:
"Each pointer starts at one end and moves only INWARD. Total pointer movements = at most n for left + n for right = 2n = O(n)."

This is the key argument. Anytime someone questions why your two-pointer is O(n) despite loops, this is your answer.

### The 3Sum O(n²) argument:
"Outer loop runs n-2 times. For each fixed element, inner two-pointer runs O(n) because each pointer moves at most n times total (left only goes right, right only goes left). Total = O(n) × O(n) = O(n²)."

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Using two pointers on an UNSORTED array when hash map is needed

**What they do:** See a two-pointer problem from before and try to apply it to an unsorted array without thinking.

**What goes wrong:** The two-pointer invariant BREAKS because sorted order is the reason you can eliminate candidates directionally. Without sorted order, moving left right doesn't guarantee anything about the sum.

**The fix:** Ask yourself: "Is this array sorted, or can I sort it without losing information?" If NO → hash map (Pattern 01). If YES → two pointers.

```cpp
// VALID: Two Sum II (sorted) → two pointers, O(1) space
// INVALID: Two Sum I (unsorted) → don't sort and use two pointers
//          (sorting changes indices, but problem asks for indices!)
//          → Use hash map instead
```

---

### Mistake 2: Duplicate skipping logic in 3Sum is wrong

**What they do:** Either skip duplicates in the wrong place, or skip them in only one place instead of all three.

**What goes wrong:** Either misses valid triplets OR produces duplicate triplets.

**The fix:** Understand the three places duplicates must be skipped:
1. Outer loop: `if (i > 0 && nums[i] == nums[i-1]) continue;`
2. After finding a valid triplet: skip left duplicates AND right duplicates
3. The `i > 0` guard in (1) is essential — without it, you'd skip the FIRST occurrence

```cpp
// WRONG: skips too aggressively
for (int i = 0; i < n-2; i++) {
    if (nums[i] == nums[i+1]) continue; // BUG: skips last occurrence, keeps second-to-last
    
// CORRECT:
for (int i = 0; i < n-2; i++) {
    if (i > 0 && nums[i] == nums[i-1]) continue; // skip 2nd, 3rd, ... occurrences
```

---

### Mistake 3: `left <= right` vs `left < right` — off-by-one in termination

**What they do:** Use `left <= right` when they should use `left < right`, allowing the same element to be paired with itself.

**What goes wrong:** For pair problems: might pair element with itself. For write-pointer problems: might read one element too many.

**The fix:** Know the rules:
- Finding pairs from DIFFERENT positions: `left < right` (never use same index)
- Write pointer scanning: `read < n` (scan entire array)
- Fast-slow pointer: `fast != null && fast->next != null` (safe null check)
- Dutch national flag: `mid <= high` (process everything up to and including high)

```cpp
// WRONG: 
while (left <= right) { // allows left == right → pairs element with itself
    if (nums[left] + nums[right] == target) ...
    
// CORRECT:
while (left < right) { // guarantees distinct indices
```

---

### Mistake 4: Not erasing from map when frequency hits 0 (in hash map + two pointers combo)

**What they do:** In problems like "longest substring with at most K distinct characters," they decrement the frequency but don't erase from the map when it hits 0.

**What goes wrong:** `map.size()` returns the number of KEYS, not the number of keys with non-zero values. If you have `{a:0, b:2}`, size is 2 but distinct count is 1.

**The fix:** Always erase the key when its frequency reaches 0 in sliding window / two pointer + map combinations.

```cpp
freq[nums[left]]--;
if (freq[nums[left]] == 0) {
    freq.erase(nums[left]); // MUST erase to keep .size() accurate
}
left++;
```

---

### Mistake 5: Integer overflow in 4Sum

**What they do:** Compute sum of four numbers with `int` arithmetic.

**What goes wrong:** For nums[i] up to 10^9, four of them summed = 4 × 10^9 which overflows `int` (max ~2.1 × 10^9). Produces incorrect comparisons silently.

**The fix:** Use `long long` for any sum computation in 4Sum, or when n is large and values are large.

```cpp
// WRONG: overflow if nums[i] ≈ 10^9
if (nums[i] + nums[j] + nums[left] + nums[right] == target)

// CORRECT: cast to long long
if ((long long)nums[i] + nums[j] + nums[left] + nums[right] == (long long)target)
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (solve in under 15 minutes each)

**1. Valid Palindrome — LC 125 — [Opposite direction pointers]**

Why this problem: Simplest possible two-pointer application. Two pointers moving toward the center, comparing characters. The complication: you must skip non-alphanumeric characters.

What to prove: You can write the `while(left < right)` loop cleanly with two inner skips. You use `isalnum()` and `tolower()` correctly.

Target time: 5 minutes.

```
Edge cases: empty string (true), all spaces (true), single char (true), 
            "A man, a plan, a canal: Panama" (true), "race a car" (false)
```

**2. Two Sum II — LC 167 — [Classic left-right sorted array]**

Why this problem: The template problem for all sorted-array two-pointer work. Exactly one solution is guaranteed.

What to prove: You use `left < right` not `left <= right`. You know WHICH direction to move based on the sum comparison.

Target time: 5 minutes.

---

### CORE — 4 problems (the real learning happens here)

**3. 3Sum — LC 15 — [Three pointers, sorting, duplicate skipping]**

What this tests: This is the MOST important problem in this pattern. It combines sorting, two pointers, and duplicate handling. Every senior engineer knows 3Sum cold. You should too.

Key insight: Sort first. Fix one element. Two-pointer on the rest. Skip duplicates in all three places.

The thing that kills people: The duplicate skipping. Practice this until it's mechanical. The exact code for skipping is muscle memory.

Time limit: 40 minutes. If taking longer, look at your duplicate logic.

**4. Container With Most Water — LC 11 — [Move the shorter side]**

What this tests: Understanding the INVARIANT and being able to explain WHY the pointer moves. Not just getting the answer, but proving correctness.

Key insight: You always move the pointer with the SHORTER height. Because: moving the taller side when bounded by the shorter side can ONLY decrease area (width decreases, height stays bounded by shorter side).

What separates you from 1600 coders: being able to prove this in an interview.

**5. Sort Colors — LC 75 — [Dutch National Flag three-way partition]**

What this tests: Three-pointer partition. This exact algorithm appears in QuickSort. Understanding it deeply sets you up for partition-based problems.

Key insight: Three regions — left is 0s, right is 2s, middle (where mid sits) is 1s. The tricky part: when you swap mid with high, you DON'T advance mid because the swapped element is unknown.

Why `mid` doesn't advance after swap with `high`: The element that came from `high` to `mid` position was previously unexamined. It might be 0, 1, or 2 — we need to examine it next iteration.

**6. Remove Duplicates from Sorted Array II — LC 80 — [Write pointer, keep at most K]**

What this tests: The generalized write-pointer pattern. Understanding that "keep at most 2 duplicates" translates to "keep if write < 2 OR nums[read] != nums[write-2]."

Key insight: Comparing with `nums[write-2]` instead of `nums[write-1]`. This generalizes perfectly: for keep at most K, compare with `nums[write-K]`.

---

### STRETCH — 3 problems (where 2000 rating lives)

**7. Trapping Rain Water — LC 42 — [Two pointers with max tracking]**

What makes this hard: You need to see that you can compute water at each position using ONLY the smaller of the two max values. The O(1) space solution requires understanding why the side with the smaller max determines the water level.

Time limit: 45 minutes. This is a standard FAANG Hard — if you solve it cleanly with two pointers (not the O(n) space prefix array version), that's a strong signal.

Key insight: At position left: if maxLeft ≤ maxRight, water = maxLeft - height[left]. We KNOW this because the right side is at least maxLeft, so left is the bottleneck.

**8. 4Sum — LC 18 — [Extend 3Sum pattern + overflow handling]**

What makes this hard: Duplicates at two levels. Integer overflow. Early termination for efficiency.

Time limit: 40 minutes. This tests whether you can systematically extend a pattern vs needing to reinvent.

Key insight: Identical structure to 3Sum but with one more outer loop. The duplicate skipping logic is the same. The only new element is using `long long`.

**9. Boats to Save People — LC 881 — [Greedy + two pointers]**

What makes this hard: You need to see that this is TWO insights combined: (1) sort by weight, (2) greedily try to pair heaviest with lightest (if possible, otherwise heaviest goes alone).

Time limit: 30 minutes. The greedy proof (exchange argument) must be in your head.

Key insight: After sorting, always try to pair the current heaviest with the lightest. If they fit: great, both rescued in one boat. If not: heaviest goes alone (can't pair with ANYONE lighter since even the lightest doesn't fit).

---

### CONTEST LEVEL — 2 problems (2000+ territory)

**10. Shortest Unsorted Continuous Subarray — LC 581 — [Two directional scans to find boundaries]**

Why this is contest level: The optimal O(n) O(1) space solution requires two separate directional scans. Most people sort and compare (O(n log n)). The O(n) insight needs careful invariant thinking.

Key insight: Forward scan: find rightmost position where an element is SMALLER than the max seen so far (this element is definitely out of order → must be in the unsorted range). Backward scan: find leftmost position where an element is LARGER than the min seen from right (same logic from other side).

**11. Number of Subsequences That Satisfy Given Sum — LC 1498 — [Two pointer + modular arithmetic + math]**

Why this is contest level: Sort the array. For each left index, binary search or two-pointer to find the rightmost index where nums[left] + nums[right] ≤ target. All subsequences within [left, right] that include nums[left] are valid: 2^(right-left) of them (each of the middle elements can be included or excluded). But 2^(right-left) can overflow! Need modular exponentiation.

Key insight: After sorting, fix left = first element. Find rightmost right where sum ≤ target. Count = 2^(right - left). This requires precomputing powers of 2 mod 1e9+7.

```cpp
int numSubseq(vector<int>& nums, int target) {
    sort(nums.begin(), nums.end());
    int n = nums.size();
    const int MOD = 1e9 + 7;
    
    // Precompute powers of 2
    vector<long long> pw(n);
    pw[0] = 1;
    for (int i = 1; i < n; i++) pw[i] = pw[i-1] * 2 % MOD;
    
    int left = 0, right = n - 1;
    long long count = 0;
    
    while (left <= right) {
        if (nums[left] + nums[right] <= target) {
            count = (count + pw[right - left]) % MOD;
            left++;
        } else {
            right--;
        }
    }
    return count;
}
```

---

## PATTERN CONNECTIONS

### How Two Pointers connects to other patterns:

**Combines with Sorting (always):**
Almost every two-pointer problem on arrays starts with sorting. Sorting enables the "directional elimination" property. Without sorting, two pointers has no invariant. Time becomes O(n log n) dominated by sort.

**Combines with Binary Search (Pattern 05) when:**
Instead of linear scan with two pointers, one pointer stays fixed and the other is binary searched. Example: "For each element, find how many elements to its right are smaller" can use two pointers OR binary search depending on the structure.

**Combines with Sliding Window (Pattern 03) when:**
Sliding window IS a specific type of same-direction two-pointer (left shrinks, right expands). The difference: sliding window is used for SUBARRAY properties, two pointers for PAIR/TRIPLET MATCHING.

**Leads to Backtracking (Pattern 15) because:**
3Sum and 4Sum are small instances of k-sum, which generalizes to finding k elements with a property. For k > 4, this becomes backtracking territory.

**Often confused with Sliding Window:**
- Two Pointers (opposite direction): finds a PAIR satisfying a condition
- Sliding Window (same direction): finds a SUBARRAY satisfying a condition
- Two Pointers (same direction): modifies the array IN-PLACE

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given a sorted array of integers, find all pairs whose sum equals a given target. Return unique pairs."

**Company type:** Amazon, Microsoft, Indian unicorn (Zepto, CRED)
**Expected time:** 10-15 minutes
**What they are testing:** Basic two-pointer on sorted array + duplicate handling

**Red flags that get candidates rejected:**
- O(n²) brute force when sorted array was mentioned
- Using a hash map when two pointers achieves O(1) space
- Collecting duplicate pairs (not skipping properly)
- Not asking clarifying question: "What if there are duplicates in the array? Return all pairs or just unique values?"

**The correct solution + explanation:**
```cpp
vector<pair<int,int>> findPairs(vector<int>& nums, int target) {
    // Already sorted — use two pointers
    vector<pair<int,int>> result;
    int left = 0, right = (int)nums.size() - 1;
    
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            result.push_back({nums[left], nums[right]});
            // Skip all duplicates of both pointers
            while (left < right && nums[left] == nums[left+1]) left++;
            while (left < right && nums[right] == nums[right-1]) right--;
            left++; right--;
        } else if (sum < target) left++;
        else right--;
    }
    return result;
}
```

**Follow-up they will ask:** "What if the array is NOT sorted?" → Then we can't use two pointers without sorting. If we sort: O(n log n) time, O(1) space. If indices matter: use hash map O(n) time, O(n) space. The tradeoff depends on the constraint.

---

### Q2: "Given an array and a number k, find if there exist three elements a, b, c such that a + b + c = k."

**Company type:** FAANG (Amazon/Facebook)
**Expected time:** 20-25 minutes
**What they are testing:** 3Sum pattern recognition + sorted invariant + duplicate elimination

**Red flags:**
- O(n³) triple loop
- Not sorting first
- Incorrect duplicate skipping (producing duplicates OR missing triplets)
- Not handling edge case: array length < 3

**What they escalate to:** "Now return ALL unique triplets" (this is exactly LC 15). Then: "Now minimize the code to also handle negatives." The negatives are already handled if you coded 3Sum correctly with no extra sign checks — which is a good sign.

---

### Q3: "You have a large array with n ≤ 10^6 elements. Find all quadruplets that sum to zero. Memory is limited — cannot use O(n²) space."

**Company type:** Google, top trading firms
**Expected time:** 35 minutes
**What they are testing:** 4Sum, but more specifically: can you reason about time/space complexity tradeoffs? Can you handle the memory constraint?

**Red flags:**
- O(n^4) brute force
- O(n²) hash map approach (which uses O(n²) space — fails the constraint!)
- Not handling integer overflow with large values
- Not handling duplicates

**The constraint is the clue:** "Cannot use O(n²) space" rules out the hash map approach. This forces the O(n³) two-pointer approach which uses O(1) extra space.

**What they want to hear:**
"There are two main approaches: sort + two-pointers in O(n³) time O(1) extra space, OR hash map approach in O(n²) time O(n²) space. Given the memory constraint, I'll use the two-pointer approach. Let me walk you through sorting, then the double outer loop with inner two pointers..."

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory. If you cannot, you are NOT ready for the sign-off.**

1. **What is the invariant of the opposite-direction two-pointer approach?**
   → Everything to the LEFT of `left` has been eliminated (too small). Everything to the RIGHT of `right` has been eliminated (too large). If the answer exists, it's in [left, right].

2. **What are the two signals in a problem that tell you to use two pointers (not hash map)?**
   → (a) The array is sorted (or can be sorted without losing information like indices). (b) The problem asks for O(1) space or asks for pairs/triplets (not indices of pairs).

3. **What is the time complexity of 3Sum and WHY?**
   → O(n²). WHY: outer loop fixes one element in O(n) iterations. For each fixed element, inner two-pointer scan runs in O(n) (each of n elements is eliminated once). Total = O(n) × O(n) = O(n²).

4. **What is the most common mistake in 3Sum?**
   → Wrong or missing duplicate skipping. Either using `nums[i] == nums[i+1]` (wrong — compares with next, not previous) instead of `i > 0 && nums[i] == nums[i-1]`, or forgetting to skip duplicates after finding a valid triplet.

5. **In Dutch National Flag, why does `mid` NOT advance after swapping with `high`?**
   → Because the element that arrives at `mid` from `high` was previously unexamined. It could be 0, 1, or 2. We need to examine it in the next iteration before advancing `mid`.

6. **In Trapping Rain Water, why do we process the side with the SMALLER maximum?**
   → Because that side's water level is fully determined. If maxLeft < maxRight, then water at `left` = maxLeft - height[left] (right side is at least maxLeft, so it can hold that much water). We don't need to know the exact right-side max.

7. **When should you use fast-slow pointers instead of left-right pointers?**
   → When the data structure has cycles (linked list, abstract sequence like Happy Number), or when you need to find the middle of a linked list, or find the kth element from the end. Left-right is for arrays where you can index both ends. Fast-slow is for linked lists or cyclic structures.

---

## C++ STL CHEAT SHEET FOR THIS PATTERN

```cpp
// ═══════════════════════════════════════════
// Sorting (needed for almost all two-pointer problems)
// ═══════════════════════════════════════════
sort(nums.begin(), nums.end());               // ascending O(n log n)
sort(nums.begin(), nums.end(), greater<int>()); // descending
sort(v.begin(), v.end(), [](auto& a, auto& b){ // custom comparator
    return a.first < b.first;
});

// ═══════════════════════════════════════════
// Character checks (for palindrome problems)
// ═══════════════════════════════════════════
isalnum(c)   // true if alphanumeric
isalpha(c)   // true if letter
isdigit(c)   // true if digit
tolower(c)   // convert to lowercase
toupper(c)   // convert to uppercase

// ═══════════════════════════════════════════
// Integer safety
// ═══════════════════════════════════════════
(long long)a + b          // cast before addition
INT_MAX, INT_MIN, LLONG_MAX, LLONG_MIN
abs(x)                    // for int
llabs(x) or abs((long long)x) // for long long

// ═══════════════════════════════════════════
// Common two-pointer loop skeletons:
// ═══════════════════════════════════════════

// Opposite direction:
int left = 0, right = n - 1;
while (left < right) { ... }

// Same direction (write pointer):
int write = 0;
for (int read = 0; read < n; read++) {
    if (shouldKeep(nums[read])) nums[write++] = nums[read];
}

// Fixed outer + two-pointer inner (3Sum style):
sort(nums.begin(), nums.end());
for (int i = 0; i < n - 2; i++) {
    if (i > 0 && nums[i] == nums[i-1]) continue;
    int left = i+1, right = n-1;
    while (left < right) { ... }
}

// Fast-slow (cycle detect):
auto slow = head, fast = head;
while (fast && fast->next) {
    slow = slow->next;
    fast = fast->next->next;
    if (slow == fast) { /* cycle found */ break; }
}
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. **Solve all 11 problems in order.** Time yourself. Record times.
2. **For 3Sum (LC 15): code it three times.** Once for correctness, once from memory 1 day later, once 3 days later. This problem appears in interviews so often it must be automatic.
3. **For Trapping Rain Water (LC 42): implement BOTH approaches.** O(n) space first (easy), then O(1) space (two pointers). Only the second is interview-worthy.
4. **Revision schedule:** Redo problems 3, 7, 9 after 3 days, 7 days, and 14 days.
5. **Attempt the sign-off** once all 11 problems are done.

---

## THE SIGN-OFF CRITERIA

When you type `SIGN OFF P02`, I will:
1. Give you one unseen two-pointer problem
2. You identify the sub-variant (opposite-direction / same-direction / fast-slow / Dutch flag)
3. You write correct C++ solution
4. You state time/space complexity BEFORE I ask
5. You handle edge cases: empty array, single element, all same elements, array already sorted, negative numbers

---

*Now go solve. Start with LC 125 (Valid Palindrome) then LC 167 (Two Sum II). After warmups, go directly to 3Sum (LC 15) — that's where the real work is. When done, paste solutions with `SOLVE` and I'll review on all 5 dimensions.*
