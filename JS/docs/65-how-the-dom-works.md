# 65 — How the DOM Works

## What is this?

The **DOM** (Document Object Model) is the browser's live, in-memory representation of an HTML page as a **tree of objects**. When a browser loads an HTML file, it parses the markup and builds a tree where every element, attribute, and piece of text becomes a **node** — a JavaScript object you can read, modify, create, or delete. Think of the DOM like a family tree: the document is the ancestor at the top, `<html>` is the first child, `<head>` and `<body>` are siblings, and all elements nested inside are their descendants.

The key word is **live** — when you change the DOM tree through JavaScript, the browser immediately updates what the user sees on screen.

---

## Why does it matter?

Every interactive web page — forms, modals, dropdowns, tabs, infinite scrolls — works by reading from and writing to the DOM. Even if you use a framework like React or Vue, they both ultimately call the same DOM APIs under the hood. Understanding the DOM tree model is the prerequisite for all browser JavaScript.

---

## The document tree — structure

```
document
└── <html>                (document.documentElement)
    ├── <head>
    │   ├── <meta>
    │   ├── <title>
    │   │   └── "My Page"    ← Text Node
    │   └── <link>
    └── <body>
        ├── <header>
        │   └── <h1>
        │       └── "Hello"  ← Text Node
        ├── <main>
        │   ├── <section id="users">
        │   │   ├── <h2>Users</h2>
        │   │   └── <ul>
        │   │       ├── <li class="user">Alice</li>
        │   │       └── <li class="user">Bob</li>
        │   └── <section id="posts">
        └── <footer>
            └── <!-- comment --> ← Comment Node
```

Everything in this tree is a **Node**. There are different types.

---

## Node types

```js
// Every node has a nodeType property (a number):
Node.ELEMENT_NODE                // 1  — <div>, <p>, <span> etc.
Node.TEXT_NODE                   // 3  — text content inside elements
Node.COMMENT_NODE                // 8  — <!-- comments -->
Node.DOCUMENT_NODE               // 9  — the document object itself
Node.DOCUMENT_TYPE_NODE          // 10 — <!DOCTYPE html>
Node.DOCUMENT_FRAGMENT_NODE      // 11 — lightweight container (off-screen sub-tree)

// Check what type a node is:
document.nodeType;               // 9  — DOCUMENT_NODE
document.body.nodeType;          // 1  — ELEMENT_NODE

const h1 = document.querySelector("h1");
h1.nodeType;                     // 1  — ELEMENT_NODE
h1.firstChild.nodeType;          // 3  — TEXT_NODE (the text "Hello" inside <h1>)
```

**Element nodes** are what you interact with most — they correspond to HTML tags. But **text nodes** are everywhere too: the whitespace between tags and the actual text content are all text nodes.

---

## Node vs Element — the distinction

```
Node (supertype — ALL nodes)
├── Element (nodeType 1 — HTML tags like <div>, <p>)
│   ├── HTMLElement
│   │   ├── HTMLDivElement
│   │   ├── HTMLInputElement
│   │   ├── HTMLButtonElement
│   │   └── ... (one class per HTML tag)
│   └── SVGElement
├── Text (nodeType 3 — text content)
├── Comment (nodeType 8)
└── Document (nodeType 9)
```

**In practice:** almost everything you work with is an `Element` (or its subclass). Properties like `.innerHTML`, `.classList`, `.style`, `.tagName` are on `Element`. Properties like `.nodeType`, `.nodeName`, `.parentNode`, `.childNodes` are on `Node` (shared by all).

---

## How it works — key node relationships

```js
const list = document.querySelector("ul");

// ── Navigating UP the tree ─────────────────────────────────────
list.parentNode;             // the element that contains list
list.parentElement;          // same, but returns null if parent is not an Element

// ── Navigating DOWN the tree ───────────────────────────────────
list.childNodes;             // NodeList of ALL children (includes text/comment nodes)
list.children;               // HTMLCollection of ELEMENT children only (no text/comment nodes)
list.firstChild;             // first child node (might be a text node — whitespace!)
list.lastChild;              // last child node
list.firstElementChild;      // first child that is an Element
list.lastElementChild;       // last child that is an Element

// ── Navigating SIDEWAYS (siblings) ────────────────────────────
list.nextSibling;            // next sibling node (might be text node)
list.previousSibling;        // previous sibling node
list.nextElementSibling;     // next sibling that is an Element
list.previousElementSibling; // previous sibling that is an Element

// ── Information ────────────────────────────────────────────────
list.tagName;                // "UL" (always uppercase for HTML elements)
list.nodeName;               // "UL" for elements, "#text" for text, "#comment" for comments
list.nodeType;               // 1 (ELEMENT_NODE)
```

---

## The document object

```js
// The entry point to the entire DOM:
document                           // the Document object (nodeType 9)

document.documentElement           // <html> element
document.head                      // <head> element
document.body                      // <body> element
document.title                     // page title (read/write)
document.URL                       // current URL
document.domain                    // current domain
document.characterSet              // "UTF-8"
document.contentType               // "text/html"
document.readyState                // "loading", "interactive", or "complete"

// Check when DOM is ready:
document.addEventListener("DOMContentLoaded", () => {
  console.log("DOM is built — safe to query elements");
});

window.addEventListener("load", () => {
  console.log("Page fully loaded including images, scripts");
});
```

---

## Example 1 — reading the DOM tree

```html
<!-- index.html -->
<ul id="user-list">
  <li class="user active">Alice</li>
  <li class="user">Bob</li>
  <li class="user">Carol</li>
</ul>
```

```js
const ul = document.getElementById("user-list");

// Children
console.log(ul.children.length);          // 3 — three <li> elements
console.log(ul.childNodes.length);        // 7 — 3 <li> + 4 text nodes (whitespace between them)

// First element child
const first = ul.firstElementChild;
console.log(first.tagName);              // "LI"
console.log(first.textContent);          // "Alice"
console.log(first.className);            // "user active"

// Iterate all element children
for (const li of ul.children) {
  console.log(li.textContent);
  // Alice
  // Bob
  // Carol
}

// Walk the tree manually (all node types):
function walkTree(node, depth = 0) {
  const indent  = "  ".repeat(depth);
  const type    = node.nodeType === 1 ? `<${node.tagName.toLowerCase()}>` :
                  node.nodeType === 3 ? `"${node.textContent.trim()}"` :
                  `[nodeType ${node.nodeType}]`;

  if (node.textContent.trim() || node.nodeType === 1) {
    console.log(indent + type);
  }

  for (const child of node.childNodes) {
    walkTree(child, depth + 1);
  }
}

walkTree(ul);
// <ul>
//   <li>
//     "Alice"
//   <li>
//     "Bob"
//   <li>
//     "Carol"
```

---

## Example 2 — real world: building a DOM tree programmatically

```js
// Building a user card without innerHTML (safe — no XSS risk)

function createUserCard(user) {
  const card  = document.createElement("article");
  card.className = "user-card";
  card.dataset.userId = user.id;     // sets data-user-id attribute

  const avatar = document.createElement("img");
  avatar.src  = user.avatarUrl;
  avatar.alt  = `${user.name}'s avatar`;
  avatar.className = "user-card__avatar";

  const info  = document.createElement("div");
  info.className = "user-card__info";

  const name  = document.createElement("h2");
  name.textContent = user.name;      // safe — no HTML parsing

  const email = document.createElement("p");
  email.textContent = user.email;

  const badge = document.createElement("span");
  badge.className = `badge badge--${user.role}`;
  badge.textContent = user.role;

  // Assemble:
  info.append(name, email, badge);
  card.append(avatar, info);

  return card;
}

// Use DocumentFragment to batch-insert without repeated reflow:
function renderUserList(users, container) {
  const fragment = document.createDocumentFragment();

  for (const user of users) {
    fragment.appendChild(createUserCard(user));
  }

  container.innerHTML = "";    // clear existing content
  container.appendChild(fragment);   // single DOM update — one reflow
}

renderUserList([
  { id: 1, name: "Alice", email: "alice@example.com", role: "admin",   avatarUrl: "/avatars/alice.png" },
  { id: 2, name: "Bob",   email: "bob@example.com",   role: "viewer",  avatarUrl: "/avatars/bob.png" },
], document.getElementById("user-list"));
```

---

## Creating, inserting, removing nodes

```js
// ── Creating ───────────────────────────────────────────────────
const div  = document.createElement("div");         // element
const text = document.createTextNode("Hello");      // text node
const frag = document.createDocumentFragment();     // off-screen container

// ── Setting content ────────────────────────────────────────────
div.textContent = "Safe text — no HTML parsed";     // safe
div.innerHTML   = "<strong>Bold</strong>";           // parses HTML — XSS risk if user input!

// ── Inserting ──────────────────────────────────────────────────
parent.appendChild(child);                  // append as last child (legacy)
parent.append(child1, child2, "text");      // modern — multiple args, accepts strings
parent.prepend(child);                      // insert as first child
parent.before(newNode);                     // insert before parent (as sibling)
parent.after(newNode);                      // insert after parent (as sibling)
referenceNode.insertAdjacentElement("beforebegin", el); // before referenceNode
referenceNode.insertAdjacentElement("afterbegin",  el); // first child of referenceNode
referenceNode.insertAdjacentElement("beforeend",   el); // last child of referenceNode
referenceNode.insertAdjacentElement("afterend",    el); // after referenceNode

// insertAdjacentHTML — efficient HTML string insertion (no full re-parse):
list.insertAdjacentHTML("beforeend", "<li>New item</li>");

// ── Removing ───────────────────────────────────────────────────
element.remove();                           // modern — remove self
parent.removeChild(child);                  // legacy — requires parent reference

// ── Cloning ────────────────────────────────────────────────────
const clone      = element.cloneNode(false);  // shallow — element only, no children
const deepClone  = element.cloneNode(true);   // deep — element + all descendants
```

---

## Attributes vs properties

There is an important distinction between **HTML attributes** (what's in the markup) and **DOM properties** (the live JavaScript values on the element object):

```js
// HTML: <input type="text" value="initial" id="username">
const input = document.querySelector("#username");

// Attributes — always strings, reflect the original HTML:
input.getAttribute("value");       // "initial" — stays "initial" even after user types
input.setAttribute("value", "new");
input.removeAttribute("placeholder");
input.hasAttribute("disabled");    // false

// Properties — live JavaScript values, change as user interacts:
input.value;                       // current value — changes as user types
input.id;                          // "username"
input.type;                        // "text"
input.disabled;                    // false (boolean, not string)
input.checked;                     // boolean for checkboxes

// Most common pairs where attribute ≠ property:
// attribute "class"   → property .className
// attribute "for"     → property .htmlFor (on <label>)
// attribute "checked" → property .checked (reflects current state)
// attribute "value"   → property .value   (reflects current user input)

// data-* attributes — stored in dataset:
// HTML: <div data-user-id="42" data-role="admin">
div.dataset.userId;                // "42" (note: camelCase in JS, kebab-case in HTML)
div.dataset.role;                  // "admin"
div.dataset.newProp = "hello";     // adds data-new-prop attribute to the HTML
```

---

## `innerHTML` vs `textContent` vs `innerText`

```js
// Given: <p id="msg">Hello <strong>World</strong></p>
const p = document.querySelector("#msg");

p.innerHTML;        // "Hello <strong>World</strong>"  — includes HTML tags
p.textContent;      // "Hello World"                   — plain text, ignores tags
p.innerText;        // "Hello World"                   — like textContent but layout-aware

// Setting:
p.innerHTML   = "<em>New content</em>";  // parsed as HTML — creates <em> element
p.textContent = "<em>New content</em>";  // NOT parsed — displays literal "<em>New content</em>"

// Security:
// NEVER do: element.innerHTML = userInput  — XSS attack vector!
// ALWAYS do: element.textContent = userInput  — safe, no HTML parsing

// innerText vs textContent:
// textContent — returns ALL text including hidden elements; does not trigger layout
// innerText   — returns VISIBLE text only; triggers layout (slower)
// Use textContent in almost all cases
```

---

## DocumentFragment — batching DOM updates

```js
// Problem: adding 1000 items one by one causes 1000 reflows (slow)
for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  list.appendChild(li);   // 1000 DOM mutations → 1000 layout recalculations
}

// Solution: DocumentFragment — off-screen container, attached in ONE operation
const fragment = document.createDocumentFragment();

for (let i = 0; i < 1000; i++) {
  const li = document.createElement("li");
  li.textContent = `Item ${i}`;
  fragment.appendChild(li);  // no reflow — fragment is off-screen
}

list.appendChild(fragment);  // ONE DOM mutation → ONE reflow
// After appending, the fragment is empty — the children moved to list
```

---

## Live vs static collections

```js
// HTMLCollection and NodeList can be LIVE (auto-updates) or STATIC (snapshot)

// LIVE — reflects current DOM changes:
const liveItems  = document.getElementsByClassName("item");  // HTMLCollection — live
const liveAll    = document.getElementsByTagName("li");       // HTMLCollection — live

// STATIC — snapshot at time of call (doesn't update):
const staticList = document.querySelectorAll(".item");   // NodeList — static

// Demonstration:
const live   = document.getElementsByClassName("todo");
const static_ = document.querySelectorAll(".todo");

console.log(live.length);    // 3
console.log(static_.length); // 3

// Add a new element with class "todo":
const newEl = document.createElement("li");
newEl.className = "todo";
document.querySelector("ul").appendChild(newEl);

console.log(live.length);    // 4 — updated automatically!
console.log(static_.length); // 3 — still the old snapshot

// For iteration safety, prefer static (querySelectorAll) or convert to Array:
Array.from(liveItems).forEach(item => { /* safe to iterate */ });
[...liveItems].forEach(item => { /* same */ });
```

---

## Tricky things

### 1. `childNodes` includes text nodes (whitespace)

```js
// HTML (with whitespace between tags):
// <ul>
//   <li>Alice</li>
//   <li>Bob</li>
// </ul>

const ul = document.querySelector("ul");

ul.childNodes.length;        // 5: "\n  ", <li>, "\n  ", <li>, "\n"
ul.children.length;          // 2: only the <li> elements

// Fix: always use .children (elements only) or .querySelectorAll() when you want elements
```

### 2. `firstChild` is likely a text node

```js
// Given: <div>\n  <p>Hello</p>\n</div>
const div = document.querySelector("div");

div.firstChild;            // Text node: "\n  " (the newline + spaces)
div.firstElementChild;     // <p> element — what you actually want
```

### 3. Moving a node removes it from its current position

```js
const item = document.querySelector(".item");
const newParent = document.querySelector(".new-list");

// Appending a node that already exists MOVES it — does not copy:
newParent.appendChild(item);   // item is now in newParent, removed from original parent
// To keep the original: use cloneNode(true) first
newParent.appendChild(item.cloneNode(true));
```

### 4. `innerHTML` completely re-creates children — destroys event listeners

```js
const ul = document.querySelector("ul");

// Event listener attached to an existing <li>:
ul.firstElementChild.addEventListener("click", handler);

// Now replace the list content:
ul.innerHTML = "<li>New item</li>";
// The original <li> is DESTROYED and a new one created
// The event listener is GONE — it was on the old element

// Fix: use DOM methods (createElement/append) to preserve listeners
// OR use event delegation on the parent (topic 69)
```

### 5. Querying elements that don't exist yet

```js
// Script runs before DOM is ready:
const btn = document.querySelector("#my-button");  // null — element not parsed yet
btn.addEventListener("click", handler);            // TypeError: Cannot read properties of null

// Fix 1: place <script> at the bottom of <body>
// Fix 2: wrap in DOMContentLoaded
document.addEventListener("DOMContentLoaded", () => {
  const btn = document.querySelector("#my-button");  // now it exists
  btn.addEventListener("click", handler);
});
// Fix 3: use defer attribute on <script> tag
// <script src="app.js" defer></script>
```

### 6. `NodeList` from `querySelectorAll` is not an Array

```js
const items = document.querySelectorAll(".item");

items.map(x => x);          // TypeError: items.map is not a function
items.filter(x => x);       // TypeError

// Fix: convert to array
Array.from(items).map(x => x.textContent);
[...items].map(x => x.textContent);
```

---

## Common mistakes

### Mistake 1 — Using `innerHTML` for user-generated content (XSS)

```js
// WRONG — XSS vulnerability
const userInput = '<img src=x onerror="alert(document.cookie)">';
div.innerHTML = userInput;   // executes the script!

// RIGHT — always use textContent for user-provided text
div.textContent = userInput;  // displays the literal string safely
// Or use DOMPurify library if you must render HTML from user input:
// div.innerHTML = DOMPurify.sanitize(userInput);
```

### Mistake 2 — Querying inside a loop with `document.querySelector` instead of relative query

```js
// WRONG — expensive: searches the entire document each time
for (const row of rows) {
  const cell = document.querySelector(".price");  // searches from document root!
}

// RIGHT — search relative to the known parent element
for (const row of rows) {
  const cell = row.querySelector(".price");  // only searches inside row
}
```

### Mistake 3 — Forgetting that appending an existing node moves it

```js
// WRONG assumption — thinks it's copying
const sidebar = document.querySelector(".sidebar");
const widget  = document.querySelector(".widget");

sidebar.appendChild(widget);
// widget is now in sidebar AND no longer in its original location

// RIGHT — clone if you need it in both places
sidebar.appendChild(widget.cloneNode(true));
```

---

## Practice exercises

### Exercise 1 — easy

Given this HTML:
```html
<div id="card">
  <h2 class="title">Product</h2>
  <p class="description">A great product.</p>
  <span class="price">$10.00</span>
</div>
```

Write JavaScript (no `querySelector` — use only DOM traversal) to:
1. Access the `#card` div using `getElementById`
2. Get its first element child
3. Get the last element child
4. Log the `textContent` of each child element using a `for...of` loop over `.children`
5. Log the `tagName` and `nodeType` of every node in `childNodes` (including text nodes)

```js
// Write your code here
```

### Exercise 2 — medium

Build a `createTable(data)` function that takes an array of objects and builds a full HTML table using DOM methods (no `innerHTML`):

```js
const users = [
  { name: "Alice", age: 30, role: "admin" },
  { name: "Bob",   age: 25, role: "viewer" },
  { name: "Carol", age: 35, role: "editor" },
];

// Creates:
// <table>
//   <thead><tr><th>name</th><th>age</th><th>role</th></tr></thead>
//   <tbody>
//     <tr><td>Alice</td><td>30</td><td>admin</td></tr>
//     ...
//   </tbody>
// </table>

const table = createTable(users);
document.body.appendChild(table);
```

Requirements:
- Column headers taken from `Object.keys(data[0])`
- Use `DocumentFragment` for the rows
- All content set via `textContent` (not innerHTML)
- Table has a `data-rows` attribute set to `users.length`

```js
// Write your code here
```

### Exercise 3 — hard

Build a `VirtualList` class that efficiently renders a large list (1000+ items) using DOM:

```js
class VirtualList {
  constructor(container, items) { /* ... */ }

  render()                    // initial render — builds the full list using DocumentFragment
  addItem(item)               // append new item (uses insertAdjacentHTML)
  removeItem(index)           // remove item at index
  updateItem(index, newText)  // update text of item at index using textContent (safe)
  filter(predicate)           // hide items where predicate(item) returns false (toggle a CSS class)
  sort(compareFn)             // re-sort the DOM items by re-appending them in sorted order
  get count()                 // returns number of VISIBLE items (not hidden ones)
}

// Usage:
const items = Array.from({ length: 1000 }, (_, i) => `Item ${i + 1}`);
const list  = new VirtualList(document.querySelector("#list"), items);

list.render();
list.addItem("Item 1001");
list.updateItem(0, "First Item (updated)");
list.filter(item => item.includes("5"));   // show only items containing "5"
console.log(list.count);                   // number of visible items
list.removeItem(0);
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| `document` | Root of the DOM — access via global `document` |
| `document.documentElement` | `<html>` |
| `document.head` / `.body` | `<head>` / `<body>` |
| Node types | 1=Element, 3=Text, 8=Comment, 9=Document, 11=Fragment |
| `.nodeType` | Numeric type of any node |
| `.nodeName` | Tag name for elements ("DIV"), "#text", "#comment" |
| `.tagName` | Uppercase tag name — elements only ("DIV", "LI") |
| `.children` | Element children only (HTMLCollection, live) |
| `.childNodes` | ALL children including text/comment (NodeList, live) |
| `.firstElementChild` / `.lastElementChild` | First/last element child |
| `.nextElementSibling` / `.previousElementSibling` | Adjacent element siblings |
| `.parentNode` / `.parentElement` | Parent node / element |
| `createElement(tag)` | Create a new element node |
| `createTextNode(str)` | Create a text node |
| `createDocumentFragment()` | Off-screen container for batch inserts |
| `element.append(...)` | Add to end (multiple args, strings OK) |
| `element.prepend(...)` | Add to start |
| `element.before()` / `.after()` | Insert before/after self |
| `element.remove()` | Remove from DOM |
| `cloneNode(true)` | Deep copy of element + all children |
| `innerHTML` | Get/set HTML string — XSS risk with user input |
| `textContent` | Get/set plain text — safe |
| `innerText` | Like textContent but layout-aware (slower) |
| `getAttribute` / `setAttribute` | HTML attribute (string) |
| `dataset.key` | Access `data-key` attribute |
| `querySelectorAll` → static | NodeList — snapshot, not live |
| `getElementsByClassName` → live | HTMLCollection — updates with DOM |
| `Array.from(nodeList)` | Convert NodeList to array for map/filter |
| `DOMContentLoaded` | DOM is ready; fires before images load |
| `load` | Entire page (images, scripts) loaded |

---

## Connected topics

- **66 — Selecting elements** — `querySelector`, `querySelectorAll`, `getElementById` — how to target specific nodes
- **67 — Manipulating the DOM** — changing text, styles, classes on nodes
- **68 — Events** — attaching user interaction handlers to DOM nodes
