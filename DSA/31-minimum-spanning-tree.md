# PATTERN 31 — MINIMUM SPANNING TREE
## Kruskal, Prim, Cut/Cycle Properties, and the MST-as-a-Subroutine Family
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**What most people get wrong about MST:**

They memorize Kruskal's algorithm (sort edges, union-find, add if no cycle) as a rote procedure and never learn **why** it's correct, or **when** to reach for Prim instead. Then they hit a problem like "Find Critical and Pseudo-Critical Edges" (LC 1489) — which requires running MST *multiple times* with edges force-included or force-excluded — and they have no framework for it, because they never internalized MST as a **subroutine you can parameterize**, not just a one-shot algorithm.

**Kruskal vs Prim — the decision that actually matters:**

| | Kruskal | Prim |
|---|---|---|
| Core idea | Sort all edges, greedily add if no cycle | Grow one tree, always take the cheapest edge leaving it |
| Data structure | DSU (Union-Find) | Priority queue (min-heap) |
| Complexity | O(E log E) — dominated by the sort | O(E log V) — dominated by heap operations |
| Best for | **Sparse graphs** (E close to V), edge list already given | **Dense graphs** (E close to V²), adjacency list/matrix given |
| Natural extension | Cycle detection, "connect components" framing, works great when edges arrive as a flat list (LC 1135, 1584) | Natural when you're told "start from a source" or the graph is implicit (complete graph on points — LC 1584 with O(V²) instead of O(V² log V)) |

**Rule of thumb:** If E ≈ V² (dense, e.g., complete graph of points), Prim with an O(V²) array-based implementation beats Kruskal's O(V² log V) sort. If E ≈ V (sparse), Kruskal's sort is cheap and DSU makes correctness trivial to reason about — **default to Kruskal** unless told otherwise. In interviews, Kruskal is almost always the safer choice because it composes so cleanly with the DSU you already know from Pattern 13.

**The skill this pattern actually tests:**

1. Can you state and use the **cut property** and **cycle property** to justify *why* greedy works here (unlike most graph problems, where greedy fails)?
2. Can you treat MST as a black-box subroutine and call it multiple times with modifications — forced edge, excluded edge, virtual node, early termination at K components?
3. Can you recognize when a problem that *doesn't look like* MST (water distribution, "connect all points") is secretly asking for one?

---

## SECTION 2 — TAXONOMY + MST THEORY

### What is a Minimum Spanning Tree?

Given a connected, undirected, weighted graph G = (V, E), a **spanning tree** is a subgraph that:
- Includes all V vertices
- Is connected
- Has exactly V-1 edges (no cycles — it's a tree)

A **Minimum Spanning Tree (MST)** is a spanning tree whose total edge weight is minimized. If the graph is disconnected, no spanning tree exists (only a **minimum spanning forest**, one tree per component).

### The Cut Property (justifies Prim, and Kruskal's correctness)

**Statement:** For any cut of the graph (a partition of V into two non-empty disjoint sets S and V-S), if edge e is the **unique minimum-weight edge** crossing the cut, then e belongs to **every** MST. More generally (with ties allowed), there always exists an MST containing the minimum-weight crossing edge.

**Why (exchange argument):** Suppose an MST T does *not* contain e = (u, v), where u ∈ S, v ∈ V-S, and e is the minimum-weight edge crossing the cut. Since T is a spanning tree, there is a unique path in T from u to v. That path must cross the cut at least once via some edge f (it starts in S, ends in V-S). Since e is the minimum-weight crossing edge, weight(e) ≤ weight(f). Swap: remove f from T, add e. The result is still a spanning tree (removing f splits T into two pieces exactly matching S-side and (V-S)-side of the path; adding e reconnects them across the same cut), and its weight is ≤ weight(T). So either T wasn't minimal, or weight(e) == weight(f) and both are valid MSTs. Either way, **some** MST contains e. ∎

This is exactly what **Prim's algorithm** exploits: at every step, the "visited" set is S, the "unvisited" set is V-S, and Prim greedily takes the minimum-weight edge crossing that exact cut.

### The Cycle Property (justifies Kruskal)

**Statement:** For any cycle in the graph, the **maximum-weight edge** on that cycle does *not* belong to *some* MST (if weights are unique, it belongs to no MST at all).

**Why (exchange argument):** Suppose MST T contains the maximum-weight edge f of some cycle C. Removing f from T splits T into two components, S and V-S. Since f is on cycle C, and removing f from the cycle leaves a path, that path (the rest of C) must cross the S / (V-S) cut via some other edge e ∈ C, e ≠ f. Because f was the maximum-weight edge on C, weight(e) ≤ weight(f). Swap: add e instead of f. Result is still a spanning tree with weight ≤ weight(T). So T wasn't strictly better than the swapped tree — some MST excludes f. ∎

This is exactly what **Kruskal's algorithm** exploits: processing edges in increasing weight order and skipping any edge that connects two vertices already in the same component is *identical* to saying "never add the maximum edge of a cycle" — because by the time you'd consider that edge, its endpoints are already connected by strictly cheaper edges (that's what "same component" means).

### Why greedy works here but fails on most graph problems

Most greedy algorithms on graphs are wrong (e.g., greedy shortest-path-by-hops isn't Dijkstra). MST is one of the few problems where a **matroid structure** makes local greedy choices globally optimal — formally, the set of forests of a graph forms a **graphic matroid**, and Kruskal is exactly the generic "greedy algorithm on a matroid" applied to it. You don't need matroid theory for interviews — the cut/cycle property proofs above are sufficient and are what interviewers expect you to sketch.

### Taxonomy of MST-adjacent problems

| Type | Signal | Approach |
|---|---|---|
| **Direct MST** | "Connect all nodes with minimum cost", edge list given | Kruskal or Prim directly |
| **Implicit complete graph** | "Connect all points", distance = some formula (Manhattan, Euclidean) | Build all C(n,2) edges, then MST (Prim favored if n is large — O(n²) beats building+sorting O(n²) edges) |
| **Virtual node MST** | "Some nodes have an independent cost" (e.g., wells) | Add virtual node 0, connect it to every real node with its independent cost, MST over n+1 nodes |
| **Edge criticality** | "Which edges are essential / can be swapped" | Run MST N times: once as baseline, once per edge excluded, once per edge force-included |
| **Multi-actor connectivity** | "Two people need the graph connected using shared/private edges" | Two DSUs, greedily consume shared edges first (cycle property: shared edges are "more valuable", use them before private ones) |
| **MST-as-clustering** | "Split into K clusters minimizing max/min edge" | Run Kruskal but stop after V-K unions (equivalent to single-linkage clustering) |

---

## SECTION 3 — TEMPLATE 1: DSU (SELF-CONTAINED, PATH COMPRESSION + UNION BY SIZE)

Every Kruskal-based template in this document builds on this DSU. It's the same core idea as Pattern 13, restated here so this file is self-contained.

```cpp
// ─────────────────────────────────────────────────────────────────
// DSU — Disjoint Set Union with path compression + union by size
// Union by SIZE (not rank) is used here because several MST problems
// (LC 1168's virtual node, LC 1579's dual DSU) benefit from tracking
// component size directly (e.g., "is everything connected?" checks).
// ─────────────────────────────────────────────────────────────────
struct DSU {
    vector<int> parent, sz;
    int components;   // number of disjoint components remaining

    DSU(int n) : parent(n), sz(n, 1), components(n) {
        iota(parent.begin(), parent.end(), 0);  // parent[i] = i initially
    }

    int find(int x) {
        if (parent[x] != x)
            parent[x] = find(parent[x]);  // path compression: flatten to root
        return parent[x];
    }

    // Returns true if a merge happened (x, y were in different components).
    // Returns false if x, y were ALREADY connected — this is the signal
    // Kruskal uses to detect "adding this edge would form a cycle."
    bool unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;  // already same component — cycle if added

        // union by size: attach smaller tree under larger tree's root
        if (sz[rx] < sz[ry]) swap(rx, ry);
        parent[ry] = rx;
        sz[rx] += sz[ry];
        components--;
        return true;  // merge happened
    }

    bool connected(int x, int y) { return find(x) == find(y); }
};
```

**CRITICAL DETAIL — the return value of `unite()` IS the cycle check.** This is the single most common source of bugs in Kruskal implementations (see Common Mistakes, Section 12, Mistake 1). If `unite(u, v)` returns `false`, you must **not** add that edge to the MST — it would create a cycle.

**Complexity:** O(α(N)) amortized per operation, where α is the inverse Ackermann function — effectively O(1) for all practical N.

---

## SECTION 4 — TEMPLATE 2: KRUSKAL'S ALGORITHM (WITH FULL TRACE)

```cpp
// ─────────────────────────────────────────────────────────────────
// Kruskal's Algorithm — Minimum Spanning Tree
// Input: n vertices (0-indexed), edges as {weight, u, v}
// Output: total MST weight, or -1 if the graph is disconnected
//
// STEPS:
//   1. Sort all edges by weight, ascending.
//   2. For each edge (u, v, w), in order:
//        if find(u) != find(v):   // adding it does NOT create a cycle
//            unite(u, v)
//            add w to total, add edge to MST edge list
//   3. Stop early once V-1 edges have been added (optional optimization).
//   4. If fewer than V-1 edges were added, graph was disconnected.
//
// WHY sorting + greedy skip is correct: this is EXACTLY the cycle
// property (Section 2). By processing in increasing weight order and
// skipping same-component pairs, we never add the maximum-weight edge
// of any cycle — because by the time that edge is considered, its
// endpoints are already joined via strictly lighter edges.
// ─────────────────────────────────────────────────────────────────

struct Edge { int w, u, v; };

int kruskalMST(int n, vector<Edge>& edges, vector<Edge>* mstOut = nullptr) {
    sort(edges.begin(), edges.end(), [](const Edge& a, const Edge& b) {
        return a.w < b.w;
    });

    DSU dsu(n);
    int totalWeight = 0, edgesUsed = 0;

    for (const Edge& e : edges) {
        if (dsu.unite(e.u, e.v)) {          // false if same component => cycle
            totalWeight += e.w;
            edgesUsed++;
            if (mstOut) mstOut->push_back(e);
            if (edgesUsed == n - 1) break;  // MST is complete, stop early
        }
    }

    return (edgesUsed == n - 1) ? totalWeight : -1;  // -1 = disconnected
}
```

**FULL TRACE on a 5-vertex graph:**

```
Vertices: 0, 1, 2, 3, 4
Edges (u, v, w):
  (0,1,1)
  (1,3,2)
  (1,2,3)
  (0,3,4)
  (2,4,5)
  (3,4,6)

Step 1 — Sort by weight:
  (0,1,1), (1,3,2), (1,2,3), (0,3,4), (2,4,5), (3,4,6)

Step 2 — Process in order:

Edge (0,1,1): find(0)=0, find(1)=1 → different. unite(0,1).
  Components: {0,1}, {2}, {3}, {4}
  MST so far: [(0,1,1)]. Total weight = 1. Edges used = 1.

Edge (1,3,2): find(1)=0, find(3)=3 → different. unite(1,3).
  Components: {0,1,3}, {2}, {4}
  MST so far: [(0,1,1), (1,3,2)]. Total weight = 3. Edges used = 2.

Edge (1,2,3): find(1)=0, find(2)=2 → different. unite(1,2).
  Components: {0,1,2,3}, {4}
  MST so far: [(0,1,1), (1,3,2), (1,2,3)]. Total weight = 6. Edges used = 3.

Edge (0,3,4): find(0)=0, find(3)=0 → SAME component. SKIP (would form a cycle).
  MST unchanged.

Edge (2,4,5): find(2)=0, find(4)=4 → different. unite(2,4).
  Components: {0,1,2,3,4}
  MST so far: [(0,1,1), (1,3,2), (1,2,3), (2,4,5)]. Total weight = 11. Edges used = 4.
  edgesUsed == n-1 == 4 → STOP.

Answer: MST weight = 11
MST edges: (0,1), (1,3), (1,2), (2,4)
```

Note that edge (3,4,6) was never even examined — Kruskal stopped as soon as V-1 = 4 edges were added, which is a real speedup when the edge list is large but the graph is dense.

---

## SECTION 5 — TEMPLATE 3: PRIM'S ALGORITHM (LAZY DELETION, WITH FULL TRACE)

```cpp
// ─────────────────────────────────────────────────────────────────
// Prim's Algorithm — Minimum Spanning Tree (lazy-deletion priority queue)
// Input: n vertices, adjacency list adj[u] = list of {weight, neighbor}
// Output: total MST weight, or -1 if the graph is disconnected
//
// STEPS:
//   1. Start from any vertex (say 0). Mark visited.
//   2. Push all its edges into a min-heap keyed by weight.
//   3. Repeatedly pop the cheapest edge (w, u, v):
//        - if v already visited, DISCARD it (lazy deletion — this edge
//          is stale, it was pushed before v got added another way)
//        - else, mark v visited, add w to total, push v's edges
//   4. Repeat until heap empty or all n vertices visited.
//
// WHY this is correct: this is EXACTLY the cut property (Section 2).
// At every step, "visited" = S, "unvisited" = V-S. The min-heap lets
// us always extract the minimum-weight edge crossing that exact cut.
//
// "Lazy deletion" means we don't bother removing stale heap entries
// when a vertex gets visited via a cheaper edge — we just check
// visited[] when we POP and skip if already visited. Simpler than
// maintaining a decrease-key priority queue, costs an extra O(log E)
// per stale entry, which is asymptotically fine.
// ─────────────────────────────────────────────────────────────────

int primMST(int n, vector<vector<pair<int,int>>>& adj) {  // adj[u] = {(weight, v), ...}
    vector<bool> visited(n, false);
    // min-heap of (weight, vertex) — smallest weight on top
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;

    pq.push({0, 0});           // start at vertex 0 with "cost 0 to enter"
    int totalWeight = 0, verticesAdded = 0;

    while (!pq.empty() && verticesAdded < n) {
        auto [w, u] = pq.top(); pq.pop();

        if (visited[u]) continue;   // STALE entry — lazy deletion in action
        visited[u] = true;
        totalWeight += w;
        verticesAdded++;

        for (auto [nw, v] : adj[u]) {
            if (!visited[v]) pq.push({nw, v});  // candidate edge crossing the cut
        }
    }

    return (verticesAdded == n) ? totalWeight : -1;  // -1 = disconnected
}
```

**FULL TRACE on the same 5-vertex graph, starting at vertex 0:**

```
Adjacency list:
  0: (1,1), (4,3)
  1: (1,0), (3,2), (2,3)
  2: (3,1), (5,4)
  3: (4,0), (2,1), (6,4)
  4: (5,2), (6,3)

pq = [(0,0)]   visited = {}

Pop (0, 0): not visited. visited={0}. totalWeight=0. verticesAdded=1.
  Push edges from 0: (1,1), (4,3)
  pq = [(1,1), (4,3)]

Pop (1, 1): not visited. visited={0,1}. totalWeight=0+1=1. verticesAdded=2.
  Push edges from 1: (3,2)[to vertex 2], (2,3)[to vertex 3]
  pq = [(2,3), (3,2), (4,3)]     ← note (4,3) means weight 4, vertex 3 (stale-to-be)

Pop (2, 3): not visited. visited={0,1,3}. totalWeight=1+2=3. verticesAdded=3.
  Push edges from 3: (4,0)[0 visited, still pushed], (6,4)
  pq = [(3,2), (4,3)_stale, (4,0)_stale, (6,4)]

Pop (3, 2): not visited. visited={0,1,2,3}. totalWeight=3+3=6. verticesAdded=4.
  Push edges from 2: (5,4)
  pq = [(4,3)_stale, (4,0)_stale, (5,4), (6,4)]

Pop (4, 3): vertex 3 ALREADY visited → DISCARD (lazy deletion, stale entry).

Pop (4, 0): vertex 0 ALREADY visited → DISCARD (lazy deletion, stale entry).

Pop (5, 4): not visited. visited={0,1,2,3,4}. totalWeight=6+5=11. verticesAdded=5.
  verticesAdded == n == 5 → loop ends (heap empties naturally too).

Answer: MST weight = 11   ← matches Kruskal exactly, as it must (MST weight is unique)
```

**Complexity check:** Every edge is pushed at most once per endpoint, so at most O(E) heap pushes, each O(log E). Total: **O(E log E)**, commonly written **O(E log V)** since log E = O(log V²) = O(2 log V) = O(log V) for simple graphs.

---

## SECTION 6 — TEMPLATE 4: MIN COST TO CONNECT ALL POINTS (LC 1584)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1584 — Min Cost to Connect All Points
// Given n points on a 2D plane, connect all points with minimum total
// cost, where cost between two points = Manhattan distance |x1-x2|+|y1-y2|.
//
// KEY INSIGHT: This is an IMPLICIT complete graph — every pair of
// points has an edge. n can be up to 1000, so C(n,2) ≈ 500,000 edges.
//   - Kruskal: build all edges O(n²), sort O(n² log n) — fine but the
//     sort dominates and is wasteful given we usually stop at n-1 edges.
//   - Prim with O(n²) array-based (no heap!) version: for each of the
//     n-1 iterations, scan all unvisited points for the minimum "cost
//     to enter the tree" — this is O(n²) total, beating Kruskal's sort
//     for large dense n. THIS is the preferred solution for LC 1584.
// ─────────────────────────────────────────────────────────────────

int manhattan(vector<int>& a, vector<int>& b) {
    return abs(a[0]-b[0]) + abs(a[1]-b[1]);
}

// ---------- APPROACH A: Kruskal (build all edges, sort, DSU) ----------
int minCostConnectPoints_Kruskal(vector<vector<int>>& points) {
    int n = points.size();
    vector<Edge> edges;
    edges.reserve(n * (n - 1) / 2);

    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            edges.push_back({manhattan(points[i], points[j]), i, j});

    return kruskalMST(n, edges);  // reuse Template 2 verbatim
}

// ---------- APPROACH B: Prim, O(n²) array-based (NO heap) ----------
// Preferred for LC 1584 because the graph is dense (complete graph).
int minCostConnectPoints_Prim(vector<vector<int>>& points) {
    int n = points.size();
    vector<bool> inMST(n, false);
    vector<int> minEdge(n, INT_MAX);  // minEdge[v] = cheapest known cost to
                                       // connect v to the current tree
    minEdge[0] = 0;
    int totalWeight = 0;

    for (int iter = 0; iter < n; iter++) {
        // Find the unvisited vertex with the smallest minEdge value —
        // this IS the min-weight edge crossing the cut (cut property).
        int u = -1;
        for (int v = 0; v < n; v++) {
            if (!inMST[v] && (u == -1 || minEdge[v] < minEdge[u])) u = v;
        }

        inMST[u] = true;
        totalWeight += minEdge[u];

        // Relax: for every unvisited v, see if going through u is cheaper
        for (int v = 0; v < n; v++) {
            if (!inMST[v]) {
                int d = manhattan(points[u], points[v]);
                minEdge[v] = min(minEdge[v], d);
            }
        }
    }

    return totalWeight;
}
```

**TRACE for `points = [[0,0],[2,2],[3,10],[5,2],[7,0]]` (LeetCode's own example, expected answer 20), using Approach B:**

```
n=5. Points: P0=(0,0) P1=(2,2) P2=(3,10) P3=(5,2) P4=(7,0)

Init: minEdge = [0, INF, INF, INF, INF]. inMST=[F,F,F,F,F]

iter 0: pick u with min minEdge → u=0 (minEdge[0]=0). inMST[0]=T. total=0.
  Relax from 0: d(0,1)=4, d(0,2)=13, d(0,3)=7, d(0,4)=7
  minEdge = [0, 4, 13, 7, 7]

iter 1: min among unvisited {1,2,3,4} → minEdge[1]=4 is smallest. u=1. inMST[1]=T. total=0+4=4.
  Relax from 1: d(1,2)=9, d(1,3)=3, d(1,4)=7
  minEdge[2]=min(13,9)=9, minEdge[3]=min(7,3)=3, minEdge[4]=min(7,7)=7
  minEdge = [0, 4, 9, 3, 7]

iter 2: min among {2,3,4} → minEdge[3]=3 is smallest. u=3. inMST[3]=T. total=4+3=7.
  Relax from 3: d(3,2)=10, d(3,4)=4
  minEdge[2]=min(9,10)=9, minEdge[4]=min(7,4)=4
  minEdge = [0, 4, 9, 3, 4]

iter 3: min among {2,4} → minEdge[4]=4 is smallest. u=4. inMST[4]=T. total=7+4=11.
  Relax from 4: d(4,2)=11
  minEdge[2]=min(9,11)=9

iter 4: only {2} left → u=2. inMST[2]=T. total=11+9=20.

Answer: 20 ✓ matches LeetCode's expected output.
```

---

## SECTION 7 — TEMPLATE 5: OPTIMIZE WATER DISTRIBUTION (LC 1168)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1168 — Optimize Water Distribution in a Village
// n houses (1-indexed 1..n). wells[i] = cost to build a well directly
// at house i+1. pipes[k] = [house1, house2, cost] = cost to connect
// two houses with a pipe. Minimize total cost so every house has water
// (either its own well, or connected via pipes to a house with a well).
//
// KEY INSIGHT: "Build a well" is really "connect this house to a FREE
// water source." Model that free source as a VIRTUAL NODE 0. Then:
//   - wells[i] becomes an edge (0, i+1, wells[i])
//   - pipes stay as edges (house1, house2, cost)
// Now the problem is: find MST over n+1 nodes (0..n). Node 0 being in
// the MST just means "some house connects to the virtual source" —
// but EVERY house must eventually connect back to node 0 (directly or
// transitively) since node 0 must be part of the single spanning tree.
// This exactly captures "every house has water, minimum total cost."
// ─────────────────────────────────────────────────────────────────

int minCostToSupplyWater(int n, vector<int>& wells, vector<vector<int>>& pipes) {
    vector<Edge> edges;
    edges.reserve(n + pipes.size());

    // Virtual node 0 = "free water source". Edge (0, house, well_cost).
    for (int i = 0; i < n; i++) {
        edges.push_back({wells[i], 0, i + 1});
    }

    // Real pipe edges between houses.
    for (auto& p : pipes) {
        edges.push_back({p[2], p[0], p[1]});
    }

    // MST over n+1 nodes (0..n). Since node 0 is connected to
    // everything by construction, the graph (0 + houses) is always
    // connected via well edges even if no pipes exist — so this
    // never returns -1 in this problem's constraints.
    return kruskalMST(n + 1, edges);
}
```

**TRACE for `n=3, wells=[1,2,2], pipes=[[1,2,1],[2,3,1]]` (LeetCode example, expected answer 3):**

```
Virtual node 0 = free source.
Well edges: (0,1,1), (0,2,2), (0,3,2)
Pipe edges: (1,2,1), (2,3,1)

All edges as {w,u,v}: (1,0,1), (2,0,2), (2,0,3), (1,1,2), (1,2,3)

Sort by weight: (1,0,1), (1,1,2), (1,2,3), (2,0,2), (2,0,3)
  [two weight-1 edges tie; order between them doesn't affect correctness]

DSU init: {0},{1},{2},{3}

Edge (1,0,1) [well at house1, cost1]: find(0)≠find(1). unite(0,1).
  Components: {0,1},{2},{3}. total=1. used=1.

Edge (1,1,2) [pipe 1-2, cost1]: find(1)=0≠find(2)=2. unite(1,2).
  Components: {0,1,2},{3}. total=1+1=2. used=2.

Edge (1,2,3) [pipe 2-3, cost1]: find(2)=0≠find(3)=3. unite(2,3).
  Components: {0,1,2,3}. total=2+1=3. used=3.
  used == n (=3) → all n+1=4 nodes connected with n edges. STOP (n-1 edges needed
  for n+1 nodes = n edges — here used=3=n, correct).

Answer: 3 ✓
Interpretation: build a well at house 1 (cost 1), then pipe 1→2 (cost 1) and
pipe 2→3 (cost 1). Houses 2 and 3 get water via the pipe network, not their
own wells — even though wells[1]=2 and wells[2]=2 were individually more
expensive than piping.
```

**WHY this is NOT "just take the cheapest well for every house":** A house might be cheaper to reach via pipe from a neighbor's well than to dig its own — exactly the tension MST resolves globally instead of greedily per-house.

---

## SECTION 8 — TEMPLATE 6: FIND CRITICAL AND PSEUDO-CRITICAL EDGES (LC 1489)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1489 — Find Critical and Pseudo-Critical Edges in MST
// Given a weighted graph, classify every edge as:
//   CRITICAL: removing it INCREASES the MST weight (or disconnects
//             the graph). It is in EVERY possible MST.
//   PSEUDO-CRITICAL: not critical, but FORCING it into the MST still
//             achieves the same minimum weight. It's in SOME MSTs
//             but not all.
//   NEITHER: forcing it in increases the total weight above the true
//             MST weight — it can never appear in any MST.
//
// ALGORITHM (MST run 2E+1 times, using MST-as-subroutine):
//   1. Compute baseline MST weight W using all edges normally.
//   2. For each edge e:
//        a. EXCLUDE e entirely, recompute MST weight W_excl.
//           If W_excl > W (or graph becomes disconnected), e is CRITICAL
//           — every MST needs it, since removing it strictly hurts.
//        b. Otherwise, FORCE e into the MST first (pre-union its
//           endpoints in the DSU before running Kruskal on the rest),
//           recompute total weight W_force (= e's weight + MST of the
//           rest). If W_force == W, e is PSEUDO-CRITICAL — some MST
//           can use it without penalty.
//        c. Otherwise (not critical, and forcing costs more), e
//           belongs to NEITHER category.
//
// WHY this works: this is a direct application of the cycle/cut
// properties as a DECISION PROCEDURE — "is edge e's weight tied for
// minimum across every cut it could be the representative of?" is
// exactly what exclude/force testing answers empirically, without
// needing to reason about ties analytically.
// ─────────────────────────────────────────────────────────────────

// Returns MST weight; if skipIdx >= 0, that edge index is excluded entirely;
// if forceIdx >= 0, that edge is pre-unioned into the DSU before Kruskal runs.
// Returns -1 if the resulting graph cannot form a spanning tree.
int mstWeightWithConstraint(int n, vector<Edge>& sortedEdges,
                             int skipIdx, int forceIdx) {
    DSU dsu(n);
    int totalWeight = 0, edgesUsed = 0;

    if (forceIdx >= 0) {
        dsu.unite(sortedEdges[forceIdx].u, sortedEdges[forceIdx].v);
        totalWeight += sortedEdges[forceIdx].w;
        edgesUsed++;
    }

    for (int i = 0; i < (int)sortedEdges.size(); i++) {
        if (i == skipIdx || i == forceIdx) continue;  // excluded or already forced
        if (dsu.unite(sortedEdges[i].u, sortedEdges[i].v)) {
            totalWeight += sortedEdges[i].w;
            edgesUsed++;
        }
    }

    return (edgesUsed == n - 1) ? totalWeight : INT_MAX;  // INT_MAX = disconnected
}

vector<vector<int>> findCriticalAndPseudoCriticalEdges(
        int n, vector<vector<int>>& edgesIn) {
    int m = edgesIn.size();
    vector<Edge> edges(m);
    vector<int> origIndex(m);
    for (int i = 0; i < m; i++) {
        edges[i] = {edgesIn[i][2], edgesIn[i][0], edgesIn[i][1]};
        origIndex[i] = i;   // remember original LeetCode index for output
    }

    // Sort edges (and keep origIndex parallel) so DSU-force logic above works
    // on a fixed, stable ordering.
    vector<int> order(m);
    iota(order.begin(), order.end(), 0);
    sort(order.begin(), order.end(), [&](int a, int b) {
        return edges[a].w < edges[b].w;
    });
    vector<Edge> sortedEdges(m);
    vector<int> sortedOrig(m);
    for (int i = 0; i < m; i++) {
        sortedEdges[i] = edges[order[i]];
        sortedOrig[i] = origIndex[order[i]];
    }

    int baseline = mstWeightWithConstraint(n, sortedEdges, -1, -1);

    vector<int> critical, pseudo;
    for (int i = 0; i < m; i++) {
        int wExcl = mstWeightWithConstraint(n, sortedEdges, i, -1);
        if (wExcl > baseline) {          // includes the disconnected (INT_MAX) case
            critical.push_back(sortedOrig[i]);
            continue;
        }
        int wForce = mstWeightWithConstraint(n, sortedEdges, -1, i);
        if (wForce == baseline) {
            pseudo.push_back(sortedOrig[i]);
        }
        // else: neither critical nor pseudo-critical — excluded from both lists
    }

    return {critical, pseudo};
}
```

**WHY force-testing needs pre-union, not just "add e first":** Simply adding e's weight to the total isn't enough — e might connect two vertices that are *already* connected by cheaper edges processed earlier in a plain re-run, silently creating a cycle if you're not careful. Pre-unioning e's endpoints in the DSU *before* the main loop guarantees e is genuinely part of the resulting tree, and the main loop correctly skips any later edge that would now form a cycle through e.

**Mini trace intuition (not a full run — the algorithm calls MST O(E) times):** Given a triangle 0-1-2 with weights 1, 1, 2 (edges A=(0,1,1), B=(1,2,1), C=(0,2,2)): baseline MST = A+B = 2. Exclude A → MST must use B+C = 3 > 2 → **A is critical**. Exclude B → MST must use A+C = 3 > 2 → **B is critical**. Exclude C → MST still uses A+B = 2 = baseline → C is not critical; force C → C+min(A,B)=2+1=3 > baseline=2 → **C is neither**.

---

## SECTION 9 — TEMPLATE 7: REMOVE MAX NUMBER OF EDGES TO KEEP GRAPH FULLY TRAVERSABLE (LC 1579)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1579 — Remove Max Number of Edges to Keep Graph Fully Traversable
// Edges have type 1 (Alice-only), type 2 (Bob-only), or type 3 (both).
// Both Alice and Bob must be able to traverse the ENTIRE graph (every
// node reachable from every other, independently for each person).
// Find the max number of edges REMOVABLE while preserving that, or -1
// if it's impossible even with all edges kept.
//
// KEY INSIGHT: maximize edges REMOVED == minimize edges KEPT ==
// build a spanning structure for Alice's view and Bob's view using
// as FEW edges as possible. This is two independent MST-style
// "spanning forest with min edge count" problems — EXCEPT type-3
// edges are shared infrastructure, so use them for BOTH people
// before spending any type-1/type-2 edges. This greedy order is
// justified by the cycle property: a type-3 edge is "worth more"
// than a same-purpose type-1 or type-2 edge because it does double
// duty, so it should never be skipped in favor of a single-purpose
// edge — always prefer it when it doesn't create a cycle for EITHER
// person.
//
// ALGORITHM:
//   1. Two separate DSUs: dsuAlice, dsuBob, both over n nodes.
//   2. Process ALL type-3 edges FIRST. For each, try to unite in BOTH
//      DSUs. If it merges something in EITHER DSU, it's "used"
//      (kept). Count used edges.
//   3. Process type-1 edges: unite in dsuAlice only. Count if merged.
//   4. Process type-2 edges: unite in dsuBob only. Count if merged.
//   5. If dsuAlice.components == 1 AND dsuBob.components == 1, answer
//      = totalEdges - usedEdges. Else, answer = -1 (impossible).
// ─────────────────────────────────────────────────────────────────

int maxNumEdgesToRemove(int n, vector<vector<int>>& edges) {
    DSU dsuAlice(n + 1), dsuBob(n + 1);  // 1-indexed nodes; index 0 unused
    int usedEdges = 0;

    // Type 3 FIRST — shared edges are strictly more valuable (cycle
    // property: never skip a type-3 edge in favor of a type-1/2 edge
    // that could be replaced by it).
    for (auto& e : edges) {
        if (e[0] == 3) {
            bool a = dsuAlice.unite(e[1], e[2]);
            bool b = dsuBob.unite(e[1], e[2]);
            if (a || b) usedEdges++;   // kept if it helped EITHER person
        }
    }

    // Type 1 — Alice only
    for (auto& e : edges) {
        if (e[0] == 1) {
            if (dsuAlice.unite(e[1], e[2])) usedEdges++;
        }
    }

    // Type 2 — Bob only
    for (auto& e : edges) {
        if (e[0] == 2) {
            if (dsuBob.unite(e[1], e[2])) usedEdges++;
        }
    }

    if (dsuAlice.components != 2 || dsuBob.components != 2) {
        // components == 2 because node 0 is unused (its own singleton
        // component) plus the "real" 1..n forming a single component.
        return -1;
    }

    return (int)edges.size() - usedEdges;
}
```

**TRACE for `n=4, edges=[[3,1,2],[3,2,3],[1,1,3],[1,2,4],[1,1,2],[2,3,4]]` (LeetCode example, expected answer 2):**

```
n=4. dsuAlice, dsuBob both over nodes 0..4 (0 unused), components start at 5.

--- Type 3 edges first ---
[3,1,2]: uniteAlice(1,2)=true, uniteBob(1,2)=true → used. usedEdges=1.
  dsuAlice components: {0},{1,2},{3},{4} → 4 left (started 5, -1)
  dsuBob   components: {0},{1,2},{3},{4} → 4 left

[3,2,3]: uniteAlice(2,3)=true (2's root ≠ 3's root), uniteBob(2,3)=true → used. usedEdges=2.
  dsuAlice: {0},{1,2,3},{4} → 3 left
  dsuBob:   {0},{1,2,3},{4} → 3 left

--- Type 1 edges (Alice only) ---
[1,1,3]: find(1)=find(3) in dsuAlice already (both in {1,2,3}) → unite returns false. NOT used.

[1,2,4]: find(2)≠find(4) in dsuAlice → unite(2,4)=true → used. usedEdges=3.
  dsuAlice: {0},{1,2,3,4} → 2 left (this IS 1 real component + node 0's own)

--- Type 2 edges (Bob only) ---
[2,3,4]: find(3)≠find(4) in dsuBob → unite(3,4)=true → used. usedEdges=4.
  dsuBob: {0},{1,2,3,4} → 2 left

Final check: dsuAlice.components == 2 ✓ (means nodes 1-4 fully connected)
             dsuBob.components == 2 ✓

Answer: totalEdges(6) - usedEdges(4) = 2 ✓ matches LeetCode's expected output.
```

**WHY type-3-first is not just an optimization but a CORRECTNESS requirement:** if you process type-1/type-2 edges before type-3, you might "waste" a type-1 edge connecting two components that a later type-3 edge would have connected anyway for both people simultaneously — leaving you needing an extra type-2 edge you didn't actually need. Processing shared edges first always weakly dominates any other order (this is the cycle property again: a type-3 edge is never worse than a type-1/2 edge for the same merge, so greedily prioritizing it can't hurt and can only help).

---

## SECTION 10 — COMPLEXITY TABLE

| Algorithm / Problem | Time | Space | Notes |
|---|---|---|---|
| DSU (find + unite) | O(α(N)) amortized | O(N) | Effectively O(1) |
| Kruskal's MST | O(E log E) | O(E + V) | Dominated by the sort |
| Prim's MST (heap) | O(E log V) | O(E + V) | Lazy deletion, standard for sparse-ish graphs |
| Prim's MST (array, no heap) | O(V²) | O(V) | Better for DENSE graphs (E ≈ V²) |
| Min Cost Connect Points (LC 1584) | O(V²) via Prim-array; O(V² log V) via Kruskal | O(V²) edges if Kruskal | Complete graph — Prim-array usually wins |
| Optimize Water Distribution (LC 1168) | O((n + p) log(n + p)) | O(n + p) | p = number of pipes; virtual node adds n edges |
| Critical/Pseudo-Critical Edges (LC 1489) | O(E² α(V)) | O(E) | MST run O(E) times, each O(E α(V)) after one sort |
| Remove Max Edges (LC 1579) | O(E α(V)) | O(V) | Two DSUs, single pass per type |
| MST-based clustering (K clusters) | O(E log E) | O(E + V) | Kruskal, stop after V-K unions |

---

## SECTION 11 — VARIANTS: MST-BASED CLUSTERING

```cpp
// ─────────────────────────────────────────────────────────────────
// VARIANT: Single-linkage clustering via MST early stop
// Given n points and a target of K clusters, minimize the maximum
// distance WITHIN any cluster (equivalently: maximize the minimum
// distance BETWEEN clusters — "maximize the spacing").
//
// KEY INSIGHT: Run Kruskal normally, but STOP after exactly n - K
// unions (instead of n - 1). At that point, the DSU has exactly K
// components. The LAST edge weight processed before stopping is
// the answer to "maximize the minimum inter-cluster distance"
// (this is literally LC 1584-style follow-up "K clusters" and the
// classic interview question "maximize spacing to divide into K
// clusters" — same idea as Kruskal's MST, just truncated early).
// ─────────────────────────────────────────────────────────────────

int maxSpacingKClusters(int n, vector<Edge>& edges, int K) {
    sort(edges.begin(), edges.end(), [](const Edge& a, const Edge& b) {
        return a.w < b.w;
    });

    DSU dsu(n);
    int lastMergeWeight = 0;
    int targetUnions = n - K;   // stop once exactly K components remain
    int unionsSoFar = 0;

    for (const Edge& e : edges) {
        if (unionsSoFar == targetUnions) break;
        if (dsu.unite(e.u, e.v)) {
            lastMergeWeight = e.w;
            unionsSoFar++;
        }
    }

    // The NEXT unprocessed edge (the first one Kruskal would have added
    // if continuing) — its weight is the max spacing between clusters.
    // (In many formulations you instead want lastMergeWeight; check the
    // exact problem statement for "distance so far" vs "next edge".)
    return lastMergeWeight;
}
```

**Why this works:** Kruskal builds the MST by adding cheapest edges first. If you stop after `n - K` merges, you've built a **minimum spanning forest** with exactly K trees — and by the cycle property, any edge NOT yet added (because it was more expensive) is the boundary between clusters. This is provably the clustering that **maximizes the minimum distance between any two clusters** among all K-clusterings — a classic greedy-correctness result (Kruskal's algorithm doubles as an optimal clustering algorithm when truncated).

---

## SECTION 12 — COMMON MISTAKES

### Mistake 1: Forgetting to check `unite()`'s return value in Kruskal

```cpp
// WRONG — always adds the edge, never checks for cycles
for (const Edge& e : sortedEdges) {
    dsu.unite(e.u, e.v);         // return value ignored!
    totalWeight += e.w;          // BUG: adds weight even if a cycle formed
}
// This sums ALL edge weights, not just the MST — completely wrong answer.

// CORRECT — only count the edge if unite() actually merged components
for (const Edge& e : sortedEdges) {
    if (dsu.unite(e.u, e.v)) {   // true only if u, v were in different components
        totalWeight += e.w;
    }
}
```

---

### Mistake 2: Prim revisiting nodes / not checking `visited[]` on pop

```cpp
// WRONG — processes a popped node without checking if it's already visited
while (!pq.empty()) {
    auto [w, u] = pq.top(); pq.pop();
    visited[u] = true;           // BUG: marks visited unconditionally
    totalWeight += w;            // BUG: double-counts stale entries
    for (auto [nw, v] : adj[u]) if (!visited[v]) pq.push({nw, v});
}
// A vertex can be pushed multiple times (once per edge into it). Without
// the visited check on POP, stale (higher-weight) entries get processed
// as if they were new, inflating totalWeight and corrupting the tree.

// CORRECT — check and skip stale entries immediately after popping
while (!pq.empty()) {
    auto [w, u] = pq.top(); pq.pop();
    if (visited[u]) continue;    // lazy deletion: discard stale entry
    visited[u] = true;
    totalWeight += w;
    for (auto [nw, v] : adj[u]) if (!visited[v]) pq.push({nw, v});
}
```

---

### Mistake 3: Comparing MST weights with `>=` instead of `>` for critical-edge detection (LC 1489)

```cpp
// WRONG — flags an edge as critical whenever excluding it doesn't
// strictly IMPROVE the MST (i.e., treats "no change" as critical too)
if (wExcl >= baseline) critical.push_back(i);
// If wExcl == baseline, the edge was NOT needed — some other edge
// combination achieves the same weight without it. That's the
// definition of NOT critical (possibly pseudo-critical instead).

// CORRECT — only strictly worse (or disconnected) means critical
if (wExcl > baseline) critical.push_back(i);   // includes wExcl == INT_MAX (disconnected)
```

---

### Mistake 4: Using O(n²) edges when the graph is already sparse and given as an edge list

```cpp
// WRONG — building a dense adjacency structure or all-pairs edges
// for a problem where the input is already a sparse edge list
// (e.g., LC 1135 Connecting Cities: given m edges directly)
vector<vector<int>> adjMatrix(n, vector<int>(n, INT_MAX));
for (auto& e : connections) adjMatrix[e[0]][e[1]] = e[2];
// Then running O(n²) Prim — wasteful when m << n²; you threw away the
// sparsity information the problem gave you for free.

// CORRECT — use Kruskal directly on the given edge list (sparse-friendly)
int minimumCost(int n, vector<vector<int>>& connections) {
    vector<Edge> edges;
    for (auto& c : connections) edges.push_back({c[2], c[0]-1, c[1]-1});
    return kruskalMST(n, edges);
}
```

---

### Mistake 5: Off-by-one on the "MST is complete" check

```cpp
// WRONG — checks edgesUsed == n instead of n - 1
if (edgesUsed == n) break;   // BUG: a tree on n vertices has n-1 edges, not n!
// This either loops one extra iteration harmlessly (if disconnected, it
// just never triggers and finishes the loop) OR, worse, in code that
// RELIES on this check to decide "connected" vs "disconnected", silently
// reports every valid MST as disconnected.

// CORRECT
if (edgesUsed == n - 1) break;
// ... and afterward: return (edgesUsed == n - 1) ? totalWeight : -1;
```

---

### Mistake 6: Forgetting the virtual node must count toward "n+1" nodes in LC 1168

```cpp
// WRONG — sizes the DSU for n nodes, but well-edges reference index 0..n
DSU dsu(n);                     // BUG: node n (last house, if 0-indexed
                                 // internally without the +1 offset) is
                                 // out of bounds, or node 0 collides with
                                 // house 1 if you forgot to shift indices.

// CORRECT — DSU must have n+1 slots: node 0 = virtual source, nodes 1..n = houses
DSU dsu(n + 1);
for (int i = 0; i < n; i++) edges.push_back({wells[i], 0, i + 1});
```

---

## SECTION 13 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Min Cost to Connect All Points | 1584 | Implicit complete graph, Prim O(V²) vs Kruskal |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 2 | Connecting Cities With Minimum Cost | 1135 | Direct Kruskal on sparse edge list, disconnected check |
| 3 | Optimize Water Distribution in a Village | 1168 | Virtual node trick, MST over n+1 nodes |

### STRETCH (solve in ≤ 45 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 4 | Find Critical and Pseudo-Critical Edges in MST | 1489 | MST-as-subroutine, exclude/force testing |
| 5 | Minimum Spanning Tree from scratch (no library, both algorithms) | — | Implement Kruskal AND Prim unaided, verify they agree |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Remove Max Number of Edges to Keep Graph Fully Traversable | 1579 | Two DSUs, greedy shared-edge-first ordering |

---

## SECTION 14 — PATTERN CONNECTIONS

1. **Pattern 13 (Union-Find):** MST's Kruskal implementation IS Pattern 13's DSU applied with one twist — the return value of `unite()` doubles as a cycle detector. Every recognition signal from Pattern 13 ("detect a cycle while adding edges") is literally Kruskal's inner loop. If you're comfortable with Pattern 13, Kruskal is not new material — it's DSU plus a sort.

2. **Pattern 14 (Shortest Path — Dijkstra):** Prim's algorithm and Dijkstra's algorithm look nearly identical in code (both: priority queue, pop the min, relax neighbors) but answer **different questions**. Dijkstra's relaxation is `dist[v] = min(dist[v], dist[u] + weight(u,v))` — cost is *cumulative from the source*. Prim's relaxation is `minEdge[v] = min(minEdge[v], weight(u,v))` — cost is *only the single edge*, with no accumulation. Confusing these is a classic bug: copy-pasting Dijkstra's `+` into Prim's code silently turns MST into "shortest path tree", which is a different (often more expensive) structure.

3. **Pattern 30 (Topological Sort):** Both MST and topo sort are "build a structure by processing elements in a specific order" (edges by weight for MST; nodes by in-degree for topo sort) but MST is fundamentally undirected/greedy-optimal while topo sort is about DAG ordering with no notion of "minimality" — don't conflate "process in order" as the unifying idea; the CORRECTNESS arguments are unrelated (cut/cycle property vs. dependency resolution).

4. **Pattern 04 / Greedy foundations:** MST is one of the canonical examples where an exchange-argument proof (cut property / cycle property) certifies that local greedy choices are globally optimal. Contrast with problems where greedy provably FAILS (e.g., 0/1 Knapsack) — MST's underlying matroid structure is *why* greedy works here specifically.

---

## SECTION 15 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 1135 — Connecting Cities With Minimum Cost

**Interviewer:** "You're given n cities and a list of connections `[city1, city2, cost]`. Connect all cities with minimum total cost. Return -1 if impossible."

**Candidate:**

*[First minute — recognize the pattern]*
> "This is asking for a Minimum Spanning Tree — connect all n cities with the minimum total edge weight, allowing any subset of the given edges. Since the input is already a sparse edge list (not a dense point cloud), I'll use Kruskal: sort edges by cost, use Union-Find to greedily add edges that don't form a cycle."

*[State correctness before coding]*
> "This greedy approach is correct by the cycle property of MSTs: processing edges in increasing order and skipping any edge whose endpoints are already connected is equivalent to never including the maximum-weight edge of any cycle, which is provably safe to exclude from an MST."

*[Code — 5 minutes]*
```cpp
int minimumCost(int n, vector<vector<int>>& connections) {
    sort(connections.begin(), connections.end(),
         [](auto& a, auto& b) { return a[2] < b[2]; });
    DSU dsu(n + 1);  // cities are 1-indexed
    int total = 0, used = 0;
    for (auto& c : connections) {
        if (dsu.unite(c[0], c[1])) {
            total += c[2];
            used++;
        }
    }
    return (used == n - 1) ? total : -1;
}
```

*[Interviewer]: "What if n = 1?"*
> "Then used == 0 == n - 1 trivially — a single city needs zero edges to be 'fully connected' to itself. My code returns 0 correctly without any special-casing, since the loop just never finds anything to union."

*[Interviewer]: "How would this change if I told you the graph could have up to 10^4 cities but only 100 edges?"*
> "Kruskal stays optimal — with E << V², sorting E edges is cheap, and the DSU handles the sparsity naturally. If it were the reverse (E ≈ V², dense), I'd switch to array-based Prim at O(V²) to avoid sorting ~V² edges."

**RED FLAGS:**
- Forgetting to check `dsu.unite()`'s return value before adding to total
- Not handling the -1 disconnected case
- Off-by-one on 1-indexed vs 0-indexed cities (DSU sized `n` instead of `n+1`)

---

### Interview Simulation 2: LC 1168 — Optimize Water Distribution in a Village

**Interviewer:** "Each house can either build its own well (independent cost) or connect to another house via a pipe (shared cost). Minimize total cost so every house has water."

**Candidate:**

*[Critical: find the reduction before coding]*
> "The tricky part is that 'build a well' isn't naturally an edge — it's a per-node cost. I'll model it as a **virtual node 0** representing an infinite, free water source. Building a well at house i becomes an edge (0, i, wells[i]). Now every house connecting back to node 0 — directly via its own well-edge, or transitively via pipes to a house that has one — is exactly what 'has water' means. The problem becomes: find the MST over n+1 nodes (0 through n)."

*[Explain why this is airtight, not just a heuristic]*
> "Because node 0 has an edge to every house, the graph on {0, 1, ..., n} is always connected regardless of pipes — so the MST always exists, and its total weight is exactly the minimum cost to guarantee water everywhere. This isn't an approximation; the reduction is exact because 'has water' and 'is connected to node 0 in the spanning tree' are logically identical statements."

*[Code — 5 minutes, reusing Kruskal]*
```cpp
int minCostToSupplyWater(int n, vector<int>& wells, vector<vector<int>>& pipes) {
    vector<Edge> edges;
    for (int i = 0; i < n; i++) edges.push_back({wells[i], 0, i + 1});
    for (auto& p : pipes) edges.push_back({p[2], p[0], p[1]});
    return kruskalMST(n + 1, edges);
}
```

*[Interviewer]: "Could you solve this with Prim instead?"*
> "Yes — start Prim from the virtual node 0 (initialize its priority-queue entry with 0 to enter), then it grows the tree exactly the same way. Since well-costs can vary a lot and pipes are usually sparse relative to n², Kruskal's sort is not a bottleneck here, so I'd default to Kruskal for simplicity, but Prim starting at node 0 is equally correct."

**RED FLAGS:**
- Trying to solve it as "greedily pick cheapest option per house" (fails — a house's optimal source depends on its neighbors' choices, exactly the trap MST avoids)
- Forgetting the DSU needs n+1 slots, not n
- Not justifying WHY the virtual node reduction is exact, not approximate

---

## SECTION 16 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: State the cut property and explain, in one sentence, which MST algorithm it directly justifies.**

> The cut property: for any partition of vertices into two non-empty sets, the minimum-weight edge crossing that partition belongs to some MST. It directly justifies **Prim's algorithm**, which at every step picks the minimum-weight edge crossing the cut between "visited" and "unvisited" vertices.

---

**Q2: State the cycle property and explain which MST algorithm it directly justifies.**

> The cycle property: for any cycle in the graph, the maximum-weight edge on that cycle is excluded from some MST. It directly justifies **Kruskal's algorithm** — processing edges in increasing weight order and skipping an edge whenever its endpoints are already connected is exactly "never add the maximum-weight edge of the cycle that edge would complete."

---

**Q3: You have a graph with E ≈ 10^6 edges and V ≈ 10^3 vertices (very dense). Which MST algorithm should you default to, and why?**

> Prim's algorithm with the O(V²) array-based (no heap) implementation. With E ≈ V², Kruskal's O(E log E) sort costs roughly O(V² log V²) = O(V² · 2 log V), which is strictly worse than Prim's clean O(V²) scan-based selection. The array-based Prim avoids heap overhead entirely because in a dense graph you're going to touch nearly every edge anyway.

---

**Q4: In Kruskal's implementation, what specific line is the "cycle detector," and what happens if you forget to check it?**

> The line is `if (dsu.unite(e.u, e.v))` — `unite()` returns `false` exactly when `e.u` and `e.v` are already in the same component, which means adding this edge would form a cycle. If you forget to check this return value and unconditionally add every edge's weight to the total, you sum the weight of ALL edges (or however many you process before some arbitrary cutoff), not just the V-1 edges of the actual MST — producing a wildly incorrect, inflated answer.

---

**Q5: For LC 1489 (Critical and Pseudo-Critical Edges), why do we need to test BOTH "exclude the edge" and "force the edge," rather than just one of the two?**

> "Exclude and see if the MST weight increases" tells us whether the edge is **necessary** (critical). But an edge that's NOT necessary might still be usable — some alternate MST could include it at no extra cost. "Force the edge in and see if the resulting weight still equals baseline" tests exactly that: whether SOME optimal MST can contain this edge, even if it's not required by ALL optimal MSTs. Testing only exclusion would miss pseudo-critical edges (falsely lumping them in with edges that can NEVER appear in any MST); testing only forcing would miss distinguishing critical (always required) from merely usable.

---

**Q6: Why does LC 1579 (Remove Max Number of Edges) process type-3 (shared) edges before type-1/type-2 (private) edges, and what would break if you processed them in input order instead?**

> A type-3 edge does double duty — it can satisfy both Alice's and Bob's connectivity requirement with a single kept edge, whereas a type-1 or type-2 edge only helps one person. By the cycle property, a type-3 edge is never worse than a type-1/type-2 edge for the same merge, so using it first can only help (or be neutral) and never hurts. If you process in arbitrary/input order, you might "spend" a type-1 edge to connect two components for Alice, then later encounter a type-3 edge that would have connected the SAME two components for both Alice and Bob simultaneously — but now that type-3 edge is wasted on Alice's side (already connected) and you still need a separate type-2 edge for Bob, using one MORE edge overall than necessary, which underreports the true maximum number of removable edges.

---

**Q7: Prim's relaxation step and Dijkstra's relaxation step both use a priority queue and "pop the min, relax neighbors." What is the ONE-LINE difference in their relaxation formulas, and why does it matter?**

> Dijkstra: `dist[v] = min(dist[v], dist[u] + weight(u,v))` — cumulative distance from the source. Prim: `minEdge[v] = min(minEdge[v], weight(u,v))` — just the single edge weight, no accumulation from a source. This matters because Dijkstra answers "what's the cheapest PATH from the source to v" while Prim answers "what's the cheapest single EDGE that would connect v to the growing tree" — these produce different trees in general (a shortest-path tree is not the same as a minimum spanning tree), and accidentally copying Dijkstra's `+` into Prim's code (or vice versa) silently produces the wrong structure while still "running" without errors.

---

## SECTION 17 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. DSU (path compression + union by size) — foundation for Kruskal
// ─────────────────────────────────────────────────────────────────
struct DSU {
    vector<int> parent, sz; int components;
    DSU(int n): parent(n), sz(n,1), components(n) { iota(parent.begin(), parent.end(), 0); }
    int find(int x) { return parent[x]==x ? x : parent[x]=find(parent[x]); }
    bool unite(int x, int y) {
        int rx=find(x), ry=find(y);
        if (rx==ry) return false;
        if (sz[rx]<sz[ry]) swap(rx,ry);
        parent[ry]=rx; sz[rx]+=sz[ry]; components--;
        return true;
    }
};

// ─────────────────────────────────────────────────────────────────
// 2. KRUSKAL — sort edges, DSU, greedy add if no cycle
// ─────────────────────────────────────────────────────────────────
sort(edges.begin(), edges.end(), [](auto&a, auto&b){ return a.w<b.w; });
DSU dsu(n); int total=0, used=0;
for (auto& e : edges) {
    if (dsu.unite(e.u, e.v)) { total += e.w; used++; if (used==n-1) break; }
}
// answer = (used==n-1) ? total : -1 (disconnected)

// ─────────────────────────────────────────────────────────────────
// 3. PRIM — heap, lazy deletion
// ─────────────────────────────────────────────────────────────────
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
vector<bool> visited(n,false);
pq.push({0,0}); int total=0, added=0;
while (!pq.empty() && added<n) {
    auto [w,u]=pq.top(); pq.pop();
    if (visited[u]) continue;          // lazy deletion — stale entry
    visited[u]=true; total+=w; added++;
    for (auto [nw,v]: adj[u]) if (!visited[v]) pq.push({nw,v});
}

// ─────────────────────────────────────────────────────────────────
// 4. PRIM — O(V^2) array version (dense graphs, e.g. LC 1584)
// ─────────────────────────────────────────────────────────────────
vector<int> minEdge(n, INT_MAX); vector<bool> inMST(n,false);
minEdge[0]=0; int total=0;
for (int it=0; it<n; it++) {
    int u=-1;
    for (int v=0; v<n; v++) if (!inMST[v] && (u==-1 || minEdge[v]<minEdge[u])) u=v;
    inMST[u]=true; total+=minEdge[u];
    for (int v=0; v<n; v++) if (!inMST[v]) minEdge[v]=min(minEdge[v], dist(u,v));
}

// ─────────────────────────────────────────────────────────────────
// 5. VIRTUAL NODE TRICK (LC 1168) — per-node independent cost -> edge to node 0
// ─────────────────────────────────────────────────────────────────
for (int i=0;i<n;i++) edges.push_back({wells[i], 0, i+1});
// then MST over n+1 nodes (0..n)

// ─────────────────────────────────────────────────────────────────
// 6. CRITICAL / PSEUDO-CRITICAL TEST (LC 1489) — exclude then force
// ─────────────────────────────────────────────────────────────────
// baseline = mstWeight(all edges)
// for each edge i:
//   wExcl  = mstWeight(skip i)          -> if wExcl  > baseline: CRITICAL
//   wForce = mstWeight(pre-union i)     -> if wForce == baseline: PSEUDO-CRITICAL

// ─────────────────────────────────────────────────────────────────
// 7. TWO-DSU SHARED-EDGE-FIRST (LC 1579)
// ─────────────────────────────────────────────────────────────────
// process type-3 edges first in BOTH DSUs, count if EITHER merged
// then type-1 in Alice's DSU only, type-2 in Bob's DSU only
// answer = totalEdges - usedEdges, valid iff both DSUs end with 1 real component

// ─────────────────────────────────────────────────────────────────
// 8. DECISION TABLE — Kruskal vs Prim
// ─────────────────────────────────────────────────────────────────
// Sparse (E ~ V), edge list given         -> Kruskal (sort + DSU)
// Dense (E ~ V^2), implicit/complete graph -> Prim, O(V^2) array version
// Need MST run repeatedly with tweaks      -> Kruskal (DSU composes cleanly
//                                             with exclude/force/pre-union)
```

---

## SECTION 18 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 6 problems in order.** LC 1489 (Critical/Pseudo-Critical Edges) is the conceptual peak of this pattern — budget 45 minutes and don't look at the solution until you've at least designed the exclude/force approach yourself.

2. **Implement BOTH Kruskal and Prim for the same problem (Problem 5) and verify they produce the same total weight** (the specific tree can differ if weights tie, but the weight cannot). This cements that they're two proofs of the same underlying theorem, not two unrelated algorithms.

3. **Revisit Pattern 13 (Union-Find)** if `unite()`'s return-value-as-cycle-detector felt unfamiliar — that single line is doing all the correctness work in Kruskal, and it's worth being fully fluent in DSU internals before treating it as a black box.

4. **Contrast Prim vs Dijkstra explicitly:** take your Prim implementation, change the relaxation formula to add `dist[u]` (making it cumulative), and observe how the resulting tree changes on a graph where they diverge. This is the single most valuable "aha" exercise for cementing the distinction from Pattern 14.

5. **Preview the clustering variant (Section 11):** try "Divide n points into K clusters to maximize the minimum inter-cluster distance" using the Kruskal-early-stop trick. This shows up as a follow-up in interviews after LC 1584 more often than candidates expect.

---

## SECTION 19 — SIGN-OFF CRITERIA

### Tier 1 — Kruskal mastered
You implement DSU from scratch (path compression + union by size) and Kruskal's algorithm cleanly, correctly checking `unite()`'s return value every time. You can state the cycle property from memory and explain in one sentence why it justifies Kruskal.

### Tier 2 — Prim mastered, Kruskal-vs-Prim judgment solid
You implement Prim with lazy deletion (heap version) AND the O(V²) array version. You can state the cut property and explain why it justifies Prim. You correctly choose Kruskal for sparse graphs and Prim-array for dense/implicit-complete-graph problems (LC 1584) without being told which to use.

### Tier 3 — MST-as-subroutine problems solid
You solve the virtual-node trick (LC 1168) and can justify why it's an EXACT reduction, not a heuristic. You solve Connecting Cities (LC 1135) including the disconnected (-1) edge case cleanly.

### Tier 4 — Hard variants complete
You solve Find Critical and Pseudo-Critical Edges (LC 1489) using the exclude/force testing framework and can explain why BOTH tests are necessary (Q5 above). You solve Remove Max Number of Edges (LC 1579) with the two-DSU, shared-edge-first approach and can explain why that ordering is a correctness requirement, not just an optimization (Q6 above).

**Sign-off threshold:** Solve 5 of 6 problems. Mandatory: LC 1584 (Min Cost to Connect All Points — must implement both Kruskal and Prim), LC 1168 (Optimize Water Distribution — virtual node trick must be self-derived, not memorized), and LC 1489 (Critical/Pseudo-Critical Edges — the hardest conceptual leap in this pattern, MST as a parameterized subroutine).

---

*Pattern 31 Complete — Minimum Spanning Tree*
*Next: Pattern 32 — Shortest Path Variants (Bellman-Ford, Floyd-Warshall, and negative-weight edge handling)*
