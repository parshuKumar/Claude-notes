# 82 — Protocols — HTTP/2, HTTP/3, WebSockets, SSE, Long Polling
## Category: HLD Components

---

## What is this?

Plain HTTP is a **vending machine**: it only ever does something when *you* put a coin in. The client asks, the server answers, the conversation ends. The machine can never tap you on the shoulder and say "hey, a fresh snack just arrived."

But almost every interesting product needs exactly that shoulder-tap. A new chat message arrives. Your Uber driver moves 40 metres. A build finishes. An LLM produces the next token. This doc is about the one question that unifies all of them: **the server has new data — how does it reach the client?** There are exactly four practical answers (short polling, long polling, SSE, WebSockets), and separately there is the question of *what transport they ride on* (HTTP/1.1, HTTP/2, HTTP/3). We cover both.

---

## Why does it matter?

**If you don't understand this**, you will reach for the wrong tool and pay for it forever:

- You'll build a notification feed on a 2-second polling loop, and when you hit 100k users your API tier is drowning in requests that return `[]`.
- You'll reach for WebSockets because it sounds "real-time and modern", then discover six months later that your load balancer can't route reconnects properly, your deploys drop every live connection, and you can't scale past one server without a message bus you didn't plan for.
- You'll ship SSE and then be baffled that only 6 tabs work on HTTP/1.1.

**Interview angle:** "Design WhatsApp / Uber / a stock ticker / a live scoreboard" is a *push* question in disguise. The first real fork in the road is which push mechanism you pick, and the interviewer wants you to **justify** it, not just name it. The follow-up is always the same and it is the hard part: *"okay, you have 10 million WebSocket connections across 500 servers — user A is on server 3 and user B is on server 217, how does A's message reach B?"* If you can draw that, you're ahead of most candidates.

**Real-work angle:** Every gateway, every proxy, every load balancer config, every mobile client you ever write will force you to make these choices explicitly.

---

## The core idea — explained simply

### The Pizza Delivery Analogy

You ordered a pizza. You want to know the moment it arrives. There are four ways to find out.

**1. Short polling — you walk to the front door every 30 seconds and look outside.**
Most of the time: no pizza. You've wasted 40 trips. And in the worst case the pizza sat on your doorstep for 29 seconds getting cold before you noticed. Simple, dumb, works with any door.

**2. Long polling — you open the door and stand there, holding it open, staring down the street.**
The moment the delivery guy appears, you grab the pizza and shut the door — then *immediately* open it again to wait for the next one. You get the pizza the instant it arrives. But you're standing in the doorway the entire time, letting the cold in, and if you stand there too long your landlord (the proxy) makes you close it.

**3. Server-Sent Events — you give the delivery guy a walkie-talkie that only he can talk on.**
He announces "leaving the shop", "two streets away", "at your door" and you just listen. One-way: you can't talk back on that channel. If the walkie-talkie battery dies it automatically reconnects and he resumes from the last thing you heard. Perfect for a *stream of updates*.

**4. WebSockets — you install a permanent open intercom line between your kitchen and the pizza shop.**
Either side can talk at any moment. You can shout "add extra cheese" mid-bake, he can shout "we're out of olives". Zero ceremony per message. But that line has to be *kept alive*, it costs the shop a dedicated handset per customer, and if the shop moves premises (server restart) everyone's line goes dead at once.

**Mapping it back:**

| In the analogy | In the system | Direction | Cost |
|---|---|---|---|
| Walking to the door every 30s | Short polling (`setInterval` + `fetch`) | Client pulls | Wasted requests, up to N seconds of latency |
| Standing in the open doorway | Long polling (server holds the request) | Client pulls, server stalls | One held connection per client |
| One-way walkie-talkie | SSE (`text/event-stream`) | Server → client only | One held connection, but cheap and native |
| Permanent intercom | WebSocket (`ws://` after HTTP Upgrade) | Both ways, any time | Stateful connection, hard to scale |

And the **transport version** (HTTP/1.1 vs /2 vs /3) is the *road* the delivery guy drives on: a single-lane street where one stalled car blocks everybody (HTTP/1.1), a multi-lane highway that still shares one bridge (HTTP/2), or independent lanes that never block each other (HTTP/3).

---

## Key concepts inside this topic

### 1. Short polling — the honest, terrible baseline

The client just asks, on a timer.

```js
// Browser (or any client). This is the entire technique.
setInterval(async () => {
  const res = await fetch('/api/messages?since=' + lastSeenId);
  const messages = await res.json();
  if (messages.length) render(messages);   // 99% of the time: length === 0
}, 2000);
```

**Do the arithmetic — this is what interviewers want to see.**

```
10,000 concurrent clients
Poll interval: every 2 seconds
  → each client makes 0.5 requests/second
  → 10,000 × 0.5 = 5,000 requests/second

How many actually carry data? If a user receives ~1 message per minute:
  → useful responses = 10,000 / 60 ≈ 167 per second
  → 5,000 - 167 = 4,833 req/s return an EMPTY array

  Waste ratio ≈ 96.7%
```

You are paying full price — TCP + TLS + HTTP headers + auth middleware + a DB query — 4,833 times a second to say "nothing new." And the user *still* waits up to 2 seconds to see a message.

Make the interval shorter and the waste multiplies. Make it longer and latency gets worse. There is no good setting. That's the whole problem.

**When it is genuinely fine:** low client count, updates that are naturally infrequent and not urgent (a dashboard refreshing every 30s, a CI status page, checking for app updates once an hour). Do not be ashamed of polling when the numbers are small — it is the only option that works through *every* proxy, firewall, and corporate network on earth.

### 2. Long polling — near-real-time with plain HTTP

The trick: the server doesn't respond immediately. It **holds the request open** until it has something to say, or until a timeout fires. The client, on receiving the response, immediately opens a new request.

```js
// Node.js / Express — the server side of long polling
const waiting = new Map();  // userId -> [express response objects]

app.get('/api/poll', (req, res) => {
  const { userId } = req.auth;

  // Any messages queued while the client was reconnecting? Answer instantly.
  const pending = queue.drain(userId);
  if (pending.length) return res.json(pending);

  // Otherwise: DO NOT respond. Park the response object and walk away.
  const list = waiting.get(userId) ?? [];
  list.push(res);
  waiting.set(userId, list);

  // Safety valve: browsers, proxies and load balancers all kill idle requests
  // eventually. Respond with an empty result BEFORE they do, so the client
  // reconnects on our terms instead of seeing a network error.
  const timer = setTimeout(() => {
    release(userId, res, []);
  }, 25_000);

  res.on('close', () => clearTimeout(timer));  // client vanished; clean up
});

// Called whenever a message is produced for this user
function deliver(userId, message) {
  const list = waiting.get(userId);
  if (!list?.length) return queue.push(userId, message);  // nobody listening
  for (const res of list.splice(0)) res.json([message]);  // wake them all up
}
```

```js
// Client: respond, then IMMEDIATELY re-ask. That's the loop.
async function longPoll() {
  while (true) {
    try {
      const res = await fetch('/api/poll');
      const messages = await res.json();
      if (messages.length) render(messages);
    } catch {
      await sleep(1000 + Math.random() * 2000);  // backoff + JITTER, see below
    }
  }
}
```

**What it costs you:**
- **A held connection per client.** 10,000 users = 10,000 open sockets and 10,000 parked `res` objects in memory. Node handles this fine (it's event-driven); a thread-per-request server like classic Apache/Tomcat does not.
- **Timeouts everywhere.** Your reverse proxy (nginx `proxy_read_timeout`, default 60s), your ALB (idle timeout, default 60s), and the browser all want to kill a "hanging" request. You must set your server timeout *below* all of theirs.
- **The reconnect storm.** This is the killer. If you restart the server, all 10,000 clients get a dropped connection *at the same instant* and all reconnect *at the same instant* — a thundering herd that kills the process you just started. **Fix: exponential backoff with jitter** — a random delay so clients spread out instead of arriving in a synchronised wave. Note this in an interview; it earns real credit.

Long polling is how Facebook Chat and Gmail chat worked before WebSockets existed. It is still the correct fallback for clients stuck behind hostile proxies.

### 3. Server-Sent Events (SSE) — the underrated one

SSE is one HTTP response that **never ends**. The server sets `Content-Type: text/event-stream`, keeps the socket open, and writes newline-delimited events into it forever. The browser has native support via `EventSource`.

Server:

```js
// Node.js / Express. This is the whole server.
app.get('/api/events', (req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',  // the magic MIME type
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no',  // tell nginx NOT to buffer, or nothing streams
  });

  const send = (event) => {
    // The wire format: an "id:" line (for resume), an optional "event:" name,
    // a "data:" line, then a BLANK LINE which terminates the event.
    res.write(`id: ${event.id}\n`);
    res.write(`event: ${event.type}\n`);
    res.write(`data: ${JSON.stringify(event.payload)}\n\n`);
  };

  // The client tells us where it left off after a dropped connection.
  const lastId = req.headers['last-event-id'];
  if (lastId) for (const e of store.since(lastId)) send(e);

  bus.on(`user:${req.auth.userId}`, send);

  // Heartbeat: a comment line. Keeps proxies from killing an "idle" stream.
  const hb = setInterval(() => res.write(': ping\n\n'), 15_000);

  req.on('close', () => {              // client navigated away / lost network
    clearInterval(hb);
    bus.off(`user:${req.auth.userId}`, send);
  });
});
```

Client — this is the part people don't believe:

```js
const es = new EventSource('/api/events');

es.addEventListener('message-received', (e) => render(JSON.parse(e.data)));
es.addEventListener('typing', (e) => showTyping(JSON.parse(e.data)));

es.onerror = () => { /* do nothing — EventSource reconnects BY ITSELF */ };
```

That's it. **Roughly 15 lines of server, 4 lines of client**, and you have server push with:
- **Automatic reconnection** — built into the browser, with a server-tunable retry interval (`retry: 3000\n\n`).
- **Resume via event IDs** — on reconnect the browser automatically sends `Last-Event-ID`, so you can replay what was missed. You get at-least-once delivery almost for free.
- **Plain HTTP** — passes through every proxy, works with your existing auth cookies, compresses, and is trivially debuggable with `curl -N`.

**Where SSE is the right answer:** notifications, live feeds, dashboards/metrics, progress bars for long jobs, build logs, stock tickers, and **LLM token streaming** (yes — that's what ChatGPT-style token-by-token output uses; it is a plain SSE stream).

**Its two real limitations:**
1. **One-directional.** Server → client only. If the client also needs to send, it just... makes a normal POST. That's usually completely fine, and the combination "SSE down + POST up" is a legitimate, boringly reliable architecture.
2. **The 6-connection limit on HTTP/1.1.** Browsers allow only ~6 concurrent connections *per domain*. An SSE stream consumes one of them permanently — so open the app in 6 tabs and the 7th tab hangs forever. **Over HTTP/2 this evaporates** (all streams multiplex over one connection, limit is ~100). If you serve SSE, serve it over HTTP/2.

### 4. WebSockets — real bidirectional, and really stateful

A WebSocket starts life as an ordinary HTTP request carrying an `Upgrade` header:

```
GET /ws HTTP/1.1
Host: chat.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

The server answers `101 Switching Protocols`, and from that instant the TCP connection **stops speaking HTTP**. It now speaks a compact binary framing protocol: a couple of bytes of header, then your payload. There are no cookies, no headers, no method line on each message. Sending "hi" costs ~2 bytes of overhead instead of ~700 bytes of HTTP headers. That is why it wins for high-frequency messaging.

```js
// Node.js with the `ws` library
import { WebSocketServer } from 'ws';
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (socket, req) => {
  const userId = authenticate(req);           // validate the token from the URL/cookie
  socket.userId = userId;
  socket.isAlive = true;
  socket.on('pong', () => { socket.isAlive = true; });   // heartbeat reply

  socket.on('message', (raw) => {
    const msg = JSON.parse(raw);
    if (msg.type === 'send') routeToRecipient(msg.to, {
      type: 'message', from: userId, body: msg.body, ts: Date.now(),
    });
  });

  socket.on('close', () => registry.remove(userId, socket));
  registry.add(userId, socket);
});

// HEARTBEAT — you MUST write this yourself. A TCP connection whose peer
// vanished (laptop lid closed, phone lost signal) can stay "open" for
// *hours* from the server's point of view. Without this you leak sockets.
setInterval(() => {
  for (const socket of wss.clients) {
    if (!socket.isAlive) return socket.terminate();  // missed last ping → dead
    socket.isAlive = false;
    socket.ping();
  }
}, 30_000);
```

```js
// Browser — note that reconnection is 100% your problem.
function connect() {
  const ws = new WebSocket('wss://chat.example.com/ws');
  let backoff = 500;

  ws.onopen = () => { backoff = 500; };
  ws.onmessage = (e) => render(JSON.parse(e.data));
  ws.onclose = () => {
    // Exponential backoff, capped, WITH jitter so 10k clients don't
    // all reconnect on the same millisecond after a deploy.
    const wait = Math.min(backoff *= 2, 30_000) * (0.5 + Math.random());
    setTimeout(connect, wait);
  };
}
connect();
```

**The costs, honestly:**
- **It is stateful.** The connection *lives on one specific server process*. A load balancer must use sticky routing, and every deploy severs every connection.
- **No automatic reconnect.** Unlike SSE, you write it. Everyone gets this wrong the first time.
- **No built-in liveness.** You write the ping/pong heartbeat. Everyone forgets this the first time and then wonders why memory grows all week.
- **Proxies and firewalls.** Corporate networks and some mobile carriers still block or mangle the Upgrade. Production systems ship a fallback (Socket.IO's whole reason for existing is that it silently degrades to long polling).
- **Scaling requires a backplane.** See below — it's the interview question.

### 5. The WebSocket scaling problem (the key insight)

One server, no problem: everyone's socket is in the same process, so you look up the recipient in a local `Map` and write to their socket.

Now put a load balancer in front and run five servers. Alice's socket is on **Server A**. Bob's socket is on **Server B**. Alice sends a message for Bob. Server A looks in its local map — **Bob isn't there.** The message is dropped.

The fix is a **pub/sub backplane** (Redis pub/sub, NATS, or Kafka). Every server subscribes to the channels for the users it currently holds. When Server A needs to reach Bob, it doesn't try to find Bob — it just **publishes to `user:bob`** and lets the backplane fan it out. Server B, which is subscribed to `user:bob`, receives it and writes to Bob's local socket.

```js
// Every gateway server runs this. Redis makes location a non-problem.
import { createClient } from 'redis';
const pub = createClient(), sub = createClient();
await pub.connect(); await sub.connect();

// When a user connects HERE, subscribe to their channel HERE.
async function onConnect(userId, socket) {
  local.set(userId, socket);
  await sub.subscribe(`user:${userId}`, (raw) => {
    // Fired no matter WHICH server originally published it.
    socket.send(raw);
  });
}
async function onDisconnect(userId) {
  local.delete(userId);
  await sub.unsubscribe(`user:${userId}`);
}

// Sending is now location-independent: we never ask "where is Bob?"
async function routeToRecipient(userId, message) {
  await pub.publish(`user:${userId}`, JSON.stringify(message));
}
```

The mental unlock: **stop trying to route to a server; route to a topic.** Sticky sessions keep a *reconnecting* client on a server that knows it, and the backplane makes it irrelevant which server that is.

### 6. The decision table (memorise this)

| Your need | Use | Why |
|---|---|---|
| Client asks, server answers, done | **Plain HTTP** | Don't overthink it. 95% of endpoints. |
| One-way, infrequent, small client count | **Short polling** | Trivial, works everywhere, costs nothing to build |
| One-way, need near-real-time, WebSockets blocked | **Long polling** | Real-time-ish over vanilla HTTP; the universal fallback |
| One-way server → client stream (feeds, notifications, progress, LLM tokens) | **SSE** | Native, auto-reconnect, resumable, dead simple. **Underused.** |
| Bidirectional and high-frequency (chat, games, collab editing, trading) | **WebSockets** | Full-duplex, tiny per-message overhead |
| Peer-to-peer audio/video/screen share | **WebRTC** | Media goes browser↔browser, not through your server |
| Service-to-service RPC with streaming | **gRPC** (rides HTTP/2) | Typed contracts, streams come free from HTTP/2 |

**The mistake to avoid:** reaching for WebSockets when the data only flows one way. You take on statefulness, sticky sessions, heartbeats, and reconnect logic — and get nothing in return that SSE wouldn't have given you for free.

### 7. HTTP/1.1 → HTTP/2 → HTTP/3 — what actually changed

**HTTP/1.1 (1997).** One request at a time, per connection. Request #2 cannot start until response #1 has fully arrived. That's **head-of-line (HOL) blocking**: one slow response stalls everything behind it. Browsers hacked around this by opening ~6 parallel TCP connections per domain — hence the classic 2010s performance hacks:
- **Sprite sheets** — glue 30 icons into one image so it's 1 request, not 30.
- **File concatenation** — bundle all JS into one giant file.
- **Domain sharding** — serve assets from `img1.example.com`, `img2.example.com` to get 6 connections *each*.

**HTTP/2 (2015).** The headline feature is **multiplexing**: many independent *streams* interleaved over **one** TCP connection. Requests no longer queue behind each other at the HTTP layer. Consequences:
- The old hacks became **unnecessary** — and **domain sharding became actively harmful**, because it forces extra TCP+TLS handshakes and splits your one nicely-multiplexed connection into several worse ones. If you see sharding in a modern codebase, delete it.
- **HPACK header compression** — HTTP headers are hugely repetitive (same cookies, same user-agent on every request). HPACK keeps a shared table of previously-seen headers and sends indices instead. Big win on request-heavy pages.
- **Server Push** — the server could pre-emptively send you `style.css` before you asked. Be honest in interviews: **it failed and has been removed.** Chrome disabled it in 2022. It pushed things clients already had cached, wasting bandwidth. `103 Early Hints` replaced it.
- **The remaining flaw:** it fixed HOL blocking at the *HTTP* layer but not at the *TCP* layer. TCP guarantees in-order delivery of the whole byte stream, so if a single packet is lost, **every** multiplexed stream on that connection stalls until it's retransmitted — even streams whose data arrived perfectly fine. On a lossy mobile network, HTTP/2 can be *worse* than HTTP/1.1's six independent connections.

**HTTP/3 (2022).** Throws away TCP and runs over **QUIC on top of UDP**. This is the whole point: QUIC implements streams itself, so **streams are genuinely independent**. A lost packet stalls only its own stream; the others keep flowing. TCP-level HOL blocking is *structurally* gone. Plus two more gifts:
- **Faster setup.** TLS is folded into the transport handshake — 1 round trip for a new connection (vs TCP's SYN/SYN-ACK *then* TLS's 2 more), and **0-RTT** for a resumed one: you send application data in the very first packet.
- **Connection migration.** A TCP connection is identified by the 4-tuple `(src IP, src port, dst IP, dst port)` — change your IP and it dies. QUIC identifies a connection by an opaque **Connection ID**, so when your phone hops from WiFi to cellular, the connection *survives*. Your download doesn't restart. Your video call doesn't drop. On mobile this is enormous.

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| **Transport** | TCP | TCP | **QUIC over UDP** |
| **Concurrency** | ~6 TCP connections | Multiplexed streams, 1 connection | Multiplexed streams, 1 connection |
| **HTTP-level HOL blocking** | Yes | No | No |
| **TCP-level HOL blocking** | Yes (per connection) | **Yes — one lost packet stalls all streams** | **No — streams are independent** |
| **Header compression** | None | HPACK | QPACK |
| **New-connection setup** | TCP (1 RTT) + TLS (2 RTT) | Same | **1 RTT; 0-RTT on resume** |
| **Survives WiFi → cellular** | No | No | **Yes (Connection ID)** |
| **Server push** | — | Yes, but **deprecated/removed** | Not a thing |
| **Header format** | Text | Binary | Binary |

**Practical takeaway:** you don't rewrite your app for HTTP/2 or HTTP/3. You enable them at the *edge* (CDN / load balancer / nginx) and the browser negotiates automatically via ALPN. Your job is to (a) turn them on, and (b) **remove the HTTP/1.1-era hacks**, which are now counterproductive.

### 8. The two neighbours worth naming

- **gRPC rides on HTTP/2.** That is not a coincidence — it's *why* gRPC gets bidirectional streaming, multiplexing, and binary framing for free. gRPC didn't invent streaming; it inherited it. (The reason gRPC is awkward from browsers is precisely that browsers don't expose raw HTTP/2 frames — hence gRPC-Web and a proxy.)
- **WebRTC is for peer-to-peer media.** Video calls, screen shares, game voice. The audio/video flows **directly between browsers** (via UDP, with STUN/TURN servers only to punch through NAT), so your servers never touch the media. Your server is only the *matchmaker* — and the signalling channel it uses to introduce the two peers is very often... a WebSocket.

---

## Visual / Diagram description

**Diagram 1 — the four push strategies on a timeline.** Time runs left to right. `REQ` is a request, `RES` a response, `▓` is a held-open connection.

```
SHORT POLLING (interval = 2s)              latency: up to 2s, ~97% waste
Client  REQ────RES(empty)  REQ────RES(empty)  REQ────RES(DATA!)
                                                   ▲
   event happened HERE ────────────────────────────┘ (sat unseen for 1.8s)

LONG POLLING                                latency: ~0, 1 held conn/client
Client  REQ▓▓▓▓▓▓▓▓▓▓▓▓RES(DATA) REQ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓RES(timeout) REQ▓▓▓
                       ▲                  server holds, then re-asks
        event happened─┘ (delivered instantly)

SSE                                         latency: ~0, one HTTP response
Client  REQ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▶
Server        └─▶evt1   └─▶evt2      └─▶evt3        └─▶evt4     (never ends)
                        (one-way only: client cannot speak on this channel)

WEBSOCKET                                   latency: ~0, both directions
Client  HTTP Upgrade ──▶ 101 Switching Protocols
        ◀════════════════ full-duplex frames ════════════════▶
        ──▶"send" ◀──"msg" ◀──"msg" ──▶"typing" ◀──"receipt" ──▶ping ◀──pong
```

**Diagram 2 — HTTP/1.1 sequential vs HTTP/2 multiplexed vs HTTP/3.** Fetching 3 assets.

```
HTTP/1.1 — one at a time per connection (HOL blocking at the HTTP layer)
 conn ├──── GET a.js ────┤├──── GET b.css ────┤├──── GET c.png ────┤
      └──────────────── total = t(a) + t(b) + t(c) ─────────────────┘
      (browser opens ~6 of these in parallel to compensate)

HTTP/2 — one TCP connection, 3 interleaved streams
 ┌────────────────── single TCP + TLS connection ──────────────────┐
 │ stream 1 (a.js)  ▓▓▓░░▓▓▓░░░▓▓▓                                 │
 │ stream 3 (b.css) ░░▓▓▓░░▓▓▓░░░                                  │  frames
 │ stream 5 (c.png) ░▓▓░░▓▓░░▓▓▓▓▓                                 │  interleave
 └─────────────────────────────────────────────────────────────────┘
   total ≈ max(t) not sum(t).  BUT:
   ✗ packet lost ──▶ TCP must retransmit in order ──▶ ALL 3 streams stall

HTTP/3 — QUIC over UDP, streams are independent
 ┌───────────────────── single QUIC connection ────────────────────┐
 │ stream 1 (a.js)  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓  ── keeps flowing ──▶           │
 │ stream 3 (b.css) ▓▓▓▓✗..stalled..▓▓▓  ── only THIS one waits    │
 │ stream 5 (c.png) ▓▓▓▓▓▓▓▓▓▓▓▓▓▓  ── keeps flowing ──▶           │
 └─────────────────────────────────────────────────────────────────┘
   + Connection ID survives an IP change:  WiFi ──▶ 5G, same connection
```

**Diagram 3 — scaling WebSockets across servers (draw this one in the interview).**

```
                         ┌───────────────────┐
        Alice ═══════════▶  Load Balancer    ◀═══════════ Bob
      (WS conn)          │ (sticky routing:  │          (WS conn)
                         │  same client →    │
                         │  same server)     │
                         └────┬─────────┬────┘
                  ┌───────────┘         └───────────┐
                  ▼                                 ▼
        ┌──────────────────┐              ┌──────────────────┐
        │  WS Server A     │              │  WS Server B     │
        │  local sockets:  │              │  local sockets:  │
        │    alice ─▶ sock │              │    bob   ─▶ sock │
        └────────┬─────────┘              └─────────▲────────┘
                 │                                  │
      PUBLISH "user:bob"                 SUBSCRIBE "user:bob"
                 │                                  │
                 ▼                                  │
        ┌────────────────────────────────────────────────────┐
        │            Redis Pub/Sub  (the BACKPLANE)          │
        │   channels: user:alice, user:bob, room:42, ...     │
        └────────────────────────────────────────────────────┘
```

Alice's message for Bob arrives at Server A. Server A has **no idea where Bob is** and doesn't need to — it publishes to the channel `user:bob`. Server B holds Bob's socket and is therefore subscribed to `user:bob`; Redis delivers it, and Server B writes it down Bob's live socket. Add a hundred more servers and nothing about this changes. That property — *senders never need to know the recipient's location* — is the entire reason the backplane exists.

---

## Real world examples

### Slack — WebSockets, with a "flannel" cache in front

Slack's desktop and web clients hold a persistent WebSocket to a gateway that pushes new messages, presence changes, typing indicators, and reactions. Because a large workspace's initial state (channels, users, emoji) is enormous, Slack introduced an edge cache layer (they publicly called it **Flannel**) that sits at the WebSocket edge and serves that metadata lazily instead of shipping the whole workspace to every client on connect. It's a good reminder that the hard part of a WebSocket system is usually not the socket — it's what you have to send down it on connect.

### ChatGPT and other LLM UIs — SSE

Token-by-token streaming in LLM chat UIs is, in the common case, a plain **Server-Sent Events** stream: the client POSTs the prompt over ordinary HTTP, and the response comes back as `text/event-stream` with one `data:` chunk per token, terminated by a sentinel event. It's a textbook fit — the flow is one-directional (server → client), the client already has a separate request channel for sending, and you get reconnection semantics for free. If you've ever wondered why streaming LLM output "just works" in a browser with no library, that's why.

### Uber — a layered approach

Uber's rider and driver apps need frequent server → client updates (driver location, trip state, surge). Uber has publicly described building a persistent-connection push layer (their RAMEN system) specifically because polling could not meet the latency and battery requirements at their scale, while also being explicit that mobile networks force you to design for constant disconnection and resumption. The general lesson holds for any mobile product: **the connection will drop — your design is judged on how gracefully it resumes**, which is exactly why HTTP/3's connection migration is such a big deal for mobile.

---

## Trade-offs

| Approach | Latency | Server cost | Bidirectional | Auto-reconnect | Scales easily | Complexity |
|---|---|---|---|---|---|---|
| **Short polling** | Up to N sec | Wasteful (many empty reqs) | No | N/A (stateless) | Trivially | Very low |
| **Long polling** | ~Instant | 1 held conn/client | No | Manual | Medium | Medium |
| **SSE** | ~Instant | 1 held conn/client (cheap) | **No** | **Yes, native** | Medium | **Low** |
| **WebSockets** | ~Instant | 1 conn/client + state | **Yes** | **No — you write it** | **Hard (needs backplane)** | High |

**WebSockets specifically — what you give up:**

| You gain | You pay |
|---|---|
| True full-duplex | Statefulness — sticky sessions, and every deploy kills every connection |
| ~2 bytes/message overhead | You must write heartbeats, reconnect, and backoff yourself |
| Sub-millisecond push | Horizontal scaling needs Redis/NATS pub-sub you now have to operate |
| Binary frames | Some proxies/firewalls block the Upgrade; you need a fallback path |

**HTTP versions:**

| You gain | You pay |
|---|---|
| HTTP/2 multiplexing kills the 6-connection limit | TCP HOL blocking remains; can be *worse* than HTTP/1.1 on lossy links |
| HTTP/3 removes HOL blocking + survives network switches | UDP is blocked on some corporate networks; you always keep an HTTP/2 fallback |

**Rule of thumb:** **Default to plain HTTP. If you need push, default to SSE. Only reach for WebSockets when the client genuinely needs to send high-frequency messages *back*.** Most "we need WebSockets" requirements are actually "we need the server to push", and SSE gives you that with a tenth of the operational burden. And enable HTTP/2 (at minimum) at your edge today — it costs you a config line and makes SSE's connection limit disappear.

---

## Common interview questions on this topic

### Q1: "Why not just use polling for a chat app?"

**Hint:** Do the arithmetic out loud — it's the whole answer. 10k users at a 2s interval is 5,000 req/s, of which ~97% return nothing, and users *still* see up to 2 seconds of latency. You're paying for TLS, headers, auth middleware and a DB hit on every empty poll. Then make the trade explicit: shorten the interval and cost explodes; lengthen it and latency gets worse. There is no good setting, which is the signal that you need push, not pull.

### Q2: "You have 10 million WebSocket connections across 500 servers. User A on server 3 sends a message to user B on server 217. How does it arrive?"

**Hint:** This is *the* question. Don't try to make server 3 find server 217. Invert it: **route to a topic, not a server.** Every gateway subscribes to `user:{id}` on a Redis/NATS pub-sub backplane for each user it currently holds. Server 3 publishes to `user:B` and forgets about it; server 217 is subscribed, receives it, writes it to B's local socket. Then mention the two extras: sticky routing at the LB so a *reconnecting* client lands somewhere sane, and a presence/registry table (userId → serverId, in Redis with a TTL) if you need to know whether B is online at all.

### Q3: "SSE or WebSockets for a live notification feed?"

**Hint:** **SSE**, and say why crisply: notifications flow one way. You get native auto-reconnect, `Last-Event-ID` resume (so you replay what was missed after a network blip), plain HTTP so your existing auth/proxies/observability all work, and no sticky-session requirement. Choosing WebSockets here means writing heartbeats, reconnect, and backoff *and* taking on stateful scaling — for a channel you'd only ever use in one direction. Add the caveat: serve it over HTTP/2, or the ~6-connections-per-domain limit will bite you when a user opens many tabs.

### Q4: "HTTP/2 already multiplexes. So why does HTTP/3 exist?"

**Hint:** Because HTTP/2 fixed head-of-line blocking at the *HTTP* layer but not at the *TCP* layer. TCP delivers one ordered byte stream, so a single lost packet stalls **every** multiplexed stream on that connection, even the ones whose bytes already arrived. HTTP/3 moves to QUIC-over-UDP, which owns the streams itself and keeps them independent — a lost packet only stalls its own stream. Then land the two bonuses: 0-RTT resumption, and connection migration via Connection ID (your connection survives WiFi → cellular, because it isn't identified by an IP:port tuple).

### Q5: "Your WebSocket server shows 50,000 connections but only 30,000 users are actually online. What's happening?"

**Hint:** Dead connections you never detected. A peer that vanished (laptop lid shut, phone lost signal, NAT box dropped the mapping) leaves a socket that looks perfectly open from the server's side — TCP won't tell you for hours, if ever. You need an application-level **heartbeat**: server pings every ~30s, marks the socket dead if the pong doesn't come back before the next tick, and calls `terminate()`. Mention the flip side too: those pings also stop intermediate proxies from killing the connection as "idle".

---

## Practice exercise

### Build a live "order tracker" — three ways, and measure the difference

Build a tiny Node server that simulates a pizza order moving through 5 states (`placed → confirmed → cooking → out_for_delivery → delivered`), advancing one state every 4 seconds. Then expose the *same* state stream through three endpoints:

1. **`GET /poll`** — returns the current state immediately. Client polls it every 2 seconds.
2. **`GET /longpoll?since=<state>`** — holds the request open until the state *changes* (or 20 seconds pass), then responds.
3. **`GET /sse`** — a `text/event-stream` that pushes each state transition as it happens. Client uses `EventSource`. Include an `id:` on every event and honour the `Last-Event-ID` header on reconnect.

**What to produce:**
- The three server handlers and three matching clients (a `<script>` tag in one HTML file is fine).
- **A counter of HTTP requests received per endpoint** over one full 20-second order lifecycle. Write the three numbers down. The polling client should have made ~10 requests to learn 5 things; the SSE client should have made **1**.
- **A latency measurement**: for each approach, the milliseconds between the server changing state and the client rendering it. Print it.
- Finally, kill the server mid-order and restart it. Observe: the `EventSource` client reconnects **by itself** and (if you implemented `Last-Event-ID`) resumes without a gap. The long-polling client throws a network error and stops unless you wrote a retry loop.

**Then answer in two sentences:** if the customer now needs to *message the driver*, which of the three would you change, and would you switch to WebSockets or just add a `POST /message` endpoint alongside the SSE stream? (There is a right answer, and it is the boring one.)

---

## Quick reference cheat sheet

- **The core question:** HTTP is request-response — the client must ask. Every technique here is a workaround for the server's inability to speak first.
- **Short polling:** `setInterval` + `fetch`. 10k clients @ 2s = 5,000 req/s, ~97% of them empty. Fine at small scale, indefensible at large scale.
- **Long polling:** the server *holds* the request open until it has data or times out; the client immediately re-asks. Near-real-time over plain HTTP. Set your timeout **below** your proxy's.
- **Reconnect storms are real:** a server restart makes every client reconnect on the same millisecond. Always use **exponential backoff with jitter**.
- **SSE = `Content-Type: text/event-stream`.** One never-ending HTTP response. ~15 lines of Node, 4 lines of `EventSource`. Auto-reconnects, resumes via `Last-Event-ID`.
- **SSE is one-way** (server → client) and, on HTTP/1.1, eats one of the browser's ~6 connections per domain. Serve it over HTTP/2 and that limit vanishes.
- **SSE is the right default for push:** notifications, dashboards, progress bars, live feeds, **LLM token streaming**.
- **WebSockets:** HTTP `Upgrade` → `101 Switching Protocols` → binary framing, ~2 bytes of overhead per message. Full-duplex. The right call for chat, games, collab editing, trading.
- **WebSockets give you nothing for free:** no auto-reconnect, no heartbeat, no liveness detection. You write all three, or you leak dead sockets for hours.
- **Scaling WebSockets = a pub/sub backplane.** Don't route to a server; publish to `user:{id}` on Redis/NATS and let whichever server holds that socket deliver it. Sticky sessions at the LB on top.
- **HTTP/1.1:** one request at a time per connection → HOL blocking → the ~6-connection workaround → sprite sheets, bundling, domain sharding.
- **HTTP/2:** multiplexed streams over **one** TCP connection + HPACK header compression. Makes those hacks pointless and **domain sharding actively harmful**. Server push is **deprecated and removed** — say so.
- **HTTP/2's flaw:** TCP-level head-of-line blocking. One lost packet stalls *every* stream on the connection.
- **HTTP/3 = QUIC over UDP.** Independent streams (a lost packet stalls only its own), 0-RTT resumption, and **connection migration** — a Connection ID instead of an IP:port tuple, so your connection survives WiFi → cellular. Huge on mobile.
- **Neighbours:** gRPC rides HTTP/2 (that's *why* it gets streaming free). WebRTC does peer-to-peer media, with a WebSocket usually doing its signalling.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [11 — How the Web Works](./11-how-the-web-works.md) — the request-response cycle these protocols all extend or escape |
| **Next** | [69 — API Design: REST, GraphQL, gRPC](./69-api-design-rest-graphql-grpc.md) — gRPC rides on HTTP/2, and GraphQL subscriptions ride on WebSockets |
| **Related** | [10 — Networking Fundamentals](./10-networking-fundamentals.md) — TCP vs UDP, which is exactly why HTTP/3 abandoned TCP |
| **Related** | [99 — HLD: Chat System](./99-hld-chat-system.md) — the WebSocket backplane from this doc, applied end to end |
| **Related** | [98 — HLD: Notification System](./98-hld-notification-system.md) — where SSE and push channels earn their keep |
