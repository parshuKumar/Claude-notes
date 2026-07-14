# 38 — Composite Pattern

## Category: LLD Patterns

---

## What is this?

The **Composite** pattern lets you treat **a single object** and **a group of objects** in exactly the same way, by giving both the same interface. Groups can contain other groups, so the whole thing naturally forms a **tree**.

The everyday example is on the machine you're reading this on. A **file** has a size. A **folder** has a size too — it's just the sum of everything inside it, which may include more folders. When you right-click either one and hit "Get Info," you don't think "wait, is this a file or a folder?" You just ask for the size. That indifference is the entire pattern.

---

## Why does it matter?

Whenever you have a **part-whole hierarchy** — something made of things, which are made of things — code that doesn't use Composite ends up riddled with the same ugly question, over and over:

```javascript
if (node.type === 'file') { total += node.size; }
else if (node.type === 'folder') { for (const c of node.children) { /* ...and now recurse, re-doing this check */ } }
```

That `if` metastasises. It shows up in `getSize()`, in `search()`, in `delete()`, in `render()`, in `export()`. Add a third node type (a symlink, a shortcut, a new UI widget) and you have to hunt down every branch. This is the exact code smell that Composite kills.

**At real work you use it constantly:** the DOM is a composite tree. React's component tree is a composite tree. An AST (which every compiler, bundler, linter and formatter walks) is a composite tree. Nested menus, org charts, product bundles, permission groups, invoice line items with sub-items, graphics scene graphs — all composites.

**In interviews:** Composite is the backbone of a classic LLD question — *"Design an in-memory file system"* — and it shows up inside many others (design a menu system, design a UI framework, design a discount engine over nested carts). Interviewers also love the "safety vs transparency" trade-off, because it's a genuine design tension with no free answer.

---

## The core idea — explained simply

### The Company Org Chart Analogy

You're the CEO and you need to know the **total headcount cost** of the company.

The naive way: get a list of every employee, sum their salaries. Fine — but the company isn't a list. It's a tree. There's you, and under you three VPs, and under each VP some directors, and under each director some managers, and under each manager some individual contributors. And some managers manage other managers.

Now imagine you ask each person one question: **"What does your organisation cost?"**

- An **individual contributor** (a leaf) answers: *"My salary. 120k."* Simple. She has nobody under her.
- A **manager** (a composite) answers: *"My salary, plus whatever my reports tell me."* So she turns around and asks the same question of each of her reports — and each of them either answers directly (if they're an IC) or asks their own reports.

The question **never changes**. `getCost()`. Every single person, regardless of rank, can answer it. The ICs answer directly; the managers answer by delegating downward and adding themselves. The CEO asks exactly **three** people (the VPs) and gets the total for a 10,000-person company, without ever writing a loop over 10,000 employees.

That's Composite. The manager and the IC are **interchangeable** from the caller's point of view because they share the same interface. The tree does the work; the recursion falls out for free.

| In the analogy | In code |
|---|---|
| "What does your org cost?" | **Component** — the shared interface (`getSize()`, `getCost()`, `render()`) |
| The individual contributor | **Leaf** — has no children; implements the operation directly |
| The manager / VP | **Composite** — holds children; implements the operation by *delegating to children and combining* |
| The CEO asking three VPs | **Client** — talks only to the Component interface, never checks "is this a leaf?" |
| The org chart itself | The **tree** structure |
| A manager also having a salary | A composite can contribute its **own** value on top of its children's |

The payoff line, and you should be able to say it in an interview verbatim: **"The client code stops caring whether it's holding one thing or a thousand things."**

---

## Key concepts inside this topic

### 1. The problem it solves — the code before the pattern

Here's a file system without Composite. Everything is a plain object with a `type` tag.

```javascript
// BAD: type tags + the client doing the branching
const tree = {
  type: 'folder', name: 'project',
  children: [
    { type: 'file', name: 'index.js', size: 2048 },
    { type: 'folder', name: 'src', children: [
      { type: 'file', name: 'app.js', size: 8192 },
      { type: 'file', name: 'util.js', size: 1024 },
    ]},
  ],
};

// The client has to know the shape of EVERY node type.
function getSize(node) {
  if (node.type === 'file') {
    return node.size;
  } else if (node.type === 'folder') {
    let total = 0;
    for (const child of node.children) total += getSize(child); // recursion + branching, tangled
    return total;
  }
  throw new Error(`Unknown node type: ${node.type}`);
}

// And now write print(). And search(). And countFiles(). And chmod().
// Each one repeats the SAME if/else. Add 'symlink' → edit all of them.
function print(node, indent = 0) {
  if (node.type === 'file') { console.log(' '.repeat(indent) + node.name); }
  else if (node.type === 'folder') {
    console.log(' '.repeat(indent) + node.name + '/');
    for (const c of node.children) print(c, indent + 2);
  }
}
```

The problems, named:
- **The branching is in the client, not the data.** Every new operation re-implements the same `if (type === ...)` dispatch.
- **It doesn't extend.** Adding a `Symlink` means editing `getSize`, `print`, `search`, and every other traversal — and the compiler (or, in JS, nothing at all) will not remind you.
- **Leaves and groups are not interchangeable.** You can't hand a single file to a function that expects "a thing with a size" without special-casing it.

Composite fixes all three by **moving the branching into the type system**: the leaf and the composite each know how to answer for themselves, and polymorphism does the dispatch.

### 2. The structure — who plays which role

```
                 ┌───────────────────────────────┐
                 │        «abstract»              │
                 │       FileSystemNode           │   ← the COMPONENT
                 │  - name                        │
                 │  + getName()                   │
                 │  + getSize()      {abstract}   │   the shared operation
                 │  + print(indent)  {abstract}   │
                 └───────────────┬───────────────┘
                                 │ extends
              ┌──────────────────┴──────────────────┐
              │                                     │
   ┌──────────▼──────────┐            ┌─────────────▼──────────────┐
   │        File          │            │        Directory            │
   │       (LEAF)         │            │       (COMPOSITE)           │
   │  - sizeBytes         │            │  - children: Node[]  ◆──────┼──┐
   │                      │            │                             │  │
   │  + getSize()         │            │  + add(node)                │  │ 0..*
   │      → sizeBytes     │            │  + remove(name)             │  │
   │  + print(indent)     │            │  + getSize()                │  │
   │                      │            │      → Σ child.getSize()    │◀─┘
   └──────────────────────┘            │  + print(indent)            │  self-reference:
                                       └─────────────────────────────┘  THIS is what
                                                                        makes it a TREE
```

| Participant | Job |
|---|---|
| **Component** (`FileSystemNode`) | Declares the operations common to leaves and composites. This is the *only* thing the client knows about. |
| **Leaf** (`File`) | The end of a branch. Has no children. Implements the operation with real work. |
| **Composite** (`Directory`) | Holds a list of `Component` children (leaves *or* other composites), implements the operation by delegating to each child and combining results. Also provides `add`/`remove`. |
| **Client** | Calls `getSize()` on whatever it's handed. Never asks "which one are you?" |

The single most important arrow in that diagram is the one looping back from `Directory` to `FileSystemNode`: **a composite holds Components, not Leaves.** That's what allows nesting to arbitrary depth, and it's the whole reason the structure is a tree.

### 3. Full JavaScript implementation

```javascript
// ============================================================================
// COMPONENT — the shared interface. Leaf and Composite BOTH satisfy this.
// In JS we use an abstract base class; methods that must be overridden throw.
// ============================================================================
class FileSystemNode {
  constructor(name) {
    if (new.target === FileSystemNode) {
      throw new TypeError('FileSystemNode is abstract — instantiate File or Directory');
    }
    this.name = name;
    this.parent = null; // set by Directory.add — lets us walk UP for getPath()
  }

  getName() { return this.name; }

  // --- The operations every node must answer. This IS the contract. ---
  getSize()             { throw new Error('getSize() not implemented'); }
  print(indent = 0)     { throw new Error('print() not implemented'); }
  find(predicate)       { throw new Error('find() not implemented'); }

  // Absolute path — identical logic for leaf and composite, so it lives here.
  getPath() {
    const segments = [];
    for (let node = this; node !== null; node = node.parent) segments.unshift(node.name);
    return '/' + segments.join('/');
  }

  // "Safe composite" hook: the client can ask instead of type-checking.
  // See section 7 — this is the pragmatic middle of the safety/transparency debate.
  isComposite() { return false; }
}

// ============================================================================
// LEAF — a File. No children. Answers getSize() directly with real data.
// ============================================================================
class File extends FileSystemNode {
  constructor(name, sizeBytes) {
    super(name);
    if (!Number.isInteger(sizeBytes) || sizeBytes < 0) {
      throw new RangeError(`File "${name}": size must be a non-negative integer`);
    }
    this.sizeBytes = sizeBytes;
  }

  // BASE CASE of the recursion. Every recursive tree walk must bottom out here.
  getSize() { return this.sizeBytes; }

  print(indent = 0) {
    console.log(`${' '.repeat(indent)}${this.name}  (${formatBytes(this.sizeBytes)})`);
  }

  // A leaf "matches or it doesn't" — it has nowhere else to look.
  find(predicate) { return predicate(this) ? [this] : []; }
}

// ============================================================================
// COMPOSITE — a Directory. Holds children (Files OR other Directories),
// and implements every operation by delegating downward and combining.
// ============================================================================
class Directory extends FileSystemNode {
  constructor(name) {
    super(name);
    this.children = new Map(); // name -> FileSystemNode (Map gives O(1) lookup by name)
  }

  isComposite() { return true; }

  // --- Child management: the methods that ONLY exist on the composite ---
  add(node) {
    if (!(node instanceof FileSystemNode)) {
      throw new TypeError('Can only add FileSystemNode instances');
    }
    if (node === this) throw new Error('A directory cannot contain itself');
    if (this.children.has(node.name)) {
      throw new Error(`"${node.name}" already exists in ${this.getPath()}`);
    }
    node.parent = this;      // maintain the back-pointer so getPath() works
    this.children.set(node.name, node);
    return this;             // chainable: dir.add(a).add(b)
  }

  remove(name) {
    const node = this.children.get(name);
    if (!node) return false;
    node.parent = null;
    this.children.delete(name);
    return true;
  }

  getChild(name) { return this.children.get(name) ?? null; }

  // --- The composite operations: RECURSION LIVES HERE, and nowhere else ---

  // A directory's size is the sum of its children's sizes.
  // Note it does NOT ask "is this child a file or a folder?" — it just asks getSize().
  // THAT is the pattern. The child figures out its own answer.
  getSize() {
    let total = 0;
    for (const child of this.children.values()) total += child.getSize();
    return total;
  }

  print(indent = 0) {
    console.log(`${' '.repeat(indent)}${this.name}/  (${formatBytes(this.getSize())})`);
    // Directories first, then files, each alphabetically — like `ls` does.
    const sorted = [...this.children.values()].sort((a, b) =>
      a.isComposite() === b.isComposite()
        ? a.name.localeCompare(b.name)
        : (a.isComposite() ? -1 : 1)
    );
    for (const child of sorted) child.print(indent + 2); // recurse, deeper indent
  }

  find(predicate) {
    const hits = predicate(this) ? [this] : [];
    for (const child of this.children.values()) hits.push(...child.find(predicate));
    return hits;
  }

  // Bonus: make the tree iterable with for...of (depth-first).
  // Recall from [44 — Iterator] that this decouples traversal from the structure.
  *[Symbol.iterator]() {
    for (const child of this.children.values()) {
      yield child;
      if (child.isComposite()) yield* child; // recursive generator delegation
    }
  }
}

function formatBytes(n) {
  if (n < 1024) return `${n} B`;
  if (n < 1024 ** 2) return `${(n / 1024).toFixed(1)} KB`;
  return `${(n / 1024 ** 2).toFixed(2)} MB`;
}

// ============================================================================
// DEMO — run with `node composite.js`
// ============================================================================
function main() {
  const root = new Directory('project');

  const src = new Directory('src');
  src.add(new File('app.js', 8_192))
     .add(new File('router.js', 4_096));

  const components = new Directory('components');
  components.add(new File('Button.jsx', 2_048))
            .add(new File('Modal.jsx', 6_144));
  src.add(components); // a Directory inside a Directory — the tree deepens

  const assets = new Directory('assets');
  assets.add(new File('logo.png', 512_000));

  root.add(src)
      .add(assets)
      .add(new File('package.json', 1_024))
      .add(new File('README.md', 3_072));

  console.log('--- print() ---');
  root.print();

  console.log('\n--- THE PAYOFF: the client does not branch ---');
  // `nodes` mixes a Directory and a File. The loop cannot tell, and does not care.
  const nodes = [root, src, new File('standalone.txt', 999), components];
  for (const node of nodes) {
    console.log(`${node.getPath().padEnd(30)} → ${formatBytes(node.getSize())}`);
  }

  console.log('\n--- find(): recursive search, one line of client code ---');
  const jsFiles = root.find(n => !n.isComposite() && n.name.endsWith('.js'));
  console.log(jsFiles.map(f => f.getPath()));

  console.log('\n--- for...of over the whole tree (Iterator) ---');
  for (const node of root) console.log('  ', node.getPath());

  console.log('\n--- remove() ---');
  root.remove('assets');
  console.log('total after removing assets/:', formatBytes(root.getSize()));
}

main();
```

Output:

```
--- print() ---
project/  (536.58 KB)
  assets/  (500.0 KB)
    logo.png  (500.0 KB)
  src/  (19.9 KB)
    components/  (8.0 KB)
      Button.jsx  (2.0 KB)
      Modal.jsx  (6.0 KB)
    app.js  (8.0 KB)
    router.js  (4.0 KB)
  README.md  (3.0 KB)
  package.json  (1.0 KB)

--- THE PAYOFF: the client does not branch ---
/project                       → 536.58 KB
/project/src                   → 19.9 KB
/standalone.txt                → 999 B
/project/src/components        → 8.0 KB
```

Look hard at that payoff loop. It calls `getSize()` on a root directory, a nested directory, and a lone file. **There is no `if`.** That's the entire return on the pattern.

### 4. Second example — a nested menu (the same shape, different domain)

Composite isn't a file-system trick; it's a *shape*. Here's a restaurant menu where a menu can contain items or other menus (a "Drinks" section inside "Dinner").

```javascript
class MenuComponent {
  getName()  { throw new Error('not implemented'); }
  getPrice() { throw new Error('not implemented'); }   // for a menu: the cheapest item? a combo total?
  render(indent = 0) { throw new Error('not implemented'); }
}

class MenuItem extends MenuComponent {   // LEAF
  constructor(name, price, vegetarian = false) {
    super();
    Object.assign(this, { name, price, vegetarian });
  }
  getName()  { return this.name; }
  getPrice() { return this.price; }
  render(indent = 0) {
    console.log(`${' '.repeat(indent)}${this.name}${this.vegetarian ? ' (v)' : ''} — $${this.price.toFixed(2)}`);
  }
}

class Menu extends MenuComponent {       // COMPOSITE
  constructor(name) { super(); this.name = name; this.items = []; }
  add(item) { this.items.push(item); return this; }
  remove(item) { this.items = this.items.filter(i => i !== item); return this; }
  getName()  { return this.name; }
  // A section's "price" = the total if you ordered everything in it. Same recursion.
  getPrice() { return this.items.reduce((sum, i) => sum + i.getPrice(), 0); }
  render(indent = 0) {
    console.log(`${' '.repeat(indent)}== ${this.name.toUpperCase()} ==`);
    for (const item of this.items) item.render(indent + 2);
  }
}

const dinner = new Menu('Dinner');
const drinks = new Menu('Drinks');
drinks.add(new MenuItem('Lassi', 4.00, true)).add(new MenuItem('Espresso', 3.50, true));
dinner.add(new MenuItem('Paneer Tikka', 14.00, true))
      .add(new MenuItem('Lamb Rogan Josh', 19.00))
      .add(drinks);                       // a Menu nested inside a Menu

dinner.render();
console.log('order-everything total: $' + dinner.getPrice().toFixed(2)); // 40.50
```

Same three roles, same recursion, zero new ideas — which is exactly why patterns are worth learning once.

### 5. Real Node.js ecosystem example — the AST and the DOM

Every JS tool you use walks a composite tree.

- **Babel / ESLint / Prettier** operate on an **AST** (Abstract Syntax Tree). A `NumericLiteral` node is a leaf. A `BinaryExpression` node is a composite — it has `.left` and `.right` children, each of which is itself any expression node. Evaluating, printing, or linting an expression is a recursive walk that never asks "what concrete class is this?" beyond dispatching on `node.type`.

```javascript
// A tiny expression AST — Composite in ~20 lines.
class Expr { evaluate(env) { throw new Error('abstract'); } toString() { throw new Error('abstract'); } }

class Num extends Expr {                      // LEAF
  constructor(v) { super(); this.v = v; }
  evaluate() { return this.v; }
  toString() { return String(this.v); }
}
class Variable extends Expr {                 // LEAF
  constructor(name) { super(); this.name = name; }
  evaluate(env) { return env[this.name]; }
  toString() { return this.name; }
}
class BinaryOp extends Expr {                 // COMPOSITE — children are Exprs
  constructor(op, left, right) { super(); Object.assign(this, { op, left, right }); }
  evaluate(env) {
    const [l, r] = [this.left.evaluate(env), this.right.evaluate(env)]; // recurse
    switch (this.op) {
      case '+': return l + r;
      case '*': return l * r;
      case '-': return l - r;
      default: throw new Error(`Unknown op ${this.op}`);
    }
  }
  toString() { return `(${this.left} ${this.op} ${this.right})`; }
}

// (x + 2) * 3
const ast = new BinaryOp('*', new BinaryOp('+', new Variable('x'), new Num(2)), new Num(3));
console.log(ast.toString());        // ((x + 2) * 3)
console.log(ast.evaluate({ x: 5 })); // 21
```

- **The DOM.** `Node` is the component. `Text` and `Comment` are leaves. `Element` is the composite — it has `appendChild`, `removeChild`, `childNodes`. `element.textContent` on a `<div>` recursively concatenates the text of every descendant. Notice that the DOM chose **transparency**: `Node` itself declares `appendChild`, so even a `Text` node has the method — calling it just throws a `HierarchyRequestError`. That's the trade-off from section 7, made by a real spec.

- **React.** A component tree is a composite: a host element (`<div>`) is a composite whose children are more elements; a string or number child is a leaf. Reconciliation is a recursive walk over that tree. When you write `<Layout><Sidebar/><Main/></Layout>`, you're hand-writing a composite.

### 6. When NOT to use it / how it's abused

- **Your hierarchy isn't a hierarchy.** If nesting is one level deep and will stay that way, a plain array is clearer than a class hierarchy. Composite earns its keep at *arbitrary* depth.
- **Leaves and composites genuinely differ.** If half your operations are "only makes sense on a folder" and the other half "only makes sense on a file", the shared interface is a lie and you'll be `instanceof`-checking anyway. That's a signal the abstraction is wrong.
- **Cycles.** Composite assumes a **tree** (each node has ≤1 parent, no cycles). Add a hard link or a circular reference and `getSize()` becomes an infinite loop. If you need a graph, you need a visited-set and, honestly, a different pattern.
- **Deep trees + recursion = stack overflow.** Node's default stack handles roughly 10k–15k frames. A pathological 50k-deep tree will blow it. Convert to an explicit-stack iterative walk if depth is unbounded.
- **The over-eager abstraction.** Turning a two-element config object into a Component/Leaf/Composite hierarchy because "it's a tree, technically" is how you get 200 lines where 10 would do.
- **Repeated `getSize()` on a huge tree is O(n) every time.** If you call it in a render loop, memoize it on the composite and invalidate up the parent chain on `add`/`remove`.

### 7. The trade-off at the heart of Composite: safety vs transparency

Here's the tension. `add(child)` and `remove(name)` make sense on a `Directory`. They make **no sense at all** on a `File`. So where do you declare them?

**Option A — Transparency (put `add`/`remove` on the Component):**

```javascript
class FileSystemNode {
  getSize() { throw new Error('abstract'); }
  add(node)    { throw new Error(`Cannot add to a ${this.constructor.name}`); }  // leaves throw
  remove(name) { throw new Error(`Cannot remove from a ${this.constructor.name}`); }
}
```
The client can treat **every** node identically — that's maximal transparency, and it's what the Gang of Four originally recommended and what the DOM actually does. The cost: the interface is **bloated with methods that are lies for half its implementers**, and `file.add(x)` compiles/parses fine and only explodes at runtime.

**Option B — Safety (put `add`/`remove` only on the Composite):**

```javascript
class FileSystemNode { getSize() { throw new Error('abstract'); } }   // clean, honest interface
class File extends FileSystemNode { /* no add/remove — it literally cannot be misused */ }
class Directory extends FileSystemNode { add(n) {...} remove(n) {...} }
```
Now `file.add(x)` is a `TypeError: file.add is not a function` — and in TypeScript it's a **compile error**. The interface is honest. The cost: any client that wants to *build* the tree must now know whether it's holding a composite, which means an `isComposite()` check or an `instanceof` — the very branching you adopted Composite to eliminate.

| | Transparency (add/remove on Component) | Safety (add/remove on Composite only) |
|---|---|---|
| **Client uniformity** | Total — never checks the type | Partial — must check before adding |
| **Interface honesty** | Poor — `File.add()` is nonsense | Excellent |
| **Error caught** | At runtime | At compile time (TS) / immediately in JS |
| **Who does this** | The DOM (`Node.appendChild`), the original GoF book | Most modern codebases, TypeScript-first designs |

**The practical resolution** — and this is what the implementation above does — is: **keep the *read* operations (`getSize`, `print`, `find`) fully transparent on the Component, and keep the *mutation* operations (`add`, `remove`) only on the Composite.** Then add a cheap `isComposite()` so the rare client that needs to build the tree can ask politely instead of using `instanceof`.

That works because of an asymmetry worth noticing: **traversal is the common, hot path and it must be branch-free; construction is rare and usually knows exactly what it's building.** Nobody writes `for (const node of tree) node.add(x)`. They *do* write `for (const node of tree) total += node.getSize()`.

### 8. Related patterns and how they differ

| Pattern | Relationship to Composite |
|---|---|
| **Iterator** ([44](./44-pattern-iterator.md)) | Almost always paired with it — Composite builds the tree, Iterator walks it. The `[Symbol.iterator]` generator above is exactly this. |
| **Visitor** ([50](./50-pattern-visitor.md)) | The natural next step. Composite puts operations *inside* the nodes; adding a new operation means editing every class. Visitor pulls operations *out* so you can add `PrettyPrintVisitor` or `TypeCheckVisitor` without touching `File`/`Directory`. Every real compiler does Composite + Visitor together. |
| **Decorator** ([35](./35-pattern-decorator.md)) | Structurally a *degenerate composite* — a wrapper with exactly one child instead of many. Both are recursive-composition patterns. |
| **Chain of Responsibility** ([47](./47-pattern-chain-of-responsibility.md)) | Composite trees often get CoR behaviour: DOM event bubbling walks a child up its parent chain until a handler consumes it. |
| **Flyweight** ([40](./40-pattern-flyweight.md)) | If a composite tree has millions of similar leaves (every character glyph in a document), share leaf instances via Flyweight. |
| **Builder** ([32](./32-pattern-builder.md)) | Composite trees are tedious to construct by hand; a fluent Builder (or JSX!) makes it readable. |

---

## Visual / Diagram description

### Diagram 1: The tree, and where the recursion bottoms out

```
                        Directory("project")            getSize() = 536,576
                        ┌────────────────┐
                        │  children[4]   │
                        └───┬──┬──┬───┬──┘
             ┌──────────────┘  │  │   └──────────────────┐
             │           ┌─────┘  └───────┐              │
             ▼           ▼                ▼              ▼
     Directory("src")  Dir("assets")   File(...)      File(...)
      getSize()=20,480  getSize()=512000  package.json   README.md
     ┌─────┬─────┬────┐   │             1,024  ◀LEAF     3,072  ◀LEAF
     │     │     │        ▼
     ▼     ▼     ▼    File("logo.png")
  File   File   Directory("components")   512,000  ◀LEAF
 app.js router.js  getSize() = 8,192
  8,192  4,096      ┌──────┬──────┐
  ◀LEAF ◀LEAF       ▼             ▼
                 File            File
              Button.jsx      Modal.jsx
                2,048  ◀LEAF    6,144  ◀LEAF

  Recursion:  Directory.getSize() = Σ child.getSize()   ← the composite delegates
              File.getSize()      = this.sizeBytes      ← the LEAF terminates it
```

Every `◀LEAF` is a base case. Every `Directory` is a recursive case. The tree *is* the call stack of `getSize()` — draw the tree and you've drawn the recursion.

### Diagram 2: What the client sees (and doesn't see)

```
  ┌────────────────────────────────────────────────────────────┐
  │  CLIENT CODE                                                │
  │                                                             │
  │    for (const node of [aFile, aFolder, deeplyNestedFolder]) │
  │        console.log(node.getSize());                         │
  │                                                             │
  │    ← NO `if (node instanceof Directory)`                    │
  │    ← NO `if (node.type === 'file')`                         │
  │    ← NO knowledge of depth, children, or recursion          │
  └───────────────────────────┬────────────────────────────────┘
                              │ sees ONLY this:
                              ▼
              ┌───────────────────────────────┐
              │ FileSystemNode { getSize() }   │   ← the whole contract
              └───────────────────────────────┘
                              │ polymorphism dispatches
              ┌───────────────┴───────────────┐
              ▼                               ▼
      ┌───────────────┐            ┌──────────────────────┐
      │ File          │            │ Directory            │
      │ return bytes  │            │ sum children,        │
      │  (BASE CASE)  │            │ each of which is     │
      └───────────────┘            │ a File or Directory  │
                                   │  (RECURSIVE CASE) ───┼──┐
                                   └──────────────────────┘  │
                                              ▲              │
                                              └──────────────┘
                                                  loops back
```

### Diagram 3: Safety vs Transparency, side by side

```
   TRANSPARENT (GoF, the DOM)          |   SAFE (most modern code, TypeScript)
                                       |
   Component                           |   Component
   ├ getSize()                         |   ├ getSize()
   ├ add(child)      ← lie for File    |   └ print()
   └ remove(child)   ← lie for File    |          │
          │                            |    ┌─────┴──────┐
    ┌─────┴──────┐                     |    ▼            ▼
    ▼            ▼                     |  File        Directory
  File        Directory                |  getSize()   getSize()
  add() → THROWS at runtime  ✗         |              add(child)     ← honest
  getSize() → bytes                    |              remove(name)   ← honest
                                       |
  Client: never type-checks   ✓        |  Client: must check before add  ✗
  Interface: bloated / dishonest ✗     |  Interface: honest              ✓
```

---

## Real world examples

### 1. The DOM — Composite as a web standard

The W3C DOM is the most-used Composite implementation on Earth. `Node` is the component interface; `Element` is the composite (`appendChild`, `removeChild`, `childNodes`); `Text` and `Comment` are leaves. Operations recurse transparently: `element.textContent` walks every descendant and concatenates their text, and `element.remove()` detaches an entire subtree in one call. The DOM deliberately chose **transparency** — `appendChild` is declared on `Node`, so you *can* call it on a `Text` node; it throws `HierarchyRequestError` at runtime. That's the safety cost, accepted in exchange for a uniform client API.

### 2. React — the component tree and reconciliation

A React element is either a **host element** (a `<div>` with children — composite), a **composite component** (your `<Layout>`, which renders more elements), or a **leaf** (a string, number, or an element with no children). Reconciliation is a depth-first recursive walk of this tree comparing old and new. JSX is, quite literally, a Builder syntax for constructing a composite tree: `<Layout><Sidebar/><Main/></Layout>` is `createElement(Layout, null, createElement(Sidebar), createElement(Main))`. React Fiber's linked-list traversal is an iterative rewrite of that recursion precisely to avoid blowing the JS call stack on deep trees — a real-world instance of the depth trade-off listed in section 6.

### 3. Babel, ESLint, and Prettier — the AST

Every one of these tools parses your source into an AST, which is a composite tree of node types (`Program` → `FunctionDeclaration` → `BlockStatement` → `ReturnStatement` → `BinaryExpression` → `Identifier`). Container nodes are composites; identifiers and literals are leaves. All three tools then apply the **Visitor** pattern over that composite — you register `{ BinaryExpression(node) { ... } }` and the traversal engine calls you at each matching node. This is the canonical Composite + Visitor pairing, and it's why "how would you add a new operation without editing every node class?" is such a common follow-up question.

### 4. AWS IAM / organisational permission trees (representative)

Cloud permission systems model accounts, organisational units, and policies as a tree: an OU is a composite that can contain accounts (leaves) or more OUs. "What is the effective policy for this account?" is answered by walking from the leaf up to the root and combining — the same recursion, run in the other direction.

---

## Trade-offs

| Pro | What you gain |
|---|---|
| **Uniform client code** | No `if (isLeaf)` anywhere. One call handles one item or ten thousand. |
| **Open/Closed for new node types** | Add `Symlink extends FileSystemNode` and *no existing traversal changes.* |
| **Recursion comes free** | The tree structure *is* the recursion; you write the base case once and the combine step once. |
| **Arbitrary nesting** | Depth is not a design decision you have to make up front. |
| **Composes with Iterator/Visitor** | Traversal and operations can both be pulled out cleanly later. |

| Con | What it costs |
|---|---|
| **Interface bloat (transparency)** | `add`/`remove` on the Component are meaningless for leaves — a runtime landmine |
| **Type safety (safety)** | Restricting `add` to the composite forces the client to type-check when building |
| **Over-generalisation** | The design says "anything can contain anything," even when your domain forbids it (a `<td>` shouldn't hold a `<html>`) — you end up validating in `add()` anyway |
| **Performance** | `getSize()` is O(n) on every call; deep trees risk stack overflow; pointer-chasing kills cache locality vs a flat array |
| **Adding a new *operation* is expensive** | You must edit *every* node class. (This is precisely the pain Visitor solves.) |
| **Debugging** | A stack trace through a 12-deep recursion is unpleasant to read |

**The sweet spot:** Use Composite when you have a genuine **part-whole hierarchy of arbitrary depth** *and* a set of operations that are meaningful on both the part and the whole (`getSize`, `render`, `validate`, `price`). Put the read operations on the shared interface for full transparency; keep `add`/`remove` on the composite for safety. If you're going to add many new operations over time, plan for **Composite + Visitor** from the start.

---

## Common interview questions on this topic

### Q1: "Design an in-memory file system."
**Hint:** This is Composite, and you should name it in the first 30 seconds. Abstract `FileSystemNode` (name, parent, `getSize()`, `getPath()`); `File` leaf (holds bytes); `Directory` composite (holds a `Map<name, Node>` for O(1) child lookup, `add`/`remove`, `getSize()` = sum of children). Then the interviewer will ask for **path resolution** (`resolve('/a/b/c.txt')` — split on `/`, walk down, handle `.` and `..` via the parent pointer), **search** (recursive `find(predicate)`), and possibly **permissions** (a bitmask on the node, checked while walking down the path — which is a **Proxy**-flavoured access check layered on the Composite).

### Q2: "What's the safety vs transparency trade-off in Composite?"
**Hint:** Where do you declare `add`/`remove`? **Transparency** = declare them on the Component, so the client never type-checks — but `File.add()` is now a nonsense method that must throw at runtime (the DOM does this; `appendChild` on a `Text` node throws). **Safety** = declare them only on the Composite, so the interface is honest and TypeScript catches misuse at compile time — but the client must now know whether it's holding a composite before it can build the tree. The pragmatic answer: transparent *read* operations, safe *mutation* operations, plus a cheap `isComposite()` escape hatch.

### Q3: "How do you add a new operation — say, `countFiles()` — to your composite?"
**Hint:** Two paths, and knowing both is the point. **(a)** Add the method to `FileSystemNode`, `File`, and `Directory` — simple, but you edit every class, and with 15 node types that's 15 edits. **(b)** Use **Visitor**: define `accept(visitor)` on each node once, then add `CountFilesVisitor` without touching the node classes ever again. Rule: if node types are stable and operations keep growing → Visitor. If operations are stable and node types keep growing → plain Composite. This is the "expression problem" and interviewers love it.

### Q4: "Your `getSize()` on the root takes 200 ms because the tree has a million nodes and it's called on every render. Fix it."
**Hint:** Memoize on the composite: cache `#cachedSize`, return it if valid. Invalidate on mutation — but you must invalidate **up the parent chain**, not just locally: `add`/`remove` walks `node.parent` to the root, clearing each cached size. That makes reads O(1) amortized and writes O(depth), which is exactly the right trade for a read-heavy tree. This is the same technique React uses with dirty-flagging subtrees.

### Q5: "What breaks if your composite tree has a cycle?"
**Hint:** Everything — `getSize()` recurses forever and blows the stack. Composite assumes a **tree**: exactly one parent per node, no cycles. Defend in `add()`: reject adding a node to itself, and walk the parent chain to reject adding an ancestor as a child (`for (let p = this; p; p = p.parent) if (p === node) throw`). If you genuinely need a DAG or graph (hard links, symlinks), you need a `visited` Set in every traversal, and `getSize()` has to decide whether shared subtrees are counted once or many times — which is a *product* decision, not a code one.

### Q6: "How is Composite different from Decorator? They both wrap objects recursively."
**Hint:** A Decorator is a **degenerate Composite with exactly one child**. But intent differs completely: Decorator's purpose is to *add behaviour* to a single wrapped object (a chain), while Composite's purpose is to *treat many objects as one* (a tree). Decorator has one component and modifies its result; Composite has N components and aggregates them.

---

## Practice exercise

### Build a UI layout tree with `render()` and `measure()`

Create `layout.js` in Node. You're building the composite core of a tiny UI framework — the same shape as React or Flutter, minus the rendering backend.

**Step 1 — The Component.** Write an abstract `Widget` class with `name`, a `parent` back-pointer, and three abstract methods:
- `measure()` → returns `{ width, height }` in "cells"
- `render(indent = 0)` → prints an ASCII tree of the layout
- `find(predicate)` → returns a flat array of matching widgets

**Step 2 — The Leaves.** Write `TextWidget(content)` where `measure()` returns `{ width: content.length, height: 1 }`, and `ImageWidget(w, h)` where `measure()` returns its fixed dimensions.

**Step 3 — The Composites.**
- `Column(name)` — children stacked vertically. `measure()` → `width = max(child widths)`, `height = sum(child heights)`.
- `Row(name)` — children side by side. `measure()` → `width = sum(child widths)`, `height = max(child heights)`.

Both get `add(widget)` (chainable, sets `parent`) and `remove(widget)`. Note the delicious detail: **`Column` and `Row` differ only in which dimension they sum and which they max.** Everything else is identical — that's the pattern doing its job.

**Step 4 — The demo.** Build this tree and prove it works:
```
Column("app")
 ├─ Row("header")   → Image(32x4), Text("My Dashboard")
 ├─ Row("body")
 │   ├─ Column("sidebar") → Text("Home"), Text("Settings"), Text("Logout")
 │   └─ Column("content") → Text("Welcome back!"), Image(60x20)
 └─ Text("© 2026")
```
Then, in a loop with **no type checks whatsoever**, print the measured size of: the root `Column`, the `sidebar` `Column`, and a bare `TextWidget`. That loop is the whole point — if you had to write an `if`, you've built it wrong.

**Step 5 — Answer these in comments at the bottom:**
1. You now need a `toJSON()` operation, and then a `countWidgets()`, and then an `applyTheme()`. What does Composite cost you for each? Which pattern would you introduce, and when exactly would you make that call?
2. `measure()` is O(n) and gets called on every frame. Add memoization with correct invalidation up the parent chain on `add`/`remove`. Prove it's correct by asserting the size changes after a `remove()`.
3. Should `add()` live on `Widget` or only on `Column`/`Row`? Argue your side in three sentences, and say which real framework agrees with you.

---

## Quick reference cheat sheet

- **Composite = treat a single object and a group of objects identically**, via one shared interface. The group can contain groups → it's a **tree**.
- **Three roles: Component** (the shared interface), **Leaf** (no children, does the real work, *terminates the recursion*), **Composite** (holds children, *delegates and combines*).
- **The composite holds `Component` children, not `Leaf` children.** That self-reference is what makes arbitrary nesting possible.
- **The payoff is one line of client code with no `if`.** `node.getSize()` works on a file, a folder, or a 10,000-node subtree.
- **Recursion is free:** the leaf is the base case, the composite is the recursive case. The tree *is* the call stack.
- **Safety vs Transparency:** put `add`/`remove` on the Component (transparent, but leaves must throw) or only on the Composite (safe, but the client must type-check). **Pragmatic answer: transparent reads, safe mutations, plus `isComposite()`.**
- **The DOM chose transparency** — `appendChild` is on `Node`, and it throws on a `Text` node.
- **Real composites you use daily:** the DOM, React's component tree, ASTs (Babel/ESLint/Prettier), nested menus, org charts, file systems.
- **Composite + Iterator** = walk the tree cleanly (`*[Symbol.iterator]()` with `yield*`).
- **Composite + Visitor** = add new operations without editing every node class. Every compiler does this.
- **Decorator is a Composite with exactly one child** — same recursion, different intent (add behaviour vs aggregate parts).
- **Watch out for:** cycles (infinite recursion — validate in `add()`), deep trees (stack overflow ~10k frames in Node), and O(n) reads on every call (memoize, invalidate up the parent chain).
- **Don't use it** when nesting is one level deep, or when leaves and composites genuinely need different operations — the shared interface would be a lie.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [37 — Proxy Pattern](./37-pattern-proxy.md) — a stand-in with the same interface that controls access to one object |
| **Next** | [39 — Bridge Pattern](./39-pattern-bridge.md) — the other structural pattern about splitting a hierarchy, this time abstraction from implementation |
| **Related** | [50 — Visitor Pattern](./50-pattern-visitor.md) — the essential partner: add new operations to a composite tree without editing every node class |
| **Related** | [44 — Iterator Pattern](./44-pattern-iterator.md) — how to walk the tree you just built without exposing its structure |
| **Related** | [132 — Design an In-Memory File System](./132-lld-file-system.md) — the LLD interview question that is Composite, end to end |
