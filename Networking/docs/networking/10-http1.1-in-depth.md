# 10 — HTTP/1.1 in Depth

> **Phase 2 — Topic 5 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`, `09-tls-ssl-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine writing a letter to a company's customer service desk using a strict form:

1. The **first line** says what you want and which department: *"PLEASE SEND me the file at /users, using form version HTTP/1.1."* That's the **request line**.
2. Below it you list details, one per line, like *"To: api.myapp.com"* and *"I can read: JSON."* Those are **headers**.
3. You leave **one blank line** so the desk knows your list of details is finished.
4. If you're *submitting* something (like a filled-out form), you attach it **below the blank line**. That's the **body**.

The clerk replies using the exact same form:

1. First line: *"HTTP/1.1 200 OK"* — a status number and a short human word.
2. Headers describing the reply.
3. A blank line.
4. The actual content (the file you asked for).

That's it. HTTP is just **plain text on a fixed form**, sent back and forth over the reliable channel that TCP already built for you (Topic 08). Everything else — methods, status codes, keep-alive — is just rules about how to fill out that form correctly.

---

## Where This Lives in the Network Stack

HTTP is a **Layer 7 (Application)** protocol. It rides *on top of* the reliable byte stream that TCP gives it, which in turn rides on IP. When it's HTTPS, a TLS layer (Topic 09) sits between HTTP and TCP and encrypts everything HTTP writes.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   ( ►► HTTP/1.1 ◄◄  lives HERE )          │  ← this topic
│  Layer 6 — Presentation  (TLS/SSL — encrypts the HTTP bytes)    │  ← Topic 09
│  Layer 5 — Session       (—)                                    │
│  Layer 4 — Transport     (TCP — the reliable byte stream)       │  ← Topic 08
│  Layer 3 — Network       (IP — routing, addresses)              │  ← Topic 04
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (fiber, copper, radio)                 │
└─────────────────────────────────────────────────────────────────┘

Key insight: HTTP has NO idea about IP, ports, or packets.
It hands a stream of text characters to TCP and says "deliver these
in order, reliably." TCP does the hard part. HTTP is just the language
spoken over the phone line TCP already connected.
```

By the time HTTP runs, the TCP 3-way handshake is done and (for HTTPS) the TLS handshake is done. HTTP is the **first thing you actually say** once the encrypted phone call connects.

---

## What Is This?

**HTTP/1.1** is the request/response text protocol that has powered the web since 1997 (RFC 2068, refined by RFC 2616, and re-specified cleanly in RFCs 7230–7235 in 2014). Even in 2026, with HTTP/2 and HTTP/3 widely deployed, HTTP/1.1 is still:

- What most internal service-to-service calls speak (behind a load balancer).
- What you type by hand when debugging with `curl`, `nc`, or `telnet`.
- The **semantic foundation** that HTTP/2 and HTTP/3 inherit — they change the *wire encoding* (binary framing, multiplexing) but keep the same methods, status codes, and header meanings.

Its defining characteristics versus its predecessor HTTP/1.0:

| Feature | HTTP/1.0 | HTTP/1.1 |
|---------|----------|----------|
| Connections | New TCP connection per request | **Persistent by default** (keep-alive) |
| `Host` header | Optional | **Mandatory** (enables virtual hosting) |
| Chunked transfer | No | **Yes** (stream bodies of unknown length) |
| Pipelining | No | Yes (in theory — see why it failed) |
| Caching controls | Basic | Rich (`Cache-Control`, `ETag`, etc.) |

The core message: **HTTP/1.1 is a human-readable, text-based, one-request-then-one-response protocol over a persistent TCP connection.**

---

## Why Does It Matter for a Backend Developer?

You write HTTP responses for a living. Every API endpoint you build produces an HTTP/1.1 (or /2, /3) message. If you don't understand the wire format:

- You return **`200 OK` with an error inside the JSON body** — and every client, cache, and monitoring dashboard thinks your failing endpoint is healthy.
- You confuse **`401` (who are you?)** with **`403` (I know who you are, you're not allowed)** — and now your retry logic loops forever re-authenticating on a permissions problem.
- You forget that HTTP/1.1 **requires the `Host` header** — and your hand-rolled health-check probe gets a `400 Bad Request` from nginx.
- You set a wrong **`Content-Length`** — and the client hangs forever waiting for bytes that never come, or truncates the body.
- You open **a new TCP connection for every request** in a hot loop — and burn 2 RTTs of handshake latency on every single call, melting your connection pool and exhausting ephemeral ports.
- You can't read a **`502 Bad Gateway`** vs **`504 Gateway Timeout`** from your load balancer — so you debug your app when the problem is upstream.

This is the protocol your job is written in. Reading a raw HTTP exchange should feel as natural as reading a stack trace.

---

## The Packet/Protocol Anatomy

HTTP is **text with a strict shape**. Every message — request or response — has the same four parts. Lines end with **CRLF** (`\r\n`, carriage-return + line-feed), and a **bare CRLF on its own line** separates headers from body.

### Request message

```
┌──────────────────────────────────────────────────────────────┐
│  START LINE (request line)                                    │
│  GET /users?page=2 HTTP/1.1\r\n                               │  ← METHOD SP path SP version CRLF
│                                                               │
│  HEADERS (zero or more, each "Name: value")                   │
│  Host: api.myapp.com\r\n                                      │  ← name ":" OWS value CRLF
│  Accept: application/json\r\n                                 │
│  User-Agent: curl/8.4.0\r\n                                   │
│                                                               │
│  \r\n                                                         │  ← BLANK LINE = end of headers
│                                                               │
│  BODY (optional — bytes as-is, length given by a header)      │
│  {"name":"Alice"}                                             │  ← present for POST/PUT/PATCH
└──────────────────────────────────────────────────────────────┘
```

### Response message

```
┌──────────────────────────────────────────────────────────────┐
│  START LINE (status line)                                     │
│  HTTP/1.1 200 OK\r\n                                          │  ← version SP code SP reason CRLF
│                                                               │
│  HEADERS                                                      │
│  Content-Type: application/json\r\n                           │
│  Content-Length: 42\r\n                                       │
│  Connection: keep-alive\r\n                                   │
│                                                               │
│  \r\n                                                         │  ← BLANK LINE = end of headers
│                                                               │
│  BODY                                                         │
│  {"users":[{"id":1,"name":"Alice"}]}                          │
└──────────────────────────────────────────────────────────────┘
```

### Seeing the invisible CRLF

The bytes that actually travel on the wire, with control characters made visible:

```
GET /users HTTP/1.1<CR><LF>
Host: api.myapp.com<CR><LF>
Accept: application/json<CR><LF>
<CR><LF>                          ← this empty line is what ends the header block

Hex of that blank-line separator:  0D 0A 0D 0A
                                   \r \n \r \n
                                   └─end of last header─┘└─blank line─┘
```

That `\r\n\r\n` (`0D 0A 0D 0A`) is the single most important byte sequence in HTTP. It is how a parser knows "headers are done, the body (if any) starts now." Get it wrong — send only `\n` instead of `\r\n` — and strict servers reject your request.

### How the parser finds the body's length

The body has no natural terminator (it's raw bytes that could contain anything, including CRLFs). So HTTP tells the receiver where the body ends in exactly **one** of these ways:

```
1. Content-Length: 42        → read exactly 42 bytes, then stop.
2. Transfer-Encoding: chunked → read chunks until a zero-length chunk.
3. Neither (response only)    → read until the server closes the TCP connection.
                                (Requires Connection: close — kills keep-alive.)
```

---

## How It Works — Step by Step

Let's trace a real `POST` to `https://api.myapp.com/users`, assuming TCP + TLS are already established (Topics 08, 09). Everything below happens **inside** the encrypted tunnel.

### Step 1 — Client writes the request bytes

```
The client serializes its request into one contiguous stream of text:

POST /users HTTP/1.1\r\n
Host: api.myapp.com\r\n
Content-Type: application/json\r\n
Content-Length: 17\r\n
\r\n
{"name":"Alice"}

It writes this whole blob to the TCP socket. TCP chops it into
segments and delivers them reliably and in order.
```

### Step 2 — Server reads and parses

```
2a. Server reads bytes until it sees the first \r\n → parses the request line.
2b. Reads line by line until it hits the blank line (\r\n\r\n) → parses headers.
2c. Sees Content-Length: 17 → reads exactly 17 more bytes as the body.
2d. Now it has a complete request. It routes /users → the POST handler.
```

### Step 3 — Server runs your handler

```
3a. Your route handler runs: validate input, insert into DB.
3b. It builds a response object: status 201, a Location header, a JSON body.
```

### Step 4 — Server writes the response bytes

```
HTTP/1.1 201 Created\r\n
Content-Type: application/json\r\n
Content-Length: 38\r\n
Location: /users/1001\r\n
Connection: keep-alive\r\n
\r\n
{"id":1001,"name":"Alice","active":true}
```

### Step 5 — Client reads the response

```
5a. Client reads status line → 201 Created.
5b. Reads headers until blank line.
5c. Sees Content-Length: 38 → reads 38 body bytes.
5d. Sees Connection: keep-alive → keeps the TCP connection OPEN.
```

### Step 6 — The connection is reused (this is the HTTP/1.1 superpower)

```
Because neither side said "Connection: close", the same TCP+TLS
connection stays open. The NEXT request skips TCP handshake (saves 1 RTT)
AND TLS handshake (saves 1–2 RTTs):

  Request 1: [TCP handshake][TLS handshake][req/resp]   ← ~3 RTTs
  Request 2: [req/resp]                                 ← ~1 RTT  ✅ reused
  Request 3: [req/resp]                                 ← ~1 RTT  ✅ reused
```

### The whole picture over one persistent connection

```
CLIENT                                                    SERVER
  │                                                          │
  │═══════ TCP 3-way handshake (once) ══════════════════════│ ~1 RTT
  │═══════ TLS handshake (once) ════════════════════════════│ ~1 RTT
  │                                                          │
  │──── POST /users HTTP/1.1 ... {body} ────────────────────►│
  │◄─── HTTP/1.1 201 Created ... {body} ────────────────────│  request 1
  │                                                          │
  │──── GET /users/1001 HTTP/1.1 ──────────────────────────►│
  │◄─── HTTP/1.1 200 OK ... {body} ─────────────────────────│  request 2 (reused)
  │                                                          │
  │──── GET /users/1002 HTTP/1.1 ──────────────────────────►│
  │◄─── HTTP/1.1 200 OK ... {body} ─────────────────────────│  request 3 (reused)
  │                                                          │
  │   (idle timeout, or explicit Connection: close)          │
  │═══════ TCP FIN/ACK teardown ════════════════════════════│
```

**The rule that makes HTTP/1.1 tick:** on one connection, requests and responses are **strictly serialized** — one full response must come back before the next request's response. That single rule is both keep-alive's strength and pipelining's fatal weakness (below).

---

## Exact Syntax Breakdown

### The request line

```
POST   /users?page=2#frag   HTTP/1.1
│      │                     │
│      │                     └── HTTP version. ALWAYS "HTTP/1.1" (uppercase, slash, dot).
│      │
│      └── request-target: the path + query string (the #fragment is NEVER sent).
│         Origin-form for normal requests. "*" for OPTIONS *. Absolute-form for proxies.
│
└── method: case-SENSITIVE, always UPPERCASE (GET, not get).

Separated by single spaces (SP, 0x20). Terminated by CRLF.
```

### A header line

```
Content-Type:  application/json ; charset=utf-8
│            │  │                              │
│            │  │                              └── value (may contain params after ;)
│            │  └── OWS = optional whitespace, ignored (usually one space)
│            └── colon — NO space before it (space before colon = malformed)
└── field-name: case-INSENSITIVE ("content-type" == "Content-Type")

Terminated by CRLF. Repeat header names are allowed for some fields
(they combine as comma-separated lists, e.g. Accept).
```

### The status line

```
HTTP/1.1   404   Not Found
│          │     │
│          │     └── reason phrase: human hint ONLY. Clients IGNORE it. Can be anything.
│          └── status code: 3-digit machine-readable result. THIS is what code checks.
└── version
```

### Two ways to invoke a raw request

```
# With curl (does TCP+TLS+HTTP for you, -v prints the raw lines)
curl -v -X POST https://api.myapp.com/users \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice"}'
│    │  │                                  │
│    │  │                                  └── -d = request body (implies POST)
│    │  └── -H = add a request header
│    └── -v = show the > (sent) and < (received) lines
└── -X = override the method (usually inferred; explicit here for clarity)

# By hand with nc (netcat) — you type the raw HTTP yourself over plain TCP
printf 'GET /get HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n' \
  | nc httpbin.org 80
│                                                                          │
│  Note: \r\n written explicitly. Host is MANDATORY. Blank line ends headers.
└── printf (not echo) so \r\n are real CR/LF bytes, not literal backslashes.
```

---

## Example 1 — Basic

Type an HTTP request **by hand** so you feel the raw protocol. We use plain HTTP (port 80, no TLS) so nothing is hidden.

```bash
# Open a raw TCP connection and speak HTTP yourself.
# printf turns \r\n into real CRLF bytes.
printf 'GET /get HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n' \
  | nc httpbin.org 80
```

What comes back (annotated):

```
HTTP/1.1 200 OK              ← status line: version, code 200, reason "OK"
Date: Sat, 12 Jul 2026 09:14:22 GMT
Content-Type: application/json   ← body is JSON
Content-Length: 195              ← body is exactly 195 bytes
Connection: close                ← server will close after this (we asked it to)
Server: gunicorn/19.9.0
                                 ← THE BLANK LINE (\r\n\r\n): headers end here
{                                ← body starts (exactly 195 bytes)
  "args": {},
  "headers": { "Host": "httpbin.org" },
  "url": "http://httpbin.org/get"
}
```

Three things to notice:

1. **You wrote `Host: httpbin.org` yourself.** Delete it and re-run — a strict HTTP/1.1 server replies `400 Bad Request`, because one IP can host thousands of sites and the server needs `Host` to know which one you want.
2. **`Content-Length: 195`** tells you exactly where the body ends. The client reads 195 bytes and stops.
3. **`Connection: close`** — we opted out of keep-alive so `nc` returns instead of hanging. Remove it and `nc` waits (the server keeps the connection open for a possible second request).

Now the same thing with `curl -v`, which does the plumbing but shows you the raw lines (`>` = sent, `<` = received):

```bash
curl -v http://httpbin.org/get
```
```
> GET /get HTTP/1.1        ← your request line (curl added the rest for you)
> Host: httpbin.org        ← curl adds Host automatically
> User-Agent: curl/8.4.0
> Accept: */*
>                          ← blank line ends your headers
< HTTP/1.1 200 OK          ← response status line
< Content-Type: application/json
< Content-Length: 195
<                          ← blank line ends response headers
{ ...body... }
```

Same protocol, same bytes — `curl` just filled in `Host`, `User-Agent`, and `Accept` for you.

---

## Example 2 — Production Scenario: "Our alerting is blind — a broken endpoint reports success"

**The situation.** Your team's payments API has an endpoint `POST /charges`. On-call gets paged at 3 a.m.: customers report failed charges, but your dashboards are **all green**. Error rate: 0.0%. The load balancer's 5xx count is zero. Yet the charges are failing. How is the monitoring lying?

**Step 1 — Reproduce with a bad input and watch the raw exchange.**

```bash
curl -v -X POST https://api.myapp.com/charges \
     -H "Content-Type: application/json" \
     -d '{"amount": -500, "currency": "USD"}'
```
```
> POST /charges HTTP/1.1
> Host: api.myapp.com
> Content-Type: application/json
> Content-Length: 36
>
< HTTP/1.1 200 OK                          ← 🚩 the smoking gun: 200 for a FAILURE
< Content-Type: application/json
< Content-Length: 78
<
{"success": false, "error": "amount must be positive", "code": "INVALID_AMOUNT"}
```

**Step 2 — Spot the anti-pattern.** The endpoint returned **`200 OK`** with an *error inside the body*. Every layer that only reads the status line — your load balancer's 5xx metric, Datadog's HTTP monitor, the client's `if (res.ok)` check, the CDN's caching logic — sees `200` and concludes **success**. The failure is invisible to everything except a human reading the JSON.

**Step 3 — Contrast with what it *should* return.** A client-supplied invalid amount is the caller's fault → **`400 Bad Request`** (or `422 Unprocessable Entity` for semantic validation). If the downstream bank timed out, that's the server's fault → **`502`/`503`/`504`**.

```
WRONG (current):                        RIGHT (fixed):
HTTP/1.1 200 OK                         HTTP/1.1 400 Bad Request
{"success": false, ...}                 {"error": "amount must be positive",
                                         "code": "INVALID_AMOUNT"}

Result: monitoring blind,               Result: LB counts a 4xx, client's
alerts silent, client thinks            res.ok is false, retries/validation
it succeeded.                           logic behaves correctly.
```

**Step 4 — A second, sneakier variant: the wrong 5xx.** Suppose invalid input instead throws an unhandled exception and returns `500`:

```bash
curl -sD - -o /dev/null -X POST https://api.myapp.com/charges \
     -H "Content-Type: application/json" -d '{"amount": "not-a-number"}'
```
```
HTTP/1.1 500 Internal Server Error   ← 🚩 blaming the SERVER for the CLIENT's bad input
```

This is the opposite failure: a `500` for what is really a `400`. Now your **error budget burns**, PagerDuty fires, and someone spends an hour debugging server code when the real fix is input validation returning `400`. `500` means *"I broke"*; `400` means *"you sent me garbage."* Confusing them sends the on-call engineer to the wrong place.

**Step 5 — The fix and how to verify it.**

```bash
# After deploying the fix, confirm each case maps to the right code:
for body in '{"amount":-500,"currency":"USD"}' '{"amount":"x"}' '{"amount":500,"currency":"USD"}'; do
  echo "=== $body ==="
  curl -s -o /dev/null -w "status=%{http_code}\n" \
    -X POST https://api.myapp.com/charges \
    -H "Content-Type: application/json" -d "$body"
done
```
```
=== {"amount":-500,"currency":"USD"} ===
status=400        ← client error, correctly signaled
=== {"amount":"x"} ===
status=400        ← was 500, now 400 ✅
=== {"amount":500,"currency":"USD"} ===
status=201        ← success creates a resource → 201, not a bare 200
```

**The lesson:** the **status code is a contract with every machine in the path**, not decoration. Load balancers, CDNs, client libraries, retry policies, and dashboards all read the 3-digit code and act on it *before any human sees the body*. Put the truth in the status line.

---

## Common Mistakes

### Mistake 1: Returning `200 OK` for errors

```
WRONG:
  HTTP/1.1 200 OK
  {"success": false, "error": "user not found"}

RIGHT:
  HTTP/1.1 404 Not Found
  {"error": "user not found"}
```
**Why it bites:** Every cache, load balancer, monitoring tool, and client library keys off the status code. A `200` with an error body makes broken endpoints look healthy, hides failures from alerting, and can even cause a CDN to cache the error as a valid response. Put the outcome in the status line.

---

### Mistake 2: Confusing `401 Unauthorized` with `403 Forbidden`

```
401 Unauthorized  → "I don't know WHO you are." (missing/invalid/expired credentials)
                    The correct response to: no token, bad token, expired token.
                    MUST include a WWW-Authenticate header telling the client how to auth.
                    Correct client reaction: (re)authenticate, then retry.

403 Forbidden     → "I know who you are, and you're NOT ALLOWED." (valid identity,
                    insufficient permission). A regular user hitting an admin-only route.
                    Correct client reaction: do NOT retry with new credentials — it won't help.
```
**Why it bites:** If you return `401` for a permissions problem, well-behaved clients enter an infinite refresh-token → retry → `401` loop, because they think their credentials are the issue. Note the naming irony: `401` is literally named "Unauthorized" but actually means *unauthenticated*.

---

### Mistake 3: Forgetting HTTP/1.1 requires a `Host` header

```
WRONG (hand-rolled probe / minimal client):
  GET /health HTTP/1.1\r\n
  \r\n
  → Response: HTTP/1.1 400 Bad Request

RIGHT:
  GET /health HTTP/1.1\r\n
  Host: internal-api\r\n
  \r\n
  → Response: HTTP/1.1 200 OK
```
**Why it bites:** HTTP/1.1 made `Host` **mandatory** so one IP/port can serve many domains (virtual hosting) and so reverse proxies (nginx, ALB) can route by hostname. Health checks, scripts, and load-balancer probes that skip `Host` get rejected. `curl` and every real library add it automatically — but the moment you write raw HTTP by hand, you must include it.

---

### Mistake 4: Wrong or missing `Content-Length`

```
Content-Length TOO SMALL:  client reads N bytes and STOPS mid-body → truncated JSON,
                           "Unexpected end of JSON input" errors.

Content-Length TOO LARGE:  client waits for bytes that never arrive → HANGS until
                           it times out. Looks like a "slow server."

Content-Length MISSING     with keep-alive on and no chunked encoding → the receiver
(on a body):               cannot tell where the body ends → hang or protocol error.
```
**Why it bites:** The receiver uses `Content-Length` to know exactly how many body bytes to read. Frameworks compute it for you — but if you stream a response and set the header manually, or mutate the body after setting it (e.g. gzip middleware forgetting to fix the length), you corrupt every response. If you don't know the length up front, **use chunked transfer encoding instead** (below).

---

### Mistake 5: Assuming the body is complete without checking length/chunked

```
WRONG mental model: "I read from the socket, I got some bytes, that's the body."

REALITY: one read() may return a PARTIAL body. You must keep reading until:
  - you've consumed Content-Length bytes, OR
  - you hit the terminating zero-length chunk (Transfer-Encoding: chunked), OR
  - the connection closes (only valid when Connection: close).
```
**Why it bites:** TCP is a *byte stream*, not a message boundary. A large JSON body arrives across many TCP segments. Naive code that treats the first chunk of bytes as "the whole response" silently truncates large payloads. Always let the HTTP library frame the body for you, or respect the length signal yourself.

---

### Mistake 6: One request per connection (no keep-alive)

```
WRONG (new connection every call):
  for each request:
     TCP handshake (1 RTT) + TLS handshake (1–2 RTTs) + request  ← 3 RTTs EVERY time

RIGHT (reuse via keep-alive / a connection pool):
  connect once (3 RTTs), then:
     request, request, request, ...                              ← 1 RTT each after
```
**Why it bites:** Sending `Connection: close` (or using an HTTP client without pooling) re-runs the TCP + TLS handshakes on every call. On a Tokyo→Virginia link (~170ms RTT) that's an extra ~340–510ms **per request**. It also floods the server with connections stuck in `TIME_WAIT` and can exhaust the client's ephemeral port range under load. HTTP/1.1 is keep-alive by default — don't turn it off, and use a pooling client (most do this automatically).

---

## Hands-On Proof

```bash
# 1. Type a raw HTTP/1.1 request BY HAND over plain TCP. Feel the CRLFs.
printf 'GET /get HTTP/1.1\r\nHost: httpbin.org\r\nConnection: close\r\n\r\n' \
  | nc httpbin.org 80

# 2. Prove Host is mandatory: OMIT it and watch the 400.
printf 'GET /get HTTP/1.1\r\nConnection: close\r\n\r\n' | nc httpbin.org 80
#    → HTTP/1.1 400 Bad Request  (server can't route without Host)

# 3. See ALL the request/response lines curl actually sends and receives.
curl -v http://httpbin.org/get 2>&1 | grep -E '^[<>]'

# 4. Print ONLY the response status code (what machines care about).
curl -s -o /dev/null -w "%{http_code}\n" https://httpbin.org/status/404
#    → 404

# 5. Force different status codes on demand and watch them.
for code in 200 201 301 400 401 403 404 429 500 503; do
  printf "%s -> " "$code"
  curl -s -o /dev/null -w "%{http_code}\n" "https://httpbin.org/status/$code"
done

# 6. Prove keep-alive reuses ONE connection for multiple requests.
#    "Re-using existing connection" appears on the 2nd fetch.
curl -v https://httpbin.org/get https://httpbin.org/headers 2>&1 \
  | grep -Ei 'connection|Re-using|Trying'

# 7. Compare a HEAD (headers only, no body) vs GET.
curl -sI https://httpbin.org/get      # -I = HEAD request; note NO body comes back

# 8. See chunked transfer encoding streaming a body of unknown length.
curl -v https://httpbin.org/stream/3 2>&1 | grep -i 'transfer-encoding'
#    → Transfer-Encoding: chunked   (no Content-Length; body streamed in chunks)

# 9. Watch method + status together for a POST.
curl -s -o /dev/null -w "%{method} -> %{http_code}\n" \
  -X POST -d '{"x":1}' -H "Content-Type: application/json" https://httpbin.org/post
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a raw exchange and label every part

```bash
# Run this and capture the raw request AND response lines:
curl -v -X POST https://httpbin.org/post \
     -H "Content-Type: application/json" \
     -d '{"name":"Bob"}' 2>&1 | grep -E '^[<>]'

# From the output, identify and write down:
# 1. The request line (method, target, version).
# 2. Two headers curl added that you did NOT specify.
# 3. The exact status line of the response.
# 4. The Content-Length curl sent for your body — does it match len('{"name":"Bob"}')?
# 5. Where is the blank line (\r\n\r\n) in both the request and response?
```

### Exercise 2 — Medium: Prove persistent connections save round-trips

```bash
# A) One connection, TWO requests (keep-alive, the default):
curl -v https://httpbin.org/get https://httpbin.org/uuid 2>&1 \
  | grep -Ei 'Trying|Connected|Re-using|Connection #'

# B) Force a fresh connection each time:
curl -v -H "Connection: close" https://httpbin.org/get 2>&1 \
  | grep -Ei 'Trying|Connected|close'

# Answer:
# 1. In (A), does the SECOND request say "Re-using existing connection"? What does that mean it skipped?
# 2. Roughly how many RTTs does request #2 save by reusing the connection? (Count TCP + TLS.)
# 3. In (B), what does "Connection: close" force the server to do after the response?
# 4. Why is turning off keep-alive catastrophic for a service making 1000s of calls/sec?
```

### Exercise 3 — Hard (Production Simulation): Audit an API's status-code correctness

```bash
# You've inherited an API. Audit whether it uses status codes correctly.
# For each scenario below, capture the status code:

# a) Fetch a resource that does NOT exist:
curl -s -o /dev/null -w "missing resource: %{http_code}\n" https://api.example.com/users/does-not-exist

# b) Send malformed JSON to a create endpoint:
curl -s -o /dev/null -w "bad body:        %{http_code}\n" \
  -X POST -H "Content-Type: application/json" -d '{bad json' https://api.example.com/users

# c) Call an authenticated endpoint with NO token:
curl -s -o /dev/null -w "no token:        %{http_code}\n" https://api.example.com/me

# d) Call it with a VALID token but a resource you don't own:
curl -s -o /dev/null -w "not yours:       %{http_code}\n" \
  -H "Authorization: Bearer $VALID_TOKEN" https://api.example.com/users/999/private

# Questions:
# 1. For each case, what code SHOULD it be? (Map: missing→404, bad body→400/422,
#    no token→401, not yours→403.) Where does the real API deviate?
# 2. Any endpoint that returns 200 with {"error":...} — write the one-line bug report.
# 3. If (c) returns 403 instead of 401, what breaks in a client's token-refresh logic?
# 4. If (b) returns 500 instead of 400, whose fault does the metric BLAME, and who
#    actually caused it? What does that do to the on-call rotation's alert fatigue?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **Draw the four parts of an HTTP request in order.** Which exact byte sequence separates the headers from the body, and what does it look like in hex?

2. **A method is "safe" and a method is "idempotent" — define both.** Give one method that is safe, one that is idempotent-but-not-safe, and one that is neither.

3. **You call `POST /orders` and the network times out. Is it safe to blindly retry? Why or why not — and what would you do instead?**

4. **A client sends `Authorization: Bearer <expired-token>`. Should the server return `401` or `403`? What header must accompany a `401`, and how should a good client react to each?**

5. **Your response has no `Content-Length` and no `Transfer-Encoding: chunked`, but does have a body. How does the client know the body ended — and what does that force about the connection?**

6. **Explain in one sentence why HTTP/1.1 pipelining failed in practice, and name the specific problem it hit.** Which later protocol fixes it, and how (one phrase)?

7. **HTTP/1.1 made one header mandatory that HTTP/1.0 didn't require. Which one, and what two capabilities does it unlock?**

---

## Quick Reference Card

### Methods — semantics, safe, idempotent

| Method | Purpose | Safe? | Idempotent? | Typical body |
|--------|---------|:-----:|:-----------:|-------------|
| **GET** | Retrieve a resource | ✅ | ✅ | none (request) |
| **HEAD** | Like GET but headers only, no body | ✅ | ✅ | none |
| **OPTIONS** | Ask what methods/CORS are allowed | ✅ | ✅ | none |
| **TRACE** | Echo the request (debugging; often disabled) | ✅ | ✅ | none |
| **POST** | Create / process / non-idempotent action | ❌ | ❌ | yes |
| **PUT** | Replace resource at a known URI | ❌ | ✅ | yes |
| **PATCH** | Partial update | ❌ | ❌* | yes |
| **DELETE** | Remove a resource | ❌ | ✅ | optional |
| **CONNECT** | Establish a tunnel (HTTPS via proxy) | ❌ | ❌ | none |

*PATCH *can* be idempotent depending on the patch semantics, but is not guaranteed.
**Safe** = no side effects (read-only). **Idempotent** = doing it N times == doing it once.

### Status codes a backend dev must know

| Code | Meaning | When you return it |
|------|---------|--------------------|
| **200** OK | Success, body follows | Successful GET / general success |
| **201** Created | Resource created | Successful POST; add a `Location` header |
| **204** No Content | Success, no body | Successful DELETE / PUT with nothing to return |
| **301** Moved Permanently | Permanent redirect | URL changed forever (cacheable) |
| **302** Found | Temporary redirect | Temporary location (method may change) |
| **304** Not Modified | Cache still valid | Conditional GET (ETag/If-None-Match matched) |
| **307** Temp Redirect | Temp redirect, **keep method** | Like 302 but method/body preserved |
| **308** Perm Redirect | Perm redirect, **keep method** | Like 301 but method/body preserved |
| **400** Bad Request | Malformed request | Bad syntax / unparseable body |
| **401** Unauthorized | **Not authenticated** | Missing/invalid/expired credentials (+`WWW-Authenticate`) |
| **403** Forbidden | **Authenticated, not allowed** | Valid identity, no permission |
| **404** Not Found | Resource doesn't exist | Unknown URI / hidden resource |
| **405** Method Not Allowed | Method not supported here | POST to a read-only URI (+`Allow` header) |
| **409** Conflict | State conflict | Duplicate create / version conflict |
| **422** Unprocessable Entity | Semantic validation failed | Well-formed but invalid data |
| **429** Too Many Requests | Rate limited | Client over quota (+`Retry-After`) |
| **500** Internal Server Error | Server bug | Unhandled exception on your side |
| **502** Bad Gateway | Upstream gave a bad response | Proxy/LB got garbage from your app |
| **503** Service Unavailable | Temporarily down/overloaded | Maintenance / no healthy backends (+`Retry-After`) |
| **504** Gateway Timeout | Upstream too slow | Proxy/LB timed out waiting for your app |

### Essential headers

| Header | Direction | What it does |
|--------|-----------|--------------|
| `Host` | request | **Mandatory in 1.1.** Which virtual host you want |
| `Content-Type` | both | Media type of the body (`application/json`, etc.) |
| `Content-Length` | both | Exact body size in bytes |
| `Transfer-Encoding: chunked` | both | Stream a body of unknown length (no Content-Length) |
| `Connection: keep-alive / close` | both | Reuse or close the TCP connection |
| `Accept` | request | Media types the client can handle |
| `User-Agent` | request | Client software identity |
| `Location` | response | Where the resource is (201) or redirect target (3xx) |
| `WWW-Authenticate` | response | How to authenticate (accompanies 401) |
| `Retry-After` | response | When to retry (accompanies 429 / 503) |

### Cheat block

```
MESSAGE SHAPE (both directions):
  start-line CRLF
  Header-Name: value CRLF
  ... more headers ...
  CRLF                       ← blank line = end of headers (hex: 0D 0A 0D 0A)
  [body]                     ← framed by Content-Length OR chunked

REQUEST LINE :  METHOD SP request-target SP HTTP/1.1 CRLF
STATUS  LINE :  HTTP/1.1 SP code SP reason-phrase CRLF

BODY FRAMING (how the receiver knows where the body ends):
  Content-Length: N          → read exactly N bytes
  Transfer-Encoding: chunked → read chunks until "0\r\n\r\n"
  neither (response only)    → read until connection close (needs Connection: close)

KEEP-ALIVE:
  default ON in HTTP/1.1. Reuse the TCP+TLS connection → skip handshakes.
  "Connection: close" opts out (server closes after the response).

CHUNKED BODY on the wire:
  1a\r\n                     ← chunk size in HEX (1a = 26 bytes)
  <26 bytes of data>\r\n
  10\r\n                     ← next chunk, 16 bytes
  <16 bytes of data>\r\n
  0\r\n                      ← zero-size chunk = END of body
  \r\n

PIPELINING (avoid): send req2 before resp1 arrives. Died from head-of-line
  blocking + broken proxies. Use HTTP/2 multiplexing instead.

QUICK CODE PICKER:
  created?            201    validation failed?   400 / 422
  deleted, no body?   204    not authenticated?   401
  cache still good?   304    authenticated, no?   403
  moved forever?      301    rate limited?        429
  temp redirect?      307    my bug?              500
  upstream bad?       502    upstream slow?       504
```

---

## When Would I Use This at Work?

### Scenario 1: Designing a REST endpoint's responses
You're building `POST /orders`. You now know: success → `201 Created` with a `Location: /orders/123` header (not a bare `200`); a missing field → `400`; a payment already captured → `409 Conflict`; the caller isn't logged in → `401` with `WWW-Authenticate`; they're logged in but not an admin → `403`. Correct codes mean clients, caches, and dashboards all behave without reading your body.

### Scenario 2: Debugging "the client says the request failed but our logs say 200"
You reach for `curl -v` (or `-w "%{http_code}"`) and discover the endpoint returns `200 OK` with `{"success": false}` in the body. The client's `res.ok` is `true`, so it proceeds as if it worked. You move the failure into the status line (`400`/`422`/`500`) and everything downstream — retries, alerts, metrics — starts working again.

### Scenario 3: Chasing connection churn / port exhaustion
Your service's outbound calls are slow and the box is drowning in `TIME_WAIT` sockets (`ss -tan | grep TIME-WAIT | wc -l` is huge). You check the HTTP client config and find keep-alive disabled or no connection pool — every call re-runs TCP + TLS handshakes. Enable pooling / keep-alive and per-request latency drops by 2–3 RTTs and the socket count collapses.

### Scenario 4: Reading load-balancer errors correctly
Your ALB logs show `502` and `504`. You know `502 Bad Gateway` = your app returned something malformed or crashed the connection; `504 Gateway Timeout` = your app was too slow and the LB gave up. That single distinction tells you whether to look at a crash/exception (`502`) or a slow query/timeout (`504`) — you debug the right thing on the first try.

### Scenario 5: Streaming a large or unknown-length response
You're returning a report generated row-by-row from the database — you don't know the total size up front. Instead of buffering the whole thing to compute `Content-Length`, you use `Transfer-Encoding: chunked` and stream each chunk as it's produced. The client starts receiving data immediately, and memory stays flat.

---

## Connected Topics

**Study before this:**
- `08-tcp-in-depth.md` — HTTP rides on TCP's reliable byte stream; keep-alive is about reusing *that* TCP connection.
- `09-tls-ssl-in-depth.md` — HTTPS is HTTP/1.1 inside a TLS tunnel; the handshakes keep-alive saves are TCP's *and* TLS's.

**Study next (in order):**
- `11-http2.md` — Fixes HTTP/1.1's head-of-line blocking with binary framing and true multiplexing over one connection (the reason pipelining died).
- `14-http-headers-in-depth.md` — Deep dive on every header touched here plus caching (`Cache-Control`, `ETag`) and security headers.
- `15-rest-over-http.md` — Applies these methods, idempotency rules, and status codes to clean REST API design.

**This topic is the vocabulary of the web.** Every HTTP-based topic in Phases 3–6 — auth, CORS, rate limiting, caching, reverse proxies, API gateways, gRPC — assumes you can read and reason about the raw HTTP/1.1 messages defined here.

---

*Doc saved: `/docs/networking/10-http1.1-in-depth.md`*
