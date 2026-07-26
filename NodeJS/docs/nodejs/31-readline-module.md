# 31 — readline module

## What is this?

`readline` is a built-in Node.js module that lets you read input **line by line** — one line at a time, as the user types it or as a file is streamed — instead of dealing with raw, chopped-up chunks of bytes. It works on top of any readable stream, most commonly `process.stdin` (keyboard input) or a file stream. Think of it like a cashier at a checkout counter who waits for you to finish saying one full sentence before responding, rather than reacting to every letter you speak — `readline` waits for a complete line (up to `\n`) before handing it to your code.

---

## Why does it matter for backend development?

Backend developers rarely build interactive terminal prompts for a REST API, but `readline` shows up constantly in adjacent, everyday tasks: writing CLI tools and setup scripts (`npx create-my-app`), building interactive database seed/migration scripts that ask "Are you sure? (y/n)", reading massive log files or CSV exports one line at a time without loading the whole file into memory, and piping data between processes (`cat access.log | node parse.js`). Any time you need to process a huge file efficiently or build a script that talks to a human at the terminal, `readline` is the tool — and it teaches the stream-based, event-driven reading pattern that shows up again later in the `stream` module (Topic 26) and CLI tools (Topic 113).

---

## Syntax / API

```js
// readline is a core module — no npm install needed
const readline = require('readline');

// ── Creating an Interface ────────────────────────────────────────────────────
// An Interface is the object that reads lines and lets you write prompts
const rl = readline.createInterface({
  input: process.stdin,    // where to read lines FROM (required)
  output: process.stdout,  // where to print prompts TO (optional, needed for .question())
  terminal: true,          // enables prompt-style features like history and cursor (default: auto-detected)
});

// ── Listening for each line as it arrives ───────────────────────────────────
rl.on('line', (line) => {
  console.log(`You typed: ${line}`);   // fires once per Enter keypress or per \n in a file
});

// ── Detecting when input ends (stdin closed, file fully read, or Ctrl+D) ───
rl.on('close', () => {
  console.log('No more input — done reading.');
  process.exit(0);
});

// ── Asking a single question and getting one answer back (promise-based) ───
rl.question('What is your name? ', (answer) => {
  console.log(`Hello, ${answer}!`);
  rl.close();   // you MUST close the interface manually after question() or it hangs
});

// ── Modern promise-based API (Node 17+) — cleaner for async/await code ─────
const readlinePromises = require('readline/promises');
const rlp = readlinePromises.createInterface({ input: process.stdin, output: process.stdout });
// const answer = await rlp.question('Continue? (y/n) ');   // no callback needed
```

---

## How it works — line by line

`readline.createInterface()` wraps a readable stream (like `process.stdin`) and watches it for newline characters. Every time it sees a `\n`, it treats everything since the last newline as one complete "line" and hands it to any function you attached with `rl.on('line', ...)`. This means your code never has to worry about partial input, buffering, or splitting text yourself — `readline` does that bookkeeping for you.

`rl.question(promptText, callback)` is a convenience method built on top of the same mechanism: it prints `promptText`, waits for exactly one line of input, then calls your callback with that line and immediately stops listening (so you can ask a second question by calling `.question()` again inside the callback, chaining them one after another).

The `'close'` event fires when the input stream ends — this happens when the user presses `Ctrl+D` (end of input) on a terminal, when a piped file finishes streaming, or when you explicitly call `rl.close()`. This is your signal that no more lines are coming, and it is the correct place to run cleanup code, print a summary, or call `process.exit()`.

Because `readline` works on any readable stream — not just `process.stdin` — you can point `input` at `fs.createReadStream('access.log')` and get the exact same line-by-line events, which is how large log files get processed without loading the entire file into memory at once.

---

## Example 1 — basic

```js
// File: src/scripts/greet.js
// A minimal interactive CLI prompt using readline

const readline = require('readline');

// Create the interface, reading from stdin and writing prompts to stdout
const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

// Ask the user's name — callback receives the typed line
rl.question('Enter your name: ', (userName) => {
  // Ask a follow-up question inside the first callback (chaining questions)
  rl.question('Enter your role (admin/user): ', (userRole) => {
    // Print a summary using both answers
    console.log(`Welcome, ${userName}! You are logged in as: ${userRole}`);

    // Always close the interface when you are done asking questions
    rl.close();
  });
});

// Runs after the interface is closed — good place for final cleanup
rl.on('close', () => {
  console.log('Session ended. Goodbye!');
  process.exit(0);   // exit cleanly (Topic 05 — process object)
});
```

---

## Example 2 — real world backend use case

```js
// File: src/scripts/seed-admin.js
// A real CLI script backend devs write: interactively seed an admin user into
// the database, asking for confirmation before writing anything destructive.

const readline = require('readline/promises');   // promise-based API (Node 17+)
const crypto   = require('crypto');               // Topic 23 — crypto module

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

// Fake "database" call — in a real app this would be a Postgres/Mongo insert
async function createAdminUser({ email, apiKey }) {
  const userId = crypto.randomUUID();               // generate a unique id
  console.log(`[db] Inserting admin user ${userId} (${email})...`);
  return { userId, email, apiKey };
}

async function main() {
  // Ask for the admin's email — await pauses until Enter is pressed
  const email = await rl.question('Admin email: ');

  // Basic validation before proceeding — never trust CLI input either
  if (!email.includes('@')) {
    console.error('Invalid email format. Aborting.');
    rl.close();
    return;
  }

  // Confirm a destructive/important action before running it
  const confirmation = await rl.question(
    `Create admin account for "${email}"? Type "yes" to continue: `
  );

  if (confirmation.trim().toLowerCase() !== 'yes') {
    console.log('Aborted — no changes made.');
    rl.close();
    return;
  }

  // Generate a one-time API key for the new admin (Topic 23 — crypto module)
  const apiKey = crypto.randomBytes(24).toString('hex');

  // Perform the "database" write
  const adminUser = await createAdminUser({ email, apiKey });

  console.log('Admin created successfully:');
  console.log(`  userId : ${adminUser.userId}`);
  console.log(`  email  : ${adminUser.email}`);
  console.log(`  apiKey : ${adminUser.apiKey}  (save this — shown only once)`);

  rl.close();   // release stdin so the process can exit naturally
}

main();

// Run with: node src/scripts/seed-admin.js
// This pattern (ask → validate → confirm → act) is the standard shape
// of every interactive setup/seed/migration script in a real backend project.
```

---

## Common mistakes

### Mistake 1 — Forgetting to close the Interface

```js
// ❌ WRONG — the process hangs forever because stdin stays open,
// waiting for more lines that will never come after question() resolves
const readline = require('readline');
const rl = readline.createInterface({ input: process.stdin, output: process.stdout });

rl.question('Enter port number: ', (port) => {
  console.log(`Starting server on port ${port}`);
  // missing rl.close() — script never exits, terminal appears "frozen"
});

// ✅ CORRECT — always close the interface once you're done reading input
rl.question('Enter port number: ', (port) => {
  console.log(`Starting server on port ${port}`);
  rl.close();   // releases stdin, allows 'close' event and process to exit
});
```

### Mistake 2 — Reading an entire large file into memory instead of streaming it line by line

```js
// ❌ WRONG — loads the whole file into RAM before processing a single line
// Fails or is dangerously slow on multi-GB log files
const fs = require('fs');
const content = fs.readFileSync('access.log', 'utf8');   // entire file in memory!
const lines = content.split('\n');
lines.forEach((line) => processLogLine(line));

// ✅ CORRECT — readline streams the file, one line in memory at a time
const readline = require('readline');
const fs = require('fs');

const rl = readline.createInterface({
  input: fs.createReadStream('access.log'),   // Topic 17 — fs streams
  crlfDelay: Infinity,   // treat \r\n as a single line break (Windows-created files)
});

rl.on('line', (line) => {
  processLogLine(line);   // handles one line at a time — constant memory usage
});
```

### Mistake 3 — Assuming rl.question() blocks synchronously (in the callback style)

```js
// ❌ WRONG — code after .question() runs immediately, BEFORE the user answers,
// because question() is asynchronous (it just registers a callback)
const readline = require('readline');
const rl = readline.createInterface({ input: process.stdin, output: process.stdout });

rl.question('Confirm delete (yes/no): ', (answer) => {
  userAnswer = answer;   // this only runs later, after Enter is pressed
});
console.log(`Proceeding with: ${userAnswer}`);   // undefined — runs BEFORE the callback!

// ✅ CORRECT — either put your logic inside the callback...
rl.question('Confirm delete (yes/no): ', (answer) => {
  console.log(`Proceeding with: ${answer}`);   // correct value, guaranteed to run after input
  rl.close();
});

// ✅ ...or use the promise-based API with async/await for linear, readable code
const readlinePromises = require('readline/promises');
async function confirmDelete() {
  const rlp = readlinePromises.createInterface({ input: process.stdin, output: process.stdout });
  const answer = await rlp.question('Confirm delete (yes/no): ');   // pauses here until answered
  console.log(`Proceeding with: ${answer}`);
  rlp.close();
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Creates a `readline` interface on `process.stdin`/`process.stdout`
2. Asks the user to type a number
3. Prints whether that number is even or odd
4. Closes the interface and exits cleanly after answering

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a script `login-prompt.js` that:
1. Asks for a `username` and then an `apiKey` (two separate `rl.question()` calls, chained)
2. Validates that neither field is empty (trim whitespace before checking)
3. If either field is empty, print an error message and exit without proceeding
4. If both are valid, print a fake "login successful" message that includes a generated `sessionId` (use `crypto.randomUUID()`)
5. Make sure the process exits cleanly in every code path (valid and invalid)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `LogAnalyzer` CLI script that:
1. Reads a text file path from `process.argv` (fallback: reject and exit with an error message if no path is given)
2. Uses `readline` with `fs.createReadStream()` to read the file line by line (do NOT use `fs.readFileSync`)
3. Counts how many lines contain the word `"ERROR"` and how many contain `"WARN"` (case-insensitive)
4. Keeps a running total of lines processed
5. On the `'close'` event, prints a summary: total lines, error count, warning count, and the percentage of lines that were errors
6. Handles the case where the file does not exist (listen for an `'error'` event on the read stream and print a friendly message instead of crashing)

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE MODULE — no install needed
  const readline = require('readline');            // callback-style API
  const readline = require('readline/promises');    // promise-based API (Node 17+)

CREATE AN INTERFACE
  readline.createInterface({
    input: process.stdin,       // required — any readable stream
    output: process.stdout,     // optional — needed for prompts/question()
    crlfDelay: Infinity,        // recommended when reading files (handles \r\n)
  })

KEY EVENTS
  rl.on('line', (line) => {})   // fires once per complete line of input
  rl.on('close', () => {})      // fires when input ends (Ctrl+D, EOF, or rl.close())

KEY METHODS
  rl.question(promptText, cb)   // callback style — asks one question, one answer
  await rlp.question(promptText) // promise style — cleaner with async/await
  rl.close()                    // MUST call after question() or the process hangs
  rl.write(text)                // writes text into the input stream (rare, testing use)

READING SOURCES
  process.stdin                       → interactive keyboard input / piped stdin
  fs.createReadStream(filePath)        → read a file line by line, low memory use

WHEN TO USE WHICH API
  Interactive multi-question CLI, sequential logic → readline/promises + async/await
  Simple one-off callback                          → readline (classic callback style)
  Huge files / logs                                 → readline + fs.createReadStream

GOTCHAS
  Forgetting rl.close()            → process never exits, appears frozen
  fs.readFileSync on huge files     → out of memory — use createReadStream instead
  Code after .question() runs early → question() is async, put logic in the callback
  Windows line endings (\r\n)        → set crlfDelay: Infinity to avoid stray \r characters
```

---

## Connected topics

- **05 — The process object** — `process.stdin`, `process.stdout`, and `process.argv` are the raw pieces `readline` builds its Interface on top of
- **17 — fs module — streams and large files** — pairing `readline` with `fs.createReadStream()` is the standard way to process huge log/CSV files one line at a time
- **113 — Building a CLI tool** — `readline` is the foundation for interactive prompts before you graduate to libraries like `commander.js` and `inquirer`
