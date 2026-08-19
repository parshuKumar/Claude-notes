# 63 — High Availability and Failover
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

A shop with one till, and a spare till in the back room that mirrors every sale.

The main till breaks. Somebody must:

1. **Decide it is actually broken** — not just slow, not just a jammed receipt roll. Decide too fast and you swap tills for nothing; decide too slow and customers walk out.
2. **Make absolutely sure the old till cannot take another sale.** Unplug it. ★ **If both tills accept sales, you now have two contradictory ledgers and no way to merge them.**
3. **Tell every cashier which till to use now.**

★ **Step 2 is the whole discipline.** The dangerous failure is not "the till broke" — it's "the till *seemed* broken, we started using the spare, and then the original came back to life and kept taking money." Two tills, two truths, and reconciling them by hand.

And the number nobody asks for until afterwards: ★ **how many sales were rung up on the main till but not yet mirrored to the spare?** Those are simply gone. That is your **RPO**, and it is a design choice you make in advance, not a surprise you discover during an incident.

---

## Where this fits in the big picture

```
   58 read replicas · 62 replication internals — ★ the mechanism
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 63 HA & FAILOVER ← YOU ARE HERE              │
        │ ★ detection, promotion, fencing, timelines   │
        └────────────────────┬─────────────────────────┘
                             ▼
              64 backup & PITR (★ the thing HA is NOT)
              68 CAP — the formal framing
```

★ **HA protects against a machine dying. It does not protect against a bad `DELETE`, a bad migration, or corruption** — all of those replicate in milliseconds. That is Topic 64, and conflating the two is the most expensive mistake in this area.

---

## What is this?

Keeping the database writable when a node fails, by promoting a standby.

**The two numbers that define the entire design:**

```
 ★ RPO — RECOVERY POINT OBJECTIVE
   "how much committed data may we lose?"
   ⇒ async replication  ⇒ ★ RPO > 0 (whatever hadn't shipped)
   ⇒ sync replication   ⇒ ★ RPO = 0 (for the sync standby)

 ★ RTO — RECOVERY TIME OBJECTIVE
   "how long may we be unwritable?"
   ⇒ detection + decision + promotion + reconfiguration
   ⇒ ★ typical well-run automated failover: 15–45 seconds
   ⇒ ★ typical manual failover: 5–30 minutes

 ★ THE TRADE THAT CANNOT BE AVOIDED:
   RPO = 0 requires synchronous replication, which means
   ★ EVERY COMMIT WAITS FOR A NETWORK ROUND TRIP,
   and ★ if the sync standby is unreachable, commits BLOCK.
   ⇒ you are choosing between losing data and losing availability.
   ⇒ ★ THIS IS CAP (Topic 68), made concrete.
```

**The four phases of a failover, and where each goes wrong:**

| Phase | Question | Failure mode |
|---|---|---|
| **Detection** | is the primary really down? | ★ false positives; split brain |
| **Election** | which standby is promoted? | ★ promoting the one furthest behind |
| **Fencing** | is the old primary definitely stopped? | ★ **split brain** |
| **Reconfiguration** | do clients and replicas follow? | ★ writes to the old primary |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE APPLICATION IS PART OF THE FAILOVER, AND USUALLY
   THE UNPREPARED PART.

 ① ★ CONNECTIONS DO NOT NOTICE.
    a TCP connection to a dead host does not error — it HANGS
    until the OS TCP timeout, which defaults to ★ MANY MINUTES.
    ⇒ your 30-second failover becomes a 15-minute outage because
      the pool is full of connections to a machine that no longer
      exists.
    ⇒ ★ FIX: connect_timeout, tcp_user_timeout, keepalives.

 ② ★ A PROMOTED REPLICA IN THE READ POOL IS WORSE THAN A DEAD ONE.
    it accepts reads AND writes, diverges, and nothing errors.
    ⇒ ★ health checks MUST include pg_is_in_recovery() (Topic 58).

 ③ ★ IN-FLIGHT TRANSACTIONS ARE LOST, INCLUDING SOME THAT
    "SUCCEEDED".
    a COMMIT whose acknowledgement was lost in transit may or may
    not have committed.
    ⇒ ★ THE ONLY DEFENCE IS IDEMPOTENCY (Topic 52).

 ⇒ ★ A DATABASE TEAM CAN BUILD PERFECT FAILOVER AND STILL PRODUCE
   A 15-MINUTE OUTAGE IF THE APPLICATION ISN'T READY.
```

---

## The physical reality

### Detection — why this is the hard part

```
 ★ YOU CANNOT DISTINGUISH "THE PRIMARY IS DEAD" FROM "THE NETWORK
   BETWEEN ME AND THE PRIMARY IS BROKEN". This is not an
   engineering gap; it is a proven impossibility in asynchronous
   systems.

 ⇒ SO EVERY HA SYSTEM PICKS A HEURISTIC AND ACCEPTS FALSE
   POSITIVES:
   • N consecutive failed health checks
   • a lease/TTL in a consensus store that expires
   • ★ quorum agreement among observers

 ★ THE TWO ERRORS, AND THEY TRADE AGAINST EACH OTHER:
   ★ FALSE POSITIVE — promote while the primary is alive
     ⇒ ★ SPLIT BRAIN: two writable primaries, divergent data,
       and ★ NO AUTOMATIC WAY TO MERGE THEM.
   ★ FALSE NEGATIVE — don't promote when the primary is dead
     ⇒ a longer outage.

 ⇒ ★ THE ASYMMETRY THAT DECIDES THE DESIGN:
   an outage is recoverable. ★ SPLIT BRAIN IS NOT.
   ⇒ ★ ALWAYS BIAS TOWARD FALSE NEGATIVES.
     A 60-second outage beats two hours of manual reconciliation
     and permanent data loss.

 ★ AND THE ONLY REAL DEFENCE: QUORUM.
   ⇒ a single observer deciding is a single point of wrongness.
   ⇒ ★ 3 or 5 nodes in a consensus store (etcd, Consul, ZooKeeper).
     A minority partition ★ CANNOT promote, by construction.
```

### Fencing — the step people skip, and the one that matters

```
 ★ FENCING = GUARANTEEING THE OLD PRIMARY CANNOT ACCEPT WRITES.
   Not "asking it nicely to stop." GUARANTEEING.

 ★ THE MECHANISMS, WEAKEST TO STRONGEST:
 ① ★ NONE — "it was down, so it's fine"
    ⇒ ★ the classic split brain. The primary was overloaded, not
      dead; it recovers and keeps serving writes.
 ② SOFTWARE — the old primary detects it lost the leader lease
    and demotes itself.
    ⇒ ✓ works when the old primary is HEALTHY BUT PARTITIONED
    ⇒ ★ ✗ fails when it is FROZEN (a long GC pause, an I/O stall,
      a hypervisor stun) and then resumes, unaware.
 ③ ★ STONITH ("shoot the other node in the head")
    an out-of-band kill: IPMI power-off, a cloud API stop-instance,
    detaching the storage volume.
    ⇒ ★ THE ONLY MECHANISM THAT WORKS AGAINST A FROZEN NODE.
 ④ ★ STORAGE/NETWORK FENCING
    revoke the volume attachment, or the VIP, or the security group.
    ⇒ ★ the node may be alive but cannot reach clients or disk.

 ★ THE PATRONI/ETCD APPROACH — a lease, and it is elegant:
   the primary must RENEW its leader key in etcd every `loop_wait`.
   ⇒ if it cannot reach etcd, it ★ DEMOTES ITSELF within ttl seconds
   ⇒ a standby may only promote if it can WRITE the leader key,
     which requires an etcd QUORUM.
   ⇒ ★ THE INVARIANT: a minority partition can neither hold nor
     acquire the leader key. ★ SPLIT BRAIN IS STRUCTURALLY
     IMPOSSIBLE, provided the primary's self-demotion is reliable.
   ⇒ ★ WHICH IS WHY `ttl` MUST EXCEED any plausible freeze, and
     why STONITH is still recommended for the frozen-node case.
```

### Timelines — how PostgreSQL prevents divergence, and its limit

```
 ★ WHEN A STANDBY IS PROMOTED, IT INCREMENTS ITS TIMELINE ID.

   before:  timeline 1, WAL: 000000010000004A0000008C
   promote: timeline 2, WAL: 000000020000004A0000008C
                        ▲▲
                        ★ the timeline is in the filename

 ⇒ ★ WHY IT EXISTS: after promotion, the new primary writes NEW
   WAL at LSNs that may ALSO have been written by the old primary
   with different content. The timeline id keeps them distinct.
 ⇒ ★ A `.history` file records where the branch happened:
   $ cat 00000002.history
   1  4A/8C001220  no recovery target specified

 ★ WHAT THIS BUYS: you cannot accidentally replay the old
   primary's WAL onto the new one. PostgreSQL refuses.
 ★ WHAT IT DOES NOT BUY: it does not RECOVER the transactions that
   were on the old primary and never shipped. Those are gone.

 ★ REJOINING THE OLD PRIMARY — pg_rewind
   the old primary is on timeline 1 and has WAL the new primary
   never saw. It cannot simply follow.
   ⇒ pg_rewind finds the divergence point and REWINDS the old
     primary's data files to it, then it can stream from the new
     primary.
   ⇒ ★ REQUIREMENTS: wal_log_hints = on (or data checksums), and
     the old primary must be CLEANLY SHUT DOWN first.
   ⇒ ★ AND IT IS DESTRUCTIVE: the diverged transactions are
     discarded. ★ Take a backup of the old primary first if that
     data might matter.
```

### The RPO you actually have — measuring it honestly

```
 ★ WITH ASYNCHRONOUS REPLICATION, YOUR RPO IS NOT A CONFIGURATION
   VALUE. IT IS WHATEVER LAG HAPPENED TO EXIST AT THE MOMENT OF
   FAILURE.

   SELECT application_name,
          pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS rpo_bytes,
          replay_lag AS rpo_time
     FROM pg_stat_replication;

 ★ AND FROM TOPIC 58: LAG IS NOT NORMALLY DISTRIBUTED.
   p50 4 ms · p99 340 ms · ★ p999 41 s · max 24 min
 ⇒ ★ YOUR REAL RPO IS THE p999, BECAUSE A NIGHTLY BULK JOB AND A
   HARDWARE FAILURE ARE INDEPENDENT EVENTS THAT WILL EVENTUALLY
   COINCIDE.

 ★ SYNCHRONOUS REPLICATION MAKES RPO = 0 — AT A PRICE:
   synchronous_commit = on, synchronous_standby_names = 'ANY 1 (a,b)'
   ⇒ commit latency 0.4 ms → 2.8 ms same-AZ → ★ 14 ms cross-AZ
   ⇒ ★ AND: with a SINGLE named standby, if it is down,
     ★ EVERY COMMIT ON THE PRIMARY BLOCKS INDEFINITELY.
     Your HA setup has made you LESS available.
   ⇒ ★ ALWAYS 'ANY 1 (a, b, c)' — a quorum, never a single name.

 ★ THE HYBRID MOST SYSTEMS SHOULD USE:
   • sync to a same-AZ standby (RPO=0, ~2.8 ms cost)
   • async to a cross-region standby (DR, RPO minutes)
   ⇒ ★ you get zero data loss for machine failure and a survivable
     copy for regional failure, without paying 14 ms per commit.
```

### `maximum_lag_on_failover` — the setting that caused a 14-minute outage

```
 ★ FROM TOPIC 42'S INCIDENT. Patroni's setting:
   maximum_lag_on_failover: 1048576      # 1 MB

 ⇒ MEANING: "do not promote a standby that is more than 1 MB
   behind."
 ⇒ ★ INTENT: bound the data loss.
 ⇒ ★ EFFECT IN PRACTICE: on a busy primary generating 88 GB/hr of
   WAL, standbys are routinely more than 1 MB behind — 1 MB is
   ★ 40 MILLISECONDS of WAL.
 ⇒ ★ RESULT: BOTH standbys were refused, no promotion occurred,
   and the cluster stayed down for 14 minutes while an engineer
   was paged.

 ★ THE LESSON, GENERALISED:
   an RPO bound expressed in BYTES on a cluster whose WAL rate you
   have not measured is ★ AN AVAILABILITY BOMB.
 ⇒ ★ SET IT FROM MEASUREMENT:
   acceptable_loss_seconds × observed_WAL_bytes_per_second
   ⇒ 5 s × 24 MB/s = ★ 120 MB, not 1 MB.
 ⇒ ★ AND ALERT when observed lag approaches it, so you learn
   before an incident does.
```

---

## How it works — step by step

### A Patroni cluster, concretely

```yaml
# patroni.yml — the settings that matter, annotated
scope: shop-cluster
namespace: /db/
name: pg-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.11:8008

etcd3:
  hosts: [10.0.1.21:2379, 10.0.1.22:2379, 10.0.1.23:2379]   # ★ 3 = quorum

bootstrap:
  dcs:
    # ★ THE THREE TIMING NUMBERS. They must satisfy:
    #   loop_wait + 2*retry_timeout <= ttl
    ttl: 30                # ★ leader lease; a frozen primary must
                           #   demote within this
    loop_wait: 10          # how often the loop runs
    retry_timeout: 10      # per-operation retry budget

    # ★ THE RPO BOUND — set from MEASURED WAL rate, not from a blog
    maximum_lag_on_failover: 134217728    # ★ 128 MB ≈ 5 s at 24 MB/s

    synchronous_mode: true          # ★ RPO = 0
    synchronous_mode_strict: false  # ★ CRITICAL — see below
    # ★ strict:true means "if no sync standby is available, BLOCK
    #   ALL WRITES." That is correct only if data loss is worse
    #   than downtime. For most systems it is NOT.

    postgresql:
      parameters:
        wal_level: replica
        max_wal_senders: 10
        max_replication_slots: 10
        max_slot_wal_keep_size: 128GB     # ★ Topic 62's guard
        wal_log_hints: on                 # ★ REQUIRED for pg_rewind
        synchronous_commit: 'on'
        archive_mode: 'on'                # ★ Topic 64 — HA ≠ backup
        archive_command: 'wal-g wal-push %p'

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.11:5432
  data_dir: /var/lib/postgresql/data
  use_pg_rewind: true       # ★ rejoin the old primary without a
                            #   full base backup
  pg_hba:
    - host replication repl 10.0.1.0/24 scram-sha-256
```

```bash
# ★ the three things to check on any Patroni cluster
patronictl -c patroni.yml list
```
```
+ Cluster: shop-cluster ---------+---------+---------+----+-----------+
| Member    | Host       | Role    | State   | TL | Lag in MB |
+-----------+------------+---------+---------+----+-----------+
| pg-node-1 | 10.0.1.11  | Leader  | running |  7 |           |
| pg-node-2 | 10.0.1.12  | Sync St | running |  7 |         0 |
| pg-node-3 | 10.0.1.13  | Replica | running |  7 |        12 |
+-----------+------------+---------+---------+----+-----------+
   ★ TL = timeline. All three agree ⇒ no split brain.
   ★ 'Sync St' = the synchronous standby. RPO = 0 against it.
```

### The application side — the part that's usually missing

```js
// ★ ① CONNECTION SETTINGS THAT MAKE FAILOVER FAST
const pool = new Pool({
  host: 'pgbouncer.internal',
  // ★ WITHOUT THESE, A DEAD HOST HANGS FOR MINUTES
  connectionTimeoutMillis: 3000,        // ★ give up connecting
  statement_timeout: 5000,              // ★ give up querying
  idle_in_transaction_session_timeout: 10000,
  // ★ libpq-level: detect a dead peer at the TCP layer
  keepAlives: 1,
  keepAlivesIdle: 5,
  keepAlivesInterval: 2,
  keepAlivesCount: 3,
  // ★ tcp_user_timeout is the single most important one:
  //   it bounds how long an ESTABLISHED connection waits for an
  //   ACK before erroring. Default is ~15 minutes.
  options: '-c tcp_user_timeout=6000',
  max: 20,
});
```

```js
// ★ ② MULTI-HOST CONNECTION STRINGS — libpq does the failover
const url = 'postgresql://user:pw@pg1:5432,pg2:5432,pg3:5432/shop'
          + '?target_session_attrs=read-write'
          + '&connect_timeout=3';
// ★ target_session_attrs=read-write makes libpq try each host and
//   KEEP ONLY the one that is not in recovery.
// ⇒ ★ this alone gives you client-side failover with no proxy.
```

```js
// ★ ③ RETRY ON THE ERRORS THAT MEAN "FAILOVER HAPPENED"
const FAILOVER_CODES = new Set([
  '57P01',  // admin_shutdown
  '57P02',  // crash_shutdown
  '57P03',  // cannot_connect_now  ★ standby still in recovery
  '08006',  // connection_failure
  '08001',  // sqlclient_unable_to_establish_connection
  '08004',  // rejected
  '25006',  // ★ read_only_sql_transaction — we hit a standby
]);

async function withFailoverRetry(fn, { attempts = 5 } = {}) {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (!FAILOVER_CODES.has(e.code) || i === attempts - 1) throw e;
      metrics.increment('db.failover_retry', { code: e.code });
      // ★ full jitter, and a longer ceiling than a normal retry —
      //   promotion takes tens of seconds
      await sleep(Math.min(8000, Math.random() * (500 * 2 ** i)));
    }
  }
}
// ★ AND: the operation MUST be idempotent (Topic 52). A commit
//   whose ack was lost may have succeeded.
```

```js
// ★ ④ HEALTH CHECKS THAT DETECT A PROMOTED REPLICA
async function checkReplica(pool) {
  const { rows: [r] } = await pool.query(`
    SELECT pg_is_in_recovery() AS in_recovery,
           pg_last_wal_replay_lsn() AS lsn,
           coalesce(extract(epoch from now()
             - pg_last_xact_replay_timestamp()), 0) AS lag_s`);
  // ★ THREE conditions, all necessary (Topic 58)
  return r.in_recovery === true && Number(r.lag_s) < 10;
  // ★ in_recovery = false ⇒ this node was PROMOTED ⇒ evict it from
  //   the READ pool immediately, or it will serve diverging data.
}
```

### Testing failover — the only way to know it works

```bash
# ★ RUN THESE IN STAGING MONTHLY, AND IN PRODUCTION QUARTERLY.
#   An untested failover is a hypothesis.

# ① PLANNED SWITCHOVER — the safe one. No data loss, no fencing
#    needed, because the primary participates.
patronictl -c patroni.yml switchover --master pg-node-1 \
                                     --candidate pg-node-2 --force
# ★ measure: time from command to first successful write

# ② UNPLANNED — kill the process
ssh pg-node-1 'sudo systemctl kill -s SIGKILL patroni'

# ③ ★ THE HARD ONE — freeze the node, don't kill it.
#    This is what a GC pause, an I/O stall or a hypervisor stun
#    looks like, and it is where fencing is actually tested.
ssh pg-node-1 'sudo kill -STOP $(pgrep -f "postgres: .*writer")'
sleep 60
ssh pg-node-1 'sudo kill -CONT $(pgrep -f "postgres: .*writer")'
# ★ THE QUESTION: did the old primary demote itself before
#   resuming? Check for any writes it accepted after the promotion.

# ④ ★ NETWORK PARTITION — isolate the primary from etcd only
ssh pg-node-1 'sudo iptables -A OUTPUT -d 10.0.1.21 -j DROP;
               sudo iptables -A OUTPUT -d 10.0.1.22 -j DROP;
               sudo iptables -A OUTPUT -d 10.0.1.23 -j DROP'
# ★ the primary can still reach clients but not the DCS.
#   It MUST demote itself within ttl seconds.
```

```sql
-- ★ AFTER EVERY TEST: prove there was no split brain
SELECT timeline_id FROM pg_control_checkpoint();
-- ★ run on every node. Divergent timelines with overlapping LSNs
--   mean data was written to two nodes.
```

---

## Concept breakdown

```
★ THE TWO NUMBERS
   RPO — how much data may be lost   ⇒ async > 0, sync = 0
   RTO — how long unwritable         ⇒ detect + decide + promote
                                        + reconfigure
   ★ RPO=0 requires sync, which means every commit waits and an
     unreachable standby can BLOCK writes. ⇒ ★ this is CAP (68).

★ THE FOUR PHASES
├── DETECTION  ★ you CANNOT distinguish "dead" from "partitioned"
│    ⇒ every system accepts false positives
│    ⇒ ★ BIAS TOWARD FALSE NEGATIVES: an outage is recoverable,
│      ★ SPLIT BRAIN IS NOT
│    ⇒ ★ QUORUM (3 or 5 observers) is the only real defence
├── ELECTION   pick the most-caught-up standby
│    ⇒ ★ maximum_lag_on_failover bounds RPO — and set wrong,
│      ★ REFUSES TO PROMOTE ANYTHING (Topic 42's 14-min outage)
├── ★ FENCING  GUARANTEE the old primary cannot write
│    none ✗ · software (lease self-demotion) · ★ STONITH ·
│    storage/network revocation
│    ⇒ ★ software fencing fails against a FROZEN node that resumes
└── RECONFIG   clients and replicas must follow the new primary

★ TIMELINES
   promotion increments the timeline id; a .history file records
   the branch point
   ⇒ ★ prevents accidentally replaying old WAL onto the new primary
   ⇒ ★ does NOT recover unshipped transactions — those are gone
   ⇒ pg_rewind rejoins the old primary (needs wal_log_hints, a
     clean shutdown, and ★ it DISCARDS diverged data)

★ YOUR REAL RPO IS THE p999 OF LAG, NOT THE AVERAGE
   p50 4 ms · ★ p999 41 s · max 24 min (Topic 58)
   ⇒ a nightly bulk job and a hardware failure WILL eventually
     coincide.

★ THE APPLICATION IS PART OF THE FAILOVER
├── ★ tcp_user_timeout / keepalives — a dead host HANGS for
│    ~15 min by default
├── ★ target_session_attrs=read-write with a multi-host URL
├── ★ retry on 57P01/57P02/57P03/08006/25006 — with idempotency (52)
└── ★ health checks MUST include pg_is_in_recovery()

★ TEST IT — four scenarios, and the third is the real one
   switchover · SIGKILL · ★ SIGSTOP-then-resume (a frozen node) ·
   ★ partition from the DCS only
```

---

## Diagrams

**Diagram 1 — big picture: the four phases and their failure modes**

```
   PRIMARY STOPS RESPONDING
            │
            ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ ① DETECTION                                                    │
 │    ★ "dead" and "partitioned" are INDISTINGUISHABLE            │
 │    ⇒ heuristic: N failed checks / an expired lease             │
 │    ⇒ ★ QUORUM of 3–5 observers, or a single point of wrongness │
 │    ✗ FALSE POSITIVE ⇒ ★ SPLIT BRAIN (unrecoverable)            │
 │    ✗ FALSE NEGATIVE ⇒ a longer outage (recoverable)            │
 │    ⇒ ★ ALWAYS BIAS TOWARD THE RECOVERABLE ERROR                │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ ② ELECTION — which standby?                                    │
 │    most-advanced replay_lsn, subject to                        │
 │    ★ maximum_lag_on_failover                                   │
 │    ✗ ★ SET TOO LOW ⇒ NOTHING IS ELIGIBLE ⇒ NO PROMOTION        │
 │      (Topic 42: 1 MB on an 88 GB/hr cluster = 14-min outage)   │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ ③ ★ FENCING — the step that prevents split brain               │
 │    software: the old primary loses its lease and self-demotes  │
 │      ✓ works when HEALTHY BUT PARTITIONED                      │
 │      ★ ✗ FAILS when FROZEN and later resuming                  │
 │    ★ STONITH: power off / stop the instance / detach the disk  │
 │      ✓ ★ the only mechanism that works against a frozen node   │
 └────────────────────────────┬───────────────────────────────────┘
                              ▼
 ┌────────────────────────────────────────────────────────────────┐
 │ ④ RECONFIGURATION                                              │
 │    promote ⇒ ★ timeline 7 → 8                                  │
 │    other replicas re-point at the new primary                  │
 │    ★ CLIENTS: a dead TCP connection HANGS for ~15 min without  │
 │      tcp_user_timeout — this is where "30-second failover"     │
 │      becomes a 15-minute outage                                │
 └────────────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: split brain, and how quorum prevents it**

```
 ✗ NO QUORUM — a single observer decides
 ┌───────────────────────────────────────────────────────────────┐
 │   ┌─────────┐   ✗ network partition   ┌─────────┐             │
 │   │PRIMARY  │◄─────────╳─────────────►│STANDBY  │             │
 │   │ alive!  │                          │+observer│             │
 │   │ serving │                          │ "primary│             │
 │   │ writes  │                          │  is down│             │
 │   └────┬────┘                          │  PROMOTE│             │
 │        │                               └────┬────┘             │
 │   app-region-A                          app-region-B           │
 │   writes ─────► order #8842                writes ─────► #8842 │
 │                                                                │
 │   ⇒ ★ TWO PRIMARIES. TWO ORDERS WITH THE SAME ID.             │
 │   ⇒ ★ TWO TIMELINES BRANCHING FROM THE SAME LSN.              │
 │   ⇒ ★ NO AUTOMATIC MERGE EXISTS. One side's writes must be    │
 │     manually extracted and replayed, or discarded.            │
 └───────────────────────────────────────────────────────────────┘

 ✓ QUORUM — 3-node etcd, leader lease
 ┌───────────────────────────────────────────────────────────────┐
 │                    etcd: e1  e2  e3                            │
 │                          ╳   │   │      ← partition            │
 │   ┌─────────┐            │   │   │      ┌─────────┐           │
 │   │PRIMARY  │────────────┘   │   │      │STANDBY  │           │
 │   │ can see │  ★ 1 of 3      │   └──────│ can see │           │
 │   │ e1 only │  ⇒ NO QUORUM   └──────────│ e2, e3  │           │
 │   │         │  ⇒ ★ CANNOT RENEW LEASE   │ ★ 2 of 3│           │
 │   │ ★ DEMOTES ITSELF within ttl (30 s)  │ ⇒ QUORUM│           │
 │   │ ⇒ read-only                          │ ⇒ ★ MAY │           │
 │   └─────────┘                            │ PROMOTE │           │
 │                                          └─────────┘           │
 │   ⇒ ★ THE MINORITY SIDE CANNOT WRITE, BY CONSTRUCTION.        │
 │   ⇒ ★ SPLIT BRAIN IS IMPOSSIBLE — provided self-demotion      │
 │     completes before the node resumes serving. Hence STONITH  │
 │     for the frozen case.                                       │
 └───────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: the application's contribution to RTO**

```
 ✗ DEFAULT CLIENT SETTINGS
 ┌───────────────────────────────────────────────────────────────┐
 │ t=0      the primary's host dies (hard power loss)             │
 │ t=0–30   ★ EVERY POOLED CONNECTION HANGS.                      │
 │          A TCP connection to a dead host does not RST — it     │
 │          retransmits. Default tcp_user_timeout ≈ ★ 15 minutes. │
 │ t=32     Patroni promotes pg-node-2. ★ THE DATABASE IS UP.     │
 │ t=32–900 ★ the application is still hanging on 20 dead sockets.│
 │          New requests queue behind them. Health checks pass    │
 │          (they use a different connection). Nobody knows why.  │
 │ t=900    the OS finally errors the sockets; the pool reconnects│
 │                                                                │
 │ ★ DATABASE RTO: 32 s.   ★ USER-VISIBLE OUTAGE: 15 MINUTES.    │
 └───────────────────────────────────────────────────────────────┘

 ✓ WITH TIMEOUTS AND MULTI-HOST
 ┌───────────────────────────────────────────────────────────────┐
 │ postgresql://u:p@pg1,pg2,pg3/shop                              │
 │   ?target_session_attrs=read-write&connect_timeout=3           │
 │ options='-c tcp_user_timeout=6000'                             │
 │ keepalives=1 keepalives_idle=5 keepalives_interval=2           │
 │                                                                │
 │ t=0      the host dies                                         │
 │ t=6      ★ sockets error (tcp_user_timeout)                    │
 │ t=6–32   ★ retries on 08006 with jittered backoff              │
 │          libpq tries pg1 (dead), pg2 (still standby ⇒ rejected │
 │          by target_session_attrs), pg3 …                       │
 │ t=32     pg-node-2 is promoted                                 │
 │ t=33     ★ the next retry finds a read-write node. Connected.  │
 │                                                                │
 │ ★ USER-VISIBLE OUTAGE: 33 SECONDS — matching the database RTO. │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Promote a standby by hand and watch the timeline change.**
```bash
psql -h replica -tAc "SELECT pg_is_in_recovery()"
```
```
 ★ t
```
```bash
psql -h replica -tAc "SELECT timeline_id FROM pg_control_checkpoint()"
```
```
 ★ 1
```
```bash
pg_ctl promote -D "$REPLICA_PGDATA"
sleep 2
psql -h replica -tAc "SELECT pg_is_in_recovery()"
psql -h replica -tAc "SELECT timeline_id FROM pg_control_checkpoint()"
```
```
 ★ f
 ★ 2        — a new timeline. It is now a primary.
```
```bash
ls "$REPLICA_PGDATA/pg_wal/" | grep history
cat "$REPLICA_PGDATA/pg_wal/00000002.history"
```
```
 00000002.history
 1  4A/8C001220  no recovery target specified
   ★ the branch point is recorded permanently.
```

**Prove a promoted node accepts writes.**
```bash
psql -h replica -c "INSERT INTO orders (customer_id, total_minor) VALUES (1,100)"
```
```
 INSERT 0 1
   ★ IF THIS NODE IS STILL IN YOUR READ POOL, IT IS NOW DIVERGING
     AND NOTHING WILL TELL YOU. (Topic 58.)
```

**Prove a dead host hangs without timeouts.**
```bash
# simulate a hard failure — drop packets rather than closing the socket
sudo iptables -A INPUT -s "$PRIMARY_IP" -j DROP

time psql "host=$PRIMARY_IP dbname=shop" -c "SELECT 1"
```
```
 ★ real  14m52s        — the default TCP retransmit budget
```
```bash
sudo iptables -D INPUT -s "$PRIMARY_IP" -j DROP

time psql "host=$PRIMARY_IP dbname=shop connect_timeout=3
           options='-c tcp_user_timeout=6000'" -c "SELECT 1"
```
```
 ★ real  0m6.04s        — 148× faster to fail
```

**Multi-host with `target_session_attrs`.**
```bash
psql "postgresql://repl@pg1:5432,pg2:5432,pg3:5432/shop\
?target_session_attrs=read-write&connect_timeout=3" \
  -tAc "SELECT inet_server_addr(), pg_is_in_recovery()"
```
```
 10.0.1.12 | f
   ★ libpq tried pg1 (dead), rejected pg2 if it was still a
     standby, and connected to the read-write node.
```

**Measure the RPO you actually have.**
```sql
SELECT application_name, sync_state,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn))
         AS unreplayed,
       replay_lag
  FROM pg_stat_replication;
```
```
 application_name | sync_state | unreplayed |  replay_lag
------------------+------------+------------+---------------
 pg-node-2        | ★ sync     | 0 bytes    | 00:00:00.001
 pg-node-3        | async      | ★ 4,182 kB | 00:00:00.184
   ★ if the primary died right now: RPO = 0 against node-2,
     RPO = 4 MB against node-3.
```

**Prove synchronous replication blocks when the standby is gone.**
```sql
ALTER SYSTEM SET synchronous_standby_names = 'pg-node-2';   -- ★ a SINGLE name
SELECT pg_reload_conf();
```
```bash
ssh pg-node-2 'sudo systemctl stop postgresql'
psql -h primary -c "INSERT INTO orders (customer_id,total_minor) VALUES (1,1)"
```
```
 ★ (hangs indefinitely)
```
```sql
-- from another session on the primary
SELECT pid, wait_event_type, wait_event, state FROM pg_stat_activity
 WHERE wait_event = 'SyncRep';
```
```
  pid  | wait_event_type | wait_event | state
-------+-----------------+------------+--------
 41202 | IPC             | ★ SyncRep  | active
   ★ THE PRIMARY IS UP AND ACCEPTING NOTHING.
     Your HA setup made you LESS available.
```
```sql
-- ★ the fix
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (pg-node-2, pg-node-3)';
SELECT pg_reload_conf();
-- ⇒ node-3 satisfies the quorum; writes resume immediately.
```

**Rejoin the old primary with `pg_rewind`.**
```bash
# ★ REQUIREMENT, set BEFORE you need it
psql -h old-primary -c "SHOW wal_log_hints"
```
```
 ★ on
```
```bash
# ★ must be cleanly shut down first
pg_ctl stop -D "$OLD_PRIMARY_PGDATA" -m fast

# ★ take a copy first — pg_rewind DISCARDS diverged transactions
cp -a "$OLD_PRIMARY_PGDATA" /backup/pre-rewind-$(date +%s)

pg_rewind --target-pgdata="$OLD_PRIMARY_PGDATA" \
          --source-server="host=new-primary user=repl dbname=postgres" \
          --progress
```
```
 servers diverged at WAL location 4A/8C001220 on timeline 1
 rewinding from last common checkpoint at 4A/8B000060 on timeline 1
 ★ Done!
```
```bash
# ★ then configure it as a standby of the new primary
touch "$OLD_PRIMARY_PGDATA/standby.signal"
echo "primary_conninfo = 'host=new-primary user=repl'" \
  >> "$OLD_PRIMARY_PGDATA/postgresql.auto.conf"
pg_ctl start -D "$OLD_PRIMARY_PGDATA"
psql -h old-primary -tAc "SELECT pg_is_in_recovery(), timeline_id
                            FROM pg_control_checkpoint()"
```
```
 t | ★ 2        — it now follows the new timeline
```

**Detect split brain.**
```bash
for h in pg1 pg2 pg3; do
  echo -n "$h: "
  psql -h $h -tAc "SELECT pg_is_in_recovery()::text || ' tl=' ||
                          (SELECT timeline_id FROM pg_control_checkpoint())"
done
```
```
 pg1: ★ f tl=1
 pg2: ★ f tl=2
 pg3: t tl=2
   ★ TWO NODES REPORT NOT-IN-RECOVERY ON DIFFERENT TIMELINES.
     THIS IS SPLIT BRAIN. Stop writes immediately.
```

---

## Example 2 — production scenario

**The situation.** A payments platform. Three-node Patroni cluster, etcd quorum, synchronous replication. The team is confident: *"we tested failover, it takes 20 seconds."*

```
 THEN, AT 03:14, THE PRIMARY'S HYPERVISOR STUNS THE VM FOR 90
 SECONDS (a storage migration on the host).

 WHAT ACTUALLY HAPPENED
   03:14:02  the VM freezes. ★ Not dead — FROZEN. No packets in
             or out. The process is not scheduled.
   03:14:32  Patroni on node-2 sees the leader key expire (ttl=30)
   03:14:33  node-2 acquires the leader key (etcd quorum: 2 of 3)
   03:14:35  node-2 promotes. ★ Timeline 7 → 8.
   03:15:32  ★ THE VM RESUMES. From its perspective, no time has
             passed. It is still the primary. It has 41 in-flight
             transactions.
   03:15:32  ★ IT ACCEPTS AND COMMITS 3 MORE TRANSACTIONS before
             its own Patroni loop runs.
   03:15:34  Patroni on node-1 discovers it lost the lease and
             demotes.
   ⇒ ★ 3 TRANSACTIONS COMMITTED ON A NODE THAT WAS NO LONGER THE
     PRIMARY, ON TIMELINE 7, AFTER TIMELINE 8 HAD BRANCHED.
   ⇒ ★ THEY ARE UNRECOVERABLE WITHOUT MANUAL EXTRACTION.
```

**Step 1 — establish exactly what was lost.**

```bash
# ★ the old primary's WAL past the divergence point
pg_waldump -p /var/lib/postgresql/data/pg_wal \
           -s 4A/8C001220 -t 7 | grep -E 'COMMIT' | tail
```
```
 rmgr: Transaction  desc: COMMIT 2026-08-20 03:15:32.884 IST
 rmgr: Transaction  desc: COMMIT 2026-08-20 03:15:33.104 IST
 rmgr: Transaction  desc: COMMIT 2026-08-20 03:15:33.402 IST
   ★ THREE COMMITS after the branch. Extract them before any
     pg_rewind, which would discard them.
```
```bash
# ★ start the old primary in an ISOLATED network, read-only,
#   to extract the rows
pg_ctl start -D /old -o "-p 5599 -c listen_addresses=localhost"
psql -p 5599 -c "SELECT * FROM payments WHERE created_at > '03:15:30'"
```
```
 ★ 3 payments, totalling ₹41,882. Reconciled manually against
   the gateway, then re-inserted on the new primary with their
   original idempotency keys (Topic 52) so nothing double-applied.
```

**Step 2 — why the design failed, precisely.**

```
 ★ THE FENCING WAS SOFTWARE-ONLY.
   Patroni's model: the primary self-demotes when it cannot renew
   the leader lease.
   ⇒ ★ THIS REQUIRES THE PRIMARY TO BE RUNNING. A frozen VM is not
     running. It cannot notice anything.
   ⇒ when it resumed, it had not yet run its loop, so it still
     believed it held the lease — and served writes.

 ★ THE WINDOW: from resume until the next Patroni loop iteration.
   loop_wait = 10 s ⇒ ★ up to 10 seconds of writes to a node that
   is no longer the primary.
   ⇒ ★ MEASURED: 2 seconds and 3 transactions. It could have been
     10 seconds and 400.
```

**Step 3 — the fixes, in order of importance.**

```yaml
# ★ ① STONITH — the only defence against a frozen node
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10

# ★ Patroni calls this before promoting a new leader.
# It must not return success unless the old node is DEFINITELY
# unable to write.
tags:
  nofailover: false

# in patroni.yml, the callback:
callbacks:
  on_role_change: /usr/local/bin/patroni-fence.sh
```
```bash
#!/usr/bin/env bash
# /usr/local/bin/patroni-fence.sh  — invoked on promotion
set -euo pipefail
ROLE="$1"; SCOPE="$2"
[ "$ROLE" = "master" ] || exit 0

OLD_LEADER=$(etcdctl get "/db/$SCOPE/leader" --print-value-only || true)
[ -n "$OLD_LEADER" ] || exit 0

# ★ HARD STOP THE OLD NODE. Not a graceful shutdown — a power cut.
aws ec2 stop-instances --instance-ids "$(node_to_instance "$OLD_LEADER")" --force
# wait for the state to be confirmed, or FAIL THE PROMOTION
for i in $(seq 1 30); do
  state=$(aws ec2 describe-instances --instance-ids … \
            --query 'Reservations[0].Instances[0].State.Name' --output text)
  [ "$state" = "stopped" ] && exit 0
  sleep 2
done
echo "★ FENCING FAILED — refusing to promote" >&2
exit 1
# ★ THE CRITICAL PROPERTY: if fencing cannot be confirmed,
#   PROMOTION MUST NOT PROCEED. An outage beats split brain.
```

```yaml
# ★ ② TIGHTEN THE WINDOW — but understand the trade
    ttl: 20            # ★ was 30 — faster detection
    loop_wait: 5       # ★ was 10 — halves the resume window
    retry_timeout: 5
    # ★ constraint: loop_wait + 2*retry_timeout <= ttl
    #   5 + 10 = 15 <= 20  ✓
    # ⇒ ★ LOWER ttl MEANS MORE FALSE POSITIVES. Do not go below
    #   what your network's p999 latency and your VM's plausible
    #   freeze duration justify.
```

```yaml
# ★ ③ SET maximum_lag_on_failover FROM MEASUREMENT
    # measured WAL rate: 24 MB/s peak
    # acceptable loss: 5 seconds
    maximum_lag_on_failover: 134217728   # ★ 128 MB
    # ★ NOT 1 MB — that is 40 ms of WAL and would refuse every
    #   candidate (Topic 42's 14-minute outage).
```

```sql
-- ★ ④ SYNCHRONOUS, AS A QUORUM, NOT A SINGLE NAME
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (pg-node-2, pg-node-3)';
ALTER SYSTEM SET synchronous_commit = 'on';
-- ★ synchronous_mode_strict: false
--   ⇒ if NO standby is available, prefer availability over RPO=0.
--   ⇒ ★ for a payments system this was debated; the decision was
--     recorded: "we would rather accept a bounded RPO than stop
--     taking payments." ★ WRITE THE DECISION DOWN.
```

**Step 4 — the application changes, which were half the fix.**

```
 ★ MEASURED DURING THE INCIDENT:
   database unwritable      03:14:32 → 03:14:35   ★ 3 seconds
   application errors       03:14:32 → ★ 03:29:41  ★ 15 minutes
 ⇒ ★ THE DATABASE FAILOVER WORKED. THE APPLICATION DID NOT NOTICE
   FOR FIFTEEN MINUTES.
```

```js
// ★ before — the defaults
const pool = new Pool({ host: 'pg-primary.internal', max: 20 });

// ★ after
const pool = new Pool({
  connectionString:
    'postgresql://app@pg1:5432,pg2:5432,pg3:5432/shop'
    + '?target_session_attrs=read-write'
    + '&connect_timeout=3',
  options: '-c tcp_user_timeout=6000',      // ★ 15 min → 6 s
  keepAlives: 1, keepAlivesIdle: 5,
  keepAlivesInterval: 2, keepAlivesCount: 3,
  connectionTimeoutMillis: 3000,
  statement_timeout: 5000,
  idle_in_transaction_session_timeout: 10000,
  max: 20,
});
```

```js
// ★ and the retry wrapper on every write path
const FAILOVER = new Set(['57P01','57P02','57P03','08006','08001','08004','25006']);

async function dbWrite(fn, { idempotencyKey }) {
  for (let i = 0; i < 6; i++) {
    try { return await fn(); }
    catch (e) {
      if (!FAILOVER.has(e.code) || i === 5) throw e;
      metrics.increment('db.failover_retry', { code: e.code });
      await sleep(Math.min(8000, Math.random() * (400 * 2 ** i)));
    }
  }
}
// ★ EVERY CALLER PASSES AN IDEMPOTENCY KEY (Topic 52).
//   A commit whose acknowledgement was lost during failover MAY
//   HAVE SUCCEEDED. Retrying without idempotency double-charges.
```

**Step 5 — a monthly game day, with the frozen-node case.**

```bash
#!/usr/bin/env bash
# ★ run in staging monthly, production quarterly, with a runbook.
set -euo pipefail

record() { echo "$(date -Ins) $*" >> /tmp/gameday.log; }

# start a continuous write probe so RTO is measured, not estimated
( while true; do
    psql "$MULTI_HOST_URL" -tAc \
      "INSERT INTO ha_probe (ts) VALUES (now())" >/dev/null 2>&1 \
      && record "write ok" || record "write FAIL"
    sleep 0.2
  done ) & PROBE=$!

record "=== scenario 3: FREEZE the primary (hypervisor stun) ==="
LEADER=$(patronictl -c /etc/patroni.yml list -f json \
          | jq -r '.[]|select(.Role=="Leader").Member')
ssh "$LEADER" 'sudo pkill -STOP -f "postgres:"'
sleep 90
ssh "$LEADER" 'sudo pkill -CONT -f "postgres:"'
sleep 30

# ★ THE ASSERTIONS THAT MATTER
record "=== assertions ==="
LEADERS=$(for h in pg1 pg2 pg3; do
  psql -h $h -tAc "SELECT pg_is_in_recovery()" 2>/dev/null; done \
  | grep -c '^f$' || true)
[ "$LEADERS" -le 1 ] || { record "★ SPLIT BRAIN: $LEADERS leaders"; exit 1; }

TLS=$(for h in pg1 pg2 pg3; do
  psql -h $h -tAc "SELECT timeline_id FROM pg_control_checkpoint()" \
    2>/dev/null; done | sort -u | wc -l)
[ "$TLS" -eq 1 ] || record "★ WARNING: divergent timelines ($TLS)"

kill $PROBE
# ★ RTO = the gap between the first FAIL and the next ok
awk '/write FAIL/{if(!s)s=$1} /write ok/{if(s){print "RTO:", $1, "from", s; exit}}' \
  /tmp/gameday.log
```
```
 RTO: 2026-08-20T03:44:18 from 2026-08-20T03:43:45
   ★ 33 seconds, measured — not estimated.
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Fencing | ★ software only | ★ **STONITH, promotion fails if unconfirmed** |
| Frozen-node write window | ★ up to 10 s | **0** (instance stopped before promotion) |
| Transactions lost/duplicated | 3, manually reconciled | **0** |
| Database RTO | 20–35 s | 25–33 s |
| **User-visible outage** | ★ **15 min** | **33 s** |
| `maximum_lag_on_failover` | 1 MB (untested) | ★ 128 MB (from measured WAL rate) |
| `synchronous_standby_names` | a single name | ★ `ANY 1 (n2, n3)` |
| Failover testing | once, at setup | ★ monthly, **including the freeze case** |
| RPO decision | assumed | ★ **written down and signed off** |

```
 ★ FIVE LESSONS:
 ① ★ SOFTWARE FENCING CANNOT FENCE A FROZEN NODE. It requires the
   node to be running in order to notice it lost the lease. STONITH
   is the only mechanism that works, and promotion must FAIL if
   fencing cannot be confirmed.
 ② ★ THE DATABASE FAILOVER WORKED PERFECTLY AND THE OUTAGE WAS 15
   MINUTES. Without tcp_user_timeout, a dead host hangs a
   connection for ~15 minutes and no amount of database HA helps.
 ③ ★ maximum_lag_on_failover MUST BE DERIVED FROM MEASURED WAL
   RATE. 1 MB sounds conservative and is 40 ms of WAL on a busy
   cluster — it refuses every candidate.
 ④ ★ SYNCHRONOUS REPLICATION WITH A SINGLE NAMED STANDBY MAKES
   YOU LESS AVAILABLE. Always a quorum.
 ⑤ ★ THE FROZEN-NODE TEST IS THE ONE THAT FINDS REAL BUGS.
   SIGKILL tests the easy path. SIGSTOP-then-resume tests fencing,
   and fencing is where split brain lives.
```

---

## Common mistakes

**1. Treating HA as backup.**
- *Symptom:* a `DROP TABLE` replicates in 3 ms and there is nothing to restore from.
- *Fix:* HA and PITR are different systems solving different problems (Topic 64).

**2. Software fencing only.**
- *Symptom:* a frozen node resumes and serves writes on the old timeline.
- *Fix:* STONITH, and **promotion must fail if fencing cannot be confirmed**.

**3. Default client timeouts.**
- *Symptom:* a 30-second database failover produces a 15-minute application outage.
- *Fix:* `tcp_user_timeout`, keepalives, `connect_timeout`, `statement_timeout`.

**4. No multi-host connection string.**
- *Symptom:* the application points at a hostname that must be repointed by a human or DNS TTL.
- *Fix:* `postgresql://…@h1,h2,h3/db?target_session_attrs=read-write`.

**5. Not retrying failover error codes.**
- *Symptom:* a burst of 500s during a successful, fast failover.
- *Fix:* retry `57P01`, `57P02`, `57P03`, `08006`, `25006` with jittered backoff — and idempotency (Topic 52).

**6. `synchronous_standby_names` with a single name.**
- *Symptom:* the standby goes down and **every commit blocks**; `wait_event = 'SyncRep'`.
- *Fix:* `ANY 1 (a, b, c)`.

**7. `maximum_lag_on_failover` set by intuition.**
- *Symptom:* no candidate is eligible and the cluster stays down (Topic 42's 14 minutes).
- *Fix:* `acceptable_loss_seconds × measured_WAL_bytes_per_second`.

**8. A promoted node left in the read pool.**
- *Symptom:* it accepts writes and diverges, silently.
- *Fix:* health checks must include `pg_is_in_recovery()`.

**9. `pg_rewind` without a backup.**
- *Symptom:* diverged transactions destroyed before anyone extracted them.
- *Fix:* copy the data directory first; `pg_waldump` the divergent WAL; reconcile, then rewind.

**10. `wal_log_hints` off.**
- *Symptom:* `pg_rewind` refuses; the old primary needs a full base backup to rejoin.
- *Fix:* `wal_log_hints = on` (or data checksums) from day one.

**11. Testing only the easy failure.**
- *Symptom:* SIGKILL passes; the first real incident is a hypervisor stun and fencing fails.
- *Fix:* test freeze-then-resume and DCS-only partitions.

**12. Quorum of two.**
- *Symptom:* a two-node etcd cluster loses one node and can no longer elect anything.
- *Fix:* 3 or 5 DCS nodes, in different failure domains.

**13. An unrecorded RPO decision.**
- *Symptom:* after data loss, nobody can say whether it was expected.
- *Fix:* write down the RPO/RTO targets, the configuration that implements them, and who signed off.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (promotion incrementing the timeline and writing a `.history` file, a promoted node accepting writes, a dead host hanging 14m52s vs 6s with `tcp_user_timeout`, `target_session_attrs=read-write` finding the writable node, measuring real RPO from `pg_stat_replication`, synchronous replication blocking on `SyncRep` with a single standby name, `pg_rewind` rejoining the old primary, and the split-brain detection loop).

**PROVE IT #9 — the frozen-node window, measured.**
```bash
LEADER_PID=$(ssh pg1 'pgrep -f "postgres: .*checkpointer"')
ssh pg1 "sudo kill -STOP $LEADER_PID"
sleep 45                       # ★ longer than ttl
ssh pg1 "sudo kill -CONT $LEADER_PID"
# ★ immediately try to write to the OLD primary
psql -h pg1 -c "INSERT INTO ha_probe (ts) VALUES (now())"
```
```
 ★ INSERT 0 1        — it accepted a write AFTER being superseded
```
```bash
sleep 15   # let Patroni's loop run
psql -h pg1 -c "INSERT INTO ha_probe (ts) VALUES (now())"
```
```
 ERROR:  cannot execute INSERT in a read-only transaction
   ★ THE WINDOW WAS ~10 SECONDS (loop_wait). STONITH closes it.
```

**PROVE IT #10 — quorum prevents promotion in a minority.**
```bash
# isolate pg1 from 2 of 3 etcd nodes
ssh pg1 'sudo iptables -A OUTPUT -d 10.0.1.22 -j DROP;
         sudo iptables -A OUTPUT -d 10.0.1.23 -j DROP'
sleep 40
patronictl -c patroni.yml list
```
```
 | pg-node-1 | 10.0.1.11 | ★ Replica | running | 8 | unknown |
 | pg-node-2 | 10.0.1.12 | ★ Leader  | running | 8 |         |
   ★ pg1 demoted itself. It could not reach a quorum and therefore
     could not renew the leader key.
```

**PROVE IT #11 — measure RTO with a probe, not a stopwatch.**
```bash
( while true; do
    if psql "$URL" -tAc "INSERT INTO ha_probe (ts) VALUES (now())" >/dev/null 2>&1
    then echo "$(date +%s.%N) ok"; else echo "$(date +%s.%N) fail"; fi
    sleep 0.1
  done ) > /tmp/probe.log &
patronictl switchover --force
sleep 60; kill %1
awk '/fail/{if(!s)s=$1} /ok/{if(s){printf "RTO: %.2fs\n", $1-s; exit}}' /tmp/probe.log
```
```
 ★ RTO: 4.82s        — a planned switchover
```

**PROVE IT #12 — extract data from a split-brain node before rewinding.**
```bash
pg_waldump -p /old/pg_wal -s "$DIVERGE_LSN" -t "$OLD_TIMELINE" \
  | grep COMMIT | wc -l
# ★ start read-only on a private port and dump the affected rows
pg_ctl start -D /old -o "-p 5599 -c listen_addresses=127.0.0.1"
psql -p 5599 -Atc "COPY (SELECT * FROM payments WHERE created_at > '…')
                   TO STDOUT WITH CSV HEADER" > /tmp/orphaned.csv
```

---

## The design decision framework

```
★★★ AN OUTAGE IS RECOVERABLE. SPLIT BRAIN IS NOT.
    EVERY TRADE-OFF FOLLOWS FROM THAT. ★★★

 ① ★ WRITE DOWN RPO AND RTO, AND GET THEM SIGNED OFF
    "how much committed data may we lose?"  ⇒ RPO
    "how long may we be unwritable?"        ⇒ RTO
    ⇒ ★ these are BUSINESS decisions. Ask, don't assume.
    ⇒ ★ then verify the configuration actually implements them.

 ② ★ YOUR REAL RPO IS THE p999 OF REPLICATION LAG
    not the average. p50 4 ms and p999 41 s (Topic 58).
    ⇒ a nightly bulk job and a hardware failure are independent
      and will eventually coincide.
    ⇒ ★ RPO = 0 requires synchronous replication:
      ✓ ALWAYS 'ANY 1 (a, b, c)' — ★ never a single name
      ✓ cost: 0.4 → 2.8 ms same-AZ, 14 ms cross-AZ
      ✓ ★ decide synchronous_mode_strict explicitly:
        strict = data loss is worse than downtime
      ★ THE HYBRID: sync same-AZ + async cross-region.

 ③ ★ FENCING IS THE WHOLE DISCIPLINE
    software self-demotion ✓ partitioned-but-healthy
                           ★ ✗ FROZEN-then-resuming
    ★ STONITH ⇒ the only mechanism for a frozen node
    ★ AND: PROMOTION MUST FAIL IF FENCING CANNOT BE CONFIRMED.

 ④ ★ QUORUM, ALWAYS
    3 or 5 DCS nodes, in ★ different failure domains
    ⇒ a minority partition can neither hold nor acquire leadership
    ⇒ ★ never 2 — losing one leaves you unable to elect.

 ⑤ ★ SET maximum_lag_on_failover FROM MEASUREMENT
    acceptable_loss_seconds × measured_WAL_bytes_per_second
    ⇒ ★ 1 MB on an 88 GB/hr cluster is 40 ms and refuses every
      candidate. It is an availability bomb.
    ⇒ ★ alert when observed lag approaches it.

 ⑥ ★ THE APPLICATION IS HALF THE RTO
    ✓ ★ tcp_user_timeout (default ~15 min ⇒ set to 5–10 s)
    ✓ keepalives_idle / _interval / _count
    ✓ connect_timeout, statement_timeout,
      idle_in_transaction_session_timeout
    ✓ ★ multi-host URL + target_session_attrs=read-write
    ✓ ★ retry 57P01/57P02/57P03/08006/08001/08004/25006
    ✓ ★ IDEMPOTENCY on every write (Topic 52) — a lost COMMIT ack
      may mean the commit succeeded
    ✓ ★ health checks include pg_is_in_recovery()

 ⑦ ★ TEST FOUR SCENARIOS — the third finds the real bugs
    ① planned switchover      ② SIGKILL
    ★ ③ SIGSTOP-then-resume (a frozen node) — tests FENCING
    ★ ④ partition from the DCS only — tests self-demotion
    ⇒ ★ measure RTO with a CONTINUOUS WRITE PROBE, not a stopwatch
    ⇒ ★ assert afterwards: at most one node with
      pg_is_in_recovery() = false, and a single timeline
    ⇒ monthly in staging, quarterly in production

 ⑧ PREPARE FOR THE REJOIN BEFORE YOU NEED IT
    ✓ ★ wal_log_hints = on (or data checksums) — pg_rewind needs it
    ✓ ★ copy the data directory BEFORE pg_rewind — it discards
      diverged transactions
    ✓ ★ pg_waldump the divergent WAL to enumerate what was lost

 ⑨ ★ AND REMEMBER WHAT HA IS NOT
    it does not protect against a bad DELETE, a bad migration, or
    corruption — ★ all replicate in milliseconds. That is Topic 64.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Promote a standby manually. Show: (a) `pg_is_in_recovery()` before and after; (b) the timeline id changing; (c) the `.history` file contents; (d) that the promoted node accepts writes. Then explain, in two sentences, why leaving it in a read pool is dangerous.

### Exercise 2 — medium (apply it)
Measure the client-side contribution to RTO. (a) Block the primary's IP with `iptables` and time a `psql` connection with default settings. (b) Repeat with `connect_timeout` and `tcp_user_timeout`. (c) Build a multi-host connection string with `target_session_attrs=read-write` and prove it finds the writable node. (d) Write the retry wrapper and list every SQLSTATE it must handle.

### Exercise 3 — hard (production simulation)
A three-node Patroni cluster with etcd quorum and synchronous replication experiences a 90-second hypervisor stun of the primary. Patroni promotes node-2 at 03:14:35. The frozen VM resumes at 03:15:32 and commits three more transactions before demoting itself.

(a) Explain precisely why software fencing failed here, and why this differs from a network partition.
(b) Compute the size of the write window and relate it to `loop_wait`.
(c) Write the commands to enumerate exactly what was committed after the divergence point, and the safe procedure for extracting it.
(d) Design the STONITH callback. State the property it must have and what must happen if it cannot confirm.
(e) The database was unwritable for 3 seconds; users saw 15 minutes. Explain the mechanism and give the four client settings that fix it.
(f) `maximum_lag_on_failover` was 1 MB. Given a measured 24 MB/s WAL rate and a 5-second acceptable loss, compute the correct value and explain what the original setting would have done.
(g) `synchronous_standby_names` was a single name. Show the failure mode with `pg_stat_activity` and give the fix.
(h) Write the monthly game-day script including the freeze scenario, a continuous write probe for RTO measurement, and the two post-conditions that detect split brain.
(i) Explain why `synchronous_mode_strict` is a business decision, and how you'd frame the question to a product owner.
(j) List everything that must be true *before* an incident for `pg_rewind` to be usable afterwards.

---

## Mental model checkpoint

1. Define RPO and RTO. Which one does synchronous replication address, and at what cost?
2. Why can you never reliably distinguish a dead primary from a partitioned one? What does that imply about tuning?
3. Why is bias toward false negatives correct?
4. Name the four fencing mechanisms. Which is the only one that works against a frozen node, and why?
5. What does a timeline increment prevent? What does it *not* recover?
6. What must be true before `pg_rewind` can be used, and what does it destroy?
7. Why is your real RPO the p999 of lag rather than the average?
8. Why is `synchronous_standby_names = 'node-2'` dangerous, and what is the correct form?
9. How does `maximum_lag_on_failover` cause an outage when set too low? Give the formula for setting it.
10. Name the four client settings that keep application RTO close to database RTO.
11. Which failover test finds the bugs the others miss, and why?
12. Why is HA not a backup?

---

## Quick reference card

**The two numbers:** **RPO** = data you may lose (async > 0; sync = 0) · **RTO** = time unwritable (detect + decide + promote + reconfigure).

**Four phases:** detection (★ quorum) → election (★ `maximum_lag_on_failover`) → ★ **fencing** → reconfiguration.

```yaml
# Patroni — the settings that matter
ttl: 30 · loop_wait: 10 · retry_timeout: 10   # loop + 2*retry <= ttl
maximum_lag_on_failover: 134217728   # ★ acceptable_s × measured_B/s
synchronous_mode: true
synchronous_mode_strict: false       # ★ an explicit business decision
postgresql: { use_pg_rewind: true, parameters: { wal_log_hints: on } }
```

```sql
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (n2, n3)';  -- ★ never one name
```

**Client — half the RTO**
```
postgresql://u:p@pg1,pg2,pg3/db?target_session_attrs=read-write&connect_timeout=3
options='-c tcp_user_timeout=6000'   # ★ default is ~15 MINUTES
keepalives=1 keepalives_idle=5 keepalives_interval=2 keepalives_count=3
★ retry: 57P01 57P02 57P03 08006 08001 08004 25006  — with idempotency (52)
★ health check: pg_is_in_recovery() = true AND lag < threshold
```

**Detect split brain**
```bash
for h in pg1 pg2 pg3; do psql -h $h -tAc \
  "SELECT pg_is_in_recovery(), (SELECT timeline_id FROM pg_control_checkpoint())";
done
# ★ more than one 'f', or divergent timelines ⇒ STOP WRITES
```

**Test:** switchover · SIGKILL · ★ **SIGSTOP-then-resume** · ★ **partition from the DCS only**. Measure RTO with a **continuous write probe**.

**★ HA is not backup.** `DROP TABLE` replicates in 3 ms (Topic 64).

---

## When would I use this at work?

1. **Reviewing any HA setup you inherit.** Three questions find most problems: *what fences the old primary?*, *what is `maximum_lag_on_failover` and where did that number come from?*, and *when was failover last tested with a frozen node?* In my experience the answers are "nothing", "a blog post", and "never".

2. **Whenever the client configuration is written.** A perfect 30-second database failover becomes a 15-minute outage without `tcp_user_timeout`. This is the single highest-leverage change on the application side and it is one connection-string parameter.

3. **Before agreeing to "zero data loss".** That means synchronous replication, which means every commit pays a round trip and an unreachable standby can block writes. Make the trade explicit, write it down, and get it signed off — so that after an incident nobody has to reconstruct what was intended.

4. **Running game days.** An untested failover is a hypothesis. The freeze-then-resume scenario is the one that finds fencing bugs, and fencing bugs are the ones that produce unrecoverable data divergence rather than a recoverable outage.

---

## Connected topics

**Understand before this:** 41/42 (WAL and recovery — timelines and replay), 58 (read replicas — lag distribution, `pg_is_in_recovery()`), 62 (replication internals — slots, `wal_log_hints`), 52 (idempotency — why retries during failover are safe).

**This unlocks:**
- **64** — backup and PITR: the protection HA does *not* provide
- **65** — connection pooling: where the client-side failover settings live
- **68** — CAP and consistency models: the formal framing of the RPO/availability trade
- **67** — performance investigation: distinguishing a failover from a slowdown
