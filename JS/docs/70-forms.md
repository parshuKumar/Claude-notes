# 70 — Forms

---

## 1. What is this?

A **form** in HTML is a collection of input elements (`<input>`, `<select>`, `<textarea>`, `<checkbox>`, etc.) that the user fills in. JavaScript gives you a rich API to read values, validate them, and control submission — without ever letting the page reload.

**Analogy — API request body parsing:**
When an Express route receives `POST /users`, you call `req.body` to access all the fields the client sent. The browser's `FormData` API is the same idea: the form is the "request", and `FormData` lets you read every field by name, just like `req.body.username`.

---

## 2. Why does it matter?

- Client-side validation gives instant feedback before the server is hit
- `preventDefault()` + `fetch` lets you submit data without page reloads (SPA behaviour)
- `FormData` serialises the entire form in one line — including file uploads
- The Constraint Validation API lets you customise native browser error messages
- Form accessibility (`<label>`, `aria-*`, focus management) is entirely JavaScript-driven

---

## 3. Syntax — the essentials

```js
// Get the form element
const form = document.querySelector("#signup-form");

// Read a single input value
const username = document.querySelector("#username").value;

// Read all fields at once
const data = new FormData(form);
data.get("username");
data.getAll("interests"); // multi-value (checkboxes, multi-select)

// Submit handler
form.addEventListener("submit", (e) => {
  e.preventDefault();       // stop page reload
  const data = new FormData(e.target);
  // send data...
});

// Native validation API
input.validity.valid;          // boolean
input.validationMessage;       // browser error string
input.setCustomValidity("msg"); // set custom error ("" = valid)
input.reportValidity();        // show native tooltip
```

---

## 4. How it works — reading form values

### Individual inputs

```js
// Text, email, password, number, url, search, tel
input.value           // always a string, even for type="number"
Number(input.value)   // convert explicitly

// Checkbox
checkbox.checked       // boolean
checkbox.value         // the HTML value attribute (default "on")

// Radio group — find the checked one
const gender = document.querySelector('input[name="gender"]:checked')?.value;

// Select (single)
select.value           // selected option's value
select.selectedIndex   // index of selected option
select.options[select.selectedIndex].text  // display text of selection

// Select (multiple)
const selected = [...select.selectedOptions].map(o => o.value);

// Textarea
textarea.value         // the full text content
```

### FormData — all fields at once

```js
const form = document.querySelector("form");
const fd   = new FormData(form);

// Read individual fields
fd.get("email");              // first value for "email"
fd.getAll("tags");            // array — for multi-select / multiple checkboxes
fd.has("newsletter");         // boolean

// Iterate all fields
for (const [name, value] of fd) {
  console.log(name, value);
}

// Convert to plain object (single-value fields)
const obj = Object.fromEntries(fd);

// Convert to JSON
const json = JSON.stringify(Object.fromEntries(fd));

// Send directly with fetch (multipart/form-data, supports files)
await fetch("/api/signup", { method: "POST", body: fd });

// Or send as JSON
await fetch("/api/signup", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(Object.fromEntries(fd))
});
```

**Gotcha:** `Object.fromEntries(fd)` loses multi-value fields (checkboxes with the same name). Use `fd.getAll("fieldName")` explicitly for those.

---

## 5. Form Submission — the full pattern

```js
const form = document.querySelector("#contact-form");
const submitBtn = form.querySelector("[type='submit']");

form.addEventListener("submit", async (e) => {
  e.preventDefault();

  // Disable button to prevent double-submit
  submitBtn.disabled = true;
  submitBtn.textContent = "Sending…";

  try {
    const fd  = new FormData(form);
    const res = await fetch("/api/contact", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(Object.fromEntries(fd))
    });

    if (!res.ok) {
      const err = await res.json();
      throw new Error(err.message || "Server error");
    }

    showSuccess("Message sent!");
    form.reset();

  } catch (err) {
    showError(err.message);
  } finally {
    submitBtn.disabled = false;
    submitBtn.textContent = "Send";
  }
});
```

---

## 6. Validation

### Three layers of validation

| Layer | Where | When to use |
|---|---|---|
| HTML attributes | Browser | First line of defence — zero JS needed |
| Constraint Validation API | Browser + JS | Customise native errors, check validity in JS |
| Custom JS validation | JS | Complex rules, cross-field, async (username taken) |

### HTML attribute validation

```html
<input type="email" required minlength="5" maxlength="100">
<input type="number" min="1" max="100" step="1">
<input type="text" pattern="[A-Za-z]{3,}" title="At least 3 letters">
<input type="url" required>
<input type="password" required minlength="8">
```

The browser validates these on submit automatically. No JS required.

### Constraint Validation API

```js
const emailInput = document.querySelector("#email");

// Check validity without showing UI
emailInput.validity.valid        // overall — true/false
emailInput.validity.valueMissing // required but empty
emailInput.validity.typeMismatch // wrong type (email, url)
emailInput.validity.patternMismatch // doesn't match pattern=""
emailInput.validity.tooShort     // below minlength
emailInput.validity.tooLong      // above maxlength
emailInput.validity.rangeUnderflow // below min
emailInput.validity.rangeOverflow  // above max
emailInput.validity.stepMismatch   // doesn't match step
emailInput.validity.customError    // setCustomValidity was used

emailInput.validationMessage  // localised browser error string

// Set a custom error message
emailInput.setCustomValidity("That email is already taken");
emailInput.reportValidity();   // show the native tooltip NOW

// Clear a custom error
emailInput.setCustomValidity(""); // empty string = valid
```

### Custom validation — real-time feedback

```js
const passwordInput = document.querySelector("#password");
const confirmInput  = document.querySelector("#confirm-password");

confirmInput.addEventListener("input", () => {
  if (confirmInput.value !== passwordInput.value) {
    confirmInput.setCustomValidity("Passwords do not match");
  } else {
    confirmInput.setCustomValidity(""); // clear error
  }
});

// Show error only after the user has interacted with the field
confirmInput.addEventListener("blur", () => {
  confirmInput.reportValidity();
});
```

### Custom validation — full validation engine

```js
const validators = {
  required:   (v)    => v.trim() !== ""           || "This field is required",
  email:      (v)    => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) || "Enter a valid email",
  minLength:  (v, n) => v.length >= n             || `Minimum ${n} characters`,
  maxLength:  (v, n) => v.length <= n             || `Maximum ${n} characters`,
  matches:    (v, sel) => v === document.querySelector(sel).value || "Fields must match",
};

function validateField(input) {
  const rules = input.dataset.validate?.split(" ") ?? [];

  for (const rule of rules) {
    const [name, arg] = rule.split(":");
    const result = validators[name]?.(input.value, arg);

    if (result !== true) {
      showFieldError(input, result);
      return false;
    }
  }

  clearFieldError(input);
  return true;
}

function showFieldError(input, message) {
  input.setAttribute("aria-invalid", "true");
  const errEl = input.parentElement.querySelector(".error-msg");
  if (errEl) errEl.textContent = message;
}

function clearFieldError(input) {
  input.removeAttribute("aria-invalid");
  const errEl = input.parentElement.querySelector(".error-msg");
  if (errEl) errEl.textContent = "";
}
```

Usage in HTML:
```html
<input id="email" data-validate="required email">
<input id="password" data-validate="required minLength:8">
<input id="confirm" data-validate="required matches:#password">
```

---

## 7. The `form` Element's Own API

The `HTMLFormElement` has convenience properties you might not know about:

```js
const form = document.querySelector("form");

// Named access — get input by name attribute
form.elements["username"]    // HTMLInputElement
form.elements.username       // same thing
form.elements[0]             // by index

// Iterate all controls
[...form.elements].forEach(el => console.log(el.name, el.value));

// Length
form.elements.length         // number of controls

// Reset all fields to their default values
form.reset();

// Trigger native validation and submit if valid
form.requestSubmit();        // fires the submit event (unlike form.submit())

// Check if form is valid without submitting
form.checkValidity();        // true/false — doesn't show UI
form.reportValidity();       // true/false — DOES show native UI
```

`form.submit()` vs `form.requestSubmit()`:
- `form.submit()` bypasses the `submit` event and all validation — **avoid it**
- `form.requestSubmit()` fires the `submit` event normally — use this when you need to trigger submission programmatically

---

## 8. Different Input Types

### Checkboxes — single

```js
const terms = document.querySelector("#accept-terms");
const accepted = terms.checked; // boolean
```

### Checkboxes — group (same name)

```html
<input type="checkbox" name="skills" value="js">
<input type="checkbox" name="skills" value="python">
<input type="checkbox" name="skills" value="rust">
```

```js
// FormData — use getAll, not get
const fd     = new FormData(form);
const skills = fd.getAll("skills"); // ["js", "rust"] — only checked ones

// Direct DOM query
const checked = [...document.querySelectorAll('input[name="skills"]:checked')]
  .map(el => el.value);
```

### Radio buttons

```html
<input type="radio" name="plan" value="free">
<input type="radio" name="plan" value="pro">
<input type="radio" name="plan" value="enterprise">
```

```js
// FormData handles this naturally
const plan = new FormData(form).get("plan"); // "pro"

// Direct DOM query
const plan = document.querySelector('input[name="plan"]:checked')?.value;
```

### File inputs

```js
const fileInput = document.querySelector("#avatar");

fileInput.addEventListener("change", () => {
  const file = fileInput.files[0]; // FileList — zero-indexed
  if (!file) return;

  console.log(file.name);    // "photo.jpg"
  console.log(file.size);    // bytes
  console.log(file.type);    // "image/jpeg"

  // Preview an image before upload
  const url = URL.createObjectURL(file);
  document.querySelector("#preview").src = url;
  // Clean up when done: URL.revokeObjectURL(url)
});

// Upload via FormData (sends as multipart/form-data)
const fd = new FormData();
fd.append("avatar", fileInput.files[0]);
await fetch("/api/upload-avatar", { method: "POST", body: fd });
```

### Range and color inputs

```js
const range = document.querySelector('input[type="range"]');
range.addEventListener("input", () => {
  document.querySelector("#volume-display").textContent = range.value;
});

const color = document.querySelector('input[type="color"]');
color.addEventListener("input", () => {
  document.body.style.setProperty("--accent", color.value);
});
```

### Date and time inputs

```js
const dateInput = document.querySelector('input[type="date"]');
dateInput.value; // "2026-05-06" — always ISO format (YYYY-MM-DD)

// Set programmatically
dateInput.value = new Date().toISOString().split("T")[0];

// Min/max
dateInput.min = "2026-01-01";
dateInput.max = "2026-12-31";
```

---

## 9. Real-World Examples

### Example 1 — Signup form with async validation

```js
class SignupForm {
  constructor(formEl) {
    this.form     = formEl;
    this.debounce = {};
    this._bindEvents();
  }

  _bindEvents() {
    this.form.addEventListener("submit", this._onSubmit.bind(this));

    // Real-time async username check (debounced)
    const usernameInput = this.form.elements["username"];
    usernameInput.addEventListener("input", () => this._checkUsername(usernameInput));

    // Real-time password match check
    const confirmInput = this.form.elements["confirm_password"];
    confirmInput.addEventListener("input", () => this._checkMatch(confirmInput));
  }

  _checkUsername(input) {
    clearTimeout(this.debounce.username);
    this.debounce.username = setTimeout(async () => {
      const username = input.value.trim();
      if (!username) return;

      const res  = await fetch(`/api/check-username?u=${encodeURIComponent(username)}`);
      const data = await res.json();

      if (!data.available) {
        input.setCustomValidity("Username is already taken");
        this._showError(input, "Username is already taken");
      } else {
        input.setCustomValidity("");
        this._clearError(input);
      }
    }, 400);
  }

  _checkMatch(confirmInput) {
    const password = this.form.elements["password"].value;
    if (confirmInput.value !== password) {
      confirmInput.setCustomValidity("Passwords do not match");
    } else {
      confirmInput.setCustomValidity("");
    }
  }

  async _onSubmit(e) {
    e.preventDefault();
    if (!this.form.checkValidity()) {
      this.form.reportValidity();
      return;
    }

    const data = Object.fromEntries(new FormData(this.form));
    delete data.confirm_password; // don't send this to server

    try {
      const res = await fetch("/api/signup", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
      });
      if (!res.ok) throw new Error((await res.json()).message);
      window.location.href = "/dashboard";
    } catch (err) {
      this._showFormError(err.message);
    }
  }

  _showError(input, msg) {
    input.setAttribute("aria-invalid", "true");
    input.nextElementSibling?.classList.contains("error-msg") &&
      (input.nextElementSibling.textContent = msg);
  }

  _clearError(input) {
    input.removeAttribute("aria-invalid");
    input.nextElementSibling?.classList.contains("error-msg") &&
      (input.nextElementSibling.textContent = "");
  }

  _showFormError(msg) {
    const el = this.form.querySelector(".form-error");
    if (el) { el.textContent = msg; el.hidden = false; }
  }
}

new SignupForm(document.querySelector("#signup-form"));
```

### Example 2 — Multi-step form wizard

```js
class FormWizard {
  constructor(formEl) {
    this.form    = formEl;
    this.steps   = [...formEl.querySelectorAll("[data-step]")];
    this.current = 0;
    this._showStep(0);
    this._bindEvents();
  }

  _bindEvents() {
    this.form.addEventListener("click", (e) => {
      if (e.target.matches("[data-action='next']"))  this._next();
      if (e.target.matches("[data-action='back']"))  this._back();
    });

    this.form.addEventListener("submit", (e) => {
      e.preventDefault();
      this._submit();
    });
  }

  _showStep(index) {
    this.steps.forEach((step, i) => {
      step.hidden = i !== index;
    });
    this.form.querySelector(".step-indicator").textContent =
      `Step ${index + 1} of ${this.steps.length}`;
  }

  _validateCurrentStep() {
    const currentStep   = this.steps[this.current];
    const inputs        = [...currentStep.querySelectorAll("input, select, textarea")];
    return inputs.every(input => {
      const valid = input.checkValidity();
      if (!valid) input.reportValidity();
      return valid;
    });
  }

  _next() {
    if (!this._validateCurrentStep()) return;
    if (this.current < this.steps.length - 1) {
      this.current++;
      this._showStep(this.current);
    }
  }

  _back() {
    if (this.current > 0) {
      this.current--;
      this._showStep(this.current);
    }
  }

  async _submit() {
    if (!this._validateCurrentStep()) return;
    const data = Object.fromEntries(new FormData(this.form));
    await fetch("/api/onboarding", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data)
    });
  }
}
```

### Example 3 — File upload with drag-and-drop + preview

```js
const dropzone   = document.querySelector("#dropzone");
const fileInput  = document.querySelector("#file-input");
const previewEl  = document.querySelector("#preview-grid");

// Click to open file picker
dropzone.addEventListener("click", () => fileInput.click());

// Drag-and-drop
dropzone.addEventListener("dragover",  (e) => { e.preventDefault(); dropzone.classList.add("dragover"); });
dropzone.addEventListener("dragleave", ()  => dropzone.classList.remove("dragover"));
dropzone.addEventListener("drop", (e) => {
  e.preventDefault();
  dropzone.classList.remove("dragover");
  handleFiles([...e.dataTransfer.files]);
});

// File input change
fileInput.addEventListener("change", () => handleFiles([...fileInput.files]));

function handleFiles(files) {
  const allowed = ["image/jpeg", "image/png", "image/webp"];
  const maxSize = 5 * 1024 * 1024; // 5 MB

  for (const file of files) {
    if (!allowed.includes(file.type)) {
      showError(`${file.name}: unsupported type`);
      continue;
    }
    if (file.size > maxSize) {
      showError(`${file.name}: exceeds 5 MB limit`);
      continue;
    }
    addPreview(file);
    uploadFile(file);
  }
}

function addPreview(file) {
  const url = URL.createObjectURL(file);
  const img  = document.createElement("img");
  img.src    = url;
  img.alt    = file.name;
  img.addEventListener("load", () => URL.revokeObjectURL(url)); // free memory
  previewEl.appendChild(img);
}

async function uploadFile(file) {
  const fd = new FormData();
  fd.append("file", file);

  const res  = await fetch("/api/files", { method: "POST", body: fd });
  const data = await res.json();
  console.log("Uploaded:", data.url);
}
```

### Example 4 — Search form with URL state

```js
// Sync form state with URL query params (browser back/forward works)
const form        = document.querySelector("#search-form");
const resultsEl   = document.querySelector("#results");

// On load — populate form from URL
const params = new URLSearchParams(window.location.search);
if (params.get("q"))        form.elements["q"].value = params.get("q");
if (params.get("category")) form.elements["category"].value = params.get("category");
if (params.get("q"))        performSearch(params);

// On submit — update URL and search
form.addEventListener("submit", async (e) => {
  e.preventDefault();
  const fd     = new FormData(form);
  const params = new URLSearchParams(fd);

  // Update URL without reload
  history.pushState({}, "", `?${params.toString()}`);
  await performSearch(params);
});

// On browser back/forward
window.addEventListener("popstate", () => {
  const params = new URLSearchParams(window.location.search);
  if (params.get("q")) form.elements["q"].value = params.get("q");
  performSearch(params);
});

async function performSearch(params) {
  const res  = await fetch(`/api/search?${params.toString()}`);
  const data = await res.json();
  renderResults(data.results);
}
```

---

## 10. Tricky Things

### 1. `input.value` is always a string — even for `type="number"`

```js
const qty = document.querySelector('#qty');  // type="number"
qty.value;          // "5" — string
typeof qty.value;   // "string"

// ✅ Convert explicitly
const n = Number(qty.value);   // or +qty.value, or parseInt(qty.value, 10)
```

### 2. `form.reset()` restores default values — not empty values

```js
<input name="role" value="user">   <!-- default value -->

// After form.reset():
form.elements["role"].value  // "user" — back to default, not ""
```

### 3. `Object.fromEntries(new FormData(form))` loses multi-value fields

```js
// HTML: three checkboxes named "skills", two are checked
const obj = Object.fromEntries(new FormData(form));
obj.skills; // "rust" — only the LAST checked value!

// ✅ Use getAll() for multi-value fields
const skills = new FormData(form).getAll("skills"); // ["js", "rust"]
```

### 4. `form.submit()` skips the submit event and validation

```js
// ❌ Bypasses your submit listener and all validation
form.submit();

// ✅ Fires normally through the event system
form.requestSubmit();
```

### 5. Autofill can change `value` without firing `input`

Some browsers autofill inputs without firing `input` or `change`. Check values on submit, not just during live validation.

### 6. `disabled` inputs are excluded from `FormData`

```js
<input name="userId" value="42" disabled>  <!-- excluded from FormData -->

// ✅ Use readonly instead if you want to include the value
<input name="userId" value="42" readonly>  <!-- included in FormData -->
```

---

## 11. Common Mistakes

### Mistake 1 — Not preventing default on submit

```js
// ❌ Page reloads, fetch never completes
form.addEventListener("submit", async (e) => {
  const data = new FormData(form);
  await fetch("/api", { method: "POST", body: data });
});

// ✅ Always preventDefault first
form.addEventListener("submit", async (e) => {
  e.preventDefault();
  // ...
});
```

### Mistake 2 — Reading `.value` from a checkbox to get checked state

```js
// ❌ checkbox.value is the HTML value attribute, not whether it's checked
if (checkbox.value) { ... }  // always truthy unless value="" explicitly

// ✅ Use .checked
if (checkbox.checked) { ... }
```

### Mistake 3 — Double submission not prevented

```js
// ❌ User clicks Submit twice — two POST requests go out
form.addEventListener("submit", async (e) => {
  e.preventDefault();
  await fetch("/api/order", { method: "POST", body: new FormData(form) });
});

// ✅ Disable the button during the async operation
form.addEventListener("submit", async (e) => {
  e.preventDefault();
  const btn = form.querySelector("[type='submit']");
  btn.disabled = true;
  try {
    await fetch("/api/order", { method: "POST", body: new FormData(form) });
  } finally {
    btn.disabled = false;
  }
});
```

### Mistake 4 — Validating only on submit (poor UX)

```js
// ❌ User fills out 10 fields then gets all errors at once on submit
form.addEventListener("submit", validateAll);

// ✅ Validate individual fields on blur, full form on submit
input.addEventListener("blur", () => validateField(input));
form.addEventListener("submit", (e) => {
  e.preventDefault();
  if (validateAll()) submitForm();
});
```

### Mistake 5 — Sending passwords in query strings

```js
// ❌ Password visible in browser history, server logs, proxy logs
await fetch(`/api/login?password=${password}`);

// ✅ Always POST sensitive data in the request body
await fetch("/api/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ username, password })
});
```

### Mistake 6 — Not sanitising before inserting form values into the DOM

```js
// ❌ XSS if value contains <script> or <img onerror=...>
resultEl.innerHTML = `Hello, ${form.elements["name"].value}`;

// ✅ Use textContent for plain text
resultEl.textContent = `Hello, ${form.elements["name"].value}`;
```

---

## 12. Practice Exercises

### Easy — Live character counter

Build a `<textarea>` with a `maxlength="280"`. Show a live counter below: "210 / 280 characters". Change the counter colour to orange when > 200, red when > 260.

Requirements:
- Use the `input` event
- Update a `<span>` below the textarea
- Use `classList.toggle` for colour states

---

### Medium — Multi-field registration form with validation

Build a registration form with:
- Username (required, 3–20 chars, letters/numbers only — pattern)
- Email (required, valid email)
- Password (required, min 8 chars)
- Confirm password (must match password)
- Country (required `<select>`)
- Terms checkbox (required)

Requirements:
- Show inline error messages below each field on blur
- Only show errors after the user has touched the field (use a `data-touched` attribute)
- On submit, validate all fields — prevent submission if any are invalid
- On successful validation, log a JSON object of all values to console

---

### Hard — Dynamic invoice builder

Build a form that lets the user create a list of line items and calculates totals.

Requirements:
- "Add item" button appends a new row: [Description input] [Qty input] [Unit price input] [× remove button]
- Each row calculates its subtotal live (`qty × price`) and displays it
- A footer shows the running total of all subtotals
- Remove button deletes the row and updates the total
- Submit sends the invoice as JSON to `/api/invoices` (log it to console)
- All qty and price inputs must be positive numbers — show inline errors if not

---

## 13. Quick Reference

```
READING VALUES
  input.value                     text, number, email, etc. (always string)
  input.checked                   checkbox / radio — boolean
  select.value                    selected option
  [...select.selectedOptions].map(o => o.value)   multi-select
  document.querySelector('input[name="x"]:checked')?.value   radio group
  new FormData(form).get("name")  any field by name
  new FormData(form).getAll("name") multi-value (checkboxes, multi-select)
  Object.fromEntries(new FormData(form))  all fields as plain object

FORM METHODS
  form.reset()                    restore all fields to defaults
  form.checkValidity()            true/false — no UI shown
  form.reportValidity()           true/false — shows native tooltips
  form.requestSubmit()            trigger submit event + validation
  form.elements["name"]           get control by name
  [...form.elements]              iterate all controls

CONSTRAINT VALIDATION
  input.validity.valid            overall
  input.validity.valueMissing     required but empty
  input.validity.typeMismatch     wrong type
  input.validity.patternMismatch  fails pattern=""
  input.validity.tooShort / tooLong
  input.validity.rangeUnderflow / rangeOverflow
  input.setCustomValidity("msg")  set error ("" to clear)
  input.reportValidity()          show tooltip immediately

EVENTS
  submit     form submitted (use e.preventDefault())
  input      every keystroke (real-time)
  change     value changed + focus left
  focus/blur field gains/loses focus (doesn't bubble)
  focusin/focusout  same but bubbles

SECURITY
  Never send passwords in query strings
  Never use innerHTML with user input — use textContent
  disabled inputs are NOT included in FormData — use readonly
```

---

## 14. Connected Topics

- **68 — Events** — `submit`, `input`, `change`, `focus`, `blur` — all the events used in forms
- **69 — Event delegation** — delegated validation listeners on a parent form element
- **58 — Fetch API** — submitting form data via `fetch` instead of page reload
- **71 — Types of JS errors** — handling validation errors and network errors
- **72 — try/catch/finally** — wrapping async form submission
- **73 — Custom error classes** — `ValidationError`, `NetworkError` in form submit handlers
- **Node.js / Express** — `req.body` mirrors `FormData`, `express-validator` mirrors the Constraint Validation API
