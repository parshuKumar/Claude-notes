# 37 — gRPC and Protocol Buffers

> **Phase 6 — Topic 3 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `11-http2.md`

---

## ELI5 — The Simple Analogy

Imagine two offices that need to talk constantly.

**REST/JSON is like sending letters.** Every time office A wants something, it writes a full letter in plain English: "Dear office B, please send me the details of customer number 42. Sincerely, A." Office B writes a full English letter back. Every word is spelled out, every message is a fresh envelope, and a human could read any letter. Friendly, but wordy and slow.

**gRPC is like a private telephone hotline with a shared codebook.** Both offices agreed in advance on a tiny codebook: "field 1 means customer id, field 2 means name, field 3 means email." Now office A doesn't say "please send me the details of customer number 42" — it just picks up the always-open hotline and says the codes: `1: 42`. Office B knows exactly what that means and replies in the same compact code. No envelopes to open and reseal, no wasted words, and the line stays open for the next call.

The catch: if you intercept the hotline, you hear gibberish (`0A 05 41...`) — you *need the codebook* (`the .proto file`) to understand it. That's the whole trade: gRPC is smaller, faster, and strongly typed, but not human-readable and not something a browser can dial directly.

---

## Where This Lives in the Network Stack

gRPC is not a new transport. It is an **application-layer framework that sits on top of HTTP/2**, which sits on top of TLS and TCP. Protocol Buffers is the **serialization format** (a presentation-layer concern) that gRPC carries as its payload.

```
┌─────────────────────────────────────────────────────────────────┐
│  YOUR CODE          stub.GetUser(req)  ← looks like a function   │
│  ───────────────────────────────────────────────────────────────│
│  gRPC framework     RPC semantics: methods, deadlines, status    │  L7
│  ───────────────────────────────────────────────────────────────│
│  Protocol Buffers   binary serialization (the payload bytes)     │  L6-ish
│  ───────────────────────────────────────────────────────────────│
│  HTTP/2             streams, multiplexing, HPACK, framing        │  L7 (Topic 11)
│  ───────────────────────────────────────────────────────────────│
│  TLS 1.3            encryption (usually mTLS internally)          │  L6 (Topic 09)
│  ───────────────────────────────────────────────────────────────│
│  TCP                reliable, ordered byte stream                 │  L4 (Topic 08)
│  ───────────────────────────────────────────────────────────────│
│  IP                 routing                                       │  L3
└─────────────────────────────────────────────────────────────────┘
```

The key insight: **gRPC did NOT invent a wire protocol.** It reused HTTP/2 (so it inherits multiplexing, header compression, flow control, and works through existing proxies/load balancers that speak HTTP/2) and reused Protocol Buffers for the body. gRPC is essentially "HTTP/2 + protobuf + a calling convention."

---

## What Is This?

**gRPC** is a high-performance **RPC (Remote Procedure Call)** framework, originally from Google, now a CNCF project. The mental model is: *you call a method on a remote server as if it were a local function in your own code.*

```
// It LOOKS like a normal function call:
user := client.GetUser(ctx, &GetUserRequest{UserId: 42})

// But under the hood it:
//   1. serializes the request to protobuf bytes
//   2. sends them over an HTTP/2 stream to a remote machine
//   3. waits for the response bytes
//   4. deserializes them back into a typed User object
```

**Protocol Buffers (protobuf)** is the schema language and binary serialization format gRPC uses. You write a `.proto` file that defines your messages and services — this is the single source of truth — and the `protoc` compiler generates strongly-typed client and server code in Go, Java, Python, C++, Node, Rust, and more.

### RPC model vs REST resource model

This is the fundamental mental shift:

```
REST thinks in NOUNS (resources) + a fixed verb set (HTTP methods):
    GET    /users/42          "read the user resource"
    POST   /users             "create a user resource"
    DELETE /users/42          "delete the user resource"
    → You design URLs and pick from GET/POST/PUT/DELETE/PATCH.

gRPC thinks in VERBS (methods/actions):
    GetUser(GetUserRequest)          returns (User)
    CreateUser(CreateUserRequest)    returns (User)
    TransferMoney(TransferRequest)   returns (TransferResult)
    → You design methods, like calling functions on a service object.
```

REST bends everything into CRUD over URLs. gRPC lets you name the *action* directly — `TransferMoney`, `RecalculateBillingCycle`, `StreamMetrics` — which often maps far more naturally to what internal services actually do.

---

## Why Does It Matter for a Backend Developer?

You will reach for gRPC the moment you have **many services talking to each other a lot**. Here's what it buys you:

- **Speed & size for internal traffic.** Binary protobuf is smaller and (de)serializes faster than JSON. On a hot service-to-service path doing 50,000 QPS, shaving payload size and CPU on every call is real money and real latency (see Example 2).
- **A contract that can't drift.** The `.proto` file is a strict, versioned schema. Client and server generate code from the same contract — no more "the docs said `userId` but the API returns `user_id`." Break the contract and the build breaks, not production.
- **Streaming for free.** Because it rides HTTP/2, gRPC gives you server-streaming, client-streaming, and full bidirectional streaming over a single long-lived connection — impossible with plain request/response REST.
- **Polyglot without hand-written clients.** A Go service and a Python service and a Java service all generate their clients from the same `.proto`. Nobody hand-writes serialization.
- **Debugging skills.** When a gRPC call hangs or returns `UNAVAILABLE`, you need to know it maps to HTTP/2 trailers and `grpc-status` codes — not HTTP status codes — to debug it (see Common Mistakes).

The flip side you must also know: gRPC is **wrong** for public/partner APIs and can't be called directly from a browser. Knowing *where the line is* is as important as knowing gRPC itself.

---

## The Packet/Protocol Anatomy

### Part A — The protobuf wire format (the payload bytes)

Every field on the wire is a **key-value pair**. The key (called the *tag*) is a varint that packs two things:

```
tag = (field_number << 3) | wire_type

┌──────────────────────────────────────────────────────────┐
│  wire_type  │ meaning                                     │
├─────────────┼─────────────────────────────────────────────┤
│     0       │ VARINT   → int32/64, uint, bool, enum       │
│     1       │ I64      → fixed64, sfixed64, double         │
│     2       │ LEN      → string, bytes, embedded messages, │
│             │            packed repeated (length-prefixed) │
│     5       │ I32      → fixed32, sfixed32, float          │
└─────────────┴─────────────────────────────────────────────┘
```

**Varint encoding** (the heart of protobuf's compactness): small numbers take fewer bytes. Each byte uses its low 7 bits for data; the high bit (MSB) is a *continuation flag* — 1 = "more bytes follow", 0 = "last byte". Groups are stored least-significant first.

```
Encode the number 300:
  300 in binary          = 1 0010 1100   (9 bits)
  split into 7-bit groups= 0000010  0101100
  little-endian order    = 0101100  0000010
  add continuation bits  = 1_0101100  0_0000010
                           └─more     └─last
  bytes                  = 0xAC       0x02      → 300 fits in 2 bytes
```

### Part B — A real byte-by-byte example (this is the whole point)

Take this schema and this data:

```proto
message User {
  string name = 1;   // = "Alice"
  int32  age  = 2;   // = 30
}
```

On the wire, protobuf emits:

```
BYTES:   0A   05   41 6C 69 63 65   10   1E
         │    │    └─────┬───────┘   │    │
         │    │      "Alice" (5 ASCII bytes)
         │    │                      │    │
         │    │                      │    └─ value: 30 (0x1E), a varint
         │    │                      └────── tag: (2<<3)|0 = 0x10  field 2, VARINT
         │    └─────────────────────────── length prefix: 5 bytes follow
         └──────────────────────────────── tag: (1<<3)|2 = 0x0A  field 1, LEN

Total: 9 bytes.
```

Now the exact same data as JSON:

```
{"name":"Alice","age":30}
└────────────── 25 bytes ──────────────┘
```

**9 bytes vs 25 bytes** — and JSON must also send the field *names* (`"name"`, `"age"`) on every single message, forever. Protobuf sends field *numbers* (`1`, `2`), which is why:

- It's **compact** — numbers instead of names, binary instead of text.
- It's **forward/backward compatible** — the receiver matches on the *number*, so renaming `name`→`full_name` in the `.proto` changes nothing on the wire. **But** reusing or renumbering a number silently corrupts meaning (see Common Mistakes).
- It's **NOT self-describing** — those 9 bytes are meaningless without the `.proto`. You cannot "just read" a protobuf payload like you can read JSON.

### Part C — How that payload rides HTTP/2

gRPC wraps each protobuf message in a 5-byte **Length-Prefixed-Message** header, then puts it inside HTTP/2 DATA frames:

```
gRPC message framing (inside an HTTP/2 DATA frame):

┌──────────┬─────────────────────┬───────────────────────────┐
│ 1 byte   │ 4 bytes             │ N bytes                   │
│ compressed│ message length      │ the protobuf message      │
│ flag     │ (big-endian uint32) │ (e.g. our 9 bytes above)  │
│ 0x00     │ 0x00 00 00 09       │ 0A 05 41 6C 69 63 65 10 1E │
└──────────┴─────────────────────┴───────────────────────────┘
```

The full HTTP/2 request for a unary call looks like this on the wire:

```
HEADERS frame (compressed by HPACK — Topic 11):
  :method: POST
  :scheme: https
  :path: /user.v1.UserService/GetUser     ← /package.Service/Method
  :authority: user-svc:50051
  content-type: application/grpc
  te: trailers
  grpc-timeout: 1S                         ← the deadline (optional)
  authorization: Bearer ...                ← custom metadata = HTTP/2 headers

DATA frame:
  [length-prefixed protobuf message]       ← the request body

--- server responds on the SAME stream ---

HEADERS frame:
  :status: 200                             ← ALWAYS 200 if gRPC itself worked
  content-type: application/grpc

DATA frame:
  [length-prefixed protobuf message]       ← the response body

TRAILERS frame (HEADERS with END_STREAM):
  grpc-status: 0                           ← the REAL status lives here
  grpc-message:                            ← human-readable error (if any)
```

The single most important thing to internalize: **HTTP status is almost always 200. The real result is the `grpc-status` trailer.** A `NOT_FOUND` still comes back as HTTP 200 with `grpc-status: 5`.

---

## How It Works — Step by Step

Let's trace a unary call `client.GetUser({user_id: 42})` end to end.

```
YOU WRITE:  user, err := client.GetUser(ctx, &GetUserRequest{UserId: 42})
```

### Step 1 — Serialize (client stub)
```
The generated stub takes your typed struct and serializes it to protobuf bytes:
  {user_id: 42}  →  08 2A          (tag 0x08 = field 1 VARINT; value 0x2A = 42)
Then prepends the 5-byte gRPC frame header:  00 00000002 08 2A
```

### Step 2 — Open (or reuse) an HTTP/2 stream
```
gRPC keeps ONE long-lived HTTP/2 connection to the server (already handshaked:
TCP + TLS done once, at startup — not per call). This call gets a fresh
HTTP/2 STREAM (a cheap integer id) on that existing connection.
No new TCP handshake. No new TLS handshake. That's the persistent-connection win.
```

### Step 3 — Send HEADERS + DATA
```
HEADERS frame → :path=/user.v1.UserService/GetUser, content-type=application/grpc,
                grpc-timeout=1S, plus any metadata (auth tokens, trace ids)
DATA frame    → the length-prefixed protobuf request
```

### Step 4 — Server dispatches
```
gRPC server reads :path, routes to the UserService.GetUser handler,
deserializes the protobuf bytes into a typed GetUserRequest{UserId: 42},
runs your business logic (DB lookup, etc.).
```

### Step 5 — Server responds on the same stream
```
HEADERS  → :status 200, content-type application/grpc
DATA     → length-prefixed protobuf User{id:42, name:"Alice", ...}
TRAILERS → grpc-status: 0 (OK), grpc-message: ""   ← END_STREAM closes the stream
```

### Step 6 — Deserialize (client stub)
```
Client reads DATA, deserializes protobuf → typed User struct.
Reads TRAILERS: grpc-status 0 → no error. Returns (user, nil) to your code.
If grpc-status were 5, the stub returns a NOT_FOUND error instead.
```

The whole point of steps 2–5: `GetUser` (stream 1), `GetOrder` (stream 3), and thousands of other calls all ride the **same** persistent HTTP/2 connection *concurrently* — TCP and TLS were handshaked once at startup, never per call.

### The four RPC types (all over HTTP/2 streams)

HTTP/2's long-lived streams are what make three of these possible at all:

```
1. UNARY  — one request, one response
   Client:  ──req──►
   Server:  ◄─resp──          rpc GetUser(Req) returns (Resp)

2. SERVER STREAMING — one request, many responses
   Client:  ──req──►
   Server:  ◄─r1─◄─r2─◄─r3─… (END_STREAM)
                              rpc ListFeatures(Rect) returns (stream Feature)

3. CLIENT STREAMING — many requests, one response
   Client:  ──r1──►──r2──►──r3──►… (END_STREAM)
   Server:  ◄────────resp────────
                              rpc UploadLog(stream Chunk) returns (Summary)

4. BIDIRECTIONAL STREAMING — both stream independently, same connection
   Client:  ──r1──►────►──r3──►
   Server:  ◄──s1───◄──s2───◄──s3──
                              rpc Chat(stream Msg) returns (stream Msg)
```

Each of these is exactly **one HTTP/2 stream**; a "stream" of messages is just multiple length-prefixed DATA frames on that stream until `END_STREAM`.

---

## Exact Syntax Breakdown

### The `.proto` file (schema-first — write this FIRST)

```proto
syntax = "proto3";              // always proto3 for modern gRPC
                                │
package user.v1;                // namespace; also part of the wire :path
                                │  → /user.v1.UserService/GetUser
                                │
message GetUserRequest {        // a message = a struct
  int32 user_id = 1;            //         ▲            ▲
}                               //         │            └─ FIELD NUMBER (on the wire!)
                                //         └─ field name (compile-time only)
message User {
  int32  id     = 1;
  string name   = 2;
  string email  = 3;
  bool   active = 4;
}

service UserService {           // a service = a collection of methods
  rpc GetUser(GetUserRequest) returns (User);
  //  ▲       ▲                        ▲
  //  │       │                        └─ response message type
  //  │       └─ request message type
  //  └─ method name → /user.v1.UserService/GetUser
}
```

### Generate typed code with `protoc`

```
protoc  --go_out=.  --go-grpc_out=.  user.proto
│       │           │                │
│       │           │                └── input .proto
│       │           └── generate gRPC client stub + server interface
│       └── generate Go structs for the messages
└── the protobuf compiler
```

This emits a **client stub** (`client.GetUser(...)` — you call it) and a **server interface** (you implement `GetUser(ctx, req)`).

### Invoke it from the CLI with `grpcurl`

`grpcurl` is `curl` for gRPC — it speaks the framing and trailers a browser/curl can't.

```
grpcurl  -plaintext  -d '{"user_id": 42}'  localhost:50051  user.v1.UserService/GetUser
│        │           │                     │                │
│        │           │                     │                └─ package.Service/Method
│        │           │                     └─ host:port (the gRPC server)
│        │           └─ request body as JSON (grpcurl encodes it to protobuf for you)
│        └─ no TLS (use h2c cleartext); drop this for a real TLS endpoint
└─ curl for gRPC
```

Schema evolution rules — the contract you must never break:

```proto
message User {
  int32  id    = 1;
  string name  = 2;
  // string email = 3;   ← field 3 was DELETED
  reserved 3;            //   → reserve the NUMBER so nobody reuses it
  reserved "email";      //   → and reserve the NAME too
  string phone = 4;      // ADD new fields with NEW numbers — safe & compatible
}
```
Rules: **add** fields with new numbers (safe). **Never change** a field's number or wire type. **Never reuse** a deleted number — `reserved` it. Renaming is fine on the wire (numbers are what count) but breaks JSON mapping / reflection, so treat it carefully.

---

## Example 1 — Basic

Stand up a tiny gRPC server and call it. We'll use a public demo plus your own.

**1. Define the contract** (`user.proto`, from the syntax section above).

**2. Run a server with reflection enabled** (reflection lets `grpcurl` discover methods without the `.proto`). Any minimal Go/Node/Python server works; here we just interact with it.

**3. List what the server exposes:**
```bash
grpcurl -plaintext localhost:50051 list
```
```
grpc.reflection.v1alpha.ServerReflection
user.v1.UserService
```

**4. Describe the service (the contract, straight from the server):**
```bash
grpcurl -plaintext localhost:50051 describe user.v1.UserService
```
```
user.v1.UserService is a service:
service UserService {
  rpc GetUser ( .user.v1.GetUserRequest ) returns ( .user.v1.User );
}
```

**5. Call the method:**
```bash
grpcurl -plaintext -d '{"user_id": 42}' localhost:50051 user.v1.UserService/GetUser
```
```json
{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com",
  "active": true
}
```

**6. See the raw HTTP/2 headers and trailers with `-v`** — this is where the gRPC wire model becomes visible:
```bash
grpcurl -plaintext -v -d '{"user_id": 42}' localhost:50051 user.v1.UserService/GetUser
```
```
Resolved method descriptor:
rpc GetUser ( .user.v1.GetUserRequest ) returns ( .user.v1.User );

Request metadata to send:
(empty)

Response headers received:
content-type: application/grpc          ← the response HEADERS frame
grpc-accept-encoding: gzip

Response contents:
{ "id": 42, "name": "Alice", ... }      ← the DATA frame (protobuf, shown as JSON)

Response trailers received:
grpc-status: 0                          ← THE actual status lives in the trailer
grpc-message:
Sent 1 request and received 1 response
```

Notice: `grpcurl` shows you *headers* then *contents* then *trailers* — a direct mirror of the HEADERS → DATA → TRAILERS frame sequence on the wire.

---

## Example 2 — Production Scenario

**The situation:** Your `checkout-svc` calls `pricing-svc` on every add-to-cart to recompute a cart total. It's an internal REST/JSON call over HTTP/1.1, and it fires **~40,000 times per second** at peak. Traces show the pricing call itself averages **9 ms** but the *network + serialization* overhead is dominating, and the JSON payloads are fat. You're asked to migrate this one hot path to gRPC and prove the win.

**Step 1 — Measure the REST baseline.**
```bash
# Payload size of the JSON request
echo -n '{"cart_id":"c_88213","items":[{"sku":"A1","qty":2},{"sku":"B7","qty":1}],"currency":"USD","customer_tier":"gold"}' | wc -c
```
```
112     ← 112 bytes per request, plus HTTP/1.1 request/response headers (~200+ bytes),
        ← plus a fresh connection or a keep-alive slot per concurrent call.
```

**Step 2 — Define the gRPC contract** (`pricing.proto`):
```proto
syntax = "proto3";
package pricing.v1;

message PriceRequest {
  string cart_id = 1;
  repeated Item items = 2;
  string currency = 3;
  string customer_tier = 4;
}
message Item { string sku = 1; uint32 qty = 2; }
message PriceResponse { uint64 total_cents = 1; string currency = 2; }

service Pricing {
  rpc Quote(PriceRequest) returns (PriceResponse);
}
```

**Step 3 — Verify the new endpoint works before flipping traffic:**
```bash
grpcurl -plaintext \
  -d '{"cart_id":"c_88213","items":[{"sku":"A1","qty":2},{"sku":"B7","qty":1}],"currency":"USD","customer_tier":"gold"}' \
  pricing-svc:50051 pricing.v1.Pricing/Quote
```
```json
{ "totalCents": "4750", "currency": "USD" }
```
The same request serialized as protobuf is **~48 bytes** vs 112 bytes of JSON, and header overhead is amortized by HPACK across one persistent HTTP/2 connection instead of repeated per call.

**Step 4 — Load test both, apples to apples.** Use `ghz` (a gRPC load tester, like `wrk` for gRPC):
```bash
ghz --insecure --proto ./pricing.proto --call pricing.v1.Pricing/Quote \
    -d '{"cart_id":"c_88213","items":[{"sku":"A1","qty":2}],"currency":"USD","customer_tier":"gold"}' \
    -c 50 -n 200000 pricing-svc:50051
```
```
Summary:
  Count:        200000
  Total:        4.10 s
  Requests/sec: 48780
Latency distribution:
  50% in 0.9 ms      ← was 2.8 ms on the REST/JSON path
  95% in 2.4 ms      ← was 7.1 ms
  99% in 4.6 ms      ← was 14 ms
Status code distribution:
  [OK]  200000 responses
```

**The measured result:**

| Metric (per Quote call)        | REST/JSON (HTTP/1.1) | gRPC (HTTP/2 + protobuf) |
|--------------------------------|----------------------|--------------------------|
| Request body size              | 112 B                | ~48 B                    |
| Header overhead per call       | ~200 B (repeated)    | ~few B (HPACK amortized) |
| Connections at 40k QPS         | large keep-alive pool| **1** multiplexed conn   |
| p50 latency (overhead only)    | 2.8 ms               | **0.9 ms**               |
| p99 latency                    | 14 ms                | **4.6 ms**               |
| CPU spent on (de)serialization | high (text parse)    | **low** (binary)         |

**Why it's faster (all four levers at once):** (1) binary protobuf is ~40% the bytes and far cheaper to parse than text JSON; (2) HTTP/2 multiplexes tens of thousands of concurrent calls over **one** persistent connection — no per-call TCP/TLS handshake, and HPACK compresses the repeated headers (Topic 11); (3) generated code does the serialization — no reflection-based JSON marshaling; (4) if pricing later needs to *stream* incremental quotes, the transport already supports it.

**The lesson:** this is the canonical gRPC win — a **chatty, high-QPS, internal** path. The public-facing `checkout` API stays REST/JSON; only the internal hop changed.

---

## Common Mistakes

### Mistake 1: Reusing or renumbering a protobuf field number
```proto
// v1 — shipped to production
message User { int32 id = 1;  string name = 2;  string email = 3; }

// v2 — someone "cleaned up" and reused number 3 for a NEW meaning
message User { int32 id = 1;  string name = 2;  bool is_admin = 3; }  // ☠️
```
**What breaks:** an old client sends `email = "a@b.com"` as *field 3, type LEN*. The new server reads *field 3* and tries to interpret those bytes as a bool. Silent data corruption — no error, wrong answer. **Fix:** field numbers are permanent. Deleted a field? `reserved 3;` and `reserved "email";` so nobody can ever reuse it. Add new meaning as a *new* number.

---

### Mistake 2: Trying to call gRPC directly from a browser
```javascript
// In browser JS — this will NOT work:
fetch("https://api.myapp.com/user.v1.UserService/GetUser", { method: "POST", body: ... })
// Browsers cannot: control HTTP/2 framing, read HTTP trailers, or send raw
// length-prefixed protobuf frames. There is no browser API for it.
```
**Fix:** use **gRPC-Web** (a browser-compatible variant that encodes trailers into the body) plus a proxy — **Envoy** with the `grpc_web` filter, or `grpcwebproxy` — that translates gRPC-Web ⇄ real gRPC. The browser talks gRPC-Web over HTTP/1.1/2; Envoy speaks real gRPC to your backend.
```
Browser ──gRPC-Web (HTTP/1.1)──► Envoy proxy ──gRPC (HTTP/2)──► your gRPC service
```

---

### Mistake 3: Using gRPC for a public or partner API
Your external customers expect to `curl` your API, read JSON, and paste it into Postman. gRPC gives them binary bytes, requires the `.proto` file, needs special tooling, and won't run in a browser. **Fix:** REST/JSON (Topic 15) at the edge for public/partner APIs; gRPC internally. Many teams run both — a REST gateway (grpc-gateway) that transcodes public JSON into internal gRPC.

---

### Mistake 4: Forgetting deadlines — calls hang forever
```go
// No deadline → if pricing-svc stalls, this call waits indefinitely,
// holding a goroutine and an HTTP/2 stream. Under load: cascading pileup.
resp, err := client.Quote(context.Background(), req)   // ☠️

// Always attach a deadline:
ctx, cancel := context.WithTimeout(context.Background(), 300*time.Millisecond)
defer cancel()
resp, err := client.Quote(ctx, req)   // sends grpc-timeout: 300m on the wire
```
The deadline is propagated as the `grpc-timeout` header and is enforced end to end; on expiry the call returns `DEADLINE_EXCEEDED` (`grpc-status: 4`) instead of hanging. **Deadlines are not optional in production.**

---

### Mistake 5: Assuming protobuf is human-readable / self-describing
```bash
# Point curl at a gRPC endpoint and you get binary garbage, not JSON:
curl https://pricing-svc:50051/pricing.v1.Pricing/Quote
# → unreadable bytes; no schema means no meaning.
```
Those bytes (`0A 05 41 6C…`) are meaningless without the `.proto`. **Fix:** use `grpcurl` (with reflection or `-proto`) to decode; keep `.proto` files versioned and shared (a schema registry / buf module). Never assume you can eyeball a captured protobuf frame.

---

### Mistake 6: Assuming one HTTP/2 connection makes packet loss irrelevant
HTTP/2 fixes *application-layer* head-of-line blocking — many streams share one connection with no per-stream queueing at L7. **But** they all ride **one TCP connection**, and TCP delivers bytes in order. A single lost segment stalls *every* multiplexed stream until it's retransmitted (TCP-level HOL blocking). On a clean data-center LAN this is negligible; over a **lossy/mobile link** it can bite. **Fix:** know this is exactly what HTTP/3 / QUIC solves by giving each stream independent delivery over UDP (Topics 11 and 12). For internal service mesh traffic on reliable networks, gRPC-over-HTTP/2 is fine.

---

## Hands-On Proof

```bash
# 0. Install the tools (macOS)
brew install grpcurl        # curl for gRPC
brew install ghz            # load tester for gRPC (optional)

# 1. Hit a public gRPC reflection server and list its services
grpcurl grpcb.in:9001 list
grpcurl grpcb.in:9000 list        # 9000 = plaintext, 9001 = TLS (varies by host)

# 2. Describe a service and a message type (the contract, from the server)
grpcurl grpcb.in:9001 describe grpcbin.GRPCBin
grpcurl grpcb.in:9001 describe .grpcbin.DummyMessage

# 3. Make a real unary call and see typed JSON come back
grpcurl -d '{"f_string": "hello"}' grpcb.in:9001 grpcbin.GRPCBin/DummyUnary

# 4. See the HTTP/2 headers + trailers (the grpc-status trailer!)
grpcurl -v -d '{"f_string": "hi"}' grpcb.in:9001 grpcbin.GRPCBin/DummyUnary

# 5. Prove it's really HTTP/2 protobuf, not JSON: point curl at it
curl -sv --http2 https://grpcb.in:9001/grpcbin.GRPCBin/DummyUnary | head
#   → you'll see http/2 negotiated, and a binary/garbled body. Not human-readable.

# 6. Trigger a non-OK gRPC status on purpose and read grpc-status
grpcurl -d '{"code": 5}' grpcb.in:9001 grpcbin.GRPCBin/SpecificError
#   → note: transport HTTP status is still 200; grpc-status carries the 5 (NOT_FOUND)

# 7. Prove the byte-size claim yourself
echo -n '{"name":"Alice","age":30}' | wc -c     # → 25 (JSON)
#   the protobuf equivalent is 9 bytes: 0A 05 41 6C 69 63 65 10 1E
```

---

## Practice Exercises

### Exercise 1 — Easy: Decode a protobuf message by hand
```
Given this schema:
    message Point { int32 x = 1;  int32 y = 2; }

And these raw wire bytes:  08 96 01 10 05

1. Split the bytes into (tag, value) pairs.
2. For each tag, compute field_number and wire_type from (field_number<<3)|wire_type.
3. Decode the varint value for field 1 (hint: 0x96 0x01 is a 2-byte varint —
   drop each MSB, reverse group order, combine).
4. What are x and y?

(Answer check: field 1 tag=0x08 → (1<<3)|0. 0x96=1001_0110 → 001_0110=22 with
 continuation; 0x01=0000_0001 → 1. value = 22 + (1<<7) = 150. field 2 tag=0x10,
 value 0x05 = 5.  So Point{x:150, y:5}.)
```

### Exercise 2 — Medium: Compare gRPC vs REST on the same data
```bash
# 1. Take this JSON and measure its byte size:
echo -n '{"order_id":"ord_10023","status":"SHIPPED","items":3,"total_cents":4750}' | wc -c

# 2. Sketch the protobuf .proto for the same message (4 fields, numbers 1-4).
#    Hand-encode the wire bytes for status="SHIPPED", items=3, total_cents=4750
#    and count them. (Strings: 1 tag + 1 length + N chars. Varints: 1 tag + 1-2 value.)

# 3. Call any public gRPC method with grpcurl -v and identify, in the output:
#      - which line corresponds to the HTTP/2 HEADERS frame
#      - which corresponds to the DATA frame
#      - which corresponds to the TRAILERS frame (grpc-status)
grpcurl -v -d '{"f_string":"x"}' grpcb.in:9001 grpcbin.GRPCBin/DummyUnary

# Questions:
#   a. Roughly what fraction of the JSON size is the protobuf?
#   b. Where does the REAL status code live — headers or trailers?
```

### Exercise 3 — Hard (Production Simulation): The backward-compatibility incident
```
Scenario: pricing-svc v1 shipped this contract, and 30 client services deployed it:

    message PriceResponse { uint64 total_cents = 1;  string currency = 2; }

A well-meaning engineer opens a PR for v2:

    message PriceResponse {
      uint64 total = 1;               // renamed total_cents -> total
      string currency = 2;
      string currency_code = 2;       // (typo — duplicate number, won't compile)
      uint32 discount_pct = 1;        // reused number 1 for a new field
    }

Your task:
  1. List every rule this PR violates.
  2. Explain, at the WIRE level, what an old client will decode when the new
     server sends discount_pct in field 1 while the client still expects
     total_cents in field 1. Why is there NO error?
  3. Rewrite the v2 .proto correctly: keep old fields intact, add discount_pct
     as a NEW field, and use `reserved` appropriately if anything is removed.
  4. Describe how you'd roll this out safely across 30 clients (hint: add-only
     changes are backward + forward compatible; deploy order doesn't matter).
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **gRPC did not invent a new transport protocol. What two existing technologies does it stand on, and what does each one provide?**

2. **On the protobuf wire, what identifies a field — its name or its number? Give one consequence of this for renaming a field and one consequence for reusing a field number.**

3. **A gRPC call succeeds at the transport level but the requested user doesn't exist. What HTTP status comes back, and where does the `NOT_FOUND` actually live?**

4. **Encode the number 42 and the string "Hi" as protobuf fields 1 (int32) and 2 (string). Write the bytes.** (Hint: tag = (field<<3)|wire_type; string is wire_type 2, length-prefixed.)

5. **You have 40,000 QPS from service A to service B. Name the four distinct reasons gRPC is faster here than REST/JSON.**

6. **Why can't a browser call a gRPC service directly, and what two pieces do you add to make it work?**

7. **HTTP/2 fixed head-of-line blocking — but gRPC can still suffer HOL blocking on a lossy link. At which layer, and which protocol (Topic 12) removes even that?**

---

## Quick Reference Card

| Concept | What it is | Key detail |
|---------|-----------|------------|
| gRPC | High-performance RPC framework | "HTTP/2 + protobuf + a calling convention" |
| Protocol Buffers | Schema + binary serialization | `.proto` → `protoc` → typed code |
| Wire tag | Field key on the wire | `(field_number << 3) \| wire_type` (a varint) |
| Wire types | 0=VARINT, 1=I64, 2=LEN, 5=I32 | LEN = strings/bytes/messages (length-prefixed) |
| Varint | Variable-length integer | 7 data bits/byte, high bit = "more bytes" |
| Field number | Identity on the wire | Permanent — never reuse/renumber; `reserved` deletes |
| :path | HTTP/2 pseudo-header | `/package.Service/Method` |
| content-type | gRPC marker | `application/grpc` |
| grpc-status | Real result code | In the **trailer**, not the HTTP status |
| grpc-timeout | Deadline on the wire | e.g. `1S`, `300m` (ms) |
| Length-prefix | gRPC message framing | 1 flag byte + 4-byte length + protobuf |

**gRPC vs REST/JSON:**

| Dimension | gRPC | REST/JSON |
|-----------|------|-----------|
| Payload size | Small (binary) | Larger (text) |
| (De)serialize speed | Fast (generated) | Slower (text parse) |
| Transport | HTTP/2 only | HTTP/1.1 or /2 |
| Streaming | 4 modes (incl. bidi) | Request/response only |
| Browser support | No (needs gRPC-Web + proxy) | Native |
| Human-readable | No (need `.proto`) | Yes |
| Schema/contract | Enforced (`.proto`) | Optional (OpenAPI) |
| Best for | Internal service-to-service | Public/partner APIs |

**gRPC status codes (grpc-status enum):**
```
0  OK                 5  NOT_FOUND           10 ABORTED
1  CANCELLED          6  ALREADY_EXISTS      11 OUT_OF_RANGE
2  UNKNOWN            7  PERMISSION_DENIED   12 UNIMPLEMENTED
3  INVALID_ARGUMENT   8  RESOURCE_EXHAUSTED  13 INTERNAL
4  DEADLINE_EXCEEDED  9  FAILED_PRECONDITION 14 UNAVAILABLE  16 UNAUTHENTICATED
```

**Key commands:**
```bash
protoc --go_out=. --go-grpc_out=. svc.proto          # generate code
grpcurl -plaintext HOST:PORT list                     # list services (reflection)
grpcurl -plaintext HOST:PORT describe pkg.Service      # show the contract
grpcurl -plaintext -d '{...}' HOST:PORT pkg.Svc/Method # call a method
grpcurl -v -d '{...}' HOST:PORT pkg.Svc/Method         # + headers & trailers
ghz --insecure --proto s.proto --call pkg.Svc.M -c 50 -n 100000 HOST:PORT  # load test
```

---

## When Would I Use This at Work?

### Scenario 1: "Our internal services are chatty and CPU is spent parsing JSON"
Ten microservices calling each other thousands of times per second, and profiling shows JSON marshal/unmarshal near the top. This is the textbook gRPC migration: define `.proto` contracts, generate clients, keep the public edge on REST. You'll cut payload bytes, serialization CPU, and connection count (one multiplexed HTTP/2 conn instead of a keep-alive pool) — exactly the Example 2 win.

### Scenario 2: "The frontend needs real-time updates from a gRPC service"
The browser can't speak raw gRPC. You stand up **Envoy** with the `grpc_web` filter (or `grpc-gateway` to transcode to REST/JSON for simple cases). The browser talks gRPC-Web; Envoy translates to real gRPC for the backend. For server-push you use a server-streaming RPC behind gRPC-Web.

### Scenario 3: "A deploy broke deserialization in a downstream service"
Someone renumbered or reused a protobuf field. You now know to check `git diff` on the `.proto` for changed/reused field numbers, know that the failure is *silent* (wrong values, not an exception), and know the fix: revert to add-only changes, `reserved` the abandoned numbers, and never renumber. Add a `buf breaking` check to CI so this can't merge again.

### Scenario 4: "gRPC calls intermittently hang under load"
Missing deadlines. Calls with no `grpc-timeout` wait forever, pile up streams and goroutines, and cascade into an outage. You add per-call deadlines end to end and set `UNAVAILABLE`/`DEADLINE_EXCEEDED` retry policies. You also verify it isn't TCP-level HOL blocking on a flaky link (Topics 11/12).

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the packet/TCP/TLS foundation every call rides on
- `11-http2.md` — multiplexing, streams, HPACK, binary framing: gRPC IS built on this

**Study alongside / after:**
- `12-http3-and-quic.md` — why TCP-level HOL blocking still matters and how QUIC removes it (gRPC-over-HTTP/3 is emerging)
- `15-rest-over-http.md` — the resource model gRPC's RPC model contrasts with; still the right choice at the public edge
- `36-service-to-service-communication.md` — internal DNS, service mesh (Envoy/Istio), and mTLS: the network gRPC lives inside
- `09-tls-ssl-in-depth.md` — mTLS is the norm for internal gRPC in a zero-trust mesh

**gRPC is the practical culmination of the HTTP/2 chapter:** it takes multiplexing, header compression, and streaming from Topic 11 and turns them into a typed, contract-first way for your services to call each other like functions.

---

*Doc saved: `/docs/networking/37-grpc-and-protocol-buffers.md`*
