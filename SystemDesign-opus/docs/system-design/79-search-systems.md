# 79 — Search Systems — How Search Engines Work
## Category: HLD Components

---

## What is this?

A **search system** is a piece of infrastructure that answers the question *"which documents are about this?"* — not *"which row has this exact id?"*. It's the difference between a librarian who fetches book #4471 from the shelf and a librarian who says "here are the 10 books most likely to help you, best one first."

The trick that makes it possible is a data structure called the **inverted index**: instead of storing "document → the words in it," you store "word → the documents that contain it." Every search engine you've ever used — Google, Elasticsearch, Lucene, Postgres full-text search, the search box in your IDE — is built on that one flip.

---

## Why does it matter?

Because the obvious approach is catastrophically wrong, and most engineers reach for it first:

```sql
SELECT * FROM posts WHERE body LIKE '%coffee%';
```

This query looks innocent. It is a disaster, and it's worth being precise about **why**:

1. **It cannot use an index.** A B-tree index (recall from [62 — Database Indexing]) is a *sorted* structure. Sorting only helps if you know the **beginning** of the string. `'coffee%'` can use an index — jump to the "c" section. `'%coffee%'` has a **leading wildcard**, so the match could start at byte 0 or byte 900 of the text. There is no ordering to exploit. The planner gives up and does a **sequential scan**.
2. **It reads every row and every byte.** 10 million posts × 2 KB of body text = 20 GB of substring matching per query. At ~1 GB/s that's 20 seconds. Your users are gone.
3. **It can't rank.** `LIKE` returns a boolean. A post that mentions coffee once in a footer and a post titled "The Complete Guide to Coffee" are equally "true."
4. **It can't handle typos.** `'%coffe%'` and `'%coffee%'` are unrelated strings to SQL.
5. **It can't stem.** Someone searching `running` gets nothing from a document that says `runs` or `ran`.
6. **It matches garbage.** `LIKE '%cat%'` happily matches "concatenate," "location," and "certificate."

**The real lesson:** search is not a harder version of lookup. It is a **different problem**. Lookup is exact and boolean. Search is fuzzy, ranked, and linguistic. Different problem → different data structure → different system.

**Interview angle:** the moment a design has a search box ("search tweets," "search products," "search messages"), the interviewer is checking whether you say "we'll put a full-text search index like Elasticsearch alongside Postgres, kept in sync asynchronously." If you say `LIKE`, the round is effectively over.

**Work angle:** this is the single most common "we shipped it, it worked in staging, it fell over in production" bug. `LIKE` on 10k rows is fine. `LIKE` on 10M rows takes the database down and every other query with it.

---

## The core idea — explained simply

### The Textbook Index Analogy

Pick up a thick textbook. It has two very different navigation aids.

**The table of contents** (front of the book) maps **chapter → topics**:

```
Chapter 3 ......... Photosynthesis, chlorophyll, sunlight
Chapter 7 ......... Mitochondria, ATP, respiration
```

This is a **forward index**. Given a chapter, you learn what's in it. Useful for reading. Useless for finding.

**The index** (back of the book) maps **topic → pages**:

```
ATP ............... 141, 142, 156
chlorophyll ....... 62, 65
mitochondria ...... 140, 141, 158
sunlight .......... 61, 62, 70
```

This is an **inverted index**. Given a word, you instantly get the pages. Nobody who wants to find "mitochondria" reads all 400 pages checking each one — that's `LIKE '%mitochondria%'`. You flip to the back, look up one word alphabetically, and jump straight to pages 140, 141, 158.

Now notice something else about the back-of-book index: it's **hand-curated for search**. It doesn't list "the," "and," "is" — those appear on every page and tell you nothing. It merges "mitochondrion" and "mitochondria" into one entry. It says *see also: respiration*. Those three human editorial decisions are exactly **stop-word removal**, **stemming**, and **synonyms** — the three stages of a real indexing pipeline.

| Book | Search system |
|------|---------------|
| Table of contents (chapter → topics) | **Forward index** — doc → terms. What your Postgres row already is. |
| Back-of-book index (topic → pages) | **Inverted index** — term → doc list. The core search structure. |
| Page numbers under a word | **Posting list** — the docs containing that term |
| Skipping "the", "and", "is" | **Stop-word removal** |
| One entry for mitochondria/mitochondrion | **Stemming** |
| "see also: respiration" | **Synonym expansion** |
| Bolded page = main discussion | **Relevance score** (TF-IDF / BM25) |
| A multi-volume encyclopedia, one index per volume | **Shards**, each with its own inverted index |

The whole doc from here is just: build that back-of-book index, keep it fresh, and rank the pages it gives you.

---

## Key concepts inside this topic

### 1. Forward index vs inverted index — the flip

Three tiny documents:

```
D1: "the quick brown fox"
D2: "the quick red fox jumps"
D3: "brown bears jump quick"
```

**Forward index** (what a normal database table is — doc → its words):

```
┌──────┬───────────────────────────────────┐
│ D1   │ the, quick, brown, fox            │
│ D2   │ the, quick, red, fox, jumps       │
│ D3   │ brown, bears, jump, quick         │
└──────┴───────────────────────────────────┘
```

To answer "who mentions *brown*?" you must open **every** document and scan its word list. O(number of docs). That is `LIKE '%brown%'`.

**Inverted index** — flip the arrow (word → the docs):

```
┌──────────┬──────────────────┐
│  TERM    │  POSTING LIST    │
├──────────┼──────────────────┤
│ bears    │ [D3]             │
│ brown    │ [D1, D3]         │
│ fox      │ [D1, D2]         │
│ jump     │ [D2, D3]         │   ← "jumps" stemmed to "jump"
│ quick    │ [D1, D2, D3]     │
│ red      │ [D2]             │
│ the      │ (dropped)        │   ← stop word
└──────────┴──────────────────┘
```

Now "who mentions *brown*?" is **one lookup** in a hash map or one binary search in a sorted term dictionary: `[D1, D3]`. O(1)-ish, independent of corpus size. That is the entire magic. Everything else in this document is refinement.

**"quick AND fox"** becomes a **set intersection**: `{D1,D2,D3} ∩ {D1,D2}` = `{D1, D2}`.
**"red OR bears"** becomes a **union**: `{D2} ∪ {D3}` = `{D2, D3}`.

Real engines keep posting lists **sorted by docId** precisely so intersection is a linear merge (walk two sorted lists with two pointers), not a hash build — and so they can be delta-encoded and compressed.

### 2. What's actually inside a posting list

A posting list is richer than a bare list of ids. The full form is:

```
term → [ { docId, termFreq, positions[] }, ... ]

"fox" → [ { docId: 1, tf: 1, pos: [3] },
          { docId: 2, tf: 1, pos: [3] } ]

"quick" → [ { docId: 1, tf: 1, pos: [1] },
            { docId: 2, tf: 1, pos: [1] },
            { docId: 3, tf: 1, pos: [3] } ]
```

- **docId** — which document. Needed to return the result.
- **termFreq (tf)** — how many times the term appears in that doc. Needed to **rank** (see section 4).
- **positions** — the word offsets within the doc. Needed for **phrase queries**.

**Why positions matter.** Search for the exact phrase `"quick brown"`. Term-level intersection gives you D1 and D3 (both contain *quick* and *brown*). But in D3 — "brown bears jump quick" — the words are nowhere near each other and in the wrong order. Only positions can tell you that:

```
D1: quick@1, brown@2   → position of brown = position of quick + 1  ✓ phrase match
D3: quick@3, brown@0   → not adjacent, wrong order                  ✗ reject
```

Positions cost storage (often the single biggest chunk of an index), which is why engines let you turn them off per field if you never do phrase search on it.

### 3. The indexing pipeline — and why the query must go through it too

Raw text does not go into the index. It goes through an **analyzer**: a pipeline of transformations. Worked example on one sentence:

```
INPUT:  "The Runners are running <b>quickly</b>, and they ran!"

① Character filtering   strip HTML, normalize unicode/accents
   → "The Runners are running quickly, and they ran!"

② Tokenization          split on non-letters into tokens
   → ["The", "Runners", "are", "running", "quickly", "and", "they", "ran"]

③ Lowercasing           so "Fox" and "fox" are the same term
   → ["the", "runners", "are", "running", "quickly", "and", "they", "ran"]

④ Stop-word removal     drop ultra-common, meaning-free words
   → ["runners", "running", "quickly", "ran"]

⑤ Stemming/lemmatizing  reduce to a root form
   → ["runner", "run", "quick", "run"]

⑥ Synonyms (optional)   expand: "quick" → also index "fast"
   → ["runner", "run", "quick", "fast", "run"]
```

Two things to be precise about:

- **Stemming vs lemmatization.** *Stemming* is crude, rule-based chopping (Porter stemmer: `running → run`, but also `university → univers`). It's fast and language-specific. *Lemmatization* uses a dictionary to find the real base word (`ran → run`, `better → good`). It's slower and more accurate. Most engines default to stemming; it's good enough.
- **THE QUERY MUST USE THE SAME PIPELINE.** This is the number-one beginner bug. If you index `"running"` as the term `run`, but at query time you look up the literal string `"running"`, you get **zero results** — the index doesn't contain that string anymore. Indexing and querying must be **symmetric**. (The one common exception: synonym expansion is often applied at *query* time only, so you can change your synonym list without reindexing 200 GB.)

### 4. Build a real inverted index in JavaScript

Nothing here is a toy in spirit — this is genuinely how Lucene starts.

```javascript
// analyzer.js — the pipeline from section 3. Used for BOTH indexing and querying.

const STOP_WORDS = new Set([
  'the', 'a', 'an', 'and', 'or', 'is', 'are', 'was', 'were',
  'of', 'to', 'in', 'it', 'that', 'this', 'they', 'for', 'on'
]);

// A deliberately tiny stemmer. Real systems use Porter/Snowball (see the
// `natural` or `snowball-stemmers` npm packages) — the point is the SHAPE.
function stem(token) {
  if (token.endsWith('ies') && token.length > 4) return token.slice(0, -3) + 'y';
  if (token.endsWith('ing') && token.length > 5) return token.slice(0, -3);
  if (token.endsWith('ers') && token.length > 4) return token.slice(0, -3);
  if (token.endsWith('es')  && token.length > 4) return token.slice(0, -2);
  if (token.endsWith('s')   && token.length > 3) return token.slice(0, -1);
  return token;
}

export function analyze(text) {
  return text
    .replace(/<[^>]+>/g, ' ')          // ① character filter: strip tags
    .normalize('NFKD')                 //    normalize accents (café -> cafe)
    .replace(/[̀-ͯ]/g, '')
    .toLowerCase()                     // ③ lowercase
    .split(/[^a-z0-9]+/)               // ② tokenize on non-alphanumerics
    .filter(Boolean)
    .filter((t) => !STOP_WORDS.has(t)) // ④ stop words
    .map(stem);                        // ⑤ stemming
}
```

```javascript
// inverted-index.js
import { analyze } from './analyzer.js';

export class InvertedIndex {
  constructor() {
    // term -> Map<docId, { tf, positions }>   (the posting list)
    this.postings = new Map();
    // docId -> { text, length }               (forward info, needed for scoring)
    this.docs = new Map();
  }

  index(docId, text) {
    const tokens = analyze(text);
    // Store the doc length now — BM25 needs it later, and recomputing means
    // re-analyzing the whole document.
    this.docs.set(docId, { text, length: tokens.length });

    tokens.forEach((term, position) => {
      if (!this.postings.has(term)) this.postings.set(term, new Map());
      const list = this.postings.get(term);
      const entry = list.get(docId) ?? { tf: 0, positions: [] };
      entry.tf += 1;
      entry.positions.push(position);
      list.set(docId, entry);
    });
  }

  /** Docs containing a single term. The one O(1) lookup the whole thing exists for. */
  docsFor(term) {
    return new Set(this.postings.get(term)?.keys() ?? []);
  }

  /** mode: 'AND' (all terms must appear) or 'OR' (any term). */
  search(query, mode = 'AND') {
    const terms = analyze(query);        // ← SAME pipeline as index(). Non-negotiable.
    if (terms.length === 0) return [];

    const sets = terms.map((t) => this.docsFor(t));

    let result;
    if (mode === 'AND') {
      // Intersect smallest-first: cheapest way to shrink the candidate set fast.
      sets.sort((a, b) => a.size - b.size);
      result = sets.reduce((acc, s) => new Set([...acc].filter((d) => s.has(d))));
    } else {
      result = sets.reduce((acc, s) => new Set([...acc, ...s]), new Set());
    }
    return [...result];
  }

  /** Exact phrase: terms must be adjacent, in order. This is why we stored positions. */
  searchPhrase(phrase) {
    const terms = analyze(phrase);
    return this.search(phrase, 'AND').filter((docId) => {
      const starts = this.postings.get(terms[0]).get(docId).positions;
      return starts.some((start) =>
        terms.every((term, i) =>
          this.postings.get(term).get(docId).positions.includes(start + i)
        )
      );
    });
  }
}
```

```javascript
// demo.js
import { InvertedIndex } from './inverted-index.js';

const idx = new InvertedIndex();
idx.index(1, 'The quick brown fox');
idx.index(2, 'The quick red fox jumps over the log');
idx.index(3, 'Brown bears jump quick and eat fish');

console.log(idx.search('quick fox', 'AND'));   // [1, 2]
console.log(idx.search('bears fox', 'OR'));    // [1, 2, 3]
console.log(idx.search('running foxes'));      // [] — no doc has "run"
console.log(idx.searchPhrase('quick brown'));  // [1]   (NOT 3 — words not adjacent)
```

Run it. `searchPhrase('quick brown')` returning `[1]` and not `[3]` is the moment positions earn their storage cost.

### 5. Relevance scoring — TF-IDF, with the arithmetic

`search()` above returns a **set**. Users need a **ranked list**. The oldest good answer is **TF-IDF**, and it's two intuitions multiplied together.

**TF — term frequency.** *How much is THIS document about the word?* A doc that says "coffee" 8 times is more about coffee than one that says it once.

**IDF — inverse document frequency.** *How rare is the word across ALL documents?* If a term appears in every document it carries no information — matching it tells you nothing. If it appears in 3 documents out of a million, matching it is a huge signal.

```
idf(term) = log( totalDocs / docsContaining(term) )
```

Worked numbers, corpus of **1,000,000** documents:

| Term | Docs containing it | idf = log(1e6 / df) | Meaning |
|------|--------------------|---------------------|---------|
| `the` | 1,000,000 | log(1) = **0.0** | worthless — scores *nothing* |
| `coffee` | 50,000 | log(20) ≈ **3.0** | moderately informative |
| `quokka` | 200 | log(5000) ≈ **8.5** | extremely informative |

That's the whole point of IDF: **it automatically kills stop words** without a stop-word list, because a word in every document gets `log(1) = 0`. And it makes the rare word dominate — for the query `the quokka`, essentially 100% of the score comes from `quokka`, which is exactly right.

Now score the query **"coffee quokka"** against two documents (assume both are 200 words long):

```
Doc A: mentions "coffee" 10 times, "quokka" 0 times
  score = tf(coffee)=10 × idf=3.0  +  tf(quokka)=0 × idf=8.5
        = 30.0 + 0
        = 30.0

Doc B: mentions "coffee" 2 times, "quokka" 3 times
  score = 2 × 3.0  +  3 × 8.5
        = 6.0 + 25.5
        = 31.5     ← WINS
```

Doc B wins despite mentioning coffee far less, because it contains the rare term. That is relevance ranking working correctly.

### 6. BM25 — what it fixes, and why it's the default everywhere

TF-IDF has two real bugs. **BM25** (Best Match 25, from the Okapi project) fixes both, and it is what Elasticsearch and Lucene use by default today.

**Bug 1: term frequency is unbounded and linear.** In TF-IDF, a document containing "coffee" 50 times scores 50× one containing it once. But that's not how relevance works. Going from 1 mention → 5 mentions is a *massive* signal. Going from 45 → 50 tells you almost nothing new — and it's exactly what keyword-stuffing spam does.

BM25 applies **saturation**. The term-frequency contribution grows fast at first, then flattens toward a ceiling:

```
tf component = (tf × (k1 + 1)) / (tf + k1)        with k1 ≈ 1.2

  tf=1  → (1 × 2.2) / (1 + 1.2)  = 1.00
  tf=2  → (2 × 2.2) / (2 + 1.2)  = 1.38
  tf=5  → (5 × 2.2) / (5 + 1.2)  = 1.77
  tf=20 → (20 × 2.2)/(20 + 1.2)  = 2.08
  tf=50 → (50 × 2.2)/(50 + 1.2)  = 2.15   ← barely moved from tf=20

Score
 2.2 ┤                    ····························  ← asymptote at k1+1
     │            ·····
 1.5 ┤      ····
     │   ··
 0.5 ┤ ·
     └─┬────┬────┬────┬────┬────┬────► term frequency
       1    5   10   20   30   50
```

Diminishing returns, exactly as intuition demands. The 50th "coffee" is worth almost nothing.

**Bug 2: long documents win unfairly.** A 10,000-word article will naturally contain "coffee" more times than a 100-word one, just by being long — not by being more *about* coffee. Raw TF rewards verbosity.

BM25 adds **length normalization**: it divides by how long the document is *relative to the average document* in the corpus.

```
                                    tf × (k1 + 1)
score(term, doc) = idf(term) × ─────────────────────────────────
                               tf + k1 × (1 - b + b × (dl / avgdl))

  dl    = this document's length in terms
  avgdl = average document length in the corpus
  k1 ≈ 1.2  → how fast TF saturates
  b  ≈ 0.75 → how hard to punish long docs (b=0 → no normalization, b=1 → full)
```

Read the denominator: if `dl = avgdl` the length factor is 1 and nothing changes. If the doc is **twice** the average length, the denominator grows, and the score **shrinks** — you need proportionally more mentions to earn the same score. A 100-word doc mentioning coffee 3 times outranks a 10,000-word doc mentioning it 5 times. Correct.

**In one line:** BM25 = TF-IDF + "extra mentions stop helping" + "being long doesn't count as being relevant." That's the whole upgrade, and it's why nobody writes new systems on plain TF-IDF.

### 7. Elasticsearch / Lucene architecture, conceptually

**Lucene** is a Java *library* that implements everything above: analyzers, inverted index, BM25. **Elasticsearch** is a distributed *system* wrapped around many Lucene instances. Four ideas do all the work.

**a) Shards.** An Elasticsearch **index** (logical: "all products") is split into N **shards**. Each shard is a **complete, self-contained Lucene index** with its own term dictionary and its own posting lists. Shard = the unit of horizontal scale — same idea as [64 — Database Sharding], applied to search. A document is routed to a shard by `hash(docId) % numShards`.

> **Gotcha worth knowing:** the number of primary shards is **fixed at index-creation time**, because changing it changes the hash routing for every document. Growing means creating a new index and reindexing. Choose shard count with your 2-year size in mind (rule of thumb: keep a shard in the 10–50 GB range).

**b) Replicas.** Each shard has R **replica** copies on other nodes. Replicas give you two things: **HA** (a node dies, a replica is promoted to primary) and **read throughput** (queries can be served by primary *or* any replica, so replicas multiply your query capacity).

**c) Scatter-gather.** A query touches **every shard**, because any shard could hold a matching document — there's no way to know in advance which shard has documents about "coffee."

```
1. Client sends "coffee" to any node → that node becomes the COORDINATOR.
2. Coordinator scatters the query to all N shards (one copy of each).
3. Each shard searches its OWN inverted index, scores with BM25,
   returns its top 10 (just docIds + scores — cheap).
4. Coordinator merges N×10 results, re-sorts globally, keeps top 10.
5. Coordinator FETCHES the full documents for those 10 winners only.
```

The two-phase "query then fetch" matters: step 3 ships only ids and floats across the network, not documents. You only pay to move the 10 documents you actually return.

**The tail-latency cost:** the query completes only when the **slowest** shard replies. If one shard is on a node doing garbage collection, your p99 is that node's GC pause. **More shards = more parallelism but more chances to hit a slow one.** This is why "just add shards" is not free, and why p99 latency in a scatter-gather system is always much worse than p50.

**d) Segments, immutability, and "near real-time."** Here is the part that surprises everyone.

A Lucene shard is not one index file. It's a set of **segments** — and **segments are immutable**. Once written, a segment file is never modified. Consequences:

- **A new document** goes into an in-memory buffer, which is flushed to a **brand-new segment**. It is not searchable until that flush happens. The flush ("refresh") runs on a timer — **default every 1 second**. That's exactly why Elasticsearch is called **near real-time**: write a doc, and for up to ~1s a search won't find it. Not a bug. A design choice.
- **A delete** does not remove anything. It writes the docId into a **tombstone** (a "deleted" bitmap). The document still sits in its segment, taking up space; it's just filtered out of results at query time.
- **An update** is a delete + a re-index into a new segment. There is no in-place edit. Ever.
- **Background merges** periodically combine many small segments into fewer big ones, and *that's* when tombstoned documents are finally physically dropped and space is reclaimed. Merges are heavy I/O — a common cause of mysterious latency spikes.

Why immutable? Because an immutable file needs **no locks** (any number of readers, no writers), can be **aggressively cached** by the OS page cache and never invalidated, and can be **compressed hard** since it will never be patched. You trade instant visibility for enormous read throughput. That is the deal.

### 8. Beyond keyword matching

Keyword search over an inverted index is the foundation. Four things get layered on top:

- **Fuzzy matching (typos).** *Edit distance* (Levenshtein) = the minimum number of insert/delete/substitute operations to turn one word into another. `coffe → coffee` is distance 1. Lucene compiles a Levenshtein automaton over the term dictionary so it can find all terms within distance 1–2 without comparing against every term. Rule: allow distance 1 for short words, 2 for long ones, 0 for very short ones — otherwise `cat` fuzzy-matches `car`, `bat`, `can`, and `cot`.
- **Autocomplete.** A different structure entirely — you're doing **prefix** matching, not term matching, and a **trie** (prefix tree) is the natural fit; see [106 — Design Search Autocomplete].
- **Faceted search.** The "Brand: Sony (42) | Price: under $50 (17)" sidebar. Implemented with a **doc values** column store (docId → field value) instead of the inverted index — it's an aggregation over the matching set, not a lookup.
- **Vector / semantic search.** Turn the query and every document into an **embedding** — a vector of floats where semantic meaning is geometric closeness — and retrieve by nearest neighbour. This finds a document about "espresso machines" for the query "coffee gear" even though *zero* words overlap, which keyword search fundamentally cannot do. Exact nearest-neighbour over millions of vectors is too slow, so systems use **ANN** (approximate nearest neighbour, e.g. HNSW graphs) to trade a little recall for 100× speed. Modern production search is increasingly **hybrid**: run BM25 and vector search in parallel, then blend the two ranked lists.

### 9. The architecture rule: search is a DERIVED read model

**Never make your search index the primary store.** This is the most important operational sentence in the document.

Elasticsearch is a *derived, rebuildable* view of data whose truth lives elsewhere (Postgres). It has no transactions, no foreign keys, no real constraints, and its documents can be silently lost during a split-brain or a bad reindex. You keep the source of truth in a database that guarantees durability, and you **project** it into the search index asynchronously.

The test: *"If Elasticsearch's disks were wiped right now, could I rebuild the whole thing from Postgres in a few hours?"* If yes, your architecture is correct. If no, you have put irreplaceable data in a cache and you will lose it.

---

## Visual / Diagram description

### Diagram 1: The forward → inverted flip

```
     FORWARD INDEX                          INVERTED INDEX
   (doc → its words)                      (word → its docs)

┌──────┬──────────────────┐            ┌──────────┬───────────────┐
│ D1   │ quick brown fox  │            │ bears    │ [D3]          │
├──────┼──────────────────┤            ├──────────┼───────────────┤
│ D2   │ quick red fox    │  ── flip ▶ │ brown    │ [D1, D3]      │
│      │ jumps            │            ├──────────┼───────────────┤
├──────┼──────────────────┤            │ fox      │ [D1, D2]      │
│ D3   │ brown bears jump │            ├──────────┼───────────────┤
│      │ quick            │            │ jump     │ [D2, D3]      │
└──────┴──────────────────┘            ├──────────┼───────────────┤
                                       │ quick    │ [D1, D2, D3]  │
  "find brown" = open EVERY            ├──────────┼───────────────┤
  doc and scan it  (O(N))              │ red      │ [D2]          │
                                       └──────────┴───────────────┘

                                         "find brown" = ONE lookup
                                          → [D1, D3]   (O(1))
```

### Diagram 2: Scatter-gather across shards

```
                       ┌──────────────┐
                       │    Client    │  q = "coffee"
                       └──────┬───────┘
                              │
                    ┌─────────▼──────────┐
                    │  COORDINATING NODE │  ── merges + re-ranks
                    └───┬──────┬──────┬──┘
              scatter   │      │      │   (query hits EVERY shard)
              ┌─────────┘      │      └─────────┐
              ▼                ▼                ▼
      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
      │   SHARD 0    │ │   SHARD 1    │ │   SHARD 2    │
      │ (Lucene idx) │ │ (Lucene idx) │ │ (Lucene idx) │
      │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │
      │ │ segments │ │ │ │ segments │ │ │ │ segments │ │
      │ │ immutable│ │ │ │ immutable│ │ │ │ immutable│ │
      │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │
      │  + replicas  │ │  + replicas  │ │  + replicas  │
      └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
             │ top10           │ top10          │ top10
             │ (ids+scores)    │                │
             └────────┬────────┴────────┬───────┘
                      ▼                 ▼
                 gather → merge 30 → global top 10 → fetch docs
                      │
              ⚠ as slow as the SLOWEST shard  (tail latency)
```

### Diagram 3: The correct production architecture

```
                      ┌──────────────┐
                      │   Clients    │
                      └───┬──────┬───┘
             writes       │      │       searches
                          ▼      ▼
                    ┌──────────────────┐
                    │    API Service   │
                    └───┬──────────┬───┘
                        │          └─────────────────┐
                        ▼                            ▼
            ┌────────────────────┐          ┌─────────────────┐
            │     POSTGRES       │          │  ELASTICSEARCH  │
            │  SOURCE OF TRUTH   │          │  DERIVED READ   │
            │                    │          │     MODEL       │
            │ • ACID             │          │ • inverted idx  │
            │ • constraints      │          │ • BM25 ranking  │
            │ • durable          │          │ • REBUILDABLE   │
            └─────────┬──────────┘          └────────▲────────┘
                      │                              │
                      │ CDC / outbox                 │ bulk upsert
                      ▼                              │
            ┌────────────────────┐        ┌──────────┴─────────┐
            │   KAFKA / QUEUE    │───────▶│   Indexer Worker   │
            │ (post.created,     │        │ • analyze text     │
            │  post.updated,     │        │ • batch 1000 docs  │
            │  post.deleted)     │        │ • retry on failure │
            └────────────────────┘        └────────────────────┘

     RULE: arrows into Elasticsearch, never out of it into Postgres.
     Search returns docIds → hydrate full objects from Postgres if needed.
     Wipe Elasticsearch → replay from Postgres → fully recovered.
```

**What this shows:** writes go to Postgres and *only* Postgres. The write emits an event (via Debezium-style **CDC** — change data capture reading the DB's write-ahead log — or a transactional **outbox** table). A worker consumes it, runs the analyzer, and bulk-upserts into Elasticsearch. Search queries hit Elasticsearch, get back ranked docIds, and (optionally) hydrate the full rows from Postgres so users never see stale field values. The index is **eventually consistent** — typically a few hundred ms to a few seconds behind — and that is an accepted, documented property, not a bug.

---

## Real world examples

### 1. GitHub — code search

GitHub publicly documented rebuilding code search (project "Blackbird") on a custom Rust engine rather than Elasticsearch, because code has properties normal text doesn't: you must match punctuation and symbols exactly, and you can't stem identifiers. They index **n-grams** (specifically trigram-style sub-strings) rather than whitespace-delimited words, so that a search for `.map(` or a substring inside `getUserById` still hits an index instead of a scan. The takeaway isn't "don't use Elasticsearch" — it's that **the analyzer is the design**, and code needs a different one from prose.

### 2. Wikipedia — CirrusSearch

Wikipedia's search runs on Elasticsearch (their extension is called CirrusSearch). MySQL remains the source of truth for article wikitext; Elasticsearch is a derived index rebuilt from it, exactly the pattern in Diagram 3. Their relevance is BM25 plus signals like incoming-link count and page popularity — a good demonstration that production ranking is *BM25 as a base* with domain signals layered on, never BM25 alone.

### 3. E-commerce search (representative architecture)

A large product catalog typically runs the hybrid stack described above: the product table lives in Postgres/MySQL; a CDC pipeline projects it into Elasticsearch; the query does BM25 over title/description/brand, **boosts** the title field (a match in the title is worth ~3× a match in the body), applies filters (in-stock, price range) as non-scoring boolean clauses, computes facets from doc values, and then re-ranks the top 100 with a machine-learned model using click-through and conversion data. Note the shape: **cheap index retrieval narrows millions → 100, expensive ML ranks the 100.** That two-stage funnel is universal.

---

## Trade-offs

| Aspect | What you gain | What it costs |
|--------|---------------|---------------|
| **Inverted index** | O(1) term lookup instead of a full scan; ranking becomes possible | Index is often **as large as or larger than** the source data (positions + term dictionary). Budget 0.5×–1.5× your text size. |
| **Analyzer (stemming, stop words)** | Users find things with natural language; "running" matches "run" | Lossy and irreversible. Change the analyzer → you must **reindex everything**. |
| **Immutable segments** | Lock-free reads, great OS cache behaviour, heavy compression | Deletes leave tombstones (wasted space until merge); background merges cause I/O spikes; near-real-time (~1s), not instant. |
| **Sharding** | Horizontal scale, parallel query execution | Every query hits every shard → **tail latency = slowest shard**. Shard count is fixed at creation. |
| **Replicas** | HA + more read throughput | Multiplies storage and write cost (every doc written R+1 times). |
| **Separate search store** | Right tool for the job; search load can't take down your primary DB | **Eventual consistency** (index lags writes by ~1s+), a whole extra distributed system to operate, and a sync pipeline that *will* break. |
| **Vector/semantic search** | Finds meaning, survives zero keyword overlap | Embedding compute cost, big RAM footprint for ANN graphs, and results are harder to debug ("why did *that* rank first?"). |

**Rule of thumb:**
- **< ~100k rows, simple needs?** Use Postgres full-text search (`tsvector` + GIN index). It's a real inverted index, it's transactional, and it's zero extra infrastructure. Do not stand up Elasticsearch for a blog.
- **Millions of docs, relevance matters, facets, typo tolerance?** Elasticsearch/OpenSearch — as a **derived read model**, synced from your source of truth via a queue or CDC.
- **Always:** the source of truth is your database. The search index is a cache you can rebuild. If losing your search cluster loses data, you designed it wrong.

---

## Common interview questions on this topic

### Q1: "Why can't you just use `LIKE '%term%'` for search?"
**Hint:** Leading wildcard ⇒ no B-tree index can be used (a B-tree is sorted by prefix, and you don't know the prefix) ⇒ full sequential scan of every row and every byte. Then add: it can't rank by relevance, can't stem, can't handle typos, and matches substrings inside unrelated words ("cat" in "location"). Search is a different problem from lookup, so it needs a different data structure: the inverted index.

### Q2: "Explain the inverted index. What's in a posting list?"
**Hint:** Forward index = doc → terms. Inverted = term → posting list. A posting is `{docId, termFrequency, positions[]}`. Draw the 3-document example. docId to return the doc; termFrequency to score it; positions to support phrase queries ("quick brown" must be adjacent — otherwise "brown bears jump quick" would falsely match). Mention posting lists are sorted by docId so AND is a linear two-pointer merge.

### Q3: "What is BM25 and why is it better than TF-IDF?"
**Hint:** Both = (how often the term is in this doc) × (how rare the term is overall). BM25 fixes two flaws: **saturation** — TF contribution flattens out (k1≈1.2), so the 50th "coffee" adds almost nothing and keyword stuffing stops working; and **length normalization** (b≈0.75) — dividing by `dl/avgdl` so a 10,000-word article doesn't outrank a tight 100-word one purely by being long. Give the IDF intuition: `log(N/df)` makes "the" score 0 automatically.

### Q4: "How does a distributed search query actually execute?"
**Hint:** Scatter-gather. Index → shards (each a self-contained Lucene index with its own inverted index) → each shard has replicas. The coordinating node scatters the query to one copy of every shard; each returns its **local top N as ids + scores only**; the coordinator merges, re-sorts globally, takes the top N, and only *then* fetches the full documents (query-then-fetch). Then volunteer the cost: **latency = the slowest shard**, so p99 degrades as shard count grows.

### Q5: "Why is Elasticsearch 'near real-time'? Why not instant?"
**Hint:** Segments are immutable. A new doc sits in an in-memory buffer and isn't searchable until a **refresh** writes a new segment — default every **1 second**. Deletes are tombstones, not removals; updates are delete + re-insert; background merges compact segments and reclaim tombstoned space. Immutability buys lock-free reads, page-cache friendliness, and heavy compression — the ~1s visibility lag is the price.

### Q6: "Your product search must never show a deleted product. How?"
**Hint:** This exposes eventual consistency. Options: (a) filter deleted ids out of results at read time using the source of truth — hydrate the top N from Postgres and drop anything gone; (b) delete synchronously from ES on the write path *before* returning; (c) accept a ~1s window. Best answer usually combines (a) with async indexing: search gives you *candidates*, the database gives you *truth*.

---

## Practice exercise

### Build and rank a mini search engine

**Time: ~35 minutes. Produce a single `search.js` you can run with `node search.js`.**

Start from the `InvertedIndex` class in section 4 and index these five documents:

```
D1: "The quick brown fox jumps over the lazy dog"
D2: "A quick brown dog outpaces a quick fox"
D3: "The lazy dog sleeps all day in the sun"
D4: "Foxes are quick quick quick quick quick quick quick animals"
D5: "Dogs and foxes are both quick, agile, clever animals that hunt and run
     across fields, forests, and open plains at remarkable speed every day"
```

**Part A — build.** Print the full inverted index (term → docIds). Verify `quick` appears in all five and that `the`/`a` are gone.

**Part B — score.** Implement `scoreTFIDF(query)` returning docs sorted by score descending. Run it for the query `quick fox`. **D4 will win** — it stuffed "quick" seven times. Note that it is a bad result.

**Part C — fix it.** Implement `scoreBM25(query)` with `k1 = 1.2`, `b = 0.75`. You'll need `avgdl` (average doc length across the corpus) and each doc's `dl`, both of which the class already tracks. Re-run `quick fox`.

**Answer these three questions in comments at the bottom of the file:**
1. Did D4 drop in the BM25 ranking? Which mechanism did it — saturation, length normalization, or both?
2. D5 is long and mentions "quick" only once. How does `b` change its position? Re-run with `b = 0` and `b = 1` and report the difference.
3. What is the IDF of `quick` in this 5-doc corpus? Explain the number you get and what it implies for the query `quick fox`.

**Stretch:** add `searchPhrase("quick brown")` and confirm it returns D1 and D2 but never D4.

---

## Quick reference cheat sheet

- **`LIKE '%x%'` cannot use an index** — leading wildcard defeats B-tree ordering → full table scan. Never ship it as search.
- **Forward index** = doc → terms. **Inverted index** = term → docs. The flip is the entire idea.
- **Posting list** = `[{docId, termFreq, positions[]}]`. docId to return, tf to rank, positions for phrase queries.
- **Analyzer pipeline** = char filter → tokenize → lowercase → stop words → stem → synonyms. **Run the identical pipeline on the query**, or nothing matches.
- **TF-IDF** = how often in this doc × how rare across all docs. `idf = log(N/df)` → "the" scores 0 for free.
- **BM25 = TF-IDF + saturation (k1≈1.2) + length normalization (b≈0.75).** Extra mentions stop helping; long docs stop winning. It's the default in Lucene/Elasticsearch.
- **Shard** = a self-contained Lucene index. **Replica** = a copy, for HA *and* read throughput. Primary shard count is fixed at index creation.
- **Scatter-gather:** query every shard → each returns local top N (ids + scores) → coordinator merges/re-ranks → fetches the winners. **Latency = the slowest shard.**
- **Segments are immutable.** Writes → new segments; deletes → tombstones; updates → delete + re-add; merges reclaim space.
- **"Near real-time" = ~1s refresh interval.** A just-written doc is briefly invisible to search. By design.
- **Typos** → fuzzy/edit distance (allow 1–2, never on short words). **Prefixes** → trie. **Meaning** → embeddings + ANN. **Hybrid** (BM25 + vector) is where modern search is heading.
- **Two-stage retrieval:** cheap index narrows millions → hundreds; expensive ranker sorts the hundreds.
- **Postgres is the source of truth; Elasticsearch is a derived, rebuildable read model** synced via queue/CDC. If wiping the search cluster loses data, the design is wrong.
- **Costs you must name in an interview:** index size ≥ data size, write amplification, eventual consistency, tail latency, operational burden.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [62 — Database Indexing](./62-database-indexing.md) — B-trees make lookup fast; this doc explains why they can't make *search* fast, and what replaces them |
| **Next** | [106 — Design Search Autocomplete](./106-hld-search-autocomplete.md) — the prefix-matching cousin: tries, top-k suggestions, and sub-100ms typeahead |
| **Related** | [64 — Database Sharding](./64-database-sharding.md) — Elasticsearch shards are the same idea; scatter-gather and tail latency come straight from here |
| **Related** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — why the source of truth stays relational while search lives in a specialized store |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — a search index is a derived, rebuildable read model; the same invalidation and staleness thinking applies |
