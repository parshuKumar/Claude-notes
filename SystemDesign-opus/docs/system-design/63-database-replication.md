# 63 — Database Replication — Master-Slave, Master-Master
## Category: HLD Components

---

## What is this?

**Database replication** means keeping copies of the same data on more than one machine. You write to one database, and that database automatically copies every change to one or more other databases.

Think of a **restaurant with a single handwritten recipe book** in the head chef's office. Every cook has to walk to the office to read a recipe — the office becomes a traffic jam, and if it burns down, the restaurant is finished. So you photocopy the book and put a copy in every kitchen station. Now cooks read locally (fast, no queue), and if the office burns, you still have the recipes. But the moment the head chef edits a recipe, the photocopies are *stale* until someone re-copies them. That gap — that staleness — is the entire subject of this chapter.

---

## Why does it matter?

Replication is the answer to **three completely different problems**, and beginners constantly blur them. Say them out loud in an interview and you instantly sound senior:

1. **Read scaling.** Most apps read 10–100× more than they write. One machine can serve maybe 5,000 reads/sec. Add 5 read replicas and you serve 30,000 reads/sec — without touching your data model.
2. **High availability (HA).** If the one database dies, your product is down. With a replica that is seconds behind, you promote it and you're back in ~30 seconds instead of ~4 hours restoring a backup.
3. **Disaster recovery (DR).** A whole *datacenter* or *region* can be lost — flood, fibre cut, a bad `DROP TABLE`. A replica in another region, or a delayed replica an hour behind, is your insurance policy.

These three motives pull the design in different directions. Read scaling wants many replicas *nearby*. DR wants replicas *far away*. HA wants the failover to be *automatic*, which is exactly what makes split-brain possible.

**Interview angle:** "How do you scale reads?" is a warm-up question in almost every HLD interview, and the follow-up is always "what if the replica is behind?" If you can name read-your-own-writes and monotonic reads violations and fix them, you're ahead of most candidates.

**Work angle:** The first time your read replica lags 40 seconds behind during a bulk import, and support tickets pour in saying "my order disappeared," you'll wish you had read this.

---

## The core idea — explained simply

### The Newspaper Printing Press Analogy

A national newspaper has **one editorial office** in the capital. That's the only place where a story can be *written or changed*. Editors argue, edit, and finally the page is locked.

The locked page is then **wired out to 12 regional printing presses**. Each press prints and distributes to its region. Readers in Chennai read a Chennai-printed copy. Readers in Delhi read a Delhi-printed copy. Nobody drives to the capital to read the news.

Now notice the properties this gives you, for free:

- **Reads scale.** 12 presses print 12× the papers.
- **A press can die.** If Chennai's press catches fire, route Chennai's papers from Bangalore. Slower, but no outage.
- **The office can die.** If the capital office is destroyed, you can *promote* a regional bureau to be the new editorial office.

And notice the one thing it costs you:

- **The wire takes time.** If the office corrects a headline at 11:58pm and the Chennai press already started rolling, Chennai's readers see the old headline. They are reading a **stale** copy. The delay between "the office changed it" and "the press has it" is **replication lag**.

| Analogy piece | Technical concept |
|---|---|
| Editorial office (the only place you can write) | **Leader** / master / primary |
| Regional printing presses (read-only copies) | **Followers** / slaves / read replicas |
| The wire carrying locked pages | **Replication log / WAL stream** |
| Time between office edit and press having it | **Replication lag** |
| Chennai reader sees yesterday's headline | **Stale read** |
| Promoting a regional bureau after the office burns | **Failover** |
| Two bureaus both think they're the office | **Split-brain** |
| A press that intentionally prints 1 hour late | **Delayed replica** (protects against `DROP TABLE`) |

The whole chapter is: **who is allowed to write, and how much staleness do readers tolerate.**

---

## Key concepts inside this topic

### 1. Single-leader replication (master-slave / primary-replica)

This is the default in PostgreSQL, MySQL, MongoDB replica sets, and every managed cloud DB. It is what you should reach for 95% of the time.

**The rule: exactly one node accepts writes.**

```
WRITES ──────▶ ┌──────────┐
               │  LEADER  │  (the only writable node)
               └────┬─────┘
                    │  replication log stream
        ┌───────────┼───────────┐
        ▼           ▼           ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │FOLLOWER1│ │FOLLOWER2│ │FOLLOWER3│   (read-only)
  └─────────┘ └─────────┘ └─────────┘
        ▲           ▲           ▲
        └───────── READS ───────┘
```

Because there is only one writer, **there are no write conflicts, ever.** That single fact is why this topology is so overwhelmingly popular. All the complexity you're about to read about in multi-leader disappears.

**How the replication log actually streams.** Every relational database already writes a **WAL (Write-Ahead Log)** — an append-only file of "here is exactly what changed on disk," written *before* the change is applied, so a crash can be recovered. Replication is a beautifully cheap trick: just **ship that same log to another machine and let it replay it.**

```
On the LEADER:
  1. Client sends:  UPDATE accounts SET balance = 90 WHERE id = 7;
  2. Leader appends to WAL:  [LSN 4821] page 12: bytes 340-348 = 90
  3. Leader applies the change to its own data pages
  4. Leader ACKs the client
  5. A "walsender" process streams WAL records from LSN 4821 onward
     over a TCP connection to each follower

On each FOLLOWER:
  6. "walreceiver" receives WAL records, writes them to local WAL
  7. A replay process applies them in exactly the same order
  8. Follower reports back: "I have received up to LSN 4821,
     flushed to 4820, replayed up to 4815"
```

That last line is important — a follower tracks **three different positions** (received / flushed to disk / actually replayed), and the gap between them is where lag hides.

**Statement-based vs row-based (WAL/logical) replication.** MySQL historically shipped the *SQL statement* itself. That breaks horribly with nondeterminism:

```sql
-- Statement-based replication: DISASTER
UPDATE sessions SET last_seen = NOW() WHERE user_id = 42;
-- The leader runs it at 10:00:00.000
-- The follower replays it at 10:00:00.900 → different value! Replicas diverge.

INSERT INTO picks (item) VALUES ((SELECT item FROM pool ORDER BY RAND() LIMIT 1));
-- Different random row on every replica. Total divergence.
```

Row-based / WAL-based replication ships **the resulting rows or physical bytes**, not the statement, so it is deterministic. Use row-based (`binlog_format=ROW` in MySQL). This is a real interview follow-up.

### 2. Synchronous vs asynchronous vs semi-synchronous

The question is: **at what moment does the leader tell the client "committed"?**

**Asynchronous (the default almost everywhere):**
```
Client ──write──▶ Leader ──"OK, committed!"──▶ Client   (t = 2 ms)
                    │
                    └──(later, whenever)──▶ Follower
```
Fast. But if the leader's disk dies between the ACK and the follower receiving the log, **that write is gone forever.** The client was told "committed." It wasn't. This is the **data-loss window** — usually milliseconds, but during a bulk load or a network hiccup it can be *minutes* of writes.

**Synchronous:**
```
Client ──write──▶ Leader ──▶ Follower ──ack──▶ Leader ──"OK"──▶ Client  (t = 2ms + RTT)
```
Zero data loss for a committed write. But now **every write pays the network round-trip** to the follower — 1 ms in the same AZ, 80 ms cross-region. Worse: if the follower is slow or down, **writes block entirely.** You have made your availability *worse* by adding a machine. A fully synchronous system with N followers is only as available as its *slowest* follower.

**Semi-synchronous (the practical middle):** one follower is synchronous, the rest are async.

```
                       ┌── SYNC follower  (must ack → no data loss, safe to promote)
Leader ── writes ──────┤
                       ├── ASYNC follower (may lag)
                       └── ASYNC follower (cross-region, for DR)
```
If the sync follower dies, the leader **promotes an async follower to be the new sync one** and keeps going. You get durability of one guaranteed copy without hostage-taking by every replica.

| Mode | Write latency | Data loss on leader crash | Availability impact |
|---|---|---|---|
| Async | ~2 ms | Yes — last N ms/sec of writes | None (best) |
| Semi-sync | ~3–5 ms (same AZ) | No (one copy guaranteed) | Small — needs 1 healthy follower |
| Fully sync | ~2 ms + slowest RTT | No | Bad — any slow follower stalls all writes |
| Sync across regions | +80–150 ms | No | Terrible for write-heavy apps |

**Rule:** semi-synchronous within a region (durability), asynchronous across regions (DR without paying cross-ocean latency on every write).

### 3. Replication lag and the three problems it causes

Recall from [09 — CAP Theorem](./09-cap-theorem.md) that when you allow reads from stale replicas, you have chosen **availability over strong consistency**. Here is what that choice *feels like* to a real user.

**(a) Read-your-own-writes violation — "my comment vanished"**

```
t=0ms   Ravi posts a comment.  POST /comments → LEADER. Committed. 200 OK.
t=5ms   His browser redirects and re-fetches the thread.
        GET /comments → load balancer → FOLLOWER 2
t=5ms   Follower 2 is 300 ms behind. The comment isn't there yet.
        Ravi sees the thread WITHOUT his comment. He panics and posts again.
t=310ms Both comments appear. Now there's a duplicate.
```

This is the single most common replication bug in production. The user is not asking for global consistency — he just wants to see **his own** writes.

**Fixes:**
- **Read-from-leader-after-write:** after any write by user U, route U's reads to the leader for the next N seconds (e.g. 10s). Cheap and effective. Costs: some read traffic hits the leader.
- **Read your own profile from the leader:** anything the user can *edit* is read from the leader; everyone else's content from followers.
- **Version token (LSN pinning):** the leader returns the log position of your write; the client sends it back; the router picks a replica that has replayed at least that position (or waits). This is the precise fix. AWS Aurora and Google Spanner expose something like this.

**(b) Monotonic reads violation — "the comment appeared, then disappeared"**

Worse than stale — **going backwards in time.**

```
Read 1 → Follower A (lag 50 ms)   → sees Ravi's comment  ✓
Read 2 → Follower B (lag 2000 ms) → comment is GONE      ✗
```

The user refreshes and content *un-happens*. Utterly bewildering.

**Guarantee wanted:** monotonic reads — "if I read a value, I will never later read an *older* value." Weaker than strong consistency, stronger than eventual.

**Fix:** **sticky routing** — hash the user ID to one replica so a given user always reads from the same follower. `replica = replicas[hash(userId) % replicas.length]`. That user may see stale data, but never *rewinding* data. (Careful: when a replica dies, everyone hashed to it moves — and may jump *backwards*. See [74 — Consistent Hashing](./74-consistent-hashing.md).)

**(c) Cross-region staleness**

Your leader is in `us-east-1`. Your follower in `ap-south-1` is 200 ms of network away, and during peak write load it lags 5–30 seconds. An Indian user updates their address (write → us-east, 250 ms round trip) and immediately reloads (read → ap-south, stale). Same bug, but now the lag is *seconds*, not milliseconds, so the sticky-session window has to be much larger — and you're paying 250 ms on every write anyway.

**Fixes:** route the whole *session* (reads and writes) of a user to their home region; or accept that writes are slow and reads are local; or go multi-leader (next section) and inherit conflict hell.

### 4. Multi-leader replication (master-master) — and the conflict hell

**Two or more nodes accept writes**, and each replicates to the others.

```
        WRITES                          WRITES
          │                                │
          ▼                                ▼
    ┌───────────┐   bidirectional    ┌───────────┐
    │ LEADER US │◀══════════════════▶│ LEADER EU │
    └─────┬─────┘   replication      └─────┬─────┘
          │                                │
     followers                        followers
```

**When you actually want it:**
- **Multi-region writes.** Writes are served locally (5 ms, not 150 ms). Each region has a leader.
- **Offline clients.** Your phone's calendar app *is* a leader — you create an event on a plane, and it syncs when you land. Every offline-capable app is secretly multi-leader.
- **Collaborative editing.** Google Docs: each browser is a replica accepting local "writes."

**The conflict.** Two users edit the same row in two regions within the replication window:

```
t=0   US leader:  UPDATE items SET title='Blue Chair'   WHERE id=9;
t=0   EU leader:  UPDATE items SET title='Navy Chair'   WHERE id=9;
t=50  The two leaders exchange logs. Both see a conflicting change.
      Row 9 has TWO valid, committed, incompatible values.
      There is no correct answer. The database must pick one — or ask you to.
```

In single-leader this is *impossible* — the second writer would have blocked on a row lock and then overwritten deliberately. Multi-leader gives up that safety.

**Resolution strategies, from worst to best:**

| Strategy | How it works | The catch |
|---|---|---|
| **Last-write-wins (LWW)** | Attach a timestamp; highest wins | **Silently destroys data.** And clocks are not synchronized — an EU server 200 ms fast wins every conflict forever. Cassandra defaults to this. |
| **Higher-node-ID wins** | Deterministic tiebreak | Also silently destroys data, just more arbitrarily |
| **Merge / concatenate** | Keep both: `"Blue Chair / Navy Chair"` | Nonsense values, but no lost data |
| **Store both, ask the user** | Keep siblings, surface a merge UI | Correct, but you must build the UI (this is what Git does) |
| **CRDTs** (Conflict-free Replicated Data Types) | Data types with a *mathematically* commutative merge — sets, counters, sequences | Beautiful for counters/sets/text; you must model your data as a CRDT. See [76 — Eventual Consistency](./76-eventual-consistency.md). |
| **Operational Transform** | Transform concurrent ops against each other (Google Docs) | Hard to get right, but the right answer for text |

The honest summary: **multi-leader trades a hard availability problem for a hard correctness problem.** Do not adopt it for "extra redundancy." Adopt it only when you genuinely need local writes in multiple places, and only when you have a real conflict story for every table.

### 5. Leaderless replication (a preview)

Dynamo, Cassandra, Riak, ScyllaDB: **no leader at all.** The client (or a coordinator) writes to *several* nodes at once and reads from *several* at once. Consistency comes from a quorum:

```
N = 3 replicas
W = 2 (a write must be acked by 2 nodes)
R = 2 (a read must consult 2 nodes)

W + R > N  →  2 + 2 > 3  →  the read set and the write set must OVERLAP
              → at least one node in your read set has the newest value
```

Stale replicas get repaired by **read repair** (fix them when a read notices a mismatch) and **anti-entropy** (background Merkle-tree comparison). This is the topic of [86 — Data Replication Strategies](./86-data-replication-strategies.md) — just know the shape now.

### 6. Failover, split-brain, and fencing

**Failover** = the leader dies; a follower is promoted. Three steps:

1. **Detect.** No heartbeat for T seconds (typically 10–30s). Too short → you fail over on a GC pause. Too long → longer outage.
2. **Elect.** Pick the follower with the **most complete replication log** (highest LSN) — losing the least data. Consensus algorithms (Raft) or an external coordinator (ZooKeeper, etcd, Orchestrator) do this. See [83 — Leader Election](./83-leader-election.md).
3. **Reconfigure.** Point the app at the new leader, and tell the old leader (if it ever comes back) that it is now a follower.

**Split-brain — the nightmare.** The old leader wasn't actually dead. It was **network-partitioned** — unreachable, but alive and still accepting writes from clients on its side of the partition.

```
                    ✂ NETWORK PARTITION ✂
   ┌──────────────────────┐  │  ┌──────────────────────┐
   │  OLD LEADER (alive!) │  │  │  NEW LEADER (promoted)│
   │  still taking writes │  │  │  taking writes        │
   │  order #5001 → Alice │  │  │  order #5001 → Bob    │
   └──────────────────────┘  │  └──────────────────────┘
        clients on left      │      clients on right
                             │
   When the partition heals: TWO conflicting histories.
   Same primary keys, different data. Money double-spent.
```

Both halves believed they were the leader. Both accepted writes. When the network heals, there is no correct merge.

**Defences:**
- **Quorum-based election.** A node may only *be* leader if a majority (⌈N/2⌉+1) of nodes vote for it. The minority side of a partition can never gather a majority, so it can never elect a leader — and the old leader must **step down** when it loses contact with a majority. This is why you always run an **odd** number of nodes.
- **Fencing tokens.** Every leadership term gets a monotonically increasing number (term 7, term 8…). Storage/clients reject any write carrying an *older* token. Even if the zombie leader (term 7) keeps writing, the shared storage says "I've seen term 8, go away." Same idea as [84 — Distributed Locking](./84-distributed-locking.md).
- **STONITH** ("shoot the other node in the head") — literally power-cycle or firewall off the old node before promoting.

**And accept the cost:** with **async** replication, failover *always* loses the writes the old leader hadn't shipped yet. GitHub had a well-known 2018 incident where a network partition triggered a failover and the two sides diverged; reconciling took ~24 hours of degraded service. Failover is not free.

### 7. A shard-free read/write router in Node.js

The application-level piece is simple, and this is exactly what an interviewer means by "how does your app know where to read from?"

```javascript
// db/router.js
import pg from 'pg';

/**
 * Routes writes to the leader and reads to followers, while preserving
 * read-your-own-writes for a user who just wrote.
 */
export class ReplicaRouter {
  constructor({ leaderUrl, followerUrls, stickyWindowMs = 10_000 }) {
    this.leader = new pg.Pool({ connectionString: leaderUrl, max: 20 });
    this.followers = followerUrls.map((url) => new pg.Pool({ connectionString: url, max: 20 }));

    // userId -> timestamp until which this user MUST read from the leader.
    // In production this lives in Redis so it works across all app servers.
    this.recentWriters = new Map();
    this.stickyWindowMs = stickyWindowMs;
    this.rr = 0; // round-robin cursor
  }

  /** Every write goes to the single leader, and we remember the writer. */
  async write(userId, sql, params) {
    const result = await this.leader.query(sql, params);
    // Pin this user to the leader briefly — otherwise he refreshes and his
    // own comment is missing from a lagging follower.
    this.recentWriters.set(userId, Date.now() + this.stickyWindowMs);
    return result;
  }

  async read(userId, sql, params) {
    const pinnedUntil = this.recentWriters.get(userId) ?? 0;
    if (Date.now() < pinnedUntil) {
      return this.leader.query(sql, params); // read-your-own-writes
    }
    this.recentWriters.delete(userId);
    return this.pickFollower(userId).query(sql, params);
  }

  /**
   * Sticky by userId, NOT round-robin. Round-robin would let a user bounce
   * between a fresh and a lagging replica → monotonic reads violation
   * ("my comment appeared, then disappeared").
   */
  pickFollower(userId) {
    const healthy = this.followers.filter((f) => !f.__unhealthy);
    if (healthy.length === 0) return this.leader; // degrade to leader, never 500
    const idx = hashString(String(userId)) % healthy.length;
    return healthy[idx];
  }
}

function hashString(s) {
  let h = 2166136261; // FNV-1a
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return h >>> 0;
}
```

**Measuring lag** — you cannot manage what you don't measure. Alert on this:

```sql
-- PostgreSQL, run ON THE LEADER: how far behind is each follower, in bytes?
SELECT client_addr,
       state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)   AS sent_lag_bytes,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
       replay_lag                                         AS replay_lag_time
FROM pg_stat_replication;

-- PostgreSQL, run ON A FOLLOWER: how stale am I, in seconds?
SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

```javascript
// Eject a follower from the read pool when it falls too far behind.
// A 30-second-stale replica serving reads is worse than no replica.
const MAX_LAG_SECONDS = 5;

setInterval(async () => {
  for (const follower of router.followers) {
    try {
      const { rows } = await follower.query(
        `SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag`
      );
      follower.__unhealthy = Number(rows[0].lag) > MAX_LAG_SECONDS;
    } catch {
      follower.__unhealthy = true; // unreachable = unhealthy
    }
  }
}, 2000);
```

---

## Visual / Diagram description

### Diagram 1 — Single-leader topology with the request path

```
                        ┌────────────────────┐
      WRITE (POST)      │  Node.js API tier   │      READ (GET)
   ──────────────────▶  │  ReplicaRouter      │  ◀──────────────────
                        └───┬────────────┬────┘
                            │            │
              writes ONLY   │            │  reads (sticky by userId)
                            ▼            │
                    ┌───────────────┐    │
                    │    LEADER     │    │
                    │  (writable)   │    │
                    │               │    │
                    │  WAL / binlog │    │
                    └───┬───┬───┬───┘    │
          walsender     │   │   │        │
        ┌───────────────┘   │   └────────────────────┐
        │  (semi-sync)      │ (async)                │ (async, cross-region)
        ▼                   ▼                        ▼
 ┌─────────────┐     ┌─────────────┐          ┌─────────────┐
 │ FOLLOWER 1  │     │ FOLLOWER 2  │          │ FOLLOWER 3  │
 │  same AZ    │     │  same region│          │ eu-west-1   │
 │  lag ~5 ms  │     │  lag ~50 ms │          │ lag ~2 s    │
 │  READ-ONLY  │     │  READ-ONLY  │          │  READ-ONLY  │
 └──────┬──────┘     └──────┬──────┘          └──────┬──────┘
        │                   │                        │
        └───────────────────┴────── READS ───────────┘
             (a lag monitor ejects any follower > 5s stale)
```

**What it shows.** One writable node. Three read-only copies fed by the same WAL stream. Follower 1 is semi-synchronous (the leader waits for its ack before saying "committed"), so it is the safe promotion candidate. Follower 3 lives in another region — it exists for **disaster recovery**, not for latency, and it lags. The router sends every write left, and reads to a *consistently chosen* follower per user.

### Diagram 2 — The failover timeline, and where split-brain enters

```
 TIME    LEADER (us-east-1a)          FOLLOWER-1 (1a)         COORDINATOR (etcd, 3 nodes)
 ─────────────────────────────────────────────────────────────────────────────────────
 t=0     healthy, LSN 9000            LSN 9000                 leader lease held by L
 t=1s    ██ NETWORK PARTITION ██      ────────────────────────────────────────────────
 t=1s    still alive!                 no heartbeat from L      no heartbeat from L
         still accepting writes  ✗
         LSN 9001, 9002...            LSN 9000 (frozen)
 t=15s   ...LSN 9040                  election starts          2 of 3 nodes reachable
                                                               → QUORUM on this side
 t=16s   ✗ SPLIT-BRAIN RISK ✗         PROMOTED. term = 8.      grants lease, term 8
         L thinks it is leader        now accepts writes
         F1 knows it is leader        LSN 9001' (different!)
 ─────────────────────────────────────────────────────────────────────────────────────
 DEFENCE:  L cannot reach a MAJORITY of the coordinator → it must STEP DOWN at t=15s.
           Shared storage / clients reject any write stamped term=7 once term=8 exists.
           (fencing token)
 ─────────────────────────────────────────────────────────────────────────────────────
 t=60s   partition heals.
         L rejoins as a FOLLOWER, rewinds to LSN 9000, replays from the new leader.
         Writes 9001–9040 that L accepted during the partition are LOST.
         → That is the async data-loss window, made visible.
```

**What it shows.** Failover is not one event but four: detect → elect → fence → reconfigure. The dangerous gap is between "old leader is unreachable" and "old leader knows it is not the leader." Quorum plus fencing tokens closes that gap; without them, both nodes write and you get two irreconcilable histories.

---

## Real world examples

### 1. GitHub — MySQL primary with `orchestrator` for failover

GitHub runs MySQL in a single-writer, many-reader configuration and uses the open-source `orchestrator` tool to detect primary failure and promote a replica. Their public post-mortem of the **October 2018 incident** is the best free education on split-brain you can get: a 43-second network partition between coasts caused an automated failover; the old primary in the other datacenter had accepted writes that the newly-promoted primary never had. Because the two sides had *diverged*, GitHub could not simply fast-forward — reconciling took over 24 hours of degraded service. The lesson they themselves drew: **cross-datacenter automatic failover with async replication trades a short outage for a long consistency problem.**

### 2. Instagram — read replicas plus a "read your own writes" carve-out

Instagram's early architecture (well documented in their engineering posts) was PostgreSQL with a primary and multiple read replicas, later sharded. The classic pattern they and nearly every large Rails/Django/Node shop converge on: reads go to replicas by default, but any request in a **short window after that user wrote** is routed to the primary, and a user's *own* profile/settings are always read from the primary. This is exactly the `ReplicaRouter` above.

### 3. Amazon Aurora — replication moved into the storage layer

Aurora is worth knowing because it *changes the shape of the problem*. Instead of shipping the WAL to full follower databases that replay it, Aurora writes the log to a **distributed storage layer replicated 6 ways across 3 availability zones** (quorum: 4/6 for writes, 3/6 for reads). Replicas share that same storage — they don't replay a log into their own copy of the data. Result: replica lag is typically **tens of milliseconds** rather than seconds, and promoting a replica is fast because it doesn't need to catch up on data it already shares. The trade-off is that you're locked into a proprietary storage engine.

### 4. CouchDB / mobile sync — multi-leader by necessity

Offline-first apps (CouchDB↔PouchDB, Apple's CloudKit, Firebase) are unavoidably multi-leader: the phone accepts writes with no network. They handle it exactly as this chapter predicts — CouchDB stores **conflicting revisions as siblings** and forces the application to pick a winner, rather than silently doing last-write-wins.

---

## Trade-offs

| Topology | Write scaling | Read scaling | Conflict risk | Operational complexity | Use when |
|---|---|---|---|---|---|
| **Single-leader** | None (one node) | Excellent | **Zero** | Low | Almost always. Default choice. |
| **Multi-leader** | Yes (per region) | Excellent | **High** | High | Multi-region writes, offline clients, collaborative editing |
| **Leaderless (quorum)** | Yes | Yes | Medium (tunable) | High | Always-on, write-heavy, tolerant of eventual consistency |

| Sync mode | You gain | You give up |
|---|---|---|
| Asynchronous | Lowest write latency; leader unaffected by slow/dead followers | Committed writes can be lost on failover |
| Semi-synchronous | Guaranteed durable second copy; a safe promotion target | A few ms per write; needs ≥1 healthy sync follower |
| Fully synchronous | Zero data loss across all replicas | Availability collapses to the slowest replica; unusable cross-region |

| Lag fix | You gain | You give up |
|---|---|---|
| Read-from-leader after write | Read-your-own-writes | Some read load lands back on the leader |
| Sticky replica per user | Monotonic reads | Uneven replica load; rewind risk on replica failure |
| Version token / LSN pinning | Precise, minimal leader load | Client and router must carry the token; some reads wait |
| Just accept staleness | Simplest, cheapest | Confusing user experience — only OK for genuinely stale-safe data (analytics, feeds, "likes" counts) |

**Rule of thumb:** Start single-leader, semi-synchronous within the region, asynchronous to one cross-region replica for DR. Route reads to followers with per-user stickiness and a 10-second read-from-leader window after any write. Alert when replica lag exceeds 5 seconds and eject the replica from the read pool. Only go multi-leader when you can name the exact conflict-resolution rule for every table — and if you can't, you don't need multi-leader.

---

## Common interview questions on this topic

### Q1: "Your read replicas are lagging 30 seconds. Users say their posts disappear after refreshing. Diagnose and fix."

**Hint:** That's a read-your-own-writes violation. *Diagnose:* check `pg_stat_replication` — is the gap in `sent_lsn` (network/leader bottleneck) or `replay_lsn` (the follower can't apply fast enough, often a long-running read query blocking WAL replay, or a bulk write on the leader)? *Fix short-term:* pin a user's reads to the leader for ~10s after any write, and eject replicas lagging > 5s from the read pool. *Fix long-term:* find the source of lag — a batch job, a missing index causing a slow replay, or an under-provisioned replica.

### Q2: "What is split-brain and how do you prevent it?"

**Hint:** A network partition leaves the old leader alive and still accepting writes while a new leader is promoted on the other side → two divergent write histories that cannot be merged. Prevent it with (a) **quorum-based election** — a leader must hold a majority, and must *step down* when it loses the majority, so the minority side can never elect or continue as leader (run an odd number of nodes); and (b) **fencing tokens** — a monotonically increasing term number, with storage rejecting writes from an older term. Say STONITH as a bonus.

### Q3: "Synchronous or asynchronous replication — which would you pick, and why?"

**Hint:** Refuse the false binary. **Semi-synchronous**: one synchronous follower (guarantees a durable second copy and a safe promotion candidate) plus asynchronous followers (read scaling and cross-region DR, without paying their latency on every write). Then say the number: fully synchronous across regions adds 80–150 ms to *every write* and makes your availability equal to your worst replica.

### Q4: "Does replication help with write scaling?"

**Hint:** **No.** In single-leader, every write still goes through one machine — replication multiplies *reads*, not writes. When writes are the bottleneck, you need **sharding** ([64 — Database Sharding](./64-database-sharding.md)), which partitions the write load. Multi-leader does split writes, but you pay in conflicts. This is a favourite trick question.

### Q5: "You have a multi-region app. Two users edit the same product listing at the same time in the US and EU. What happens?"

**Hint:** Name the topology first. If single-leader, one write blocks on a row lock and then overwrites — no conflict, but the EU user paid 150 ms. If multi-leader, you have a genuine conflict and must choose a resolution: LWW (fast, silently loses data, and depends on clock sync you don't have), store both siblings and ask the user (correct, needs UI), or a CRDT (correct and automatic, but you must model the field as a CRDT — a set or a counter, not an opaque string).

---

## Practice exercise

### Design and instrument the read path for "Threadly"

Threadly is a comment app: 50,000 writes/day, 5,000,000 reads/day (a 100:1 read/write ratio), one region, one PostgreSQL leader that is at 80% CPU on reads.

**Produce these five things (~30 min):**

1. **A topology diagram** (whiteboard style, box characters). Decide how many followers, which one is semi-synchronous, and whether you want a cross-region or delayed replica. Justify each with one of the three motives: read scaling / HA / DR.

2. **The arithmetic.** If the leader currently does 5M reads/day, what is that in QPS at a 3× peak factor? (Show the steps: 5,000,000 / 86,400 = ~58 rps average → ×3 peak = ~174 rps.) How many followers do you need if each handles 150 rps comfortably, and how does that change your CPU picture on the leader?

3. **A written trace of the read-your-own-writes bug.** Write the exact timeline (t=0, t=5ms, …) for Ravi posting a comment and seeing it vanish, then write the timeline again with your fix applied.

4. **Extend the `ReplicaRouter`** from this doc to support **version-token pinning** instead of a time window: `write()` should return the leader's `pg_current_wal_lsn()`, the caller passes it back on the next `read()`, and `pickFollower()` should skip any follower whose `pg_last_wal_replay_lsn()` is behind that token (falling back to the leader if none qualify).

5. **Answer in three sentences:** "Our writes are now the bottleneck — should we add more replicas?" Explain why not, and what you'd do instead.

---

## Quick reference cheat sheet

- **Three motives, not one:** replication buys **read scaling**, **high availability**, and **disaster recovery** — they pull the design in different directions. Name all three.
- **Single-leader is the default.** One writable node ⇒ **zero write conflicts**. Reach for this 95% of the time.
- **Replication = shipping the WAL.** The leader already writes a write-ahead log for crash recovery; replication streams that same log to followers, which replay it in order.
- **Use row-based, not statement-based** replication — `NOW()` and `RAND()` make statement-based replicas diverge.
- **Replication does NOT scale writes.** Every write still hits one leader. Write bottleneck ⇒ **sharding**, not more replicas.
- **Async = data-loss window.** The leader ACKs before the follower has the write; if it dies, that write is gone. Usually ms; during bulk loads, minutes.
- **Semi-synchronous is the practical default:** one sync follower (durability + safe promotion target), the rest async (scale + DR).
- **Fully synchronous makes availability worse** — your writes are only as fast and as available as your slowest follower.
- **Read-your-own-writes violation:** user writes, refreshes, sees stale data. Fix = read from the leader for ~10s after that user's write, or pin an LSN token.
- **Monotonic reads violation:** data appears, then disappears (two followers with different lag). Fix = **sticky routing** — always send a given user to the same follower.
- **Eject lagging replicas.** A replica 30 seconds stale should be pulled from the read pool automatically. Alert on `pg_stat_replication`.
- **Failover = detect → elect (highest LSN) → fence → reconfigure.** All four steps, in that order.
- **Split-brain:** partition leaves the old leader alive and writing while a new one is promoted → divergent histories. Defence = **quorum** (majority, odd node count, step down when you lose it) + **fencing tokens** (monotonic term number, storage rejects old terms).
- **Multi-leader trades an availability problem for a correctness problem.** LWW silently destroys data and depends on clock sync you don't have. Prefer siblings or CRDTs.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [62 — Database Indexing](./62-database-indexing.md) — squeeze one machine first; indexing is the cheapest read win before you add replicas |
| **Next** | [64 — Database Sharding](./64-database-sharding.md) — when replication can't help because *writes* are the bottleneck |
| **Related** | [09 — CAP Theorem](./09-cap-theorem.md) — replication lag is CAP's "eventual consistency" showing up in your support inbox |
| **Related** | [83 — Leader Election](./83-leader-election.md) — how a new leader is actually chosen, and why quorum prevents split-brain |
| **Related** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — leaderless quorum replication (W + R > N), read repair, and anti-entropy |
