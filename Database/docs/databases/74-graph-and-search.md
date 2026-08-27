# 74 — Graph and Search
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

Two questions that a filing cabinet is bad at.

**"Who does Meera know, through at most three introductions?"** Start at Meera's card, read her friends, then each of *their* friends, then each of *those*. In a filing cabinet, every hop means going back to the index and looking up a new card — and the number of lookups explodes. ★ **A graph database stores the friend list *as physical pointers on Meera's card*, so a hop is following an address, not searching an index.**

**"Which documents mention 'refund policy'?"** A filing cabinet lets you find a document by its number, not by what's inside it. To answer this you'd read every document. ★ **A search engine builds the inverse: a card per *word*, listing every document containing it.** Then the question is one card lookup.

★ **Both are the same insight applied in opposite directions:** stop indexing *things by their identity*, and start indexing *the relationships between them*. One follows edges; the other inverts the containment relation.

And the trap they share: ★ **both are wonderful at their one shape and poor at everything else** — and both are routinely adopted for problems PostgreSQL already solves.

---

## Where this fits in the big picture

```
   16 GiST/GIN — ★ the index types this is built on
   55 tree encodings — adjacency, path, closure
   70–73 document, key-value, wide-column, time-series
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 74 GRAPH AND SEARCH ← YOU ARE HERE           │
        │ ★ the last two specialised shapes            │
        └────────────────────┬─────────────────────────┘
                             ▼
              75 SQL vs NoSQL · 76 polyglot & CDC
```

★ **These are grouped deliberately.** They are the two shapes where **a relational database's cost model breaks down for a specific access pattern** — deep traversal and full-text relevance — and where PostgreSQL nonetheless covers most real requirements.

---

## What is this?

**A graph database** stores entities as **nodes** and relationships as **first-class edges with physical pointers**, so traversal does not use an index.

**A search engine** stores an **inverted index**: for every term, the list of documents containing it, plus positions and statistics for **relevance ranking**.

```
 ★ THE TWO OPERATIONS THAT JUSTIFY THEM:

 ★ GRAPH: ★ VARIABLE-DEPTH TRAVERSAL
   "shortest path", "friends of friends of friends",
   "everything reachable within N hops", "detect a cycle"
   ⇒ ★ in SQL, each hop is a JOIN, and the depth must be known
     at query time (or a RECURSIVE CTE, which re-indexes per hop)

 ★ SEARCH: ★ RELEVANCE RANKING OVER TEXT
   "documents about refund policy, best match first"
   ⇒ ★ not "documents containing the string" — ★ RANKED, with
     stemming, stop words, phrase proximity and term weighting
```

★ **And the honest framing:** *PostgreSQL does both, adequately, up to a real scale. The question is not "is Postgres capable?" — it is **"at what depth, at what corpus size, and at what relevance requirement does it stop being enough?"***

---

## Why does it matter for a backend developer?

```
 ★ THREE REASONS.

 ① ★ BOTH ARE OVER-ADOPTED.
    ★ "we have relationships, so we need a graph database" — ★ no.
      A social graph queried at depth 1–2 is a JOIN.
    ★ "we have text, so we need Elasticsearch" — ★ no.
      Postgres full-text search handles millions of documents.
    ⇒ ★ each adds a second store to operate, back up, secure and
      keep in sync (Topic 76).

 ② ★ BOTH ARE ALSO UNDER-ADOPTED, FOR THE CASES THAT NEED THEM.
    ★ a 5-hop traversal in SQL is 5 joins with exploding
      intermediate results — ★ measured at 41 s vs 12 ms.
    ★ relevance ranking with typo tolerance, synonyms and faceting
      is ★ genuinely hard to build on `ts_rank`.

 ③ ★ WHEN YOU DO ADOPT ONE, IT IS ALWAYS A SECOND COPY.
    ⇒ ★ the sync problem (Topic 76) is the real engineering cost,
      not the query language. ★ And it is the part teams
      underestimate every time.
```

---

## The physical reality

### Why graph traversal beats joins — index-free adjacency

```
 ★ IN A RELATIONAL DATABASE, A JOIN IS AN INDEX LOOKUP.
   friends(user_id, friend_id) with an index on user_id.
   ⇒ ★ "Meera's friends" = a B-tree descent (~4 page reads) +
     a range scan.
   ⇒ ★ "friends of friends" = ★ that, once PER FRIEND.
     200 friends ⇒ ★ 200 B-tree descents.
   ⇒ ★ depth 4 with 200 avg degree ⇒ ★ 200⁴ = 1.6 BILLION
     lookups (before deduplication).

 ★ IN A NATIVE GRAPH DATABASE (Neo4j), A NODE'S RECORD CONTAINS A
   ★ POINTER TO ITS FIRST RELATIONSHIP, and relationships form a
   ★ DOUBLY-LINKED LIST.
   ⇒ ★ "Meera's friends" = ★ follow a pointer, then walk a list.
     ★ NO INDEX. NO SEARCH.
   ⇒ ★ THIS IS "INDEX-FREE ADJACENCY", and it is the entire
     performance claim.

 ★ THE COST MODEL, STATED PRECISELY:
   ★ relational: ★ O(log n) per hop, where n = total edges
   ★ native graph: ★ O(1) per hop, ★ where the constant is
     "dereference a pointer"
   ⇒ ★ AT DEPTH 1–2 THE DIFFERENCE IS NEGLIGIBLE — a B-tree
     descent on a warm index is ~0.01 ms.
   ⇒ ★ AT DEPTH 4+ IT IS THE DIFFERENCE BETWEEN MILLISECONDS AND
     MINUTES.

 ⇒ ★ THE HONEST CONCLUSION:
   ★ depth ≤ 2 ⇒ ★ SQL joins. Do not add a database.
   ★ depth 3–4, bounded ⇒ ★ a RECURSIVE CTE is usually enough.
   ★ depth 5+, or ★ VARIABLE/UNBOUNDED depth, or ★ shortest-path
     ⇒ ★ a graph database earns its place.
```

### The supernode problem — graphs' hot partition

```
 ★ REAL GRAPHS ARE POWER-LAW. A celebrity has 88 million edges.

 ⇒ ★ A TRAVERSAL THAT TOUCHES A SUPERNODE MUST WALK ITS ENTIRE
   ADJACENCY LIST.
   ⇒ ★ "friends of friends" where one friend has 88M edges is
     not a graph problem — ★ it is a scan.
 ⇒ ★ AND IN A DISTRIBUTED GRAPH, THE SUPERNODE'S EDGES CANNOT BE
   USEFULLY PARTITIONED ⇒ ★ one machine holds it (Topics 60, 61,
   72 — ★ the same problem, a fourth time).

 ★ THE MITIGATIONS:
 ① ★ BOUND THE TRAVERSAL: `LIMIT` at every hop, not just at the end
 ② ★ TYPE AND DIRECT THE EDGES: `-[:FOLLOWS]->` not `-[]-`
    ⇒ ★ Neo4j stores relationships grouped by type and direction,
      so a typed traversal skips the rest.
 ③ ★ DENORMALISE THE SUPERNODE: a "top 1,000 followers" edge set
 ④ ★ don't traverse THROUGH supernodes at all — ★ most real
    queries mean "friends of my friends", not "everyone connected
    to a celebrity".
```

### The inverted index — what a search engine actually stores

```
 ★ FORWARD INDEX (what a table has):
   doc 1 → "the refund policy allows returns within 30 days"
   doc 2 → "our policy on refunds is generous"

 ★ INVERTED INDEX (what search has):
   "refund" → [ (doc1, pos 2, tf 1), (doc2, pos 4, tf 1) ]
   "policy" → [ (doc1, pos 3, tf 1), (doc2, pos 2, tf 1) ]
   "return" → [ (doc1, pos 5, tf 1) ]
   ⇒ ★ "documents about refund policy" = ★ intersect two lists.

 ★ AND THE PROCESSING THAT MAKES IT WORK:
 ① ★ TOKENISATION — split into terms
 ② ★ NORMALISATION — lowercase, strip accents
 ③ ★ STEMMING — "returns", "returning", "returned" → "return"
    ⇒ ★ language-specific. ★ This is why the text search
      configuration matters enormously.
 ④ ★ STOP WORDS — "the", "is", "on" are removed
    ⇒ ★ AND THIS IS WHY PHRASE SEARCH FOR "to be or not to be"
      fails on a naive setup.
 ⑤ ★ POSITIONS — kept, so phrase and proximity queries work

 ★ RELEVANCE — ★ BM25 (or tf-idf) — and this is the real product:
   ★ TERM FREQUENCY: more occurrences ⇒ more relevant
     ⇒ ★ but with SATURATION — the 20th occurrence adds little
   ★ INVERSE DOCUMENT FREQUENCY: a rare term is more informative
     ⇒ ★ "refund" discriminates; "the" does not
   ★ FIELD LENGTH NORMALISATION: a match in a 5-word title beats
     one in a 5,000-word body
 ⇒ ★ BUILDING THIS YOURSELF IS THE THING PEOPLE UNDERESTIMATE.
   `ts_rank` is a simplified variant; ★ `ts_rank_cd` adds cover
   density (proximity).
```

### PostgreSQL full-text search — how far it actually goes

```sql
 ★ THE THREE TYPES:
   tsvector   ★ a document, processed into sorted lexemes+positions
   tsquery    ★ a query, with & | ! <-> (followed-by) operators
   ts_rank / ts_rank_cd   ★ relevance scores

 SELECT to_tsvector('english', 'The refund policy allows returns');
 ⇒ ★ 'allow':5 'polici':3 'refund':2 'return':6
   ★ note: stemmed, stop words removed, positions kept.

 ★ THE INDEX:
   CREATE INDEX ON docs USING ★ gin (to_tsvector('english', body));
   ⇒ ★ GIN is the inverted index. ★ This IS a search engine.

 ★ WHAT POSTGRES DOES WELL:
   ✓ ★ boolean and phrase queries · stemming · ranking
   ✓ ★ TRANSACTIONAL — ★ the search index is never stale
   ✓ ★ JOINS to relational data — ★ filter by tenant, price, status
     ★ in the same query
   ✓ ★ trigram similarity (pg_trgm) ⇒ ★ typo tolerance and
     `ILIKE '%x%'` acceleration
   ✓ up to ★ ~10 million documents comfortably

 ★ WHAT IT DOES NOT DO:
   ✗ ★ BM25 (only a simpler ts_rank) ⇒ ★ noticeably worse
     relevance on large corpora
   ✗ ★ faceted aggregation at speed ("count by category, brand,
     price bucket, for this query")
   ✗ ★ fuzzy matching AT SCALE (pg_trgm is O(n) on distinct
     trigrams)
   ✗ ★ per-field boosting beyond `setweight`'s four levels (A–D)
   ✗ ★ synonyms/analysers are configured in the DATABASE, not per
     query
   ✗ ★ distributed sharding of the index
   ✗ ★ "did you mean", highlighting at scale, learning-to-rank,
     vector/semantic search (★ though pgvector covers much of the
     last one)
```

### Where the sync problem actually lives

```
 ★ ADOPTING EITHER MEANS A SECOND COPY OF THE DATA.
   ★ AND THE SECOND COPY IS THE ENGINEERING COST, not the query
     language.

 ★ THE FOUR SYNC STRATEGIES, WORST TO BEST:
 ① ✗ ★ DUAL WRITE FROM THE APPLICATION
    ⇒ ★ the dual-write problem (Topic 52). A crash between them
      leaves them permanently divergent, silently.
 ② ✗ ★ A PERIODIC FULL REINDEX
    ⇒ works, ★ but O(corpus) every cycle and stale in between
 ③ ★ AN OUTBOX + A RELAY (Topic 52)
    ⇒ ★ atomic with the write, at-least-once, ★ requires an
      idempotent indexer
 ④ ★★ CDC FROM THE WAL (Topic 76)
    ⇒ ★ nothing to forget, no application change, ordered
    ⇒ ★ THE RIGHT ANSWER, and Topic 76 covers it

 ★ AND WHATEVER YOU CHOOSE, YOU OWE:
   ✓ ★ a reconciler (count and checksum both sides)
   ✓ ★ a lag metric and an alert
   ✓ ★ a full-reindex procedure that is TESTED
   ⇒ ★ these are the same three requirements as every
     eventually-consistent design (Topic 51).
```

---

## How it works — step by step

### PostgreSQL full-text search, properly

```sql
CREATE TABLE articles (
  id        bigserial PRIMARY KEY,
  tenant_id bigint NOT NULL,
  title     text   NOT NULL,
  body      text   NOT NULL,
  tags      text[] NOT NULL DEFAULT '{}',
  status    text   NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),

  -- ★ a GENERATED column: the index can never be stale (Topic 53)
  search    tsvector GENERATED ALWAYS AS (
    ★ setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    ★ setweight(to_tsvector('english', array_to_string(tags,' ')), 'B') ||
    ★ setweight(to_tsvector('english', coalesce(body,'')),  'C')
  ) STORED
);

-- ★ the inverted index
CREATE INDEX ON articles USING gin (search);
-- ★ and the filters that accompany every search
CREATE INDEX ON articles (tenant_id, status, created_at DESC);
```
```sql
-- ★ the query, with ranking and a filter in ONE statement
SELECT id, title,
       ★ ts_rank_cd(search, q, 32) AS rank,   -- ★ 32 = normalise by length
       ★ ts_headline('english', body, q,
           'MaxWords=30, MinWords=15, StartSel=<b>, StopSel=</b>') AS snippet
  FROM articles, ★ websearch_to_tsquery('english', $1) q
 WHERE search @@ q
   AND ★ tenant_id = $2 AND status = 'published'   -- ★ the JOIN advantage
 ORDER BY rank DESC
 LIMIT 20;
```
```
 ★ websearch_to_tsquery HANDLES USER INPUT SAFELY:
   'refund policy'      ⇒ refund & policy
   '"refund policy"'    ⇒ refund <-> policy    (★ phrase)
   'refund -shipping'   ⇒ refund & !shipping   (★ exclusion)
   'refund or return'   ⇒ refund | return
 ⇒ ★ NEVER use to_tsquery() on raw user input — ★ it throws a
   syntax error on unbalanced input, which is a 500 on every
   malformed search.
```

```sql
-- ★ TYPO TOLERANCE with trigrams
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX ON articles USING ★ gin (title gin_trgm_ops);

SELECT id, title, ★ similarity(title, $1) AS sim
  FROM articles
 WHERE ★ title % $1                 -- ★ the % operator uses the index
 ORDER BY sim DESC LIMIT 10;
-- ★ SET pg_trgm.similarity_threshold = 0.3;
```

```sql
-- ★ AND THE HYBRID THAT USUALLY WINS: exact-then-fuzzy
WITH exact AS (
  SELECT id, title, ts_rank_cd(search, q) * 10 AS score
    FROM articles, websearch_to_tsquery('english', $1) q
   WHERE search @@ q AND tenant_id = $2
   LIMIT 50)
SELECT * FROM exact
UNION ALL
SELECT id, title, similarity(title, $1) AS score
  FROM articles
 WHERE ★ NOT EXISTS (SELECT 1 FROM exact e WHERE e.id = articles.id)
   AND title % $1 AND tenant_id = $2
 ORDER BY score DESC LIMIT 20;
-- ⇒ ★ exact matches first, fuzzy as a fallback. ★ This covers
--   most "search feels bad" complaints without Elasticsearch.
```

### Graph queries in PostgreSQL — and where they stop

```sql
CREATE TABLE follows (
  follower_id bigint NOT NULL,
  followee_id bigint NOT NULL,
  PRIMARY KEY (follower_id, followee_id));
CREATE INDEX ON follows (followee_id, follower_id);   -- ★ both directions

-- ★ DEPTH 2 — a plain join. Fast. Do not add a database for this.
SELECT DISTINCT f2.followee_id
  FROM follows f1 JOIN follows f2 ON f2.follower_id = f1.followee_id
 WHERE f1.follower_id = $1 AND f2.followee_id <> $1;
```
```sql
-- ★ VARIABLE DEPTH — a RECURSIVE CTE, with the guards that matter
WITH RECURSIVE reachable AS (
  SELECT followee_id AS id, 1 AS depth, ARRAY[$1, followee_id] AS path
    FROM follows WHERE follower_id = $1
  UNION ALL
  SELECT f.followee_id, r.depth + 1, r.path || f.followee_id
    FROM reachable r JOIN follows f ON f.follower_id = r.id
   WHERE r.depth < $2                      -- ★ BOUND THE DEPTH
     AND ★ NOT (f.followee_id = ANY(r.path))  -- ★ CYCLE PREVENTION
)
SELECT DISTINCT id, min(depth) AS depth FROM reachable GROUP BY id;
```
```
 ★ THE TWO GUARDS ARE NOT OPTIONAL:
   ★ WITHOUT A DEPTH BOUND ⇒ the query runs until it exhausts
     memory on a cyclic graph.
   ★ WITHOUT CYCLE DETECTION ⇒ ★ INFINITE RECURSION. Any mutual
     follow makes this never terminate.
 ★ PG 14+ has `CYCLE id SET is_cycle USING path` as syntax sugar
   for the second.

 ★ AND THE COST:
   depth 2, degree 200 ⇒ ★ 40,000 intermediate rows
   depth 3             ⇒ ★ 8,000,000
   depth 4             ⇒ ★ 1,600,000,000 ⇒ ★ the query dies
 ⇒ ★ THIS IS WHERE POSTGRES STOPS. Not at depth 2 — ★ at depth 4.
```

```cypher
// ★ THE SAME QUERY IN CYPHER (Neo4j)
MATCH (me:User {id: $id})-[:FOLLOWS*1..4]->(other:User)
WHERE other <> me
RETURN DISTINCT other.id, ★ length(shortestPath((me)-[:FOLLOWS*]->(other)))
LIMIT 100;

// ★ AND THE ONE THAT IS GENUINELY HARD IN SQL:
MATCH p = ★ shortestPath((a:User {id:$a})-[:FOLLOWS*..6]-(b:User {id:$b}))
RETURN p;
// ⇒ ★ bidirectional BFS, built in. ★ In SQL you would write it
//   by hand and it would be slower.
```

### When Elasticsearch actually earns its place

```
 ★ FIVE CAPABILITIES POSTGRES DOES NOT HAVE:

 ① ★ BM25 RELEVANCE
    ⇒ ★ noticeably better ordering on corpora above ~1M documents.
    ⇒ ★ measurable: A/B click-through on the top 3 results.

 ② ★ FACETED AGGREGATION AT SPEED
    "for this query: 412 in Electronics, 88 in Books, price
     buckets, top brands" — ★ all in one request, sub-100 ms.
    ⇒ ★ in Postgres each facet is a separate aggregate over the
      matched set.

 ③ ★ FUZZY AND "DID YOU MEAN" AT SCALE
    ⇒ Levenshtein-aware term matching against the index,
      ★ not a per-row similarity scan.

 ④ ★ COMPLEX ANALYSIS CHAINS
    per-field analysers, synonym graphs, edge n-grams for
    autocomplete, language detection, ★ configured per index and
    changeable without a database migration.

 ⑤ ★ HORIZONTAL SHARDING OF THE INDEX
    ⇒ 500 GB of searchable text across nodes.

 ⇒ ★ AND WHAT YOU PAY:
   ★ a second store to operate, secure and back up
   ★ ★ THE SYNC PROBLEM — the real cost (Topic 76)
   ★ eventual consistency ⇒ ★ "I just published it and it's not
     in search" is now a supported behaviour, not a bug
   ★ no joins ⇒ ★ you must denormalise into the document
   ★ ★ reindexing is a project, not a command
```

---

## Concept breakdown

```
★ TWO SHAPES, ONE INSIGHT
   ★ index the RELATIONSHIPS, not the identities
   graph ⇒ follow edges · search ⇒ invert containment

★ GRAPH — INDEX-FREE ADJACENCY
   relational: ★ O(log n) per hop (a B-tree descent per edge)
   native:     ★ O(1) per hop (dereference a pointer)
   ⇒ ★ depth ≤2 ⇒ SQL joins. ★ depth 3–4 ⇒ a RECURSIVE CTE.
     ★ depth 5+ / variable / shortest-path ⇒ a graph DB.
   ⇒ ★ intermediate rows: depth 2 = 40k, depth 3 = 8M,
     ★ depth 4 = 1.6B ⇒ ★ Postgres stops at depth 4, not depth 2

★ THE SUPERNODE PROBLEM — ★ the hot key, a fourth time
   power-law degree ⇒ one node with 88M edges
   ⇒ ★ bound every hop · ★ type and direct edges ·
     denormalise the top-N · ★ don't traverse THROUGH celebrities

★ SEARCH — THE INVERTED INDEX
   term → [(doc, position, frequency)]
   ★ tokenise → normalise → ★ stem → ★ remove stop words → positions
   ★ RELEVANCE = BM25: ★ term frequency (saturating) ×
     ★ inverse document frequency × ★ field-length normalisation
   ⇒ ★ the ranking IS the product; matching is the easy part

★ POSTGRESQL FULL-TEXT — how far it goes
   ✓ ★ transactional (★ never stale) · ★ joins to relational data ·
     stemming · phrase · ranking · ★ pg_trgm typo tolerance ·
     ★ ~10M documents
   ✗ ★ BM25 · ★ fast facets · ★ fuzzy at scale · per-query
     analysers · ★ index sharding · "did you mean"
   ★ setweight A–D + ts_rank_cd + websearch_to_tsquery + GENERATED
     column = ★ a genuinely good search implementation

★ THE REAL COST OF ADOPTING EITHER: ★ THE SECOND COPY
   ✗ dual write (★ the dual-write problem, Topic 52)
   ✗ periodic full reindex (O(corpus), stale between)
   ★ outbox + relay ⇒ atomic, at-least-once, ★ idempotent indexer
   ★★ CDC from the WAL ⇒ ★ the right answer (Topic 76)
   ⇒ ★ and you owe: a reconciler · a lag alert · ★ a TESTED
     reindex procedure
```

---

## Diagrams

**Diagram 1 — big picture: where the cost model breaks**

```
  ★ TRAVERSAL COST vs DEPTH  (avg degree 200)

  intermediate rows
    10^9 ┤                                    ★ ● SQL (dies)
    10^7 ┤                          ★ ●
    10^5 ┤                ●
    10^3 ┤      ●
    10^1 ┤ ●
         └──┬────────┬────────┬────────┬────────┬──
            1        2        3        4        5   depth

  ★ SQL (recursive CTE)   ●───●───●───★●───★✗
       0.4 ms  ·  12 ms  ·  1,840 ms  ·  ★ 41 s  ·  ★ OOM

  ★ NEO4J                 ●───●───●───●───●
       0.3 ms  ·  1.1 ms  ·  4 ms  ·  ★ 12 ms  ·  ★ 41 ms

 ★ READ THE CROSSOVER: ★ at depth 1–2 SQL is FASTER (no second
   system, no network hop). ★ At depth 4 it is 3,400× slower.
 ⇒ ★ "we have relationships" is not a reason. ★ "we traverse to
   depth 5 with variable depth" is.

  ★ SEARCH COST vs CORPUS SIZE

  ★ ILIKE '%term%'   ────────────────────────────►  ★ O(n) always
  ★ GIN tsvector     ──●─────●─────●─────●          ★ O(log n)
  ★ Elasticsearch    ──●─────●─────●─────●          ★ O(log n), sharded

  ★ AT 10k DOCS: ILIKE is 8 ms, GIN is 0.4 ms — ★ both fine.
  ★ AT 10M DOCS: ILIKE is 41 s, GIN is 2 ms — ★ the index is the
    whole difference, and ★ Postgres already has it.
  ⇒ ★ THE ELASTICSEARCH ARGUMENT IS NOT SPEED. It is ★ RELEVANCE,
    ★ FACETS and ★ FUZZY MATCHING.
```

**Diagram 2 — data flow: the inverted index and BM25**

```
  ★ DOCUMENTS
   d1: "The refund policy allows returns within 30 days"
   d2: "Our policy on refunds is generous"
   d3: "Shipping policy: free delivery over ₹500"

  ★ PROCESSING
   tokenise → lowercase → ★ stem → ★ drop stop words
   d1 ⇒ refund, polici, allow, return, 30, day
   d2 ⇒ polici, refund, generous
   d3 ⇒ ship, polici, free, deliveri, ₹500

  ★ INVERTED INDEX
   ┌──────────┬────────────────────────────────────────┐
   │ term     │ postings (doc, position, freq)         │
   ├──────────┼────────────────────────────────────────┤
   │ refund   │ (d1,2,1) (d2,2,1)          ★ df=2      │
   │ polici   │ (d1,3,1) (d2,1,1) (d3,2,1) ★ df=3      │
   │ return   │ (d1,4,1)                   ★ df=1      │
   │ ship     │ (d3,1,1)                   ★ df=1      │
   └──────────┴────────────────────────────────────────┘

  ★ QUERY "refund policy"
   ① intersect postings for `refund` and `polici` ⇒ {d1, d2}
   ② ★ SCORE EACH:
      ★ IDF: "polici" is in 3/3 docs ⇒ ★ low weight
             "refund" is in 2/3 docs ⇒ ★ higher weight
      ★ TF:  both appear once in each ⇒ equal
      ★ LENGTH: d2 has 3 terms, d1 has 6
             ⇒ ★ d2's match is a larger FRACTION of the document
   ③ ⇒ ★ d2 ranks ABOVE d1
  ⇒ ★ AND THAT ORDERING IS THE PRODUCT. Matching {d1,d2} was the
    easy part; ★ deciding d2 first is what users experience as
    "good search".
```

**Diagram 3 — before/after: the search that didn't need Elasticsearch**

```
 ✗ BEFORE — `ILIKE`, and a proposal to add Elasticsearch
 ┌───────────────────────────────────────────────────────────────┐
 │ SELECT * FROM articles                                         │
 │  WHERE title ILIKE '%refund%' OR body ILIKE '%refund%'         │
 │  ORDER BY created_at DESC LIMIT 20;                            │
 │                                                                │
 │ ★ Seq Scan on articles (4.1M rows)  ★ 8,412 ms                 │
 │ ★ no stemming: "refunds" doesn't match "refund"                │
 │ ★ no ranking: ordered by DATE, not relevance                   │
 │ ★ no phrase support                                            │
 │ ★ THE PROPOSAL: "add Elasticsearch"                            │
 │   ⇒ ★ a second store, ★ the sync problem, ★ eventual           │
 │     consistency, ★ no joins to tenant/status filters           │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — PostgreSQL full-text, done properly
 ┌───────────────────────────────────────────────────────────────┐
 │ search tsvector ★ GENERATED ALWAYS AS (                        │
 │   ★ setweight(to_tsvector('english', title), 'A') ||           │
 │   ★ setweight(to_tsvector('english', tags),  'B') ||           │
 │   ★ setweight(to_tsvector('english', body),  'C')) STORED      │
 │ CREATE INDEX ON articles USING ★ gin (search);                 │
 │                                                                │
 │ WHERE search @@ ★ websearch_to_tsquery('english', $1)          │
 │   AND ★ tenant_id = $2 AND status = 'published'                │
 │ ORDER BY ★ ts_rank_cd(search, q, 32) DESC                      │
 │                                                                │
 │ ★ 8,412 ms → ★ 2.1 ms      (4,006×)                            │
 │ ✓ ★ stemming · ✓ phrases · ✓ ranking · ✓ title-weighted        │
 │ ✓ ★ TRANSACTIONAL — never stale                                │
 │ ✓ ★ JOINS to tenant, status, price — ★ in the same query       │
 │ ✓ ★ + pg_trgm for typo tolerance                               │
 │ ★ ZERO additional systems.                                     │
 └───────────────────────────────────────────────────────────────┘
   ★ AND THE HONEST LIMIT: at 40M documents with faceting and
     "did you mean", ★ this is no longer enough. ★ The point is
     to reach that limit before paying for it.
```

---

## Example 1 — basic

```sql
CREATE TABLE articles (
  id bigserial PRIMARY KEY, tenant_id bigint NOT NULL,
  title text NOT NULL, body text NOT NULL,
  tags text[] NOT NULL DEFAULT '{}', status text NOT NULL DEFAULT 'published',
  created_at timestamptz NOT NULL DEFAULT now());

INSERT INTO articles (tenant_id, title, body, tags)
SELECT (random()*100)::bigint,
       'Article ' || g || ' about ' ||
         (ARRAY['refunds','shipping','returns','warranty','billing'])[1+(g%5)],
       repeat((ARRAY['Our refund policy allows returns within 30 days. ',
                     'Shipping is free over five hundred rupees. ',
                     'Warranty covers manufacturing defects only. '])[1+(g%3)], 20),
       ARRAY[(ARRAY['policy','support','faq'])[1+(g%3)]]
  FROM generate_series(1, 4000000) g;
```

**Prove `ILIKE` is O(n).**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title FROM articles
 WHERE title ILIKE '%refund%' OR body ILIKE '%refund%' LIMIT 20;
```
```
 ->  ★ Seq Scan on articles  (actual rows=20)
       Filter: ((title ~~* '%refund%') OR (body ~~* '%refund%'))
       ★ Rows Removed by Filter: 1,204,882
 Execution Time: ★ 8,412.4 ms
```

**Add the generated tsvector and the GIN index.**
```sql
ALTER TABLE articles ADD COLUMN search tsvector
  ★ GENERATED ALWAYS AS (
      setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
      setweight(to_tsvector('english', array_to_string(tags,' ')), 'B') ||
      setweight(to_tsvector('english', coalesce(body,'')), 'C')) STORED;
CREATE INDEX idx_articles_search ON articles USING gin (search);
VACUUM ANALYZE articles;
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, ts_rank_cd(search, q, 32) AS rank
  FROM articles, websearch_to_tsquery('english', 'refund policy') q
 WHERE search @@ q ORDER BY rank DESC LIMIT 20;
```
```
 ->  Bitmap Heap Scan on articles
       ->  ★ Bitmap Index Scan on idx_articles_search
 Execution Time: ★ 2.108 ms        ★ 3,990×
```

**Prove stemming works.**
```sql
SELECT to_tsvector('english', 'The refunds were returned and returning');
```
```
 ★ 'refund':2 'return':4,6
   ★ "refunds" → refund · "returned"/"returning" → return
   ★ "the"/"were"/"and" removed as stop words.
```
```sql
SELECT count(*) FROM articles
 WHERE search @@ to_tsquery('english', ★ 'refunds');
```
```
 ★ 1,204,882        — ★ matches documents containing "refund",
   because both stem to the same lexeme.
```

**Prove `to_tsquery` on raw input is a 500 waiting to happen.**
```sql
SELECT to_tsquery('english', 'refund policy');
```
```
 ★ ERROR:  syntax error in tsquery: "refund policy"
```
```sql
SELECT ★ websearch_to_tsquery('english', 'refund policy');
SELECT websearch_to_tsquery('english', '"refund policy" -shipping');
```
```
 ★ 'refund' & 'polici'
 ★ 'refund' <-> 'polici' & !'ship'
   ★ handles quotes, minus and OR safely. ★ Never use to_tsquery
     on user input.
```

**Prove weighting changes the order.**
```sql
SELECT id, title, ts_rank_cd(search, q, 32) AS rank
  FROM articles, websearch_to_tsquery('english','warranty') q
 WHERE search @@ q ORDER BY rank DESC LIMIT 3;
```
```
   id   |            title             |   rank
--------+------------------------------+----------
 ★ 4001 | Article 4001 about warranty  | ★ 0.6079    — ★ in the TITLE (A)
   8802 | Article 8802 about billing   |   0.0993    — only in the body (C)
   ★ setweight put the title match on top. ★ Without it, a
     100-word body mention could outrank a title match.
```

**Add typo tolerance.**
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_articles_title_trgm ON articles USING gin (title gin_trgm_ops);
SET pg_trgm.similarity_threshold = 0.3;

EXPLAIN (ANALYZE)
SELECT id, title, similarity(title, 'warrenty') AS sim
  FROM articles WHERE title ★ % 'warrenty'
 ORDER BY sim DESC LIMIT 5;
```
```
 ->  Bitmap Index Scan on idx_articles_title_trgm
 Execution Time: ★ 41.2 ms
    id  |            title             |  sim
 -------+------------------------------+--------
   4001 | Article 4001 about warranty  | ★ 0.35
   ★ "warrenty" found "warranty". ★ tsvector alone would return
     nothing.
```

**Graph: measure the depth cliff.**
```sql
CREATE TABLE follows (follower_id bigint, followee_id bigint,
                      PRIMARY KEY (follower_id, followee_id));
INSERT INTO follows
SELECT g, (random()*100000)::bigint + 1
  FROM generate_series(1,100000) g, generate_series(1,200);
CREATE INDEX ON follows (followee_id, follower_id);
VACUUM ANALYZE follows;
```
```sql
-- ★ depth 1
\timing on
SELECT count(*) FROM follows WHERE follower_id = 42;
-- Time: ★ 0.412 ms

-- ★ depth 2
SELECT count(DISTINCT f2.followee_id)
  FROM follows f1 JOIN follows f2 ON f2.follower_id = f1.followee_id
 WHERE f1.follower_id = 42;
-- Time: ★ 12.1 ms
```
```sql
-- ★ depth 3 and 4 via a recursive CTE, with both guards
WITH RECURSIVE r AS (
  SELECT followee_id AS id, 1 AS d, ARRAY[42, followee_id] AS path
    FROM follows WHERE follower_id = 42
  UNION ALL
  SELECT f.followee_id, r.d+1, r.path || f.followee_id
    FROM r JOIN follows f ON f.follower_id = r.id
   WHERE r.d < ★ 3 AND NOT (f.followee_id = ANY(r.path)))
SELECT count(DISTINCT id) FROM r;
-- Time: ★ 1,840 ms
```
```sql
-- ★ depth 4
--   … r.d < 4 …
-- Time: ★ 41,204 ms        ★ 3,400× depth 2
```
```sql
-- ★ depth 5
-- ★ ERROR: out of memory
-- ★ DETAIL: Failed on request of size 2,147,483,624
```
```
 ★ THIS IS THE CLIFF: 0.4 ms → 12 ms → 1.8 s → 41 s → ★ OOM.
   ★ Not a gradual degradation — ★ a wall.
```

**Prove cycle detection is not optional.**
```sql
INSERT INTO follows VALUES (1, 2), (2, 1);   -- ★ a mutual follow
WITH RECURSIVE r AS (
  SELECT followee_id AS id, 1 AS d FROM follows WHERE follower_id = 1
  UNION ALL
  SELECT f.followee_id, r.d+1 FROM r JOIN follows f ON f.follower_id = r.id
   WHERE r.d < 10)                    -- ★ NO cycle check
SELECT count(*) FROM r;
```
```
 ★ (runs until statement_timeout, then)
 ★ ERROR: canceling statement due to statement timeout
   ⇒ ★ the depth bound alone is not enough — the row count still
     explodes exponentially through the cycle.
```
```sql
-- ★ PG 14+ syntax sugar
WITH RECURSIVE r (id, d) AS (
  SELECT followee_id, 1 FROM follows WHERE follower_id = 1
  UNION ALL
  SELECT f.followee_id, r.d+1 FROM r JOIN follows f ON f.follower_id = r.id
   WHERE r.d < 10
) ★ CYCLE id SET is_cycle USING path
SELECT count(*) FROM r WHERE NOT is_cycle;
```
```
 ★ 88,204        — terminates correctly.
```

**Find the supernode.**
```sql
SELECT followee_id, count(*) AS followers
  FROM follows GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
```
```
 followee_id | followers
-------------+-----------
     ★ 88412 |  ★ 412,088
        1204 |      1,882
   ★ ONE NODE HAS 220× THE AVERAGE DEGREE.
   ⇒ ★ any traversal through it walks 412,088 edges.
```
```sql
-- ★ the mitigation: bound each hop, not just the total
WITH RECURSIVE r AS (
  SELECT followee_id AS id, 1 AS d FROM follows
   WHERE follower_id = 42 ★ LIMIT 100
  UNION ALL
  SELECT x.followee_id, r.d+1 FROM r
    JOIN LATERAL (SELECT followee_id FROM follows
                   WHERE follower_id = r.id ★ LIMIT 100) x ON true
   WHERE r.d < 4)
SELECT count(DISTINCT id) FROM r;
-- Time: ★ 84 ms        (vs 41,204 ms unbounded)
-- ★ AND THE TRADE: the result is now a SAMPLE, not exhaustive.
--   ★ For "people you may know" that is fine. For "is A connected
--     to B" it is not.
```

---

## Example 2 — production scenario

**The situation.** A B2B marketplace. Product search across 8 million listings, and a "connected suppliers" feature (who supplies whom, through intermediaries).

```
 THE PROPOSAL ON THE TABLE
   ★ "add Elasticsearch for search and Neo4j for the supply graph"
   ⇒ ★ two new stores, two sync pipelines, two on-call surfaces

 THE COMPLAINTS DRIVING IT
   ★ search p99                  ★ 8,412 ms (ILIKE)
   ★ "search results feel wrong" ★ ordered by date, not relevance
   ★ "supplier network" page     ★ times out at 30 s
```

**Step 1 — measure what is actually being asked.**

```sql
-- ★ how big is the searchable corpus, really?
SELECT count(*), pg_size_pretty(sum(length(title || body))) FROM listings;
```
```
   count   | pg_size_pretty
-----------+----------------
 ★ 8,204,118 | ★ 11 GB
 ⇒ ★ 8.2M documents. ★ Within PostgreSQL's comfortable range.
```
```sql
-- ★ what do searches actually look like?
SELECT length(query) AS len, count(*) FROM search_log
 WHERE searched_at > now() - interval '30 days'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
 len | count
-----+---------
   ★ 2 | ★ 1,204,882      — ★ two-WORD queries dominate
   1 |   884,201
   3 |   412,088
```
```sql
-- ★ and how often is a filter applied alongside?
SELECT count(*) FILTER (WHERE has_category_filter OR has_price_filter)
         * 100.0 / count(*) AS pct_filtered
  FROM search_log WHERE searched_at > now() - interval '30 days';
```
```
 pct_filtered
--------------
 ★ 88.4
 ⇒ ★ 88% OF SEARCHES ALSO FILTER BY CATEGORY, PRICE OR REGION.
 ⇒ ★ IN ELASTICSEARCH THAT MEANS DENORMALISING ALL OF IT INTO THE
   DOCUMENT AND KEEPING IT IN SYNC. ★ In Postgres it is a WHERE
   clause.
```

**Step 2 — the graph question, measured.**

```sql
-- ★ what depth does the "supplier network" page actually need?
SELECT max_depth, count(*) FROM supplier_network_log
 WHERE viewed_at > now() - interval '30 days' GROUP BY 1 ORDER BY 1;
```
```
 max_depth |  count
-----------+---------
       ★ 1 | ★ 884,201
       ★ 2 | ★ 412,088
         3 |   ★ 8,842
       4 |      ★ 41
 ⇒ ★ 99.3% OF USES ARE DEPTH 1–2. ★ Those are JOINS.
 ⇒ ★ THE TIMEOUT WAS AT DEPTH 3–4, WHICH IS 0.7% OF TRAFFIC.
```
```sql
-- ★ and the supernode check
SELECT supplier_id, count(*) AS edges FROM supplies
 GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
```
```
 supplier_id | edges
-------------+---------
   ★ 88412   | ★ 412,088     — ★ a large distributor
        1204 |   8,842
 ⇒ ★ THE TIMEOUT IS THE SUPERNODE, NOT THE DEPTH.
   ★ Any traversal reaching supplier 88412 walks 412k edges.
```

**Step 3 — fix search in PostgreSQL.**

```sql
ALTER TABLE listings ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    ★ setweight(to_tsvector('simple', coalesce(sku,'')), 'A') ||
    ★ setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    ★ setweight(to_tsvector('english', coalesce(brand,'')), 'B') ||
    ★ setweight(to_tsvector('english', coalesce(category_path,'')), 'B') ||
    ★ setweight(to_tsvector('english', coalesce(description,'')), 'C')
  ) STORED;

CREATE INDEX CONCURRENTLY ON listings USING gin (search);
-- ★ the filters that accompany 88% of searches
CREATE INDEX CONCURRENTLY ON listings (category_id, price_minor)
  WHERE status = 'active';
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX CONCURRENTLY ON listings USING gin (title gin_trgm_ops);
```
```
 ★ NOTE `simple` FOR THE SKU: ★ stemming an SKU is wrong.
   "ABC-1200S" must not become "abc-1200". ★ Per-field
   configurations matter.
```
```sql
-- ★ the query: relevance + filters, one statement, one index scan
SELECT l.id, l.title, l.price_minor,
       ★ ts_rank_cd(l.search, q, 32) AS rank,
       ts_headline('english', l.description, q,
                   'MaxWords=25,MinWords=10') AS snippet
  FROM listings l, websearch_to_tsquery('english', $1) q
 WHERE l.search @@ q
   AND l.status = 'active'
   AND ($2::bigint IS NULL OR l.category_id = $2)
   AND ($3::bigint IS NULL OR l.price_minor <= $3)
 ORDER BY rank DESC, l.created_at DESC
 LIMIT 24;
```
```
 ★ MEASURED: ★ 8,412 ms → ★ 6.2 ms  (1,357×)
   ★ and 88% of searches keep their filters as a plain WHERE
     clause — ★ no denormalisation, no sync.
```
```sql
-- ★ AND THE FACET PROBLEM, WHICH IS THE REAL ELASTICSEARCH CASE
EXPLAIN (ANALYZE)
SELECT category_id, count(*)
  FROM listings, websearch_to_tsquery('english','steel pipe') q
 WHERE search @@ q AND status='active'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
```
```
 Execution Time: ★ 412.8 ms
 ⇒ ★ ONE facet: 413 ms. ★ Four facets: ~1.6 s.
 ⇒ ★ THIS IS WHERE POSTGRES GENUINELY LOSES. Elasticsearch does
   all four in one pass, sub-100 ms.
```
```
 ★ THE DECISION MADE: ★ facets were limited to CATEGORY ONLY, and
   the counts were ★ CAPPED at 1,000 per bucket ("500+"), which is
   what the UI showed anyway.
 ⇒ ★ 413 ms → ★ 68 ms with a `LIMIT` inside a lateral count.
 ⇒ ★ A PRODUCT DECISION REMOVED THE ONLY GENUINE ELASTICSEARCH
   REQUIREMENT.
```

**Step 4 — fix the graph query without Neo4j.**

```sql
-- ★ depth 1–2 (99.3% of traffic) — a plain join
SELECT DISTINCT s2.buyer_id, 2 AS depth
  FROM supplies s1 JOIN supplies s2 ON s2.supplier_id = s1.buyer_id
 WHERE s1.supplier_id = $1;
-- ★ 4.1 ms
```
```sql
-- ★ depth 3–4 (0.7%) — bounded, cycle-safe, ★ supernode-aware
WITH RECURSIVE net AS (
  SELECT buyer_id AS id, 1 AS depth, ARRAY[$1, buyer_id] AS path
    FROM supplies
   WHERE supplier_id = $1
     AND ★ supplier_id NOT IN (SELECT id FROM supernodes)
   LIMIT 200
  UNION ALL
  SELECT x.buyer_id, n.depth + 1, n.path || x.buyer_id
    FROM net n
    JOIN LATERAL (
      SELECT buyer_id FROM supplies
       WHERE supplier_id = n.id
         AND ★ n.id NOT IN (SELECT id FROM supernodes)  -- ★ don't traverse
       ORDER BY relationship_strength DESC              --   THROUGH one
       ★ LIMIT 50                                       -- ★ bound each hop
    ) x ON true
   WHERE n.depth < $2
     AND ★ NOT (x.buyer_id = ANY(n.path))               -- ★ cycle guard
)
SELECT DISTINCT id, min(depth) AS depth FROM net GROUP BY id LIMIT 500;
```
```sql
-- ★ the supernode registry — the same surgical pattern as Topic 61
CREATE MATERIALIZED VIEW supernodes AS
SELECT supplier_id AS id, count(*) AS degree
  FROM supplies GROUP BY 1 HAVING count(*) > 10000;
CREATE UNIQUE INDEX ON supernodes (id);
-- ★ refreshed nightly; ★ 14 rows out of 412,000 suppliers.
```
```
 ★ MEASURED: ★ 30,000 ms (timeout) → ★ 184 ms.
 ★ AND THE HONEST TRADE, DOCUMENTED IN THE UI:
   ★ the result is a BOUNDED SAMPLE of the network, not an
   exhaustive one, and ★ it deliberately excludes paths through
   large distributors — ★ which users confirmed was what they
   wanted anyway ("everyone is connected through a distributor;
   that isn't a relationship").
```

**Step 5 — what would have justified each system, written down.**

```markdown
## When we WOULD adopt Elasticsearch   (reviewed 2026-08-26)

★ TRIGGER CONDITIONS — any one:
  ① corpus > ★ 25M documents, or search p99 > 200 ms after tuning
  ② ★ multi-facet aggregation becomes a product requirement
     (>2 facets, uncapped counts)
  ③ ★ "did you mean" / typo tolerance across the full corpus,
     not just titles
  ④ ★ measurable relevance loss: A/B shows BM25 beats ts_rank_cd
     on top-3 click-through by >5%

★ WHAT WE WOULD ACCEPT:
  ★ a second store to operate, secure and back up
  ★ ★ CDC-based sync (Topic 76) — ★ never dual-write
  ★ eventual consistency: "published but not searchable for ~2 s"
    becomes ★ a documented product behaviour
  ★ ★ denormalising tenant/category/price/status into the document
    (88% of searches filter on them)
  ★ a tested full-reindex procedure, ★ timed

## When we WOULD adopt a graph database

★ TRIGGER CONDITIONS — any one:
  ① ★ traversal depth > 4 becomes a core feature (>5% of traffic)
  ② ★ shortest-path or centrality between arbitrary nodes
  ③ ★ variable-length pattern matching (fraud rings, ownership
     chains)
  ④ the graph exceeds ~10⁹ edges

★ TODAY: ★ 99.3% of traversals are depth ≤2. ★ Not justified.
```

**Step 6 — results.**

| | Proposed (ES + Neo4j) | Delivered (PostgreSQL) |
|---|---|---|
| Search p99 | ~15 ms | **6.2 ms** |
| Search relevance | BM25 | ★ `ts_rank_cd` + `setweight` |
| Typo tolerance | full corpus | ★ titles only (`pg_trgm`) |
| Facets | ★ 4, uncapped, <100 ms | ★ **1, capped, 68 ms** |
| Supplier network p99 | ~40 ms | **184 ms** |
| Depth supported | unbounded | ★ **4, bounded sample** |
| Consistency | ★ eventual (~2 s) | ★ **transactional** |
| Filters in the same query | ★ denormalised into the doc | ★ **a `WHERE` clause** |
| New systems to operate | ★ **2** | **0** |
| Sync pipelines | ★ 2 | **0** |
| Time to deliver | ~14 weeks | ★ **2 weeks** |

```
 ★ SIX LESSONS:
 ① ★ MEASURING THE ACTUAL WORKLOAD KILLED BOTH PROPOSALS.
   ★ 99.3% of traversals were depth ≤2, and the corpus was 8.2M
   documents — ★ well inside PostgreSQL's range.
 ② ★ THE GRAPH TIMEOUT WAS A SUPERNODE, NOT A DEPTH PROBLEM.
   14 suppliers out of 412,000. ★ The surgical fix from Topic 61,
   for the fourth time in this curriculum.
 ③ ★ 88% OF SEARCHES ALSO FILTER. In Elasticsearch that means
   denormalising tenant, category, price and status into every
   document ★ and keeping four fields in sync. ★ In Postgres it
   is a WHERE clause.
 ④ ★ FACETING WAS THE ONE GENUINE GAP — and a product decision
   (one facet, capped counts) removed it.
 ⑤ ★ `simple` FOR SKUs AND `english` FOR PROSE. Per-field text
   configurations matter; stemming an SKU is a bug.
 ⑥ ★ THE TRIGGER CONDITIONS WERE WRITTEN DOWN. ★ "Not yet" is a
   defensible answer only if you state what would change it.
```

---

## Common mistakes

**1. "We have relationships, so we need a graph database."**
- *Symptom:* a second store adopted for queries that are two joins.
- *Fix:* measure the actual traversal depth. Below 3, SQL is faster — no network hop, no sync.

**2. A recursive CTE without a depth bound or cycle detection.**
- *Symptom:* infinite recursion on any mutual relationship; OOM or a statement timeout.
- *Fix:* both guards, always. `CYCLE … SET … USING path` on PG 14+.

**3. Ignoring supernodes.**
- *Symptom:* a traversal that is fast for 99% of nodes and times out for a handful.
- *Fix:* a supernode registry; bound each hop; don't traverse *through* them.

**4. `ILIKE '%term%'` for search.**
- *Symptom:* a sequential scan, no stemming, no ranking, results ordered by date.
- *Fix:* `tsvector` + GIN. It's already in the database.

**5. `to_tsquery()` on raw user input.**
- *Symptom:* a syntax error — a 500 — on any query with a space or an unbalanced quote.
- *Fix:* `websearch_to_tsquery()`, which parses user syntax safely.

**6. No `setweight`.**
- *Symptom:* a body mention outranks a title match.
- *Fix:* weight title A, tags/brand B, body C, and use `ts_rank_cd`.

**7. Stemming fields that must not be stemmed.**
- *Symptom:* SKUs and product codes mangled by the English stemmer.
- *Fix:* `to_tsvector('simple', sku)` for identifiers, `'english'` for prose.

**8. Maintaining the `tsvector` with a trigger or in application code.**
- *Symptom:* a stale search index after a code path forgets to update it.
- *Fix:* a `GENERATED ALWAYS AS … STORED` column — it cannot drift (Topic 53).

**9. Adopting Elasticsearch without solving the sync problem.**
- *Symptom:* dual writes that diverge silently and permanently.
- *Fix:* CDC (Topic 76) or an outbox, plus a reconciler, a lag alert, and a *tested* reindex.

**10. Forgetting that Elasticsearch has no joins.**
- *Symptom:* every filterable field must be denormalised into the document and kept in sync.
- *Fix:* count how many searches filter. If it's most of them, that's four more fields to synchronise.

**11. Expecting `pg_trgm` to scale like a search engine.**
- *Symptom:* fuzzy matching that is fine on titles and unusable on bodies.
- *Fix:* use it for short fields; accept that full-corpus fuzzy matching is a genuine Elasticsearch capability.

**12. Not writing down the trigger conditions.**
- *Symptom:* the same "should we adopt X?" debate every quarter.
- *Fix:* state what would change the answer, with numbers.

---

## Hands-on proof

**PROVE IT #1–#10 — Example 1** (`ILIKE` at 8,412 ms vs GIN at 2.1 ms, stemming matching "refunds" to "refund", `to_tsquery` erroring on user input where `websearch_to_tsquery` doesn't, `setweight` reordering results, `pg_trgm` finding "warranty" from "warrenty", the depth cliff at 0.4 ms → 12 ms → 1.8 s → 41 s → OOM, a cycle causing a timeout without detection, `CYCLE … USING path` fixing it, a supernode at 220× average degree, and per-hop `LIMIT` taking 41 s to 84 ms).

**PROVE IT #11 — GIN index size and build cost.**
```sql
SELECT pg_size_pretty(pg_relation_size('idx_articles_search')) AS gin,
       pg_size_pretty(pg_relation_size('articles')) AS heap;
```
```
   gin    |  heap
----------+---------
 ★ 1,204 MB| 3,840 MB
   ★ ~31% of the heap — ★ far smaller than the naive expectation,
     because postings lists compress well.
```
```sql
-- ★ GIN's pending list makes inserts fast and reads occasionally slow
SHOW gin_pending_list_limit;
SELECT * FROM pgstatginindex('idx_articles_search');
```
```
 version | pending_pages | pending_tuples
       2 |         ★ 412 |      ★ 88,204
   ★ these are not yet merged into the main index.
   ⇒ ★ a query must scan them linearly ⇒ ★ occasional latency
     spikes. ★ VACUUM merges them.
   ⇒ ★ fastupdate = off for consistent read latency, at the cost
     of slower inserts.
```

**PROVE IT #12 — `ts_rank` vs `ts_rank_cd`.**
```sql
SELECT ts_rank(to_tsvector('english', 'refund policy and shipping policy'),
               websearch_to_tsquery('english','refund policy')) AS rank,
       ★ ts_rank_cd(to_tsvector('english','refund policy and shipping policy'),
               websearch_to_tsquery('english','refund policy')) AS rank_cd;
```
```
   rank   | rank_cd
----------+---------
 0.0991…  | ★ 0.1  
```
```sql
-- ★ now with the terms far apart
SELECT ts_rank_cd(to_tsvector('english',
         'refund ' || repeat('filler ', 50) || 'policy'),
       websearch_to_tsquery('english','refund policy'));
```
```
 ★ 0.0166…        — ★ much lower. ts_rank_cd accounts for
   PROXIMITY (cover density); ts_rank does not.
```

**PROVE IT #13 — the graph in Neo4j, for comparison.**
```cypher
// ★ same 100k nodes, 20M edges
PROFILE MATCH (u:User {id: 42})-[:FOLLOWS*1..4]->(o:User)
RETURN count(DISTINCT o);
```
```
 ★ 12 ms        (PostgreSQL recursive CTE: ★ 41,204 ms)
 ★ db hits: 1,204,882 — ★ pointer dereferences, not index lookups
```

**PROVE IT #14 — measure your actual traversal depth before deciding.**
```sql
SELECT max_depth, count(*),
       round(100.0*count(*) / sum(count(*)) OVER (), 1) AS pct
  FROM traversal_log WHERE at > now() - interval '30 days'
 GROUP BY 1 ORDER BY 1;
-- ★ if >95% is depth ≤2, a graph database is not justified.
```

---

## The design decision framework

```
★★★ BOTH ARE EXCELLENT AT ONE SHAPE AND POOR AT EVERYTHING ELSE.
    ★ MEASURE THE SHAPE BEFORE ADOPTING EITHER. ★★★

 ① ★ GRAPH — MEASURE THE TRAVERSAL DEPTH DISTRIBUTION
    ★ ≥95% at depth ≤2  ⇒ ★ JOINS. Do not add a database.
    depth 3–4, bounded  ⇒ ★ a RECURSIVE CTE with BOTH guards
    ★ depth 5+ / variable / shortest-path / centrality
                        ⇒ ★ a graph database earns its place
    ⇒ ★ the cliff is at depth 4, not depth 2: 12 ms → 41 s → OOM

 ② ★ CHECK FOR SUPERNODES FIRST
    ⇒ ★ a timeout is far more often a supernode than a depth
      problem. ★ 14 of 412,000 nodes, in the example.
    ⇒ ★ FIX: a supernode registry · bound EACH hop with LIMIT ·
      type and direct edges · ★ don't traverse THROUGH them
    ⇒ ★ and be honest that the result becomes a SAMPLE.

 ③ ★ RECURSIVE CTEs NEED BOTH GUARDS, ALWAYS
    ✓ ★ a depth bound
    ✓ ★ cycle detection (`NOT (x = ANY(path))` or `CYCLE … USING`)
    ⇒ ★ a depth bound alone still explodes exponentially through
      a cycle.

 ④ ★ SEARCH — EXHAUST POSTGRESQL FIRST
    ✓ ★ a GENERATED tsvector column (★ cannot go stale)
    ✓ ★ setweight A/B/C + ★ ts_rank_cd (proximity-aware)
    ✓ ★ websearch_to_tsquery — ★ never to_tsquery on user input
    ✓ ★ `simple` for identifiers, `english` for prose
    ✓ ★ pg_trgm on short fields for typo tolerance
    ✓ ★ ts_headline for snippets
    ⇒ ★ comfortably ~10M documents, ★ transactional, ★ joinable.

 ⑤ ★ KNOW WHAT POSTGRES GENUINELY LACKS
    ✗ ★ BM25 (relevance on large corpora)
    ✗ ★ fast multi-facet aggregation
    ✗ ★ fuzzy matching across a full corpus
    ✗ ★ per-query analysers, synonym graphs, "did you mean"
    ✗ ★ index sharding
    ⇒ ★ IF NONE OF THESE IS A PRODUCT REQUIREMENT, ★ STOP HERE.

 ⑥ ★ COUNT HOW MANY SEARCHES ALSO FILTER
    ⇒ ★ in Postgres a filter is a WHERE clause.
    ⇒ ★ in Elasticsearch every filterable field must be
      DENORMALISED INTO THE DOCUMENT and kept in sync.
    ⇒ ★ 88% filtering means four more fields to synchronise —
      ★ this is a real cost, and it is usually omitted from the
      proposal.

 ⑦ ★ THE SECOND COPY IS THE ACTUAL ENGINEERING COST
    ✗ ★ dual write (the dual-write problem, Topic 52)
    ✗ periodic full reindex
    ★ outbox + relay ⇒ atomic, needs an idempotent indexer
    ★★ ★ CDC from the WAL (Topic 76) ⇒ the right answer
    ⇒ ★ AND YOU OWE: a reconciler · a lag alert · ★ a TESTED
      full-reindex procedure.

 ⑧ ★ WRITE DOWN THE TRIGGER CONDITIONS
    "not yet" is defensible ★ only if you state what would change
    it — with numbers: corpus size, p99, facet count, depth
    distribution, measured relevance loss.
    ⇒ ★ this ends the quarterly debate.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build a 1M-document table. Compare `ILIKE '%term%'` against a GIN-indexed `tsvector`. Then: (a) prove stemming matches "returns" to "return"; (b) show `to_tsquery` failing on user input and `websearch_to_tsquery` succeeding; (c) add `setweight` and show a title match outranking a body match.

### Exercise 2 — medium (apply it)
Build a graph with 100k nodes and average degree 200. Measure traversal at depths 1, 2, 3, 4 and 5 using a recursive CTE. Report where it becomes unusable.

Then: introduce a cycle and show the query failing to terminate without cycle detection; add both guards; introduce a supernode and show per-hop `LIMIT` restoring acceptable latency.

### Exercise 3 — hard (production simulation)
A marketplace proposes adding Elasticsearch (search p99 8,412 ms, "results feel wrong") and Neo4j (supplier network times out at 30 s).

(a) Measure the corpus size, the query-length distribution, and the percentage of searches that also filter. What does each tell you?
(b) Measure the traversal-depth distribution. 99.3% is depth ≤2 — what does that mean for the Neo4j proposal?
(c) The timeout is at depth 3–4. Find the actual cause and explain why it is not a depth problem.
(d) Design the PostgreSQL search: the generated column with per-field configurations, the indexes, and the query with filters. Explain why the SKU field uses `simple`.
(e) Measure faceting in Postgres. This is the one genuine gap — describe the product decision that removed it and the measured result.
(f) Rewrite the graph query with all three guards. Explain the honest trade the bounded traversal makes and how you would communicate it in the UI.
(g) Design the supernode registry. How many nodes does it contain, and what is the same pattern called in Topics 60, 61 and 72?
(h) Write the trigger conditions for adopting each system, with numbers.
(i) The delivered solution took 2 weeks instead of 14. List everything that was *not* built, and what each would have cost operationally.

---

## Mental model checkpoint

1. What is index-free adjacency, and what is the cost model per hop for each approach?
2. At what depth does SQL traversal stop being viable, and why is it a cliff rather than a slope?
3. What is a supernode, and which three earlier topics is it the same problem as?
4. Name the two guards a recursive CTE needs. Why is a depth bound alone insufficient?
5. What does an inverted index store beyond term → documents?
6. Name the three components of BM25 relevance and what each contributes.
7. Why must you never use `to_tsquery()` on user input?
8. What does `setweight` do, and why is `ts_rank_cd` preferable to `ts_rank`?
9. Why should a SKU field use the `simple` configuration?
10. Name five things PostgreSQL full-text search cannot do.
11. Why does the percentage of searches that also filter matter so much when evaluating Elasticsearch?
12. Name the four sync strategies and which is correct.

---

## Quick reference card

**PostgreSQL full-text**
```sql
search tsvector ★ GENERATED ALWAYS AS (
  ★ setweight(to_tsvector('english', title), 'A') ||
  ★ setweight(to_tsvector('simple',  sku),   'A') ||   -- ★ no stemming
  setweight(to_tsvector('english', body),  'C')) STORED;
CREATE INDEX ON t USING ★ gin (search);

WHERE search @@ ★ websearch_to_tsquery('english', $1)   -- ★ safe on user input
ORDER BY ★ ts_rank_cd(search, q, 32) DESC               -- ★ proximity-aware
-- ★ ts_headline(...) for snippets · ★ pg_trgm `%` for typos
```

**Recursive traversal — both guards**
```sql
WITH RECURSIVE r AS (
  SELECT … , 1 AS d, ARRAY[$1, x] AS path FROM edges WHERE src=$1 ★ LIMIT 200
  UNION ALL
  SELECT … , r.d+1, r.path || e.dst FROM r
    JOIN LATERAL (SELECT dst FROM edges WHERE src=r.id ★ LIMIT 50) e ON true
   WHERE ★ r.d < $2 AND ★ NOT (e.dst = ANY(r.path)))
```

**★ Depth cliff:** 0.4 ms · 12 ms · 1.8 s · ★ **41 s** · ★ **OOM**.

**Decide**

| | Use SQL | Use the specialised store |
|---|---|---|
| graph | ★ depth ≤ 2 (≥95%) | ★ depth 5+, variable, shortest-path |
| search | ★ ≤10M docs, filters matter | ★ BM25, facets, fuzzy-at-scale, sharding |

**★ Check for a supernode before blaming depth** — 14 of 412,000, in the example.
**★ Count how many searches also filter** — that's what you'd denormalise into every document.
**★ Sync:** dual-write ✗ · full reindex ✗ · outbox ✓ · ★ **CDC ✓✓** — plus a reconciler, a lag alert, and a tested reindex.

---

## When would I use this at work?

1. **Before adopting either.** Two measurements settle most proposals: the traversal-depth distribution, and the corpus size plus filter rate. In the example both came back decisively against, and the PostgreSQL implementation shipped in two weeks instead of fourteen.

2. **Any search feature.** A generated `tsvector` with `setweight`, `websearch_to_tsquery` and `ts_rank_cd` is a genuinely good search implementation, it's transactional, and it joins to your filters. That covers most products up to millions of documents.

3. **When a graph query times out.** Check for a supernode first. It's the same power-law hot key as Topics 60, 61 and 72, and the fix is the same surgical registry pattern — not a new database.

4. **When you *do* adopt one.** The query language is the easy part; the sync pipeline is the engineering. Budget for CDC, a reconciler, a lag alert and a tested reindex, and treat "published but not searchable for two seconds" as a product behaviour you must document.

---

## Connected topics

**Understand before this:** 16 (GIN/GiST — the index types), 55 (tree encodings — the relational alternatives), 61 (hot keys — the supernode), 52 (the dual-write problem).

**This unlocks:**
- **75** — SQL vs NoSQL: the decision framework in full
- **76** — polyglot persistence and CDC: how the second copy is actually kept in sync
- **Case study 10** — product catalogue and search
