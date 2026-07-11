# 15 — REST over HTTP

> **Phase 3 — Topic 2 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`, `14-http-headers-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a giant library.

- Every **book** is a *resource* (a user, an order, a product).
- Every book has a fixed **shelf address** so you can always find it again — that's the **URL** (`/books/451`).
- You don't shout "please go run the getBook function for book 451!" at the librarian. You just point at a shelf location and say what you want to *do* with it.
- The **verb** is how you interact: *read it* (GET), *put a new book on the shelf* (POST), *replace the whole book* (PUT), *fix one page* (PATCH), *remove it* (DELETE).

REST is just the agreement that **the address names a thing (a noun)** and **the verb says what you're doing to it**. The librarian (server) keeps no memory of your last visit — every request tells the full story of who you are and what you want.

The magic rule: if you ask the librarian to *remove book 451* and the message gets lost in the noise so you ask again, the book is still just... gone. Asking twice does no extra harm. That property — **idempotency** — is the whole reason REST survives on an unreliable network.

---

## Where This Lives in the Network Stack

REST is **not a protocol**. It is a *design style* that lives entirely inside the HTTP payload at Layer 7. There is no "REST packet." Every REST call is an ordinary HTTP request riding on the exact stack you already know.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application                                           │
│    ┌───────────────────────────────────────────────────────┐    │
│    │  REST  ← a CONVENTION for using HTTP verbs + URLs       │   │ ← you design this
│    │  HTTP/1.1, HTTP/2, HTTP/3  ← the actual protocol         │   │
│    └───────────────────────────────────────────────────────┘    │
│  Layer 6 — Presentation  (TLS — REST usually rides HTTPS)       │
│  Layer 4 — Transport     (TCP — reliable ordered byte stream)   │
│  Layer 3 — Network       (IP — routing)                         │
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (fiber, copper, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key point:** REST adds *zero* bytes to the wire beyond what HTTP already carries. When you "design a REST API" you are deciding: which URL names each resource, which HTTP method maps to each action, and which status code the response uses. The transport (TCP), the encryption (TLS), the framing (HTTP) are all inherited unchanged from the topics before this one.

---

## What Is This?

**REST** = **RE**presentational **S**tate **T**ransfer. It was defined in Roy Fielding's 2000 PhD dissertation (Fielding co-authored the HTTP spec, so this is not academic hand-waving — it's the design philosophy of the web itself).

Fielding described REST as a set of **six architectural constraints**. An API is "RESTful" only if it obeys them:

| Constraint | What it means in practice |
|-----------|---------------------------|
| **Client–Server** | UI/client and data/server are separate; they evolve independently. |
| **Stateless** | The server stores *no client session*. Every request carries everything needed to process it (auth, context). |
| **Cacheable** | Responses must declare whether they can be cached (`Cache-Control`, `ETag`). |
| **Uniform Interface** | Resources are identified by URLs; you manipulate them through standard methods and representations (JSON/XML). This is the big one. |
| **Layered System** | A client can't tell if it's talking to the origin server or a proxy/gateway/CDN in between. |
| **Code-on-Demand** (optional) | Server *may* send executable code (e.g. JavaScript). Rarely used; the only optional constraint. |

### Honest truth: "REST" vs "HTTP JSON API"

Almost nothing you build at work is *truly* RESTful by Fielding's strict definition. True REST requires **HATEOAS** (Hypermedia As The Engine Of Application State) — responses should include links telling the client what actions are possible next, so the client discovers the API at runtime instead of hardcoding URLs.

Almost nobody does HATEOAS. What 99% of the industry calls "a REST API" is really a **resource-oriented JSON-over-HTTP API**: noun-based URLs, HTTP verbs mapped to CRUD, JSON bodies, sensible status codes. That's Richardson Maturity Model **Level 2**. Full HATEOAS is Level 3.

```
Richardson Maturity Model
─────────────────────────
Level 0:  One URL, one verb (POST), RPC tunneled through HTTP  ← SOAP-style
Level 1:  Multiple resource URLs, still mostly POST            ← "RPC with nouns"
Level 2:  Resource URLs + proper HTTP verbs + status codes     ← what everyone calls "REST"
Level 3:  Level 2 + HATEOAS (hypermedia links)                 ← true REST, rare
```

**This doc teaches Level 2 deliberately** — because that's the pragmatic, industry-standard target. When someone says "build a REST endpoint," they mean Level 2. We'll be honest about the difference rather than pretend everyone ships HATEOAS.

---

## Why Does It Matter for a Backend Developer?

REST design decisions are the part of your API that is **almost impossible to change later** without breaking every client. Get it wrong and you live with it for years.

Concretely, understanding REST properly is what lets you:

- **Design URLs that don't need rewriting in v2** — because you chose nouns and hierarchy correctly the first time.
- **Make retries safe.** Networks drop packets (Topic 01). Clients *will* retry. If your `POST /charges` isn't idempotency-protected, retries **double-charge customers**. This is a real outage class, not theory.
- **Return the right status code** so clients, load balancers, CDNs, and monitoring dashboards behave correctly. A `200 OK` wrapping an error body is invisible to every alerting system on earth.
- **Scale horizontally.** Statelessness means any server can handle any request — put 20 instances behind a load balancer (Topic 28) and it just works. Server-side sessions break this.
- **Play nicely with the whole HTTP ecosystem** — caching (Topic 20), gateways (Topic 34), rate limiters (Topic 18) — all of which key off methods, status codes, and cache headers you set.

Bad REST design isn't a style nit. It's the difference between an API that a caching proxy can safely accelerate and one that silently corrupts data on the first network blip.

---

## The Packet/Protocol Anatomy

There is no REST header on the wire. A REST call *is* an HTTP message. But the REST-relevant fields — the ones **you** design — sit in specific places:

```
REQUEST (what the client sends)
┌───────────────────────────────────────────────────────────────┐
│  POST /v1/users/123/orders?notify=true HTTP/1.1                │
│   │    │        │      │      │                                 │
│   │    │        │      │      └── query string (filter/sort/page)│  ← you design
│   │    │        │      └── sub-collection (hierarchy)            │  ← you design
│   │    │        └── resource id                                  │  ← you design
│   │    └── version + collection noun                             │  ← you design
│   └── HTTP METHOD (verb = the action)                            │  ← you design
│                                                                  │
│  Host: api.myapp.com                                             │
│  Content-Type: application/json     ← format of the body I sent  │
│  Accept: application/json           ← format I want back         │
│  Authorization: Bearer eyJhbGci...  ← auth travels in EVERY req  │  ← statelessness
│  Idempotency-Key: 9f8a-4c2b-...     ← makes a retry safe         │  ← you design
│                                                                  │
│  {"item_id": 55, "qty": 2}          ← the representation (JSON)  │
└───────────────────────────────────────────────────────────────┘

RESPONSE (what the server sends back)
┌───────────────────────────────────────────────────────────────┐
│  HTTP/1.1 201 Created               ← STATUS CODE = the outcome  │  ← you design
│   │        │   │                                                 │
│   │        │   └── reason phrase (cosmetic, ignore it)           │
│   │        └── status code (machines read THIS)                  │
│   └── protocol version                                           │
│                                                                  │
│  Location: /v1/users/123/orders/987 ← where the new thing lives  │  ← you design
│  Content-Type: application/json                                  │
│  ETag: "a1b2c3"                     ← version tag for caching     │
│                                                                  │
│  {"id": 987, "item_id": 55, "qty": 2, "status": "pending"}      │
└───────────────────────────────────────────────────────────────┘
```

The three things REST is really about all live in this picture:
1. **The request line** — method + URL — is the *verb + noun*.
2. **The status line** — the response code — is the *outcome*, machine-readable.
3. **Auth + idempotency headers** — repeated on *every* request — are what make REST stateless and retry-safe.

---

## How It Works — Step by Step

Let's walk one full REST interaction: creating an order for a user, then reading it back.

### Step 1 — Client picks the verb + noun

```
Intent: "create a new order for user 123"

Action = create      → HTTP method POST
Thing  = orders under user 123 → URL /v1/users/123/orders

Request line:  POST /v1/users/123/orders HTTP/1.1
```
POST targets a **collection** (`/orders`). The server will mint a new id — the client does **not** choose it.

### Step 2 — Client attaches representation + context

```
Content-Type: application/json     ← "the body below is JSON"
Accept: application/json           ← "send the result back as JSON"
Authorization: Bearer eyJ...       ← identity, in THIS request (stateless)
Idempotency-Key: order-req-8f3a    ← "if you already processed this key, don't redo it"

{"item_id": 55, "qty": 2}
```

### Step 3 — Server processes and chooses a status code

```
Server logic:
  - Auth valid?          no  → 401 Unauthorized     (who are you?)
  - Allowed to do this?  no  → 403 Forbidden         (I know you, you can't)
  - User 123 exists?     no  → 404 Not Found
  - Body valid JSON?     no  → 400 Bad Request       (I can't parse this)
  - Fields make sense?   no  → 422 Unprocessable     (parsed, but qty=-5 is invalid)
  - Idempotency-Key seen before? yes → return the ORIGINAL 201 (no double-create)
  - All good?                → 201 Created + Location header
```

### Step 4 — Server responds with 201 + Location

```
HTTP/1.1 201 Created
Location: /v1/users/123/orders/987
Content-Type: application/json

{"id": 987, "item_id": 55, "qty": 2, "status": "pending"}
```
`201 Created` says "a new resource now exists." `Location` tells the client its permanent URL.

### Step 5 — Client reads the resource back (safe, idempotent)

```
GET /v1/users/123/orders/987 HTTP/1.1
Authorization: Bearer eyJ...        ← auth AGAIN — server kept no session

HTTP/1.1 200 OK
Content-Type: application/json
ETag: "a1b2c3"

{"id": 987, "item_id": 55, "qty": 2, "status": "pending"}
```
GET is **safe** (changes nothing) and **idempotent** (call it 1000×, same result, no side effects). That's why proxies and browsers freely cache and pre-fetch GETs — and why you must **never** hide a state change behind a GET.

### The full picture

```
┌──────────────┐                                   ┌──────────────────────┐
│   Client     │                                   │  api.myapp.com       │
│              │  POST /v1/users/123/orders        │                      │
│              │  Idempotency-Key: order-req-8f3a  │  create order        │
│              │──────────────────────────────────►│  mint id 987         │
│              │◄──────────────────────────────────│  201 Created         │
│              │       Location: .../orders/987    │  (store key→result)  │
│              │                                   │                      │
│  [retry?]    │  POST (same Idempotency-Key)      │  key seen before →   │
│              │──────────────────────────────────►│  return SAME 201     │
│              │◄──────────────────────────────────│  NO second order     │
│              │                                   │                      │
│              │  GET /v1/users/123/orders/987     │                      │
│              │──────────────────────────────────►│  read (safe)         │
│              │◄──────────────────────────────────│  200 OK + body       │
└──────────────┘                                   └──────────────────────┘
```

---

## Exact Syntax Breakdown

### The five methods with curl

```bash
# GET — read a resource (default method, no -X needed)
curl https://api.myapp.com/v1/users/123

# POST — create a new resource in a collection
curl -X POST https://api.myapp.com/v1/users \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice","email":"alice@example.com"}'

# PUT — REPLACE a resource entirely (send the whole object)
curl -X PUT https://api.myapp.com/v1/users/123 \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice","email":"new@example.com","role":"admin"}'

# PATCH — partial update (send only what changes)
curl -X PATCH https://api.myapp.com/v1/users/123 \
     -H "Content-Type: application/json" \
     -d '{"email":"new@example.com"}'

# DELETE — remove a resource
curl -X DELETE https://api.myapp.com/v1/users/123
```

### Anatomy of the POST command

```
curl  -X POST   -H "Content-Type: application/json"   -d '{...}'   https://.../users
│     │         │                                     │           │
│     │         │                                     │           └── target URL (the collection)
│     │         │                                     └── request body (the representation)
│     │         └── header: tells the server the body is JSON
│     └── set the HTTP method to POST
└── the curl command
```

### Reading the status line (use -i or -w)

```bash
# -i includes response headers so you can SEE the status code
curl -i -X POST https://api.myapp.com/v1/users \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice"}'

# Response starts with the status line — read it:
#   HTTP/1.1 201 Created      ← 201 = created successfully
#   Location: /v1/users/456   ← the new resource's URL

# Print ONLY the status code (great for scripts/CI)
curl -o /dev/null -s -w "%{http_code}\n" https://api.myapp.com/v1/users/123
# → 200
```

### Resource URL grammar

```
/v1        /users       /123      /orders       ?status=paid&sort=-created&page=2
 │          │            │         │             │
 │          │            │         │             └── query: filter, sort, paginate
 │          │            │         └── sub-collection (this user's orders)
 │          │            └── specific resource (one user, by id)
 │          └── collection (plural noun)
 └── version prefix
```

---

## Example 1 — Basic

Full CRUD lifecycle of one `user` resource against a local API on port 3000. Each block shows the command, then the response. Watch the **method** and the **status code** on every step.

```bash
# CREATE — POST to the collection
curl -i -X POST http://localhost:3000/v1/users \
     -H "Content-Type: application/json" -d '{"name":"Alice","email":"alice@example.com"}'

  HTTP/1.1 201 Created                                 ← new resource made
  Location: /v1/users/1                                ← its permanent address
  {"id":1,"name":"Alice","email":"alice@example.com"}

# READ — GET the resource by id
curl -i http://localhost:3000/v1/users/1

  HTTP/1.1 200 OK
  {"id":1,"name":"Alice","email":"alice@example.com"}

# PARTIAL UPDATE — PATCH just the email (name untouched)
curl -i -X PATCH http://localhost:3000/v1/users/1 \
     -H "Content-Type: application/json" -d '{"email":"alice@newmail.com"}'

  HTTP/1.1 200 OK
  {"id":1,"name":"Alice","email":"alice@newmail.com"}  ← name unchanged

# FULL REPLACE — PUT the WHOLE object (omit a field and it's GONE)
curl -i -X PUT http://localhost:3000/v1/users/1 \
     -H "Content-Type: application/json" -d '{"name":"Alice Smith","email":"alice@newmail.com"}'

  HTTP/1.1 200 OK
  {"id":1,"name":"Alice Smith","email":"alice@newmail.com"}

# DELETE — remove it. 204 = success, no body to return
curl -i -X DELETE http://localhost:3000/v1/users/1

  HTTP/1.1 204 No Content                              ← gone

# READ AGAIN — it's gone now
curl -i http://localhost:3000/v1/users/1

  HTTP/1.1 404 Not Found
  {"error":{"code":"not_found","message":"User 1 does not exist"}}

# DELETE AGAIN — prove idempotency: repeating causes NO new change
curl -o /dev/null -s -w "Second DELETE → %{http_code}\n" -X DELETE http://localhost:3000/v1/users/1

  Second DELETE → 404
```
The **state** ("user 1 does not exist") is identical whether you DELETE once or ten times — the second call caused no *additional* change. That is idempotency. (Returning 404 vs 204 on the repeat is a style choice; both are fine as long as no new side effect occurs.)

---

## Example 2 — Production Scenario: The double-charge bug

**The situation.** You run a payments service. The endpoint that charges a customer is:

```
POST /v1/charges     body: {"customer_id":"cus_88","amount":4999,"currency":"usd"}
```

A customer taps "Pay" on a flaky mobile connection. The request reaches your server, you charge the card, but the **response is lost on the way back** (packet dropped — Topic 01: networks are unreliable). The mobile app sees a timeout, assumes failure, and **automatically retries**. Your server receives a *second* identical POST and charges the card **again**. The customer is billed **$99.98** for one $49.99 order. Support tickets flood in.

### Why this happens

```
POST is NOT idempotent by definition.
Each POST /v1/charges is treated as "create a NEW charge."
Two POSTs = two charges. The server has no way to know
the second one was a retry of the first, not a new purchase.
```

This isn't a bug in your code logic — it's a **design gap**. The retry is *correct* client behavior on an unreliable network. The server must be built to absorb it.

### Diagnose it

```bash
# Simulate the retry: same charge sent twice, no protection
BODY='{"customer_id":"cus_88","amount":4999,"currency":"usd"}'
curl -s -X POST http://localhost:3000/v1/charges -H "Content-Type: application/json" -d "$BODY"
curl -s -X POST http://localhost:3000/v1/charges -H "Content-Type: application/json" -d "$BODY"

  {"id":"ch_A1","amount":4999,"status":"succeeded"}   ← charge #1
  {"id":"ch_B2","amount":4999,"status":"succeeded"}   ← charge #2 — DOUBLE CHARGE!
```
Two different charge ids. The customer paid twice.

### The fix: an Idempotency-Key (the Stripe pattern)

The client generates a **unique key per logical operation** (a UUID) and sends it in a header, reusing the *same* key on every retry of that one payment. The server stores `key → result` the first time, and on any repeat of the key returns the **stored original result** instead of charging again.

```bash
KEY="idem-$(uuidgen)"    # one key for THIS payment attempt, reused on retry
curl -s -X POST http://localhost:3000/v1/charges -H "Content-Type: application/json" -H "Idempotency-Key: $KEY" -d "$BODY"
curl -s -X POST http://localhost:3000/v1/charges -H "Content-Type: application/json" -H "Idempotency-Key: $KEY" -d "$BODY"

  {"id":"ch_A1","amount":4999,"status":"succeeded"}   ← charge created
  {"id":"ch_A1","amount":4999,"status":"succeeded"}   ← SAME id, no second charge
```
Same charge id both times. The card was charged exactly **once**. The retry was safely absorbed.

### Server logic behind the key

```
On POST /v1/charges with Idempotency-Key: K
  1. Look up K in the idempotency store (Redis/Postgres).
  2. If K exists AND request completed:
        → return the stored response verbatim (200/201). Do NOT re-charge.
  3. If K exists but is still processing:
        → return 409 Conflict ("a request with this key is in flight").
  4. If K is new:
        → record K as "processing", perform the charge,
          store the result under K (with a TTL, e.g. 24h),
          return the response.
```

**Real-world reference:** This is exactly how Stripe's `Idempotency-Key` header works, and how AWS, PayPal, and every serious payments API handle retries. If you build a mutation endpoint that costs money or is otherwise irreversible, **you must support idempotency keys.**

### The related fix: kill verb-based URLs

While auditing this API, you find the legacy endpoints were RPC-style:

```
Bad (verb-in-URL, RPC-over-HTTP):        Good (resource-oriented REST):
  GET  /getCharge?id=ch_A1          →      GET    /v1/charges/ch_A1
  POST /createCharge                →      POST   /v1/charges
  POST /refundCharge?id=ch_A1       →      POST   /v1/charges/ch_A1/refunds
  POST /updateCustomer?id=cus_88    →      PATCH  /v1/customers/cus_88
  POST /deleteCustomer?id=cus_88    →      DELETE /v1/customers/cus_88
```
The verb-based scheme buries *everything* under POST, so no proxy, cache, or client can reason about safety — `getCharge` looks identical to `deleteCustomer` at the HTTP layer. The resource-oriented scheme lets the whole ecosystem (caches, gateways, retry logic) work automatically because the method *is* the semantics.

---

## Common Mistakes

### Mistake 1: Verbs in the URL

```
BAD   POST /createUser
BAD   GET  /getUserById?id=123
BAD   POST /users/123/delete
BAD   GET  /api/fetchAllOrders
```
**Why it's wrong:** The HTTP method already IS the verb. Putting the action in the path is redundant and turns your API into RPC-over-HTTP, defeating the entire uniform-interface constraint. Now `POST /users/123/delete` can never be cached, and clients can't reason about safety.
```
GOOD  POST   /users            (create)
GOOD  GET    /users/123        (read)
GOOD  DELETE /users/123        (delete)
```
**Rule:** URLs are **nouns**. Methods are **verbs**. If you see a verb in a path, you designed it wrong. (Exception: genuine *actions* that aren't CRUD, like `POST /users/123/actions/verify-email`, are pragmatically acceptable — but treat that as a resource-y sub-collection, not a free-for-all.)

---

### Mistake 2: Using GET to change state

```javascript
// BAD — CATASTROPHIC
app.get('/users/123/delete', (req, res) => { db.deleteUser(123); ... });
```
**Why it's dangerous:** GET is defined as **safe** — it must have no side effects. The entire web assumes this:
- Browsers **pre-fetch** GET links. A crawler or the browser's link-prefetcher can delete your data by just *looking* at the page.
- **Proxies and CDNs cache** GET responses. The delete "succeeds" from cache without ever hitting your server.
- **Retries** of GET are automatic and unlimited, because GET is supposed to be harmless.
```
GOOD  Use DELETE /users/123 for deletion.
GOOD  Use POST/PATCH/PUT for any state change.
GOOD  GET must NEVER modify anything. Ever.
```
Google famously learned this in 2005 when a "prefetch links" browser accelerator silently deleted content by crawling GET-based "delete" links in an admin panel.

---

### Mistake 3: Returning 200 OK for errors

```json
BAD:  HTTP/1.1 200 OK
      {"success": false, "error": "user not found"}
```
**Why it's wrong:** The status code is the **machine-readable** outcome. Monitoring dashboards, load balancers, retry libraries, and CDNs all read the status *code*, not your JSON body. A 200-wrapped error is **invisible** to every one of them:
- Your uptime graph shows 100% success while users see failures.
- Client retry logic (`if (status >= 500) retry`) never triggers.
- A caching proxy happily caches the "error" as a valid response.
```json
GOOD: HTTP/1.1 404 Not Found
      {"error": {"code": "not_found", "message": "user not found"}}
```
**Rule:** The status code carries the outcome. The body carries the *details*. Never lie with a 200.

---

### Mistake 4: Non-idempotent PUT/DELETE (or ignoring retries on POST)

```javascript
// BAD — PUT that appends instead of replacing — NOT idempotent
app.put('/users/123/tags', (req, res) => {
  user.tags.push(req.body.tag);   // second call adds a SECOND copy!
});
```
**Why it's wrong:** PUT and DELETE are **required by spec to be idempotent** — calling them N times must leave the same state as calling them once. Clients and infrastructure retry them freely *because* of this guarantee. If your PUT appends or your DELETE decrements a counter, retries corrupt state.
```javascript
// GOOD — PUT REPLACES the whole collection — idempotent
app.put('/users/123/tags', (req, res) => {
  user.tags = req.body.tags;      // same result no matter how many times
});
```
And the flip side: **POST is not idempotent**, so if a POST creates something important, you must add an **Idempotency-Key** (see Example 2). Ignoring retries on a money-moving POST is the double-charge bug.

---

### Mistake 5: Inconsistent pluralization and versioning

```
BAD   GET /user/123          (singular here...)
BAD   GET /orders            (...plural there)
BAD   GET /getProductList    (verb + no version)
BAD   /api/users  vs  /v2/orders  vs  /orders   (three schemes at once)
```
**Why it's wrong:** Inconsistency forces every client developer to memorize special cases and re-check the docs for every endpoint. It also makes future changes ambiguous.
```
GOOD  Always plural collections:   /users, /orders, /products
GOOD  Consistent version prefix:   /v1/users, /v1/orders
GOOD  Pick ONE versioning scheme (URL /v1/ OR header) and use it everywhere
```

---

### Mistake 6: Wrong status code family

```
BAD   400 Bad Request  for "you're not logged in"     (that's 401)
BAD   401 Unauthorized for "you can't touch this"      (that's 403)
BAD   200 OK           for "created"                    (that's 201)
BAD   500              for "your input is invalid"      (that's 400/422)
```
**Why it's wrong:** 4xx means "**you** (client) made an error — don't blindly retry." 5xx means "**I** (server) failed — retrying may help." Mixing them makes clients retry un-retryable errors (hammering your API with the same invalid request) or give up on transient ones. Return `4xx` for client faults, `5xx` only for genuine server faults.

---

## Hands-On Proof

```bash
# 1. Prove GET is safe + idempotent: call it 5×, state never changes
for i in 1 2 3 4 5; do
  curl -o /dev/null -s -w "GET #$i → %{http_code}\n" https://httpbin.org/get
done

# 2. httpbin's /put endpoint ONLY accepts PUT — it echoes your JSON body back
curl -s -X PUT -d '{"x":1}' -H "Content-Type: application/json" https://httpbin.org/put | grep '"json"'

# 3. Watch different outcomes map to different status codes
for c in 200 201 404 422 429; do
  curl -o /dev/null -s -w "status/$c → %{http_code}\n" https://httpbin.org/status/$c
done

# 4. See 405 + the Allow header when you use the wrong verb on a resource
#    (httpbin's /get only allows GET — POST to it returns 405)
curl -i -s -X POST https://httpbin.org/get | head -5
#    Look for:  HTTP/1.1 405 METHOD NOT ALLOWED
#               Allow: GET, HEAD, OPTIONS      ← tells you what IS allowed

# 5. Full request line + status line for a PATCH in one view
curl -i -s -X PATCH https://httpbin.org/patch \
     -H "Content-Type: application/json" -d '{"email":"x@y.com"}' | head -3
```

---

## Practice Exercises

### Exercise 1 — Easy: Fix the broken URL scheme

Below is a real (bad) API. Rewrite every endpoint into clean, resource-oriented REST with the correct HTTP method. State the method AND the URL.

```
1.  GET  /getAllProducts
2.  GET  /getProduct?productId=55
3.  POST /createProduct
4.  POST /updateProductPrice?id=55
5.  POST /deleteProduct?id=55
6.  GET  /getProductReviews?productId=55
```
```
# Answer format:
# 1. GET /v1/products
# 2. ...
# For each: which Fielding constraint did the original violate?
```

### Exercise 2 — Medium: Assign the right status code

For each scenario, give the **single most correct** status code and one sentence of justification.

```
a. Client POSTs a new user; it's created successfully.
b. Client sends malformed JSON that won't parse.
c. Client sends valid JSON but "age": -5 (fails business validation).
d. Client requests /users/999 which doesn't exist.
e. Client has no auth token at all.
f. Client is authenticated but not allowed to delete this resource.
g. Client sends DELETE to a read-only /reports endpoint.
h. Client tries to register an email that's already taken.
i. Client has sent 10,000 requests this minute (limit is 100).
j. Client GETs a resource successfully.

# Hint tables: 200 vs 201 vs 204 | 400 vs 422 | 401 vs 403 | 404 | 405 | 409 | 429
```

### Exercise 3 — Hard (Production Simulation): Design a retry-safe order API

You're designing the checkout endpoint for an e-commerce backend. Requirements:

```
- Creating an order charges a credit card (irreversible, costs money).
- Mobile clients retry automatically on timeout.
- Clients can read, list (paginated), and cancel orders.
- An order can't be cancelled twice; cancelling an already-cancelled order
  must not error out the client's retry logic.

Deliverables:
1. Design the full URL scheme (create, read, list-with-pagination, cancel).
   Decide: is "cancel" a verb-in-URL violation, or is there a resource-y way?
2. For CREATE: which method? Is it idempotent? What header makes retries safe,
   and describe the server-side store (what's the key, what's the value, TTL)?
3. For CANCEL: which method? Is it idempotent by nature? What status code on
   the first cancel vs. a repeated cancel?
4. Write the curl command that creates an order safely on a flaky network.
5. What status code does the server return if two requests with the SAME
   idempotency key arrive concurrently (one still processing)?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **What is the difference between "safe" and "idempotent"?** Give one method that is both, one that is idempotent but not safe, and one that is neither.

2. **Your `PUT /users/123` is called three times with an identical body due to network retries. How many times should the user's data change, and why does the HTTP spec guarantee this is safe?**

3. **A teammate ships `GET /orders/500/cancel` to cancel an order. Name two concrete ways this will silently break in production** (think: what does the browser/CDN do to GET requests?).

4. **When do you return `201` vs `200` vs `204`? What extra header should accompany a `201`?**

5. **Explain to a junior why `POST /charges` needs an Idempotency-Key but `DELETE /charges/ch_1` does not.**

6. **What is the difference between `401` and `403`? Between `400` and `422`?**

7. **"REST is stateless." If the server keeps no session, how does the second request in a conversation know who the user is?**

---

## Quick Reference Card

### HTTP methods — safe & idempotent

| Method | CRUD | Safe? | Idempotent? | Body? | Why the idempotency answer |
|--------|------|-------|-------------|-------|----------------------------|
| **GET** | Read | Yes | Yes | No | Reads nothing changes; call it forever, same result |
| **POST** | Create | No | No | Yes | Each call creates a *new* resource, so repeats = duplicates |
| **PUT** | Replace | No | Yes | Yes | Replaces with the *same* full payload, so same end state |
| **PATCH** | Partial update | No | Not necessarily | Yes | `{"email":x}` is idempotent; `{"balance":"+5"}` is not |
| **DELETE** | Delete | No | Yes | Rare | After first delete it's gone; repeats leave same state |
| **HEAD** | Read headers | Yes | Yes | No | Like GET but no body |
| **OPTIONS** | Discover | Yes | Yes | No | Asks what methods/CORS are allowed |

> **Safe** = no side effects (read-only). **Idempotent** = N identical calls leave the same *server state* as 1 call. All safe methods are idempotent; not all idempotent methods are safe.

### Method → typical status code

| Method | Success | Common errors |
|--------|---------|---------------|
| GET | `200 OK` | `404` not found, `304` not modified (cache) |
| POST (create) | `201 Created` + `Location` | `400`/`422` bad input, `409` conflict/duplicate |
| POST (action) | `200 OK` / `202 Accepted` | `409`, `422` |
| PUT | `200 OK` or `204 No Content` | `404`, `409` version conflict |
| PATCH | `200 OK` or `204` | `404`, `422` |
| DELETE | `204 No Content` | `404` (or `204` again if idempotent) |
| any | — | `401` no auth, `403` forbidden, `405` wrong method, `429` rate limited |

### Status codes that trip people up

| Code | Meaning | Use when |
|------|---------|----------|
| `200` | OK | Read succeeded, or update returning the body |
| `201` | Created | New resource made — **add `Location` header** |
| `204` | No Content | Success, nothing to return (typical DELETE / PUT) |
| `400` | Bad Request | Malformed — server can't even parse it |
| `401` | Unauthorized | *Not authenticated* — "who are you?" (misnamed) |
| `403` | Forbidden | Authenticated but *not allowed* — "you can't" |
| `404` | Not Found | Resource doesn't exist |
| `405` | Method Not Allowed | Wrong verb — **add `Allow` header** listing valid ones |
| `409` | Conflict | Duplicate / version clash / in-flight idempotency key |
| `422` | Unprocessable Entity | Parsed fine, but *semantically* invalid (age = -5) |
| `429` | Too Many Requests | Rate limited — **add `Retry-After`** (Topic 18) |

### URL design cheat block

```
GOOD                                    BAD
────────────────────────────────────    ──────────────────────────────
GET    /v1/users                        GET  /getAllUsers
GET    /v1/users/123                     GET  /getUser?id=123
POST   /v1/users                         POST /createUser
PUT    /v1/users/123                     POST /updateUser?id=123
PATCH  /v1/users/123                     POST /users/123/update
DELETE /v1/users/123                     GET  /users/123/delete   ← unsafe!
GET    /v1/users/123/orders             GET  /getOrdersForUser?u=123

Nouns not verbs · plural collections · hierarchy for ownership
Filter/sort/page in query string, never the path:
  GET /v1/orders?status=paid&sort=-created_at&page=2&limit=20
Version once, consistently:  /v1/...  (or Accept: application/vnd.myapp.v1+json)
```

### Consistent JSON error envelope

```json
{
  "error": {
    "code": "validation_failed",
    "message": "age must be a positive integer",
    "field": "age",
    "request_id": "req_9f8a2c"
  }
}
```
Same shape for *every* error, at every endpoint. Machines read `code`; humans read `message`; support reads `request_id`.

### Idempotency-Key (retry-safe POST)

```
Client:  POST /v1/charges
         Idempotency-Key: <uuid, one per logical operation, reused on retries>

Server:  first time  → do the work, store key→response (TTL ~24h)
         key seen    → return the STORED response (no re-execution)
         in-flight   → 409 Conflict
```

---

## When Would I Use This at Work?

### Scenario 1: "Customers are getting charged twice"
You now recognize this instantly: a non-idempotent `POST` plus client retries on a flaky network (Topic 01 — networks drop packets). The fix is an `Idempotency-Key` header with a server-side key→result store, exactly like Stripe. You'd never "fix" it by telling clients to stop retrying — retries are correct; the server must be built to absorb them.

### Scenario 2: "Our monitoring says 100% uptime but users report errors"
Someone is returning `200 OK` with `{"success": false}` in the body. Your dashboards and load balancer only read status *codes*, so failures are invisible. You audit the endpoints, switch error paths to proper `4xx`/`5xx`, and suddenly the alerts and retry logic work — and your "uptime" honestly reflects reality.

### Scenario 3: "We're designing v2 of the public API"
This is where getting REST right pays off. You choose plural noun collections, consistent `/v1/` versioning, hierarchy for ownership, query params for filtering/sorting/pagination, and a single JSON error envelope — because these are the decisions you *can't* cheaply change once external clients depend on them. An API gateway (Topic 34) will route by these paths and a CDN (Topic 20/30) will cache your GETs, so clean method semantics directly enable the rest of your infrastructure.

### Scenario 4: "The load balancer keeps sending users to the wrong server after a scale-up"
Someone built server-side sessions, violating statelessness. Because state lives in one server's memory, requests must be pinned there (sticky sessions — Topic 28), which breaks horizontal scaling. The REST-correct fix: make every request carry its own auth (Bearer token — Topic 16), so *any* server can handle *any* request and you scale freely.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — why networks are unreliable (the reason idempotency and retries matter at all)
- `10-http1.1-in-depth.md` — the raw methods, status codes, and request/response format REST builds on
- `14-http-headers-in-depth.md` — `Content-Type`, `Accept`, `Location`, `Cache-Control` — the headers REST leans on

**Study next (in order):**
- `16-authentication-over-http.md` — how each stateless request carries identity (Bearer tokens, cookies, JWT)
- `17-cors-in-depth.md` — the preflight `OPTIONS` request browsers send before your REST calls
- `18-rate-limiting-at-http-level.md` — the `429` status and `Retry-After` header your REST API returns under load
- `20-caching-in-depth.md` — why safe/idempotent GETs are cacheable and unsafe methods are not

**This topic connects forward to:**
- `34-api-gateways.md` — gateways route, authenticate, and rate-limit based on the REST URL + method design you choose here
- `37-grpc-and-protocol-buffers.md` — the RPC alternative to REST for internal services, and when to pick it instead

---

*Doc saved: `/docs/networking/15-rest-over-http.md`*
