# Pattern 07 — Linked Lists (Complete)
## Difficulty Level: CORE
## Interview Frequency: OFTEN (especially Amazon, Microsoft)
## Estimated time to master: 3-4 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Linked lists are the pattern that separates people who "understand pointers" from people who "think they understand pointers."

The algorithm itself is never the hard part. Reverse a linked list — most people know the idea in 10 seconds. But then they write the code, it passes 3 test cases, and then their program crashes on a two-element list. Or they lose the `next` pointer mid-swap and create a cycle they can't unwind. Or their beautiful iterative reversal works perfectly and then they try to use it in a larger problem and it silently corrupts the list because they didn't handle the tail correctly.

**The real challenge of linked lists is pointer discipline.** Every pointer manipulation has an order — swap in the wrong order and your data is gone forever. Unlike arrays, there's no "undo." If you lose a pointer to a node, that entire tail of the list is gone, leaked in memory forever.

**What to internalize before writing a single line:**
1. **Draw the state before and after.** Always. Even if you think you "see" it. Draw arrows on paper showing what each pointer points to before the operation, then draw the target state. Your code is just mechanically achieving that drawing.
2. **The dummy node is your best friend.** Any time a linked list operation might modify the head (deletions, insertions at front, merges), use a dummy node. `ListNode* dummy = new ListNode(0); dummy->next = head;`. Your answer is `dummy->next`. This eliminates 90% of edge case bugs.
3. **Fast-slow pointers are a single technique with three applications.** Detect a cycle. Find the cycle's start. Find the middle of the list. All three use the same two-pointer mechanics — understand one, you understand all three.

**At your 1600 level, you:**
- Can reverse a linked list (probably)
- Cannot reliably reverse in k-groups under interview pressure
- Forget to use dummy nodes and fight edge cases for 10 minutes
- Cannot explain the Floyd cycle detection PROOF — you just "know it works"
- Make off-by-one errors on "remove Nth from end" consistently

**After this document, you will:**
- Have a pointer manipulation checklist that eliminates crashes
- Understand Floyd's cycle detection completely — where the two pointers meet and WHY
- Use dummy nodes automatically without thinking
- Implement LRU Cache (doubly linked list + hash map) — a problem asked in literally every FAANG loop

The interview data is clear: Amazon asks linked list problems in 40%+ of SDE-2 rounds. Microsoft asks them in 50%+ of rounds. If you can't do these cleanly and fast, you are filtering yourself out.

---

## ELI5 — THE POINTER DISCIPLINE ANALOGY

### Singly Linked List as a Treasure Hunt

Imagine a treasure hunt where each clue has a note that says "the next clue is in the red mailbox on Oak Street." You hold the first clue. To get to clue 5, you must visit clue 1, read where clue 2 is, go there, read where clue 3 is, and so on. There is no shortcut. There is no "jump to index 4."

This is a singly linked list. The only way to reach node 5 is to start at the head and follow `.next` five times. If you're standing at clue 3 and you drop clue 4's address, the rest of the treasure hunt is lost forever — you've orphaned nodes.

**This is why pointer discipline matters.** Before you reassign any pointer, you must save what it was pointing to.

### The Reversal as Rerouting Arrows

Before reversal: `A → B → C → D → null`
After reversal: `null ← A ← B ← C ← D`

Each arrow needs to be turned around. You need three things to turn one arrow around:
1. `prev` — the node you just reversed to (initially `null`)
2. `curr` — the node whose arrow you're currently reversing
3. `next` — the node curr was pointing to (save this BEFORE you break the arrow)

The order is always: save `next`, then reverse `curr`'s arrow, then advance `prev` and `curr`. Get the order wrong and you lose `next` forever.

### The Dummy Node as a Sentinel Guard

Imagine you're managing a line of people. Sometimes you need to remove the person at the front, or add someone at the front, or merge two lines. Handling "what if the front position changes" is always a special case.

The dummy node trick is to put a fake bouncer at position 0. This bouncer never leaves. Now the "front person" is always `dummy->next`. You never have to handle "what if head changes" because the dummy node absorbs all that complexity. After you're done, the answer is `dummy->next` — the first real person in line.

### Fast-Slow Pointers as the Tortoise and Hare

Think of a circular running track. Two runners start at the same point. Tortoise runs 1 step at a time. Hare runs 2 steps at a time. If the track is a straight line, hare reaches the end first — easy. If the track is a loop, **they will always meet.** The hare laps the tortoise, and they'll be at the same point eventually.

This is Floyd's algorithm. If there's a cycle in the list, hare will catch tortoise. If there's no cycle, hare reaches `null` first and we know the list is acyclic.

**Why do they meet exactly at the cycle start** (when you reset slow to head)? That's the mathematical proof — and we'll cover it completely in the Templates section.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:

**Traversal and Search:**
1. Find the Kth node from the end (two pointer gap technique)
2. Find the middle of a linked list (fast-slow pointer)
3. Detect if a linked list has a cycle
4. Find the start node of a cycle (Floyd cycle detection extension)

**In-Place Modification:**
1. Reverse a linked list (iterative and recursive)
2. Reverse k groups at a time
3. Remove a node (from Nth position from end, remove duplicates)
4. Rotate the list by K positions

**Construction:**
1. Merge two sorted linked lists
2. Merge K sorted linked lists
3. Deep copy a linked list with random pointers
4. Add two numbers represented as linked lists

**Design:**
1. LRU Cache (doubly linked list + hash map — the hardest linked list problem)
2. Implement queue/stack with linked list

### The invariant that makes linked list problems tractable:

At every step of a linked list operation, you must be able to answer: "which pointers do I currently hold, what do they point to, and will I lose access to any node if I reassign one of them?"

This invariant — tracking your "handles" to the list — is what distinguishes clean code from code that crashes on edge cases.

---

## WHEN TO RECOGNIZE THIS PATTERN

**Signal 1:** The problem explicitly uses a linked list (`ListNode*` structure)
→ Directly apply the appropriate technique. The key question to ask first: "Does the head node potentially change?" If yes — dummy node. "Do I need to find something relative to the end?" If yes — two-pointer gap or reverse approach. "Do I need to find the middle?" If yes — fast-slow pointer.

**Signal 2:** "Reverse the list" or "reverse a portion of the list" or "reverse in k-groups"
→ In-place reversal template. The standard three-pointer iterative reversal: `prev = null, curr = head`, then loop: save `next = curr->next`, redirect `curr->next = prev`, advance `prev = curr, curr = next`. This template handles all reversal variants.

**Signal 3:** "Does the list have a cycle?" or "find the start of the cycle"
→ Floyd's cycle detection. If fast and slow pointers meet → cycle exists. To find the start: reset one pointer to head, keep other at meeting point, advance both by 1 step — they meet at cycle start. The mathematical proof is below.

**Signal 4:** "Find the Kth node from the end" or "remove the Nth node from the end"
→ Two-pointer gap technique. Advance the fast pointer K steps ahead of the slow pointer. When fast reaches `null`, slow is at the Kth node from the end. To remove it, stop when fast is at the LAST node (not null) — then slow is at the node BEFORE the target.

**Signal 5:** "Merge two sorted lists" or "merge K sorted lists"
→ Dummy node + compare heads. For two lists: dummy node, always attach the smaller head, advance that list's pointer. For K lists: use a min-heap of `(value, ListNode*)` — extract min, attach to result, push the next node from that list into the heap.

**Signal 6:** "Copy the list with random pointers" or "deep copy"
→ Two approaches: (1) hash map — `map[original] = new copy node`, two passes; (2) interleaving — weave copies between originals, set random pointers, then unweave. The interleaving approach is O(1) space.

**Signal 7:** "Design an LRU cache" or "implement a cache with eviction policy"
→ Doubly linked list (for O(1) removal from any position) + hash map (for O(1) lookup). The most recently used node is the head; evict from the tail. This combination appears in 30%+ of Amazon onsite rounds.

**Signal 8:** "Reorder list" or "rearrange nodes" — first half forward, second half backward, interleaved
→ Three-step technique: (1) find middle with fast-slow pointer, (2) reverse second half, (3) merge two halves alternately. This exact decomposition applies to several similar problems.

### Signals that LOOK like linked lists but are NOT:

**Fake signal 1:** "Reverse a sequence of numbers"
→ Just use a `vector<int>` and `reverse()` or two-pointer swap. Only use a linked list when the input is actually a `ListNode*`.

**Fake signal 2:** "Find the Kth largest element in a sequence"
→ Quickselect or heap. Not a linked list problem — linked lists don't support random access so Kth largest is awkward.

**Fake signal 3:** "Design a data structure with O(1) access by index"
→ Array or hash map. Linked lists are O(n) for index access by design. If O(1) index access is needed, the data structure is not a linked list.

---

## THE TEMPLATES (C++)

### Template 1: In-Place Reversal — Iterative (The Foundation)

```cpp
// Use when: reverse the ENTIRE list
// Returns the new head (which was the old tail)
// Time: O(n), Space: O(1)

ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;   // Initially, nothing before head
    ListNode* curr = head;      // Start at the first node
    
    while (curr != nullptr) {
        ListNode* next = curr->next;  // STEP 1: SAVE next (NEVER skip this)
        curr->next = prev;            // STEP 2: reverse the arrow
        prev = curr;                  // STEP 3: advance prev
        curr = next;                  // STEP 4: advance curr
    }
    
    return prev;  // curr is now null; prev is the new head (old tail)
}

// WHY prev = nullptr at the start:
// After reversing, the OLD head becomes the new TAIL.
// The new tail must point to null. prev starts as null so the
// first node (old head) will correctly have its next set to null.

// TRACE for [1 -> 2 -> 3 -> null]:
// Init:    prev=null, curr=1
// Step 1:  next=2, 1->null, prev=1, curr=2
// Step 2:  next=3, 2->1,    prev=2, curr=3
// Step 3:  next=null, 3->2, prev=3, curr=null
// Loop ends. Return prev = 3.
// Result:  3 -> 2 -> 1 -> null ✓
```

### Template 2: In-Place Reversal — Recursive

```cpp
// Use when: you prefer recursive thinking or the problem has recursive structure
// WARNING: O(n) stack space — for very long lists, iterative is safer

ListNode* reverseList(ListNode* head) {
    // Base case: empty or single node
    if (head == nullptr || head->next == nullptr) return head;
    
    // Recursive step: reverse the rest of the list
    ListNode* newHead = reverseList(head->next);
    
    // head->next still points to the last node of the reversed sublist
    // Make that node point back to head
    head->next->next = head;
    head->next = nullptr;  // head becomes the new tail — must point to null
    
    return newHead;  // new head doesn't change as we unwind the recursion
}

// TRACE for [1 -> 2 -> 3 -> null]:
// reverseList(1): calls reverseList(2)
//   reverseList(2): calls reverseList(3)
//     reverseList(3): base case, returns 3
//   Back in reverseList(2): newHead=3, 2->next=3 (still), 3->next=2, 2->next=null
//   Returns 3
// Back in reverseList(1): newHead=3, 1->next=2 (still), 2->next=1, 1->next=null
// Returns 3
// Final: 3 -> 2 -> 1 -> null ✓
```

### Template 3: Fast-Slow Pointer — Detect Cycle (Floyd's)

```cpp
// Use when: determine if a linked list has a cycle
// Time: O(n), Space: O(1)

bool hasCycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;          // moves 1 step
        fast = fast->next->next;    // moves 2 steps
        
        if (slow == fast) return true;  // they met — cycle detected
    }
    
    return false;  // fast reached null — no cycle
}

// WHY fast != nullptr && fast->next != nullptr?
// fast->next->next requires BOTH fast AND fast->next to be non-null.
// Check fast first (short-circuit), then fast->next.
// If either is null, we've reached the end of an acyclic list.

// WHY do they always meet in a cycle?
// Let cycle length = C. When slow enters the cycle, fast is already somewhere in it.
// The gap between them changes by 1 each step (fast gains 1 on slow per step,
// since fast moves 2 and slow moves 1 within the cycle).
// Since the gap decreases by 1 per step and the cycle has finite length C,
// they MUST meet within at most C steps after slow enters the cycle.
```

### Template 4: Fast-Slow Pointer — Find Cycle Start (Floyd's Extended)

```cpp
// Use when: find the NODE where the cycle begins
// Prerequisite: a cycle exists (use Template 3 first, or combine)

ListNode* detectCycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    // Phase 1: detect the cycle
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;  // meeting point found
    }
    
    // No cycle
    if (fast == nullptr || fast->next == nullptr) return nullptr;
    
    // Phase 2: find cycle start
    slow = head;  // reset slow to head
    // fast stays at the meeting point
    
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next;  // now BOTH move 1 step at a time
    }
    
    return slow;  // they meet exactly at the cycle start
}

// ===== THE MATHEMATICAL PROOF =====
// Let:
//   F = distance from head to cycle start (in nodes)
//   C = cycle length
//   k = distance from cycle start to meeting point (inside the cycle)
//
// When slow and fast meet:
//   slow traveled: F + k
//   fast traveled: F + k + m*C  (fast lapped slow m times, m >= 1)
//   fast = 2 * slow (fast moves twice as fast)
//
// So: F + k + m*C = 2*(F + k)
//     m*C = F + k
//     F = m*C - k
//
// Now in Phase 2:
//   slow starts at head (distance 0)
//   fast starts at meeting point (which is k steps INTO the cycle)
//
//   After F steps:
//   slow is at: head + F = cycle start
//   fast is at: meeting point + F = (k steps in cycle) + F
//             = k + m*C - k = m*C from cycle start = cycle start (full laps!)
//
// They BOTH arrive at the cycle start after exactly F steps. QED.
```

### Template 5: Fast-Slow Pointer — Find Middle of List

```cpp
// Use when: find the middle node
// For even-length: returns the SECOND middle node (right middle)
// For odd-length: returns the exact middle

ListNode* findMiddle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    
    while (fast != nullptr && fast->next != nullptr) {
        slow = slow->next;
        fast = fast->next->next;
    }
    
    return slow;  // slow is at the middle
}

// For [1,2,3,4,5]: fast reaches 5, slow is at 3 (middle)
// For [1,2,3,4]:   fast reaches null (after 4), slow is at 3 (second middle)

// VARIANT: find the FIRST middle for even-length lists
// Use: while (fast->next != nullptr && fast->next->next != nullptr)
// For [1,2,3,4]: fast stops at 3 (last non-null double step), slow is at 2 (first middle)
// Use this variant when you need to split the list INTO two halves cleanly
// (so you can set slow->next = null to terminate the first half)
```

### Template 6: Two-Pointer Gap — Remove Nth From End

```cpp
// Use when: find/remove the Nth node from the end in ONE PASS
// Uses a dummy node to handle removal of the head (N = length of list)

ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode* dummy = new ListNode(0);
    dummy->next = head;
    
    ListNode* fast = dummy;
    ListNode* slow = dummy;
    
    // Advance fast by n+1 steps so the gap between fast and slow is n
    for (int i = 0; i <= n; i++) fast = fast->next;
    
    // Move both until fast reaches null
    while (fast != nullptr) {
        slow = slow->next;
        fast = fast->next;
    }
    
    // slow is now at the node BEFORE the target
    ListNode* toDelete = slow->next;
    slow->next = slow->next->next;
    delete toDelete;
    
    ListNode* result = dummy->next;
    delete dummy;
    return result;
}

// WHY n+1 steps (not n)?
// We want slow to stop at the node BEFORE the Nth-from-end (to allow deletion).
// If we advance fast by n, slow ends up AT the Nth-from-end node (for traversal only).
// If we advance fast by n+1, slow ends up ONE BEFORE the Nth-from-end (for deletion). 

// WHY dummy node?
// If N equals the length of the list, we're removing the head.
// Without dummy: slow would be at null before the head — invalid pointer.
// With dummy: slow would be at the dummy node, and slow->next = head (correct target).
```

### Template 7: Dummy Node — Merge Two Sorted Lists

```cpp
// Use when: merge two sorted lists into one sorted list
// Dummy node eliminates the "which list's head is smaller" special case

ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    ListNode* dummy = new ListNode(0);
    ListNode* curr = dummy;
    
    while (l1 != nullptr && l2 != nullptr) {
        if (l1->val <= l2->val) {
            curr->next = l1;
            l1 = l1->next;
        } else {
            curr->next = l2;
            l2 = l2->next;
        }
        curr = curr->next;
    }
    
    // Attach the remaining non-empty list
    curr->next = (l1 != nullptr) ? l1 : l2;
    
    ListNode* result = dummy->next;
    delete dummy;
    return result;
}

// HOW TO ADAPT:
// - For merge sort on linked lists: use this as the merge step
// - For sorted insert: treat the new node as a single-element list and merge
// - The <= ensures stability (equal elements from l1 come before l2)
```

### Template 8: Reverse in K-Groups (The Hard Reversal)

```cpp
// LC 25: Reverse Nodes in k-Group
// Reverses every k consecutive nodes. If remaining < k, leave them as-is.

class Solution {
    // Helper: reverse nodes from head through k nodes (if k nodes exist)
    // Returns {new_head, new_tail, next_group_head}
    // If fewer than k nodes remain, returns {head, ..., ...} signaling no reversal
    
    ListNode* getKthNode(ListNode* curr, int k) {
        while (curr != nullptr && k > 0) {
            curr = curr->next;
            k--;
        }
        return curr;  // returns the (k+1)th node, or null if fewer than k remain
    }
    
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        ListNode* dummy = new ListNode(0);
        dummy->next = head;
        ListNode* groupPrev = dummy;  // the node before the current group
        
        while (true) {
            // Check if k nodes remain starting from groupPrev->next
            ListNode* kth = getKthNode(groupPrev->next, k - 1);
            if (kth == nullptr) break;  // fewer than k nodes remain, leave them
            
            ListNode* groupNext = kth->next;  // save the start of the NEXT group
            
            // Reverse the group [groupPrev->next ... kth]
            ListNode* prev = groupNext;  // tail of this group should point to next group
            ListNode* curr = groupPrev->next;
            
            while (curr != groupNext) {
                ListNode* next = curr->next;
                curr->next = prev;
                prev = curr;
                curr = next;
            }
            // Now prev = kth (the new head of this reversed group)
            // The old head (groupPrev->next before reversal) is now the new tail
            
            ListNode* oldGroupHead = groupPrev->next;
            groupPrev->next = kth;       // connect previous section to new group head
            groupPrev = oldGroupHead;    // advance groupPrev to the new tail of this group
        }
        
        ListNode* result = dummy->next;
        delete dummy;
        return result;
    }
};

// TRACE for [1,2,3,4,5], k=2:
// Group 1: groupPrev=dummy, kth=node(2), groupNext=node(3)
//   Reverse [1,2]: 2->1->3..., groupPrev moves to node(1)
// Group 2: groupPrev=node(1), kth=node(4), groupNext=node(5)
//   Reverse [3,4]: 4->3->5, groupPrev moves to node(3)
// Group 3: fewer than 2 nodes (only node(5)), break.
// Result: [2,1,4,3,5] ✓
```

### Template 9: Deep Copy With Random Pointers — Interleaving Method

```cpp
// LC 138: Two methods — hash map (O(n) space) and interleaving (O(1) space)

// METHOD 1: Hash Map (simpler, O(n) space)
Node* copyRandomList_HashMap(Node* head) {
    if (!head) return nullptr;
    
    unordered_map<Node*, Node*> map;  // original → copy
    
    // Pass 1: create all copies (without setting next/random yet)
    Node* curr = head;
    while (curr) {
        map[curr] = new Node(curr->val);
        curr = curr->next;
    }
    
    // Pass 2: set next and random pointers using the map
    curr = head;
    while (curr) {
        if (curr->next)   map[curr]->next   = map[curr->next];
        if (curr->random) map[curr]->random = map[curr->random];
        curr = curr->next;
    }
    
    return map[head];
}

// METHOD 2: Interleaving (O(1) extra space, O(n) total)
// Step 1: Weave copies: A -> A' -> B -> B' -> C -> C'
// Step 2: Set random: A'->random = A->random->next (the copy of A's random)
// Step 3: Unweave: separate original and copy lists

Node* copyRandomList(Node* head) {
    if (!head) return nullptr;
    
    // Step 1: Weave
    Node* curr = head;
    while (curr) {
        Node* copy = new Node(curr->val);
        copy->next = curr->next;
        curr->next = copy;
        curr = copy->next;  // advance to the original next node
    }
    
    // Step 2: Set random pointers for copies
    curr = head;
    while (curr) {
        if (curr->random)
            curr->next->random = curr->random->next;  // copy's random = copy of original's random
        curr = curr->next->next;  // skip over the copy to the next original
    }
    
    // Step 3: Unweave
    Node* dummy = new Node(0);
    Node* copyTail = dummy;
    curr = head;
    while (curr) {
        copyTail->next = curr->next;        // attach copy to copy list
        copyTail = copyTail->next;
        curr->next = curr->next->next;      // restore original's next
        curr = curr->next;                  // advance original
    }
    
    Node* result = dummy->next;
    delete dummy;
    return result;
}
```

### Template 10: LRU Cache — Doubly Linked List + Hash Map

```cpp
// LC 146: The canonical design problem. Both get() and put() must be O(1).
// Key insight: doubly linked list for O(1) removal from any position;
// hash map for O(1) lookup by key.
// Convention: head->next = most recently used, tail->prev = least recently used.

class LRUCache {
    struct Node {
        int key, val;
        Node *prev, *next;
        Node(int k, int v) : key(k), val(v), prev(nullptr), next(nullptr) {}
    };
    
    int capacity;
    unordered_map<int, Node*> map;  // key → node
    Node* head;  // dummy head (most recent end)
    Node* tail;  // dummy tail (LRU end)
    
    void remove(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }
    
    void insertAtFront(Node* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }
    
public:
    LRUCache(int capacity) : capacity(capacity) {
        head = new Node(0, 0);  // dummy head
        tail = new Node(0, 0);  // dummy tail
        head->next = tail;
        tail->prev = head;
    }
    
    int get(int key) {
        if (!map.count(key)) return -1;
        Node* node = map[key];
        remove(node);          // remove from current position
        insertAtFront(node);   // move to most recently used position
        return node->val;
    }
    
    void put(int key, int value) {
        if (map.count(key)) {
            Node* node = map[key];
            node->val = value;
            remove(node);
            insertAtFront(node);
        } else {
            if ((int)map.size() == capacity) {
                // Evict LRU: the node just before dummy tail
                Node* lru = tail->prev;
                remove(lru);
                map.erase(lru->key);
                delete lru;
            }
            Node* node = new Node(key, value);
            insertAtFront(node);
            map[key] = node;
        }
    }
};

// HOW TO ADAPT:
// LFU Cache (LC 460): replace doubly linked list with two maps: freq→bucket, key→freq.
// Each bucket is a doubly linked list (ordered by recency within same frequency).
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

### VARIANT 1: Reorder List (LC 143) — The Three-Step Decomposition

This problem is solved by combining three templates in sequence. This pattern of "decompose into known sub-problems" is what interviewers look for.

**Problem:** Given `1→2→3→4→5`, reorder to `1→5→2→4→3`.

```cpp
void reorderList(ListNode* head) {
    if (!head || !head->next) return;
    
    // Step 1: Find middle (use the "first middle" variant)
    ListNode* slow = head, *fast = head;
    while (fast->next && fast->next->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    // slow is now at the FIRST middle (for even-length, we want first half longer)
    
    // Step 2: Reverse the second half
    ListNode* secondHalf = slow->next;
    slow->next = nullptr;  // CUT the list at the midpoint
    
    ListNode* prev = nullptr, *curr = secondHalf;
    while (curr) {
        ListNode* next = curr->next;
        curr->next = prev;
        prev = curr;
        curr = next;
    }
    secondHalf = prev;  // secondHalf now points to the reversed second half
    
    // Step 3: Merge alternately
    ListNode* first = head, *second = secondHalf;
    while (second) {
        ListNode* firstNext = first->next;
        ListNode* secondNext = second->next;
        first->next = second;
        second->next = firstNext;
        first = firstNext;
        second = secondNext;
    }
}

// WHY cut the list (slow->next = nullptr)?
// Without cutting, the first half and second half share the original links.
// After reversing secondHalf in place, those original links are broken.
// Cutting ensures the first half is properly terminated.
```

---

### VARIANT 2: Add Two Numbers (LC 2) — Carry Propagation

Classic carry propagation on a linked list. The key insight: process digits simultaneously from the head (which represents the least significant digit), carry forward.

```cpp
ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
    ListNode* dummy = new ListNode(0);
    ListNode* curr = dummy;
    int carry = 0;
    
    while (l1 || l2 || carry) {
        int sum = carry;
        if (l1) { sum += l1->val; l1 = l1->next; }
        if (l2) { sum += l2->val; l2 = l2->next; }
        carry = sum / 10;
        curr->next = new ListNode(sum % 10);
        curr = curr->next;
    }
    
    ListNode* result = dummy->next;
    delete dummy;
    return result;
}

// The while condition `l1 || l2 || carry` handles:
// - Lists of different lengths (one runs out before the other)
// - A carry generated by the last digit pair (e.g., 99 + 1 = 100 needs an extra node)
```

---

### VARIANT 3: Merge K Sorted Lists (LC 23) — Min-Heap on List Heads

```cpp
// Three approaches:
// 1. Brute force: collect all values, sort, rebuild — O(n log n)
// 2. One-by-one merge: O(nk) — too slow
// 3. Min-heap: O(n log k) — optimal

ListNode* mergeKLists(vector<ListNode*>& lists) {
    // Min-heap: (node_value, list_index) — compare by value
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    
    // Initialize heap with the head of each non-empty list
    for (ListNode* head : lists) {
        if (head) pq.push(head);
    }
    
    ListNode* dummy = new ListNode(0);
    ListNode* curr = dummy;
    
    while (!pq.empty()) {
        ListNode* node = pq.top(); pq.pop();
        curr->next = node;
        curr = curr->next;
        if (node->next) pq.push(node->next);  // push the next node from that list
    }
    
    ListNode* result = dummy->next;
    delete dummy;
    return result;
}

// TIME COMPLEXITY: O(n log k) where n = total nodes, k = number of lists
// Each of the n nodes is pushed into the heap exactly once: O(log k) per push/pop.
// SPACE COMPLEXITY: O(k) for the heap (at most k elements at any time).
```

---

### VARIANT 4: Rotate List (LC 61) — Connect to Cycle, Find New Tail

```cpp
// Key insight: rotation by k is equivalent to making the list circular
// then breaking it at a new point.

ListNode* rotateRight(ListNode* head, int k) {
    if (!head || !head->next || k == 0) return head;
    
    // Step 1: Find length and tail
    int len = 1;
    ListNode* tail = head;
    while (tail->next) { tail = tail->next; len++; }
    
    // Step 2: Normalize k (k >= len means full rotations, wasteful)
    k = k % len;
    if (k == 0) return head;  // already in correct position
    
    // Step 3: Make the list circular
    tail->next = head;
    
    // Step 4: Find the new tail (len-k steps from old head)
    ListNode* newTail = head;
    for (int i = 0; i < len - k - 1; i++) newTail = newTail->next;
    
    // Step 5: Break the cycle
    ListNode* newHead = newTail->next;
    newTail->next = nullptr;
    
    return newHead;
}
```

---

### VARIANT 5: Palindrome Linked List (LC 234) — Find Middle + Reverse + Compare

```cpp
// Three steps: find middle, reverse second half, compare, (optionally restore)
bool isPalindrome(ListNode* head) {
    if (!head || !head->next) return true;
    
    // Find end of first half
    ListNode* slow = head, *fast = head;
    while (fast->next && fast->next->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    
    // Reverse second half
    ListNode* prev = nullptr, *curr = slow->next;
    while (curr) {
        ListNode* next = curr->next;
        curr->next = prev;
        prev = curr;
        curr = next;
    }
    
    // Compare both halves
    ListNode* p1 = head, *p2 = prev;
    bool result = true;
    while (p2) {  // second half is shorter or equal in length
        if (p1->val != p2->val) { result = false; break; }
        p1 = p1->next;
        p2 = p2->next;
    }
    
    return result;
    // Note: for interview cleanliness, restore the list (reverse back)
    // but most problems don't require it.
}
```

---

### VARIANT 6: Intersection of Two Linked Lists (LC 160) — Length Equalization

```cpp
// Key insight: if two lists intersect, align them by their ENDS (not starts).
// Advance the longer list's pointer by the length difference.

ListNode* getIntersectionNode(ListNode* headA, ListNode* headB) {
    if (!headA || !headB) return nullptr;
    
    ListNode* a = headA, *b = headB;
    
    // The elegant two-pointer trick:
    // When a reaches the end of list A, redirect it to headB.
    // When b reaches the end of list B, redirect it to headA.
    // If they intersect: they'll meet at the intersection after traversing
    //   combined length A + B (with the path equalization built in).
    // If they don't intersect: both reach null simultaneously.
    
    while (a != b) {
        a = (a == nullptr) ? headB : a->next;
        b = (b == nullptr) ? headA : b->next;
    }
    
    return a;  // null if no intersection, intersection node otherwise
}

// WHY this works:
// Pointer A traverses: (all of A) + (all of B up to intersection)
// Pointer B traverses: (all of B) + (all of A up to intersection)
// Both travel the same TOTAL distance before meeting.
// If no intersection: both reach null after traversing A+B total.
```

---

### VARIANT 7: Sort a Linked List in O(n log n) — Merge Sort

```cpp
// The canonical O(n log n) sort for linked lists.
// Merge sort is preferred over quicksort because linked lists have no random access.
// Find middle (fast-slow), split, sort both halves recursively, merge.

ListNode* sortList(ListNode* head) {
    if (!head || !head->next) return head;
    
    // Find middle and split
    ListNode* slow = head, *fast = head->next;  // note: fast starts at head->next
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    ListNode* mid = slow->next;
    slow->next = nullptr;  // split into two halves
    
    // Recursively sort both halves
    ListNode* left = sortList(head);
    ListNode* right = sortList(mid);
    
    // Merge
    return mergeTwoLists(left, right);
}

// WHY fast starts at head->next (not head)?
// For a 2-element list [1,2]: with fast=head->next=2 and slow=head=1,
// after the first (and only) iteration: slow=2, fast=null.
// mid = slow->next = null, slow->next = null.
// Split: left=[1], right=[2]. Correct!
// If fast starts at head: slow would advance to 2, mid=null — no real split.
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Traversal (find node, length) | O(n) | O(1) | Visit each node once |
| Reversal (iterative) | O(n) | O(1) | Visit each node once, 3 pointer updates |
| Reversal (recursive) | O(n) | O(n) | Recursion stack depth = list length |
| Detect cycle (Floyd's) | O(n) | O(1) | Fast pointer traverses at most 2n steps |
| Find cycle start | O(n) | O(1) | Two passes of at most n steps each |
| Find middle (fast-slow) | O(n) | O(1) | Fast traverses n steps, slow n/2 |
| Remove Nth from end | O(n) | O(1) | One pass; gap technique |
| Merge two sorted lists | O(m+n) | O(1) | Visit each node of both lists exactly once |
| Reverse k-group | O(n) | O(1) | Each node reversed exactly once |
| Sort linked list (merge sort) | O(n log n) | O(log n) | log n levels of recursion, O(1) per level except stack |
| Merge K sorted lists (heap) | O(n log k) | O(k) | Each of n nodes does 1 heap push/pop = O(log k) |
| LRU get/put | O(1) | O(capacity) | Hash map + doubly linked list, both O(1) |
| Copy list with random (hash map) | O(n) | O(n) | Two passes, hash map stores n entries |
| Copy list with random (interleave) | O(n) | O(1) | Three passes, no extra structure |

### The O(1) Space Intuition — Say This Out Loud:

"Linked list operations are O(1) space because I manipulate EXISTING pointers — I'm rewiring arrows between nodes that already exist. I don't allocate new arrays or use recursion. The list itself already takes O(n) space (unavoidably), but my ALGORITHM uses only a constant number of pointer variables: prev, curr, next, slow, fast, dummy — all fixed-size regardless of input."

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Accessing `curr->next` Without Checking `curr != nullptr`

**What they do:** Write `while (fast->next->next != nullptr)` without checking `fast` is non-null first.

**What goes wrong:** When the list reaches its end, `fast` becomes `nullptr`. Calling `nullptr->next` is undefined behavior — the program crashes (segfault). This happens on even-length lists, single-node lists, or any list where the loop reaches the end.

**The fix:** Always check the pointer BEFORE dereferencing it. The correct compound check for fast-slow: `while (fast != nullptr && fast->next != nullptr)`.

```cpp
// WRONG: crashes when fast is null (segfault on nullptr->next)
while (fast->next != nullptr && fast->next->next != nullptr) {
    fast = fast->next->next;
}

// CORRECT: check fast before fast->next (short-circuit evaluation)
while (fast != nullptr && fast->next != nullptr) {
    fast = fast->next->next;
}
// In C++, if fast is null, fast->next is never evaluated due to short-circuit &&
```

---

### Mistake 2: Losing the `next` Pointer During Reversal

**What they do:** Reassign `curr->next = prev` BEFORE saving `curr->next` to a temp variable.

**What goes wrong:** `curr->next` is overwritten to point to `prev`. The original `next` node is now unreachable — the entire remainder of the list is orphaned. The rest of the reversal operates on `null`, producing a truncated reversed list. No crash, no error — silent data loss.

**The fix:** ALWAYS save `next` as the FIRST operation inside the reversal loop.

```cpp
// WRONG: overwrites curr->next before saving it — orphans the rest of the list
while (curr != nullptr) {
    curr->next = prev;    // curr->next is now prev — we lost the original next!
    prev = curr;
    curr = curr->next;    // curr->next is prev, not the next unprocessed node!
}

// CORRECT: save next FIRST, then do everything else
while (curr != nullptr) {
    ListNode* next = curr->next;  // SAVE FIRST
    curr->next = prev;            // now safe to redirect
    prev = curr;
    curr = next;                  // use the saved next
}
```

---

### Mistake 3: Not Using a Dummy Node When Head Might Change

**What they do:** Handle head modification as a special case with `if (head should be removed)` branching.

**What goes wrong:** The special-case logic is error-prone, especially under time pressure. Removing the head requires `head = head->next` but the caller's head pointer isn't updated unless you return the new head. Forgetting to return it or double-handling it causes leaks or wrong output.

**The fix:** Use a dummy node. Always. The cost is one extra `new` and one extra pointer dereference. The benefit is eliminating all head-change edge cases.

```cpp
// WRONG: manual head-change handling (brittle under edge cases)
ListNode* removeElements(ListNode* head, int val) {
    while (head != nullptr && head->val == val)
        head = head->next;  // easy to forget this must be returned, not just done
    ListNode* curr = head;
    while (curr != nullptr && curr->next != nullptr) {
        if (curr->next->val == val)
            curr->next = curr->next->next;
        else
            curr = curr->next;
    }
    return head;
}

// CORRECT: dummy node absorbs all head-change cases
ListNode* removeElements(ListNode* head, int val) {
    ListNode* dummy = new ListNode(0);
    dummy->next = head;
    ListNode* curr = dummy;
    while (curr->next != nullptr) {
        if (curr->next->val == val)
            curr->next = curr->next->next;  // skip the node to delete
        else
            curr = curr->next;
    }
    ListNode* result = dummy->next;
    delete dummy;
    return result;
}
```

---

### Mistake 4: Off-by-One in "Remove Nth From End" — Advancing Fast by `n` Instead of `n+1`

**What they do:** Advance the fast pointer by exactly `n` steps to create a gap of `n`, then move both until fast reaches `null`. This puts slow AT the node to delete, not BEFORE it.

**What goes wrong:** To delete `slow`, you need to do `slow->prev->next = slow->next`, but singly linked lists have no backward pointer. You need slow to be ONE BEFORE the target to do `slow->next = slow->next->next`.

**The fix:** Advance fast by `n+1` steps, not `n`. Or advance fast by `n` steps and stop when fast is at the last node (not null).

```cpp
// WRONG: advances fast n steps, slow ends up AT the target (can't delete it)
ListNode* fast = head, *slow = head;
for (int i = 0; i < n; i++) fast = fast->next;  // gap = n
while (fast != nullptr) { slow = slow->next; fast = fast->next; }
// slow is at the Nth from end — but we need to modify slow->prev, which we don't have!

// CORRECT: advance fast n+1 steps (using dummy to allow slow to start one before head)
ListNode* dummy = new ListNode(0); dummy->next = head;
ListNode* fast = dummy, *slow = dummy;
for (int i = 0; i <= n; i++) fast = fast->next;  // gap = n+1 from dummy
while (fast != nullptr) { slow = slow->next; fast = fast->next; }
slow->next = slow->next->next;  // slow is now ONE BEFORE the target — correct
```

---

### Mistake 5: Incorrect Floyd's Meeting Point — Moving Fast by 1 Instead of Restoring Slow to Head

**What they do:** After detecting the meeting point, they try to advance from the meeting point alone to find the cycle start (moving just fast or just slow further).

**What goes wrong:** The mathematical guarantee is SPECIFIC: reset slow to `head`, keep fast at the meeting point, advance BOTH by 1 step at a time — they will meet exactly at the cycle start. Moving only one pointer from the meeting point gives you an arbitrary position in the cycle, not the start.

**The fix:** Follow the exact Phase 2 protocol: reset slow to head, advance both slow and fast by 1 step at a time until they meet.

```cpp
// WRONG: trying to find cycle start without resetting slow to head
// (Common mistake: continuing to move fast at 2x speed, or only moving fast)
ListNode* findCycleStart(ListNode* head) {
    ListNode* slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next; fast = fast->next->next;
        if (slow == fast) {
            // WRONG attempt: move only fast to find start
            while (fast->next != slow) fast = fast->next;  // arbitrary cycle traversal
            return fast->next;  // WRONG: this isn't guaranteed to be the start
        }
    }
    return nullptr;
}

// CORRECT: Phase 2 exactly — reset slow to head, advance BOTH by 1
ListNode* detectCycle(ListNode* head) {
    ListNode* slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next; fast = fast->next->next;
        if (slow == fast) break;
    }
    if (!fast || !fast->next) return nullptr;  // no cycle
    
    slow = head;  // RESET slow to head
    while (slow != fast) {
        slow = slow->next;  // both advance by 1
        fast = fast->next;
    }
    return slow;  // cycle start
}
```

---

### Mistake 6: Memory Leaks — Not Deleting Nodes or Dummy Nodes

**What they do:** Allocate `new ListNode(...)` for dummy nodes or deleted nodes without `delete`-ing them.

**What goes wrong:** In a contest, it doesn't matter. In an interview or a codebase, it matters enormously. More importantly, forgetting to `delete lru` in LRU Cache eviction means the cache's memory grows without bound — a real bug that would affect production systems.

**The fix:** For every `new`, there should be a corresponding `delete`. For dummy nodes, `delete dummy` before returning. For deleted nodes, save the pointer before relinking, then `delete`.

```cpp
// WRONG: dummy node leaked
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode* dummy = new ListNode(0);
    dummy->next = head;
    // ... process ...
    return dummy->next;  // dummy is never deleted — MEMORY LEAK
}

// CORRECT: clean up dummy before returning
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode* dummy = new ListNode(0);
    dummy->next = head;
    // ... process ...
    ListNode* result = dummy->next;
    delete dummy;        // clean up dummy node
    return result;
}

// CORRECT: for deleted nodes
ListNode* toDelete = slow->next;
slow->next = slow->next->next;
delete toDelete;         // clean up the deleted node
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (build pointer discipline)

**1. Reverse Linked List — LC 206 — [Template 1: Iterative reversal]**

Why this problem: The foundation. If you cannot write the 3-pointer iterative reversal (prev/curr/next) in 2 minutes from memory without bugs, every other linked list problem will fight you. This is your handwriting drill — it must be mechanical.

What to prove: `prev = nullptr` initially (the old head becomes the new tail, which must point to null). Save `next` before redirecting `curr->next`. Return `prev` (not `curr`, which is null at the end). The loop condition is `curr != nullptr`.

Target time: 2 minutes (iterative), 5 minutes (recursive).

Edge cases: empty list (return null). Single node (return head — prev stays null, curr == head, first iteration: next=null, head->next=null, prev=head, curr=null; return head). Two nodes.

---

**2. Linked List Cycle — LC 141 — [Template 3: Floyd's cycle detection]**

Why this problem: Fast-slow pointer must be your first instinct for any cycle problem. The classic `O(n^2)` solution (check all pairs) is never acceptable. This problem forces you to internalize the compound null check `while (fast && fast->next)`.

What to prove: Why does the hare always catch the tortoise in a cyclic list? (The gap decreases by exactly 1 per step — within a cycle, fast gains 1 on slow each step — so they meet within C steps of slow entering the cycle.) Why does fast reaching null mean no cycle? (Null is the sentinel for "end of list"; if it can be reached, there's no loop-back.)

Target time: 3 minutes.

Edge cases: Empty list (return false). Single node with no self-loop. Single node pointing to itself (cycle of length 1 — fast and slow both start at head and both move to head on the first step — they meet immediately, return true).

---

### CORE — 4 problems (the essential techniques)

**3. Merge Two Sorted Lists — LC 21 — [Template 7: Dummy node + merge]**

Why this problem: Two important lessons: (1) dummy node eliminates the "which list starts the result" decision, and (2) the `curr->next = (l1 != nullptr) ? l1 : l2` line at the end — you do NOT need to loop to attach the remaining nodes; just attach the whole remaining list.

What to prove: The dummy node design. `curr` always stays one step BEHIND the node you just attached (so you can attach the next one). After the while loop, exactly one of l1/l2 is non-null — attach it directly.

Target time: 8 minutes.

Edge cases: One list is empty (return the other — dummy handles this: while loop exits immediately, curr->next = the non-empty list). Both lists are empty (return null — dummy->next is null).

Common mistake: Continuing to process remaining nodes one-by-one after the while loop instead of just attaching the remaining head.

---

**4. Remove Nth Node From End of List — LC 19 — [Template 6: Gap technique + dummy]**

Why this problem: Forces you to internalize both the dummy node (handles removing the head) and the n+1 gap (slow stops BEFORE the target, not at it). The "one pass" requirement is the interviewer's way of saying "use the two-pointer gap technique."

What to prove: Advance fast by n+1 steps from the dummy (not n). When fast reaches null, slow is at the node before the target. The operation is `slow->next = slow->next->next`.

Target time: 12 minutes.

Edge cases: n equals list length (removing the head — the dummy node handles this cleanly). n = 1 (removing the last node). List with a single node and n = 1.

---

**5. Linked List Cycle II — LC 142 — [Template 4: Floyd's extended to find cycle start]**

Why this problem: Forces you to understand the PROOF, not just apply the template. Interviewers at Google and Meta ask "why does resetting slow to head make them meet at the cycle start?" If you can't answer this, you reveal you memorized the algorithm without understanding it. Re-read the proof in Template 4 until you can reproduce it.

What to prove: Phase 1 (detect with standard Floyd). Phase 2 (reset slow to head, advance BOTH by 1 until they meet). The mathematical identity: `F = mC - k` where F = distance to cycle start, C = cycle length, k = distance from cycle start to meeting point.

Target time: 20 minutes.

Edge cases: No cycle (Phase 1's loop terminates without a meeting point — check `if (fast == null || fast->next == null) return null`). Cycle starts at head (F=0 — fast is already at the meeting point, slow starts at head, they meet at head immediately in Phase 2).

---

**6. Reorder List — LC 143 — [Three-step decomposition: middle + reverse + merge]**

Why this problem: Tests whether you can break a complex problem into sub-problems you already know. The "eureka" moment is recognizing that `1→2→3→4→5` → `1→5→2→4→3` means "first half interleaved with reversed second half." Once you see that, the three steps follow naturally.

What to prove: The "first middle" variant of fast-slow (stop condition: `fast->next && fast->next->next`) ensures the first half is longer for even-length lists, so the merge terminates correctly. Cutting the list at `slow->next = nullptr` before reversing the second half.

Target time: 25 minutes.

Edge cases: Two nodes: [1,2] → [1,2] (already in order, but still need to handle: first half=[1], second half=[2], reversed second half=[2], merge: 1→2). One node: return immediately.

---

### STRETCH — 3 problems (deeper pointer mastery)

**7. Copy List with Random Pointer — LC 138 — [Two approaches: hash map and interleaving]**

Why this problem: The hash map approach is straightforward but O(n) space. The interleaving approach eliminates the extra space by weaving copies between originals, setting random pointers using the weave, then unweaving. Interviewers love this problem because it tests spatial reasoning about pointer structures.

What to prove: In the interleaving approach: `curr->next->random = curr->random->next` — "the copy of curr's random target is the node AFTER curr's random target (which is the copy)." In the unweaving step: you must restore original list's next pointers WHILE extracting the copy list.

Target time: 30 minutes. Implement hash map approach first. Then implement interleaving as the follow-up.

Edge cases: Node with `random = null` (guard with `if (curr->random)` before setting random). Single node. Node that points to itself via random.

---

**8. Add Two Numbers — LC 2 — [Carry propagation with dummy node]**

Why this problem: Trains the "process both lists simultaneously, handle different lengths, handle carry overflow" pattern. The while condition `while (l1 || l2 || carry)` is the elegant way to handle all three termination conditions at once.

What to prove: The final digit pair can generate a carry (e.g., 99 + 1 = 100), creating an extra node. The `|| carry` in the while condition handles this. Without it, the most significant digit is lost.

Target time: 15 minutes.

Edge cases: Lists of different lengths. Final carry creating a new digit (e.g., [9,9] + [1] = [0,0,1]).

---

**9. LRU Cache — LC 146 — [Doubly linked list + hash map, canonical design problem]**

Why this problem: The hardest and most important linked list problem. Tests your ability to design a compound data structure from scratch. The key insight: you need O(1) deletion from any position in the list — only a doubly linked list (with both prev and next pointers) gives you this.

What to prove: `get()` must both retrieve AND move the node to the front (update recency). `put()` with existing key must update value AND move to front. `put()` with new key when at capacity must evict the LRU (tail->prev) AND update the map. The dual-dummy (dummy head + dummy tail) eliminates all edge cases in the `remove()` helper — you never have to check "is this node the head or tail?"

Target time: 40 minutes. Budget 10 minutes for the design discussion, 30 minutes for clean implementation.

Edge cases: Capacity = 1. Getting a key that doesn't exist (return -1). Putting a key that already exists (update value, move to MRU). The evicted node's key must be erased from the map (easy to forget).

Common mistake: Erasing from the map AFTER deleting the node — accessing `lru->key` after `delete lru` is undefined behavior. Save the key first.

---

### CONTEST — 2 problems

**10. Merge K Sorted Lists — LC 23 — [Min-heap of list heads]**

Why this is contest level: Combines two patterns: linked list merge (Template 7) and heap (Pattern 10). The naive approach of merging lists one by one is O(nk) — too slow. The heap approach runs the merge in parallel, pulling the globally smallest node at each step in O(log k) time.

What to prove: The priority_queue comparator for ListNode: `auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; }` — the `>` makes it a MIN-heap (smallest value at top). After extracting the minimum node, push `node->next` if it exists.

Target time: 25 minutes.

Edge cases: Empty `lists` vector. Lists containing empty lists (null heads — filter before pushing). All lists of length 1.

---

**11. Reverse Nodes in k-Group — LC 25 — [Template 8: Group reversal with groupPrev tracking]**

Why this is contest level: Demands combining reversal with group tracking. The hardest part is maintaining `groupPrev` — the node BEFORE the current group — because after reversing a group, the old head becomes the new tail, which becomes the new `groupPrev`.

What to prove: The `getKthNode` helper correctly checks for exactly k available nodes (returns null if fewer). In the reversal, `prev = groupNext` (the start of the NEXT group), so the reversed group's tail automatically connects to the next group. `groupPrev->next = kth` (kth is the last node of the group, which becomes the new head after reversal).

Target time: 35 minutes. Trace through [1,2,3,4,5], k=2 manually before coding.

Edge cases: k = 1 (no reversal needed — return head immediately). List length < k (no reversal — handled by `getKthNode` returning null). List length exactly k (reverse the whole list).

---

## PATTERN CONNECTIONS

**Connects to Fast-Slow Pointer in Arrays (Pattern 02):**
The same fast-slow technique that detects cycles in linked lists also finds duplicates in arrays (Floyd's algorithm on arrays — the value at each index is treated as the "next pointer"). If you truly understand Template 3/4, you can solve LC 287 (Find the Duplicate Number) in O(n) time O(1) space.

**Connects to Heap (Pattern 10):**
Merge K Sorted Lists is the bridge problem between linked lists and heaps. The heap handles the "which of K candidates is next" decision in O(log K); the linked list provides the structure. These two patterns combine in interview problems regularly.

**Connects to Hash Map (Pattern 01):**
Copy List with Random Pointer (hash map approach) and LRU Cache both use hash maps for O(1) lookup. Linked lists provide the ordering/sequencing; hash maps provide the lookup. Together they create O(1) everything data structures.

**Connects to Binary Search (Pattern 05):**
"Find the middle" is used in binary search on linked lists (LC 109 — Convert Sorted List to BST). Fast-slow finds the middle in O(n/2); then you build the BST recursively by treating the middle as the root.

**Connects to Sorting (Pattern 34 — Greedy + Arrays):**
Merge sort on a linked list uses the same merge logic as Template 7. The O(n log n) sort uses repeated merge passes, but on linked lists instead of arrays. QuickSort on linked lists is less natural due to lack of random access — merge sort is always preferred.

**Connects to Trees (Pattern 08):**
A tree is just a linked list with multiple `next` pointers. Almost every tree traversal concept (DFS, recursion, parent tracking) has a linked list analog. Solving linked list problems builds the pointer intuition that makes tree problems feel natural.

**Connects to DP (Pattern 17):**
Some linked list problems have DP solutions. "Reverse a portion of a list to make it palindromic" can be framed as a string DP on the sequence of values. More directly: problems that ask "minimum operations to..." on a linked list often have DP solutions where the linked list structure defines the subproblem boundaries.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Reverse a linked list. Now, can you reverse every k nodes in the list? If the remaining nodes are fewer than k, leave them as is."

**Company type:** Microsoft phone screen, Amazon OA, Google (LC 206 + LC 25 — progressive difficulty)
**Expected time:** 5 minutes for basic reversal, 25 minutes for k-group
**What they are testing:** Pointer discipline, ability to generalize a simple operation, edge case handling under pressure

**Red flags that get candidates rejected:**
- Using `vector<int>` to collect values, reverse, rebuild — shows you don't know in-place pointer manipulation; interviewers explicitly watch for this
- Writing the basic reversal but losing the pointer during k-group (trying to "apply the reversal function" without tracking `groupPrev`)
- Crashing on single-node lists or lists shorter than k due to missing null checks
- Taking more than 3 minutes on the basic reversal — this must be mechanical
- In k-group: forgetting to connect `groupPrev->next = kth` (the reversed group floats unconnected to the rest of the list)
- Not using a dummy node for k-group, then having separate logic for when the first group is reversed

**What they want you to say:**
"For basic reversal: three pointers — prev (null), curr (head), and a temp next. Each step: save next, redirect curr->next to prev, advance both. Return prev. For k-group: I'll use a dummy node and a groupPrev pointer. At each iteration, I check if k nodes remain. If yes, I reverse that group (using the same 3-pointer technique, but I initialize prev to groupNext — the start of the next group — so the tail of this reversed group automatically connects forward). Then I advance groupPrev to the old head of the group (new tail)."

**The follow-up:** "Can you do this without counting the list length first?" → Use `getKthNode` — count k nodes forward from groupPrev. If you can't reach k nodes, stop.

---

### Q2: "Given a linked list, determine if it has a cycle. If it does, return the node where the cycle begins."

**Company type:** Google, Meta, Amazon — Floyd's cycle detection appears in 20%+ of linked list rounds
**Expected time:** 15 minutes (cycle detection: 5 min, cycle start: 10 min)
**What they are testing:** Knowledge of Floyd's algorithm, ability to explain the mathematical proof, pointer discipline with two simultaneous pointers

**Red flags that get candidates rejected:**
- Using a hash set of visited nodes — O(n) space; interviewer will ask for O(1) space, and if you say "I don't know," that's the end
- Moving fast by 1 step after the meeting point (instead of resetting slow to head and moving both by 1)
- Not being able to explain WHY Phase 2 works — just saying "it's a known algorithm" is a red flag at Google
- Skipping the null check before Phase 2 (if no cycle, fast is null — proceeding to Phase 2 crashes)
- Checking `slow == fast` before the first move (they both start at head, so they'd meet immediately — check AFTER each move)

**What they want you to say:**
"Phase 1: fast and slow both start at head. Advance slow by 1, fast by 2 each step. If they meet, a cycle exists. If fast reaches null, no cycle. Phase 2: reset slow to head, keep fast at the meeting point, advance BOTH by 1 step at a time. They'll meet exactly at the cycle start. Why? Mathematically: let F = distance from head to cycle start, C = cycle length, k = distance from cycle start to meeting point. When they meet: fast has traveled F + k + mC and slow has traveled F + k. Since fast = 2×slow: mC = F + k, so F = mC - k. In Phase 2, slow walks F steps from head (reaching cycle start), and fast walks F steps from k-into-cycle = k + mC - k = cycle start. They meet at the cycle start."

**The follow-up:** "What if there are duplicate values in the list — does the algorithm still work?" → Yes. Floyd's algorithm compares POINTER identity (`slow == fast`), not values. Two nodes with the same value at different positions are different pointers and will NOT cause a false positive.

---

### Q3: "Design an LRU (Least Recently Used) Cache with O(1) get and put."

**Company type:** Amazon onsite (appears in ~30% of Amazon SDE-2 loops), Microsoft, Uber, Lyft
**Expected time:** 35 minutes
**What they are testing:** Compound data structure design, why doubly linked list + hash map, O(1) deletion from middle, eviction policy implementation

**Red flags that get candidates rejected:**
- Saying "I'll use an ordered map or deque" — neither gives you O(1) deletion from the middle
- Using singly linked list — can't do O(1) deletion without knowing the previous node
- Forgetting to move the accessed node to the front on `get()` — the problem says "get makes it recently used"
- Forgetting to update the hash map when evicting a node — the map retains a stale pointer to a deleted node, causing undefined behavior on the next access
- Using `capacity` check on EVERY operation instead of only on `put()` when inserting a new key
- Not using dummy head and tail nodes — then every `remove()` call needs `if (node == head)` and `if (node == tail)` special casing that causes bugs

**What they want you to say:**
"I need O(1) for both get (lookup by key) and put (insert and evict). A hash map gives O(1) lookup by key. But I also need to maintain recency order and evict the LRU in O(1). An array can't give O(1) deletion from the middle. A singly linked list can't either — deletion needs the previous node. A doubly linked list CAN give O(1) deletion from any position. So: doubly linked list for ordering (most recent at head, LRU at tail) + hash map (key → node pointer). I'll use dummy head and tail sentinels to eliminate null checks. `get`: look up node in map, move to front, return value. `put`: if key exists, update and move to front. If new key and at capacity, evict the LRU (tail->prev), erase from map, then insert new node at front."

**The follow-up they ALWAYS ask:** "What is the time and space complexity?" → get: O(1) time. put: O(1) amortized time. Space: O(capacity) for both the hash map and the linked list (at most capacity nodes stored). Then: "What changes if you need an LFU (Least Frequently Used) cache?" → Maintain a frequency map and a map of frequency → doubly linked list. Much harder — O(1) still achievable but requires more structure.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

**Q1:** I'm reversing a linked list iteratively. After my loop ends, what does `curr` point to, and what does `prev` point to? What do I return?
> **A:** After the loop ends, `curr` is `nullptr` (the loop condition was `while (curr != nullptr)`). `prev` points to the last node processed — which is the OLD tail, now the NEW head of the reversed list. Return `prev`, not `curr`.

**Q2:** I need to remove the 3rd node from the end of a list with 5 nodes. I use a dummy node. How many steps do I advance `fast` before starting to move both pointers?
> **A:** `n + 1 = 4` steps (from the dummy node). After advancing fast 4 steps from dummy: fast is at node 4 (0-indexed from dummy). Then advance both until fast is null: slow ends at node 1 (the node BEFORE node 2, the 3rd from end). `slow->next = slow->next->next` removes node 2. The dummy trick means this also works if n = list length (removing the head).

**Q3:** In Floyd's cycle detection, if I reset slow to head after Phase 1, why can't I also reset fast to head (and advance both by 1)?
> **A:** If you reset both to head, you're just doing a fresh linear scan from head — you've lost all information about where the meeting happened. The whole point of Phase 2 is that fast starts INSIDE the cycle (at the meeting point, which is `k` steps into the cycle), while slow starts at head. The mathematical guarantee that they meet at the cycle start REQUIRES fast to start at the meeting point.

**Q4:** LRU Cache: when I call `get(key)` on an existing key, what two things must I do (not just one)?
> **A:** (1) REMOVE the node from its current position in the doubly linked list. (2) INSERT it at the FRONT (most recently used position). Most people remember "move to front" but implement it as "insert at front" without first removing from current position — this leaves the node in TWO places simultaneously.

**Q5:** For "Reorder List" on [1,2,3,4], after finding the middle with `while (fast->next && fast->next->next)`, what does slow point to, and how do I split the list?
> **A:** slow points to node 2 (the FIRST of the two middle nodes for even-length lists). I split by: `secondHalf = slow->next; slow->next = nullptr;`. This gives first half [1,2] and second half [3,4]. The `slow->next = nullptr` is critical — it terminates the first half so later operations don't accidentally traverse into the second half.

**Q6:** In Merge K Sorted Lists, I pop a node from the min-heap. What do I push into the heap after attaching this node to the result list?
> **A:** `node->next` — the next node from the same list this node came from. If `node->next` is null (this list is exhausted), I push nothing. The heap always contains at most one node per input list (the current "frontier" of that list).

**Q7:** Copy List with Random Pointer — interleaving method. In Step 2, what is the formula for setting a copy node's random pointer, and why?
> **A:** `curr->next->random = curr->random->next`. Reading right to left: `curr->random` is the original node that `curr` points to via random. `curr->random->next` is the copy of that original node (because in the interleaved list, every original node is immediately followed by its copy). So the copy of `curr` (which is `curr->next`) should have its random pointer set to the copy of `curr->random` (which is `curr->random->next`).

---

## C++ LINKED LIST CHEAT SHEET

```cpp
// ============================================================
// LINKED LIST NODE DEFINITION
// ============================================================

struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x) : val(x), next(nullptr) {}
};

// For doubly linked list (LRU Cache, etc.):
struct DListNode {
    int key, val;
    DListNode *prev, *next;
    DListNode(int k, int v) : key(k), val(v), prev(nullptr), next(nullptr) {}
};

// ============================================================
// THE DUMMY NODE PATTERN
// ============================================================
ListNode* dummy = new ListNode(0);
dummy->next = head;
// ... operations ...
ListNode* result = dummy->next;
delete dummy;
return result;

// ============================================================
// BUILD A LIST FROM VECTOR (for testing)
// ============================================================
ListNode* buildList(vector<int> vals) {
    ListNode* dummy = new ListNode(0);
    ListNode* curr = dummy;
    for (int v : vals) { curr->next = new ListNode(v); curr = curr->next; }
    return dummy->next;
    // Note: don't delete dummy here — just return dummy->next
}

// ============================================================
// PRINT A LIST (for debugging)
// ============================================================
void printList(ListNode* head) {
    while (head) { cout << head->val << " -> "; head = head->next; }
    cout << "null\n";
}

// ============================================================
// REVERSAL TEMPLATE (memorize this)
// ============================================================
ListNode* prev = nullptr, *curr = head;
while (curr) {
    ListNode* next = curr->next;
    curr->next = prev;
    prev = curr;
    curr = next;
}
// result: prev

// ============================================================
// FAST-SLOW POINTER (memorize the check)
// ============================================================
ListNode* slow = head, *fast = head;
while (fast != nullptr && fast->next != nullptr) {
    slow = slow->next;
    fast = fast->next->next;
}
// slow at middle for odd; second middle for even

// ============================================================
// MERGE TWO SORTED LISTS (memorize this)
// ============================================================
ListNode* dummy = new ListNode(0), *curr = dummy;
while (l1 && l2) {
    if (l1->val <= l2->val) { curr->next = l1; l1 = l1->next; }
    else                    { curr->next = l2; l2 = l2->next; }
    curr = curr->next;
}
curr->next = l1 ? l1 : l2;

// ============================================================
// MIN-HEAP FOR MERGE K LISTS
// ============================================================
auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
// Push all non-null heads, then pop → attach → push next, repeat

// ============================================================
// CYCLE DETECTION (memorize the exact two-phase structure)
// ============================================================
// Phase 1: find meeting point
// Phase 2: reset slow = head, advance both by 1
// Meeting point = cycle start

// ============================================================
// CEILING DIVISION REMINDER (sometimes needed in linked list problems)
// ============================================================
// ceil(n / 2) = (n + 1) / 2  (integer arithmetic)
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. Implement Template 1 (reversal), Template 3 (cycle detect), and Template 7 (merge sorted) three times each from memory — on paper first, then in the IDE. Do not look at notes.

2. For LRU Cache (LC 146): spend 10 minutes drawing the doubly linked list state before and after each operation on paper. Then code from the drawing. This is the only way to avoid pointer bugs.

3. In REVISION-TRACKER.md, note the following as "revisit weekly":
   - Floyd's cycle start proof (can you derive F = mC - k from scratch?)
   - The difference between `fast != null && fast->next != null` vs `fast->next != null && fast->next->next != null` (when finding middle for splitting)
   - The k+1 vs k step count in Remove Nth From End

4. Connecting forward: every tree operation in Pattern 08 uses the same pointer discipline. Before starting Pattern 08, re-read Template 1 (reversal) — the "save, redirect, advance" loop is identical in structure to tree node manipulation.

5. Contest application: Linked list problems in contests usually combine with OTHER patterns. "Sort the linked list" = merge sort (Pattern 34) + linked list. "Serialize the list" = convert to array + process. "Find Kth smallest in K sorted lists" = Merge K Sorted Lists (Template merge + heap). The linked list is always the structure; the algorithm comes from another pattern.

---

## SIGN-OFF CRITERIA

**Tier 1 (Non-negotiable):**
- [ ] Iterative reversal from memory in under 2 minutes — no bugs, correct return value
- [ ] Floyd's cycle detection (hasCycle) from memory — correct null check order
- [ ] Floyd's cycle start (detectCycle) from memory — Phase 2 reset understood, not just memorized
- [ ] Can explain WHY Phase 2 of Floyd's works mathematically (F = mC - k)

**Tier 2:**
- [ ] LC 19 (Remove Nth From End) — uses dummy node + n+1 gap, no special-case for head removal
- [ ] LC 21 (Merge Two Sorted Lists) — dummy node, single line to attach remaining list
- [ ] LC 143 (Reorder List) — decomposed into three steps without hints

**Tier 3:**
- [ ] LC 138 (Copy with Random Pointer) — implemented BOTH hash map AND interleaving approaches
- [ ] LC 146 (LRU Cache) — implemented from scratch with dual-dummy doubly linked list

**Tier 4 (attempt before sign-off):**
- [ ] LC 25 (Reverse k-Group) — groupPrev tracking understood, not just working code

**Sign-off test:** I give you one unseen linked list problem. You must:
1. Identify the sub-pattern (reversal / fast-slow / merge / design)
2. State whether you need a dummy node and why
3. Write complete C++ solution
4. State time and space complexity

Type `SIGN OFF P07` when ready.

---

*Pattern 07 Complete — Linked Lists (Reversal + Fast-Slow + Merge + LRU Design)*
*Next: Pattern 08 — Trees: DFS Complete*
