# 58 — fetch API

## What is this?

The `fetch` API is the **modern, built-in way to make HTTP requests** from JavaScript — both in the browser and in Node.js (v18+). It replaces the old `XMLHttpRequest` (XHR) API and returns Promises, making it work naturally with async/await. Think of `fetch` as a polite postal worker: you hand them a request (URL + options), they go off and handle the network communication, and they come back with a `Response` object — a sealed envelope you have to explicitly open to read what's inside.

## Why does it matter?

Every backend developer makes HTTP requests constantly — to external APIs, microservices, authentication servers, payment gateways, third-party services. In Node.js backends you'll use `fetch` to call other services. In the browser you'll use it for every API call. Understanding `fetch` completely — including its non-obvious behaviours (HTTP errors don't auto-reject!), request configuration, streaming, AbortController, and authentication — means you can handle any HTTP scenario correctly without reaching for Axios or other libraries as a crutch.

---

## Syntax

```js
// Minimal GET request
const response = await fetch("https://api.example.com/users/1");
const data     = await response.json();

// Full options
const response = await fetch("https://api.example.com/users", {
  method:  "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer eyJhbGci...",
  },
  body: JSON.stringify({ name: "Alice", email: "alice@example.com" }),
});
```

---

## How it works — step by step

```js
const response = await fetch(url, options);
```
1. Creates an HTTP request with the given URL and options
2. Returns a Promise that resolves with a `Response` object when the **headers** arrive
3. Note: at this point the **body has NOT been read yet** — only headers

```js
const data = await response.json();
```
4. Reads the response body stream and parses it as JSON
5. Returns a Promise that resolves with the parsed JavaScript object

This two-step process is intentional — it lets you inspect headers before deciding whether to read the body.

---

## The Response object

```js
const response = await fetch("https://api.example.com/data");

// Status information
console.log(response.status);      // 200, 404, 500, etc.
console.log(response.statusText);  // "OK", "Not Found", etc.
console.log(response.ok);          // true if status is 200-299, false otherwise

// Headers
console.log(response.headers.get("Content-Type"));   // "application/json"
console.log(response.headers.get("X-RateLimit-Remaining"));  // "95"

// Reading the body (each method returns a Promise, can only be called ONCE):
const json    = await response.json();     // parse as JSON
const text    = await response.text();     // read as string
const blob    = await response.blob();     // read as Blob (binary data)
const buffer  = await response.arrayBuffer(); // read as ArrayBuffer
const form    = await response.formData(); // read as FormData

// Check if body has been consumed:
console.log(response.bodyUsed);   // true after reading, false before
```

---

## The most important `fetch` gotcha: HTTP errors do NOT reject

This trips up almost every developer new to `fetch`:

```js
// WRONG assumption — this DOES NOT throw on 404 or 500
try {
  const response = await fetch("https://api.example.com/users/99999");
  const data     = await response.json();
  console.log(data);  // might log { error: "Not found" }
} catch (err) {
  console.error("Caught:", err);  // only fires for NETWORK errors
}

// The Promise from fetch() only rejects for:
//   - Network failure (no internet, DNS failure, CORS blocked)
//   - Request aborted (AbortController)
//   - Timeout (if you implement one)
// It does NOT reject for HTTP error status codes (400, 401, 403, 404, 500...)

// CORRECT — always check response.ok
async function safeFetch(url, options) {
  const response = await fetch(url, options);

  if (!response.ok) {
    // Attempt to read error body for a better message:
    let errorMessage;
    try {
      const errorBody = await response.json();
      errorMessage = errorBody.message ?? errorBody.error ?? response.statusText;
    } catch {
      errorMessage = response.statusText;
    }
    throw new Error(`HTTP ${response.status}: ${errorMessage}`);
  }

  return response;
}
```

---

## Example 1 — GET requests

```js
// Basic GET
async function getUser(userId) {
  const response = await fetch(`https://jsonplaceholder.typicode.com/users/${userId}`);

  if (!response.ok) {
    throw new Error(`Failed to fetch user: ${response.status}`);
  }

  return response.json();
}

// GET with query parameters (URLSearchParams is the clean way)
async function searchUsers(query, limit = 10, page = 1) {
  const params = new URLSearchParams({
    q:      query,
    limit:  limit.toString(),
    page:   page.toString(),
    sort:   "createdAt",
    order:  "desc",
  });

  const response = await fetch(`https://api.example.com/users?${params}`);

  if (!response.ok) throw new Error(`Search failed: ${response.status}`);
  return response.json();
}

// GET with authentication headers
async function getProtectedResource(token) {
  const response = await fetch("https://api.example.com/protected", {
    headers: {
      "Authorization": `Bearer ${token}`,
      "Accept":        "application/json",
    },
  });

  if (response.status === 401) throw new Error("Authentication required");
  if (response.status === 403) throw new Error("Insufficient permissions");
  if (!response.ok)            throw new Error(`Request failed: ${response.status}`);

  return response.json();
}
```

---

## Example 2 — POST, PUT, PATCH, DELETE

```js
// POST — create a resource
async function createUser(userData) {
  const response = await fetch("https://api.example.com/users", {
    method:  "POST",
    headers: { "Content-Type": "application/json" },
    body:    JSON.stringify(userData),
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({ message: response.statusText }));
    throw new Error(`Create user failed: ${err.message}`);
  }

  return response.json();   // returns the created resource (usually with generated ID)
}

// PUT — full replace
async function replaceUser(userId, userData) {
  const response = await fetch(`https://api.example.com/users/${userId}`, {
    method:  "PUT",
    headers: { "Content-Type": "application/json" },
    body:    JSON.stringify(userData),
  });

  if (!response.ok) throw new Error(`Replace failed: ${response.status}`);
  return response.json();
}

// PATCH — partial update
async function updateUserEmail(userId, newEmail) {
  const response = await fetch(`https://api.example.com/users/${userId}`, {
    method:  "PATCH",
    headers: { "Content-Type": "application/json" },
    body:    JSON.stringify({ email: newEmail }),
  });

  if (!response.ok) throw new Error(`Update failed: ${response.status}`);
  return response.json();
}

// DELETE
async function deleteUser(userId) {
  const response = await fetch(`https://api.example.com/users/${userId}`, {
    method: "DELETE",
    headers: { "Authorization": `Bearer ${getToken()}` },
  });

  if (response.status === 204) return null;  // No Content — success with no body
  if (!response.ok) throw new Error(`Delete failed: ${response.status}`);
  return response.json();
}
```

---

## Example 3 — real world: API client class

```js
class ApiClient {
  #baseUrl;
  #defaultHeaders;
  #timeout;

  constructor(baseUrl, options = {}) {
    this.#baseUrl        = baseUrl.replace(/\/$/, "");  // remove trailing slash
    this.#timeout        = options.timeout ?? 10_000;
    this.#defaultHeaders = {
      "Content-Type": "application/json",
      "Accept":       "application/json",
      ...options.headers,
    };
  }

  // Set auth token (called after login)
  setAuthToken(token) {
    if (token) {
      this.#defaultHeaders["Authorization"] = `Bearer ${token}`;
    } else {
      delete this.#defaultHeaders["Authorization"];
    }
  }

  async #request(method, endpoint, options = {}) {
    const url        = `${this.#baseUrl}${endpoint}`;
    const controller = new AbortController();
    const timeoutId  = setTimeout(() => controller.abort(), this.#timeout);

    try {
      const response = await fetch(url, {
        method,
        headers: { ...this.#defaultHeaders, ...options.headers },
        body:    options.body ? JSON.stringify(options.body) : undefined,
        signal:  controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        let errorDetail = response.statusText;
        try {
          const body = await response.json();
          errorDetail = body.message ?? body.error ?? errorDetail;
        } catch { /* body might not be JSON */ }

        const err = new Error(`${method} ${endpoint} → ${response.status}: ${errorDetail}`);
        err.statusCode = response.status;
        err.response   = response;
        throw err;
      }

      // Handle 204 No Content
      if (response.status === 204) return null;

      return response.json();

    } catch (err) {
      clearTimeout(timeoutId);
      if (err.name === "AbortError") {
        throw new Error(`Request timed out after ${this.#timeout}ms: ${method} ${endpoint}`);
      }
      throw err;
    }
  }

  get(endpoint, params)        { return this.#request("GET",    endpoint + (params ? `?${new URLSearchParams(params)}` : "")); }
  post(endpoint, body)         { return this.#request("POST",   endpoint, { body }); }
  put(endpoint, body)          { return this.#request("PUT",    endpoint, { body }); }
  patch(endpoint, body)        { return this.#request("PATCH",  endpoint, { body }); }
  delete(endpoint)             { return this.#request("DELETE", endpoint); }
}

// Usage:
const api = new ApiClient("https://api.example.com", { timeout: 5000 });
api.setAuthToken(localStorage.getItem("token"));

const user    = await api.get("/users/1");
const newUser = await api.post("/users", { name: "Alice", email: "alice@example.com" });
await api.delete(`/users/${user.id}`);
```

---

## AbortController — cancelling requests

```js
// AbortController gives you a signal to cancel an in-flight fetch
const controller = new AbortController();
const signal     = controller.signal;

// Start the request
const fetchPromise = fetch("https://api.example.com/slow-data", { signal });

// Cancel it after 3 seconds
const timeoutId = setTimeout(() => {
  controller.abort();   // sends abort signal to fetch
}, 3000);

try {
  const response = await fetchPromise;
  clearTimeout(timeoutId);
  return await response.json();
} catch (err) {
  if (err.name === "AbortError") {
    console.log("Request was cancelled");
  } else {
    throw err;
  }
}

// --- 

// Real use case: cancel request when component unmounts (React pattern)
function startSearch(query) {
  const controller = new AbortController();

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(r => r.json())
    .then(results => displayResults(results))
    .catch(err => {
      if (err.name !== "AbortError") throw err;
      // AbortError = user typed something new — safe to ignore
    });

  // Return cancel function
  return () => controller.abort();
}

// Usage:
let cancelSearch;

searchInput.addEventListener("input", (e) => {
  if (cancelSearch) cancelSearch();          // cancel previous search
  cancelSearch = startSearch(e.target.value); // start new search
});
```

---

## Sending different body types

```js
// 1. JSON (most common)
fetch("/api/data", {
  method:  "POST",
  headers: { "Content-Type": "application/json" },
  body:    JSON.stringify({ key: "value" }),
});

// 2. URL-encoded form data (HTML form default)
const formData = new URLSearchParams({ username: "alice", password: "secret" });
fetch("/login", {
  method:  "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body:    formData,
});

// 3. Multipart form data (file upload)
const form = new FormData();
form.append("username", "alice");
form.append("avatar",   fileInput.files[0]);  // File object
// DO NOT set Content-Type manually — browser sets it with the correct boundary
fetch("/api/profile", {
  method: "POST",
  body:   form,   // no Content-Type header — fetch sets it automatically
});

// 4. Plain text
fetch("/api/log", {
  method:  "POST",
  headers: { "Content-Type": "text/plain" },
  body:    "This is a log message",
});

// 5. Binary data (ArrayBuffer or Blob)
fetch("/api/upload", {
  method:  "POST",
  headers: { "Content-Type": "application/octet-stream" },
  body:    someArrayBuffer,
});
```

---

## Reading different response types

```js
async function downloadFile(url) {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`Download failed: ${response.status}`);

  // Check content type to decide how to read the body
  const contentType = response.headers.get("Content-Type") ?? "";

  if (contentType.includes("application/json")) {
    return { type: "json", data: await response.json() };
  }

  if (contentType.includes("text/")) {
    return { type: "text", data: await response.text() };
  }

  if (contentType.includes("image/")) {
    const blob = await response.blob();
    return { type: "image", url: URL.createObjectURL(blob) };
  }

  // Binary
  return { type: "binary", buffer: await response.arrayBuffer() };
}
```

---

## Fetch in Node.js

Since Node.js 18, `fetch` is globally available (no import needed). For older versions use `node-fetch`:

```js
// Node.js 18+ — global fetch
const response = await fetch("https://api.github.com/users/octocat");
const user     = await response.json();
console.log(user.name);

// Node.js < 18 — install node-fetch
// npm install node-fetch
import fetch from "node-fetch";

// Or use the https module (built-in, callback-based):
const https = require("https");

function httpsGet(url) {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      let data = "";
      res.on("data",  chunk => { data += chunk; });
      res.on("end",   ()    => resolve(JSON.parse(data)));
      res.on("error", err   => reject(err));
    }).on("error", reject);
  });
}
```

---

## Retry logic with exponential backoff

A production necessity for unreliable external APIs:

```js
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  const retryableStatuses = new Set([429, 500, 502, 503, 504]);
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);

      // Don't retry client errors (4xx except 429 rate limit)
      if (!response.ok && !retryableStatuses.has(response.status)) {
        const body = await response.json().catch(() => ({}));
        throw new Error(`HTTP ${response.status}: ${body.message ?? response.statusText}`);
      }

      if (!response.ok) {
        // Rate-limited or server error — prepare to retry
        const retryAfter = response.headers.get("Retry-After");
        if (retryAfter) {
          await new Promise(r => setTimeout(r, parseInt(retryAfter) * 1000));
        }
        throw new Error(`HTTP ${response.status} — will retry`);
      }

      return response;   // success

    } catch (err) {
      lastError = err;

      if (attempt < maxRetries) {
        // Exponential backoff: 500ms, 1000ms, 2000ms...
        const backoff = 500 * Math.pow(2, attempt - 1);
        const jitter  = Math.random() * 100;   // prevent thundering herd
        console.warn(`Attempt ${attempt} failed — retrying in ${backoff}ms:`, err.message);
        await new Promise(r => setTimeout(r, backoff + jitter));
      }
    }
  }

  throw new Error(`All ${maxRetries} attempts failed. Last error: ${lastError.message}`);
}

// Usage:
const response = await fetchWithRetry("https://api.example.com/data", {
  headers: { Authorization: `Bearer ${token}` },
}, 3);
const data = await response.json();
```

---

## Interceptors pattern — wrapping fetch globally

In frontend apps or Node.js services, you often want to intercept all requests/responses:

```js
// Wrap the global fetch with interceptors
function createFetchWithInterceptors(requestInterceptors = [], responseInterceptors = []) {
  return async function interceptedFetch(url, options = {}) {
    // Run request interceptors
    let [finalUrl, finalOptions] = [url, options];
    for (const interceptor of requestInterceptors) {
      [finalUrl, finalOptions] = await interceptor(finalUrl, finalOptions);
    }

    // Make the actual request
    let response = await fetch(finalUrl, finalOptions);

    // Run response interceptors
    for (const interceptor of responseInterceptors) {
      response = await interceptor(response);
    }

    return response;
  };
}

// Example interceptors:
const authInterceptor = (url, options) => {
  const token = getAuthToken();
  if (token) {
    options.headers = { ...options.headers, Authorization: `Bearer ${token}` };
  }
  return [url, options];
};

const logInterceptor = (url, options) => {
  console.log(`→ ${options.method ?? "GET"} ${url}`);
  return [url, options];
};

const errorInterceptor = async (response) => {
  if (response.status === 401) {
    await refreshToken();
    // Could retry original request here
  }
  return response;
};

const apiFetch = createFetchWithInterceptors(
  [logInterceptor, authInterceptor],
  [errorInterceptor]
);
```

---

## CORS — what it is and how fetch relates

```
CORS (Cross-Origin Resource Sharing) is a browser security mechanism.
A "cross-origin" request = different domain, protocol, or port.

Browser fetch to api.example.com from app.example.com:
  1. Browser sends ACTUAL request + "Origin: https://app.example.com" header
  2. Server must respond with "Access-Control-Allow-Origin: https://app.example.com"
  3. If missing → browser BLOCKS the response (network error in JS)

For "preflighted" requests (POST with JSON body, custom headers):
  1. Browser sends OPTIONS request first
  2. Server must respond with CORS headers
  3. THEN the actual request is made

CORS is enforced by the BROWSER — NOT by Node.js fetch:
  - fetch() in Node.js (backend) has NO CORS restrictions
  - fetch() in a browser respects CORS
  - fetch() errors due to CORS = response.ok is false, OR a network error
```

```js
// fetch with CORS mode options:
fetch("https://other-domain.com/api", {
  mode: "cors",          // default — browser enforces CORS
  mode: "no-cors",       // request sent, but response is OPAQUE (unreadable) — use carefully
  mode: "same-origin",   // only allow same-origin requests — throw if cross-origin
});

// Credentials (cookies, auth headers) with CORS:
fetch("https://api.example.com/profile", {
  credentials: "include",    // send cookies and auth headers cross-origin
  credentials: "same-origin", // default — only send credentials to same origin
  credentials: "omit",        // never send credentials
});
```

---

## Tricky things

### 1. Body can only be read ONCE

```js
const response = await fetch("/api/data");
const text = await response.text();
const json = await response.json();  // TypeError: body already consumed

// Fix — clone the response if you need to read it multiple times:
const response = await fetch("/api/data");
const clone    = response.clone();

const json = await response.json();
const text = await clone.text();   // reading the clone — both work
```

### 2. `fetch` doesn't send cookies by default in cross-origin requests

```js
// By default — no cookies sent cross-origin
fetch("https://api.example.com/me");

// To include cookies:
fetch("https://api.example.com/me", { credentials: "include" });
// Server must also respond with: Access-Control-Allow-Credentials: true
```

### 3. Network error vs HTTP error — different catch scenarios

```js
try {
  const response = await fetch("https://api.example.com/data");

  // Network error (no internet, CORS) → jump to catch
  // HTTP 404/500 → response.ok = false, BUT still resolves here

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);  // manually throw for HTTP errors
  }

  return await response.json();

} catch (err) {
  if (err.name === "TypeError" && err.message === "Failed to fetch") {
    console.error("Network error — check connection or CORS");
  } else if (err.name === "AbortError") {
    console.error("Request aborted");
  } else {
    console.error("Request error:", err.message);
  }
}
```

### 4. `response.json()` throws if body is empty or not valid JSON

```js
const response = await fetch("/api/delete/1");
// 204 No Content — body is empty

const data = await response.json();
// SyntaxError: Unexpected end of JSON input — body is empty!

// Fix — check status before reading body:
if (response.status === 204 || response.headers.get("Content-Length") === "0") {
  return null;
}
return response.json();
```

### 5. Timeouts are not built in — you must implement them

```js
// fetch has NO built-in timeout — requests can hang forever

// Fix — AbortController + setTimeout
async function fetchWithTimeout(url, options = {}, timeoutMs = 10_000) {
  const controller = new AbortController();
  const id         = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await fetch(url, { ...options, signal: controller.signal });
  } finally {
    clearTimeout(id);   // always clear the timeout
  }
}
```

### 6. Parallel fetch — all start simultaneously even with `await Promise.all`

```js
// Both fetches START at the same moment
const [users, products] = await Promise.all([
  fetch("/api/users").then(r => r.json()),
  fetch("/api/products").then(r => r.json()),
]);
// Total time = max(users time, products time)

// Contrast with sequential (WRONG when independent):
const users    = await fetch("/api/users").then(r => r.json());
const products = await fetch("/api/products").then(r => r.json());
// Total time = users time + products time
```

---

## Common mistakes

### Mistake 1 — Not checking `response.ok`

```js
// WRONG — silently processes error response as if it were success data
async function getUser(id) {
  const response = await fetch(`/api/users/${id}`);
  return response.json();   // if 404, this returns { error: "Not found" } as "user"
}

// RIGHT — always check ok
async function getUser(id) {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error(`User fetch failed: ${response.status}`);
  return response.json();
}
```

### Mistake 2 — Forgetting `Content-Type` header on POST

```js
// WRONG — body is sent but server receives it as plain text, not JSON
fetch("/api/users", {
  method: "POST",
  body:   JSON.stringify({ name: "Alice" }),
  // missing Content-Type: application/json
});

// RIGHT
fetch("/api/users", {
  method:  "POST",
  headers: { "Content-Type": "application/json" },
  body:    JSON.stringify({ name: "Alice" }),
});
```

### Mistake 3 — Setting `Content-Type` for FormData (breaks multipart boundary)

```js
// WRONG — manually setting Content-Type for FormData breaks it
const form = new FormData();
form.append("file", file);
fetch("/upload", {
  method:  "POST",
  headers: { "Content-Type": "multipart/form-data" },  // ← breaks it!
  body:    form,
});
// Server won't be able to parse the multipart boundary

// RIGHT — let browser set it automatically for FormData
fetch("/upload", {
  method: "POST",
  body:   form,  // no Content-Type header
});
```

### Mistake 4 — Awaiting response body twice

```js
// WRONG
const response = await fetch("/api/data");
const a = await response.json();
const b = await response.json();  // TypeError: body already consumed

// RIGHT — if you need it twice, clone first
const response = await fetch("/api/data");
const clone    = response.clone();
const a        = await response.json();
const b        = await clone.json();   // from the clone
```

---

## Practice exercises

### Exercise 1 — easy

Using the public JSONPlaceholder API (`https://jsonplaceholder.typicode.com`):
- Write `getPost(id)` — GET `/posts/{id}`, return the post object
- Write `getPostComments(postId)` — GET `/posts/{postId}/comments`, return comments array
- Write `createPost(title, body, userId)` — POST `/posts`, return the created post
- Write `getPostWithComments(id)` — use `Promise.all` to fetch both simultaneously

All functions must check `response.ok` and throw descriptive errors if not.

```js
// Write your code here
```

### Exercise 2 — medium

Build a `GitHubClient` class using the GitHub public API (`https://api.github.com`):
- Constructor takes optional `token` for authenticated requests (higher rate limits)
- `getUser(username)` — GET `/users/{username}`
- `getRepos(username, { sort = "updated", per_page = 10 })` — GET `/users/{username}/repos` with query params
- `getRepo(owner, repo)` — GET `/repos/{owner}/{repo}`
- `searchRepos(query, language, sort = "stars")` — GET `/search/repositories`

Include:
- Proper `Authorization` header when token is provided
- `Accept: application/vnd.github.v3+json` header on all requests
- Rate limit awareness: check `X-RateLimit-Remaining` response header and warn if below 10
- Request timeout of 8 seconds using AbortController

```js
// Write your code here
```

### Exercise 3 — hard

Build a production-grade `HttpClient` class:

Features:
- Base URL + default headers in constructor
- `get`, `post`, `put`, `patch`, `delete` methods
- Configurable timeout (default 10s) using AbortController
- Automatic retry with exponential backoff (configurable: `maxRetries`, `retryableStatuses`)
- Request/response interceptor arrays (add via `addRequestInterceptor(fn)`, `addResponseInterceptor(fn)`)
- Response body auto-detection: JSON, text, or raw based on Content-Type
- Error normalisation: all failures throw an `HttpError` with `statusCode`, `url`, `method`, `body` (parsed error response)
- Optional request logging (enabled via constructor `{ debug: true }`)
- `cancelAll()` method that aborts all in-flight requests

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Basic fetch | `const res = await fetch(url)` → `const data = await res.json()` |
| `response.ok` | `true` for 200-299; **fetch does NOT reject on 4xx/5xx** |
| `response.status` | HTTP status code (200, 404, 500...) |
| Read body | `.json()`, `.text()`, `.blob()`, `.arrayBuffer()`, `.formData()` |
| Body read once | Each body reader can only be called once — use `.clone()` if needed |
| POST with JSON | Set `Content-Type: application/json` + `body: JSON.stringify(data)` |
| POST with FormData | Do NOT set `Content-Type` — browser auto-sets it with boundary |
| Query params | `new URLSearchParams({ key: val })` → append to URL |
| Auth header | `"Authorization": "Bearer " + token` |
| Abort request | `const ctrl = new AbortController()` → `{ signal: ctrl.signal }` → `ctrl.abort()` |
| Timeout (manual) | `setTimeout(() => controller.abort(), ms)` |
| Network error | Promise rejects — `err.name === "TypeError"` |
| CORS error | Appears as network error in browser; `no-cors` gives opaque response |
| Credentials | `{ credentials: "include" }` to send cookies cross-origin |
| 204 No Content | Body is empty — don't call `.json()`; return `null` |
| Retry | Manual — check status code, `setTimeout` delay, loop or recursion |
| Node.js 18+ | Global `fetch` available natively |
| Node.js < 18 | `npm install node-fetch` or use `https` module |
| `response.headers.get` | `response.headers.get("Content-Type")` |

---

## Connected topics

- **52 — Sync vs async + event loop** — `fetch` is the most common Web API operation; understanding it puts fetch in the event loop model.
- **54 — Promises** — `fetch` returns a Promise; chaining `.then()` on it is covered there.
- **56 — async/await** — The standard way to use `fetch` — `await fetch(url)` then `await response.json()`.
- **57 — Error handling** — Handling `fetch` errors (network errors, HTTP errors, JSON parse errors) requires the patterns from that topic.
