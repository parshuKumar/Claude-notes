# 39 — Server-Sent Events (SSE)

## What is this?

Server-Sent Events (SSE) is a way for a server to push a continuous stream of updates to a browser over a single, long-lived HTTP connection — without the client ever having to ask again. It is **one-way only**: server → client. Think of it like a radio broadcast — the station keeps transmitting and any tuned-in radio just keeps receiving, but the radio can't talk back to the station on that same channel. The browser has a built-in `EventSource` object that connects once and automatically reconnects if the connection drops.

## Why does it matter for backend development?

Backend developers build features like live notifications, stock ticker updates, order-status tracking, deployment logs streaming to a dashboard, and AI chat responses that "type" word-by-word — all of these are naturally one-directional (server has new data, client just needs to see it). SSE gives you this for the cost of a plain HTTP endpoint: no new protocol, no extra library, no separate port, and it works through normal HTTP proxies and load balancers that already understand streaming responses. Reaching for a full WebSocket server for a "the server just needs to push updates" problem is usually over-engineering — SSE is simpler to build, simpler to scale, and the browser handles reconnection for free.

---

## Syntax / API

```js
// Import Node's built-in HTTP module — SSE needs nothing extra, it's just HTTP
const http = require('http');

// Create a plain HTTP server
const server = http.createServer((req, res) => {
  // Only handle the SSE endpoint — everything else gets a normal response
  if (req.url === '/events') {
    // These three headers are what turn a normal response into an SSE stream
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',   // tells browser: this is an SSE stream
      'Cache-Control': 'no-cache',           // prevents proxies/browsers from caching the stream
      'Connection': 'keep-alive',            // keep the TCP connection open indefinitely
    });

    // res.write() sends a chunk without ending the response — connection stays open
    res.write('data: connected\n\n');        // every SSE message MUST end with a blank line ("\n\n")

    // Send a new message every 2 seconds — simulating real-time updates
    const intervalId = setInterval(() => {
      const payload = JSON.stringify({ time: new Date().toISOString() }); // data must be a string
      res.write(`data: ${payload}\n\n`);      // "data: " prefix + payload + blank-line terminator
    }, 2000);

    // Node fires 'close' when the client disconnects (tab closed, network drop, etc.)
    req.on('close', () => {
      clearInterval(intervalId);              // stop writing to a socket nobody is reading anymore
    });

    return; // don't fall through to the default response below
  }

  // Default response for any other route
  res.writeHead(404).end('Not found');
});

server.listen(3000); // start listening on port 3000

// ── Client side (runs in the browser, not Node) ─────────────────────────────
// const source = new EventSource('http://localhost:3000/events');
// source.onmessage = (event) => console.log(event.data);   // fires on every "data:" message
// source.onerror    = () => console.log('connection lost, browser will auto-retry');
```

---

## How it works — line by line

An SSE response is just a normal HTTP response that never technically "finishes" — the server keeps the connection open and keeps writing small text chunks to it over time.

- `res.writeHead(200, { 'Content-Type': 'text/event-stream', ... })` sends the response headers immediately, telling the browser "don't wait for this to end, treat every chunk as a stream of events."
- `res.write('data: connected\n\n')` sends one SSE **message**. The wire format is a plain text protocol: a line starting with `data:` holds the payload, and a **blank line** (`\n\n`) tells the browser "this message is complete, deliver it now."
- The server never calls `res.end()` — that would close the stream. Instead it keeps calling `res.write()` again and again, whenever there's new data to send.
- On the browser side, `new EventSource(url)` opens the connection and listens forever. Each time a complete `data: ...\n\n` block arrives, the browser fires the `message` event with that payload — your JavaScript code never has to poll or ask again.
- If the connection drops for any reason (server restart, network blip), the browser's `EventSource` **automatically reconnects** after a short delay — this reconnection logic is built into every browser, you don't write it yourself.
- `req.on('close', ...)` is the server's way of knowing when a specific client has actually left, so it can stop wasting CPU/memory writing to a dead connection.

---

## Example 1 — basic

```js
// File: src/sse-clock.js
// A minimal SSE server that pushes the current time to the browser every second.

const http = require('http'); // built-in module, no install needed

const server = http.createServer((req, res) => {
  if (req.url !== '/clock') {
    res.writeHead(404, { 'Content-Type': 'text/plain' }); // any other route → 404
    res.end('Not found');
    return;
  }

  // SSE-required response headers
  res.writeHead(200, {
    'Content-Type': 'text/event-stream', // marks this as an event stream, not JSON/HTML
    'Cache-Control': 'no-cache',         // don't let CDNs/browsers cache a live stream
    'Connection': 'keep-alive',          // hold the socket open instead of closing after one reply
  });

  // Send one tick immediately so the client sees something right away
  res.write(`data: ${new Date().toLocaleTimeString()}\n\n`);

  // Every 1000ms, push the current time as a new SSE message
  const tickId = setInterval(() => {
    const currentTime = new Date().toLocaleTimeString(); // format: "14:32:10"
    res.write(`data: ${currentTime}\n\n`);                // "data:" + payload + blank line
  }, 1000);

  // Clean up when the browser tab closes or navigates away
  req.on('close', () => {
    clearInterval(tickId);   // stop the timer — no more writes to a closed socket
    console.log('Client disconnected from /clock');
  });
});

server.listen(4000, () => {
  console.log('SSE clock server running on http://localhost:4000/clock');
});

// In a browser console, test it with:
// const source = new EventSource('http://localhost:4000/clock');
// source.onmessage = (event) => console.log('Tick:', event.data);
```

---

## Example 2 — real world backend use case

```js
// File: src/order-status-stream.js
// Realistic pattern: push live order status updates to a customer's dashboard.
// Multiple browser tabs can be watching the same order at once.

const http = require('http');
const { EventEmitter } = require('events'); // Node's built-in pub/sub mechanism (Topic 19)

// One shared event bus — order-service.js would call orderEvents.emit() when a status changes
const orderEvents = new EventEmitter();

// Track which res streams are currently subscribed to which orderId
const subscribers = new Map(); // orderId -> Set of response objects

function subscribe(orderId, res) {
  if (!subscribers.has(orderId)) {
    subscribers.set(orderId, new Set()); // first subscriber for this order
  }
  subscribers.get(orderId).add(res);     // add this client's response stream
}

function unsubscribe(orderId, res) {
  const clients = subscribers.get(orderId);
  if (!clients) return;
  clients.delete(res);                   // remove this specific client
  if (clients.size === 0) subscribers.delete(orderId); // clean up empty sets
}

// Whenever the order service emits a status change, push it to every subscribed client
orderEvents.on('status-change', ({ orderId, status }) => {
  const clients = subscribers.get(orderId);
  if (!clients) return; // nobody is watching this order right now

  const message = JSON.stringify({ orderId, status, updatedAt: Date.now() });
  for (const clientRes of clients) {
    clientRes.write(`event: status-update\n`);  // named event — client listens with addEventListener
    clientRes.write(`data: ${message}\n\n`);     // payload + terminator
  }
});

const server = http.createServer((req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`); // Topic 22 — url module

  if (url.pathname === '/orders/stream') {
    const orderId = url.searchParams.get('orderId'); // e.g. /orders/stream?orderId=ORD-1042
    const authToken = req.headers['authorization'];   // in real code: verify this JWT (Topic 57)

    if (!orderId || !authToken) {
      res.writeHead(400).end('Missing orderId or auth token');
      return;
    }

    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'X-Accel-Buffering': 'no', // tells Nginx not to buffer this stream (important in production!)
    });

    res.write(`retry: 3000\n`);          // tells browser: if disconnected, wait 3s before reconnecting
    res.write(`data: subscribed\n\n`);   // confirm the subscription immediately

    subscribe(orderId, res); // start receiving future updates for this order

    // Send a heartbeat comment every 20s to keep proxies/load-balancers from timing out the socket
    const heartbeatId = setInterval(() => {
      res.write(': heartbeat\n\n'); // a line starting with ":" is a comment — browser ignores it
    }, 20000);

    req.on('close', () => {
      clearInterval(heartbeatId);   // stop the heartbeat timer
      unsubscribe(orderId, res);    // remove this client so we stop writing to a dead socket
    });

    return;
  }

  res.writeHead(404).end('Not found');
});

server.listen(5000, () => console.log('Order status SSE server on :5000'));

// Elsewhere in the app, when an order actually changes (e.g. in a controller or worker):
// orderEvents.emit('status-change', { orderId: 'ORD-1042', status: 'shipped' });

module.exports = { orderEvents }; // exported so other modules can emit status changes
```

---

## Common mistakes

### Mistake 1 — Forgetting the required SSE response headers

```js
// ❌ WRONG — no Content-Type means the browser treats this as a normal response,
// EventSource never fires onmessage, and some browsers buffer/close it early
res.writeHead(200);
res.write('data: hello\n\n');

// ✅ CORRECT — these three headers are what make EventSource recognize the stream
res.writeHead(200, {
  'Content-Type': 'text/event-stream', // required — declares the SSE wire format
  'Cache-Control': 'no-cache',         // prevents caching a live, ever-changing stream
  'Connection': 'keep-alive',          // keeps the underlying TCP socket open
});
res.write('data: hello\n\n');
```

### Mistake 2 — Missing the blank-line message terminator

```js
// ❌ WRONG — single "\n" does not close the message; the browser keeps buffering
// and never fires onmessage because it's still waiting for the terminator
res.write(`data: ${JSON.stringify({ userId: 'user_42' })}\n`);

// ✅ CORRECT — every SSE message MUST end with a blank line ("\n\n")
res.write(`data: ${JSON.stringify({ userId: 'user_42' })}\n\n`);
// If you send multiple "data:" lines for one message, only the FINAL line
// needs the extra "\n" that creates the blank-line terminator
```

### Mistake 3 — Not cleaning up when the client disconnects

```js
// ❌ WRONG — the interval keeps running forever, even after the browser tab is closed,
// silently leaking memory and CPU as more and more dead clients pile up
const intervalId = setInterval(() => {
  res.write(`data: ${Date.now()}\n\n`); // throws/fails silently once socket is dead
}, 1000);
// no cleanup registered — process keeps this handler and timer alive indefinitely

// ✅ CORRECT — always listen for 'close' and clear timers / remove from tracking structures
const intervalId = setInterval(() => {
  res.write(`data: ${Date.now()}\n\n`);
}, 1000);

req.on('close', () => {
  clearInterval(intervalId); // stop the timer — free the resource immediately
});
```

---

## Practice exercises

### Exercise 1 — easy

Build an SSE endpoint `/random` on a plain `http` server that:
1. Sets the three required SSE headers
2. Sends a new random number (1–100) as a `data:` message every 1.5 seconds
3. Properly clears its interval when the client disconnects (`req.on('close', ...)`)
4. Logs to the console each time a new client connects and each time one disconnects

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a live log-streaming endpoint `/logs/stream` that:
1. Uses a **named event** (`event: log-line`) instead of the default unnamed message
2. Maintains an in-memory array of "log lines" (just strings you push to periodically with `setInterval`, simulating a real app producing logs)
3. Streams every new log line to **all currently connected clients** as it's produced (multiple browser tabs open at once should all receive the same lines)
4. Sends a heartbeat comment (`: heartbeat\n\n`) every 15 seconds so the connection survives idle periods
5. Cleans up properly per-client on disconnect, without affecting other connected clients

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `NotificationStream` module that supports **resuming after reconnect** using SSE's `id:` field and the `Last-Event-ID` header:
1. Each notification you send must include an `id:` line with an incrementing number, in addition to `event:` and `data:`
2. Keep the last 100 notifications in memory (an array works fine) so they can be replayed
3. When a client connects, read the `Last-Event-ID` request header (browsers send this automatically on reconnect) — if present, immediately replay every notification with an id **greater than** that value before streaming new ones live
4. Expose a function `pushNotification(userId, message)` that both stores the notification and immediately delivers it to any currently-connected client for that `userId`
5. Support multiple simultaneous users, each only receiving their own notifications
6. Handle disconnects cleanly so memory doesn't grow unbounded as clients come and go

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT SSE IS
  One-way stream: SERVER → CLIENT only, over a single long-lived HTTP connection
  Built on plain HTTP — no new protocol, no upgrade handshake (unlike WebSockets)

REQUIRED RESPONSE HEADERS
  Content-Type: text/event-stream   → marks the response as an SSE stream
  Cache-Control: no-cache           → stop proxies/browsers from caching it
  Connection: keep-alive            → keep the socket open indefinitely

WIRE FORMAT (each field is its own line, message ends with a blank line)
  data: <payload>\n\n               → the message body (required)
  event: <name>\n                   → optional custom event name (default: "message")
  id: <number>\n                    → optional, enables Last-Event-ID resume on reconnect
  retry: <ms>\n                     → optional, tells browser how long to wait before reconnecting
  : <comment>\n\n                   → a line starting with ":" is ignored — used for heartbeats

SERVER SIDE (Node)
  res.writeHead(200, { headers })   → send SSE headers, never call res.end()
  res.write(`data: ${x}\n\n`)       → send one message, connection stays open
  req.on('close', cleanup)         → ALWAYS clean up timers/listeners on disconnect

CLIENT SIDE (browser, built-in — no library needed)
  const source = new EventSource(url);
  source.onmessage        = (e) => ...        // default "message" events
  source.addEventListener('log-line', (e) => ...) // named events
  source.onerror           = (e) => ...        // fires on drop; browser auto-reconnects
  Last-Event-ID header sent automatically on reconnect if server sent "id:" lines

SSE vs WEBSOCKETS vs POLLING
  Polling      → client repeatedly asks "anything new?" — simple but wasteful, laggy
  SSE          → server pushes, one-way only, plain HTTP, auto-reconnect built in
  WebSockets   → full duplex (both directions), needs upgrade handshake, more setup

WHEN TO USE SSE
  ✓ Notifications, live dashboards, progress bars, log tails, AI streaming text
  ✓ You only need server → client, never client → server on the same channel
  ✗ Chat apps, multiplayer games, anything needing client → server in real time
  ✗ Binary data — SSE is text-only (use WebSockets or base64-encode if forced)

GOTCHAS
  Browsers limit ~6 concurrent EventSource connections per domain over HTTP/1.1
  Nginx/reverse proxies may buffer responses — set "X-Accel-Buffering: no"
  A single dropped "\n" breaks the message terminator — always use "\n\n"
  Never forget cleanup on 'close' — dead connections leak memory over time
```

---

## Connected topics

- **38 — WebSockets fundamentals** — the two-way alternative to SSE; know when full-duplex communication is actually needed versus one-way push
- **20 — http module** — SSE is built entirely on the raw `http` module's `res.write()` streaming behavior, no extra protocol involved
- **99 — Socket.IO in depth** — a higher-level real-time library (built on WebSockets) worth comparing against plain SSE for rooms, broadcasting, and reconnection handling
