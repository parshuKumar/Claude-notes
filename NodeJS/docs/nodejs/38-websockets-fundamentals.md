# 38 — WebSockets fundamentals

## What is this?

A WebSocket is a **persistent, two-way communication channel** between a client and a server that stays open over a single TCP connection. Unlike normal HTTP where the client must ask a question to get an answer, either side can send data at any moment once the connection is established. Think of the difference between mailing letters back and forth (HTTP — one request, one response, then the connection closes) versus being on a live phone call (WebSocket — both people can speak whenever they want, and the line stays open until someone hangs up).

## Why does it matter for backend development?

Plenty of backend features need the server to push data the instant something happens, without the client repeatedly asking "anything new?" — chat apps, live order tracking, stock tickers, multiplayer games, collaborative editors, and live dashboards all rely on this. Building this with plain HTTP means constant polling, which wastes bandwidth and adds latency. A backend developer reaches for WebSockets (usually via the `ws` library, or a higher-level wrapper like Socket.IO — Topic 99) whenever the product requirement is genuinely "real-time, bidirectional" rather than just "load fresh data occasionally."

---

## Syntax / API

```js
// Install first: npm install ws
const WebSocket = require('ws');       // the 'ws' library — the standard low-level WebSocket implementation for Node

// ── Creating a WebSocket SERVER ──────────────────────────────────────────────
const wss = new WebSocket.Server({ port: 8080 });   // starts listening for WebSocket connections on port 8080

// 'connection' fires once per client that successfully connects
wss.on('connection', (clientSocket, incomingRequest) => {
  console.log('New client connected from:', incomingRequest.socket.remoteAddress); // log where the client came from

  // 'message' fires every time this specific client sends data
  clientSocket.on('message', (rawData) => {
    console.log('Received:', rawData.toString());   // rawData arrives as a Buffer — convert to string to read it
  });

  // 'close' fires when this client disconnects (tab closed, network drop, etc.)
  clientSocket.on('close', () => {
    console.log('Client disconnected');
  });

  // send() pushes data to THIS client at any time — no request needed from them
  clientSocket.send('Welcome! You are connected.');
});

// ── Creating a WebSocket CLIENT (from Node, or the same API works in browsers) ─
const socket = new WebSocket('ws://localhost:8080');   // 'ws://' is the WebSocket protocol (wss:// for TLS/encrypted)

socket.on('open', () => {                 // fires once the handshake succeeds and the connection is ready
  socket.send('Hello from the client');   // send a message to the server
});

socket.on('message', (data) => {          // fires whenever the server sends something
  console.log('Server says:', data.toString());
});
```

---

## How it works — line by line

A WebSocket connection is born from a regular HTTP request that asks to be "upgraded":

1. The client sends a normal-looking HTTP GET request, but with two special headers: `Upgrade: websocket` and `Connection: Upgrade`, plus a random key (`Sec-WebSocket-Key`).
2. If the server supports WebSockets, instead of replying with a normal HTTP response, it replies with status `101 Switching Protocols` and a matching `Sec-WebSocket-Accept` header (computed from the client's key, proving it understood the request).
3. At that moment, the underlying TCP connection **stops behaving like HTTP** and becomes a raw, persistent, full-duplex pipe — this is called the **upgrade handshake**, and it happens exactly once per connection.
4. From then on, both sides can call `.send()` whenever they want, and the other side's `'message'` event fires. There is no more request/response pairing — messages flow independently in both directions.
5. The connection stays open until either side closes it (`.close()`) or the network drops it, at which point `'close'` fires on both ends.

```
Client                                  Server
  |-- GET /chat  Upgrade: websocket -->|   (1) normal-looking HTTP request
  |<-- 101 Switching Protocols --------|   (2) server agrees, handshake done
  |==== now a raw TCP pipe, no more HTTP framing ====|
  |-- send("hi") ---------------------->|   (3) either side sends anytime
  |<------------------- send("hi back")-|
  |-- close() ------------------------->|   (4) either side can close
```

The `ws` library hides all the handshake math and framing details behind `.on('connection')`, `.send()`, and `.on('message')` — you never construct the raw HTTP upgrade headers yourself.

---

## Example 1 — basic

```js
// File: server.js — a minimal echo server: whatever the client sends, it sends back
const WebSocket = require('ws');   // load the ws library

const wss = new WebSocket.Server({ port: 4000 });   // listen for WebSocket connections on port 4000
console.log('WebSocket server running on ws://localhost:4000');

wss.on('connection', (clientSocket) => {             // runs once per connected client
  console.log('Client connected');

  clientSocket.on('message', (rawData) => {          // runs every time this client sends a message
    const text = rawData.toString();                 // convert Buffer to a readable string
    console.log('Client sent:', text);
    clientSocket.send(`Echo: ${text}`);               // send the same text back, prefixed
  });

  clientSocket.on('close', () => {                    // runs when this client disconnects
    console.log('Client disconnected');
  });
});
```

```js
// File: client.js — connects to the echo server and sends one message
const WebSocket = require('ws');                     // ws also works as a Node-side client library

const socket = new WebSocket('ws://localhost:4000');  // connect to the local server

socket.on('open', () => {                             // fires once the handshake completes
  socket.send('Hello server!');                       // send a test message
});

socket.on('message', (data) => {                      // fires when the server replies
  console.log('Server replied:', data.toString());     // → "Server replied: Echo: Hello server!"
  socket.close();                                       // close the connection, we're done
});
```

---

## Example 2 — real world backend use case

```js
// File: src/realtime/orderStatusServer.js
// Real-world pattern: push live order-status updates to the specific browser tab
// that placed the order — not to every connected client (that would leak other users' data).

const WebSocket = require('ws');
const { verifyAuthToken } = require('../auth/tokenService');   // your own JWT/session verification (Topic 57)

const wss = new WebSocket.Server({ port: 8081 });

// Map of userId -> Set of that user's open sockets (a user might have multiple tabs open)
const userConnections = new Map();

wss.on('connection', (clientSocket, incomingRequest) => {
  // Extract the auth token the client passed in the connection URL, e.g. ws://host?token=xyz
  const url = new URL(incomingRequest.url, 'ws://localhost');   // parse the upgrade request's URL (Topic 22)
  const authToken = url.searchParams.get('token');

  let userId;
  try {
    userId = verifyAuthToken(authToken);           // throws if the token is missing/invalid/expired
  } catch (err) {
    clientSocket.close(4001, 'Unauthorized');       // custom close code — reject unauthenticated sockets
    return;                                          // stop here, never register this connection
  }

  // Register this socket under the authenticated user
  if (!userConnections.has(userId)) {
    userConnections.set(userId, new Set());          // first connection for this user — create the set
  }
  userConnections.get(userId).add(clientSocket);

  clientSocket.on('close', () => {
    const sockets = userConnections.get(userId);
    sockets.delete(clientSocket);                    // remove this specific tab/socket
    if (sockets.size === 0) {
      userConnections.delete(userId);                // clean up fully once no tabs remain — avoids memory leaks
    }
  });
});

// Called from elsewhere in the app (e.g. after a payment webhook updates order status)
function notifyOrderStatusChanged(userId, orderId, newStatus) {
  const sockets = userConnections.get(userId);       // find this user's open sockets, if any
  if (!sockets) return;                                // user has no live connection right now — nothing to push

  const payload = JSON.stringify({                    // always send structured JSON, never raw strings
    type: 'ORDER_STATUS_UPDATE',
    orderId,
    status: newStatus,
    updatedAt: new Date().toISOString(),
  });

  for (const socket of sockets) {
    if (socket.readyState === WebSocket.OPEN) {        // only send if the socket is actually still connected
      socket.send(payload);
    }
  }
}

module.exports = { notifyOrderStatusChanged };
```

---

## Common mistakes

### Mistake 1 — Never cleaning up disconnected clients (memory leak)

```js
// ❌ WRONG — clients are added to a list but never removed on disconnect
const allClients = [];
wss.on('connection', (clientSocket) => {
  allClients.push(clientSocket);      // grows forever — dead sockets pile up and are never freed
});

// ✅ CORRECT — always remove the socket from tracking structures on 'close'
const allClients = new Set();
wss.on('connection', (clientSocket) => {
  allClients.add(clientSocket);
  clientSocket.on('close', () => {
    allClients.delete(clientSocket);  // frees memory and stops broadcasting to a dead connection
  });
});
```

### Mistake 2 — Assuming every incoming message is valid JSON

```js
// ❌ WRONG — JSON.parse throws on malformed input and crashes the whole process
clientSocket.on('message', (rawData) => {
  const requestBody = JSON.parse(rawData.toString());   // throws SyntaxError on bad input, unhandled
  handleMessage(requestBody);
});

// ✅ CORRECT — wrap parsing in try/catch and reject bad messages gracefully (Topic 07)
clientSocket.on('message', (rawData) => {
  let requestBody;
  try {
    requestBody = JSON.parse(rawData.toString());
  } catch (err) {
    clientSocket.send(JSON.stringify({ error: 'Invalid JSON payload' }));
    return;                                                // stop processing this message, connection stays open
  }
  handleMessage(requestBody);
});
```

### Mistake 3 — Accepting connections without checking authentication

```js
// ❌ WRONG — anyone who knows the URL can connect and receive/send data as if authenticated
wss.on('connection', (clientSocket) => {
  clientSocket.send('Welcome to your private dashboard feed');   // no identity check at all
});

// ✅ CORRECT — verify the auth token during the handshake, before trusting the connection
wss.on('connection', (clientSocket, incomingRequest) => {
  const url = new URL(incomingRequest.url, 'ws://localhost');
  const authToken = url.searchParams.get('token');
  let userId;
  try {
    userId = verifyAuthToken(authToken);       // reject immediately if this fails
  } catch (err) {
    clientSocket.close(4001, 'Unauthorized');
    return;
  }
  clientSocket.send(`Welcome, user ${userId}`);
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a WebSocket server on port 5000 using `ws` that:
1. Logs `"Client connected"` whenever a client connects
2. Sends the message `"Connected to server"` immediately on connection
3. Whenever it receives a message from a client, converts it to uppercase and sends it back
4. Logs `"Client disconnected"` when the connection closes

Then write a small client script that connects, sends `"hello world"`, logs the server's reply, and closes the connection.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a simple broadcast chat server on port 6000 where:
1. Every connected client is tracked in a `Set`
2. When any client sends a message, the server broadcasts it to **every other connected client** (not back to the sender)
3. Each broadcast message is JSON in the shape `{ type: 'CHAT_MESSAGE', text, sentAt }`
4. When a client disconnects, it is removed from the tracked set and every remaining client receives a `{ type: 'USER_LEFT' }` notice

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `RoomManager`-backed WebSocket server on port 7000 for a live-notifications system:
1. Clients connect with a URL like `ws://localhost:7000?userId=user_42&room=orders`
2. A `RoomManager` class maintains a `Map<roomName, Set<socket>>` of which sockets belong to which room
3. When a client connects, parse `userId` and `room` from the query string and register the socket into that room (reject the connection with close code `4000` if either is missing)
4. Expose a method `broadcastToRoom(room, payload)` that sends a JSON payload to every socket currently in that room
5. Every 30 seconds, the server sends a `{ type: 'PING' }` to all connected sockets; if a client does not respond with `{ type: 'PONG' }` within 10 seconds, forcibly close that socket (a basic heartbeat / dead-connection detector)
6. On disconnect, remove the socket from its room, and delete the room entry entirely once it is empty

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT IT IS
  Persistent, full-duplex (two-way) connection over a single TCP socket
  Starts as HTTP, then "upgrades" — HTTP framing stops, raw messages begin

THE HANDSHAKE
  Client → GET request with: Upgrade: websocket, Connection: Upgrade, Sec-WebSocket-Key
  Server → 101 Switching Protocols + Sec-WebSocket-Accept
  After this: no more request/response pairing — either side sends anytime

PROTOCOL / URL SCHEME
  ws://   → unencrypted WebSocket   (like http://)
  wss://  → encrypted WebSocket     (like https://) — always use in production

'ws' LIBRARY — SERVER
  new WebSocket.Server({ port })         → start listening
  wss.on('connection', (socket, req) => {})  → new client connected; req = original HTTP upgrade request
  socket.on('message', (data) => {})     → data is a Buffer — call .toString() to read text
  socket.send(data)                      → push data to this one client
  socket.on('close', () => {})           → client disconnected — ALWAYS clean up tracking here
  socket.readyState === WebSocket.OPEN   → check before sending, avoid errors on dead sockets

'ws' LIBRARY — CLIENT
  new WebSocket(url)                     → connect to a server
  socket.on('open', () => {})            → handshake succeeded, safe to send now
  socket.on('message', (data) => {})     → server pushed data
  socket.close()                         → close the connection

GOTCHAS
  Messages are just bytes/strings — always JSON.stringify/JSON.parse yourself, and wrap parse in try/catch
  A disconnected client must be removed from any list/Set/Map you tracked it in — or it leaks memory forever
  Never trust the connection is authenticated by default — verify a token during the handshake
  Broadcasting to "everyone" by default is a common data-leak bug — scope broadcasts to the right room/user
  Long-lived connections need a heartbeat (ping/pong) to detect silently-dead clients (e.g. laptop closed lid)

WHEN TO USE WEBSOCKETS VS ALTERNATIVES
  Need true two-way, low-latency messaging (chat, games, collab editing) → WebSockets
  Only need server → client one-way updates (live feed, notifications)  → Server-Sent Events (Topic 39)
  Need rooms, namespaces, auto-reconnect, fallback transports out of the box → Socket.IO (Topic 99)
```

---

## Connected topics

- **99 — Socket.IO in depth** — a higher-level library built on top of raw WebSockets that adds rooms, namespaces, and automatic reconnection/fallback, so you rarely write raw `ws` server code in large production apps.
- **39 — Server-Sent Events (SSE)** — a simpler one-way (server-to-client only) alternative to WebSockets; important to know when a full duplex connection is overkill.
- **36 — HTTP deep dive — headers, methods, status codes** — the `Upgrade`, `Connection`, and `101 Switching Protocols` mechanics that make the WebSocket handshake possible are HTTP concepts covered there in detail.
