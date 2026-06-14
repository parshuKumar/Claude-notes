# Pattern 08 — Trees: DFS Complete
## Difficulty Level: CORE
## Interview Frequency: ALWAYS
## Estimated time to master: 5-6 days
## Problems in this set: 11 problems

---

## THE GRANDMASTER'S HONEST TAKE

Trees are where recursion stops being a trick and becomes a way of thinking.

Every array problem, every string problem, every graph problem — you can get by with loops and hash maps and mostly avoid "real" recursion. Trees will not let you. Trees force you to think recursively: *what does this subtree tell me, and what do I tell my parent?*

The single most important question in every tree problem is: **what does my recursive function return to the node that called it?**

Get this question wrong — or dodge it by reaching for a global variable when you shouldn't — and your code will be wrong in subtle ways that your small test cases won't catch. The code will look right. It will pass a 3-node example. And then it will silently give the wrong answer on a 50-node tree because the "return value from subtree" logic is broken.

**At your 1600 level, you:**
- Can compute the maximum depth of a binary tree (LC 104) — this is easy for you
- Can invert a binary tree (LC 226) — also easy
- Struggle with LC 236 (LCA of binary tree) under time pressure — you get the idea but your implementation has subtle bugs in the base case or the two-subtree combination
- Cannot reliably solve LC 124 (Binary Tree Maximum Path Sum) in 25 minutes — the "split vs extend" insight is not yet automatic for you
- Write BST validation wrong: you check `node->val > node->left->val` (direct parent-child), which fails for trees like `[5, 4, 6, null, null, 3, 7]` where 3 is in the right subtree of 5 and 3 < 5 violates the BST invariant, but your check only sees `3 < 6` (direct parent) and passes
- Get confused about when to use preorder vs inorder vs postorder — you pick one and hope

**After this document, you will:**
- Know exactly which traversal order applies to which class of problems, AND WHY — this becomes a mechanical decision
- Have the "return value from subtree" pattern as your default mental model for any tree problem that asks about subtree properties
- Implement all six traversal variants (3 recursive + 3 iterative) without reference
- Solve BST validation correctly with bounds propagation — guaranteed, forever
- Understand the "split" insight for any-to-any path problems — this is what makes LC 124 tractable and it shows up in DP-on-trees problems too

**The interview data:**
- Trees appear in 70%+ of all technical screening rounds at Amazon, Google, Microsoft
- LC 124 (Max Path Sum) and LC 236 (LCA) appear more than any other tree problem at FAANG
- BST validation (LC 98) appears in nearly every Microsoft interview loop
- Serialize/Deserialize (LC 297) is Google's favorite tree design problem
- If you can solve these four cold, you will pass every standard tree DFS round

---

## ELI5 — THE TREE THINKING ANALOGY

### The Company Org Chart

Imagine a company. The CEO is at the top. Each manager has a team. Each team member may have sub-teams. If you want to know "what is the total headcount in Alice's department," you ask each of Alice's managers for THEIR headcount, add them up, and add Alice herself. Then you report to the CEO.

This is **postorder DFS**: process children first (get headcount from sub-departments), then process the current node (add yourself, report to your boss). The answer "flows UP" from leaves to root.

This is the dominant pattern in tree problems. When you see "what property does each subtree have?", you're computing headcounts — process children first, aggregate, report up.

### Top-Down vs Bottom-Up

**Top-down (preorder):** The CEO tells Alice "you're managing the East region." Alice tells her managers "you're managing the East region's districts." Information flows DOWN. Each node gets information FROM its parent and acts on it.

Example: Count Good Nodes (LC 1448). You pass the "maximum value seen so far on the path from root" downward. Each node receives this from its parent, decides if it's a "good" node, then passes the updated maximum to its children.

**Bottom-up (postorder):** Each leaf tells its manager "I'm a leaf, my height is 1." Each manager tells their boss "the tallest subtree under me is 3 levels deep." Information flows UP. Each node gets information FROM its children and computes something to report to its parent.

Example: Diameter of Binary Tree (LC 543). Each node asks its children "what is your height?" Then computes the diameter passing through itself as `leftHeight + rightHeight`.

**The confusion people have:** They use global variables when they should return values. Using a global works (usually), but it creates hidden dependencies, is error-prone in multi-function solutions, and interviewers WILL ask "can you do this without a global?" If you can't, it signals shallow understanding.

### The Traversal Orders as Reading Strategies

**Preorder (root → left → right):** Read the title of a book before reading chapters. You see the node before you see what's underneath it. Good for: printing tree structure, serialization (you need to know the root before you can reconstruct children), problems where you pass information downward.

**Inorder (left → root → right):** Read all sub-sections of chapter 1, then the chapter 1 heading, then sub-sections of chapter 2. For BSTs: this visits nodes in sorted (ascending) order. **This is THE most important property of inorder traversal.** If the problem involves a BST and you need sorted order, inorder gives it for free.

**Postorder (left → right → root):** Read all sub-chapters before reading the chapter summary. You process children before their parent. Good for: computing height, computing diameter, LCA, any problem where a node's answer depends on its children's computed answers.

**The mnemonic:** Pre = root comes FIRST (before children). In = root is IN THE MIDDLE (between children). Post = root comes LAST (after children).

### The "What Do I Return?" Question

Before writing any tree function, answer this question: "When I finish processing a subtree, what single value do I hand back to the node above me?"

For height: I hand back the height of my subtree. For LCA: I hand back the LCA if found in my subtree, or null if not found. For max path sum: I hand back the best single-arm extension from my root node upward.

If you can answer this question clearly, the code writes itself. If you can't, stop and think longer — don't start typing.

---

## THE PATTERN — FORMALLY DEFINED

### What this pattern solves:

**Traversal (enumerate all nodes):**
1. Preorder traversal — root before children (recursive and iterative)
2. Inorder traversal — left subtree, then root, then right subtree; BST gives sorted order (recursive and iterative)
3. Postorder traversal — children before root (recursive and iterative)

**Bottom-up Aggregation (collect from children, report to parent):**
1. Height and depth of binary tree
2. Diameter of binary tree
3. Balance check (is height difference ≤ 1 at every node?)
4. Any property that depends on children's computed values

**BST Operations (exploit the ordered structure):**
1. Search in BST (navigate left/right by value)
2. Insert into BST
3. Validate BST — the bounds-propagation approach (NOT direct parent-child comparison)
4. Kth smallest element (inorder = sorted)

**Lowest Common Ancestor:**
1. LCA in BST — navigate using BST property (O(h) time, O(1) space iteratively)
2. LCA in general binary tree — postorder search ("find p in my subtrees and return it")

**Serialize and Deserialize:**
1. Preorder with null markers — the canonical approach; root-first enables reconstruction

**Path Problems:**
1. Root-to-leaf path problems — pass running sum/state DOWNWARD as parameter
2. Any-to-any path problems — the "split" insight: through-path uses both arms; returned value uses only one arm

### The invariant:

Before writing any recursive tree function, explicitly answer:
1. What does this function return to its parent?
2. What information does it pass down to its children?
3. What does it do with the results from its children?

If you cannot answer all three, you don't have a complete solution yet. Write the answers in a comment before writing the code.

---

## WHEN TO RECOGNIZE THIS PATTERN

**Signal 1:** "Traverse the tree" or "visit all nodes" or "print nodes in [some] order"
→ Direct traversal problem. The order reveals which traversal: "sorted order in BST" = inorder. "Parent before children" = preorder. "Children before parent" = postorder. For "convert recursive to iterative" — explicit stack, using Templates 4, 5, or 6.

**Signal 2:** "Compute a property for each subtree" — height, diameter, balance check, sum of nodes, path count
→ Bottom-up postorder DFS. Your function returns the relevant property of the subtree rooted at the current node. Children's recursive calls come BEFORE you compute the current node's value. The fundamental structure: `int leftAns = f(root->left); int rightAns = f(root->right); return combine(leftAns, rightAns, root->val);`

**Signal 3:** "Given a BST, find/return something in sorted order" or "find the Kth element in a BST"
→ Inorder traversal of the BST, which gives nodes in ascending order. Either collect all values into a vector (simpler, O(n) space), or count during traversal with early termination when the Kth element is reached (O(h) space, terminates early).

**Signal 4:** "Find the lowest common ancestor" of two given nodes
→ Two separate approaches: (A) If the tree is a BST: compare both target values with current node — if one is less and one is greater (or one equals current), current IS the LCA. Implement iteratively for O(1) space. (B) If it is a general binary tree: postorder — return non-null from a subtree if p or q (or their LCA) was found there. If both subtrees return non-null, current node is the LCA.

**Signal 5:** "Is there a root-to-leaf path with sum X?" or "find all root-to-leaf paths satisfying a condition"
→ Top-down DFS. Pass the remaining target (or accumulated path) downward as a parameter. At each leaf, check if the condition is satisfied. Backtrack when returning up (remove the current node from the accumulated path).

**Signal 6:** "Find the maximum/minimum path sum in the tree" where the path can start and end at ANY node
→ The "split" insight. A path THROUGH a node uses both left and right arms for the global maximum computation. But the value returned to the PARENT uses only one arm (a path cannot branch). The two rules: (1) global max = `node->val + max(L, 0) + max(R, 0)`; (2) return to parent = `node->val + max(max(L, 0), max(R, 0))`.

**Signal 7:** "Serialize the tree to a string and reconstruct it from the string"
→ Preorder with null markers. Preorder ensures the root is encoded first — which is the order you need for reconstruction (create root, then left subtree, then right subtree). Use "X" or "#" or "null" as the null marker. Deserialize using a queue of tokens, consumed in preorder order.

**Signal 8:** "Count nodes satisfying a condition based on their path from root" or "pass information downward through the tree"
→ Top-down DFS with state parameter. The state (maximum value on path so far, current depth, current running sum) is passed as a parameter to each recursive call. Children receive the UPDATED state from their parent. Return the aggregated count (not the state).

### Signals that LOOK like tree DFS but are NOT:

**Fake signal 1:** "Find the SHORTEST path between two nodes" in a binary tree
→ This is BFS, not DFS. BFS gives minimum distance in unweighted graphs. DFS explores a full branch before trying alternatives — it finds A path, not the SHORTEST path. (Pattern 09 covers BFS on trees.)

**Fake signal 2:** "Process the tree level by level" or "print nodes at each level" or "right side view"
→ BFS / level-order traversal with a queue. DFS doesn't naturally give you level information unless you pass the depth as a parameter. (Pattern 09 covers this.)

**Fake signal 3:** "Find connected components" in a tree after removing certain edges
→ This becomes a graph problem, potentially requiring Union-Find (Pattern 13) or graph BFS/DFS (Pattern 12). A tree problem assumes the tree structure is fixed.

---

## THE TEMPLATES (C++)

### Template 1: Recursive Preorder Traversal

```cpp
// Preorder: root → left → right
// Use when: process/visit a node BEFORE its children
// Archetypal problems: serialize tree, print structure,
//                      pass information DOWN to children

void preorder(TreeNode* root, vector<int>& result) {
    if (root == nullptr) return;          // base case — null = end of branch
    
    result.push_back(root->val);          // PROCESS root FIRST
    preorder(root->left, result);         // then left subtree
    preorder(root->right, result);        // then right subtree
}

// For top-down state-passing (e.g., current max from root, running sum):
int preorderWithState(TreeNode* root, int stateFromParent) {
    if (!root) return 0;
    
    // Use stateFromParent to process current node
    int isGood = (root->val >= stateFromParent) ? 1 : 0;
    int newState = max(stateFromParent, root->val);   // update state for children
    
    // Pass NEW state to children (not the old state)
    return isGood
         + preorderWithState(root->left,  newState)
         + preorderWithState(root->right, newState);
}
// This exact structure is LC 1448 (Count Good Nodes).
```

### Template 2: Recursive Inorder Traversal

```cpp
// Inorder: left → root → right
// CRITICAL PROPERTY: inorder traversal of a BST visits nodes in SORTED ASCENDING ORDER
// Use when: BST problems needing sorted sequence, Kth smallest, range queries on BST

void inorder(TreeNode* root, vector<int>& result) {
    if (root == nullptr) return;
    
    inorder(root->left, result);          // left subtree FIRST
    result.push_back(root->val);          // PROCESS (in sorted position for BST)
    inorder(root->right, result);         // right subtree LAST
}

// For Kth smallest: count during traversal, stop early when Kth is found
void inorderKth(TreeNode* root, int k, int& count, int& result) {
    if (!root || count >= k) return;      // early exit if already found
    
    inorderKth(root->left, k, count, result);
    
    count++;
    if (count == k) { result = root->val; return; }
    
    inorderKth(root->right, k, count, result);
}

// TRACE for BST [4, 2, 6, 1, 3, 5, 7], inorder:
// Visits: 1 → 2 → 3 → 4 → 5 → 6 → 7 (sorted ascending) ✓
// This is why "Kth smallest in BST" = "Kth element in inorder traversal"
```

### Template 3: Recursive Postorder Traversal

```cpp
// Postorder: left → right → root
// Use when: process children BEFORE their parent (bottom-up aggregation)
// Archetypal problems: height, diameter, balanced check, LCA, max path sum

void postorder(TreeNode* root, vector<int>& result) {
    if (root == nullptr) return;
    
    postorder(root->left, result);        // left subtree FIRST
    postorder(root->right, result);       // right subtree SECOND
    result.push_back(root->val);          // PROCESS root LAST
}

// THE GENERIC "RETURN VALUE FROM SUBTREE" SKELETON:
// This is the structure for the vast majority of tree DFS problems.
int postorderDP(TreeNode* root) {
    if (!root) return BASE_VALUE;              // null → base (0 for height, etc.)
    
    int leftAns  = postorderDP(root->left);    // step 1: get left subtree's answer
    int rightAns = postorderDP(root->right);   // step 2: get right subtree's answer
    
    // step 3: combine to compute THIS node's answer
    int thisAnswer = combineFunction(leftAns, rightAns, root->val);
    
    // Optional: update a global (e.g., for diameter, max path)
    // globalAnswer = max(globalAnswer, sideEffect(leftAns, rightAns, root->val));
    
    return thisAnswer;  // return to parent — may differ from what's in globalAnswer
}
```

### Template 4: Iterative Preorder (explicit stack)

```cpp
// Use when: very deep trees (avoid recursion stack overflow),
//           or when asked to implement iteratively

vector<int> preorderIterative(TreeNode* root) {
    if (!root) return {};
    
    vector<int> result;
    stack<TreeNode*> st;
    st.push(root);
    
    while (!st.empty()) {
        TreeNode* node = st.top(); st.pop();
        result.push_back(node->val);      // PROCESS immediately upon popping
        
        // Push RIGHT first, then LEFT (so LEFT is processed first — LIFO)
        if (node->right) st.push(node->right);
        if (node->left)  st.push(node->left);
    }
    
    return result;
}

// WHY push right before left?
// Stack is LIFO. We want left to be processed before right.
// Push right first so it sits at the bottom; push left so it's on top.
// Left gets popped (and processed) before right. ✓
```

### Template 5: Iterative Inorder (explicit stack) — MEMORIZE THIS

```cpp
// The most common iterative tree question in interviews.
// The "go left as far as possible, then process, then go right" pattern.

vector<int> inorderIterative(TreeNode* root) {
    vector<int> result;
    stack<TreeNode*> st;
    TreeNode* curr = root;
    
    while (curr != nullptr || !st.empty()) {
        // Phase 1: go left as far as possible, pushing every node
        while (curr != nullptr) {
            st.push(curr);
            curr = curr->left;
        }
        
        // Phase 2: no more left — pop and process
        curr = st.top(); st.pop();
        result.push_back(curr->val);
        
        // Phase 3: move to right subtree (if any), repeat from Phase 1
        curr = curr->right;
    }
    
    return result;
}

// TRACE for BST [4 (root), left=2 (left=1, right=3), right=6]:
// Init: curr=4
// Phase 1: push 4, curr=2; push 2, curr=1; push 1, curr=null
// Phase 2: pop 1, result=[1]; Phase 3: curr=null (1 has no right)
// Phase 1: nothing to push (curr is null)
// Phase 2: pop 2, result=[1,2]; Phase 3: curr=3
// Phase 1: push 3, curr=null
// Phase 2: pop 3, result=[1,2,3]; Phase 3: curr=null
// Phase 2: pop 4, result=[1,2,3,4]; Phase 3: curr=6
// Phase 1: push 6, curr=null
// Phase 2: pop 6, result=[1,2,3,4,6] ✓ (sorted for BST)
```

### Template 6: Iterative Postorder (two-stack method)

```cpp
// Method: reversed preorder (root → right → left) stored in s2,
//         then read s2 backward = postorder (left → right → root)
// This is the cleanest approach for interviews.

vector<int> postorderIterative(TreeNode* root) {
    if (!root) return {};
    
    vector<int> result;
    stack<TreeNode*> s1;
    s1.push(root);
    
    while (!s1.empty()) {
        TreeNode* node = s1.top(); s1.pop();
        result.push_back(node->val);       // collect in reverse postorder order
        
        // Push LEFT first, then RIGHT (gives root→right→left in result)
        if (node->left)  s1.push(node->left);
        if (node->right) s1.push(node->right);
    }
    
    // result is currently: root → right subtree → left subtree
    // Reverse it: left subtree → right subtree → root = POSTORDER
    reverse(result.begin(), result.end());
    return result;
}

// WHY does this work?
// Normal preorder: root → left → right (push right, then left)
// Modified preorder: root → right → left (push left, then right)
// Reverse of (root → right → left) = (left → right → root) = POSTORDER ✓
```

### Template 7: Bottom-Up Height Computation — Diameter of Binary Tree

```cpp
// This template is the FOUNDATION for all "bottom-up" tree problems.
// The function returns HEIGHT. It tracks DIAMETER as a side effect.
// This pattern of "return X, track Y globally" recurs throughout tree DFS.

class Solution {
    int diameter = 0;  // global — tracks maximum diameter seen across all nodes
    
    // Returns: height of subtree rooted at `node`
    // Side effect: updates `diameter` with the widest path through `node`
    int height(TreeNode* node) {
        if (!node) return 0;            // null subtree has height 0
        
        int leftH  = height(node->left);    // height of left subtree
        int rightH = height(node->right);   // height of right subtree
        
        // Diameter through this node = edges from leftmost leaf to rightmost leaf
        diameter = max(diameter, leftH + rightH);
        
        // Return HEIGHT (not diameter) — parent needs this for ITS own diameter calc
        return 1 + max(leftH, rightH);
    }
    
public:
    int diameterOfBinaryTree(TreeNode* root) {
        height(root);
        return diameter;
    }
};

// WHY return height but track diameter globally?
// A diameter through a node uses BOTH arms (left + right).
// But a node can only CONTRIBUTE one arm to its parent's diameter.
// (The parent's diameter = parent's value + parent's left arm + parent's right arm;
//  a child is ONE arm of the parent — you can't extend in two directions at once.)
// Returning height gives the parent the "arm length" it needs.
// The global max captures the widest diameter seen anywhere.

// THIS SAME STRUCTURE applies to:
// - Balanced Binary Tree: check |leftH - rightH| <= 1 at each node, return height or -1
// - Maximum Depth: just return 1 + max(leftH, rightH) directly from main function
// - Binary Tree Maximum Path Sum: track global max of (val + leftGain + rightGain)
```

### Template 8: BST Validation — Bounds Propagation (The ONLY Correct Way)

```cpp
// The WRONG approach (checking only parent-child) fails for:
// [5, 4, 6, null, null, 3, 7]
// Node 3 is in the right subtree of 5 (via 6's left child).
// Direct check: 3 < 6 ✓ — passes incorrectly.
// BST requirement: 3 must be > 5 (went RIGHT from 5) — fails correctly.

// CORRECT: every node must satisfy ALL bounds from ALL ancestors
bool isValidBST(TreeNode* root) {
    return validate(root, LLONG_MIN, LLONG_MAX);
}

bool validate(TreeNode* node, long long minVal, long long maxVal) {
    if (!node) return true;   // null subtree is always valid
    
    // Strict inequality: BST has no duplicate values
    if (node->val <= minVal || node->val >= maxVal) return false;
    
    // Left child: all values must be < node->val → maxVal tightens
    // Right child: all values must be > node->val → minVal tightens
    return validate(node->left,  minVal,      node->val)
        && validate(node->right, node->val,   maxVal);
}

// WHY long long instead of int for bounds?
// If the tree contains INT_MIN as a node value and we use int bounds,
// the initial minVal = INT_MIN means we'd check `node->val <= INT_MIN` which
// is always false for INT_MIN — the constraint is lost.
// LLONG_MIN and LLONG_MAX provide safe outer boundaries that no int value equals.

// TRACE for [5, 4, 6, null, null, 3, 7]:
// validate(5, -∞, +∞): 5 in range ✓
//   validate(4, -∞, 5): 4 in range ✓ → leaves both null → true
//   validate(6, 5, +∞): 6 in range ✓
//     validate(3, 5, 6): 3 ≤ 5 → RETURN FALSE ← catches the bug ✓
//     (short-circuit: validate(7, ...) is never called)
// Result: false ✓
```

### Template 9: LCA — General Binary Tree (Postorder Search)

```cpp
// The approach: postorder DFS, returning non-null if p, q, or their LCA is found.
// A node is the LCA if p is found in one of its subtrees and q in the other.

TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    // Base case 1: null → p and q are not in this direction
    // Base case 2: found p or q → return it (so the caller knows it was found here)
    if (!root || root == p || root == q) return root;
    
    TreeNode* left  = lowestCommonAncestor(root->left,  p, q);
    TreeNode* right = lowestCommonAncestor(root->right, p, q);
    
    // Both sides found something → p is on one side, q on the other
    // Current root is the first node to "see both" → it IS the LCA
    if (left && right) return root;
    
    // Only one side found something → both p and q are on that side
    // Propagate that side's result upward (it's either the LCA or one of p/q)
    return left ? left : right;
}

// WHY does `root == p` return early without searching p's subtree for q?
// The problem GUARANTEES both p and q exist in the tree.
// If we find p at this node, q is somewhere in the tree.
// When the PARENT of this node gets `left = p` (or `right = p`) back,
// and the OTHER subtree also returns something (q or their LCA),
// the parent correctly identifies itself as the LCA.
// If q is in p's subtree: the function returns p, and p's PARENT sees
// p returned from one side and null from the other side → p is the LCA ✓

// TRACE for tree [3,5,1,6,2,0,8], LCA(5, 1):
// LCA(3):
//   LCA(5): root == p → return 5       (left result = 5)
//   LCA(1): root == q → return 1       (right result = 1)
//   left=5, right=1: both non-null → return 3 ✓
```

### Template 10: LCA — BST (Exploit BST Property, Iterative O(1) Space)

```cpp
// In a BST: the LCA is the FIRST node where p and q "split" directions.
// If both < current → both in left subtree → LCA is on the left
// If both > current → both in right subtree → LCA is on the right
// Otherwise (one ≤ current ≤ other, or one equals current) → current IS the LCA

TreeNode* lowestCommonAncestorBST(TreeNode* root, TreeNode* p, TreeNode* q) {
    while (root) {
        if (p->val < root->val && q->val < root->val) {
            root = root->left;    // both in left subtree
        } else if (p->val > root->val && q->val > root->val) {
            root = root->right;   // both in right subtree
        } else {
            return root;          // split point → current node is the LCA
        }
    }
    return nullptr;  // not reached if p and q are guaranteed in the tree
}

// TIME: O(h) — each step moves one level down the BST
// SPACE: O(1) — iterative, no recursion stack
// Compare to general LCA: O(n) time, O(h) space — BST version is strictly better
```

### Template 11: Serialize and Deserialize Binary Tree

```cpp
// Preorder serialization with "X" as null markers.
// Serialize: root first (preorder), "X" for null children.
// Deserialize: consume tokens in preorder order using a queue.

class Codec {
public:
    string serialize(TreeNode* root) {
        if (!root) return "X";
        return to_string(root->val) + ","
             + serialize(root->left)  + ","
             + serialize(root->right);
    }
    
    TreeNode* deserialize(string data) {
        queue<string> tokens;
        stringstream ss(data);
        string token;
        while (getline(ss, token, ',')) tokens.push(token);
        return buildTree(tokens);
    }
    
private:
    TreeNode* buildTree(queue<string>& tokens) {
        string val = tokens.front(); tokens.pop();
        if (val == "X") return nullptr;       // null marker → null node
        
        TreeNode* node = new TreeNode(stoi(val));
        node->left  = buildTree(tokens);      // build left subtree first (preorder)
        node->right = buildTree(tokens);      // then right subtree
        return node;
    }
};

// WHY preorder (not inorder or postorder) for serialization?
// Preorder visits the ROOT first. During deserialization, you need to CREATE the root
// before you can attach children. With inorder, the root appears in the middle —
// you don't know which token is the root without additional structure.
// Preorder's root-first order matches the "create root, then children" construction.

// EXAMPLE: tree [1, left=2, right=3]:
// Serialize: "1,2,X,X,3,X,X"
// Deserialize:
//   take "1" → create node(1)
//     left → take "2" → create node(2)
//       left  → take "X" → null
//       right → take "X" → null
//     right → take "3" → create node(3)
//       left  → take "X" → null
//       right → take "X" → null
// Result: original tree ✓
```

### Template 12: Any-to-Any Path Maximum — The "Split" Insight

```cpp
// LC 124: Binary Tree Maximum Path Sum
// A valid path can start and end at any node; it cannot visit a node twice;
// and it cannot branch (it's a sequence, not a tree).

class Solution {
    int globalMax = INT_MIN;  // initialized to INT_MIN — handles all-negative trees
    
    // Returns: the best SINGLE-ARM extension from this node upward
    //          (one direction only: left arm, or right arm, or just this node)
    // Side effect: updates globalMax with the best THROUGH-path at this node
    int maxGain(TreeNode* node) {
        if (!node) return 0;
        
        // max(gain, 0): if an arm is negative, don't use it (start/end the path here)
        int leftGain  = max(maxGain(node->left),  0);
        int rightGain = max(maxGain(node->right), 0);
        
        // Through-path at this node: uses BOTH arms
        int throughPath = node->val + leftGain + rightGain;
        globalMax = max(globalMax, throughPath);
        
        // Return to parent: ONLY ONE arm (a path cannot branch at this node)
        return node->val + max(leftGain, rightGain);
    }
    
public:
    int maxPathSum(TreeNode* root) {
        globalMax = INT_MIN;  // MUST reset — don't use 0 (fails for all-negative trees)
        maxGain(root);
        return globalMax;
    }
};

// THE SPLIT INSIGHT (the core of this problem):
// When computing the best path THROUGH a node, we can use BOTH arms
//   → left arm + node + right arm (forms an inverted V shape, valid path)
// When RETURNING to the parent, we can only extend in ONE direction
//   → parent + one arm of ours forms a valid path
//   → if we returned both arms, the parent would create a Y-shape: INVALID path
//
// max(leftGain, 0):
//   If the left subtree's best path is negative, exclude it entirely.
//   "Start the path here" rather than dragging in a negative contribution.
//   This enables paths that begin/end in the middle of the tree.
//
// globalMax initialized to INT_MIN (not 0):
//   If ALL nodes are negative, the answer is the single least-negative node.
//   Initializing to 0 would incorrectly return 0 (pretending we can take an empty path).
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

### VARIANT 1: Construct Binary Tree from Preorder and Inorder (LC 105)

The key insight: in preorder, the FIRST element is always the root. In inorder, the root SPLITS the array into left subtree (everything left of root) and right subtree (everything right of root).

```cpp
class Solution {
    unordered_map<int, int> inorderIndex;  // value → index in inorder (for O(1) lookup)
    
public:
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        for (int i = 0; i < (int)inorder.size(); i++)
            inorderIndex[inorder[i]] = i;
        return build(preorder, 0, preorder.size() - 1, 0, inorder.size() - 1);
    }
    
private:
    TreeNode* build(vector<int>& pre, int preL, int preR, int inL, int inR) {
        if (preL > preR) return nullptr;   // no nodes remaining
        
        int rootVal = pre[preL];              // first of preorder = root
        int rootIdx = inorderIndex[rootVal];  // root's position in inorder
        int leftSize = rootIdx - inL;         // number of nodes in left subtree
        
        TreeNode* node = new TreeNode(rootVal);
        // Left subtree: preorder[preL+1 .. preL+leftSize], inorder[inL .. rootIdx-1]
        node->left  = build(pre, preL + 1, preL + leftSize, inL, rootIdx - 1);
        // Right subtree: preorder[preL+leftSize+1 .. preR], inorder[rootIdx+1 .. inR]
        node->right = build(pre, preL + leftSize + 1, preR, rootIdx + 1, inR);
        
        return node;
    }
};

// WHY hash map?
// Finding root in inorder by linear scan = O(n) per call = O(n²) total.
// Hash map gives O(1) per call = O(n) total.

// EXAMPLE: preorder=[3,9,20,15,7], inorder=[9,3,15,20,7]
// Root=3, rootIdx in inorder=1, leftSize=1-0=1
// Left:  preorder=[9],       inorder=[9]       → single node 9
// Right: preorder=[20,15,7], inorder=[15,20,7] → recurse → root=20, left=15, right=7
```

---

### VARIANT 2: Balanced Binary Tree Check (LC 110) — Return -1 as "Invalid" Sentinel

```cpp
// Combine height computation with balance checking using a sentinel return value.
// Returns: height if balanced, -1 if any subtree is unbalanced.

bool isBalanced(TreeNode* root) { return checkHeight(root) != -1; }

int checkHeight(TreeNode* node) {
    if (!node) return 0;
    
    int leftH = checkHeight(node->left);
    if (leftH == -1) return -1;   // left is unbalanced — short-circuit, propagate up
    
    int rightH = checkHeight(node->right);
    if (rightH == -1) return -1;  // right is unbalanced — short-circuit
    
    if (abs(leftH - rightH) > 1) return -1;  // THIS node unbalanced
    
    return 1 + max(leftH, rightH);  // balanced — return actual height
}

// WHY return -1 instead of using a separate "isBalanced" boolean?
// Returning -1 makes the computation FAIL FAST: the moment any subtree is unbalanced,
// all callers propagate -1 immediately without recursing further.
// A separate boolean would still recurse through the entire tree even after
// finding an imbalance. The -1 sentinel is equivalent to throwing an early exception.
```

---

### VARIANT 3: Path Sum — Root-to-Leaf Paths

```cpp
// Path Sum I (LC 112): does any root-to-leaf path sum to targetSum?
bool hasPathSum(TreeNode* root, int targetSum) {
    if (!root) return false;
    
    // At a leaf: check if this node completes the target
    if (!root->left && !root->right) return root->val == targetSum;
    
    // Recurse: subtract current node's value, check either subtree
    return hasPathSum(root->left,  targetSum - root->val)
        || hasPathSum(root->right, targetSum - root->val);
}

// Path Sum II (LC 113): collect ALL root-to-leaf paths that sum to target
void pathHelper(TreeNode* node, int remaining,
                vector<int>& current, vector<vector<int>>& result) {
    if (!node) return;
    
    current.push_back(node->val);
    remaining -= node->val;
    
    if (!node->left && !node->right && remaining == 0) {
        result.push_back(current);    // found valid path — record it
    }
    
    pathHelper(node->left,  remaining, current, result);
    pathHelper(node->right, remaining, current, result);
    
    current.pop_back();   // BACKTRACK — undo the push before returning to parent
}
// The backtrack (`pop_back`) is the same pattern as in explicit backtracking problems.
// Tree structure provides the "undo" signal naturally: when the function returns,
// we're back at the parent — but we must manually undo our push to the current path.
```

---

### VARIANT 4: BST Insert and Delete

```cpp
// BST Insert: navigate left/right by comparison, insert at null position
TreeNode* insertIntoBST(TreeNode* root, int val) {
    if (!root) return new TreeNode(val);   // null = found the insertion point
    if (val < root->val) root->left  = insertIntoBST(root->left, val);
    else                 root->right = insertIntoBST(root->right, val);
    return root;
}

// BST Delete (LC 450): three cases
// Case 1: node has no children → just delete (return null)
// Case 2: node has one child → replace node with that child
// Case 3: node has two children → replace with inorder successor (leftmost of right subtree)
TreeNode* deleteNode(TreeNode* root, int key) {
    if (!root) return nullptr;
    
    if (key < root->val) {
        root->left = deleteNode(root->left, key);
    } else if (key > root->val) {
        root->right = deleteNode(root->right, key);
    } else {
        // Found the node to delete
        if (!root->left)  return root->right;  // Case 1 or 2 (left missing)
        if (!root->right) return root->left;   // Case 2 (right missing)
        
        // Case 3: two children — find inorder successor (leftmost of right subtree)
        TreeNode* successor = root->right;
        while (successor->left) successor = successor->left;
        root->val = successor->val;                          // copy successor's value
        root->right = deleteNode(root->right, successor->val);  // delete successor
    }
    return root;
}
```

---

### VARIANT 5: Count Good Nodes (LC 1448) — The Pure Top-Down Pattern

```cpp
// A node is "good" if no node on its path from root has a larger value.
// Top-down: pass the maximum value seen so far on the path from root.

int goodNodes(TreeNode* root) {
    return dfs(root, INT_MIN);
}

int dfs(TreeNode* node, int maxSoFar) {
    if (!node) return 0;
    
    int isGood = (node->val >= maxSoFar) ? 1 : 0;
    int newMax  = max(maxSoFar, node->val);  // update max BEFORE passing to children
    
    return isGood
         + dfs(node->left,  newMax)
         + dfs(node->right, newMax);
}

// This is the PURE top-down pattern:
// - Information flows FROM parent TO children (the current max)
// - Children receive the UPDATED max that includes the current node
// - Return value aggregates the total count from each subtree (bottom-up for the count)
// The combination of "pass state down + aggregate counts up" is common in tree DP.
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Operation | Time | Space | Why |
|-----------|------|-------|-----|
| Preorder / Inorder / Postorder traversal | O(n) | O(h) | Visit every node once; recursion stack depth = tree height h |
| Iterative traversal (stack-based) | O(n) | O(h) | Explicit stack holds at most h nodes at any time |
| BST search | O(h) | O(h) recursive, O(1) iterative | Each step goes left or right — eliminates one side |
| BST insert | O(h) | O(h) recursive, O(1) iterative | Find insertion point like search |
| BST delete | O(h) | O(h) recursive | Finding successor is additional O(h) traversal |
| BST validate | O(n) | O(h) | Must visit every node — can't skip based on value alone |
| LCA — general binary tree | O(n) | O(h) | May need to search entire tree |
| LCA — BST (iterative) | O(h) | O(1) | Exploit BST property to navigate directly, no recursion |
| Diameter of binary tree | O(n) | O(h) | Visit every node once |
| Max path sum | O(n) | O(h) | Visit every node once |
| Serialize/Deserialize | O(n) | O(n) | Must encode all n nodes; output string/queue is O(n) |
| Construct from preorder + inorder | O(n) | O(n) | Hash map for O(1) lookup; O(n) maps and O(h) recursion stack |
| Balanced binary tree check | O(n) | O(h) | Visit every node once with short-circuit |

### The O(h) Space Invariant — Say This Out Loud:

"Tree recursion always uses O(h) stack space, where h is the tree height. For a balanced tree, h = O(log n). For a skewed tree (effectively a linked list), h = O(n). When I say 'space O(n),' I mean worst-case for a completely skewed tree. When the problem states the tree is balanced, I say O(log n) space. Iterative traversals with an explicit stack also use O(h) space — the stack holds at most one node per level (the rightmost unprocessed node at each level). The space is O(h) whether the stack is implicit (recursion) or explicit (iterative)."

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Null Dereference — Accessing Children Without Checking Null First

**What they do:** Write `if (root->left->val > root->val)` or access `root->left->val` without first verifying `root->left != nullptr`.

**What goes wrong:** When a node has no left child, `root->left` is `nullptr`. Calling `nullptr->val` is undefined behavior — immediate segfault. This happens for any node that is a leaf, or has only one child. It's the most common crash in tree code.

**The fix:** The FIRST thing every tree function must do is check `if (!root) return ...`. Before accessing any field of a pointer, verify the pointer is non-null. This applies to both the function's own parameter AND any child pointer you're about to dereference.

```cpp
// WRONG: crashes when any node is missing a left or right child
bool isValidBST_Wrong(TreeNode* root) {
    if (!root) return true;
    // CRASH: root->left might be null; root->left->val is undefined behavior
    if (root->left->val >= root->val)  return false;
    // CRASH: root->right might be null
    if (root->right->val <= root->val) return false;
    return isValidBST_Wrong(root->left) && isValidBST_Wrong(root->right);
}

// CORRECT: check null before dereferencing, use bounds propagation
bool validate(TreeNode* node, long long lo, long long hi) {
    if (!node) return true;               // null is always valid
    if (node->val <= lo || node->val >= hi) return false;
    return validate(node->left, lo, node->val)
        && validate(node->right, node->val, hi);
}
```

---

### Mistake 2: BST Validation — Checking Only Direct Parent-Child Relationship

**What they do:** At each node, check only `node->left->val < node->val` and `node->right->val > node->val` (immediate children only).

**What goes wrong:** This misses violations between a node and non-adjacent ancestors. Classic failing case: `[5, 4, 6, null, null, 3, 7]`. Node 3 is in the right subtree of 5 (because it's the left child of 6). The check `3 < 6` passes. But 3 < 5, and since we went RIGHT from 5, everything in that right subtree must be > 5. The violation with ancestor 5 is missed.

**The fix:** Pass min/max bounds down through the recursion. Every node must satisfy ALL bounds from ALL ancestors, not just its parent.

```cpp
// WRONG: only checks direct parent-child — fails for [5,4,6,null,null,3,7]
bool isValidBST_Wrong(TreeNode* root) {
    if (!root) return true;
    if (root->left  && root->left->val  >= root->val) return false;
    if (root->right && root->right->val <= root->val) return false;
    return isValidBST_Wrong(root->left) && isValidBST_Wrong(root->right);
    // Node 3 in right subtree: 3 < 6 ✓ (direct check) but 3 < 5 ✗ (ancestor violation)
}

// CORRECT: bounds propagation catches all ancestor violations
bool validate(TreeNode* node, long long lo, long long hi) {
    if (!node) return true;
    if (node->val <= lo || node->val >= hi) return false;
    return validate(node->left, lo, node->val) && validate(node->right, node->val, hi);
}
bool isValidBST(TreeNode* root) { return validate(root, LLONG_MIN, LLONG_MAX); }
// Node 3: validate(3, 5, 6) → 3 ≤ 5 → returns false ✓
```

---

### Mistake 3: Max Path Sum — Returning Both Arms to Parent (Creating Y-Shape)

**What they do:** In `maxPathSum`, return `node->val + leftGain + rightGain` to the parent after updating the global maximum.

**What goes wrong:** A path cannot branch. If the parent uses both your left and right arms, the result is a Y-shape — three endpoints, not a valid simple path. The global maximum correctly uses both arms for a through-path at the current node. But what you RETURN to the parent must be a single-direction extension.

**The fix:** Update the global max with `node->val + leftGain + rightGain` (both arms). Return to parent `node->val + max(leftGain, rightGain)` (one arm only).

```cpp
// WRONG: returns both arms to parent — creates an invalid Y-shaped "path"
int maxGain_Wrong(TreeNode* node, int& globalMax) {
    if (!node) return 0;
    int L = max(maxGain_Wrong(node->left, globalMax), 0);
    int R = max(maxGain_Wrong(node->right, globalMax), 0);
    globalMax = max(globalMax, node->val + L + R);
    return node->val + L + R;  // WRONG: parent would form a Y-shape using this
}

// CORRECT: return only one arm to parent
int maxGain_Correct(TreeNode* node, int& globalMax) {
    if (!node) return 0;
    int L = max(maxGain_Correct(node->left, globalMax), 0);
    int R = max(maxGain_Correct(node->right, globalMax), 0);
    globalMax = max(globalMax, node->val + L + R);  // through-path: both arms ✓
    return node->val + max(L, R);                   // to parent: one arm only ✓
}
```

---

### Mistake 4: Max Path Sum — Initializing globalMax to 0 Instead of INT_MIN

**What they do:** Initialize `globalMax = 0` before calling the recursion.

**What goes wrong:** If all node values are negative, the correct answer is the least-negative single node (e.g., -1 in `[-3, -1, -2]`). With `globalMax = 0`, the function never updates it (because even the least-negative path is still negative, and 0 > -1), and the function returns 0 — which is wrong. There is no empty path; the problem requires at least one node.

**The fix:** Initialize `globalMax = INT_MIN`. Every non-empty path (including a single negative node) is a valid candidate and will correctly update INT_MIN.

```cpp
// WRONG: globalMax = 0 — fails for all-negative trees
int maxPathSum_Wrong(TreeNode* root) {
    int globalMax = 0;  // WRONG: 0 implies empty path is valid; it's not
    maxGain(root, globalMax);
    return globalMax;  // returns 0 for [-3, -1, -2] but correct answer is -1
}

// CORRECT: globalMax = INT_MIN — all single-node paths are valid candidates
int maxPathSum(TreeNode* root) {
    int globalMax = INT_MIN;  // INT_MIN ensures any real path updates this
    maxGain(root, globalMax);
    return globalMax;
    // For [-3, -1, -2]: maxGain computes -1 at node(-1) as through-path,
    // globalMax updates to -1 ✓
}
```

---

### Mistake 5: LCA — Wrong Base Case (Returning null Instead of root When p or q Found)

**What they do:** When `root == p` or `root == q`, they return `null` instead of `root`, and use a separate global variable to record the LCA.

**What goes wrong:** The recursive propagation relies on non-null returns to signal "I found p (or q) in my subtree." If you return null when finding p, the parent's `left` and `right` results are both null — the `if (left && right) return root` logic never fires. The LCA is never identified.

**The fix:** The base case MUST return `root` (not null) when `root == p || root == q`. This non-null return bubbles up to the LCA node, which sees non-null from both sides and correctly identifies itself.

```cpp
// WRONG: returns null when p or q is found — propagation fails
TreeNode* LCA_Wrong(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root) return nullptr;
    if (root == p || root == q) return nullptr;  // WRONG: should return root
    TreeNode* L = LCA_Wrong(root->left, p, q);
    TreeNode* R = LCA_Wrong(root->right, p, q);
    if (L && R) return root;  // never fires because L and R are always null
    return L ? L : R;         // always returns null — LCA never found
}

// CORRECT: return root when p or q is found — this is the signal to callers
TreeNode* LCA_Correct(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;  // KEY: return root, not null
    TreeNode* L = LCA_Correct(root->left, p, q);
    TreeNode* R = LCA_Correct(root->right, p, q);
    if (L && R) return root;  // fires correctly when p and q are on different sides
    return L ? L : R;
}
```

---

### Mistake 6: Iterative Inorder — Missing `curr = curr->right` After Processing

**What they do:** After popping a node and recording its value, they forget to set `curr = curr->right` to move to the right subtree.

**What goes wrong:** The outer while loop's condition is `curr != nullptr || !st.empty()`. After processing a node, if `curr` isn't updated to the right child, the next iteration immediately enters Phase 1 (`while curr != nullptr`) with the same `curr` that was already null — and the loop terminates early (missing the right subtree of every processed node). In the worst case, the stack re-pushes the same leftward chain, causing an infinite loop.

**The fix:** Phase 3 of the iterative inorder is `curr = curr->right`. This is not optional — it is the instruction to "move to the right subtree and repeat."

```cpp
// WRONG: missing curr = curr->right — loop terminates after visiting leftmost path only
vector<int> inorder_Wrong(TreeNode* root) {
    vector<int> res; stack<TreeNode*> st; TreeNode* curr = root;
    while (curr || !st.empty()) {
        while (curr) { st.push(curr); curr = curr->left; }
        curr = st.top(); st.pop();
        res.push_back(curr->val);
        // MISSING: curr = curr->right;
        // Without this: curr is null, loop re-checks empty stack, exits — misses right subtrees
    }
    return res;
}

// CORRECT: the three-phase structure must be complete
vector<int> inorder_Correct(TreeNode* root) {
    vector<int> res; stack<TreeNode*> st; TreeNode* curr = root;
    while (curr || !st.empty()) {
        while (curr) { st.push(curr); curr = curr->left; }
        curr = st.top(); st.pop();
        res.push_back(curr->val);
        curr = curr->right;  // ← CRITICAL: explore right subtree in next iteration
    }
    return res;
}
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (build the recursive reflex)

**1. Maximum Depth of Binary Tree — LC 104 — [Postorder: return 1 + max(left, right)]**

Why this problem: The "hello world" of tree recursion. If you write `if (!root) return 0; return 1 + max(maxDepth(root->left), maxDepth(root->right));` in under 60 seconds without thinking, your recursion reflex is working. If it takes 3 minutes, the postorder pattern is not yet mechanical. This problem IS the base case for every subsequent tree problem.

What to prove: You cannot compute a node's depth without first knowing its children's depths → postorder. The null base case (`!root → return 0`) handles empty trees AND leaf nodes' missing children. The `1 +` accounts for the current node. `max(left, right)` picks the deeper subtree.

Target time: 2 minutes (should be instant — if it isn't, practice until it is).

Edge cases: Empty tree (return 0 — the base case handles this directly). Single node (both left and right return 0; answer is 1). Completely skewed tree (one branch is always null; the null base case handles it each time).

---

**2. Invert Binary Tree — LC 226 — [Preorder or postorder: swap children at each node]**

Why this problem: Tests whether you can apply a structural transformation recursively. The key insight: if you swap a node's left and right children, then recursively invert each child, the result is a completely mirrored tree. Both preorder (swap first, then recurse into children) and postorder (recurse into children first, then swap) produce the correct result — understanding WHY BOTH work deepens your intuition.

What to prove: After swapping at the root, `root->left` and `root->right` are swapped. Recursing into both (regardless of order) inverts each subtree. You don't need to "think" about which side to recurse into — the tree structure handles it. The one-liner iterative version uses a BFS queue to process nodes level by level.

Target time: 3 minutes.

Edge cases: Empty tree (return null immediately). Single node (swap null and null — no-op, return the node). Two nodes with only one child.

---

### CORE — 4 problems (the essential techniques)

**3. Diameter of Binary Tree — LC 543 — [Bottom-up: return height, track max(left + right) globally]**

Why this problem: The canonical "return ≠ answer" tree problem. Your function needs to return HEIGHT to its parent (so the parent can compute its own diameter), but the actual ANSWER is the maximum diameter seen anywhere. This is the template for all problems where "what I compute for this node" is different from "what I return to my parent." Missing this distinction is the #1 source of wrong answers in tree DFS problems.

What to prove: Height of a node = 1 + max(leftHeight, rightHeight). Diameter THROUGH a node = leftHeight + rightHeight (this counts edges — LC 543 counts edges). The global max tracks the widest diameter across all nodes. Return height to parent (not the diameter) — the parent needs height for its own diameter computation.

Target time: 12 minutes.

Edge cases: Single node (diameter = 0, height = 1 — leftH and rightH are both 0, diameter update is 0 + 0 = 0 ✓). Skewed tree (diameter = 0 for a straight line — each node only has one child, one of leftH/rightH is always 0). Tree with 2 nodes (diameter = 1).

---

**4. Validate Binary Search Tree — LC 98 — [Top-down bounds propagation: validate(node, lo, hi)]**

Why this problem: Guarantees you internalize the bounds-propagation approach forever. This is asked in nearly every Microsoft interview loop. The "obvious" direct parent-child check fails. Getting this right cold is a strong signal that you understand the BST invariant at a deep level.

What to prove: A node at any position in the BST must satisfy ALL bounds imposed by ALL ancestors. Passing `(minVal, maxVal)` down encodes every ancestor's constraint. Use `LLONG_MIN` / `LLONG_MAX` to handle integer edge values. Strict inequality: BST requires `minVal < node->val < maxVal`.

Target time: 15 minutes. On first attempt with bounds approach: 20 minutes is fine.

Edge cases: Tree containing `INT_MIN` or `INT_MAX` as node values — using `INT_MIN/INT_MAX` as bounds fails; `LLONG_MIN/LLONG_MAX` are required. Root is null (return true — empty tree is valid). Tree with equal values — problem states "no duplicates" but verify with problem constraints; strict `<` and `>` handle this.

---

**5. Lowest Common Ancestor of a BST — LC 235 — [Iterative: navigate using BST property]**

Why this problem: The easy version of LCA. Using the BST property to navigate directly — no search needed — is elegant and efficient. Implement it iteratively to achieve O(1) space. This problem is a prerequisite for LC 236 (the harder version).

What to prove: The first node where p and q split directions IS the LCA. "Split directions" means: one value goes left (< current) and one goes right (> current), OR one of them equals the current node. The iterative implementation uses a simple while loop — no recursion stack needed.

Target time: 8 minutes.

Edge cases: p IS the root (root is the LCA — the split condition: `p->val == root->val` satisfies the "else" branch). p is an ancestor of q (p is the LCA — when we reach p, since p < root is false and p > root is false, we return p). Both p and q are the same node (trivially the LCA of itself).

---

**6. Lowest Common Ancestor of Binary Tree — LC 236 — [Postorder: return non-null if p/q found, return root if both sides non-null]**

Why this problem: The hard version of LCA. No BST property — must search the entire tree. The postorder approach is the cleanest possible solution: 4 lines of meaningful code. Must implement it correctly from memory. Interviewers at Google and Meta specifically probe whether you can EXPLAIN the base case and the two-sides logic, not just write it.

What to prove: The base case `if (!root || root == p || root == q) return root` is the key — returning `root` (not null) when p or q is found signals their presence to callers. When both `left` and `right` are non-null, we found p in one subtree and q in the other — current node is the LCA. When only one side is non-null, both are in that subtree — propagate that result upward.

Target time: 20 minutes.

Edge cases: One node is the ancestor of the other (the algorithm handles this correctly via the early base case return — if we reach p before finding q, we return p immediately, and p's ancestor eventually sees p from one side and q from the other, returning p as the LCA). p and q are on the same side (both returns go up through the same branch).

Common mistake: Not understanding why returning early at `root == p` is correct even though q might be in p's subtree. (The answer: when the ancestor of p sees `left = p` and `right = null` or `right = p-based-LCA`, the correct LCA propagates up regardless.)

---

### STRETCH — 3 problems (mastery-level techniques)

**7. Binary Tree Maximum Path Sum — LC 124 — [Split insight: both arms for global max; one arm returned to parent]**

Why this problem: The hardest standard tree DFS problem. THE problem to solve if you want to prove tree mastery. Combines bottom-up recursion with the "split vs extend" insight. Once you can solve this cold in 25 minutes with a correct explanation, your tree DFS is at FAANG level.

What to prove: The split insight — a through-path at a node uses both arms; the return value to the parent uses only one arm. `max(leftGain, 0)` — if a subtree contributes negatively, don't include it (allows paths to start/end mid-tree). `globalMax = INT_MIN` — handles all-negative trees. The answer is NOT always a root-to-leaf path.

Target time: 25 minutes. On first attempt: 35 minutes acceptable. If over 45 minutes, re-read Template 12 and trace through `[-3, -1, -2]` and `[1, 2, 3]` manually.

Edge cases: All negative values (answer is the single least-negative node — requires `INT_MIN` initialization). Single node (globalMax = node->val; return = node->val). Path consisting of a single node in the middle of the tree (valid — max(L,0) and max(R,0) both return 0, through-path = node->val alone).

Common mistakes: (1) Initializing globalMax to 0 — fails for all-negative trees. (2) Returning both arms to parent. (3) Not taking `max(gain, 0)` — negative arms drag the sum down incorrectly.

---

**8. Construct Binary Tree from Preorder and Inorder Traversal — LC 105 — [Divide and conquer: preorder[0] = root, split inorder at root's index]**

Why this problem: Tests deep understanding of what preorder and inorder traversals reveal about tree structure. The "eureka" insight — preorder[0] is always the root, and the root's position in inorder splits the array into left and right subtrees — is the kind of structural insight that interviewers at Google test specifically.

What to prove: Preorder always starts with the root. The root's position in inorder = `rootIdx`. Left subtree has `leftSize = rootIdx - inL` nodes. Both arrays can be split consistently using `leftSize`. Hash map for O(1) root-position lookup makes the total complexity O(n) instead of O(n²).

Target time: 25 minutes.

Edge cases: Tree with only a right chain (`preorder = [1, 2, 3]`, `inorder = [1, 2, 3]`). Single node. The recursion terminates when `preL > preR` — verify this is the correct base case.

---

**9. Serialize and Deserialize Binary Tree — LC 297 — [Preorder + null markers: serialize is trivial; deserialize uses a queue of tokens]**

Why this problem: Tests your ability to design a reversible encoding for a tree. The preorder with null markers approach is elegant and the most common solution in interviews. Serialize is simple recursive preorder with nulls. Deserialize mirrors the serialization using a queue of tokens — each token corresponds to exactly one recursive call.

What to prove: The preorder null-marker encoding is UNIQUE — no two different binary trees produce the same serialization. Null markers distinguish the structure even when values are the same. Deserializing with a queue works because: token order matches preorder, and consuming one token per call (null or not) correctly mirrors the recursive structure.

Target time: 30 minutes.

Edge cases: Empty tree (serialize → "X"; deserialize → null). Single node (serialize → "val,X,X"). All nodes have the same value (nulls uniquely identify structure). The `getline(ss, token, ',')` parsing correctly handles leading/trailing commas and multi-digit numbers.

---

### CONTEST — 2 problems

**10. Kth Smallest Element in a BST — LC 230 — [Inorder with counter; early termination at Kth]**

Why this is contest level: Requires combining the BST-inorder-is-sorted insight with efficient early-termination. The naive approach (collect all into array, return index k-1) is O(n) time AND space. The optimized approach uses O(h) space and terminates as soon as the Kth element is visited. The follow-up asks about frequent modifications — requiring augmented BST with subtree size counts.

What to prove: Inorder traversal of BST visits elements in ascending order. Counting during inorder: when `count == k`, the current node is the Kth smallest. The `if (!node || count >= k)` guard prevents unnecessary recursion after finding the answer.

Target time: 15 minutes.

Edge cases: k = 1 (leftmost node — smallest). k = total number of nodes (rightmost node — largest). BST with a single node and k = 1. A BST where the Kth smallest is the root.

---

**11. Count Good Nodes in Binary Tree — LC 1448 — [Top-down: pass maxSoFar from root to each node]**

Why this is contest level: Tests the top-down state-passing pattern in its cleanest form. Every node must know the maximum value on the path from root to itself — this information can ONLY come from above (from parent to child). The bottom-up return aggregates the count. The combination of "pass state down + aggregate up" is the pattern that directly leads into DP on trees.

What to prove: Top-down pattern: pass `maxSoFar` as a parameter to each recursive call (NOT as a global — it changes per path). At each node, compute `isGood = (node->val >= maxSoFar) ? 1 : 0`. Update `newMax = max(maxSoFar, node->val)` BEFORE recursing (children need the max INCLUDING the current node).

Target time: 10 minutes.

Edge cases: All nodes with the same value (all are good nodes — each node equals the max so far). Strictly decreasing values from root to leaves (only the root is a good node). Single node (always good — maxSoFar starts at INT_MIN).

---

## PATTERN CONNECTIONS

**Connects to Linked Lists (Pattern 07):**
A binary tree node is a linked list node with TWO `next` pointers (`left` and `right`) instead of one. Every pointer-discipline lesson from Pattern 07 applies directly: null checks before dereferencing, traversing a "chain" of pointers, the dummy node concept (preorder serialization's null markers serve the same structural role). The `if (!root) return` base case in tree recursion is the exact analog of `if (!head) return` in linked list recursion.

**Connects to Stack (Pattern 04):**
Iterative DFS uses an explicit stack (Templates 4, 5, 6). The stack mimics what happens in the call stack during recursive DFS. This connection — "recursion IS a stack" — is the conceptual bridge between the two patterns. Understanding it lets you convert any recursive tree algorithm to iterative, which interviewers sometimes request.

**Connects to Backtracking (Pattern 15):**
Path Sum II (collect all root-to-leaf paths) IS a tree + backtracking problem. You push to the current path, recurse, then pop (backtrack). The tree structure provides the natural backtracking trigger: when the function returns to the parent, you undo the push. Pattern 15's backtracking is the generalization of this to non-tree structures.

**Connects to Binary Search (Pattern 05):**
BST operations (search, insert, validate) are binary search on a tree structure. Each step eliminates one half of the remaining candidates. The `validate` function's `(minVal, maxVal)` bounds are exactly analogous to the `[lo, hi]` interval in binary search — both maintain the invariant "the answer must lie in this range."

**Connects to DP on Trees (Pattern 22):**
Every "return value from subtree" template in this pattern IS tree DP. `height(node) = 1 + max(h(left), h(right))` is a recurrence relation on a tree structure. Pattern 22 formalizes this into harder problems: house robber on tree (can't take adjacent nodes), rerooting technique (compute answer for every node as root in O(n)), path DP. The foundation is exactly Template 7 and Template 12 from this pattern.

**Connects to Graphs BFS/DFS (Pattern 12):**
A tree is a special graph (connected, acyclic, exactly one path between any two nodes). All tree DFS algorithms work on general graphs with one addition: a `visited` set or array (graphs can have cycles; trees cannot). If you master tree DFS here, graph DFS in Pattern 12 is a small conceptual step — add cycle prevention and handle arbitrary structure.

**Connects to Divide and Conquer:**
Construct Binary Tree from Preorder + Inorder (LC 105) is textbook divide and conquer on trees. The root divides the problem into two independent subproblems: build the left subtree, build the right subtree. These are solved recursively and combined at the root. The same thinking appears in merge sort (split + sort left + sort right + merge) and quicksort (partition + sort left + sort right).

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Find the maximum depth of a binary tree. Now extend — how would you find the diameter?"

**Company type:** Amazon phone screen, Microsoft OA, entry-level Google — this is the warm-up question before harder tree problems
**Expected time:** 2 minutes for depth, 12 minutes for diameter
**What they are testing:** Whether recursion is natural and fast for you, the "return value ≠ answer" insight, ability to identify the right traversal order

**Red flags that get candidates rejected:**
- Taking more than 3 minutes on max depth — at interview level this must be instant, and slowness signals tree problems will be a struggle
- For diameter: returning the diameter from the recursive function instead of height — gives wrong answers because a parent node computing its own diameter needs its children's HEIGHTS, not their diameters
- Using a non-recursive approach (BFS with a level counter) for max depth — works but slower to write, and signals you don't have the tree recursion reflex
- For diameter: off-by-one between "diameter in nodes" vs "diameter in edges" — LC 543 counts edges, the answer is `leftH + rightH`, not `leftH + rightH + 1`
- Not being able to explain the global variable — "I just know you need it" is a red flag; you should say "I need the maximum of leftH + rightH across all nodes, but my function must RETURN height to its parent"

**What they want you to say:**
"For max depth: base case is null returns 0. For any node, depth is 1 plus the maximum of left and right children's depths. It's postorder — I need children's depths before computing mine. For diameter: I reuse the height computation. At each node, the diameter through it equals leftHeight + rightHeight (edges between the leftmost and rightmost leaves through this node). I track the global maximum of this across all nodes. My function RETURNS height to the parent — not the diameter — because the parent needs height to compute its own diameter. The global variable captures the answer separately."

**The follow-up:** "Can you do this without a global variable?" → Return a `pair<int, int>` from the recursive function: `{height, maxDiameterInSubtree}`. At each node: left_result = recurse(left), right_result = recurse(right). Height = 1 + max(left.first, right.first). Max diameter in this subtree = max(left.first + right.first, left.second, right.second). Return this pair.

---

### Q2: "Find the lowest common ancestor of two nodes p and q in a binary tree. Now if the tree is a BST, can you do better?"

**Company type:** Google, Meta — appears in nearly every loop; Amazon SDE-2 rounds
**Expected time:** 20 minutes for general binary tree, 8 minutes for BST variant
**What they are testing:** Postorder tree reasoning, BST property exploitation, O(1) space optimization, ability to EXPLAIN the algorithm not just write it

**Red flags that get candidates rejected:**
- Not being able to articulate why the base case returns `root` (not null) when `root == p` — this is the most asked follow-up, and if you say "it just works" you'll be probed further
- For the general tree: using a global variable to store the LCA result instead of the clean return-based propagation — signals surface-level pattern matching
- For BST: writing a recursive solution when the iterative is O(1) space and strictly better — Google always asks for optimal space
- Confusing the two cases: "both subtrees returned something" vs "only one did" — saying "I return root when both are not null" without explaining WHY (p is on one side, q is on the other, current node is the first to see both)
- Edge case blindness: not handling "what if p is an ancestor of q?" — the algorithm handles this correctly, but you must be able to EXPLAIN it, not just claim it works

**What they want you to say:**
"For general binary tree: postorder DFS. Base case: null returns null; p or q returns itself (non-null signals the caller that this target was found here). I search both subtrees. If both return non-null: p is in one subtree and q is in the other — the current node is the first node that sees both — it's the LCA. If only one side returned non-null: both are on that side, so the LCA must be there — propagate that result up. For BST: I exploit the BST property to navigate directly in a while loop. If both p and q are less than current, go left. If both greater, go right. Otherwise (they split here), current IS the LCA. This is O(h) time, O(1) space — better than the general approach's O(h) space from recursion."

**The follow-up:** "What if node values are not unique?" → General binary tree: algorithm compares POINTER identity (`root == p`), not values — duplicates don't affect correctness. BST: the algorithm uses value comparison for navigation — with duplicates, the BST property breaks down and the navigation logic needs modification (potentially handle equal values explicitly).

---

### Q3: "Find the maximum sum of any path in a binary tree, where the path can start and end at any node."

**Company type:** Amazon (most frequent), Google, Microsoft senior rounds — this is the tree problem interviewers use to test FAANG-level depth
**Expected time:** 25 minutes
**What they are testing:** The "split vs extend" insight, ability to distinguish two separate quantities from one function, INT_MIN initialization, negative value handling

**Red flags that get candidates rejected:**
- Initializing `globalMax = 0` — fails for all-negative trees; interviewers may test this explicitly with `[-3]`
- Not using `max(leftGain, 0)` — negative subtrees incorrectly reduce the sum, preventing paths from starting or ending at the right node
- Returning `node->val + leftGain + rightGain` to the parent — the Y-shape mistake; must return only ONE arm to the parent
- Not being able to explain WHY the return value uses only one arm — just saying "that's how the algorithm works" is a MAJOR red flag; you must say "a path can't branch; when the parent extends this path, it can only go through one direction from me"
- Not considering that the optimal path might be a single node (when all values are negative), or that the optimal path might not pass through the root

**What they want you to say:**
"The key insight: a path through a node can use BOTH its left and right arms — that's the through-path, and I use this for the global maximum. But when I return a value to my parent, the parent can only extend the path in ONE direction from me — it can't branch. So I separate two computations: (1) through-path = node->val + max(L, 0) + max(R, 0) → update globalMax. (2) Return to parent = node->val + max(max(L, 0), max(R, 0)) → single arm only. I use max(gain, 0) because if a subtree contributes negatively, it's better to start or end the path at the current node rather than extending into the negative arm. I initialize globalMax to INT_MIN — not 0 — because the problem requires at least one node, and if all values are negative, the answer is the least-negative single node (which would be incorrectly returned as 0 with zero initialization)."

**The follow-up:** "How would you print the actual path (not just the sum)?" → Track parent pointers during the traversal. At each node, if the through-path improves globalMax, record which arms were used and the path's endpoint nodes. Then reconstruct from root to each endpoint using parent pointers. This is significantly more complex than just computing the sum — acknowledge the added complexity.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

**Q1:** A tree function returns height to its parent but tracks diameter globally. Why must it return height (not diameter) to its parent?
> **A:** A parent node computes the diameter THROUGH ITSELF as `leftHeight + rightHeight`. To do this, the parent needs its children's HEIGHTS — how far each subtree extends outward. If the function returned the diameter of the subtree instead, the parent would have no information about how "tall" (extendable) the subtree is. Height is the "reach" — how far this subtree can contribute to the parent's through-path. The global diameter tracks the widest path seen anywhere; height is what flows upward.

**Q2:** What is the single most important property of inorder traversal of a BST, and name two LeetCode problems it directly solves?
> **A:** Inorder traversal of a BST visits nodes in strictly ascending (sorted) order. It directly solves: (1) LC 230 — Kth Smallest Element in a BST: the Kth node visited during inorder traversal IS the Kth smallest. (2) LC 98 — Validate BST (alternative approach): during inorder traversal, each value must be strictly greater than the previous value; if any value is ≤ previous, the BST is invalid. Also directly solves "convert BST to sorted array" and "BST iterator."

**Q3:** In LCA of general binary tree (LC 236), what does it mean when the function returns non-null from both the left and right subtree calls?
> **A:** A non-null return from a subtree means "p, q, or their LCA was found somewhere in that subtree." If both left AND right return non-null, it means p was found in one subtree and q in the other (or vice versa). The CURRENT node is therefore the first node in the tree that "sees" both p and q on opposite sides of it — which is exactly the definition of the Lowest Common Ancestor. So return the current node (root) as the LCA.

**Q4:** You're validating a BST. The tree is `[10, 5, 15, null, null, 6, 20]`. Without writing code, trace which call fails in the bounds-propagation approach and why.
> **A:** `validate(15, 10, +∞)`: 15 > 10 ✓ → valid so far. Then `validate(6, 10, 15)`: check if `6 > 10` → 6 ≤ 10 → FAIL. Returns false immediately. The bounds propagation caught it because when we went RIGHT from 10, minVal was set to 10. Node 6 must be strictly greater than 10 (we went right from 10), but 6 is not. Direct parent-child check would have passed (6 < 15 ✓), missing this violation.

**Q5:** In max path sum, you compute `int L = max(maxGain(node->left), 0)`. Why take `max(..., 0)` instead of just using `maxGain(node->left)` directly?
> **A:** If the left subtree's best path sum is NEGATIVE, including it in our through-path would DECREASE the total sum. Taking `max(leftGain, 0)` means: if the left subtree contributes positively, use it; if it contributes negatively, treat it as "don't include the left subtree" (effectively starting or ending the path at the current node). This allows optimal paths to start or end in the middle of the tree rather than always at a leaf. Without this, paths into negative subtrees would incorrectly reduce the global maximum.

**Q6:** In the serialize/deserialize problem, why is preorder (not inorder) chosen for serialization?
> **A:** Preorder visits the ROOT before its children. During deserialization, you reconstruct the tree top-down: create the root first, then build its left and right subtrees. Since preorder gives you the root as the first token, you can create the root node immediately and then recursively build its children. With inorder, the root appears in the middle of the sequence — you can't determine which token is the root without additional information. Postorder gives the root last — you'd need to process tokens in reverse, which complicates the reconstruction logic.

**Q7:** What are the three traversal orders, and for each: when is it the right choice? Give one archetypal problem.
> **A:** (1) Preorder (root → left → right): use when you need the node BEFORE its children — either passing information downward or building the node before its subtree. Archetype: Serialize binary tree (need root before children to reconstruct). Also: Count Good Nodes (pass maxSoFar from root downward). (2) Inorder (left → root → right): use when you need the node IN THE CONTEXT OF ITS LEFT AND RIGHT subtrees — primarily for BSTs where sorted order is needed. Archetype: Kth Smallest in BST (inorder = sorted order). Also: BST validation, BST range queries. (3) Postorder (left → right → root): use when you need children's ANSWERS before computing the current node's answer. Archetype: Diameter of Binary Tree (need children's heights first). Also: LCA, max path sum, balanced tree check — any "return value from subtree" pattern.

---

## C++ BINARY TREE CHEAT SHEET

```cpp
// ============================================================
// TREENODE DEFINITION
// ============================================================
struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

// ============================================================
// BUILD TREE FROM LEVEL-ORDER ARRAY (for testing)
// Use INT_MIN or -1 to represent null nodes in the input
// ============================================================
TreeNode* buildTree(vector<int> vals) {
    if (vals.empty() || vals[0] == INT_MIN) return nullptr;
    TreeNode* root = new TreeNode(vals[0]);
    queue<TreeNode*> q;
    q.push(root);
    int i = 1;
    while (!q.empty() && i < (int)vals.size()) {
        TreeNode* node = q.front(); q.pop();
        if (i < (int)vals.size() && vals[i] != INT_MIN) {
            node->left = new TreeNode(vals[i]); q.push(node->left);
        }
        i++;
        if (i < (int)vals.size() && vals[i] != INT_MIN) {
            node->right = new TreeNode(vals[i]); q.push(node->right);
        }
        i++;
    }
    return root;
}

// ============================================================
// THE POSTORDER "RETURN VALUE" SKELETON — MEMORIZE THIS
// ============================================================
int postorderDP(TreeNode* node) {
    if (!node) return BASE;               // null base case
    int L = postorderDP(node->left);      // compute left subtree answer
    int R = postorderDP(node->right);     // compute right subtree answer
    return combine(L, R, node->val);      // combine to get this node's answer
}

// ============================================================
// BST VALIDATE — MEMORIZE THIS (long long bounds!)
// ============================================================
bool validate(TreeNode* n, long long lo, long long hi) {
    if (!n) return true;
    if (n->val <= lo || n->val >= hi) return false;
    return validate(n->left, lo, n->val) && validate(n->right, n->val, hi);
}
bool isValidBST(TreeNode* root) { return validate(root, LLONG_MIN, LLONG_MAX); }

// ============================================================
// LCA BINARY TREE — MEMORIZE THIS (4 lines)
// ============================================================
TreeNode* lca(TreeNode* r, TreeNode* p, TreeNode* q) {
    if (!r || r == p || r == q) return r;
    TreeNode* L = lca(r->left, p, q);
    TreeNode* R = lca(r->right, p, q);
    if (L && R) return r;
    return L ? L : R;
}

// ============================================================
// LCA BST — MEMORIZE THIS (iterative, O(1) space)
// ============================================================
TreeNode* lcaBST(TreeNode* r, TreeNode* p, TreeNode* q) {
    while (r) {
        if (p->val < r->val && q->val < r->val) r = r->left;
        else if (p->val > r->val && q->val > r->val) r = r->right;
        else return r;
    }
    return nullptr;
}

// ============================================================
// MAX PATH SUM — MEMORIZE THIS (INT_MIN, max(gain,0), one arm to parent)
// ============================================================
int gMax;
int gain(TreeNode* n) {
    if (!n) return 0;
    int L = max(gain(n->left), 0);
    int R = max(gain(n->right), 0);
    gMax = max(gMax, n->val + L + R);
    return n->val + max(L, R);
}
int maxPathSum(TreeNode* root) { gMax = INT_MIN; gain(root); return gMax; }

// ============================================================
// ITERATIVE INORDER — MEMORIZE THIS (3 phases)
// ============================================================
vector<int> inorderIter(TreeNode* root) {
    vector<int> res; stack<TreeNode*> st; TreeNode* c = root;
    while (c || !st.empty()) {
        while (c) { st.push(c); c = c->left; }      // Phase 1: go left
        c = st.top(); st.pop(); res.push_back(c->val); // Phase 2: process
        c = c->right;                                  // Phase 3: go right
    }
    return res;
}

// ============================================================
// SERIALIZE / DESERIALIZE — MEMORIZE THE STRUCTURE
// ============================================================
string ser(TreeNode* r) {
    if (!r) return "X";
    return to_string(r->val) + "," + ser(r->left) + "," + ser(r->right);
}
TreeNode* des(queue<string>& q) {
    string v = q.front(); q.pop();
    if (v == "X") return nullptr;
    TreeNode* n = new TreeNode(stoi(v));
    n->left = des(q); n->right = des(q);
    return n;
}

// ============================================================
// BST INSERT (recursive)
// ============================================================
TreeNode* insert(TreeNode* root, int val) {
    if (!root) return new TreeNode(val);
    if (val < root->val) root->left  = insert(root->left, val);
    else                 root->right = insert(root->right, val);
    return root;
}

// ============================================================
// DIAMETER TEMPLATE (returns height, tracks diameter globally)
// ============================================================
int diam = 0;
int ht(TreeNode* n) {
    if (!n) return 0;
    int L = ht(n->left), R = ht(n->right);
    diam = max(diam, L + R);
    return 1 + max(L, R);
}
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. Implement all six traversal variants from memory — recursive preorder, inorder, postorder; then iterative preorder, inorder, postorder. Write them on paper first (no IDE), then verify in the IDE. The iterative inorder (Template 5) must be mechanical. If you can't write the iterative inorder without reference in 3 minutes, practice it more.

2. For BST validate (LC 98): solve it TWICE deliberately. First attempt: write the direct parent-child check. Then submit — watch it fail on a case like `[5, 4, 6, null, null, 3, 7]`. Then rewrite with bounds propagation. This deliberate failure permanently cements why bounds propagation is necessary.

3. For max path sum (LC 124): after solving it, trace through `[-3]` (single negative node — answer should be -3, not 0), `[-1, -2, -3]` (all negative — answer should be -1), and `[1, 2, 3]` (positive — answer should be 6). All three must produce correct results. If your initialization is wrong, one of these will fail.

4. For LCA (LC 236): implement it, then cover your code and write it again from memory. The four lines `if (!root || root==p || root==q) return root; L=lca(left); R=lca(right); if (L&&R) return root; return L?L:R` must be completely mechanical. Be able to explain each line out loud.

5. In REVISION-TRACKER.md, add these as weekly revisit problems:
   - LC 124 (Max Path Sum) — the split insight requires repeated exposure to become automatic
   - LC 105 (Construct from Preorder+Inorder) — the index arithmetic for left/right subtree ranges is error-prone
   - LC 98 (Validate BST) — bounds propagation with LLONG_MIN/LLONG_MAX must remain instant

6. Connecting forward to Pattern 09 (Trees BFS): BFS uses a queue instead of a stack (or recursion). The level-order traversal template with a queue is the tree equivalent of BFS on graphs. Before starting Pattern 09, make sure your iterative inorder (Template 5) is solid — the "explicit data structure to manage traversal order" thinking transfers directly.

7. Connecting forward to Pattern 22 (DP on Trees): the Templates 7 and 12 in this pattern ARE tree DP. When you reach Pattern 22, you'll recognize that `dp[node] = combine(dp[node->left], dp[node->right])` is exactly what you've been doing here. The only extension is harder state definitions and the rerooting technique.

---

## SIGN-OFF CRITERIA

**Tier 1 (Non-negotiable):**
- [ ] All 6 traversal variants (3 recursive + 3 iterative) from memory — no bugs, correct outputs
- [ ] BST validate with bounds propagation — no direct parent-child check — `LLONG_MIN/LLONG_MAX` — handles INT_MIN/INT_MAX values
- [ ] LCA of general binary tree — 4-line solution from memory — can explain WHY each line is correct
- [ ] Max path sum — `INT_MIN` initialization, `max(gain, 0)`, single arm returned to parent — correct for all-negative trees

**Tier 2:**
- [ ] LC 543 (Diameter) — return height, track max(L+R) globally
- [ ] LC 235 (LCA BST) — iterative, O(1) space
- [ ] LC 236 (LCA binary tree) — can explain the propagation logic cold

**Tier 3:**
- [ ] LC 105 (Construct from Preorder+Inorder) — hash map for O(1) root lookup, index arithmetic correct
- [ ] LC 297 (Serialize/Deserialize) — preorder + null markers, queue-based deserialization

**Tier 4 (attempt before sign-off):**
- [ ] LC 230 (Kth Smallest in BST) — inorder with early termination
- [ ] LC 1448 (Count Good Nodes) — top-down maxSoFar passing understood, not just working code

**Sign-off test:** I give you one unseen tree problem. You must:
1. Identify: top-down or bottom-up? Which traversal order? Why?
2. State: what does the recursive function return to its parent?
3. Write complete C++ solution in one pass
4. State time and space complexity with the O(h) explanation

Type `SIGN OFF P08` when ready.

---

*Pattern 08 Complete — Trees: DFS Complete (Traversal + BST + LCA + Path Problems)*
*Next: Pattern 09 — Trees: BFS and Level Order*
