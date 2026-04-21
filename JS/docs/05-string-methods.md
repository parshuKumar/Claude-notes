# 05 — String Methods

## What is this?
A string is a sequence of characters, and JavaScript gives every string a built-in toolkit of methods you can call on it. Think of a string like a physical ribbon of text — you can measure it, cut it, search it, transform it, and glue pieces together. String methods are the scissors, rulers, and highlighters for that ribbon. They don't modify the original string (strings are **immutable** in JS) — they always return a new one.

## Why does it matter?
Real applications are full of string work: formatting names, parsing user input, building URLs, validating emails, extracting data from text. Knowing string methods deeply means you can manipulate text cleanly without ugly loops or regex when a simple method will do.

---

## The golden rule: strings are immutable

Every string method returns a **new** string. The original is never changed.

```js
const name = "parsh";
name.toUpperCase();         // returns "PARSH" — but name is still "parsh"
console.log(name);          // "parsh" — unchanged

const upper = name.toUpperCase();
console.log(upper);         // "PARSH" — you must store the result
```

---

## Syntax overview

```js
const sentence = "  Hello, World!  ";

sentence.length                          // 18 — number of characters (includes spaces)
sentence.toUpperCase()                   // "  HELLO, WORLD!  "
sentence.toLowerCase()                   // "  hello, world!  "
sentence.trim()                          // "Hello, World!"  — removes leading/trailing whitespace
sentence.trimStart()                     // "Hello, World!  " — only leading
sentence.trimEnd()                       // "  Hello, World!" — only trailing
sentence.includes("World")               // true
sentence.startsWith("  Hello")           // true
sentence.endsWith("!  ")                 // true
sentence.indexOf("o")                    // 5  — first occurrence, -1 if not found
sentence.lastIndexOf("o")               // 9  — last occurrence
sentence.slice(2, 7)                     // "Hello" — from index 2 up to (not including) 7
sentence.replace("World", "Parsh")       // "  Hello, Parsh!  "
sentence.replaceAll("l", "L")            // "  HeLLo, WorLd!  "
sentence.split(", ")                     // ["  Hello", "World!  "]
sentence.repeat(2)                       // "  Hello, World!    Hello, World!  "
sentence.padStart(20, "*")               // "**  Hello, World!  "
sentence.padEnd(20, "-")                 // "  Hello, World!  ---"
sentence.charAt(2)                       // "H" — character at index 2
sentence[2]                              // "H" — bracket notation (same result)
sentence.charCodeAt(2)                   // 72  — Unicode code of "H"
sentence.at(-1)                          // " " — last character (negative indexing)
```

---

## How it works — deep dive per method

### `length`
Not a method — it's a **property**. No parentheses.
```js
"hello".length     // 5
"".length          // 0
" ".length         // 1 — space is a character
```

### `toUpperCase()` / `toLowerCase()`
Returns a new string with all characters shifted in case. Locale-insensitive. Used for case-insensitive comparisons.
```js
const userInput = "  PARSH  ";
const normalized = userInput.trim().toLowerCase();   // "parsh"
// Always normalize before comparing — "Admin" !== "admin"
```

### `trim()` / `trimStart()` / `trimEnd()`
Removes whitespace (spaces, tabs, newlines) from the edges. Critical for cleaning form input.
```js
const email = "  user@example.com   ";
email.trim()   // "user@example.com"
// Does NOT remove whitespace inside the string — only the edges
"  hello   world  ".trim()   // "hello   world" — inner spaces stay
```

### `includes(searchString, fromIndex?)`
Returns `true`/`false`. Case-sensitive. Optional second argument sets where to start searching.
```js
const plan = "premium-annual";
plan.includes("premium")     // true
plan.includes("Premium")     // false — case sensitive!
plan.includes("annual", 5)   // true — start searching from index 5
```

### `startsWith(str, position?)` / `endsWith(str, length?)`
```js
const filename = "invoice_2024.pdf";
filename.startsWith("invoice")   // true
filename.endsWith(".pdf")        // true
filename.endsWith(".pdf", 15)    // treats string as if it ends at position 15
```

### `indexOf(searchString, fromIndex?)` / `lastIndexOf()`
Returns the **index** of the first match, or `-1` if not found. Use this when you need the position, not just existence.
```js
const tag = "javascript-tutorial-javascript";
tag.indexOf("javascript")       // 0  — first occurrence starts at 0
tag.lastIndexOf("javascript")   // 20 — last occurrence starts at 20
tag.indexOf("python")           // -1 — not found

// Common pattern — check if found
if (tag.indexOf("tutorial") !== -1) {
  console.log("Found it");
}
// Modern alternative — use includes() if you don't need the position
```

### `slice(start, end?)`
Extracts a portion of a string. `start` is inclusive, `end` is exclusive. Negative indices count from the end.
```js
const email = "parsh@example.com";

email.slice(0, 5)     // "parsh"        — chars 0,1,2,3,4
email.slice(6)        // "example.com"  — from index 6 to the end
email.slice(-3)       // "com"          — last 3 characters
email.slice(-7, -4)   // "exa"          — 7 from end, up to 4 from end (exclusive)

// Get everything after "@"
const domain = email.slice(email.indexOf("@") + 1);   // "example.com"
```

**`slice` vs `substring` — what's the difference?**
- `slice` supports negative indices. `substring` treats negatives as 0.
- `slice(5, 2)` returns `""` (start > end → empty). `substring(5, 2)` swaps them and returns `substring(2, 5)`.
- **Use `slice` — it's more predictable.**

### `replace(search, replacement)` / `replaceAll(search, replacement)`
`replace` replaces only the **first** match. `replaceAll` replaces every match.
```js
const message = "Hello World, Hello JS";
message.replace("Hello", "Hi")      // "Hi World, Hello JS"  — only first
message.replaceAll("Hello", "Hi")   // "Hi World, Hi JS"     — all

// With a regex — replace can be powerful
"hello world".replace(/\b\w/g, c => c.toUpperCase())   // "Hello World"
// (Regex is a separate topic — just know replace() accepts one)
```

### `split(separator, limit?)`
Converts a string into an **array** by splitting on a separator.
```js
"apple,banana,cherry".split(",")     // ["apple", "banana", "cherry"]
"hello".split("")                    // ["h", "e", "l", "l", "o"] — split every char
"hello world".split(" ")             // ["hello", "world"]
"one two three".split(" ", 2)        // ["one", "two"] — limit to 2 results
"hello".split("x")                   // ["hello"] — separator not found, whole string in array
```

### `repeat(count)`
```js
"ha".repeat(3)          // "hahaha"
"-".repeat(20)          // "--------------------" — useful for dividers in logs
"".repeat(5)            // ""
```

### `padStart(targetLength, padString?)` / `padEnd()`
Pads the string to reach a target length.
```js
"5".padStart(3, "0")     // "005" — useful for formatting IDs, time values
"42".padStart(5, "0")    // "00042"
"9".padEnd(3, "0")       // "900"

const hour   = "9";
const minute = "5";
console.log(`${hour.padStart(2, "0")}:${minute.padStart(2, "0")}`);   // "09:05"
```

### `at(index)`
Like bracket notation but supports **negative indices**. Cleaner than `str[str.length - 1]`.
```js
const word = "keyboard";
word.at(0)    // "k"  — first character
word.at(-1)   // "d"  — last character
word.at(-2)   // "r"  — second to last
// word[word.length - 1] does the same as word.at(-1) — but .at() is much cleaner
```

---

## Template literals — the full picture

Template literals use backticks (`` ` ``) instead of quotes and let you embed any JS **expression** inside `${}`.

```js
const productName = "Keyboard";
const price       = 89.99;
const inStock     = true;
const quantity    = 3;

// Basic interpolation
console.log(`Product: ${productName}`);          // "Product: Keyboard"

// Expressions inside ${}
console.log(`Total: $${price * quantity}`);       // "Total: $269.97"
console.log(`Status: ${inStock ? "In stock" : "Out of stock"}`);

// Multi-line strings — backticks preserve newlines
const receipt = `
  Order Summary
  -------------
  Item:  ${productName}
  Price: $${price}
  Qty:   ${quantity}
  Total: $${(price * quantity).toFixed(2)}
`;

// Nested template literals
const items = ["apple", "banana"];
console.log(`You have ${items.length} item${items.length === 1 ? "" : "s"}.`);
// "You have 2 items."

// You can call any method or function inside ${}
const rawName = "  parsh shah  ";
console.log(`Hello, ${rawName.trim().split(" ").map(w => w[0].toUpperCase() + w.slice(1)).join(" ")}!`);
// "Hello, Parsh Shah!" — complex expression, perfectly valid
```

---

## Method chaining — combining methods

Because every method returns a new string, you can chain them:

```js
const userInput = "  HELLO WORLD  ";

const clean = userInput
  .trim()           // "HELLO WORLD"
  .toLowerCase()    // "hello world"
  .replace("world", "parsh");  // "hello parsh"

console.log(clean);   // "hello parsh"

// Real world: normalise an email from a form
const rawEmail = "  USER@EXAMPLE.COM  ";
const normalizedEmail = rawEmail.trim().toLowerCase();   // "user@example.com"
```

---

## Example 1 — basic

```js
// Working with a product title from an API (might have messy casing/whitespace)
const rawTitle = "  wireless mechanical KEYBOARD  ";

const cleanTitle   = rawTitle.trim();                    // "wireless mechanical KEYBOARD"
const titleCased   = cleanTitle.toLowerCase();           // "wireless mechanical keyboard"
const displayTitle = titleCased
  .split(" ")
  .map(word => word.charAt(0).toUpperCase() + word.slice(1))
  .join(" ");                                            // "Wireless Mechanical Keyboard"

console.log(displayTitle);                               // "Wireless Mechanical Keyboard"
console.log(`Length: ${displayTitle.length}`);           // "Length: 28"
console.log(`Has "Mechanical": ${displayTitle.includes("Mechanical")}`);  // true
```

---

## Example 2 — real world

```js
// Parsing a CSV-style config string and generating user cards
const csvData = "parsh_dev,Parsh Shah,premium,Mumbai";

const [username, fullName, plan, city] = csvData.split(",");

const initials = fullName
  .split(" ")
  .map(word => word[0])
  .join("")
  .toUpperCase();   // "PS"

const planLabel = plan.charAt(0).toUpperCase() + plan.slice(1);   // "Premium"

const cardId = username.padEnd(15, ".") + initials;   // "parsh_dev......PS"

console.log(`
  === User Card ===
  Name:    ${fullName}
  Handle:  @${username}
  Plan:    ${planLabel}
  City:    ${city}
  Card ID: ${cardId}
`);
```

---

## Common mistakes

### 1. Forgetting to store the result of a method
```js
// WRONG — method result is discarded
let username = "  PARSH  ";
username.trim();
username.toLowerCase();
console.log(username);   // "  PARSH  " — unchanged!

// RIGHT — store or chain
let username = "  PARSH  ";
username = username.trim().toLowerCase();
console.log(username);   // "parsh"
```

### 2. Using `replace` when you need `replaceAll`
```js
// WRONG — only replaces the first match
const buggySlug = "hello world hello";
const slug1 = buggySlug.replace(" ", "-");   // "hello-world hello" — only first space

// RIGHT
const slug2 = buggySlug.replaceAll(" ", "-");   // "hello-world-hello"
// Or use regex with /g flag
const slug3 = buggySlug.replace(/ /g, "-");     // "hello-world-hello"
```

### 3. Treating `indexOf` result as boolean
```js
// WRONG — indexOf returns 0 when found at position 0, and 0 is falsy!
const tag = "javascript basics";
if (tag.indexOf("javascript")) {
  console.log("Found");   // DOESN'T RUN — indexOf returned 0, which is falsy
}

// RIGHT — always compare to -1
if (tag.indexOf("javascript") !== -1) {
  console.log("Found");   // works correctly
}

// BEST — use includes() if you just want true/false
if (tag.includes("javascript")) {
  console.log("Found");
}
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — `slice` with `indexOf` for real parsing
```js
const url = "https://example.com/products/keyboard";
const path = url.slice(url.indexOf("/", 8));   // "/products/keyboard"
// The second argument to indexOf says "start searching from index 8" — skips "https://"
```

### Trick 2 — `split` + `join` is the standard way to replace all (before `replaceAll` existed)
```js
const str = "one-two-three";
str.split("-").join(" ");   // "one two three"
// You'll see this pattern in older codebases
```

### Trick 3 — Accessing characters with bracket notation on strings
```js
const word = "hello";
console.log(word[0]);    // "h"
console.log(word[100]);  // undefined — no error, just undefined
// word[0] = "H"  → silently fails in non-strict mode — strings are immutable
```

### Trick 4 — Empty `split` vs split by space
```js
"hello world".split()      // ["hello world"] — no separator → whole string in array
"hello world".split(" ")   // ["hello", "world"] — split on space
"hello world".split("")    // ["h","e","l","l","o"," ","w","o","r","l","d"] — every char
```

### Trick 5 — Template literals evaluate expressions, not just variables
```js
const a = 5, b = 3;
console.log(`${a} + ${b} = ${a + b}`);    // "5 + 3 = 8"
console.log(`${a > b ? "a wins" : "b wins"}`);   // "a wins"

// You can even call functions:
function greet(name) { return `Hi, ${name}`; }
console.log(`Message: ${greet("Parsh")}`);   // "Message: Hi, Parsh"
```

### Trick 6 — `String.raw` for escaping backslashes
```js
// Normal template literal — \n is a newline
console.log(`C:\new\folder`);       // C:
                                    //   ew
                                    //      older   (weird!)

// String.raw — backslashes are treated literally
console.log(String.raw`C:\new\folder`);   // C:\new\folder
// Useful for Windows paths and regex strings
```

---

## Practice exercises

### Exercise 1 — easy
Given this string:
```js
const movieTitle = "  the dark KNIGHT rises  ";
```

Write code that produces and logs:
1. The trimmed string
2. The title in proper Title Case (first letter of every word capitalised, rest lowercase) — e.g. `"The Dark Knight Rises"`
3. The total character count of the trimmed title (no leading/trailing spaces)
4. Whether it includes the word "knight" (case-insensitive check)

```js
// Write your code here
```

---

### Exercise 2 — medium
You receive a URL string from an API:
```js
const apiUrl = "https://api.example.com/users/parsh_dev/orders?limit=10&page=2";
```

Write code that extracts and logs:
1. The **protocol** — `"https"`
2. The **domain** — `"api.example.com"`
3. The **username** — `"parsh_dev"`
4. The **query string** — `"limit=10&page=2"` (everything after `?`)
5. Whether the URL is paginated (contains `"page="`)

Use only string methods — no regex, no `URL` constructor.
```js
// Write your code here
```

---

### Exercise 3 — hard
You are building a username validator and formatter. Given raw input from a sign-up form:

```js
const rawInput = "  Parsh_Dev 99  ";
```

Apply the following rules and log the result at each step:
1. Trim whitespace from both ends
2. Replace any spaces inside the string with underscores
3. Convert to lowercase
4. The username must be between 4 and 20 characters — log whether it's valid
5. It must not start with a number — log whether this rule passes
6. Generate a display name from the username: replace all underscores with spaces, then Title Case each word
7. Create a user ID in the format: `USR-` + first 4 chars of username (uppercase) + `-` + the username's length padded to 3 digits with zeros

Log every step with a label. Final output should look like:
```
Raw input:     "  Parsh_Dev 99  "
After trim:    "Parsh_Dev 99"
After spaces:  "Parsh_Dev_99"
Lowercase:     "parsh_dev_99"
Length valid:  true
Starts valid:  true
Display name:  "Parsh Dev 99"
User ID:       "USR-PARS-012"
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Properties (no parentheses):**
```
str.length           → number of characters
```

**Finding / searching:**
```
str.includes(s)      → true/false
str.startsWith(s)    → true/false
str.endsWith(s)      → true/false
str.indexOf(s)       → index or -1
str.lastIndexOf(s)   → index or -1 (from the right)
```

**Extracting:**
```
str.slice(start, end)       → new string (supports negatives)
str.at(index)               → character (supports negatives)
str.charAt(index)           → character (no negatives)
str.split(separator)        → array of strings
```

**Transforming:**
```
str.toUpperCase()           → all caps
str.toLowerCase()           → all lowercase
str.trim()                  → remove edge whitespace
str.trimStart()             → remove leading whitespace
str.trimEnd()               → remove trailing whitespace
str.replace(a, b)           → replace first match
str.replaceAll(a, b)        → replace all matches
str.repeat(n)               → repeat n times
str.padStart(len, char)     → pad left to length
str.padEnd(len, char)       → pad right to length
```

**Template literals:**
```
`Hello, ${name}`            → interpolation
`${a + b}`                  → any expression
`Line 1\nLine 2`            → multi-line
String.raw`C:\path`         → no escape processing
```

**Key rules:**
- Strings are **immutable** — always store the result of a method
- `indexOf` returns `0` when found at position 0 — `0` is falsy — always compare to `-1`
- `replace` only replaces the **first** match — use `replaceAll` or regex `/g` for all
- `slice` supports negatives, `substring` does not — prefer `slice`
- `split("")` splits every character — `split()` with no arg gives the whole string in an array

---

## Connected topics
- **06 — Number Methods** — the numeric counterpart: `toFixed`, `parseInt`, `parseFloat`, `Math`
- **33 — Array Destructuring** — `split()` returns an array; destructuring is how you cleanly unpack it
- **04 — Operators** — template literals use `${}` which evaluates any JS expression, including operators
