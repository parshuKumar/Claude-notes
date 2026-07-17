# 132 — Design an In-Memory File System
## Category: LLD Case Study

---

## What is this?

An **in-memory file system** is a program that models files and folders as objects in RAM (not on a real disk) and supports the operations you use every day: create a file or directory, write and read content, list a folder, move things around, compute how much space a folder uses, and search by name.

Real-world analogy: think of a **filing cabinet**. A drawer (directory) can hold folders, and each folder can hold more folders or actual documents (files). To answer "how much paper is in this drawer?" you open it, and for every folder inside you recurse — open that folder, sum its documents, and add sub-folders the same way. That recursive "a container holds items that may themselves be containers" shape is the whole problem, and it is the textbook case for the **Composite** design pattern.

---

## Why does it matter?

This is *the* canonical Composite-pattern interview question, and a beautiful capstone for low-level design because a directory tree is the cleanest real example of a recursive object structure.

**Interview angle.** "Design a file system" is asked at Amazon, Google, and Microsoft. Interviewers use it to see whether you reach for Composite naturally — whether you can spot that `File` and `Directory` should share ONE interface so `getSize()` works uniformly on either — or whether you write two parallel code paths with `if (isDirectory)` checks everywhere. The good answer is short and recursive; the bad answer is long and branchy.

**Real-work angle.** The same tree-of-nodes shape shows up constantly: a DOM tree (elements containing elements), a UI component tree (a `Panel` containing `Button`s and more `Panel`s), an org chart (managers with reports who are also managers), a JSON document, an AST in a compiler. Learn to model it once here and you will recognize it everywhere. If you get the recursion wrong — say `getSize()` doesn't descend, or `delete()` orphans children — you leak memory or report wrong totals in production.

---

## The core idea — explained simply

### Analogy: the Russian nesting doll (matryoshka)

Open a matryoshka doll and you find… another doll. Open that one and you find another, until finally you reach the smallest solid doll that opens no further.

- The **solid innermost doll** is a **File** — a *leaf*. It has content but no children.
- Every **openable doll** is a **Directory** — a *composite*. It contains other dolls, which may themselves be solid (files) or openable (directories).
- The magic: you can ask *any* doll "how heavy are you, including everything inside you?" A solid doll just reports its own weight. An openable doll reports its own weight **plus the recursive weight of every doll inside** — and it doesn't care whether those inner dolls are solid or openable, because they all answer the same question.

That single uniform question — `getSize()` — asked on a shared interface, with the composite recursing into its children, IS the Composite pattern.

| Nesting doll world | File system world | Composite pattern role |
|---|---|---|
| Any doll | `FileSystemNode` | Component (the shared interface) |
| Solid innermost doll | `File` | Leaf (no children) |
| Openable doll | `Directory` | Composite (holds children) |
| "How heavy, all-in?" | `getSize()` | Uniform operation |
| Opening a doll to weigh its contents | recursing into `children` | The composite delegating to children |
| The dolls inside could be either kind | children are `FileSystemNode`s | Leaf and composite share a type |

The reader should burn in one sentence: **the client calls `getSize()` on a node without knowing or caring whether it's a file or a folder** — the folder just quietly recurses.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method. Composite is the star, and it appears in steps 2, 3, and 4.

### 1. Requirements & clarifying questions

**Functional requirements.**
- Model **files** and **directories**. A directory contains files AND other directories — i.e. an unbounded **tree**.
- Operations:
  - `mkdir(path)` — create a directory (and missing parents).
  - `createFile(path, content)` — create a file with content.
  - `read(path)` / `write(path, content)` — read and overwrite file content.
  - `delete(path)` — remove a file, or a directory *and its whole subtree*.
  - `move(src, dst)` — relocate a node under a new parent.
  - `ls(path)` — list a directory's immediate children.
  - `totalSize(path)` — compute the total bytes under a node (recursive).
  - `find(name)` — search the whole tree for nodes with a given name.

**Non-functional.** In-memory (no disk), single-threaded for now, correct recursion, O(1) child lookup by name.

**Clarifying questions to ask the interviewer** (asking these signals seniority):
- *Permissions?* Do we need rwx per node and per user? (Assume no for v1; it's an extension.)
- *Symlinks / hard links?* Can a node point at another node? (No for v1 — keeps the tree a true tree with no cycles.)
- *In-memory only, or persistence?* (In-memory; serialization is an extension.)
- *Path syntax?* Absolute POSIX-style `/home/user/a.txt`, `/` as separator, root is `/`? (Yes.)
- *Same name, file and dir, in one folder?* (No — a directory's children are keyed by unique name.)
- *Metadata* like created/modified timestamps? (Extension.)

We lock v1 to: absolute paths, `/` separator, no permissions, no symlinks, in-memory.

### 2. Identify core objects (nouns)

Read the requirements and underline the nouns: **file**, **directory**, **content**, **path**, **size**, **name**, **tree**, **root**.

The **key insight** that makes or breaks this design: a `File` and a `Directory` must be treated **uniformly** wherever possible, because a directory's own operations — `getSize()`, `delete()`, `getPath()` — recurse over its children, and those children may themselves be files or directories. If files and directories were unrelated types, every recursion would need a type switch. Give them a **common supertype** and the recursion becomes trivially uniform.

That is exactly **Composite**. So our objects are:

| Object | Role | Responsibility |
|---|---|---|
| `FileSystemNode` (abstract) | **Component** | Shared interface: `getName()`, `getSize()`, `getPath()`, `delete()`. Knows its `parent`. |
| `File` | **Leaf** | Holds a content string. `getSize()` = content length. No children. |
| `Directory` | **Composite** | Holds a `Map` of name → child node. `getSize()` = recursive sum. Adds/removes children. |
| `FileSystem` | **Facade** | Owns the root `Directory`; exposes path-based operations, hiding tree walking. |
| `Path` (helper) | Value/util | Splits and normalizes `/a/b/c` into segments; resolves a path to a node. |

### 3. Behaviours (verbs) + who owns them

Assign each verb to exactly one owner. Notice how the **shared** behaviours live on the base and the **container-only** behaviours live on `Directory`.

| Behaviour | Owner | Notes |
|---|---|---|
| `getName()` | `FileSystemNode` (base) | Every node has a name. |
| `getPath()` | `FileSystemNode` (base) | Walk `parent` pointers up to root, join with `/`. Uniform. |
| `getSize()` | **abstract** on base; overridden in `File` and `Directory` | **Leaf: content length. Composite: SUM of children's `getSize()` — recursive.** This is the star. |
| `delete()` | `FileSystemNode` (base) | Ask parent to remove me. For a directory this drops the whole subtree. |
| `write` / `read` / `append` | `File` only | Content operations are meaningless on a directory. |
| `add(node)` | `Directory` only | Insert a child, set `child.parent = this`. |
| `remove(name)` | `Directory` only | Delete a child by name. |
| `getChild(name)` / `list()` | `Directory` only | O(1) lookup; list children. |
| `print(indent)` | `Directory` (recursive) / `File` | Recursively render the tree. |

The recursion over a **uniform interface** (`getSize`, `getPath`, `print`) is the entire point. The directory never asks "is this child a file?" — it just calls the same method on it.

### 4. The COMPOSITE pattern — the star

Recall from **[38 — Composite](./38-pattern-composite.md)**: Composite lets you treat individual objects (leaves) and compositions of objects (composites) **uniformly** through a shared component interface, so a client can operate on a whole tree the same way it operates on a single element.

Here, both `File` (leaf) and `Directory` (composite) implement the **same** `FileSystemNode` interface. So client code can call `getSize()` or `print()` on **any** node without knowing whether it's a file or a folder. When the node is a folder, it simply recurses into its children and combines their results.

**The tree, and a `getSize()` recursion, in ASCII:**

```
                 / (Directory)
                 |
        ┌────────┴────────┐
      home (Dir)        etc (Dir)
        |                 |
      user (Dir)        hosts.txt (File, 40)
        |
   ┌────┴─────┐
 docs (Dir)  pics (Dir)
   |            |
 a.txt (File)  b.jpg (File)
   (12)          (2048)

getSize("/home") recurses:
  home  = user
  user  = docs + pics
  docs  = a.txt(12)            = 12
  pics  = b.jpg(2048)          = 2048
  user  = 12 + 2048            = 2060
  home  = 2060                 = 2060   <-- one number, computed by recursion
```

The client asked one question — `home.getSize()` — and the tree computed it by each composite delegating to its children. No `if (file) … else …` anywhere.

**The classic Composite trade-off — transparency vs. safety.** Where do `add(node)` / `remove(name)` live?

- **Transparent design** — put `add`/`remove` on the base `FileSystemNode`. *Every* node has the same full interface, so the client never type-checks. But a `File` has no children, so `File.add()` must throw or no-op — you've put a method on a type where it's meaningless (a lie in the interface).
- **Safe design** — put `add`/`remove` only on `Directory`. The type system guarantees you can only add children to something that can hold them. Cost: the client must confirm a node is a `Directory` before adding (`node instanceof Directory`).

**We pick the safe design.** Child-management belongs only on `Directory`; the base holds only the operations that are genuinely universal (`getSize`, `getPath`, `delete`, `getName`). This matches the safety-over-transparency reasoning in [38 — Composite](./38-pattern-composite.md): a `File.add()` that throws at runtime is a worse failure than an honest compile/lookup-time distinction. The uniform operations (`getSize`, `print`) stay on the shared interface — that's where the Composite payoff lives — while the container-only operations stay honest.

### 5. Class diagram + tree diagram

**Class hierarchy:**

```
                 ┌───────────────────────────────┐
                 │      FileSystemNode (abstract) │  <-- Component
                 │-------------------------------│
                 │ name: string                  │
                 │ parent: Directory | null      │
                 │-------------------------------│
                 │ getName()                     │
                 │ getPath()      (walks parent) │
                 │ getSize()      << abstract >>  │
                 │ delete()       (parent.remove)│
                 └──────────────┬────────────────┘
                                │  extends
              ┌─────────────────┴──────────────────┐
              │                                     │
   ┌──────────▼───────────┐          ┌──────────────▼───────────────┐
   │  File  (Leaf)         │          │  Directory (Composite)         │
   │----------------------│          │--------------------------------│
   │ content: string       │          │ children: Map<string,Node>     │
   │----------------------│          │--------------------------------│
   │ getSize()=len(content)│          │ getSize()=Σ child.getSize()    │
   │ read() write() append()│         │ add() remove() getChild()      │
   └──────────────────────┘          │ list() print(indent)           │
                                      └────────────────────────────────┘

   ┌──────────────────────────┐        holds a root
   │  FileSystem  (Facade)     │──────────────────────▶ Directory "/"
   │--------------------------│
   │ mkdir createFile read     │   uses ┌───────────────┐
   │ write delete move ls      │───────▶│ Path (helper) │  split / resolve
   │ find totalSize            │        └───────────────┘
   └──────────────────────────┘
```

**Example directory tree with sizes** (leaf number = content bytes):

```
/                       -> totalSize 2100
├── home                -> 2060
│   └── user            -> 2060
│       ├── docs        -> 12
│       │   └── a.txt   -> 12   "hello, world"
│       └── pics        -> 2048
│           └── b.jpg   -> 2048 (binary blob, represented as a 2048-char string)
└── etc                 -> 40
    └── hosts.txt       -> 40
```

### 6. Full JavaScript implementation

Real, complete, runnable. Save as `filesystem.js` and run `node filesystem.js`.

```js
'use strict';

// ─────────────────────────────────────────────────────────────
// Component: the shared interface for BOTH files and directories.
// This is the heart of Composite — one type, uniform operations.
// ─────────────────────────────────────────────────────────────
class FileSystemNode {
  constructor(name) {
    if (new.target === FileSystemNode) {
      throw new Error('FileSystemNode is abstract; instantiate File or Directory.');
    }
    this.name = name;
    this.parent = null; // Directory | null. Root's parent is null.
  }

  getName() {
    return this.name;
  }

  // Abstract: every concrete node MUST answer this. The whole design
  // rests on the fact that a client can call getSize() on ANY node.
  getSize() {
    throw new Error('getSize() is abstract and must be overridden.');
  }

  // Uniform for leaf AND composite: walk parent pointers up to the root,
  // then join the collected names with '/'. No type checks needed.
  getPath() {
    const segments = [];
    let node = this;
    while (node.parent !== null) { // stop before the root
      segments.unshift(node.name);
      node = node.parent;
    }
    return '/' + segments.join('/'); // root itself -> "/"
  }

  // Detach me from my parent. For a Directory this drops the whole
  // subtree (children become unreachable and get garbage-collected).
  delete() {
    if (this.parent === null) {
      throw new Error('Cannot delete the root directory.');
    }
    this.parent.remove(this.name);
  }
}

// ─────────────────────────────────────────────────────────────
// Leaf: a File holds content and has NO children.
// ─────────────────────────────────────────────────────────────
class File extends FileSystemNode {
  constructor(name, content = '') {
    super(name);
    this.content = content;
  }

  // A file's size is simply its content length. Base case of the recursion.
  getSize() {
    return this.content.length;
  }

  read() {
    return this.content;
  }

  write(content) {
    this.content = content;
    return this;
  }

  append(content) {
    this.content += content;
    return this;
  }

  // Leaf print: just the file name and size.
  print(indent = '') {
    console.log(`${indent}- ${this.name} (${this.getSize()} bytes)`);
  }
}

// ─────────────────────────────────────────────────────────────
// Composite: a Directory holds a Map of name -> child node.
// Its operations RECURSE over children — the star of the show.
// ─────────────────────────────────────────────────────────────
class Directory extends FileSystemNode {
  constructor(name) {
    super(name);
    this.children = new Map(); // name -> FileSystemNode (File OR Directory)
  }

  // Container-only operations live HERE, not on the base (safe design).
  add(node) {
    if (this.children.has(node.name)) {
      throw new Error(`'${node.name}' already exists in '${this.getPath()}'.`);
    }
    node.parent = this;          // maintain the upward pointer
    this.children.set(node.name, node);
    return node;
  }

  remove(name) {
    const node = this.children.get(name);
    if (!node) throw new Error(`No such child '${name}' in '${this.getPath()}'.`);
    node.parent = null;
    this.children.delete(name);
    return node;
  }

  getChild(name) {
    return this.children.get(name) || null; // O(1) lookup
  }

  list() {
    return [...this.children.values()];
  }

  // *** THE STAR ***: a directory's size is the SUM of its children's
  // sizes. Each child answers getSize() itself — a File returns its
  // length, a sub-Directory recurses again. Uniform, no type checks.
  getSize() {
    let total = 0;
    for (const child of this.children.values()) {
      total += child.getSize(); // recursion happens right here
    }
    return total;
  }

  // Recursively print the whole subtree, files and folders alike.
  print(indent = '') {
    console.log(`${indent}+ ${this.name}/ (${this.getSize()} bytes total)`);
    for (const child of this.children.values()) {
      child.print(indent + '  '); // uniform call: File.print or Directory.print
    }
  }
}

// ─────────────────────────────────────────────────────────────
// Path helper: turn "/home/user/a.txt" into ["home","user","a.txt"].
// ─────────────────────────────────────────────────────────────
const Path = Object.freeze({
  split(path) {
    if (typeof path !== 'string' || !path.startsWith('/')) {
      throw new Error(`Path must be absolute (start with '/'): got '${path}'`);
    }
    return path.split('/').filter(Boolean); // drop empty segments
  },
  basename(path) {
    const parts = Path.split(path);
    return parts[parts.length - 1] || null; // null for "/"
  },
  dirname(path) {
    const parts = Path.split(path);
    parts.pop();
    return '/' + parts.join('/'); // parent path
  },
});

// ─────────────────────────────────────────────────────────────
// Facade: FileSystem hides all tree-walking behind path-based ops.
// Clients never touch parent pointers or the children Map directly.
// ─────────────────────────────────────────────────────────────
class FileSystem {
  constructor() {
    this.root = new Directory(''); // root's name is '' so getPath() -> '/'
  }

  // Resolve a path to an existing node, or null. Walks from the root.
  _resolve(path) {
    if (path === '/') return this.root;
    let node = this.root;
    for (const segment of Path.split(path)) {
      if (!(node instanceof Directory)) return null; // can't descend into a file
      node = node.getChild(segment);
      if (!node) return null;
    }
    return node;
  }

  // Resolve a path to a Directory, throwing a clear error otherwise.
  _resolveDir(path) {
    const node = this._resolve(path);
    if (!node) throw new Error(`No such path: '${path}'`);
    if (!(node instanceof Directory)) throw new Error(`Not a directory: '${path}'`);
    return node;
  }

  // Create a directory, making any missing parent directories (like mkdir -p).
  mkdir(path) {
    let node = this.root;
    for (const segment of Path.split(path)) {
      let next = node.getChild(segment);
      if (!next) {
        next = node.add(new Directory(segment)); // create missing parent
      } else if (!(next instanceof Directory)) {
        throw new Error(`'${segment}' exists and is a file, not a directory.`);
      }
      node = next;
    }
    return node;
  }

  createFile(path, content = '') {
    const parent = this._resolveDir(Path.dirname(path));
    return parent.add(new File(Path.basename(path), content));
  }

  read(path) {
    const node = this._resolve(path);
    if (!(node instanceof File)) throw new Error(`Not a file: '${path}'`);
    return node.read();
  }

  write(path, content) {
    const node = this._resolve(path);
    if (!(node instanceof File)) throw new Error(`Not a file: '${path}'`);
    return node.write(content);
  }

  // Delete a file, or a directory AND its whole subtree (Composite delete()).
  delete(path) {
    const node = this._resolve(path);
    if (!node) throw new Error(`No such path: '${path}'`);
    node.delete();
  }

  // Move src under the directory dstDir, keeping the same base name.
  move(srcPath, dstDirPath) {
    const node = this._resolve(srcPath);
    if (!node) throw new Error(`No such path: '${srcPath}'`);
    const dstDir = this._resolveDir(dstDirPath);
    node.parent.remove(node.name); // detach from old parent
    dstDir.add(node);              // re-attach under new parent (parent updated)
    return node;
  }

  ls(path) {
    return this._resolveDir(path).list().map((n) => n.getName());
  }

  // Total bytes under a path — delegates to the recursive Composite getSize().
  totalSize(path) {
    const node = this._resolve(path);
    if (!node) throw new Error(`No such path: '${path}'`);
    return node.getSize();
  }

  // Recursive search: collect the paths of every node whose name matches.
  find(name, start = this.root, hits = []) {
    if (start.getName() === name && start !== this.root) {
      hits.push(start.getPath());
    }
    if (start instanceof Directory) {
      for (const child of start.list()) {
        this.find(name, child, hits); // recurse into the subtree
      }
    }
    return hits;
  }

  print() {
    this.root.print();
  }
}

// ─────────────────────────────────────────────────────────────
// Demo: build a tree, prove recursion, move, search, delete.
// ─────────────────────────────────────────────────────────────
function main() {
  const fs = new FileSystem();

  // 1. Build a small tree (mkdir -p style).
  fs.mkdir('/home/user/docs');
  fs.mkdir('/home/user/pics');
  fs.mkdir('/etc');

  fs.createFile('/home/user/docs/a.txt', 'hello, world'); // 12 bytes
  fs.createFile('/home/user/pics/b.jpg', 'x'.repeat(2048)); // 2048 bytes
  fs.createFile('/etc/hosts.txt', '127.0.0.1 localhost # simple hosts file!'); // 40

  console.log('=== Initial tree ===');
  fs.print();

  // 2. Prove the recursive Composite getSize().
  console.log('\n=== Sizes (computed by recursion) ===');
  console.log('totalSize("/home")      =', fs.totalSize('/home'));      // 2060
  console.log('totalSize("/home/user/docs") =', fs.totalSize('/home/user/docs')); // 12
  console.log('totalSize("/")          =', fs.totalSize('/'));          // 2100

  // 3. Read / write / append file content.
  console.log('\n=== Read / write ===');
  console.log('read a.txt:', JSON.stringify(fs.read('/home/user/docs/a.txt')));
  fs.write('/home/user/docs/a.txt', 'updated content here');
  console.log('after write, size of /home/user/docs =', fs.totalSize('/home/user/docs'));

  // 4. Move a file to a new directory (path resolution + re-parenting).
  console.log('\n=== Move a.txt -> /etc ===');
  fs.move('/home/user/docs/a.txt', '/etc');
  console.log('ls /etc:', fs.ls('/etc'));
  console.log('new path of a.txt:', fs._resolve('/etc/a.txt').getPath());

  // 5. Recursive search by name.
  console.log('\n=== find("b.jpg") ===');
  console.log(fs.find('b.jpg')); // ['/home/user/pics/b.jpg']

  // 6. Delete a directory — Composite delete drops the whole subtree.
  console.log('\n=== delete /home/user/pics (recursive) ===');
  fs.delete('/home/user/pics');
  console.log('ls /home/user:', fs.ls('/home/user')); // pics gone

  console.log('\n=== Final tree ===');
  fs.print();
}

main();

module.exports = { FileSystemNode, File, Directory, FileSystem, Path };
```

**Expected output (abridged):**

```
=== Initial tree ===
+ / (2100 bytes total)
  + home/ (2060 bytes total)
    + user/ (2060 bytes total)
      + docs/ (12 bytes total)
        - a.txt (12 bytes)
      + pics/ (2048 bytes total)
        - b.jpg (2048 bytes)
  + etc/ (40 bytes total)
    - hosts.txt (40 bytes)

=== Sizes (computed by recursion) ===
totalSize("/home")      = 2060
...
=== find("b.jpg") ===
[ '/home/user/pics/b.jpg' ]
```

Every number in that tree — `/` is 2100, `/home` is 2060 — was produced by the same recursive `getSize()`, walking down through composites until it hit files.

### 7. Design patterns used and WHY

| Pattern | Where | Why |
|---|---|---|
| **Composite** (the star) | `FileSystemNode` ← `File` / `Directory` | Treat leaf files and composite directories **uniformly** so recursive operations (`getSize`, `getPath`, `print`, `delete`) work on any node with no type switching. This is [38 — Composite](./38-pattern-composite.md) in its purest form. |
| **Facade** | `FileSystem` | Hides tree traversal and parent-pointer bookkeeping behind simple path-based operations (`mkdir`, `read`, `move`). The client says `fs.read('/etc/hosts.txt')`, not "walk from root, get child etc, get child hosts.txt, read". |
| **Visitor** (worth mentioning) | not built in v1 | See below — the clean way to add NEW operations over the tree without touching the node classes. |

**Visitor — recall [50 — Visitor](./50-pattern-visitor.md).** Composite is great when the set of *operations* is stable but the node types recurse. But when you keep adding *new* operations — "count all `.txt` files", "list files larger than 1 KB", "compute total size", "export to JSON" — putting each one as a new method on `File` and `Directory` makes those classes swell and mixes unrelated concerns. Visitor externalizes the operation: you write one `SizeVisitor` or `TxtCountVisitor` object with a `visitFile(file)` and `visitDirectory(dir)`, and each node has a single `accept(visitor)` method. New operation = new visitor class; the node classes never change.

**The honest trade-off** (from [50 — Visitor](./50-pattern-visitor.md)): Visitor makes adding *operations* cheap but adding a *new node type* expensive (every existing visitor must gain a new `visitX`). Composite-with-methods is the reverse. For a file system the node types (`File`, `Directory`) are fixed forever but operations may grow, so **Visitor is the right tool the moment you have three or more tree-walking operations** — for one or two, a plain recursive method is simpler and we kept it inline.

---

## Visual / Diagram description

A **sequence diagram** for `fs.totalSize('/home')`, showing the recursion bottoming out at files:

```
Client         FileSystem      home:Dir      user:Dir     docs:Dir   a.txt:File
  │  totalSize("/home")│           │             │            │          │
  ├───────────────────▶│           │             │            │          │
  │                    │ _resolve  │             │            │          │
  │                    │ getSize() │             │            │          │
  │                    ├──────────▶│             │            │          │
  │                    │           │ getSize()   │            │          │
  │                    │           ├────────────▶│            │          │
  │                    │           │             │ getSize()  │          │
  │                    │           │             ├───────────▶│          │
  │                    │           │             │            │ getSize()│
  │                    │           │             │            ├─────────▶│
  │                    │           │             │            │◀──── 12 ─┤
  │                    │           │             │◀─── 12 ────┤          │
  │                    │           │◀── 2060 ────┤ (docs 12 + pics 2048) │
  │                    │◀── 2060 ──┤             │            │          │
  │◀──────── 2060 ─────┤           │             │            │          │
```

Each composite forwards `getSize()` to its children and sums the answers. The call stack goes as deep as the tree, and every leaf returns a concrete number that bubbles back up. That upward flow of summed numbers is what "recursive Composite" looks like on a whiteboard — draw it and you've explained the whole design.

---

## Real world examples

### Unix / Linux VFS (Virtual File System)

The Linux kernel's VFS models everything as an **inode**, and directory inodes hold entries pointing to child inodes — files or more directories — forming exactly this tree. `du -sh /home` recurses the tree summing block usage, the real-world twin of our recursive `getSize()`. (Representative of the concept; the kernel adds blocks, hard links, and mount points.)

### The browser DOM

An HTML page is a Composite tree: a `<div>` (composite) contains `<span>`s (which may be leaves) and more `<div>`s. `element.textContent` and layout/reflow recurse the subtree exactly like `getSize()` and `print()` here. React's virtual DOM and every UI toolkit's component tree share this shape. (Representative.)

### Amazon S3 "folders"

S3 has no real directories — object keys like `photos/2024/a.jpg` are flat strings — but the console *presents* a Composite tree by splitting keys on `/`. Our `Path` helper does the same split-and-walk. It's a nice reminder that the tree can be a presentation over flat storage. (Representative/conceptual.)

---

## Trade-offs

**Composite: transparency vs. safety**

| Design | Pros | Cons |
|---|---|---|
| Transparent (`add`/`remove` on base) | Client never type-checks; fully uniform interface | `File.add()` is meaningless and must throw at runtime — a lie in the type |
| Safe (`add`/`remove` on `Directory` only) — **our choice** | Type system prevents adding children to a file; honest interface | Client must confirm a node is a `Directory` before adding |

**Adding operations: recursive method vs. Visitor**

| Approach | Pros | Cons |
|---|---|---|
| Method on the nodes (our `getSize`, `print`) | Simple; the operation lives with the data | Node classes grow with every new operation |
| Visitor ([50](./50-pattern-visitor.md)) | New operation = new class; nodes untouched | Adding a new *node type* touches every visitor; more moving parts |

**Storage of children: Map vs. Array**

| Choice | Pros | Cons |
|---|---|---|
| `Map<name, node>` — **our choice** | O(1) lookup, rename, uniqueness enforced | No inherent ordering (insertion order in JS Maps is fine) |
| `Array<node>` | Ordered; simple | O(n) lookup by name |

**The sweet spot:** model the tree with a shared `FileSystemNode` component, keep child-management on `Directory` (safety), store children in a `Map` for O(1) access, put uniform recursive operations on the nodes, and reach for a **Visitor** once you have three-plus tree-walking operations.

---

## Common interview questions on this topic

### Q1: "Why is this the classic Composite example?"
**Hint:** Because a directory contains items that are *either* files or more directories, and its operations (size, delete, print) must recurse over a mix of both. Giving files and directories a common `FileSystemNode` supertype lets those operations be written once, uniformly, with no `if (isDirectory)` branching. That "uniform treatment of leaf and composite" IS Composite.

### Q2: "Should `add()` and `remove()` be on the base class or only on Directory?"
**Hint:** The transparency-vs-safety trade-off. On the base = fully uniform interface but `File.add()` must throw (dishonest). On `Directory` only = type-safe but the client must check the type before adding. State both, then pick safety and justify it — a runtime throw is a worse failure than a lookup-time distinction.

### Q3: "How does `delete()` on a directory avoid leaking its children?"
**Hint:** `delete()` just asks the parent to `remove` this node from its children Map. Once the subtree root is unreachable from the root directory, the entire subtree becomes garbage and JS's GC reclaims it — no manual recursion needed. (In a language without GC you'd recurse and free explicitly.)

### Q4: "Now add a 'count all .txt files' operation without touching File/Directory. How?"
**Hint:** Visitor ([50](./50-pattern-visitor.md)). Add one `accept(visitor)` method to each node, then write a `TxtCountVisitor` with `visitFile`/`visitDirectory`. The node classes stay closed; new operations are new visitor classes. Mention the trade-off: adding a new node type would then touch every visitor.

### Q5: "How do you resolve `/home/user/a.txt` to a node?"
**Hint:** Split the path on `/` into segments, start at the root Directory, and for each segment call `getChild(segment)`, descending one level per segment. Return null if any segment is missing or if you try to descend into a File. That's the `_resolve` method — a walk of exactly path-depth steps.

---

## Practice exercise

### Build a `SizeVisitor` and a `find`-by-extension

Starting from the implementation above (copy it into `filesystem.js`):

1. Add an `accept(visitor)` method to both `File` and `Directory`. In `File.accept`, call `visitor.visitFile(this)`; in `Directory.accept`, call `visitor.visitDirectory(this)` and then recurse: `for (const c of this.list()) c.accept(visitor)`.
2. Write a `SizeVisitor` class with `visitFile(file)` that accumulates `file.getSize()` into `this.total`, and a no-op `visitDirectory(dir)`. Run it over `/home` and confirm you get **2060** — the same number the recursive `getSize()` produced.
3. Write a `FindByExtVisitor` that collects the paths of every file whose name ends in a given extension (e.g. `.jpg`). Run it over `/` and print the matches.
4. Write two sentences: which is cleaner for *this* operation — the method or the visitor — and why. Note what would break if you added a new node type (a `Symlink`).

**Produce:** the updated `filesystem.js` plus your two-sentence trade-off note. Target ~30 minutes. This makes the Composite-vs-Visitor trade-off concrete instead of theoretical.

---

## Quick reference cheat sheet

- **Composite pattern** = treat a single object (leaf) and a group of objects (composite) through one shared interface so operations recurse uniformly.
- **Component** = `FileSystemNode` (the abstract base with `getSize`, `getPath`, `delete`).
- **Leaf** = `File` — has content, no children, `getSize()` = content length.
- **Composite** = `Directory` — has a `Map` of children, `getSize()` = **recursive sum**.
- **The star operation:** `Directory.getSize()` sums `child.getSize()` for every child — files return length, sub-dirs recurse.
- **Transparency vs. safety:** child-management on the base (uniform but `File.add` lies) vs. on `Directory` only (type-safe, our choice).
- **Facade** = `FileSystem` — path-based ops (`mkdir`, `read`, `move`) hiding the tree walk.
- **Path resolution** = split on `/`, walk from root calling `getChild` per segment.
- **`delete()` a directory** = detach from parent; the whole unreachable subtree is GC'd.
- **`getPath()`** = walk `parent` pointers up to root, join names with `/` — uniform for file and dir.
- **Store children in a `Map`** = O(1) lookup and enforced unique names.
- **Visitor ([50])** = add NEW tree operations without editing node classes; costly when node types change.
- **Same shape appears in:** DOM trees, UI component trees, org charts, ASTs, JSON.
- **Rule of thumb:** if a container holds items that may themselves be containers, reach for Composite.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [38 — Composite pattern](./38-pattern-composite.md) | The pattern this whole case study applies — leaf/composite uniformity and the safety-vs-transparency trade-off. |
| **Next** | [50 — Visitor pattern](./50-pattern-visitor.md) | The clean way to add new operations over the file tree without modifying `File`/`Directory`. |
| **Related** | [111 — LLD approach framework](./111-lld-approach-framework.md) | The nouns→verbs→relationships→patterns→code method this doc follows step by step. |
| **Related** | [13 — OOP four pillars](./13-oop-four-pillars.md) | Inheritance and polymorphism are what let `getSize()` dispatch uniformly across `File` and `Directory`. |
| **Related** | [23 — Class relationships](./23-class-relationships.md) | The composition (`Directory` *has-a* Map of nodes) and inheritance (`File`/`Directory` *is-a* node) at the core of the tree. |

---

*The end of the LLD journey.* Across these case studies you have run the same loop every time: pull the **nouns** out of the requirements to find your objects; pull the **verbs** to find your behaviours and decide who owns each; map the **relationships** (is-a vs. has-a) between them; recognize the **pattern** the shape is begging for — Strategy, State, Observer, Factory, and here Composite; and only then write the **code**. A file system distilled that loop to its cleanest form: one recursive interface, a leaf, a composite, and a facade. Master that loop and no "design X" question is unfamiliar — it is always nouns, verbs, relationships, patterns, code.
