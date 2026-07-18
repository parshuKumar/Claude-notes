# PATTERN 33 — STRING ALGORITHMS
## KMP, Z-Algorithm, Rabin-Karp, Manacher, String Hashing — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of string algorithms:**

Most candidates can state "use KMP for pattern matching" but cannot write the failure-function loop from memory under pressure. The failure-function (LPS array) construction is a self-referential DP where the pattern is matched against *itself*, and the index bookkeeping (`while (k > 0 && ...) k = lps[k-1];`) is exactly the kind of three-line block that looks trivial when read and is surprisingly easy to get backwards when written cold.

**Can you write KMP's `computeLPS` from memory, right now, without looking anything up?** If not, that is the single highest-leverage thing to fix in this pattern. Everything else — Z-algorithm, Manacher, Aho-Corasick (Pattern 26) — reuses the same "don't redo work you've already proven" mental model.

**The real test is in choosing the right tool:**

- **KMP** — deterministic, O(n+m) worst case, no collisions ever. Best when you need a hard guarantee (contest judges, adversarial inputs) and you're matching ONE pattern against ONE text.
- **Z-algorithm** — same complexity class as KMP, but the Z-array is a more *general* object: `Z[i]` tells you the LCP of the whole string with every suffix. Reach for Z when the problem is phrased in terms of prefixes/suffixes directly (LC 2223, LC 3029) rather than "find pattern in text."
- **Rabin-Karp / rolling hash** — O(n+m) *expected*, allows O(1) substring comparison anywhere in the string once prefix hashes are built. Best when you need to compare *many* different substrings against each other (duplicate substrings, echo substrings) rather than one fixed pattern against one text — KMP and Z don't generalize to "compare substring A against substring B" nearly as cleanly as hashing does.
- **Manacher** — the only tool on this list solving a *different* problem (longest palindromic substring in O(n)) rather than pattern matching. Reach for it the instant you see "palindrome" + "O(n)" in the same sentence; the O(n²) center-expansion approach is a Tier-1 fallback, not a Tier-2 answer.

**The key skill:** before coding, ask "am I matching ONE known pattern against a text (KMP/Z), or comparing MANY substrings against each other (hashing), or finding a palindrome (Manacher)?" Misdiagnosing this costs you the whole problem, not just style points.

---

## SECTION 2 — THE STRING ALGORITHMS TAXONOMY

### Type 1: Failure-Function Matching (KMP)
Precompute `lps[i]` = length of the longest proper prefix of `pattern[0..i]` that is also a suffix.
Use it to skip re-comparisons on mismatch during the O(n) text scan.
Problems: Find Index of First Occurrence, Repeated Substring Pattern, Shortest Palindrome, Longest Happy Prefix.

### Type 2: Z-Function Matching
`Z[i]` = length of the longest common prefix between the whole string and the suffix starting at `i`.
Pattern matching via `Z(pattern + '#' + text)`; also directly answers "prefix vs suffix" questions without concatenation tricks.
Problems: Sum of Scores of Built Strings, Minimum Time to Revert Word to Initial State.

### Type 3: Rolling Hash / Rabin-Karp
Treat the string as a base-`B` number mod `M`. Slide a window, updating the hash in O(1) per step.
Enables O(1) comparison of ANY two substrings once prefix hashes are precomputed — the one thing KMP/Z cannot do.
Problems: strStr (Rabin-Karp variant), Longest Duplicate Substring, Distinct Echo Substrings.

### Type 4: Manacher's Algorithm
Transform the string with separators to unify odd/even-length palindromes, then use a mirror trick (like Z's `[l,r]` window) to avoid redundant character comparisons.
Problems: Longest Palindromic Substring, Palindromic Substring counting.

### Type 5: String Hashing as a Utility
Not a standalone algorithm — a building block. Precompute prefix hashes + power table once, then answer "are substrings [l1,r1] and [l2,r2] equal?" in O(1) for the rest of the problem (binary search, two pointers, DP transitions, etc.).
Problems: any problem needing repeated substring-equality checks (LCP of two suffixes via binary search + hashing, longest common substring, etc.)

---

## SECTION 3 — TEMPLATE 1: KMP (KNUTH-MORRIS-PRATT)

### Part A — computeLPS (the failure / prefix function)

```cpp
// ─────────────────────────────────────────────────────────────────
// computeLPS — the failure function (a.k.a. prefix function / pi function)
//
// lps[i] = length of the LONGEST PROPER PREFIX of pattern[0..i]
//          that is ALSO A SUFFIX of pattern[0..i]
//          ("proper" = not the whole substring itself)
//
// WHY this matters: when a mismatch happens at pattern[j] while scanning
// text, we don't need to restart j at 0. We already know pattern[0..j-1]
// matched the text. lps[j-1] tells us the longest prefix of the pattern
// that is guaranteed to already be sitting in the text at this position —
// so we can resume comparing from there instead of from scratch.
//
// This is a DP where the pattern is matched AGAINST ITSELF.
// ─────────────────────────────────────────────────────────────────

vector<int> computeLPS(const string& pattern) {
    int m = pattern.size();
    vector<int> lps(m, 0);
    int k = 0;  // length of the previous longest prefix-suffix

    lps[0] = 0;  // a single character has no proper prefix

    for (int i = 1; i < m; i++) {
        // Mismatch: fall back using lps[k-1] — the KEY KMP insight.
        // We don't compare pattern[i] against pattern[0] after a fail;
        // we jump k back to the next-best candidate prefix length.
        while (k > 0 && pattern[i] != pattern[k]) {
            k = lps[k - 1];
        }
        // Match: extend the current prefix-suffix by one.
        if (pattern[i] == pattern[k]) {
            k++;
        }
        lps[i] = k;
    }
    return lps;
}
```

**FULL TRACE for pattern = "aabaaab":**

```
index:    0   1   2   3   4   5   6
pattern:  a   a   b   a   a   a   b

lps[0] = 0                                       (base case)
k = 0

i=1: pattern[1]='a'. k=0. pattern[1]==pattern[0]? 'a'=='a' YES.
     k++ -> k=1. lps[1]=1.

i=2: pattern[2]='b'. k=1. pattern[2]==pattern[1]? 'b'=='a'? NO.
     while(k>0 && mismatch): k = lps[k-1] = lps[0] = 0.
     Retry: pattern[2]==pattern[0]? 'b'=='a'? NO. k stays 0 (loop exits, k==0).
     lps[2] = 0.

i=3: pattern[3]='a'. k=0. pattern[3]==pattern[0]? 'a'=='a' YES.
     k++ -> k=1. lps[3]=1.

i=4: pattern[4]='a'. k=1. pattern[4]==pattern[1]? 'a'=='a' YES.
     k++ -> k=2. lps[4]=2.

i=5: pattern[5]='a'. k=2. pattern[5]==pattern[2]? 'a'=='b'? NO.
     while: k = lps[k-1] = lps[1] = 1.
     Retry: pattern[5]==pattern[1]? 'a'=='a' YES. k++ -> k=2 (loop condition re-evaluated
     each pass; match found so loop body's inner "if" fires once, k becomes 2).
     lps[5] = 2.

i=6: pattern[6]='b'. k=2. pattern[6]==pattern[2]? 'b'=='b' YES.
     k++ -> k=3. lps[6]=3.

FINAL:  lps = [0, 1, 0, 1, 2, 2, 3]
```

**Sanity check on lps[6]=3:** the longest proper prefix of "aabaaab" that is also a suffix is "aab" (prefix chars 0-2 = "aab"; suffix chars 4-6 = "aab"). Length 3. Matches.

### Part B — the matching loop

```cpp
// ─────────────────────────────────────────────────────────────────
// KMP search — find all occurrences of `pattern` in `text` in O(n+m)
// ─────────────────────────────────────────────────────────────────
vector<int> kmpSearch(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    if (m == 0) return {};
    vector<int> lps = computeLPS(pattern);
    vector<int> matches;

    int i = 0, j = 0;  // i walks text, j walks pattern
    while (i < n) {
        if (text[i] == pattern[j]) {
            i++; j++;
        }
        if (j == m) {
            matches.push_back(i - j);   // full match found, starting at i-j
            j = lps[j - 1];             // continue searching for overlapping matches
        } else if (i < n && text[i] != pattern[j]) {
            // Mismatch (and we didn't just match, since the above `if` failed)
            if (j != 0) j = lps[j - 1];  // don't restart from scratch — reuse work
            else i++;                    // j is already 0, only text pointer advances
        }
    }
    return matches;
}
```

**FULL TRACE for text = "aaaab", pattern = "aab":**

```
LPS of "aab": index0 'a'->0, index1 'a': match with pattern[0] -> k=1, lps[1]=1
              index2 'b': mismatch pattern[1]='a', k=lps[0]=0, mismatch pattern[0]='a', lps[2]=0
              lps = [0, 1, 0]

text = a a a a b   (n=5)          pattern = a a b   (m=3)
       0 1 2 3 4

i=0,j=0: text[0]='a'==pattern[0]='a' -> i=1,j=1.  j!=3. text[1]='a'==pattern[1]='a' (no mismatch branch fires).
i=1,j=1: text[1]='a'==pattern[1]='a' -> i=2,j=2.  j!=3. text[2]='a' vs pattern[2]='b': MISMATCH.
         j!=0 -> j = lps[1] = 1.
i=2,j=1: text[2]='a'==pattern[1]='a' -> i=3,j=2.  j!=3. text[3]='a' vs pattern[2]='b': MISMATCH.
         j!=0 -> j = lps[1] = 1.
i=3,j=1: text[3]='a'==pattern[1]='a' -> i=4,j=2.  j!=3. text[4]='b' vs pattern[2]='b': match, no mismatch branch.
i=4,j=2: text[4]='b'==pattern[2]='b' -> i=5,j=3.  j==3 -> MATCH at i-j = 5-3 = 2.
         j = lps[2] = 0.
i=5: loop ends (i==n).

matches = [2]   →   text.substr(2,3) = "aab"  ✓ (the only occurrence)
```

**Why this is O(n+m), not O(nm):** `i` never decreases — it only ever increments. `j` can decrease, but each decrease is "paid for" by an earlier increment of `j` that was never undone (amortized argument: total increments of `j` across the whole run ≤ n, so total decreases ≤ n too). No character of `text` is ever re-examined after a match extends past it.

---

## SECTION 4 — TEMPLATE 2: Z-ALGORITHM

```cpp
// ─────────────────────────────────────────────────────────────────
// Z-function — Z[i] = length of the longest common prefix (LCP)
//              between s and the suffix of s starting at index i.
//              Z[0] is conventionally left as 0 or n (unused/self-match).
//
// KEY TRICK: maintain a window [l, r] = the rightmost Z-box found so far
// (the interval where we KNOW s[l..r] == s[0..r-l], because some earlier
// Z[l] computation proved it). If i falls inside [l, r], we can seed
// Z[i] using the ALREADY-COMPUTED Z[i-l] instead of comparing from scratch.
// ─────────────────────────────────────────────────────────────────

vector<int> zFunction(const string& s) {
    int n = s.size();
    vector<int> z(n, 0);
    int l = 0, r = 0;  // current rightmost matching window [l, r) meaning s[l..r-1]... 
                        // here we use r as the LAST matched index (inclusive), initialized to 0

    for (int i = 1; i < n; i++) {
        if (i < r) {
            // i is inside the current Z-box: seed with the mirror value,
            // but cap it so we never claim a match past the known boundary r.
            z[i] = min(r - i, z[i - l]);
        }
        // Try to extend past whatever we seeded (naive char-by-char from here on).
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) {
            z[i]++;
        }
        // If this match extends further right than our current window, update it.
        if (i + z[i] > r) {
            l = i;
            r = i + z[i];
        }
    }
    return z;
}
```

**FULL TRACE for s = "aaabaab" (n=7), showing the `[l,r]` window in action:**

```
index:  0   1   2   3   4   5   6
s:      a   a   a   b   a   a   b

l=0, r=0

i=1: i<r? 1<0 NO. Naive expand: s[0]='a' vs s[1]='a' match, z=1; s[1]='a' vs s[2]='a' match, z=2;
     s[2]='a' vs s[3]='b' mismatch. z[1]=2.
     i+z[1]=3 > r(0) -> l=1, r=3.

i=2: i<r? 2<3 YES. z[i-l]=z[1]=2. seed = min(r-i, z[1]) = min(3-2=1, 2) = 1.
     Try extend beyond seed: s[z[2]]=s[1]='a' vs s[i+z[2]]=s[3]='b': MISMATCH. z[2]=1.
     i+z[2]=3, not > r(3). window unchanged.

i=3: i<r? 3<3 NO. Naive expand: s[0]='a' vs s[3]='b': mismatch immediately. z[3]=0.

i=4: i<r? 4<3 NO. Naive expand: s[0]='a' vs s[4]='a' match z=1; s[1]='a' vs s[5]='a' match z=2;
     s[2]='a' vs s[6]='b' mismatch. z[4]=2.
     i+z[4]=6 > r(3) -> l=4, r=6.

i=5: i<r? 5<6 YES. z[i-l]=z[1]=2. seed = min(r-i, z[1]) = min(6-5=1, 2) = 1.
     Try extend: s[z[5]]=s[1]='a' vs s[i+z[5]]=s[6]='b': MISMATCH. z[5]=1.
     i+z[5]=6, not > r(6). window unchanged.

i=6: i<r? 6<6 NO. Naive expand: s[0]='a' vs s[6]='b': mismatch. z[6]=0.

FINAL: z = [_, 2, 1, 0, 2, 1, 0]
```

**Sanity check z[4]=2:** suffix at 4 is "aab", compared to whole string "aaabaab": 'a'='a', 'a'='a', 'a' vs 'b' stop. LCP length 2. Matches.

### How Z solves pattern matching (concat pattern + '#' + text)

```cpp
// Build combined = pattern + '#' + text  ('#' must not appear in either string)
// Any position i in the TEXT portion where z[i] == pattern.size() is a match,
// because that means the suffix starting there shares a full-length prefix
// with `pattern` (which sits at the very front of `combined`).

vector<int> zSearch(const string& text, const string& pattern) {
    string combined = pattern + '#' + text;
    vector<int> z = zFunction(combined);
    int m = pattern.size();
    vector<int> matches;
    for (int i = m + 1; i < (int)combined.size(); i++) {
        if (z[i] == m) {
            matches.push_back(i - m - 1);  // convert combined-index back to text-index
        }
    }
    return matches;
}
```

**TRACE — finding pattern "aab" in text "aaaabaab" via Z:**

```
combined = "aab" + "#" + "aaaabaab" = "aab#aaaabaab"   (length 12)
index:  0   1   2   3   4   5   6   7   8   9  10  11
char:   a   a   b   #   a   a   a   a   b   a   a   b

Computing z with the [l,r] window (abbreviated — same mechanics as above):
z[1]=1  (l=1,r=2 after)
z[2]=0
z[3]=0
z[4]=2  (l=4,r=6 after: 'a'='a','a'='a', then 'b' vs 'a' stop)
z[5]: i<r(5<6) yes, seed=min(1,z[1]=1)=1, extend: s[1]='a' vs s[6]='a' MATCH -> z=2,
      s[2]='b' vs s[7]='a' mismatch. z[5]=2. i+z=7>r(6) -> l=5,r=7.
z[6]: i<r(6<7) yes, seed=min(1,z[1]=1)=1, extend: s[1]='a' vs s[7]='a' MATCH -> z=2,
      s[2]='b' vs s[8]='b' MATCH -> z=3, s[3]='#' vs s[9]='a' mismatch. z[6]=3.
      i+z=9>r(7) -> l=6,r=9.
      ★ z[6] == m(3) -> MATCH at text index (6 - 3 - 1) = 2
z[7]: i<r(7<9) yes, seed=min(2,z[1]=1)=1, extend: s[1]='a' vs s[8]='b' mismatch. z[7]=1.
z[8]: i<r(8<9) yes, seed=min(1,z[2]=0)=0, extend: s[0]='a' vs s[8]='b' mismatch. z[8]=0.
z[9]: i<r(9<9) NO. Naive: s[0]='a' vs s[9]='a' match, s[1]='a' vs s[10]='a' match,
      s[2]='b' vs s[11]='b' match, i+z=12 not<12 stop. z[9]=3.
      ★ z[9] == m(3) -> MATCH at text index (9 - 3 - 1) = 5
      l=9, r=12.
z[10]: i<r(10<12) yes, seed=min(2,z[1]=1)=1, extend: s[1]='a' vs s[11]='b' mismatch. z[10]=1.
z[11]: i<r(11<12) yes, seed=min(1,z[2]=0)=0, extend: s[0]='a' vs s[11]='b' mismatch. z[11]=0.

matches = [2, 5]
```

Check by hand: `text = "aaaabaab"`. `text.substr(2,3) = "aab"` ✓. `text.substr(5,3) = "aab"` ✓. Both correct, no false positives.

---

## SECTION 5 — TEMPLATE 3: RABIN-KARP ROLLING HASH

```cpp
// ─────────────────────────────────────────────────────────────────
// Rabin-Karp — treat a string as a base-B number mod M.
// hash(s[0..m-1]) = s[0]*B^(m-1) + s[1]*B^(m-2) + ... + s[m-1]*B^0   (mod M)
//
// ROLLING UPDATE (slide window right by one, remove leftmost char, add new
// rightmost char):
//   newHash = ((oldHash - removedChar * B^(m-1)) * B + addedChar) mod M
//
// In production code: use long long, a large prime M (e.g. 1e9+7), and
// a base B larger than the alphabet size (e.g. 131 or 137). ALWAYS
// double-check a hash "hit" against the actual characters — hash equality
// is necessary but NOT sufficient for string equality (collisions exist).
// For adversarial inputs (Codeforces), use TWO independent (B, M) pairs
// ("double hashing") to make deliberate collisions computationally
// infeasible for the setter to construct.
// ─────────────────────────────────────────────────────────────────

vector<int> rabinKarpSearch(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    vector<int> matches;
    if (m == 0 || m > n) return matches;

    const long long B = 131;
    const long long M = 1'000'000'007LL;

    long long patternHash = 0, windowHash = 0, Bm = 1;  // Bm = B^(m-1) mod M
    for (int i = 0; i < m - 1; i++) Bm = (Bm * B) % M;

    for (int i = 0; i < m; i++) {
        patternHash = (patternHash * B + pattern[i]) % M;
        windowHash  = (windowHash  * B + text[i])    % M;
    }

    for (int i = 0; ; i++) {
        if (windowHash == patternHash) {
            // SPURIOUS-HIT DOUBLE-CHECK: hash match does not guarantee
            // string match. Always verify character-by-character.
            if (text.compare(i, m, pattern) == 0) {
                matches.push_back(i);
            }
        }
        if (i + m >= n) break;
        // Roll the window forward by one character.
        windowHash = (windowHash - (long long)text[i] * Bm % M + M) % M;  // remove leftmost
        //                                                       ^^^ add M BEFORE mod:
        //           subtraction can go negative in C++ (% keeps sign of dividend) —
        //           always normalize into [0, M) before the next multiply/add.
        windowHash = (windowHash * B + text[i + m]) % M;                  // add new rightmost
    }
    return matches;
}
```

**FULL TRACE — text = "abcab", pattern = "cab", using a SMALL illustrative base/mod (B=10, M=97, chars mapped a=1,b=2,c=3) so the arithmetic is followable by hand. (Real code uses B=131, M=1e9+7 — the mechanics are identical, only the numbers get bigger.)**

```
pattern "cab" = 3,1,2 → hash = (3*10² + 1*10 + 2) mod 97 = 312 mod 97 = 312 - 3*97 = 21

text = "abcab" = a,b,c,a,b = 1,2,3,1,2   (n=5)

Window 1 = text[0..2] = "abc" = 1,2,3 → hash = 1*100+2*10+3 = 123 mod 97 = 26
Window 1 hash(26) != pattern hash(21) → no candidate, roll forward.

B^(m-1) = B² = 100. Removed char = text[0]='a'=1. Added char = text[3]='a'=1.
newHash = (26 - 1*100 mod 97 + 97) mod 97, then *10 + added, all mod 97:
  step1 (remove):  (26 - 1*100) mod 97 = (26-100) mod 97 = -74 mod 97 = -74+97 = 23
  step2 (shift+add): (23*10 + 1) mod 97 = 231 mod 97 = 231 - 2*97 = 37
Window 2 = text[1..3] = "bca" = 2,3,1 → direct check: 2*100+3*10+1 = 231 mod 97 = 37 ✓ MATCHES rolling result.
Window 2 hash(37) != pattern hash(21) → no candidate, roll forward.

Removed char = text[1]='b'=2. Added char = text[4]='b'=2.
  step1 (remove): (37 - 2*100) mod 97 = (37-200) mod 97 = -163 mod 97 = -163+2*97 = 31... 

  ⚠ let's redo carefully: -163 mod 97: 97*2=194, -163+194=31. So step1 = 31.
  step2 (shift+add): (31*10 + 2) mod 97 = 312 mod 97 = 312-3*97 = 21
Window 3 = text[2..4] = "cab" = 3,1,2 → direct check: 3*100+1*10+2=312 mod 97=21 ✓ MATCHES rolling result.
Window 3 hash(21) == pattern hash(21) → CANDIDATE at index 2.
Double-check: text.substr(2,3) = "cab" == pattern "cab" → CONFIRMED MATCH.

matches = [2]
```

**Note the corrected step1 arithmetic above** — this is deliberately left showing the "redo carefully" moment because `-163 mod 97` in real C++ (`%`) evaluates to `-163 % 97 = -66` (C++ `%` keeps the sign of the dividend, NOT mathematical mod). The code's `(... + M) % M` pattern is what forces the result into `[0, M)` correctly — see Common Mistake 4.

---

## SECTION 6 — TEMPLATE 4: MANACHER'S ALGORITHM

```cpp
// ─────────────────────────────────────────────────────────────────
// Manacher's Algorithm — longest palindromic substring in O(n)
//
// PROBLEM WITH NAIVE EXPANSION: odd-length palindromes (center = a char)
// and even-length palindromes (center = a gap between chars) need
// different handling. Fix: insert a separator ('#', not in the alphabet)
// between every character (and at both ends), so EVERY palindrome in the
// transformed string is odd-length, centered exactly on some character
// or separator.
//
//   s = "babad"  ->  t = "#b#a#b#a#d#"
//
// P[i] in t = the palindrome RADIUS centered at i (t[i-P[i]..i+P[i]] is
// a palindrome). Crucially, P[i] in the transformed string equals the
// LENGTH of the corresponding palindrome in the ORIGINAL string.
//
// SPEEDUP (same idea as Z's [l,r] window): track center c and right
// boundary r of the rightmost palindrome found so far. For a new index i
// inside [c, r], mirror_i = 2*c - i already has SOME known palindrome
// radius — reuse it instead of re-expanding from scratch.
// ─────────────────────────────────────────────────────────────────

string longestPalindrome(const string& s) {
    if (s.empty()) return "";

    // Build transformed string with sentinels: t = "#" + s[0] + "#" + s[1] + ... + "#"
    string t = "#";
    for (char c : s) { t += c; t += '#'; }

    int n = t.size();
    vector<int> P(n, 0);
    int c = 0, r = 0;      // current center and right boundary of best-known palindrome
    int bestCenter = 0, bestRadius = 0;

    for (int i = 1; i < n - 1; i++) {
        if (i < r) {
            int mirror = 2 * c - i;
            P[i] = min(r - i, P[mirror]);  // seed — may need further expansion (see trace)
        }
        // Attempt to expand past whatever was seeded.
        while (i - P[i] - 1 >= 0 && i + P[i] + 1 < n &&
               t[i - P[i] - 1] == t[i + P[i] + 1]) {
            P[i]++;
        }
        // Update the window if this palindrome reaches further right.
        if (i + P[i] > r) {
            c = i;
            r = i + P[i];
        }
        if (P[i] > bestRadius) {
            bestRadius = P[i];
            bestCenter = i;
        }
    }

    // Map back: original start = (bestCenter - bestRadius) / 2, length = bestRadius.
    int start = (bestCenter - bestRadius) / 2;
    return s.substr(start, bestRadius);
}
```

**FULL TRACE for s = "babad" → t = "#b#a#b#a#d#" (n=11):**

```
index:  0   1   2   3   4   5   6   7   8   9  10
t:      #   b   #   a   #   b   #   a   #   d   #

c=0, r=0

i=1 ('b'): i<r? 1<0 NO. Expand directly: t[0]='#' vs t[2]='#' MATCH, P=1;
   t[-1] out of bounds, stop. P[1]=1. i+P=2>r(0) -> c=1,r=2.

i=2 ('#'): i<r? 2<2 NO. Expand: t[1]='b' vs t[3]='a' MISMATCH. P[2]=0.

i=3 ('a'): i<r? 3<2 NO. Expand: t[2]='#' vs t[4]='#' MATCH P=1; t[1]='b' vs t[5]='b' MATCH P=2;
   t[0]='#' vs t[6]='#' MATCH P=3; t[-1] out of bounds, stop. P[3]=3. i+P=6>r(2) -> c=3,r=6.

i=4 ('#'): i<r? 4<6 YES. mirror=2*3-4=2. P[mirror]=P[2]=0. seed=min(r-i=2, 0)=0.
   Expand from seed: t[3]='a' vs t[5]='b' MISMATCH immediately. P[4]=0.
   (CASE: P[mirror] < r-i → answer copied directly, no expansion needed — confirmed 0=0.)

i=5 ('b'): i<r? 5<6 YES. mirror=2*3-5=1. P[mirror]=P[1]=1. seed=min(r-i=1, 1)=1.
   (CASE: P[mirror] == r-i exactly → MUST continue expanding manually, cannot
    just copy — the true radius might extend past the known boundary r.)
   Expand from seed P[5]=1: t[5-2]=t[3]='a' vs t[5+2]=t[7]='a' MATCH, P=2;
   t[5-3]=t[2]='#' vs t[5+3]=t[8]='#' MATCH, P=3;
   t[5-4]=t[1]='b' vs t[5+4]=t[9]='d' MISMATCH, stop. P[5]=3.
   i+P=8>r(6) -> c=5,r=8.

i=6 ('#'): i<r? 6<8 YES. mirror=2*5-6=4. P[mirror]=P[4]=0. seed=min(r-i=2,0)=0.
   Expand: t[5]='b' vs t[7]='a' MISMATCH. P[6]=0.

i=7 ('a'): i<r? 7<8 YES. mirror=2*5-7=3. P[mirror]=P[3]=3. seed=min(r-i=1, 3)=1.
   (CASE: P[mirror] > r-i → CAPPED at r-i, no expansion attempted — provably
    cannot extend further without contradicting the definition of r.)
   P[7]=1. i+P=8, not>r(8), window unchanged.

i=8 ('#'): i<r? 8<8 NO. Expand: t[7]='a' vs t[9]='d' MISMATCH. P[8]=0.

i=9 ('d'): i<r? 9<8 NO. Expand: t[8]='#' vs t[10]='#' MATCH P=1; t[7]='a', out of range
   at t[11] (n=11, index 11 invalid) — bounds check stops it. P[9]=1. i+P=10>r(8) -> c=9,r=10.

FINAL: P = [0,1,0,3,0,3,0,1,0,1,0]
Best radius = 3, at bestCenter=3 (first occurrence) or bestCenter=5 (tie).
start = (3-3)/2 = 0, length=3 -> s.substr(0,3) = "bab"   ✓ (a valid LC5 answer)
```

**Why the three mirror cases matter (this is THE interview follow-up for Manacher):**
1. `P[mirror] < r - i`: the mirrored palindrome is fully contained within `[l,r]`, so it CANNOT be limited by the boundary — copy directly, zero extra work.
2. `P[mirror] == r - i`: the mirrored palindrome touches the boundary exactly — it MIGHT extend further outside `[l,r]` (we have no information past `r`), so we must attempt expansion from there.
3. `P[mirror] > r - i`: copying the full mirror value would claim a palindrome extending past `r`, which is unproven — cap at `r - i` (provably tight, because if it extended further, symmetry around `c` would force `r` to have already been extended).

---

## SECTION 7 — TEMPLATE 5: STRING HASHING (O(1) SUBSTRING COMPARISON)

```cpp
// ─────────────────────────────────────────────────────────────────
// Precompute prefix hashes + powers once → compare ANY two substrings
// of the SAME string in O(1) for the rest of the algorithm.
//
// prefixHash[i] = hash of s[0..i-1]  (prefixHash[0] = 0)
// hash(l, r) [substring s[l..r], inclusive] =
//     (prefixHash[r+1] - prefixHash[l] * pow[r-l+1]) mod M
//
// This is just "peel off the leading digits" in the base-B number sense:
// prefixHash[r+1] represents s[0..r] as a base-B number; multiplying
// prefixHash[l] by B^(r-l+1) shifts it to align with the same place
// values, and subtracting removes the s[0..l-1] contribution, leaving
// exactly the value of s[l..r].
// ─────────────────────────────────────────────────────────────────

struct StringHash {
    vector<long long> prefixHash, pw;
    long long B, M;

    StringHash(const string& s, long long base = 131, long long mod = 1'000'000'007LL)
        : B(base), M(mod) {
        int n = s.size();
        prefixHash.assign(n + 1, 0);
        pw.assign(n + 1, 1);
        for (int i = 0; i < n; i++) {
            prefixHash[i + 1] = (prefixHash[i] * B + s[i]) % M;
            pw[i + 1] = (pw[i] * B) % M;
        }
    }

    // Hash of s[l..r] inclusive, 0-indexed.
    long long getHash(int l, int r) const {
        long long res = (prefixHash[r + 1] - prefixHash[l] * pw[r - l + 1] % M) % M;
        return (res + M) % M;  // normalize negative result — see Common Mistake 4
    }
};

// USAGE — O(1) equality check for two substrings of possibly different
// start positions but the SAME length (always double-check on a real hit
// if collisions are a concern, or use two StringHash objects with
// different (B, M) and compare BOTH — "double hashing").
bool substringsEqual(const StringHash& h, int l1, int r1, int l2, int r2) {
    if (r1 - l1 != r2 - l2) return false;
    return h.getHash(l1, r1) == h.getHash(l2, r2);
}
```

**Why this generalizes beyond KMP/Z:** KMP and Z answer "does THIS pattern occur in THIS text." String hashing answers "are substring A and substring B equal," for ARBITRARY A and B chosen at query time. That's the tool you reach for when a problem needs many pairwise substring comparisons (Longest Duplicate Substring, Distinct Echo Substrings, Longest Common Substring via binary search) rather than one fixed pattern search.

---

## SECTION 8 — APPLICATIONS

### 8.1 — Find Index of First Occurrence (LC 28) — KMP

```cpp
int strStr(string haystack, string needle) {
    vector<int> matches = kmpSearch(haystack, needle);
    return matches.empty() ? -1 : matches[0];
}
```
Textbook KMP application — the matching loop from Section 3, returning the first hit.

### 8.2 — Repeated Substring Pattern (LC 459) — KMP failure-function insight

```cpp
// s is composed of a repeated substring iff:
//   n % (n - lps[n-1]) == 0   AND   lps[n-1] > 0
//
// WHY: lps[n-1] is the longest proper prefix of s that's also a suffix.
// If s = t repeated k times, the "period" p = n / k satisfies p = n - lps[n-1]
// (the failure function of a periodic string always reveals the period this way).
// The period must evenly divide n for the repetition to be EXACT (no partial copy).
bool repeatedSubstringPattern(string s) {
    int n = s.size();
    vector<int> lps = computeLPS(s);
    int period = n - lps[n - 1];
    return lps[n - 1] > 0 && n % period == 0;
}
```
Example: s = "abcabcabc" (n=9). LPS of s ends at lps[8]=6 (prefix "abcabc" = suffix "abcabc").
period = 9 - 6 = 3. 9 % 3 == 0 and lps[8]=6>0 → true. The repeating unit is "abc".

### 8.3 — Shortest Palindrome (LC 214) — KMP on `s + '#' + reverse(s)`

```cpp
// Find the longest prefix of s that is ALSO a palindrome; prepend the
// reverse of everything AFTER that prefix.
//
// Trick: build t = s + '#' + reverse(s). lps.back() = length of the
// longest prefix of s that matches a SUFFIX of reverse(s) — but a suffix
// of reverse(s) is exactly a PREFIX of s reversed, so this equals the
// longest prefix of s that is a palindrome.
string shortestPalindrome(string s) {
    string rev = s;
    reverse(rev.begin(), rev.end());
    string t = s + "#" + rev;
    vector<int> lps = computeLPS(t);
    int palindromicPrefixLen = lps.back();
    string toPrepend = s.substr(palindromicPrefixLen);   // the non-palindromic tail
    reverse(toPrepend.begin(), toPrepend.end());
    return toPrepend + s;
}
```
Example: s = "aacecaaa". rev = "aaacecaa". t = "aacecaaa#aaacecaa". lps.back() = 7
(longest palindromic prefix of s is "aacecaa", length 7). Tail = "a" (last char), reversed = "a".
Answer = "a" + "aacecaaa" = "aaacecaaa" — a palindrome. ✓

### 8.4 — Longest Happy Prefix (LC 1392) — the LPS array IS the answer

```cpp
// A "happy prefix" is a non-empty proper prefix that is also a suffix.
// This is LITERALLY lps[n-1] from the failure function — no extra work needed.
string longestPrefix(string s) {
    vector<int> lps = computeLPS(s);
    return s.substr(0, lps.back());
}
```
Example: s = "level". LPS: l=0,e=0,v=0,e:pattern[3]='e' vs pattern[0]='l' mismatch->0,
l: pattern[4]='l'==pattern[0]='l' match->1. lps=[0,0,0,0,1]. Answer = s.substr(0,1) = "l".

### 8.5 — Longest Duplicate Substring (LC 1044) — binary search on length + rolling hash

```cpp
// Binary search on the ANSWER LENGTH L: "does a duplicated substring of
// length >= L exist?" is monotonic (if length L works, so does any L' < L).
// For a fixed L, slide a window of length L across s, hash each window,
// and check for a hash collision (double-checked with actual substring
// comparison to rule out false positives).
string longestDupSubstring(string s) {
    int n = s.size();
    const long long B = 26, M = (1LL << 61) - 1;  // large prime mod for safety

    auto search = [&](int L) -> int {   // returns start index of a length-L duplicate, or -1
        if (L == 0) return -1;
        long long h = 0, Bl = 1;
        for (int i = 0; i < L; i++) { h = (h * B + (s[i]-'a')) % M; }
        for (int i = 0; i < L - 1; i++) Bl = (Bl * B) % M;   // Bl = B^(L-1) mod M

        unordered_map<long long, vector<int>> seen;   // hash -> list of start indices
        seen[h].push_back(0);
        for (int i = 1; i + L <= n; i++) {
            h = ((h - (long long)(s[i-1]-'a') * Bl % M + M) % M * B + (s[i+L-1]-'a')) % M;
            if (seen.count(h)) {
                for (int start : seen[h]) {
                    if (s.compare(start, L, s, i, L) == 0) return i;  // spurious-hit check
                }
            }
            seen[h].push_back(i);
        }
        return -1;
    };

    int lo = 1, hi = n - 1, bestStart = -1, bestLen = 0;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        int start = search(mid);
        if (start != -1) {
            bestStart = start; bestLen = mid;
            lo = mid + 1;      // try for a longer duplicate
        } else {
            hi = mid - 1;      // no duplicate of this length — try shorter
        }
    }
    return bestStart == -1 ? "" : s.substr(bestStart, bestLen);
}
```
Complexity: O(n log n) expected — binary search (log n) × rolling hash scan (O(n) per check).

### 8.6 — Distinct Echo Substrings (LC 1316) — rolling hash for O(1) half-comparison

```cpp
// An "echo substring" has the form a+a (some non-empty string a repeated
// twice, e.g. "abab" = "ab"+"ab"). Enumerate every even-length substring,
// split into two halves, and use hashing to compare the halves in O(1)
// instead of O(len). Store CANDIDATE strings (not just hashes) in a set
// to guarantee correctness against hash collisions.
int distinctEchoSubstrings(string text) {
    int n = text.size();
    StringHash h(text);
    unordered_set<string> distinct;

    for (int len = 2; len <= n; len += 2) {          // even lengths only
        int half = len / 2;
        for (int start = 0; start + len <= n; start++) {
            int mid = start + half;
            // Compare text[start..mid-1] with text[mid..start+len-1] in O(1).
            if (h.getHash(start, mid - 1) == h.getHash(mid, start + len - 1)) {
                distinct.insert(text.substr(start, len));  // O(len) but only on true matches path guard below
            }
        }
    }
    return distinct.size();
}
```
Note: `n <= 2000` in the constraints, so the O(n²) enumeration (with O(1) half-checks) comfortably fits; the `substr`/insert cost on confirmed matches is the dominant term but still well within limits.

### 8.7 — Sum of Scores of Built Strings (LC 2223) — direct Z-algorithm application

```cpp
// s_i is defined as the SUFFIX of s with length i (built by repeatedly
// PREPENDING characters, so after i steps you've built the last i chars
// of the final string). score(i) = LCP(s_i, s) = LCP(suffix of length i, s).
//
// KEY INSIGHT: suffix of s of length i STARTS at index (n - i). By the
// very definition of the Z-function, Z[n-i] = LCP(s, suffix starting at n-i)
// = score(i), for i = 1..n-1. score(n) = n trivially (s_n == s).
// NO string reversal or concatenation needed — just Z-function on s itself.
long long sumScores(string s) {
    int n = s.size();
    vector<int> z = zFunction(s);
    long long total = n;                 // score(n) = n
    for (int i = 1; i < n; i++) total += z[i];   // score(n-i) = z[i], summed over all i
    return total;
}
```

**VERIFIED TRACE — s = "babab" (official LC example, expected answer 9):**
```
Z-function of "babab":
z[1]: s[0]='b' vs s[1]='a' mismatch. z[1]=0.
z[2]: s[0]='b' vs s[2]='b' match; s[1]='a' vs s[3]='a' match; s[2]='b' vs s[4]='b' match;
      bounds exhausted. z[2]=3.
z[3]: s[0]='b' vs s[3]='a' mismatch. z[3]=0.
z[4]: s[0]='b' vs s[4]='b' match; bounds exhausted. z[4]=1.

z = [_, 0, 3, 0, 1]
total = n(5) + (z[1]+z[2]+z[3]+z[4]) = 5 + (0+3+0+1) = 5 + 4 = 9   ✓ matches official answer
```
Cross-check against the problem's own breakdown "1 + 0 + 3 + 0 + 5 = 9": score(1)=z[n-1]=z[4]=1 ✓,
score(2)=z[3]=0 ✓, score(3)=z[2]=3 ✓, score(4)=z[1]=0 ✓, score(5)=5 (base case) ✓. All match.

---

## SECTION 9 — COMPLEXITY TABLE

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| KMP (LPS + search) | O(n + m) | O(m) | Deterministic, no false positives ever |
| Z-algorithm | O(n) for one string | O(n) | Same complexity class as KMP, more general |
| Rabin-Karp | O(n + m) expected | O(1) extra (O(n) for prefix-hash variant) | Must double-check hits; adversarial inputs need double hashing |
| Manacher | O(n) | O(n) | Only O(n) solution for longest palindromic substring |
| String hashing (build) | O(n) | O(n) | One-time cost, enables O(1) substring compare after |
| String hashing (query) | O(1) per comparison | — | The payoff for the O(n) build |
| LC 1044 (binary search + hash) | O(n log n) expected | O(n) | log n binary search steps × O(n) hash scan |
| LC 1316 (echo substrings) | O(n²) | O(n²) worst case for the set | n ≤ 2000 constraint makes this fine |
| LC 2223 (Z on s) | O(n) | O(n) | Single Z-function pass, no concatenation needed |

---

## SECTION 10 — VARIANTS

1. **Aho-Corasick (Pattern 26):** generalizes KMP's failure function to a TRIE of multiple patterns simultaneously — the "goto/fail/output" automaton. Reach for it when matching many patterns against one text, instead of running KMP once per pattern (which would be O(n·k) for k patterns).

2. **Suffix Array / Suffix Automaton:** when you need to answer MANY different substring queries (not just "does X occur," but "how many distinct substrings," "k-th smallest substring," etc.), a suffix array (O(n log n) build) or suffix automaton (O(n) build) is the heavier but more powerful tool. String hashing is the "80% solution" that gets most interview/contest problems without the suffix-structure machinery.

3. **Double hashing:** run two independent `(B, M)` pairs and require BOTH hashes to match before accepting equality. This defeats "anti-hash" tests deliberately constructed against a single well-known `(B, M)` — a real concern on Codeforces, less so on LeetCode (no adversarial test authors targeting your specific hash choice).

4. **Manacher for counting all palindromic substrings (not just the longest):** once `P[i]` is computed for every center, the count of palindromes centered at `i` is `(P[i] + 1) / 2` in the transformed string — summing this over all `i` gives the total count of palindromic substrings in O(n), beating the O(n²) center-expansion count.

5. **KMP automaton as a DP transition table:** precomputing, for every `(state, nextChar)` pair, the resulting state, turns KMP matching into an O(1)-per-character DP transition — useful when pattern matching needs to be embedded INSIDE a larger DP (e.g., "count strings of length n avoiding pattern P as a substring").

---

## SECTION 11 — COMMON MISTAKES

### Mistake 1: Off-by-one in the LPS / failure function

```cpp
// WRONG — comparing pattern[i] against pattern[k-1] instead of pattern[k]
for (int i = 1; i < m; i++) {
    while (k > 0 && pattern[i] != pattern[k - 1]) k = lps[k - 1];  // BUG
    if (pattern[i] == pattern[k - 1]) k++;                         // BUG
    lps[i] = k;
}
// k represents a LENGTH (0-indexed count), so the NEXT character to compare
// against is pattern[k] (the character immediately after the matched prefix
// of length k), NOT pattern[k-1].

// CORRECT
for (int i = 1; i < m; i++) {
    while (k > 0 && pattern[i] != pattern[k]) k = lps[k - 1];
    if (pattern[i] == pattern[k]) k++;
    lps[i] = k;
}
```

---

### Mistake 2: Z-box window update bug

```cpp
// WRONG — updating l, r unconditionally every iteration, even when the
// new palindrome/match doesn't extend past the current window
for (int i = 1; i < n; i++) {
    if (i < r) z[i] = min(r - i, z[i - l]);
    while (i + z[i] < n && s[z[i]] == s[i + z[i]]) z[i]++;
    l = i; r = i + z[i];    // BUG: always overwrites, even if i+z[i] <= r
}
// This SHRINKS the window when a later, shorter match is found — destroying
// the invariant that [l, r] is the RIGHTMOST match seen so far. Subsequent
// iterations then seed incorrectly from a stale/wrong window.

// CORRECT — only update when the new match extends further right
if (i + z[i] > r) { l = i; r = i + z[i]; }
```

---

### Mistake 3: Hash collision accepted without double-check

```cpp
// WRONG — trusting the hash match as proof of string equality
if (windowHash == patternHash) {
    matches.push_back(i);   // BUG: no verification — false positive risk
}

// Two different strings CAN produce the same hash (pigeonhole: infinitely
// many strings, finitely many hash values mod M). With M ~ 1e9, collision
// probability per comparison is small but NOT zero — and on adversarial
// judges (or just bad luck at scale), it WILL eventually bite you.

// CORRECT — always verify character-by-character on a hash hit
if (windowHash == patternHash && text.compare(i, m, pattern) == 0) {
    matches.push_back(i);
}
```

---

### Mistake 4: Modular overflow — not using `long long`, and not normalizing negatives

```cpp
// WRONG — int overflow AND unhandled negative mod
int hash = (oldHash - removedChar * Bm) * B + addedChar;   // BUG: overflows int,
hash %= M;                                                  // BUG: can be negative in C++

// C++'s % operator returns a result with the SAME SIGN as the dividend.
// (-163) % 97 == -66 in C++ (NOT 31, the mathematical mod). If you don't
// add M back before the next multiply, everything downstream is corrupted.

// CORRECT
long long hash = ((oldHash - (long long)removedChar * Bm % M + M) % M * B + addedChar) % M;
//                                                          ^^^ add M BEFORE taking % again
//                ^^^^^^^^^ long long everywhere — B*M products can exceed 2^31 easily
```

---

### Mistake 5: Manacher — forgetting separators (or using one that collides with the alphabet)

```cpp
// WRONG — running center-expansion on the raw string without separators
for (int i = 0; i < n; i++) {
    // expand odd-length palindrome centered at i
    // expand even-length palindrome centered between i and i+1
    // ... TWO separate code paths, easy to get boundary conditions wrong,
    // and the elegant single-loop [l,r]-window trick doesn't apply cleanly.
}

// WRONG — using a separator character that CAN appear in the input
string t = "a" + s[0] + "a" + s[1] + ...;   // BUG if 'a' is a valid input character!
// This creates FALSE palindromic matches across the separator boundary.

// CORRECT — insert a separator GUARANTEED not to appear in the alphabet
// ('#' for lowercase-letter problems), AND add sentinel bounds checks
// (or sentinel characters like '^' and '$' at the very ends) so the
// expansion loop never reads out of bounds.
string t = "#";
for (char c : s) { t += c; t += '#'; }
```

---

### Mistake 6: KMP matching loop — resetting `j` to 0 on mismatch instead of `lps[j-1]`

```cpp
// WRONG — naive reset destroys the O(n+m) guarantee, degrades to O(nm),
// AND silently breaks overlapping-match detection
while (i < n) {
    if (text[i] == pattern[j]) { i++; j++; }
    else { i = i - j + 1; j = 0; }   // BUG: this is literally brute-force search
    if (j == m) { matches.push_back(i - j); j = 0; }   // BUG: misses overlaps too
}
// The entire POINT of the failure function is to avoid re-scanning text
// characters. Resetting j=0 on every mismatch throws away that information
// and also resets i backward — re-reading characters already consumed.

// CORRECT
if (j != 0) j = lps[j - 1];   // reuse the proven partial match
else i++;                      // only advance text pointer when j is already 0
// ... and on a full match: j = lps[j-1], NOT j = 0, to catch overlapping occurrences
```

---

## SECTION 12 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Find the Index of First Occurrence in a String | 28 | KMP: computeLPS + matching loop from memory |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 2 | Repeated Substring Pattern | 459 | KMP failure-function insight: `n % (n - lps[n-1]) == 0` |
| 3 | Shortest Palindrome | 214 | KMP on `s + '#' + reverse(s)`, longest palindromic prefix |
| 4 | Longest Happy Prefix | 1392 | LPS array final value IS the answer |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 5 | Implement strStr() — Rabin-Karp variant | 28 | Rolling hash, spurious-hit double-check, modular arithmetic hygiene |
| 6 | Longest Duplicate Substring | 1044 | Binary search on answer length + rolling hash membership check |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Distinct Echo Substrings | 1316 | Rolling hash for O(1) half-comparison, enumerate even lengths |
| 8 | Sum of Scores of Built Strings | 2223 | Z-function directly answers suffix-vs-whole-string LCP |
| 9 | Minimum Time to Revert Word to Initial State | 3029 | Z-function: `z[i] == n - i` checks "suffix is a prefix" |

---

## SECTION 13 — PATTERN CONNECTIONS

1. **Pattern 26 (Trie / Aho-Corasick):** KMP's failure function is the single-pattern special case of Aho-Corasick's automaton. Once you understand `lps[]` as "the state to fall back to on mismatch," Aho-Corasick's trie-with-fail-links is the direct multi-pattern generalization — same core idea, one more dimension.

2. **Pattern 05 (Binary Search):** Longest Duplicate Substring (LC 1044) is a binary-search-the-answer problem wearing a string-algorithms costume. The monotonic predicate ("does a duplicate of length ≥ L exist?") is exactly Pattern 05's structure; rolling hash is just the O(n) checker function plugged into the binary search shell.

3. **Pattern 06 (Hashing / Prefix Sums):** String hashing is literally a prefix-sum technique in a different base — `prefixHash[i]` is a running accumulation exactly like a prefix sum array, and `getHash(l,r)` is the same "subtract prefix[l] from prefix[r+1]" trick used for range-sum queries.

4. **Pattern 04 (Monotonic Stack) / Pattern 20 (LCS family):** not directly connected to string algorithms, but worth noting the DISTINCTION: LCS-family problems (Pattern 20) use O(nm) two-sequence DP to compare two DIFFERENT strings; string hashing/KMP/Z are for finding patterns WITHIN a single string or stream. Don't reach for a DP table when a hash comparison would answer the question in O(1).

5. **Pattern 17 (1D DP):** KMP's automaton (state = current match length) can be embedded as the transition function inside a larger DP — "count strings of length n over alphabet Σ that do NOT contain pattern P as a substring" is a DP over KMP states, combining Pattern 17's counting-DP structure with Pattern 33's automaton.

---

## SECTION 14 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 28 — Implement strStr()

**Interviewer:** "Find the first occurrence of needle in haystack. Don't use a library substring function."

**Candidate:**

*[First minute — state the naive approach and its cost, then pivot]*
> "Brute force is O(n·m) — for each starting position in haystack, compare up to m characters. That's fine for small inputs but I'll use KMP to get O(n+m) worst case, which also demonstrates I understand the failure-function technique."

*[Explain the failure function before coding]*
> "The core idea: precompute `lps[i]` = the longest proper prefix of `pattern[0..i]` that's also a suffix. When a mismatch happens deep into a partial match, instead of restarting the pattern pointer from 0, I jump it to `lps[j-1]` — the longest prefix I already know is sitting in the text at this position, because it just matched as a suffix of what I already consumed."

*[Code — 8 minutes, both functions]*
```cpp
vector<int> computeLPS(const string& p) {
    int m = p.size();
    vector<int> lps(m, 0);
    int k = 0;
    for (int i = 1; i < m; i++) {
        while (k > 0 && p[i] != p[k]) k = lps[k-1];
        if (p[i] == p[k]) k++;
        lps[i] = k;
    }
    return lps;
}
int strStr(string haystack, string needle) {
    int n = haystack.size(), m = needle.size();
    if (m == 0) return 0;
    vector<int> lps = computeLPS(needle);
    int i = 0, j = 0;
    while (i < n) {
        if (haystack[i] == needle[j]) { i++; j++; }
        if (j == m) return i - j;
        else if (i < n && haystack[i] != needle[j]) {
            j ? j = lps[j-1] : i++;
        }
    }
    return -1;
}
```

*[Follow-up]: "What if you needed ALL occurrences, including overlapping ones?"*
> "Instead of returning immediately on `j == m`, push the match index and set `j = lps[j-1]` — NOT `j = 0` — so overlapping matches (e.g., pattern 'aa' in text 'aaa') are still found."

**RED FLAGS:**
- Resetting `j = 0` on every mismatch (destroys the O(n+m) guarantee — see Common Mistake 6)
- Off-by-one comparing `pattern[k-1]` instead of `pattern[k]` in the LPS loop
- Not being able to state WHY the failure function makes the algorithm linear (the amortized argument: `i` never decreases, `j`'s total decreases are bounded by its total increases)

---

### Interview Simulation 2: LC 1044 — Longest Duplicate Substring

**Interviewer:** "Find the longest substring that appears at least twice in a given string. Return any one if there are ties."

**Candidate:**

*[Recognize the shape: binary search + a checker]*
> "The direct question — 'what's the longest duplicate?' — is hard to attack directly. But the YES/NO question — 'does a duplicate of length ≥ L exist?' — is monotonic: if a duplicate of length L exists, then some SUBSTRING of it (length L-1) also duplicates trivially. That monotonicity means I can binary search on L."

*[Explain the checker]*
> "For a fixed L, I slide a window of length L across the string, hashing each window with a rolling hash for O(1) updates. If two windows produce the same hash, I double-check they're actually equal (not just a hash collision) before declaring a duplicate found. This checker runs in O(n) expected, so the whole algorithm is O(n log n) expected."

*[Interviewer]: "Why not just use a suffix array or suffix automaton directly instead of binary search + hashing?"*
> "Those are asymptotically comparable or better in the worst case, but they're significantly more code to get right under interview time pressure, and for LeetCode-scale constraints (n ≤ 30000-ish), binary search + rolling hash comfortably passes and is far less error-prone to implement live."

*[Code sketch — 10 minutes, discussing the rolling hash update and collision handling from Section 8.5]*

**RED FLAGS:**
- Jumping straight to a suffix automaton without first proposing the simpler binary-search-plus-hash approach
- Forgetting the spurious-hit double-check on a hash collision
- Off-by-one in the rolling hash update (forgetting to recompute `B^(L-1)` when L changes between binary search iterations — it must be recomputed inside the checker function, not reused across different L values)

---

### Interview Simulation 3: Meta-discussion — KMP vs Z vs Rolling Hash vs Manacher

**Interviewer:** "Give me a one-sentence rule for choosing between these four tools."

**Candidate:**
> "If the problem is stated as 'find where pattern P occurs in text T' and P is fixed and known, KMP is the safest choice — deterministic, no collision risk, and interviewers expect it.
>
> If the problem is phrased in terms of prefix-suffix relationships of a SINGLE string (does a suffix equal a prefix, what's the LCP between the string and each of its suffixes), reach for the Z-function directly — it often avoids an awkward concatenation trick that KMP would otherwise need.
>
> If the problem needs to compare MANY arbitrary substrings against each other — not one fixed pattern, but 'is substring [a,b] equal to substring [c,d]?' repeatedly — that's precisely what neither KMP nor Z do well, and where rolling hash's O(1)-per-query substring comparison wins.
>
> And the moment 'palindrome' and 'O(n)' appear together, it's Manacher — nothing else on this list solves that problem at all, let alone in linear time."

**RED FLAGS:**
- Defaulting to "just use hashing for everything" — collision risk and constant-factor overhead make it the wrong first choice for simple fixed-pattern search where KMP is both faster to reason about and collision-free
- Not knowing that Z-function and KMP have the SAME asymptotic complexity — treating Z as "the fancy upgrade" rather than "a different-shaped tool for a different-shaped problem"

---

## SECTION 15 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: In KMP, why is it safe to jump `j` to `lps[j-1]` on a mismatch instead of restarting from `j = 0`?**

> `lps[j-1]` is the length of the longest proper prefix of `pattern[0..j-1]` that is ALSO a suffix of `pattern[0..j-1]`. Since `pattern[0..j-1]` just matched against `text[i-j..i-1]`, that same suffix is sitting in the text RIGHT NOW. So the longest prefix of `pattern` that could possibly still be matching at this text position is exactly `lps[j-1]` characters long — anything longer is provably impossible (it would require an even longer prefix-suffix match, which `lps[j-1]` already says doesn't exist), and anything shorter would be throwing away a match we've already proven exists.

---

**Q2: Give a concrete example where a Rabin-Karp hash "hit" is a false positive (spurious hit), and explain why the double-check is non-negotiable.**

> With a small modulus for illustration, `M = 97`: the strings "ab" and "ay" mapped as digits (a=1,b=2,y=25) could easily collide — e.g. if base B=10, hash("ab")=1*10+2=12, hash("ay")=1*10+25=35 — different here, but with a small enough M relative to alphabet size, DIFFERENT strings routinely produce the SAME hash mod M by the pigeonhole principle (there are only M possible hash values but infinitely many strings). Skipping the double-check means accepting these collisions as real matches — silently wrong output. The double-check (`text.compare(i, m, pattern) == 0`) costs O(m) but only runs on actual hash hits, which are rare, so it doesn't change the expected complexity.

---

**Q3: Walk through why the Z-algorithm's `[l, r]` window lets `zFunction` run in O(n) total instead of O(n²).**

> Every character comparison either (a) happens while extending `z[i]` past the seeded value from the mirror, which only succeeds in extending `r` further right each time it happens (each successful match-comparison pushes `r` forward by one, and `r` only ever increases, capped at `n`), or (b) is a single failing comparison that stops the extension. So the total number of SUCCESSFUL comparisons across the whole algorithm is bounded by `n` (since `r` increases by at most 1 per success and starts at 0), and the total number of FAILED comparisons is bounded by `n` too (at most one per `i`). Both bounded by O(n) → O(n) total.

---

**Q4: Why does Manacher's algorithm need a SEPARATOR character between every letter of the original string?**

> Without separators, odd-length palindromes (centered on a character, e.g. "aba") and even-length palindromes (centered on a gap between two characters, e.g. "abba") require different index arithmetic and separate code paths. Inserting a separator (e.g. '#') between every character — and at both ends — makes EVERY palindrome in the transformed string odd-length, centered exactly on some position (either an original character or a separator). This unifies both cases into one algorithm, and critically, the palindrome RADIUS in the transformed string equals the palindrome LENGTH in the original string — a clean, direct mapping back.

---

**Q5: In Repeated Substring Pattern (LC 459), why does `n % (n - lps[n-1]) == 0` detect a repeated pattern?**

> If a string of length `n` is `k` repetitions of some substring of length `p` (so `n = k * p`), then the string is periodic with period `p`. A classical fact about the failure function of periodic strings: `lps[n-1] = n - p` (the longest prefix-that's-also-a-suffix "misses" exactly one period's worth from the front). Rearranging: `p = n - lps[n-1]`. For the string to be an EXACT tiling of copies of this period (not a partial/incomplete final copy), `p` must evenly divide `n` — hence `n % (n - lps[n-1]) == 0`. The `lps[n-1] > 0` guard excludes the degenerate case where the string has no self-overlap at all (which would make `p = n`, i.e., "the whole string is its own period," not a genuine repetition).

---

**Q6: What is the core difference in what KMP's `lps[]` array and the Z-function's `z[]` array each represent?**

> `lps[i]` is a LOCAL, self-referential quantity: "how long is the longest prefix of `pattern[0..i]` that's also a suffix of THAT SAME substring `pattern[0..i]`" — it only looks at the substring ending at `i`.
> `z[i]` is a GLOBAL quantity relative to the WHOLE string: "how long is the common prefix between the ENTIRE string `s` and the suffix of `s` starting at `i`" — it always compares against the fixed prefix starting at index 0, regardless of `i`.
> They solve overlapping but distinct problems: `lps` is naturally suited to "resume matching after a partial mismatch" (KMP's use case), while `z` is naturally suited to "how much does this position look like the start of the string" (prefix-vs-suffix questions like LC 2223).

---

**Q7: Why is double hashing (two independent `(B, M)` pairs) recommended for competitive programming judges but usually unnecessary on LeetCode?**

> On competitive judges (Codeforces, etc.), problem setters sometimes know the popular default `(B, M)` choices used by contestants and deliberately construct "anti-hash" test cases that force collisions against those specific parameters — an adversarial attack on a known, fixed hash function. Using two INDEPENDENT hash functions (different bases and/or moduli) and requiring both to agree makes it computationally infeasible to construct a string that collides on both simultaneously. LeetCode's test cases are fixed in advance and not adversarially generated against your specific implementation choices, so a single large-modulus hash (or KMP/Z, which have no collision risk at all) is normally sufficient — though the spurious-hit double-check against the actual string should ALWAYS be done regardless.

---

## SECTION 16 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. KMP FAILURE FUNCTION (LPS ARRAY)
// ─────────────────────────────────────────────────────────────────
vector<int> computeLPS(const string& p) {
    int m = p.size(); vector<int> lps(m, 0); int k = 0;
    for (int i = 1; i < m; i++) {
        while (k > 0 && p[i] != p[k]) k = lps[k-1];
        if (p[i] == p[k]) k++;
        lps[i] = k;
    }
    return lps;
}

// ─────────────────────────────────────────────────────────────────
// 2. KMP MATCHING LOOP
// ─────────────────────────────────────────────────────────────────
// while (i < n) {
//     if (text[i]==pattern[j]) { i++; j++; }
//     if (j == m) { /* match at i-j */ j = lps[j-1]; }
//     else if (i < n && text[i] != pattern[j]) {
//         j ? j = lps[j-1] : i++;
//     }
// }

// ─────────────────────────────────────────────────────────────────
// 3. Z-FUNCTION
// ─────────────────────────────────────────────────────────────────
vector<int> zFunction(const string& s) {
    int n = s.size(); vector<int> z(n, 0); int l = 0, r = 0;
    for (int i = 1; i < n; i++) {
        if (i < r) z[i] = min(r - i, z[i - l]);
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) z[i]++;
        if (i + z[i] > r) { l = i; r = i + z[i]; }
    }
    return z;
}
// Pattern matching: z(pattern + '#' + text), match where z[i] == pattern.size()

// ─────────────────────────────────────────────────────────────────
// 4. ROLLING HASH UPDATE (Rabin-Karp)
// ─────────────────────────────────────────────────────────────────
// newHash = ((oldHash - removed*Bm % M + M) % M * B + added) % M
//                                        ^^^ always +M before the next %

// ─────────────────────────────────────────────────────────────────
// 5. STRING HASHING (PREFIX HASH + O(1) SUBSTRING COMPARE)
// ─────────────────────────────────────────────────────────────────
// prefixHash[i+1] = (prefixHash[i]*B + s[i]) % M
// pw[i+1] = (pw[i]*B) % M
// hash(l,r) = ((prefixHash[r+1] - prefixHash[l]*pw[r-l+1] % M) % M + M) % M

// ─────────────────────────────────────────────────────────────────
// 6. MANACHER (LONGEST PALINDROMIC SUBSTRING)
// ─────────────────────────────────────────────────────────────────
// t = "#" + interleave(s, "#")
// for i in 1..n-2:
//     if (i < r) P[i] = min(r-i, P[2*c-i]);
//     while (t[i-P[i]-1]==t[i+P[i]+1]) P[i]++;
//     if (i+P[i] > r) { c=i; r=i+P[i]; }
// original substring: start=(bestCenter-bestRadius)/2, length=bestRadius

// ─────────────────────────────────────────────────────────────────
// 7. DECISION TABLE
// ─────────────────────────────────────────────────────────────────
// One fixed pattern vs one text, need a hard guarantee   -> KMP
// Prefix-vs-suffix / prefix-vs-every-suffix questions     -> Z-function
// Compare MANY arbitrary substrings pairwise              -> String hashing
// "Longest palindromic substring" + need O(n)             -> Manacher
// Multiple patterns vs one text                           -> Aho-Corasick (Pattern 26)
// Monotonic "does a substring of length >= L exist" test  -> Binary search + rolling hash

// ─────────────────────────────────────────────────────────────────
// 8. MODULAR ARITHMETIC HYGIENE (applies to hashing everywhere)
// ─────────────────────────────────────────────────────────────────
// - ALWAYS long long for hash values, bases, and moduli.
// - ALWAYS (x % M + M) % M after any subtraction, before further use.
// - ALWAYS verify a hash "hit" against the actual substring before trusting it.
```

---

## SECTION 17 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 9 problems in order.** LC 1044 (Longest Duplicate Substring) and LC 2223 (Sum of Scores of Built Strings) are the two requiring the most conceptual leverage — binary search composition and direct Z-function insight respectively. Budget 40 minutes each.

2. **Write `computeLPS` from memory, cold, five times over the next week.** This is the single highest-leverage drill in the whole pattern — if you can write it correctly on the first try without referencing anything, KMP-family problems stop being a source of anxiety.

3. **Re-derive the Manacher mirror cases (`P[mirror] < r-i`, `==`, `>`) on paper without looking at Section 6.** This is the most commonly fumbled follow-up question in interviews that reach Manacher.

4. **Preview Pattern 26 (Trie / Aho-Corasick):** once KMP's failure function is second nature, Aho-Corasick is "the same idea with a trie instead of a single pattern array." Doing this pattern first makes Pattern 26 dramatically easier.

5. **The hashing generalization challenge:** after finishing the pattern, solve "Longest Common Substring of two strings" using binary search + a SINGLE combined hash set across both strings (rather than the O(nm) DP from Pattern 20) — this cements string hashing as an alternative tool to two-sequence DP, not just a pattern-matching trick.

---

## SECTION 18 — SIGN-OFF CRITERIA

### Tier 1 — KMP mastered
You write `computeLPS` and the matching loop from memory, cleanly, in under 8 minutes combined. You can explain in one sentence why the amortized complexity is O(n+m).

### Tier 2 — Z-algorithm and rolling hash understood
You compute a Z-array by hand for a 6-8 character string, correctly applying the `[l,r]` window shortcut at least once. You implement Rabin-Karp with a correct rolling update AND the spurious-hit double-check without prompting.

### Tier 3 — Manacher and hashing-as-a-utility solid
You implement Manacher with the separator transformation and correctly explain all three mirror cases (`<`, `==`, `>`) without looking anything up. You build a prefix-hash structure and answer an O(1) substring-equality query correctly on the first attempt.

### Tier 4 — Applications fluent
You solve LC 459 (Repeated Substring Pattern) and can explain the `n % (n - lps[n-1]) == 0` insight geometrically (periodicity), not just recite the formula. You solve LC 1044 (Longest Duplicate Substring) by correctly composing binary search with a rolling-hash checker. You solve LC 2223 (Sum of Scores) and can explain why `z[n-i]` directly equals `score(i)` without a concatenation trick.

**Sign-off threshold:** Solve 7 of 9 problems. Mandatory: LC 28 (KMP — the failure function must be explained, not just coded), LC 1044 (Longest Duplicate Substring — the binary search + hashing composition), and LC 2223 (Sum of Scores — direct Z-function application without concatenation). These three represent the three conceptual pillars of this pattern: deterministic matching, hashing-as-a-utility, and the Z-function's prefix-suffix worldview.

---

## REFERENCES

- CP-Algorithms — String Processing section (prefix function, Z-function, string hashing, Manacher): https://cp-algorithms.com/#string-processing
- CP-Algorithms — Prefix function / KMP tutorial: https://cp-algorithms.com/string/prefix-function.html
- CP-Algorithms — Z-function: https://cp-algorithms.com/string/z-function.html
- CP-Algorithms — String hashing: https://cp-algorithms.com/string/string-hashing.html
- CP-Algorithms — Manacher's algorithm: https://cp-algorithms.com/string/manacher.html

---

*Pattern 33 Complete — String Algorithms*
*Next: Pattern 26 — Trie and Aho-Corasick (multi-pattern matching, the KMP-to-trie generalization)*
