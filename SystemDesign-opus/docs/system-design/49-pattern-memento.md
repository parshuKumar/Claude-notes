# 49 — Memento Pattern

## Category: LLD Patterns

---

## What is this?

The **Memento pattern** lets you take a snapshot of an object's internal state, hand that snapshot to somebody else for safekeeping, and later use it to restore the object — **without ever exposing the object's private fields to the outside world**.

Think of a **save game**. When you save, the game writes a save file. You can copy it, back it up, delete it, keep 20 of them. But you cannot open the save file in a text editor and set your gold to 999,999 — it's opaque to you. Only the *game* knows how to read it. You hold the file; the game holds the knowledge of what's inside it. That separation is the entire pattern.

---

## Why does it matter?

Because **undo is one of the most requested features in software, and the naive implementation destroys your encapsulation.**

Here's the trap. To restore an object later, you need its state. So the obvious move is to make the fields public, or add `getCursorPosition()`, `getSelection()`, `getUndoBuffer()`, `getFontStack()` getters. Now every internal detail of your editor is part of its public API — and the "history" class outside is now coupled to your internals. Change one private field and the history class breaks. You have turned a private implementation detail into a public contract, permanently.

Memento is the pattern that says: **you can have the snapshot without the getters.**

**What breaks without it:**
- Undo/redo that half-works because you forgot to restore one field.
- A history class that stops compiling every time you refactor the editor.
- "Rollback" that mutates a shared object because you saved a *reference*, not a *copy*.

**Interview angle:** Memento is a standard follow-up on any editor/undo/game/transaction question. The interviewer's real probe is: **"snapshot or diff?"** — and the memory cost conversation that follows. Getting that right separates a mid-level answer from a senior one.

**Real-work angle:** database savepoints, git stash, React DevTools time-travel, VM snapshots, `structuredClone` checkpoints in a workflow engine — all mementos. You will implement one the first time a PM says "can we add undo?"

---

## The core idea — explained simply

### The Save Game Analogy

You're 40 hours into an RPG. Before the boss fight you hit **Save**.

**What actually happens:**

1. The **game** (which knows everything about your character — hidden stats, RNG seed, quest flags, item durability) serializes all of it into a save file.
2. The **save file** is a blob. It's opaque. It's not a menu of your stats; it's a sealed envelope.
3. The **save manager** (the UI listing "Slot 1, Slot 2, Slot 3") holds these files. It can list them, name them, delete them, sort them by date. It **cannot read them**. It does not know what a "quest flag" is.
4. You die to the boss. You hit **Load**. The save manager hands the file back to the game. The **game** — and only the game — knows how to unpack it and put every hidden stat back exactly where it was.

Now notice the design property that matters: the save manager was written by a UI developer who has **no idea** what's inside a character. When the game team adds a new hidden stat next patch, the save manager **does not change**. That's Memento.

**Mapping the analogy:**

| Save game | Memento pattern | In the editor example |
|---|---|---|
| The game / your character | **Originator** — owns the state, creates and restores snapshots | `Editor` |
| The save file (opaque blob) | **Memento** — the snapshot; nobody but the originator can read it | `EditorSnapshot` |
| The save-slot manager UI | **Caretaker** — stores and orders mementos, never peeks inside | `History` |
| "Save" button | `originator.save()` → returns a Memento | `editor.save()` |
| "Load" button | `originator.restore(memento)` | `editor.restore(snap)` |
| Save manager can't edit your gold | The caretaker has **no access** to the memento's fields | `#state` private field |
| Save file gets bigger every patch | Memory cost — full snapshots are expensive | the diff-vs-snapshot trade-off |

The word "memento" just means *a keepsake, a thing that helps you remember*. That's literally all it is: a keepsake of a past state.

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

You want undo in a text editor. Here's the version everyone writes first.

```javascript
// ---------- BAD #1: expose everything, let the outside world copy it ----------

class Editor {
  constructor() {
    this.content = '';           // public
    this.cursor = 0;             // public
    this.selectionStart = 0;     // public
    this.selectionEnd = 0;       // public
    this.fontSize = 14;          // public
  }
  type(text) {
    this.content =
      this.content.slice(0, this.cursor) + text + this.content.slice(this.cursor);
    this.cursor += text.length;
  }
}

class History {
  constructor() { this.stack = []; }

  // The History class must know EVERY field of Editor. That is the disease.
  backup(editor) {
    this.stack.push({
      content: editor.content,
      cursor: editor.cursor,
      selectionStart: editor.selectionStart,
      selectionEnd: editor.selectionEnd,
      fontSize: editor.fontSize,
      // ...and tomorrow, when someone adds `editor.theme`, they will forget this line,
      //    undo will silently stop restoring the theme, and nobody will notice for 3 months.
    });
  }

  undo(editor) {
    const s = this.stack.pop();
    editor.content = s.content;
    editor.cursor = s.cursor;
    editor.selectionStart = s.selectionStart;
    editor.selectionEnd = s.selectionEnd;
    editor.fontSize = s.fontSize;
  }
}
```

Everything is wrong here:

1. **Encapsulation is gone.** `Editor` has no private state left. Every field is public *forever* because `History` depends on it.
2. **The field list is duplicated in three places** (constructor, `backup`, `undo`) and they will drift apart.
3. **`History` is coupled to `Editor`'s internals.** Refactor `cursor` into a `Cursor` object and `History` breaks.
4. It's a **silent** failure mode. A missed field doesn't throw; undo just quietly does the wrong thing.

The other bad version is even more common:

```javascript
// ---------- BAD #2: "just save the object" ----------
class History {
  backup(editor) {
    this.stack.push(editor);        // saves a REFERENCE, not a copy
  }
  undo() {
    return this.stack.pop();        // ...which is the same, already-mutated editor. Undo does nothing.
  }
}
```

This one is a classic bug in real code. Every entry in `stack` points at the *same live object*. You've built a stack of identical pointers. Undo appears to work in the demo (because you only tested one step) and fails in production.

### 2. The structure — the three roles

Memento has exactly three participants, and **the discipline of the pattern is entirely about who is allowed to read what.**

| Role | Responsibility | Can it read the memento's guts? |
|---|---|---|
| **Originator** | Owns the real state. Creates mementos (`save()`), restores itself from one (`restore(m)`). | **Yes** — it's the only one that can |
| **Memento** | An immutable, opaque snapshot of the originator's state at one moment | It *is* the guts |
| **Caretaker** | Stores mementos (a stack, a list, a DB). Decides *when* to save and undo. | **No** — never |

The caretaker's ignorance is the point. It can only do three things with a memento: **hold it, order it, and hand it back**.

```
   Caretaker ──── holds ────▶ Memento (opaque)
       │                          ▲
       │ save()/restore()         │ creates & reads
       ▼                          │
   Originator ────────────────────┘
```

In Java, this is enforced with nested classes and package-private accessors. **In JavaScript we enforce it with `#private` fields**, which are a true hard-privacy mechanism — reaching for `snapshot.#state` from outside the class is a *syntax error*, not a runtime convention. That makes JS an unusually good language for demonstrating this pattern honestly.

### 3. Full JavaScript implementation — Editor + Snapshot + History

```javascript
// ============================================================
//  MEMENTO — the opaque snapshot
// ============================================================

class EditorSnapshot {
  // TRUE privacy. Not `_state`. Not a convention. A hard language-level barrier.
  #state;
  #takenAt;

  constructor(state) {
    // Defensive copy on the way IN: the caller can't mutate our copy afterwards.
    this.#state = structuredClone(state);
    this.#takenAt = Date.now();
    Object.freeze(this);
  }

  // The ONLY thing the outside world may know: a human label for the history UI.
  // This is deliberate — the caretaker needs to render "Undo: typing" without
  // learning anything about cursors or fonts.
  get label() {
    return `${new Date(this.#takenAt).toISOString().slice(11, 19)} — ${this.#state.content.length} chars`;
  }

  // "Narrow interface": only the Originator is meant to call this.
  // We can't make it package-private in JS, so we guard it with a token
  // that only the Originator holds. See section 4 for a stricter alternative.
  _reveal(token) {
    if (token !== EditorSnapshot.ACCESS_TOKEN) {
      throw new Error('Only the Originator may read a snapshot.');
    }
    return structuredClone(this.#state);   // defensive copy on the way OUT too
  }

  static ACCESS_TOKEN = Symbol('editor-snapshot-access');
}

// ============================================================
//  ORIGINATOR — owns the state, makes and eats snapshots
// ============================================================

class Editor {
  #content = '';
  #cursor = 0;
  #selection = { start: 0, end: 0 };
  #fontSize = 14;

  // ---- normal editing operations (mutate private state) ----
  type(text) {
    this.#content =
      this.#content.slice(0, this.#cursor) + text + this.#content.slice(this.#cursor);
    this.#cursor += text.length;
    return this;
  }

  deleteBack(n = 1) {
    const start = Math.max(0, this.#cursor - n);
    this.#content = this.#content.slice(0, start) + this.#content.slice(this.#cursor);
    this.#cursor = start;
    return this;
  }

  select(start, end) { this.#selection = { start, end }; return this; }
  setFontSize(px)    { this.#fontSize = px; return this; }

  // ---- MEMENTO API ----

  /** Package up ALL private state into an opaque snapshot. */
  save() {
    return new EditorSnapshot({
      content: this.#content,
      cursor: this.#cursor,
      selection: this.#selection,
      fontSize: this.#fontSize,
      // Add a new private field tomorrow? You add ONE line here.
      // The History class does not change. That is the win.
    });
  }

  /** Only the Editor knows how to unpack an EditorSnapshot. */
  restore(snapshot) {
    if (!(snapshot instanceof EditorSnapshot)) {
      throw new TypeError('restore() expects an EditorSnapshot');
    }
    const s = snapshot._reveal(EditorSnapshot.ACCESS_TOKEN);
    this.#content = s.content;
    this.#cursor = s.cursor;
    this.#selection = s.selection;
    this.#fontSize = s.fontSize;
  }

  // Read-only view for rendering. Note: this exposes NOTHING that undo needs.
  render() {
    return `"${this.#content}" [cursor=${this.#cursor}, font=${this.#fontSize}px]`;
  }
}

// ============================================================
//  CARETAKER — holds history, never peeks inside
// ============================================================

class History {
  #undoStack = [];
  #redoStack = [];
  #limit;

  constructor(editor, { limit = 50 } = {}) {
    this.editor = editor;
    this.#limit = limit;            // memory guard — see the trade-offs section
  }

  /** Call BEFORE a mutating command, so we capture the pre-change state. */
  backup() {
    this.#undoStack.push(this.editor.save());
    if (this.#undoStack.length > this.#limit) this.#undoStack.shift();  // drop oldest
    this.#redoStack = [];           // a new action invalidates the redo branch
  }

  undo() {
    if (this.#undoStack.length === 0) return false;
    this.#redoStack.push(this.editor.save());       // remember where we were
    this.editor.restore(this.#undoStack.pop());
    return true;
  }

  redo() {
    if (this.#redoStack.length === 0) return false;
    this.#undoStack.push(this.editor.save());
    this.editor.restore(this.#redoStack.pop());
    return true;
  }

  // The caretaker can build a whole undo MENU knowing nothing about editors.
  list() { return this.#undoStack.map((m) => m.label); }
}

// ============================================================
//  DEMO
// ============================================================

function main() {
  const editor = new Editor();
  const history = new History(editor, { limit: 5 });

  history.backup(); editor.type('Hello');
  history.backup(); editor.type(', world');
  history.backup(); editor.setFontSize(20);
  history.backup(); editor.type('!!!');

  console.log('now      :', editor.render());
  // "Hello, world!!!" [cursor=15, font=20px]

  history.undo();
  console.log('undo x1  :', editor.render());   // the "!!!" is gone

  history.undo();
  console.log('undo x2  :', editor.render());   // font back to 14 — a NON-text field restored too

  history.redo();
  console.log('redo x1  :', editor.render());   // font 20 again

  console.log('history  :', history.list());

  // ---- THE PROOF that encapsulation holds ----
  const snap = editor.save();
  console.log('label    :', snap.label);        // fine — the label is public by design
  try {
    snap._reveal('guessed-token');              // caretaker tries to peek
  } catch (e) {
    console.log('peek     :', e.message);       // "Only the Originator may read a snapshot."
  }
  // And this line is not even *runnable* JavaScript — it is a SyntaxError at parse time:
  //     console.log(snap.#state);
  //     ^ Private field '#state' must be declared in an enclosing class
}

main();
```

Run it with `node editor.js`.

**The two things to notice:**

1. `History` contains the word `Editor` only as a type it holds. It never touches `content`, `cursor`, `selection`, or `fontSize`. Add a `theme` field to `Editor` tomorrow and `History` **does not change**.
2. `snapshot.#state` is not merely "discouraged" — it is **impossible**. The parser rejects it. This is why `#private` fields are the right tool for teaching Memento in JS: the encapsulation claim is not a promise, it's enforced.

### 4. Two implementation notes worth knowing

**a) The token guard vs the closure trick.** The `ACCESS_TOKEN` above is a readable way to signal "originator only", but a determined caller could still grab `EditorSnapshot.ACCESS_TOKEN`. If you want it airtight, put the reader function *inside* the memento as a closure that the originator captures at creation time:

```javascript
class Editor {
  #content = '';
  save() {
    const state = { content: this.#content /* ... */ };
    // The snapshot object exposes only `label`. The state lives in a closure
    // that only THIS editor instance can invoke, because it holds the fn.
    const reader = () => structuredClone(state);
    const memento = Object.freeze({ label: `${state.content.length} chars` });
    this.#readers.set(memento, reader);      // WeakMap: memento -> reader
    return memento;
  }
  #readers = new WeakMap();
  restore(memento) {
    const read = this.#readers.get(memento);
    if (!read) throw new Error('Foreign or expired memento');
    Object.assign(this, read());   // (assign to #fields explicitly in real code)
  }
}
```
A `WeakMap` keyed by the memento is the most idiomatic JS way to give the originator privileged access without exposing anything. It also garbage-collects cleanly.

**b) Deep copy is not optional.** `structuredClone(state)` (built into Node 17+) is what makes the snapshot a *snapshot*. If you write `this.#state = state`, you have saved a **reference**, and any later mutation of the editor's nested `selection` object will silently rewrite history. This is BAD #2 from section 1, hiding in a corner. **The rule: a memento must own its data.**

### 5. The real cost — memory, and the diff/command alternative

This is the part most candidates skip and where senior answers get made.

A full snapshot copies **all** the state, every time. Do the arithmetic on a plausible editor:

```
Document:              1 MB of text  (a ~200-page manuscript)
Snapshot per keystroke: ~1 MB (deep copy of the whole document)
Typing speed:           ~5 keystrokes/sec
Undo history:           100 steps

  Memory  = 100 snapshots × 1 MB = 100 MB   ← just for undo, of a 1 MB document
  Copy cost per keystroke = O(document size) = 1 MB memcpy, 5×/sec = 5 MB/s of churn
```

100 MB of RAM and 5 MB/s of allocation to support typing. That is unacceptable, and it's the reason no real editor does naive full-snapshot Memento per keystroke.

**The alternative: store the change, not the state.** This is the **Command pattern** (topic 43): each edit becomes an object that knows how to `execute()` and `undo()` itself.

```javascript
// COMMAND-based undo: store the DELTA, not the whole document
class InsertTextCommand {
  constructor(editor, text, at) { this.editor = editor; this.text = text; this.at = at; }
  execute() { this.editor.insertAt(this.at, this.text); }
  undo()    { this.editor.deleteRange(this.at, this.at + this.text.length); }  // inverse op
}
// Memory per undo step: ~length of the inserted text (bytes), not the whole doc (megabytes).
//   100 steps × ~10 bytes = 1 KB.  vs 100 MB.  A 100,000× difference.
```

**When does each one win?**

| | **Memento (full snapshot)** | **Command / diff (inverse operations)** |
|---|---|---|
| Memory per step | O(size of entire state) | O(size of the change) |
| Time to undo | O(1) — just swap the state in | O(1) per step, but O(k) to rewind k steps |
| Time to jump to step N | **O(1)** — restore snapshot N directly | O(N) — replay/rewind from the nearest known point |
| Correctness burden | Low — you copy everything, nothing can be missed | **High** — every command must have a correct *inverse*. Get one wrong and history corrupts. |
| Works for non-invertible ops | **Yes** (e.g. "sort", "normalize", "apply ML filter") | No — you can't invert a lossy operation without... a memento |
| Best when | State is **small**, operations are **complex or lossy**, you need random access to history | State is **large**, operations are **small and cleanly invertible** |

**The professional answer is a hybrid, and it has a name: checkpoints.** Take a full memento every N operations, and store commands/diffs in between. To reach step 743, restore the snapshot at step 700 and replay 43 commands. You get bounded memory *and* fast seeking.

```
snapshot        commands...        snapshot        commands...       snapshot
   │  c c c c c c c c c c c c c  │   c c c c c c   │  c c c c c c c    │
   0 ────────────────────────── 100 ──────────── 200 ─────────────── 300
   ▲                                                                   ▲
   restore from here...                              ...replay to here

   Memory: (total_ops / N) full snapshots + total_ops small deltas
```

This is exactly how databases do it: a **checkpoint** plus a **write-ahead log** of changes. It's how video codecs do it: a **keyframe** plus **delta frames**. It's how git does it: **object snapshots** with **packfile deltas**. Same idea, three industries.

### 6. A real Node.js ecosystem example

**Redux DevTools time-travel debugging.** Redux state is immutable, so every dispatched action produces a *new* state object — which means every state in history is, structurally, already a memento. The DevTools extension is the **caretaker**: it holds the list of past states and lets you click any one to jump there, and it knows nothing about your app's domain. The store is the **originator**. This works precisely because Redux's constraint (never mutate state) gives you free, cheap-ish snapshots via structural sharing (unchanged sub-trees are reused, not copied) — a nice illustration of how immutable data structures make Memento affordable.

Related in Node:
- **`structuredClone()`** (global since Node 17) — the standard deep-copy primitive you build mementos with.
- **Sequelize / TypeORM transactions** — `SAVEPOINT` and `ROLLBACK TO SAVEPOINT` are database-level mementos; the ORM is the caretaker.
- **Immer's `produce()`** — patches (`produceWithPatches`) give you the *diff* variant of the same idea, with `applyPatches` / `inversePatches` as the undo.

### 7. When NOT to use it / how it's abused

- **Snapshotting huge state per operation.** The 100 MB arithmetic above. If your state is large and your operations are small, use Command (43), not Memento.
- **Snapshotting things you can't clone.** Open sockets, file handles, DB connections, running timers. `structuredClone` will throw on functions and class instances with behaviour. A memento must be *data*.
- **When state is trivially reconstructible.** If the state is derived from something cheap and stable (a pure function of an input), just recompute it instead of storing it.
- **Unbounded history.** The single most common production bug with Memento: an undo stack that grows forever and OOMs a long-lived editor session. **Always cap it** (`limit` in the code above) or make it a ring buffer.
- **Leaky mementos.** If you find yourself adding `getContent()` to the memento so the caretaker can "just show a preview," you are one step from re-exposing everything. Give the memento a purpose-built `label`/`metadata` getter instead (as we did) and stop there.
- **Mutable mementos.** If you don't deep-copy on the way in and out, and don't `Object.freeze`, the "snapshot" is a live wire. History silently rewrites itself.

### 8. Related patterns and how they differ

| Pattern | How it relates to Memento |
|---|---|
| **Command (43)** | The most important comparison. Command stores *the operation and its inverse*; Memento stores *the resulting state*. Command = small memory, needs invertible ops. Memento = big memory, works for anything. **In practice, Command is the caretaker and Memento is what it saves** — a `SortCommand` that can't compute an inverse takes a memento before executing and restores it on undo. They are partners, not rivals. |
| **Prototype (33)** | Prototype's `clone()` also copies an object's state — and for simple objects, `clone()` *is* a serviceable memento. The difference is intent and encapsulation: a clone is a fully usable, public duplicate object; a memento is an opaque, read-only artefact meant only to be handed back. |
| **Iterator (44)** | Iterators sometimes use mementos to capture their traversal position so it can be resumed. |
| **State (46)** | If the originator's state is expressed as a State object, the memento can be as small as "which state was I in + its data". |
| **Observer (41)** | Often paired: the caretaker observes the originator and takes a backup on every `changed` event, instead of the originator calling `backup()` itself. |

The one-liner interviewers want: **"Memento saves *what it was*; Command saves *what you did*."**

---

## Visual / Diagram description

### Diagram 1: The three roles and who may read what

```
 ┌──────────────────────────────┐              ┌──────────────────────────────┐
 │        CARETAKER             │              │        ORIGINATOR            │
 │        (History)             │              │        (Editor)              │
 │                              │              │                              │
 │  - undoStack : Memento[]     │              │  - #content   : string       │
 │  - redoStack : Memento[]     │  save()      │  - #cursor    : number       │
 │  - limit     : number        │─────────────▶│  - #selection : {s,e}        │
 │                              │              │  - #fontSize  : number       │
 │  + backup()                  │◀─ Memento ───│                              │
 │  + undo()                    │              │  + type(text)                │
 │  + redo()                    │  restore(m)  │  + save()    : Memento       │
 │  + list()  : string[]        │─────────────▶│  + restore(m : Memento)      │
 └──────────────┬───────────────┘              └──────────────┬───────────────┘
                │                                             │
         holds  │                                             │ creates & reads
                │           ┌─────────────────────┐           │
                └──────────▶│      MEMENTO        │◀──────────┘
                            │   (EditorSnapshot)  │
                            │                     │
                            │   - #state   ◀──────┼── caretaker CANNOT touch this
                            │   - #takenAt        │      (SyntaxError, not a convention)
                            │                     │
                            │   + get label()  ◀──┼── the ONLY public surface
                            │   + _reveal(token)  │      (originator-only)
                            └─────────────────────┘

  SOLID arrow  = "can call / can read"
  The caretaker's arrow to Memento is a HOLD arrow only. It stores it. It cannot open it.
```

### Diagram 2: The undo sequence (draw this on the whiteboard)

```
 Caretaker            Originator             Memento
    │                     │                     │
    │  1. backup()        │                     │
    ├────── save() ──────▶│                     │
    │                     ├──── new(state) ────▶│   (deep copy, frozen)
    │◀───── memento ──────┤◀────────────────────┤
    │                     │                     │
   push to undoStack      │                     │
    │                     │                     │
    │            2. user types "!!!"            │
    │                     ├── #content changes  │
    │                     │                     │
    │  3. undo()          │                     │
    ├── save() (for redo)─▶                     │
    │◀── current memento ─┤                     │
   push to redoStack      │                     │
    │                     │                     │
   pop from undoStack     │                     │
    ├─ restore(memento) ─▶│                     │
    │                     ├──── _reveal() ─────▶│
    │                     │◀─── {state copy} ───┤
    │                     │  #content restored  │
    │                     │                     │
```

**What the sequence shows:** the caretaker drives *when*, the originator drives *what*. The caretaker's two lines of code (`push`, `pop`) contain zero knowledge of what an editor is. Note step 3's first move — you must snapshot the *current* state before undoing, or redo has nothing to go back to.

### Diagram 3: Snapshot vs diff memory (the trade-off, visualised)

```
FULL SNAPSHOTS (Memento)                DIFFS / COMMANDS (Command pattern)
──────────────────────────              ────────────────────────────────────
 step 1 │████████████████│ 1 MB          step 1 │▌│  10 B  (+"H")
 step 2 │████████████████│ 1 MB          step 2 │▌│  10 B  (+"e")
 step 3 │████████████████│ 1 MB          step 3 │▌│  10 B  (+"l")
  ...                                     ...
 step100│████████████████│ 1 MB          step100│▌│  10 B
        ══════════════════                      ═════
        TOTAL: 100 MB                           TOTAL: ~1 KB

 Jump to step 42:  O(1)  ✔                Jump to step 42:  replay 42 ops  ✘
 Undo a "sort":    works  ✔               Undo a "sort":    no inverse     ✘

HYBRID (what real systems do): snapshot every 100 ops + diffs in between.
        │████│ ▌▌▌▌▌▌▌▌▌▌▌▌ │████│ ▌▌▌▌▌▌▌▌▌▌▌▌ │████│
         ▲ checkpoint          ▲ checkpoint       ▲ checkpoint
```

---

## Real world examples

### 1. PostgreSQL savepoints and transaction rollback

Inside a transaction, `SAVEPOINT sp1` records a point you can return to; `ROLLBACK TO SAVEPOINT sp1` restores it. Your application is the caretaker — it names savepoints and decides when to roll back — but it has **no idea** what the engine stored (which pages, which tuple versions, which locks). Postgres implements this with MVCC row versions plus the WAL: conceptually a checkpoint-plus-log hybrid, exactly the pattern from diagram 3. The application-facing API is a pure Memento: an opaque handle you can hold and hand back.

### 2. Redux DevTools time-travel debugging

The DevTools panel shows every dispatched action and lets you click backwards through them, with the UI re-rendering to that historical state. Because Redux state is immutable, each past state object is already an opaque, frozen snapshot — a memento. The DevTools extension is a domain-agnostic caretaker: it works on *any* Redux app without knowing anything about that app's state shape. This is Memento's promised property (caretaker independent of originator internals) demonstrated at scale.

### 3. VM and container snapshots

VMware/VirtualBox "snapshot" and a Docker image layer are mementos of an entire machine. The snapshot manager UI lets you name, list, branch, and revert to snapshots; it cannot interpret the guest's memory pages. And here too the memory trade-off is the central engineering problem, solved the same way: **copy-on-write / incremental snapshots** — take one full base image, then store only the changed blocks. That is precisely "snapshot + diffs," proving out the hybrid from section 5 at the infrastructure layer.

---

## Trade-offs

| Pros | What you gain |
|---|---|
| **Encapsulation preserved** | You get undo without turning private fields into public API |
| **Caretaker is domain-agnostic** | `History` never changes when `Editor` gains a field |
| **Works for any operation** | Even non-invertible ones (sort, normalize, "apply filter") |
| **O(1) restore and O(1) random access** | Jumping to any point in history is just a state swap |
| **Simple and hard to get wrong** | Copy everything → nothing can be silently missed |

| Cons | What you pay |
|---|---|
| **Memory** | O(full state) per step. 100 steps × 1 MB doc = 100 MB. This is the dominant cost. |
| **Copy time** | Deep-copying large state on every operation adds latency and GC churn |
| **Unbounded growth** | Long sessions OOM unless you cap the history |
| **Uncloneable state** | Sockets, handles, timers, closures cannot be snapshotted |
| **Language friction** | True "only-originator-can-read" is awkward in JS (token/WeakMap tricks) |

| Situation | Memento or Command? |
|---|---|
| Small state (a game character, a form, a config) | **Memento** — simple, safe, cheap |
| Large state (a 1 MB document, a big canvas) | **Command** — store deltas |
| Operations are hard/impossible to invert | **Memento** — no inverse needed |
| You need to jump to an arbitrary point in history fast | **Memento** — O(1) seek |
| Long history over large state | **Hybrid** — checkpoint every N ops + diffs between |

**The sweet spot:** use Memento when the state is small enough that copying it is cheap, or when operations are too complex to invert reliably. **Rule of thumb: if `sizeof(state) × historyDepth` fits comfortably in memory, snapshot. If it doesn't, store commands and take a memento only as a periodic checkpoint.**

---

## Common interview questions on this topic

### Q1: "Implement undo for a text editor. Would you store snapshots or diffs?"
**Hint:** Ask about document size and history depth first — that's the whole decision. Small state → snapshots (Memento): simple, safe, O(1) restore, no inverse operations to get wrong. Large state + small edits → commands/diffs: O(change) memory. Then propose the hybrid — checkpoint snapshot every N ops with diffs in between — and mention that this is what databases (checkpoint + WAL) and video codecs (keyframe + delta frames) do. Show the arithmetic: 100 steps × 1 MB doc = 100 MB, versus ~1 KB of deltas.

### Q2: "How is Memento different from just calling `JSON.parse(JSON.stringify(obj))`?"
**Hint:** Mechanically, a deep clone *is* how you build the memento's payload — but the pattern is about **who is allowed to read it**. A plain clone is a fully readable public object; the caretaker can inspect and mutate it, which re-couples it to your internals. A memento is opaque: the caretaker can only hold it. Also, `JSON` round-tripping silently destroys `Date`, `Map`, `Set`, `undefined`, and `BigInt` — use `structuredClone()`.

### Q3: "How does the caretaker really not read the memento in JavaScript?"
**Hint:** `#private` fields are enforced by the parser — `memento.#state` outside the declaring class is a **SyntaxError**, not a lint warning. For the originator's privileged read, use either a `Symbol` access token or (better) a `WeakMap` inside the originator that maps memento → reader closure. That gives the originator access with zero public surface, and it garbage-collects with the memento.

### Q4: "Memento vs Command — when do they team up rather than compete?"
**Hint:** Command is the *caretaker* in most real undo systems, and Memento is its escape hatch. An `InsertTextCommand` implements `undo()` with an inverse operation (cheap). A `SortDocumentCommand` has no inverse — so it takes a memento in `execute()` and restores it in `undo()` (expensive but correct). One undo stack, two strategies, chosen per command. That answer usually ends the question.

### Q5: "What goes wrong with Memento in production?"
**Hint:** Three things, in order of frequency. (1) **Unbounded history** → OOM in long sessions; cap it or ring-buffer it. (2) **Shallow copy** → you stored a reference, so the "snapshot" mutates along with the live object and undo does nothing; deep-copy in *and* out, and freeze. (3) **Leaky memento** → someone adds a getter "just for the preview," and six months later the caretaker depends on five internals again.

---

## Practice exercise

### Undo for a Drawing Canvas

Build a small drawing app in Node with full undo/redo, then make it memory-efficient.

**Part A — Memento (~15 min).**
Write three classes:
- `Canvas` (**Originator**): private `#shapes` array, private `#background`, private `#zoom`. Methods `addShape(shape)`, `clear()`, `setBackground(c)`, `save()`, `restore(m)`, `render()`.
- `CanvasSnapshot` (**Memento**): a `#state` private field, frozen, deep-copied on the way in. Public surface: **only** a `get summary()` returning something like `"3 shapes, bg #fff"`.
- `HistoryManager` (**Caretaker**): `backup()`, `undo()`, `redo()`, `history()`. **Constraint: `HistoryManager` may not contain the strings `shapes`, `background`, or `zoom` anywhere in its source.** If it does, you've broken encapsulation — fix it.

Demo: add 3 shapes, change the background, `clear()`, then undo twice and confirm the shapes AND the background come back.

**Part B — prove the encapsulation (~5 min).**
Add a line to your demo that tries to read `snapshot.#state` from outside. Note that it doesn't even parse. Then try `Object.keys(snapshot)` and `JSON.stringify(snapshot)` and observe what a caretaker *can* see. Write a one-line comment on what that tells you.

**Part C — the memory problem (~15 min).**
1. Instrument it: after 1,000 `addShape()` calls each backed up, print `process.memoryUsage().heapUsed`. With a 5,000-shape canvas, how much RAM does a 1,000-deep history cost? Show the arithmetic.
2. Now implement the **Command** alternative: `AddShapeCommand { execute(); undo(); }` that stores only the shape it added. Measure again.
3. Finally, implement the **hybrid**: a memento checkpoint every 50 operations, commands in between, and an `undoTo(step)` that restores the nearest checkpoint and replays forward.
4. `clear()` is the interesting case — it has no cheap inverse. Explain in a comment why `ClearCommand` must fall back to taking a memento.

**Deliverable:** one file, a `main()` that runs all three modes and prints the memory numbers side by side.

---

## Quick reference cheat sheet

- **Intent:** capture and externalize an object's internal state so it can be restored later, **without violating encapsulation**.
- **The analogy:** a **save game**. You hold the save file; only the game can read it.
- **Three roles:** **Originator** (owns state, `save()` / `restore()`), **Memento** (opaque snapshot), **Caretaker** (holds history, never peeks).
- **The defining property:** the caretaker must be writable by someone who knows *nothing* about the originator's fields.
- **In JS, enforce it with `#private` fields** — `memento.#state` from outside is a **SyntaxError**, not a convention. Use a `WeakMap` or `Symbol` token to give the originator privileged read access.
- **Deep-copy in AND out.** `structuredClone()`. Then `Object.freeze()`. Storing a reference is the classic silent bug.
- **The cost is memory:** O(full state) per step. `1 MB doc × 100 undo steps = 100 MB`.
- **The alternative is Command (43):** store the operation + its inverse. O(change) memory, but every op needs a correct inverse.
- **Memento saves *what it was*; Command saves *what you did*.**
- **Use Memento when:** state is small, ops are non-invertible (sort/normalize), or you need O(1) random access to history.
- **Use Command when:** state is large and edits are small and cleanly invertible.
- **The professional answer is the hybrid:** checkpoint snapshot every N ops + diffs between. DB = checkpoint + WAL. Video = keyframe + delta frames. Git = objects + packfile deltas.
- **Always cap the history.** Unbounded undo stacks are the #1 production OOM with this pattern.
- **In the wild:** undo/redo in editors, DB savepoints/rollback, Redux DevTools time travel, VM/container snapshots, `git stash`.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [48 — Mediator Pattern](./48-pattern-mediator.md) — centralize communication so objects stop referencing each other |
| **Next** | [50 — Visitor Pattern](./50-pattern-visitor.md) — add new operations to a stable class hierarchy without touching it |
| **Related** | [43 — Command Pattern](./43-pattern-command.md) — the diff-based alternative to Memento for undo; the two are partners, not rivals |
| **Related** | [46 — State Pattern](./46-pattern-state.md) — when the originator's state is an object, the memento gets much smaller |
