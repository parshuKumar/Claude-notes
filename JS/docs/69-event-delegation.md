# 69 — Event Delegation

---

## 1. What is this?

**Event delegation** is the pattern of attaching a **single event listener to a parent element** instead of attaching separate listeners to each child element. When a child is interacted with, the event bubbles up to the parent, and the handler inspects `e.target` to decide what to do.

**Analogy — reception desk:**
Imagine a company where 50 employees each get their own phone line. That's 50 phones to manage. Instead, a single receptionist handles all incoming calls and routes them: "You want Sales? Let me transfer you."

Event delegation is the receptionist. One listener handles all children — even ones that didn't exist when the listener was attached.

**Analogy — Express middleware:**
In Node.js, you don't write `app.get("/api/users/1")`, `app.get("/api/users/2")`, etc. for each user ID. You write `app.get("/api/users/:id", handler)` — one route, matched dynamically. Event delegation is identical: one handler, matched dynamically via `e.target`.

---

## 2. Why does it matter?

### Without delegation — the naive approach

```js
const buttons = document.querySelectorAll(".delete-btn");
buttons.forEach(btn => btn.addEventListener("click", handleDelete));
```

Problems:
1. **Memory** — 1000 items = 1000 listeners
2. **Dynamic content** — buttons added to the DOM *after* this code runs have no listener
3. **Maintenance** — if you add a new button type, you have to remember to wire it up
4. **Setup cost** — every `addEventListener` call has overhead

### With delegation

```js
list.addEventListener("click", handleDelete);
```

- One listener handles all current and future children
- Works perfectly with dynamic lists (add/remove items freely)
- Negligible memory usage regardless of child count

---

## 3. Syntax

```js
// Basic pattern
parent.addEventListener("click", (e) => {
  if (e.target.matches(".delete-btn")) {
    // handle delete
  }
});

// Safer pattern — handles clicks on child elements inside the button
parent.addEventListener("click", (e) => {
  const btn = e.target.closest(".delete-btn");
  if (!btn || !parent.contains(btn)) return;
  // handle delete
});
```

---

## 4. How it works — line by line

```js
const list = document.querySelector("#todo-list");

list.addEventListener("click", (e) => {
  // e.target  = the exact element clicked (could be deep child)
  // e.currentTarget = list (where the listener lives)

  const deleteBtn = e.target.closest("[data-action='delete']");
  if (!deleteBtn) return;                    // click wasn't on or inside a delete button

  const item = deleteBtn.closest("li");      // find the parent list item
  const id   = item.dataset.id;             // read the data attribute

  removeItem(id);
  item.remove();
});
```

Step by step:
1. User clicks anywhere inside `#todo-list` — the click bubbles up to the list
2. `e.target` is whatever was actually clicked (could be an `<svg>` icon inside the button)
3. `.closest("[data-action='delete']")` walks up from `e.target` until it finds a matching ancestor (or itself), returns `null` if none found
4. Early return if the click wasn't on a delete button
5. Walk up again to find the `<li>` and read its `data-id`

---

## 5. `e.target.matches()` vs `e.target.closest()`

This is the most important decision in delegation.

### `matches()` — checks only the exact element clicked

```js
parent.addEventListener("click", (e) => {
  if (e.target.matches(".btn")) {
    // ✅ Works if user clicks directly on .btn
    // ❌ Fails if user clicks on an <svg> or <span> INSIDE .btn
  }
});
```

```html
<button class="btn">
  <svg>...</svg>   <!-- user clicks here — e.target is <svg>, not .btn -->
  Save
</button>
```

### `closest()` — walks up the tree

```js
parent.addEventListener("click", (e) => {
  const btn = e.target.closest(".btn");
  if (btn) {
    // ✅ Works whether user clicks the button itself OR any child inside it
  }
});
```

**Rule of thumb:** use `closest()` for anything that might contain child elements (buttons with icons, list items with nested content). Use `matches()` only for simple text-only elements.

### Safety check — `contains()`

```js
parent.addEventListener("click", (e) => {
  const btn = e.target.closest(".btn");

  // Without this check, closest() could match an ancestor OUTSIDE parent
  // (e.g., if .btn is a class used higher up the tree)
  if (!btn || !parent.contains(btn)) return;

  handleClick(btn);
});
```

---

## 6. Multiple Actions on One Parent

A single parent listener can handle many different action types cleanly using `data-action`:

```html
<ul id="task-list">
  <li data-id="1">
    Task One
    <button data-action="edit">Edit</button>
    <button data-action="delete">Delete</button>
    <button data-action="complete">Done</button>
  </li>
</ul>
```

```js
const taskList = document.querySelector("#task-list");

const actions = {
  edit:     (item) => openEditModal(item.dataset.id),
  delete:   (item) => deleteTask(item.dataset.id),
  complete: (item) => markComplete(item.dataset.id)
};

taskList.addEventListener("click", (e) => {
  const btn  = e.target.closest("[data-action]");
  if (!btn || !taskList.contains(btn)) return;

  const action = btn.dataset.action;
  const item   = btn.closest("li[data-id]");

  actions[action]?.(item); // optional chaining handles unknown actions gracefully
});
```

Adding a new action type only requires:
1. Adding a button with `data-action="newAction"` in HTML
2. Adding `newAction: (item) => ...` to the `actions` object

No new `addEventListener` calls needed.

---

## 7. Delegation with Dynamically Added Elements

This is where delegation genuinely shines over per-element listeners.

```js
const container = document.querySelector("#comment-list");

// ✅ This listener handles ALL comments — even ones loaded via fetch later
container.addEventListener("click", (e) => {
  const likeBtn = e.target.closest(".like-btn");
  if (!likeBtn) return;

  const commentId = likeBtn.closest("[data-comment-id]").dataset.commentId;
  likeComment(commentId);
  updateLikeCount(likeBtn);
});

// Load more comments dynamically — the listener already handles them
async function loadMoreComments() {
  const res  = await fetch("/api/comments?page=2");
  const html = await res.text();
  container.insertAdjacentHTML("beforeend", html);
  // No need to re-attach listeners
}
```

Compare to the naive approach:

```js
// ❌ These new buttons have no listeners — they were added after the forEach ran
async function loadMoreComments() {
  const res  = await fetch("/api/comments?page=2");
  const html = await res.text();
  container.insertAdjacentHTML("beforeend", html);

  // "Fix" — but this doubles up listeners on existing buttons
  container.querySelectorAll(".like-btn").forEach(btn =>
    btn.addEventListener("click", handleLike)
  );
}
```

---

## 8. Real-World Examples

### Example 1 — Full Todo List

```js
class TodoList {
  constructor(containerEl) {
    this.container = containerEl;
    this.items = new Map(); // id → { text, done }
    this._bindEvents();
  }

  _bindEvents() {
    // ONE listener handles add, delete, toggle, edit — all current and future items
    this.container.addEventListener("click", (e) => {
      const target = e.target.closest("[data-action]");
      if (!target || !this.container.contains(target)) return;

      const action = target.dataset.action;
      const id     = target.closest("[data-id]")?.dataset.id;

      switch (action) {
        case "delete":   return this.delete(id);
        case "toggle":   return this.toggle(id);
        case "edit":     return this.startEdit(id);
        case "save":     return this.saveEdit(id, target);
      }
    });

    this.container.addEventListener("keydown", (e) => {
      if (e.key === "Enter" && e.target.matches("[data-edit-input]")) {
        const id = e.target.closest("[data-id]").dataset.id;
        this.saveEdit(id, e.target);
      }
    });
  }

  add(id, text) {
    this.items.set(id, { text, done: false });
    this.container.insertAdjacentHTML("beforeend", `
      <li data-id="${id}">
        <span data-action="toggle">${this._escape(text)}</span>
        <button data-action="edit">Edit</button>
        <button data-action="delete">Delete</button>
      </li>
    `);
  }

  delete(id) {
    this.items.delete(id);
    this.container.querySelector(`[data-id="${id}"]`).remove();
  }

  toggle(id) {
    const item = this.items.get(id);
    item.done = !item.done;
    this.container.querySelector(`[data-id="${id}"]`)
      .classList.toggle("done", item.done);
  }

  startEdit(id) {
    const li   = this.container.querySelector(`[data-id="${id}"]`);
    const span = li.querySelector("[data-action='toggle']");
    const text = this.items.get(id).text;
    span.replaceWith(
      Object.assign(document.createElement("input"), {
        value: text,
        dataset: { editInput: "", action: "save" }
      })
    );
  }

  saveEdit(id, inputEl) {
    const newText = inputEl.value.trim();
    if (!newText) return;
    this.items.get(id).text = newText;
    const span = document.createElement("span");
    span.dataset.action = "toggle";
    span.textContent = newText;
    inputEl.replaceWith(span);
  }

  _escape(text) {
    return text.replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/>/g,"&gt;");
  }
}
```

### Example 2 — Data table with sortable columns

```js
const table = document.querySelector("#data-table");
let sortState = { col: null, dir: "asc" };

// One listener on <thead> handles all column header clicks
table.querySelector("thead").addEventListener("click", (e) => {
  const th = e.target.closest("th[data-col]");
  if (!th) return;

  const col = th.dataset.col;
  sortState.dir = (sortState.col === col && sortState.dir === "asc") ? "desc" : "asc";
  sortState.col = col;

  sortTable(col, sortState.dir);
  updateSortIndicators(th, sortState.dir);
});

function sortTable(col, dir) {
  const tbody = table.querySelector("tbody");
  const rows  = [...tbody.querySelectorAll("tr")];

  rows.sort((a, b) => {
    const aVal = a.querySelector(`[data-col="${col}"]`).textContent;
    const bVal = b.querySelector(`[data-col="${col}"]`).textContent;
    return dir === "asc"
      ? aVal.localeCompare(bVal, undefined, { numeric: true })
      : bVal.localeCompare(aVal, undefined, { numeric: true });
  });

  tbody.append(...rows); // re-insert in sorted order (moves existing nodes)
}
```

### Example 3 — Accordion / collapsible panels

```js
const accordion = document.querySelector(".accordion");

accordion.addEventListener("click", (e) => {
  const header = e.target.closest(".accordion-header");
  if (!header) return;

  const panel  = header.nextElementSibling; // the content panel
  const isOpen = panel.hidden === false;

  // Close all panels (only-one-open behaviour)
  accordion.querySelectorAll(".accordion-panel").forEach(p => {
    p.hidden = true;
    p.previousElementSibling.setAttribute("aria-expanded", "false");
  });

  // Toggle clicked panel
  if (!isOpen) {
    panel.hidden = false;
    header.setAttribute("aria-expanded", "true");
  }
});
```

### Example 4 — Context menu (right-click menu)

```js
const contextMenu = document.querySelector("#context-menu");
let targetRow = null;

// Delegation on the table body — right-click on any row
document.querySelector("tbody").addEventListener("contextmenu", (e) => {
  const row = e.target.closest("tr[data-id]");
  if (!row) return;

  e.preventDefault(); // stop native browser context menu
  targetRow = row;

  contextMenu.style.left = `${e.clientX}px`;
  contextMenu.style.top  = `${e.clientY}px`;
  contextMenu.hidden = false;
});

// Delegation on the context menu items
contextMenu.addEventListener("click", (e) => {
  const item = e.target.closest("[data-menu-action]");
  if (!item) return;

  const action = item.dataset.menuAction;
  const id     = targetRow?.dataset.id;

  if (action === "view")   viewRecord(id);
  if (action === "edit")   editRecord(id);
  if (action === "delete") deleteRecord(id);

  contextMenu.hidden = true;
  targetRow = null;
});

// Close menu on outside click
document.addEventListener("click", () => {
  contextMenu.hidden = true;
});
```

### Example 5 — Form builder (delegated validation)

```js
const form = document.querySelector("#dynamic-form");

// One listener validates all fields — including ones added dynamically
form.addEventListener("blur", (e) => {
  const field = e.target.closest("[data-validate]");
  if (!field) return;

  const rule    = field.dataset.validate;
  const value   = field.value;
  const isValid = validate(rule, value);

  field.classList.toggle("invalid", !isValid);
  field.setAttribute("aria-invalid", String(!isValid));

  const errEl = field.parentElement.querySelector(".field-error");
  if (errEl) errEl.textContent = isValid ? "" : getErrorMessage(rule, value);
}, true); // capture: true — blur doesn't bubble, so capture phase catches it

function validate(rule, value) {
  const rules = {
    required:  (v) => v.trim().length > 0,
    email:     (v) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
    minLength: (v) => v.trim().length >= 8,
    numeric:   (v) => /^\d+$/.test(v),
  };
  return rules[rule]?.(value) ?? true;
}
```

> Note: `blur` doesn't bubble, so the listener uses `{ capture: true }` to intercept it during the capture phase. This is one of the few valid use cases for capture-phase listeners.

---

## 9. Tricky Things

### 1. `blur` and `focus` don't bubble — use capture or `focusout`/`focusin`

```js
// ❌ blur doesn't bubble — parent never receives it
form.addEventListener("blur", handler);

// ✅ Option A: use focusout (bubbles)
form.addEventListener("focusout", handler);

// ✅ Option B: use capture phase
form.addEventListener("blur", handler, { capture: true });
```

### 2. `closest()` can escape the parent

```js
// If your page has a .btn class on an element ABOVE parent,
// closest() will find it and your guard is skipped

const parent = document.querySelector("#widget");

parent.addEventListener("click", (e) => {
  const btn = e.target.closest(".btn");
  // btn might be outside #widget!

  // ✅ Always verify it's inside parent
  if (!btn || !parent.contains(btn)) return;
});
```

### 3. `stopPropagation` inside a child breaks delegation

```js
// ❌ This child's stopPropagation prevents the parent's delegated handler from firing
child.addEventListener("click", (e) => {
  e.stopPropagation(); // event never reaches parent
  doChildThing();
});

// ✅ Use e.target checks in the parent instead of stopping propagation
parent.addEventListener("click", (e) => {
  if (e.target.closest(".child")) return; // just skip child clicks
  doParentThing();
});
```

### 4. SVG and `matches()` quirks

```js
// SVG elements have different class handling — classList works, but
// e.target.matches(".icon") may fail on SVG child elements like <path>

// ✅ Use closest() which handles this more reliably
const icon = e.target.closest(".icon-btn");
```

### 5. Event delegation doesn't work for non-bubbling events

| Event | Bubbles? | Delegation possible? |
|---|---|---|
| `click` | ✅ yes | ✅ yes |
| `input` | ✅ yes | ✅ yes |
| `change` | ✅ yes | ✅ yes |
| `submit` | ✅ yes | ✅ yes |
| `focus` | ❌ no | use `focusin` or capture |
| `blur` | ❌ no | use `focusout` or capture |
| `mouseenter` | ❌ no | use `mouseover` |
| `mouseleave` | ❌ no | use `mouseout` |
| `scroll` | ❌ no (on element) | ❌ no |

---

## 10. Common Mistakes

### Mistake 1 — Delegating to `document` unnecessarily

```js
// ❌ Technically works but every click on the page runs through this
document.addEventListener("click", (e) => {
  if (e.target.matches(".delete-btn")) deleteItem(e.target);
});

// ✅ Delegate to the closest meaningful container
document.querySelector("#item-list").addEventListener("click", (e) => {
  if (e.target.closest(".delete-btn")) deleteItem(e.target.closest(".delete-btn"));
});
```

Delegating to `document` adds noise to every single click on the page. Pick the closest stable ancestor that contains all the relevant children.

### Mistake 2 — Checking `e.target` directly for elements with children

```js
// ❌ Only works if the button has no child elements
parent.addEventListener("click", (e) => {
  if (e.target.matches("button.save")) save();
  //  Fails when user clicks the <svg> inside <button class="save"><svg>...</svg></button>
});

// ✅ Use closest()
parent.addEventListener("click", (e) => {
  if (e.target.closest("button.save")) save();
});
```

### Mistake 3 — Forgetting to check containment

```js
// ❌ Could match elements outside the intended container
parent.addEventListener("click", (e) => {
  const item = e.target.closest(".list-item"); // could escape parent!
  if (item) handleItem(item);
});

// ✅ Guard with contains()
parent.addEventListener("click", (e) => {
  const item = e.target.closest(".list-item");
  if (item && parent.contains(item)) handleItem(item);
});
```

### Mistake 4 — Using delegation for events that don't bubble (without workaround)

```js
// ❌ mouseenter doesn't bubble — parent never fires for children
list.addEventListener("mouseenter", (e) => {
  const item = e.target.closest("li");
  if (item) highlightItem(item); // never runs
});

// ✅ Use mouseover (which does bubble) + closest()
list.addEventListener("mouseover", (e) => {
  const item = e.target.closest("li");
  if (item && list.contains(item)) highlightItem(item);
});

// Also handle mouse leaving a child (fires mouseover on parent)
list.addEventListener("mouseover", (e) => {
  const item = e.target.closest("li");
  list.querySelectorAll("li").forEach(li => li.classList.remove("hover"));
  if (item) item.classList.add("hover");
});
```

---

## 11. Practice Exercises

### Easy — Delegated highlight

Create a `<ul>` with 5 `<li>` items. When any `<li>` is clicked, add a class `"selected"` to it and remove it from all others. Use one listener on the `<ul>`.

---

### Medium — Delegated tag system

Build a tag input UI:
- An `<input>` and "Add" button
- Tags are rendered as `<span class="tag">text <button class="remove">×</button></span>`
- Clicking the × removes that tag
- Use **one delegated listener** on the tag container for all removes
- Tags added dynamically must also be removable without re-attaching listeners

---

### Hard — Data grid with inline editing

Build a `<table>` that renders 10 rows of data (id, name, email).

Requirements:
- **Double-click** any cell to make it editable (`<td>` → `<input>`)
- **Enter** or **blur** saves the change
- **Escape** cancels the edit and restores the original value
- Use **one delegated `dblclick` listener** on `<tbody>` for entering edit mode
- Use **one delegated `keydown` listener** on `<tbody>` for keyboard interactions
- Use **one delegated `focusout` listener** (or capture blur) for blur-save

No libraries. Track the original value in a `data-original` attribute.

---

## 12. Quick Reference

```
PATTERN
  parent.addEventListener(type, handler)
  handler: check e.target.closest(selector) → early return if null → act

KEY METHODS
  e.target.closest(".selector")     walk up from clicked el, return match or null
  e.target.matches(".selector")     check only the exact clicked el
  parent.contains(el)               verify el is inside parent (safety guard)

DATA ATTRIBUTES FOR DISPATCH
  data-action="delete|edit|toggle"  encode what should happen
  data-id="123"                     encode which item to act on

MULTI-ACTION PATTERN
  const actions = { delete: fn, edit: fn, toggle: fn };
  actions[btn.dataset.action]?.(item);

NON-BUBBLING WORKAROUNDS
  focus/blur    → use focusin/focusout  (or capture: true)
  mouseenter    → use mouseover         (+ closest() to simulate enter)
  mouseleave    → use mouseout

WHEN TO DELEGATE
  ✅ Dynamic lists (items added/removed at runtime)
  ✅ Large lists (50+ items)
  ✅ Multiple action types on list items
  ✅ Reusable components (accordion, tabs, menus)

WHEN NOT TO DELEGATE
  ❌ Single, static element (just attach directly)
  ❌ Non-bubbling events without a workaround
  ❌ Delegating to document for unrelated elements
```

---

## 13. Connected Topics

- **68 — Events** — bubbling, `e.target` vs `e.currentTarget`, `closest()`, `matches()`; prerequisite for this topic
- **67 — Manipulating the DOM** — `dataset`, `classList`, creating/removing nodes that delegation must handle
- **66 — Selecting elements** — `closest()`, `matches()`, `contains()` are the selector tools used inside delegated handlers
- **70 — Forms** — form validation is a prime use case for delegated `input`/`blur`/`submit` listeners
- **Node.js EventEmitter** — same pub/sub concept; delegation ≈ one `emitter.on("data", handler)` routing many message types
