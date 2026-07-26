# 104 — Webhooks — sending and receiving

## What is this?

A webhook is a **reverse API call** — instead of your app repeatedly asking another system "did anything happen yet?" (polling), that system sends *you* an HTTP POST the moment something happens. Think of it like giving your phone number to a restaurant instead of calling every five minutes to ask if your table is ready — they call you when it's time. In backend systems, webhooks flow in both directions: you **receive** webhooks from third parties (Stripe tells you "payment succeeded"), and you **send** webhooks to your own customers' systems (your SaaS tells their server "order shipped").

## Why does it matter for backend development?

Almost every payment gateway, messaging provider, and SaaS platform (Stripe, GitHub, Twilio, Slack, Shopify) delivers real-time events exclusively through webhooks — a backend dev must know how to receive them safely, which means **verifying the sender is who they claim to be** using an HMAC signature, not just trusting the payload. On the flip side, if you build a platform that other developers integrate with, you will need to **send** outbound webhooks reliably — which means signing your payloads, and retrying deliveries with backoff when the receiver's server is briefly down or returns an error. Getting either side wrong causes either forged fake events (security breach) or silently dropped events (data loss, angry customers).

---

## Syntax / API

```js
// crypto is Node's built-in module for hashing and HMAC — no install needed
const crypto = require('crypto');

// ── Signing an outbound payload ─────────────────────────────────────────────
function signPayload(requestBody, webhookSecret) {
  // requestBody must be the EXACT raw string bytes you will actually send
  return crypto
    .createHmac('sha256', webhookSecret)  // HMAC keyed with the shared secret
    .update(requestBody)                  // feed in the raw payload bytes
    .digest('hex');                       // output signature as a hex string
}

// ── Verifying an incoming payload ────────────────────────────────────────────
function verifySignature(requestBody, receivedSignature, webhookSecret) {
  // Recompute the signature ourselves using the SAME secret and SAME raw body
  const expectedSignature = signPayload(requestBody, webhookSecret);

  // Convert both to buffers of equal length before comparing
  const expected = Buffer.from(expectedSignature, 'utf8');
  const received = Buffer.from(receivedSignature, 'utf8');

  // Buffers of different length would throw inside timingSafeEqual — guard first
  if (expected.length !== received.length) return false;

  // timingSafeEqual compares in constant time — prevents timing-attack leakage
  return crypto.timingSafeEqual(expected, received);
}

// ── Sending with retry (the shape every outbound webhook sender needs) ──────
async function sendWebhookWithRetry(url, requestBody, webhookSecret, maxAttempts = 5) {
  const signature = signPayload(requestBody, webhookSecret); // sign once, reuse

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,   // receiver verifies this header
        },
        body: requestBody,
      });

      if (response.ok) return { delivered: true, attempt };  // 2xx = success, stop retrying

      throw new Error(`Receiver returned ${response.status}`); // 4xx/5xx = retry
    } catch (err) {
      if (attempt === maxAttempts) return { delivered: false, error: err.message };

      const delayMs = 2 ** attempt * 1000;          // exponential backoff: 2s, 4s, 8s, 16s...
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}
```

---

## How it works — line by line

**Signing** — `createHmac('sha256', webhookSecret)` creates a keyed hash function. Unlike a plain hash (`createHash`), HMAC requires a secret key, so only someone who knows the secret can produce a matching signature — this proves the payload really came from the sender and was not tampered with in transit.

**The raw body matters** — the signature is computed over the exact bytes of the request body, not a parsed-and-re-serialized JavaScript object. If you `JSON.parse()` the body and later `JSON.stringify()` it back to check the signature, key order or whitespace can change and the signature will no longer match, even though nothing malicious happened.

**Verifying** — the receiver does not "decrypt" the signature (HMAC is one-way). Instead it recomputes its own signature from the raw body it received plus the shared secret, then compares the two signatures. If they match, the body was not altered and the sender knows the secret.

**Constant-time comparison** — `crypto.timingSafeEqual` compares byte-by-byte in a fixed amount of time regardless of where a mismatch occurs. A normal `===` string comparison can return slightly faster the earlier it finds a mismatched character, which — over thousands of requests — leaks enough timing information for an attacker to guess the correct signature one byte at a time. This is why you always use `timingSafeEqual` for secrets, never `===`.

**Retry with backoff** — outbound webhook receivers are often briefly unavailable (deploys, restarts, transient network errors). Retrying immediately in a tight loop just floods a struggling server. Exponential backoff (`2 ** attempt * 1000`) spaces retries further apart each time, giving the receiver room to recover, and `maxAttempts` guarantees you eventually stop instead of retrying forever.

---

## Example 1 — basic

```js
// File: examples/webhook-basic.js
// Minimal end-to-end demo: sign a payload, verify it, detect tampering.

const crypto = require('crypto');

const webhookSecret = 'whsec_test_12345';           // shared secret, never sent over the wire

// The event we want to notify a receiver about
const eventPayload = {
  eventId: 'evt_8f2a',
  eventType: 'order.created',
  orderId: 'order_991',
  amount: 4999,
};

// The signature MUST be computed on the exact string that will be sent
const requestBody = JSON.stringify(eventPayload);   // serialize once, sign this exact string

// Sign it — this is what a sender does before dispatching the webhook
const signature = crypto
  .createHmac('sha256', webhookSecret)              // key the hash with the secret
  .update(requestBody)                              // hash the raw JSON string
  .digest('hex');                                   // hex signature to send in a header

console.log('Sending payload  :', requestBody);
console.log('X-Webhook-Signature:', signature);

// ── Receiver side ────────────────────────────────────────────────────────────
function verify(receivedBody, receivedSignature) {
  const expectedSignature = crypto
    .createHmac('sha256', webhookSecret)            // same secret, same algorithm
    .update(receivedBody)                           // same raw bytes the sender hashed
    .digest('hex');

  const expected = Buffer.from(expectedSignature, 'utf8');
  const received = Buffer.from(receivedSignature, 'utf8');

  if (expected.length !== received.length) return false;  // different lengths can't match
  return crypto.timingSafeEqual(expected, received);       // safe comparison
}

console.log('Valid signature? ', verify(requestBody, signature));            // → true

// Simulate an attacker tampering with the amount after the fact
const tamperedBody = requestBody.replace('4999', '1');
console.log('Tampered valid?  ', verify(tamperedBody, signature));           // → false
```

---

## Example 2 — real world backend use case

```js
// File: src/webhooks/dispatcher.js
// A SaaS backend notifying customer-registered URLs when events happen,
// plus an Express endpoint receiving webhooks from a payment provider.

const crypto = require('crypto');
const express = require('express');

// ── OUTBOUND: sending webhooks to customers with retry ──────────────────────

class WebhookDispatcher {
  constructor(deliveryLog) {
    this.deliveryLog = deliveryLog;   // e.g. a DB table logging every attempt
  }

  sign(requestBody, webhookSecret) {
    return crypto.createHmac('sha256', webhookSecret).update(requestBody).digest('hex');
  }

  // Called whenever a domain event happens (order shipped, invoice paid, etc.)
  async dispatch({ url, webhookSecret, eventType, eventData }) {
    const requestBody = JSON.stringify({
      eventId: crypto.randomUUID(),        // unique id lets receivers dedupe
      eventType,                            // e.g. 'invoice.paid'
      createdAt: new Date().toISOString(),  // when the event occurred
      data: eventData,                      // the actual event payload
    });

    const signature = this.sign(requestBody, webhookSecret);
    const maxAttempts = 5;

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        const response = await fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': signature,
            'X-Webhook-Attempt': String(attempt),
          },
          body: requestBody,
          signal: AbortSignal.timeout(5000),   // don't hang forever on a dead receiver
        });

        await this.deliveryLog.record({ url, attempt, status: response.status });

        if (response.ok) return { delivered: true, attempt };  // stop on first 2xx
      } catch (err) {
        await this.deliveryLog.record({ url, attempt, status: 'error', error: err.message });
      }

      if (attempt < maxAttempts) {
        const delayMs = 2 ** attempt * 1000;               // 2s, 4s, 8s, 16s
        await new Promise((resolve) => setTimeout(resolve, delayMs));
      }
    }

    return { delivered: false };   // exhausted retries — caller can alert or dead-letter
  }
}

// ── INBOUND: receiving and verifying webhooks (Stripe-style pattern) ────────

const app = express();

// express.raw() keeps the body as a Buffer — required so the signature check
// runs against the EXACT bytes the sender signed, before any JSON parsing
app.post(
  '/webhooks/payments',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const receivedSignature = req.get('X-Webhook-Signature');
    const paymentWebhookSecret = process.env.PAYMENT_WEBHOOK_SECRET; // never hardcode

    const expectedSignature = crypto
      .createHmac('sha256', paymentWebhookSecret)
      .update(req.body)              // req.body is the raw Buffer here, not parsed JSON
      .digest('hex');

    const expected = Buffer.from(expectedSignature, 'utf8');
    const received = Buffer.from(receivedSignature || '', 'utf8');

    const isValid =
      expected.length === received.length && crypto.timingSafeEqual(expected, received);

    if (!isValid) {
      return res.status(401).json({ error: 'invalid signature' }); // reject forged requests
    }

    // Only NOW is it safe to parse and trust the payload
    const event = JSON.parse(req.body.toString('utf8'));

    // Respond fast — process the event asynchronously so the sender doesn't retry
    res.status(200).json({ received: true });

    processPaymentEvent(event).catch((err) => {
      console.error(`[webhook] failed processing ${event.eventId}:`, err.message);
    });
  }
);

async function processPaymentEvent(event) {
  // Business logic goes here — update order status, send receipt email, etc.
  console.log(`[webhook] processing ${event.eventType} for ${event.data?.orderId}`);
}

module.exports = { WebhookDispatcher, app };
```

---

## Common mistakes

### Mistake 1 — Verifying the signature against re-parsed JSON instead of raw bytes

```js
// ❌ WRONG — express.json() has already parsed and re-serialized the body,
// so JSON.stringify(req.body) may not byte-match what the sender signed
app.use(express.json());
app.post('/webhooks/payments', (req, res) => {
  const requestBody = JSON.stringify(req.body);   // key order/whitespace may differ!
  const expected = signPayload(requestBody, webhookSecret);
  if (expected !== req.get('X-Webhook-Signature')) {
    return res.status(401).end();                 // valid webhooks get rejected here
  }
});

// ✅ CORRECT — capture the raw body BEFORE any JSON parsing happens
app.post(
  '/webhooks/payments',
  express.raw({ type: 'application/json' }),      // req.body is now a raw Buffer
  (req, res) => {
    const expected = signPayload(req.body, webhookSecret); // hash the exact bytes received
    // ...verify, then JSON.parse(req.body) only after verification passes
  }
);
```

### Mistake 2 — Comparing signatures with `===` instead of a constant-time compare

```js
// ❌ WRONG — string equality short-circuits on the first mismatched character,
// leaking timing information an attacker can use to guess the signature
if (expectedSignature === receivedSignature) {
  // process the event
}

// ✅ CORRECT — timingSafeEqual takes the same amount of time regardless of
// where the buffers differ, so no timing information leaks
const expected = Buffer.from(expectedSignature, 'utf8');
const received = Buffer.from(receivedSignature, 'utf8');
const isValid =
  expected.length === received.length && crypto.timingSafeEqual(expected, received);
```

### Mistake 3 — Sending outbound webhooks with no retry, or retrying forever with no backoff

```js
// ❌ WRONG — a single failed fetch means the event is lost forever, even though
// the receiver's server might just be redeploying for two seconds
async function sendWebhook(url, requestBody) {
  await fetch(url, { method: 'POST', body: requestBody }); // no try/catch, no retry
}

// ❌ ALSO WRONG — retrying instantly in a loop hammers a struggling server
while (true) {
  const response = await fetch(url, { method: 'POST', body: requestBody });
  if (response.ok) break;   // no delay, no max attempts — can loop forever
}

// ✅ CORRECT — bounded retries with exponential backoff, then give up cleanly
async function sendWebhook(url, requestBody, maxAttempts = 5) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const response = await fetch(url, { method: 'POST', body: requestBody });
      if (response.ok) return true;
    } catch (err) {
      // network error — fall through to retry
    }
    if (attempt < maxAttempts) {
      await new Promise((r) => setTimeout(r, 2 ** attempt * 1000)); // 2s, 4s, 8s...
    }
  }
  return false;   // log this, alert on it, or write to a dead-letter store
}
```

---

## Practice exercises

### Exercise 1 — easy

Write two functions: `signPayload(requestBody, webhookSecret)` that returns a hex-encoded HMAC-SHA256 signature of `requestBody`, and `verifySignature(requestBody, receivedSignature, webhookSecret)` that returns `true` or `false` using `crypto.timingSafeEqual`. Test it with a sample payload: sign it, verify it succeeds, then change one character in the payload and verify the check now returns `false`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an async function `sendWebhookWithRetry(url, requestBody, webhookSecret, maxAttempts)` that signs the payload, sends it via `fetch` as a POST with an `X-Webhook-Signature` header, and retries with exponential backoff (`2 ** attempt * 1000` ms) on any network error or non-2xx response. It should stop retrying as soon as a 2xx response is received, give up after `maxAttempts`, and return an object like `{ delivered: true, attempt: 2 }` or `{ delivered: false }`. Test it against a URL that always fails (e.g. `http://localhost:9999/nowhere`) and confirm it retries the expected number of times before giving up.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a complete webhook receiver as an Express app with the following behavior:
1. A `POST /webhooks/events` route using `express.raw({ type: 'application/json' })` so the raw body is available for signature verification.
2. A signature verification step using HMAC-SHA256 and `crypto.timingSafeEqual` — reject with `401` if it fails.
3. **Replay protection** — the sender includes an `X-Webhook-Timestamp` header (Unix seconds); reject the request with `401` if that timestamp is more than 5 minutes old or more than 1 minute in the future (sign the string `${timestamp}.${requestBody}` instead of the raw body alone, matching how Stripe does it).
4. **Idempotency** — each event payload includes an `eventId`; keep an in-memory `Set` of processed event IDs and skip re-processing (but still respond `200`) if the same `eventId` arrives twice.
5. Respond `200` immediately after verification passes, then process the event asynchronously (simulate with a `setTimeout` or a fake async DB write) so the sender isn't kept waiting.

Test it by writing a small client script that signs and sends a valid event, resends the exact same event (should be skipped), and sends an event with a stale timestamp (should be rejected with `401`).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE IDEA
  Webhook = reverse API call — the other system POSTs to YOUR url when an event happens
  You SEND webhooks to integrators, and RECEIVE webhooks from providers (Stripe, GitHub...)

SIGNING (outbound)
  crypto.createHmac('sha256', webhookSecret).update(requestBody).digest('hex')
  → sign the EXACT raw bytes you are about to send, before any transformation

VERIFYING (inbound)
  1. Capture the RAW body (express.raw(), NOT express.json())
  2. Recompute the HMAC yourself with the shared secret
  3. Compare with crypto.timingSafeEqual — NEVER with === (timing attack risk)
  4. Only JSON.parse() the body AFTER verification passes

COMMON HEADERS SENT WITH A WEBHOOK
  X-Webhook-Signature   → the HMAC hex/base64 signature
  X-Webhook-Timestamp   → Unix time signed alongside the body (replay protection)
  X-Webhook-Attempt     → which retry attempt this delivery is
  X-Webhook-Event-Id    → unique id for idempotency / dedup on the receiver side

OUTBOUND RETRY PATTERN
  - Retry on network errors and non-2xx responses
  - Exponential backoff: 2s, 4s, 8s, 16s, 32s...
  - Cap maxAttempts, then log/alert/dead-letter — never retry forever
  - Set a request timeout (AbortSignal.timeout) so a hung receiver can't block you

RECEIVER SAFETY CHECKLIST
  ✓ Verify signature before trusting anything in the payload
  ✓ Reject stale timestamps (replay protection window, e.g. 5 minutes)
  ✓ Dedupe by eventId (idempotency — senders WILL retry and send duplicates)
  ✓ Respond 200 fast, do slow processing asynchronously after responding
  ✓ Never log or leak the webhook secret

NEVER DO
  JSON.stringify(req.body) to re-derive the signed string  → breaks on key order
  signature === receivedSignature                          → timing attack
  await fetch(url, ...) with no try/catch and no retry      → silently lost events
  Hardcoding the webhook secret in source code               → use process.env
```

---

## Connected topics

- **23 — crypto module** — `createHmac`, `randomBytes`, `timingSafeEqual` are the exact primitives webhook signing and verification are built on.
- **51 — Retry and timeout patterns** — the exponential backoff and `AbortSignal.timeout` used for outbound webhook delivery are general async retry patterns applied here.
- **100 — Message queues — BullMQ** — production webhook senders usually queue deliveries as jobs (rather than sending inline) so retries, backoff, and delivery logs survive process restarts.
