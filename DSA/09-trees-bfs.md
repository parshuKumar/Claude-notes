# Pattern 09 — Trees: BFS and Level Order
## Difficulty Level: CORE
## Interview Frequency: OFTEN
## Estimated time to master: 2-3 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

After Pattern 08, you know how to think recursively about trees. BFS is the antidote — it's the pattern you reach for when recursion gives you the WRONG traversal order.

The core insight that most people miss: **DFS and BFS solve fundamentally different questions on trees.** DFS answers "what is the property of this subtree?" — the answer bubbles up. BFS answers "what is the state at this depth?" — the answer is discovered level by level. Once you internalize this distinction, you stop second-guessing which to use.

**The practical rule:** If the problem mentions levels, depths between two nodes, or requires processing nodes that are equidistant from the root, it's BFS. If it mentions subtree properties, paths from root to leaf, or ancestor-descendant relationships, it's DFS.

**At your 1600 level, you:**
- Know BFS exists and can code it for graphs
- Struggle with the "level-by-level" variant — you either use a single queue without level tracking, or you add a dummy `null` sentinel and confuse yourself
- Haven't internalized the `size = q.size()` inner loop trick for level processing
- Cannot solve Maximum Width of Binary Tree (LC 662) due to the position/index trick under pressure
- Mix up BFS vs DFS when problems have both valid solutions — wasting time choosing

**After this document, you will:**
- Have one clean, canonical BFS template with level tracking — no sentinel tricks, no confusion
- Know exactly which problems need BFS vs DFS
- Understand the "virtual index" trick for width problems — and why integer overflow matters
- Solve all 5 common BFS-on-tree variants (level order, right view, zigzag, connect pointers, width) from a single template
- Know when BFS gives a WRONG answer where DFS gives the right one, and vice versa

**The interview data:**
- Level Order Traversal (LC 102) is asked as a warm-up at every company — you must do it in under 5 minutes
- Right Side View (LC 199) appears in 30%+ of Microsoft rounds and 20%+ of Meta rounds
- Maximum Width of Binary Tree (LC 662) is the "gotcha" tree problem — easy to get the wrong answer with the wrong index method
- Bus Routes (LC 815) shows up at Google as a disguised BFS problem

---

## ELI5 — THE LEVEL-BY-LEVEL THINKING ANALOGY

### BFS as Ripples in a Pond

Drop a stone in a pond. The ripple expands outward in concentric circles. Every point at distance 1 from the center gets hit before any point at distance 2. Every point at distance 2 before any point at distance 3.

BFS on a tree is exactly this. The root is the stone. Level 1 nodes (root's children) get visited before level 2 nodes (grandchildren). Level 2 before level 3. You explore the tree in expanding concentric "rings" outward from the root.

DFS, by contrast, is like a river — it flows all the way to the sea in one branch before doubling back to explore a different tributary. A river reaches level 5 in one direction before touching level 2 in another direction.

**When do you want ripples vs rivers?**
- "What do I see looking at the tree from the right side?" → You want the RIGHTMOST node at each LEVEL → BFS ripples
- "What is the maximum path sum from any node to any node?" → You want to explore complete subtrees → DFS rivers

### The Queue as a Waiting Room

A queue is a waiting room. Everyone who arrives gets processed in order. When you process someone, you add their children to the end of the waiting room.

BFS level tracking: at the START of each "round," count how many people are currently waiting. Process exactly that many. Everyone they add is in the NEXT round. This is the `size = q.size(); for (int i = 0; i < size; i++)` inner loop pattern.

The queue naturally ensures that all nodes from level K are processed before any node from level K+1, because level K's children (level K+1) are added to the queue only AFTER all of level K is already in the queue.

### The "Right Side View" Intuition

Imagine standing to the right of the tree and looking left. You see exactly one node per level — the rightmost visible one. In BFS terms: the LAST node you process at each level is the rightmost node. The `size = q.size()` pattern lets you detect "last node of this level" exactly — it's the node at `i == size - 1`.

### Why DFS Gets Right Side View Wrong

If you tried to solve right side view with DFS (going right before left), the FIRST node you visit at each depth would be the rightmost. But "first visited at depth d" in right-first DFS ≠ "rightmost at level d" in general. In a balanced tree it coincidentally works. In an unbalanced tree (e.g., a node only has a left child), it fails silently.

BFS guarantees the ACTUAL rightmost visible node because it processes ALL nodes at a level in spatial order.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:

**Level-by-level traversal:**
1. Level order traversal — collect all nodes level by level
2. Level averages, level maximums, level sums — aggregate per level
3. Level counts, level sizes — metadata about the tree's shape

**Level-based extraction:**
1. Right side view — the last node at each level (rightmost visible)
2. Left side view — the first node at each level (leftmost visible)
3. Bottom-up level order — same as level order, but reverse the levels vector

**Structural traversal:**
1. Zigzag traversal — alternate left-to-right and right-to-left per level
2. Symmetric tree check — compare mirror-image nodes level by level
3. Even-odd tree constraints — constraints on alternating levels

**Width and position problems:**
1. Maximum width of binary tree — virtual index tracking (not node counting)
2. Vertical order traversal — column-based grouping with BFS

**Tree modification with level awareness:**
1. Connect next right pointers — link nodes at the same level
2. O(1) space BFS using the `next` pointers themselves

**Shortest path on tree:**
1. Minimum depth to leaf — BFS stops at the FIRST leaf encountered
2. BFS on transformed graph where tree is one component (Bus Routes)

### The invariant:

At every BFS step, the queue contains all nodes at the CURRENT level and nothing else. This invariant is maintained by the `size = q.size()` inner loop: you process exactly the nodes from the current level, and the children you add form the next level without mixing.

---

## WHEN TO RECOGNIZE THIS PATTERN

**Signal 1:** "Level order traversal" or "print nodes level by level" or "collect nodes at each depth"
→ The canonical BFS template. Queue, `size = q.size()` inner loop, collect into `vector<vector<int>>`. This is the template every other BFS-on-tree problem is derived from. Know it cold.

**Signal 2:** "Right side view" or "left side view" or "what would you see from the [side]"
→ BFS with "take the last (or first) element from each level." In the inner loop, check `if (i == size - 1)` (for right view) or `if (i == 0)` (for left view) and record the node's value. A DFS alternative exists (mark first/last seen at each depth in a hash map) but BFS is cleaner and more natural.

**Signal 3:** "Zigzag traversal" or "spiral traversal" or "alternate direction per level"
→ BFS with a direction flag. Collect each level into a vector. Even levels: append normally (left to right). Odd levels: reverse before appending (or use a deque and push to front/back based on direction). The BFS order itself doesn't change — only how you store the collected level.

**Signal 4:** "Minimum depth" or "shortest path to leaf" or "first leaf encountered"
→ BFS stops at the FIRST leaf (node with no children). BFS guarantees the first leaf found IS at the minimum depth because BFS visits nodes in order of increasing depth. DFS would find A leaf but not necessarily the SHALLOWEST one. This is a key advantage of BFS over DFS.

**Signal 5:** "Connect nodes at the same level" or "populate next right pointer" or "link siblings"
→ BFS with level grouping. Within each level, connect nodes left to right. The cleaner O(1) space approach uses the `next` pointer itself: process the previous level's nodes and use their `next` pointers to iterate the current level without a queue.

**Signal 6:** "Maximum width of the binary tree" or "widest level"
→ Virtual index BFS. Assign each node a position index (root = 1, left child of node at index i = 2*i, right child = 2*i+1). Width of a level = last_index - first_index + 1. Must normalize indices at the start of each level (subtract the first index of that level) to prevent integer overflow on skewed trees.

**Signal 7:** "Vertical order traversal" or "column-based grouping"
→ BFS with (node, column) pairs in the queue. Left child: column-1. Right child: column+1. Collect nodes by column. For strict vertical order (same column, sort by row, then by value), BFS naturally gives row order within a column, but values within the same row and column need sorting.

**Signal 8:** The problem involves reaching from one node type to another through a BFS "expansion" — words that differ by one letter, buses sharing stops, cells adjacent to a value
→ BFS on an implicit graph. The tree or grid is just one layer; the real graph is defined by the transition rule. Build adjacency on the fly. Multi-source BFS if there are multiple starting points.

### Signals that LOOK like BFS but are NOT:

**Fake signal 1:** "Find the depth of a node" — if the tree is small or you're given a specific node reference
→ DFS passing the current depth downward. BFS works too, but DFS with a depth parameter is simpler and avoids allocating a queue.

**Fake signal 2:** "Find all root-to-leaf paths" — collecting complete paths
→ DFS with a current-path vector and backtracking. BFS could theoretically collect paths, but each queue entry would need to carry the full path so far — O(n²) space. DFS handles this naturally with one shared vector and backtrack.

**Fake signal 3:** "Mirror/Symmetric" at the node level — checking if left and right subtrees are mirrors
→ Can be DFS (recursively compare left.left with right.right, left.right with right.left). BFS works (level-by-level comparison using a queue of pairs), but DFS is more natural for structural recursion.

---

## THE TEMPLATES (C++)

### Template 1: Level Order Traversal — The Canonical BFS Template

```cpp
// Use for: every BFS-on-tree problem. Everything else is a variant of this.
// Returns a 2D vector: result[level] = list of node values at that level.

vector<vector<int>> levelOrder(TreeNode* root) {
    if (!root) return {};
    
    vector<vector<int>> result;
    queue<TreeNode*> q;
    q.push(root);
    
    while (!q.empty()) {
        int size = q.size();          // number of nodes at the CURRENT level
        vector<int> level;
        
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        
        result.push_back(level);
    }
    
    return result;
}

// WHY `int size = q.size()` at the start of the outer loop?
// The queue mixes nodes from the current level AND the next level.
// By capturing `size` BEFORE the inner loop, we know exactly how many nodes
// belong to the current level. After processing those, everything remaining
// in the queue (added by their children) is the next level.
//
// WHY NOT use a null sentinel (push null after each level)?
// Sentinels work but are error-prone: you can accidentally push null for empty level,
// you need to check `if (node)` before accessing node->val, and the logic
// is harder to reason about. The `size = q.size()` pattern is strictly cleaner.

// TRACE for tree [3, left=9, right=20, right-left=15, right-right=7]:
// Init: q=[3]
// Outer loop 1: size=1, process 3, push 9 and 20. level=[3]. q=[9,20].
// Outer loop 2: size=2, process 9 (no children), process 20, push 15,7.
//               level=[9,20]. q=[15,7].
// Outer loop 3: size=2, process 15 and 7 (no children). level=[15,7]. q=[].
// Result: [[3],[9,20],[15,7]] ✓
```

### Template 2: Right Side View

```cpp
// The last node processed at each level is the rightmost node at that level.
// BFS processes levels left-to-right, so the LAST node in each level = rightmost.

vector<int> rightSideView(TreeNode* root) {
    if (!root) return {};
    
    vector<int> result;
    queue<TreeNode*> q;
    q.push(root);
    
    while (!q.empty()) {
        int size = q.size();
        
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            
            if (i == size - 1) result.push_back(node->val);  // LAST of this level
            
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
    
    return result;
}

// For LEFT side view: change `i == size - 1` to `i == 0`.
// For BOTH views simultaneously: record both first and last.

// HOW TO ADAPT for DFS alternative (if asked):
// void dfs(TreeNode* node, int depth, vector<int>& result) {
//     if (!node) return;
//     if (depth == result.size()) result.push_back(node->val);  // first seen at this depth
//     else result[depth] = node->val;                           // overwrite with rightmost
//     dfs(node->left, depth+1, result);
//     dfs(node->right, depth+1, result);
// }
// Note: DFS right-side-view must go LEFT then RIGHT (rightmost overwrites leftmost).
```

### Template 3: Zigzag Level Order Traversal

```cpp
// Even levels: left to right. Odd levels: right to left.
// Approach: BFS normally, but reverse odd-numbered level vectors.

vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
    if (!root) return {};
    
    vector<vector<int>> result;
    queue<TreeNode*> q;
    q.push(root);
    bool leftToRight = true;
    
    while (!q.empty()) {
        int size = q.size();
        vector<int> level(size);  // preallocate to allow fill from either end
        
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            
            // Fill index: left-to-right uses i, right-to-left uses (size-1-i)
            int idx = leftToRight ? i : (size - 1 - i);
            level[idx] = node->val;
            
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        
        result.push_back(level);
        leftToRight = !leftToRight;
    }
    
    return result;
}

// ALTERNATIVE (simpler, slightly less efficient): collect normally, then reverse odd levels
// for (auto& lv : result) if (lv_index % 2 == 1) reverse(lv.begin(), lv.end());
//
// The preallocated-vector approach is cleaner for interviews — no extra reverse pass.
// The fill direction changes, but the BFS order (children pushed left-then-right) stays the same.
```

### Template 4: Minimum Depth of Binary Tree — BFS Stops at First Leaf

```cpp
// BFS guarantees: the FIRST leaf encountered is at the MINIMUM depth.
// Stop as soon as you see a leaf — don't process the entire tree.

int minDepth(TreeNode* root) {
    if (!root) return 0;
    
    queue<TreeNode*> q;
    q.push(root);
    int depth = 0;
    
    while (!q.empty()) {
        depth++;
        int size = q.size();
        
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            
            // Leaf node: both children are null
            if (!node->left && !node->right) return depth;  // STOP HERE
            
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
    
    return depth;  // not reached if root is non-null, but satisfies compiler
}

// WHY BFS and not DFS?
// DFS computes min depth by returning 1 + min(left_depth, right_depth).
// BUT: a node with only one child CANNOT return 0 for the missing child side —
// the minimum path must go to an ACTUAL LEAF.
// DFS handles this with: if one child is null, return the depth of the other child.
// BFS handles it naturally: first leaf encountered = minimum depth. No special case.

// COMMON DFS MISTAKE: `min(left, right)` returns 0 when one side is null,
// incorrectly treating non-leaf nodes as leaves.
// DFS CORRECT version:
// if (!root->left && !root->right) return 1;  // leaf
// if (!root->left)  return 1 + minDepth(root->right);   // only right child
// if (!root->right) return 1 + minDepth(root->left);    // only left child
// return 1 + min(minDepth(root->left), minDepth(root->right));  // both children
```

### Template 5: Connect Next Right Pointers — O(n) Space BFS, then O(1) Space

```cpp
// LC 116/117: Populate next right pointers for nodes at the same level.
// O(n) space solution: straightforward BFS, link consecutive nodes in each level.

Node* connect_ON_space(Node* root) {
    if (!root) return nullptr;
    
    queue<Node*> q;
    q.push(root);
    
    while (!q.empty()) {
        int size = q.size();
        
        for (int i = 0; i < size; i++) {
            Node* node = q.front(); q.pop();
            
            // Link to the next node in the SAME level (not across levels)
            if (i < size - 1) node->next = q.front();  // peek without popping
            
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        // After inner loop: last node of this level has next = null (default)
    }
    
    return root;
}

// O(1) EXTRA SPACE SOLUTION (LC 116 — perfect binary tree):
// Use the PREVIOUSLY connected level to iterate the current level.
// At each level, we have a "leftmost" starting node and we use .next pointers
// to traverse the level while building the next level's connections.

Node* connect_O1_space(Node* root) {
    if (!root) return nullptr;
    
    Node* leftmost = root;  // leftmost node of the current level
    
    while (leftmost->left) {  // while there's a next level (not a leaf level)
        Node* curr = leftmost;
        
        while (curr) {
            // Connect left child to right child (within same parent)
            curr->left->next = curr->right;
            
            // Connect right child to left child of next parent (across parents)
            if (curr->next)
                curr->right->next = curr->next->left;
            
            curr = curr->next;  // use the ALREADY established next pointer to advance
        }
        
        leftmost = leftmost->left;  // go to the next level
    }
    
    return root;
}

// WHY O(1) works for a PERFECT binary tree but needs modification for arbitrary trees:
// Perfect binary tree: every internal node has exactly 2 children.
// The cross-parent connection `curr->right->next = curr->next->left` is always valid
// because curr->next always has a left child.
// LC 117 (arbitrary tree): must handle missing children, which requires tracking
// the "previously connected child node" to connect to the next available child.
```

### Template 6: Maximum Width — Virtual Index BFS

```cpp
// LC 662: Width of a level = position of last node - position of first node + 1.
// Virtual positions: root = 1, left child of node at pos k = 2k, right child = 2k+1.
// This mirrors a complete binary tree's array representation.

int widthOfBinaryTree(TreeNode* root) {
    if (!root) return 0;
    
    // Queue stores {node, position_index}
    queue<pair<TreeNode*, unsigned long long>> q;
    q.push({root, 1});
    int maxWidth = 0;
    
    while (!q.empty()) {
        int size = q.size();
        unsigned long long first = q.front().second;  // position of leftmost in this level
        unsigned long long last  = first;
        
        for (int i = 0; i < size; i++) {
            auto [node, pos] = q.front(); q.pop();
            
            // Normalize: subtract the first position of this level to prevent overflow
            unsigned long long normPos = pos - first;
            last = normPos;
            
            if (node->left)  q.push({node->left,  2 * normPos});
            if (node->right) q.push({node->right, 2 * normPos + 1});
        }
        
        maxWidth = max(maxWidth, (int)(last - 0 + 1));  // last - first + 1, first is 0 after normalize
    }
    
    return maxWidth;
}

// WHY normalization (pos - first)?
// In a skewed tree, positions can grow as 2^h which overflows int (and even long long
// for deeply skewed trees). By resetting the leftmost node's position to 0 at each level,
// we keep positions bounded by the width of the level (at most 2^h - 1 within a level,
// but the total never exceeds the level's actual width).
//
// Example of overflow without normalization:
// A right-skewed tree: root=1, right child=3, right's right=7, ...
// At depth 31: rightmost node position = 2^32 - 1 → OVERFLOWS signed int.
// With normalization: at each level, the leftmost is 0, rightmost is level_width - 1.
//
// WHY unsigned long long?
// Even with normalization, intermediate computations (2 * normPos + 1) can momentarily
// exceed int range. unsigned long long gives 64 bits of headroom.
```

### Template 7: Vertical Order Traversal

```cpp
// LC 987: Group nodes by column. Within same column: sort by row, then by value.
// BFS naturally gives row order. Within same row+column, sort by value.

vector<vector<int>> verticalTraversal(TreeNode* root) {
    if (!root) return {};
    
    // {column: {row: [values]}} — collect all nodes
    map<int, map<int, vector<int>>> colMap;
    
    // BFS: queue stores {node, row, col}
    queue<tuple<TreeNode*, int, int>> q;
    q.push({root, 0, 0});
    
    while (!q.empty()) {
        auto [node, row, col] = q.front(); q.pop();
        colMap[col][row].push_back(node->val);
        
        if (node->left)  q.push({node->left,  row + 1, col - 1});
        if (node->right) q.push({node->right, row + 1, col + 1});
    }
    
    // Build result: columns in sorted order, rows in sorted order,
    // values within same row sorted
    vector<vector<int>> result;
    for (auto& [col, rowMap] : colMap) {
        vector<int> colVals;
        for (auto& [row, vals] : rowMap) {
            sort(vals.begin(), vals.end());  // sort values in same row+col
            for (int v : vals) colVals.push_back(v);
        }
        result.push_back(colVals);
    }
    
    return result;
}

// WHY map<int, map<int, vector<int>>> instead of unordered_map?
// We need sorted column and row order in the final output.
// std::map<int, ...> gives sorted keys automatically.
// The nested structure: outer map sorted by column, inner map sorted by row.
```

### Template 8: BFS on Implicit Graph — Bus Routes (Multi-Level BFS)

```cpp
// LC 815: Find minimum number of buses to reach destination.
// Key insight: BFS on the STOPS graph is too slow (connects every stop to every other).
// Instead, BFS on ROUTES: treat each bus route as a "node" that you board.
// When you board a route, you can reach any stop on that route in 0 additional buses.
// Moving to a new route = 1 additional bus.

int numBusesToDestination(vector<vector<int>>& routes, int source, int target) {
    if (source == target) return 0;
    
    // Build stop → set of routes that serve this stop
    unordered_map<int, vector<int>> stopToRoutes;
    for (int r = 0; r < (int)routes.size(); r++)
        for (int stop : routes[r])
            stopToRoutes[stop].push_back(r);
    
    // BFS on routes
    queue<int> q;       // routes to explore
    unordered_set<int> visitedRoutes;
    unordered_set<int> visitedStops;
    
    // Initialize: add all routes that serve the source stop
    visitedStops.insert(source);
    for (int r : stopToRoutes[source]) {
        if (!visitedRoutes.count(r)) {
            visitedRoutes.insert(r);
            q.push(r);
        }
    }
    
    int buses = 1;
    while (!q.empty()) {
        int size = q.size();
        
        for (int i = 0; i < size; i++) {
            int route = q.front(); q.pop();
            
            for (int stop : routes[route]) {
                if (stop == target) return buses;  // reached target
                
                if (!visitedStops.count(stop)) {
                    visitedStops.insert(stop);
                    for (int r : stopToRoutes[stop]) {
                        if (!visitedRoutes.count(r)) {
                            visitedRoutes.insert(r);
                            q.push(r);
                        }
                    }
                }
            }
        }
        buses++;
    }
    
    return -1;  // cannot reach target
}

// WHY BFS on routes, not stops?
// If BFS on stops: edge between every pair of stops on the same route = O(n²) edges.
// BFS on routes: each route is visited at most once, each stop is visited at most once.
// Total work: O(sum of route sizes) — linear in total input size.
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

### VARIANT 1: Bottom-Up Level Order Traversal (LC 107)

Same as LC 102, but return the result in reverse order (deepest level first).

```cpp
vector<vector<int>> levelOrderBottom(TreeNode* root) {
    vector<vector<int>> result = levelOrder(root);  // reuse Template 1
    reverse(result.begin(), result.end());
    return result;
}

// Or: use a deque and push to FRONT instead of back at each level.
// deque<vector<int>> result;
// ... same BFS logic ...
// result.push_front(level);  // prepend each level
```

---

### VARIANT 2: Average of Levels (LC 637)

Aggregate per level — the most common variation of basic level order.

```cpp
vector<double> averageOfLevels(TreeNode* root) {
    if (!root) return {};
    
    vector<double> result;
    queue<TreeNode*> q;
    q.push(root);
    
    while (!q.empty()) {
        int size = q.size();
        double sum = 0;
        
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            sum += node->val;
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        
        result.push_back(sum / size);
    }
    
    return result;
}
// Same structure works for: max of each level, min of each level, sum of each level.
// The only thing that changes is what you compute inside the inner loop.
```

---

### VARIANT 3: Symmetric Tree (LC 101) — BFS with Pair Comparison

```cpp
// A tree is symmetric if its left subtree is the mirror of its right subtree.
// BFS approach: queue stores PAIRS of nodes that should be mirrors.

bool isSymmetric(TreeNode* root) {
    if (!root) return true;
    
    queue<pair<TreeNode*, TreeNode*>> q;
    q.push({root->left, root->right});
    
    while (!q.empty()) {
        auto [left, right] = q.front(); q.pop();
        
        if (!left && !right) continue;    // both null → symmetric at this point
        if (!left || !right) return false; // one null, one not → asymmetric
        if (left->val != right->val) return false;  // different values → asymmetric
        
        // Push mirror pairs for the next level
        q.push({left->left,  right->right});  // outer pair
        q.push({left->right, right->left});   // inner pair
    }
    
    return true;
}

// DFS alternative (equally clean, recursive):
// bool isMirror(TreeNode* L, TreeNode* R) {
//     if (!L && !R) return true;
//     if (!L || !R) return false;
//     return (L->val == R->val)
//          && isMirror(L->left, R->right)   // outer
//          && isMirror(L->right, R->left);  // inner
// }
```

---

### VARIANT 4: Maximum Width — The Overflow Pitfall in Detail

This variant gets its own deep-dive because of the subtle overflow bug.

```cpp
// WITHOUT normalization: overflow on a right-skewed tree at depth > 32
// The position of the rightmost node: 2^depth - 1
// At depth 32: 2^32 - 1 = 4,294,967,295 → overflows signed int (max ~2.1 billion)
//
// WITH normalization: at each level, the leftmost node gets position 0.
// The rightmost node's position ≤ number of nodes in that level - 1.
// Since a binary tree at depth d has at most 2^d nodes, the max position ≤ 2^d - 1.
// But the ACTUAL tree has far fewer nodes, so overflow is prevented.
//
// The subtle point: when computing children's positions:
//   left child of node at normPos = 2 * normPos
//   right child = 2 * normPos + 1
// These intermediate values CAN overflow if normPos itself is large.
// Use unsigned long long throughout.

// Example tree that breaks naive approach:
//     1
//      \
//       3
//      / \
//     5   3
// At level 2: positions without normalization: root=1, 3=3, then 5=2*3=6, 3=2*3+1=7
// Width = 7 - 6 + 1 = 2 ✓ (actually correct here, but a deeper right-skewed tree breaks)
```

---

### VARIANT 5: Even Odd Tree (LC 1609)

An "Even-Odd Tree" has even-indexed levels with strictly increasing odd values, and odd-indexed levels with strictly decreasing even values. BFS with level index tracking.

```cpp
bool isEvenOddTree(TreeNode* root) {
    if (!root) return true;
    
    queue<TreeNode*> q;
    q.push(root);
    int level = 0;
    
    while (!q.empty()) {
        int size = q.size();
        int prev = (level % 2 == 0) ? INT_MIN : INT_MAX;  // initial compare value
        
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            int val = node->val;
            
            if (level % 2 == 0) {
                // Even level: values must be odd AND strictly increasing
                if (val % 2 == 0 || val <= prev) return false;
            } else {
                // Odd level: values must be even AND strictly decreasing
                if (val % 2 != 0 || val >= prev) return false;
            }
            
            prev = val;
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        level++;
    }
    
    return true;
}
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Level order traversal | O(n) | O(w) | Visit every node once; queue holds at most one level at a time |
| Right side view | O(n) | O(w) | Same as level order; one extra check per node |
| Zigzag traversal | O(n) | O(w) | Level order + preallocated vector fill |
| Minimum depth (BFS) | O(n) worst | O(w) | Stops at first leaf; worst case = full traversal |
| Connect next right (BFS) | O(n) | O(w) | Visits every node once |
| Connect next right (O(1)) | O(n) | O(1) | Uses existing next pointers, no queue |
| Maximum width (virtual index) | O(n) | O(w) | BFS with pair in queue |
| Vertical order traversal | O(n log n) | O(n) | BFS O(n) + sort within same row/col |
| Symmetric tree check | O(n) | O(w) | Visits every mirror pair once |
| Bus Routes (BFS on routes) | O(sum of route sizes) | O(total stops) | Each route and stop visited once |

where **w = maximum width of the tree** = maximum number of nodes at any single level.

### The O(w) Space Invariant — Say This Out Loud:

"BFS space is O(w) where w is the maximum tree width, NOT O(n) and NOT O(h). In the worst case — a perfect binary tree — the bottom level has n/2 nodes, so w = O(n) and BFS space is O(n). In the best case — a skewed tree — every level has 1 node, w = O(1), and BFS space is O(1). This contrasts with DFS recursion which uses O(h) = O(log n) for balanced trees and O(n) for skewed trees. For perfectly balanced trees: BFS uses O(n) space (wide bottom level), DFS uses O(log n) space (shallow height). For perfectly skewed trees: BOTH use O(n) space (but for different reasons — BFS queue has 1 node per level × n levels = 1 node at a time; DFS stack has n call frames)."

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Not Capturing `size = q.size()` Before the Inner Loop — Mixing Levels

**What they do:** Use a single `while (!q.empty())` loop and process one node at a time without tracking level boundaries. Or they capture `size` inside the inner loop (re-evaluating as nodes are added).

**What goes wrong:** The queue continuously grows as children are added. Without `size`, you process ALL nodes without knowing which level they're on. You get a flat list of all node values, not a level-by-level grouping. For right side view: you record the last node ever processed (deepest rightmost node), not the last of each level.

**The fix:** Capture `int size = q.size()` at the START of each outer loop iteration, before the inner loop begins. This freezes the count of current-level nodes.

```cpp
// WRONG: no level tracking — produces flat list, not level-by-level
vector<vector<int>> levelOrder_Wrong(TreeNode* root) {
    if (!root) return {};
    queue<TreeNode*> q; q.push(root);
    vector<int> flat;
    while (!q.empty()) {
        TreeNode* node = q.front(); q.pop();
        flat.push_back(node->val);          // ALL nodes in one flat vector
        if (node->left)  q.push(node->left);
        if (node->right) q.push(node->right);
    }
    return {flat};  // wrong: returns one "level" with all nodes
}

// CORRECT: capture size before inner loop
vector<vector<int>> levelOrder_Correct(TreeNode* root) {
    if (!root) return {};
    queue<TreeNode*> q; q.push(root);
    vector<vector<int>> result;
    while (!q.empty()) {
        int size = q.size();   // ← CRITICAL: captured BEFORE inner loop starts
        vector<int> level;
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(level);
    }
    return result;
}
```

---

### Mistake 2: Minimum Depth — Taking `min(leftDepth, rightDepth)` Without Handling Null Children

**What they do:** Write `return 1 + min(minDepth(root->left), minDepth(root->right))` and assume null children return 0.

**What goes wrong:** A node with only a left child (no right child) would have `minDepth(root->right) = 0` (null returns 0). Then `min(leftDepth, 0) = 0`, and the function returns `1 + 0 = 1` — treating this non-leaf node as a leaf. The minimum depth is reported as 1 regardless of how deep the actual leaf is.

**The fix:** A "leaf" is a node with NO children. If a node has only one child, the minimum depth goes through that child (there's no path of length 0 to a non-existent child). Check for null children explicitly before taking the minimum.

```cpp
// WRONG: treats nodes with one child as leaves (null side returns 0)
int minDepth_Wrong(TreeNode* root) {
    if (!root) return 0;
    return 1 + min(minDepth_Wrong(root->left), minDepth_Wrong(root->right));
    // If root has only left child: min(leftDepth, 0) = 0 → returns 1. WRONG.
}

// CORRECT: handle one-child nodes explicitly
int minDepth_Correct(TreeNode* root) {
    if (!root) return 0;
    if (!root->left && !root->right) return 1;         // leaf
    if (!root->left)  return 1 + minDepth_Correct(root->right);  // only right
    if (!root->right) return 1 + minDepth_Correct(root->left);   // only left
    return 1 + min(minDepth_Correct(root->left), minDepth_Correct(root->right));
}

// BFS version avoids this problem entirely:
// The first leaf encountered by BFS IS at the minimum depth. No null-child special case.
```

---

### Mistake 3: Maximum Width — Integer Overflow Without Normalization

**What they do:** Store positions as `int` and compute `2 * pos` for left child, `2 * pos + 1` for right child, starting with root at position 1.

**What goes wrong:** For a right-skewed tree of depth 32+, the position value reaches or exceeds 2^31 - 1 (max signed int). The computation `2 * pos` overflows, producing negative values or wrapping around. The width calculation `last - first` then gives a wildly incorrect result or undefined behavior.

**The fix:** Normalize positions at the start of each level by subtracting the first position. This resets the leftmost node to position 0 each level, keeping intermediate computations bounded. Use `unsigned long long` for the position type.

```cpp
// WRONG: no normalization, uses int — overflows on skewed trees
int widthOfBinaryTree_Wrong(TreeNode* root) {
    queue<pair<TreeNode*, int>> q;
    q.push({root, 1});
    int maxW = 0;
    while (!q.empty()) {
        int size = q.size();
        int first = q.front().second, last = first;
        for (int i = 0; i < size; i++) {
            auto [node, pos] = q.front(); q.pop();
            last = pos;  // overflow potential
            if (node->left)  q.push({node->left, 2 * pos});        // overflow!
            if (node->right) q.push({node->right, 2 * pos + 1});   // overflow!
        }
        maxW = max(maxW, last - first + 1);
    }
    return maxW;
}

// CORRECT: normalize by subtracting first position of each level
int widthOfBinaryTree_Correct(TreeNode* root) {
    queue<pair<TreeNode*, unsigned long long>> q;
    q.push({root, 1ULL});
    int maxW = 0;
    while (!q.empty()) {
        int size = q.size();
        unsigned long long first = q.front().second;
        unsigned long long last = first;
        for (int i = 0; i < size; i++) {
            auto [node, pos] = q.front(); q.pop();
            unsigned long long normPos = pos - first;   // normalize!
            last = normPos;
            if (node->left)  q.push({node->left,  2 * normPos});
            if (node->right) q.push({node->right, 2 * normPos + 1});
        }
        maxW = max(maxW, (int)(last + 1));  // last - 0 + 1, since first is 0 after normalize
    }
    return maxW;
}
```

---

### Mistake 4: Connect Next Right Pointers — Connecting Across Levels

**What they do:** In the BFS solution, they link ALL consecutive nodes from the queue — including the last node of one level to the first node of the next level.

**What goes wrong:** `node->next` should be null if the node is the rightmost node at its level. If you link across levels, you create a `next` pointer that crosses level boundaries, corrupting the tree structure. Every subsequent use of `next` pointers will traverse across levels instead of staying horizontal.

**The fix:** Only link `node->next = q.front()` when `i < size - 1` (not the last node of the level). The last node of each level should keep `next = nullptr` (the default).

```cpp
// WRONG: links last of level k to first of level k+1
Node* connect_Wrong(Node* root) {
    if (!root) return nullptr;
    queue<Node*> q; q.push(root);
    while (!q.empty()) {
        int size = q.size();
        for (int i = 0; i < size; i++) {
            Node* node = q.front(); q.pop();
            if (!q.empty()) node->next = q.front();  // WRONG: links across levels!
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
    return root;
}

// CORRECT: only link within the same level
Node* connect_Correct(Node* root) {
    if (!root) return nullptr;
    queue<Node*> q; q.push(root);
    while (!q.empty()) {
        int size = q.size();
        for (int i = 0; i < size; i++) {
            Node* node = q.front(); q.pop();
            if (i < size - 1) node->next = q.front();  // ONLY if not last in level
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
    return root;
}
```

---

### Mistake 5: Using DFS When the Problem Requires BFS Correctness Guarantee

**What they do:** Solve "minimum depth" with `return 1 + min(dfs(left), dfs(right))` (with the null fix), or solve "right side view" with DFS, and it works on balanced test cases but fails on unbalanced ones.

**What goes wrong:** For minimum depth: the DFS implementation is correct IF you handle null children properly. But many people get the null handling wrong (Mistake 2). For right side view with DFS: if you go RIGHT before LEFT in DFS, you record the first node at each depth, which happens to be the rightmost for balanced trees. For unbalanced trees (e.g., left child is 5 levels deep, right child is 2 levels deep), the DFS records a node from the left subtree for the deeper levels — but a viewer from the right side would see NOTHING at those depths from the left subtree (the right subtree is shorter).

**The fix:** For minimum depth: use BFS — the first leaf encountered IS the minimum depth, zero ambiguity, zero special cases. For right side view: BFS is the correct approach because it processes all nodes at each level in spatial order; the last node at each level is unambiguously the rightmost.

```cpp
// WRONG: DFS right-side view, fails for trees with unequal-depth subtrees
// Consider:  1
//             \
//              2
//               \
//                5
// AND:       1
//           /
//          3
//           \
//            4
// DFS going right-first records first node at each depth.
// For the combined tree, DFS misses nodes that BFS catches correctly.

// CORRECT: BFS right side view
vector<int> rightSideView_Correct(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    queue<TreeNode*> q; q.push(root);
    while (!q.empty()) {
        int size = q.size();
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            if (i == size - 1) result.push_back(node->val);  // actual rightmost
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
    return result;
}
```

---

### Mistake 6: Zigzag — Reversing the BFS Order Instead of the Output Order

**What they do:** For right-to-left levels, they push children RIGHT-then-LEFT into the queue (instead of the normal LEFT-then-RIGHT), trying to make BFS travel right-to-left.

**What goes wrong:** If you push right-then-left, the children of right-to-left levels will themselves be processed in a different order, causing the NEXT level's children to be misaligned. The zigzag pattern ripples incorrectly through the tree.

**The fix:** ALWAYS push children left-then-right into the queue, regardless of the level direction. The BFS traversal order never changes. What changes is how you STORE the collected level — either normally or in reverse (using the `idx = leftToRight ? i : size-1-i` technique from Template 3).

```cpp
// WRONG: changes BFS push order to simulate right-to-left traversal
vector<vector<int>> zigzag_Wrong(TreeNode* root) {
    queue<TreeNode*> q; q.push(root);
    bool leftToRight = true;
    vector<vector<int>> result;
    while (!q.empty()) {
        int size = q.size();
        vector<int> level;
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            if (leftToRight) {               // WRONG: conditional push order
                if (node->left)  q.push(node->left);
                if (node->right) q.push(node->right);
            } else {
                if (node->right) q.push(node->right); // push right first
                if (node->left)  q.push(node->left);  // causes misalignment!
            }
        }
        result.push_back(level);
        leftToRight = !leftToRight;
    }
    return result;
}

// CORRECT: always push left-then-right; only change HOW you store the level
vector<vector<int>> zigzag_Correct(TreeNode* root) {
    queue<TreeNode*> q; q.push(root);
    bool leftToRight = true;
    vector<vector<int>> result;
    while (!q.empty()) {
        int size = q.size();
        vector<int> level(size);
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            int idx = leftToRight ? i : (size - 1 - i);
            level[idx] = node->val;  // fill from appropriate direction
            if (node->left)  q.push(node->left);   // ALWAYS left-then-right
            if (node->right) q.push(node->right);
        }
        result.push_back(level);
        leftToRight = !leftToRight;
    }
    return result;
}
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (build the BFS template reflex)

**1. Binary Tree Level Order Traversal — LC 102 — [Template 1: canonical BFS with `size = q.size()`]**

Why this problem: Every other BFS-on-tree problem is a variant of this one. If you cannot write Template 1 from memory in under 3 minutes — null check, queue init, outer while, `size = q.size()`, inner for loop, pop, process, push children — you will struggle with every subsequent problem in this pattern. This is your penmanship drill.

What to prove: The queue only contains nodes from ONE level at any time (during the inner loop). The `size` variable is captured BEFORE the inner loop modifies the queue. Children added during the inner loop belong to the NEXT outer iteration. The outer while loop runs exactly (number of levels) times.

Target time: 3 minutes. If it takes longer, practice writing this template until it's mechanical.

Edge cases: Empty tree (return `{}` — the `if (!root)` guard handles this). Single node (one iteration: size=1, no children pushed, result=[[root->val]]). Skewed tree (every level has exactly 1 node — the loop runs n times).

---

**2. Minimum Depth of Binary Tree — LC 111 — [BFS stops at first leaf; OR DFS with null-child guard]**

Why this problem: Forces you to understand when BFS is BETTER than DFS (not just different). BFS stops at the first leaf, which IS the minimum depth, with zero ambiguity. The DFS solution requires handling one-child nodes explicitly — a source of common bugs. Do both implementations.

What to prove: For BFS: the `if (!node->left && !node->right) return depth` check identifies a leaf and returns immediately — BFS guarantees this is the shallowest leaf. For DFS: `min(left, right)` is WRONG without null checks. Correct DFS: if only one child exists, don't take the min — take that child's depth.

Target time: 8 minutes (BFS solution). 15 minutes (also implementing DFS correctly).

Edge cases: Root is a leaf (return 1). Root has only a left subtree, no right — the minimum path must go through the left (DFS without null-guard would return 1 incorrectly).

---

### CORE — 4 problems (the essential BFS-on-tree techniques)

**3. Binary Tree Right Side View — LC 199 — [BFS: take the last node of each level]**

Why this problem: The single most frequently asked BFS tree problem at Microsoft and Meta. The key is recognizing that "rightmost visible node" = "last node at each BFS level." The `i == size - 1` condition is the entire solution. Also practice the DFS alternative (recording the first/last seen at each depth) — interviewers sometimes ask for both.

What to prove: BFS processes nodes left-to-right within each level. The last node processed in a level is the rightmost at that level. A viewer from the right side sees exactly one node per level — the rightmost one at each depth.

Target time: 8 minutes.

Edge cases: Single node tree (only the root is visible). Tree where the right subtree is shorter than the left — the deeper levels of the left subtree ARE visible from the right (there's nothing in the right subtree to block them). The BFS naturally handles this; DFS (right-first) fails here.

---

**4. Binary Tree Zigzag Level Order Traversal — LC 103 — [BFS with direction flag; preallocated level vector with fill-direction]**

Why this problem: Tests whether you understand that zigzag is a STORAGE variation, not a traversal variation. The BFS order itself never changes — children are always pushed left-then-right. Only the direction you fill the preallocated level vector changes per level.

What to prove: Always push children left-then-right into the queue regardless of the level's direction. Use `int idx = leftToRight ? i : (size - 1 - i)` to fill the level vector in the correct direction. Toggle the direction flag after each level.

Target time: 12 minutes.

Edge cases: Single node (one level, leftToRight=true, result=[[root->val]]). Two nodes (level 0: [root], level 1: [right, left] or [left, right] depending on which child exists and direction).

---

**5. Populating Next Right Pointers in Each Node — LC 116 — [BFS O(n) space; then O(1) space using next pointers]**

Why this problem: Two-for-one: the BFS solution teaches the level boundary guard (`i < size - 1`), and the O(1) space solution teaches how to use already-established `next` pointers to traverse a level without a queue. The O(1) solution is what Google asks for.

What to prove: BFS: only connect `node->next = q.front()` when `i < size - 1`. The last node of each level retains `next = nullptr`. O(1): use a `leftmost` pointer to track the start of each level, and a `curr` pointer to traverse the current level via `curr->next`. Within each level, establish connections for the NEXT level.

Target time: 10 minutes (BFS). 20 minutes (O(1) space solution for LC 116's perfect binary tree).

Edge cases: Single node (no connections to make, return root). LC 116 guarantees a perfect binary tree, so every internal node has two children. LC 117 (arbitrary tree) requires tracking the "last successfully connected node" as you traverse — harder.

---

**6. Average of Levels in Binary Tree — LC 637 — [BFS: aggregate sum and count per level]**

Why this problem: A warm variation of level order that tests your template's flexibility. The inner loop collects `sum += node->val` instead of appending to a vector. After the inner loop, `result.push_back(sum / size)`. This is the simplest aggregate BFS problem — master it to show you understand the template.

What to prove: `size` is captured before the inner loop, so dividing `sum / size` correctly computes the average for the level. Integer overflow: `sum` should be `double` or `long long` to avoid overflow before dividing by `size`.

Target time: 5 minutes.

Edge cases: Single node (average = root->val). Nodes with large values: if `sum` is `int` and values are close to INT_MAX, `sum` overflows. Use `double sum = 0` or `long long sum = 0`.

---

### STRETCH — 3 problems

**7. Symmetric Tree — LC 101 — [BFS with pair queue; or recursive DFS mirror check]**

Why this problem: Builds the "pair-of-nodes" BFS technique — instead of tracking single nodes, you track PAIRS that should be mirrors. This pattern appears in several tree problems where you compare two subtrees simultaneously. Implement both BFS (pair queue) and DFS (recursive mirror function) versions.

What to prove: BFS: push `{left->left, right->right}` (outer pair) and `{left->right, right->left}` (inner pair) — these are the pairs that must be mirrors. DFS: `isMirror(L, R) = (L->val == R->val) && isMirror(L->left, R->right) && isMirror(L->right, R->left)`.

Target time: 15 minutes.

Edge cases: Root with only one child — returns false immediately (outer while checks `!left || !right`). Tree with all same values — must still check structural symmetry. Null tree (return true — symmetric by definition of an empty tree).

---

**8. Even Odd Tree — LC 1609 — [BFS with level-parity constraints; two different validation rules per level]**

Why this problem: Forces you to use the level index explicitly (not just track a boolean), and to maintain a `prev` value within each level to check strict monotonicity. This is the hardest of the standard BFS variants.

What to prove: Even-indexed levels: values must be ODD and STRICTLY INCREASING left-to-right. Odd-indexed levels: values must be EVEN and STRICTLY DECREASING. Two conditions per level. Use `int prev = INT_MIN or INT_MAX` as the initial comparison value (direction-dependent). Update `prev = val` after each node.

Target time: 18 minutes.

Edge cases: Single node (level 0, value must be odd — check this). Tree with two nodes where the second level value violates parity or monotonicity.

---

**9. Maximum Width of Binary Tree — LC 662 — [Virtual index BFS with normalization to prevent overflow]**

Why this problem: The "gotcha" tree problem. Naive implementation (no normalization) passes most test cases but fails on skewed trees. Forces you to understand virtual positions and integer overflow. The normalization trick (`pos - first` at each level) is the kind of implementation detail that distinguishes careful coders from careless ones.

What to prove: Virtual positions follow the complete binary tree indexing. Normalization: subtract the first position of each level so the leftmost node always gets position 0. Width of a level = last_position - 0 + 1 = last_position + 1. Use `unsigned long long` for position values to avoid overflow even with normalization.

Target time: 20 minutes.

Edge cases: Single node (width = 1). Right-skewed tree — the width is 1 at every level (only one node per level). Perfect binary tree at depth d — width at deepest level = 2^d (valid as long as d ≤ 63 with unsigned long long).

---

### CONTEST — 2 problems

**10. Vertical Order Traversal — LC 987 — [BFS with (node, row, col) tracking; map<int, map<int, vector>> for sorted output]**

Why this is contest level: Combines BFS with a sorting step. Nodes at the same row AND column must be sorted by value. The nested map structure (`map<col, map<row, vector<int>>>`) handles column and row ordering automatically; only same-row-same-column values need explicit sorting. Getting all three tiebreaking conditions right (column → row → value) is the challenge.

What to prove: BFS naturally gives row ordering. Within same column: row is increasing as BFS progresses. Within same row and column: sort values explicitly. Use `map` (not `unordered_map`) for automatic key sorting in the output.

Target time: 25 minutes.

Edge cases: Two nodes at the same position (same row, same column, different values — must appear sorted by value). Root at column 0, row 0.

---

**11. Bus Routes — LC 815 — [BFS on routes, not stops; stop→routes mapping; visited sets for both stops and routes]**

Why this is contest level: Classic BFS graph disguise. The "obvious" BFS on stops creates O(n²) edges (all stops on the same route are connected). The insight — BFS on ROUTES — reduces complexity dramatically. Each route is visited at most once; the number of buses taken = number of route-changes = BFS depth.

What to prove: Build `stopToRoutes` map. Initialize BFS with all routes serving the source stop. For each route dequeued, check all its stops: if we reach `target`, return current bus count. For unvisited stops on this route, enqueue all their routes. Two visited sets: one for routes (to avoid re-enqueuing a route), one for stops (to avoid re-processing a stop's routes).

Target time: 30 minutes.

Edge cases: Source equals target (return 0 immediately). Source and target share the same route (return 1). No path exists (BFS exhausts — return -1).

---

## PATTERN CONNECTIONS

**Connects to Trees DFS (Pattern 08):**
DFS and BFS are complementary. Every tree problem can be solved by one or the other; many can be solved by both. The rule: level-based properties (right side view, minimum depth to leaf, width) → BFS. Subtree-based properties (height, LCA, path sum) → DFS. Understanding where this line is drawn makes both patterns stronger.

**Connects to Graphs BFS/DFS (Pattern 12):**
Tree BFS is graph BFS without a visited set (trees have no cycles). The same queue-based template applies. In graph BFS you must track `visited[]` to avoid re-processing nodes. In tree BFS, the tree structure guarantees you'll never revisit a node (no edges going "up"). Pattern 12's graph BFS is a direct extension of this pattern.

**Connects to Linked Lists (Pattern 07):**
The "Connect Next Right Pointers" O(1) space solution turns each level of a tree into a linked list via `next` pointers, then traverses that list to connect the next level. This combines linked list traversal concepts (following `next` pointers) with tree BFS concepts (processing levels). Knowing both patterns makes the O(1) solution elegant and obvious.

**Connects to Heap (Pattern 10):**
Vertical Order Traversal uses a `map` for sorted column-row grouping. Some solutions use a priority queue (heap) to extract nodes in sorted column order. BFS on K sorted lists (Merge K Sorted Lists, Pattern 07/10) uses the same "queue of frontier nodes" concept. Understanding both makes you more flexible in choosing the right data structure.

**Connects to Shortest Path (Pattern 14):**
BFS on trees gives shortest paths (by depth) in unweighted settings. BFS on general graphs also gives shortest paths for unweighted graphs. Pattern 14's Dijkstra's algorithm is BFS extended to weighted graphs. Bus Routes (LC 815) is a bridge problem: it looks like a shortest path problem but is solved with BFS (all route transitions cost 1 bus, so unweighted BFS is correct).

**Connects to DP on Trees (Pattern 22):**
Some problems can be solved with either BFS (level-by-level iteration) or DP (top-down or bottom-up on the tree). Maximum Width is more natural with BFS. Properties like "sum of all nodes at even depths" could be solved with BFS (level parity) or DFS (pass depth as parameter). Knowing both opens multiple solution paths.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given a binary tree, return the right side view — the nodes visible when you look at the tree from the right side."

**Company type:** Microsoft (appears in ~30% of rounds), Meta, Amazon
**Expected time:** 8 minutes
**What they are testing:** Whether you reach for BFS naturally for level-based problems, the `i == size - 1` condition, ability to explain WHY BFS is correct here

**Red flags that get candidates rejected:**
- Using DFS and not being able to explain why it gives the wrong answer for asymmetric trees — saying "it works because I go right before left" without acknowledging that it fails when the right subtree is shallower than the left
- Not knowing the `size = q.size()` pattern — asking "how do I know when a level ends?" during the interview
- Using a null sentinel (`push(nullptr)` between levels) instead of the cleaner `size` approach — it works but signals you don't know the canonical pattern
- Off-by-one: using `i == size` instead of `i == size - 1`, so you accidentally take the first node of the NEXT level
- Not handling the empty tree edge case (`if (!root) return {}`)

**What they want you to say:**
"This is a BFS problem because we need the rightmost node at each level — which is a level-based property. DFS doesn't naturally give us level boundaries. My approach: standard BFS template with `size = q.size()` at the start of each outer iteration. The inner loop processes exactly `size` nodes (all nodes at the current level). The last node processed (`i == size - 1`) is the rightmost at this level — I add its value to the result. Children are pushed left-then-right so BFS processes each level in spatial order."

**The follow-up:** "What if you wanted the left side view instead?" → Change `i == size - 1` to `i == 0`. Then: "Can you do this with O(h) space instead of O(w) space?" → DFS: pass depth as a parameter, record the first node seen at each depth in a hash map (going left before right for left view, right before left for right view). The hash map acts as `result[depth] = node->val`.

---

### Q2: "Given a binary tree, find the maximum width of the tree. The width of a level is the number of nodes between the leftmost and rightmost non-null nodes, including the null nodes between them."

**Company type:** Google, Amazon (appears in hard tree rounds)
**Expected time:** 20 minutes
**What they are testing:** Whether you understand virtual indexing, integer overflow awareness, normalization technique

**Red flags that get candidates rejected:**
- Counting actual nodes in a level — completely misses the "including null nodes between" requirement
- Not normalizing positions — code passes most test cases but fails on a right-skewed tree with depth > 31 (overflow)
- Using `int` for position instead of `unsigned long long` (or at least `long long`) — overflow
- Not being able to explain why normalization prevents overflow when asked
- Computing `last - first + 1` AFTER the inner loop without realizing `first` was normalized away to 0 (the width = last + 1 after normalization)

**What they want you to say:**
"The key insight: nodes between the leftmost and rightmost nodes COUNT even if they're null. So I can't just count actual nodes in the queue. Instead, I assign virtual positions using complete binary tree indexing: root=1, left child of pos k = 2k, right child = 2k+1. Width of a level = last_position - first_position + 1. To prevent integer overflow (positions grow as 2^depth), I normalize: at the start of each level, I subtract the first position from all positions in that level. The leftmost node becomes position 0, and all other positions are relative to it. I use unsigned long long throughout to handle intermediate computations."

**The follow-up:** "Why does normalization prevent overflow specifically?" → Without normalization, positions reach 2^h where h is tree height. For h=63, 2^63 overflows even unsigned long long. With normalization, the position of any node is at most (number of nodes in that level - 1) ≤ (2^h - 1), but the ACTUAL count in a real binary tree is much smaller. More importantly, normalization resets the leftmost to 0 each level, so intermediate computations never accumulate across levels.

---

### Q3: "Design a solution to find the minimum number of buses a passenger must take to travel from source stop to target stop, given a list of bus routes."

**Company type:** Google (this exact problem has appeared in Google onsite rounds)
**Expected time:** 30 minutes
**What they are testing:** Problem reduction skills (recognizing the hidden graph), BFS on non-obvious graph structure, time complexity analysis

**Red flags that get candidates rejected:**
- Modeling the graph as stops connected by direct edges (O(n²) edges) without recognizing the O(n²) complexity issue — the approach technically works but is too slow for the given constraints
- Not building the `stopToRoutes` reverse mapping — without this, finding "which routes serve this stop" takes O(routes × stops per route) per step
- Having only one visited set (stops only, or routes only) — with only stops, you re-examine the same route from different stops; with only routes, you miss the fact that each route connects many stops in O(1)
- Not returning 0 when `source == target` — the BFS would process routes unnecessarily
- Returning buses count incorrectly — incrementing INSIDE the inner loop instead of AFTER each BFS level

**What they want you to say:**
"The naive graph: connect every stop to every other stop on the same route. This creates O(sum of route_size²) edges — too many. Better model: BFS on ROUTES, not stops. We 'board' a route (cost 1 bus). While on a route, we can reach any stop on that route for free. At each stop we arrive at, we can board any route serving that stop (cost 1 more bus). So: BFS where each route is a node, and two routes are connected if they share a stop. I build a `stopToRoutes` map so I can find all routes at a stop in O(routes at that stop). Two visited sets: visited routes (don't re-enqueue a route) and visited stops (don't re-process a stop's routes). BFS depth = number of buses taken."

**The follow-up:** "What is the time complexity?" → Building `stopToRoutes`: O(total stops across all routes). BFS: each route processed once O(routes), each stop processed once, each stop's routes processed once — total O(total stops across all routes). Overall: O(N) where N = total stops across all routes.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

**Q1:** What does `int size = q.size()` accomplish in the BFS template, and what goes wrong if you skip it?
> **A:** It captures the number of nodes currently in the queue — which equals the number of nodes at the CURRENT level. The inner `for` loop processes exactly `size` nodes (all of the current level). Without it, you process a flat BFS without level boundaries — no way to know when one level ends and the next begins. For right side view: without `size`, you can't identify the last node of each level. For level order traversal: you produce one flat list instead of a 2D vector.

**Q2:** BFS guarantees the minimum depth to a leaf. Why can't DFS make the same guarantee without additional logic?
> **A:** DFS explores a complete branch (root to leaf) before trying any other branch. The first leaf DFS finds may be very deep (down one subtree) while a shallower leaf exists in another subtree. DFS has no concept of "depth-first among depths" — it goes deep fast. BFS explores ALL nodes at depth 1 before ANY node at depth 2, so the first leaf encountered by BFS is provably at the minimum depth. DFS can find the minimum depth, but it must explore the ENTIRE tree (or use pruning) to guarantee the minimum.

**Q3:** In the zigzag traversal, why must you ALWAYS push children left-then-right into the queue regardless of the current level's direction?
> **A:** The BFS queue represents the CURRENT LEVEL's processing order. Children pushed into the queue form the NEXT level. If you push right-then-left for odd levels, the next-next level's children will also be mis-ordered, cascading errors through the tree. The zigzag direction only affects HOW you display (store) the current level's values, not the traversal order. The level vector is filled using `level[leftToRight ? i : size-1-i]` — changing the storage index, not the BFS order.

**Q4:** Two trees: (A) a perfect binary tree of depth 20, (B) a right-skewed tree of depth 20. Compare BFS space complexity for each.
> **A:** (A) Perfect binary tree depth 20: bottom level has 2^20 ≈ 1 million nodes. BFS queue holds the entire bottom level at once: O(2^20) = O(n) space. (B) Right-skewed tree depth 20: each level has exactly 1 node. BFS queue never holds more than 1 node at a time: O(1) extra space. General formula: BFS space = O(w) where w is the maximum width. Perfect binary tree w = n/2, skewed tree w = 1. DFS space for both: O(h) — O(20) = O(1) for the skewed tree, and O(20) = O(log n) for the perfect tree.

**Q5:** In the maximum width problem, you normalize positions by subtracting the first position of each level. After normalization, what is the formula for the level's width?
> **A:** After normalization, the leftmost node at the level has position 0. If the rightmost node at that level has normalized position `last`, the width = `last - 0 + 1 = last + 1`. Without normalization, width = `last_position - first_position + 1`. The normalization just makes `first_position = 0` implicitly, simplifying the computation.

**Q6:** In "Connect Next Right Pointers" with O(1) space, how do you traverse the current level to build the next level's connections without a queue?
> **A:** Use the `next` pointers that were established at the PREVIOUS step. Start with `leftmost = root` (leftmost node of the current level). Use a `curr` pointer initialized to `leftmost`. To traverse the current level: `curr = curr->next`. For each `curr`, connect its children: `curr->left->next = curr->right` (within the same parent), and `curr->right->next = curr->next->left` (across parents). After finishing the current level, advance `leftmost = leftmost->left` to start the next level. The already-established `next` pointers make the traversal O(1) space.

**Q7:** What distinguishes "Bus Routes" from a standard BFS shortest-path problem, and what is the key modeling decision that makes it efficient?
> **A:** Standard BFS shortest-path: each step costs 1 unit. Bus Routes: each step is "board a new bus" regardless of how many stops you traverse on that bus. The key modeling decision is to BFS on ROUTES (not stops): a route is a "node" you can be "on", and transitioning to a new route costs 1 unit (1 more bus). This avoids the O(n²) edge explosion from connecting every stop to every other stop on the same route. The `stopToRoutes` reverse mapping lets you efficiently find all routes at any stop in O(routes at that stop) time.

---

## C++ BFS CHEAT SHEET

```cpp
// ============================================================
// THE CANONICAL BFS TEMPLATE — MEMORIZE THIS
// ============================================================
vector<vector<int>> levelOrder(TreeNode* root) {
    if (!root) return {};
    vector<vector<int>> result;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int size = q.size();       // ← ALWAYS capture size first
        vector<int> level;
        for (int i = 0; i < size; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(level);
    }
    return result;
}

// ============================================================
// RIGHT SIDE VIEW (change i==size-1 to i==0 for left view)
// ============================================================
// if (i == size - 1) result.push_back(node->val);

// ============================================================
// ZIGZAG (preallocate level, use direction-based index)
// ============================================================
// vector<int> level(size);
// int idx = leftToRight ? i : (size - 1 - i);
// level[idx] = node->val;
// leftToRight = !leftToRight;   // after each level

// ============================================================
// MINIMUM DEPTH (BFS: return on first leaf)
// ============================================================
// if (!node->left && !node->right) return depth;

// ============================================================
// CONNECT NEXT RIGHT (BFS: guard against cross-level linking)
// ============================================================
// if (i < size - 1) node->next = q.front();

// ============================================================
// MAXIMUM WIDTH (normalize positions to prevent overflow)
// ============================================================
// queue<pair<TreeNode*, unsigned long long>> q;
// q.push({root, 1ULL});
// unsigned long long first = q.front().second;   // at start of each level
// unsigned long long normPos = pos - first;       // normalize
// children: {left, 2*normPos}, {right, 2*normPos+1}
// width = last_normPos + 1

// ============================================================
// VERTICAL ORDER (map<col, map<row, vector<int>>>)
// ============================================================
// queue<tuple<TreeNode*, int, int>> q;  // node, row, col
// q.push({root, 0, 0});
// colMap[col][row].push_back(val);
// left child: {node->left, row+1, col-1}
// right child: {node->right, row+1, col+1}
// sort values within same row+col

// ============================================================
// SYMMETRIC TREE (BFS with pair queue)
// ============================================================
// queue<pair<TreeNode*, TreeNode*>> q;
// q.push({root->left, root->right});
// check: if both null → continue; if one null → false; if vals differ → false
// push: {left->left, right->right} and {left->right, right->left}

// ============================================================
// BFS ON IMPLICIT GRAPH (Bus Routes pattern)
// ============================================================
// Build stopToRoutes map first.
// BFS on ROUTES (not stops).
// Two visited sets: visitedRoutes and visitedStops.
// When popping a route: check all its stops; push unvisited routes at each stop.
// Increment bus count after each BFS level.

// ============================================================
// LEVEL ORDER — BOTTOM UP (LC 107)
// ============================================================
// Same as Template 1, then: reverse(result.begin(), result.end());

// ============================================================
// COMPARE BFS VS DFS FOR COMMON TREE PROBLEMS
// ============================================================
// Level order traversal     → BFS (natural)
// Right / Left side view    → BFS preferred; DFS alternative exists
// Minimum depth             → BFS (stops at first leaf, no null-child bug)
// Maximum depth             → DFS preferred (O(h) space vs BFS O(w) space)
// Zigzag traversal          → BFS (level grouping is natural)
// Diameter / path sum       → DFS (subtree aggregation, bottom-up)
// LCA                       → DFS (postorder search)
// Maximum width             → BFS (virtual index)
// Symmetric                 → either (BFS pair queue or DFS mirror recursion)
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. Write Template 1 (canonical BFS) from memory three times — on paper, no reference. Make `int size = q.size()` the first line of every outer loop, automatically.

2. For minimum depth (LC 111): implement BOTH the BFS solution and the correct DFS solution (with explicit null-child guards). Compare them side by side. Understand why the naive DFS (`min(left, right)`) fails.

3. For maximum width (LC 662): first write the version WITHOUT normalization. Then submit — watch it fail on a skewed tree test case. Then fix it with normalization. The deliberate failure permanently cements why normalization matters.

4. In REVISION-TRACKER.md, add these for weekly revisit:
   - LC 662 (Maximum Width) — overflow and normalization detail
   - LC 815 (Bus Routes) — BFS on routes, not stops (the modeling insight)
   - LC 116 (Populate Next Right Pointers) — the O(1) space solution

5. Connecting forward to Pattern 12 (Graphs BFS/DFS): tree BFS is graph BFS without a visited array. Before starting Pattern 12, make sure your BFS template is mechanical. In Pattern 12, you'll add `if (visited[next]) continue;` to every BFS step — a trivial addition once the base template is solid.

6. DFS vs BFS decision framework — internalize this:
   - Need level information? → BFS
   - Need subtree information? → DFS
   - Shortest path to a condition? → BFS (first match = shortest)
   - All paths or aggregate across paths? → DFS
   - Space matters and tree is wide? → DFS (O(h) vs BFS O(w))
   - Space matters and tree is deep? → BFS (O(w) vs DFS O(h))

---

## SIGN-OFF CRITERIA

**Tier 1 (Non-negotiable):**
- [ ] Template 1 (canonical BFS with `size = q.size()`) from memory — under 3 minutes, no bugs
- [ ] Right side view — `i == size - 1` condition, correct for asymmetric trees
- [ ] Can explain WHY BFS is correct for right side view but DFS can be wrong

**Tier 2:**
- [ ] LC 111 (Minimum Depth) — BFS version with early termination at first leaf
- [ ] LC 103 (Zigzag) — correct index `leftToRight ? i : size-1-i`, always push left-then-right
- [ ] LC 116 (Next Right Pointers) — BFS solution with `i < size - 1` guard

**Tier 3:**
- [ ] LC 662 (Maximum Width) — normalization understood and implemented, unsigned long long
- [ ] LC 101 (Symmetric Tree) — BFS pair queue version AND recursive DFS version
- [ ] LC 987 (Vertical Order) — map-based column/row grouping

**Tier 4 (attempt before sign-off):**
- [ ] LC 815 (Bus Routes) — BFS on routes, stopToRoutes mapping, two visited sets

**Sign-off test:** I give you one unseen BFS-on-tree problem. You must:
1. Identify whether it needs BFS or DFS, and state why
2. Identify which BFS template variant applies
3. Write complete C++ solution
4. State time and space complexity in terms of n (nodes) AND w (max width) AND h (height)

Type `SIGN OFF P09` when ready.

---

*Pattern 09 Complete — Trees: BFS and Level Order*
*Next: Pattern 10 — Heap and Priority Queue*
