# 68 — Events

---

## 1. What is this?

An **event** is a signal that something happened in the browser — a user clicked a button, pressed a key, resized the window, or the page finished loading. JavaScript lets you *listen* for those signals and run code when they fire.

**Analogy — webhook in backend terms:**
When a payment is made, Stripe sends a webhook to your server (`POST /webhook`). Your server has a route handler that runs. That's exactly how browser events work:

- The browser (Stripe) fires an event (sends the webhook)
- Your `addEventListener` call registers the route handler
- The handler function runs and receives an **event object** (the request body)

The same model is used inside Node.js itself — `EventEmitter` is built on the identical concept. You already know this pattern.

---

## 2. Why does it matter?

Every piece of interactivity on the web is events. Without them you'd have a static document with no way to respond to the user. Events are also the mechanism behind:

- Form submission and validation
- Keyboard shortcuts
- Drag-and-drop
- Real-time UI updates (typing, live search)
- Accessibility (focus, blur, keyboard nav)
- Custom component communication (custom events)
- Performance optimisations (passive scroll listeners)

---

## 3. Syntax

```js
// --- Adding a listener ---
element.addEventListener(type, handler, options);

// --- Removing a listener ---
element.removeEventListener(type, handler, options);

// --- Handler signature ---
function handler(event) { /* event = the event object */ }

// --- Options object ---
{
  capture: false,   // listen in capture phase instead of bubble phase
  once:    false,   // auto-remove after firing once
  passive: false,   // promise you won't call preventDefault (perf win for scroll/touch)
  signal:  controller.signal  // AbortSignal to remove the listener externally
}

// --- Dispatching events ---
element.dispatchEvent(new Event("myevent"));
element.dispatchEvent(new CustomEvent("myevent", { detail: { id: 42 } }));
```

---

## 4. How it works — line by line

### Registering a listener

```js
const btn = document.querySelector("#submit");

btn.addEventListener("click", handleClick);

function handleClick(event) {
  console.log(event.type);     // "click"
  console.log(event.target);   // the element that was actually clicked
}
```

- `"click"` — the event type string (lowercase, no "on" prefix)
- `handleClick` — a reference to the function, **not a call** (`handleClick` not `handleClick()`)
- `event` — the browser passes this automatically; you just name the parameter

### Removing a listener

```js
btn.removeEventListener("click", handleClick);
```

This only works if:
1. The **exact same function reference** is used (not a new arrow function)
2. The **capture option matches** what was passed to `addEventListener`

```js
// ❌ This does NOT remove the listener — different function reference
btn.addEventListener("click", () => doThing());
btn.removeEventListener("click", () => doThing()); // different arrow func each time
```

---

## 5. The Event Object

When an event fires, the browser creates an **Event object** and passes it to every handler. It carries everything about what happened.

### Core properties

| Property | Type | What it is |
|---|---|---|
| `event.type` | string | `"click"`, `"keydown"`, etc. |
| `event.target` | Element | The element the event **originated from** (what was actually clicked) |
| `event.currentTarget` | Element | The element the **listener is attached to** (these differ in delegation) |
| `event.bubbles` | boolean | Whether the event bubbles up the DOM |
| `event.cancelable` | boolean | Whether `preventDefault()` has any effect |
| `event.timeStamp` | number | Milliseconds since page load when the event fired |
| `event.isTrusted` | boolean | `true` for real user actions, `false` for scripted dispatches |

### Core methods

```js
event.preventDefault();           // stop the default browser action (form submit, link nav)
event.stopPropagation();          // stop bubbling/capturing — parent handlers don't fire
event.stopImmediatePropagation(); // stop propagation AND prevent other listeners on this same element
```

### target vs currentTarget

```js
<ul id="list">
  <li>Item 1</li>
  <li>Item 2</li>
</ul>

document.querySelector("#list").addEventListener("click", (e) => {
  console.log(e.target);        // <li>Item 1</li>  ← what was actually clicked
  console.log(e.currentTarget); // <ul id="list">   ← where the listener lives
});
```

This difference is the entire foundation of **event delegation** (topic 69).

---

## 6. Event Phases — Capture and Bubble

Every event travels in three phases:

```
                      Window
                        │
                    Document              ← CAPTURE phase going DOWN
                        │
                      <html>
                        │
                      <body>
                        │
                       <div>
                        │
                      <button>            ← TARGET phase (event fires here)
                        │
                       <div>
                        │                ← BUBBLE phase going UP
                      <body>
                        │
                    Document
                        │
                      Window
```

1. **Capture phase** — event travels from Window down to the target. Listeners with `capture: true` fire here.
2. **Target phase** — the event is at the element it originated from.
3. **Bubble phase** — event travels back up to Window. Default listeners fire here.

```js
// Capture phase listener (fires BEFORE the target's own handler)
parent.addEventListener("click", handler, { capture: true });

// Bubble phase listener (default)
parent.addEventListener("click", handler);
// same as:
parent.addEventListener("click", handler, { capture: false });
```

### Why does bubbling exist?

It lets a single parent element handle events from many children — the basis of event delegation.

### stopPropagation vs stopImmediatePropagation

```js
child.addEventListener("click", (e) => {
  e.stopPropagation();
  // parent's click handler will NOT fire
  // but if child has ANOTHER click handler, it still fires
});

child.addEventListener("click", (e) => {
  e.stopImmediatePropagation();
  // parent's handler won't fire
  // AND any further click handlers on child registered AFTER this one won't fire either
});
```

---

## 7. Options Object in Depth

### `once: true`

```js
// Handler fires exactly once, then auto-removes itself
btn.addEventListener("click", () => {
  console.log("First click only");
}, { once: true });
```

Use case: one-time initialisation, "accept terms" button, tutorial tooltips.

### `passive: true`

```js
// Tells the browser: "I promise not to call preventDefault()"
// Browser can start scrolling immediately without waiting for JS to run
window.addEventListener("scroll", trackScrollPosition, { passive: true });
document.addEventListener("touchmove", handleSwipe, { passive: true });
```

Without `passive: true`, the browser must wait for your scroll handler to finish before scrolling — this causes jank. Modern frameworks set this automatically for touch/wheel events.

You **cannot** call `preventDefault()` inside a passive listener — the browser silently ignores it.

### `signal` (AbortSignal)

```js
const controller = new AbortController();

document.addEventListener("keydown", handler, { signal: controller.signal });
window.addEventListener("resize", handler, { signal: controller.signal });
element.addEventListener("click", handler, { signal: controller.signal });

// Remove ALL of them at once
controller.abort();
```

This is the modern pattern for cleanup — especially useful in single-page apps when a component unmounts. Much cleaner than tracking every `removeEventListener` call separately.

---

## 8. Common Event Types

### Mouse events

```js
element.addEventListener("click", handler);       // left click (down + up on same element)
element.addEventListener("dblclick", handler);    // double click
element.addEventListener("mousedown", handler);   // mouse button pressed
element.addEventListener("mouseup", handler);     // mouse button released
element.addEventListener("mousemove", handler);   // mouse moves over element
element.addEventListener("mouseenter", handler);  // mouse enters element (does NOT bubble)
element.addEventListener("mouseleave", handler);  // mouse leaves element (does NOT bubble)
element.addEventListener("mouseover", handler);   // like mouseenter but DOES bubble (fires for children too)
element.addEventListener("mouseout", handler);    // like mouseleave but DOES bubble
element.addEventListener("contextmenu", handler); // right click
```

`mouseenter`/`mouseleave` vs `mouseover`/`mouseout`:
```
<div id="parent">        ← mouseenter fires once when you enter the parent
  <span>child</span>     ← mouseover fires again when you enter the child
</div>
```

### Keyboard events

```js
document.addEventListener("keydown", (e) => {
  console.log(e.key);     // "a", "Enter", "ArrowUp", "Escape", " " (space)
  console.log(e.code);    // "KeyA", "Enter", "ArrowUp", "Escape", "Space"
  console.log(e.ctrlKey); // true if Ctrl held
  console.log(e.shiftKey);// true if Shift held
  console.log(e.altKey);  // true if Alt held
  console.log(e.metaKey); // true if Cmd (Mac) or Win key held
});

document.addEventListener("keyup", handler);
// keypress is deprecated — use keydown
```

`e.key` vs `e.code`:
- `e.key` — **what** character was typed (layout-aware): `"a"` or `"A"` depending on Shift
- `e.code` — **which physical key** was pressed: always `"KeyA"` regardless of Shift

```js
// Keyboard shortcut: Ctrl+K
document.addEventListener("keydown", (e) => {
  if (e.ctrlKey && e.key === "k") {
    e.preventDefault(); // stop browser default (e.g., focus URL bar)
    openCommandPalette();
  }
});
```

### Form events

```js
input.addEventListener("input", handler);    // fires on every keystroke (real-time)
input.addEventListener("change", handler);   // fires when value changes AND focus leaves
input.addEventListener("focus", handler);    // element gains focus (does NOT bubble)
input.addEventListener("blur", handler);     // element loses focus (does NOT bubble)
input.addEventListener("focusin", handler);  // like focus but DOES bubble
input.addEventListener("focusout", handler); // like blur but DOES bubble

form.addEventListener("submit", (e) => {
  e.preventDefault(); // stop page reload
  // handle submission
});
form.addEventListener("reset", handler);

// input vs change
searchInput.addEventListener("input", (e) => {
  // fires every keystroke — use for live search
  liveSearch(e.target.value);
});

selectEl.addEventListener("change", (e) => {
  // fires when selection is confirmed — correct for <select>
  updateFilter(e.target.value);
});
```

### Window / Document events

```js
// DOM is parsed and ready, but images/stylesheets may still be loading
document.addEventListener("DOMContentLoaded", () => {
  initApp(); // safe to query DOM here
});

// Everything (images, fonts, stylesheets) fully loaded
window.addEventListener("load", () => {
  hideLoadingSpinner();
});

// User is about to leave the page
window.addEventListener("beforeunload", (e) => {
  if (hasUnsavedChanges()) {
    e.preventDefault();
    e.returnValue = ""; // required in some browsers to show confirmation dialog
  }
});

window.addEventListener("resize", handler);
window.addEventListener("scroll", handler);

// Visibility change (tab switching)
document.addEventListener("visibilitychange", () => {
  if (document.hidden) pauseVideo();
  else resumeVideo();
});
```

### Clipboard events

```js
document.addEventListener("copy", (e) => {
  e.preventDefault();
  e.clipboardData.setData("text/plain", "custom text");
});

document.addEventListener("paste", (e) => {
  const text = e.clipboardData.getData("text/plain");
  console.log("Pasted:", text);
});
```

---

## 9. Custom Events

You can create your own event types and dispatch them manually. This is how UI components communicate without tight coupling.

```js
// Create and dispatch
const event = new CustomEvent("user:login", {
  bubbles: true,           // bubble up the DOM
  cancelable: true,        // allow preventDefault
  detail: {                // arbitrary data payload
    userId: 42,
    username: "alice"
  }
});

document.dispatchEvent(event);

// Listen anywhere in the tree
document.addEventListener("user:login", (e) => {
  console.log(e.detail.username); // "alice"
});
```

Convention: use `namespace:action` naming (`cart:itemAdded`, `modal:close`) to avoid conflicts.

### Custom event bus (pub/sub)

```js
const EventBus = {
  emit(type, detail) {
    document.dispatchEvent(new CustomEvent(type, { detail, bubbles: false }));
  },
  on(type, handler, options) {
    document.addEventListener(type, handler, options);
    return () => document.removeEventListener(type, handler);  // returns cleanup fn
  }
};

// Publisher
EventBus.emit("cart:itemAdded", { productId: 99, qty: 1 });

// Subscriber (in another component)
const unsubscribe = EventBus.on("cart:itemAdded", (e) => {
  updateCartCount(e.detail.qty);
});

// Cleanup
unsubscribe();
```

This pattern mirrors `EventEmitter` in Node.js almost exactly.

---

## 10. Real-World Examples

### Example 1 — Live search with debounce

```js
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

const searchInput = document.querySelector("#search");

const handleSearch = debounce(async (e) => {
  const query = e.target.value.trim();
  if (!query) return clearResults();

  const results = await fetchSearchResults(query);
  renderResults(results);
}, 300);

searchInput.addEventListener("input", handleSearch);
```

### Example 2 — Modal with keyboard trap

```js
class Modal {
  constructor(el) {
    this.el = el;
    this.controller = new AbortController();
    const sig = { signal: this.controller.signal };

    el.addEventListener("click", this._onOverlayClick.bind(this), sig);
    document.addEventListener("keydown", this._onKeydown.bind(this), sig);
  }

  open() {
    this.el.hidden = false;
    this.el.querySelector("[autofocus]")?.focus();
  }

  close() {
    this.el.hidden = true;
    this.controller.abort(); // removes ALL listeners at once
  }

  _onKeydown(e) {
    if (e.key === "Escape") this.close();
    if (e.key === "Tab")    this._trapFocus(e);
  }

  _onOverlayClick(e) {
    if (e.target === this.el) this.close(); // click on backdrop, not content
  }

  _trapFocus(e) {
    const focusable = [...this.el.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )];
    const first = focusable[0];
    const last  = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      e.preventDefault();
      last.focus();
    } else if (!e.shiftKey && document.activeElement === last) {
      e.preventDefault();
      first.focus();
    }
  }
}
```

### Example 3 — Infinite scroll

```js
const sentinel = document.querySelector("#load-more-sentinel");

const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) loadNextPage();
}, { rootMargin: "200px" }); // trigger 200px before it's visible

observer.observe(sentinel);
```

> Note: IntersectionObserver is the modern replacement for scroll events on infinite scroll. It's passive by nature and far more performant.

### Example 4 — Drag and drop file upload

```js
const dropzone = document.querySelector("#dropzone");

dropzone.addEventListener("dragover", (e) => {
  e.preventDefault(); // required — without this, drop won't fire
  dropzone.classList.add("drag-over");
});

dropzone.addEventListener("dragleave", () => {
  dropzone.classList.remove("drag-over");
});

dropzone.addEventListener("drop", (e) => {
  e.preventDefault();
  dropzone.classList.remove("drag-over");

  const files = [...e.dataTransfer.files];
  files.forEach(uploadFile);
});

async function uploadFile(file) {
  const formData = new FormData();
  formData.append("file", file);

  const res = await fetch("/api/upload", { method: "POST", body: formData });
  const data = await res.json();
  console.log("Uploaded:", data.url);
}
```

### Example 5 — Component with full cleanup lifecycle

```js
class SearchComponent {
  constructor(containerEl) {
    this.container = containerEl;
    this.controller = null;
  }

  mount() {
    this.controller = new AbortController();
    const { signal } = this.controller;

    this.container.querySelector("input")
      .addEventListener("input", this.handleInput, { signal });

    this.container.querySelector("button")
      .addEventListener("click", this.handleClear, { signal });

    window.addEventListener("resize", this.handleResize, { signal, passive: true });
  }

  unmount() {
    this.controller?.abort(); // all listeners removed in one call
    this.controller = null;
  }

  handleInput = (e) => { /* ... */ };
  handleClear = ()  => { /* ... */ };
  handleResize = () => { /* ... */ };
}

const comp = new SearchComponent(document.querySelector("#search-widget"));
comp.mount();

// When navigating away in a SPA:
comp.unmount();
```

---

## 11. Tricky Things

### 1. Arrow functions lose `this` — use class fields or bind

```js
class Counter {
  count = 0;

  // ❌ `this` is undefined inside the handler
  handleClick() {
    this.count++; // TypeError if called as a callback
  }

  // ✅ Class field arrow function — `this` is bound at definition time
  handleClick = () => {
    this.count++;
  };
}

const c = new Counter();
btn.addEventListener("click", c.handleClick); // ✅ works with class field
```

### 2. `mouseenter` vs `mouseover` — pick carefully

```js
// mouseover fires repeatedly as you move over child elements
// mouseenter fires ONCE when you enter the parent — usually what you want

tooltip.addEventListener("mouseover", show);  // ❌ flickers on child elements
tooltip.addEventListener("mouseenter", show); // ✅ stable
```

### 3. `focus` and `blur` don't bubble — use `focusin`/`focusout` for delegation

```js
// ❌ This never fires for child elements gaining focus
form.addEventListener("focus", handler); // focus doesn't bubble

// ✅ focusin bubbles
form.addEventListener("focusin", handler);
```

### 4. `preventDefault` on `dragover` is required for `drop` to work

```js
// Without this, drop never fires — the browser cancels it
el.addEventListener("dragover", (e) => e.preventDefault());
el.addEventListener("drop", handleDrop);
```

### 5. Adding listeners inside loops creates duplicates

```js
// ❌ Every time updateList() runs, a new listener is added to each button
function updateList() {
  buttons.forEach(btn => btn.addEventListener("click", handler)); // accumulates!
}

// ✅ Use event delegation — one listener on the parent (see topic 69)
list.addEventListener("click", (e) => {
  if (e.target.matches("button")) handler(e);
});
```

### 6. `isTrusted` distinguishes real clicks from scripted clicks

```js
btn.addEventListener("click", (e) => {
  if (!e.isTrusted) {
    console.warn("Programmatic click — skipping payment");
    return;
  }
  processPurchase();
});

// e.isTrusted is false when you call btn.click() in code
// It's true for actual user clicks
```

---

## 12. Common Mistakes

### Mistake 1 — Calling the function instead of passing a reference

```js
// ❌ Calls handleClick() immediately and registers undefined as the handler
btn.addEventListener("click", handleClick());

// ✅ Pass the function reference
btn.addEventListener("click", handleClick);
```

### Mistake 2 — Anonymous function makes removeEventListener impossible

```js
// ❌ Can't remove this — no reference to the function
btn.addEventListener("click", () => doThing());
btn.removeEventListener("click", () => doThing()); // different function object — no-op

// ✅ Named reference
const onClick = () => doThing();
btn.addEventListener("click", onClick);
btn.removeEventListener("click", onClick);
```

### Mistake 3 — Forgetting preventDefault on form submit

```js
// ❌ Page reloads, you never see the console.log
form.addEventListener("submit", (e) => {
  const data = new FormData(form);
  console.log(data);
});

// ✅ Stop the default browser behaviour first
form.addEventListener("submit", (e) => {
  e.preventDefault();
  const data = new FormData(form);
  console.log(data);
});
```

### Mistake 4 — Using `keypress` (deprecated)

```js
// ❌ Deprecated — doesn't fire for all keys, not in new specs
input.addEventListener("keypress", handler);

// ✅ Use keydown or keyup
input.addEventListener("keydown", handler);
```

### Mistake 5 — Not cleaning up listeners (memory leaks in SPAs)

```js
// ❌ Every time the component re-mounts, a new listener stacks on window
function mountSearchBar() {
  window.addEventListener("keydown", handler);
}

// ✅ Clean up on unmount
function mountSearchBar() {
  window.addEventListener("keydown", handler);
  return () => window.removeEventListener("keydown", handler); // return cleanup
}
const cleanup = mountSearchBar();
cleanup(); // call when unmounting
```

### Mistake 6 — Mutating DOM in a scroll listener without throttling

```js
// ❌ Fires 60+ times per second, causes layout thrashing
window.addEventListener("scroll", () => {
  header.style.opacity = 1 - (window.scrollY / 300);
});

// ✅ Use passive + requestAnimationFrame to throttle
let ticking = false;
window.addEventListener("scroll", () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      header.style.opacity = 1 - (window.scrollY / 300);
      ticking = false;
    });
    ticking = true;
  }
}, { passive: true });
```

---

## 13. Practice Exercises

### Easy — Keyboard counter

Build a `<div>` that shows how many times the user has pressed any key since the page loaded. Display the key name of the last key pressed.

```
Keys pressed: 7   Last key: "Enter"
```

Requirements:
- Use `keydown` on `document`
- Update the display in real time
- No page reload behaviour needed

---

### Medium — Password strength meter

Build a password input with a live strength indicator. As the user types, show a coloured bar:
- Red = weak (< 8 chars or no uppercase)
- Orange = medium (8+ chars, has uppercase, has number)
- Green = strong (12+ chars, has uppercase, number, and special char)

Requirements:
- Use `input` event
- Update a `<div>` with class `strength-bar` using `classList`
- Show a text label ("Weak / Medium / Strong")
- Do not use any library

---

### Hard — Drag-sortable list

Build a `<ul>` whose `<li>` items can be dragged to reorder.

Requirements:
- Use `dragstart`, `dragover`, `drop`, `dragend`
- Highlight the drop target visually
- After drop, the DOM order should match the new visual order
- Store the final order in an array and log it to console
- No libraries — pure DOM events

---

## 14. Quick Reference

```
ADDING / REMOVING
  el.addEventListener(type, fn, options)   add listener
  el.removeEventListener(type, fn, options) remove listener (same fn ref + options)
  controller.abort()                        remove all listeners sharing the signal

OPTIONS OBJECT
  { capture: false }   listen in bubble phase (default)
  { capture: true }    listen in capture phase
  { once: true }       auto-remove after first fire
  { passive: true }    promise no preventDefault — perf win for scroll/touch
  { signal: sig }      AbortSignal for external cleanup

EVENT OBJECT
  e.type               "click", "keydown", etc.
  e.target             element that triggered the event
  e.currentTarget      element the listener is on
  e.bubbles            does this event bubble?
  e.isTrusted          true = real user action
  e.timeStamp          ms since page load

  e.preventDefault()            stop browser default (link nav, form submit)
  e.stopPropagation()           stop bubbling — parent handlers won't fire
  e.stopImmediatePropagation()  stop propagation + other handlers on this element

KEYBOARD
  e.key    logical key ("a", "A", "Enter", "Escape", "ArrowUp")
  e.code   physical key ("KeyA", "Enter", "ArrowUp")
  e.ctrlKey / e.shiftKey / e.altKey / e.metaKey

CUSTOM EVENTS
  new CustomEvent("type", { bubbles, cancelable, detail: {} })
  el.dispatchEvent(event)

COMMON TYPES
  Mouse:    click  dblclick  mousedown  mouseup  mousemove
            mouseenter  mouseleave  mouseover  mouseout  contextmenu
  Keyboard: keydown  keyup
  Form:     input  change  submit  reset  focus  blur  focusin  focusout
  Window:   DOMContentLoaded  load  beforeunload  resize  scroll  visibilitychange
  Pointer:  pointerdown  pointerup  pointermove  pointerenter  pointerleave
  Drag:     dragstart  dragover  dragleave  drop  dragend
```

---

## 15. Connected Topics

- **69 — Event delegation** — handling events from many children with one parent listener; uses `e.target` and `e.target.closest()`
- **67 — Manipulating the DOM** — `classList`, `dataset`, and style changes triggered by events
- **66 — Selecting elements** — `querySelector`, `closest`, `matches` inside event handlers
- **62 — Generators** — async generators can model event streams
- **52 — Async JS / Promises** — event-driven patterns and `Promise`-based event wrappers
- **58 — Fetch API** — fetch is initiated in response to events (form submit, button click)
- **Node.js EventEmitter** — identical pub/sub concept; `on`, `emit`, `off` mirror `addEventListener`, `dispatchEvent`, `removeEventListener`
