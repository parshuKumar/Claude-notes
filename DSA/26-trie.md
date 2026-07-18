# PATTERN 26 — TRIE
## Prefix Trees, Wildcard DFS, Bit Tries for XOR — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**Why a Trie exists when a hash set/map already answers "is this string present?"**

A `unordered_set<string>` answers membership in O(L) (L = string length, via hashing the whole string). That's the same asymptotic cost as a Trie lookup. So why does the Trie exist at all?

Because a hash set answers exactly one question well: *"is this exact string in the set?"* It cannot answer, in better than O(N·L):

- **"How many stored strings start with this prefix?"** — a hash set must scan every entry and check `substr` equality. A Trie walks the prefix once, O(L), and reads a counter at the final node.
- **"Is there a stored string matching this pattern with wildcards (`.`)?"** — a hash set has no notion of "partial match, try all letters here." A Trie's tree structure makes this a natural DFS branch.
- **"What is the maximum XOR I can form between X and any stored number?"** — this is the one that surprises people. Represent numbers as 32-bit binary strings and insert them into a **bit trie** (2 children per node instead of 26). Then a greedy MSB-first walk finds the best XOR partner in O(32) instead of O(N) per query. A hash set literally cannot do this — XOR has no useful hash-based shortcut.
- **"Replace every word in a sentence with its shortest dictionary root"** — this is a shared-prefix compression problem. A Trie lets many words share storage for their common prefixes; a hash set duplicates every prefix's cost per lookup.

**The honest boundary:** if your problem only ever asks "is X present" or "is X present with an exact key," a hash set/map is simpler, uses less memory (no pointer-per-character overhead), and you should use it. Reach for a Trie exactly when the query is about **prefixes**, **wildcards**, or **bitwise structure** — those are the three doors a Trie opens that a hash table cannot.

**The 3-minute build-from-scratch bar:**

Every grandmaster-level candidate can write this cold, no hesitation:

```cpp
struct TrieNode {
    TrieNode* children[26];
    bool isEnd;
    TrieNode() : isEnd(false) {
        for (int i = 0; i < 26; i++) children[i] = nullptr;
    }
};
```

...plus `insert`, `search`, `startsWith` — about 25 lines total. If you have to think about the node shape, you are not ready for the harder variants (wildcard DFS, bit trie, Word Search II pruning) that this pattern actually tests at the 1900+ level.

---

## SECTION 2 — THE TRIE TAXONOMY

### Type 1: Array-of-26 Node (lowercase-letter Tries)
Node: `TrieNode* children[26]` indexed by `c - 'a'`.
Pros: O(1) child access, cache-friendly, no hashing overhead.
Cons: 26 pointers per node = 208 bytes (64-bit pointers) even if only 1–2 children are used — wasteful for sparse alphabets or Unicode.
Use when: alphabet is small and fixed (lowercase English is the overwhelming majority of interview problems).

### Type 2: Map/Hash Node (`unordered_map<char, TrieNode*>`)
Node: a map from character to child pointer.
Pros: memory-proportional to actual branching factor; works for any alphabet (Unicode, mixed case, digits).
Cons: hashing overhead per step; ~3-5x slower in practice than array indexing.
Use when: alphabet is large/unknown, or the trie is extremely sparse (few words, long strings).

### Type 3: Insert / Search / StartsWith (the base API)
`insert(word)`: walk/create nodes for each character, mark the final node `isEnd = true`.
`search(word)`: walk nodes for each character; return `false` immediately if a child is missing; at the end, return `node->isEnd`.
`startsWith(prefix)`: same walk as search, but return `true` as soon as the walk completes (don't check `isEnd`).
Problems: Implement Trie (LC 208).

### Type 4: DFS With Wildcard
State: recursive `dfs(index, node)`. On a literal character, descend one child. On `.` (or `*`), fan out into **all 26** children and OR the results together.
Problems: Add and Search Word (LC 211).

### Type 5: Trie + Board DFS (word search on a grid)
Build one Trie from the whole word list, then DFS the grid ONCE, following Trie edges instead of trying every word independently against every starting cell. Requires **pruning** (delete exhausted branches) to avoid TLE — this is the single biggest complexity trap in the entire pattern.
Problems: Word Search II (LC 212).

### Type 6: Bit Trie for XOR
Node: `TrieNode* children[2]` (bit 0 / bit 1). Insert every number as a fixed-width binary string, **most-significant bit first**. To find the number that maximizes XOR with `x`, greedily try to walk into the *opposite* bit of `x` at every level (that bit contributes the most to the XOR result, since higher bits dominate the numeric value of an XOR).
Problems: Maximum XOR of Two Numbers (LC 421), Maximum XOR With an Element From Array (LC 1707).

### Type 7: Prefix Counting / Aggregation in Trie
Store extra payload at nodes (a count, a sum, a list of word-indices) rather than just a boolean. Every insertion propagates the payload delta along the insertion path, so a prefix query is a single O(L) walk to a node whose payload already holds the aggregated answer.
Problems: Map Sum Pairs (LC 677), Replace Words (LC 648, propagate nothing — just early-exit at first `isEnd`).

### Type 8: Reversed / Suffix Trie for Streaming
When queries ask "does any pattern end at the character I just received," insert words **reversed** and walk the query buffer **backward** from the most recent character. This turns a suffix-matching problem into a standard prefix-matching problem.
Problems: Stream of Characters (LC 1032).

### Type 9: Offline + Incremental Trie
When queries have a constraint like "only use numbers ≤ m," sort queries by `m` and sort the numbers once; then insert numbers into the trie incrementally as the query's threshold allows, and query the trie in its current (partial) state. This avoids rebuilding the trie per query.
Problems: Maximum XOR With an Element From Array (LC 1707).

---

## SECTION 3 — TEMPLATE 1: IMPLEMENT TRIE (LC 208, THE FOUNDATION)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 208 — Implement Trie (Prefix Tree)
// insert(word), search(word) [exact match], startsWith(prefix)
//
// Node shape: array-of-26 children + isEnd flag.
// insert:      walk word, create missing nodes, mark isEnd=true at the last node
// search:      walk word, fail fast on missing child, require isEnd=true at the end
// startsWith:  walk prefix, fail fast on missing child, do NOT require isEnd
// ─────────────────────────────────────────────────────────────────

class Trie {
private:
    struct TrieNode {
        TrieNode* children[26];
        bool isEnd;
        TrieNode() : isEnd(false) {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };
    TrieNode* root;

    // Shared walk used by both search() and startsWith()
    TrieNode* walk(const string& s) {
        TrieNode* cur = root;
        for (char c : s) {
            int idx = c - 'a';
            if (cur->children[idx] == nullptr) return nullptr;
            cur = cur->children[idx];
        }
        return cur;
    }

public:
    Trie() { root = new TrieNode(); }

    void insert(string word) {
        TrieNode* cur = root;
        for (char c : word) {
            int idx = c - 'a';
            if (cur->children[idx] == nullptr)
                cur->children[idx] = new TrieNode();
            cur = cur->children[idx];
        }
        cur->isEnd = true;
    }

    bool search(string word) {
        TrieNode* node = walk(word);
        return node != nullptr && node->isEnd;
    }

    bool startsWith(string prefix) {
        return walk(prefix) != nullptr;
    }
};
```

**FULL TRACE:**
```
insert("apple")
  a -> p -> p -> l -> e   (5 new nodes created), isEnd[e]=true

search("apple")  -> walk succeeds, node.isEnd=true  -> TRUE
search("app")    -> walk succeeds at node 'p'(2nd), node.isEnd=false -> FALSE
startsWith("app") -> walk succeeds -> TRUE (isEnd not checked)

insert("app")
  a -> p -> p             (all 3 nodes already exist, reused), isEnd[p(2nd)]=true

search("app")    -> walk succeeds, node.isEnd=true NOW -> TRUE
```

**Why `search` and `startsWith` share `walk`:** the only difference between them is whether the final node must be a completed word. Duplicating the walk loop is a common interview time-sink — factor it out immediately.

---

## SECTION 4 — TEMPLATE 2: ADD AND SEARCH WORD, WILDCARD DFS (LC 211)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 211 — Design Add and Search Words Data Structure
// addWord(word): same as Trie insert.
// search(word): word may contain '.' which matches ANY single letter.
//
// KEY IDEA: search becomes a DFS. On a literal char, descend exactly
// one child (like normal Trie search). On '.', try ALL 26 children
// and return true if ANY of them lead to a full match.
// ─────────────────────────────────────────────────────────────────

class WordDictionary {
private:
    struct TrieNode {
        TrieNode* children[26];
        bool isEnd;
        TrieNode() : isEnd(false) {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };
    TrieNode* root;

    bool dfs(const string& word, int idx, TrieNode* node) {
        if (node == nullptr) return false;          // dead branch
        if (idx == (int)word.size()) return node->isEnd; // consumed whole word

        char c = word[idx];
        if (c == '.') {
            for (int i = 0; i < 26; i++) {
                if (node->children[i] && dfs(word, idx + 1, node->children[i]))
                    return true;                     // one branch succeeded — done
            }
            return false;                            // all 26 branches failed
        } else {
            return dfs(word, idx + 1, node->children[c - 'a']);
        }
    }

public:
    WordDictionary() { root = new TrieNode(); }

    void addWord(string word) {
        TrieNode* cur = root;
        for (char c : word) {
            int idx = c - 'a';
            if (!cur->children[idx]) cur->children[idx] = new TrieNode();
            cur = cur->children[idx];
        }
        cur->isEnd = true;
    }

    bool search(string word) { return dfs(word, 0, root); }
};
```

**TRACE for addWord("bad"), addWord("dad"), addWord("mad"), then queries:**
```
Trie after inserts:
  root -> b -> a -> d* 
  root -> d -> a -> d*
  root -> m -> a -> d*
  (* = isEnd)

search("pad"):
  idx=0, c='p'. root->children['p'-'a'] = nullptr. dfs returns false. -> FALSE

search("bad"):
  idx=0 'b' -> node(b). idx=1 'a' -> node(ba). idx=2 'd' -> node(bad).
  idx=3 == size. return node(bad).isEnd = true. -> TRUE

search(".ad"):
  idx=0 '.': try all 26 children of root. Only 'b','d','m' exist.
    branch 'b': dfs(1, node(b)) -> 'a' matches -> dfs(2, node(ba))
                 -> 'd' matches -> dfs(3, node(bad)) -> isEnd=true -> TRUE
  First successful branch short-circuits the loop. -> TRUE

search("b.."):
  idx=0 'b' -> node(b). idx=1 '.': try all 26 children of node(b).
    Only 'a' exists -> dfs(2, node(ba)).
    idx=2 '.': try all 26 children of node(ba). Only 'd' exists -> dfs(3, node(bad)).
    idx=3==size, isEnd=true -> TRUE (bubbles back up through both '.' loops)
```

**Complexity:** worst case (query is all dots, e.g., `"...."`), the DFS explores every node at that depth — O(26^k) where k = number of dots. This is the accepted worst case for LC 211; the trie still massively prunes compared to comparing against every stored word individually when few dots are present.

---

## SECTION 5 — TEMPLATE 3: WORD SEARCH II (LC 212, TRIE + BACKTRACKING)

This is the hardest "core" problem in the pattern: combine a Trie (Type 5) with grid backtracking (Pattern 15), and add **pruning** or it times out.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 212 — Word Search II
// Given an m×n board of letters and a list of words, return all
// words that can be formed by a path of adjacent cells (4-directional,
// no cell reused within one word).
//
// NAIVE: run Word Search I (LC 79) once per word -> O(W * m*n*4^L). TLE.
// CORRECT: build ONE Trie of all words. DFS the board ONCE. At each
// board cell, only descend into the Trie edge matching board[r][c] —
// this explores all words SIMULTANEOUSLY and shares common prefixes.
// PRUNING: once a Trie leaf has no remaining children AND is not a
// word-end anymore (its word was already found), delete it from its
// parent. Without this, dense dictionaries with many short words
// still cause the DFS to revisit dead subtrees over and over -> TLE.
// ─────────────────────────────────────────────────────────────────

class Solution {
    struct TrieNode {
        TrieNode* children[26];
        string word;  // non-empty exactly at nodes that complete a dictionary word
        TrieNode() : word("") {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };

    TrieNode* buildTrie(vector<string>& words) {
        TrieNode* root = new TrieNode();
        for (string& w : words) {
            TrieNode* cur = root;
            for (char c : w) {
                int idx = c - 'a';
                if (!cur->children[idx]) cur->children[idx] = new TrieNode();
                cur = cur->children[idx];
            }
            cur->word = w;
        }
        return root;
    }

    void dfs(vector<vector<char>>& board, int r, int c,
             TrieNode* parent, vector<string>& result) {
        char ch = board[r][c];
        if (ch == '#' || !parent->children[ch - 'a']) return;  // visited or dead branch

        TrieNode* node = parent->children[ch - 'a'];
        if (!node->word.empty()) {
            result.push_back(node->word);
            node->word = "";   // dedupe: prevent adding the same word twice
        }

        board[r][c] = '#';     // mark visited for THIS path
        static const int dr[] = {-1, 1, 0, 0};
        static const int dc[] = {0, 0, -1, 1};
        for (int d = 0; d < 4; d++) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr >= 0 && nr < (int)board.size() &&
                nc >= 0 && nc < (int)board[0].size())
                dfs(board, nr, nc, node, result);
        }
        board[r][c] = ch;      // restore for other paths

        // PRUNE: if this node is now a dead end (no children, not a word),
        // detach it so future DFS calls fail fast at the parent check.
        bool hasChild = false;
        for (int i = 0; i < 26; i++)
            if (node->children[i]) { hasChild = true; break; }
        if (!hasChild && node->word.empty())
            parent->children[ch - 'a'] = nullptr;
    }

public:
    vector<string> findWords(vector<vector<char>>& board, vector<string>& words) {
        TrieNode* root = buildTrie(words);
        vector<string> result;
        for (int r = 0; r < (int)board.size(); r++)
            for (int c = 0; c < (int)board[0].size(); c++)
                dfs(board, r, c, root, result);
        return result;
    }
};
```

**TRACE for the classic LC 212 example:**
```
board = [['o','a','a','n'],
         ['e','t','a','e'],
         ['i','h','k','r'],
         ['i','f','l','v']]
words = ["oath", "pea", "eat", "rain"]

Trie built from words:
  root -> o-a-t-h*
  root -> p-e-a*
  root -> e-a-t*
  root -> r-a-i-n*

DFS finds "oath":
  (0,0)='o' matches root->o. push? node.word="" (not end yet).
    (0,1)='a' matches o->a. 
      (1,1)='t' matches o-a->t.
        (2,1)='h' matches o-a-t->h. node.word="oath" -> RESULT += "oath". Clear node.word.

DFS finds "eat":
  (1,3)='e' matches root->e.
    (1,2)='a' matches e->a (adjacent to (1,3)).
      (1,1)='t' matches e-a->t. node.word="eat" -> RESULT += "eat". Clear node.word.

"pea": no cell in the board contains 'p' at all, so root->children['p'-'a']
       is checked at every (r,c) and fails immediately — dfs() returns
       on its very first line without recursing. The "pea" trie branch
       is simply never visited (not a bug — nothing ever matches 'p').

"rain": DFS reaches 'r' at (2,3), but neither neighbor of (2,3)
        — (1,3)='e', (3,3)='v', (2,2)='k' — is 'a'. Dead end, no match.

Final result: ["eat", "oath"]  (order may vary)
```

**Why the dedupe (`node->word = ""` after collecting) matters:** a word can be reachable via more than one path in the grid (e.g., a word appearing twice by coincidence, or a palindrome-like board). Without clearing `word`, the same string would be pushed into `result` once per successful path — LC 212 requires each matched word to appear **once**.

---

## SECTION 6 — TEMPLATE 4: REPLACE WORDS (LC 648, SHORTEST-PREFIX LOOKUP)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 648 — Replace Words
// Given a dictionary of "roots" and a sentence, replace every word
// in the sentence with the SHORTEST root that is a prefix of it.
// If no root matches, leave the word unchanged.
//
// KEY IDEA: build a Trie of roots. For each sentence word, walk the
// Trie character by character; the INSTANT you hit an isEnd node,
// that prefix is by definition the shortest matching root (because
// we walked it in increasing length order) — stop immediately.
// ─────────────────────────────────────────────────────────────────

class Solution {
    struct TrieNode {
        TrieNode* children[26];
        bool isEnd;
        TrieNode() : isEnd(false) {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };

    string shortestRoot(TrieNode* root, const string& word) {
        TrieNode* cur = root;
        string prefix;
        for (char c : word) {
            int idx = c - 'a';
            if (!cur->children[idx]) return word;   // no root matches -> keep original
            prefix += c;
            cur = cur->children[idx];
            if (cur->isEnd) return prefix;           // shortest root found — stop now
        }
        return word;   // walked the whole word, never hit isEnd
    }

public:
    string replaceWords(vector<string>& dictionary, string sentence) {
        TrieNode* root = new TrieNode();
        for (string& r : dictionary) {
            TrieNode* cur = root;
            for (char c : r) {
                int idx = c - 'a';
                if (!cur->children[idx]) cur->children[idx] = new TrieNode();
                cur = cur->children[idx];
            }
            cur->isEnd = true;
        }

        stringstream ss(sentence);
        string word, result;
        while (ss >> word) {
            if (!result.empty()) result += ' ';
            result += shortestRoot(root, word);
        }
        return result;
    }
};
```

**TRACE for dictionary = ["cat","bat","rat"], sentence = "the cattle was rattled by the battery":**
```
Trie: root -> c-a-t*, root -> b-a-t*, root -> r-a-t*

"the":      c/b/r? root->children['t'-'a'] doesn't exist -> unchanged: "the"
"cattle":   c matches. a matches. t matches -> isEnd=true at "cat" -> STOP. Result: "cat"
"was":      root->children['w'-'a'] doesn't exist -> unchanged: "was"
"rattled":  r matches. a matches. t matches -> isEnd=true at "rat" -> STOP. Result: "rat"
"by":       root->children['b'-'a'] exists (from "bat")... but next char 'y':
            node(b)->children['y'-'a'] doesn't exist -> unchanged: "by"
"the":      unchanged: "the"
"battery":  b matches. a matches. t matches -> isEnd=true at "bat" -> STOP. Result: "bat"

Final: "the cat was rat by the bat"
```

---

## SECTION 7 — TEMPLATE 5: MAXIMUM XOR OF TWO NUMBERS (LC 421, BIT TRIE)

This is the template that separates 1600-level Trie users from 2000-level ones. The alphabet is `{0, 1}` instead of `{a..z}`, and the goal is a greedy walk, not an exact match.

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 421 — Maximum XOR of Two Numbers in an Array
// Find max(nums[i] XOR nums[j]) over all pairs.
//
// KEY IDEA: XOR is maximized by making the HIGHEST bit differ first
// (a 1 in bit position k contributes 2^k, more than every lower bit
// combined). So: insert every number into a bit-trie, MSB first.
// For each number, greedily walk toward the OPPOSITE bit at every
// level if that child exists — that maximizes the running XOR.
// ─────────────────────────────────────────────────────────────────

class Solution {
    struct TrieNode {
        TrieNode* children[2];   // children[0] = bit 0 path, children[1] = bit 1 path
        TrieNode() { children[0] = children[1] = nullptr; }
    };
    static const int BITS = 31;  // nums[i] < 2^31 (non-negative int) -> bits 31..0

    void insert(TrieNode* root, int num) {
        TrieNode* cur = root;
        for (int i = BITS; i >= 0; i--) {
            int bit = (num >> i) & 1;
            if (!cur->children[bit]) cur->children[bit] = new TrieNode();
            cur = cur->children[bit];
        }
    }

    int queryMax(TrieNode* root, int num) {
        TrieNode* cur = root;
        int result = 0;
        for (int i = BITS; i >= 0; i--) {
            int bit = (num >> i) & 1;
            int wanted = 1 - bit;              // opposite bit maximizes this position
            if (cur->children[wanted]) {
                result |= (1 << i);
                cur = cur->children[wanted];
            } else {
                cur = cur->children[bit];       // forced: only same-bit child exists
            }
        }
        return result;
    }

public:
    int findMaximumXOR(vector<int>& nums) {
        TrieNode* root = new TrieNode();
        for (int num : nums) insert(root, num);

        int maxXor = 0;
        for (int num : nums) maxXor = max(maxXor, queryMax(root, num));
        return maxXor;
    }
};
```

**FULL TRACE for nums = [3, 10, 5, 25, 2, 8]** (answer should be 28, from 5 XOR 25).

For clarity we trace with **5 bits** (bit4..bit0) since every value here is < 32; production code uses `BITS = 31` for full 32-bit generality.

```
Binary (5-bit, MSB=bit4 first):
   3 = 0 0 0 1 1
  10 = 0 1 0 1 0
   5 = 0 0 1 0 1
  25 = 1 1 0 0 1
   2 = 0 0 0 1 0
   8 = 0 1 0 0 0

All six inserted into the bit trie (shared prefixes merge, e.g. all
values with bit4=0 — that's 3,10,5,2,8 — share the root's '0' child;
only 25 goes down the '1' child).

queryMax(root, 5):   5 = 0 0 1 0 1
  bit4=0, want 1. root has a '1' child (from 25). Take it. result |= 10000 = 16.
    Now we're in the subtree that ONLY contains 25 (00101... wait — only
    25 has bit4=1, so this subtree has exactly one path: 25's remaining bits).
  bit3=0, want 1. 25's bit3 = 1 (25 = 11001). Match! Take it. result |= 01000 -> 24.
  bit2=1, want 0. 25's bit2 = 0. Match! Take it. result |= 00100 -> 28.
  bit1=0, want 1. 25's bit1 = 0. No match — forced down bit1's own '0' child. result unchanged (28).
  bit0=1, want 0. 25's bit0 = 1. No match — forced down bit0's own '1' child. result unchanged (28).
  queryMax(5) = 28.

Verify directly: 5 = 00101, 25 = 11001, XOR = 11100 = 16+8+4 = 28. ✓ MATCHES.

maxXor over all six numbers' queries = 28 (achieved by the pair (5, 25)).
```

**Why greedy MSB-first is provably correct:** at each level, choosing the opposite bit sets that bit position to 1 in the XOR result, which is worth more than the sum of ALL lower bits combined (`2^k > 2^(k-1) + 2^(k-2) + ... + 2^0`). So a greedy choice at a higher bit is never beaten by any combination of choices at lower bits — this is the same "greedy digit-DP" argument used in bitmask/greedy-construction problems.

---

## SECTION 8 — TEMPLATE 6: LONGEST WORD IN DICTIONARY (LC 720)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 720 — Longest Word in Dictionary
// Find the longest word that can be built one character at a time
// by OTHER words in the list (i.e., every prefix of the answer must
// ALSO be a complete word in the dictionary). Break ties by
// lexicographically smallest.
//
// KEY IDEA: build the Trie, then DFS from the root but ONLY descend
// into a child if that child is itself isEnd=true — this guarantees
// every prefix along the current path is a real dictionary word.
// ─────────────────────────────────────────────────────────────────

class Solution {
    struct TrieNode {
        TrieNode* children[26];
        bool isEnd;
        TrieNode() : isEnd(false) {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };

    string best;

    void dfs(TrieNode* node, string& path) {
        // path is currently a valid chain of complete words (or empty at root)
        if (path.size() > best.size() ||
            (path.size() == best.size() && path < best)) {
            best = path;
        }
        for (int i = 0; i < 26; i++) {
            if (node->children[i] && node->children[i]->isEnd) {
                path.push_back('a' + i);
                dfs(node->children[i], path);
                path.pop_back();
            }
        }
    }

public:
    string longestWord(vector<string>& words) {
        TrieNode* root = new TrieNode();
        for (string& w : words) {
            TrieNode* cur = root;
            for (char c : w) {
                int idx = c - 'a';
                if (!cur->children[idx]) cur->children[idx] = new TrieNode();
                cur = cur->children[idx];
            }
            cur->isEnd = true;
        }
        best = "";
        string path = "";
        dfs(root, path);
        return best;
    }
};
```

**TRACE for words = ["a","banana","app","appl","ap","apply","apple"]:**
```
Trie isEnd flags: a*, ap*, app*, appl*, apple*, apply*, banana(only full word, no prefix chain)

dfs(root, ""):
  best="" (path="" beats nothing... actually best starts as "", tie broken trivially)
  child 'a': root->children['a'] exists AND isEnd=true ("a" is a word) -> descend
    path="a". best="a" (len1 > len0).
    child 'p': node(a)->children['p'] exists AND isEnd=true ("ap" is a word) -> descend
      path="ap". best="ap" (len2>len1).
      child 'p': node(ap)->children['p'] exists AND isEnd=true ("app") -> descend
        path="app". best="app".
        child 'l': node(app)->children['l'] exists AND isEnd=true ("appl") -> descend
          path="appl". best="appl".
          child 'e': isEnd=true ("apple") -> descend. path="apple". best="apple" (len5).
          child 'y': isEnd=true ("apply") -> descend. path="apply".
            len5 == len(best)=5, compare "apply" < "apple"? 'l' < 'y' so "apple" < "apply".
            best stays "apple" (lexicographically smaller wins the tie).
  child 'b': root->children['b'] exists but is "banana" only isEnd at the FULL word —
             intermediate nodes 'b','ba','ban',... are NOT isEnd (not in the word list)
             so node->children['b']->isEnd is FALSE at the first level check --
             wait: we check node->children[i]->isEnd for the CHILD before descending.
             root->children['b'] is NOT isEnd (only "banana" is a word, not "b").
             So this branch is never entered at all. "banana" is unreachable.

Final: best = "apple"
```

---

## SECTION 9 — TEMPLATE 7: MAP SUM PAIRS (LC 677, PREFIX SUM IN TRIE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 677 — Map Sum Pairs
// insert(key, val): associates val with key (may OVERWRITE an
//   existing key — this is the trap most solutions miss).
// sum(prefix): return the sum of vals for every inserted key that
//   starts with prefix.
//
// KEY IDEA: store a running sum at EVERY node along each key's
// insertion path. On overwrite, insert only the DELTA (newVal - oldVal)
// so previously-summed prefixes stay correct instead of double-counting.
// ─────────────────────────────────────────────────────────────────

class MapSum {
    struct TrieNode {
        TrieNode* children[26];
        int sum;   // sum of vals of all keys passing through this node
        TrieNode() : sum(0) {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };
    TrieNode* root;
    unordered_map<string, int> keyVal;   // tracks current value per key, for delta calc

public:
    MapSum() { root = new TrieNode(); }

    void insert(string key, int val) {
        int delta = val - keyVal[key];   // 0 if key is new (map default-constructs to 0)
        keyVal[key] = val;

        TrieNode* cur = root;
        for (char c : key) {
            int idx = c - 'a';
            if (!cur->children[idx]) cur->children[idx] = new TrieNode();
            cur = cur->children[idx];
            cur->sum += delta;           // propagate delta to every node on the path
        }
    }

    int sum(string prefix) {
        TrieNode* cur = root;
        for (char c : prefix) {
            int idx = c - 'a';
            if (!cur->children[idx]) return 0;
            cur = cur->children[idx];
        }
        return cur->sum;
    }
};
```

**TRACE:**
```
insert("apple", 3):
  delta = 3 - 0 = 3. keyVal["apple"]=3.
  path a-p-p-l-e, each node's sum += 3.
  sum("ap") = 3   (walk a->p, node(ap).sum = 3)

insert("app", 2):
  delta = 2 - 0 = 2. keyVal["app"]=2.
  path a-p-p, each node's sum += 2.
  node(a).sum = 3+2=5, node(ap).sum=3+2=5, node(app).sum=3+2=5
  (node(appl).sum and node(apple).sum are untouched by this insert, still 3)

  sum("ap") = 5   (3 from "apple" + 2 from "app")

insert("apple", 5):     <-- OVERWRITE of an existing key
  delta = 5 - keyVal["apple"] = 5 - 3 = 2. keyVal["apple"]=5.
  path a-p-p-l-e, each node's sum += 2 (NOT += 5 — that would double count the old 3).
  node(a).sum = 5+2=7, node(ap).sum=5+2=7, node(app).sum=5+2=7,
  node(appl).sum=3+2=5, node(apple).sum=3+2=5

  sum("ap") = 7   ( = 2 from "app" + 5 from "apple", correctly updated)
  sum("appl") = 5 ( = "apple"'s current value, "app" doesn't reach this node)
```

**Why the `unordered_map<string,int> keyVal` is mandatory:** without tracking each key's last-inserted value, a re-insertion of `"apple"` with a new value would add the FULL new value to every node's `sum`, silently double-counting the original insertion. This is the single most common bug in LC 677 submissions.

---

## SECTION 10 — TEMPLATE 8: STREAM OF CHARACTERS (LC 1032, REVERSED TRIE)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1032 — Stream of Characters
// query(letter): feed one character at a time. Return true if any
// word in the dictionary is a SUFFIX of the stream seen so far.
//
// KEY IDEA: a suffix-of-the-stream match is equivalent to a
// prefix-of-the-REVERSED-word match walked backward from the most
// recent character. So: insert every dictionary word REVERSED into
// a normal trie. On each query, walk backward through a buffer of
// the last `maxLen` characters, following Trie edges; if we ever
// land on an isEnd node, some word matches as a suffix.
// ─────────────────────────────────────────────────────────────────

class StreamChecker {
    struct TrieNode {
        TrieNode* children[26];
        bool isEnd;
        TrieNode() : isEnd(false) {
            for (int i = 0; i < 26; i++) children[i] = nullptr;
        }
    };
    TrieNode* root;
    string buffer;     // last (at most maxLen) characters of the stream
    int maxLen;

public:
    StreamChecker(vector<string>& words) {
        root = new TrieNode();
        maxLen = 0;
        for (string& w : words) {
            maxLen = max(maxLen, (int)w.size());
            TrieNode* cur = root;
            for (int i = (int)w.size() - 1; i >= 0; i--) {   // insert REVERSED
                int idx = w[i] - 'a';
                if (!cur->children[idx]) cur->children[idx] = new TrieNode();
                cur = cur->children[idx];
            }
            cur->isEnd = true;
        }
    }

    bool query(char letter) {
        buffer += letter;
        if ((int)buffer.size() > maxLen)
            buffer.erase(buffer.begin(), buffer.end() - maxLen);  // cap buffer length

        TrieNode* cur = root;
        for (int i = (int)buffer.size() - 1; i >= 0; i--) {       // walk BACKWARD
            int idx = buffer[i] - 'a';
            if (!cur->children[idx]) return false;
            cur = cur->children[idx];
            if (cur->isEnd) return true;    // suffix match found ending at current char
        }
        return false;
    }
};
```

**TRACE for words = ["cd", "f", "kl"], stream = 'a','b','c','d','e','f':**
```
maxLen = 2
Reversed trie: "cd" -> d-c*,  "f" -> f*,  "kl" -> l-k*

query('a'): buffer="a".  walk backward: 'a' -> root has no 'a' child. FALSE.
query('b'): buffer="ab". walk backward from 'b': root has no 'b' child. FALSE.
query('c'): buffer="bc" (capped to last 2). walk backward from 'c':
            root->children['c'-'a']? NO — only 'd','f','l' exist at root
            (since words were reversed: "dc","f","lk" — root's children are d,f,l).
            root has no 'c' child. FALSE.
query('d'): buffer="cd". walk backward from 'd' (last char):
            root->children['d'-'a'] EXISTS (from reversed "cd"->"dc"). Take it.
            node(d).isEnd? "dc" full path not done yet — isEnd is at 'c' after 'd'. FALSE so far.
            next char back: 'c'. node(d)->children['c'-'a'] EXISTS. Take it.
            node(dc).isEnd = TRUE ("cd" reversed = "dc", fully matched). -> TRUE.
query('e'): buffer="de". walk backward from 'e': root has no 'e' child. FALSE.
query('f'): buffer="ef" (capped). walk backward from 'f':
            root->children['f'-'a'] EXISTS ("f" is itself a 1-char word). 
            node(f).isEnd = TRUE immediately. -> TRUE.
```

**Why cap the buffer to `maxLen`:** without capping, `buffer` (and the backward walk) grows unbounded over a long stream, turning each `query` call into O(stream length so far) instead of O(maxLen). Over Q queries this is the difference between O(Q · maxLen) and O(Q²).

---

## SECTION 11 — TEMPLATE 9: MAXIMUM XOR WITH AN ELEMENT FROM ARRAY (LC 1707)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 1707 — Maximum XOR With an Element From Array
// queries[i] = [x_i, m_i]: answer = max XOR of x_i with any nums[j]
// such that nums[j] <= m_i. If no such nums[j] exists, answer = -1.
//
// NAIVE: for each query, filter nums by m_i, then run LC421-style
// max-XOR over the filtered set. O(Q * N) worst case. TLE for large Q,N.
//
// OFFLINE + INCREMENTAL TRIE:
//   1. Sort nums ascending.
//   2. Sort QUERY INDICES by m_i ascending.
//   3. Walk queries in that order, inserting nums into the trie
//      incrementally as each query's threshold m_i allows (a two-
//      pointer style advance — numbers are inserted AT MOST once,
//      total insertion work O(N * BITS) across ALL queries).
//   4. Query the trie in its CURRENT (partial) state for that x_i.
// ─────────────────────────────────────────────────────────────────

class Solution {
    struct TrieNode {
        TrieNode* children[2];
        TrieNode() { children[0] = children[1] = nullptr; }
    };
    static const int BITS = 29;   // nums[i], x_i <= 1e9 < 2^30 -> bits 29..0 (30 bits, headroom)

    void insert(TrieNode* root, int num) {
        TrieNode* cur = root;
        for (int i = BITS; i >= 0; i--) {
            int bit = (num >> i) & 1;
            if (!cur->children[bit]) cur->children[bit] = new TrieNode();
            cur = cur->children[bit];
        }
    }

    int queryMax(TrieNode* root, int num) {
        TrieNode* cur = root;
        int result = 0;
        for (int i = BITS; i >= 0; i--) {
            int bit = (num >> i) & 1;
            int wanted = 1 - bit;
            if (cur->children[wanted]) {
                result |= (1 << i);
                cur = cur->children[wanted];
            } else {
                cur = cur->children[bit];   // guaranteed to exist (trie is non-empty)
            }
        }
        return result;
    }

public:
    vector<int> maximizeXor(vector<int>& nums, vector<vector<int>>& queries) {
        sort(nums.begin(), nums.end());
        int q = queries.size();
        vector<int> order(q);
        iota(order.begin(), order.end(), 0);
        sort(order.begin(), order.end(), [&](int a, int b) {
            return queries[a][1] < queries[b][1];   // sort by m_i ascending
        });

        vector<int> ans(q, -1);
        TrieNode* root = new TrieNode();
        int idx = 0;    // pointer into sorted nums, never resets — total O(N) advances

        for (int qi : order) {
            int x = queries[qi][0], m = queries[qi][1];
            while (idx < (int)nums.size() && nums[idx] <= m) {
                insert(root, nums[idx]);
                idx++;
            }
            if (idx == 0) ans[qi] = -1;          // no valid nums for this query yet
            else          ans[qi] = queryMax(root, x);
        }
        return ans;
    }
};
```

**TRACE for nums = [0,1,2,3,4], queries = [[3,1],[1,3],[5,6]] (expected [3,3,7]):**
```
sorted nums = [0,1,2,3,4]
queries sorted by m ascending: index0 [3,1] (m=1), index1 [1,3] (m=3), index2 [5,6] (m=6)

Process qi=0, x=3, m=1:
  insert nums <= 1: 0, 1. idx now = 2. Trie holds {0,1}.
  queryMax(3): 3^0=3, 3^1=2 -> best=3. ans[0] = 3.

Process qi=1, x=1, m=3:
  insert nums <= 3 starting from idx=2: 2, 3. idx now = 4. Trie holds {0,1,2,3}.
  queryMax(1): 1^0=1, 1^1=0, 1^2=3, 1^3=2 -> best=3. ans[1] = 3.

Process qi=2, x=5, m=6:
  insert nums <= 6 starting from idx=4: 4. idx now = 5. Trie holds {0,1,2,3,4}.
  queryMax(5): 5^0=5, 5^1=4, 5^2=7, 5^3=6, 5^4=1 -> best=7. ans[2] = 7.

Final answers in original order: ans = [3, 3, 7] ✓ MATCHES expected.
```

**Why sort-by-`m` instead of a fresh trie per query:** each number is inserted into the trie **at most once** across the entire run (the `idx` pointer only moves forward), giving total insertion cost O(N · BITS) instead of O(Q · N · BITS) for rebuilding per query. This offline technique — "sort queries by their constraint, sweep a pointer, answer with a growing data structure" — reappears constantly (it is the same idea behind offline range-query techniques like Mo's algorithm).

---

## SECTION 12 — MEMORY MANAGEMENT: WHY ARRAY-OF-26 IS FAST

A `TrieNode` with `TrieNode* children[26]` costs `26 * 8 = 208` bytes of pointers alone (64-bit build), plus the payload (`bool isEnd`, `string word`, `int sum`, etc.). For a dictionary of 10,000 five-letter words, worst case that's 50,000 nodes × ~216 bytes ≈ 10.8 MB — well within typical contest/interview limits, and this is the **worst** case (no shared prefixes at all). Real dictionaries share prefixes heavily, so actual node counts are far lower.

**Why array indexing beats `unordered_map<char, TrieNode*>` in practice:**
1. **O(1) truly constant-time access** — `children[c - 'a']` is a single pointer-arithmetic + dereference. A hash map access involves computing a hash, resolving a bucket, and (on collision) a linear probe/chain walk — 3-10x slower per step even though both are "O(1)."
2. **Cache locality** — an array of 26 pointers is one contiguous 208-byte block; hardware prefetching loves it. A hash map's internal buckets/nodes are scattered across the heap.
3. **No hashing overhead** — computing a hash for a single `char` is wasted work when `c - 'a'` already IS a perfect, collision-free index.

**When to switch to a map node instead:** if the alphabet is large (Unicode) or the trie is extremely sparse (e.g., a handful of long random strings sharing almost no prefixes), the 208-bytes-per-node overhead of unused pointers dominates memory, and `unordered_map<char, TrieNode*>` (or a small sorted `vector<pair<char,TrieNode*>>` for very few children) becomes the better trade.

**Destructor discipline:** none of the templates above free memory (`delete` every node in a destructor) — this mirrors standard interview/contest practice where the process exits and the OS reclaims memory. In a **long-lived service** (e.g., a Trie backing an autocomplete API that is rebuilt repeatedly), you MUST write a recursive destructor:
```cpp
~TrieNode() {
    for (int i = 0; i < 26; i++) delete children[i];
}
```
This recursively frees the whole subtree; omitting it in a long-running process is a genuine memory leak, not just a style nit.

---

## SECTION 13 — COMPLEXITY TABLE

| Operation / Problem | Time | Space | Notes |
|---|---|---|---|
| insert(word) | O(L) | O(L) worst case (new nodes) | L = word length; O(1) extra if prefix already exists |
| search(word) / startsWith(prefix) | O(L) | O(1) | Fail-fast on first missing child |
| Add and Search Word, `.` wildcard | O(26^d · L) worst case | O(L) recursion depth | d = number of dots; typically far better in practice |
| Word Search II (LC 212) | O(m·n·4^maxLen) worst case, pruned in practice | O(sum of word lengths) for trie | Pruning is essential — see Mistake 4 |
| Replace Words (LC 648) | O(total dict chars + total sentence chars) | O(total dict chars) | Early-exit at first isEnd |
| Max XOR of Two Numbers (LC 421) | O(N · 32) | O(N · 32) worst case nodes | Bit trie, 2 children per node |
| Longest Word in Dictionary (LC 720) | O(total chars) | O(total chars) | DFS only through isEnd chains |
| Map Sum Pairs (LC 677) | O(L) insert, O(L) sum | O(total chars) | Delta trick for overwrites |
| Stream of Characters (LC 1032) | O(maxLen) per query | O(total dict chars) | Reversed trie + capped buffer |
| Max XOR With Element From Array (LC 1707) | O((N+Q)·30·log) | O(N · 30) | Offline sort + incremental insert |

General rule: **insert/search/startsWith are always O(L)** where L is the string (or bit-width) length — completely independent of how many other strings are stored. This is the core value proposition of a Trie over a hash set for prefix-shaped queries.

---

## SECTION 14 — VARIANTS

1. **Compressed Trie / Radix Tree:** collapse chains of single-child nodes into one edge labeled with a substring, instead of one node per character. Reduces node count from O(total chars) to O(number of branching points) — used when memory is tight and words share long non-branching runs (e.g., URLs, file paths).

2. **Ternary Search Trie (TST):** each node has only 3 children (less-than, equal, greater-than) instead of 26/2. Slower per-character (O(log alphabet) vs O(1)) but far more memory-efficient for sparse or large alphabets — a middle ground between array-of-26 and hash-map nodes.

3. **Suffix Trie / Suffix Tree:** insert every SUFFIX of a single string (not every string in a list) to answer substring-existence and substring-count queries in O(pattern length). Overkill for most interview problems; Pattern 27 (or dedicated string-matching patterns like Z-function/KMP) usually covers substring search more directly.

4. **Trie with node deletion (reference counting):** to support a `remove(word)` operation, store a `refCount` (or `passCount`) at each node — the number of inserted words passing through it. `insert` increments along the path; `remove` decrements, and any node whose count hits 0 is detached from its parent (and can be `delete`d). This generalizes the "prune dead branches" trick from Word Search II into a first-class trie operation:
```cpp
struct TrieNode {
    TrieNode* children[26];
    int passCount = 0;   // number of words currently using this node
    int endCount = 0;    // number of words that END exactly here (handles duplicates)
    TrieNode() { for (int i=0;i<26;i++) children[i]=nullptr; }
};
void remove(TrieNode* root, const string& word) {
    TrieNode* cur = root;
    vector<TrieNode*> path = {root};
    for (char c : word) { cur = cur->children[c-'a']; path.push_back(cur); }
    cur->endCount--;
    for (int i = (int)word.size(); i >= 1; i--) {
        path[i]->passCount--;
        if (path[i]->passCount == 0) {
            path[i-1]->children[word[i-1]-'a'] = nullptr;
            delete path[i];
        }
    }
}
```

5. **XOR Trie with subtree size (for "count pairs with XOR ≥ K"):** augment the bit trie's nodes with a subtree count; combined with the greedy MSB walk, this answers "how many stored numbers XOR with x to at least K" in O(BITS) instead of O(N) — a common contest extension of LC 421.

---

## SECTION 15 — COMMON MISTAKES

### Mistake 1: Forgetting `isEnd` (confusing "prefix exists" with "word exists")

```cpp
// WRONG — search() without checking isEnd
bool search(string word) {
    TrieNode* cur = root;
    for (char c : word) {
        int idx = c - 'a';
        if (!cur->children[idx]) return false;
        cur = cur->children[idx];
    }
    return true;   // BUG: this is startsWith's logic, not search's!
}
// If "app" was inserted but "ap" was never inserted as its own word,
// search("ap") WRONGLY returns true — it only confirms "ap" is a PREFIX,
// not that "ap" was itself an inserted word.

// CORRECT
bool search(string word) {
    TrieNode* node = walk(word);
    return node != nullptr && node->isEnd;
}
```

### Mistake 2: Wildcard DFS not restoring state / not short-circuiting

```cpp
// WRONG — collects results from ALL 26 branches even after finding one match
bool dfs(const string& word, int idx, TrieNode* node) {
    if (idx == word.size()) return node->isEnd;
    if (word[idx] == '.') {
        bool found = false;
        for (int i = 0; i < 26; i++)
            if (node->children[i])
                found = found || dfs(word, idx+1, node->children[i]);  // BUG: no early return
        return found;
    }
    ...
}
// This still gives the correct ANSWER, but explores all 26 branches even
// after the first one succeeds — for a query like "....." on a large trie
// this can be an order of magnitude slower than necessary.

// CORRECT — return true immediately on the first successful branch
for (int i = 0; i < 26; i++)
    if (node->children[i] && dfs(word, idx+1, node->children[i]))
        return true;
return false;
```

### Mistake 3: Bit trie walked LSB-first instead of MSB-first

```cpp
// WRONG — LSB first
for (int i = 0; i <= BITS; i++) {   // BUG: ascending, starts from bit 0
    int bit = (num >> i) & 1;
    ...
}
// Greedily maximizing the LOWEST bit first is meaningless — a 1 at bit 0
// is worth 1, while a 1 at bit 31 is worth 2^31. You must lock in the
// highest bits first because they dominate the final numeric value.

// CORRECT — MSB first
for (int i = BITS; i >= 0; i--) { ... }
```

### Mistake 4: Word Search II without pruning — TLE on dense dictionaries

```cpp
// WRONG — build trie, DFS board, but NEVER remove exhausted branches
void dfs(vector<vector<char>>& board, int r, int c, TrieNode* node, vector<string>& result) {
    char ch = board[r][c];
    if (ch == '#' || !node->children[ch-'a']) return;
    TrieNode* next = node->children[ch-'a'];
    if (!next->word.empty()) { result.push_back(next->word); next->word = ""; }
    board[r][c] = '#';
    for (four directions) dfs(...);
    board[r][c] = ch;
    // BUG: missing the prune step. Every future DFS call that reaches
    // this now-useless node still walks its (now childless, wordless)
    // subtree structure over and over across every starting cell —
    // on large boards with ~10,000-word dictionaries this alone flips
    // an Accepted solution into a TLE.
}

// CORRECT — after backtracking, detach dead nodes (see Section 5's full code):
bool hasChild = false;
for (int i = 0; i < 26; i++) if (next->children[i]) { hasChild = true; break; }
if (!hasChild && next->word.empty()) node->children[ch-'a'] = nullptr;
```

### Mistake 5: Map Sum Pairs — re-inserting a key without delta correction

```cpp
// WRONG — always adds the full new value, ignoring any prior value for the same key
void insert(string key, int val) {
    TrieNode* cur = root;
    for (char c : key) {
        cur = cur->children[c-'a'] ? cur->children[c-'a'] : (cur->children[c-'a'] = new TrieNode());
        cur->sum += val;   // BUG: if key was inserted before, this double-(or more-)counts
    }
}
// insert("apple", 3) then insert("apple", 5):
//   after 2nd insert, sum("apple") = 3 + 5 = 8. WRONG — should be 5 (the value was overwritten).

// CORRECT — track each key's current value, insert only the delta
int delta = val - keyVal[key];
keyVal[key] = val;
... cur->sum += delta ...   // see full Section 9 code
```

### Mistake 6: Off-by-one in "shortest root" (Replace Words) — not stopping at the first isEnd

```cpp
// WRONG — always walks the FULL word, ignoring shorter matching roots found along the way
string shortestRoot(TrieNode* root, const string& word) {
    TrieNode* cur = root;
    string prefix;
    for (char c : word) {
        int idx = c - 'a';
        if (!cur->children[idx]) return word;
        prefix += c;
        cur = cur->children[idx];
    }
    return cur->isEnd ? prefix : word;   // BUG: only checks isEnd at the VERY END
}
// dictionary=["cat"], word="cattle": walks c-a-t-t-l-e, fails at the 4th char
// ('t' after "cat" has no child in a trie containing ONLY "cat") -> returns "cattle"
// unchanged, when it should have returned "cat" the moment isEnd was hit at step 3.

// CORRECT — check isEnd INSIDE the loop, return immediately
for (char c : word) {
    ...
    cur = cur->children[idx];
    if (cur->isEnd) return prefix;   // stop at the FIRST (shortest) match
}
return word;
```

---

## SECTION 16 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Implement Trie (Prefix Tree) | 208 | Array-of-26 node, insert/search/startsWith |
| 2 | Design Add and Search Words Data Structure | 211 | Wildcard DFS with early-exit fan-out |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Word Search II | 212 | Trie + backtracking on a grid, pruning to avoid TLE |
| 4 | Replace Words | 648 | Shortest-prefix walk, stop at first isEnd |
| 5 | Maximum XOR of Two Numbers in an Array | 421 | Bit trie, greedy MSB-first walk |
| 6 | Longest Word in Dictionary | 720 | DFS restricted to isEnd chains, lexicographic tie-break |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Map Sum Pairs | 677 | Prefix-sum payload, delta trick for key overwrites |
| 8 | Palindrome Pairs | 336 | Trie of reversed words + palindrome-prefix/suffix checks |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Stream of Characters | 1032 | Reversed trie, backward walk, capped buffer |
| 10 | Maximum XOR With an Element From Array | 1707 | Offline sort by constraint + incremental bit trie |

---

## SECTION 17 — PATTERN CONNECTIONS

1. **Pattern 15 (Backtracking):** Word Search II (LC 212) is fundamentally a backtracking DFS over the grid (Pattern 15's "explore, mark visited, recurse, unmark" template) with the Trie serving as an O(1)-per-step pruning oracle — instead of checking "does this partial string appear as a prefix of any word?" by scanning the whole word list, the Trie answers it in one pointer hop. Any grid backtracking problem that also involves a fixed dictionary of targets is a Trie + Pattern 15 combination.

2. **Pattern 28 (Bit Manipulation / BIT):** the bit trie (LC 421, LC 1707) is one of two standard approaches to "maximize/count XOR pairs" problems — the other being a **Binary Indexed Tree over bit-reversed values** or a **sorted-prefix + hashing** trick. When a bit-trie solution feels natural but memory is tight (very large N with wide bit-width), Pattern 28's BIT-based alternative is worth knowing as a fallback, though it's typically more complex to derive than the trie.

3. **Hashing (Pattern 01/05 family):** a Trie and a hash set both answer "is X present" in O(L), and a Trie can always be replaced by a hash set when prefix/wildcard/XOR structure isn't needed. Conversely, whenever a hash-set-based solution to a string problem starts doing `substr()` comparisons in a loop to check prefixes, that's the signal to switch to a Trie — it eliminates the repeated substring-allocation cost.

4. **Pattern 20 (LCS / Edit Distance family):** both Tries and 2D string DP solve "structural relationships between strings," but Tries exploit SHARED PREFIXES across MANY strings, while LCS/Edit Distance compare exactly TWO strings character-by-character regardless of prefix sharing. If the problem says "given a LIST of words" + "prefix/word-boundary" language, think Trie; if it says "given TWO strings" + "transform/align" language, think Pattern 20.

5. **Pattern 22 (DP on Trees):** a Trie IS a tree, and problems like Longest Word in Dictionary (LC 720) are literally a tree DFS with a validity constraint at each node (only descend if the child is a complete word) — the same "constrained DFS accumulating a running best" shape that recurs throughout Pattern 22's tree-DP problems.

---

## SECTION 18 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 212 — Word Search II

**Interviewer:** "Given a board of letters and a list of words, find all words that can be traced on the board."

**Candidate:**

*[First minute — reject the naive approach out loud]*
> "Brute force: run Word Search I once per word, O(W · m·n·4^L). With W up to 10,000+ words this times out. The fix: build ONE Trie of all words, then DFS the board exactly once. At each board cell I only follow the Trie edge matching that letter — this explores all words simultaneously and shares their common prefixes for free."

*[State the pruning requirement before coding — this is the differentiator]*
> "One more thing before I code: without pruning, the DFS keeps re-walking dead subtrees — branches whose words are already found and have no live children left. I'll detach those nodes from their parent after backtracking out of them. This is the difference between Accepted and TLE on the harder test cases."

*[Code — 8 minutes, includes both the trie build and the pruning step]*

*[Interviewer]: "Walk me through why you clear `node->word` after collecting it."*
> "Two reasons. First, correctness — the same word could be reachable via multiple paths on the board, and the problem wants each match reported once, so clearing it acts as a 'already collected' flag. Second, it doubles as the trigger for pruning: a node with an empty `word` AND no children is now completely dead weight, safe to detach."

**RED FLAGS:**
- Running Word Search I independently per word (no shared Trie)
- Forgetting the `board[r][c] = '#'` visited-marking / restore step (reusing a cell within one path)
- No pruning — passes small tests, TLEs on the large dictionary test case
- Not clearing the matched word, causing duplicate entries in the result

---

### Interview Simulation 2: LC 421 — Maximum XOR of Two Numbers

**Interviewer:** "Given an array of integers, find the maximum XOR of any two elements. Can you do better than O(N²)?"

**Candidate:**

*[Identify the tool before touching code]*
> "O(N²) checks every pair directly. To beat that, I need a data structure that finds the best XOR PARTNER for a given number faster than scanning everyone. A bit trie does this: insert every number as a 32-bit binary string, then for any number x, greedily walk toward the OPPOSITE bit at each level — that's provably the choice that maximizes XOR, since higher bit positions dominate lower ones combined."

*[Justify greedy correctness]*
> "Why is greedy MSB-first correct? Because `2^k` is strictly greater than the sum of `2^(k-1) + ... + 2^0`. So locking in a 1 at a higher bit position is never beaten by any combination of choices at lower positions — there's no need to backtrack or consider alternatives once a higher bit is fixed."

*[Code — 6 minutes, insert-all-then-query-all]*

*[Interviewer]: "Why insert ALL numbers first instead of inserting-and-querying one at a time?"*
> "Both work, but inserting one at a time means the first number you process has nothing to query against yet — you'd need a special case. Inserting everything first, then querying everyone against the complete trie, is simpler and the same O(N · 32) total cost. It also naturally handles finding a number's XOR with itself without extra bookkeeping, since XOR-with-self is always 0 and never the max unless all numbers are equal."

**RED FLAGS:**
- Walking bits LSB-first instead of MSB-first (breaks the greedy argument entirely)
- Forgetting to handle the "no opposite-bit child exists" case (must fall back to the same-bit child, not return early)
- Big-O confusion: claiming O(N log N) instead of the correct O(N · BITS), where BITS is a constant (32) — it's linear in N, not N log N

---

### Interview Simulation 3: Meta-discussion — When does a Trie actually help?

**Interviewer:** "I could solve most of these problems with a hash set. When do you actually reach for a Trie?"

**Candidate:**
> "Three signals, and I check for them before writing a single line:
>
> First, **prefix queries** — 'how many stored strings start with X' or 'find the shortest/longest stored string that is a prefix of X.' A hash set can't answer this without scanning everything; a Trie answers it in O(L) by construction.
>
> Second, **wildcard or fuzzy matching** — '.' matches any character, or 'find all stored words within edit distance 1.' The Trie's branching structure turns 'try every possibility at this position' into a natural DFS fan-out; a hash set has no structural hook for that at all.
>
> Third, and the one people forget — **bitwise structure**, specifically maximizing or counting XOR between numbers. Representing numbers as fixed-width bit strings and building a 2-ary trie turns an O(N) linear scan per query into an O(32) greedy walk. This is the one case where the 'alphabet' isn't even characters — it's bits — but the Trie mechanics are identical.
>
> If none of those three are present — the problem only ever asks 'is X in the set' or 'what's the value for key X' — I use a hash set/map. It's simpler, has less pointer overhead, and does exactly what's needed without extra machinery."

**RED FLAGS:**
- Defaulting to a Trie for every string problem regardless of whether prefix/wildcard/bitwise structure is actually present (over-engineering)
- Not recognizing XOR problems as a Trie use case at all (a very common gap even among strong candidates)

---

## SECTION 19 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why can't a plain `unordered_set<string>` efficiently answer "how many stored strings start with prefix P"?**

> A hash set is indexed by the FULL string's hash — there's no relationship between the hash of a string and the hash of its prefixes. Answering the query requires either scanning every stored string and checking `str.substr(0, P.size()) == P` (O(N·L)), or maintaining a separate prefix-count structure. A Trie answers it natively: walk P's characters (O(L)), and the node you land on already represents "everything that shares this prefix" — if you store a pass-count at each node, the answer is a single field read.

---

**Q2: In the wildcard DFS (LC 211), why must the loop over 26 children `return true` immediately on the first success rather than accumulating `found = found || dfs(...)`?**

> Both give the correct boolean answer, but early-return prunes the search — once one branch confirms a match, there's no reason to explore the remaining (up to) 25 branches at that '.' position, and no reason to explore ANY of their sub-branches either. For a query with several dots, `found = found || dfs(...)` can be exponentially slower in the worst case (all-dot query) because it always explores every branch to completion, while early-return stops the instant any valid completion is found.

---

**Q3: Why is the bit trie for XOR problems walked MOST-significant-bit first, and what breaks if you walk LSB-first?**

> XOR's numeric value is dominated by its highest set bit: `2^k > sum(2^0..2^(k-1))`. Greedily choosing the opposite bit at the highest available level locks in the largest possible contribution before any lower-bit decision could matter. Walking LSB-first destroys this — greedily maximizing bit 0 first is worthless information, because whatever gets decided at bit 31 later can override the entire result regardless of what bit 0 contributed. The greedy proof ONLY holds top-down.

---

**Q4: In Word Search II, what specifically causes a TLE if the pruning step is omitted, and why doesn't the pruning step change the SET of words found?**

> Without pruning, every future DFS call that reaches a node whose subtree is entirely "used up" (no live children, no un-collected word) still pays the cost of checking that node's children array and recursing into dead ends — across potentially thousands of starting cells and thousands of trie nodes, this multiplies into a large constant-factor slowdown that flips Accepted into TLE on adversarial inputs (large board, large dictionary with many short overlapping words). Pruning doesn't change correctness because a node is only ever detached AFTER it has already contributed its word (if any) to the result and has no children left to contribute future words — it is genuinely dead weight, not a false prune.

---

**Q5: Map Sum Pairs — why does re-inserting an existing key require inserting a DELTA rather than the raw new value?**

> Every node along a key's insertion path accumulates a running sum of ALL keys passing through it. If key "apple" was inserted once with value 3 (adding +3 along its path) and is later reinserted with value 5, adding the raw new value (+5) would leave the path holding `3 + 5 = 8` — double-counting the original insertion, when the correct total contribution of "apple" going forward is just 5. Inserting the delta (`5 - 3 = +2`) corrects the running sums to reflect the key's CURRENT value only.

---

**Q6: Stream of Characters (LC 1032) — explain precisely why reversing the dictionary words (instead of reversing the incoming stream) is the right transformation.**

> The stream only grows in one direction (append-only), so we can't reverse it — reversing incoming data isn't physically meaningful for a stream. But "is dictionary word W a suffix of the stream" is logically equivalent to "read the stream BACKWARD from the most recent character — does it spell out W forward" which is the same as "does it spell out reverse(W) when walked in the trie forward from the root." So reversing the (fixed, known-in-advance) dictionary words at construction time, then always walking the (append-only) stream buffer backward at query time, converts a suffix-matching problem into a standard prefix-matching problem without ever needing to reverse the stream itself.

---

**Q7: What is the single sentence that decides "Trie" vs "hash set" for a new string problem you've never seen before?**

> Ask: "does the query care about ANYTHING other than exact whole-string membership — specifically prefixes, wildcards, or bitwise/XOR structure?" If yes to any of those three, reach for a Trie (character-alphabet for prefix/wildcard, bit-alphabet for XOR). If the query is purely "is this exact string/number present, or what value maps to this exact key," a hash set/map is simpler and sufficient — don't build a Trie you don't need.

---

## SECTION 20 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. BASE NODE (array-of-26, the default choice for lowercase strings)
// ─────────────────────────────────────────────────────────────────
struct TrieNode {
    TrieNode* children[26];
    bool isEnd;
    TrieNode() : isEnd(false) {
        for (int i = 0; i < 26; i++) children[i] = nullptr;
    }
};

// ─────────────────────────────────────────────────────────────────
// 2. INSERT / SEARCH / STARTSWITH (LC 208 skeleton)
// ─────────────────────────────────────────────────────────────────
void insert(TrieNode* root, const string& word) {
    TrieNode* cur = root;
    for (char c : word) {
        int idx = c - 'a';
        if (!cur->children[idx]) cur->children[idx] = new TrieNode();
        cur = cur->children[idx];
    }
    cur->isEnd = true;
}
// search: walk, require final node non-null AND isEnd
// startsWith: walk, require final node non-null (isEnd irrelevant)

// ─────────────────────────────────────────────────────────────────
// 3. WILDCARD DFS ('.' matches any char) — LC 211 skeleton
// ─────────────────────────────────────────────────────────────────
bool dfs(const string& w, int i, TrieNode* node) {
    if (!node) return false;
    if (i == (int)w.size()) return node->isEnd;
    if (w[i] == '.') {
        for (int k = 0; k < 26; k++)
            if (node->children[k] && dfs(w, i+1, node->children[k])) return true;
        return false;
    }
    return dfs(w, i+1, node->children[w[i]-'a']);
}

// ─────────────────────────────────────────────────────────────────
// 4. TRIE + GRID DFS WITH PRUNING — LC 212 skeleton
// ─────────────────────────────────────────────────────────────────
// node carries a "word" field (non-empty at completions).
// On match: push word, then node->word = "" (dedupe).
// On backtrack: if node has no children AND node->word.empty(),
//               detach it from parent->children[ch-'a'].

// ─────────────────────────────────────────────────────────────────
// 5. BIT TRIE FOR XOR — LC 421 / LC 1707 skeleton
// ─────────────────────────────────────────────────────────────────
struct BitNode { BitNode* children[2]; BitNode(){children[0]=children[1]=nullptr;} };
const int BITS = 31;   // adjust to input's bit-width - 1

void insertBits(BitNode* root, int num) {
    BitNode* cur = root;
    for (int i = BITS; i >= 0; i--) {
        int b = (num >> i) & 1;
        if (!cur->children[b]) cur->children[b] = new BitNode();
        cur = cur->children[b];
    }
}
int queryMaxXor(BitNode* root, int num) {
    BitNode* cur = root; int result = 0;
    for (int i = BITS; i >= 0; i--) {
        int b = (num >> i) & 1, want = 1 - b;
        if (cur->children[want]) { result |= (1 << i); cur = cur->children[want]; }
        else cur = cur->children[b];
    }
    return result;
}

// ─────────────────────────────────────────────────────────────────
// 6. PREFIX-SUM PAYLOAD (Map Sum Pairs) — delta on overwrite
// ─────────────────────────────────────────────────────────────────
// delta = newVal - keyVal[key]; keyVal[key] = newVal;
// propagate `delta` (not newVal!) to cur->sum along the insertion path.

// ─────────────────────────────────────────────────────────────────
// 7. REVERSED TRIE FOR SUFFIX/STREAM MATCHING (LC 1032)
// ─────────────────────────────────────────────────────────────────
// Build: insert each dictionary word with i from size()-1 down to 0.
// Query: maintain a capped buffer (last maxLen chars); walk buffer
//        backward (from most recent char); return true at first isEnd.

// ─────────────────────────────────────────────────────────────────
// 8. OFFLINE + INCREMENTAL TRIE (LC 1707)
// ─────────────────────────────────────────────────────────────────
// sort(nums); sort query INDICES by their threshold m ascending;
// walk sorted queries, advancing an insertion pointer into nums
// (never resets), inserting while nums[idx] <= m; then query.

// ─────────────────────────────────────────────────────────────────
// 9. DECISION TABLE: TRIE VS HASH SET
// ─────────────────────────────────────────────────────────────────
// Exact membership / key->value only:      hash set / hash map
// Prefix count / shortest-or-longest-prefix: Trie (char alphabet)
// Wildcard / fuzzy pattern matching:         Trie + DFS fan-out
// Max/count XOR between numbers:             Trie (bit alphabet, 2-ary)
// Suffix-of-a-growing-stream matching:       Trie (reversed insertion)
```

---

## SECTION 21 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 10 problems in order.** Word Search II and the two XOR problems (LC 421, LC 1707) are the ones requiring genuine insight beyond "translate the definition into code." Budget 40 minutes each for those three.

2. **Practice the "is a Trie even needed" filter:** for every string problem you solve going forward (in any pattern), pause and ask Q7 from the Mental Model Checkpoint before reaching for a Trie. Over-applying Tries to plain membership problems is a common self-inflicted complexity tax.

3. **Word Search II → build the destructive-prune version yourself from scratch** without looking at Section 5's code, timing yourself. This is the problem most likely to appear verbatim (or near-verbatim) in a real onsite loop at the 1900+ level.

4. **Bit trie → extend LC 421 to "count pairs with XOR ≥ K"** (Variant 5 in Section 14) using a subtree-size-augmented bit trie. This is the natural next step once the greedy MSB walk is second nature.

5. **Preview Pattern 27 (String Matching: KMP/Z-function) and Pattern 28 (Bit Manipulation/BIT):** Trie handles PREFIX/WILDCARD/XOR structure; Pattern 27 handles SUBSTRING/PATTERN-occurrence structure (a different kind of string query); Pattern 28 revisits XOR-pair problems from a Binary-Indexed-Tree angle as an alternate tool for the same problem class.

---

## SECTION 22 — SIGN-OFF CRITERIA

### Tier 1 — Base Trie mastered
You implement insert/search/startsWith (LC 208) from a blank file in under 5 minutes, including the array-of-26 node struct, with no reference material. You can explain in one sentence why `search` needs `isEnd` but `startsWith` doesn't.

### Tier 2 — DFS-augmented Trie understood
You solve Add and Search Word (LC 211) with correct early-exit wildcard fan-out, and Replace Words (LC 648) with the correct "stop at first isEnd" shortest-prefix logic — both without looking up the transition.

### Tier 3 — Grid + bit tries solid
You solve Word Search II (LC 212) INCLUDING the pruning step (not just a version that passes small tests), and Maximum XOR of Two Numbers (LC 421) with a correct MSB-first greedy bit trie, and can justify the greedy correctness argument out loud in under 30 seconds.

### Tier 4 — Advanced payload/offline tries complete
You solve Map Sum Pairs (LC 677) with the delta-on-overwrite trick, Stream of Characters (LC 1032) with the reversed-trie + capped-buffer technique, and Maximum XOR With an Element From Array (LC 1707) with the offline sort-and-sweep incremental trie. You can explain why each of these three needed a variant beyond the base Trie template, not just recite the code.

**Sign-off threshold:** Solve 8 of 10 problems. Mandatory: LC 208 (base Trie — must be automatic), LC 212 (Word Search II — pruning must be present and explained), and LC 421 (Maximum XOR — MSB-first greedy reasoning must be articulated). These three represent the structural foundation, the hardest backtracking combination, and the signature "Trie for a non-string problem" insight of this pattern.

---

*Pattern 26 Complete — Trie*
*Next: Pattern 27 — String Matching (KMP, Z-function, Rabin-Karp — substring and pattern-occurrence queries)*
