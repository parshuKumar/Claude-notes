# PATTERN 37 — COMPANY-SPECIFIC PATTERNS
## Signature Interview Styles Across FAANG, Indian Unicorns, and Startups — Complete Grandmaster Guide
### From 1600 → 2000+ | C++ | Interview + Contest Ready

---

## SECTION 1 — HONEST TAKE

**This document is not an algorithm pattern. It is meta-knowledge.**

Every other pattern in this curriculum (01-35) teaches you a technique that transfers everywhere — a sliding window works the same at Google as it does at a 10-person startup. Pattern 37 teaches you something different: **where to point that technique first, given limited prep time.**

**The core claim, stated honestly:** companies have measurable, persistent tendencies in the problems they ask, driven by three real forces:

1. **Interviewer question banks are internal and reused.** Amazon SDE interviewers pull from a shared internal bank that's audited for LP (Leadership Principle) tie-in potential. Google interviewers are trained to prefer problems with rich follow-up space (to probe past the "solved it" moment). These banks don't change randomly — they reflect what each company's interview process is *designed to measure*.
2. **Engineering culture selects for certain skills.** A company whose production systems are dominated by event-driven, graph-shaped infrastructure (dependency graphs, service meshes) will produce interviewers who reach for graph problems naturally, because that's the mental model they live in daily. This is not conspiracy — it's revealed preference.
3. **Question-bank rotation is slow.** Loop structures, round counts, and even specific problems leak (Glassdoor, Blind, LeetCode Discuss) and get reused for 1-2 years before rotating out. Recency matters, but "recency" here means quarters, not days.

**The equally honest caveat: this is a probability shift, not a guarantee.**

- Individual interviewers deviate from company norms constantly. A Google interviewer *can* ask you a pure implementation problem. An Amazon bar raiser *can* ask a DP problem if it's their personal favorite.
- LeetCode's company tags are crowd-sourced and gated behind Premium — they are a noisy signal, not ground truth (Section 6 covers how to weight them correctly).
- **Fundamentals still dominate the outcome.** If you have not mastered Patterns 01-15 (arrays, two pointers, sliding window, stacks, binary search, prefix sums, linked lists, trees, heaps, intervals, graphs BFS/DFS, backtracking), no amount of "Amazon-specific" prep saves you — those patterns appear in every company's loop, unconditionally.
- Targeted prep is a **multiplier on a strong base**, not a substitute for one. Read this document as: "given that I already have Patterns 01-21 solid, how do I spend my last 7-10 days most efficiently for Company X?"

**The skill you're actually building:** the ability to read a job posting, a LeetCode company tag list, and 5-10 Glassdoor reviews, and within 30 minutes produce an accurate probability distribution over which of the 35 patterns will show up — then bias your remaining practice time toward the top 40% of that distribution while never fully abandoning the rest.

**Abbreviations used throughout this document:**

| Abbreviation | Meaning |
|---|---|
| OA | Online Assessment — the automated, unproctored-or-proctored coding test stage before human interviews |
| LP | Leadership Principle (Amazon's 16-point evaluation framework, referenced throughout Sections 2.1, 7, 8, 9) |
| LLD | Low-Level Design — class/interface-level object-oriented design, evaluated via a "machine coding" round |
| HLD | High-Level Design — distributed-systems-scale design (services, data stores, scaling trade-offs) |
| SDE / IC | Software Development Engineer / Individual Contributor — generic leveling shorthand used across Section 7's leveling table |
| STAR | Situation, Task, Action, Result — the standard structure expected for behavioral answers, especially at Amazon |

---

## SECTION 2 — PER-COMPANY BREAKDOWNS

Each breakdown gives: signature style, favored patterns (cross-referenced by number to Patterns 01-35), typical difficulty, round structure, and 5-8 representative LeetCode problems. Difficulty and round structure reflect the L4/L5-equivalent (mid-level SDE) loop unless noted; senior loops add a system design round and raise the DP/graph depth bar.

---

### 2.1 AMAZON

**Signature style:** Grid and graph traversal (BFS/DFS), heaps for top-K/scheduling, interval merging, greedy — all wrapped in a story that maps to a Leadership Principle (LP). Amazon interviewers are trained to evaluate *both* the code and which LP the candidate demonstrated while solving it (e.g., "Dive Deep" when you catch an edge case unprompted, "Bias for Action" when you commit to an approach instead of over-deliberating).

| Favored Patterns | Frequency |
|---|---|
| 12 — Graphs BFS/DFS (grid flood-fill, multi-source BFS) | ALWAYS |
| 10 — Heap/Priority Queue (top-K, scheduling) | ALWAYS |
| 11 — Intervals (merge, meeting rooms) | ALWAYS |
| 34 — Greedy | ALWAYS |
| 01 — Arrays/Hashing | OFTEN |
| 07 — Linked Lists (LRU/LFU design) | OFTEN |
| 26 — Trie (search autocomplete) | OFTEN |

**Typical difficulty:** Medium, occasionally Medium-Hard. Amazon rarely asks LC Hard in early rounds; the bar raiser round is where difficulty spikes.

**Round structure:**
1. **OA (Online Assessment):** 2 coding problems (45-70 min combined) + a "work simulation" (situational judgment questions mapped to LPs) + sometimes a debugging exercise.
2. **Phone screen:** 1 coding problem + LP behavioral questions (STAR format expected).
3. **Onsite loop (4-6 rounds):** Each round blends ~15 min behavioral (2 LP stories) + 30-40 min coding (1-2 problems). One round is the **bar raiser** — an interviewer from outside the hiring team whose sole job is to protect the long-term hiring bar; they weight LP evidence more heavily than raw coding speed.

**Representative problems:**
- LC 200 — Number of Islands (grid BFS/DFS)
- LC 973 — K Closest Points to Origin (heap)
- LC 56 — Merge Intervals
- LC 621 — Task Scheduler (greedy + heap)
- LC 994 — Rotting Oranges (multi-source BFS)
- LC 146 — LRU Cache (design)
- LC 253 — Meeting Rooms II
- LC 1268 — Search Suggestions System (trie)

---

### 2.2 GOOGLE

**Signature style:** DP, math/number theory, and graph theory, with a strong emphasis on **provably correct, clean code** and the interviewer's ability to keep pushing follow-ups ("now what if the array is a stream?", "can you do this in O(1) extra space?"). Google interviewers are explicitly trained to leave headroom in the problem so a strong candidate can keep demonstrating depth for the full 45 minutes.

| Favored Patterns | Frequency |
|---|---|
| 16-21 — DP family (fundamentals through LIS) | ALWAYS |
| 35 — Math and Number Theory | OFTEN |
| 12/14 — Graphs (BFS/DFS, shortest path) | ALWAYS |
| 30 — Topological Sort | OFTEN |
| 20 — LCS Family (Edit Distance, string DP) | OFTEN |
| 15 — Backtracking | OFTEN |

**Typical difficulty:** Medium-Hard to Hard. Google is the company most willing to ask an LC Hard in a standard onsite round, especially for L4+.

**Round structure:**
1. **Phone screen:** 1 coding problem (45 min) via Google Doc or CoderPad — no auto-complete, tests communication under a harsher-than-usual interface.
2. **Onsite (4-5 rounds):** Almost entirely coding + algorithm design, no explicit LP framework — instead scored across 4 axes: General Cognitive Ability (GCA), Role-Related Knowledge (RRK), Leadership, and "Googleyness." One round may be a **Googleyness & Leadership** behavioral round.
3. **Senior (L5+):** Adds a system design round; a hiring committee (not the interviewers alone) makes the final call, so written feedback quality matters as much as raw performance.

**Representative problems:**
- LC 72 — Edit Distance
- LC 127 — Word Ladder
- LC 329 — Longest Increasing Path in a Matrix
- LC 42 — Trapping Rain Water
- LC 253 — Meeting Rooms II
- LC 68 — Text Justification
- LC 4 — Median of Two Sorted Arrays
- LC 149 — Max Points on a Line (geometry/math)

---

### 2.3 MICROSOFT

**Signature style:** Trees, linked lists, strings, and arrays — Microsoft's loop skews toward the "classic CS fundamentals" end of the spectrum rather than exotic DP or advanced graph theory. Team-specific rounds vary more than at Amazon/Google (an Azure infra team may probe systems/OS depth that a Office product team never touches).

| Favored Patterns | Frequency |
|---|---|
| 07 — Linked Lists | ALWAYS |
| 08 — Trees DFS | ALWAYS |
| 33 — String Algorithms | ALWAYS |
| 01 — Arrays/Hashing | OFTEN |
| 09 — Trees BFS | OFTEN |
| 12 — Graphs BFS/DFS (Clone Graph, matrix traversal) | OFTEN |

**Typical difficulty:** Easy-Medium to Medium. Microsoft is generally considered the most approachable FAANG-adjacent bar among the majors — the differentiator is clean, bug-free code and clear communication, not algorithmic novelty.

**Round structure:**
1. **OA (sometimes, HackerRank-based):** 2 problems, 60-90 min, used mainly for university/early-career hiring.
2. **Phone screen:** 1-2 coding problems.
3. **Onsite loop (4-5 rounds):** Mix of coding rounds and a "resume/project deep dive" round where the interviewer probes a past project in real depth (design decisions, trade-offs, what you'd change). Behavioral is framed around Microsoft's "growth mindset" culture rather than a fixed LP list.

**Representative problems:**
- LC 206 — Reverse Linked List
- LC 236 — Lowest Common Ancestor of a Binary Tree
- LC 297 — Serialize and Deserialize Binary Tree
- LC 54 — Spiral Matrix
- LC 133 — Clone Graph
- LC 8 — String to Integer (atoi)
- LC 227 — Basic Calculator II

---

### 2.4 META / FACEBOOK

**Signature style:** Arrays/strings/hashing dominate by raw tag frequency, followed by BFS on trees and interval problems. Meta is famous for a **fast, two-problems-per-35-minute-round** format — this format structurally favors patterns with short, clean implementations (arrays, hashing, two pointers) over patterns that need 30+ minutes just to set up state (heavy multi-dimensional DP rarely fits).

| Favored Patterns | Frequency |
|---|---|
| 01 — Arrays/Hashing | ALWAYS |
| 11 — Intervals | ALWAYS |
| 03 — Sliding Window | OFTEN |
| 06 — Prefix Sums | OFTEN |
| 09 — Trees BFS (vertical order, level order) | OFTEN |
| 02 — Two Pointers | OFTEN |

**Typical difficulty:** Medium, but under tight time pressure — the effective difficulty is higher than the LC label suggests because you're expected to finish two mediums in 35 minutes including a working, tested solution.

**Round structure:**
1. **Phone screens (2 rounds, coding only):** Each ~45 min, typically 1-2 problems.
2. **Onsite (4-6 rounds):** 2 coding rounds (2 problems each), 1 **behavioral/culture-fit** round scored independently of coding, and for senior candidates 1-2 **system design** rounds split explicitly into "Product Design" (build a feature end-to-end) and "Infrastructure Design" (scale a backend system) as two distinct interview types.

**Representative problems:**
- LC 680 — Valid Palindrome II
- LC 56 — Merge Intervals
- LC 560 — Subarray Sum Equals K
- LC 314 — Binary Tree Vertical Order Traversal
- LC 528 — Random Pick with Weight
- LC 341 — Flatten Nested List Iterator
- LC 227 — Basic Calculator II
- LC 240 — Search a 2D Matrix II

---

### 2.5 APPLE

**Signature style:** The least standardized of the majors — Apple's secrecy culture means less public data, and team-to-team variance is the highest of any FAANG company. What's consistent: practical, no-frills problems (fewer "trick" DP problems), a strong bias toward **object-oriented design** questions, and for platform/systems teams, real depth on memory management, concurrency, or OS internals that a pure-algorithms candidate won't expect.

| Favored Patterns | Frequency |
|---|---|
| 01 — Arrays/Hashing | OFTEN |
| 07 — Linked Lists (LRU-style design) | OFTEN |
| 02 — Two Pointers | OFTEN |
| 08 — Trees DFS | OFTEN |
| 33 — String Algorithms | OFTEN |

**Typical difficulty:** Easy-Medium for algorithmic rounds; the differentiator is a dedicated **design round** (OOD, not distributed systems) that other majors fold into a single "design" label.

**Round structure:**
1. **Recruiter screen**, then **1-2 technical phone screens** (coding).
2. **Onsite (4-5 rounds):** Mix of algorithmic coding, a practical design round (e.g., "design a parking system" or "design Twitter's follow feature" at the class/interface level, not distributed infra scale), and team-specific rounds that can diverge sharply (a Core OS team may ask about memory allocators; a Services team may ask standard LC mediums).

**Representative problems:**
- LC 1 — Two Sum
- LC 167 — Two Sum II (sorted input)
- LC 146 — LRU Cache
- LC 380 — Insert Delete GetRandom O(1)
- LC 359 — Logger Rate Limiter (design)
- LC 1472 — Design Browser History

---

### 2.6 INDIAN UNICORNS — ZEPTO / SWIGGY / RAZORPAY / MEESHO / FLIPKART / CRED

**Signature style:** Arrays, sliding window, binary search on the answer, hashing, heaps, and a sprinkling of DP — heavily influenced by HackerRank/HackerEarth-style OAs (2-3 problems, one deliberately hard to separate the top decile). The single biggest differentiator from FAANG: nearly all of these companies run a **dedicated machine-coding / Low-Level Design (LLD) round** — build a working, compilable, testable mini-system (Splitwise, a rate limiter, an in-memory cache, a parking lot) in 60-90 minutes, evaluated on OOP design, extensibility, and whether it actually runs, not just DSA correctness.

| Favored Patterns | Frequency |
|---|---|
| 03 — Sliding Window | ALWAYS |
| 05 — Binary Search (incl. binary search on answer) | ALWAYS |
| 01 — Arrays/Hashing | ALWAYS |
| 10 — Heap | OFTEN |
| 04 — Monotonic Stack/Deque | OFTEN |
| 17/19 — 1D DP, Knapsack | SOMETIMES |
| 34 — Greedy | OFTEN |

**Typical difficulty:** Medium on average, but the OA's "hard" problem and the LLD round are where candidates are actually filtered — many candidates who clear FAANG-style DSA rounds fail the LLD round because they've never designed a class hierarchy for extensibility.

**Round structure:**
1. **OA (HackerRank/HackerEarth):** 2-3 problems in 90 min, one deliberately above-medium difficulty (often a binary-search-on-answer or DP problem).
2. **Technical round 1-2:** 1-2 DSA problems each, live coding, often on a shared doc/CoderPad without auto-complete.
3. **Machine coding / LLD round:** A self-contained system built from scratch — no starter code, no framework, pure OOP + data structures, graded on running tests within the time box.
4. **Hiring manager / culture round:** Ownership and "how do you operate with ambiguity" style questions — the startup/scale-up equivalent of Amazon's LPs, but far less formalized.

**Representative problems:**
- LC 239 — Sliding Window Maximum
- LC 53 — Maximum Subarray (Kadane's)
- LC 875 — Koko Eating Bananas (binary search on answer)
- LC 347 — Top K Frequent Elements
- LC 56 — Merge Intervals
- LC 3 — Longest Substring Without Repeating Characters
- LC 84 — Largest Rectangle in Histogram (recurring Flipkart favorite)
- Machine coding: design an in-memory rate limiter, Splitwise-style expense splitter, or LRU-based cache — not a single LC number, but the equivalent of building LC 146 (LRU Cache) and LC 355 (Design Twitter) as full production-quality classes with tests.

---

### 2.7 STARTUPS (SEED TO SERIES C)

**Signature style:** Breadth over depth. Arrays, strings, hashing, and basic problem-solving under real time pressure — startups rarely have the interviewer bandwidth to run a 5-round FAANG-style loop, and they're optimizing for "can this person ship working code on our actual stack," not algorithmic virtuosity.

| Favored Patterns | Frequency |
|---|---|
| 01 — Arrays/Hashing | ALWAYS |
| 02 — Two Pointers | OFTEN |
| 03 — Sliding Window | OFTEN |
| 33 — String Algorithms | OFTEN |
| 11 — Intervals | SOMETIMES |

**Typical difficulty:** Easy-Medium, or no LC-style round at all — many startups replace algorithmic rounds with a **take-home assignment** (build a small feature/API against a spec) evaluated on code quality, test coverage, and real-world trade-off decisions rather than time complexity.

**Round structure:**
1. **Recruiter/founder call.**
2. **1-2 technical rounds:** Either live coding (1-2 easy/medium problems) OR pair programming on a real snippet of their codebase OR a take-home project with a 3-7 day window.
3. **Founder/culture chat:** Frequently the deciding round — "why us," ownership mentality, comfort with ambiguity.

**Representative problems:**
- LC 1 — Two Sum
- LC 242 — Valid Anagram
- LC 49 — Group Anagrams
- LC 3 — Longest Substring Without Repeating Characters
- LC 56 — Merge Intervals
- Take-home equivalent: extend a small REST endpoint, fix a bug in an existing repo, or implement a documented feature end-to-end.

---

### 2.8 QUICK REFERENCE — ROUND STRUCTURE ACROSS ALL SEVEN TRACKS

Use this table to sanity-check your prep-week schedule against what the loop actually contains — if the target company has no LLD round, don't burn a day on Section 4.3's Day 4-5 activities; if it has no formal LP framework, don't over-rehearse rigid STAR templates.

| Company | OA Stage? | Onsite Rounds | Dedicated Behavioral Round? | LLD/Machine Coding Round? | System Design Stage |
|---|---|---|---|---|---|
| Amazon | Yes (2 coding + work sim) | 4-6 | Woven into every coding round + 1 bar raiser | No (design appears inside coding rounds at senior level) | Senior levels only |
| Google | No (phone screen substitutes) | 4-5 | Yes (Googleyness & Leadership) | No | L5+ |
| Microsoft | Sometimes (early career) | 4-5 | Light, folded into project deep-dive | No | Team-dependent |
| Meta | No (2 phone screens substitute) | 4-6 | Yes, scored independently | No | Senior levels, split into Product + Infra Design |
| Apple | No | 4-5 | Light, team-dependent | Sometimes (as "practical OOD," not full machine coding) | Team-dependent |
| Indian Unicorns | Yes (HackerRank/HackerEarth) | 2-3 + LLD | Yes, less formalized ("ownership" framing) | **Yes — near-universal at this tier** | Product-scale, often lighter than FAANG |
| Startups | Rare | 1-2 (or take-home) | Yes (founder/culture chat, often decisive) | No | Rare |

---

## SECTION 3 — PATTERN × COMPANY FREQUENCY MATRIX

Legend: **ALWAYS** = appears in nearly every loop; **OFTEN** = appears regularly, plan for it; **SOMETIMES** = occasional, know it but don't over-invest; **RARE** = very unlikely in a standard interview loop (but common in contests — see Section 5).

Read this table as a *prioritization aid*, not a checklist to memorize verbatim — it compresses hundreds of data points (Glassdoor, LC company tags, personal interview reports) into a single heuristic grade per cell.

| # | Pattern | Amazon | Google | Microsoft | Meta | Apple | Indian Unicorns | Startups |
|---|---------|--------|--------|-----------|------|-------|------------------|----------|
| 01 | Arrays and Hashing | OFTEN | OFTEN | OFTEN | ALWAYS | OFTEN | ALWAYS | ALWAYS |
| 02 | Two Pointers | OFTEN | SOMETIMES | OFTEN | OFTEN | OFTEN | OFTEN | OFTEN |
| 03 | Sliding Window | SOMETIMES | SOMETIMES | SOMETIMES | OFTEN | SOMETIMES | ALWAYS | OFTEN |
| 04 | Stack / Monotonic Stack | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | OFTEN | RARE |
| 05 | Binary Search | OFTEN | OFTEN | SOMETIMES | SOMETIMES | SOMETIMES | ALWAYS | SOMETIMES |
| 06 | Prefix Sums / Diff Arrays | SOMETIMES | SOMETIMES | SOMETIMES | OFTEN | SOMETIMES | OFTEN | SOMETIMES |
| 07 | Linked Lists | OFTEN | SOMETIMES | ALWAYS | SOMETIMES | OFTEN | SOMETIMES | RARE |
| 08 | Trees — DFS | OFTEN | OFTEN | ALWAYS | OFTEN | OFTEN | SOMETIMES | RARE |
| 09 | Trees — BFS | OFTEN | SOMETIMES | OFTEN | OFTEN | SOMETIMES | SOMETIMES | RARE |
| 10 | Heap / Priority Queue | ALWAYS | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | OFTEN | RARE |
| 11 | Intervals | ALWAYS | OFTEN | SOMETIMES | ALWAYS | SOMETIMES | OFTEN | SOMETIMES |
| 12 | Graphs — BFS/DFS | ALWAYS | ALWAYS | OFTEN | OFTEN | SOMETIMES | SOMETIMES | RARE |
| 13 | Graphs — Union Find | SOMETIMES | SOMETIMES | RARE | SOMETIMES | RARE | SOMETIMES | RARE |
| 14 | Graphs — Shortest Path | SOMETIMES | OFTEN | RARE | SOMETIMES | RARE | RARE | RARE |
| 15 | Backtracking | SOMETIMES | OFTEN | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | RARE |
| 16 | DP Fundamentals | SOMETIMES | ALWAYS | SOMETIMES | SOMETIMES | RARE | SOMETIMES | RARE |
| 17 | 1D DP | SOMETIMES | OFTEN | SOMETIMES | SOMETIMES | RARE | SOMETIMES | RARE |
| 18 | 2D DP / Grid DP | RARE | OFTEN | RARE | SOMETIMES | RARE | RARE | RARE |
| 19 | Knapsack Family | RARE | SOMETIMES | RARE | RARE | RARE | SOMETIMES | RARE |
| 20 | LCS Family | RARE | OFTEN | SOMETIMES | SOMETIMES | RARE | RARE | RARE |
| 21 | LIS Family | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 22 | DP on Trees | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 23 | Interval DP | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 24 | Bitmask DP | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 25 | Digit DP | RARE | RARE | RARE | RARE | RARE | RARE | RARE |
| 26 | Trie | OFTEN | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | RARE |
| 27 | Segment Tree | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 28 | Fenwick Tree | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 29 | Monotonic Deque | SOMETIMES | SOMETIMES | RARE | SOMETIMES | RARE | OFTEN | RARE |
| 30 | Topological Sort | OFTEN | OFTEN | SOMETIMES | SOMETIMES | RARE | SOMETIMES | RARE |
| 31 | Minimum Spanning Tree | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 32 | Strongly Connected Components | RARE | SOMETIMES | RARE | RARE | RARE | RARE | RARE |
| 33 | String Algorithms | SOMETIMES | OFTEN | ALWAYS | OFTEN | OFTEN | OFTEN | OFTEN |
| 34 | Greedy | ALWAYS | OFTEN | SOMETIMES | SOMETIMES | SOMETIMES | OFTEN | SOMETIMES |
| 35 | Math and Number Theory | RARE | OFTEN | SOMETIMES | SOMETIMES | SOMETIMES | SOMETIMES | RARE |

**How to read the RARE column for patterns 18-32 at most companies:** this is not "these patterns don't matter" — it's "don't spend your last 7 days on Interval DP for an Amazon loop." These same patterns are *dominant* in Codeforces contests (Section 5). If you're doing both interview prep and rating climbing, patterns 22-32 are where the two tracks diverge most sharply.

---

## SECTION 4 — SEVEN-DAY PREP PLANS

These assume you already have Patterns 01-21 at Tier 2+ mastery (per each pattern's own sign-off criteria) and are doing a *final, targeted* week before a specific loop — not learning DSA from scratch in a week.

### 4.1 AMAZON — 7-DAY PLAN

| Day | Focus | Activities |
|---|---|---|
| 1 | Graphs BFS/DFS on grids (P12) | Solve LC 200, 994, 733, 417, 1091. Drill multi-source BFS until it's automatic. |
| 2 | Heaps + greedy (P10, P34) | Solve LC 973, 347, 621, 767, 502. Time-box each to 20 min. |
| 3 | Intervals (P11) | Solve LC 56, 57, 253, 435, 759. Write the sort-then-sweep template from memory 3 times. |
| 4 | Linked lists + design (P07) | Solve LC 146, 23, 138. Practice explaining doubly-linked-list + hashmap for LRU out loud. |
| 5 | LP story bank | Write 2-3 STAR stories per LP (Customer Obsession, Ownership, Dive Deep, Bias for Action, Deliver Results — minimum). Rehearse out loud, not just on paper. |
| 6 | Mixed mock — coding + LP in same session | Full mock: 1 medium graph/heap problem + 2 LP questions in 45 min, simulating the actual round format. |
| 7 | Light review + rest | Re-skim your weakest pattern's cheat sheet. No new problems. Sleep is a bigger ROI than one more LC problem the night before. |

### 4.2 GOOGLE — 7-DAY PLAN

| Day | Focus | Activities |
|---|---|---|
| 1 | DP fundamentals + 1D DP refresh (P16-17) | Solve LC 198, 139, 300 cold, no hints. Verbalize state/transition/base case before coding. |
| 2 | 2D DP + LCS family (P18, P20) | Solve LC 221, 174, 72, 1143. For each, explain WHY the state is defined that way before writing code. |
| 3 | Graphs + topological sort (P12, P30) | Solve LC 207, 210, 127, 329. Practice both BFS and DFS implementations of the same problem. |
| 4 | Math / number theory (P35) | Solve LC 149, 264, handful of GCD/modular arithmetic warmups. Review complexity of √n factorization, sieve. |
| 5 | Follow-up drilling | Pick 3 solved problems; for each, generate your own follow-up ("what if input is a stream," "what if we need O(1) space") and solve it. This simulates Google's actual interview pressure. |
| 6 | Full mock, clean-code emphasis | 45-min mock on an unfamiliar Hard, graded on code cleanliness and communication, not just correctness. |
| 7 | Light review + rest | Re-read your DP and graph cheat sheets. No new hard problems — consolidate, don't cram. |

### 4.3 INDIAN UNICORN (e.g., RAZORPAY/FLIPKART) — 7-DAY PLAN

| Day | Focus | Activities |
|---|---|---|
| 1 | Sliding window + two pointers (P02-03) | Solve LC 3, 76, 239, 424, 992. Drill fixed vs. variable window templates until automatic. |
| 2 | Binary search on answer (P05) | Solve LC 875, 1011, 410, 1552. Write the `while(lo<hi)` binary-search-on-answer template from memory. |
| 3 | Hashing + heaps (P01, P10) | Solve LC 347, 560, 128, 973. Mix in one HackerRank-style "OA simulation" set, timed 90 min for 2-3 problems. |
| 4 | Machine coding / LLD dry run #1 | Design and fully implement (compilable, with a small test) an LRU cache or rate limiter from scratch, no notes, 75 min. Review your own class design for extensibility afterward. |
| 5 | Machine coding / LLD dry run #2 | Repeat with a different system (Splitwise-style expense splitter or a parking lot). Focus on SOLID principles and how you'd extend it if requirements changed. |
| 6 | Full mixed mock | Simulate the real loop: 1 OA-style timed set (2 problems, 90 min) + 1 LLD round (75 min) back to back. |
| 7 | Light review + rest | Skim your sliding window and binary-search-on-answer cheat sheets. Re-read your LLD code for obvious bugs. No new systems the night before. |

---

## SECTION 5 — CODEFORCES MAPPING: WHICH PATTERNS DOMINATE EACH DIFFICULTY BAND

Contest rating and interview readiness are correlated but **not the same skill**, and the pattern distribution diverges sharply past a certain difficulty. Understanding this divergence prevents you from either (a) over-investing in exotic DS for interviews or (b) under-investing in them if your real goal is Codeforces rating.

| Slot | Dominant Patterns | Notes |
|---|---|---|
| Div 2 A | 01 (ad-hoc arrays), 34 (simulation/greedy), basic 35 (parity/arithmetic) | Almost never a "named" algorithm — pure careful reading + implementation. |
| Div 2 B | 01, 02, 34, simple 05 | One small insight beyond brute force; still implementation-heavy. |
| Div 2 C | 16/17 (DP fundamentals, 1D DP), 05 (binary search on answer), 12 (basic BFS/DFS), 34 | This is where "recognize the pattern" starts to matter as much as coding speed. |
| Div 2 D | 18/19 (2D DP, knapsack), 13 (DSU), 14 (shortest path), 24 (small-N bitmask DP), 27/28 (segment tree/Fenwick) | The jump from C to D is usually "add a data structure" or "add a dimension to the DP." |
| Div 2 E | 20-25 (LCS/LIS/tree DP/interval DP/digit DP), 31/32 (MST/SCC), 33 (advanced string algorithms — Z-function, suffix automaton, KMP variants), heavy 35 (number theory, combinatorics, modular inverses) | This is squarely where Patterns 22-32 — RARE in the Section 3 table — become essential. |
| Div 3 | Same pattern pool as Div 2 A-D, but **problems are explicitly telegraphed by position** (the letter tells you roughly how many steps of insight are needed) and edge cases are gentler. Designed for sub-1600 to ~1900 rated climbers. |

**Approximate per-problem rating bands (Div 2), for calibrating practice difficulty:**

| Slot | Approx. Rating Band | What this means for practice |
|---|---|---|
| A | 800-1200 | If you're missing these, the gap is reading comprehension and implementation speed, not algorithms. |
| B | 1200-1600 | First point where "spot the greedy/two-pointer insight" separates finishers from non-finishers. |
| C | 1500-1900 | The pattern-recognition tier — most of Patterns 01-17 pay off here directly. |
| D | 1900-2200 | Patterns 18-19, 13-14, 24, 27-28 start being load-bearing, not optional. |
| E | 2200-2500 | Patterns 20-25, 31-33, 35 dominate; expect at least one genuinely novel combination of two techniques. |

These bands overlap and shift per-round (a "hard C, easy D" contest is common) — treat them as a coarse compass, not a guarantee, and always calibrate against your own recent contest performance rather than the table alone.

**Why contest prep and interview prep diverge past Pattern ~21:**

- **Interviews optimize for depth on one problem + communication.** A 45-minute round has room for exactly one non-trivial idea, explained clearly, with tests. Patterns 22-32 (interval DP, bitmask DP, segment trees, SCC, MST) require enough setup that they rarely fit a single round cleanly — and most company interviewer banks were curated with that constraint in mind.
- **Contests optimize for breadth under time pressure across 5-8 problems in 2 hours.** A Div 2 E problem assumes you already have segment trees, DSU, and modular combinatorics as zero-thought tools, so the contest can spend its "difficulty budget" on a genuinely novel combination of them.
- **Practical implication:** if your only goal is interview-readiness, Patterns 22-32 are optional (helpful, but not load-bearing) — solidify 01-21 and Section 33-35 deeply instead. If you also want Codeforces rating past ~1900, Patterns 22-32 become mandatory, and you should budget real weeks for segment trees (P27), DSU (P13, already core for interviews too), and string algorithms (P33) specifically, since these three carry the most weight across the C-E band.

---

## SECTION 6 — HOW TO RESEARCH A SPECIFIC COMPANY

A repeatable 30-45 minute research protocol before you start targeted prep:

1. **LeetCode company tag (requires Premium).** Filter by the specific company, then sort/filter by the **"Last 6 Months"** or **"Last 30 Days"** window rather than "All Time." All-time tags accumulate a decade of stale data; loops rotate every 1-2 years, so recent-window frequency is the strongest single signal LeetCode gives you. A problem appearing 15+ times in the last 6 months is a real signal; a problem appearing twice in 5 years is noise.
2. **Cross-reference with Glassdoor.** Search "[Company] [Role] interview questions," sort by most recent, and read the last 15-20 reviews. Look specifically for candidates who describe the *actual problem content* (not just "they asked a medium leetcode question") — vague reviews are low-signal, specific ones ("asked to merge intervals then extended to handle overlapping meeting rooms") are high-signal and often map directly onto a known LC problem.
3. **Check Blind / Team Blind and recent LeetCode Discuss "Interview Experience" posts.** These skew more recent than Glassdoor and often include near-verbatim problem statements from loops in the last few weeks — the highest-recency signal available, but also the noisiest (unverified, sometimes exaggerated).
4. **Read 2-3 recent engineering blog posts from the company.** Not for DSA content directly, but to understand what kind of systems the company actually runs (heavy graph/dependency infra → expect more graph problems; heavy stream processing → expect more sliding window / prefix sum problems). This is the "why" behind the tag frequency, and it helps you generalize when the exact tagged problem isn't the one you get.
5. **Weight all of this against Section 3's baseline table.** If your research contradicts the general company archetype (e.g., you find unusually many DP-heavy reports for a company Section 3 marks OFTEN/SOMETIMES on DP), trust the fresher, more specific data — the baseline table is a prior, not a ceiling.

**The single biggest mistake in this research step:** treating a single data point (one Glassdoor review, one Blind post) as ground truth and over-fitting your entire prep week to it. Look for a *pattern across at least 5-10 independent reports* before you let it override the baseline.

**Worked example (illustrative walkthrough of the protocol):**

Say you have a screen booked in 8 days with a mid-size fintech unicorn you haven't researched yet. Here's how the 30-45 minute protocol plays out in practice:

1. *LC company tag, last 6 months:* You find 40 tagged submissions, clustering heavily around sliding window, binary search on answer, and one recurring hard DP problem. This alone would nudge you toward Section 3's "Indian Unicorn" column — but you don't stop here.
2. *Glassdoor, last 15 reviews:* 6 of them mention a "build a mini rate limiter / cache in 60 minutes" round with specific implementation constraints (must compile, must pass provided tests). This is the strongest single finding — it's not on LeetCode at all, and the baseline table (Section 2.8) already flags LLD as near-universal at this tier, so the finding *confirms* rather than contradicts the prior.
3. *Blind search:* Two recent posts (within the last 2 months) independently describe the same OA structure — 2 DSA problems, one clearly binary-search-on-answer shaped, plus a short "system trade-offs" written question. This converges with finding #1.
4. *Engineering blog:* A recent post describes migrating a core ledger service to an event-sourced architecture — this doesn't change your DSA prep, but it tells you the LLD round is likely to reward candidates who reason about idempotency and audit trails, not just raw OOP structure. You fold this into how you *narrate* your LLD round, not into which pattern you drill.
5. *Weigh against baseline:* Everything found agrees with Section 3's Indian Unicorn column and Section 2.8's LLD flag — no override needed. Final decision: run the Section 4.3 plan essentially as written, with Day 4's LLD dry run specifically built around a rate limiter (matching finding #2) rather than a generic Splitwise clone.

This is the shape every research pass should take: multiple independent sources, checked for convergence, reconciled against the baseline table, translated into a concrete change to *which* problems and *which* dry runs you pick for the week — not a vague "focus more on X."

---

## SECTION 7 — BEHAVIORAL AND SYSTEM DESIGN CONTEXT PER COMPANY (BRIEF)

| Company | Behavioral Framework | System Design Emphasis |
|---|---|---|
| Amazon | 16 Leadership Principles, mandatory STAR format, bar raiser round specifically audits LP evidence across the whole loop | Scale + cost + operational excellence (LPs: Ownership, Frugality directly show up in design trade-off discussions) |
| Google | "Googleyness & Leadership" — no fixed framework, scored across GCA/RRK/Communication/Googleyness axes by a hiring committee, not just the interviewer | Classic distributed systems fundamentals (consistency, partitioning, caching) — expected from L5+ |
| Microsoft | "Growth mindset" culture framing (post-2014 Nadella era), collaboration and learning-from-failure narratives valued | Highly team-dependent — ranges from none (junior product roles) to deep infra/OS design (platform teams) |
| Meta | Separate "Motivation/Behavioral/Culture" round, distinct from coding rounds and scored independently | Split into two explicit interview types at senior level: Product Design and Infrastructure Design |
| Apple | Least standardized; secrecy culture limits public prep material; often folded into technical rounds rather than a separate behavioral round | Practical OOD (class/interface design) more than distributed-systems scale; team-specific systems depth possible |
| Indian Unicorns | Hiring-manager round emphasizes ownership, hustle, comfort with ambiguity — less formalized than Amazon's LPs but similar intent | Product-pragmatic scale (millions, not billions, of users); LLD/machine-coding bar is often *higher* than FAANG, HLD bar often lower |
| Startups | Founder/culture-fit conversation, "what would you build in the first 30 days" style, direct evaluation of self-direction | Minimal to none; if present, it's "walk me through how you'd build feature X" rather than formal distributed systems design |

**Leveling and timeline context (approximate — varies by team, region, and market conditions):**

| Company | Typical Entry/Mid Track | Approx. Application-to-Offer Timeline | Note |
|---|---|---|---|
| Amazon | SDE I / SDE II | 3-6 weeks | A split decision can trigger an extra bar-raiser-driven round rather than an outright reject. |
| Google | L3 / L4 | 6-10 weeks | Slowest of the majors — a hiring committee reviews packets independently of the interviewers' live impressions. |
| Microsoft | SDE / SDE II (roughly level 59-63) | 3-5 weeks | Team-matching after the loop can add real time even after a positive decision. |
| Meta | E3 / E4 (IC) | 3-5 weeks | Among the fastest post-onsite decision turnarounds of the majors. |
| Apple | ICT2 / ICT3 | 4-8 weeks, highly variable | Team-specific loops mean the variance is structural, not incidental. |
| Indian Unicorns | SDE-1 / SDE-2 | 2-4 weeks | Often the fastest overall; the LLD round is frequently scheduled the same day as the DSA rounds. |
| Startups | Engineer / Senior Engineer | Days to 3 weeks | Founder-driven pace; a take-home's turnaround window is usually the single biggest timeline variable. |

**How to use this table:** timeline doesn't change what you study, but it changes how you *pace* Section 4's 7-day plans — a startup loop happening in 3 days compresses the plan into whatever fits, while a Google loop with weeks between phone screen and onsite gives you room to run a full 7-day plan more than once before the final round.

---

## SECTION 8 — COMMON MISTAKES IN COMPANY-SPECIFIC PREP

### Mistake 1: Over-indexing on a single stale data point

**Wrong approach:** You read one 3-year-old Glassdoor review claiming "Company X always asks graph problems" and spend your whole week on Pattern 12, ignoring everything else.

**Why it fails:** Interview loops rotate on a roughly 1-2 year cycle as question banks get retired for being too widely known. A single old data point tells you almost nothing about the current loop.

**Correct approach:** Require convergence across at least 5-10 independent, recent (last 6 months where possible) sources before letting any signal override the Section 3 baseline table.

---

### Mistake 2: Treating LP/culture prep as separate from coding prep (Amazon-specific)

**Wrong approach:** Spend 3 days doing pure LC grinding, then 1 day cramming Leadership Principle stories the night before, treating them as two unrelated tracks.

**Why it fails:** Amazon interviewers score the *same* coding round on both axes simultaneously — a candidate who solves the problem correctly but never narrates their reasoning, never mentions why they chose an edge-case check, or gets defensive when challenged, loses LP credit even with perfect code. The two are evaluated in the same 45 minutes by the same person.

**Correct approach:** Practice narrating LP-relevant reasoning *while* solving mock coding problems (Section 4.1, Day 6) rather than preparing them as isolated behavioral answers.

---

### Mistake 3: Preparing for an Indian unicorn using only FAANG resources

**Wrong approach:** Grind Blind 75 / Neetcode 150 exclusively and walk into a Razorpay or Flipkart loop assuming the DSA rounds are the whole story.

**Why it fails:** The machine-coding/LLD round is often the actual differentiator at these companies — plenty of candidates who ace two DSA rounds fail the LLD round because they've never designed a class hierarchy for extensibility under time pressure, a skill FAANG-style prep resources don't teach at all.

**Correct approach:** Budget explicit days for LLD dry runs (Section 4.3, Days 4-5) — design and fully implement a small system from scratch, not just read about design patterns.

---

### Mistake 4: Applying Google-level depth expectations to every company

**Wrong approach:** Burn your prep time forcing every problem to LC Hard difficulty with multiple follow-ups and O(1) space optimization, regardless of which company you're actually interviewing with next week.

**Why it fails:** A Meta round rewards finishing two clean mediums in 35 minutes, not agonizing over a single Hard with five follow-ups — the depth-first approach that wins at Google can actively hurt you at Meta by burning your limited round time. Effort invested in the wrong shape doesn't transfer.

**Correct approach:** Match your practice format (not just problem selection) to the target company's actual round structure — timed two-problem sprints for Meta, single-problem depth-with-follow-ups drilling for Google.

---

### Mistake 5: Over-investing in Patterns 22-35 for interview prep because they "feel more advanced"

**Wrong approach:** Assume that because segment trees and bitmask DP are objectively harder topics, mastering them will impress interviewers more than being fast and clean on arrays/hashing.

**Why it fails:** Section 3's table shows Patterns 22-32 sitting at RARE for nearly every company column — these patterns are advanced in an absolute sense but low-frequency in the specific context of a 45-minute interview round. Meanwhile Pattern 01 (Arrays/Hashing) sits at OFTEN or ALWAYS everywhere, and slow or shaky execution there is a much more common rejection reason than "candidate didn't know segment trees."

**Correct approach:** Keep Patterns 01-15 at automatic, zero-hesitation fluency regardless of target company — this is the non-negotiable floor under all company-specific prep, not an optional warmup.

---

### Mistake 6: Stopping at "it works" on Google-style follow-ups

**Wrong approach:** Get a correct, brute-force-to-optimal solution working, then go quiet and wait for the interviewer to move on.

**Why it fails:** Google interviewers deliberately leave headroom in problems to see how far a strong candidate can be pushed. Silence after "it works" reads as having hit your ceiling, even if you technically could go further with a nudge.

**Correct approach:** Proactively propose your own follow-up ("this works in O(n log n) — if we needed to handle a live stream of updates, I'd switch to X") before being asked. Section 4.2, Day 5 is built specifically to practice this reflex.

---

## SECTION 9 — INTERVIEW SIMULATIONS: COMPANY CONTEXT IN ACTION

### Simulation 1: Amazon — Coding and LP Evaluated Simultaneously

**Interviewer:** "Given a list of meeting intervals, merge all overlapping ones." (LC 56 — Merge Intervals)

**Candidate:**

*[States approach, but also narrates the reasoning trail an LP-aware interviewer is listening for]*
> "I'll sort by start time first — that's the key move that turns this from pairwise comparison into a single linear sweep. Before I code, let me check: are intervals guaranteed non-empty? Can start equal end? I'll assume closed intervals and touching intervals like [1,2] and [2,3] count as overlapping unless told otherwise — I'll confirm that assumption with you before I finalize."

*[Interviewer confirms the assumption]*

> "Good — I'll note that as a stated assumption in a comment so it's clear what edge case I resolved and why."

**Why this reads well for Amazon specifically:** the candidate surfaced an ambiguous edge case unprompted (LP: Dive Deep), stated an assumption rather than freezing (LP: Bias for Action), and made the resolution auditable via a comment (LP: Ownership — leaving the codebase clearer for the next person). None of this changed the algorithm; it changed how the *same* algorithm was evaluated.

**RED FLAGS an Amazon interviewer is trained to note:** silently assuming edge cases without flagging them, coding correctly but never explaining trade-offs, or becoming defensive when the interviewer probes "what if the input isn't sorted and is huge — would you still sort in memory?"

---

### Simulation 2: Google — Pushing Past "It Works"

**Interviewer:** "Find the shortest transformation sequence from beginWord to endWord, changing one letter at a time, using only words in a given wordList." (LC 127 — Word Ladder)

**Candidate:**

*[Solves with BFS on the implicit word graph, gets a working O(N · L² ) solution using L = word length]*

> "This works — BFS from beginWord, treating each single-letter mutation as an edge, gives the shortest path in O(N·L²) where N is dictionary size."

*[Proactively extends, without being asked]*

> "One follow-up I'd raise myself: if wordList is huge and mostly irrelevant to beginWord/endWord, I could do **bidirectional BFS** — grow the search frontier from both ends and meet in the middle — which cuts the effective branching factor roughly in half in the exponent. Want me to implement that variant instead?"

**Why this reads well for Google specifically:** the candidate treated the first working solution as a floor, not a finish line, and generated their own follow-up in the direction Google interviewers are trained to probe (further optimization, alternate algorithmic strategy) rather than waiting to be asked.

**RED FLAGS a Google interviewer is trained to note:** stopping at first correct solution without prompting, being unable to state the current complexity precisely, or proposing an "optimization" that doesn't actually change the asymptotic bound.

---

### Simulation 3: Indian Unicorn — Machine Coding / LLD Round Walkthrough

**Interviewer:** "Design and implement a rate limiter that allows at most N requests per user per rolling 60-second window. It should compile and pass the tests I'll give you at the end."

**Candidate:**

*[Starts with class design before any code, out loud]*
> "I'll model this as a `RateLimiter` class with a `bool allow(userId)` method. Internally, per user, I need a sliding window of request timestamps — I'll use a `deque<long long>` per user so I can pop expired timestamps off the front in O(1) amortized. I'll wrap per-user state behind an interface so if we swap the algorithm later — say, to a token bucket — callers don't change."

*[Implements incrementally, narrating extensibility decisions]*
> "I'm putting the window-check logic behind a small `RateLimitStrategy` interface rather than hardcoding the sliding-window logic directly into `RateLimiter`, specifically so a future requirement like 'support token bucket for premium users' doesn't require touching this class at all — just add a new strategy."

*[Interviewer]: "What if this needs to run across multiple servers, not just in-memory?"*
> "Then per-user state can't live in local process memory — I'd move the deque of timestamps into something like Redis with a sorted set keyed by user ID and timestamp scores, using `ZREMRANGEBYSCORE` to expire old entries and `ZCARD` to count the current window. The interface I designed doesn't change; only the implementation behind `RateLimitStrategy` does."

**Why this reads well for an Indian unicorn LLD round specifically:** the candidate designed for extensibility *before* writing code (not after), used an interface boundary to isolate the part of the design likely to change, and handled the "what if this needs to scale" follow-up as a natural extension of the existing design rather than a rewrite — exactly the signal these rounds are built to surface, and exactly what pure LC-style DSA prep does not train.

**RED FLAGS an LLD interviewer is trained to note:** jumping straight to code with no class diagram or interface discussion, a design so tightly coupled that a single new requirement forces a rewrite, or code that doesn't actually compile/pass the interviewer's tests at the end despite "looking right."

---

### Simulation 4: Meta — Two Problems, One Round, Speed as a First-Class Signal

**Interviewer:** "First problem: given a string, return true if it can become a palindrome by removing at most one character." (LC 680 — Valid Palindrome II) *"You'll have about 15 minutes, then we'll move to a second problem."*

**Candidate:**

*[Immediately states the two-pointer approach without narrating a brute-force detour — Meta's format punishes spending time on an approach you're not going to submit]*
> "Two pointers from both ends. If they mismatch, the string can only be a palindrome by skipping the left character or the right character — so I check both remaining substrings for palindrome-ness and return true if either works. That's the whole idea; let me code it directly."

*[Codes it in ~6 minutes, tests it against the mismatch case and the already-palindrome case out loud, moves on]*

> "That covers it — happy to take the next problem."

*[Interviewer moves to problem 2 with about 25 minutes left: "Given an array, find if there exists a subarray with sum divisible by k." (Related to LC 974 — Subarray Sums Divisible by K)]*

> "Prefix sum mod k, with a hash map counting how many prefixes land on each remainder — two prefixes with the same remainder mod k means the subarray between them is divisible by k. Coding now."

**Why this reads well for Meta specifically:** the candidate skipped narrating alternatives they weren't going to use, moved from insight to code in under a minute both times, and treated "finish two clean mediums" as the actual goal rather than trying to impress with a single over-explored problem — precisely the shape Section 2.4 and Mistake 4 (Section 8) describe as load-bearing at Meta.

**RED FLAGS a Meta interviewer is trained to note:** spending 10+ minutes talking through a brute force you already know is wrong, running out of time on problem 1 and never reaching problem 2, or a working solution with no tests run before declaring "done."

---

## MENTAL MODEL CHECKPOINT

Seven questions. Answer before reading.

---

**Q1: If Amazon and Google both ask a "medium" LC problem, what differs in the evaluation, not the algorithm?**

> Amazon evaluates the code AND which Leadership Principle you demonstrated while solving it — an interviewer is simultaneously scoring your BFS implementation and noting whether you showed "Dive Deep" by catching an edge case unprompted, or "Bias for Action" by committing to an approach rather than over-deliberating. Google evaluates the code against 4 fixed axes (GCA, RRK, Communication, Googleyness) with no LP overlay, and specifically leaves room in the problem for follow-ups — a "medium" at Google is really a floor, and the real evaluation happens in how far you can be pushed past it (streaming input, O(1) space, generalize to k dimensions). Same problem, structurally different rubric.

---

**Q2: Why is LC's company tag alone an unreliable signal, and what strengthens it?**

> Company tags are crowd-sourced and self-reported by candidates after their interviews, gated behind Premium, and accumulate across all-time by default — so a tag can reflect a single candidate's guess from 6 years ago sitting next to genuinely current data with equal visual weight. What strengthens the signal: filtering to the "Last 6 Months" window (loops rotate on a ~1-2 year cycle, so recent tags are far more predictive), and cross-referencing against Glassdoor/Blind reports that independently describe the same or similar problem content. Convergence across multiple independent, recent sources is the real signal — any single source in isolation is not.

---

**Q3: Name two patterns that dominate Codeforces Div 2 D/E but almost never appear in interviews. Why the divergence?**

> Segment Trees (Pattern 27) and Interval/Bitmask/Digit DP (Patterns 23-25) are staples of Div 2 D/E but rare in standard interview loops. The divergence is structural: a 45-minute interview round has room for exactly one well-explained, well-tested idea — patterns that require heavy setup (build a segment tree, define a 3-dimensional bitmask DP state) eat most of the round just on scaffolding, leaving no time to demonstrate communication and correctness under a single interviewer's real-time judgment. Contests, by contrast, allocate a fixed 2-hour budget across 5-8 problems and expect these data structures as zero-thought tools by the D/E tier, so the "difficulty budget" gets spent on novel combinations of them instead.

---

**Q4: You have 1 week before an Indian unicorn onsite that includes an LLD round. Besides DSA patterns, what MUST you additionally prepare that Amazon/Google prep doesn't cover?**

> A working, extensible, compilable object-oriented design built from scratch under time pressure — e.g., an LRU cache, a rate limiter, or a Splitwise-style expense splitter — graded on class design (SOLID principles, extensibility to plausible future requirements), not on time/space complexity. This is a fundamentally different skill from LC-style DSA: it rewards clean interfaces and the ability to reason about "what changes if requirement X is added later," which a pure algorithms-only prep track (the FAANG-majority approach) does not build at all. Section 4.3's Day 4-5 dry runs exist specifically to close this gap.

---

**Q5: Why does Meta's fast two-questions-per-round format favor certain patterns (arrays/strings/hashing) over others (heavy multi-hour DP)?**

> The format allocates roughly 15-17 minutes per problem within a 35-minute round including discussion — enough time for a pattern with a short, clean setup (hash map, two pointers, sliding window) to be fully coded, tested, and discussed twice, but not enough time for a pattern that needs 10+ minutes just to correctly state the DP recurrence and base cases before any code is written. The format is a direct constraint on which patterns are *feasible* to evaluate twice per round, and Meta's interviewer bank was built around that constraint — hence arrays/strings/hashing (Pattern 01) sits at ALWAYS while 2D/interval/tree DP sit at SOMETIMES or RARE in Section 3's table.

---

**Q6: A candidate has mastered patterns 01-21 (through LIS) but not 22-35. Which company tracks are they interview-ready for, and which are they under-prepared for?**

> They are well-prepared for Amazon, Meta, Microsoft, Apple, Indian unicorns, and startups — Section 3's table shows patterns 22-32 sitting at RARE or SOMETIMES across every one of those columns, meaning the candidate's gap barely intersects with what those loops actually test. They are under-prepared specifically for Google at the senior/L5+ level, where Patterns 20 (LCS family) and occasionally 22-24 (tree/interval/bitmask DP) show up at OFTEN/SOMETIMES rather than RARE — and they are severely under-prepared for Codeforces rating climbing past ~1900, where Patterns 22-32 are load-bearing for the D-E problem band (Section 5).

---

**Q7: What's the single biggest mistake candidates make when using "company-specific prep," and how do you avoid it?**

> Treating the company-specific tables and tag frequencies as a *replacement* for fundamentals rather than a *bias* on top of them — e.g., grinding only Amazon-tagged graph/heap problems for three weeks while never touching binary search or DP, then failing when a bar raiser (who deviates from the norm by design) asks something off-distribution. The fix: always keep Patterns 01-15 at full strength regardless of target company (they appear ALWAYS or OFTEN across every column in Section 3's table), and treat this document's tables as the last 20% allocation decision for your final 1-2 weeks of prep — not the whole plan.

---

## WHAT TO DO AFTER THIS PATTERN

1. **Run the Section 6 research protocol on your actual target company** before doing anything else in this pattern — a generic 7-day plan is a starting template, not a substitute for 30 minutes of real, current research on the specific loop you're facing.

2. **Pick the matching 7-day plan from Section 4** (or adapt it — the structure of "2-3 days pattern drilling, 1-2 days company-specific extras like LP stories or LLD, 1 day full mock, 1 day rest" generalizes to any company once you've re-weighted Day 1-3 using Section 3's table for your target).

3. **If you're also pursuing Codeforces rating, read Section 5 twice** and consciously decide how much time this week goes to interview-shaped prep (Patterns 01-21, 33-35) versus contest-shaped prep (Patterns 22-32) — trying to do both at full intensity in the same week usually means neither gets done well.

4. **Build your own personal "company research doc"** for every company you're actively interviewing with — a living note with: last-6-months LC tag list, 5-10 Glassdoor/Blind excerpts, and your own post-interview debrief after each round (which patterns actually showed up vs. what you predicted). This compounds — your prediction accuracy for a company improves every time you interview there or hear from someone who did.

5. **Move to Pattern 38 (Hard Problem Decomposition)** once you have a specific company and timeline locked in — that pattern teaches you what to do when the interviewer's problem doesn't cleanly map to any single pattern in this curriculum, which is the realistic worst case this document can't fully insulate you from.

---

## SIGN-OFF CRITERIA

### Tier 1 — Baseline meta-awareness
You can name, without looking, the top 3 favored patterns for at least 4 of the 7 company tracks in Section 2, and you understand why Section 3's table treats Patterns 22-32 as RARE for interviews despite their dominance in Codeforces (Section 5).

### Tier 2 — Research skill demonstrated
You have run the Section 6 protocol on at least one real target company and produced your own frequency read that you can defend against the baseline table — including at least 5 independent, recent (last 6 months) data points, not a single Glassdoor post.

### Tier 3 — Plan built and executed
You have completed a full 7-day targeted plan (Section 4 template or your own adaptation) for a specific upcoming interview, including at least one full-loop mock that combines coding with the company's actual behavioral/LP/culture format — not just isolated LC grinding.

### Tier 4 — Meta-skill internalized
You can take a company you've never researched, given only its engineering blog and a handful of Glassdoor reviews, and produce an accurate Section 3-style prediction within 30 minutes — and you instinctively know when a specific interviewer's question is deviating from that company's norm rather than assuming the norm is wrong.

**Sign-off threshold:** Reach Tier 3 for at least one real, upcoming interview before considering this pattern complete. This pattern has no LeetCode problem set of its own to grind — its "problem set" is the research and planning artifact you produce for your actual next interview.

---

*Pattern 37 Complete — Company-Specific Patterns*
*Next: Pattern 38 — Hard Problem Decomposition (breaking down problems that don't cleanly match any single pattern in this curriculum)*
