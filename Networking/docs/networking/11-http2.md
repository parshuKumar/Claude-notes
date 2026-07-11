# 11 — HTTP/2

> **Phase 2 — Topic 6 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a single-lane road (**one TCP connection**) between your house and a warehouse.

**HTTP/1.1** is like sending one delivery truck down that road at a time. The truck drives to the warehouse, gets loaded with your first order, drives back, and *only then* can the next truck leave for your second order. If the first order is a giant slow-to-pack sofa, every other order — even a tiny pack of screws — waits behind it. To go faster, browsers cheat: they build **six separate roads** (six TCP connections) so six trucks can run at once. But building roads is expensive, and six is the hard limit.

**HTTP/2** rebuilds that one road as a **multi-lane highway**. Now hundreds of small labeled trucks drive down the *same* road at the same time, interleaved bumper to bumper. Each truck carries a **label (stream ID)** so the warehouse knows which order each piece belongs to. The screws don't wait behind the sofa — they ride alongside it. And instead of every truck carrying a fat repeated paperwork folder ("here's my name, my address, my account number" — the same on every trip), HTTP/2 says "you already have my folder from last time, just reference folder #3" (**header compression**).

One road. Many trucks at once. Shorter paperwork. That's HTTP/2.

The one catch: it's still a *single physical road*. If a pothole (**a lost packet**) blocks the road, *every* truck behind it stops — even though they're carrying unrelated orders. That last problem is what HTTP/3 goes on to fix.

---

## Where This Lives in the Network Stack

HTTP/2 is still an **application-layer (Layer 7)** protocol — same layer as HTTP/1.1. It does **not** change TCP, IP, or TLS. It only changes how HTTP messages are *serialized* onto the connection.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   HTTP/2  ← binary framing + multiplexing │  ← THIS TOPIC
│                          (HPACK header compression lives here)   │
│  Layer 6 — Presentation  TLS 1.2/1.3  ← ALPN negotiates "h2"     │  ← how browsers pick HTTP/2
│  Layer 4 — Transport     TCP  ← ONE connection, still ordered    │  ← the remaining HOL flaw
│  Layer 3 — Network       IP                                      │
│  Layer 2 — Data Link     Ethernet / Wi-Fi                        │
│  Layer 1 — Physical      fiber / copper / radio                  │
└─────────────────────────────────────────────────────────────────┘
```

The critical insight: **HTTP/2 is a new way to write bytes at Layer 7, sitting on the exact same Layer 4 TCP connection you already understand.** Everything below HTTP is unchanged. That's why upgrading to HTTP/2 usually requires *zero application code changes* — your `GET /users` still means `GET /users`. Only the wire format underneath changed.

---

## What Is This?

HTTP/2 (RFC 7540, published 2015; revised as RFC 9113 in 2022) is a major revision of the HTTP protocol whose entire purpose is **efficiency on a single connection**. It keeps all of HTTP's semantics — methods (`GET`, `POST`), status codes (`200`, `404`), headers, URLs, cookies — completely identical. What it changes is the **transport of those semantics onto the wire.**

Four big changes define HTTP/2:

1. **Binary framing** — messages are no longer plaintext lines terminated by `\r\n`. They're packed into binary **frames** with a fixed 9-byte header. Machines parse binary far faster and more safely than text.
2. **Multiplexing** — one TCP connection carries many concurrent **streams** (independent request/response pairs). Frames from different streams are interleaved on the wire. This kills HTTP-level head-of-line blocking.
3. **HPACK header compression** — repeated headers (`cookie`, `user-agent`, `accept`) are compressed against a shared table so they aren't re-sent in full on every request.
4. **Stream prioritization & flow control** — the client can hint which resources matter most, and both sides throttle data with per-stream and connection-level credit.

It was born from Google's experimental **SPDY** protocol (2009–2015). HTTP/2 is essentially standardized, cleaned-up SPDY. In browsers it is **almost always run over TLS** and negotiated via **ALPN** as the protocol name `"h2"`.

---

## Why Does It Matter for a Backend Developer?

Because it changes how your server behaves under load — often invisibly — and because "just turn it on" is frequently the single highest-leverage latency win for a page that makes many requests.

Concretely, you need HTTP/2 knowledge to:

- **Stop over-provisioning connections.** Under HTTP/1.1, browsers open 6 TCP connections per origin, each with its own TLS handshake. Under HTTP/2, they use **one**. That's 6× fewer handshakes, sockets, and slow-start ramps on your server.
- **Kill old "optimizations" that are now anti-patterns.** Domain sharding, sprite sheets, inlining assets, and concatenating 50 JS files into one bundle were all HTTP/1.1 workarounds. Under HTTP/2 they *hurt*.
- **Configure your reverse proxy correctly.** Enabling `http2` on nginx/Envoy/ALB is a one-line change, but understanding what it does (and doesn't) fix determines whether you also need HTTP/3.
- **Debug "it's fast in the office, slow on mobile."** HTTP/2 multiplexing over one TCP connection means a single lost packet on a lossy mobile network stalls *every* in-flight request. This is TCP-level head-of-line blocking, and it's the reason HTTP/2 can be *slower* than HTTP/1.1 on bad networks.
- **Understand gRPC.** gRPC runs *on top of* HTTP/2 and depends directly on its streams and framing (Topic 37). You can't reason about gRPC without this.

---

## The Packet/Protocol Anatomy

### The core hierarchy: Connection → Stream → Message → Frame

```
ONE TCP CONNECTION  (survives for the whole session)
│
├── Stream 1   ── one HTTP request/response  ← client-initiated (ODD id)
├── Stream 3   ── one HTTP request/response  ← client-initiated (ODD id)
├── Stream 5   ── one HTTP request/response  ← client-initiated (ODD id)
├── Stream 2   ── a server PUSH             ← server-initiated (EVEN id)
└── Stream 0   ── connection control        ← SETTINGS, PING, WINDOW_UPDATE, GOAWAY
        │
        └── each Stream carries MESSAGES (a request is one message, a response another)
                │
                └── each Message is split into FRAMES (HEADERS, DATA, …)
                        │
                        └── each Frame is 9 bytes of header + a payload
```

- **Stream IDs are never reused** on a connection and only increase. Client streams are **odd** (1, 3, 5…), server-pushed streams are **even** (2, 4…), and **stream 0** is reserved for connection-wide control frames.

### The HTTP/2 frame layout (the atom of the protocol)

Every frame — no exceptions — begins with the same **9-byte header**:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-----------------------------------------------+
|                 Length (24)                   |  ← payload size in bytes (max 16,384 by default)
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |                  ← what kind of frame + per-type flag bits
+-+-------------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                     |  ← which stream this frame belongs to
+=+=============================================================+
|                   Frame Payload (0 .. Length)               ...  ← the actual bytes (headers, body, etc.)
+---------------------------------------------------------------+

R = 1 reserved bit (always 0).  Header is exactly 9 bytes. Payload follows.
```

Field-by-field:

| Field | Size | Meaning |
|-------|------|---------|
| Length | 24 bits | Payload length in bytes. Capped at `SETTINGS_MAX_FRAME_SIZE` (default 16,384 = 2¹⁴). |
| Type | 8 bits | Frame type (see table below). |
| Flags | 8 bits | Type-specific booleans, e.g. `END_STREAM`, `END_HEADERS`. |
| R | 1 bit | Reserved, must be 0. |
| Stream Identifier | 31 bits | The stream this frame belongs to. 0 = connection-level control. |
| Payload | variable | The frame's data. |

### The frame types you must know

```
Type  Name           Hex    What it does
────  ─────────────  ────   ─────────────────────────────────────────────
 0    DATA           0x0    Request/response BODY bytes. Flow-controlled.
 1    HEADERS        0x1    Opens a stream + carries HPACK-compressed headers.
 2    PRIORITY       0x2    Declares a stream's dependency/weight (deprecated in 9113).
 3    RST_STREAM     0x3    Abruptly cancel ONE stream (without killing the connection).
 4    SETTINGS       0x4    Connection config (max streams, window size, frame size…).
 5    PUSH_PROMISE   0x5    Server announces it will push a resource (largely dead).
 6    PING           0x6    Round-trip / keepalive probe. Echoed back by peer.
 7    GOAWAY         0x7    Graceful shutdown: "stop opening streams, I'm closing."
 8    WINDOW_UPDATE  0x8    Flow control: "I can accept N more bytes."
 9    CONTINUATION   0x9    Extra HEADERS bytes when they don't fit in one frame.
```

### One request as frames

A simple `GET /users` with a JSON response looks like this on the wire:

```
Client → Server:  HEADERS frame, stream 1, flags=END_STREAM|END_HEADERS
                  (payload = HPACK-encoded :method GET, :path /users, …)
                  ← no body, so END_STREAM closes the request half immediately

Server → Client:  HEADERS frame, stream 1, flags=END_HEADERS
                  (payload = HPACK-encoded :status 200, content-type …)
Server → Client:  DATA frame,    stream 1, flags=0           ← first chunk of JSON
Server → Client:  DATA frame,    stream 1, flags=END_STREAM  ← last chunk; stream closes
```

Notice: **no `\r\n`, no plaintext request line, no blank-line-terminates-headers.** The `END_STREAM` and `END_HEADERS` *flags* replace all the text delimiters HTTP/1.1 relied on.

---

## How It Works — Step by Step

Let's trace a browser loading `https://shop.example.com/` (which needs the HTML, one CSS file, and two images) over HTTP/2.

### Step 1 — TCP + TLS with ALPN negotiation

```
1a. TCP 3-way handshake to shop.example.com:443           (Topic 08)
1b. TLS ClientHello includes ALPN extension:
        "I speak: h2, http/1.1"                            ← the protocol offer
1c. TLS ServerHello selects ALPN:
        "Let's use: h2"                                    ← HTTP/2 chosen HERE, during TLS
1d. TLS handshake completes (Topic 09)
```

ALPN (Application-Layer Protocol Negotiation) is how HTTP/2 gets chosen **without an extra round trip**. The protocol name is piggybacked inside the TLS handshake. If the server doesn't offer `h2`, the client silently falls back to `http/1.1`.

### Step 2 — The connection preface

```
2a. Client sends the magic preface bytes (a fixed 24-byte string):
        PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
    (This confirms "yes, I really mean HTTP/2" and cannot be produced by an HTTP/1.x parser by accident.)
2b. Client sends its first SETTINGS frame on stream 0.
2c. Server sends its own SETTINGS frame on stream 0.
2d. Both sides ACK each other's SETTINGS.
```

Now the connection is live and multiplexing can begin.

### Step 3 — Fire ALL requests at once (multiplexing)

```
The browser doesn't wait. It opens four streams immediately:

  HEADERS stream 1  → GET /            (the HTML)
  HEADERS stream 3  → GET /style.css
  HEADERS stream 5  → GET /hero.jpg
  HEADERS stream 7  → GET /logo.png

All four go out back-to-back on the SAME TCP connection.
Under HTTP/1.1 this would have needed 4 connections (or serialized on 1).
```

### Step 4 — Server responds with interleaved frames

Here is the heart of HTTP/2. The server streams all four responses back **at the same time**, interleaving their frames:

```
TIME ──────────────────────────────────────────────────────────────►

Wire:  [H:1][D:1][H:3][D:5][D:1][D:3][H:5][D:5][H:7][D:1][D:7][D:5]...
        │    │    │    │    │    │    │    │    │    │    │    │
        │    │    │    │    │    │    │    │    │    │    │    └ img1 body chunk
        │    │    │    │    │    │    │    │    │    │    └ png body chunk
        │    │    │    │    │    │    │    │    │    └ html body chunk
        │    │    │    │    │    │    │    │    └ png headers
        │    │    │    │    │    │    │    └ img1 body chunk
        │    │    │    │    │    │    └ img1 headers
        │    │    │    │    │    └ css body chunk
        │    │    │    │    └ html body chunk
        │    │    │    └ img1 body chunk
        │    │    └ css headers
        │    └ html body chunk
        └ html headers

  H:n = HEADERS frame for stream n     D:n = DATA frame for stream n

The receiver DEMULTIPLEXES by stream ID: all D:1 frames reassemble into
the HTML, all D:3 frames into the CSS, and so on — independently.
```

The tiny `logo.png` doesn't have to wait behind the big `hero.jpg`. Its frames slot into the gaps. **That is the fix for HTTP-level head-of-line blocking.**

### Step 5 — HPACK keeps the headers tiny

The second, third, and fourth requests share almost all their headers with the first (`:authority`, `user-agent`, `accept-encoding`, `cookie`…). HPACK sends those as **1–2 byte table references** instead of repeating hundreds of bytes. (Full detail in the next section.)

### Step 6 — Flow control keeps a fast stream from drowning a slow one

```
The browser advertises a receive window per stream and for the whole connection.
As it consumes DATA frames, it sends WINDOW_UPDATE frames back:
    WINDOW_UPDATE stream 5, increment=65535   ← "send me 64KB more of hero.jpg"
    WINDOW_UPDATE stream 0, increment=65535   ← "…and 64KB more on the connection overall"
```

### Step 7 — Streams close independently; the connection stays open

```
Each stream ends when a frame with END_STREAM arrives. Stream 7 can finish
while stream 5 is still downloading. The TCP connection stays ESTABLISHED,
ready for the next click — no new handshake needed.
When the tab closes, the client sends GOAWAY and the connection tears down.
```

---

## Exact Syntax Breakdown

### `curl --http2 -v https://example.com/`

```
curl  --http2   -v        https://example.com/
│     │         │         │
│     │         │         └── URL (https ⇒ ALPN will offer h2)
│     │         └── verbose: show TLS, ALPN, and per-stream frame info
│     └── request HTTP/2 over TLS (falls back to 1.1 if server refuses)
└── the transfer tool
```

Related flags you'll actually use:

| Command / flag | What it does |
|----------------|--------------|
| `curl --http2 URL` | Use HTTP/2 **over TLS**, negotiated via ALPN. Falls back to 1.1. |
| `curl --http2-prior-knowledge URL` | Use **cleartext h2c** with no upgrade — assumes the server speaks HTTP/2. |
| `curl -w "%{http_version}\n" -o /dev/null -s URL` | Print the negotiated version (`2`, `1.1`, `3`). |
| `curl --http1.1 URL` | Force HTTP/1.1 (useful for A/B comparison). |
| `nghttp -v URL` | Dedicated HTTP/2 client; prints every frame sent/received. |
| `nghttp -ns URL` | `-n` discard body, `-s` print per-stream timing stats. |

### What `-v` shows you (the ALPN moment)

```
*   Trying 93.184.216.34:443...
* Connected to example.com (93.184.216.34) port 443
* ALPN: curl offers h2,http/1.1        ← client's protocol menu, sent inside ClientHello
* TLSv1.3 (IN), TLS handshake, ...
* ALPN: server accepted h2             ← THE decision: server picked HTTP/2
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://example.com/
* [HTTP/2] [1] [:method: GET]          ← pseudo-headers (note the leading colon)
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: example.com]
* [HTTP/2] [1] [:path: /]
* [HTTP/2] [1] [user-agent: curl/8.4.0]
> GET / HTTP/2                          ← curl reconstructs a readable view for you
> Host: example.com
>
< HTTP/2 200                            ← note: "HTTP/2 200", no reason phrase ("OK") anymore
< content-type: text/html
<
```

Two details worth burning in:
- **Pseudo-headers** (`:method`, `:scheme`, `:authority`, `:path`, and on responses `:status`) replace HTTP/1.1's request line and status line. They always start with `:` and must come *before* normal headers.
- **No reason phrase.** HTTP/1.1 sent `HTTP/1.1 200 OK`. HTTP/2 sends only `:status: 200`. The `OK` is gone.

---

## Example 1 — Basic

Prove your machine can negotiate HTTP/2 and see the version it lands on.

```bash
# Ask for HTTP/2 and print ONLY the negotiated protocol version
curl --http2 -o /dev/null -s -w "Negotiated: HTTP/%{http_version}\n" https://www.cloudflare.com/

# Expected:
# Negotiated: HTTP/2
```

Now watch the ALPN handshake actually happen:

```bash
curl --http2 -v -o /dev/null -s https://www.cloudflare.com/ 2>&1 | grep -i alpn

# Expected:
# * ALPN: curl offers h2,http/1.1
# * ALPN: server accepted h2
```

Compare against a forced downgrade so you can *see* the difference:

```bash
# Force HTTP/1.1 on the same server
curl --http1.1 -o /dev/null -s -w "Negotiated: HTTP/%{http_version}\n" https://www.cloudflare.com/
# Negotiated: HTTP/1.1

# Force HTTP/2
curl --http2  -o /dev/null -s -w "Negotiated: HTTP/%{http_version}\n" https://www.cloudflare.com/
# Negotiated: HTTP/2
```

**What you just proved:** the *same server, same URL, same request* can run over either protocol. The choice happens at ALPN time inside TLS, and nothing about your request semantics changed — only the wire encoding.

---

## Example 2 — Production Scenario

**The situation:** Your product page `shop.example.com/catalog` loads 40 small assets — a stylesheet, some JS chunks, and ~35 tiny product thumbnail images. Users complain it "feels slow to fully render." Your nginx origin is currently HTTP/1.1 only. You suspect connection contention.

### Step 1 — Confirm you're on HTTP/1.1 and measure

```bash
curl -o /dev/null -s -w "version=%{http_version}  total=%{time_total}s\n" \
  https://shop.example.com/catalog
# version=1.1  total=1.84s
```

With HTTP/1.1, the browser can only run **6 parallel connections** to your origin. 40 assets ÷ 6 lanes = a lot of queuing. Each of those 6 connections also paid for its own TLS handshake.

### Step 2 — Enable HTTP/2 on nginx (the entire change)

```nginx
# /etc/nginx/sites-available/shop.example.com
server {
    listen 443 ssl;          # BEFORE
    listen 443 ssl http2;    # AFTER — add the http2 token. (nginx ≥ 1.25.1 uses:  http2 on;)

    server_name shop.example.com;
    ssl_certificate     /etc/letsencrypt/live/shop.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shop.example.com/privkey.pem;

    root /var/www/shop;
    # ... your existing config is UNCHANGED. No app code touched. ...
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx     # test config, then hot-reload
```

That is the *whole* backend change. Your application code, routes, and handlers are untouched — HTTP/2 is transparent to them.

### Step 3 — Verify and re-measure

```bash
curl --http2 -o /dev/null -s -w "version=%{http_version}  total=%{time_total}s\n" \
  https://shop.example.com/catalog
# version=2  total=0.71s
```

Use `nghttp` to *see* the multiplexing — one connection, all 40 assets as interleaved streams:

```bash
nghttp -ans https://shop.example.com/catalog
#                                    id  responseEnd requestStart  ...  path
# id  responseEnd  requestStart  process  code  size  path
#  13     +71.2ms        +0.5ms   70.7ms   200   4.2K  /catalog
#  15     +88.1ms        +0.6ms   87.5ms   200   1.1K  /style.css
#  17     +95.3ms        +0.6ms   94.7ms   200   820   /app.js
#  19    +102.4ms        +0.7ms  101.7ms   200   3.1K  /img/thumb-01.jpg
#  ...            (all on ONE connection, requestStart nearly identical — fired together)
#  91    +140.9ms        +1.9ms  139.0ms   200   2.8K  /img/thumb-35.jpg
```

Notice every `requestStart` is within ~2ms — all 40 requests left essentially **at once**, over **one** connection. Page total dropped from **1.84s → 0.71s**.

### Step 4 — The counter-example: when HTTP/2 does *not* help

You deploy the same change and a colleague on a flaky hotel Wi-Fi (5% packet loss) reports it's *no faster, maybe worse*. Simulate it:

```bash
# Add 200ms latency + 5% loss to your local interface (Linux; requires root)
sudo tc qdisc add dev eth0 root netem delay 200ms loss 5%

# Now compare — same page, forced 1.1 vs forced h2, over the LOSSY link
curl --http1.1 -o /dev/null -s -w "h1.1 total=%{time_total}s\n" https://shop.example.com/catalog
curl --http2   -o /dev/null -s -w "h2   total=%{time_total}s\n" https://shop.example.com/catalog

# Clean up:
sudo tc qdisc del dev eth0 root
```

On the lossy link, HTTP/2 can lose its advantage. **Why:** all 40 streams share **one** TCP connection. A single dropped TCP segment forces TCP to hold back *every* byte behind it — for *all* streams — until the retransmission arrives. HTTP/1.1's 6 independent connections localize a loss to just one of six lanes.

**The lesson:** HTTP/2 wins big on clean networks with many small requests. On lossy networks it hits **TCP-level head-of-line blocking**, which HTTP/2 *cannot* fix because the flaw lives in TCP, one layer down. That limitation is the entire reason HTTP/3 exists (Topic 12).

---

## Common Mistakes

### Mistake 1: "We need to rewrite our app to use HTTP/2"

```
Belief:  HTTP/2 is a new API my code must call.
Reality: HTTP/2 changes only the WIRE FORMAT below your app. Your
         GET /users, your JSON, your status codes are byte-for-byte
         identical in meaning. You flip a switch on nginx / the load
         balancer / the framework — ZERO handler code changes.
```

Enabling HTTP/2 is a config/infrastructure task, not an application rewrite. The framing, HPACK, and multiplexing are handled entirely by the server and client libraries.

---

### Mistake 2: Still doing domain sharding (now an anti-pattern)

```
HTTP/1.1 era trick:  split assets across img1.cdn.com, img2.cdn.com,
                     img3.cdn.com so the browser opens 6 connections
                     PER shard → more parallelism.

Under HTTP/2 this BACKFIRES:
  - Each extra hostname = a separate TCP connection + separate TLS
    handshake + separate HPACK table + separate congestion window.
  - You DEFEAT multiplexing, which wanted everything on ONE connection.
  - You also lose HPACK's cross-request header savings.
```

**Fix:** serve everything from **one origin** so the browser coalesces it onto a single multiplexed HTTP/2 connection. The same applies to concatenating/inlining assets — largely unnecessary and often counterproductive under HTTP/2.

---

### Mistake 3: Betting on HTTP/2 Server Push

```
The dream:  Server proactively PUSHES /style.css and /app.js alongside
            the HTML via PUSH_PROMISE, before the browser even asks.

The reality (2026):
  - The server can't know what the client already has CACHED, so it
    routinely pushes bytes the client didn't need → wasted bandwidth.
  - Prioritization interactions made it easy to push the WRONG thing first.
  - Chrome REMOVED HTTP/2 push support in Chrome 106 (Sept 2022).
    Firefox and most CDNs followed. It is effectively DEAD.
```

**Fix:** don't build on server push. For the "hint the browser early" use case, use **`103 Early Hints`** with `Link: rel=preload` headers, or plain `<link rel="preload">` in your HTML. Assume push is unavailable.

---

### Mistake 4: Believing HTTP/2 eliminates ALL head-of-line blocking

```
HTTP/2 fixes HOL blocking at the HTTP/APPLICATION layer:
  ✓ A slow response no longer blocks other responses (streams are independent).

HTTP/2 does NOT fix HOL blocking at the TCP/TRANSPORT layer:
  ✗ All streams ride ONE TCP connection. TCP guarantees in-order bytes.
    One lost segment → TCP stalls EVERY stream until it's retransmitted.
```

On a network with meaningful packet loss, this TCP-level stall can make HTTP/2 perform *worse* than HTTP/1.1's parallel connections. **Fix:** there is no HTTP/2 fix — this is precisely what HTTP/3/QUIC (independent streams over UDP) was designed to solve (Topic 12).

---

### Mistake 5: Assuming HTTP/2 always means encrypted, or that h2c is a drop-in

```
Spec-wise:  HTTP/2 can run in CLEARTEXT ("h2c") — via an Upgrade: h2c
            header, or with --http2-prior-knowledge.
In practice: NO major browser will ever speak h2c. Browsers require TLS
            and negotiate "h2" via ALPN. h2c only appears server-to-server
            (e.g., a load balancer to an internal service on a trusted net).
```

**Fix:** for anything a browser touches, HTTP/2 = TLS + ALPN. Reach for `h2c` only on internal, trusted links where you deliberately configured it.

---

## Hands-On Proof

```bash
# 1. Confirm a site serves HTTP/2 and see the ALPN negotiation
curl --http2 -v -o /dev/null -s https://www.google.com/ 2>&1 | grep -iE "alpn|using http"
#   * ALPN: curl offers h2,http/1.1
#   * ALPN: server accepted h2
#   * using HTTP/2

# 2. Print just the negotiated version for several sites
for u in https://www.cloudflare.com https://www.google.com https://example.com; do
  curl --http2 -o /dev/null -s -w "%{http_version}  $u\n" "$u"
done

# 3. See EVERY frame on the wire with nghttp (install: brew install nghttp2 / apt install nghttp2-client)
nghttp -v https://nghttp2.org/ 2>&1 | head -40
#   look for:  send SETTINGS frame   / send HEADERS frame <... stream_id=13>
#              recv HEADERS frame    / recv DATA frame <... stream_id=13>

# 4. Watch multiplexing: many assets, ONE connection, interleaved streams
nghttp -ans https://nghttp2.org/
#   All requestStart times cluster together = fired concurrently on one connection.

# 5. Force HTTP/1.1 vs HTTP/2 on the same URL and compare total time
curl --http1.1 -o /dev/null -s -w "h1.1: %{time_total}s\n" https://nghttp2.org/
curl --http2   -o /dev/null -s -w "h2  : %{time_total}s\n" https://nghttp2.org/

# 6. Inspect the raw TLS ALPN offer with openssl
echo | openssl s_client -alpn h2 -connect www.cloudflare.com:443 2>/dev/null | grep -i "ALPN"
#   ALPN protocol: h2
```

---

## Practice Exercises

### Exercise 1 — Easy: Confirm and contrast the protocol

```bash
# Pick any three big sites and record which HTTP version each negotiates.
for u in https://www.cloudflare.com https://github.com https://www.wikipedia.org; do
  curl --http2 -o /dev/null -s -w "%{http_version}  <- $u\n" "$u"
done

# Answer:
# 1. Which sites negotiated HTTP/2? Did any give you HTTP/1.1?
# 2. Re-run one with --http1.1. Does the SAME server still answer? What does that
#    tell you about whether HTTP/2 "replaces" HTTP/1.1 or coexists with it?
# 3. In one sentence, where in the connection setup did the version get decided?
```

### Exercise 2 — Medium: See the streams and HPACK savings

```bash
# Use nghttp's verbose mode against a page with several assets.
nghttp -v https://nghttp2.org/ 2>&1 | grep -E "send HEADERS|recv HEADERS|stream_id" | head -20

# Answer:
# 1. List the stream_ids you see the client OPEN. Are they odd or even? Why?
# 2. Find a SETTINGS frame. What is stream_id for SETTINGS, and why that value?
# 3. In the HEADERS frames, identify the pseudo-headers (the ones starting with ":").
#    Which four appear on a request? Which one appears on a response?
# 4. Explain in your own words how HPACK makes the SECOND request's headers
#    smaller than the first request's headers.
```

### Exercise 3 — Hard (Production Simulation): Prove TCP head-of-line blocking

```bash
# Goal: demonstrate that HTTP/2's advantage collapses under packet loss,
#       because all streams share one TCP connection.

# On a Linux box (VM is fine), pick a test page that loads many small assets.
# You'll compare HTTP/1.1 vs HTTP/2 on a CLEAN link, then a LOSSY link.

# --- CLEAN network baseline ---
curl --http1.1 -o /dev/null -s -w "clean h1.1: %{time_total}s\n" https://YOUR_TEST_PAGE/
curl --http2   -o /dev/null -s -w "clean h2  : %{time_total}s\n" https://YOUR_TEST_PAGE/

# --- Add 150ms latency + 4% packet loss ---
sudo tc qdisc add dev eth0 root netem delay 150ms loss 4%

curl --http1.1 -o /dev/null -s -w "lossy h1.1: %{time_total}s\n" https://YOUR_TEST_PAGE/
curl --http2   -o /dev/null -s -w "lossy h2  : %{time_total}s\n" https://YOUR_TEST_PAGE/

# --- Clean up ---
sudo tc qdisc del dev eth0 root

# Questions:
# 1. On the CLEAN link, which protocol was faster? By how much?
# 2. On the LOSSY link, did HTTP/2's advantage shrink, vanish, or reverse?
# 3. Explain WHY using the words "one TCP connection" and "in-order delivery."
# 4. What protocol solves this, and what transport does it move to? (Topic 12)
```

---

## Mental Model Checkpoint

Answer from memory. If you stumble, re-read the linked section.

1. **HTTP/1.1 vs HTTP/2 on the same request** — you send `GET /users` over each. What is *identical* between the two, and what is *different*? (Hint: semantics vs. wire format.)

2. **Multiplexing** — how does HTTP/2 send 40 requests concurrently over a single TCP connection when HTTP/1.1 could only do 6 at a time (across 6 connections)? Name the unit that gets interleaved and the number that tells the receiver which response each piece belongs to.

3. **The frame header** — how many bytes is every HTTP/2 frame header, and name three of its fields. Which field distinguishes a `HEADERS` frame from a `DATA` frame?

4. **HPACK** — you send 50 requests to the same host, each with an identical 800-byte `Cookie` header. Roughly how many times does the full 800-byte cookie string travel across the wire, and why?

5. **Server push** — what was PUSH_PROMISE meant to do, and what is its status in real browsers today? What should you use instead?

6. **The remaining flaw** — HTTP/2 fixes head-of-line blocking at which layer, but leaves it unsolved at which layer? Explain what a single lost TCP segment does to 40 concurrent streams and why.

7. **Negotiation** — a browser and server both support HTTP/2. At what exact moment, and via what mechanism, do they agree to use `h2` instead of `http/1.1`? Why can't a plain browser use cleartext `h2c`?

---

## Quick Reference Card

| Concept | What it is | Key fact |
|---------|-----------|----------|
| Binary framing | HTTP/2's wire format | 9-byte frame header + payload; no plaintext lines |
| Stream | One request/response over the shared connection | Client IDs odd, server (push) even, stream 0 = control |
| Multiplexing | Many streams interleaved on ONE TCP connection | Fixes HTTP-level head-of-line blocking |
| Message | A full HTTP request or response | Composed of one or more frames |
| HPACK | Header compression | Static table (61 entries) + dynamic table + Huffman coding |
| Pseudo-headers | Replace request/status line | `:method :scheme :authority :path` (req), `:status` (resp) |
| Flow control | Credit-based backpressure | `WINDOW_UPDATE`; per-stream + connection (stream 0); default 65,535 B |
| Server push | Proactive resource delivery | `PUSH_PROMISE`; DEPRECATED — removed from Chrome 106 (2022) |
| ALPN | Protocol negotiation inside TLS | Name `h2` (over TLS) or `h2c` (cleartext, rare) |
| TCP HOL blocking | The unfixed flaw | One lost segment stalls ALL streams → motivates HTTP/3 |

**Frame types cheat block:**
```
0x0 DATA          body bytes (flow-controlled)
0x1 HEADERS       opens a stream + HPACK headers
0x2 PRIORITY      stream dependency/weight (deprecated in RFC 9113)
0x3 RST_STREAM    cancel one stream
0x4 SETTINGS      connection config (on stream 0)
0x5 PUSH_PROMISE  server push announcement (dead in practice)
0x6 PING          RTT / keepalive
0x7 GOAWAY        graceful connection shutdown
0x8 WINDOW_UPDATE flow-control credit
0x9 CONTINUATION  overflow HEADERS bytes
```

**Command cheat block:**
```bash
curl --http2 -v URL                                  # negotiate h2, show ALPN + frames
curl -o /dev/null -s -w "%{http_version}\n" URL      # print negotiated version
curl --http2-prior-knowledge URL                     # cleartext h2c, no upgrade
curl --http1.1 URL                                   # force 1.1 for comparison
nghttp -v URL                                        # dump every frame
nghttp -ans URL                                      # per-stream timing (see multiplexing)
openssl s_client -alpn h2 -connect host:443          # inspect raw ALPN
```

**The one-line summary:** *HTTP/2 keeps HTTP's meaning but rewrites its bytes — binary frames, one connection, many interleaved streams, compressed headers — fixing HTTP-level HOL blocking while leaving TCP-level HOL blocking for HTTP/3.*

---

## When Would I Use This at Work?

### Scenario 1: "Our asset-heavy dashboard is slow, and it's not the backend"
Your API responses are fast (TTFB is fine) but the page takes seconds to fully render because it pulls 60 small files. Check the protocol: `curl -w "%{http_version}"`. If it's `1.1`, the browser is bottlenecked on 6 connections. Enabling HTTP/2 on your reverse proxy (one config line) lets all 60 load as interleaved streams over one connection — often a dramatic, code-free win.

### Scenario 2: "We shard assets across 4 subdomains for speed"
That's an HTTP/1.1-era optimization. Under HTTP/2 it's actively harmful — it splits traffic across 4 connections, defeating multiplexing and duplicating TLS handshakes and HPACK tables. Consolidate to one origin so the browser coalesces everything onto a single multiplexed connection.

### Scenario 3: "Mobile users on bad networks report it's slower since we 'upgraded'"
Classic TCP head-of-line blocking. All your HTTP/2 streams share one TCP connection, so a single lost packet on a lossy cellular link stalls every in-flight request. This is a known HTTP/2 limitation, not a bug in your config. The real fix is HTTP/3/QUIC (Topic 12), which gives each stream independent delivery over UDP.

### Scenario 4: "We're adding gRPC between two internal services"
gRPC is built directly on HTTP/2 streams and framing (Topic 37). Understanding streams, flow control, and `GOAWAY` here is prerequisite to debugging gRPC connection churn, load-balancing quirks, and `RST_STREAM` errors later. Internally you may even run gRPC over `h2c` (cleartext HTTP/2) on a trusted network.

---

## Connected Topics

**Study before this:**
- `10-http1.1-in-depth.md` — you must know HTTP/1.1's request/response format, keep-alive, and pipelining problems to appreciate what HTTP/2 fixes.
- `08-tcp-in-depth.md` — HTTP/2 rides one TCP connection; TCP's in-order delivery is exactly what causes the remaining HOL blocking.
- `09-tls-ssl-in-depth.md` — ALPN (how `h2` is chosen) lives inside the TLS handshake.

**Study next:**
- `12-http3-and-quic.md` — the direct sequel: QUIC over UDP gives each stream independent delivery, finally killing TCP-level head-of-line blocking.
- `13-websockets.md` — a different concurrency model (full-duplex frames) for a different problem.

**Connects forward to:**
- `37-grpc-and-protocol-buffers.md` — gRPC is HTTP/2 all the way down; streams and framing are its foundation.
- `28-load-balancers.md` / `29-reverse-proxies.md` — where you actually flip HTTP/2 on and where h2c server-to-server hops appear.

---

*Doc saved: `/docs/networking/11-http2.md`*
