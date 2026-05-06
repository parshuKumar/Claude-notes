# 67 — Manipulating the DOM

## What is this?

**DOM manipulation** is the act of changing what the user sees on screen by modifying the DOM tree through JavaScript — updating text, changing styles, adding/removing CSS classes, showing/hiding elements, inserting new elements, and removing existing ones. Think of the DOM as a live blueprint of your webpage: manipulation is picking up a pen and redrawing parts of it while the user is watching, and the browser instantly reflects every change.

---

## Why does it matter?

Static HTML never changes after the server sends it. Every dynamic behaviour — showing a success message, toggling a menu, updating a cart count, adding a comment without refreshing the page — is DOM manipulation. It's the fundamental skill that makes web pages interactive.

---

## Changing text content

```js
const heading = document.querySelector("h1");

// textContent — plain text, no HTML parsing — ALWAYS prefer this for user data
heading.textContent = "New Heading";           // sets text
heading.textContent;                           // reads current text

// innerHTML — parses HTML string — use only for trusted content
heading.innerHTML = "<em>Italic Heading</em>"; // creates <em> element
heading.innerHTML;                             // returns HTML string including tags

// innerText — like textContent but layout-aware (slower, avoids hidden text)
heading.innerText = "Visible Heading";

// NEVER use innerHTML with user input — XSS vulnerability:
const userInput = '<img src=x onerror="stealCookies()">';
heading.innerHTML  = userInput;   // ✗ DANGEROUS — executes the script
heading.textContent = userInput;  // ✓ SAFE — displays the literal string
```

---

## Changing styles

### Inline styles via `.style`

```js
const box = document.querySelector(".box");

// Setting individual styles:
box.style.backgroundColor = "red";        // camelCase in JS, not background-color
box.style.width            = "200px";      // values must include units for lengths
box.style.fontSize         = "18px";
box.style.display          = "none";       // hide element
box.style.display          = "";           // reset to stylesheet default (remove inline style)

// Reading inline styles (only returns what's set inline — not CSS stylesheet values):
box.style.backgroundColor;   // "red" if set inline; "" if from stylesheet

// Reading the COMPUTED style (what the browser actually applies, including stylesheet):
const computed = window.getComputedStyle(box);
computed.backgroundColor;    // e.g. "rgb(255, 0, 0)" — final computed value
computed.fontSize;            // "18px"
computed.display;             // "block", "flex", etc.
computed.getPropertyValue("background-color");  // with hyphens works here

// Setting multiple styles at once via cssText (replaces ALL inline styles):
box.style.cssText = "width: 200px; height: 100px; background: blue;";
// CAREFUL — this overwrites any previously set inline styles on the element
```

### CSS custom properties (variables)

```js
// Set a CSS variable on an element (scopes to that element + descendants):
document.documentElement.style.setProperty("--primary-color", "#3498db");
document.documentElement.style.setProperty("--font-size-base", "16px");

// Read a CSS variable:
getComputedStyle(document.documentElement).getPropertyValue("--primary-color");
// → " #3498db"

// Remove a CSS variable:
document.documentElement.style.removeProperty("--primary-color");
```

---

## Changing classes — `classList`

`classList` is the modern, clean API for working with CSS classes. Prefer it over `className`.

```js
const card = document.querySelector(".card");

// ── Add / remove / toggle ──────────────────────────────────────
card.classList.add("active");              // add class
card.classList.add("featured", "pinned"); // add multiple classes
card.classList.remove("inactive");         // remove class (no error if not present)
card.classList.toggle("open");             // add if absent, remove if present
card.classList.toggle("open", true);       // force-add (second arg = boolean)
card.classList.toggle("open", false);      // force-remove

// ── Check ──────────────────────────────────────────────────────
card.classList.contains("active");         // true / false
card.classList.has("active");              // alias for contains (newer browsers)

// ── Replace ────────────────────────────────────────────────────
card.classList.replace("inactive", "active");  // replaces one class with another; returns boolean

// ── Inspect ────────────────────────────────────────────────────
card.classList.length;                     // number of classes
card.classList.item(0);                    // first class name
[...card.classList];                       // array of all class names
card.className;                            // "card active featured" — space-separated string

// ── vs className (old way) ─────────────────────────────────────
// WRONG — overwrites all existing classes:
card.className = "active";                 // removes "card" and all other classes!

// RIGHT with className — must read and rebuild:
card.className = card.className + " active";  // fragile

// RIGHT — use classList instead:
card.classList.add("active");              // preserves existing classes
```

---

## Changing attributes

```js
const link  = document.querySelector("a.nav-link");
const input = document.querySelector("input[name='email']");

// ── Read / write / remove attributes ──────────────────────────
link.getAttribute("href");                 // "/about"
link.setAttribute("href", "/new-path");    // change href
link.setAttribute("target", "_blank");    // add new attribute
link.removeAttribute("target");            // remove attribute
link.hasAttribute("disabled");             // false

// ── Common property shortcuts (faster than getAttribute) ───────
link.href;                                 // absolute URL: "https://example.com/about"
link.getAttribute("href");                 // original value: "/about" (relative)
input.value;                               // current typed value (live)
input.getAttribute("value");              // initial value from HTML
input.disabled = true;                     // sets disabled attribute AND property
input.type;                                // "email"
input.placeholder = "Enter email";

// ── data-* attributes — dataset API ───────────────────────────
// HTML: <article data-post-id="101" data-author-name="alice">
const article = document.querySelector("article");

article.dataset.postId;         // "101"   (camelCase JS ← kebab-case HTML)
article.dataset.authorName;     // "alice"
article.dataset.newProp = "x";  // adds data-new-prop="x" to element
delete article.dataset.newProp; // removes data-new-prop attribute
"postId" in article.dataset;    // true

// Always strings — convert if needed:
const postId = Number(article.dataset.postId);   // 101

// ── Boolean attributes ─────────────────────────────────────────
// Presence = true, absence = false (regardless of value)
input.setAttribute("disabled", "");     // disables (value doesn't matter)
input.setAttribute("disabled", "false");// STILL disabled! Presence = true
input.removeAttribute("disabled");      // removes — now enabled
input.disabled = false;                 // property way — removes the attribute
```

---

## Showing and hiding elements

```js
// Method 1 — display style (removes from layout flow)
element.style.display = "none";          // hidden, no space taken
element.style.display = "";              // restore to stylesheet default
element.style.display = "block";         // force show as block
element.style.display = "flex";          // show as flex container

// Method 2 — visibility (hidden but still takes up space)
element.style.visibility = "hidden";     // invisible, still occupies space
element.style.visibility = "visible";    // show again

// Method 3 — opacity (still takes space, can be transitioned with CSS)
element.style.opacity = "0";             // invisible but in layout
element.style.opacity = "1";             // fully visible

// Method 4 — hidden attribute (HTML5 — equivalent to display: none)
element.hidden = true;                   // hide
element.hidden = false;                  // show
element.setAttribute("hidden", "");      // same thing via attribute

// Method 5 — CSS class toggle (preferred — keeps JS out of style decisions)
element.classList.add("hidden");         // CSS: .hidden { display: none; }
element.classList.remove("hidden");      // show

// Method 6 — toggle a CSS class based on state
function showModal(modal, overlay) {
  modal.classList.add("modal--open");
  overlay.classList.add("overlay--visible");
  document.body.classList.add("scroll-locked");
}

function hideModal(modal, overlay) {
  modal.classList.remove("modal--open");
  overlay.classList.remove("overlay--visible");
  document.body.classList.remove("scroll-locked");
}
```

---

## Creating and inserting elements

```js
// ── Creating ───────────────────────────────────────────────────
const div = document.createElement("div");
div.className   = "notification";
div.textContent = "Action completed successfully";
div.dataset.type = "success";

// ── Inserting ──────────────────────────────────────────────────
parent.append(div);                      // insert as LAST child
parent.prepend(div);                     // insert as FIRST child
referenceEl.before(div);                 // insert BEFORE referenceEl (as sibling)
referenceEl.after(div);                  // insert AFTER referenceEl (as sibling)

// insertAdjacentElement positions (relative to referenceEl):
//   "beforebegin" — before the element itself
//   "afterbegin"  — just inside the element, before first child
//   "beforeend"   — just inside the element, after last child
//   "afterend"    — after the element itself
referenceEl.insertAdjacentElement("afterend", div);

// insertAdjacentHTML — efficient HTML string insertion:
list.insertAdjacentHTML("beforeend", `<li class="item">${escapeHtml(text)}</li>`);
// Only use with trusted or properly escaped content

// replaceWith — replace element with one or more others:
oldEl.replaceWith(newEl);
oldEl.replaceWith(el1, el2, "text string");

// replaceChildren — replace all children:
parent.replaceChildren();                // clears all children (better than innerHTML = "")
parent.replaceChildren(child1, child2);  // replace children with specific elements
```

---

## Removing elements

```js
// Modern — remove self:
element.remove();                        // removes element from DOM

// Legacy — requires parent reference:
parent.removeChild(child);

// Remove all children:
parent.innerHTML = "";                   // parses HTML — slower, destroys event listeners
parent.replaceChildren();                // modern — fastest, preferred
while (parent.firstChild) {              // explicit loop — safe fallback
  parent.removeChild(parent.firstChild);
}
```

---

## Moving elements

```js
// Appending an existing node MOVES it — does not copy:
const item = document.querySelector(".item");
const newParent = document.querySelector(".other-list");

newParent.append(item);    // item MOVED from old parent to newParent
// item is no longer in its original location

// To keep original AND insert copy:
newParent.append(item.cloneNode(true));    // deep clone → item stays in original place
```

---

## Example 1 — notification system

```js
class NotificationManager {
  #container;

  constructor(containerSelector = "#notifications") {
    this.#container = document.querySelector(containerSelector);
    if (!this.#container) {
      this.#container = document.createElement("div");
      this.#container.id = "notifications";
      this.#container.setAttribute("aria-live", "polite");
      document.body.appendChild(this.#container);
    }
  }

  show(message, type = "info", duration = 4000) {
    const toast = document.createElement("div");
    toast.className   = `toast toast--${type}`;
    toast.textContent = message;               // safe — no innerHTML
    toast.setAttribute("role", "alert");

    const closeBtn = document.createElement("button");
    closeBtn.textContent = "×";
    closeBtn.className   = "toast__close";
    closeBtn.setAttribute("aria-label", "Close notification");
    closeBtn.addEventListener("click", () => this.#dismiss(toast));

    toast.appendChild(closeBtn);
    this.#container.appendChild(toast);

    // Auto-dismiss:
    if (duration > 0) {
      setTimeout(() => this.#dismiss(toast), duration);
    }

    return toast;
  }

  #dismiss(toast) {
    toast.classList.add("toast--exiting");
    toast.addEventListener("animationend", () => toast.remove(), { once: true });
  }

  success(msg, duration) { return this.show(msg, "success", duration); }
  error(msg, duration)   { return this.show(msg, "error",   duration); }
  warn(msg, duration)    { return this.show(msg, "warn",    duration); }
}

const notify = new NotificationManager();
notify.success("Profile saved!");
notify.error("Failed to connect to server", 0);   // duration 0 = stays until closed
```

---

## Example 2 — real world: dynamic list with add / remove / reorder

```js
class TodoList {
  #container;
  #items = [];

  constructor(selector) {
    this.#container = document.querySelector(selector);
    this.#render();
  }

  add(text) {
    const item = { id: Date.now(), text, done: false };
    this.#items.push(item);
    this.#appendItemEl(item);
  }

  toggle(id) {
    const item = this.#items.find(i => i.id === id);
    if (!item) return;
    item.done = !item.done;

    const el = this.#container.querySelector(`[data-id="${id}"]`);
    el.classList.toggle("done", item.done);     // second arg = force state
    el.querySelector(".todo-text").style.textDecoration = item.done ? "line-through" : "";
  }

  remove(id) {
    this.#items = this.#items.filter(i => i.id !== id);
    this.#container.querySelector(`[data-id="${id}"]`)?.remove();
  }

  #render() {
    this.#container.replaceChildren();    // clear
    const fragment = document.createDocumentFragment();
    for (const item of this.#items) {
      fragment.appendChild(this.#createItemEl(item));
    }
    this.#container.appendChild(fragment);
  }

  #createItemEl(item) {
    const li = document.createElement("li");
    li.dataset.id = item.id;
    li.className  = `todo-item${item.done ? " done" : ""}`;

    const span = document.createElement("span");
    span.className   = "todo-text";
    span.textContent = item.text;                         // safe
    if (item.done) span.style.textDecoration = "line-through";

    const del = document.createElement("button");
    del.className   = "todo-delete";
    del.textContent = "Delete";
    del.dataset.action = "delete";

    li.append(span, del);
    return li;
  }

  #appendItemEl(item) {
    this.#container.appendChild(this.#createItemEl(item));
  }
}

const todos = new TodoList("#todo-list");

todos.add("Buy groceries");
todos.add("Write documentation");
todos.add("Deploy to production");

// Event delegation — one listener on the container handles all clicks:
document.querySelector("#todo-list").addEventListener("click", (e) => {
  const btn  = e.target.closest("[data-action]");
  const item = e.target.closest("[data-id]");
  if (!btn || !item) return;

  const id = Number(item.dataset.id);

  if (btn.dataset.action === "delete") todos.remove(id);
  if (btn.dataset.action === "toggle") todos.toggle(id);
});
```

---

## DOM manipulation performance tips

```js
// 1. Batch reads before writes — avoid layout thrashing
// WRONG — forces browser to calculate layout on every iteration (slow):
const boxes = document.querySelectorAll(".box");
boxes.forEach(box => {
  const width = box.offsetWidth;           // READ — triggers layout
  box.style.width = (width * 2) + "px";   // WRITE — invalidates layout
  // next iteration READ forces re-layout of the invalidated layout!
});

// RIGHT — batch all reads, then batch all writes:
const widths = [...boxes].map(box => box.offsetWidth);      // all READs first
boxes.forEach((box, i) => {
  box.style.width = (widths[i] * 2) + "px";                // then all WRITEs
});

// 2. Use DocumentFragment for multiple insertions
const fragment = document.createDocumentFragment();
data.forEach(item => {
  const el = document.createElement("li");
  el.textContent = item.name;
  fragment.appendChild(el);
});
list.appendChild(fragment);   // one DOM update

// 3. Use CSS class toggle instead of inline style per property
// WRONG — multiple style recalculations:
el.style.display    = "flex";
el.style.padding    = "16px";
el.style.background = "#fff";

// RIGHT — one class change → one style recalculation:
el.classList.add("card--expanded");   // define all styles in CSS

// 4. Cache DOM queries — don't re-query in loops
// WRONG:
for (const item of data) {
  document.querySelector(".list").appendChild(createEl(item));  // re-queries each time!
}

// RIGHT:
const list = document.querySelector(".list");  // query once
for (const item of data) {
  list.appendChild(createEl(item));
}
```

---

## Tricky things

### 1. `style.display = ""` vs `style.display = "block"`

```js
// Setting display to "" removes the inline style, falling back to the stylesheet
element.style.display = "none";    // hides
element.style.display = "";        // restores to what CSS says (could be flex, grid, etc.)
element.style.display = "block";   // FORCES block — may break flex/grid layouts

// Best practice: toggle a CSS class, let the stylesheet decide the display value
element.classList.add("hidden");    // CSS: .hidden { display: none; }
element.classList.remove("hidden"); // CSS restores the correct display value
```

### 2. `classList.toggle` second argument

```js
const card = document.querySelector(".card");

// Without second arg — toggles based on current state:
card.classList.toggle("active");        // adds if absent, removes if present

// With second arg (boolean) — force add or remove:
card.classList.toggle("active", true);   // always ADD — like .add() but returns boolean
card.classList.toggle("active", false);  // always REMOVE — like .remove() but returns boolean

// Useful when you have a state variable:
card.classList.toggle("selected", isSelected);  // reflects state directly
```

### 3. `element.remove()` does not free memory if you hold a reference

```js
const el = document.querySelector(".notification");
el.remove();   // removed from DOM tree

// BUT — el variable still holds a reference to the object in memory
// It's a "detached" DOM node — not in the tree, but still in memory
console.log(el.textContent);   // still works on the detached node

// The node (and all its children/event listeners) will be GC'd
// ONLY when all references are gone:
// el = null;  ← only now can GC collect it
```

### 4. Reading `offsetWidth` / `offsetHeight` forces layout ("reflow")

```js
// These properties force the browser to compute layout immediately:
element.offsetWidth;     // triggers reflow
element.offsetHeight;    // triggers reflow
element.clientWidth;     // triggers reflow
element.getBoundingClientRect();  // triggers reflow

// Safe to read (no reflow):
element.style.width;          // only inline styles
window.getComputedStyle(el);  // batches reads without extra reflows when done once
```

### 5. `innerHTML = ""` vs `replaceChildren()`

```js
// innerHTML = "" — re-parses the empty string as HTML (minor overhead, destroys listeners)
parent.innerHTML = "";         // clears children

// replaceChildren() — modern, direct, no parsing (preferred)
parent.replaceChildren();      // clears all children efficiently

// Both remove all child DOM nodes and their event listeners
// Prefer replaceChildren() for clarity and performance
```

### 6. Setting `value` vs setting the `value` attribute

```js
const input = document.querySelector("input");

// Sets the PROPERTY (what user sees now):
input.value = "new value";          // updates displayed value

// Sets the HTML ATTRIBUTE (default value — reset target):
input.setAttribute("value", "default");  // changes what shows after form.reset()

// After user types: input.value ≠ input.getAttribute("value")
```

---

## Common mistakes

### Mistake 1 — Using `innerHTML` with user data (XSS)

```js
// WRONG — XSS vulnerability
const commentText = '<script>stealData()</script>';
commentDiv.innerHTML = commentText;    // executes the script!

// RIGHT — textContent is always safe for user content
commentDiv.textContent = commentText;  // displays the literal string, no script execution
```

### Mistake 2 — `className =` overwrites all existing classes

```js
// WRONG — removes "card" and "featured" classes that were already there
const card = document.querySelector(".card.featured");
card.className = "active";    // now only has "active" — lost "card" and "featured"!

// RIGHT — add/remove specific classes
card.classList.add("active");         // preserves existing classes
```

### Mistake 3 — Layout thrashing (read/write interleaving in a loop)

```js
// WRONG — each offsetHeight read forces browser to recalculate layout
// because the previous style.height write invalidated it
elements.forEach(el => {
  const h = el.offsetHeight;         // READ  → forces layout
  el.style.height = (h + 10) + "px"; // WRITE → invalidates layout
});

// RIGHT — batch reads, then batch writes
const heights = elements.map(el => el.offsetHeight);  // all reads
elements.forEach((el, i) => {
  el.style.height = (heights[i] + 10) + "px";          // all writes
});
```

---

## Practice exercises

### Exercise 1 — easy

Given this HTML:
```html
<div id="profile-card">
  <img id="avatar" src="default.png" alt="User avatar">
  <h2 id="username">Guest</h2>
  <p id="bio">No bio yet.</p>
  <span id="role-badge" class="badge">viewer</span>
  <button id="edit-btn">Edit</button>
</div>
```

Write JavaScript to:
1. Change the `#username` text to `"Alice Chen"`
2. Change `#bio` to `"Full-stack developer at Example Corp."`
3. Set `#avatar`'s `src` to `"alice.png"` and `alt` to `"Alice Chen's avatar"` using `setAttribute`
4. Replace the `#role-badge` text with `"admin"` and add class `"badge--admin"` using `classList`
5. Set a `data-user-id` of `"42"` on the `#profile-card` using `dataset`
6. Hide the `#edit-btn` by setting `hidden = true`
7. Add an inline style to `#role-badge`: `background-color: #e74c3c`
8. Read the computed `display` value of `#profile-card` using `getComputedStyle`

```js
// Write your code here
```

### Exercise 2 — medium

Build a `Modal` class that creates and manages a modal dialog entirely through DOM manipulation:

```js
class Modal {
  // constructor(options) — options: { title, content, onConfirm, onCancel }
  // Creates these elements programmatically (no innerHTML for content):
  //   .modal-overlay
  //     .modal
  //       .modal__header  → h2 with title
  //       .modal__body    → paragraph with content text
  //       .modal__footer  → "Cancel" and "Confirm" buttons
  //
  // open()  — appends overlay to document.body, adds "modal--open" class
  // close() — removes "modal--open" class, removes from DOM after transition (300ms)
  // setTitle(text)   — update the title
  // setContent(text) — update the body text
}

// Usage:
const confirmDelete = new Modal({
  title:     "Delete User",
  content:   "Are you sure you want to delete this user? This cannot be undone.",
  onConfirm: () => { console.log("User deleted"); },
  onCancel:  () => { console.log("Cancelled"); },
});

document.querySelector("#delete-btn").addEventListener("click", () => {
  confirmDelete.open();
});
```

Requirements:
- All text set via `textContent` (safe)
- Pressing Escape key closes the modal
- Clicking the overlay (but not the modal itself) closes it
- `close()` adds a `modal--exiting` class, then removes from DOM after 300ms timeout

```js
// Write your code here
```

### Exercise 3 — hard

Build a `DataTable` class that renders a dynamic, sortable, filterable table from an array of objects:

```js
class DataTable {
  // constructor(container, data, options)
  //   options: { columns, pageSize = 10, sortable = true }
  //   columns: [{ key, label, render? }]
  //   render: optional fn(value, row) → string (safe: must use textContent, not innerHTML)
  //
  // Methods:
  //   render()          — initial render
  //   sort(key, dir)    — sort by column key asc/desc; update column header classes
  //   filter(query)     — show only rows where any cell textContent includes query
  //   page(n)           — go to page n; render only that slice of data
  //   get visibleCount() — number of currently visible rows
  //   get totalPages()   — total pages based on pageSize
}

// Usage:
const table = new DataTable(
  document.querySelector("#table-container"),
  [
    { id: 1, name: "Alice",  age: 30, role: "admin" },
    { id: 2, name: "Bob",    age: 25, role: "viewer" },
    { id: 3, name: "Carol",  age: 35, role: "editor" },
    // ... more rows
  ],
  {
    columns: [
      { key: "id",   label: "ID" },
      { key: "name", label: "Name" },
      { key: "age",  label: "Age" },
      { key: "role", label: "Role", render: (v) => v.toUpperCase() },
    ],
    pageSize: 2,
    sortable: true,
  }
);

table.render();
table.sort("age", "asc");
table.filter("a");          // shows Alice, Carol (contain "a")
console.log(table.visibleCount);   // 2
table.page(1);
```

All content must use `textContent` (safe from XSS). Use `DocumentFragment` for batch row insertion. Sort header clicks should toggle asc/desc and update a `data-sort-dir` attribute on the `<th>`.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Task | Method / Property |
|---|---|
| Set text safely | `el.textContent = "text"` |
| Set HTML | `el.innerHTML = "<b>bold</b>"` — never with user input |
| Get text | `el.textContent` |
| Add CSS class | `el.classList.add("name")` |
| Remove CSS class | `el.classList.remove("name")` |
| Toggle CSS class | `el.classList.toggle("name")` |
| Force add/remove | `el.classList.toggle("name", bool)` |
| Check class | `el.classList.contains("name")` |
| Replace class | `el.classList.replace("old", "new")` |
| All classes as array | `[...el.classList]` |
| Set inline style | `el.style.backgroundColor = "red"` (camelCase) |
| Remove inline style | `el.style.backgroundColor = ""` (empty string) |
| Computed style | `getComputedStyle(el).propertyName` |
| CSS variable | `el.style.setProperty("--name", val)` |
| Read attribute | `el.getAttribute("name")` |
| Write attribute | `el.setAttribute("name", "val")` |
| Remove attribute | `el.removeAttribute("name")` |
| Check attribute | `el.hasAttribute("name")` |
| `data-*` read | `el.dataset.myKey` |
| `data-*` write | `el.dataset.myKey = "val"` |
| Hide (no space) | `el.hidden = true` or `el.style.display = "none"` |
| Show | `el.hidden = false` or `el.style.display = ""` |
| Create element | `document.createElement("div")` |
| Insert at end | `parent.append(el)` |
| Insert at start | `parent.prepend(el)` |
| Insert before | `ref.before(el)` |
| Insert after | `ref.after(el)` |
| Replace element | `old.replaceWith(new)` |
| Remove element | `el.remove()` |
| Clear children | `parent.replaceChildren()` |
| Batch insert | `DocumentFragment` + one `append` |
| Move element | `parent.append(existingEl)` — moves, does not copy |
| Copy element | `el.cloneNode(true)` — deep clone |

---

## Connected topics

- **65 — How the DOM works** — the tree structure you're manipulating
- **66 — Selecting elements** — how to get a reference to the element first
- **68 — Events** — triggering manipulations in response to user actions
- **69 — Event delegation** — efficient pattern for dynamically added elements
