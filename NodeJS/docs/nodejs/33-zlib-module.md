# 33 — zlib module

## What is this?

`zlib` is Node's built-in module for compressing and decompressing data using the gzip, deflate, and brotli algorithms. It is the same compression technology your browser and server use to shrink HTTP responses before sending them over the network. Think of it like a vacuum-seal bag for your data — you squeeze the air (redundant bytes) out before shipping it, and the receiver un-seals it back to the original size on arrival.

## Why does it matter for backend development?

Every backend API sends JSON, HTML, or file responses over the network, and network bandwidth is almost always the slowest part of the request/response cycle. Compressing a response with gzip or brotli before sending it can shrink a 200KB JSON payload down to 20-30KB, which means faster page loads, lower bandwidth bills, and happier mobile users on slow connections. `zlib` is also what powers the `compression` middleware in Express, what `.gz` log rotation uses, and what lets you stream-compress large files without loading them entirely into memory. Any backend dev who has ever seen a `Content-Encoding: gzip` header has used this module, directly or indirectly.

---

## Syntax / API

```js
// zlib is a Node core module — no npm install needed
const zlib = require('zlib');

// ── One-shot (buffer-in, buffer-out) methods ────────────────────────────────

// gzip: compress a buffer/string using the gzip format (most common on the web)
zlib.gzip(inputBuffer, (err, compressedBuffer) => {
  // compressedBuffer is a Buffer — smaller than inputBuffer for most text data
});

// gunzip: decompress a gzip-compressed buffer back to the original
zlib.gunzip(compressedBuffer, (err, originalBuffer) => {
  // originalBuffer matches the data before compression
});

// deflate: compress using raw zlib/deflate format (slightly smaller headers than gzip)
zlib.deflate(inputBuffer, (err, compressedBuffer) => {});

// inflate: decompress data that was compressed with deflate
zlib.inflate(compressedBuffer, (err, originalBuffer) => {});

// brotli: newer algorithm, better compression ratio than gzip, supported by all modern browsers
zlib.brotliCompress(inputBuffer, (err, compressedBuffer) => {});
zlib.brotliDecompress(compressedBuffer, (err, originalBuffer) => {});

// ── Sync versions (block the event loop — use only for scripts/CLI tools) ──
const compressedSync = zlib.gzipSync(inputBuffer);   // returns Buffer directly
const originalSync   = zlib.gunzipSync(compressedSync);

// ── Streaming versions (for large files/responses — never buffer everything) ─
const gzipStream   = zlib.createGzip();      // Transform stream: raw in, gzip out
const gunzipStream = zlib.createGunzip();    // Transform stream: gzip in, raw out
const deflateStream = zlib.createDeflate();  // Transform stream: raw in, deflate out
const inflateStream = zlib.createInflate();  // Transform stream: deflate in, raw out

// Usage: readableStream.pipe(gzipStream).pipe(writableStream)
```

---

## How it works — line by line

`zlib` gives you the same compression logic in three shapes, and choosing the right shape matters:

- **Callback methods** (`zlib.gzip`, `zlib.gunzip`, etc.) take the entire input as a Buffer or string, compress it in one go on a background thread (via libuv's thread pool), and hand you back the complete result through a callback. Good for small-to-medium payloads like a JSON API response.
- **Sync methods** (`zlib.gzipSync`, etc.) do the exact same work but block the main thread until done — no callback, just a direct return value. Fine for one-off CLI scripts or build tooling, dangerous in a running server because it freezes all other requests while compressing.
- **Stream methods** (`zlib.createGzip()`, etc.) return a **Transform stream** — data flows in one chunk at a time and compressed chunks flow out, without ever holding the whole file in memory. This is the only safe option for large files (videos, big CSV exports, log archives) because it keeps memory usage flat no matter how big the input is.

Compression works by finding repeated patterns in data and replacing them with shorter references — this is why text-based formats (JSON, HTML, CSV) compress extremely well (often 70-90% smaller) while already-compressed formats (JPEG, MP4, ZIP) barely shrink at all and sometimes even grow slightly. Decompression (`gunzip`/`inflate`) reverses the process exactly, byte for byte — it is lossless, unlike image or video compression.

On the web, the client tells the server what it can decompress using the `Accept-Encoding` request header (e.g. `gzip, deflate, br`), and the server responds with a `Content-Encoding` header naming which one it actually used, so the browser knows how to un-compress the body before rendering it.

---

## Example 1 — basic

```js
// File: src/examples/zlib-basics.js
const zlib = require('zlib');

// The data we want to compress — a repetitive string compresses very well
const requestBody = JSON.stringify({
  userId: 'user_42',
  role: 'admin',
  permissions: ['read', 'write', 'delete', 'read', 'write', 'delete'], // repeated values compress great
});

// Compress the string using gzip — output is asynchronous, result is a Buffer
zlib.gzip(requestBody, (err, compressedBuffer) => {
  if (err) {
    console.error('Compression failed:', err.message);
    return;
  }

  // Compare sizes — gzip usually shrinks JSON significantly
  console.log('Original size  :', Buffer.byteLength(requestBody), 'bytes');
  console.log('Compressed size:', compressedBuffer.length, 'bytes');

  // Now reverse it — decompress the buffer back to the original string
  zlib.gunzip(compressedBuffer, (err, decompressedBuffer) => {
    if (err) {
      console.error('Decompression failed:', err.message);
      return;
    }

    const restored = decompressedBuffer.toString('utf8'); // Buffer → string
    console.log('Restored equals original:', restored === requestBody); // true
  });
});
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A raw http server that gzip-compresses JSON responses when the client supports it.
// This is exactly what compression middleware does under the hood in Express/Fastify.

const http = require('http');
const zlib = require('zlib');

const server = http.createServer((req, res) => {
  // Simulate a large-ish API response — an array of user records
  const responseData = {
    users: Array.from({ length: 500 }, (_, i) => ({
      userId: `user_${i}`,
      email: `user${i}@example.com`,
      role: 'member',
    })),
  };

  const jsonBody = JSON.stringify(responseData);         // stringify once, reuse below
  const acceptEncoding = req.headers['accept-encoding'] || ''; // what the client can handle

  // Only compress if the client says it supports gzip — never guess
  if (acceptEncoding.includes('gzip')) {
    zlib.gzip(jsonBody, (err, compressedBody) => {
      if (err) {
        // Compression failed — fall back to sending uncompressed rather than crashing
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(jsonBody);
        return;
      }

      // Tell the client this body is gzip-encoded so it knows to decompress it
      res.writeHead(200, {
        'Content-Type': 'application/json',
        'Content-Encoding': 'gzip',           // critical header — without it browser shows garbage
        'Vary': 'Accept-Encoding',            // tells caches responses differ by this header
      });
      res.end(compressedBody);                // send the smaller, compressed buffer
    });
  } else {
    // Client didn't ask for compression (rare, e.g. old tools/curl without --compressed)
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(jsonBody);
  }
});

server.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
  // Test with: curl -H "Accept-Encoding: gzip" --compressed http://localhost:3000
});
```

```js
// File: src/utils/compress-file.js
// Streaming file compression — used for compressing log files or exports
// without ever loading the whole file into memory (safe for multi-GB files).

const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream');

function compressLogFile(filePath, callback) {
  const readStream = fs.createReadStream(filePath);        // reads the source file in chunks
  const gzipStream = zlib.createGzip();                     // Transform stream: raw → gzip
  const writeStream = fs.createWriteStream(`${filePath}.gz`); // writes compressed output

  // pipeline() connects the streams and handles errors/cleanup automatically
  pipeline(readStream, gzipStream, writeStream, (err) => {
    if (err) {
      callback(err);       // something broke mid-stream — surface it to the caller
      return;
    }
    callback(null, `${filePath}.gz`); // success — return the path to the new .gz file
  });
}

module.exports = { compressLogFile };

// Usage:
// compressLogFile('/var/log/app/access.log', (err, gzPath) => {
//   if (err) return console.error('Compression failed:', err.message);
//   console.log('Compressed log written to:', gzPath);
// });
```

---

## Common mistakes

### Mistake 1 — Buffering an entire large file into memory instead of streaming

```js
// ❌ WRONG — reads the whole file into RAM before compressing
// A 2GB export file means 2GB+ held in memory at once — can crash the process
const fs = require('fs');
const zlib = require('zlib');

const fileBuffer = fs.readFileSync('/data/exports/full-report.csv'); // loads everything
zlib.gzip(fileBuffer, (err, compressed) => {
  fs.writeFileSync('/data/exports/full-report.csv.gz', compressed);  // then writes everything
});

// ✅ CORRECT — stream the file through gzip in small chunks, memory stays flat
const { pipeline } = require('stream');

pipeline(
  fs.createReadStream('/data/exports/full-report.csv'),
  zlib.createGzip(),
  fs.createWriteStream('/data/exports/full-report.csv.gz'),
  (err) => {
    if (err) console.error('Stream compression failed:', err.message);
    else console.log('Compressed without loading the file into memory');
  }
);
```

### Mistake 2 — Sending Content-Encoding: gzip without actually compressing the body

```js
// ❌ WRONG — sets the header but sends the raw, uncompressed JSON
// The browser will try to gunzip plain text and show a decode error or garbled page
res.writeHead(200, {
  'Content-Type': 'application/json',
  'Content-Encoding': 'gzip',   // LIES to the client — body was never compressed
});
res.end(jsonBody); // raw string, not gzip data

// ✅ CORRECT — only set the header on the actual compressed buffer
zlib.gzip(jsonBody, (err, compressedBody) => {
  if (err) throw err;
  res.writeHead(200, {
    'Content-Type': 'application/json',
    'Content-Encoding': 'gzip',
  });
  res.end(compressedBody); // the header now matches the real body format
});
```

### Mistake 3 — Using sync zlib methods inside a running request handler

```js
// ❌ WRONG — gzipSync blocks the entire event loop while compressing
// Under load, every other request waits until this compression finishes
const http = require('http');
const zlib = require('zlib');

http.createServer((req, res) => {
  const compressed = zlib.gzipSync(bigJsonBody); // blocks ALL concurrent requests
  res.end(compressed);
}).listen(3000);

// ✅ CORRECT — use the async callback (or stream) version so other requests keep flowing
http.createServer((req, res) => {
  zlib.gzip(bigJsonBody, (err, compressed) => {
    if (err) {
      res.writeHead(500);
      res.end('Compression error');
      return;
    }
    res.writeHead(200, { 'Content-Encoding': 'gzip' });
    res.end(compressed); // event loop stays free for other requests while this runs
  });
}).listen(3000);
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Creates a JSON string representing a `sessionId`, `userId`, and an array of at least 20 repeated role strings (e.g. `'viewer'` repeated).
2. Compresses it using `zlib.gzip`.
3. Logs the original size in bytes and the compressed size in bytes (use `Buffer.byteLength` and `.length`).
4. Decompresses the result with `zlib.gunzip` and logs whether the restored string strictly equals the original.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build two small functions using promises (wrap the callback-based zlib methods, or use `require('util').promisify`):
1. `compressData(inputString)` — returns a Promise that resolves with the gzip-compressed Buffer.
2. `decompressData(buffer)` — returns a Promise that resolves with the original string.

Then write an async function `roundTrip(inputString)` that:
1. Compresses the input with `compressData`.
2. Decompresses the result with `decompressData`.
3. Returns an object: `{ originalSize, compressedSize, compressionRatio, restoredMatches }` where `compressionRatio` is `compressedSize / originalSize` (rounded to 2 decimals) and `restoredMatches` is a boolean.

Test it with a large repetitive string (e.g. a JSON array of 200 similar `apiKey` records) and log the result object.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small Express-free HTTP server (`http.createServer`) that serves a `GET /export` endpoint returning a large CSV string (generate at least 1000 rows of fake data with columns like `userId,email,createdAt`). Requirements:
1. Check the `accept-encoding` request header — if it includes `'br'` (brotli), compress the response with `zlib.brotliCompress` and set `Content-Encoding: br`.
2. Else if it includes `'gzip'`, compress with `zlib.gzip` and set `Content-Encoding: gzip`.
3. Else send the CSV uncompressed.
4. In all cases set `Content-Type: text/csv` and a `Vary: Accept-Encoding` header.
5. Add a second endpoint `GET /export-stream` that does the same CSV generation but writes it to a temp file first, then streams it through `zlib.createGzip()` directly into the response (using `pipeline`) instead of buffering it — compare memory behavior conceptually between the two endpoints in a comment.

Test both endpoints with `curl -H "Accept-Encoding: gzip" --compressed http://localhost:3000/export --output out.csv` and verify the file decompresses correctly.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE MODULE — no install needed: const zlib = require('zlib');

ALGORITHMS
  gzip     → .gz format, most widely supported, used for HTTP + log files
  deflate  → raw zlib format, slightly smaller headers than gzip, less universal
  brotli   → newer, better compression ratio, supported by all modern browsers (2020+)

THREE API SHAPES (pick based on need)
  Callback (async, non-blocking)   → zlib.gzip(buf, cb)      / zlib.gunzip(buf, cb)
  Sync (blocks event loop)         → zlib.gzipSync(buf)      / zlib.gunzipSync(buf)
  Stream (Transform, chunked)      → zlib.createGzip()       / zlib.createGunzip()

  Deflate equivalents: zlib.deflate/deflateSync/createDeflate, zlib.inflate/.../createInflate
  Brotli equivalents:  zlib.brotliCompress/.../brotliCompressSync, brotliDecompress/...

WHEN TO USE WHICH SHAPE
  Small JSON API response       → callback methods (zlib.gzip)
  One-off CLI script            → sync methods OK
  Large files / uploads/exports → ALWAYS streams (createGzip + pipeline)
  Never use *Sync inside a live request handler — blocks all other requests

HTTP HEADERS INVOLVED
  Request:  Accept-Encoding: gzip, deflate, br   → what the CLIENT can decompress
  Response: Content-Encoding: gzip               → what the SERVER actually used
  Response: Vary: Accept-Encoding                → tells caches to key on this header

GOTCHAS
  Setting Content-Encoding without actually compressing the body → broken response
  Compressing already-compressed data (jpg, mp4, zip) → little/no benefit, wastes CPU
  gzipSync/brotliCompressSync on the request path → blocks the event loop under load
  Forgetting pipeline()'s error handling on streams → silent partial files on failure

TYPICAL EXPRESS EQUIVALENT
  const compression = require('compression');
  app.use(compression());   // does exactly this pattern automatically for all routes
```

---

## Connected topics

- **26 — stream module in depth** — `zlib.createGzip()` returns a Transform stream; understanding Readable/Writable/Transform explains exactly how streaming compression flows through `pipeline()`.
- **20 — http module** — compressing responses only matters in the context of req/res headers (`Accept-Encoding`, `Content-Encoding`) covered when building raw HTTP servers.
- **71 — Compression and performance headers** — the `compression` Express middleware wraps this exact `zlib` API automatically for every route, plus adds ETag/Cache-Control interplay.
