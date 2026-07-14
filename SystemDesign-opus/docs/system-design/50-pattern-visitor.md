# 50 — Visitor Pattern
## Category: LLD Patterns

---

## What is this?

The Visitor pattern lets you **add a new operation to a family of classes without editing a single one of them.** You move the operation *out* of the objects and into a separate "visitor" object, and the objects simply agree to let visitors in.

Think of a **building inspector**. An apartment building has rooms: Kitchen, Bedroom, Bathroom, Garage. Today the fire inspector walks through and checks each room for fire hazards. Tomorrow the electrical inspector walks through and checks wiring. Next month the tax assessor walks through and estimates value. The **rooms never change** — they just open the door and say "come in, I'm a Kitchen." Each new inspector is a new *visitor*, not a change to the building.

**Be warned up front: this is the hardest pattern in the Gang of Four book.** Not because the code is long, but because the control flow bounces — you call `node.accept(visitor)` and the *visitor's* method runs, which calls `accept` again on children. It feels backwards the first three times you read it. That's normal. We're going to go slowly.

---

## Why does it matter?

Here is the situation you will actually hit at work.

You build a small expression language — a formula engine for a spreadsheet, a rules engine for pricing, a query filter for an API. You parse text into a tree of nodes: `Number`, `Variable`, `Add`, `Multiply`. That tree — the **AST** (Abstract Syntax Tree, just a tree where each node is a piece of the parsed expression) — is *done*. It has 4 node types and it will have 4 node types next year.

But the **operations** on that tree never stop coming:

1. Week 1: "Evaluate the expression." → you write `evaluate()` on all 4 classes.
2. Week 3: "Show the formula back to the user." → you write `print()` on all 4 classes.
3. Week 6: "Simplify `x * 1` to `x`." → you write `optimize()` on all 4 classes.
4. Week 9: "Reject formulas that add a string to a number." → you write `typeCheck()` on all 4 classes.
5. Week 12: "Tell me which variables a formula depends on." → `collectVars()` on all 4 classes.

Every new feature means **opening and editing every node class**. That's an **Open/Closed Principle violation** (recall from [15 — SOLID: Open/Closed](./15-solid-open-closed.md): software should be open to extension, closed to modification). And worse: the logic for "optimize" is now smeared across four files. If optimization has a bug, you read four files to find it.

**Visitor flips this.** Node classes get one method — `accept(visitor)` — and are then **never touched again**. Each new operation is **one new class in one new file**.

**Interview angle:** Visitor is the classic "do you actually understand double dispatch?" question. Interviewers love it because most candidates can recite the structure but cannot explain *why* `accept` exists. If you can explain double dispatch in 60 seconds, you look senior.

**Real-work angle:** If you have ever written an ESLint rule or a Babel plugin, you have already written a Visitor. You just didn't know its name.

---

## The core idea — explained simply

### The Museum Tour Analogy

A museum has exhibits: a **Painting**, a **Sculpture**, and a **Dinosaur Skeleton**. These exhibits have been there for 40 years and are not changing.

Different people walk through the museum, and each does something *different* at each exhibit:

| Visitor | At the Painting | At the Sculpture | At the Dinosaur |
|---------|-----------------|------------------|-----------------|
| **Tourist** | Takes a photo | Takes a photo | Takes a selfie |
| **Insurance Appraiser** | Estimates $2M | Estimates $400k | Estimates "priceless" |
| **Cleaner** | Dusts the frame | Polishes the bronze | Vacuums the base |
| **Art Historian** | Writes a 20-page essay | Writes 3 pages | Ignores it entirely |

Notice two things.

**First:** the behaviour depends on **two** things — *which exhibit* and *which visitor*. Painting + Appraiser = "$2M". Painting + Cleaner = "dust the frame". Same exhibit, totally different action. That two-way dependency is the whole pattern, and it has a name: **double dispatch**.

**Second:** to add a **new visitor** (say, a School Group), you add one new person. The exhibits don't change. But to add a **new exhibit** (say, a Meteorite), *every single visitor* now needs to know what to do at the meteorite. The tourist, the appraiser, the cleaner, the historian — all four must be updated.

That asymmetry is the pattern's soul, and its price:

| The museum | The code |
|---|---|
| Exhibit (Painting, Sculpture, …) | **Element** — the node class in your stable structure |
| "Come in, I'm a Painting" | `accept(visitor)` — the one method every element has |
| Visitor (Tourist, Appraiser, …) | **Visitor** — one class per operation |
| "What do I do at a Painting?" | `visitPainting(painting)` — one method per element type |
| Adding a new visitor is cheap | Adding a new **operation** is cheap |
| Adding a new exhibit is expensive | Adding a new **element type** is expensive |

Hold on to that last pair. It decides whether you should use this pattern at all.

---

## Key concepts inside this topic

### 1. The problem — painful code without the pattern

Here is an expression AST doing it the "obvious" OOP way: each node knows how to do everything.

```javascript
// ❌ BAD — every new operation forces an edit to EVERY class.

class NumberNode {
  constructor(value) { this.value = value; }
  evaluate(env)  { return this.value; }
  print()        { return String(this.value); }
  optimize()     { return this; }
  typeCheck()    { return 'number'; }
  // ...and next sprint: collectVars(), toSQL(), toLatex(), estimateCost()...
}

class VariableNode {
  constructor(name) { this.name = name; }
  evaluate(env)  { return env[this.name]; }
  print()        { return this.name; }
  optimize()     { return this; }
  typeCheck()    { return 'number'; }
}

class AddNode {
  constructor(left, right) { this.left = left; this.right = right; }
  evaluate(env)  { return this.left.evaluate(env) + this.right.evaluate(env); }
  print()        { return `(${this.left.print()} + ${this.right.print()})`; }
  optimize()     { /* 25 lines of constant folding */ }
  typeCheck()    { /* 15 lines */ }
}

class MultiplyNode {
  constructor(left, right) { this.left = left; this.right = right; }
  evaluate(env)  { return this.left.evaluate(env) * this.right.evaluate(env); }
  print()        { return `(${this.left.print()} * ${this.right.print()})`; }
  optimize()     { /* another 25 lines */ }
  typeCheck()    { /* another 15 lines */ }
}
```

Three things are now wrong, and they get worse every sprint:

1. **OCP violation.** Adding `toSQL()` means opening all four files. Four files touched, four chances to break something, four files in the code review.
2. **Scattered logic.** "Optimization" is one coherent idea, but you cannot read it in one place — it lives in four fragments in four classes.
3. **Fat classes.** `AddNode` should model "a plus b". Instead it's a 200-line grab bag of every operation anyone ever wanted, most of which have nothing to do with addition.

### 2. The Visitor inversion

Give each node **exactly one** method — `accept(visitor)` — whose entire job is to call the *right* method back on the visitor:

```javascript
class AddNode {
  constructor(left, right) { this.left = left; this.right = right; }

  // The ONLY method this class will ever have again.
  // It knows one thing the visitor cannot know: "I am an Add."
  accept(visitor) {
    return visitor.visitAdd(this);
  }
}
```

Read that call carefully: `visitor.visitAdd(this)`. The node calls *back into* the visitor, naming its own type, and hands over itself. That's it. That's the whole trick.

Now each operation is one class:

```javascript
class EvalVisitor {
  visitNumber(node)   { /* how to evaluate a number */ }
  visitVariable(node) { /* how to evaluate a variable */ }
  visitAdd(node)      { /* how to evaluate an addition */ }
  visitMultiply(node) { /* how to evaluate a multiplication */ }
}
```

Want printing? Write `PrintVisitor` with the same four method names. Want optimization? Write `OptimizeVisitor`. **The node classes are never opened again.** You have inverted the axis of change.

### 3. Double dispatch — the part everyone struggles with

This is the concept the interviewer is really testing. Slow down here.

**Single dispatch** is what every OOP language gives you for free. When you write `obj.method()`, the runtime picks the implementation based on **one** thing: the runtime type of `obj`. One type in, one method chosen. That's "single".

But our problem needs the method chosen by **two** types: the node's type **and** the visitor's type. `Add + Eval` should run addition; `Add + Print` should run string-joining. JavaScript has no built-in way to say "dispatch on two types at once."

**The naive fix — and why it rots.** You could type-switch inside the visitor:

```javascript
// ❌ Type-sniffing. Works, and rots.
class EvalVisitor {
  visit(node) {
    if (node instanceof NumberNode)        return node.value;
    else if (node instanceof VariableNode) return this.env[node.name];
    else if (node instanceof AddNode)      return this.visit(node.left) + this.visit(node.right);
    else if (node instanceof MultiplyNode) return this.visit(node.left) * this.visit(node.right);
    else throw new Error('Unknown node type');
  }
}
```

Every visitor now repeats this `if/else instanceof` ladder. Add a node type and you must hunt down every ladder — and if you forget one, you get a **silent runtime failure**, not a compile error. The ladder also gets slower and uglier with each type.

**The real fix.** Only the node itself reliably knows its own concrete type — no `instanceof` needed, no ladder. So we let the node *tell* us, by calling a differently-named method:

```
Step 1 (first dispatch)   node.accept(visitor)
                          └─▶ picked by NODE's type. We land in AddNode.accept.
                              AddNode.accept knows it is an Add. It calls:

Step 2 (second dispatch)  visitor.visitAdd(node)
                          └─▶ picked by VISITOR's type. We land in EvalVisitor.visitAdd.

Result: EvalVisitor.visitAdd ran. Chosen by BOTH types. Double dispatch.
```

Two ordinary single dispatches, chained, give you dispatch on two types. `accept` exists for exactly one reason: **to recover the node's concrete type without asking `instanceof`.** That sentence is your interview answer.

Trace it concretely. `new AddNode(...)` and `new PrintVisitor()`:

```
call:  addNode.accept(printVisitor)
  ──▶  AddNode.accept runs        (dispatch #1: on AddNode)
  ──▶  it calls printVisitor.visitAdd(addNode)
  ──▶  PrintVisitor.visitAdd runs (dispatch #2: on PrintVisitor)
  ──▶  which recurses: addNode.left.accept(printVisitor)  ← same dance, one level down
```

Swap `printVisitor` for `evalVisitor` and dispatch #1 is **identical** — `AddNode.accept` doesn't change at all — but dispatch #2 lands somewhere completely different. That's the pattern earning its keep.

### 4. The structure — who plays which role

| Participant | Role | In our example |
|---|---|---|
| **Element** | Abstract thing that can be visited; declares `accept(visitor)` | `ExprNode` (base class) |
| **ConcreteElement** | A real node type; implements `accept` by calling its *own* visit method | `NumberNode`, `VariableNode`, `AddNode`, `MultiplyNode` |
| **Visitor** | Declares one `visitX` method per concrete element | `Visitor` (base class) |
| **ConcreteVisitor** | One operation, fully implemented across all node types | `EvalVisitor`, `PrintVisitor` |
| **ObjectStructure** | Holds/produces the elements and kicks off traversal | The AST root, or a `Parser` |

The one rule that makes it work: **`ConcreteElement.accept` must call the visit method for its own type, and nothing else.** `AddNode.accept` calls `visitAdd`. Never `visitMultiply`. It is a one-line method and it should stay a one-line method.

### 5. Full JavaScript implementation

Save as `visitor.js` and run with `node visitor.js`.

```javascript
// ============================================================
// ELEMENTS — the stable object structure. Written once, then frozen.
// ============================================================

class ExprNode {
  // Every node promises to let a visitor in. That's its ONLY promise.
  accept(visitor) {
    throw new Error('accept() must be implemented by subclass');
  }
}

class NumberNode extends ExprNode {
  constructor(value) { super(); this.value = value; }
  accept(visitor) { return visitor.visitNumber(this); }
}

class VariableNode extends ExprNode {
  constructor(name) { super(); this.name = name; }
  accept(visitor) { return visitor.visitVariable(this); }
}

class AddNode extends ExprNode {
  constructor(left, right) { super(); this.left = left; this.right = right; }
  accept(visitor) { return visitor.visitAdd(this); }
}

class MultiplyNode extends ExprNode {
  constructor(left, right) { super(); this.left = left; this.right = right; }
  accept(visitor) { return visitor.visitMultiply(this); }
}

// ============================================================
// VISITOR BASE — the contract. JS has no interfaces, so we throw
// on unimplemented methods. This turns "I forgot a node type" from
// a silent `undefined` into a loud, immediate crash.
// ============================================================

class Visitor {
  visitNumber(node)   { throw new Error(`${this.constructor.name} must implement visitNumber`); }
  visitVariable(node) { throw new Error(`${this.constructor.name} must implement visitVariable`); }
  visitAdd(node)      { throw new Error(`${this.constructor.name} must implement visitAdd`); }
  visitMultiply(node) { throw new Error(`${this.constructor.name} must implement visitMultiply`); }
}

// ============================================================
// CONCRETE VISITOR 1 — evaluate the expression to a number.
// The ENTIRE evaluation algorithm lives in this one class.
// ============================================================

class EvalVisitor extends Visitor {
  // env maps variable names to values: { x: 5, y: 2 }
  constructor(env = {}) { super(); this.env = env; }

  visitNumber(node) { return node.value; }

  visitVariable(node) {
    if (!(node.name in this.env)) throw new Error(`Undefined variable: ${node.name}`);
    return this.env[node.name];
  }

  // Recursion happens HERE, in the visitor — not in the node.
  // The visitor decides the traversal order, which means a different
  // visitor could traverse differently if it needed to.
  visitAdd(node) {
    return node.left.accept(this) + node.right.accept(this);
  }

  visitMultiply(node) {
    return node.left.accept(this) * node.right.accept(this);
  }
}

// ============================================================
// CONCRETE VISITOR 2 — render the expression back to a string.
// Note: we did NOT touch a single node class to add this.
// ============================================================

class PrintVisitor extends Visitor {
  visitNumber(node)   { return String(node.value); }
  visitVariable(node) { return node.name; }
  visitAdd(node)      { return `(${node.left.accept(this)} + ${node.right.accept(this)})`; }
  visitMultiply(node) { return `(${node.left.accept(this)} * ${node.right.accept(this)})`; }
}

// ============================================================
// CONCRETE VISITOR 3 — collect every variable name used.
// Proof of the payoff: a brand-new operation, zero edits elsewhere.
// A visitor may carry state across the traversal (here: a Set).
// ============================================================

class CollectVarsVisitor extends Visitor {
  constructor() { super(); this.names = new Set(); }

  visitNumber(node)   { /* numbers contain no variables */ }
  visitVariable(node) { this.names.add(node.name); }
  visitAdd(node)      { node.left.accept(this); node.right.accept(this); }
  visitMultiply(node) { node.left.accept(this); node.right.accept(this); }
}

// ============================================================
// CONCRETE VISITOR 4 — constant folding.
// Returns a NEW tree instead of a value. A visitor's return type is
// whatever you want it to be: number, string, or another node.
//   3 + 4     -> 7          (both sides are constants: fold them)
//   x * 1     -> x          (identity: drop the multiply)
//   x * 0     -> 0          (annihilator)
//   x + 0     -> x          (identity)
// ============================================================

class OptimizeVisitor extends Visitor {
  visitNumber(node)   { return node; }
  visitVariable(node) { return node; }

  visitAdd(node) {
    const left = node.left.accept(this);   // optimize children first (bottom-up)
    const right = node.right.accept(this);

    if (left instanceof NumberNode && right instanceof NumberNode) {
      return new NumberNode(left.value + right.value);
    }
    if (left instanceof NumberNode && left.value === 0) return right;  // 0 + x -> x
    if (right instanceof NumberNode && right.value === 0) return left; // x + 0 -> x
    return new AddNode(left, right);
  }

  visitMultiply(node) {
    const left = node.left.accept(this);
    const right = node.right.accept(this);

    if (left instanceof NumberNode && right instanceof NumberNode) {
      return new NumberNode(left.value * right.value);
    }
    // x * 0 -> 0  (safe here: our language has no side effects or NaN inputs)
    if (left instanceof NumberNode && left.value === 0) return new NumberNode(0);
    if (right instanceof NumberNode && right.value === 0) return new NumberNode(0);
    if (left instanceof NumberNode && left.value === 1) return right;  // 1 * x -> x
    if (right instanceof NumberNode && right.value === 1) return left; // x * 1 -> x
    return new MultiplyNode(left, right);
  }
}

// ============================================================
// DEMO
// ============================================================

function main() {
  // Build the AST for:  (x * 1) + (2 * 3)
  //
  //            Add
  //           /   \
  //      Multiply  Multiply
  //       /   \     /   \
  //     x      1   2     3
  const ast = new AddNode(
    new MultiplyNode(new VariableNode('x'), new NumberNode(1)),
    new MultiplyNode(new NumberNode(2), new NumberNode(3))
  );

  const printer = new PrintVisitor();
  console.log('Expression :', ast.accept(printer));

  const evaluator = new EvalVisitor({ x: 10 });
  console.log('Eval (x=10):', ast.accept(evaluator));

  const collector = new CollectVarsVisitor();
  ast.accept(collector);
  console.log('Variables  :', [...collector.names]);

  const optimized = ast.accept(new OptimizeVisitor());
  console.log('Optimized  :', optimized.accept(printer));
  console.log('Eval again :', optimized.accept(new EvalVisitor({ x: 10 })));
}

main();
```

**Output:**

```
Expression : ((x * 1) + (2 * 3))
Eval (x=10): 16
Variables  : [ 'x' ]
Optimized  : (x + 6)
Eval again : 16
```

Look at what just happened. `(x * 1) + (2 * 3)` became `(x + 6)` — the `* 1` vanished and `2 * 3` was folded into `6` at "compile time". Both expressions still evaluate to 16. And we added four completely different operations — evaluate, print, collect, optimize — **without editing `AddNode` even once.**

### 6. When NOT to use it — the killer trade-off

Visitor is not free. It trades one kind of extensibility for the other, and you must know which kind you need.

| You want to add… | Without Visitor (methods on nodes) | With Visitor |
|---|---|---|
| **A new operation** (`toSQL`) | Edit **every** node class | **One** new file. Zero edits. |
| **A new node type** (`Subtract`) | **One** new file. Zero edits. | Edit **every** visitor class |

This is the **expression problem**, and it is genuinely fundamental — no pattern gives you both cheaply. You get to pick which axis is cheap. So:

- **Structure stable + operations multiplying** → Visitor. (ASTs, document trees, scene graphs, IR in compilers.)
- **Operations stable + types multiplying** → **Do NOT use Visitor.** Put the methods on the classes. If you have a `Shape` hierarchy where new shapes ship every month but the only operation is `draw()`, a Visitor is pure pain: every new shape means editing every visitor.

Three more reasons to walk away:

1. **Broken encapsulation.** Visitors need to reach into node internals (`node.left`, `node.value`), so nodes end up with public fields or lots of getters. The pattern leaks state by design.
2. **Cognitive cost.** Junior engineers will be lost in the `accept` → `visitX` → `accept` bounce for a while. If your structure has 3 node types and 2 operations, plain methods are clearer and you should just write plain methods.
3. **Cycles.** Naive visitors recurse forever on a cyclic structure. Trees are fine; general graphs need a `visited` Set.

### 7. Related patterns — and how they differ

The #1 interview follow-up. Know these cold.

| Pattern | Confusion with Visitor | The actual difference |
|---|---|---|
| **Iterator** ([44](./44-pattern-iterator.md)) | Both walk a structure | Iterator gives you elements **one at a time, all treated the same**. Visitor does **type-specific work** at each node. Iterator = "give me the next thing"; Visitor = "do the right thing for *this kind* of thing". They compose: iterate a collection, `accept` a visitor on each item. |
| **Strategy** ([40](./40-pattern-strategy.md)) | Both extract an algorithm into its own object | Strategy has **one** method and is chosen by the *caller*. Visitor has **N** methods (one per element type) and the method is chosen by the *element*. Strategy = single dispatch; Visitor = double dispatch. |
| **Composite** ([38](./38-pattern-composite.md)) | Both live on trees | Composite *is* the tree structure. Visitor is the operation you run *over* a Composite. They are best friends — Composite builds the AST, Visitor operates on it. |
| **Command** ([43](./43-pattern-command.md)) | Both turn an action into an object | Command wraps **one** action to defer/queue/undo it. Visitor wraps a **family** of type-dispatched actions to run across a structure. |
| **Interpreter** | Almost the same code | Interpreter puts `interpret()` *on the nodes*. Visitor pulls it out. Interpreter is the "bad example" from section 1 — and it's fine when you only ever need to evaluate. |

---

## Visual / Diagram description

### Diagram 1: Class structure

```
   ┌────────────────────────┐           ┌───────────────────────────────┐
   │      «abstract»        │           │         «abstract»            │
   │       ExprNode         │           │          Visitor              │
   ├────────────────────────┤           ├───────────────────────────────┤
   │ + accept(visitor)      │           │ + visitNumber(node)           │
   └───────────┬────────────┘           │ + visitVariable(node)         │
               │                        │ + visitAdd(node)              │
      ┌────────┼────────┬──────────┐    │ + visitMultiply(node)         │
      │        │        │          │    └───────────────┬───────────────┘
      ▼        ▼        ▼          ▼                    │
 ┌────────┐ ┌──────┐ ┌──────┐ ┌──────────┐    ┌─────────┼──────────┐
 │Number  │ │Var   │ │Add   │ │Multiply  │    │         │          │
 │Node    │ │Node  │ │Node  │ │Node      │    ▼         ▼          ▼
 ├────────┤ ├──────┤ ├──────┤ ├──────────┤ ┌────────┐┌────────┐┌──────────┐
 │+value  │ │+name │ │+left │ │+left     │ │Eval    ││Print   ││Optimize  │
 │        │ │      │ │+right│ │+right    │ │Visitor ││Visitor ││Visitor   │
 │accept()│ │accept│ │accept│ │accept()  │ ├────────┤├────────┤├──────────┤
 │  └─calls│ │()   │ │()   │ │           │ │visitNum││visitNum││visitNum  │
 │  visitor│ │     │ │     │ │           │ │visitVar││visitVar││visitVar  │
 │  .visit │ │     │ │     │ │           │ │visitAdd││visitAdd││visitAdd  │
 │  Number │ │     │ │     │ │           │ │visitMul││visitMul││visitMul  │
 └────────┘ └──────┘ └──────┘ └──────────┘ └────────┘└────────┘└──────────┘
      ▲                                          │
      └──────── every accept() calls ────────────┘
               ONE method on the visitor
```

**What it shows:** two parallel hierarchies. Left = the object structure (frozen). Right = the operations (grows freely). The arrow at the bottom is the callback: each element's `accept` reaches across to exactly one method on the visitor. Adding a box to the **right** column is free. Adding a box to the **left** column forces an edit in *every* box on the right.

### Diagram 2: Double dispatch, step by step

```
  YOU                    AddNode                    PrintVisitor
   │                        │                            │
   │  accept(printVisitor)  │                            │
   ├───────────────────────▶│                            │
   │                        │  ── DISPATCH #1 ──         │
   │                        │  Chosen by the NODE's type │
   │                        │  We are in AddNode.accept  │
   │                        │  It knows: "I am an Add"   │
   │                        │                            │
   │                        │   visitAdd(this)           │
   │                        ├───────────────────────────▶│
   │                        │                            │  ── DISPATCH #2 ──
   │                        │                            │  Chosen by the VISITOR's type
   │                        │                            │  We are in PrintVisitor.visitAdd
   │                        │                            │
   │                        │  left.accept(this) ◀───────┤  (recurse into children)
   │                        │  right.accept(this) ◀──────┤
   │                        │                            │
   │                        │      "(x + 6)"             │
   │◀───────────────────────┴────────────────────────────┤
   │                                                     │
```

**What it shows:** the method that finally runs — `PrintVisitor.visitAdd` — was selected by **two** types, in two hops. Hop 1 recovers the node's type. Hop 2 recovers the visitor's type. Neither hop uses `instanceof`. Redraw this on a whiteboard and you can explain double dispatch to anyone.

---

## Real world examples

### 1. ESLint — every rule is a Visitor

ESLint parses your JS into an AST (node types: `Identifier`, `CallExpression`, `VariableDeclaration`, `BinaryExpression`, …). That node vocabulary is defined by the ECMAScript spec and is **extremely stable**. But the *operations* — the rules — number in the thousands across ESLint core and its plugin ecosystem.

That is the exact shape Visitor is for. An ESLint rule literally returns a visitor object, keyed by node type:

```javascript
// A real-shaped ESLint rule: ban `console.log`.
// The exported object's `create` returns a VISITOR — keys are node types.
module.exports = {
  meta: { type: 'problem', docs: { description: 'disallow console.log' } },

  create(context) {
    return {
      // ESLint walks the AST and calls this for EVERY CallExpression node.
      // This is `visitCallExpression` — just spelled with an object key.
      CallExpression(node) {
        const callee = node.callee;
        if (
          callee.type === 'MemberExpression' &&
          callee.object.name === 'console' &&
          callee.property.name === 'log'
        ) {
          context.report({ node, message: 'Unexpected console.log' });
        }
      },

      // Same visitor, second node type. Ban `debugger` too.
      DebuggerStatement(node) {
        context.report({ node, message: 'Unexpected debugger statement' });
      },
    };
  },
};
```

ESLint's traverser does the `accept` half for you: it walks the tree and, at each node, looks up `visitor[node.type]` and calls it if present. Notice the payoff — writing a *new rule* means writing *one new file*. Nobody has ever had to modify the `CallExpression` node class to add a rule.

### 2. Babel — plugins are Visitors that rewrite the tree

A Babel plugin is a visitor whose methods **mutate or replace** nodes — exactly like our `OptimizeVisitor`, but for real JavaScript:

```javascript
// A Babel plugin that rewrites every `var` into `let`.
module.exports = function () {
  return {
    visitor: {
      VariableDeclaration(path) {
        if (path.node.kind === 'var') path.node.kind = 'let';
      },
    },
  };
};
```

Babel adds two refinements worth knowing. First, `path` (not the raw node) — a wrapper that also knows the node's parent and scope, so you can call `path.replaceWith(...)` or `path.remove()` safely. Second, **enter/exit** hooks: `Identifier: { enter(path) {...}, exit(path) {...} }` lets you act on the way down *and* on the way back up. Both are practical extensions of the same double-dispatch core.

### 3. Compilers and query builders

**Compilers.** Almost every compiler is a pipeline of visitors over one stable AST: a type-check pass, a constant-folding pass, a dead-code-elimination pass, then a code-generation pass. Each is a visitor; each is independently testable; the AST classes were written once. TypeScript's own compiler is structured this way (its "transformers" are visitor-shaped).

**Query builders / ORMs.** You build a filter tree — `And(Eq('status','active'), Gt('age', 21))` — and then run different visitors over it: one emits PostgreSQL SQL, another emits a MongoDB query object, a third emits a human-readable description for an audit log. Three completely different backends, one tree, zero changes to the node classes. Adding a fourth backend is a fourth visitor.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **New operations are cheap** | One new class. The Open/Closed Principle, honoured properly. |
| **Related logic stays together** | The whole optimizer is in one file, not smeared across four node classes. |
| **Nodes stay thin** | `AddNode` models addition, not addition + printing + typing + SQL. |
| **Visitors can accumulate state** | Walk once, collect a Set, count nodes, build a symbol table. |
| **Easy to test** | Test one visitor in isolation with a hand-built tree. |

| Cons | What it costs you |
|---|---|
| **New element types are expensive** | Adding `SubtractNode` means editing *every* visitor. Miss one → runtime crash. |
| **Encapsulation leaks** | Visitors read node internals; nodes need public fields or many getters. |
| **Hard to read at first** | The `accept` → `visitX` bounce is genuinely confusing to newcomers. |
| **Boilerplate** | Every node needs an `accept`. Every visitor needs N methods. |
| **Cycles break naive traversal** | Fine for trees; graphs need explicit cycle detection. |

**Rule of thumb:** Count your axes of change. If **node types are stable and operations keep multiplying**, use Visitor — it is the right tool and nothing else comes close. If **operations are stable and node types keep multiplying**, use plain polymorphic methods on the classes and never speak of Visitor again. And if you have fewer than ~4 node types and ~3 operations, skip the pattern entirely: plain methods are simpler, and simplicity wins until it doesn't.

---

## Common interview questions on this topic

### Q1: "Explain double dispatch. Why does `accept()` exist at all?"
**Hint:** Ordinary method calls dispatch on **one** type (the receiver). We need the behaviour to depend on **two** — the element's type and the visitor's type. `accept` is a first dispatch that lands us inside the concrete element, which is the only place the concrete type is known for free; it then makes a second dispatch by calling `visitor.visitAdd(this)`. Two chained single dispatches = dispatch on two types, with no `instanceof` anywhere. Without `accept`, the visitor would need a type-switch ladder in every single visitor class.

### Q2: "What's the biggest downside of Visitor?"
**Hint:** Adding a new *element type* means editing every existing visitor — and in JavaScript a forgotten one fails at runtime, not at build time. This is the **expression problem**: you can make operations cheap to add or types cheap to add, but not both. Visitor picks operations. Say this plainly and name the trade-off — that's the answer they want.

### Q3: "Give a real Node.js example of Visitor."
**Hint:** ESLint rules and Babel plugins. Both return an object keyed by AST node type (`CallExpression(node) {...}`) — that object *is* the ConcreteVisitor, and the framework's traverser plays the `accept` role. The ECMAScript node types are frozen by spec; the rules/plugins are infinite. Perfect fit.

### Q4: "How is Visitor different from Strategy?"
**Hint:** Strategy = one algorithm, one method, swapped by the caller — single dispatch. Visitor = a *family* of methods, one per element type, and the element picks which one runs — double dispatch. Strategy varies *how* one thing is done; Visitor varies *what is done to each kind of thing.*

### Q5: "How would you make Visitor safer against a forgotten node type?"
**Hint:** Three levels. (1) A base `Visitor` class whose methods `throw` — a missing override fails loudly instead of returning `undefined`. (2) TypeScript with a discriminated union + an exhaustiveness check (`const _never: never = node`) — the *compiler* then refuses to build if you add a node type and miss a visitor. (3) A `defaultVisit(node)` hook that visitors can override for a sensible fallback, so leaf-only visitors don't need all N methods.

---

## Practice exercise

### Extend the Expression Interpreter

Start from the `visitor.js` file above. Do these in order (~30-40 min):

**Part A — feel the cost of a new node type (this is the point of the exercise).**
1. Add a `SubtractNode` (with `left` and `right`) and a `NegateNode` (with a single `operand`).
2. Give each an `accept` method.
3. Now go and update **every** visitor: `EvalVisitor`, `PrintVisitor`, `CollectVarsVisitor`, `OptimizeVisitor`. Add `visitSubtract` and `visitNegate` to each, plus the `throw` stubs on the `Visitor` base class.
4. **Count the files you touched.** Write that number down. *That* is the price of Visitor, and you just paid it.

**Part B — feel the payoff of a new operation.**
5. Write a `DepthVisitor` that returns the height of the tree (a `NumberNode` has depth 1; an `AddNode` has `1 + max(depth(left), depth(right))`).
6. Write a `ToJsonVisitor` that turns the AST into a plain nested object like `{ type: 'add', left: {...}, right: {...} }`.
7. **Count the files you touched.** It should be **one each**. Compare to your Part A number.

**Part C — reflect (write 3 sentences, seriously).**
8. If this were a *drawing app* where new shapes ship every sprint but the only operations are `draw()` and `area()`, would you use Visitor? Why not? What would you do instead?

**Deliverable:** a working `visitor.js` printing eval/print/depth/json results for `-(x - 3) * (2 + 0)`, plus your two file counts and the three-sentence answer to Part C.

---

## Quick reference cheat sheet

- **Visitor** = add new operations to a class hierarchy **without modifying the classes**.
- **The one-liner:** move the operation out of the objects and into a separate visitor object.
- **`accept(visitor)`** is the *only* method your element classes need — and it's one line.
- **`accept` calls back:** `AddNode.accept(v)` → `v.visitAdd(this)`. Never anything else.
- **Double dispatch** = the running method depends on **both** the element's type and the visitor's type, achieved by chaining two ordinary single dispatches.
- **`accept` exists** to recover the element's concrete type without `instanceof`.
- **Cheap:** adding an operation = one new visitor class, zero edits.
- **Expensive:** adding an element type = edit **every** visitor. This is the killer trade-off.
- **The expression problem:** no pattern makes both axes cheap. Pick the one you'll actually use.
- **Use it when:** the structure is **stable** and the operations **keep multiplying** (ASTs, compilers, document trees, scene graphs, query builders).
- **Don't use it when:** the operations are stable and new types keep arriving — use plain methods on the classes.
- **Node.js in the wild:** ESLint rules, Babel plugins, TypeScript transformers — all visitors keyed by AST node type.
- **Safety net:** base-class methods that `throw`, or TypeScript exhaustiveness checks, so a forgotten node type fails loudly.
- **Not Strategy:** Strategy = one method, single dispatch. Visitor = N methods, double dispatch.
- **Best friend:** [Composite](./38-pattern-composite.md) builds the tree; Visitor operates on it.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [49 — Memento Pattern](./49-pattern-memento.md) — capture and restore state; the last of the "behaviour over a structure" patterns before this one |
| **Next** | [51 — Producer-Consumer Pattern](./51-pattern-producer-consumer.md) — decoupling work production from work consumption via a shared buffer |
| **Related** | [38 — Composite Pattern](./38-pattern-composite.md) — builds the tree that Visitor walks; the two are almost always used together |
| **Related** | [15 — SOLID: Open/Closed Principle](./15-solid-open-closed.md) — Visitor is the purest expression of "open to extension, closed to modification" |
