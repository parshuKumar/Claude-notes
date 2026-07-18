# PATTERN 25 — DIGIT DP
## Counting Numbers by Property Over a Range — pos / tight / leading — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**The hidden difficulty of Digit DP:**

The setup line is always the same: *"Count how many integers in `[L, R]` satisfy property X."* Every single time, `R` can be `10^15` or a 100-digit string, so brute force iteration is impossible. The property X — digit sum, no repeated digits, no two consecutive equal bits, adjacent digits differ by 1 — is checkable on ONE number in O(digits) time, but you cannot check `10^15` numbers one at a time.

Digit DP is the technique that builds numbers **digit by digit** (most significant to least significant) instead of enumerating them, using recursion + memoization over a tiny state space: which digit position you're at, and a compressed summary of everything that matters about the prefix built so far.

**The three ideas that make or break every digit DP problem:**

1. **`f(R) - f(L-1)` reduction.** Almost no digit DP problem is solved directly on `[L, R]`. Instead, you write `f(N)` = "count of valid numbers in `[0, N]`", and the range answer is `f(R) - f(L-1)`. This turns a two-boundary problem into a single-boundary problem, which is the only version the recursive template naturally handles. When `L` is a huge string, `L - 1` requires manual big-number subtraction (Section 13).

2. **The `tight` flag.** When building a number digit by digit, at each position you're choosing a digit `0..9`. But you cannot always choose freely — if every digit chosen SO FAR has matched `N`'s digits exactly, then the current digit is capped by `N`'s digit at this position (choosing higher would make your number exceed `N`). The moment you choose a digit STRICTLY LESS than `N`'s digit, every remaining position is fully free (0-9), because your number is now guaranteed smaller than `N` regardless of what follows. This "am I still capped by `N`" bit is `tight`.

3. **The `leading` flag (leading zeros).** Numbers don't have fixed digit-lengths — `7` and `007` are the same number. To build every number using a fixed-length digit array (padded with leading zeros), you need to track whether you are STILL inside the leading-zero prefix. This matters because leading zeros are not "real" digits for many properties (a leading `0` should not count toward "digits used," "digit sum," or "last digit seen"). Get this wrong and you silently miscount every number shorter than `N`.

**The one rule that governs correctness:** memoization is valid **only** when `!tight && !leading`. Both flags encode "this call is a special one-off path tied to `N`'s literal digits or to the number's actual length" — reusing a memoized value across different tight/leading contexts corrupts the answer. This single rule is the #1 source of wrong-answer bugs in digit DP, and Section 16 dedicates its first mistake entirely to it.

**The key skill:** before touching code, always answer:
1. What is the base property to check — and does the ANSWER need a count of numbers, or a count of digit occurrences / a sum over numbers (233 is different from 357)?
2. What extra state (beyond `pos`, `tight`, `leading`) does the transition need to remember — a bitmask of used digits? A running digit sum? The last digit placed?
3. Is `N` small enough to fit in `int`/`long long` (use `N-1` directly), or is it given as a string (need manual big-number subtraction)?

Get these three answers right, and the recursive skeleton is identical across every problem in this document.

---

## SECTION 2 — THE UNIVERSAL DIGIT DP TEMPLATE, PARAMETER BY PARAMETER

Every digit DP function in this document has this signature shape:

```cpp
returnType dp(int pos, /* problem-specific state */, bool tight, bool leading)
```

### `pos` — the current digit index (0 = most significant digit)

We convert `N` into a digit array/string `s` of length `len` (via `to_string(N)` for `int`/`long long`, or use the string directly for big-number inputs). `pos` walks from `0` to `len`. When `pos == len`, we've placed every digit — this is the **base case**, where we check whatever the accumulated state says about validity (or, for aggregate problems like LC 233, return an aggregate value).

### `tight` — "am I still bounded by N's digit at this position?"

- `tight == true` means every digit chosen at positions `0..pos-1` was EXACTLY equal to `s[0..pos-1]`. The digit at `pos` is capped: `limit = s[pos] - '0'`.
- `tight == false` means some earlier digit was chosen strictly less than `s`'s digit, so the number built so far is already guaranteed `< N`'s prefix. The digit at `pos` is free: `limit = 9`.
- Transition: `newTight = tight && (d == limit)`. Once `tight` drops to `false`, it can never become `true` again for the rest of this path.

### `leading` — "are all digits chosen so far still zero (i.e., have I started the real number yet)?"

- `leading == true` means every digit chosen at positions `0..pos-1` was `0`. We haven't "started" the number yet — we're still in the padding.
- The moment we choose a nonzero digit `d` while `leading == true`, that digit becomes the number's actual first digit, and `leading` flips to `false` from that point on.
- While `leading` stays `true` at a `0` choice, problem-specific state (digit-used mask, digit sum, last digit) must **NOT** be updated by that `0` — it's padding, not a real digit.
- Transition: `newLeading = leading && (d == 0)`.

### Problem-specific state (varies per problem)

This is the piece that changes from problem to problem — it's the compressed summary of "everything about the prefix built so far that the future needs to know":

| Problem | Extra state | Why |
|---|---|---|
| Unique Digits (357) | `mask` (10-bit, digits used) | need to forbid repeats |
| At Most N Given Digit Set (902) | none (closed form) | digit set is fixed and sorted, no memo needed |
| Count of Integers (2719) | `sum` (running digit sum, capped) | need sum bounded in `[minSum, maxSum]` |
| No Consecutive Ones (600) | `prev` (last **bit**, 0/1) | need to forbid `11` in binary |
| Digit One Count (233) | none extra, but returns a **pair** `(count, ones)` | aggregate over completions, not boolean |
| Special Integers (2376) | `mask` (10-bit) | same as 357, bounded by `N` |
| Repeated Digits (1012) | reuses 2376's `mask` DP | complement trick |
| Stepping Numbers (2801) | `last` (last digit 0-9) | adjacent digits must differ by exactly 1 |
| Rotated Digits (788) | `hasDiff` (bool) | needs ≥1 digit from `{2,5,6,9}`, none from `{3,4,7}` |

### The memoization rule (the single most important sentence in this document)

> **Memoize `(pos, extra_state)` only when `!tight && !leading`.**

Why: when `tight == true`, the set of allowed digits at this position depends on `N`'s literal digit at `pos` — a call with `tight=true` computes something that is only valid for THIS specific prefix-matches-`N` scenario, not reusable for other prefixes. There is at most ONE tight path through the whole recursion tree at any given time, so memoizing it wastes memory and, worse, if you memoize it keyed only by `(pos, extra_state)` you will return a stale value the next time the same `(pos, extra_state)` pair occurs under `tight=false` (or vice versa) — those are computing DIFFERENT quantities (bounded-suffix count vs. free-suffix count) that happen to collide in the memo table.

Similarly, when `leading == true`, the extra state (say `mask == 0`) means "no real digits placed yet, still padding" — but `mask == 0` under `leading == false` would mean "somehow zero digits used while past the leading prefix," which is a different (often impossible, but structurally distinct) situation. Mixing the two under one memo cell corrupts results the moment a problem's `0`-handling differs between the two flags (357, 2376, 1012, 2801, 788 all differ this way).

### Full skeleton (reused, adapted per problem below)

```cpp
#include <bits/stdc++.h>
using namespace std;

class DigitDPSkeleton {
public:
    string num;                  // digit string of N
    int len;
    // memo dimensions: [pos][extra-state...] — sized per problem
    // vector<vector<long long>> memo; vector<vector<bool>> vis;

    long long dp(int pos, /* int state, */ bool tight, bool leading) {
        // 1) BASE CASE — all digits placed
        if (pos == len) {
            // return whatever "valid completion" means for this problem
            return 1; // placeholder
        }

        // 2) MEMO LOOKUP — only valid when free of both special flags
        // if (!tight && !leading && vis[pos][state]) return memo[pos][state];

        // 3) DIGIT LOOP
        int limit = tight ? (num[pos] - '0') : 9;
        long long res = 0;
        for (int d = 0; d <= limit; d++) {
            bool newTight   = tight && (d == limit);
            bool newLeading = leading && (d == 0);
            // int newState = transition(state, d, leading); // problem-specific
            res += dp(pos + 1, /* newState, */ newTight, newLeading);
        }

        // 4) MEMO STORE — only when free of both special flags
        // if (!tight && !leading) { vis[pos][state] = true; memo[pos][state] = res; }

        return res;
    }

    long long f(long long N) {
        if (N < 0) return 0;          // guard: f(-1) = 0, needed for f(L-1) when L=0
        num = to_string(N);
        len = num.size();
        // reset memo here (memo is tied to THIS specific num string!)
        return dp(0, /* initial state, */ true, true);
    }
};
```

**Two guards you will forget at least once:** (a) `f(N)` for `N < 0` must return `0` directly (this happens when `L = 0` and you compute `f(L-1) = f(-1)`) — never call `to_string(-1)` into the digit array. (b) The memo table is tied to the specific `num` string being processed; if you call `f()` twice (once for `R`, once for `L-1`), you **must** reset/reallocate the memo between calls, because the two calls use different `N` and the SAME `(pos, state)` cell means different things for different `N`.

---

## SECTION 3 — TEMPLATE 1: THE MASTER RECURSIVE TEMPLATE (BARE MECHANICS)

Before adding any property, let's verify the skeleton itself is correct by counting **all** integers in `[0, N]` — no property at all, `dp` just returns `1` at every leaf.

```cpp
// Trivial digit DP: count all integers in [0, N]. Answer must equal N + 1.
// This isolates tight/leading mechanics with zero problem-specific state.

class TrivialCount {
public:
    string num;
    int len;

    long long dp(int pos, bool tight, bool leading) {
        if (pos == len) return 1;              // every completed path is "valid"
        // No memo needed here (no repeated (pos) subproblems worth caching for
        // this trivial example — but a real implementation WOULD memo on !tight&&!leading)
        int limit = tight ? (num[pos] - '0') : 9;
        long long res = 0;
        for (int d = 0; d <= limit; d++) {
            bool newTight   = tight && (d == limit);
            bool newLeading = leading && (d == 0);   // leading unused here but tracked for pattern consistency
            res += dp(pos + 1, newTight, newLeading);
        }
        return res;
    }

    long long f(long long N) {
        if (N < 0) return 0;
        num = to_string(N);
        len = num.size();
        return dp(0, true, true);
    }
};
```

**FULL TRACE for N = 25 (expect 26 — the integers 0..25):**

```
num = "25", len = 2

dp(0, tight=true, leading=true):  limit = num[0]-'0' = 2
  d=0: newTight=(0==2)=false, newLeading=true  -> dp(1, false, true)
  d=1: newTight=(1==2)=false, newLeading=false -> dp(1, false, false)
  d=2: newTight=(2==2)=true,  newLeading=false -> dp(1, true,  false)

dp(1, false, true):  limit = 9 (tight false)
  d=0..9 (10 values), each -> dp(2, ..., ...) which returns 1 (pos==len)
  total = 10

dp(1, false, false):  limit = 9
  d=0..9 (10 values), each -> dp(2,...) = 1
  total = 10

dp(1, true, false):  limit = num[1]-'0' = 5
  d=0..5 (6 values), each -> dp(2,...) = 1
  total = 6

dp(0,...) = 10 (from d=0) + 10 (from d=1) + 6 (from d=2) = 26

Answer: 26 = integers 0..25 inclusive ✓ (25 - 0 + 1 = 26)
```

This trace demonstrates the core mechanic cleanly: the `d=0` branch (still `leading`) and `d=1` branch (already free) both collapse to "count everything for the remaining `len-pos-1` positions" = `10^(remaining)`, while the `d=2` branch (still `tight`) stays constrained. Every problem below is this exact skeleton with a property check bolted onto the base case and a state transition bolted onto the digit loop.

---

## SECTION 4 — TEMPLATE 2: COUNT NUMBERS WITH UNIQUE DIGITS (LC 357)

> Given `n`, count integers `x` with `0 <= x < 10^n` that have all unique digits. (LeetCode phrases the bound as `10^n`, so we run digit DP against `N = 10^n - 1`, a string of `n` nines.)

**State:** `mask` — a 10-bit bitmask of digits already used by the REAL (non-leading-zero) part of the number.

```cpp
// LC 357 — Count Numbers with Unique Digits
// dp(pos, mask, tight, leading): count of completions from pos onward
// where `mask` tracks which digits 0-9 have already been used by real digits.
//
// KEY: while leading==true, choosing d==0 must NOT set a bit in mask —
// a leading zero is not a "digit used," it's padding.

class Solution357 {
public:
    string num;
    int len;
    vector<vector<int>> memo;
    vector<vector<bool>> vis;

    int dp(int pos, int mask, bool tight, bool leading) {
        if (pos == len) return 1;   // every completed path (including "0" itself) is valid
        if (!tight && !leading && vis[pos][mask]) return memo[pos][mask];

        int limit = tight ? (num[pos] - '0') : 9;
        int res = 0;
        for (int d = 0; d <= limit; d++) {
            bool newTight = tight && (d == limit);
            if (leading && d == 0) {
                res += dp(pos + 1, mask, newTight, true);      // mask unchanged: 0 is padding
            } else {
                if (mask & (1 << d)) continue;                 // digit already used — skip
                res += dp(pos + 1, mask | (1 << d), newTight, false);
            }
        }
        if (!tight && !leading) { vis[pos][mask] = true; memo[pos][mask] = res; }
        return res;
    }

    int countNumbersWithUniqueDigits(int n) {
        string N(n, '9');                      // N = 10^n - 1
        num = N;
        len = num.size();
        memo.assign(len, vector<int>(1 << 10, 0));
        vis.assign(len, vector<bool>(1 << 10, false));
        return dp(0, 0, true, true);
    }
};
```

**TRACE for n = 1 (N = "9", expect 10 — all of 0..9 trivially have unique digits):**

```
num = "9", len = 1

dp(0, mask=0, tight=true, leading=true): limit = 9
  d=0: leading&&d==0 -> dp(1, 0, false, true) -> pos==len -> returns 1
  d=1..8: leading&&d!=0 -> dp(1, 1<<d, false, false) -> returns 1 each  (8 contributions)
  d=9: leading&&d!=0, newTight=(9==9)=true -> dp(1, 1<<9, true, false) -> returns 1

Total = 1 (d=0) + 8 (d=1..8) + 1 (d=9) = 10 ✓
```

**TRACE for n = 2 (N = "99", expect 91 — known closed form 10 + 9·9):**

```
num = "99", len = 2

dp(0, 0, true, true): limit = 9
  d=0 (leading stays true): dp(1, 0, false, true)
  d=1..9 (leading becomes false, mask={d}): dp(1, 1<<d, tight=(d==9), false)

dp(1, 0, false, true): limit=9 (tight false)
  d=0: leading stays -> dp(2,...) = 1
  d=1..9: mask={d} -> dp(2,...) = 1 each (9 values)
  total = 1 + 9 = 10   (this is exactly the n=1 answer — single-digit numbers 0..9)

dp(1, 1<<d0, false, false) for d0 in 1..8 (tight false since d0 != 9): limit=9
  digit already used = d0, so 9 remaining digits allowed (0-9 minus d0) = 9 valid choices
  each -> dp(2,...) = 1, total = 9 per such d0

dp(1, 1<<9, true, false) for d0=9 (tight true): limit = num[1]-'0' = 9
  d=0..9, exclude d==9 (already used) -> 9 valid choices (0-8), each -> 1
  total = 9

Total dp(0,...) = 10 (d=0 branch)
                 + 9*9 (d=1..9 branches, each contributing 9)  -- wait, careful: d=1..8 use "tight=false" (9 each),
                   and d=9 uses "tight=true" (also 9, computed above) -> so 9 branches each contributing 9 = 81
                 = 10 + 81 = 91 ✓
```

This matches the classic combinatorial answer for `n=2`: `10 + 9×9 = 91`.

---

## SECTION 5 — TEMPLATE 3: NUMBERS AT MOST N GIVEN DIGIT SET (LC 902)

> Given a sorted array of unique digit-strings `digits` (each a single character, no `'0'` per LC constraints) and an integer `n`, count positive integers `<= n` that can be formed using digits from the set (repetition allowed, any length up to `len(n)`).

This problem is the instructive **degenerate case**: because `digits` has no `0`, there's no leading-zero subtlety, and because the free branch (`tight=false`) always has a clean closed form (`|digits|^remaining`), no memo table is actually needed — the "digit DP" collapses into a single O(len · |digits|) scan. It's included here specifically to show what a digit DP problem looks like when the state space degenerates to nothing.

```cpp
// LC 902 — Numbers At Most N Given Digit Set
// Because `digits` excludes '0', every combination of length < len(N) is
// automatically a valid, leading-zero-free number: exactly |digits|^length of them.
// For length == len(N), walk position by position under the tight constraint.

class Solution902 {
public:
    int atMostNGivenDigitSet(vector<string>& digits, int n) {
        string N = to_string(n);
        int len = N.size(), D = digits.size();

        vector<long long> pw(len + 1);
        pw[0] = 1;
        for (int i = 1; i <= len; i++) pw[i] = pw[i-1] * D;

        long long count = 0;

        // Numbers with fewer digits than N: all free, D^length each.
        for (int length = 1; length < len; length++) count += pw[length];

        // Numbers with the SAME number of digits as N: tight walk.
        bool matchedFull = true;   // stays true only if N itself is exactly formable
        for (int i = 0; i < len; i++) {
            bool matched = false;
            for (auto& dg : digits) {                 // digits is sorted ascending
                if (dg[0] < N[i]) {
                    count += pw[len - i - 1];          // strictly smaller here -> rest fully free
                } else if (dg[0] == N[i]) {
                    matched = true;
                    break;                             // sorted: no need to look further for equal
                } else {
                    break;                              // sorted: everything after is even bigger
                }
            }
            if (!matched) { matchedFull = false; break; }   // can't stay tight -> N-length branch dies here
        }
        if (matchedFull) count++;                      // N itself is achievable digit-by-digit

        return (int)count;
    }
};
```

**TRACE for `digits = ["1","3","5","7"]`, `n = 100` (expect 20):**

```
N = "100", len = 3, D = 4
pw = [1, 4, 16, 64]

Shorter-length numbers: length=1 -> pw[1]=4, length=2 -> pw[2]=16.  running count = 20.

Same-length (len=3) tight walk:
  i=0, N[0]='1':
    dig="1": '1' < '1'? no.  '1' == '1'? yes -> matched=true, break.
    (no count added here — the ONLY digit equal to '1' matched, nothing smaller exists)
  i=1, N[1]='0':
    dig="1": '1' < '0'? no. '1' == '0'? no. else -> break (sorted, all remaining are >= '1' > '0')
    matched = false -> matchedFull = false, stop the tight walk entirely.

count (from tight walk) += 0.
matchedFull = false -> no +1.

Total = 20 ✓ (matches the official LC 902 example)
```

Notice how the tight walk dies at `i=1` precisely because no available digit (`1,3,5,7`) is `≤ '0'` — meaning zero 3-digit numbers starting with `1` can be formed at all from this digit set that stay `≤ 100`. All 20 valid numbers come from the 1- and 2-digit combinations.

---

## SECTION 6 — TEMPLATE 4: COUNT OF INTEGERS (LC 2719)

> Given `num1`, `num2` as (possibly huge) decimal strings and `min_sum`, `max_sum`, count integers `x` with `num1 <= x <= num2` and `min_sum <= digitSum(x) <= max_sum`, mod `1e9+7`.

**State:** `sum` — running digit sum, capped/pruned once it exceeds `max_sum` (no need to track beyond that). **No leading flag complication here** — a leading `0` contributes `0` to the digit sum regardless, so treating leading zeros as ordinary zero-digits is harmless. This is a deliberate teaching contrast with Section 4/9/11/12 where leading zeros DO need special handling.

This is also the first problem requiring **big-number `N - 1`** (Section 13), since `num1` can be up to 22 digits — far beyond `long long`.

```cpp
// LC 2719 — Count of Integers
// answer = f(num2) - f(subtractOne(num1)),  f(N) = count of x in [0,N] with
// min_sum <= digitSum(x) <= max_sum.

class Solution2719 {
public:
    const int MOD = 1e9 + 7;
    string num;
    int minSum, maxSum;
    vector<vector<int>> memo;
    vector<vector<bool>> vis;

    long long dp(int pos, int sum, bool tight) {
        if (sum > maxSum) return 0;               // prune: no way to recover once over budget
        if (pos == (int)num.size()) {
            return (sum >= minSum) ? 1 : 0;        // sum <= maxSum already guaranteed by the prune above
        }
        if (!tight && vis[pos][sum]) return memo[pos][sum];

        int limit = tight ? (num[pos] - '0') : 9;
        long long res = 0;
        for (int d = 0; d <= limit; d++) {
            res = (res + dp(pos + 1, sum + d, tight && d == limit)) % MOD;
        }
        if (!tight) { vis[pos][sum] = true; memo[pos][sum] = (int)res; }
        return res;
    }

    long long f(string N) {
        num = N;
        // sum can range 0..(9*len) <= 9*22 = 198, size memo generously to 401
        memo.assign(num.size(), vector<int>(401, 0));
        vis.assign(num.size(), vector<bool>(401, false));
        return dp(0, 0, true);
    }

    string subtractOne(string s) {
        int i = (int)s.size() - 1;
        while (i >= 0 && s[i] == '0') { s[i] = '9'; i--; }
        s[i]--;                                    // guaranteed s[i] was >= '1' here (num1 >= 1)
        int start = 0;
        while (start + 1 < (int)s.size() && s[start] == '0') start++;  // strip extra leading zeros, keep >=1 digit
        return s.substr(start);
    }

    int count(string num1, string num2, int min_sum, int max_sum) {
        minSum = min_sum; maxSum = max_sum;
        long long hi = f(num2);
        long long lo = f(subtractOne(num1));
        return (int)(((hi - lo) % MOD + MOD) % MOD);
    }
};
```

**TRACE — cross-check with brute enumeration, `num1="1"`, `num2="12"`, `min_sum=1`, `max_sum=8` (expect 11):**

```
subtractOne("1"):
  i=0, s[0]='1' != '0' -> s[0]-- -> "0"
  strip leading zeros: start=0 (size==1, loop doesn't run) -> return "0"

f("0"): dp(0, 0, true) on num="0", len=1
  limit = 0 (tight, num[0]='0')
  d=0: dp(1, 0, tight&&(0==0)=true) -> pos==len -> sum=0, 0>=minSum(1)? no -> returns 0
  total f("0") = 0

f("12"): manually verify against brute force (digit sums of 0..12):
  0:0  1:1  2:2  3:3  4:4  5:5  6:6  7:7  8:8  9:9  10:1  11:2  12:3
  in [1,8]: {1,2,3,4,5,6,7,8} (8 numbers) + {10,11,12} (3 numbers) = 11 numbers
  f("12") = 11

count = (f("12") - f("0")) mod = 11 - 0 = 11 ✓ matches official LC 2719 example
```

**A short formal recursion trace (isolated, N="23", pretend `minSum=2, maxSum=4`, to see `tight`/`sum` branching mechanics):**

```
num = "23", len = 2

dp(0, sum=0, tight=true): limit = num[0]-'0' = 2
  d=0: dp(1, 0, false)
  d=1: dp(1, 1, false)
  d=2: dp(1, 2, true)   (tight stays true: 2==2)

dp(1, 0, false): limit=9.  d=0..9 -> sum becomes 0..9 at pos==len.
  valid (sum in [2,4]): d=2,3,4 -> 3 completions

dp(1, 1, false): sum becomes 1..10.  valid (in [2,4]): d=1,2,3 -> sums 2,3,4 -> 3 completions

dp(1, 2, true): limit = num[1]-'0' = 3.  d=0..3 -> sum becomes 2,3,4,5.
  valid (in [2,4]): d=0,1,2 (sums 2,3,4) -> 3 completions;  d=3 (sum=5) invalid.

Total dp(0,...) = 3 (d=0) + 3 (d=1) + 3 (d=2) = 9

Brute check, digit sums of 0..23 in [2,4]:
  2,3,4 (sums 2,3,4); 11(2),12(3),13(4); 20(2),21(3),22(4)  = 9 numbers ✓
```

---

## SECTION 7 — TEMPLATE 5: NON-NEGATIVE INTEGERS WITHOUT CONSECUTIVE ONES (LC 600)

> Count integers in `[0, n]` whose **binary representation** contains no two consecutive `1` bits.

**State:** `prev` — the last bit placed (0 or 1). Digits here are binary (`0`/`1`), not decimal. **No leading flag needed**: a leading `0` bit is indistinguishable from a genuine `0` bit for this property — `prev = 0` is exactly the correct "no violation risk" baseline whether it's padding or real.

```cpp
// LC 600 — Non-negative Integers without Consecutive Ones
// Binary digit DP. prev = last bit chosen (starts at 0, the correct baseline
// since a leading zero behaves identically to a real zero bit here).

class Solution600 {
public:
    int findIntegers(int n) {
        string bin;
        for (int i = 31; i >= 0; i--) bin += char('0' + ((n >> i) & 1));
        int len = bin.size();

        vector<vector<int>> memo(len, vector<int>(2, -1));

        function<int(int,int,bool)> dp = [&](int pos, int prev, bool tight) -> int {
            if (pos == len) return 1;
            if (!tight && memo[pos][prev] != -1) return memo[pos][prev];

            int limit = tight ? (bin[pos] - '0') : 1;   // binary: limit is 0 or 1
            int res = 0;
            for (int d = 0; d <= limit; d++) {
                if (prev == 1 && d == 1) continue;       // forbidden: two consecutive 1-bits
                res += dp(pos + 1, d, tight && d == limit);
            }
            if (!tight) memo[pos][prev] = res;
            return res;
        };

        return dp(0, 0, true);
    }
};
```

**TRACE for n = 5 (binary "101", expect 5 — valid: 0,1,2,4,5; excluded: 3="011" has consecutive 1s):**

```
bin = "101", len = 3

dp(0, prev=0, tight=true): limit = bin[0]-'0' = 1
  d=0: dp(1, 0, false)
  d=1: dp(1, 1, true)      (tight: 1==1)

dp(1, prev=0, tight=false): limit=1
  d=0: dp(2, 0, false)
  d=1: dp(2, 1, false)

dp(2, prev=0, tight=false): limit=1
  d=0: dp(3,0,false)=1 (base)
  d=1: dp(3,1,false)=1 (base, prev0&d1 fine)
  total = 2  -> memo[2][0]=2

dp(2, prev=1, tight=false): limit=1
  d=0: dp(3,0,false)=1
  d=1: prev==1 && d==1 -> SKIP
  total = 1  -> memo[2][1]=1

dp(1,0,false) = dp(2,0,false) + dp(2,1,false) = 2 + 1 = 3   -> memo[1][0]=3

dp(1, prev=1, tight=true): limit = bin[1]-'0' = 0
  d=0: prev==1 && d==0 -> fine (not consecutive). newTight = true&&(0==0)=true.
       dp(2, 0, true): limit = bin[2]-'0' = 1
         d=0: dp(3,0,false)=1
         d=1: dp(3,1,true)=1 (base, unaffected by tight)
         total = 2
  total dp(1,1,true) = 2   (only d=0 possible since limit=0)

dp(0,0,true) = dp(1,0,false) [d=0] + dp(1,1,true) [d=1] = 3 + 2 = 5 ✓
```

---

## SECTION 8 — TEMPLATE 6: NUMBER OF DIGIT ONE (LC 233)

> Count the **total number of occurrences of the digit `1`** across all integers in `[1, n]` — NOT the count of numbers containing a `1`. This is the sharpest illustration of the "count-of-numbers vs. count-of-occurrences" distinction (Mistake 4, Section 16).

**Key trick:** since the answer is a SUM over completions (each completion can contribute 0, 1, or more `1`-digits), the memo must return a **pair**: `(numberOfCompletions, totalOnesAcrossCompletions)`. A single scalar cannot carry enough information, because when we place a digit `d == 1` at position `pos`, EVERY one of the `c` completions of the suffix gets `+1` one — the contribution scales with the completion count, not just with whether `d==1`.

```cpp
// LC 233 — Number of Digit One
// dp returns {count of completions, total '1' occurrences across those completions}.

class Solution233 {
public:
    string num;
    int len;
    vector<vector<long long>> memoCnt, memoOnes;
    vector<vector<bool>> vis;

    pair<long long,long long> dp(int pos, bool tight) {
        if (pos == len) return {1, 0};
        if (!tight && vis[pos][0]) return {memoCnt[pos][0], memoOnes[pos][0]};

        int limit = tight ? (num[pos] - '0') : 9;
        long long cnt = 0, ones = 0;
        for (int d = 0; d <= limit; d++) {
            auto [c, o] = dp(pos + 1, tight && d == limit);
            cnt  += c;
            ones += o + (d == 1 ? c : 0);   // every one of `c` completions inherits a '1' from THIS digit
        }
        if (!tight) { vis[pos][0] = true; memoCnt[pos][0] = cnt; memoOnes[pos][0] = ones; }
        return {cnt, ones};
    }

    int countDigitOne(int n) {
        num = to_string(n);
        len = num.size();
        memoCnt.assign(len, vector<long long>(1, 0));
        memoOnes.assign(len, vector<long long>(1, 0));
        vis.assign(len, vector<bool>(1, false));
        return (int)dp(0, true).second;
    }
};
```

**TRACE for n = 13 (expect 6 — occurrences: 1→1, 10→1, 11→2, 12→1, 13→1, total = 6):**

```
num = "13", len = 2

dp(2, tight) = {1, 0} for any tight (base case).

dp(1, tight=true): pos=1, limit = num[1]-'0' = 3
  d=0: dp(2,false)={1,0}; cnt+=1; ones += 0 + (d==1?1:0)=0
  d=1: dp(2,false)={1,0}; cnt+=1; ones += 0 + 1 (d==1, c=1) = 1
  d=2: dp(2,false)={1,0}; cnt+=1; ones += 0
  d=3: dp(2, tight&&3==3=true)={1,0}; cnt+=1; ones += 0
  dp(1,true) = {cnt=4, ones=1}

dp(1, tight=false) [memoized state]: limit=9
  d=0..9, each dp(2,false)={1,0}; only d==1 contributes ones+=1
  dp(1,false) = {cnt=10, ones=1}

dp(0, tight=true): limit = num[0]-'0' = 1
  d=0: newTight=false -> dp(1,false)={10,1}; cnt+=10; ones += 1 + (d==1?10:0) = 1 + 0 = 1
  d=1: newTight=true  -> dp(1,true) ={4,1};  cnt+=4;  ones += 1 + (d==1?4:0)  = 1 + 4 = 5

dp(0,true) = {cnt=14, ones=6}

cnt=14 double-checks as |{0..13}| = 14 numbers ✓
ones=6 matches expected total occurrences of digit '1' in [0,13] (= [1,13] since 0 has none) ✓
```

This pair-return technique — `(count, aggregate)` — generalizes to any digit DP that asks for a SUM rather than a boolean count: sum of all digit sums, sum of all numbers themselves, etc.

---

## SECTION 9 — TEMPLATE 7: COUNT SPECIAL INTEGERS (LC 2376)

> Count integers in `[1, n]` whose digits are **all distinct** (no repeated digit). This is LC 357's engine (Section 4) but now bounded tightly by an arbitrary `n` instead of a full `10^k - 1`, so both `tight` and `leading` genuinely matter together with the `mask`.

```cpp
// LC 2376 — Count Special Integers
// Identical mechanics to 357's mask DP, but the base case must EXCLUDE the
// all-zero path (that represents "0", and this problem counts from 1).

class Solution2376 {
public:
    string num;
    int len;
    vector<vector<int>> memo;
    vector<vector<bool>> vis;

    int dp(int pos, int mask, bool tight, bool leading) {
        if (pos == len) return leading ? 0 : 1;   // leading still true at the end == the number "0" -> excluded
        if (!tight && !leading && vis[pos][mask]) return memo[pos][mask];

        int limit = tight ? (num[pos] - '0') : 9;
        int res = 0;
        for (int d = 0; d <= limit; d++) {
            bool newTight = tight && d == limit;
            if (leading && d == 0) {
                res += dp(pos + 1, mask, newTight, true);          // padding, mask untouched
            } else {
                if (mask & (1 << d)) continue;                     // repeated digit — forbidden
                res += dp(pos + 1, mask | (1 << d), newTight, false);
            }
        }
        if (!tight && !leading) { vis[pos][mask] = true; memo[pos][mask] = res; }
        return res;
    }

    int countSpecialNumbers(int n) {
        num = to_string(n);
        len = num.size();
        memo.assign(len, vector<int>(1 << 10, 0));
        vis.assign(len, vector<bool>(1 << 10, false));
        return dp(0, 0, true, true);
    }
};
```

**TRACE for n = 20 (expect 19 — all of 1..20 except 11, which repeats digit `1`):**

```
num = "20", len = 2

dp(0, mask=0, tight=true, leading=true): limit = num[0]-'0' = 2
  d=0: leading -> dp(1, 0, false, leading=true)
  d=1: not leading branch, mask={1} -> dp(1, 2, false, leading=false)     (tight: 1==2? no)
  d=2: not leading branch, mask={2} -> dp(1, 4, true,  leading=false)     (tight: 2==2? yes)

dp(1, mask=0, tight=false, leading=true): limit=9
  d=0: leading -> dp(2,0,false,true) -> pos==len, leading true -> returns 0
  d=1..9 (9 values): first real digit -> dp(2, 1<<d, false, false) -> pos==len, leading false -> returns 1
  total = 0 + 9 = 9        (represents single-digit numbers 1..9, all trivially unique)

dp(1, mask=2 (bit1), tight=false, leading=false): limit=9
  d=0: bit0 not set -> dp(2, 3, false, false) -> 1
  d=1: bit1 SET -> skip
  d=2..9 (8 values): not set -> 1 each
  total = 1 + 0 + 8 = 9     (numbers 10,12,13,...,19 — i.e. "1X" excluding "11" — 9 numbers)

dp(1, mask=4 (bit2), tight=true, leading=false): limit = num[1]-'0' = 0
  d=0: bit0 not set -> dp(2, 5, true, false) -> pos==len, leading false -> 1
  total = 1                 (represents the number "20" itself, digits 2,0 distinct)

Total dp(0,...) = 9 (d=0) + 9 (d=1) + 1 (d=2) = 19 ✓
```

---

## SECTION 10 — TEMPLATE 8: NUMBERS WITH REPEATED DIGITS (LC 1012)

> Count integers in `[1, n]` that contain **at least one repeated digit**. This is the exact **complement** of Section 9: `answer = n - countSpecialNumbers(n)`.

```cpp
// LC 1012 — Numbers With Repeated Digits
// Complement trick: "has a repeat" = total count - "all unique digits" count.
// Reuses the identical mask-DP engine from LC 2376 (Section 9).

class Solution1012 {
public:
    int helperCountUnique(const string& s) {
        int len = s.size();
        vector<vector<int>> memo(len, vector<int>(1 << 10, 0));
        vector<vector<bool>> vis(len, vector<bool>(1 << 10, false));

        function<int(int,int,bool,bool)> dp = [&](int pos, int mask, bool tight, bool leading) -> int {
            if (pos == len) return leading ? 0 : 1;
            if (!tight && !leading && vis[pos][mask]) return memo[pos][mask];
            int limit = tight ? (s[pos] - '0') : 9;
            int res = 0;
            for (int d = 0; d <= limit; d++) {
                bool newTight = tight && d == limit;
                if (leading && d == 0) {
                    res += dp(pos + 1, mask, newTight, true);
                } else {
                    if (mask & (1 << d)) continue;
                    res += dp(pos + 1, mask | (1 << d), newTight, false);
                }
            }
            if (!tight && !leading) { vis[pos][mask] = true; memo[pos][mask] = res; }
            return res;
        };
        return dp(0, 0, true, true);
    }

    int numDupDigitsAtMostN(int n) {
        return n - helperCountUnique(to_string(n));
    }
};
```

**TRACE for n = 20:** reusing Section 9's result directly, `countSpecialNumbers(20) = 19`, so `numDupDigitsAtMostN(20) = 20 - 19 = 1` (only `11` has a repeated digit in `[1,20]`) — matches the official LC 1012 example (`n=20 -> 1`).

**Why the complement is safe here (and not always):** the complement trick works cleanly whenever "property X" and "not property X" partition `[1,n]` exactly, with no third bucket to worry about (every integer either has all-unique digits or has a repeat — no in-between case). This won't work directly for problems where the base set isn't `[1,n]` itself (e.g., you still need `f(R)-f(L-1)` first, THEN complement within that if the range doesn't start at 1).

---

## SECTION 11 — TEMPLATE 9: COUNT STEPPING NUMBERS IN RANGE (LC 2801)

> A **stepping number** has every pair of adjacent digits differing by exactly `1` (e.g., `123`, `321`, `10`). Given `low`, `high` as (potentially huge, up to 100 digits) decimal strings, count stepping numbers in `[low, high]`, mod `1e9+7`.

**State:** `last` — the most recently placed real digit (0-9), used to enforce `|d - last| == 1` for the next digit. Combines with `leading` (the FIRST real digit needs no adjacency check) and requires the big-number `subtractOne` from Section 13.

```cpp
// LC 2801 — Count Stepping Numbers in Range
// dp(pos, last, tight, leading): last = most recent REAL digit placed.

class Solution2801 {
public:
    const int MOD = 1e9 + 7;
    string num;
    int len;
    vector<vector<long long>> memo;
    vector<vector<bool>> vis;

    long long dp(int pos, int last, bool tight, bool leading) {
        if (pos == len) return leading ? 0 : 1;   // exclude "0" itself (range starts >= 1)
        if (!tight && !leading && vis[pos][last]) return memo[pos][last];

        int limit = tight ? (num[pos] - '0') : 9;
        long long res = 0;
        for (int d = 0; d <= limit; d++) {
            bool newTight = tight && d == limit;
            if (leading && d == 0) {
                res = (res + dp(pos + 1, 0, newTight, true)) % MOD;         // still padding
            } else if (leading) {                                          // d != 0: first real digit
                res = (res + dp(pos + 1, d, newTight, false)) % MOD;       // no adjacency check yet
            } else if (abs(d - last) == 1) {                               // normal case: enforce step
                res = (res + dp(pos + 1, d, newTight, false)) % MOD;
            }
        }
        if (!tight && !leading) { vis[pos][last] = true; memo[pos][last] = res; }
        return res;
    }

    long long f(const string& N) {
        num = N;
        len = num.size();
        memo.assign(len, vector<long long>(10, 0));
        vis.assign(len, vector<bool>(10, false));
        return dp(0, 0, true, true);
    }

    string subtractOne(string s) {
        int i = (int)s.size() - 1;
        while (i >= 0 && s[i] == '0') { s[i] = '9'; i--; }
        s[i]--;
        int start = 0;
        while (start + 1 < (int)s.size() && s[start] == '0') start++;
        return s.substr(start);
    }

    int countSteppingNumbers(string low, string high) {
        long long ans = (f(high) - f(subtractOne(low)) % MOD + MOD) % MOD;
        return (int)ans;
    }
};
```

**TRACE for `low="1"`, `high="11"` (expect 10 — stepping numbers 1..9 (trivial single digits) plus 10; 11 fails since `|1-1|=0`):**

```
f(subtractOne("1")) = f("0") = 0  (single digit '0', leading stays true at pos==len -> excluded)

f("11"): num="11", len=2

dp(0, last=0, tight=true, leading=true): limit = num[0]-'0' = 1
  d=0: leading&d0 -> dp(1, 0, false, true)
  d=1: leading&d!=0 -> dp(1, 1, true, false)     (newTight: 1==1)

dp(1, _, tight=false, leading=true): limit=9
  d=0: leading&d0 -> dp(2,0,false,true) -> pos==len, leading true -> 0
  d=1..9 (9 values): first real digit -> dp(2,d,false,false) -> pos==len, leading false -> 1 each
  total = 0 + 9 = 9      (single-digit numbers 1..9, all trivially stepping)

dp(1, last=1, tight=true, leading=false): limit = num[1]-'0' = 1
  d=0: |0-1|=1 -> OK -> dp(2,0,false,false) -> 1
  d=1: |1-1|=0 -> SKIP
  total = 1              (represents "10": step from 1->0 is valid; "11" excluded)

Total dp(0,...) = 9 (d=0 branch) + 1 (d=1 branch) = 10

Answer = f("11") - f("0") = 10 - 0 = 10 ✓
```

---

## SECTION 12 — TEMPLATE 10: ROTATED DIGITS (LC 788)

> A number is **good** if, after rotating each digit 180° (`0→0, 1→1, 2→5, 5→2, 6→9, 9→6, 8→8`; digits `3,4,7` have no valid rotation), the result is a **different, valid** number. Count good numbers in `[1, n]`.

**State:** `hasDiff` (bool) — whether at least one digit from `{2,5,6,9}` (a digit that maps to something else) has been used. Digits `{3,4,7}` immediately disqualify a number and are simply skipped in the loop (pruned, not counted as a state).

```cpp
// LC 788 — Rotated Digits
// same={0,1,8} (rotate to themselves); diff={2,5,6,9} (rotate to a DIFFERENT
// valid digit); invalid={3,4,7} (no valid rotation at all -> number disqualified).
// "Good" requires: zero invalid digits used, AND at least one diff digit used.

class Solution788 {
public:
    int rotatedDigits(int n) {
        string s = to_string(n);
        int len = s.size();
        vector<vector<int>> memo(len, vector<int>(2, -1));

        auto isDiff    = [](int d) { return d == 2 || d == 5 || d == 6 || d == 9; };
        auto isInvalid = [](int d) { return d == 3 || d == 4 || d == 7; };

        function<int(int,bool,bool,bool)> dp = [&](int pos, bool hasDiff, bool tight, bool leading) -> int {
            if (pos == len) return (!leading && hasDiff) ? 1 : 0;
            if (!tight && !leading && memo[pos][hasDiff] != -1) return memo[pos][hasDiff];

            int limit = tight ? (s[pos] - '0') : 9;
            int res = 0;
            for (int d = 0; d <= limit; d++) {
                bool newTight = tight && d == limit;
                if (leading && d == 0) {
                    res += dp(pos + 1, hasDiff, newTight, true);
                    continue;
                }
                if (isInvalid(d)) continue;                      // 3,4,7 kill this whole branch
                bool nd = hasDiff || isDiff(d);
                res += dp(pos + 1, nd, newTight, false);
            }
            if (!tight && !leading) memo[pos][hasDiff] = res;
            return res;
        };

        return dp(0, false, true, true);
    }
};
```

**TRACE for n = 10 (expect 4 — good numbers among 1..10 are exactly {2,5,6,9}):**

```
s = "10", len = 2

dp(0, hasDiff=false, tight=true, leading=true): limit = s[0]-'0' = 1
  d=0: leading&d0 -> dp(1, false, false, true)
  d=1: not invalid, isDiff(1)=false -> nd=false -> dp(1, false, true, false)   (newTight: 1==1)

dp(1, false, tight=false, leading=true): limit=9
  d=0: leading&d0 -> dp(2,false,false,true) -> pos==len, leading true -> 0
  d=1: nd=false (1 is 'same') -> dp(2,false,false,false) -> leading false, hasDiff false -> 0
  d=2: nd=true  -> dp(2,true,...) -> 1
  d=3: invalid -> skip
  d=4: invalid -> skip
  d=5: nd=true  -> 1
  d=6: nd=true  -> 1
  d=7: invalid -> skip
  d=8: nd=false ('same') -> 0
  d=9: nd=true  -> 1
  total = 0+0+1+0+0+1+1+0+0+1 = 4    (single-digit good numbers: {2,5,6,9})

dp(1, false, tight=true, leading=false): limit = s[1]-'0' = 0
  d=0: not invalid, isDiff(0)=false -> nd=false -> dp(2,false,true,false) -> leading false, hasDiff false -> 0
  total = 0     (number "10": digits {1,0} both 'same'-category -> never good)

Total dp(0,...) = 4 (d=0 branch) + 0 (d=1 branch) = 4 ✓
```

---

## SECTION 13 — BIG-NUMBER "N - 1" AND DIGIT-ARRAY SETUP

Two setup steps recur across nearly every digit DP problem, and both deserve to be nailed down once rather than re-derived each time.

### 13.1 — When `N` fits in `long long`

```cpp
long long N = 123456789LL;
string num = to_string(N);      // digit array as a string, MSB first
int len = num.size();
// f(N-1) is trivial: just call f(N - 1) with N-1 also within long long range,
// EXCEPT the boundary N == 0, where f(-1) must be special-cased to return 0
// directly (never call to_string(-1)).
long long lo = (N == 0) ? 0 : f(N - 1);
```

### 13.2 — When `N` is given as a decimal string (up to ~10^15–10^100)

Ordinary integer subtraction overflows or simply doesn't apply. Manual big-number decrement:

```cpp
// Subtract 1 from a positive decimal string. Assumes s represents a value >= 1
// (true for every problem in this document, since ranges start at L >= 1... but
// LC 2719 / 2801 both guard num1/low >= 1 per their constraints, so s[i]-- is safe).
string subtractOne(string s) {
    int i = (int)s.size() - 1;
    while (i >= 0 && s[i] == '0') { s[i] = '9'; i--; }   // borrow chain: 500 -> 499
    s[i]--;                                              // guaranteed a nonzero digit exists at i
    int start = 0;
    while (start + 1 < (int)s.size() && s[start] == '0') start++;   // strip new leading zero(s)
    return s.substr(start);                              // e.g. "100" -> "099" -> stripped -> "99"
}
```

**Walkthrough on `"100"`:** `i=2`, `s[2]='0'` → set to `'9'`, `i=1`; `s[1]='0'` → set to `'9'`, `i=0`; `s[0]='1'` (not `'0'`) → stop loop, `s[0]--` → `'0'`. Result string so far: `"099"`. Strip loop: `start=0`, `s[0]='0'` and `start+1 < 3` → `start=1`. `s[1]='9' != '0'` → stop. Return `s.substr(1) = "99"`. Correct: `100 - 1 = 99`.

**Walkthrough on `"1"`:** `i=0`, `s[0]='1'` (not `'0'`) → stop, `s[0]--` → `'0'`. String is `"0"`. Strip loop: `start+1 < 1` is false immediately (size is 1) → no stripping. Return `"0"`. Correct: `1 - 1 = 0`, and every `f()` in this document treats the string `"0"` correctly (it's a valid single-digit input, the digit-array setup handles it like any other number).

### 13.3 — Why this matters more than it looks

If you skip the leading-zero strip step, `f("099")` would build a digit array of length 3 instead of length 1. This is **usually still correct** IF your `leading` flag is implemented properly (a `leading=true` prefix of zeros is treated as padding, functionally identical to a shorter number) — but it's a common source of off-by-length bugs if a problem's memo dimensions are sized off the wrong `len`, or if a problem (rarely) treats position count as meaningful even under `leading`. Stripping is cheap insurance; always do it.

---

## SECTION 14 — COMPLEXITY TABLE

| Problem | Extra state | State size | Time per `f()` call | # of `f()` calls |
|---|---|---|---|---|
| Unique Digits (357) | `mask` | ≤ 1024 | O(len · 1024 · 10) | 1 |
| At Most N Digit Set (902) | none (closed form) | O(1) | O(len · \|digits\|) | 1 |
| Count of Integers (2719) | `sum` | ≤ 9·22 ≈ 200 | O(len · 200 · 10) | 2 |
| No Consecutive Ones (600) | `prev` (bit) | 2 | O(len · 2 · 2) | 1 |
| Digit One Count (233) | none (pair return) | O(1) | O(len · 10) | 1 |
| Special Integers (2376) | `mask` | ≤ 1024 | O(len · 1024 · 10) | 1 |
| Repeated Digits (1012) | `mask` (reused) | ≤ 1024 | O(len · 1024 · 10) | 1 |
| Stepping Numbers (2801) | `last` (0-9) | 10 | O(len · 10 · 10) | 2 |
| Rotated Digits (788) | `hasDiff` | 2 | O(len · 2 · 10) | 1 |

`len` is the digit-length of `N` — typically ≤ 19 for `int`/`long long` bounds, up to ~100+ for string-encoded ranges (2719, 2801). The universal shape is **`O(digits × states × 10)`**, where `states` is the product of all problem-specific dimensions (and implicitly ×2×2 for `tight`/`leading`, though those two only ever have ONE live combination at a time along any recursion path, so they don't multiply the memo table size — only `pos` and the extra state do).

---

## SECTION 15 — VARIANTS

Beyond the ten template problems, these variants reuse the exact same skeleton with a different extra state:

- **Divisibility by K in range:** state = `remainder` (running value mod `K`, 0..K-1). Transition: `newRem = (rem * 10 + d) % K`. Classic combination with digit sum constraints ("count numbers divisible by K with digit sum S").
- **No specific substring/pattern (e.g., no "13" as a substring):** state = position in a KMP/Aho-Corasick automaton built from the forbidden pattern(s), transitioning via the automaton's failure links as each digit is placed.
- **Palindromic numbers in range:** usually solved by generating palindromes directly (half the digits determine the rest) rather than pure digit DP, but a digit-DP formulation exists with state = the mirrored-position digit still to be matched.
- **Sum of all valid numbers (not just count):** like LC 233's pair trick, return `(count, sumOfNumbers)`, where placing digit `d` at position `pos` contributes `d * 10^(len-pos-1) * count` to the running sum for every completion.
- **K-th smallest number with a property:** binary search on the answer value, using `f(mid)` (count of valid numbers `≤ mid`) as the monotonic predicate — digit DP as a counting oracle inside binary search.
- **Two simultaneous digit constraints (e.g., stepping AND divisible by 3):** state = `(last, remainder)` — just concatenate the state dimensions; the skeleton doesn't change, only the tuple size grows.

---

## SECTION 16 — COMMON MISTAKES

### Mistake 1: Memoizing while `tight == true`

```cpp
// WRONG — memo keyed only by (pos, mask), stored/read regardless of tight
if (memo[pos][mask] != -1) return memo[pos][mask];
...
memo[pos][mask] = res;

// This mixes "bounded by N's remaining digits" results with "fully free"
// results under the same key. A tight-true computation for THIS specific N
// gets cached and then wrongly reused when a DIFFERENT (freer) context hits
// the same (pos, mask).

// CORRECT — only touch the memo when both special flags are false
if (!tight && !leading && vis[pos][mask]) return memo[pos][mask];
...
if (!tight && !leading) { vis[pos][mask] = true; memo[pos][mask] = res; }
```

### Mistake 2: Marking a leading zero as a "used digit" in the mask

```cpp
// WRONG — always sets the bit, even during the leading-zero prefix
for (int d = 0; d <= limit; d++) {
    if (mask & (1 << d)) continue;
    res += dp(pos + 1, mask | (1 << d), tight && d == limit, leading && d == 0);
}
// Under this code, the number "05" would mark bit 0 as "used" before placing
// the 5 — meaning "50" would then be wrongly rejected later as a "duplicate 0",
// or worse, numbers like "5" get merged into the wrong mask state entirely.

// CORRECT — leading zeros bypass the mask completely
if (leading && d == 0) {
    res += dp(pos + 1, mask, tight && d == limit, true);   // mask UNCHANGED
} else {
    if (mask & (1 << d)) continue;
    res += dp(pos + 1, mask | (1 << d), tight && d == limit, false);
}
```

### Mistake 3: Off-by-one in `f(R) - f(L-1)`

```cpp
// WRONG — using f(R) - f(L), which excludes L itself from the count
int answer = f(R) - f(L);

// WRONG — forgetting the L == 0 (or L == "0") special case, calling
// to_string(-1) or subtractOne("0") and corrupting the digit array
int answer = f(R) - f(L - 1);   // crashes/misbehaves when L == 0

// CORRECT
long long lo = (L == 0) ? 0 : f(L - 1);   // int case
long long answer = f(R) - lo;
// string case: guard subtractOne, or rely on problem constraints (L >= 1)
// as LC 2719 and LC 2801 both guarantee.
```

### Mistake 4: Conflating "count of numbers" with "count of digit occurrences"

```cpp
// WRONG (for LC 233) — treating it like a boolean-count problem
int dp(int pos, bool tight) {
    if (pos == len) return 1;              // this only counts COMPLETIONS
    ...
    for (int d = 0; d <= limit; d++)
        res += dp(pos + 1, tight && d == limit) + (d == 1 ? 1 : 0);  // WRONG:
        // adds a flat +1 per digit-1 choice, not scaled by how many
        // completions follow — massively undercounts.

// CORRECT — return (count, ones) and scale properly (Section 8)
pair<long long,long long> dp(int pos, bool tight) {
    if (pos == len) return {1, 0};
    ...
    auto [c, o] = dp(pos + 1, tight && d == limit);
    cnt += c;
    ones += o + (d == 1 ? c : 0);   // scales by c completions, not by 1
}
```

### Mistake 5: Not resetting the memo table between calls to `f()`

```cpp
// WRONG — reusing a class-level memo across f(R) and f(L-1) without resetting
class Solution {
    vector<vector<int>> memo;   // populated once for R's digit string...
    int f(string N) {
        num = N;
        // ...memo NOT reset here...
        return dp(0, 0, true, true);   // now reads stale values computed for
                                         // a DIFFERENT N (wrong len, wrong digits)
    }
};

// CORRECT — memo is tied to the specific num string; reallocate every call
int f(string N) {
    num = N;
    len = num.size();
    memo.assign(len, vector<int>(STATE_SIZE, -1));   // fresh table, sized to THIS N
    vis.assign(len, vector<bool>(STATE_SIZE, false));
    return dp(0, 0, true, true);
}
```

### Mistake 6: Forgetting to prune/bound a growing numeric state, causing array overrun

```cpp
// WRONG (for LC 2719) — sum can theoretically reach 9 * len, but if len is
// computed from a 22-digit string and the memo array is sized for a smaller
// bound (e.g., only up to 100), sum+d indexes out of bounds silently (UB).
int memo[20][100];   // too small: 9*22 = 198 > 100

// CORRECT — size generously and prune early
int memo[23][401];   // safely covers 9*22 = 198, room to spare
int dp(int pos, int sum, bool tight) {
    if (sum > maxSum) return 0;   // prune BEFORE it can index anything with sum
    ...
}
```

---

## SECTION 17 — PROBLEM SET

### WARMUP (solve in ≤ 15 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Count Numbers with Unique Digits | 357 | Basic mask DP, leading-zero-safe mask updates |
| 2 | Numbers At Most N Given Digit Set | 902 | Degenerate digit DP — closed-form free branch |

### CORE (solve in ≤ 30 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Count of Integers | 2719 | Sum-bounded state, big-number `L-1`, sum needs no leading flag |
| 4 | Non-negative Integers without Consecutive Ones | 600 | Binary digit DP, `prev`-bit state |
| 5 | Number of Digit One | 233 | Aggregate `(count, sum)` pair return — NOT boolean counting |

### STRETCH (solve in ≤ 45 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 6 | Count Special Integers | 2376 | Mask DP bounded tightly by arbitrary `n` |
| 7 | Numbers With Repeated Digits | 1012 | Complement trick over LC 2376's engine |
| 8 | Digit Count in Range | 1067 | Per-digit occurrence counting generalized across a range |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 9 | Count Stepping Numbers in Range | 2801 | `last`-digit adjacency state + big-number `L-1` as string |
| 10 | Rotated Digits | 788 | Two-way digit categorization + `hasDiff` boolean state |

---

## SECTION 18 — PATTERN CONNECTIONS

1. **Pattern 24 (Bitmask DP):** The `mask` state in LC 357 (Section 4), LC 2376 (Section 9), and LC 1012 (Section 10) is a direct bitmask-DP state, identical in spirit to `dp[mask]` from Pattern 24 — the only difference is that here the mask lives INSIDE a digit-position recursion instead of standing alone. Anyone comfortable with Pattern 24's "which subset of digits/items have I used" reasoning will recognize this immediately.

2. **Pattern 35 (Combinatorics):** LC 902 (Section 5) is the clearest bridge — the whole problem reduces to a closed-form combinatorial count (`|digits|^length` sums) with only a thin tight-walk on top. Many digit DP problems admit a pure combinatorics solution when the property is "simple" (no memo needed at all); digit DP is the fallback machinery for when the property is complex enough that a closed form doesn't exist (e.g., "digit sum in range" has no simple product formula, forcing Section 6's full DP).

3. **Pattern 16 (Backtracking / State-Space Search):** The digit DP recursion — choose a digit, recurse, and effectively "unchoose" on the next loop iteration — is structurally backtracking with a bounded branching factor (≤10 per level) and, crucially, memoization on top to collapse the exponential tree into a polynomial one. If Pattern 16 taught you to reason about "what does the recursion tree look like, and where can I prune," digit DP is that same tree with an added memo table wherever prefixes are provably interchangeable (`!tight && !leading`).

---

## SECTION 19 — INTERVIEW SIMULATIONS

*(A note on frequency: digit DP is genuinely rare as a cold-open interview question — it's dense, has sharp edge cases, and doesn't test the "can you communicate a tradeoff" skill interviewers usually want. It is, however, extremely common in Codeforces Div 1/Div 2 problem sets (~1900-2400 rated) and in interview FOLLOW-UPS to a simpler counting question ("now count them for x up to 10^18"). Treat these simulations as calibration for that follow-up moment, or for contest self-talk.)*

### Interview Simulation 1: LC 2376 — Count Special Integers

**Interviewer:** "Count integers from 1 to n with all distinct digits."

**Candidate:**

*[First establish why brute force fails, and name the technique]*
> "n can be up to 2×10^9, so I can't check each number one by one. This is a digit DP problem: I'll build the number digit by digit from the most significant digit, tracking which digits I've already used with a 10-bit mask, and two control flags — `tight` (am I still bounded by n's actual digits) and `leading` (am I still in a leading-zero prefix, i.e. haven't placed a real digit yet)."

*[State the base case subtlety before coding]*
> "One trap: if `leading` is still true when I reach the end, that means every digit chosen was 0 — that represents the number 0, which this problem excludes since we start counting at 1. So the base case returns 0 in that scenario, not 1."

*[Code — 6 minutes]*
```cpp
int countSpecialNumbers(int n) {
    string s = to_string(n);
    int len = s.size();
    vector<vector<int>> memo(len, vector<int>(1<<10, -1));
    function<int(int,int,bool,bool)> dp = [&](int pos, int mask, bool tight, bool leading) -> int {
        if (pos == len) return leading ? 0 : 1;
        if (!tight && !leading && memo[pos][mask] != -1) return memo[pos][mask];
        int limit = tight ? s[pos]-'0' : 9;
        int res = 0;
        for (int d = 0; d <= limit; d++) {
            bool nt = tight && d == limit;
            if (leading && d == 0) res += dp(pos+1, mask, nt, true);
            else if (!(mask & (1<<d))) res += dp(pos+1, mask | (1<<d), nt, false);
        }
        if (!tight && !leading) memo[pos][mask] = res;
        return res;
    };
    return dp(0, 0, true, true);
}
```

*[Follow-up]: "Why can't you memoize when `tight` is true?"*
> "Because `tight=true` means the digit choices at this position are capped by n's literal digit — that's a one-off constraint specific to this exact prefix matching n. If I cache that result under `(pos, mask)` and later reach the same `(pos, mask)` under `tight=false` (a fully free context), I'd either overwrite a valid free-context answer with a bounded one, or return a bounded answer where a free one was needed — both wrong."

**RED FLAGS:**
- Memoizing without excluding `tight`/`leading` from the cache condition
- Setting a mask bit for a leading-zero digit
- Returning `1` unconditionally at the base case (miscounts "0" as valid)

---

### Interview Simulation 2: LC 233 — Number of Digit One

**Interviewer:** "Count the total number of digit `1`s that appear when you write out every integer from 1 to n."

**Candidate:**

*[Flag the trap immediately — this is NOT a counting-of-numbers problem]*
> "This is subtly different from 'how many numbers contain a 1' — I need to count every OCCURRENCE. The number 11 contributes two 1s, not one. So a plain digit DP that returns a boolean-style count per completion won't work; I need the recursion to return an aggregate — specifically a pair: how many completions exist from this state, and how many total 1-digits those completions contain combined."

*[Explain the scaling insight before coding]*
> "When I place digit `d=1` at position `pos`, every one of the `c` completions that follow inherits exactly one extra '1' from this choice — so the contribution to the running ones-count is `c`, not `1`. That's the crux of the recursion."

*[Code — 7 minutes, uses the pair-return from Section 8]*

*[Interviewer]: "Can this generalize to counting occurrences of digit 7 instead of 1?"*
> "Yes — the only line that changes is `(d == 1 ? c : 0)` becomes `(d == 7 ? c : 0)`. The count/ones pair-return structure is identical."

**RED FLAGS:**
- Returning a single scalar count instead of a `(count, aggregate)` pair
- Adding a flat `+1` per digit-1 choice instead of `+c` (undercounts badly on multi-digit ranges)
- Confusing this with LC 2376's "does this number qualify" boolean framing

---

### Interview Simulation 3: Meta-discussion — When does a counting problem need digit DP?

**Interviewer:** "How do you recognize a problem needs digit DP instead of a simpler technique?"

**Candidate:**
> "Three signals, usually together: (1) the range bound is enormous — beyond what any iteration or even O(n) preprocessing could touch, often stated as up to `10^9`, `10^15`, or given as a string because it doesn't even fit in 64 bits. (2) The property being counted is checkable on ONE number in O(digits) time, but has no obvious closed-form formula (if it did, like 'numbers divisible by 7', I'd just use `R/7 - (L-1)/7`, no DP needed). (3) The property is naturally 'local' or 'prefix-summarizable' — it can be verified by scanning digits left to right while remembering only a small amount of state (a mask, a running sum, a last digit) rather than the whole prefix.

> If a problem satisfies all three, digit DP applies: build the digit array of the upper bound, recurse position by position with `tight`/`leading` plus whatever compressed state the property needs, and reduce range queries via `f(R) - f(L-1)`.

> The single biggest 'gotcha' I watch for before coding: does the answer want a COUNT of qualifying numbers, or some AGGREGATE over them (a sum, a total occurrence count)? That decides whether the recursion returns a scalar or a pair."

**RED FLAGS:**
- Reaching for digit DP when a closed-form divisibility/counting formula exists (over-engineering)
- Not identifying upfront whether the query is a count or an aggregate
- Forgetting that `f(R) - f(L-1)` — not `f(R) - f(L)` — is the standard range reduction

---

## SECTION 20 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: Why is memoization only valid when `!tight && !leading`?**

> Both flags encode context that makes the current `(pos, state)` cell NOT interchangeable with other calls to the same cell. `tight=true` means the allowed digit range at this position is capped by `N`'s literal digit — a one-off constraint tied to this exact prefix matching `N` so far; there is at most one live tight path in the whole tree at any time, so caching it is both wasteful and, if keyed the same as free calls, actively wrong (it would return a bounded answer where an unbounded one was needed, or vice versa). `leading=true` means the state (e.g., `mask=0`) represents "no real digit placed yet, still padding" — a DIFFERENT situation from `mask=0` reached with `leading=false` (which, depending on the problem, may be impossible or mean something else entirely). Only when both flags are false is `(pos, state)` a "generic, fully free" subproblem whose answer is genuinely reusable regardless of how we arrived there.

---

**Q2: In LC 2719 (Count of Integers), why doesn't the `sum` state need a `leading` flag, while LC 357's `mask` state does?**

> A leading zero contributes exactly `0` to a running digit sum — treating it as an ordinary zero-digit is completely harmless; `sum + 0 = sum`, no distortion. But a leading zero contributes exactly nothing to a used-digit mask conceptually TOO, except the mask ALSO needs to know when a nonzero digit has appeared, because two different numbers with the same nonzero digits but different amounts of leading padding (e.g. "5" vs "05") must be treated as the identical number — and the mask must not record that "0" was ever "used" for either, since "50" (which really does use digit 0) must be distinguished from "5" (which doesn't). The mask's binary "used or not" semantics break under padding in a way that a running numeric sum's "add zero, no-op" semantics don't.

---

**Q3: What does the recursion return for LC 233, and why can't it return a single integer?**

> It returns a pair `(count of completions, total count of digit-1 occurrences across those completions)`. A single integer can't work because the contribution of choosing `d=1` at a given position isn't "add one" — it's "add one to EACH of the `c` completions that follow," i.e. `+c`. Without tracking `c` (the completion count) alongside the aggregate, there's no way to compute that scaled contribution. This pair-return pattern generalizes to any "sum over valid numbers" digit DP (sum of digit sums, sum of the numbers themselves, etc.).

---

**Q4: Why is `f(R) - f(L-1)` used instead of `f(R) - f(L)`?**

> `f(N)` is defined as "count of valid numbers in `[0, N]`" (inclusive on both ends). To get the count in `[L, R]` inclusive, we need `f(R)` (everything up to and including `R`) minus everything STRICTLY BELOW `L`, which is `f(L-1)` (everything up to and including `L-1`). Using `f(L)` instead would subtract off `L` itself, wrongly excluding it from the final count whenever `L` itself satisfies the property.

---

**Q5: Why does LC 2801 (Stepping Numbers) need BOTH a `last`-digit state AND a `leading` flag, while LC 600 (Consecutive Ones) needs only `prev` and no `leading`?**

> In LC 2801, `last` must only start being enforced once a REAL digit has been placed — the very first real digit has no "previous digit" to compare against, so we need `leading` to know when to skip the `abs(d-last)==1` check versus when to enforce it. In LC 600, the check is `prev==1 && d==1` (forbidding consecutive ONE bits specifically) — and `prev` starts at `0`, which is exactly the correct "no violation possible yet" baseline whether that `0` is a real bit or a padding bit. Since a leading zero and the initial `prev=0` value behave IDENTICALLY for this specific check, no separate `leading` flag is needed. The general rule: you need `leading` whenever a leading zero would behave differently from a "real" zero for the property being tracked; LC 2801's adjacency check differs (no adjacency check on the very first digit) while LC 600's doesn't.

---

**Q6: In the big-number `subtractOne` function, why do we strip leading zeros after decrementing, and what would break if we didn't?**

> Decrementing a string like `"100"` produces `"099"` before stripping — a valid representation of the number `99`, but with an extra leading-zero digit compared to the "natural" 2-digit string `"99"`. If we build the digit-DP array directly from `"099"` without stripping, `len` would be `3` instead of `2`. This is USUALLY still correct because a properly implemented `leading` flag treats the extra leading zero as harmless padding — but it wastes a recursion level, risks off-by-one errors if any part of the code assumes `len` equals the "true" digit count of the number (e.g., sizing arrays, or any place accidentally comparing `pos` against a hardcoded expected length), and is simply not the invariant you want to reason about under time pressure. Stripping keeps `len` meaning exactly what it should: the true digit count of the target number.

---

**Q7: How would you extend LC 2376's mask-DP to instead count numbers where at most K digits repeat (not zero repeats)?**

> The bitmask alone isn't enough — you'd need to track HOW MANY repeat-events have occurred, not just WHICH digits are used. Extend the state to `(mask, repeatCount)`, where `repeatCount` increments by 1 every time a chosen digit `d` is already set in `mask` (instead of forbidding the choice outright as LC 2376 does). The base case then checks `repeatCount <= K` instead of always returning `1`. This is a direct illustration of Section 15's "concatenate state dimensions" principle: the skeleton is unchanged, only the extra-state tuple grows from `mask` to `(mask, repeatCount)`, and the memo table grows a dimension to match.

---

## SECTION 21 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. THE UNIVERSAL SKELETON
// ─────────────────────────────────────────────────────────────────
long long dp(int pos, /*state*/, bool tight, bool leading) {
    if (pos == len) return /* base-case check on state/leading */;
    if (!tight && !leading && vis[pos][state]) return memo[pos][state];
    int limit = tight ? (num[pos]-'0') : 9;
    long long res = 0;
    for (int d = 0; d <= limit; d++) {
        bool nt = tight && d == limit;
        bool nl = leading && d == 0;
        /* newState = transition(state, d, leading) */
        res += dp(pos+1, /*newState,*/ nt, nl);
    }
    if (!tight && !leading) { vis[pos][state]=true; memo[pos][state]=res; }
    return res;
}

// ─────────────────────────────────────────────────────────────────
// 2. RANGE REDUCTION
// ─────────────────────────────────────────────────────────────────
long long answer = f(R) - ((L == 0) ? 0 : f(L - 1));   // int case
// string case: guard L=="0" per problem constraints, else:
long long answer = ((f(high) - f(subtractOne(low))) % MOD + MOD) % MOD;

// ─────────────────────────────────────────────────────────────────
// 3. BIG-NUMBER "N - 1" (STRING)
// ─────────────────────────────────────────────────────────────────
string subtractOne(string s) {
    int i = (int)s.size()-1;
    while (i >= 0 && s[i]=='0') { s[i]='9'; i--; }
    s[i]--;
    int start=0;
    while (start+1 < (int)s.size() && s[start]=='0') start++;
    return s.substr(start);
}

// ─────────────────────────────────────────────────────────────────
// 4. MASK STATE (unique/repeated digits — 357, 2376, 1012)
// ─────────────────────────────────────────────────────────────────
if (leading && d == 0) {
    res += dp(pos+1, mask, nt, true);            // padding: mask untouched
} else {
    if (mask & (1<<d)) continue;                 // forbid repeat
    res += dp(pos+1, mask | (1<<d), nt, false);
}

// ─────────────────────────────────────────────────────────────────
// 5. RUNNING-SUM STATE (digit sum in range — 2719)
// ─────────────────────────────────────────────────────────────────
if (sum > maxSum) return 0;                       // prune
// no leading flag needed: a padding zero adds 0, harmless
res += dp(pos+1, sum+d, nt);

// ─────────────────────────────────────────────────────────────────
// 6. LAST-DIGIT STATE (adjacency — 2801)
// ─────────────────────────────────────────────────────────────────
if (leading && d == 0)      res += dp(pos+1, 0, nt, true);
else if (leading)           res += dp(pos+1, d, nt, false);   // first real digit
else if (abs(d-last) == 1)  res += dp(pos+1, d, nt, false);

// ─────────────────────────────────────────────────────────────────
// 7. AGGREGATE (count, sum)-PAIR RETURN (occurrence counting — 233)
// ─────────────────────────────────────────────────────────────────
pair<long long,long long> dp(int pos, bool tight) {
    if (pos == len) return {1, 0};
    ...
    auto [c, o] = dp(pos+1, tight && d==limit);
    cnt += c;
    ones += o + (d == TARGET_DIGIT ? c : 0);      // scale by completions, not by 1
}

// ─────────────────────────────────────────────────────────────────
// 8. COMPLEMENT TRICK (has-property = total - lacks-property — 1012)
// ─────────────────────────────────────────────────────────────────
int numDupDigitsAtMostN(int n) { return n - helperCountUnique(to_string(n)); }

// ─────────────────────────────────────────────────────────────────
// 9. DECISION TABLE: DOES THIS STATE NEED A LEADING-ZERO GUARD?
// ─────────────────────────────────────────────────────────────────
// YES — a leading zero behaves DIFFERENTLY from a real zero:
//   used-digit masks (357, 2376, 1012), first-digit-no-adjacency (2801),
//   has-a-diff-rotation-digit (788), base case must reject all-zero "0" (2376,2801,788)
// NO — a leading zero behaves IDENTICALLY to a real zero:
//   running digit sum (2719: sum += 0 is a no-op),
//   binary consecutive-ones prev-bit (600: prev=0 is the correct starting baseline)
```

---

## SECTION 22 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 10 problems in order.** LC 233 and LC 2801 are the two requiring genuine new insight (aggregate pair-return; last-digit adjacency + big-number subtraction). Budget 45 minutes each on first attempt.

2. **Do the "why not `f(R)-f(L)`" exercise on paper.** Pick any of the ten problems, compute `f(R)`, `f(L)`, and `f(L-1)` by hand for a tiny range, and confirm which combination gives the right answer. This cements the boundary logic permanently.

3. **Rebuild LC 357 from scratch without looking, twice.** Once as digit DP (this document's approach), once using the pure combinatorial formula (`9 × 9 × 8 × 7 × ... `). Comparing the two answers on the same `n` is the fastest way to internalize what digit DP buys you over closed-form counting — and to recognize when a closed form exists and digit DP is overkill.

4. **Deliberately break the memo rule and observe the failure.** Take LC 2376's working solution, remove the `!tight && !leading` guard from the memo condition (memoize unconditionally), and run it against a case where `tight` stays true for at least two adjacent digits differing from `N`'s digits nearby. Watching it produce a wrong answer — and figuring out exactly which cached cell was polluted — builds intuition no amount of reading can.

5. **Preview Pattern 24 (Bitmask DP)** if not already covered: the `mask` state used across 357/2376/1012 is literally a bitmask-DP dimension nested inside a digit-position recursion. Understanding Pattern 24's `dp[mask]` transitions in isolation makes this pattern's mask-handling feel obvious rather than magical.

---

## SECTION 23 — SIGN-OFF CRITERIA

### Tier 1 — Mechanics understood
You can write the bare skeleton (Section 3) from memory, correctly implementing `tight`/`leading` transitions, and explain in one sentence why memoization requires `!tight && !leading`. You solve LC 357 and LC 902 cleanly.

### Tier 2 — State design solid
You solve LC 2719 (sum state, correctly reasoning about why no leading flag is needed), LC 600 (binary digit DP, prev-bit state), and can explain — for any new problem — what extra state the transition needs before writing a single line of code.

### Tier 3 — Aggregate and complement techniques mastered
You solve LC 233 using the `(count, aggregate)` pair-return pattern without referring back to this document, and can explain why a scalar return is insufficient. You solve LC 2376 and derive LC 1012 as its complement in under 5 minutes once 2376 is done.

### Tier 4 — Big-number and multi-dimensional state complete
You solve LC 2801 (last-digit state + string-based `L-1` subtraction) and LC 788 (categorical digit state) unassisted. You can implement `subtractOne` for a decimal string from memory and correctly trace it on an input ending in multiple zeros (e.g., `"1000"`).

**Sign-off threshold:** Solve 8 of 10 problems. Mandatory: LC 357 (mask DP fundamentals), LC 233 (aggregate pair-return — the sharpest "count vs. sum" trap in the pattern), and LC 2719 or LC 2801 (big-number range handling). These three cover the three conceptual axes of digit DP: state design, return-value design, and range/precision handling.

---

*Pattern 25 Complete — Digit DP*
*Next: Pattern 26 — (continue the 1600→2000 curriculum)*
