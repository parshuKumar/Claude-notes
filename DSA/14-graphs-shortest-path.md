# PATTERN 14 — GRAPHS: SHORTEST PATH
## Dijkstra, Bellman-Ford, 0-1 BFS, Floyd-Warshall — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**What most people get wrong about shortest path:**

They memorize Dijkstra's code and call it a day. Then they encounter a problem with K stops, or a grid where moves cost 0 or 1, or negative weights — and they reach for Dijkstra anyway and get wrong answers or TLE.

The real skill is **algorithm selection**: knowing which variant applies to which constraints, and *why* Dijkstra fails when it does. There is a clean decision tree that resolves this in 10 seconds once you understand the underlying properties.

**The core insight that unifies everything:**

All shortest path algorithms maintain a "relaxation" idea: if going through node v makes the path to node u shorter, update `dist[u]`. The algorithms differ in:
- **What order** they process nodes (greedy by distance vs arbitrary)
- **What guarantees** they require (non-negative weights vs any weights)
- **How many passes** they make (one greedy pass vs V-1 passes)

**When each algorithm applies:**

| Condition | Algorithm | Complexity |
|-----------|-----------|------------|
| Unweighted graph | BFS | O(V + E) |
| Non-negative weights | Dijkstra | O((V + E) log V) |
| Negative weights, no negative cycles | Bellman-Ford | O(V × E) |
| Weights ∈ {0, 1} | 0-1 BFS | O(V + E) |
| All-pairs, small N (≤ 300) | Floyd-Warshall | O(V³) |
| Constrained Dijkstra (K stops, states) | Modified BFS/Dijkstra | O(V × K × log) |

**The interview reality:** Dijkstra appears in ~70% of shortest path problems. The other variants appear precisely when the interviewer wants to test whether you understand *why* Dijkstra works — and where it doesn't.

---

## SECTION 2 — ELI5

Imagine you're trying to find the fastest way from your house to a friend's house through a city. You have a map with road distances.

**BFS (unweighted):** All roads take exactly 1 minute. You explore in ripples — first all 1-minute destinations, then all 2-minute destinations, etc. The first time you reach a place, that's the shortest.

**Dijkstra:** Roads have different times. You use a "sticky note" approach — always pick the unvisited destination with the smallest known total time so far, mark it as "finalized," and update neighbors. Works because you can never find a shortcut later (no negative weights → going further can only make things longer).

**Bellman-Ford:** Some roads have negative times (time machines!). You can no longer greedily finalize — a later discovery might shortcut an earlier path. So you relax ALL edges V-1 times, guaranteeing you've found the optimal after V-1 rounds. Slow but handles any weights.

**0-1 BFS:** Roads cost either 0 or 1 minute. Cleverly use a deque: 0-cost roads push to the FRONT (don't increase time, "free" moves), 1-cost roads push to the BACK (normal queue order). This achieves O(V+E) instead of O((V+E) log V).

**Floyd-Warshall:** You want all pairs. "Does going through city k make the path from city i to city j shorter?" — check every possible intermediate city k.

---

## SECTION 3 — FORMALLY DEFINED

### Dijkstra's Algorithm

**Precondition:** All edge weights ≥ 0.

**Invariant:** When a node u is extracted from the priority queue with distance d, `dist[u] = d` is finalized (optimal). This holds because: any alternative path to u must go through at least one unvisited node, and all unvisited nodes have distance ≥ d (since we always pop the minimum), so extending any such path can only increase the distance.

**Why it fails with negative weights:** A later-discovered path through a negative edge might provide a shorter route to an already-"finalized" node. The greedy invariant breaks.

```
Dijkstra(graph, source):
    dist[source] = 0, dist[all others] = ∞
    pq = min-heap with (0, source)
    while pq not empty:
        d, u = pq.pop_min()
        if d > dist[u]: skip  // stale entry
        for each neighbor v with edge weight w:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                pq.push(dist[v], v)
    return dist
```

### Bellman-Ford Algorithm

**Precondition:** No negative cycles (if negative cycles exist, distances can be −∞).

**Correctness:** In a graph with V nodes, the shortest path (if no negative cycles) has at most V-1 edges. After k relaxation passes, `dist[v]` is the shortest path using at most k edges. After V-1 passes, all shortest paths are found.

**Negative cycle detection:** If a V-th pass still relaxes any edge, a negative cycle exists.

```
BellmanFord(graph, source):
    dist[source] = 0, dist[all others] = ∞
    for i in 1 to V-1:
        for each edge (u, v, w):
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
    // Optional: detect negative cycles
    for each edge (u, v, w):
        if dist[u] + w < dist[v]: return "negative cycle"
    return dist
```

### Floyd-Warshall Algorithm

**Precondition:** No negative cycles.

**Recurrence:** `dp[i][j][k]` = shortest path from i to j using only nodes {1..k} as intermediates.
- `dp[i][j][k] = min(dp[i][j][k-1], dp[i][k][k-1] + dp[k][j][k-1])`
- Optimized: in-place, iterate k in outer loop.

```
FloydWarshall(dist[V][V]):
    for k in 0..V-1:
        for i in 0..V-1:
            for j in 0..V-1:
                dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j])
```

---

## SECTION 4 — RECOGNITION SIGNALS

**Read the problem. If you see these, think shortest path:**

✅ **Strong signals:**
1. "Minimum cost/time/distance from source to destination" — classic SSSP
2. "Shortest path" in the literal problem statement
3. Weighted graph, find minimum weight path — Dijkstra
4. "At most K steps/stops" as a constraint — modified Dijkstra or Bellman-Ford
5. "Minimum effort" where effort = maximum edge on path — Dijkstra with max instead of sum
6. Grid with variable movement costs — Dijkstra on grid

⚠️ **Medium signals (verify constraint before picking algorithm):**
7. Edge weights are 0 or 1 → 0-1 BFS (don't use Dijkstra — it works but is slower)
8. All-pairs queries on small graph (N ≤ 300) → Floyd-Warshall
9. Probability multiplication (maximize probability) → Dijkstra with max-heap, multiply instead of add
10. "Find path that minimizes maximum edge weight" → binary search + BFS OR Dijkstra variant

❌ **FAKE signals:**
- "Count number of paths" → DP or DFS with backtracking, not Dijkstra
- "Find the path itself" → Dijkstra with parent tracking (works, but interviewers often mean "what IS the path", not cost)
- "Minimum spanning tree" → Kruskal's/Prim's, NOT shortest path (MST minimizes total edge cost to connect all nodes; SSSP minimizes path cost to reach one node)

**The three questions to ask immediately:**
1. **Are there negative weights?** Yes → Bellman-Ford (or SPFA). No → continue.
2. **Are all weights equal (or 0/1)?** Yes → BFS or 0-1 BFS. No → continue.
3. **Single source or all pairs?** Single → Dijkstra. All pairs small N → Floyd-Warshall.

---

## SECTION 5 — TEMPLATE 1: DIJKSTRA ON ADJACENCY LIST

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE 1: Standard Dijkstra — Single Source Shortest Path
// LC 743 — Network Delay Time
// Input: times[i] = {u, v, w}, n nodes, source k
// Output: minimum time for all nodes to receive signal, or -1
// ─────────────────────────────────────────────────────────────────

int networkDelayTime(vector<vector<int>>& times, int n, int k) {
    // Build adjacency list: adj[u] = {(v, w), ...}
    vector<vector<pair<int,int>>> adj(n + 1);  // 1-indexed
    for (auto& t : times) {
        adj[t[0]].push_back({t[1], t[2]});
    }

    // dist[i] = shortest distance from k to i
    vector<int> dist(n + 1, INT_MAX);
    dist[k] = 0;

    // Min-heap: (distance, node)
    // C++ priority_queue is MAX by default — negate or use greater<>
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    pq.push({0, k});

    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();

        // CRITICAL: skip stale entries
        // The heap may contain (old_dist, u) if dist[u] was updated
        // after this entry was pushed. We only process the current best.
        if (d > dist[u]) continue;

        for (auto [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});  // push new best (old entry becomes stale)
            }
        }
    }

    // Find max dist among all nodes (last node to receive signal)
    int ans = *max_element(dist.begin() + 1, dist.end());
    return ans == INT_MAX ? -1 : ans;
}
```

**TRACE through example:**
```
n=4, k=2, times=[[2,1,1],[2,3,1],[3,4,1]]

Adjacency list:
  2 → {(1,1), (3,1)}
  3 → {(4,1)}

dist = [∞, ∞, 0, ∞, ∞]   (1-indexed, dist[2]=0)
pq = [(0,2)]

Step 1: pop (0,2). d=0, u=2. d==dist[2]=0, process.
  neighbor (1,1): dist[1]=∞, 0+1=1 < ∞ → dist[1]=1, push (1,1)
  neighbor (3,1): dist[3]=∞, 0+1=1 < ∞ → dist[3]=1, push (1,3)
  pq = [(1,1),(1,3)]

Step 2: pop (1,1). d=1, u=1. d==dist[1]=1, process.
  No outgoing edges from 1. Nothing to do.
  pq = [(1,3)]

Step 3: pop (1,3). d=1, u=3. d==dist[3]=1, process.
  neighbor (4,1): dist[4]=∞, 1+1=2 < ∞ → dist[4]=2, push (2,4)
  pq = [(2,4)]

Step 4: pop (2,4). d=2, u=4. d==dist[4]=2, process.
  No outgoing edges.
  pq = []

Final dist = [∞, 1, 0, 1, 2]
max(dist[1..4]) = max(1, 0, 1, 2) = 2

Answer: 2
```

**WHY the stale-entry check `if (d > dist[u]) continue` is critical:**

Without this check, if node u is pushed twice — once with dist=10, then updated to dist=5 — when the old entry (10, u) is eventually popped, we'd re-process u with the wrong distance and potentially push incorrect updates to neighbors. The stale check ensures each node is effectively processed at most once (at its optimal distance).

**HOW TO ADAPT:**
- For **path reconstruction**: add `parent[v] = u` when updating `dist[v]`, then trace back from destination.
- For **number of shortest paths**: add `count[v]` and update it when relaxing.
- For **maximize probability**: use `max-heap`, `prob[v] = max(prob[v], prob[u] * w)` (Section 8).

---

## SECTION 6 — TEMPLATE 2: DIJKSTRA ON GRID

Grid problems where each cell has a movement cost. The "grid as graph" transformation: each cell is a node, each move to adjacent cell is an edge with weight = the cell's cost (or transition cost).

```cpp
// LC 1631 — Path With Minimum Effort
// effort = maximum absolute difference in height on the path
// Minimize the maximum edge on any path from (0,0) to (m-1,n-1)

int minimumEffortPath(vector<vector<int>>& heights) {
    int m = heights.size(), n = heights[0].size();

    // dist[i][j] = minimum "maximum effort" to reach (i,j) from (0,0)
    vector<vector<int>> dist(m, vector<int>(n, INT_MAX));
    dist[0][0] = 0;

    // min-heap: (effort_so_far, row, col)
    priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<>> pq;
    pq.push({0, 0, 0});

    int dirs[4][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!pq.empty()) {
        auto [effort, r, c] = pq.top(); pq.pop();

        if (effort > dist[r][c]) continue;  // stale

        // Early exit: reached destination
        if (r == m-1 && c == n-1) return effort;

        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;

            // "cost" of this edge = absolute height difference
            int newEffort = max(effort, abs(heights[nr][nc] - heights[r][c]));

            if (newEffort < dist[nr][nc]) {
                dist[nr][nc] = newEffort;
                pq.push({newEffort, nr, nc});
            }
        }
    }

    return dist[m-1][n-1];
}
```

**KEY INSIGHT — "max edge" Dijkstra vs "sum edge" Dijkstra:**

Standard Dijkstra: `dist[v] = min(dist[v], dist[u] + w)` — minimizes total path weight.

This problem: `dist[v] = min(dist[v], max(dist[u], w))` — minimizes the maximum edge on the path.

Both use the SAME greedy structure (Dijkstra's invariant: finalized node has optimal value). The only change is the "combination function":
- Sum: `dist[u] + w`
- Max: `max(dist[u], w)`

Both satisfy the required monotonicity property: combination never decreases when going further. This is why Dijkstra works for both.

**TRACE through small example:**
```
heights = [[1,2,2],[3,8,2],[5,3,5]]
Source: (0,0), Dest: (2,2)

dist[0][0]=0. Push (0,0,0).
Pop (0,0,0). Neighbors:
  (0,1): effort=max(0,|2-1|)=1 < ∞ → dist[0][1]=1, push (1,0,1)
  (1,0): effort=max(0,|3-1|)=2 < ∞ → dist[1][0]=2, push (2,1,0)

Pop (1,0,1). Neighbors:
  (0,0): max(1,|1-2|)=1 ≥ dist[0][0]=0. No update.
  (0,2): max(1,|2-2|)=1 < ∞ → dist[0][2]=1, push (1,0,2)
  (1,1): max(1,|8-2|)=6 < ∞ → dist[1][1]=6, push (6,1,1)

Pop (1,0,2). Neighbors:
  (1,2): max(1,|2-2|)=1 < ∞ → dist[1][2]=1, push (1,1,2)
  ...

Pop (1,1,2). Neighbors:
  (2,2): max(1,|5-2|)=3 < ∞ → dist[2][2]=3, push (3,2,2)
  ...

Pop (2,1,0). ... eventually (3,2,2) is popped.
Pop (3,2,2): r=2,c=2 = destination. Return 3.

Answer: 2 (optimal path: (0,0)→(0,1)→(0,2)→(1,2)→(2,2), max diff=max(1,0,0,3)=3)
Wait — actual answer is 2. Let me check. 
Actually the optimal path goes (0,0)→(1,0)→(2,0)→(2,1)→(2,2): diffs=|3-1|=2,|5-3|=2,|3-5|=2,|5-3|=2 → max=2.
So dist[2][2] would be updated when (2,2) is reached via this path. The Dijkstra explores optimally.
Final answer: 2
```

---

## SECTION 7 — TEMPLATE 3: BELLMAN-FORD (CHEAPEST FLIGHTS WITH K STOPS)

Bellman-Ford is ideal when you have a constraint like "at most K edges/stops" — because exactly K passes of Bellman-Ford give you the shortest path using AT MOST K edges.

```cpp
// LC 787 — Cheapest Flights Within K Stops
// K stops = K+1 edges maximum
// Classic Bellman-Ford: after k passes, dist[v] = shortest path using ≤ k edges

int findCheapestPrice(int n, vector<vector<int>>& flights,
                       int src, int dst, int k) {
    // dist[v] = cheapest price from src to v using at most i edges
    vector<int> dist(n, INT_MAX);
    dist[src] = 0;

    // K stops = K+1 edges → run K+1 relaxation passes
    for (int i = 0; i <= k; i++) {
        // CRITICAL: use a copy to ensure we only count edges from the PREVIOUS pass
        // Without copy, a single pass might use multiple edges from this same round
        vector<int> temp = dist;

        for (auto& f : flights) {
            int u = f[0], v = f[1], w = f[2];
            if (dist[u] != INT_MAX && dist[u] + w < temp[v]) {
                temp[v] = dist[u] + w;
            }
        }

        dist = temp;  // commit this round's updates
    }

    return dist[dst] == INT_MAX ? -1 : dist[dst];
}
```

**WHY the `temp` copy is essential — trace without it:**

```
Scenario: src=0, dst=2, k=1 (at most 1 stop = 2 edges)
Edges: (0→1, cost 1), (1→2, cost 1)

WITHOUT temp copy (wrong):
  Pass 1, process (0→1): dist[1] = 0+1 = 1
  Pass 1, process (1→2): dist[2] = 1+1 = 2  ← BUG! Uses dist[1] updated THIS pass
  This means we used 2 edges in a single "1-edge" pass.

WITH temp copy (correct):
  Pass 1 starts: dist = [0, ∞, ∞]
  temp starts as copy: [0, ∞, ∞]
  Process (0→1): dist[0]=0 ≠ INT_MAX, 0+1=1 < temp[1]=∞ → temp[1]=1
  Process (1→2): dist[1]=∞ (original!), skip.
  End of pass 1: dist = [0, 1, ∞]
  
  Pass 2 starts: dist = [0, 1, ∞]
  Process (1→2): dist[1]=1, 1+1=2 < ∞ → temp[2]=2
  End of pass 2: dist = [0, 1, 2]

After k+1=2 passes: dist[2] = 2 (using exactly 2 edges — 0→1→2)
Answer: 2 ✓
```

**TRACE through full example:**
```
n=3, src=0, dst=2, k=1
flights = [[0,1,100],[1,2,100],[0,2,500]]

Initial: dist = [0, ∞, ∞]

Pass i=0 (1 edge allowed):
  temp = [0, ∞, ∞]
  (0→1,100): dist[0]=0, 0+100=100 < temp[1]=∞ → temp[1]=100
  (1→2,100): dist[1]=∞, skip
  (0→2,500): dist[0]=0, 0+500=500 < temp[2]=∞ → temp[2]=500
  dist = [0, 100, 500]

Pass i=1 (2 edges allowed):
  temp = [0, 100, 500]
  (0→1,100): dist[0]=0, 100 = temp[1]=100. No update.
  (1→2,100): dist[1]=100, 100+100=200 < temp[2]=500 → temp[2]=200
  (0→2,500): dist[0]=0, 500 = temp[2]=500. No update (200 already set).

dist = [0, 100, 200]

Answer: dist[2] = 200
```

**ALTERNATIVE: BFS/DP approach (often preferred in interviews for K stops problems):**

```cpp
// Same problem, BFS-style DP — slightly cleaner and same complexity
int findCheapestPrice(int n, vector<vector<int>>& flights,
                       int src, int dst, int k) {
    vector<int> dist(n, INT_MAX);
    dist[src] = 0;

    for (int i = 0; i <= k; i++) {
        vector<int> temp = dist;
        for (auto& f : flights) {
            int u = f[0], v = f[1], w = f[2];
            if (dist[u] != INT_MAX)
                temp[v] = min(temp[v], dist[u] + w);
        }
        dist = temp;
    }

    return dist[dst] == INT_MAX ? -1 : dist[dst];
}
```

---

## SECTION 8 — TEMPLATE 4: DIJKSTRA WITH STATE (MODIFIED DIJKSTRA)

For problems with extra constraints that modify the state space. The key insight: instead of `dist[node]`, we use `dist[node][extra_state]`.

### Variant A: Maximum Probability Path (LC 1514)

```cpp
// LC 1514 — Path with Maximum Probability
// Multiply probabilities instead of adding weights
// Use MAX-heap (we want maximum probability, not minimum cost)

double maxProbability(int n, vector<vector<int>>& edges,
                       vector<double>& succProb, int start, int end) {
    // Build adjacency list
    vector<vector<pair<int,double>>> adj(n);
    for (int i = 0; i < edges.size(); i++) {
        adj[edges[i][0]].push_back({edges[i][1], succProb[i]});
        adj[edges[i][1]].push_back({edges[i][0], succProb[i]});  // undirected
    }

    vector<double> prob(n, 0.0);
    prob[start] = 1.0;

    // MAX-heap: (probability, node)
    priority_queue<pair<double,int>> pq;  // default is max-heap
    pq.push({1.0, start});

    while (!pq.empty()) {
        auto [p, u] = pq.top(); pq.pop();

        if (p < prob[u]) continue;  // stale

        if (u == end) return p;  // early exit

        for (auto [v, w] : adj[u]) {
            if (prob[u] * w > prob[v]) {
                prob[v] = prob[u] * w;
                pq.push({prob[v], v});
            }
        }
    }

    return 0.0;
}
```

**WHY max-heap and multiply:** We want the maximum probability path. `prob[u] * w > prob[v]` replaces `dist[u] + w < dist[v]`. The Dijkstra invariant still holds because probabilities ∈ [0,1] — multiplying can only decrease (or stay same), never increase. So once a node is popped from the max-heap with the highest probability seen so far, that is optimal.

### Variant B: Dijkstra with K-constraint as extra state dimension

```cpp
// Shortest path with at most K special edges allowed
// State = (node, remaining_special_edges)

// dist[node][k_remaining] = shortest path using node with k special edges left
// This is the "BFS with state" pattern
```

---

## SECTION 9 — TEMPLATE 5: 0-1 BFS

When edge weights are only 0 or 1, use a **deque** instead of a priority queue. This achieves O(V+E) instead of O((V+E) log V) — the same asymptotic as BFS, but for weighted graphs.

**The key insight:** In a deque-based BFS, the front of the deque always has the smallest distance. When we take a 0-weight edge, the distance doesn't increase, so we push to the FRONT. When we take a 1-weight edge, the distance increases by 1, so we push to the BACK.

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: 0-1 BFS
// LC 1368 — Minimum Cost to Make at Least One Valid Path
// Grid where moving in arrow direction costs 0, against arrow costs 1
// ─────────────────────────────────────────────────────────────────

int minCost(vector<vector<int>>& grid) {
    int m = grid.size(), n = grid[0].size();

    // Direction encoding: 1=right, 2=left, 3=down, 4=up
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};

    vector<vector<int>> dist(m, vector<int>(n, INT_MAX));
    dist[0][0] = 0;

    deque<pair<int,int>> dq;  // ← deque, not queue or priority_queue
    dq.push_front({0, 0});

    while (!dq.empty()) {
        auto [r, c] = dq.front(); dq.pop_front();

        for (int dir = 0; dir < 4; dir++) {
            int nr = r + dr[dir], nc = c + dc[dir];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;

            // Cost 0 if moving in the cell's arrow direction, 1 otherwise
            // grid[r][c] is 1-indexed: 1=right(dir0), 2=left(dir1), etc.
            int cost = (grid[r][c] == dir + 1) ? 0 : 1;
            int newDist = dist[r][c] + cost;

            if (newDist < dist[nr][nc]) {
                dist[nr][nc] = newDist;
                if (cost == 0) {
                    dq.push_front({nr, nc});   // ← 0-cost: push to FRONT
                } else {
                    dq.push_back({nr, nc});    // ← 1-cost: push to BACK
                }
            }
        }
    }

    return dist[m-1][n-1];
}
```

**TRACE through small example:**
```
grid = [[1,1,1,1],[2,2,2,2],[1,1,1,1],[2,2,2,2]] (4x4 all-right and all-left)
(Simplified: grid = [[1,1],[2,1]], m=2, n=2)
  grid[0][0]=1 (right), grid[0][1]=1 (right), grid[1][0]=2 (left), grid[1][1]=1 (right)

dist = [[0,∞],[∞,∞]]
dq = [(0,0)]

Pop (0,0). dist[0][0]=0.
  dir=0 (right): (0,1). cost=(grid[0][0]==1)=0. newDist=0. dist[0][1]=0. push FRONT.
  dir=1 (left):  (-) OOB.
  dir=2 (down):  (1,0). cost=(grid[0][0]==3)=1. newDist=1. dist[1][0]=1. push BACK.
  dir=3 (up):    (-) OOB.
dq = [(0,1), (1,0)]   ← (0,1) at front (0-cost), (1,0) at back (1-cost)

Pop (0,1). dist[0][1]=0.
  dir=0 (right): OOB.
  dir=2 (down): (1,1). cost=(grid[0][1]==3)=1. newDist=1. dist[1][1]=1. push BACK.
  dir=1 (left): (0,0). newDist=0+1=1 ≥ dist[0][0]=0. No update.
dq = [(1,0), (1,1)]

Pop (1,0). dist[1][0]=1.
  dir=1 (left): OOB.
  dir=0 (right): (1,1). cost=(grid[1][0]==1)?0:1=0 (grid[1][0]=2, dir=0 is right, 2≠1, cost=1).
                 newDist=1+1=2 > dist[1][1]=1. No update.
  dir=3 (up): (0,0). newDist=1+1=2 > 0. No update.

Pop (1,1). dist[1][1]=1.

Answer: dist[1][1] = 1
```

**CRITICAL: Why deque and not priority queue?**

Both work correctly. But deque gives O(V+E) while priority queue gives O((V+E) log V). For 0-1 weights, deque is strictly better. The deque maintains the invariant that elements are sorted by distance — distance 0 costs don't increase the distance, so pushing to front maintains sorted order.

---

## SECTION 10 — TEMPLATE 6: FLOYD-WARSHALL (ALL-PAIRS SHORTEST PATH)

```cpp
// ─────────────────────────────────────────────────────────────────
// TEMPLATE: Floyd-Warshall
// LC 1334 — Find the City With Smallest Number of Neighbors at Threshold
// ─────────────────────────────────────────────────────────────────

int findTheCity(int n, vector<vector<int>>& edges, int distanceThreshold) {
    // Initialize distance matrix
    vector<vector<int>> dist(n, vector<int>(n, INT_MAX / 2));  // ÷2 to avoid overflow
    for (int i = 0; i < n; i++) dist[i][i] = 0;
    for (auto& e : edges) {
        dist[e[0]][e[1]] = e[2];
        dist[e[1]][e[0]] = e[2];  // undirected
    }

    // Floyd-Warshall: try every intermediate node k
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                }
            }
        }
    }

    // Find city with fewest reachable neighbors within threshold
    int ans = -1, minNeighbors = INT_MAX;
    for (int i = 0; i < n; i++) {
        int count = 0;
        for (int j = 0; j < n; j++) {
            if (i != j && dist[i][j] <= distanceThreshold) count++;
        }
        // Tie-breaking: prefer larger city index
        if (count <= minNeighbors) {
            minNeighbors = count;
            ans = i;
        }
    }

    return ans;
}
```

**TRACE through example:**
```
n=4, edges=[[0,1,3],[1,2,1],[1,3,4],[2,3,1]], distanceThreshold=4

Initial dist matrix:
    0    1    2    3
0 [ 0,   3, INF, INF]
1 [ 3,   0,   1,   4]
2 [INF,  1,   0,   1]
3 [INF,  4,   1,   0]

k=0 (try node 0 as intermediate):
  dist[1][0] + dist[0][j] doesn't help anything new.
  (All paths through 0 are longer since 0 only connects to 1)

k=1 (try node 1 as intermediate):
  dist[0][2] = min(INF, dist[0][1]+dist[1][2]) = min(INF, 3+1) = 4
  dist[0][3] = min(INF, dist[0][1]+dist[1][3]) = min(INF, 3+4) = 7
  dist[2][0] = 4, dist[3][0] = 7

k=2 (try node 2 as intermediate):
  dist[0][3] = min(7, dist[0][2]+dist[2][3]) = min(7, 4+1) = 5
  dist[1][3] = min(4, dist[1][2]+dist[2][3]) = min(4, 1+1) = 2
  dist[3][0] = 5, dist[3][1] = 2

k=3:
  dist[0][1] = min(3, dist[0][3]+dist[3][1]) = min(3, 5+2) = 3. No change.
  ...

Final dist matrix:
    0   1   2   3
0 [ 0,  3,  4,  5]
1 [ 3,  0,  1,  2]
2 [ 4,  1,  0,  1]
3 [ 5,  2,  1,  0]

Threshold = 4:
  City 0: neighbors with dist ≤ 4: {1(3), 2(4)} = 2 neighbors
  City 1: neighbors with dist ≤ 4: {0(3), 2(1), 3(2)} = 3 neighbors
  City 2: neighbors with dist ≤ 4: {0(4), 1(1), 3(1)} = 3 neighbors
  City 3: neighbors with dist ≤ 4: {1(2), 2(1)} = 2 neighbors

Cities 0 and 3 both have 2 neighbors. Prefer larger index → answer: 3
```

**WHY `INT_MAX / 2` not `INT_MAX`?**
`dist[i][k] + dist[k][j]` — if both are `INT_MAX`, the addition overflows to a negative number, which would incorrectly update distances. Using `INT_MAX / 2` ≈ 10^9 prevents overflow for reasonable weights.

**HOW TO ADAPT Floyd-Warshall:**
- Transitive closure (reachability): `dist[i][j] = dist[i][j] || (dist[i][k] && dist[k][j])`
- Negative cycle detection: if `dist[i][i] < 0` after all passes, negative cycle exists
- Minimum bottleneck path: `dist[i][j] = min(dist[i][j], max(dist[i][k], dist[k][j]))`

---

## SECTION 11 — TEMPLATE 7: DIJKSTRA WITH BITMASK STATE (BFS + STATE)

For problems where the "state" is not just the current node but includes extra information like "which keys have been collected" or "how many special moves remain."

```cpp
// LC 864 — Shortest Path to Get All Keys
// State = (row, col, keys_collected_bitmask)

int shortestPathAllKeys(vector<string>& grid) {
    int m = grid.size(), n = grid[0].size();
    int totalKeys = 0;
    int startR = 0, startC = 0;

    // Find start position and count keys
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '@') { startR = i; startC = j; }
            if (islower(grid[i][j])) totalKeys++;
        }
    }

    int fullMask = (1 << totalKeys) - 1;  // all keys collected

    // BFS with state: (steps, row, col, key_mask)
    // Use BFS (not Dijkstra) since all edges have weight 1
    // dist[r][c][mask] = minimum steps to reach (r,c) with key_mask collected
    vector<vector<vector<int>>> dist(m, vector<vector<int>>(n, vector<int>(1<<totalKeys, INT_MAX)));
    dist[startR][startC][0] = 0;

    queue<tuple<int,int,int>> q;  // (row, col, mask)
    q.push({startR, startC, 0});

    int steps = 0;
    int dirs[4][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!q.empty()) {
        int sz = q.size();
        steps++;

        while (sz--) {
            auto [r, c, mask] = q.front(); q.pop();

            for (auto& d : dirs) {
                int nr = r + d[0], nc = c + d[1];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (grid[nr][nc] == '#') continue;

                int newMask = mask;
                char cell = grid[nr][nc];

                // If it's a lock, check if we have the key
                if (isupper(cell)) {
                    int keyBit = 1 << (cell - 'A');
                    if (!(mask & keyBit)) continue;  // don't have key, can't enter
                }

                // If it's a key, collect it
                if (islower(cell)) {
                    newMask |= (1 << (cell - 'a'));
                }

                if (newMask == fullMask) return steps;  // found all keys!

                if (dist[nr][nc][newMask] == INT_MAX) {
                    dist[nr][nc][newMask] = steps;
                    q.push({nr, nc, newMask});
                }
            }
        }
    }

    return -1;
}
```

**WHY BFS not Dijkstra here:** All moves cost 1. BFS gives optimal for unit-weight graphs. Dijkstra would also work but adds unnecessary O(log) overhead.

**The key insight for state-space BFS:** Instead of `visited[node]`, we use `visited[node][state]`. The state captures all relevant information that affects future decisions. For key collection, `state = bitmask of collected keys` because:
- Same (row, col) with different key masks → different effective positions (can/can't open locks)
- Same (row, col) with same key mask → truly the same state, no need to revisit

---

## SECTION 12 — TEMPLATE 8: SHORTEST PATH WITH CONSTRAINTS (NUMBER OF WAYS)

```cpp
// LC 1976 — Number of Ways to Arrive at Destination
// Count paths achieving the shortest distance (not just the distance itself)
// Add a "count" array alongside "dist" array in Dijkstra

int countPaths(int n, vector<vector<int>>& roads) {
    const int MOD = 1e9 + 7;

    vector<vector<pair<int,int>>> adj(n);
    for (auto& r : roads) {
        adj[r[0]].push_back({r[1], r[2]});
        adj[r[1]].push_back({r[0], r[2]});
    }

    vector<long long> dist(n, LLONG_MAX);
    vector<long long> ways(n, 0);  // number of shortest paths to this node
    dist[0] = 0;
    ways[0] = 1;

    priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
    pq.push({0, 0});

    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();

        if (d > dist[u]) continue;  // stale

        for (auto [v, w] : adj[u]) {
            long long newDist = dist[u] + w;

            if (newDist < dist[v]) {
                dist[v] = newDist;
                ways[v] = ways[u];  // new shortest path — inherit count
                pq.push({dist[v], v});
            } else if (newDist == dist[v]) {
                ways[v] = (ways[v] + ways[u]) % MOD;  // same length — add count
                // Note: don't push to pq again — dist[v] didn't change
            }
        }
    }

    return ways[n-1];
}
```

**WHY the two cases for updating `ways`:**
- `newDist < dist[v]`: Found a shorter path. The old count is irrelevant — reset `ways[v] = ways[u]`.
- `newDist == dist[v]`: Found another path of the same length. The count adds: `ways[v] += ways[u]`.

This is a standard technique: run Dijkstra for distances, then accumulate counts by following the same relaxation logic.

---

## SECTION 13 — COMPLEXITY TABLE

| Algorithm | Time | Space | Negative Weights | Negative Cycles | All-Pairs | Notes |
|-----------|------|-------|-----------------|-----------------|-----------|-------|
| BFS | O(V+E) | O(V) | No | No | No | Only unweighted |
| Dijkstra (binary heap) | O((V+E) log V) | O(V) | No | No | No | Sparse graphs |
| Dijkstra (Fibonacci heap) | O(V log V + E) | O(V) | No | No | No | Theory only |
| Bellman-Ford | O(V×E) | O(V) | Yes | Detects | No | Slow but general |
| SPFA (Bellman-Ford queue) | O(kE) avg | O(V) | Yes | Detects | No | O(VE) worst case |
| 0-1 BFS | O(V+E) | O(V) | No (weights 0 or 1) | No | No | Deque-based |
| Floyd-Warshall | O(V³) | O(V²) | Yes | Detects | Yes | Use for N ≤ 300 |
| Johnson's | O(V² log V + VE) | O(V²) | Yes | Detects | Yes | Sparse all-pairs |

**Practical bounds for competitive programming:**
- Dijkstra: V, E up to 10^5 easily, 10^6 is fine
- Bellman-Ford: V up to 10^3, E up to 10^4 (10^7 ops)
- Floyd-Warshall: V up to 300-500 (5×10^7 ops)
- 0-1 BFS: same as BFS, handles grids up to 10^6 cells

---

## SECTION 14 — COMMON MISTAKES

### Mistake 1: Using Dijkstra with negative weights

```cpp
// WRONG — Dijkstra with negative edge
// edges: A→B weight -5, A→C weight 2, C→B weight 1
// Dijkstra "finalizes" B at dist[A]+(-5) = -5 early
// But then discovers A→C→B = 3 — which it ignores because B is "done"

// The stale-entry check if (d > dist[u]) continue does NOT help here
// because the node is actually popped with the wrong optimal distance

// CORRECT — use Bellman-Ford for negative weights
// OR: if no negative cycles and you need SSSP from one source,
//     check if Bellman-Ford with the K-stops trick applies
```

---

### Mistake 2: Integer overflow in Dijkstra

```cpp
// WRONG — dist initialized to INT_MAX, then dist[u] + w overflows
vector<int> dist(n, INT_MAX);
// ...
if (dist[u] + w < dist[v])  // BUG: if dist[u] = INT_MAX, dist[u]+w overflows!

// CORRECT — either use long long, or check before adding
vector<long long> dist(n, LLONG_MAX);
if (dist[u] != LLONG_MAX && dist[u] + w < dist[v])

// OR: initialize to a large but safe sentinel
vector<int> dist(n, 1e9);  // safe for typical weight constraints ≤ 10^4 * 10^4 = 10^8
```

---

### Mistake 3: Missing the `temp` copy in Bellman-Ford with edge constraints

```cpp
// WRONG — single pass uses updates from the SAME pass
for (int i = 0; i <= k; i++) {
    for (auto& f : flights) {
        if (dist[f[0]] != INT_MAX)
            dist[f[1]] = min(dist[f[1]], dist[f[0]] + f[2]);  // BUG: in-place update
    }
}

// CORRECT — copy ensures we only use distances from the PREVIOUS pass
for (int i = 0; i <= k; i++) {
    vector<int> temp = dist;  // ← copy first
    for (auto& f : flights) {
        if (dist[f[0]] != INT_MAX)
            temp[f[1]] = min(temp[f[1]], dist[f[0]] + f[2]);  // read dist, write temp
    }
    dist = temp;
}
```

---

### Mistake 4: 0-1 BFS: pushing to wrong end of deque

```cpp
// WRONG — backwards: 0-cost pushed to back, 1-cost pushed to front
// This destroys the sorted-by-distance invariant
if (cost == 0) dq.push_back({nr, nc});   // WRONG
else           dq.push_front({nr, nc});   // WRONG

// CORRECT:
if (cost == 0) dq.push_front({nr, nc});  // 0-cost: no distance increase → front
else           dq.push_back({nr, nc});   // 1-cost: distance increases → back
```

---

### Mistake 5: Floyd-Warshall loop order (k must be outermost)

```cpp
// WRONG — k is not the outermost loop
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        for (int k = 0; k < n; k++)  // ← k innermost: WRONG!
            dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);

// CORRECT — k must be OUTERMOST
// Reason: dp[i][j][k] depends on dp[i][k][k-1] and dp[k][j][k-1]
// If k is inner, when computing dist[i][j] for k=5, dist[i][5] might have been updated
// using k=5 as intermediate already — violating the sub-problem structure
for (int k = 0; k < n; k++)          // ← k outermost: CORRECT
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);
```

---

### Mistake 6: State-space Dijkstra — forgetting to include extra state in visited check

```cpp
// WRONG — only tracking visited by node, not by (node, state)
vector<bool> visited(n, false);
// When multiple states can reach the same node, this marks node as done
// too early, missing paths with different key/fuel states

// CORRECT — track visited by (node, state)
// For BFS with bitmask state:
vector<vector<int>> dist(m, vector<int>(n, vector<int>(1<<K, INT_MAX)));
// Never need explicit "visited" — dist[node][state] serves as the visited check
if (dist[nr][nc][newMask] == INT_MAX) { dist[nr][nc][newMask] = steps; }
```

---

## SECTION 15 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Network Delay Time | 743 | Pure Dijkstra — implement from scratch |
| 2 | Path with Maximum Probability | 1514 | Max-heap Dijkstra, multiply not add |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Cheapest Flights Within K Stops | 787 | Bellman-Ford with temp copy + K passes |
| 4 | Path With Minimum Effort | 1631 | Grid Dijkstra with max-edge combination |
| 5 | Swim in Rising Water | 778 | Grid Dijkstra (also DSU approach from P13) |
| 6 | Shortest Path in Binary Matrix | 1091 | BFS — all edges weight 1, not Dijkstra |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Find the City With Smallest Number of Neighbors | 1334 | Floyd-Warshall all-pairs |
| 8 | Shortest Path to Get All Keys | 864 | BFS with bitmask state |
| 9 | Minimum Cost to Make at Least One Valid Path | 1368 | 0-1 BFS on grid |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Number of Ways to Arrive at Destination | 1976 | Dijkstra + path counting |
| 11 | Shortest Path Visiting All Nodes | 847 | BFS + bitmask (TSP on graph) |

---

**Solve in this order.** LC 743 is mandatory — Dijkstra template must be muscle memory. LC 787 is THE interview problem for this pattern. LC 1368 is the definitive 0-1 BFS problem.

---

## SECTION 16 — PATTERN CONNECTIONS

**Shortest path connects to:**

1. **Pattern 10 (Heap):** Dijkstra's priority queue is a direct application of the min-heap. The stale-entry pattern (`if d > dist[u]: skip`) is a form of lazy deletion identical to the heap lazy deletion template in Pattern 10.

2. **Pattern 13 (DSU):** Swim in Rising Water (LC 778) can be solved with BOTH Dijkstra (Template 2 above) AND DSU (Pattern 13 Template). Understanding both solutions deepens insight into what "minimum bottleneck path" means.

3. **Pattern 31 (MST):** Prim's algorithm is essentially Dijkstra but tracking the minimum EDGE weight to the growing MST instead of the minimum TOTAL path weight. The code structures are nearly identical.

4. **Pattern 24 (Bitmask DP):** Shortest Path Visiting All Nodes (LC 847) combines BFS with bitmask state — a perfect intersection of this pattern and Bitmask DP (Pattern 24). The state space is `O(V × 2^V)`.

5. **Pattern 19 (Knapsack / DP):** Cheapest Flights Within K Stops is both a Bellman-Ford problem and a DP problem: `dp[k][v]` = cheapest price to reach v using exactly k edges. Recognizing this dual nature is a sign of maturity.

6. **Pattern 12 (BFS/DFS):** BFS is the degenerate case of Dijkstra when all weights are equal. When you see "shortest path on an unweighted graph," resist the urge to use Dijkstra — BFS is O(V+E) vs O((V+E) log V).

**Decision tree for shortest path:**
```
Shortest path problem?
├─ All weights = 1 (or unweighted)?
│   └─ BFS (Pattern 12)
├─ Weights ∈ {0, 1}?
│   └─ 0-1 BFS (deque)
├─ Non-negative weights?
│   ├─ Single source → Dijkstra
│   └─ All pairs, small N (≤300) → Floyd-Warshall
├─ Negative weights, no negative cycles?
│   ├─ Single source, K-step constraint → Bellman-Ford (K passes)
│   └─ Single source, no constraint → Bellman-Ford (V-1 passes)
└─ State = (node + extra info)?
    └─ BFS/Dijkstra with state space expansion
```

---

## SECTION 17 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 743 — Network Delay Time

**Interviewer:** "N network nodes labeled 1 to N. Some nodes send signals via directed edges with travel times. Given source node k, find the time until ALL nodes have received the signal."

**Candidate:**

*[First 2 minutes — analyze]*
> "This is a single-source shortest path problem. I need the shortest time from k to every other node. The answer is the maximum of all these shortest times, since ALL nodes must receive the signal. All edge weights are positive, so Dijkstra applies."

*[State assumptions explicitly]*
> "I'll assume the graph is connected — if not, return -1. All weights are positive integers. The graph is directed."

*[Code — 4 minutes]*
```cpp
// [Writes Template 1 from Section 5]
```

*[Complexity]*
> "O((V+E) log V) for Dijkstra with a binary heap. Space O(V+E) for adjacency list and O(V) for dist array."

*[Follow-up]: "What if some edges have negative weights?"*
> "Dijkstra breaks. I'd switch to Bellman-Ford: O(V×E). Run V-1 relaxation passes over all edges. If the graph is small, this is fine. If performance is critical and there are no negative cycles, SPFA (queue-based Bellman-Ford) is faster in practice."

**RED FLAGS:**
- Not knowing the stale-entry skip (`if d > dist[u] continue`) — shows copy-paste of Dijkstra without understanding
- Forgetting to handle disconnected graphs (returning max before checking for INT_MAX)
- Using BFS instead of Dijkstra (BFS doesn't account for edge weights)

---

### Interview Simulation 2: LC 787 — Cheapest Flights Within K Stops

**Interviewer:** "Find the cheapest price from src to dst with at most K stops."

**Candidate:**

*[First 2 minutes — catch the constraint]*
> "The key constraint is 'at most K stops' = K+1 edges. Standard Dijkstra doesn't natively handle edge-count constraints. Two approaches: (1) Modified Dijkstra with state (node, stops_remaining), or (2) K+1 passes of Bellman-Ford. I'll go with Bellman-Ford — it's simpler to implement correctly."

*[Critical insight to state before coding]*
> "Bellman-Ford normally runs V-1 passes. Here, one pass = one more edge used. After i passes, dist[v] = cheapest path from src to v using at most i edges. I need K+1 passes for K stops."

*[Most important thing: the temp copy]*
> "I MUST use a temp copy of dist in each pass. Otherwise, in a single pass, I might chain multiple edges — which would mean I used more than 1 edge in that 'pass', violating the K constraint."

*[Code — 5 minutes]*
```cpp
// [Writes Template 3 from Section 7]
```

**RED FLAGS:**
- Missing the temp copy (incorrect solution that passes basic tests but fails edge cases)
- Confusing "K stops" with "K edges" (K stops = K+1 edges — off by one is instant rejection)
- Using Dijkstra without state expansion (would give shortest price but possibly with more than K stops)

---

### Interview Simulation 3: Design question — When would you use 0-1 BFS over Dijkstra?

**Interviewer:** "You have a grid where moving in a cell's arrow direction costs 0 and against it costs 1. What algorithm would you use? Why not Dijkstra?"

**Candidate:**

> "Dijkstra would work correctly here — it handles non-negative weights. But since weights are only 0 or 1, there's a faster algorithm: 0-1 BFS using a deque. Time complexity drops from O(N² log N) to O(N²) where N is the number of cells."

> "The key insight: in BFS, the queue is sorted by distance. In 0-1 BFS, we maintain this invariant using a deque. A 0-cost move doesn't increase distance, so we push to the front — keeping it at the same 'level' as the current node. A 1-cost move increases distance, so we push to the back — moving it to the 'next level'. The front of the deque always has the minimum distance."

> "The deque replaces the priority queue. All BFS operations (push, pop) are O(1) vs O(log N) for a heap."

*[Code - writes Template 5 from Section 9]*

*[Follow-up]: "Could you use this for weights 0 and 2?"*
> "Not directly. 0-1 BFS relies on weights being consecutive integers starting at 0. For {0, 2}, you'd use Dial's algorithm (bucket queue with d+1 buckets), or just use Dijkstra. The deque invariant requires that the difference between adjacent elements in the deque is at most 1."

**RED FLAGS:**
- Not knowing 0-1 BFS exists and defaulting to "just use Dijkstra" (misses O(log N) factor improvement)
- Getting the push direction backwards (0-cost to back, 1-cost to front — completely wrong)
- Not being able to explain WHY the deque maintains sorted order

---

## SECTION 18 — MENTAL MODEL CHECKPOINT

Seven questions. Answer each before reading the answer.

---

**Q1: Why does Dijkstra fail on negative-weight edges? Give a concrete counterexample.**

> Consider: nodes A, B, C. Edges: A→B(w=3), A→C(w=4), C→B(w=-2).
>
> Real shortest path A→B: via A→C→B = 4+(-2) = 2.
> Dijkstra: dist[A]=0. Pop A. Set dist[B]=3, dist[C]=4. Pop B (dist=3 — it's the minimum). Mark B as finalized. Pop C. Try to update B: dist[C]+(-2)=2 < dist[B]=3 — but B is already "finalized"!
>
> With the stale-entry approach (no explicit "visited" marking), Dijkstra would push (2, B) again and pop it, then try to update neighbors of B with the correct dist[B]=2. So some implementations "work" for small negatives. But this is wrong in general — it can lead to infinite loops with negative cycles or incorrect results for longer chains.

---

**Q2: In Bellman-Ford for LC 787, why do we need K+1 passes instead of K passes?**

> "K stops" means at most K intermediate nodes between source and destination. That means the path has at most K+1 edges (K stops = K intermediate nodes = K+1 edge segments: src→stop1→stop2→...→stopK→dst).
>
> Each Bellman-Ford pass adds at most one more edge to the paths. After i passes, we have optimal paths using at most i edges. For paths of at most K+1 edges, we need K+1 passes.
>
> Mnemonic: K stops → K+1 edges → K+1 passes.

---

**Q3: What is the time complexity of Dijkstra with a binary heap? Why is it O((V+E) log V) and not O(E log V)?**

> Each edge (u, v) causes at most one push to the priority queue when dist[v] is updated. So total pushes ≤ E. Each push costs O(log heap_size). The heap can have at most E entries (one per push), so heap_size ≤ E, giving O(log E) = O(log V²) = O(log V) per operation.
>
> Total for edges: O(E log V).
> Each node is popped at most once (after which stale entries are discarded): V pops at O(log V) each = O(V log V).
>
> Total: O((V+E) log V). For dense graphs E = O(V²), this is O(V² log V). For sparse graphs E = O(V), this is O(V log V).

---

**Q4: In Floyd-Warshall, why must k be the outermost loop?**

> Floyd-Warshall builds solutions inductively: `dp[i][j][k]` = shortest path from i to j using only nodes {0..k-1} as intermediates.
>
> The transition: `dp[i][j][k] = min(dp[i][j][k-1], dp[i][k][k-1] + dp[k][j][k-1])`
>
> This says: "the shortest path from i to j through nodes {0..k} is either: (a) the best path not using k, or (b) the best path from i to k, then from k to j — both using only {0..k-1} as intermediates."
>
> For this to be correct, when we compute `dp[i][j][k]`, we need `dp[i][k][k-1]` and `dp[k][j][k-1]` — the k-1 layer values. If k is the outermost loop, these are computed in the previous iteration of k. If k is inner, the k-th row and column might already be updated with the k-layer values, breaking the recurrence.

---

**Q5: When should you use 0-1 BFS vs Dijkstra? Is 0-1 BFS always better when weights are 0 or 1?**

> For weights strictly in {0, 1}, 0-1 BFS is asymptotically better: O(V+E) vs O((V+E) log V). So yes, when weights are 0/1, prefer 0-1 BFS.
>
> Practically, if E is small relative to V, the log factor doesn't matter much. But in a grid of size N×N (V=N², E=4N²), the difference is O(N²) vs O(N² log N) — meaningful for large N.
>
> The deque approach requires that weights are integers in {0, 1}. For {0, w} with w > 1, you'd need Dial's algorithm or just use Dijkstra.

---

**Q6: In the "number of ways to reach destination at minimum cost" problem, why don't we push to the priority queue again when we find an equal-distance path?**

> When `newDist == dist[v]`, we've found another path of the same length to v. We update `ways[v] += ways[u]` to count this additional path. But `dist[v]` hasn't changed — it's still the same optimal distance.
>
> If we pushed (dist[v], v) again, we'd pop v again later and re-process its neighbors with the same distance. This would cause `ways` to be incremented multiple times for the same path. It's not necessary to push because the distance update doesn't change what neighbors see when they compute paths through v.
>
> In the Dijkstra structure, a node u is effectively "processed" once it's popped with its optimal distance. Any further pops of u with the same distance (if we pushed again) are handled correctly but are redundant.

---

**Q7: Shortest Path Visiting All Nodes (LC 847) — why is the state space O(V × 2^V) and not just O(V)?**

> Simply tracking the current node is insufficient. Consider two paths to the same node — one has collected nodes {A, B, C} and another has collected {A, B, D}. From the same physical node, these states have different futures (what keys remain to collect). They cannot be merged.
>
> The full state is (current_node, set_of_visited_nodes). The set of visited nodes is represented as a bitmask of V bits → 2^V possible subsets. Combined with V possible current nodes → O(V × 2^V) states.
>
> For V ≤ 12 (the problem's constraint), 2^12 × 12 = 49,152 states. BFS over this state space is perfectly feasible.

---

## SECTION 19 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// DIJKSTRA — Standard template (memorize this completely)
// ─────────────────────────────────────────────────────────────────
auto dijkstra = [&](int src, int n, vector<vector<pair<int,int>>>& adj) {
    vector<long long> dist(n, LLONG_MAX);
    dist[src] = 0;
    priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;  // stale entry
        for (auto [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
};

// ─────────────────────────────────────────────────────────────────
// DIJKSTRA ON GRID — (row, col) as nodes
// ─────────────────────────────────────────────────────────────────
// dist[r][c] = min cost to reach (r,c)
// pq: (cost, row, col) with greater<tuple<int,int,int>>
// Edge cost function: depends on problem (sum, max, 0-1)

// ─────────────────────────────────────────────────────────────────
// BELLMAN-FORD with K passes (Cheapest Flights pattern)
// ─────────────────────────────────────────────────────────────────
vector<int> dist(n, INT_MAX); dist[src] = 0;
for (int i = 0; i <= k; i++) {   // K+1 passes for K stops
    vector<int> temp = dist;      // ← COPY IS ESSENTIAL
    for (auto& [u, v, w] : edges)
        if (dist[u] != INT_MAX)
            temp[v] = min(temp[v], dist[u] + w);
    dist = temp;
}

// ─────────────────────────────────────────────────────────────────
// 0-1 BFS — deque instead of priority_queue
// ─────────────────────────────────────────────────────────────────
vector<int> dist(n, INT_MAX); dist[src] = 0;
deque<int> dq; dq.push_back(src);
while (!dq.empty()) {
    int u = dq.front(); dq.pop_front();
    for (auto [v, w] : adj[u]) {   // w is 0 or 1
        if (dist[u] + w < dist[v]) {
            dist[v] = dist[u] + w;
            if (w == 0) dq.push_front(v);  // 0-cost: FRONT
            else        dq.push_back(v);   // 1-cost: BACK
        }
    }
}

// ─────────────────────────────────────────────────────────────────
// FLOYD-WARSHALL — all-pairs
// ─────────────────────────────────────────────────────────────────
vector<vector<int>> dist(n, vector<int>(n, INT_MAX/2));
for (int i = 0; i < n; i++) dist[i][i] = 0;
for (auto& [u,v,w] : edges) { dist[u][v]=min(dist[u][v],w); dist[v][u]=min(dist[v][u],w); }
for (int k = 0; k < n; k++)   // k OUTERMOST
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);

// ─────────────────────────────────────────────────────────────────
// DIJKSTRA — PATH COUNTING variant
// ─────────────────────────────────────────────────────────────────
vector<long long> dist(n, LLONG_MAX), ways(n, 0);
dist[src]=0; ways[src]=1;
// In relaxation:
if (newDist < dist[v])       { dist[v]=newDist; ways[v]=ways[u]; pq.push({dist[v],v}); }
else if (newDist == dist[v]) { ways[v]=(ways[v]+ways[u])%MOD; }

// ─────────────────────────────────────────────────────────────────
// COMMON PITFALLS QUICK REFERENCE
// ─────────────────────────────────────────────────────────────────
// 1. Always: if (d > dist[u]) continue;        // skip stale
// 2. K stops = K+1 edges = K+1 Bellman passes
// 3. Floyd: INT_MAX/2 to avoid overflow on addition
// 4. Floyd: k outermost loop (NEVER innermost)
// 5. 0-1 BFS: 0-cost → front, 1-cost → back
// 6. Probability: max-heap, multiply (not add), max (not min) comparison
// 7. State Dijkstra: dist[node][state], not just dist[node]
```

---

## SECTION 20 — WHAT TO DO AFTER THIS PATTERN

1. **Immediately solve (in order):** LC 743 → 1514 → 787 → 1631 → 1091 → 778 → 1334 → 864 → 1368 → 1976 → 847

2. **Solidify Dijkstra as muscle memory:** Write the full Dijkstra template from memory without looking at notes. Time yourself — target under 3 minutes. This is the most important implementation skill in the entire graph section.

3. **Connect to Pattern 31 (MST):** After finishing this pattern, Kruskal's MST (Pattern 31) will feel natural — it's "sort edges by weight + DSU" while Dijkstra is "greedy by distance + heap". The underlying greedy insight is the same.

4. **Contest readiness:** In Codeforces Div 2 C/D, the shortest path problems almost always have a twist — state expansion (extra dimension), non-standard combination function (max, multiply), or K-constraint. Practice identifying the twist before you start coding.

5. **The "graph as generalization" perspective:** After completing Patterns 12-14, you should be able to see ANY graph problem and immediately identify: connectivity check (DSU/BFS), shortest path (Dijkstra/BF/BFS), topological order (Pattern 30), or spanning tree (Pattern 31). The graph section is now complete.

---

## SECTION 21 — SIGN-OFF CRITERIA

### Tier 1 — Can implement
You can write Dijkstra from scratch (adjacency list, priority_queue, stale skip, dist array) in under 3 minutes without reference.

### Tier 2 — Can apply
You solve LC 743 and LC 787 cleanly: Dijkstra for 743, Bellman-Ford with temp copy for 787. You correctly handle the K stops → K+1 passes conversion. You never overflow dist arrays.

### Tier 3 — Can adapt
You solve LC 1631 (max-edge Dijkstra), LC 1368 (0-1 BFS), and LC 1334 (Floyd-Warshall) in under 30 minutes each. You know the Floyd-Warshall loop order and the 0-1 BFS push direction without looking them up.

### Tier 4 — Can combine
You solve LC 864 (BFS + bitmask state) and LC 847 (BFS + bitmask, TSP on graph) in under 40 minutes. You can explain when to use each algorithm and why Dijkstra fails on negative weights using a counterexample.

**Sign-off threshold:** Solve 8 of 11 problems, including LC 743 (Dijkstra template), LC 787 (Bellman-Ford K stops), and at least one of {LC 1368, LC 864}. Sign off only when Dijkstra feels as automatic as BFS.

---

*Pattern 14 Complete — Graphs: Shortest Path*
*Next: Pattern 15 — Backtracking*
