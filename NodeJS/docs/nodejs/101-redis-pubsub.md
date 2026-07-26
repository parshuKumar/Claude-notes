# 101 — Pub/Sub with Redis

## What is this?

Redis Pub/Sub (Publish/Subscribe) is a messaging pattern where one part of your system **publishes** a message to a named **channel**, and every other part of your system that has **subscribed** to that channel instantly receives a copy of it. Think of it like a radio station: the station (publisher) broadcasts on a frequency (channel), and anyone with a radio tuned to that frequency (subscriber) hears it — the station doesn't know or care who's listening, and listeners don't need to know who's broadcasting. Redis just relays the message; it does not store it anywhere.

## Why does it matter for backend development?

Real backend systems are rarely a single process. You might have multiple Node.js server instances behind a load balancer, a worker process handling background jobs, and a notification service — all needing to react to the same event, like "order placed" or "user went online." Pub/Sub lets you broadcast that event once and have every interested process react independently, without them calling each other directly (no tight coupling, no service discovery needed). It is the backbone of scaling WebSocket servers horizontally (Topic 99 uses this exact mechanism), building live dashboards, invalidating caches across servers, and decoupling microservices that just need to "shout" an event into the system.

---

## Syntax / API

```js
// Install first: npm install redis

const { createClient } = require('redis');

// ── Publisher side ──────────────────────────────────────────────────────────
// A regular Redis client CAN publish — no special mode needed for publishing
const publisherClient = createClient({ url: 'redis://localhost:6379' });
await publisherClient.connect();                       // open the TCP connection

// publish(channelName, message) — message must be a string (stringify objects yourself)
await publisherClient.publish(
  'order-events',                                       // channel name — subscribers pick this
  JSON.stringify({ orderId: 'ord_501', status: 'placed' }) // payload as a string
);

// ── Subscriber side ──────────────────────────────────────────────────────────
// IMPORTANT: once a client subscribes, it enters "subscriber mode" and can
// ONLY run pub/sub commands on that connection — use a SEPARATE client for it
const subscriberClient = createClient({ url: 'redis://localhost:6379' });
await subscriberClient.connect();                       // open a second connection

// subscribe(channelName, callback) — callback fires every time a message arrives
await subscriberClient.subscribe('order-events', (message, channel) => {
  console.log(`Received on ${channel}:`, JSON.parse(message)); // parse back to an object
});

// ── Pattern subscribe — listen to multiple channels matching a glob ─────────
await subscriberClient.pSubscribe('order-events:*', (message, channel) => {
  console.log(`Pattern match on ${channel}:`, message);  // channel tells you WHICH one fired
});

// ── Unsubscribing ────────────────────────────────────────────────────────────
await subscriberClient.unsubscribe('order-events');      // stop listening to one channel
await subscriberClient.pUnsubscribe('order-events:*');    // stop listening to a pattern
```

---

## How it works — line by line

- `createClient({ url })` builds a Redis client object but does not connect yet — it just holds the connection settings.
- `.connect()` actually opens the TCP connection to the Redis server; every operation must wait for this to finish.
- `.publish(channel, message)` sends a message down a named channel. Redis broadcasts it to every client currently subscribed to that exact channel name, at that exact moment, and then forgets the message forever — nothing is saved.
- The message argument to `.publish()` must be a plain string. If you want to send an object, you convert it to text with `JSON.stringify()` before publishing, and the receiver converts it back with `JSON.parse()`.
- `.subscribe(channel, callback)` tells Redis "notify this connection whenever something is published to this channel." The callback function you pass in is what actually runs each time a message shows up.
- A subscribed client is locked into subscriber mode — Redis blocks it from running normal commands like `GET` or `SET` on that same connection, which is why real apps always keep two separate client objects: one purely for publishing, one purely for subscribing.
- `.pSubscribe(pattern, callback)` ("pattern subscribe") uses a glob-style pattern like `order-events:*` so one subscription can catch many similarly-named channels — for example `order-events:501` and `order-events:502` would both match, and the callback receives the exact channel name that fired so you know which one it was.
- `.unsubscribe()` and `.pUnsubscribe()` tell Redis to stop delivering messages for that channel or pattern on this connection; the client stays connected, it just stops listening to that specific thing.
- If no one is subscribed to a channel when you publish to it, the message is simply lost — Pub/Sub has no memory and no delivery guarantee, which is the single most important thing to understand before using it.

---

## Example 1 — basic

```js
// File: src/basics/pubsubBasic.js
// Two clients in the same process — one publishes, one subscribes — to see the mechanism work.

const { createClient } = require('redis');

async function main() {
  // Create the subscriber connection first so it's ready to catch messages
  const subscriberClient = createClient({ url: 'redis://localhost:6379' });
  await subscriberClient.connect();                      // open connection #1

  // Create a completely separate connection for publishing
  const publisherClient = createClient({ url: 'redis://localhost:6379' });
  await publisherClient.connect();                       // open connection #2

  // Subscribe to a channel called 'greetings' — callback runs on every message
  await subscriberClient.subscribe('greetings', (message) => {
    console.log('Subscriber got:', message);             // prints whatever was published
  });

  // Small delay so the subscription registers with Redis before we publish
  await new Promise((resolve) => setTimeout(resolve, 200));

  // Publish a plain string message to the 'greetings' channel
  await publisherClient.publish('greetings', 'hello from the publisher');

  // Publish a second message — the same subscriber receives both
  await publisherClient.publish('greetings', 'this is message number two');

  // Give the subscriber a moment to print, then clean up both connections
  setTimeout(async () => {
    await subscriberClient.unsubscribe('greetings');       // stop listening
    await subscriberClient.quit();                        // close connection #1
    await publisherClient.quit();                          // close connection #2
  }, 500);
}

main().catch((err) => console.error('Pub/Sub error:', err)); // catch connection/publish errors
```

---

## Example 2 — real world backend use case

```js
// File: src/realtime/notificationFanOut.js
// Real pattern: multiple Node.js server instances (behind a load balancer) all
// subscribe to the same channel, so a notification created on ANY instance
// reaches a user's WebSocket connection no matter WHICH instance they're on.

const { createClient } = require('redis');

const NOTIFICATIONS_CHANNEL = 'user-notifications';         // one shared channel name

class NotificationFanOut {
  constructor(redisUrl) {
    this.redisUrl = redisUrl;                                // store connection string
    this.publisherClient = null;                             // filled in during connect()
    this.subscriberClient = null;                            // separate client, per the rule above
    this.localSocketsByUserId = new Map();                   // userId -> this instance's live sockets
  }

  // Call once at server startup
  async connect() {
    this.publisherClient = createClient({ url: this.redisUrl });
    await this.publisherClient.connect();                    // connection for publish()

    this.subscriberClient = createClient({ url: this.redisUrl });
    await this.subscriberClient.connect();                    // connection for subscribe()

    // Every server instance runs this same subscription
    await this.subscriberClient.subscribe(NOTIFICATIONS_CHANNEL, (rawMessage) => {
      this.deliverIfLocal(rawMessage);                        // handle each incoming message
    });
  }

  // Called by a WebSocket connection handler when a user connects to THIS instance
  registerSocket(userId, socket) {
    this.localSocketsByUserId.set(userId, socket);            // track locally, no Redis needed
  }

  registerDisconnect(userId) {
    this.localSocketsByUserId.delete(userId);                 // stop tracking on disconnect
  }

  // Any part of the app calls this to notify a user — it doesn't know or care
  // which server instance that user is actually connected to
  async notifyUser(userId, notification) {
    const payload = JSON.stringify({ userId, notification }); // must publish a string
    await this.publisherClient.publish(NOTIFICATIONS_CHANNEL, payload); // fan out to ALL instances
  }

  // Runs on every instance for every message — only the instance holding
  // that user's live socket actually does anything with it
  deliverIfLocal(rawMessage) {
    const { userId, notification } = JSON.parse(rawMessage);  // rebuild the object
    const socket = this.localSocketsByUserId.get(userId);      // is the user connected HERE?

    if (!socket) return;                                       // not our user — ignore silently

    socket.send(JSON.stringify(notification));                 // push it down the live WebSocket
  }
}

module.exports = { NotificationFanOut };

// Usage in server.js:
// const fanOut = new NotificationFanOut(process.env.REDIS_URL);
// await fanOut.connect();
// wsServer.on('connection', (socket, userId) => fanOut.registerSocket(userId, socket));
// await fanOut.notifyUser('user_42', { type: 'comment', text: 'New reply on your post' });
```

---

## Common mistakes

### Mistake 1 — Reusing one Redis client for both publishing and subscribing

```js
// ❌ WRONG — once .subscribe() is called, this client is locked into subscriber
// mode and any other command on it will throw or hang
const client = createClient();
await client.connect();
await client.subscribe('order-events', (msg) => console.log(msg));
await client.publish('order-events', 'test');   // throws — client is in subscriber mode

// ✅ CORRECT — always use two separate client connections
const subscriberClient = createClient();
await subscriberClient.connect();
await subscriberClient.subscribe('order-events', (msg) => console.log(msg));

const publisherClient = createClient();          // second, independent connection
await publisherClient.connect();
await publisherClient.publish('order-events', 'test');   // works fine
```

### Mistake 2 — Treating Pub/Sub as a reliable queue

```js
// ❌ WRONG — assuming the message is saved if no one is listening yet
// If the subscriber connects AFTER this runs, the message is gone forever —
// Redis Pub/Sub has zero persistence and zero delivery guarantee
await publisherClient.publish('payment-completed', JSON.stringify({ orderId: 'ord_9' }));
// ... server restarts, subscriber reconnects 3 seconds later ... message is LOST

// ✅ CORRECT — if you need guaranteed delivery, use a real queue instead
// (BullMQ / Redis Streams / RabbitMQ — see Topic 100 and Topic 103), or make
// sure subscribers are already connected and healthy before anything publishes
// Pub/Sub is for "fire and forget" fan-out, not for critical business events
```

### Mistake 3 — Forgetting to JSON.stringify / JSON.parse the payload

```js
// ❌ WRONG — publishing a raw object directly
const orderData = { orderId: 'ord_501', status: 'shipped' };
await publisherClient.publish('order-events', orderData);
// TypeError: the redis client expects a string, not an object — crashes or
// silently sends "[object Object]" depending on the client version

// ✅ CORRECT — always stringify before publishing, parse after receiving
await publisherClient.publish('order-events', JSON.stringify(orderData));

await subscriberClient.subscribe('order-events', (rawMessage) => {
  const orderData = JSON.parse(rawMessage);       // convert back to a usable object
  console.log(orderData.orderId, orderData.status);
});
```

---

## Practice exercises

### Exercise 1 — easy

Using the `redis` package, write a small script that:
1. Creates and connects two separate Redis clients — one for publishing, one for subscribing
2. Subscribes to a channel named `'system-alerts'`
3. Publishes three different string messages to that channel, one second apart (`setTimeout` or `setInterval`)
4. Logs each received message to the console along with a timestamp of when it arrived
5. Cleanly disconnects both clients after all three messages have been received

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `RoomBroadcaster` class for a simple chat-room backend that:
1. Takes a `redisUrl` in its constructor and exposes an async `connect()` method that sets up both a publisher and subscriber client
2. Has a method `joinRoom(roomId, onMessage)` that pattern-subscribes to a channel scheme like `chat-room:{roomId}` and calls `onMessage(messageData)` whenever something arrives in that room
3. Has a method `sendMessage(roomId, senderId, text)` that publishes a JSON payload `{ senderId, text, sentAt }` to that room's channel
4. Has a method `leaveRoom(roomId)` that unsubscribes from that room's channel
5. Test it by joining `'room-1'` from two separate `RoomBroadcaster` instances (simulating two connected users), sending a message from one, and confirming both receive it (since Redis fan-out doesn't distinguish sender from receiver, decide how your real app would avoid echoing a message back to its own sender)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `PresenceTracker` system across multiple simulated "server instances" (just multiple Node.js processes or async functions in one file, each with its own Redis clients) that:
1. Uses Redis Pub/Sub to broadcast `user-online` and `user-offline` events on a channel named `'presence-events'`, with payload `{ userId, status, instanceId }`
2. Each simulated instance maintains its OWN local `Map` of `userId -> status`, updated only by consuming the pub/sub channel (never read directly from another instance's memory)
3. Write a function `getGlobalOnlineUsers(instance)` that returns an array of every `userId` that instance believes is currently online, based purely on the events it has received
4. Simulate: instance A marks `user_1` online, instance B marks `user_2` online, instance A marks `user_1` offline — after all events settle, call `getGlobalOnlineUsers()` on instance B and confirm it correctly shows only `user_2` as online (proving instance B learned about `user_1`'s full online-then-offline lifecycle purely through Pub/Sub, never talking to instance A directly)
5. Add a `pSubscribe` pattern version so a *new* instance C — which joins AFTER the events already happened — correctly explains why it will show zero online users (write this as a code comment; this demonstrates Pub/Sub's lack of history, and is the reason production presence systems back this with a Redis key/TTL as well, not Pub/Sub alone)

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CONCEPT
  Publisher  → sends a message to a named CHANNEL
  Subscriber → listens on that CHANNEL, callback fires per message
  Redis      → just relays; stores NOTHING, guarantees NOTHING

KEY RULE
  A client in subscriber mode CANNOT run other commands on that
  same connection — ALWAYS use two separate clients:
    publisherClient  → for .publish()
    subscriberClient → for .subscribe() / .pSubscribe()

METHODS (npm package: redis)
  client.publish(channel, message)          → message must be a string
  client.subscribe(channel, callback)       → exact channel name match
  client.pSubscribe(pattern, callback)      → glob pattern, e.g. 'room:*'
  client.unsubscribe(channel)               → stop listening to one channel
  client.pUnsubscribe(pattern)              → stop listening to a pattern

MESSAGE PAYLOADS
  Always JSON.stringify() before publish()
  Always JSON.parse()     after  receiving in the callback

WHAT PUB/SUB IS GOOD FOR
  - Fan-out across multiple server instances (Socket.IO scaling, Topic 99)
  - Cache invalidation broadcasts
  - Live dashboards / presence pings
  - Decoupled "shout an event" architecture

WHAT PUB/SUB IS *NOT* GOOD FOR
  - Guaranteed delivery      → use BullMQ (Topic 100) or Streams
  - Message persistence      → nothing is saved, ever
  - Late subscribers         → they miss everything published before they joined
  - Work distribution        → EVERY subscriber gets EVERY message (not load-balanced)

MENTAL MODEL
  Radio station (publisher) broadcasts on a frequency (channel).
  Anyone tuned in (subscriber) hears it live. Turn the radio on late,
  you missed the broadcast — there's no replay.
```

---

## Connected topics

- **99 — Socket.IO in depth** — the Redis adapter for Socket.IO uses this exact Pub/Sub mechanism to fan messages out across multiple server instances.
- **100 — Message queues with BullMQ** — when you need guaranteed delivery and retries instead of fire-and-forget broadcasting, BullMQ (built on Redis) is the right tool.
- **92 — Connecting to Redis** — the foundational client setup (ioredis/redis, connection basics) that this topic builds directly on top of.
