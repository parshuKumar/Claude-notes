# 41 — Working with JSON at scale

## What is this?

JSON (`JSON.parse` / `JSON.stringify`) is easy when a file is small enough to fit entirely in memory — but backend systems routinely deal with JSON files that are hundreds of megabytes or gigabytes in size (exports, logs, analytics dumps, third-party data feeds). "Working with JSON at scale" means handling those cases safely: knowing the edge cases that break `JSON.parse`/`stringify`, and switching to **streaming JSON parsers** that read and process the data piece by piece instead of loading the whole thing into RAM at once. Think of it like reading a phone book — flipping through it page by page (streaming) versus trying to photograph the entire book in one shot and hold it in your head (`JSON.parse` on the whole file).

## Why does it matter for backend development?

`JSON.parse(fs.readFileSync(...))` is one of the most common causes of production crashes in Node.js backends: it loads the entire file into a string, then builds an entire in-memory object tree, often requiring 3-5x the file's size in extra RAM. A 500MB JSON export can easily exceed Node's default heap limit and crash the process with `RangeError: Invalid string length` or an out-of-memory kill. Backend developers who import data feeds, generate reports, process webhook payloads, or migrate large datasets must know when to reach for streaming JSON parsers (like `JSONStream` or `stream-json`) instead of the synchronous built-in methods, and must understand `JSON.stringify`'s edge cases (circular references, `undefined`, `BigInt`, `Date`) to avoid subtle data corruption bugs.

---

## Syntax / API

```js
// ── Built-in JSON methods (small/medium payloads) ───────────────────────────
const fs = require('fs');

// Parse a JSON string into a JS object — throws SyntaxError on invalid JSON
const requestBody = JSON.parse('{"userId": 42, "isActive": true}');

// Convert a JS object into a JSON string
const jsonText = JSON.stringify({ userId: 42, isActive: true });

// stringify with pretty-printing — 3rd arg is indentation spaces
const pretty = JSON.stringify({ userId: 42 }, null, 2);

// stringify with a replacer function — filter or transform values before output
const safeJson = JSON.stringify(requestBody, (key, value) => {
  if (key === 'authToken') return undefined; // drop sensitive fields
  return value;
});

// parse with a reviver function — transform values as they're parsed
const parsed = JSON.parse(jsonText, (key, value) => {
  if (key === 'createdAt') return new Date(value); // revive date strings
  return value;
});

// ── Streaming JSON (large files) ─────────────────────────────────────────────
// npm install JSONStream
const JSONStream = require('JSONStream');

// Create a stream that parses one array element at a time (path syntax below)
const parseStream = JSONStream.parse('*'); // '*' = each item in a top-level array

// Pipe a large file through the parser instead of reading it all into memory
fs.createReadStream('./large-export.json')
  .pipe(parseStream)
  .on('data', (record) => {
    // 'record' is one fully-parsed object — process it immediately
  })
  .on('end', () => {
    // whole file has been streamed through
  });
```

---

## How it works — line by line

`JSON.parse` and `JSON.stringify` are **synchronous, all-at-once** operations: they need the complete string in memory before they can start, and they build (or consume) the complete object tree in one pass. That is fine for small payloads like an API request body, but it does not scale to files that are gigabytes in size — the process would need to hold the raw text, the parsed string buffer, and the resulting object graph all in memory simultaneously.

`JSONStream.parse('*')` works differently. It wraps a streaming JSON tokenizer that reads the file in small chunks (as they arrive from disk), incrementally recognizes JSON tokens (`{`, `}`, `[`, `,`, string, number...), and **emits complete objects one at a time** as soon as each one is fully parsed — without ever holding the whole file in memory. The `'*'` path tells it "this file is a top-level JSON array, emit each array element as a separate `data` event." You can also target nested paths like `'rows.*'` to stream elements inside a `{ "rows": [...] }` structure.

The `replacer` function in `JSON.stringify(value, replacer, space)` is called for every key/value pair before it's written to the output string — returning `undefined` removes that key entirely, which is exactly how you strip sensitive fields like passwords or tokens before sending data to a client or log file. The `reviver` function in `JSON.parse(text, reviver)` does the mirror-image job during parsing — it lets you convert plain strings (like ISO date strings) back into richer types (like `Date` objects) as the object is being built.

---

## Example 1 — basic

```js
// File: examples/json-basics.js

// JSON.stringify — converting a JS object to a JSON string
const userRecord = {
  userId: 101,
  email: 'dev@example.com',
  isActive: true,
  lastLogin: undefined,      // undefined values are DROPPED by stringify
  metadata: { role: 'admin' },
};

console.log(JSON.stringify(userRecord));
// → '{"userId":101,"email":"dev@example.com","isActive":true,"metadata":{"role":"admin"}}'
// Notice: lastLogin (undefined) silently disappeared — no error, no warning

// Pretty-printed version — 3rd argument controls indentation (2 spaces here)
console.log(JSON.stringify(userRecord, null, 2));

// JSON.parse — converting a JSON string back into a JS object
const jsonText = '{"userId":101,"roles":["admin","editor"]}';
const parsedUser = JSON.parse(jsonText);
console.log(parsedUser.userId);   // → 101 (a real number, not a string)
console.log(parsedUser.roles);    // → ['admin', 'editor'] (a real array)

// Invalid JSON throws a SyntaxError — always wrap in try/catch when parsing
// untrusted or external input (request bodies, files, API responses)
try {
  JSON.parse('{userId: 101}'); // missing quotes around key — invalid JSON
} catch (err) {
  console.log('Parse failed:', err.message); // → Unexpected token u in JSON...
}
```

---

## Example 2 — real world backend use case

```js
// File: src/services/importUsersFromExport.js
// Imports a large user-export JSON file (could be 100MB+) into the database
// without loading the whole file into memory — used for data migrations,
// bulk imports from a partner API dump, or nightly batch jobs.

const fs = require('fs');
const JSONStream = require('JSONStream'); // npm install JSONStream

/**
 * Streams a large JSON array file and inserts each record into the DB
 * one at a time, so memory usage stays flat regardless of file size.
 * @param {string} filePath - absolute path to the export file
 * @param {object} dbConnection - an object with an insertUser(record) method
 */
function importUsersFromExport(filePath, dbConnection) {
  return new Promise((resolve, reject) => {
    let processedCount = 0; // track progress for logging

    // Open a read stream instead of fs.readFileSync — reads in small chunks
    const fileStream = fs.createReadStream(filePath, { encoding: 'utf8' });

    // '*' means: the file is a top-level JSON array, emit each element
    const jsonParser = JSONStream.parse('*');

    fileStream
      .pipe(jsonParser)              // pipe raw bytes into the JSON tokenizer
      .on('data', async (userRecord) => {
        // 'userRecord' is one fully-parsed { userId, email, ... } object
        jsonParser.pause();          // pause the stream while we await the DB write

        try {
          await dbConnection.insertUser(userRecord); // insert one record at a time
          processedCount += 1;

          if (processedCount % 1000 === 0) {
            console.log(`[import] ${processedCount} users processed so far`);
          }
          jsonParser.resume();       // resume reading more records
        } catch (err) {
          jsonParser.destroy();      // stop the stream on a hard failure
          reject(err);
        }
      })
      .on('error', (err) => reject(err))   // malformed JSON in the file
      .on('end', () => resolve(processedCount)); // all records processed
  });
}

module.exports = { importUsersFromExport };

// Usage:
// const count = await importUsersFromExport('/data/exports/users-2026-07.json', dbConnection);
// console.log(`Import finished: ${count} users imported`);
```

---

## Common mistakes

### Mistake 1 — Reading a huge file synchronously and calling JSON.parse on it

```js
// ❌ WRONG — loads the ENTIRE file into a string, then builds the ENTIRE
// object tree in memory. A 1GB file can crash the process outright, or
// throw "RangeError: Invalid string length" if it exceeds V8's string limit.
const fs = require('fs');
const rawData = fs.readFileSync('/data/exports/large-export.json', 'utf8');
const records = JSON.parse(rawData); // process may run out of heap here

// ✅ CORRECT — stream the file and process one record at a time
const JSONStream = require('JSONStream');
fs.createReadStream('/data/exports/large-export.json')
  .pipe(JSONStream.parse('*'))
  .on('data', (record) => {
    // handle one record; memory stays flat regardless of file size
  })
  .on('end', () => console.log('Done streaming large file'));
```

### Mistake 2 — Assuming JSON.stringify handles every JS value

```js
// ❌ WRONG — circular references throw, BigInt throws, functions/undefined
// silently vanish — none of this is obvious from reading the code
const sessionCache = { sessionId: 'abc123' };
sessionCache.self = sessionCache;         // circular reference
JSON.stringify(sessionCache);
// → TypeError: Converting circular structure to JSON

const apiKey = { keyId: 1, rateLimit: 9007199254740993n }; // BigInt
JSON.stringify(apiKey);
// → TypeError: Do not know how to serialize a BigInt

// ✅ CORRECT — handle these cases explicitly before/while stringifying
// 1. Break circular refs with a replacer that tracks seen objects
function safeStringify(value) {
  const seen = new WeakSet();
  return JSON.stringify(value, (key, val) => {
    if (typeof val === 'object' && val !== null) {
      if (seen.has(val)) return '[Circular]'; // replace repeat refs
      seen.add(val);
    }
    if (typeof val === 'bigint') return val.toString(); // BigInt → string
    return val;
  });
}
console.log(safeStringify(sessionCache)); // works, no crash
```

### Mistake 3 — Trusting JSON.parse on untrusted input without try/catch

```js
// ❌ WRONG — a malformed request body (bad client, network corruption,
// truncated upload) crashes the whole route handler with an uncaught
// SyntaxError, potentially taking down the request (or the process)
function handleWebhook(requestBody) {
  const payload = JSON.parse(requestBody); // throws on invalid JSON — unhandled
  return payload.eventType;
}

// ✅ CORRECT — always wrap JSON.parse on external/untrusted input
function handleWebhook(requestBody) {
  let payload;
  try {
    payload = JSON.parse(requestBody);
  } catch (err) {
    // return a clean 400 response instead of crashing
    throw new Error(`Invalid JSON payload: ${err.message}`);
  }
  return payload.eventType;
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Builds a JS object representing a user profile with fields: `userId`, `email`, `password` (fake value), `lastLogin` set to `undefined`, and `createdAt` set to a `new Date()`.
2. Uses `JSON.stringify` with a **replacer function** to remove the `password` field entirely from the output.
3. Logs the resulting JSON string, pretty-printed with 2-space indentation.
4. Parses that JSON string back with `JSON.parse` using a **reviver function** that converts the `createdAt` string back into a real `Date` object, then logs `typeof parsedProfile.createdAt` to confirm it is `object` (a `Date`), not `string`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Create a file `products.json` containing a top-level JSON array of at least 5 product objects (fields: `productId`, `name`, `price`, `inStock`). Then write a script that:
1. Uses `JSONStream.parse('*')` piped from a `fs.createReadStream` to stream the file.
2. On each `data` event, checks if `inStock` is `true` — if so, pushes the product into an `inStockProducts` array; otherwise skips it.
3. On the `end` event, logs the total count of in-stock products and their combined `price` sum.
4. On the `error` event, logs a clear error message.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `JsonBatchImporter` class for a scenario where you must import a very large JSON array file (e.g. `orders-export.json`, potentially containing 500,000+ objects) into a "database" **in batches**, not one record at a time, to reduce the number of write calls:

1. Constructor takes `filePath` and `batchSize` (default `500`).
2. Has a `run(insertBatchFn)` method that:
   - Streams the file using `JSONStream.parse('*')`.
   - Accumulates parsed records into an in-memory buffer array.
   - Once the buffer reaches `batchSize`, calls `await insertBatchFn(buffer)` (an async function simulating a bulk DB insert), then clears the buffer — **pause the stream while awaiting** the insert so records don't pile up faster than they're written.
   - After the stream ends, flushes any remaining records in the buffer (even if fewer than `batchSize`) with one final call to `insertBatchFn`.
   - Resolves with the total number of records imported once everything is flushed.
3. Handles stream `error` events by rejecting the returned promise.
4. Test it against a JSON file you generate with a small script (e.g. 2,000 fake `{ orderId, amount }` objects) and log the final imported count.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
BUILT-IN JSON METHODS
  JSON.parse(text, reviver?)        → string → JS value; throws SyntaxError on bad input
  JSON.stringify(value, replacer?, space?) → JS value → string
  replacer: function(key, value) or array of allowed keys — filters/transforms output
  reviver:  function(key, value) — transforms values while parsing
  space: number (indent spaces) or string (custom indent) — pretty-printing

STRINGIFY EDGE CASES
  undefined, functions, Symbols       → silently OMITTED from output
  undefined inside an array           → becomes null (kept, not removed)
  circular references                 → throws TypeError
  BigInt                               → throws TypeError (must .toString() first)
  Date objects                         → auto-converted via .toISOString()
  NaN, Infinity, -Infinity             → converted to null

PARSE EDGE CASES
  Trailing commas                      → SyntaxError (not valid JSON, unlike JS objects)
  Single quotes for strings             → SyntaxError (JSON requires double quotes)
  Unquoted keys                         → SyntaxError
  Very large numbers (> 2^53)           → silently lose precision (use string or BigInt)

WHEN TO USE BUILT-IN JSON.parse/stringify
  - Request/response bodies (typically KBs, not GBs)
  - Config files, small exports, cache values
  - Anything that comfortably fits in memory 2-3x over

WHEN TO STREAM INSTEAD
  - Files larger than ~50-100MB
  - Data with unknown/unbounded size (exports, log dumps, ETL pipelines)
  - You need to start processing before the whole file finishes downloading/reading

STREAMING JSON LIBRARIES
  JSONStream.parse('*')          → emit each element of a top-level array
  JSONStream.parse('rows.*')     → emit each element inside { "rows": [...] }
  JSONStream.stringify()         → stream OUT large arrays as JSON without full buffering
  stream-json (alternative pkg) → lower-level tokenizer, more control, actively maintained

STREAMING PATTERN
  fs.createReadStream(filePath)
    .pipe(JSONStream.parse('*'))
    .on('data', record => { ... })   // one object at a time
    .on('error', err => { ... })     // malformed JSON / stream error
    .on('end', () => { ... })        // done

MEMORY RULE OF THUMB
  JSON.parse(fs.readFileSync(file))  → needs ~3-5x file size in RAM
  Streaming parser                    → needs roughly ONE record's worth of RAM at a time
```

---

## Connected topics

- **17 — fs module — streams and large files** — `createReadStream`/`pipe()` are the exact stream primitives that `JSONStream` builds on top of.
- **26 — stream module in depth** — backpressure, pausing/resuming streams, and `pipeline()` explain why the batch-import pattern above pauses the stream during async writes.
- **42 — CSV and file parsing** — the same streaming approach (read in chunks, process one row/record at a time) applies directly to CSV files with libraries like `csv-parse`.
