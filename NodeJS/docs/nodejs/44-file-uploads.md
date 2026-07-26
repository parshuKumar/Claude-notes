# 44 — File uploads

## What is this?

File upload is the process of a client sending a binary file (an image, a PDF, a video) to your backend over HTTP, using a special request format called `multipart/form-data` that lets a single request carry both text fields and raw file bytes together. The backend's job is to read that incoming stream of bytes, decide where it's safe to store it, and write it to disk (or cloud storage) without ever trusting anything the client claims about the file. Think of it like a courier dropping off a sealed package at a warehouse — the warehouse doesn't just believe the label on the box; it inspects the contents, assigns its own internal tracking ID, and puts it on a shelf it controls, never letting the sender dictate exactly where in the warehouse it gets placed.

## Why does it matter for backend development?

Almost every real product needs file uploads — profile pictures, resumes, chat attachments, product images, CSV imports. Getting this wrong is one of the most common ways backend apps get breached: a filename like `../../etc/passwd` or `../server.js` sent as the upload name can let an attacker overwrite files outside the intended folder (**path traversal**), an unbounded upload can fill your disk or crash your process (denial of service), and trusting a client-supplied file extension can let someone upload an executable `.php` or `.exe` file disguised as a `.jpg`. A backend developer must understand multipart parsing, streaming large files without loading them fully into memory, and defensive filename/path handling — this is non-negotiable production knowledge, not an edge case.

---

## Syntax / API

```js
// npm install busboy — the industry-standard streaming multipart parser
// (Express's most popular upload middleware, multer, is built directly on top of busboy)
const http    = require('http');
const busboy  = require('busboy');
const fs      = require('fs');
const path    = require('path');
const crypto  = require('crypto');

// Directory where uploaded files will actually be stored — resolved once, at startup
const UPLOAD_DIR = path.join(__dirname, 'uploads');

const server = http.createServer((req, res) => {
  // Only handle multipart POST requests here — everything else is a 404 in this example
  if (req.method !== 'POST' || !req.headers['content-type']?.startsWith('multipart/form-data')) {
    res.writeHead(404);
    return res.end('Not found');
  }

  // busboy reads the raw request stream and splits it into fields + files for you
  const bb = busboy({
    headers: req.headers,               // busboy needs the boundary from content-type header
    limits: { fileSize: 5 * 1024 * 1024 }, // hard cap: 5MB per file — never trust unbounded uploads
  });

  bb.on('file', (fieldName, fileStream, info) => {
    // fieldName → the <input name="..."> from the form, e.g. 'avatar'
    // fileStream → a Readable stream of the raw file bytes — never fully in memory at once
    // info.filename → CLIENT-SUPPLIED name — NEVER trust or use this for the real path
    const safeFileName = crypto.randomUUID() + path.extname(info.filename); // generate our own name
    const destinationPath = path.join(UPLOAD_DIR, safeFileName);            // always inside UPLOAD_DIR

    // Pipe the incoming file stream straight to disk in chunks (Topic 17)
    fileStream.pipe(fs.createWriteStream(destinationPath));
  });

  bb.on('field', (fieldName, value) => {
    // Non-file text fields in the same form, e.g. a caption or description
    console.log(`Field [${fieldName}]:`, value);
  });

  bb.on('close', () => {
    // Fires once all files and fields have been fully parsed and written
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ message: 'Upload complete' }));
  });

  bb.on('error', (err) => {
    // Fires on malformed multipart data, or if a file exceeds the size limit
    res.writeHead(400);
    res.end(`Upload failed: ${err.message}`);
  });

  req.pipe(bb); // feed the raw incoming request bytes into busboy
});

server.listen(3000);
```

---

## How it works — line by line

When a browser submits a form with `enctype="multipart/form-data"`, it doesn't just send raw JSON — it sends the request body as a series of **parts**, each separated by a random "boundary" string that's declared in the `Content-Type` header (e.g. `multipart/form-data; boundary=----abc123`). Each part has its own small header block describing whether it's a text field or a file (and the file's original name and MIME type), followed by the actual raw bytes, followed by the boundary marking the start of the next part.

The raw Node `req` object is a **Readable stream of Buffer chunks** — it never gives you the whole body as one string, it hands you pieces of binary data as they arrive over the network (Topic 25 covers Buffers, Topic 17 covers streams generally). Parsing the boundary markers, headers, and encoding by hand is fiddly and easy to get wrong, so in practice everyone uses a battle-tested streaming parser like `busboy`. It reads the incoming `req` stream chunk-by-chunk, recognizes boundary markers, and emits a `'file'` event (with its own Readable stream for that file's bytes) or a `'field'` event (with the plain value) for each part it finds — all without ever buffering the entire request body in memory. That last point matters enormously: a naive parser that waits for the whole body before processing could be handed a 10GB "file" and crash your server; a streaming parser processes it incrementally and can be size-limited safely.

---

## Example 1 — basic

```js
// File: upload-basic.js
// A minimal single-file upload endpoint using raw http + busboy

const http   = require('http');
const busboy = require('busboy');
const fs     = require('fs');
const path   = require('path');

const uploadDir = path.join(__dirname, 'uploads'); // where files land, built with __dirname (Topic 06)

// Make sure the folder exists before the server starts accepting uploads
if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true }); // create nested folders if needed
}

const server = http.createServer((req, res) => {
  if (req.method !== 'POST') {
    res.writeHead(405);           // 405 = Method Not Allowed
    return res.end('POST only');
  }

  const bb = busboy({ headers: req.headers }); // reads boundary from content-type header
  let savedFileName = null;                    // track what we actually saved

  bb.on('file', (fieldName, fileStream, info) => {
    const ext = path.extname(info.filename);          // pull the extension, e.g. '.png'
    savedFileName = `upload-${Date.now()}${ext}`;      // simple unique name — timestamp based
    const outPath = path.join(uploadDir, savedFileName); // always inside uploadDir

    const writeStream = fs.createWriteStream(outPath);   // stream destination on disk
    fileStream.pipe(writeStream);                        // chunks flow file → disk directly
  });

  bb.on('close', () => {
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`Saved as: ${savedFileName}`); // tell the client what happened
  });

  req.pipe(bb); // start the parsing — this triggers everything above
});

server.listen(3000, () => console.log('Upload server on http://localhost:3000'));

// Test with curl:
// curl -F "avatar=@./photo.jpg" http://localhost:3000
```

---

## Example 2 — real world backend use case

```js
// File: routes/avatar-upload.js
// A realistic profile-picture upload handler: validates type/size, generates a
// collision-proof filename, and defends against path traversal before writing.

const http     = require('http');
const busboy   = require('busboy');
const fs       = require('fs');
const path     = require('path');
const crypto   = require('crypto');

const UPLOAD_DIR   = path.join(__dirname, '..', 'storage', 'avatars'); // fixed, server-controlled root
const ALLOWED_MIME = new Set(['image/jpeg', 'image/png', 'image/webp']); // whitelist, not blacklist
const MAX_BYTES    = 2 * 1024 * 1024; // 2MB — reasonable cap for an avatar image

if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR, { recursive: true });

// Confirms a resolved path still lives inside UPLOAD_DIR — the core traversal defense
function isInsideUploadDir(candidatePath) {
  const resolved = path.resolve(candidatePath);           // normalize away any '..' segments
  const rootWithSep = path.resolve(UPLOAD_DIR) + path.sep; // require it to be a real child, not a sibling
  return resolved.startsWith(rootWithSep);
}

function handleAvatarUpload(req, res, userId) {
  const bb = busboy({
    headers: req.headers,
    limits: { fileSize: MAX_BYTES, files: 1 }, // one file, hard size cap enforced by busboy itself
  });

  let responded = false;

  bb.on('file', (fieldName, fileStream, info) => {
    const { mimeType } = info; // MIME type as reported by the client — verified below, never trusted blindly

    if (!ALLOWED_MIME.has(mimeType)) {
      fileStream.resume();               // drain the stream so the request can still close cleanly
      if (!responded) {
        responded = true;
        res.writeHead(415);              // 415 = Unsupported Media Type
        res.end(JSON.stringify({ error: 'Only JPEG, PNG, or WEBP images are allowed' }));
      }
      return;
    }

    // Build our OWN filename — the client's original name is never used in the path
    const extByMime = { 'image/jpeg': '.jpg', 'image/png': '.png', 'image/webp': '.webp' };
    const generatedName = `${userId}-${crypto.randomUUID()}${extByMime[mimeType]}`;
    const destinationPath = path.join(UPLOAD_DIR, generatedName);

    // Defense in depth: even though generatedName has no traversal chars, verify anyway
    if (!isInsideUploadDir(destinationPath)) {
      fileStream.resume();
      if (!responded) {
        responded = true;
        res.writeHead(400);
        res.end(JSON.stringify({ error: 'Invalid upload path' }));
      }
      return;
    }

    const writeStream = fs.createWriteStream(destinationPath);
    fileStream.pipe(writeStream);

    writeStream.on('finish', () => {
      if (responded) return; // avoid double-responding if size limit already fired
      responded = true;
      // In a real app: save `generatedName` against userId in the database here
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ userId, avatarFile: generatedName }));
    });
  });

  bb.on('error', (err) => {
    if (responded) return;
    responded = true;
    res.writeHead(400);
    res.end(JSON.stringify({ error: `Upload rejected: ${err.message}` }));
  });

  // busboy enforces limits.fileSize by emitting 'error' — some versions instead
  // emit a 'limit' event on the file stream itself; handling both is safest in production
  bb.on('filesLimit', () => {
    if (responded) return;
    responded = true;
    res.writeHead(400);
    res.end(JSON.stringify({ error: 'Too many files in one request' }));
  });

  req.pipe(bb);
}

module.exports = { handleAvatarUpload };
```

---

## Common mistakes

### Mistake 1 — Trusting the client-supplied filename to build the storage path

```js
// ❌ WRONG — info.filename comes straight from the client and can contain '../' segments
// A malicious upload named '../../../../etc/cron.d/evil' can escape UPLOAD_DIR entirely
bb.on('file', (fieldName, fileStream, info) => {
  const outPath = path.join(UPLOAD_DIR, info.filename); // path traversal vulnerability
  fileStream.pipe(fs.createWriteStream(outPath));
});

// ✅ CORRECT — generate the filename yourself, only keep the extension from the client
bb.on('file', (fieldName, fileStream, info) => {
  const ext = path.extname(info.filename).toLowerCase();     // keep only the extension
  const safeName = `${crypto.randomUUID()}${ext}`;            // server controls the whole name
  const outPath = path.join(UPLOAD_DIR, safeName);             // guaranteed inside UPLOAD_DIR
  fileStream.pipe(fs.createWriteStream(outPath));
});
```

### Mistake 2 — Not limiting upload size, letting a request fill the disk or crash the process

```js
// ❌ WRONG — no size limit means a client can stream an effectively unlimited file,
// filling the server's disk (or memory, if buffered) and taking the app down
const bb = busboy({ headers: req.headers });

// ✅ CORRECT — set a hard limit; busboy stops the stream and emits an error past it
const bb = busboy({
  headers: req.headers,
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB hard cap — tune per use case (avatar vs. video)
    files: 1,                  // reject requests trying to sneak in multiple files
  },
});

bb.on('error', (err) => {
  res.writeHead(400);
  res.end(`Upload rejected: ${err.message}`); // client gets a clear reason, server stays safe
});
```

### Mistake 3 — Trusting the client-reported MIME type or extension as proof of file content

```js
// ❌ WRONG — info.mimeType is whatever the CLIENT'S browser/HTTP client claims it is;
// an attacker can rename evil.exe to photo.jpg and set Content-Type: image/jpeg by hand
bb.on('file', (fieldName, fileStream, info) => {
  fileStream.pipe(fs.createWriteStream(path.join(UPLOAD_DIR, info.filename)));
  // No check at all — anything gets saved and could later be served/executed
});

// ✅ CORRECT — whitelist allowed MIME types, AND verify actual file bytes for anything sensitive
const ALLOWED_MIME = new Set(['image/jpeg', 'image/png', 'image/webp']);

bb.on('file', (fieldName, fileStream, info) => {
  if (!ALLOWED_MIME.has(info.mimeType)) {
    fileStream.resume();          // drain so the connection can close normally
    return res.end('Rejected: unsupported file type');
  }
  // For high-stakes apps, also inspect the first few bytes ("magic numbers") of the
  // stream — e.g. real JPEGs start with 0xFFD8 — using a library like `file-type`
  fileStream.pipe(fs.createWriteStream(path.join(UPLOAD_DIR, `${crypto.randomUUID()}.jpg`)));
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a plain HTTP server (`http.createServer`) with `busboy` that:
1. Accepts a single file upload on `POST /upload`.
2. Saves it into an `uploads/` folder next to the script, built using `__dirname`.
3. Generates the saved filename as `file-<Date.now()><original-extension>` — never using the client's raw name directly as the path.
4. Responds with a JSON body `{ savedAs: "<generated filename>" }` once the write finishes.

Test it with: `curl -F "file=@./somefile.txt" http://localhost:3000/upload`

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `createUploadHandler({ uploadDir, allowedExtensions, maxBytes })` that returns an HTTP request handler which:
1. Uses `busboy` with `limits.fileSize` set to `maxBytes`.
2. Rejects (with status `415`) any file whose extension (lowercased) is not in `allowedExtensions`.
3. Generates a safe filename using `crypto.randomUUID()` plus the original extension.
4. Verifies the fully resolved destination path is still inside `uploadDir` before writing (reject with `400` if not — even though your generated name should always be safe, write the check anyway).
5. On success, responds with `{ savedAs, sizeLimitBytes: maxBytes }`.
6. On a `busboy` `'error'` event (e.g. file too large), responds with status `400` and the error message.

Test it by wiring it into an `http.createServer` for two different configs — one for images (`.jpg`, `.png`) with a 2MB cap, one for documents (`.pdf`) with a 10MB cap — and confirm each rejects the other's file type.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `ChunkedUploadManager` for resumable large-file uploads (the pattern real apps use for big video/dataset uploads over unreliable connections):
1. `startUpload(uploadId, fileName, totalChunks)` — records upload metadata in memory (e.g. a `Map`) and creates a temp directory `temp/<uploadId>/` for chunk parts.
2. `writeChunk(uploadId, chunkIndex, chunkBuffer)` — validates `uploadId` exists and `chunkIndex` is a non-negative integer within `totalChunks`, then writes the chunk to `temp/<uploadId>/chunk-<chunkIndex>` using `fs.createWriteStream` (reject with a thrown error if `uploadId` looks suspicious, e.g. contains `..` or `/`).
3. `isComplete(uploadId)` — returns true only once every chunk index from `0` to `totalChunks - 1` exists on disk.
4. `finalizeUpload(uploadId, finalDir)` — once complete, streams and concatenates all chunk files in order into a single file inside `finalDir` (server-controlled, not client-supplied), generating a fresh UUID-based final filename, then deletes the temp chunk directory.
5. Guard every path you build (`uploadId`, `chunkIndex`) against traversal — reject anything that isn't validated against a strict pattern (e.g. `uploadId` must match `/^[a-zA-Z0-9_-]+$/`) before it ever touches `path.join`.

Simulate a 4-chunk upload end-to-end and confirm the final assembled file matches the original content, then confirm calling `finalizeUpload` on an incomplete upload throws an error.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
MULTIPART/FORM-DATA
  Content-Type: multipart/form-data; boundary=----abc123
  Body = parts separated by the boundary string, each part is either:
    - a text field (fieldName + value)
    - a file (fieldName + filename + mimeType + raw bytes)

READING THE UPLOAD (busboy — npm install busboy)
  const bb = busboy({ headers: req.headers, limits: { fileSize, files } });
  bb.on('file',  (fieldName, fileStream, info) => {...})  → info.filename, info.mimeType
  bb.on('field', (fieldName, value) => {...})              → plain text fields
  bb.on('close', () => {...})                               → all parts fully parsed
  bb.on('error', (err) => {...})                            → malformed data / size exceeded
  req.pipe(bb);                                             → start parsing

STORING FILES SAFELY
  - NEVER use info.filename directly as (or inside) the storage path
  - Generate your own name: crypto.randomUUID() + extension
  - Always path.join(UPLOAD_DIR, generatedName) — never string-concatenate
  - Enforce limits.fileSize on busboy itself — don't rely on checking after the fact
  - Whitelist allowed MIME types / extensions — never blacklist

PATH TRAVERSAL PREVENTION
  const resolved = path.resolve(candidatePath);
  const isSafe = resolved.startsWith(path.resolve(UPLOAD_DIR) + path.sep);
  - Reject '../', absolute paths, and null bytes in any client-influenced segment
  - Defense in depth: check even when you generated the name yourself

BINARY STREAMS
  fileStream is a Readable stream of Buffer chunks (Topic 25, Topic 17)
  fileStream.pipe(fs.createWriteStream(path))  → streams to disk, memory stays flat
  fileStream.resume()                          → drain a rejected file so req can still close

GOTCHAS
  - A "file too large" error can fire mid-stream — always handle bb 'error' AND clean up partial files
  - MIME type from the client is a claim, not proof — verify magic bytes for high-stakes uploads
  - Always create UPLOAD_DIR with { recursive: true } before the server starts accepting requests
  - Don't forget files.length / files limit — a form can smuggle multiple files in one field
```

---

## Connected topics

- **17 — fs module (streams and large files)** — `createWriteStream()` and `.pipe()` are exactly how uploaded bytes get written to disk without loading the whole file into memory.
- **25 — buffer module** — the raw chunks `busboy` and `req` hand you are `Buffer` instances; understanding encodings and byte handling explains what's actually flowing through the stream.
- **75 — Input validation and sanitization** — filename/MIME whitelisting and path-traversal checks here are a specific application of the general "never trust user input" principle covered in depth there.
