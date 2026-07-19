# Topic 66 — Full-Text Search
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you own a library with 500,000 books, and someone walks up to the desk and says:

> "I'm looking for books about **running fast**."

There are two ways you could help them.

**The naive librarian** grabs every book off every shelf and flips through every page looking for the exact letters `running fast`. This takes days. And worse — a book that says "he **ran quickly**" gets missed entirely, because the letters don't match. A book titled "Fast Runners" also gets missed, because the words are in the wrong order. This is what `LIKE '%running fast%'` does: brute-force letter-by-letter scanning, no understanding of language.

**The smart librarian** did something clever years ago. When each book arrived, she read it once and built an **index card** at the back of a giant filing cabinet. On the card for the word "run" she wrote down every book that mentions running — and here's the trick — she also filed "running", "ran", and "runner" under that **same card**, because they're all the same idea. She dropped useless words like "the", "a", "is" entirely — nobody searches for those. Now when you ask for "running fast", she reduces your request to two root ideas — `run` and `fast` — walks to two index cards, and reads off the books that appear on **both** cards. Milliseconds, not days.

That filing cabinet is a **`tsvector`** (the pre-digested, indexed form of a document). Your question, reduced to root ideas, is a **`tsquery`**. The act of checking whether a document's card matches your query is the **`@@` operator**. And the process of chopping "running" down to "run" and throwing away "the" is called **stemming** and **stop-word removal** — together, the **text search configuration**.

That's PostgreSQL Full-Text Search. It's not "search inside a string." It's "understand the language once, index the meaning, then answer questions about meaning instantly."

---

## 2. Connection to SQL Internals

Full-text search (FTS) touches more internals than almost any other feature, because it introduces two custom data types, custom operators, and a specialized index type.

**The `tsvector` type — a sorted lexeme map.** A `tsvector` is not free text. Internally it is a **sorted array of lexemes**, each with an optional list of integer positions and weight labels. `to_tsvector('english', 'The fat cats sat')` produces `'cat':3 'fat':2 'sat':4` — normalized (lowercased, stemmed), stop-words removed, positions retained. Because the lexemes are stored sorted, matching a query lexeme against a document is a binary search, not a linear scan.

**The GIN index — an inverted index.** The Generalized Inverted Index (GIN) is the workhorse. Conceptually it inverts the mapping: instead of `document → lexemes`, it stores `lexeme → list of row TIDs (heap tuple pointers)`. This is exactly the "index card" from the analogy. A GIN index is a **B-tree over lexemes**, where each leaf points to a compressed **posting list** (or posting tree, for very common lexemes) of the heap tuples containing that lexeme. When you search for `run & fast`, GIN fetches the posting list for `run`, the posting list for `fast`, and intersects them. This is why FTS scales to millions of documents.

**The GiST index — a lossy signature tree.** The Generalized Search Tree (GiST) alternative stores a **fixed-length signature (a bitmap hash)** of each document's lexemes in a balanced tree. It is lossy: a signature collision means the index says "maybe match", forcing a **recheck** against the heap. GiST is smaller and faster to update, but produces false positives that cost heap fetches.

**MVCC and index bloat.** Like every PostgreSQL index, GIN indexes obey MVCC — an `UPDATE` to a document writes a new heap tuple and new index entries; old entries linger until `VACUUM`. GIN has a special **pending list** (`fastupdate`) that buffers inserts to avoid the cost of updating the main index on every write, flushed in bulk. This makes writes cheaper but can make reads temporarily scan the unsorted pending list.

**The planner and the `@@` operator.** `@@` is just an operator with an associated **operator class** (`gin_tsvector_ops`, `gist_tsvector_ops`). The planner treats a `tsvector @@ tsquery` predicate like any other indexable condition: if a matching GIN index exists and the cost model favors it, you get a `Bitmap Index Scan` on the GIN index feeding a `Bitmap Heap Scan`. Ranking functions (`ts_rank`) are **not** indexable — they run after the heap fetch, in the same phase as any expression in the target list.

---

## 3. Logical Execution Order Context

```
FROM documents
WHERE tsv @@ query      ← the @@ match runs in the WHERE phase (index-accelerated)
GROUP BY ...
HAVING ...
SELECT ts_rank(tsv, query) AS rank,   ← ranking computed in SELECT phase, per surviving row
       ts_headline(body, query)        ← highlighting also SELECT phase, expensive
ORDER BY rank DESC       ← sort AFTER ranking is computed
LIMIT 20                 ← the classic "top-N search results" shape
```

The critical ordering facts for FTS:

- **`@@` runs in `WHERE`** — it is a filter, and it is the *only* part of the pipeline that a GIN/GiST index can accelerate. Get the filter indexed and selective, and everything downstream operates on a small surviving set.
- **`ts_rank` / `ts_rank_cd` run in `SELECT`, after filtering.** Ranking is a per-row scalar function; it cannot use an index. If your `@@` filter survives 2 million rows, you compute rank 2 million times, then sort 2 million rows for `ORDER BY rank`. This is the number-one FTS performance mistake — a non-selective query turns ranking into a full-set sort.
- **`ts_headline` runs in `SELECT`, and it re-parses the original document text.** It is dramatically more expensive than ranking because it does not use the `tsvector` — it re-tokenizes the raw `body` column to find fragment boundaries. Never compute `ts_headline` before `LIMIT`; compute it only for the final page of results.
- **`ORDER BY rank ... LIMIT n`** is the canonical shape. Because ranking is not indexable, PostgreSQL must compute rank for every matched row before it can sort. The mitigation is making `@@` selective enough that "every matched row" is small — or using RUM indexes / `<=>` distance operators (an extension) that can return rows in rank order directly.

---

## 4. What Is Full-Text Search?

Full-text search is a language-aware search system built into PostgreSQL that matches **normalized lexemes** rather than raw character substrings. It converts documents into `tsvector` values and queries into `tsquery` values, matches them with the `@@` operator, and accelerates the match with an inverted (GIN) or signature (GiST) index. Unlike `LIKE`, it understands word boundaries, stemming (running → run), stop words, and boolean/phrase query logic.

```sql
SELECT id, title
FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('english', 'running & fast')
ORDER BY ts_rank(to_tsvector('english', body), to_tsquery('english', 'running & fast')) DESC
LIMIT 20;
```

```
to_tsvector('english', title || ' ' || body)
│                       │
│                       └── the document: concatenated text, cast to lexemes
└── text search configuration: controls stemming + stop-word list + parser
    │
    └── 'english' → Snowball English stemmer, English stop words
        (other options: 'simple' = no stemming/no stopwords, 'spanish', 'german', ...)

@@   ← the MATCH operator: returns TRUE if the tsvector satisfies the tsquery

to_tsquery('english', 'running & fast')
│                      │       │
│                      │       └── & = AND. Also: | = OR, ! = NOT, <-> = FOLLOWED BY (phrase)
│                      └── each term is stemmed with the SAME config → 'run' & 'fast'
└── same config MUST be used for query and document, or matches silently fail

ts_rank(tsvector, tsquery)
│       │
│       └── ranks by term frequency + proximity + weights → higher = more relevant
└── NOT index-accelerated; computed per surviving row in the SELECT phase
```

### The core building blocks

| Piece | Type | Role |
|-------|------|------|
| `to_tsvector(config, text)` | `text → tsvector` | Parse + stem + de-stop-word a document into sorted lexemes |
| `to_tsquery(config, text)` | `text → tsquery` | Parse a **structured** boolean query (`&`, `\|`, `!`, `<->`) |
| `plainto_tsquery(config, text)` | `text → tsquery` | Turn plain words into an AND query (ignores operators) |
| `phraseto_tsquery(config, text)` | `text → tsquery` | Turn text into a **phrase** query (words must be adjacent) |
| `websearch_to_tsquery(config, text)` | `text → tsquery` | Google-style: quotes, `or`, `-` exclusion; never errors |
| `@@` | `tsvector × tsquery → bool` | The match operator |
| `ts_rank` / `ts_rank_cd` | `→ float4` | Relevance ranking |
| `ts_headline` | `→ text` | Highlight matched terms with `<b>...</b>` |
| `setweight` | `tsvector → tsvector` | Tag lexemes A/B/C/D for field-boosted ranking |

---

## 5. Why Full-Text Search Mastery Matters in Production

1. **`LIKE '%term%'` cannot use a standard B-tree index and does not understand language.** A leading-wildcard `LIKE` forces a sequential scan of every row. On a 5M-row articles table that is a multi-second query on every keystroke of a search box. FTS with a GIN index answers the same query in single-digit milliseconds *and* matches "ran" when the user typed "running."

2. **The config-mismatch bug is silent and total.** If you build the `tsvector` with `'english'` but query with `'simple'` (or the default `default_text_search_config` differs between your migration and your runtime), matches return **zero rows with no error**. Whole search features ship broken because the two configs stemmed the word differently. Mastery means always pinning the config explicitly on both sides.

3. **Ranking on a non-selective filter melts CPU.** A search for a common word ("the", "data", "order") that survives millions of rows forces `ts_rank` + sort over the whole matched set. Search endpoints that were fast in staging fall over in production when a user searches a common term. Understanding that ranking is post-filter and non-indexable is what lets you cap it.

4. **`ts_headline` on unbounded result sets is a latency bomb.** Because it re-parses raw document text (not the tsvector), computing headlines for 10,000 matches before `LIMIT` can be 100× slower than the search itself. Knowing to compute it only on the final page is production-critical.

5. **Choosing FTS vs `pg_trgm` vs Elasticsearch is an architecture decision.** FTS is great for word/language matching; it is *bad* at typo tolerance and substring/autocomplete (that's `pg_trgm`); and it does not scale to the relevance tuning, faceting, and horizontal sharding of Elasticsearch. Picking wrong means either reinventing a search engine in SQL or dragging in an entire cluster you didn't need.

6. **Generated columns + GIN is the correct production pattern.** Hand-maintaining a `tsvector` column via triggers is error-prone; a `GENERATED ALWAYS AS ... STORED` column keeps it perfectly in sync with the source text and lets the GIN index stay simple. Not knowing this leads to stale search indexes — documents that were edited but still match their old text.

---

## 6. Deep Technical Content

### 6.1 The `tsvector`: what a document actually becomes

`to_tsvector` runs the text through the configuration's **parser** (splits into tokens: words, numbers, URLs, emails, etc.), then each token type is fed through the config's **dictionaries** (stemmers, stop-word lists, synonym maps). The output is a sorted set of lexemes with positions.

```sql
SELECT to_tsvector('english', 'The fat cats sat on the mats, running fast.');
--                'cat':3 'fast':9 'fat':2 'mat':7 'run':8 'sat':4
```

Observe:
- `the`, `on` — **stop words**, dropped entirely.
- `cats` → `cat`, `mats` → `mat`, `running` → `run` — **stemmed** to root lexemes.
- Numbers next to each lexeme are **positions** in the original text (used for phrase matching and proximity ranking). `cat` was the 3rd word.
- The lexemes are stored **sorted alphabetically**, not in document order.

Compare the `simple` configuration, which does no stemming and no stop-word removal:

```sql
SELECT to_tsvector('simple', 'The fat cats sat on the mats, running fast.');
-- 'cats':3 'fast':10 'fat':2 'mats':7 'on':5 'running':9 'sat':4 'the':1,6
```

`simple` is the right choice for exact-token fields: usernames, tags, SKUs, country codes — anything where "running" must not match "run" and "the" is meaningful.

### 6.2 The `tsquery`: structured search logic

A `tsquery` is a boolean tree of lexemes. The operators:

| Operator | Meaning | Example |
|----------|---------|---------|
| `&` | AND | `run & fast` — both must be present |
| `\|` | OR | `run \| jog` — either |
| `!` | NOT | `run & !marathon` — run but not marathon |
| `<->` | FOLLOWED BY (phrase, distance 1) | `fast <-> car` — "fast" immediately before "car" |
| `<N>` | FOLLOWED BY at distance N | `fast <2> car` — two positions apart |
| `( )` | grouping | `(run \| jog) & fast` |

```sql
SELECT to_tsquery('english', 'running & !(marathon | sprint)');
-- 'run' & !( 'marathon' | 'sprint' )   ← note 'running' stemmed to 'run'
```

**Critical:** `to_tsquery` requires **valid syntax** and will raise an error on user input containing stray punctuation, unbalanced parens, or bare spaces:

```sql
SELECT to_tsquery('english', 'running fast');
-- ERROR:  syntax error in tsquery: "running fast"
```

This is why you never feed raw user input to `to_tsquery`. Use one of the plain/websearch variants below.

### 6.3 The four query-builder functions — when to use each

```sql
-- to_tsquery: full operator syntax; ERRORS on invalid input. For trusted/programmatic queries.
SELECT to_tsquery('english', 'cat & dog');          -- 'cat' & 'dog'

-- plainto_tsquery: plain words → AND'd together; operators are IGNORED (treated as words). Never errors.
SELECT plainto_tsquery('english', 'cat dog running'); -- 'cat' & 'dog' & 'run'
SELECT plainto_tsquery('english', 'cat & dog');       -- 'cat' & 'dog'  (the & is dropped as noise)

-- phraseto_tsquery: plain words → phrase (adjacency required). Uses <-> between words.
SELECT phraseto_tsquery('english', 'fast running cars'); -- 'fast' <-> 'run' <-> 'car'

-- websearch_to_tsquery: Google-style syntax; the ONLY one safe for raw user input with intent.
SELECT websearch_to_tsquery('english', 'cat "running fast" or dog -marathon');
-- 'cat' & 'run' <-> 'fast' & 'dog' & !'marathon'   (roughly; quotes = phrase, or = |, - = !)
```

**Decision guide:**
- Raw search box → `websearch_to_tsquery` (handles quotes, `or`, `-`, never throws).
- Simple "all these words" → `plainto_tsquery`.
- Exact phrase → `phraseto_tsquery`.
- You are constructing operators yourself in code → `to_tsquery` (and you own escaping).

### 6.4 The `@@` operator and its overloads

`@@` has several forms so you can skip explicit conversions:

```sql
tsvector @@ tsquery   -- the canonical form
text     @@ tsquery   -- implicitly to_tsvector(get_current_ts_config(), text) — uses DEFAULT config
text     @@ text      -- both sides implicitly converted with the default config
tsquery  @@ tsvector  -- commutative variant
```

The `text @@ ...` forms use `get_current_ts_config()` (the `default_text_search_config` GUC). **This is a trap in production**: if two servers or two sessions have different defaults, the same query returns different results. Always write the explicit `tsvector @@ tsquery` form with the config named on both sides.

### 6.5 Text search configurations and stemming

A configuration binds a **parser** to a chain of **dictionaries** per token type. `english` uses the Snowball stemmer; `simple` uses none. Inspect what a config does to a token:

```sql
SELECT * FROM ts_debug('english', 'The running cats');
-- alias   | description | token   | dictionaries  | dictionary   | lexemes
-- asciiword | Word...   | The     | {english_stem}| english_stem | {}       ← stop word → empty
-- asciiword | Word...   | running | {english_stem}| english_stem | {run}
-- asciiword | Word...   | cats    | {english_stem}| english_stem | {cat}
```

You can build a **custom configuration**, e.g. English stemming plus a synonym dictionary or an `unaccent` filter so "café" matches "cafe":

```sql
CREATE TEXT SEARCH CONFIGURATION public.english_unaccent ( COPY = pg_catalog.english );
ALTER TEXT SEARCH CONFIGURATION public.english_unaccent
  ALTER MAPPING FOR hword, hword_part, word
  WITH unaccent, english_stem;

SELECT to_tsvector('public.english_unaccent', 'Café Résumé running');
-- 'cafe':1 'resume':2 'run':3
```

**Stemming caveats to internalize:**
- Stemming is algorithmic, not dictionary-perfect. `universe` and `university` both stem to `univers` — a false conflation. "Better" does not stem to "good" (that would need a synonym/thesaurus dictionary).
- Stemming is language-specific. Applying `english` to German text mangles it. Store a language column and pick the config per row for multilingual corpora (`to_tsvector(doc.lang::regconfig, doc.body)`).

### 6.6 Weighting fields with `setweight`

Real documents have structure — a match in the **title** should rank higher than a match in the **body**. `setweight` labels every lexeme in a `tsvector` with a class `A`, `B`, `C`, or `D` (A is highest). Ranking functions apply per-class weights.

```sql
SELECT
  setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
  setweight(to_tsvector('english', coalesce(summary,'')), 'B') ||
  setweight(to_tsvector('english', coalesce(body,'')), 'C')
FROM articles;
-- title lexemes tagged :A, summary :B, body :C — concatenated into one tsvector
```

The `||` operator concatenates tsvectors (merging position lists). At rank time you pass a weight array `{D-weight, C-weight, B-weight, A-weight}`:

```sql
ts_rank('{0.1, 0.2, 0.4, 1.0}', doc_tsv, query)
--       D    C    B    A       ← array order is D,C,B,A (reverse alphabetical!)
```

### 6.7 Ranking: `ts_rank` vs `ts_rank_cd`

Both return a `float4` relevance score; neither is index-accelerated.

- **`ts_rank`** — weights matches by frequency and field weight. More occurrences of query terms → higher score. Does not consider proximity.
- **`ts_rank_cd`** — "cover density" ranking. Rewards query terms appearing **close together** (a tight cover). Requires positional information in the tsvector (positions are lost if you strip them, e.g. with `strip()`). Better for phrase-like relevance.

```sql
SELECT id,
       ts_rank(tsv, q)    AS rank_freq,
       ts_rank_cd(tsv, q) AS rank_proximity
FROM articles, websearch_to_tsquery('english', 'machine learning') q
WHERE tsv @@ q
ORDER BY rank_proximity DESC
LIMIT 10;
```

**Normalization flag** (third argument, a bitmask) controls document-length normalization — e.g. `32` divides rank by `rank+1` (0..1 scale), `1`/`2` divide by document length so long documents don't dominate purely by having more words:

```sql
ts_rank(tsv, q, 32)      -- normalize into (0,1)
ts_rank(tsv, q, 1|32)    -- also divide by 1 + log(document length)
```

### 6.8 Highlighting with `ts_headline`

`ts_headline` returns a snippet of the **original text** with matched terms wrapped in markup. It does **not** use the tsvector — it re-parses the raw text — which is why it is expensive and why it needs the original `body`, not just `tsv`.

```sql
SELECT ts_headline(
  'english',
  body,
  websearch_to_tsquery('english', 'machine learning'),
  'StartSel=<mark>, StopSel=</mark>, MaxWords=35, MinWords=15, MaxFragments=2'
)
FROM articles
WHERE tsv @@ websearch_to_tsquery('english', 'machine learning')
ORDER BY ts_rank(tsv, websearch_to_tsquery('english', 'machine learning')) DESC
LIMIT 10;   -- headline computed for at most 10 rows, AFTER ranking + limit
```

Options: `StartSel`/`StopSel` (markup), `MaxWords`/`MinWords` (fragment size), `MaxFragments` (number of snippets; 0 = one contiguous window), `HighlightAll` (highlight whole doc).

### 6.9 The generated `tsvector` column — the production pattern

Do not compute `to_tsvector(...)` in the `WHERE` clause of every query (it re-parses the document each time and can only use an *expression* index). Instead, materialize the tsvector into a stored, generated column and index that.

```sql
ALTER TABLE articles
  ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')),   'A') ||
    setweight(to_tsvector('english', coalesce(body,'')),    'B')
  ) STORED;

CREATE INDEX articles_tsv_gin ON articles USING GIN (tsv);
```

`GENERATED ALWAYS AS ... STORED` (PostgreSQL 12+) recomputes the tsvector automatically on every INSERT/UPDATE — it can never drift from the source columns, and you cannot forget a trigger. The generation expression must be `IMMUTABLE`; `to_tsvector('english', ...)` with a **literal** config is immutable, but `to_tsvector(col::regconfig, ...)` (dynamic config) is **not**, so multilingual tables must fall back to a trigger.

**Pre-12 / dynamic-config alternative — a trigger:**

```sql
CREATE FUNCTION articles_tsv_trigger() RETURNS trigger AS $$
BEGIN
  NEW.tsv :=
    setweight(to_tsvector('english', coalesce(NEW.title,'')), 'A') ||
    setweight(to_tsvector('english', coalesce(NEW.body,'')),  'B');
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_tsv_update BEFORE INSERT OR UPDATE
  ON articles FOR EACH ROW EXECUTE FUNCTION articles_tsv_trigger();
```

### 6.10 GIN vs GiST — the index choice

| Property | GIN | GiST |
|----------|-----|------|
| Structure | Inverted index (lexeme → posting list) | Signature tree (lossy bitmap) |
| Lookup speed | Faster (exact posting lists) | Slower (lossy → heap rechecks) |
| Build/size | Larger, slower to build | Smaller, faster to build |
| Update cost | Higher (mitigated by `fastupdate` pending list) | Lower |
| False positives | None | Yes → recheck against heap |
| Best for | Static/read-heavy documents (the common case) | Write-heavy, or small/volatile datasets |

**Default to GIN.** It is the right choice for the overwhelming majority of FTS workloads (documents are read far more than written). Reach for GiST only when write amplification from GIN is measured and painful, or the dataset is small and churny.

GIN tuning knobs:
- `fastupdate = on` (default): buffers inserts in a pending list, flushed in bulk — cheaper writes, occasionally slower reads.
- `gin_pending_list_limit`: size of that pending list before a forced flush.
- Consider the **RUM** extension (not core) if you need index-ordered ranking (`tsvector <=> tsquery` distance) or positional queries returned in rank order without a post-sort.

### 6.11 What FTS deliberately does *not* do

- **No substring / infix matching.** FTS matches whole lexemes. Searching `run` will not find `overrun` or `runny`. For substring/autocomplete, use `pg_trgm` trigram indexes.
- **No typo tolerance.** `runing` (misspelled) stems to `rune`/itself and will not match `running`. Fuzzy matching is `pg_trgm` similarity or `levenshtein`.
- **No relevance learning / faceting / distributed sharding.** That is Elasticsearch/OpenSearch territory.
- **Prefix matching is limited.** `to_tsquery('run:*')` does lexeme-prefix matching (`run:*` matches `run`, `running`→`run`, but prefix is on the *stemmed* lexeme), which is coarse compared to trigram autocomplete.

---

## 7. EXPLAIN — Full-Text Search in the Plan

### The good plan — GIN Bitmap Index Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title
FROM articles
WHERE tsv @@ websearch_to_tsquery('english', 'machine learning')
LIMIT 20;
```

```
Limit  (cost=68.20..92.51 rows=20 width=45)
       (actual time=1.42..1.88 rows=20 loops=1)
  ->  Bitmap Heap Scan on articles  (cost=68.20..1284.90 rows=1000 width=45)
                                     (actual time=1.41..1.85 rows=20 loops=1)
        Recheck Cond: (tsv @@ '''machin'' & ''learn'''::tsquery)
        Heap Blocks: exact=18
        ->  Bitmap Index Scan on articles_tsv_gin
                (cost=0.00..67.95 rows=1000 width=0)
                (actual time=1.05..1.05 rows=1240 loops=1)
              Index Cond: (tsv @@ '''machin'' & ''learn'''::tsquery)
  Buffers: shared hit=42
Planning Time: 0.31 ms
Execution Time: 1.95 ms
```

**Reading it:**
- `Bitmap Index Scan on articles_tsv_gin` — the GIN index fetched the posting lists for lexemes `machin` and `learn` and intersected them → a bitmap of 1,240 candidate TIDs. Note the query terms are shown **stemmed** (`machin`, `learn`).
- `Bitmap Heap Scan` — those TIDs are fetched from the heap. `Recheck Cond` re-verifies `@@` on each fetched row (for GIN this recheck is usually a formality; for GiST it filters real false positives).
- `Heap Blocks: exact=18` — only 18 heap pages touched, because `LIMIT 20` stopped early.
- Total 1.95 ms for a full-text search — this is the target.

### The bad plan — Seq Scan with on-the-fly to_tsvector

```sql
EXPLAIN (ANALYZE)
SELECT id, title
FROM articles
WHERE to_tsvector('english', body) @@ plainto_tsquery('english', 'machine learning');
-- No index on the expression, or query doesn't match the indexed expression
```

```
Seq Scan on articles  (cost=0.00..184530.00 rows=5000 width=45)
                      (actual time=0.09..3182.44 rows=4870 loops=1)
  Filter: (to_tsvector('english', body) @@ '''machin'' & ''learn'''::tsquery)
  Rows Removed by Filter: 995130
Planning Time: 0.12 ms
Execution Time: 3184.10 ms
```

**Reading it:**
- `Seq Scan` — every one of 1,000,000 rows is read and `to_tsvector('english', body)` is **computed on the fly** for each, then matched. 3.18 **seconds**.
- `Rows Removed by Filter: 995130` — 99.5% of the work was thrown away.
- The fix: index a stored/generated `tsv` column (or create a matching expression index `CREATE INDEX ... USING GIN (to_tsvector('english', body))` and query the *exact same expression*).

### The ranking cost — non-selective query

```sql
EXPLAIN (ANALYZE)
SELECT id, ts_rank(tsv, q) AS rank
FROM articles, plainto_tsquery('english', 'data') q
WHERE tsv @@ q
ORDER BY rank DESC
LIMIT 20;
```

```
Limit  (actual time=612.4..612.5 rows=20 loops=1)
  ->  Sort  (actual time=612.4..612.4 rows=20 loops=1)
        Sort Key: (ts_rank(articles.tsv, q.q)) DESC
        Sort Method: top-N heapsort  Memory: 28kB
        ->  Nested Loop  (actual time=2.1..541.7 rows=420000 loops=1)
              ->  Function Scan on q  (actual rows=1 loops=1)
              ->  Bitmap Heap Scan on articles  (actual rows=420000 loops=1)
                    Recheck Cond: (tsv @@ q.q)
                    ->  Bitmap Index Scan on articles_tsv_gin (actual rows=420000 loops=1)
Execution Time: 613.02 ms
```

**Reading it:**
- The GIN scan is fine, but `data` is a **common lexeme** — 420,000 rows survive `@@`.
- `ts_rank` is then computed for **all 420,000** rows, and `top-N heapsort` sorts them. 613 ms — dominated by ranking + sort, not the index.
- The lesson: ranking cost is proportional to matched-row count, not result-page size. Selectivity of `@@` is what controls ranking cost. Mitigations: require more/rarer terms, pre-filter by category/date, or cap candidates with a subquery `LIMIT` before ranking.

---

## 8. Query Examples

### Example 1 — Basic: search a single column

```sql
-- Find articles mentioning "postgres" (stemmed) in the body.
-- Uses the default config via the text @@ text form — fine for a quick query,
-- but pin the config explicitly in real code (see Example 3).
SELECT id, title
FROM articles
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'postgres')
ORDER BY id
LIMIT 25;
```

### Example 2 — Intermediate: weighted multi-field search with ranking and a user-safe query

```sql
-- Search title (weight A) + body (weight B), rank by relevance, take top 20.
-- websearch_to_tsquery makes raw user input safe (quotes, or, -exclusion, never errors).
WITH q AS (
  SELECT websearch_to_tsquery('english', 'distributed "consensus protocol" -blockchain') AS query
)
SELECT
  a.id,
  a.title,
  ts_rank(
    '{0.1, 0.2, 0.4, 1.0}',                      -- D,C,B,A weights: title matches count 2.5x body
    setweight(to_tsvector('english', a.title), 'A') ||
    setweight(to_tsvector('english', a.body),  'B'),
    q.query
  ) AS rank
FROM articles a, q
WHERE (setweight(to_tsvector('english', a.title), 'A') ||
       setweight(to_tsvector('english', a.body),  'B')) @@ q.query
ORDER BY rank DESC
LIMIT 20;
-- NOTE: this recomputes tsvectors per row — correct but slow. Example 3 fixes that.
```

### Example 3 — Production Grade: generated column, GIN index, ranked + highlighted, paginated

```sql
-- Table: articles ~8M rows, avg body ~2KB.
-- Index: GIN on generated column articles.tsv (built once, ~1.2GB).
-- Perf expectation: @@ filter + rank + headline for a selective 2-term query: 5–20ms.
--
-- One-time schema (from section 6.9):
--   ALTER TABLE articles ADD COLUMN tsv tsvector GENERATED ALWAYS AS (
--     setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
--     setweight(to_tsvector('english', coalesce(body,'')),  'B')) STORED;
--   CREATE INDEX articles_tsv_gin ON articles USING GIN (tsv);

WITH q AS (
  SELECT websearch_to_tsquery('english', $1) AS query   -- $1 = raw user search string
),
matched AS (                                             -- filter + rank FIRST, cheaply
  SELECT a.id, a.title, a.body, ts_rank_cd(a.tsv, q.query) AS rank
  FROM articles a, q
  WHERE a.tsv @@ q.query                                 -- GIN-accelerated
    AND a.published_at >= now() - interval '2 years'     -- extra selectivity, uses btree
  ORDER BY rank DESC
  LIMIT 20 OFFSET $2                                     -- paginate BEFORE highlighting
)
SELECT
  m.id,
  m.title,
  m.rank,
  ts_headline('english', m.body, q.query,                -- headline only for the 20 shown rows
    'StartSel=<mark>, StopSel=</mark>, MaxFragments=2, MaxWords=30, MinWords=12') AS snippet
FROM matched m, q
ORDER BY m.rank DESC;
```

```
-- EXPLAIN (ANALYZE, BUFFERS) sketch for the matched CTE on a selective query:
-- Limit (actual time=6.1..6.4 rows=20)
--   ->  Sort (Sort Method: top-N heapsort  Memory: 30kB)
--         ->  Bitmap Heap Scan on articles (actual rows=3100)
--               Recheck Cond: (tsv @@ ...)
--               Filter: (published_at >= ...)
--               ->  Bitmap Index Scan on articles_tsv_gin (actual rows=3400)
-- Execution Time: ~12 ms  (headline adds a few ms for 20 rows only)
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Config mismatch between document and query — silent zero rows

```sql
-- WRONG: document built with 'english', query built with 'simple'
SELECT id FROM articles
WHERE to_tsvector('english', body) @@ to_tsquery('simple', 'running');
-- 'english' stems the document's "running" → 'run'
-- 'simple'  leaves the query as → 'running'
-- 'run' never equals 'running' → ZERO ROWS, no error, no warning.
```

**Why it's wrong at the execution level:** the `@@` operator does a lexeme-set membership test. The document side contains the lexeme `run`; the query side asks for the lexeme `running`. They are different strings in the sorted lexeme array, so the binary search finds no match. Nothing errors because both sides are individually valid tsvector/tsquery values.

```sql
-- RIGHT: identical config on both sides (and pin it explicitly, never rely on defaults)
SELECT id FROM articles
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'running');
-- both stem to 'run' → matches.
```

### Wrong 2: Feeding raw user input to `to_tsquery` — runtime errors on every space

```sql
-- WRONG: user types "machine learning" (with a space) into a search box
SELECT id FROM articles WHERE tsv @@ to_tsquery('english', 'machine learning');
-- ERROR: syntax error in tsquery: "machine learning"
-- to_tsquery requires operators between terms; a bare space is illegal.
-- Your search endpoint 500s the moment a user types two words.
```

**Why it's wrong:** `to_tsquery` parses a *structured* query grammar. Two lexemes with no operator between them is a syntax error, as are stray `-`, `'`, `:`, or unbalanced `(`. User input is unstructured, so it routinely violates the grammar.

```sql
-- RIGHT: websearch_to_tsquery never errors and gives users Google-like semantics
SELECT id FROM articles WHERE tsv @@ websearch_to_tsquery('english', 'machine learning');
-- → 'machin' & 'learn'. Quotes give phrases, "or" gives |, "-x" gives !x. Never throws.
```

### Wrong 3: `to_tsvector()` in WHERE that doesn't match the index — Seq Scan

```sql
-- WRONG: GIN index is on the stored column tsv, but the query re-derives a tsvector
SELECT id FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ websearch_to_tsquery('english', 'kafka');
-- The planner cannot use articles_tsv_gin because the WHERE expression
-- (to_tsvector of title||body) is NOT the same expression the index was built on (the tsv column).
-- Result: Seq Scan, to_tsvector recomputed for every row. Seconds on a large table.
```

**Why it's wrong:** an index can only be used when the query predicate matches the index's indexed expression *exactly*. The stored `tsv` column and an ad-hoc `to_tsvector(title || ' ' || body)` are different expressions, so the planner has no indexable path and falls back to a sequential scan with per-row parsing.

```sql
-- RIGHT: query the exact indexed column
SELECT id FROM articles
WHERE tsv @@ websearch_to_tsquery('english', 'kafka');
-- Uses articles_tsv_gin → Bitmap Index Scan. Milliseconds.
```

### Wrong 4: `ts_headline` computed before LIMIT — latency explosion

```sql
-- WRONG: headline computed for every matched row, then sorted, then limited
SELECT id, title,
       ts_headline('english', body, q.query) AS snippet,
       ts_rank(tsv, q.query) AS rank
FROM articles, websearch_to_tsquery('english', 'data') q
WHERE tsv @@ q.query
ORDER BY rank DESC
LIMIT 20;
-- 'data' matches 420,000 rows. ts_headline re-parses the RAW body of all 420,000
-- rows (it does not use tsv) BEFORE the LIMIT throws 419,980 of them away. Multi-second.
```

**Why it's wrong at the execution level:** `ts_headline` is in the `SELECT` list, which is evaluated for every row that survives `WHERE`, before `ORDER BY`/`LIMIT`. And unlike ranking, it re-tokenizes the original document text, so it is the single most expensive per-row operation in the query. Doing it 420,000 times to display 20 snippets is pure waste.

```sql
-- RIGHT: rank + limit in a subquery, headline only the final 20 rows
WITH q AS (SELECT websearch_to_tsquery('english', 'data') AS query),
top20 AS (
  SELECT id, title, body, ts_rank(tsv, q.query) AS rank
  FROM articles, q
  WHERE tsv @@ q.query
  ORDER BY rank DESC
  LIMIT 20
)
SELECT t.id, t.title, t.rank,
       ts_headline('english', t.body, q.query) AS snippet
FROM top20 t, q;
-- ts_headline runs 20 times, not 420,000. Orders of magnitude faster.
```

### Wrong 5: Using FTS for autocomplete / substring — wrong tool

```sql
-- WRONG: trying to autocomplete "postg" as the user types, expecting to match "postgresql"
SELECT id FROM articles WHERE tsv @@ to_tsquery('english', 'postg');
-- 'postg' is a lexeme that does not exist in any document (docs have 'postgresql'/'postgr').
-- Zero rows. FTS matches whole lexemes, not prefixes/substrings (except limited :* prefix).
```

**Why it's wrong:** FTS indexes complete lexemes. `postg` is not a lexeme anyone's document contains; it is a fragment. Even `to_tsquery('postg:*')` prefix-matching operates on stemmed lexemes and is coarse. Substring/prefix search is a fundamentally different index problem.

```sql
-- RIGHT: use pg_trgm for substring/prefix/autocomplete
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX articles_title_trgm ON articles USING GIN (title gin_trgm_ops);

SELECT id, title FROM articles
WHERE title ILIKE '%postg%'          -- trigram GIN index accelerates even leading-wildcard ILIKE
ORDER BY similarity(title, 'postg') DESC
LIMIT 10;
```

---

## 10. Performance Profile

### Cost model by phase

| Phase | Indexable? | Cost driver |
|-------|-----------|-------------|
| `@@` filter | Yes (GIN/GiST) | Selectivity of the query lexemes (rare lexemes = tiny posting lists) |
| `ts_rank` / `ts_rank_cd` | No | Number of rows surviving `@@` (computed once per matched row) |
| `ORDER BY rank` | No (can't index rank) | Sort over the matched set (top-N heapsort if LIMIT) |
| `ts_headline` | No | Matched rows × raw document size (re-parses original text) |

### Scaling behavior

| Rows | `@@` on GIN, selective (rare 2-term query) | `@@` + rank on a common term |
|------|-------------------------------------------|------------------------------|
| 1M | 1–5 ms | 50–300 ms (rank dominates) |
| 10M | 3–15 ms | 300 ms–2 s (rank + sort dominates) |
| 100M | 10–50 ms (posting lists still small) | seconds — must add extra selectivity |

The headline number: **GIN `@@` scales sublinearly with table size** because it fetches only the posting lists of the queried lexemes — a rare 2-term AND query touches roughly the same number of rows at 100M as at 1M. What does *not* scale is ranking a common term: if `@@` survives N rows, you pay O(N) ranking + O(N log k) sort regardless of index quality.

### Memory and CPU

- **GIN index size** is typically 1–3× the size of the tsvector data — substantial. A 8M-row article table can carry a 1–2 GB GIN index. Budget disk and shared_buffers accordingly.
- **Build time**: GIN builds are CPU-heavy (parsing + sorting all lexemes). Build with elevated `maintenance_work_mem` (e.g. 2GB) and consider `CREATE INDEX CONCURRENTLY` to avoid locking writes.
- **Write amplification**: every INSERT/UPDATE to the source columns recomputes the tsvector (CPU) and updates the GIN index (I/O). `fastupdate=on` batches the index updates via the pending list. High-write tables should monitor pending-list flush cost.
- **Ranking is pure CPU** on the matched set — no I/O once rows are in the buffer pool. This is why a common-term search saturates a CPU core rather than the disk.

### Optimization techniques specific to FTS

1. **Increase selectivity before ranking.** Combine `@@` with a cheap, indexed predicate (`published_at >`, `category_id =`, `status =`) so fewer rows reach `ts_rank`. Often the biggest win.
2. **Cap candidates.** Rank a bounded candidate set: pull the first, say, 1,000 `@@` matches (any order) in a subquery, rank only those. Trades perfect global ranking for bounded latency; acceptable for most search UIs.
3. **Store `ts_rank` weights in the tsvector via `setweight`**, not in the rank call, so a single GIN index serves weighted ranking.
4. **`ts_rank_cd` needs positions** — do not `strip()` positions from stored tsvectors if you want proximity ranking.
5. **Consider RUM indexes** (extension) when `ORDER BY rank LIMIT n` on large matched sets is the bottleneck — RUM can return rows in rank order from the index, eliminating the post-filter sort.
6. **Partial GIN index** for hot subsets: `CREATE INDEX ... USING GIN (tsv) WHERE status = 'published'` keeps the index smaller and faster when most searches only hit published rows.
7. **`work_mem`** must be large enough that the top-N sort of ranked results stays in memory; a disk-spilling sort on a common-term search is the classic latency cliff.

---

## 11. Node.js Integration

### 11.1 Basic search endpoint with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Raw user text goes straight into websearch_to_tsquery — NEVER build a tsquery by
// string-concatenating user input into to_tsquery (both a syntax-error and injection risk).
async function searchArticles(userQuery, limit = 20, offset = 0) {
  const { rows } = await pool.query(
    `SELECT id, title,
            ts_rank_cd(tsv, q.query) AS rank
     FROM articles,
          websearch_to_tsquery('english', $1) AS q(query)
     WHERE tsv @@ q.query
     ORDER BY rank DESC
     LIMIT $2 OFFSET $3`,
    [userQuery, limit, offset]
  );
  return rows;
}
```

The `$1` placeholder is passed as a **bound parameter**, so even though it is user input it can never break out of the string literal — this is both the injection-safe form and the reason `websearch_to_tsquery` (which never raises a syntax error) is the right builder.

### 11.2 Ranked + highlighted results, headline only on the page

```javascript
async function searchWithSnippets(userQuery, limit = 20, offset = 0) {
  const { rows } = await pool.query(
    `WITH q AS (SELECT websearch_to_tsquery('english', $1) AS query),
     top AS (
       SELECT id, title, body, ts_rank_cd(tsv, q.query) AS rank
       FROM articles, q
       WHERE tsv @@ q.query
         AND published_at >= now() - interval '2 years'
       ORDER BY rank DESC
       LIMIT $2 OFFSET $3            -- paginate BEFORE the expensive headline
     )
     SELECT t.id, t.title, t.rank,
            ts_headline('english', t.body, q.query,
              'StartSel=<mark>,StopSel=</mark>,MaxFragments=2,MaxWords=30') AS snippet
     FROM top t, q
     ORDER BY t.rank DESC`,
    [userQuery, limit, offset]
  );
  return rows;
}
```

### 11.3 Handling empty / stop-word-only queries

```javascript
// websearch_to_tsquery('english', 'the a of') → empty tsquery, which matches NOTHING.
// Detect it before running the search so you can short-circuit or fall back.
async function safeSearch(userQuery, limit = 20) {
  const check = await pool.query(
    `SELECT websearch_to_tsquery('english', $1) AS q`, [userQuery]
  );
  const parsed = check.rows[0].q;        // '' when input was only stop words / punctuation
  if (!parsed) {
    return { results: [], reason: 'query_empty_after_normalization' };
  }
  const { rows } = await pool.query(
    `SELECT id, title, ts_rank_cd(tsv, q.query) AS rank
     FROM articles, websearch_to_tsquery('english', $1) AS q(query)
     WHERE tsv @@ q.query
     ORDER BY rank DESC LIMIT $2`,
    [userQuery, limit]
  );
  return { results: rows, reason: 'ok' };
}
```

### 11.4 Insert relies on the generated column — nothing to maintain

```javascript
// With a GENERATED ALWAYS AS ... STORED tsv column, the app inserts plain text.
// The tsvector and GIN index update automatically — no app-side tsvector building.
async function createArticle({ title, body, publishedAt }) {
  const { rows } = await pool.query(
    `INSERT INTO articles (title, body, published_at)
     VALUES ($1, $2, $3)
     RETURNING id`,               -- tsv is computed by the DB, not sent by the app
    [title, body, publishedAt]
  );
  return rows[0].id;
}
```

**On ORMs:** most ORMs have no first-class `tsvector`/`@@` support — you almost always drop to raw SQL for the search query itself (see section 12). The generated column, however, is defined in a migration and is invisible to the ORM at write time, so inserts/updates through the ORM "just work" and keep the index current.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do FTS?** — Partially. Prisma has a `search` filter for PostgreSQL that maps to `to_tsquery` + `@@`, but with real limitations.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Requires previewFeatures = ["fullTextSearchPostgres"] in the generator block.
// The `search` string uses to_tsquery SYNTAX (& | ! <->), NOT plain words —
// so you must translate user input into that syntax yourself (and handle errors).
const articles = await prisma.article.findMany({
  where: { body: { search: 'machine & learning' } },
  take: 20,
});

// Ranking, ts_rank_cd, ts_headline, weighted setweight, websearch_to_tsquery,
// and choosing the config are all UNSUPPORTED. Use raw SQL:
const ranked = await prisma.$queryRaw`
  SELECT id, title, ts_rank_cd(tsv, q.query) AS rank
  FROM articles, websearch_to_tsquery('english', ${userInput}) AS q(query)
  WHERE tsv @@ q.query
  ORDER BY rank DESC
  LIMIT 20
`;
```

**Where it breaks:** the `search` field uses `to_tsquery` grammar, so raw user input with spaces throws; there is no ranking, no highlighting, no config selection, and no way to target a generated tsvector column or a custom GIN index. It builds `to_tsvector(column)` on the fly, which may not match your index.

**Verdict:** usable only for trivial "does this word appear" filters. Any real search UI (ranking + snippets + safe input) uses `$queryRaw`.

---

### Drizzle ORM

**Can Drizzle do FTS?** — Yes, cleanly, via the `sql` template tag; Drizzle also supports declaring `tsvector` columns and GIN indexes in the schema.

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';
import { articles } from './schema';

// Schema can declare the generated tsvector column + GIN index (drizzle-kit emits the DDL).
// Query with the sql tag for full control:
const results = await db
  .select({
    id: articles.id,
    title: articles.title,
    rank: sql<number>`ts_rank_cd(${articles.tsv}, q.query)`,
  })
  .from(articles)
  .innerJoin(
    sql`websearch_to_tsquery('english', ${userInput}) AS q(query)`,
    sql`true`
  )
  .where(sql`${articles.tsv} @@ q.query`)
  .orderBy(sql`rank DESC`)
  .limit(20);
```

**Where it breaks:** there is no dedicated FTS DSL, so `@@`, `ts_rank`, `ts_headline` all go through `sql\`...\``. That is idiomatic in Drizzle and stays fully parameterized (`${userInput}` is bound), so it is not really a limitation — just not "typed sugar."

**Verdict:** best-in-class among ORMs. Schema-level tsvector/GIN support plus transparent `sql` interpolation covers the whole feature set safely.

---

### Sequelize

**Can Sequelize do FTS?** — Only via `sequelize.literal` / raw queries. No native `@@` or tsvector operator.

```javascript
const { QueryTypes } = require('sequelize');

// Raw query is the practical path — bind user input as a replacement, never string-concat.
const results = await sequelize.query(
  `SELECT id, title, ts_rank_cd(tsv, q.query) AS rank
   FROM articles, websearch_to_tsquery('english', :q) AS q(query)
   WHERE tsv @@ q.query
   ORDER BY rank DESC
   LIMIT 20`,
  { replacements: { q: userInput }, type: QueryTypes.SELECT }
);

// A findAll with Op — awkward and unsafe unless carefully escaped; not recommended:
const Op = Sequelize.Op;
await Article.findAll({
  where: sequelize.literal(`tsv @@ websearch_to_tsquery('english', ${sequelize.escape(userInput)})`),
  limit: 20,
});
```

**Where it breaks:** no model-level support for tsvector columns or GIN indexes (declare them in a raw migration). `sequelize.literal` bypasses parameter binding, so you must call `sequelize.escape()` manually — an injection footgun.

**Verdict:** use `sequelize.query()` with `replacements` for anything real. Model the tsvector column and GIN index in raw migration SQL.

---

### TypeORM

**Can TypeORM do FTS?** — Via QueryBuilder `.where()` raw fragments or `entityManager.query()`. No native FTS operators.

```typescript
const results = await dataSource
  .getRepository(Article)
  .createQueryBuilder('a')
  .select(['a.id', 'a.title'])
  .addSelect(`ts_rank_cd(a.tsv, websearch_to_tsquery('english', :q))`, 'rank')
  // parameterized — TypeORM binds :q safely
  .where(`a.tsv @@ websearch_to_tsquery('english', :q)`, { q: userInput })
  .orderBy('rank', 'DESC')
  .limit(20)
  .getRawMany();
```

The tsvector column can be declared with `@Column({ type: 'tsvector', ... })` (or a plain column typed as text and managed by a generated-column migration), and the GIN index via `@Index()` with a raw expression or a migration.

**Where it breaks:** no typed `@@`/rank helpers, so it is raw fragments throughout; `getRawMany()` is needed because ranking columns are not entity fields. Named parameters (`:q`) are bound, which keeps it injection-safe.

**Verdict:** workable with QueryBuilder raw fragments; parameters stay bound. Declare the tsvector column and GIN index in a migration.

---

### Knex.js

**Can Knex do FTS?** — Yes, transparently, via `knex.raw` with bindings. Knex is the most SQL-literal of the group.

```javascript
const results = await knex('articles')
  .select('id', 'title')
  .select(knex.raw('ts_rank_cd(tsv, q.query) AS rank'))
  .crossJoin(knex.raw(`websearch_to_tsquery('english', ?) AS q(query)`, [userInput]))
  .whereRaw('tsv @@ q.query')
  .orderBy('rank', 'desc')
  .limit(20);

// Or fully raw:
const rows = (await knex.raw(
  `SELECT id, title, ts_rank_cd(tsv, q.query) AS rank
   FROM articles, websearch_to_tsquery('english', ?) AS q(query)
   WHERE tsv @@ q.query
   ORDER BY rank DESC LIMIT ?`,
  [userInput, 20]
)).rows;
```

**Where it breaks:** nothing FTS-specific breaks — `?` bindings keep user input safe. The tsvector generated column and GIN index are created via `knex.schema.raw(...)` in a migration since Knex's schema builder has no `tsvector`/GIN vocabulary.

**Verdict:** excellent. `knex.raw` with positional bindings gives you the full FTS feature set with parameter safety and query-builder ergonomics.

---

### ORM Summary Table

| ORM | Native `@@`? | Ranking / headline | Safe user input | tsvector column + GIN | Verdict |
|-----|-------------|--------------------|-----------------|----------------------|---------|
| Prisma | `search` (to_tsquery syntax) | Raw SQL only | Must translate to operators | Not modeled; raw | Trivial filters only |
| Drizzle | via `sql` tag | via `sql` tag | Bound `${}` | Schema-level support | Best overall |
| Sequelize | No (literal/raw) | Raw query | `replacements` (or manual escape) | Raw migration | Use `sequelize.query()` |
| TypeORM | No (raw fragment) | `getRawMany()` | Named params `:q` | `@Index`/migration | Workable, all raw |
| Knex | `whereRaw` | `knex.raw` | `?` bindings | `schema.raw` migration | Most SQL-transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `products(id, name, description, category_id, price)`

Write a query that returns the `id` and `name` of every product whose `name` or `description` contains the word "wireless" (matched with English stemming, so "wirelessly" should also match). Use `to_tsvector`/`websearch_to_tsquery` explicitly with the `'english'` config. Then explain, in one sentence, why `WHERE description LIKE '%wireless%'` would give a different result set.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines prior topics)

Given:
- `articles(id, title, body, author_id, published_at)`
- `users(id, name)` (authors)

Return the top 10 articles matching a search term, showing `title`, author `name` (via INNER JOIN — remember Topic 11), a relevance `rank`, and a highlighted `snippet`. Requirements:
- Title matches must rank higher than body matches (use `setweight`).
- Only include articles published in the last year.
- Compute the `ts_headline` snippet only for the rows actually returned (not before the LIMIT).

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

The `documents(id, title, body, tsv, org_id, created_at)` table has 40M rows. `tsv` is a generated column with a GIN index. A user in one organization searches for a very common word like `report`. The naive query:

```sql
SELECT id, title, ts_rank_cd(tsv, q.query) AS rank
FROM documents, websearch_to_tsquery('english', 'report') AS q(query)
WHERE tsv @@ q.query
ORDER BY rank DESC
LIMIT 20;
```

...takes 4 seconds because `report` matches 6 million rows and ranking + sorting all of them dominates. The user only ever sees documents in their own `org_id` (say, ~2,000 docs).

1. Explain precisely which phase is slow and why the GIN index does not help it.
2. Rewrite the query so it is fast for this access pattern.
3. What index would you add to make your rewrite optimal?

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are handed a search feature that "sometimes returns no results even though the word is clearly in the document." Given `articles(id, body)` and a query built as:

```sql
WHERE to_tsvector(body) @@ to_tsquery('english', 'running')
```

1. Identify at least two distinct root causes that could produce silent zero-row results here.
2. For each, show the corrected query.
3. Describe how you would prevent this class of bug across the whole codebase (config discipline).

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — "How is PostgreSQL full-text search different from `LIKE '%term%'`, and when would you still use `LIKE`?"

**Junior answer:** "Full-text search is faster and can use an index, `LIKE` scans everything."

**Principal answer:** They solve different problems. `LIKE '%term%'` does raw character-substring matching with no language understanding, and a *leading* wildcard cannot use a B-tree index, so it sequentially scans. FTS normalizes text into lexemes — lowercasing, stemming (running → run), stop-word removal — indexes them in an inverted GIN index, and matches whole *words/concepts* with boolean and phrase logic in milliseconds. So FTS is the choice for word/language search. But FTS deliberately cannot do substring or infix matching: it matches lexemes, not fragments, so it will never find `run` inside `overrun`. For substring, prefix, autocomplete, or typo-tolerant matching you want `pg_trgm` trigram indexes (which *can* accelerate even `%term%`), and for truly exact non-linguistic matching (a code, a UUID fragment) plain `LIKE`/`=` is still correct. The decision is about whether you're matching *meaning* (FTS), *characters* (trgm/LIKE), or *exact tokens* (=/simple config).

**Interviewer follow-up:** "Your search needs both — word search *and* autocomplete on the same field. What do you build?" *(Expected: two indexes — a GIN tsvector index for word search and a GIN `gin_trgm_ops` index for the autocomplete/substring path — because no single index serves both; route by query type.)*

---

### Q2 — "A colleague's FTS query returns zero rows even though the term is obviously in the document. Walk me through how you'd debug it."

**Junior answer:** "I'd check if the data is really there and maybe rebuild the index."

**Principal answer:** The overwhelmingly likely cause is a **configuration mismatch** between the document and query sides. I'd first run `to_tsvector('english', body)` and `to_tsquery('english', 'term')` (or whichever config) side by side and compare lexemes — if the document stemmed `running` to `run` but the query used `'simple'` and kept `running`, `@@` finds no membership match and returns zero with no error. Second cause: the query was built with the `text @@ text` form, which uses `default_text_search_config`, and that GUC differs between where the data was indexed and where the query runs. Third: the term is actually a stop word in that config (`the`, `a`) so the tsquery is empty and matches nothing. I'd use `ts_debug('english', 'running')` to see exactly which dictionary produced which lexeme. The fix in all cases is to pin the config explicitly and identically on both sides and never rely on the default.

**Interviewer follow-up:** "How do you make config mismatches impossible, not just fixed this once?" *(Expected: materialize the tsvector in a generated column with a literal config, wrap search in one shared function/repository method that hard-codes the same config, and add a test that round-trips a known stemmed word.)*

---

### Q3 — "Search works great in staging but a specific query type melts a CPU in production. What's happening and how do you fix it?"

**Junior answer:** "Add more indexes or a bigger server."

**Principal answer:** The symptom points at **ranking a non-selective query**, not at the index. The GIN `@@` filter is index-accelerated and scales sublinearly, but `ts_rank`/`ts_rank_cd` run in the SELECT phase for *every* row that survives `@@`, and `ORDER BY rank` then sorts that whole set — neither is indexable. In staging the test data is small so any matched set is small; in production a user searches a common word that matches millions of rows, and you now compute rank millions of times and sort millions of rows to show 20. That is pure CPU, which is why one core saturates while disk is idle. Fixes, in order: add selectivity (combine `@@` with an indexed predicate like `org_id`, `status`, or a date range so far fewer rows reach ranking); cap the candidate set (rank only the first N `@@` matches in a subquery); or, if globally-correct rank order on huge matched sets is a hard requirement, adopt a RUM index that can return rows in rank order from the index and skip the sort. And never compute `ts_headline` before the LIMIT — it re-parses raw document text and is far more expensive than ranking.

**Interviewer follow-up:** "The product owner insists on exact global rank order across all 6 million matches — no candidate capping. Now what?" *(Expected: RUM index with a distance operator, or accept the cost and move search to Elasticsearch which is built for exactly this ranking-at-scale workload; be explicit about the tradeoff rather than pretending core FTS makes it free.)*

---

### Q4 — "When would you reach past PostgreSQL FTS to Elasticsearch, and when is that premature?"

**Junior answer:** "Elasticsearch is always better for search, so use it when search matters."

**Principal answer:** Premature until you hit a wall PostgreSQL FTS genuinely can't clear. FTS is the right default when: the corpus lives in Postgres already, you want transactional consistency (search index updated atomically with the row via a generated column), and your needs are word/phrase search with basic relevance. You reach for Elasticsearch/OpenSearch when you need: advanced relevance tuning (BM25, per-field boosting beyond A/B/C/D, function scoring), rich faceting/aggregations over search results, typo-tolerance and fuzzy matching as first-class features, multi-language analyzers at scale, or horizontal sharding across nodes because the index outgrew one machine. The cost of ES is real: a separate cluster, an ingestion/sync pipeline (CDC or dual-writes) that introduces eventual consistency and its own failure modes, and operational burden. So the rule is: stay in Postgres FTS until a concrete requirement — relevance sophistication, faceting, fuzzy, or scale — forces the move, and prefer `pg_trgm` for the specific case of typo/substring before assuming you need a whole cluster.

**Interviewer follow-up:** "You've decided to move to Elasticsearch. How do you keep it in sync with Postgres?" *(Expected: change-data-capture — logical replication / Debezium / outbox pattern — rather than best-effort dual writes, and a plan for reindexing and consistency lag.)*

---

## 15. Mental Model Checkpoint

1. A document is stored with `to_tsvector('english', body)` and searched with `to_tsquery('simple', 'running')`. The word "running" is plainly in the body. How many rows match, and precisely why?

2. You run `SELECT to_tsvector('english', 'The universities of the universe')`. What lexemes come out, how many positions, and what does the result reveal about the difference between stemming and true synonymy?

3. Your search query is `WHERE to_tsvector('english', title || ' ' || body) @@ q`, and you have a GIN index on a generated `tsv` column. Will the index be used? Why or why not, and what is the one-line fix?

4. A search for the word "data" returns in 900 ms; a search for "quantum chromodynamics" returns in 3 ms — same table, same index. Explain the difference in terms of which execution phase dominates each.

5. You need to (a) rank title matches above body matches and (b) still use a single GIN index. Where does the field-weight information have to live for both to be true simultaneously?

6. Why is `ts_headline` dramatically more expensive than `ts_rank`, and what does that imply about where in the query pipeline you are allowed to call it?

7. A product manager asks for "search that finds `iphone` when the user types `iphon`" (autocomplete). Is that a full-text search problem? If not, what is it, and what index type solves it?

---

## 16. Quick Reference Card

```sql
-- ============ BUILD DOCUMENTS ============
to_tsvector('english', text)          -- parse + stem + drop stop words → sorted lexemes
setweight(tsvector, 'A')              -- tag lexemes A/B/C/D (A highest) for weighted ranking
tsv_a || tsv_b                        -- concatenate tsvectors (merges positions)
'simple' config                      -- NO stemming, NO stop words (exact tokens: tags, SKUs)

-- ============ BUILD QUERIES ============
to_tsquery('english', 'a & b | c')    -- full operators; ERRORS on bad input (trusted only)
plainto_tsquery('english', 'a b')     -- words → AND; ignores operators; never errors
phraseto_tsquery('english', 'a b')    -- words → phrase (a <-> b); adjacency required
websearch_to_tsquery('english', txt)  -- Google-style: "phrase", or, -exclude; USER-SAFE

-- tsquery operators:  & AND   | OR   ! NOT   <-> phrase(adjacent)   <N> distance   prefix:*

-- ============ MATCH ============
tsv @@ query                          -- TRUE if tsvector satisfies tsquery (index-accelerated)
text @@ query                         -- uses default_text_search_config — AVOID (config trap)

-- ============ RANK (not indexable — runs per matched row) ============
ts_rank(tsv, query)                   -- by term frequency + field weight
ts_rank_cd(tsv, query)                -- cover density: rewards proximity; needs positions
ts_rank('{0.1,0.2,0.4,1.0}', tsv, q)  -- weight array order is D,C,B,A
ts_rank(tsv, q, 32)                   -- normalization flag: 32 → scale into (0,1)

-- ============ HIGHLIGHT (expensive — re-parses raw text; call AFTER limit) ============
ts_headline('english', body, query, 'StartSel=<mark>,StopSel=</mark>,MaxFragments=2')

-- ============ PRODUCTION SCHEMA ============
ALTER TABLE t ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (setweight(to_tsvector('english',coalesce(title,'')),'A') ||
                       setweight(to_tsvector('english',coalesce(body,'')), 'B')) STORED;
CREATE INDEX t_tsv_gin ON t USING GIN (tsv);   -- default choice
CREATE INDEX t_tsv_gist ON t USING GiST (tsv); -- only if write-heavy/small & measured
```

**Perf rules of thumb:**
- `@@` on GIN scales *sublinearly* — a rare 2-term query is ~as fast at 100M rows as at 1M.
- Ranking + `ORDER BY rank` cost is O(rows surviving `@@`), NOT O(result page). Make `@@` selective.
- Compute `ts_headline` only for the final LIMITed page — never before.
- Same config on both sides, pinned explicitly, or you get silent zero rows.
- FTS = words/meaning. `pg_trgm` = substring/prefix/typo. Elasticsearch = relevance tuning + faceting + scale.

**Interview one-liners:**
- "FTS matches lexemes, not substrings — that's why it can't autocomplete `iphon`."
- "The `@@` filter is indexable; `ts_rank` and `ts_headline` are not — that's where latency hides."
- "Config mismatch is the silent killer: `english` doc + `simple` query = zero rows, no error."
- "Use a `GENERATED ALWAYS AS ... STORED` tsvector so the index can never drift from the text."
- "Reach for Elasticsearch when you need faceting, fuzzy, or sharding — not before."

---

## Connected Topics

- **Topic 65 — Array Operations** (previous): GIN indexes serve both `tsvector @@ tsquery` and array containment (`@>`, `&&`); the inverted-index mental model transfers directly from arrays to full-text.
- **Topic 67 — Materialized Views** (next): a materialized view holding a pre-computed, pre-ranked `tsvector` is a common way to cap ranking cost for expensive search aggregates — the natural next step when a generated column isn't enough.
- **Topic 11 — INNER JOIN in Depth**: search results are routinely joined back to authors/categories; the ranked-then-joined pattern and fan-out awareness apply directly.
- **Topic 07 — NULL in Depth**: `coalesce(col,'')` inside `to_tsvector` exists because a NULL fed to `to_tsvector` yields NULL, which silently drops the field from the document.
- **GIN / GiST index internals**: the inverted-index vs signature-tree tradeoff underpins the index choice here and reappears for JSONB and arrays.
- **The planner and Bitmap Index/Heap Scan**: FTS reads exactly like any other indexable predicate in EXPLAIN — Bitmap Index Scan feeding a Bitmap Heap Scan with a Recheck Cond.
- **`pg_trgm` (trigram matching)**: the complementary tool for substring, prefix, autocomplete, and fuzzy/typo search that FTS deliberately does not handle.
- **`default_text_search_config` GUC**: the session/cluster setting behind the `text @@ text` config trap; understanding GUC scoping explains why the same query behaves differently across environments.
