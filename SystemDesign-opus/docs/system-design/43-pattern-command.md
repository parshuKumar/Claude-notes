# 43 — Command Pattern
## Category: LLD Patterns

---

## What is this?

The **Command pattern** turns a *request* ("make this text bold") into a **standalone object** that carries everything needed to perform the work — the receiver, the arguments, and an `execute()` method. Because the request is now an object sitting in a variable, you can do things to it that you can't do to a plain function call: put it in a queue, save it to a log, run it later, run it again, or **undo** it.

Think of a restaurant order ticket. The waiter doesn't cook. The waiter writes a ticket ("Table 4: one pasta, no cheese") and pins it to a rail. The ticket is a physical object — it can be queued behind other tickets, re-fired if the dish burns, or cancelled. The chef never talks to the customer.

---

## Why does it matter?

Without Command, every operation in your system is a **method call** — and a method call is invisible. Once `editor.makeBold()` runs, it's gone. There is nothing left to inspect, retry, reverse, or store. Any feature that needs to *treat operations as data* becomes impossible without ugly hacks.

Features that are trivial with Command and painful without it:

| Feature | Why it needs Command |
|---|---|
| Undo / Redo | You need a stack of *things you did*, each knowing how to reverse itself |
| Job queues (BullMQ, SQS workers) | A job **is** a serialized command sitting in Redis until a worker picks it up |
| Audit logs / event sourcing | You store the commands, not the results — replay them to rebuild state |
| Scheduling ("send this email at 9am") | You must store the operation *now* and run it *later* |
| Retries with backoff | You need to hold onto the failed operation to try it again |

**Interview angle:** "Design a text editor with undo/redo" and "Design a task scheduler" are both Command-pattern questions in disguise. Interviewers also love the follow-up: *"How is Command different from Strategy?"* (Answer in the Related Patterns section — they look identical in code and are completely different in intent.)

**Real-work angle:** If you've ever dispatched a Redux action, pushed a job to BullMQ, or written a CQRS `CreateOrderCommand` handler — you have already used this pattern. This doc names it.

---

## The core idea — explained simply

### The Restaurant Order-Ticket Analogy

Picture a busy kitchen.

A customer tells the waiter: *"I'd like the pasta, no cheese."*

The waiter could walk into the kitchen and shout the order at the chef directly. That's a **method call** — synchronous, coupled, and gone the instant it's spoken. If the chef is busy, the waiter waits. If the customer changes their mind, there's nothing to cancel. If the dish burns, nobody remembers what was ordered.

Instead, the waiter **writes a ticket** and pins it to the rail. Now:

- The ticket **waits in line** (queue) — the chef pulls tickets when free.
- The ticket can be **re-fired** if the dish burns (retry).
- The ticket can be **pulled off the rail** before cooking starts (cancel).
- Spent tickets go in a spike on the counter — a **record of everything cooked tonight** (audit log).
- The chef never speaks to the customer. The chef only reads tickets. (**Decoupling.**)

The ticket is the Command object.

| Restaurant | Command Pattern | In our editor code |
|---|---|---|
| Customer | **Client** — decides what should happen | The code calling `invoker.run(...)` |
| Waiter + rail | **Invoker** — holds and fires commands, doesn't know how to cook | `Invoker` with its undo/redo stacks |
| Order ticket | **Command** — a request as an object | `BoldCommand`, `InsertTextCommand` |
| "Make pasta, no cheese" | Command's **parameters**, bound at creation | `new InsertTextCommand(doc, 5, "hi")` |
| Chef | **Receiver** — actually does the work | `Document` |
| Cooking the dish | `execute()` | `command.execute()` |
| Throwing the dish away | `undo()` | `command.undo()` |
| Spike of spent tickets | **Command history** | The undo stack |

The one thing that makes the whole pattern work: **the ticket carries its own arguments.** The invoker fires it without knowing what's on it.

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

Here's a text editor toolbar written without the pattern. It works. Now the product manager asks for Ctrl+Z.

```javascript
// BAD — operations are method calls. They vanish the moment they run.
class Editor {
  constructor() { this.text = ''; this.bold = false; }
  insertText(pos, str) { this.text = this.text.slice(0, pos) + str + this.text.slice(pos); }
  deleteRange(pos, len) { this.text = this.text.slice(0, pos) + this.text.slice(pos + len); }
  toggleBold() { this.bold = !this.bold; }
}

class Toolbar {
  constructor(editor) { this.editor = editor; }
  onBoldClick() { this.editor.toggleBold(); }
  onTypeKey(c)  { this.editor.insertText(this.editor.text.length, c); }

  undo() {
    // Now add undo. What do we even push onto the stack? We'd have to remember
    // WHICH method ran, with WHICH arguments, and what its inverse is. So we
    // start inventing: this.history.push({ type: 'INSERT', pos: 5, str: 'hi' })
    // ...plus a giant switch statement to reverse each type — a switch that
    // grows every time anyone adds a feature. THAT is the smell.
  }
}
```

Two problems, and they're the two problems Command always fixes:

1. **The request isn't reified.** ("Reify" = turn into a *thing*.) You cannot store, queue, or reverse a method call that already happened.
2. **The undo logic is centralised.** A `switch (type)` in the invoker means every new operation forces you to edit the invoker — violating the **Open/Closed Principle** (open for extension, closed for modification).

Command fixes both by moving "how do I do this" **and** "how do I undo this" *into the same small class*.

### 2. The structure — who plays which role

```
Client ──creates──▶ Command ──calls──▶ Receiver
                       ▲
                       │ holds & fires
                    Invoker
```

| Participant | Job | Rule of thumb |
|---|---|---|
| **Command** (interface) | Declares `execute()` and optionally `undo()` | Tiny. No business logic. |
| **ConcreteCommand** | Binds a receiver + arguments; implements `execute`/`undo` | Stores enough state to reverse itself |
| **Receiver** | Knows how to actually do the work | Has no idea Command exists |
| **Invoker** | Triggers commands; owns the history | Never uses `instanceof` — treats all commands identically |
| **Client** | Wires receiver + args into a concrete command | The only place that knows concrete classes |

The critical constraint: **the invoker must never branch on command type.** If it does, you've re-created the switch statement.

### 3. Making undo actually correct

A command can undo itself in one of two ways:

- **Inverse operation** — "I inserted 3 chars at position 5, so undo deletes 3 chars at position 5." Cheap in memory, but you must be sure the inverse is exact.
- **State snapshot (Memento)** — "Before I ran, the document looked like *this*; restore it." Always correct, but memory-heavy. This is the **Memento pattern** (topic 49), and it pairs with Command constantly.

Our `BoldCommand` and `InsertTextCommand` use inverse operations. Our `DeleteCommand` *must* snapshot — you can't un-delete text you didn't save.

**The redo rule that everyone gets wrong:** when the user performs a *new* command after undoing, the redo stack must be **cleared**. Otherwise you can redo a future that no longer exists.

### 4. Full JavaScript implementation

```javascript
// ─── RECEIVER ────────────────────────────────────────────────
// The Document knows nothing about commands, undo, or history.
// It just knows how to mutate itself. Keep receivers dumb.
class Document {
  constructor(text = '') {
    this.text = text;
    this.boldRanges = []; // [start, end] pairs, kept simple on purpose
  }

  insert(pos, str) { this.text = this.text.slice(0, pos) + str + this.text.slice(pos); }
  delete(pos, len) { this.text = this.text.slice(0, pos) + this.text.slice(pos + len); }
  read(pos, len)   { return this.text.slice(pos, pos + len); }

  addBold(start, end)    { this.boldRanges.push([start, end]); }
  removeBold(start, end) {
    const i = this.boldRanges.findIndex(([s, e]) => s === start && e === end);
    if (i !== -1) this.boldRanges.splice(i, 1);
  }

  render() {
    const marks = this.boldRanges.map(([s, e]) => `${s}-${e}`).join(',') || 'none';
    return `"${this.text}"  [bold: ${marks}]`;
  }
}

// ─── COMMAND INTERFACE ───────────────────────────────────────
// JS has no interfaces, so we use a base class that throws.
// Subclasses that forget a method fail loudly instead of silently.
class Command {
  execute() { throw new Error('execute() not implemented'); }
  undo()    { throw new Error('undo() not implemented'); }
  get label() { return this.constructor.name; }
}

// ─── CONCRETE COMMANDS ───────────────────────────────────────
class InsertTextCommand extends Command {
  constructor(doc, pos, str) {
    super();
    this.doc = doc;   // the receiver
    this.pos = pos;   // arguments, bound at creation time
    this.str = str;
  }
  execute() { this.doc.insert(this.pos, this.str); }
  // Inverse operation: delete exactly what we inserted.
  undo()    { this.doc.delete(this.pos, this.str.length); }
  get label() { return `Insert "${this.str}" @${this.pos}`; }
}

class DeleteCommand extends Command {
  constructor(doc, pos, len) {
    super();
    this.doc = doc;
    this.pos = pos;
    this.len = len;
    this.deleted = null; // snapshot — we CANNOT reverse a delete without it
  }
  execute() {
    // Capture before mutating. This is the Memento pattern in miniature.
    this.deleted = this.doc.read(this.pos, this.len);
    this.doc.delete(this.pos, this.len);
  }
  undo() { this.doc.insert(this.pos, this.deleted); }
  get label() { return `Delete ${this.len} @${this.pos}`; }
}

class BoldCommand extends Command {
  constructor(doc, start, end) {
    super();
    this.doc = doc;
    this.start = start;
    this.end = end;
  }
  execute() { this.doc.addBold(this.start, this.end); }
  undo()    { this.doc.removeBold(this.start, this.end); }
  get label() { return `Bold ${this.start}-${this.end}`; }
}

// ─── MACRO / COMPOSITE COMMAND ───────────────────────────────
// A command made of commands. Because it also implements execute/undo,
// the invoker cannot tell it apart from a leaf command. (Composite, topic 38.)
class MacroCommand extends Command {
  constructor(name, commands) {
    super();
    this.name = name;
    this.commands = commands;
  }
  execute() { this.commands.forEach(c => c.execute()); }
  // Undo in REVERSE order — later commands may depend on earlier ones.
  undo()    { [...this.commands].reverse().forEach(c => c.undo()); }
  get label() { return `Macro[${this.name}]`; }
}

// ─── INVOKER ─────────────────────────────────────────────────
// Notice: zero knowledge of Document, zero `instanceof`, zero switch.
// Adding a new command type requires NO change here. That is the payoff.
class Invoker {
  constructor(limit = 100) {
    this.undoStack = [];
    this.redoStack = [];
    this.limit = limit; // bound memory — an infinite history is a leak
  }

  run(command) {
    command.execute();
    this.undoStack.push(command);
    if (this.undoStack.length > this.limit) this.undoStack.shift();
    // A new action invalidates any redoable future.
    this.redoStack.length = 0;
    return command;
  }

  undo() {
    const command = this.undoStack.pop();
    if (!command) return null;
    command.undo();
    this.redoStack.push(command);
    return command;
  }

  redo() {
    const command = this.redoStack.pop();
    if (!command) return null;
    command.execute(); // re-running execute() must be safe — keep commands idempotent-ish
    this.undoStack.push(command);
    return command;
  }

  history() { return this.undoStack.map(c => c.label); }
}

// ─── DEMO ────────────────────────────────────────────────────
function main() {
  const doc = new Document();
  const invoker = new Invoker();
  const show = (msg) => console.log(`${msg.padEnd(26)} ${doc.render()}`);

  invoker.run(new InsertTextCommand(doc, 0, 'Hello'));  show('insert "Hello"');
  invoker.run(new InsertTextCommand(doc, 5, ' world')); show('insert " world"');
  invoker.run(new BoldCommand(doc, 0, 5));             show('bold 0-5');
  invoker.run(new DeleteCommand(doc, 5, 6));           show('delete " world"');

  console.log('\nhistory:', invoker.history());

  invoker.undo(); show('undo (delete)');   // " world" comes back from the snapshot
  invoker.undo(); show('undo (bold)');     // bold removed
  invoker.redo(); show('redo (bold)');     // bold returns

  // A macro: three commands the Invoker cannot tell apart from a single one.
  invoker.run(new MacroCommand('Shout', [
    new DeleteCommand(doc, 0, 5),
    new InsertTextCommand(doc, 0, 'HELLO'),
    new InsertTextCommand(doc, doc.text.length, '!!!'),
  ]));
  show('run macro');

  invoker.undo(); show('undo macro (all 3)');  // ONE Ctrl+Z reverses all three steps
}

main();
```

Running it prints (abridged):

```
insert "Hello"             "Hello"  [bold: none]
bold 0-5                   "Hello world"  [bold: 0-5]
delete " world"            "Hello"  [bold: 0-5]
undo (delete)              "Hello world"  [bold: 0-5]   ← deleted text restored from snapshot
undo (bold)                "Hello world"  [bold: none]
redo (bold)                "Hello world"  [bold: 0-5]
run macro                  "HELLO world!!!"  [bold: 0-5]
undo macro (all 3)         "Hello world"  [bold: 0-5]   ← ONE Ctrl+Z, three operations reversed
```

The macro is the moment the pattern earns its keep: **one Ctrl+Z undid three operations**, and the `Invoker` needed not a single line of new code.

### 5. The other superpowers Command unlocks

Undo is the famous one. These are the ones that matter more at work.

```javascript
// (a) QUEUING — a job IS a command. This is BullMQ/SQS/Sidekiq in eight lines.
class CommandQueue {
  constructor() { this.queue = []; }
  enqueue(cmd) { this.queue.push(cmd); }
  async drain() {
    while (this.queue.length) await this.queue.shift().execute();
  }
}

// (b) LOGGING + REPLAY — because a command is data, you can serialize it.
class CommandLog {
  constructor() { this.entries = []; }
  record(cmd) { this.entries.push({ at: Date.now(), type: cmd.constructor.name, args: cmd.toJSON() }); }
  // Rebuild any past state by replaying from the beginning. That's EVENT SOURCING.
  replay(registry, receiver) {
    for (const e of this.entries) registry[e.type].fromJSON(receiver, e.args).execute();
  }
}

// (c) RETRYING — the failed command is still in your hand. You cannot retry a
// method call you already made; you CAN retry an object you still hold.
async function withRetry(cmd, attempts = 3) {
  for (let i = 1; i <= attempts; i++) {
    try { return await cmd.execute(); }
    catch (err) {
      if (i === attempts) throw err;
      await new Promise(r => setTimeout(r, 2 ** i * 100)); // exponential backoff
    }
  }
}
```

**(d) Scheduling** — hold the command, fire it later: `setTimeout(() => cmd.execute(), runAt - Date.now())`, or persist `{ runAt, command }` to a table and let a poller fire due rows.

**(e) Rate limiting, permissions, dry-runs** — all become middleware around `execute()`, because there is now a single choke point every operation flows through.

Store the command log instead of the state, and your state becomes a *derived value*. That's **event sourcing** — see [68 — Event-Driven Architecture](./68-event-driven-architecture.md).

### 6. A real Node.js ecosystem example

**Redux** is Command with different vocabulary. An action `{ type: 'todo/added', payload: 'buy milk' }` is a serialized command; `dispatch` is the invoker; the reducer is the receiver. Redux DevTools gives you time-travel debugging for free — *because every operation is an object it can keep.* That's not a Redux feature. That's the Command pattern's feature.

**BullMQ / Bee-Queue:** `queue.add('sendEmail', { to, subject })` writes a command into Redis. A worker in a different process, possibly on a different machine, deserializes and executes it. The Command object crossed a network boundary — which is only possible because it's data, not a call.

**CQRS** (Command Query Responsibility Segregation, topic 68): the "C" is literally this pattern. `CreateOrderCommand` goes to a `CreateOrderHandler`. Reads take a different path entirely.

### 7. When NOT to use it / how it's abused

- **Every method becomes a class.** If a command has no undo, no queue, no log, and no retry, it's a function with extra steps. `new SaveCommand(doc).execute()` is strictly worse than `doc.save()`.
- **The 200-command codebase.** Teams that adopt CQRS zealously end up with `GetUserByIdCommand`, `GetUserByEmailCommand`... Commands are for *writes* and *operations*, not reads.
- **Commands with business logic inside.** The command should orchestrate the receiver, not *be* the receiver. If `execute()` is 80 lines, your receiver is anaemic.
- **Undo that lies.** An inverse that isn't *exactly* inverse (floating-point math, non-deterministic side effects, an already-sent email) silently corrupts state. If you can't reverse it exactly, snapshot it — or don't offer undo for it.

### 8. Related patterns and how they differ

| Pattern | Relationship |
|---|---|
| **Strategy** (42) | Both wrap behaviour in an object. **Strategy answers "HOW"** — swap one algorithm for another (`QuickSort` vs `MergeSort`), same input, same output, no history. **Command answers "WHAT"** — a specific request with its arguments baked in, meant to be stored and possibly reversed. Strategy is a *parameter*; Command is a *record*. |
| **Memento** (49) | Command's usual undo partner. Memento stores the *state before*; Command stores the *operation*. Big states → Memento. Small reversible ops → inverse `undo()`. |
| **Composite** (38) | `MacroCommand` is literally a Composite of Commands. |
| **Chain of Responsibility** (47) | CoR passes ONE request through MANY handlers until one takes it. Command hands ONE request to ONE known receiver. |
| **Observer** (41) | Observer *broadcasts* "something happened" (past tense, many listeners). Command *instructs* "do this" (imperative, one receiver). Events vs commands — a distinction that shows up again in event-driven architecture. |

---

## Visual / Diagram description

### Diagram 1 — Class structure

```
        ┌──────────────────────┐
        │       Client         │  creates commands, binds receiver + args
        │  (Toolbar / handler) │
        └──────────┬───────────┘
                   │ creates
                   ▼
        ┌──────────────────────┐        ┌────────────────────────┐
        │   «abstract»         │        │       Invoker          │
        │      Command         │◀───────│  - undoStack: Cmd[]    │
        │──────────────────────│  holds │  - redoStack: Cmd[]    │
        │ + execute()          │        │────────────────────────│
        │ + undo()             │        │ + run(cmd)             │
        │ + label              │        │ + undo()  + redo()     │
        └──────────▲───────────┘        └────────────────────────┘
                   │ extends
   ┌───────────────┼──────────────┬──────────────────┐
┌──┴──────────┐ ┌──┴─────────┐ ┌──┴────────┐  ┌──────┴────────┐
│InsertText   │ │Delete      │ │Bold       │  │Macro          │
│Command      │ │Command     │ │Command    │  │Command        │
│- doc        │ │- doc       │ │- doc      │  │- commands[]   │
│- pos, str   │ │- pos, len  │ │- start,end│  │ (Composite!)  │
│             │ │- deleted ◀─┼─┼─ snapshot │  │               │
└──────┬──────┘ └─────┬──────┘ └─────┬─────┘  └───────────────┘
       └──────────────┴──────────────┘
                      │ acts on
                      ▼
             ┌──────────────────┐
             │    Document      │   ◀── RECEIVER
             │ + insert(pos,s)  │       knows NOTHING about commands
             │ + delete(pos,n)  │
             │ + addBold(s,e)   │
             └──────────────────┘
```

The single most important arrow is the one that **isn't there**: `Invoker` has no arrow to `Document`. The invoker can run, undo, and redo operations on a document it has never heard of. That decoupling is why the same `Invoker` class would work for a drawing app, a spreadsheet, or a database migration runner.

### Diagram 2 — Sequence: run, then undo

```
Client        Invoker         BoldCommand        Document
  │              │                  │                │
  │ new BoldCommand(doc, 0, 5) ────▶│ (receiver + args stored inside)
  │              │                  │                │
  │ run(cmd) ───▶│ execute() ──────▶│ addBold(0,5) ─▶│  text now bold
  │              │                  │                │
  │              │ undoStack.push(cmd); redoStack.clear()
  │◀─────────────┤                  │                │
  │              │                  │                │
  │ undo() ─────▶│ cmd = undoStack.pop()             │
  │              │ undo() ─────────▶│ removeBold(0,5)▶  bold gone
  │              │ redoStack.push(cmd)               │
  │◀─────────────┤                  │                │
```

Read the vertical lines as objects and the horizontal arrows as calls. Notice the Invoker's arrows only ever point at `Command` — never at `Document`.

---

## Real world examples

### 1. Redux (and the Flux family)

Every state change in a Redux app is an **action object** — `{ type, payload }` — dispatched to a store. That's a serialized command. Because these objects are recorded in order, Redux DevTools can replay them, step backwards through them, and export the full session as JSON for a bug report. Time-travel debugging isn't magic; it's a consequence of reifying every operation as an object. The reducer is the receiver, `dispatch` is the invoker.

### 2. Git

Every `git commit` is a stored, replayable operation on a tree. `git revert` creates the *inverse* command (a new commit that undoes the previous one) rather than mutating history — exactly the "inverse operation" undo strategy. `git cherry-pick` re-executes a command in a different context. `git rebase` is a macro: replay a list of commands onto a new base. Git's entire mental model is a persistent command log.

### 3. Job queues at scale (BullMQ, AWS SQS, Sidekiq)

When an e-commerce site sends an order-confirmation email, the web request doesn't send it. The request writes a command — a JSON blob naming a handler and its arguments — to a queue, then returns immediately. A worker pool drains the queue. Commands that throw get retried with exponential backoff; commands that keep failing land in a **dead-letter queue** for a human to inspect. Retry, backoff, DLQ, scheduling, and priority are all only possible because the operation is a **durable object** rather than a stack frame. See [67 — Message Queues](./67-message-queues.md).

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| Undo / redo | Falls out almost for free once commands own their inverse |
| Decoupling | Invoker doesn't know receivers; receivers don't know commands |
| Open/Closed | New operation = new class. No existing file is edited. |
| Operations become data | Queue them, log them, schedule them, ship them over a network |
| Composability | Macro commands via Composite, with no invoker changes |
| A single choke point | Logging, auth, rate limiting, metrics wrap `execute()` once |

| Cons | The cost you pay |
|---|---|
| Class explosion | 30 operations = 30 classes. A lot of files for small apps. |
| Indirection | Reading the code, you can't jump straight from click to logic |
| Memory | An undo stack holds objects (and snapshots) — always bound it |
| Undo correctness is hard | External side effects (sent emails, charged cards) can't be truly reversed |

**Rule of thumb:** Reach for Command the moment an operation needs a **second life** — to be undone, queued, retried, logged, scheduled, or sent across a process boundary. If an operation only ever needs to happen once, right now, in this process, a plain method call is the better design. Don't pay for a ticket rail when there's one customer and one chef.

---

## Common interview questions on this topic

### Q1: "Design undo/redo for a text editor."
**Hint:** Command + two stacks. Each command stores its receiver and arguments and implements `execute()`/`undo()`. `run()` pushes to the undo stack **and clears the redo stack** (a new action kills the redoable future). `undo()` pops from undo → executes `undo()` → pushes to redo. Mention bounding the stack size, and that irreversible ops (like `delete`) must snapshot the removed data — that's the Memento pattern.

### Q2: "What's the difference between Command and Strategy? They look identical."
**Hint:** They *are* structurally identical (an object wrapping behaviour) and semantically opposite. **Strategy = HOW**: interchangeable algorithms for the same job, chosen at runtime, no memory of being used, no arguments baked in (`sorter.setStrategy(new QuickSort())`). **Command = WHAT**: a specific request with its receiver and arguments pre-bound, designed to be *stored* — queued, logged, undone. Test: does it make sense to keep a history of them? Then it's a Command.

### Q3: "How would you implement a job queue using this pattern?"
**Hint:** A job *is* a command. Producer builds `new SendEmailCommand(mailer, to, subject)`, serializes it to `{ type, args }`, pushes to Redis. A worker pops it, looks `type` up in a handler registry, deserializes the args, and calls `execute()`. Retry = re-execute the same object with exponential backoff; N failures → dead-letter queue. Emphasize idempotency: `execute()` may run more than once (at-least-once delivery), so it must be safe to repeat.

### Q4: "How do you undo something irreversible, like sending an email or charging a card?"
**Hint:** You don't — you **compensate**. The undo is a new forward action ("send an apology", "issue a refund"), not a reversal. This is the **Saga / compensating transaction** pattern in distributed systems. Say clearly that an honest design *refuses* to expose undo for irreversible commands rather than pretending.

### Q5: "What happens if a MacroCommand fails halfway through?"
**Hint:** You need atomicity. Track which sub-commands succeeded, and on failure call `undo()` on those in reverse order — a mini rollback. Note the catch: if one sub-command was irreversible, you can't fully roll back, and you must either forbid such commands inside macros or accept a compensating-action model.

---

## Practice exercise

### Build a Smart Home Remote with Undo, Macros, and a Scheduler

Write a single Node file (`remote.js`) that runs with `node remote.js`.

**Receivers** (dumb devices, zero knowledge of commands): `Light` (`on`, `off`, `dim(pct)`), `Thermostat` (`setTemp(c)`), `Speaker` (`play(track)`, `stop()`).

**Commands** (each with `execute()` and `undo()`): `LightOnCommand`, `LightOffCommand`, `DimCommand`, `SetTempCommand`, `PlayCommand`. Watch the trap: `DimCommand` and `SetTempCommand` must store the **previous** value to undo correctly — not just reset to a hardcoded default.

**Invoker:** a `RemoteControl` with 4 slots. `pressButton(slot)` runs that slot's command and pushes it onto an undo stack (capped at 10). `pressUndo()` reverses the last action.

**Then extend it three ways:**
1. A `MacroCommand` **"Movie Night"** = dim lights to 20% + set temp to 21 + play "intro.mp3". One `pressUndo()` must reverse all three, in reverse order.
2. A `Scheduler` with `schedule(command, delayMs)` that fires the command later via `setTimeout` — and prove the *same command objects* work unchanged.
3. Give every command a `toJSON()`, then write `replay(log)` that rebuilds device state from a saved array of commands. Reset the devices, replay, confirm the final state matches.

**Produce:** the file, plus a console demo showing state after each press, an undo, a macro undo, and a successful replay. **The success criterion:** you added the macro and the scheduler *without editing `RemoteControl` at all*. If you had to touch it, your invoker knows too much.

---

## Quick reference cheat sheet

- **One line:** Command turns a request into an object with `execute()` — so it can be stored, queued, logged, retried, or undone.
- **Participants:** Client (creates) → Command (`execute`/`undo`) → Receiver (does the work); Invoker holds and fires.
- **The invoker must never branch on command type.** No `switch`, no `instanceof`. That's the whole point.
- **Undo = inverse op or snapshot.** Cheap reversible ops → inverse. Destructive ops → snapshot (Memento).
- **A new command clears the redo stack.** Forget this and you can redo a future that never happened.
- **Undo macros in reverse order.** Later steps may depend on earlier ones.
- **A queued job IS a command; a Redux action IS a command.** Same idea, serialized. Time-travel debugging is a free consequence.
- **Command vs Strategy:** Command = *what to do* (stored, has args, reversible). Strategy = *how to do it* (swappable algorithm, stateless).
- **Irreversible side effects can't be undone** — only *compensated* (refund, apology email). Say so explicitly in interviews.
- **Bound your undo stack.** An unbounded history is a memory leak with a nice name.
- **Don't use it** when the operation happens once, now, in-process, and never needs a second life.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [42 — Strategy Pattern](./42-pattern-strategy.md) — the pattern Command is most often confused with; read both, then read the comparison table above again |
| **Next** | [44 — Iterator Pattern](./44-pattern-iterator.md) — traverse a collection without exposing its internals |
| **Related** | [49 — Memento Pattern](./49-pattern-memento.md) — how Command implements undo when an operation can't be inverted |
| **Related** | [67 — Message Queues and Async Processing](./67-message-queues.md) — where commands go to live when you serialize them |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — CQRS and event sourcing, both built directly on this pattern |
