# 122 — Design a Logging Framework
## Category: LLD Case Study

---

## What is this?

A **logging framework** is the plumbing that lets any line of code in your application say "record that this happened" — and then reliably decides whether that message is important enough to keep, formats it, and writes it to one or more destinations (your terminal, a file, a remote log server).

Think of it like the **flight recorder ("black box") on an airplane**. Pilots and systems constantly emit events; the recorder decides what to store, timestamps everything, and writes it to durable storage so that when something goes wrong, investigators have a trail. Your logging framework is your program's black box.

This is a beloved LLD interview problem because it is *pattern-rich*: it naturally combines **Chain of Responsibility** (level filtering), **Strategy** (pluggable formatters), **Singleton** (a globally reachable logger), and an **Observer-like fanout** (one message, many outputs). It's a great vehicle to show you can pick the right pattern — and be honest about when a pattern is overkill.

---

## Why does it matter?

**If you don't understand this**, you end up with `console.log("here")` and `console.log("value is", x)` scattered everywhere — impossible to turn off in production, impossible to redirect to a file, impossible to search, no timestamps, no severity. When a 3 a.m. incident hits, you're blind.

**Interview angle:** Logging is the canonical "design a framework using patterns" LLD question. Interviewers use it to see whether you can (a) decompose a fuzzy requirement into objects, (b) apply GoF patterns *appropriately*, and (c) resist over-engineering. Saying "I'll use Chain of Responsibility for levels" earns points; adding "...though a single `if (level >= threshold)` is usually clearer, so here's the trade-off" earns *more*.

**Real-work angle:** Every serious backend depends on a logging framework — Winston, Pino, Bunyan (Node), Log4j/Logback (Java), `logging` (Python). Understanding levels, appenders, formatters, and async flushing is the difference between logs that help you and logs that either lie, cost a fortune, or slow down your request path.

---

## The core idea — explained simply

### Analogy: The corporate mailroom

Imagine a large office. Every employee (every line of your code) can drop a memo into the mailroom. The mailroom is your logging framework.

1. **Severity stamp.** Each memo is marked ROUTINE, IMPORTANT, or URGENT. The mailroom has a policy: "today we only process IMPORTANT and above." ROUTINE memos are shredded immediately. (This is **level filtering**.)
2. **Copies to many mailboxes.** A surviving memo isn't sent to one place — the mailroom photocopies it and drops copies into several outboxes: the wall bulletin board (console), the filing cabinet (file), and the courier to head office (remote server). (This is the **appender fanout**.)
3. **Different letterhead per outbox.** The bulletin-board copy is printed in a friendly human format; the head-office courier copy is printed as a strict structured form the head office's computers can parse. (This is the **formatter strategy**.)
4. **One mailroom for the whole building.** Nobody builds their own mailroom; there's one well-known mailroom everyone uses. (This is the **singleton** accessor.)

Map it back:

| Mailroom concept | Technical concept |
|---|---|
| Employee dropping a memo | `logger.info("user logged in")` |
| Severity stamp (ROUTINE/URGENT) | `LogLevel` (DEBUG…FATAL) |
| "Only process IMPORTANT+" policy | Minimum level threshold |
| Shredding low-severity memos | Level filtering (Chain of Responsibility *or* an `if`) |
| Photocopying to many outboxes | Appenders (Console/File/Remote) |
| Different letterhead per outbox | Formatters (Plain/JSON) via Strategy |
| The one shared mailroom | `Logger.getInstance()` singleton |
| Memo with timestamp + body + tags | `LogRecord` |

The whole design is: **a message flows through a filter, then fans out to N appenders, each of which formats it its own way and writes it somewhere.**

---

## Key concepts inside this topic

We'll follow the **7-step LLD method**: requirements → nouns → verbs/ownership → patterns → class diagram → code → patterns-justified, then extensions.

### 1. Requirements & clarifying questions

**Functional requirements:**
- Emit log messages at different **levels**: `DEBUG < INFO < WARN < ERROR < FATAL`.
- A configurable **minimum level** — only messages at or above it are recorded.
- Support **multiple output destinations (appenders)** simultaneously: console, file, remote/HTTP.
- **Configurable formatting** — plain human-readable text *or* structured JSON.
- **Easy to use from anywhere** in the app (a single call like `logger.info(...)`, no wiring).
- Attach **context** (structured key/value data) to a message.

**Non-functional requirements:**
- Low overhead: a filtered-out `debug()` call must be cheap.
- Extensible: adding a new appender or formatter shouldn't touch existing code (Open/Closed).
- Robust: one broken appender shouldn't crash the app or block the others.

**Clarifying questions (say these out loud in an interview):**
- **Async logging?** Should writes block the caller, or buffer and flush on a queue? (Production frameworks are async so logging never stalls the request path — see Extensions.)
- **Thread / worker safety?** Node is single-threaded per event loop, but with `worker_threads` or `cluster`, do multiple workers write the same file? (File locking / one-writer-per-file matters.)
- **Structured / JSON logs?** Are logs consumed by humans (plain text) or machines like Elasticsearch/Datadog (JSON)? Usually *both*, per destination.
- **Log rotation?** Should the file appender roll over by size/date so files don't grow unbounded?
- **Per-module levels?** Do we want `payments` at DEBUG while everything else is at INFO?
- **Sampling?** For extremely high-volume debug logs, do we keep only 1-in-N?

For this design we'll build synchronous logging with pluggable appenders/formatters and per-appender levels, and show how async, rotation, per-module, and sampling slot in later.

### 2. Identify core objects (the nouns)

Read the requirements and underline the nouns:

| Object | Responsibility |
|---|---|
| `LogLevel` | An **enum** of severities with a numeric order so we can compare them. |
| `LogRecord` | One log event: level, message text, timestamp, and a context object. |
| `Formatter` | Turns a `LogRecord` into a string. Base class + `PlainFormatter`, `JsonFormatter`. |
| `Appender` (a.k.a. Handler) | A destination that *writes* a formatted record. Base + `ConsoleAppender`, `FileAppender`, `RemoteAppender`. Holds its own formatter and its own minimum level. |
| `Logger` | The front door. Holds a minimum level and a list of appenders; filters then dispatches. |
| `LoggerConfig` | (optional) a plain object describing the level + appenders to build a logger from. |

### 3. Behaviours (verbs) + who owns them

| Behaviour | Owned by | Why |
|---|---|---|
| `log(level, message, context)` | `Logger` | The single entry point; decides IF a message passes and dispatches it. |
| `shouldLog(level)` | `Logger` (and each `Appender`) | Level comparison against a threshold. |
| `format(record) → string` | `Formatter` | Owns HOW a record is rendered. |
| `append(record)` / `write(text)` | `Appender` | Owns WHERE the record goes. |
| convenience `debug/info/warn/error/fatal` | `Logger` | Ergonomics — thin wrappers over `log`. |
| `getInstance()` | `Logger` | Global access (singleton). |

**Ownership summary — the key insight:** the **Logger** decides *if* a message survives the level filter and *dispatches* it; each **Appender** decides *where* (and applies its own optional stricter level); each **Formatter** decides *how* it looks. Clean separation of concerns — three independent axes you can vary without touching the others.

### 4. The pattern design — which pattern fits where

This problem is *really* about matching patterns to sub-problems. Let's take each honestly.

**(a) Level filtering — Chain of Responsibility vs. a simple threshold.**

The **textbook** answer is **Chain of Responsibility (CoR)**: build a chain of handlers — `DebugHandler → InfoHandler → WarnHandler → ErrorHandler → FatalHandler`. A record enters the chain; each handler asks "is this *my* level?" If yes it handles it, otherwise it passes the record to the next handler. (Recall **[47 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md)**: decouple sender from receiver by letting a request travel a chain until someone handles it.)

The **practical, modern** answer is a single integer comparison:

```js
if (record.level.priority >= this.minLevel.priority) { /* dispatch */ }
```

**Be honest in the interview:** CoR here is arguably *over-engineering*. Levels are totally ordered integers; a chain of five handler objects to express `level >= threshold` is more code, more indirection, and slower, for zero flexibility gain. This is exactly the **YAGNI / "pattern-itis"** trap — using a pattern because it's clever, not because the problem needs it. A logging framework's real extensibility lives in *appenders and formatters*, not in the level comparison.

Because the connected topic is 47, we'll **show the CoR version** so you can speak to it — but the production `Logger` uses the simple threshold, and we'll say why. Knowing *when not to* apply a pattern is a senior signal.

**(b) Appender fanout — Observer-like.**

A surviving record is written to **all** configured appenders. One event, many independent observers reacting — that's the **Observer** shape (a.k.a. publish/subscribe / event fanout). We won't build a formal `Subject`/`observers` registry with `subscribe()`; a plain array of appenders the `Logger` iterates is the idiomatic, minimal realization of the same idea.

**(c) Formatters — Strategy.**

*How* a record is rendered (plain text vs. JSON) is a family of interchangeable algorithms. That's textbook **Strategy** (recall **[42 — Strategy](./42-pattern-strategy.md)**: encapsulate each algorithm behind a common interface and inject it). Each `Appender` *has-a* `Formatter` injected via its constructor — swap `PlainFormatter` for `JsonFormatter` without touching the appender. This is a genuinely warranted pattern: formatting really does vary independently and open-endedly (add `LogfmtFormatter`, `SyslogFormatter`, …).

**(d) Global access — Singleton.**

We want any file to `require('./logger')` and log without threading a logger object through every function. That's **Singleton** (recall **[29 — Singleton](./29-pattern-singleton.md)**). In Node the idiomatic singleton is just `module.exports = loggerInstance` — the module cache guarantees one instance. We'll also show a classic `Logger.getInstance()`.

**The honest caveat (say it):** a global singleton logger is *global mutable state*. It hurts **testability** (tests can't easily inject a fake logger or assert on captured output without reaching into a global), and it hides dependencies. The cleaner-for-testing approach is **dependency injection** — pass a `logger` into constructors. In practice most codebases accept a singleton for the top-level app logger *and* allow injecting one in unit-tested classes. We'll present both.

### 5. Class diagram (ASCII)

```
                         ┌──────────────────────────────┐
     logger.info(msg) ──▶│           Logger             │  (Singleton accessor:
                         │  - minLevel : LogLevel       │   Logger.getInstance())
                         │  - appenders: Appender[]     │
                         │  + log(level, msg, ctx)      │
                         │  + debug/info/warn/error/... │
                         └───────────────┬──────────────┘
                                         │ 1. build LogRecord
                                         │ 2. if(level >= minLevel)   ◀── threshold
                                         │    (or run CoR chain)          filter
                                         │ 3. dispatch to ALL appenders (fanout / Observer)
                         ┌───────────────┼───────────────────────┐
                         ▼               ▼                        ▼
             ┌────────────────┐ ┌────────────────┐   ┌──────────────────┐
             │ ConsoleAppender│ │  FileAppender  │   │  RemoteAppender  │   Appender (base)
             │ minLevel: DEBUG│ │ minLevel: INFO │   │ minLevel: WARN   │   - minLevel
             │ formatter ───┐ │ │ formatter ──┐  │   │ formatter ──┐    │   - formatter
             └──────────────┼─┘ └─────────────┼──┘   └─────────────┼────┘   + append(record)
                            ▼                 ▼                    ▼
                   ┌────────────────┐ ┌────────────────┐  ┌────────────────┐
                   │ PlainFormatter │ │ PlainFormatter │  │ JsonFormatter  │   Formatter (base)
                   │  format(rec)   │ │  format(rec)   │  │  format(rec)   │   + format(record)
                   └────────────────┘ └────────────────┘  └────────────────┘   (Strategy)

     LogLevel (enum, Object.freeze):  DEBUG=0 < INFO=1 < WARN=2 < ERROR=3 < FATAL=4
     LogRecord: { level, message, timestamp, context }
```

The Logger owns the *if*; each Appender owns the *where* (plus its own optional stricter level); each Formatter owns the *how*.

### 6. Full JavaScript implementation

Complete and runnable with `node logger.js`. The `FileAppender` writes to an in-memory buffer so the demo runs anywhere without touching the real filesystem (a comment shows the real `fs` line). The `RemoteAppender` mocks an HTTP send.

```js
'use strict';

// ─────────────────────────────────────────────────────────────
// 1. LogLevel — enum via Object.freeze, with a numeric priority
//    so DEBUG < INFO < WARN < ERROR < FATAL is a simple compare.
// ─────────────────────────────────────────────────────────────
const LogLevel = Object.freeze({
  DEBUG: { name: 'DEBUG', priority: 0 },
  INFO:  { name: 'INFO',  priority: 1 },
  WARN:  { name: 'WARN',  priority: 2 },
  ERROR: { name: 'ERROR', priority: 3 },
  FATAL: { name: 'FATAL', priority: 4 },
});

// ─────────────────────────────────────────────────────────────
// 2. LogRecord — one immutable log event.
// ─────────────────────────────────────────────────────────────
class LogRecord {
  constructor(level, message, context = {}) {
    this.level = level;                 // a LogLevel value
    this.message = message;             // string
    this.timestamp = new Date();        // when it happened
    this.context = context;             // structured key/value data
  }
}

// ─────────────────────────────────────────────────────────────
// 3. Formatter — STRATEGY. Base defines the interface;
//    subclasses own HOW a record is rendered.
// ─────────────────────────────────────────────────────────────
class Formatter {
  format(record) {
    throw new Error('Formatter.format() must be implemented by a subclass');
  }
}

class PlainFormatter extends Formatter {
  // e.g. "[2026-07-18T10:00:00.000Z] [WARN] disk almost full {\"pct\":91}"
  format(record) {
    const ts = record.timestamp.toISOString();
    const ctx = Object.keys(record.context).length
      ? ' ' + JSON.stringify(record.context)
      : '';
    return `[${ts}] [${record.level.name}] ${record.message}${ctx}`;
  }
}

class JsonFormatter extends Formatter {
  // Structured logging: machine-parseable single-line JSON.
  // (Recall 80 — Monitoring & Observability: JSON logs are what
  //  Elasticsearch / Datadog / Loki ingest and index.)
  format(record) {
    return JSON.stringify({
      timestamp: record.timestamp.toISOString(),
      level: record.level.name,
      message: record.message,
      ...record.context,             // flatten context into the object
    });
  }
}

// ─────────────────────────────────────────────────────────────
// 4. Appender — base "Handler". Owns WHERE a record is written,
//    holds its own Formatter (Strategy) and its own minLevel so
//    each destination can be stricter than the Logger.
// ─────────────────────────────────────────────────────────────
class Appender {
  constructor(formatter, minLevel = LogLevel.DEBUG) {
    if (new.target === Appender) {
      throw new Error('Appender is abstract; subclass it');
    }
    this.formatter = formatter;
    this.minLevel = minLevel;
  }

  // The Logger calls this. Guard on the appender's own level, then write.
  append(record) {
    if (record.level.priority < this.minLevel.priority) return;
    try {
      this.write(this.formatter.format(record), record);
    } catch (err) {
      // One broken appender must never crash the app or block the others.
      console.error(`[Appender error] ${this.constructor.name}: ${err.message}`);
    }
  }

  write(text, record) {
    throw new Error('Appender.write() must be implemented by a subclass');
  }
}

class ConsoleAppender extends Appender {
  write(text, record) {
    // Route ERROR/FATAL to stderr, everything else to stdout.
    if (record.level.priority >= LogLevel.ERROR.priority) {
      console.error(text);
    } else {
      console.log(text);
    }
  }
}

class FileAppender extends Appender {
  constructor(formatter, minLevel, filename = 'app.log') {
    super(formatter, minLevel);
    this.filename = filename;
    this.buffer = [];   // in-memory to stay runnable anywhere
  }

  write(text, record) {
    // Real version:
    //   const fs = require('fs');
    //   fs.appendFileSync(this.filename, text + '\n');
    this.buffer.push(text);
  }

  // Helper for the demo so we can prove what landed in the "file".
  dump() {
    return this.buffer.slice();
  }
}

class RemoteAppender extends Appender {
  constructor(formatter, minLevel, endpoint = 'https://logs.example.com/ingest') {
    super(formatter, minLevel);
    this.endpoint = endpoint;
    this.sent = [];   // capture for the demo
  }

  write(text, record) {
    // Real version would POST asynchronously:
    //   fetch(this.endpoint, { method: 'POST', body: text });
    // Here we mock the network call so the demo is deterministic.
    this.sent.push(text);
    console.log(`   (→ mock POST to ${this.endpoint})`);
  }
}

// ─────────────────────────────────────────────────────────────
// 5a. OPTIONAL: Chain of Responsibility version of level filtering.
//     Shown because it's the "textbook" answer (connected topic 47).
//     Each handler decides whether a record clears its bar and
//     passes it down the chain. In practice the single-compare
//     threshold in Logger below is clearer — see the write-up.
// ─────────────────────────────────────────────────────────────
class LevelHandler {
  constructor(level) {
    this.level = level;
    this.next = null;
  }
  setNext(handler) { this.next = handler; return handler; } // for chaining
  handle(record) {
    if (record.level.priority >= this.level.priority) return true; // clears the bar
    if (this.next) return this.next.handle(record);
    return false;
  }
}
// Build once: FATAL→ERROR→WARN→INFO→DEBUG. A record "passes" if it
// clears the strictest handler whose threshold it meets. (Demonstrative.)
function buildLevelChain(minLevel) {
  return new LevelHandler(minLevel); // a one-node chain == the threshold; shown for completeness
}

// ─────────────────────────────────────────────────────────────
// 5b. Logger — the front door. Owns the level filter + fanout.
//     Uses the simple threshold (honest choice); a Singleton
//     accessor gives global reach.
// ─────────────────────────────────────────────────────────────
class Logger {
  constructor(minLevel = LogLevel.INFO, appenders = []) {
    this.minLevel = minLevel;
    this.appenders = appenders;
  }

  addAppender(appender) {
    this.appenders.push(appender);
    return this;
  }

  setLevel(level) { this.minLevel = level; return this; }

  // Cheap fast-path: a filtered-out debug() barely costs anything.
  shouldLog(level) {
    return level.priority >= this.minLevel.priority;
  }

  log(level, message, context = {}) {
    if (!this.shouldLog(level)) return;                 // LEVEL FILTER (threshold)
    const record = new LogRecord(level, message, context);
    for (const appender of this.appenders) {            // FANOUT (Observer-like)
      appender.append(record);                          // each may further filter/format
    }
  }

  // Convenience methods — ergonomic sugar over log().
  debug(msg, ctx) { this.log(LogLevel.DEBUG, msg, ctx); }
  info(msg, ctx)  { this.log(LogLevel.INFO,  msg, ctx); }
  warn(msg, ctx)  { this.log(LogLevel.WARN,  msg, ctx); }
  error(msg, ctx) { this.log(LogLevel.ERROR, msg, ctx); }
  fatal(msg, ctx) { this.log(LogLevel.FATAL, msg, ctx); }

  // ── SINGLETON accessor ──
  static getInstance() {
    if (!Logger._instance) {
      Logger._instance = new Logger(LogLevel.INFO, [
        new ConsoleAppender(new PlainFormatter()),
      ]);
    }
    return Logger._instance;
  }
}
Logger._instance = null;

// In real Node code the idiomatic singleton is simply:
//   module.exports = Logger.getInstance();
// so `const log = require('./logger')` gives everyone the same instance.

// ─────────────────────────────────────────────────────────────
// 6. main() — prove it runs.
// ─────────────────────────────────────────────────────────────
function main() {
  console.log('=== Logging framework demo ===\n');

  // Console appender: PLAIN format, logs DEBUG and up.
  const consoleAppender = new ConsoleAppender(new PlainFormatter(), LogLevel.DEBUG);

  // File appender: PLAIN format, logs INFO and up (in-memory buffer).
  const fileAppender = new FileAppender(new PlainFormatter(), LogLevel.INFO);

  // Remote appender: JSON (structured), only WARN and up — we don't ship noise.
  const remoteAppender = new RemoteAppender(new JsonFormatter(), LogLevel.WARN);

  // Logger threshold = DEBUG so we can watch per-appender filtering too.
  const logger = new Logger(LogLevel.DEBUG, [
    consoleAppender, fileAppender, remoteAppender,
  ]);

  // Log at EVERY level with some structured context.
  logger.debug('cache warm started', { keys: 1200 });
  logger.info('user logged in',      { userId: 42 });
  logger.warn('disk almost full',    { pct: 91 });
  logger.error('payment failed',     { userId: 42, orderId: 'A17' });
  logger.fatal('database unreachable', { host: 'db-primary' });

  // ── Prove the filtering worked ──
  console.log('\n--- What the FILE appender kept (INFO+) ---');
  console.log(fileAppender.dump().join('\n'));

  console.log('\n--- What the REMOTE appender shipped (WARN+, JSON) ---');
  console.log(remoteAppender.sent.join('\n'));

  console.log('\n--- Now raise the Logger threshold to WARN ---');
  logger.setLevel(LogLevel.WARN);
  logger.debug('this DEBUG is dropped at the Logger, never reaches any appender');
  logger.info('this INFO is dropped too');
  logger.warn('this WARN gets through', { retry: 3 });

  // ── Singleton reach ──
  console.log('\n--- Singleton: same instance everywhere ---');
  Logger.getInstance().info('logged via Logger.getInstance()');
  console.log('same instance?', Logger.getInstance() === Logger.getInstance());

  // ── Chain of Responsibility filter, demonstrated ──
  console.log('\n--- CoR level check (textbook alt to the threshold) ---');
  const chain = buildLevelChain(LogLevel.WARN);
  const info = new LogRecord(LogLevel.INFO, 'x');
  const err  = new LogRecord(LogLevel.ERROR, 'y');
  console.log('INFO passes WARN chain?', chain.handle(info)); // false
  console.log('ERROR passes WARN chain?', chain.handle(err)); // true
}

main();
```

**What the demo proves when you run it:**
- `debug` and `info` are formatted for the console but the **remote** appender (WARN+) ignores them — its `sent` array only holds `warn/error/fatal` as JSON.
- The **file** appender (INFO+) drops the `debug` line but keeps INFO and up as plain text.
- Raising the Logger threshold to WARN drops DEBUG/INFO *at the source* so no appender even sees them (the cheap fast-path).
- `Logger.getInstance() === Logger.getInstance()` prints `true` — the singleton holds.
- The CoR chain independently confirms level ordering, so you can talk to both approaches.

### 7. Design patterns used and WHY

| Pattern | Where | Genuinely warranted? |
|---|---|---|
| **Chain of Responsibility** ([47](./47-pattern-chain-of-responsibility.md)) | Level filtering (`LevelHandler` chain) | **Textbook, but usually overkill here.** Levels are ordered ints; `if (level >= threshold)` is clearer and faster. Show CoR to demonstrate you know it; use the threshold in production. |
| **Strategy** ([42](./42-pattern-strategy.md)) | `Formatter` injected into each `Appender` | **Yes.** Formatting varies open-endedly (plain/JSON/logfmt/syslog) and independently of destination. Clean swap, Open/Closed. |
| **Singleton** ([29](./29-pattern-singleton.md)) | `Logger.getInstance()` / `module.exports = logger` | **Convenient, with a caveat.** Global reach is the whole point of a logger — but it's global mutable state that hurts testability. Inject the logger in unit-tested classes; keep a singleton for the app-wide default. |
| **Observer-like fanout** | Logger iterates `appenders[]` | **Yes, in its minimal form.** One event → many independent outputs. A plain array is the idiomatic realization; a full `subscribe()` registry would be over-engineering. |
| **Template Method** (bonus) | `Appender.append()` guards + formats, then calls abstract `write()` | Small but real: the base fixes the *skeleton* (level guard → format → write, with try/catch), subclasses fill only `write()`. |

**The senior signal:** name all four, then say *which are warranted*. Strategy (formatters) and the appender fanout carry their weight. CoR for levels is a pattern looking for a problem — mention it, then justify the simpler threshold. That honesty is exactly what interviewers reward.

### 8. Extensions — "now add X"

Because the axes (filter / destination / format) are decoupled, each new requirement slots into exactly one place.

- **Async logging (buffer + flush).** Today `append()` writes synchronously — a slow RemoteAppender would block the request path, a real production hazard. Add an `AsyncAppender` (decorator) that wraps another appender: `append()` pushes the record onto an in-memory queue and returns immediately; a `setInterval` (or flush-on-size) worker drains the queue in the background and forwards to the wrapped appender. Add a `flush()` and hook it to process shutdown so nothing is lost. *No change to `Logger` or `Formatter`.*
- **Log rotation (FileAppender).** Give `FileAppender` a `maxBytes` (or date) policy. In `write()`, before appending, check the current size; if over the limit, rename `app.log → app.log.1` (shift older ones down) and open a fresh file. *Fully contained inside one appender.*
- **Per-module loggers with hierarchical levels.** Add `Logger.getLogger(name)` returning named child loggers keyed by dotted namespace (`payments`, `payments.refund`). Each child inherits its parent's level/appenders unless overridden — so you can set `payments` to DEBUG while the root stays at INFO. This is exactly how Log4j/Winston namespaces work. *A registry + parent pointer; the record/appender/formatter classes are untouched.*
- **Sampling for high-volume debug.** Wrap or configure the Logger with a `SamplingFilter`: for DEBUG records, keep only 1-in-N (e.g. a counter modulo N, or `Math.random() < rate`). Insert the check right after `shouldLog` in `log()`. Higher-severity records are never sampled. *One guard clause; the rest of the pipeline is unaware.*

Each extension touches a single seam — proof the decomposition (filter vs. destination vs. format vs. access) was the right one.

---

## Visual / Diagram description

The **sequence** of one `logger.warn("disk almost full", {pct:91})` call:

```
Caller        Logger              ConsoleApp        FileApp          RemoteApp
  │             │                     │                │                 │
  │ warn(msg) ─▶│                     │                │                 │
  │             │ shouldLog(WARN)?    │                │                 │
  │             │  WARN(2) >= INFO(1) │                │                 │
  │             │  → yes              │                │                 │
  │             │ new LogRecord(...)  │                │                 │
  │             │ ── append(rec) ────▶│                │                 │
  │             │        WARN>=DEBUG ✓ format(plain)   │                 │
  │             │        console.log("[..][WARN] disk..{pct:91}")        │
  │             │ ── append(rec) ─────────────────────▶│                 │
  │             │                        WARN>=INFO ✓ format(plain)→buffer│
  │             │ ── append(rec) ─────────────────────────────────────── ▶│
  │             │                                 WARN>=WARN ✓ format(JSON)→POST
  │             │                                                          │
```

**In prose:** the caller makes one call. The Logger runs the global threshold check *once*. If it passes, it builds a single immutable `LogRecord` and hands the *same* record to every appender in order. Each appender applies its *own* (possibly stricter) level gate, formats the record with its *own* strategy, and writes to its *own* destination — the console at DEBUG+ in plain text, the file at INFO+ in plain text, the remote at WARN+ in JSON. A DEBUG record would clear the console gate only; a WARN clears all three. That two-tier filtering (Logger threshold, then per-appender threshold) is what gives you both cheap global suppression and fine-grained per-destination control.

---

## Real world examples

### Winston (Node.js)
Winston models exactly this design: a `Logger` with a configurable `level`, an array of **transports** (its name for appenders — Console, File, HTTP, and third-party ones for Datadog/CloudWatch), and pluggable **formats** (`winston.format.json()`, `format.simple()`, `format.combine(...)`) that map directly onto our Strategy formatters. Each transport can carry its own `level`, precisely the per-appender threshold we built. Winston's default logger is effectively a singleton you `require`.

### Pino (Node.js)
Pino's headline feature is **speed via structured logging**: it emits newline-delimited **JSON only** (our `JsonFormatter`) and pushes formatting/transport work off the hot path — it writes minimal JSON in the main thread and ships heavier processing (pretty-printing, HTTP) to a **worker thread transport**. That's our async-logging extension taken to its logical end: the request path never waits on I/O. Pino is the go-to when logging throughput matters.

### Log4j / Logback (Java, representative)
The classic that popularized this vocabulary: **Loggers** (hierarchical, per-package, with inherited levels — our per-module extension), **Appenders** (Console, File, RollingFile with rotation, Socket, JDBC), and **Layouts** (our Formatters — `PatternLayout` for plain, `JsonLayout` for structured). If you describe your logging design using "logger / appender / layout / level," any interviewer immediately recognizes the lineage.

---

## Trade-offs

**Chain of Responsibility vs. simple threshold (for level filtering):**

| | Chain of Responsibility | Simple `if (level >= threshold)` |
|---|---|---|
| Clarity | More indirection, N handler objects | One line, obvious |
| Speed | Walks a chain | Single integer compare |
| Flexibility gained | ~None — levels are totally ordered | Same result |
| When it's right | If "handling" were genuinely branchy/pluggable per level | Almost always here |

**Singleton vs. dependency injection (for logger access):**

| | Singleton (`getInstance` / module export) | Dependency injection |
|---|---|---|
| Ease of use | Log from anywhere, zero wiring | Must pass logger in |
| Testability | Poor — global state, hard to fake/assert | Excellent — inject a fake, capture output |
| Config safety | One global config to reason about | Per-consumer config possible |
| Best for | App-wide default logger | Unit-tested classes/libraries |

**Synchronous vs. async appenders:**

| | Sync | Async (buffer + flush) |
|---|---|---|
| Latency on caller | Blocks on I/O (bad for remote) | Returns immediately |
| Loss on crash | None (already written) | Buffered records can be lost unless flushed |
| Complexity | Simple | Queue, flush, backpressure, shutdown hook |

**The sweet spot:** a Logger with a single global threshold (skip the CoR chain), a list of appenders each with its own level and an **injected** Strategy formatter, exposed as a singleton for app-wide convenience *but* injectable into classes you unit-test — and made **async** at the appender layer once remote/file I/O latency starts touching your request path.

---

## Common interview questions on this topic

### Q1: "Why would you use Chain of Responsibility for log levels, and would you actually?"
**Hint:** CoR lets each level-handler decide to handle or pass along, and it's the pattern interviewers expect you to *name*. But levels are totally ordered integers, so a single `if (level >= threshold)` is clearer, faster, and just as capable. State CoR, then justify the threshold — showing you resist pattern-itis is the real point.

### Q2: "How do you send plain text to the console but JSON to your log aggregator, simultaneously?"
**Hint:** Strategy. Each `Appender` holds an injected `Formatter`. Give the `ConsoleAppender` a `PlainFormatter` and the `RemoteAppender` a `JsonFormatter`. The same `LogRecord` is formatted differently per destination; the appender and formatter vary independently.

### Q3: "The logger is a global singleton. What's the downside and how do you keep code testable?"
**Hint:** It's global mutable state — hard to fake, easy to leak config between tests. Keep the singleton for the app-wide default, but **dependency-inject** a logger into classes you unit-test so a test can pass a capturing fake and assert on output. Present both; don't pretend the singleton is free.

### Q4: "Logging to a remote server adds 50ms per call. How do you stop that from slowing requests?"
**Hint:** Make it async. Wrap the remote appender so `append()` enqueues the record and returns immediately; a background worker flushes the queue. Add a shutdown flush so nothing is lost. Point at Pino's worker-thread transports as the production example.

### Q5: "How would you add per-module log levels — `payments` at DEBUG, everything else at INFO?"
**Hint:** A logger registry keyed by dotted namespace, with child loggers that inherit the parent's level/appenders unless overridden. `getLogger('payments')` returns a named logger you can set to DEBUG independently. That's exactly Log4j/Winston's hierarchy — and it touches only the Logger layer.

---

## Practice exercise

### Build and extend the framework (30-40 min)

1. Type out (don't copy-paste) the `LogLevel`, `LogRecord`, `Formatter`/`PlainFormatter`/`JsonFormatter`, `Appender`/`ConsoleAppender`/`FileAppender`/`RemoteAppender`, and `Logger` classes above, and run `node logger.js`. Confirm DEBUG/INFO are filtered out of the remote (WARN+) appender while WARN/ERROR/FATAL reach it as JSON.
2. **Add a `LogfmtFormatter`** (Strategy practice) that renders `level=WARN msg="disk almost full" pct=91`. Attach it to a fourth appender. You should touch *no* existing class — proof the Strategy seam works.
3. **Add an `AsyncAppender`** that wraps any appender: `append()` pushes to a queue; a `setInterval(…, 100)` drains it to the wrapped appender. Add a `flush()` method and call it before the demo exits. Wrap the `RemoteAppender` with it.
4. **Write one unit-style test WITHOUT the singleton:** construct a `Logger` with a single `FileAppender`, log two messages, and assert `fileAppender.dump()` has the expected lines. Notice how much easier this is than testing through `Logger.getInstance()` — that's the DI lesson.

**Produce:** a working `logger.js` plus the three additions, and a one-paragraph note on which patterns you actually needed versus which were textbook decoration.

---

## Quick reference cheat sheet

- **LogLevel** — ordered enum: `DEBUG(0) < INFO(1) < WARN(2) < ERROR(3) < FATAL(4)`. Ordering is what makes filtering a single compare.
- **Threshold filter** — `if (level.priority >= minLevel.priority)`. Cheap; drop a message before building/formatting it.
- **Chain of Responsibility** — the *textbook* level-filter answer (topic 47). Name it, but a threshold is usually clearer — beware pattern-itis.
- **Appender / Handler** — owns WHERE a record goes (console/file/remote). Holds its own formatter and its own minLevel.
- **Two-tier filtering** — Logger threshold (global, cheap) then per-appender threshold (fine-grained per destination).
- **Formatter** — owns HOW a record renders. **Strategy** pattern (topic 42): `PlainFormatter` vs. `JsonFormatter`, injected.
- **Structured (JSON) logs** — machine-parseable; what aggregators (Datadog/ELK/Loki) ingest. Tie to observability (topic 80).
- **Fanout** — one `LogRecord`, dispatched to ALL appenders. Observer-like; a plain array is the minimal, idiomatic form.
- **Singleton** — global logger access (topic 29). `Logger.getInstance()` or `module.exports = logger`. Convenient; hurts testability.
- **Dependency injection** — inject the logger into unit-tested classes so tests can fake/capture. Cleaner than reaching for the global.
- **Async logging** — buffer + background flush so slow I/O never blocks the request path (Pino's model). Flush on shutdown.
- **Log rotation** — roll the file by size/date so it doesn't grow unbounded. Contained in the FileAppender.
- **Robustness** — a broken appender must be try/caught so it never crashes the app or blocks sibling appenders.
- **The decomposition** — filter (Logger) / destination (Appender) / format (Formatter) / access (Singleton) are four independent seams; each new feature slots into exactly one.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [47 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md) | The pattern the textbook applies to level filtering — study it, then judge whether it beats a simple threshold here. |
| **Next** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) | Where your structured JSON logs go: aggregation, dashboards, alerting. Logging is the first pillar of observability. |
| **Related** | [42 — Strategy](./42-pattern-strategy.md) | The pattern behind pluggable formatters (Plain vs. JSON) injected into each appender. |
| **Related** | [29 — Singleton](./29-pattern-singleton.md) | The global-logger access pattern — and its testability caveat versus dependency injection. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step nouns→verbs→patterns method this doc follows; the general recipe for any LLD problem. |
