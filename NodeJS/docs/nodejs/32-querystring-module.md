# 32 — querystring module

## What is this?

`querystring` is one of Node's oldest built-in modules — it converts a URL query string like `"userId=42&role=admin"` into a plain JavaScript object `{ userId: '42', role: 'admin' }`, and back again. Think of it as a translator between two shapes of the same data: the flat text format that travels inside a URL, and the structured object format your JavaScript code actually wants to work with. It predates the modern `URLSearchParams` API by many years and is now considered **legacy** — still shipped with Node, still works, but no longer the recommended tool for new code.

## Why does it matter for backend development?

Even though `querystring` is legacy, you will still see it in older codebases, in some framework internals, and in interview questions that test whether you understand *why* Node moved away from it. A backend dev needs to recognize `qs.parse()` and `qs.stringify()` on sight, know exactly how they differ from the modern `URLSearchParams` class, and know which one to reach for in new code (spoiler: almost always `URLSearchParams` or `URL`). Understanding this module also teaches a core backend lesson: query strings are just serialized objects, and serialization always has edge cases — arrays, special characters, duplicate keys — that trip up developers who don't understand the underlying format.

---

## Syntax / API

```js
// Import the built-in querystring module — no npm install needed
const qs = require('querystring');

// ── parse() — turns a query string into a plain object ─────────────────────
const parsed = qs.parse('userId=42&role=admin&role=editor');
// → { userId: '42', role: [ 'admin', 'editor' ] }
// NOTE: duplicate keys automatically become an array — no config needed

// ── stringify() — turns a plain object into a query string ─────────────────
const built = qs.stringify({ userId: 42, role: 'admin', active: true });
// → 'userId=42&role=admin&active=true'
// NOTE: numbers/booleans are auto-converted to strings, no explicit cast needed

// ── Custom separators — parse()/stringify() accept sep and eq as 2nd/3rd args
const custom = qs.parse('userId:42;role:admin', ';', ':');
// → { userId: '42', role: 'admin' }   (uses ';' instead of '&', ':' instead of '=')

// ── escape() / unescape() — the encode/decode helpers used internally ──────
console.log(qs.escape('hello world & more'));   // → 'hello%20world%20%26%20more'
console.log(qs.unescape('hello%20world'));      // → 'hello world'
```

---

## How it works — line by line

`qs.parse(str)` scans the string for `&` characters to split it into key-value pairs, then splits each pair on the first `=` sign, then URL-decodes both the key and the value. If the same key appears more than once, `parse()` silently collects all of its values into an array instead of overwriting the earlier one — this is different from plain object literals, where a duplicate key would just overwrite the previous value.

`qs.stringify(obj)` does the reverse: it walks every own property on the object, URL-encodes the key and value, joins them with `=`, and joins all the pairs with `&`. If a property's value is an array, `stringify()` repeats the key once per array element (e.g. `role=admin&role=editor`) rather than producing a single combined value.

Both functions operate on **plain strings and plain objects only** — they know nothing about the rest of a URL (protocol, host, path, hash). That is the single biggest limitation compared to the modern approach: `querystring` only ever deals with the part after the `?`, and you are responsible for slicing that part out of the full URL yourself.

---

## Example 1 — basic

```js
// Import the legacy querystring module
const qs = require('querystring');

// A raw query string exactly as it would appear after the '?' in a URL
const rawQuery = 'search=node.js&page=2&sort=recent&sort=popular';

// parse() converts it into a usable object
const queryObject = qs.parse(rawQuery);
console.log(queryObject);
// → { search: 'node.js', page: '2', sort: [ 'recent', 'popular' ] }
// Notice: 'page' stayed a STRING '2', not a number — parse() never infers types

// Access individual fields like any normal object
console.log('Search term:', queryObject.search);   // 'node.js'
console.log('Page number:', Number(queryObject.page)); // manual conversion to number: 2

// stringify() does the reverse — object back into a query string
const rebuilt = qs.stringify({
  search: 'express js',   // spaces will be encoded automatically
  page: 1,                // numbers are converted to strings
  active: true,           // booleans are converted to strings too
});
console.log(rebuilt);
// → 'search=express%20js&page=1&active=true'
```

---

## Example 2 — real world backend use case

```js
// File: src/utils/legacyQueryParser.js
// Scenario: maintaining an older internal service that still uses querystring
// to parse raw request URLs received from an http.Server (no framework).

const http = require('http');
const qs   = require('querystring');
const url  = require('url'); // legacy url.parse() pairs naturally with querystring

const server = http.createServer((req, res) => {
  // url.parse(req.url, true) splits path from query and auto-parses the query
  // the 'true' flag tells it to run querystring.parse() on the query part for us
  const parsedUrl = url.parse(req.url, true);

  const pathname     = parsedUrl.pathname;   // e.g. '/api/users'
  const queryObject  = parsedUrl.query;      // already an object, thanks to 'true' flag

  if (pathname === '/api/users') {
    // Extract expected query params with safe fallbacks
    const page     = Number(queryObject.page) || 1;      // default to page 1
    const pageSize = Number(queryObject.pageSize) || 20; // default page size
    const roleFilter = queryObject.role; // could be a string OR an array (duplicate keys)

    // Normalize roleFilter into an array either way — a common gotcha (see Mistake 1)
    const roles = Array.isArray(roleFilter)
      ? roleFilter
      : roleFilter ? [roleFilter] : [];

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      page,
      pageSize,
      roles,
      // Demonstrate stringify() by echoing back a "next page" link
      nextPageLink: '/api/users?' + qs.stringify({
        ...queryObject,          // keep existing filters
        page: page + 1,          // bump the page number
      }),
    }));
    return;
  }

  res.writeHead(404);
  res.end('Not found');
});

server.listen(3000, () => {
  console.log('Legacy server listening on http://localhost:3000');
  // Try: http://localhost:3000/api/users?page=1&role=admin&role=editor
});

module.exports = server;
```

---

## Common mistakes

### Mistake 1 — Assuming a query param is always a single string

```js
// ❌ WRONG — treating queryObject.role as always a plain string
const qs = require('querystring');
const queryObject = qs.parse('role=admin&role=editor'); // duplicate key!

// Crashes or misbehaves: queryObject.role is actually an ARRAY here
if (queryObject.role === 'admin') {
  console.log('User is admin'); // never runs — role is ['admin', 'editor'], not a string
}

// ✅ CORRECT — always normalize to an array before checking values
const rolesArray = Array.isArray(queryObject.role)
  ? queryObject.role
  : [queryObject.role].filter(Boolean); // wrap single value, drop undefined

if (rolesArray.includes('admin')) {
  console.log('User is admin'); // runs correctly regardless of single or duplicate keys
}
```

### Mistake 2 — Reaching for querystring in new code instead of URLSearchParams

```js
// ❌ WRONG — using the legacy module for a brand-new feature in 2026
const qs = require('querystring');
const queryString = qs.stringify({ userId: 42, includeDeleted: false });
// Works, but querystring is a legacy module — no longer actively developed,
// and it can't be combined with a full URL object the way URLSearchParams can

// ✅ CORRECT — modern code should use URLSearchParams (global, no require needed)
const params = new URLSearchParams({ userId: '42', includeDeleted: 'false' });
const queryString2 = params.toString();
// → 'userId=42&includeDeleted=false'
// Bonus: URLSearchParams integrates directly with the WHATWG URL class
const requestUrl = new URL('https://api.example.com/orders');
requestUrl.search = params.toString();
console.log(requestUrl.href); // → full URL with query string attached, built correctly
```

### Mistake 3 — Expecting parse() to convert types (numbers, booleans)

```js
// ❌ WRONG — comparing a parsed value directly against a number
const qs = require('querystring');
const queryObject = qs.parse('page=2&active=true');

if (queryObject.page === 2) {           // '2' === 2 is FALSE — page is a string!
  console.log('On page 2');             // never runs
}
if (queryObject.active === true) {      // 'true' === true is FALSE — also a string!
  console.log('Filter is active');      // never runs
}

// ✅ CORRECT — explicitly convert types after parsing, every single time
const page   = Number(queryObject.page);              // '2' → 2
const active = queryObject.active === 'true';          // 'true' string → real boolean

if (page === 2) console.log('On page 2');              // runs correctly
if (active === true) console.log('Filter is active');  // runs correctly
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that takes the raw query string `"userId=101&sessionId=abc123&remember=true"` and:
1. Parses it with `qs.parse()`
2. Logs the resulting object
3. Converts `remember` into a real boolean and logs the converted value
4. Uses `qs.stringify()` to turn a NEW object `{ userId: 101, remember: false }` back into a query string and logs it

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `buildPaginationLink(basePath, currentQuery, direction)` that:
1. Accepts `basePath` (e.g. `'/api/products'`), a parsed query object `currentQuery` (e.g. `{ page: '3', pageSize: '10', category: 'shoes' }`), and `direction` which is either `'next'` or `'prev'`
2. Computes the new page number (current page + 1 for `'next'`, current page − 1 for `'prev'`, but never below 1)
3. Builds a full link string like `/api/products?page=4&pageSize=10&category=shoes` using `qs.stringify()` — keeping every other query param unchanged
4. Returns the built link

Test it with at least two calls (one `'next'`, one `'prev'`, including a `'prev'` case starting at page 1 to verify it doesn't go below 1) and log the results.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small module `queryNormalizer.js` that exports a function `normalizeQuery(rawQueryString)` which:
1. Parses the raw query string using `qs.parse()`
2. For every key whose value is a comma-separated string (e.g. `tags=node,express,api`), splits it into an actual array of trimmed strings
3. For every key whose value is already an array (from duplicate keys), leaves it as an array but trims each entry
4. Converts any value that looks like a valid number (using a regex or `!isNaN`) into an actual JavaScript number
5. Converts the exact strings `'true'` and `'false'` into real booleans
6. Returns the fully normalized object

Test it against this raw string and log the final object, verifying every field has the correct JS type (not just strings):
```
"page=2&pageSize=20&tags=node,express,api&role=admin&role=editor&active=true&featured=false"
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
IMPORT
  const qs = require('querystring');   // built-in, no install needed

CORE METHODS
  qs.parse(str)              → string  → object
  qs.stringify(obj)          → object  → string
  qs.escape(str)              → URL-encode a single string
  qs.unescape(str)            → URL-decode a single string

CUSTOM SEPARATORS
  qs.parse(str, sep, eq)      → e.g. qs.parse('a:1;b:2', ';', ':')
  qs.stringify(obj, sep, eq)  → same custom separator/assignment chars

KEY BEHAVIORS
  Duplicate keys on parse()   → collected into an ARRAY automatically
  Array values on stringify() → repeated as multiple key=value pairs
  ALL parsed values           → always STRINGS (or arrays of strings) — never numbers/booleans
  Only handles the QUERY part → knows nothing about protocol/host/path

LEGACY vs MODERN
  querystring.parse()  →  Object.fromEntries(new URLSearchParams(str))  (roughly)
  querystring.stringify() →  new URLSearchParams(obj).toString()
  URLSearchParams        →  global (no require), integrates with URL class,
                            has .get()/.getAll()/.set()/.append()/.delete()/.sort()
  querystring            →  Node-only, older API, still shipped but NOT recommended
                            for new code — Node docs call it "Legacy"

WHEN TO USE WHICH
  New code                    → URLSearchParams / new URL()
  Maintaining old codebases   → querystring (recognize it, don't fear it)
  Need custom separators      → querystring (URLSearchParams only supports & and =)

GOTCHAS
  '2' === 2           → false   (always cast parsed values explicitly)
  'true' === true     → false   (compare against the string, or cast manually)
  role=a&role=a again → { role: ['a','a'] } — duplicates are kept, not deduped
```

---

## Connected topics

- **22 — url module** — `new URL()` and `URLSearchParams` are the modern replacement for everything `querystring` does, with tighter integration into full URL parsing
- **20 — http module** — query strings arrive as part of `req.url` on every raw HTTP request; this is where `querystring`/`URLSearchParams` gets used in practice
- **54 — Express routing in depth** — Express exposes parsed query params on `req.query` using its own parser, but understanding the raw query string format underneath makes debugging edge cases (arrays, encoding) much easier
