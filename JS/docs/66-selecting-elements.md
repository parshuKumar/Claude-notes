# 66 — Selecting Elements

## What is this?

**Selecting elements** is how you get a reference to one or more DOM nodes from JavaScript so you can read or modify them. The browser gives you several methods — some old (targeting by ID or class name), some modern (CSS selector–based) — that search the document tree and return the matching element(s). Think of it like a search box for your HTML: you describe what you're looking for (an ID, a class, a CSS selector), and the browser hands you back the matching node(s).

---

## Why does it matter?

You cannot do anything with the DOM without first selecting the element you want to work with. Every interactive feature — updating text, showing/hiding elements, reading form values, adding animations — starts with a selection. Knowing which method to reach for, what each returns, and the performance differences makes your code faster and more reliable.

---

## Syntax overview

```js
// By ID — returns one Element or null
document.getElementById("user-card")

// By class name — returns live HTMLCollection
document.getElementsByClassName("btn")

// By tag name — returns live HTMLCollection
document.getElementsByTagName("li")

// By CSS selector — returns FIRST matching Element or null
document.querySelector(".card .title")

// By CSS selector — returns static NodeList of ALL matches
document.querySelectorAll("ul > li.active")

// By name attribute (forms) — returns live NodeList
document.getElementsByName("email")

// Scoped to a specific element (search within, not whole document):
parentElement.querySelector(".child-class")
parentElement.querySelectorAll("td")
```

---

## How each method works

### `getElementById`

```js
// Returns the ONE element with that exact id, or null
// id must be unique in the document
const card = document.getElementById("user-card");

card;                  // HTMLElement or null
card.id;               // "user-card"

// No "#" prefix — it's just the raw id string:
document.getElementById("user-card");    // ✓ correct
document.getElementById("#user-card");   // ✗ wrong — returns null
```

**Performance:** This is the fastest selector — browsers keep a direct lookup table for IDs. Prefer it when you have an ID.

---

### `querySelector`

```js
// Returns the FIRST element matching the CSS selector, or null
// Accepts any valid CSS selector

// By ID (same as getElementById but slower):
document.querySelector("#user-card");

// By class:
document.querySelector(".btn");
document.querySelector(".btn.primary");   // element with both classes

// By tag:
document.querySelector("h1");
document.querySelector("section > h2");   // h2 that is a direct child of section

// By attribute:
document.querySelector("[data-role='admin']");
document.querySelector("input[type='email']");
document.querySelector("input:not([disabled])");

// Compound (complex CSS selectors work):
document.querySelector("nav ul li:first-child a");
document.querySelector(".card:nth-child(2) .price");

// Returns null if nothing found — always check!
const btn = document.querySelector("#submit-btn");
if (btn) {
  btn.addEventListener("click", handleSubmit);
}
```

---

### `querySelectorAll`

```js
// Returns a STATIC NodeList of ALL matching elements
// Same selector support as querySelector

const items = document.querySelectorAll(".todo-item");

items.length;           // number of matches

// NodeList is NOT an array — cannot use .map(), .filter() directly:
items.map(x => x);      // TypeError!

// Convert first:
Array.from(items).map(item => item.textContent);
[...items].map(item => item.textContent);

// OR use forEach (NodeList has forEach natively):
items.forEach(item => {
  item.classList.add("processed");
});

// ── Common patterns ───────────────────────────────────────────
// Select all elements with data attribute:
document.querySelectorAll("[data-expandable]");

// Select all checked checkboxes:
document.querySelectorAll("input[type='checkbox']:checked");

// Select all visible inputs (not hidden):
document.querySelectorAll("input:not([type='hidden'])");

// Select elements by multiple classes:
document.querySelectorAll(".card.featured");

// Select direct children only (vs all descendants):
document.querySelectorAll(".list > .item");   // only direct children
document.querySelectorAll(".list .item");      // all descendants
```

---

### `getElementsByClassName` and `getElementsByTagName`

```js
// LIVE HTMLCollection — auto-updates when DOM changes
const buttons   = document.getElementsByClassName("btn");
const allLists  = document.getElementsByTagName("ul");
const allEls    = document.getElementsByTagName("*");    // every element

// Multiple class names (space-separated — ALL must be present):
document.getElementsByClassName("btn primary");   // elements with both classes

// ── Live collection trap ──────────────────────────────────────
const items = document.getElementsByClassName("item");

// WRONG — infinite loop! items.length decreases as items are removed,
// but the loop re-evaluates items.length each time:
for (let i = 0; i < items.length; i++) {
  items[i].remove();   // items.length shrinks → misses every other element!
}

// RIGHT — convert to static array first, then operate:
[...items].forEach(item => item.remove());
// OR iterate backwards:
for (let i = items.length - 1; i >= 0; i--) {
  items[i].remove();
}
```

---

### `getElementsByName`

```js
// Selects elements by their "name" attribute — primarily used for form elements
// Returns a live NodeList
const emailInputs = document.getElementsByName("email");
const radioGroup  = document.getElementsByName("payment-method");

radioGroup.forEach(radio => {
  if (radio.checked) console.log("Selected:", radio.value);
});
```

---

### Scoped queries — searching within an element

Every element has `querySelector` and `querySelectorAll` — they search only within that element's subtree:

```js
// WRONG — expensive: searches the entire document
for (const row of tableRows) {
  const price = document.querySelector(".price");   // searches everywhere!
}

// RIGHT — scoped: searches only inside each row
for (const row of tableRows) {
  const price = row.querySelector(".price");   // fast, correct
}

// Practical example: each user card has its own .badge
const cards = document.querySelectorAll(".user-card");

cards.forEach(card => {
  const badge = card.querySelector(".badge");    // relative to card, not document
  const name  = card.querySelector("h2");
  console.log(name.textContent, badge.textContent);
});
```

---

### `closest` — navigate UP the tree to find a matching ancestor

```js
// opposite of querySelector — goes UP the tree, not down
// Returns the nearest ancestor matching the selector (or the element itself), or null

const btn = document.querySelector("#delete-btn");

btn.addEventListener("click", (event) => {
  const card = event.target.closest(".user-card");
  // walks UP from the clicked element until it finds .user-card
  if (card) {
    const userId = card.dataset.userId;
    deleteUser(userId);
  }
});
```

---

### `matches` — test if an element matches a selector

```js
// Returns true/false — does THIS element match this selector?
const el = document.querySelector(".card");

el.matches(".card");               // true
el.matches(".card.featured");      // depends on element's classes
el.matches("div");                 // true if el is a <div>
el.matches("[data-active]");       // true if element has data-active attribute

// Useful in event delegation:
document.addEventListener("click", (event) => {
  if (event.target.matches(".btn-delete")) {
    handleDelete(event.target);
  }
  if (event.target.matches("input[type='checkbox']")) {
    handleCheckboxChange(event.target);
  }
});
```

---

## Example 1 — selecting and reading form data

```html
<form id="signup-form">
  <input type="text"     name="username"  id="username"  value="">
  <input type="email"    name="email"     id="email"     value="">
  <input type="password" name="password"  id="password"  value="">
  <select id="role">
    <option value="viewer">Viewer</option>
    <option value="editor" selected>Editor</option>
    <option value="admin">Admin</option>
  </select>
  <button type="submit" id="submit-btn">Sign Up</button>
</form>
```

```js
// Select form and all inputs inside it:
const form     = document.getElementById("signup-form");
const inputs   = form.querySelectorAll("input");          // 3 inputs
const username = form.querySelector("#username");          // scoped to form
const roleEl   = form.querySelector("#role");

// Read values:
console.log(username.value);        // current typed value
console.log(roleEl.value);          // selected option value: "editor"

// Collect all form values as an object:
function getFormData(form) {
  const data = {};
  form.querySelectorAll("input, select, textarea").forEach(field => {
    if (field.name) {
      data[field.name] = field.value;
    }
  });
  return data;
}

document.getElementById("submit-btn").addEventListener("click", (e) => {
  e.preventDefault();
  const data = getFormData(form);
  console.log(data);
  // { username: "...", email: "...", password: "...", role: "editor" }
});
```

---

## Example 2 — real world: dashboard component selection

```js
// A dashboard with multiple stat cards, a data table, and filter controls

class Dashboard {
  constructor(rootSelector) {
    this.root = document.querySelector(rootSelector);
    if (!this.root) throw new Error(`Dashboard root "${rootSelector}" not found`);

    // Cache all selectors on init — avoids querying the DOM on every interaction
    this.els = {
      statCards:    this.root.querySelectorAll(".stat-card"),
      tableBody:    this.root.querySelector(".data-table tbody"),
      tableRows:    () => this.root.querySelectorAll(".data-table tbody tr"),  // fn = live
      searchInput:  this.root.querySelector(".search-input"),
      filterBtns:   this.root.querySelectorAll(".filter-btn"),
      loadingSpinner: this.root.querySelector(".spinner"),
      emptyState:   this.root.querySelector(".empty-state"),
    };
  }

  updateStat(cardId, value) {
    const card = this.root.querySelector(`[data-stat="${cardId}"]`);
    if (card) {
      card.querySelector(".stat-value").textContent = value;
    }
  }

  filterRows(query) {
    const q = query.toLowerCase();

    this.els.tableRows().forEach(row => {
      const text = row.textContent.toLowerCase();
      row.hidden = !text.includes(q);   // hide non-matching rows
    });

    const visibleRows = this.root.querySelectorAll(".data-table tbody tr:not([hidden])");
    this.els.emptyState.hidden = visibleRows.length > 0;
  }

  setActiveFilter(activeBtn) {
    this.els.filterBtns.forEach(btn => btn.classList.remove("active"));
    activeBtn.classList.add("active");
  }
}

const dash = new Dashboard("#main-dashboard");

dash.els.searchInput.addEventListener("input", (e) => {
  dash.filterRows(e.target.value);
});

dash.els.filterBtns.forEach(btn => {
  btn.addEventListener("click", (e) => {
    dash.setActiveFilter(e.currentTarget);
    dash.filterRows(btn.dataset.filter);
  });
});
```

---

## Comparison table — all selector methods

```
┌──────────────────────────────┬────────────────────┬──────────────────────┬────────────────┐
│ Method                       │ Returns            │ Live/Static          │ Match count    │
├──────────────────────────────┼────────────────────┼──────────────────────┼────────────────┤
│ getElementById(id)           │ Element or null    │ N/A (single)         │ One            │
│ querySelector(css)           │ Element or null    │ N/A (single)         │ First match    │
│ querySelectorAll(css)        │ NodeList           │ Static               │ All matches    │
│ getElementsByClassName(cls)  │ HTMLCollection     │ LIVE                 │ All matches    │
│ getElementsByTagName(tag)    │ HTMLCollection     │ LIVE                 │ All matches    │
│ getElementsByName(name)      │ NodeList           │ LIVE                 │ All matches    │
│ element.closest(css)         │ Element or null    │ N/A (single)         │ First ancestor │
│ element.matches(css)         │ Boolean            │ N/A                  │ Test only      │
└──────────────────────────────┴────────────────────┴──────────────────────┴────────────────┘
```

---

## Performance guide

```
Fastest:
  getElementById                — direct hashtable lookup
  element.querySelector         — scoped to subtree, not whole document

Medium:
  querySelector / querySelectorAll  — CSS engine traversal

Slower:
  getElementsByClassName, getElementsByTagName  — same speed as querySelector
  but live collections add overhead when DOM changes frequently

Tips:
  1. Cache selectors in variables — don't query the DOM in a loop
  2. Use the most specific selector you can — fewer elements to check
  3. Scope queries to a parent when possible (element.querySelector vs document.querySelector)
  4. Prefer static (querySelectorAll) over live collections for iteration safety
```

---

## Tricky things

### 1. `getElementById` does NOT take a `#` prefix

```js
document.getElementById("my-id");    // ✓
document.getElementById("#my-id");   // returns null — "#" is not part of the id
```

### 2. `querySelector` with `:scope` for relative queries

```js
const section = document.querySelector("section");

// Without :scope — this matches .child anywhere in document, not just inside section
section.querySelectorAll(".child");   // correct — scoped by default in modern browsers

// With :scope — explicitly means "relative to this element"
section.querySelectorAll(":scope > .child");   // only DIRECT children with class .child
// Same as: section.children (filtered) or section.querySelectorAll(":scope > .child")
```

### 3. `null` return — always guard against it

```js
// WRONG — crashes if element not found
const modal = document.querySelector(".modal");
modal.classList.add("visible");   // TypeError: Cannot read properties of null

// RIGHT — guard or use optional chaining
const modal = document.querySelector(".modal");
modal?.classList.add("visible");  // silent if null

// Or assert and fail loud in development:
function getEl(selector, parent = document) {
  const el = parent.querySelector(selector);
  if (!el) throw new Error(`Element not found: "${selector}"`);
  return el;
}
```

### 4. Live collection mutation trap — iterating while removing

```js
const active = document.getElementsByClassName("active");

// WRONG — removes every other element because .length shrinks
for (let i = 0; i < active.length; i++) {
  active[i].classList.remove("active");  // shrinks live collection mid-loop!
}

// RIGHT — snapshot first
[...active].forEach(el => el.classList.remove("active"));
```

### 5. `querySelectorAll` is static — stored results don't auto-update

```js
const items = document.querySelectorAll(".item");
console.log(items.length);   // 3

// Add a new .item to the DOM
const newItem = document.createElement("li");
newItem.className = "item";
list.appendChild(newItem);

console.log(items.length);   // still 3 — static snapshot!
// Must call querySelectorAll again to get updated results
```

### 6. CSS selector special characters need escaping

```js
// IDs or classes with special characters (dots, colons, brackets in the value):
// HTML: <div id="user.profile">
document.querySelector("#user.profile");   // WRONG — interpreted as id="user" AND class="profile"
document.querySelector("#user\\.profile"); // RIGHT — escaped dot

// Better: use getElementById for IDs with special chars
document.getElementById("user.profile");   // works — no CSS parsing
```

---

## Common mistakes

### Mistake 1 — Not guarding against `null`

```js
// WRONG
document.querySelector(".non-existent").textContent = "hello";
// TypeError: Cannot set properties of null

// RIGHT
const el = document.querySelector(".non-existent");
if (el) el.textContent = "hello";
// OR:
document.querySelector(".non-existent")?.textContent = "hello";   // optional chaining
// Wait — optional chaining on left side of assignment is not valid. Use if:
const el = document.querySelector(".non-existent");
if (el) el.textContent = "hello";
```

### Mistake 2 — Querying the whole document when you should scope

```js
// WRONG — finds the FIRST .title anywhere on the page (might be wrong one)
document.querySelectorAll(".card").forEach(card => {
  const title = document.querySelector(".title");   // global search — bug!
  console.log(title.textContent);
});

// RIGHT — scope to each card
document.querySelectorAll(".card").forEach(card => {
  const title = card.querySelector(".title");   // relative to THIS card
  console.log(title.textContent);
});
```

### Mistake 3 — Using `innerHTML` to "select" by constructing selectors

```js
// WRONG — rebuilding innerHTML to extract values loses all listeners and state
const value = document.querySelector("#myList").innerHTML.match(/something/);

// RIGHT — query the specific element you need
const item = document.querySelector("#myList .highlighted");
const value = item?.textContent;
```

---

## Practice exercises

### Exercise 1 — easy

Given this HTML, write the JavaScript to perform each selection — use the most appropriate method for each:

```html
<div id="app">
  <header class="site-header">
    <nav>
      <a class="nav-link active" href="/">Home</a>
      <a class="nav-link" href="/about">About</a>
      <a class="nav-link" href="/contact">Contact</a>
    </nav>
  </header>
  <main>
    <article class="post featured" data-post-id="101">
      <h2 class="post-title">Hello World</h2>
      <p class="post-body">Content here.</p>
    </article>
    <article class="post" data-post-id="102">
      <h2 class="post-title">Second Post</h2>
      <p class="post-body">More content.</p>
    </article>
  </main>
</div>
```

1. Select `#app` by ID
2. Select the first `.nav-link` that has class `active`
3. Select all `.nav-link` elements
4. Select the `.post` with `data-post-id="102"` using a CSS attribute selector
5. Scope a query to the second `article` to get its `.post-title`
6. Use `closest` on the first `<a>` to find its parent `<nav>`
7. Use `matches` to check if the first `<a>` has class `active`
8. Log `tagName`, `textContent`, and `className` of every result

```js
// Write your code here
```

### Exercise 2 — medium

Build a `$(selector, context)` utility function that:
- If `selector` starts with `#`, uses `getElementById` (fastest)
- Otherwise, uses `querySelector` scoped to `context` (defaulting to `document`)
- Returns `null` if not found (never throws)

Then build a `$$(selector, context)` that:
- Returns an Array (not NodeList) of all matches using `querySelectorAll` scoped to `context`
- If `context` is not provided, defaults to `document`
- Returns empty array `[]` if nothing found

Then use both utilities to:
1. Get the form with id `"contact-form"` using `$`
2. Get all `input[required]` fields inside it using `$$`
3. Validate: log which required fields are empty
4. Log the `name` attribute of each empty field

```js
// Write your code here
```

### Exercise 3 — hard

Build a `ComponentRegistry` that manages selecting and caching DOM components:

```js
class ComponentRegistry {
  // constructor(rootSelector) — root element to scope all queries to
  // register(name, selector) — register a named component selector
  // get(name) — return the first matching element for that name (cached)
  // getAll(name) — return all matching elements as Array (fresh query each time)
  // refresh(name) — clear cache for that name so next .get() re-queries
  // refreshAll() — clear all caches
  // exists(name) — returns true if at least one matching element exists
  // on(name, event, handler) — attach event listener to the component element
  // delegate(name, childSelector, event, handler) — event delegation within component
}

// Usage:
const registry = new ComponentRegistry("#app");

registry.register("modal",       ".modal");
registry.register("closeBtn",    ".modal .close-btn");
registry.register("overlay",     ".modal-overlay");
registry.register("userCards",   ".user-card");
registry.register("deleteButtons", "[data-action='delete']");

registry.on("closeBtn", "click", () => {
  registry.get("modal")?.classList.remove("open");
  registry.get("overlay")?.classList.remove("visible");
});

// Event delegation — all .delete-btn clicks inside the userCards container:
registry.delegate("userCards", "[data-action='delete']", "click", (event, matchedEl) => {
  const card = matchedEl.closest(".user-card");
  console.log("Delete user:", card.dataset.userId);
});

console.log(registry.exists("modal"));   // true or false
console.log(registry.getAll("userCards").length);   // count of all user cards
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Goal | Method | Notes |
|---|---|---|
| By ID | `getElementById("id")` | No `#`; fastest method |
| First CSS match | `querySelector("css")` | Returns `null` if none |
| All CSS matches | `querySelectorAll("css")` | Returns static NodeList |
| By class (live) | `getElementsByClassName("cls")` | Live HTMLCollection |
| By tag (live) | `getElementsByTagName("tag")` | Live HTMLCollection |
| By `name` attr | `getElementsByName("name")` | Live NodeList, forms |
| Nearest ancestor | `element.closest("css")` | Goes UP the tree |
| Test match | `element.matches("css")` | Returns boolean |
| Scope to element | `element.querySelector(...)` | Searches inside element only |
| Convert NodeList | `Array.from(list)` or `[...list]` | For `.map()`, `.filter()` |
| Guard null | `el?.doSomething()` or `if (el)` | Always guard single-element methods |
| Live → static | `[...liveCollection]` | Safe to iterate while modifying |
| `:scope` | `:scope > .child` | Explicit relative selector |
| Cache selectors | Store in variables, not in loops | Prevents redundant DOM traversal |

---

## Connected topics

- **65 — How the DOM works** — the tree structure these methods traverse
- **67 — Manipulating the DOM** — what you do with elements once selected
- **68 — Events** — attaching event handlers to selected elements
- **69 — Event delegation** — using `closest` and `matches` for efficient event handling
