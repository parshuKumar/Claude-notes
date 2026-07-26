# 25 — buffer module

## What is this?

A `Buffer` is a fixed-size chunk of raw binary data — a block of memory outside V8's normal JavaScript heap, represented as a sequence of bytes (numbers 0–255). JavaScript strings only understand text, but computers, files, and networks deal in raw bytes: images, PDFs, encrypted data, TCP packets. Think of a `Buffer` as a sealed crate of numbered bricks (bytes) — you can look at any brick's number, replace it, or ask "if I read all these bricks as UTF-8 text, what does it spell?" Node.js gives you the `Buffer` class globally, in every file, with no `require()` needed.

## Why does it matter for backend development?

Every backend touches binary data at some point: reading an uploaded image, hashing a password with crypto, streaming a video file, parsing a TCP packet, or reading a file from disk before it becomes a JS string. Node's `fs`, `http`, `net`, `crypto`, and `stream` modules all hand you `Buffer` objects by default — not strings — because I/O is fundamentally byte-based. If you don't understand Buffers, you'll write bugs converting binary data to a string too early (corrupting images), or you'll misunderstand why `Content-Length` doesn't match `string.length` for non-ASCII text. Buffers are the bridge between "bytes on the wire" and "usable JavaScript data."

---

## Syntax / API

```js
// Buffer is a GLOBAL — no require('buffer') needed for the class itself

// ── Creating Buffers ─────────────────────────────────────────────────────────

// From a string — encodes text into bytes (utf8 is the default encoding)
const authToken = Buffer.from('user_42:secret123', 'utf8');

// From an array of byte values (0-255) — manual byte construction
const rawBytes = Buffer.from([72, 101, 108, 108, 111]); // → "Hello" in ASCII codes

// Allocate an empty buffer of a fixed size, zero-filled (SAFE — always use this)
const emptyBuffer = Buffer.alloc(16); // 16 bytes, all zeros

// Allocate without zeroing (FASTER but may contain old memory — use with caution)
const fastBuffer = Buffer.allocUnsafe(16); // must be fully overwritten before reading

// ── Reading Buffers ──────────────────────────────────────────────────────────

console.log(authToken.length);        // → 19 (byte length, NOT character count)
console.log(authToken.toString());    // → 'user_42:secret123' (decodes back to utf8 string)
console.log(authToken[0]);            // → 117 (byte value of 'u', index access like an array)

// ── Encodings — how bytes map to/from text ───────────────────────────────────
const requestBody = Buffer.from('Hello World', 'utf8');

console.log(requestBody.toString('utf8'));   // → 'Hello World' (human-readable text)
console.log(requestBody.toString('hex'));    // → '48656c6c6f20576f726c64' (2 hex chars per byte)
console.log(requestBody.toString('base64')); // → 'SGVsbG8gV29ybGQ=' (compact, safe for JSON/URLs)

// ── Converting back from an encoded string ───────────────────────────────────
const fromHex    = Buffer.from('48656c6c6f', 'hex');    // → <Buffer 48 65 6c 6c 6f>
const fromBase64 = Buffer.from('SGVsbG8=', 'base64');   // → <Buffer 48 65 6c 6c 6f>

// ── Combining and comparing ───────────────────────────────────────────────────
const combined = Buffer.concat([authToken, requestBody]); // joins multiple buffers into one
const isEqual  = authToken.equals(fromHex);               // → false (byte-for-byte comparison)
```

---

## How it works — line by line

A `Buffer` is a fixed-length list of byte values. Once created at a given size, it cannot grow or shrink — you'd need to create a new, bigger buffer and copy data into it. Each slot holds one byte (a number from 0 to 255), so a 16-byte buffer always uses 16 bytes of memory no matter what you put in it.

`Buffer.from(string, encoding)` takes a JavaScript string and converts each character into its byte representation using the encoding rule you specify. `'utf8'` is the default and handles all standard text, including emoji and non-English characters (though some characters take more than 1 byte). `Buffer.from(array)` instead takes literal byte numbers you supply yourself, useful when building binary protocols by hand.

`Buffer.alloc(size)` reserves `size` bytes and fills every one with zero — safe because you never accidentally read leftover data from another part of memory. `Buffer.allocUnsafe(size)` skips the zero-filling step, making it faster, but the bytes could be **anything** left over from previous memory use — you must overwrite every byte before reading from it, or you risk leaking old, possibly sensitive data.

`.toString(encoding)` reverses the process — it reads the raw bytes and reconstructs a string according to the encoding rule. `'hex'` turns each byte into two hexadecimal characters (so a 5-byte buffer becomes a 10-character hex string) — commonly used for things like showing hashes. `'base64'` packs 3 bytes into 4 printable characters, making it compact and safe to embed in JSON, URLs, or email — commonly used for encoding binary file data as text (e.g. image thumbnails, JWT segments).

`Buffer.concat([...])` copies several buffers into one new buffer, in order — used when you receive data in chunks (like an HTTP request body arriving over multiple network packets) and need to reassemble it into one complete piece. `.equals()` compares two buffers byte-by-byte and returns `true` only if every byte matches — the correct way to compare binary data, since `===` only works for reference equality on objects.

---

## Example 1 — basic

```js
// File: src/examples/buffer-basics.js

// Buffer is global — always available, no require() call needed
const message = 'Node.js Backend';

// Convert a string into raw bytes (utf8 encoding by default)
const buf = Buffer.from(message);
console.log('Buffer object :', buf);              // → <Buffer 4e 6f 64 65 2e 6a 73 20 42 61 63 6b 65 6e 64>
console.log('Byte length   :', buf.length);        // → 16 (16 bytes for 16 ASCII chars)

// Decode the buffer back into a readable string
console.log('As utf8 text  :', buf.toString('utf8')); // → 'Node.js Backend'

// Show the SAME bytes in different encodings — same data, different representation
console.log('As hex        :', buf.toString('hex'));    // → '4e6f64652e6a73204261636b656e64'
console.log('As base64     :', buf.toString('base64')); // → 'Tm9kZS5qcyBCYWNrZW5k'

// Access an individual byte by index — returns the numeric byte value
console.log('First byte    :', buf[0]);   // → 78 (this is 'N' in ASCII)

// Allocate a zero-filled buffer and manually write into it
const idBuffer = Buffer.alloc(4);         // 4 empty bytes: [0, 0, 0, 0]
idBuffer.writeUInt32BE(42, 0);            // write the number 42 as a 4-byte big-endian integer
console.log('ID buffer     :', idBuffer); // → <Buffer 00 00 00 2a>
console.log('Read back     :', idBuffer.readUInt32BE(0)); // → 42

// Check whether unicode text takes more bytes than characters
const emoji = Buffer.from('👋');
console.log('Emoji chars   :', '👋'.length);  // → 2 (JS string length, UTF-16 code units)
console.log('Emoji bytes   :', emoji.length); // → 4 (UTF-8 encodes it as 4 bytes)
```

---

## Example 2 — real world backend use case

```js
// File: src/services/avatarUpload.service.js
// Handles a base64-encoded avatar image sent in a JSON request body,
// converts it to a real binary file, and validates its size before saving.

const fs   = require('fs');
const path = require('path');

const MAX_AVATAR_BYTES = 2 * 1024 * 1024; // 2 MB size limit
const UPLOAD_DIR       = path.join(__dirname, '..', '..', 'uploads', 'avatars');

/**
 * Saves a base64 avatar string to disk as a real image file.
 * @param {string} userId       - the user this avatar belongs to
 * @param {string} base64Image  - image data, e.g. "data:image/png;base64,iVBOR..."
 * @returns {string} the saved file path
 */
function saveAvatarFromBase64(userId, base64Image) {
  // Strip the "data:image/png;base64," prefix if the client sent a data URL
  const commaIndex = base64Image.indexOf(',');
  const rawBase64   = commaIndex !== -1 ? base64Image.slice(commaIndex + 1) : base64Image;

  // Decode the base64 text back into real binary bytes
  const imageBuffer = Buffer.from(rawBase64, 'base64');

  // Validate the decoded size BEFORE writing to disk — base64 text length lies about real size
  if (imageBuffer.length > MAX_AVATAR_BYTES) {
    throw new Error(`Avatar too large: ${imageBuffer.length} bytes (max ${MAX_AVATAR_BYTES})`);
  }

  // Ensure the upload directory exists (recursive so parent dirs are created too)
  fs.mkdirSync(UPLOAD_DIR, { recursive: true });

  // Build a safe, predictable file path scoped to this user
  const filePath = path.join(UPLOAD_DIR, `${userId}.png`);

  // Write the raw bytes straight to disk — no string conversion, so the image stays intact
  fs.writeFileSync(filePath, imageBuffer);

  return filePath;
}

/**
 * Generates a short, deterministic ETag-style hash for a file's contents,
 * useful for caching headers (Content-based cache validation).
 */
function generateContentHash(fileBuffer) {
  const crypto = require('crypto'); // Topic 23 — crypto module
  // Hash the raw bytes directly, then represent the digest as compact hex text
  return crypto.createHash('sha256').update(fileBuffer).digest('hex').slice(0, 16);
}

module.exports = { saveAvatarFromBase64, generateContentHash };

// Usage in a route handler:
// const { saveAvatarFromBase64 } = require('./avatarUpload.service');
//
// app.post('/api/users/:userId/avatar', (req, res) => {
//   const { userId }  = req.params;
//   const { image }   = req.body;      // base64 string from client
//   const filePath     = saveAvatarFromBase64(userId, image);
//   res.json({ success: true, path: filePath });
// });
```

---

## Common mistakes

### Mistake 1 — Using `new Buffer()` instead of `Buffer.from()` / `Buffer.alloc()`

```js
// ❌ WRONG — new Buffer() is deprecated and unsafe: behavior changes based on argument type
// It can silently expose uninitialized memory, a known security issue (CVE-2018-XXXX class bugs)
const legacyBuffer = new Buffer('user data'); // deprecated since Node 6, removed guidance in v10+

// ✅ CORRECT — always use the explicit, safe factory methods
const authToken = Buffer.from('user data', 'utf8'); // for existing data (string, array)
const scratchBuf = Buffer.alloc(64);                // for a new, zero-filled empty buffer
```

### Mistake 2 — Treating buffer `.length` as the character count

```js
// ❌ WRONG — assuming byte length equals string character count
const username = 'Zoë Müller'; // contains accented characters (multi-byte in utf8)
const buf = Buffer.from(username, 'utf8');
console.log(username.length === buf.length); // → false! string has 10 chars, buffer has 12 bytes

// If you validate "max 10 characters" using buf.length, you'll reject valid short names
// and accept overly long ones with only ASCII characters.

// ✅ CORRECT — use string.length for character/text limits, buf.length only for byte size
if (username.length > 50) {
  throw new Error('Username too long'); // character-based validation, uses the STRING
}
const contentLength = Buffer.byteLength(username, 'utf8'); // correct way to get byte size
// used for HTTP Content-Length headers, which MUST be byte counts, not character counts
```

### Mistake 3 — Using `Buffer.allocUnsafe()` without immediately overwriting every byte

```js
// ❌ WRONG — allocUnsafe skips zero-filling; unwritten bytes may contain old memory data
// This buffer could leak previously-freed sensitive data (e.g. another request's session token)
const responseBuf = Buffer.allocUnsafe(1024);
responseBuf.write('short reply'); // only fills the first 11 bytes — the other 1013 are garbage
sendToClient(responseBuf); // BUG: sends 1013 bytes of unknown leftover memory to the client

// ✅ CORRECT OPTION 1 — use alloc() unless you have a measured performance reason not to
const safeBuf = Buffer.alloc(1024); // zero-filled, always safe
safeBuf.write('short reply');
sendToClient(safeBuf.subarray(0, 11)); // slice to only the bytes you actually wrote

// ✅ CORRECT OPTION 2 — if using allocUnsafe, always slice to exactly what was written
const fastBuf = Buffer.allocUnsafe(1024);
const bytesWritten = fastBuf.write('short reply');
sendToClient(fastBuf.subarray(0, bytesWritten)); // only send the real, intended bytes
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Creates a Buffer from the string `'sessionId:abc123xyz'` using utf8 encoding
2. Logs its byte length
3. Logs the same data as `hex` and as `base64`
4. Converts the hex string back into a new Buffer, and confirms with `.equals()` that it matches the original buffer
5. Logs whether they are equal

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `encodeApiKey(rawKey)` and `decodeApiKey(encodedKey)` that:
1. `encodeApiKey` takes a plain string API key, converts it to a Buffer, and returns it as a base64 string (this simulates safely transmitting a key in a URL or header)
2. `decodeApiKey` takes that base64 string back and returns the original plain string
3. Add a helper `getKeyByteSize(rawKey)` that returns how many bytes the key takes up in utf8 (use `Buffer.byteLength`, not `.length` on the buffer)
4. Test with at least one key containing only ASCII characters and one containing an emoji or accented character, and log the byte size difference

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `ChunkAssembler` class that simulates reassembling a file that arrives over the network in multiple binary chunks (like an HTTP request body streaming in):
1. Constructor takes an `expectedTotalBytes` number
2. Has an `addChunk(buffer)` method that stores each incoming Buffer in an internal array and tracks the running total of bytes received
3. Has an `isComplete()` method that returns `true` once the running total meets or exceeds `expectedTotalBytes`
4. Has a `getAssembledBuffer()` method that uses `Buffer.concat()` to join all stored chunks into one final Buffer, but throws an error if called before `isComplete()` is true
5. Has a `getProgressPercent()` method that returns how much of the expected data has arrived so far, rounded to the nearest whole number

Test it by feeding in 3-4 chunks of different sizes (e.g. `Buffer.from('part1')`, `Buffer.from('part2')`, etc.) with a made-up `expectedTotalBytes`, checking progress after each chunk, and calling `getAssembledBuffer()` at the end to confirm the reassembled text is correct.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT IT IS
  Buffer  → a fixed-length, raw binary data container (bytes 0-255)
  Global class — no require('buffer') needed for Buffer itself

CREATING BUFFERS
  Buffer.from(string, encoding)   → encode text into bytes ('utf8' is default)
  Buffer.from([byte, byte, ...])  → build from literal byte numbers
  Buffer.alloc(size)              → zero-filled, SAFE, use by default
  Buffer.allocUnsafe(size)        → faster, UNSAFE until fully overwritten

READING / CONVERTING
  buf.toString('utf8')    → decode as human-readable text
  buf.toString('hex')     → 2 hex chars per byte (e.g. hashes, checksums)
  buf.toString('base64')  → compact text-safe encoding (JSON, URLs, JWTs)
  buf.length              → byte count (NOT character count!)
  buf[i]                  → read/write individual byte by index

KEY METHODS
  Buffer.concat([buf1, buf2])     → join multiple buffers into one
  buf.equals(otherBuf)            → byte-for-byte comparison (not ===)
  buf.slice(start, end) / subarray(start, end) → view into part of a buffer
  Buffer.byteLength(str, 'utf8')  → byte size of a STRING without creating a buffer
  buf.writeUInt32BE(num, offset)  → write a number as bytes at a position
  buf.readUInt32BE(offset)        → read bytes back as a number

ENCODINGS CHEAT SHEET
  utf8    → default, human text, variable bytes per character
  hex     → 1 byte = 2 hex characters, doubles the string length
  base64  → 3 bytes = 4 characters, ~33% larger than raw bytes, safe for text transport
  ascii   → 1 byte = 1 character, only for basic English text (legacy)

NEVER DO
  new Buffer(...)                 → deprecated, unsafe — use .from()/.alloc()
  buf.length for text validation  → use string.length or Buffer.byteLength()
  allocUnsafe() without slicing to written bytes → leaks old memory

WHERE BUFFERS SHOW UP
  fs.readFile() results (without encoding option) → returns a Buffer
  http request/response bodies                    → arrive as Buffer chunks
  crypto hash/cipher input and output              → Buffers in, Buffers/hex out
  net/TCP sockets                                  → raw Buffer data
```

---

## Connected topics

- **23 — crypto module** — hashing, HMAC, and encryption all operate directly on Buffers; understanding Buffers is required before using `createHash().update()` correctly
- **17 — fs module — streams and large files** — file streams emit `data` events with Buffer chunks; `Buffer.concat()` is exactly how you'd reassemble a stream manually
- **26 — stream module in depth** — Readable/Writable/Transform streams pass Buffers between each step by default; this topic builds directly on Buffer fundamentals
