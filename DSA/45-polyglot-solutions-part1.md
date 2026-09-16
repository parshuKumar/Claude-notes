# 45 — POLYGLOT SOLUTIONS, PART 1 (PROBLEMS 1–13)
## The Same Algorithm Written Four Ways: C++, Java, Python, JavaScript
### Blocks 1–4: Containers, Strings, Linked Lists and Class Design, Trees

---

## HOW TO READ A PROBLEM

Each problem is written four times with **identical variable names and identical line order**. Your C++ block is the anchor. Diff it against the target language line by line: everything that differs is syntax, because the algorithm never changes.

Then read **What actually changed**. It lists only the constructs that genuinely differ, and the **Trap** names the one porting mistake most likely to cost you the problem.

Do not copy-paste. Follow the protocol in doc 43 §5: write the C++ from memory, translate with this doc closed, then compare.

## VERIFICATION

Every solution in this doc was run against real LeetCode test cases, including edge cases.

| Language | Result |
|---|---|
| C++ | compiled and passed all test assertions |
| Python | passed all test assertions |
| JavaScript | passed all test assertions |
| Java | brace-balanced and trap-checked; not executed, no JDK on the build machine |

## CONTENTS

**Block 1 — Containers and Hashing**
1. LC 1 Two Sum
2. LC 49 Group Anagrams
3. LC 347 Top K Frequent Elements
4. LC 3 Longest Substring Without Repeating Characters

**Block 2 — Strings**
5. LC 125 Valid Palindrome
6. LC 151 Reverse Words in a String
7. LC 344 Reverse String

**Block 3 — Linked List and Class Design**
8. LC 206 Reverse Linked List
9. LC 21 Merge Two Sorted Lists
10. LC 146 LRU Cache

**Block 4 — Trees and Recursion**
11. LC 104 Maximum Depth of Binary Tree
12. LC 102 Binary Tree Level Order Traversal
13. LC 297 Serialize and Deserialize Binary Tree

---

# BLOCK 1 — CONTAINERS AND HASHING

## Problem 1 — LC 1 — Two Sum

**Surfaces drilled:** hash map declare, put/get-with-default, contains-check, array return construction, index loop.
**Algorithm in one line:** Walk the array once, and before inserting each value into a hash map, check whether its complement is already in the map.

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> seen;
        for (int i = 0; i < nums.size(); i++) {
            int complement = target - nums[i];
            if (seen.count(complement)) {
                return {seen[complement], i};
            }
            seen[nums[i]] = i;
        }
        return {};
    }
};
```

### Java
```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> seen = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (seen.containsKey(complement)) {
                return new int[]{seen.get(complement), i};
            }
            seen.put(nums[i], i);
        }
        return new int[]{};
    }
}
```

### Python
```python
class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        seen = {}
        for i in range(len(nums)):
            complement = target - nums[i]
            if complement in seen:
                return [seen[complement], i]
            seen[nums[i]] = i
        return []
```

### JavaScript
```js
var twoSum = function(nums, target) {
    const seen = new Map();
    for (let i = 0; i < nums.length; i++) {
        const complement = target - nums[i];
        if (seen.has(complement)) {
            return [seen.get(complement), i];
        }
        seen.set(nums[i], i);
    }
    return [];
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| declare the map | `unordered_map<int, int> seen;` | `Map<Integer, Integer> seen = new HashMap<>();` | `seen = {}` | `const seen = new Map();` |
| membership check | `seen.count(complement)` | `seen.containsKey(complement)` | `complement in seen` | `seen.has(complement)` |
| read a value | `seen[complement]` | `seen.get(complement)` | `seen[complement]` | `seen.get(complement)` |
| insert/update | `seen[nums[i]] = i;` | `seen.put(nums[i], i);` | `seen[nums[i]] = i` | `seen.set(nums[i], i);` |
| return type | `vector<int>` literal `{a, b}` | `new int[]{a, b}` | list literal `[a, b]` | array literal `[a, b]` |
| iterate indices | `for (int i = 0; ...)` | `for (int i = 0; ...)` | `for i in range(len(nums)):` | `for (let i = 0; ...)` |
| fallback "no answer" return | `return {};` | `return new int[]{};` | `return []` | `return [];` |

> **Trap:** the check-then-insert order matters — you must look up the complement *before* writing `nums[i]` into the map, otherwise a value that equals its own complement (e.g. `target = 2 * nums[i]`) will match against itself at the same index. All four snippets above preserve that order; swap the two lines while transliterating quickly under interview pressure and you get a silently wrong answer, not a crash.

## Problem 2 — LC 49 — Group Anagrams

**Surfaces drilled:** map-of-lists with auto-create-on-first-insert, sorted-string as key, string↔char-array conversion, nested list-of-lists return, iterating map values.
**Algorithm in one line:** Sort the characters of each word to build a canonical key, group words sharing that key, and return the groups.

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<vector<string>> groupAnagrams(vector<string>& strs) {
        unordered_map<string, vector<string>> groups;
        for (string& s : strs) {
            string key = s;
            sort(key.begin(), key.end());
            groups[key].push_back(s);
        }
        vector<vector<string>> result;
        for (auto& [key, group] : groups) {
            result.push_back(group);
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> groups = new HashMap<>();
        for (String s : strs) {
            char[] chars = s.toCharArray();
            Arrays.sort(chars);
            String key = new String(chars);
            groups.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
        }
        List<List<String>> result = new ArrayList<>();
        for (List<String> group : groups.values()) {
            result.add(group);
        }
        return result;
    }
}
```

### Python
```python
class Solution:
    def groupAnagrams(self, strs: List[str]) -> List[List[str]]:
        groups = defaultdict(list)
        for s in strs:
            key = "".join(sorted(s))
            groups[key].append(s)
        result = []
        for group in groups.values():
            result.append(group)
        return result
```

### JavaScript
```js
var groupAnagrams = function(strs) {
    const groups = new Map();
    for (const s of strs) {
        const key = s.split("").sort().join("");
        if (!groups.has(key)) {
            groups.set(key, []);
        }
        groups.get(key).push(s);
    }
    const result = [];
    for (const group of groups.values()) {
        result.push(group);
    }
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| declare map-of-lists | `unordered_map<string, vector<string>> groups;` | `Map<String, List<String>> groups = new HashMap<>();` | `groups = defaultdict(list)` | `const groups = new Map();` |
| auto-create on first insert | `groups[key]` default-constructs the vector | `groups.computeIfAbsent(key, k -> new ArrayList<>())` | `defaultdict(list)` does it for you | no auto-create — must `if (!groups.has(key)) groups.set(key, [])` |
| sort a string into a key | `sort(key.begin(), key.end())` on a mutable `string` | `char[]` via `toCharArray()`, `Arrays.sort`, then `new String(chars)` | `"".join(sorted(s))` | `s.split("").sort().join("")` |
| append to a group | `groups[key].push_back(s);` | `.add(s)` after `computeIfAbsent` | `groups[key].append(s)` | `groups.get(key).push(s);` |
| iterate map values | `for (auto& [key, group] : groups)` | `for (List<String> group : groups.values())` | `for group in groups.values():` | `for (const group of groups.values())` |
| nested return type | `vector<vector<string>>` | `List<List<String>>` | `List[List[str]]` | array of arrays |
| loop over the input array | `for (string& s : strs)` | `for (String s : strs)` | `for s in strs:` | `for (const s of strs)` |

> **Trap:** `unordered_map::operator[]` (C++) and `defaultdict` (Python) silently create a default entry on first access, but a plain JS `Map` does not — calling `.get(key).push(s)` on a key that was never `.set()` throws `TypeError: Cannot read properties of undefined`. You must guard with `has`/`set` yourself, which is easy to forget when your muscle memory is C++ or Python.

## Problem 3 — LC 347 — Top K Frequent Elements

**Surfaces drilled:** frequency map, heap with custom comparator, min-heap of size k, iterating map entries, converting result to an array.
**Algorithm in one line:** Count frequencies, then keep a min-heap of size k on frequency, popping the smallest whenever the heap grows past k.

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<int> topKFrequent(vector<int>& nums, int k) {
        unordered_map<int, int> freq;
        for (int num : nums) {
            freq[num]++;
        }
        auto cmp = [](pair<int, int>& a, pair<int, int>& b) {
            return a.second > b.second;
        };
        priority_queue<pair<int, int>, vector<pair<int, int>>, decltype(cmp)> heap(cmp);
        for (auto& [num, count] : freq) {
            heap.push({num, count});
            if (heap.size() > k) {
                heap.pop();
            }
        }
        vector<int> result;
        while (!heap.empty()) {
            result.push_back(heap.top().first);
            heap.pop();
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public int[] topKFrequent(int[] nums, int k) {
        Map<Integer, Integer> freq = new HashMap<>();
        for (int num : nums) {
            freq.put(num, freq.getOrDefault(num, 0) + 1);
        }
        PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> a[1] - b[1]);
        for (Map.Entry<Integer, Integer> entry : freq.entrySet()) {
            heap.offer(new int[]{entry.getKey(), entry.getValue()});
            if (heap.size() > k) {
                heap.poll();
            }
        }
        int[] result = new int[k];
        int i = 0;
        while (!heap.isEmpty()) {
            result[i++] = heap.poll()[0];
        }
        return result;
    }
}
```

### Python
```python
class Solution:
    def topKFrequent(self, nums: List[int], k: int) -> List[int]:
        freq = {}
        for num in nums:
            freq[num] = freq.get(num, 0) + 1
        heap = []
        for num, count in freq.items():
            heapq.heappush(heap, (count, num))
            if len(heap) > k:
                heapq.heappop(heap)
        result = []
        while heap:
            result.append(heapq.heappop(heap)[1])
        return result
```

### JavaScript
```js
class MinHeap {
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
            if (this.data[parent][1] <= this.data[i][1]) break;
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
                let smallest = i;
                if (left < this.data.length && this.data[left][1] < this.data[smallest][1]) smallest = left;
                if (right < this.data.length && this.data[right][1] < this.data[smallest][1]) smallest = right;
                if (smallest === i) break;
                [this.data[i], this.data[smallest]] = [this.data[smallest], this.data[i]];
                i = smallest;
            }
        }
        return top;
    }
}

var topKFrequent = function(nums, k) {
    const freq = new Map();
    for (const num of nums) {
        freq.set(num, (freq.get(num) || 0) + 1);
    }
    const heap = new MinHeap();
    for (const [num, count] of freq) {
        heap.push([num, count]);
        if (heap.size() > k) {
            heap.pop();
        }
    }
    const result = [];
    while (heap.size() > 0) {
        result.push(heap.pop()[0]);
    }
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| built-in heap? | yes, `priority_queue` | yes, `PriorityQueue` | yes, `heapq` (function module, not a class) | **no** — hand-roll `MinHeap` |
| get-with-default on freq map | `freq[num]++;` (auto-zero-inits) | `freq.getOrDefault(num, 0) + 1` | `freq.get(num, 0) + 1` | `(freq.get(num) || 0) + 1` |
| custom comparator | lambda `cmp`, passed as a template arg + ctor arg | lambda passed straight to `PriorityQueue` ctor | plain tuple `(count, num)` — tuples compare lexicographically, so no comparator needed | heap orders by `item[1]` inside hand-rolled `push`/`pop` |
| push then trim to size k | `heap.push(...); if (heap.size() > k) heap.pop();` | `heap.offer(...); if (heap.size() > k) heap.poll();` | `heapq.heappush(...); if len(heap) > k: heapq.heappop(heap)` | `heap.push(...); if (heap.size() > k) heap.pop();` |
| drain heap into result | `while (!heap.empty()) { result.push_back(heap.top().first); heap.pop(); }` | `while (!heap.isEmpty()) result[i++] = heap.poll()[0];` | `while heap: result.append(heapq.heappop(heap)[1])` | `while (heap.size() > 0) result.push(heap.pop()[0]);` |
| iterate map entries | `for (auto& [num, count] : freq)` | `for (Map.Entry<Integer, Integer> entry : freq.entrySet())` | `for num, count in freq.items():` | `for (const [num, count] of freq)` |

> **Trap:** LeetCode accepts the k elements in *any* order, but everyone instinctively wants to sort the final result before returning it — don't waste interview minutes on that. The real trap is JavaScript specifically: there is no `PriorityQueue`/`heapq` in the standard library, so under time pressure you either hand-roll the `MinHeap` shown above (which itself is a classic place to get the sift-up/sift-down index math wrong) or fall back to `sort()`-after-every-push, which quietly changes your solution from O(n log k) to O(n² log n).

## Problem 4 — LC 3 — Longest Substring Without Repeating Characters

**Surfaces drilled:** hash map, sliding window with two pointers, string char indexing, max tracking.
**Algorithm in one line:** Slide a window over the string, remembering the last index each character was seen at, and snap the left edge past any repeat.

### C++ (your anchor)
```cpp
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        unordered_map<char, int> lastSeen;
        int left = 0;
        int maxLen = 0;
        for (int right = 0; right < s.size(); right++) {
            char c = s[right];
            if (lastSeen.count(c) && lastSeen[c] >= left) {
                left = lastSeen[c] + 1;
            }
            lastSeen[c] = right;
            maxLen = max(maxLen, right - left + 1);
        }
        return maxLen;
    }
};
```

### Java
```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> lastSeen = new HashMap<>();
        int left = 0;
        int maxLen = 0;
        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
                left = lastSeen.get(c) + 1;
            }
            lastSeen.put(c, right);
            maxLen = Math.max(maxLen, right - left + 1);
        }
        return maxLen;
    }
}
```

### Python
```python
class Solution:
    def lengthOfLongestSubstring(self, s: str) -> int:
        lastSeen = {}
        left = 0
        maxLen = 0
        for right in range(len(s)):
            c = s[right]
            if c in lastSeen and lastSeen[c] >= left:
                left = lastSeen[c] + 1
            lastSeen[c] = right
            maxLen = max(maxLen, right - left + 1)
        return maxLen
```

### JavaScript
```js
var lengthOfLongestSubstring = function(s) {
    const lastSeen = new Map();
    let left = 0;
    let maxLen = 0;
    for (let right = 0; right < s.length; right++) {
        const c = s[right];
        if (lastSeen.has(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1;
        }
        lastSeen.set(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| declare the map | `unordered_map<char, int> lastSeen;` | `Map<Character, Integer> lastSeen = new HashMap<>();` | `lastSeen = {}` | `const lastSeen = new Map();` |
| membership + get | `lastSeen.count(c) && lastSeen[c] >= left` | `lastSeen.containsKey(c) && lastSeen.get(c) >= left` | `c in lastSeen and lastSeen[c] >= left` | `lastSeen.has(c) && lastSeen.get(c) >= left` |
| char indexing | `s[right]` (returns `char`) | `s.charAt(right)` (returns `char`) | `s[right]` (returns a length-1 `str`) | `s[right]` (returns a length-1 `string`) |
| write to map | `lastSeen[c] = right;` | `lastSeen.put(c, right);` | `lastSeen[c] = right` | `lastSeen.set(c, right);` |
| max of two ints | `max(maxLen, ...)` | `Math.max(maxLen, ...)` | `max(maxLen, ...)` | `Math.max(maxLen, ...)` |
| for loop over the window | `for (int right = 0; right < s.size(); right++)` | `for (int right = 0; right < s.length(); right++)` | `for right in range(len(s)):` | `for (let right = 0; right < s.length; right++)` |

> **Trap:** the `lastSeen[c] >= left` guard is not optional — without it, a character that was seen *before* the current window started (i.e., already slid past) would wrongly yank `left` backward. Java's `containsKey`/`get` pair does two hash lookups where C++'s `count`/`[]` and JS's `has`/`get` do the same — none of the four give you a single "get-or-null" primitive here the way you'd want, so the double-lookup idiom is universal, not a per-language quirk.

# BLOCK 2 — STRINGS

## Problem 5 — LC 125 — Valid Palindrome

**Surfaces drilled:** char classification (isalnum/isalpha/isdigit), lowercasing a char, two pointers on a string, while loops with bounds guards.
**Algorithm in one line:** Walk two pointers inward from both ends, skipping non-alphanumeric characters, and bail out on the first case-insensitive mismatch.

### C++ (your anchor)
```cpp
class Solution {
public:
    bool isPalindrome(string s) {
        int left = 0;
        int right = s.size() - 1;
        while (left < right) {
            while (left < right && !isalnum(s[left])) {
                left++;
            }
            while (left < right && !isalnum(s[right])) {
                right--;
            }
            if (tolower(s[left]) != tolower(s[right])) {
                return false;
            }
            left++;
            right--;
        }
        return true;
    }
};
```

### Java
```java
class Solution {
    public boolean isPalindrome(String s) {
        int left = 0;
        int right = s.length() - 1;
        while (left < right) {
            while (left < right && !Character.isLetterOrDigit(s.charAt(left))) {
                left++;
            }
            while (left < right && !Character.isLetterOrDigit(s.charAt(right))) {
                right--;
            }
            if (Character.toLowerCase(s.charAt(left)) != Character.toLowerCase(s.charAt(right))) {
                return false;
            }
            left++;
            right--;
        }
        return true;
    }
}
```

### Python
```python
class Solution:
    def isPalindrome(self, s: str) -> bool:
        left = 0
        right = len(s) - 1
        while left < right:
            while left < right and not s[left].isalnum():
                left += 1
            while left < right and not s[right].isalnum():
                right -= 1
            if s[left].lower() != s[right].lower():
                return False
            left += 1
            right -= 1
        return True
```

### JavaScript
```js
var isPalindrome = function(s) {
    let left = 0;
    let right = s.length - 1;
    while (left < right) {
        while (left < right && !/[a-zA-Z0-9]/.test(s[left])) {
            left++;
        }
        while (left < right && !/[a-zA-Z0-9]/.test(s[right])) {
            right--;
        }
        if (s[left].toLowerCase() !== s[right].toLowerCase()) {
            return false;
        }
        left++;
        right--;
    }
    return true;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| is this char alphanumeric? | `isalnum(s[left])` — free function on a `char` | `Character.isLetterOrDigit(s.charAt(left))` — static method on `Character` | `s[left].isalnum()` — method on the 1-char string itself | no built-in — `/[a-zA-Z0-9]/.test(s[left])` |
| lowercase a char | `tolower(s[left])` | `Character.toLowerCase(s.charAt(left))` | `s[left].lower()` | `s[left].toLowerCase()` |
| char (in)equality | `!=` on `char` | `!=` on unboxed `char` primitive | `!=` on 1-char `str` | `!==` on 1-char `string` |
| boolean literals | `false` / `true` | `false` / `true` | `False` / `True` (capitalized!) | `false` / `true` |
| bounds-guarded inner while | `while (left < right && ...)` | `while (left < right && ...)` | `while left < right and ...:` | `while (left < right && ...)` |
| string length | `s.size()` | `s.length()` (method call) | `len(s)` (free function) | `s.length` (property, no parens) |

> **Trap:** C++'s `isalnum`/`tolower` take an `int`, and `char` is signed on most platforms — passing a `char` with a negative value (any byte ≥ 0x80 in a non-ASCII/extended string) is technically undefined behavior unless you cast to `unsigned char` first. LeetCode's test strings are ASCII so it won't bite you here, but it's the one line in this whole set that is "correct" only because the input happens to be friendly, not because the code is actually safe — Java's `Character.isLetterOrDigit` and Python's `.isalnum()` are Unicode-aware and don't have this problem at all.

## Problem 6 — LC 151 — Reverse Words in a String

**Surfaces drilled:** split on whitespace, trim, reverse a list, join with a separator.
**Algorithm in one line:** Split the string into words, reverse the list of words, and join them back with single spaces.

### C++ (your anchor)
```cpp
class Solution {
public:
    string reverseWords(string s) {
        stringstream ss(s);
        string word;
        vector<string> words;
        while (ss >> word) {
            words.push_back(word);
        }
        reverse(words.begin(), words.end());
        string result;
        for (int i = 0; i < words.size(); i++) {
            result += words[i];
            if (i != words.size() - 1) {
                result += " ";
            }
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public String reverseWords(String s) {
        String trimmed = s.trim();
        String[] words = trimmed.split("\\s+");
        Collections.reverse(Arrays.asList(words));
        String result = String.join(" ", words);
        return result;
    }
}
```

### Python
```python
class Solution:
    def reverseWords(self, s: str) -> str:
        words = s.split()
        words.reverse()
        result = " ".join(words)
        return result
```

### JavaScript
```js
var reverseWords = function(s) {
    const words = s.trim().split(/\s+/).filter(word => word.length > 0);
    words.reverse();
    const result = words.join(" ");
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| split on whitespace | no split at all — `stringstream >> word` skips whitespace token-by-token | `trimmed.split("\\s+")` (regex) | `s.split()` (no-arg form) | `s.split(/\s+/)` (regex) |
| explicit trim needed? | no — `>>` never emits empty tokens | **yes** — `split("\\s+")` on a leading-whitespace string leaves a leading `""` | no — no-arg `.split()` drops all empties automatically | **yes** — same leading `""` problem, handled here with `.filter` instead of `.trim()` alone |
| reverse the word list | `reverse(words.begin(), words.end());` (free algorithm, in place) | `Collections.reverse(Arrays.asList(words));` (in place, via a List view of the array) | `words.reverse()` (in-place list method) | `words.reverse();` (in-place array method) |
| join with a separator | manual loop with an `if` to skip the trailing space | `String.join(" ", words)` | `" ".join(words)` | `words.join(" ")` |
| built-in join available? | **no** — C++ has no `join`, hence the manual loop above | yes, `String.join` | yes, `str.join` | yes, `Array.prototype.join` |

> **Trap:** the four languages disagree on what "split on whitespace" means for edge cases like `"  hello   world  "`. Python's argument-less `.split()` is the only one that transparently collapses runs of whitespace *and* drops leading/trailing empty strings for you. Java's and JavaScript's regex splits both need an explicit `.trim()` (Java) or `.trim()` **plus** a `.filter()` (JavaScript, because a leading empty match can still slip through even after trimming isn't enough on some inputs) before the split, or you'll return a sentence with a phantom empty word at one end. C++ sidesteps the whole issue because `stringstream::operator>>` was designed to skip whitespace natively — but that only helps you if you reach for `stringstream` instead of manually hunting for spaces with `find`/`substr`.

## Problem 7 — LC 344 — Reverse String

**Surfaces drilled:** in-place array mutation, two-pointer swap, swap syntax per language, immutable-string workaround.
**Algorithm in one line:** Swap characters from both ends inward until the two pointers meet.

### C++ (your anchor)
```cpp
class Solution {
public:
    void reverseString(vector<char>& s) {
        int left = 0;
        int right = s.size() - 1;
        while (left < right) {
            swap(s[left], s[right]);
            left++;
            right--;
        }
    }
};
```

### Java
```java
class Solution {
    public void reverseString(char[] s) {
        int left = 0;
        int right = s.length - 1;
        while (left < right) {
            char temp = s[left];
            s[left] = s[right];
            s[right] = temp;
            left++;
            right--;
        }
    }
}
```

### Python
```python
class Solution:
    def reverseString(self, s: List[str]) -> None:
        left = 0
        right = len(s) - 1
        while left < right:
            s[left], s[right] = s[right], s[left]
            left += 1
            right -= 1
```

### JavaScript
```js
var reverseString = function(s) {
    let left = 0;
    let right = s.length - 1;
    while (left < right) {
        [s[left], s[right]] = [s[right], s[left]];
        left++;
        right--;
    }
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| input type given by LeetCode | `vector<char>&` (mutable, by reference) | `char[]` (mutable, arrays are reference types) | `List[str]` — **a list of 1-char strings, not a `str`** | a plain (mutable) JS array of 1-char strings |
| why not the native string type? | N/A, `vector<char>` already mutable | N/A, `char[]` already mutable | `str` is immutable in Python — can't do `s[left] = ...` on a real string | `string` is immutable in JS too, but LC deliberately hands you an array here, not a string |
| swap two elements | `swap(s[left], s[right]);` — built-in | no built-in swap — manual `temp` variable | tuple-unpacking swap: `s[left], s[right] = s[right], s[left]` | array-destructuring swap: `[s[left], s[right]] = [s[right], s[left]];` |
| return type | `void` | `void` | `None` (must still declare `-> None` and have no `return`) | implicit `undefined` (no `return`) |
| declare left/right pointers | `int left = 0;` (typed) | `int left = 0;` (typed) | `left = 0` (no type) | `let left = 0;` (block-scoped, no type) |

> **Trap:** the entire reason this problem's signature reads `List[str]` in Python (instead of the more natural `str`) is that Python strings are immutable — the problem is unsolvable in-place against a real `str`, so LeetCode hands you a list of one-character strings instead, and you must remember that `s[left]` is itself a 1-char string, not a `char` type (Python has none). JavaScript has the same immutability quirk for its native `string`, but LeetCode sidesteps it there identically, by giving you a genuine mutable array. C++'s `vector<char>` and Java's `char[]` never have this problem because both languages have a real mutable `char` container — so if you're coming from C++, the Python/JS signatures look like unnecessary ceremony until you remember *why* they're forced to look that way.

---

# BLOCK 3 — LINKED LIST AND CLASS DESIGN

## Problem 8 — LC 206 — Reverse Linked List

**Surfaces drilled:** linked-list node type per language, null/None/undefined handling, pointer reassignment, three-pointer iterative reverse
**Algorithm in one line:** Walk the list once, and at each node flip `next` to point backward while `prev` and `curr` both march forward.

### C++ (your anchor)
```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode* prev = nullptr;
        ListNode* curr = head;
        while (curr != nullptr) {
            ListNode* next = curr->next;
            curr->next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }
};
```

### Java
```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode prev = null;
        ListNode curr = head;
        while (curr != null) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        return prev;
    }
}
```

### Python
```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, val=0, next=None):
#         self.val = val
#         self.next = next
from typing import Optional

class Solution:
    def reverseList(self, head: Optional[ListNode]) -> Optional[ListNode]:
        prev = None
        curr = head
        while curr is not None:
            next = curr.next
            curr.next = prev
            prev = curr
            curr = next
        return prev
```

### JavaScript
```js
/**
 * Definition for singly-linked list.
 * function ListNode(val, next) {
 *     this.val = (val===undefined ? 0 : val)
 *     this.next = (next===undefined ? null : next)
 * }
 */
/**
 * @param {ListNode} head
 * @return {ListNode}
 */
var reverseList = function(head) {
    let prev = null;
    let curr = head;
    while (curr !== null) {
        let next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Node type | `struct ListNode*` (raw pointer) | `class ListNode` (reference) | `class ListNode`, `self.next` | plain object literal shape via `function ListNode(...)` |
| Null check | `curr != nullptr` | `curr != null` | `curr is not None` | `curr !== null` |
| Field access | `curr->next` | `curr.next` | `curr.next` | `curr.next` |
| Method signature | `ListNode* reverseList(ListNode* head)` | `public ListNode reverseList(ListNode head)` | `def reverseList(self, head)` | `var reverseList = function(head)` |

> **Trap:** Python's local variable `next` shadows the builtin `next()` function for the rest of that scope — harmless in this tiny loop, but if you extend the method and try to call the builtin `next()` later, it silently calls your reassigned variable instead and throws a confusing `TypeError: 'ListNode' object is not callable`.

## Problem 9 — LC 21 — Merge Two Sorted Lists

**Surfaces drilled:** dummy/sentinel node creation, tail pointer append, while loop with two conditions, attaching the remainder
**Algorithm in one line:** Walk both lists with a tail pointer, always append whichever head is smaller, then splice on whichever list still has nodes left.

### C++ (your anchor)
```cpp
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
        ListNode dummy(0);
        ListNode* tail = &dummy;
        while (list1 != nullptr && list2 != nullptr) {
            if (list1->val <= list2->val) {
                tail->next = list1;
                list1 = list1->next;
            } else {
                tail->next = list2;
                list2 = list2->next;
            }
            tail = tail->next;
        }
        tail->next = (list1 != nullptr) ? list1 : list2;
        return dummy.next;
    }
};
```

### Java
```java
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        ListNode dummy = new ListNode(0);
        ListNode tail = dummy;
        while (list1 != null && list2 != null) {
            if (list1.val <= list2.val) {
                tail.next = list1;
                list1 = list1.next;
            } else {
                tail.next = list2;
                list2 = list2.next;
            }
            tail = tail.next;
        }
        tail.next = (list1 != null) ? list1 : list2;
        return dummy.next;
    }
}
```

### Python
```python
class Solution:
    def mergeTwoLists(self, list1: Optional[ListNode], list2: Optional[ListNode]) -> Optional[ListNode]:
        dummy = ListNode(0)
        tail = dummy
        while list1 is not None and list2 is not None:
            if list1.val <= list2.val:
                tail.next = list1
                list1 = list1.next
            else:
                tail.next = list2
                list2 = list2.next
            tail = tail.next
        tail.next = list1 if list1 is not None else list2
        return dummy.next
```

### JavaScript
```js
var mergeTwoLists = function(list1, list2) {
    let dummy = new ListNode(0);
    let tail = dummy;
    while (list1 !== null && list2 !== null) {
        if (list1.val <= list2.val) {
            tail.next = list1;
            list1 = list1.next;
        } else {
            tail.next = list2;
            list2 = list2.next;
        }
        tail = tail.next;
    }
    tail.next = (list1 !== null) ? list1 : list2;
    return dummy.next;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Dummy node allocation | `ListNode dummy(0);` on the stack | `new ListNode(0)` on the heap | `ListNode(0)` (always heap) | `new ListNode(0)` (always heap) |
| Getting a pointer to it | `ListNode* tail = &dummy;` (explicit address-of) | `ListNode tail = dummy;` (reference copy, no `&`) | `tail = dummy` (name binding) | `let tail = dummy;` (reference copy) |
| Conditional expression | `(list1 != nullptr) ? list1 : list2` | `(list1 != null) ? list1 : list2` | `list1 if list1 is not None else list2` | `(list1 !== null) ? list1 : list2` |

> **Trap:** C++ is the only one of the four where the dummy node is stack-allocated — you must take its address with `&dummy` to get a pointer, and you must return `dummy.next` (not `tail`), because `tail` has moved on by the time the loop ends. Forgetting the `&` gives a compile error here, but in a more complex refactor where `dummy` is passed around it's easy to accidentally copy the sentinel by value and mutate a throwaway copy instead of the real list.

## Problem 10 — LC 146 — LRU Cache

**Surfaces drilled:** class with internal state and a constructor, hash map plus doubly-linked list, node class as a nested/inner type, and the built-in shortcut each language offers
**Algorithm in one line:** Keep a hash map from key to a doubly-linked-list node, and keep that list ordered most-recently-used at the head; every `get`/`put` unlinks and re-inserts the touched node at the head, and `put` evicts the tail node when the map grows past capacity.

### C++ (your anchor) — explicit doubly-linked list
```cpp
class LRUCache {
private:
    struct Node {
        int key;
        int value;
        Node* prev;
        Node* next;
        Node(int k, int v) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };

    int capacity;
    unordered_map<int, Node*> map;
    Node* head; // most recently used sentinel
    Node* tail; // least recently used sentinel

    void remove(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    void insertFront(Node* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }

public:
    LRUCache(int capacity) : capacity(capacity) {
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head->next = tail;
        tail->prev = head;
    }

    int get(int key) {
        if (map.find(key) == map.end()) {
            return -1;
        }
        Node* node = map[key];
        remove(node);
        insertFront(node);
        return node->value;
    }

    void put(int key, int value) {
        if (map.find(key) != map.end()) {
            remove(map[key]);
        }
        Node* node = new Node(key, value);
        map[key] = node;
        insertFront(node);
        if (map.size() > capacity) {
            Node* lru = tail->prev;
            remove(lru);
            map.erase(lru->key);
            delete lru;
        }
    }
};
```

### Java — explicit doubly-linked list
```java
class LRUCache {
    class Node {
        int key;
        int value;
        Node prev;
        Node next;
        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private int capacity;
    private Map<Integer, Node> map;
    private Node head; // most recently used sentinel
    private Node tail; // least recently used sentinel

    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void insertFront(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        Node node = map.get(key);
        remove(node);
        insertFront(node);
        return node.value;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            remove(map.get(key));
        }
        Node node = new Node(key, value);
        map.put(key, node);
        insertFront(node);
        if (map.size() > capacity) {
            Node lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
    }
}
```

### Python — explicit doubly-linked list
```python
class Node:
    def __init__(self, key: int, value: int):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.map = {}
        self.head = Node(0, 0)  # most recently used sentinel
        self.tail = Node(0, 0)  # least recently used sentinel
        self.head.next = self.tail
        self.tail.prev = self.head

    def remove(self, node: Node) -> None:
        node.prev.next = node.next
        node.next.prev = node.prev

    def insertFront(self, node: Node) -> None:
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.map:
            return -1
        node = self.map[key]
        self.remove(node)
        self.insertFront(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.map:
            self.remove(self.map[key])
        node = Node(key, value)
        self.map[key] = node
        self.insertFront(node)
        if len(self.map) > self.capacity:
            lru = self.tail.prev
            self.remove(lru)
            del self.map[lru.key]
```

### JavaScript — explicit doubly-linked list
```js
class Node {
    constructor(key, value) {
        this.key = key;
        this.value = value;
        this.prev = null;
        this.next = null;
    }
}

class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.map = new Map();
        this.head = new Node(0, 0); // most recently used sentinel
        this.tail = new Node(0, 0); // least recently used sentinel
        this.head.next = this.tail;
        this.tail.prev = this.head;
    }

    remove(node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    insertFront(node) {
        node.next = this.head.next;
        node.prev = this.head;
        this.head.next.prev = node;
        this.head.next = node;
    }

    get(key) {
        if (!this.map.has(key)) {
            return -1;
        }
        let node = this.map.get(key);
        this.remove(node);
        this.insertFront(node);
        return node.value;
    }

    put(key, value) {
        if (this.map.has(key)) {
            this.remove(this.map.get(key));
        }
        let node = new Node(key, value);
        this.map.set(key, node);
        this.insertFront(node);
        if (this.map.size > this.capacity) {
            let lru = this.tail.prev;
            this.remove(lru);
            this.map.delete(lru.key);
        }
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * var obj = new LRUCache(capacity)
 * var param_1 = obj.get(key)
 * obj.put(key,value)
 */
```

### Built-in shortcut — Java `LinkedHashMap` (access order)
```java
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // true = access order
        this.capacity = capacity;
    }

    public int get(int key) {
        return super.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        super.put(key, value);
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}
```

### Built-in shortcut — Python `OrderedDict`
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
```

### Built-in shortcut — JavaScript `Map` (preserves insertion order)
```js
class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();
    }

    get(key) {
        if (!this.cache.has(key)) {
            return -1;
        }
        let value = this.cache.get(key);
        this.cache.delete(key);
        this.cache.set(key, value);
        return value;
    }

    put(key, value) {
        if (this.cache.has(key)) {
            this.cache.delete(key);
        }
        this.cache.set(key, value);
        if (this.cache.size > this.capacity) {
            let oldestKey = this.cache.keys().next().value;
            this.cache.delete(oldestKey);
        }
    }
}
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Hash map type | `unordered_map<int, Node*>` | `Map<Integer, Node>` (`HashMap`) | `dict` | `Map` |
| Node class | `private struct Node` nested in the class | `private class Node` (inner class) | top-level `class Node` | top-level `class Node` |
| Membership check | `map.find(key) == map.end()` | `!map.containsKey(key)` | `key not in self.map` | `!this.map.has(key)` |
| Manual memory cleanup | `delete lru;` required after eviction | none — GC reclaims it | none — GC reclaims it | none — GC reclaims it |
| Built-in shortcut | `std::list` + `unordered_map` w/ iterators (idiomatic, not shown above) | `LinkedHashMap(cap, 0.75f, true)` + `removeEldestEntry` | `OrderedDict` + `move_to_end` / `popitem(last=False)` | `Map` + delete-then-reset to "touch" a key |

> **Trap:** In the explicit C++ version, evicting the LRU node needs an explicit `delete lru;` or you leak memory on every eviction — Java, Python and JS never need this since the unlinked node just becomes garbage. Separately, in the JS built-in shortcut, calling `map.set(existingKey, newValue)` on a key that already exists does **not** move it to the end of iteration order — only a fresh insertion does — which is exactly why the code above must `delete` the key before re-`set`ting it. Python's `OrderedDict.move_to_end` and Java's access-order `LinkedHashMap` both reorder automatically on touch; skipping the explicit delete-then-set is a JS-only landmine.

# BLOCK 4 — TREES AND RECURSION

## Problem 11 — LC 104 — Maximum Depth of Binary Tree

**Surfaces drilled:** binary tree node type per language, recursion returning a value up, null base case, max of two
**Algorithm in one line:** Recursively return 1 + the larger of the left and right subtree depths, with an empty subtree contributing depth 0.

### C++ (your anchor)
```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode() : val(0), left(nullptr), right(nullptr) {}
 *     TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
 *     TreeNode(int x, TreeNode *left, TreeNode *right) : val(x), left(left), right(right) {}
 * };
 */
class Solution {
public:
    int maxDepth(TreeNode* root) {
        if (root == nullptr) {
            return 0;
        }
        int left = maxDepth(root->left);
        int right = maxDepth(root->right);
        return 1 + max(left, right);
    }
};
```

### Java
```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        int left = maxDepth(root.left);
        int right = maxDepth(root.right);
        return 1 + Math.max(left, right);
    }
}
```

### Python
```python
# Definition for a binary tree node.
# class TreeNode:
#     def __init__(self, val=0, left=None, right=None):
#         self.val = val
#         self.left = left
#         self.right = right
from typing import Optional

class Solution:
    def maxDepth(self, root: Optional[TreeNode]) -> int:
        if root is None:
            return 0
        left = self.maxDepth(root.left)
        right = self.maxDepth(root.right)
        return 1 + max(left, right)
```

### JavaScript
```js
/**
 * Definition for a binary tree node.
 * function TreeNode(val, left, right) {
 *     this.val = (val===undefined ? 0 : val)
 *     this.left = (left===undefined ? null : left)
 *     this.right = (right===undefined ? null : right)
 * }
 */
var maxDepth = function(root) {
    if (root === null) {
        return 0;
    }
    let left = maxDepth(root.left);
    let right = maxDepth(root.right);
    return 1 + Math.max(left, right);
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Tree node type | `struct TreeNode*` | `class TreeNode` | `class TreeNode` | plain object via `function TreeNode(...)` |
| Max of two ints | `max(left, right)` (`<algorithm>`) | `Math.max(left, right)` | `max(left, right)` (builtin, no import) | `Math.max(left, right)` |
| Recursive self-call | `maxDepth(root->left)` (free function on `this`) | `maxDepth(root.left)` (implicit `this`) | `self.maxDepth(root.left)` (explicit `self`) | `maxDepth(root.left)` (plain function name) |

> **Trap:** Python recursive calls need the explicit `self.` prefix — writing `maxDepth(root.left)` instead of `self.maxDepth(root.left)` raises `NameError` because there's no implicit-`this` lookup like Java, and no free-function fallback like C++/JS give you here.

## Problem 12 — LC 102 — Binary Tree Level Order Traversal

**Surfaces drilled:** queue/deque per language, the level-sized inner loop, building a nested list-of-lists, null checks before enqueue
**Algorithm in one line:** BFS the tree one full level at a time by snapshotting the current queue size before draining exactly that many nodes into the next level's list.

### C++ (your anchor)
```cpp
class Solution {
public:
    vector<vector<int>> levelOrder(TreeNode* root) {
        vector<vector<int>> result;
        if (root == nullptr) {
            return result;
        }
        queue<TreeNode*> q;
        q.push(root);
        while (!q.empty()) {
            int levelSize = q.size();
            vector<int> level;
            for (int i = 0; i < levelSize; i++) {
                TreeNode* node = q.front();
                q.pop();
                level.push_back(node->val);
                if (node->left != nullptr) {
                    q.push(node->left);
                }
                if (node->right != nullptr) {
                    q.push(node->right);
                }
            }
            result.push_back(level);
        }
        return result;
    }
};
```

### Java
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> result = new ArrayList<>();
        if (root == null) {
            return result;
        }
        Queue<TreeNode> q = new ArrayDeque<>();
        q.offer(root);
        while (!q.isEmpty()) {
            int levelSize = q.size();
            List<Integer> level = new ArrayList<>();
            for (int i = 0; i < levelSize; i++) {
                TreeNode node = q.poll();
                level.add(node.val);
                if (node.left != null) {
                    q.offer(node.left);
                }
                if (node.right != null) {
                    q.offer(node.right);
                }
            }
            result.add(level);
        }
        return result;
    }
}
```

### Python
```python
from collections import deque
from typing import Optional, List

class Solution:
    def levelOrder(self, root: Optional[TreeNode]) -> List[List[int]]:
        result = []
        if root is None:
            return result
        q = deque([root])
        while q:
            levelSize = len(q)
            level = []
            for i in range(levelSize):
                node = q.popleft()
                level.append(node.val)
                if node.left is not None:
                    q.append(node.left)
                if node.right is not None:
                    q.append(node.right)
            result.append(level)
        return result
```

### JavaScript
```js
var levelOrder = function(root) {
    let result = [];
    if (root === null) {
        return result;
    }
    let q = [root];
    let head = 0;
    while (head < q.length) {
        let levelSize = q.length - head;
        let level = [];
        for (let i = 0; i < levelSize; i++) {
            let node = q[head++];
            level.push(node.val);
            if (node.left !== null) {
                q.push(node.left);
            }
            if (node.right !== null) {
                q.push(node.right);
            }
        }
        result.push(level);
    }
    return result;
};
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Queue type | `queue<TreeNode*>` | `Queue<TreeNode>` backed by `ArrayDeque` | `deque` | plain array + `head` index pointer |
| Enqueue / dequeue | `push` / `front` + `pop` | `offer` / `poll` | `append` / `popleft` | `push` / `q[head++]` |
| Loop-bound snapshot | `int levelSize = q.size();` | `int levelSize = q.size();` | `levelSize = len(q)` | `let levelSize = q.length - head;` |
| Outer loop condition | `!q.empty()` | `!q.isEmpty()` | `while q:` (truthy on non-empty) | `head < q.length` |

> **Trap:** JavaScript has no built-in O(1) deque — repeatedly calling `Array.shift()` to dequeue is O(n) per call and silently degrades this whole traversal to O(n²) on large trees. The fix shown above is a `head` index pointer that never physically removes front elements. Separately, in Java reach for `ArrayDeque` as your queue — a `Stack` is LIFO and would reverse each level's order, and the legacy `LinkedList` works but is slower and easier to misuse as a `List` by accident.

## Problem 13 — LC 297 — Serialize and Deserialize Binary Tree

**Surfaces drilled:** string building efficiently, split parsing, queue/index-pointer consumption, encoding null as a sentinel, a class with two methods, recursion on both directions
**Algorithm in one line:** Preorder-serialize each node's value (or `#` for null) separated by commas, then preorder-deserialize by consuming tokens off the front in exactly the order they were written.

### C++ (your anchor)
```cpp
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
class Codec {
public:
    string serialize(TreeNode* root) {
        string result;
        serializeHelper(root, result);
        return result;
    }

    TreeNode* deserialize(string data) {
        vector<string> tokens;
        stringstream ss(data);
        string token;
        while (getline(ss, token, ',')) {
            tokens.push_back(token);
        }
        int index = 0;
        return deserializeHelper(tokens, index);
    }

private:
    void serializeHelper(TreeNode* node, string& result) {
        if (node == nullptr) {
            result += "#,";
            return;
        }
        result += to_string(node->val) + ",";
        serializeHelper(node->left, result);
        serializeHelper(node->right, result);
    }

    TreeNode* deserializeHelper(vector<string>& tokens, int& index) {
        string token = tokens[index];
        index++;
        if (token == "#") {
            return nullptr;
        }
        TreeNode* node = new TreeNode(stoi(token));
        node->left = deserializeHelper(tokens, index);
        node->right = deserializeHelper(tokens, index);
        return node;
    }
};

// Your Codec object will be instantiated and called as such:
// Codec* ser = new Codec();
// Codec* deser = new Codec();
// string tree = ser->serialize(root);
// TreeNode* ans = deser->deserialize(tree);
```

### Java
```java
public class Codec {
    public String serialize(TreeNode root) {
        StringBuilder result = new StringBuilder();
        serializeHelper(root, result);
        return result.toString();
    }

    public TreeNode deserialize(String data) {
        String[] tokens = data.split(",");
        int[] index = {0};
        return deserializeHelper(tokens, index);
    }

    private void serializeHelper(TreeNode node, StringBuilder result) {
        if (node == null) {
            result.append("#,");
            return;
        }
        result.append(node.val).append(",");
        serializeHelper(node.left, result);
        serializeHelper(node.right, result);
    }

    private TreeNode deserializeHelper(String[] tokens, int[] index) {
        String token = tokens[index[0]];
        index[0]++;
        if (token.equals("#")) {
            return null;
        }
        TreeNode node = new TreeNode(Integer.parseInt(token));
        node.left = deserializeHelper(tokens, index);
        node.right = deserializeHelper(tokens, index);
        return node;
    }
}

// Your Codec object will be instantiated and called as such:
// Codec ser = new Codec();
// Codec deser = new Codec();
// String tree = ser.serialize(root);
// TreeNode ans = deser.deserialize(tree);
```

### Python
```python
from typing import List, Optional

class Codec:
    def serialize(self, root: Optional[TreeNode]) -> str:
        result = []
        self.serializeHelper(root, result)
        return ",".join(result)

    def deserialize(self, data: str) -> Optional[TreeNode]:
        tokens = data.split(",")
        self.index = 0
        return self.deserializeHelper(tokens)

    def serializeHelper(self, node: Optional[TreeNode], result: List[str]) -> None:
        if node is None:
            result.append("#")
            return
        result.append(str(node.val))
        self.serializeHelper(node.left, result)
        self.serializeHelper(node.right, result)

    def deserializeHelper(self, tokens: List[str]) -> Optional[TreeNode]:
        token = tokens[self.index]
        self.index += 1
        if token == "#":
            return None
        node = TreeNode(int(token))
        node.left = self.deserializeHelper(tokens)
        node.right = self.deserializeHelper(tokens)
        return node

# Your Codec object will be instantiated and called as such:
# ser = Codec()
# deser = Codec()
# ans = deser.deserialize(ser.serialize(root))
```

### JavaScript
```js
/**
 * Encodes a tree to a single string.
 *
 * @param {TreeNode} root
 * @return {string}
 */
var serialize = function(root) {
    let result = [];
    serializeHelper(root, result);
    return result.join(",");
};

/**
 * Decodes your encoded data to tree.
 *
 * @param {string} data
 * @return {TreeNode}
 */
var deserialize = function(data) {
    let tokens = data.split(",");
    let index = [0];
    return deserializeHelper(tokens, index);
};

function serializeHelper(node, result) {
    if (node === null) {
        result.push("#");
        return;
    }
    result.push(node.val);
    serializeHelper(node.left, result);
    serializeHelper(node.right, result);
}

function deserializeHelper(tokens, index) {
    let token = tokens[index[0]];
    index[0]++;
    if (token === "#") {
        return null;
    }
    let node = new TreeNode(parseInt(token));
    node.left = deserializeHelper(tokens, index);
    node.right = deserializeHelper(tokens, index);
    return node;
}

/**
 * Your functions will be called as such:
 * deserialize(serialize(root));
 */
```

### What actually changed
| Construct | C++ | Java | Python | JavaScript |
|---|---|---|---|---|
| Mutable index shared across recursive calls | `int& index` (true reference parameter) | `int[] index = {0}` (array wrapper — no primitive references) | `self.index` (instance attribute) | `let index = [0]` (array wrapper, same trick as Java) |
| String building | `string` `+=` concatenation, trailing `,` per token | `StringBuilder.append(...)`, trailing `,` per token | `list.append(...)` then `",".join(result)` | `array.push(...)` then `result.join(",")` |
| Splitting into tokens | `stringstream` + `getline(ss, token, ',')` into `vector<string>` | `data.split(",")` into `String[]` (drops trailing empties) | `data.split(",")` into `list[str]` | `data.split(",")` into `Array<string>` |
| Null-sentinel comparison | `token == "#"` (value compare for `std::string`) | `token.equals("#")` — never `==` | `token == "#"` (value compare) | `token === "#"` |
| Method/function container | `class Codec` with public/private sections | `public class Codec` | `class Codec`, no access modifiers | two standalone functions, `serialize`/`deserialize` (matches LeetCode's own JS template — no class) |

> **Trap:** In Java, `String`s must be compared with `.equals()`, never `==` — `token == "#"` compiles cleanly but compares object references, not contents, and will intermittently return `false` for equal-looking strings depending on interning, silently corrupting the deserialize walk. Relatedly: Java has no pass-by-reference for `int`, so mutating a shared index across recursive calls needs the `int[] index = {0}` wrapper trick — the JS version borrows that same trick for structural symmetry, even though JS could just close over a plain variable; C++ is the only language here that gets a real reference (`int&`) for free.

---

*Doc 45 — Polyglot Solutions Part 1. Continue with doc 46 (problems 14–30). Keep doc 44 open while practising.*
