# 49 — Streams as async iterables

## What is this?

Every Readable stream in modern Node.js (fs streams, HTTP request bodies, sockets) is also an **async iterable** — meaning you can loop over it with plain `for await...of` instead of juggling `'data'`, `'end'`, and `'error'` event listeners. Think of a stream like a conveyor belt delivering boxes one at a time: instead of setting up a camera that fires an event every time a box passes (event-based), you can just stand there and say "hand me the next box when it's ready" (`for await...of`) — much closer to how you already read arrays with a normal `for...of` loop.

## Why does it matter for backend development?

Backend code constantly processes data it can't fully trust to fit in memory: multi-gigabyte log files, CSV exports, HTTP request bodies, database export cursors. `for await...of` lets you write that processing as a simple, top-to-bottom `async` function with real `try/catch` error handling, instead of a tangle of event listeners and manual buffering. It also gives you **automatic backpressure** — the loop only pulls the next chunk once your `await`ed work on the current chunk finishes, so a slow database write or slow disk naturally throttles a fast file read, with zero extra code. This is the pattern used to build NDJSON exporters, log processors, CSV importers, and streaming API handlers in production Node services.

---

## Syntax / API

```js
// fs is Node's built-in file system module — createReadStream returns a Readable stream
const fs = require('fs');

async function readFileWithForAwait(filePath) {
  // Open the file as a readable byte stream (does NOT load the whole file into memory)
  const readableStream = fs.createReadStream(filePath, { encoding: 'utf8' });

  // for await...of pulls one chunk at a time from the stream's async iterator
  for await (const chunk of readableStream) {
    // chunk is a string here because we set { encoding: 'utf8' } above
    console.log('Received chunk of length:', chunk.length);
  }

  // Loop exits automatically when the stream emits 'end' — no extra listener needed
  console.log('Done reading', filePath);
}

// ── Async generator: a function that can `yield` values one at a time, asynchronously ──
async function* readLines(readableStream) {
  let leftover = '';                       // holds a partial line split across two chunks

  for await (const chunk of readableStream) {
    leftover += chunk;                     // append new data to whatever is left over
    const lines = leftover.split('\n');    // split on newlines
    leftover = lines.pop();                // last piece may be incomplete — save it for next round

    for (const line of lines) {
      yield line;                          // yield each COMPLETE line, one at a time
    }
  }

  if (leftover) yield leftover;            // yield the final line if the file didn't end with '\n'
}

// ── Consuming a custom async generator exactly like a stream ──
async function countLines(filePath) {
  const readableStream = fs.createReadStream(filePath, { encoding: 'utf8' });
  let count = 0;

  for await (const line of readLines(readableStream)) {  // readLines() returns an async iterable
    count++;                                              // runs once per complete line
  }

  return count;
}

module.exports = { readFileWithForAwait, readLines, countLines };
```

---

## How it works — line by line

`fs.createReadStream()` gives you back a `Readable` stream object. Since Node 10, every `Readable` implements the **async iterator protocol** — it has a `Symbol.asyncIterator` method under the hood. That's the only thing `for await...of` needs to work: it calls that method, gets back an iterator, and repeatedly calls `.next()` on it, `await`ing each result before moving to the next chunk.

Compared to the old event-based style:

```
Event style:                          for await style:
stream.on('data', chunk => {...})     for await (const chunk of stream) {...}
stream.on('end', () => {...})         // loop just ends naturally
stream.on('error', err => {...})      // wrap the whole loop in try/catch
```

The `async function*` syntax defines an **async generator** — a function that can pause and resume, handing back one value at a time via `yield`, while still being able to `await` inside it. `readLines()` above uses this to turn a *stream of chunks* (which may cut lines in half) into a *stream of complete lines* — each `yield` produces exactly one line, and the function remembers leftover partial text between calls using a variable that persists across `yield`s (a normal function can't do this; only generators can pause mid-execution and resume later with state intact).

Because `readLines()` itself is an async generator, calling `readLines(someStream)` returns another async iterable — which means you can `for await...of` it exactly like you would a raw stream. This is what makes async iterables **composable**: you can chain generator after generator (`parse` → `filter` → `transform`), and each stage only pulls from the one before it when asked, keeping memory flat no matter how big the underlying file is.

---

## Example 1 — basic

```js
// File: src/scripts/count-log-lines.js
const fs = require('fs');
const path = require('path');

// Path to a log file we want to summarize — built safely with __dirname (see Topic 06)
const filePath = path.join(__dirname, '..', 'logs', 'requests.log');

async function summarizeLogFile(filePath) {
  // Open the file as a UTF-8 text stream — Node reads it in chunks, not all at once
  const readableStream = fs.createReadStream(filePath, { encoding: 'utf8' });

  let totalChars = 0;   // running total of characters seen so far
  let totalChunks = 0;  // how many chunks the OS delivered (varies by file size / buffer size)

  try {
    // for await pulls the next chunk only once the previous iteration's body has finished
    for await (const chunk of readableStream) {
      totalChunks++;              // count this chunk
      totalChars += chunk.length; // accumulate character count
    }
  } catch (err) {
    // Any read error (file missing, permissions, disk failure) lands here
    console.error('Failed to read log file:', err.message);
    throw err;
  }

  console.log(`Read ${totalChunks} chunk(s), ${totalChars} character(s) total.`);
  return { totalChunks, totalChars };
}

summarizeLogFile(filePath)
  .then((stats) => console.log('Summary:', stats))
  .catch(() => process.exit(1)); // non-zero exit code signals failure to shell/CI
```

---

## Example 2 — real world backend use case

```js
// File: src/jobs/process-order-export.js
// Streams a large NDJSON (newline-delimited JSON) file of orders, filters completed
// orders, and writes only those to a new output file — all without loading the
// full dataset into memory. This is the exact shape of a nightly export/report job.

const fs = require('fs');
const path = require('path');

// ── Stage 1: turn raw byte chunks into complete text lines ──
async function* readLines(readableStream) {
  let leftover = '';
  for await (const chunk of readableStream) {
    leftover += chunk;
    const lines = leftover.split('\n');
    leftover = lines.pop();
    for (const line of lines) {
      if (line.trim() !== '') yield line;  // skip blank lines
    }
  }
  if (leftover.trim() !== '') yield leftover;
}

// ── Stage 2: parse each line as JSON, skipping and logging malformed rows ──
async function* parseJsonLines(linesIterable) {
  for await (const line of linesIterable) {
    try {
      yield JSON.parse(line);              // yield the parsed order record
    } catch (parseErr) {
      console.error('Skipping malformed line:', parseErr.message);
      // no yield here — bad rows are simply dropped from the pipeline
    }
  }
}

// ── Stage 3: keep only orders matching a given status ──
async function* filterByStatus(ordersIterable, status) {
  for await (const order of ordersIterable) {
    if (order.status === status) {
      yield order;                        // only completed orders continue downstream
    }
  }
}

async function processOrderExport(inputPath, outputPath) {
  const inputStream  = fs.createReadStream(inputPath, { encoding: 'utf8' });
  const outputStream = fs.createWriteStream(outputPath, { encoding: 'utf8' });

  // Compose the three generators into a single pipeline — nothing runs until iterated
  const completedOrders = filterByStatus(
    parseJsonLines(readLines(inputStream)),
    'completed'
  );

  let processedCount = 0;   // total orders parsed
  let matchedCount    = 0;  // orders that passed the filter
  let totalAmountCents = 0; // sum of matched order amounts

  for await (const order of completedOrders) {
    processedCount++;
    matchedCount++;
    totalAmountCents += order.amountCents;

    // outputStream.write() returns false when its internal buffer is full —
    // awaiting a 'drain' event here would add backpressure for the write side too
    const canContinue = outputStream.write(JSON.stringify(order) + '\n');
    if (!canContinue) {
      await new Promise((resolve) => outputStream.once('drain', resolve));
    }
  }

  outputStream.end(); // flush and close the output file

  console.log(`Matched ${matchedCount} completed orders (total $${totalAmountCents / 100}).`);
  return { processedCount, matchedCount, totalAmountCents };
}

module.exports = { processOrderExport, readLines, parseJsonLines, filterByStatus };

// Usage:
// const inputPath  = path.join(__dirname, '..', 'data', 'orders-2026-07-26.ndjson');
// const outputPath = path.join(__dirname, '..', 'data', 'orders-completed.ndjson');
// processOrderExport(inputPath, outputPath).catch((err) => {
//   console.error('Export job failed:', err);
//   process.exit(1);
// });
```

---

## Common mistakes

### Mistake 1 — Mixing `.on('data')` with `for await...of` on the same stream

```js
// ❌ WRONG — attaching a 'data' listener switches the stream into flowing mode,
// which conflicts with the internal paused-mode reading for await...of relies on;
// chunks get consumed by the listener and the loop below sees nothing (or crashes)
readableStream.on('data', (chunk) => console.log('data event:', chunk.length));

for await (const chunk of readableStream) {
  console.log('for await:', chunk.length); // may never run, or run with missing data
}

// ✅ CORRECT — pick ONE consumption style per stream, never both
for await (const chunk of readableStream) {
  console.log('for await:', chunk.length); // this is the only consumer, always safe
}
```

### Mistake 2 — Forgetting chunks are Buffers by default, not strings

```js
// ❌ WRONG — createReadStream defaults to raw Buffer chunks; calling string
// methods on a Buffer either throws or silently does the wrong thing
const readableStream = fs.createReadStream(filePath); // no encoding specified
for await (const chunk of readableStream) {
  const lines = chunk.split('\n'); // TypeError: chunk.split is not a function
}

// ✅ CORRECT — set an encoding so chunks arrive as strings
const readableStream = fs.createReadStream(filePath, { encoding: 'utf8' });
for await (const chunk of readableStream) {
  const lines = chunk.split('\n'); // works — chunk is already a string
}
// (Or call chunk.toString('utf8') per-chunk if you need Buffers for other logic too)
```

### Mistake 3 — Trying to iterate the same stream from two places at once

```js
// ❌ WRONG — a Readable stream is single-pass; two concurrent for-await loops
// fight over the same underlying data, so each one only gets SOME of the chunks
const readableStream = fs.createReadStream(filePath, { encoding: 'utf8' });

async function countLines() {
  let n = 0;
  for await (const _chunk of readableStream) n++;  // steals chunks from the loop below
  return n;
}
async function countChars() {
  let n = 0;
  for await (const chunk of readableStream) n += chunk.length; // gets leftovers, wrong result
  return n;
}
await Promise.all([countLines(), countChars()]); // both results are unreliable

// ✅ CORRECT — read the stream ONCE, and derive every metric inside that single loop
async function analyzeFile(filePath) {
  const readableStream = fs.createReadStream(filePath, { encoding: 'utf8' });
  let lineCount = 0;
  let charCount = 0;
  for await (const chunk of readableStream) {
    charCount += chunk.length;
    lineCount += (chunk.match(/\n/g) || []).length;
  }
  return { lineCount, charCount }; // one pass, both numbers guaranteed correct
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Uses `fs.createReadStream()` with `{ encoding: 'utf8' }` on any text file of your choice
2. Uses `for await...of` to read it chunk by chunk
3. Tracks and prints the total number of chunks received and the total character count
4. Wraps the loop in `try/catch` and logs a clear error message if the file doesn't exist

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an async generator function `readLines(readableStream)` (from scratch, don't copy the one above) that:
1. Accepts a readable stream already opened with `{ encoding: 'utf8' }`
2. Correctly handles the case where a single line is split across two separate chunks
3. `yield`s one complete line at a time (skip empty/blank lines)
4. Also `yield`s the final line even if the file doesn't end with a trailing newline

Then write a `printNumberedLines(filePath)` function that opens a file, consumes your `readLines()` generator with `for await...of`, and prints each line prefixed with its line number, e.g. `1: some log entry here`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a composable async-generator pipeline for processing a large NDJSON file of user signup events, where each line looks like:
`{"userId": "user_482", "plan": "pro", "signupSource": "referral", "amountCents": 2900}`

1. Write `readLines(readableStream)` — same contract as Exercise 2
2. Write `parseJsonLines(linesIterable)` — an async generator that parses each line as JSON, and on a parse error logs the bad line and **continues** instead of crashing the whole pipeline
3. Write `filterBySource(recordsIterable, sourceName)` — an async generator that only yields records where `signupSource === sourceName`
4. Write `mapToRevenue(recordsIterable)` — an async generator that yields just `{ userId, amountCents }` for each matched record
5. Write a `summarizeReferralRevenue(inputPath)` function that composes all four generators together, sums `amountCents` across every matched record, counts how many matched, and returns `{ matchedCount, totalAmountCents }`
6. Make sure the whole pipeline never loads the full file into memory at once — verify by reasoning about it (or testing on a large generated file)

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT'S ASYNC-ITERABLE OUT OF THE BOX
  fs.createReadStream()   → Readable stream, async-iterable
  http.IncomingMessage    → request body stream, async-iterable
  net.Socket              → TCP socket, async-iterable
  Any custom Readable     → async-iterable automatically (Node 10+)

BASIC LOOP
  for await (const chunk of readableStream) { ... }
  → pulls next chunk only after the current loop body finishes (built-in backpressure)
  → loop exits automatically on stream 'end'
  → wrap in try/catch to handle stream 'error' events

ASYNC GENERATORS
  async function* name(iterable) {
    for await (const item of iterable) {
      yield transformed(item);   // produces one output value at a time
    }
  }
  → calling name(stream) returns ANOTHER async iterable — chainable

COMPOSING PIPELINES
  filterX(mapY(parseZ(readLines(stream))))
  → each stage only runs when the next stage asks for a value ("pull" based)
  → memory stays flat regardless of total file size

GOTCHAS
  - Never mix .on('data') listeners with for await...of on the same stream
  - Default encoding is Buffer, not string — pass { encoding: 'utf8' } or call .toString()
  - A stream can only be consumed ONCE — no parallel for-await loops on one stream
  - break/return out of a for-await loop auto-destroys the underlying stream (safe cleanup)
  - Check writable.write() return value; if false, await the 'drain' event before writing more

WHEN TO REACH FOR THIS
  - Processing files too large to fit in memory (logs, NDJSON, CSV exports)
  - Streaming HTTP request bodies without buffering the whole payload first
  - Building ETL-style pipelines: read → parse → filter → transform → write
```

---

## Connected topics

- **26 — stream module in depth** — Readable/Writable/Transform internals, `pipeline()`, and manual backpressure handling that underpins everything `for await...of` does automatically
- **47 — async/await in Node — advanced** — async iterators are the general-purpose version of this pattern; this topic is the streams-specific application of it
- **17 — fs module — streams and large files** — `createReadStream`/`createWriteStream` fundamentals that Example 1 and Example 2 build directly on top of
