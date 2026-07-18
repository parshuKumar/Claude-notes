# PATTERN 34 — GREEDY (FORMAL TREATMENT)
## Exchange Arguments, Stays-Ahead Proofs, and the Line Between Greedy and DP
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**Greedy is not "the obvious thing." Greedy is a proof.**

At 1600, "greedy" means: sort the array somehow, walk through it once, make the choice that looks best right now. Most of the time this *works*, and most solvers never ask *why*. That's Tier 1 greedy — pattern matching against problems you've seen before.

At 2000, a greedy algorithm is only correct if you can **prove** that the locally optimal choice at every step never forecloses a globally optimal solution. Without that proof, "greedy" is just "an algorithm I haven't found a counterexample for yet." Codeforces Div 2 problems are specifically designed to punish this — a greedy strategy that passes examples 1-3 and fails on test 47 is the single most common way to lose rating on a greedy problem.

**Why this matters more than for any other pattern:**

- DP is self-verifying in a sense: if your recurrence is correct, you've implicitly explored the whole search space, so correctness follows from the state definition.
- Two pointers / sliding window are usually correct by monotonicity arguments that are visually obvious.
- Greedy has **no built-in safety net**. You make an irrevocable choice at every step and never look back. If the greedy choice is wrong even once, the whole algorithm is wrong — and it will *look* right on small, friendly test cases.

**The two things a greedy solution needs, in order:**

1. **A candidate strategy** — sort by X, then process left to right, always picking Y. (This is the easy part; brainstorming usually gets you here in under a minute.)
2. **A proof that the strategy is optimal** — via exchange argument or greedy-stays-ahead (Section 2). This is the part that separates a correct greedy submission from a "looks-plausible" WA.

**When greedy fails and DP is required:**

Greedy fails whenever the locally best choice can create a **worse global outcome that cannot be recovered from**. Two canonical examples:

- **0/1 Knapsack:** Sorting by value/weight ratio and greedily taking items is optimal for the *fractional* knapsack (you can take part of an item) — the exchange argument holds because fractional swaps are always possible. It **fails** for 0/1 knapsack, because you cannot "trade back" part of an already-taken item once you discover it doesn't fit with a later, better item. Counterexample: capacity 10, items (weight 6, value 10) and (weight 4, value 6) and (weight 4, value 6). Greedy by ratio picks item 1 (ratio 1.67) then can't fit either of the other two → total value 10. Optimal: skip item 1, take both remaining items → weight 8, value 12. Greedy loses.
- **Coin Change with arbitrary denominations:** Greedy ("always take the largest coin ≤ remaining amount") works for canonical systems like US currency, but fails for arbitrary denominations. Counterexample: coins = {1, 3, 4}, target = 6. Greedy takes 4, then 1, then 1 → 3 coins (4+1+1). Optimal: 3+3 → 2 coins. Greedy loses.

**The diagnostic question:** *"If I make this greedy choice now, can any future decision ever make me regret it — in a way I cannot compensate for?"* If yes, you need DP (which explores all choices and keeps the best) instead of committing early. This document is about the problems where the answer is provably "no."

---

## SECTION 2 — THE TWO PROOF TECHNIQUES

There are exactly two standard templates for proving a greedy algorithm correct. Almost every greedy proof you will ever write is one of these two, dressed up for the specific problem.

### (a) EXCHANGE ARGUMENT

**Statement:** Assume, for contradiction, that there exists an optimal solution `OPT` that disagrees with the greedy solution `G`. Find the *first* point where they differ. Show that you can modify `OPT` — by swapping/exchanging the disagreeing element for greedy's choice — without making `OPT` worse (and without breaking feasibility). This produces a new optimal solution `OPT'` that agrees with `G` on one more element than before. Repeat this exchange until `OPT'` = `G`. Since every exchange step never worsens the objective, `G` is optimal.

**How to apply it, step by step:**
1. Assume optimal `OPT` ≠ greedy `G`. Let `i` be the first index/position where they differ.
2. Look at what `OPT` chose at position `i` versus what `G` chose.
3. Argue that swapping `G`'s choice into `OPT`'s solution at position `i` keeps the solution feasible (doesn't violate constraints).
4. Argue that the swap doesn't decrease (for maximization) or increase (for minimization) the objective.
5. Conclude: an optimal solution exists that agrees with `G` on the first `i` positions. Induct.

**Worked example — Activity Selection (sort by finish time):**

Claim: greedily picking the activity with the earliest finish time among all remaining compatible activities produces a maximum-size set of non-overlapping activities.

Proof: Let `OPT` be an optimal solution, sorted by start time (equivalently finish time, since they're non-overlapping). Let `g` be greedy's first pick (earliest finish time overall) and `o` be `OPT`'s first pick. Since `g` has the earliest finish time among *all* activities, `f(g) ≤ f(o)`.

Now build `OPT' = OPT` with `o` replaced by `g`. Is `OPT'` still feasible? Every other activity in `OPT` starts after `f(o)` (since `OPT` is non-overlapping and `o` is first). Since `f(g) ≤ f(o)`, every other activity in `OPT` also starts after `f(g)`. So `g` doesn't overlap with anything else in `OPT'`. `OPT'` is feasible, same size as `OPT`, hence also optimal, and now agrees with `G` on the first pick. Repeat this argument on the remaining sub-problem (activities that start after `f(g)`). By induction, greedy is optimal. ∎

### (b) GREEDY STAYS AHEAD

**Statement:** Define a measurable quantity that captures "how good is the solution built so far" after `k` steps (e.g., the finish time of the k-th chosen activity, the farthest index reachable, the accumulated tank). Prove by induction on `k` that greedy's measurement is *at least as good* as any other valid solution's measurement at step `k`. Since greedy never falls behind at any prefix, it cannot finish behind at the end.

**How to apply it, step by step:**
1. Define a per-step measure `M(k)` that is monotonic with the final objective.
2. Base case: show `M_greedy(1)` is at least as good as `M_any(1)`.
3. Inductive step: assume `M_greedy(k) ≥ M_any(k)` (or `≤`, for minimization). Show this implies `M_greedy(k+1) ≥ M_any(k+1)`, using the fact that greedy had at least as much "room"/"budget" at step `k` as any competitor.
4. Conclude greedy's final measure dominates, hence greedy achieves an optimal (or no-worse) result.

**Worked example — Activity Selection, again, via stays-ahead (to show both techniques apply):**

Let `g_1, g_2, ..., g_k` be greedy's picks in order of finish time, and let `o_1, ..., o_m` be *any* valid non-overlapping selection, sorted by finish time. Claim: for every `k ≤ m`, `f(g_k) ≤ f(o_k)`.

Base case `k=1`: greedy picks the globally earliest-finishing activity, so `f(g_1) ≤ f(o_1)` trivially.

Inductive step: assume `f(g_k) ≤ f(o_k)`. Since the `o` sequence is non-overlapping, `o_{k+1}` starts at or after `f(o_k) ≥ f(g_k)` — so `o_{k+1}` is a valid candidate for greedy's `(k+1)`-th pick (it's compatible with `g_1..g_k`). Greedy, faced with this candidate pool, picks the minimum-finish-time option, so `f(g_{k+1}) ≤ f(o_{k+1})`.

By induction this holds for all `k ≤ m`. In particular it means greedy is *never* stuck before any other solution would be — if `OPT` can extend to `m` activities, greedy's prefix at every step has an equal-or-earlier finish time, so greedy can always extend at least as far. Hence greedy achieves size ≥ `m` for any valid `OPT`, i.e., greedy is optimal. ∎

**Which technique to use when:** Exchange arguments are usually cleaner for "sort and pick" problems where solutions are *sets* (Activity Selection, Assign Cookies, Minimum Arrows). Stays-ahead is usually cleaner for problems with a running/accumulating quantity that must never dip below a threshold (Jump Game, Gas Station, Full Bloom). Both are acceptable in an interview — pick whichever is faster to state precisely.

---

## SECTION 3 — TEMPLATE 1: ACTIVITY SELECTION / INTERVAL SCHEDULING

```cpp
// ─────────────────────────────────────────────────────────────────
// Classic Activity Selection (foundation for LC 435, LC 452, LC 646)
// Given n activities with [start, end) times, select the maximum
// number of NON-OVERLAPPING activities.
//
// GREEDY RULE: sort by END time. Always take the next activity whose
// start >= the end of the last taken activity.
//
// PROOF: see Section 2(a) — exchange argument. Sorting by end time
// and taking greedily never does worse than any optimal solution,
// because the earliest-finishing compatible activity leaves maximum
// room for everything after it.
// ─────────────────────────────────────────────────────────────────

int activitySelection(vector<pair<int,int>>& activities) {
    sort(activities.begin(), activities.end(),
         [](const pair<int,int>& a, const pair<int,int>& b) {
             return a.second < b.second;   // sort by END time
         });

    int count = 0;
    int lastEnd = INT_MIN;

    for (auto& [start, end] : activities) {
        if (start >= lastEnd) {     // compatible with last chosen activity
            count++;
            lastEnd = end;
        }
    }
    return count;
}
```

**TRACE for activities = [(1,4),(3,5),(0,6),(5,7),(3,8),(5,9),(6,10),(8,11),(8,12),(2,13),(12,14)]:**
```
Sorted by end time:
(1,4) (3,5) (0,6) (5,7) (3,8) (5,9) (6,10) (8,11) (8,12) (2,13) (12,14)

Take (1,4): count=1, lastEnd=4
(3,5): start=3 < 4 -> skip
(0,6): start=0 < 4 -> skip
(5,7): start=5 >= 4 -> take. count=2, lastEnd=7
(3,8): skip (3<7)
(5,9): skip (5<7)
(6,10): skip (6<7)
(8,11): start=8>=7 -> take. count=3, lastEnd=11
(8,12): skip
(2,13): skip
(12,14): start=12>=11 -> take. count=4

Answer: 4 activities: (1,4),(5,7),(8,11),(12,14)
```

**Why sort by end time and not by start time or by duration?** Sorting by start time greedily fails: a short activity that starts early but ends very late can block many later activities. Sorting by *duration* also fails: a short activity might still end late (e.g., starts at 100, duration 1 → ends 101, but blocks everything until 101 while a longer activity ending at 10 would free up the rest of the timeline). Only end time guarantees "leaves maximum room for the future," which is exactly what the exchange argument in Section 2(a) formalizes.

---

## SECTION 4 — TEMPLATE 2: ASSIGN COOKIES (LC 455)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 455 — Assign Cookies
// Each child i has a greed factor g[i] (minimum cookie size to be
// content). Each cookie has size s[j]. One cookie per child.
// Maximize the number of content children.
//
// GREEDY RULE: sort both g and s ascending. Use two pointers.
// Try to satisfy the LEAST greedy unsatisfied child with the
// SMALLEST unused cookie that is big enough.
//
// PROOF (exchange argument):
//   Suppose OPT assigns a cookie s[j] to child g[i] where a smaller
//   cookie s[j'] < s[j] also exists and satisfies g[i] (s[j'] >= g[i])
//   but is unused or assigned to a LESS greedy child g[i''] (g[i'']
//   < g[i], so g[i''] would also be satisfied by an even smaller
//   cookie, or doesn't need s[j'] at all).
//   Swapping: give the least greedy remaining child the smallest
//   sufficient cookie. This never uses "more cookie than necessary"
//   on any child, which can only free up larger cookies for
//   greedier children later — never worse than OPT, so this
//   exchange preserves optimality. Repeating this exchange
//   transforms OPT into greedy without ever decreasing the count.
// ─────────────────────────────────────────────────────────────────

int findContentChildren(vector<int>& g, vector<int>& s) {
    sort(g.begin(), g.end());
    sort(s.begin(), s.end());

    int i = 0;                 // pointer into children (least greedy first)
    int j = 0;                 // pointer into cookies (smallest first)
    int satisfied = 0;

    while (i < g.size() && j < s.size()) {
        if (s[j] >= g[i]) {    // this cookie satisfies this child
            satisfied++;
            i++;                // move to next child
        }
        j++;                    // this cookie is consumed either way
    }
    return satisfied;
}
```

**TRACE for g=[1,2,3], s=[1,1]:**
```
sorted g=[1,2,3], sorted s=[1,1]
i=0,j=0: s[0]=1 >= g[0]=1 -> satisfied=1, i=1
i=1,j=1: s[1]=1 >= g[1]=2? NO -> j=2
j=2 == s.size() -> loop ends

Answer: 1 content child
```

**Why "smallest sufficient cookie" and not "biggest available cookie"?** Giving a big cookie to a child who only needs a small one *wastes* capacity that could satisfy a greedier child later. The exchange argument formalizes this: any assignment that "overspends" a cookie on an easily-satisfied child can be exchanged for a cheaper cookie without hurting that child (they only need `s[j] >= g[i]`, any sufficient cookie works equally well from their perspective), freeing the bigger cookie for someone who actually needs it.

---

## SECTION 5 — TEMPLATE 3: JUMP GAME (LC 55)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 55 — Jump Game
// nums[i] = max jump length from index i. Starting at index 0,
// determine if you can reach the last index.
//
// GREEDY RULE: track the farthest index reachable so far. If at
// any point the current index exceeds the farthest reachable index,
// you're stuck. If farthest ever reaches/exceeds the last index,
// success.
//
// PROOF (greedy stays ahead):
//   Let farthest(i) = max over all j <= i of (j + nums[j]), i.e.
//   the farthest index reachable using only indices 0..i.
//   CLAIM: farthest(i), maintained incrementally, is a valid upper
//   bound on reachability regardless of WHICH path you took to get
//   to any j <= i. Reachability is monotonic: if index i is
//   reachable, then i is reachable from *some* prior reachable
//   index, and farthest(i) >= i + nums[i] accounts for that jump.
//   Since we take the MAX over all prior reachable positions,
//   farthest(i) is always the true maximum reachable index using
//   the first i+1 cells — no other strategy can do better, because
//   any strategy's reachable set at step i is a subset of "all
//   indices reachable via some prefix," which is exactly what
//   farthest(i) computes. Greedy "stays ahead" trivially because it
//   IS the maximum, not merely an approximation of it.
// ─────────────────────────────────────────────────────────────────

bool canJump(vector<int>& nums) {
    int n = nums.size();
    int farthest = 0;

    for (int i = 0; i < n; i++) {
        if (i > farthest) return false;    // current cell is unreachable
        farthest = max(farthest, i + nums[i]);
        if (farthest >= n - 1) return true; // early exit
    }
    return true;
}
```

**TRACE for nums=[2,3,1,1,4]:**
```
i=0: 0<=farthest(0). farthest=max(0,0+2)=2
i=1: 1<=2. farthest=max(2,1+3)=4. 4>=n-1=4 -> return true
```

**TRACE for nums=[3,2,1,0,4] (should fail):**
```
i=0: farthest=max(0,3)=3
i=1: farthest=max(3,1+2)=3
i=2: farthest=max(3,2+1)=3
i=3: farthest=max(3,3+0)=3
i=4: i=4 > farthest=3 -> return false
```
Index 4 (value 4, could jump huge distances) is never reached because index 3 has value 0 and everything before it caps out at index 3.

---

## SECTION 6 — TEMPLATE 4: JUMP GAME II (LC 45, MINIMUM JUMPS)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 45 — Jump Game II
// Same setup as Jump Game, but now find the MINIMUM number of jumps
// to reach the last index (guaranteed reachable).
//
// GREEDY RULE: this is BFS "by levels" in disguise. Track:
//   currentEnd   = farthest index reachable using the jumps taken SO FAR
//   farthest     = farthest index reachable using ONE MORE jump from
//                   anywhere in the current level
// When i reaches currentEnd, we've exhausted the current jump count —
// commit to the next jump: jumps++, currentEnd = farthest.
//
// PROOF (greedy stays ahead, BFS-layer argument):
//   Think of index 0 as BFS layer 0. Layer k = the set of indices
//   reachable in EXACTLY k jumps (well-defined as a contiguous-ish
//   frontier because jumps only move forward). farthest, as computed
//   while scanning layer k, is exactly max reachable index using k+1
//   jumps — by the same argument as Jump Game's farthest(i). Since
//   we always advance currentEnd to the TRUE maximum reachable
//   frontier, no strategy can reach a farther index in k+1 jumps
//   than greedy's frontier. Hence greedy reaches the last index in
//   the minimum possible number of layers = minimum number of jumps.
// ─────────────────────────────────────────────────────────────────

int jump(vector<int>& nums) {
    int n = nums.size();
    int jumps = 0, currentEnd = 0, farthest = 0;

    for (int i = 0; i < n - 1; i++) {   // no need to process last index
        farthest = max(farthest, i + nums[i]);
        if (i == currentEnd) {          // exhausted this jump's range
            jumps++;
            currentEnd = farthest;
            if (currentEnd >= n - 1) break;  // early exit
        }
    }
    return jumps;
}
```

**TRACE for nums=[2,3,1,1,4]:**
```
n=5, jumps=0, currentEnd=0, farthest=0

i=0: farthest=max(0,0+2)=2. i==currentEnd(0) -> jumps=1, currentEnd=2
i=1: farthest=max(2,1+3)=4. i(1)!=currentEnd(2)
i=2: farthest=max(4,2+1)=4. i==currentEnd(2) -> jumps=2, currentEnd=4. 4>=n-1(4) -> break

Answer: 2 jumps. (0->1->4, i.e. jump to index1 then to index4)
```

**Why this is correct BFS in O(n) instead of O(n^2):** A naive BFS would enqueue every reachable index explicitly. This greedy compresses each BFS layer into two integers (`currentEnd`, `farthest`), because within a layer we only care about the layer's rightmost boundary, never individual indices. This is a common greedy-as-compressed-BFS trick worth recognizing.

---

## SECTION 7 — TEMPLATE 5: GAS STATION (LC 134, CIRCULAR GREEDY)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 134 — Gas Station
// n stations in a circle. gas[i] = fuel gained at station i.
// cost[i] = fuel needed to travel from station i to station i+1.
// Starting with an empty tank at some station, determine the
// starting index that allows a full circular trip, or -1 if none
// exists. Guaranteed: if a solution exists, it is unique.
//
// GREEDY RULE: scan once. Track running tank (using diff[i] =
// gas[i]-cost[i]) and total. Whenever tank goes negative at station
// i, NO station from the current start through i can be a valid
// start — reset start = i+1, tank = 0. If total >= 0 at the end,
// the final `start` is the answer.
// ─────────────────────────────────────────────────────────────────

int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int n = gas.size();
    long long total = 0, tank = 0;
    int start = 0;

    for (int i = 0; i < n; i++) {
        int diff = gas[i] - cost[i];
        total += diff;
        tank += diff;
        if (tank < 0) {
            // No station in [start, i] can start a full loop.
            start = i + 1;
            tank = 0;
        }
    }

    return (total >= 0) ? start : -1;
}
```

**PROOF, part 1 — why `total >= 0` is necessary and sufficient for SOME solution to exist:**

Necessary: summing fuel used over the full circle equals `total`. If `total < 0`, the car burns more fuel than it gains over one full loop regardless of starting point, so it can never complete a full circuit from any start. Trivial.

Sufficient (this is the non-obvious half): if `total >= 0`, a valid start exists. Let `P[i]` be the prefix sum of `diff[0..i-1]` (so `P[0] = 0`, `P[n] = total >= 0`). Let `s` be an index minimizing `P[s]` over `0 <= s <= n-1` (the point where the running prefix sum is most negative — i.e., right after the worst possible deficit). **Claim: starting at `s` works.**

For any `i` with `s <= i <= n-1` (no wraparound yet): the fuel accumulated from `s` to `i` is `P[i+1] - P[s]`. Since `P[s]` is the minimum over `0..n-1` and `i+1` ranges over `1..n`, we have `P[i+1] >= P[s]` (the only index not directly covered by the minimality range is `i+1=n`, but `P[n]=total>=0>=... ` — handled below), so this is `>= 0`. Tank never goes negative before wrapping.

For any `j` with `0 <= j < s` (after wrapping past station `n-1` back to station `0`): fuel accumulated from `s` all the way around to `j` is `(P[n] - P[s]) + P[j+1] = total - P[s] + P[j+1]`. Since `s` is the minimizer over `0..n-1` and `j+1 <= s`, we have `P[s] <= P[j+1]`, i.e., `P[j+1] - P[s] >= 0`. So the expression is `total + (P[j+1]-P[s]) >= total >= 0`. Tank never goes negative after wrapping either.

Hence starting at `s` (the point right after the deepest deficit), the tank never dips below zero — a full circuit is achievable. ∎ This is *exactly* the index the reset algorithm produces: every time `tank` dips below 0 during the scan, it's a signal that the running prefix from the current `start` has hit a new low, so we reset to just past it — which is precisely tracking the running argmin of `P`.

**PROOF, part 2 — why the reset point is correct, more intuitively (exchange argument flavor):** Suppose the car starts at `start` and the tank first goes negative at station `i`. Claim: no station `k` with `start <= k <= i` can be a valid starting point either. Why? The fuel from `start` to `k` is `>= 0` (tank hadn't gone negative yet at `k`, by definition of `i` being the FIRST negative point). So the fuel from `k` to `i` equals (fuel from `start` to `i`) minus (fuel from `start` to `k`) `<=` (fuel from `start` to `i`), which is negative. So starting at `k` and driving to `i` also results in negative tank — `k` is disqualified too. This justifies skipping the entire range `[start, i]` in one shot and resetting to `i+1`, which is what the algorithm does.

**TRACE for gas=[1,2,3,4,5], cost=[3,4,5,1,2]:**
```
diff = [-2,-2,-2,3,3]

i=0: total=-2, tank=-2. tank<0 -> start=1, tank=0
i=1: total=-4, tank=-2. tank<0 -> start=2, tank=0
i=2: total=-6, tank=-2. tank<0 -> start=3, tank=0
i=3: total=-3, tank=3.
i=4: total=0,  tank=6.

total=0 >= 0 -> answer = start = 3
Verify: start at station 3. tank: 4-1=3, then 3+5-2=6, then 6+1-3=4, then 4+2-4=2, then 2+3-5=0. Never negative. ✓
```

**Edge case to be careful about:** if `total == 0` exactly, the car finishes with exactly 0 fuel at the last leg — still valid (the problem only requires the tank to never go *negative*, ending at exactly 0 is fine).

---

## SECTION 8 — TEMPLATE 6: CANDY (LC 135, TWO-PASS GREEDY)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 135 — Candy
// n children in a line, each with a rating. Each child gets at
// least 1 candy. A child with a higher rating than an immediate
// neighbor must get MORE candy than that neighbor. Minimize total
// candy.
//
// GREEDY RULE: two independent left-to-right constraints combined.
//   Pass 1 (left -> right): if ratings[i] > ratings[i-1], then
//     candy[i] must be > candy[i-1]. Greedily set candy[i] = candy[i-1]+1.
//   Pass 2 (right -> left): if ratings[i] > ratings[i+1], then
//     candy[i] must be > candy[i+1]. Greedily set
//     candy[i] = max(candy[i], candy[i+1]+1).
// Sum the final candy array.
//
// PROOF: candy[i] must satisfy BOTH constraints independently
// (with left neighbor AND right neighbor). Pass 1 computes the
// MINIMUM candy[i] needed to satisfy the left-neighbor constraint
// alone (a simple chain: each "ascending run" from the left gets
// 1,2,3,...). Pass 2 computes the minimum needed for the
// right-neighbor constraint alone, similarly. Taking the max of the
// two per index satisfies both constraints simultaneously, because:
//   - It's clearly a LOWER BOUND satisfier: max(a,b) >= a and >= b,
//     so both individual constraints (which only require a
//     particular direction) remain satisfied.
//   - It's MINIMAL: any smaller value would violate at least one of
//     the two passes' requirement, and each pass's requirement is
//     itself the smallest value satisfying that direction (proved
//     inductively: pass 1's candy[i]=candy[i-1]+1 is forced whenever
//     ratings[i]>ratings[i-1], and cannot be smaller).
// ─────────────────────────────────────────────────────────────────

int candy(vector<int>& ratings) {
    int n = ratings.size();
    vector<int> candies(n, 1);

    // Pass 1: enforce left-neighbor constraint
    for (int i = 1; i < n; i++) {
        if (ratings[i] > ratings[i-1]) {
            candies[i] = candies[i-1] + 1;
        }
    }

    // Pass 2: enforce right-neighbor constraint, combine with pass 1
    for (int i = n - 2; i >= 0; i--) {
        if (ratings[i] > ratings[i+1]) {
            candies[i] = max(candies[i], candies[i+1] + 1);
        }
    }

    return accumulate(candies.begin(), candies.end(), 0);
}
```

**TRACE for ratings=[1,0,2]:**
```
Pass 1 (left->right): candies=[1,1,1]
  i=1: ratings[1]=0 > ratings[0]=1? NO. candies[1]=1
  i=2: ratings[2]=2 > ratings[1]=0? YES. candies[2]=candies[1]+1=2
  candies=[1,1,2]

Pass 2 (right->left): 
  i=1: ratings[1]=0 > ratings[2]=2? NO.
  i=0: ratings[0]=1 > ratings[1]=0? YES. candies[0]=max(1, candies[1]+1)=max(1,2)=2
  candies=[2,1,2]

Sum = 5
Verify: child0(rating1,candy2) > child1(rating0,candy1) ✓ (1>0, 2>1)
        child2(rating2,candy2) > child1(rating0,candy1) ✓ (2>0, 2>1)
```

**TRACE for ratings=[1,2,2]:** note equal ratings need NO strict inequality.
```
Pass 1: i=1: 2>1 -> candies[1]=2. i=2: 2>2? NO -> candies[2]=1. candies=[1,2,1]
Pass 2: i=1: ratings[1]=2>ratings[2]=2? NO (equal, no constraint). i=0: 1>2? NO.
candies=[1,2,1], sum=4
```

---

## SECTION 9 — TEMPLATE 7: TASK SCHEDULER (LC 621, FREQUENCY GREEDY)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 621 — Task Scheduler
// Tasks (chars A-Z), each unit time. Same task must be separated by
// at least `k` units (idle or other tasks allowed in between).
// Find minimum total time to finish all tasks.
//
// GREEDY RULE / FORMULA:
//   maxFreq  = highest frequency among all task types
//   numMax   = how many task types share that highest frequency
//   answer   = max(n, (maxFreq - 1) * (k + 1) + numMax)
//   where n = total number of tasks.
//
// DERIVATION (the "brick wall" construction):
//   Take the most frequent task and lay out (maxFreq - 1) "blocks,"
//   each block being [the task] followed by (k) cooldown slots:
//     A _ _ ... _ A _ _ ... _ A ... (maxFreq - 1 copies of "A + k gaps")
//   This uses (maxFreq - 1) * (k + 1) slots, with maxFreq A's placed
//   at the FRONT of every block, and the gaps to be filled by other
//   tasks. Then place the FINAL occurrence of A (and any other tasks
//   that also hit maxFreq) at the very end: + numMax.
//   This skeleton has length (maxFreq-1)*(k+1) + numMax.
//   Fill every gap with the remaining tasks (round-robin across
//   distinct less-frequent task types so no two same tasks are
//   adjacent within k) — this is always possible because no other
//   task type has MORE occurrences than maxFreq, so it can never
//   "overflow" a single block and force two same-tasks adjacent.
//   If there are enough OTHER tasks to fill every idle gap and then
//   some, those extra tasks simply extend the schedule with no idle
//   time at all — in that case the answer is just n (no idle slots
//   needed, everything fits back-to-back). Hence answer = max(n, skeleton).
// ─────────────────────────────────────────────────────────────────

int leastInterval(vector<char>& tasks, int k) {
    array<int, 26> freq{};
    for (char t : tasks) freq[t - 'A']++;

    int maxFreq = *max_element(freq.begin(), freq.end());
    int numMax = count(freq.begin(), freq.end(), maxFreq);

    int skeleton = (maxFreq - 1) * (k + 1) + numMax;
    return max((int)tasks.size(), skeleton);
}
```

**TRACE for tasks=[A,A,A,B,B,B], k=2:**
```
freq: A=3, B=3. maxFreq=3, numMax=2 (both A and B hit 3)
skeleton = (3-1)*(2+1) + 2 = 2*3+2 = 8
n = 6
answer = max(6, 8) = 8

Construction: A B _ A B _ A B  (length 8)
              blocks: [A B _][A B _][A B]  -- 2 full blocks of size 3, then final AB
Check spacing: A at 0,3,6 -> gaps of 3 >= k+1=3 ✓. B at 1,4,7 -> gaps of 3 ✓.
```

**TRACE for tasks=[A,A,A,B,B,B,C,C,C,D,D,E,E], k=2 (many task types, idle never needed):**
```
freq: A=3,B=3,C=3,D=2,E=2. maxFreq=3, numMax=3
skeleton = (3-1)*3 + 3 = 6+3 = 9
n = 13
answer = max(13, 9) = 13  (plenty of other tasks fill every gap; no idle time)
```

**Alternative (priority-queue simulation)** — useful when you need the actual schedule, not just the count:
```cpp
int leastIntervalSimulated(vector<char>& tasks, int k) {
    array<int,26> freq{};
    for (char t : tasks) freq[t-'A']++;
    priority_queue<int> pq;
    for (int f : freq) if (f > 0) pq.push(f);

    int time = 0;
    queue<pair<int,int>> cooldown;  // {count, readyAtTime}

    while (!pq.empty() || !cooldown.empty()) {
        time++;
        if (!pq.empty()) {
            int cnt = pq.top(); pq.pop();
            if (--cnt > 0) cooldown.push({cnt, time + k});
        }
        if (!cooldown.empty() && cooldown.front().second == time) {
            pq.push(cooldown.front().first);
            cooldown.pop();
        }
    }
    return time;
}
```

---

## SECTION 10 — TEMPLATE 8: MINIMUM NUMBER OF ARROWS TO BURST BALLOONS (LC 452)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 452 — Minimum Number of Arrows to Burst Balloons
// Balloons given as [xstart, xend] intervals (real-valued, can
// overlap at boundaries). An arrow shot at position x bursts every
// balloon whose interval contains x. Find the minimum arrows to
// burst all balloons.
//
// This is "minimum points to stab all intervals" — the interval
// COVER dual of activity selection.
//
// GREEDY RULE: sort by END coordinate. Shoot an arrow at the end of
// the first (unburst) balloon. This arrow also bursts every balloon
// that overlaps with it (start <= arrowPos). Move to the next
// balloon whose start > current arrow position; repeat.
//
// PROOF (exchange argument): consider the balloon B with the
// smallest end coordinate. ANY valid solution must shoot at least
// one arrow within B's interval (to burst B). Placing that arrow
// EXACTLY at B's end coordinate is never worse than placing it
// anywhere else inside B, because B's end is the smallest possible
// end among all balloons overlapping it at that point — so an arrow
// there bursts the maximum possible set of OTHER balloons that also
// overlap B (any balloon overlapping B necessarily contains points
// up to at least B's end, by interval overlap definition — actually
// more precisely: any balloon overlapping B and considered "in
// range" has start <= B.end, so shooting at B.end bursts it too).
// So this exchange (move any arrow inside B to exactly B.end) only
// weakly increases the set of balloons burst by that arrow — never
// worse. Repeating this argument on the remaining un-burst balloons
// (recursively the same sub-problem) proves optimality by induction.
// ─────────────────────────────────────────────────────────────────

int findMinArrowShots(vector<vector<int>>& points) {
    if (points.empty()) return 0;

    sort(points.begin(), points.end(),
         [](const vector<int>& a, const vector<int>& b) {
             return a[1] < b[1];   // sort by end coordinate
         });

    int arrows = 1;
    long long arrowPos = points[0][1];   // long long: coords can be up to 2^31-1

    for (auto& p : points) {
        if (p[0] > arrowPos) {     // this balloon starts after current arrow
            arrows++;
            arrowPos = p[1];
        }
        // else: p[0] <= arrowPos <= p[1] (guaranteed since sorted by end,
        // and arrowPos was <= this balloon's own end at the time it was set)
        // so this balloon is already burst by the current arrow.
    }
    return arrows;
}
```

**Note the subtle difference from Activity Selection:** Activity Selection uses strict `start >= lastEnd` (touching endpoints are still compatible/non-overlapping since intervals there are usually treated as `[start, end)`). Minimum Arrows uses `p[0] > arrowPos` (touching endpoints DO count as overlapping — a balloon `[2,4]` and one starting at `4` both get burst by an arrow at `x=4`, since `4` is a legal integer coordinate inside both closed intervals). Always re-derive the boundary condition from the problem's exact interval semantics — this is the single most common off-by-one bug in interval greedy problems.

**TRACE for points=[[10,16],[2,8],[1,6],[7,12]]:**
```
sorted by end: [1,6] [2,8] [7,12] [10,16]

arrows=1, arrowPos=6
[2,8]: start=2 <= 6 -> already burst
[7,12]: start=7 > 6 -> arrows=2, arrowPos=12
[10,16]: start=10 <= 12 -> already burst

Answer: 2
```

**BONUS — Minimum Platforms (GFG classic, same family):** given arrival and departure times of trains at a station, find the minimum number of platforms needed so no train waits. This is "maximum number of intervals overlapping at any single point," which is the *opposite* framing from Minimum Arrows but solved with the same sort-and-sweep greedy:

```cpp
// Minimum Platforms (GFG)
// Sort arrivals and departures SEPARATELY. Two-pointer sweep:
// every new arrival that happens before the earliest still-active
// departure needs a NEW platform; every departure frees one.
int minPlatforms(vector<int> arr, vector<int> dep) {
    sort(arr.begin(), arr.end());
    sort(dep.begin(), dep.end());

    int platforms = 0, maxPlatforms = 0;
    int i = 0, j = 0, n = arr.size();

    while (i < n && j < n) {
        if (arr[i] <= dep[j]) {     // a new train arrives before another departs
            platforms++;
            maxPlatforms = max(maxPlatforms, platforms);
            i++;
        } else {                     // a train departs, freeing a platform
            platforms--;
            j++;
        }
    }
    return maxPlatforms;
}
// PROOF: at any instant, the number of platforms needed equals the
// number of trains simultaneously "in the station" (arrived but not
// departed). Sorting both arrays and sweeping in time order tracks
// exactly this count at every event, so the running maximum IS the
// answer — this is not even really a "choice-based" greedy, it's a
// direct simulation of the definition, which is why no exchange
// argument is needed: correctness is by definition, not by proof of
// non-obvious optimality.
```

---

## SECTION 11 — TEMPLATE 9: QUEUE RECONSTRUCTION BY HEIGHT (LC 406)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 406 — Queue Reconstruction by Height
// people[i] = [h, k] where h = height, k = number of people in
// front of this person who have height >= h. Reconstruct a valid
// queue order.
//
// GREEDY RULE: sort by height DESCENDING; for ties, by k ASCENDING.
// Then insert each person into the result list at index k.
//
// PROOF SKETCH: Process people from tallest to shortest. When
// inserting a person P=[h,k], every person ALREADY placed is >= h
// (we process tall-to-short). So inserting P at index k guarantees
// exactly k taller-or-equal people are in front of P at THIS point
// in construction — matching P's requirement.
// Crucially, inserting P does NOT break the k-values of the
// already-placed (taller) people: since P is SHORTER than everyone
// already placed, P is invisible to their "count of people with
// height >= mine" constraint (only heights >= theirs count, and P
// is shorter). Every future insertion (of even shorter people) will
// likewise not affect anyone taller. Hence each person's k
// constraint, once satisfied at insertion time, remains satisfied
// forever. By induction over the tallest-to-shortest processing
// order, the final arrangement is valid for everyone.
// ─────────────────────────────────────────────────────────────────

vector<vector<int>> reconstructQueue(vector<vector<int>>& people) {
    sort(people.begin(), people.end(),
         [](const vector<int>& a, const vector<int>& b) {
             if (a[0] != b[0]) return a[0] > b[0];  // taller first
             return a[1] < b[1];                     // then smaller k first
         });

    vector<vector<int>> result;
    for (auto& p : people) {
        result.insert(result.begin() + p[1], p);    // insert at index k
    }
    return result;
}
```

**TRACE for people=[[7,0],[4,4],[7,1],[5,0],[6,1],[5,2]]:**
```
Sorted (height desc, then k asc):
[7,0] [7,1] [6,1] [5,0] [5,2] [4,4]

Insert [7,0] at index 0: [[7,0]]
Insert [7,1] at index 1: [[7,0],[7,1]]
Insert [6,1] at index 1: [[7,0],[6,1],[7,1]]
Insert [5,0] at index 0: [[5,0],[7,0],[6,1],[7,1]]
Insert [5,2] at index 2: [[5,0],[7,0],[5,2],[6,1],[7,1]]
Insert [4,4] at index 4: [[5,0],[7,0],[5,2],[6,1],[4,4],[7,1]]

Answer: [[5,0],[7,0],[5,2],[6,1],[4,4],[7,1]]
```

**Complexity note:** `vector::insert` at an arbitrary index is O(n), so total time is O(n²) with a vector. Using a `list` avoids the shifting cost but loses O(1) random access needed for indexed insertion, so a Fenwick-tree / order-statistics-tree approach is required to get O(n log n) — worth mentioning as a follow-up in interviews, but the O(n²) vector solution is standard and accepted.

---

## SECTION 12 — TEMPLATE 10: PARTITION LABELS (LC 763)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 763 — Partition Labels
// Partition a string into as many parts as possible so that each
// letter appears in AT MOST one part. Return the sizes of the parts.
//
// GREEDY RULE: precompute the LAST occurrence index of every char.
// Scan left to right, extending the current partition's end to
// max(end, lastOccurrence[current char]). When the scan index
// reaches `end`, the partition is complete — cut here.
//
// PROOF: A partition boundary at index i is valid iff no character
// appearing in [start, i] also appears again after i. Equivalently,
// end must be >= the last occurrence of every character seen so
// far in the current partition. The greedy maintains this exactly:
// `end` is always the max last-occurrence among characters seen in
// [start, current]. The EARLIEST valid cut point is exactly when
// the scan pointer catches up to this running max — cutting any
// earlier would split a character's occurrences across two
// partitions (invalid); cutting here (as soon as possible) 
// maximizes the NUMBER of partitions, since any larger partition
// containing this valid range as a proper subset is unnecessary —
// by an exchange argument, replacing a larger partition with the
// minimal valid one is always at least as good (frees the
// remainder to be partitioned further if possible, never worse).
// ─────────────────────────────────────────────────────────────────

vector<int> partitionLabels(string s) {
    array<int, 26> lastOcc{};
    for (int i = 0; i < s.size(); i++) lastOcc[s[i]-'a'] = i;

    vector<int> result;
    int start = 0, end = 0;

    for (int i = 0; i < s.size(); i++) {
        end = max(end, lastOcc[s[i]-'a']);
        if (i == end) {                 // reached the boundary
            result.push_back(end - start + 1);
            start = i + 1;
        }
    }
    return result;
}
```

**TRACE for s="abac" (indices 0-3):**
```
lastOcc: a=2, b=1, c=3

i=0 'a': end=max(0,lastOcc[a]=2)=2. i(0)!=end(2)
i=1 'b': end=max(2,lastOcc[b]=1)=2. i(1)!=end(2)
i=2 'a': end=max(2,lastOcc[a]=2)=2. i==end(2) -> size=2-0+1=3. start=3
i=3 'c': end=max(2,lastOcc[c]=3)=3. i==end(3) -> size=3-3+1=1. start=4

Answer: [3, 1]
Check: partition0="aba" (a at 0,2 both inside; b at 1 inside; no leak),
       partition1="c". ✓
```

---

## SECTION 13 — TEMPLATE 11: EARLIEST POSSIBLE DAY OF FULL BLOOM (LC 2136)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 2136 — Earliest Possible Day of Full Bloom
// n seeds. plantTime[i] = days needed to plant seed i (plant seeds
// one at a time, sequentially — only one seed can be planted per
// day, but planting order is your choice). growTime[i] = days after
// FINISHING planting seed i for it to bloom. Find the earliest day
// on which ALL seeds have bloomed, given optimal planting order.
//
// GREEDY RULE: plant seeds in DESCENDING order of growTime.
//
// PROOF (adjacent-swap exchange argument): Let `start` be the day
// planting begins for a block of two adjacent seeds A, B, with
// pA=plantTime[A], pB=plantTime[B], gA=growTime[A], gB=growTime[B].
// Regardless of order, whichever seed is planted SECOND finishes
// planting at start+pA+pB (planting time is additive either way).
//
// Order "A then B" (out-of-greedy-order case, gA < gB):
//   bloom(A) = start+pA+gA        (A planted first, finishes early)
//   bloom(B) = start+pA+pB+gB     (B planted second, finishes late)
//   local max = start+pA+pB+gB    (dominated by B, which has the
//                                   LARGER growTime AND finishes late)
//
// Order "B then A" (greedy order, since gB > gA):
//   bloom(B) = start+pB+gB        (B planted first, finishes early)
//   bloom(A) = start+pA+pB+gA     (A planted second, finishes late)
//   local max = max(start+pB+gB, start+pA+pB+gA)
//             <= start+pA+pB+gB   (since gA < gB, both terms are
//                                   bounded by the original local max)
//
// So swapping to put the larger-growTime seed first never increases
// the local max, hence never increases the overall makespan. Any
// adjacent out-of-order pair can be swapped this way with no loss;
// repeating until fully sorted descending by growTime reaches the
// greedy order without ever worsening the objective — the standard
// adjacent-swap argument (same shape as minimizing lateness in job
// sequencing).
// ─────────────────────────────────────────────────────────────────

int earliestFullBloom(vector<int>& plantTime, vector<int>& growTime) {
    int n = plantTime.size();
    vector<int> idx(n);
    iota(idx.begin(), idx.end(), 0);

    // Sort indices by DESCENDING growTime
    sort(idx.begin(), idx.end(), [&](int a, int b) {
        return growTime[a] > growTime[b];
    });

    long long day = 0, maxBloom = 0;
    for (int i : idx) {
        day += plantTime[i];                     // finish planting this seed
        maxBloom = max(maxBloom, day + growTime[i]);
    }
    return (int)maxBloom;
}
```

**TRACE for plantTime=[1,4,3], growTime=[2,3,1]:**
```
Sort by growTime desc: seed1(grow=3) seed0(grow=2) seed2(grow=1)

day=0
plant seed1: day=0+4=4. bloom=4+3=7. maxBloom=7
plant seed0: day=4+1=5. bloom=5+2=7. maxBloom=7
plant seed2: day=5+3=8. bloom=8+1=9. maxBloom=9

Answer: 9
```

**Why descending growTime and not descending plantTime or ascending anything?** The seed with the LONGEST growTime benefits most from being planted early (its clock starts ticking sooner, and it has the most "runway" to absorb). Seeds with short growTime don't mind waiting — their bloom day is dominated by whenever they finish planting, not by growTime, so pushing them later costs almost nothing extra relative to their own small growTime contribution.

---

## SECTION 14 — TEMPLATE 12: MINIMUM REPLACEMENTS TO SORT ARRAY (LC 2366)

```cpp
// ─────────────────────────────────────────────────────────────────
// LC 2366 — Minimum Replacements to Sort Array
// You may replace any element with two or more elements that sum to
// the original value. Find the minimum number of replacement
// OPERATIONS to make the array non-decreasing.
//
// GREEDY RULE (reverse greedy — process RIGHT TO LEFT):
//   Maintain `maxAllowed` = the largest value the CURRENT element's
//   pieces are allowed to end with (initially nums.back(), since the
//   last element has no right neighbor constraint).
//   For nums[i], we must split it into k pieces, each <= maxAllowed,
//   using the MINIMUM number of pieces: k = ceil(nums[i] / maxAllowed).
//   That costs (k - 1) replacement operations (k pieces from 1
//   original = k-1 splits).
//   To not over-constrain elements further to the left, make the
//   pieces as EQUAL as possible: pieceSize = nums[i] / k (integer
//   division). The new maxAllowed for the next element (to the left)
//   becomes this pieceSize — the leftmost/smallest of the k roughly
//   equal pieces, since pieces are placed left-to-right in ascending
//   order and the next element only needs to be <= the smallest one.
//
// PROOF: fewer splits means fewer operations, so k must be the
// minimum piece count satisfying "each piece <= maxAllowed," which
// is ceil(nums[i]/maxAllowed) — using more pieces than this can
// always be merged back down, strictly reducing operations without
// violating anything (an exchange argument). Given a fixed minimal
// k, distributing nums[i] as EQUALLY as possible MAXIMIZES the
// smallest piece (pigeonhole: unequal pieces force the minimum below
// the average nums[i]/k). A larger minimum piece gives the next
// element to the left more room (a higher maxAllowed), which can
// only help reduce ITS split count. So "minimal pieces, as equal as
// possible" is optimal at every step, and this local-to-global
// monotonic argument makes the right-to-left greedy scan globally
// optimal.
// ─────────────────────────────────────────────────────────────────

long long minimumReplacement(vector<int>& nums) {
    int n = nums.size();
    long long operations = 0;
    long long maxAllowed = nums[n-1];   // last element has no right constraint

    for (int i = n - 2; i >= 0; i--) {
        if (nums[i] <= maxAllowed) {
            maxAllowed = nums[i];        // no split needed; this becomes the new ceiling
            continue;
        }
        long long k = (nums[i] + maxAllowed - 1) / maxAllowed;  // ceil division
        operations += (k - 1);
        maxAllowed = nums[i] / k;        // smallest of the k equal-ish pieces
    }
    return operations;
}
```

**TRACE for nums=[3,9,3]:**
```
n=3
maxAllowed = nums[2] = 3

i=1: nums[1]=9 > maxAllowed=3.
     k = ceil(9/3) = 3. operations += (3-1) = 2. operations=2
     maxAllowed = 9/3 = 3
     (9 splits into [3,3,3] -- 2 operations)

i=0: nums[0]=3 <= maxAllowed=3. maxAllowed=3. no operation.

Answer: 2 operations. Final array conceptually: [3,3,3,3,3] -- non-decreasing. ✓
```

**TRACE for nums=[1,2,3,4,5] (already sorted — should need 0 operations):**
```
maxAllowed=5
i=3: 4<=5 -> maxAllowed=4
i=2: 3<=4 -> maxAllowed=3
i=1: 2<=3 -> maxAllowed=2
i=0: 1<=2 -> maxAllowed=1
operations = 0 ✓
```

---

## SECTION 15 — WHEN GREEDY FAILS: A CATALOG

Every entry below is a problem that LOOKS like it should have a greedy solution (sort by some key, pick greedily) but provably does not — a DP formulation is required because a locally-optimal choice can be globally suboptimal in a way that cannot be repaired.

### 15.1 — 0/1 Knapsack

```cpp
// FAILS: greedy by value/weight ratio
// capacity=10. items: (w=6,v=10), (w=5,v=8), (w=5,v=8).
// ratios: 1.67, 1.6, 1.6. Greedy takes item1 (w=6,v=10) first (highest
// ratio); remaining capacity=4 -- can't fit either remaining w=5 item.
// Total = 10.
// Optimal: skip item1, take BOTH w=5 items: weight=10 (exactly fits),
// value=16.
// Greedy (10) < Optimal (16). FAILS.
//
// WHY: once item1 is committed, its 6 units of capacity are LOCKED
// IN even though a different combination uses the capacity better.
// There is no way to "give back" part of item1 — the 0/1 constraint
// makes the choice irrevocable in a way fractional knapsack's
// continuous divisibility does not.
//
// FIX: 0/1 Knapsack DP — dp[i][w] = max value using first i items
// with capacity w. Explores BOTH "take" and "skip" for every item,
// which greedy's single irrevocable choice cannot do.
```

### 15.2 — Coin Change with Arbitrary Denominations

```cpp
// FAILS: greedy "always take the largest coin <= remaining amount"
// coins = {1, 3, 4}, target = 6.
// Greedy: take 4 (remaining=2), take 1 (remaining=1), take 1
// (remaining=0). Total: 3 coins (4+1+1).
// Optimal: 3+3 = 2 coins.
// Greedy (3 coins) is suboptimal.
//
// WHY: greedy assumes "biggest coin first" always leads toward the
// minimum count, but this only holds for CANONICAL coin systems
// (like standard currency: 1,5,10,25,...) where a mathematical
// property (every denomination is a "nice" multiple-ish of smaller
// ones) guarantees greedy optimality. Without that structural
// guarantee, greedy can get stuck in a locally-good-looking but
// globally bad configuration.
//
// FIX: Coin Change DP — dp[amount] = min coins to make `amount`,
// built up from dp[amount - coin] for every coin, exploring ALL
// combinations rather than committing to the largest coin first.
```

### 15.3 — Longest Increasing Subsequence (naive greedy)

A greedy "always extend if next element is bigger than the last chosen" fails: `[3, 4, 5, 1, 2]` — greedy picks `3,4,5` (length 3) and gets stuck, but if it had chosen differently it still can't beat length 3 here — but consider `[10, 1, 2, 3]`: naive greedy starting from index 0 picks just `[10]` (nothing after is bigger), length 1, while the true LIS is `[1,2,3]`, length 3. The correct approach (patience sorting / DP with binary search) maintains multiple candidate "tails" rather than committing to one sequence — this is greedy *combined with* an auxiliary DP-like structure, not pure greedy.

### 15.4 — The general lesson

Greedy fails whenever the problem has **overlapping subproblems where the best local choice can block a better global combination that only reveals itself later**, and no exchange argument can be constructed because counterexamples like the ones above exist. The tell: if you can construct *any* input where the greedy choice provably loses to an alternative, greedy is wrong — full stop, no matter how many test cases it happens to pass.

---

## SECTION 16 — COMPLEXITY TABLE

| Problem | Time | Space | Proof Technique | Notes |
|---------|------|-------|-----------------|-------|
| Activity Selection | O(n log n) | O(1) extra | Exchange / Stays-ahead | Sort dominates |
| Assign Cookies (455) | O(n log n) | O(1) extra | Exchange | Two-pointer after sort |
| Jump Game (55) | O(n) | O(1) | Stays-ahead | No sort needed |
| Jump Game II (45) | O(n) | O(1) | Stays-ahead (BFS layers) | No sort needed |
| Gas Station (134) | O(n) | O(1) | Stays-ahead / prefix-min | Single pass, no sort |
| Candy (135) | O(n) | O(n) | Two independent exchange proofs | Two passes |
| Task Scheduler (621) | O(n) | O(1) — 26 buckets | Combinatorial (frequency formula) | Bucket derivation |
| Minimum Arrows (452) | O(n log n) | O(1) extra | Exchange | Sort by end |
| Minimum Platforms (GFG) | O(n log n) | O(1) extra | Direct simulation | Sort arr & dep separately |
| Queue Reconstruction (406) | O(n²) | O(n) | Exchange (insertion invariant) | O(n log n) with BIT |
| Partition Labels (763) | O(n) | O(1) — 26 buckets | Exchange (minimal valid cut) | Single pass |
| Full Bloom (2136) | O(n log n) | O(n) | Adjacent-swap exchange | Sort by growTime desc |
| Min Replacements (2366) | O(n) | O(1) | Local-to-global monotonic | Right-to-left scan |

**Pattern:** almost every entry is `O(n log n)` dominated by a single sort, followed by an `O(n)` linear scan. The handful that are pure `O(n)` (Jump Game, Gas Station, Task Scheduler, Partition Labels, Min Replacements) don't need sorting because the greedy criterion is derivable directly from a running scan invariant rather than requiring a global reordering first.

---

## SECTION 17 — VARIANTS

- **Non-overlapping Intervals (LC 435):** "Minimum removals to make intervals non-overlapping" = `total - activitySelection()`. Same sort-by-end greedy, framed as a complement.
- **Merge Intervals (LC 56):** Sort by start, merge overlapping — a different (but related) interval greedy, not a selection problem.
- **Job Sequencing with Deadlines:** each job has a deadline and profit, each job takes 1 unit of time, maximize profit scheduling jobs to not miss deadlines. Greedy: sort by profit descending, place each job in the latest available free slot at or before its deadline (using a DSU or simple backward scan) — classic exchange-argument greedy.
- **Minimum Number of Refueling Stops (LC 871):** greedy with a max-heap — pass every reachable station, and whenever you'd run out of fuel, pop the largest fuel amount seen so far. A "delayed greedy" variant worth recognizing.
- **Boats to Save People (LC 881):** sort weights, two-pointer, pair heaviest with lightest if they fit — exchange argument nearly identical to Assign Cookies.
- **Two City Scheduling (LC 1029):** sort by `costA - costB`, send the first half to city A — exchange argument on the DIFFERENCE, a generalization of the simple sort-and-pick pattern.
- **Reorganize String (LC 767) / Task Scheduler generalization:** frequency-based greedy with a max-heap instead of a closed-form formula, for when cooldown isn't uniform.

---

## SECTION 18 — COMMON MISTAKES

### Mistake 1: Assuming greedy without proof

```cpp
// WRONG mindset: "sort by X, take greedily, submit."
// This is how contestants get WA on test 47.

// CORRECT mindset: before coding, either sketch an exchange argument
// OR find a stays-ahead invariant. If you cannot articulate WHY the
// locally best choice cannot be beaten later, you don't have a
// greedy solution yet — you have a guess.
```

### Mistake 2: Sorting by the wrong key

```cpp
// WRONG — Activity Selection sorted by START time
sort(activities.begin(), activities.end());  // sorts by start (pair default)
// A long early-starting activity blocks everything, even though it
// might finish very late. Greedy-by-start is provably NOT optimal
// (counterexample: [(0,100),(1,2),(2,3),(3,4)] -- sorting by start
// picks (0,100) first, count=1; sorting by end picks (1,2),(2,3),
// (3,4), count=3).

// CORRECT — sort by END time (see Section 3's exchange argument).
sort(activities.begin(), activities.end(),
     [](auto& a, auto& b){ return a.second < b.second; });
```

### Mistake 3: Gas Station — forgetting the `total >= 0` feasibility check

```cpp
// WRONG — only running the reset-scan without checking total
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int tank = 0, start = 0;
    for (int i = 0; i < gas.size(); i++) {
        tank += gas[i] - cost[i];
        if (tank < 0) { start = i + 1; tank = 0; }
    }
    return start;   // BUG: always returns SOME index, even if no
                     // solution exists (total < 0)!
}

// CORRECT — must separately verify total >= 0
// (see Section 7's full implementation with the `total` accumulator)
```

### Mistake 4: Task Scheduler — off-by-one in the skeleton formula

```cpp
// WRONG — forgetting the "-1" on maxFreq (over-counts trailing gap)
int skeleton = maxFreq * (k + 1) + numMax;
// This adds one extra full block of cooldown after the LAST
// occurrence of the most frequent task, which is never needed —
// there's nothing after the last block that requires cooldown padding.

// CORRECT
int skeleton = (maxFreq - 1) * (k + 1) + numMax;
```

### Mistake 5: Candy — running only ONE pass

```cpp
// WRONG — only left-to-right pass
for (int i = 1; i < n; i++)
    if (ratings[i] > ratings[i-1]) candies[i] = candies[i-1] + 1;
// This satisfies "higher rating than LEFT neighbor gets more candy"
// but completely ignores the RIGHT neighbor constraint. Descending
// runs (ratings dropping left to right) are never fixed up.
// Example: ratings=[3,2,1]. Pass 1 alone never triggers (nothing is
// increasing left to right), so candies=[1,1,1]. But every adjacent
// pair is DESCENDING, requiring candy[0]>candy[1]>candy[2] — the
// correct answer is [3,2,1]. Pass 1 alone fails completely; the
// right-neighbor constraint needs its own pass.

// CORRECT — both passes, then max() per index (see Section 8).
```

### Mistake 6: Minimum Arrows — reusing Activity Selection's `>=` boundary

```cpp
// WRONG — treating touching balloons as non-overlapping like Activity Selection
if (p[0] >= arrowPos) { arrows++; arrowPos = p[1]; }
// This UNDER-counts arrows: a balloon [1,4] and a balloon [4,8] DO
// share the point x=4 and can be burst by ONE arrow at x=4, but this
// condition treats them as needing separate arrows.

// CORRECT — strict '>' since touching endpoints ARE burst together
if (p[0] > arrowPos) { arrows++; arrowPos = p[1]; }
```

---

## SECTION 19 — PROBLEM SET

### WARMUP (solve in ≤ 10 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 1 | Assign Cookies | 455 | Two-pointer greedy after sorting both arrays |
| 2 | Jump Game | 55 | Farthest-reachable invariant, stays-ahead proof |

### CORE (solve in ≤ 25 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 3 | Gas Station | 134 | Circular greedy, prefix-sum-min proof |
| 4 | Candy | 135 | Two-pass greedy, combining two lower bounds |
| 5 | Task Scheduler | 621 | Frequency-count formula derivation |
| 6 | Minimum Number of Arrows | 452 | Interval-cover greedy, boundary semantics |

### STRETCH (solve in ≤ 40 minutes each)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 7 | Queue Reconstruction by Height | 406 | Sort + positional insertion invariant |
| 8 | Partition Labels | 763 | Last-occurrence window expansion |
| 9 | Minimum Platforms | GFG | Two-pointer sweep over separate sorted arrays |

### CONTEST (attempt under time pressure)

| # | Problem | LC # | Key Skill |
|---|---------|------|-----------|
| 10 | Earliest Possible Day of Full Bloom | 2136 | Adjacent-swap exchange, sort by growTime desc |
| 11 | Minimum Replacements to Sort Array | 2366 | Reverse greedy, ceil-division + equal-split proof |

---

## SECTION 20 — PATTERN CONNECTIONS

1. **Pattern 11 (Intervals / Merge Intervals):** Activity Selection, Minimum Arrows, and Minimum Platforms are all interval problems solved by sorting on an endpoint. Pattern 11 covers the interval-manipulation toolkit (merge, insert, overlap-count) that these greedy proofs build directly on top of.

2. **Pattern 10 (Heaps / Priority Queues):** Task Scheduler's simulation variant and Minimum Refueling Stops both use a heap to defer a choice until it's forced — "greedy with a heap" is a hybrid where the immediate choice isn't fully determined by sorting alone but by a dynamically maintained priority structure. Recognize this whenever a simple sort-once greedy isn't expressive enough because the "best available option" changes as you go.

3. **Pattern 16 (DP fundamentals) — the contrast pattern:** Section 15 of this document exists specifically to draw the line between Pattern 34 and Pattern 16/19 (Knapsack). The diagnostic: if you cannot construct a valid exchange argument or stays-ahead invariant — and especially if you CAN construct a counterexample — the problem needs DP's exhaustive-but-memoized exploration instead of greedy's single irrevocable pass.

4. **Pattern 04 (Monotonic Stack) / Pattern 33 (Sorting-based techniques):** Many greedy solutions are "sort, then single monotonic scan" — the same shape as monotonic stack problems, except the stack there tracks a *feasibility* invariant while greedy typically tracks an *accumulator* (tank, farthest, maxAllowed). Recognizing the "sort + single pass with one running variable" shape is often the fastest way to spot that a problem is greedy at all.

---

## SECTION 21 — INTERVIEW SIMULATIONS

### Interview Simulation 1: LC 134 — Gas Station

**Interviewer:** "There are n gas stations in a circle. Find a starting station that lets you complete the circuit, or return -1."

**Candidate:**

*[First, state the candidate strategy]*
> "My instinct is a single greedy pass: track a running tank using `gas[i]-cost[i]`. Whenever the tank goes negative, none of the stations I've tried since my last reset can be a valid start — reset the candidate start to the next station."

*[Interviewer]: "Why does that work? Prove it."*
> "Two parts. First, a solution exists at all iff `sum(gas) >= sum(cost)` — if total fuel is less than total cost, no starting point can complete the loop, since the deficit is fixed regardless of rotation. Second, if total is non-negative, I claim the station right after the point of maximum cumulative deficit is always a valid start. Formally, let `P[i]` be the prefix sum of `gas[j]-cost[j]` for `j<i`. If `s` minimizes `P[s]`, then for any station after `s`, the cumulative sum from `s` never dips below zero — because either you're still within the prefix (bounded below by `P[s]` being the min) or you've wrapped around, in which case the wrapped total is `total - P[s] + P[j+1]`, and since `P[s] <= P[j+1]` for all `j+1 <= s`, this stays non-negative too."

*[Code — 5 minutes]*
```cpp
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    long long total = 0, tank = 0;
    int start = 0;
    for (int i = 0; i < gas.size(); i++) {
        int diff = gas[i] - cost[i];
        total += diff; tank += diff;
        if (tank < 0) { start = i + 1; tank = 0; }
    }
    return total >= 0 ? start : -1;
}
```

*[Interviewer]: "What if there are multiple valid starting stations?"*
> "The problem guarantees uniqueness when a solution exists — but even without that guarantee, this algorithm finds A valid one (the one right after the global cumulative minimum). Proving uniqueness would require showing any two valid starts must be separated by a zero-sum stretch, which isn't needed for correctness here."

**RED FLAGS:**
- Not checking `total >= 0` before returning `start` (returns garbage answer for infeasible inputs)
- Using `int` instead of `long long` for `tank`/`total` when constraints allow large sums (overflow)
- Claiming the algorithm is "obviously correct" without being able to state the prefix-sum argument

---

### Interview Simulation 2: LC 621 — Task Scheduler

**Interviewer:** "Given tasks and a cooldown `k` between identical tasks, find the minimum total time."

**Candidate:**

*[State the key structural insight before coding]*
> "The bottleneck is whichever task has the highest frequency — call it `maxFreq`. It forces a skeleton of `(maxFreq-1)` blocks, each of size `k+1` (the task itself plus `k` cooldown slots), plus however many OTHER tasks are tied for that same max frequency, which get appended at the end since they can't fill their own cooldown gaps. That gives a skeleton length of `(maxFreq-1)*(k+1) + numMax`."

*[Interviewer]: "Why does this formula give the true minimum, not just a valid schedule?"*
> "Two directions. First, it's a valid LOWER BOUND: you cannot schedule the most frequent task faster than one occurrence every `k+1` slots, so `(maxFreq-1)*(k+1)+numMax` slots are structurally required just to space out the max-frequency tasks — this length can't be beaten. Second, it's ACHIEVABLE: every gap in the skeleton can be filled by some other task, because no other task type has a HIGHER frequency than maxFreq, so no other task type could possibly need to appear twice within the same cooldown block — there's always a distinct-enough task to slot into every gap. If there are more tasks than gaps, they append with zero idle time, which is why the final answer is `max(n, skeleton)` — if leftover tasks overflow the skeleton, there's no need for idle time at all."

*[Code]*
```cpp
int leastInterval(vector<char>& tasks, int k) {
    array<int,26> freq{};
    for (char t : tasks) freq[t-'A']++;
    int maxFreq = *max_element(freq.begin(), freq.end());
    int numMax = count(freq.begin(), freq.end(), maxFreq);
    return max((int)tasks.size(), (maxFreq-1)*(k+1) + numMax);
}
```

**RED FLAGS:**
- Jumping straight to a heap simulation without first deriving the closed-form (both are acceptable, but the closed-form shows deeper understanding and is O(n) vs O(n log 26))
- Getting the `-1` on `maxFreq` wrong (off-by-one, see Common Mistake 4)
- Not being able to explain why "no gap is ever unfillable" — this is the crux of the achievability half of the proof

---

### Interview Simulation 3: Meta-discussion — "Prove your greedy is correct"

**Interviewer:** "You've proposed sorting by end time for an interval problem. How do I know this is actually optimal and not just a heuristic that happens to work on your examples?"

**Candidate:**
> "I'd give an exchange argument. Assume an optimal solution `OPT` disagrees with my greedy choice at the first activity. My greedy picks the interval with the globally smallest end time among all valid options, so its end time is `<=` whatever `OPT` picked first. I can swap `OPT`'s first pick for greedy's — this stays feasible, because everything else in `OPT` was already compatible with a LATER end time, so it's automatically compatible with my earlier-ending choice too. This swap doesn't change the size of the solution, so `OPT` is still optimal after the swap, and now it agrees with greedy on one more element. Repeating this argument inductively down the rest of the sequence shows greedy is optimal.

> Alternatively, I could argue 'greedy stays ahead': show by induction that after `k` picks, greedy's `k`-th interval's end time is never later than any valid solution's `k`-th interval's end time — meaning greedy always has at least as much room left as any competitor at every step, so it can never finish with fewer total intervals."

**RED FLAGS:**
- "It's obviously the right sort key" with no argument
- Confusing "passes the given examples" with "is proven correct"
- Not being able to produce a counterexample when challenged on a DIFFERENT proposed sort key (e.g., "why not sort by duration instead?") — a candidate with real command of the proof technique should be able to construct `[(0,100),(1,2),(2,3)]` on the spot to show sort-by-start fails, or a similar quick counterexample for sort-by-duration.

---

## SECTION 22 — MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: What are the two standard proof techniques for greedy correctness, and when would you prefer one over the other?**

> (a) Exchange argument: assume an optimal solution differs from greedy at the first point of disagreement, show swapping in greedy's choice never worsens the solution, induct. Best for problems where the output is a SET or SEQUENCE of discrete choices (Activity Selection, Assign Cookies, Minimum Arrows).
> (b) Greedy stays ahead: define a per-step measurable quantity, prove by induction that greedy's value at every step dominates any competitor's value at that step. Best for problems with a running ACCUMULATOR that must stay within bounds (Jump Game's farthest reachable, Gas Station's tank, Full Bloom's bloom day).

---

**Q2: In Gas Station, why does resetting `start = i+1` the moment `tank < 0` at station `i` never skip over a valid starting station?**

> Suppose the true valid start `k` were inside the skipped range `[start, i]`. Since `tank >= 0` at every station strictly before `i` (by definition of `i` being the FIRST station where tank went negative since the last reset), the cumulative sum from `start` to `k` is `>= 0`. So the cumulative sum from `k` to `i` equals (cumulative from `start` to `i`, which is negative) minus (cumulative from `start` to `k`, which is `>=0`) — meaning cumulative from `k` to `i` is even MORE negative. So `k` cannot be a valid start either; it would also run out of gas by station `i`. Hence every station in `[start, i]` is disqualified, justifying the jump to `i+1`.

---

**Q3: Why does 0/1 Knapsack break the exchange argument that makes Fractional Knapsack's greedy work?**

> Fractional Knapsack's greedy (sort by value/weight ratio, take greedily, take a FRACTION of the last item if needed) has a clean exchange argument: any solution using less than the full capacity, or using a lower-ratio item while a higher-ratio item's capacity is unfilled, can be improved by shifting weight from the lower-ratio item to the higher-ratio one — this shift is always possible because fractions are continuously divisible. In 0/1 Knapsack, items are indivisible — once you've committed to including item A whole, you cannot "shift out" part of it to make room for a better combination discovered later. The continuous shift the exchange argument relies on doesn't exist in the discrete/indivisible setting, so the proof — and the greedy algorithm itself — breaks down.

---

**Q4: In Task Scheduler, why is the answer `max(n, skeleton)` rather than just `skeleton`?**

> The skeleton formula `(maxFreq-1)*(k+1) + numMax` computes the minimum time assuming there might not be enough OTHER tasks to fill every cooldown gap (requiring idle slots). But if there are MANY other distinct task types with enough total occurrences, they can fill every gap in the skeleton and then some — in that case there's no idle time at all, and the answer is simply the total task count `n` (everything scheduled back-to-back with zero gaps). Taking `max(n, skeleton)` correctly picks whichever bound is binding: the frequency-driven lower bound, or the simple task-count lower bound.

---

**Q5: In Candy, why does taking `max()` of the two pass results at each index correctly satisfy BOTH neighbor constraints simultaneously, rather than just approximately?**

> Pass 1 computes, for each index, the minimum candy count needed to satisfy ONLY the "greater than left neighbor if rating is higher" constraint, assuming no right-neighbor constraint existed. Pass 2 computes the same for the right-neighbor constraint in isolation. Since `max(a,b) >= a` and `max(a,b) >= b`, taking the max at each index guarantees the value is AT LEAST what each individual pass required — so both constraints, which are each "candy[i] must be greater than some specific neighbor's candy count," remain satisfied. It's minimal because using anything less than `max(a,b)` at index `i` would violate whichever of the two passes required the larger value.

---

**Q6: Why must Minimum Number of Arrows sort by END coordinate and use strict `>` (not `>=`) when advancing to a new arrow, while Activity Selection uses `>=` for compatibility?**

> The two problems have different overlap semantics. Activity Selection intervals are typically treated as `[start, end)` (half-open) or as "compatible if one starts exactly when the other ends" — touching endpoints don't overlap, hence `start >= lastEnd` is the compatibility check. Minimum Arrows treats intervals as closed `[start, end]` where a shared boundary point IS a legal shooting position that bursts both balloons — so a balloon starting exactly at the current arrow's position (`p[0] == arrowPos`) is still burst by that arrow, and only `p[0] > arrowPos` requires a new arrow. Always derive the boundary condition from the problem's literal interval semantics rather than copying a template blindly.

---

**Q7: Give an example (not from the problem set) where a "sort and pick greedily" approach LOOKS plausible but is actually wrong, and explain how you'd detect this before submitting.**

> Example: "maximize the number of non-overlapping intervals by sorting by interval LENGTH (duration) ascending, picking greedily." This looks plausible — shorter intervals seem to "use less of the timeline." Counterexample: `[(0,2),(1,3),(2,4)]`. Lengths are all 2, so a length-based sort is ambiguous, but suppose ties break by start time, giving order `(0,2),(1,3),(2,4)`. Greedy takes `(0,2)`, then `(1,3)` overlaps (skip), then `(2,4)` is compatible (take) — count 2. That happens to match the true optimum here, which is exactly the danger: it takes a carefully constructed case to expose the flaw, e.g. `[(0,10),(0,1),(1,2),...,(8,9)]` (one length-10 interval plus nine length-1 intervals tiling `[0,10)`) — sort-by-length picks all nine length-1 intervals correctly by luck, so length alone isn't even the cleanest counterexample generator. The real lesson is the DETECTION METHOD, not any one counterexample: for ANY proposed sort key, try to construct the exchange argument directly. If, at the step where you'd swap greedy's choice into `OPT`, you cannot show the swap preserves feasibility AND doesn't worsen the objective, that is your signal the sort key is wrong — stop and re-derive rather than trusting a heuristic that merely "looks reasonable." (For interval SELECTION specifically, sort-by-end is the unique key with a valid exchange proof, as shown in Section 3; sort-by-start provably fails via `[(0,100),(1,2),(2,3),(3,4)]`, Common Mistake 2.)

---

## SECTION 23 — C++ CHEAT SHEET

```cpp
// ─────────────────────────────────────────────────────────────────
// 1. ACTIVITY SELECTION / INTERVAL SCHEDULING
// ─────────────────────────────────────────────────────────────────
sort(iv.begin(), iv.end(), [](auto&a,auto&b){ return a.second<b.second; });
int count=0, lastEnd=INT_MIN;
for (auto&[s,e]:iv) if (s>=lastEnd) { count++; lastEnd=e; }

// ─────────────────────────────────────────────────────────────────
// 2. ASSIGN COOKIES (two-pointer after double sort)
// ─────────────────────────────────────────────────────────────────
sort(g.begin(),g.end()); sort(s.begin(),s.end());
int i=0,j=0,sat=0;
while (i<g.size() && j<s.size()) { if (s[j]>=g[i]) { sat++; i++; } j++; }

// ─────────────────────────────────────────────────────────────────
// 3. JUMP GAME (farthest reachable)
// ─────────────────────────────────────────────────────────────────
int farthest=0;
for (int i=0;i<n;i++) {
    if (i>farthest) return false;
    farthest=max(farthest, i+nums[i]);
}

// ─────────────────────────────────────────────────────────────────
// 4. JUMP GAME II (BFS-layer greedy)
// ─────────────────────────────────────────────────────────────────
int jumps=0,curEnd=0,farthest=0;
for (int i=0;i<n-1;i++) {
    farthest=max(farthest,i+nums[i]);
    if (i==curEnd) { jumps++; curEnd=farthest; }
}

// ─────────────────────────────────────────────────────────────────
// 5. GAS STATION (circular reset greedy)
// ─────────────────────────────────────────────────────────────────
long long total=0,tank=0; int start=0;
for (int i=0;i<n;i++) {
    int d=gas[i]-cost[i]; total+=d; tank+=d;
    if (tank<0) { start=i+1; tank=0; }
}
return total>=0 ? start : -1;

// ─────────────────────────────────────────────────────────────────
// 6. CANDY (two-pass, combine with max)
// ─────────────────────────────────────────────────────────────────
vector<int> c(n,1);
for (int i=1;i<n;i++) if (r[i]>r[i-1]) c[i]=c[i-1]+1;
for (int i=n-2;i>=0;i--) if (r[i]>r[i+1]) c[i]=max(c[i],c[i+1]+1);

// ─────────────────────────────────────────────────────────────────
// 7. TASK SCHEDULER (frequency formula)
// ─────────────────────────────────────────────────────────────────
int maxFreq=*max_element(freq.begin(),freq.end());
int numMax=count(freq.begin(),freq.end(),maxFreq);
int ans=max((int)tasks.size(), (maxFreq-1)*(k+1)+numMax);

// ─────────────────────────────────────────────────────────────────
// 8. MINIMUM ARROWS (interval cover, strict > boundary)
// ─────────────────────────────────────────────────────────────────
sort(pts.begin(),pts.end(),[](auto&a,auto&b){return a[1]<b[1];});
int arrows=1; long long pos=pts[0][1];
for (auto&p:pts) if (p[0]>pos) { arrows++; pos=p[1]; }

// ─────────────────────────────────────────────────────────────────
// 9. QUEUE RECONSTRUCTION (sort desc height, insert at k)
// ─────────────────────────────────────────────────────────────────
sort(people.begin(),people.end(),[](auto&a,auto&b){
    return a[0]!=b[0] ? a[0]>b[0] : a[1]<b[1];
});
vector<vector<int>> res;
for (auto&p:people) res.insert(res.begin()+p[1], p);

// ─────────────────────────────────────────────────────────────────
// 10. PARTITION LABELS (last-occurrence window)
// ─────────────────────────────────────────────────────────────────
int start=0,end=0;
for (int i=0;i<s.size();i++) {
    end=max(end,last[s[i]-'a']);
    if (i==end) { result.push_back(end-start+1); start=i+1; }
}

// ─────────────────────────────────────────────────────────────────
// 11. DECISION TABLE: WHICH PROOF DOES THIS PROBLEM NEED?
// ─────────────────────────────────────────────────────────────────
// Output is a discrete SET/SEQUENCE of chosen items -> Exchange argument
// Output is a running/accumulating THRESHOLD (reachability, tank,
//   bloom day, HP) -> Greedy stays ahead
// Cannot construct either proof, OR you found a counterexample ->
//   it's not greedy. Reach for DP (Pattern 16/19).
```

---

## SECTION 24 — WHAT TO DO AFTER THIS PATTERN

1. **Solve all 11 problems in order, and for EACH ONE, write down which proof technique you used before looking at the solution.** This forces the discipline the whole pattern is about — a working greedy without a stated proof doesn't count as mastered.

2. **Deliberately try to break your own greedy.** After solving Gas Station, try to construct an input where the reset-scan gives the wrong answer. You won't find one (the proof in Section 7 guarantees it), but the exercise of TRYING builds the counterexample-hunting instinct that separates 2000-rated solvers from 1600-rated ones.

3. **Revisit Section 15 after finishing the problem set.** Try to write a greedy for 0/1 Knapsack yourself, fail to find a valid exchange argument, and only then look at the DP solution (Pattern 19). Understanding exactly WHERE the exchange argument breaks is more valuable than memorizing that "0/1 knapsack needs DP."

4. **Preview Pattern 11 (Intervals):** Minimum Arrows, Minimum Platforms, and Activity Selection are really "interval algorithms" wearing a greedy proof. Pattern 11 goes deeper on interval manipulation (merging, insertion, sweep-line counting) that these greedy proofs assume as a prerequisite.

5. **Contest practice:** on Codeforces, filter for tag `greedy` at your rating band and, for every problem, force yourself to write a one-paragraph proof sketch in a comment before submitting. Track how often your first "obvious" greedy idea turns out to need a different sort key once you actually try to prove it — this ratio is a good personal signal of how much of this pattern has actually sunk in.

---

## SECTION 25 — SIGN-OFF CRITERIA

### Tier 1 — Basic greedy mechanics
You solve Assign Cookies and Jump Game cleanly, and can state (even informally) why the greedy choice at each step cannot be beaten by waiting.

### Tier 2 — Proof-literate
You can produce a correct exchange argument for Activity Selection / Minimum Arrows from scratch, without looking it up, including identifying the correct sort key and justifying it (not just asserting it). You solve Gas Station and can explain why the reset point works using the prefix-sum argument.

### Tier 3 — Multi-constraint and derived-formula greedy solid
You solve Candy (two-pass combination) and Task Scheduler (frequency formula, including correctly deriving the `-1` and the `max(n, skeleton)` guard) without hesitation. You can explain Queue Reconstruction's insertion invariant.

### Tier 4 — Contest-ready
You solve Full Bloom and Minimum Replacements under time pressure, correctly identifying the adjacent-swap / reverse-greedy structure without hints. You can, when given an unfamiliar greedy-shaped problem, correctly decide within a few minutes whether an exchange argument or stays-ahead proof applies — or correctly identify that the problem actually needs DP by constructing a counterexample.

**Sign-off threshold:** Solve 8 of 11 problems. Mandatory: LC 134 (Gas Station — full prefix-sum proof required), LC 621 (Task Scheduler — formula derivation required), and at least one of LC 2136 / LC 2366 (contest-tier reverse/adjacent-swap reasoning). These three represent the three hardest proof-obligations in this pattern.

---

*Pattern 34 Complete — Greedy (Formal Treatment)*
*Next: Pattern 35 — Backtracking Fundamentals (state-space search, pruning, the difference between "generate all" and "generate valid")*
