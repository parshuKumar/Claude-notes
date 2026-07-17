# 90 — Disaster Recovery and Backup Strategies
## Category: HLD Components

---

## What is this?

Disaster Recovery (DR) is the plan, the infrastructure, and the practiced muscle memory that lets your system come back after something goes catastrophically wrong — a region outage, a bad deploy that shredded a table, ransomware, or an intern running `DELETE FROM orders` without a `WHERE` clause.

Think of it like a **fire drill in an office building**. Fire extinguishers on the wall are your backups. The lit exit signs are your runbooks. The drill you run twice a year — where everyone actually walks down 14 flights of stairs and gets timed — is the restore test. A building with extinguishers nobody has ever tested and staircases nobody has ever walked is not fire-safe; it just *looks* fire-safe.

**Disaster Recovery is not "we have backups." It is "we have proven, on a stopwatch, that we can be serving customers again within N minutes, having lost at most M minutes of data."** Those two letters — N and M — are the whole subject.

---

## Why does it matter?

Because the day it matters, it *really* matters, and you cannot build it retroactively.

**What breaks without it:**
- A migration drops a column. You restore last night's dump. You've lost 9 hours of orders and you cannot tell customers which ones.
- Ransomware encrypts your production DB. It also encrypts your backup volume, because the backup volume was mounted with the same credentials.
- `us-east-1` goes down. You have a "DR region," but the DNS record has a 24-hour TTL, and the runbook for switching it lives in a Confluence page hosted in `us-east-1`.
- You do restore successfully — after 31 hours, because nobody had ever timed a restore of a 4 TB database and it turns out `pg_restore` on a single core takes that long.

**The uncomfortable truth:** most real-world outages are **not** caused by hardware or acts of God. They are caused by **change** — a deploy, a config push, a feature flag, a certificate that expired, a schema migration. Google's SRE book puts roughly 70% of outages at "a change to a live system." Meaning: your DR plan must handle *"we broke it ourselves, five minutes ago"* far more often than *"the datacenter is on fire."*

**Interview angle:** Almost every senior HLD interview has a "what happens when this region dies?" moment. The interviewer is not looking for "we'd fail over." They're looking for: *what is your RPO? what is your RTO? how do you know?* Naming those two numbers and defending them separates a senior candidate from a mid-level one.

**Work angle:** DR is a compliance requirement (SOC 2, PCI-DSS, ISO 27001 all demand documented, *tested* recovery). It's also the thing that decides whether a bad Tuesday is a 20-minute blip or a company-ending event.

---

## The core idea — explained simply

### The Restaurant Fire Analogy

You run a busy restaurant. One night, the kitchen catches fire.

Two questions decide how bad this is:

**Question 1: "How much do we lose?"**
Your recipes, your reservation book, tonight's orders — how much of that is gone forever? If you photocopy the reservation book every morning, you lose one day. If you photocopy it every hour, you lose an hour. If you have a second person writing every reservation into a duplicate book in real time, you lose nothing. **That's RPO — Recovery Point Objective.** It looks *backwards* from the fire.

**Question 2: "How long until we're serving food again?"**
- If you have nothing: find a new location, buy ovens, hire staff. Weeks. (**Cold**)
- If you already rent an empty kitchen across town with the gas hooked up: install ovens, move in. Days. (**Pilot light**)
- If that second kitchen is small but *running*, serving a handful of takeaway orders: just expand it. Hours. (**Warm standby**)
- If you run two identical full restaurants and diners already eat at both: send tonight's diners to the other one. Minutes. (**Hot / active-active**)

**That's RTO — Recovery Time Objective.** It looks *forwards* from the fire.

And here's the part that gets restaurants — and companies — killed: **the photocopies in the safe are useless if nobody has ever opened the safe.** Half the pages are blank. The ink faded. The safe key is in the burned kitchen. **An untested backup is not a backup. It is a hope.**

### Mapping the analogy back

| Restaurant | System |
|---|---|
| The fire | Region outage, ransomware, bad deploy, `DROP TABLE` |
| Photocopying the reservation book | Taking a backup / replicating writes |
| How many hours of reservations are lost | **RPO** (data loss tolerance) |
| How long until you serve food again | **RTO** (downtime tolerance) |
| The empty rented kitchen | Pilot light DR region |
| The small running kitchen | Warm standby |
| Two full restaurants | Active-active |
| Opening the safe to check the photocopies | **Restore drill** — the only thing that makes DR real |
| Setting a small controlled fire on purpose to practise | **Chaos engineering** |

---

## Key concepts inside this topic

### 1. What actually counts as a "disaster"

Beginners picture a flood. Reality is far more mundane and far more frequent:

| Disaster | Frequency | What DR capability saves you |
|---|---|---|
| Bad deploy / bad config push | **Weekly, somewhere** | Fast rollback; feature flags |
| Fat-fingered `DROP TABLE` / `DELETE` without `WHERE` | Yearly, and it *will* happen | **Point-in-time recovery (PITR)** |
| Bad schema migration corrupts data | Quarterly | PITR + migration dry-runs |
| Expired TLS cert / expired credential | Yearly | Monitoring + automation, not DR |
| Single instance dies | Daily at scale | Autoscaling / health checks |
| Single AZ (datacenter) fails | ~yearly per cloud | **Multi-AZ** — cheap, do it always |
| Whole cloud **region** fails | Rare, but real (S3 2017, us-east-1 repeatedly) | **Multi-region DR** — expensive |
| Ransomware / compromised admin account | Rising fast | **Immutable, air-gapped backups** |
| Accidental account/project deletion | Rare, catastrophic | Backups in a *separate account* |

The pattern: **the cheap, frequent disasters are self-inflicted and data-shaped. The expensive, rare ones are infrastructural.** Most teams over-invest in surviving the rare one and under-invest in being able to rewind ten minutes.

---

### 2. RPO and RTO — the two numbers that drive every decision

These are the vocabulary of DR. Get them exactly right.

**RPO (Recovery Point Objective)** — the maximum amount of **data** you are willing to **lose**, measured **backwards** from the moment of disaster.
- RPO = 24h → "we take one nightly dump; if we blow up at 5pm we lose the whole day."
- RPO = 5 min → "we ship WAL segments to object storage every 5 minutes."
- RPO = 0 → "no committed write may ever be lost" → **synchronous replication**, which taxes every single write with a cross-region round trip.

**RPO dictates your replication/backup frequency.**

**RTO (Recovery Time Objective)** — the maximum **time** you are willing to be **down**, measured **forwards** from the moment of disaster.
- RTO = 24h → restore from cold backup, provision fresh infra.
- RTO = 15 min → pilot light: DB is already there, boot the app tier.
- RTO = 0 (near-zero) → active-active: traffic just shifts.

**RTO dictates your standby strategy.**

**The critical framing: these are BUSINESS decisions, not engineering ones.** You do not get to pick them; you *ask* for them. The question you put to the business is:

> "What does one hour of downtime cost us — in revenue, in SLA penalties, in churn, in regulatory fine? And what is one hour of lost writes worth?"

Because RPO=0 / RTO=0 is *achievable* — and costs perhaps 2.5–3× your infrastructure bill plus permanent added write latency plus a multi-region consistency problem you now have to solve forever. That is a bargain for a payments ledger and absurd for an internal analytics dashboard.

**Realistic RPO/RTO by tier:**

| Tier | Example system | RPO | RTO | Strategy | Rough cost multiple |
|---|---|---|---|---|---|
| 0 — Critical | Bank ledger, payment authorization | **0** (no committed write lost) | < 1 min | Active-active, sync replication | 2.5–3× |
| 1 — High | E-commerce checkout, ride dispatch | ~1 min | 5–15 min | Warm standby, async replication | 1.5–2× |
| 2 — Medium | User profiles, CMS, product catalog | ~15 min | 1–4 h | Pilot light + PITR | 1.2× |
| 3 — Low | Internal BI dashboard, reporting | 24 h | 1–3 days | Nightly backup & restore | ~1.05× |
| 4 — Rebuildable | Derived caches, search index | ∞ (re-derive it) | Hours | No backup — rebuild from source of truth | 1× |

Note tier 4: **not everything needs a backup.** A Redis cache or an Elasticsearch index rebuilt from Postgres does not need DR — it needs a documented, tested *rebuild* procedure. Knowing what you *don't* have to protect is part of the skill.

---

### 3. The four standby strategies

This is the table you should be able to draw from memory in an interview.

| # | Strategy | What's running in the DR region | RTO | RPO | Cost | Use when |
|---|---|---|---|---|---|---|
| 1 | **Backup & restore (cold)** | *Nothing.* Backups sit in object storage (S3/GCS), cross-region replicated. | **Hours → days** | Hours (last backup) | **$** — storage only | Tier 2–3. Non-critical services. The honest default for most internal tools. |
| 2 | **Pilot light** | The **data layer only** — a read replica of the DB, continuously replicating. App servers exist as machine images but are **switched off**. | **10s of minutes** | Seconds–minutes | **$$** | Tier 2. You accept 30 min down but *cannot* accept losing a day of data. |
| 3 | **Warm standby** | A **scaled-down but fully functional** copy — a couple of app servers, a smaller DB, LB configured. Handles no (or tiny) traffic. | **Minutes** | Seconds | **$$$** | Tier 1. You can scale up and shift DNS/anycast quickly. |
| 4 | **Hot standby / active-active** | **Full capacity**, serving **live production traffic** in both regions right now. | **Near zero** | ~0 (with sync replication) | **$$$$** | Tier 0. Also gives you lower latency for global users as a side benefit. |

**The step-changes to understand:**

- **1 → 2** is where you stop paying for compute and start paying for *continuously replicated data*. The DB is the slow part of any recovery; keeping it warm is where most of the RTO reduction comes from.
- **2 → 3** is where you stop having a "we think this will boot" region and start having a region that is *provably working*, because it's actually running your code. Warm standby catches config drift; pilot light does not.
- **3 → 4** is the big one. The moment both regions serve **writes**, you own a **multi-region data consistency problem**: conflicting writes, replication lag, split-brain. You are now doing distributed systems on hard mode. (This is a whole topic on its own — see [134 — Multi-Region Architecture](./134-multi-region-architecture.md).) Many teams run **active-active for reads, active-passive for writes** as a pragmatic middle ground: both regions serve traffic, but all writes route to one primary.

**A concrete cost sketch.** Say your primary region costs $50k/month (compute $30k, DB $15k, storage $5k):

```
Cold backup   : $5k cross-region storage + egress          ≈ $55k/mo  (1.1×)
Pilot light   : + $15k standby DB replica                  ≈ $70k/mo  (1.4×)
Warm standby  : + $15k DB + $9k (30% app tier)             ≈ $79k/mo  (1.6×)
Active-active : + $15k DB + $30k full app tier + traffic   ≈ $100k+/mo (2×+)
```

Now compare against downtime cost. If an hour down costs you $200k, spending $50k/month to cut RTO from 6 hours to 2 minutes is obviously correct. If an hour down costs you $2k, it obviously isn't. **The math is the argument. Do the math.**

---

### 4. Backups done properly

**Full vs incremental vs differential:**

| Type | What it stores | Backup cost | Restore cost |
|---|---|---|---|
| **Full** | Everything, every time | High (time + space) | **Lowest** — restore one file |
| **Incremental** | Only what changed since the **last backup of any kind** | **Lowest** | **Highest** — need the full + *every* incremental in the chain, in order. One corrupt link breaks the chain. |
| **Differential** | Everything changed since the **last full** | Medium (grows daily) | Medium — need full + *one* differential |

The trade-off is a see-saw: **cheap backups mean expensive restores.** And you optimize for the restore, because restores happen on your worst day. A common shape: weekly full + daily differential + continuous WAL archiving.

**Point-in-time recovery (PITR) — the single most valuable backup capability.**

Every serious database already writes a sequential log of every change before applying it — Postgres calls it the **WAL** (Write-Ahead Log), MySQL calls it the **binlog**. If you continuously ship those log segments to object storage, then recovery becomes:

> take the last full snapshot, then **replay the log forward** to any timestamp you choose.

Which means you can rewind to **10:42:59** — one second before the bad `DELETE` at 10:43:00.

```bash
# Postgres: continuous WAL archiving to S3 (postgresql.conf)
# archive_mode = on
# archive_command = 'aws s3 cp %p s3://my-wal-archive/%f'

# Recovery to one second before the incident (recovery.signal + postgresql.conf)
# restore_command   = 'aws s3 cp s3://my-wal-archive/%f %p'
# recovery_target_time = '2026-07-12 10:42:59+00'
```

This is what turns "we lost everything since last night's dump" (RPO = 12 hours) into "we lost 30 seconds" (RPO = 30 seconds) — *without* running a second live database. **If you implement exactly one thing from this doc, implement PITR.**

**The 3-2-1 rule (and the modern addition):**

```
3 copies of the data
2 different media / storage types
1 copy off-site (different region or provider)
──────────────────────────────────────────────
1 copy IMMUTABLE / air-gapped   ← the modern, non-negotiable addition
```

Why the fourth line exists: **ransomware and compromised admin credentials delete backups too.** That is now the *standard* attack: encrypt production, then wipe the backups so you must pay. If your app's IAM role, or your on-call engineer's admin role, has `s3:DeleteObject` on the backup bucket, then your backups are exactly as compromised as production.

> **A backup that your production credentials can delete is not a backup.**

Concretely, that means:
- Backups in a **separate cloud account** with a separate root of trust.
- **Object Lock / WORM** (write-once-read-many) with a retention period — S3 Object Lock in compliance mode, so *nobody*, not even the account root, can delete before the retention expires.
- **Versioning + MFA delete** on the bucket.
- The backup writer has `PutObject` only. Never `DeleteObject`.

```js
// The backup writer's IAM policy should look like this — no delete, ever.
const backupWriterPolicy = {
  Version: "2012-10-17",
  Statement: [{
    Effect: "Allow",
    // Note the absence of s3:DeleteObject and s3:PutBucketPolicy.
    // Retention is enforced by Object Lock, not by this principal.
    Action: ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
    Resource: ["arn:aws:s3:::acme-backups-vault/*"],
  }],
};
```

**The cardinal rule: an untested backup is not a backup — it is a hope.**

Restores fail. Constantly. Real reasons real restores have failed:
- The dump contained data but not the **schema** (`pg_dump --data-only`, forgotten).
- The archive was **truncated** — the upload silently failed at 94% and nobody alerted on it.
- **Encoding mismatch** — dumped from a `LATIN1` DB, restored into `UTF8`, emojis became mojibake.
- A **forgotten dependency** — the DB restored fine, but the S3 bucket holding user uploads had no backup at all.
- The **encryption key** for the backups lived only in the KMS of the region that died.
- Restore of the 4 TB DB took **31 hours**, not the 2 hours on the slide, because nobody ever measured it.

So you must run **real restore drills** — restore into a clean, isolated environment, run integrity checks, and **time it with a stopwatch.**

> **The measured restore time IS your true RTO. Not the number on the slide.**

Here's a restore drill you can actually run on a schedule:

```js
// scripts/restore-drill.js
// Runs weekly in CI. Restores the latest backup into a throwaway environment,
// verifies integrity, and records the wall-clock time. Fails loudly if anything drifts.
import { execFile } from "node:child_process";
import { promisify } from "node:util";
const run = promisify(execFile);

async function restoreDrill() {
  const startedAt = Date.now();
  const target = `drill-${Date.now()}`;

  // 1. Pull the LATEST backup — not a known-good one you keep around.
  //    The whole point is to test the backup you'd actually use.
  const { stdout: key } = await run("aws", [
    "s3api", "list-objects-v2",
    "--bucket", "acme-backups-vault",
    "--prefix", "pg/full/",
    "--query", "sort_by(Contents,&LastModified)[-1].Key",
    "--output", "text",
  ]);
  await run("aws", ["s3", "cp", `s3://acme-backups-vault/${key.trim()}`, "/tmp/backup.dump"]);

  // 2. Restore into a fresh, isolated database.
  await run("createdb", [target]);
  await run("pg_restore", ["--dbname", target, "--jobs", "8", "/tmp/backup.dump"]);

  // 3. Verify it is actually USABLE — not just that the command exited 0.
  //    A restore that produces an empty schema exits 0 too.
  const checks = [
    { name: "users exist",        sql: "SELECT count(*) FROM users",            min: 100_000 },
    { name: "orders exist",       sql: "SELECT count(*) FROM orders",           min: 500_000 },
    { name: "no orphaned orders", sql: "SELECT count(*) FROM orders o LEFT JOIN users u ON o.user_id=u.id WHERE u.id IS NULL", max: 0 },
    { name: "recent data present",sql: "SELECT count(*) FROM orders WHERE created_at > now() - interval '48 hours'", min: 1 },
  ];

  for (const c of checks) {
    const { stdout } = await run("psql", ["-d", target, "-tAc", c.sql]);
    const value = Number(stdout.trim());
    if (c.min !== undefined && value < c.min) throw new Error(`FAILED "${c.name}": got ${value}, expected >= ${c.min}`);
    if (c.max !== undefined && value > c.max) throw new Error(`FAILED "${c.name}": got ${value}, expected <= ${c.max}`);
  }

  const minutes = (Date.now() - startedAt) / 60_000;
  console.log(`Restore drill PASSED in ${minutes.toFixed(1)} min`);

  // 4. Publish the measured time. THIS is your real RTO — alert if it drifts
  //    past what you promised the business, because data grows and restores slow down.
  await emitMetric("dr.restore_drill.minutes", minutes);
  if (minutes > 60) throw new Error(`RTO BREACH: restore took ${minutes.toFixed(1)} min, budget is 60`);

  await run("dropdb", [target]);
}

restoreDrill().catch((err) => { console.error(err); process.exit(1); });
```

Note the last check: **alert when the restore time drifts past budget.** Your data grows 5% a month. Your restore time grows with it. The RTO you validated 18 months ago is fiction today unless you keep measuring.

---

### 5. Failover mechanics

Having a healthy standby is half the job. **Getting traffic to it** is the other half.

**DNS failover and the TTL lag.** The classic mechanism: a health check pings the primary; when it fails, you rewrite the DNS record to point at the standby. But recall from [54 — DNS] that DNS answers are **cached** by resolvers for the duration of the **TTL**. A 300-second TTL means clients keep hitting the dead region for up to 5 minutes after you flip — and some ISP resolvers and JVM clients ignore TTLs entirely and cache far longer.

Mitigations:
- Keep DR-relevant records at a **low TTL (30–60s)** *permanently*. Lowering TTL *during* an incident does nothing — the old, long TTL is already cached.
- Prefer **anycast / global load balancers** (AWS Global Accelerator, Cloudflare, GCLB) which fail over at the network layer in seconds, with no DNS involved.

**Health checks must be meaningful.** A check that pings `/health` and returns `200 OK` because the HTTP server is up — while the database connection pool is exhausted — is worse than no check at all. Health checks should exercise the **critical dependency path**, and you need *multiple* checkers in *different* regions, or you'll fail over because of a network partition between the checker and a perfectly healthy primary.

**Automatic vs manual failover — a genuine, important trade-off:**

| | **Automatic** | **Manual (human-triggered)** |
|---|---|---|
| Speed | Seconds | Minutes (paging, assessing, deciding) |
| Risk of **flapping** | High — a 30s blip triggers a failover, then a failback, then... | None |
| Risk of **split-brain** | Real — if it was a network partition, both regions now think they're primary and both accept writes | Low — a human confirms the primary is genuinely dead |
| Works at 3am | Yes | Only if the pager works and someone answers |
| Best for | **Stateless** tiers, read replicas, single-AZ instance failure | **Stateful** primaries, whole-region failover |

The industry pattern is a hybrid: **automatic within a region** (kill a bad instance, promote a replica in another AZ — cheap, safe, reversible), **manual across regions** (a human confirms and pushes the button, but the button is *one command* that has been rehearsed). The failure mode of automatic cross-region failover — two regions both accepting writes to the same accounts — is far worse than 10 extra minutes of downtime.

**Failing back is harder than failing over.** Everyone rehearses the failover. Almost nobody rehearses the return. When the primary region comes back, its data is *stale* — it's missing every write the DR region took while it was down. You cannot just flip DNS back. You must **reverse the replication direction**, let the old primary catch up, verify, *then* switch during a planned window. Teams routinely get stranded running out of their DR region for weeks because nobody planned the trip home. **Your runbook must have a "fail back" section.**

---

### 6. Chaos engineering — the discipline of proving DR works

Everything above is theory until you break something on purpose.

**Chaos engineering** is deliberately injecting failure into a system — ideally in **production** — to discover weaknesses before they discover you. Netflix's **Chaos Monkey** famously kills random EC2 instances during business hours. The point is not sadism. The point is: **if instances die every single day, engineers build systems that survive instance death as a matter of course.** Failure stops being an event and becomes background noise. You cannot ship code that can't survive a restart, because it would be killed within the hour.

The method is a scientific experiment, in four steps:

1. **Define the steady state.** A measurable business metric, not a CPU graph. "Checkout success rate stays above 99.5%." "Video starts per second stays within 5% of baseline."
2. **Hypothesize.** "If we kill one AZ's app servers, the steady state will hold, because the LB will route to the other two AZs."
3. **Inject the failure.** Kill instances. Add 300ms of latency to the payment service. Blackhole traffic to the primary DB. Fill a disk.
4. **Observe and learn.** Did the steady state hold? If yes, you've *earned* confidence. If not, you have found a bug on a Tuesday afternoon with the whole team watching — which is the cheapest possible time to find it.

**Safety rules — non-negotiable:**
- **Start in staging.** Graduate to production only once staging is boring.
- **Limit the blast radius.** One AZ, 1% of traffic, one service. Never everything at once.
- **Have an abort button** and make sure everyone knows it. The experiment must be stoppable in seconds.
- **Run during business hours**, with the owning team watching. Chaos at 3am with nobody around is not an experiment; it's an outage.
- **Never run it during an incident** or a code freeze.

```js
// A tiny latency-injection middleware — chaos you can run safely in staging.
// Real teams use Gremlin / AWS FIS / Chaos Mesh, but the shape is exactly this.
export function chaosMiddleware({ enabled, probability = 0.01, latencyMs = 300, errorRate = 0 }) {
  return async (req, res, next) => {
    // Two independent kill switches: env gate AND a runtime flag you can flip
    // to zero in one second. Never ship chaos with a single point of control.
    if (!enabled || !globalThis.__CHAOS_ARMED__) return next();

    // Never inject into anything on the critical money path or the health check
    // used to make failover decisions — you'd be causing the very failure you're testing for.
    if (req.path.startsWith("/health") || req.path.startsWith("/payments")) return next();

    if (Math.random() < probability) {
      req.log.warn({ chaos: true, path: req.path }, "chaos: injecting latency");
      await new Promise((r) => setTimeout(r, latencyMs));
      if (Math.random() < errorRate) {
        return res.status(503).json({ error: "chaos-injected failure" });
      }
    }
    next();
  };
}
```

Chaos engineering is DR's **feedback loop**. A DR plan without it is a document. With it, it's a capability.

---

### 7. Multi-AZ vs multi-region, and the correlated-failure trap

**Availability Zones (AZs)** are separate datacenters within one region, a few kilometres apart, with independent power and cooling but connected by low-latency (~1–2ms) private fibre.

| | Multi-AZ | Multi-region |
|---|---|---|
| Protects against | Datacenter fire, power loss, flooded rack | Region-wide outage, regional regulation, natural disaster |
| Failure frequency | **Common** — happens somewhere every year | **Rare** — but has happened repeatedly |
| Latency between | 1–2 ms → **synchronous replication is free** | 50–150 ms → sync replication taxes every write |
| Cost | ~1.1–1.3× (often just a checkbox) | 1.5–3× plus enormous complexity |
| Verdict | **Always do it.** No excuses. | Do it when the business math says so. |

**Multi-AZ is table stakes.** It's usually a config flag (`MultiAZ: true` on RDS) and it buys you the most common infrastructure failure for almost nothing. If you're not multi-AZ, fix that before reading further.

**The correlated-failure trap** — the mistake that kills otherwise-good DR plans. Your "independent" backup is not independent if it shares a hidden dependency:

- Your DR region depends on the **same global control plane** (IAM, DNS, the cloud console). When `us-east-1` sneezes, IAM and Route53 control planes have historically degraded *globally* — so you can't even *authenticate to run* your failover.
- Your **runbook lives in Confluence**, which is hosted in the region that's down. Print it. Keep it in a repo mirrored elsewhere. Keep it in the on-call's phone.
- Your **deploy pipeline** runs in the dead region — so you can't push the fix.
- Your **monitoring and paging** run in the dead region — so you don't even know you're down. (See [80 — Monitoring and Observability](./80-monitoring-and-observability.md); your observability stack must not share a fate with the thing it observes.)
- Your **secrets** are in a KMS key that's regional.
- Both regions run the **same buggy deploy**. A software bug is *perfectly correlated* across your regions. Multi-region does not save you from yourself — only **staged rollouts** do.

**The test:** for every dependency in your failover path, ask *"is this thing alive if the primary region is not?"* If you can't answer with confidence, it's a correlated failure waiting to happen.

---

## Visual / Diagram description

### Diagram 1 — The RPO/RTO timeline

This is the diagram to draw first, every time, on any DR whiteboard question.

```
                      DATA LOSS                      DOWNTIME
              ◀───────────────────────▶     ◀────────────────────────▶

  ────┬──────────────┬───────────────╳──────────────────┬────────────▶ time
      │              │            DISASTER              │
   last good      last            10:43:00           service
    backup /      committed                          restored
   replicated      write                             11:05:00
     write        (lost!)
    10:38:00      10:42:59
      │                                                 │
      └──── RPO = 5 min ────▶ ╳ ◀──── RTO = 22 min ─────┘
        "how much data we      │       "how long we
         are willing to        │        are willing
         LOSE"                 │        to be DOWN"
                               │
              measured BACKWARDS│FORWARDS measured
```

**Reading it:** the disaster is the `╳` at 10:43. Everything to the **left** — the gap between your last durable copy and the moment of death — is **data you lost**. That's RPO. Everything to the **right** — the gap between death and being back in business — is **time you were down**. That's RTO. They are completely independent knobs: you can have RPO=0 with RTO=6 hours (perfect synchronous backups, but you provision the recovery environment from scratch), or RPO=24h with RTO=1 minute (an instantly-available standby serving yesterday's data — which is usually worse than useless).

**RPO shrinks by backing up/replicating more often. RTO shrinks by keeping more infrastructure warm.** Different levers, different bills.

### Diagram 2 — The four standby strategies

```
   COST ──────────────────────────────────────────────────────────▶ HIGH
   RTO ◀────────────────────────────────────────────────────────── FAST

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ 1. BACKUP &  │  │ 2. PILOT     │  │ 3. WARM      │  │ 4. HOT /     │
│    RESTORE   │  │    LIGHT     │  │    STANDBY   │  │ ACTIVE-ACTIVE│
│    (cold)    │  │              │  │              │  │              │
├──────────────┤  ├──────────────┤  ├──────────────┤  ├──────────────┤
│              │  │              │  │  ┌────────┐  │  │  ┌────────┐  │
│              │  │              │  │  │ App×2  │  │  │  │ App×10 │  │
│  (nothing    │  │  (app OFF,   │  │  │(small, │  │  │  │ (FULL, │  │
│   running)   │  │   AMI ready) │  │  │ live)  │  │  │  │  live) │  │
│              │  │              │  │  └───┬────┘  │  │  └───┬────┘  │
│              │  │              │  │      │       │  │      │       │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌────▼─────┐ │  │ ┌────▼─────┐ │
│ │ S3 dump  │ │  │ │ DB       │ │  │ │ DB       │ │  │ │ DB       │ │
│ │ (cold)   │ │  │ │ replica  │ │  │ │ replica  │ │  │ │ primary  │ │
│ │          │ │  │ │ (LIVE)   │ │  │ │ (LIVE)   │ │  │ │ (LIVE,   │ │
│ │          │ │  │ │          │ │  │ │          │ │  │ │  writes) │ │
│ └──────────┘ │  │ └────▲─────┘ │  │ └────▲─────┘ │  │ └────▲─────┘ │
└──────────────┘  └──────┼───────┘  └──────┼───────┘  └──────┼───────┘
   RTO: hours–days   RTO: 10s of min    RTO: minutes     RTO: ~zero
   RPO: hours        RPO: seconds       RPO: seconds     RPO: ~zero
      $                  $$                 $$$              $$$$
        ▲                  │                  │                │
        │                  └──────────────────┴────────────────┘
   nightly dump                  continuous replication
   from primary                      from primary
```

**Reading it:** left to right, you are progressively **pre-paying** for recovery. The boxes fill up with running infrastructure, the RTO collapses, and the bill climbs. Notice that the **database is warm from strategy 2 onwards** — that's where the biggest RTO win lives, because restoring a large database is almost always the long pole. Notice also that only strategy 4 has the DR region taking **writes** — which is exactly where multi-region consistency becomes your problem.

---

## Real world examples

### Netflix — chaos as a daily habit

Netflix runs across multiple AWS regions in an active-active configuration for its core streaming path, and is the origin of the **Simian Army**: Chaos Monkey (randomly terminates production instances), and later Chaos Kong (simulated the failure of an *entire AWS region*). The strategic insight was cultural, not technical: by making instance death an hourly certainty, they made "survives instance death" a *precondition for shipping*, rather than a nice-to-have someone would get to next quarter. They also publicly ran regional evacuation exercises — proving they could shift all traffic out of a region in minutes — *before* they needed to. The design principle: assume every component is already failing, and structure the system so the user never notices.

### GitLab's 2017 database incident — the canonical backup horror story

In January 2017 GitLab suffered an incident that is required reading. During an emergency response to a spam-driven load spike, an engineer, intending to clear a broken replica, ran a directory removal against the **primary** database instead. Roughly 300 GB of production data was gone. Then came the part that made it famous: GitLab publicly documented that **five separate backup/replication mechanisms had all failed or were not working as intended** — pg_dump was silently failing due to a version mismatch, snapshots weren't taken on the relevant volume, and so on. They recovered from a staging snapshot that happened to be about six hours old and lost roughly six hours of data. GitLab live-streamed the recovery and published a full postmortem. Every single lesson in section 4 of this doc is in that postmortem. **Nobody knew the backups were broken because nobody had ever tried to restore from them.**

### AWS S3, us-east-1, February 2017 — the correlated-failure lesson

A typo in a command during routine debugging removed more S3 capacity in `us-east-1` than intended, and S3 in that region was substantially degraded for about four hours. A huge slice of the internet went down with it. The lesson that landed hardest: many companies had a "status page" or a monitoring dashboard whose *assets were hosted on S3 in us-east-1* — including, reportedly, AWS's own status dashboard's health icons. Their ability to *tell anyone they were down* depended on the thing that was down. **Check every hidden dependency in your failure path, including the ones you use to respond to failure.**

---

## Trade-offs

**Cost vs recovery speed:**

| Choice | You gain | You give up |
|---|---|---|
| Lower RTO (hot standby) | Near-zero downtime | 2–3× infra bill; a multi-region consistency problem; more surface area to keep in sync |
| Lower RPO (sync replication) | No data loss | Higher write latency on **every** write, forever; the primary can be blocked by a slow remote |
| Cold backups only | Very cheap | You will be down for hours or days, and you'll find out on your worst day |
| Automatic failover | Fast, works at 3am | Flapping; split-brain; failing over for a network blip that would have healed |
| Manual failover | A human confirms it's real; no split-brain | Minutes of extra downtime; depends on the pager working |
| Incremental backups | Small, fast, cheap | Slow, fragile restores — a single corrupt link breaks the whole chain |
| Chaos engineering in prod | Real, earned confidence | Real (bounded) customer impact; needs org buy-in and a mature on-call |
| Multi-AZ only | Cheap, survives the *common* failure | A region outage takes you fully down |
| Multi-region | Survives the rare, catastrophic failure | Big cost, big complexity — and it still doesn't save you from a bad deploy |

**The sweet spot for most teams:** **multi-AZ everywhere** (non-negotiable, nearly free) + **PITR with continuous WAL/binlog archiving** (this is what actually saves you from the `DROP TABLE`, which is the disaster you'll actually have) + **immutable, cross-account backups** (this is what saves you from ransomware) + **a quarterly, timed, honest-to-god restore drill** (this is what makes all of the above true instead of aspirational). Add a **pilot light** in a second region for your Tier-1 services. Only go **active-active** when someone can show you the arithmetic proving an hour of downtime costs more than the extra region.

**Rule of thumb:** *Spend your first DR dollar on being able to rewind ten minutes, not on surviving a region fire. The ten-minute rewind is the disaster you're actually going to have.*

---

## Common interview questions on this topic

### Q1: "What are RPO and RTO, and how would you set them for this system?"

**Hint:** Define them crisply — RPO is data loss measured *backwards* from the disaster; RTO is downtime measured *forwards*. Then **immediately turn it into a business question**: "What does an hour of downtime cost, and what's the value of an hour of writes?" Give tiered answers: the payments ledger gets RPO=0/RTO<1min (sync replication, active-active); the recommendation feed gets RPO=24h/RTO=hours (nightly backup, rebuild from source). The senior signal is refusing to pick numbers without asking what they're worth, and applying *different* numbers to different parts of one system.

### Q2: "Someone runs `DELETE FROM orders` with no `WHERE` clause in production. Walk me through the next 30 minutes."

**Hint:** (1) **Stop the bleeding** — put the app in read-only mode or take it offline immediately, so you don't build new state on top of corrupt state and make recovery harder. (2) **Establish the exact timestamp** of the bad statement from the audit/query log. (3) **Do not touch the primary.** Restore a *new* instance from the last base backup + replay WAL to `T - 1 second` (PITR). (4) **Reconcile** — if the disaster was 40 minutes ago, you must decide: full rollback (lose 40 min of good writes) or surgical restore (export the deleted rows from the PITR clone and re-insert them into the live primary — better, but you must handle sequences and FKs). (5) Postmortem: why did a human have `DELETE` on prod? Blameless — the fix is a safety net (`safe-updates` mode, mandatory transactions, a review-gated migration pipeline), not a scolding.

### Q3: "Would you use automatic or manual failover for a cross-region database failover?"

**Hint:** **Manual**, and be able to say why. Automatic cross-region failover for a *stateful write primary* risks **split-brain**: if the trigger was a network partition rather than a genuine death, you now have two primaries taking conflicting writes, and reconciling that is worse than the original outage. A human confirms it's real. But — and this is the key nuance — **the human's job is to make the decision, not to perform the surgery.** The failover itself must be a single, pre-written, rehearsed command. Contrast with *within*-region, *stateless* failover, which should absolutely be automatic. Mention flapping and hysteresis (require N consecutive failures from M independent checkers).

### Q4: "How do you know your backups actually work?"

**Hint:** The only honest answer: **because we restored from them last Tuesday and it took 41 minutes.** Describe an automated restore drill in CI: pull the *latest* backup (not a curated one), restore to a clean environment, run data-integrity assertions (row counts, referential integrity, recency of the newest row), **record the wall-clock time as a metric**, and alert when that metric drifts past the RTO budget — because data grows and restores get slower. Bonus points: mention that `pg_restore` exiting 0 tells you nothing; you must assert on the *content*. Cite GitLab 2017 as why.

### Q5: "You're multi-region. What might still take you completely down?"

**Hint:** Correlated failures. Enumerate: (a) a **bad deploy** — a software bug is perfectly correlated across regions; only staged/canary rollouts help. (b) A **global control plane** — IAM, DNS, the cloud console; if you can't authenticate, you can't fail over. (c) **Data corruption replicating** — async replication will faithfully copy your `DROP TABLE` to the DR region within milliseconds, which is precisely why *replication is not a backup*. (d) The **incident-response tooling** itself living in the dead region (runbook, monitoring, pager, deploy pipeline). This question is really testing whether you understand that redundancy only helps against *independent* failures.

---

## Practice exercise

### "Write the DR plan for an e-commerce checkout"

You own a Node.js e-commerce system on AWS `us-east-1`: an API tier behind an ALB, Postgres (RDS, single-AZ), Redis for sessions, S3 for product images, and an Elasticsearch index for search. Revenue is **$3M/month**, roughly evenly spread across the month.

Produce a one-page DR plan containing:

1. **A tier table.** Classify each of the five components into a DR tier (0–4). Justify each. (Hint: the Elasticsearch index is *derived* — what tier is that, really?)

2. **The downtime math.** Compute what one hour of full downtime costs in lost revenue. Show the arithmetic ($3M/month → per hour). Then state the RPO and RTO you will *propose to the business* for the checkout path, and the *maximum* you'd be willing to spend per month to achieve it.

3. **A chosen strategy** from the four (cold / pilot light / warm / hot), with a one-paragraph defence. Estimate the cost multiple.

4. **A backup design** for Postgres: full backup cadence, WAL archiving interval, retention, and specifically **how you make one copy immutable** and which IAM principal is allowed to write vs delete it.

5. **The RPO/RTO timeline diagram** for a disaster at 14:20, redrawn by hand from memory.

6. **A restore-drill checklist** — the exact steps and the exact integrity assertions you'd run, and where you record the measured time.

7. **One chaos experiment** for this system, written as a formal hypothesis: steady state → injection → blast radius → abort condition → what you expect to learn.

**Deliverable:** the one-page plan, plus one paragraph answering: *"If your DR region is an exact live replica of your primary, why does it not protect you from a bad migration?"* (This is the question that separates people who understand DR from people who have merely configured replication.)

---

## Quick reference cheat sheet

- **RPO = data you can lose**, measured **backwards** from the disaster. It dictates your **backup/replication frequency**.
- **RTO = time you can be down**, measured **forwards** from the disaster. It dictates your **standby strategy**.
- **RPO and RTO are business decisions, not engineering ones.** Ask what an hour of downtime costs before you pick a number.
- **Most outages are caused by CHANGE** — a deploy, a config push, a migration — not by hardware or fire. Design for that first.
- **The four strategies, in order of cost:** backup & restore (hours–days) → pilot light (10s of min) → warm standby (minutes) → hot/active-active (~zero).
- **PITR is the highest-value backup capability.** Continuous WAL/binlog archiving lets you rewind to one second before the bad `DELETE`.
- **3-2-1-1:** 3 copies, 2 media, 1 off-site, **1 immutable/air-gapped**.
- **A backup your production credentials can delete is not a backup.** Separate account, Object Lock, no `DeleteObject` on the writer role.
- **An untested backup is not a backup — it is a hope.** Restores fail constantly, and silently.
- **Your measured restore time IS your RTO.** Not the number on the slide. Time it, track it, alert when it drifts.
- **Replication is not a backup.** It faithfully replicates your `DROP TABLE` in milliseconds. Backups protect against *you*; replication protects against *hardware*.
- **Automatic failover for stateless/in-region; manual for stateful cross-region.** Automatic risks flapping and split-brain.
- **Failing back is harder than failing over.** Rehearse the trip home; put it in the runbook.
- **Chaos engineering** = steady-state hypothesis → inject → observe → learn. Start in staging, limit the blast radius, keep an abort button.
- **Multi-AZ is table stakes** (cheap, survives the common failure). **Multi-region is a business decision** (expensive, survives the rare one).
- **Watch for correlated failures:** shared control plane, runbook hosted in the dead region, monitoring in the dead region, the same bad deploy in both regions.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — DR is how you *actually deliver* the nines you promised; SLOs set the target, RPO/RTO set the recovery budget. |
| **Next** | [134 — Multi-Region Architecture](./134-multi-region-architecture.md) — the moment you choose hot standby / active-active, multi-region data consistency becomes your problem. |
| **Related** | [63 — Database Replication](./63-database-replication.md) — the mechanism behind pilot light, warm, and hot standbys; also the doc that explains why replication is *not* a backup. |
| **Related** | [138 — Testing Distributed Systems](./138-testing-distributed-systems.md) — chaos engineering and restore drills are DR's test suite; an untested plan is a document, not a capability. |
| **Related** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) — you cannot recover from what you cannot detect, and your monitoring must not share a fate with the system it watches. |
