# 106 — Design a Search Autocomplete System

## Category: HLD Case Study

---

## What is this?

Autocomplete (a.k.a. **typeahead** or **search suggestions**) is the little dropdown that appears the instant you start typing into a search box. Type `wea` into Google and, before your finger has left the `a`, a list drops down: *weather, weather today, weather tomorrow, weather radar…* You didn't press enter. You didn't finish the word. The system read three letters and guessed the rest.

That guessing is not magic and it is not AI in the modern sense. It is a **ranked lookup of the most popular queries that begin with the letters you've typed so far**, served back to you faster than you can blink. The whole product is: *given a prefix, return the top few most-searched-for completions of that prefix, ranked by how popular they are.*

The reason it's a classic system-design interview is that it looks trivial ("just search a list of strings") and is secretly one of the most latency-brutal systems you will ever build. The answer has to arrive **before the user has typed their next character**. Miss that window and the feature is worse than useless — a suggestion that pops up after the user has already finished typing is just visual noise.

Think of it as a **card catalogue in a library that rewrites itself every night based on what everyone borrowed yesterday** — and that must find your card in the time it takes you to tap one key.

---

## Why does it matter?

Autocomplete matters in interviews because it is the cleanest possible motivation for a **specialised in-memory data structure — the trie** — and for the single most important idea in read-heavy system design: **precompute the answer, don't compute it on the request.**

Almost every candidate reaches for a database and a `LIKE 'wea%'` query. That instinct is exactly the trap. The whole interview is a controlled demolition of that instinct, replacing it with a purpose-built structure that turns "search a billion strings by prefix" into "walk three pointers." If you can explain *why the database is wrong here and the trie is right*, you've demonstrated the thing the interview is actually testing: matching the data structure to the access pattern.

It matters in real work because the pattern generalises. The moment you have **a huge read-mostly dataset, a punishing latency budget, and a query shape known in advance**, the answer is almost always the same three-move combination you'll build here: *precompute offline → serve from an in-memory structure → cache the popular answers at the edge.* Autocomplete, ad keyword matching, product search suggestions, emoji pickers, IDE code completion, and command palettes are all the same skeleton.

And it teaches the hardest lesson in latency work: **the fast path is so fast that everything around it — the network, the debounce, the cache — becomes the design.** When your lookup is O(prefix length), the microseconds are won and lost in the layers you'd normally ignore.

---

## The core idea — explained simply

### The Airport Departures Board Analogy

Imagine you run the information desk at a giant airport. All day, travellers walk up and ask half-questions:

> "Flights to **San**…?"

They haven't finished. They might mean San Francisco, San Diego, San Jose, San Juan, San Salvador. Your job is to instantly say the **five most likely** completions — and "most likely" means the ones the most people ask about. San Francisco gets asked a thousand times an hour; San Salvador twice a day. So you answer *"San Francisco, San Diego, San Jose…"* in that order, immediately, without thinking.

How can you answer *immediately*? Two bad options and one good one:

**Bad option A — search the whole flight list every time.** Someone says "San" and you run your finger down all 40,000 flights in the airport looking for ones starting with "San", then sort them by popularity. By the time you're done, the traveller has walked to the gate. This is the `LIKE 'San%'` database scan.

**Bad option B — memorise every possible answer for every possible half-word.** You could pre-write index cards: one for "S", one for "Sa", one for "San", one for "Sana"… but you'd redo all that work from scratch for each and there'd be millions of cards with nothing shared between them.

**The good option — a nested set of pigeonholes (the trie).** You build a wall of pigeonholes. The first row is one hole per starting letter: A, B, C… S. Inside the "S" hole is another little wall: Sa, Sb, Sc… Inside "Sa" is a wall for San, Sap, Sat… And — here's the trick — **pinned to the front of each pigeonhole is a small card listing the five most popular flights reachable from that point**, already sorted. When someone says "San", you don't search anything. You walk your hand S → Sa → San (three moves, regardless of how many flights exist) and read the card pinned to the "San" hole. Done. Under a second, every time.

| Analogy piece | Technical concept |
|---|---|
| The half-question ("San…") | The **prefix** the user has typed |
| Walking S → Sa → San | Traversing the **trie**, one node per character |
| A pigeonhole | A **trie node** |
| The card pinned to each hole | The **precomputed top-K list** stored on each node |
| "Most people ask about it" | Ranking by **query popularity / frequency** |
| Redoing the flight list every night | The **offline rebuild** from yesterday's query logs |
| Not re-searching for every traveller | Serving from an **in-memory, read-only structure** |

That pinned card is the entire secret. Everything else in this design exists to build those cards cheaply, keep them fresh, and copy them to enough desks that a million travellers can be answered at once.

---

## Key concepts inside this topic

We'll run the standard 5-step framework: **requirements → estimation → data structure → architecture → deep dives**, then bottlenecks. The centre of gravity is the **trie** and how to serve it at sub-100ms latency.

---

### 1. Requirements

**Clarifying questions to ask the interviewer** (say these out loud):

- "Are suggestions ranked by **global popularity**, or personalised to the user? I'll build global first and add personalization as a deep dive."
- "How many suggestions per prefix — 5? 10?"
- "How **fresh** must trending terms be? Can a term that surged this morning wait until tonight's rebuild, or do we need it in minutes?"
- "Do we support typos and multi-word queries, or exact prefixes only?"
- "What's the latency target?" — and then answer it yourself, because this is the whole game.

**Functional requirements:**

| # | Requirement | Note |
|---|---|---|
| F1 | Given a prefix, return the **top K** (5–10) completions | The one feature |
| F2 | Completions ranked by **popularity / frequency** | Not alphabetical — by how often the query is searched |
| F3 | Suggestions **update as trends change** | New/surging terms must eventually appear |
| F4 | Suggestions cover a **massive vocabulary** | Millions of distinct queries |

**Non-functional requirements** — and note that the first one *dominates the entire design*:

| # | Requirement | Why it drives everything |
|---|---|---|
| **N1** | **Latency < ~100ms, end to end** | A suggestion that arrives after the user has typed the next character is worthless. This single number forces the trie, the precomputation, the in-memory serving, and the edge cache. **This is THE requirement.** |
| N2 | Extremely **read-heavy** | Reads outnumber writes by orders of magnitude. Optimise reads at the cost of write complexity. |
| N3 | **High availability** | The search box must keep working. But — see N4 — this feature can *degrade* rather than fail. |
| N4 | Graceful degradation | If autocomplete is down, the user just doesn't see suggestions; they can still type and press enter. Never let a broken dropdown block search. |
| N5 | Fresh enough to surface trending queries | "Fresh enough" is a deliberate trade, not "real-time." |

The reason N1 is written in bold and repeated is that it is not a nice-to-have — it is the *specification*. Every later decision (why not a DB, why precompute, why in-memory, why an edge cache) is downstream of "under 100ms or it's pointless."

---

### 2. Capacity estimation

The insight that surprises people: **autocomplete fires on nearly every keystroke.** A user typing a 10-character query can generate up to ~10 separate requests to the autocomplete service — one per letter (before we add client-side debouncing, which we will). So the autocomplete QPS is a *multiple* of the search QPS, not a fraction of it.

**Assumptions:**

- 100M searches/day.
- Each search is preceded by typing that fires, say, **~5 autocomplete requests** (after debouncing trims the raw 10 keystrokes roughly in half).

**Requests per day:**

```
100M searches/day  ×  5 autocomplete requests/search
  = 500M autocomplete requests/day
```

**Average QPS:**

```
500,000,000 / 86,400 s  ≈  5,800 QPS  (average)
Peak (≈ 3× average)     ≈  17,000 QPS
```

So autocomplete runs at **several thousand to ~17k QPS** — *far more than the ~1,200 QPS of actual searches* (100M/86,400). The feature that looks like a garnish on search is generating **5× the traffic of search itself**. That's the justification, in one calculation, for why it must be blisteringly fast and heavily cached.

**Storage / memory:** the served artifact is the trie, held in RAM.

```
Vocabulary:           ~10M distinct popular queries kept
Avg query length:     ~20 chars
Raw strings:          10M × 20 B          = 200 MB
Trie node overhead + top-K lists per node ≈ a few GB
```

Order of magnitude: **a few GB in memory** — small enough to fit the whole trie on one machine, which is what makes replication cheap and sharding optional until the vocabulary explodes. (We'll revisit when it doesn't fit.)

**Cache hit insight:** query prefixes follow a brutal power law. A handful of prefixes (`wea`, `you`, `ama`, `fac`) account for a huge share of all requests. That means an edge/CDN cache can absorb the majority of traffic before it ever reaches a trie server — the single biggest lever for hitting N1 at scale.

---

### 3. The core data structure — the TRIE (prefix tree)

Forget serving for a moment. First, the structure.

A **trie** (pronounced "try", from re**trie**val; also called a prefix tree) is a tree where **each edge is labelled with a character**, and **the path from the root to a node spells out a prefix**. Words that share a prefix share the same path — they only diverge where they differ. Store `cat`, `car`, `card`, `dog`, `do`:

```
                (root)
                /    \
              c        d
              |        |
              a        o        ← "do" ends here (a word)
             / \       |
            t   r      g        ← "dog" ends here
            |   |
          (cat) |
                d
                |
              (card)
```

- `cat`, `car`, `card` all share the path `c → a`, then split.
- `card` is just `car` with one more edge — no duplicated storage of `car`.
- A node marked as a **word end** (●) means the path to it is itself a complete query. Internal nodes can be word-ends too (`do` is a word *and* a prefix of `dog`).

**Why a trie and not a database `LIKE 'pre%'`?**

| | Database `WHERE q LIKE 'pre%'` | Trie |
|---|---|---|
| How it finds matches | **Scans** rows (or walks a B-tree range) and filters | **Walks** one pointer per prefix character |
| Cost of a lookup | Grows with **vocabulary size** | O(**length of the prefix**) — independent of vocabulary size |
| Prefix of `"a"` (millions of matches) | Enormous scan | 1 pointer hop |
| Ranking top-K | `ORDER BY` a sort over all matches | Read a precomputed list (next section) |

The killer property: **a trie lookup costs O(length of the prefix), not O(size of the vocabulary).** Whether you have 10 thousand queries or 10 billion, finding the `wea` node is exactly three pointer hops. A database's `LIKE 'wea%'` has to touch every candidate row before it can rank them. At autocomplete's QPS and latency budget, that's a non-starter.

**A basic trie in JavaScript** (insert + collect-all-completions — deliberately naive, we fix it next):

```js
class TrieNode {
  constructor() {
    this.children = new Map(); // char -> TrieNode
    this.isWord = false;
    this.frequency = 0;        // how many times this query was searched
  }
}

class BasicTrie {
  constructor() {
    this.root = new TrieNode();
  }

  // Insert a query with a popularity score.
  insert(word, frequency = 1) {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
      node = node.children.get(ch);
    }
    node.isWord = true;
    node.frequency += frequency;
  }

  // Walk to the prefix node — O(prefix length).
  _findNode(prefix) {
    let node = this.root;
    for (const ch of prefix) {
      node = node.children.get(ch);
      if (!node) return null; // no query starts with this prefix
    }
    return node;
  }

  // NAIVE search: walk the ENTIRE subtree, collect every completion, then sort.
  search(prefix, k = 5) {
    const start = this._findNode(prefix);
    if (!start) return [];
    const results = [];
    const dfs = (node, path) => {                 // <-- the expensive part
      if (node.isWord) results.push({ word: path, frequency: node.frequency });
      for (const [ch, child] of node.children) dfs(child, path + ch);
    };
    dfs(start, prefix);
    results.sort((a, b) => b.frequency - a.frequency); // sort ALL matches
    return results.slice(0, k).map((r) => r.word);
  }
}
```

Correct — but that `dfs` is the problem. For a popular prefix like `a`, the subtree under it contains a huge fraction of the entire vocabulary, and we walk and sort *all of it* on every keystroke. That's the exact opposite of O(prefix length). Section 4 fixes it.

---

### 4. The critical optimization — precomputing top-K at each node

The naive trie's fatal flaw: answering a prefix means **walking the whole subtree** below it to gather and rank completions. For `a`, that subtree is millions of nodes. Doing that thousands of times per second inside a 100ms budget is impossible.

**The fix — the whole trick of this design:** on each trie node, **store the top K completions (with their frequency scores) directly, precomputed.** Instead of computing the answer at request time, you compute it once, offline, and pin it to the node — exactly the card pinned to the pigeonhole.

Now answering a prefix is two cheap steps:
1. **Walk** to the prefix node — O(prefix length).
2. **Return** that node's cached `topK` list — O(K), effectively O(1).

No subtree walk. No sort. Ever. On the request path.

**An annotated node.** Each node carries its character *and* the pre-sorted top-K completions reachable through it:

```
               ┌─────────────────────────────────────┐
   prefix "ca" │ node: 'a'   (path so far = "ca")     │
               │ topK (precomputed, pre-sorted):      │
               │   1. "cat"    freq = 9500            │
               │   2. "car"    freq = 8100            │
               │   3. "card"   freq = 3300            │
               │   4. "cards"  freq =  900            │
               │   5. "cargo"  freq =  400            │
               └─────────────────────────────────────┘
   Request for prefix "ca":  walk root→c→a  (2 hops),
   read topK, return.  No DFS, no sort.  Sub-millisecond.
```

**The trade-off — space for time.** You're duplicating: `cat` appears in the top-K list of `c`, `ca`, `cat` (and shares the `car`/`card` lists don't include it). Every ancestor node of a popular query stores that query in its list. Memory goes up — meaningfully, maybe several-fold over the bare trie. In exchange, **read latency collapses from O(subtree) to O(prefix length).**

For a system that is **overwhelmingly read-heavy (N2)** with a **hard latency ceiling (N1)**, that is unambiguously the right call. You pay memory (cheap, and it's paid once, offline) to buy latency (precious, and it's spent on every one of thousands of requests per second). This is the canonical *precompute-for-reads* trade, and autocomplete is its poster child.

**Trie with precomputed top-K + prefix search, in Node.js:**

```js
const K = 5;

class TopKNode {
  constructor() {
    this.children = new Map();      // char -> TopKNode
    this.isWord = false;
    this.frequency = 0;
    this.topK = [];                 // precomputed [{word, frequency}], sorted desc
  }
}

class AutocompleteTrie {
  constructor(k = K) {
    this.root = new TopKNode();
    this.k = k;
  }

  // ---- BUILD PHASE (offline / on rebuild) --------------------------------

  insert(word, frequency) {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new TopKNode());
      node = node.children.get(ch);
    }
    node.isWord = true;
    node.frequency += frequency;
  }

  // Bottom-up: compute each node's topK by merging its children's topK lists
  // plus the word (if any) that terminates at the node itself.
  // Runs ONCE at build time — never on the request path.
  build() {
    const compute = (node, path) => {
      const candidates = [];
      if (node.isWord) candidates.push({ word: path, frequency: node.frequency });
      for (const [ch, child] of node.children) {
        compute(child, path + ch);       // child.topK is ready after this
        for (const entry of child.topK) candidates.push(entry);
      }
      candidates.sort((a, b) => b.frequency - a.frequency);
      node.topK = candidates.slice(0, this.k);   // keep only K -> bounded memory
    };
    compute(this.root, "");
  }

  // ---- SERVE PHASE (request path) ----------------------------------------

  _findNode(prefix) {
    let node = this.root;
    for (const ch of prefix) {          // O(prefix length)
      node = node.children.get(ch);
      if (!node) return null;
    }
    return node;
  }

  // The hot path: walk to prefix, return its cached list. O(prefix length + K).
  suggest(prefix) {
    const node = this._findNode(prefix);
    if (!node) return [];
    return node.topK.map((e) => e.word);
  }
}

// --- usage -----------------------------------------------------------------
const t = new AutocompleteTrie(5);
[["cat", 9500], ["car", 8100], ["card", 3300], ["cards", 900],
 ["cargo", 400], ["dog", 7000], ["do", 5000]].forEach(([w, f]) => t.insert(w, f));
t.build();                              // precompute all topK lists, once
console.log(t.suggest("ca"));           // ["cat","car","card","cards","cargo"]
console.log(t.suggest("car"));          // ["car","card","cards","cargo"]
console.log(t.suggest("d"));            // ["dog","do"]
```

Note the shape: **all the work is in `build()`**, which runs offline. `suggest()` — the thing that runs 17,000 times a second — does nothing but walk pointers and copy a 5-element list. That asymmetry *is* the design.

---

### 5. Ranking and how the frequencies get updated

Suggestions are ranked by **past query popularity** — the `frequency` numbers above. Two questions follow: where do those numbers come from, and how do they stay current?

**Where the counts come from.** Every search (and often every accepted suggestion) is **logged**. Those logs are the raw material. A pipeline **aggregates** them — counting how many times each distinct query was searched over some window — and the resulting `(query, frequency)` pairs feed the trie build. This is a textbook **batch/stream processing** job (see topic 87): read a firehose of log lines, group by query string, sum the counts.

**How the trie gets updated — and the freshness trade-off.** You do **not** update the trie on every single query in real time. Incrementing a node and re-sorting its ancestors' top-K lists on every one of billions of daily searches would drown the write path and defeat the read optimisation. Instead you **batch**. Two broad strategies:

| Strategy | How | Pro | Con |
|---|---|---|---|
| **Full nightly rebuild** | Aggregate the last N days of logs → build a fresh trie from scratch → atomically swap it in | Dead simple; the served trie is always internally consistent; easy to reason about | Trending terms **lag** — a query that surged this morning won't appear until tonight's rebuild |
| **Incremental updates** | Periodically (every few minutes) fold recent counts into the live trie, updating affected nodes' top-K | **Fresher**; trending terms appear sooner | More complex; must update every ancestor node's list; concurrency with live reads is fiddly |

Most real systems do a **hybrid**: a stable trie rebuilt daily for the long tail, plus a lightweight fast path for surging terms (deep dive 7d). The key mental model: **the trie is a slowly-changing, read-optimised artifact.** Freshness is a knob you turn (nightly vs every-few-minutes vs a real-time side-channel), and turning it toward "fresher" costs complexity, not correctness of the common case.

A subtlety worth saying in the interview: raw frequency isn't always the best rank. Real systems blend in **recency** (decay old counts so last year's popularity fades), **time-of-day / seasonality**, and sometimes personalization. But "rank by frequency, decayed over time" is the honest default and a fine answer.

---

### 6. High-level architecture — serving the trie at scale

The unifying idea: **the trie is a read-only served artifact.** You build it offline from logs, then serve copies of it from memory behind a heavy cache. That splits the system cleanly into an **offline build pipeline** and an **online serving path**.

**Serving concerns:**

- **Shard the trie.** Once the vocabulary outgrows one machine, split it — e.g. by **first letter** (a-shard, b-shard…) or by **prefix range**. The catch: query prefixes are wildly uneven. Far more queries start with `s` than with `z` or `x`. Naive first-letter sharding hands one server a firehose and another a trickle — a **hot shard**. Fixes: shard by a **finer prefix** (`sa`, `sb`… as separate units), **group cold letters** together and split hot ones across several servers, or route by a **balanced hash of the prefix** while keeping enough of the trie co-located to answer. Balance the load, not the alphabet.
- **Replicate each shard.** Because the trie is read-only between rebuilds, replication is trivial and safe — copy the artifact to N servers behind a load balancer for read throughput and HA. No write coordination to worry about.
- **Cache aggressively.** Prefix requests are power-law distributed, so the same handful of prefixes (`wea`, `you`, `ama`) are queried constantly. Put a **CDN / edge cache** in front, keyed by prefix, with a short TTL. A huge fraction of autocomplete traffic never reaches a trie server at all — it's served from the edge in single-digit milliseconds. This is the single biggest contributor to hitting N1 at scale (see topic 59).

**The serving architecture:**

```
   ┌────────────┐   debounced keystrokes (prefix)
   │  Client    │   + client-side cache + request cancellation
   │ (browser / │──────────────┐
   │  mobile)   │               │  GET /suggest?q=wea
   └────────────┘               ▼
                        ┌───────────────────┐   most requests stop here
                        │   CDN / Edge cache│◀── power-law prefixes cached
                        │   (keyed by prefix)│    (short TTL)
                        └─────────┬─────────┘
                          cache miss │
                                     ▼
                        ┌───────────────────┐
                        │   Load balancer   │
                        └─────────┬─────────┘
                                  │  route by prefix shard
             ┌────────────────────┼────────────────────┐
             ▼                    ▼                    ▼
      ┌────────────┐       ┌────────────┐       ┌────────────┐
      │ Trie srv   │       │ Trie srv   │       │ Trie srv   │
      │ shard a–f  │×N     │ shard g–q  │×N     │ shard r–z  │×N   (replicated
      │ (in-memory)│       │ (in-memory)│       │ (in-memory)│     for HA+reads)
      └─────▲──────┘       └─────▲──────┘       └─────▲──────┘
            │                    │                    │
            └──── load fresh trie artifact on swap ───┘
                                  ▲
              ═══════════ OFFLINE BUILD PIPELINE ═══════════
                                  │  publish built trie
                        ┌───────────────────┐
                        │  Trie Builder     │  build() top-K per node
                        └─────────▲─────────┘
                                  │  (query, frequency) pairs
                        ┌───────────────────┐
                        │  Aggregator       │  MapReduce / stream job
                        │  (batch/stream 87)│  count queries
                        └─────────▲─────────┘
                                  │
                        ┌───────────────────┐
                        │  Query logs       │  every search appended here
                        └───────────────────┘
```

Read the picture as two loops running at wildly different speeds: the **online loop** (client → edge → trie server) runs millions of times a minute and touches nothing but memory; the **offline loop** (logs → aggregate → build → swap) runs once a night (plus a fast path for trends) and does all the heavy lifting. Keeping those two loops decoupled is what lets the read path be fast.

---

### 7. Deep dives

#### 7a. Client-side optimizations — where much of the latency budget is won

The fastest server call is the one you never make. Before the request leaves the browser:

- **Debouncing.** Don't fire on every keystroke. Wait ~50–100ms after the user *stops* typing, then send one request. A fast typist entering "weather" fires one request, not seven. This is what turned the "10 requests per query" nightmare of the estimation into a manageable ~5.
- **Request cancellation.** When a new key is pressed while a request is in flight, **abort** the old one — its answer is already stale (the prefix changed). Prevents wasted work and out-of-order responses painting the wrong list.
- **Client-side caching.** If the user typed `wea` then backspaced to `we` then typed `wea` again, the browser already has the answer. Cache recent prefix→results locally; also, results for `weat` are a subset you can sometimes filter from `wea` without a round trip.

```js
// Debounce + cancel-in-flight, browser-side.
let timer, controller;
function onInput(prefix) {
  clearTimeout(timer);
  timer = setTimeout(async () => {
    if (controller) controller.abort();      // cancel the stale request
    controller = new AbortController();
    try {
      const res = await fetch(`/suggest?q=${encodeURIComponent(prefix)}`,
                              { signal: controller.signal });
      render(await res.json());
    } catch (e) { if (e.name !== "AbortError") throw e; }
  }, 80);                                     // ~80ms debounce window
}
```

#### 7b. Personalization and context

Global popularity is a strong default, but real autocomplete leans on **who and where you are**: your location (a "wea" in London vs Mumbai wants a different city's weather), your search history, your language. The problem: **personalization breaks the shared cache.** A globally-cached response for `wea` can be served to everyone; a *personalized* one is unique per user and can't be cached at the edge. The usual compromise: serve the shared, cacheable global list and **lightly re-rank it per user** on the client or a thin edge function, rather than computing a bespoke list server-side. You keep most of the cache benefit and still bias toward the user.

#### 7c. Handling typos (fuzzy matching)

Users type `wheather`. A strict trie walk fails at `wh…e` and returns nothing. Fuzzy autocomplete tolerates small **edit distances** (insertions, deletions, transpositions). Approaches: a **BK-tree** or a **Levenshtein automaton** over the vocabulary, or a pragmatic hack — for a short prefix, also probe a few single-edit variants of it against the trie and merge results. It's costlier than an exact walk, so it's typically a **fallback** triggered only when the exact walk returns too few results.

#### 7d. Trending / real-time terms

The nightly rebuild can't surface a term that started surging an hour ago (breaking news, a viral moment). The fix is a **separate fast path**: a stream job (topic 87) watches recent query counts, detects sharp spikes, and pushes those hot terms into a small, frequently-updated **overlay** that the serving layer merges on top of the stable trie's results. The big trie stays slow and stable; the small overlay stays fast and fresh. This is the practical face of the "hybrid" freshness strategy from section 5.

#### 7e. The offline pipeline in more detail

The aggregator is a classic **MapReduce** (topic 87): *map* each log line to `(query, 1)`, *reduce* by summing per query, emit `(query, frequency)`. A second pass filters out queries below a frequency threshold (there's no point storing a query searched twice), applies **time decay** to age out stale popularity, and strips PII / offensive terms. The output feeds `build()`. The built artifact is versioned and rolled out to serving replicas with an **atomic swap** so no server ever serves a half-built trie.

#### 7f. Multi-language and Unicode

`children` keyed by a single `char` quietly assumes ASCII. Real queries are Unicode: accents, CJK, emoji, right-to-left scripts, and **combining characters** where one visual glyph is several code points. Key trie edges by **grapheme cluster** (user-perceived character), normalise input (NFC) so `é` and `e`+combining-accent collide correctly, and shard/route per language where the vocabularies are disjoint. CJK autocomplete is often over **pinyin/romaji input** mapping to characters — effectively a second trie in front of the first.

---

### 8. Bottlenecks, failure modes, and how to scale further

| Bottleneck / failure | Symptom | Mitigation |
|---|---|---|
| **Trie memory** for a huge vocabulary | Trie no longer fits in one machine's RAM | Prune low-frequency queries (long tail adds little value); compress with a **radix/PATRICIA trie** (merge single-child chains); shard across machines (section 6) |
| **Rebuild pipeline lagging** | Trending terms take too long to appear; the served trie is hours stale | Decouple freshness: keep the nightly rebuild for the tail, add the real-time trending overlay (7d); monitor build/publish lag as a first-class metric |
| **Hot shard** | One shard (the `s` server) is saturated while others idle | Balance by prefix load not alphabet; split hot prefixes across replicas; over-replicate the hot shard; lean harder on the edge cache for its popular prefixes |
| **Cache stampede** | A popular prefix's edge entry expires and thousands of misses hit one trie server at once | Request coalescing / single-flight at the edge; stale-while-revalidate; jittered TTLs (topic 59) |
| **Autocomplete service down** | The suggestion dropdown fails | **Degrade gracefully (N4):** the client hides the dropdown and the search box keeps working — type and press enter as normal. The feature is optional; search is not. Never let a broken typeahead block a search. |
| **Poisoned suggestions** | Offensive or manipulated queries appear as suggestions | Blocklist filtering in the offline pipeline; frequency thresholds; manual overrides; detect coordinated query-stuffing that games the counts |

The through-line of every mitigation: **protect the read path and keep it in memory.** The moment autocomplete has to touch a disk or a database on the request path, the 100ms budget is gone.

---

## Visual / Diagram description

If you have to reproduce this on a whiteboard, draw three things in this order:

1. **The trie.** Root at top, characters on edges, a couple of shared prefixes splitting (`ca` → `cat`/`car`/`card`). Mark word-end nodes with a dot. This proves you understand *why* prefix lookup is O(prefix length) — the shared paths are the whole point.

2. **One annotated node.** Blow up a single node (say the `ca` node) and draw the **precomputed top-K list** pinned to it, pre-sorted by frequency. Write "O(1) read — no subtree walk" next to it. This one box is the interview; it's where you show you know to precompute.

3. **The serving architecture** (the two-loop diagram in section 6). Client with *debounce* labelled on the arrow → **edge cache** (label "absorbs most traffic") → load balancer → **sharded, replicated in-memory trie servers**. Then, below and feeding up, the **offline pipeline**: query logs → aggregate (MapReduce) → build top-K → atomic swap into the servers. Draw the online loop and the offline loop as visibly separate speeds.

Say the sentence that ties it together: *"Build the answer offline, keep it in memory, serve it from the edge — the request path only ever walks pointers."*

---

## Real world examples

### Google Search autocomplete
The canonical example. Suggestions appear within tens of milliseconds of each keystroke, ranked by aggregated query popularity, localised by region and language, with heavy debouncing and edge caching. Google also famously *removes* suggestions (offensive, defamatory, personal) — the poisoned-suggestion problem from section 8 at planetary scale.

### Amazon / e-commerce product search
Typeahead over the product catalogue plus historical search queries. Popularity here blends query frequency with commercial signals (conversion, in-stock), and personalization (your past purchases) is stronger than in web search — a clear case of the shared-cache-vs-personalization tension from 7b.

### IDE / editor code completion (VS Code, IntelliJ)
Same trie-shaped problem, different corpus: completions are symbols in scope, ranked by relevance and recency rather than global popularity, and computed locally with a hard latency budget because it fires as you type. Proof that the pattern generalises well beyond search boxes.

### YouTube / streaming search
Autocomplete over video-title queries with a strong trending component — surging terms (a new release, a viral clip) must appear fast, which is exactly the real-time overlay fast path of 7d layered on a stable popularity trie.

---

## Trade-offs

| Decision | Option A | Option B | When to pick which |
|---|---|---|---|
| **Lookup structure** | Trie (in-memory) | Database `LIKE 'pre%'` | Trie for the real system — O(prefix) and cacheable. DB only for a tiny prototype where latency doesn't matter. |
| **When to rank** | Precompute top-K per node (build time) | Compute at request time (subtree walk) | Precompute, always, for a read-heavy system. Request-time ranking blows the latency budget on popular prefixes. |
| **Freshness** | Nightly full rebuild | Incremental / real-time overlay | Nightly for the long tail (simple, consistent). Add the overlay only for trending terms — pay complexity only where freshness matters. |
| **Sharding key** | First letter | Balanced prefix / hash | Never naive first-letter — `s` becomes a hot shard. Balance by load. |
| **Personalization** | Global shared list (cacheable) | Per-user list (uncacheable) | Serve the cacheable global list; re-rank lightly per user. Full per-user server-side lists destroy the cache and blow up cost. |
| **Space vs time** | Store top-K on every node (more RAM) | Bare trie (less RAM, slow reads) | Spend the RAM. Memory is cheap and paid once offline; latency is precious and paid on every request. |

The meta-trade-off, stated plainly: autocomplete trades **write-side simplicity and memory** for **read-side latency**. That's the correct trade for any system where reads dominate writes by orders of magnitude and the latency ceiling is unforgiving — which is exactly what N1 and N2 told you at the start.

---

## Common interview questions on this topic

### Q1: "Why not just use a database with `SELECT ... WHERE query LIKE 'wea%' ORDER BY freq DESC LIMIT 5`?"
Because its cost grows with the vocabulary, not the prefix. For a popular prefix it scans and sorts a huge number of rows on every keystroke, and there are thousands of keystrokes per second under a 100ms ceiling. A trie makes the lookup O(prefix length), independent of vocabulary size, and lets you precompute the ranking. The database is right for the tiny prototype and wrong for the real thing.

### Q2: "The prefix is 'a'. Its subtree is millions of nodes. How do you answer in under 100ms?"
You don't walk the subtree at request time. You **precompute the top-K completions and store them on the node itself** during the offline build. Answering 'a' becomes: walk one pointer to the 'a' node, read its cached 5-element list, return. O(1) after the walk. The subtree walk happens once, offline, in `build()`.

### Q3: "How do the popularity numbers get updated when trends change?"
Search queries are logged, an offline job aggregates them into `(query, frequency)` counts (a MapReduce), and the trie is rebuilt from those counts — typically nightly. You do **not** update the trie on every query in real time; that would drown the write path. For genuinely trending terms you add a separate real-time overlay that a stream job feeds and the serving layer merges on top. Freshness is a tunable trade between simple-but-laggy and fresh-but-complex.

### Q4: "You shard the trie by first letter. What goes wrong?"
Hot shards. Query prefixes are wildly uneven — far more start with 's' than 'z' — so the 's' server gets swamped while the 'z' server idles. Fix by balancing on *load*: shard on a finer prefix, group cold letters and split hot ones across replicas, or route by a balanced hash. Balance the traffic, not the alphabet.

### Q5: "The autocomplete service is completely down. What does the user experience?"
Ideally, nothing broken. The client detects the failure, hides the suggestion dropdown, and the search box keeps working — the user types and presses enter exactly as before. Autocomplete is an enhancement, not a dependency; it must **degrade gracefully** and never block a search.

### Q6: "Where is most of the latency budget actually spent, and how do you protect it?"
Once the lookup is O(prefix length), the microseconds move to the layers around it: the network round trip, whether you even make the request (debouncing kills most of them client-side), and whether the answer is already at the edge. Protect the budget by debouncing and caching on the client, caching popular prefixes at the CDN, and keeping the trie purely in memory so the request path never touches disk or a database.

---

## Practice exercise

### Build it, then measure the trick

1. **Build a working top-K trie.** Take the `AutocompleteTrie` from section 4. Feed it a real word list (grab any English word frequency list) as `(word, frequency)`. Call `build()`, then `suggest("th")`, `suggest("pre")`, `suggest("a")`.

2. **Prove the optimization matters.** Implement *both* `search()` (the naive subtree-walk from section 3) and `suggest()` (the precomputed version). Time each over 100,000 random prefixes, weighting toward short popular prefixes like `a`, `s`, `th`. Watch the naive version's time explode on short prefixes while the precomputed version stays flat. That gap — flat vs exploding — *is* the reason precomputation exists. Write down the ratio.

3. **Add debouncing.** Simulate a user typing "weather" character by character. Show that without debouncing you issue 7 requests, and with an 80ms debounce you issue 1. Compute the QPS reduction across a million users.

4. **Break the sharding.** Split your vocabulary across 26 first-letter shards and measure how many requests each shard would get given a realistic prefix distribution. Watch 's', 't', 'a' dwarf 'z', 'x', 'q'. Propose and implement one rebalancing scheme, then re-measure.

5. **Defend three decisions out loud** as if in an interview: (a) why a trie over a database, (b) why precompute top-K instead of ranking at request time, (c) why nightly rebuild instead of updating on every query. If you can say each in two sentences, you've got the topic.

---

## Quick reference cheat sheet

- **The framework:** Requirements → Estimation → Data structure (trie) → Architecture → Deep dives → Bottlenecks. Never open with the diagram.
- **THE requirement:** **latency < ~100ms.** A suggestion that arrives after the user finishes typing is worthless. Every other decision is downstream of this one number.
- **It's read-heavy *and* high-QPS:** autocomplete fires on ~every keystroke, so 100M searches/day × ~5 requests ≈ **500M requests/day ≈ 5–17k QPS — more traffic than search itself.**
- **The data structure is a TRIE:** path from root spells the prefix, shared prefixes share paths, lookup is **O(prefix length), independent of vocabulary size.** A database `LIKE 'pre%'` scans and is the wrong tool.
- **The one trick — precompute top-K on every node.** Don't walk the subtree at request time; store the sorted top-K list on the node during the offline `build()`. Reads become walk-to-node + return-cached-list. **Space for time — the right trade for read-heavy.**
- **Ranking = query popularity/frequency**, sourced from logged queries, **aggregated offline (MapReduce, topic 87)**, decayed over time.
- **Update by batching, not per-query:** nightly full rebuild (simple, laggy) vs incremental/real-time overlay (fresh, complex). Hybrid in practice.
- **Serve it read-only:** build offline → **shard** (balance by load, not by first letter — 's' is a hot shard) → **replicate** each shard → **cache popular prefixes at the CDN/edge** (topic 59), where most traffic is absorbed.
- **Client-side wins the budget:** **debounce** (~80ms), **cancel in-flight** requests, and **cache** results locally.
- **Deep dives:** typos (edit distance, as a fallback), personalization (breaks the shared cache — re-rank a shared list instead), trending (real-time overlay), Unicode (key edges by grapheme cluster).
- **When it's down, degrade gracefully:** hide the dropdown, keep the search box working. Autocomplete is optional; search is not.
- **One-liner:** *"Build the answer offline, keep it in memory, serve it from the edge — the request path only walks pointers."*

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [93 — The HLD Approach Framework](./93-hld-approach-framework.md) — the 5-step method (requirements → estimation → data structure → architecture → deep dives) this doc executes end to end; read it first, then watch it applied to typeahead |
| **Related** | [79 — Search Systems](./79-search-systems.md) — autocomplete is the front door to search; the inverted index behind full query search is the sibling structure to the trie, and near-real-time indexing is the same freshness problem |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — the CDN/edge prefix cache that absorbs most autocomplete traffic, plus TTLs, stampede protection, and request coalescing that keep a hot prefix from crushing a trie server |
| **Related** | [103 — Design Twitter / Social Feed](./103-hld-twitter.md) — the other precompute-for-reads case study; timeline fanout-on-write is the same "do the work offline so the read is cheap" move, and trending topics reuse the stream-processing fast path |
| **Related** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — why the query store and the served trie are different tools, and why a `LIKE 'pre%'` query is the wrong access pattern for this workload |
