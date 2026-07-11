# 13 — WebSockets

> **Phase 2 — Topic 8 of 8**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`

---

## ELI5 — The Simple Analogy

Normal HTTP is like sending letters through the post. Every time you want to know something, you write a letter ("Any new messages?"), mail it, and wait for a reply ("Nope."). If you want to keep checking, you keep mailing letters. It works, but it's slow and wasteful — you send a hundred letters just to eventually get one that says "Yes, here's a new message."

A **WebSocket** is like calling your friend and *keeping the phone line open*. You dial once (the handshake), and after they pick up, the two of you can both talk whenever you want — no re-dialing, no waiting for permission to speak. Either side can blurt out something the instant it happens. The line stays open until one of you hangs up.

The clever part: to get that open phone line, WebSockets *start* as an ordinary HTTP request — the same "letter" the post office already knows how to deliver — and then say "actually, let's turn this into a phone call." That's the **Upgrade handshake**.

---

## Where This Lives in the Network Stack

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP handshake → WebSocket frames)    │  ← WebSocket lives here
│  Layer 6 — Presentation  (TLS/SSL — this is the wss:// part)    │
│  Layer 5 — Session       (the long-lived WS "session")          │
│  Layer 4 — Transport     (TCP — ONE connection, reused forever) │  ← never torn down mid-session
│  Layer 3 — Network       (IP)                                   │
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (fiber, copper, radio)                 │
└─────────────────────────────────────────────────────────────────┘
```

A WebSocket is an **application-layer protocol (Layer 7)** that rides on a **single, persistent TCP connection (Layer 4)**. It begins life as an HTTP/1.1 request, so it inherits HTTP's ports (80/443) and proxy-friendliness. After the handshake, the bytes on that TCP connection are no longer HTTP — they're WebSocket **frames**.

If you use `wss://` (which you always should in production), a TLS layer (Layer 6) sits underneath, exactly like `https://`.

---

## What Is This?

**WebSocket** (RFC 6455) is a protocol that provides **full-duplex, bidirectional communication** over a single long-lived TCP connection. "Full-duplex" means both the client and the server can send data at any time, independently — unlike HTTP's strict request-then-response model where the client must ask before the server may answer.

Three defining properties:

1. **Persistent** — one TCP connection stays open for the life of the session (seconds to hours), instead of one connection per request.
2. **Bidirectional / full-duplex** — the server can push data to the client *without the client asking first*. This is the killer feature.
3. **Low overhead** — after the initial HTTP handshake, each message is wrapped in a tiny frame header (as little as **2 bytes**), versus HTTP's hundreds of bytes of headers per request.

**URL schemes:**
- `ws://host:80/path` — plaintext WebSocket (default port 80)
- `wss://host:443/path` — WebSocket over TLS (default port 443)

Because the handshake is plain HTTP on port 80/443, WebSocket traffic sails through corporate proxies and firewalls that would block a custom protocol on a weird port.

---

## Why Does It Matter for a Backend Developer?

You reach for WebSockets the moment your product needs the server to **tell the client something the client didn't ask for**:

- **Chat / messaging** — a message from Alice must appear on Bob's screen instantly.
- **Live dashboards / trading** — price ticks, metrics, order-book updates streaming in.
- **Collaborative editing** — Google-Docs-style cursors and edits from other users.
- **Multiplayer games** — low-latency position updates in both directions.
- **Notifications / presence** — "user is typing…", "3 people online".

Without WebSockets your only option is to **poll** ("has anything changed? has anything changed?"), which wastes bandwidth, adds latency, and hammers your server with empty requests.

But WebSockets also change how you *operate* your backend. A stateless HTTP request lands on any server behind the load balancer. A WebSocket is **stateful and long-lived** — pinned to one server for its entire life. That single fact ripples into your load balancer config, autoscaling, deploy strategy, and memory budget, and it's why connections mysteriously drop after 60 seconds behind nginx (idle timeouts and missing `Upgrade` headers — see Example 2).

---

## The Packet/Protocol Anatomy

### Part 1 — The handshake (plain HTTP/1.1)

The client sends a normal-looking `GET` with four special headers:

```
GET /chat HTTP/1.1
Host: api.myapp.com
Upgrade: websocket                          ← "please switch protocols"
Connection: Upgrade                         ← "this hop-by-hop header is about upgrading"
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ== ← random 16 bytes, base64-encoded (per connection)
Sec-WebSocket-Version: 13                   ← the RFC 6455 version, always 13
Origin: https://app.myapp.com               ← browser sends this; servers should check it
```

The server, if it agrees, replies with **status 101** and computes an accept token:

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=   ← proves the server understood WS
```

**How `Sec-WebSocket-Accept` is computed** (this is not encryption — it just proves the server isn't a dumb cache blindly echoing headers):

```
accept = base64( SHA1( Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ) )
                                            └────────── the "magic GUID" ──────────┘
  key    = "dGhlIHNhbXBsZSBub25jZQ=="
  concat = "dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
  base64 = "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

After the `101`, **the HTTP conversation is over**. The same TCP socket is now a raw bidirectional pipe carrying WebSocket frames. No more request lines, no more HTTP headers.

### Part 2 — The frame format (RFC 6455)

Every message after the handshake is one or more **frames**. This is the wire format:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+---------------------------------------------------------------+
```

Field by field:

| Field | Bits | Meaning |
|-------|------|---------|
| **FIN** | 1 | `1` = this is the final frame of the message. `0` = more frames follow (fragmentation). |
| **RSV1-3** | 3 | Reserved; `0` unless an extension (e.g. `permessage-deflate` compression) negotiated it. |
| **opcode** | 4 | What kind of frame this is (table below). |
| **MASK** | 1 | `1` = payload is XOR-masked. Client→server: **MUST be 1**. Server→client: **MUST be 0**. |
| **Payload len** | 7 | `0–125` = actual length. `126` = read next 2 bytes as length. `127` = read next 8 bytes. |
| **Masking-key** | 0 or 32 | Present only if MASK=1. 4 random bytes used to XOR the payload. |
| **Payload** | var | The actual data. |

**Opcodes:**

```
0x0  Continuation  — a follow-on fragment of a previous frame
0x1  Text          — UTF-8 text payload
0x2  Binary        — arbitrary binary payload (Blob/ArrayBuffer)
0x8  Close         — connection is closing (optional 2-byte status code + reason)
0x9  Ping          — heartbeat: "are you alive?"
0xA  Pong          — heartbeat reply: "yes, alive"  (also valid unsolicited)
0x3-0x7   reserved for future non-control frames
0xB-0xF   reserved for future control frames
```

Opcodes `0x8`, `0x9`, `0xA` are **control frames**: they must have a payload ≤ 125 bytes and must not be fragmented.

### Part 3 — Masking (why client frames look scrambled)

Every client→server frame XORs its payload with a 4-byte key the client picks at random:

```
transformed[i] = original[i] XOR maskingKey[i % 4]
```

Why? Not for security (the key is sent in the clear). It's an **anti-cache-poisoning** defense: it stops a malicious page from crafting a WebSocket payload that a dumb intermediary proxy would mistake for a valid HTTP request and cache. Servers **MUST** reject an unmasked client frame; clients (browsers) do this automatically.

**Worked example** — the text `"Hello"` sent by a client:

```
Unmasked bytes:   0x48 0x65 0x6c 0x6c 0x6f          ("Hello")
Masking key:      0x37 0xfa 0x21 0x3d

Frame on the wire (11 bytes total):
  81            FIN=1, opcode=0x1 (text)
  85            MASK=1, payload len = 5
  37 fa 21 3d   masking key
  7f 9f 4d 51 58   masked payload  (0x48^0x37=0x7f, 0x65^0xfa=0x9f, ...)
```

The **same "Hello" sent by the server** (no mask) is only 7 bytes:

```
  81            FIN=1, opcode=0x1 (text)
  05            MASK=0, payload len = 5
  48 65 6c 6c 6f   "Hello"
```

---

## How It Works — Step by Step

Tracing a full WebSocket session from `new WebSocket("wss://api.myapp.com/chat")`:

### Step 1 — DNS + TCP + TLS (identical to HTTPS)
```
1a. DNS: api.myapp.com → 142.250.80.46           (Topic 06)
1b. TCP 3-way handshake to port 443              (Topic 08)
1c. TLS 1.3 handshake — encrypts the pipe        (Topic 09)
```
For `wss://`, everything from here on is inside the TLS tunnel.

### Step 2 — The HTTP Upgrade request
```
Client → Server:
  GET /chat HTTP/1.1
  Host: api.myapp.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13
```

### Step 3 — The server's 101 response
```
Server → Client:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

The client verifies Sec-WebSocket-Accept matches its own SHA-1 computation.
If it doesn't match → abort. If it matches → the socket is now a WebSocket.
```

### Step 4 — Full-duplex frame exchange (the whole point)
```
Now EITHER side may send at any time. No request/response coupling.

  Client → Server:  [FIN=1, text, MASK=1] "{"type":"join","room":"general"}"
  Server → Client:  [FIN=1, text, MASK=0] "{"type":"history","messages":[...]}"
  Server → Client:  [FIN=1, text, MASK=0] "{"type":"msg","from":"Bob","text":"hi"}"   ← unprompted!
  Client → Server:  [FIN=1, text, MASK=1] "{"type":"msg","text":"hey Bob"}"
```

### Step 5 — Heartbeat (keeping the pipe alive)
```
Every ~30s, one side sends a Ping; the other MUST reply Pong.

  Server → Client:  [FIN=1, ping, MASK=0]  (empty or small payload)
  Client → Server:  [FIN=1, pong, MASK=1]  (echoes the ping payload)

If no Pong arrives within a timeout → assume the connection is dead, close it.
This also stops idle proxies/load balancers from silently killing the socket.
```

### Step 6 — Graceful close
```
  Either side → other:  [FIN=1, close, code=1000] "normal closure"
  Other side  → back:   [FIN=1, close, code=1000]
  Then the TCP connection is torn down (FIN/ACK, Topic 08).
```

Common close codes: `1000` normal, `1001` going away (page closed/server restart), `1006` abnormal (no close frame — connection just dropped), `1011` server error, `1009` message too big.

---

## Exact Syntax Breakdown

### Browser client
```javascript
const ws = new WebSocket("wss://api.myapp.com/chat");
//         │                └── scheme wss = TLS on port 443 (ws = plaintext, port 80)
//         └── constructor immediately begins the Upgrade handshake

ws.onopen    = ()  => ws.send("hello");        // fired after the 101 response
ws.onmessage = (e) => console.log(e.data);     // fired on every inbound frame
ws.onclose   = (e) => console.log(e.code);     // fired on close frame / drop
ws.onerror   = (e) => console.error(e);        // transport error
ws.send(JSON.stringify({ type: "msg" }));      // send text frame
ws.send(new Uint8Array([1,2,3]).buffer);       // send binary frame
ws.close(1000, "done");                        // send close frame with code + reason
```

### Node.js server with the `ws` library
```javascript
import { WebSocketServer } from "ws";
//       └── the de-facto Node WebSocket library:  npm install ws

const wss = new WebSocketServer({ port: 8080 });
//                                └── ws handles the 101 handshake for you

wss.on("connection", (socket, req) => {
//     │              │       └── the raw HTTP upgrade request (headers, url, ip)
//     │              └── one WebSocket connection (one client)
//     └── fired once per successful handshake

  socket.on("message", (data, isBinary) => { /* inbound frame */ });
  socket.on("pong",    () => { /* client answered our ping */ });
  socket.send("welcome");                       // server→client frame (unmasked)
  socket.ping();                                // send a ping control frame
  socket.close(1000, "bye");                    // send close frame
});
```

---

## Example 1 — Basic

A minimal echo server plus a client, end to end.

**Server (`server.js`):**
```javascript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (socket, req) => {
  console.log("client connected from", req.socket.remoteAddress);

  socket.send("👋 connected — send me anything and I'll echo it");

  socket.on("message", (data, isBinary) => {
    console.log("received:", data.toString());
    socket.send(`echo: ${data}`, { binary: isBinary });
  });

  socket.on("close", (code, reason) => {
    console.log(`closed: ${code} ${reason}`);
  });
});

console.log("WebSocket server listening on ws://localhost:8080");
```

**Client (`client.js`):**
```javascript
import WebSocket from "ws";

const ws = new WebSocket("ws://localhost:8080");

ws.on("open",    () => { ws.send("hello server"); });
ws.on("message", (data) => { console.log("got:", data.toString()); });
ws.on("close",   (code) => { console.log("closed:", code); });
```

**Run it:**
```bash
npm install ws
node server.js &     # start server in background
node client.js       # run client
```

**Output:**
```
# server terminal
WebSocket server listening on ws://localhost:8080
client connected from ::1
received: hello server

# client terminal
got: 👋 connected — send me anything and I'll echo it
got: echo: hello server
```

That's a complete full-duplex round trip: the server pushed a greeting *before the client said anything* (impossible with request/response HTTP), then echoed the client's message.

---

## Example 2 — Production Scenario

**The situation:** You ship a chat app. Locally it works perfectly. In production, behind nginx + an AWS ALB, every connection **dies after exactly 60 seconds**, and the browser console shows `WebSocket closed: 1006` (abnormal — no close frame). Users get logged out of chat constantly.

**Step 1 — Reproduce and observe.** Connect directly, bypassing nginx:
```bash
# Talk to the Node app directly on its internal port — works fine, stays open
npx wscat -c ws://10.0.1.20:8080
# Connected (press CTRL+C to quit)
# ... still alive after 5 minutes ✓
```
Now through the public nginx endpoint:
```bash
npx wscat -c wss://chat.myapp.com/ws
# Connected (press CTRL+C to quit)
# Disconnected (code: 1006) ← dies at ~60s
```
The app is fine. The **proxy** is the culprit. Two classic bugs, often both at once.

**Bug A — nginx isn't forwarding the Upgrade headers.** A default `proxy_pass` block strips the hop-by-hop `Upgrade`/`Connection` headers, so nginx never even negotiates a WebSocket — it treats it as a plain HTTP request that hangs. Symptom: the handshake returns `200` or `400` instead of `101`.

Broken config:
```nginx
location /ws {
    proxy_pass http://chat_backend;   # ← WebSocket handshake never completes
}
```
Fixed config:
```nginx
location /ws {
    proxy_pass http://chat_backend;
    proxy_http_version 1.1;                        # keep-alive required for upgrade
    proxy_set_header Upgrade    $http_upgrade;      # forward "Upgrade: websocket"
    proxy_set_header Connection "upgrade";          # forward "Connection: Upgrade"
    proxy_set_header Host       $host;
    proxy_read_timeout  3600s;                      # ← Bug B fix (see below)
    proxy_send_timeout  3600s;
}
```
Verify the handshake now returns `101`:
```bash
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://chat.myapp.com/ws
#
# HTTP/1.1 101 Switching Protocols        ← success!
# Upgrade: websocket
# Connection: upgrade
# Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**Bug B — the idle timeout kills silent connections.** nginx's default `proxy_read_timeout` is **60 seconds**. If no bytes flow for 60s, nginx closes the upstream connection → the `1006` you saw. The ALB has a similar default (60s idle timeout). Raising the timeout helps, but the *correct* fix is an **application-level heartbeat** so the connection is never truly idle:

```javascript
import { WebSocketServer } from "ws";
const wss = new WebSocketServer({ port: 8080 });

function heartbeat() { this.isAlive = true; }

wss.on("connection", (socket) => {
  socket.isAlive = true;
  socket.on("pong", heartbeat);   // client answered our ping → still alive
});

// Every 30s: drop the dead, ping the living
const interval = setInterval(() => {
  wss.clients.forEach((socket) => {
    if (socket.isAlive === false) return socket.terminate();  // no pong last round
    socket.isAlive = false;
    socket.ping();               // sends a 0x9 control frame; browser auto-replies pong
  });
}, 30000);

wss.on("close", () => clearInterval(interval));
```

**Result:** a ping every 30s keeps traffic flowing (defeats idle timeouts on nginx, the ALB, and any corporate proxy), and the `isAlive` check reaps connections whose TCP silently died (laptop closed, wifi dropped) instead of leaking them forever. Connections now survive indefinitely.

**The lesson:** WebSockets *look* like HTTP to your app but behave like a phone call to your infrastructure. Every proxy, load balancer, and firewall between client and server must be told to (a) pass the Upgrade handshake through and (b) tolerate a long-idle connection — and you keep it non-idle with heartbeats.

---

## Common Mistakes

### Mistake 1: No heartbeat — connections die and you don't notice
```
Symptom: connections drop after 30–120s in production; browser sees code 1006.
```
**Root cause:** Proxies, load balancers, and NAT gateways kill idle TCP connections. Without ping/pong, an idle WebSocket looks dead and gets reaped. Worse, a *half-open* connection (client's laptop slept) stays in your server's memory forever because TCP never told you it died.
**Fix:** Server pings every ~30s and terminates any socket that didn't pong (see Example 2). Never rely on `onclose` firing — a dropped connection may never send a close frame.

---

### Mistake 2: Proxy not configured to pass Upgrade/Connection headers
```nginx
# WRONG — handshake returns 200, never upgrades:
location /ws { proxy_pass http://backend; }
```
**Root cause:** `Upgrade` and `Connection` are *hop-by-hop* headers; a proxy strips them unless explicitly told to forward them, and must use HTTP/1.1.
**Fix:**
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade    $http_upgrade;
proxy_set_header Connection "upgrade";
```

---

### Mistake 3: Assuming WebSockets auto-reconnect
```javascript
// WRONG — one network blip and the client is offline forever
const ws = new WebSocket(url);
ws.onmessage = handle;
```
**Root cause:** The WebSocket API does **not** reconnect. When the connection drops (wifi change, server restart, deploy), it's gone until *you* create a new one.
**Fix:** Implement reconnection with exponential backoff, and re-sync missed state on reconnect:
```javascript
let backoff = 1000;
function connect() {
  const ws = new WebSocket(url);
  ws.onopen    = () => { backoff = 1000; };
  ws.onmessage = handle;
  ws.onclose   = () => {
    setTimeout(connect, backoff);
    backoff = Math.min(backoff * 2, 30000);   // cap at 30s
  };
}
connect();
```

---

### Mistake 4: Using WebSockets where SSE or polling would do
```
Building a stock ticker that only streams server→client?  → use SSE, not WebSockets.
Checking a job status every 10s?                          → use polling, not WebSockets.
```
**Root cause:** WebSockets are stateful, harder to scale, harder to load-balance, and bypass HTTP caching/auth middleware. If you only need **server→client** streaming, **Server-Sent Events (SSE)** is simpler, auto-reconnects for free, and works over plain HTTP.
**Fix:** Pick the lightest tool that meets the requirement — see the comparison table below.

---

### Mistake 5: Ignoring sticky sessions when scaling horizontally
```
2 app servers behind a round-robin LB. Alice's socket lives on server A.
Bob's message arrives on server B. Bob's server has no idea Alice exists → message lost.
```
**Root cause:** A WebSocket is pinned to one server, but a horizontally-scaled fleet has N servers each holding a different subset of connections.
**Fix:** Two independent things: (1) **sticky sessions** (or IP hash) so a client's handshake and its connection land on the same server; (2) a **pub/sub backplane** (Redis, NATS, Kafka) so any server can broadcast to sockets held by *other* servers. Sticky sessions alone don't solve cross-server messaging.

---

## Hands-On Proof

```bash
# 1. Watch the raw Upgrade handshake with curl (works because handshake IS http)
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://echo.websocket.org/
# Look for: "HTTP/1.1 101 Switching Protocols" and the Sec-WebSocket-Accept header

# 2. Verify the accept-token math yourself (should print s3pPLMBiTxaQ9kYGzzhZRbK+xOo=)
printf '%s' 'dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11' \
  | openssl sha1 -binary | base64

# 3. Interact with a live WebSocket from the terminal
npx wscat -c wss://echo.websocket.org
# type a message, press Enter — it echoes back. Ctrl+C to close.

# 4. Capture the frames on the wire (plaintext ws:// only — wss is encrypted)
sudo tcpdump -i any -A -s0 'tcp port 8080'
# run your Example 1 server/client and watch the GET / 101 / frame bytes fly by

# 5. Inspect in the browser: DevTools → Network → click the WS request →
#    "Messages" tab shows every frame, direction, opcode, and payload live.
```

---

## Practice Exercises

### Exercise 1 — Easy: Prove the handshake is just HTTP
```bash
# Use curl to complete a real WebSocket handshake against a public echo server.
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" \
  https://echo.websocket.org/

# Answer:
# 1. What HTTP status code comes back? Why is it not 200?
# 2. Copy the Sec-WebSocket-Accept value. Now recompute it yourself with openssl
#    (hint: Hands-On Proof #2). Do they match?
# 3. Why can't curl actually *use* the connection after the 101?
```

### Exercise 2 — Medium: Build a broadcast chat
```javascript
// Extend the Example 1 server so that when ANY client sends a message,
// EVERY connected client receives it (a group chat).
// Hint: iterate wss.clients and send to each socket whose readyState === OPEN.

wss.on("connection", (socket) => {
  socket.on("message", (data) => {
    // TODO: broadcast `data` to all clients
  });
});

// Test it: open 3 wscat clients, send from one, confirm the other two receive it.
// Then answer: what happens if you DON'T check readyState before sending?
```

### Exercise 3 — Hard: Make it survive two servers
```
Run TWO copies of your chat server (ports 8080 and 8081) behind a load balancer.
A message sent to a client on 8080 must reach a client on 8081.

1. Add a Redis pub/sub backplane:
   - On "message": PUBLISH the payload to a Redis channel "chat".
   - Each server SUBSCRIBEs to "chat" and broadcasts received payloads to its
     own local clients.
2. Configure the load balancer with sticky sessions (ip_hash in nginx).

Answer:
- Why do you need BOTH the backplane AND sticky sessions? What breaks if you
  drop each one?
- When you deploy a new version and restart both servers, what code 1001 vs 1006
  will clients see, and how does your Exercise-2/reconnect logic handle it?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A WebSocket connection begins as what kind of request, and what exact HTTP status code signals success?** Why is starting as HTTP a practical advantage?

2. **What are the four request headers the client must send in the handshake, and how does the server derive `Sec-WebSocket-Accept`?** Is that value encryption?

3. **In the frame format, what does the FIN bit mean, and what do opcodes `0x1`, `0x2`, `0x8`, `0x9`, `0xA` represent?**

4. **Client→server frames MUST be masked; server→client frames MUST NOT be. Why does masking exist — what attack does it prevent?**

5. **Your WebSocket connections all die after ~60 seconds in production but work locally. Name the two most likely causes and the fix for each.**

6. **You have 3 chat servers behind a load balancer. A user on server A sends a message meant for a user on server C. What two pieces of infrastructure do you need, and what does each one solve?**

7. **You need to stream live notifications from server to browser, one direction only. Should you use WebSockets, SSE, or long-polling — and why?**

---

## Quick Reference Card

| Concept | Detail |
|---------|--------|
| Protocol / RFC | WebSocket, RFC 6455 |
| URL schemes | `ws://` (port 80), `wss://` (port 443, TLS) |
| Handshake | HTTP/1.1 `GET` + `Upgrade: websocket` → `101 Switching Protocols` |
| Magic GUID | `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` |
| Accept token | `base64(SHA1(key + GUID))` |
| Min frame overhead | 2 bytes (server→client) / 6 bytes (client→server, incl. mask) |
| Masking | client→server MUST mask; server→client MUST NOT |
| Heartbeat | ping `0x9` / pong `0xA`, every ~30s |
| Version header | always `Sec-WebSocket-Version: 13` |

**Opcodes:**
```
0x0 continuation   0x1 text   0x2 binary
0x8 close          0x9 ping   0xA pong
```

**Close codes:** `1000` normal · `1001` going away · `1006` abnormal (no close frame) · `1009` too big · `1011` server error

**WebSockets vs Long-Polling vs SSE:**

| | WebSockets | Long-Polling | Server-Sent Events (SSE) |
|--|-----------|--------------|--------------------------|
| Direction | Full-duplex (both ways) | Client→server (server replies late) | Server→client only |
| Transport | 1 persistent TCP, WS frames | Repeated HTTP requests | 1 persistent HTTP response stream |
| Reconnect | Manual (you code it) | Automatic (new request each time) | Automatic (built into `EventSource`) |
| Browser API | `WebSocket` | `fetch`/`XHR` in a loop | `EventSource` |
| Data type | Text + binary | Text + binary | Text only (UTF-8) |
| Overhead per msg | ~2–6 bytes | Full HTTP headers each time | Small (`data:` line) |
| Best for | Chat, games, collab, trading | Legacy/fallback where WS blocked | Notifications, feeds, live logs, dashboards |

**nginx WebSocket proxy block (memorize this):**
```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host       $host;
    proxy_read_timeout 3600s;
}
```

**Node.js `ws` cheat block:**
```javascript
import { WebSocketServer } from "ws";
const wss = new WebSocketServer({ port: 8080 });
wss.on("connection", (s) => {
  s.send("hi");                       // server→client (unmasked)
  s.on("message", (d) => s.send(d));  // echo
  s.on("pong", () => { s.isAlive = true; });
  s.ping();                           // heartbeat
});
```

---

## When Would I Use This at Work?

### Scenario 1: "Add live notifications to our dashboard"
First ask: **which direction does data flow?** If it's purely server→client (a bell icon lighting up, a feed updating), reach for **SSE** — it's one HTTP endpoint, auto-reconnects, and passes through your existing auth middleware untouched. Only escalate to WebSockets if the client also needs to push (typing indicators, acks).

### Scenario 2: "Chat/presence works locally but users get disconnected in prod"
You immediately suspect the proxy layer. Check three things: (1) does the handshake return `101` through the LB (`curl -i` with Upgrade headers)? (2) is nginx passing `Upgrade`/`Connection` and running `proxy_http_version 1.1`? (3) is there an idle timeout shorter than your heartbeat interval? This is the single most common WebSocket production incident — see Example 2 and Topics `28-load-balancers.md` / `29-reverse-proxies.md`.

### Scenario 3: "Our real-time service can't scale past one box"
You now know WebSockets are stateful and pinned. To go horizontal you need **sticky sessions** (so the handshake and its long-lived socket stay on one server) *plus* a **Redis/NATS pub/sub backplane** (so a message arriving on server B can reach a socket held on server A). You also budget memory per connection (each idle socket costs kilobytes of kernel + app buffers), so 100k concurrent connections is a real capacity-planning number, not an afterthought.

### Scenario 4: "Collaborative editor feels laggy"
Round-trip latency dominates. A WebSocket removes per-message TCP/TLS/HTTP-header overhead, so each keystroke or cursor move is a ~2-byte frame on an already-open pipe — the lowest-latency option the browser offers short of WebRTC.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the full request lifecycle WebSockets build on
- `08-tcp-in-depth.md` — WebSockets ride one long-lived TCP connection
- `09-tls-ssl-in-depth.md` — `wss://` is WebSocket over exactly this
- `10-http1.1-in-depth.md` — the handshake IS an HTTP/1.1 request; `Upgrade` is an HTTP mechanism

**Study next:**
- `11-http2.md` / `12-http3-and-quic.md` — how multiplexing changes the streaming picture
- `28-load-balancers.md` — L7 LBs, sticky sessions, and why WebSockets need them
- `29-reverse-proxies.md` — the nginx `Upgrade` header pass-through that Example 2 fixes

**This topic connects forward to:** any real-time feature — chat, notifications, live dashboards, multiplayer, collaborative editing — and to the scaling patterns (pub/sub backplanes, sticky routing) that make them work in production.

---

*Doc saved: `/docs/networking/13-websockets.md`*
