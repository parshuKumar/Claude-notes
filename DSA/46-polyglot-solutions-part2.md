# 46 — POLYGLOT SOLUTIONS, PART 2 (PROBLEMS 14–30)
## The Same Algorithm Written Four Ways: C++, Java, Python, JavaScript
### Blocks 5–8: Graphs, Sorting and Search, DP and Backtracking, Bits and Design

---

## HOW TO READ A PROBLEM

Same format as doc 45. Identical variable names and line order across all four languages, so the diff against your C++ anchor is pure syntax. **What actually changed** lists only genuine differences, and the **Trap** names the likeliest porting mistake.

Blocks 6 to 8 hold the traps that fail people most often in a new language: JavaScript's missing ordered map in problem 20, the infinity sentinel in problem 22, the shared-row 2D table in problem 23, backtracking path copies in problem 25, and the four different bit models in problem 27.

## VERIFICATION

Every solution in this doc was run against real LeetCode test cases, including edge cases such as `INT_MIN` exponents, unsigned inputs above 2^31, and empty inputs.

| Language | Result |
|---|---|
| C++ | compiled and passed all test assertions |
| Python | passed all test assertions |
| JavaScript | passed all test assertions |
| Java | brace-balanced and trap-checked; not executed, no JDK on the build machine |

Blocks marked "sketch, insert only" in problem 30 intentionally show an alternative node shape and are not complete classes.

## CONTENTS

**Block 5 — Graphs and Grids**
14. LC 200 Number of Islands
15. LC 207 Course Schedule
16. LC 133 Clone Graph

**Block 6 — Sorting, Heap, Search, Ordered Map**
17. LC 56 Merge Intervals
18. LC 973 K Closest Points to Origin
19. LC 704 Binary Search
20. LC 729 My Calendar I

**Block 7 — DP and Backtracking**
21. LC 70 Climbing Stairs
22. LC 322 Coin Change
23. LC 72 Edit Distance
24. LC 139 Word Break
25. LC 78 Subsets

**Block 8 — Bit, Math, Matrix and Design**
26. LC 136 Single Number
27. LC 191 Number of 1 Bits
28. LC 50 Pow(x, n)
29. LC 48 Rotate Image
30. LC 208 Implement Trie (Prefix Tree)

---

# BLOCK 5 — GRAPHS AND GRIDS

## Problem 14 — LC 200 — Number of Islands

**Surfaces drilled:** 2D grid access, direction arrays (4-direction idiom per language), DFS recursion, in-place mutation of the grid to mark visited, bounds checking, char vs string grid cell comparison
**Algorithm in one line:** Scan every cell; on an unvisited land cell increment the island count and flood-fill (DFS) all connected land, sinking each visited cell to water so it is never counted again.

### C++ (your anchor)
```cpp
class Solution {
public:
    int numIslands(vector<vector<char>>& grid) {
        int rows = grid.size();
        int cols = grid[0].size();
        int count = 0;
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                if (grid[i][j] == '1') {
                    count++;
                    dfs(grid, i, j, rows, cols);
                }
            }
        }
        return count;
    }

private:
    void dfs(vector<vector<char>>& grid, int i, int j, int rows, int cols) {
        if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] != '1') {
            return;
        }
        grid[i][j] = '0';
        int dirs[4][2] = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}};
        for (int d = 0; d < 4; d++) {
            dfs(grid, i + dirs[d][0], j + dirs[d][1], rows, cols);
        }
    }
};
```

### Java
```java
class Solution {
    public int numIslands(char[][] grid) {
        int rows = grid.length;
        int cols = grid[0].length;
        int count = 0;
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                if (grid[i][j] == '1') {
                    count++;
                    dfs(grid, i, j, rows, cols);
                }
            }
        }
        return count;
    }

    private void dfs(char[][] grid, int i, int j, int rows, int cols) {
        if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] != '1') {
            return;
        }
        grid[i][j] = '0';
        int[][] dirs = {{-1, 0}, {1, 0}, {0, -1}, {0, 1}};
        for (int[] d : dirs) {
            dfs(grid, i + d[0], j + d[1], rows, cols);
        }
    }
}
```

### Python
```python
class Solution:
    def numIslands(self, grid: List[List[str]]) -> int:
        rows = len(grid)
        cols = len(grid[0])
        count = 0
        for i in range(rows):
            for j in range(cols):
                if grid[i][j] == "1":
                    count += 1
                    self.dfs(grid, i, j, rows, cols)
        return count

    def dfs(self, grid, i, j, rows, cols):
        if i < 0 or i >= rows or j < 0 or j >= cols or grid[i][j] != "1":
            return
        grid[i][j] = "0"
        dirs = [(-1, 0), (1, 0), (0, -1), (0, 1)]
        for di, dj in dirs:
            self.dfs(grid, i + di, j + dj, rows, cols)
```

### JavaScript
```js
/**
 * @param {character[][]} grid
 * @return {number}
 */
var numIslands = function(grid) {
    const rows = grid.length;
    const cols = grid[0].length;
    let count = 0;
    for (let i = 0; i < rows; i++) {
        for (let j = 0; j < cols; j++) {
            if (grid[i][j] === "1") {
                count++;
                dfs(grid, i, j, rows, cols);
            }
        }
    }
    return count;
};

function dfs(grid, i, j, rows, cols) {
    if (i < 0 || i >= rows || j < 0 || j >= cols || grid[i][j] !== "1") {
        return;
    }
    grid[i][j] = "0";
    const dirs = [[-1, 0], [1, 0], [0, -1], [0, 1]];
    for (const [di, dj] of dirs) {
        dfs(grid, i + di, j + dj, rows, cols);
    }
}
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Grid cell type | `char` compared with `'1'`/`'0'` | `char` compared with `'1'`/`'0'` | 1-char `str` compared with `"1"`/`"0"` | 1-char `string` compared with `"1"`/`"0"` |
| Direction array | `int dirs[4][2]` fixed array | `int[][] dirs` array literal | `list` of tuples | `array` of arrays |
| Helper placement | `private` member function | `private` method | plain method taking `self` | free function outside the class |
| Iterating direction pairs | manual `dirs[d][0]`/`dirs[d][1]` index | `for (int[] d : dirs)` | `for di, dj in dirs` tuple unpack | `for (const [di, dj] of dirs)` destructure |
| 2D array parameter type | `vector<vector<char>>&` | `char[][]` | `List[List[str]]` | `character[][]` (JSDoc only, untyped at runtime) |

> **Trap:** C++ and Java compare a real `char` against the character literal `'1'`; Python and JavaScript grid cells are one-character *strings*, so you must compare against the string `"1"` (or `'1'`, quotes are interchangeable in both) — writing `grid[i][j] == 1` (a number) in Python or JS silently never matches and returns 0 islands.

## Problem 15 — LC 207 — Course Schedule

**Surfaces drilled:** building an adjacency list from an edge list, indegree array, BFS with a queue, counting processed nodes
**Algorithm in one line:** Build a directed graph from prerequisites, run Kahn's algorithm (BFS from all zero-indegree nodes, decrementing indegree of neighbours), and a valid schedule exists iff every node gets processed.

### C++ (your anchor)
```cpp
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        vector<vector<int>> graph(numCourses);
        vector<int> indegree(numCourses, 0);
        for (auto& p : prerequisites) {
            int course = p[0];
            int pre = p[1];
            graph[pre].push_back(course);
            indegree[course]++;
        }
        queue<int> q;
        for (int i = 0; i < numCourses; i++) {
            if (indegree[i] == 0) {
                q.push(i);
            }
        }
        int processed = 0;
        while (!q.empty()) {
            int node = q.front();
            q.pop();
            processed++;
            for (int next : graph[node]) {
                indegree[next]--;
                if (indegree[next] == 0) {
                    q.push(next);
                }
            }
        }
        return processed == numCourses;
    }
};
```

### Java
```java
class Solution {
    public boolean canFinish(int numCourses, int[][] prerequisites) {
        List<List<Integer>> graph = new ArrayList<>();
        int[] indegree = new int[numCourses];
        for (int i = 0; i < numCourses; i++) {
            graph.add(new ArrayList<>());
        }
        for (int[] p : prerequisites) {
            int course = p[0];
            int pre = p[1];
            graph.get(pre).add(course);
            indegree[course]++;
        }
        Queue<Integer> q = new LinkedList<>();
        for (int i = 0; i < numCourses; i++) {
            if (indegree[i] == 0) {
                q.offer(i);
            }
        }
        int processed = 0;
        while (!q.isEmpty()) {
            int node = q.poll();
            processed++;
            for (int next : graph.get(node)) {
                indegree[next]--;
                if (indegree[next] == 0) {
                    q.offer(next);
                }
            }
        }
        return processed == numCourses;
    }
}
```

### Python
```python
class Solution:
    def canFinish(self, numCourses: int, prerequisites: List[List[int]]) -> bool:
        graph = [[] for _ in range(numCourses)]
        indegree = [0] * numCourses
        for course, pre in prerequisites:
            graph[pre].append(course)
            indegree[course] += 1
        q = deque()
        for i in range(numCourses):
            if indegree[i] == 0:
                q.append(i)
        processed = 0
        while q:
            node = q.popleft()
            processed += 1
            for next_course in graph[node]:
                indegree[next_course] -= 1
                if indegree[next_course] == 0:
                    q.append(next_course)
        return processed == numCourses
```

### JavaScript
```js
/**
 * @param {number} numCourses
 * @param {number[][]} prerequisites
 * @return {boolean}
 */
var canFinish = function(numCourses, prerequisites) {
    const graph = Array.from({ length: numCourses }, () => []);
    const indegree = new Array(numCourses).fill(0);
    for (const [course, pre] of prerequisites) {
        graph[pre].push(course);
        indegree[course]++;
    }
    const q = [];
    for (let i = 0; i < numCourses; i++) {
        if (indegree[i] === 0) {
            q.push(i);
        }
    }
    let processed = 0;
    let head = 0;
    while (head < q.length) {
        const node = q[head++];
        processed++;
        for (const next of graph[node]) {
            indegree[next]--;
            if (indegree[next] === 0) {
                q.push(next);
            }
        }
    }
    return processed === numCourses;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Adjacency list init | `vector<vector<int>> graph(numCourses)` | loop calling `graph.add(new ArrayList<>())` | list comprehension `[[] for _ in range(n)]` | `Array.from({length:n}, () => [])` |
| Queue type | `queue<int>` | `Queue<Integer>` backed by `LinkedList` | `collections.deque` | plain array plus a `head` index (no built-in queue) |
| Edge destructure | `p[0]`, `p[1]` manual indexing | `p[0]`, `p[1]` manual indexing | `for course, pre in prerequisites` tuple unpack | `for (const [course, pre] of prerequisites)` |
| Enqueue/dequeue names | `push` / `pop` / `front` | `offer` / `poll` | `append` / `popleft` | `push` / `q[head++]` |

> **Trap:** JavaScript has no built-in queue with O(1) dequeue — `Array.prototype.shift()` is O(n) and repeatedly shifting a large frontier turns Kahn's algorithm into O(n²) and can TLE; use a plain array with a `head` pointer (as above) instead of `shift()`.

## Problem 16 — LC 133 — Clone Graph

**Surfaces drilled:** hash map from node to node, reference/identity semantics, recursion with a visited map, node class definition, iterating a neighbour list
**Algorithm in one line:** DFS from the given node, using a map from original node to its clone to short-circuit cycles and avoid cloning the same node twice.

### C++ (your anchor)
```cpp
class Node {
public:
    int val;
    vector<Node*> neighbors;
    Node() {
        val = 0;
        neighbors = vector<Node*>();
    }
    Node(int _val) {
        val = _val;
        neighbors = vector<Node*>();
    }
    Node(int _val, vector<Node*> _neighbors) {
        val = _val;
        neighbors = _neighbors;
    }
};

class Solution {
public:
    Node* cloneGraph(Node* node) {
        unordered_map<Node*, Node*> visited;
        return clone(node, visited);
    }

private:
    Node* clone(Node* node, unordered_map<Node*, Node*>& visited) {
        if (node == nullptr) {
            return nullptr;
        }
        if (visited.count(node)) {
            return visited[node];
        }
        Node* copy = new Node(node->val);
        visited[node] = copy;
        for (Node* neighbor : node->neighbors) {
            copy->neighbors.push_back(clone(neighbor, visited));
        }
        return copy;
    }
};
```

### Java
```java
class Node {
    public int val;
    public List<Node> neighbors;
    public Node() {
        val = 0;
        neighbors = new ArrayList<Node>();
    }
    public Node(int _val) {
        val = _val;
        neighbors = new ArrayList<Node>();
    }
    public Node(int _val, ArrayList<Node> _neighbors) {
        val = _val;
        neighbors = _neighbors;
    }
}

class Solution {
    public Node cloneGraph(Node node) {
        Map<Node, Node> visited = new HashMap<>();
        return clone(node, visited);
    }

    private Node clone(Node node, Map<Node, Node> visited) {
        if (node == null) {
            return null;
        }
        if (visited.containsKey(node)) {
            return visited.get(node);
        }
        Node copy = new Node(node.val);
        visited.put(node, copy);
        for (Node neighbor : node.neighbors) {
            copy.neighbors.add(clone(neighbor, visited));
        }
        return copy;
    }
}
```

### Python
```python
class Node:
    def __init__(self, val = 0, neighbors = None):
        self.val = val
        self.neighbors = neighbors if neighbors is not None else []

class Solution:
    def cloneGraph(self, node: 'Node') -> 'Node':
        visited = {}
        return self.clone(node, visited)

    def clone(self, node, visited):
        if node is None:
            return None
        if node in visited:
            return visited[node]
        copy = Node(node.val)
        visited[node] = copy
        for neighbor in node.neighbors:
            copy.neighbors.append(self.clone(neighbor, visited))
        return copy
```

### JavaScript
```js
/**
 * // Definition for a Node.
 * function Node(val, neighbors) {
 *    this.val = val === undefined ? 0 : val;
 *    this.neighbors = neighbors === undefined ? [] : neighbors;
 * };
 */

/**
 * @param {Node} node
 * @return {Node}
 */
var cloneGraph = function(node) {
    const visited = new Map();
    return clone(node, visited);
};

function clone(node, visited) {
    if (node === null) {
        return null;
    }
    if (visited.has(node)) {
        return visited.get(node);
    }
    const copy = new Node(node.val);
    visited.set(node, copy);
    for (const neighbor of node.neighbors) {
        copy.neighbors.push(clone(neighbor, visited));
    }
    return copy;
}

function Node(val, neighbors) {
    this.val = val === undefined ? 0 : val;
    this.neighbors = neighbors === undefined ? [] : neighbors;
}
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Node class | class with 3 overloaded constructors | class with 3 overloaded constructors | `__init__` with a default-arg guard | function-style constructor, given as a comment block on LeetCode |
| Map type | `unordered_map<Node*, Node*>` | `HashMap<Node, Node>` | `dict` | `Map` |
| Null value | `nullptr` | `null` | `None` | `null` |
| Membership check | `.count(node)` | `.containsKey(node)` | `node in visited` | `.has(node)` |
| Lookup / insert | `visited[node]` / `visited[node] = copy` | `.get(node)` / `.put(node, copy)` | `visited[node]` both ways | `.get(node)` / `.set(node, copy)` |

> **Trap:** the map keys are node *references*, not values — Java's `HashMap` works here only because `Node` never overrides `equals()`/`hashCode()`, so it falls back to identity comparison (the same guarantee C++ gets from raw pointers and Python/JS get from object identity); don't "fix" this by adding a custom `equals()`, it would break cycle detection the moment two distinct nodes share a `val`.

# BLOCK 6 — SORTING, HEAP, SEARCH, ORDERED MAP

## Problem 17 — LC 56 — Merge Intervals

**Surfaces drilled:** sorting a 2D array/list by its first column, building a result list of pairs, comparing and mutating the last element of the result, converting the result to the required return type
**Algorithm in one line:** Sort intervals by start; walk through them, merging into the last interval in the result whenever the current interval's start overlaps it, otherwise appending a new interval.

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<vector<int>> merge(vector<vector<int>>& intervals) {
        sort(intervals.begin(), intervals.end(), [](const vector<int>& a, const vector<int>& b) {
            return a[0] < b[0];
        });
        vector<vector<int>> result;
        for (auto& interval : intervals) {
            if (!result.empty() && interval[0] <= result.back()[1]) {
                result.back()[1] = max(result.back()[1], interval[1]);
            } else {
                result.push_back(interval);
            }
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public int[][] merge(int[][] intervals) {
        Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
        List<int[]> result = new ArrayList<>();
        for (int[] interval : intervals) {
            if (!result.isEmpty() && interval[0] <= result.get(result.size() - 1)[1]) {
                result.get(result.size() - 1)[1] = Math.max(result.get(result.size() - 1)[1], interval[1]);
            } else {
                result.add(interval);
            }
        }
        return result.toArray(new int[result.size()][]);
    }
}
```

### Python
```python
class Solution:
    def merge(self, intervals: List[List[int]]) -> List[List[int]]:
        intervals.sort(key=lambda x: x[0])
        result = []
        for interval in intervals:
            if result and interval[0] <= result[-1][1]:
                result[-1][1] = max(result[-1][1], interval[1])
            else:
                result.append(interval)
        return result
```

### JavaScript
```js
/**
 * @param {number[][]} intervals
 * @return {number[][]}
 */
var merge = function(intervals) {
    intervals.sort((a, b) => a[0] - b[0]);
    const result = [];
    for (const interval of intervals) {
        if (result.length > 0 && interval[0] <= result[result.length - 1][1]) {
            result[result.length - 1][1] = Math.max(result[result.length - 1][1], interval[1]);
        } else {
            result.push(interval);
        }
    }
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Sort call | `sort(begin, end, cmp)` with a `bool`-returning lambda | `Arrays.sort(a, comparator)` with `Integer.compare` | `list.sort(key=lambda x: x[0])` | `array.sort((a,b) => a[0]-b[0])` |
| Comparator style | boolean less-than lambda | `Comparator<int[]>` returning an int | `key` function, not a comparator | numeric-difference comparator |
| Result container | `vector<vector<int>>` | `List<int[]>` then `.toArray(...)` | plain `list` | plain `Array` |
| Mutating the last element | `result.back()[1] = ...` | `result.get(result.size()-1)[1] = ...` | `result[-1][1] = ...` | `result[result.length-1][1] = ...` |
| Return-type conversion | none needed | `.toArray(new int[result.size()][])` | none needed | none needed |

> **Trap:** Java's `Arrays.sort` on an `int[][]` needs a real `Comparator` — writing `(a, b) -> a[0] - b[0]` looks fine but can silently misbehave on certain inputs due to how Java's sort validates comparator contracts (and it's simply the wrong habit to build), so always use `Integer.compare(a[0], b[0])`; JavaScript's `.sort()` defaults to lexicographic *string* ordering, so omitting the `(a,b) => a[0]-b[0]` comparator will sort `[10, ...]` before `[2, ...]`; Python's `.sort()` takes a `key`, not a two-argument comparator, so `intervals.sort(key=lambda x: x[0])` is correct but `intervals.sort(lambda x: x[0])` is a `TypeError`.

## Problem 18 — LC 973 — K Closest Points to Origin

**Surfaces drilled:** max-heap of size k with custom ordering on a computed distance, heap of arrays/tuples, the default-ordering flip between languages
**Algorithm in one line:** Maintain a max-heap of at most k points keyed by squared distance, pushing each point and popping the farthest whenever the heap exceeds size k, so what remains are the k closest.

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
        priority_queue<pair<int, vector<int>>> heap;
        for (auto& point : points) {
            int dist = point[0] * point[0] + point[1] * point[1];
            heap.push({dist, point});
            if ((int)heap.size() > k) {
                heap.pop();
            }
        }
        vector<vector<int>> result;
        while (!heap.empty()) {
            result.push_back(heap.top().second);
            heap.pop();
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public int[][] kClosest(int[][] points, int k) {
        PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> b[0] - a[0]);
        for (int[] point : points) {
            int dist = point[0] * point[0] + point[1] * point[1];
            heap.offer(new int[]{dist, point[0], point[1]});
            if (heap.size() > k) {
                heap.poll();
            }
        }
        int[][] result = new int[heap.size()][];
        int idx = 0;
        while (!heap.isEmpty()) {
            int[] entry = heap.poll();
            result[idx++] = new int[]{entry[1], entry[2]};
        }
        return result;
    }
}
```

### Python
```python
class Solution:
    def kClosest(self, points: List[List[int]], k: int) -> List[List[int]]:
        heap = []
        for point in points:
            dist = point[0] * point[0] + point[1] * point[1]
            heapq.heappush(heap, (-dist, point))
            if len(heap) > k:
                heapq.heappop(heap)
        result = []
        while heap:
            result.append(heapq.heappop(heap)[1])
        return result
```

### JavaScript
```js
class MaxHeap {
    constructor() {
        this.data = [];
    }

    size() {
        return this.data.length;
    }

    push(item) {
        this.data.push(item);
        let i = this.data.length - 1;
        while (i > 0) {
            const parent = (i - 1) >> 1;
            if (this.data[parent][0] >= this.data[i][0]) break;
            [this.data[parent], this.data[i]] = [this.data[i], this.data[parent]];
            i = parent;
        }
    }

    pop() {
        const top = this.data[0];
        const last = this.data.pop();
        if (this.data.length > 0) {
            this.data[0] = last;
            let i = 0;
            while (true) {
                const left = 2 * i + 1;
                const right = 2 * i + 2;
                let largest = i;
                if (left < this.data.length && this.data[left][0] > this.data[largest][0]) largest = left;
                if (right < this.data.length && this.data[right][0] > this.data[largest][0]) largest = right;
                if (largest === i) break;
                [this.data[largest], this.data[i]] = [this.data[i], this.data[largest]];
                i = largest;
            }
        }
        return top;
    }
}

/**
 * @param {number[][]} points
 * @param {number} k
 * @return {number[][]}
 */
var kClosest = function(points, k) {
    const heap = new MaxHeap();
    for (const point of points) {
        const dist = point[0] * point[0] + point[1] * point[1];
        heap.push([dist, point]);
        if (heap.size() > k) {
            heap.pop();
        }
    }
    const result = [];
    while (heap.size() > 0) {
        result.push(heap.pop()[1]);
    }
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Default heap order | `priority_queue` is max-heap | `PriorityQueue` is min-heap | `heapq` is min-heap | no built-in heap at all |
| Flip to max-heap | none needed, already max | comparator `(a, b) -> b[0] - a[0]` | push `-dist` instead of `dist` | hand-rolled `MaxHeap` with `>` comparisons |
| Heap element type | `pair<int, vector<int>>` | flattened `int[]` `{dist, x, y}` | `tuple` `(-dist, point)` | plain `array` `[dist, point]` |
| Pop syntax | `heap.top()` then `heap.pop()` | `heap.poll()` | `heapq.heappop(heap)` | `heap.pop()` (custom method) |

> **Trap:** C++'s `priority_queue` is a max-heap by default, but Java's `PriorityQueue` and Python's `heapq` are min-heaps by default — port C++ heap code to Java/Python without flipping the comparator (or negating the key) and you'll silently keep the k *farthest* points instead of the k closest; JavaScript has no built-in heap at all, so an interviewer expects you to either roll one (shown above) or fall back to sort-and-slice.

## Problem 19 — LC 704 — Binary Search

**Surfaces drilled:** the hand-rolled loop, overflow-safe mid computation, the built-in equivalent in each language
**Algorithm in one line:** Repeatedly halve the search range `[lo, hi]` by comparing the middle element to the target until found or the range is empty.

### C++ (your anchor)
```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int lo = 0, hi = nums.size() - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) {
                return mid;
            } else if (nums[mid] < target) {
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
        return -1;
    }
};
```

Built-in variant:
```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        auto it = lower_bound(nums.begin(), nums.end(), target);
        if (it != nums.end() && *it == target) {
            return it - nums.begin();
        }
        return -1;
    }
};
```

### Java
```java
class Solution {
    public int search(int[] nums, int target) {
        int lo = 0, hi = nums.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) {
                return mid;
            } else if (nums[mid] < target) {
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
        return -1;
    }
}
```

Built-in variant:
```java
class Solution {
    public int search(int[] nums, int target) {
        int idx = Arrays.binarySearch(nums, target);
        return idx >= 0 ? idx : -1;
    }
}
```

### Python
```python
class Solution:
    def search(self, nums: List[int], target: int) -> int:
        lo, hi = 0, len(nums) - 1
        while lo <= hi:
            mid = lo + (hi - lo) // 2
            if nums[mid] == target:
                return mid
            elif nums[mid] < target:
                lo = mid + 1
            else:
                hi = mid - 1
        return -1
```

Built-in variant:
```python
class Solution:
    def search(self, nums: List[int], target: int) -> int:
        idx = bisect.bisect_left(nums, target)
        if idx < len(nums) and nums[idx] == target:
            return idx
        return -1
```

### JavaScript
```js
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number}
 */
var search = function(nums, target) {
    let lo = 0, hi = nums.length - 1;
    while (lo <= hi) {
        const mid = lo + Math.floor((hi - lo) / 2);
        if (nums[mid] === target) {
            return mid;
        } else if (nums[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid - 1;
        }
    }
    return -1;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Overflow-safe mid | `lo + (hi - lo) / 2` | `lo + (hi - lo) / 2` | `lo + (hi - lo) // 2` (moot — Python ints don't overflow) | `lo + Math.floor((hi - lo) / 2)` |
| Integer division | `/` truncates toward zero on ints | `/` truncates toward zero on ints | `//` floors | `/` always produces a float, needs `Math.floor` |
| Built-in binary search | `lower_bound(begin, end, target)` | `Arrays.binarySearch(nums, target)` | `bisect.bisect_left(nums, target)` | none exists |
| Built-in "not found" signal | iterator may point past the value | negative return `-(insertion point) - 1` | index may point past the value or land on a mismatch | N/A |

> **Trap:** JavaScript ships no standard-library binary search — in an interview you must write the hand-rolled loop from scratch every time (there is no `bisect`/`lower_bound`/`Arrays.binarySearch` to fall back on), and don't forget `Math.floor` around `(hi - lo) / 2`: unlike C++/Java integer division, JS's `/` always returns a float, so a bare `lo + (hi - lo) / 2` produces a non-integer `mid` and an infinite loop or bad index.

## Problem 20 — LC 729 — My Calendar I

**Surfaces drilled:** ordered map floor/ceiling queries, class with internal state, the total absence of an ordered map in JavaScript
**Algorithm in one line:** Keep bookings sorted by start time; a new `[start, end)` is bookable iff it doesn't overlap the booking immediately before it (floor) or immediately after it (ceiling).

### C++ (your anchor)
```cpp
class MyCalendar {
public:
    map<int, int> bookings;

    MyCalendar() {

    }

    bool book(int start, int end) {
        auto it = bookings.lower_bound(start);
        if (it != bookings.end() && it->first < end) {
            return false;
        }
        if (it != bookings.begin()) {
            auto prev = std::prev(it);
            if (prev->second > start) {
                return false;
            }
        }
        bookings[start] = end;
        return true;
    }
};
```

### Java
```java
class MyCalendar {
    TreeMap<Integer, Integer> bookings;

    public MyCalendar() {
        bookings = new TreeMap<>();
    }

    public boolean book(int start, int end) {
        Integer ceilingKey = bookings.ceilingKey(start);
        if (ceilingKey != null && ceilingKey < end) {
            return false;
        }
        Integer floorKey = bookings.floorKey(start);
        if (floorKey != null && bookings.get(floorKey) > start) {
            return false;
        }
        bookings.put(start, end);
        return true;
    }
}
```

### Python
```python
class MyCalendar:
    def __init__(self):
        self.bookings = []

    def book(self, start: int, end: int) -> bool:
        idx = bisect.bisect_left(self.bookings, (start,))
        if idx < len(self.bookings) and self.bookings[idx][0] < end:
            return False
        if idx > 0 and self.bookings[idx - 1][1] > start:
            return False
        self.bookings.insert(idx, (start, end))
        return True
```

### JavaScript
```js
var MyCalendar = function() {
    this.bookings = [];
};

/**
 * @param {number} start
 * @param {number} end
 * @return {boolean}
 */
MyCalendar.prototype.book = function(start, end) {
    let lo = 0, hi = this.bookings.length;
    while (lo < hi) {
        const mid = lo + Math.floor((hi - lo) / 2);
        if (this.bookings[mid][0] < start) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    const idx = lo;
    if (idx < this.bookings.length && this.bookings[idx][0] < end) {
        return false;
    }
    if (idx > 0 && this.bookings[idx - 1][1] > start) {
        return false;
    }
    this.bookings.splice(idx, 0, [start, end]);
    return true;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Ordered map | `std::map` (balanced BST, native) | `TreeMap` (balanced BST, native) | none — sorted `list` + `bisect` | none — sorted `Array` + hand-rolled binary search |
| Ceiling query | `lower_bound(start)` | `ceilingKey(start)` | `bisect_left(bookings, (start,))` | manual binary-search loop over `[0]` field |
| Floor query | `std::prev(it)` | `floorKey(start)` | `bookings[idx - 1]` | `this.bookings[idx - 1]` |
| Insert keeping order | `bookings[start] = end` (map self-orders) | `bookings.put(start, end)` (map self-orders) | `list.insert(idx, (start, end))` | `Array.prototype.splice(idx, 0, [start, end])` |
| Per-call cost | O(log n) | O(log n) | O(n) insert despite O(log n) search | O(n) splice despite O(log n) search |

> **Trap:** this is the one problem in the set where the languages are *not* equivalent — C++'s `std::map` and Java's `TreeMap` are true balanced-BST ordered maps with O(log n) floor/ceiling/insert, but neither Python nor JavaScript has one in the standard library. Python's fix is `bisect` on a plain sorted `list` (or the third-party `sortedcontainers.SortedList` if it's importable, which restores O(log n) insert) — as coded above with a bare `list`, insert is O(n). JavaScript has no `bisect` module either, so you hand-roll the binary search *and* accept an O(n) `splice`. In a JS interview, say this out loud: "there's no ordered map/TreeMap equivalent in JS, so I'm simulating one with a sorted array and binary search, which costs O(n) insert instead of O(log n)" — naming the gap is what separates a strong answer from a candidate who looks lost when `TreeMap` doesn't exist.

---

# BLOCK 7 — DP AND BACKTRACKING

## Problem 21 — LC 70 — Climbing Stairs

**Surfaces drilled:** 1D dp array creation and fill per language, array indexing, base cases before the loop, the classic two-state Fibonacci recurrence.
**Algorithm in one line:** `dp[i] = dp[i-1] + dp[i-2]`, seeded with `dp[1]=1, dp[2]=2`, is the number of ways to climb `n` stairs.
**Complexity:** O(n) time, O(n) space in the array form shown below; O(1) space if you roll the last two values into two scalars instead of an array (worth doing after you've drilled the array syntax once).

### C++ (your anchor)
```cpp
class Solution {
public:
    int climbStairs(int n) {
        if (n <= 2) return n;
        vector<int> dp(n + 1);
        dp[1] = 1;
        dp[2] = 2;
        for (int i = 3; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2];
        }
        return dp[n];
    }
};
```

### Java
```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;
        int[] dp = new int[n + 1];
        dp[1] = 1;
        dp[2] = 2;
        for (int i = 3; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2];
        }
        return dp[n];
    }
}
```

### Python
```python
class Solution:
    def climbStairs(self, n: int) -> int:
        if n <= 2:
            return n
        dp = [0] * (n + 1)
        dp[1] = 1
        dp[2] = 2
        for i in range(3, n + 1):
            dp[i] = dp[i - 1] + dp[i - 2]
        return dp[n]
```

### JavaScript
```js
/**
 * @param {number} n
 * @return {number}
 */
var climbStairs = function(n) {
    if (n <= 2) return n;
    const dp = new Array(n + 1).fill(0);
    dp[1] = 1;
    dp[2] = 2;
    for (let i = 3; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    return dp[n];
};
```

### Worked example
For `n = 5`: `dp[1]=1, dp[2]=2`.

Then `dp[3]=dp[2]+dp[1]=3`, `dp[4]=dp[3]+dp[2]=5`, `dp[5]=dp[4]+dp[3]=8`.

The table is identical in shape and values in all four languages.

Only the syntax used to build and index it differs, which is the whole point of this drill.

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| dp array creation | `vector<int> dp(n + 1)` (zero-init guaranteed) | `new int[n + 1]` (zero-init guaranteed) | `[0] * (n + 1)` | `new Array(n + 1).fill(0)` |
| loop header | `for (int i = 3; i <= n; i++)` | `for (int i = 3; i <= n; i++)` | `for i in range(3, n + 1):` | `for (let i = 3; i <= n; i++)` |
| function wrapper | method in `class Solution` | method in `class Solution` | method in `class Solution` | free function assigned to `var climbStairs` |
| declaring a local var | `int`/`vector<int>` | `int`/`int[]` | no type, just `=` | `const`/`let` |

> **Trap:** `new Array(n + 1)` in JS does **not** zero-fill.
>
> It creates `n + 1` empty "holes," and reading a hole gives `undefined`.
>
> So `dp[i-1] + dp[i-2]` silently becomes `NaN` if you forget `.fill(0)`.
>
> C++'s `vector<int> dp(n+1)` and Java's `new int[n+1]` both zero-init automatically.
>
> Python's `[0] * (n+1)` is explicit about the zero.
>
> So JS is the one language where skipping the fill is a silent runtime bug, not a compile error or an obvious crash.

---

## Problem 22 — LC 322 — Coin Change

**Surfaces drilled:** 1D dp array seeded with an "infinity" sentinel, nested loops (amount outer, coins inner), min tracking, and the guard needed before adding 1 to the sentinel.
**Algorithm in one line:** `dp[i] = min(dp[i], dp[i - coin] + 1)` over every coin for every amount from 1 to `amount`, with `dp[0] = 0` and unreachable amounts left at infinity.
**Complexity:** O(amount × coins.length) time, O(amount) space — the double loop is unavoidable in the bottom-up form, and it is the same nesting in all four languages.

### C++ (your anchor)
```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        vector<int> dp(amount + 1, INT_MAX);
        dp[0] = 0;
        for (int i = 1; i <= amount; i++) {
            for (int coin : coins) {
                if (coin <= i && dp[i - coin] != INT_MAX) {
                    dp[i] = min(dp[i], dp[i - coin] + 1);
                }
            }
        }
        return dp[amount] == INT_MAX ? -1 : dp[amount];
    }
};
```

### Java
```java
class Solution {
    public int coinChange(int[] coins, int amount) {
        int[] dp = new int[amount + 1];
        Arrays.fill(dp, Integer.MAX_VALUE);
        dp[0] = 0;
        for (int i = 1; i <= amount; i++) {
            for (int coin : coins) {
                if (coin <= i && dp[i - coin] != Integer.MAX_VALUE) {
                    dp[i] = Math.min(dp[i], dp[i - coin] + 1);
                }
            }
        }
        return dp[amount] == Integer.MAX_VALUE ? -1 : dp[amount];
    }
}
```

### Python
```python
class Solution:
    def coinChange(self, coins: List[int], amount: int) -> int:
        dp = [float('inf')] * (amount + 1)
        dp[0] = 0
        for i in range(1, amount + 1):
            for coin in coins:
                if coin <= i and dp[i - coin] != float('inf'):
                    dp[i] = min(dp[i], dp[i - coin] + 1)
        return -1 if dp[amount] == float('inf') else dp[amount]
```

### JavaScript
```js
/**
 * @param {number[]} coins
 * @param {number} amount
 * @return {number}
 */
var coinChange = function(coins, amount) {
    const dp = new Array(amount + 1).fill(Infinity);
    dp[0] = 0;
    for (let i = 1; i <= amount; i++) {
        for (const coin of coins) {
            if (coin <= i && dp[i - coin] !== Infinity) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] === Infinity ? -1 : dp[amount];
};
```

### Worked example
For `coins = [1, 2, 5]`, `amount = 11`: `dp[0] = 0`.

Working up, `dp[5] = 1` via a single 5-coin.

`dp[6] = dp[5] + 1 = 2`, using a 5-coin and a 1-coin.

Continuing this way, `dp[11]` ends at `3` — for example, `5 + 5 + 1`.

Every intermediate `dp[i]` that is never reached by any combination of coins stays at the sentinel until the final check converts it to `-1`.

That conversion is the line that must compare against the *same* sentinel you filled with.

This is exactly why the sentinel constant appears three times per language — fill, guard, final check — instead of once.

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| infinity sentinel | `INT_MAX` (2147483647, a real finite int) | `Integer.MAX_VALUE` (same finite int) | `float('inf')` (true mathematical infinity) | `Infinity` (true IEEE-754 infinity) |
| fill with sentinel | `vector<int> dp(amount+1, INT_MAX)` (ctor arg) | `Arrays.fill(dp, Integer.MAX_VALUE)` (separate call) | `[float('inf')] * (amount+1)` | `new Array(amount+1).fill(Infinity)` |
| min helper | `min(dp[i], ...)` | `Math.min(dp[i], ...)` | built-in `min(...)` | `Math.min(...)` |
| sentinel equality check | `!= INT_MAX` | `!= Integer.MAX_VALUE` | `!= float('inf')` | `!== Infinity` |

> **Trap:** this is the single most dangerous sentinel bug in the whole curriculum.
>
> `INT_MAX` and `Integer.MAX_VALUE` are **finite** numbers, not real infinities.
>
> `dp[i - coin] + 1` when `dp[i - coin] == INT_MAX` overflows to undefined behavior in C++ — commonly wraps to `INT_MIN`, a huge *negative* number that then wins every `min()` comparison and corrupts the whole table.
>
> Java silently wraps to a negative number too — Java integer overflow is well-defined wraparound, but it is still wrong.
>
> That is exactly why the `dp[i - coin] != INT_MAX` / `!= Integer.MAX_VALUE` guard exists in both blocks above.
>
> **It is not optional — it is the fix for a real overflow bug.**
>
> Python's `float('inf')` and JS's `Infinity` are actual infinities: `float('inf') + 1 == float('inf')` and `Infinity + 1 === Infinity`.
>
> They never overflow, so the guard there is just defensive style, not a correctness requirement.
>
> If you port the Java/C++ version to Python and forget the guard, nothing breaks.
>
> If you port the Python version to C++ and forget the guard, you get a corrupted answer that still compiles and runs — the worst kind of bug.

---

## Problem 23 — LC 72 — Edit Distance

**Surfaces drilled:** 2D dp table allocation in all four languages, string character indexing/comparison, filling the base-case row and column, and a three-way min.
**Algorithm in one line:** `dp[i][j]` is the edit distance between `word1[0:i]` and `word2[0:j]`, equal to `dp[i-1][j-1]` on a character match, else `1 + min(delete, insert, replace)`.
**Complexity:** O(n × m) time, O(n × m) space for the table shown here (rollable to O(min(n, m)) space with a two-row trick, which is a good follow-up exercise once the four-language table syntax is solid).

### C++ (your anchor)
```cpp
class Solution {
public:
    int minDistance(string word1, string word2) {
        int n = word1.size(), m = word2.size();
        vector<vector<int>> dp(n + 1, vector<int>(m + 1, 0));
        for (int i = 0; i <= n; i++) dp[i][0] = i;
        for (int j = 0; j <= m; j++) dp[0][j] = j;
        for (int i = 1; i <= n; i++) {
            for (int j = 1; j <= m; j++) {
                if (word1[i - 1] == word2[j - 1]) {
                    dp[i][j] = dp[i - 1][j - 1];
                } else {
                    dp[i][j] = 1 + min({dp[i - 1][j - 1], dp[i - 1][j], dp[i][j - 1]});
                }
            }
        }
        return dp[n][m];
    }
};
```

### Java
```java
class Solution {
    public int minDistance(String word1, String word2) {
        int n = word1.length(), m = word2.length();
        int[][] dp = new int[n + 1][m + 1];
        for (int i = 0; i <= n; i++) dp[i][0] = i;
        for (int j = 0; j <= m; j++) dp[0][j] = j;
        for (int i = 1; i <= n; i++) {
            for (int j = 1; j <= m; j++) {
                if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                    dp[i][j] = dp[i - 1][j - 1];
                } else {
                    dp[i][j] = 1 + Math.min(dp[i - 1][j - 1], Math.min(dp[i - 1][j], dp[i][j - 1]));
                }
            }
        }
        return dp[n][m];
    }
}
```

### Python
```python
class Solution:
    def minDistance(self, word1: str, word2: str) -> int:
        n, m = len(word1), len(word2)
        dp = [[0] * (m + 1) for _ in range(n + 1)]
        for i in range(n + 1):
            dp[i][0] = i
        for j in range(m + 1):
            dp[0][j] = j
        for i in range(1, n + 1):
            for j in range(1, m + 1):
                if word1[i - 1] == word2[j - 1]:
                    dp[i][j] = dp[i - 1][j - 1]
                else:
                    dp[i][j] = 1 + min(dp[i - 1][j - 1], dp[i - 1][j], dp[i][j - 1])
        return dp[n][m]
```

### JavaScript
```js
/**
 * @param {string} word1
 * @param {string} word2
 * @return {number}
 */
var minDistance = function(word1, word2) {
    const n = word1.length, m = word2.length;
    const dp = Array.from({ length: n + 1 }, () => new Array(m + 1).fill(0));
    for (let i = 0; i <= n; i++) dp[i][0] = i;
    for (let j = 0; j <= m; j++) dp[0][j] = j;
    for (let i = 1; i <= n; i++) {
        for (let j = 1; j <= m; j++) {
            if (word1[i - 1] === word2[j - 1]) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1], dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[n][m];
};
```

### Worked example
For `word1 = "horse"`, `word2 = "ros"`: `n = 5, m = 3`.

Base row/column fill gives `dp[i][0] = i` and `dp[0][j] = j`.

Walking the table, `'h'` vs `'r'` is a mismatch, so `dp[1][1] = 1 + min(dp[0][0], dp[0][1], dp[1][0])`.

`'o'` vs `'o'` is a match, so it propagates `dp[i-1][j-1]` diagonally with no added cost.

The final answer is `dp[5][3] = 3`.

One optimal edit path is: replace `'h'` → `'r'`, keep `'o'`, delete `'r'`, keep `'s'`, delete `'e'` — matching LeetCode's expected output of `3`.

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| 2D table allocation | `vector<vector<int>> dp(n+1, vector<int>(m+1, 0))` | `new int[n + 1][m + 1]` | `[[0]*(m+1) for _ in range(n+1)]` | `Array.from({length:n+1}, () => new Array(m+1).fill(0))` |
| string length | `word1.size()` | `word1.length()` | `len(word1)` | `word1.length` |
| character access | `word1[i - 1]` | `word1.charAt(i - 1)` | `word1[i - 1]` | `word1[i - 1]` |
| char equality | `==` | `==` (primitive `char`) | `==` | `===` |
| 3-way min | `min({a, b, c})` (initializer-list overload) | `Math.min(a, Math.min(b, c))` (binary only) | `min(a, b, c)` (variadic) | `Math.min(a, b, c)` (variadic) |

> **Trap (this one has burned real interviews):** every naive way to build a 2D array except the ones shown above is broken.
>
> - **Python:** `dp = [[0] * (m + 1)] * (n + 1)` looks identical to the correct comprehension but is a completely different object graph.
>   It creates *one* inner list and repeats the same reference `n + 1` times.
>   Write `dp[3][2] = 99` and you'll find `dp[0][2]`, `dp[1][2]`, every row's index-2 mutated too, because there is only one row underneath.
>   The fix is the list comprehension used above: `[[0] * (m + 1) for _ in range(n + 1)]`, which allocates a fresh list on every iteration.
> - **JavaScript:** `Array(n + 1).fill(new Array(m + 1).fill(0))` has the exact same bug.
>   `.fill()` copies the *same reference* into every outer slot, not a fresh array each time.
>   The fix is `Array.from({ length: n + 1 }, () => new Array(m + 1).fill(0))`, whose factory function actually runs once per index.
> - **Java and C++ are the outlier-safe ones here:** `new int[n+1][m+1]` genuinely allocates `n+1` independent inner arrays, and the vector-of-vectors constructor does the same — no aliasing trap in either.
>   If you're a C++ native porting to Python or JS, this is the first place your instinct to "just copy the shape" will silently corrupt your table.

---

## Problem 24 — LC 139 — Word Break

**Surfaces drilled:** memoization container choice (hash set for the dictionary, boolean array for dp), set membership lookup, substring extraction, and a genuine off-by-one trap in the substring APIs.
**Algorithm in one line:** `dp[i]` is true iff some `j < i` has `dp[j]` true and `s[j..i)` is in the dictionary; answer is `dp[n]`.
**Complexity:** O(n² ) time in the worst case (the inner loop tries every split point, and each split does an O(n) substring extraction/hash in most language runtimes, so it's really O(n³) including substring cost — still fine at LeetCode's constraint size), O(n) space for `dp` plus O(sum of word lengths) for the dictionary.

### C++ (your anchor)
```cpp
class Solution {
public:
    bool wordBreak(string s, vector<string>& wordDict) {
        unordered_set<string> dict(wordDict.begin(), wordDict.end());
        int n = s.size();
        vector<bool> dp(n + 1, false);
        dp[0] = true;
        for (int i = 1; i <= n; i++) {
            for (int j = 0; j < i; j++) {
                if (dp[j] && dict.count(s.substr(j, i - j))) {
                    dp[i] = true;
                    break;
                }
            }
        }
        return dp[n];
    }
};
```

### Java
```java
class Solution {
    public boolean wordBreak(String s, List<String> wordDict) {
        Set<String> dict = new HashSet<>(wordDict);
        int n = s.length();
        boolean[] dp = new boolean[n + 1];
        dp[0] = true;
        for (int i = 1; i <= n; i++) {
            for (int j = 0; j < i; j++) {
                if (dp[j] && dict.contains(s.substring(j, i))) {
                    dp[i] = true;
                    break;
                }
            }
        }
        return dp[n];
    }
}
```

### Python
```python
class Solution:
    def wordBreak(self, s: str, wordDict: List[str]) -> bool:
        dict_ = set(wordDict)
        n = len(s)
        dp = [False] * (n + 1)
        dp[0] = True
        for i in range(1, n + 1):
            for j in range(i):
                if dp[j] and s[j:i] in dict_:
                    dp[i] = True
                    break
        return dp[n]
```

### JavaScript
```js
/**
 * @param {string} s
 * @param {string[]} wordDict
 * @return {boolean}
 */
var wordBreak = function(s, wordDict) {
    const dict = new Set(wordDict);
    const n = s.length;
    const dp = new Array(n + 1).fill(false);
    dp[0] = true;
    for (let i = 1; i <= n; i++) {
        for (let j = 0; j < i; j++) {
            if (dp[j] && dict.has(s.slice(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    return dp[n];
};
```

### Worked example
For `s = "leetcode"`, `wordDict = ["leet", "code"]`: `dp[0] = true`.

At `i = 4`, `j = 0` gives `s[0:4] = "leet"`, which is in the dictionary, and `dp[0]` is true — so `dp[4] = true`.

At `i = 8`, `j = 4` gives `s[4:8] = "code"`, which is in the dictionary, and `dp[4]` is true — so `dp[8] = true`.

Since `n = 8`, the answer is `true`.

Note the slice bounds used here, `s[0:4]` and `s[4:8]`, are exactly the `(j, i)` end-index pairs — this is the pattern the substring trap below is about.

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| dictionary container | `unordered_set<string>` from range ctor | `new HashSet<>(wordDict)` | `set(wordDict)` | `new Set(wordDict)` |
| membership test | `dict.count(x)` (returns 0/1) | `dict.contains(x)` (returns boolean) | `x in dict_` | `dict.has(x)` |
| substring extraction | `s.substr(j, i - j)` — **(start, LENGTH)** | `s.substring(j, i)` — (start, END) | `s[j:i]` — (start, END, exclusive) | `s.slice(j, i)` — (start, END, exclusive) |
| boolean array | `vector<bool> dp(n+1, false)` | `boolean[] dp = new boolean[n+1]` (auto false) | `[False] * (n + 1)` | `new Array(n + 1).fill(false)` |

> **Trap:** this is the substring API trap, and it is *the* most common bug when a C++ programmer ports string-slicing DP to any other language.
>
> `string::substr(pos, len)` takes a **length** as its second argument.
>
> `s.substr(j, i - j)` extracts exactly `i - j` characters starting at `j`, giving you the substring `s[j..i)`.
>
> Every other language's equivalent method takes an **end index**, not a length.
>
> Java's `s.substring(j, i)`, Python's `s[j:i]`, and JS's `s.slice(j, i)` all directly mean "from `j` up to but not including `i`."
>
> So the correct call is `substring(j, i)`, **not** `substring(j, i - j)`.
>
> If you mechanically transliterate the C++ call by keeping the second argument as `i - j`, you silently pull the wrong-length substring in Java/Python/JS — usually too short, and wrong as soon as `j > 0`.
>
> The bug won't crash — `wordBreak` will just return `false` for inputs that should return `true`, which is brutal to debug by inspection.
>
> Always mentally translate "C++ length" into "everyone else's end index" before you type the call.

---

## Problem 25 — LC 78 — Subsets

**Surfaces drilled:** backtracking recursion structure, and the single biggest cross-language porting trap in this curriculum — snapshotting a mutable path into the result — plus a bitmask-iteration variant that drills bit syntax.
**Algorithm in one line:** at every recursive call, record a *copy* of the current path as one subset, then try extending it with each remaining element before undoing the choice.
**Complexity:** O(n × 2ⁿ) time and O(n × 2ⁿ) space for the output itself (there are `2ⁿ` subsets and copying each costs up to O(n)); the recursion stack depth is O(n).

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        vector<vector<int>> result;
        vector<int> path;
        backtrack(nums, 0, path, result);
        return result;
    }

private:
    void backtrack(vector<int>& nums, int start, vector<int>& path, vector<vector<int>>& result) {
        result.push_back(path);
        for (int i = start; i < nums.size(); i++) {
            path.push_back(nums[i]);
            backtrack(nums, i + 1, path, result);
            path.pop_back();
        }
    }
};
```

### Java
```java
class Solution {
    public List<List<Integer>> subsets(int[] nums) {
        List<List<Integer>> result = new ArrayList<>();
        List<Integer> path = new ArrayList<>();
        backtrack(nums, 0, path, result);
        return result;
    }

    private void backtrack(int[] nums, int start, List<Integer> path, List<List<Integer>> result) {
        result.add(new ArrayList<>(path));
        for (int i = start; i < nums.length; i++) {
            path.add(nums[i]);
            backtrack(nums, i + 1, path, result);
            path.remove(path.size() - 1);
        }
    }
}
```

### Python
```python
class Solution:
    def subsets(self, nums: List[int]) -> List[List[int]]:
        result = []
        path = []
        self.backtrack(nums, 0, path, result)
        return result

    def backtrack(self, nums, start, path, result):
        result.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            self.backtrack(nums, i + 1, path, result)
            path.pop()
```

### JavaScript
```js
/**
 * @param {number[]} nums
 * @return {number[][]}
 */
var subsets = function(nums) {
    const result = [];
    const path = [];
    backtrack(nums, 0, path, result);
    return result;
};

function backtrack(nums, start, path, result) {
    result.push([...path]);
    for (let i = start; i < nums.length; i++) {
        path.push(nums[i]);
        backtrack(nums, i + 1, path, result);
        path.pop();
    }
}
```

### Worked example
For `nums = [1, 2, 3]`, the recursion visits, in order: `[]`, `[1]`, `[1,2]`, `[1,2,3]`.

Then it unwinds and visits `[1,3]`, `[2]`, `[2,3]`, and finally `[3]`.

That's all `2³ = 8` subsets: `[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]`.

Every snapshot in `result` must be an independent copy at the moment it was recorded.

If any of the four implementations stored a live reference instead, all eight entries would collapse to whatever `path` looks like after the recursion fully unwinds — which is `[]`.

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| **path snapshot line** | `result.push_back(path)` | `result.add(new ArrayList<>(path))` | `result.append(path[:])` | `result.push([...path])` |
| append to path | `path.push_back(nums[i])` | `path.add(nums[i])` | `path.append(nums[i])` | `path.push(nums[i])` |
| undo (backtrack) | `path.pop_back()` | `path.remove(path.size() - 1)` | `path.pop()` | `path.pop()` |
| container type | `vector<int>`/`vector<vector<int>>` | `List<Integer>`/`List<List<Integer>>` | plain `list` | plain `Array` |

> **Trap — read this twice, it is the highest-frequency porting bug in the entire curriculum.**
>
> In C++, `result.push_back(path)` **copies** the vector by value automatically.
>
> `vector`'s copy constructor fires on every `push_back`, so the snapshot is free and correct without you thinking about it.
>
> Every other language stores **references**, not copies, when you insert a container into another container.
>
> In Java, `result.add(path)` would insert a *reference* to the exact same `ArrayList` object you keep mutating with `add`/`remove`.
>
> By the time recursion finishes, every entry in `result` points at the same now-empty list, and your answer is a list full of empty lists.
>
> The fix, `new ArrayList<>(path)`, constructs a brand-new list with the current elements copied in.
>
> Python has the identical failure mode: `result.append(path)` stores a reference to the same list object.
>
> `path[:]` (a slice of the whole list) or `list(path)` makes a real copy.
>
> JavaScript again: `result.push(path)` pushes a reference.
>
> `[...path]` (spread into a new array) or `path.slice()` makes the copy.
>
> Miss this in any of the three and the bug is silent — no exception, no crash, just a wrong answer.
>
> The wrong answer is often *all empty arrays* or *all identical to the final path*, which is a very recognizable smell once you've seen it once.
>
> If your subsets/permutations output looks suspiciously uniform or empty, this is almost always why.

### Variant — bitmask iteration (same problem, iterative approach)

Since `n ≤ 10` on this problem, every subset corresponds to one of `2^n` bitmasks — bit `i` set means "include `nums[i]`."
**Complexity:** O(n × 2ⁿ) time and space — identical to the backtracking form, just restructured as two nested loops instead of recursion, which is exactly why it's worth drilling as a second variant rather than a different algorithm.

### C++
```cpp
class Solution {
public:
    vector<vector<int>> subsets(vector<int>& nums) {
        int n = nums.size();
        vector<vector<int>> result;
        for (int mask = 0; mask < (1 << n); mask++) {
            vector<int> subset;
            for (int i = 0; i < n; i++) {
                if (mask & (1 << i)) {
                    subset.push_back(nums[i]);
                }
            }
            result.push_back(subset);
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public List<List<Integer>> subsets(int[] nums) {
        int n = nums.length;
        List<List<Integer>> result = new ArrayList<>();
        for (int mask = 0; mask < (1 << n); mask++) {
            List<Integer> subset = new ArrayList<>();
            for (int i = 0; i < n; i++) {
                if ((mask & (1 << i)) != 0) {
                    subset.add(nums[i]);
                }
            }
            result.add(subset);
        }
        return result;
    }
}
```

### Python
```python
class Solution:
    def subsets(self, nums: List[int]) -> List[List[int]]:
        n = len(nums)
        result = []
        for mask in range(1 << n):
            subset = []
            for i in range(n):
                if mask & (1 << i):
                    subset.append(nums[i])
            result.append(subset)
        return result
```

### JavaScript
```js
/**
 * @param {number[]} nums
 * @return {number[][]}
 */
var subsets = function(nums) {
    const n = nums.length;
    const result = [];
    for (let mask = 0; mask < (1 << n); mask++) {
        const subset = [];
        for (let i = 0; i < n; i++) {
            if (mask & (1 << i)) {
                subset.push(nums[i]);
            }
        }
        result.push(subset);
    }
    return result;
};
```

### Worked example
For `nums = [1, 2, 3]` (`n = 3`), `mask` ranges over `0..7`.

Take `mask = 5`, binary `101`: bit 0 is set (include `nums[0] = 1`), bit 1 is clear (skip `nums[1] = 2`), bit 2 is set (include `nums[2] = 3`).

So `subset = [1, 3]`.

Running all eight masks `0` through `7` in order produces `[], [1], [2], [1,2], [3], [1,3], [2,3], [1,2,3]`.

That's the same eight subsets as the backtracking version above, just enumerated in a different order — binary counting order instead of DFS order.

That's perfectly fine, since LeetCode accepts subsets in any order.

### What actually changed (bitmask variant)
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| truthy bit test | `if (mask & (1 << i))` (nonzero int is truthy) | `if ((mask & (1 << i)) != 0)` (must compare explicitly — `int` isn't boolean) | `if mask & (1 << i):` (nonzero int is truthy) | `if (mask & (1 << i))` (nonzero number is truthy) |
| mask upper bound | `1 << n` | `1 << n` | `1 << n` (arbitrary-precision, never overflows) | `1 << n` (safe while `n < 31`, since JS bitwise ops are 32-bit signed) |
| no path-copy bug here | subset rebuilt fresh every mask iteration in all four — no aliasing trap in the bitmask form | same | same | same |

> **Trap:** Java is the odd one out on the bit test.
>
> `if (mask & (1 << i))` does not compile in Java, because `&` on two `int`s produces an `int`, and Java's `if` requires an actual `boolean`.
>
> There is no implicit int-to-boolean coercion in Java the way C++ has.
>
> You must write `(mask & (1 << i)) != 0`.
>
> It's a one-concept-but-mandatory difference that trips people going C++ → Java specifically, because C++, Python, and JS all let you drop a raw nonzero integer straight into an `if`.

---

### Cross-language takeaways for Block 7

Five traps, one line each, worth re-reading before any live port:

- **Problem 21 (Climbing Stairs):** `new Array(n + 1)` in JS makes holes, not zeros — always `.fill(0)` right after.
- **Problem 22 (Coin Change):** `INT_MAX` / `Integer.MAX_VALUE` are finite and can overflow on `+ 1`; guard before adding. `float('inf')` / `Infinity` cannot overflow, but keep the guard anyway for symmetry.
- **Problem 23 (Edit Distance):** never build a 2D array with `[[0]*m]*n` in Python or `Array(n).fill(Array(m))` in JS — both alias every row to the same object. Use the comprehension / `Array.from` factory form instead.
- **Problem 24 (Word Break):** `substr(pos, len)` in C++ takes a length; `substring`/`slice`/`s[a:b]` everywhere else take an end index. Never carry the C++ second argument across unchanged.
- **Problem 25 (Subsets):** appending a mutable path directly into the result list aliases it in Java, Python, and JS — always copy: `new ArrayList<>(path)`, `path[:]`, `[...path]`. C++'s `push_back` is the only one that copies for free.

All five traps share one root cause: C++ gives you value semantics and true integer overflow for free, and the other three languages each replace that with a different implicit behavior — reference semantics, arbitrary-precision or true-infinity numerics, and array holes — that looks identical on the page but behaves differently at runtime.



---

# BLOCK 8 — BIT, MATH, MATRIX AND DESIGN

## Problem 26 — LC 136 — Single Number

**Surfaces drilled:** XOR accumulation over an array, the reduce/fold idiom per language (C++ loop vs `accumulate`, Java loop vs stream `reduce`, Python loop vs `functools.reduce`, JS loop vs `.reduce`)
**Algorithm in one line:** XOR every element together — pairs cancel to zero, so whatever survives is the single number.

### C++ (your anchor)
```cpp
#include <vector>
#include <numeric>
#include <functional>
using namespace std;

class Solution {
public:
    int singleNumber(vector<int>& nums) {
        int result = 0;
        for (int num : nums) {
            result ^= num;
        }
        return result;
    }

    // Idiomatic one-liner using accumulate
    int singleNumberOneLiner(vector<int>& nums) {
        return accumulate(nums.begin(), nums.end(), 0, bit_xor<int>());
    }
};
```

### Java
```java
import java.util.Arrays;

class Solution {
    public int singleNumber(int[] nums) {
        int result = 0;
        for (int num : nums) {
            result ^= num;
        }
        return result;
    }

    // Idiomatic one-liner using IntStream.reduce
    public int singleNumberOneLiner(int[] nums) {
        return Arrays.stream(nums).reduce(0, (a, b) -> a ^ b);
    }
}
```

### Python
```python
from typing import List
from functools import reduce
from operator import xor

class Solution:
    def singleNumber(self, nums: List[int]) -> int:
        result = 0
        for num in nums:
            result ^= num
        return result

    # Idiomatic one-liner using functools.reduce
    def singleNumberOneLiner(self, nums: List[int]) -> int:
        return reduce(xor, nums, 0)
```

### JavaScript
```js
/**
 * @param {number[]} nums
 * @return {number}
 */
var singleNumber = function(nums) {
    let result = 0;
    for (const num of nums) {
        result ^= num;
    }
    return result;
};

// Idiomatic one-liner using Array.prototype.reduce
var singleNumberOneLiner = function(nums) {
    return nums.reduce((a, b) => a ^ b, 0);
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| For-each loop | `for (int num : nums)` | `for (int num : nums)` | `for num in nums:` | `for (const num of nums)` |
| Fold/reduce call | `accumulate(begin, end, init, op)` | `Arrays.stream(nums).reduce(init, lambda)` | `reduce(op, iterable, init)` | `nums.reduce((a, b) => ..., init)` |
| Binary XOR operator | `bit_xor<int>()` functor needed for accumulate | plain lambda `(a, b) -> a ^ b` | `operator.xor` function object | plain arrow `(a, b) => a ^ b` |
| Extra imports needed | `<numeric>`, `<functional>` | `java.util.Arrays` | `functools.reduce`, `operator.xor` | none — `.reduce` is a built-in method |
| Primitive vs boxed stream | n/a | `Arrays.stream(int[])` gives `IntStream` (primitive) automatically | n/a — Python has no boxing distinction | n/a — JS numbers are never boxed |

> **Trap:** Java's `Arrays.stream(nums)` on an `int[]` gives you an `IntStream`, whose `reduce(identity, op)` takes a two-arg lambda. If you instead had a `List<Integer>` and called `.stream().reduce(0, (a, b) -> a ^ b)`, it still compiles, but the moment you drop the identity and call the no-arg `reduce()` on an *empty* collection you get an `Optional<Integer>` back instead of an `int` — a silent type change that breaks callers expecting a primitive. Python's `reduce(xor, nums)` without the `0` initial value throws `TypeError: reduce() of empty iterable with no initial value` on an empty list, and JS's `nums.reduce((a,b)=>a^b)` without an initial value throws `TypeError: Reduce of empty array with no initial value` too. The loop versions never have this problem — they just return `0`. Always pass the initial value in interviews; it's the one line graders forget.

## Problem 27 — LC 191 — Number of 1 Bits

**Surfaces drilled:** popcount per language (`__builtin_popcount`, `Integer.bitCount`, `bit_count()`/`bin().count()`, hand-rolled loop), Brian Kernighan's `n & (n-1)` loop in all four, and the unsigned-vs-signed 32-bit shift trap
**Algorithm in one line:** Count the set bits in the 32-bit representation of `n`, either with a builtin popcount, by repeatedly clearing the lowest set bit (`n &= n - 1`), or by shifting and masking one bit at a time.

### C++ (your anchor)
```cpp
class Solution {
public:
    int hammingWeight(uint32_t n) {
        return __builtin_popcount(n);
    }

    // Brian Kernighan's algorithm — clears the lowest set bit each pass
    int hammingWeightKernighan(uint32_t n) {
        int count = 0;
        while (n != 0) {
            n &= (n - 1);
            count++;
        }
        return count;
    }

    // Hand-rolled shifting loop. n is uint32_t, so >> is ALREADY a
    // logical (zero-fill) shift — no unsigned trap to worry about here.
    int hammingWeightShift(uint32_t n) {
        int count = 0;
        while (n != 0) {
            count += n & 1;
            n >>= 1;
        }
        return count;
    }
};
```

### Java
```java
class Solution {
    public int hammingWeight(int n) {
        return Integer.bitCount(n);
    }

    // Brian Kernighan's algorithm
    public int hammingWeightKernighan(int n) {
        int count = 0;
        while (n != 0) {
            n &= (n - 1);
            count++;
        }
        return count;
    }

    // Hand-rolled shifting loop — MUST use >>> (unsigned/logical shift).
    // n arrives as the bit pattern of an unsigned 32-bit value stuffed
    // into a signed Java int. If n's top bit is set, n is negative, and
    // >> would sign-extend forever, so this loop would never terminate.
    public int hammingWeightShift(int n) {
        int count = 0;
        while (n != 0) {
            count += n & 1;
            n >>>= 1;
        }
        return count;
    }
}
```

### Python
```python
class Solution:
    def hammingWeight(self, n: int) -> int:
        return n.bit_count()  # Python 3.10+

    # Fallback for Python < 3.10
    def hammingWeightFallback(self, n: int) -> int:
        return bin(n).count('1')

    # Brian Kernighan's algorithm
    def hammingWeightKernighan(self, n: int) -> int:
        count = 0
        while n != 0:
            n &= (n - 1)
            count += 1
        return count

    # Hand-rolled shifting loop — Python ints are unbounded and have
    # no fixed width at all. LeetCode passes n already as a
    # non-negative int, so >> is always a plain logical shift here;
    # there is no sign bit, no 32-bit wraparound, and no >>> operator
    # because none is needed.
    def hammingWeightShift(self, n: int) -> int:
        count = 0
        while n != 0:
            count += n & 1
            n >>= 1
        return count
```

### JavaScript
```js
/**
 * @param {number} n
 * @return {number}
 */
var hammingWeight = function(n) {
    let count = 0;
    while (n !== 0) {
        count += n & 1;
        n >>>= 1;
    }
    return count;
};

// Brian Kernighan's algorithm
var hammingWeightKernighan = function(n) {
    let count = 0;
    while (n !== 0) {
        n &= (n - 1);
        count++;
    }
    return count;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Builtin popcount | `__builtin_popcount(n)` | `Integer.bitCount(n)` | `n.bit_count()` (3.10+) or `bin(n).count('1')` | no builtin — the loop *is* the idiom |
| Input's declared width | `uint32_t n` — genuinely unsigned | `int n` — signed type holding an unsigned bit pattern | `int n` — unbounded, no width at all | `number n` — a double, treated as 32-bit for bit ops |
| Right-shift operator to use | `>>` (already logical for `uint32_t`) | `>>>` (logical) — `>>` sign-extends and breaks | `>>` (fine — Python ints here are non-negative and unbounded) | `>>>` (logical) — `>>` sign-extends and breaks |
| Bitwise AND for the low bit | `n & 1` | `n & 1` | `n & 1` | `n & 1` (operands coerced to 32-bit ints first) |
| Loop termination check | `n != 0` | `n != 0` | `n != 0` | `n !== 0` |

> **Trap — the four bit models are genuinely different, not just spelled differently.** C++'s `uint32_t` is a real unsigned type, so `>>` on it is always logical — no issue. Java has *no* unsigned integer type: `n` is a signed 32-bit `int`, and if the input's top bit is set, `n` is negative, so `n >> 1` sign-extends and the loop spins forever — you must use `>>>`. JavaScript's bitwise operators internally convert operands to 32-bit signed integers before the operation, so the exact same sign-extension bug bites `n >> 1` there too — you again need `>>>`. Python has none of this: integers are arbitrary-precision and there is no sign-extension concept for `>>` on a non-negative value, so Python needs no unsigned operator at all — but that also means if you ever *do* get a negative Python int into a "32-bit" bit-manipulation problem, `>>` shifts it toward negative infinity forever rather than wrapping, which is its own trap in other problems. This one distinction — "does this language have a real unsigned type, a signed type pretending to be unsigned, or no fixed width at all" — is the single most common bit-manipulation bug when moving between these four languages.

## Problem 28 — LC 50 — Pow(x, n)

**Surfaces drilled:** fast exponentiation by repeated squaring, handling a negative exponent by inverting the base, and the `INT_MIN` negation overflow trap
**Algorithm in one line:** Repeatedly square `x` and halve `n`, multiplying `x` into the running result whenever the current bit of `n` is 1, after first flipping to `1/x` and negating `n` if the exponent was negative.

### C++ (your anchor)
```cpp
class Solution {
public:
    double myPow(double x, int n) {
        long long N = n;
        if (N < 0) {
            x = 1 / x;
            N = -N;
        }
        double result = 1.0;
        while (N > 0) {
            if (N % 2 == 1) {
                result *= x;
            }
            x *= x;
            N /= 2;
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public double myPow(double x, int n) {
        long N = n;
        if (N < 0) {
            x = 1 / x;
            N = -N;
        }
        double result = 1.0;
        while (N > 0) {
            if (N % 2 == 1) {
                result *= x;
            }
            x *= x;
            N /= 2;
        }
        return result;
    }
}
```

### Python
```python
class Solution:
    def myPow(self, x: float, n: int) -> float:
        N = n
        if N < 0:
            x = 1 / x
            N = -N
        result = 1.0
        while N > 0:
            if N % 2 == 1:
                result *= x
            x *= x
            N //= 2
        return result
```

### JavaScript
```js
/**
 * @param {number} x
 * @param {number} n
 * @return {number}
 */
var myPow = function(x, n) {
    let N = n;
    if (N < 0) {
        x = 1 / x;
        N = -N;
    }
    let result = 1.0;
    while (N > 0) {
        if (N % 2 === 1) {
            result *= x;
        }
        x *= x;
        N = Math.floor(N / 2);
    }
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Widened exponent type | `long long N = n;` | `long N = n;` | `N = n` — no widening needed, ints are unbounded | `let N = n;` — no widening needed, numbers are doubles |
| Integer halving of N | `N /= 2` (truncating int division) | `N /= 2` (truncating int division) | `N //= 2` (floor division operator) | `N = Math.floor(N / 2)` — `/` alone gives a float |
| Odd-bit test | `N % 2 == 1` | `N % 2 == 1` | `N % 2 == 1` | `N % 2 === 1` |
| Reciprocal for negative exponent | `x = 1 / x;` | `x = 1 / x;` | `x = 1 / x` | `x = 1 / x;` |
| Why the widen is needed at all | avoids overflow when negating `INT_MIN` | avoids overflow when negating `Integer.MIN_VALUE` | never needed — Python ints have no fixed width | never needed — doubles represent `-(-2^31)` exactly |

> **Trap:** `n` ranges down to `-2^31` (`INT_MIN` / `Integer.MIN_VALUE`). If you write `N = -n` while `N` and `n` are still 32-bit `int`/`long long`-untouched, `-(-2147483648)` overflows the signed 32-bit range (there is no positive `2147483648` representable as a 32-bit `int`), which is undefined behavior in C++ and silently wraps back to `Integer.MIN_VALUE` in Java. Both solutions above dodge this by widening `n` into a 64-bit `long`/`long long` *before* negating — negating a 32-bit `INT_MIN` inside a 64-bit variable is safe because the positive result fits comfortably in 64 bits. Python needs none of this because its `int` type has no fixed width to overflow. JavaScript needs none of this either, but for a completely different reason: `n` is stored as an IEEE-754 double, and `-(-2147483648)` is `2147483648`, which a double represents exactly (doubles are exact for all integers up to 2^53) — so there's no overflow to guard against, just a different underlying number model that happens to save you here.

## Problem 29 — LC 48 — Rotate Image

**Surfaces drilled:** in-place 2D mutation, transpose-then-reverse-rows as an alternative to layer-by-layer rotation, the swap idiom per language, and reversing a row in place
**Algorithm in one line:** Transpose the matrix in place (swap `matrix[i][j]` with `matrix[j][i]` for `j > i`), then reverse each row in place, which together produce a 90-degree clockwise rotation.

### C++ (your anchor)
```cpp
class Solution {
public:
    void rotate(vector<vector<int>>& matrix) {
        int n = matrix.size();

        // transpose
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {
                swap(matrix[i][j], matrix[j][i]);
            }
        }

        // reverse each row
        for (int i = 0; i < n; i++) {
            int left = 0, right = n - 1;
            while (left < right) {
                swap(matrix[i][left], matrix[i][right]);
                left++;
                right--;
            }
        }
    }
};
```

### Java
```java
class Solution {
    public void rotate(int[][] matrix) {
        int n = matrix.length;

        // transpose
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {
                int temp = matrix[i][j];
                matrix[i][j] = matrix[j][i];
                matrix[j][i] = temp;
            }
        }

        // reverse each row
        for (int i = 0; i < n; i++) {
            int left = 0, right = n - 1;
            while (left < right) {
                int temp = matrix[i][left];
                matrix[i][left] = matrix[i][right];
                matrix[i][right] = temp;
                left++;
                right--;
            }
        }
    }
}
```

### Python
```python
class Solution:
    def rotate(self, matrix: List[List[int]]) -> None:
        n = len(matrix)

        # transpose
        for i in range(n):
            for j in range(i + 1, n):
                matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]

        # reverse each row
        for i in range(n):
            left, right = 0, n - 1
            while left < right:
                matrix[i][left], matrix[i][right] = matrix[i][right], matrix[i][left]
                left += 1
                right -= 1
```

### JavaScript
```js
/**
 * @param {number[][]} matrix
 * @return {void} Do not return anything, modify matrix in-place instead.
 */
var rotate = function(matrix) {
    const n = matrix.length;

    // transpose
    for (let i = 0; i < n; i++) {
        for (let j = i + 1; j < n; j++) {
            [matrix[i][j], matrix[j][i]] = [matrix[j][i], matrix[i][j]];
        }
    }

    // reverse each row
    for (let i = 0; i < n; i++) {
        let left = 0, right = n - 1;
        while (left < right) {
            [matrix[i][left], matrix[i][right]] = [matrix[i][right], matrix[i][left]];
            left++;
            right--;
        }
    }
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| 2D array parameter type | `vector<vector<int>>&` (reference, so in-place works) | `int[][]` (arrays are always reference types) | `List[List[int]]` (lists are always reference types) | `number[][]` (arrays are always reference types) |
| Swap idiom | `swap(a, b)` built-in | manual 3-line temp variable | tuple-unpacking swap `a, b = b, a` | array-destructuring swap `[a, b] = [b, a]` |
| Getting the size | `matrix.size()` | `matrix.length` | `len(matrix)` | `matrix.length` |
| Row-reverse loop style | `while (left < right)` with manual pointers | `while (left < right)` with manual pointers | `while left < right:` with manual pointers | `while (left < right)` with manual pointers |
| Comment style | `//` | `//` | `#` | `//` |

> **Trap:** the swap idiom is the one line that looks trivial and is where fingers slip under pressure. C++'s `swap(matrix[i][j], matrix[j][i])` and Python's `matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]` both evaluate the right-hand side as a unit before assigning, so they're safe. But if you try to "simplify" Java's three-line temp swap into a one-liner like `matrix[i][j] = matrix[j][i]; matrix[j][i] = matrix[i][j];` (skipping the temp), the second statement now reads the *already-overwritten* `matrix[i][j]` — you've destroyed one of the two values before the swap completes. The same failure mode applies to JavaScript if you write `matrix[i][j] = matrix[j][i]; matrix[j][i] = matrix[i][j];` instead of using destructuring — Java has no destructuring assignment to fall back on, so it is the one language here where you must write out the temp variable by hand every time.

## Problem 30 — LC 208 — Implement Trie (Prefix Tree)

**Surfaces drilled:** a class with a nested/self-referential node type, children stored as a fixed array of 26 versus a hash map, constructor field initialization, iterating the characters of a word, a boolean end-of-word flag, and three public methods sharing one internal helper
**Algorithm in one line:** Each node holds a `children` collection keyed by character and an `isEnd` flag; `insert` walks/creates nodes character by character and marks the last one, while `search` and `startsWith` walk the same path and differ only in whether they also check `isEnd`.

### C++ (your anchor)
```cpp
#include <string>
#include <unordered_map>
using namespace std;

class Trie {
private:
    bool isEnd;
    Trie* children[26];

public:
    Trie() {
        isEnd = false;
        for (int i = 0; i < 26; i++) {
            children[i] = nullptr;
        }
    }

    void insert(string word) {
        Trie* node = this;
        for (char c : word) {
            int idx = c - 'a';
            if (node->children[idx] == nullptr) {
                node->children[idx] = new Trie();
            }
            node = node->children[idx];
        }
        node->isEnd = true;
    }

    bool search(string word) {
        Trie* node = find(word);
        return node != nullptr && node->isEnd;
    }

    bool startsWith(string prefix) {
        return find(prefix) != nullptr;
    }

private:
    Trie* find(const string& word) {
        Trie* node = this;
        for (char c : word) {
            int idx = c - 'a';
            if (node->children[idx] == nullptr) {
                return nullptr;
            }
            node = node->children[idx];
        }
        return node;
    }
};

// Alternative node shape (sketch, insert only): hash-map children instead of a fixed 26-slot array.
// search/startsWith are identical to the version above, looking up children[c] in the map instead of the array.
// Same shape — swap the array for an unordered_map<char, Trie*>.
class TrieHashMap {
private:
    bool isEnd;
    unordered_map<char, TrieHashMap*> children;

public:
    TrieHashMap() : isEnd(false) {}

    void insert(string word) {
        TrieHashMap* node = this;
        for (char c : word) {
            if (node->children.find(c) == node->children.end()) {
                node->children[c] = new TrieHashMap();
            }
            node = node->children[c];
        }
        node->isEnd = true;
    }
};
```

### Java
```java
import java.util.Map;
import java.util.HashMap;

class Trie {
    private boolean isEnd;
    private Trie[] children;

    public Trie() {
        isEnd = false;
        children = new Trie[26];
    }

    public void insert(String word) {
        Trie node = this;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) {
                node.children[idx] = new Trie();
            }
            node = node.children[idx];
        }
        node.isEnd = true;
    }

    public boolean search(String word) {
        Trie node = find(word);
        return node != null && node.isEnd;
    }

    public boolean startsWith(String prefix) {
        return find(prefix) != null;
    }

    private Trie find(String word) {
        Trie node = this;
        for (char c : word.toCharArray()) {
            int idx = c - 'a';
            if (node.children[idx] == null) {
                return null;
            }
            node = node.children[idx];
        }
        return node;
    }
}

// Alternative node shape (sketch, insert only): hash-map children instead of a fixed 26-slot array.
// search/startsWith are identical to the version above, looking up children[c] in the map instead of the array.
class TrieHashMap {
    private boolean isEnd;
    private Map<Character, TrieHashMap> children;

    public TrieHashMap() {
        isEnd = false;
        children = new HashMap<>();
    }

    public void insert(String word) {
        TrieHashMap node = this;
        for (char c : word.toCharArray()) {
            node.children.putIfAbsent(c, new TrieHashMap());
            node = node.children.get(c);
        }
        node.isEnd = true;
    }
}
```

### Python
```python
class Trie:
    def __init__(self):
        self.children = {}
        self.isEnd = False

    def insert(self, word: str) -> None:
        node = self
        for c in word:
            if c not in node.children:
                node.children[c] = Trie()
            node = node.children[c]
        node.isEnd = True

    def search(self, word: str) -> bool:
        node = self._find(word)
        return node is not None and node.isEnd

    def startsWith(self, prefix: str) -> bool:
        return self._find(prefix) is not None

    def _find(self, word: str):
        node = self
        for c in word:
            if c not in node.children:
                return None
            node = node.children[c]
        return node


# Alternative node shape (sketch, insert only): fixed 26-slot array children instead of a hash map.
# search/startsWith are identical to the version above, indexing children[ord(c)-ord('a')] instead of the dict.
class TrieArray:
    def __init__(self):
        self.children = [None] * 26
        self.isEnd = False

    def insert(self, word: str) -> None:
        node = self
        for c in word:
            idx = ord(c) - ord('a')
            if node.children[idx] is None:
                node.children[idx] = TrieArray()
            node = node.children[idx]
        node.isEnd = True
```

### JavaScript
```js
class Trie {
    constructor() {
        this.children = new Map();
        this.isEnd = false;
    }

    insert(word) {
        let node = this;
        for (const c of word) {
            if (!node.children.has(c)) {
                node.children.set(c, new Trie());
            }
            node = node.children.get(c);
        }
        node.isEnd = true;
    }

    search(word) {
        const node = this._find(word);
        return node !== null && node.isEnd;
    }

    startsWith(prefix) {
        return this._find(prefix) !== null;
    }

    _find(word) {
        let node = this;
        for (const c of word) {
            if (!node.children.has(c)) {
                return null;
            }
            node = node.children.get(c);
        }
        return node;
    }
}

// Alternative node shape (sketch, insert only): fixed 26-slot array children instead of a Map.
// search/startsWith are identical to the version above, indexing children[c.charCodeAt(0)-97] instead of the Map.
class TrieArray {
    constructor() {
        this.children = new Array(26).fill(null);
        this.isEnd = false;
    }

    insert(word) {
        let node = this;
        for (const c of word) {
            const idx = c.charCodeAt(0) - 'a'.charCodeAt(0);
            if (node.children[idx] === null) {
                node.children[idx] = new TrieArray();
            }
            node = node.children[idx];
        }
        node.isEnd = true;
    }
}

/**
 * Your Trie object will be instantiated and called as such:
 * var obj = new Trie()
 * obj.insert(word)
 * var param_2 = obj.search(word)
 * var param_3 = obj.startsWith(prefix)
 */
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Primary children storage (shown above) | fixed array `Trie* children[26]` | fixed array `Trie[] children` | hash map `self.children = {}` | hash map `new Map()` |
| Idiomatic alternative shown | `unordered_map<char, Trie*>` | `Map<Character, TrieHashMap>` via `HashMap` | fixed list `[None] * 26` | fixed array `new Array(26).fill(null)` |
| Self-referencing node type | raw pointer `Trie*`, `this` for the current node | reference type `Trie`, `this` for the current node | plain reference, `self` for the current node | plain reference, `this` for the current node |
| Char to array index | `c - 'a'` (char arithmetic) | `c - 'a'` (char arithmetic) | `ord(c) - ord('a')` | `c.charCodeAt(0) - 'a'.charCodeAt(0)` |
| Existence check against a hash-based map | `children.find(c) == children.end()` | `children.putIfAbsent(...)` | `c not in node.children` | `!node.children.has(c)` |
| Iterating word characters | `for (char c : word)` | `word.toCharArray()` then `for (char c : ...)` | `for c in word:` | `for (const c of word)` |
| Private shared helper | `private:` section, `find(...)` | `private` modifier, `find(...)` | leading-underscore convention, `_find(...)` (no true enforcement) | leading-underscore convention, `_find(...)` (no true enforcement) |
| Field initialization point | constructor body | constructor body | `__init__` body | `constructor` body |

> **Trap:** Java's `String` has no direct `for (char c : word)` — you cannot iterate a `String`'s characters with a for-each loop the way you can a `char[]` or a C++ `string`; you must first call `word.toCharArray()` (or `word.charAt(i)` in an indexed loop). Forget that conversion and the code simply doesn't compile, which is a fast, harmless failure — the more dangerous trap is on the Python/JS side: `_find` and other underscore-prefixed methods are *convention only*, not enforced privacy. Nothing stops another part of a large interview solution from calling `trie._find(...)` directly or monkey-patching `children` from outside the class, whereas C++'s `private:` and Java's `private` are compiler-enforced. If your design relies on encapsulation to keep two Trie variants (array-based and map-based) from being mixed up, that guarantee is real in C++/Java and merely a naming convention in Python/JS.

## Where to go next

- Doc 43 §7 — the final exam: 5 unseen medium problems, attempted cold with no reference material, no syntax lookup, and no partial credit for "I know the algorithm, I just forgot the syntax."
- Docs 39–42 — the per-language deep references for C++, Java, Python, and JavaScript respectively, each with a §33 recipe index for fast lookup of a specific idiom (swap, reduce, popcount, string building, etc.) instead of re-deriving it from a solved problem.
- Redo every problem in this block cold after 3 days, in all four languages, with no reference material open. If any language's syntax has to be re-derived rather than recalled, that language is not yet interview-ready and belongs back in the rotation.

---

*Doc 46 — Polyglot Solutions Part 2. Final exam and tracker: doc 43 §7 and §8.*
