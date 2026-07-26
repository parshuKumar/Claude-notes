# 103 — Working with RabbitMQ

## What is this?

RabbitMQ is a **message broker** — a standalone server that sits between your applications and holds messages until they are safely processed. One service publishes a message into RabbitMQ, and RabbitMQ delivers it to one or more consumers, keeping it safe even if a consumer crashes mid-processing. Think of it like a post office: instead of you personally handing a letter to the recipient (and losing it if they're not home), you drop it in a mailbox, and the post office guarantees it will be delivered — retrying, holding, and routing it correctly even if the recipient is temporarily unavailable.

`amqplib` is the Node.js client library used to talk to RabbitMQ using the AMQP 0-9-1 protocol — the language RabbitMQ speaks.

## Why does it matter for backend development?

Backend systems often need to do work that shouldn't block the HTTP response — sending emails, generating PDFs, processing payments, resizing images, syncing data across microservices. Instead of doing this work inline (slow, fragile, and hard to scale), you publish a message describing the job to RabbitMQ and let separate worker processes consume it independently. This decouples services — the order service doesn't need to know anything about the email service, it just publishes an "order placed" message. RabbitMQ also gives you routing (send different messages to different consumers based on rules), retries, and guaranteed delivery via acknowledgements — features that make it a backbone of production microservice architectures, alongside tools like BullMQ and Kafka.

---

## Syntax / API

```js
// Install first: npm install amqplib
const amqp = require('amqplib');

// ── Connecting ───────────────────────────────────────────────────────────────
async function connectToBroker() {
  // connect() opens a TCP connection to the RabbitMQ server
  const dbConnection = await amqp.connect('amqp://localhost'); // amqp:// is the protocol

  // A channel is a lightweight virtual connection inside the TCP connection
  // Almost all operations (publish, consume, ack) happen on a channel, not the connection
  const channel = await dbConnection.createChannel();

  return { dbConnection, channel };
}

// ── Queues ───────────────────────────────────────────────────────────────────
// assertQueue creates the queue if it doesn't exist, or confirms it does
await channel.assertQueue('emailQueue', { durable: true }); // durable = survives broker restart

// ── Publishing directly to a queue (no exchange routing) ───────────────────
channel.sendToQueue(
  'emailQueue',                                  // target queue name
  Buffer.from(JSON.stringify({ to: 'user@example.com' })), // payload must be a Buffer
  { persistent: true }                           // persistent = message survives broker restart
);

// ── Exchanges ────────────────────────────────────────────────────────────────
// An exchange receives messages and routes them to queues based on rules
await channel.assertExchange('orderEvents', 'direct', { durable: true });

// ── Binding a queue to an exchange with a routing key ───────────────────────
await channel.bindQueue('shippingQueue', 'orderEvents', 'order.created');

// ── Publishing through an exchange ──────────────────────────────────────────
channel.publish(
  'orderEvents',                                 // exchange name
  'order.created',                               // routing key — decides which queue(s) get it
  Buffer.from(JSON.stringify({ orderId: 'ord_1' })),
  { persistent: true }
);

// ── Consuming and acknowledging ──────────────────────────────────────────────
channel.consume('emailQueue', (msg) => {
  const payload = JSON.parse(msg.content.toString()); // decode the Buffer back to an object
  console.log('Processing:', payload);
  channel.ack(msg); // tell RabbitMQ "I'm done, you can delete this message"
});
```

---

## How it works — line by line

A **connection** is the actual network link between your Node process and the RabbitMQ server — it's expensive to open, so you make one per application.

A **channel** is a lightweight lane inside that connection — you can open many channels on one connection, and almost every RabbitMQ operation (declaring queues, publishing, consuming) happens through a channel, not the raw connection.

A **queue** is a named buffer that stores messages in order until a consumer takes them. `assertQueue` is idempotent — calling it repeatedly with the same name and options is safe; it creates the queue only if it's missing.

An **exchange** is a routing agent — it never stores messages itself. When you publish to an exchange, the exchange looks at the message's **routing key** and its own type (`direct`, `topic`, `fanout`, `headers`) to decide which bound queues should receive a copy.

A **binding** connects a queue to an exchange using a routing key — `bindQueue('shippingQueue', 'orderEvents', 'order.created')` means "any message published to the `orderEvents` exchange with the exact routing key `order.created` should be copied into `shippingQueue`."

**Acknowledgement (`ack`)** is how a consumer tells RabbitMQ "I successfully processed this message, you can remove it from the queue." If the consumer crashes before calling `ack`, RabbitMQ automatically re-queues the message and delivers it to another consumer — this is what makes RabbitMQ reliable instead of "fire and forget."

---

## Example 1 — basic

```js
// File: src/queue/basic-example.js
// Sends one message into a queue, then a separate consumer reads it back.

const amqp = require('amqplib');

async function sendMessage() {
  // Connect to the local RabbitMQ broker (default port 5672)
  const dbConnection = await amqp.connect('amqp://localhost');

  // Open a channel to perform operations on
  const channel = await dbConnection.createChannel();

  const queueName = 'taskQueue'; // name of the queue we'll use

  // Make sure the queue exists before we use it — durable survives broker restarts
  await channel.assertQueue(queueName, { durable: true });

  // Build the message payload as a plain object, then serialize to JSON
  const requestBody = { task: 'resize-image', filePath: '/uploads/photo.jpg' };

  // sendToQueue requires a Buffer, not a plain string or object
  channel.sendToQueue(
    queueName,
    Buffer.from(JSON.stringify(requestBody)),
    { persistent: true } // message is written to disk, not just kept in memory
  );

  console.log('Sent:', requestBody);

  // Close the channel and connection after sending (short-lived script)
  await channel.close();
  await dbConnection.close();
}

async function receiveMessage() {
  const dbConnection = await amqp.connect('amqp://localhost');
  const channel = await dbConnection.createChannel();
  const queueName = 'taskQueue';

  await channel.assertQueue(queueName, { durable: true });

  console.log('Waiting for messages...');

  // consume() registers a callback that fires every time a message arrives
  channel.consume(queueName, (msg) => {
    if (msg === null) return; // consumer was cancelled, nothing to process

    const requestBody = JSON.parse(msg.content.toString()); // Buffer → string → object
    console.log('Received:', requestBody);

    channel.ack(msg); // confirm processing so RabbitMQ removes it from the queue
  });
}

sendMessage(); // run the producer
receiveMessage(); // run the consumer (in real life, this runs in a separate process)
```

---

## Example 2 — real world backend use case

```js
// File: src/queue/order-events.js
// A realistic pattern: an "order service" publishes an event through a direct
// exchange, and two independent worker services (email + shipping) each get
// their own queue bound with different routing keys.

const amqp = require('amqplib');

const EXCHANGE_NAME = 'orderEvents'; // exchange all order-related events go through

// ── Producer: called from the order-placement endpoint ─────────────────────
async function publishOrderCreated(orderId, userId) {
  const dbConnection = await amqp.connect(process.env.RABBITMQ_URL || 'amqp://localhost');
  const channel = await dbConnection.createChannel();

  // A "direct" exchange routes messages to queues based on an exact routing key match
  await channel.assertExchange(EXCHANGE_NAME, 'direct', { durable: true });

  const requestBody = { orderId, userId, createdAt: new Date().toISOString() };

  // Publish once — RabbitMQ fans it out to every queue bound to 'order.created'
  channel.publish(
    EXCHANGE_NAME,
    'order.created', // routing key describing what happened
    Buffer.from(JSON.stringify(requestBody)),
    { persistent: true, contentType: 'application/json' }
  );

  console.log(`[order-service] published order.created for ${orderId}`);

  await channel.close();
  await dbConnection.close();
}

// ── Consumer 1: email worker, its own dedicated queue ───────────────────────
async function startEmailWorker() {
  const dbConnection = await amqp.connect(process.env.RABBITMQ_URL || 'amqp://localhost');
  const channel = await dbConnection.createChannel();

  await channel.assertExchange(EXCHANGE_NAME, 'direct', { durable: true });
  await channel.assertQueue('emailNotificationQueue', { durable: true });
  await channel.bindQueue('emailNotificationQueue', EXCHANGE_NAME, 'order.created');

  // prefetch(1) means: don't send this consumer a new message until it acks the current one
  // This prevents one slow worker from being overloaded with a huge backlog
  channel.prefetch(1);

  channel.consume('emailNotificationQueue', async (msg) => {
    const { orderId, userId } = JSON.parse(msg.content.toString());
    try {
      console.log(`[email-worker] sending confirmation email for order ${orderId} to user ${userId}`);
      // await sendConfirmationEmail(userId, orderId);  // real email logic here
      channel.ack(msg); // success — remove message from the queue permanently
    } catch (error) {
      console.error('[email-worker] failed:', error.message);
      channel.nack(msg, false, true); // failure — requeue so another attempt happens
    }
  });
}

// ── Consumer 2: shipping worker, a completely separate queue ────────────────
async function startShippingWorker() {
  const dbConnection = await amqp.connect(process.env.RABBITMQ_URL || 'amqp://localhost');
  const channel = await dbConnection.createChannel();

  await channel.assertExchange(EXCHANGE_NAME, 'direct', { durable: true });
  await channel.assertQueue('shippingQueue', { durable: true });
  await channel.bindQueue('shippingQueue', EXCHANGE_NAME, 'order.created');

  channel.prefetch(1);

  channel.consume('shippingQueue', async (msg) => {
    const { orderId } = JSON.parse(msg.content.toString());
    console.log(`[shipping-worker] scheduling shipment for order ${orderId}`);
    channel.ack(msg);
  });
}

module.exports = { publishOrderCreated, startEmailWorker, startShippingWorker };

// Usage:
// In the HTTP order-creation route: await publishOrderCreated(order.id, order.userId);
// In two separate worker processes: startEmailWorker(); / startShippingWorker();
// Both workers receive their own copy of every order.created event, independently.
```

---

## Common mistakes

### Mistake 1 — Forgetting to acknowledge messages

```js
// ❌ WRONG — never calling ack() means RabbitMQ thinks the message is still
// "in flight" forever; on consumer restart, it gets redelivered and reprocessed
channel.consume('emailQueue', (msg) => {
  const payload = JSON.parse(msg.content.toString());
  sendEmail(payload); // even if this succeeds, the message is never marked done
  // missing: channel.ack(msg)
});

// ✅ CORRECT — always ack after successful processing (or nack on failure)
channel.consume('emailQueue', async (msg) => {
  try {
    const payload = JSON.parse(msg.content.toString());
    await sendEmail(payload);
    channel.ack(msg); // tells RabbitMQ this message is fully handled
  } catch (error) {
    channel.nack(msg, false, true); // requeue on failure instead of losing the error
  }
});
```

### Mistake 2 — Publishing to a queue that doesn't exist yet (or with mismatched options)

```js
// ❌ WRONG — sendToQueue silently does nothing useful if the queue was never
// declared, or throws a channel error if it exists with DIFFERENT options
// (e.g. it was created elsewhere as durable: false)
channel.sendToQueue('reportQueue', Buffer.from('generate report'));

// ✅ CORRECT — always assertQueue with the SAME options every producer and
// consumer uses, before publishing or consuming
await channel.assertQueue('reportQueue', { durable: true });
channel.sendToQueue('reportQueue', Buffer.from('generate report'), { persistent: true });
```

### Mistake 3 — Sending a plain object or string instead of a Buffer

```js
// ❌ WRONG — amqplib requires a Buffer for message content; passing a raw
// object throws "TypeError: content is not a Buffer" at runtime
const requestBody = { userId: 'user_42', action: 'password-reset' };
channel.sendToQueue('authQueue', requestBody); // crashes

// ✅ CORRECT — always JSON.stringify then wrap in Buffer.from()
channel.sendToQueue('authQueue', Buffer.from(JSON.stringify(requestBody)));

// And on the consuming side, always convert back:
channel.consume('authQueue', (msg) => {
  const requestBody = JSON.parse(msg.content.toString()); // Buffer → string → object
  channel.ack(msg);
});
```

---

## Practice exercises

### Exercise 1 — easy

Install `amqplib` and connect to a local RabbitMQ instance (`amqp://localhost` — you can run one quickly with `docker run -d -p 5672:5672 rabbitmq:3-management`). Write a producer script that:
1. Declares a durable queue named `notificationQueue`
2. Sends 3 different messages into it, each a JSON object like `{ userId: 'user_1', message: 'Welcome!' }`
3. Closes the connection after sending

Then write a separate consumer script that connects, declares the same queue, consumes all messages, logs each one, and acknowledges it.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `topic` exchange named `logEvents` that routes messages based on severity. Routing keys will look like `log.error`, `log.warning`, `log.info`.

1. Create a producer function `publishLog(level, message)` that publishes to the `logEvents` exchange with routing key `log.${level}`
2. Create a consumer that binds a queue named `errorLogsQueue` to the exchange using the pattern `log.error` (exact match is fine, or use `log.*` if you want to experiment with topic wildcards)
3. Create a second consumer that binds a queue named `allLogsQueue` using the wildcard pattern `log.#` so it receives every log level
4. Publish a few logs of different levels and verify `errorLogsQueue` only gets errors while `allLogsQueue` gets everything

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a resilient job-processing system for a `videoTranscodeQueue`:

1. Producer: a function `enqueueTranscodeJob(filePath, userId)` that publishes a persistent message containing `{ filePath, userId, attempts: 0 }`
2. Consumer: uses `channel.prefetch(1)` so it only handles one job at a time
3. On processing, simulate a job that randomly fails (`Math.random() < 0.3` throws an error)
4. On failure, instead of infinitely requeuing, read the `attempts` field, increment it, and if `attempts < 3`, republish the message to the SAME queue with the incremented count (then `ack` the original so it isn't duplicated); if `attempts >= 3`, publish it to a separate `deadLetterQueue` instead and log that it was permanently failed
5. On success, `ack` normally and log completion

Test it by enqueueing 5 jobs and observing how failing ones eventually land in `deadLetterQueue` after 3 attempts.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CONCEPTS
  Connection   → the TCP link to the RabbitMQ server (make one per app)
  Channel      → lightweight virtual connection; all real operations happen here
  Queue        → named buffer that stores messages until consumed
  Exchange     → routes incoming messages to queues; never stores messages itself
  Routing key  → a string attached to a message used by exchanges to decide routing
  Binding      → the link between an exchange and a queue, keyed by routing key pattern

EXCHANGE TYPES
  direct   → exact routing key match (e.g. "order.created" → "order.created")
  topic    → wildcard patterns: * = one word, # = zero or more words (e.g. "log.#")
  fanout   → broadcasts to ALL bound queues, ignores routing key entirely
  headers  → routes based on message header values instead of routing key

SETUP
  const amqp = require('amqplib');
  const dbConnection = await amqp.connect('amqp://localhost');
  const channel = await dbConnection.createChannel();

QUEUES
  await channel.assertQueue(name, { durable: true });   // create/confirm queue
  channel.sendToQueue(name, Buffer.from(data), { persistent: true });

EXCHANGES
  await channel.assertExchange(name, 'direct', { durable: true });
  await channel.bindQueue(queueName, exchangeName, routingKey);
  channel.publish(exchangeName, routingKey, Buffer.from(data), { persistent: true });

CONSUMING
  channel.prefetch(1);                    // limit unacked messages per consumer
  channel.consume(queueName, (msg) => {
    const data = JSON.parse(msg.content.toString());
    channel.ack(msg);                     // success — remove message
    // channel.nack(msg, false, true);    // failure — requeue for retry
    // channel.nack(msg, false, false);   // failure — discard or send to dead-letter
  });

GOTCHAS
  Message content must ALWAYS be a Buffer, never a raw object or string
  assertQueue/assertExchange options must match EXACTLY across all producers/consumers
  Forgetting ack() causes messages to be redelivered forever on reconnect
  durable: true on queue/exchange + persistent: true on message = survives broker restart
  prefetch(1) prevents one worker from hoarding a large backlog of unacked messages
```

---

## Connected topics

- **100 — Message queues (BullMQ)** — a Redis-backed alternative to RabbitMQ with built-in retries, delays, and progress tracking; compare the two approaches to job queues
- **101 — Pub/Sub with Redis** — a simpler fan-out messaging pattern without persistence or acknowledgements, useful to contrast with RabbitMQ's durability guarantees
- **102 — Event-driven architecture basics** — the conceptual foundation (events vs commands, decoupling with events) that RabbitMQ implements at the infrastructure level
