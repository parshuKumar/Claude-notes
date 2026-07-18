# PATTERN 22 — DP ON TREES
## Postorder DP, Rerooting, Path DP, and DSU-on-Tree — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The one question that unlocks every tree DP:**

> "What does this subtree return to its parent?"

That's it. That's the whole skill. Every tree DP problem — from Diameter of Binary Tree to Sum of Distances in Tree — is an exercise in deciding what a subtree hands upward, and what the parent does with it.

Most candidates fail tree DP not because recursion is hard, but because they conflate two different things:

1. **The value returned** (what the parent needs to continue its own computation)
2. **The answer being tracked** (what the problem actually asks for)

These are often NOT the same quantity. In Binary Tree Maximum Path Sum (LC 124), a node returns "the best single downward path through me" (needed by the parent, because a path can only bend once), but it *updates* a global answer using "the best path through me using BOTH children" (which cannot be returned upward, because a path that bends can't extend further).

**The four skills this pattern builds:**

1. **Postorder discipline** — children must finish before the parent can compute anything. This is non-negotiable; you cannot do tree DP with a preorder traversal.
2. **State compression** — deciding whether a node needs to return one value, a pair (take/skip), or a small fixed set of states (0/1/2 for cameras).
3. **The path-vs-return distinction** — a "path" that bends at a node cannot be extended by the parent; only a straight, single-direction contribution can be returned upward.
4. **Rerooting** — computing an answer *for every node as root* in O(n) total, instead of O(n) per node (O(n²) overall). This is the single most powerful upgrade from "I can do basic tree DP" to "I can do 1900+ tree DP."

**Where candidates get stuck (ranked by frequency):**

- Returning the "bent" value instead of the "straight" value from a path-sum function (breaks the parent's math).
- Forgetting to clamp negative contributions to 0 (a negative subtree should simply not be included).
- Solving "answer rooted at every node" by re-running full DFS from each node — O(n²), TLEs on any n > ~2000.
- Recursion depth: an unbalanced/skewed tree with n = 10⁵ nodes blows the call stack (~1MB default stack, ~10⁴–10⁵ frames depending on frame size). This is a real, frequently-tested failure mode.
- Confusing binary-tree idioms (`TreeNode* left, *right`) with general-tree idioms (`vector<int> adj[]`, arbitrary branching, need explicit parent tracking to avoid revisiting).

**The meta-skill:** once you can answer "what does this subtree return, and does the parent need MORE than that one value," every problem in this pattern becomes a 10–20 line function.

---

## SECTION 2 — THE TREE DP TAXONOMY

### Type 1: Postorder DP (single accumulated value)
State: `dfs(node)` returns one scalar summarizing the subtree.
Direction: children first, then node (postorder).
Problems: Diameter of Binary Tree, subtree sums, subtree sizes.

### Type 2: Take/Skip on Nodes (paired state)
State: `dfs(node)` returns `{included, excluded}` — best answer if this node IS/ISN'T used.
Direction: postorder; parent combines both children's pairs.
Problems: House Robber III, Maximum Independent Set on Trees.

### Type 3: Path DP (best single downward path)
State: `dfs(node)` returns the best path that starts at `node` and goes downward through ONE child only (a "straight line," never bending).
Side effect: a global/nonlocal variable is updated with the best path that bends AT `node` (uses both children).
Problems: Binary Tree Maximum Path Sum, Diameter (a special case: all weights = 1, "sum" = "count of edges"), Longest Path With Different Adjacent Characters.

### Type 4: Multi-State DP (small fixed automaton per node)
State: `dfs(node)` returns one of a small number of discrete states (not a number, a *category*).
Problems: Binary Tree Cameras (3 states: not covered / covered-no-camera / has-camera).

### Type 5: Rerooting (answer for every node as root, O(n) total)
Two DFS passes:
  - **Pass 1 (postorder):** compute subtree-only answers rooted arbitrarily at node 0.
  - **Pass 2 (preorder):** propagate a correction from parent to child in O(1) per edge, converting the subtree-only answer into the true whole-tree answer for every node.
Problems: Sum of Distances in Tree, Sum of Subtree Sizes variants, "for each node as root, compute X."

### Type 6: Subtree-Sum / Split DP
State: `dfs(node)` returns the sum (or count) of the subtree; the ANSWER is a function of that sum compared against a global total (computed via edge removal, splitting the tree into two components).
Problems: Maximum Product of Splitted Binary Tree, Delete Edge to Maximize Product-like problems.

### Type 7: DSU-on-Tree / Value-Sorted Union
State: process nodes/edges in sorted order of VALUE (not tree order); union-find merges components; answer accumulates at merge time.
Problems: Number of Good Paths, Minimize Malware Spread (tree variant), Tree of Coprimes (ancestor lookup, a cousin technique).

### Type 8: Interval DP Disguised as Tree Building
Some "build an optimal tree from a sequence" problems (e.g., Minimum Cost Tree From Leaf Values) are tree DP in disguise but are solved with **interval DP** over the leaf array, not recursion over an existing tree. Included here because the *result* is a tree-shaped cost structure, but the *technique* borrows from Pattern 23 (Interval DP).

---

## SECTION 3 — TEMPLATE 1: DIAMETER OF BINARY TREE (FOUNDATION)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 543 — Diameter of Binary Tree
// The diameter is the length (in EDGES) of the longest path between
// any two nodes. The path may or may not pass through the root.
//
// KEY INSIGHT: for any node, the longest path THROUGH that node
// (bending at it) is height(left subtree) + height(right subtree).
// The overall diameter is the MAX of this quantity over all nodes.
//
// What does this subtree return to its parent?
//   → its HEIGHT (longest downward chain of edges), because the
//     parent needs to know how far down it can extend a path.
// What does this subtree update globally?
//   → the best "bent" path using BOTH children, because that bent
//     path cannot be extended further upward (it already turns once).
// ─────────────────────────────────────────────────────────────────

struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

class Solution {
public:
    int diameter = 0;

    // Returns HEIGHT of subtree rooted at node (0 for null, so a leaf has height 1)
    int height(TreeNode* node) {
        if (!node) return 0;
        int lh = height(node->left);
        int rh = height(node->right);
        diameter = max(diameter, lh + rh);   // best path BENDING at node
        return 1 + max(lh, rh);              // best STRAIGHT path extending upward
    }

    int diameterOfBinaryTree(TreeNode* root) {
        diameter = 0;
        height(root);
        return diameter;
    }
};
```

**FULL TRACE for tree `[1,2,3,4,5]`:**
```
        1
       / \
      2   3
     / \
    4   5

height(4): leaf. lh=0, rh=0. diameter=max(0, 0+0)=0. return 1.
height(5): leaf. lh=0, rh=0. diameter=max(0, 0+0)=0. return 1.
height(2): lh=height(4)=1, rh=height(5)=1.
           diameter=max(0, 1+1)=2. return 1+max(1,1)=2.
height(3): leaf. lh=0, rh=0. diameter=max(2, 0)=2. return 1.
height(1): lh=height(2)=2, rh=height(3)=1.
           diameter=max(2, 2+1)=3. return 1+max(2,1)=3.

Answer: diameter = 3
Path: 4 → 2 → 1 → 3  (3 edges) ✓
```

**Verified by compiling and running against the LC 543 example: output matches 3.**

**Why height, not diameter, is returned:** the parent (node 1) needs to know "how far can a straight path extend down through my child 2?" — that's `height(2) = 2`. It does NOT need to know "what's the best bent path inside subtree 2?" (that's already been folded into the global `diameter` and can never be extended further, since it already uses both of 2's children).

---

## SECTION 4 — TEMPLATE 2: HOUSE ROBBER III (TAKE/SKIP PAIR)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 337 — House Robber III
// Binary tree of house values. Can't rob two directly-connected
// houses (parent + child). Maximize total value robbed.
//
// What does this subtree return to its parent?
//   → a PAIR {rob, skip}:
//     rob  = max value obtainable from this subtree IF this node IS robbed
//     skip = max value obtainable from this subtree IF this node is NOT robbed
//
// If node IS robbed: children must NOT be robbed (skip only).
// If node is NOT robbed: children can be robbed or not — take the better of each.
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    // returns {robThis, skipThis}
    pair<int,int> dfs(TreeNode* node) {
        if (!node) return {0, 0};
        auto [lr, ls] = dfs(node->left);
        auto [rr, rs] = dfs(node->right);

        int robThis  = node->val + ls + rs;         // must skip both children
        int skipThis = max(lr, ls) + max(rr, rs);    // children free to choose best

        return {robThis, skipThis};
    }

    int rob(TreeNode* root) {
        auto [r, s] = dfs(root);
        return max(r, s);
    }
};
```

**FULL TRACE for tree `[3,2,3,null,3,null,1]`:**
```
        3(a)
       /   \
      2(b)  3(c)
       \      \
        3(d)   1(e)

dfs(d): leaf. return {3, 0}.
dfs(e): leaf. return {1, 0}.
dfs(b): left=dfs(null)={0,0}, right=dfs(d)={3,0}.
        robThis  = 2 + 0(ls) + 0(rs) = 2
        skipThis = max(0,0) + max(3,0) = 3
        return {2, 3}
dfs(c): left=dfs(null)={0,0}, right=dfs(e)={1,0}.
        robThis  = 3 + 0 + 0 = 3
        skipThis = max(0,0) + max(1,0) = 1
        return {3, 1}
dfs(a): left=dfs(b)={2,3}, right=dfs(c)={3,1}.
        robThis  = 3 + ls(3) + rs(1) = 7
        skipThis = max(2,3) + max(3,1) = 3 + 3 = 6
        return {7, 6}

Answer: max(7, 6) = 7
Rob: a(3) + d(3) + e(1) = 7 ✓
```

**Verified by compiling and running against the LC 337 example: output matches 7.**

**Common trap:** returning a single `int best = max(robThis, skipThis)` per node. This is WRONG because a parent that decides to rob itself needs `skip` from its children specifically — it cannot use a child's own already-collapsed "best," because that collapsed best might have included robbing the child.

---

## SECTION 5 — TEMPLATE 3: BINARY TREE MAXIMUM PATH SUM (CORE PATTERN)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 124 — Binary Tree Maximum Path Sum
// A "path" is any sequence of nodes connected by edges, where each
// node appears at most once, and it does NOT need to pass through
// the root. Find the maximum path sum (node values can be negative).
//
// What does this subtree return to its parent?
//   → the best STRAIGHT downward path sum starting at this node and
//     going through AT MOST ONE child (because a path can only bend
//     ONCE across the whole tree — if the parent extends through this
//     node, it can only continue in one direction).
// What does this subtree update globally?
//   → the best path that BENDS at this node, using BOTH children.
//     (val + leftGain + rightGain). This can never be returned
//     upward because it already uses both children — extending it
//     further would require a THIRD branch, which paths don't have.
//
// Negative-child clamp: if a child's best gain is negative, we must
// NOT include it (max(childGain, 0)) — a negative contribution only
// hurts the sum, so we simply don't walk into that child at all.
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    int best = INT_MIN;

    int maxGain(TreeNode* node) {
        if (!node) return 0;

        int leftGain  = max(maxGain(node->left),  0);   // clamp negatives to 0
        int rightGain = max(maxGain(node->right), 0);

        // Best path BENDING at this node (uses both children) — update global only
        best = max(best, node->val + leftGain + rightGain);

        // Best STRAIGHT path through this node — returned upward
        return node->val + max(leftGain, rightGain);
    }

    int maxPathSum(TreeNode* root) {
        best = INT_MIN;
        maxGain(root);
        return best;
    }
};
```

**FULL TRACE for tree `[-10,9,20,null,null,15,7]`:**
```
        -10
        /  \
       9    20
           /  \
          15    7

maxGain(9): leaf. leftGain=0, rightGain=0.
            best=max(-inf, 9+0+0)=9. return 9+max(0,0)=9.
maxGain(15): leaf. leftGain=0, rightGain=0.
             best=max(9, 15+0+0)=15. return 15.
maxGain(7): leaf. leftGain=0, rightGain=0.
            best=max(15, 7)=15. return 7.
maxGain(20): leftGain=max(maxGain(15)=15,0)=15
             rightGain=max(maxGain(7)=7,0)=7
             best=max(15, 20+15+7=42)=42.
             return 20+max(15,7)=35.
maxGain(-10): leftGain=max(maxGain(9)=9,0)=9
              rightGain=max(maxGain(20)=35,0)=35
              best=max(42, -10+9+35=34)=42  (34 < 42, no change)
              return -10+max(9,35)=25.

Answer: best = 42
Path: 15 → 20 → 7 = 42 ✓ (root -10 is NOT part of the best path — this is
                            exactly why "path sum" does not require the root.)
```

**Verified by compiling and running against the LC 124 example: output matches 42.**

**Why we return max of ONE side but update with BOTH sides — the geometric proof:** picture the tree as a set of edges. A "path" is a simple chain — no branching, no revisiting a node. If node X's path continues upward into its parent P, then from X's perspective, the path can go into AT MOST ONE of X's children (it already has one direction "used up" going toward P). But when we're standing AT X considering "what's the best path with X as the TURNING POINT," X is allowed to use both children, because X itself is the bend and doesn't need to extend further in either direction.

---

## SECTION 6 — TEMPLATE 4: BINARY TREE CAMERAS (3-STATE GREEDY DP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 968 — Binary Tree Cameras
// A camera at a node covers itself, its parent, and its children.
// Find the minimum number of cameras to cover every node.
//
// This is a GREEDY-DP hybrid: greedy because we always prefer to
// place a camera as low (as close to leaves) as possible; DP because
// we need each subtree to report ITS state so the parent can decide.
//
// THREE STATES per node (encoded as int, but conceptually an enum):
//   0 = NOT COVERED       (this node has no camera, and none of its
//                           children have a camera covering it — the
//                           PARENT is now obligated to place one)
//   1 = COVERED, no camera (this node is watched by a child's camera,
//                           but has no camera itself)
//   2 = HAS CAMERA          (this node itself holds a camera)
//
// Null nodes return 1 ("covered") — NOT 0 — because a leaf with a
// null child should NOT be forced to place a camera just because
// its (nonexistent) child is "uncovered." This is the trick that
// makes leaves report {0 (both children covered-null), ...} → their
// OWN state ends up 0 only if BOTH children are literally state-0,
// which can't happen for a leaf, so leaves correctly come out as 0
// via the "return 0" fallback, forcing their PARENT to place cameras
// at the right places, not every leaf itself.
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    int cameras = 0;

    int dfs(TreeNode* node) {
        if (!node) return 1;   // treat "no node" as already covered

        int left  = dfs(node->left);
        int right = dfs(node->right);

        if (left == 0 || right == 0) {
            cameras++;
            return 2;   // a child is uncovered — must place a camera HERE
        }
        if (left == 2 || right == 2) {
            return 1;   // a child already has a camera — this node is covered
        }
        return 0;       // both children covered but neither has a camera —
                         // this node itself is NOT covered; parent must act
    }

    int minCameraCover(TreeNode* root) {
        cameras = 0;
        if (dfs(root) == 0) cameras++;   // root itself ended up uncovered
        return cameras;
    }
};
```

**FULL TRACE for tree `[0,0,null,0,0]`:**
```
        a(0)
       /
      b(0)
     /  \
    c(0)  d(0)

dfs(c): leaf. left=dfs(null)=1, right=dfs(null)=1.
        neither 0 nor 2 present → return 0 (c is not covered).
dfs(d): leaf. same as c → return 0.
dfs(b): left=dfs(c)=0, right=dfs(d)=0.
        left==0 → cameras++ (cameras=1). return 2 (b now has a camera).
dfs(a): left=dfs(b)=2, right=dfs(null)=1.
        left==2 → return 1 (a is covered, by b's camera).

dfs(root a) = 1, not 0 → no extra camera added.
Answer: cameras = 1 ✓
(One camera at node b covers a, b, c, and d — matches LC 968 example 1.)
```

**Verified by compiling and running against the LC 968 example: output matches 1.**

**Why greedy-from-leaves is optimal here:** if a leaf were forced to hold a camera, that camera would only cover the leaf and its parent — a strictly worse (or equal) choice than placing the camera at the PARENT, which covers the leaf, the parent, AND the parent's other children/parent. Pushing camera placement as low as possible while still satisfying coverage is provably optimal — this is the greedy justification underneath the DP transition.

---

## SECTION 7 — TEMPLATE 5: LONGEST PATH WITH DIFFERENT ADJACENT CHARACTERS (GENERAL TREE, TOP-2 CHILDREN)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 2246 — Longest Path With Different Adjacent Characters
// General tree given via parent[] array (node 0 is root). s[i] is
// the character at node i. Find the longest path (in NODES) such
// that no two adjacent nodes on the path share the same character.
//
// This is the GENERAL-TREE analogue of Binary Tree Diameter/Path
// Sum — but because a node can have MANY children (not just 2), we
// need the TOP-TWO longest valid child-paths, not just "left vs right."
//
// What does this subtree return to its parent?
//   → the length (in nodes) of the best downward path starting at
//     this node, IF the parent's character differs from this node's
//     (the parent decides whether to use it — we just report length
//     assuming compatibility; caller filters by character match).
// What does this subtree update globally?
//   → best1 + best2 + 1 — the best path BENDING at this node using
//     its two best (different-character) children branches.
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    int longestPath(vector<int>& parent, string s) {
        int n = parent.size();
        vector<vector<int>> children(n);
        for (int i = 1; i < n; i++) children[parent[i]].push_back(i);

        int ans = 1;   // a single node is always a path of length 1

        function<int(int)> dfs = [&](int u) -> int {
            int best1 = 0, best2 = 0;   // top two child-path lengths (in nodes)
            for (int v : children[u]) {
                int len = dfs(v);
                if (s[v] == s[u]) continue;   // can't cross a same-character edge
                if (len > best1) { best2 = best1; best1 = len; }
                else if (len > best2) { best2 = len; }
            }
            ans = max(ans, best1 + best2 + 1);  // bend at u using its 2 best branches
            return best1 + 1;                    // straight path returned to parent
        };

        dfs(0);
        return ans;
    }
};
```

**FULL TRACE for `parent = [-1,0,0,1,1,2]`, `s = "abacbe"`:**
```
Tree (0-indexed, char in parens):
        0(a)
       /    \
      1(b)   2(a)
     /  \      \
    3(c) 4(b)   5(e)

dfs(3): leaf 'c'. best1=best2=0. ans=max(1,1)=1. return 1.
dfs(4): leaf 'b'. best1=best2=0. ans=max(1,1)=1. return 1.
dfs(1)='b': children 3('c'), 4('b').
   v=3: len=dfs(3)=1. s[3]='c' != s[1]='b' → best1=1, best2=0.
   v=4: len=dfs(4)=1. s[4]='b' == s[1]='b' → SKIP (same char, can't cross).
   ans=max(1, 1+0+1=2)=2. return best1+1=2.
dfs(5): leaf 'e'. best1=best2=0. ans stays 2. return 1.
dfs(2)='a': children 5('e').
   v=5: len=dfs(5)=1. s[5]='e' != s[2]='a' → best1=1, best2=0.
   ans=max(2, 1+0+1=2)=2. return best1+1=2.
dfs(0)='a': children 1('b'), 2('a').
   v=1: len=dfs(1)=2. s[1]='b' != s[0]='a' → best1=2, best2=0.
   v=2: len=dfs(2)=2. s[2]='a' == s[0]='a' → SKIP.
   ans=max(2, 2+0+1=3)=3. return best1+1=3.

Answer: ans = 3
Path: node 3(c) → node 1(b) → node 0(a) = "cba" (3 nodes, all adjacent differ) ✓
```

**Verified by compiling and running against the LC 2246 example: output matches 3.**

**General-tree idiom vs binary-tree idiom:** notice there's no `TreeNode* left/right` here — the tree is `vector<vector<int>> children[]` built from `parent[]`. The "top-2" pattern (`best1`, `best2`) generalizes "left vs right" from binary trees to "the best two branches among however many children exist." This same top-2 trick appears in general-tree diameter problems.

---

## SECTION 8 — TEMPLATE 6: SUM OF DISTANCES IN TREE (THE REROOTING TECHNIQUE)

This is the single most important upgrade in this pattern. Naively computing "sum of distances from node i to all others" for every `i` via a fresh BFS/DFS is O(n²) — too slow for n up to 3×10⁴ (LC constraint) and catastrophically slow for n = 10⁵. Rerooting solves ALL n queries in O(n) total.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 834 — Sum of Distances in Tree
// n nodes, n-1 edges (undirected tree). For every node i, compute
// the sum of distances from i to every other node.
//
// STEP 1 — Root the tree arbitrarily at node 0. Compute, via ONE
// postorder DFS:
//   count[u]  = size of the subtree rooted at u
//   answer[u] = sum of distances from u to every node IN u's OWN
//               SUBTREE ONLY (not the whole tree yet)
//
// Recurrence for answer[0] (the true answer for the root, since the
// root's "subtree" IS the whole tree):
//   answer[u] = sum over children v of ( answer[v] + count[v] )
//   (each node in v's subtree is 1 edge farther from u than from v)
//
// STEP 2 — REROOT. Walk the tree again (preorder, parent → child)
// and convert "subtree-only answer" into "whole-tree answer" for
// every other node using O(1) algebra per edge:
//
//   When the root moves from u to a child v:
//     - every node INSIDE v's subtree (count[v] of them) becomes
//       1 CLOSER to the new root.
//     - every node OUTSIDE v's subtree (n - count[v] of them,
//       including u itself) becomes 1 FARTHER from the new root.
//
//   answer[v] = answer[u] - count[v] + (n - count[v])
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    vector<int> answer, cnt;
    vector<vector<int>> graph;
    int n;

    vector<int> sumOfDistancesInTree(int N, vector<vector<int>>& edges) {
        n = N;
        graph.assign(n, {});
        cnt.assign(n, 1);
        answer.assign(n, 0);
        for (auto& e : edges) {
            graph[e[0]].push_back(e[1]);
            graph[e[1]].push_back(e[0]);
        }
        dfs1(0, -1);   // postorder: subtree sizes + subtree-only distance sums
        dfs2(0, -1);   // preorder: reroot correction
        return answer;
    }

    // Pass 1: children finish first (postorder). Fills cnt[] and answer[]
    // for the tree rooted at 0. answer[0] is ALREADY the final answer.
    void dfs1(int u, int parent) {
        for (int v : graph[u]) {
            if (v == parent) continue;
            dfs1(v, u);
            cnt[u]    += cnt[v];
            answer[u] += answer[v] + cnt[v];
        }
    }

    // Pass 2: parent finishes first (preorder). Converts answer[u]
    // (already correct) into answer[v] for each child v in O(1).
    void dfs2(int u, int parent) {
        for (int v : graph[u]) {
            if (v == parent) continue;
            answer[v] = answer[u] - cnt[v] + (n - cnt[v]);
            dfs2(v, u);
        }
    }
};
```

**FULL TRACE for `n=6`, `edges=[[0,1],[0,2],[2,3],[2,4],[2,5]]`:**

Tree rooted at 0 for pass 1:
```
        0
       / \
      1   2
         /|\
        3 4 5
```

**Pass 1 (postorder, dfs1):**
```
dfs1(1, 0): no children other than 0. cnt[1]=1, answer[1]=0.
  back at 0: cnt[0] += cnt[1] = 1 → cnt[0]=2
             answer[0] += answer[1] + cnt[1] = 0+1 = 1 → answer[0]=1

dfs1(2, 0): children 3, 4, 5.
  dfs1(3,2): leaf. cnt[3]=1, answer[3]=0.
    cnt[2] += 1 → cnt[2]=2. answer[2] += 0+1=1 → answer[2]=1.
  dfs1(4,2): leaf. cnt[4]=1, answer[4]=0.
    cnt[2] += 1 → cnt[2]=3. answer[2] += 0+1=1 → answer[2]=2.
  dfs1(5,2): leaf. cnt[5]=1, answer[5]=0.
    cnt[2] += 1 → cnt[2]=4. answer[2] += 0+1=1 → answer[2]=3.
  back at 0: cnt[0] += cnt[2]=4 → cnt[0]=6
             answer[0] += answer[2]+cnt[2] = 3+4=7 → answer[0]=1+7=8

After pass 1: cnt = [6,1,4,1,1,1]
              answer = [8, 0, 3, 0, 0, 0]   (only answer[0]=8 is FINAL so far)
```

**Pass 2 (preorder, dfs2), rerooting from 0:**
```
dfs2(0,-1): children 1, 2.
  v=1: answer[1] = answer[0] - cnt[1] + (n-cnt[1]) = 8 - 1 + (6-1) = 8-1+5 = 12 ✓
  dfs2(1,0): no further children.
  v=2: answer[2] = answer[0] - cnt[2] + (n-cnt[2]) = 8 - 4 + (6-4) = 8-4+2 = 6 ✓
  dfs2(2,0): children 3,4,5.
    v=3: answer[3] = answer[2] - cnt[3] + (n-cnt[3]) = 6 - 1 + 5 = 10 ✓
    v=4: answer[4] = answer[2] - cnt[4] + (n-cnt[4]) = 6 - 1 + 5 = 10 ✓
    v=5: answer[5] = answer[2] - cnt[5] + (n-cnt[5]) = 6 - 1 + 5 = 10 ✓

Final answer = [8, 12, 6, 10, 10, 10] ✓ (matches LC 834 example 1 exactly)
```

**Verified by compiling and running: output matches `[8, 12, 6, 10, 10, 10]` exactly.**

**Why the correction formula is `- cnt[v] + (n - cnt[v])`:** think of it as "every node in the universe either moved 1 closer or 1 farther when the root shifted from u to v." Nodes inside v's subtree (there are `cnt[v]` of them) are now on the "root side" of the edge — 1 step closer. Every other node (there are `n - cnt[v]` of them, including u) is now on the far side of the newly-crossed edge — 1 step farther. Net change = `(n - cnt[v]) - cnt[v]`, added to the parent's already-correct total.

---

## SECTION 9 — TEMPLATE 7: MAXIMUM PRODUCT OF SPLITTED BINARY TREE (SUBTREE SUMS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1339 — Maximum Product of Splitted Binary Tree
// Remove exactly one edge, splitting the tree into two components.
// Maximize the product of the two components' value sums. Return
// answer mod 1e9+7.
//
// Removing the edge above node X splits the tree into:
//   - the subtree rooted at X            (sum = subtreeSum(X))
//   - everything else                    (sum = totalSum - subtreeSum(X))
//
// So we need TWO passes:
//   Pass 1: compute totalSum (sum of the whole tree).
//   Pass 2: for every node X, compute subtreeSum(X) and update
//           best = max(best, subtreeSum(X) * (totalSum - subtreeSum(X))).
//
// (A single combined pass is possible but fragile — totalSum isn't
// known until the whole tree is visited, so cutting a corner and
// trying to reuse one traversal for both purposes invites bugs.
// Two clean O(n) passes are just as fast and much easier to reason
// about — and easier to keep bug-free under interview pressure.)
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    const int MOD = 1'000'000'007;
    long long totalSum = 0;
    long long best = 0;

    long long computeSum(TreeNode* node) {
        if (!node) return 0;
        return node->val + computeSum(node->left) + computeSum(node->right);
    }

    long long dfs(TreeNode* node) {
        if (!node) return 0;
        long long s = node->val + dfs(node->left) + dfs(node->right);
        best = max(best, (totalSum - s) * s);
        return s;
    }

    int maxProduct(TreeNode* root) {
        totalSum = computeSum(root);   // pass 1: total tree sum
        best = 0;
        dfs(root);                      // pass 2: try every possible cut
        return (int)(best % MOD);
    }
};
```

**FULL TRACE for tree `[1,2,3,4,5,6]`:**
```
      1
     / \
    2   3
   / \    \
  4   5    6

computeSum(root) = 1+2+3+4+5+6 = 21 = totalSum

dfs(4): leaf. s=4. best=max(0, (21-4)*4=68)=68. return 4.
dfs(5): leaf. s=5. best=max(68, (21-5)*5=80)=80. return 5.
dfs(2): s=2+4+5=11. best=max(80, (21-11)*11=110)=110. return 11.
dfs(6): leaf. s=6. best=max(110, (21-6)*6=90)=110 (no change). return 6.
dfs(3): s=3+0+6=9. best=max(110, (21-9)*9=108)=110 (no change). return 9.
dfs(1): s=1+11+9=21. best=max(110, (21-21)*21=0)=110 (no change). return 21.

Answer: 110 mod (1e9+7) = 110 ✓
Cut: the edge above node 2, splitting {2,4,5} (sum=11) from {1,3,6} (sum=10)...
```

Wait — let's double check which split gives 110: subtree sum 11 at node 2, remainder = 21-11 = 10, product = 110. That's the split at node 2's incoming edge: `{2,4,5}` sum=11 vs `{1,3,6}` sum=10. Product = 11×10 = 110. Matches.

**Verified by compiling and running against the LC 1339 example: output matches 110.**

**The mod-and-overflow trap:** `totalSum` and subtree sums can each be up to ~5×10⁴ × n (node values up to 5×10⁴, n up to 5×10⁴ nodes) ≈ 2.5×10⁹ — already overflows a 32-bit int. The PRODUCT of two such sums can reach ~6×10¹⁸ — right at the edge of `long long` range (max ≈ 9.2×10¹⁸), so `long long` is safe but `int` is not. Apply `% MOD` only at the very end, never mid-computation (comparing "best" requires the true, un-modded product, since modding early breaks the `max()` comparison).

---

## SECTION 10 — TEMPLATE 8: NUMBER OF GOOD PATHS (DSU ON TREE / VALUES SORTED)

This is the hardest problem in this pattern — it combines tree structure with Union-Find (Pattern 13), processed in VALUE order rather than tree order.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 2421 — Number of Good Paths
// n nodes, tree edges, vals[i] = value at node i. A "good path" is
// a path between two nodes (possibly the same node) such that every
// node ON the path has a value <= max(vals of the two endpoints),
// i.e. the endpoints hold the MAXIMUM value along the path (no node
// strictly in between exceeds that value).
//
// KEY INSIGHT: process nodes in INCREASING order of value. When we
// process node u, union it with any neighbor v whose value <= vals[u]
// (i.e., v has already been processed, or has equal value). This
// builds up components where every node inside currently has value
// <= the "current threshold" vals[u].
//
// Track, per DSU component root:
//   mxVal[root] = the maximum value currently inside that component
//   cnt[root]   = how many nodes with value == mxVal[root] live in it
//
// When we UNION two components that both have mxVal == vals[u] (the
// current threshold), EVERY max-value node in one component can now
// reach EVERY max-value node in the other via a path whose interior
// values are all <= vals[u] — that's cnt[ru] * cnt[rv] NEW good paths.
//
// Every single node, alone, is trivially a good path (path of length 0)
// — start the answer at n, not 0.
// ─────────────────────────────────────────────────────────────────

class Solution {
public:
    vector<int> par, cnt, mxVal;

    int find(int x) {
        while (par[x] != x) { par[x] = par[par[x]]; x = par[x]; }  // path halving
        return x;
    }

    int numberOfGoodPaths(vector<int>& vals, vector<vector<int>>& edges) {
        int n = vals.size();
        par.resize(n); iota(par.begin(), par.end(), 0);
        cnt.assign(n, 1);      // each singleton has 1 max-value node: itself
        mxVal = vals;           // each singleton's max value is its own value

        vector<vector<int>> adj(n);
        for (auto& e : edges) {
            adj[e[0]].push_back(e[1]);
            adj[e[1]].push_back(e[0]);
        }

        vector<int> order(n);
        iota(order.begin(), order.end(), 0);
        sort(order.begin(), order.end(), [&](int a, int b) { return vals[a] < vals[b]; });

        long long good = n;   // every node alone is a good path

        for (int u : order) {
            for (int v : adj[u]) {
                if (vals[v] > vals[u]) continue;   // only union with <= (already processed)
                int ru = find(u), rv = find(v);
                if (ru == rv) continue;

                if (mxVal[ru] == mxVal[rv]) {
                    // both components currently peak at the same value —
                    // every max-value node in one can now pair with every
                    // max-value node in the other
                    good += (long long)cnt[ru] * cnt[rv];
                    cnt[ru] += cnt[rv];
                    par[rv] = ru;
                } else if (mxVal[ru] > mxVal[rv]) {
                    par[rv] = ru;   // ru's higher peak absorbs rv; rv's max-count is now irrelevant
                } else {
                    par[ru] = rv;   // symmetric case
                }
            }
        }
        return (int)good;
    }
};
```

**FULL TRACE for `vals=[1,3,2,1,3]`, `edges=[[0,1],[0,2],[2,3],[2,4]]`:**
```
Tree:
      0(1)
     /    \
   1(3)   2(2)
          /   \
        3(1)  4(3)

Processing order by value (ties broken arbitrarily): value 1 → nodes {0,3};
value 2 → node {2}; value 3 → nodes {1,4}.

good starts at n = 5 (each node alone is a good path).

Process node 0 (val=1): neighbors 1(val=3, SKIP >1), 2(val=2, SKIP >1). No unions.
Process node 3 (val=1): neighbor 2(val=2, SKIP >1). No unions.
Process node 2 (val=2): neighbors 0(val=1<=2, union), 3(val=1<=2, union), 4(val=3, SKIP >2).
   union(2,0): mxVal[2]=2, mxVal[0]=1 → different, mxVal[2]>mxVal[0] → par[0]=2 (or root(0)) absorbed.
   union(2,3): mxVal[find(2)]=2, mxVal[find(3)]=1 → different → absorb 3 into 2's root.
   No same-mxVal merges yet → no new good paths from this step.
Process node 1 (val=3): neighbor 0(val=1<=3, union with node1's singleton).
   ru=find(1) [mxVal=3, cnt=1], rv=find(0)=root of {0,2,3} component [mxVal=2, cnt=1(node2)].
   mxVal differ (3 vs 2) → absorb the smaller-peak component into node 1's root (par[rv]=ru).
Process node 4 (val=3): neighbor 2(val=2<=3, union).
   ru=find(4) [mxVal=3, cnt=1], rv=find(2) → now merged into node1's component [mxVal=3, cnt=1 (only node1, since previous merge didn't touch cnt because mxVal differed)].
   NOW mxVal[ru]==mxVal[rv]==3 → good += cnt[ru]*cnt[rv] = 1*1 = 1. good = 5+1 = 6.
   cnt[ru] += cnt[rv] → cnt=2 (nodes 1 and 4, both value 3, now in one component).

Final good = 6 ✓ (matches LC 2421 example 1)
```

**Verified by compiling and running against 3 test cases (including the LC 2421 published examples): all outputs match — example 1 gives 6, example 2 (`vals=[1,1,2,2,3]`) gives 7, single-node case gives 1.**

**Why sort by VALUE, not by tree structure:** a "good path" is defined by a value-domination property, not a structural one. Union-Find with values processed in ascending order is exactly the "Kruskal's-algorithm-style" trick: we only ever connect through edges/nodes that are "small enough," which guarantees that when two same-peak components merge, every interior node genuinely has value ≤ that peak.

---

## SECTION 11 — BOTH IDIOMS: BINARY TREE vs GENERAL TREE

```cpp
// ── BINARY TREE IDIOM ──────────────────────────────────────────
struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

int dfsBinary(TreeNode* node) {
    if (!node) return /* base case */ 0;
    int L = dfsBinary(node->left);
    int R = dfsBinary(node->right);
    // combine L, R, node->val
    return /* value returned to parent */ 0;
}

// ── GENERAL TREE IDIOM (adjacency list, must track parent) ────
vector<int> adj[100005];   // or vector<vector<int>> adj(n)

int dfsGeneral(int u, int parent) {
    int acc = /* base case, e.g. 0 or 1 */ 0;
    for (int v : adj[u]) {
        if (v == parent) continue;   // CRITICAL: undirected edges go both ways —
                                       // without this check you'll recurse back
                                       // into the node you came from and infinite-loop
        int childResult = dfsGeneral(v, u);
        acc = /* combine childResult into acc */ acc;
    }
    return acc;
}

// ── GENERAL TREE VIA EXPLICIT parent[] ARRAY (LC 2246 style) ──
// Build children[] once, then dfs never needs a "parent" check because
// the tree is already directed root → leaves.
vector<vector<int>> children(n);
for (int i = 1; i < n; i++) children[parent[i]].push_back(i);
```

**The one rule that prevents 90% of general-tree bugs:** if your tree is given as an undirected adjacency list, you MUST pass (and check) the parent in every recursive call. If it's given as a `parent[]` array, build a `children[]` list ONCE up front and you never need parent-checking again — the recursion is naturally rooted and directed.

---

## SECTION 12 — COMPLEXITY TABLE

| Problem | Time | Space (aux) | State returned | Notes |
|---|---|---|---|---|
| Diameter of Binary Tree | O(n) | O(h) recursion | height (int) | Global var tracks diameter |
| House Robber III | O(n) | O(h) recursion | {rob, skip} pair | No global var needed |
| Binary Tree Max Path Sum | O(n) | O(h) recursion | best straight path | Global var tracks bent path |
| Binary Tree Cameras | O(n) | O(h) recursion | 3-state enum | Greedy-DP, global camera count |
| Longest Path Diff Chars | O(n) | O(h) recursion + O(n) children lists | best straight path length | Top-2 children generalization |
| Sum of Distances in Tree | O(n) total (2 passes) | O(n) | count + subtree-answer, then rerooted answer | THE rerooting template |
| Max Product Split Tree | O(n) (2 passes) | O(h) recursion | subtree sum | mod applied only at the end |
| Number of Good Paths | O(n α(n) log n) | O(n) | DSU root state | Sort by value, union-find |
| Count Good BSTs (Catalan) | O(n²) or O(n log n) | O(n) | subtree count | Combinatorial, see Problem Set |
| Min Cost Tree From Leaves | O(n²) or O(n log n) (interval DP) | O(n²) | interval [i,j] max, min cost | Interval DP, not recursion-on-tree |
| Tree of Coprimes | O(n × 50) | O(n × 50) | ancestor stack per prime factor | gcd-lookup trick |

`h` = tree height (O(log n) balanced, O(n) worst-case skewed — see Common Mistakes for the stack-overflow implication).

---

## SECTION 13 — VARIANTS

1. **Maximum Independent Set on Trees (generalized House Robber III):** same take/skip pair idea, generalized to arbitrary weights and arbitrary branching (sum over ALL children's `skip` when taking a node, sum over `max(take,skip)` when not).

2. **Tree Diameter with weighted edges:** identical to LC 543 but `height()` accumulates edge WEIGHTS instead of counting 1 per edge: `return edgeWeight + max(lh, rh)`.

3. **Diameter on a general tree (not binary):** use the top-2-children trick from LC 2246 — track the best and second-best downward depth among all children, update `diameter = max(diameter, best1 + best2)`.

4. **Path Sum III (LC 437) and prefix-sum-on-tree:** a different flavor — uses a running prefix-sum hashmap during DFS, not a returned "gain" value. Worth knowing it's a DIFFERENT technique (prefix sums, Pattern 06) applied to tree traversal, not classic tree DP.

5. **Minimum Cost Tree From Leaf Values (LC 1130):** looks like tree DP ("build an optimal tree") but the efficient solution is a **monotonic stack** (O(n)) or **interval DP** (O(n³) naive, O(n²) with the right recurrence) over the leaf ARRAY — the actual tree structure is never built or recursed over. Included in the Problem Set as a contest-tier bridge to Pattern 23.

6. **Tree of Coprimes (LC 1766):** for each node, find its nearest ANCESTOR whose value is coprime with it. Solved via DFS with a stack-per-value-factor: maintain, for each of the up to 50 possible values, a stack of `(node, depth)` pairs seen on the current root-to-node path; at each node, check all ≤50 values coprime with `vals[u]` and take the deepest ancestor among their stacks. Backtrack (pop) on the way out of the DFS. This is tree DP married to a rolling ancestor index — a distinct but closely related technique.

7. **Rerooting for OTHER quantities:** the same two-pass skeleton from LC 834 generalizes to "diameter as seen from every node," "height of tree if rooted at node i," "number of nodes within distance k of node i," etc. The general recipe: Pass 1 computes a subtree-local answer + subtree size; Pass 2 propagates an O(1) correction per edge.

---

## SECTION 14 — COMMON MISTAKES

### Mistake 1: Returning the "bent" value instead of the "straight" value (Path Sum family)

```cpp
// WRONG — returns the value that already uses BOTH children
int maxGain(TreeNode* node) {
    if (!node) return 0;
    int L = max(maxGain(node->left), 0);
    int R = max(maxGain(node->right), 0);
    best = max(best, node->val + L + R);
    return node->val + L + R;   // BUG: this "bent" value handed to the
                                  // parent would let the parent think it
                                  // can extend through BOTH of this
                                  // node's children — but a real path
                                  // can only continue in ONE direction.
}

// CORRECT — return only the best SINGLE-direction extension
return node->val + max(L, R);
```

---

### Mistake 2: Forgetting to clamp negative child contributions to 0

```cpp
// WRONG — a negative subtree gain gets added anyway, dragging the sum down
int leftGain = maxGain(node->left);
int rightGain = maxGain(node->right);
best = max(best, node->val + leftGain + rightGain);

// CORRECT — a negative gain should simply not be walked into
int leftGain  = max(maxGain(node->left),  0);
int rightGain = max(maxGain(node->right), 0);
```
Without the clamp, a subtree with sum -1000 would subtract 1000 from every ancestor's computation, even though the correct choice is to simply not extend the path into that subtree at all.

---

### Mistake 3: Recomputing "answer rooted at every node" from scratch — O(n²)

```cpp
// WRONG — O(n) work per node, O(n) nodes = O(n²) total. TLEs for n > ~5000.
vector<int> result(n);
for (int i = 0; i < n; i++) {
    result[i] = bfsSumOfDistances(i);   // full O(n) traversal EVERY time
}

// CORRECT — rerooting: ONE postorder pass + ONE preorder pass = O(n) total
dfs1(0, -1);   // subtree-local answers
dfs2(0, -1);   // O(1) correction per edge, propagated root → leaves
```
This is the single highest-leverage upgrade in this whole pattern — recognizing "I need the answer FOR EVERY NODE AS ROOT" as a rerooting problem, not n independent single-source problems.

---

### Mistake 4: Stack overflow on deep/skewed trees

```cpp
// WRONG (in spirit) — assuming recursion is always safe
// A skewed tree (e.g., built by repeatedly inserting increasing values
// into a BST with no balancing) can have depth n. For n = 10^5, plain
// recursion can exceed the default stack size (commonly ~1MB / ~8MB)
// and crash with a segfault — NOT a clean runtime error, making it hard
// to diagnose in an interview or contest setting.

// MITIGATIONS:
// 1. Convert to an explicit stack-based iterative postorder traversal
//    (mechanical but verbose — push {node, state} pairs).
// 2. In competitive programming, request a larger stack size at the
//    OS/judge level if permitted (not always possible on LeetCode).
// 3. Recognize the risk EXPLICITLY in an interview: "this recursion
//    assumes O(log n) depth; for adversarial/skewed inputs I'd convert
//    to an iterative traversal with an explicit stack."
```

---

### Mistake 5: General tree — forgetting the parent check, causing infinite recursion

```cpp
// WRONG — undirected adjacency list, no parent tracking
int dfs(int u) {
    int acc = 0;
    for (int v : adj[u]) {
        acc += dfs(v);   // BUG: v might be u's PARENT (edges are bidirectional
                          // in an undirected adjacency list) — this recurses
                          // straight back up, infinite loop / stack overflow.
    }
    return acc;
}

// CORRECT — always pass and check parent
int dfs(int u, int parent) {
    int acc = 0;
    for (int v : adj[u]) {
        if (v == parent) continue;
        acc += dfs(v, u);
    }
    return acc;
}
```

---

### Mistake 6: DSU-on-values — unioning in tree order instead of value order (Number of Good Paths)

```cpp
// WRONG — running a plain DFS/BFS union of the whole tree first,
// THEN trying to figure out "good paths" from the fully-merged
// single component. Once everything is merged, you've lost the
// information about WHICH value-threshold each merge happened at
// — you can no longer tell whether a path's interior values ever
// exceeded the endpoints.

// CORRECT — sort NODES by value; only union a node with neighbors
// whose value is <= the current node's value. This guarantees that
// every merge you perform corresponds to "connecting through nodes
// with value <= the current threshold," which is exactly the
// good-path condition.
sort(order.begin(), order.end(), [&](int a, int b){ return vals[a] < vals[b]; });
```

---

## SECTION 15 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Diameter of Binary Tree | 543 | Return height, update global bent-path |
| 2 | House Robber III | 337 | Return {rob, skip} pair |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Binary Tree Maximum Path Sum | 124 | Return ONE side, update with BOTH; negative clamp |
| 4 | Binary Tree Cameras | 968 | 3-state greedy/DP, null returns "covered" |
| 5 | Longest Path With Different Adjacent Characters | 2246 | General tree, top-2 children |

### STRETCH (solve in ≤ 45 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Sum of Distances in Tree | 834 | REROOTING — two DFS passes, O(n) for all nodes |
| 7 | Maximum Product of Splitted Binary Tree | 1339 | Subtree sums vs total sum, mod at the end only |
| 8 | Count Number of Maximum BSTs | — | Catalan-number counting via subtree DP |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Minimum Cost Tree From Leaf Values | 1130 | Interval DP / monotonic stack (bridge to Pattern 23) |
| 10 | Tree of Coprimes | 1766 | Ancestor-stack-per-value during DFS |
| 11 | Number of Good Paths | 2421 | DSU on tree, values sorted, merge-time counting |

---

## SECTION 16 — PATTERN CONNECTIONS

1. **Pattern 08 (Tree DFS/BFS Traversal):** Pattern 08 teaches you HOW to walk a tree — preorder, inorder, postorder, level-order. Pattern 22 is entirely built on postorder discipline: every template in this document requires children to fully finish before the parent computes anything. If Pattern 08 wasn't second nature, tree DP will feel unstable.

2. **Pattern 16 (DP Fundamentals — 1D):** the take/skip pairing in House Robber III (LC 337) is a direct structural echo of House Robber I/II from Pattern 16 (`dp[i] = max(dp[i-1], dp[i-2]+nums[i])`). The only difference is that "previous element" becomes "children," and the linear chain becomes a tree.

3. **Pattern 13 (Union-Find / DSU):** Number of Good Paths (LC 2421) is impossible without a solid DSU implementation (path compression at minimum; union by rank/size for the tightest complexity). This pattern assumes Pattern 13 is already fluent — DSU is used here as a TOOL, not re-taught.

4. **Pattern 23 (Interval DP):** Minimum Cost Tree From Leaf Values (LC 1130) is included in this pattern's problem set specifically because it LOOKS like tree DP (you're building a tree!) but the efficient solution is interval DP over the leaf array — `dp[i][j]` = min cost to combine leaves i..j into one tree, with `dp[i][j] = min over k in [i,j) of dp[i][k] + dp[k+1][j] + maxLeaf(i,k)*maxLeaf(k+1,j)`. Solving it here previews Pattern 23's core recurrence shape.

5. **Pattern 06 (Prefix Sums) — tangential:** Path Sum III (LC 437, mentioned in Variants) combines tree DFS with a prefix-sum hashmap, illustrating that not every "count paths in a tree" problem is classic tree DP — some are prefix-sum problems that merely happen to run on top of a tree traversal.

---

## SECTION 17 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 124 — Binary Tree Maximum Path Sum

**Interviewer:** "Find the maximum path sum in a binary tree. A path can start and end at any node, and node values may be negative."

**Candidate:**

*[First minute — establish the core distinction before writing code]*
> "The key subtlety: a 'path' can bend at most once — at whichever node is its highest point. If I'm standing at node X, deciding X's contribution to my PARENT's path, I can only extend in ONE direction (left or right), because the path is already 'using up' the edge going to my parent. But if X itself is the BEND point of the best overall path, X is allowed to use both children. So I need two different quantities: what I RETURN (single direction) and what I use to UPDATE the answer (both directions)."

*[State the negative-clamp explicitly]*
> "Also, if a child's best contribution is negative, I should treat it as 0 — meaning 'don't walk into that child at all,' since including a negative number can only hurt."

*[Code — 5 minutes]*
```cpp
int best = INT_MIN;
int maxGain(TreeNode* node) {
    if (!node) return 0;
    int L = max(maxGain(node->left), 0);
    int R = max(maxGain(node->right), 0);
    best = max(best, node->val + L + R);
    return node->val + max(L, R);
}
int maxPathSum(TreeNode* root) { best = INT_MIN; maxGain(root); return best; }
```

*[Interviewer]: "Why is `best` initialized to INT_MIN rather than 0?"*
> "Because all node values could be negative — e.g., a single node with value -5. The correct answer is -5 (you must include at least one node), not 0. If I initialized `best` to 0, I'd wrongly report 0 as if 'choosing an empty path' were allowed."

**RED FLAGS:**
- Returning `node->val + L + R` (the bent value) to the parent instead of `node->val + max(L, R)`.
- Forgetting `max(childGain, 0)`, letting negative subtrees corrupt every ancestor's sum.
- Initializing `best = 0`, silently producing wrong answers on all-negative trees.

---

### Interview Simulation 2: LC 834 — Sum of Distances in Tree

**Interviewer:** "Given an unrooted tree of n nodes, for every node compute the sum of distances to all other nodes."

**Candidate:**

*[Immediately name the naive approach and its cost, before proposing the fix]*
> "The naive approach — BFS from every node — is O(n) per node, O(n²) total. For n up to 3×10⁴ that's ~10⁹ operations, likely too slow. I want the rerooting technique: two DFS passes, O(n) total."

*[Explain both passes conceptually before coding]*
> "Pass 1: root the tree arbitrarily at node 0, and in postorder compute `cnt[u]` (subtree size) and `answer[u]` (sum of distances from u to nodes IN u's subtree only). After this pass, `answer[0]` is already correct — the root's subtree IS the whole tree.
>
> Pass 2: walk root-to-leaves. When the root conceptually moves from a parent u to child v, every node inside v's subtree gets 1 closer, every node outside gets 1 farther. So `answer[v] = answer[u] - cnt[v] + (n - cnt[v])`. This is O(1) per edge."

*[Code — 6 minutes, then trace on a small example to confirm]*

*[Interviewer]: "Convince me the O(1) correction is actually correct, not just plausible."*
> "Partition all n nodes into two sets relative to the edge (u,v): the `cnt[v]` nodes inside v's subtree, and the `n - cnt[v]` nodes everywhere else, including u. Every node in the first set is now 1 step closer to the new root v than it was to u — that's a total change of `-cnt[v]`. Every node in the second set is now 1 step farther — that's `+(n - cnt[v])`. The distances between any two nodes NOT on the u-v boundary don't change relative to each other; only their distance to the (shifted) root changes. Sum these deltas onto `answer[u]` and you get `answer[v]` exactly."

**RED FLAGS:**
- Jumping straight to code without naming rerooting — signals memorized-pattern-matching rather than understanding.
- Getting the correction formula backwards (`+cnt[v] - (n-cnt[v])`) — always sanity check on a 2-node tree by hand.
- Not initializing `answer[0]` correctly in pass 1 (forgetting `+ cnt[v]` when accumulating from children).

---

### Interview Simulation 3: Meta-discussion — When do you need rerooting vs. a single DFS?

**Interviewer:** "How do you recognize that a tree problem needs rerooting instead of one traversal?"

**Candidate:**
> "The signal is in the question, not the tree: if the problem asks for a SINGLE global answer (diameter, max path sum, is this tree balanced), one postorder DFS from any fixed root suffices — the root choice doesn't matter because the traversal touches every node and every edge regardless of orientation.
>
> But if the problem asks 'compute X **for every node as if it were the root**' — sum of distances, height if rooted here, eccentricity — then a fixed-root DFS only gives you the right answer for ONE node (whichever you happened to root at). Naively repeating the DFS for every candidate root is O(n) work times O(n) roots = O(n²).
>
> Rerooting fixes this by observing that the answer at a CHILD can usually be derived from the answer at its PARENT with an O(1) correction, once you've done one bookkeeping pass (usually subtree sizes or subtree-local aggregates) to know exactly how much changes when the root shifts by one edge. The two-pass structure — postorder to gather, preorder to propagate — is the tell-tale shape."

**RED FLAGS:**
- Confusing "I need to visit every node" (any single DFS does this) with "I need the answer relative to every node as root" (requires rerooting).
- Not being able to state, precisely, WHAT quantity is corrected in O(1) when the root shifts by one edge — this is the crux of every rerooting problem and differs per problem (distance sum, subtree count, max depth, etc.).

---

## SECTION 18 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: In Binary Tree Maximum Path Sum, why can a node use BOTH children when updating the global best, but only ONE child when returning a value to its parent?**

> A "path" is a simple chain of nodes with no branching. If a node X's contribution is returned to X's parent P, then P intends to potentially extend the path further upward through P. That means the path segment coming UP from X can only travel in one direction through X (it can't fork). But if X itself is the topmost point ("bend") of the best path in the whole tree, X doesn't need to extend anywhere else — it's allowed to be a "V" shape using both children, because nothing above X is part of this particular path.

---

**Q2: Why does House Robber III return a PAIR {rob, skip} instead of a single "best" value per node?**

> Because the parent's decision to rob ITSELF depends on whether each child was robbed or not — specifically, if the parent robs itself, its children must NOT be robbed (skip only). If we collapsed each child to a single `max(rob, skip)` value, the parent would lose the ability to distinguish "the child's best value assuming it wasn't robbed" from "the child's best value overall" — and might illegally combine "parent robbed" with "child also robbed."

---

**Q3: What does the `max(childGain, 0)` clamp in LC 124 actually mean geometrically?**

> It means "don't walk into this child at all if doing so would only subtract value." A path doesn't have to touch every node — it can start and stop wherever is optimal. If a child's best downward contribution is negative, the optimal path simply excludes that entire branch, contributing exactly 0 (not a negative number) to the parent's calculation.

---

**Q4: In Sum of Distances in Tree, why is `answer[0]` (from pass 1 alone) already correct, but `answer[v]` for any other node is NOT correct until pass 2 runs?**

> `answer[0]` is built entirely from `answer[u] += answer[child] + count[child]` accumulated across the WHOLE tree, because node 0's "subtree" (as rooted at 0) IS the entire tree — there's nothing outside it to account for. Every other node's subtree (in the arbitrary rooting at 0) is only a PART of the tree; `answer[v]` after pass 1 only counts distances to nodes inside v's own subtree, completely ignoring all the nodes "above" v (v's ancestors and their other branches). Pass 2 is what folds in those missing "outside the subtree" distances via the O(1) reroot correction.

---

**Q5: Why does Binary Tree Cameras treat a null child as state 1 ("covered"), not state 0 ("not covered")?**

> If null returned 0, then EVERY leaf node (whose children are both null) would see `left==0 || right==0` and be forced to place a camera on itself — even though placing that camera one level up (at the leaf's parent) would cover strictly more nodes for the same cost. Returning 1 for null means "there's nothing there that needs covering," so a leaf correctly computes to state 0 (not covered) instead, correctly deferring the camera placement to its parent — which is the greedy-optimal choice.

---

**Q6: Why is sorting nodes by VALUE (not traversing in tree order) essential to Number of Good Paths?**

> A "good path" requires every node ON the path to have a value ≤ the max of the two endpoints. If we processed nodes in ordinary tree order (e.g., BFS/DFS from the root), we could end up merging two components through a high-value node before we've correctly established what "the max value along the connecting route" actually is. By processing strictly in increasing value order and only unioning a node with neighbors of ≤ its own value, every merge we perform is guaranteed to only pass through values ≤ the current threshold — which is exactly the good-path condition, verified incrementally and correctly.

---

**Q7: You're given a tree problem that asks "for every node, compute [some value] as if it were the root." What's your first move, before writing any code?**

> Identify what LOCAL, subtree-based quantity you can compute in one postorder pass (usually a size, a sum, or a count), rooted arbitrarily (node 0 is fine). Then ask: "when the conceptual root shifts from a node to one of its children, what changes, and can I express that change in O(1) using only the subtree-local quantity I already computed?" If yes, you have a rerooting solution in O(n) total. If the "what changes" step genuinely requires re-scanning the whole tree, rerooting doesn't apply and you likely need a fundamentally different technique.

---

## SECTION 19 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. POSTORDER DP SKELETON (single returned value)
// ─────────────────────────────────────────────────────────────────
int dfs(TreeNode* node) {
    if (!node) return /* base case */;
    int L = dfs(node->left);
    int R = dfs(node->right);
    /* update any global tracker using L, R, node->val (the "bend") */
    return /* value to hand to parent — usually ONE direction only */;
}

// ─────────────────────────────────────────────────────────────────
// 2. TAKE/SKIP PAIR SKELETON
// ─────────────────────────────────────────────────────────────────
pair<int,int> dfs(TreeNode* node) {   // {take, skip}
    if (!node) return {0, 0};
    auto [lt, ls] = dfs(node->left);
    auto [rt, rs] = dfs(node->right);
    int take = node->val + ls + rs;          // children must be skipped
    int skip = max(lt, ls) + max(rt, rs);     // children free to choose
    return {take, skip};
}

// ─────────────────────────────────────────────────────────────────
// 3. PATH DP SKELETON (return one side, update with both)
// ─────────────────────────────────────────────────────────────────
int best = INT_MIN;
int dfs(TreeNode* node) {
    if (!node) return 0;
    int L = max(dfs(node->left), 0);   // clamp negatives
    int R = max(dfs(node->right), 0);
    best = max(best, node->val + L + R);   // bend — global only
    return node->val + max(L, R);           // straight — returned
}

// ─────────────────────────────────────────────────────────────────
// 4. MULTI-STATE (CAMERAS) SKELETON
// ─────────────────────────────────────────────────────────────────
// 0 = not covered, 1 = covered no camera, 2 = has camera
int cameras = 0;
int dfs(TreeNode* node) {
    if (!node) return 1;
    int L = dfs(node->left), R = dfs(node->right);
    if (L == 0 || R == 0) { cameras++; return 2; }
    if (L == 2 || R == 2) return 1;
    return 0;
}

// ─────────────────────────────────────────────────────────────────
// 5. GENERAL TREE, TOP-2 CHILDREN SKELETON
// ─────────────────────────────────────────────────────────────────
int ans = 0;
int dfs(int u) {
    int best1 = 0, best2 = 0;
    for (int v : children[u]) {
        int len = dfs(v);
        if (/* v incompatible with u, e.g. same char */) continue;
        if (len > best1) { best2 = best1; best1 = len; }
        else if (len > best2) best2 = len;
    }
    ans = max(ans, best1 + best2 + /* combine constant */);
    return best1 + /* extend constant */;
}

// ─────────────────────────────────────────────────────────────────
// 6. REROOTING SKELETON (two-pass)
// ─────────────────────────────────────────────────────────────────
vector<int> subtreeAns(n, 0), subtreeSize(n, 1), finalAns(n, 0);

void dfs1(int u, int parent) {      // postorder: gather
    for (int v : adj[u]) {
        if (v == parent) continue;
        dfs1(v, u);
        subtreeSize[u] += subtreeSize[v];
        subtreeAns[u]  += subtreeAns[v] + subtreeSize[v];   // problem-specific combine
    }
}
void dfs2(int u, int parent) {      // preorder: propagate correction
    for (int v : adj[u]) {
        if (v == parent) continue;
        finalAns[v] = finalAns[u] - subtreeSize[v] + (n - subtreeSize[v]);  // problem-specific
        dfs2(v, u);
    }
}
// driver:
subtreeAns[0] handled by dfs1; finalAns[0] = subtreeAns[0]; dfs1(0,-1); dfs2(0,-1);

// ─────────────────────────────────────────────────────────────────
// 7. GENERAL TREE FROM UNDIRECTED EDGES — ALWAYS TRACK PARENT
// ─────────────────────────────────────────────────────────────────
vector<vector<int>> adj(n);
for (auto& e : edges) { adj[e[0]].push_back(e[1]); adj[e[1]].push_back(e[0]); }
int dfs(int u, int parent) {
    for (int v : adj[u]) if (v != parent) dfs(v, u);
    return 0;
}

// ─────────────────────────────────────────────────────────────────
// 8. DSU-ON-VALUES SKELETON (Number of Good Paths style)
// ─────────────────────────────────────────────────────────────────
vector<int> order(n); iota(order.begin(), order.end(), 0);
sort(order.begin(), order.end(), [&](int a,int b){ return vals[a] < vals[b]; });
for (int u : order)
    for (int v : adj[u])
        if (vals[v] <= vals[u]) /* union find(u), find(v), accumulate at merge */;
```

---

## SECTION 20 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order.** Sum of Distances in Tree (rerooting) and Number of Good Paths (DSU-on-values) are the two that require genuinely new machinery — budget 45–60 minutes each and don't be discouraged if the first attempt fails.

2. **Practice articulating "what does this subtree return" BEFORE writing any code**, for every new tree DP problem you encounter. This single habit is what separates candidates who solve tree DP fluently from those who pattern-match and get stuck the moment a variant deviates from a memorized template.

3. **Binary Tree Maximum Path Sum → Longest Univalue Path (LC 687) → Longest Path With Different Adjacent Characters (LC 2246):** solve these three back to back. They share the exact same "return straight, update with bend" skeleton, just with different compatibility conditions (any value vs. same value vs. different value) and different tree representations (binary vs. general).

4. **Rerooting deep dive:** after LC 834, attempt "for every node, compute the height of the tree if rooted there" and "for every node, compute the number of nodes within distance k" — both use the exact same two-pass skeleton with a different O(1) correction formula. This cements rerooting as a REUSABLE technique, not a memorized one-off.

5. **Bridge to Pattern 23:** solve Minimum Cost Tree From Leaf Values (LC 1130) and notice that despite "tree" being in the problem's DNA, the efficient solution never recurses over an actual tree object — it's interval DP over an array. This is intentional: it previews Pattern 23 and reinforces that "involves a tree" and "is solved via tree-recursion" are not the same thing.

---

## SECTION 21 — SIGN-OFF CRITERIA

### Tier 1 — Postorder DP mastered
You solve Diameter of Binary Tree and House Robber III cleanly, and can explain in under 30 seconds why House Robber III needs a pair return while Diameter needs only a single int plus a global tracker.

### Tier 2 — Path DP and multi-state DP solid
You solve Binary Tree Maximum Path Sum without looking up the code, correctly explaining the "return one side, update with both" distinction and the negative-clamp. You solve Binary Tree Cameras and can justify all three states and why null returns "covered."

### Tier 3 — General trees and rerooting understood
You solve Longest Path With Different Adjacent Characters using the top-2-children generalization. You solve Sum of Distances in Tree using two clean DFS passes and can derive the O(1) correction formula from first principles (not from memory) on a fresh whiteboard example.

### Tier 4 — Advanced combinations complete
You solve Maximum Product of Splitted Binary Tree with correct overflow handling (long long, mod applied only at the end). You solve Number of Good Paths using DSU-on-values and can explain why sorting by value (not tree order) is essential. You can name the stack-overflow risk on skewed trees unprompted and describe at least one mitigation.

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 124 (Maximum Path Sum — the return-vs-update distinction must be explained), LC 834 (Sum of Distances in Tree — the rerooting derivation must be your own, not recited), and LC 968 (Binary Tree Cameras — the 3-state greedy-DP reasoning must be complete). These three represent the three hardest conceptual challenges in this pattern.

---

*Pattern 22 Complete — DP on Trees*
*Next: Pattern 23 — Interval DP (Matrix Chain Multiplication, Burst Balloons, Minimum Cost Tree From Leaf Values — the "combine two ranges" family)*
