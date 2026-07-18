# PATTERN 30 — TOPOLOGICAL SORT
## Kahn's Algorithm, DFS-Based Ordering, Cycle Detection, DAG DP — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of topological sort:**

Topological sort is one of the easiest patterns to *describe* and one of the easiest to *botch* under pressure. Almost every 1600-level candidate can say "find nodes with in-degree zero, process them, decrement neighbors' in-degree." Far fewer can write it cleanly in under five minutes, correctly orient every edge, and — critically — remember the ONE check that separates a correct solution from a silently wrong one: **`processedCount == V` at the end. If not, there's a cycle.**

**The recognition signal is verbal, not structural.** Watch for these phrases in a problem statement:
- "must happen before" / "must be completed before"
- "prerequisite" / "dependency" / "depends on"
- "build order" / "compile order" / "install order"
- "course schedule" / "task scheduling with constraints"
- "valid sequence given these constraints"
- "X must come before Y in the result"

Any of these should trigger an immediate mental template: **this is a DAG (or I need to verify it is one), and I need a linear order consistent with every directed edge `u → v` meaning "u before v."**

**The specific gap between 1600 and 2000:**
- At 1600: you recognize "prerequisites" as "probably some graph thing," look up Kahn's algorithm syntax, and often get the edge direction backward (`prerequisites[i] = [a, b]` — is the edge `a→b` or `b→a`?).
- At 2000: you write Kahn's algorithm from memory in under 3 minutes, know instantly that a DFS-based topo sort requires appending in **post-order and reversing**, can detect cycles two different ways (queue starvation in Kahn's, gray-node revisit in DFS), and recognize when a problem is topological sort **wearing a disguise** — Alien Dictionary (build the graph yourself from string comparisons), Longest Increasing Path in a Matrix (implicit DAG, no explicit edges), Sort Items by Groups (topological sort at two levels simultaneously).

**The three things that separate mastery from memorization:**

1. **Kahn's algorithm from memory, cold, no notes.** In-degree array, queue seeded with zero-in-degree nodes, decrement-and-push. If you have to think about whether it's `indegree[v]++` or `indegree[u]++` when adding edge `u→v`, you don't have it yet. (Answer: the edge points INTO v, so `indegree[v]++`.)

2. **Cycle detection as a FREE byproduct of topological sort, not a separate algorithm.** You never need a separate "does this graph have a cycle" pass before running Kahn's. Just run Kahn's — if it terminates with fewer than V nodes processed, a cycle exists among the unprocessed nodes. This single fact is the answer to Course Schedule (LC 207) and dozens of variants.

3. **Building the graph is often the actual problem.** In Alien Dictionary, Sort Items by Groups, and Build a Matrix With Conditions, the topological sort itself is a five-line function you've already memorized. The hard part — and where interviews actually differentiate candidates — is correctly deriving the edges from the problem's constraints, including the edge cases (a word that is a prefix of the next word contributes NO information; a longer word appearing before its own prefix is invalid input).

**The interview data:**
- Course Schedule (LC 207) and Course Schedule II (LC 210) are top-15 most-asked problems at Google, Amazon, and Meta — often the FIRST graph question in a loop.
- Alien Dictionary (LC 269) is a well-known "hard" filter question — the difficulty is 90% edge-case handling, 10% algorithm.
- Sort Items by Groups Respecting Dependencies (LC 1203) is a favorite "have you actually mastered this pattern" contest/onsite problem — a naive single-level topo sort fails it.

---

## SECTION 2 — THE TOPOLOGICAL SORT TAXONOMY

### Type 1: Kahn's Algorithm (BFS + In-Degree)
Mechanism: seed a queue with all in-degree-0 nodes, repeatedly pop, emit, and decrement neighbors' in-degree, pushing any that reach 0.
Direction: naturally produces a valid left-to-right order (no reversal needed).
Problems: Course Schedule, Course Schedule II, Parallel Courses, Minimum Height Trees (undirected variant).

### Type 2: DFS-Based Topological Sort (Reverse Finish Order)
Mechanism: run DFS from every unvisited node; when a node finishes (all descendants explored), push it onto a stack (or prepend to a list). The finish order, reversed, is a valid topological order.
Direction: requires an explicit **reverse** at the end — this is the #1 source of off-by-one bugs in this variant.
Problems: any topo sort problem, but especially useful when you also need finish-time information (e.g., Strongly Connected Components / Kosaraju's algorithm builds on this directly).

### Type 3: Cycle Detection in Directed Graphs
Mechanism A (Kahn's): if `processedCount < V` when the queue empties, the unprocessed nodes form (or are part of) a cycle.
Mechanism B (DFS 3-color): WHITE (unvisited) → GRAY (on current recursion stack) → BLACK (fully finished). An edge to a GRAY node is a **back edge** — a cycle. An edge to a BLACK node is safe (already fully explored, no cycle through it).
Problems: Course Schedule, any "is this a valid DAG" precheck.

### Type 4: All Topological Orderings (Backtracking)
Mechanism: standard backtracking over "choose any current 0-in-degree node, recurse, undo." Exponential in the worst case (a graph with no edges has V! orderings).
Problems: rare as a standalone LeetCode problem, but a common whiteboard follow-up ("now list ALL valid orderings").

### Type 5: Longest/Shortest Path in a DAG (Topo Sort + DP)
Mechanism: process nodes in topological order; because every predecessor of a node is guaranteed to be processed first, a single forward DP pass computes shortest/longest/count-of-paths correctly in O(V+E) — no relaxation loop, no Dijkstra needed.
Problems: Parallel Courses (longest path = minimum semesters), Longest Increasing Path in a Matrix (implicit DAG), Course Schedule IV (reachability = a boolean DP over topo order).

### Type 6: Building the Graph From Constraints
Mechanism: the edges are not given directly — you derive them from pairwise comparisons, adjacency rules, or grouping constraints. This is where most of the "hard" rating in topo-sort problems actually lives.
Problems: Alien Dictionary (edges from first-differing-character of adjacent words), Build a Matrix With Conditions (edges from row/column order constraints), Sequence Reconstruction (edges from adjacent pairs in each sub-sequence).

### Type 7: Two-Level / Grouped Topological Sort
Mechanism: items belong to groups; you must respect both item-level dependencies AND group-level dependencies (derived from cross-group item dependencies). Solve with two independent topo sorts — one on items, one on groups — then merge.
Problems: Sort Items by Groups Respecting Dependencies (LC 1203).

### Type 8: Implicit-Graph / Grid Topological Sort
Mechanism: the DAG is never materialized as an adjacency list — edges are "this cell to that adjacent cell if the value is strictly greater." Solve either with memoized DFS (treating it as DAG DP) or with an explicit Kahn's "onion-peeling" pass from local minima.
Problems: Longest Increasing Path in a Matrix.

**Quick backtracking template for Type 4 (all orderings) — rarely coded in full, but know the shape:**

```cpp
// Enumerate ALL valid topological orderings (exponential — use only on small graphs)
void allTopoOrders(int n, vector<vector<int>>& adj, vector<int>& indegree,
                    vector<int>& path, vector<bool>& visited,
                    vector<vector<int>>& results) {
    bool extended = false;
    for (int v = 0; v < n; v++) {
        if (!visited[v] && indegree[v] == 0) {
            // CHOOSE v
            visited[v] = true;
            path.push_back(v);
            for (int nb : adj[v]) indegree[nb]--;

            allTopoOrders(n, adj, indegree, path, visited, results);

            // UN-CHOOSE v (backtrack)
            visited[v] = false;
            path.pop_back();
            for (int nb : adj[v]) indegree[nb]++;
            extended = true;
        }
    }
    // A leaf of the recursion where no node could be chosen AND
    // we used all n nodes = one complete, valid ordering.
    if (!extended && (int)path.size() == n) {
        results.push_back(path);
    }
}
```

---

## SECTION 3 — TEMPLATE 1: KAHN'S ALGORITHM (BFS + IN-DEGREE)

```cpp
// ─────────────────────────────────────────────────────────────────
// Generic Kahn's Algorithm — the algorithm you must know cold.
// Given n nodes (0..n-1) and a directed adjacency list, return
// a valid topological order, or an empty vector if a cycle exists.
//
// State:      indegree[v] = number of edges pointing INTO v
// Mechanism:  a node with indegree 0 has no unmet dependencies —
//             it is always safe to process next.
// Invariant:  every node pushed to the queue has ALL its
//             predecessors already processed.
// ─────────────────────────────────────────────────────────────────

vector<int> kahnTopoSort(int n, vector<vector<int>>& adj) {
    vector<int> indegree(n, 0);
    for (int u = 0; u < n; u++)
        for (int v : adj[u])
            indegree[v]++;                  // edge u->v points INTO v

    queue<int> q;
    for (int i = 0; i < n; i++)
        if (indegree[i] == 0)
            q.push(i);                      // seed with all "free" nodes

    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u]) {
            if (--indegree[v] == 0)         // u is done -> v loses a dependency
                q.push(v);
        }
    }

    // THE CRITICAL CHECK: if we didn't process every node, a cycle
    // exists among the leftover nodes (their in-degree never hit 0).
    if ((int)order.size() != n) return {};
    return order;
}
```

**FULL TRACE.** Graph: 6 nodes (0..5), directed edges:
`5→0, 5→2, 4→0, 4→1, 2→3, 3→1`

```
Build indegree:
  edge 5->0: indegree[0]++  -> indegree[0]=1
  edge 5->2: indegree[2]++  -> indegree[2]=1
  edge 4->0: indegree[0]++  -> indegree[0]=2
  edge 4->1: indegree[1]++  -> indegree[1]=1
  edge 2->3: indegree[3]++  -> indegree[3]=1
  edge 3->1: indegree[1]++  -> indegree[1]=2

Final indegree: [2, 2, 1, 1, 0, 0]   (index = node id)

Seed queue with indegree==0: nodes 4, 5
queue = [4, 5]

Pop 4: order=[4]
  neighbors of 4: 0, 1
  indegree[0]: 2->1 (not 0, don't push)
  indegree[1]: 2->1 (not 0, don't push)
  queue = [5]

Pop 5: order=[4, 5]
  neighbors of 5: 0, 2
  indegree[0]: 1->0  -> push 0
  indegree[2]: 1->0  -> push 2
  queue = [0, 2]

Pop 0: order=[4, 5, 0]
  neighbors of 0: (none)
  queue = [2]

Pop 2: order=[4, 5, 0, 2]
  neighbors of 2: 3
  indegree[3]: 1->0 -> push 3
  queue = [3]

Pop 3: order=[4, 5, 0, 2, 3]
  neighbors of 3: 1
  indegree[1]: 1->0 -> push 1
  queue = [1]

Pop 1: order=[4, 5, 0, 2, 3, 1]
  neighbors of 1: (none)
  queue = []

order.size() == 6 == n  -> valid.
FINAL ORDER: 4, 5, 0, 2, 3, 1
```

**Verify correctness:** every edge must go left-to-right in the final order.
`5→0`: 5 is at index 1, 0 is at index 2. ✓
`5→2`: 5 at index 1, 2 at index 3. ✓
`4→0`: 4 at index 0, 0 at index 2. ✓
`4→1`: 4 at index 0, 1 at index 5. ✓
`2→3`: 2 at index 3, 3 at index 4. ✓
`3→1`: 3 at index 4, 1 at index 5. ✓
All six edges respected — valid topological order.

---

## SECTION 4 — TEMPLATE 2: DFS-BASED TOPO SORT WITH CYCLE DETECTION

```cpp
// ─────────────────────────────────────────────────────────────────
// DFS-based topological sort using 3-color marking.
//
// WHITE = unvisited
// GRAY  = currently on the recursion stack (an ancestor of the
//         node we're exploring right now — "in progress")
// BLACK = fully finished (this node and everything reachable
//         from it has been completely explored — safe forever)
//
// Cycle rule: if DFS from node u reaches a GRAY node, that GRAY
// node is an ANCESTOR of u in the current DFS tree -> back edge
// -> cycle. Reaching a BLACK node is fine (cross/forward edge,
// no cycle possible through an already-finished subtree).
//
// Topo order: append each node to `order` when it turns BLACK
// (post-order). Reverse `order` at the very end.
// ─────────────────────────────────────────────────────────────────

enum Color { WHITE, GRAY, BLACK };

bool dfsVisit(int u, vector<vector<int>>& adj, vector<int>& color,
              vector<int>& order) {
    color[u] = GRAY;
    for (int v : adj[u]) {
        if (color[v] == GRAY) return false;              // back edge -> cycle
        if (color[v] == WHITE && !dfsVisit(v, adj, color, order))
            return false;                                  // cycle found deeper
    }
    color[u] = BLACK;
    order.push_back(u);                                    // post-order emission
    return true;
}

vector<int> dfsTopoSort(int n, vector<vector<int>>& adj) {
    vector<int> color(n, WHITE), order;
    for (int i = 0; i < n; i++) {
        if (color[i] == WHITE) {
            if (!dfsVisit(i, adj, color, order)) return {}; // cycle -> no valid order
        }
    }
    reverse(order.begin(), order.end());                    // CRITICAL STEP
    return order;
}
```

**FULL TRACE — same graph as Template 1:** edges `5→0, 5→2, 4→0, 4→1, 2→3, 3→1`, adjacency:
`adj[0]=[], adj[1]=[], adj[2]=[3], adj[3]=[1], adj[4]=[0,1], adj[5]=[0,2]`

```
color = [W,W,W,W,W,W], order = []

i=0: color[0]==WHITE -> dfsVisit(0)
  color[0]=GRAY
  adj[0] is empty -> no neighbors to check
  color[0]=BLACK, order=[0]
  return true

i=1: color[1]==WHITE -> dfsVisit(1)
  color[1]=GRAY
  adj[1] is empty
  color[1]=BLACK, order=[0,1]
  return true

i=2: color[2]==WHITE -> dfsVisit(2)
  color[2]=GRAY
  neighbor 3: color[3]==WHITE -> dfsVisit(3)
      color[3]=GRAY
      neighbor 1: color[1]==BLACK -> safe, skip (already finished)
      color[3]=BLACK, order=[0,1,3]
      return true
  color[2]=BLACK, order=[0,1,3,2]
  return true

i=3: color[3]==BLACK -> skip (already visited)

i=4: color[4]==WHITE -> dfsVisit(4)
  color[4]=GRAY
  neighbor 0: color[0]==BLACK -> safe, skip
  neighbor 1: color[1]==BLACK -> safe, skip
  color[4]=BLACK, order=[0,1,3,2,4]
  return true

i=5: color[5]==WHITE -> dfsVisit(5)
  color[5]=GRAY
  neighbor 0: color[0]==BLACK -> safe, skip
  neighbor 2: color[2]==BLACK -> safe, skip
  color[5]=BLACK, order=[0,1,3,2,4,5]
  return true

order (pre-reverse) = [0, 1, 3, 2, 4, 5]
REVERSE -> [5, 4, 2, 3, 1, 0]
FINAL ORDER: 5, 4, 2, 3, 1, 0
```

**Verify:** `5→0` (idx0→idx5 ✓), `5→2` (idx0→idx2 ✓), `4→0` (idx1→idx5 ✓), `4→1` (idx1→idx4 ✓), `2→3` (idx2→idx3 ✓), `3→1` (idx3→idx4 ✓). All valid.

**CYCLE TRACE.** Add one more edge to the same graph: `1→4` (creating a cycle `4→1→4`).

```
i=0, i=1: process and finish normally as above (color[0]=BLACK, color[1]=BLACK... 
  wait — order matters. Restart with the NEW edge set from a fresh graph:
  adj[4] = [0, 1], adj[1] = [4]  (the new back-reference)

i=4: color[4]==WHITE -> dfsVisit(4)
  color[4]=GRAY
  neighbor 0: color[0]==WHITE -> dfsVisit(0) -> finishes, color[0]=BLACK, order=[0]
  neighbor 1: color[1]==WHITE -> dfsVisit(1)
      color[1]=GRAY
      neighbor 4: color[4]==GRAY  <-- GRAY node hit!
      return false   (CYCLE DETECTED: 4 -> 1 -> 4)
  dfsVisit(4) propagates false
  return false

dfsTopoSort returns {} immediately.
```

Notice exactly WHY this works: node 4 is GRAY (on the current DFS stack, meaning we are still "inside" processing it) when node 1's exploration tries to reach it again. That is the definition of a cycle: a path that returns to an ancestor still being processed.

---

## SECTION 5 — TEMPLATE 3: COURSE SCHEDULE (LC 207)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 207 — Course Schedule
// prerequisites[i] = [a, b] means: to take course a, you must
// first take course b.  So the DIRECTED EDGE is b -> a
// (b must come before a in the schedule).
//
// Question: can all numCourses be finished? (i.e., is this a DAG?)
// Answer: run Kahn's. If we can process all numCourses nodes,
// there's no cycle -> return true. Otherwise -> false.
// ─────────────────────────────────────────────────────────────────

bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> adj(numCourses);
    vector<int> indegree(numCourses, 0);

    for (auto& p : prerequisites) {
        int a = p[0], b = p[1];   // a depends on b: edge b -> a
        adj[b].push_back(a);
        indegree[a]++;
    }

    queue<int> q;
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) q.push(i);

    int processed = 0;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        processed++;
        for (int v : adj[u])
            if (--indegree[v] == 0) q.push(v);
    }

    return processed == numCourses;  // false -> cycle exists
}
```

**EDGE-DIRECTION SANITY CHECK (memorize this once, never get it wrong again):**
`prerequisites[i] = [a, b]` reads as **"a requires b"**, i.e., "b before a." The arrow of dependency points from the thing you need FIRST to the thing that needs it: `b → a`. If you instead wrote `adj[a].push_back(b)`, you've built the graph backward — Kahn's would seed the queue with courses that have NO prerequisites-of-them (wrong semantics) instead of courses that ARE NO ONE's prerequisite.

**TRACE:** `numCourses=4`, `prerequisites=[[1,0],[2,0],[3,1],[3,2]]` (course 3 needs 1 and 2; courses 1 and 2 both need 0).

```
Edges: 0->1, 0->2, 1->3, 2->3
indegree: [0, 1, 1, 2]

queue seeded: [0]
pop 0: processed=1. neighbors 1,2: indegree[1]=0->push1, indegree[2]=0->push2.
queue=[1,2]
pop 1: processed=2. neighbor 3: indegree[3]=1 (not 0 yet).
pop 2: processed=3. neighbor 3: indegree[3]=0 -> push3.
queue=[3]
pop 3: processed=4. no neighbors.

processed==4==numCourses -> return true.
```

---

## SECTION 6 — TEMPLATE 4: COURSE SCHEDULE II (LC 210)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 210 — Course Schedule II
// Same setup as LC 207, but return an ACTUAL valid order instead
// of just true/false. Return {} if impossible (cycle).
// This is literally kahnTopoSort() with the edge direction fixed
// for the "a requires b" semantics.
// ─────────────────────────────────────────────────────────────────

vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> adj(numCourses);
    vector<int> indegree(numCourses, 0);

    for (auto& p : prerequisites) {
        int a = p[0], b = p[1];
        adj[b].push_back(a);   // b -> a  (b before a)
        indegree[a]++;
    }

    queue<int> q;
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) q.push(i);

    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u])
            if (--indegree[v] == 0) q.push(v);
    }

    if ((int)order.size() != numCourses) return {};  // cycle
    return order;
}
```

**TRACE:** Same input as Template 3. Kahn's produces `order = [0, 1, 2, 3]`. Check: `0` before `1` ✓, `0` before `2` ✓, `1` before `3` ✓, `2` before `3` ✓. Valid.

**Cycle case:** `numCourses=2`, `prerequisites=[[1,0],[0,1]]`. Edges: `0→1, 1→0`. `indegree=[1,1]`. Queue seeded empty (no node has indegree 0) → loop never runs → `order.size()==0 != 2` → return `{}`.

---

## SECTION 7 — TEMPLATE 5: ALIEN DICTIONARY (LC 269)

This is the pattern's signature "build the graph yourself" problem, and the hardest part is the edge cases, not the topo sort.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 269 — Alien Dictionary
// Given a list of words sorted according to an unknown alphabet's
// order, determine that order. Return "" if no valid order exists.
//
// GRAPH CONSTRUCTION:
// Compare each pair of ADJACENT words (words[i], words[i+1]).
// Find the FIRST position where they differ: that gives one edge
// (earlier char -> later char) — and only ONE edge per adjacent
// pair; later characters give no information (unlike LCS-style
// full comparison).
//
// CRITICAL EDGE CASE:
// If NO differing character is found within min(len1, len2)
// characters (i.e., one word is a strict PREFIX of the other),
// the pair is valid ONLY IF the shorter word comes first.
// If the LONGER word comes first and the shorter is its prefix
// (e.g., "abc" before "ab"), the input is invalid — a
// correctly-sorted dictionary can never have a longer word
// immediately followed by its own prefix. Return "".
// ─────────────────────────────────────────────────────────────────

string alienOrder(vector<string>& words) {
    unordered_map<char, unordered_set<char>> adj;
    unordered_map<char, int> indegree;

    // Every character that appears anywhere gets an indegree entry
    // (even if it turns out to be 0) so it's included in the output.
    for (string& w : words)
        for (char c : w)
            indegree[c] = 0;

    for (int i = 0; i + 1 < (int)words.size(); i++) {
        string& w1 = words[i];
        string& w2 = words[i + 1];
        int minLen = min(w1.size(), w2.size());
        bool foundDifference = false;

        for (int j = 0; j < minLen; j++) {
            if (w1[j] != w2[j]) {
                if (!adj[w1[j]].count(w2[j])) {   // avoid duplicate edges
                    adj[w1[j]].insert(w2[j]);
                    indegree[w2[j]]++;
                }
                foundDifference = true;
                break;   // only the FIRST difference matters
            }
        }

        // EDGE CASE: longer word appears before its own prefix -> invalid
        if (!foundDifference && w1.size() > w2.size()) return "";
    }

    queue<char> q;
    for (auto& [ch, deg] : indegree)
        if (deg == 0) q.push(ch);

    string order;
    while (!q.empty()) {
        char c = q.front(); q.pop();
        order += c;
        for (char nxt : adj[c])
            if (--indegree[nxt] == 0) q.push(nxt);
    }

    return order.size() == indegree.size() ? order : "";  // size mismatch = cycle
}
```

**FULL TRACE:** `words = ["wrt", "wrf", "er", "ett", "rftt"]`

```
Unique characters seen: w, r, t, f, e  (indegree all start at 0)

Pair ("wrt","wrf"): compare w=w, r=r, t vs f -> differ at index 2.
  edge t -> f.  indegree[f]=1

Pair ("wrf","er"): compare w vs e -> differ at index 0.
  edge w -> e.  indegree[e]=1

Pair ("er","ett"): compare e=e, r vs t -> differ at index 1.
  edge r -> t.  indegree[t]=1

Pair ("ett","rftt"): compare e vs r -> differ at index 0.
  edge e -> r.  indegree[r]=1

Final graph:
  adj: t->{f}, w->{e}, r->{t}, e->{r}
  indegree: w=0, r=1, t=1, f=1, e=1

Seed queue: only w has indegree 0.  queue=[w]

pop w: order="w". neighbor e: indegree[e] 1->0 -> push e. queue=[e]
pop e: order="we". neighbor r: indegree[r] 1->0 -> push r. queue=[r]
pop r: order="wer". neighbor t: indegree[t] 1->0 -> push t. queue=[t]
pop t: order="wert". neighbor f: indegree[f] 1->0 -> push f. queue=[f]
pop f: order="wertf". no neighbors. queue=[]

order.size()=5 == indegree.size()=5  -> valid.
FINAL ANSWER: "wertf"
```

**INVALID-PREFIX TRACE:** `words = ["abc", "ab"]`

```
Pair ("abc","ab"): minLen=2. compare a=a, b=b -> no difference found
  within the first 2 characters. foundDifference = false.
  w1.size()=3 > w2.size()=2  -> LONGER word came first, and the
  shorter word is exactly its prefix.
  This can NEVER happen in a validly-sorted dictionary (if "ab"
  sorts before "abc" in any alphabet, being a strict prefix always
  sorts first). INVALID INPUT.

return ""
```

Contrast with the VALID case `words = ["ab", "abc"]` (shorter-then-longer, the normal case): `foundDifference=false`, but `w1.size()=2` is NOT greater than `w2.size()=3`, so no edge is added and no error is raised — this pair simply contributes no ordering information, which is correct.

---

## SECTION 8 — TEMPLATE 6: MINIMUM HEIGHT TREES (LC 310)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 310 — Minimum Height Trees
// Given an UNDIRECTED tree (n nodes, n-1 edges), find all node(s)
// that, if chosen as root, minimize the tree's height.
// There are at most 2 such nodes (the "center(s)" of the tree).
//
// KEY INSIGHT: this is topological sort's "reverse" idea applied
// to an undirected graph — instead of peeling nodes with
// indegree 0, we ITERATIVELY PEEL LEAVES (degree 1 nodes) layer
// by layer, like Kahn's algorithm running from the outside in.
// The last 1 or 2 nodes left standing are the tree's center(s) —
// exactly the roots that minimize height.
// ─────────────────────────────────────────────────────────────────

vector<int> findMinHeightTrees(int n, vector<vector<int>>& edges) {
    if (n == 1) return {0};   // single node: itself is the only MHT root

    vector<unordered_set<int>> adj(n);
    vector<int> degree(n, 0);
    for (auto& e : edges) {
        adj[e[0]].insert(e[1]);
        adj[e[1]].insert(e[0]);
        degree[e[0]]++;
        degree[e[1]]++;
    }

    queue<int> leaves;
    for (int i = 0; i < n; i++)
        if (degree[i] == 1) leaves.push(i);

    int remaining = n;
    while (remaining > 2) {
        int leafCount = leaves.size();
        remaining -= leafCount;
        for (int i = 0; i < leafCount; i++) {
            int leaf = leaves.front(); leaves.pop();
            for (int nb : adj[leaf]) {
                if (--degree[nb] == 1) leaves.push(nb);
            }
        }
    }

    vector<int> result;
    while (!leaves.empty()) {
        result.push_back(leaves.front());
        leaves.pop();
    }
    return result;
}
```

**FULL TRACE:** `n=6`, `edges=[[3,0],[3,1],[3,2],[3,4],[5,4]]`

```
adj: 0-3, 1-3, 2-3, 3-4, 4-5
degree: [1, 1, 1, 4, 2, 1]   (index = node)

Seed leaves (degree==1): 0, 1, 2, 5
leaves queue = [0, 1, 2, 5],  remaining = 6

remaining(6) > 2, one trimming round:
  leafCount = 4, remaining = 6 - 4 = 2

  pop leaf 0: neighbor 3. degree[3]: 4->3 (not 1, no push)
  pop leaf 1: neighbor 3. degree[3]: 3->2 (not 1, no push)
  pop leaf 2: neighbor 3. degree[3]: 2->1 -> push 3
  pop leaf 5: neighbor 4. degree[4]: 2->1 -> push 4

  leaves queue = [3, 4]

remaining(2) is not > 2 -> loop ends.

FINAL RESULT: [3, 4]
```

**Why leaf-trimming finds the center:** repeatedly removing all current leaves is equivalent to shrinking the tree symmetrically from every direction at once — like burning a rope from both ends simultaneously. The point(s) where the burning meets is exactly the middle of the longest path (the tree's diameter), which is provably the height-minimizing root. If the diameter has odd length, one node remains (unique center); if even, two adjacent nodes remain (both are valid centers, height-equivalent).

---

## SECTION 9 — TEMPLATE 7: COURSE SCHEDULE IV (LC 1462)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1462 — Course Schedule IV
// Given prerequisites (directed edges) and queries [u, v], answer
// for each query: "is u a prerequisite of v?" — i.e., is there a
// DIRECTED PATH from u to v (not necessarily a direct edge)?
//
// KEY INSIGHT: process nodes in TOPOLOGICAL ORDER (via Kahn's).
// When we pop node u and relax edge u->v, u's own reachability
// set (which prerequisite-of relationships point INTO u) is
// ALREADY COMPLETE, because Kahn's only pops u after every
// predecessor of u has already been processed. So we can safely
// propagate: everything that reaches u also reaches v.
// ─────────────────────────────────────────────────────────────────

vector<bool> checkIfPrerequisite(int n, vector<vector<int>>& prerequisites,
                                  vector<vector<int>>& queries) {
    vector<vector<int>> adj(n);
    vector<int> indegree(n, 0);
    for (auto& p : prerequisites) {
        adj[p[0]].push_back(p[1]);
        indegree[p[1]]++;
    }

    vector<vector<bool>> reach(n, vector<bool>(n, false));

    queue<int> q;
    for (int i = 0; i < n; i++)
        if (indegree[i] == 0) q.push(i);

    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) {
            reach[u][v] = true;
            // Propagate: any ancestor a of u also reaches v.
            for (int a = 0; a < n; a++)
                if (reach[a][u]) reach[a][v] = true;
            if (--indegree[v] == 0) q.push(v);
        }
    }

    vector<bool> ans;
    for (auto& qr : queries)
        ans.push_back(reach[qr[0]][qr[1]]);
    return ans;
}
```

**FULL TRACE:** `n=3`, `prerequisites=[[1,2],[1,0],[2,0]]`, `queries=[[1,0],[1,2]]`

```
adj: 1->[2,0], 2->[0]
indegree: [2, 0, 1]

queue seeded: [1]   (only node 1 has indegree 0)

pop 1:
  edge 1->2: reach[1][2]=true.
    propagate: is any reach[a][1] true? no ancestors reach 1 yet -> nothing more.
    indegree[2]: 1->0 -> push 2
  edge 1->0: reach[1][0]=true.
    propagate: no ancestors reach 1 -> nothing more.
    indegree[0]: 2->1

queue = [2]

pop 2:
  edge 2->0: reach[2][0]=true.
    propagate: check all a: reach[1][2]==true -> reach[1][0]=true (already true, no-op).
    indegree[0]: 1->0 -> push 0

queue = [0]

pop 0: adj[0] empty. nothing to do.

Final reach table (only true entries shown):
  reach[1][2]=true, reach[1][0]=true, reach[2][0]=true

Query [1,0]: reach[1][0] = true
Query [1,2]: reach[1][2] = true

FINAL ANSWER: [true, true]
```

---

## SECTION 10 — TEMPLATE 8: LONGEST INCREASING PATH IN A MATRIX (LC 329)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 329 — Longest Increasing Path in a Matrix
// Find the length of the longest strictly-increasing path in an
// m x n matrix, moving only to up/down/left/right neighbors.
//
// WHY THIS IS A TOPOLOGICAL SORT PROBLEM (even though we solve it
// with memoized DFS): treat every cell as a node. Draw a directed
// edge from cell A to adjacent cell B if matrix[B] > matrix[A].
// Because values only ever INCREASE along an edge, no cycle is
// possible — this is an IMPLICIT DAG. The "longest path in a DAG"
// problem is exactly Taxonomy Type 5: process nodes in an order
// where all predecessors are resolved first (naturally achieved
// here via memoized DFS instead of an explicit Kahn's pass, since
// building an explicit adjacency list for every cell is wasteful).
// ─────────────────────────────────────────────────────────────────

int longestIncreasingPath(vector<vector<int>>& matrix) {
    int m = matrix.size(), n = matrix[0].size();
    vector<vector<int>> memo(m, vector<int>(n, 0));  // 0 = not computed yet
    int dr[] = {0, 0, 1, -1}, dc[] = {1, -1, 0, 0};

    function<int(int, int)> dfs = [&](int r, int c) -> int {
        if (memo[r][c] != 0) return memo[r][c];
        int best = 1;   // path of just this cell
        for (int d = 0; d < 4; d++) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n &&
                matrix[nr][nc] > matrix[r][c]) {
                best = max(best, 1 + dfs(nr, nc));
            }
        }
        return memo[r][c] = best;
    };

    int ans = 0;
    for (int r = 0; r < m; r++)
        for (int c = 0; c < n; c++)
            ans = max(ans, dfs(r, c));
    return ans;
}
```

**FULL TRACE:** `matrix = [[9,9,4],[6,6,8],[2,1,1]]` (expected answer: 4)

```
dfs(2,1) [value=1]:  neighbors -> (1,1)=6>1: recurse. (2,0)=2>1: recurse. (2,2)=1: not >, skip.

  dfs(1,1) [value=6]: neighbors -> (0,1)=9>6: recurse. (1,2)=8>6: recurse.
                                    (1,0)=6: not >, skip. (2,1)=1: not >, skip.

    dfs(0,1) [value=9]: neighbors -> (0,0)=9 not>, (0,2)=4 not>, (1,1)=6 not>.
                          best=1.  memo[0][1]=1

    dfs(1,2) [value=8]: neighbors -> (0,2)=4 not>, (1,1)=6 not>, (2,2)=1 not>.
                          best=1.  memo[1][2]=1

    dfs(1,1) = max(1 + memo[0][1], 1 + memo[1][2]) = max(1+1, 1+1) = 2
    memo[1][1] = 2

  dfs(2,0) [value=2]: neighbors -> (1,0)=6>2: recurse. (2,1)=1 not>, skip.

    dfs(1,0) [value=6]: neighbors -> (0,0)=9>6: recurse.
                                      (1,1)=6 not>, (2,0)=2 not>.

      dfs(0,0) [value=9]: neighbors -> (0,1)=9 not>(equal doesn't count),
                            (1,0)=6 not>. best=1. memo[0][0]=1

      dfs(1,0) = 1 + memo[0][0] = 1 + 1 = 2.  memo[1][0]=2

    dfs(2,0) = 1 + memo[1][0] = 1 + 2 = 3.  memo[2][0]=3

  dfs(2,1) = 1 + max(memo[1][1]=2, memo[2][0]=3) = 1 + 3 = 4
  memo[2][1] = 4

Outer loop scans every cell; the maximum memo value found is 4
(achieved at cell (2,1)).

FINAL ANSWER: 4
Path: matrix[2][1]=1 -> matrix[2][0]=2 -> matrix[1][0]=6 -> matrix[0][0]=9
      (1 -> 2 -> 6 -> 9, length 4) ✓
```

**Alternative topo-sort formulation (worth knowing as a follow-up answer):** build explicit edges from every cell to each strictly-greater neighbor, compute in-degree = "number of strictly-smaller neighbors," and run Kahn's, peeling local minima layer by layer ("onion peeling"). The number of peeling rounds until the queue empties equals the answer. This is O(mn) exactly like the memoized DFS, but demonstrates the DAG-DP connection explicitly if an interviewer asks "can you solve this without recursion?"

---

## SECTION 11 — TEMPLATE 9: PARALLEL COURSES (LC 1136)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1136 — Parallel Courses
// n courses (1..n), relations[i]=[a,b] means a must be completed
// before b. In each semester, take any number of courses whose
// prerequisites are all satisfied. Return the MINIMUM number of
// semesters to finish all courses, or -1 if impossible (cycle).
//
// KEY INSIGHT: this is Kahn's algorithm processed LEVEL BY LEVEL
// (like BFS level-order traversal). Each "level" (all nodes with
// indegree 0 at the same time) = one semester. The number of
// levels = the LONGEST PATH in the DAG (Taxonomy Type 5) + 1.
// ─────────────────────────────────────────────────────────────────

int minimumSemesters(int n, vector<vector<int>>& relations) {
    vector<vector<int>> adj(n + 1);
    vector<int> indegree(n + 1, 0);
    for (auto& r : relations) {
        adj[r[0]].push_back(r[1]);
        indegree[r[1]]++;
    }

    queue<int> q;
    for (int i = 1; i <= n; i++)
        if (indegree[i] == 0) q.push(i);

    int semesters = 0, studied = 0;
    while (!q.empty()) {
        int levelSize = q.size();
        semesters++;                       // one semester = one BFS level
        for (int i = 0; i < levelSize; i++) {
            int u = q.front(); q.pop();
            studied++;
            for (int v : adj[u])
                if (--indegree[v] == 0) q.push(v);
        }
    }

    return studied == n ? semesters : -1;
}
```

**FULL TRACE:** `n=4`, `relations=[[1,2],[2,3],[3,4]]` (a strict chain)

```
adj: 1->[2], 2->[3], 3->[4]
indegree: [_, 0, 1, 1, 1]   (index 0 unused)

queue seeded: [1]

Semester 1: levelSize=1
  pop 1: studied=1. neighbor 2: indegree[2] 1->0 -> push 2.
  queue=[2]

Semester 2: levelSize=1
  pop 2: studied=2. neighbor 3: indegree[3] 1->0 -> push 3.
  queue=[3]

Semester 3: levelSize=1
  pop 3: studied=3. neighbor 4: indegree[4] 1->0 -> push 4.
  queue=[4]

Semester 4: levelSize=1
  pop 4: studied=4. no neighbors.
  queue=[]

studied(4) == n(4) -> return semesters = 4
```

A WIDE example clarifies why levels matter: if instead `relations=[[1,2],[1,3],[1,4]]` (course 1 unlocks 2, 3, 4 simultaneously), semester 1 processes {1}, semester 2 processes {2,3,4} all at once — answer is 2 semesters, not 4, because level-batching (not one-node-at-a-time) is exactly what "parallel" means here.

---

## SECTION 12 — TEMPLATE 10: SORT ITEMS BY GROUPS RESPECTING DEPENDENCIES (LC 1203)

The hardest template in this pattern — a topological sort at TWO levels simultaneously.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1203 — Sort Items by Groups Respecting Dependencies
// n items, m groups. group[i] = the group of item i, or -1 if
// ungrouped. beforeItems[i] = list of items that must come before
// item i. Return a valid item ordering that ALSO keeps items of
// the same group contiguous... actually the real constraint is
// items must be ordered validly overall, AND if two items are in
// different groups with a dependency between them, their GROUPS
// must also respect that order.
//
// STRATEGY (two-level topo sort):
// 1. Give every ungrouped item (-1) its own unique singleton group.
// 2. Build TWO graphs simultaneously: an item graph (full detail)
//    and a group graph (coarser — only edges crossing group
//    boundaries).
// 3. Topologically sort items -> gives a valid ordering respecting
//    ALL beforeItems constraints (including cross-group ones).
// 4. Topologically sort groups -> gives a valid group order.
// 5. Bucket items into their groups, PRESERVING the relative order
//    from step 3's item-topo-sort within each bucket.
// 6. Concatenate buckets in the group order from step 4.
// ─────────────────────────────────────────────────────────────────

vector<int> topoSortGeneric(int n, vector<vector<int>>& adj, vector<int> indegree) {
    queue<int> q;
    for (int i = 0; i < n; i++)
        if (indegree[i] == 0) q.push(i);

    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u])
            if (--indegree[v] == 0) q.push(v);
    }
    return order;   // caller checks order.size() against n
}

vector<int> sortItems(int n, int m, vector<int>& group,
                       vector<vector<int>>& beforeItems) {
    // Step 1: give ungrouped items unique new group ids
    int groupCount = m;
    for (int i = 0; i < n; i++)
        if (group[i] == -1) group[i] = groupCount++;

    // Step 2: build item graph and group graph together
    vector<vector<int>> itemAdj(n);
    vector<int> itemIndegree(n, 0);
    vector<vector<int>> groupAdj(groupCount);
    vector<int> groupIndegree(groupCount, 0);

    for (int i = 0; i < n; i++) {
        for (int prev : beforeItems[i]) {
            itemAdj[prev].push_back(i);
            itemIndegree[i]++;
            if (group[prev] != group[i]) {          // crosses a group boundary
                groupAdj[group[prev]].push_back(group[i]);
                groupIndegree[group[i]]++;
            }
        }
    }

    // Step 3 & 4: topo sort both levels
    vector<int> itemOrder = topoSortGeneric(n, itemAdj, itemIndegree);
    if ((int)itemOrder.size() != n) return {};              // item-level cycle

    vector<int> groupOrder = topoSortGeneric(groupCount, groupAdj, groupIndegree);
    if ((int)groupOrder.size() != groupCount) return {};    // group-level cycle

    // Step 5: bucket items by group, preserving itemOrder's relative order
    vector<vector<int>> itemsInGroup(groupCount);
    for (int item : itemOrder)
        itemsInGroup[group[item]].push_back(item);

    // Step 6: concatenate in group-topo-order
    vector<int> result;
    for (int g : groupOrder)
        for (int item : itemsInGroup[g])
            result.push_back(item);

    return result;
}
```

**FULL TRACE:** `n=2, m=2, group=[0,1], beforeItems=[[1],[]]`
(item 0 is in group 0, item 1 is in group 1; item 0 depends on item 1)

```
Step 1: no ungrouped items (-1). groupCount stays 2.

Step 2: for item i=0, beforeItems[0]=[1]:
  itemAdj[1].push_back(0).  itemIndegree[0]=1
  group[1]=1, group[0]=0 -> different groups:
    groupAdj[1].push_back(0).  groupIndegree[0]=1
  (item i=1 has beforeItems[1]=[], nothing to add)

itemAdj: [ [], [0] ]        itemIndegree: [1, 0]
groupAdj: [ [], [0] ]       groupIndegree: [1, 0]

Step 3: topoSortGeneric on items:
  seed queue with indegree 0: item 1. queue=[1]
  pop 1: order=[1]. neighbor 0: itemIndegree[0] 1->0 -> push 0.
  pop 0: order=[1,0]. no neighbors.
  itemOrder = [1, 0].  size 2 == n -> valid.

Step 4: topoSortGeneric on groups:
  seed queue with indegree 0: group 1. queue=[1]
  pop 1: order=[1]. neighbor 0: groupIndegree[0] 1->0 -> push 0.
  pop 0: order=[1,0]. no neighbors.
  groupOrder = [1, 0].  size 2 == groupCount -> valid.

Step 5: bucket by group, following itemOrder=[1,0]:
  item 1 -> group[1]=1 -> itemsInGroup[1] = [1]
  item 0 -> group[0]=0 -> itemsInGroup[0] = [0]

Step 6: concatenate following groupOrder=[1,0]:
  group 1 -> itemsInGroup[1] = [1] -> append 1
  group 0 -> itemsInGroup[0] = [0] -> append 0

FINAL RESULT: [1, 0]
```

Verify: item 0 depends on item 1, and in the result, 1 appears before 0. ✓ Correct.

**Why itemOrder alone is not the final answer:** in larger cases with multiple groups and multiple items per group, itemOrder is a valid GLOBAL order, but the problem's grouping structure requires the answer to still look "grouped" in a way consistent with group-level dependencies. Re-bucketing and re-concatenating by groupOrder guarantees this while item-level order is preserved via the stable bucketing in Step 5.

---

## SECTION 13 — COMPLEXITY TABLE

| Algorithm / Problem | Time | Space | Notes |
|---|---|---|---|
| Kahn's Algorithm (BFS) | O(V+E) | O(V+E) | Queue-based, cycle detection via `processed < V` |
| DFS-Based Topo Sort | O(V+E) | O(V+E) | Recursion stack + color array; must reverse post-order |
| Cycle Detection (Kahn's) | O(V+E) | O(V) | Free byproduct — no separate pass needed |
| Cycle Detection (3-color DFS) | O(V+E) | O(V) | GRAY-to-GRAY edge = cycle |
| All Topological Orderings | O(V! · (V+E)) worst case | O(V) | Backtracking; exponential, rarely coded in full |
| Course Schedule (LC 207) | O(V+E) | O(V+E) | Kahn's or DFS, either works |
| Course Schedule II (LC 210) | O(V+E) | O(V+E) | Kahn's preferred (order falls out naturally) |
| Alien Dictionary (LC 269) | O(C) + O(26) | O(26²) worst | C = total characters across all words |
| Minimum Height Trees (LC 310) | O(V) | O(V) | Undirected leaf-trimming, not Kahn's on a DAG |
| Course Schedule IV (LC 1462) | O(V² + V·E) | O(V²) | Boolean reachability matrix via topo + DP |
| Longest Increasing Path (LC 329) | O(mn) | O(mn) | Memoized DFS on implicit DAG |
| Parallel Courses (LC 1136) | O(V+E) | O(V+E) | BFS level count = longest path length |
| Sort Items by Groups (LC 1203) | O(V+E) | O(V+E) | Two independent Kahn's passes |

---

## SECTION 14 — VARIANTS

**Lexicographically smallest topological order:** replace the `queue<int>` with a `priority_queue<int, vector<int>, greater<int>>`. Popping the smallest available node at each step guarantees the smallest valid order (used in "Course Schedule III" style follow-ups and Build a Matrix With Conditions when a tie-break is required).

**Detecting a UNIQUE topological order:** a DAG has exactly one valid topological order if and only if, at every step of Kahn's algorithm, the queue never contains more than 1 node simultaneously (i.e., there's always exactly one "free" choice). This check appears in Sequence Reconstruction (LC 444) — the true task is not just "can we reconstruct the sequence" but "is this reconstruction the UNIQUE one consistent with all given subsequences."

**DAG shortest / longest path via topo order + DP:** once you have a topological order, a single forward pass computes shortest or longest paths from any source in O(V+E) — dramatically simpler and faster than Dijkstra (no relaxation loop, no priority queue, and it correctly handles negative edge weights, which Dijkstra cannot). `dist[v] = min/max over all edges u->v of dist[u] + weight(u,v)`, processed strictly in topo order.

**SCC condensation + topological sort:** for a general directed graph (possibly with cycles), first collapse each Strongly Connected Component (via Tarjan's or Kosaraju's algorithm) into a single "super-node." The resulting condensation graph is guaranteed to be a DAG, and can then be topologically sorted. This combo appears in hard-rated problems asking for "the minimum number of edges to add to make the graph strongly connected" and similar.

**Grid / implicit-DAG topological sort:** as seen in Longest Increasing Path — no explicit adjacency list is built; edges are derived on the fly from a comparison rule (`neighbor value > current value`). Solvable via memoized DFS (simpler to code) or explicit Kahn's "onion peeling" from local extrema (clarifies the DAG-DP connection).

**Soft / grouped dependencies:** Sort Items by Groups Respecting Dependencies is the canonical example — solve with two independent topological sorts (fine-grained + coarse-grained) rather than trying to force a single-pass solution.

---

## SECTION 15 — COMMON MISTAKES

### Mistake 1: Not detecting the cycle (missing the size check)

```cpp
// WRONG — returns whatever partial order was built, even with a cycle
vector<int> topoSort(int n, vector<vector<int>>& adj) {
    vector<int> indegree(n, 0);
    for (int u = 0; u < n; u++)
        for (int v : adj[u]) indegree[v]++;
    queue<int> q;
    for (int i = 0; i < n; i++) if (indegree[i] == 0) q.push(i);
    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u]) if (--indegree[v] == 0) q.push(v);
    }
    return order;   // BUG: might be shorter than n and caller won't know!
}

// CORRECT — explicitly check and signal failure
vector<int> topoSort(int n, vector<vector<int>>& adj) {
    // ... same body ...
    if ((int)order.size() != n) return {};   // cycle exists — caller MUST check this
    return order;
}
```
This is the single most common bug in this entire pattern. A queue that empties early means the remaining nodes' in-degrees never reached 0 — they form a cycle (or depend on one).

---

### Mistake 2: Alien Dictionary — missing the invalid-prefix edge case

```cpp
// WRONG — only checks for differing characters, never checks prefix validity
for (int i = 0; i + 1 < words.size(); i++) {
    string& w1 = words[i], & w2 = words[i+1];
    for (int j = 0; j < min(w1.size(), w2.size()); j++) {
        if (w1[j] != w2[j]) {
            adj[w1[j]].insert(w2[j]);
            break;
        }
    }
    // BUG: silently accepts ["abc", "ab"] as valid input — it isn't!
}

// CORRECT — explicitly reject longer-word-before-its-own-prefix
for (int i = 0; i + 1 < words.size(); i++) {
    string& w1 = words[i], & w2 = words[i+1];
    int minLen = min(w1.size(), w2.size());
    bool found = false;
    for (int j = 0; j < minLen; j++) {
        if (w1[j] != w2[j]) { adj[w1[j]].insert(w2[j]); found = true; break; }
    }
    if (!found && w1.size() > w2.size()) return "";  // invalid dictionary order
}
```

---

### Mistake 3: Building edges in the wrong direction

```cpp
// WRONG — Course Schedule: prerequisites[i]=[a,b] means "a needs b",
// but this builds the edge a -> b (backward!)
for (auto& p : prerequisites) {
    adj[p[0]].push_back(p[1]);   // BUG: a -> b is backward
    indegree[p[1]]++;
}
// This makes courses-with-no-prerequisites LOOK like they have high
// indegree, and courses that unlock nothing look "free" — completely
// inverted semantics. Kahn's will either produce a nonsensical order
// or falsely detect a cycle where none exists.

// CORRECT — the edge must point from "comes first" to "comes after"
for (auto& p : prerequisites) {
    int a = p[0], b = p[1];      // a requires b
    adj[b].push_back(a);         // b -> a  (b before a)
    indegree[a]++;
}
```

---

### Mistake 4: In-degree double-counted (or under-counted) from duplicate edges

```cpp
// WRONG — Alien Dictionary building the SAME edge twice from two
// different adjacent-word pairs inflates indegree, causing Kahn's
// to never fully drain that node's indegree to 0 -> false cycle.
adj[w1[j]].insert(w2[j]);   // set insert dedups the ADJACENCY,
indegree[w2[j]]++;           // but this increments EVERY time regardless!

// CORRECT — only increment indegree on a genuinely NEW edge
if (!adj[w1[j]].count(w2[j])) {
    adj[w1[j]].insert(w2[j]);
    indegree[w2[j]]++;
}
```
This bug is subtle because the graph still "looks right" if you print `adj` — the adjacency set correctly de-duplicates. But `indegree` was incremented on every occurrence, not just the first, so it never reaches exactly 0 and Kahn's silently drops that branch, producing a shorter (wrongly "cyclic") order.

---

### Mistake 5: DFS-based topo sort — forgetting to reverse (or reversing when using a stack already handles it)

```cpp
// WRONG — appending in post-order but NOT reversing before returning
vector<int> order;
// ... dfsVisit appends to order on BLACK ...
return order;   // BUG: this is REVERSE topological order, not topological order!

// CORRECT — reverse after all DFS calls finish
reverse(order.begin(), order.end());
return order;

// ALTERNATIVE that avoids the bug entirely: push to a stack instead
// of a vector, then pop everything at the end (stack naturally
// reverses insertion order for you).
stack<int> st;
// ... dfsVisit does st.push(u) on BLACK instead of order.push_back(u) ...
vector<int> order;
while (!st.empty()) { order.push_back(st.top()); st.pop(); }
```

---

### Mistake 6: Single boolean `visited` array instead of 3-color for cycle detection

```cpp
// WRONG — a single visited[] array cannot distinguish "currently on
// the recursion stack" (GRAY) from "fully finished, safe" (BLACK).
// This causes FALSE POSITIVE cycle detection on diamond-shaped DAGs.
bool dfs(int u, vector<vector<int>>& adj, vector<bool>& visited) {
    if (visited[u]) return false;   // BUG: treats "already finished" as a cycle!
    visited[u] = true;
    for (int v : adj[u])
        if (!dfs(v, adj, visited)) return false;
    return true;
}
// Example failure: DAG with edges 0->1, 0->2, 1->3, 2->3 (a diamond).
// DFS(0) visits 1, which visits 3 (marks visited). Backtrack to 0,
// visit 2, which tries to visit 3 — but 3 is ALREADY visited (from
// the 1->3 branch, fully finished, no cycle) — this WRONGLY reports
// a cycle, even though 0->1->3 and 0->2->3 is a perfectly valid DAG.

// CORRECT — 3-color distinguishes "in-progress ancestor" from "done"
enum Color { WHITE, GRAY, BLACK };
bool dfs(int u, vector<vector<int>>& adj, vector<int>& color) {
    color[u] = GRAY;
    for (int v : adj[u]) {
        if (color[v] == GRAY) return false;              // real cycle
        if (color[v] == WHITE && !dfs(v, adj, color)) return false;
        // color[v] == BLACK -> already fully explored, definitely safe
    }
    color[u] = BLACK;
    return true;
}
```

---

## SECTION 16 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Course Schedule | 207 | Kahn's algorithm, cycle detection via processed count |
| 2 | Course Schedule II | 210 | Kahn's algorithm, producing an actual valid order |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Alien Dictionary | 269 | Building the graph from constraints, invalid-prefix edge case |
| 4 | Minimum Height Trees | 310 | Iterative leaf trimming on an undirected tree |
| 5 | Course Schedule IV | 1462 | Reachability via topo order + boolean DP |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Longest Increasing Path in a Matrix | 329 | Implicit DAG, memoized DFS |
| 7 | Parallel Courses | 1136 | Longest path in a DAG via level-order Kahn's |
| 8 | Sequence Reconstruction | 444 | Uniqueness of topological order (queue size ≤ 1 at every step) |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Sort Items by Groups Respecting Dependencies | 1203 | Two-level topological sort (items + groups) |
| 10 | Build a Matrix With Conditions | 2392 | Two independent topo sorts (rows + columns) merged into a grid |

---

## SECTION 17 — PATTERN CONNECTIONS

1. **Pattern 12 (Graphs: BFS/DFS):** Topological sort is literally BFS (Kahn's) or DFS (post-order + reverse) with one extra piece of bookkeeping (in-degree, or 3-color state). Every template in this pattern assumes fluency with Pattern 12's adjacency-list construction and traversal mechanics. If Course Schedule feels hard, the gap is usually Pattern 12, not this pattern.

2. **Pattern 16 (DP on DAG / Graph DP):** Once you have a topological order, DAG DP becomes trivial — a single forward pass, no memoization needed, no worrying about "have I computed this subproblem yet" because topo order GUARANTEES every dependency is already resolved. Longest Increasing Path, Parallel Courses, and Course Schedule IV in this document are all secretly Pattern 16 problems wearing a topological-sort disguise.

3. **Pattern 31 (Minimum Spanning Tree):** Both patterns build a structure incrementally from a priority/queue-driven process over a graph (Kahn's queue vs. Prim's/Kruskal's greedy edge selection), and both rely on the same core discipline: process elements in an order where correctness at each step depends only on already-finalized state. MST algorithms don't require a DAG, but the "greedily consume the next safe/cheapest choice" mental model transfers directly.

4. **Pattern 09 (Union-Find / Disjoint Set):** For UNDIRECTED cycle detection, Union-Find is the standard tool (union two endpoints; if they're already in the same set, adding the edge creates a cycle) — contrast this with directed cycle detection, which requires topological sort or 3-color DFS specifically because Union-Find has no notion of edge direction.

5. **Pattern 21 (Backtracking):** "Enumerate ALL valid topological orderings" (Taxonomy Type 4) is a direct application of the standard backtracking template — choose a currently-available node, recurse, undo. It is the natural follow-up question after a candidate produces ONE valid topo sort successfully.

6. **Pattern 13 (Dijkstra / Weighted Shortest Path):** Topological-sort-based DAG shortest/longest path is strictly faster (O(V+E), no priority queue) and handles negative weights, unlike Dijkstra. Recognizing "this weighted graph happens to be a DAG" is a strong 2000+ signal — it means you should reach for topo sort + DP instead of Dijkstra.

---

## SECTION 18 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 207 — Course Schedule

**Interviewer:** "You're given `numCourses` and a list of prerequisite pairs. Can you finish all courses?"

**Candidate:**

*[First 30 seconds — recognize the pattern]*
> "This is a cycle-detection problem on a directed graph. `prerequisites[i] = [a, b]` means a requires b, so the edge is `b → a`. I can finish all courses if and only if this graph is a DAG — no cycles. I'll run Kahn's algorithm: if I can process every node, it's a DAG; if the queue empties early, there's a cycle."

*[State the edge direction explicitly before coding — this is the #1 place candidates slip]*
> "One thing I want to get right immediately: the edge points from the PREREQUISITE to the course that needs it, because that's the 'must happen before' direction. `adj[b].push_back(a)`, `indegree[a]++`."

*[Code — 5 minutes]*
```cpp
bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
    vector<vector<int>> adj(numCourses);
    vector<int> indegree(numCourses, 0);
    for (auto& p : prerequisites) {
        adj[p[1]].push_back(p[0]);
        indegree[p[0]]++;
    }
    queue<int> q;
    for (int i = 0; i < numCourses; i++)
        if (indegree[i] == 0) q.push(i);
    int processed = 0;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        processed++;
        for (int v : adj[u])
            if (--indegree[v] == 0) q.push(v);
    }
    return processed == numCourses;
}
```

*[Follow-up]: "Can you solve it with DFS instead?"*
> "Yes — 3-color marking. WHITE/GRAY/BLACK. If DFS from any node reaches a GRAY node, that's a back edge to an ancestor still being processed — a cycle. I'd prefer Kahn's here because it also directly answers 'give me a valid order' (LC 210) with zero extra work, whereas DFS needs the post-order reversal step."

**RED FLAGS:**
- Building the edge in the wrong direction (`adj[a].push_back(b)` instead of `adj[b].push_back(a)`)
- Forgetting the final `processed == numCourses` check and just returning `true` unconditionally
- Not being able to explain WHY an early-emptying queue implies a cycle

---

### Interview Simulation 2: LC 269 — Alien Dictionary

**Interviewer:** "Given a list of words sorted according to the rules of an alien language's alphabet, derive a valid alphabet order."

**Candidate:**

*[Identify the two sub-problems immediately]*
> "There are two separate things to get right here: building the graph correctly from adjacent word comparisons, and then a completely standard topological sort. The graph-building is where the real difficulty is."

*[State the core rule and the edge case BEFORE coding]*
> "For each pair of adjacent words, I look at the FIRST character where they differ — that gives one directed edge, earlier char before later char. I only need the first difference; anything after it gives no ordering information. The edge case: if I scan the full length of the shorter word and find NO difference, that means one word is a prefix of the other. If the SHORTER word comes first, that's fine and expected — no edge, no problem. But if the LONGER word comes first and the shorter one is its prefix — like `['abc', 'ab']` — that's an invalid input, because no valid alphabet ordering ever sorts a word before its own prefix. I need to return empty string in that case."

*[Code — 8 minutes, narrating the edge case explicitly while writing it]*
> "...and here's the check I mentioned — `if (!foundDifference && w1.size() > w2.size()) return "";`"

*[Interviewer]: "What if a character never appears in any prerequisite relationship?"*
> "It still needs to appear somewhere in the output — I pre-seed the indegree map with every character seen in any word, defaulted to 0, before processing any edges. That way a character with zero constraints still gets picked up by the initial queue seeding and appears in the final order."

**RED FLAGS:**
- Missing the invalid-prefix edge case entirely (very common — most candidates get the main algorithm right and miss this)
- Comparing full words instead of stopping at the first difference (wastes time and can introduce wrong edges)
- Forgetting to pre-register every character (a character with no constraints silently vanishes from the output)

---

### Interview Simulation 3: Meta-Discussion — Kahn's vs. DFS-Based Topological Sort

**Interviewer:** "You have two ways to compute a topological sort. When would you choose one over the other?"

**Candidate:**
> "Both are O(V+E) and both work on any DAG, so the choice usually comes down to what ELSE the problem needs.

> I reach for **Kahn's (BFS)** when: I also need level information (Parallel Courses — semesters are BFS levels), I want the lexicographically smallest order (swap the queue for a min-heap), or I just want the simplest possible cycle check (`processed != n`). It's also generally easier to reason about out loud — 'nodes with no unmet dependencies go first' is intuitive.

> I reach for **DFS-based** when: I'm already doing a DFS for another reason and want the topo order as a side effect (e.g., as a stepping stone toward Kosaraju's algorithm for strongly connected components, which explicitly needs DFS finish-order), or when the graph is naturally expressed recursively and an iterative BFS would require building extra structure.

> The one bug that's specific to DFS-based topo sort and doesn't exist in Kahn's: forgetting to reverse the post-order list. Kahn's has no equivalent trap — the order falls out correctly in forward direction by construction."

**RED FLAGS:**
- Treating the two as interchangeable without being able to articulate a reason to prefer one
- Not knowing that DFS-based topo sort requires a reversal step
- Confusing "DFS finds A topological order" with "DFS finds THE topological order" — there can be many valid orders, and DFS's choice depends on iteration order of the adjacency list, exactly like Kahn's does

---

## SECTION 19 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why does Kahn's algorithm correctly detect a cycle when `processedCount < V` at the end?**

> Every node that gets pushed to the queue has, by construction, an in-degree of exactly 0 at that moment — meaning every predecessor it has has already been fully processed. If a set of nodes forms a cycle, every node in that cycle has at least one predecessor WITHIN the cycle that can never be processed first (there's no valid starting point inside a cycle — it's circular by definition). So none of those nodes' in-degrees will ever reach 0, they never enter the queue, and they never get counted in `processedCount`. A shortfall in the final count exactly identifies "there exists a cycle somewhere among the unprocessed nodes."

---

**Q2: In DFS-based topo sort, why do we append nodes in POST-order and then reverse, rather than appending in pre-order?**

> A node should appear in the topological order only AFTER everything it depends on (its descendants in the DFS tree, i.e., everything reachable from it) has already been fully placed — because those descendants must come first if there's an edge from the node to them... wait, more precisely: if there's an edge `u → v`, u must come BEFORE v in the final order. In DFS, `v` (the descendant) finishes and is marked BLACK/pushed BEFORE `u` finishes (since `u`'s DFS call is still "in progress" while exploring `v`). So post-order naturally produces the REVERSE of what we want — v gets appended before u, but we need u before v. Reversing the entire post-order list fixes this: after reversal, u (finished later, appended later, ends up earlier after reversal) correctly precedes v.

---

**Q3: Alien Dictionary — why is "a longer word appearing immediately before its own prefix" invalid? Give a concrete example.**

> Example: `["abc", "ab"]`. In ANY valid alphabet ordering, if two strings share a common prefix and one is strictly the prefix of the other, the SHORTER one must sort first — this is how dictionary/lexicographic order works universally, regardless of what the alphabet's actual letter order is. `"ab"` is a prefix of `"abc"`, so `"ab"` must come before `"abc"` in any valid sorted list. If the input claims `"abc"` comes before `"ab"`, that's a direct contradiction of how sorting works — no character reordering can fix it, because the comparison never even reaches a differing character. This makes the entire input invalid, and the correct response is to return `""`.

---

**Q4: Course Schedule: `prerequisites[i] = [a, b]` means "to take a, take b first." Which direction is the edge, `b → a` or `a → b`, and why does it matter for in-degree computation?**

> The edge is `b → a`. Read the dependency as "b must be completed before a" — the arrow points from the thing that happens FIRST to the thing that happens AFTER, matching the general rule "topological sort edges point in the direction of required ordering." Getting this backward doesn't just shift the output — it inverts the entire in-degree structure: courses with NO prerequisites (which should start with in-degree 0 and be processed first) would instead get artificially high in-degree (equal to however many other courses list them as a prerequisite), while courses that depend on many things would incorrectly show in-degree 0. Kahn's algorithm would then seed the queue with completely wrong "starting" nodes, either producing a nonsensical order or falsely reporting a cycle.

---

**Q5: Minimum Height Trees — why does iteratively removing leaves converge to the graph's "center," and why can there be at most 2 such nodes?**

> A tree's height (when rooted at some node r) equals the length of the longest path from r to any other node. This is minimized when r is as close as possible to ALL leaves simultaneously — which is exactly the middle of the tree's DIAMETER (the longest path between any two nodes in the whole tree). Repeatedly trimming all current leaves shrinks the tree symmetrically inward from every direction at the same rate (one "ring" per round), like a rope burning from both ends toward the middle. The node(s) left when the burning meets are precisely the diameter's midpoint. If the diameter has an ODD number of edges, the two burn-fronts meet exactly at one node — a unique center. If EVEN, they meet at two adjacent nodes simultaneously — both are equally valid centers (rooting at either produces the same minimum height), which is why the answer can have at most 2 elements.

---

**Q6: Why is 3-color marking necessary for DFS cycle detection instead of a single boolean `visited` array? Give an example where the single-array version fails.**

> A single boolean array cannot distinguish "this node is an ANCESTOR of the node I'm currently exploring (still on the recursion stack, in progress)" from "this node was fully explored via a COMPLETELY DIFFERENT branch earlier and is provably safe." Example: a diamond DAG with edges `0→1, 0→2, 1→3, 2→3` (0 splits into two branches that both converge on 3 — a completely valid DAG, no cycle). DFS from 0 visits 1, which visits 3 and marks it visited, then finishes and backtracks. DFS then visits 2, which tries to visit 3 — but 3 is already marked visited (boolean), so a naive check `if (visited[u]) return false` WRONGLY reports a cycle, even though 3 was reached via two entirely separate, non-overlapping paths and there's no actual cycle. 3-color marking fixes this: node 3 is BLACK (fully finished) by the time the second branch reaches it, not GRAY (in-progress ancestor), so it's correctly recognized as safe.

---

**Q7: When should you prefer Kahn's (BFS) topological sort over DFS-based, and vice versa?**

> Prefer **Kahn's** when: you need level/layer information (minimum semesters, BFS-distance-style problems), you want the lexicographically smallest valid order (swap the queue for a min-heap), or you want the simplest possible cycle-detection logic (`processedCount != n`). Prefer **DFS-based** when: you're already running DFS for another purpose and want topo order as a natural side effect (most notably as a building block for Kosaraju's Strongly Connected Components algorithm, which explicitly requires DFS finish-order on the original graph before a second DFS pass on the transposed graph), or the input is more naturally expressed as a recursive structure. In practice, for a plain "give me a topological order" interview question, Kahn's is usually the safer default — it's easier to narrate, has one fewer trap (no reversal step), and generalizes more easily to variants that need extra bookkeeping (levels, lexicographic order, reachability).

---

## SECTION 20 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. KAHN'S ALGORITHM (BFS + in-degree) — THE DEFAULT CHOICE
// ─────────────────────────────────────────────────────────────────
vector<int> indegree(n, 0);
for (int u = 0; u < n; u++)
    for (int v : adj[u]) indegree[v]++;

queue<int> q;
for (int i = 0; i < n; i++)
    if (indegree[i] == 0) q.push(i);

vector<int> order;
while (!q.empty()) {
    int u = q.front(); q.pop();
    order.push_back(u);
    for (int v : adj[u])
        if (--indegree[v] == 0) q.push(v);
}
// CYCLE CHECK — NEVER SKIP THIS:
if ((int)order.size() != n) { /* cycle exists */ }

// ─────────────────────────────────────────────────────────────────
// 2. DFS-BASED TOPO SORT (3-color, cycle-aware)
// ─────────────────────────────────────────────────────────────────
enum Color { WHITE, GRAY, BLACK };
vector<int> color(n, WHITE), order;
bool dfs(int u) {
    color[u] = GRAY;
    for (int v : adj[u]) {
        if (color[v] == GRAY) return false;         // cycle
        if (color[v] == WHITE && !dfs(v)) return false;
    }
    color[u] = BLACK;
    order.push_back(u);
    return true;
}
// After all dfs(i) calls: reverse(order.begin(), order.end());

// ─────────────────────────────────────────────────────────────────
// 3. LEXICOGRAPHICALLY SMALLEST TOPO ORDER
// ─────────────────────────────────────────────────────────────────
priority_queue<int, vector<int>, greater<int>> pq;   // min-heap instead of queue

// ─────────────────────────────────────────────────────────────────
// 4. UNIQUE TOPOLOGICAL ORDER CHECK (Sequence Reconstruction style)
// ─────────────────────────────────────────────────────────────────
// At every step of Kahn's, the queue must contain AT MOST 1 node.
// If q.size() > 1 at any point, more than one valid order exists.
if (q.size() > 1) { /* not unique */ }

// ─────────────────────────────────────────────────────────────────
// 5. DAG LONGEST/SHORTEST PATH (topo order + one DP pass)
// ─────────────────────────────────────────────────────────────────
vector<int> topo = kahnTopoSort(n, adj);   // must be a valid DAG
vector<int> dist(n, /* -INF for longest, +INF for shortest */);
dist[source] = 0;
for (int u : topo) {
    if (dist[u] == /* unreachable sentinel */) continue;
    for (auto [v, w] : weightedAdj[u])
        dist[v] = max(dist[v], dist[u] + w);   // or min() for shortest
}

// ─────────────────────────────────────────────────────────────────
// 6. UNDIRECTED "CENTER OF TREE" VIA LEAF TRIMMING (LC 310 style)
// ─────────────────────────────────────────────────────────────────
queue<int> leaves;
for (int i = 0; i < n; i++) if (degree[i] == 1) leaves.push(i);
int remaining = n;
while (remaining > 2) {
    int sz = leaves.size(); remaining -= sz;
    for (int i = 0; i < sz; i++) {
        int leaf = leaves.front(); leaves.pop();
        for (int nb : adj[leaf])
            if (--degree[nb] == 1) leaves.push(nb);
    }
}
// whatever's left in `leaves` (1 or 2 nodes) = the center(s)

// ─────────────────────────────────────────────────────────────────
// 7. EDGE-DIRECTION SANITY CHECK (memorize this phrase)
// ─────────────────────────────────────────────────────────────────
// "prerequisites[i] = [a, b]" means "a needs b" means edge b -> a
// The arrow ALWAYS points from "must happen first" to "happens after."
// indegree[the thing that happens after]++
```

---

## SECTION 21 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 10 problems in order.** Alien Dictionary and Sort Items by Groups are the two that require real insight beyond the core algorithm. Budget 45 minutes each on first attempt.

2. **Drill the edge-direction rule until it's automatic.** Write Course Schedule from a blank file three times across three different days, without looking at your notes. The goal is that `adj[b].push_back(a)` for "a needs b" becomes as automatic as `dp[i] = dp[i-1] + dp[i-2]` is for Fibonacci.

3. **After Alien Dictionary, explicitly construct 3 invalid inputs** (a true cycle, a longer-word-before-prefix case, and a case with a completely disconnected/unconstrained character) and verify your code returns `""` or includes the free character correctly for each. This cements the edge-case handling that interviews actually probe.

4. **Preview Pattern 16 (DP on DAG):** Course Schedule IV, Parallel Courses, and Longest Increasing Path are all "topo sort gives you the safe processing order, then a DP pass computes the real answer." Pattern 16 generalizes this to weighted DAGs and counting problems (number of distinct paths, etc.).

5. **The two-level topo sort challenge:** After finishing Sort Items by Groups Respecting Dependencies (LC 1203), try Build a Matrix With Conditions (LC 2392) immediately — it's the same "two independent topological sorts merged into one output" shape, but applied to rows and columns of a grid instead of items and groups. Recognizing the shared skeleton between these two problems is a strong signal you've internalized the pattern rather than memorized individual solutions.

---

## SECTION 22 — SIGN-OFF CRITERIA

### Tier 1 — Core algorithm mastered
You write Kahn's algorithm from a blank file in under 4 minutes, correctly orient every edge on the first try, and never forget the `processed == n` cycle check. You solve Course Schedule and Course Schedule II cleanly.

### Tier 2 — Cycle detection understood both ways
You can implement DFS-based topo sort with 3-color marking from memory, correctly explain why post-order requires a reversal, and explain — with a concrete diamond-DAG example — why a single boolean `visited` array produces false-positive cycles.

### Tier 3 — Graph construction from constraints solid
You solve Alien Dictionary cleanly, including the invalid-prefix edge case, without prompting. You solve Minimum Height Trees using leaf-trimming and can explain why the result has at most 2 nodes. You solve Course Schedule IV using the topo-order-plus-propagation technique.

### Tier 4 — Advanced variants complete
You solve Longest Increasing Path in a Matrix and can articulate why it's an implicit DAG. You solve Parallel Courses and explain the level-batching semantics of "parallel." You solve Sort Items by Groups Respecting Dependencies using the two-level topo sort strategy, and can explain why bucketing items by `itemOrder` before concatenating by `groupOrder` is necessary (rather than just running one combined topo sort).

**Sign-off threshold:** Solve 8 of 10 problems. Mandatory: LC 207 (Course Schedule — edge direction and the cycle-check must be automatic), LC 269 (Alien Dictionary — the invalid-prefix edge case must be handled without hints), and LC 1203 (Sort Items by Groups — the two-level topo sort structure must be explainable, not just codeable). These three represent the three hardest conceptual challenges in this pattern.

---

*Pattern 30 Complete — Topological Sort*
*Next: Pattern 31 — Minimum Spanning Tree (Prim's, Kruskal's, Union-Find-driven greedy graph construction)*
