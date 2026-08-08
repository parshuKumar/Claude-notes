# 05 — Social Feed Fan-out
## The celebrity problem, and why one design cannot serve both ends of a power law

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## WHY THIS CASE STUDY EXISTS

Tier 1 was about **contention and correctness** — many writers, one row, money on the line.

Tier 2 is about **volume**. Nothing here is hard to get *correct*; it is hard to make *fit*. One post by one user can generate forty million writes, and the same system must also serve a user with three followers without wasting anything.

The lesson that generalises far beyond feeds: **when your data follows a power law, no single strategy is right for the whole distribution.** The correct design is almost always *two* strategies with a threshold between them — and choosing that threshold is engineering, not intuition.

---

## 1. THE BRIEF

A social platform. Users follow other users; posts from people you follow appear in your home timeline, newest first.

> **80 million users. 300,000 timeline reads/sec. 20,000 posts/sec.**
> **Follower distribution follows a power law: the median user has 180 followers; the top account has 41 million.**

Business rules:

1. **Home timeline** = posts from everyone you follow, reverse-chronological, paginated.
2. **Freshness**: a new post should appear in followers' timelines within ~5 seconds. Not instant, not a minute.
3. **Deletes and blocks must take effect** — a deleted post disappears; a blocked user's posts vanish from your timeline immediately.
4. **Timelines are bounded**: nobody scrolls past ~800 posts. You do not need to materialise infinite history.
5. **Follows/unfollows are frequent**: 4,000/sec, and a new follow should reflect in the timeline reasonably quickly.
6. **The profile timeline** (just one user's own posts) is a separate, much easier problem — but it shares storage.

Infrastructure: PostgreSQL 16 (primary + 4 replicas), Redis cluster, Kafka, 200 Node.js pods.

---

## 2. ACCESS PATTERN TABLE

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness |
|---|---|---|---|---|---|---|
| A | `GET /timeline` — page 1 (20 posts) | 300,000/s | 20 | 0 | **80 ms** | 5 s |
| B | `GET /timeline?cursor=` — deeper pages | 40,000/s | 20 | 0 | 150 ms | 30 s |
| C | `POST /posts` | 20,000/s | 1 | **1 … 41,000,000** | 300 ms | — |
| D | `GET /users/:id/posts` — profile timeline | 60,000/s | 20 | 0 | 100 ms | 5 s |
| E | `POST /follow` / `DELETE /follow` | 4,000/s | 2 | 1 (+ backfill) | 400 ms | 30 s |
| F | `DELETE /posts/:id` | 200/s | 1 | 1 (+ fan-out cleanup) | 1 s | 5 s |
| G | `POST /block` | 100/s | 2 | 1 | 500 ms | **0** |
| H | Trim job — cap timeline length | continuous | ~ | ~ | n/a | n/a |

**What jumps out:**

- **A at 300,000/s with an 80 ms budget** is the entire product. Everything else exists to make A fast.
- **C's write amplification is unbounded**: "1 … 41,000,000". That range is the whole problem. A design that's correct for the median is catastrophic for the tail.
- **G tolerates zero staleness** — blocking someone must work immediately, which rules out purely precomputed timelines unless you filter at read time.
- **Note what's missing**: nobody asked for "posts from 2019." Requirement 4 caps the working set, and that cap is what makes the whole thing tractable.

---

## 3. THE INVARIANTS

```
I1.  ORDERING: a timeline is strictly reverse-chronological by post
     creation time, with a total order (ties broken deterministically).

I2.  COMPLETENESS: if A follows B at time T, every post B makes after T
     appears in A's timeline. No silent drops.

I3.  DELETION: a deleted post appears in NO timeline within 5 seconds.

I4.  BLOCKING: if A blocks B, B's posts appear in A's timeline
     IMMEDIATELY — not eventually. (Safety requirement, not performance.)

I5.  IDEMPOTENCE: fan-out delivered twice must not duplicate a post
     in a timeline.

I6.  BOUNDEDNESS: no user's materialised timeline exceeds N entries.
```

**I4 is the one that constrains the architecture.** It means a precomputed timeline can never be served *raw* — there must be a filter step at read time, or blocking must trigger an immediate purge. That single requirement kills the naive "just read the precomputed list" design.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA (fan-out on read)

```sql
CREATE TABLE users (
  id bigserial PRIMARY KEY,
  handle text NOT NULL UNIQUE,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE follows (
  follower_id bigint NOT NULL REFERENCES users(id),
  followee_id bigint NOT NULL REFERENCES users(id),
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee ON follows (followee_id);

CREATE TABLE posts (
  id         bigserial   PRIMARY KEY,
  author_id  bigint      NOT NULL REFERENCES users(id),
  body       text        NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz NULL
);
CREATE INDEX idx_posts_author_created ON posts (author_id, created_at DESC);
```

```sql
-- the timeline query: join and sort at read time
SELECT p.id, p.author_id, p.body, p.created_at
FROM posts p
JOIN follows f ON f.followee_id = p.author_id
WHERE f.follower_id = $1
  AND p.deleted_at IS NULL
ORDER BY p.created_at DESC
LIMIT 20;
```

This is the **fan-out on read** design. It is correct, simple, and has one enormous virtue: a post costs exactly **one write**.

---

## 5. WHERE IT BREAKS

### 5.1 The read is a merge of N sorted streams

```sql
EXPLAIN (ANALYZE, BUFFERS) <the timeline query for a user following 400 people>;
```
```
Limit  (actual time=2841.4..2841.5 rows=20 loops=1)
  ->  Sort  (actual time=2841.4..2841.4 rows=20 loops=1)
        Sort Key: p.created_at DESC
        Sort Method: top-N heapsort  Memory: 34kB
        ->  Nested Loop  (actual time=0.08..2718.2 rows=1204882 loops=1)
              ->  Index Scan using idx_follows_follower on follows f
                    (actual rows=400 loops=1)
              ->  Index Scan using idx_posts_author_created on posts p
                    (actual rows=3012 loops=400)          ← ⚠ 400 LOOPS
                    Index Cond: (author_id = f.followee_id)
                    Filter: (deleted_at IS NULL)
              Buffers: shared hit=88204 read=412008
Execution Time: 2842.1 ms
```

**2.8 seconds.** Read the shape: to return **20 rows**, the executor produced **1,204,882** — every post by all 400 followees — and then sorted them to find the newest 20.

**The mechanism:** there is no index that orders posts by `created_at` *across an arbitrary set of authors*. `(author_id, created_at)` is sorted within each author. Merging 400 sorted streams to get a global top-20 requires touching all of them.

### 5.2 It gets worse with more follows, and the distribution is brutal

```sql
SELECT width_bucket(cnt, 0, 5000, 10) AS bucket, count(*) AS users, max(cnt) AS max_follows
FROM (SELECT follower_id, count(*) cnt FROM follows GROUP BY 1) t
GROUP BY 1 ORDER BY 1;
```
```
 bucket | users    | max_follows
--------+----------+-------------
      1 | 62000000 |         500
      2 | 12000000 |        1000
      3 |  4000000 |        1500
      ...
     10 |    41000 |        5000
```

```
 follows │ query time
 ────────┼────────────
      50 │    180 ms
     200 │  1,204 ms
     400 │  2,842 ms
   1,000 │  7,910 ms
   5,000 │ 41,200 ms       ← and 41,000 users are here
```

**Roughly linear in follow count.** At 300,000 reads/sec with a mean of ~1.2 s, you would need ~360,000 concurrent query executions. There is no instance size that does this.

### 5.3 A LATERAL rewrite helps — and still isn't enough

The obvious optimisation: don't fetch *all* posts per author, fetch only the newest 20 from each, then merge.

```sql
SELECT p.* FROM follows f
CROSS JOIN LATERAL (
  SELECT id, author_id, body, created_at FROM posts
   WHERE author_id = f.followee_id AND deleted_at IS NULL
   ORDER BY created_at DESC LIMIT 20
) p
WHERE f.follower_id = $1
ORDER BY p.created_at DESC LIMIT 20;
```
```
Limit  (actual time=182.1..182.2 rows=20 loops=1)
  ->  Sort (actual rows=8000 loops=1)          ← 400 × 20 = 8,000, not 1.2M
        ->  Nested Loop (actual rows=8000 loops=1)
              ->  Index Scan on follows (actual rows=400)
              ->  Limit (actual rows=20 loops=400)     ← still 400 index descents
Execution Time: 182.6 ms
```

**2,842 ms → 183 ms. 15× better.** And still not enough:

- 183 ms × 300,000/s = 54,900 concurrent executions. Not achievable.
- It scales with follow count: a 5,000-follow user is still 2.3 s.
- Pagination is worse — page 5 needs `LIMIT 100` per author, so 500,000 rows.
- Every read does 400 random index descents. Cache behaviour is terrible.

**The fundamental problem: the work is done at read time, and reads outnumber writes 15:1.** You are paying 300,000 times per second for something that changes 20,000 times per second.

---

## 6. ATTEMPT 2 — FAN-OUT ON WRITE (and why it explodes)

Invert it. When a post is created, **write one row into each follower's timeline**. Reads become a single index scan.

```sql
CREATE TABLE timeline (
  user_id    bigint      NOT NULL,
  post_id    bigint      NOT NULL,
  author_id  bigint      NOT NULL,
  created_at timestamptz NOT NULL,
  PRIMARY KEY (user_id, created_at DESC, post_id)
);
```

```sql
-- the read: one index range scan. Beautiful.
SELECT t.post_id, t.author_id, t.created_at
FROM timeline t WHERE t.user_id = $1
ORDER BY t.created_at DESC LIMIT 20;
```
```
Limit  (actual time=0.024..0.031 rows=20 loops=1)
  ->  Index Only Scan using timeline_pkey on timeline
        Index Cond: (user_id = 4471)
        Heap Fetches: 0
        Buffers: shared hit=6
Execution Time: 0.041 ms
```

**0.041 ms. 69,000× faster than Attempt 1.** Reads are solved completely.

**Now the write:**

```js
async function createPost(authorId, body) {
  const post = await db.query(
    'INSERT INTO posts (author_id, body) VALUES ($1,$2) RETURNING id, created_at',
    [authorId, body]);

  // ⚠ fan out to every follower
  await db.query(`
    INSERT INTO timeline (user_id, post_id, author_id, created_at)
    SELECT f.follower_id, $1, $2, $3 FROM follows f WHERE f.followee_id = $2`,
    [post.rows[0].id, authorId, post.rows[0].created_at]);
}
```

```
 MEDIAN USER (180 followers):
   1 post → 180 timeline rows. ~4 ms. Completely fine.

 POPULAR USER (2,000,000 followers):
   1 post → 2,000,000 rows.
   at ~40 bytes/row = 80 MB of heap + ~120 MB of WAL
   INSERT ... SELECT duration: ~94 seconds
   ⇒ one transaction holding locks and a snapshot for 94 seconds
     → blocks vacuum cluster-wide (Topic 46)
     → the API call times out
     → replicas lag by minutes

 THE TOP ACCOUNT (41,000,000 followers):
   1 post → 41,000,000 rows.
   ~1.6 GB of heap + 2.5 GB of WAL. ~32 minutes.
   ⇒ ★ THE SYSTEM IS DOWN.
```

**And the aggregate arithmetic:**

```sql
SELECT sum(cnt) AS total_timeline_rows_per_second
FROM (SELECT followee_id, count(*) cnt FROM follows GROUP BY 1) f
JOIN (SELECT author_id, count(*) posts_per_sec FROM recent_posts GROUP BY 1) p
  ON p.author_id = f.followee_id;
```
```
 average followers per post ≈ 2,400
 20,000 posts/s × 2,400 = 48,000,000 timeline writes/second
```

**48 million writes/second.** A well-tuned PostgreSQL node does ~20,000. You would need 2,400 nodes, for a feed.

**Plus the second-order costs:**

| Cost | Impact |
|---|---|
| Storage | 48M rows/s × 40 B × 86,400 s = **166 TB/day** before trimming |
| Follow (pattern E) | a new follow must backfill the last 800 posts of the followee — 800 writes per follow, 4,000/s = 3.2M writes/s more |
| Delete (pattern F) | deleting one post by a 2M-follower account means deleting 2M timeline rows |
| Block (pattern G) | must purge that author's rows from the blocker's timeline |

**Neither pure strategy works.** Fan-out on read dies on the read side; fan-out on write dies on the write side. And critically — **each dies on a different part of the distribution.**

---

## 7. THE PRODUCTION DESIGN — HYBRID FAN-OUT

```
 THE INSIGHT
 ───────────────────────────────────────────────────────────────────
 Fan-out on WRITE is cheap when the author has FEW followers.
 Fan-out on READ  is cheap when you're merging FEW streams.

 A user's home timeline is a merge of:
   • their PUSHED timeline  (materialised from ordinary authors)
   • a PULL over the handful of CELEBRITIES they follow

 Because the number of celebrities ANY user follows is small (median 3,
 p99 ~40), the pull side is a 3–40 stream merge — exactly the case
 where fan-out on read is fast.

 ⇒ Both strategies applied to the part of the distribution where each
   is cheap. This is the whole design.
```

### 7.1 The schema

```sql
-- ═══════════════════════════════════════════════════════════════════
-- AUTHORS — the celebrity flag drives everything
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE users (
  id             bigserial   PRIMARY KEY,
  handle         text        NOT NULL UNIQUE,
  follower_count bigint      NOT NULL DEFAULT 0,
  is_celebrity   boolean     NOT NULL DEFAULT false,   -- ★ derived, cached
  created_at     timestamptz NOT NULL DEFAULT now()
) WITH (fillfactor = 85);
CREATE INDEX idx_users_celebrity ON users (id) WHERE is_celebrity;

-- ═══════════════════════════════════════════════════════════════════
-- FOLLOWS — needs BOTH directions, and the celebrity split
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE follows (
  follower_id bigint      NOT NULL,
  followee_id bigint      NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (follower_id, followee_id)
);
-- fan-out needs "who follows X" — the reverse direction
CREATE INDEX idx_follows_followee ON follows (followee_id, follower_id);

-- ★ THE KEY OPTIMISATION FOR THE READ PATH:
--   "which celebrities does user X follow?" must be ~free.
--   A partial index over a join isn't possible, so denormalise it.
CREATE TABLE celebrity_follows (
  follower_id bigint NOT NULL,
  followee_id bigint NOT NULL,
  PRIMARY KEY (follower_id, followee_id)
);
-- typically 3–40 rows per user; the whole table fits in RAM

-- ═══════════════════════════════════════════════════════════════════
-- POSTS — the source of truth. Never fanned out.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE posts (
  id          bigint      NOT NULL,        -- ★ SNOWFLAKE/ULID: time-sortable
  author_id   bigint      NOT NULL,
  body        text        NOT NULL,
  reply_to_id bigint      NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  deleted_at  timestamptz NULL,
  PRIMARY KEY (created_at, id)
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_posts_author ON posts (author_id, id DESC) WHERE deleted_at IS NULL;

-- ═══════════════════════════════════════════════════════════════════
-- TIMELINE — materialised, bounded, ONLY for non-celebrity authors
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE timeline (
  user_id   bigint NOT NULL,
  post_id   bigint NOT NULL,        -- time-sortable, so this IS the sort key
  author_id bigint NOT NULL,
  PRIMARY KEY (user_id, post_id DESC)
) PARTITION BY HASH (user_id);
-- 64 partitions; trimmed to 800 entries per user by the trim job
```

**Every decision justified:**

| Decision | Why |
|---|---|
| `post_id` is a Snowflake/ULID | **time-sortable**, so `ORDER BY post_id DESC` = chronological. Saves storing `created_at` in every timeline row (8 bytes × 48M/s) and makes the B-tree append-friendly (Topic 11) |
| `timeline` has no `created_at` | derivable from `post_id`. At this volume, 8 bytes/row matters |
| `PRIMARY KEY (user_id, post_id DESC)` | exactly matches pattern A: one range scan, already ordered, index-only |
| `timeline` partitioned by **hash(user_id)** | spreads write load; each partition's index stays small enough to cache (Topic 11's cliff) |
| `posts` partitioned by **range(created_at)** | retention and archival via `DROP PARTITION` (Topic 59) |
| `celebrity_follows` as a separate table | makes "which celebrities do I follow?" a 3–40 row lookup instead of a join with a filter |
| `is_celebrity` denormalised onto `users` | the fan-out path must decide in microseconds; a count() per post is impossible |

### 7.2 Choosing the threshold — this is the actual engineering

Not intuition. Arithmetic.

```
 LET  T = the follower-count threshold above which an author is a celebrity.

 WRITE COST (fan-out) = Σ over posts by authors with followers < T
                        of (their follower count)

 READ COST (pull)     = for each timeline read, the number of celebrities
                        that user follows

 From the actual distribution:

   T        fanout writes/s   avg celebrities followed   p99 celebs followed
 ────────────────────────────────────────────────────────────────────────────
   ∞ (none)   48,000,000          0                        0
   1,000,000  12,400,000          0.4                      4
     100,000   3,100,000          1.8                     14
      10,000     880,000          6.2                     38     ★
       1,000     210,000         41.0                    210
         100      48,000        312.0                  1,400

 CONSTRAINTS:
   • write budget: ~1,000,000 timeline writes/s across the fleet
   • read budget: the pull merge must stay under ~40 streams at p99
     (from Attempt 1: 40 streams ≈ 18 ms — fits the 80 ms budget)

 ⇒ T = 10,000 satisfies BOTH:
      880,000 writes/s   ✓ under the write budget
      p99 38 streams     ✓ under the read budget

 ★ NOTE HOW NARROW THE WINDOW IS. T=1,000 blows the read budget;
   T=100,000 blows nothing but wastes 3.5× the writes. The threshold
   is not arbitrary — it is the intersection of two constraints, and
   it MOVES as the distribution changes. Recompute it quarterly.
```

```sql
-- The query that produces that table — run it on your real data
WITH thresholds AS (SELECT unnest(ARRAY[100,1000,10000,100000,1000000]) AS t),
     fc AS (SELECT followee_id, count(*) AS followers FROM follows GROUP BY 1)
SELECT th.t,
       (SELECT sum(fc.followers * pr.posts_per_sec)
          FROM fc JOIN post_rates pr ON pr.author_id = fc.followee_id
         WHERE fc.followers < th.t)                       AS fanout_writes_per_sec,
       (SELECT round(avg(c), 1) FROM (
          SELECT f.follower_id, count(*) FILTER (WHERE fc2.followers >= th.t) AS c
            FROM follows f JOIN fc fc2 ON fc2.followee_id = f.followee_id
           GROUP BY 1) x)                                 AS avg_celebs_followed
FROM thresholds th ORDER BY th.t;
```

### 7.3 The write path

```js
const CELEB_THRESHOLD = 10_000;

async function createPost({ authorId, body, idempotencyKey }) {
  const postId = snowflake();            // time-sortable, generated app-side

  // ── STEP 1: the source of truth. ONE row. Always fast. ──
  const post = await withTransaction(async (tx) => {
    const p = await tx.query(
      `INSERT INTO posts (id, author_id, body, created_at)
       VALUES ($1,$2,$3, now())
       ON CONFLICT (created_at, id) DO NOTHING
       RETURNING id, author_id, created_at`,
      [postId, authorId, body]);
    if (p.rowCount === 0) return null;   // idempotent replay

    // no dual write — the outbox carries the fan-out job (Topic 52)
    await tx.query(
      `INSERT INTO outbox (topic, payload) VALUES ('post.created', $1)`,
      [JSON.stringify({ postId, authorId })]);
    return p.rows[0];
  });

  return post;   // ★ the API returns HERE. Fan-out is asynchronous.
}
```

```js
// ── STEP 2: the fan-out worker, consuming from Kafka ──
async function fanOutWorker({ postId, authorId }) {
  const author = await db.query(
    'SELECT is_celebrity, follower_count FROM users WHERE id=$1', [authorId]);

  if (author.rows[0].is_celebrity) return;      // ★ celebrities are PULLED

  // Batched, bounded, resumable. NEVER one giant INSERT...SELECT.
  const BATCH = 5_000;
  let cursor = 0;
  for (;;) {
    const { rowCount } = await db.query(`
      WITH batch AS (
        SELECT follower_id FROM follows
         WHERE followee_id = $1 AND follower_id > $2
         ORDER BY follower_id LIMIT $3
      )
      INSERT INTO timeline (user_id, post_id, author_id)
      SELECT b.follower_id, $4, $1 FROM batch b
      ON CONFLICT (user_id, post_id) DO NOTHING       -- ★ I5: idempotent
      RETURNING user_id`,
      [authorId, cursor, BATCH, postId]);

    if (rowCount === 0) break;
    cursor = /* max follower_id from the batch */;
    // Each batch is its own short transaction: no long locks, no snapshot
    // pinning, resumable if the worker dies mid-fan-out.
  }
}
```

**Two properties that matter more than they look:**
- **`ON CONFLICT DO NOTHING`** makes redelivery from Kafka harmless (I5). Without it, at-least-once delivery duplicates posts in timelines.
- **Batching** bounds transaction length. A 9,999-follower fan-out is two batches of short transactions, not one 9,999-row transaction. (Standing rule: every background job that touches a hot table must be batched — case study 01.)

### 7.4 The read path — the merge

```js
const PAGE = 20;

async function getTimeline({ userId, cursor }) {
  const [pushed, celebs, blocked] = await Promise.all([
    // (a) the materialised timeline — one index-only range scan
    db.query(`
      SELECT post_id, author_id FROM timeline
       WHERE user_id = $1 AND ($2::bigint IS NULL OR post_id < $2)
       ORDER BY post_id DESC LIMIT $3`,
      [userId, cursor, PAGE * 2]),          // over-fetch for filtering

    // (b) which celebrities does this user follow? 3–40 rows, always cached
    db.query('SELECT followee_id FROM celebrity_follows WHERE follower_id=$1', [userId]),

    // (c) I4: blocks must apply IMMEDIATELY, so they are read fresh
    redis.smembers(`blocks:${userId}`),
  ]);

  // (d) pull the celebrities' recent posts — a small LATERAL merge
  const celebPosts = celebs.rows.length === 0 ? { rows: [] } : await db.query(`
    SELECT p.id AS post_id, p.author_id FROM unnest($1::bigint[]) AS c(id)
    CROSS JOIN LATERAL (
      SELECT id, author_id FROM posts
       WHERE author_id = c.id AND deleted_at IS NULL
         AND ($2::bigint IS NULL OR id < $2)
       ORDER BY id DESC LIMIT $3
    ) p
    ORDER BY p.id DESC LIMIT $3`,
    [celebs.rows.map(r => r.followee_id), cursor, PAGE * 2]);

  // (e) merge two sorted streams — post_id is time-sortable, so this is
  //     a plain descending merge. O(n), in memory, ~40 elements.
  const merged = mergeDesc(pushed.rows, celebPosts.rows, r => r.post_id);

  // (f) FILTER AT READ TIME — this is what makes I3 and I4 achievable
  const blockedSet = new Set(blocked.map(Number));
  const visible = merged.filter(r => !blockedSet.has(Number(r.author_id)));

  // (g) hydrate post bodies from cache
  const ids = visible.slice(0, PAGE).map(r => r.post_id);
  const bodies = await hydratePosts(ids);      // Redis MGET, DB fallback

  return { posts: bodies.filter(Boolean), nextCursor: ids[ids.length - 1] };
}
```

**Why the read-time filter is non-negotiable:**

```
 I3 (deletes) and I4 (blocks) both require that a post STOP appearing.

 OPTION A: purge fan-out rows on delete/block
   • delete a post by a 9,999-follower author → 9,999 deletes
   • block a user → delete their rows from your timeline
   ✗ For I4 this is a WRITE on a SAFETY path. If it's slow or fails,
     a blocked user's content stays visible. Unacceptable.

 OPTION B: filter at read time                              ★ CHOSEN
   • the timeline row stays; the read drops it
   • blocks come from Redis, so a block takes effect on the NEXT read
   • deleted posts fail hydration (step g) and are dropped
   ✓ I4 is satisfied in milliseconds with zero write amplification
   ✗ cost: over-fetch (PAGE × 2) so filtering still yields a full page

 ⇒ The materialised timeline is a CANDIDATE SET, not the answer.
   That reframing is what makes the whole design work.
```

### 7.5 Promotion and demotion — the threshold in motion

```sql
-- Hourly: recompute who is a celebrity, with hysteresis
WITH counts AS (
  SELECT followee_id AS user_id, count(*) AS n FROM follows GROUP BY 1
)
UPDATE users u SET
  follower_count = c.n,
  is_celebrity = CASE
    -- ★ HYSTERESIS: promote at 10,000, demote only below 8,000.
    --   Without a gap, a user oscillating around the threshold would
    --   flap between strategies, and each flip costs a backfill.
    WHEN u.is_celebrity AND c.n >= 8000  THEN true
    WHEN NOT u.is_celebrity AND c.n >= 10000 THEN true
    ELSE false
  END
FROM counts c WHERE c.user_id = u.id
  AND (u.follower_count <> c.n OR u.is_celebrity <> (c.n >= 10000));
```

```
 PROMOTION (ordinary → celebrity):
   Their existing fanned-out rows stay in timelines. Harmless — the read
   path merges pushed and pulled, and dedups by post_id.
   Future posts are pulled. No backfill needed. ★ Cheap.

 DEMOTION (celebrity → ordinary):
   Their followers' timelines have NO rows from them. Future posts are
   pushed, but the last 800 posts are missing until enough new ones
   accumulate.
   ⇒ Backfill: for each follower, insert that author's recent posts.
     Rare (a few hundred users/day), batched, low priority.

 ⇒ Promotion is free; demotion costs a backfill. Hysteresis makes
   demotion rare. That asymmetry is why the gap is on the low side.
```

### 7.6 Follow, unfollow, and trim

```js
// FOLLOW — backfill the pushed timeline, or just note the celebrity
async function follow(followerId, followeeId) {
  await withTransaction(async (tx) => {
    await tx.query(
      `INSERT INTO follows (follower_id, followee_id) VALUES ($1,$2)
       ON CONFLICT DO NOTHING`, [followerId, followeeId]);

    const f = await tx.query('SELECT is_celebrity FROM users WHERE id=$1', [followeeId]);
    if (f.rows[0].is_celebrity) {
      await tx.query(
        `INSERT INTO celebrity_follows (follower_id, followee_id) VALUES ($1,$2)
         ON CONFLICT DO NOTHING`, [followerId, followeeId]);
      return;                                     // ★ no backfill needed
    }
    await tx.query(`INSERT INTO outbox (topic,payload) VALUES ('follow.backfill',$1)`,
                   [JSON.stringify({ followerId, followeeId })]);
  });
}

// the backfill worker: 200 recent posts, not 800 — the rest arrive naturally
async function backfillWorker({ followerId, followeeId }) {
  await db.query(`
    INSERT INTO timeline (user_id, post_id, author_id)
    SELECT $1, p.id, p.author_id FROM posts p
     WHERE p.author_id = $2 AND p.deleted_at IS NULL
     ORDER BY p.id DESC LIMIT 200
    ON CONFLICT DO NOTHING`, [followerId, followeeId]);
}
```

```sql
-- TRIM (I6): keep each timeline bounded. Runs continuously, batched.
WITH victims AS (
  SELECT user_id, post_id FROM (
    SELECT user_id, post_id,
           row_number() OVER (PARTITION BY user_id ORDER BY post_id DESC) AS rn
    FROM timeline WHERE user_id = ANY($1)
  ) t WHERE rn > 800
)
DELETE FROM timeline t USING victims v
 WHERE t.user_id = v.user_id AND t.post_id = v.post_id;
```

⚠ The trim job is a **large source of dead tuples**. Tune it explicitly:

```sql
ALTER TABLE timeline SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_vacuum_cost_delay = 2,
  fillfactor = 90
);
```

---

## 8. THE NUMBERS

| Metric | Attempt 1 (read) | Attempt 1 + LATERAL | Attempt 2 (write) | Production (hybrid) |
|---|---|---|---|---|
| Timeline read p99, 400 follows | 2,842 ms | 183 ms | **0.04 ms** | **2.1 ms** |
| Timeline read p99, 5,000 follows | 41,200 ms | 2,310 ms | 0.04 ms | **8.4 ms** |
| Post write (median author) | 1 row, 2 ms | same | 180 rows, 4 ms | **1 row, 2 ms** (async fan-out) |
| Post write (2M-follower author) | 1 row, 2 ms | same | **94 s ✗** | **1 row, 2 ms** (pulled) |
| Post write (41M-follower author) | 1 row, 2 ms | same | **32 min ✗** | **1 row, 2 ms** (pulled) |
| Timeline writes/sec | 0 | 0 | **48,000,000 ✗** | **880,000** |
| Timeline storage (after trim) | 0 | 0 | 166 TB/day | **2.1 TB total** |
| API p99 for `POST /posts` | 8 ms | 8 ms | unbounded | **12 ms** (bounded) |
| Block takes effect | immediately | immediately | needs a purge | **immediately** (read filter) |

**The two headline numbers:** timeline writes 48M/s → 880K/s (**55× reduction**), and the worst-case post write goes from 32 minutes to 12 milliseconds.

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Fan-out worker lags** | New posts missing from timelines | Posts are in `posts` regardless (I2 preserved eventually). Alert on outbox age. **Degrade gracefully**: if lag > 30 s, the read path can pull recent posts from *all* followees as a fallback for that user |
| **A user crosses the threshold mid-fan-out** | Half pushed, half pulled | Harmless — the read path merges and dedups by `post_id` |
| **Kafka redelivers a fan-out job** | Duplicate timeline rows | `ON CONFLICT (user_id, post_id) DO NOTHING` (I5) |
| **A celebrity is demoted** | Followers miss their last 800 posts | Backfill job; hysteresis makes it rare. Alert if demotions/day > 500 |
| **Redis block-set unavailable** | I4 at risk | ★ **Fail closed**: if the block set can't be read, fall back to a DB query. Never serve an unfiltered timeline — this is a safety requirement, not a performance one |
| **Trim job falls behind** | Timelines grow unbounded; `timeline` bloats | Alert on `max(rows per user)`. The read path only ever reads the top 20, so it degrades gracefully — but storage and vacuum do not |
| **A user follows 40,000 accounts** | Pull merge is huge | Cap follows (product decision) or cap the pull at the 40 most-recently-active celebrities and accept incompleteness for that user |
| **Post deleted** | Must vanish within 5 s (I3) | Timeline rows remain; hydration (step g) drops them. A lazy cleanup job removes rows for deleted posts |
| **Hot partition** | One hash partition takes disproportionate write load | `PARTITION BY HASH (user_id)` distributes by follower, not author — so a celebrity's fan-out spreads across all 64 partitions ✓ |

**The five alerts:**

```sql
-- 1. Fan-out lag — the core freshness SLO
SELECT max(now() - created_at) FROM outbox WHERE published_at IS NULL;
-- ALERT if > 20 s

-- 2. Trim backlog
SELECT max(cnt) FROM (SELECT user_id, count(*) cnt FROM timeline
  WHERE user_id = ANY(sample_users()) GROUP BY 1) t;
-- ALERT if > 2000

-- 3. Threshold drift — is T still correct?
SELECT count(*) FROM users WHERE is_celebrity;
SELECT round(avg(c)) FROM (SELECT follower_id, count(*) c
  FROM celebrity_follows GROUP BY 1) x;
-- ALERT if avg celebrities followed > 60 (read budget breach)

-- 4. timeline table bloat (the trim job creates a lot of dead tuples)
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables
WHERE relname LIKE 'timeline%' ORDER BY n_dead_tup DESC;

-- 5. Fan-out write rate vs budget
SELECT sum(n_tup_ins) FROM pg_stat_user_tables WHERE relname LIKE 'timeline%';
-- ALERT if the rate exceeds 1.2M/s
```

---

## 10. WHAT BREAKS AT 10×

**800M users, 3M timeline reads/s, 200,000 posts/s.**

| Wall | Why | Next design |
|---|---|---|
| Single PostgreSQL write capacity | 8.8M timeline writes/s | **Shard `timeline` by `user_id`.** It shards perfectly — a timeline read touches exactly one user, so there are no cross-shard queries (Topic 60) |
| `timeline` storage | 21 TB | Move the materialised timeline out of PostgreSQL entirely: Redis sorted sets (`ZADD user:N post_id`) or a purpose-built store. It is **derived and rebuildable** — losing it costs a backfill, not data (standing rule #3) |
| The pull merge at p99 | 380 celebrities followed | Cap the pull at the 40 most-recently-active; accept incompleteness for extreme followers. Or add a **second tier**: "mega-celebrities" whose posts are served from a global cache |
| Threshold recomputation | full `follows` aggregate over 40B rows | Maintain `follower_count` incrementally via CDC (Topic 76) rather than a periodic aggregate |
| Read fan-out to 3M/s | 200 pods each doing 2 queries + Redis | Precompute the *merged* page-1 timeline into Redis for active users, invalidated on new posts. Falls back to the live merge on a miss |

---

## 11. THE SEVEN QUESTIONS

1. Why is fan-out on read slow even with a perfect index on `(author_id, created_at DESC)`? Name the specific thing no index can provide.
2. Compute the aggregate timeline writes/sec for pure fan-out on write, from the brief's numbers. Show your working.
3. How is the celebrity threshold chosen? Name the two constraints it must simultaneously satisfy and what happens if you set it too low, and too high.
4. Why does promotion (ordinary → celebrity) cost nothing while demotion costs a backfill? What design element makes demotion rare?
5. Why must blocks be filtered at read time rather than purged from timelines? Name the invariant and why the write-based approach violates it.
6. Why is `post_id` a Snowflake/ULID rather than a `bigserial`? Give two distinct benefits.
7. `timeline` is partitioned by `HASH(user_id)`, not by author or by time. Explain why each of the alternatives would be worse.

---

## 12. TRANSFERABLE LESSONS

**1. When the data follows a power law, use two strategies with a threshold.**
Any system where one entity can have 6 orders of magnitude more relationships than the median — followers, order lines per customer, events per device, tenants per shard — will break under a single uniform strategy. The design is: cheap strategy for the head, cheap strategy for the tail, and an explicitly computed boundary. → Reappears in **13 (leaderboards)**, **15 (multi-tenant whales)**, **19 (video view counts)**.

**2. Compute the threshold; don't guess it.**
It is the intersection of two independently-measured budgets (write capacity and read latency). Write the query that produces the trade-off table on your real distribution, and **recompute it quarterly** — the distribution moves. A guessed threshold is the difference between 880K and 12M writes/second.

**3. Add hysteresis to any threshold that triggers work.**
Promote at 10,000, demote at 8,000. Without a gap, entities oscillating near the boundary flap between strategies and each flip costs a migration. This applies to autoscaling, cache tiers, and shard rebalancing equally.

**4. A materialised view of anything is a *candidate set*, not the answer.**
The timeline is precomputed for speed, then filtered at read time for correctness (deletes, blocks). Trying to keep the materialisation perfectly correct means write amplification on every safety-critical path. **Precompute for speed; filter for correctness.** → Reappears in **10 (search results)**, **14 (ad candidate sets)**, **13 (leaderboards)**.

**5. Make the safety-critical path a read, not a write.**
Blocking is a safety feature. Implementing it as "purge N timeline rows" means a slow or failed write leaves harmful content visible. Implementing it as a read-time filter means it works immediately and cannot fail open. **Ask of every safety feature: does it fail open or closed?**

**6. Asynchronous fan-out with a bounded, batched, resumable worker.**
The API returns after one row. Fan-out goes through an outbox → queue → batched worker. Every batch is a short transaction, so a 2M-row fan-out never holds a lock or a snapshot for more than milliseconds. → The batching rule from **01**, applied at 100× the scale.

**7. Time-sortable IDs turn a sort into a scan.**
Because `post_id` is monotonic in time, `ORDER BY post_id DESC` is chronological, the merge of two streams is a trivial O(n) walk, and the B-tree appends instead of splitting randomly. One ID choice removed a column, a sort, and a write-amplification problem. → **Topic 22**, and **06 (chat)**.

---

## FILES TO READ NEXT

- **06 — Chat / messaging** (this folder): the same fan-out question with a different shape — bounded recipients but unbounded per-user read state
- **19 — Video platform metadata** (this folder): the power-law lesson applied to view counts
- **55 — Denormalisation patterns** (curriculum): the timeline is a precomputed aggregate; this is the theory
- **60 — Sharding** (curriculum): why `timeline` shards perfectly and `posts` doesn't
- **52 — Idempotency and transactional messaging** (curriculum): the outbox and `ON CONFLICT DO NOTHING`
- **22 — Primary keys** (curriculum): Snowflake/ULID, and why it matters here
