# 98 — Design a Notification System
## Category: HLD Case Study

---

## What is this?

A **notification system** is the piece of infrastructure every other service in your company calls when it needs to *tell a human something*: "your order shipped," "someone liked your photo," "your OTP is 458213," "50% off this weekend."

Think of it as the **company post office**. Your accounting department doesn't own trucks, doesn't know postal codes, doesn't negotiate with couriers. It writes a letter, drops it in the outbox, and the post office figures out the rest — which courier, which route, retry if the address bounced, and a tracking number so you know it arrived. In software, the "couriers" are Apple's push service, Google's push service, Twilio for SMS, and SendGrid for email. Your notification system is the post office that stands between 40 internal services and those 4 external couriers.

---

## Why does it matter?

Because notifications are the system where **every hard distributed-systems problem shows up at once**, and where the failures are visible to *users*, not just to your dashboards.

**What breaks if you get it wrong:**

- **Duplicates.** A user gets "Your card was charged $4,200" five times. They panic, they call support, they tweet. This is not a cosmetic bug — it's a trust-destroying bug. And it happens *by default*, because message queues give you **at-least-once** delivery.
- **Lost transactional messages.** The OTP never arrives. The user can't log in. Your conversion drops and you never find out why.
- **Head-of-line blocking.** A marketing blast to 10M users floods the same queue that carries password-reset emails. Now logins are broken for two hours because a promo for winter jackets is in front of them.
- **Provider outages.** Twilio has a bad hour. Your workers hammer it with retries, your queue backs up, your workers pile up in a connection pool waiting on 30-second timeouts, and the whole notification tier falls over. One vendor took your system down with it.
- **Legal exposure.** A user unsubscribed. You sent anyway. Under GDPR/CAN-SPAM/TCPA that's a fine, not a bug report.

**Interview angle:** This is one of the most common HLD questions at Amazon, Uber, Swiggy, Stripe, and most product companies, because it cleanly separates people who have *read* about queues from people who understand **why** a queue exists. If you can explain load levelling, at-least-once + idempotency, and circuit breakers in one coherent story, you're demonstrating almost the entire HLD curriculum in a single design.

**Real-work angle:** Almost every backend engineer eventually maintains or integrates with one of these. And "our notification service double-sent" is a real incident you will one day be on call for.

---

## The core idea — explained simply

### The Post Office Analogy

Imagine your company's mail room during a normal week: a steady trickle of letters, maybe 10 an hour. One clerk handles it easily.

Now marketing decides to mail a catalogue to every customer — 10 million of them — and they want it out this afternoon. They dump 10 million envelopes on the clerk's desk at once.

Two things can happen:

**Option A (no queue):** The clerk is buried. Every other letter — including the legal notice that must go out today — is now under 10 million catalogues. The clerk quits. This is your API server crashing under a burst.

**Option B (a queue):** The envelopes go into **mail bins**. The clerks pull from the bins at whatever rate they can sustain. The bins absorb the spike; the clerks work at a steady pace; nothing is lost, it just takes longer to drain. And critically — there is a **separate bin for urgent mail** and a **separate bin for bulk mail**, so the legal notice never gets stuck behind the catalogues.

That is the entire architecture, and it maps one-to-one:

| Post office | Notification system |
|---|---|
| Departments dropping off letters | Internal services calling the Notification API |
| "Is this address on the do-not-mail list?" | User preference / opt-out check |
| Form letter with `<name>` blanks | Template + variables (never store rendered text) |
| Mail bins | The message queue — **load levelling** |
| Separate bin for urgent vs bulk | Priority queues (transactional > promotional) |
| Separate bin per courier | One queue **per channel** (push / SMS / email) |
| Clerks working the bins | Workers, one pool per channel |
| FedEx, USPS, DHL | APNs, FCM, Twilio, SendGrid |
| Tracking number | `notification_id` — your idempotency key |
| "Courier accepted it" ≠ "recipient read it" | Provider ACK ≠ delivered ≠ opened |
| Undeliverable pile | Dead-letter queue (DLQ) |
| "FedEx is on strike, use DHL" | Circuit breaker + multi-provider failover |
| Returned envelope: "moved, no forwarding address" | Expired/invalid device token — prune it |

The one idea to burn in: **the queue is not an optimization, it is the load-bearing wall.** Remove it and the whole design collapses the moment marketing presses Send.

---

## Key concepts inside this topic

We'll walk the **5-step framework from [93 — How to Answer an HLD Interview Question]**: requirements → estimation → API → data model → architecture, then deep dives and failure modes.

---

### 1. Requirements

**Clarifying questions to ask the interviewer first** (asking these *is* part of the score):

- Which channels? (Push, SMS, email, in-app? Assume all four.)
- Do we distinguish **transactional** vs **promotional**? (Yes — this changes everything.)
- Do we build the push/SMS/email delivery infrastructure? (**No.** We integrate third-party providers. Building an SMTP reputation or an APNs equivalent is a different, multi-year company.)
- Do users need opt-outs? (Yes — legally required.)
- Do we need scheduling? Time-zone-aware? (Yes.)
- Do we need delivery/open tracking? (Yes.)

**Functional requirements**

| # | Requirement | Notes |
|---|---|---|
| F1 | Send via **multiple channels**: push (iOS via APNs, Android via FCM), SMS, email, in-app | One API, many transports |
| F2 | Support **transactional** ("your order shipped") and **promotional** ("50% off") | Transactional = must arrive. Promotional = best effort, droppable |
| F3 | **Templating** with variables | Store `template_id` + `{"name":"Riya","order":"#4471"}` — never the rendered string |
| F4 | **User preferences and opt-outs** | Per-channel, per-category. Legally mandatory |
| F5 | **Scheduling** | "Send at 9am in the user's local time" |
| F6 | **Delivery status tracking** | queued → sent → delivered → opened / failed |

**Non-functional requirements** (these are the ones that actually shape the architecture)

| # | Requirement | Architectural consequence |
|---|---|---|
| N1 | **High throughput with sudden bursts** — a campaign is 10M messages in 10 minutes | → **a queue** (load levelling), + fanout worker |
| N2 | **Reliability** — a transactional notification must NOT be lost | → durable queue, retries, DLQ, at-least-once |
| N3 | **No duplicates** | → **idempotency** on `notification_id` (at-least-once forces this) |
| N4 | **Low latency for transactional** (OTP in < 5s end-to-end) | → **priority queues**, separate worker pools |
| N5 | **Graceful third-party failure handling** | → **circuit breaker** per provider + backup provider failover |
| N6 | Scalable to new channels (WhatsApp next quarter) | → channel abstraction, queue-per-channel |

Note the tension baked into N2 and N3: the queue gives you at-least-once (never lose it) which *directly causes* duplicates. You don't get to pick one. You take at-least-once **and then add idempotency on top**. That pairing is the single most important sentence in this document.

---

### 2. Capacity estimation

Do the arithmetic out loud. Round aggressively — the interviewer wants the *shape*, not the decimals.

**Baseline (steady state):**

```
Users                       = 10,000,000
Notifications / user / day  = 5
Total notifications / day   = 10M × 5 = 50,000,000/day

Seconds in a day            = 86,400  (call it ~100,000 for mental math)
Average QPS                 = 50,000,000 / 86,400 ≈ 580/s
                            → round to ~600/s average
```

600/s is nothing. A couple of Node processes handle that. **If this were the whole story, you would not need a queue.**

**Now the marketing blast:**

```
Campaign size               = 10,000,000 users
Delivery window             = 10 minutes = 600 seconds
Peak QPS                    = 10,000,000 / 600 ≈ 16,700/s
                            → round to ~17,000/s
Spike factor                = 17,000 / 600 ≈ 28x   → call it a 30x spike
```

**Say this explicitly in the interview:**

> "Average load is ~600/s, which is trivial. But a campaign is a **30x burst — ~17,000/s for ten minutes**. I am not going to provision 30x the machines to sit idle 23 hours a day, and I cannot let that burst reach the third-party providers directly because they rate-limit me anyway. So I absorb the burst in a **queue** and let workers drain it at a sustainable rate. **This spike is the entire architectural driver of the design.** Recall from [67 — Message Queues] that this is exactly *load levelling*: the queue converts a spike in *arrival rate* into a spike in *queue depth*, which is a number on a dashboard instead of an outage."

**Storage:**

```
Row size (id, user_id, channel, template_id, small JSON payload,
          status, priority, timestamps, attempts)  ≈ 500 bytes

Per day    = 50M × 500 B      = 25 GB/day
Per year   = 25 GB × 365      ≈ 9 TB/year
Over 5 yrs = ~45 TB
```

That is too much to keep hot. So: **keep 90 days in the operational DB (~2.3 TB), then archive to cold storage (S3/Glacier) and roll aggregates into an analytics store.** Nobody queries an individual push notification from 2023; they query "what was the open rate of campaign X."

**Bandwidth / cost:** a push payload is ~1 KB. 50M/day × 1 KB = 50 GB/day egress — negligible. The real cost is **per-message provider fees**: SMS is roughly $0.005–$0.01 each, so a 10M-user SMS blast costs **$50,000–$100,000**. This is why pruning dead device tokens and honoring opt-outs is a *finance* problem too, not just a correctness one.

---

### 3. API design

Keep the public API tiny. Services should not know what a queue is.

```
POST /v1/notifications
  Idempotency-Key: <caller-supplied UUID>     # see [85 — Idempotency]
  {
    "user_id":      "u_8891",
    "template_id":  "order_shipped_v3",
    "variables":    { "order_id": "#4471", "eta": "Tuesday" },
    "channels":     ["push", "email"],        # or "auto" → use user preference
    "type":         "transactional",          # transactional | promotional
    "priority":     "high",                   # high | normal | low
    "scheduled_at":  null                     # ISO ts, or null = send now
  }
  → 202 Accepted
  { "notification_id": "n_01HZX...", "status": "queued" }

GET  /v1/notifications/:id            → status, attempts, per-channel results
POST /v1/campaigns                    → { template_id, segment_id, schedule } → 202 { campaign_id }
GET  /v1/users/:id/preferences        → channel + category opt-ins, quiet hours, timezone
PUT  /v1/users/:id/preferences
POST /v1/webhooks/:provider           → provider calls US with delivery receipts
```

Two things to notice:

1. **`202 Accepted`, not `200 OK`.** You have not sent anything yet. You have *accepted responsibility* for sending it. Returning 200 "sent" would be a lie, and lying in an API is how you get paged.
2. **`POST /v1/campaigns` takes a segment, not 10 million user IDs.** More on that in the fanout deep dive — this is the difference between a design that works and one that times out.

---

### 4. Data model

**`notifications`** — the core operational table.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID / ULID (PK) | **This is the idempotency key.** Generated at API ingress, carried through the queue |
| `user_id` | UUID (indexed) | |
| `channel` | enum(`push`,`sms`,`email`,`in_app`) | One row **per channel** — "push + email" = 2 rows |
| `template_id` | string | FK to `templates` |
| `payload` | JSONB | The **variables**, not the rendered text |
| `status` | enum(`queued`,`sent`,`delivered`,`opened`,`failed`,`suppressed`) | |
| `priority` | enum(`high`,`normal`,`low`) | |
| `type` | enum(`transactional`,`promotional`) | Drives priority + opt-out rules |
| `scheduled_at` | timestamptz (nullable) | |
| `sent_at` | timestamptz (nullable) | When the *provider accepted* it |
| `attempts` | int, default 0 | |
| `provider` | string (nullable) | `twilio` / `twilio_backup` / `sns` — which one actually took it |
| `provider_msg_id` | string (nullable) | For matching the delivery webhook back to this row |
| `campaign_id` | UUID (nullable, indexed) | |

**`user_preferences`**

| Column | Type | Notes |
|---|---|---|
| `user_id` | UUID (PK) | |
| `channels` | JSONB | `{"push":true,"sms":false,"email":true}` |
| `categories` | JSONB | `{"marketing":false,"security":true,"order_updates":true}` |
| `quiet_hours` | JSONB | `{"start":"22:00","end":"08:00"}` |
| `timezone` | string | IANA, e.g. `Asia/Kolkata` |
| `global_opt_out` | boolean | The legal nuclear switch |

**`templates`**

| Column | Type |
|---|---|
| `id` | string (PK), e.g. `order_shipped_v3` |
| `channel` | enum |
| `subject` | string (email only) |
| `body` | text with `{{placeholders}}` |
| `locale` | string, e.g. `en-IN` |
| `version` | int |

**`device_tokens`** — the one people forget.

| Column | Type | Notes |
|---|---|---|
| `id` | UUID (PK) | |
| `user_id` | UUID (indexed) | **A user has many devices** — phone, tablet, old phone |
| `token` | string (unique) | The APNs/FCM token |
| `platform` | enum(`ios`,`android`,`web`) | |
| `last_seen_at` | timestamptz | |
| `is_active` | boolean | |

**Device tokens expire and get invalidated.** The user reinstalls the app, factory-resets, or just stops using that old tablet. APNs/FCM will tell you (in the response, or via their feedback endpoint) that a token is dead. **If you do not prune dead tokens, you are paying to send notifications to phones that no longer exist**, and your delivery-rate metric silently rots. Rule: on an `Unregistered` / `InvalidRegistration` response, mark `is_active = false` immediately.

**DB choice:** PostgreSQL for `user_preferences`, `templates`, `device_tokens` — small, relational, needs correctness. For `notifications` at 50M rows/day, use a partitioned table (by day) or a wide-column store like Cassandra keyed by `(user_id, created_at)` — the access pattern is write-heavy, append-only, read-by-user or read-by-campaign, with a TTL. Plus **Redis** for the preference cache and the dedupe set (both are on the hot path and must be sub-millisecond).

---

### 5. High-level architecture

```
                                  INTERNAL CALLERS
              ┌──────────┬──────────────┬──────────────┬──────────────┐
              │  Order   │   Payments   │  Marketing   │    Auth      │
              │ Service  │   Service    │   Console    │   Service    │
              └────┬─────┘  └────┬──────┘  └────┬──────┘  └────┬──────┘
                   │             │              │              │
                   └─────────────┴──────┬───────┴──────────────┘
                                        │  POST /v1/notifications
                                        ▼
                      ┌─────────────────────────────────────┐
                      │      NOTIFICATION API (Node)        │
                      │  • authn/authz the calling service  │
                      │  • validate schema                  │
                      │  • assign notification_id (ULID)    │
                      │  • rate-limit per caller + per user │
                      │  → returns 202 Accepted FAST        │
                      └───────────────┬─────────────────────┘
                                      │
                      ┌───────────────▼─────────────────────┐        ┌───────────────┐
                      │   PREFERENCE / OPT-OUT CHECK        │◀──────▶│  Redis cache  │
                      │  • global opt-out?                  │        │  (prefs, 5m)  │
                      │  • channel enabled?                 │        └───────┬───────┘
                      │  • category opted in?               │                │ miss
                      │  • quiet hours? → defer, not drop   │        ┌───────▼───────┐
                      │  MUST be fast — it is on every send │        │  Postgres     │
                      └───────────────┬─────────────────────┘        │  user_prefs   │
                                      │  allowed                     └───────────────┘
                                      ▼
                      ┌─────────────────────────────────────┐        ┌───────────────┐
                      │        TEMPLATE SERVICE             │◀──────▶│   templates   │
                      │  render(template_id, variables)     │        │   (+ cache)   │
                      │  store the ID + vars, NOT the text  │        └───────────────┘
                      └───────────────┬─────────────────────┘
                                      │
                      ┌───────────────▼─────────────────────┐
                      │   SCHEDULER  (if scheduled_at set)  │
                      │   holds it until due, then enqueues │
                      └───────────────┬─────────────────────┘
                                      │
      ════════════════════════════════▼═══════════════════════════════════════
                    THE QUEUE LAYER  —  ONE QUEUE PER CHANNEL,
                            EACH WITH HIGH / LOW PRIORITY
      ══════════════════════════════════════════════════════════════════════════
        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │  push.high   │  │  sms.high    │  │  email.high  │  │  inapp.high  │
        │  push.low    │  │  sms.low     │  │  email.low   │  │  inapp.low   │
        └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
               │                 │                 │                 │
               ▼                 ▼                 ▼                 ▼
        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │ PUSH WORKERS │  │ SMS WORKERS  │  │EMAIL WORKERS │  │IN-APP WORKERS│
        │  (autoscale) │  │  (autoscale) │  │  (autoscale) │  │              │
        │              │  │              │  │              │  │              │
        │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │  │  write to    │
        │ │  DEDUPE  │ │  │ │  DEDUPE  │ │  │ │  DEDUPE  │ │  │  inbox DB +  │
        │ │  RETRY   │ │  │ │  RETRY   │ │  │ │  RETRY   │ │  │  websocket   │
        │ │ BREAKER  │ │  │ │ BREAKER  │ │  │ │ BREAKER  │ │  │              │
        │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │  │              │
        └──┬────────┬──┘  └──┬────────┬──┘  └──┬────────┬──┘  └──────────────┘
           │        │        │        │        │        │
           │        └────────┼────────┴────────┼────────┴───────┐
           │   fail          │                 │                │  permanent
           ▼   permanently   ▼                 ▼                ▼  failure
     ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────────┐
     │   APNs   │      │  Twilio  │      │ SendGrid │      │ DEAD-LETTER  │
     │   FCM    │      │  ──────  │      │  ──────  │      │    QUEUE     │
     │ (iOS/And)│      │  backup: │      │  backup: │      │  + alert on  │
     │          │      │  AWS SNS │      │  AWS SES │      │  DLQ depth!  │
     └────┬─────┘      └────┬─────┘      └────┬─────┘      └──────────────┘
          │                 │                 │
          │  delivery receipts (async webhooks — minutes later)
          └─────────────────┴─────────────────┴──────┐
                                                     ▼
                                    ┌────────────────────────────┐
                                    │   WEBHOOK RECEIVER         │
                                    │   (idempotent! providers   │
                                    │    retry their webhooks)   │
                                    └──────────────┬─────────────┘
                                                   ▼
                          ┌──────────────────────────────────────────┐
                          │  TRACKING DB  →  ANALYTICS / DASHBOARDS  │
                          │  sent / delivered / opened / failed      │
                          └──────────────────────────────────────────┘
```

**Reading the diagram, box by box:**

- **Notification API.** The single front door. It authenticates the *calling service*, validates the request, mints a `notification_id`, and returns `202` in a few milliseconds. It does almost no work — because the caller (a payments service, say) must never be slowed down by SendGrid being sluggish.
- **Validation + preference check.** "Should this user receive this at all?" Global opt-out, channel toggle, category toggle, quiet hours. **This runs on every single send — 17,000/s during a blast — so it must be fast.** A Postgres round-trip per message would melt the DB. Cache preferences in Redis with a short TTL and invalidate on write. Note the distinction: a *quiet-hours* hit means **defer** (re-schedule for 8am), while an *opt-out* means **suppress** (record `status=suppressed`, never send). Deleting vs delaying is a legal difference.
- **Template Service.** Renders `order_shipped_v3` + `{order_id: "#4471"}` into real text. **Store the template ID and the variables, never the rendered output.** Why? Because when you find the typo in "Your ordre has shipped," you fix one template row and every future send is correct — instead of discovering that 40 million rows contain the misspelling. It also makes localization free (same ID, different `locale` row) and keeps rows small.
- **The Queue.** See below — this is the heart.
- **Workers, one pool per channel.** Each pool pulls from its own queue, does dedupe → provider call → retry → circuit-break, and writes status.
- **Third-party providers.** APNs (iOS), FCM (Android), Twilio (SMS), SendGrid or Amazon SES (email). **You do not build these.** You integrate them. And you must design assuming they **will fail, will rate-limit you, and will have outages** — because they will.
- **Tracking DB + analytics.** Every state transition lands here. This powers "did the OTP go out?" for support and "what's the open rate?" for marketing.

---

### 6. Why a queue **per channel** (not one shared queue)

This is the design point interviewers push on. The answer has four parts:

**(a) Head-of-line blocking.** With one shared queue, a backed-up email provider stalls the workers, and now the OTP push notification sitting behind 2 million marketing emails waits 40 minutes. The user cannot log in. **A slow email provider must never be able to delay an urgent push.** Separate queues = separate fates. This is *bulkheading*: you build watertight compartments so one flooded compartment doesn't sink the ship.

**(b) Different throughput profiles.** Push can do tens of thousands/sec. SMS providers might cap you at 100/sec on your account tier. Email sits in between. One shared queue means one shared consumer rate — you'd be throttled to the *slowest* channel.

**(c) Different failure characteristics.** APNs failing means "your certificate expired." Twilio failing means "you're out of credit." SendGrid failing means "your IP reputation tanked." These need different retry policies, different circuit breakers, different alerts, different on-call runbooks.

**(d) Independent scaling.** During a campaign you might need 200 email workers and 5 SMS workers. With one queue you can't scale them separately.

**And within each channel, priority:**

```
push.high  ← OTP, "your payment failed", "your driver is here"
push.low   ← "Riya just posted a photo", "50% off jackets"
```

Workers drain `high` until it's empty, *then* touch `low` (with a starvation guard — e.g. always give `low` 10% of capacity so a firehose of high-priority traffic doesn't leave promo messages stuck forever). This is how a transactional OTP jumps ahead of a marketing blast that's already 8 million messages deep in the queue.

**Implementation note:** with SQS you literally create separate queues. With Kafka, separate topics (and the partition count per topic becomes your parallelism knob). With RabbitMQ, separate queues bound to a topic exchange on routing key `push.high`, `email.low`, etc.

---

## Visual / Diagram description

The big architecture diagram is above. Here is the second one you must be able to draw — **the lifecycle of a single notification**, because it's the diagram that makes the reliability story concrete.

```
 t=0ms    Order Service ──POST──▶ API
                                  │  mint notification_id = n_01HZX
                                  │  202 Accepted ─────▶ back to caller (done in ~8ms)
                                  ▼
 t=10ms   prefs check (Redis hit, 1ms)   ── opted out? ──▶ status=SUPPRESSED  (END)
                                  │ allowed
                                  ▼
 t=12ms   render template → enqueue to push.high     status=QUEUED
                                  │
          ┌───────────────────────┴───────────────────────┐
          │            (message sits in queue)            │
          └───────────────────────┬───────────────────────┘
                                  ▼
 t=300ms  Worker receives msg
                                  │
                    ┌─────────────▼─────────────┐
                    │ SETNX dedupe:n_01HZX      │──── already set? ──▶ ACK & DROP
                    │ (idempotency guard)       │                     (duplicate!)
                    └─────────────┬─────────────┘
                                  │ first time
                    ┌─────────────▼─────────────┐
                    │ circuit breaker: is APNs   │─── OPEN ──▶ fail fast
                    │ healthy?                   │             → backup provider
                    └─────────────┬─────────────┘             → or requeue w/ delay
                                  │ CLOSED
                                  ▼
 t=350ms  call APNs ──▶ 200 OK    status=SENT       ("provider ACCEPTED it")
                    ──▶ 5xx/timeout → retry w/ exponential backoff + jitter
                                     (1s, 2s, 4s, 8s… ±30% jitter)
                    ──▶ still failing after N tries → DEAD-LETTER QUEUE + alert
                                  │
          ┌───────────────────────┴───────────────────────┐
          │        (asynchronous — seconds to hours)      │
          └───────────────────────┬───────────────────────┘
                                  ▼
 t=4s     APNs → webhook → "delivered"    status=DELIVERED
 t=90s    user taps it → SDK → API        status=OPENED
```

Two things this diagram is *designed to teach you*:

1. **The caller is gone by t=8ms.** Everything after that is asynchronous. That decoupling is why a Twilio outage does not take down your checkout flow.
2. **`SENT` and `DELIVERED` are different states, and `SENT` is the only one you control.** More on that below — it is the subtlest point in the whole design.

---

## The reliability deep dive (the most important section)

This is where the interview is won. Everything above is table stakes.

### R1. At-least-once delivery + idempotency

Every real message queue (SQS, Kafka, RabbitMQ) gives you **at-least-once** delivery. It cannot give you exactly-once across a network boundary — that's a theoretical impossibility once the provider call is involved. Concretely:

> Your worker calls APNs. APNs sends the push. The worker then tries to ACK the message back to the queue — and the worker's pod is killed mid-ACK. The queue never saw the ACK, so after the visibility timeout it **redelivers the message to another worker**, which sends the push **again**.

The user just got charged-notification #2. Nothing was "broken." This is the queue working *exactly as designed*.

**Recall from [85 — Idempotency]:** an operation is idempotent if doing it twice has the same effect as doing it once. Push notifications are *not* naturally idempotent, so you must **make** them idempotent by remembering what you've already done, keyed on the `notification_id` minted at the API.

The guard must be **atomic** (two workers can get the same message at the same instant) and it must go **before** the provider call:

```js
// dedupe.js — Redis SET NX is atomic: exactly one caller gets `true`.
const DEDUPE_TTL_SECONDS = 24 * 60 * 60; // outlive any retry window

export async function claimNotification(redis, notificationId) {
  // "SET key value NX EX ttl" → returns "OK" only if the key did NOT exist.
  const result = await redis.set(
    `dedupe:${notificationId}`,
    'in-flight',
    'NX',
    'EX',
    DEDUPE_TTL_SECONDS
  );
  return result === 'OK'; // true = you own it. false = someone else already sent it.
}
```

**Why 24h TTL?** It must be longer than the longest possible retry/redelivery window, or a late redelivery finds an expired key and re-sends. Too long and you leak memory. 24h is a safe default for a queue whose max backoff is minutes.

**Why not just check the DB `status` column?** Because `SELECT status` then `if (status !== 'sent')` is a **read-then-write race** — two workers both read `queued` and both send. `SET NX` collapses the check and the claim into one atomic operation. (If you want DB-only, use `UPDATE notifications SET status='sending' WHERE id=$1 AND status='queued'` and check `rowCount === 1` — same trick, atomic conditional write.)

### R2. Retries with exponential backoff + jitter

A `503` from Twilio is usually *transient*. Retry it. But retry it **politely**:

```js
// backoff.js
export function backoffMs(attempt, { base = 1000, cap = 60_000, jitter = 0.3 } = {}) {
  const exp = Math.min(cap, base * 2 ** attempt);        // 1s, 2s, 4s, 8s, 16s...
  const delta = exp * jitter;                            // ±30%
  return Math.round(exp - delta + Math.random() * 2 * delta);
}
```

**Why jitter?** Because a provider blip fails 5,000 in-flight messages *at the same instant*. Without jitter, all 5,000 retry at exactly t+1s, then all at t+3s — a **thundering herd** that re-kills the provider the moment it recovers. Jitter smears the retries across a window so recovery actually sticks.

**Retry only transient failures.** Classify:

| Provider response | Class | Action |
|---|---|---|
| 429 Too Many Requests | Transient | Back off (honor `Retry-After`), retry |
| 500 / 502 / 503, timeout, connection reset | Transient | Retry with backoff |
| 400 Bad Request (malformed payload) | **Permanent** | Do NOT retry → DLQ. It will fail forever |
| 401 / 403 (bad credentials) | **Permanent** | Do NOT retry → DLQ **and page someone**, this is a config bug |
| `Unregistered` / `InvalidRegistration` (dead device token) | **Permanent** | Do NOT retry. **Delete the token**, mark suppressed |

Retrying a permanent failure 5 times is just burning money and queue capacity.

### R3. Dead-letter queue

After N attempts (typically 5), stop. Move the message to a **dead-letter queue** — a separate queue for messages that will never succeed. This does three things:

1. **It stops the poison pill.** One malformed message that crashes your worker would otherwise be redelivered forever, taking down the pool in a loop. The DLQ ejects it after N tries.
2. **It preserves the evidence.** Nothing is silently dropped. You can inspect, fix, and replay.
3. **It becomes an alarm.** `DLQ depth > 0` for transactional messages should page someone. `DLQ depth growing at 500/min` means a provider is down or you shipped a bad payload. **DLQ growth rate is one of the highest-signal alerts you will ever configure** — it's the one metric that says "users are not getting messages."

### R4. Circuit breaker per provider (+ multi-provider failover)

**Recall from [73 — Circuit Breaker]:** when a downstream dependency is clearly broken, stop calling it. Fail *fast* instead of failing *slow*.

Why this matters here: if Twilio starts timing out at 30 seconds, and you have 100 workers, all 100 workers are now blocked for 30 seconds per message. Your effective throughput drops to ~3/s, the SMS queue backs up into the millions, and your retries are actively making Twilio's recovery harder. **The failure amplifies itself.**

The breaker has three states:

```
   CLOSED ──(failure rate > 50% over N calls)──▶ OPEN
     ▲                                            │
     │                                            │ (after cooldown, e.g. 30s)
     │                                            ▼
     └────(a trial call succeeds)──────────── HALF_OPEN
                                                  │
                                    (trial call fails) │
                                                  └──▶ OPEN
```

- **CLOSED** — normal, calls go through.
- **OPEN** — the provider is considered down. Reject immediately, in microseconds, without touching the network.
- **HALF_OPEN** — after a cooldown, let *one* call through as a probe. Success → CLOSED. Failure → back to OPEN.

**And here is the impressive part: when the breaker for Twilio is OPEN, you don't just fail — you fail *over*.** Route the message to a **backup provider** (AWS SNS for SMS, Amazon SES for email). Multi-provider failover means a Twilio outage becomes a *cost* event and a *latency* event, not a *downtime* event. Real companies do exactly this, and saying it out loud in an interview is a strong signal.

The trade-off to name: running two providers means maintaining two integrations, two sets of credentials, two sets of quirks, and reconciling two sets of delivery webhooks. You take that cost for SMS and email (where outages are common and the money is big) and often skip it for push (where APNs/FCM are the only games in town — there is no backup for APNs).

### R5. What "delivered" actually means

This is the point that separates a good answer from a great one.

**When APNs returns `200 OK`, that means: "I have accepted this message." It does NOT mean the user's phone got it.** The phone might be off. It might be in airplane mode over the Atlantic. The user might have force-quit the app.

So your states must be honest:

| Status | Who tells you | Meaning |
|---|---|---|
| `queued` | you | It's in the queue |
| `sent` | provider's synchronous HTTP response | **The provider accepted it.** This is the last thing you control |
| `delivered` | provider's **async webhook**, seconds to hours later | It reached the device / inbox |
| `opened` | your app SDK / an email tracking pixel | The human actually saw it |
| `failed` | webhook or sync response | It bounced |

**Therefore you need a webhook receiver**, because providers report delivery *back to you*, asynchronously. And here is the trap:

> **Provider webhooks are themselves at-least-once.** Twilio and SendGrid retry a webhook if your endpoint is slow or returns a non-2xx. **You will receive the same delivery receipt multiple times.** So the webhook handler must *also* be idempotent — and it must handle **out-of-order** events (you can receive `delivered` *before* your own worker has finished writing `sent`).

The fix for out-of-order is a **monotonic status ladder**: never move a notification *backwards* through the lifecycle.

```js
// webhookHandler.js
const RANK = { queued: 0, sent: 1, delivered: 2, opened: 3, failed: 3 };

export async function handleDeliveryWebhook(db, redis, event) {
  // 1. Idempotency: providers retry webhooks. Same event, same key → no-op.
  const eventKey = `webhook:${event.provider}:${event.eventId}`;
  const isNew = await redis.set(eventKey, '1', 'NX', 'EX', 7 * 24 * 3600);
  if (isNew !== 'OK') return { ok: true, deduped: true };

  // 2. Monotonic update: a late 'sent' must never overwrite a 'delivered'.
  //    The DB does the ordering check for us, atomically.
  await db.query(
    `UPDATE notifications
        SET status = $1, updated_at = now()
      WHERE provider_msg_id = $2
        AND $3 > (CASE status
                    WHEN 'queued'    THEN 0 WHEN 'sent'   THEN 1
                    WHEN 'delivered' THEN 2 ELSE 3 END)`,
    [event.status, event.providerMsgId, RANK[event.status]]
  );

  // 3. A dead token is money you're burning. Kill it now.
  if (event.status === 'failed' && event.reason === 'unregistered') {
    await db.query(
      `UPDATE device_tokens SET is_active = false WHERE token = $1`,
      [event.deviceToken]
    );
  }
  return { ok: true };
}
```

### R6. The worker — dedupe + retry + circuit breaker, together

This is the code the interviewer wants to see. Everything above, in one file.

```js
// worker.js — one channel's worker. Run N of these per channel.
import { claimNotification } from './dedupe.js';
import { backoffMs } from './backoff.js';

// ── Circuit breaker (recall topic 73) ──────────────────────────────────────
class CircuitBreaker {
  constructor(name, { threshold = 5, cooldownMs = 30_000 } = {}) {
    this.name = name;
    this.threshold = threshold;      // consecutive failures before we trip
    this.cooldownMs = cooldownMs;
    this.state = 'CLOSED';
    this.failures = 0;
    this.openedAt = 0;
  }

  canAttempt() {
    if (this.state === 'CLOSED') return true;
    if (this.state === 'OPEN' && Date.now() - this.openedAt > this.cooldownMs) {
      this.state = 'HALF_OPEN';      // let exactly one probe through
      return true;
    }
    return this.state === 'HALF_OPEN';
  }

  onSuccess() { this.state = 'CLOSED'; this.failures = 0; }

  onFailure() {
    this.failures++;
    if (this.state === 'HALF_OPEN' || this.failures >= this.threshold) {
      this.state = 'OPEN';
      this.openedAt = Date.now();
      metrics.increment('breaker.opened', { provider: this.name });
    }
  }
}

class ProviderDownError extends Error {}
class PermanentError extends Error {}   // never retry these

// ── The worker ─────────────────────────────────────────────────────────────
export class NotificationWorker {
  /**
   * @param providers  ordered list: [primary, backup]. Failover walks this list.
   */
  constructor({ queue, dlq, redis, db, providers, maxAttempts = 5 }) {
    this.queue = queue;
    this.dlq = dlq;
    this.redis = redis;
    this.db = db;
    this.providers = providers;
    this.maxAttempts = maxAttempts;
    this.breakers = new Map(
      providers.map((p) => [p.name, new CircuitBreaker(p.name)])
    );
  }

  async run() {
    // Drain HIGH priority first; give LOW a slice so it never starves.
    while (true) {
      const msg = (await this.queue.receive('high')) ?? (await this.queue.receive('low'));
      if (!msg) { await sleep(200); continue; }
      try {
        await this.process(msg);
        await this.queue.ack(msg);          // ACK only after success/terminal state
      } catch (err) {
        // Do NOT ack → the queue redelivers. That is the at-least-once guarantee
        // working; our dedupe guard is what makes the redelivery safe.
        logger.error({ err, id: msg.body.notificationId }, 'worker failure');
      }
    }
  }

  async process(msg) {
    const n = msg.body;   // { notificationId, userId, channel, rendered, token, priority }

    // 1. IDEMPOTENCY GUARD — before any side effect. Atomic; only one worker wins.
    const claimed = await claimNotification(this.redis, n.notificationId);
    if (!claimed) {
      metrics.increment('notification.duplicate_dropped');
      return;   // Already sent by someone. Silently succeed — this is the whole point.
    }

    // 2. Try providers in order (primary → backup), each guarded by a breaker.
    for (const provider of this.providers) {
      const breaker = this.breakers.get(provider.name);

      if (!breaker.canAttempt()) {
        // Provider is OPEN: fail FAST, don't hammer it, move to the backup.
        metrics.increment('provider.short_circuited', { provider: provider.name });
        continue;
      }

      try {
        const result = await this.sendWithRetries(provider, n);
        breaker.onSuccess();
        await this.markSent(n, provider.name, result.providerMsgId);
        return;
      } catch (err) {
        if (err instanceof PermanentError) break;    // backup won't help a bad payload
        breaker.onFailure();
        logger.warn({ provider: provider.name }, 'provider failed, trying backup');
      }
    }

    // 3. Every provider failed (or the payload is permanently bad) → DLQ + alert.
    //    Release the dedupe claim so a manual DLQ replay is actually able to send.
    await this.redis.del(`dedupe:${n.notificationId}`);
    await this.dlq.send({ ...n, failedAt: new Date().toISOString() });
    await this.db.query(
      `UPDATE notifications SET status='failed', attempts=attempts+1 WHERE id=$1`,
      [n.notificationId]
    );
    metrics.increment('notification.dead_lettered', { channel: n.channel });
  }

  async sendWithRetries(provider, n) {
    let lastErr;
    for (let attempt = 0; attempt < this.maxAttempts; attempt++) {
      try {
        return await provider.send(n);         // → { providerMsgId }
      } catch (err) {
        lastErr = err;
        if (err instanceof PermanentError) throw err;   // 400/401 → never retry
        if (attempt === this.maxAttempts - 1) break;
        const delay = backoffMs(attempt);      // 1s, 2s, 4s… with ±30% jitter
        metrics.increment('provider.retry', { provider: provider.name });
        await sleep(delay);
      }
    }
    throw new ProviderDownError(`${provider.name}: ${lastErr?.message}`);
  }

  async markSent(n, providerName, providerMsgId) {
    // 'sent' = the PROVIDER ACCEPTED IT. Not "the user saw it."
    // 'delivered' comes later, via webhook.
    await this.db.query(
      `UPDATE notifications
          SET status='sent', sent_at=now(), attempts=attempts+1,
              provider=$2, provider_msg_id=$3
        WHERE id=$1`,
      [n.notificationId, providerName, providerMsgId]
    );
    metrics.timing('notification.latency_ms', Date.now() - n.enqueuedAt);
  }
}

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
```

Trace the order once more, because the order is the design: **claim (dedupe) → breaker check → send with backoff → failover → DLQ.** Move dedupe after the send and you double-send. Move the breaker check inside the retry loop and you hammer a dead provider. Skip the DLQ and you lose messages forever.

---

## Deep dives

### D1. The fanout / blast problem

Marketing clicks **Send to all 10M users**. The naive implementation:

```js
// ❌ BAD — this request will time out, and probably OOM first.
app.post('/v1/campaigns', async (req, res) => {
  const users = await db.query('SELECT id FROM users WHERE segment = $1', [req.body.segment]);
  for (const user of users) {          // 10,000,000 iterations
    await queue.send(buildMessage(user, req.body));
  }
  res.json({ ok: true });
});
```

Ten million sequential enqueues at ~1ms each is **~3 hours** in a single HTTP request. The load balancer kills it at 60 seconds. If it retries, you've now got a partial double-send. And you tried to hold 10M user rows in memory.

**The fix: enqueue ONE job, not ten million.**

```js
// ✅ GOOD — the API does O(1) work. A fanout worker does the O(N) work, in batches.
app.post('/v1/campaigns', async (req, res) => {
  const campaignId = ulid();
  await db.query(
    `INSERT INTO campaigns (id, template_id, segment_id, status) VALUES ($1,$2,$3,'pending')`,
    [campaignId, req.body.templateId, req.body.segmentId]
  );
  await fanoutQueue.send({ campaignId });   // ONE message
  res.status(202).json({ campaignId });     // returns in ~10ms
});

// fanoutWorker.js — expands the campaign into per-user messages, in chunks.
export async function expandCampaign({ campaignId }) {
  const campaign = await loadCampaign(campaignId);
  const BATCH = 1000;
  let cursor = campaign.lastCursor ?? '0';   // resumable! (see below)

  for await (const batch of streamUsersBySegment(campaign.segmentId, cursor, BATCH)) {
    const messages = [];
    for (const user of batch) {
      // Preference check happens HERE, in bulk — not one Postgres query per user.
      if (!(await prefs.allows(user.id, 'marketing', campaign.channel))) continue;
      messages.push({
        notificationId: ulid(),              // deterministic dedupe key per user
        userId: user.id,
        campaignId,
        priority: 'low',                     // promo → LOW queue, never blocks OTPs
        ...campaign.payload,
      });
    }
    await queue.sendBatch(`${campaign.channel}.low`, messages);   // batched enqueue
    await saveCursor(campaignId, batch.at(-1).id);                // checkpoint
  }
  await markCampaignComplete(campaignId);
}
```

Three details that matter:

- **Batch the enqueue.** SQS `SendMessageBatch` takes 10 at a time; Kafka producers batch natively. One network call per 1,000 users, not per user.
- **Checkpoint the cursor.** If the fanout worker dies at user 6,000,000, it resumes from there — it does not restart from zero and re-send to 6M people. (And the per-user dedupe key means even if it *did* overlap, nobody gets two copies.)
- **Shard the fanout.** For real scale, split the segment into 100 ranges and run 100 fanout workers in parallel.

### D2. Rate limiting per user

A bug in the order service fires the same webhook 500 times. Without a guard, one user gets **500 pushes in a minute**. This is a real incident that has happened at many companies, and it is unrecoverable — you cannot un-send a push.

**Recall from [70 — Rate Limiting]:** a token bucket per key, in Redis.

```js
// perUserLimit.js — max 10 notifications per user per hour (promotional).
export async function allowUser(redis, userId, type) {
  if (type === 'transactional') return true;   // never throttle an OTP
  const key = `rl:user:${userId}:${new Date().getUTCHours()}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 3600);
  if (count > 10) {
    metrics.increment('notification.user_rate_limited');
    return false;    // suppress, and log it loudly — this usually means a caller has a bug
  }
  return true;
}
```

Note the carve-out: **you throttle promotional traffic, not transactional.** Rate-limiting someone's OTP is worse than the spam you were trying to prevent. But *do* alert when the limiter fires a lot — a spike in `user_rate_limited` is almost always a caller with a loop bug, and you want to know before the user does.

### D3. Rate limiting **toward the provider**

The providers rate-limit *you*. Your Twilio account might be capped at 100 messages/sec. Blow past it and you get 429s — which you'll then *retry*, generating more 429s. You have built a feedback loop that makes things worse.

So the workers must **self-throttle**: a shared, distributed token bucket in Redis, refilled at the provider's allowed rate, that every worker must take a token from before calling out.

```js
// providerLimit.js — distributed token bucket. All workers share one bucket.
export async function acquireProviderToken(redis, provider, ratePerSec) {
  const key = `rl:provider:${provider}:${Math.floor(Date.now() / 1000)}`;
  const used = await redis.incr(key);
  if (used === 1) await redis.expire(key, 2);
  return used <= ratePerSec;   // false → sleep and retry; do NOT call the provider
}
```

If the token is denied, the worker sleeps briefly and retries — the message stays in the queue, which is exactly what the queue is for. **Your throughput is now bounded by the provider's quota instead of by the provider's patience.** And always honor a `Retry-After` header when you do get a 429; that's the provider telling you exactly how long to wait.

### D4. Scheduling and time zones

"Send the daily digest at 9am" means **9am in the user's local time** — not 9am UTC. A user in Tokyo and a user in São Paulo must get it 12 hours apart.

The naive approach (one cron job that scans all 10M users at every tick) is a disaster: it hammers the DB and re-scans 10M rows 24 times a day.

**Better:** bucket users by timezone offset. There are ~38 distinct UTC offsets in use. Run **one fanout job per timezone bucket**, each firing when it's 9am in that zone:

```
00:00 UTC → fanout for users where tz_offset = +09:00  (it's 9am in Tokyo)
03:30 UTC → fanout for users where tz_offset = +05:30  (it's 9am in India)
08:00 UTC → fanout for users where tz_offset = +01:00  (it's 9am in Berlin)
14:00 UTC → fanout for users where tz_offset = -05:00  (it's 9am in New York)
```

Index `user_preferences` on `timezone` and each job touches ~1/38th of your users. For arbitrary one-off `scheduled_at` times, use a **delayed queue** (SQS delay, or a Redis sorted set scored by the send timestamp that a poller pops from every second) rather than polling `SELECT * FROM notifications WHERE scheduled_at < now()` across a huge table.

**Quiet hours** ride on the same machinery: if it's 2am for the user and the message is promotional, don't drop it — **re-schedule it for 8am**. If it's transactional (fraud alert), send it anyway. Quiet hours never apply to security notifications.

### D5. Priority handling

Priority is not a field you sort by — sorting a queue with 10 million messages in it is not a thing you can do. **Priority is a separate queue.**

```
push.high  ← workers drain this FIRST, always
push.low   ← workers touch this only when .high is empty
```

The starvation guard: dedicate a slice of capacity to `low` (e.g. every 10th poll goes to `low` regardless), or run a small dedicated `low` worker pool. Otherwise a sustained flood of high-priority traffic means the marketing campaign never sends at all — and marketing will find you.

**The concrete win:** during a 10M-message campaign sitting in `push.low`, a user requests a password reset. It lands in `push.high` and goes out in **under a second**, because the high-priority workers were never touching the campaign backlog at all. Draw this on the whiteboard — it's the payoff of the whole queue-per-channel-per-priority design.

---

## Real world examples

### Uber

Uber's notification volume is enormous and almost entirely time-critical — "your driver has arrived" is worthless 90 seconds late. Their engineering blog describes a system built around **Kafka** for durable buffering, per-channel consumer groups, and heavy investment in deduplication, precisely because a rider receiving the same "driver arrived" push three times is a support ticket. The transactional/promotional split is stark for them: a surge-pricing promo can wait; a trip notification cannot.

### Airbnb

Airbnb has publicly discussed a unified notification platform where product teams don't call APNs or SendGrid directly — they publish an event, and a central platform handles channel selection, user preferences, templating, and delivery. This is the **"post office"** model in its purest form: the value is that 200 engineers don't each independently reimplement retry logic and each independently forget to check the opt-out table.

### Stripe / payment providers (representative)

Any payments company faces the strictest version of the no-duplicates problem: "your card was charged $4,200" sent twice generates a panicked support call and possibly a chargeback. The standard industry pattern — and the one you should describe — is an idempotency key minted at the API boundary, carried through the queue, and checked atomically before the provider call. That is exactly the `claimNotification` function above.

*(Where a specific company's internals aren't public, treat the above as the representative architecture the industry converged on — the patterns are universal even where the implementation details differ.)*

---

## Trade-offs

| Decision | You gain | You give up |
|---|---|---|
| **Queue between API and providers** | Burst absorption (30x spike survivable), decoupling, retries, durability | End-to-end latency (queue hop adds 100ms–seconds), operational complexity, you can't tell the caller "it was delivered" |
| **Queue per channel** | No head-of-line blocking, independent scaling, per-channel policies | More queues to monitor, more infra, more config |
| **Priority queues** | OTPs beat marketing blasts | Starvation risk for low priority; more moving parts |
| **At-least-once + idempotency** | Never lose a transactional message | You MUST build dedupe (Redis dependency on the hot path) |
| **At-most-once instead** | Trivially no duplicates | You silently lose messages. **Unacceptable for transactional** |
| **Circuit breaker per provider** | One vendor outage ≠ your outage; fail fast | Tuning is genuinely hard (too twitchy = you fail over on a blip; too lax = it never trips) |
| **Multi-provider failover** | Survives a full vendor outage | Two integrations, two bills, two webhook formats, two sets of quirks. Real ongoing cost |
| **Store template_id + variables** | Fix a typo once; localization is free; small rows | An extra render step; you can't trivially answer "what exact text did we send in March?" (mitigate by versioning templates, never mutating them) |
| **Cache user preferences in Redis** | Preference check stays fast at 17k/s | Staleness — an opt-out may take up to the TTL to take effect. **Mitigate with write-through invalidation**, because this one has legal consequences |

**Rule of thumb:** **Transactional = reliability at any cost (retry hard, failover, never drop, page on DLQ). Promotional = throughput at low cost (low priority, aggressively rate-limited, droppable, and honestly, if 0.1% of a marketing blast fails, nobody notices).** Almost every decision in this system falls out of correctly classifying the message.

---

## Common interview questions on this topic

### Q1: "A user complains they got the same 'payment received' push five times. Walk me through how that happened and how you'd fix it."

**Hint:** Name the root cause precisely: the queue is **at-least-once**, so a worker that crashes *after* calling the provider but *before* ACKing the message will have that message redelivered and re-sent. It is not a bug in the queue — it is the queue's contract. The fix is an **idempotency guard keyed on `notification_id`**, minted at the API and checked *atomically* (Redis `SET NX`, or a conditional `UPDATE ... WHERE status='queued'`) **before** the provider call. Then mention the second source: provider webhooks are also at-least-once, so the *status-update* path needs dedupe too. Bonus: mention that a naive `SELECT status; if (...) send()` is a read-then-write race that does not fix it.

### Q2: "Why not just call APNs directly from the order service and skip all this?"

**Hint:** Three reasons, and lead with the numbers. (1) **The 30x burst** — 17,000/s during a campaign would either melt your service or get you rate-limited by the provider; a queue does load levelling. (2) **Coupling** — if SendGrid is slow, your checkout is now slow, because you made a synchronous call to a third party inside a user-facing request. (3) **Cross-cutting logic** — opt-out checks, retries, templating, tracking would be reimplemented (and half-forgotten) in all 40 services. The opt-out one is a *legal* argument, which usually ends the discussion.

### Q3: "Twilio goes down for an hour. What happens to your system?"

**Hint:** Walk the layers. Workers start seeing timeouts → the **circuit breaker for Twilio trips OPEN** after the failure threshold → workers stop calling it and start failing fast in microseconds instead of blocking for 30s each → messages **fail over to the backup provider (AWS SNS)** and keep flowing. If there's no backup, they retry with exponential backoff and the SMS queue *grows* — which is fine, that's what the queue is for, and it's a dashboard number rather than lost data. The breaker moves to HALF_OPEN after the cooldown, probes with one call, and closes when Twilio recovers. **Crucially: push and email are completely unaffected, because they're on different queues with different workers.** That's the payoff of the bulkhead.

### Q4: "How do you send 10 million marketing notifications without falling over?"

**Hint:** Do **not** enqueue 10M messages from the HTTP request — it will time out and it will OOM. Enqueue **one campaign job**; a **fanout worker** streams users from the segment in batches of ~1,000, checks preferences in bulk, and does **batched enqueues** into the **low-priority** queue. Checkpoint the cursor so a worker crash resumes rather than restarts. Shard the fanout across N workers by user-ID range. Because it's on the low-priority queue, it drains at whatever rate the provider quota allows and **never blocks transactional traffic.**

### Q5: "Your dashboard says 99.9% 'delivered'. Is that true?"

**Hint:** This is a trap, and the right answer is to interrogate the word. If "delivered" means "the provider returned 200," then it means **the provider accepted it** — not that any human saw it. True delivery is only knowable from the provider's **asynchronous webhook**, which arrives seconds to hours later. Also: a push to an **expired device token** may be cheerfully accepted and then silently dropped, so if you aren't pruning dead tokens, your delivery number is inflated by phones that no longer exist. Separate the states — `sent` / `delivered` / `opened` — and be honest about which one you're actually reporting.

---

## Practice exercise

### Build the worker and prove the duplicate is impossible

**~35 minutes. Produce running Node code, not a diagram.**

1. Write a `FakeQueue` class with `send(msg)`, `receive()`, and `ack(msg)` — and make it **deliberately faulty**: `receive()` returns a message but does *not* remove it until `ack()` is called, and if the message isn't ACKed within 2 seconds, it becomes available again. (This is exactly SQS's visibility-timeout model, and it is the source of every duplicate.)

2. Write a `FakeProvider` with a `send(n)` method that:
   - Throws a transient error 30% of the time.
   - Otherwise pushes the `notificationId` onto a module-level `deliveredMessages` array and returns `{ providerMsgId }`.

3. Wire up the `NotificationWorker` from this doc against them, using an in-memory `Map` in place of Redis for the dedupe claim.

4. Now the experiment. Enqueue **100 notifications**. Run **3 workers concurrently**. Kill a worker mid-flight (make one worker `process.exit`-style abort *after* calling the provider but *before* ACKing — this is the crash that causes redelivery).

5. **Assert `deliveredMessages.length === 100` and that there are zero duplicate IDs.**

6. Then **delete the `claimNotification` call** and run it again. Watch the duplicates appear. Count them.

Step 6 is the entire lesson. Once you have *seen* the duplicates show up the moment you remove the guard, you will never again forget why idempotency and at-least-once are a package deal.

**Stretch:** add a circuit breaker and make `FakeProvider` fail 100% of the time for 5 seconds. Verify the breaker opens, that calls fail in microseconds instead of blocking, and that traffic fails over to a `BackupProvider`.

---

## Quick reference cheat sheet

- **The spike is the whole design.** ~600/s average, ~17,000/s during a 10M blast = a **30x burst**. That burst is why the queue exists (load levelling, [67 — Message Queues]).
- **Queue per CHANNEL, not one shared queue.** Different throughput, different providers, different failures — and a backed-up email provider must never block an urgent push (**head-of-line blocking**).
- **Priority queues within each channel.** `push.high` (OTP) drains before `push.low` (marketing). Add a starvation guard for `low`.
- **At-least-once is a guarantee, not a bug.** The queue *will* redeliver. Your worker *will* see the same message twice.
- **Therefore: idempotency, always.** Dedupe on `notification_id` with an **atomic** claim (Redis `SET NX`) **before** the provider call. Not after. Not with a `SELECT`-then-`if`.
- **Retry transient, never retry permanent.** 5xx/429/timeout → exponential backoff **with jitter**. 400/401/dead-token → straight to the DLQ.
- **Jitter prevents the thundering herd** — without it, 5,000 failed messages all retry at the same millisecond and re-kill the provider on recovery.
- **DLQ + alert on DLQ depth.** A growing DLQ is the highest-signal alert in the system: it literally means "users are not getting messages."
- **Circuit breaker per provider** ([73 — Circuit Breaker]): CLOSED → OPEN → HALF_OPEN. Fail *fast*, don't fail *slow*. Then **fail over to a backup provider** (Twilio → SNS, SendGrid → SES).
- **`sent` ≠ `delivered`.** A 200 from the provider means *the provider accepted it*. Real delivery arrives later via an **async webhook** — and **those webhooks are also at-least-once, so dedupe them too** (and never move a status backwards).
- **Store `template_id` + variables, never rendered text.** Fix a typo once instead of in 40M rows. Localization comes free.
- **Prune dead device tokens.** Users reinstall apps. Sending to an expired token costs money and silently inflates your delivery metrics.
- **Never fan out 10M enqueues from an HTTP request.** Enqueue ONE campaign job; a fanout worker expands it in checkpointed batches.
- **Rate-limit per user** (a buggy caller will spam someone 500 times — but **never throttle an OTP**) **and toward the provider** (they enforce quotas, and retrying a 429 makes it worse).
- **Alert on:** queue depth, DLQ growth rate, provider error rate, circuit-breaker state changes, and **transactional** p99 end-to-end latency.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [67 — Message Queues](./67-message-queues.md) — the load-levelling queue is the load-bearing wall of this entire design; read it first if the 30x-spike argument didn't land |
| **Next** | [125 — LLD: Notification Service](./125-lld-notification-service.md) — zoom in from the boxes-and-arrows to the actual classes: channel strategies, template renderers, the worker's object model |
| **Related** | [85 — Idempotency](./85-idempotency.md) — the single most important guarantee here; at-least-once delivery makes duplicates inevitable unless you dedupe on `notification_id` |
| **Related** | [73 — Circuit Breaker](./73-circuit-breaker.md) — how you survive Twilio having a bad hour without taking your own system down with it |
| **Related** | [70 — Rate Limiting](./70-rate-limiting.md) — you need it in two directions: per-user (stop a buggy caller spamming) and per-provider (respect their quota) |
