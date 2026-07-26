# 42 — CSV and file parsing

## What is this?

CSV (Comma-Separated Values) is a plain-text format where each line is a row and each value in that row is separated by a comma. Parsing CSV means turning those raw text lines into structured JavaScript objects or arrays you can actually work with in code. Think of a CSV file like a spreadsheet exported as plain text — parsing is the process of reading that spreadsheet row by row and turning each row into a usable record, instead of one giant blob of text.

## Why does it matter for backend development?

Backend developers constantly deal with CSV: importing a bulk list of users from HR, ingesting a product catalog from a supplier, exporting a report for finance, or processing a bank statement upload. Doing this with naive `string.split(',')` breaks the moment a value contains a comma, a quote, or a newline inside quotes — which real-world CSV files always eventually have. Using a stream-based parser like `csv-parse` also means you can process a 2 GB CSV file with constant, low memory usage instead of loading the whole file into RAM. This is the difference between an import feature that works in a demo and one that survives production data.

---

## Syntax / API

```js
// npm install csv-parse
// csv-parse is the standard, battle-tested CSV parsing library for Node.js
const fs = require('fs');
const { parse } = require('csv-parse'); // named export: the streaming parser factory

// ── Basic streaming parse ────────────────────────────────────────────────────
const filePath = './data/users-import.csv'; // path to the CSV file on disk

fs.createReadStream(filePath)          // 1. open the file as a readable stream (chunks, not all at once)
  .pipe(parse({                        // 2. pipe raw bytes into the CSV parser
    columns: true,                     //    treat the first row as headers, emit objects instead of arrays
    trim: true,                        //    strip whitespace around each value
    skip_empty_lines: true,            //    ignore blank lines in the file
  }))
  .on('data', (row) => {               // 3. fires once per parsed row (an object like { name: 'Amit', age: '29' })
    console.log(row);                  //    handle/transform the row here
  })
  .on('end', () => {                   // 4. fires once after the last row has been parsed
    console.log('CSV import finished');
  })
  .on('error', (err) => {              // 5. fires if the file is malformed or unreadable
    console.error('CSV parse error:', err.message);
  });

// ── papaparse (alternative library, popular in browser + Node) ──────────────
// npm install papaparse
const Papa = require('papaparse');
Papa.parse(fs.createReadStream(filePath), { // Papaparse can also take a readable stream
  header: true,                             // first row = object keys (like columns: true above)
  step: (result) => {                       // called once per parsed row
    console.log(result.data);               // the row as an object
  },
  complete: () => console.log('done'),      // called once parsing finishes
});
```

---

## How it works — line by line

- `fs.createReadStream(filePath)` opens the file but does **not** read it all into memory — it reads it in small chunks as the OS delivers them.
- `.pipe(parse({...}))` sends each chunk of raw bytes into the CSV parser as soon as it arrives. The parser buffers only partial rows internally, not the whole file.
- `columns: true` tells the parser to use the first line of the file as field names, so every future row becomes `{ fieldName: value }` instead of a plain array like `['Amit', '29']`.
- `trim: true` removes accidental leading/trailing spaces that creep in from spreadsheet exports (e.g. `" Amit"` becomes `"Amit"`).
- `skip_empty_lines: true` ignores stray blank lines at the end of the file, which almost every CSV export has.
- The `'data'` event fires once **per row**, as soon as that row is fully parsed — not after the whole file is done. This is what makes the approach memory-efficient for huge files.
- The `'end'` event fires once, after every row has been emitted and the stream has fully drained.
- The `'error'` event fires if something goes wrong — a malformed row, a missing file, or a permissions issue — and must always be handled or the process can crash.

---

## Example 1 — basic

```js
// File: scripts/read-csv-basic.js
// Reads a small CSV file and prints every row as a JavaScript object.

const fs = require('fs');                 // core module — file access
const { parse } = require('csv-parse');    // csv-parse's streaming parser

const filePath = './data/products.csv';    // sample file: name,price,inStock

const rows = [];                           // collect parsed rows here

fs.createReadStream(filePath)              // open the CSV file as a stream
  .pipe(parse({ columns: true, trim: true })) // parse with header row as keys
  .on('data', (row) => {
    rows.push(row);                        // row → { name: 'Keyboard', price: '1200', inStock: 'true' }
  })
  .on('end', () => {
    console.log(`Parsed ${rows.length} rows`); // total row count
    console.log(rows[0]);                       // inspect the first parsed row
  })
  .on('error', (err) => {
    console.error('Failed to parse CSV:', err.message); // always handle errors
  });
```

---

## Example 2 — real world backend use case

```js
// File: src/services/importUsersFromCsv.js
// A realistic bulk-import job: read a CSV of new users, validate + transform each
// row, and insert valid rows into the database while collecting rejected rows.

const fs = require('fs');
const { parse } = require('csv-parse');

/**
 * Streams a CSV of users, transforms each row, and calls dbConnection to insert it.
 * Returns a summary of how many rows succeeded vs failed.
 */
function importUsersFromCsv(filePath, dbConnection) {
  return new Promise((resolve, reject) => {
    const insertedUsers = [];              // successfully transformed + inserted rows
    const rejectedRows = [];               // rows that failed validation, with reasons

    const parser = parse({                 // configure the parser once, reuse the instance
      columns: true,                       // first row = header names (name, email, role)
      trim: true,                          // clean stray whitespace from spreadsheet exports
      skip_empty_lines: true,              // ignore blank trailing lines
    });

    fs.createReadStream(filePath)          // stream the uploaded CSV file from disk
      .pipe(parser)                        // feed raw bytes into the parser
      .on('data', async (row) => {
        parser.pause();                    // pause the stream so async DB work doesn't overlap rows

        const email = row.email?.toLowerCase().trim(); // normalize email casing
        const isValidEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email || '');

        if (!email || !isValidEmail) {
          rejectedRows.push({ row, reason: 'invalid or missing email' }); // track bad rows
          parser.resume();                 // resume reading the next row
          return;
        }

        try {
          const userId = await dbConnection.query(   // insert the transformed row
            'INSERT INTO users (name, email, role) VALUES ($1, $2, $3) RETURNING id',
            [row.name, email, row.role || 'member']  // default role if missing
          );
          insertedUsers.push({ userId, email });      // track what was inserted
        } catch (dbErr) {
          rejectedRows.push({ row, reason: dbErr.message }); // record insert failure
        }

        parser.resume();                   // always resume, success or failure
      })
      .on('end', () => {
        resolve({                          // final summary once the whole file is processed
          insertedCount: insertedUsers.length,
          rejectedCount: rejectedRows.length,
          rejectedRows,
        });
      })
      .on('error', (err) => {
        reject(err);                       // surface file-read or parse errors to the caller
      });
  });
}

module.exports = { importUsersFromCsv };

// Usage in an Express route (Topic 53):
// const { insertedCount, rejectedCount } = await importUsersFromCsv(req.file.path, dbConnection);
// res.json({ insertedCount, rejectedCount });
```

---

## Common mistakes

### Mistake 1 — Splitting CSV manually with `.split(',')`

```js
// ❌ WRONG — breaks the instant a value contains a comma or a quoted newline
const line = '"Sharma, Amit",29,"Bengaluru"';
const fields = line.split(','); // → ['"Sharma', ' Amit"', '29', '"Bengaluru"'] — corrupted!

// ✅ CORRECT — let a real CSV parser handle quoting, escaping, and embedded commas
const { parse } = require('csv-parse/sync'); // sync helper for small strings/files
const records = parse(line, { columns: false });
console.log(records[0]); // → ['Sharma, Amit', '29', 'Bengaluru'] — correct
```

### Mistake 2 — Loading the whole file into memory before parsing

```js
// ❌ WRONG — reads a potentially multi-GB file entirely into RAM first
const fs = require('fs');
const { parse } = require('csv-parse/sync');

const fileContent = fs.readFileSync('./data/huge-export.csv', 'utf8'); // loads everything!
const records = parse(fileContent, { columns: true }); // then parses the giant string
// Process crashes with "JavaScript heap out of memory" on large files

// ✅ CORRECT — stream the file so memory stays constant no matter the file size
const { parse } = require('csv-parse');
fs.createReadStream('./data/huge-export.csv')
  .pipe(parse({ columns: true }))
  .on('data', (row) => { /* process one row at a time */ })
  .on('end', () => console.log('done'));
```

### Mistake 3 — Not handling malformed rows, letting one bad row crash the import

```js
// ❌ WRONG — a single malformed row throws and takes down the whole import job
fs.createReadStream(filePath)
  .pipe(parse({ columns: true }))
  .on('data', (row) => {
    const age = parseInt(row.age, 10);
    if (age < 0) throw new Error('invalid age'); // unhandled throw inside a 'data' handler
  });
// Uncaught exception — entire process can crash depending on how it's wired up

// ✅ CORRECT — validate defensively per row and collect failures instead of throwing
fs.createReadStream(filePath)
  .pipe(parse({ columns: true }))
  .on('data', (row) => {
    const age = parseInt(row.age, 10);
    if (Number.isNaN(age) || age < 0) {
      console.warn('Skipping invalid row:', row); // log and skip, don't crash the stream
      return;
    }
    // process the valid row normally
  })
  .on('error', (err) => console.error('Stream error:', err.message)); // handle stream-level errors too
```

---

## Practice exercises

### Exercise 1 — easy

Create a CSV file `data/employees.csv` with columns `name,department,salary` and at least 5 rows of sample data. Write a script that:
1. Streams the file using `csv-parse`
2. Prints each row as it is parsed
3. Prints the total number of rows once parsing finishes
4. Handles and logs any parse errors

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `parseCsvToJson(filePath, outputPath)` that:
1. Streams and parses a CSV file with a header row
2. Transforms each row: convert any field named `salary` or `age` from a string to a number
3. Collects all transformed rows into an array
4. Once parsing is complete, writes that array as a pretty-printed JSON file to `outputPath` using `fs.writeFile`
5. Returns a promise that resolves with the total number of rows written

Test it against the `employees.csv` file from Exercise 1 and verify the output JSON has numeric `salary` values, not strings.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `CsvImporter` class for a backend service that:
1. Takes `filePath` and a `validateRow(row)` function in its constructor — `validateRow` returns `{ valid: true }` or `{ valid: false, reason: '...' }`
2. Has a `run()` method (returns a Promise) that streams the CSV using `csv-parse`
3. For each row, calls `validateRow(row)`:
   - If valid, pushes it to an internal `accepted` array
   - If invalid, pushes `{ row, reason }` to an internal `rejected` array
4. Emits progress: every 100 rows processed, log `Processed ${count} rows...`
5. On `'end'`, resolves with `{ acceptedCount, rejectedCount, rejected }`
6. On `'error'`, rejects the promise with the underlying error
7. Handles the case where the file does not exist by rejecting with a clear error message before attempting to parse

Test it with a CSV where some rows have negative numbers in a `quantity` column, using a `validateRow` that rejects any row with `quantity < 0`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
LIBRARIES
  csv-parse   → most popular, streaming-first, Node-native, sync + async APIs
  papaparse   → works in browser AND Node, good for smaller files, step/complete callbacks
  fast-csv    → alternative streaming parser, similar API shape to csv-parse

BASIC STREAMING PATTERN
  fs.createReadStream(filePath)
    .pipe(parse({ columns: true, trim: true, skip_empty_lines: true }))
    .on('data', row => { ... })   // fires per row
    .on('end', () => { ... })     // fires once, after last row
    .on('error', err => { ... })  // ALWAYS attach this

KEY PARSE OPTIONS (csv-parse)
  columns: true          → first row becomes object keys (row = object, not array)
  trim: true             → strips stray whitespace around values
  skip_empty_lines: true → ignores blank lines
  delimiter: ';'         → for non-comma-separated files (e.g. semicolon CSV)
  cast: true             → auto-converts numeric-looking strings to numbers

SYNC vs STREAM
  require('csv-parse/sync')  → parse(stringOrBuffer, opts) — for small files only
  require('csv-parse')       → streaming — use for anything large or uploaded

WHY STREAMING MATTERS
  readFileSync + parse whole string → loads entire file into RAM → OOM on large files
  createReadStream + pipe(parse())  → constant memory, processes row by row

NEVER DO
  line.split(',')                  → breaks on quoted commas/newlines
  readFileSync on untrusted uploads → memory risk, no backpressure
  throw inside a 'data' handler     → can crash the whole stream/process

ALWAYS DO
  Attach an 'error' listener on every stream
  Validate/transform each row defensively — collect rejects, don't crash
  Use parser.pause() / parser.resume() around async work per row (DB inserts, API calls)
```

---

## Connected topics

- **17 — fs module — streams and large files** — the `createReadStream` foundation this topic pipes CSV data through
- **26 — stream module in depth** — backpressure, `pipe()`, and `Transform` streams explained in full, which is what `csv-parse` is built on
- **44 — File uploads** — CSV imports in a real backend usually arrive as an uploaded file first, before being streamed and parsed
