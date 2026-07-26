# 17 — fs module (streams and large files)

## What is this?

Streams are Node's way of processing data **in small chunks, as it arrives**, instead of loading the entire file into memory first. `fs.createReadStream()` reads a file piece by piece, and `fs.createWriteStream()` writes a file piece by piece — and `.pipe()` connects the two so chunks flow straight from source to destination. Think of it like a garden hose filling a bucket versus carrying the entire river in one bucket: a hose moves water continuously in a manageable flow, while trying to scoop an entire river at once would crush you.

## Why does it matter for backend development?

Real backend servers deal with files that can be gigabytes in size — video uploads, database backups, CSV exports, log files. If you use `fs.readFile()` on a 2GB file, Node tries to load all 2GB into RAM as a single Buffer, which can crash your process with an out-of-memory (OOM) error and will freeze your event loop while it happens. Streams solve this by processing data in small chunks (default 64KB), keeping memory usage flat regardless of file size. Every backend developer who handles file uploads, downloads, log processing, or data exports needs streams — it is the difference between a server that survives production traffic and one that falls over the first time someone uploads a large file.

---

## Syntax / API

```js
// Import the fs module — streams live on the same module as readFile/writeFile
const fs = require('fs');

// ── Creating a readable stream ──────────────────────────────────────────────
// Reads the file in chunks instead of all at once
const readStream = fs.createReadStream('./uploads/large-video.mp4', {
  encoding: null,        // null = raw Buffer chunks (use 'utf8' for text files)
  highWaterMark: 65536,  // chunk size in bytes — default is 64KB (64 * 1024)
});

// ── Creating a writable stream ──────────────────────────────────────────────
// Writes data to disk in chunks instead of all at once
const writeStream = fs.createWriteStream('./uploads/copy-of-video.mp4', {
  flags: 'w',            // 'w' = create/overwrite, 'a' = append
});

// ── Connecting them with pipe() ─────────────────────────────────────────────
// Every chunk read from readStream is automatically written to writeStream
// pipe() also handles backpressure — pausing the read if the write is slower
readStream.pipe(writeStream);

// ── Listening for stream events ─────────────────────────────────────────────
readStream.on('data', (chunk) => {
  // Fires once per chunk received — chunk.length tells you the chunk size
  console.log(`Received ${chunk.length} bytes`);
});

readStream.on('end', () => {
  // Fires once, after the last chunk has been read
  console.log('Finished reading the file');
});

readStream.on('error', (err) => {
  // Fires if the file is missing, permissions fail, or disk read fails
  console.error('Read stream error:', err.message);
});

writeStream.on('finish', () => {
  // Fires once all data has been flushed to disk
  console.log('Finished writing the file');
});
```

---

## How it works — line by line

`fs.createReadStream(filePath)` does not read the file immediately in full. It opens a file handle and returns a `Readable` stream object. Data only starts flowing once you attach a `'data'` listener or call `.pipe()` — until then, the stream sits paused.

`fs.createWriteStream(filePath)` opens a file handle for writing and returns a `Writable` stream object. You feed it data with `.write(chunk)`, and it flushes each chunk to disk, then signals `'finish'` when done.

`.pipe(destination)` is the glue: it takes every chunk your readable stream emits and automatically calls `.write()` on the writable stream for you. It also manages **backpressure** — if the destination (say, a slow network connection or slow disk) can't keep up, `.pipe()` automatically pauses the source stream until the destination catches up, then resumes it. This is the exact mechanism that prevents memory from ballooning: chunks are never held in memory waiting to be written, because the flow is throttled to match the slowest link.

The `highWaterMark` option controls the internal buffer size — how much data can sit in memory at once before the stream pauses reading more. A smaller value uses less memory per chunk but requires more chunks (more overhead); a larger value uses more memory but processes fewer, bigger chunks. The default (64KB) is a good balance for most cases.

Because streams are `EventEmitter`s (Topic 19), you attach listeners with `.on(eventName, callback)`. The key events are `'data'` (a chunk arrived), `'end'` (no more data, read side), `'finish'` (all data flushed, write side), and `'error'` (something went wrong on either side) — and you must always handle `'error'`, because an unhandled stream error can crash your entire process.

---

## Example 1 — basic

```js
// File: examples/copy-file-stream.js
// Copies a file from one location to another using streams — works for files of any size

const fs = require('fs');

const sourcePath      = './data/report.csv';        // file we want to copy
const destinationPath = './data/report-backup.csv'; // where the copy goes

// Create the readable side — starts opening the source file
const readStream = fs.createReadStream(sourcePath);

// Create the writable side — starts opening the destination file
const writeStream = fs.createWriteStream(destinationPath);

// Track total bytes processed, just for visibility
let totalBytes = 0;

// 'data' fires every time a new chunk is read from disk
readStream.on('data', (chunk) => {
  totalBytes += chunk.length;               // add this chunk's size to the running total
  console.log(`Read chunk: ${chunk.length} bytes (total so far: ${totalBytes})`);
});

// 'end' fires once, after the entire source file has been read
readStream.on('end', () => {
  console.log(`Finished reading. Total size: ${totalBytes} bytes`);
});

// Always handle errors on both streams — missing file, bad permissions, full disk, etc.
readStream.on('error', (err) => {
  console.error('Failed to read source file:', err.message);
});
writeStream.on('error', (err) => {
  console.error('Failed to write destination file:', err.message);
});

// 'finish' fires once all chunks have been flushed to the destination file
writeStream.on('finish', () => {
  console.log('Copy complete:', destinationPath);
});

// Pipe connects the two streams — chunks flow from readStream straight into writeStream
readStream.pipe(writeStream);
```

---

## Example 2 — real world backend use case

```js
// File: src/controllers/downloadController.js
// Streams a large file (e.g. a generated report or user backup) to an HTTP response
// without ever loading the whole file into memory — critical for GB-sized files.

const fs   = require('fs');
const path = require('path');

// This function is used inside an Express-style route handler: GET /api/files/:fileId/download
function downloadFile(req, res) {
  const fileId  = req.params.fileId;                          // e.g. "invoice_2026_07"
  const filePath = path.join(__dirname, '..', 'storage', `${fileId}.pdf`); // build safe path

  // Check the file exists before attempting to stream it
  fs.stat(filePath, (statErr, stats) => {
    if (statErr) {
      // File missing on disk — respond with 404, never crash the server
      return res.status(404).json({ error: 'File not found' });
    }

    // Set headers BEFORE streaming so the client knows what's coming
    res.setHeader('Content-Type', 'application/pdf');            // tell browser it's a PDF
    res.setHeader('Content-Length', stats.size);                 // total file size in bytes
    res.setHeader('Content-Disposition', `attachment; filename="${fileId}.pdf"`); // force download

    // Create the read stream for the actual file on disk
    const fileStream = fs.createReadStream(filePath);

    // If reading fails mid-stream (disk error, corrupted file), fail gracefully
    fileStream.on('error', (streamErr) => {
      console.error(`Stream error for ${filePath}:`, streamErr.message);
      // Only send an error response if headers haven't already been sent
      if (!res.headersSent) {
        res.status(500).json({ error: 'Failed to stream file' });
      }
    });

    // Pipe the file directly into the HTTP response —
    // Node streams the file to the client in chunks as fast as their connection allows
    // Memory usage stays flat even if the PDF is 500MB, because chunks are never buffered whole
    fileStream.pipe(res);
  });
}

module.exports = { downloadFile };

// Why this matters:
// - fs.readFile(filePath) here would load the ENTIRE pdf into RAM before sending anything
// - With 100 concurrent downloads of a 200MB file, readFile could OOM-crash the server
// - createReadStream + pipe(res) keeps memory flat no matter how large the file or how many
//   concurrent downloads are happening
```

---

## Common mistakes

### Mistake 1 — Using fs.readFile on large files instead of streaming

```js
// ❌ WRONG — loads the ENTIRE file into memory as one Buffer before you can do anything with it
const fs = require('fs');

fs.readFile('./backups/database-dump.sql', (err, data) => {
  // If database-dump.sql is 3GB, Node tries to allocate a 3GB Buffer
  // This can crash the process with "JavaScript heap out of memory"
  console.log(data.length);
});

// ✅ CORRECT — stream the file in small chunks, memory usage stays flat
const readStream = fs.createReadStream('./backups/database-dump.sql');

let totalBytes = 0;
readStream.on('data', (chunk) => {
  totalBytes += chunk.length;   // process each chunk without ever holding the whole file
});
readStream.on('end', () => {
  console.log('Total bytes:', totalBytes);
});
```

### Mistake 2 — Forgetting to handle the 'error' event on streams

```js
// ❌ WRONG — no error listener means an unhandled 'error' event crashes the entire process
const fs = require('fs');

const readStream = fs.createReadStream('./data/missing-file.csv');
readStream.pipe(fs.createWriteStream('./data/copy.csv'));
// If missing-file.csv doesn't exist, Node throws an uncaught error and kills the process

// ✅ CORRECT — always attach 'error' listeners to both the read and write stream
const readStream2  = fs.createReadStream('./data/missing-file.csv');
const writeStream2 = fs.createWriteStream('./data/copy.csv');

readStream2.on('error', (err) => {
  console.error('Read failed:', err.message);   // handled gracefully, process stays alive
});
writeStream2.on('error', (err) => {
  console.error('Write failed:', err.message);
});

readStream2.pipe(writeStream2);
```

### Mistake 3 — Manually piping multiple streams without backpressure handling

```js
// ❌ WRONG — writing chunks manually in a 'data' handler ignores backpressure signals,
// so if the destination is slower than the source, chunks pile up in memory unbounded
const fs = require('fs');

const readStream  = fs.createReadStream('./data/huge-log.txt');
const writeStream = fs.createWriteStream('./data/huge-log-copy.txt');

readStream.on('data', (chunk) => {
  writeStream.write(chunk);   // ignores the return value — no backpressure control
  // If write() returns false (internal buffer full), this loop keeps shoving more data in
});

// ✅ CORRECT — let pipe() do it, or use stream.pipeline() (Topic 26) for multi-stream chains
// pipe() automatically pauses the read stream when the write stream's buffer is full
readStream.pipe(writeStream);

// For chaining multiple streams (e.g. through a compression transform), prefer pipeline()
// because it also handles cleanup and error propagation across the whole chain:
const { pipeline } = require('stream');
const zlib = require('zlib');

pipeline(
  fs.createReadStream('./data/huge-log.txt'),
  zlib.createGzip(),                          // compress chunks as they pass through
  fs.createWriteStream('./data/huge-log.txt.gz'),
  (err) => {
    if (err) console.error('Pipeline failed:', err.message);
    else console.log('Pipeline succeeded');
  }
);
```

---

## Practice exercises

### Exercise 1 — easy

Create two files: a source text file with a few lines of content, and a script that:
1. Uses `fs.createReadStream()` to read the source file
2. Uses `fs.createWriteStream()` to write to a new destination file
3. Connects them with `.pipe()`
4. Logs `"Copy started"` before the pipe and `"Copy finished"` when the write stream emits `'finish'`
5. Handles errors on both streams by logging them (do not let the process crash)

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `countLines(filePath, callback)` that:
1. Opens the given file with `fs.createReadStream()` using `encoding: 'utf8'` so chunks arrive as strings, not Buffers
2. Counts the total number of newline characters (`\n`) across all chunks as they arrive (do NOT read the whole file into one string first — count incrementally, chunk by chunk)
3. Calls `callback(null, lineCount)` when the stream ends
4. Calls `callback(err)` if the read stream emits an error
5. Test it against a file with a known number of lines and verify the count is correct

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `streamFileToClient(filePath, res)` function meant for use in an HTTP server (you can simulate `res` with a fake writable object for testing, or use Node's `http` module) that:
1. Uses `fs.stat()` first to check the file exists and get its size — if it doesn't exist, call `res.end('Not found')` and return
2. Creates a read stream for the file
3. Supports **range requests** — if a `range` header like `bytes=0-1023` is present (passed in as a `rangeHeader` argument), only stream that byte range using the `start` and `end` options of `fs.createReadStream()`
4. Without a range header, streams the entire file
5. Handles read-stream errors by logging them and calling `res.end()` safely (checking a `res.headersSent`-style flag first)
6. Pipes the resulting stream to `res`

This mimics how real servers support partial downloads (e.g. resuming a paused video).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHY STREAMS EXIST
  fs.readFile()   → loads ENTIRE file into memory at once → risk of OOM on large files
  fs.createReadStream() → reads file in small chunks (default 64KB) → memory stays flat

CREATING STREAMS
  fs.createReadStream(path, options)
    options.encoding      → 'utf8' for text chunks, omit/null for raw Buffer chunks
    options.highWaterMark → chunk size in bytes (default 65536 = 64KB)
    options.start / end   → byte range to read (used for partial/range requests)

  fs.createWriteStream(path, options)
    options.flags → 'w' (overwrite, default), 'a' (append)

CONNECTING STREAMS
  readStream.pipe(writeStream)
    - moves chunks automatically from source to destination
    - handles backpressure automatically (pauses source if destination is slow)
    - use stream.pipeline() instead of raw pipe() when chaining multiple streams
      (transform/compress/encrypt) — pipeline() also propagates errors correctly

KEY EVENTS
  Readable stream:  'data' (chunk arrived), 'end' (done reading), 'error'
  Writable stream:  'drain' (buffer ready for more), 'finish' (done writing), 'error'

RULES
  - ALWAYS attach an 'error' listener on every stream — unhandled errors crash the process
  - NEVER use fs.readFile/readFileSync on files that could be large or user-uploaded
  - Prefer pipe() or pipeline() over manually calling .write() in a 'data' handler
  - Use range options (start/end) to support resumable/partial downloads

WHEN TO USE STREAMS
  - Serving large file downloads over HTTP (videos, PDFs, backups)
  - Handling file uploads (multipart form data)
  - Copying/compressing/encrypting large files
  - Processing large CSV/log files line by line without loading them fully
```

---

## Connected topics

- **15 — fs module (callbacks)** — the non-streaming API (`readFile`/`writeFile`) this topic is the memory-safe alternative to for large files
- **16 — fs module (promises API)** — same operations with async/await; streams are still preferred over `fs.promises.readFile` for large files
- **26 — stream module in depth** — the full Readable/Writable/Transform/Duplex API, `pipeline()`, and backpressure mechanics referenced throughout this doc
