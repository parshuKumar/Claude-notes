# Pattern 10 — Heap and Priority Queue
## Difficulty Level: CORE
## Interview Frequency: ALWAYS
## Estimated time to master: 4-5 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Heap is the pattern that separates candidates who "know data structures" from candidates who "think with data structures." The knowledge bar is low — everyone knows what a heap is. The execution bar is high — very few people can choose the right heap direction, write a correct custom comparator, or maintain the two-heap median invariant cleanly under interview pressure.

**The big counterintuition you must internalize before anything else:**

To find the **K LARGEST** elements, you use a **MIN-heap**. To find the **K SMALLEST**, you use a **MAX-heap**. This is backwards from what everyone's instinct says, and it trips up good candidates every time. The logic: a min-heap of size K keeps the K largest elements seen so far by evicting the smallest whenever the heap exceeds size K. The top of the heap is the Kth largest — your cutoff. New elements that beat the cutoff replace it.

**At your 1600 level, you:**
- Can code `priority_queue<int>` (max-heap) and probably know `greater<int>` for min-heap
- Cannot write a custom comparator correctly on the first try under pressure
- Have never implemented the two-heap median technique from scratch
- Don't know lazy deletion exists, which means LC 480 (Sliding Window Median) is completely inaccessible to you
- Get confused when to use heap vs sort vs two pointers for "Top K" problems

**After this document, you will:**
- Know exactly which heap direction to use for every problem type — min or max, and why
- Write custom comparators for pairs, structs, and lambdas without hesitation
- Implement the two-heap median from memory with the invariant stated explicitly
- Understand lazy deletion and when it's the only practical option
- Solve the task scheduler problem both with heap simulation AND the O(1) math formula
- Know which heap problems disguise themselves as something else (and which "heap-looking" problems are actually binary search)

**The interview data:**
- "Find Median from Data Stream" (LC 295) appears in 25%+ of Amazon and Google senior rounds — it's a litmus test for data structure fluency
- "Merge K Sorted Lists" (LC 23) is one of the most asked hard problems at every company
- "Top K Frequent Elements" (LC 347) is a warm-up at every company — must be under 10 minutes
- The two-heap pattern shows up disguised in problems about "K closest stocks to a price" or "streaming data with quantile tracking"
- Heap with custom comparator appears in roughly 40% of contest problems involving priority

---

## ELI5 — THE PRIORITY WAITING ROOM ANALOGIES

### Heap as a "Self-Sorting Waiting Room"

Imagine a doctor's clinic where patients wait in a queue — but instead of first-come-first-served, the most critical patient (highest priority) is always seen next. When a new patient arrives, they are placed in the queue and the queue instantly re-sorts to keep the most critical at the front. That is a max-heap.

A min-heap is the same room but for a different doctor — the doctor who sees the LEAST sick patient first (perhaps a checkup queue that serves the healthiest patients to clear them quickly). The least-priority patient is always at the front.

The key property: **the heap only tells you the front (top) efficiently.** You cannot see "the 3rd most critical patient" in O(1) — you'd have to remove the top two first. This is why heap works for "give me the best candidate right now" but not for "give me the ranked list of all candidates."

### The "VIP Room" Metaphor for Top K

A concert has 1000 attendees but only 10 VIP spots (the K largest spenders). You want to track who belongs in the VIP room as tickets are purchased.

**Wrong approach (max-heap):** Sort all 1000 by spending, take top 10. This requires processing everyone before deciding. Memory: O(1000).

**Smart approach (min-heap of size 10):** Maintain a waiting room of 10 people. The person with the LOWEST spending in the room is at the door (min-heap top = weakest VIP). When someone new arrives with a higher spend than the weakest VIP, they replace them. When a new arrival is below the weakest VIP, they don't enter. At any point, the room holds exactly the current top 10.

The min-heap IS the VIP room. Its top is the weakest current VIP (the one who'd be kicked out next). Its size is always exactly K.

### The "Split Room" for Running Median

A party has people streaming in, and you want to know the median age of everyone in the room at any moment.

Split the room physically: **everyone shorter than median is in the left room, everyone taller in the right room.** The boundary between rooms is the median.

Left room: maintains a max-heap — you always know the tallest person on the left (the one closest to the boundary).
Right room: maintains a min-heap — you always know the shortest person on the right (the one closest to the boundary).

When the left room has one more person than the right, the median is the tallest person in the left room. When they're equal, the median is the average of the two boundary people.

When someone new arrives: if they're shorter than the current right-room minimum, they go left; otherwise right. Then rebalance if the rooms become too unequal (one room exceeds the other by more than 1 person).

### Why Heap Beats Sorting for Streaming Data

Sorting gives you the complete ordered list — but it requires ALL data to be present first, and any new data requires a full re-sort. O(n log n) per new element if you're naive about it.

Heap maintains a PARTIAL order — only the top element is precisely known. Insertion and extraction are O(log n). For problems where you only care about the "best current candidate" or "K best seen so far," heap gives you O(log n) per operation instead of O(n log n) per re-sort.

The tradeoff: heap cannot answer "give me the kth element" for arbitrary k without O(k log n) work. If you need full sorted order, sort. If you need only the top K dynamically, heap.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:

**K-selection problems:**
1. Top K largest / smallest elements from a stream or array
2. Kth largest / smallest element (the exact Kth, not top K set)
3. K closest points, K nearest neighbors (custom distance comparator)
4. K most frequent elements (frequency map → heap on frequency)

**Merge problems:**
1. Merge K sorted arrays / linked lists / streams
2. Find the smallest range covering at least one element from each of K lists

**Running / streaming statistics:**
1. Running median (two heaps — the median maintenance pattern)
2. Sliding window median (two heaps + lazy deletion)
3. Kth largest in a stream (persistent min-heap of size K)

**Scheduling / greedy with heap:**
1. Task scheduler with cooldown (always execute the most frequent available task)
2. Reorganize string (prevent adjacent duplicates)
3. IPO / maximize profit (two heaps: filter eligible → pick max profit)

**Graph substrate:**
1. Dijkstra's algorithm (priority queue of {distance, node} — covered in Pattern 14)
2. Prim's MST algorithm (covered in Pattern 31)

### The core heap operations (C++ `priority_queue`):

```
push(val)      — O(log n) — insert element, heap re-orders
top()          — O(1)     — view the priority element (max for max-heap, min for min-heap)
pop()          — O(log n) — remove the priority element
size()         — O(1)
empty()        — O(1)
```

There is no `find(val)`, `remove(val)`, or `update(val)` in O(log n). These limitations drive the need for lazy deletion.

---

## WHEN TO RECOGNIZE THIS PATTERN

**Signal 1:** "Find the Kth largest element" or "return the K largest elements" or "K most frequent"
→ Heap of size K. The exact heap direction depends on the problem: K largest → min-heap of size K. K smallest → max-heap of size K. The counterintuitive direction is the right one. If the problem says "return the K elements" (not just the Kth), the heap contains them all at the end. If it says "return the Kth element specifically," the heap's top is the answer.

**Signal 2:** "Design a class that supports adding numbers and finding the Kth largest at any time" or "streaming data with K-selection"
→ Maintain a persistent min-heap of size K. Each `add()` operation pushes to the heap and pops if size exceeds K. The top is always the current Kth largest. This is the "Kth Largest in a Stream" (LC 703) pattern.

**Signal 3:** "Find the median from a data stream" or "running median" or "maintain the middle element as data arrives"
→ Two-heap pattern. Lower half in a max-heap, upper half in a min-heap. The invariant: max-heap has equal or exactly 1 more element than min-heap. The max-heap's top and min-heap's top are adjacent in the sorted order of all elements seen.

**Signal 4:** "Merge K sorted lists/arrays" or "find the smallest element across K sorted sources at each step"
→ Min-heap containing one element from each source. Pop the minimum, add its successor to the heap. Continue until all sources are exhausted. The heap size is always exactly K (or less as sources empty).

**Signal 5:** "Schedule tasks with a cooldown period" or "arrange characters so no two same characters are adjacent" or "always execute the highest priority available item"
→ Greedy with heap. The heap maintains available items sorted by priority (usually frequency). The cooldown queue holds items that are unavailable. At each step: execute the top of the heap, put it in the cooldown queue, and re-add it to the heap when its cooldown expires.

**Signal 6:** "K closest points to origin" or "K nearest to a given value" or "K cheapest" with a non-standard key
→ Heap with custom comparator. For "K closest" (K smallest distances), use a max-heap of size K ordered by distance — the largest distance at the top gets popped when a closer point arrives. Write the comparator carefully: `comp(a, b)` returns true means "a has lower priority than b" → b comes out first.

**Signal 7:** "Remove an arbitrary element from the heap efficiently" or the problem requires you to "invalidate" previous heap entries
→ Lazy deletion. Maintain a separate "deleted" count map. When you logically delete an element, increment its deletion count. When you `top()` or `pop()`, drain elements whose deletion count is positive before returning the result.

**Signal 8:** Constraints: n up to 10^6, queries are online (process as they arrive), and you need repeated "give me the best current candidate"
→ Heap is the right complexity class. Sorting would require offline processing. Heap handles online queries in O(log n) each.

### Signals that LOOK like Heap but are NOT:

**Fake signal 1:** "Find the Kth smallest element in a sorted matrix" (LC 378)
→ Binary search on the answer is optimal O(n log n). Heap works (O(K log n)) but is not the intended solution and may TLE for large K. Binary search on value + "count elements ≤ mid" is faster.

**Fake signal 2:** "Find the K most similar strings" where similarity is Hamming distance
→ If all strings have the same length and the alphabet is small, this might be better solved with radix sort + selection. Heap works but may be overkill if you can sort efficiently.

**Fake signal 3:** "Sort the array and take the first K elements" — when the array fits entirely in memory and K is close to n
→ Just sort in O(n log n). A heap of size K is only better when K << n (the heap approach is O(n log K)). If K is n/2 or larger, just sort.

**Fake signal 4:** "Two Sum — find if any two elements sum to a target"
→ Hash set, not heap. Heap doesn't accelerate lookup. The problem doesn't have a "take the best candidate" structure.

---

## THE TEMPLATES (C++)

### Template 1: Min-Heap of Size K — Top K Largest Elements

```cpp
// Use for: K largest, Kth largest, K most frequent (after frequency mapping),
//          K closest (with distance as key)
//
// THE COUNTERINTUITION: to find K LARGEST, use a MIN-HEAP.
// The min-heap's top = the SMALLEST of the K largest seen so far.
// This is the "cutoff": anything larger than the top replaces it.
// Anything smaller cannot possibly be in the K largest.

int findKthLargest(vector<int>& nums, int k) {
    // Min-heap of size at most K
    priority_queue<int, vector<int>, greater<int>> minHeap;
    
    for (int num : nums) {
        minHeap.push(num);
        if ((int)minHeap.size() > k) {
            minHeap.pop();  // evict the smallest → the heap keeps the K largest
        }
    }
    
    return minHeap.top();  // the Kth largest = smallest element of the K largest
}

// WHY NOT a max-heap?
// Max-heap approach: push all n elements, then pop K times.
// → O(n + K log n) time, O(n) space.
// Min-heap of size K: push n elements, pop whenever size > K.
// → O(n log K) time, O(K) space.
// For n = 10^6 and K = 10, log K ≈ 3 vs log n ≈ 20. Min-heap is ~6x faster.
// More importantly: min-heap of size K handles STREAMING data — you can add one
// more element at any time without reprocessing everything. Max-heap cannot.

// HOW TO ADAPT — for "return all K largest elements" (not just the Kth):
vector<int> topKLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;
    for (int num : nums) {
        minHeap.push(num);
        if ((int)minHeap.size() > k) minHeap.pop();
    }
    vector<int> result;
    while (!minHeap.empty()) {
        result.push_back(minHeap.top());
        minHeap.pop();
    }
    return result;  // in ascending order (min-heap extraction order)
}

// HOW TO ADAPT — for K SMALLEST (use MAX-HEAP of size K):
// Replace greater<int> with less<int> (or omit it — less<int> is default).
// Now the max-heap top = the LARGEST of the K smallest seen so far.
// When a smaller element arrives, it evicts the largest → heap keeps K smallest.

// TRACE for nums=[3,1,4,1,5,9,2,6], k=3:
// heap=[], push 3 → [3]
// push 1 → [1,3], size=2 ≤ 3
// push 4 → [1,3,4], size=3 ≤ 3
// push 1 → [1,1,3,4], size=4 > 3 → pop min(1) → [1,3,4]
// push 5 → [1,3,4,5] → pop min(1) → [3,4,5]
// push 9 → [3,4,5,9] → pop min(3) → [4,5,9]
// push 2 → [2,4,5,9] → pop min(2) → [4,5,9]
// push 6 → [4,5,6,9] → pop min(4) → [5,6,9]
// Return top = 5. The 3rd largest in [3,1,4,1,5,9,2,6] = 5 ✓
```

---

### Template 2: Kth Largest in a Stream (Persistent Min-Heap)

```cpp
// LC 703: Kth Largest in a Stream
// Maintain a min-heap of size K. Each add() call updates it.
// The top of the heap is always the current Kth largest.

class KthLargest {
    priority_queue<int, vector<int>, greater<int>> minHeap;  // min-heap
    int k;
    
public:
    KthLargest(int k, vector<int>& nums) : k(k) {
        for (int num : nums) add(num);
    }
    
    int add(int val) {
        minHeap.push(val);
        if ((int)minHeap.size() > k) minHeap.pop();
        return minHeap.top();  // current Kth largest
    }
};

// WHY the constructor calls add() instead of pushing directly:
// If nums has more than K elements, we must apply the "evict if size > k" logic
// during initialization too. Calling add() reuses that logic cleanly.
//
// WHY return minHeap.top() and not minHeap.size()?
// After all pushes and pops, the heap has exactly min(current_count, k) elements.
// If fewer than k elements have arrived, we return the smallest seen so far
// (the heap top), which is technically the "kth largest" in a set of < k elements.
// LC 703 guarantees there will always be at least k elements when add() is called,
// so the edge case doesn't arise in practice.
```

---

### Template 3: Custom Comparator — K Closest Points to Origin

```cpp
// LC 973: K Closest Points to Origin
// "Closest" means smallest Euclidean distance. We want K of them.
// K SMALLEST distances → MAX-HEAP of size K ordered by distance.
// The largest distance at the top gets evicted when a closer point arrives.

// METHOD 1: Lambda comparator (C++14+)
vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
    // comp(a, b) returns true → "a has lower priority than b" → b comes out first
    // We want LARGEST distance at top (to evict it) → b has higher priority when dist(b) > dist(a)
    // So: comp(a, b) = (dist(a) < dist(b)) → this makes a "lower priority" when dist(a) is smaller
    //                                       → b (larger distance) at top → max-heap by distance
    auto cmp = [](const vector<int>& a, const vector<int>& b) {
        return a[0]*a[0] + a[1]*a[1] < b[0]*b[0] + b[1]*b[1];
        // True when dist(a) < dist(b) → b has higher priority → larger distance at top
    };
    priority_queue<vector<int>, vector<vector<int>>, decltype(cmp)> maxHeap(cmp);
    
    for (auto& point : points) {
        maxHeap.push(point);
        if ((int)maxHeap.size() > k) maxHeap.pop();  // evict the farthest
    }
    
    vector<vector<int>> result;
    while (!maxHeap.empty()) {
        result.push_back(maxHeap.top());
        maxHeap.pop();
    }
    return result;
}

// METHOD 2: Store {distance, index} pairs — simpler comparator logic
vector<vector<int>> kClosest_v2(vector<vector<int>>& points, int k) {
    // Max-heap of {dist_squared, original_index}
    // greater<pair<int,int>> would give min-heap — we want max-heap (default pair ordering)
    priority_queue<pair<int,int>> maxHeap;  // {dist, index}, max-heap by dist
    
    for (int i = 0; i < (int)points.size(); i++) {
        int dist = points[i][0]*points[i][0] + points[i][1]*points[i][1];
        maxHeap.push({dist, i});
        if ((int)maxHeap.size() > k) maxHeap.pop();  // evict farthest
    }
    
    vector<vector<int>> result;
    while (!maxHeap.empty()) {
        result.push_back(points[maxHeap.top().second]);
        maxHeap.pop();
    }
    return result;
}

// THE COMPARATOR RULE (memorize this):
// For a MAX-HEAP on property f: comp(a, b) = (f(a) < f(b))
//   → "a has lower priority when f(a) is smaller" → larger f at top
// For a MIN-HEAP on property f: comp(a, b) = (f(a) > f(b))
//   → "a has lower priority when f(a) is larger" → smaller f at top
//
// DEFAULT priority_queue uses less<T> (max-heap). So:
//   priority_queue<T, vector<T>, less<T>>    → MAX-HEAP (default)
//   priority_queue<T, vector<T>, greater<T>> → MIN-HEAP
```

---

### Template 4: Top K Frequent Elements — Frequency Map + Heap

```cpp
// LC 347: Top K Frequent Elements
// Phase 1: count frequencies with a hash map
// Phase 2: use a min-heap of size K on frequency — keep K most frequent

vector<int> topKFrequent(vector<int>& nums, int k) {
    // Phase 1: count frequencies
    unordered_map<int, int> freq;
    for (int num : nums) freq[num]++;
    
    // Phase 2: min-heap of {frequency, value} — evict the least frequent
    // Min-heap: smallest frequency at top → evicted first
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>> minHeap;
    
    for (auto& [val, count] : freq) {
        minHeap.push({count, val});
        if ((int)minHeap.size() > k) minHeap.pop();  // evict least frequent
    }
    
    vector<int> result;
    while (!minHeap.empty()) {
        result.push_back(minHeap.top().second);
        minHeap.pop();
    }
    return result;
}

// ALTERNATIVE: Bucket sort approach (O(n) time, O(n) space)
// Create frequency array indexed by count (0 to n)
// bucket[count] holds all elements with that frequency
// Iterate from n down to 1, collect until K elements found
//
// When to use bucket sort: if the interviewer asks for O(n) time
// When to use heap: when you need to handle streaming or incremental updates

// ALTERNATIVE 2: nth_element (O(n) average, in-place, but not stable)
// Convert to {freq, val} pairs, use nth_element on frequency descending

// NOTE: `greater<pair<int,int>>` for pairs works component-wise:
// First compares the first element, then the second.
// So {2, 3} < {3, 1} → {2, 3} has lower priority in greater<> (i.e., {2,3} comes out first)
// This is what we want: min-heap by frequency.
```

---

### Template 5: Merge K Sorted Lists — Min-Heap of Heads

```cpp
// LC 23: Merge K Sorted Lists
// At each step, extract the globally minimum element across all K lists.
// Maintain one node from each list in a min-heap.
// When you extract a node, push its successor to keep the heap "fed."

struct ListNode { int val; ListNode* next; ListNode(int x) : val(x), next(nullptr) {} };

ListNode* mergeKLists(vector<ListNode*>& lists) {
    // Min-heap comparator: smaller val = higher priority (comes out first)
    auto cmp = [](ListNode* a, ListNode* b) {
        return a->val > b->val;  // comp(a,b) = true means "a has lower priority"
                                  // → b comes out first → smaller val comes out first → min-heap
    };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> minHeap(cmp);
    
    // Initialize: push the head of each non-null list
    for (ListNode* head : lists) {
        if (head) minHeap.push(head);
    }
    
    ListNode dummy(0);
    ListNode* curr = &dummy;
    
    while (!minHeap.empty()) {
        ListNode* node = minHeap.top(); minHeap.pop();
        curr->next = node;
        curr = curr->next;
        
        if (node->next) minHeap.push(node->next);  // push the successor
    }
    
    return dummy.next;
}

// WHY dummy node? The head of the result list is unknown until after the first
// extraction. A dummy node lets you treat the first node identically to all others.

// TRACE for lists = [[1,4,5], [1,3,4], [2,6]]:
// Init: heap = {1(L1), 1(L2), 2(L3)} — heap has 3 nodes
// Step 1: pop 1(L1), push 4(L1). result: 1. heap = {1(L2), 2(L3), 4(L1)}
// Step 2: pop 1(L2), push 3(L2). result: 1→1. heap = {2(L3), 3(L2), 4(L1)}
// Step 3: pop 2(L3), push 6(L3). result: 1→1→2. heap = {3(L2), 4(L1), 6(L3)}
// Step 4: pop 3(L2), push 4(L2). result: 1→1→2→3. heap = {4(L1), 4(L2), 6(L3)}
// Step 5: pop 4(L1), push 5(L1). result: 1→1→2→3→4. heap = {4(L2), 5(L1), 6(L3)}
// Step 6: pop 4(L2), no next. result: 1→1→2→3→4→4. heap = {5(L1), 6(L3)}
// Step 7: pop 5(L1), no next. result: ...→4→5. heap = {6(L3)}
// Step 8: pop 6(L3), no next. result: 1→1→2→3→4→4→5→6. heap = {}
// Final: [1,1,2,3,4,4,5,6] ✓

// Complexity: O(N log K) where N = total nodes, K = number of lists
// At any time, heap has at most K nodes (one per list) → log K per operation
// N total operations → O(N log K)

// HOW TO ADAPT for K sorted ARRAYS instead of linked lists:
// Store {value, array_index, element_index} in the heap.
// When you pop {val, arr_idx, elem_idx}, push {arrays[arr_idx][elem_idx+1], arr_idx, elem_idx+1}
// if elem_idx+1 < arrays[arr_idx].size().
```

---

### Template 6: Two Heaps — Running Median

```cpp
// LC 295: Find Median from Data Stream
//
// INVARIANT (two conditions that must hold at all times):
//   1. PARTITION: maxHeap contains the lower half, minHeap the upper half.
//                 Every element in maxHeap ≤ every element in minHeap.
//                 (maxHeap.top() ≤ minHeap.top())
//   2. BALANCE:   maxHeap.size() == minHeap.size()        (even total: average of tops)
//                 OR maxHeap.size() == minHeap.size() + 1 (odd total: maxHeap.top() is median)
//                 maxHeap always has equal or exactly one MORE element.

class MedianFinder {
    priority_queue<int>                              maxHeap;  // lower half, max-heap
    priority_queue<int, vector<int>, greater<int>>  minHeap;  // upper half, min-heap
    
public:
    void addNum(int num) {
        // Step 1: Always push to maxHeap first (put in the lower half initially)
        maxHeap.push(num);
        
        // Step 2: Fix PARTITION violation — if maxHeap's top crept into the upper half
        // This happens when num > minHeap.top() (we put a large number in the lower half)
        if (!minHeap.empty() && maxHeap.top() > minHeap.top()) {
            minHeap.push(maxHeap.top());
            maxHeap.pop();
        }
        
        // Step 3: Fix BALANCE violation
        if ((int)maxHeap.size() > (int)minHeap.size() + 1) {
            // maxHeap has too many — move one to minHeap
            minHeap.push(maxHeap.top());
            maxHeap.pop();
        }
        if ((int)minHeap.size() > (int)maxHeap.size()) {
            // minHeap has too many — move one to maxHeap
            maxHeap.push(minHeap.top());
            minHeap.pop();
        }
    }
    
    double findMedian() {
        if (maxHeap.size() == minHeap.size())
            return (maxHeap.top() + minHeap.top()) / 2.0;
        return maxHeap.top();  // odd count: maxHeap has the extra element = median
    }
};

// TRACE — addNum calls with [5, 2, 10]:
//
// addNum(5):
//   Step 1: push 5 to maxHeap. maxHeap=[5], minHeap=[]
//   Step 2: minHeap is empty, no partition check
//   Step 3: maxHeap.size()=1, minHeap.size()=0 → 1 == 0+1 → balanced
//   findMedian: sizes unequal → return maxHeap.top() = 5 ✓
//
// addNum(2):
//   Step 1: push 2 to maxHeap. maxHeap=[5,2], minHeap=[]
//   Step 2: minHeap empty, no partition check
//   Step 3: maxHeap.size()=2 > minHeap.size()+1=1 → push 5 to minHeap, pop from maxHeap
//           maxHeap=[2], minHeap=[5]
//   findMedian: sizes equal → return (2+5)/2.0 = 3.5 ✓
//
// addNum(10):
//   Step 1: push 10 to maxHeap. maxHeap=[10,2], minHeap=[5]
//   Step 2: maxHeap.top()=10 > minHeap.top()=5 → PARTITION VIOLATION!
//           push 10 to minHeap, pop from maxHeap. maxHeap=[2], minHeap=[5,10]
//   Step 3: minHeap.size()=2 > maxHeap.size()=1 → push minHeap.top()=5 to maxHeap, pop
//           maxHeap=[5,2], minHeap=[10]
//   findMedian: maxHeap.size()=2 > minHeap.size()=1 → return maxHeap.top() = 5 ✓
//   (Actual median of [2,5,10] = 5 ✓)

// WHY "always push to maxHeap first"?
// We need to place num in the correct half. But we don't know which half it belongs to
// until we check the current partition boundary (minHeap.top()).
// By pushing to maxHeap first and then potentially moving it to minHeap (Step 2),
// we handle both cases with one code path instead of an if-else.
// Alternative (cleaner intent, same result):
//   if (maxHeap.empty() || num <= maxHeap.top()) maxHeap.push(num);
//   else minHeap.push(num);
//   then rebalance. — Both approaches are equivalent.
```

---

### Template 7: Lazy Deletion in Heap

```cpp
// Use when: you need to "remove" elements from a heap that are no longer valid,
// but you cannot rebuild the heap or use a different data structure.
// Common scenarios: sliding window median, "best element that hasn't been used,"
// problems where elements expire.

// The pattern: keep a separate "deletion count" map. When an element should be removed,
// increment its count. Before accessing top() or pop(), drain expired elements.

class LazyHeap {
    priority_queue<int> heap;        // max-heap (use greater<int> for min-heap)
    unordered_map<int, int> pending; // value → how many times it's been lazily deleted
    int realSize = 0;                // actual size (excluding lazily deleted)
    
    void cleanup() {
        // Remove elements from the top that have been lazily deleted
        while (!heap.empty() && pending.count(heap.top()) && pending[heap.top()] > 0) {
            pending[heap.top()]--;
            if (pending[heap.top()] == 0) pending.erase(heap.top());
            heap.pop();
        }
    }
    
public:
    void push(int val) {
        heap.push(val);
        realSize++;
    }
    
    void remove(int val) {
        // Logical removal: mark as deleted, don't touch the heap
        pending[val]++;
        realSize--;
    }
    
    int top() {
        cleanup();
        return heap.top();
    }
    
    void pop() {
        cleanup();
        heap.pop();
        realSize--;
    }
    
    bool empty() { return realSize == 0; }
    int size()   { return realSize; }
};

// WHY lazy deletion?
// C++ priority_queue has no O(log n) "remove arbitrary element" operation.
// Rebuilding the heap from scratch = O(n) per removal.
// Lazy deletion = O(1) logical removal (just mark it), O(log n) amortized actual removal
//   (the element is removed the next time it reaches the top).
// Total cleanup cost across all operations: each element is pushed once and popped once
//   → O(n log n) total, O(log n) amortized per operation.

// PRACTICAL USE — Sliding Window Median (LC 480):
// Maintain two lazy-deletion heaps (max-heap + min-heap, like Template 6).
// When the window slides, the element leaving the window is "removed" lazily.
// The cleanup happens automatically as you access top().
// Don't forget to also handle the size balance after lazy removal.
```

---

### Template 8: Task Scheduler — Heap + Cooldown Queue

```cpp
// LC 621: Task Scheduler
// Rule: same task cannot be repeated within n time units.
// Goal: minimum total time to complete all tasks.
// Greedy: always execute the most frequent AVAILABLE task.
// If no task is available: CPU is idle.

int leastInterval(vector<char>& tasks, int n) {
    // Count frequencies
    unordered_map<char, int> freq;
    for (char t : tasks) freq[t]++;
    
    // Max-heap of frequencies (we want to execute the most frequent task first)
    priority_queue<int> maxHeap;
    for (auto& [ch, cnt] : freq) maxHeap.push(cnt);
    
    // Cooldown queue: {remaining_count, time_when_available}
    queue<pair<int,int>> cooldown;
    int time = 0;
    
    while (!maxHeap.empty() || !cooldown.empty()) {
        time++;
        
        if (!maxHeap.empty()) {
            int cnt = maxHeap.top() - 1;  // execute the most frequent task
            maxHeap.pop();
            if (cnt > 0) {
                cooldown.push({cnt, time + n});  // available again at time+n
            }
        }
        // else: no task available → idle (time increments but we do nothing)
        
        // Re-add tasks whose cooldown has expired
        if (!cooldown.empty() && cooldown.front().second == time) {
            maxHeap.push(cooldown.front().first);
            cooldown.pop();
        }
    }
    
    return time;
}

// THE MATHEMATICAL FORMULA (O(1) — know this as a follow-up):
// maxFreq = maximum task frequency
// maxCount = number of tasks with maximum frequency
// The "frame" concept: (maxFreq-1) complete frames of size (n+1),
// each containing one of the most frequent tasks + n other tasks.
// Last frame: maxCount tasks.
// Answer = max(tasks.size(), (maxFreq-1)*(n+1) + maxCount)
//
// Why max with tasks.size()? If there are enough diverse tasks to fill all cooldown
// periods, there is no idle time and the answer is just the total number of tasks.

int leastInterval_Math(vector<char>& tasks, int n) {
    vector<int> freq(26, 0);
    for (char t : tasks) freq[t - 'A']++;
    sort(freq.begin(), freq.end());
    int maxFreq = freq[25];
    int maxCount = count(freq.begin(), freq.end(), maxFreq);
    return max((int)tasks.size(), (maxFreq - 1) * (n + 1) + maxCount);
}

// TRACE for tasks = [A,A,A,B,B,B], n = 2:
// freq = {A:3, B:3}. maxFreq = 3, maxCount = 2.
// Formula: max(6, (3-1)*(2+1) + 2) = max(6, 8) = 8
// Schedule: A B _ A B _ A B → 8 time units ✓
// The "frame" is [A, B, _]. There are 2 complete frames + 1 last frame [A, B].
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

### VARIANT 1: Top K Frequent Elements via Bucket Sort (O(n) Alternative to Heap)

The heap approach is O(n log K). But bucket sort can do it in O(n).

```cpp
// Phase 1: count frequencies — O(n)
// Phase 2: bucket[freq] = list of elements with that frequency — O(n) setup
// Phase 3: iterate buckets from high to low frequency, collect K elements — O(n)

vector<int> topKFrequent_Bucket(vector<int>& nums, int k) {
    unordered_map<int, int> freq;
    for (int num : nums) freq[num]++;
    
    // Bucket: index = frequency (0 to n)
    vector<vector<int>> bucket(nums.size() + 1);
    for (auto& [val, cnt] : freq) {
        bucket[cnt].push_back(val);
    }
    
    vector<int> result;
    for (int cnt = nums.size(); cnt >= 1 && (int)result.size() < k; cnt--) {
        for (int val : bucket[cnt]) {
            result.push_back(val);
            if ((int)result.size() == k) break;
        }
    }
    
    return result;
}

// WHEN to use: when the interviewer asks "can you do better than O(n log n)?"
// or when constraints make O(n) necessary (n up to 10^7).
// WHEN NOT to use: when you need to handle STREAMING data (can't preallocate bucket array).
```

---

### VARIANT 2: Reorganize String (LC 767) — Greedy Heap Character Placement

```cpp
// Place characters so no two adjacent characters are the same.
// Greedy: always place the most frequent remaining character.
// Constraint: cannot place the same character as the PREVIOUS position.
// → "The most frequent available character" = most frequent character that ≠ prev.

string reorganizeString(string s) {
    unordered_map<char, int> freq;
    for (char c : s) freq[c]++;
    
    // Max-heap of {frequency, character}
    priority_queue<pair<int,char>> maxHeap;
    for (auto& [ch, cnt] : freq) maxHeap.push({cnt, ch});
    
    string result;
    pair<int,char> prev = {0, '#'};  // previous character placed (initially none)
    
    while (!maxHeap.empty()) {
        auto [cnt, ch] = maxHeap.top(); maxHeap.pop();
        
        result += ch;
        
        // Re-add the previous character (it's now "available" again — one step gap)
        if (prev.first > 0) maxHeap.push(prev);
        
        // This character becomes "unavailable" for the next step
        prev = {cnt - 1, ch};
    }
    
    // If we used the prev (last character) had remaining count, it couldn't be placed
    if (prev.first > 0) return "";  // impossible: most frequent character has no valid position
    
    return result;
}

// KEY INSIGHT: After placing character ch, we don't immediately re-add it to the heap.
// We hold it in `prev` for one step (the cooldown of 1). The next step, we re-add `prev`
// and hold the newly placed character. This ensures no two adjacent same characters.
//
// IMPOSSIBILITY CHECK: if any character appears more than (n+1)/2 times, it's impossible.
// The heap solution naturally returns "" in this case (prev.first > 0 at the end).

// COMPARISON with Task Scheduler:
// Same greedy principle: always pick the most frequent available character.
// Task Scheduler has cooldown of n; Reorganize String has cooldown of 1.
// Both use max-heap + cooldown mechanism.
```

---

### VARIANT 3: IPO (LC 502) — Two Heaps for Sequential Decision Making

```cpp
// IPO: given capital requirements and profits for projects, start with capital W.
// Pick at most k projects (one at a time) to maximize final capital.
// Greedy: among all AFFORDABLE projects, pick the one with MAX PROFIT.
// As your capital grows, new projects become affordable → re-check.

int findMaximizedCapital(int k, int w, vector<int>& profits, vector<int>& capital) {
    int n = profits.size();
    
    // Sort projects by capital requirement (ascending)
    vector<pair<int,int>> projects(n);
    for (int i = 0; i < n; i++) projects[i] = {capital[i], profits[i]};
    sort(projects.begin(), projects.end());
    
    // Max-heap of profits for CURRENTLY AFFORDABLE projects
    priority_queue<int> availableProfits;
    int idx = 0;  // pointer into sorted projects array
    
    for (int i = 0; i < k; i++) {
        // Add all projects now affordable (capital[j] <= current capital w)
        while (idx < n && projects[idx].first <= w) {
            availableProfits.push(projects[idx].second);
            idx++;
        }
        
        if (availableProfits.empty()) break;  // no affordable project → stop
        
        w += availableProfits.top();  // pick the most profitable affordable project
        availableProfits.pop();
    }
    
    return w;
}

// TWO-HEAP INSIGHT:
// Heap 1 (implicit — a sorted array with pointer): projects sorted by capital.
//   Use a pointer `idx` to sweep from left (cheapest) as capital grows.
//   This acts as a "filter" that releases projects into Heap 2 as they become affordable.
// Heap 2 (max-heap of profits): profits of currently affordable projects.
//   Pick the maximum each round.
//
// This pattern — "filter available items into a heap, then greedily pick from heap" —
// appears in many scheduling/resource allocation problems.
//
// Time: O(n log n) for sorting + O(k log n) for k heap operations = O(n log n)
// Space: O(n) for the sorted array and heap
```

---

### VARIANT 4: Smallest Range Covering Elements from K Lists (LC 632)

```cpp
// LC 632: Find the smallest range [l, r] such that at least one element from each
// of K sorted lists falls within [l, r].
//
// APPROACH: sliding window approach using a min-heap.
// At all times, maintain one element from each list in the heap.
// Track the GLOBAL MAXIMUM across all current elements.
// The current window = [minHeap.top(), globalMax].
// Advance the window by replacing the minimum element with its successor.

vector<int> smallestRange(vector<vector<int>>& nums) {
    // Min-heap of {value, list_index, element_index}
    using T = tuple<int, int, int>;
    priority_queue<T, vector<T>, greater<T>> minHeap;
    
    int globalMax = INT_MIN;
    
    // Initialize: push the first element of each list
    for (int i = 0; i < (int)nums.size(); i++) {
        minHeap.push({nums[i][0], i, 0});
        globalMax = max(globalMax, nums[i][0]);
    }
    
    int rangeL = 0, rangeR = INT_MAX;
    
    while (!minHeap.empty()) {
        auto [val, listIdx, elemIdx] = minHeap.top(); minHeap.pop();
        
        // Current window: [val, globalMax]
        if (globalMax - val < rangeR - rangeL) {
            rangeL = val;
            rangeR = globalMax;
        }
        
        // Advance: replace the minimum with its successor
        if (elemIdx + 1 < (int)nums[listIdx].size()) {
            int next = nums[listIdx][elemIdx + 1];
            minHeap.push({next, listIdx, elemIdx + 1});
            globalMax = max(globalMax, next);
        } else {
            break;  // One list is exhausted → no valid range can include all lists
        }
    }
    
    return {rangeL, rangeR};
}

// WHY break when one list is exhausted?
// Once we advance past the last element of any list, there's no element from that list
// to include in the range. So no valid range [l, r] exists for any larger window starting
// from that point. The minimum valid range was already checked.
//
// KEY: globalMax tracks the MAXIMUM across all currently-maintained elements.
// When we advance the minimum, globalMax can only stay the same or increase.
// The range can only grow or stay the same as we advance past the smallest elements.
// → We check the range at EVERY step and track the minimum range seen.
```

---

### VARIANT 5: Last Stone Weight (LC 1046) — Max-Heap Simulation

```cpp
// Repeatedly smash the two heaviest stones. If equal: both destroyed.
// If unequal: heavier stone remains with weight = diff.
// Return weight of last stone (0 if none).

int lastStoneWeight(vector<int>& stones) {
    priority_queue<int> maxHeap(stones.begin(), stones.end());
    
    while (maxHeap.size() > 1) {
        int y = maxHeap.top(); maxHeap.pop();  // heaviest
        int x = maxHeap.top(); maxHeap.pop();  // second heaviest
        
        if (x != y) maxHeap.push(y - x);  // remainder survives
        // if x == y: both destroyed, push nothing
    }
    
    return maxHeap.empty() ? 0 : maxHeap.top();
}

// INITIALIZATION TRICK: priority_queue supports range initialization from iterators.
// priority_queue<int> pq(stones.begin(), stones.end()) initializes the heap
// in O(n) time using the heap construction algorithm (heapify), which is faster
// than n individual push() calls (which would be O(n log n)).
// For competitive programming: always use range initialization when you have the full data.
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Build heap from n elements | O(n) | O(n) | Heapify (Floyd's) is O(n), not O(n log n) |
| push() one element | O(log n) | — | Sift up: at most height of heap = O(log n) |
| pop() one element | O(log n) | — | Sift down after replacing root |
| top() | O(1) | — | Root is always at index 0 |
| Min-heap of size K (stream of n) | O(n log K) | O(K) | n pushes, each with possible pop; heap height = log K |
| Merge K lists, N total nodes | O(N log K) | O(K) | N pops, each with one push; heap size ≤ K |
| Two-heap median (n insertions) | O(n log n) | O(n) | Each insert does O(log n) heap operations |
| Task scheduler (n tasks) | O(n log D) | O(D) | D = distinct task types; heap size ≤ D |
| Lazy deletion (m operations) | O(m log n) amortized | O(n) | Each element popped at most once total |
| K closest points (n points) | O(n log K) | O(K) | Same as min-heap of size K |
| Smallest range K lists | O(N log K) | O(K) | N total elements pushed/popped; heap size = K |

### The Key Complexity Fact to State Out Loud:

"Heap construction from n elements is O(n) — not O(n log n). This is because Floyd's heapify algorithm starts from the bottom: leaf nodes (about n/2 of them) need 0 sift-down work, nodes one level up need at most 1 sift-down, etc. The total work sums to O(n) by geometric series: sum(k * n/2^k) for k from 0 to log n = O(n). But inserting n elements one by one is O(n log n) because each insertion is O(log n). In C++, `priority_queue<int> pq(v.begin(), v.end())` uses the O(n) heapify; repeated `pq.push()` for all elements is O(n log n)."

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Using Max-Heap When Min-Heap Is Needed for "Top K Largest"

**What they do:** Write `priority_queue<int> maxHeap` (max-heap) to solve "find K largest elements." They push all elements, then pop K times to get the K largest in descending order.

**What goes wrong:** This is technically correct for an offline problem (all data present), but it's O(n + K log n) time and O(n) space. For streaming data, there is no way to process elements one-by-one — you must wait for all data. More critically: this approach produces the K largest in the WRONG order (descending from max-heap) when the problem wants ascending or any particular order. And for "Kth largest in a stream" (LC 703), you simply can't process one element at a time with a max-heap without re-running the entire computation.

**The fix:** Use a min-heap of size K. Push each element, pop if size exceeds K. The heap always contains exactly the K largest elements seen so far. The top is the Kth largest (the "weakest" of the K largest).

```cpp
// WRONG: max-heap for K largest (works offline but not streaming, O(n) space)
int findKthLargest_Wrong(vector<int>& nums, int k) {
    priority_queue<int> maxHeap(nums.begin(), nums.end());
    for (int i = 0; i < k - 1; i++) maxHeap.pop();
    return maxHeap.top();  // technically correct but O(n) space, not streaming-compatible
}

// CORRECT: min-heap of size K (works streaming, O(K) space)
int findKthLargest_Correct(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;
    for (int num : nums) {
        minHeap.push(num);
        if ((int)minHeap.size() > k) minHeap.pop();
    }
    return minHeap.top();
}
```

---

### Mistake 2: Getting the Custom Comparator Logic Backwards

**What they do:** When writing a custom comparator for a min-heap by some key `f`, they write `comp(a, b) = (f(a) < f(b))` — which actually creates a MAX-HEAP by f (the element with the LARGEST f value comes out first). They realize their output is wrong but can't figure out why.

**What goes wrong:** The C++ `priority_queue` comparator `comp(a, b)` means "a should come AFTER b in the priority order" (a has lower priority). For a MAX-heap by f: `comp(a, b) = (f(a) < f(b))` says "a has lower priority when f(a) is smaller" → element with LARGEST f is at top. For MIN-heap by f, you need the OPPOSITE: `comp(a, b) = (f(a) > f(b))`.

**The fix:** Use the mnemonic: **the comparator for min-heap looks like `>`** (greater than), just like `greater<int>` is for min-heap. For max-heap by f: use `f(a) < f(b)` (looks like `less`). For min-heap by f: use `f(a) > f(b)` (looks like `greater`).

```cpp
// WRONG: comp(a,b) = (f(a) < f(b)) — this gives MAX-heap by f, NOT min-heap
auto cmp_Wrong = [](const vector<int>& a, const vector<int>& b) {
    int distA = a[0]*a[0] + a[1]*a[1];
    int distB = b[0]*b[0] + b[1]*b[1];
    return distA < distB;  // "a has lower priority when dist(a) is smaller"
                            // → LARGER distance at top = MAX-heap by distance ✓ for K closest
    // BUT: if you THOUGHT you were writing a min-heap by distance, this is wrong.
};
// NOTE: for K closest, a MAX-heap by distance IS what you want (evict farthest).
// The confusion arises when you mix up your intent.

// RULE:
// To get MAX-HEAP by property f:  comp(a,b) returns (f(a) < f(b))  ← "less than"
// To get MIN-HEAP by property f:  comp(a,b) returns (f(a) > f(b))  ← "greater than"
// → Matches: greater<int> gives MIN-heap (smallest at top)
//            less<int>    gives MAX-heap (largest at top)

// CORRECT — min-heap by distance:
auto cmp_Correct_MinByDist = [](const vector<int>& a, const vector<int>& b) {
    int distA = a[0]*a[0] + a[1]*a[1];
    int distB = b[0]*b[0] + b[1]*b[1];
    return distA > distB;  // "a has lower priority when dist(a) is LARGER" → smallest dist at top
};
```

---

### Mistake 3: Two-Heap Median — Skipping the Partition Check (Step 2)

**What they do:** They implement the two-heap median but only do the balance check (Step 3). They skip checking whether `maxHeap.top() > minHeap.top()` after pushing.

**What goes wrong:** When you push a number larger than `minHeap.top()` to `maxHeap` first, `maxHeap` now contains a number that belongs in the upper half. The PARTITION invariant is violated: an element of the lower half is larger than an element of the upper half. The median calculation becomes incorrect. This bug appears silently on most test cases (only triggers when a number is larger than the current median and you're pushing to maxHeap first).

**The fix:** After pushing to maxHeap, ALWAYS check if `maxHeap.top() > minHeap.top()`. If so, move the maxHeap's top to minHeap. This fixes the partition before the balance check.

```cpp
// WRONG: only balance check, no partition check
void addNum_Wrong(int num) {
    maxHeap.push(num);
    // Skip partition check — BUG: if num > minHeap.top(), partition is violated
    if (maxHeap.size() > minHeap.size() + 1) {
        minHeap.push(maxHeap.top()); maxHeap.pop();
    }
    if (minHeap.size() > maxHeap.size()) {
        maxHeap.push(minHeap.top()); minHeap.pop();
    }
}
// Fails on: addNum(1), addNum(7), addNum(3) — try tracing manually with the fix omitted

// CORRECT: partition check (Step 2) before balance check (Step 3)
void addNum_Correct(int num) {
    maxHeap.push(num);
    
    // Step 2: Fix partition
    if (!minHeap.empty() && maxHeap.top() > minHeap.top()) {
        minHeap.push(maxHeap.top()); maxHeap.pop();
    }
    
    // Step 3: Fix balance
    if ((int)maxHeap.size() > (int)minHeap.size() + 1) {
        minHeap.push(maxHeap.top()); maxHeap.pop();
    }
    if ((int)minHeap.size() > (int)maxHeap.size()) {
        maxHeap.push(minHeap.top()); minHeap.pop();
    }
}
```

---

### Mistake 4: Merge K Lists — Forgetting to Push the Successor Node

**What they do:** They pop from the heap and append the node to the result. But they forget to push `node->next` to the heap. The heap drains after K pops (one per list head), producing only K elements instead of all N.

**What goes wrong:** The heap is initialized with K head nodes. Each time you pop and forget to push the next, the heap shrinks by 1. After K pops, the heap is empty and the function returns a list of only K nodes — missing all the rest. This is a silent bug: no crash, just wrong output.

**The fix:** After extracting a node from the heap, ALWAYS push `node->next` if it's non-null. The heap is "fed" as you consume it.

```cpp
// WRONG: forgot to push node->next
ListNode* mergeKLists_Wrong(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> minHeap(cmp);
    for (ListNode* head : lists) if (head) minHeap.push(head);
    
    ListNode dummy(0); ListNode* curr = &dummy;
    while (!minHeap.empty()) {
        ListNode* node = minHeap.top(); minHeap.pop();
        curr->next = node; curr = curr->next;
        // FORGOT: if (node->next) minHeap.push(node->next);
        // Result: only the head nodes of each list are extracted → wrong output
    }
    return dummy.next;
}

// CORRECT:
ListNode* mergeKLists_Correct(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> minHeap(cmp);
    for (ListNode* head : lists) if (head) minHeap.push(head);
    
    ListNode dummy(0); ListNode* curr = &dummy;
    while (!minHeap.empty()) {
        ListNode* node = minHeap.top(); minHeap.pop();
        curr->next = node; curr = curr->next;
        if (node->next) minHeap.push(node->next);  // ← CRITICAL: feed the heap
    }
    return dummy.next;
}
```

---

### Mistake 5: Lazy Deletion — Not Maintaining Correct `realSize`

**What they do:** They implement lazy deletion with a pending map, but forget to track the "real" size separately. They use `heap.size()` to check emptiness, which includes lazily-deleted elements. Their loop runs too many iterations, accessing elements that have logically been deleted.

**What goes wrong:** `heap.size()` counts ALL elements — both valid and logically deleted. If 50 elements were logically deleted but not physically cleaned up yet, `heap.size()` reports 50 more elements than actually exist. Code that loops `while (heap.size() > 0)` or checks heap size for business logic will produce wrong results.

**The fix:** Maintain a separate `realSize` counter. Increment on push, decrement on logical remove (before cleanup). Use `realSize == 0` for the empty check. Only use `heap.size()` internally for cleanup detection.

```cpp
// WRONG: using heap.size() includes lazily-deleted elements
bool isEmpty_Wrong() { return heap.empty(); }  // returns false even if all are deleted

// CORRECT: maintain realSize separately
bool isEmpty_Correct() { return realSize == 0; }
// Track: realSize++ on push, realSize-- on remove (logical)
```

---

### Mistake 6: Task Scheduler — Incrementing Time in the Wrong Place / Using the Wrong Cooldown Condition

**What they do:** They check `cooldown.front().second < time` (strictly less than) instead of `== time` to re-add cooled-down tasks. Or they increment time after the inner logic instead of before, causing off-by-one in cooldown tracking.

**What goes wrong:** If the cooldown condition is wrong, tasks are re-added one step too early (effectively reducing the cooldown by 1) or one step too late (adding unnecessary idle time). A cooldown of n means you can do task A again at time `current + n + 1` — so if you execute A at time t, it becomes available again at `t + n`. If you check `<= time` instead of `== time` for re-addition, tasks are added one time unit too early.

**The fix:** Execute first (`time++`), then check cooldown with `cooldown.front().second == time` (the task becomes available at exactly that time). The re-add check must match the push condition exactly.

```cpp
// WRONG: cooldown check with wrong operator
while (!maxHeap.empty() || !cooldown.empty()) {
    time++;
    if (!maxHeap.empty()) {
        int cnt = maxHeap.top() - 1; maxHeap.pop();
        if (cnt > 0) cooldown.push({cnt, time + n});
    }
    // WRONG: <= instead of ==, re-adds tasks one step too early
    if (!cooldown.empty() && cooldown.front().second <= time) {
        maxHeap.push(cooldown.front().first); cooldown.pop();
    }
}

// CORRECT: == for exact cooldown match
while (!maxHeap.empty() || !cooldown.empty()) {
    time++;
    if (!maxHeap.empty()) {
        int cnt = maxHeap.top() - 1; maxHeap.pop();
        if (cnt > 0) cooldown.push({cnt, time + n});
    }
    if (!cooldown.empty() && cooldown.front().second == time) {  // ← exact match
        maxHeap.push(cooldown.front().first); cooldown.pop();
    }
}
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (build the heap syntax reflex)

**1. Kth Largest Element in a Stream — LC 703 — [Template 2: persistent min-heap of size K]**

Why this problem: Forces you to write the min-heap of size K template that is the foundation of every "Top K" problem. The streaming nature — `add(val)` called repeatedly — makes it impossible to "just sort" and forces the heap approach. Must be under 5 minutes including the class definition.

What to prove: The min-heap always contains exactly the K largest elements seen so far. The top is the Kth largest. When a new element is smaller than the current Kth largest (smaller than the top), it cannot be in the K largest set — the heap correctly discards it automatically (size never exceeds K, so no pop happens).

Target time: 5 minutes.

Edge cases: Initial `nums` has fewer than K elements (heap has fewer than K elements; `add()` still works but `top()` may not return a meaningful Kth largest — LC 703 guarantees this doesn't happen when `add()` is called). K = 1 (the heap always has exactly 1 element — the current maximum). Large K equal to nums.size() (all elements are in the heap, top = minimum).

---

**2. Last Stone Weight — LC 1046 — [Max-heap simulation: repeatedly extract top 2 and push remainder]**

Why this problem: The simplest heap simulation. Forces you to write `priority_queue<int>` (max-heap default) and the extract-two, push-remainder loop. Also introduces the range constructor initialization trick. If you can't do this in 4 minutes, your heap syntax is not ready.

What to prove: The `while (heap.size() > 1)` condition correctly handles 0, 1, and 2+ remaining stones. When both extracted stones are equal, you push nothing (both destroyed). The return handles the empty heap case (all stones destroyed) with `return maxHeap.empty() ? 0 : maxHeap.top()`.

Target time: 4 minutes.

Edge cases: Single stone (return its weight without smashing). All stones equal (one pair at a time: [2,2,2,2] → [2,2] → [0] → but 0 is not pushed → heap empty → return 0). Two stones equal (both destroyed, return 0).

---

### CORE — 4 problems (the essential heap techniques)

**3. K Closest Points to Origin — LC 973 — [Max-heap of size K with custom distance comparator]**

Why this problem: Your first custom comparator problem. Forces you to think through: "I want K SMALLEST distances → use MAX-heap of size K (to evict the farthest) → comp(a,b) returns true when dist(a) < dist(b)." The custom comparator is the entire intellectual challenge. Once you get the direction right, the rest is identical to Template 1.

What to prove: The max-heap correctly keeps the K closest points by evicting the farthest point whenever a closer one arrives. The comparator `comp(a, b) = (dist(a) < dist(b))` gives a max-heap by distance (largest distance at top). You do NOT need to take the square root for distance comparison — compare squared distances to avoid floating point.

Target time: 10 minutes.

Edge cases: K equals the number of points (return all of them). Multiple points at the same distance as the cutoff (any K of them is valid, so tie-breaking doesn't matter for the answer). All points at the same distance (any K is a valid answer).

---

**4. Top K Frequent Elements — LC 347 — [Frequency map + min-heap of size K on frequency]**

Why this problem: The "two-phase" heap problem. Phase 1 is counting (hash map). Phase 2 is selecting the top K by frequency. A very common interview pattern: transform the data into (key, frequency) pairs, then apply the min-heap of size K. The bucket sort alternative (Phase 2 in O(n)) is a good follow-up.

What to prove: The min-heap of size K on frequency correctly keeps the K most frequent elements. After processing all elements: `heap.top().first` is the Kth highest frequency (the cutoff). The heap's K elements ARE the K most frequent. The pair ordering for `greater<pair<int,int>>` is component-wise: compares first element (frequency), then second (value) as tiebreaker.

Target time: 12 minutes.

Edge cases: K equals the number of distinct elements (return all distinct elements). All elements have the same frequency (any K distinct elements is valid). K = 1 (return the single most frequent element; if tie, any is valid).

---

**5. Kth Largest Element in an Array — LC 215 — [Min-heap of size K OR quickselect in O(n) average]**

Why this problem: Two valid approaches, and you must know both. Heap approach: O(n log K) time, O(K) space. Quickselect: O(n) average time, O(1) extra space (in-place). Interviewers often ask for quickselect as a follow-up. Know the heap solution cold for production code; know quickselect exists and its tradeoffs.

What to prove: Heap: identical to Template 1. Quickselect: partition the array around a pivot; if pivot lands at index (n-k), it IS the Kth largest. If pivot lands left of (n-k), recurse right. If right, recurse left. Average O(n) but worst case O(n²) for sorted input — use random pivot to avoid worst case.

Target time: 8 minutes (heap solution). 18 minutes (also implementing quickselect).

Edge cases: Duplicate elements — both heap and quickselect handle correctly. K = 1 (return the maximum). K = n (return the minimum).

---

**6. Merge K Sorted Lists — LC 23 — [Min-heap of heads; feed heap as you consume]**

Why this problem: THE merge problem. Combines heap with linked list traversal. The most asked hard problem at FAANG. Must know: dummy node for head tracking, custom comparator for ListNode, push `node->next` after popping, handle null list heads in initialization.

What to prove: The heap contains at most K nodes at any time (one per non-exhausted list). Each extraction is O(log K). Total nodes = N → total time O(N log K). The dummy node allows treating the first node identically to subsequent nodes (no special case for head assignment).

Target time: 15 minutes.

Edge cases: Empty lists vector (return nullptr). Lists with null heads (skip them in initialization). Lists of different lengths (shorter lists' heads become null earlier — the `if (node->next)` guard handles this). Single list (heap has 1 element, degenerates to list traversal — correct but not optimal).

---

### STRETCH — 3 problems

**7. Find Median from Data Stream — LC 295 — [Two-heap median: max-heap lower half, min-heap upper half]**

Why this problem: The quintessential heap design problem. Tests whether you can maintain a non-trivial invariant across multiple operations. The two conditions — PARTITION (maxHeap.top ≤ minHeap.top) and BALANCE (maxHeap has equal or 1 more) — must both hold after every `addNum`. If you can implement this cleanly from the invariant description, you have heap fluency.

What to prove: After every `addNum()`, both invariants hold. The partition check (Step 2) must happen BEFORE the balance check (Step 3) — otherwise, you might "balance" a partitioned heap, leaving the partition violation in place. `findMedian()` uses the sizes to determine whether the total count is odd (return maxHeap.top()) or even (average the two tops).

Target time: 20 minutes.

Edge cases: Single element (only maxHeap has it, return it). Two equal elements (one in each heap, average = the element itself). Large dataset with all same values (both heaps end up with approximately n/2 of the same value — functionally correct).

---

**8. Task Scheduler — LC 621 — [Max-heap of frequencies + cooldown queue; ALSO know the math formula]**

Why this problem: Tests whether you can combine heap with a simulation (cooldown queue). The heap provides the "most frequent available" selection; the queue provides the cooldown mechanism. Know BOTH approaches: simulation (generalizable) and math formula O(1) (elegant for this specific problem).

What to prove: Simulation: the heap always contains AVAILABLE tasks (cooldown expired). The cooldown queue holds tasks that were just executed and are waiting out their cooldown. At each time step: execute the top of heap (if any), put the used task in cooldown, re-add from cooldown if it's ready. Math: `(maxFreq - 1) * (n + 1) + maxCount` gives the minimum time needed. The answer is `max(tasks.size(), formula)` because if tasks are diverse enough, no idle time is needed.

Target time: 20 minutes (simulation). 10 minutes (math formula after understanding the frame concept).

Edge cases: n = 0 (no cooldown → answer = tasks.size()). All tasks are the same type (maximum idle time needed). All tasks are distinct (no idle time → answer = tasks.size()).

---

**9. Reorganize String — LC 767 — [Max-heap of {freq, char}; "hold previous" cooldown of 1]**

Why this problem: The "cooldown of 1" version of task scheduler — simpler but requires understanding the "hold prev, re-add prev" trick. Forces you to think about the greedy correctness proof: always placing the most frequent available character minimizes the risk of getting stuck with too many of one character at the end.

What to prove: The "hold previous for one step" correctly implements a cooldown of 1. If the most frequent remaining character was placed last, the next most frequent is placed instead. After one step, the previously placed character is re-added. If `prev.first > 0` at the end, the last held character could not be placed — the arrangement is impossible.

Target time: 18 minutes.

Edge cases: String with a single character type (impossible if length > 1). String with all distinct characters (always possible — any order works). Longest character appears exactly ⌈n/2⌉ times (possible, the boundary case).

---

### CONTEST — 2 problems

**10. IPO — LC 502 — [Two heaps: sort by capital + max-heap of available profits]**

Why this is contest level: The "filter → select → repeat" greedy pattern with two data structures. You must recognize that as capital grows, NEW projects become affordable — so a static filter doesn't work. The sorted array with a pointer sweeps available projects into the profit heap as capital increases. Getting both the "which projects are now affordable" and "which affordable project gives max profit" parts right simultaneously is the challenge.

What to prove: Sort projects by capital requirement. Use a pointer `idx` to sweep through them: whenever `capital[idx] ≤ current_w`, push its profit to the max-heap. After sweeping, take the max-profit project (heap top). Repeat k times. If the heap is empty (no affordable projects), stop early.

Target time: 25 minutes.

Edge cases: Cannot afford ANY project initially (W = 0 and all capitals > 0) — skip directly. Only one project with very high profit but very high capital requirement — may be unreachable. k = 0 (return initial capital W unchanged).

---

**11. Smallest Range Covering Elements from K Lists — LC 632 — [Min-heap of K list pointers + global max tracking]**

Why this is contest level: Extends the "merge K sorted" pattern with a window-tracking objective. The invariant is subtle: the current window is `[minHeap.top(), globalMax]`. To shrink the window, you must advance the minimum (pop and push successor) — not the maximum (you can't control globalMax directly, only let it grow). Knowing WHEN to stop (when a list is exhausted) requires careful reasoning.

What to prove: The heap always contains exactly one element per list (initially the first element). The current range = `[heap.top(), globalMax]`. At each step, the minimum element's successor is the only way to potentially shrink the lower bound. globalMax can only grow as you advance. The minimum valid range was already checked before any list is exhausted.

Target time: 30 minutes.

Edge cases: K = 1 (any single element is a valid range [x, x]; return the minimum element across all). All lists have the same single element (range = [x, x]). Lists with a single element each (the heap contains K elements and never grows; the answer is checked after the first pop attempt).

---

## PATTERN CONNECTIONS

**Connects to Sorting (implicit — Pattern 01):**
Heap is a partial sort: it gives you the top 1 element efficiently. Sorting gives you all elements ordered. The choice is: if you need the top K out of n where K << n, heap (O(n log K)) beats full sort (O(n log n)). If K ≈ n, just sort. This distinction shows up in "Top K" problems — always ask yourself whether K << n before reaching for a heap.

**Connects to Sliding Window (Pattern 03):**
Sliding Window Median (LC 480) is the marriage of two heaps (Pattern 10) and sliding window (Pattern 03). The window advances; elements enter (push to heap) and leave (lazy deletion from heap). The median is maintained via the two-heap invariant across the window. This is the hardest combination problem from both patterns.

**Connects to Graphs — Shortest Path (Pattern 14):**
Dijkstra's algorithm IS a heap problem. The priority queue stores `{dist, node}` pairs; the min-heap ensures you always process the nearest unvisited node first. Pattern 10's heap templates are the foundation for Pattern 14's Dijkstra. Once you internalize the "always process the best candidate" greedy insight from heap problems, Dijkstra becomes obvious.

**Connects to Graphs — MST (Pattern 31):**
Prim's MST algorithm uses a min-heap of `{edge_weight, node}` to greedily add the cheapest available edge. The heap structure is identical to Dijkstra's. Pattern 10 is a prerequisite for Patterns 14 and 31 — the heap gives both algorithms their core mechanism.

**Connects to Greedy (Pattern 34):**
Many heap problems are "greedy + heap": always pick the best candidate (greedy choice) from a dynamically maintained set (heap provides O(log n) best-candidate access). Task Scheduler (greedy: execute most frequent) and IPO (greedy: pick highest profit) are explicit greedy + heap combinations. Understanding the greedy correctness proof for heap problems strengthens both patterns.

**Connects to Divide and Conquer (Merge K Sorted → divide-and-conquer merge):**
Merge K sorted lists can be solved with a divide-and-conquer approach: pair up lists, merge each pair, repeat. This is O(N log K) — same as heap — but uses recursion and no additional data structure. The heap approach is iterative and simpler to code under pressure. Both approaches are valid; the heap approach is preferred for interviews.

**Connects to DP on Trees (Pattern 22):**
The "K-th smallest in a BST" problem (LC 230) is solved with inorder traversal (Pattern 08). But if the tree is dynamic (insertions/deletions during queries), a heap maintained over the BST elements gives O(log n) per operation. Understanding when heap beats BST-traversal requires knowing both patterns.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Design a class that continuously receives stock prices and can return the K-th highest price at any time."

**Company type:** Amazon (appears in Amazon leadership principles design rounds), Microsoft, Apple
**Expected time:** 10 minutes
**What they are testing:** Whether you immediately identify this as "Kth largest in a stream," choose min-heap of size K, explain the invariant, and handle the edge cases

**Red flags that get candidates rejected:**
- Sorting the array on every `add()` call — O(n log n) per query, completely ignoring the streaming nature of the problem
- Using a max-heap and popping K times on every query — O(K log n) per query, not amortized efficiently
- Not knowing how to initialize the class with a pre-existing list of prices (forgetting the constructor initialization loop)
- Off-by-one: using `heap.size() >= k` instead of `heap.size() > k` for the eviction condition — produces the (K+1)th largest instead of Kth
- Getting the return value wrong: returning `minHeap.top()` is CORRECT (the Kth largest = smallest element of the K largest set), but many candidates second-guess themselves and return `minHeap.top()` of the MAX-heap instead

**What they want you to say:**
"This is Kth largest in a stream. I maintain a min-heap of exactly size K. The min-heap's top is always the Kth largest element seen so far — it's the 'cutoff' for the K-largest set. When a new price arrives: push it to the heap, and if the heap grows beyond K, pop the minimum (evicting the element that is no longer in the K largest). Each add() is O(log K). The heap takes O(K) space. The invariant: the min-heap always contains exactly the K largest prices seen so far."

**The follow-up:** "What if K changes dynamically? Say the user can query 'give me the Kth largest' for any K up to K_max." → You can't use a fixed-size heap anymore. Sort+binary search for offline queries. For online: maintain a sorted multiset (C++ `multiset` which is a balanced BST, O(log n) insert, O(log n) kth element with order statistics) — though standard C++ multiset doesn't support O(log n) kth element. A Fenwick tree or order statistics tree (policy-based in C++) would work. This is a signal that you understand the limits of each approach.

---

### Q2: "Implement a running median — support adding numbers and finding the current median at any time."

**Company type:** Google (appears in ~25% of Google onsite rounds), Amazon (senior rounds), Bloomberg
**Expected time:** 20 minutes
**What they are testing:** Two-heap invariant maintenance, correct comparator setup for both heaps, the partition + balance two-step fix, median formula based on heap sizes

**Red flags that get candidates rejected:**
- Storing all numbers in a sorted array — O(n) per insertion, O(1) median. Works but wrong complexity — the interviewer is looking for O(log n) per insertion.
- Using TWO max-heaps or TWO min-heaps — completely misses the two-different-heap structure needed for median finding
- Forgetting the partition check (Step 2) — only doing the balance — produces incorrect medians for sequences like [1, 7, 3]
- Confusing which heap is the max-heap and which is the min-heap — lower half in max-heap (top = largest of lower half), upper half in min-heap (top = smallest of upper half)
- Wrong median formula: returning `(maxHeap.top() + minHeap.top()) / 2` when sizes are UNEQUAL (should return just `maxHeap.top()` when maxHeap has one more)
- Integer division in the median: `(maxHeap.top() + minHeap.top()) / 2` — correct only if both are ints and you want an integer result. Return `double`, use `/ 2.0`

**What they want you to say:**
"I use two heaps: a max-heap for the lower half and a min-heap for the upper half. Two invariants must hold: (1) PARTITION: every element in the lower half ≤ every element in the upper half — so maxHeap.top() ≤ minHeap.top(). (2) BALANCE: maxHeap has equal or one more element than minHeap. When I add a number: push to maxHeap first. Then check if the partition is violated — if maxHeap.top() > minHeap.top(), move maxHeap's top to minHeap. Then check balance: if maxHeap has 2 more than minHeap, move one over; if minHeap has more than maxHeap, move one back. The median: if sizes are equal, return the average of the two tops; otherwise return maxHeap's top."

**The follow-up:** "What if elements can also be DELETED from the stream?" → Lazy deletion. When an element is deleted, add it to a `pending` map. Before accessing either heap's top, drain any pending-deleted elements. This requires maintaining real size counts separately from heap sizes.

---

### Q3: "You have K sorted lists of integers. Find the smallest range [l, r] that includes at least one number from each list."

**Company type:** Google (this exact problem has appeared in Google onsite rounds), Jane Street, Two Sigma
**Expected time:** 30 minutes
**What they are testing:** Extension of "merge K sorted" thinking, window invariant reasoning, knowing WHEN to stop the search

**Red flags that get candidates rejected:**
- Trying to use a sliding window directly on a flattened sorted array — the merge takes O(N log K), and you still need to track which list each element came from. Many candidates don't see how to merge the tracking with the window
- Not tracking globalMax — they try to maintain both the min (heap) and max (somehow) simultaneously, ending up with an incorrect or O(N²) approach
- Not knowing to stop when a list is exhausted — they continue pushing and don't realize the window can't be smaller once one list runs out of elements
- Computing range incorrectly: the range is `[heap.top().value, globalMax]` — many candidates compute `globalMax - heap.top().value + 1` (the number of integers in the range) instead of the range endpoints themselves
- Forgetting to initialize globalMax with the initial element of each list

**What they want you to say:**
"This extends 'merge K sorted lists.' I maintain a min-heap with one element per list — the current 'candidate' from each list. At all times, the smallest candidate is at the heap top, and I track the globally largest candidate seen. The current window is [heap.top(), globalMax]. To minimize this range, I advance the minimum (pop the top, push its successor). GlobalMax can only grow as I advance. I update the answer whenever the current window is smaller than the best seen. I stop when a list is exhausted — no valid range can include all K lists once one is empty."

**The follow-up:** "What is the time complexity?" → O(N log K) where N = total elements across all lists. Each element is pushed and popped from the min-heap once. The heap size is always exactly K (one from each list). "Can you solve it with a different approach?" → One alternative: merge all elements into a sorted list with their source list labeled, then use a sliding window that ensures all K lists are represented. This is also O(N log K) for the sort but conceptually simpler — harder to implement correctly under pressure.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

**Q1:** Why do you use a MIN-heap to find the K LARGEST elements? What is the invariant, and what does the heap's top represent?
> **A:** A min-heap of size K maintains the K largest elements seen so far. The invariant: the heap contains exactly the K largest. When a new element arrives, push it; if heap size exceeds K, pop the minimum. The minimum is evicted because it's NOT in the K largest — it's been displaced by the new element. The heap's top (the minimum of the heap) = the Kth largest element = the "weakest" member of the K largest set. Any new element larger than the top displaces it; any element smaller is automatically rejected (the size guard pops it on the very next step).

**Q2:** State both invariants of the two-heap median solution. Which invariant is checked first when adding a new number, and why does order matter?
> **A:** INVARIANT 1 (PARTITION): maxHeap.top() ≤ minHeap.top(). Every element in the lower half ≤ every element in the upper half. INVARIANT 2 (BALANCE): maxHeap.size() == minHeap.size() OR maxHeap.size() == minHeap.size() + 1. The partition invariant is checked FIRST (Step 2) before the balance invariant (Step 3). If you reverse the order: you might "balance" a partitioned heap by moving maxHeap's top to minHeap — but if maxHeap's top is GREATER than minHeap's current top, you're moving a large element to the wrong side, creating a partition violation that you then can't detect in Step 2 (you already did Step 2). The partition check MUST happen while the partitioned element is still on maxHeap.

**Q3:** In the comparator for C++ `priority_queue`, `comp(a, b)` returning `true` means what exactly? How does this determine whether you get a max-heap or min-heap?
> **A:** `comp(a, b) = true` means "a has LOWER priority than b" — b will come out of the queue before a. The `priority_queue` puts the element with the HIGHEST priority at the top. For default `less<int>`: comp(a,b) = (a < b) → b has higher priority when b > a → LARGEST element at top → MAX-HEAP. For `greater<int>`: comp(a,b) = (a > b) → b has higher priority when b < a → SMALLEST at top → MIN-HEAP. For custom comparator targeting max-heap by f: `comp(a,b) = (f(a) < f(b))`. For min-heap by f: `comp(a,b) = (f(a) > f(b))`. The mnemonic: the comparator for min-heap uses `>` (like `greater<int>`).

**Q4:** What is the time complexity of merge K sorted lists with a heap, and why is it O(N log K) and not O(N log N)?
> **A:** O(N log K) where N = total nodes, K = number of lists. The heap size is at most K at any time (one node from each list). Each of N nodes is pushed exactly once and popped exactly once. Each push/pop operation costs O(log heap_size) = O(log K) (not log N, because the heap never grows beyond K). Total: N operations × O(log K) each = O(N log K). Compare: O(N log N) would be if the heap held all N nodes simultaneously. The key: by feeding the heap incrementally (push successor of popped node), heap size stays bounded at K.

**Q5:** What is lazy deletion, and when is it the only practical option?
> **A:** Lazy deletion: instead of physically removing an element from a heap (which C++ priority_queue cannot do in O(log n)), mark it as "pending deletion" in a separate data structure (map or set). When accessing `top()`, drain all pending-deleted elements from the top first. When `pop()` is called, also drain first, then pop. Maintain a separate `realSize` counter (not `heap.size()`) for the logical size. It's the only practical option when: (1) you must remove arbitrary (non-top) elements from a heap — there's no O(log n) remove-by-value in C++ priority_queue, (2) elements "expire" or become invalid (sliding window median, where elements leave the window), (3) rebuilding the heap would be O(n) per removal — lazy deletion amortizes removal cost to O(log n).

**Q6:** For the Task Scheduler problem, what is the mathematical formula for the minimum time, and what does each term represent?
> **A:** `answer = max(tasks.size(), (maxFreq - 1) * (n + 1) + maxCount)` where maxFreq = the highest frequency of any task, maxCount = number of distinct tasks with that highest frequency, n = cooldown period. The formula: imagine arranging tasks in "frames" of size (n+1). Each frame has one of the most-frequent tasks + n other tasks (or idle slots). There are (maxFreq - 1) complete frames, each requiring (n+1) time units. The last frame has maxCount tasks (one for each task type with maximum frequency). Total from formula: (maxFreq-1)(n+1) + maxCount. But if there are enough diverse tasks to fill every cooldown slot, no idle time is needed and the answer is just tasks.size(). The `max` ensures the answer is at least the total number of tasks.

**Q7:** Compare heap vs sorting vs quickselect for "find the Kth largest element in an array." When does each approach win?
> **A:** SORTING: O(n log n) time, O(1) extra space (in-place). Best when: you need the full sorted order anyway, or K ≈ n, or n is small. MIN-HEAP OF SIZE K: O(n log K) time, O(K) space. Best when: K << n (log K << log n → significant speedup), or the data is streaming (can process one element at a time). QUICKSELECT: O(n) average time (O(n²) worst case), O(1) extra space. Best when: n is large, K is arbitrary, you don't need the data sorted, and you can afford to mutate the array. Randomized quickselect avoids the O(n²) worst case. Which to use in interviews: heap if K << n or streaming; quickselect if asked for O(n) time; sorting if K ≈ n or asked for simplicity.

---

## C++ HEAP CHEAT SHEET

```cpp
// ============================================================
// PRIORITY QUEUE SYNTAX — ALL VARIANTS
// ============================================================

// MAX-HEAP (default) — largest at top
priority_queue<int> maxHeap;
maxHeap.push(5);
maxHeap.top();  // returns max
maxHeap.pop();

// MIN-HEAP — smallest at top
priority_queue<int, vector<int>, greater<int>> minHeap;

// MIN-HEAP OF SIZE K (Top K Largest):
// push(val); if (size() > k) pop();
// top() = Kth largest

// MAX-HEAP OF PAIRS — sorts by first element desc, then second desc
priority_queue<pair<int,int>> maxPairHeap;
// e.g., push({freq, value}) → most frequent at top

// MIN-HEAP OF PAIRS — sorts by first element asc
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>> minPairHeap;

// CUSTOM COMPARATOR (lambda — must pass to constructor)
auto cmp = [](int a, int b) { return a > b; };  // min-heap (greater-than → min)
priority_queue<int, vector<int>, decltype(cmp)> customHeap(cmp);

// CUSTOM COMPARATOR RULE:
// MAX-HEAP by f: comp(a,b) = (f(a) < f(b))  → "a has lower priority when f(a) < f(b)"
// MIN-HEAP by f: comp(a,b) = (f(a) > f(b))  → "a has lower priority when f(a) > f(b)"

// INITIALIZATION FROM VECTOR (O(n) heapify, not O(n log n)):
vector<int> v = {3, 1, 4, 1, 5};
priority_queue<int> pq(v.begin(), v.end());  // max-heap, O(n) construction

// ============================================================
// TEMPLATE SELECTOR — WHICH HEAP FOR WHICH PROBLEM
// ============================================================
// K largest elements        → min-heap of size K (top = Kth largest)
// K smallest elements       → max-heap of size K (top = Kth smallest)
// K closest points          → max-heap of size K by distance (top = farthest, evict it)
// K most frequent           → freq map + min-heap of size K by freq
// Kth largest in stream     → class with min-heap of size K, add() method
// Running median            → max-heap (lower half) + min-heap (upper half)
// Merge K sorted            → min-heap of {val, list_idx, elem_idx}
// Task scheduler            → max-heap of freq + cooldown queue
// Reorganize string         → max-heap of {freq, char} + "hold prev" trick
// IPO (filter + max)        → sorted array + pointer + max-heap of profits
// Smallest range K lists    → min-heap of {val, list_idx, elem_idx} + globalMax

// ============================================================
// TWO-HEAP MEDIAN — TEMPLATE
// ============================================================
// priority_queue<int> maxHeap;           // lower half
// priority_queue<int, vector<int>, greater<int>> minHeap;  // upper half
// addNum(num):
//   1. maxHeap.push(num)
//   2. if (!minHeap.empty() && maxHeap.top() > minHeap.top()):
//          minHeap.push(maxHeap.top()); maxHeap.pop()
//   3a. if (maxHeap.size() > minHeap.size() + 1):
//          minHeap.push(maxHeap.top()); maxHeap.pop()
//   3b. if (minHeap.size() > maxHeap.size()):
//          maxHeap.push(minHeap.top()); minHeap.pop()
// findMedian():
//   if (sizes equal) return (maxHeap.top() + minHeap.top()) / 2.0
//   return maxHeap.top()

// ============================================================
// MERGE K SORTED LISTS — TEMPLATE
// ============================================================
// auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
// priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> minHeap(cmp);
// for (ListNode* head : lists) if (head) minHeap.push(head);
// ListNode dummy(0); ListNode* curr = &dummy;
// while (!minHeap.empty()):
//   node = minHeap.top(); minHeap.pop()
//   curr->next = node; curr = curr->next
//   if (node->next) minHeap.push(node->next)  // ← CRITICAL
// return dummy.next

// ============================================================
// LAZY DELETION — PATTERN
// ============================================================
// priority_queue<int> heap;
// unordered_map<int, int> pending;  // val → deletion count
// int realSize = 0;
// push(val): heap.push(val); realSize++
// remove(val): pending[val]++; realSize--
// top(): while (pending.count(heap.top()) && pending[heap.top()] > 0): clean up; return heap.top()
// pop(): top(); heap.pop(); realSize--
// empty(): return realSize == 0

// ============================================================
// KEY COMPLEXITY FACTS
// ============================================================
// Build heap from n elements:    O(n) via heapify (NOT O(n log n))
// push / pop:                    O(log n) each
// top:                           O(1)
// Min-heap of size K, n pushes:  O(n log K) total, O(K) space
// Merge K lists, N total nodes:  O(N log K)
// Two-heap median, n operations: O(n log n) total, O(n) space
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. Write the min-heap-of-size-K template from memory: push, evict if size > K, top = Kth largest. Do this three times on paper. State the invariant out loud each time.

2. For LC 295 (Find Median from Data Stream): implement from memory, then trace through `addNum(5), addNum(2), addNum(10)` step by step on paper, stating which invariant is checked at each step. Verify the median after each addition.

3. Deliberately implement the WRONG lazy deletion (using `heap.size()` instead of `realSize`) and observe how it fails on LC 480 (Sliding Window Median). Then fix it. The deliberate failure cements why `realSize` is necessary.

4. In REVISION-TRACKER.md, add these for weekly revisit:
   - LC 295 (Find Median from Data Stream) — two-heap invariant
   - LC 632 (Smallest Range) — globalMax tracking, when to stop
   - LC 502 (IPO) — two-heap greedy, pointer-based sweep

5. Connecting forward to Pattern 11 (Intervals): the interval merge and meeting rooms II problems use a heap (min-heap of end times) to track when "rooms" or "resources" become available. The heap pattern from this section directly applies. "What is the smallest end time currently in use?" = min-heap top.

6. Connecting forward to Pattern 14 (Shortest Path — Dijkstra): Dijkstra is heap-core. The priority queue stores `{dist, node}`; min-heap on dist ensures you process the nearest unfinished node. Every heap concept from this pattern transfers directly to Dijkstra. Before starting Pattern 14, make sure you can write `priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>>` without hesitation.

7. Comparator fluency drill: write THREE custom comparators from scratch:
   - Min-heap of structs by a specific field
   - Max-heap of pairs by the second element
   - Min-heap by floating-point value (avoid sqrt — compare squared distances)

---

## SIGN-OFF CRITERIA

**Tier 1 (Non-negotiable):**
- [ ] Min-heap of size K template from memory — direction correct, `>` vs `>=` in size check correct, top = Kth largest
- [ ] Can state the TWO invariants of the two-heap median before writing any code
- [ ] Custom comparator: can write `comp(a,b)` for both max-heap AND min-heap by an arbitrary property

**Tier 2:**
- [ ] LC 703 (Kth Largest in Stream) — class with `add()`, under 5 minutes
- [ ] LC 973 (K Closest Points) — custom comparator correct, direction correct
- [ ] LC 23 (Merge K Sorted Lists) — push successor after pop, dummy node, O(N log K)

**Tier 3:**
- [ ] LC 295 (Find Median) — both invariants maintained, correct median formula
- [ ] LC 621 (Task Scheduler) — BOTH heap simulation AND mathematical formula
- [ ] LC 347 (Top K Frequent) — two phases: freq map + heap. Know bucket sort O(n) alternative

**Tier 4 (attempt before sign-off):**
- [ ] LC 502 (IPO) — sorted array pointer + profit max-heap, greedy argument stated
- [ ] LC 632 (Smallest Range) — globalMax tracking, heap feeds as list advances, stop on exhaustion

**Sign-off test:** I give you one unseen heap problem. You must:
1. Identify the sub-pattern (Top K? Two heap? Merge K? Lazy deletion? Scheduling?) and state why
2. Choose the correct heap direction (min or max) and justify it
3. Write complete C++ solution including correct comparator syntax
4. State time and space complexity in terms of n (input size) AND K (if applicable)

Type `SIGN OFF P10` when ready.

---

*Pattern 10 Complete — Heap and Priority Queue*
*Next: Pattern 11 — Intervals*
