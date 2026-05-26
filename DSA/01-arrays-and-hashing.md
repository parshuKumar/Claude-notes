# Pattern 01 — Arrays and Hashing
## Difficulty Level: FOUNDATION
## Interview Frequency: ALWAYS
## Estimated time to master: 3-4 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Arrays and Hashing is NOT a single pattern — it is a toolkit of 6+ sub-patterns that all revolve around one core idea: **trading space for time by using O(1) lookup**. Most 1600-rated coders think they "know" this because they can solve Two Sum. But when I ask them to solve "Subarray Sum Equals K" or "Longest Consecutive Sequence" under time pressure, they freeze. They don't have the instinct for WHEN to reach for a hash map and WHAT to store as the key vs the value. 

Mastery of this pattern means: you NEVER write a nested O(n²) loop to find something that a hash map can find in O(1). You NEVER miss the insight that "I can precompute this and store it." Every pattern after this one — sliding window, two pointers, graphs, DP — all use hash maps as auxiliary structures. If your hash map instincts are slow, everything else is slower too.

---

## ELI5 — THE CORE IDEA

Imagine you're a librarian with 10,000 books. Someone asks: "Do you have 'War and Peace'?"

**Bad librarian (O(n)):** Walks through every shelf, one by one, reading titles.
**Good librarian (O(1)):** Has a card catalog indexed by title → shelf location.

That card catalog IS a hash map. The title is the KEY. The shelf location is the VALUE.

Now the real question: **what should you index by?**

- If people ask "do you have this title?" → index by title
- If people ask "what books are by this author?" → index by author
- If people ask "what books are on this topic?" → index by topic

**The entire art of Arrays + Hashing is choosing the right KEY to index by.** The key determines what questions you can answer in O(1). Choose the wrong key and you're back to O(n) scanning. Choose the right key and the problem dissolves.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:
Any problem where you need to:
1. **Find something quickly** (existence check, complement lookup)
2. **Count things** (frequency of elements, number of occurrences)
3. **Group things** (by some shared property)
4. **Remember what you've seen** (detect duplicates, track state)
5. **Compute cumulative information** (prefix sums stored with hash maps)

### The invariant:
At any point during processing, the hash map contains exactly the information needed to answer queries about previously-seen elements in O(1) time.

### The decision framework:
```
I need to find X quickly?                    → unordered_set (existence)
I need to find X and know something about it? → unordered_map (key → info)
I need to count occurrences?                  → unordered_map<element, count>
I need to group by a property?               → unordered_map<property, vector<elements>>
I need cumulative info + quick lookup?       → unordered_map<prefix_state, index/count>
```

### The answer is extracted from:
Either the hash map itself (lookup result, count) or by using the hash map to eliminate work (skip elements already seen, find complements without scanning).

---

## WHEN TO RECOGNIZE THIS PATTERN

### Signals that SCREAM "use a hash map":

**Signal 1:** "Find two elements that satisfy a condition" + array is unsorted
→ Store one element, look up the complement. (Two Sum family)

**Signal 2:** "Count the number of subarrays with sum/property equal to K"
→ Prefix sum + hash map counting prefix states. (Subarray Sum = K family)

**Signal 3:** "Group elements by some property" (anagrams, same frequency, etc.)
→ Hash map with property as key, group as value.

**Signal 4:** "Find longest consecutive sequence" or "find if sequence exists"
→ Hash set for O(1) membership testing.

**Signal 5:** "Check if two things are equivalent" (anagrams, permutations)
→ Frequency maps and compare, or sort as key.

**Signal 6:** Constraints say n ≤ 10^5 and the brute force is O(n²)
→ Almost always means there's a hash map solution in O(n).

### Signals that LOOK LIKE this pattern but are NOT:

**Fake signal 1:** "Find two elements in a SORTED array" → Use two pointers instead (Pattern 02). Hash map works but is overkill and uses extra space.

**Fake signal 2:** "Find subarray of fixed length with maximum sum" → Sliding window (Pattern 03). No hash map needed.

**Fake signal 3:** "Find elements that satisfy a SORTED ORDER property" → Probably needs sorting + binary search or two pointers, not just hashing.

---

## THE TEMPLATES (C++)

### Template 1: Frequency Counting

```cpp
// Frequency Count Template
// Time: O(n) — single pass
// Space: O(k) — where k is number of distinct elements

#include <bits/stdc++.h>
using namespace std;

// Count frequency of every element
unordered_map<int, int> buildFrequencyMap(vector<int>& nums) {
    unordered_map<int, int> freq;
    for (int num : nums) {
        freq[num]++;
    }
    return freq;
}

// HOW TO ADAPT:
// 1. Change the key type for what you're counting (char, string, int)
// 2. After building freq map, iterate it to find answer
//    - Most frequent: track max count
//    - K most frequent: use heap or bucket sort
//    - Elements with count == K: filter the map
```

### Template 2: Complement Lookup (Two Sum Pattern)

```cpp
// Complement Lookup Template
// Time: O(n) — single pass
// Space: O(n) — store seen elements

vector<int> twoSumPattern(vector<int>& nums, int target) {
    // Key insight: for each element x, we need (target - x) to exist
    // Store elements we've seen, check if complement exists
    
    unordered_map<int, int> seen; // value → index
    
    for (int i = 0; i < nums.size(); i++) {
        int complement = target - nums[i];
        
        if (seen.count(complement)) {
            return {seen[complement], i};
        }
        
        seen[nums[i]] = i; // store AFTER checking (avoid using same element twice)
    }
    
    return {}; // no solution found
}

// HOW TO ADAPT:
// 1. Change `complement` calculation for different conditions
//    - Product: complement = target / nums[i] (check divisibility)
//    - XOR: complement = target ^ nums[i]
//    - Difference: complement = nums[i] - target OR target + nums[i]
// 2. Change what you store as value (index, count, boolean)
// 3. For "count pairs": instead of returning, count += seen[complement]
```

### Template 3: Grouping by Key

```cpp
// Group By Key Template
// Time: O(n * k) — where k is cost of computing the key
// Space: O(n) — store all elements in groups

vector<vector<string>> groupByKey(vector<string>& strs) {
    // Key insight: elements with the same PROPERTY go in same group
    // Choose the property as the hash map key
    
    unordered_map<string, vector<string>> groups;
    
    for (string& s : strs) {
        // Compute the key — this is the CREATIVE PART
        string key = computeKey(s); // e.g., sort the string for anagrams
        groups[key].push_back(s);
    }
    
    // Extract groups into result
    vector<vector<string>> result;
    for (auto& [key, group] : groups) {
        result.push_back(group);
    }
    return result;
}

// HOW TO ADAPT:
// 1. Change key computation:
//    - Anagrams: sort the string → sorted string is key
//    - Same frequency: frequency vector → serialize as string key
//    - Same remainder: num % k → remainder is key
// 2. Change value type: vector<int>, vector<string>, count, etc.
```

### Template 4: Prefix Sum + Hash Map (CRITICAL — this is where 1600 coders fail)

```cpp
// Prefix Sum + Hash Map Template
// Time: O(n) — single pass
// Space: O(n) — store prefix sums

int subarraySumEqualsK(vector<int>& nums, int k) {
    // Key insight: if prefix[j] - prefix[i] == k, then subarray (i,j] sums to k
    // So for each j, we need: does prefix[j] - k exist as a previous prefix sum?
    // This is literally Two Sum but with prefix sums!
    
    unordered_map<int, int> prefixCount; // prefix_sum → how many times we've seen it
    prefixCount[0] = 1; // empty prefix (before index 0) has sum 0
    
    int currentSum = 0;
    int count = 0;
    
    for (int num : nums) {
        currentSum += num;
        
        // How many previous prefix sums equal (currentSum - k)?
        // Each one represents a valid subarray ending here
        int complement = currentSum - k;
        if (prefixCount.count(complement)) {
            count += prefixCount[complement];
        }
        
        // Record current prefix sum
        prefixCount[currentSum]++;
    }
    
    return count;
}

// HOW TO ADAPT:
// 1. For "subarray sum divisible by K": 
//    key = ((currentSum % k) + k) % k  (handle negative mod)
// 2. For "subarray with equal 0s and 1s": 
//    treat 0 as -1, then find subarray sum = 0
// 3. For "longest subarray with sum K":
//    store first index of each prefix sum, answer = max(i - prefixCount[complement])
// 4. For "number of subarrays with XOR = K":
//    use prefix XOR instead of prefix sum
```

### Template 5: Hash Set for Intelligent Start / Existence

```cpp
// Hash Set for O(1) Membership Template
// Time: O(n) — despite nested-looking code
// Space: O(n) — store all elements

int longestConsecutiveSequence(vector<int>& nums) {
    // Key insight: don't start counting from every element
    // Only start from elements that are the BEGINNING of a sequence
    // An element x is the beginning if (x-1) does NOT exist in the set
    
    unordered_set<int> numSet(nums.begin(), nums.end());
    int longest = 0;
    
    for (int num : numSet) {
        // Only start counting if this is the START of a sequence
        if (numSet.find(num - 1) == numSet.end()) {
            int currentNum = num;
            int currentLength = 1;
            
            while (numSet.find(currentNum + 1) != numSet.end()) {
                currentNum++;
                currentLength++;
            }
            
            longest = max(longest, currentLength);
        }
    }
    
    return longest;
}

// WHY THIS IS O(n) AND NOT O(n²):
// Each element is visited at most twice:
// - Once when checking if it's a sequence start
// - Once when extending from some start
// Total work = 2n = O(n)

// HOW TO ADAPT:
// 1. Change the "start condition" for what you're sequencing
// 2. Change the "extend condition" for what counts as consecutive
// 3. For "longest consecutive in a stream": use DSU instead (Pattern 13)
```

### Template 6: Encoding State as Key

```cpp
// State Encoding Template
// Time: O(n * k) — where k is cost of encoding
// Space: O(n) — store states

// Example: Encode character frequency as a key
string encodeFrequency(const string& s) {
    // For anagram grouping: two strings are anagrams if they have same char frequency
    // Encode frequency as string key: "a2b1c3" or "2#1#0#3#0#..."
    
    vector<int> count(26, 0);
    for (char c : s) count[c - 'a']++;
    
    string key;
    for (int i = 0; i < 26; i++) {
        key += to_string(count[i]) + '#';
    }
    return key;
}

// Alternative encoding: sort the string (simpler but O(k log k))
string encodeSorted(const string& s) {
    string sorted_s = s;
    sort(sorted_s.begin(), sorted_s.end());
    return sorted_s;
}

// HOW TO ADAPT:
// 1. For matrix states: serialize row by row
// 2. For graph states: serialize adjacency
// 3. For game states: serialize board configuration
// 4. For sliding window states: serialize character counts
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

---

### VARIANT 1: Frequency Counting

**What it is:** Count how many times each element appears. Use the counts to answer questions.

**When you need it:**
- "Find the most frequent element"
- "Find elements that appear more than n/3 times"
- "Find the top K frequent elements"
- "Check if two strings are anagrams"

**The core code change from base template:**
```cpp
// Basic frequency counting
unordered_map<int, int> freq;
for (int x : nums) freq[x]++;

// Then answer questions:
// Most frequent: iterate freq, track max
// Top K: push all into min-heap of size K, or bucket sort
// Anagram check: build freq for both strings, compare
// Majority element: Moore's voting (not even hash map needed)
```

**Bucket Sort optimization for Top K:**
```cpp
// O(n) approach for "Top K Frequent Elements"
// Instead of sorting or using heap, use BUCKET SORT by frequency
vector<vector<int>> buckets(nums.size() + 1); // bucket[i] = elements with frequency i
for (auto& [num, count] : freq) {
    buckets[count].push_back(num);
}
// Iterate buckets from high to low, collect K elements
vector<int> result;
for (int i = buckets.size() - 1; i >= 0 && result.size() < k; i--) {
    for (int num : buckets[i]) {
        result.push_back(num);
        if (result.size() == k) break;
    }
}
```

**Interview trap:** Interviewers love asking "what if there are ties?" Make sure your solution handles ties correctly for Top K problems.

---

### VARIANT 2: Complement Lookup (Two Sum Family)

**What it is:** For each element, compute what OTHER element you need, and check if it exists.

**The family:**
- **Two Sum** (LC 1): target - nums[i]
- **Two Sum with multiple pairs**: count all pairs
- **3Sum** (LC 15): fix one, two-sum the rest (but use two pointers since sorted)
- **4Sum** (LC 18): fix two, two-sum the rest
- **Pair with given difference**: nums[i] + k or nums[i] - k
- **Subarray sum** = prefix sum complement (see Template 4)

**Critical distinction:**
```cpp
// If you need to find ONE pair: use seen map, return immediately
// If you need to COUNT pairs: count += seen[complement] (accumulate before storing)
// If you need ALL pairs: store list of indices, handle duplicates

// COUNT all pairs with sum = target:
int countPairs(vector<int>& nums, int target) {
    unordered_map<int, int> seen;
    int count = 0;
    for (int num : nums) {
        count += seen[target - num]; // count BEFORE adding current
        seen[num]++;
    }
    return count;
}
```

**Why "store AFTER checking" matters:**
```cpp
// WRONG: store first, then check → might pair element with itself
seen[nums[i]] = i;
if (seen.count(complement)) // BUG: might find nums[i] itself

// CORRECT: check first, then store → only finds previously seen elements
if (seen.count(complement)) return {seen[complement], i};
seen[nums[i]] = i;
```

---

### VARIANT 3: Grouping by Key

**What it is:** Elements that share a property belong to the same group. Hash map key = property, value = group.

**Key generation strategies:**

```cpp
// Strategy 1: Sort as key (for anagrams)
// "eat" → "aet", "tea" → "aet", "ate" → "aet" → same group
string key = s; sort(key.begin(), key.end());

// Strategy 2: Frequency encoding as key (faster for long strings)
// "eat" → "1#0#0#0#1#...#1#..." (count of each char)
string key; 
vector<int> cnt(26,0);
for(char c:s) cnt[c-'a']++;
for(int i=0;i<26;i++) key += to_string(cnt[i]) + '#';

// Strategy 3: Modular key (for grouping by remainder)
// Group numbers by num % k
int key = ((num % k) + k) % k; // handle negatives

// Strategy 4: Difference pattern as key
// For shifted strings: "abc" and "bcd" have same diff pattern [1,1]
string key;
for(int i=1;i<s.size();i++) key += to_string((s[i]-s[i-1]+26)%26) + '#';

// Strategy 5: Prime product as key (risky due to overflow but clever)
// Map each char to a prime, multiply all → anagrams have same product
```

---

### VARIANT 4: Prefix Sum + Hash Map

**What it is:** Convert "subarray property" into "difference of prefix values." Then it becomes a complement lookup problem on prefix values.

**This is THE most important sub-pattern for contests.** It transforms O(n²) subarray problems into O(n).

**The general principle:**
```
subarray[i..j] has property P
↕ (transform)
prefix[j] - prefix[i-1] satisfies some condition related to P
↕ (rearrange)
prefix[j] - K = prefix[i-1]  (complement lookup!)
```

**Key variants:**

```cpp
// Variant A: Count subarrays with sum = K (LC 560)
// prefix[j] - prefix[i] = K → look for prefix[j] - K in map
// Already shown in Template 4 above

// Variant B: Count subarrays with sum divisible by K (LC 974)
// (prefix[j] - prefix[i]) % K = 0 → prefix[j] % K = prefix[i] % K
// So group by (prefix % K), count pairs within each group
int subarraysDivByK(vector<int>& nums, int k) {
    unordered_map<int, int> remainderCount;
    remainderCount[0] = 1;
    int sum = 0, count = 0;
    for (int num : nums) {
        sum += num;
        int rem = ((sum % k) + k) % k; // handle negative numbers!
        count += remainderCount[rem];
        remainderCount[rem]++;
    }
    return count;
}

// Variant C: Longest subarray with sum = K
// Instead of counting, store FIRST occurrence of each prefix sum
// Answer = max(j - firstSeen[prefix[j] - K])
int longestSubarraySumK(vector<int>& nums, int k) {
    unordered_map<int, int> firstSeen; // prefix_sum → first index
    firstSeen[0] = -1; // empty prefix at virtual index -1
    int sum = 0, maxLen = 0;
    for (int i = 0; i < nums.size(); i++) {
        sum += nums[i];
        if (firstSeen.count(sum - k)) {
            maxLen = max(maxLen, i - firstSeen[sum - k]);
        }
        if (!firstSeen.count(sum)) { // only store FIRST occurrence
            firstSeen[sum] = i;
        }
    }
    return maxLen;
}

// Variant D: Longest subarray with equal 0s and 1s (LC 525)
// Transform: treat 0 as -1 → find longest subarray with sum = 0
// Same as Variant C with K = 0

// Variant E: Subarray XOR = K
// prefix_xor[j] ^ prefix_xor[i] = K → prefix_xor[j] ^ K = prefix_xor[i]
int countSubarraysXOR(vector<int>& nums, int k) {
    unordered_map<int, int> xorCount;
    xorCount[0] = 1;
    int xorSum = 0, count = 0;
    for (int num : nums) {
        xorSum ^= num;
        count += xorCount[xorSum ^ k];
        xorCount[xorSum]++;
    }
    return count;
}
```

**WHY prefixCount[0] = 1:**
This represents the empty prefix (before index 0). If the entire subarray from index 0 to current index has sum = K, then currentSum - K = 0, and we need to count that case. Without this initialization, we'd miss subarrays starting at index 0.

---

### VARIANT 5: Index Mapping

**What it is:** Store the index of elements for quick position-related queries.

```cpp
// Store first and last occurrence of each element
unordered_map<int, int> firstIndex, lastIndex;
for (int i = 0; i < nums.size(); i++) {
    if (!firstIndex.count(nums[i])) firstIndex[nums[i]] = i;
    lastIndex[nums[i]] = i;
}

// Store all indices of each element
unordered_map<int, vector<int>> indices;
for (int i = 0; i < nums.size(); i++) {
    indices[nums[i]].push_back(i);
}
// Now can binary search within indices[x] for nearest occurrence
```

**Use cases:**
- "Shortest distance between two occurrences of same element"
- "Find the maximum distance between equal elements"
- "For each element, find next occurrence of X" (precompute)

---

### VARIANT 6: Cyclic Sort / Index as Hash

**What it is:** When elements are in range [1, n] or [0, n-1], the array itself is the hash map. Place element x at index x-1. Whatever's missing tells you the answer.

```cpp
// First Missing Positive (LC 41) — O(1) space hash map!
int firstMissingPositive(vector<int>& nums) {
    int n = nums.size();
    
    // Place each number at its correct index: nums[i] should be i+1
    for (int i = 0; i < n; i++) {
        while (nums[i] > 0 && nums[i] <= n && nums[nums[i] - 1] != nums[i]) {
            swap(nums[i], nums[nums[i] - 1]);
        }
    }
    
    // First place where nums[i] != i+1 is the answer
    for (int i = 0; i < n; i++) {
        if (nums[i] != i + 1) return i + 1;
    }
    return n + 1;
}

// WHY this is O(n): each swap places one element in correct position
// Each element is swapped at most once → total swaps ≤ n
```

**When to use cyclic sort vs hash map:**
- Elements in range [1,n] → cyclic sort (O(1) space)
- Elements NOT in bounded range → hash map (O(n) space)

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

### You must state these without hesitation:

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Single lookup in unordered_map | O(1) average | — | Hash function + direct access |
| Insert in unordered_map | O(1) average | — | Hash + insert |
| Build frequency map of n elements | O(n) | O(k) where k = distinct elements | One pass, one insert per element |
| Two Sum (complement lookup) | O(n) | O(n) | One pass, store each element |
| Group Anagrams | O(n * k log k) | O(n * k) | n strings, sorting each of length k |
| Group Anagrams (frequency key) | O(n * k) | O(n * k) | n strings, counting chars in each |
| Subarray Sum = K (prefix + map) | O(n) | O(n) | One pass, store prefix sums |
| Longest Consecutive Sequence | O(n) | O(n) | Build set O(n), each element visited ≤ 2 times |
| First Missing Positive (cyclic) | O(n) | O(1) | Each element swapped at most once |
| Top K Frequent (bucket sort) | O(n) | O(n) | Build freq O(n), bucket O(n) |
| Top K Frequent (heap) | O(n log k) | O(n + k) | Build freq O(n), heap of size k |

### WHY unordered_map is O(1) average but O(n) worst case:
- Average: hash function distributes keys evenly → each bucket has O(1) elements
- Worst case: all keys hash to same bucket → linked list → O(n) lookup
- In practice: for interview and CP purposes, ALWAYS assume O(1)
- If worst case matters: use custom hash or switch to sorted map O(log n)

### The amortized argument for Longest Consecutive Sequence:
"But there's a while loop inside a for loop — isn't that O(n²)?"
NO. The while loop only triggers for sequence STARTS (elements without predecessor). Each element is the inner part of at most ONE sequence. Total elements visited by all while loops combined = n. So total work = O(n).

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Forgetting prefixCount[0] = 1

**What they do:** Start prefix sum counting without initializing the empty prefix.

**What goes wrong:** Miss all subarrays that start at index 0 and have sum = K. Get wrong answer (usually off by 1 to several).

**The fix:** ALWAYS write `prefixCount[0] = 1` (or `firstSeen[0] = -1` for longest variant) BEFORE the loop. Make this line muscle memory. You write it without thinking.

```cpp
// WRONG
unordered_map<int, int> prefixCount;
int sum = 0, count = 0;
for (int num : nums) { ... }

// CORRECT
unordered_map<int, int> prefixCount;
prefixCount[0] = 1; // ← THIS LINE. ALWAYS.
int sum = 0, count = 0;
for (int num : nums) { ... }
```

---

### Mistake 2: Using map instead of unordered_map

**What they do:** Write `map<int, int>` out of habit instead of `unordered_map<int, int>`.

**What goes wrong:** O(log n) per operation instead of O(1). For n = 10^5, this adds a factor of 17x. TLE on tight time limits.

**The fix:** Default to `unordered_map` unless you need sorted order. If you need sorted order, you probably need a different approach anyway.

```cpp
// SLOW (but correct): O(n log n) total
map<int, int> freq;

// FAST: O(n) total  
unordered_map<int, int> freq;

// WHEN you actually need map:
// - Need to iterate keys in sorted order
// - Need lower_bound / upper_bound on keys
// - Working with custom objects without hash function
```

---

### Mistake 3: Negative modulo in "divisible by K" problems

**What they do:** Use `sum % k` directly without handling negative numbers.

**What goes wrong:** In C++, `-7 % 3 = -1` (NOT 2). This means negative prefix sums produce negative remainders that don't match positive remainders. Wrong grouping → wrong answer.

**The fix:** Always use `((sum % k) + k) % k` for modular arithmetic with potentially negative values.

```cpp
// WRONG: produces negative remainders
int rem = sum % k;

// CORRECT: always produces non-negative remainder in [0, k-1]
int rem = ((sum % k) + k) % k;
```

---

### Mistake 4: Not handling the "same element" edge case in Two Sum

**What they do:** Store element first, then look for complement. Or use the element at the same index as its own complement.

**What goes wrong:** `[3, 3]` with target 6 → might return same index twice, or might miss the pair entirely.

**The fix:** For Two Sum (find ONE pair): look up complement BEFORE storing current. For "count pairs with same value": handle frequency counting correctly.

```cpp
// WRONG ORDER:
seen[nums[i]] = i;              // store first
if (seen.count(target - nums[i])) // then look up → might find itself!

// CORRECT ORDER:
if (seen.count(target - nums[i])) return result; // look up first
seen[nums[i]] = i;              // store after
```

---

### Mistake 5: Not choosing the right key for grouping

**What they do:** Use a key that doesn't uniquely identify the group, or use an expensive key (like sorted string O(k log k)) when a cheaper one exists (frequency encoding O(k)).

**What goes wrong:** Either wrong grouping (wrong key) or TLE (expensive key computation).

**The fix:** Always ask: "What PROPERTY makes two elements belong to the same group?" That property IS your key. Then ask: "What's the cheapest way to compute this property?"

```cpp
// Anagrams — three valid keys:
// 1. Sorted string: "eat" → "aet" (O(k log k) per string)
// 2. Frequency array as string: "1#0#0#0#1#..." (O(k) per string) ← FASTER
// 3. Prime product: 2*5*...*20 (O(k) but overflow risk)

// For shifted strings ("abc", "bcd", "xyz"):
// Key = difference pattern: (b-a, c-b) = (1,1) for all three
// WRONG key: the string itself (each string is different!)
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (solve in under 15 minutes each)

These prove you understand the base templates. If these take more than 15 minutes, re-read the templates above.

**1. Two Sum — LC 1 — [Complement Lookup]**

Why this problem: The foundation of all hash map thinking. You should solve this in 3 minutes with your eyes closed. If not, your hash map instinct is not fast enough.

What to prove: You can do single-pass complement lookup. You know to check BEFORE storing.

Target time: 5 minutes.

**2. Contains Duplicate — LC 217 — [Hash Set Existence]**

Why this problem: The simplest hash set application. Insert into set, check if already exists.

What to prove: You know `unordered_set` and its O(1) insert/find.

Target time: 3 minutes.

---

### CORE — 4 problems (the real learning happens here)

These require adapting the templates. Expect 20-40 minutes each. Do NOT look at solutions until you have spent 40 minutes trying.

**3. Valid Anagram — LC 242 — [Frequency Counting]**

What variant this tests: Build frequency map, compare two frequency maps. Or better: build one map incrementing for s, decrementing for t, check all zeros.

Key insight to discover: You don't need TWO maps. One map with +1/-1 is enough.

```cpp
// The elegant one-map approach:
bool isAnagram(string s, string t) {
    if (s.size() != t.size()) return false;
    int count[26] = {};
    for (int i = 0; i < s.size(); i++) {
        count[s[i]-'a']++;
        count[t[i]-'a']--;
    }
    for (int c : count) if (c != 0) return false;
    return true;
}
```

**4. Group Anagrams — LC 49 — [Grouping by Key]**

What variant this tests: Choose a key that represents "same anagram group." Group all strings by that key.

Key insight to discover: The key can be the sorted string OR the frequency encoding. Frequency encoding is O(k) vs sorted is O(k log k).

Watch for: Using `unordered_map<string, vector<string>>` — you need a string key for the hash map.

**5. Top K Frequent Elements — LC 347 — [Frequency + Bucket Sort]**

What variant this tests: Build frequency map, then extract top K. Three approaches: sort (O(n log n)), heap (O(n log k)), bucket sort (O(n)).

Key insight to discover: Bucket sort by frequency. Maximum possible frequency = n. Create n+1 buckets. O(n) total.

What separates 1600 from 2000: The 1600 coder uses a heap. The 2000 coder uses bucket sort because they recognize the frequency is bounded by n.

**6. Product of Array Except Self — LC 238 — [Prefix/Suffix Product]**

What variant this tests: Not a hash map problem per se, but tests the "precompute prefix information" mindset that is central to this pattern.

Key insight to discover: prefix[i] = product of all elements before i. suffix[i] = product of all elements after i. Answer[i] = prefix[i] * suffix[i]. Can be done in O(1) extra space with two passes.

Constraint: Cannot use division. This forces the prefix/suffix approach.

---

### STRETCH — 3 problems (this is where 2000 rating lives)

These require insight beyond the template. Expect 30-60 minutes. If stuck after 45 minutes, look at hints only — not full solution.

**7. Longest Consecutive Sequence — LC 128 — [Hash Set + Intelligent Start]**

What makes this hard: The naive approach is O(n log n) with sorting. Getting O(n) requires the "only start counting from sequence beginnings" insight.

Key insight: An element x is the START of a sequence if and only if (x-1) is NOT in the set. Only start the while-loop from starts. This guarantees O(n) total.

Common mistake: Starting the count from every element → O(n²) in worst case (e.g., [1,2,3,...,n]).

**8. Subarray Sum Equals K — LC 560 — [Prefix Sum + Hash Map]**

What makes this hard: You cannot use sliding window here because elements can be negative. The prefix sum + hash map approach is the only O(n) solution.

Key insight: This is literally Two Sum on prefix sums. "Does (currentPrefix - K) exist as a previous prefix sum?"

What the 1600 coder gets wrong: Tries sliding window (fails with negatives). Or forgets `prefixCount[0] = 1`.

**9. Encode and Decode Strings — LC 271 — [State Encoding Design]**

What makes this hard: This is a design problem. You need to encode a list of strings into a single string and decode it back. The tricky part: strings can contain ANY character including your delimiter.

Key insight: Use length-prefixed encoding. Each string is stored as `[length]#[string]`. Since length is a number followed by #, the decoder knows exactly how many characters to read.

```cpp
// Encode: "hello", "world" → "5#hello5#world"
string encode(vector<string>& strs) {
    string result;
    for (const string& s : strs) {
        result += to_string(s.size()) + '#' + s;
    }
    return result;
}
// Decode: read number until '#', read that many chars, repeat
```

---

### CONTEST LEVEL — 2 problems (2000+ territory)

These are hard problems. If you cannot solve them after studying this pattern, come back after Pattern 06 (Prefix Sums).

**10. First Missing Positive — LC 41 — [Cyclic Sort / Index as Hash]**

Why this is contest level: Must achieve O(n) time AND O(1) space. The insight that "the array itself is the hash map" is non-obvious. You use the array indices as hash keys.

The aha moment: The answer must be in range [1, n+1]. Place each element at its "correct" index (element x goes to index x-1). Then scan for first mismatch.

**11. Minimum Window Substring — LC 76 — [Frequency Matching + Sliding Window]**

Why this is contest level: This is actually a sliding window problem (Pattern 03) but the hash map frequency matching component makes it hard. You need two frequency maps and a "matched characters" counter.

Note: This bridges Pattern 01 and Pattern 03. If you find this too hard now, revisit after Pattern 03.

---

## PATTERN CONNECTIONS

### How Arrays + Hashing connects to other patterns:

**Combines with Sliding Window (Pattern 03) when:**
The sliding window needs to track character frequencies or check validity. The hash map is the "state tracker" inside the window. Example: Minimum Window Substring, Longest Substring Without Repeating Characters.

**Combines with Prefix Sums (Pattern 06) when:**
You need to answer subarray queries efficiently. Prefix sum gives you the cumulative data; hash map gives you O(1) lookup of previous cumulative states. Example: Subarray Sum Equals K.

**Combines with Two Pointers (Pattern 02) when:**
The two-pointer approach needs O(1) lookups to check conditions. Example: 3Sum can use hash set for the third element (though two pointers is better after sorting).

**Leads to Union Find (Pattern 13) because:**
Union Find is essentially a "dynamic grouping" structure. Hash maps do static grouping; Union Find does grouping that changes as you process.

**Often confused with:**
- **Sorting + Binary Search:** When you need to find elements quickly, sometimes sorting + binary search (O(n log n) time, O(1) space) is better than hashing (O(n) time, O(n) space). Know the tradeoff.
- **Two Pointers on sorted array:** If the array is already sorted, two pointers gives O(1) space. Don't waste space on a hash map.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given an array of integers and a target sum, find all unique pairs that sum to the target."

**Company type:** Indian unicorn (Swiggy, Razorpay, Meesho)
**Expected time:** 15 minutes
**What they are testing:** Hash map complement lookup + duplicate handling

**The trap:** "Unique pairs" means [1,5] and [5,1] are the same. Also, if array is [3,3,3] and target is 6, the answer is just one pair [3,3], not three pairs.

**Red flags that get candidates rejected:**
- Using O(n²) brute force
- Not handling duplicates (reporting same pair multiple times)
- Not clarifying: "unique" means unique VALUES or unique INDICES?
- Using `set<pair<int,int>>` as a bandaid instead of handling it correctly

**The correct approach:**
```cpp
vector<pair<int,int>> findPairs(vector<int>& nums, int target) {
    sort(nums.begin(), nums.end()); // Sort to easily skip duplicates
    vector<pair<int,int>> result;
    int left = 0, right = nums.size() - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            result.push_back({nums[left], nums[right]});
            while (left < right && nums[left] == nums[left+1]) left++; // skip dupes
            while (left < right && nums[right] == nums[right-1]) right--;
            left++; right--;
        } else if (sum < target) left++;
        else right--;
    }
    return result;
}
```

Note: This actually uses Two Pointers (Pattern 02) because sorting makes it cleaner. A hash map approach works too but duplicate handling is messier.

---

### Q2: "Given a stream of integers, design a data structure that supports: (a) add a number, (b) check if there exists a pair summing to a target."

**Company type:** FAANG (Google/Meta)
**Expected time:** 20 minutes
**What they are testing:** Hash map design + amortized analysis + tradeoff discussion

**The trap:** "Stream" means you add elements one by one. You need to support both operations efficiently. The follow-up will be: "What if check is called much more often than add?"

**Red flags:**
- Not discussing the time/space tradeoff
- Not handling the case where target/2 exists but only appears once
- Not discussing amortized complexity

**The correct approach:**
```cpp
class TwoSumDesign {
    unordered_map<int, int> freq; // number → count
public:
    void add(int number) {
        freq[number]++;
    }
    
    bool find(int target) {
        for (auto& [num, count] : freq) {
            int complement = target - num;
            if (complement == num) {
                if (count > 1) return true; // need two of the same
            } else {
                if (freq.count(complement)) return true;
            }
        }
        return false;
    }
};
// add: O(1), find: O(n) — good when add is frequent, find is rare
```

**Follow-up discussion:** If find() is called more often, precompute all possible sums in a hash set during add(). add becomes O(n), find becomes O(1).

---

### Q3: "Given an array, find the length of the longest subarray with at most K distinct elements."

**Company type:** FAANG (Amazon/Microsoft)
**Expected time:** 20 minutes
**What they are testing:** Hash map as frequency tracker inside a sliding window

**The trap:** This is actually a Sliding Window problem (Pattern 03), but the hash map component is what makes it work. They want to see that you naturally reach for a hash map to track the window state.

**Red flags:**
- Not using a hash map to track distinct elements in window
- Shrinking the window incorrectly (not decrementing frequency / removing from map)
- Off-by-one errors in window boundaries

**The correct approach:**
```cpp
int longestWithKDistinct(vector<int>& nums, int k) {
    unordered_map<int, int> windowFreq;
    int left = 0, maxLen = 0;
    
    for (int right = 0; right < nums.size(); right++) {
        windowFreq[nums[right]]++;
        
        while (windowFreq.size() > k) {
            windowFreq[nums[left]]--;
            if (windowFreq[nums[left]] == 0) {
                windowFreq.erase(nums[left]); // MUST erase to reduce distinct count
            }
            left++;
        }
        
        maxLen = max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

**Critical detail:** You MUST erase from the map when frequency hits 0. Otherwise `.size()` doesn't reflect actual distinct count. This is the #1 bug candidates write.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory. If you cannot, you are NOT ready for the sign-off.**

1. **What is the invariant of the hash map approach?**
   → At any point, the hash map contains exactly the information about previously-processed elements needed to answer queries in O(1).

2. **What are the two signals in a problem that tell you to use a hash map?**
   → (a) You need to find a complement/match for each element quickly (O(1) lookup needed). (b) The constraint says n ≤ 10^5 and brute force is O(n²) — suggests O(n) with hash map.

3. **What is the time complexity of the prefix sum + hash map approach, and WHY?**
   → O(n) time, O(n) space. WHY: single pass through array, each element does O(1) hash map operations (insert + lookup). Total = n * O(1) = O(n).

4. **What is the most common mistake at this pattern?**
   → Forgetting `prefixCount[0] = 1` initialization in prefix sum + hash map problems. This causes missing all valid subarrays that start from index 0.

5. **What variant is hardest and what changes in the template?**
   → Prefix sum + hash map with modular arithmetic (divisible by K). The change: key becomes `((sum % k) + k) % k` instead of `sum`. The +k handles negative modulo in C++.

6. **How does this pattern combine with Sliding Window (Pattern 03)?**
   → The hash map serves as the "window state tracker." It stores character frequencies or element counts within the current window. When the window shrinks, you decrement/erase from the map.

7. **When should you NOT use a hash map and use sorting + two pointers instead?**
   → When the array is already sorted, or when the problem requires pairs/triplets in sorted order, or when you need O(1) space. Two pointers on sorted array = O(1) space. Hash map = O(n) space. If space matters, sort first.

---

## ADVANCED TECHNIQUES FOR CONTEST USE

---

### Technique 1: Custom Hash to Avoid unordered_map TLE

In competitive programming, `unordered_map` can be hacked to produce O(n²) worst case via collision attacks. Use a custom hash:

```cpp
struct custom_hash {
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t FIXED_RANDOM = chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + FIXED_RANDOM);
    }
};

unordered_map<int, int, custom_hash> safe_map;
```

When to use: Codeforces contests where you suspect hack cases. LeetCode doesn't have this issue.

---

### Technique 2: Using array instead of hash map when keys are bounded

```cpp
// If keys are in range [0, 10^6], use array instead of hash map
// Array access = guaranteed O(1), no hash collision, better cache performance

int freq[1000001] = {}; // MUCH faster than unordered_map<int,int>
for (int x : nums) freq[x]++;

// When to use:
// - Character frequency: int freq[26] = {} or int freq[128] = {}
// - Small integer range: int freq[MAX_VAL+1] = {}
// - Coordinate compression + array (when values are large but count is small)
```

---

### Technique 3: Multi-map for Multiple Values per Key

```cpp
// When you need ALL indices of each element, not just one
unordered_map<int, vector<int>> allIndices;
for (int i = 0; i < n; i++) allIndices[nums[i]].push_back(i);

// When you need first AND last index
unordered_map<int, pair<int,int>> range; // value → (first_index, last_index)
for (int i = 0; i < n; i++) {
    if (!range.count(nums[i])) range[nums[i]] = {i, i};
    else range[nums[i]].second = i;
}
```

---

### Technique 4: Rolling Hash for String Problems (Preview of Pattern 33)

```cpp
// Hash a substring in O(1) after O(n) precomputation
// Useful for "find if any substring matches" problems
const long long MOD = 1e9 + 7;
const long long BASE = 31;

vector<long long> hashPow(n), prefixHash(n+1, 0);
hashPow[0] = 1;
for (int i = 1; i < n; i++) hashPow[i] = hashPow[i-1] * BASE % MOD;
for (int i = 0; i < n; i++) prefixHash[i+1] = (prefixHash[i] * BASE + s[i] - 'a' + 1) % MOD;

// Hash of substring [l, r]:
auto getHash = [&](int l, int r) -> long long {
    return (prefixHash[r+1] - prefixHash[l] * hashPow[r-l+1] % MOD + MOD * MOD) % MOD;
};
```

---

## C++ STL CHEAT SHEET FOR THIS PATTERN

```cpp
// ═══════════════════════════════════════════
// unordered_map operations:
// ═══════════════════════════════════════════
unordered_map<int, int> m;

m[key] = value;           // insert or overwrite
m[key]++;                 // increment (auto-initializes to 0)
m.count(key);             // returns 0 or 1 (existence check)
m.find(key);              // returns iterator (end() if not found)
m.erase(key);             // remove key
m.size();                 // number of entries
m.clear();                // remove all

// Iteration:
for (auto& [key, value] : m) { ... }  // C++17 structured bindings

// ═══════════════════════════════════════════
// unordered_set operations:
// ═══════════════════════════════════════════
unordered_set<int> s;

s.insert(x);             // insert
s.count(x);              // 0 or 1
s.find(x);               // iterator
s.erase(x);              // remove

// Initialize from vector:
unordered_set<int> s(nums.begin(), nums.end());

// ═══════════════════════════════════════════
// Common patterns:
// ═══════════════════════════════════════════

// Count frequency:
for (int x : nums) freq[x]++;

// Find max frequency:
int maxFreq = 0;
for (auto& [k, v] : freq) maxFreq = max(maxFreq, v);

// Check if all frequencies are equal:
unordered_set<int> freqSet;
for (auto& [k, v] : freq) freqSet.insert(v);
bool allEqual = (freqSet.size() == 1);

// Two-pass frequency (increment then decrement):
for (char c : s) freq[c]++;
for (char c : t) freq[c]--;
// If all zeros → s and t are anagrams
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. **Solve all 11 problems in order.** Time yourself. Record times.
2. **For each problem solved, write ONE sentence:** "The key insight was ___"
3. **Mark problems that took > 30 minutes** as CHALLENGE in your revision tracker
4. **Attempt the sign-off** — I will give you ONE unseen problem. You have 25 minutes.
5. **Revision schedule:** Redo problems 7, 8, 10 after 3 days, 7 days, and 14 days.

---

## THE SIGN-OFF CRITERIA (what I will test you on)

When you type SIGN OFF, I will:
1. Give you a problem you haven't seen
2. You identify it as an Arrays/Hashing problem (without me telling you)
3. You write correct, optimal C++ solution
4. You state time/space complexity BEFORE I ask
5. You handle edge cases (empty array, single element, all duplicates, negative numbers)
6. You explain in 60 seconds: what told you this was a hash map problem

If you pass → we move to Pattern 02 (Two Pointers).
If you fail → 2 more problems from CORE set, then retry with different sign-off problem.

---

*Now go solve. Start with LC 1 (Two Sum) and LC 217 (Contains Duplicate). Time yourself. When done, paste your solutions and type SOLVE for review.*
