# 23 — crypto module

## What is this?

`crypto` is Node's built-in module for hashing, encrypting, and generating random values — no npm install needed. Think of it as the toolbox a bank uses: a hash is a tamper-evident seal on a document (any change breaks the seal), HMAC is that same seal signed with a secret key so only someone holding the key can verify it, and encryption is a locked safe that only a key can open. Node ships all of this directly, backed by OpenSSL under the hood.

## Why does it matter for backend development?

Every backend developer eventually needs to: hash passwords before storing them, generate a secure random session ID or API key, verify that a webhook payload really came from Stripe/GitHub (not an attacker), or encrypt sensitive data before saving it to a database. Getting any of these wrong is a security incident, not a bug. The `crypto` module is the foundation underneath higher-level libraries like `bcrypt` and `jsonwebtoken` — understanding it directly means you know what those libraries are doing for you, and you can build custom auth flows (like signed webhook verification) without pulling in a dependency.

---

## Syntax / API

```js
// crypto is a Node core module — no npm install required
const crypto = require('crypto');

// ── Hashing — one-way, same input always produces same output ──────────────
const hash = crypto
  .createHash('sha256')          // choose the algorithm: 'sha256', 'md5', 'sha512'
  .update('some data to hash')   // feed in the data (string or Buffer)
  .digest('hex');                // output format: 'hex', 'base64', 'base64url'

// ── HMAC — hashing with a secret key, proves the sender knows the key ──────
const hmac = crypto
  .createHmac('sha256', 'my-secret-key')  // algorithm + secret key
  .update('payload to sign')              // feed in the data
  .digest('hex');                         // output as hex string

// ── Random bytes — cryptographically secure randomness ──────────────────────
const randomToken = crypto.randomBytes(32).toString('hex');
// randomBytes(32) → 32 random bytes (256 bits), .toString('hex') → readable string

// ── Random UUID — a ready-made unique identifier ────────────────────────────
const uniqueId = crypto.randomUUID();
// → e.g. '3f9a1c2e-4b5d-4a6f-8e7c-1d2b3a4c5e6f' — no library needed since Node 14.17+

// ── Symmetric encryption — encrypt data you need to read back later ────────
const algorithm  = 'aes-256-cbc';           // AES with 256-bit key, CBC mode
const secretKey  = crypto.randomBytes(32);  // 32-byte key (256 bits) — store this safely
const iv         = crypto.randomBytes(16);  // initialization vector — must be random per encryption

const cipher = crypto.createCipheriv(algorithm, secretKey, iv); // create the encryptor
let encrypted = cipher.update('sensitive data', 'utf8', 'hex');  // encrypt the text
encrypted += cipher.final('hex');                                // flush remaining output

const decipher = crypto.createDecipheriv(algorithm, secretKey, iv); // create the decryptor
let decrypted = decipher.update(encrypted, 'hex', 'utf8');           // decrypt back
decrypted += decipher.final('utf8');                                 // flush remaining output
```

---

## How it works — line by line

- `createHash(algorithm)` starts a one-way hashing process. You feed data in with `.update()`, and `.digest(format)` produces the final fixed-length fingerprint. The same input always gives the same output, but you cannot reverse a hash back into the original data.
- `createHmac(algorithm, secretKey)` works like `createHash`, but it mixes in a secret key during hashing. This means the resulting signature can only be reproduced by someone who also knows the secret key — this is exactly how webhook signature verification works (Stripe, GitHub, etc. all use HMAC).
- `randomBytes(size)` asks the operating system's secure random number generator for `size` random bytes. This is NOT the same as `Math.random()` — `Math.random()` is predictable and must never be used for security-sensitive values like tokens or session IDs.
- `randomUUID()` is a convenience function that generates a version-4 UUID (a globally unique 36-character ID) using the same secure randomness internally. It replaced the need for the `uuid` npm package for most use cases.
- `createCipheriv(algorithm, key, iv)` sets up a two-way encryption stream. The "iv" (initialization vector) is a random value that ensures encrypting the same text twice produces different output each time — critical for security. `.update()` encrypts a chunk, `.final()` flushes any remaining buffered bytes.
- `createDecipheriv(algorithm, key, iv)` reverses the process — you must supply the exact same key and iv used to encrypt, otherwise decryption fails or produces garbage.
- Hashing is **one-way** (for passwords, checksums, integrity checks). Encryption is **two-way** (for data you must read back later, like a credit card number in a database).

---

## Example 1 — basic

```js
// File: src/utils/hashDemo.js

const crypto = require('crypto');

// ── SHA-256 hash of a string ────────────────────────────────────────────────
const sha256Hash = crypto
  .createHash('sha256')             // pick the algorithm
  .update('hello backend world')    // input data
  .digest('hex');                   // output as a hex string
console.log('SHA-256:', sha256Hash);
// → always the same 64-character hex string for this exact input

// ── MD5 hash — fast but NOT secure, only use for non-security checksums ────
const md5Hash = crypto
  .createHash('md5')                // MD5 is broken for security, fine for checksums
  .update('hello backend world')
  .digest('hex');
console.log('MD5:', md5Hash);

// ── HMAC — hash signed with a secret key ────────────────────────────────────
const secret = 'webhook-shared-secret';
const signature = crypto
  .createHmac('sha256', secret)     // algorithm + secret key
  .update('order.created:12345')    // the payload being signed
  .digest('hex');
console.log('HMAC signature:', signature);

// ── Secure random values ─────────────────────────────────────────────────────
const sessionId = crypto.randomBytes(16).toString('hex');   // 32-char hex string
console.log('Session ID:', sessionId);

const requestId = crypto.randomUUID();                       // ready-made UUID
console.log('Request ID:', requestId);
```

---

## Example 2 — real world backend use case

```js
// File: src/services/webhookVerifier.js
// Verifying an incoming webhook signature — the pattern used by Stripe, GitHub,
// Shopify, and virtually every payment/integration provider.

const crypto = require('crypto');

const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET; // shared secret from the provider dashboard

// Compute the expected HMAC signature for a raw request body
function computeSignature(rawBody) {
  return crypto
    .createHmac('sha256', WEBHOOK_SECRET)  // sign with our shared secret
    .update(rawBody)                        // the exact raw bytes the provider sent
    .digest('hex');                         // hex-encoded signature string
}

// Verify the signature sent in the request header matches what we compute
function verifyWebhookSignature(rawBody, signatureHeader) {
  const expectedSignature = computeSignature(rawBody);

  // Convert both to Buffers of equal length for a timing-safe comparison
  const expectedBuffer = Buffer.from(expectedSignature, 'hex');
  const receivedBuffer = Buffer.from(signatureHeader, 'hex');

  // Lengths must match before comparing — timingSafeEqual throws on mismatched lengths
  if (expectedBuffer.length !== receivedBuffer.length) {
    return false;
  }

  // timingSafeEqual prevents attackers from guessing the secret via response-time attacks
  return crypto.timingSafeEqual(expectedBuffer, receivedBuffer);
}

// Express-style middleware usage:
function webhookAuthMiddleware(req, res, next) {
  const signatureHeader = req.headers['x-webhook-signature']; // provider sends this header
  const rawBody = req.rawBody;                                  // must be captured BEFORE JSON parsing

  if (!signatureHeader || !verifyWebhookSignature(rawBody, signatureHeader)) {
    return res.status(401).json({ error: 'Invalid webhook signature' }); // reject forged requests
  }

  next(); // signature valid — safe to process the webhook payload
}

module.exports = { verifyWebhookSignature, webhookAuthMiddleware };

// ── A second common case: generating and hashing an API key for storage ────
function generateApiKey(userId) {
  const rawKey = crypto.randomBytes(24).toString('hex');   // the key given to the user ONCE
  const keyHash = crypto
    .createHash('sha256')
    .update(rawKey)
    .digest('hex');                                          // this is what gets stored in the DB

  return { rawKey, keyHash };  // return rawKey to the user, save only keyHash in dbConnection
}

module.exports.generateApiKey = generateApiKey;
```

---

## Common mistakes

### Mistake 1 — Using Math.random() for tokens, session IDs, or API keys

```js
// ❌ WRONG — Math.random() is not cryptographically secure and can be predicted
const sessionId = Math.random().toString(36).substring(2); // guessable, insecure
const apiKey    = Math.random().toString(36);               // attacker can brute-force patterns

// ✅ CORRECT — crypto.randomBytes pulls from the OS's secure random source
const sessionId = crypto.randomBytes(16).toString('hex');   // unpredictable, safe for auth tokens
const apiKey    = crypto.randomBytes(24).toString('hex');   // safe for API keys
```

### Mistake 2 — Hashing passwords with a plain hash (sha256/md5) instead of a slow, salted algorithm

```js
// ❌ WRONG — sha256/md5 are FAST, which means attackers can brute-force
// billions of password guesses per second using GPUs (rainbow tables, etc.)
const passwordHash = crypto.createHash('sha256').update(userPassword).digest('hex');
// storing this in the database is a security risk — never do this for passwords

// ✅ CORRECT — use a purpose-built slow hashing algorithm like bcrypt or crypto.scrypt
// crypto.scrypt is built into Node and is intentionally slow + salted
const salt = crypto.randomBytes(16).toString('hex');        // unique salt per user
crypto.scrypt(userPassword, salt, 64, (err, derivedKey) => { // scrypt is CPU/memory-hard
  if (err) throw err;
  const passwordHash = derivedKey.toString('hex');           // safe to store alongside the salt
});
// (In real projects, most teams use the `bcrypt` npm package for this instead)
```

### Mistake 3 — Comparing signatures/hashes with === instead of a timing-safe comparison

```js
// ❌ WRONG — regular string comparison leaks timing information.
// An attacker can measure response times to guess the correct signature byte-by-byte
if (expectedSignature === receivedSignature) {
  // vulnerable to a timing attack
}

// ✅ CORRECT — crypto.timingSafeEqual always takes the same amount of time,
// regardless of how many characters match, so timing reveals nothing
const expectedBuffer = Buffer.from(expectedSignature, 'hex');
const receivedBuffer = Buffer.from(receivedSignature, 'hex');

const isValid =
  expectedBuffer.length === receivedBuffer.length &&        // must check length first
  crypto.timingSafeEqual(expectedBuffer, receivedBuffer);    // constant-time comparison
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Takes a string (e.g. a `filePath`'s content, hardcode any sample string)
2. Produces its SHA-256 hash and its MD5 hash, printing both as hex strings
3. Generates a `sessionId` using `crypto.randomBytes(16)` and prints it as hex
4. Generates a `requestId` using `crypto.randomUUID()` and prints it
5. Prints a clear label before each value so the output is readable

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small module `apiKeyManager.js` that:
1. Exports a function `createApiKey()` that generates a random 32-byte API key (hex string) and returns both the raw key and its SHA-256 hash
2. Exports a function `verifyApiKey(rawKey, storedHash)` that hashes the incoming `rawKey` and compares it against `storedHash` using `crypto.timingSafeEqual` (handle the case where lengths differ without throwing)
3. Simulate storage with a plain in-memory object keyed by `userId` (e.g. `{ 'user_1': storedHash }`)
4. Write a small test flow: create a key for `userId = 'user_101'`, store its hash, then verify both a correct raw key (should pass) and a tampered raw key (should fail)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `SecureVault` class for encrypting and decrypting sensitive strings (e.g. before saving them to a database) that:
1. Constructor takes a `secretKey` (a 32-byte Buffer — generate one with `crypto.randomBytes(32)` if not provided) and stores it
2. Has an `encrypt(plainText)` method that:
   - Generates a fresh random `iv` (16 bytes) for every call
   - Uses `createCipheriv('aes-256-cbc', ...)` to encrypt `plainText`
   - Returns a single string combining the `iv` and the ciphertext, separated by a colon, both hex-encoded (e.g. `"<ivHex>:<cipherHex>"`) so it can be stored as one database column
3. Has a `decrypt(encryptedString)` method that:
   - Splits the stored string back into `iv` and ciphertext
   - Uses `createDecipheriv` with the same `secretKey` and extracted `iv` to recover the original plain text
   - Returns the decrypted string
4. Handles the case where `decrypt` is called with a malformed string (missing the colon separator) by throwing a clear error instead of crashing with a cryptic OpenSSL error

Test it:
```js
const vault = new SecureVault();
const encrypted = vault.encrypt('4111-1111-1111-1111');  // pretend credit card number
console.log('Encrypted:', encrypted);
console.log('Decrypted:', vault.decrypt(encrypted));      // should print the original number
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE IMPORT
  const crypto = require('crypto');   // built-in, no npm install

HASHING (one-way, cannot reverse)
  crypto.createHash('sha256').update(data).digest('hex')
  Algorithms: 'sha256' (safe, standard), 'sha512' (stronger), 'md5' (broken, checksums only)
  Use for: file integrity checks, non-secret fingerprints — NOT passwords

HMAC (hash + secret key — proves sender knows the key)
  crypto.createHmac('sha256', secretKey).update(data).digest('hex')
  Use for: webhook signature verification, signed tokens, message authentication

RANDOMNESS (cryptographically secure — NEVER use Math.random() for these)
  crypto.randomBytes(n).toString('hex')   → n random bytes as a hex string
  crypto.randomUUID()                      → ready-made v4 UUID string
  Use for: session IDs, API keys, tokens, IVs, salts

PASSWORD HASHING (slow + salted — NOT createHash)
  crypto.scrypt(password, salt, keylen, callback)   → built-in, CPU/memory-hard
  In practice: most teams use the `bcrypt` npm package instead

SYMMETRIC ENCRYPTION (two-way — for data you must read back)
  crypto.createCipheriv(algorithm, key, iv)     → encrypt
  crypto.createDecipheriv(algorithm, key, iv)   → decrypt
  algorithm: 'aes-256-cbc' is a common safe default
  key: 32 random bytes | iv: 16 random bytes, MUST be fresh per encryption, store it alongside ciphertext

SAFE COMPARISON
  crypto.timingSafeEqual(bufferA, bufferB)   → constant-time compare, prevents timing attacks
  Check buffer.length matches BEFORE calling it — it throws on unequal lengths

NEVER DO
  Math.random() for tokens/keys/session IDs   → predictable, insecure
  createHash for passwords                     → too fast, brute-forceable
  === for comparing signatures/hashes          → leaks timing information
  Reusing the same iv across encryptions       → weakens AES-CBC security

COMMON PATTERNS
  Webhook verification: HMAC-SHA256 of raw body, compare with timingSafeEqual
  API key storage: generate randomBytes, give raw key to user once, store only its hash
  Session ID: randomBytes(16 to 32).toString('hex') or randomUUID()
```

---

## Connected topics

- **25 — buffer module** — hashes, keys, and IVs are all `Buffer` objects under the hood; understanding buffers explains the `.toString('hex')` conversions used everywhere in `crypto`
- **57 — Authentication patterns (JWT)** — `jsonwebtoken` uses HMAC/RSA internally to sign tokens; this topic is the low-level mechanism behind that library
- **104 — Webhooks (sending and receiving)** — verifying incoming webhook signatures with HMAC, exactly as shown in Example 2, is the standard security pattern for this topic
