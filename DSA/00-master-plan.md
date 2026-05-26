# DSA GRANDMASTER MASTERY — COMPLETE WAR PLAN
## From 1600 → 2000+ Rating | 4 Months | C++
## Created by your Grandmaster Tutor

---

## HONEST ASSESSMENT FIRST

Let me be real with you:

- **4 months at 2 hours/weekday + 4 hours/weekend = ~340 total hours**
- **2000 LeetCode rating** is achievable if you follow this plan with zero deviation
- **Codeforces 2000 (Candidate Master)** in 4 months from scratch restart is aggressive — realistic target is 1800-1900 CF
- **Interview-ready for any FAANG/unicorn** — absolutely achievable in 4 months
- **The bottleneck is DP** — it takes most people 4-6 weeks alone. I've allocated 4 weeks. You may need 5.
- **Contests are non-negotiable** — rating comes from contests, not from grinding problems in comfort

If you skip contests, you will stay at 1600 forever. I have seen it 100 times.

---

## COMPLETE FILE MAP

```
c:\Users\Parsh\Desktop\NOTES\DSA\
├── 00-master-plan.md          ← YOU ARE HERE
├── 01-arrays-and-hashing.md
├── 02-two-pointers.md
├── 03-sliding-window.md
├── 04-stack-and-monotonic-stack.md
├── 05-binary-search.md
├── 06-prefix-sums-and-difference-arrays.md
├── 07-linked-lists.md
├── 08-trees-dfs.md
├── 09-trees-bfs.md
├── 10-heap-and-priority-queue.md
├── 11-intervals.md
├── 12-graphs-bfs-dfs.md
├── 13-graphs-union-find.md
├── 14-graphs-shortest-path.md
├── 15-backtracking.md
├── 16-dp-fundamentals.md
├── 17-dp-1d.md
├── 18-dp-2d-grid.md
├── 19-dp-knapsack.md
├── 20-dp-lcs-family.md
├── 21-dp-lis-family.md
├── 22-dp-on-trees.md
├── 23-dp-interval.md
├── 24-dp-bitmask.md
├── 25-dp-digit.md
├── 26-trie.md
├── 27-segment-tree.md
├── 28-fenwick-tree.md
├── 29-monotonic-deque.md
├── 30-topological-sort.md
├── 31-minimum-spanning-tree.md
├── 32-strongly-connected-components.md
├── 33-string-algorithms.md
├── 34-greedy.md
├── 35-math-and-number-theory.md
├── 36-interview-framework.md
├── 37-company-specific-patterns.md
├── 38-hard-problem-decomposition.md
├── REVISION-TRACKER.md         ← problems to revisit frequently
├── CONTEST-LOG.md              ← contest results and analysis
└── PROGRESS.md                 ← checklist of completed patterns
```

---

## PROGRESS TRACKER CHECKLIST

```
[ ] PHASE 1 — FOUNDATION (Weeks 1-3)
    [ ] Pattern 01 — Arrays and Hashing
    [ ] Pattern 02 — Two Pointers
    [ ] Pattern 03 — Sliding Window
    [ ] Pattern 04 — Stack and Monotonic Stack
    [ ] Pattern 05 — Binary Search (Complete)
    [ ] Pattern 06 — Prefix Sums and Difference Arrays

[ ] PHASE 2 — CORE DATA STRUCTURES (Weeks 3-6)
    [ ] Pattern 07 — Linked Lists
    [ ] Pattern 08 — Trees — DFS Complete
    [ ] Pattern 09 — Trees — BFS and Level Order
    [ ] Pattern 10 — Heap and Priority Queue
    [ ] Pattern 11 — Intervals
    [ ] Pattern 12 — Graphs — BFS and DFS
    [ ] Pattern 13 — Graphs — Union Find (DSU)
    [ ] Pattern 14 — Graphs — Shortest Path
    [ ] Pattern 15 — Backtracking

[ ] PHASE 3 — DYNAMIC PROGRAMMING (Weeks 6-10)
    [ ] Pattern 16 — DP Fundamentals
    [ ] Pattern 17 — 1D DP
    [ ] Pattern 18 — 2D DP and Grid DP
    [ ] Pattern 19 — Knapsack Family
    [ ] Pattern 20 — LCS Family
    [ ] Pattern 21 — LIS Family
    [ ] Pattern 22 — DP on Trees
    [ ] Pattern 23 — Interval DP
    [ ] Pattern 24 — Bitmask DP
    [ ] Pattern 25 — Digit DP

[ ] PHASE 4 — ADVANCED PATTERNS (Weeks 10-14)
    [ ] Pattern 26 — Trie
    [ ] Pattern 27 — Segment Tree
    [ ] Pattern 28 — Fenwick Tree
    [ ] Pattern 29 — Monotonic Deque
    [ ] Pattern 30 — Topological Sort
    [ ] Pattern 31 — Minimum Spanning Tree
    [ ] Pattern 32 — Strongly Connected Components
    [ ] Pattern 33 — String Algorithms
    [ ] Pattern 34 — Greedy (Formal)
    [ ] Pattern 35 — Math and Number Theory

[ ] PHASE 5 — INTERVIEW MASTERCLASS (Weeks 14-16)
    [ ] Pattern 36 — Interview Framework
    [ ] Pattern 37 — Company-Specific Patterns
    [ ] Pattern 38 — Hard Problem Decomposition

[ ] CONTESTS
    [ ] First LeetCode Weekly Contest completed
    [ ] First Codeforces Virtual Contest completed
    [ ] 10 contests completed
    [ ] 20 contests completed
    [ ] 30 contests completed
```

---

---

# COMPLETE PATTERN CURRICULUM — EXPANDED

---

## PATTERN 01 — ARRAYS AND HASHING

**Mastery looks like:** You see any array problem and within 30 seconds know whether a hash map reduces it from O(n²) to O(n). You never write a nested loop when a hash map exists.

**Difficulty:** FOUNDATION
**Interview Frequency:** ALWAYS
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- Frequency counting (count occurrences, find duplicates)
- Two-sum pattern (complement lookup in hash map)
- Grouping by key (anagram grouping, bucket sort by property)
- Prefix sum + hash map combo (subarray sum equals K)
- Index mapping (next occurrence, previous occurrence)
- Encoding/hashing for state (serialize state as key)

**Problem Set (solve in this exact order):**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Two Sum | 1 | Warmup — complement lookup | Easy |
| 2 | Contains Duplicate | 217 | Warmup — hash set existence | Easy |
| 3 | Valid Anagram | 242 | Core — frequency counting | Easy |
| 4 | Group Anagrams | 49 | Core — grouping by sorted key | Medium |
| 5 | Top K Frequent Elements | 347 | Core — frequency + bucket sort | Medium |
| 6 | Product of Array Except Self | 238 | Core — prefix/suffix without division | Medium |
| 7 | Longest Consecutive Sequence | 128 | Stretch — hash set + intelligent start | Medium |
| 8 | Subarray Sum Equals K | 560 | Stretch — prefix sum + hash map | Medium |
| 9 | Encode and Decode Strings | 271 | Stretch — encoding design | Medium |
| 10 | First Missing Positive | 41 | Contest — cyclic sort / index mapping | Hard |
| 11 | Minimum Window Substring (hash part) | 76 | Contest — frequency matching | Hard |

**Revision problems (do these every 2 weeks):**
- Two Sum (LC 1) — must solve in under 3 minutes
- Subarray Sum Equals K (LC 560) — prefix sum + map is fundamental
- Longest Consecutive Sequence (LC 128) — the "start of sequence" insight

**References for deeper study:**
- CP-algorithms: Hashing — https://cp-algorithms.com/string/string-hashing.html
- Competitive Programmer's Handbook (Laaksonen) — Chapter 4

---

## PATTERN 02 — TWO POINTERS

**Mastery looks like:** You see a sorted array or a problem asking about pairs/triplets and immediately think two pointers. You know the invariant at each step. You never confuse when to move left vs right.

**Difficulty:** FOUNDATION
**Interview Frequency:** ALWAYS
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- Left-right pointers on sorted array (two sum II, three sum)
- Fast-slow pointers (cycle detection, middle of linked list)
- Same-direction pointers (remove duplicates, move zeros)
- Opposite-direction pointers (container with most water, trapping rain water)
- Three pointers (3Sum, sort colors / Dutch national flag)
- Partitioning (quick select partition)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Valid Palindrome | 125 | Warmup — opposite direction | Easy |
| 2 | Two Sum II - Input Array Is Sorted | 167 | Warmup — classic left-right | Medium |
| 3 | 3Sum | 15 | Core — three pointers, skip duplicates | Medium |
| 4 | Container With Most Water | 11 | Core — shrink the lesser side | Medium |
| 5 | Sort Colors | 75 | Core — Dutch national flag | Medium |
| 6 | Remove Duplicates from Sorted Array II | 80 | Core — same direction write pointer | Medium |
| 7 | Trapping Rain Water | 42 | Stretch — two pointer with max tracking | Hard |
| 8 | 4Sum | 18 | Stretch — extend 3Sum pattern | Medium |
| 9 | Boats to Save People | 881 | Stretch — greedy + two pointers | Medium |
| 10 | Shortest Unsorted Continuous Subarray | 581 | Contest — find boundaries | Medium |
| 11 | Number of Subsequences That Satisfy Given Sum | 1498 | Contest — two pointer + modular arithmetic | Medium |

**Revision problems:**
- 3Sum (LC 15) — must handle duplicates perfectly every time
- Trapping Rain Water (LC 42) — classic interview problem
- Container With Most Water (LC 11) — prove WHY you move the shorter side

**References:**
- "Two Pointers Technique" — GeeksforGeeks deep dive
- Competitive Programmer's Handbook — Chapter 8 (Amortized analysis for two pointers)

---

## PATTERN 03 — SLIDING WINDOW

**Mastery looks like:** You can identify fixed vs variable window in 10 seconds. You write the expand-shrink loop without thinking. You know the at-most-K trick for exactly-K problems cold.

**Difficulty:** FOUNDATION
**Interview Frequency:** ALWAYS
**Estimated days to master:** 4-5 days (this takes longer than people expect)
**Sub-patterns you MUST know:**
- Fixed-size window (max sum of subarray of size K)
- Variable-size window — find minimum length (minimum window substring)
- Variable-size window — find maximum length (longest substring without repeating)
- At-most-K trick (exactly K = atMost(K) - atMost(K-1))
- Window with frequency map (permutation in string)
- Window with deque (sliding window maximum — cross-reference Pattern 29)
- Shrinkable vs non-shrinkable window templates

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Best Time to Buy and Sell Stock | 121 | Warmup — track min, compute max profit | Easy |
| 2 | Maximum Average Subarray I | 643 | Warmup — fixed window | Easy |
| 3 | Longest Substring Without Repeating Characters | 3 | Core — variable window, hash set | Medium |
| 4 | Longest Repeating Character Replacement | 424 | Core — variable window, max frequency trick | Medium |
| 5 | Permutation in String | 567 | Core — fixed window, frequency match | Medium |
| 6 | Minimum Window Substring | 76 | Core — variable window, shrink to minimize | Hard |
| 7 | Subarrays with K Different Integers | 992 | Stretch — exactly K via at-most-K trick | Hard |
| 8 | Minimum Size Subarray Sum | 209 | Stretch — variable window, prefix sum alternative | Medium |
| 9 | Max Consecutive Ones III | 1004 | Stretch — at-most-K zeros variant | Medium |
| 10 | Sliding Window Maximum | 239 | Contest — deque-based window (Pattern 29 preview) | Hard |
| 11 | Minimum Number of Operations to Make Array Continuous | 2009 | Contest — sorted + sliding window | Hard |

**Revision problems:**
- Longest Substring Without Repeating Characters (LC 3) — under 5 minutes
- Minimum Window Substring (LC 76) — the quintessential sliding window problem
- Subarrays with K Different Integers (LC 992) — the at-most-K trick must be instant

**References:**
- "Sliding Window for Beginners" editorial on LeetCode Discuss (by lee215)
- Aditya Verma YouTube — Sliding Window playlist (excellent for building intuition)

---

## PATTERN 04 — STACK AND MONOTONIC STACK

**Mastery looks like:** You see "next greater element" or "largest rectangle" and immediately think monotonic stack. You know whether to use increasing or decreasing stack. You can trace through the stack state mentally.

**Difficulty:** FOUNDATION
**Interview Frequency:** OFTEN
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Basic stack for matching (valid parentheses, calculator)
- Monotonic increasing stack (next greater element)
- Monotonic decreasing stack (next smaller element)
- Stack for histogram (largest rectangle in histogram)
- Stack for expression evaluation (basic calculator family)
- Stack for nested structures (decode string, flatten nested list)
- Contribution technique with stack (sum of subarray minimums)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Valid Parentheses | 20 | Warmup — basic matching | Easy |
| 2 | Min Stack | 155 | Warmup — auxiliary stack tracking | Medium |
| 3 | Next Greater Element I | 496 | Core — monotonic stack intro | Easy |
| 4 | Daily Temperatures | 739 | Core — monotonic decreasing stack | Medium |
| 5 | Evaluate Reverse Polish Notation | 150 | Core — stack-based evaluation | Medium |
| 6 | Largest Rectangle in Histogram | 84 | Core — monotonic stack classic | Hard |
| 7 | Trapping Rain Water (stack approach) | 42 | Stretch — stack alternative to two pointers | Hard |
| 8 | Sum of Subarray Minimums | 907 | Stretch — contribution technique | Medium |
| 9 | Online Stock Span | 901 | Stretch — monotonic stack + counting | Medium |
| 10 | Maximal Rectangle | 85 | Contest — histogram per row | Hard |
| 11 | Sum of Subarray Ranges | 2104 | Contest — min and max contribution | Medium |

**Revision problems:**
- Daily Temperatures (LC 739) — monotonic stack must be automatic
- Largest Rectangle in Histogram (LC 84) — THE stack problem for interviews
- Sum of Subarray Minimums (LC 907) — contribution technique is contest gold

**References:**
- "Monotonic Stack Explained" — LeetCode editorial for LC 84
- Competitive Programmer's Handbook — Chapter 8 (Nearest smaller values)

---

## PATTERN 05 — BINARY SEARCH (COMPLETE)

**Mastery looks like:** You don't just binary search on sorted arrays. You binary search on the ANSWER. You see constraints like "minimize the maximum" or "find the minimum capacity" and immediately think binary search on answer. You never get off-by-one errors.

**Difficulty:** FOUNDATION
**Interview Frequency:** ALWAYS
**Estimated days to master:** 5-6 days (binary search on answer takes time)
**Sub-patterns you MUST know:**
- Classic binary search (find target in sorted array)
- Find first/last position (lower_bound/upper_bound)
- Search in rotated sorted array (identify sorted half)
- Find peak element (compare with neighbor)
- Binary search on answer (minimize maximum / maximize minimum)
- Binary search on answer with greedy check function
- Binary search on floating point answer (precision-based)
- Bisect on monotonic predicate (first true in T T T ... pattern)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Binary Search | 704 | Warmup — pure binary search | Easy |
| 2 | Search Insert Position | 35 | Warmup — lower_bound equivalent | Easy |
| 3 | Find First and Last Position | 34 | Core — two binary searches | Medium |
| 4 | Search in Rotated Sorted Array | 33 | Core — identify sorted half | Medium |
| 5 | Find Peak Element | 162 | Core — binary search on non-sorted | Medium |
| 6 | Koko Eating Bananas | 875 | Core — binary search on answer (classic) | Medium |
| 7 | Capacity To Ship Packages | 1011 | Stretch — binary search on answer + greedy | Medium |
| 8 | Split Array Largest Sum | 410 | Stretch — minimize the maximum | Hard |
| 9 | Median of Two Sorted Arrays | 4 | Stretch — binary search on partition | Hard |
| 10 | Aggressive Cows (SPOJ) / Magnetic Force Between Two Balls | 1552 | Contest — maximize the minimum | Medium |
| 11 | Minimize Max Distance to Gas Station | 774 | Contest — binary search on floating point | Hard |
| 12 | Find in Mountain Array | 1095 | Contest — peak + two binary searches | Hard |

**Revision problems:**
- Koko Eating Bananas (LC 875) — binary search on answer template problem
- Split Array Largest Sum (LC 410) — "minimize the maximum" archetype
- Search in Rotated Sorted Array (LC 33) — always asked in interviews

**References:**
- "Binary Search — A Different Perspective" by Errichto (YouTube)
- CP-algorithms: Binary Search — https://cp-algorithms.com/num_methods/binary_search.html
- TopCoder Binary Search tutorial

---

## PATTERN 06 — PREFIX SUMS AND DIFFERENCE ARRAYS

**Mastery looks like:** You see "subarray sum" and prefix sum is instant. You see "range update" and difference array is instant. You can do 2D prefix sums without bugs. You combine prefix sums with hash maps naturally.

**Difficulty:** FOUNDATION
**Interview Frequency:** OFTEN
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- 1D prefix sum (range sum query)
- Prefix sum + hash map (subarray sum equals K, divisible by K)
- 2D prefix sum (matrix region sum)
- Difference array (range addition, range increment)
- Prefix XOR (subarray XOR queries)
- Prefix product (with zero handling)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Running Sum of 1d Array | 1480 | Warmup — basic prefix sum | Easy |
| 2 | Range Sum Query - Immutable | 303 | Warmup — prefix sum application | Easy |
| 3 | Subarray Sum Equals K | 560 | Core — prefix sum + hash map | Medium |
| 4 | Continuous Subarray Sum | 523 | Core — prefix sum mod + hash map | Medium |
| 5 | Range Sum Query 2D - Immutable | 304 | Core — 2D prefix sum | Medium |
| 6 | Product of Array Except Self | 238 | Core — prefix/suffix product | Medium |
| 7 | Subarray Sums Divisible by K | 974 | Stretch — modular prefix sum | Medium |
| 8 | Count Number of Nice Subarrays | 1248 | Stretch — prefix sum on transformed array | Medium |
| 9 | Maximum Subarray Sum with One Deletion | 1186 | Stretch — prefix + suffix | Medium |
| 10 | Range Addition | 370 | Contest — difference array | Medium |
| 11 | Corporate Flight Bookings | 1109 | Contest — difference array application | Medium |

**Revision problems:**
- Subarray Sum Equals K (LC 560) — prefix sum + map is EVERYWHERE
- Range Sum Query 2D (LC 304) — 2D prefix sum formula must be memorized
- Subarray Sums Divisible by K (LC 974) — modular arithmetic + prefix sum

**References:**
- CP-algorithms: Prefix sums — related to partial sums section
- "Difference Array" technique — USACO Guide

---

## PATTERN 07 — LINKED LISTS

**Mastery looks like:** You manipulate pointers in your sleep. You never lose track of next pointers during reversal. You use dummy nodes automatically when the head might change. Fast-slow pointer for cycle detection is mechanical.

**Difficulty:** CORE
**Interview Frequency:** OFTEN (especially Amazon, Microsoft)
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- In-place reversal (iterative and recursive)
- Fast-slow pointer (find middle, detect cycle, find cycle start)
- Merge two sorted lists
- Dummy node technique (when head might change)
- Runner technique (reorder list, interleave)
- Copy with random pointer (hash map approach, interleaving approach)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Reverse Linked List | 206 | Warmup — basic reversal | Easy |
| 2 | Linked List Cycle | 141 | Warmup — fast-slow detection | Easy |
| 3 | Merge Two Sorted Lists | 21 | Core — merge with dummy | Easy |
| 4 | Remove Nth Node From End | 19 | Core — two pointer gap technique | Medium |
| 5 | Linked List Cycle II | 142 | Core — find cycle entry point | Medium |
| 6 | Reorder List | 143 | Core — find middle + reverse + merge | Medium |
| 7 | Copy List with Random Pointer | 138 | Stretch — hash map or interleaving | Medium |
| 8 | Add Two Numbers | 2 | Stretch — carry propagation | Medium |
| 9 | LRU Cache | 146 | Stretch — doubly linked list + hash map | Medium |
| 10 | Merge K Sorted Lists | 23 | Contest — heap + linked list | Hard |
| 11 | Reverse Nodes in k-Group | 25 | Contest — batch reversal | Hard |

**Revision problems:**
- Reverse Linked List (LC 206) — must be instant, under 2 minutes
- LRU Cache (LC 146) — asked in every FAANG interview
- Merge K Sorted Lists (LC 23) — combines heap + linked list

**References:**
- "Linked List Problems" — Stanford CS Education handout
- Cracking the Coding Interview — Chapter 2

---

## PATTERN 08 — TREES — DFS COMPLETE

**Mastery looks like:** You think recursively without effort. For any tree problem, you know: what does this node return to its parent? What information does it need from its children? You can convert between recursive and iterative DFS.

**Difficulty:** CORE
**Interview Frequency:** ALWAYS
**Estimated days to master:** 5-6 days
**Sub-patterns you MUST know:**
- Preorder traversal (root → left → right) — recursive and iterative
- Inorder traversal (left → root → right) — recursive and iterative
- Postorder traversal (left → right → root) — recursive and iterative
- "Return value from subtree" pattern (height, diameter, path sum)
- BST operations (search, insert, delete, validate, inorder = sorted)
- Lowest Common Ancestor (binary tree and BST)
- Serialize/Deserialize (preorder with null markers)
- Path problems (root-to-leaf, any-to-any)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Maximum Depth of Binary Tree | 104 | Warmup — basic recursion | Easy |
| 2 | Invert Binary Tree | 226 | Warmup — simple recursive transform | Easy |
| 3 | Diameter of Binary Tree | 543 | Core — return height, track diameter | Easy |
| 4 | Validate Binary Search Tree | 98 | Core — BST invariant with bounds | Medium |
| 5 | Lowest Common Ancestor of BST | 235 | Core — BST property exploitation | Medium |
| 6 | Lowest Common Ancestor of Binary Tree | 236 | Core — postorder + null propagation | Medium |
| 7 | Binary Tree Maximum Path Sum | 124 | Stretch — any-to-any path, split vs extend | Hard |
| 8 | Construct Binary Tree from Preorder and Inorder | 105 | Stretch — divide and conquer on indices | Medium |
| 9 | Serialize and Deserialize Binary Tree | 297 | Stretch — preorder with null markers | Hard |
| 10 | Kth Smallest Element in BST | 230 | Contest — inorder traversal application | Medium |
| 11 | Count Good Nodes in Binary Tree | 1448 | Contest — pass state downward | Medium |

**Revision problems:**
- Binary Tree Maximum Path Sum (LC 124) — THE hard tree problem for interviews
- Lowest Common Ancestor (LC 236) — asked at every company
- Validate BST (LC 98) — the "pass bounds down" pattern

**References:**
- "Tree Problems Patterns" — Neetcode roadmap
- Competitive Programmer's Handbook — Chapter 14 (Tree algorithms)

---

## PATTERN 09 — TREES — BFS AND LEVEL ORDER

**Mastery looks like:** You know when BFS is better than DFS on trees (level-by-level processing, shortest path to leaf). Your BFS template with level tracking is muscle memory.

**Difficulty:** CORE
**Interview Frequency:** OFTEN
**Estimated days to master:** 2-3 days
**Sub-patterns you MUST know:**
- Level order traversal (process level by level)
- Right side view (last node of each level)
- Zigzag traversal (alternate direction per level)
- BFS for shortest path on tree (minimum depth)
- Connect next right pointers (level linking)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Binary Tree Level Order Traversal | 102 | Warmup — basic BFS template | Medium |
| 2 | Minimum Depth of Binary Tree | 111 | Warmup — BFS stops at first leaf | Easy |
| 3 | Binary Tree Right Side View | 199 | Core — last of each level | Medium |
| 4 | Binary Tree Zigzag Level Order | 103 | Core — alternate direction | Medium |
| 5 | Populating Next Right Pointers | 116 | Core — O(1) space BFS | Medium |
| 6 | Average of Levels in Binary Tree | 637 | Core — level processing | Easy |
| 7 | Symmetric Tree | 101 | Stretch — BFS with pair comparison | Easy |
| 8 | Even Odd Tree | 1609 | Stretch — level parity constraints | Medium |
| 9 | Maximum Width of Binary Tree | 662 | Stretch — index tracking per level | Medium |
| 10 | Vertical Order Traversal | 987 | Contest — column-based BFS | Hard |
| 11 | Bus Routes | 815 | Contest — BFS on graph (tree intuition) | Hard |

**Revision problems:**
- Binary Tree Level Order Traversal (LC 102) — BFS template
- Binary Tree Right Side View (LC 199) — very common interview question
- Maximum Width of Binary Tree (LC 662) — index overflow handling

---

## PATTERN 10 — HEAP AND PRIORITY QUEUE

**Mastery looks like:** You reach for a heap whenever you need "K largest/smallest" or "always process the current best." You know C++ priority_queue syntax cold including custom comparators. Two-heap for median is automatic.

**Difficulty:** CORE
**Interview Frequency:** ALWAYS
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Top K elements (K largest, K most frequent)
- K closest (custom comparator with distance)
- Merge K sorted streams (min-heap of heads)
- Two heaps for running median
- Lazy deletion in heap (mark deleted, skip on pop)
- Task scheduling with heap (greedy + cooldown)
- Heap for Dijkstra-like problems

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Kth Largest Element in a Stream | 703 | Warmup — min-heap of size K | Easy |
| 2 | Last Stone Weight | 1046 | Warmup — max-heap simulation | Easy |
| 3 | K Closest Points to Origin | 973 | Core — max-heap of size K | Medium |
| 4 | Top K Frequent Elements | 347 | Core — frequency + heap | Medium |
| 5 | Kth Largest Element in an Array | 215 | Core — heap or quickselect | Medium |
| 6 | Merge K Sorted Lists | 23 | Core — min-heap of list heads | Hard |
| 7 | Find Median from Data Stream | 295 | Stretch — two heaps | Hard |
| 8 | Task Scheduler | 621 | Stretch — greedy with heap | Medium |
| 9 | Reorganize String | 767 | Stretch — greedy character placement | Medium |
| 10 | IPO | 502 | Contest — two heaps (available + best profit) | Hard |
| 11 | Smallest Range Covering Elements from K Lists | 632 | Contest — min-heap + window | Hard |

**Revision problems:**
- Find Median from Data Stream (LC 295) — two heap technique
- Merge K Sorted Lists (LC 23) — heap + merge
- Task Scheduler (LC 621) — heap + greedy

**References:**
- C++ priority_queue documentation — cppreference.com
- "Heap Problems Patterns" — LeetCode Discuss compilation

---

## PATTERN 11 — INTERVALS

**Mastery looks like:** Sort by start time is your first instinct. You handle the merge logic (overlap check, merge, insert) without thinking. You know the greedy proof for non-overlapping intervals.

**Difficulty:** CORE
**Interview Frequency:** OFTEN
**Estimated days to master:** 3 days
**Sub-patterns you MUST know:**
- Merge overlapping intervals (sort + merge)
- Insert interval (binary search + merge)
- Count non-overlapping (greedy, sort by end)
- Meeting rooms (sort + count overlaps)
- Line sweep (event-based, +1 at start, -1 at end)
- Interval intersection

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Meeting Rooms | 252 | Warmup — overlap detection | Easy |
| 2 | Merge Intervals | 56 | Warmup — sort + merge | Medium |
| 3 | Insert Interval | 57 | Core — find position + merge | Medium |
| 4 | Non-overlapping Intervals | 435 | Core — greedy by end time | Medium |
| 5 | Meeting Rooms II | 253 | Core — min heap or line sweep | Medium |
| 6 | Interval List Intersections | 986 | Core — two pointer on intervals | Medium |
| 7 | Minimum Number of Arrows to Burst Balloons | 452 | Stretch — greedy overlap counting | Medium |
| 8 | Employee Free Time | 759 | Stretch — merge + gap finding | Hard |
| 9 | Remove Covered Intervals | 1288 | Stretch — sort + greedy | Medium |
| 10 | My Calendar I/II/III | 729/731/732 | Contest — line sweep variants | Medium/Hard |
| 11 | Data Stream as Disjoint Intervals | 352 | Contest — merge on insert | Hard |

**Revision problems:**
- Merge Intervals (LC 56) — must be under 5 minutes
- Meeting Rooms II (LC 253) — sweep line technique
- Non-overlapping Intervals (LC 435) — greedy proof

---

## PATTERN 12 — GRAPHS — BFS AND DFS

**Mastery looks like:** You build adjacency lists in C++ without thinking. You know BFS = shortest path (unweighted), DFS = explore all paths. You handle visited arrays correctly. You detect cycles in directed and undirected graphs differently.

**Difficulty:** CORE
**Interview Frequency:** ALWAYS
**Estimated days to master:** 5-6 days
**Sub-patterns you MUST know:**
- BFS on grid (flood fill, number of islands, shortest path in grid)
- DFS on grid (connected components, area calculations)
- BFS on graph (shortest path unweighted, word ladder)
- DFS on graph (clone graph, all paths)
- Cycle detection — undirected (parent tracking)
- Cycle detection — directed (color marking: white/gray/black)
- Multi-source BFS (rotting oranges, walls and gates)
- Bipartite check (2-coloring BFS/DFS)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Number of Islands | 200 | Warmup — BFS/DFS flood fill | Medium |
| 2 | Flood Fill | 733 | Warmup — basic DFS on grid | Easy |
| 3 | Clone Graph | 133 | Core — DFS + hash map | Medium |
| 4 | Rotting Oranges | 994 | Core — multi-source BFS | Medium |
| 5 | Pacific Atlantic Water Flow | 417 | Core — reverse BFS from borders | Medium |
| 6 | Course Schedule | 207 | Core — cycle detection in directed graph | Medium |
| 7 | Word Ladder | 127 | Stretch — BFS on implicit graph | Hard |
| 8 | Surrounded Regions | 130 | Stretch — border DFS | Medium |
| 9 | Shortest Path in Binary Matrix | 1091 | Stretch — BFS shortest path | Medium |
| 10 | Walls and Gates | 286 | Contest — multi-source BFS | Medium |
| 11 | Graph Valid Tree | 261 | Contest — connected + no cycle | Medium |
| 12 | Shortest Bridge | 934 | Contest — DFS to find + BFS to expand | Medium |

**Revision problems:**
- Number of Islands (LC 200) — must be instant
- Course Schedule (LC 207) — directed cycle detection
- Word Ladder (LC 127) — BFS on transformed strings

**References:**
- CP-algorithms: Graph traversal — https://cp-algorithms.com/graph/breadth-first-search.html
- USACO Guide: Graph Traversal module
- Competitive Programmer's Handbook — Chapter 12

---

## PATTERN 13 — GRAPHS — UNION FIND (DSU)

**Mastery looks like:** You implement DSU with path compression and union by rank from memory in 2 minutes. You know when Union Find beats BFS/DFS (dynamic connectivity, Kruskal's MST). You can count connected components with DSU.

**Difficulty:** CORE
**Interview Frequency:** OFTEN
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- Basic Union Find with path compression + union by rank
- Connected components counting
- Detect cycle in undirected graph (if u and v same component before union → cycle)
- Dynamic connectivity (process edges one by one)
- Weighted Union Find (for relative relations)
- Union Find on 2D grid (convert 2D to 1D index)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Number of Connected Components | 323 | Warmup — basic DSU | Medium |
| 2 | Redundant Connection | 684 | Warmup — find the cycle edge | Medium |
| 3 | Number of Provinces | 547 | Core — DSU on adjacency matrix | Medium |
| 4 | Accounts Merge | 721 | Core — DSU with string mapping | Medium |
| 5 | Longest Consecutive Sequence (DSU approach) | 128 | Core — alternative DSU solution | Medium |
| 6 | Satisfiability of Equality Equations | 990 | Core — union equals, check not-equals | Medium |
| 7 | Number of Islands II | 305 | Stretch — dynamic connectivity | Hard |
| 8 | Smallest String With Swaps | 1202 | Stretch — DSU + sort within components | Medium |
| 9 | Most Stones Removed | 947 | Stretch — DSU on rows and columns | Medium |
| 10 | Swim in Rising Water (DSU approach) | 778 | Contest — sort edges + DSU | Hard |
| 11 | Minimize Malware Spread | 924 | Contest — DSU + size tracking | Hard |

**Revision problems:**
- Accounts Merge (LC 721) — common interview problem
- Redundant Connection (LC 684) — DSU cycle detection
- Number of Islands II (LC 305) — dynamic DSU

**References:**
- CP-algorithms: Disjoint Set Union — https://cp-algorithms.com/data_structures/disjoint_set_union.html
- "DSU on Tree" technique — Codeforces blog by anta

---

## PATTERN 14 — GRAPHS — SHORTEST PATH

**Mastery looks like:** You implement Dijkstra with priority_queue in C++ from memory. You know when to use Dijkstra vs Bellman-Ford vs BFS. You handle the "K stops" constraint with modified BFS/Dijkstra.

**Difficulty:** CORE
**Interview Frequency:** SOMETIMES (but when it appears, it's decisive)
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Dijkstra's algorithm (non-negative weights, priority queue)
- Bellman-Ford (negative weights, V-1 relaxations)
- 0-1 BFS (edge weights 0 or 1, deque-based)
- BFS with state (node + extra state like fuel/stops)
- Floyd-Warshall (all pairs, small N)
- Modified Dijkstra with constraints (cheapest flights K stops)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Network Delay Time | 743 | Warmup — pure Dijkstra | Medium |
| 2 | Path with Maximum Probability | 1514 | Warmup — max-heap Dijkstra variant | Medium |
| 3 | Cheapest Flights Within K Stops | 787 | Core — Bellman-Ford or modified BFS | Medium |
| 4 | Path With Minimum Effort | 1631 | Core — binary search + BFS or Dijkstra | Medium |
| 5 | Swim in Rising Water | 778 | Core — Dijkstra on grid | Hard |
| 6 | Shortest Path in Binary Matrix | 1091 | Core — BFS (all weights = 1) | Medium |
| 7 | Find the City With Smallest Number of Neighbors | 1334 | Stretch — Floyd-Warshall | Medium |
| 8 | Shortest Path to Get All Keys | 864 | Stretch — BFS with bitmask state | Hard |
| 9 | Minimum Cost to Make at Least One Valid Path | 1368 | Stretch — 0-1 BFS | Hard |
| 10 | Number of Ways to Arrive at Destination | 1976 | Contest — Dijkstra + counting paths | Medium |
| 11 | Shortest Path Visiting All Nodes | 847 | Contest — BFS + bitmask | Hard |

**Revision problems:**
- Network Delay Time (LC 743) — Dijkstra template
- Cheapest Flights Within K Stops (LC 787) — modified BFS/Bellman-Ford
- Path With Minimum Effort (LC 1631) — Dijkstra on grid

**References:**
- CP-algorithms: Dijkstra — https://cp-algorithms.com/graph/dijkstra.html
- CP-algorithms: Bellman-Ford — https://cp-algorithms.com/graph/bellman_ford.html
- William Fiset graph theory YouTube series

---

## PATTERN 15 — BACKTRACKING

**Mastery looks like:** You see the decision tree in your head. You know: what choice am I making, what are my constraints, when do I backtrack. You can prune efficiently. You handle duplicates without producing duplicate results.

**Difficulty:** CORE
**Interview Frequency:** OFTEN
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Subsets (include or exclude each element)
- Permutations (place each element at each position)
- Combinations (choose K from N, maintain order)
- Constraint-based (N-queens, Sudoku — check validity before placing)
- String partitioning (palindrome partitioning)
- Path finding (word search on grid)
- Duplicate handling (sort + skip duplicates at same level)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Subsets | 78 | Warmup — include/exclude | Medium |
| 2 | Permutations | 46 | Warmup — arrangement | Medium |
| 3 | Combinations | 77 | Core — choose K | Medium |
| 4 | Combination Sum | 39 | Core — unlimited reuse | Medium |
| 5 | Subsets II | 90 | Core — handle duplicates | Medium |
| 6 | Permutations II | 47 | Core — duplicates in permutations | Medium |
| 7 | Palindrome Partitioning | 131 | Stretch — partitioning + validation | Medium |
| 8 | Word Search | 79 | Stretch — grid backtracking | Medium |
| 9 | N-Queens | 51 | Stretch — constraint placement | Hard |
| 10 | Sudoku Solver | 37 | Contest — heavy constraint checking | Hard |
| 11 | Letter Combinations of Phone Number | 17 | Contest — multi-branch | Medium |

**Revision problems:**
- Subsets II (LC 90) — duplicate handling pattern
- Combination Sum (LC 39) — reuse pattern
- N-Queens (LC 51) — constraint-based placement

**References:**
- "Backtracking Template" by leetcode user — LeetCode Discuss
- Competitive Programmer's Handbook — Chapter 5 (Complete search)

---

## PATTERN 16 — DP FUNDAMENTALS AND THE DP MINDSET

**Mastery looks like:** Before writing ANY DP code, you can clearly state: (1) what is my state, (2) what is my transition, (3) what are my base cases, (4) what is my answer. You know when top-down is better than bottom-up. You can optimize space.

**Difficulty:** CORE
**Interview Frequency:** ALWAYS (DP appears in 60%+ of hard interview problems)
**Estimated days to master:** 3 days (this is a meta-pattern, mostly conceptual)
**Sub-patterns you MUST know:**
- Top-down memoization (recursion + cache)
- Bottom-up tabulation (iterative DP table)
- State reduction (when you only need previous row/state)
- State definition strategy (what information do I need at each step?)
- Transition formulation (what choices do I have at each state?)
- The "copy previous row or not" insight for many 1D/2D DPs

**Problem Set:** (These are intro problems — see Patterns 17-25 for real drill)

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Fibonacci Number | 509 | Warmup — basic DP intro | Easy |
| 2 | Climbing Stairs | 70 | Warmup — 1D state | Easy |
| 3 | Min Cost Climbing Stairs | 746 | Core — 1D with cost | Easy |
| 4 | House Robber | 198 | Core — take or skip | Medium |
| 5 | House Robber II | 213 | Core — circular constraint | Medium |
| 6 | Unique Paths | 62 | Core — 2D state | Medium |
| 7 | Coin Change | 322 | Stretch — unbounded knapsack disguise | Medium |
| 8 | Longest Increasing Subsequence | 300 | Stretch — subsequence DP | Medium |
| 9 | Word Break | 139 | Stretch — string DP | Medium |
| 10 | Maximum Subarray | 53 | Reference — Kadane's (DP perspective) | Medium |

**Revision problems:**
- House Robber (LC 198) — the "take or skip" archetype
- Coin Change (LC 322) — unbounded knapsack template
- Longest Increasing Subsequence (LC 300) — subsequence DP template

---

## PATTERN 17 — 1D DP

**Mastery looks like:** You identify 1D DP instantly when current answer depends on previous 1-3 states. You optimize space to O(1) when only last 1-2 states matter. Jump game, decode ways, word break are all mechanical.

**Difficulty:** CORE
**Interview Frequency:** ALWAYS
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Linear DP (dp[i] depends on dp[i-1], dp[i-2], ...)
- Decision at each step (take or skip, extend or start new)
- Jumping / reachability (jump game family)
- String DP 1D (decode ways, word break)
- Maximum subarray / subsequence variants

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Climbing Stairs | 70 | Warmup — fibonacci variant | Easy |
| 2 | Min Cost Climbing Stairs | 746 | Warmup — minimum cost path | Easy |
| 3 | House Robber | 198 | Core — take or skip | Medium |
| 4 | House Robber II | 213 | Core — circular array | Medium |
| 5 | Decode Ways | 91 | Core — string to number mapping | Medium |
| 6 | Jump Game | 55 | Core — reachability | Medium |
| 7 | Jump Game II | 45 | Stretch — minimum jumps (greedy/DP) | Medium |
| 8 | Word Break | 139 | Stretch — substring DP | Medium |
| 9 | Maximum Product Subarray | 152 | Stretch — track min and max | Medium |
| 10 | Longest Turbulent Subarray | 978 | Contest — alternating direction | Medium |
| 11 | Delete and Earn | 740 | Contest — house robber disguise | Medium |
| 12 | Minimum Cost For Tickets | 983 | Contest — multi-choice DP | Medium |

**Revision problems:**
- House Robber II (LC 213) — circular array handling
- Word Break (LC 139) — substring DP pattern
- Maximum Product Subarray (LC 152) — track min AND max

---

## PATTERN 18 — 2D DP AND GRID DP

**Mastery looks like:** You define dp[i][j] and immediately know: does it represent "answer using first i items and first j items" or "answer ending at cell (i,j)"? You optimize to 1D when possible.

**Difficulty:** CORE
**Interview Frequency:** OFTEN
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Grid path problems (unique paths, min path sum)
- Square/rectangle problems (maximal square)
- Triangle DP (top-down or bottom-up on triangle)
- Two-sequence 2D DP (edit distance, LCS — see Pattern 20)
- State = (index1, index2) for two arrays

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Unique Paths | 62 | Warmup — basic grid DP | Medium |
| 2 | Unique Paths II | 63 | Warmup — obstacles | Medium |
| 3 | Minimum Path Sum | 64 | Core — minimum cost grid | Medium |
| 4 | Triangle | 120 | Core — triangle bottom-up | Medium |
| 5 | Maximal Square | 221 | Core — dp[i][j] = side length | Medium |
| 6 | Dungeon Game | 174 | Core — reverse DP (bottom-right to top-left) | Hard |
| 7 | Cherry Pickup | 741 | Stretch — two agents simultaneously | Hard |
| 8 | Longest Common Subsequence | 1143 | Stretch — two string DP (see Pattern 20) | Medium |
| 9 | Interleaving String | 97 | Stretch — can s3 be formed from s1, s2? | Medium |
| 10 | Minimum Falling Path Sum | 931 | Contest — grid with diagonal movement | Medium |
| 11 | Maximal Rectangle | 85 | Contest — histogram per row + DP | Hard |

**Revision problems:**
- Maximal Square (LC 221) — the dp[i][j] = min(top, left, diag) + 1 insight
- Dungeon Game (LC 174) — reverse DP thinking
- Cherry Pickup (LC 741) — two-agent DP

---

## PATTERN 19 — KNAPSACK FAMILY

**Mastery looks like:** You see "partition into two subsets with equal sum" and immediately think 0/1 knapsack. You see "how many ways to make change" and think unbounded knapsack. The disguise doesn't fool you anymore.

**Difficulty:** ADVANCED
**Interview Frequency:** OFTEN
**Estimated days to master:** 5-6 days
**Sub-patterns you MUST know:**
- 0/1 Knapsack (each item used once)
- Unbounded Knapsack (each item used unlimited times)
- Bounded Knapsack (each item used K times)
- Subset Sum (target exists? count ways?)
- The "knapsack disguise" — recognizing knapsack in partition problems
- Space optimization (2D → 1D by iterating backwards for 0/1)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Partition Equal Subset Sum | 416 | Warmup — 0/1 knapsack disguise | Medium |
| 2 | Coin Change | 322 | Warmup — unbounded knapsack | Medium |
| 3 | Coin Change II | 518 | Core — unbounded count ways | Medium |
| 4 | Target Sum | 494 | Core — 0/1 knapsack transform | Medium |
| 5 | Last Stone Weight II | 1049 | Core — minimize remaining (knapsack) | Medium |
| 6 | Ones and Zeroes | 474 | Core — 2D knapsack | Medium |
| 7 | Combination Sum IV | 377 | Stretch — permutation counting (order matters) | Medium |
| 8 | Perfect Squares | 279 | Stretch — unbounded knapsack variant | Medium |
| 9 | Profitable Schemes | 879 | Stretch — 3D knapsack | Hard |
| 10 | Minimum Cost to Fill Given Weight | — | Contest — classic knapsack | Medium |
| 11 | Tallest Billboard | 956 | Contest — knapsack on difference | Hard |

**Revision problems:**
- Partition Equal Subset Sum (LC 416) — the knapsack disguise archetype
- Target Sum (LC 494) — the S1 - S2 = target transform
- Coin Change II (LC 518) — combinations vs permutations loop order

---

## PATTERN 20 — LCS FAMILY (LONGEST COMMON SUBSEQUENCE)

**Mastery looks like:** You see two strings and immediately think "dp[i][j] = answer using first i chars of s1 and first j chars of s2." Edit distance is just LCS with costs. You know the reconstruction path.

**Difficulty:** ADVANCED
**Interview Frequency:** OFTEN
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- LCS base case (match → diagonal + 1, no match → max(left, top))
- Edit distance (insert/delete/replace costs)
- Longest common substring (vs subsequence)
- Shortest common supersequence (from LCS)
- Distinct subsequences (counting paths)
- Reconstruction (backtrack through DP table)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Longest Common Subsequence | 1143 | Warmup — classic LCS | Medium |
| 2 | Edit Distance | 72 | Warmup — classic edit distance | Medium |
| 3 | Longest Palindromic Subsequence | 516 | Core — LCS with reversed string | Medium |
| 4 | Shortest Common Supersequence | 1092 | Core — LCS reconstruction | Hard |
| 5 | Distinct Subsequences | 115 | Core — count paths | Hard |
| 6 | Minimum ASCII Delete Sum | 712 | Core — weighted LCS variant | Medium |
| 7 | Interleaving String | 97 | Stretch — can string be interleaved? | Medium |
| 8 | Wildcard Matching | 44 | Stretch — DP matching with wildcards | Hard |
| 9 | Regular Expression Matching | 10 | Stretch — DP with * and . | Hard |
| 10 | Longest Palindromic Substring | 5 | Contest — expand around center or DP | Medium |
| 11 | Minimum Insertion Steps to Make Palindrome | 1312 | Contest — LPS based | Hard |

**Revision problems:**
- Edit Distance (LC 72) — asked in Google interviews constantly
- LCS (LC 1143) — the base template
- Distinct Subsequences (LC 115) — tricky counting DP

---

## PATTERN 21 — LIS FAMILY (LONGEST INCREASING SUBSEQUENCE)

**Mastery looks like:** You implement LIS in both O(n²) and O(n log n). You understand the patience sorting insight. You can modify LIS for "longest chain" problems and 2D extensions.

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- O(n²) LIS (dp[i] = longest ending at i)
- O(n log n) LIS (patience sorting / binary search on tails)
- Number of LIS (count all longest subsequences)
- 2D extension (Russian doll envelopes — sort + LIS)
- Longest chain problems (longest string chain, max chain of pairs)
- LIS reconstruction (backtrack parent pointers)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Longest Increasing Subsequence | 300 | Warmup — classic LIS | Medium |
| 2 | Longest String Chain | 1048 | Warmup — LIS with word insertion | Medium |
| 3 | Number of Longest Increasing Subsequence | 673 | Core — count LIS | Medium |
| 4 | Increasing Triplet Subsequence | 334 | Core — simplified LIS | Medium |
| 5 | Maximum Length of Pair Chain | 646 | Core — greedy or LIS | Medium |
| 6 | Russian Doll Envelopes | 354 | Stretch — sort + LIS O(n log n) | Hard |
| 7 | Longest Arithmetic Subsequence | 1027 | Stretch — LIS with diff state | Medium |
| 8 | Make Array Strictly Increasing | 1187 | Stretch — LIS + optional replacement | Hard |
| 9 | Minimum Operations to Make Array Increasing | 1827 | Contest — greedy LIS variant | Easy |
| 10 | Longest Ideal Subsequence | 2370 | Contest — LIS with gap constraint | Medium |
| 11 | Maximum Height by Stacking Cuboids | 1691 | Contest — 3D sort + LIS | Hard |

**Revision problems:**
- LIS O(n log n) (LC 300) — must implement from memory
- Russian Doll Envelopes (LC 354) — 2D LIS classic
- Number of LIS (LC 673) — count tracking is tricky

---

## PATTERN 22 — DP ON TREES

**Mastery looks like:** You think "what does this subtree return to its parent" and define DP states on tree nodes. Rerooting technique lets you compute "answer if root were at node i" for all nodes in O(n).

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Postorder DP (children computed first, parent aggregates)
- The "take or not-take" on tree nodes (house robber on tree)
- Path DP (max path sum — each node returns best single path)
- Rerooting technique (compute answer rooted at all nodes)
- Tree diameter / center via DP
- Edge coloring DP (binary tree cameras)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Diameter of Binary Tree | 543 | Warmup — basic tree DP | Easy |
| 2 | House Robber III | 337 | Warmup — take/skip on tree | Medium |
| 3 | Binary Tree Maximum Path Sum | 124 | Core — path DP | Hard |
| 4 | Binary Tree Cameras | 968 | Core — greedy/DP on tree | Hard |
| 5 | Longest Path With Different Adjacent Characters | 2246 | Core — path DP with constraint | Hard |
| 6 | Sum of Distances in Tree | 834 | Stretch — rerooting technique | Hard |
| 7 | Maximum Product of Splitted Binary Tree | 1339 | Stretch — subtree sum DP | Medium |
| 8 | Count Number of Maximum BSTs | — | Stretch — Catalan on structure | Medium |
| 9 | Minimum Cost Tree From Leaf Values | 1130 | Contest — interval DP on tree | Medium |
| 10 | Tree of Coprimes | 1766 | Contest — DFS with ancestor tracking | Hard |
| 11 | Number of Good Paths | 2421 | Contest — DSU on tree | Hard |

**Revision problems:**
- Binary Tree Maximum Path Sum (LC 124) — tree DP archetype
- Sum of Distances in Tree (LC 834) — rerooting technique
- House Robber III (LC 337) — tree DP entry point

---

## PATTERN 23 — INTERVAL DP

**Mastery looks like:** You see "merge adjacent" or "remove from middle" and think interval DP. You define dp[i][j] as the answer for subarray/subrange [i..j]. You know the O(n³) loop order.

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES (but appears in FAANG hard rounds)
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- dp[i][j] = best answer for interval [i..j]
- Split point enumeration (try every k in [i..j-1] as split)
- Burst/remove from interval (burst balloons)
- Matrix chain multiplication pattern
- Palindrome partitioning (minimum cuts)
- String compression DP

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Longest Palindromic Subsequence | 516 | Warmup — interval on string | Medium |
| 2 | Palindrome Partitioning II | 132 | Warmup — min cuts | Hard |
| 3 | Burst Balloons | 312 | Core — classic interval DP | Hard |
| 4 | Minimum Cost to Cut a Stick | 1547 | Core — MCM variant | Hard |
| 5 | Matrix Chain Multiplication | — | Core — THE interval DP problem | Medium |
| 6 | Stone Game | 877 | Core — game theory interval DP | Medium |
| 7 | Strange Printer | 664 | Stretch — interval DP on characters | Hard |
| 8 | Minimum Score Triangulation of Polygon | 1039 | Stretch — polygon partitioning | Medium |
| 9 | Minimum Cost to Merge Stones | 1000 | Stretch — K-way merge | Hard |
| 10 | Zuma Game | 488 | Contest — interval DP with removal | Hard |
| 11 | Allocate Mailboxes | 1478 | Contest — interval + partition | Hard |

**Revision problems:**
- Burst Balloons (LC 312) — interval DP archetype
- Minimum Cost to Cut a Stick (LC 1547) — MCM disguise
- Strange Printer (LC 664) — hard interval DP

---

## PATTERN 24 — BITMASK DP

**Mastery looks like:** You see N ≤ 20 and immediately think bitmask DP. You represent subsets as integers. You enumerate submasks efficiently. You know the O(2^n * n) complexity and why it works.

**Difficulty:** ELITE
**Interview Frequency:** RARE (but appears in Codeforces Div 2 D/E)
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- State = bitmask representing which elements are used
- TSP (Traveling Salesman Problem)
- Assignment problems (match elements from two groups)
- Subset enumeration / subset sum with bitmask
- Submask enumeration (iterate over all subsets of a mask)
- dp[mask] = answer using the subset represented by mask

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Subsets (bitmask approach) | 78 | Warmup — enumerate with bits | Medium |
| 2 | Partition to K Equal Sum Subsets | 698 | Warmup — bitmask + pruning | Medium |
| 3 | Shortest Path Visiting All Nodes | 847 | Core — BFS + bitmask | Hard |
| 4 | Minimum XOR Sum of Two Arrays | 1879 | Core — assignment with bitmask | Hard |
| 5 | Maximum Students Taking Exam | 1349 | Core — bitmask DP on rows | Hard |
| 6 | Number of Ways to Wear Different Hats | 1434 | Stretch — people as state | Hard |
| 7 | Find the Shortest Superstring | 943 | Stretch — TSP on strings | Hard |
| 8 | Parallel Courses II | 1494 | Stretch — bitmask + topological order | Hard |
| 9 | Maximize Score After N Operations | 1799 | Contest — pair matching | Hard |
| 10 | Can I Win | 464 | Contest — game theory + bitmask | Medium |
| 11 | Beautiful Arrangement | 526 | Contest — permutation with bitmask | Medium |

**Revision problems:**
- Shortest Path Visiting All Nodes (LC 847) — BFS + bitmask
- Minimum XOR Sum (LC 1879) — assignment template
- Can I Win (LC 464) — game theory with memoization

---

## PATTERN 25 — DIGIT DP

**Mastery looks like:** You see "count numbers in range [L, R] with property X" and digit DP is automatic. You handle the tight constraint, leading zeros, and digit count tracking without bugs.

**Difficulty:** ELITE
**Interview Frequency:** RARE (Codeforces Div 1-2, never in typical interviews)
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Tight constraint (is the current number still bounded by limit?)
- Leading zeros handling
- Digit sum tracking
- Digit frequency tracking
- f(R) - f(L-1) technique for range queries
- The recursive + memoization template

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Count Numbers with Unique Digits | 357 | Warmup — combinatorics or digit DP | Medium |
| 2 | Numbers At Most N Given Digit Set | 902 | Warmup — digit DP intro | Hard |
| 3 | Count of Integers (with digit sum) | 2719 | Core — digit sum DP | Hard |
| 4 | Non-negative Integers without Consecutive Ones | 600 | Core — digit constraint | Hard |
| 5 | Number of Digit One | 233 | Core — count specific digit | Hard |
| 6 | Count Special Integers | 2376 | Stretch — unique digits | Hard |
| 7 | Numbers With Repeated Digits | 1012 | Stretch — complement counting | Hard |
| 8 | Digit Count in Range | 1067 | Stretch — range digit frequency | Hard |
| 9 | Count Stepping Numbers in Range | 2801 | Contest — adjacent digit constraint | Hard |
| 10 | Rotated Digits | 788 | Contest — digit property | Medium |

**Revision problems:**
- Count of Integers (LC 2719) — digit DP template
- Non-negative Integers without Consecutive Ones (LC 600) — classic
- Count Special Integers (LC 2376) — bitmask + digit DP

---

## PATTERN 26 — TRIE

**Mastery looks like:** You implement a Trie from scratch in C++ in 3 minutes. You know when Trie beats hash map (prefix search, XOR optimization). Bit Trie for XOR problems is in your toolkit.

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- Basic Trie (insert, search, startsWith)
- Word search with Trie (DFS + Trie pruning)
- Bit Trie (maximum XOR of two numbers)
- Autocomplete / prefix counting
- Trie with deletion / reference counting

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Implement Trie | 208 | Warmup — basic implementation | Medium |
| 2 | Design Add and Search Words | 211 | Warmup — Trie with wildcard DFS | Medium |
| 3 | Word Search II | 212 | Core — backtracking + Trie | Hard |
| 4 | Replace Words | 648 | Core — prefix replacement | Medium |
| 5 | Maximum XOR of Two Numbers in Array | 421 | Core — bit Trie | Medium |
| 6 | Longest Word in Dictionary | 720 | Core — Trie BFS/DFS | Medium |
| 7 | Map Sum Pairs | 677 | Stretch — Trie with prefix sum | Medium |
| 8 | Palindrome Pairs | 336 | Stretch — Trie for reverse matching | Hard |
| 9 | Stream of Characters | 1032 | Contest — reverse Trie + sliding | Hard |
| 10 | Maximum XOR With an Element From Array | 1707 | Contest — offline + Trie | Hard |

**Revision problems:**
- Implement Trie (LC 208) — must be instant from memory
- Word Search II (LC 212) — Trie + backtracking combination
- Maximum XOR (LC 421) — bit Trie technique

---

## PATTERN 27 — SEGMENT TREE

**Mastery looks like:** You build a segment tree from scratch in C++ (no library) for range sum, range min, range max. You implement lazy propagation for range updates. This is THE competitive programming data structure.

**Difficulty:** ELITE
**Interview Frequency:** RARE in interviews, ALWAYS in CP
**Estimated days to master:** 5-7 days (lazy propagation takes time)
**Sub-patterns you MUST know:**
- Point update + range query (basic segment tree)
- Range update + point query (difference array or lazy)
- Range update + range query (lazy propagation)
- Merge sort tree (for order statistics queries)
- Persistent segment tree (historical queries)
- Segment tree with coordinate compression

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Range Sum Query - Mutable | 307 | Warmup — basic segment tree | Medium |
| 2 | Count of Smaller Numbers After Self | 315 | Core — segment tree on values | Hard |
| 3 | Count of Range Sum | 327 | Core — merge sort / segment tree | Hard |
| 4 | Falling Squares | 699 | Core — coordinate compression + seg tree | Hard |
| 5 | My Calendar I/II/III | 729/731/732 | Core — segment tree intervals | Various |
| 6 | Reverse Pairs | 493 | Stretch — merge sort / BIT / seg tree | Hard |
| 7 | The Skyline Problem | 218 | Stretch — sweep line + structure | Hard |
| 8 | Rectangle Area II | 850 | Contest — sweep + segment tree | Hard |
| 9 | Longest Increasing Subsequence II | 2407 | Contest — segment tree optimization | Hard |
| 10 | Range Module | 715 | Contest — interval segment tree | Hard |

**Revision problems:**
- Range Sum Query Mutable (LC 307) — segment tree template
- Count of Smaller Numbers After Self (LC 315) — practical application
- Falling Squares (LC 699) — lazy propagation

**References:**
- CP-algorithms: Segment Tree — https://cp-algorithms.com/data_structures/segment_tree.html
- "Efficient and Easy Segment Trees" by Al.Cash (Codeforces blog)

---

## PATTERN 28 — FENWICK TREE (BIT)

**Mastery looks like:** You implement a Fenwick tree in 10 lines of C++. You know it handles prefix sum queries and point updates in O(log n). You know when BIT is simpler than segment tree (no lazy propagation needed).

**Difficulty:** ADVANCED
**Interview Frequency:** RARE in interviews, OFTEN in CP
**Estimated days to master:** 2-3 days
**Sub-patterns you MUST know:**
- Point update + prefix query (basic BIT)
- Range update + point query (difference BIT)
- 2D BIT (matrix updates and queries)
- Order statistics with BIT (count inversions)
- Coordinate compression + BIT

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Range Sum Query - Mutable (BIT approach) | 307 | Warmup — basic BIT | Medium |
| 2 | Count of Smaller Numbers After Self | 315 | Core — BIT on coordinate compressed values | Hard |
| 3 | Reverse Pairs | 493 | Core — BIT for counting | Hard |
| 4 | Create Sorted Array through Instructions | 1649 | Core — BIT for cost counting | Hard |
| 5 | Count Good Triplets in an Array | 2179 | Stretch — two BITs | Hard |
| 6 | Range Sum Query 2D - Mutable | 308 | Stretch — 2D BIT | Hard |
| 7 | Global and Local Inversions | 775 | Contest — inversion counting | Medium |
| 8 | Longest Increasing Subsequence (BIT approach) | 300 | Contest — BIT optimization | Medium |

**Revision problems:**
- Count of Smaller Numbers After Self (LC 315) — BIT application
- Reverse Pairs (LC 493) — BIT counting pattern

---

## PATTERN 29 — MONOTONIC DEQUE

**Mastery looks like:** You see "sliding window maximum/minimum" and deque is instant. You maintain the deque invariant (remove from back when new element violates monotonicity). You combine with DP for Jump Game VI type problems.

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES
**Estimated days to master:** 2-3 days
**Sub-patterns you MUST know:**
- Sliding window maximum (deque stores indices, front = max)
- Sliding window minimum
- Monotonic deque + DP (Jump Game VI, constrained subsequence)
- Shortest subarray with sum ≥ K (deque + prefix sum)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Sliding Window Maximum | 239 | Warmup — classic monotonic deque | Hard |
| 2 | Jump Game VI | 1696 | Core — deque + DP | Medium |
| 3 | Constrained Subsequence Sum | 1425 | Core — deque optimized DP | Hard |
| 4 | Shortest Subarray with Sum at Least K | 862 | Stretch — deque + prefix sum | Hard |
| 5 | Longest Continuous Subarray With Abs Diff ≤ Limit | 1438 | Stretch — two deques (max and min) | Medium |
| 6 | Max Value of Equation | 1499 | Contest — deque optimization | Hard |
| 7 | Maximum Number of Robots Within Budget | 2398 | Contest — deque + sliding window | Hard |

**Revision problems:**
- Sliding Window Maximum (LC 239) — deque template
- Shortest Subarray with Sum at Least K (LC 862) — hard deque problem

---

## PATTERN 30 — TOPOLOGICAL SORT

**Mastery looks like:** You see "prerequisites" or "ordering with dependencies" and topological sort is instant. You implement Kahn's (BFS) from memory. You detect cycles in directed graphs via topo sort failure.

**Difficulty:** ADVANCED
**Interview Frequency:** OFTEN
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- Kahn's algorithm (BFS with in-degree counting)
- DFS-based topological sort (finish time ordering)
- Cycle detection via topological sort (if not all nodes processed → cycle)
- All topological orderings (backtracking on zero in-degree nodes)
- Longest path in DAG (topo sort + DP)
- Alien dictionary pattern

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Course Schedule | 207 | Warmup — detect if topo sort possible | Medium |
| 2 | Course Schedule II | 210 | Warmup — output the topo order | Medium |
| 3 | Alien Dictionary | 269 | Core — build graph + topo sort | Hard |
| 4 | Minimum Height Trees | 310 | Core — iterative leaf removal | Medium |
| 5 | Course Schedule IV | 1462 | Core — reachability via topo order | Medium |
| 6 | Longest Increasing Path in a Matrix | 329 | Stretch — implicit DAG + topo sort/memo | Hard |
| 7 | Parallel Courses | 1136 | Stretch — longest path in DAG | Medium |
| 8 | Sequence Reconstruction | 444 | Stretch — unique topo sort | Medium |
| 9 | Sort Items by Groups Respecting Dependencies | 1203 | Contest — two-level topo sort | Hard |
| 10 | Build a Matrix With Conditions | 2392 | Contest — topo sort for row and column | Hard |

**Revision problems:**
- Course Schedule II (LC 210) — topo sort template
- Alien Dictionary (LC 269) — graph construction from constraints
- Longest Increasing Path in a Matrix (LC 329) — memo on DAG

---

## PATTERN 31 — MINIMUM SPANNING TREE

**Mastery looks like:** You implement Kruskal's (sort edges + DSU) and Prim's (priority queue) from memory. You know Kruskal's is better for sparse graphs and Prim's for dense.

**Difficulty:** ADVANCED
**Interview Frequency:** RARE
**Estimated days to master:** 2-3 days
**Sub-patterns you MUST know:**
- Kruskal's algorithm (sort edges + Union Find)
- Prim's algorithm (priority queue greedy)
- MST properties (cut property, cycle property)
- Find critical and pseudo-critical edges
- MST as clustering (stop early for K clusters)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Min Cost to Connect All Points | 1584 | Warmup — Kruskal's or Prim's | Medium |
| 2 | Connecting Cities With Minimum Cost | 1135 | Core — basic MST | Medium |
| 3 | Optimize Water Distribution | 1168 | Core — virtual node + MST | Hard |
| 4 | Find Critical and Pseudo-Critical Edges | 1489 | Stretch — MST properties | Hard |
| 5 | Minimum Spanning Tree (Kruskal) | — | Stretch — implement from scratch | Medium |
| 6 | Remove Max Number of Edges | 1579 | Contest — two DSUs | Hard |

**Revision problems:**
- Min Cost to Connect All Points (LC 1584) — MST template
- Find Critical and Pseudo-Critical Edges (LC 1489) — MST theory

---

## PATTERN 32 — STRONGLY CONNECTED COMPONENTS

**Mastery looks like:** You implement Kosaraju's or Tarjan's algorithm. You can find SCCs and use them for 2-SAT problems. This is competitive programming territory.

**Difficulty:** ELITE
**Interview Frequency:** RARE (almost never in interviews, Codeforces Div 2 E)
**Estimated days to master:** 3-4 days
**Sub-patterns you MUST know:**
- Kosaraju's algorithm (two DFS passes)
- Tarjan's algorithm (single DFS with low-link values)
- SCC condensation (DAG of SCCs)
- 2-SAT reduction to SCC
- Bridges and articulation points (Tarjan's variant)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Critical Connections in a Network | 1192 | Warmup — find bridges | Hard |
| 2 | Strongly Connected Components (implementation) | — | Core — Kosaraju's | Medium |
| 3 | SCC with Tarjan (implementation) | — | Core — Tarjan's | Medium |
| 4 | Minimum Number of Days to Disconnect Island | 1568 | Stretch — articulation point | Hard |
| 5 | 2-SAT problems (Codeforces) | — | Contest — SCC application | Hard |

**Revision problems:**
- Critical Connections (LC 1192) — bridges via Tarjan's

---

## PATTERN 33 — STRING ALGORITHMS

**Mastery looks like:** You implement KMP from memory. You know when to use rolling hash vs KMP vs Z-algorithm. You can solve "find pattern in text" problems optimally.

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- KMP (failure function + pattern matching)
- Z-algorithm (Z-array computation)
- Rabin-Karp (rolling hash for pattern matching)
- Longest palindrome (Manacher's algorithm)
- String hashing for comparison
- Suffix concepts (suffix array basics)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Find the Index of First Occurrence | 28 | Warmup — KMP | Easy |
| 2 | Repeated Substring Pattern | 459 | Core — KMP failure function insight | Easy |
| 3 | Shortest Palindrome | 214 | Core — KMP for longest palindromic prefix | Hard |
| 4 | Longest Happy Prefix | 1392 | Core — KMP / Z-algorithm | Hard |
| 5 | Implement strStr (Rabin-Karp) | 28 | Stretch — rolling hash approach | Easy |
| 6 | Longest Duplicate Substring | 1044 | Stretch — binary search + rolling hash | Hard |
| 7 | Distinct Echo Substrings | 1316 | Contest — rolling hash | Hard |
| 8 | Sum of Scores of Built Strings | 2223 | Contest — Z-algorithm | Hard |
| 9 | Minimum Time to Revert Word to Initial State | 3029 | Contest — Z-function | Hard |

**Revision problems:**
- Shortest Palindrome (LC 214) — KMP application
- Longest Duplicate Substring (LC 1044) — binary search + hash

**References:**
- CP-algorithms: String section — https://cp-algorithms.com/string/
- "KMP Algorithm Explained" — TopCoder tutorial

---

## PATTERN 34 — GREEDY (FORMAL TREATMENT)

**Mastery looks like:** You don't just "feel" greedy is correct — you can argue WHY using exchange argument or greedy stays ahead proof. You know when greedy fails and DP is needed.

**Difficulty:** ADVANCED
**Interview Frequency:** OFTEN
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- Activity selection / interval scheduling (sort by end)
- Task scheduling with cooldown (heap-based greedy)
- Jump game greedy (farthest reachable)
- Candy / distribute resources (two pass)
- Gas station (circular greedy)
- Exchange argument (prove swapping choices doesn't help)
- Fractional vs 0/1 (when greedy works vs when DP is needed)

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Assign Cookies | 455 | Warmup — sort + greedy match | Easy |
| 2 | Jump Game | 55 | Warmup — farthest reachable | Medium |
| 3 | Gas Station | 134 | Core — circular greedy | Medium |
| 4 | Candy | 135 | Core — two pass greedy | Hard |
| 5 | Task Scheduler | 621 | Core — greedy with cooldown | Medium |
| 6 | Minimum Number of Arrows | 452 | Core — interval greedy | Medium |
| 7 | Queue Reconstruction by Height | 406 | Stretch — sort + insert | Medium |
| 8 | Partition Labels | 763 | Stretch — last occurrence greedy | Medium |
| 9 | Minimum Platforms (GFG) | — | Stretch — sweep line greedy | Medium |
| 10 | Earliest Possible Day of Full Bloom | 2136 | Contest — sort by grow time | Hard |
| 11 | Minimum Replacements to Sort Array | 2366 | Contest — reverse greedy | Hard |

**Revision problems:**
- Gas Station (LC 134) — circular greedy proof
- Candy (LC 135) — two pass technique
- Task Scheduler (LC 621) — frequency-based greedy

---

## PATTERN 35 — MATH AND NUMBER THEORY

**Mastery looks like:** GCD in one line. Sieve of Eratosthenes from memory. Modular exponentiation for large powers. nCr mod p using Fermat's little theorem. These show up in Codeforces constantly.

**Difficulty:** ADVANCED
**Interview Frequency:** SOMETIMES
**Estimated days to master:** 4-5 days
**Sub-patterns you MUST know:**
- GCD/LCM (Euclidean algorithm)
- Sieve of Eratosthenes (prime generation)
- Prime factorization
- Modular arithmetic (addition, multiplication, inverse)
- Fast exponentiation (binary exponentiation)
- Combinatorics (nCr mod p, Pascal's triangle)
- Inclusion-Exclusion principle

**Problem Set:**

| # | Problem | LC # | Type | Difficulty |
|---|---------|------|------|-----------|
| 1 | Count Primes | 204 | Warmup — sieve | Medium |
| 2 | Power of Two/Three | 231/326 | Warmup — bit manipulation / math | Easy |
| 3 | GCD of Strings | 1071 | Core — GCD application | Easy |
| 4 | Pow(x, n) | 50 | Core — fast exponentiation | Medium |
| 5 | Count Good Numbers | 1922 | Core — modular arithmetic | Medium |
| 6 | Unique Paths (math approach) | 62 | Core — combinatorics nCr | Medium |
| 7 | Super Pow | 372 | Stretch — modular exponentiation | Medium |
| 8 | Largest Component Size by Common Factor | 952 | Stretch — prime factorization + DSU | Hard |
| 9 | Count Anagrams | 2514 | Contest — modular inverse + factorial | Hard |
| 10 | Apply Operations to Make Sum of Array ≥ k | — | Contest — math optimization | Medium |

**Revision problems:**
- Count Primes (LC 204) — sieve template
- Pow(x, n) (LC 50) — binary exponentiation
- Count Good Numbers (LC 1922) — mod arithmetic

**References:**
- CP-algorithms: Number Theory section — https://cp-algorithms.com/algebra/
- Competitive Programmer's Handbook — Chapters 21-22

---

## PATTERN 36 — THE INTERVIEW FRAMEWORK

**Mastery looks like:** In any interview, you spend the first 5 minutes understanding the problem, asking clarifying questions, stating brute force, then optimizing. You communicate throughout. You never go silent for more than 30 seconds.

**Difficulty:** FOUNDATION (for interviews)
**Interview Frequency:** ALWAYS (meta-skill)
**Estimated days to master:** 2-3 days of deliberate practice
**What this covers:**
- The 5-minute problem understanding protocol
- How to communicate while thinking
- How to transition from brute force to optimal
- How to handle "I don't know"
- Body language and confidence signals
- Common rejection reasons that have nothing to do with code

---

## PATTERN 37 — COMPANY-SPECIFIC PATTERNS

**Mastery looks like:** You know that Amazon loves BFS/DFS + greedy, Google loves DP + math, Microsoft loves trees + linked lists, and Indian unicorns love arrays + sliding window + binary search.

**Difficulty:** N/A (meta-knowledge)
**Interview Frequency:** ALWAYS (preparation strategy)
**Estimated days:** 2-3 days to study, then ongoing

---

## PATTERN 38 — HARD PROBLEM DECOMPOSITION

**Mastery looks like:** You see a hard problem and break it into two medium components. You read constraints and know the intended time complexity. You identify the "aha" moment that makes hard problems tractable.

**Difficulty:** ELITE
**Interview Frequency:** SOMETIMES (only for L5+ / SDE-3+ roles)
**Estimated days to master:** ongoing (this is a lifelong skill)

---

---

# 4-MONTH WEEKLY SCHEDULE

---

## MONTH 1 — FOUNDATION + CORE (Weeks 1-4)

### Week 1: Arrays, Hashing, Two Pointers
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 01 doc study + 2 warmup problems | 2 hrs |
| Tue | Pattern 01: 3 core problems (timed) | 2 hrs |
| Wed | Pattern 01: 2 stretch problems + review | 2 hrs |
| Thu | Pattern 01 sign-off attempt + Pattern 02 doc study | 2 hrs |
| Fri | Pattern 02: 2 warmup + 2 core problems | 2 hrs |
| Sat | Pattern 02: 3 stretch + LeetCode Weekly Contest | 4 hrs |
| Sun | Pattern 02 sign-off + Contest review | 3 hrs |

### Week 2: Sliding Window, Stack
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 03 doc study + 2 warmup problems | 2 hrs |
| Tue | Pattern 03: 3 core problems (timed) | 2 hrs |
| Wed | Pattern 03: 2 stretch + 1 contest problem | 2 hrs |
| Thu | Pattern 03 sign-off + Pattern 04 doc study | 2 hrs |
| Fri | Pattern 04: 2 warmup + 2 core problems | 2 hrs |
| Sat | Pattern 04: 3 stretch + LeetCode Weekly Contest | 4 hrs |
| Sun | Pattern 04 sign-off + Contest review | 3 hrs |

### Week 3: Binary Search, Prefix Sums
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 05 doc study + 2 warmup problems | 2 hrs |
| Tue | Pattern 05: 3 core problems (timed, focus on BS on answer) | 2 hrs |
| Wed | Pattern 05: 3 stretch problems (this pattern is critical) | 2 hrs |
| Thu | Pattern 05 sign-off + Pattern 06 doc study | 2 hrs |
| Fri | Pattern 06: 2 warmup + 3 core problems | 2 hrs |
| Sat | Pattern 06: stretch + LeetCode Weekly Contest | 4 hrs |
| Sun | Pattern 06 sign-off + Contest review + Codeforces Virtual | 4 hrs |

### Week 4: Linked Lists, Trees DFS (start)
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 07 doc study + 3 problems (linked lists go fast) | 2 hrs |
| Tue | Pattern 07: remaining problems + sign-off | 2 hrs |
| Wed | Pattern 08 doc study + 2 warmup (Trees DFS) | 2 hrs |
| Thu | Pattern 08: 3 core problems | 2 hrs |
| Fri | Pattern 08: 2 stretch problems | 2 hrs |
| Sat | Pattern 08 sign-off + LeetCode Weekly Contest | 4 hrs |
| Sun | Contest review + revision of weak spots from Weeks 1-3 | 3 hrs |

---

## MONTH 2 — CORE STRUCTURES + DP START (Weeks 5-8)

### Week 5: Trees BFS, Heap, Intervals
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 09 (Trees BFS) — full set (it's shorter) | 2 hrs |
| Tue | Pattern 09 sign-off + Pattern 10 doc (Heap) | 2 hrs |
| Wed | Pattern 10: 4 problems | 2 hrs |
| Thu | Pattern 10: remaining + sign-off | 2 hrs |
| Fri | Pattern 11 (Intervals): full set | 2 hrs |
| Sat | Pattern 11 sign-off + LeetCode Weekly Contest | 4 hrs |
| Sun | Contest review + Codeforces Virtual | 3 hrs |

### Week 6: Graphs BFS/DFS, Union Find
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 12 doc study + 2 warmup (Graphs) | 2 hrs |
| Tue | Pattern 12: 4 core problems | 2 hrs |
| Wed | Pattern 12: 3 stretch + sign-off | 2 hrs |
| Thu | Pattern 13 (Union Find) doc + 3 problems | 2 hrs |
| Fri | Pattern 13: remaining + sign-off | 2 hrs |
| Sat | Pattern 14 (Shortest Path) doc + 3 problems + Contest | 4 hrs |
| Sun | Pattern 14: remaining + Contest review | 3 hrs |

### Week 7: Backtracking, DP Fundamentals
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 14 sign-off + Pattern 15 doc (Backtracking) | 2 hrs |
| Tue | Pattern 15: 4 problems | 2 hrs |
| Wed | Pattern 15: remaining + sign-off | 2 hrs |
| Thu | Pattern 16 (DP Fundamentals) — STUDY DEEPLY | 2 hrs |
| Fri | Pattern 16: practice problems + mental model | 2 hrs |
| Sat | Pattern 17 (1D DP) doc + 3 problems + Contest | 4 hrs |
| Sun | Pattern 17: 3 more problems + Contest review | 3 hrs |

### Week 8: 1D DP complete, 2D DP start
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 17: remaining + sign-off | 2 hrs |
| Tue | Pattern 18 (2D DP) doc + 3 problems | 2 hrs |
| Wed | Pattern 18: 3 problems | 2 hrs |
| Thu | Pattern 18: remaining + sign-off | 2 hrs |
| Fri | REVISION DAY — redo failed problems from Weeks 5-7 | 2 hrs |
| Sat | LeetCode Weekly + Biweekly Contests | 4 hrs |
| Sun | Contest review + full progress assessment | 3 hrs |

---

## MONTH 3 — DP MASTERY (Weeks 9-12)

### Week 9: Knapsack, LCS
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 19 (Knapsack) doc — STUDY THE DISGUISES | 2 hrs |
| Tue | Pattern 19: 4 problems | 2 hrs |
| Wed | Pattern 19: remaining + sign-off | 2 hrs |
| Thu | Pattern 20 (LCS) doc + 3 problems | 2 hrs |
| Fri | Pattern 20: 3 problems | 2 hrs |
| Sat | Pattern 20: remaining + Contest | 4 hrs |
| Sun | Pattern 20 sign-off + Contest review | 3 hrs |

### Week 10: LIS, DP on Trees
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 21 (LIS) doc + O(n²) and O(n log n) implementation | 2 hrs |
| Tue | Pattern 21: problems + sign-off | 2 hrs |
| Wed | Pattern 22 (DP on Trees) doc + 3 problems | 2 hrs |
| Thu | Pattern 22: remaining | 2 hrs |
| Fri | Pattern 22 sign-off | 2 hrs |
| Sat | Contest + Codeforces Virtual | 4 hrs |
| Sun | Contest review + DP revision (weak patterns) | 3 hrs |

### Week 11: Interval DP, Bitmask DP
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 23 (Interval DP) doc — this is HARD, take time | 2 hrs |
| Tue | Pattern 23: 3 problems (may need full 40 min each) | 2 hrs |
| Wed | Pattern 23: remaining + sign-off | 2 hrs |
| Thu | Pattern 24 (Bitmask DP) doc + 2 problems | 2 hrs |
| Fri | Pattern 24: 3 problems | 2 hrs |
| Sat | Pattern 24: remaining + Contest | 4 hrs |
| Sun | Pattern 24 sign-off + Contest review | 3 hrs |

### Week 12: Digit DP + DP BUFFER WEEK
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 25 (Digit DP) doc | 2 hrs |
| Tue | Pattern 25: 3 problems | 2 hrs |
| Wed | Pattern 25: remaining | 2 hrs |
| Thu | DP REVISION — redo hardest problem from each DP pattern | 2 hrs |
| Fri | DP REVISION — sign-off retries if any failed | 2 hrs |
| Sat | Contest + Virtual Contest | 4 hrs |
| Sun | Full DP assessment + Contest review | 3 hrs |

---

## MONTH 4 — ADVANCED + INTERVIEW PREP (Weeks 13-16)

### Week 13: Trie, Segment Tree, Fenwick
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 26 (Trie) doc + problems | 2 hrs |
| Tue | Pattern 26 sign-off + Pattern 27 (Segment Tree) doc | 2 hrs |
| Wed | Pattern 27: implement from scratch + 2 problems | 2 hrs |
| Thu | Pattern 27: lazy propagation + 2 problems | 2 hrs |
| Fri | Pattern 28 (Fenwick) + 3 problems | 2 hrs |
| Sat | Patterns 27-28 sign-offs + Contest | 4 hrs |
| Sun | Contest review + advanced problem practice | 3 hrs |

### Week 14: Monotonic Deque, Topo Sort, MST, SCC, Strings
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 29 (Monotonic Deque) + Pattern 30 (Topo Sort) | 2 hrs |
| Tue | Pattern 30: problems + sign-off | 2 hrs |
| Wed | Pattern 31 (MST) + Pattern 32 (SCC) | 2 hrs |
| Thu | Pattern 33 (String Algorithms) doc + KMP implementation | 2 hrs |
| Fri | Pattern 33: problems + Pattern 34 (Greedy) doc | 2 hrs |
| Sat | Pattern 34-35 problems + Contest | 4 hrs |
| Sun | All advanced sign-offs + Contest review | 3 hrs |

### Week 15: Interview Masterclass
| Day | Activity | Time |
|-----|----------|------|
| Mon | Pattern 36 (Interview Framework) — study and practice | 2 hrs |
| Tue | Pattern 37 (Company-Specific) — make company target list | 2 hrs |
| Wed | Pattern 38 (Hard Problem Decomposition) — 3 hard problems | 2 hrs |
| Thu | Mock interview simulation #1 (45 min, 2 problems) | 2 hrs |
| Fri | Mock interview simulation #2 | 2 hrs |
| Sat | Mock interview #3 + Contest | 4 hrs |
| Sun | Full review + Contest analysis | 3 hrs |

### Week 16: FINAL REVISION + CONTEST GRIND
| Day | Activity | Time |
|-----|----------|------|
| Mon | Weak pattern revision (based on contest performance) | 2 hrs |
| Tue | REVISION — hardest problem from each FOUNDATION pattern | 2 hrs |
| Wed | REVISION — hardest problem from each CORE pattern | 2 hrs |
| Thu | REVISION — hardest problem from each DP pattern | 2 hrs |
| Fri | Mock interview simulation #4 | 2 hrs |
| Sat | Final Contest + Virtual Contest | 4 hrs |
| Sun | Comprehensive review + next steps planning | 3 hrs |

---

---

# INTERVIEW READINESS MAP

---

## AMAZON (SDE-1/SDE-2)

Amazon asks the MOST predictable questions. They love:

| Priority | Pattern | Problems They Ask |
|----------|---------|-------------------|
| 1 | Arrays + Hashing | Two Sum, Product of Array Except Self |
| 2 | Trees DFS/BFS | LCA, Validate BST, Level Order |
| 3 | Graphs BFS/DFS | Number of Islands, Rotting Oranges |
| 4 | Sliding Window | Minimum Window Substring |
| 5 | Heap | Merge K Sorted, Top K Frequent |
| 6 | DP (1D/2D) | Word Break, LIS |
| 7 | Linked Lists | LRU Cache, Reverse Linked List |
| 8 | Greedy | Task Scheduler, Meeting Rooms |

**Amazon prep in 1 week:** Focus on Patterns 01, 08, 12, 03, 10 (in that order)

---

## GOOGLE (L3/L4/L5)

Google asks the HARDEST questions. They love:

| Priority | Pattern | Problems They Ask |
|----------|---------|-------------------|
| 1 | DP (all types) | Edit Distance, Burst Balloons, LIS |
| 2 | Graphs (advanced) | Shortest Path, Topo Sort, SCC |
| 3 | Binary Search | BS on Answer, Median of Two Arrays |
| 4 | Trees DFS | Max Path Sum, Serialize/Deserialize |
| 5 | Math/Number Theory | Modular arithmetic, combinatorics |
| 6 | String Algorithms | KMP, Rolling Hash |
| 7 | Segment Tree/BIT | Range queries (for L5+) |
| 8 | Backtracking | Word Search II, N-Queens |

**Google prep in 1 week:** Focus on Patterns 16-23 (DP), 14, 05, 08

---

## MICROSOFT (SDE/SDE-2)

Microsoft asks medium difficulty, clean code focused:

| Priority | Pattern | Problems They Ask |
|----------|---------|-------------------|
| 1 | Trees (all) | BST operations, LCA, serialize |
| 2 | Linked Lists | Reverse, merge, cycle detect |
| 3 | Arrays + Two Pointers | 3Sum, Container With Water |
| 4 | Stack | Valid Parentheses, Calculator |
| 5 | Graphs BFS | Shortest Path, Clone Graph |
| 6 | DP (1D/2D) | House Robber, Unique Paths |
| 7 | Binary Search | Search in Rotated, Peak Element |
| 8 | Intervals | Merge Intervals, Meeting Rooms |

**Microsoft prep in 1 week:** Focus on Patterns 07, 08, 09, 02, 04

---

## INDIAN UNICORNS (Zepto, Swiggy, Razorpay, Meesho, PhonePe, CRED)

These companies ask LeetCode Medium consistently. They love:

| Priority | Pattern | Problems They Ask |
|----------|---------|-------------------|
| 1 | Arrays + Hashing | Subarray Sum, Longest Consecutive |
| 2 | Sliding Window | Longest Substring, Minimum Window |
| 3 | Binary Search | Search Rotated, BS on Answer |
| 4 | Stack | Next Greater, Largest Rectangle |
| 5 | Two Pointers | 3Sum, Trapping Rain Water |
| 6 | Trees DFS | Diameter, LCA, Validate BST |
| 7 | DP (1D) | House Robber, Coin Change, LIS |
| 8 | Graphs | Number of Islands, Course Schedule |

**Indian unicorn prep in 1 week:** Focus on Patterns 01, 03, 05, 04, 02

---

## CODEFORCES DIV 2 D/E (What 2000+ rated coders need)

| Priority | Pattern | Why It Appears |
|----------|---------|----------------|
| 1 | Binary Search on Answer | 30% of Div 2 D problems |
| 2 | DP (Bitmask, Interval, Digit) | Complex state DP |
| 3 | Segment Tree / BIT | Range query optimization |
| 4 | Math + Number Theory | Modular arithmetic, combinatorics |
| 5 | Greedy with Proof | Exchange argument problems |
| 6 | Graphs (Topo Sort, MST, SCC) | Graph theory heavy |
| 7 | String Algorithms | KMP, Z-function, Hashing |
| 8 | Constructive Algorithms | Build answer rather than find |

---

---

# CONTEST SCHEDULE

---

## When Contests Start: WEEK 1 (yes, immediately)

You do NOT wait until you are "ready." You start contesting NOW.
Your first contests will be humbling. That is the point.
Contest rating is the only HONEST measure of your skill.

## Contest Cadence:

| Week | Contests |
|------|----------|
| 1-4 | LeetCode Weekly (Sat) + 1 CF Virtual Div 3 (Wed) |
| 5-8 | LeetCode Weekly (Sat) + Biweekly (Sun when available) + 1 CF Virtual Div 2 (Wed) |
| 9-12 | LeetCode Weekly + Biweekly + 2 CF Virtual Div 2 (Mon, Wed) |
| 13-16 | LeetCode Weekly + Biweekly + 2 CF Virtual Div 2 + 1 live CF round |

## Post-Contest Protocol (MANDATORY — do within 24 hours):

1. Paste problems solved, time taken, problems failed
2. I do full analysis:
   - Was your approach optimal?
   - What pattern was the failed problem?
   - What should you have seen?
   - 3 targeted problems to fix the gap
3. Update CONTEST-LOG.md
4. Add failed problem patterns to REVISION-TRACKER.md

---

---

# REVISION SYSTEM — NEVER FORGET

---

## The Spaced Repetition Schedule:

Every problem you struggle with goes into REVISION-TRACKER.md with:
- Problem name and LC number
- Pattern it belongs to
- What you got wrong
- Date first solved
- Revision dates: Day+3, Day+7, Day+14, Day+30

## Weekly Revision Sessions (built into schedule):

| Day | Revision Activity |
|-----|-------------------|
| Every Monday | Re-solve 2 problems from 2 weeks ago |
| Every Friday | Re-solve 1 problem from 1 month ago |
| Every 4th Sunday | Full revision: 1 problem from EACH completed pattern |

## The "Challenge Problems" List:

After completing each pattern, mark the 2-3 hardest problems as "CHALLENGE."
These go into a separate section in REVISION-TRACKER.md.
You re-solve EVERY challenge problem at:
- 1 week after completion
- 2 weeks after completion
- 1 month after completion
- Before any interview

---

---

# ADDITIONAL REFERENCES AND RESOURCES

---

## Books (ranked by priority):
1. **Competitive Programmer's Handbook** (Laaksonen) — FREE PDF, covers everything for CP
2. **Algorithm Design Manual** (Skiena) — best for interview prep algorithm intuition
3. **Introduction to Algorithms** (CLRS) — reference only, don't read cover to cover
4. **CP-algorithms website** (cp-algorithms.com) — THE reference for every algorithm implementation

## Video Resources:
1. **Errichto** (YouTube) — binary search, DP, thinking process
2. **William Fiset** (YouTube) — graph algorithms, data structures
3. **NeetCode** (YouTube) — interview-focused, pattern-organized
4. **Aditya Verma** (YouTube) — DP playlist (Hindi), excellent for knapsack/LCS/MCM
5. **Striver (take U forward)** (YouTube) — graphs, trees, DP in C++

## Practice Platforms:
1. **LeetCode** — primary for interviews, contests every week
2. **Codeforces** — primary for CP rating, problems are harder and more creative
3. **AtCoder** — excellent for DP and math problems
4. **USACO Guide** — structured learning path for advanced CP topics

---

---

# SUMMARY — THE 4-MONTH NUMBERS

| Metric | Target |
|--------|--------|
| Total problems solved | 380-420 new problems |
| Problems per day (weekday) | 2-3 problems |
| Problems per day (weekend) | 4-5 problems |
| Patterns mastered | All 35 algorithmic patterns |
| Contests completed | 30+ contests |
| LeetCode rating target | 2000+ |
| Codeforces rating target | 1800-1900 (if consistent) |
| Interview readiness | Any FAANG/unicorn backend DSA round |

---

---

# COMMANDS REMINDER

| Command | What It Does |
|---------|--------------|
| START | Begin Pattern 01, generate full in-depth doc |
| NEXT | Move to next pattern (only after sign-off) |
| SOLVE [problem] | Paste solution for 5-dimension review |
| CONTEST REVIEW | Paste contest results for breakdown |
| VIRTUAL CONTEST | Timed 4-problem simulation |
| WEAK SPOT DRILL | 5 problems targeting weak areas |
| SIGN OFF | Attempt current pattern sign-off |
| HINT | One nudge (not the answer) |
| PROGRESS | Full checklist with status |
| REDO [pattern] | Regenerate pattern doc |
| COMPLEXITY CHECK | State complexity, I verify |
| INTERVIEW SIM | 45-minute mock interview |

---

---

## Does this plan look good? Type START to begin Pattern 01, or tell me what to change.
