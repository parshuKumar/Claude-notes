# 14 — HTTP Headers in Depth

> **Phase 3 — Topic 1 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine you mail a physical package to a friend.

The **box itself** — the thing you actually want them to have — is the HTTP **body** (the JSON, the HTML, the image).

But you also stick a **shipping label** and a bunch of **sticky notes** all over the outside:

- "To: 24 Main Street" (**Host**)
- "This box contains: fragile glassware" (**Content-Type**)
- "Weight: 2.4 kg" (**Content-Length**)
- "I only speak English — send the manual in English" (**Accept-Language**)
- "Here's my membership card so you know it's me" (**Authorization**)
- "Do not open until Christmas" (**Cache-Control**)
- "If nothing changed since last time, don't bother re-sending — just wave" (**If-None-Match**)

The person on the other end reads the sticky notes **before** they even open the box, and decides what to do based on them. They might send it straight back with a note that says "nothing changed, you already have this" (**304 Not Modified**) — without re-shipping the contents at all.

**Headers are the sticky notes on the HTTP envelope.** They are metadata *about* the message — who sent it, what's inside, in what language and encoding, whether it can be cached, and how the browser should treat it for security. The body is the payload; the headers are everything the two sides need to agree on to handle that payload correctly.

---

## Where This Lives in the Network Stack

Headers are part of HTTP, which sits at **Layer 7 (Application)**. They ride *inside* everything you learned about in Topics 08–09.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP request line + HEADERS + body)   │  ← YOU ARE HERE
│  Layer 6 — Presentation  (TLS encrypts the whole HTTP message)  │
│  Layer 5 — Session       (—)                                    │
│  Layer 4 — Transport     (TCP — reliable byte stream, port 443) │
│  Layer 3 — Network       (IP — routes the packets)              │
│  Layer 2 — Data Link     (Ethernet / Wi-Fi frames)              │
│  Layer 1 — Physical      (fiber, copper, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

Critically: **headers live above TLS**. When you use HTTPS, the entire HTTP message — request line, every header, and the body — is encrypted inside the TLS record (Layer 6). A router or your ISP sees only the destination IP and (until Encrypted Client Hello ships) the SNI hostname. It cannot see your `Authorization: Bearer …` token or your `Cookie:` header. That is exactly why sending credentials over plain HTTP is catastrophic — the headers travel in cleartext.

In HTTP/1.1 headers are literal ASCII text on the wire. In HTTP/2 and HTTP/3 the *same logical headers* are compressed into a binary format (HPACK / QPACK) and header names are forced lowercase — but the meaning is identical. This topic teaches the semantics; Topics 11–12 cover the binary framing.

---

## What Is This?

An **HTTP header** is a single `Name: Value` line of metadata attached to a request or a response. A message can carry dozens of them. They are grouped by who sends them and what they describe:

- **Request headers** — sent by the client. Describe the client, what it wants, and the conditions under which it wants it (`Host`, `User-Agent`, `Accept`, `Authorization`, `Cookie`, `If-None-Match`).
- **Response headers** — sent by the server. Describe the server and how to handle the response (`Server`, `Set-Cookie`, `Location`, `Vary`, `Cache-Control`).
- **Representation (entity) headers** — describe the *body* itself, and can appear on either side (`Content-Type`, `Content-Length`, `Content-Encoding`, `Content-Language`, `ETag`, `Last-Modified`).

Headers power four huge mechanisms in HTTP, all of which are just headers under the hood:

1. **Content negotiation** — client says what formats/languages/encodings it accepts; server picks one and tells you which it chose.
2. **Caching & validation** — how long a response is fresh, and how to cheaply re-check without re-downloading.
3. **Security** — instructions to the browser that harden the page against XSS, clickjacking, protocol downgrade, and MIME-sniffing attacks.
4. **State & identity** — cookies, auth tokens, and request-tracing IDs.

Header **names are case-insensitive** by definition (RFC 9110). `Content-Type`, `content-type`, and `CONTENT-TYPE` are the same header. **Values are generally case-sensitive** (except where a specific header defines otherwise, e.g. scheme tokens).

---

## Why Does It Matter for a Backend Developer?

Headers are the single most common thing you configure, debug, and get wrong in day-to-day backend work. Without a solid grasp of them you cannot:

- **Serve the right representation.** Return JSON to an API client and HTML to a browser off the same endpoint using `Accept`.
- **Make your API fast and cheap.** A correct `Cache-Control` + `ETag` turns a 2 MB response into a 150-byte `304 Not Modified`. Get it wrong and you either serve stale data or hammer your origin.
- **Avoid a caching outage.** Forget one `Vary: Accept-Encoding` and a CDN can serve gzip-compressed bytes to a client that asked for plaintext — garbage on the screen for everyone.
- **Pass a security audit.** Missing `Strict-Transport-Security`, `Content-Security-Policy`, or `X-Content-Type-Options: nosniff` are the top findings in every pentest report. These are *your* responsibility as the backend, not the frontend's.
- **Debug "it works in Postman but not the browser."** Nine times out of ten it's a header — CORS, cookies, content type, or an auth header the browser strips.
- **Trace a request across microservices.** `X-Request-ID` / `traceparent` are how one user complaint becomes a single searchable line across ten services.

Every framework (Express, Spring, Django, Rails, Go net/http) is, at bottom, a machine for reading request headers and writing response headers. Understanding them is understanding your framework.

---

## The Packet/Protocol Anatomy

Here is a real HTTP/1.1 request and response, on the wire, byte for byte. The **request line / status line**, the **headers**, a **blank line (CRLF)**, then the optional **body**.

```
REQUEST (client → server)
┌────────────────────────────────────────────────────────────────┐
│ GET /api/products/42 HTTP/1.1        ← request line (not a header)│
├────────────────────────────────────────────────────────────────┤
│ Host: shop.example.com               ┐                          │
│ User-Agent: curl/8.4.0               │                          │
│ Accept: application/json             │  REQUEST + REPRESENTATION │
│ Accept-Encoding: gzip, br            │  HEADERS                  │
│ Accept-Language: en-US,en;q=0.9      │  (each is Name: Value)    │
│ Authorization: Bearer eyJhbGciOi...  │                          │
│ If-None-Match: "a1b2c3d4"            ┘                          │
├────────────────────────────────────────────────────────────────┤
│ (CRLF — blank line = END OF HEADERS)                            │
├────────────────────────────────────────────────────────────────┤
│ (no body for a GET)                                             │
└────────────────────────────────────────────────────────────────┘

RESPONSE (server → client)
┌────────────────────────────────────────────────────────────────┐
│ HTTP/1.1 200 OK                      ← status line (not a header)│
├────────────────────────────────────────────────────────────────┤
│ Server: nginx/1.25.3                 ┐                          │
│ Date: Sat, 12 Jul 2026 09:14:22 GMT  │                          │
│ Content-Type: application/json; charset=utf-8                   │
│ Content-Length: 1287                 │  RESPONSE +               │
│ Content-Encoding: gzip               │  REPRESENTATION HEADERS   │
│ Cache-Control: public, max-age=300   │                          │
│ ETag: "a1b2c3d4"                     │                          │
│ Vary: Accept-Encoding, Accept-Language                          │
│ Strict-Transport-Security: max-age=31536000; includeSubDomains ┘│
├────────────────────────────────────────────────────────────────┤
│ (CRLF — blank line = END OF HEADERS)                            │
├────────────────────────────────────────────────────────────────┤
│ {"id":42,"name":"Wireless Mouse","price":29.99, ... }  ← body   │
└────────────────────────────────────────────────────────────────┘
```

**Anatomy of one header line:**

```
Cache-Control:  public, max-age=300
│            │  │       │
│            │  │       └── directive with a value (seconds)
│            │  └────────── directive (a "token")
│            └───────────── separator: colon + optional whitespace
└────────────────────────── field-name (case-INSENSITIVE)

Wire bytes end each line with CRLF (\r\n = 0x0D 0x0A).
Multiple values are comma-separated on ONE line, OR repeated on
several lines with the same name (both are legal for most headers).
```

**HTTP/2 / HTTP/3 difference:** the exact same headers exist, but they are (a) compressed with HPACK/QPACK against a shared dynamic table so `Host`/cookies aren't re-sent in full on every request, and (b) **lowercased** — `content-type`, never `Content-Type`. Also, HTTP/2 replaces the `Host` header with the `:authority` pseudo-header. Your code still reads them case-insensitively, so you rarely notice.

---

## How It Works — Step by Step

Let's walk the two mechanisms that trip people up most: **content negotiation** and **conditional caching**.

### Content negotiation — the client asks, the server picks

```
Step 1  Client advertises its preferences as "Accept-*" headers:
          Accept:          application/json, text/html;q=0.8
          Accept-Encoding: gzip, br, deflate
          Accept-Language: fr-FR, fr;q=0.9, en;q=0.5

        The "q" value (0.0–1.0) is a QUALITY WEIGHT / preference rank.
        No q means q=1.0 (most preferred). Higher q wins.

Step 2  Server looks at what IT can produce and intersects with the
        client's list. It picks the best mutually-supported option.
          → It has JSON  → matches Accept       → serves JSON
          → It has br    → matches Accept-Encoding → compresses with brotli
          → It has fr    → matches Accept-Language  → French copy

Step 3  Server ANNOUNCES its choices with "Content-*" headers:
          Content-Type:     application/json; charset=utf-8
          Content-Encoding: br
          Content-Language: fr-FR

Step 4  Server MUST tell caches which request headers changed the answer:
          Vary: Accept-Encoding, Accept-Language
        Without this, a shared cache will serve one variant to everyone.
```

### Conditional requests — the 304 round trip

Caching has two independent decisions: **is my copy still fresh?** (freshness) and, if not, **has it actually changed?** (validation). Freshness is `Cache-Control: max-age`. Validation is `ETag` / `Last-Modified`.

```
──────────────── FIRST REQUEST (cold) ────────────────
Client:  GET /api/products/42 HTTP/1.1

Server:  HTTP/1.1 200 OK
         Cache-Control: max-age=60      ← fresh for 60 seconds
         ETag: "a1b2c3d4"               ← version fingerprint of this body
         Last-Modified: Sat, 12 Jul 2026 09:00:00 GMT
         Content-Length: 1287
         <...1287 bytes of body...>     ← full payload sent

Client caches the body + the ETag + the Last-Modified.

──── WITHIN 60s: request is served from cache. ZERO network. ────

──────────── AFTER 60s (stale → revalidate) ────────────
Client:  GET /api/products/42 HTTP/1.1
         If-None-Match: "a1b2c3d4"                    ← "do you still have this ETag?"
         If-Modified-Since: Sat, 12 Jul 2026 09:00:00 GMT

Server compares "a1b2c3d4" against the CURRENT version:

   ┌─ IF UNCHANGED ─────────────────────────────────┐
   │ HTTP/1.1 304 Not Modified                      │
   │ Cache-Control: max-age=60                       │
   │ ETag: "a1b2c3d4"                                │
   │ (NO BODY — saves 1287 bytes)                    │  ← the whole point
   │ Client reuses cached body, resets freshness.    │
   └────────────────────────────────────────────────┘

   ┌─ IF CHANGED ───────────────────────────────────┐
   │ HTTP/1.1 200 OK                                 │
   │ ETag: "z9y8x7w6"        ← new fingerprint       │
   │ Content-Length: 1350                            │
   │ <...new full body...>   ← full payload again    │
   └────────────────────────────────────────────────┘
```

That 304 is the workhorse of the web: the browser still makes a round trip, but pays only the RTT, not the download. `ETag` beats `Last-Modified` because it's exact (a hash) rather than 1-second-granular, and it detects "changed then reverted to identical content" correctly. If both are present, `ETag` / `If-None-Match` takes precedence. (Full caching treatment — CDN layers, `stale-while-revalidate`, `immutable` — is Topic 20.)

---

## Exact Syntax Breakdown

### `curl -I` — fetch headers only (HEAD request)

```
curl  -I         https://example.com
│     │          │
│     │          └── URL
│     └── --head: send a HEAD request, print ONLY the response headers
└── the tool

Use -I when you only care about status + headers (cache, redirects,
content type) and don't want to download the body.
```

### `curl -H` — send a custom request header

```
curl  -H 'Accept: application/json'  -H 'Authorization: Bearer TOKEN'  URL
│     │  │                           │
│     │  │                           └── repeat -H for each header you add
│     │  └── the literal "Name: Value" string to send
│     └── --header: add ONE request header

To send content negotiation preferences:
  curl -H 'Accept-Language: fr-FR'  -H 'Accept-Encoding: gzip'  URL

To test a conditional request (force a 304):
  curl -H 'If-None-Match: "a1b2c3d4"'  URL
```

### `curl -v` — see BOTH request and response headers

```
curl -v https://example.com

Line prefixes in -v output:
  *   ← curl's own diagnostics (DNS, TCP, TLS)
  >   ← REQUEST headers curl SENT   (outgoing)
  <   ← RESPONSE headers curl GOT   (incoming)

Example:
  > GET / HTTP/2
  > Host: example.com          ← what you sent
  > accept: */*
  >
  < HTTP/2 200                 ← what came back
  < content-type: text/html
  < cache-control: max-age=3600
```

### Useful companions

```
curl -sD - -o /dev/null URL          # dump response headers, discard body
curl -o /dev/null -sw '%{http_code}\n' URL   # print just the status code
```

---

## Example 1 — Basic

**Goal:** see the real headers a live server sends, and drive content negotiation by hand.

```bash
# 1. Response headers only — nice and clean
curl -I https://httpbin.org/json
```
```
HTTP/2 200
date: Sat, 12 Jul 2026 09:20:01 GMT
content-type: application/json          ← the server's chosen representation
content-length: 429
server: gunicorn/19.9.0
access-control-allow-origin: *
cache-control: no-cache                 ← "revalidate every time before using"
```

```bash
# 2. Ask for gzip and watch the server compress the body
curl -H 'Accept-Encoding: gzip' -I https://httpbin.org/json
```
```
HTTP/2 200
content-type: application/json
content-encoding: gzip                  ← server honored our Accept-Encoding
vary: Accept-Encoding                   ← correctly tells caches the body varies
```

```bash
# 3. See what YOU sent and what came back, together
curl -v https://httpbin.org/get 2>&1 | grep -E '^[<>]'
```
```
> GET /get HTTP/2
> Host: httpbin.org
> user-agent: curl/8.4.0
> accept: */*                           ← curl's default: "I accept anything"
>
< HTTP/2 200
< content-type: application/json
< content-length: 256
```

**What you just proved:**
- `curl -I` shows only response headers; `-v` shows both directions.
- Sending `Accept-Encoding: gzip` changed the server's behavior (it compressed) — that's content negotiation.
- The server added `Vary: Accept-Encoding` so a cache won't hand the gzip bytes to a client that didn't ask for them.
- HTTP/2 header names came back **lowercase** on the wire.

---

## Example 2 — Production Scenario

**The situation.** You run a product API behind a CDN. It serves the same endpoint in multiple languages based on `Accept-Language`, and gzip-compresses when asked. Support tickets flood in: **French users are seeing English text**, and a handful of users report **"garbled/binary junk" on the page.** Nothing changed in the code. It's intermittent — depends on who hit the cache first.

**Step 1 — Reproduce and read the headers.**

```bash
# What does the origin send for a French request?
curl -sD - -o /dev/null \
  -H 'Accept-Language: fr-FR' \
  -H 'Accept-Encoding: gzip' \
  https://api.shop.example.com/v1/banner
```
```
HTTP/2 200
content-type: text/html; charset=utf-8
content-language: fr-FR                  ← origin DID return French
content-encoding: gzip                   ← origin DID gzip
cache-control: public, max-age=600
age: 412                                 ← this came from the CDN cache (412s old)
x-cache: HIT
                                         ← ✗ NO Vary header anywhere!
```

**Step 2 — Spot the bug.** The response depends on **two** request headers (`Accept-Language` and `Accept-Encoding`) but carries **no `Vary` header**. So the CDN caches the *first* response it sees under a single key `/v1/banner` and serves it to everyone:

```
        First user (French, gzip)          Cache stores:
        ─────────────────────────►         key = "/v1/banner"
        GET /v1/banner                      value = <gzip French bytes>
        Accept-Language: fr-FR
        Accept-Encoding: gzip

        Next user (English, NO gzip)        Cache HIT on "/v1/banner"
        ─────────────────────────►         → returns GZIP FRENCH bytes
        GET /v1/banner                      → client didn't send Accept-Encoding
        Accept-Language: en-US              → can't decompress → BINARY GARBAGE
        (no Accept-Encoding)                → and it's French, not English
```

Both symptoms — wrong language and binary junk — are the **same root cause**: a missing `Vary`.

**Step 3 — Fix at the origin.** Tell every cache that this response is keyed by those two request headers:

```
Vary: Accept-Encoding, Accept-Language
```

Express/Node example of the fix:

```js
app.get('/v1/banner', (req, res) => {
  res.vary('Accept-Encoding');   // res.vary appends without duplicating
  res.vary('Accept-Language');
  const lang = pickLanguage(req.headers['accept-language']); // 'fr-FR' | 'en-US'
  res.set('Content-Language', lang);
  res.set('Cache-Control', 'public, max-age=600');
  res.send(renderBanner(lang));  // Express handles gzip + Content-Encoding
});
```

**Step 4 — Purge the poisoned cache and verify.**

```bash
# After purging the CDN, confirm the origin now varies correctly
curl -sD - -o /dev/null \
  -H 'Accept-Language: fr-FR' -H 'Accept-Encoding: gzip' \
  https://api.shop.example.com/v1/banner | grep -i '^vary'
```
```
vary: Accept-Encoding, Accept-Language   ← now the CDN caches ONE variant PER combo
```

```bash
# Prove the two variants are now distinct: English, no compression
curl -sD - -o /dev/null \
  -H 'Accept-Language: en-US' \
  https://api.shop.example.com/v1/banner | grep -iE '^(content-language|content-encoding|x-cache)'
```
```
content-language: en-US                  ← correct language now
                                         ← (no content-encoding: not gzipped)
x-cache: MISS                            ← different Vary key → separate cache entry
```

**The lesson.** Any response whose body changes based on a **request header** MUST list that header in `Vary`, or a shared cache will serve the wrong variant. The two most-forgotten values are `Accept-Encoding` (compression mismatch → garbage) and `Accept-Language` / `Authorization` (privacy + wrong-user data). This class of bug is invisible in local dev — it only appears once a shared cache/CDN sits in front.

---

## Common Mistakes

### Mistake 1: Response varies by a request header, but no `Vary`

```
WRONG:  Response body depends on Accept-Language / Accept-Encoding / Authorization,
        but the response has no Vary header.
        → A shared cache/CDN serves one variant to everyone (Example 2).

RIGHT:  Vary: Accept-Encoding, Accept-Language

Rule: if changing a request header would change the response body,
      that header MUST be in Vary. Careful: `Vary: *` or varying on a
      high-cardinality header (User-Agent, Cookie) can make the cache
      almost useless — every request becomes a unique key.
```

### Mistake 2: Caching private data publicly

```
WRONG:  Cache-Control: public, max-age=3600     on a per-user "/me" response.
        → A CDN/proxy stores user A's profile and serves it to user B.
          A genuine, headline-grade data leak.

RIGHT (per-user / authenticated data):
        Cache-Control: private, no-store
        or at least: Cache-Control: private, max-age=0, must-revalidate

Rules of thumb:
  public   → any shared cache may store it (safe for truly public assets)
  private  → only the end-user's browser may store it, never a shared proxy
  no-store → do not write to ANY cache, ever (tokens, PII, banking)
  no-cache → MAY store, but MUST revalidate with the origin before each use
```

### Mistake 3: `no-cache` vs `no-store` confusion (and `max-age` mixups)

```
Belief:  "no-cache means don't cache." → WRONG.

no-cache  = store it, but ALWAYS revalidate (ETag/If-None-Match) before serving.
            You still get 304s and save bandwidth. Great for HTML that changes.
no-store  = never write it to disk/RAM at all. Use for secrets/PII only.

Also wrong: setting `no-store` on your static JS/CSS bundles.
  → every page load re-downloads megabytes. Use long max-age + a hashed
    filename (app.9f3a1c.js) instead, ideally `max-age=31536000, immutable`.
```

### Mistake 4: Missing security headers

```
WRONG:  App served over HTTPS with none of:
        Strict-Transport-Security, Content-Security-Policy,
        X-Content-Type-Options, X-Frame-Options.
        → downgrade attacks, XSS, clickjacking, MIME-sniffing all open.

RIGHT (sane baseline for an HTML-serving app):
        Strict-Transport-Security: max-age=31536000; includeSubDomains
        X-Content-Type-Options: nosniff
        Content-Security-Policy: default-src 'self'
        X-Frame-Options: DENY            (or CSP frame-ancestors 'none')
        Referrer-Policy: strict-origin-when-cross-origin

Note: HSTS only takes effect over HTTPS and is remembered by the browser.
Ship it too aggressively (huge max-age + preload) before you're ready to
be HTTPS-forever, and you can lock users out of the domain over HTTP.
```

### Mistake 5: Treating header names as case-sensitive

```
WRONG (Node.js):
  const type = req.headers['Content-Type'];   // undefined!
  → Node LOWERCASES all incoming header names. The key is 'content-type'.

RIGHT:
  const type = req.headers['content-type'];   // works

Header NAMES are case-insensitive per spec. Never compare them with ===
against a fixed case. Frameworks normalize to lowercase; rely on that,
or use a case-insensitive getter (res.get('Content-Type') in Express).
```

### Mistake 6: Oversized headers (and stuffing state into them)

```
WRONG:  Cramming a giant JWT + 30 cookies + a base64 blob into headers.
        → Most servers cap total header size at ~8 KB (nginx
          large_client_header_buffers, Node ~16 KB). Exceed it and you get:
          431 Request Header Fields Too Large  (or a silent 400/dropped conn).

Common trigger: cookie bloat. Every cookie on the domain is re-sent on
EVERY request to it. A 4 KB session cookie × every asset request = waste
and, past the limit, hard failures. Keep tokens lean; store big state
server-side and reference it by a small ID.
```

---

## Hands-On Proof

```bash
# 1. See a real Cache-Control + ETag on a public asset
curl -I https://www.google.com/favicon.ico
#   Look for: cache-control, expires, age, etag

# 2. Prove the 304 round trip yourself.
#    First grab the ETag:
ETAG=$(curl -sI https://httpbin.org/etag/demo123 | awk -F'"' '/etag/{print $2}')
echo "ETag = $ETAG"
#    Now send it back as If-None-Match and watch for 304:
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "If-None-Match: \"$ETAG\"" \
  https://httpbin.org/etag/demo123
#    → prints 304  (server says "you already have it, no body sent")

# 3. Drive content negotiation and read the echo
curl -s -H 'Accept-Language: de-DE,de;q=0.9' \
        -H 'Accept-Encoding: gzip, br' \
        https://httpbin.org/headers
#    httpbin echoes back exactly the request headers it received

# 4. Watch request AND response headers side by side
curl -v https://github.com 2>&1 | grep -E '^[<>]'
#    > lines = what you sent, < lines = what GitHub sent (note the
#    security headers: strict-transport-security, content-security-policy)

# 5. Audit a site's security headers in one shot
curl -sI https://github.com | grep -iE \
  'strict-transport|content-security|x-frame|x-content-type|referrer-policy|permissions-policy'

# 6. Trigger the oversized-header failure on purpose (optional)
curl -H "X-Big: $(head -c 20000 </dev/zero | tr '\0' 'a')" -I https://httpbin.org/get
#    → often 431 Request Header Fields Too Large, or a dropped connection
```

---

## Practice Exercises

### Exercise 1 — Easy: Read a real cache profile
```bash
# Pick any static asset and dump its headers:
curl -I https://cdn.jsdelivr.net/npm/react@18/package.json

# Answer from the output:
# 1. What is the max-age? How many hours/days of freshness is that?
# 2. Is it "public" or "private"? Why is that correct for this asset?
# 3. Does it carry an ETag? A Last-Modified? Both?
# 4. If your browser already has this file cached and it's still fresh,
#    how many bytes travel over the network on the next request?
```

### Exercise 2 — Medium: Force and confirm a 304
```bash
# Step 1: capture the ETag of a resource
curl -sI https://httpbin.org/etag/hello | grep -i etag

# Step 2: replay the request WITH If-None-Match set to that ETag.
#         Print only the status code.
curl -s -o /dev/null -w 'status=%{http_code}\n' \
  -H 'If-None-Match: "hello"' https://httpbin.org/etag/hello

# Step 3: now send a WRONG ETag and print the status.
curl -s -o /dev/null -w 'status=%{http_code}\n' \
  -H 'If-None-Match: "wrong-tag"' https://httpbin.org/etag/hello

# Answer:
# 1. Which request returned 304 and which returned 200? Why?
# 2. In the 304 case, was a body sent? (add -sD - and inspect content-length)
# 3. Explain in one sentence how this saves bandwidth at scale.
```

### Exercise 3 — Hard (Production Simulation): Diagnose a cache-poisoning bug
```
Scenario: An endpoint /api/greeting returns JSON in the user's language
based on Accept-Language, and gzips when Accept-Encoding: gzip is present.
It sits behind a shared CDN. Users report: sometimes they get the wrong
language; sometimes they get unreadable binary. The code "hasn't changed."

Your tasks:
# 1. Using curl, capture the FULL response headers for two different
#    requests to the same URL (one fr-FR + gzip, one en-US no-gzip):
curl -sD - -o /dev/null -H 'Accept-Language: fr-FR' -H 'Accept-Encoding: gzip' URL
curl -sD - -o /dev/null -H 'Accept-Language: en-US'                              URL

# 2. Identify the ONE missing response header that causes BOTH symptoms.
# 3. Write the exact header line the origin must add.
# 4. Explain: why does the bug NEVER show up when you test locally without
#    a CDN in front? What is the general rule for when Vary is required?
# 5. Bonus: name one request header you should NEVER blindly add to Vary,
#    and explain what it does to your cache hit rate.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the linked section.

1. **A response body changes depending on the `Accept-Encoding` request header. What single response header must you add, and what breaks in a CDN if you forget it?**

2. **Explain the difference between `Cache-Control: no-cache` and `Cache-Control: no-store` in one sentence each.** Which one still lets you get 304s?

3. **Walk the ETag conditional-request round trip:** what does the server send first, what does the client echo back on revalidation, and what status + body does the server return if nothing changed?

4. **Your `/api/me` endpoint returns the logged-in user's profile. What `Cache-Control` value must it have, and what disaster happens if you set `public, max-age=3600` instead?**

5. **Name four security response headers and, for each, the one attack it defends against.** (Hint: HSTS, CSP, nosniff, frame-ancestors/X-Frame-Options.)

6. **Why does `req.headers['Content-Type']` return `undefined` in Node.js while `req.headers['content-type']` works?** What rule about header names explains this?

7. **What is the practical difference between the `Accept` header and the `Content-Type` header?** Which side sends which, and what does each describe?

---

## Quick Reference Card

### Request headers (client → server)
| Header | What it says | Example |
|--------|--------------|---------|
| `Host` | Which virtual host on this IP | `Host: api.shop.com` |
| `User-Agent` | Client software identity | `User-Agent: curl/8.4.0` |
| `Accept` | Body formats I can handle | `Accept: application/json` |
| `Accept-Encoding` | Compressions I can decode | `Accept-Encoding: gzip, br` |
| `Accept-Language` | Languages I prefer | `Accept-Language: fr-FR,en;q=0.5` |
| `Authorization` | Credentials | `Authorization: Bearer eyJ…` |
| `Cookie` | Stored cookies for this domain | `Cookie: session=abc123` |
| `Referer` | Page that linked here (note: misspelled in spec) | `Referer: https://google.com/` |
| `Origin` | Scheme+host of the calling page (CORS/CSRF) | `Origin: https://app.shop.com` |
| `If-None-Match` | Revalidate against this ETag | `If-None-Match: "a1b2c3"` |
| `If-Modified-Since` | Revalidate against this date | `If-Modified-Since: Sat, 12 Jul 2026 09:00:00 GMT` |

### Response headers (server → client)
| Header | What it says | Example |
|--------|--------------|---------|
| `Content-Type` | Body's MIME type + charset | `Content-Type: application/json; charset=utf-8` |
| `Content-Length` | Body size in bytes | `Content-Length: 1287` |
| `Content-Encoding` | Compression applied to body | `Content-Encoding: gzip` |
| `Content-Language` | Language of the body | `Content-Language: fr-FR` |
| `Set-Cookie` | Store this cookie | `Set-Cookie: session=abc; HttpOnly; Secure` |
| `Location` | Redirect / created-resource URL | `Location: /api/products/42` |
| `Cache-Control` | Caching policy | `Cache-Control: public, max-age=300` |
| `ETag` | Version fingerprint of the body | `ETag: "a1b2c3d4"` |
| `Last-Modified` | When the body last changed | `Last-Modified: Sat, 12 Jul 2026 09:00:00 GMT` |
| `Vary` | Request headers this response depends on | `Vary: Accept-Encoding, Accept-Language` |
| `Server` | Origin software (often omit for security) | `Server: nginx` |

### Cache-Control directives
| Directive | Meaning |
|-----------|---------|
| `max-age=N` | Fresh for N seconds |
| `public` | Any cache (incl. shared CDN/proxy) may store |
| `private` | Only the end-user's browser may store |
| `no-cache` | May store, but must revalidate before each use |
| `no-store` | Never store anywhere (secrets/PII) |
| `must-revalidate` | Once stale, MUST revalidate — never serve stale on error |
| `immutable` | Content will never change; don't even revalidate |

### Security headers — what each defends against
| Header | Defends against |
|--------|-----------------|
| `Strict-Transport-Security` | HTTPS→HTTP downgrade / SSL-strip; forces HTTPS |
| `Content-Security-Policy` | XSS / unauthorized script & resource loading |
| `X-Content-Type-Options: nosniff` | MIME-sniffing (browser guessing a safer type into a dangerous one) |
| `X-Frame-Options` / CSP `frame-ancestors` | Clickjacking (your page embedded in a hostile iframe) |
| `Referrer-Policy` | Leaking full URLs (with tokens) via the `Referer` header |
| `Permissions-Policy` | Unwanted use of camera / mic / geolocation etc. |

### Cheat block
```
# Headers only
curl -I URL

# Send custom request headers
curl -H 'Accept: application/json' -H 'Authorization: Bearer TOKEN' URL

# See request (>) AND response (<) headers
curl -v URL 2>&1 | grep -E '^[<>]'

# Dump response headers, discard body
curl -sD - -o /dev/null URL

# Force a conditional request (expect 304)
curl -H 'If-None-Match: "etag-value"' -I URL

# Audit security headers
curl -sI URL | grep -iE 'strict-transport|content-security|x-frame|x-content-type|referrer'

REMEMBER:
  Header NAMES  = case-insensitive     (Content-Type == content-type)
  HTTP/2 & /3   = names forced lowercase, compressed (HPACK/QPACK)
  Vary          = required whenever the body depends on a request header
  private/no-store = per-user & secret data must NEVER hit a shared cache
```

---

## When Would I Use This at Work?

### Scenario 1: "The mobile app gets HTML instead of JSON"
Your endpoint content-negotiates. The app forgot to send `Accept: application/json`, so your server fell back to its default (HTML). Fix the client to send `Accept`, and/or make your API default to JSON. One `curl -H 'Accept: application/json'` reproduces and confirms it in seconds.

### Scenario 2: "Our CDN bill and origin load are way too high"
Your responses have no `Cache-Control` or `no-cache` everywhere, so nothing caches and every request hits the origin. Add `Cache-Control: public, max-age=…` with `ETag` validation to static and semi-static responses. Cacheable assets move to `max-age=31536000, immutable` with hashed filenames. Origin traffic drops by an order of magnitude; the 304 flow handles the rest.

### Scenario 3: "Security audit flagged us — missing HSTS and CSP"
A pentest report lists missing `Strict-Transport-Security`, `Content-Security-Policy`, and `X-Content-Type-Options`. You add them at the app or reverse-proxy layer, verify with `curl -sI | grep -iE 'strict-transport|content-security'`, and re-test. These are backend responsibilities — the frontend cannot set response headers.

### Scenario 4: "One user's request is failing — find it across 10 services"
You propagate `X-Request-ID` (or the W3C `traceparent`) from the edge through every internal call and log it everywhere. Now a single "request 3f9a…" search reconstructs the whole path. Modern convention: prefer `traceparent` / unprefixed names over `X-` (RFC 6648 deprecated the `X-` convention because yesterday's "experimental" header becomes tomorrow's standard and the prefix sticks forever).

### Scenario 5: "Login works in Postman but not the browser"
The browser is enforcing rules Postman ignores. It's almost always a header: the response is missing `Access-Control-Allow-Origin` (CORS, Topic 17), or `Set-Cookie` lacks `SameSite`/`Secure` so the cookie is dropped (Topic 19), or an `Authorization` header the browser won't forward cross-origin. `curl -v` vs the browser's Network tab shows the difference immediately.

---

## Connected Topics

**Study before this:**
- `10-http1.1-in-depth.md` — the request/response format, methods, and status codes these headers attach to.
- `09-tls-ssl-in-depth.md` — why headers are safe (encrypted) over HTTPS and exposed over HTTP.

**Study next:**
- `15-rest-over-http.md` — how method semantics, status codes, and headers combine into a clean API.
- `16-authentication-over-http.md` — deep dive on `Authorization`, `WWW-Authenticate`, Bearer tokens, and cookie auth.
- `17-cors-in-depth.md` — `Origin`, `Access-Control-*`, and the preflight `OPTIONS` request in full.
- `19-cookies-and-sessions.md` — everything `Set-Cookie` / `Cookie` (`HttpOnly`, `Secure`, `SameSite`).
- `20-caching-in-depth.md` — the complete caching story: browser vs CDN vs server, every `Cache-Control` directive, `stale-while-revalidate`, and the full ETag/Last-Modified validation flow introduced here.

**This topic is the vocabulary of HTTP.** Almost every remaining Phase 3 topic is really "one specific family of headers, in depth."

---

*Doc saved: `/docs/networking/14-http-headers-in-depth.md`*
