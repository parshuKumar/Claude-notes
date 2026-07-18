# PATTERN 32 — STRONGLY CONNECTED COMPONENTS
## Kosaraju, Tarjan, Bridges, Articulation Points, 2-SAT — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**This pattern lives almost entirely in competitive programming, not interviews.**

Strongly Connected Components (SCC) theory — Kosaraju's algorithm, Tarjan's algorithm, bridges, articulation points, and 2-SAT — is Codeforces Div 2 E / Div 1 C territory, not FAANG-interview territory. You will almost never be asked to implement Tarjan's SCC from scratch in a 45-minute interview. What DOES show up in interviews:

- **Critical Connections in a Network (LC 1192)** — a disguised "find bridges" problem, occasionally asked at senior/staff level or by infra-heavy companies (network reliability, distributed systems).
- **Course Schedule variants** — cycle detection, which is a degenerate case of SCC (a single SCC of size > 1 means a cycle exists).
- Conceptual questions: "How would you detect a group of mutually-reachable servers?" — testing whether you *know* SCC exists as a concept, not whether you can code Tarjan on a whiteboard in 10 minutes.

Where this pattern is **essential**: competitive programming. SCC condensation (collapsing each SCC into a single DAG node) is one of the most powerful "trick reveals" in CP — turn an arbitrary directed graph into a DAG, then apply DAG DP/topological sort techniques (Pattern 30) on the condensation. 2-SAT is a full "hidden pattern" in itself: an enormous class of constraint-satisfaction problems ("at most one of A, B can be true," "if A then not B," scheduling with binary choices) reduce to 2-SAT, which reduces to SCC.

**Kosaraju vs Tarjan — which to actually learn:**

| | Kosaraju | Tarjan |
|---|---|---|
| DFS passes | 2 (forward + on transpose) | 1 |
| Extra graph needed | Yes — must build transpose | No |
| Conceptual complexity | Lower — "obvious" once you see it | Higher — low-link values take practice |
| Component numbering | Natural topological order (source = 0) | Natural reverse-topological order (sink = 0) |
| Interview-friendliness | Easier to explain and re-derive under pressure | Faster to code once memorized, but easy to mess up on-the-fly |
| CP preference | Rare — extra graph construction is wasteful | Dominant — single pass, extends directly to bridges/articulation points |

**The honest recommendation:** learn Kosaraju first to *understand* why SCC condensation works (it's intuitive: run DFS, then run DFS again in reverse on the "most finished" vertices first). Then learn Tarjan because it is what you will actually use in contests — the low-link machinery it introduces is the SAME machinery used for bridges and articulation points, so mastering it once gives you three algorithms.

**The key skill this pattern tests:**

1. Do you understand *why* SCC condensation always produces a DAG (no cycles across components)?
2. Can you correctly derive and defend the bridge condition `low[v] > disc[u]` (strict inequality) vs articulation condition `low[v] >= disc[u]` (non-strict) — these look almost identical but differ for a precise structural reason?
3. Can you reduce a 2-SAT constraint problem to an implication graph without hand-waving?

If you can do all three cleanly, you're operating at CF 1900+ graph-theory level.

---

## SECTION 2 — DEFINITIONS

**Strongly Connected Component (SCC):** A maximal set of vertices `C` in a directed graph such that for every pair `u, v ∈ C`, there is a directed path from `u` to `v` AND a directed path from `v` to `u`. Every vertex belongs to exactly one SCC (SCCs partition the vertex set).

**Condensation graph (component graph):** Collapse each SCC into a single "super-node." Draw an edge from super-node `A` to super-node `B` if there is at least one edge in the original graph from some vertex in `A` to some vertex in `B` (A ≠ B). **The condensation is always a DAG** — if it had a cycle among super-nodes, all vertices in that cycle of super-nodes would be mutually reachable, meaning they'd have to be a single SCC, contradicting that they were separate super-nodes.

**Discovery time `disc[u]`:** The timestamp (an increasing counter) when vertex `u` is first visited by DFS.

**Low-link value `low[u]`:** The smallest discovery time reachable from `u` by taking zero or more tree edges (DFS-tree edges) followed by **at most one** back/forward/cross edge. Intuitively: "how far back up the DFS tree can I reach from the subtree rooted at `u`, using at most one non-tree edge to jump?" This is the single idea underlying Tarjan's SCC, Tarjan's bridges, and Tarjan's articulation points — only the exact comparison at the end differs.

**Bridge (cut edge):** An edge `(u, v)` in an **undirected** graph whose removal increases the number of connected components (disconnects the graph, or a subgraph).

**Articulation point (cut vertex):** A vertex in an **undirected** graph whose removal (along with all incident edges) increases the number of connected components.

**2-SAT:** A Boolean satisfiability problem restricted to clauses with exactly 2 literals, e.g. `(x1 ∨ ¬x3) ∧ (¬x2 ∨ x4) ∧ ...`. Unlike general 3-SAT (NP-complete), 2-SAT is solvable in **linear time** via SCC.

---

## SECTION 3 — TEMPLATE 1: KOSARAJU'S ALGORITHM

```cpp
// ─────────────────────────────────────────────────────────────────
// KOSARAJU'S SCC ALGORITHM
//
// Idea: 
//  1) Run DFS on G, record vertices by FINISH time (post-order).
//  2) Build the transpose graph G^T (reverse every edge).
//  3) Process vertices in DECREASING finish-time order; for each
//     unvisited vertex, DFS on G^T. Each DFS tree found = one SCC.
//
// WHY IT WORKS: the vertex with the highest finish time in step 1
// belongs to a "source" SCC of the condensation DAG (no incoming
// edges from any unvisited SCC). Running DFS on the TRANSPOSE from
// that vertex can only walk into vertices of the SAME SCC — because
// any edge leading to a DIFFERENT SCC in G^T would mean, in the
// original graph G, an edge FROM another SCC INTO this one, which
// would contradict it being a source. Repeating this peels off
// SCCs one at a time, always taking the current topological source.
//
// Component numbering: comp[] is assigned in DISCOVERY order during
// step 3, which is FORWARD topological order of the condensation
// (comp 0 = source-most SCC, comp (k-1) = sink-most SCC).
// ─────────────────────────────────────────────────────────────────

void dfs1(int u, vector<vector<int>>& adj, vector<bool>& visited, vector<int>& order) {
    visited[u] = true;
    for (int v : adj[u]) {
        if (!visited[v]) dfs1(v, adj, visited, order);
    }
    order.push_back(u);  // push on FINISH (post-order) — this is the key trick
}

void dfs2(int u, vector<vector<int>>& radj, vector<bool>& visited,
          vector<int>& comp, int compId) {
    visited[u] = true;
    comp[u] = compId;
    for (int v : radj[u]) {
        if (!visited[v]) dfs2(v, radj, visited, comp, compId);
    }
}

// Returns number of SCCs. Fills comp[v] = component id of vertex v
// (comp id 0 = topologically first / source-most component).
int kosarajuSCC(int n, vector<vector<int>>& adj) {
    // Step 1: build transpose graph
    vector<vector<int>> radj(n);
    for (int u = 0; u < n; u++)
        for (int v : adj[u])
            radj[v].push_back(u);

    // Step 2: first DFS pass on original graph — get finish order
    vector<bool> visited(n, false);
    vector<int> order;
    order.reserve(n);
    for (int i = 0; i < n; i++)
        if (!visited[i]) dfs1(i, adj, visited, order);

    // Step 3: second DFS pass on transpose, in DECREASING finish-time order
    fill(visited.begin(), visited.end(), false);
    vector<int> comp(n, -1);
    int compId = 0;
    for (int i = n - 1; i >= 0; i--) {
        int u = order[i];
        if (!visited[u]) {
            dfs2(u, radj, visited, comp, compId);
            compId++;
        }
    }
    return compId;   // comp[] now holds each vertex's SCC id
}
```

**FULL TRACE.** Graph: `0→1, 1→2, 2→0, 2→3, 3→4, 4→5, 5→6, 6→4, 6→7`.

```
Adjacency (adj):
  0: [1]
  1: [2]
  2: [0, 3]
  3: [4]
  4: [5]
  5: [6]
  6: [4, 7]
  7: []

Expected SCCs: {0,1,2}, {3}, {4,5,6}, {7}   (condensation: {0,1,2}→{3}→{4,5,6}→{7})
```

**Step 2 — first DFS pass (starting at vertex 0), pushing on finish:**
```
dfs1(0): visit 0 → dfs1(1): visit 1 → dfs1(2): visit 2
  2's neighbors: 0 (visited, skip), 3 (unvisited) → dfs1(3): visit 3
    dfs1(4): visit 4 → dfs1(5): visit 5 → dfs1(6): visit 6
      6's neighbors: 4 (visited, skip), 7 (unvisited) → dfs1(7): visit 7
        7 has no neighbors → FINISH 7 → order=[7]
      FINISH 6 → order=[7,6]
    FINISH 5 → order=[7,6,5]
  FINISH 4 → order=[7,6,5,4]
FINISH 3 → order=[7,6,5,4,3]
FINISH 2 → order=[7,6,5,4,3,2]
FINISH 1 → order=[7,6,5,4,3,2,1]
FINISH 0 → order=[7,6,5,4,3,2,1,0]
```

**Step 1 (transpose) — reverse every edge:**
```
Transpose adjacency (radj):
  0: [2]
  1: [0]
  2: [1]
  3: [2]
  4: [3, 6]
  5: [4]
  6: [5]
  7: [6]
```

**Step 3 — process `order` from index n-1 down to 0 (decreasing finish time = 0,1,2,3,4,5,6,7), DFS on transpose:**
```
u=0 (unvisited) → dfs2(0, compId=0):
  visit 0, comp[0]=0. neighbors of 0 in radj: [2]
    visit 2, comp[2]=0. neighbors of 2 in radj: [1]
      visit 1, comp[1]=0. neighbors of 1 in radj: [0] (visited, skip)
  SCC #0 = {0, 2, 1} = {0,1,2} ✓   compId → 1

u=1 (visited, skip). u=2 (visited, skip).
u=3 (unvisited) → dfs2(3, compId=1):
  visit 3, comp[3]=1. neighbors of 3 in radj: [2] (visited, skip)
  SCC #1 = {3} ✓   compId → 2

u=4 (unvisited) → dfs2(4, compId=2):
  visit 4, comp[4]=2. neighbors of 4 in radj: [3 (visited), 6 (unvisited)]
    visit 6, comp[6]=2. neighbors of 6 in radj: [5]
      visit 5, comp[5]=2. neighbors of 5 in radj: [4] (visited, skip)
  SCC #2 = {4, 6, 5} = {4,5,6} ✓   compId → 3

u=5 (visited). u=6 (visited).
u=7 (unvisited) → dfs2(7, compId=3):
  visit 7, comp[7]=3. neighbors of 7 in radj: [6] (visited, skip)
  SCC #3 = {7} ✓   compId → 4

Final comp[] = [0,0,0,1,2,2,2,3]  →  4 SCCs, exactly as expected.
Component ids are in FORWARD topological order of the condensation:
  comp 0 ({0,1,2}) → comp 1 ({3}) → comp 2 ({4,5,6}) → comp 3 ({7})
```

---

## SECTION 4 — TEMPLATE 2: TARJAN'S SCC ALGORITHM (SINGLE-PASS)

```cpp
// ─────────────────────────────────────────────────────────────────
// TARJAN'S SCC ALGORITHM — single DFS pass using low-link values.
//
// State per vertex:
//   disc[u]     = discovery time (timestamp when first visited)
//   low[u]      = smallest disc[] reachable from u's subtree via
//                 tree edges + at most one back edge
//   onStack[u]  = is u currently on the explicit "candidate" stack?
//
// Every vertex is pushed onto `stk` when first visited. A vertex u
// is the ROOT of an SCC exactly when low[u] == disc[u] — meaning
// nothing in u's subtree can reach further back than u itself, so
// u and everything above it (still on the stack) that got pulled
// down to u's level form one closed SCC. Pop the stack down to and
// including u — that popped set is one full SCC.
//
// CRITICAL: when relaxing low[u] via an edge to an already-visited
// vertex v, you must check `onStack[v]`. If v is visited but NOT on
// the stack, it means v's SCC has ALREADY been finalized and popped
// — that edge point into a completely different (already-closed)
// SCC, and must be ignored, or low[] values get corrupted.
//
// Component numbering: comp[] is assigned in COMPLETION order
// (when low[u]==disc[u] fires), which is REVERSE topological order
// of the condensation (comp 0 = sink-most SCC, comp (k-1) = source-
// most SCC) — the OPPOSITE convention from Kosaraju.
// ─────────────────────────────────────────────────────────────────

class TarjanSCC {
public:
    int n, timer = 0, compCount = 0;
    vector<vector<int>>& adj;
    vector<int> disc, low, comp;
    vector<bool> onStack;
    stack<int> stk;

    TarjanSCC(int n, vector<vector<int>>& adj)
        : n(n), adj(adj), disc(n, -1), low(n, -1), comp(n, -1), onStack(n, false) {}

    void dfs(int u) {
        disc[u] = low[u] = timer++;
        stk.push(u);
        onStack[u] = true;

        for (int v : adj[u]) {
            if (disc[v] == -1) {                 // tree edge — unvisited child
                dfs(v);
                low[u] = min(low[u], low[v]);
            } else if (onStack[v]) {              // back edge to an ACTIVE ancestor
                low[u] = min(low[u], disc[v]);
            }
            // visited AND not onStack → edge into an already-closed SCC — IGNORE
        }

        if (low[u] == disc[u]) {                  // u is the root of an SCC
            while (true) {
                int v = stk.top(); stk.pop();
                onStack[v] = false;
                comp[v] = compCount;
                if (v == u) break;
            }
            compCount++;
        }
    }

    int run() {
        for (int i = 0; i < n; i++)
            if (disc[i] == -1) dfs(i);
        return compCount;
    }
};
```

**FULL TRACE on the same graph** (`0→1, 1→2, 2→0, 2→3, 3→4, 4→5, 5→6, 6→4, 6→7`), DFS starting at 0:

```
dfs(0): disc[0]=low[0]=0. push 0. stk=[0]
  → dfs(1): disc[1]=low[1]=1. push 1. stk=[0,1]
    → dfs(2): disc[2]=low[2]=2. push 2. stk=[0,1,2]
       neighbor 0: visited & onStack → low[2]=min(2, disc[0]=0) = 0
       neighbor 3: unvisited → dfs(3): disc[3]=low[3]=3. push 3. stk=[0,1,2,3]
          → dfs(4): disc[4]=low[4]=4. push 4. stk=[..,4]
             → dfs(5): disc[5]=low[5]=5. push 5. stk=[..,5]
                → dfs(6): disc[6]=low[6]=6. push 6. stk=[..,6]
                   neighbor 4: visited & onStack → low[6]=min(6, disc[4]=4) = 4
                   neighbor 7: unvisited → dfs(7): disc[7]=low[7]=7. push 7. stk=[..,7]
                      no neighbors.
                      low[7](7) == disc[7](7) → ROOT. Pop until 7: SCC#0={7}. compCount→1
                      stk = [0,1,2,3,4,5,6]
                   back in 6: low[6]=min(low[6]=4, low[7]=7) = 4
                   low[6](4) == disc[6](6)? NO — not root.
             back in 5: low[5]=min(low[5]=5, low[6]=4) = 4
             low[5](4) == disc[5](5)? NO.
          back in 4: low[4]=min(low[4]=4, low[5]=4) = 4
          low[4](4) == disc[4](4)? YES — ROOT.
             Pop until 4: pop 6, pop 5, pop 4 → SCC#1={6,5,4}={4,5,6}. compCount→2
             stk = [0,1,2,3]
       back in 3: low[3]=min(low[3]=3, low[4]=4) = 3
       low[3](3) == disc[3](3)? YES — ROOT.
          Pop until 3: pop 3 → SCC#2={3}. compCount→3
          stk = [0,1,2]
    back in 2: low[2]=min(low[2]=0, low[3]=3) = 0
    low[2](0) == disc[2](2)? NO.
  back in 1: low[1]=min(low[1]=1, low[2]=0) = 0
  low[1](0) == disc[1](1)? NO.
back in 0: low[0]=min(low[0]=0, low[1]=0) = 0
low[0](0) == disc[0](0)? YES — ROOT.
   Pop until 0: pop 2, pop 1, pop 0 → SCC#3={2,1,0}={0,1,2}. compCount→4
   stk = []

Final comp[] = [3,3,3,2,1,1,1,0]
SCCs discovered in order: {7}(comp0), {4,5,6}(comp1), {3}(comp2), {0,1,2}(comp3)
```

Notice the completion order is `{7} → {4,5,6} → {3} → {0,1,2}` — exactly the **reverse** of Kosaraju's discovery order. This is expected: Tarjan pops sinks first (deepest closed subtree), Kosaraju's second pass discovers sources first.

---

## SECTION 5 — TEMPLATE 3: SCC CONDENSATION INTO A DAG

```cpp
// ─────────────────────────────────────────────────────────────────
// Build the condensation graph: one node per SCC, edge between
// SCCs A→B if any original edge crosses from a vertex in A to a
// vertex in B (A != B). Deduplicate parallel condensation edges.
// ─────────────────────────────────────────────────────────────────

vector<vector<int>> buildCondensation(int n, vector<vector<int>>& adj,
                                       vector<int>& comp, int numSCC) {
    vector<set<int>> dedupe(numSCC);
    for (int u = 0; u < n; u++) {
        for (int v : adj[u]) {
            if (comp[u] != comp[v]) {
                dedupe[comp[u]].insert(comp[v]);
            }
        }
    }
    vector<vector<int>> dag(numSCC);
    for (int c = 0; c < numSCC; c++)
        dag[c].assign(dedupe[c].begin(), dedupe[c].end());
    return dag;
}
```

**Applied to the example graph** with Kosaraju's numbering (comp 0={0,1,2}, comp 1={3}, comp 2={4,5,6}, comp 3={7}):

```
Original edges crossing SCC boundaries: 2→3 (comp0→comp1), 3→4 (comp1→comp2), 6→7 (comp2→comp3)

Condensation DAG:
  0: [1]
  1: [2]
  2: [3]
  3: []

This is a simple chain — a valid DAG, exactly as guaranteed by theory.
Once you have this DAG, ALL of Pattern 30 (topological sort, DAG DP,
longest/shortest path in a DAG) applies directly on top of it.
```

**Why this is the single most useful trick in this pattern:** any directed graph problem that says "find the answer considering cycles" almost always means: (1) find SCCs, (2) collapse each SCC to a super-node carrying an aggregated weight (e.g., sum of node values inside it), (3) solve the now-acyclic problem with DAG DP. This shows up constantly in CF (e.g. "maximum value path allowing you to revisit nodes within a cycle for free").

---

## SECTION 6 — TEMPLATE 4: TARJAN'S BRIDGES (CRITICAL CONNECTIONS, LC 1192)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1192 — Critical Connections in a Network
// n servers, `connections` = undirected edges. Find all bridges
// (edges whose removal disconnects the network).
//
// Same low-link machinery as Tarjan SCC, but on an UNDIRECTED graph:
//   disc[u] = discovery time
//   low[u]  = smallest disc[] reachable from u's subtree using tree
//             edges + at most one back edge (NOT through the parent edge)
//
// BRIDGE CONDITION: edge (u, v) where v is a DFS-child of u is a
// bridge IFF   low[v] > disc[u]   (STRICT inequality).
//
// WHY strict >, not >=: if low[v] == disc[u], there IS a back edge
// from v's subtree reaching EXACTLY u — meaning v's subtree can get
// back to u through a path OTHER than the tree edge (u,v). That
// means removing (u,v) does NOT disconnect v's subtree from u — an
// alternate route exists. Only if low[v] > disc[u] (v's subtree
// cannot reach u or anything above u at all) is (u,v) the ONLY link.
//
// PARENT-EDGE HANDLING: track the EDGE ID you arrived on, not just
// the parent VERTEX. If there are two parallel edges between u and
// its parent, skipping by vertex would incorrectly ignore the
// second parallel edge (which DOES provide an alternate path, so
// neither parallel edge is a bridge). Skipping by edge id fixes this.
// ─────────────────────────────────────────────────────────────────

class Bridges {
public:
    int n, timer = 0;
    vector<vector<pair<int,int>>> adj;   // {neighbor, edgeId}
    vector<int> disc, low;
    vector<bool> visited;
    vector<vector<int>> bridges;

    Bridges(int n) : n(n), adj(n), disc(n, -1), low(n, -1), visited(n, false) {}

    void addEdge(int u, int v, int id) {
        adj[u].push_back({v, id});
        adj[v].push_back({u, id});
    }

    void dfs(int u, int parentEdgeId) {
        visited[u] = true;
        disc[u] = low[u] = timer++;
        for (auto& [v, edgeId] : adj[u]) {
            if (edgeId == parentEdgeId) continue;   // skip only the exact parent EDGE
            if (!visited[v]) {
                dfs(v, edgeId);
                low[u] = min(low[u], low[v]);
                if (low[v] > disc[u]) {              // STRICT >
                    bridges.push_back({u, v});
                }
            } else {
                low[u] = min(low[u], disc[v]);        // back edge
            }
        }
    }

    vector<vector<int>> criticalConnections(int nIn, vector<vector<int>>& connections) {
        for (int id = 0; id < (int)connections.size(); id++)
            addEdge(connections[id][0], connections[id][1], id);
        for (int i = 0; i < n; i++)
            if (!visited[i]) dfs(i, -1);
        return bridges;
    }
};
```

**FULL TRACE.** `n=4`, `connections = [[0,1],[1,2],[2,0],[1,3]]` (a triangle 0-1-2 plus a pendant edge 1-3). Edge ids: `0:(0,1)`, `1:(1,2)`, `2:(2,0)`, `3:(1,3)`. Expected: only `(1,3)` is a bridge (the triangle edges all have alternate paths).

```
adj[0] = [{1,0}, {2,2}]
adj[1] = [{0,0}, {2,1}, {3,3}]
adj[2] = [{1,1}, {0,2}]
adj[3] = [{1,3}]

dfs(0, parentEdgeId=-1): disc[0]=low[0]=0
  {1,0}: not parent edge, unvisited → dfs(1, parentEdgeId=0): disc[1]=low[1]=1
    {0,0}: edgeId 0 == parentEdgeId(0) → SKIP (parent edge)
    {2,1}: not parent edge, unvisited → dfs(2, parentEdgeId=1): disc[2]=low[2]=2
      {1,1}: edgeId 1 == parentEdgeId(1) → SKIP
      {0,2}: not parent edge, visited(0) → back edge: low[2]=min(2, disc[0]=0)=0
    back in 1: low[1]=min(low[1]=1, low[2]=0)=0
      bridge check: low[2](0) > disc[1](1)? NO → (1,2) NOT a bridge ✓
    {3,3}: not parent edge, unvisited → dfs(3, parentEdgeId=3): disc[3]=low[3]=3
      {1,3}: edgeId 3 == parentEdgeId(3) → SKIP
    back in 1: low[1]=min(low[1]=0, low[3]=3)=0
      bridge check: low[3](3) > disc[1](1)? YES → (1,3) IS a bridge ✓
  back in 0: low[0]=min(low[0]=0, low[1]=0)=0
    bridge check: low[1](0) > disc[0](0)? NO → (0,1) NOT a bridge ✓
  {2,2}: not parent edge, visited(2) → back edge: low[0]=min(low[0]=0, disc[2]=2)=0

Final bridges = [(1,3)]  ✓ matches expectation exactly.
```

---

## SECTION 7 — TEMPLATE 5: TARJAN'S ARTICULATION POINTS

```cpp
// ─────────────────────────────────────────────────────────────────
// Articulation points via the same low-link machinery.
//
// NON-ROOT rule: vertex u (with DFS-child v) is an articulation
// point IFF   low[v] >= disc[u]   (NON-STRICT — the key difference
// from bridges!).
//
// WHY non-strict here but strict for bridges: an articulation point
// only requires that v's subtree CANNOT get back to a point ABOVE u
// — reaching u itself does not help, because removing u still cuts
// v's subtree off (u is exactly the vertex being removed). For
// bridges, reaching u itself DOES provide an alternate path around
// the edge (since the edge, not the vertex, is what's removed).
//
// ROOT special case: the root of a DFS tree has no parent, so the
// low[v] >= disc[u] rule doesn't apply to it (disc[root]=0 always
// makes it trivially "true" but is meaningless). Instead: the root
// is an articulation point IFF it has 2 OR MORE children in the
// DFS tree. Two children means removing the root leaves those two
// subtrees permanently disconnected from each other.
// ─────────────────────────────────────────────────────────────────

class ArticulationPoints {
public:
    int n, timer = 0;
    vector<vector<int>> adj;
    vector<int> disc, low;
    vector<bool> visited, isArt;

    ArticulationPoints(int n)
        : n(n), adj(n), disc(n, -1), low(n, -1), visited(n, false), isArt(n, false) {}

    void addEdge(int u, int v) {
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    // rootChildren is written back for the caller to apply the root rule.
    void dfs(int u, int parent, int& rootChildren, bool isRoot) {
        visited[u] = true;
        disc[u] = low[u] = timer++;
        int children = 0;

        for (int v : adj[u]) {
            if (v == parent) continue;   // simple graph assumed; for multigraphs, use edge ids
            if (!visited[v]) {
                children++;
                int dummy = 0;
                dfs(v, u, dummy, false);
                low[u] = min(low[u], low[v]);

                if (!isRoot && low[v] >= disc[u]) {   // NON-STRICT
                    isArt[u] = true;
                }
            } else {
                low[u] = min(low[u], disc[v]);         // back edge
            }
        }

        if (isRoot) rootChildren = children;
    }

    vector<int> findArticulationPoints() {
        for (int i = 0; i < n; i++) {
            if (!visited[i]) {
                int rootChildren = 0;
                dfs(i, -1, rootChildren, true);
                if (rootChildren > 1) isArt[i] = true;   // root special case
            }
        }
        vector<int> result;
        for (int i = 0; i < n; i++)
            if (isArt[i]) result.push_back(i);
        return result;
    }
};
```

**FULL TRACE.** Undirected edges: `0-1, 1-2, 2-0` (triangle) plus `1-3, 3-4` (a tail). Expected articulation points: `{1, 3}` (removing 1 disconnects {3,4} from {0,2}; removing 3 disconnects {4}; removing 0 or 2 disconnects nothing — the triangle still holds via the other edge).

```
dfs(0, parent=-1, isRoot=true): disc[0]=low[0]=0, children=0
  neighbor 1: != parent, unvisited → children=1
    dfs(1, parent=0, isRoot=false): disc[1]=low[1]=1
      neighbor 0: == parent → SKIP
      neighbor 2: unvisited →
        dfs(2, parent=1, isRoot=false): disc[2]=low[2]=2
          neighbor 0: != parent(1), visited → back edge: low[2]=min(2, disc[0]=0)=0
          neighbor 1: == parent → SKIP
        back in 1: low[1]=min(low[1]=1, low[2]=0)=0
        check: low[2](0) >= disc[1](1)? NO → no articulation flag from this child
      neighbor 3: unvisited →
        dfs(3, parent=1, isRoot=false): disc[3]=low[3]=3
          neighbor 1: == parent → SKIP
          neighbor 4: unvisited →
            dfs(4, parent=3, isRoot=false): disc[4]=low[4]=4
              neighbor 3: == parent → SKIP
            back in 3: low[3]=min(low[3]=3, low[4]=4)=3
            check: low[4](4) >= disc[3](3)? YES → isArt[3]=true ✓
        back in 1: low[1]=min(low[1]=0, low[3]=3)=0
        check: low[3](3) >= disc[1](1)? YES → isArt[1]=true ✓
  back in 0: low[0]=min(low[0]=0, low[1]=0)=0
  neighbor 2: != parent(-1), visited → back edge: low[0]=min(low[0]=0, disc[2]=2)=0

rootChildren for 0 = 1 (only neighbor 1 was unvisited; neighbor 2 was already visited
                         via 1's subtree by the time 0's loop reaches it)
Root rule: rootChildren(1) > 1? NO → 0 is NOT an articulation point ✓

Final articulation points = {1, 3}  ✓ matches expectation exactly.
```

---

## SECTION 8 — TEMPLATE 6: 2-SAT VIA SCC

```cpp
// ─────────────────────────────────────────────────────────────────
// 2-SAT: n boolean variables x_0..x_{n-1}, clauses of the form
// (lit_a OR lit_b). Determine satisfiability and produce a valid
// assignment if one exists.
//
// LITERAL ENCODING: represent each variable with 2 graph nodes.
//   node 2*i     = literal x_i (true form)
//   node 2*i + 1 = literal ¬x_i (negated form)
//   negate(lit) = lit ^ 1   (flips between the pair)
//
// IMPLICATION GRAPH CONSTRUCTION: a clause (a OR b) is logically
// equivalent to BOTH implications:
//   ¬a → b   (if a is false, b MUST be true to satisfy the clause)
//   ¬b → a   (symmetric)
// Add both directed edges to the implication graph for EVERY clause.
// A "unit clause" forcing literal `lit` to be true is just (lit OR lit),
// which reduces to a single edge ¬lit → lit.
//
// SOLVING: run Tarjan SCC on the 2n-node implication graph.
//   - If x_i and ¬x_i end up in the SAME SCC, the instance is
//     UNSATISFIABLE (this literally means "x_i implies ¬x_i AND
//     ¬x_i implies x_i" — a direct logical contradiction).
//   - Otherwise, SATISFIABLE. Assignment rule (using TARJAN's
//     completion-order numbering, where comp 0 = sink-most SCC):
//         x_i = true   IFF   comp[2*i] < comp[2*i + 1]
//     Intuition: a smaller Tarjan comp id means "closer to a sink of
//     the condensation DAG" = topologically LATER. Implications flow
//     forward (earlier → later), so setting the topologically LATER
//     literal of the pair to true never violates an implication that
//     flows INTO it from something earlier. (If you build comp ids
//     via KOSARAJU instead — where comp 0 = source — the rule flips
//     to x_i = true IFF comp[2*i] > comp[2*i+1]. Always double-check
//     which numbering convention your SCC routine produces!)
// ─────────────────────────────────────────────────────────────────

class TwoSAT {
public:
    int n;                          // number of variables
    vector<vector<int>> adj;        // implication graph, 2n nodes
    vector<int> disc, low, comp;
    vector<bool> onStack;
    stack<int> stk;
    int timer = 0, compCount = 0;

    TwoSAT(int n) : n(n), adj(2 * n) {}

    int negate(int lit) { return lit ^ 1; }

    // clause (litA OR litB)
    void addClause(int litA, int litB) {
        adj[negate(litA)].push_back(litB);
        adj[negate(litB)].push_back(litA);
    }

    void addUnit(int lit) { addClause(lit, lit); }   // forces lit = true

    void dfs(int u) {
        disc[u] = low[u] = timer++;
        stk.push(u);
        onStack[u] = true;
        for (int v : adj[u]) {
            if (disc[v] == -1) {
                dfs(v);
                low[u] = min(low[u], low[v]);
            } else if (onStack[v]) {
                low[u] = min(low[u], disc[v]);
            }
        }
        if (low[u] == disc[u]) {
            while (true) {
                int v = stk.top(); stk.pop();
                onStack[v] = false;
                comp[v] = compCount;
                if (v == u) break;
            }
            compCount++;
        }
    }

    // returns {satisfiable?, assignment[i] = truth value of x_i}
    pair<bool, vector<bool>> solve() {
        int N = 2 * n;
        disc.assign(N, -1); low.assign(N, -1);
        comp.assign(N, -1); onStack.assign(N, false);

        for (int i = 0; i < N; i++)
            if (disc[i] == -1) dfs(i);

        vector<bool> assignment(n);
        for (int i = 0; i < n; i++) {
            if (comp[2*i] == comp[2*i + 1]) return {false, {}};   // UNSAT
            assignment[i] = comp[2*i] < comp[2*i + 1];
        }
        return {true, assignment};
    }
};
```

**FULL TRACE.** 2 variables `x0, x1`. Clauses: `(x0 OR x1)`, `(¬x0 OR ¬x1)` (not both true), and a unit clause forcing `x0` true. Expect: `x0 = true`, `x1 = false`.

```
Literal nodes: x0=0, ¬x0=1, x1=2, ¬x1=3

addClause(x0=0, x1=2):     adj[negate(0)=1].push(2)   → edge 1→2
                            adj[negate(2)=3].push(0)   → edge 3→0
addClause(¬x0=1, ¬x1=3):   adj[negate(1)=0].push(3)   → edge 0→3
                            adj[negate(3)=2].push(1)   → edge 2→1
addUnit(x0=0) = addClause(0,0): adj[negate(0)=1].push(0) → edge 1→0 (added twice, harmless)

Implication graph:
  0: [3]
  1: [2, 0]
  2: [1]
  3: [0]

Run Tarjan starting at node 0:
  dfs(0): disc[0]=low[0]=0. push 0.
    neighbor 3: unvisited → dfs(3): disc[3]=low[3]=1. push 3.
       neighbor 0: visited & onStack → low[3]=min(1, disc[0]=0)=0
       low[3](0) == disc[3](1)? NO.
    back in 0: low[0]=min(low[0]=0, low[3]=0)=0
    low[0](0) == disc[0](0)? YES — ROOT.
       pop 3, pop 0 → SCC#0 = {3, 0}. compCount → 1

  next unvisited: node 1 → dfs(1): disc[1]=low[1]=2. push 1.
    neighbor 2: unvisited → dfs(2): disc[2]=low[2]=3. push 2.
       neighbor 1: visited & onStack → low[2]=min(3, disc[1]=2)=2
       low[2](2) == disc[2](3)? NO.
    back in 1: low[1]=min(low[1]=2, low[2]=2)=2
    neighbor 0: visited but NOT onStack (already popped in SCC#0) → IGNORE
    low[1](2) == disc[1](2)? YES — ROOT.
       pop 2, pop 1 → SCC#1 = {2, 1}. compCount → 2

comp[] : comp[0]=0, comp[1]=1, comp[2]=1, comp[3]=0

Check UNSAT: comp[x0]=comp[0]=0 vs comp[¬x0]=comp[1]=1 → different, OK.
             comp[x1]=comp[2]=1 vs comp[¬x1]=comp[3]=0 → different, OK.
             SATISFIABLE.

Assignment:
  x0 = true IFF comp[0] < comp[1]  →  0 < 1  → TRUE   ✓ (unit clause satisfied)
  x1 = true IFF comp[2] < comp[3]  →  1 < 0  → FALSE  ✓ (not-both-true clause satisfied,
                                                          and x0 OR x1 satisfied since x0=true)
```

---

## SECTION 9 — COMPLEXITY TABLE

| Algorithm | Time | Space | Passes | Notes |
|---|---|---|---|---|
| Kosaraju's SCC | O(V + E) | O(V + E) | 2 DFS + transpose build | Simple to explain, extra graph |
| Tarjan's SCC | O(V + E) | O(V + E) | 1 DFS | Single pass, low-link based |
| SCC Condensation | O(V + E) | O(V + E) | — | Dedup with `set` costs O(E log E); use hashing for O(E) |
| Tarjan's Bridges | O(V + E) | O(V + E) | 1 DFS | Same machinery, `low[v] > disc[u]` |
| Tarjan's Articulation Points | O(V + E) | O(V + E) | 1 DFS | `low[v] >= disc[u]` + root special case |
| 2-SAT (via Tarjan) | O(V + E) where V=2n, E=2×clauses | O(V + E) | 1 DFS | Linear in variables + clauses |

All of these are strictly linear in graph size — the entire value of this pattern is packing rich structural information (cycles, cut points, satisfiability) into a single linear-time traversal.

---

## SECTION 10 — VARIANTS

- **Counting SCCs of size > 1 (nontrivial SCCs):** After computing `comp[]`, bucket vertices by component and count buckets with size ≥ 2. Useful for "does this graph contain a cycle reachable from X" type questions.
- **SCC + DAG DP for "best path allowing free cycle traversal":** Collapse SCCs, aggregate node weights within each SCC (sum, max, etc.), then run longest/shortest path DP on the condensation DAG (Pattern 30). Classic CF pattern: "you may revisit any node inside a strongly connected group for free."
- **Edge-biconnected components (via bridges):** Remove all bridges from the graph; the remaining connected components are edge-biconnected components (any single edge removal inside one keeps it connected). Build a "bridge tree" (contract each component to a node) — this bridge tree is always a TREE, useful for LCA-style queries about "minimum bridges to cross between u and v."
- **Vertex-biconnected components (via articulation points):** Similar contraction using articulation points instead of bridges — more subtle because a single articulation point can belong to multiple biconnected components simultaneously (unlike bridges, which cleanly partition edges).
- **2-SAT with "at most one of k" constraints:** Encode via a chain of implications or a binary counter structure rather than O(k²) pairwise `(¬a ∨ ¬b)` clauses, to stay within O(V+E) instead of blowing up quadratically.
- **Online/incremental SCC:** Not covered here — algorithms like this exist for edge insertions but are well beyond 2000-rating scope; know that Tarjan/Kosaraju assume a static graph.
- **Eulerian circuit existence checks** sometimes combine with SCC (a directed graph has an Eulerian circuit iff it's connected as one SCC when ignoring isolated vertices, and in-degree == out-degree for every vertex).

---

## SECTION 11 — COMMON MISTAKES

### Mistake 1: Bridge condition — using `>=` instead of strict `>`

```cpp
// WRONG
if (low[v] >= disc[u]) bridges.push_back({u, v});
// This is the ARTICULATION POINT condition, not the bridge condition!
// If low[v] == disc[u], there IS an alternate path from v's subtree
// back to u (a back edge landing exactly on u) — the edge (u,v) is
// NOT the only connection, so it's NOT a bridge.

// CORRECT
if (low[v] > disc[u]) bridges.push_back({u, v});
// Strict: v's subtree cannot reach u or anything above u at all.
```

### Mistake 2: Articulation point — forgetting the root special case

```cpp
// WRONG — applying the non-root rule to the root too
void dfs(int u, int parent) {
    ...
    if (low[v] >= disc[u]) isArt[u] = true;   // fires even when u is the root!
}
// disc[root] = 0 always, and low[v] >= 0 is ALWAYS true for any child v,
// so this would incorrectly mark EVERY root with at least one child as
// an articulation point — even isolated leaf-only trees.

// CORRECT — track children count separately for the root, apply the
// "children >= 2" rule ONLY for the root, and the low[v]>=disc[u] rule
// ONLY for non-root vertices (see Section 7's isRoot flag).
```

### Mistake 3: Tarjan SCC — forgetting the `onStack[]` check

```cpp
// WRONG — relaxing low[u] for ANY visited neighbor
if (disc[v] != -1) {
    low[u] = min(low[u], disc[v]);   // BUG: doesn't check onStack[v]
}
// If v was visited but already POPPED (its SCC is finalized), this edge
// points into an already-closed, unrelated SCC. Using disc[v] here can
// artificially shrink low[u], MERGING two SCCs that should stay separate
// — a silent correctness bug that produces too few, oversized SCCs.

// CORRECT
if (disc[v] == -1) {
    dfs(v);
    low[u] = min(low[u], low[v]);
} else if (onStack[v]) {              // only relax through ACTIVE ancestors
    low[u] = min(low[u], disc[v]);
}
```

### Mistake 4: Kosaraju — forgetting to build (or reversing incorrectly) the transpose

```cpp
// WRONG — running the second DFS on the SAME graph instead of the transpose
for (int i = n - 1; i >= 0; i--) {
    int u = order[i];
    if (!visited[u]) dfs2(u, adj, visited, comp, compId++);   // BUG: adj, not radj!
}
// Without the transpose, the second DFS just re-explores the same
// reachability as the first pass — you'll get ONE giant "SCC" per
// weakly-connected component instead of the true SCCs, because forward
// reachability alone doesn't require mutual reachability.

// CORRECT — build radj (every edge reversed) and DFS on radj in the
// second pass. This is precisely what turns "reachable" into "mutually
// reachable" (see Section 3).
```

### Mistake 5: Bridges — skipping by parent VERTEX instead of parent EDGE

```cpp
// WRONG — skip any edge back to the parent vertex
for (int v : adj[u]) {
    if (v == parent) continue;   // BUG for multigraphs
    ...
}
// If there are TWO parallel edges between u and its parent, this skips
// BOTH of them. But the second parallel edge is a legitimate alternate
// path back to the parent — it should count as a back edge and correctly
// prevent (u,parent) [via the FIRST edge] from being flagged a bridge.
// Skipping by vertex makes both parallel edges look like bridges when
// NEITHER is (removing one still leaves the other connecting them).

// CORRECT — track edge ids, skip only the exact edge you arrived on
// (see Section 6's Bridges class using {neighbor, edgeId} pairs).
```

### Mistake 6: 2-SAT — checking `x_i == ¬x_i` comp equality with the wrong node indices

```cpp
// WRONG — mixing up the encoding, e.g. checking adjacent indices
// incorrectly or using 1-indexed variables against 0-indexed nodes
if (comp[i] == comp[i + 1]) return false;   // BUG if i is odd — this
                                             // compares ¬x_k with x_{k+1},
                                             // two UNRELATED literals!

// CORRECT — with the 2*i / 2*i+1 encoding, always compare the pair
// for the SAME variable i:
if (comp[2*i] == comp[2*i + 1]) return false;   // x_i vs ¬x_i, same i
```

---

## SECTION 12 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes)

| # | Problem | Source | Key Skill |
|---|---------|--------|-----------|
| 1 | Critical Connections in a Network | LC 1192 | Bridges via low-link, `low[v] > disc[u]` |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | Source | Key Skill |
|---|---------|--------|-----------|
| 2 | Strongly Connected Components — Kosaraju implementation | GeeksforGeeks / CSES "Planets and Kingdoms" | Two-pass DFS, transpose graph, finish-time ordering |
| 3 | Strongly Connected Components — Tarjan implementation | CSES "Planets and Kingdoms" (Tarjan variant) | Single-pass low-link, on-stack discipline |

### STRETCH (solve in ≤ 40 minutes)

| # | Problem | Source | Key Skill |
|---|---------|--------|-----------|
| 4 | Minimum Number of Days to Disconnect Island | LC 1568 | Articulation-point intuition (is 0, 1, or 2 removals enough?) plus BFS/DFS connectivity checks |

### CONTEST (attempt under time pressure)

| # | Problem | Source | Key Skill |
|---|---------|--------|-----------|
| 5 | 2-SAT problems (various) | Codeforces (e.g. CF 468B-style, CSES "Giant Pizza") | Implication graph construction, SCC application, UNSAT detection |

---

## SECTION 13 — PATTERN CONNECTIONS

1. **Pattern 12 (Graphs / DFS fundamentals):** SCC, bridges, and articulation points are ALL built directly on top of the DFS traversal template from Pattern 12. If your DFS discipline (visited arrays, recursion stack awareness) is shaky, this entire pattern will be shaky. Master Pattern 12 first.

2. **Pattern 30 (Topological Sort):** SCC condensation always produces a DAG — and the whole point of condensing is to unlock Pattern 30's toolbox (topological sort, DAG longest/shortest path, DAG DP) on graphs that originally had cycles. Think of Pattern 32 as "the preprocessing step that makes Pattern 30 applicable to arbitrary directed graphs."

3. **Pattern 13 (Union-Find / DSU):** DSU finds CONNECTED components in undirected graphs in near-O(1) per operation, but has NO notion of edge direction — it cannot find SCCs (which require directional mutual reachability). Conversely, DSU is exactly the right tool for building bridge trees / vertex-biconnected component trees AFTER you've identified bridges/articulation points with Tarjan — contract non-bridge edges with DSU to get the final bridge-tree structure in near-linear time.

---

## SECTION 14 — INTERVIEW SIMULATIONS

*(Reminder: this whole pattern is rare in interviews outside senior/infra-focused roles. These simulations assume the interviewer explicitly signals graph-theory depth is in scope.)*

### Interview Simulation 1: LC 1192 — Critical Connections in a Network

**Interviewer:** "Given `n` servers and a list of undirected connections, find all connections that, if removed, would leave some server unable to reach some other server."

**Candidate:**

*[First minute — name the concept]*
> "This is exactly the 'find all bridges' problem in graph theory. A bridge is an edge whose removal disconnects the graph. I'll use Tarjan's bridge-finding algorithm — a single DFS pass tracking discovery time and low-link value per node."

*[State the core rule before coding]*
> "For a tree edge (u, v) where v is discovered as a DFS-child of u: it's a bridge if `low[v] > disc[u]` — strictly greater. That means nothing in v's subtree, even taking one back edge, can reach u or anything discovered before u. If it could reach exactly u, there'd be an alternate path around the edge, so it wouldn't be critical."

*[Code — ~6 minutes, using edge ids to be safe against multi-edges]*
```cpp
vector<vector<int>> criticalConnections(int n, vector<vector<int>>& connections) {
    vector<vector<pair<int,int>>> adj(n);
    for (int id = 0; id < (int)connections.size(); id++) {
        int u = connections[id][0], v = connections[id][1];
        adj[u].push_back({v, id});
        adj[v].push_back({u, id});
    }
    vector<int> disc(n, -1), low(n, -1);
    vector<vector<int>> bridges;
    int timer = 0;

    function<void(int,int)> dfs = [&](int u, int parentEdge) {
        disc[u] = low[u] = timer++;
        for (auto& [v, eid] : adj[u]) {
            if (eid == parentEdge) continue;
            if (disc[v] == -1) {
                dfs(v, eid);
                low[u] = min(low[u], low[v]);
                if (low[v] > disc[u]) bridges.push_back({u, v});
            } else {
                low[u] = min(low[u], disc[v]);
            }
        }
    };
    for (int i = 0; i < n; i++) if (disc[i] == -1) dfs(i, -1);
    return bridges;
}
```

*[Interviewer]: "Why did you track edge ids instead of just the parent vertex?"*
> "If the input can contain parallel edges between the same pair of servers, skipping by parent vertex would skip BOTH parallel edges, hiding the fact that a second edge provides an alternate route. Skipping by the exact edge id I arrived on avoids that bug. For LC 1192 specifically the constraints guarantee no duplicate connections, but I'd rather write it correctly by default."

**RED FLAGS:**
- Using `low[v] >= disc[u]` (confusing this with articulation points)
- Skipping by parent vertex without acknowledging the multigraph edge case
- Not recognizing this as a "find bridges" problem at all and attempting brute-force edge removal + BFS reachability check for each edge (O(E × (V+E)) — works but is a clear signal of not knowing the pattern)

---

### Interview Simulation 2: Conceptual — "How would you find clusters of mutually-dependent services?"

**Interviewer:** "We have a service dependency graph — service A 'depends on' service B is a directed edge. We suspect some services form circular dependency clusters. How would you find them?"

**Candidate:**

> "This is asking for strongly connected components — maximal groups of services that are all mutually reachable from each other via dependency edges, which is exactly what a circular dependency cluster looks like: A depends on B, B depends on C, C depends back on A.

> I'd run Tarjan's SCC algorithm — a single DFS pass in O(V+E) tracking a discovery time and a 'low-link' value per node, which is the earliest discovery time reachable from that node's subtree. When a node's low-link equals its own discovery time, everything currently pending above it on an explicit stack forms one SCC — I pop them off as a group.

> Any SCC with more than one service IS a circular dependency cluster. Once identified, I'd collapse each into a single node — the 'condensation' — which is guaranteed to be a DAG, and that DAG gives you a clean build/deploy order for the non-circular parts, with the clusters needing separate handling (e.g., flag for refactoring, or deploy together as a unit)."

*[Interviewer]: "What if I just want the total count of circular clusters, not the exact code?"*
> "Same algorithm, I just count how many SCCs have size ≥ 2 after grouping — no code change needed beyond a final tally over `comp[]` bucket sizes."

**RED FLAGS:**
- Proposing to just run cycle detection (DFS with a recursion-stack check) — that finds *whether* a cycle exists but not the *grouping* of all nodes into their mutual-reachability clusters, which is what "clusters" implies
- Confusing SCC (directed) with connected components (undirected) — the interviewer said "depends on," which is directional, and this distinction matters
- Not mentioning the O(V+E) complexity as the reason this beats an O(V²) or O(V³) all-pairs reachability approach

---

## SECTION 15 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why must the condensation of any directed graph's SCCs always be a DAG — never have a cycle among the super-nodes?**

> Suppose the condensation had a cycle: SCC_A → SCC_B → SCC_C → SCC_A. Then every vertex in A can reach every vertex in B (via the edge A→B and A's internal mutual reachability), B can reach C, and C can reach back to A. Chaining these, every vertex in A, B, and C can reach every other vertex among all three — meaning A, B, and C were never maximal separate SCCs to begin with; they should have been ONE SCC. This contradicts the premise that A, B, C were distinct (maximal) SCCs. Hence no cycle can exist in the condensation.

---

**Q2: What is the precise, structural reason the bridge condition uses strict `low[v] > disc[u]` while the articulation-point condition uses non-strict `low[v] >= disc[u]`?**

> For BRIDGES, we're asking "does removing this EDGE disconnect the subtree?" If `low[v] == disc[u]`, v's subtree has a back edge reaching exactly u — meaning there's a path from v's subtree back to u that does NOT use the tree edge (u,v). That's an alternate route around the edge, so the edge is not critical (strict > required).
>
> For ARTICULATION POINTS, we're asking "does removing this VERTEX disconnect the subtree?" If `low[v] == disc[u]`, the back edge reaches u itself — but u is exactly the vertex being removed! Reaching u doesn't help v's subtree escape, because u is gone. So even the equal case means v's subtree is cut off once u is removed (non-strict >= required).

---

**Q3: In Tarjan's algorithm, why is checking `onStack[v]` necessary when relaxing `low[u]`, instead of just checking `disc[v] != -1`?**

> A vertex being "visited" (`disc[v] != -1`) only means its DFS call has started at some point — it does NOT mean it's still part of an "open" SCC search. Once a vertex's SCC is finalized (its low[]==disc[] condition fired and it was popped), it is visited but no longer onStack — it belongs to an ALREADY CLOSED, separate SCC. An edge to such a vertex crosses INTO a different, earlier-finished SCC (this is exactly a condensation-DAG edge) and must not affect `low[u]`, or two genuinely separate SCCs would get incorrectly merged.

---

**Q4: Kosaraju assigns SCC ids in forward topological order (source = comp 0); Tarjan assigns them in reverse (sink = comp 0). Why are the two conventions opposite?**

> Kosaraju's second DFS pass starts from the vertex with the HIGHEST finish time from pass 1 — and that vertex is provably in a SOURCE SCC of the condensation (no incoming edges from any unvisited SCC). So the FIRST SCC Kosaraju discovers is a source, giving it id 0 — hence increasing ids move toward the sink.
>
> Tarjan, by contrast, uses ordinary DFS recursion: it only "closes" (roots) an SCC once it has fully explored everything reachable below it, which means the DEEPEST, most sink-like SCCs finish (and get id 0) FIRST — the vertex where the DFS actually started, which tends toward a source SCC, is the LAST to close. So Tarjan's ids increase toward the source — exactly reversed from Kosaraju.

---

**Q5: In the 2-SAT reduction, why does a clause `(a OR b)` translate to the implications `¬a → b` AND `¬b → a`, rather than just one of them?**

> A clause `(a OR b)` is satisfied as long as at least one of a, b is true. Logically, `(a OR b)` is equivalent to the single implication `¬a → b` ("if a is false, b must be true to keep the clause satisfied"). But by symmetry it is EQUALLY equivalent to `¬b → a`. Both must be added because the implication graph needs to correctly propagate constraints in BOTH directions — if only `¬a → b` were added, the graph wouldn't capture "if b is false, a must be true," and the SCC-based consistency check (and the resulting assignment) would miss real constraints, potentially declaring a genuinely unsatisfiable instance satisfiable.

---

**Q6: Give a concrete 2-clause example that is UNSATISFIABLE, and explain what happens to `comp[x_i]` and `comp[¬x_i]` in that case.**

> Take a single variable x with clauses `(x OR x)` [forces x true] and `(¬x OR ¬x)` [forces x false] — directly contradictory. Implication edges: from `(x OR x)`: `¬x → x`. From `(¬x OR ¬x)`: `x → ¬x`. Now we have BOTH `¬x → x` and `x → ¬x` — a 2-cycle between the two literal nodes. Tarjan (or Kosaraju) will find that `x` and `¬x` are mutually reachable and therefore land in the SAME SCC. The `comp[2*i] == comp[2*i+1]` check catches exactly this, correctly reporting UNSAT.

---

**Q7: Why can't you use plain Union-Find (DSU) to compute strongly connected components of a directed graph?**

> DSU merges sets based on an edge existing between two elements, with no notion of DIRECTION — it treats every union as symmetric ("u and v are in the same set now"). SCC membership requires MUTUAL reachability: u must be able to reach v AND v must be able to reach u via directed paths, which can involve long chains through many intermediate vertices, not just a direct edge. A single directed edge u→v does NOT imply u and v belong in the same SCC (v might not be able to get back to u at all) — so naively unioning on every directed edge produces wildly incorrect, oversized groupings (essentially just weakly-connected components, ignoring direction entirely).

---

## SECTION 16 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. KOSARAJU'S SCC (2-pass)
// ─────────────────────────────────────────────────────────────────
// Pass 1: DFS on G, push vertex to `order` on FINISH.
// Pass 2: DFS on transpose G^T, processing `order` from back to front.
// comp id 0 = topologically FIRST (source-most) SCC.

// ─────────────────────────────────────────────────────────────────
// 2. TARJAN'S SCC (1-pass, low-link)
// ─────────────────────────────────────────────────────────────────
disc[u] = low[u] = timer++;  stk.push(u);  onStack[u] = true;
for (int v : adj[u]) {
    if (disc[v] == -1) { dfs(v); low[u] = min(low[u], low[v]); }
    else if (onStack[v]) low[u] = min(low[u], disc[v]);
}
if (low[u] == disc[u]) {                    // pop SCC
    while (true) { int v = stk.top(); stk.pop(); onStack[v] = false;
                   comp[v] = compCount; if (v == u) break; }
    compCount++;
}
// comp id 0 = REVERSE topological order (sink-most SCC first).

// ─────────────────────────────────────────────────────────────────
// 3. TARJAN'S BRIDGES (undirected, edge-id parent skip)
// ─────────────────────────────────────────────────────────────────
for (auto& [v, eid] : adj[u]) {
    if (eid == parentEdge) continue;
    if (disc[v] == -1) {
        dfs(v, eid);
        low[u] = min(low[u], low[v]);
        if (low[v] > disc[u]) bridges.push_back({u, v});   // STRICT >
    } else low[u] = min(low[u], disc[v]);
}

// ─────────────────────────────────────────────────────────────────
// 4. TARJAN'S ARTICULATION POINTS (undirected)
// ─────────────────────────────────────────────────────────────────
// non-root:  low[v] >= disc[u]  → isArt[u] = true      (NON-STRICT)
// root:      children in DFS tree >= 2 → isArt[root] = true

// ─────────────────────────────────────────────────────────────────
// 5. SCC CONDENSATION → DAG
// ─────────────────────────────────────────────────────────────────
for (int u = 0; u < n; u++)
    for (int v : adj[u])
        if (comp[u] != comp[v]) dedupe[comp[u]].insert(comp[v]);
// Then run Pattern 30 (topo sort / DAG DP) on the resulting graph.

// ─────────────────────────────────────────────────────────────────
// 6. 2-SAT ENCODING
// ─────────────────────────────────────────────────────────────────
// literal x_i = node 2*i, literal ¬x_i = node 2*i+1, negate = lit ^ 1
// clause (a OR b): add edges  negate(a)->b  and  negate(b)->a
// UNSAT iff comp[2*i] == comp[2*i+1] for any i
// (Tarjan numbering) x_i = true  IFF  comp[2*i] < comp[2*i+1]

// ─────────────────────────────────────────────────────────────────
// 7. DECISION TABLE
// ─────────────────────────────────────────────────────────────────
// "mutual reachability groups in a DIRECTED graph"     → SCC (Kosaraju/Tarjan)
// "critical EDGE whose removal disconnects"            → Bridges  (low[v] >  disc[u])
// "critical VERTEX whose removal disconnects"          → Articulation Points (low[v] >= disc[u], + root rule)
// "binary constraint satisfaction (at most 2 literals)" → 2-SAT → implication graph → SCC
// "turn a cyclic directed graph into something DP-able" → SCC condensation → DAG DP (Pattern 30)
```

---

## SECTION 17 — WHAT TO DO AFTER THIS PATTERN

1. **Implement both Kosaraju AND Tarjan for the same graph and diff the `comp[]` arrays.** They will assign different NUMBERS to each component (opposite topological conventions), but the GROUPINGS (which vertices share a component) must be identical. This is the single best self-check for correctness.

2. **Solve LC 1192 twice** — once tracking edges by `{neighbor, edgeId}` pairs (robust to multigraphs) and once tracking by parent vertex only (works for LC 1192's constraints but is fragile). Understand exactly why the second version would break on a multigraph input.

3. **Derive the articulation-point root special case from first principles**, not from memory: draw a star graph (one center, 3 leaves) and manually verify the center is an articulation point (3 children) while a path graph's endpoints are not (root has only 1 child in that DFS orientation).

4. **Build a bridge tree.** After finding bridges with Tarjan, use DSU to union all vertices connected by NON-bridge edges. The resulting contracted graph (bridge edges only, connecting DSU-representative "super-nodes") is guaranteed to be a tree — verify this on a graph with at least 2 bridges and confirm |edges| = |super-nodes| - 1.

5. **Attempt one 2-SAT problem from Codeforces cold**, translating the problem statement's constraints into implication-graph edges yourself before looking at any editorial. This translation step — not the SCC code itself — is the actual skill being tested.

6. **Preview Pattern 30 (Topological Sort)** if you haven't done it yet: SCC condensation is the "unlock" that makes Pattern 30's DAG techniques apply to graphs that originally have cycles. The two patterns are meant to be chained.

---

## SECTION 18 — SIGN-OFF CRITERIA

### Tier 1 — SCC fundamentals understood
You can implement Kosaraju's algorithm from memory, including correctly building the transpose graph, and explain in one sentence why the second DFS pass must run on the transpose (not the original graph).

### Tier 2 — Tarjan mastered
You can implement Tarjan's SCC algorithm from memory, correctly including the `onStack[]` check, and can explain why omitting it silently merges unrelated SCCs. You can trace low-link value propagation on a graph with at least 8 vertices without referring to notes.

### Tier 3 — Bridges and articulation points solid
You can implement both Tarjan's bridges (`low[v] > disc[u]`, strict) and Tarjan's articulation points (`low[v] >= disc[u]`, non-strict, plus root special case) and can explain — without hedging — the precise structural reason the two conditions differ. You solve LC 1192 cleanly, handling multi-edges via edge ids.

### Tier 4 — 2-SAT and condensation complete
You can take a written constraint-satisfaction problem statement and translate it into a correct implication graph without looking up the `¬a → b, ¬b → a` rule. You can build an SCC condensation and chain it into a DAG DP solution (Pattern 30) for a problem involving cycles. You can solve at least one Codeforces 2-SAT problem end to end.

**Sign-off threshold:** Solve all 5 problems in the problem set. Mandatory: LC 1192 (bridges — the strict-vs-non-strict distinction must be explained without prompting), the Kosaraju SCC implementation (transpose-graph reasoning), and the Tarjan SCC implementation (on-stack discipline). These three represent the non-negotiable conceptual core; 2-SAT and articulation points build directly on top of them.

---

*Pattern 32 Complete — Strongly Connected Components*
*This concludes the graph-theory depth track. Combine with Pattern 30 (Topological Sort) and Pattern 12 (Graph Traversal) for full directed-graph mastery at the 2000+ level.*
