# 26 — stream module in depth

## What is this?

The `stream` module gives you a way to process data **piece by piece** instead of loading it all into memory at once. A stream is like a water pipe — data flows through in small chunks from a source to a destination, and you can attach filters along the way that transform the water as it passes. Node.js has four core stream types: **Readable** (a source you read from), **Writable** (a destination you write to), **Duplex** (both at once, like a socket), and **Transform** (a Duplex that modifies data as it passes through, like a filter).

## Why does it matter for backend development?

Every backend app deals with data that can be too large to hold in RAM in one go — a 2GB video upload, a multi-gigabyte database export, a live log file being tailed. If you read that entire file into memory with `fs.readFileSync`, your process can crash with an out-of-memory error, or at minimum spike memory usage and slow down every other request being served. Streams solve this by processing data in small chunks (default 64KB for buffers), so memory usage stays flat no matter how big the input is. Every HTTP request/response object in Node is a stream, every file read/write in production code should be a stream, and every proxy, compression layer, and file upload handler is built on streams under the hood.

---

## Syntax / API

```js
// Node's stream classes all live in the built-in 'stream' module
const { Readable, Writable, Duplex, Transform, pipeline } = require('stream');
const fs = require('fs');

// ── Readable stream — a SOURCE of data ──────────────────────────────────────
const readStream = fs.createReadStream('./data/large-export.csv');
// Emits 'data' events with chunks, or can be consumed with for-await-of

// ── Writable stream — a DESTINATION for data ────────────────────────────────
const writeStream = fs.createWriteStream('./data/output.csv');
// write() returns false when the internal buffer is full (backpressure signal)

// ── Piping — connects a Readable's output straight into a Writable's input ─
readStream.pipe(writeStream);
// Handles backpressure automatically — pauses the source if the destination is slow

// ── pipeline() — the modern, safe way to connect streams ───────────────────
pipeline(
  readStream,                       // source
  writeStream,                      // destination
  (err) => {                        // callback — fires on success OR failure
    if (err) console.error('Pipeline failed:', err.message);
    else console.log('Pipeline succeeded.');
  }
);
// Automatically destroys all streams and cleans up listeners if any stream errors

// ── Custom Transform stream — modifies data as it flows through ────────────
class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // chunk arrives as a Buffer; convert, transform, push the result onward
    const upper = chunk.toString().toUpperCase();
    this.push(upper);              // send transformed data to the next stream in the chain
    callback();                    // signal "done with this chunk, send me the next one"
  }
}
```

---

## How it works — line by line

- `require('stream')` gives you the base classes: `Readable`, `Writable`, `Duplex`, and `Transform` — every stream in Node (files, HTTP, sockets, zlib) is built on top of these four.
- A **Readable** stream is a source. It has internal buffering, and you either listen for `'data'` events (flowing mode) or `for await...of` it (async iteration, covered in Topic 49).
- A **Writable** stream is a destination. Calling `.write(chunk)` pushes data into it. If the destination's internal buffer fills up faster than it can drain (e.g., a slow disk or network socket), `.write()` returns `false` — that's the **backpressure** signal telling you to slow down.
- `.pipe(destination)` connects a Readable directly to a Writable. It automatically pauses the Readable when the Writable signals it's overwhelmed, and resumes it once the Writable emits `'drain'`. This is backpressure handling done for you.
- `pipeline(...)` is the modern replacement for manually chaining `.pipe()` calls. It takes any number of streams, connects them in order, and — critically — calls its callback exactly once whether the chain succeeds or any stream errors. It also destroys every stream in the chain on failure, preventing memory leaks from streams left half-open.
- A **Transform** stream sits in the middle of a pipeline. You implement `_transform(chunk, encoding, callback)` — Node calls this method for every chunk that arrives. Inside it, you do your work, call `this.push(result)` to hand the transformed data downstream, then call `callback()` to tell Node "I'm ready for the next chunk."
- A **Duplex** stream is both readable and writable at the same time but the two sides are independent — a TCP socket is the classic example: you can read incoming data and write outgoing data on the same object without them being linked. A Transform is a special kind of Duplex where the writable side and readable side ARE linked — what you write in eventually comes out the readable side, transformed.

---

## Example 1 — basic

```js
// File: examples/basic-stream.js
const { Readable, Writable } = require('stream');

// ── Custom Readable stream — generates data instead of reading a file ──────
class NumberSource extends Readable {
  constructor(options) {
    super(options);         // must call super() first, like any class extending another
    this.current = 1;       // internal counter — this stream will emit numbers 1 to 5
  }

  _read() {
    // Node calls _read() whenever it wants more data from this stream
    if (this.current > 5) {
      this.push(null);      // pushing null signals "no more data" — ends the stream
      return;
    }
    this.push(`${this.current}\n`);  // push a chunk (string or Buffer) downstream
    this.current++;                  // move to the next number for the next _read() call
  }
}

// ── Custom Writable stream — logs whatever it receives ──────────────────────
class ConsoleLogger extends Writable {
  _write(chunk, encoding, callback) {
    // Node calls _write() for every chunk written to this stream
    process.stdout.write(`[received] ${chunk}`);  // print the chunk as-is
    callback();            // MUST call this — tells Node "ready for the next chunk"
  }
}

const source = new NumberSource();     // create the custom readable
const logger = new ConsoleLogger();    // create the custom writable

source.pipe(logger);                   // connect them — data flows source → logger

source.on('end', () => {
  console.log('Stream finished — no more data.');  // fires after push(null) is processed
});
```

---

## Example 2 — real world backend use case

```js
// File: src/services/exportUserLogs.js
// Streams a large user activity log from disk, filters + transforms each line,
// gzip-compresses it, and writes it to a new file — all WITHOUT loading the
// whole file into memory. This is the pattern for CSV exports, log processing,
// and file upload pipelines in real backend systems.

const fs = require('fs');
const zlib = require('zlib');
const { Transform, pipeline } = require('stream');

// ── Custom Transform: filters out log lines below a severity level ─────────
class SeverityFilter extends Transform {
  constructor(minSeverity, options) {
    super(options);
    this.minSeverity = minSeverity;   // e.g. 'ERROR' — only keep lines at/above this
    this.buffer = '';                 // holds partial lines that span chunk boundaries
  }

  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();          // append new data to any leftover partial line
    const lines = this.buffer.split('\n');    // split into complete lines
    this.buffer = lines.pop();                // last entry may be incomplete — save for next chunk

    for (const line of lines) {
      // only forward lines that mention the target severity (simple example filter)
      if (line.includes(this.minSeverity)) {
        this.push(line + '\n');               // push the kept line downstream
      }
    }
    callback();                                // ready for the next chunk
  }

  _flush(callback) {
    // called once, right before the stream ends — flush any leftover partial line
    if (this.buffer.includes(this.minSeverity)) {
      this.push(this.buffer + '\n');
    }
    callback();
  }
}

function exportFilteredLogs(userId, minSeverity) {
  const inputPath  = `./logs/${userId}-activity.log`;   // large raw log file
  const outputPath = `./exports/${userId}-filtered.log.gz`; // compressed, filtered result

  const readStream    = fs.createReadStream(inputPath);      // source: read the raw log
  const filterStream  = new SeverityFilter(minSeverity);      // middle: keep only matching lines
  const gzipStream    = zlib.createGzip();                    // middle: compress the output
  const writeStream   = fs.createWriteStream(outputPath);      // destination: final .gz file

  // pipeline() chains all four stages and handles backpressure + errors together
  pipeline(
    readStream,
    filterStream,
    gzipStream,
    writeStream,
    (err) => {
      if (err) {
        console.error(`Export failed for ${userId}:`, err.message);
        return;
      }
      console.log(`Export complete: ${outputPath}`);
    }
  );
}

module.exports = { exportFilteredLogs };

// Usage:
// exportFilteredLogs('user_42', 'ERROR');
// Even if the log file is 10GB, memory usage stays flat — data flows chunk by chunk.
```

---

## Common mistakes

### Mistake 1 — Reading a large file entirely into memory instead of streaming it

```js
// ❌ WRONG — loads the ENTIRE file into RAM before doing anything with it
const fs = require('fs');

function sendLargeFile(res, filePath) {
  const content = fs.readFileSync(filePath);  // blocks event loop, spikes memory
  res.end(content);                            // a 2GB file = 2GB in RAM at once
}

// ✅ CORRECT — stream the file straight to the response, chunk by chunk
const fs = require('fs');
const { pipeline } = require('stream');

function sendLargeFile(res, filePath) {
  const readStream = fs.createReadStream(filePath);   // reads in small chunks
  pipeline(readStream, res, (err) => {                // pipes chunks directly to the HTTP response
    if (err) console.error('File stream failed:', err.message);
  });
  // Memory usage stays flat (one chunk at a time), regardless of file size
}
```

### Mistake 2 — Chaining `.pipe()` manually and swallowing errors

```js
// ❌ WRONG — errors on any stream in the chain are NOT caught here,
// and if writeStream errors, readStream is left open (memory/file-handle leak)
const fs = require('fs');
const zlib = require('zlib');

fs.createReadStream('./input.txt')
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream('./output.txt.gz'));
// If the disk is full and writeStream errors, this throws an UNHANDLED error
// and readStream keeps its file descriptor open forever

// ✅ CORRECT — pipeline() catches errors from ANY stream and cleans up all of them
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream');

pipeline(
  fs.createReadStream('./input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('./output.txt.gz'),
  (err) => {
    if (err) {
      console.error('Compression pipeline failed:', err.message);
      return;
    }
    console.log('File compressed successfully.');
  }
);
```

### Mistake 3 — Ignoring the backpressure signal from `.write()`

```js
// ❌ WRONG — writing in a tight loop without checking the return value of write()
// This can buffer an unbounded amount of data in memory if the destination is slow
function writeMillionRows(writeStream, rows) {
  for (const row of rows) {
    writeStream.write(JSON.stringify(row) + '\n');  // ignores backpressure entirely
  }
  writeStream.end();
}

// ✅ CORRECT — respect the false return value; pause and wait for 'drain'
function writeMillionRows(writeStream, rows, callback) {
  let index = 0;

  function writeNext() {
    let ok = true;
    while (index < rows.length && ok) {
      const row = rows[index++];
      const isLast = index === rows.length;
      if (isLast) {
        writeStream.end(JSON.stringify(row) + '\n', callback);  // last chunk, then close
      } else {
        ok = writeStream.write(JSON.stringify(row) + '\n');     // false = buffer is full
      }
    }
    if (index < rows.length) {
      // buffer is full — wait for 'drain' before writing more, instead of forcing it
      writeStream.once('drain', writeNext);
    }
  }

  writeNext();
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a custom `Readable` stream called `WordStream` that:
1. Takes an array of strings (words) in its constructor
2. Emits each word one at a time via `_read()`, followed by a space
3. Pushes `null` once all words have been emitted, ending the stream
4. Pipe it to `process.stdout` and confirm all the words print out in order, space-separated

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a script that:
1. Creates a read stream on any large-ish text file you have (or generate one with a loop that writes 50,000 lines)
2. Creates a custom `Transform` stream called `LineCounter` that counts how many `\n` characters pass through it, without modifying the data itself (pass every chunk through unchanged with `this.push(chunk)`)
3. Uses `pipeline()` to connect: read stream → `LineCounter` → a write stream that copies the file to `output-copy.txt`
4. After the pipeline callback fires with no error, log the total line count the `LineCounter` counted

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `JsonlValidator` Transform stream and a small pipeline around it that:
1. Reads a `.jsonl` file (JSON Lines format — one JSON object per line) via a readable stream
2. In `_transform`, buffers incoming chunks and splits them into complete lines (handle lines split across chunk boundaries, like the `SeverityFilter` example)
3. For each complete line, tries `JSON.parse()` — if it succeeds AND the object has a `userId` field, push the line through unchanged; if parsing fails or `userId` is missing, skip the line and increment an internal `invalidCount` property instead of pushing it
4. Implements `_flush()` to handle any leftover partial line the same way
5. Pipes the validator's output to a write stream producing `valid-only.jsonl`
6. Uses `pipeline()` to wire it all together, and once the pipeline callback confirms success, logs `validator.invalidCount` to show how many bad lines were filtered out

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
FOUR CORE STREAM TYPES
  Readable   → a source you read FROM       (fs.createReadStream, http.IncomingMessage)
  Writable   → a destination you write TO   (fs.createWriteStream, http.ServerResponse)
  Duplex     → both, independently           (net.Socket — TCP connections)
  Transform  → Duplex where write-side feeds read-side, modified (zlib.createGzip, crypto streams)

CONNECTING STREAMS
  readable.pipe(writable)                    → simple connection, handles backpressure
  pipeline(readable, ...transforms, writable, callback)
                                              → PREFERRED — auto cleanup + single error callback

BACKPRESSURE
  writable.write(chunk) → true               → buffer had room, keep writing
  writable.write(chunk) → false              → buffer full, STOP and wait for 'drain' event
  .pipe() and pipeline() handle this for you automatically

CUSTOM STREAM METHODS TO IMPLEMENT
  Readable   → _read(size)                        → call this.push(chunk) or this.push(null) to end
  Writable   → _write(chunk, encoding, callback)   → process chunk, then call callback()
  Transform  → _transform(chunk, enc, callback)    → this.push(result), then callback()
  Transform  → _flush(callback)                    → optional — handle leftover buffered data at the end

KEY EVENTS
  Readable: 'data' (flowing mode), 'end', 'error', 'close'
  Writable: 'drain' (buffer emptied, safe to write again), 'finish', 'error'

MODES OF READABLE STREAMS
  Flowing mode → data pushed to you via 'data' event as fast as possible
  Paused mode  → you pull data manually with .read()
  Async iteration → for await (const chunk of readableStream) { ... }  (Topic 49)

GOTCHAS
  fs.readFileSync on large files      → loads everything into RAM, avoid for big files
  Manual .pipe() chains                → errors don't propagate cleanly, use pipeline() instead
  Forgetting callback() in _transform  → stream hangs forever, waiting for "next chunk" signal
  Ignoring write() === false           → unbounded memory growth if destination is slow

DEFAULT CHUNK SIZE (highWaterMark)
  Buffer streams  → 64 KB (65536 bytes) default
  Object mode     → 16 objects default
```

---

## Connected topics

- **17 — fs module — streams and large files** — `createReadStream`/`createWriteStream` are the most common Readable/Writable streams a backend dev uses daily; this topic is the deep-dive behind that surface API
- **25 — buffer module** — every chunk that flows through a stream (unless in object mode) is a `Buffer`; understanding Buffers is required to work with raw stream data
- **49 — streams as async iterables** — covers the modern `for await...of` way to consume Readable streams, an alternative to `.pipe()` and `'data'` event listeners shown here
