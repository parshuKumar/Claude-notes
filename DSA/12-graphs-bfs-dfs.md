# Pattern 12 — Graphs: BFS and DFS
## Difficulty Level: CORE
## Interview Frequency: ALWAYS
## Estimated time to master: 5-6 days
## Problems in this set: 12 problems

---

## THE GRANDMASTER'S HONEST TAKE

Graphs are the most important pattern in the entire curriculum for interview performance. More hard problems reduce to graphs than any other single pattern. Trees are special cases of graphs. Intervals can be modeled as graphs. Shortest path, connectivity, cycle detection, dependency ordering — all graphs.

**The brutal truth: at your 1600 level, you can probably solve Number of Islands (LC 200). The problem is that you likely "solve" it by hacking together a DFS without a clean template. You don't have a systematic approach that generalizes. You probably cannot:**
- Implement cycle detection in directed graphs (3-color marking) from memory
- Build an adjacency list in C++ from an edge list in under 2 minutes
- Correctly implement multi-source BFS (starting from ALL sources before the first BFS step)
- Reason about WHY BFS gives shortest paths on unweighted graphs
- Solve Word Ladder (BFS on an implicit graph where neighbors must be computed)

**The specific gap between 1600 and 2000:**
- At 1600: you recognize grid problems as "probably DFS/BFS" and fumble through them
- At 2000: you see "connectivity" → instantaneous BFS/DFS template selection, you see "shortest path unweighted" → BFS, you see "all paths/components" → DFS, you see "cycle in directed graph" → 3-color DFS, you see "minimum steps to reach from multiple starting points" → multi-source BFS

**After this document, you will:**
- Have 9 distinct graph templates memorized — grid BFS, grid DFS, graph BFS, graph DFS, undirected cycle detection, directed cycle detection (3-color), multi-source BFS, bipartite check, implicit graph BFS
- Know the one most critical BFS rule: **mark nodes as visited BEFORE pushing to the queue**, not after popping
- Understand WHEN BFS gives the optimal solution (unweighted shortest path) and when DFS is better (connectivity, all paths, cycle detection)
- Write adjacency list construction from an edge list in 3 lines of C++
- Solve Word Ladder (implicit graph BFS) — the hardest graph pattern that appears in almost every senior-level interview

**The interview data:**
- Number of Islands (LC 200) appears as a warm-up or screening problem at essentially every tech company
- Course Schedule (LC 207) — directed cycle detection — is a top-10 most-asked problem at Google and Amazon
- Word Ladder (LC 127) appears in 20%+ of Google L4/L5 rounds and is a known filtering problem
- Rotting Oranges (LC 994) is a favorite at Amazon, Uber, and Microsoft for its multi-source BFS structure
- Pacific Atlantic Water Flow (LC 417) tests the "reverse direction" insight — asked repeatedly at Google

---

## ELI5 — GRAPH AS A MAP OF CONNECTIONS

### Graphs as Maps

A graph is just a map. Cities are NODES (also called vertices). Roads between cities are EDGES. Your GPS navigation system is solving a graph problem every time you get directions.

**Undirected graph:** the road between cities goes both ways. You can drive from A to B or B to A.
**Directed graph:** a one-way street. You can go from A to B but not necessarily B to A.

A grid (matrix) is a special kind of graph where every cell is a node and each cell connects to its 4 (or 8) neighbors.

### BFS vs DFS — The Explorer Metaphor

Imagine you're exploring a cave system with multiple branching tunnels.

**DFS (Depth-First Search):** You take one tunnel and go as deep as possible before backtracking. You mark where you've been with chalk. If you reach a dead end, you backtrack to the last junction and try the next tunnel. You explore FULLY before moving on to siblings.

**BFS (Breadth-First Search):** You send scouts forward simultaneously. All tunnels 1 step away are explored before any tunnel 2 steps away. All tunnels 2 steps away before 3 steps away. You explore BROADLY — layer by layer.

**Why BFS gives shortest paths:** Since you expand outward in layers (1 step away, then 2, then 3...), the FIRST time you reach any destination, you reached it via the FEWEST steps. The moment a BFS finds the destination, that's the shortest path. DFS might find a long winding path to the destination before finding the short path.

**Why DFS handles connected components:** DFS from a starting node will reach EVERY node in the same connected component (following all edges deeply). Multiple DFS calls, each from an unvisited node, identify all distinct components. The count of DFS calls = number of components.

### Multi-Source BFS — The Spreading Pandemic

Regular BFS: one starting node spreads outward. Multi-source BFS: MULTIPLE starting nodes spread outward simultaneously, like a pandemic starting in multiple cities at once.

Rotting Oranges: all rotten oranges (multiple sources) spread rot to fresh neighbors simultaneously each minute. To model this correctly: add ALL rotten oranges to the BFS queue BEFORE starting. Then process the queue level by level. Each level = one minute. This is not multiple separate BFS runs — it's ONE BFS starting from all sources at once.

### Directed Cycle Detection — The Dependency Problem

Imagine course prerequisites: Course A requires Course B, Course B requires Course C. If Course C requires Course A, you have a cycle — you can never take any of them.

3-color marking: WHITE = haven't started this course, GRAY = currently taking this course (it's in our current "path"), BLACK = finished this course (all its prerequisites are done). If during enrollment of a GRAY course, we encounter another GRAY course — we're trying to enroll in a course we're already enrolled in → cycle detected.

---

## THE PATTERN — FORMALLY DEFINED

### Graph Representations in C++:

```cpp
// Adjacency List (most common for sparse graphs — interviews and CP):
int n = 5;  // 5 nodes (0-indexed: 0, 1, 2, 3, 4)
vector<vector<int>> adj(n);

// Build from edge list (undirected):
vector<pair<int,int>> edges = {{0,1},{1,2},{2,3},{3,4}};
for (auto& [u, v] : edges) {
    adj[u].push_back(v);
    adj[v].push_back(u);  // both directions for undirected
}

// Build from edge list (directed):
for (auto& [u, v] : edges) {
    adj[u].push_back(v);  // only u → v
}

// Adjacency Matrix (for dense graphs or when checking if edge exists in O(1)):
vector<vector<int>> matrix(n, vector<int>(n, 0));
matrix[u][v] = 1;  // directed edge u → v
matrix[u][v] = matrix[v][u] = 1;  // undirected

// Grid as implicit graph (no explicit adj list needed):
// Each cell (r, c) connects to (r±1, c) and (r, c±1)
int dr[] = {0, 0, 1, -1};  // 4-directional movement
int dc[] = {1, -1, 0, 0};

// For 8-directional (diagonal moves allowed):
int dr8[] = {0, 0, 1, -1, 1, 1, -1, -1};
int dc8[] = {1, -1, 0, 0, 1, -1, 1, -1};
```

### Sub-Patterns This Covers:

1. **Grid BFS:** Flood fill, shortest path on grid, multi-source grid spread
2. **Grid DFS:** Connected components, area calculation, path finding
3. **Graph BFS:** Shortest path (unweighted), level-by-level processing
4. **Graph DFS:** All paths, connectivity, cycle detection
5. **Undirected cycle detection:** Parent tracking in DFS
6. **Directed cycle detection:** 3-color (white/gray/black) DFS
7. **Multi-source BFS:** Simultaneous spread from multiple starting nodes
8. **Bipartite check:** 2-coloring with BFS/DFS
9. **Implicit graph BFS:** Word Ladder, lock combinations

---

## WHEN TO RECOGNIZE THIS PATTERN

**Signal 1:** "Number of connected components" or "count distinct groups" or "how many islands" or "how many provinces"
→ DFS or BFS on a graph/grid. Count how many times you start a new traversal on an unvisited node. Each new start = one new component. For grids: iterate all cells, start DFS/BFS from each unvisited '1' (or relevant) cell.

**Signal 2:** "Find the shortest path / minimum steps / minimum moves" on an UNWEIGHTED graph or grid
→ BFS. BFS guarantees shortest path on unweighted graphs because it explores by layers (level = distance). DFS does NOT guarantee shortest path. If weights exist, use Dijkstra (Pattern 14). If weights are 0 or 1, use 0-1 BFS (deque-based, Pattern 14). If all weights equal 1: BFS.

**Signal 3:** "Detect if a cycle exists" in a graph
→ Undirected graph: DFS with parent tracking (if you reach a visited node that isn't your direct parent → cycle). Directed graph: DFS with 3-color marking (if you reach a GRAY node → cycle). Also achievable via Kahn's topological sort (Pattern 30): if not all nodes are processed → cycle.

**Signal 4:** "Spread from multiple starting points simultaneously" or "time to infect/rot all cells" or "walls and gates distance"
→ Multi-source BFS. Push ALL starting sources into the queue BEFORE running BFS. The BFS distance represents the minimum distance from any source.

**Signal 5:** "Can you color the graph with 2 colors such that no two adjacent nodes share a color?" or "check if the graph is bipartite" or "divide into two groups"
→ 2-coloring BFS/DFS. Attempt to assign alternating colors (0/1) to nodes. If any adjacent pair shares a color → not bipartite.

**Signal 6:** "Clone a graph" or "deep copy" or "visit all nodes while tracking state"
→ DFS with a hash map from original node to clone. The visited set IS the clone map.

**Signal 7:** "Find if transformation X can reach transformation Y in minimum steps" where "neighbors" are implicitly defined by a rule
→ BFS on an implicit graph. Compute neighbors on-the-fly (don't pre-build the graph). Word Ladder: neighbors of a word = all valid words differing by one character.

**Signal 8:** "Border-connected" problems or "safe from border" or "connected to ocean/source"
→ Reverse BFS/DFS. Instead of asking "can each cell reach the border?", start BFS FROM the border inward. Much more efficient: one BFS from all border cells instead of one BFS per cell.

### Signals that LOOK like Graph but are NOT:

**Fake signal 1:** "Count subarrays with at most K distinct elements"
→ Sliding window (Pattern 03), not graph. The "distinct" constraint is a window invariant, not a connectivity problem.

**Fake signal 2:** "Find the maximum path sum in a binary tree"
→ Tree DFS (Pattern 08), not general graph BFS/DFS. Tree DFS uses the "return value from subtree" pattern with no visited array needed.

**Fake signal 3:** "Reach index j from index i if nums[i] <= nums[j]"
→ If it asks for the COUNT of reachable pairs, it's DP or sorting, not graph traversal. The "graph" would have O(n²) edges — BFS/DFS would be O(n²). Look for DP or sorting approaches instead.

---

## THE TEMPLATES (C++)

### Template 1: BFS on Grid — The Fundamental Grid Traversal

```cpp
// Number of Islands (LC 200): count connected components of '1' cells on a grid.
// Core template: BFS from each unvisited '1', marking visited as we go.
//
// CRITICAL RULE: mark visited BEFORE pushing to queue, not after popping.
// Why? If you mark when popping, multiple copies of the same cell can queue up
// before any copy is popped and marked → duplicate processing, wrong counts.

int numIslands(vector<vector<char>>& grid) {
    int m = grid.size(), n = grid[0].size();
    int islands = 0;
    
    // 4-directional movement arrays (used in every grid problem)
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};
    
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == '1') {
                islands++;
                // BFS from this unvisited land cell
                queue<pair<int,int>> q;
                q.push({i, j});
                grid[i][j] = '0';  // mark visited by modifying grid (save memory vs separate array)
                
                while (!q.empty()) {
                    auto [r, c] = q.front(); q.pop();
                    
                    for (int d = 0; d < 4; d++) {
                        int nr = r + dr[d];
                        int nc = c + dc[d];
                        
                        // Bounds check BEFORE accessing grid[nr][nc]
                        if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
                            grid[nr][nc] = '0';  // mark BEFORE pushing ← CRITICAL
                            q.push({nr, nc});
                        }
                    }
                }
            }
        }
    }
    return islands;
}

// THE VISITED MARKING CHOICE:
// Option A: modify grid in-place (grid[r][c] = '0')
//   → Pros: O(1) extra space, clean code
//   → Cons: destroys original grid (restore needed if grid must be preserved)
// Option B: separate visited array: vector<vector<bool>> visited(m, vector<bool>(n, false))
//   → Pros: grid preserved
//   → Cons: O(m*n) extra space
// In interviews: ask "can I modify the grid?" — usually yes. Use Option A.
//
// TRACE for grid=[["1","1","0"],["1","0","0"],["0","0","1"]]:
// i=0,j=0: '1' → islands=1. BFS from (0,0): mark (0,0),(0,1),(1,0) as '0'. Queue processes all.
// i=0,j=1: now '0' (marked) → skip
// ... continue scanning → i=2,j=2: '1' → islands=2. BFS from (2,2): mark (2,2).
// Return 2 ✓
```

---

### Template 2: DFS on Grid — Recursive and Iterative

```cpp
// Flood Fill (LC 733): DFS variant — change all connected same-colored cells to new color.
// Also: DFS template for any grid traversal (area calculation, path marking, etc.)

// RECURSIVE DFS (clean, preferred for interviews unless deep recursion is a concern)
void dfs(vector<vector<int>>& image, int r, int c, int origColor, int newColor) {
    int m = image.size(), n = image[0].size();
    // Base case: out of bounds OR already changed OR wrong color
    if (r < 0 || r >= m || c < 0 || c >= n) return;
    if (image[r][c] != origColor) return;  // not the color to fill OR already changed
    
    image[r][c] = newColor;  // mark as visited / flood-filled
    
    dfs(image, r+1, c, origColor, newColor);
    dfs(image, r-1, c, origColor, newColor);
    dfs(image, r, c+1, origColor, newColor);
    dfs(image, r, c-1, origColor, newColor);
}

vector<vector<int>> floodFill(vector<vector<int>>& image, int sr, int sc, int color) {
    int origColor = image[sr][sc];
    if (origColor != color) dfs(image, sr, sc, origColor, color);
    // If origColor == color: nothing to do (already the right color, no infinite loop)
    return image;
}

// ITERATIVE DFS (use stack instead of recursion — avoids stack overflow for large grids)
void dfs_iterative(vector<vector<int>>& grid, int startR, int startC) {
    int m = grid.size(), n = grid[0].size();
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};
    
    stack<pair<int,int>> stk;
    stk.push({startR, startC});
    grid[startR][startC] = 0;  // mark visited
    
    while (!stk.empty()) {
        auto [r, c] = stk.top(); stk.pop();
        
        for (int d = 0; d < 4; d++) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == 1) {
                grid[nr][nc] = 0;  // mark BEFORE pushing (same rule as BFS)
                stk.push({nr, nc});
            }
        }
    }
}

// WHEN to use DFS vs BFS on grids:
// DFS: connected component detection, area calculation, path existence, marking reachable cells
// BFS: shortest path, minimum distance, "how many steps to reach"
// Both: count components, flood fill, mark reachable
// DFS is simpler to write recursively; BFS handles depth/level tracking naturally.
```

---

### Template 3: BFS on Adjacency List Graph — Shortest Path (Unweighted)

```cpp
// BFS on a general graph (adjacency list representation).
// Returns shortest distance from source to all nodes.
// Distance is -1 if unreachable.

vector<int> bfs_shortest(int source, int n, vector<vector<int>>& adj) {
    vector<int> dist(n, -1);  // -1 = unvisited/unreachable
    queue<int> q;
    
    dist[source] = 0;  // distance to source is 0
    q.push(source);
    
    while (!q.empty()) {
        int node = q.front(); q.pop();
        
        for (int neighbor : adj[node]) {
            if (dist[neighbor] == -1) {  // unvisited
                dist[neighbor] = dist[node] + 1;
                q.push(neighbor);
            }
        }
    }
    
    return dist;
}

// NOTE: We mark dist[neighbor] BEFORE pushing (dist != -1 means visited).
// This is the "mark before push" rule applied to graphs.
// Using a separate `visited` array is equivalent but dist array doubles as visited tracking.

// HOW TO ADAPT for "find shortest path to a specific target" (early exit):
// Add: if (neighbor == target) return dist[neighbor];
// inside the loop, right after setting dist[neighbor].

// HOW TO ADAPT for "reconstruct the shortest path":
// Add a `parent` array: parent[neighbor] = node;
// After BFS, backtrack from target to source using parent pointers.

// TRACE for adj = [[1,2],[0,3],[0,3],[1,2,4],[3]], source = 0:
// dist=[0,-1,-1,-1,-1]. Queue=[0].
// Pop 0: neighbors 1,2. dist[1]=1, dist[2]=1. Queue=[1,2].
// Pop 1: neighbors 0(visited),3. dist[3]=2. Queue=[2,3].
// Pop 2: neighbors 0(visited),3(visited). Queue=[3].
// Pop 3: neighbors 1(visited),2(visited),4. dist[4]=3. Queue=[4].
// Pop 4: neighbors 3(visited). Queue=[].
// dist = [0, 1, 1, 2, 3] ✓
```

---

### Template 4: DFS on Adjacency List Graph — Connected Components + All Paths

```cpp
// DFS for counting connected components and tracking which component each node belongs to.

int countComponents(int n, vector<vector<int>>& edges) {
    vector<vector<int>> adj(n);
    for (auto& e : edges) {
        adj[e[0]].push_back(e[1]);
        adj[e[1]].push_back(e[0]);
    }
    
    vector<bool> visited(n, false);
    int components = 0;
    
    function<void(int)> dfs = [&](int node) {
        visited[node] = true;
        for (int neighbor : adj[node]) {
            if (!visited[neighbor]) dfs(neighbor);
        }
    };
    
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            components++;
            dfs(i);
        }
    }
    return components;
}

// DFS for finding ALL paths from source to target (backtracking):
vector<vector<int>> allPaths(int source, int target, vector<vector<int>>& adj) {
    vector<vector<int>> result;
    vector<int> path = {source};
    vector<bool> inPath(adj.size(), false);
    inPath[source] = true;
    
    function<void(int)> dfs = [&](int node) {
        if (node == target) {
            result.push_back(path);
            return;
        }
        for (int neighbor : adj[node]) {
            if (!inPath[neighbor]) {
                inPath[neighbor] = true;
                path.push_back(neighbor);
                dfs(neighbor);
                path.pop_back();        // backtrack
                inPath[neighbor] = false;
            }
        }
    };
    
    dfs(source);
    return result;
}

// KEY: for all-paths DFS, we undo the visited marking after returning (backtrack).
// This allows the same node to appear in different paths.
// For simple connectivity DFS, we NEVER un-mark (visited stays true permanently).
```

---

### Template 5: Cycle Detection — Undirected Graph (DFS with Parent Tracking)

```cpp
// In an undirected graph, a cycle exists if DFS finds a visited node
// that is NOT the direct parent of the current node.
// Why "not the parent"? In undirected graphs, every edge appears in both directions.
// When we go from A→B, the edge B→A is also present. Without parent tracking,
// B→A would look like a "back edge" — but it's just the reverse of the edge we came from.

bool hasCycleUndirected(int n, vector<vector<int>>& edges) {
    vector<vector<int>> adj(n);
    for (auto& e : edges) {
        adj[e[0]].push_back(e[1]);
        adj[e[1]].push_back(e[0]);
    }
    
    vector<bool> visited(n, false);
    
    function<bool(int, int)> dfs = [&](int node, int parent) -> bool {
        visited[node] = true;
        for (int neighbor : adj[node]) {
            if (!visited[neighbor]) {
                if (dfs(neighbor, node)) return true;
            } else if (neighbor != parent) {
                // Visited AND not our parent → back edge → CYCLE
                return true;
            }
        }
        return false;
    };
    
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            if (dfs(i, -1)) return true;  // -1 = no parent for the first call
        }
    }
    return false;
}

// IMPORTANT EDGE CASE: multiple edges between same two nodes (multigraph).
// Parent tracking by node ID fails for multigraphs!
// Solution: track the EDGE index, not the parent node.
// Pass the parent EDGE INDEX; skip only the specific edge we came from.
// For standard interview problems (simple graphs), parent node tracking is fine.
```

---

### Template 6: Cycle Detection — Directed Graph (3-Color DFS)

```cpp
// In a directed graph, the "parent" trick doesn't work.
// A cycle requires a path from a node BACK TO ITSELF following edge directions.
// Detection: if DFS encounters a node in the CURRENT path (GRAY), it's a cycle.
//
// Colors:
// WHITE (0) = unvisited
// GRAY  (1) = currently in the DFS recursion stack (being processed)
// BLACK (2) = fully processed (all descendants explored, no cycle through here)

bool hasCycleDirected(int n, vector<vector<int>>& adj) {
    vector<int> color(n, 0);  // 0=WHITE, 1=GRAY, 2=BLACK
    
    function<bool(int)> dfs = [&](int node) -> bool {
        color[node] = 1;  // GRAY: mark as "in current path"
        
        for (int neighbor : adj[node]) {
            if (color[neighbor] == 1) return true;   // GRAY → back edge → CYCLE
            if (color[neighbor] == 0) {               // WHITE → unvisited
                if (dfs(neighbor)) return true;
            }
            // BLACK → already fully processed, skip (cross/forward edge, not a cycle)
        }
        
        color[node] = 2;  // BLACK: fully processed, safe
        return false;
    };
    
    for (int i = 0; i < n; i++) {
        if (color[i] == 0) {  // WHITE: unvisited
            if (dfs(i)) return true;
        }
    }
    return false;
}

// LC 207 (Course Schedule): given prerequisites, can you finish all courses?
// = Is the directed graph (prerequisites as edges) acyclic?
// Return: !hasCycleDirected(numCourses, adj)

// WHY 3 colors and not 2?
// With 2 colors (visited/unvisited), you'd mark GRAY AND BLACK as "visited."
// Then encountering a BLACK node (already processed, no cycle) would ALSO look like
// "back to visited node" → false positive cycle detection.
// GRAY specifically means "in the current stack" — only GRAY nodes indicate a real back edge.

// ALTERNATIVE: Kahn's BFS (topological sort) — covered in Pattern 30.
// Both detect cycles in directed graphs. Use 3-color DFS in Pattern 12 problems;
// use Kahn's when you also need the topological order.
```

---

### Template 7: Multi-Source BFS — Simultaneous Spread from All Starting Points

```cpp
// LC 994: Rotting Oranges
// All rotten oranges (2s) spread to adjacent fresh oranges (1s) simultaneously each minute.
// Find minimum minutes for all fresh oranges to rot.
//
// KEY INSIGHT: Push ALL starting sources into the queue BEFORE running BFS.
// Then BFS processes layer by layer: all sources' neighbors at distance 1,
// then all of those at distance 2, etc.
// This is NOT "run BFS from source 1, then source 2, then ..." — that would be WRONG.
// It IS "run ONE BFS starting from ALL sources at the same time."

int orangesRotting(vector<vector<int>>& grid) {
    int m = grid.size(), n = grid[0].size();
    queue<pair<int,int>> q;
    int fresh = 0;
    
    // Initialize: add ALL rotten oranges and count fresh ones
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 2) q.push({i, j});        // all sources at once
            else if (grid[i][j] == 1) fresh++;
        }
    }
    
    if (fresh == 0) return 0;  // no fresh oranges to rot
    
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};
    int minutes = 0;
    
    while (!q.empty()) {
        int size = q.size();  // process this "minute" (one level)
        minutes++;            // we're about to process the next wave
        
        for (int k = 0; k < size; k++) {
            auto [r, c] = q.front(); q.pop();
            
            for (int d = 0; d < 4; d++) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2;  // rot this orange (mark visited)
                    fresh--;
                    q.push({nr, nc});  // add to NEXT level's processing
                }
            }
        }
    }
    
    // Edge case: if some fresh oranges are trapped (disconnected), fresh > 0
    return fresh == 0 ? minutes - 1 : -1;
    // Why minutes-1? We increment minutes BEFORE processing each level.
    // The last iteration: the queue was non-empty but we found no new oranges.
    // We over-counted by 1. Subtract 1 at the end.
}

// ALTERNATIVE (cleaner increment logic): only increment if we actually added to next level
// Track: add to queue → if anything was added → increment minutes
// The "size" loop approach above increments once per level regardless.
// Check the final fresh count to handle the impossible case.

// MULTI-SOURCE BFS TEMPLATE (general):
// 1. Initialize queue with ALL sources: for each source → push(source), mark(source)
// 2. Run BFS normally: for each level, expand to all unvisited neighbors
// 3. The BFS distance from ANY source to any node = minimum distance from the nearest source
```

---

### Template 8: Bipartite Check — 2-Coloring with BFS

```cpp
// A graph is bipartite if its nodes can be colored with 2 colors such that
// no two adjacent nodes share the same color.
// Equivalently: the graph has no odd-length cycles.
// Common disguise: "can you divide people into two groups with no internal conflict?"

bool isBipartite(vector<vector<int>>& graph) {
    int n = graph.size();
    vector<int> color(n, -1);  // -1 = uncolored
    
    for (int start = 0; start < n; start++) {
        if (color[start] != -1) continue;  // already colored (visited)
        
        // BFS to 2-color this connected component
        queue<int> q;
        q.push(start);
        color[start] = 0;  // assign first color
        
        while (!q.empty()) {
            int node = q.front(); q.pop();
            
            for (int neighbor : graph[node]) {
                if (color[neighbor] == -1) {
                    // Uncolored: assign opposite color
                    color[neighbor] = 1 - color[node];
                    q.push(neighbor);
                } else if (color[neighbor] == color[node]) {
                    // Same color as current node → adjacent nodes conflict → NOT bipartite
                    return false;
                }
                // Different color: consistent ✓ — do nothing
            }
        }
    }
    return true;
}

// WHY we loop over ALL start nodes (not just start = 0):
// The graph may be disconnected. Each connected component must be 2-colorable independently.
// If we only start from node 0 and there's a component that's not reachable from 0,
// we'd miss its bipartite check.

// DFS VERSION (equivalent, sometimes cleaner in recursive style):
bool dfs2color(int node, int c, vector<vector<int>>& graph, vector<int>& color) {
    color[node] = c;
    for (int neighbor : graph[node]) {
        if (color[neighbor] == c) return false;  // conflict
        if (color[neighbor] == -1 && !dfs2color(neighbor, 1-c, graph, color)) return false;
    }
    return true;
}
```

---

### Template 9: BFS on Implicit Graph — Word Ladder Pattern

```cpp
// LC 127: Word Ladder
// Transform beginWord → endWord one character at a time.
// Each intermediate word must be in the wordList.
// Find the minimum number of transformations (length of shortest transformation sequence).
//
// This is BFS on an IMPLICIT graph:
// - Nodes are words
// - Edges connect words that differ by exactly one character
// - We DO NOT pre-build the full adjacency list (too expensive: O(L * 26 * N) upfront)
// - Instead, compute neighbors on-the-fly during BFS

int ladderLength(string beginWord, string endWord, vector<string>& wordList) {
    unordered_set<string> wordSet(wordList.begin(), wordList.end());
    if (!wordSet.count(endWord)) return 0;  // endWord not in dictionary → impossible
    
    queue<string> q;
    q.push(beginWord);
    wordSet.erase(beginWord);  // remove beginWord from set (mark as visited)
    int steps = 1;
    
    while (!q.empty()) {
        int size = q.size();  // process one level at a time (one transformation step)
        
        for (int i = 0; i < size; i++) {
            string word = q.front(); q.pop();
            
            // Try all possible one-character transformations
            for (int j = 0; j < (int)word.size(); j++) {
                char original = word[j];
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == original) continue;
                    word[j] = c;
                    
                    if (word == endWord) return steps + 1;
                    
                    if (wordSet.count(word)) {
                        wordSet.erase(word);  // mark visited by removing from set (BEFORE pushing)
                        q.push(word);
                    }
                }
                word[j] = original;  // restore
            }
        }
        steps++;
    }
    return 0;  // endWord unreachable
}

// WHY erase from wordSet instead of a separate visited set?
// The wordSet serves as both the "valid word dictionary" and the "unvisited" tracker.
// Erasing when adding to queue (mark BEFORE push) prevents adding duplicates.
// Using a separate visited set works too but doubles the memory.
//
// WHY process level by level (size loop)?
// We need the STEP COUNT when we reach endWord.
// Processing level by level tracks how many transformations have been made.
// When we find endWord, we return steps + 1 (current level + 1 for the final step).
//
// TIME COMPLEXITY: O(M² × N) where M = word length, N = dict size.
// For each word (N words in queue max), for each position (M positions),
// for each character (26), we check if the transformed word is in the set (O(M) for hash).
// Total: O(N × M × 26 × M) = O(M² × N).
//
// OPTIMIZATION — Bidirectional BFS:
// Start BFS from both beginWord AND endWord simultaneously.
// At each step, expand the smaller frontier. Meet in the middle.
// Reduces O(M² × N) to O(M² × √N) in practice — roughly halves the BFS depth.
```

---

## THE VARIANTS — DEEP DIVE INTO EACH SUB-PATTERN

### VARIANT 1: Reverse BFS from Borders (Pacific Atlantic Water Flow, LC 417)

```cpp
// Pacific Atlantic: water flows from a cell to adjacent cells if adjacent height <= current height.
// Water can flow to Pacific (top/left border) and Atlantic (bottom/right border).
// Find all cells from which water can reach BOTH oceans.
//
// NAIVE approach (WRONG for interview): for each cell, BFS/DFS toward both oceans.
// → O(m*n) starts × O(m*n) per BFS = O(m²n²). TLE for large grids.
//
// REVERSE DIRECTION insight: instead of "can water at cell X reach the ocean?",
// ask "can water START from the ocean and flow UPHILL to reach cell X?"
// The reverse flow condition: height[nr][nc] >= height[r][c] (water goes UPHILL in reverse).
// Start reverse BFS from ALL Pacific border cells simultaneously.
// Start reverse BFS from ALL Atlantic border cells simultaneously.
// Result: cells reachable in BOTH reverse BFS runs.

vector<vector<int>> pacificAtlantic(vector<vector<int>>& heights) {
    int m = heights.size(), n = heights[0].size();
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};
    
    // Multi-source BFS from Pacific (top row + left column)
    auto bfs = [&](queue<pair<int,int>> q) {
        vector<vector<bool>> reachable(m, vector<bool>(n, false));
        while (!q.empty()) {
            auto [r, c] = q.front(); q.pop();
            reachable[r][c] = true;
            for (int d = 0; d < 4; d++) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n
                    && !reachable[nr][nc]
                    && heights[nr][nc] >= heights[r][c]) {  // reverse: uphill is reachable
                    reachable[nr][nc] = true;
                    q.push({nr, nc});
                }
            }
        }
        return reachable;
    };
    
    queue<pair<int,int>> pacific, atlantic;
    for (int i = 0; i < m; i++) {
        pacific.push({i, 0});
        atlantic.push({i, n-1});
    }
    for (int j = 0; j < n; j++) {
        pacific.push({0, j});
        atlantic.push({m-1, j});
    }
    
    auto pac = bfs(pacific);
    auto atl = bfs(atlantic);
    
    vector<vector<int>> result;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (pac[i][j] && atl[i][j]) result.push_back({i, j});
    
    return result;
}

// THE REVERSE DIRECTION PATTERN applies to:
// - "Border-connected" problems: Surrounded Regions (LC 130) — mark 'O' cells connected to border
// - "Reachable from source" problems where computing from source is expensive
// - Any problem where you want "cells that can drain to X" → reverse: "cells reachable FROM X going uphill"
```

---

### VARIANT 2: DFS + BFS Two-Phase — Shortest Bridge (LC 934)

```cpp
// Two islands in a binary matrix. Find minimum flips (0→1) to connect them.
// Phase 1: DFS to find all cells of the FIRST island. Add border cells (those adjacent to 0s) to queue.
// Phase 2: BFS from all first-island border cells. Return steps when BFS reaches the second island.

int shortestBridge(vector<vector<int>>& grid) {
    int n = grid.size();
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};
    queue<pair<int,int>> q;
    bool found = false;
    
    // Phase 1: DFS to identify first island, add its cells to BFS queue
    function<void(int, int)> dfs = [&](int r, int c) {
        if (r < 0 || r >= n || c < 0 || c >= n || grid[r][c] != 1) return;
        grid[r][c] = 2;   // mark first island's cells as 2
        q.push({r, c});   // add to BFS queue (will be sources for Phase 2)
        for (int d = 0; d < 4; d++) dfs(r + dr[d], c + dc[d]);
    };
    
    // Find the first '1' cell and DFS from it
    for (int i = 0; i < n && !found; i++) {
        for (int j = 0; j < n && !found; j++) {
            if (grid[i][j] == 1) {
                dfs(i, j);
                found = true;
            }
        }
    }
    
    // Phase 2: BFS from all first-island cells, expanding into water (0s) and second island (1s)
    int steps = 0;
    while (!q.empty()) {
        int size = q.size();
        for (int k = 0; k < size; k++) {
            auto [r, c] = q.front(); q.pop();
            for (int d = 0; d < 4; d++) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr < 0 || nr >= n || nc < 0 || nc >= n || grid[nr][nc] == 2) continue;
                if (grid[nr][nc] == 1) return steps;  // reached second island!
                grid[nr][nc] = 2;  // mark water as visited
                q.push({nr, nc});
            }
        }
        steps++;
    }
    return -1;  // shouldn't reach here given valid input
}
```

---

### VARIANT 3: Clone Graph (LC 133) — DFS with a Clone Map

```cpp
// Deep copy of a graph: each node has a val and a list of neighbors.
// Must create new nodes (not reference original nodes).
// Use a hash map: original_node → cloned_node (acts as both visited set and result storage).

class Node {
public:
    int val;
    vector<Node*> neighbors;
    Node(int v) : val(v) {}
};

Node* cloneGraph(Node* node) {
    if (!node) return nullptr;
    
    unordered_map<Node*, Node*> cloned;  // original → clone
    
    function<Node*(Node*)> dfs = [&](Node* orig) -> Node* {
        if (cloned.count(orig)) return cloned[orig];  // already cloned → return existing clone
        
        Node* clone = new Node(orig->val);  // create new clone
        cloned[orig] = clone;               // register BEFORE recursing (handles cycles)
        
        for (Node* neighbor : orig->neighbors) {
            clone->neighbors.push_back(dfs(neighbor));
        }
        return clone;
    };
    
    return dfs(node);
}

// CRITICAL: register cloned[orig] = clone BEFORE recursing into neighbors.
// Why? The graph may have cycles. If node A points to B and B points back to A:
// cloneA → recurse to cloneB → recurse to cloneA.
// Without pre-registration, we'd infinite loop. With pre-registration, the second
// encounter of A returns the already-created clone immediately.
//
// This "register before recursing" pattern also appears in:
// - Copy List with Random Pointer (Pattern 07)
// - Memoization in DP (register the result before returning to handle cycles in recursion)
```

---

### VARIANT 4: Graph Valid Tree (LC 261) — Cycle + Connectivity

```cpp
// A graph is a valid tree if:
// 1. It is connected (all n nodes are reachable from any node)
// 2. It has no cycles
// Equivalent necessary + sufficient condition: EXACTLY n-1 edges AND connected.
// (n nodes, n-1 edges, connected → no cycles by definition of tree)

bool validTree(int n, vector<vector<int>>& edges) {
    // Quick check: a tree with n nodes has exactly n-1 edges
    if ((int)edges.size() != n - 1) return false;
    
    vector<vector<int>> adj(n);
    for (auto& e : edges) {
        adj[e[0]].push_back(e[1]);
        adj[e[1]].push_back(e[0]);
    }
    
    // Check connectivity with BFS from node 0
    vector<bool> visited(n, false);
    queue<int> q;
    q.push(0);
    visited[0] = true;
    int count = 1;
    
    while (!q.empty()) {
        int node = q.front(); q.pop();
        for (int neighbor : adj[node]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                q.push(neighbor);
                count++;
            }
        }
    }
    
    return count == n;  // all n nodes were visited → connected
    // Combined with edges.size() == n-1, this guarantees: valid tree (no cycles)
}

// WHY (n-1 edges + connected) → no cycles:
// A tree is a connected acyclic graph. For n nodes: |E| = n-1 always.
// If |E| < n-1: disconnected. If |E| > n-1: at least one cycle.
// |E| == n-1 + connected → exactly a tree (no cycles).
// So we can avoid the cycle detection DFS entirely!
```

---

### VARIANT 5: Surrounded Regions (LC 130) — Border DFS Then Flip

```cpp
// Any 'O' NOT connected to the border is surrounded. Change surrounded 'O's to 'X'.
// Approach: mark ALL 'O's connected to ANY border cell as safe ('T').
// Then: 'O' → 'X' (surrounded), 'T' → 'O' (restore safe ones).

void solve(vector<vector<char>>& board) {
    int m = board.size(), n = board[0].size();
    int dr[] = {0, 0, 1, -1};
    int dc[] = {1, -1, 0, 0};
    
    // DFS to mark border-connected 'O's as 'T' (temporarily safe)
    function<void(int, int)> dfs = [&](int r, int c) {
        if (r < 0 || r >= m || c < 0 || c >= n || board[r][c] != 'O') return;
        board[r][c] = 'T';  // safe
        for (int d = 0; d < 4; d++) dfs(r + dr[d], c + dc[d]);
    };
    
    // Start DFS from all border 'O' cells
    for (int i = 0; i < m; i++) {
        dfs(i, 0);
        dfs(i, n-1);
    }
    for (int j = 0; j < n; j++) {
        dfs(0, j);
        dfs(m-1, j);
    }
    
    // Final pass: 'O' → 'X' (surrounded), 'T' → 'O' (restore safe)
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (board[i][j] == 'O') board[i][j] = 'X';
            else if (board[i][j] == 'T') board[i][j] = 'O';
}
```

---

## TIME AND SPACE COMPLEXITY — COLD RECITATION

| Algorithm | Time | Space | Why |
|-----------|------|-------|-----|
| BFS/DFS on grid (m×n) | O(m×n) | O(m×n) | Each cell visited once; queue/stack holds O(m×n) in worst case |
| BFS/DFS on graph (V,E) | O(V+E) | O(V+E) | Each vertex and edge visited once; adj list is O(V+E) |
| Multi-source BFS | O(m×n) or O(V+E) | O(m×n) or O(V) | Same as regular BFS; extra sources don't add asymptotic cost |
| Cycle detection (directed, 3-color) | O(V+E) | O(V) | DFS visits each node and edge once; color array is O(V) |
| Cycle detection (undirected, parent) | O(V+E) | O(V) | Same as DFS |
| Bipartite check (BFS/DFS) | O(V+E) | O(V) | Standard BFS/DFS with color array |
| Word Ladder (implicit BFS) | O(M²×N) | O(M×N) | N words × M positions × 26 chars × O(M) hash; set stores N words of length M |
| Shortest path BFS (unweighted) | O(V+E) | O(V) | BFS visits each node/edge once; dist array is O(V) |
| Build adjacency list | O(V+E) | O(V+E) | Process each edge once |

### The O(V+E) Fact to State Out Loud:

"BFS and DFS on a graph are O(V+E) — not O(V²). Each VERTEX is visited once (O(V) total) and each EDGE is traversed once (O(E) total). For sparse graphs (E = O(V)), this is O(V). For dense graphs (E = O(V²)), this is O(V²). The adjacency list representation ensures we only traverse edges that exist — adjacency matrix would always be O(V²) regardless of edge count."

### When Does BFS Give Shortest Path?

BFS gives shortest path ONLY when **all edges have equal weight** (usually weight = 1). If edges have different weights: Dijkstra (non-negative) or Bellman-Ford (negative). If weights are 0 or 1: 0-1 BFS with deque. If the problem says "minimum number of steps/moves/transformations" and each step costs 1: BFS.

---

## COMMON MISTAKES AT THIS PATTERN (At YOUR 1600 Rating Level)

---

### Mistake 1: Marking Visited AFTER Popping Instead of BEFORE Pushing

**What they do:** They pop a node from the queue, then check `if visited, skip`, then mark it visited. They think this is equivalent to marking before pushing.

**What goes wrong:** Multiple copies of the same node can be pushed to the queue before any copy is processed and marked visited. For BFS, this causes: (1) duplicate processing with extra work, (2) incorrect level-by-level counting (same node counted in multiple levels), (3) for multi-source BFS, completely wrong distances. For the "mark when popping" approach with a `continue`, it's functionally correct for simple BFS but O(E) extra enqueue operations (each edge explored twice for each duplicate).

**The fix:** Mark as visited BEFORE pushing. Check "is this node unvisited?" as the condition for pushing, not as a condition after popping.

```cpp
// WRONG: mark after popping (can cause duplicate queue entries)
while (!q.empty()) {
    int node = q.front(); q.pop();
    if (visited[node]) continue;  // duplicates skip here — wasteful
    visited[node] = true;         // ← WRONG: should mark before pushing
    for (int neighbor : adj[node]) {
        q.push(neighbor);  // might push same neighbor multiple times!
    }
}

// CORRECT: mark before pushing (no duplicates in queue)
while (!q.empty()) {
    int node = q.front(); q.pop();
    for (int neighbor : adj[node]) {
        if (!visited[neighbor]) {
            visited[neighbor] = true;  // mark BEFORE pushing ← CRITICAL
            q.push(neighbor);
        }
    }
}
```

---

### Mistake 2: Using Parent Tracking for Directed Graph Cycle Detection

**What they do:** They use the undirected cycle detection template (parent tracking) for directed graphs. The check: "if visited and neighbor != parent → cycle" doesn't translate to directed graphs.

**What goes wrong:** In a directed graph, a node can have multiple incoming edges. The "parent" of a node in DFS is not uniquely defined. More critically: two separate paths reaching the same node (a "diamond" shape A→B, A→C, B→D, C→D) would be flagged as a cycle by the parent check — but it's NOT a cycle in a directed graph (D has two parents, but there's no circular dependency). This produces false positives.

**The fix:** Use 3-color (white/gray/black) DFS for directed graphs. Only a gray-to-gray back edge is a cycle. A gray-to-black cross/forward edge is NOT a cycle.

```cpp
// WRONG: parent tracking on directed graph
bool hasCycle_Wrong(int node, int parent, vector<vector<int>>& adj, vector<bool>& visited) {
    visited[node] = true;
    for (int neighbor : adj[node]) {
        if (!visited[neighbor]) {
            if (hasCycle_Wrong(neighbor, node, adj, visited)) return true;
        } else if (neighbor != parent) {
            return true;  // FALSE POSITIVE in directed graph!
            // In A→B, A→C, B→D, C→D: when we visit D from C, B already visited it.
            // B != parent(C) = A → false cycle reported. WRONG.
        }
    }
    return false;
}

// CORRECT: 3-color for directed graph
bool hasCycle_Correct(int node, vector<vector<int>>& adj, vector<int>& color) {
    color[node] = 1;  // GRAY
    for (int neighbor : adj[node]) {
        if (color[neighbor] == 1) return true;  // GRAY back edge → real cycle
        if (color[neighbor] == 0 && hasCycle_Correct(neighbor, adj, color)) return true;
        // color[neighbor] == 2 (BLACK): safe, skip
    }
    color[node] = 2;  // BLACK
    return false;
}
```

---

### Mistake 3: Multi-Source BFS — Starting Sources One at a Time Instead of Simultaneously

**What they do:** For multi-source BFS problems (Rotting Oranges, Walls and Gates), they run separate BFS calls from each source and take the minimum distance per cell.

**What goes wrong:** Running separate BFS calls ignores the simultaneous spreading. In Rotting Oranges: if rotten orange A is at (0,0) and rotten orange B is at (0,3), a fresh orange at (0,2) is 2 minutes from A and 1 minute from B. A separate BFS from A gives distance 2, and from B gives distance 1. Taking the min gives 1 — technically correct for this particular problem, but the approach is O(K × m × n) where K = number of sources. More importantly, it's conceptually wrong: the problem says ALL rotten oranges spread simultaneously, not sequentially. For problems where the interaction between sources matters (overlapping spread fronts), separate BFS is incorrect.

**The fix:** Add ALL sources to the queue before starting BFS. One single BFS handles all sources simultaneously, naturally taking the minimum distance.

```cpp
// WRONG: separate BFS calls from each source (O(K × m × n))
int wrongAnswer(vector<vector<int>>& grid) {
    int maxTime = 0;
    for (auto& src : rottensources) {
        // run BFS from src alone, get distances
        // update maxTime
    }
    // Returns wrong result for fresh oranges between two rotten sources
}

// CORRECT: ALL sources at once in ONE BFS (O(m × n))
queue<pair<int,int>> q;
for (int i = 0; i < m; i++)
    for (int j = 0; j < n; j++)
        if (grid[i][j] == 2) {
            q.push({i, j});  // add ALL sources before starting BFS
        }
// Now run ONE BFS with this pre-loaded queue
```

---

### Mistake 4: Grid Bounds Check — Wrong Order of Conditions (Accessing Out-of-Bounds)

**What they do:** They write the bounds check as `grid[nr][nc] == '1' && nr >= 0 && nr < m && nc >= 0 && nc < n`. The grid access comes BEFORE the bounds check.

**What goes wrong:** In C++, short-circuit evaluation of `&&` processes conditions left to right. `grid[nr][nc]` is evaluated BEFORE checking if `nr` and `nc` are valid indices. Accessing `grid[-1][nc]` or `grid[m][nc]` is undefined behavior — usually a crash or garbage value read.

**The fix:** ALWAYS put the bounds check BEFORE the grid access in the same condition chain.

```cpp
// WRONG: grid access before bounds check (undefined behavior / crash)
if (grid[nr][nc] == '1' && nr >= 0 && nr < m && nc >= 0 && nc < n) {
    // grid[nr][nc] is accessed before knowing if nr, nc are valid!
}

// CORRECT: bounds check FIRST, then grid access
if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
    // Short circuit: if any bound check fails, grid[nr][nc] is never accessed
}
```

---

### Mistake 5: Not Handling Disconnected Graphs — Starting BFS/DFS from Only One Node

**What they do:** For problems on disconnected graphs (e.g., "check if bipartite" or "count components"), they run BFS/DFS from only node 0, assuming all nodes are reachable from node 0.

**What goes wrong:** If the graph has multiple connected components, nodes in other components are never visited. The bipartite check would miss odd cycles in unreachable components (false positive "it's bipartite"). Component count would return 1 (false). Any algorithm requiring all nodes to be processed produces wrong results.

**The fix:** For disconnected graphs, loop over ALL nodes and start a new BFS/DFS from each unvisited node.

```cpp
// WRONG: only start from node 0
bool isBipartite_Wrong(vector<vector<int>>& graph) {
    int n = graph.size();
    vector<int> color(n, -1);
    // Only runs BFS from node 0 — misses components not connected to 0
    bfsColor(0, 0, graph, color);
    // Returns wrong result for disconnected graphs with odd cycles elsewhere
    return true;
}

// CORRECT: loop over ALL nodes
bool isBipartite_Correct(vector<vector<int>>& graph) {
    int n = graph.size();
    vector<int> color(n, -1);
    for (int start = 0; start < n; start++) {
        if (color[start] == -1) {  // unvisited → new component
            if (!bfsColor(start, 0, graph, color)) return false;
        }
    }
    return true;
}
```

---

### Mistake 6: Word Ladder — Using Vector Instead of Unordered_Set for Word Lookup

**What they do:** They pass the `wordList` as a `vector<string>` and check membership with `find(wordList.begin(), wordList.end(), word) != wordList.end()` — an O(n) linear scan for each word.

**What goes wrong:** For each BFS level, for each word in the queue, for each position (M), for each character (26), we check if the transformed word is in the list. With O(n) lookup: O(N × M × 26 × n) = O(M × N²) per BFS step. For N=1000 words of length M=5, this is 1000 × 1000 × 5 × 26 = 130 million operations → TLE.

**The fix:** Convert the word list to an `unordered_set<string>` upfront. This gives O(M) average-case lookup (for string hashing). Total complexity drops to O(M² × N).

```cpp
// WRONG: vector<string> lookup is O(n) per check
int ladderLength_Wrong(string begin, string end, vector<string>& wordList) {
    // Using wordList directly for membership checks is O(n) each → TLE
    for (char c = 'a'; c <= 'z'; c++) {
        word[j] = c;
        auto it = find(wordList.begin(), wordList.end(), word);  // O(n)!
        if (it != wordList.end()) { /* ... */ }
    }
}

// CORRECT: convert to unordered_set first for O(M) average lookup
int ladderLength_Correct(string begin, string end, vector<string>& wordList) {
    unordered_set<string> wordSet(wordList.begin(), wordList.end());  // O(n × M) upfront
    // Now each wordSet.count(word) is O(M) average (string hashing)
}
```

---

## THE PROBLEM SET — SOLVE IN THIS EXACT ORDER

---

### WARMUP — 2 problems (build BFS/DFS grid reflex)

**1. Number of Islands — LC 200 — [Template 1: BFS on grid, count components]**

Why this problem: The canonical BFS/DFS on grid problem. Every FAANG company uses it as a warm-up or phone screen filter. The template — iterate all cells, BFS/DFS from each unvisited '1', count BFS starts — must be automatic. Either BFS or DFS works. Target: 5 minutes, zero bugs, correct visited marking.

What to prove: The outer loop over all (i,j) cells is necessary — not all cells are reachable from a single starting point. Marking visited by modifying the grid (grid[r][c] = '0') is equivalent to a separate visited array. The BFS correctly marks all connected '1' cells when it terminates.

Target time: 5 minutes.

Edge cases: All water (return 0). All land (return 1). Grid with a single cell. Islands of size 1.

---

**2. Flood Fill — LC 733 — [Template 2: DFS on grid; handle same-color edge case]**

Why this problem: The DFS on grid template, plus one non-obvious edge case: if `origColor == newColor`, calling DFS without a guard would infinite-loop (every cell already has the target color, satisfying the "same as origColor" condition forever). Must check `if (origColor == newColor) return image;` before starting DFS.

What to prove: The DFS correctly marks and recursively fills all 4-directionally connected same-colored cells. The base case handles bounds, wrong color, AND the already-changed color (since changed cells no longer equal origColor).

Target time: 4 minutes.

Edge cases: Starting cell's color equals new color (return immediately — no-op). Single cell. Entire grid is same color.

---

### CORE — 4 problems (the fundamental graph techniques)

**3. Clone Graph — LC 133 — [Variant 3: DFS with clone map; register BEFORE recursing]**

Why this problem: Forces you to write DFS with a hash map serving as both visited set and result storage. The "register clone BEFORE recursing into neighbors" trick is essential for handling cycles — without it, graphs with cycles cause infinite loops. This pattern (register result before recursing) also appears in memoization and topological sort.

What to prove: `cloned[orig] = clone` is set BEFORE `dfs(neighbor)` is called. This way, when a cycle causes us to revisit `orig`, `cloned.count(orig)` returns true and the existing clone is returned immediately.

Target time: 12 minutes.

Edge cases: Null input (return null). Graph with a single node and no neighbors. Graph where every node connects to every other (complete graph).

---

**4. Rotting Oranges — LC 994 — [Template 7: Multi-source BFS; ALL sources before BFS starts]**

Why this problem: The canonical multi-source BFS problem. The key insight: push ALL rotten oranges into the queue before starting BFS, then process level by level. Getting the level counting right (minutes) and the final check for trapped fresh oranges (return -1) are where most candidates make mistakes.

What to prove: Adding ALL rotten oranges before BFS correctly models "simultaneous spreading." The `minutes - 1` correction (or equivalent logic) handles the off-by-one in minute counting. The `fresh == 0` check after BFS handles the case where some fresh oranges are trapped (surrounded by water or walls with no rotten neighbor).

Target time: 15 minutes.

Edge cases: No fresh oranges initially (return 0). Fresh oranges that can never be reached (return -1). All oranges already rotten (return 0). Single rotten orange in a line of fresh oranges.

---

**5. Pacific Atlantic Water Flow — LC 417 — [Variant 1: Reverse multi-source BFS from borders]**

Why this problem: Forces the "reverse direction" insight. Direct approach (BFS from every cell) is O(m²n²) — too slow. Reverse: BFS from Pacific border cells uphill, then from Atlantic border cells uphill. Cells in both reachable sets are the answer. The reversal of the flow condition (height >= instead of height <=) is the entire key.

What to prove: The reverse BFS with `height[nr][nc] >= height[r][c]` correctly identifies all cells that can drain to a particular ocean. Running two separate multi-source BFS operations (one per ocean) gives the correct reachability sets.

Target time: 20 minutes.

Edge cases: 1×1 grid (the single cell borders both oceans → it's in the answer). Monotonically increasing grid (all cells drain to both oceans). Flat grid where all heights are equal (any cell adjacent to any ocean border drains there).

---

**6. Course Schedule — LC 207 — [Template 6: Directed cycle detection with 3-color DFS]**

Why this problem: THE directed cycle detection problem. Forces you to understand why undirected cycle detection (parent tracking) fails on directed graphs. The 3-color (white/gray/black) approach is the definitive answer. Must also know: equivalently solvable via Kahn's topological sort (Pattern 30).

What to prove: The 3-color marking correctly distinguishes back edges (GRAY → GRAY, cycle) from cross/forward edges (GRAY → BLACK, no cycle). The outer loop over all nodes handles disconnected graphs (some courses may have no prerequisites and are isolated nodes).

Target time: 15 minutes.

Edge cases: No prerequisites (trivially schedulable). Single node with self-loop (cycle → return false). All courses form a single chain (no cycle → return true).

---

### STRETCH — 3 problems

**7. Word Ladder — LC 127 — [Template 9: BFS on implicit graph; O(M²×N) neighbors computed on-the-fly]**

Why this problem: The hardest graph pattern in this set. Tests whether you can model an IMPLICIT graph (no pre-built adjacency list) and compute neighbors on-the-fly. The `unordered_set` requirement is not optional — vector gives TLE. The level-by-level BFS (size loop) tracks the step count. This problem is a Google L4/L5 filtering problem — if you can't solve it cleanly, you won't pass.

What to prove: Each word's neighbors are found by trying all 26 replacements at each position. Removing from the word set when pushing (mark before push) prevents revisiting. The level-by-level processing correctly tracks transformation steps.

Target time: 25 minutes.

Edge cases: `endWord` not in `wordList` (return 0 immediately). `beginWord == endWord` (return 1 — 0 transformations, but the sequence has length 1). No path exists (return 0).

---

**8. Surrounded Regions — LC 130 — [Variant 5: Border DFS, three-pass transformation]**

Why this problem: Tests the "start from border" insight. Any 'O' that is NOT connected to the border is surrounded and should be flipped. Starting DFS from every 'O' cell and checking if it reaches the border is O(m²n²). Starting DFS from ALL border 'O' cells once is O(m×n). The three-pass approach (mark safe, flip others, restore safe) is clean and elegant.

What to prove: Only 'O' cells connected to the border are truly safe. The DFS from border marks exactly these cells. The two-pass update correctly flips remaining 'O' cells and restores safe ones.

Target time: 18 minutes.

Edge cases: No 'O' cells. All 'O' cells connected to the border (nothing is surrounded). Grid with all 'X' cells. A 1×n or m×1 grid (all cells border — nothing surrounded).

---

**9. Shortest Path in Binary Matrix — LC 1091 — [BFS shortest path on grid with 8-directional movement]**

Why this problem: A direct "shortest path on grid" BFS application, but with 8-directional movement (diagonals allowed). Forces you to extend the standard 4-directional template to 8 directions. Also tests the edge case: if the start or end cell is 1 (blocked), return -1 immediately.

What to prove: BFS gives the shortest path (minimum number of cells in the path) because each step has equal cost (1 cell). The 8-direction movement array has 8 entries. The path length = number of CELLS visited (including start and end) = BFS distance + 1 (since dist from start = 0 for the start cell itself, and we want path length = cells = dist + 1).

Target time: 15 minutes.

Edge cases: Start or end cell is 1 (return -1 immediately). 1×1 grid where the single cell is 0 (return 1). No path exists (return -1).

---

### CONTEST — 3 problems

**10. Walls and Gates — LC 286 — [Multi-source BFS from ALL gates simultaneously]**

Why this is contest level: Tests whether you see that pushing ALL gate positions into the queue before BFS is the correct approach — not running BFS from each gate separately. The result: each empty room gets the distance to the NEAREST gate automatically, without any extra logic.

What to prove: Multi-source BFS automatically fills each INF cell with the distance to its nearest gate. Starting all gates simultaneously means BFS fronts from different gates don't "compete" — the first time a cell is reached by any gate's BFS front is the minimum distance (BFS shortest path property).

Target time: 15 minutes.

Edge cases: No gates (rooms remain INF). No empty rooms (nothing to fill). Room completely surrounded by walls (stays INF).

---

**11. Graph Valid Tree — LC 261 — [Variant 4: n-1 edges + connected]**

Why this is contest level: Requires combining two properties: no cycle AND connected. The elegant shortcut — check `edges.size() == n-1` first, then check connectivity — avoids explicit cycle detection. Knowing WHY `n-1 edges + connected → no cycles` (tree property) is the insight.

What to prove: A tree on n nodes has exactly n-1 edges. If `|E| != n-1`, it's definitely not a tree. If `|E| == n-1` AND connected, then by the handshake lemma and tree properties, it has no cycles. The connectivity BFS only needs to visit all n nodes — if `count != n`, the graph is disconnected.

Target time: 15 minutes.

Edge cases: n=1, no edges (valid tree — single node). n=2, one edge (valid tree). n=2, no edges (invalid — disconnected). Nodes with self-loops (self-loop → edge count check fails: edges.size() >= n).

---

**12. Shortest Bridge — LC 934 — [Variant 2: DFS to find first island + BFS to expand toward second]**

Why this is contest level: Combines DFS (to identify and mark the first island) with BFS (to find minimum distance from first island to second). The two-phase approach requires knowing WHICH technique to use for each phase — DFS for "find and mark all connected cells" and BFS for "minimum distance expansion." The simultaneous BFS from all first-island cells is multi-source BFS.

What to prove: DFS from the first '1' found marks ALL cells of the first island (changing them to 2) and adds them to the BFS queue. BFS then expands from all first-island cells as sources, counting steps until a '1' cell (second island) is reached.

Target time: 25 minutes.

Edge cases: Islands touching each other (answer = 0, but this won't happen given the constraint of 2 distinct islands). Minimum-size 2×2 grid. Islands that are a single cell each.

---

## PATTERN CONNECTIONS

**Connects to Trees DFS (Pattern 08) and Trees BFS (Pattern 09):**
Trees are special graphs — connected, acyclic, with n nodes and n-1 edges. Every tree algorithm is a graph algorithm with the acyclic/connected constraint. Tree DFS and tree BFS use the same templates as graph DFS/BFS, but without the `visited` array (because trees have no cycles — you'll never revisit a node by going forward). If you've mastered Patterns 08-09, graph DFS/BFS is a small mental step.

**Connects to Topological Sort (Pattern 30):**
Course Schedule (LC 207) — directed cycle detection — is the prerequisite for topological sort. Pattern 12 teaches: how to detect if a topological sort exists (no cycle). Pattern 30 teaches: how to actually compute the topological order (Kahn's BFS). Mastering Pattern 12's directed cycle detection makes Pattern 30 trivial.

**Connects to Shortest Path (Pattern 14):**
BFS gives shortest path only for unweighted graphs. Pattern 14 generalizes to weighted graphs (Dijkstra, Bellman-Ford). Everything in Pattern 12's BFS shortest path template transfers directly to Pattern 14's Dijkstra template — just add weights to the priority queue. Pattern 12 is the PREREQUISITE for Pattern 14.

**Connects to Union Find (Pattern 13):**
Union Find is an alternative for "connected components" and "cycle detection in undirected graphs." Pattern 12 (BFS/DFS) and Pattern 13 (DSU) solve the same connectivity problems with different strengths: BFS/DFS is better for "find the shortest path through a component," DSU is better for "dynamically connect nodes one by one and query connectivity at any point."

**Connects to Backtracking (Pattern 15):**
DFS on implicit graphs (all paths, all combinations) is backtracking. Word Ladder variant: "find ALL transformations from begin to end" requires backtracking DFS, not BFS. Pattern 15 formalizes this: what are my choices, what are my constraints, when do I backtrack. Pattern 12's DFS is the substrate; Pattern 15's backtracking adds the decision-tree structure.

---

## INTERVIEW SIMULATION QUESTIONS

---

### Q1: "Given an m×n binary grid where 1 represents land and 0 represents water, return the number of islands."

**Company type:** Every company — literally every single one. Used as warm-up at Amazon, Google, Microsoft, and all Indian unicorns. Also used as the ENTIRE screen at some startups.
**Expected time:** 5-8 minutes
**What they are testing:** BFS/DFS grid template, visited marking, component counting loop

**Red flags that get candidates rejected:**
- Writing `grid[nr][nc] == '1' && nr >= 0 && nr < m` (bounds after access) → likely crash on edge cases
- Marking visited AFTER popping → duplicates in queue → potentially TLE or wrong count
- Missing the outer loop over all (i,j) cells → only counts the component containing (0,0)
- Not handling the `grid[0].size()` for n (using grid.size() for both dimensions)
- Spending more than 8 minutes — this is supposed to be instant at any competitive level

**What they want you to say:**
"Grid problems are implicit graphs where each cell is a node and edges go to 4-directional neighbors. I'll iterate all cells and start a BFS from each unvisited '1' cell, counting how many BFS starts I make. Each BFS start is a new island. I'll mark visited cells by changing '1' to '0' to save the O(m×n) visited array."

**Follow-up:** "Can you do it with DFS instead?" → Same structure, replace `queue` with recursion or `stack`. "What's the space complexity?" → O(m×n) in the worst case (one island that's the entire grid — BFS queue holds all cells). "What if you can't modify the grid?" → Use a separate `visited` boolean matrix with the same O(m×n) space.

---

### Q2: "There are n courses labeled 0 to n-1. You are given prerequisites[i] = [a, b] meaning course b must be taken before course a. Can you finish all courses?"

**Company type:** Google (top 10 most asked), Amazon, Meta
**Expected time:** 15 minutes
**What they are testing:** Directed graph cycle detection, adjacency list construction, 3-color DFS OR Kahn's BFS — must know both approaches

**Red flags that get candidates rejected:**
- Using undirected cycle detection (parent tracking) — produces false positives for directed graphs with "diamond" shapes
- Using only 2 colors (visited/unvisited) instead of 3 — BLACK nodes (fully processed) trigger false cycle detection
- Not building the adjacency list first — trying to use the prerequisites array directly in DFS
- Not handling disconnected graphs (courses with no prerequisites are isolated → must loop all nodes)
- Not knowing the Kahn's alternative when asked "is there another way?"

**What they want you to say:**
"This is directed cycle detection. Build adjacency list from prerequisites (b → a means b must come before a, so a depends on b, directed edge a→b in prerequisite graph). Use 3-color DFS: white (unvisited), gray (currently in DFS path), black (fully processed). If we encounter a gray node during DFS → back edge → cycle → can't finish all courses. Run DFS from all unvisited nodes to handle disconnected components."

**Follow-up:** "What if you need to return the actual course order?" → Topological sort (Pattern 30). "What is the time complexity?" → O(V+E) where V=n courses, E=prerequisites.size(). "Can you do it without recursion?" → Kahn's algorithm: BFS with in-degree counting. Process nodes with in-degree 0 first; if we process all n nodes → no cycle → return true.

---

### Q3: "Given two words, beginWord and endWord, and a word dictionary, find the length of the shortest transformation sequence from beginWord to endWord, where each transformation changes exactly one letter and every intermediate word must be in the dictionary."

**Company type:** Google (appears in 20%+ of L4/L5 rounds), Meta, Bloomberg (senior rounds)
**Expected time:** 25 minutes
**What they are testing:** Recognition that this is BFS on an IMPLICIT graph, correct neighbor generation, `unordered_set` for O(1) lookup, level-by-level BFS for step counting

**Red flags that get candidates rejected:**
- Attempting DFS (produces incorrect answer — DFS finds A path, not the SHORTEST path)
- Pre-building the full adjacency list (N² pairs, each needing O(M) comparison → O(N²M) upfront, wasteful)
- Using `vector<string>` for word lookup instead of `unordered_set` → O(N) per lookup → TLE
- Forgetting to check `if (!wordSet.count(endWord)) return 0` early
- Off-by-one in step counting: returning `steps` when you should return `steps + 1` or vice versa
- Not erasing words from the set when pushing (marking visited AFTER popping → duplicate paths)

**What they want you to say:**
"This is BFS on an implicit graph — each word is a node, and edges connect words differing by exactly one character. The implicit graph has N nodes and up to N×M×26 edges. BFS gives the shortest path. I won't pre-build the adjacency list — for each word in the BFS queue, I generate its neighbors on-the-fly by trying all 26 substitutions at each position and checking against the word set. I use an `unordered_set` for O(M) average lookup. I process BFS level by level (the size loop) to track transformation steps. Words are removed from the set when pushed to prevent revisiting."

**Follow-up:** "Can you do better than O(M²×N)?" → Bidirectional BFS: start from both beginWord and endWord simultaneously, expand the smaller frontier at each step, stop when the frontiers meet. Roughly halves the BFS depth in practice → faster in practice but same worst-case. "What if you need all shortest paths?" → This requires a modified BFS where you don't remove from the word set within a level (allow multiple paths to reach the same node at the same distance), then DFS/backtracking to reconstruct paths.

---

## MENTAL MODEL CHECKPOINT

**Answer all 7 from memory before attempting sign-off.**

**Q1:** Why does BFS guarantee the shortest path on unweighted graphs, but DFS does NOT?
> **A:** BFS explores nodes in order of increasing distance from the source — first all nodes at distance 1, then all at distance 2, then 3, etc. The first time BFS reaches any node, it has traveled the minimum number of edges to get there (because all shorter paths would have been explored in previous layers). Formally: if BFS reaches node v at distance d, there is no shorter path, because all paths of length < d were explored in previous layers. DFS does NOT explore in order of distance — it dives deep along one path, potentially reaching a node via a long winding path before discovering a shorter path. DFS can find A path to the destination, but it's not guaranteed to be the shortest.

**Q2:** State the 3-color cycle detection algorithm for directed graphs. What does each color mean? When is a cycle detected?
> **A:** WHITE (0) = node not yet visited. GRAY (1) = node currently being processed (it's in the active DFS recursion stack — we've entered it but not finished it). BLACK (2) = node fully processed (all descendants have been explored, no cycle through this node). DFS marks a node GRAY on entry and BLACK on exit. A cycle is detected when DFS encounters a neighbor that is currently GRAY — this means there's a path from the current node BACK TO an ancestor that's still on the stack, forming a cycle. Encountering a BLACK neighbor is NOT a cycle — it's a cross or forward edge (a node from a different branch that was already fully processed).

**Q3:** What is the "mark before push" rule for BFS? Give the reason it matters for multi-source BFS specifically.
> **A:** The rule: when adding a node to the BFS queue, mark it as visited IMMEDIATELY (before the push), not when you pop it. This prevents the same node from being added to the queue multiple times. For multi-source BFS, this matters critically: if you have sources A and B that are both adjacent to cell C, and you mark C as visited only when popping, C gets added to the queue TWICE (once by A, once by B). When C is popped the first time, its distance is set correctly. When it's popped the second time, it tries to process C again — potentially overwriting correct distances or creating incorrect expansions. For BFS correctness: "mark when pushing" ensures each node is enqueued exactly once.

**Q4:** How do you detect a cycle in an UNDIRECTED graph using DFS? What is the parent check, and why doesn't it work for directed graphs?
> **A:** In undirected graph DFS, track the parent of each node (the node we came from). While exploring node `u`'s neighbors: if a neighbor `v` is already visited AND `v != parent(u)`, we found a back edge → cycle. The "not the parent" check is necessary because in an undirected graph, the edge `u → v` also appears as `v → u`. Without the check, every edge would be a false positive (we'd see `v` as "already visited" from `u` and cry cycle). For directed graphs, the parent check fails because a directed graph can have "diamond" shapes (A→B, A→C, B→D, C→D) where D has two incoming edges but no cycle. The parent check would trigger falsely when C processes D (D was already visited by B, and B ≠ parent of C). The 3-color approach correctly handles this by distinguishing "in current stack" (GRAY, cycle) from "already processed" (BLACK, no cycle).

**Q5:** Describe multi-source BFS. Give an example problem and explain WHY you push all sources before starting BFS.
> **A:** Multi-source BFS starts BFS from multiple starting nodes simultaneously in a single BFS run. ALL sources are pushed into the queue and marked visited BEFORE the first BFS iteration. Example: Rotting Oranges — all initially rotten oranges are sources. By pushing all sources before starting, the BFS front expands simultaneously from all rotten oranges. Each cell gets the distance to its NEAREST rotten source automatically — no min() computation needed. Why push all before starting: if you started BFS from source A first, then source B, the cell between them would get an incorrect distance (distance from A, potentially larger than its true distance from B). Multi-source BFS is equivalent to adding a virtual "super-source" node connected to all real sources with distance 0, then running regular BFS from the super-source.

**Q6:** What is an implicit graph? How is Word Ladder a BFS on an implicit graph? What's the time complexity, and why?
> **A:** An implicit graph is a graph where edges are not explicitly stored — instead, the neighbors of any node are computed on-the-fly by applying a rule or transformation. Word Ladder: each word is a node. Two words are connected (edge exists) if they differ by exactly one character. The graph is implicit because we DON'T pre-build the adjacency list (too expensive at O(N²M)). Instead, during BFS, for each word at the front of the queue, we compute its neighbors by trying all 26 character replacements at each of M positions and checking if the result is in the word set. Time complexity O(M²×N): N words can be in the queue; for each word, M positions × 26 chars × O(M) hash check = O(M² × 26) ≈ O(M²); total O(N × M²). Space: O(N×M) for the word set and queue.

**Q7:** For Pacific Atlantic Water Flow, why is the "reverse direction" approach better than brute force? What changes in the BFS/DFS condition when reversing?
> **A:** Brute force: for each of O(m×n) cells, run BFS/DFS to check if it can reach Pacific AND Atlantic. Each BFS/DFS is O(m×n). Total: O(m²n²). Too slow. Reverse direction: water flows downhill from any cell to adjacent cells with height ≤ current height. REVERSE: start at the ocean borders and flow UPHILL. A cell can drain to Pacific if it's reachable FROM Pacific flowing uphill. Multi-source BFS from Pacific border cells with the condition `height[nr][nc] >= height[r][c]` (instead of ≤) correctly identifies all Pacific-drainable cells in ONE BFS sweep, O(m×n). Same for Atlantic. Two O(m×n) sweeps total — O(m×n) overall. The key change: the flow condition reverses from `neighbor_height ≤ current` (forward, water flows down) to `neighbor_height ≥ current` (reverse, we flow up from ocean). This reversal is the entire insight.

---

## C++ GRAPHS CHEAT SHEET

```cpp
// ============================================================
// GRAPH CONSTRUCTION
// ============================================================
int n;  // number of nodes (0-indexed)
vector<vector<int>> adj(n);

// Undirected from edge list:
for (auto& e : edges) { adj[e[0]].push_back(e[1]); adj[e[1]].push_back(e[0]); }

// Directed from edge list:
for (auto& e : edges) { adj[e[0]].push_back(e[1]); }

// ============================================================
// GRID MOVEMENT ARRAYS
// ============================================================
int dr4[] = {0, 0, 1, -1};        // 4-directional
int dc4[] = {1, -1, 0, 0};
int dr8[] = {0,0,1,-1,1,1,-1,-1}; // 8-directional
int dc8[] = {1,-1,0,0,1,-1,1,-1};

// Bounds check template (ALWAYS bounds before grid access):
if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1')

// ============================================================
// BFS TEMPLATE (GRAPH OR GRID)
// ============================================================
// BFS graph:
vector<bool> visited(n, false);
queue<int> q; q.push(start); visited[start] = true;  // mark BEFORE push
while (!q.empty()) {
    int node = q.front(); q.pop();
    for (int neighbor : adj[node])
        if (!visited[neighbor]) { visited[neighbor] = true; q.push(neighbor); }
}

// BFS grid (modify in place):
queue<pair<int,int>> q; q.push({r,c}); grid[r][c] = '0';
while (!q.empty()) {
    auto [r,c] = q.front(); q.pop();
    for (int d = 0; d < 4; d++) {
        int nr = r+dr4[d], nc = c+dc4[d];
        if (nr>=0 && nr<m && nc>=0 && nc<n && grid[nr][nc]=='1')
            { grid[nr][nc]='0'; q.push({nr,nc}); }
    }
}

// ============================================================
// DFS TEMPLATE (RECURSIVE)
// ============================================================
function<void(int)> dfs = [&](int node) {
    visited[node] = true;
    for (int neighbor : adj[node])
        if (!visited[neighbor]) dfs(neighbor);
};

// DFS grid (recursive):
void dfs(vector<vector<char>>& grid, int r, int c, int m, int n) {
    if (r<0||r>=m||c<0||c>=n||grid[r][c]!='1') return;
    grid[r][c]='0';
    dfs(grid,r+1,c,m,n); dfs(grid,r-1,c,m,n);
    dfs(grid,r,c+1,m,n); dfs(grid,r,c-1,m,n);
}

// ============================================================
// DIRECTED CYCLE DETECTION (3-COLOR)
// ============================================================
vector<int> color(n, 0);  // 0=white, 1=gray, 2=black
function<bool(int)> dfs3 = [&](int u) -> bool {
    color[u] = 1;
    for (int v : adj[u]) {
        if (color[v]==1) return true;   // cycle!
        if (color[v]==0 && dfs3(v)) return true;
    }
    color[u] = 2; return false;
};
for (int i=0; i<n; i++) if (color[i]==0) if (dfs3(i)) return true; // cycle

// ============================================================
// MULTI-SOURCE BFS
// ============================================================
queue<pair<int,int>> q;
for (all sources) { q.push(source); mark(source); }  // ALL sources BEFORE BFS
while (!q.empty()) { /* standard BFS */ }

// ============================================================
// BIPARTITE CHECK
// ============================================================
vector<int> color(n, -1);
for (int start=0; start<n; start++) {
    if (color[start] != -1) continue;
    queue<int> q; q.push(start); color[start]=0;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) {
            if (color[v]==-1) { color[v]=1-color[u]; q.push(v); }
            else if (color[v]==color[u]) return false;
        }
    }
}
return true;

// ============================================================
// BFS SHORTEST PATH (unweighted)
// ============================================================
vector<int> dist(n, -1); dist[src]=0;
queue<int> q; q.push(src);
while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : adj[u]) if (dist[v]==-1) { dist[v]=dist[u]+1; q.push(v); }
}

// ============================================================
// IMPLICIT GRAPH BFS (Word Ladder style)
// ============================================================
unordered_set<string> ws(wordList.begin(), wordList.end());
queue<string> q; q.push(begin); ws.erase(begin);
int steps = 1;
while (!q.empty()) {
    int sz = q.size(); steps++;
    for (int i=0; i<sz; i++) {
        string w = q.front(); q.pop();
        for (int j=0; j<w.size(); j++) {
            char orig=w[j];
            for (char c='a';c<='z';c++) {
                w[j]=c;
                if (w==end) return steps;
                if (ws.count(w)) { ws.erase(w); q.push(w); }
            }
            w[j]=orig;
        }
    }
}
```

---

## WHAT TO DO AFTER COMPLETING THIS PATTERN

1. Write the 4-directional grid BFS template from memory three times — the `dr[]`/`dc[]` arrays, bounds check order (`bounds first, then grid access`), mark before push. Do it under 3 minutes each time.

2. Write the directed cycle detection (3-color DFS) from memory. Then trace through a diamond graph (A→B, A→C, B→D, C→D) and verify NO cycle is reported. Then trace a cycle graph (A→B→C→A) and verify a cycle IS reported.

3. Implement multi-source BFS for Rotting Oranges WITHOUT looking at your notes. Specifically: push ALL rotten oranges first, then run BFS with the size loop. Get the minute counting right and the `-1` impossible case right.

4. For Word Ladder: implement it using `vector<string>` first (on purpose) and measure that it TLEs. Then switch to `unordered_set<string>`. This makes the O(M²×N) vs O(M²×N²) difference tangible.

5. In REVISION-TRACKER.md, add these for weekly revisit:
   - LC 207 (Course Schedule) — 3-color directed cycle detection
   - LC 417 (Pacific Atlantic) — reverse BFS from borders
   - LC 127 (Word Ladder) — implicit graph BFS, `unordered_set` required

6. Connecting forward to Pattern 13 (Union Find): if a graph connectivity problem arrives and you need dynamic connectivity (edges added one by one, query component membership between adds), Union Find beats BFS/DFS. Pattern 13 is the "alternative for connectivity" when Pattern 12 is too slow or structurally wrong.

7. Connecting forward to Pattern 14 (Shortest Path): Pattern 12's BFS handles unweighted graphs. Pattern 14 adds weights. The Dijkstra template is BFS with a priority queue replacing the regular queue and distances replacing the step count. If you've internalized Pattern 12's BFS template, Pattern 14's Dijkstra is one data structure substitution away.

---

## SIGN-OFF CRITERIA

**Tier 1 (Non-negotiable):**
- [ ] Grid BFS template from memory — correct bounds check order, mark before push, 4-direction arrays — under 3 minutes
- [ ] State the difference between undirected and directed cycle detection — parent tracking vs 3-color — and WHY they differ
- [ ] State the multi-source BFS rule: ALL sources pushed BEFORE BFS starts (not one at a time)

**Tier 2:**
- [ ] LC 200 (Number of Islands) — 5 minutes, zero bugs
- [ ] LC 994 (Rotting Oranges) — multi-source BFS, correct minute count, -1 case
- [ ] LC 207 (Course Schedule) — 3-color directed cycle, outer loop for disconnected graph

**Tier 3:**
- [ ] LC 417 (Pacific Atlantic) — reverse BFS from borders, height >= condition
- [ ] LC 133 (Clone Graph) — register clone BEFORE recursing to handle cycles
- [ ] LC 127 (Word Ladder) — implicit graph BFS, unordered_set, level-by-level step counting

**Tier 4 (attempt before sign-off):**
- [ ] LC 130 (Surrounded Regions) — border DFS then flip remaining
- [ ] LC 934 (Shortest Bridge) — DFS to find first island + BFS to expand, two-phase

**Sign-off test:** I give you one unseen graph problem. You must:
1. Classify it: grid or adjacency-list graph? BFS or DFS? Which sub-pattern (connectivity, shortest path, cycle, bipartite, multi-source, implicit)?
2. State the time complexity before coding
3. Write the solution including graph construction if needed
4. Handle the disconnected graph edge case (outer loop over all nodes)

Type `SIGN OFF P12` when ready.

---

*Pattern 12 Complete — Graphs: BFS and DFS*
*Next: Pattern 13 — Graphs: Union Find (DSU)*
