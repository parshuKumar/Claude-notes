# 69 — API Design — REST vs GraphQL vs gRPC
## Category: HLD Components

---

## What is this?

An **API** (Application Programming Interface) is the *contract* between two pieces of software: "send me a request shaped like THIS, and I promise to send back a response shaped like THAT." Everything else — how you store data, what language you wrote it in — is your business, not the caller's.

Think of it like a **restaurant menu**. The menu is the API. It says what you can order and roughly what you'll get. You don't get to walk into the kitchen and rearrange the fridge. REST, GraphQL, and gRPC are three different *styles of menu* — a fixed printed menu, a build-your-own-bowl counter, and a private ordering line for the staff.

---

## Why does it matter?

The API is the **hardest thing in your system to change.** You can swap Postgres for Cassandra, rewrite a service in Rust, add ten caching layers — and nobody outside notices. But change `GET /users/:id` to return `full_name` instead of `name`, and every mobile app in the wild breaks, including the ones on phones that will never install an update.

**What breaks without good API design:**
- Mobile clients making 6 round-trips to render one screen → the screen takes 3 seconds on 4G
- A retry after a network timeout double-charges a customer (because `POST /payments` isn't idempotent)
- A "page 2" that shows the same row twice because someone inserted a record while the user was paging
- A single expensive GraphQL query taking down your database

**In interviews:** After you draw the architecture, the interviewer *always* says "OK, what does the API look like?" You should be able to write endpoint signatures, status codes, and a pagination scheme within a minute. Getting cursor pagination right is a strong senior signal.

**At work:** You will design or consume APIs every single week. Versioning and deprecation strategy is the difference between shipping calmly and living in fear of breaking prod.

---

## The core idea — explained simply

### The Three Restaurants Analogy

You're hungry. There are three restaurants on the street.

**Restaurant 1 — REST (the classic diner with a printed menu).**
The menu has fixed dishes: "Breakfast Plate," "Burger," "Salad." You order by pointing at a *thing* on the menu. If you want the burger without pickles and a side of the salad's dressing, tough — you order the burger, you order the salad, and you throw away the parts you don't want. Every table gets the exact same menu, and that predictability is the whole point: the kitchen is simple, and the waiter can memorize it.

**Restaurant 2 — GraphQL (the build-your-own-bowl counter).**
You slide your tray along and say exactly what you want: "rice, half chicken, extra beans, no cheese." One trip, exactly the food you wanted, zero waste. Beautiful. The catch: the counter staff have to be smart, the line moves slower when someone builds an absurd bowl, and you can't just re-heat "the usual" from a warming tray — every bowl is bespoke, so nothing can be pre-made (read: cached).

**Restaurant 3 — gRPC (the kitchen's internal intercom).**
The chefs don't use the menu at all. They shout terse codes at each other over an intercom: "TWO-SEVEN, FIRE." It's incomprehensible to customers, but it's *fast*, and everybody in the kitchen was trained on the exact same codebook, so mistakes are near-zero. This is not for customers. This is staff-to-staff.

**Mapping it back:**

| Analogy | Technical reality |
|---|---|
| Printed menu, fixed dishes | REST: fixed endpoints, fixed response shapes |
| Throwing away the parts you didn't want | **Over-fetching** — REST sends fields you don't need |
| Ordering 3 dishes to assemble one meal | **Under-fetching** — REST needs N round-trips |
| Build-your-own bowl | GraphQL: client specifies the exact fields |
| Can't pre-make bespoke bowls | GraphQL is hard to cache at the HTTP layer |
| An absurd 40-ingredient bowl jamming the line | Query-complexity DoS |
| Kitchen intercom codebook | Protocol Buffers: a shared, compiled schema |
| Chefs shouting, not customers | gRPC is for internal service-to-service |

---

## Key concepts inside this topic

### 1. REST, done properly

REST = **Re**presentational **S**tate **T**ransfer. In practice, "doing REST properly" means four things.

**a) Resources are NOUNS, not verbs.** The URL names a *thing*. The HTTP method says what you're doing to it.

```
BAD  (verbs in the URL — this is RPC wearing a REST costume)
  POST /createUser
  POST /getUserById
  POST /deleteUserAccount
  GET  /users/123/updateEmail?email=x@y.com

GOOD (nouns + HTTP verbs)
  POST   /users            → create a user
  GET    /users/123        → read a user
  PATCH  /users/123        → partially update a user
  DELETE /users/123        → delete a user
  GET    /users/123/orders → the orders belonging to user 123 (nesting = ownership)
```

Plural nouns. Lowercase. Hyphens not underscores (`/shipping-addresses`). Nest only one level deep — `/users/123/orders/456/items/789` is a sign you should just expose `/order-items/789`.

**b) Statelessness.** Every request carries everything the server needs to handle it. The server stores **no session memory** between requests.

```
STATEFUL (bad for scaling):
  Request 1 → Server A  ("logged in as user 7" stored in Server A's RAM)
  Request 2 → Server B  → "Who are you? 401." ← the whole system falls over

STATELESS (correct):
  Request 1 → Server A  (Authorization: Bearer <jwt>)  ✓
  Request 2 → Server B  (Authorization: Bearer <jwt>)  ✓
  Request 3 → Server C  (Authorization: Bearer <jwt>)  ✓
```
Statelessness is what lets you put N identical servers behind a load balancer and add more at will (recall [55 — Load Balancing](./55-load-balancing.md)). Session state, if you need it, goes in a shared store like Redis — not in the process.

**c) Correct status codes.** The status code is the *first thing* a client branches on. Using `200 OK` with `{"error": "not found"}` inside is a bug, because every retry library, cache, and monitor in the world reads the code, not your body.

| Code | Name | Use it when |
|---|---|---|
| **200** | OK | Successful GET / PATCH / PUT with a body |
| **201** | Created | POST created a resource. Include a `Location: /users/123` header |
| **202** | Accepted | You queued the work; it isn't done yet (async jobs) |
| **204** | No Content | Success, deliberately empty body (typical for DELETE) |
| **301/308** | Moved Permanently | The resource lives at a new URL forever |
| **304** | Not Modified | Client's `ETag`/`If-None-Match` still valid — save the bandwidth |
| **400** | Bad Request | Malformed syntax, missing required field, bad type |
| **401** | Unauthorized | You are not authenticated (really means "unauthenticated") |
| **403** | Forbidden | You ARE authenticated, but you may not do this |
| **404** | Not Found | No such resource (also used to hide existence from unauthorized users) |
| **405** | Method Not Allowed | `DELETE /users` when only GET/POST exist there |
| **409** | Conflict | Duplicate email; optimistic-lock version mismatch |
| **410** | Gone | It existed, it's permanently deleted — useful for deprecated API versions |
| **422** | Unprocessable Entity | Syntactically fine, semantically invalid ("end_date before start_date") |
| **429** | Too Many Requests | Rate limited — see [70 — Rate Limiting](./70-rate-limiting.md) |
| **500** | Internal Server Error | You broke. Never leak stack traces |
| **502/503/504** | Bad Gateway / Unavailable / Gateway Timeout | Upstream dead, overloaded, or too slow |

Rule: **4xx = the caller's fault. 5xx = your fault.** Clients retry 5xx and 429; they must never blindly retry 4xx.

**d) Idempotency of each verb — and why retries live or die on it.**

**Idempotent** means: doing it once and doing it five times leave the system in the **same final state**. (It does *not* mean the response is identical every time — `DELETE` returning 204 then 404 is still idempotent, because the *state* is the same: gone.)

| Verb | Safe (no side effects)? | Idempotent? | Cacheable? |
|---|---|---|---|
| GET | Yes | Yes | Yes |
| HEAD | Yes | Yes | Yes |
| PUT | No | **Yes** (full replace → same result every time) | No |
| DELETE | No | **Yes** (deleted stays deleted) | No |
| POST | No | **NO** | Rarely |
| PATCH | No | **Not necessarily** (`{"$inc": {"balance": -10}}` is not!) | No |

Why this is the whole ballgame:

```
Client                     Network                    Server
  │  POST /payments {$100}    │                          │
  ├──────────────────────────►│─────────────────────────►│  charges $100 ✓
  │                           │                          │  writes response
  │                           │   ✗ TCP connection dies  │
  │  ⏳ timeout               │                          │
  │  "no response... retry?"  │                          │
  │  POST /payments {$100}    │                          │
  ├──────────────────────────►│─────────────────────────►│  charges $100 AGAIN ✗
```
The client **cannot distinguish** "the request never arrived" from "it arrived and the response was lost." With a non-idempotent POST, retrying is a coin flip between correctness and double-charging. That's why every serious write API accepts an **idempotency key** — a client-generated UUID the server remembers, so the second request returns the *first* request's stored response instead of re-executing. Full treatment in [85 — Idempotency](./85-idempotency.md).

**e) HATEOAS — and why almost nobody does it.**

HATEOAS ("Hypermedia As The Engine Of Application State") is REST's top maturity level: responses embed *links* telling the client what it can do next, so the client never hardcodes URLs.

```json
{
  "id": 456, "status": "PENDING", "total": 4999,
  "_links": {
    "self":   { "href": "/orders/456", "method": "GET" },
    "pay":    { "href": "/orders/456/payment", "method": "POST" },
    "cancel": { "href": "/orders/456", "method": "DELETE" }
  }
}
```
The theory is lovely: the server can move URLs freely, and the client discovers that "cancel" disappeared once the order shipped. **Reality:** virtually nobody builds clients that actually *follow* links — they read `order.id` and construct `/orders/456/payment` themselves, because that's simpler and every code generator works that way. So HATEOAS is pure payload bloat for most teams. Know the term for interviews ("Richardson Maturity Model level 3"), and know that the honest answer is "we stop at level 2 — nouns plus verbs plus status codes."

---

### 2. GraphQL — what it actually fixes

**The problem: over-fetching and under-fetching.** Picture a mobile profile screen showing a user's name/avatar, their 3 latest posts, each post's like-count, and the names of the last 2 commenters on each.

```
REST — 5+ round-trips, ~200ms each on a bad 4G link → ~1 second of waterfall
 1. GET /users/42                        → 40 fields; you needed 2   (over-fetch)
 2. GET /users/42/posts?limit=3          → 3 posts
 3. GET /posts/1/comments?limit=2  ─┐
 4. GET /posts/2/comments?limit=2   ├─ under-fetch: N+1 at the network layer
 5. GET /posts/3/comments?limit=2  ─┘
```

```
GraphQL — 1 round-trip, exactly the fields asked for
POST /graphql
query {
  user(id: 42) {
    name
    avatarUrl
    posts(limit: 3) {
      title
      likeCount
      comments(limit: 2) { author { name } }
    }
  }
}
```
One request, no wasted bytes. On a high-latency mobile network, collapsing 5 sequential round-trips into 1 is often a **5x improvement in time-to-render** — and that, not "elegance," is why Facebook built it.

**Schema + resolvers.** You declare a typed schema; each field has a *resolver* function that knows how to fetch it.

```js
// schema.js — the contract. Strongly typed, introspectable.
export const typeDefs = `#graphql
  type User    { id: ID!, name: String!, posts(limit: Int = 10): [Post!]! }
  type Post    { id: ID!, title: String!, likeCount: Int!, author: User! }
  type Query   { user(id: ID!): User }
`;

// resolvers.js — one function per field. GraphQL walks the query tree
// and calls the resolver for every field the client asked for.
export const resolvers = {
  Query: {
    user: (_parent, { id }, ctx) => ctx.db.users.findById(id),
  },
  User: {
    // called once per User node in the result tree
    posts: (user, { limit }, ctx) => ctx.db.posts.findByAuthor(user.id, limit),
  },
  Post: {
    author: (post, _args, ctx) => ctx.loaders.userById.load(post.authorId), // see below
  },
};
```

**Cost 1 — the N+1 resolver problem.** The killer. Ask for 100 posts and each post's author, and the naive `Post.author` resolver fires **100 separate `SELECT * FROM users WHERE id = ?` queries.** 1 query for posts + 100 for authors = N+1.

The fix is **DataLoader**: it collects every `.load(id)` call made during one tick of the event loop, then makes ONE batched call.

```js
import DataLoader from 'dataloader';

// Build a FRESH loader per request — it caches, and you must not leak
// one user's data (or a stale row) into another request.
function createLoaders(db) {
  return {
    userById: new DataLoader(async (ids) => {
      // ids arrives as e.g. [7, 7, 9, 12, 9] → deduped to [7, 9, 12]
      const rows = await db.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
      const byId = new Map(rows.map((r) => [String(r.id), r]));
      // MUST return results in the SAME ORDER as the input ids
      return ids.map((id) => byId.get(String(id)) ?? null);
    }),
  };
}
// 100 queries → 1 query.
```

**Cost 2 — you lose HTTP caching.** Every GraphQL request is `POST /graphql` with a different body. CDNs and browser caches key on **method + URL**, so they see one opaque endpoint and cache nothing. Recall from [60 — CDN](./60-cdn.md) that a REST `GET /posts/1` can be cached at the edge for free; GraphQL cannot. You have to rebuild caching yourself — persisted queries (hash the query, send the hash, `GET /graphql?sha256=...`), or a client-side normalized cache (Apollo/Relay), or a server-side cache per resolver.

**Cost 3 — query-complexity DoS.** The client controls the query, which means the client controls your database load.

```graphql
query BringDownProd {          # legal, tiny, and lethal
  user(id: 1) { friends { friends { friends { friends { name } } } } }
}
```
Defences (do all three): a **depth limit** (reject depth > ~10), a **complexity score** (assign each field a cost, multiply by list sizes, reject > budget), and **persisted queries** (only run queries you shipped and pre-approved).

**Cost 4 — no free 404s.** GraphQL returns `200 OK` with `{"data": {...}, "errors": [...]}` even for "not found" or partial failures. Your monitoring dashboards, which count non-2xx responses, will show a serene 100% success rate while half your resolvers are throwing. You must instrument the `errors` array explicitly.

---

### 3. gRPC — the internal workhorse

gRPC is Google's RPC (Remote Procedure Call) framework. You call `client.GetUser({id: 42})` and it *feels* like a local function call, but it's a network hop.

**a) Protocol Buffers (protobuf)** — the schema language and the binary wire format.

```protobuf
// user.proto — the single source of truth, shared by every service
syntax = "proto3";
package user.v1;

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc StreamUserEvents (StreamRequest) returns (stream UserEvent); // server streaming
}

message GetUserRequest { string id = 1; }

message User {
  string id    = 1;   // ← the field NUMBER, not the name, goes on the wire
  string name  = 2;
  string email = 3;
}
```
On the wire, `{"id":"42","name":"Ana"}` becomes roughly `0A 02 34 32 12 03 41 6E 61` — the field number and type packed into one byte, then the raw value. No field names, no quotes, no braces. Typical payload is **3–10x smaller than JSON** and parses far faster (no string scanning, no type coercion).

Those field numbers are what give you **safe evolution**: adding `string phone = 4;` is backwards-compatible — old clients simply ignore field 4. *Never reuse or renumber a field.*

**b) HTTP/2 multiplexing and binary framing.** gRPC runs on HTTP/2 (see [82 — Protocols](./82-content-negotiation-and-protocols.md)). Many concurrent requests share **one TCP connection**, interleaved as binary frames — no head-of-line blocking at the HTTP layer, no per-request connection setup, and headers are compressed (HPACK).

**c) Four streaming modes** — this is gRPC's superpower, and REST has no equivalent:

| Mode | Shape | Example |
|---|---|---|
| **Unary** | 1 req → 1 res | `GetUser(id)` — normal RPC |
| **Server streaming** | 1 req → N res | "subscribe to price ticks for AAPL" |
| **Client streaming** | N req → 1 res | upload 10,000 telemetry points, get one ack |
| **Bidirectional** | N ↔ N | live chat, real-time collaborative editing |

**d) Code generation.** `protoc` compiles the `.proto` into typed client stubs and server skeletons in every language. The client and the server literally cannot disagree about the contract — it's checked at compile time, not discovered at 2am in production.

```js
// server.js — Node gRPC server
import grpc from '@grpc/grpc-js';
import protoLoader from '@grpc/proto-loader';

const pkgDef = protoLoader.loadSync('./user.proto');
const proto  = grpc.loadPackageDefinition(pkgDef).user.v1;

const server = new grpc.Server();
server.addService(proto.UserService.service, {
  // Node gRPC uses (call, callback); `call.request` is the decoded message
  GetUser: async (call, callback) => {
    const user = await db.users.findById(call.request.id);
    if (!user) {
      // gRPC has its OWN status codes (not HTTP ones)
      return callback({ code: grpc.status.NOT_FOUND, message: 'user not found' });
    }
    callback(null, { id: user.id, name: user.name, email: user.email });
  },

  // server streaming: push many messages down one open call
  StreamUserEvents: (call) => {
    const sub = eventBus.subscribe(call.request.userId, (evt) => call.write(evt));
    call.on('cancelled', () => sub.unsubscribe()); // client hung up — clean up!
  },
});
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => server.start());
```

**e) Why it dominates internal traffic, and why browsers hate it.** Internally you control both ends: you *want* a compiler-enforced contract, tiny payloads, and one warm connection between services doing 50k QPS. From a browser, you can't: browser JS has no API to control HTTP/2 frames directly, so a raw gRPC call is impossible. You need **gRPC-Web** plus a proxy (Envoy) to translate — extra infrastructure, and you still lose client-streaming. Also: you can't `curl` it, you can't read it in DevTools, and a CDN can't cache it. Hence the near-universal split: **gRPC behind the wall, REST/GraphQL at the front door.**

---

### 4. Choosing: the decision table

| Situation | Pick | Why |
|---|---|---|
| Public API for third-party developers | **REST** | Everyone already knows it; curl-able; cacheable; no client library needed |
| Browser-facing CRUD (admin panels, dashboards) | **REST** | Simple, cacheable, boring — boring is a feature |
| Mobile app assembling one screen from many entities | **GraphQL** | Kills the round-trip waterfall on high-latency links |
| Many diverse clients (iOS/Android/web/TV) needing different field sets | **GraphQL** | One schema, each client takes what it needs; no BFF-per-client |
| Internal service→service, high throughput / low latency | **gRPC** | Binary + HTTP/2 + typed contracts |
| Real-time bidirectional streams between services | **gRPC** | Native bidi streaming |
| Static assets, files, anything a CDN should cache | **REST** | HTTP caching is free and enormous |
| Polyglot microservices that must not break each other | **gRPC** | Codegen makes the contract compiler-checked |

**These are not exclusive.** The most common real architecture: browsers/apps → GraphQL or REST gateway → gRPC between the internal services behind it.

---

### 5. Versioning — because you *will* need to change it

| Strategy | Looks like | Verdict |
|---|---|---|
| **URL path** | `/v1/users`, `/v2/users` | Ugly, "unRESTful," and **the one everyone uses.** Trivially visible in logs, cacheable, easy to route in a gateway, testable in a browser bar. Use this. |
| **Custom header** | `X-API-Version: 2` | Clean URLs, but invisible in logs/caches and easy to forget; the version silently defaults |
| **Content negotiation** | `Accept: application/vnd.acme.v2+json` | The "purist" HTTP answer; correct, and nearly nobody does it because it's painful to test |
| **Query param** | `/users?version=2` | Messes with caching and gets dropped by proxies. Avoid. |

**Version only on BREAKING changes.** Adding an optional field is not breaking — additive changes ship in place. Breaking = removing a field, renaming one, changing a type, changing default behaviour, or tightening validation.

**How to actually deprecate (the 4-phase playbook):**
1. **Announce.** Ship v2. Start returning `Deprecation: true` and `Sunset: Wed, 01 Apr 2026 00:00:00 GMT` headers on every v1 response, plus a `Link` to the migration guide.
2. **Measure.** Log v1 usage *per API key*. You cannot kill what you cannot count. Email the top 20 callers directly.
3. **Brownout.** Return `503` for v1 for 1 hour, at a pre-announced time, twice — a month apart. This flushes out the clients whose owners "forgot." It is far kinder than a surprise permanent outage.
4. **Sunset.** Return `410 Gone` with a body pointing to v2. Keep the 410 alive for a year — a hard 404 tells the client nothing.

---

### 6. Pagination — the thing candidates get wrong

**Offset/limit (what everyone writes first, and why it breaks):**

```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 100000;
```
```json
GET /posts?limit=20&offset=100000
{ "data": [...20 items...], "total": 4813200, "limit": 20, "offset": 100000 }
```

Two fatal flaws.

**Flaw A — it gets slower the deeper you go.** `OFFSET 100000` does not skip to row 100,000. The database must **read and discard 100,000 rows** first. Page 1 is 2ms; page 5,000 is 900ms; page 50,000 times out. Cost is O(offset).

**Flaw B — inserts shift the window, so you see duplicates.** The feed is sorted newest-first and new posts arrive constantly:

```
t=0  DB rows (newest first): [P100 P99 P98 P97 P96 P95 ...]
     Client: GET /posts?limit=3&offset=0  →  [P100, P99, P98]

t=1  Someone posts P101. Rows shift right by one:
     DB rows:                [P101 P100 P99 P98 P97 P96 ...]
                              idx0 idx1 idx2 idx3 ...
     Client: GET /posts?limit=3&offset=3 → [P98, P97, P96]
                                              ▲
                            P98 ALREADY SHOWN on page 1. Duplicate.
                            (And on deletes, rows get SKIPPED entirely.)
```
Every infinite-scroll feed built on offsets shows users the same post twice. It's not a rare race — at feed scale it happens constantly.

**Cursor / keyset pagination (the correct answer).** Don't say "skip 100,000 rows." Say **"give me what comes after this exact row."**

```sql
-- Page 1
SELECT id, title, created_at FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Page 2: seek directly using the last row of page 1 as the anchor.
-- The tuple comparison handles ties in created_at correctly.
SELECT id, title, created_at FROM posts
WHERE (created_at, id) < ($1_last_created_at, $2_last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- With an index on (created_at DESC, id DESC), this is an index SEEK:
-- ~2ms whether it's page 2 or page 200,000. O(limit), not O(offset).
```

```json
GET /posts?limit=20
{
  "data": [ ... ],
  "page_info": {
    "next_cursor": "eyJjIjoiMjAyNi0wNy0xMlQwOTowMDowMFoiLCJpIjo5ODc2fQ==",
    "has_more": true
  }
}
// next call:  GET /posts?limit=20&cursor=eyJjIjoiMjAyNi0wNy0xMlQwOTowMDowMFoiLCJpIjo5ODc2fQ==
```
The cursor is just the anchor row's sort key, base64'd (opaque **on purpose**, so you can change its internals without breaking clients).

```js
// Encode/decode the cursor. Opaque to the client, structured to you.
const encodeCursor = (row) =>
  Buffer.from(JSON.stringify({ c: row.created_at.toISOString(), i: row.id })).toString('base64url');

const decodeCursor = (cursor) => {
  try {
    const { c, i } = JSON.parse(Buffer.from(cursor, 'base64url').toString());
    if (!c || !Number.isInteger(i)) throw new Error();
    return { createdAt: new Date(c), id: i };
  } catch {
    const err = new Error('Malformed cursor');
    err.status = 400;                 // a bad cursor is the CALLER's fault
    throw err;
  }
};

app.get('/posts', async (req, res, next) => {
  try {
    // Always CLAMP the limit — never let a client ask for 1,000,000 rows.
    const limit  = Math.min(Math.max(parseInt(req.query.limit ?? '20', 10), 1), 100);
    const cursor = req.query.cursor ? decodeCursor(req.query.cursor) : null;

    // Fetch limit+1 to learn whether a next page exists, without a COUNT(*).
    const rows = cursor
      ? await db.query(
          `SELECT id, title, created_at FROM posts
             WHERE (created_at, id) < ($1, $2)
             ORDER BY created_at DESC, id DESC LIMIT $3`,
          [cursor.createdAt, cursor.id, limit + 1])
      : await db.query(
          `SELECT id, title, created_at FROM posts
             ORDER BY created_at DESC, id DESC LIMIT $1`,
          [limit + 1]);

    const hasMore = rows.length > limit;
    const page    = hasMore ? rows.slice(0, limit) : rows;

    res.json({
      data: page,
      page_info: {
        next_cursor: hasMore ? encodeCursor(page[page.length - 1]) : null,
        has_more: hasMore,
      },
    });
  } catch (e) { next(e); }
});
```

| | Offset/limit | Cursor/keyset |
|---|---|---|
| Deep-page performance | O(offset) — degrades badly | O(limit) — constant |
| Stable under inserts/deletes | **No** (dupes and skips) | **Yes** |
| Jump to page 47 | Yes | No — next/prev only |
| Total count | Easy (but `COUNT(*)` is itself slow) | Awkward; use an estimate |
| Best for | Small admin tables with page numbers | Feeds, timelines, infinite scroll, any large table |

**Rule of thumb:** page numbers in an admin panel → offset. Anything a user scrolls, or anything over ~10k rows → cursor.

---

### 7. Filtering, sorting, errors, and the rest of the universal concerns

**Filtering & sorting** — keep it in query params, and **allowlist everything**:
```
GET /orders?status=shipped&created_after=2026-01-01&sort=-created_at,total&fields=id,total
                                                          ▲ leading '-' = DESC
```
```js
// NEVER interpolate user input into SQL/ORDER BY. Map it through an allowlist.
const SORTABLE = { created_at: 'created_at', total: 'total_cents', id: 'id' };

function buildOrderBy(sortParam = '-created_at') {
  const clauses = sortParam.split(',').map((token) => {
    const desc   = token.startsWith('-');
    const column = SORTABLE[desc ? token.slice(1) : token];
    if (!column) { const e = new Error(`Cannot sort by "${token}"`); e.status = 400; throw e; }
    return `${column} ${desc ? 'DESC' : 'ASC'}`;
  });
  return `ORDER BY ${clauses.join(', ')}, id DESC`; // always tie-break on a unique column
}
```

**Error response shape** — one shape, everywhere, forever. Machine-readable code + human message + field-level detail + a trace id:
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "The request body is invalid.",
    "details": [
      { "field": "email", "code": "INVALID_FORMAT", "message": "Not a valid email address." },
      { "field": "age",   "code": "OUT_OF_RANGE",   "message": "Must be between 13 and 120." }
    ],
    "request_id": "req_01HZX8Q2K3"
  }
}
```
`code` is a stable enum the client can `switch` on. `message` is for humans and may change. `request_id` is what the user pastes into a support ticket so you can find the exact log line.

**Rate limiting** — every public API needs it. Return `429` with `Retry-After` and `X-RateLimit-*` headers. Full treatment in [70 — Rate Limiting](./70-rate-limiting.md).

**Idempotency keys** — for every non-idempotent write:
```js
app.post('/payments', async (req, res, next) => {
  const key = req.get('Idempotency-Key');
  if (!key) return res.status(400).json({ error: { code: 'IDEMPOTENCY_KEY_REQUIRED' } });

  // Atomic claim: only the FIRST request with this key wins the insert.
  const claimed = await redis.set(`idem:${key}`, 'IN_PROGRESS', { NX: true, EX: 86400 });
  if (!claimed) {
    const stored = await redis.get(`idem:${key}`);
    if (stored === 'IN_PROGRESS') return res.status(409).json({ error: { code: 'REQUEST_IN_PROGRESS' } });
    return res.status(200).json(JSON.parse(stored)); // replay the original response
  }

  const result = await paymentService.charge(req.body);
  await redis.set(`idem:${key}`, JSON.stringify(result), { EX: 86400 });
  res.status(201).json(result);
});
```
See [85 — Idempotency](./85-idempotency.md) for the full design, including how to handle a crash mid-charge.

---

## Visual / Diagram description

### Diagram 1: The three styles, side by side

```
                              ┌──────────────────┐
                              │  Mobile / Browser│
                              └────────┬─────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        ▼ REST                         ▼ GraphQL                      ▼ gRPC-Web
┌────────────────┐            ┌────────────────┐            ┌────────────────┐
│ GET /users/42  │            │ POST /graphql  │            │ needs an Envoy │
│ GET /posts?..  │            │ { user(id:42){ │            │ proxy to talk  │
│ GET /comments  │            │     name       │            │ HTTP/2 frames  │
│  (N round-trips│            │     posts{...} │            │  (awkward!)    │
│   over JSON)   │            │ }}  (1 trip)   │            │                │
└───────┬────────┘            └───────┬────────┘            └───────┬────────┘
        │  cacheable at CDN ✓         │  NOT CDN-cacheable ✗       │
        └──────────────┬──────────────┴──────────────┬──────────────┘
                       ▼                             ▼
              ┌──────────────────────────────────────────────┐
              │             API GATEWAY  (auth, rate limit)  │
              └───────────────────┬──────────────────────────┘
                                  │  ══ gRPC over HTTP/2 ══  (binary, multiplexed,
        ┌─────────────────────────┼─────────────────────────┐   one warm connection)
        ▼                         ▼                         ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│ User Service  │◀──────▶│ Post Service  │◀──────▶│ Feed Service  │
│ (protobuf)    │  gRPC  │ (protobuf)    │  gRPC  │ (protobuf)    │
└───────────────┘        └───────────────┘        └───────────────┘
```
**What it shows:** the front door is REST or GraphQL (public, debuggable, cacheable); everything behind the gateway is gRPC (fast, typed, streaming). gRPC-Web is drawn as the awkward path it is.

### Diagram 2: Offset pagination duplicating a row

```
TIME ──────────────────────────────────────────────────────────────▶

t=0  DB (newest first) │ P100 │ P99 │ P98 │ P97 │ P96 │
     index              0      1     2     3     4
     ┌───────────────────────────────────────────────┐
     │ GET /posts?limit=3&offset=0  → P100, P99, P98 │  page 1 ✓
     └───────────────────────────────────────────────┘

t=1  ✱ P101 is created. Everything shifts one slot right.

     DB (newest first) │ P101 │ P100 │ P99 │ P98 │ P97 │ P96 │
     index              0      1      2     3     4     5
                                            ▲
                                       offset=3 lands HERE
     ┌───────────────────────────────────────────────┐
     │ GET /posts?limit=3&offset=3  → P98, P97, P96  │  page 2 ✗
     └───────────────────────────────────────────────┘
                                    ▲
                     P98 was ALREADY on page 1 → USER SEES IT TWICE

CURSOR FIX: page 2 asks "give me rows strictly AFTER P98" ─▶ P97, P96, P95.
            An insert at the head cannot shift a row you anchor on by identity.
```

---

## Real world examples

### 1. Stripe — REST as a masterclass
Stripe's API is the reference implementation of "boring REST done perfectly." Resources are nouns (`/v1/charges`, `/v1/customers`). Versioning is **date-pinned per account** (`Stripe-Version: 2024-06-20`) — your integration is frozen against the version you signed up on, so Stripe can ship breaking changes without breaking you, and you opt in when ready. Every mutating call accepts an `Idempotency-Key` header. Pagination is cursor-based (`starting_after=<object_id>`), never offset. Errors have a stable `type` + `code`. This is the design to copy.

### 2. GitHub — REST *and* GraphQL, deliberately
GitHub ships both. The REST API (v3) is what most scripts and CI tools use — cacheable, curl-able, ETag-supported. The GraphQL API (v4) exists because GitHub's data is a deeply connected graph (repo → issues → comments → authors → orgs) and clients like the GitHub mobile app and third-party dashboards were making dozens of REST calls to render one view. GitHub's GraphQL rate limit is *not* "5,000 requests/hour" — it's a **point budget** computed from the query's complexity, which is exactly the query-complexity defence described above.

### 3. Google / Netflix — gRPC internally
gRPC came out of Google's internal "Stubby," which handles the traffic between Google's services. Externally, Google publishes REST/JSON; internally, service-to-service is protobuf RPC over a shared connection. Netflix similarly standardized internal service communication on gRPC with protobuf-defined contracts, and uses a GraphQL federation layer at the edge to assemble device-specific responses from those internal services — the exact "REST/GraphQL front, gRPC behind" split in Diagram 1.

---

## Trade-offs

| | REST | GraphQL | gRPC |
|---|---|---|---|
| **Payload format** | JSON (text, verbose) | JSON (text) | Protobuf (binary, compact) |
| **Transport** | HTTP/1.1 or 2 | HTTP (usually POST) | HTTP/2 required |
| **Browser support** | Native, perfect | Native, perfect | Needs gRPC-Web + proxy |
| **HTTP caching / CDN** | Excellent (free) | Poor (needs persisted queries) | None |
| **Over/under-fetching** | Both, badly | Solved | Not solved (fixed messages) |
| **Streaming** | No (use SSE/WebSockets) | Subscriptions (bolted on) | Native, 4 modes |
| **Contract enforcement** | Docs + hope (OpenAPI helps) | Schema, runtime-checked | Compile-time, codegen |
| **Debuggability** | `curl` it. Trivial. | Playground/GraphiQL — good | Needs `grpcurl`; opaque bytes |
| **Learning curve** | Tiny | Medium (resolvers, DataLoader) | Medium (protoc toolchain) |
| **Biggest footgun** | Round-trip waterfalls; offset pagination | N+1 resolvers; complexity DoS | Browser hostility; ops complexity |

**The sweet spot:** Start with REST. It is boring, universally understood, cacheable, and 90% of APIs never need anything else. Add **GraphQL** only when you have provably many diverse clients suffering a round-trip waterfall — and the day you add it, add DataLoader and a depth limit in the same PR. Reach for **gRPC** the moment two of *your own* services are chatting at high volume behind the firewall. Do not put gRPC on the public internet, and do not put GraphQL on a CDN-cacheable content API.

---

## Common interview questions on this topic

### Q1: "Which HTTP methods are idempotent, and why does it matter?"
**Hint:** GET, HEAD, PUT, DELETE are idempotent; POST is not; PATCH may or may not be. It matters because of **network ambiguity**: a client that times out cannot tell "never arrived" from "arrived, reply lost." Idempotent verbs are safe to retry blindly. For POST, you need an **idempotency key** so the server can dedupe. Bonus: explain that "idempotent" is about final *state*, not identical *responses* — DELETE returning 204 then 404 is still idempotent.

### Q2: "Your feed API uses `?page=2&limit=20`. What's wrong with that at scale?"
**Hint:** Two things. (1) `OFFSET N` forces the DB to scan and discard N rows — page 5,000 is a full-table scan. (2) Concurrent inserts shift the window, so page 2 re-shows rows from page 1 (and deletes cause skips). Fix: keyset/cursor pagination — `WHERE (created_at, id) < (last_created_at, last_id) ORDER BY created_at DESC, id DESC LIMIT 20`, backed by a matching composite index. Return an opaque base64 cursor.

### Q3: "When would you NOT use GraphQL?"
**Hint:** When you have one client and simple CRUD (the flexibility buys nothing and costs you resolvers + DataLoader + complexity limits). When HTTP/CDN caching matters — public content APIs lose edge caching entirely. When the API is public and untrusted — clients controlling query shape means clients controlling your DB load. When your team is small: GraphQL's operational surface (N+1, complexity scoring, error monitoring that ignores your 200s) is real ongoing work.

### Q4: "Why is gRPC used between services but rarely from a browser?"
**Hint:** Browsers can't manipulate raw HTTP/2 frames from JS, so you need gRPC-Web + an Envoy proxy, and you still lose client-side streaming. You also give up: readable DevTools, `curl`, and CDN caching. Internally none of that matters and the wins are large — 3–10x smaller payloads, one multiplexed connection, compile-time-checked contracts across languages, and native bidi streaming.

### Q5: "How do you version an API and retire the old version?"
**Hint:** Version in the **URL path** (`/v1/`) — unfashionable but visible in logs, routable at the gateway, and cacheable. Only version on *breaking* changes (removals, renames, type changes); additive fields ship in place. To retire: announce with `Deprecation`/`Sunset` headers → measure usage per API key and email the top callers → run scheduled brownouts (return 503 for an hour, twice) → finally return `410 Gone`, and keep the 410 for a year. Mention Stripe's date-pinned versioning as the gold standard.

---

## Practice exercise

### Design the API for a Twitter-like feed

Don't write a service. Write the **contract**, in a single markdown file. ~30 minutes.

**Part A — REST endpoints.** Write the full signature for each: method, path, request body, success status code, and the error codes it can return.
1. Create a tweet
2. Get one tweet
3. Delete a tweet
4. Get a user's home timeline (paginated!)
5. Like a tweet / unlike a tweet ← *think hard: is "like" idempotent? Is `POST /tweets/:id/likes` the right shape, or `PUT /tweets/:id/likes/:userId`?*

**Part B — Pagination.** For the home timeline, write the exact SQL for page 1 and page 2 using **cursor** pagination, plus the JSON response including `page_info`. State the index you need. Then, in two sentences, explain what specifically goes wrong if you used `OFFSET`.

**Part C — Idempotency.** "Create a tweet" is a POST. A user on a flaky train taps Send once and the request times out. Their app retries. Design the exact mechanism that prevents a duplicate tweet. What header? What's stored, where, and with what TTL? What status code does the retry get back?

**Part D — Rewrite it as GraphQL.** Write the schema (`type Tweet`, `type User`, `type Query`, `type Mutation`) and the single query the mobile home screen would send. Then answer: which field in your schema is going to cause an N+1, and how does DataLoader fix it?

**Produce:** one markdown file with four sections. Get the status codes right — that's the part interviewers actually check.

---

## Quick reference cheat sheet

- **Resources are nouns, methods are verbs.** `POST /users`, never `POST /createUser`.
- **4xx = caller's fault, 5xx = your fault.** Retry 5xx and 429; never blindly retry 4xx.
- **Idempotent verbs:** GET, HEAD, PUT, DELETE. **NOT idempotent:** POST (and PATCH, usually).
- **Idempotency is about final state**, not identical responses. It's what makes retries safe.
- **Stateless servers** are what make horizontal scaling possible — session state goes in Redis, never in the process.
- **HATEOAS** = level 3 REST maturity. Know the word; know that almost nobody implements it.
- **GraphQL solves over-fetching (too many fields) and under-fetching (too many round-trips)** — 5 REST calls → 1 query.
- **GraphQL's three taxes:** N+1 resolvers (fix: DataLoader batching), no HTTP/CDN caching (fix: persisted queries), and query-complexity DoS (fix: depth + cost limits).
- **GraphQL returns 200 for errors** — your uptime dashboard will lie to you unless you instrument the `errors` array.
- **gRPC = protobuf + HTTP/2.** Binary, 3–10x smaller than JSON, multiplexed, compile-time contracts, 4 streaming modes.
- **gRPC is for behind the wall.** Browsers need gRPC-Web + a proxy; you lose curl, DevTools, and CDN caching.
- **Version in the URL path** (`/v1/`). Only on breaking changes. Deprecate with `Sunset` headers → brownouts → `410 Gone`.
- **Never ship offset pagination on a feed.** O(offset) cost + duplicate rows on insert. Use cursor/keyset.
- **Allowlist sort and filter fields.** Interpolating user input into `ORDER BY` is a SQL injection.
- **One error shape everywhere:** stable machine `code`, human `message`, per-field `details`, and a `request_id`.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — async messaging; APIs are the synchronous counterpart |
| **Next** | [70 — Rate Limiting](./70-rate-limiting.md) — every public API needs a throttle, and a `429` with `Retry-After` |
| **Related** | [58 — API Gateway](./58-api-gateway.md) — the front door that routes, authenticates, and version-switches your API |
| **Related** | [85 — Idempotency](./85-idempotency.md) — how idempotency keys make POST safe to retry |
| **Related** | [82 — Protocols: HTTP/2, HTTP/3, WebSockets, SSE](./82-content-negotiation-and-protocols.md) — the transport layer gRPC is built on |
