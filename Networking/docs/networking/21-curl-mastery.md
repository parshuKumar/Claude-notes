# 21 — curl Mastery

> **Phase 4 — Topic 1 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a web browser is a fancy restaurant with a waiter, table service, menus, and photos on the wall. You point at pictures, the waiter fetches things, and everything is dressed up nicely.

`curl` is walking into the **kitchen** and talking to the cook directly.

- No waiter interpreting your order (no browser rendering, no JavaScript).
- You say *exactly* what you want, in the exact words the cook understands ("one HTTP GET, hold the cookies, extra Authorization header").
- The cook hands you back *exactly* what came out of the pan — raw, unplated, no garnish (the raw HTTP response).

Because you're talking straight to the kitchen, you can prove what the kitchen actually does — separate from whatever the fancy dining room *shows* you. When your app says "the API is broken," curl lets you walk into the kitchen and ask the cook yourself. If the cook makes the dish perfectly for you but the dining room still serves it wrong, you now know the bug is in the dining room (your app), not the kitchen (the server).

That's the whole point of curl: **remove every layer between you and the raw HTTP conversation.**

---

## Where This Lives in the Network Stack

curl is a **Layer 7 (Application)** client. It is a program that *drives* the whole stack downward for you — it speaks HTTP, and to do that it triggers DNS, TCP, and TLS underneath, exactly as your app would.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP/1.1, HTTP/2, HTTP/3)  ← curl SPEAKS this
│  Layer 6 — Presentation  (TLS/SSL)                   ← curl negotiates this
│  Layer 5 — Session       (connection reuse)                     │
│  Layer 4 — Transport     (TCP / QUIC-over-UDP, ports)  ← curl opens this
│  Layer 3 — Network       (IP — DNS resolves the address)  ← curl triggers this
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (cables, fiber, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

The magic of curl for a backend developer: with flags like `-v`, `--trace-ascii`, and `-w`, it **exposes every one of those layers as text you can read**. DNS time, TCP connect time, TLS handshake time, the exact bytes of your HTTP request — all visible. Your browser hides all of this. curl surfaces it.

`curl` is built on **libcurl**, the same C library embedded inside countless languages and tools. When you learn curl flags, you're learning the vocabulary of HTTP itself.

---

## What Is This?

`curl` = **"Client for URLs."** It is a command-line tool (and library) for transferring data to and from a server using a URL. It was first released in 1997 and ships by default on macOS, most Linux distributions, and Windows 10+.

It supports far more than HTTP — FTP, SMTP, IMAP, MQTT, and more — but for a backend developer, **99% of your use is HTTP and HTTPS**.

What curl gives you that a browser or Postman does not:

- **Total control.** You set the exact method, headers, body, TLS version, and even which IP to connect to. Nothing is added behind your back except a couple of defaults you can see and override.
- **Reproducibility.** A curl command is a plain string. You can paste it into a ticket, a Slack thread, or a runbook, and anyone can run the *exact* same request. Browser DevTools even has "Copy as cURL."
- **Scriptability.** It composes into shell scripts, CI pipelines, health checks, and `for` loops.
- **Raw truth.** curl shows you the literal HTTP bytes. No rendering, no auto-retries you didn't ask for, no silent cookie storage.

Check your version — flags and defaults have changed over the years:

```
$ curl --version
curl 8.7.1 (x86_64-apple-darwin25.0) libcurl/8.7.1 (SecureTransport)
  LibreSSL/3.3.6 zlib/1.2.12 nghttp2/1.68.0
Release-Date: 2024-03-27
Protocols: dict file ftp ftps ... http https imap imaps ... smtp smtps ...
Features: alt-svc AsynchDNS HSTS HTTP2 HTTPS-proxy IPv6 ... SSL threadsafe
```

The `Features` line matters: `HTTP2` means `--http2` works, `IPv6` means it can use v6, and the SSL library name (here LibreSSL/SecureTransport) affects `--tlsv1.3` and client-cert behavior.

---

## Why Does It Matter for a Backend Developer?

curl is the single most important debugging tool in your kit. When something is wrong with an HTTP service, curl is the first thing you reach for. Here is why:

- **It isolates client from server.** Your app says "the API returns 500." Is it your code, your headers, your auth token, or the server? Send the *exact same request* with curl. If curl also gets 500, the server is at fault. If curl works, your app is doing something different — and now you diff the two.
- **It tells you where latency lives.** The `-w` timing breakdown splits a request into DNS, TCP, TLS, server-processing, and transfer time. "The API is slow" becomes "TLS handshake is taking 400ms" — a completely different fix. (This is the entire subject of Topic 27.)
- **It reproduces production bugs safely.** You can replay a failing request from your laptop, tweak one header at a time, and find exactly which header breaks it — without redeploying your app.
- **It tests infrastructure before it goes live.** `--resolve` lets you hit a brand-new server by IP while still sending the real hostname and getting the real certificate — so you can validate a server *before* you cut DNS over to it.
- **It's the universal contract.** Every backend engineer, every SRE, every API doc uses curl examples. "Send us this curl command and paste the output" is the universal support request.

If you internalize one tool from this entire phase, make it curl.

---

## The Packet/Protocol Anatomy

curl doesn't have its own wire format — it *produces* the HTTP request on the wire. When you run `curl https://api.myapp.com/users`, the bytes curl generates and puts inside the TLS tunnel are:

```
┌──────────────────────────────────────────────────────────────┐
│ WHAT CURL ACTUALLY WRITES TO THE SOCKET (an HTTP/1.1 request) │
├──────────────────────────────────────────────────────────────┤
│ GET /users HTTP/1.1\r\n           ← request line: METHOD PATH VERSION
│ Host: api.myapp.com\r\n            ← curl adds this automatically
│ User-Agent: curl/8.7.1\r\n         ← curl adds this automatically
│ Accept: */*\r\n                    ← curl adds this automatically
│ \r\n                               ← blank line = end of headers
│ (no body for a GET)                                           │
└──────────────────────────────────────────────────────────────┘
```

Those three auto-added headers (`Host`, `User-Agent`, `Accept`) are the *only* things curl adds without being told. Everything else you control. You can override any of them with `-H`.

When you add a body (`-d '...'`), the anatomy changes — curl adds `Content-Length` and a default `Content-Type`:

```
POST /users HTTP/1.1\r\n
Host: api.myapp.com\r\n
User-Agent: curl/8.7.1\r\n
Accept: */*\r\n
Content-Length: 31\r\n                      ← curl counts the body bytes
Content-Type: application/x-www-form-urlencoded\r\n  ← DEFAULT for -d (surprise!)
\r\n                                          ← blank line ends headers
{"name":"Alice","role":"admin"}              ← the body
```

That default `Content-Type` is a classic trap: **`-d` sends `application/x-www-form-urlencoded` unless you override it.** If your API expects JSON, you must add `-H "Content-Type: application/json"` (or use `--json`). More on this in Common Mistakes.

The direction markers curl uses in verbose mode map directly onto this anatomy:

```
> = a byte curl SENT   (your request line, headers, body)
< = a byte curl RECEIVED (the server's status line, headers, body)
* = curl's own commentary (DNS, TCP, TLS — not HTTP bytes)
```

---

## How It Works — Step by Step

Let's trace exactly what `curl -v https://api.github.com/zen` does, from pressing Enter to printing the body. Every phase maps to a timing variable you'll meet in the `-w` section.

```
YOU RUN: curl -v https://api.github.com/zen
```

### Step 1 — Parse the URL
```
scheme  = https   → port 443, TLS required
host    = api.github.com
path    = /zen
```
curl decides: default method GET, no body, follow-redirects OFF (unless `-L`).

### Step 2 — DNS Resolution  → measured by time_namelookup
```
* Host api.github.com:443 was resolved.
* IPv6: (list of AAAA records)
* IPv4: 140.82.112.6, ...
```
curl asks the OS resolver for the IP. `--resolve` or `--connect-to` can short-circuit this. (Topic 22 covers DNS debugging in depth.)

### Step 3 — TCP Connect  → time_connect
```
*   Trying 140.82.112.6:443...
* Connected to api.github.com (140.82.112.6) port 443
```
The TCP 3-way handshake (SYN → SYN-ACK → ACK). `time_connect` is the moment the socket is open. (Topic 08 covers TCP.)

### Step 4 — TLS Handshake  → time_appconnect
```
* ALPN: curl offers h2,http/1.1              ← curl offers to speak HTTP/2
* (OUT), TLS handshake, Client hello (1):
* (IN),  TLS handshake, Server hello (2):
* (IN),  TLS handshake, Certificate (11):
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* ALPN: server accepted h2                   ← server chose HTTP/2
* Server certificate:
*  subject: CN=github.com
*  issuer: ...
*  SSL certificate verify ok.               ← chain validated to a trusted root
```
`time_appconnect` = TLS is up and the secure channel is ready. (Topic 09 covers TLS.) For plain `http://`, this step is skipped and `time_appconnect` = 0.

### Step 5 — Send the HTTP Request  → time_pretransfer
```
> GET /zen HTTP/2
> Host: api.github.com
> User-Agent: curl/8.7.1
> Accept: */*
>
```
`time_pretransfer` = curl is done with all setup and about to send. Everything is queued.

### Step 6 — Receive the Response  → time_starttransfer (TTFB)
```
< HTTP/2 200
< content-type: text/plain; charset=utf-8
< content-length: 33
<
Keep your working spaces clean.   ← the response body
```
`time_starttransfer` = the **first byte** of the response body arrived. The gap between `time_pretransfer` and `time_starttransfer` is your **server's processing time** (plus one network round-trip). This is TTFB — Time To First Byte.

### Step 7 — Finish and Close  → time_total
```
* Connection #0 to host api.github.com left intact
```
`time_total` = the last byte arrived and the transfer is complete. curl prints the body to stdout and exits. Exit code 0 means the transfer succeeded (note: **an HTTP 500 is still a "success" to curl** — it received a valid response; use `-f`/`--fail` to make HTTP errors non-zero).

---

## Exact Syntax Breakdown

Here is a realistic production-grade command. Every flag matters. We'll dissect it.

```bash
curl -sS -X POST \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"name":"Alice","role":"admin"}' \
     --compressed \
     -w "\n[%{http_code}] %{time_total}s\n" \
     https://api.myapp.com/v1/users
```

```
curl                                    ← the program
 -sS                                    ← -s silent (no progress bar)
 │                                        + -S show errors anyway
 -X POST                                ← set method to POST (see note below)
 -H "Content-Type: application/json"    ← override the default form content-type
 -H "Authorization: Bearer $TOKEN"      ← auth header; $TOKEN expanded by the shell
 -d '{"name":"Alice","role":"admin"}'   ← request BODY (this is what makes it a POST)
 --compressed                           ← ask server to gzip/br the response, auto-decode
 -w "\n[%{http_code}] %{time_total}s\n" ← after the body, print status + total time
 https://api.myapp.com/v1/users         ← the target URL (always last, by convention)
```

**Critical subtlety:** `-X POST` alone does *not* create a POST body. It only changes the method name. The thing that actually makes this a real POST with a payload is `-d`. In fact, **`-d` implies `-X POST` all by itself** — so the `-X POST` above is redundant when `-d` is present. You write `-X` explicitly only when you need a method that doesn't happen automatically (like `-X PUT` or `-X DELETE`).

Reading order tip: curl doesn't care about flag order (except that the last non-flag argument is the URL). But by convention, keep the URL last and group related flags for readability.

---

## Example 1 — Basic

### GET (the default)

```bash
curl https://api.github.com/zen
```
Output (just the body, straight to your terminal):
```
Keep your working spaces clean.
```
No `-X` needed. **GET is the default method.** No body, no fuss.

### See the whole conversation with -i and -v

```bash
# -i  = include the RESPONSE headers in the output (body + headers)
curl -i https://api.github.com/zen
```
```
HTTP/2 200
server: GitHub.com
content-type: text/plain; charset=utf-8
x-ratelimit-limit: 60
x-ratelimit-remaining: 58
content-length: 42

Approachable is better than simple.
```

```bash
# -v = verbose: DNS, TCP, TLS, the request you SENT, the response you got
curl -v https://api.github.com/zen
```
```
*   Trying 140.82.112.6:443...
* Connected to api.github.com (140.82.112.6) port 443
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* Server certificate:
*  subject: CN=github.com
*  SSL certificate verify ok.
> GET /zen HTTP/2          ← ">" lines are what curl SENT
> Host: api.github.com
> User-Agent: curl/8.7.1
> Accept: */*
>
< HTTP/2 200               ← "<" lines are what the server SENT BACK
< content-type: text/plain; charset=utf-8
<
Approachable is better than simple.
```

### A basic POST with a JSON body

```bash
curl -X POST https://httpbin.org/post \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice","role":"admin"}'
```
httpbin echoes back what it received, proving the body and headers arrived intact:
```json
{
  "data": "{\"name\":\"Alice\",\"role\":\"admin\"}",
  "headers": {
    "Content-Length": "31",
    "Content-Type": "application/json",   ← our override took effect
    "Host": "httpbin.org",
    "User-Agent": "curl/8.7.1"
  },
  "json": { "name": "Alice", "role": "admin" },   ← server parsed it as JSON ✓
  "url": "https://httpbin.org/post"
}
```

That's it — GET, headers, verbose, POST. You already have enough to debug 80% of API problems.

---

## Example 2 — Production Scenario

**The situation:** Your Node.js app calls the payments API at `https://api.payments.internal/v1/charge`. In production, the app logs `402 Payment Required` for a request that *should* succeed. QA swears the same charge works from Postman. You need to answer one question: **is the bug in our app, or on the server?**

### Step 1 — Reproduce the *exact* request the app sends

Pull the real headers, body, and auth from your app's logs, then replay them byte-for-byte with curl:

```bash
curl -v -X POST https://api.payments.internal/v1/charge \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PAYMENTS_TOKEN" \
  -H "Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7" \
  -d '{"amount":4999,"currency":"usd","source":"tok_visa"}'
```

The verbose output shows the server responds `402` — so curl reproduces the bug. **The server is not the problem in isolation; something about the request is.** Now bisect the headers one at a time. You remove `Idempotency-Key`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.payments.internal/v1/charge \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PAYMENTS_TOKEN" \
  -d '{"amount":4999,"currency":"usd","source":"tok_visa"}'
# → 200
```

It succeeds. **Root cause found:** the app is re-using a stale `Idempotency-Key` from a previously-declined charge, and the server correctly replays the old `402`. The bug is client-side. Without curl, this is hours of guessing; with curl, it's three commands.

### Step 2 — Test the new server BEFORE the DNS cutover

You're migrating the payments service to a new host `10.20.30.40`, but DNS still points `api.payments.internal` at the old box. You want to prove the new server serves correct responses **and** presents a valid TLS cert for the real hostname — without touching DNS or `/etc/hosts`.

```bash
# --resolve forces curl to connect to 10.20.30.40:443,
# but still send Host: api.payments.internal AND validate the cert for that name.
curl -v --resolve api.payments.internal:443:10.20.30.40 \
  https://api.payments.internal/healthz
```
```
* Added api.payments.internal:443:10.20.30.40 to DNS cache
*   Trying 10.20.30.40:443...              ← went straight to the NEW box
* Connected to api.payments.internal (10.20.30.40) port 443
* Server certificate:
*  subject: CN=api.payments.internal      ← cert is valid for the real name ✓
*  SSL certificate verify ok.
< HTTP/2 200
{"status":"ok"}
```

The new server is healthy and its certificate is valid. You can cut DNS over with confidence. (`--connect-to api.payments.internal:443:10.20.30.40:443` does the same thing and is handy when ports differ.)

### Step 3 — Find where the latency lives

The new server is up but users report it feels slow. Use the timing breakdown to split the request into phases:

```bash
curl -o /dev/null -s -w "\
DNS lookup:          %{time_namelookup}s
TCP connect:         %{time_connect}s
TLS handshake:       %{time_appconnect}s
Server processing:   %{time_starttransfer}s
Total:               %{time_total}s
HTTP code:           %{http_code}
Downloaded:          %{size_download} bytes
" --resolve api.payments.internal:443:10.20.30.40 \
  https://api.payments.internal/v1/charge
```
```
DNS lookup:          0.004s
TCP connect:         0.021s     ← TCP fine (17ms RTT)
TLS handshake:       0.310s     ← ⚠ TLS took ~290ms — way too long
Server processing:   0.325s     ← server itself is fast (15ms after TLS)
Total:               0.331s
HTTP code:           200
Downloaded:          128 bytes
```

The whole cost is in the **TLS handshake** — 290ms. The server is doing full TLS 1.2 handshakes with a huge cert chain and no session resumption. The fix is server-side config (enable TLS 1.3, session tickets, OCSP stapling), not app code. Again: curl turned "it's slow" into a precise, actionable finding. (Topic 27 builds a full slow-API methodology on exactly this technique.)

---

## Common Mistakes

### Mistake 1: `-X POST` with no `-d` — expecting a body that isn't there
```bash
# WRONG — this sends a POST with an EMPTY body
curl -X POST https://api.myapp.com/users -H "Content-Type: application/json"
# Server sees Content-Length: 0 and returns 400 "missing body"
```
`-X` only renames the method. **The body comes from `-d` / `--data` / `--json` / `-F`.**
```bash
# RIGHT — the -d both supplies the body AND implies POST
curl -d '{"name":"Alice"}' -H "Content-Type: application/json" https://api.myapp.com/users
```
Corollary: when you use `-d`, you rarely need `-X POST` at all — `-d` sets the method to POST for you.

---

### Mistake 2: Sending JSON but getting `application/x-www-form-urlencoded`
```bash
# WRONG — -d defaults the Content-Type to form-urlencoded, NOT json
curl -d '{"name":"Alice"}' https://api.myapp.com/users
# Server tries to parse JSON as a form → 400 / null fields
```
`-d` always defaults to `Content-Type: application/x-www-form-urlencoded`. You must override it, or use `--json` which sets both the header **and** the Accept header for you:
```bash
# RIGHT (explicit)
curl -H "Content-Type: application/json" -d '{"name":"Alice"}' https://api.myapp.com/users
# RIGHT (curl 7.82+ shortcut: sets Content-Type AND Accept to application/json)
curl --json '{"name":"Alice"}' https://api.myapp.com/users
```

---

### Mistake 3: Shell-quoting JSON wrong (the `$`, `!`, and `"` traps)
```bash
# WRONG — double quotes let the shell eat $role and mangle the JSON
curl -d "{\"name\":\"$name\",\"role\":\"admin\"}" ...   # brittle, $name may be empty
# WRONG — the ! triggers history expansion in interactive bash
curl -d '{"msg":"hi!"}' ...
```
**Rule: wrap JSON bodies in single quotes** so the shell leaves them completely alone:
```bash
# RIGHT
curl -d '{"name":"Alice","role":"admin"}' -H "Content-Type: application/json" ...
```
When the body genuinely contains a shell variable, or is large, read it from a file or stdin with `@`:
```bash
curl -d @payload.json -H "Content-Type: application/json" ...   # body from file
echo "$JSON" | curl -d @- -H "Content-Type: application/json" ...   # body from stdin
```

---

### Mistake 4: Using `-k` / `--insecure` in production (and shipping it)
```bash
# DANGEROUS — disables ALL certificate verification
curl -k https://api.myapp.com/users
```
`-k` tells curl to accept *any* certificate, including expired, self-signed, wrong-hostname, or an attacker's cert in a man-in-the-middle. It "makes the error go away" — which is exactly why it's dangerous. A TLS error is curl protecting you; `-k` silences the alarm. It leaks into scripts, cron jobs, and health checks and quietly disables security everywhere. **Fix the actual cause instead:**
```bash
# Corporate/internal CA not trusted? Point curl at the right CA bundle:
curl --cacert /etc/ssl/internal-ca.pem https://api.internal/users
# Testing a server before DNS? Use --resolve so the cert still validates for the real name:
curl --resolve api.myapp.com:443:10.0.0.5 https://api.myapp.com/users
```

---

### Mistake 5: Forgetting `-L` and reading a redirect body instead of the real response
```bash
# WRONG — server 301-redirects http→https; without -L curl STOPS at the 301
curl http://api.myapp.com/users
# Output: <html><body>301 Moved Permanently</body></html>   ← not your data!
```
By default curl does **not** follow redirects. Add `-L` to follow them:
```bash
curl -L http://api.myapp.com/users     # follows the 301 to https and returns real data
```
Watch the method-change gotcha: on a **301/302** redirect, curl (like browsers) may rewrite your POST into a GET and drop the body. On **307/308** the method and body are preserved. If a POST silently "becomes" a GET after a redirect, that's why — check the status code with `-v`.

---

### Mistake 6: Confusing `-i` (include headers) with `-I` (HEAD request)
```bash
curl -i https://api.myapp.com/users   # GET; prints RESPONSE HEADERS + BODY
curl -I https://api.myapp.com/users   # HEAD; sends a HEAD request, prints HEADERS ONLY (no body)
```
- **lowercase `-i`** = still a normal GET, just *also* show the response headers above the body. Use it to inspect status + headers + body together.
- **UPPERCASE `-I`** = send an HTTP **HEAD** request. The server returns headers and *no body at all*. Great for checking `Content-Length`, `Last-Modified`, or whether a URL is alive — but if the endpoint behaves differently for HEAD vs GET, you'll be testing the wrong thing.

---

### Mistake 7: Not URL-encoding query/form data
```bash
# WRONG — spaces and & in the value break the request
curl -d "q=hello world&filter=a&b" https://api.myapp.com/search
# The & is read as a field separator; "hello world" space may be mishandled
```
Use `--data-urlencode` to encode a single field safely, and `-G` to move `-d` data into the query string of a GET:
```bash
# RIGHT — each value encoded correctly; -G makes it GET /search?q=hello%20world&...
curl -G https://api.myapp.com/search \
     --data-urlencode "q=hello world & more" \
     --data-urlencode "filter=a&b"
# Produces: GET /search?q=hello%20world%20%26%20more&filter=a%26b
```

---

## Hands-On Proof

Run these against public test servers to see every concept fire for real. `httpbin.org` echoes back whatever you send, so it's perfect for proving what actually went on the wire.

```bash
# 1. Prove GET is the default (no -X needed)
curl https://httpbin.org/get

# 2. Prove your headers arrive — httpbin echoes them back
curl -H "X-Trace: proof-123" -A "my-service/1.0" https://httpbin.org/headers

# 3. Prove -d makes a POST and see the default Content-Type trap
curl -d "a=1&b=2" https://httpbin.org/post
#    Look for: "Content-Type": "application/x-www-form-urlencoded"

# 4. Prove --json sets the right header for you
curl --json '{"x":1}' https://httpbin.org/post
#    Look for: "Content-Type": "application/json"

# 5. Prove -L follows redirects (and count them with -w)
curl -sL -o /dev/null -w "final=%{http_code} redirects=%{num_redirects} url=%{url_effective}\n" \
  https://httpbin.org/redirect/3
#    → final=200 redirects=3 url=https://httpbin.org/get

# 6. Prove -I sends HEAD (no body comes back)
curl -I https://httpbin.org/get

# 7. Prove Basic auth (-u) builds the Authorization header
curl -u alice:s3cret https://httpbin.org/basic-auth/alice/s3cret
#    → {"authenticated": true, "user": "alice"}

# 8. The full timing breakdown (the one-liner from Topic 01)
curl -o /dev/null -s -w "DNS:%{time_namelookup}s TCP:%{time_connect}s TLS:%{time_appconnect}s TTFB:%{time_starttransfer}s Total:%{time_total}s\n" \
  https://httpbin.org/get

# 9. Dump the literal bytes curl sends and receives
curl --trace-ascii - https://httpbin.org/get | head -40
#    "=> Send header" and "<= Recv header" show the raw HTTP bytes

# 10. Prove --compressed asks for gzip and curl auto-decodes it
curl -sv --compressed https://httpbin.org/gzip -o /dev/null 2>&1 | grep -i "accept-encoding\|content-encoding"
```

Expected shape of the timing output (numbers vary with your network):
```
DNS:0.021s TCP:0.058s TLS:0.140s TTFB:0.205s Total:0.206s
```
Read it as cumulative checkpoints from t=0: DNS finished at 21ms, TCP at 58ms, TLS at 140ms, first byte at 205ms. The *phase* costs are the differences: TCP took 58−21=37ms, TLS took 140−58=82ms, server+network took 205−140=65ms.

---

## Practice Exercises

### Exercise 1 — Easy: Read a request off the wire
```bash
# Send a POST with a custom header and a JSON body to httpbin, which echoes it back:
curl -s -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -H "X-Request-Id: ex1" \
  -d '{"hello":"world"}'

# Answer from the JSON that comes back:
# 1. What Content-Type did the server receive? Does it match what you sent?
# 2. Under which key does httpbin show your parsed body — "json", "data", or "form"?
# 3. What User-Agent did curl send by default?
# 4. Add -v. Which lines start with ">" and which with "<"? What does each direction mean?
```

### Exercise 2 — Medium: Make the same call three ways and diff them
```bash
# Send the same key/value pair as (a) form data, (b) JSON, (c) query string.
# (a) form-urlencoded (the -d default):
curl -s https://httpbin.org/anything -d "name=Alice"
# (b) JSON:
curl -s https://httpbin.org/anything --json '{"name":"Alice"}'
# (c) query string via -G:
curl -s https://httpbin.org/anything -G --data-urlencode "name=Alice"

# Answer:
# 1. In which response does "name=Alice" appear under "form", "json", and "args"?
# 2. Which HTTP method did each use? (httpbin echoes "method" — check it)
# 3. Why did (c) become a GET while (a) and (b) were POST?
# 4. Which Content-Type header did each send?
```

### Exercise 3 — Hard (Production Simulation): Isolate a failing, slow, redirecting call
```bash
# Scenario: an endpoint redirects, is slow, and you must find out where the time goes
# AND capture the final status — in ONE command.

# Step 1: measure a redirecting endpoint WITHOUT following, then WITH following:
curl -o /dev/null -s -w "no-L:  code=%{http_code} redirects=%{num_redirects} total=%{time_total}s\n" \
  https://httpbin.org/redirect/2
curl -o /dev/null -s -L -w "with-L: code=%{http_code} redirects=%{num_redirects} total=%{time_total}s\n" \
  https://httpbin.org/redirect/2

# Step 2: build a full timing breakdown for the FINAL response only.
# Hint: with -L, %{time_starttransfer} covers the last hop; num_redirects tells you hop count.
curl -o /dev/null -s -L -w "\
DNS:%{time_namelookup} TCP:%{time_connect} TLS:%{time_appconnect} \
TTFB:%{time_starttransfer} Total:%{time_total} redirects:%{num_redirects}\n" \
  https://httpbin.org/redirect/2

# Questions:
# 1. Why is the http_code different between the -L and no-L runs?
# 2. Each redirect costs roughly how much extra total time? Why?
#    (Think: each hop may need a fresh TCP+TLS unless the connection is reused.)
# 3. If time_appconnect is 0 in your output, what does that tell you about the scheme?
# 4. Rewrite the command so it ALSO fails the shell (non-zero exit) if the final
#    status is 4xx/5xx. (Hint: --fail / -f, and check "echo $?" afterward.)
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **You run `curl -X POST https://api/x` with no `-d`. What body does the server receive, and what does `-X` actually control?**

2. **You send `curl -d '{"a":1}' https://api/x` and the server rejects it as invalid JSON. What Content-Type did curl send, and what are the two ways to fix it?**

3. **What is the difference between `-i` and `-I`? For each, say whether a response body is printed and what HTTP method is sent.**

4. **A colleague's health-check script uses `curl -k https://internal-api/health`. Why is `-k` dangerous, and what should replace it if the failure is an untrusted internal CA?**

5. **In the `-w` timing output, `time_appconnect` is `0.000000`. What does that tell you about the request?** (Hint: what phase does appconnect measure, and when is it skipped?)

6. **You POST to a URL that returns `301`. With `-L`, the request arrives at the target as a GET with no body. Which redirect status codes would have preserved your POST body instead?**

7. **You need to test a new server at `10.0.0.9` while `api.myapp.com` still resolves to the old IP in DNS, and you need the TLS certificate to validate for `api.myapp.com`. Which single flag makes this work, and why is it better than `-k` or editing `/etc/hosts`?**

---

## Quick Reference Card

### Methods & body
| Flag | What it does |
|------|--------------|
| *(none)* | GET — the default method |
| `-X POST/PUT/PATCH/DELETE` | Set the method name (does **not** add a body) |
| `-d '...'` / `--data` | Send a body; implies POST; default type `x-www-form-urlencoded` |
| `--data-raw '...'` | Like `-d` but does **not** treat a leading `@` as a filename |
| `--data-urlencode "k=v"` | URL-encode a field safely |
| `--json '...'` | Send JSON body + set `Content-Type` and `Accept` to `application/json` (7.82+) |
| `-F "field=value"` / `-F "file=@path"` | multipart/form-data (file uploads) |
| `-G` | Move `-d` data into the query string (turns POST into GET) |
| `-d @file` / `-d @-` | Read body from a file / from stdin |

### Headers, auth & cookies
| Flag | What it does |
|------|--------------|
| `-H "Header: value"` | Add/override a request header (repeatable) |
| `-A "agent"` | Set `User-Agent` |
| `-e "url"` | Set `Referer` |
| `-b "name=val"` / `-b file` | Send cookies (inline or from a file) |
| `-c file` | Save received cookies to a file (cookie jar) |
| `-u user:pass` | HTTP Basic auth (builds the `Authorization: Basic` header) |
| `-H "Authorization: Bearer <token>"` | Bearer token auth (manual) |
| `--oauth2-bearer <token>` | Bearer token auth (sets the header for you) |

### Output & diagnostics
| Flag | What it does |
|------|--------------|
| `-o file` / `-O` | Write body to `file` / to the URL's remote filename |
| `-s` | Silent (no progress meter or errors) |
| `-S` | Show errors even with `-s` (use `-sS` together) |
| `-i` | Include response headers **with** the body (still GET) |
| `-I` | Send **HEAD**; headers only, no body |
| `-v` | Verbose: DNS, TCP, TLS, request & response headers |
| `--trace-ascii file` / `--trace file` | Full byte-level dump (ascii / hex) |
| `-f` / `--fail` | Return non-zero exit code on HTTP 4xx/5xx |
| `-w "%{...}"` | Print variables after the transfer (see below) |

### TLS, redirects & protocol
| Flag | What it does |
|------|--------------|
| `-L` | Follow redirects |
| `--max-redirs N` | Cap redirect hops |
| `-k` / `--insecure` | Skip cert verification (**never in prod**) |
| `--cacert file` | Trust a specific CA bundle |
| `--cert file --key file` | Present a client certificate (mTLS) |
| `--tlsv1.3` / `--tlsv1.2` | Force a minimum TLS version |
| `--resolve host:port:ip` | Override DNS for one host (cert still validates) |
| `--connect-to h:p:H:P` | Connect to a different host/port, keep original Host |
| `--http1.1` / `--http2` / `--http3` | Force protocol version |
| `--compressed` | Request gzip/br and auto-decode the response |

### `-w` write-out variables (timing & result)
| Variable | Meaning |
|----------|---------|
| `%{time_namelookup}` | Cumulative time until DNS resolved |
| `%{time_connect}` | Cumulative time until TCP connected |
| `%{time_appconnect}` | Cumulative time until TLS handshake done (0 for http://) |
| `%{time_pretransfer}` | Cumulative time until request is about to be sent |
| `%{time_starttransfer}` | Cumulative time until first response byte (TTFB) |
| `%{time_total}` | Total time for the whole transfer |
| `%{http_code}` | Final HTTP status code |
| `%{size_download}` | Bytes downloaded |
| `%{speed_download}` | Average download speed (bytes/sec) |
| `%{num_redirects}` | Number of redirects followed |
| `%{url_effective}` | Final URL after redirects |
| `%{remote_ip}` | IP curl actually connected to |

**The one-liner every backend dev should memorize:**
```bash
curl -o /dev/null -s -w "\
  DNS: %{time_namelookup}s
  TCP: %{time_connect}s
  TLS: %{time_appconnect}s
 TTFB: %{time_starttransfer}s
Total: %{time_total}s
 Code: %{http_code}\n" https://api.myapp.com/health
```

**Cheat block — the requests you'll type most:**
```bash
curl URL                                   # GET, body to stdout
curl -i URL                                # GET + response headers
curl -I URL                                # HEAD only
curl -v URL                                # full DNS/TCP/TLS/HTTP trace
curl -sS URL                               # quiet, but still show errors
curl -L URL                                # follow redirects
curl --json '{"k":"v"}' URL                # POST JSON the easy way
curl -H "Authorization: Bearer $T" URL     # bearer auth
curl -u user:pass URL                      # basic auth
curl -F "file=@./photo.png" URL            # multipart upload
curl --resolve api.x.com:443:1.2.3.4 URL   # test a server before DNS cutover
curl -o out.json URL                       # save body to a file
```

---

## When Would I Use This at Work?

### Scenario 1: "The API works in Postman but 500s from our service"
Copy the app's real request (headers, body, token) into a curl command and run it from the app's own server (network location matters). If curl also 500s, the server or the request is wrong — bisect headers one at a time until it works. If curl succeeds, your app is sending something different (an extra header, a mangled body, a stale token) — diff the two requests. This is the single most common use of curl on the job.

### Scenario 2: "We're cutting DNS over to a new load balancer tonight"
Before touching DNS, validate the new IP end-to-end with `--resolve host:443:NEW_IP`. You confirm the app responds correctly *and* the TLS certificate is valid for the real hostname — all without editing `/etc/hosts` or disabling cert checks. If health checks pass, you flip DNS knowing the target is good. If they fail, you caught it before customers did.

### Scenario 3: "This endpoint is slow and nobody knows why"
Run the `-w` timing breakdown. In seconds you know whether the cost is DNS, TCP, TLS, server processing (TTFB), or transfer. "It's slow" becomes "TLS handshake is 300ms" or "TTFB is 2s while TCP is 20ms" — each pointing at a totally different team and fix. This is the foundation of the slow-API methodology in Topic 27.

### Scenario 4: "Write a health check / smoke test for CI"
`curl -fsS -o /dev/null -w "%{http_code}\n" https://api/health` gives a clean pass/fail: `-f` makes HTTP errors return non-zero so CI fails correctly, `-sS` stays quiet but still prints real connection errors, and `%{http_code}` logs the status. Drop it in a deploy pipeline or Kubernetes readiness probe.

### Scenario 5: "Reproduce a customer's exact request from a bug report"
Ask the customer (or your logs) for the "Copy as cURL" string from browser DevTools. You now have their exact method, headers, cookies, and body. Replay it, tweak one variable, and find the break — no need to reproduce their entire environment.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the DNS → TCP → TLS → HTTP journey that curl's timing variables measure.
- `10-http1.1-in-depth.md` — the request/response format, methods, and status codes that curl sends and reads.

**Study next (in order):**
- `22-dig-and-dns-debugging.md` — when curl's `time_namelookup` is high or you get `Could not resolve host`, `dig` is where you diagnose the DNS side.
- `27-debugging-a-slow-api.md` — the full methodology built on curl's `-w` timing breakdown: splitting DNS / TCP / TLS / TTFB / transfer and knowing exactly where to look.

**Related tooling in this phase:**
- `23-tcpdump-and-wireshark.md` — when you need to see the raw packets *below* what curl shows at the HTTP layer.
- `24-netstat-and-ss.md` — inspect the TCP connections curl (and your app) actually open.

**This topic connects forward to everything HTTP.** Every API you build, debug, or integrate with will be poked at with curl first.

---

*Doc saved: `/docs/networking/21-curl-mastery.md`*
