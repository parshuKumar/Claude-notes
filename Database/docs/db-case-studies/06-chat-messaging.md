# 06 — Chat / Messaging
## When the read-state table is bigger than the messages table

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## WHY THIS CASE STUDY EXISTS

Case study 05 was fan-out with an unbounded blast radius — one post, 41 million writes.

Here the fan-out is **bounded** (a channel has a known membership) and the messages themselves are easy. The difficulty is somewhere most people don't look: **per-user, per-channel read state.**

One message sent to a 5,000-member channel is one row. But "is this unread for me?" and "how many unread do I have?" are questions asked **per user per channel, on every app open, for every channel in the sidebar**. That's a read-state problem, and the naive schema for it generates more rows and more writes than all the messages combined.

The generalisable lesson: **when N users each hold an opinion about M objects, you have an N×M problem, and you must find a way to make it N+M.**

---

## 1. THE BRIEF

A team-chat platform (think Slack).

> **2,000,000 concurrent WebSocket connections. 100,000 messages/sec at peak.**
> **40M users, 8M channels. Largest channel: 180,000 members. Median: 12.**
> **Messages retained 5 years for compliance; searchable.**

Business rules:

1. **Ordering is strict** within a channel. Two messages sent in the same millisecond must have a stable, total order that every client agrees on.
2. **Unread counts** appear on every channel in the sidebar, on every app open. A user may be in 200 channels.
3. **Read receipts** — for DMs and small channels, show who has read a message.
4. **Threads** — a message can have replies; the thread has its own unread state.
5. **Edits and deletes** — a message can be edited (with history) and deleted (tombstoned, not erased — compliance).
6. **Catch-up**: opening the app after a week must load efficiently — "everything since my last read", not "everything".
7. **Search** across 5 years of messages, scoped to channels the user can see.
8. **Retention**: messages older than 5 years are purged; some workspaces set 30 days.

Infrastructure: PostgreSQL 16 (primary + 4 replicas), Redis cluster, Kafka, 400 Node.js pods.

---

## 2. ACCESS PATTERN TABLE

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness |
|---|---|---|---|---|---|---|
| A | `GET /channels` — sidebar with unread counts (200 channels) | 60,000/s | **200 counts** | 0 | **150 ms** | 3 s |
| B | `GET /channels/:id/messages` — latest 50 | 90,000/s | 50 | 0 | 100 ms | 1 s |
| C | `GET /channels/:id/messages?before=` — scrollback | 25,000/s | 50 | 0 | 200 ms | 30 s |
| D | `POST /messages` | 100,000/s | 2 | 1 (+ fan-out) | 200 ms | — |
| E | `POST /read` — mark channel read up to message X | 40,000/s | 1 | 1 | 150 ms | 3 s |
| F | Read receipts for a message (small channels only) | 8,000/s | ~12 | 0 | 200 ms | 5 s |
| G | `PATCH /messages/:id` — edit | 2,000/s | 2 | 2 | 300 ms | 1 s |
| H | Search | 4,000/s | ~50 | 0 | 400 ms | 60 s |
| I | Retention purge | nightly | ~ | ~ | n/a | n/a |

**What jumps out:**

- **A is the whole problem.** 60,000 requests/sec × 200 channels = **12 million unread counts computed per second**. Nothing about the message table is that hard.
- **E at 40,000/s** is a *write* per user per channel — this is the N×M table, and it churns constantly.
- **D at 100,000/s** is high but structurally easy: append-only, one row, bounded fan-out.
- **F is only for small channels.** That's a product decision that saves the design — read receipts for a 180,000-member channel would be 180,000 rows per message.

---

## 3. THE INVARIANTS

```
I1.  TOTAL ORDER: within a channel, messages have a strict total order
     agreed by all clients, stable across retries and clock skew.

I2.  MONOTONIC READ STATE: a user's last_read marker never moves
     backwards. (Two devices racing must not un-read messages.)

I3.  NO LOST MESSAGES: a message accepted by the API (2xx) is
     eventually visible to every current channel member.

I4.  IDEMPOTENT SEND: the same client message, retried, appears once.

I5.  MEMBERSHIP GATES VISIBILITY: a user sees only messages in channels
     they were a member of at the time — including history rules
     ("new members see history" vs "only from join time").

I6.  TOMBSTONES, NOT DELETES: a deleted message leaves a record
     (compliance). Content is erased; existence is not.
```

**I1 is deceptively hard.** With 400 pods generating messages, `now()` from the application gives you clock skew, and a `bigserial` from the database gives you gaps and contention. The resolution shapes the primary key, which shapes everything else.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA

```sql
CREATE TABLE channels (
  id bigserial PRIMARY KEY, workspace_id bigint NOT NULL, name text NOT NULL
);

CREATE TABLE channel_members (
  channel_id bigint NOT NULL REFERENCES channels(id),
  user_id    bigint NOT NULL REFERENCES users(id),
  joined_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (channel_id, user_id)
);

CREATE TABLE messages (
  id         bigserial   PRIMARY KEY,
  channel_id bigint      NOT NULL REFERENCES channels(id),
  sender_id  bigint      NOT NULL REFERENCES users(id),
  body       text        NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  edited_at  timestamptz NULL,
  deleted_at timestamptz NULL
);
CREATE INDEX idx_messages_channel ON messages (channel_id, created_at DESC);

-- ★ THE N×M TABLE — one row per user per message
CREATE TABLE message_reads (
  message_id bigint NOT NULL REFERENCES messages(id),
  user_id    bigint NOT NULL REFERENCES users(id),
  read_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (message_id, user_id)
);
```

```sql
-- the sidebar query (pattern A)
SELECT c.id, c.name,
       (SELECT count(*) FROM messages m
         WHERE m.channel_id = c.id
           AND NOT EXISTS (SELECT 1 FROM message_reads r
                            WHERE r.message_id = m.id AND r.user_id = $1)
       ) AS unread
FROM channels c
JOIN channel_members cm ON cm.channel_id = c.id AND cm.user_id = $1;
```

This is what people write. It is precisely correct, and it dies on contact with reality.

---

## 5. WHERE IT BREAKS

### 5.1 The read-state table is bigger than everything else combined

```
 100,000 messages/sec.
 Average channel membership (weighted by message volume): 48 members.

 message_reads rows generated = 100,000 × 48 = 4,800,000 rows/second

 vs messages:                                     100,000 rows/second
                                                  ─────────────────────
 THE READ-STATE TABLE IS 48× THE MESSAGE TABLE.

 Per day:  414,720,000,000 rows.   ~415 BILLION rows/day.
 At 40 B/row: 16.6 TB/day, before indexes.

 ⇒ Not a tuning problem. The DESIGN generates more data than the
   product does.
```

And it isn't even correct: a user in a 180,000-member channel who never opens it generates no rows, so "unread" can't be computed by counting reads — you'd have to count the *absence* of rows, which is the `NOT EXISTS` above.

### 5.2 The sidebar query is catastrophic

```sql
EXPLAIN (ANALYZE, BUFFERS) <the sidebar query for a user in 200 channels>;
```
```
Nested Loop  (actual time=8.1..18402.4 rows=200 loops=1)
  ->  Index Scan using channel_members_pkey on channel_members cm
        (actual rows=200 loops=1)
  ->  Index Scan using channels_pkey on channels c (loops=200)
  SubPlan 1
    ->  Aggregate (actual time=91.2..91.2 rows=1 loops=200)      ← ⚠ ×200
          ->  Index Scan using idx_messages_channel on messages m
                (actual rows=41204 loops=200)                     ← ⚠ 8.2M rows
                Filter: (NOT (SubPlan 2))
                SubPlan 2
                  ->  Index Scan using message_reads_pkey
                        (actual rows=1 loops=8240800)             ← ⚠⚠ 8.2M
  Buffers: shared hit=48204112 read=8204882
Execution Time: 18403.1 ms
```

**18.4 seconds to draw a sidebar.** The correlated `NOT EXISTS` runs once per message per channel — 8.2 million index lookups for one page load.

At 60,000 requests/sec this needs 1.1 million concurrent executions. There is no hardware.

### 5.3 `bigserial` breaks ordering under concurrency

```
 I1 requires a total order agreed by all clients. `bigserial` seems fine:
 it's monotonic. But:

  t=0.000  pod A: nextval() → 1001, begins transaction
  t=0.001  pod B: nextval() → 1002, begins transaction
  t=0.002  pod B: COMMITS.   Message 1002 is now visible.
  t=0.003  client polls, gets messages up to 1002, records cursor=1002
  t=0.004  pod A: COMMITS.   Message 1001 becomes visible.
  t=0.005  client polls "since 1002" → ★ NEVER SEES MESSAGE 1001.

 ⇒ Sequence values are allocated in order but COMMIT in any order.
   Any cursor based on a sequence silently drops messages.
   This violates I3, and it is invisible in testing because the window
   is milliseconds wide.
```

And `nextval()` on one sequence at 100,000/s is itself a contention point.

### 5.4 Scrollback degrades with channel age

```sql
EXPLAIN (ANALYZE) SELECT * FROM messages
WHERE channel_id = 4471 ORDER BY created_at DESC LIMIT 50 OFFSET 20000;
```
```
Limit  (actual time=1204.8..1206.1 rows=50 loops=1)
  ->  Index Scan using idx_messages_channel (actual rows=20050 loops=1)
Execution Time: 1206.4 ms
```
`OFFSET 20000` reads and discards 20,000 rows. Page 400 of a busy channel is a full scan of everything newer.

### 5.5 The messages table grows without bound

5 years × 100,000/s = **15.8 trillion rows**. No partitioning, no retention strategy, and a nightly `DELETE` for expired workspaces would generate billions of dead tuples (Topic 47).

---

## 6. ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH

**Replace per-message read state with a per-channel watermark.** Instead of "which messages have I read," store "the last message I read."

```sql
DROP TABLE message_reads;

CREATE TABLE channel_read_state (
  channel_id      bigint NOT NULL,
  user_id         bigint NOT NULL,
  last_read_msg_id bigint NOT NULL,
  updated_at      timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (channel_id, user_id)
);
```

```sql
-- unread count becomes a range count
SELECT count(*) FROM messages
WHERE channel_id = $1 AND id > $2;      -- $2 = last_read_msg_id
```

**This is a genuine, large win — the key insight of the whole case study:**

```
 message_reads:       4,800,000 rows/sec, unbounded growth
 channel_read_state:  ONE ROW PER (user, channel). 40M users × avg 30
                      channels = 1.2 BILLION rows TOTAL, not per day.
                      Updated 40,000/s (pattern E), never inserted after
                      the first read.

 ⇒ From N×M growing forever, to N×M fixed and small.
   ★ 415 billion rows/day → 1.2 billion rows, total.
```

Measure the sidebar:

```
 18,403 ms → 412 ms
```

**45× better. And still 3× over budget, with three problems remaining:**

| Still broken | Why the watermark doesn't fix it |
|---|---|
| 200 `count(*)` per sidebar load | each is an index range scan over potentially thousands of rows. 200 × 2 ms = 400 ms |
| Ordering (5.3) | `bigserial` still commits out of order |
| Scrollback `OFFSET` | unchanged |
| Unbounded growth | unchanged |
| `channel_read_state` churn | 40,000 updates/s on 1.2B rows = massive MVCC pressure (Topic 46) |

**And a new one:** `channel_read_state` is updated 40,000 times/second. Each update is a non-HOT row rewrite if `last_read_msg_id` is indexed, and the table bloats fast (Topic 17).

---

## 7. THE PRODUCTION DESIGN

```
 PROBLEM                            SOLUTION                          TOPIC
 ────────────────────────────────────────────────────────────────────────
 Out-of-order commits break I1  →  Snowflake IDs + per-channel seq     22
 200 count(*) per sidebar       →  Denormalised counters + Redis       55,57
 Read-state churn               →  Write-behind through Redis          57
 OFFSET scrollback              →  Keyset pagination on the ID          —
 Unbounded growth               →  Partition by time + DROP             59
 Search over 5 years            →  Derived index, not the OLTP table    76
```

### 7.1 The ID: Snowflake, plus a per-channel sequence

```
 SNOWFLAKE (64 bits, generated APP-SIDE):
 ┌──────────────────────────┬────────────┬──────────────┐
 │ timestamp_ms (41 bits)   │ node (10)  │ counter (12) │
 └──────────────────────────┴────────────┴──────────────┘
   69 years from epoch        1024 pods    4096/ms/pod

 ✓ TIME-SORTABLE → ORDER BY id IS chronological (no created_at needed
   in the index, and B-tree inserts append — Topic 11)
 ✓ GENERATED APP-SIDE → no sequence contention, no round trip
 ✓ GLOBALLY UNIQUE → safe across shards
 ✗ ⚠ STILL COMMITS OUT OF ORDER. Snowflake fixes allocation order,
   not commit order. 5.3 is NOT solved by Snowflake alone.

 ⇒ THE SECOND HALF: a PER-CHANNEL SEQUENCE, assigned inside the
   transaction, under a per-channel advisory lock:

     seq_no  1, 2, 3, …   monotonic and gapless WITHIN a channel

   Because it's assigned under a lock held to COMMIT, a client that has
   seen seq_no=42 can be certain seq_no=41 is already visible.
   ⇒ Cursors use seq_no. I1 and I3 are both satisfied.

 ★ THE TRADE: a per-channel advisory lock serialises sends WITHIN one
   channel. At a realistic max of ~50 messages/sec in a single channel
   and ~1 ms hold time, that is far from the ceiling. Across 8M channels
   there is no global contention. (Contrast case study 01, where every
   request hit ONE row.)
```

### 7.2 The schema

```sql
-- ═══════════════════════════════════════════════════════════════════
-- CHANNELS — carries the denormalised counter that makes A possible
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE channels (
  id            bigint      PRIMARY KEY,          -- snowflake
  workspace_id  bigint      NOT NULL,
  name          text        NOT NULL,
  kind          smallint    NOT NULL,             -- 1=public 2=private 3=dm
  history_mode  smallint    NOT NULL DEFAULT 1,   -- 1=all 2=from_join (I5)
  member_count  int         NOT NULL DEFAULT 0,
  last_seq_no   bigint      NOT NULL DEFAULT 0,   -- ★ the per-channel counter
  last_msg_id   bigint      NULL,
  retention_days int        NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
) WITH (fillfactor = 80);                          -- ★ last_seq_no is hot
CREATE INDEX idx_channels_workspace ON channels (workspace_id);

-- ═══════════════════════════════════════════════════════════════════
-- MEMBERSHIP — and this is where read state lives (N×M, but bounded)
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE channel_members (
  channel_id       bigint      NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
  user_id          bigint      NOT NULL REFERENCES users(id)    ON DELETE CASCADE,
  joined_at        timestamptz NOT NULL DEFAULT now(),
  joined_seq_no    bigint      NOT NULL,          -- ★ I5: history boundary

  -- ── READ STATE, colocated with membership. One row, not N. ──
  last_read_seq_no bigint      NOT NULL DEFAULT 0,
  unread_count     int         NOT NULL DEFAULT 0,   -- ★ DENORMALISED
  mention_count    int         NOT NULL DEFAULT 0,
  muted            boolean     NOT NULL DEFAULT false,

  PRIMARY KEY (channel_id, user_id)
) WITH (fillfactor = 70);                          -- ★ updated constantly
-- ★ THE SIDEBAR INDEX: everything pattern A needs, in one index-only scan
CREATE INDEX idx_members_user ON channel_members (user_id)
  INCLUDE (channel_id, unread_count, mention_count, muted, last_read_seq_no);

-- ═══════════════════════════════════════════════════════════════════
-- MESSAGES — partitioned, append-only, time-sortable
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE messages (
  id          bigint      NOT NULL,               -- snowflake
  channel_id  bigint      NOT NULL,
  seq_no      bigint      NOT NULL,               -- ★ per-channel, gapless
  sender_id   bigint      NOT NULL,
  body        text,                               -- NULL when tombstoned
  thread_root_id bigint   NULL,                   -- threads (rule 4)
  reply_count int         NOT NULL DEFAULT 0,
  edited_at   timestamptz NULL,
  deleted_at  timestamptz NULL,                   -- ★ I6: tombstone
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (created_at, id)
) PARTITION BY RANGE (created_at);
-- ★ THE ONLY INDEX PATTERNS B AND C NEED
CREATE UNIQUE INDEX idx_messages_channel_seq ON messages (channel_id, seq_no DESC);
CREATE INDEX idx_messages_thread ON messages (thread_root_id, seq_no)
  WHERE thread_root_id IS NOT NULL;

-- I4: idempotent send
CREATE TABLE message_idempotency (
  client_msg_id uuid        PRIMARY KEY,
  message_id    bigint      NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);                 -- 24h retention

-- I6 + rule 5: edit history
CREATE TABLE message_revisions (
  message_id bigint      NOT NULL,
  revision   smallint    NOT NULL,
  body       text        NOT NULL,
  edited_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (message_id, revision)
) PARTITION BY RANGE (edited_at);

-- ═══════════════════════════════════════════════════════════════════
-- READ RECEIPTS — ONLY for small channels. A product-gated N×M table.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE message_receipts (
  channel_id bigint NOT NULL,
  seq_no     bigint NOT NULL,
  user_id    bigint NOT NULL,
  read_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (channel_id, seq_no, user_id)
) PARTITION BY RANGE (read_at);
-- ★ Written ONLY when channels.member_count <= 50. This single product
--   constraint is what keeps the table from being 415 billion rows/day.
```

**Every decision justified:**

| Decision | Why |
|---|---|
| `seq_no` per channel | I1 + I3 — gapless, so a cursor cannot skip a message |
| Snowflake `id` | time-sortable, app-generated, shard-safe (Topic 22) |
| `unread_count` **on `channel_members`** | ★ turns 200 `count(*)` into one index-only scan |
| `INCLUDE` on `idx_members_user` | pattern A becomes index-only: zero heap fetches (Topic 12) |
| `fillfactor = 70` on `channel_members` | updated 40,000/s — HOT updates are essential (Topic 17) |
| `joined_seq_no` | I5 — the history boundary, without a separate table |
| `messages` partitioned by `created_at` | retention = `DROP PARTITION`; keeps the active index in RAM (Topic 11) |
| `deleted_at` + NULL `body` | I6 — the tombstone survives, the content doesn't |
| `message_receipts` gated at 50 members | the only thing preventing an N×M explosion |

### 7.3 The send path

```js
async function sendMessage({ channelId, senderId, body, clientMsgId, threadRootId }) {
  const msgId = snowflake();

  return withTransaction(async (tx) => {
    await tx.query("SET LOCAL lock_timeout = '500ms'");

    // ── I4: idempotency, enforced by a unique index, not a SELECT ──
    const idem = await tx.query(
      `INSERT INTO message_idempotency (client_msg_id, message_id)
       VALUES ($1,$2) ON CONFLICT (client_msg_id) DO NOTHING RETURNING message_id`,
      [clientMsgId, msgId]);
    if (idem.rowCount === 0) {
      const prev = await tx.query(
        'SELECT message_id FROM message_idempotency WHERE client_msg_id=$1', [clientMsgId]);
      return { replayed: true, messageId: prev.rows[0].message_id };
    }

    // ── I1: serialise ONLY this channel. Not a row lock — an advisory
    //        lock, so no MVCC version, no bloat, released at COMMIT. ──
    await tx.query('SELECT pg_advisory_xact_lock($1)', [channelId]);

    const seq = await tx.query(
      `UPDATE channels SET last_seq_no = last_seq_no + 1, last_msg_id = $2
        WHERE id = $1 RETURNING last_seq_no`, [channelId, msgId]);
    const seqNo = seq.rows[0].last_seq_no;

    await tx.query(
      `INSERT INTO messages (id, channel_id, seq_no, sender_id, body, thread_root_id)
       VALUES ($1,$2,$3,$4,$5,$6)`,
      [msgId, channelId, seqNo, senderId, body, threadRootId]);

    if (threadRootId) {
      await tx.query('UPDATE messages SET reply_count = reply_count + 1 WHERE id = $1',
                     [threadRootId]);
    }

    // ── No dual write. The outbox carries the fan-out. (Topic 52) ──
    await tx.query(
      `INSERT INTO outbox (topic, payload) VALUES ('message.sent', $1)`,
      [JSON.stringify({ channelId, messageId: msgId, seqNo, senderId })]);

    return { messageId: msgId, seqNo };
  });
}
```

**The API returns here.** Unread-count fan-out is asynchronous.

### 7.4 The unread fan-out — batched, and mostly in Redis

```js
// Kafka consumer. Bounded batches, resumable.
async function unreadFanOut({ channelId, seqNo, senderId }) {
  const BATCH = 5_000;
  let cursor = 0;
  for (;;) {
    const { rows } = await db.query(`
      WITH batch AS (
        SELECT user_id FROM channel_members
         WHERE channel_id = $1 AND user_id > $2 AND user_id <> $3 AND NOT muted
         ORDER BY user_id LIMIT $4
      )
      UPDATE channel_members cm
         SET unread_count = cm.unread_count + 1
        FROM batch b
       WHERE cm.channel_id = $1 AND cm.user_id = b.user_id
      RETURNING cm.user_id`,
      [channelId, cursor, senderId, BATCH]);

    if (rows.length === 0) break;
    cursor = rows[rows.length - 1].user_id;

    // push to connected sockets — Redis knows who's online
    await redis.publish(`ch:${channelId}`, JSON.stringify({ seqNo }));
  }
}
```

⚠ **For very large channels this is still expensive** — 180,000 row updates per message. The mitigation:

```
 LARGE-CHANNEL RULE (member_count > 5,000):
   • do NOT maintain unread_count per member
   • the sidebar computes it lazily for those channels:
       unread = channels.last_seq_no - channel_members.last_read_seq_no
     ⇒ ★ AN ARITHMETIC SUBTRACTION, not a count(*), and both values are
       already in the sidebar's index-only scan.
   • flag it: channel_members.unread_count = -1 means "compute lazily"

 ⇒ THE SAME HYBRID SHAPE AS CASE STUDY 05: maintain eagerly where the
   fan-out is cheap, compute lazily where it isn't. The threshold is
   again chosen by arithmetic, not intuition.
```

### 7.5 The sidebar — pattern A, solved

```sql
SELECT cm.channel_id, c.name, c.kind, cm.mention_count, cm.muted,
       CASE WHEN cm.unread_count >= 0
            THEN cm.unread_count                              -- eager
            ELSE GREATEST(c.last_seq_no - cm.last_read_seq_no, 0)  -- lazy
       END AS unread
FROM channel_members cm
JOIN channels c ON c.id = cm.channel_id
WHERE cm.user_id = $1
ORDER BY c.last_msg_id DESC NULLS LAST
LIMIT 200;
```
```
Limit  (actual time=0.31..0.88 rows=200 loops=1)
  ->  Nested Loop (actual rows=200 loops=1)
        ->  Index Only Scan using idx_members_user on channel_members cm
              Index Cond: (user_id = 4471)
              Heap Fetches: 0                      ← ★ covering index
              (actual rows=200 loops=1)
        ->  Index Scan using channels_pkey on channels c (loops=200)
  Buffers: shared hit=412
Execution Time: 0.94 ms
```

**18,403 ms → 0.94 ms — 19,600×.** And with the Redis layer in front (3 s TTL, invalidated by the fan-out), PostgreSQL sees ~600 of these per second instead of 60,000.

### 7.6 Marking read — write-behind, because 40,000/s of UPDATEs would bloat

```js
// I2: monotonic. Redis absorbs the churn; PostgreSQL is flushed periodically.
async function markRead({ userId, channelId, seqNo }) {
  // ★ Lua guarantees the marker never moves backwards, even across devices
  await redis.eval(`
    local cur = tonumber(redis.call('HGET', KEYS[1], ARGV[1]) or '0')
    if tonumber(ARGV[2]) > cur then
      redis.call('HSET', KEYS[1], ARGV[1], ARGV[2])
      redis.call('SADD', KEYS[2], KEYS[1]..':'..ARGV[1])
    end
    return 1`,
    2, `read:${userId}`, 'read:dirty', String(channelId), String(seqNo));
}

// Flush worker: every 2 s, batched
async function flushReadState(batch) {           // [{userId, channelId, seqNo}]
  await db.query(`
    UPDATE channel_members cm
       SET last_read_seq_no = GREATEST(cm.last_read_seq_no, b.seq_no),
           unread_count = CASE WHEN cm.unread_count < 0 THEN -1 ELSE 0 END,
           mention_count = 0
      FROM jsonb_to_recordset($1::jsonb)
             AS b(user_id bigint, channel_id bigint, seq_no bigint)
     WHERE cm.user_id = b.user_id AND cm.channel_id = b.channel_id
       AND cm.last_read_seq_no < b.seq_no`,       -- ★ I2: never backwards
    [JSON.stringify(batch)]);
}
```

```
 40,000 individual UPDATEs/sec  →  ~20 batched statements/sec of 2,000 rows.
 ⇒ 2,000× fewer transactions, and the MVCC churn collapses because
   `fillfactor = 70` keeps almost all of them HOT (Topic 17).
```

### 7.7 Scrollback — keyset pagination

```sql
-- ✗ OFFSET: reads and discards everything before the page
SELECT * FROM messages WHERE channel_id=$1 ORDER BY seq_no DESC LIMIT 50 OFFSET 20000;

-- ✓ KEYSET: one index descent, 50 rows, constant cost at ANY depth
SELECT id, seq_no, sender_id, body, edited_at, deleted_at, created_at
FROM messages
WHERE channel_id = $1 AND seq_no < $2          -- $2 = cursor from the last page
ORDER BY seq_no DESC LIMIT 50;
```
```
 OFFSET 20000: 1,206 ms · 20,050 rows read
 KEYSET:           0.4 ms ·     50 rows read     ← 3,000×, and CONSTANT
```

⚠ **Partition pruning caveat:** `messages` is partitioned by `created_at`, but this query filters on `seq_no`. PostgreSQL cannot prune partitions from a `seq_no` predicate alone. The fix — carry the timestamp in the cursor:

```sql
WHERE channel_id = $1 AND seq_no < $2 AND created_at <= $3   -- $3 from the cursor
```

`seq_no` and `created_at` are monotonic together within a channel, so this is safe and enables pruning.

### 7.8 Retention and search

```sql
-- Retention: DROP, never DELETE (Topic 59)
DROP TABLE messages_2021_03;      -- O(1), zero dead tuples

-- Per-workspace retention (some are 30 days) — a targeted job, batched
DELETE FROM messages m USING channels c
 WHERE m.channel_id = c.id AND c.retention_days IS NOT NULL
   AND m.created_at < now() - (c.retention_days || ' days')::interval
   AND m.ctid = ANY (ARRAY(SELECT ctid FROM messages ... LIMIT 5000));
```

**Search (pattern H) does not live in PostgreSQL.** 5 years × 100k/s is 15.8 trillion rows; a GIN index on that is not viable (Topic 16's write cost). Search is a **derived store** fed by CDC from the outbox (Topic 76) — rebuildable, and I5 (membership gating) is applied as a filter at query time using the user's channel list.

---

## 8. THE NUMBERS

| Metric | Attempt 1 | Attempt 2 (watermark) | Production |
|---|---|---|---|
| Sidebar (A) p99 | 18,403 ms | 412 ms | **0.94 ms** (0.1 ms via Redis) |
| Sidebar load on PostgreSQL | 60,000/s | 60,000/s | **~600/s** |
| Read-state rows/day | 415 billion | — | **0** (1.2B total, updated in place) |
| Read-state writes/sec | 4,800,000 | 40,000 | **~20 batched statements/s** |
| Scrollback, page 400 | 1,206 ms | 1,206 ms | **0.4 ms** |
| Message send p99 | 8 ms | 8 ms | **11 ms** (async fan-out) |
| Ordering correctness (I1/I3) | **broken** | broken | **guaranteed** |
| Storage, 5 years | 16.6 TB/day read-state | — | **partitioned, DROP-able** |
| Retention purge | hours of DELETEs | same | **O(1) DROP PARTITION** |

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Redis loses read state** | Unread counts revert to the last flush (≤2 s of marks lost) | Acceptable — the user re-marks by scrolling. `GREATEST()` in the flush means a stale value never moves the marker backwards (I2) |
| **Unread fan-out lags** | Counts are stale, messages still arrive | Messages are in `messages` regardless (I3 preserved). Alert on outbox age. The lazy path (`last_seq_no − last_read_seq_no`) is always available as a fallback |
| **Advisory lock contention in one channel** | Sends serialise in that channel | 500 ms `lock_timeout` → fail fast with a retry. Only affects that channel; there is no global lock |
| **A channel crosses 5,000 members** | Eager counters become expensive | Promotion sets `unread_count = -1` for all members (one batched update), switching that channel to lazy |
| **Duplicate send retry** | Two messages | `message_idempotency` unique PK (I4) |
| **Message edited after a client cached it** | Stale body | `edited_at` bumps; clients refetch. Websocket push carries the new revision |
| **Partition not pre-created** | Inserts fail at midnight | `pg_partman` pre-creates 30 days ahead; alert if the newest partition is < 7 days out |
| **Search index diverges** | Missing results | It is **derived and rebuildable** from `messages` (standing rule #3). Reindex from a partition range |
| **`channel_members` bloat** | Sidebar slows | 40k updates/s on 1.2B rows. `fillfactor=70` + aggressive autovacuum. **Monitor HOT ratio — this is the table most at risk** |

**The five alerts:**

```sql
-- 1. Unread fan-out lag
SELECT max(now() - created_at) FROM outbox WHERE published_at IS NULL;
-- ALERT if > 10 s

-- 2. channel_members HOT ratio — the highest-churn table in the system
SELECT n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables WHERE relname='channel_members';
-- ALERT if hot_pct < 90

-- 3. Sequence gaps — I1/I3 violation detector
SELECT channel_id, count(*) AS gaps FROM (
  SELECT channel_id, seq_no,
         lag(seq_no) OVER (PARTITION BY channel_id ORDER BY seq_no) AS prev
  FROM messages WHERE created_at > now() - interval '1 hour') t
WHERE seq_no <> prev + 1 GROUP BY 1;
-- ALERT if > 0.  Should be structurally impossible.

-- 4. Partition runway
SELECT max(to_date(substring(relname from '\d{4}_\d{2}'), 'YYYY_MM'))
FROM pg_class WHERE relname LIKE 'messages_%';
-- ALERT if < 7 days out

-- 5. Advisory lock waits
SELECT count(*) FROM pg_locks WHERE locktype='advisory' AND NOT granted;
-- ALERT if > 50
```

---

## 10. WHAT BREAKS AT 10×

**20M concurrent sockets, 1M messages/sec, 400M users.**

| Wall | Why | Next design |
|---|---|---|
| Single primary write capacity | ~1M inserts/s + fan-out | **Shard by `workspace_id`.** A workspace is a natural boundary — no cross-workspace queries exist, so this shards perfectly (Topic 60) |
| `channel_members` at 12B rows | the sidebar index alone is TBs | Shard with the workspace. Per-shard it returns to ~1.2B |
| Unread fan-out | 1M msgs/s × 48 members = 48M updates/s | Lower the eager/lazy threshold to ~500 members. Most channels become lazy; the arithmetic subtraction costs nothing |
| Redis read-state | 400,000 marks/s | Shard by `user_id`; the flush worker consumes per-shard streams |
| Websocket fan-out | 20M sockets | This stops being a database problem — a dedicated pub/sub tier with per-channel topics |
| Search | 158 trillion rows | Tiered: last 90 days hot in OpenSearch, older in object storage queried on demand |

---

## 11. THE SEVEN QUESTIONS

1. Why does per-message read state generate 48× more rows than messages? Compute the daily figure from the brief.
2. Explain why `bigserial` breaks cursor-based catch-up. Give the exact interleaving, and say why Snowflake IDs alone don't fix it.
3. What does the per-channel `seq_no` guarantee that a global ID cannot? What does it cost, and why is that cost acceptable here but not in case study 01?
4. Why is `unread_count` denormalised onto `channel_members` rather than computed? What makes the sidebar query index-only?
5. Explain the eager/lazy threshold for unread counts. Why is the lazy path a *subtraction* rather than a `count(*)`?
6. Why is read-state marking written through Redis rather than straight to PostgreSQL? Name two distinct problems it solves.
7. `messages` is partitioned by `created_at` but scrollback filters on `seq_no`. What breaks, and how does the cursor design fix it?

---

## 12. TRANSFERABLE LESSONS

**1. N×M per-user state is the hidden scaling problem.**
Whenever N users each hold an opinion about M objects — read/unread, seen, liked, dismissed, permissions — the naive table is N×M and grows forever. Look for a **watermark**: one row per (user, container) recording a position, instead of one row per (user, object). → Reappears in **09 (notification delivery state)**, **18 (parcel subscriptions)**, and any "seen" feature.

**2. A monotonic per-container sequence is worth its contention.**
Global IDs give you uniqueness and rough ordering. Only a **gapless per-container sequence** lets a client say "I have everything up to 42" with certainty. The advisory lock that provides it is cheap *because the contention is partitioned by container* — 8M channels means 8M independent locks. Contrast case study 01, where a single row was the contention point. **Partitioned contention is not contention.**

**3. Denormalise the counter, not the data.**
`unread_count` on `channel_members` turns 200 aggregate queries into one index-only scan. The counter is derived and rebuildable — if it drifts, recompute from `last_seq_no − last_read_seq_no`. **A denormalised value you can always recompute is a cache, not a second source of truth.** → Topic 55, and case studies 03, 05.

**4. The hybrid threshold appears again — and again it's arithmetic.**
Eager counters below 5,000 members, lazy subtraction above. Same shape as case study 05's celebrity threshold: cheap strategy for the head, cheap strategy for the tail, boundary computed from measured budgets. **Whenever you see a power law, expect a threshold.**

**5. Write-behind through a fast store when the write rate is churn, not truth.**
Read markers are 40,000 writes/second of information nobody audits. Absorbing them in Redis and flushing in batches cuts transactions 2,000× and eliminates the MVCC bloat that would otherwise dominate the table. **Ask of every high-frequency write: does anyone need this durable to the millisecond?** → Topic 57, and case study 07.

**6. Keyset pagination, always. `OFFSET` is a bug at scale.**
`OFFSET N` reads and discards N rows. Keyset pagination on a monotonic key is constant-cost at any depth. This is universal — feeds, logs, exports, admin tables. → Reappears everywhere.

**7. Product constraints are design tools.**
"Read receipts only for channels under 50 members" is the single decision that prevents a 415-billion-row table. Engineering the impossible version would have cost a quarter; asking whether anyone needs receipts in a 180,000-member channel cost five minutes. **Negotiate the requirement before you engineer around it.**

---

## FILES TO READ NEXT

- **07 — IoT telemetry** (this folder): pure ingest volume, where partitioning and compression are the whole answer
- **09 — Notification delivery** (this folder): the outbox and job-claiming patterns used here, at scale
- **22 — Primary keys** (curriculum): Snowflake/ULID vs sequences, in full
- **59 — Partitioning** (curriculum): the retention design
- **57 — Caching** (curriculum): the write-behind pattern for read state
- **05 — Social feed** (this folder): compare lesson 4 here with lesson 1 there
