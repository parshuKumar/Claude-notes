# PATTERN 13 — GRAPHS: UNION FIND (DSU)
## Disjoint Set Union — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**What most people get wrong about Union Find:**

They learn `find()` and `union()` by rote, pass a few easy problems, and think they know it. Then they hit "Accounts Merge" (LC 721) and freeze — because they never learned how to map arbitrary labels (strings, coordinates, equations) to DSU indices.

The **real skill** is not implementing DSU. Any programmer can implement it in 5 minutes. The real skill is **recognizing** when a problem is secretly a DSU problem, and **adapting** the structure to handle weighted edges, 2D grids, or indirect connectivity.

**Where DSU beats BFS/DFS:**
- You need to **process edges one at a time** (online queries) — BFS/DFS require the full graph
- You need to **detect cycles** while building a graph
- You need **Kruskal's MST** (sort edges + DSU)
- N is huge but edges are sparse (DSU is O(α(N)) per operation, nearly O(1))

**Where BFS/DFS still wins:**
- Shortest path (DSU knows connectivity, not distance)
- Finding actual paths (DSU only answers "same component?")
- Directed graphs (DSU is inherently undirected)

**The α(N) fact:** With path compression + union by rank, every operation is O(α(N)) amortized, where α is the **inverse Ackermann function**. For all practical N (up to 10^600), α(N) ≤ 5. It is effectively O(1).

---

## SECTION 2 — ELI5 (EXPLAIN LIKE I'M 5)

Imagine you have a bunch of friend groups. Each person starts in their own group (alone).

When two people become friends, their groups **merge** into one. You need a "representative" (the **root**) for each group so you can quickly check "are Alice and Bob in the same group?"

**Without optimization:** You'd have to ask "who's your group leader?" → they ask their leader → their leader asks their leader → ... This chain can be very long.

**Path compression:** The first time you trace a long chain to find the root, you **shortcut** every node directly to the root. Future queries on the same chain are instant.

**Union by rank:** When merging two groups, you always attach the **smaller group's tree under the larger group's root**. This prevents the tree from growing tall.

Together, these two tricks make every "who's in my group?" query **nearly instant**, no matter how many merges happened.

---

## SECTION 3 — FORMALLY DEFINED

**Definition:** A Disjoint Set Union (DSU) / Union-Find data structure maintains a partition of a set S = {0, 1, ..., N-1} into disjoint subsets. It supports two operations:

1. **find(x):** Returns the **canonical representative** (root) of the subset containing x
2. **union(x, y):** Merges the subsets containing x and y into a single subset

**Representation:**
- `parent[i]`: parent of node i. If `parent[i] == i`, then i is the root of its component.
- `rank[i]` (or `size[i]`): used to keep the tree balanced during union

**Path Compression (find with compression):**
```
find(x):
    if parent[x] != x:
        parent[x] = find(parent[x])   // ← direct link to root
    return parent[x]
```

**Union by Rank:**
```
union(x, y):
    rx = find(x), ry = find(y)
    if rx == ry: return   // already same component
    if rank[rx] < rank[ry]: swap(rx, ry)
    parent[ry] = rx       // attach smaller rank under larger rank
    if rank[rx] == rank[ry]: rank[rx]++
```

**Amortized Complexity:** O(α(N)) per operation with both optimizations, where α is the inverse Ackermann function (≤ 5 for all practical N). Space: O(N).

**Key properties:**
- Only works correctly on **undirected** connectivity
- `find(x) == find(y)` ↔ x and y are in the same connected component
- Merging is **irreversible** (standard DSU cannot split components)

---

## SECTION 4 — RECOGNITION SIGNALS

**Read the problem statement. If you see ANY of these, think DSU:**

✅ **Strong signals (almost always DSU or graph problem where DSU works):**
1. "Find connected components" or "count groups/clusters" — classic
2. "Check if two nodes are in the same component" — find() query
3. "Process edges/connections one by one and answer queries" — online dynamic connectivity
4. "Detect a cycle while adding edges" — if `find(u) == find(v)` before `union(u, v)`, cycle!
5. "Accounts with same email belong to same person" — component merging by shared element
6. "Merge groups of people/items based on a relation" — union by relation

⚠️ **Medium signals (may be DSU, verify by thinking about connectivity):**
7. "N ≤ 10^5 nodes, M ≤ 10^5 edges, many queries about connectivity" — DSU faster than repeated BFS
8. "Grid problem where you add cells one by one" — 2D DSU with index mapping
9. "Equality/inequality constraints on variables" — weighted DSU or union by equality
10. "Kruskal's MST" — always uses DSU

❌ **FAKE signals (looks like DSU but isn't):**
- "Find the shortest path between two nodes" → Dijkstra/BFS, DSU doesn't know distances
- "Directed graph, does A reach B?" → BFS/DFS on directed graph, DSU is undirected
- "Topological ordering" → Topo sort, DSU doesn't handle direction
- "Find all paths" → DFS backtracking, DSU only answers yes/no connectivity

**The one-question test:** *"Do I only need to know whether two things are connected, or do I need the actual path/distance?"* — If only connectivity, DSU works. If path/distance, use BFS/Dijkstra.

---

## SECTION 5 — THE CORE DSU TEMPLATE

This is the complete, production-ready DSU class you will use for 95% of problems.

```cpp
// ─────────────────────────────────────────────
// TEMPLATE 1: Standard DSU (path compression + union by rank)
// ─────────────────────────────────────────────
struct DSU {
    vector<int> parent, rank_;
    int components;   // track number of connected components

    DSU(int n) : parent(n), rank_(n, 0), components(n) {
        iota(parent.begin(), parent.end(), 0);  // parent[i] = i
    }

    int find(int x) {
        if (parent[x] != x)
            parent[x] = find(parent[x]);  // path compression
        return parent[x];
    }

    bool unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;  // already connected — no merge done

        // union by rank: attach smaller tree under larger tree
        if (rank_[rx] < rank_[ry]) swap(rx, ry);
        parent[ry] = rx;
        if (rank_[rx] == rank_[ry]) rank_[rx]++;
        components--;
        return true;  // merge happened
    }

    bool connected(int x, int y) {
        return find(x) == find(y);
    }
};
```

**WHY `iota`?** It fills `parent[0..n-1]` with 0, 1, 2, ..., n-1 in one line. Each node starts as its own root.

**WHY return bool from `unite`?** Returning false when already connected is essential for cycle detection (the whole point of that check). Returning true tells you "yes, a new edge was added."

**WHY `rank_` not `rank`?** `rank` is a reserved identifier in some C++ contexts. Use `rnk`, `rank_`, or `sz` to be safe.

**HOW TO ADAPT:**
- Replace `rank_` with `size_` for **union by size** (attach smaller component under larger; `size_[rx] += size_[ry]` instead of rank check)
- Add `maxComponent` field, update `maxComponent = max(maxComponent, size_[rx])` after each union — useful for "largest component size" queries
- For **weighted DSU**, add `weight[]` array (see Template 5)

---

## SECTION 6 — TEMPLATE 2: CONNECTED COMPONENTS COUNTING

```cpp
// Count number of connected components in an undirected graph
// LC 323 — Number of Connected Components in an Undirected Graph
// Input: n nodes, edges list
// Output: number of components

int countComponents(int n, vector<vector<int>>& edges) {
    DSU dsu(n);
    for (auto& e : edges) {
        dsu.unite(e[0], e[1]);
    }
    return dsu.components;  // decremented by 1 on each successful union
}
```

**TRACE through example:**
```
n = 5, edges = [[0,1],[1,2],[3,4]]

Initial state:
  parent = [0, 1, 2, 3, 4]
  components = 5

unite(0, 1): find(0)=0, find(1)=1, different → merge. parent[1]=0, components=4
  parent = [0, 0, 2, 3, 4]

unite(1, 2): find(1)=find(1)→parent[1]=0→return 0. find(2)=2.
             0 and 2 differ → merge. parent[2]=0, components=3
  parent = [0, 0, 0, 3, 4]

unite(3, 4): find(3)=3, find(4)=4, different → merge. parent[4]=3, components=2
  parent = [0, 0, 0, 3, 3]

Answer: dsu.components = 2
Components: {0, 1, 2} and {3, 4}
```

**WHY `components` starts at n:** Every node is its own component initially. Each successful `unite()` call reduces by exactly 1. Final count = initial n - (number of edges that actually connected different components).

---

## SECTION 7 — TEMPLATE 3: CYCLE DETECTION IN UNDIRECTED GRAPH

```cpp
// Detect if adding an edge creates a cycle
// LC 684 — Redundant Connection
// Key insight: if find(u) == find(v) BEFORE uniting, the edge (u,v) is redundant
//              because u and v are ALREADY connected → adding this edge creates a cycle

vector<int> findRedundantConnection(vector<vector<int>>& edges) {
    int n = edges.size();
    DSU dsu(n + 1);  // nodes are 1-indexed

    for (auto& e : edges) {
        int u = e[0], v = e[1];
        if (dsu.connected(u, v)) {
            return e;           // ← this edge creates the cycle
        }
        dsu.unite(u, v);
    }
    return {};  // unreachable for valid input
}
```

**WHY this works — proof by invariant:**
- **Invariant:** After processing edges 1..k, `dsu` represents exactly the connectivity induced by edges 1..k.
- When processing edge (u, v):
  - If `find(u) != find(v)`: u and v are in different components. This edge connects them. **No cycle** — unite.
  - If `find(u) == find(v)`: u and v are already in the same component. There is already a path from u to v. Adding this edge creates a **second path** → cycle. **Return this edge.**
- The problem guarantees exactly one redundant edge, so we return the first one we find.

**TRACE:**
```
edges = [[1,2],[1,3],[2,3]]

unite(1,2): find(1)=1, find(2)=2, different → unite. No cycle.
unite(1,3): find(1)=1, find(3)=3, different → unite. No cycle.
unite(2,3): find(2)=find(2)→1. find(3)=find(3)→1. Same root 1!
            → return [2,3]   ← this is the redundant edge
```

---

## SECTION 8 — TEMPLATE 4: ACCOUNTS MERGE (DSU WITH STRING MAPPING)

This is the template for problems where the "nodes" are not integers but strings/emails/arbitrary labels. You maintain a **mapping from label to integer index**.

```cpp
// LC 721 — Accounts Merge
// Key idea: emails belonging to the same account are connected.
//           If two accounts share an email, they are the same person.
//           All emails of one person form a connected component.

vector<vector<string>> accountsMerge(vector<vector<string>>& accounts) {
    // Step 1: Map each unique email to an integer index
    unordered_map<string, int> emailIndex;   // email → index
    unordered_map<string, string> emailName; // email → owner name
    int idx = 0;

    for (auto& account : accounts) {
        string name = account[0];
        for (int i = 1; i < account.size(); i++) {
            string email = account[i];
            if (emailIndex.find(email) == emailIndex.end()) {
                emailIndex[email] = idx++;   // assign new index
            }
            emailName[email] = name;
        }
    }

    // Step 2: Build DSU on email indices
    DSU dsu(idx);  // idx = total unique emails

    for (auto& account : accounts) {
        // Union all emails in this account with the first email
        int root = emailIndex[account[1]];
        for (int i = 2; i < account.size(); i++) {
            dsu.unite(root, emailIndex[account[i]]);
        }
    }

    // Step 3: Group emails by their root representative
    unordered_map<int, vector<string>> components;
    for (auto& [email, i] : emailIndex) {
        int rootIdx = dsu.find(i);
        components[rootIdx].push_back(email);
    }

    // Step 4: Build result (sort each group, prepend name)
    vector<vector<string>> result;
    for (auto& [rootIdx, emails] : components) {
        sort(emails.begin(), emails.end());
        string name = emailName[emails[0]];
        vector<string> merged = {name};
        merged.insert(merged.end(), emails.begin(), emails.end());
        result.push_back(merged);
    }

    return result;
}
```

**TRACE through small example:**
```
accounts = [
  ["John", "johnsmith@mail.com", "john_newyork@mail.com"],
  ["John", "johnsmith@mail.com", "john00@mail.com"],
  ["Mary", "mary@mail.com"],
  ["John", "johnnybravo@mail.com"]
]

Step 1 — email→index map:
  johnsmith@mail.com    → 0  (owner: John)
  john_newyork@mail.com → 1  (owner: John)
  john00@mail.com       → 2  (owner: John)
  mary@mail.com         → 3  (owner: Mary)
  johnnybravo@mail.com  → 4  (owner: John)

Step 2 — union by account:
  Account 1: root=0. unite(0,1) → {0,1}
  Account 2: root=0. unite(0,2) → {0,1,2}   [0 is already root of 0]
  Account 3: root=3. no more emails. {3}
  Account 4: root=4. no more emails. {4}

Step 3 — group by root:
  find(0)=0, find(1)=0, find(2)=0 → component 0: [johnsmith, john_newyork, john00]
  find(3)=3 → component 3: [mary@mail.com]
  find(4)=4 → component 4: [johnnybravo@mail.com]

Step 4 — sort + add name:
  ["John", "john00@mail.com", "john_newyork@mail.com", "johnsmith@mail.com"]
  ["John", "johnnybravo@mail.com"]
  ["Mary", "mary@mail.com"]
```

**HOW TO ADAPT this pattern:**
- Replace emails with any "shared identifier" (phone numbers, employee IDs, coordinates)
- The key invariant: if two accounts share ANY identifier, they belong to the same component
- When unioning, always pick `account[1]` as the "anchor" and union all others to it — this is enough because transitivity handles chains

---

## SECTION 9 — TEMPLATE 5: WEIGHTED DSU (RELATIVE RELATIONS)

For problems where edges carry **weights representing relationships** (ratios, differences), not just connectivity. The `weight[x]` means "what is the value at x relative to its root?"

**Classic problem:** LC 990 — Satisfiability of Equality Equations (simplified weighted DSU)
**Classic hard problem:** LC 399 — Evaluate Division (ratio DSU)

```cpp
// Weighted DSU for ratio problems (like LC 399 Evaluate Division)
// weight[x] = value(x) / value(root(x))
// So if we know value(a)/value(b) = 2.0, and a and b are in the same component,
// we maintain: weight[a] = value(a) / value(root)
//              weight[b] = value(b) / value(root)

struct WeightedDSU {
    vector<int> parent;
    vector<double> weight;  // weight[x] = x / root(x)

    WeightedDSU(int n) : parent(n), weight(n, 1.0) {
        iota(parent.begin(), parent.end(), 0);
    }

    // Returns {root, weight_to_root}
    pair<int, double> find(int x) {
        if (parent[x] == x) return {x, 1.0};

        auto [root, w] = find(parent[x]);
        parent[x] = root;
        weight[x] *= w;   // path compression: update weight to point directly to root
        return {root, weight[x]};
    }

    // Register: value(x) / value(y) = ratio
    void unite(int x, int y, double ratio) {
        auto [rx, wx] = find(x);
        auto [ry, wy] = find(y);
        if (rx == ry) return;

        parent[rx] = ry;
        // wx = x/rx, wy = y/ry, ratio = x/y
        // we need rx/ry:
        // x = wx * rx
        // y = wy * ry
        // x/y = ratio → (wx * rx) / (wy * ry) = ratio
        // rx/ry = ratio * wy / wx
        weight[rx] = ratio * wy / wx;
    }

    // Query: what is value(x) / value(y)?
    double query(int x, int y) {
        auto [rx, wx] = find(x);
        auto [ry, wy] = find(y);
        if (rx != ry) return -1.0;  // not connected — unknown
        // wx = x/rx, wy = y/ry, rx == ry
        // x/y = (x/rx) / (y/ry) = wx / wy
        return wx / wy;
    }
};
```

**For the simpler LC 990 (equality equations — no actual weights, just same/different group):**

```cpp
// LC 990 — Satisfiability of Equality Equations
// "a==b" means unite(a, b)
// "a!=b" means check that find(a) != find(b) after all unions

bool equationsPossible(vector<string>& equations) {
    DSU dsu(26);  // only lowercase letters

    // First pass: process all == constraints (union)
    for (auto& eq : equations) {
        if (eq[1] == '=') {  // eq is "a==b" or "a!=b"
            dsu.unite(eq[0] - 'a', eq[3] - 'a');
        }
    }

    // Second pass: check all != constraints
    for (auto& eq : equations) {
        if (eq[1] == '!') {
            if (dsu.connected(eq[0] - 'a', eq[3] - 'a')) {
                return false;  // contradiction: eq says a!=b but they're connected
            }
        }
    }

    return true;
}
```

**WHY two passes?** All equality constraints must be processed FIRST to build the full connectivity graph. Only then can you check inequality constraints. If you interleave, you might check `a != b` before the chain `a == c == b` is fully built.

---

## SECTION 10 — TEMPLATE 6: DYNAMIC CONNECTIVITY (NUMBER OF ISLANDS II)

For problems where cells/nodes are **added one by one** and you need to maintain connectivity after each addition. This is where DSU truly shines over BFS — BFS would need to re-run from scratch on each addition.

```cpp
// LC 305 — Number of Islands II
// Grid starts empty. We "land" cells one at a time.
// After each land operation, report the count of islands.

// 2D → 1D index mapping: (r, c) → r * cols + c

vector<int> numIslands2(int rows, int cols, vector<vector<int>>& positions) {
    DSU dsu(rows * cols);
    vector<vector<bool>> grid(rows, vector<bool>(cols, false));
    vector<int> result;
    int islands = 0;

    int dirs[4][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    for (auto& pos : positions) {
        int r = pos[0], c = pos[1];
        int cur = r * cols + c;

        if (grid[r][c]) {
            // Duplicate position — island count unchanged
            result.push_back(islands);
            continue;
        }

        grid[r][c] = true;
        islands++;  // new land cell = new island initially
        dsu.parent[cur] = cur;  // make it its own root

        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            int neighbor = nr * cols + nc;
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                && grid[nr][nc]
                && !dsu.connected(cur, neighbor)) {
                dsu.unite(cur, neighbor);
                islands--;  // merging two islands → one fewer island
            }
        }

        result.push_back(islands);
    }

    return result;
}
```

**TRACE through small example:**
```
rows=3, cols=3, positions=[[0,0],[0,1],[1,2],[2,1],[1,1]]

Initial: grid all false, islands=0

Add (0,0): cur=0. Mark grid[0][0]=true. islands=1.
           Neighbors: (0,1)=false, (1,0)=false. No merges.
           islands=1. Result: [1]

Add (0,1): cur=1. Mark grid[0][1]=true. islands=2.
           Check neighbor (0,0): grid[0][0]=true, !connected(1,0). Unite. islands=1.
           Check neighbor (0,2): false. (1,1): false.
           islands=1. Result: [1,1]

Add (1,2): cur=5. Mark true. islands=2.
           Neighbors: (0,2)=false, (2,2)=false, (1,1)=false.
           No merges. islands=2. Result: [1,1,2]

Add (2,1): cur=7. Mark true. islands=3.
           Neighbors: (2,0)=false,(2,2)=false,(1,1)=false,(3,1) OOB.
           No merges. islands=3. Result: [1,1,2,3]

Add (1,1): cur=4. Mark true. islands=4.
           Check (0,1): true. unite(4,1). find(4)=4, find(1)=0. Unite! islands=3.
           Check (1,0): false.
           Check (1,2): true. unite(4,5). find(4)=0(via 1), find(5)=5. Unite! islands=2.
           Check (2,1): true. unite(4,7). find(4)=0, find(7)=7. Unite! islands=1.
           islands=1. Result: [1,1,2,3,1]
```

**KEY INSIGHT about the `islands` counter:**
- Start each new land as `islands++` (new island)
- For each neighboring land cell NOT already connected: `unite()` merges two islands → `islands--`
- The `if (!dsu.connected(cur, neighbor))` check prevents double-counting when two neighbors are already in the same component (don't decrement twice for the same merge)

---

## SECTION 11 — TEMPLATE 7: DSU WITH SIZE (UNION BY SIZE)

Alternative to union by rank. Preferred when you need to **query component sizes** quickly.

```cpp
struct DSUWithSize {
    vector<int> parent, size_;
    int components;

    DSUWithSize(int n) : parent(n), size_(n, 1), components(n) {
        iota(parent.begin(), parent.end(), 0);
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    bool unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;

        // attach smaller component under larger
        if (size_[rx] < size_[ry]) swap(rx, ry);
        parent[ry] = rx;
        size_[rx] += size_[ry];  // ← accumulate size at root
        components--;
        return true;
    }

    int getSize(int x) { return size_[find(x)]; }

    bool connected(int x, int y) { return find(x) == find(y); }
};
```

**WHY size over rank for some problems:**
- `rank` only tracks an upper bound on tree height, not actual component size
- `getSize(x)` tells you how many nodes are in x's component — essential for LC 924 (minimize malware), LC 778 (swim in rising water analysis), LC 1202 (smallest string with swaps)

**HOW TO ADAPT:**
- For "largest component size ever seen": add `int maxComp = 1;` and update `maxComp = max(maxComp, size_[rx])` after each union
- For "sum of values in component": add `vector<int> sumVal;` and `sumVal[rx] += sumVal[ry]` on merge

---

## SECTION 12 — TEMPLATE 8: SMALLEST STRING WITH SWAPS (DSU + COMPONENT SORTING)

This template handles a common pattern: **nodes in the same component can be rearranged freely**, so sort within each component to minimize the result.

```cpp
// LC 1202 — Smallest String With Swaps
// Pairs (i,j) mean we can swap s[i] and s[j] any number of times
// → indices i and j are in the same component
// → within a component, sort characters to get lexicographically smallest result

string smallestStringWithSwaps(string s, vector<vector<int>>& pairs) {
    int n = s.size();
    DSUWithSize dsu(n);

    for (auto& p : pairs) {
        dsu.unite(p[0], p[1]);
    }

    // Group indices by their root
    unordered_map<int, vector<int>> components;
    for (int i = 0; i < n; i++) {
        components[dsu.find(i)].push_back(i);
    }

    string result = s;
    for (auto& [root, indices] : components) {
        // Extract characters in this component
        vector<char> chars;
        for (int idx : indices) chars.push_back(s[idx]);

        // Sort indices and characters separately, then place sorted chars at sorted positions
        sort(indices.begin(), indices.end());  // sorted positions
        sort(chars.begin(), chars.end());       // sorted characters

        for (int k = 0; k < indices.size(); k++) {
            result[indices[k]] = chars[k];     // assign smallest char to smallest index
        }
    }

    return result;
}
```

**WHY can we sort freely within a component?**

If i and j are connected, we can swap s[i] and s[j]. If i-j-k are all connected (via any chain of pairs), we can achieve **any permutation** of {s[i], s[j], s[k]} through a sequence of adjacent swaps. Proof: any permutation can be expressed as a product of transpositions, and transitively connected swaps give us all transpositions within the component.

**TRACE:**
```
s = "dcab", pairs = [[0,3],[1,2]]

DSU: unite(0,3) → {0,3}. unite(1,2) → {1,2}.

Components:
  find(0)=0: indices [0,3], chars = s[0]='d', s[3]='b' → sorted: ['b','d']
  find(1)=1: indices [1,2], chars = s[1]='c', s[2]='a' → sorted: ['a','c']

Place sorted chars at sorted indices:
  indices [0,3] get chars ['b','d'] → result[0]='b', result[3]='d'
  indices [1,2] get chars ['a','c'] → result[1]='a', result[2]='c'

result = "bacd"
```

---

## SECTION 13 — VARIANT: MOST STONES REMOVED (DSU ON ROWS AND COLUMNS)

This is a clever trick: treat rows and columns as nodes. A stone at (r, c) connects **row r** to **column c**. Stones in the same component can all be removed except one (you always leave one per component).

```cpp
// LC 947 — Most Stones Removed with Same Row or Column
// Key insight: stones (r1,c1) and (r2,c2) where r1==r2 or c1==c2 are in same component
// A connected component of k stones → can remove k-1 stones
// Answer = n - (number of components)

// Trick: offset column indices by 10001 to avoid collision with row indices
// (rows and columns become nodes in the same DSU)

int removeStones(vector<vector<int>>& stones) {
    const int OFFSET = 10001;
    DSU dsu(OFFSET * 2);  // rows: 0..10000, cols: 10001..20001

    for (auto& stone : stones) {
        dsu.unite(stone[0], stone[1] + OFFSET);
    }

    // Count distinct components that CONTAIN at least one stone
    unordered_set<int> roots;
    for (auto& stone : stones) {
        roots.insert(dsu.find(stone[0]));
    }

    return stones.size() - roots.size();
}
```

**WHY offset columns?** If we simply use row and column indices, row 5 and column 5 would be treated as the same node. Adding OFFSET to column indices gives columns a separate namespace in the same DSU. This is the "virtual node" trick.

**WHY count roots via stone positions only?**
We don't want to count row/column nodes that have no stones. A component is "live" only if it contains at least one stone position. Iterating over stone positions and finding their roots gives exactly the set of live component roots.

---

## SECTION 14 — VARIANT: SWIM IN RISING WATER (DSU APPROACH)

```cpp
// LC 778 — Swim in Rising Water
// DSU approach: sort all cells by elevation, add them in order
// We need the minimum T such that (0,0) and (n-1,n-1) are connected
// at time T = max elevation on the path

int swimInWater(vector<vector<int>>& grid) {
    int n = grid.size();
    // Create list of (elevation, row, col), sort by elevation
    vector<tuple<int,int,int>> cells;
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            cells.push_back({grid[i][j], i, j});
    sort(cells.begin(), cells.end());

    DSU dsu(n * n);
    int dirs[4][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    for (auto& [elev, r, c] : cells) {
        // "Add" this cell: connect to already-added neighbors
        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= n || nc < 0 || nc >= n) continue;
            // Only connect if neighbor has lower/equal elevation (already added)
            if (grid[nr][nc] <= elev) {
                dsu.unite(r * n + c, nr * n + nc);
            }
        }
        // Check if (0,0) and (n-1,n-1) are now connected
        if (dsu.connected(0, (n-1)*n + (n-1))) {
            return elev;  // current elevation T = this cell's elevation
        }
    }

    return -1;  // unreachable
}
```

**WHY this is correct:** We process cells in increasing elevation order. When we "add" a cell, its elevation is T. All cells already added have elevation ≤ T. Connecting to lower-elevation neighbors captures all possible path connections at time T. The first moment (0,0) and (n-1,n-1) become connected is our answer.

**Alternative approaches:** Binary search + BFS (also O(n² log n)), Dijkstra (treats elevation as "cost" to step on a cell). DSU approach is clean and O(n² α(n²)) ≈ O(n²).

---

## SECTION 15 — COMPLEXITY TABLE

| Operation | Basic (no optimization) | + Path Compression | + Path Comp + Union by Rank/Size |
|-----------|------------------------|-------------------|----------------------------------|
| `find(x)` | O(N) worst | O(log N) amortized | O(α(N)) amortized ≈ O(1) |
| `unite(x,y)` | O(N) worst | O(log N) amortized | O(α(N)) amortized ≈ O(1) |
| Space | O(N) | O(N) | O(N) |
| N operations | O(N²) | O(N log N) | O(N α(N)) ≈ O(N) |

**α(N) values in practice:**
| N | α(N) |
|---|------|
| 1 - 4 | 1 |
| 5 - 16 | 2 |
| 17 - 10^10000 | ≤ 4 |

You will **never** see α(N) > 4 in a competitive programming problem. Treat it as constant.

**Real-world cost comparison for N = 10^5 operations:**
- No optimization: ~10^10 ops (TLE guaranteed)
- Path compression only: ~1.7 × 10^6 ops
- Both optimizations: ~5 × 10^5 ops (essentially linear)

**Space:** All variants are O(N) — two arrays of size N.

---

## SECTION 16 — COMMON MISTAKES

### Mistake 1: Forgetting path compression (using iterative find without compression)

```cpp
// WRONG — iterative find without compression
// This is O(log N) amortized with union by rank but misses the O(α(N)) speedup
// More importantly, it's WRONG if you use rank-based union without compression because
// find() is supposed to return the ROOT, but rank-based trees can be tall without compression
int find(int x) {
    while (parent[x] != x) x = parent[x];  // no compression!
    return x;
}

// CORRECT — recursive with path compression
int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
}

// ALSO CORRECT — iterative with path compression (two-pass)
int find(int x) {
    int root = x;
    while (parent[root] != root) root = parent[root];  // find root
    while (parent[x] != root) {
        int next = parent[x];
        parent[x] = root;   // compress
        x = next;
    }
    return root;
}
```

**WHY it matters:** Without compression, in the worst case (long chains), repeated `find()` calls are O(log N) each. With compression + rank, they're O(α(N)). For 10^5 nodes and 10^5 queries, the difference is noticeable.

---

### Mistake 2: Union without checking if already connected (missing the return value)

```cpp
// WRONG — for cycle detection, you MUST check if already connected
void unite(int x, int y) {
    parent[find(y)] = find(x);  // always merges, even if same component!
}

// Usage for cycle detection:
for (auto& e : edges) {
    unite(e[0], e[1]);  // BUG: doesn't detect the cycle
}

// CORRECT — return false if already same component (that's the cycle!)
bool unite(int x, int y) {
    int rx = find(x), ry = find(y);
    if (rx == ry) return false;  // ← CYCLE DETECTED HERE
    // ... rest of union logic
    return true;
}

// Usage:
for (auto& e : edges) {
    if (!dsu.unite(e[0], e[1])) {
        // This edge created a cycle — handle it
    }
}
```

---

### Mistake 3: Off-by-one in DSU initialization (1-indexed vs 0-indexed nodes)

```cpp
// WRONG — node labels are 1-indexed but DSU is initialized for 0-indexed
// LC 684: nodes are 1-indexed (1 to N)
int n = edges.size();
DSU dsu(n);  // BUG: creates indices 0..n-1, but nodes go up to n

for (auto& e : edges) {
    dsu.unite(e[0], e[1]);  // e[0] and e[1] can be up to n → out of bounds!
}

// CORRECT
DSU dsu(n + 1);  // nodes 0..n, we use 1..n
```

**HOW TO REMEMBER:** Always check if the problem uses 1-indexed nodes. If so, initialize DSU with `n+1`.

---

### Mistake 4: Wrong 2D→1D index mapping

```cpp
// WRONG — using row-major but getting the formula backwards
int idx = c * rows + r;  // BUG: this is column-major

// CORRECT — row-major (standard)
int idx = r * cols + c;  // r is row, cols is total columns

// VERIFY: for a 3x4 grid, cell (2,3) should be index 2*4+3 = 11 (last cell)
// (2,3) → 2*4+3 = 11 ✓
// (0,0) → 0 ✓
// (1,0) → 4 ✓ (start of second row)
```

---

### Mistake 5: Interleaving union and inequality checks in LC 990 (processing order matters)

```cpp
// WRONG — process == and != in a single pass
for (auto& eq : equations) {
    if (eq[1] == '=') dsu.unite(eq[0]-'a', eq[3]-'a');
    else {
        // BUG: at this point, not all == constraints may have been processed yet!
        // e.g., if ["a==b", "b==c", "a!=c"] comes in that order, when we check a!=c,
        // we might not have processed b==c yet if it comes after a!=c
        if (dsu.connected(eq[0]-'a', eq[3]-'a')) return false;
    }
}

// CORRECT — two separate passes
for (auto& eq : equations)
    if (eq[1] == '=') dsu.unite(eq[0]-'a', eq[3]-'a');

for (auto& eq : equations)
    if (eq[1] == '!') {
        if (dsu.connected(eq[0]-'a', eq[3]-'a')) return false;
    }
return true;
```

---

### Mistake 6: Re-using a DSU that's been modified (state carries over between test cases)

```cpp
// WRONG — modifying DSU in solve() affects subsequent calls
DSU dsu(n);  // declared globally or as member

void solve(vector<pair<int,int>>& edges) {
    for (auto& [u, v] : edges) dsu.unite(u, v);
    // ... answer queries
    // But dsu is now modified! If solve() is called again, the old unions persist!
}

// CORRECT — always create a fresh DSU per test case / per problem instance
void solve(int n, vector<pair<int,int>>& edges) {
    DSU dsu(n);  // ← local, fresh each time
    for (auto& [u, v] : edges) dsu.unite(u, v);
    // ...
}
```

---

## SECTION 17 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Number of Connected Components | 323 | Basic DSU + component count |
| 2 | Redundant Connection | 684 | Cycle detection — return false from unite |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Number of Provinces | 547 | DSU on adjacency matrix (not edge list) |
| 4 | Accounts Merge | 721 | DSU + string-to-index mapping |
| 5 | Longest Consecutive Sequence (DSU) | 128 | Alternative DSU approach vs hash set |
| 6 | Satisfiability of Equality Equations | 990 | Two-pass: union then check inequalities |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Number of Islands II | 305 | Dynamic DSU, 2D→1D index |
| 8 | Smallest String With Swaps | 1202 | DSU + sort within components |
| 9 | Most Stones Removed | 947 | DSU on rows + columns (offset trick) |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Swim in Rising Water | 778 | Sort by elevation + DSU |
| 11 | Minimize Malware Spread | 924 | DSU with size tracking |

---

**Solve in this order.** LC 323 and 684 are mandatory warmup — they introduce the two fundamental operations. 721 (Accounts Merge) is THE core interview problem for DSU. 305 (Islands II) is THE dynamic DSU problem.

---

## SECTION 18 — PATTERN CONNECTIONS

**DSU is closely connected to:**

1. **Pattern 12 (BFS/DFS):** DSU is an alternative to BFS/DFS for static connectivity. BFS gives shortest path; DSU gives connectivity. Both solve "number of islands" but DSU is preferred for dynamic versions.

2. **Pattern 31 (MST — Kruskal's):** Kruskal's algorithm is literally "sort edges by weight, add edge if DSU says it doesn't create a cycle." DSU is the core of Kruskal's.

3. **Pattern 14 (Shortest Path):** After building MST with DSU, you can answer "what is the maximum edge on the path between u and v in the MST?" efficiently using LCA on the MST (advanced, but the connection is real).

4. **Pattern 34 (Greedy):** DSU + greedy is a common combination. Redundant Connection is solved greedily (process edges in order) with DSU as the data structure.

5. **Pattern 30 (Topological Sort):** Both check for cycles, but Topo Sort handles directed graphs while DSU handles undirected. When you see cycle detection, first ask: directed or undirected?

**When to use DSU vs BFS/DFS:**

| Scenario | Better Choice |
|----------|---------------|
| Static graph, check connectivity | Either (DSU slightly faster) |
| Dynamic graph (add edges online) | DSU (BFS would need to rerun) |
| Find shortest path | BFS/Dijkstra |
| Count connected components | Either (DSU cleaner) |
| Directed graph cycle detection | DFS (3-color) — DSU is undirected! |
| Building MST (Kruskal's) | DSU |
| All pairs connectivity | Floyd-Warshall or DSU |

---

## SECTION 19 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 547 — Number of Provinces

**Interviewer:** "You're given an N×N adjacency matrix where `isConnected[i][j] = 1` if cities i and j are directly connected. Find the total number of provinces."

**Candidate approach:**

*[First 2 minutes — clarify and plan]*
> "So a province is a set of directly or indirectly connected cities. That's essentially counting connected components. My first instinct is BFS/DFS, but this is also a clean DSU problem. I'll use DSU since it's more efficient for just counting components."

*[State the plan before coding]*
> "I'll initialize a DSU with N nodes. For each pair (i,j) where `isConnected[i][j] == 1` and `i < j` (to avoid double-processing), I'll call `unite(i, j)`. At the end, `dsu.components` is my answer."

*[Code — under 3 minutes]*
```cpp
int findCircleNum(vector<vector<int>>& isConnected) {
    int n = isConnected.size();
    DSU dsu(n);

    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {  // j = i+1 to avoid duplicates
            if (isConnected[i][j] == 1) {
                dsu.unite(i, j);
            }
        }
    }

    return dsu.components;
}
```

*[Analysis]*
> "Time: O(N² × α(N)) for iterating the matrix and DSU operations. Space: O(N) for the DSU arrays."

*[Interviewer follows up]: "What if instead of a matrix, we had an edge list with M edges? Would you change anything?"*
> "No change to the algorithm — just iterate the edge list instead of the matrix. Time becomes O((N+M) × α(N))."

**RED FLAGS:**
- Using DFS without noting that DSU is simpler here (not wrong, just less elegant)
- Missing the `j = i+1` optimization (not critical, but shows attention to detail)
- Not knowing the complexity of DSU operations

---

### Interview Simulation 2: LC 721 — Accounts Merge

**Interviewer:** "Given a list of accounts where each account is `[name, email1, email2, ...]`, merge accounts that share any email. Return sorted merged accounts."

**Candidate approach:**

*[First 3 minutes — clarify edge cases]*
> "Can two different names share the same email? The problem says emails are globally unique to a person, so no. Can the same account appear multiple times? I'll handle that via DSU naturally."

*[Key insight — state it explicitly before coding]*
> "This is a graph connectivity problem. Two accounts are the same person if they share any email. The 'share email' relation is transitive — if A shares email1 with B, and B shares email2 with C, then A, B, C are the same person. Classic DSU."

*[Plan]*
> "Step 1: Map every email to an integer index. Step 2: For each account, union all its emails (all emails in one account belong to the same person — connect them all to the first email). Step 3: Group emails by their DSU root. Step 4: Sort each group and prepend name."

*[Code — 8-10 minutes for this problem]*
```cpp
// [Writes Template 4 from Section 8]
```

*[Complexity analysis]*
> "N accounts, each with up to K emails. Total emails = N*K. Mapping step O(N*K). Union step O(N*K * α(N*K)). Grouping step O(N*K). Sort step O(N*K * log(N*K)). Overall: O(N*K * log(N*K))."

**RED FLAGS:**
- Jumping to code without stating the DSU insight
- Forgetting to initialize `emailIndex` before first access (causes segfault)
- Forgetting to sort BOTH `indices` and `chars` in the same order (result would be wrong — chars sorted but placed at wrong positions)
- Using `account.size()` as int directly without casting (signed/unsigned comparison warning)

---

### Interview Simulation 3: LC 990 — Satisfiability of Equality Equations

**Interviewer:** "Given equations like 'a==b', 'b!=c', determine if all equations can be satisfied simultaneously."

**Candidate approach:**

*[First 1 minute — spot the key constraint]*
> "So == is transitive: a==b and b==c implies a==c. This is exactly what a connected component captures. I need to check: for every != equation, are its two sides in different components? If yes for all, satisfiable. Otherwise, no."

*[Critical insight to state]*
> "Order matters: I MUST process ALL == equations before checking any !=. Otherwise I might check a != c before discovering a==b and b==c, and incorrectly conclude it's satisfiable."

*[Code — 3 minutes]*
```cpp
bool equationsPossible(vector<string>& equations) {
    DSU dsu(26);
    for (auto& eq : equations)
        if (eq[1] == '=') dsu.unite(eq[0]-'a', eq[3]-'a');
    for (auto& eq : equations)
        if (eq[1] == '!') 
            if (dsu.connected(eq[0]-'a', eq[3]-'a')) return false;
    return true;
}
```

*[Edge cases]*
> "What about `a != a`? That's a self-contradiction — `dsu.connected('a', 'a')` is always true since they have the same root, so we'd return false correctly."

**RED FLAGS:**
- Mixing the two passes (single-pass approach is wrong)
- Using `eq[1] == '!'` check but forgetting the second char is `=` (the format is `a!=b`, so `eq[1]` is `!` and `eq[2]` is `=`)
- Initializing DSU with wrong size (should be 26 for lowercase letters)

---

## SECTION 20 — MENTAL MODEL CHECKPOINT

Seven questions. Answer each before reading the answer.

---

**Q1: Why does DSU use path compression? What problem does it solve?**

> Without path compression, `find(x)` traces the parent chain up to the root. In a worst case (long chain), this is O(N) per call. After performing `find(x)` once, path compression makes `parent[x] = root` directly, so future `find(x)` calls are O(1). Over a sequence of M operations, this drops the amortized cost from O(log N) to O(α(N)).

---

**Q2: Can DSU work on directed graphs?**

> No. Standard DSU only tracks undirected connectivity. If edge (u→v) exists, DSU would record u and v in the same component — but in a directed graph, that doesn't mean v can reach u. For directed cycle detection, use the 3-color DFS (Pattern 12). For directed connectivity, use BFS/DFS or Kosaraju's SCC.

---

**Q3: In the Number of Islands II problem (LC 305), why do we check `!dsu.connected(cur, neighbor)` before calling `unite()`?**

> Suppose the newly added cell has 3 land neighbors A, B, C, and A and B are already in the same component. Without the check, we'd call `unite(cur, A)` → `islands--`, then `unite(cur, B)` → `islands--` again, even though A and B were already merged. That double-decrements `islands` incorrectly. The check ensures we only decrement once per distinct component that gets merged.

---

**Q4: What is the "two-pass" trick in LC 990 and why is it necessary?**

> In LC 990 (equality equations), if we process `==` and `!=` in a single pass, we might check a `!=` constraint before all the `==` constraints that might connect those variables have been processed. Example: equations `["a==b","b==c","a!=c"]` — if we check `a!=c` before seeing `b==c`, we'd think a and c are in different components and return true incorrectly. Two passes: first union all `==`, then check all `!=` on the complete union structure.

---

**Q5: In union by rank, what is `rank` measuring? Why doesn't it literally equal tree height?**

> `rank[x]` is an upper bound on the height of the subtree rooted at x. It starts at 0 for isolated nodes. When two trees of equal rank are merged, the new root's rank is incremented by 1. When trees of unequal rank are merged, the lower-rank root is attached under the higher-rank root, and the higher rank is unchanged. Path compression flattens trees, so the actual height after compression is often much less than `rank`. That's why `rank` is an upper bound, not the exact height.

---

**Q6: In the "Most Stones Removed" problem, why do we add OFFSET to column indices?**

> We're treating rows AND columns as nodes in a single DSU. Without offset, row index 5 and column index 5 would be the same DSU node. The offset (e.g., 10001) separates the two namespaces: rows use indices 0..10000, columns use 10001..20001. A stone at (r, c) then connects DSU node r to DSU node (c + OFFSET). This "virtual node" trick is used whenever you need to connect two different types of objects in the same DSU.

---

**Q7: In Accounts Merge, why do we union all emails in an account to `account[1]` (the first email) rather than chaining them (1→2→2→3→3→4...)?**

> Either approach is correct due to transitivity of DSU. If we unite each consecutive pair (1,2), (2,3), (3,4), all four end up in the same component. Uniting all to `account[1]` directly — unite(1,2), unite(1,3), unite(1,4) — achieves the same result. The "union all to first" approach is slightly cleaner and avoids any potential confusion. However, "chain unions" also work correctly. What matters is that all emails in the same account entry are in the same component after processing that entry.

---

## SECTION 21 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// DSU WITH RANK — Full production template
// ─────────────────────────────────────────────────────────────────
struct DSU {
    vector<int> parent, rnk;
    int comp;
    DSU(int n) : parent(n), rnk(n, 0), comp(n) {
        iota(parent.begin(), parent.end(), 0);
    }
    int find(int x) {
        return parent[x] == x ? x : parent[x] = find(parent[x]);
    }
    bool unite(int x, int y) {
        x = find(x); y = find(y);
        if (x == y) return false;
        if (rnk[x] < rnk[y]) swap(x, y);
        parent[y] = x;
        if (rnk[x] == rnk[y]) rnk[x]++;
        comp--;
        return true;
    }
    bool same(int x, int y) { return find(x) == find(y); }
};

// ─────────────────────────────────────────────────────────────────
// DSU WITH SIZE — Use when you need component sizes
// ─────────────────────────────────────────────────────────────────
struct DSU {
    vector<int> parent, sz;
    int comp;
    DSU(int n) : parent(n), sz(n, 1), comp(n) {
        iota(parent.begin(), parent.end(), 0);
    }
    int find(int x) {
        return parent[x] == x ? x : parent[x] = find(parent[x]);
    }
    bool unite(int x, int y) {
        x = find(x); y = find(y);
        if (x == y) return false;
        if (sz[x] < sz[y]) swap(x, y);
        parent[y] = x; sz[x] += sz[y];
        comp--; return true;
    }
    int size(int x) { return sz[find(x)]; }
    bool same(int x, int y) { return find(x) == find(y); }
};

// ─────────────────────────────────────────────────────────────────
// COMMON PATTERNS — Quick reference
// ─────────────────────────────────────────────────────────────────

// 1. Cycle detection in undirected graph
if (!dsu.unite(u, v)) { /* cycle detected! */ }

// 2. Count components
DSU dsu(n);
for (auto& e : edges) dsu.unite(e[0], e[1]);
int ans = dsu.comp;

// 3. Dynamic 2D grid (r,c) → index
int idx = r * cols + c;
if (grid[r][c] == LAND) {
    for (auto [nr,nc] : neighbors)
        if (grid[nr][nc] == LAND) dsu.unite(r*cols+c, nr*cols+nc);
}

// 4. Row+Column DSU (offset trick)
const int OFF = 10001;
dsu.unite(row, col + OFF);  // stone at (row,col) connects row to col

// 5. String → index mapping
unordered_map<string, int> idx;
int cnt = 0;
auto getId = [&](const string& s) {
    if (!idx.count(s)) idx[s] = cnt++;
    return idx[s];
};
DSU dsu(maxN);  // pre-allocate generously

// 6. Two-pass for constraint satisfaction (LC 990 pattern)
for (auto& eq : eqs) if (isEquality(eq)) dsu.unite(a(eq), b(eq));
for (auto& eq : eqs) if (isInequality(eq)) if (dsu.same(a(eq), b(eq))) return false;

// 7. Sort within components (LC 1202 pattern)
unordered_map<int, vector<int>> groups;
for (int i = 0; i < n; i++) groups[dsu.find(i)].push_back(i);
for (auto& [root, idxs] : groups) {
    vector<char> chars;
    for (int i : idxs) chars.push_back(s[i]);
    sort(idxs.begin(), idxs.end());
    sort(chars.begin(), chars.end());
    for (int k = 0; k < idxs.size(); k++) result[idxs[k]] = chars[k];
}
```

---

## SECTION 22 — WHAT TO DO AFTER THIS PATTERN

1. **Immediately solve (in order):** LC 323 → 684 → 547 → 721 → 990 → 305 → 1202 → 947 → 778 → 924

2. **Consolidate with Pattern 12:** Go back to Number of Islands (LC 200) and solve it with DSU instead of BFS. Notice: DSU needs you to start with all cells and "activate" land cells — slightly awkward. BFS is cleaner for static graphs. DSU shines in dynamic (Islands II, LC 305).

3. **Preview Pattern 31 (MST):** Kruskal's algorithm is the next natural step after DSU. It's literally: sort edges by weight + DSU cycle detection = MST.

4. **Contest application:** In Codeforces Div 2 C/D, whenever you see "group elements by relation" or "dynamic connectivity with queries," DSU is the go-to tool.

5. **Identify the DSU pattern in past problems:** Look at LC 128 (Longest Consecutive Sequence) — you solved it with a hash set, but there's a DSU solution. Understanding both deepens your pattern recognition.

---

## SECTION 23 — SIGN-OFF CRITERIA (PATTERN MASTERY TIERS)

### Tier 1 — Can implement
You can write the DSU struct from scratch in under 3 minutes with correct path compression and union by rank.

### Tier 2 — Can apply
You solve LC 323, 684, 547, and 990 cleanly without hints. You know the two-pass trick for constraint problems. You initialize DSU with correct size (watch for 1-indexed nodes).

### Tier 3 — Can adapt
You solve LC 721 (string-to-index mapping) and LC 305 (dynamic 2D grid) in under 30 minutes. You know when to use `size` vs `rank`. You can write the "sort within components" pattern.

### Tier 4 — Can combine
You solve LC 947 (row+column DSU with offset trick), LC 778 (sort edges + DSU), and LC 924 (component size tracking). You can explain WHY DSU beats BFS in dynamic settings. You implement Kruskal's MST from memory.

**Sign-off threshold:** Solve 8 out of 11 problems in the problem set, including at least 3 of the CORE problems and LC 721. Sign off only when Accounts Merge feels mechanical.

---

*Pattern 13 Complete — Union Find (DSU)*
*Next: Pattern 14 — Graphs: Shortest Path (Dijkstra, Bellman-Ford, 0-1 BFS)*
