# 14 — path module

## What is this?

The `path` module is Node's built-in toolkit for working with file and directory paths correctly, no matter which operating system the code runs on. Think of it like a universal translator for addresses — Windows writes paths with backslashes (`C:\Users\data`) and Linux/Mac use forward slashes (`/home/data`), and `path` quietly handles that difference for you so you never have to think about it. Instead of gluing strings together by hand, you hand `path` the pieces and it assembles a correct, clean path for whatever machine the code is running on.

---

## Why does it matter for backend development?

Backend code constantly touches the filesystem — reading config files, saving uploaded avatars, serving static assets, writing log files, loading templates. Every one of those operations needs a correct file path, and paths built by hand with string concatenation (`+`) break in subtle ways: double slashes, missing slashes, wrong separators on Windows, or `..` segments that don't resolve the way you expect. A backend that works perfectly on your Mac but crashes when deployed to a Linux server is often a `path` module problem. Every serious Node backend — Express apps, CLI tools, build scripts — uses `path.join()` and `path.resolve()` dozens of times, because they are the only reliable way to build a path that works everywhere.

---

## Syntax / API

```js
// path is a built-in core module — no npm install needed
const path = require('path');

// ── join() — glues path segments together, normalizes slashes ───────────────
path.join('users', 'uploads', 'avatar.png');
// → "users/uploads/avatar.png" (or "users\\uploads\\avatar.png" on Windows)

// ── resolve() — same as join, but ALWAYS returns an ABSOLUTE path ───────────
path.resolve('users', 'uploads', 'avatar.png');
// → "/current/working/dir/users/uploads/avatar.png"

// ── dirname() — returns the folder portion of a path ────────────────────────
path.dirname('/app/src/config/database.js');
// → "/app/src/config"

// ── basename() — returns the last segment (the file name) ───────────────────
path.basename('/app/src/config/database.js');
// → "database.js"

// ── basename() with an extension argument strips that extension too ────────
path.basename('/app/src/config/database.js', '.js');
// → "database"

// ── extname() — returns just the extension, including the dot ──────────────
path.extname('database.js');
// → ".js"

// ── sep — the OS-specific path separator character ──────────────────────────
path.sep;
// → "/" on Linux/Mac, "\\" on Windows

// ── parse() — breaks a full path into a labeled object ──────────────────────
path.parse('/app/uploads/avatar.png');
// → { root: '/', dir: '/app/uploads', base: 'avatar.png', ext: '.png', name: 'avatar' }

// ── isAbsolute() — checks if a path is already absolute ────────────────────
path.isAbsolute('/app/uploads');   // → true
path.isAbsolute('uploads');        // → false
```

---

## How it works — line by line

- `path.join(...segments)` takes any number of string pieces, glues them together with the correct separator for the current OS, and cleans up the result — it collapses `a/b/../c` into `a/c` and removes duplicate slashes. It does **not** care whether the result is absolute or relative; it just joins whatever you give it.
- `path.resolve(...segments)` works like `join()`, but it always processes the segments as if the last one is the destination and works backward, and if none of the segments start from an absolute path, it prepends `process.cwd()` (the directory Node was launched from). The result is always a full, absolute path.
- `path.dirname(somePath)` looks at the string and strips off everything after the last separator, leaving just the containing folder.
- `path.basename(somePath)` does the opposite — it keeps only what comes after the last separator, the actual file (or folder) name. Passing a second argument (an extension) removes that suffix from the result too.
- `path.extname(somePath)` finds the last `.` in the file name and returns everything from that dot onward. A file with no dot, or a dot only at the start (like `.gitignore`), returns an empty string.
- `path.sep` is just a single character constant (`/` or `\`) — useful when you need to split a path string manually or display it to a user, but you almost never need it for building paths (that is what `join`/`resolve` are for).
- `path.parse(somePath)` runs `dirname`, `basename`, and `extname` all at once and hands back the pieces as one object — handy when you need several parts of the same path.

---

## Example 1 — basic

```js
// File: src/scripts/pathDemo.js

const path = require('path');   // load the built-in path module

// A sample path to a file, as if it came from a database record
const filePath = '/app/uploads/invoices/invoice_2026_07.pdf';

// Get just the folder the file lives in
const folder = path.dirname(filePath);
console.log('Folder   :', folder);
// → "/app/uploads/invoices"

// Get just the file name (with extension)
const fileName = path.basename(filePath);
console.log('File name:', fileName);
// → "invoice_2026_07.pdf"

// Get the file name WITHOUT the extension
const nameOnly = path.basename(filePath, path.extname(filePath));
console.log('Name only:', nameOnly);
// → "invoice_2026_07"

// Get just the extension
const extension = path.extname(filePath);
console.log('Extension:', extension);
// → ".pdf"

// Join pieces safely into a new path — no manual slashes needed
const backupPath = path.join(folder, 'backups', fileName);
console.log('Backup   :', backupPath);
// → "/app/uploads/invoices/backups/invoice_2026_07.pdf"

// Show the OS-specific separator (rarely needed directly, shown for learning)
console.log('Separator:', JSON.stringify(path.sep));
// → "/" on Linux/Mac, "\\" on Windows
```

---

## Example 2 — real world backend use case

```js
// File: src/services/avatarService.js
// A realistic pattern: saving a user's uploaded avatar and computing its public URL.
// This is the kind of code every backend dev writes for file-upload features.

const path = require('path');
const fs   = require('fs');

// Project root — resolved once, reused everywhere (see Topic 06 for __dirname patterns)
const PROJECT_ROOT = path.resolve(__dirname, '..', '..');

// Where uploaded avatars physically live on disk
const AVATAR_DIR = path.join(PROJECT_ROOT, 'uploads', 'avatars');

// Make sure the upload directory exists before we ever try to write into it
if (!fs.existsSync(AVATAR_DIR)) {
  fs.mkdirSync(AVATAR_DIR, { recursive: true });   // create nested folders in one call
}

// Saves an uploaded avatar buffer for a given user and returns metadata about it.
function saveAvatar(userId, originalFileName, fileBuffer) {
  // Only trust the EXTENSION from the original name, never the whole name (Mistake 3 below)
  const extension = path.extname(originalFileName).toLowerCase();   // e.g. ".png"

  // Build a safe, predictable file name — never use user input directly as a file name
  const safeFileName = `${userId}${extension}`;                     // e.g. "user_42.png"

  // Full path on disk where this avatar will be written
  const diskPath = path.join(AVATAR_DIR, safeFileName);

  fs.writeFileSync(diskPath, fileBuffer);   // write the actual image bytes to disk

  // The public-facing URL path — always forward slashes, this is a URL, not an OS path
  const publicUrl = `/static/avatars/${safeFileName}`;

  return {
    userId,
    fileName: safeFileName,
    extension,                       // useful for validating allowed types (.png, .jpg)
    diskPath,                        // where it actually lives — for internal use only
    publicUrl,                       // what gets sent back to the frontend
  };
}

// Removes a user's avatar file from disk, if it exists
function deleteAvatar(userId, extension) {
  const diskPath = path.join(AVATAR_DIR, `${userId}${extension}`);
  if (fs.existsSync(diskPath)) {
    fs.unlinkSync(diskPath);         // delete the file
    return true;
  }
  return false;                      // nothing to delete
}

module.exports = { saveAvatar, deleteAvatar, AVATAR_DIR };

// Usage in a route handler:
// const { saveAvatar } = require('./avatarService');
// const result = saveAvatar(req.user.id, req.file.originalname, req.file.buffer);
// res.json({ avatarUrl: result.publicUrl });
```

---

## Common mistakes

### Mistake 1 — String-concatenating paths instead of using path.join()

```js
// ❌ WRONG — manual concatenation breaks on Windows and with trailing/missing slashes
const uploadsDir = '/app/uploads';
const avatarPath = uploadsDir + '/' + 'avatars' + '/' + userId + '.png';
// If uploadsDir already ends in "/", you get "//avatars" — a subtle, hard-to-spot bug
// On Windows this also produces mixed forward/back slashes when other parts use path.sep

// ✅ CORRECT — path.join() normalizes separators and collapses duplicate slashes for you
const path = require('path');
const avatarPath2 = path.join(uploadsDir, 'avatars', `${userId}.png`);
// Works identically and correctly on Linux, Mac, and Windows
```

### Mistake 2 — Using join() when you need an absolute path (or vice versa)

```js
// ❌ WRONG — join() does NOT guarantee an absolute path, it just glues segments together
const path = require('path');
const configPath = path.join('config', 'database.json');
// If process.cwd() is not the project root, this relative path resolves to the WRONG place
fs.readFileSync(configPath, 'utf8');   // may throw ENOENT depending on where node was run

// ✅ CORRECT — use resolve() (ideally anchored to __dirname) when you need a guaranteed absolute path
const configPath2 = path.resolve(__dirname, 'config', 'database.json');
fs.readFileSync(configPath2, 'utf8');  // always reads the correct file, regardless of cwd
```

### Mistake 3 — Trusting user-supplied file names without sanitizing them

```js
// ❌ WRONG — using a raw file name from user input lets an attacker escape the target folder
const path = require('path');
function getUploadPath(userSuppliedFileName) {
  // If userSuppliedFileName is "../../etc/passwd", this escapes the uploads folder entirely!
  return path.join(UPLOAD_DIR, userSuppliedFileName);   // path traversal vulnerability
}

// ✅ CORRECT — strip directory info from user input and only keep a safe extension/basename
function getUploadPathSafe(userSuppliedFileName) {
  const safeBaseName = path.basename(userSuppliedFileName);  // discards any "../" segments
  const extension    = path.extname(safeBaseName);           // only trust the extension part
  const finalPath    = path.join(UPLOAD_DIR, `${Date.now()}${extension}`); // generate our own name
  return finalPath;   // guaranteed to stay inside UPLOAD_DIR
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that takes a hardcoded file path string like `'/var/www/app/public/images/logo.png'` and logs:
1. The directory it lives in
2. The file name (with extension)
3. The file name (without extension)
4. Just the extension
5. Whether `path.isAbsolute()` says this path is absolute

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `buildLogFilePath(baseDir, serviceName, date)` that:
1. Accepts a base directory, a service name string (e.g. `'auth-service'`), and a `Date` object
2. Builds a file name in the format `{serviceName}-{YYYY-MM-DD}.log` from the date
3. Uses `path.join()` to combine `baseDir`, a `'logs'` subfolder, and the generated file name
4. Returns the final path

Then call it three times with different service names and dates, and log each result. Also write a second function `getLogFileName(fullPath)` that takes one of the paths you generated and returns just the base name without the `.log` extension, using `path.basename()` with its second argument.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `SafeUploadResolver` class that protects against path traversal attacks:
1. Constructor takes a `rootDir` (absolute path) that all uploads must stay inside
2. Has a method `resolvePath(userSuppliedRelativePath)` that:
   - Joins `rootDir` with the user-supplied path using `path.join()`
   - Uses `path.resolve()` on both the root and the joined result
   - Compares the resolved result against the resolved root — if the result does NOT start with the root directory (i.e. someone tried to escape with `../../`), throw an `Error('Path traversal attempt blocked')`
   - Otherwise returns the safe, resolved absolute path
3. Has a method `getSafeFileName(originalFileName)` that returns only the basename plus a random-looking suffix (you can use `Date.now()`) combined with the original extension, discarding any directory components entirely

Test it with:
```js
const resolver = new SafeUploadResolver('/app/uploads');
console.log(resolver.resolvePath('avatars/user_1.png'));      // should succeed
console.log(resolver.getSafeFileName('../../etc/passwd.txt')); // should discard the traversal, keep only "passwd" + extension
try {
  resolver.resolvePath('../../../etc/passwd');  // should throw
} catch (err) {
  console.log('Blocked as expected:', err.message);
}
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE METHODS
  path.join(...segments)     → glue path pieces, normalize separators, collapse ".." — NOT always absolute
  path.resolve(...segments)  → same as join, but ALWAYS returns an absolute path (anchors to cwd if needed)
  path.dirname(p)            → folder portion of a path                 "/a/b/c.js" → "/a/b"
  path.basename(p)           → last segment (file/folder name)          "/a/b/c.js" → "c.js"
  path.basename(p, ext)      → last segment, with given extension removed → "c.js" → "c"
  path.extname(p)            → extension including the dot              "c.js" → ".js"
  path.parse(p)              → { root, dir, base, ext, name } all at once
  path.format(obj)           → opposite of parse() — build a string from the parts
  path.isAbsolute(p)         → true/false, is this already a full path
  path.sep                   → OS separator character: "/" (POSIX) or "\\" (Windows)
  path.normalize(p)          → cleans up "a//b/../c" style messiness without joining anything

JOIN vs RESOLVE
  join('a', 'b')             → "a/b"                       (relative stays relative)
  resolve('a', 'b')          → "/cwd/a/b"                   (always absolute)
  Rule of thumb: anchor resolve() to __dirname for reliability, not process.cwd()

NEVER DO
  dir + '/' + file           → breaks on Windows, double-slash bugs
  Trusting user file names directly → path traversal risk, always path.basename() first
  Assuming join() is absolute → it is NOT, use resolve() when you need guarantees

COMMON PATTERNS
  Anchor to current file:     path.resolve(__dirname, 'config.json')
  Project root from src/x/y:  path.resolve(__dirname, '..', '..')
  Strip extension:             path.basename(file, path.extname(file))
  Guard against traversal:      path.basename(userInput)  // discard any "../" segments
```

---

## Connected topics

- **06 — __dirname and __filename** — the values you almost always pass as the first argument to `path.join()`/`path.resolve()` to anchor a path to the current file
- **15 — fs module — callbacks** — where paths built with the `path` module are actually used to read, write, and check files on disk
- **44 — File uploads** — path traversal prevention (Mistake 3 and Exercise 3 above) is the exact defense used when handling real user-uploaded files
