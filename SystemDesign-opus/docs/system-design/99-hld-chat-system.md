# 99 — Design WhatsApp / Real-Time Chat
## Category: HLD Case Study

---

## What is this?

A chat system is a machine for moving small pieces of text from one person's phone to another person's phone **fast, in order, and without ever losing one.** WhatsApp, iMessage, Slack, Discord, Instagram DMs — under the hood they all solve the same three problems: hold millions of open connections, find the right connection for the person you're messaging, and never lose a message even if a server catches fire mid-send.

Think of it as a **post office where every customer is standing at the counter all day long, holding the clerk's hand.** Normal web systems are like mail: you drop a request, the server responds, the connection closes. Chat is different — the connection stays open, the server can talk to *you* whenever it wants, and it has to remember which clerk is holding whose hand.

---

## Why does it matter?

Chat is one of the two or three most-asked system design questions at every large company, and it's asked *because* it breaks all your normal instincts:

- Your normal instinct is **stateless servers behind a load balancer**. That doesn't work here. A chat server is **stateful** — it physically owns the TCP socket to a specific user. If a message for Bob arrives at a server that isn't holding Bob's socket, that message goes nowhere.
- Your normal instinct is **request/response over HTTP**. That doesn't work here either. The server needs to *push* to the client, unprompted.
- Your normal instinct is **a relational database with an auto-increment primary key**. That doesn't work at 2 billion writes a day across a sharded cluster.

**At work**, you'll hit these same walls the moment you build anything real-time: live notifications, collaborative editing, multiplayer state, live dashboards, order tracking. Chat is the canonical version of that whole family.

**In the interview**, the signal the interviewer is hunting for is one specific thing: *do you realize that message routing between stateful servers is the hard part?* Candidates who draw "client → load balancer → chat service → database" and stop have failed. Candidates who say "server 1 holds Alice's socket, server 47 holds Bob's socket, so how does server 1 reach server 47?" have passed the first gate.

---

## The core idea — explained simply

### The Hotel Switchboard Analogy

Imagine a hotel with 10 million guests (yes, big hotel) and 200 switchboard operators.

Every guest who is **awake and in their room** picks up their phone and keeps the line open to one specific operator. Operator #1 is holding the line for Alice, Ivan, and 49,998 others. Operator #47 is holding the line for Bob and 49,999 others.

Now Alice wants to say "hey" to Bob.

1. Alice speaks into her already-open line. **Operator #1** hears it.
2. Operator #1 has no idea where Bob is. She's holding 50,000 lines, and Bob isn't one of them.
3. So she looks at **the hotel's central guest register** — a big whiteboard in the lobby that says `Bob → Operator #47`.
4. She calls Operator #47 on the internal staff line and says "message for Bob: 'hey'".
5. Operator #47 speaks it down the line she's already holding for Bob. Bob hears it instantly.

And critically: **before Operator #1 does any of that, she writes the message down in the hotel's permanent logbook.** If the whole switchboard catches fire, the message survives. That logbook is your database, and writing to it *before* telling Alice "sent ✓" is the entire durability guarantee.

If Bob is **asleep / out of the hotel** (his phone backgrounded the app, the socket is closed), the guest register says `Bob → offline`. So Operator #1 leaves the message in the logbook and asks the front desk to **slip a note under Bob's door** — that's your push notification. When Bob wakes up and picks up his phone again, he says "the last thing I heard was message #412, what did I miss?" and the operator reads him everything after #412.

That's the whole system. Everything else is detail.

| Hotel thing | System design thing |
|---|---|
| Guest with an open phone line | Client holding a **WebSocket** |
| Switchboard operator | **Chat server** (stateful — owns the sockets) |
| Central guest register (whiteboard) | **Session / service-discovery store** (Redis: `user_id → server_id`) |
| Internal staff line between operators | **Message routing** (direct RPC, or a pub/sub bus) |
| Permanent logbook | **Message store** (Cassandra) — written to *before* the ack |
| Note slipped under the door | **Push notification** (APNs / FCM) |
| "The last thing I heard was #412" | **Reconnect + gap sync** by last-seen message ID |
| Front desk knows who's in the building | **Presence service** (Redis key with a TTL, refreshed by heartbeat) |

---

## Key concepts inside this topic

We'll run the **5-step framework from [93 — How to Attack an HLD Interview](./93-hld-approach-framework.md)**: requirements → estimation → API → data model → architecture, then deep dives.

---

### 1. Requirements

**Clarifying questions to ask the interviewer first** (ask these out loud — it's scored):

- "Is this 1:1 only, or group chat too? If groups, what's the max size?" → *Assume both; cap groups at 100.*
- "Do we need media (photos, video), or text only?" → *Assume media, but design it so bytes never cross the chat socket.*
- "Do we need end-to-end encryption?" → *Discuss it conceptually; it changes what the server can do.*
- "Do we store full message history forever, or does it expire?" → *Assume we store it.*
- "Mobile, web, or both?" → *Both — and mobile is what forces the push-notification path.*

**Functional requirements**

| # | Requirement |
|---|---|
| F1 | 1:1 messaging between two users |
| F2 | Group chat, capped at **100 members** |
| F3 | Online / offline **presence** ("last seen 3m ago") |
| F4 | **Delivery receipts**: sent (✓), delivered (✓✓), read (✓✓ blue) |
| F5 | **Push notification** when the recipient is offline |
| F6 | **Media attachments** (images, video, files) |
| F7 | **Message history** — scroll back through a conversation |

**Non-functional requirements** — these are the ones that actually drive the design:

| # | Requirement | What it forces |
|---|---|---|
| N1 | **Very low latency** (p99 end-to-end < 300ms) | Persistent connections, not polling. In-memory routing. |
| N2 | **Durability — messages must NEVER be lost** | Persist *before* acking. Replicated store. |
| N3 | **In-order delivery within a conversation** | Per-conversation sequence numbers, not wall-clock timestamps. |
| N4 | **High availability** | Any chat server can die; clients reconnect elsewhere. |
| N5 | **Hundreds of millions of concurrent connections** | This is the one that breaks everything. See below. |

Note what is **not** required: strong global consistency, complex joins, analytics queries. That absence is a gift — it's what lets you pick a non-relational store later.

---

### 2. Capacity estimation

**Traffic**

```
DAU                          = 50,000,000
Messages per user per day    = 40
Messages per day             = 50M × 40 = 2,000,000,000  (2B)

Seconds in a day             = 86,400  (round to 100,000 for mental math)

Average writes/sec           = 2B / 100,000  ≈ 20,000 msg/s
                               (exact: 2B / 86,400 ≈ 23,000 msg/s)

Peak                         ≈ 2× average    ≈ 50,000 msg/s
```

Fine. 50k writes/sec is a lot but not scary — that's a Kafka-sized number, or a modest Cassandra ring.

**Now the number that actually designs the system: CONNECTIONS.**

```
Concurrently online users     = 10,000,000   (20% of DAU online at once)
Each online user holds        = 1 WebSocket

Connections a single box can hold:
  A tuned Linux box (epoll, raised ulimit, ~10KB kernel + app memory per idle socket)
  realistically holds  50,000 – 100,000  concurrent WebSockets.
  (The "C10K problem" was solved long ago; C1M on one box is a stunt, not production.)

Chat servers needed = 10,000,000 / 75,000  ≈ 133 servers
                    → call it 150–200 servers, with headroom for failure
```

**Stop and look at that.** You need ~150 servers **just to hold the sockets open** — before a single message is sent. Memory, not CPU, is the binding constraint. Each server is doing almost nothing most of the time; it's just *remembering* 75,000 people.

This is why the design is what it is. If connections were cheap and stateless, you'd use a normal LB and a normal service. They aren't, so you can't.

**Storage**

```
Bytes per message row  ≈ 200 bytes
  (message_id 16B + conversation_id 16B + sender_id 8B + timestamp 8B
   + body ~100B + status/flags ~20B + column overhead)

Per day    = 2B × 200B          = 400 GB/day
Per year   = 400 GB × 365       ≈ 146 TB/year
Over 5 yr  = 146 TB × 5         ≈ 730 TB
With RF=3 (3 replicas)          ≈ 2.2 PB of raw disk
```

Media is separate and much bigger — but media goes to **blob storage (S3) + CDN**, not into the message table. The message table only stores a URL (~100 bytes). Say 10% of messages have media at 300KB average:

```
Media/day = 2B × 10% × 300KB = 60 TB/day   ← this alone justifies S3 + CDN
```

**Bandwidth through the chat tier**

```
Ingress = 50,000 msg/s × 200 B ≈ 10 MB/s   ← trivially small!
```

Ten megabytes a second. The *data* is nothing. The *connections* are everything. Say that sentence in the interview.

---

### 3. The protocol decision — and why WebSocket

Recall from **[82 — Content Negotiation and Protocols](./82-content-negotiation-and-protocols.md)** that HTTP is fundamentally client-initiated: the server cannot speak unless spoken to. Chat needs the opposite. Here are your four options:

| Option | How it works | Latency | Server cost | Verdict for chat |
|---|---|---|---|---|
| **Short polling** | Client asks "anything new?" every 2s | 0–2s of lag | Brutal — 10M users × 1 req/2s = **5M req/s** of mostly-empty responses | ❌ Wasteful and laggy |
| **Long polling** | Client asks; server *holds* the request open until there's news, then responds; client immediately re-asks | Near-real-time | Better, but every message costs a full HTTP request/response teardown + reconnect | ⚠️ Workable fallback, not a first choice |
| **SSE (Server-Sent Events)** | One long-lived HTTP stream, server → client only | Near-real-time | Cheap | ❌ **One-directional.** Fine for a news feed. Chat needs the client to send too. |
| **WebSocket** | HTTP request with `Upgrade: websocket` → the TCP connection becomes a **full-duplex** byte pipe. Either side can send a frame at any time. | ~single-digit ms | One socket + ~10KB per user, held open | ✅ **This.** |

**Commit: WebSocket.** The reasoning, in one line for the interviewer:

> "Chat is **bidirectional** and **high-frequency**, so I want a persistent **full-duplex** connection. WebSocket gives me that with a few bytes of frame overhead per message, versus long polling's full HTTP round-trip per message."

Keep **long polling as a fallback** for clients behind proxies that mangle the `Upgrade` handshake. That's a one-sentence mention that shows maturity.

#### The mobile reality — the thing candidates forget

**A phone cannot hold a WebSocket open while the app is backgrounded.** iOS suspends your process within seconds. Android doze-mode kills it. The OS will not let you keep a socket alive so you can play chat server on someone's battery.

So the socket **dies** every time the user swipes away from the app. Which means:

> **You need a second, completely different delivery path for offline/backgrounded users: push notifications (APNs on iOS, FCM on Android).**

This is a **dual-path design**, and it's non-negotiable:

```
Recipient ONLINE  (socket alive)  → push the message down the WebSocket
Recipient OFFLINE (socket gone)   → persist it + send a wake-up push via APNs/FCM
                                     → user taps → app reconnects → syncs the gap
```

Both paths write to the same message store. The push notification carries a *preview*, not the source of truth — the real message is fetched on reconnect. (See **[98 — HLD: Notification System](./98-hld-notification-system.md)** for how that fanout tier is built.)

---

### 4. API design

Two surfaces: a **WebSocket protocol** (the hot path) and a small **HTTP API** (the cold path — history, media, auth).

**WebSocket — client → server frames** (JSON over `wss://`):

```jsonc
// Send a message
{ "type": "SEND",      "clientMsgId": "uuid-abc",  // for idempotency/dedupe
  "conversationId": "conv_9f3", "body": "hey", "mediaUrl": null }

// Acknowledge that the client received a message (drives ✓✓)
{ "type": "DELIVERED_ACK", "messageIds": ["m_1021", "m_1022"] }

// The user actually looked at it (drives ✓✓ blue)
{ "type": "READ_ACK", "conversationId": "conv_9f3", "upToMessageId": "m_1022" }

// Keep-alive; also refreshes the presence TTL
{ "type": "HEARTBEAT" }
```

**WebSocket — server → client frames:**

```jsonc
{ "type": "MESSAGE",  "messageId": "m_1023", "conversationId": "conv_9f3",
  "senderId": "u_42", "body": "hey", "seq": 1023, "sentAt": 1752300000000 }

{ "type": "ACK",      "clientMsgId": "uuid-abc", "messageId": "m_1023", "seq": 1023 }
{ "type": "RECEIPT",  "messageId": "m_1023", "status": "DELIVERED" | "READ", "byUserId": "u_77" }
{ "type": "PRESENCE", "userId": "u_77", "status": "ONLINE" | "OFFLINE", "lastSeen": 1752299900000 }
```

**HTTP API** (everything that isn't the hot path):

```
POST   /v1/conversations                    → { participantIds: [...] } → { conversationId }
GET    /v1/conversations/:id/messages?before=<messageId>&limit=50
                                            → paginated history, newest-first
GET    /v1/sync?lastSeenMessageId=m_1002    → the gap the client missed while offline
POST   /v1/media/upload-url                 → { presignedS3Url, mediaUrl }  ← media NEVER goes over the socket
POST   /v1/groups/:id/members               → add member (server enforces the 100 cap)
GET    /v1/presence?userIds=u_1,u_2,u_3     → batch presence lookup
```

Note `POST /v1/media/upload-url`. **You never push image bytes through the chat WebSocket.** The client asks for a pre-signed S3 URL, uploads directly to S3, and then sends a *text message whose body is a URL*. The chat tier stays lightweight; the CDN does the heavy lifting.

---

### 5. Data model — and the database choice

This is the second place interviewers separate candidates. Look hard at the access pattern before you pick anything:

| Property of chat data | Consequence |
|---|---|
| Enormously **write-heavy** (every message is a write; reads are mostly "the last 50") | Need cheap, fast, append-friendly writes |
| Data is **time-series-like** — an append-only log per conversation | Sequential, clustered-by-time storage is ideal |
| The **only** query is: *"give me the last N messages for THIS conversation, newest first"* | You need one partition key and one sort key. That's it. |
| **No joins.** No "find all messages containing X across all users." | You are not using SQL's superpower, so you're paying for it for nothing |
| Volume: **~730 TB over 5 years** | Must shard horizontally from day one |

Recall from **[61 — SQL vs NoSQL](./61-databases-sql-vs-nosql.md)** that a relational DB earns its keep with joins, ad-hoc queries, and transactional integrity across tables. **Chat needs none of those.** What it needs is a store that eats writes for breakfast and reads a single partition sequentially.

> **Choose a wide-column store: Cassandra** (or ScyllaDB, or HBase — Facebook Messenger famously ran on HBase for years).
>
> Cassandra is a **log-structured** store: writes go to an in-memory memtable + an append-only commit log, then flush to disk sequentially. Writes are ~O(1) and never seek. That is exactly the shape of chat.

#### The partition key choice — the beautiful part

Recall from **[75 — Partitioning Strategies](./75-data-partitioning.md)** and **[64 — Database Sharding](./64-database-sharding.md)** that the partition key decides which physical node holds a row, and that **a query that stays inside one partition is enormously faster than one that fans out.**

So: partition by `conversation_id`, cluster (sort) by `message_id` descending.

```sql
-- Cassandra CQL
CREATE TABLE messages (
    conversation_id  uuid,        -- PARTITION KEY: all messages of one chat live together
    message_id       bigint,      -- CLUSTERING KEY: time-sortable (Snowflake-style)
    sender_id        bigint,
    body             text,        -- ciphertext if E2EE
    media_url        text,
    created_at       timestamp,
    PRIMARY KEY ((conversation_id), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

Why this is exactly right:

- **`WHERE conversation_id = ? LIMIT 50`** hits **one node**, reads **one contiguous run of disk**, and returns the newest 50 messages already in order. No scatter-gather, no merge, no sort. It is the cheapest read a distributed database can do.
- Because `message_id` is time-sortable and the clustering order is `DESC`, "the latest messages" are physically at the front of the partition.
- Pagination is trivial: `WHERE conversation_id = ? AND message_id < ? LIMIT 50`.
- Load spreads evenly, because there are hundreds of millions of conversations — no hot partition. (Exception: a giant group chat. That's precisely why you cap groups at 100.)

**The other tables:**

```sql
-- Which conversations does a user belong to? (For the inbox screen.)
CREATE TABLE user_conversations (
    user_id           bigint,
    last_message_ts   timestamp,   -- clustering key, DESC → inbox sorted by recency
    conversation_id   uuid,
    unread_count      int,
    PRIMARY KEY ((user_id), last_message_ts, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_ts DESC);

-- Group membership (small, bounded at 100)
CREATE TABLE conversation_members (
    conversation_id  uuid,
    user_id          bigint,
    joined_at        timestamp,
    role             text,
    PRIMARY KEY ((conversation_id), user_id)
);
```

**And the two things that live in Redis, not Cassandra:**

```
sess:{user_id}     → "chat-server-47"     (TTL 60s, refreshed by heartbeat)   ← service discovery
presence:{user_id} → "online"             (TTL 30s, refreshed by heartbeat)   ← presence
```

Both are ephemeral, tiny, and read on nearly every message. They belong in memory. `10M users × ~50 bytes ≈ 500 MB` — one Redis node could technically hold it; you'd run a small cluster for HA.

**Users, profiles, contacts, auth** — that's low-volume, relational, joinable data. Keep it in **PostgreSQL**. Polyglot persistence: use the right store per access pattern, not one store for everything.

---

### 6. High-level architecture

```
                         ┌──────────────┐   ┌──────────────┐
   MOBILE / WEB CLIENTS  │  Alice (iOS) │   │ Bob (Android)│
                         └──────┬───────┘   └──────┬───────┘
                                │ wss://           │ wss://
                                ▼                  ▼
                   ┌────────────────────────────────────────┐
                   │   L4 LOAD BALANCER (sticky / by conn)  │   ← routes NEW connections only
                   └───────┬──────────────────────┬─────────┘
                           │                      │
              ┌────────────▼──────────┐  ┌────────▼──────────────┐
              │  CHAT SERVER 1        │  │  CHAT SERVER 47       │   ...  ~150 of these
              │  ── STATEFUL ──       │  │  ── STATEFUL ──       │
              │  holds 75k sockets    │  │  holds 75k sockets    │
              │  { u_42: <socket> }   │  │  { u_77: <socket> }   │
              └──┬────┬────┬──────────┘  └──────────┬────┬───┬───┘
                 │    │    │                        │    │   │
      ┌──────────┘    │    └─────────┐    ┌─────────┘    │   └──────────┐
      │               │              │    │              │              │
      ▼               ▼              ▼    ▼              ▼              ▼
┌───────────┐  ┌─────────────┐  ┌───────────────┐  ┌──────────┐  ┌─────────────┐
│  SESSION  │  │   MESSAGE   │  │   MESSAGE     │  │ PRESENCE │  │ NOTIFICATION│
│  SERVICE  │  │    STORE    │  │   ROUTING     │  │ SERVICE  │  │   SERVICE   │
│  (Redis)  │  │ (Cassandra) │  │ (Redis pub/sub│  │ (Redis   │  │ (Kafka →    │
│           │  │             │  │  or Kafka)    │  │  + TTL)  │  │  APNs/FCM)  │
│ user_id → │  │ partition by│  │               │  │          │  │             │
│ server_id │  │ conv_id     │  │ channel per   │  │ heartbeat│  │ only when   │
│           │  │ RF=3        │  │ chat server   │  │ every 20s│  │ recipient   │
└───────────┘  └─────────────┘  └───────────────┘  └──────────┘  │ is OFFLINE  │
                                                                  └──────┬──────┘
                                                                         ▼
                                                              ┌────────────────────┐
                                                              │  APNs  /  FCM      │
                                                              │  (Apple / Google)  │
                                                              └────────────────────┘

   MEDIA PATH (completely separate — bytes NEVER touch the chat socket):

   Client ──POST /media/upload-url──▶ Media Service ──▶ pre-signed S3 URL
   Client ──PUT bytes────────────────────────────────▶  S3 / Blob Storage
   Client ──SEND {mediaUrl}──────────▶ Chat Server   (just a ~100-byte string)
   Recipient ──GET mediaUrl──────────▶ CDN ──(cache miss)──▶ S3
```

**Reading the diagram:**

- The **load balancer is only used for the initial handshake.** Once the WebSocket is established, it's a long-lived TCP connection pinned to one chat server. The LB does not, and cannot, re-route individual messages. Say this out loud — it's the moment the interviewer realizes you understand statefulness.
- **Chat servers are stateful.** Server 1 has an in-memory `Map<userId, WebSocket>`. That map is the whole point of the server. It cannot be replicated, cannot be shared, and vanishes if the process dies.
- The **Session Service** is the only reason cross-server delivery is possible: it's the global answer to "which box is holding this person's socket?"
- The **Notification Service** is fed *only* for offline recipients. It's an async, at-least-once, fire-and-forget path (see [67 — Message Queues](./67-message-queues.md)).

---

### 7. The message routing problem — the heart of the design

Alice is on **server 1**. Bob is on **server 47**. Alice sends "hey". Server 1 has never heard of Bob.

**How does the message get from server 1 to server 47?**

This is the question the interviewer is *actually* asking. There are two answers, and you should draw both.

#### Approach A — Session lookup + direct RPC

```
  Alice                Server 1              Redis            Server 47            Bob
    │                     │                    │                  │                 │
    │──SEND "hey"────────▶│                    │                  │                 │
    │                     │                    │                  │                 │
    │                     │ 1. persist to Cassandra (DURABILITY FIRST)              │
    │                     │────────────────────┼──▶ [message store]                 │
    │                     │                    │                  │                 │
    │                     │ 2. GET sess:bob    │                  │                 │
    │                     │───────────────────▶│                  │                 │
    │                     │◀───"server-47"─────│                  │                 │
    │                     │                    │                  │                 │
    │                     │ 3. gRPC deliver(bob, msg)             │                 │
    │                     │──────────────────────────────────────▶│                 │
    │                     │                    │                  │──ws.send()─────▶│
    │◀───ACK (✓ sent)─────│                    │                  │                 │
```

| Pros | Cons |
|---|---|
| **Lowest latency** — one Redis GET + one direct network hop | Every chat server must be able to reach **every other** chat server → an **N×N mesh** of connections (150 servers → 22,350 potential pairs) |
| Simple to reason about | **Stale session data**: if Bob reconnected to server 12 one second ago and Redis hasn't caught up, you RPC into the void |
| No extra infrastructure beyond Redis | Server 47 being down means server 1 must handle the failure inline |
| | Server 1 is now **tightly coupled** to the chat-server topology |

#### Approach B — Publish to a channel the target server subscribes to

Every chat server subscribes to its own channel. Server 1 doesn't call server 47 — it *publishes* and lets the bus do the routing.

```
  Alice            Server 1          Redis Pub/Sub (or Kafka)      Server 47          Bob
    │                 │                        │                       │               │
    │──SEND "hey"────▶│                        │                       │               │
    │                 │ 1. persist to Cassandra ───▶ [message store]   │               │
    │                 │                        │                       │               │
    │                 │ 2. GET sess:bob ──▶ "server-47"                │               │
    │                 │                        │                       │               │
    │                 │ 3. PUBLISH channel:server-47  { to: bob, msg } │               │
    │                 │───────────────────────▶│                       │               │
    │                 │                        │  (server 47 is already SUBSCRIBEd)    │
    │                 │                        │──────── deliver ─────▶│               │
    │                 │                        │                       │──ws.send()───▶│
    │◀──ACK (✓ sent)──│                        │                       │               │
```

| Pros | Cons |
|---|---|
| **Fully decoupled** — server 1 knows nothing about server 47's existence, IP, or health | One extra network hop → slightly higher latency (~1-3ms with Redis pub/sub) |
| No N×N mesh; every server has **one** connection to the bus | The bus is now a **critical dependency** and a potential bottleneck |
| Easy to add/remove chat servers — just subscribe/unsubscribe | Redis pub/sub is **fire-and-forget**: if server 47 is momentarily disconnected, the message is dropped from the bus |
| With **Kafka** instead of Redis: durable, replayable, ordered per partition | Kafka adds real latency (~10ms+) — often too slow for the hot path |

#### Which do you pick?

**Say this:** *"I'd start with Approach B using Redis pub/sub, because decoupling is worth ~2ms and it makes chat servers trivially add/removable. The pub/sub drop risk doesn't scare me, because **the message was already persisted to Cassandra before I published it.** If the push is lost, the recipient's client will pull it during its reconnect gap-sync anyway. The bus is a *fast path*, not the source of truth — Cassandra is."*

That sentence — **"the bus is an optimization, the database is the truth"** — is the single most valuable thing you can say in this interview.

(A production hybrid: use the bus for the fast path, and have a small **durable outbox** in Kafka for retries when a recipient is briefly unreachable. See [67 — Message Queues](./67-message-queues.md).)

---

### 8. Message IDs and ORDERING

Messages appearing out of order is a **visible, embarrassing bug**. "See you at 8" arriving before "Can we move dinner?" is a user-facing disaster. So how do you number messages?

**What you cannot use:**

| ❌ Bad idea | Why it fails |
|---|---|
| Database auto-increment | Your DB is **sharded** across dozens of nodes. There is no single counter. Making one would mean a global lock — an instant bottleneck at 50k writes/s. |
| **Client timestamps** | Phone clocks are wrong (drift, timezones, users manually setting the clock). And a malicious client can *lie* — send `timestamp: 0` and pin their message to the top of the conversation forever. **Never trust a client clock for ordering.** |
| Server wall-clock time | Better, but two servers' clocks differ by milliseconds. Two messages within the same millisecond tie. |
| Random UUIDs | Not sortable. You'd have to sort by a separate field anyway — and it destroys your Cassandra clustering key. |

**What works — two good options:**

**Option 1 — Per-conversation sequence number.** Keep a counter in Redis per conversation and `INCR` it:

```js
const seq = await redis.incr(`seq:${conversationId}`);  // atomic, monotonic, gap-free
```

- Simple, gap-free, and **strictly ordered within the conversation**.
- Redis `INCR` is atomic, so two servers writing to the same conversation concurrently get distinct, increasing numbers.
- Cost: a Redis round-trip on the write path (~0.5ms), and Redis must be durable enough not to reset the counter.

**Option 2 — Snowflake-style time-sortable IDs.** A 64-bit integer:

```
 ┌────────────────────────┬──────────────┬──────────────┐
 │  41 bits: timestamp ms │ 10b: node ID │ 12b: sequence│
 └────────────────────────┴──────────────┴──────────────┘
   ~69 years of range       1,024 nodes    4,096 IDs/ms/node
```

- Generated **locally on each chat server** — zero coordination, zero network hop.
- Sorts by time when compared as an integer → perfect as a Cassandra clustering key.
- Cost: relies on server clocks being roughly in sync (NTP), and ordering between two servers within the same millisecond is arbitrary.

#### The crucial simplification

> **Ordering only needs to be consistent WITHIN a conversation. Not globally.**

Nobody on Earth cares whether your message to your mother came "before" a stranger's message to *their* mother. There is no observer who can see both. Global ordering is a distributed-systems nightmare (it needs consensus); **per-conversation ordering is nearly free.**

State this explicitly in the interview. It shows you know where to *not* spend complexity. And notice how neatly it composes with your partition key: **the partition is the conversation, and the ordering domain is the conversation.** One decision, two problems solved.

**Handling duplicates:** the client attaches a `clientMsgId` (a UUID) to every send. If the network hiccups and the client retries, the server sees a `clientMsgId` it has already processed and returns the *original* `messageId` instead of writing a second row. That's **idempotency**, and it's what turns your at-least-once transport into effectively-once delivery.

---

### 9. Delivery guarantees and the ACK flow

Here is the whole lifecycle, and the one rule that governs it:

> **NEVER acknowledge a message before it is durably stored.** If you ack first and the server dies before the write lands, the sender believes it was delivered and the message is gone forever. That's an N2 violation — the unforgivable bug in a chat system.

```
 Alice's        Server 1        Cassandra      Redis/Bus      Server 47      Bob's
 client                                                                      client
   │               │                │              │              │            │
   │──1. SEND──────▶│                │              │              │            │
   │  {clientMsgId} │                │              │              │            │
   │               │                │              │              │            │
   │               │──2. INSERT ────▶│              │              │            │
   │               │   (RF=3, QUORUM)│              │              │            │
   │               │◀── written ─────│              │              │            │
   │               │                │              │              │            │
   │◀──3. ACK ─────│   ✓ SENT       │              │              │            │
   │   {messageId} │   ── the single grey tick appears NOW ──     │            │
   │               │                │              │              │            │
   │               │──4. lookup sess:bob ─────────▶│              │            │
   │               │◀──── "server-47" ─────────────│              │            │
   │               │                │              │              │            │
   │               │──5. PUBLISH channel:server-47─▶│              │            │
   │               │                │              │─────────────▶│            │
   │               │                │              │              │──6. push──▶│
   │               │                │              │              │            │
   │               │                │              │              │◀─7. DELIVERED_ACK
   │               │                │              │              │            │
   │               │◀─────── 8. relay receipt ─────────────────────│            │
   │◀──9. RECEIPT──│  ✓✓ DELIVERED  │              │              │            │
   │   (two grey ticks appear)      │              │              │            │
   │               │                │              │              │            │
   │               │                │              │   Bob opens the chat window │
   │               │                │              │              │◀─10. READ_ACK
   │               │◀────── 11. relay read receipt ────────────────│            │
   │◀─12. RECEIPT──│  ✓✓ READ (blue)│              │              │            │
   │               │                │              │              │            │
```

The four states, and who causes each:

| Tick | State | Triggered by |
|---|---|---|
| (clock icon) | **PENDING** | Client-side only. The message is in the client's local outbox; no server has seen it. |
| ✓ | **SENT** | Server has **durably persisted** it. This is the server's promise: *"I will not lose this."* |
| ✓✓ | **DELIVERED** | The recipient's *device* received it and acked. Not read — just landed. |
| ✓✓ blue | **READ** | The recipient's app rendered the message in a visible chat window. |

#### The offline case

Step 4 comes back **empty** — `sess:bob` doesn't exist. Bob's socket is gone.

```
  1. Message is ALREADY persisted (step 2 happened regardless). Nothing is lost.
  2. Mark it undelivered for Bob (it stays in the store; his sync will pick it up).
  3. Publish an event to the Notification Service (via Kafka).
  4. Notification Service calls APNs (iOS) or FCM (Android) with a preview payload.
  5. Bob's phone shows a banner. Bob taps it.
  6. App foregrounds → opens a WebSocket → lands on whatever server the LB picks (say server 12)
  7. Server 12 writes sess:bob = "server-12" in Redis
  8. Client sends: { "type": "SYNC", "lastSeenMessageId": "m_1002" }
  9. Server 12 queries Cassandra:
       SELECT * FROM messages WHERE conversation_id = ? AND message_id > 1002
     ...for each of Bob's conversations, and streams the gap down.
 10. Client acks them all → sender's ticks flip to ✓✓
```

Alice's message spent an hour sitting in Cassandra and got delivered the moment Bob came back. **The persist-first rule is what made that possible.**

---

### 10. Deep dives

#### (a) Group chat fanout

A message to a 100-person group is **not one delivery — it's up to 100 deliveries.**

The naive client-side approach (the client sends 100 copies, one per recipient) is terrible: it burns the sender's battery and mobile data, and it's slow. **Do the fanout on the server.**

```
Alice sends 1 message to group G (100 members)
        │
        ▼
Chat server 1:
   1. Persist ONCE:  INSERT INTO messages (conversation_id = G, ...)   ← one row, not 100
   2. Look up members: SELECT user_id FROM conversation_members WHERE conversation_id = G
   3. For each of the 99 other members:
        - session lookup (a single Redis MGET, not 99 GETs!)
        - online?  → publish to their server's channel
        - offline? → enqueue a push notification
```

**Note the write amplification asymmetry**: **one** write to the message store, but **up to 99** socket pushes. Storage is cheap; fanout is what costs you.

**Why cap groups at 100?** Do the math. A "community" of 100,000 members where 500 people are chatting actively:

```
500 msg/s × 100,000 members = 50,000,000 socket pushes/second
```

That single group would consume more fanout capacity than your entire 50M-user platform. It's a **hot partition** in Cassandra *and* a fanout bomb in the chat tier. This is exactly why WhatsApp historically capped groups in the hundreds and why huge broadcast groups are built on a **completely different architecture** — a *pull* model (clients poll a feed) rather than a *push* model. Broadcast/channel features are a fundamentally different system, not a bigger group chat. Say that.

#### (b) Presence — and the fanout trap

**How presence works:** the client sends a `HEARTBEAT` every 20 seconds. The server refreshes a Redis key with a TTL:

```js
await redis.set(`presence:${userId}`, 'online', { EX: 30 });  // TTL 30s > heartbeat 20s
```

If the client dies, crashes, or loses signal, the heartbeats stop, the key expires after 30s, and the user is automatically offline. **You get failure detection for free from a TTL** — no explicit "goodbye" message needed (and you couldn't rely on one anyway; a phone that falls in a lake doesn't send a graceful disconnect).

**Now the trap.** The obvious design is: *when a user comes online, broadcast their status to all their friends.*

```
User with 1,000 contacts comes online
   → 1,000 presence pushes

Now imagine a user on a flaky train ride, flapping online/offline every 10 seconds:
   → 6 flaps/minute × 1,000 contacts = 6,000 pushes/minute FROM ONE USER

Scale to 10M online users with a 500-contact average:
   → a presence storm that dwarfs your actual message traffic
```

Presence fanout can genuinely exceed message fanout. Three fixes, in order of importance:

1. **Only send presence to people who are actually looking.** When Alice opens a chat with Bob, her client *subscribes* to Bob's presence. When she closes the window, it unsubscribes. You do not push Bob's status to 1,000 people who have the app in their pocket. This alone kills 95%+ of the traffic.
2. **Debounce / coalesce.** Don't emit an OFFLINE event the instant the heartbeat lapses — wait for the TTL, and suppress rapid flaps. A user who goes offline and online within 30 seconds should generate *zero* events.
3. **Let clients poll for bulk presence.** For the inbox list (30 conversations), a single `GET /v1/presence?userIds=...` batch call every 30 seconds is far cheaper than 30 live subscriptions.

#### (c) Reconnection and message sync

Sockets die constantly — subway tunnels, wifi/LTE handoff, app backgrounding, server deploys. Reconnection isn't an edge case, it's the **normal case**, and the client must handle it invisibly.

The protocol:

```
 1. Socket drops. Client detects (heartbeat timeout or explicit close event).
 2. Client reconnects with EXPONENTIAL BACKOFF + JITTER.
      ← the jitter matters enormously: if a chat server holding 75,000 sockets dies,
        all 75,000 clients try to reconnect. Without jitter they arrive as one
        synchronized wave — a THUNDERING HERD that kills the next server too.
 3. New socket lands on server 12 (a fresh LB decision). Server 12 sets sess:bob = "server-12".
 4. Client sends its high-water mark:  { type: "SYNC", lastSeenMessageId: "m_1002" }
 5. Server queries the gap and streams it down, in order.
 6. Client acks; delivery receipts flow back to the senders.
```

The client stores messages in a **local SQLite database** and only ever asks for the delta. This is why WhatsApp still shows your history in an elevator with no signal: the phone is the primary read replica. The server is the durable backup and the relay.

#### (d) End-to-end encryption (E2EE), conceptually

In an E2EE system (WhatsApp uses the **Signal protocol**), the **server relays ciphertext it cannot read.**

- Every device generates a keypair. The **private key never leaves the device.** Public keys are published to a server-side key directory.
- To message Bob, Alice fetches Bob's public key bundle, derives a shared secret, and encrypts locally.
- The server stores and forwards an **opaque blob**. It knows *that* Alice messaged Bob, and *when*, and *how big* — it does not know *what*.

**What E2EE breaks** — and naming these is the real signal:

| Broken capability | Why |
|---|---|
| **Server-side search** | The server sees ciphertext. Search must run entirely on-device, over the local SQLite DB. |
| **Cloud history restore** | A new phone can't decrypt old messages without a key. Hence "encrypted backups" gated on a password/recovery key — lose it and your history is *genuinely* gone. |
| **Server-side spam/abuse ML** | You can't scan content you can't read. Moderation shifts to metadata signals and user reports. |
| **Multi-device** | Each device needs its own key, so a message must be encrypted **once per recipient device**, not once per recipient. Fanout multiplies. |
| **Rich link previews on the server** | The client has to fetch them, leaking the URL to the client's network. |

Note what E2EE does **not** change: routing, presence, session lookup, ordering, delivery receipts, storage volume. The body is just an opaque byte array. **Your architecture is unchanged** — a nice thing to point out, because it shows you can separate the security layer from the transport layer.

---

## Visual / Diagram description

The three diagrams above are the ones to memorize and reproduce on a whiteboard:

1. **The architecture diagram (section 6)** — clients → LB → *stateful* chat servers → the five supporting services (session, message store, routing bus, presence, notification), plus the separate media path. Draw the chat servers as a *row* of boxes, not one box; the plurality is the whole point.
2. **The routing diagram (section 7)** — two versions, direct RPC vs pub/sub. This is the one the interviewer is waiting for.
3. **The ACK sequence diagram (section 9)** — the vertical lifelines with the persist-before-ack ordering made explicit.

The one thing every diagram must show is the **asymmetry between the two axes**: the *vertical* axis (client ↔ its own chat server) is a persistent socket; the *horizontal* axis (chat server ↔ chat server) is a lookup plus a hop. Almost every mistake in this design comes from confusing those two.

---

## Node.js implementation

A minimal but real chat server: WebSocket termination, Redis session registry, Redis pub/sub routing, persist-before-ack, presence TTL, and offline push.

```js
// chat-server.js  —  one node of the ~150-server chat tier
import { WebSocketServer } from 'ws';
import { createClient } from 'redis';
import { randomUUID } from 'node:crypto';

const SERVER_ID = process.env.SERVER_ID || `chat-${randomUUID().slice(0, 8)}`;
const HEARTBEAT_TTL = 30; // seconds — must exceed the client's 20s heartbeat interval

// We need TWO Redis clients: a connection in subscriber mode can't run normal commands.
const redis = createClient({ url: process.env.REDIS_URL });
const sub = redis.duplicate();
const pub = redis.duplicate();
await Promise.all([redis.connect(), sub.connect(), pub.connect()]);

// THE state that makes this server stateful. It lives only in this process's memory.
// If the process dies, this map dies, and 75,000 clients must reconnect elsewhere.
const sockets = new Map(); // userId -> WebSocket

// ---------------------------------------------------------------------------
// INBOUND ROUTING: every server subscribes to exactly one channel — its own.
// Other servers publish here when they hold a message for a user WE own.
// ---------------------------------------------------------------------------
await sub.subscribe(`channel:${SERVER_ID}`, (raw) => {
  const { toUserId, frame } = JSON.parse(raw);
  const ws = sockets.get(toUserId);
  // If the socket vanished between the sender's session lookup and now, we just drop it.
  // That is SAFE: the message is already in Cassandra, and the client's reconnect
  // gap-sync will pull it. The bus is a fast path, never the source of truth.
  if (ws?.readyState === ws.OPEN) ws.send(JSON.stringify(frame));
});

// ---------------------------------------------------------------------------
// CONNECTION LIFECYCLE
// ---------------------------------------------------------------------------
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', async (ws, req) => {
  const userId = await authenticate(req); // validate the JWT from the query string / header
  if (!userId) return ws.close(4001, 'unauthorized');

  sockets.set(userId, ws);

  // Announce to the world: "I am holding this user's socket."
  // Every other chat server will find this when it needs to reach them.
  await redis.set(`sess:${userId}`, SERVER_ID, { EX: HEARTBEAT_TTL });
  await redis.set(`presence:${userId}`, 'online', { EX: HEARTBEAT_TTL });

  ws.on('message', (buf) => handleFrame(userId, ws, JSON.parse(buf)));

  ws.on('close', async () => {
    sockets.delete(userId);
    // Best-effort cleanup. We do NOT depend on this running — a crashed process
    // never gets here. The TTL is the real failure detector.
    await redis.del(`sess:${userId}`);
    await redis.del(`presence:${userId}`);
  });
});

async function handleFrame(userId, ws, frame) {
  switch (frame.type) {
    case 'HEARTBEAT':
      // Refreshing the TTL is the entire presence mechanism. No heartbeat → key
      // expires → the user is offline. Works even if their phone falls in a lake.
      await redis.expire(`sess:${userId}`, HEARTBEAT_TTL);
      await redis.expire(`presence:${userId}`, HEARTBEAT_TTL);
      break;
    case 'SEND':
      await handleSend(userId, ws, frame);
      break;
    case 'DELIVERED_ACK':
      await markDelivered(userId, frame.messageIds); // → relays a RECEIPT to the sender
      break;
    case 'READ_ACK':
      await markRead(userId, frame.conversationId, frame.upToMessageId);
      break;
  }
}

// ---------------------------------------------------------------------------
// THE HOT PATH — send a message
// ---------------------------------------------------------------------------
async function handleSend(senderId, senderWs, frame) {
  const { clientMsgId, conversationId, body, mediaUrl } = frame;

  // IDEMPOTENCY: the client retries on flaky networks. Without this, one message
  // becomes three. Return the ORIGINAL id so the client's optimistic UI reconciles.
  const existing = await redis.get(`idem:${clientMsgId}`);
  if (existing) {
    return senderWs.send(JSON.stringify({ type: 'ACK', clientMsgId, messageId: existing }));
  }

  // ORDERING: an atomic per-conversation counter. Not a client timestamp (clocks lie,
  // and users can lie harder), not a global sequence (that would need consensus).
  // Ordering only has to hold WITHIN a conversation — and that is nearly free.
  const seq = await redis.incr(`seq:${conversationId}`);
  const messageId = `${conversationId}:${seq}`;

  // ══ DURABILITY FIRST ══
  // Persist BEFORE acking. If we ack first and crash here, Alice sees "✓ sent"
  // for a message that no longer exists anywhere in the universe. Unforgivable.
  await cassandra.execute(
    `INSERT INTO messages (conversation_id, message_id, sender_id, body, media_url, created_at)
     VALUES (?, ?, ?, ?, ?, ?)`,
    [conversationId, seq, senderId, body, mediaUrl, new Date()],
    { prepare: true, consistency: 'QUORUM' } // QUORUM: survives losing one replica
  );
  await redis.set(`idem:${clientMsgId}`, messageId, { EX: 86400 });

  // NOW it is safe to promise the sender we have it. Single grey tick.
  senderWs.send(JSON.stringify({ type: 'ACK', clientMsgId, messageId, seq }));

  // ══ FANOUT ══ (async from the sender's point of view — they already got their ✓)
  const members = await getMembers(conversationId);     // capped at 100
  const recipients = members.filter((id) => id !== senderId);

  // ONE round-trip for all session lookups, not N. At 100 members that's the
  // difference between 1ms and 100ms of Redis latency.
  const keys = recipients.map((id) => `sess:${id}`);
  const servers = keys.length ? await redis.mGet(keys) : [];

  const outFrame = { type: 'MESSAGE', messageId, conversationId, senderId, body, mediaUrl, seq };

  await Promise.all(
    recipients.map(async (recipientId, i) => {
      const targetServer = servers[i];

      if (!targetServer) {
        // OFFLINE PATH. The message is already durably stored — the recipient will
        // pull it on reconnect. All we do now is wake their phone up.
        return enqueuePushNotification(recipientId, conversationId, senderId, body);
      }

      if (targetServer === SERVER_ID) {
        // Lucky: we hold their socket ourselves. Skip the bus entirely.
        const ws = sockets.get(recipientId);
        if (ws?.readyState === ws.OPEN) ws.send(JSON.stringify(outFrame));
        return;
      }

      // CROSS-SERVER: publish to the channel that server is subscribed to.
      // Note we never learn its IP, never open a connection to it, never health-check
      // it. Adding or removing chat servers requires zero coordination here.
      await pub.publish(`channel:${targetServer}`, JSON.stringify({ toUserId: recipientId, frame: outFrame }));
    })
  );
}

// Offline delivery is a DIFFERENT system with different guarantees: async,
// at-least-once, best-effort. Kafka decouples it so a slow APNs never slows chat.
async function enqueuePushNotification(userId, conversationId, senderId, body) {
  await kafka.send({
    topic: 'push-notifications',
    messages: [{
      key: userId, // key by user → all of one user's pushes stay ordered in one partition
      value: JSON.stringify({
        userId, conversationId, senderId,
        preview: body.slice(0, 100), // a teaser, not the truth. The real message is in Cassandra.
      }),
    }],
  });
}
```

The client side, with the two details that actually matter — **backoff with jitter**, and **gap sync**:

```js
// client.js (browser / React Native)
class ChatClient {
  constructor(url, token) {
    this.url = url;
    this.token = token;
    this.attempt = 0;
    this.lastSeenMessageId = loadFromLocalDb(); // the client's high-water mark
  }

  connect() {
    this.ws = new WebSocket(`${this.url}?token=${this.token}`);

    this.ws.onopen = () => {
      this.attempt = 0;
      // Ask for everything we missed while we were away. This is what makes the
      // pub/sub bus safe to be lossy — the client always reconciles against the DB.
      this.ws.send(JSON.stringify({ type: 'SYNC', lastSeenMessageId: this.lastSeenMessageId }));
      this.heartbeat = setInterval(() => this.ws.send(JSON.stringify({ type: 'HEARTBEAT' })), 20_000);
    };

    this.ws.onmessage = (e) => {
      const frame = JSON.parse(e.data);
      if (frame.type === 'MESSAGE') {
        saveToLocalDb(frame);                 // local SQLite is the client's read replica
        this.lastSeenMessageId = frame.messageId;
        this.ws.send(JSON.stringify({ type: 'DELIVERED_ACK', messageIds: [frame.messageId] }));
      }
    };

    this.ws.onclose = () => {
      clearInterval(this.heartbeat);
      // EXPONENTIAL BACKOFF + JITTER. When a chat server dies, 75,000 clients
      // reconnect at once. Without jitter they arrive as a single synchronized
      // wave and take down the next server too — a thundering herd.
      const base = Math.min(1000 * 2 ** this.attempt++, 30_000);
      const delay = base * (0.5 + Math.random() * 0.5); // spread over 50–100% of the window
      setTimeout(() => this.connect(), delay);
    };
  }
}
```

---

## Real world examples

### WhatsApp

The famous data point: WhatsApp served ~450 million users with an engineering team of about 35 people, running on **Erlang/OTP** — a runtime whose lightweight processes and per-process isolation are almost purpose-built for holding vast numbers of concurrent connections. They publicly demonstrated **2 million concurrent connections on a single tuned FreeBSD box** (a heavily optimized benchmark, not the everyday number, but it shows what the ceiling looks like when you tune the kernel for it). They ship **Signal-protocol end-to-end encryption** by default, which is exactly why WhatsApp cloud backups are separately encrypted and why losing your key means losing your history. And they have historically kept group sizes bounded (hundreds, not millions) — the fanout math above is why.

### Facebook Messenger

Messenger's message store ran on **HBase** (a wide-column store, same family as Cassandra) for years — precisely the write-heavy, time-ordered, single-partition-read access pattern described in section 5 — before a later migration to their own **MyRocks/LSM-tree**-based store. Same principle either way: a **log-structured storage engine** that turns random writes into sequential ones. They also famously use **MQTT** rather than raw WebSocket for the mobile transport, because MQTT's frame overhead is smaller and it was designed for constrained, unreliable mobile networks — a good reminder that "WebSocket" is a default, not a law.

### Slack

Slack's clients hold a persistent WebSocket to a per-connection "gateway" service. Their historical bootstrap flow is instructive: the client called `rtm.start`, which returned a **huge** initial-state payload (every channel, every user in the workspace) *and* the WebSocket URL. As workspaces grew to tens of thousands of users, that payload grew to megabytes and became a serious startup bottleneck. They replaced it with `rtm.connect` — return the socket URL immediately, and let the client **lazily fetch** the rest. That's the same lesson as the gap-sync above: **connect fast, sync incrementally, never block the socket on a big fetch.**

---

## Trade-offs

**Protocol**

| Choice | You gain | You give up |
|---|---|---|
| WebSocket | Full-duplex, low latency, low per-message overhead | Stateful servers, complex LB, hard reconnect logic, no free HTTP caching |
| Long polling | Works everywhere, dead simple, stateless-ish | Higher latency, an HTTP round trip per message, more server churn |

**Routing**

| Choice | You gain | You give up |
|---|---|---|
| Direct RPC (session lookup → gRPC) | Lowest latency (~1 extra hop) | N×N mesh, tight coupling to topology, stale-session failures handled inline |
| Redis pub/sub | Decoupling, trivial autoscaling, one connection per server | ~1-3ms extra latency; fire-and-forget (safe *only* because you persisted first) |
| Kafka | Durable, replayable, ordered per partition | ~10ms+ latency — usually too slow for the hot path; great for the *push* path |

**Storage**

| Choice | You gain | You give up |
|---|---|---|
| Cassandra / wide-column | Massive write throughput, linear scale, single-partition history reads | Joins, ad-hoc queries, strong consistency by default, transactional integrity |
| PostgreSQL | Joins, transactions, familiarity | Falls over well before 50k writes/s without heavy manual sharding |

**Message IDs**

| Choice | You gain | You give up |
|---|---|---|
| Redis `INCR` per conversation | Gap-free, strictly ordered, dead simple | A network round-trip on the write path; Redis becomes durability-critical |
| Snowflake IDs | Zero coordination, generated locally, time-sortable | Clock-sync dependency; ties within a millisecond across servers are arbitrary |

**Rule of thumb:** **Persist first, route second, notify third.** The database is the truth; the pub/sub bus and the push notification are both *optimizations* over "the client will pull it eventually." Design every path so that if the fast path fails, the slow path still delivers the message. That single principle is what makes a chat system that never loses a message.

---

## Common interview questions on this topic

### Q1: "User A is on chat server 1 and user B is on chat server 47. How does A's message reach B?"

**Hint:** This is the whole interview. Server 1 looks up `sess:B` in Redis and gets `"server-47"`. Then either (a) direct gRPC to server 47, or (b) `PUBLISH channel:server-47`. Draw both, then commit to pub/sub and justify it: decoupling is worth 2ms, and the pub/sub drop risk is acceptable **because the message was persisted to Cassandra before publishing**, so a lost push is recovered by the recipient's reconnect gap-sync. Say "the bus is the fast path; the database is the truth."

### Q2: "Why can't you just put the chat servers behind a normal load balancer?"

**Hint:** Because chat servers are **stateful** — server 47 physically owns the TCP socket to user B. A load balancer distributes *new* connections, but a message for B must reach *the one specific server holding B's socket*. Round-robining it to any healthy server would drop it. The LB is used exactly once, at handshake time; after that, every message routes by **session lookup**, not by load balancing.

### Q3: "How do you guarantee a message is never lost?"

**Hint:** **Persist before you ack.** The server writes to Cassandra at QUORUM consistency, and only *then* sends ✓ to the sender. If the server crashes before the write, the client never got an ack, so its retry (with the same `clientMsgId`, for idempotency) re-sends it. If the server crashes *after* the write but before the push, the message is safely stored and the recipient pulls it on reconnect. Every failure mode collapses into "it's in the database, and the client will sync."

### Q4: "How do you order messages? Can't you just use timestamps?"

**Hint:** No. Client clocks drift, differ by timezone, and can be *maliciously set* — a client could pin its message to the top of a conversation forever. DB auto-increment doesn't exist because the store is sharded. Use a **per-conversation Redis `INCR`** or **Snowflake IDs**. Then land the key insight: **ordering only needs to hold within a conversation, not globally** — nobody can observe two unrelated conversations at once, so global ordering (which would require consensus) is complexity you can simply decline to buy.

### Q5: "What happens when a chat server holding 75,000 connections crashes?"

**Hint:** All 75,000 sockets drop simultaneously. Their `sess:*` and `presence:*` keys expire from Redis within 30s (the TTL — no cleanup code required). Clients detect the close and reconnect with **exponential backoff and jitter**; the LB spreads them across the surviving servers, and each writes a fresh `sess:` entry. **No messages are lost** — anything acked was persisted first, and each client's `SYNC` with `lastSeenMessageId` pulls the gap. The real danger is the **thundering herd**: 75,000 simultaneous reconnects. Jitter is what saves you.

### Q6: "Why do you need push notifications if you already have WebSockets?"

**Hint:** Because a backgrounded mobile app **cannot hold a socket open** — iOS suspends the process, Android dozes it. The socket is gone the moment the user swipes away. So you need a second delivery path that goes through the OS vendor (APNs/FCM), which owns a single persistent connection on behalf of *every* app on the device. The push carries a *preview*; the real message is fetched over the socket on reconnect. This dual-path design is mandatory, and candidates routinely forget it.

---

## Practice exercise

### Whiteboard the routing decision, then break it

**Part 1 — Draw it (15 min).** On a real whiteboard or paper, from memory, draw:
- The architecture diagram, with **at least three** chat servers drawn as separate boxes.
- Alice on server 1, Bob on server 47. Trace her message all the way to his screen, labeling every hop with the *number of milliseconds* you estimate it takes.

**Part 2 — The estimation (10 min).** Redo the connection math for a *different* scale: **200M DAU, 25% concurrently online**, and assume a server holds **60,000** sockets. How many chat servers? Now assume each server costs $200/month — what's the monthly bill *just to hold sockets open*? Compare that to the bandwidth cost of the actual messages (50k msg/s × 200 bytes). Write down the ratio. This number is the reason chat is expensive.

**Part 3 — Break it (15 min).** Write down, in prose, exactly what happens in each of these cases. For each one, state whether a message can be lost, and why or why not:
1. Server 1 crashes **after** the Cassandra write but **before** the `PUBLISH`.
2. Server 47 crashes **between** server 1's `sess:B` lookup and its `PUBLISH`.
3. Bob's socket closes **while** server 47 is mid-`ws.send()`.
4. Redis (the whole session store) goes down for 30 seconds.
5. Alice's phone loses signal after sending, never receives the ✓, and retries the same message.

If your answer to any of 1–3 is "the message is lost," go back and re-read section 9 — you'll find you moved the persist step. Case 4 is the genuinely interesting one: *what degrades, and what still works?*

**Deliverable:** one page of diagrams, one page of numbers, one page of failure prose.

---

## Quick reference cheat sheet

- **The connection count drives the design, not the message rate.** 10M concurrent users ÷ 75k sockets/server ≈ **150 chat servers** — just to hold the sockets. The actual message bandwidth (~10 MB/s) is trivial.
- **WebSocket, because chat is bidirectional and high-frequency.** Full-duplex over one persistent TCP connection. Long polling is the fallback; SSE is one-directional and therefore disqualified.
- **Chat servers are STATEFUL.** They own the sockets. A normal load balancer cannot route a message to a specific user — it only distributes the *initial handshake*.
- **Session service = `user_id → server_id` in Redis.** This map is the only reason cross-server delivery is possible.
- **Routing: session lookup + direct RPC, OR publish to the target server's pub/sub channel.** Prefer pub/sub for decoupling. It's safe to be lossy **only because you persisted first**.
- **PERSIST BEFORE YOU ACK.** Never send ✓ for a message that isn't durably stored. This single rule is the entire durability guarantee.
- **Wide-column store (Cassandra), not SQL.** Chat is write-heavy, time-series-shaped, join-free. Log-structured writes are ~O(1).
- **Partition by `conversation_id`, cluster by `message_id DESC`.** "Last 50 messages in this chat" becomes a single-partition sequential read — the cheapest read a distributed DB can do.
- **Never trust client clocks for ordering.** They drift, and users can lie. Use a per-conversation Redis `INCR` or Snowflake IDs.
- **Ordering only matters WITHIN a conversation, never globally.** No observer can see two conversations at once. This simplification is free — take it.
- **Idempotency via `clientMsgId`.** Networks retry; without a dedupe key, one message becomes three.
- **Dual delivery path: WebSocket when online, APNs/FCM when offline.** A backgrounded phone cannot hold a socket. Candidates forget this.
- **Media never crosses the chat socket.** Pre-signed S3 upload → send the URL → recipient fetches from the CDN.
- **Presence = a Redis key with a TTL, refreshed by heartbeat.** Expiry *is* your failure detector. Beware the fanout: only push presence to people with the chat window actually open.
- **Cap group size (≈100).** Fanout, not storage, is the cost: one write, 99 pushes. Broadcast channels are a *pull* system, not a bigger group chat.
- **Reconnect with exponential backoff + JITTER, then sync by `lastSeenMessageId`.** 75,000 clients reconnecting in unison is a thundering herd.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [98 — HLD: Notification System](./98-hld-notification-system.md) — the offline delivery path in this design *is* a notification system; chat is its most demanding client |
| **Next** | [82 — Content Negotiation and Protocols](./82-content-negotiation-and-protocols.md) — the full polling vs long-polling vs SSE vs WebSocket comparison this doc leans on |
| **Related** | [67 — Message Queues](./67-message-queues.md) — the pub/sub bus that routes between chat servers, and the Kafka topic that feeds push notifications |
| **Related** | [64 — Database Sharding](./64-database-sharding.md) — why `conversation_id` is the right partition key, and why an auto-increment message ID is impossible |
| **Related** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — the reasoning behind choosing Cassandra over PostgreSQL for a write-heavy, join-free workload |
