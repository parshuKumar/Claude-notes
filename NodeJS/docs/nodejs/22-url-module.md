# 22 — url module

## What is this?

The `url` module is Node's built-in tool for parsing, building, and manipulating URLs. Instead of splitting strings on `?`, `&`, and `=` by hand, you get a `URL` object that breaks a link into clean, structured pieces — protocol, host, path, and query parameters — and a `URLSearchParams` object for working with query strings safely. Think of it like a mail-sorting machine: you drop in a raw address string and it hands you back neatly labeled slots for street, city, zip, and any special delivery instructions, instead of you scanning the string character by character.

## Why does it matter for backend development?

Every incoming HTTP request has a URL, and most APIs rely on query strings for filtering, pagination, and search (`/products?category=shoes&page=2&sort=price`). A backend developer needs to reliably read those parameters, validate them, and — just as often — build outgoing URLs for redirects, webhooks, pagination links, or third-party API calls (adding an `apiKey`, encoding a `redirectUrl`, appending a `sessionId`). Doing this with manual string splitting is error-prone (forgetting to URL-decode `%20` or `+`, missing repeated keys, breaking on trailing slashes). The `url` module handles encoding/decoding, repeated keys, and edge cases correctly, which is why it sits underneath frameworks like Express when they parse `req.query`.

---

## Syntax / API

```js
// url is a built-in Node module — no npm install needed
const { URL, URLSearchParams } = require('url');

// ── Parsing a URL string into a structured object ───────────────────────────
const requestUrl = new URL('https://api.example.com:8080/users/42?active=true&role=admin#profile');

console.log(requestUrl.protocol);   // 'https:'          — the scheme
console.log(requestUrl.hostname);   // 'api.example.com' — host without port
console.log(requestUrl.port);       // '8080'            — port as a string
console.log(requestUrl.pathname);   // '/users/42'        — path only, no query
console.log(requestUrl.search);     // '?active=true&role=admin' — raw query string
console.log(requestUrl.hash);       // '#profile'        — fragment identifier
console.log(requestUrl.origin);     // 'https://api.example.com:8080'

// ── Reading query parameters via searchParams ────────────────────────────────
console.log(requestUrl.searchParams.get('active'));   // 'true' (always a string)
console.log(requestUrl.searchParams.has('role'));      // true
console.log(requestUrl.searchParams.getAll('role'));   // ['admin'] — array, handles repeated keys

// ── Building a URL programmatically ──────────────────────────────────────────
const outgoingUrl = new URL('https://api.stripe.com/v1/charges');
outgoingUrl.searchParams.set('limit', '10');       // add or overwrite a param
outgoingUrl.searchParams.append('status', 'paid'); // add without removing existing
console.log(outgoingUrl.toString());
// → 'https://api.stripe.com/v1/charges?limit=10&status=paid'

// ── Parsing a bare query string (no full URL needed) ─────────────────────────
const params = new URLSearchParams('page=2&limit=20&sort=-createdAt');
console.log(params.get('page'));   // '2'
console.log(Object.fromEntries(params)); // { page: '2', limit: '20', sort: '-createdAt' }
```

---

## How it works — line by line

`new URL(string)` takes any full URL string and immediately validates and splits it into properties — protocol, hostname, port, pathname, search, and hash — throwing a `TypeError` if the string is not a valid URL. This is a real parse, not a regex hack, so it correctly handles ports, IPv6 hosts, and percent-encoded characters.

`requestUrl.searchParams` is a live `URLSearchParams` object tied to the URL's query string. "Live" means editing it (via `.set()`, `.append()`, `.delete()`) automatically updates `requestUrl.search` and `requestUrl.href` — you never have to manually re-stitch the query string back together.

`URLSearchParams` itself understands the query-string format the way a browser does: `key=value` pairs joined by `&`, with `+` and `%XX` sequences decoded back to real characters. Every value returned by `.get()` is always a plain string — if you expect a number or boolean, you must convert it yourself (`Number(params.get('page'))`).

The difference between `.set()` and `.append()` matters: `.set(key, value)` replaces all existing values for that key, while `.append(key, value)` adds another value alongside any existing ones — necessary for query strings like `?tag=node&tag=backend` where the same key repeats intentionally (read back with `.getAll('tag')`).

Calling `.toString()` (or just template-interpolating the URL object, or reading `.href`) reconstructs the full URL string with all your changes applied and properly percent-encoded.

---

## Example 1 — basic

```js
// Parse an incoming request-like URL and read its pieces
const { URL } = require('url');

// A typical URL a client might hit on your API
const rawUrl = 'https://shop.example.com/api/products?category=shoes&inStock=true&page=2';

const parsedUrl = new URL(rawUrl);

// Structural pieces
console.log('Path       :', parsedUrl.pathname);   // '/api/products'
console.log('Query string:', parsedUrl.search);     // '?category=shoes&inStock=true&page=2'

// Read individual query params — .get() always returns a string or null
const category = parsedUrl.searchParams.get('category');  // 'shoes'
const inStock  = parsedUrl.searchParams.get('inStock');    // 'true' (string, not boolean!)
const page     = parsedUrl.searchParams.get('page');       // '2' (string, not number!)
const missing  = parsedUrl.searchParams.get('brand');       // null — not present

console.log('category:', category);
console.log('inStock :', inStock === 'true');  // convert manually → real boolean
console.log('page    :', Number(page));         // convert manually → real number
console.log('brand   :', missing);              // null

// Loop over every query param without knowing the keys in advance
for (const [key, value] of parsedUrl.searchParams) {
  console.log(`${key} = ${value}`);
}
```

---

## Example 2 — real world backend use case

```js
// File: src/services/paginationLink.js
// Builds "next page" links for a paginated API response — a pattern used in
// almost every REST API (GitHub, Stripe, Shopify all do this).

const { URL } = require('url');

/**
 * Given the current request URL and the current page number,
 * returns the absolute URL for the next page — preserving every
 * other existing query param (filters, sort order, etc).
 */
function buildNextPageUrl(currentUrlString, currentPage) {
  const nextUrl = new URL(currentUrlString);   // parse the incoming request URL

  // Overwrite just the "page" param — everything else stays untouched
  nextUrl.searchParams.set('page', String(currentPage + 1));

  return nextUrl.toString();   // fully reconstructed URL, ready to send to the client
}

// ── Simulating an Express-style request handler ──────────────────────────────
function handleGetProducts(req, res) {
  // req.url is usually just the path + query, e.g. '/api/products?category=shoes&page=1'
  // Building a full URL object requires a base — use the request's host header
  const fullRequestUrl = new URL(req.url, `https://${req.headers.host}`);

  const category = fullRequestUrl.searchParams.get('category') || 'all';
  const page      = Number(fullRequestUrl.searchParams.get('page')) || 1;
  const limit     = Number(fullRequestUrl.searchParams.get('limit')) || 20;

  // ... fetch products from the database using category, page, limit ...
  const products = []; // pretend this came from a dbConnection query

  const responseBody = {
    data: products,
    pagination: {
      page,
      limit,
      nextPage: buildNextPageUrl(fullRequestUrl.toString(), page),
    },
  };

  res.end(JSON.stringify(responseBody));
}

module.exports = { buildNextPageUrl, handleGetProducts };

// ── Another common case: building an outgoing OAuth redirect URL ────────────
function buildOAuthRedirectUrl(clientId, redirectUri, sessionId) {
  const authUrl = new URL('https://accounts.google.com/o/oauth2/v2/auth');

  authUrl.searchParams.set('client_id', clientId);
  authUrl.searchParams.set('redirect_uri', redirectUri);   // auto-encoded correctly
  authUrl.searchParams.set('response_type', 'code');
  authUrl.searchParams.set('scope', 'email profile');
  authUrl.searchParams.set('state', sessionId);             // CSRF protection token

  return authUrl.toString();
  // → 'https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&state=...'
}
```

---

## Common mistakes

### Mistake 1 — Treating searchParams values as their "real" type

```js
// ❌ WRONG — every value from searchParams is a STRING, even 'true' and '2'
const { URL } = require('url');
const parsedUrl = new URL('https://api.example.com/products?inStock=true&page=2');

if (parsedUrl.searchParams.get('inStock')) {
  // BUG: this runs even when inStock is 'false', because 'false' is a truthy string!
  console.log('Filtering in-stock only');
}

// ✅ CORRECT — explicitly convert to the type you need
const inStock = parsedUrl.searchParams.get('inStock') === 'true';   // real boolean
const page    = Number(parsedUrl.searchParams.get('page')) || 1;     // real number, with fallback
```

### Mistake 2 — Passing a relative URL to `new URL()` without a base

```js
// ❌ WRONG — req.url from Node's raw http module is only a path, not a full URL
// Throws: TypeError [ERR_INVALID_URL]: Invalid URL
const { URL } = require('url');
function handleRequest(req) {
  const parsedUrl = new URL(req.url);   // req.url is '/api/users?id=42' — no protocol/host
}

// ✅ CORRECT — pass a base URL as the second argument
function handleRequest(req) {
  const baseUrl   = `https://${req.headers.host}`;   // reconstruct scheme + host
  const parsedUrl = new URL(req.url, baseUrl);         // now it resolves correctly
  console.log(parsedUrl.searchParams.get('id'));       // '42'
}
```

### Mistake 3 — Manually building query strings with string concatenation

```js
// ❌ WRONG — breaks the moment a value contains special characters
const search      = 'node.js & backend';
const redirectUrl = 'https://myapp.com/callback';
const badUrl = `https://api.example.com/search?q=${search}&redirect=${redirectUrl}`;
// → '...?q=node.js & backend&redirect=https://myapp.com/callback'
// The '&' inside the search term breaks parsing, and redirectUrl's own '?'/'&' corrupts the query

// ✅ CORRECT — let URLSearchParams handle encoding for you
const { URL } = require('url');
const goodUrl = new URL('https://api.example.com/search');
goodUrl.searchParams.set('q', search);              // auto-encoded: 'node.js+%26+backend'
goodUrl.searchParams.set('redirect', redirectUrl);   // auto-encoded safely as one value
console.log(goodUrl.toString());
// → 'https://api.example.com/search?q=node.js+%26+backend&redirect=https%3A%2F%2Fmyapp.com%2Fcallback'
```

---

## Practice exercises

### Exercise 1 — easy

Given the URL string `'https://api.example.com/orders/789?status=shipped&express=true'`, write a script that:
1. Parses it with `new URL()`
2. Logs the `pathname`, `hostname`, and full `search` string separately
3. Logs the value of the `status` query param
4. Logs whether the `express` param equals the string `'true'`, converted to a real boolean
5. Logs `null` for a param called `courier` that does not exist in the URL

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `buildSearchUrl(baseUrl, filters)` that:
1. Takes a base URL string (e.g. `'https://api.example.com/products'`) and a plain object of filters (e.g. `{ category: 'electronics', minPrice: 100, inStock: true }`)
2. Builds a `URL` object from the base URL
3. Adds each key/value from `filters` as a query parameter (convert numbers/booleans to strings before setting)
4. Returns the final URL as a string

Then call it with at least two different filter objects and log the results. Also test what happens when `filters` is an empty object — it should just return the base URL with no `?`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `QueryBuilder` class that wraps `URLSearchParams` for constructing complex API request URLs:
1. Constructor takes a `baseUrl` string
2. `.addFilter(key, value)` — sets a single-value query param (overwrites if called again with the same key)
3. `.addMultiple(key, valuesArray)` — appends multiple values for the same key (e.g. `tags=node&tags=backend&tags=api`)
4. `.setPage(pageNumber, limit)` — sets both `page` and `limit` params, converting numbers to strings
5. `.removeFilter(key)` — removes a query param entirely if present
6. `.build()` — returns the final constructed URL as a string
7. `.getParamsObject()` — returns a plain object of all current query params (use `Object.fromEntries`, noting it will only keep the last value for repeated keys — mention this limitation in a comment)

Test it:
```js
const productSearch = new QueryBuilder('https://api.example.com/products');
productSearch.addFilter('category', 'shoes');
productSearch.addMultiple('tags', ['running', 'nike', 'sale']);
productSearch.setPage(2, 25);
console.log(productSearch.build());
productSearch.removeFilter('category');
console.log(productSearch.build());
console.log(productSearch.getParamsObject());
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CREATING A URL
  new URL(urlString)              → parse a full absolute URL, throws if invalid
  new URL(relativePath, baseUrl)  → resolve a relative path (req.url) against a base

URL OBJECT PROPERTIES
  .protocol   → 'https:'                 (includes trailing colon)
  .hostname   → 'api.example.com'         (no port)
  .port       → '8080' or '' if default
  .pathname   → '/users/42'
  .search     → '?active=true'            (raw, includes leading ?)
  .searchParams → live URLSearchParams object tied to .search
  .hash       → '#section'                (includes leading #)
  .href       → the full reconstructed URL string
  .origin     → 'https://api.example.com:8080'

URLSEARCHPARAMS METHODS
  .get(key)          → first value as STRING, or null if missing
  .getAll(key)       → array of ALL values for that key
  .has(key)          → boolean, true if key exists at all
  .set(key, value)   → overwrite (or create) — removes duplicates
  .append(key, value)→ add another value WITHOUT removing existing ones
  .delete(key)       → remove a param entirely
  for (const [k, v] of params) → iterate all key/value pairs
  Object.fromEntries(params)   → convert to plain object (drops duplicate keys!)

GOTCHAS
  Every value from searchParams is a STRING — always convert:
    Number(params.get('page')) || 1
    params.get('active') === 'true'
  req.url from raw http/net servers is a PATH, not a full URL — pass a base:
    new URL(req.url, `https://${req.headers.host}`)
  .searchParams is LIVE — mutating it updates .search and .href automatically
  Never build query strings with string concatenation — encoding will break

TYPICAL PATTERNS
  Read pagination params:
    const page  = Number(url.searchParams.get('page'))  || 1;
    const limit = Number(url.searchParams.get('limit')) || 20;

  Build outgoing URL with params:
    const u = new URL('https://api.example.com/resource');
    u.searchParams.set('key', 'value');
    u.toString();

  Parse a bare query string (no host needed):
    new URLSearchParams('a=1&b=2')
```

---

## Connected topics

- **20 — http module** — the `req.url` you get in a raw HTTP server handler is exactly what you feed into `new URL(req.url, baseUrl)` to parse query strings and paths
- **32 — querystring module** — the older, Node-specific way to parse query strings; `URLSearchParams`/`URL` is the modern, spec-compliant (WHATWG) replacement covered here
- **54 — Express routing in depth** — Express's `req.query` object is built internally using the same query-string parsing concepts this topic covers
