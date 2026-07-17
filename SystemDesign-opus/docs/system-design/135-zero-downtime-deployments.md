# 135 — Zero Downtime Deployments
## Category: Advanced

---

## What is this?

Zero-downtime deployment is the practice of shipping new versions of your software **without anyone noticing** — no error pages, no "site under maintenance" banner, no dropped requests. Users keep browsing, buying, and clicking while you swap the engine of the plane mid-flight.

The naive way to deploy is: stop the old version, start the new one. In between, your service is dead. And if the new version is broken, you now have a full outage instead of a quiet failed deploy. Zero-downtime deployment is the collection of techniques — rolling updates, blue-green, canaries, careful database migrations, feature flags — that let teams ship dozens of times a day without a blip.

Analogy: it's like replacing the tyres on a race car **without stopping the race**. You can't halt; you swap one wheel at a time while the car keeps moving, and you keep a spare ready in case the new tyre is bad.

---

## Why does it matter?

If you deploy by stopping and restarting, every deploy is a mini-outage. That trains the whole company to **fear deploys** — so they batch a month of changes into one giant risky release, deploy it at 2 a.m. on a Sunday, and pray. Big batches mean big blast radius, and when something breaks you have no idea which of the 300 changes did it.

**The real-work angle:** modern engineering culture is built on shipping small and often. That's only safe if a deploy costs the user nothing. Zero-downtime deployment is what turns "deploy" from a scary event into a boring, hourly non-event. It's also a hard requirement for anything with an SLA — you cannot promise 99.99% uptime (about 52 minutes of downtime **per year**) if every deploy burns 5 minutes.

**The interview angle:** "How do you deploy a new version with no downtime?" and "How do you rename a database column that's in use by live traffic?" are extremely common. They test whether you understand that during a deploy **two versions of your code run at the same time**, and whether you can reason about backward compatibility and safe schema migrations. Getting the expand-contract pattern right in an interview signals real production maturity.

---

## The core idea — explained simply

### The analogy: renovating a busy restaurant that never closes

Imagine you own a restaurant that is open 24/7 and can **never** close, not even for an hour. You want to renovate the kitchen. You cannot just lock the doors. So instead:

- You have **several identical kitchens** (not one). You can take one offline, renovate it, and bring it back while the others keep cooking. Diners never notice.
- Before you send orders to a renovated kitchen, you check that it actually works — a **test plate** — and only then route real orders to it.
- If the new kitchen produces bad food, you instantly stop sending it orders and fall back to the old ones.
- When you change the **menu** (the shared thing everyone depends on), you can't just delete the old dish while orders for it are still in flight. You add the new dish first, let both run, then remove the old one later.

Map that back to software:

| Restaurant thing | Software concept |
|---|---|
| Several identical kitchens | **Multiple stateless instances** behind a load balancer |
| Taking one kitchen offline at a time | **Rolling update** |
| The test plate before real orders | **Readiness health check** |
| Instantly stop routing to a bad kitchen | **Rollback / traffic shift** |
| The shared menu everyone reads | The **database schema** |
| Add new dish, run both, remove old later | **Expand-contract migration** |
| A dish cooked but not served to test it | **Shadow traffic** |
| A dish on the menu but marked "coming soon" | **Feature flag** (deploy off, release later) |

**The single most important insight:** during almost every zero-downtime deploy, the **old version and the new version run at the same time**, serving live traffic, against the **same database**. Everything else in this doc follows from that one fact. If your new code can't coexist with the old code, you don't have zero downtime — you have a race condition.

---

## Key concepts inside this topic

### 1. The prerequisite — stateless services behind a load balancer

You cannot do any of this with a single server. Zero-downtime deployment **requires** that your service is:

- **Stateless** (recall [53 — HLD Fundamentals / stateless services]): no request depends on data stored in one specific instance's memory. Sessions live in Redis or a signed cookie, not in a local variable. Any instance can handle any request.
- **Horizontally scaled**: you run several interchangeable instances (recall [55 — Load Balancing]).
- **Behind a load balancer**: the LB is the single dial you turn to move traffic between instances and versions.

Why this is non-negotiable: the whole trick is to **take instances out of rotation one at a time**, replace them, and put them back — while the others keep serving. If a user's shopping cart lives only in the RAM of the instance you just killed, that user just lost their cart. Statelessness is what makes instances **interchangeable**, and interchangeability is what makes them **replaceable without downtime**.

Say this out loud in interviews first: "Zero-downtime deploys assume stateless, horizontally-scaled services behind a load balancer. If the service is stateful, we solve that first."

---

### 2. Rolling update

Replace instances **a few at a time**. The load balancer only routes to instances that pass their **readiness probe** (recall [88 — Containers and Orchestration]), so traffic never hits an instance that isn't ready.

How it works, step by step, with 4 instances (v1) rolling to v2:

1. Start 1 new v2 instance. Wait for its readiness check to pass.
2. Add v2 to the LB pool; remove 1 v1 instance (drain it first — see graceful shutdown, concept 7).
3. Repeat until all 4 are v2.

```
Rolling update (batch size 1, 4 instances)

Step 0:  [v1] [v1] [v1] [v1]      all old
Step 1:  [v2] [v1] [v1] [v1]      one replaced, both versions live
Step 2:  [v2] [v2] [v1] [v1]      50/50
Step 3:  [v2] [v2] [v2] [v1]
Step 4:  [v2] [v2] [v2] [v2]      done
              ▲
        during steps 1-3, v1 AND v2 serve real traffic at the same time
```

Kubernetes does this natively:

```yaml
# A rolling update in Kubernetes: never drop below 3 ready, never exceed 5.
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 instance down at a time
      maxSurge: 1         # at most 1 extra instance during the roll
  template:
    spec:
      containers:
        - name: api
          readinessProbe:          # LB only sends traffic once this passes
            httpGet: { path: /healthz/ready, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 3
```

**Trade-offs:** No extra infrastructure (you reuse the same instance budget), and it's gradual. But: (a) **both versions serve traffic during the roll**, which *forces backward compatibility*; (b) **rollback is slow** — to undo it you roll back the same slow way, instance by instance; (c) a bad deploy affects a growing share of users as the roll proceeds until you notice.

---

### 3. Blue-Green

Run **two complete environments**: **blue** (current, live) and **green** (the new version). Deploy to green, test it privately with production-like checks, then **flip all traffic** to green at the load balancer in one move. Keep blue warm and untouched so you can flip back **instantly** if green misbehaves.

```
Blue-Green

Before flip:                 After flip:
  Users                        Users
    │                            │
    ▼                            ▼
[  LB  ] ── 100% ──> [BLUE v1]  [  LB  ] ── 100% ──> [GREEN v2]
    x  (green idle)     GREEN v2    x  (blue idle, warm)  BLUE v1
                                                          ↑ instant rollback
```

The flip is usually a single change: update the LB target group, or swap a DNS/router weight from 0/100 to 100/0.

```js
// Conceptual: flip the router from blue to green in one atomic switch.
async function cutover(router) {
  await router.setWeights({ blue: 0, green: 100 }); // one atomic move
  // Rollback is symmetric and instant:
  // await router.setWeights({ blue: 100, green: 0 });
}
```

**Trade-offs:** **Fast cutover** and **instant rollback** (just flip back). But it **doubles your infrastructure** during the switch (you run two full environments), and you must handle two hard edges cleanly: **in-flight requests** on blue at the moment of flip (drain them, don't kill them), and the **shared database** — green and blue both talk to the *same* DB, so schema changes still must be backward-compatible (see expand-contract). Blue-green makes the *code* swap clean; it does **not** magically make *data* changes safe.

---

### 4. Canary

Release the new version to a **small percentage** of real traffic, watch the metrics, and only widen if it's healthy. A canary is a gradual, metrics-gated rollout: `1% → 5% → 25% → 50% → 100%`, pausing at each step.

```
Canary rollout

  Users ──> [ LB / traffic splitter ]
                 │            │
              95% │        5% │
                 ▼            ▼
            [v1] [v1] [v1]  [v2]  <-- the "canary"
                 │            │
                 └── metrics ─┴──> error rate? p99 latency? 5xx?
                        if bad  ─> auto rollback (send 0% to v2)
                        if good ─> widen to 25%, then 100%
```

This is **progressive delivery**: instead of one big switch, you advance in stages and let **automated analysis** decide whether to proceed. At each step you compare the canary's metrics against the baseline (recall [80 — Monitoring and Observability]) — error rate, latency percentiles, saturation — and **automatically roll back** if the canary is worse.

```js
// Sketch of automated canary analysis between steps.
const steps = [1, 5, 25, 50, 100];   // % of traffic to the new version
async function runCanary(deploy, metrics) {
  for (const pct of steps) {
    await deploy.setCanaryWeight(pct);
    await sleep(5 * 60_000);          // bake time: let real traffic hit it
    const c = await metrics.canary(); // new version's numbers
    const b = await metrics.baseline();
    // Fail fast: if the canary is meaningfully worse, abort and roll back.
    if (c.errorRate > b.errorRate * 1.5 || c.p99 > b.p99 * 1.3) {
      await deploy.setCanaryWeight(0); // instant rollback
      throw new Error(`Canary failed at ${pct}% — errorRate ${c.errorRate}`);
    }
  }
  await deploy.promote();             // canary becomes the new baseline
}
```

**Trade-offs:** The **safest** strategy for risky changes — a bad release only ever touches a tiny slice of users before it's caught and reverted. But it needs **good metrics and traffic-splitting infrastructure**, it's the **most complex** to set up, and rollouts are **slower** (bake time at each stage). Canary and rolling both run **both versions simultaneously**, so backward compatibility is mandatory.

---

### 5. Shadow / dark launch

Send a **copy** of real production traffic to the new version, but **throw away its responses** — users never see them. The new version gets exercised under real load, with real request shapes, without any risk to users. You compare its outputs and performance against the live version offline.

```
Shadow traffic

  Users ──> [ proxy ] ──> [v1] ──> real response to user
                 │
                 └── mirror ──> [v2 shadow] ──> response DISCARDED
                                                 (logged/compared only)
```

Use it to de-risk a rewrite, a new database, or a heavy algorithm change before it ever serves a user. **Caveat:** shadowed requests must not cause **side effects** — don't let the shadow charge a card, send an email, or write duplicate rows. Mirror reads freely; stub or sandbox writes.

---

### Strategy comparison

| Strategy | Extra infra cost | Rollback speed | Risk to users | Complexity |
|---|---|---|---|---|
| **Rolling** | None (reuses budget) | Slow (roll back the same way) | Grows during roll | Low |
| **Blue-Green** | High (2x during switch) | Instant (flip back) | Low if tested, but all-at-once | Medium |
| **Canary** | Low (a few extra instances) | Instant (weight to 0%) | Lowest (tiny slice) | High |
| **Shadow** | Medium (runs a copy) | N/A (never serves users) | None (responses discarded) | High |

---

### 6. Backward and forward compatibility — the real discipline

During a rolling or canary deploy, **old code and new code run at the same time**, against the **same database**, and often **call each other** (two services, or two instances of the same service behind an LB). This forces two rules:

- **Backward compatible:** new code must handle **old data** and **old requests**.
- **Forward compatible:** old code must not choke on **new data** it doesn't understand (e.g. an extra JSON field) — it should ignore what it doesn't know.

Concrete failures this prevents:

- New code writes a row with a new required column. Old code (still running) inserts a row *without* it → constraint violation, 500s.
- New code sends a request with a renamed field. An old instance receives it (LB round-robins), doesn't recognise the field → dropped data.
- New code assumes a column exists that the migration hasn't added yet on the replica it hit.

The rule of thumb: **change data and code in separate, individually-safe steps, never in one atomic "big bang."** Any single deploy must be safe to run **while the previous version is still live** and safe to **roll back to the previous version**. That constraint is the entire reason the next concept exists.

---

### 7. Database migrations without downtime — expand-contract (parallel change)

This is the most valuable practical skill in the doc. You **cannot** just `ALTER TABLE users DROP COLUMN name` while old code still reads `name`. The moment you drop it, every running old instance throws. Likewise you can't add a `NOT NULL` column with no default while old code inserts rows without it.

The safe technique is **expand-contract** (also called **parallel change**). You split one dangerous change into a sequence of individually-safe steps spread across **multiple deploys**:

1. **EXPAND** — make an **additive, non-breaking** schema change. Add the new column/table. Old code ignores it; new code isn't using it yet. Safe.
2. **Dual-write / migrate code** — deploy code that **writes both** the old and the new column, and **reads the new one with a fallback** to the old. Now both shapes are kept in sync.
3. **BACKFILL** — copy existing rows' old values into the new column, in **small batches** so you never lock the whole table.
4. **Flip reads** — once backfill is complete and verified, deploy code that reads **only** the new column (still writing both, for safety/rollback).
5. **Stop writing old** — deploy code that no longer touches the old column at all.
6. **CONTRACT** — now that **no running code** references the old column, drop it. Safe.

Each arrow between steps is a separate deploy, and **every intermediate state is safe to roll back to.**

#### Worked example: renaming `users.name` → `users.full_name`

```
Expand-Contract timeline: rename `name` -> `full_name`

DEPLOY 0   [schema]  name
           [code]    reads name, writes name
             │
             ▼  EXPAND: add nullable column, no lock, no default requirement
DEPLOY 1   [schema]  name | full_name(NULL)
           [code]    still only uses name        <- rollback-safe
             │
             ▼  DUAL WRITE + READ-WITH-FALLBACK
DEPLOY 2   [code]    writes BOTH; reads full_name ?? name
             │
             ▼  BACKFILL in batches (a job, not a deploy)
   UPDATE users SET full_name = name
   WHERE full_name IS NULL AND id BETWEEN ? AND ?;   -- 1000 rows at a time
             │
             ▼  FLIP READS
DEPLOY 3   [code]    reads full_name only; still writes both
             │
             ▼  STOP WRITING OLD
DEPLOY 4   [code]    uses full_name only; never touches name
             │
             ▼  CONTRACT: nothing references `name` anymore
DEPLOY 5   [schema]  full_name        (ALTER TABLE ... DROP COLUMN name)
```

The code at the crucial dual-write step:

```js
// DEPLOY 2: write both columns, read the new one but fall back to the old.
// This is what makes old rows (not yet backfilled) still work.
async function saveUser(db, user) {
  await db.query(
    `INSERT INTO users (id, name, full_name) VALUES ($1, $2, $2)
     ON CONFLICT (id) DO UPDATE SET name = $2, full_name = $2`,
    [user.id, user.fullName]                 // keep both in sync
  );
}

async function loadUser(db, id) {
  const row = (await db.query(`SELECT name, full_name FROM users WHERE id=$1`, [id])).rows[0];
  // Read new column, fall back to old for rows not yet backfilled.
  return { id, fullName: row.full_name ?? row.name };
}
```

The backfill job, batched so it never holds a long lock:

```js
// Backfill without locking the whole table: small batches, brief pauses.
async function backfill(db) {
  let last = 0;
  for (;;) {
    const res = await db.query(
      `UPDATE users SET full_name = name
       WHERE full_name IS NULL AND id > $1
       ORDER BY id LIMIT 1000 RETURNING id`, [last]   // 1000 rows per batch
    );
    if (res.rows.length === 0) break;
    last = res.rows[res.rows.length - 1].id;
    await sleep(100);   // let normal traffic breathe; avoid replica lag spikes
  }
}
```

**Three rules that make or break DB migrations:**

- **Never take a long lock on a big table.** `ALTER TABLE` that rewrites a large table can lock it for minutes and hang every query. Use **online schema-change tools** (`pt-online-schema-change`, `gh-ost` for MySQL; concurrent index builds and additive changes in Postgres) that build a shadow copy and swap without a long lock.
- **Additive changes are cheap; destructive changes are expensive.** Adding a nullable column is instant and safe. Dropping, renaming, changing type, or adding `NOT NULL` are the dangerous ones — always route them through expand-contract.
- **Make every migration reversible.** Each step must have a rollback. If you can't undo a step, you can't safely deploy the code that depends on it.

---

### 8. Feature flags — decouple deploy from release

**Deploy** = the code is shipped to production (but the feature may be dark/off). **Release** = the feature is turned on for users. A **feature flag** is a runtime switch that lets you separate the two.

```js
// Deploy the code disabled; release it later without another deploy.
async function checkout(user, cart, flags) {
  if (await flags.isEnabled("new-pricing-engine", { userId: user.id })) {
    return newPricingEngine(cart);   // dark until the flag turns on
  }
  return legacyPricing(cart);        // safe default while flag is off
}
```

Why this is the modern backbone of safe delivery:

- **Deploy risky code disabled.** The new path ships to prod but stays off. No user is affected until you decide.
- **Progressive release without redeploy.** Turn it on for internal users, then 1%, then 5%, then everyone — by changing a flag value, not by deploying.
- **Instant kill switch.** If the feature misbehaves, flip the flag off. That's **seconds**, versus a full rollback deploy (minutes).
- **It separates two risks.** A deploy can fail for infra reasons; a feature can fail for logic reasons. Flags let you debug them independently.

Feature flags are how canary-style *release* happens at the **feature** level rather than the **binary** level — and they pair perfectly with expand-contract (gate the "read new column" behaviour behind a flag).

---

### 9. Health checks and graceful shutdown

Two probes, doing different jobs (recall [88 — Containers and Orchestration]):

- **Readiness probe** — "should the LB send me traffic *right now*?" Gates traffic. Fails while the instance is warming up (loading config, filling caches, opening DB pools) and during shutdown. **This is what makes rolling deploys safe** — new instances get no traffic until ready.
- **Liveness probe** — "am I alive, or wedged and needing a restart?" Failing it gets you killed and replaced.

**Graceful shutdown (connection draining)** is the other half. When an instance is told to stop, it must **not** die instantly — it has in-flight requests. The correct sequence:

1. **Fail readiness** so the LB stops sending *new* requests to you.
2. **Finish in-flight requests** (a drain window — keep serving what's already accepted).
3. **Close connections and exit** only after the drain, or after a timeout.

```js
// Graceful shutdown: stop taking new work, drain the rest, then exit.
let ready = true;
app.get("/healthz/ready", (_req, res) =>
  res.status(ready ? 200 : 503).end());   // LB reads this

process.on("SIGTERM", async () => {
  ready = false;                            // 1) LB stops routing new requests
  await sleep(5000);                        //    give the LB time to notice
  server.close(async () => {                // 2) stop accepting, drain in-flight
    await db.end();                         // 3) close pools, then exit
    process.exit(0);
  });
  setTimeout(() => process.exit(1), 30000); // hard cap so we never hang forever
});
```

Skip draining and you **drop requests mid-deploy**: the orchestrator kills the pod, in-flight requests get a connection reset, users see errors — and your "zero-downtime" deploy just caused downtime for those users.

---

## Visual / Diagram description

The three main strategies side by side. Redraw these on a whiteboard:

```
ROLLING                 BLUE-GREEN                 CANARY
                                                   
[LB]                    [LB]                       [LB]
 ├ v2  (new)             │  flip                    ├─95%→ v1 v1 v1
 ├ v2                    ├→ BLUE  v1  (idle,warm)    └── 5%→ v2  (watch metrics)
 ├ v1  (being drained)   └→ GREEN v2  (live)         
 └ v1                                               widen only if healthy:
                        instant rollback = flip back  1%→5%→25%→100%
both versions live       one clean switch            or auto-rollback to 0%
during the roll          two full environments       tiny blast radius
```

- **Rolling:** the LB pool is a mix of v1 and v2 while the roll proceeds; each v1 is drained (readiness off) before removal.
- **Blue-Green:** two complete stacks; the LB points at exactly one. Rollback is pointing it back.
- **Canary:** one LB splitting a weighted stream; a metrics loop decides "widen" or "abort."

And the migration timeline (the money diagram) — see concept 7's expand-contract chart above: **EXPAND → dual-write → backfill → flip reads → stop writing old → CONTRACT**, each an independently rollback-safe deploy.

---

## Real world examples

### Netflix — Spinnaker and automated canaries

Netflix built and open-sourced **Spinnaker**, its continuous-delivery platform, and pioneered **automated canary analysis** (Kayenta). Deployments push a canary alongside a baseline, and an automated statistical comparison of metrics decides whether to promote or roll back. This is the canonical progressive-delivery setup and is widely cited in Netflix's own engineering blog posts (factual, from public sources).

### Amazon — many small deploys, blue-green style

Amazon has publicly stated it deploys extremely frequently (on the order of **thousands of deployments per day** across its services). Their tooling and AWS services (CodeDeploy supports blue-green and rolling; Elastic Load Balancing does connection draining) are built around exactly the strategies in this doc. The cultural point — small, frequent, low-risk deploys — is well documented.

### GitHub — gh-ost for online schema changes

GitHub open-sourced **gh-ost**, a triggerless online schema-migration tool for MySQL, specifically to alter huge production tables **without downtime and without long locks** — a direct implementation of the "never take a long lock, build a shadow copy and swap" rule. Their engineering blog describes running it on their largest tables (factual, from public sources).

---

## Trade-offs

**Strategy trade-offs** (summary of the comparison table):

| Concern | Rolling | Blue-Green | Canary |
|---|---|---|---|
| Infra cost | Lowest | Highest (2x) | Low |
| Rollback speed | Slow | Instant | Instant |
| Blast radius | Medium/growing | All-at-once after flip | Smallest |
| Setup complexity | Low | Medium | High |

**Zero-downtime deployment overall:**

| Pros | Cons |
|---|---|
| Ship many times a day, no user-visible downtime | Forces backward/forward compatibility discipline |
| Small batches → tiny blast radius, easy to debug | DB changes become multi-step (expand-contract) |
| Instant rollback / kill switch (with flags/canary) | Needs LB, health checks, metrics, automation |
| Enables high SLAs (99.99%+) | More moving parts to get wrong |

**The sweet spot:** default to **rolling updates** with proper readiness probes and graceful shutdown for everyday deploys; reach for **canary** on risky or high-blast-radius changes; use **blue-green** when you need instant rollback and can afford the double infra; and put **every** schema change through **expand-contract** while gating behaviour behind **feature flags**. The strategies are complementary, not either/or.

---

## Common interview questions on this topic

### Q1: "How do you deploy a new version with zero downtime?"

**Hint:** State the prerequisite first — stateless, horizontally-scaled instances behind a load balancer. Then describe a rolling update gated by readiness probes with graceful shutdown/connection draining. Mention that during the roll both versions serve traffic, so the deploy must be backward-compatible. Offer blue-green (instant rollback) and canary (safest for risky changes) as alternatives with their trade-offs.

### Q2: "You need to rename a column that live traffic reads. Walk me through it."

**Hint:** Expand-contract. Add the new nullable column (expand), deploy code that dual-writes and reads new-with-fallback, backfill in small batches, flip reads to the new column, stop writing the old, then drop it (contract). Emphasise that each step is a separate deploy and every intermediate state is rollback-safe, and that you never take a long lock on a big table.

### Q3: "What's the difference between deploy and release, and why separate them?"

**Hint:** Deploy = ship the code (possibly dark). Release = turn the feature on. Feature flags decouple them so you can deploy risky code disabled, release progressively to 1% then wider, and kill it instantly without a redeploy. This isolates infra risk from feature risk.

### Q4: "During a rolling deploy, what breaks if the new code isn't backward-compatible?"

**Hint:** Old and new run simultaneously against the same DB and call each other. New code writing a new required column while old code still inserts without it → constraint violations. New request shapes hitting old instances → dropped data. The fix is that any single deploy must be safe alongside the previous version and safe to roll back to.

### Q5: "Why isn't blue-green enough to make database changes safe?"

**Hint:** Blue and green share the **same database**. Blue-green makes the *code* cutover clean, but a destructive schema change still breaks whichever version isn't expecting it. Data changes always need expand-contract regardless of the deploy strategy; blue-green only solves the binary swap and in-flight-request draining.

---

## Practice exercise

### Migrate a column with zero downtime, end to end

Take a small Node.js + Postgres (or SQLite) app with a `users` table that has a single `name` column, and a `/users` API that reads and writes it. Your task: **rename `name` to `full_name` with zero downtime, simulated as a sequence of deploys.**

Produce:

1. **Deploy 1 (expand):** a migration adding a nullable `full_name` column. App code unchanged.
2. **Deploy 2 (dual-write):** update `saveUser` to write both columns and `loadUser` to read `full_name ?? name`. Seed a few rows the *old* way first, so some rows have `full_name = NULL`, and prove the fallback works.
3. **A backfill script** that copies `name → full_name` in batches of, say, 2 rows (so you can see batching), with a short pause between batches.
4. **Deploy 3 (flip reads):** read only `full_name`.
5. **Deploy 4 (stop writing old) + Deploy 5 (contract):** stop touching `name`, then a migration that drops it.

For each step, write one sentence explaining **why that intermediate state is safe to roll back to.** Bonus: add a feature flag around the "read new column" behaviour so you can flip reads without a code deploy. Expect ~30-40 minutes. The learning is in seeing that the app keeps working at *every* step.

---

## Quick reference cheat sheet

- **Zero downtime =** ship without users seeing errors; the old and new versions run **at the same time** during most deploys.
- **Prerequisite:** stateless, horizontally-scaled instances behind a load balancer — that's what makes instances interchangeable and replaceable.
- **Rolling update:** replace a few instances at a time; LB routes only to ready ones. Cheap, gradual, slow rollback, both versions live.
- **Blue-Green:** two full environments; flip all traffic at once; instant rollback; 2x infra; shared DB still needs care.
- **Canary:** 1% → 5% → 25% → 100%, gated by metrics, auto-rollback on regression. Safest for risky changes; needs metrics + traffic splitting.
- **Shadow / dark launch:** mirror real traffic to the new version, discard its responses; test under real load with no user risk (beware side effects).
- **Backward compatible:** new code handles old data/requests. **Forward compatible:** old code ignores new fields. Both required during a roll.
- **Expand-contract:** EXPAND (add column) → dual-write + read-with-fallback → BACKFILL in batches → flip reads → stop writing old → CONTRACT (drop). Multiple deploys, each rollback-safe.
- **Never long-lock a big table.** Use online schema-change tools (gh-ost, pt-online-schema-change, concurrent index builds).
- **Feature flags:** decouple **deploy** (ship code, dark) from **release** (turn it on). Instant kill switch, progressive rollout, no redeploy.
- **Readiness probe** gates traffic; **liveness probe** triggers restart. New instances get no traffic until ready.
- **Graceful shutdown:** fail readiness → drain in-flight requests → close and exit. Skip it and you drop requests mid-deploy.
- **Rule of thumb:** small, frequent, reversible deploys beat rare big-bang releases every time.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [88 — Containers and Orchestration](./88-containers-and-orchestration.md) | Rolling updates, readiness/liveness probes, and graceful shutdown are executed by the orchestrator you learned there. |
| **Next** | [139 — Evolutionary Architecture](./139-evolutionary-architecture.md) | Zero-downtime deploys are the mechanics; evolutionary architecture is the broader discipline of changing systems safely over time. |
| **Related** | [55 — Load Balancing](./55-load-balancing.md) | The load balancer is the dial you turn to shift traffic between versions in every strategy here. |
| **Related** | [63 — Database Replication](./63-database-replication.md) | Migrations and backfills must respect replica lag; both versions read from the same replicated store. |
| **Related** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) | Canary analysis and rollback decisions are only as good as the metrics you watch during the rollout. |
