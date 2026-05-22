# 78 — Clean Code Principles

---

## 1. What is this?

**Clean code** is code that is easy to read, easy to understand, easy to change, and easy to delete. It communicates its intent clearly — another developer (or you, six months from now) can read it and immediately understand what it does and why.

Clean code is not:
- The cleverest code
- The shortest code
- The most performant code (though clean code is often well-optimised too)

**Analogy — a well-written recipe vs a chef's personal shorthand:**
A chef's personal notebook might use abbreviations only they understand. A published recipe book must be readable by anyone. Code that runs in production is a published recipe — you are writing it for the next person who reads it, not just for the machine that executes it.

**The guiding rule:** Code is read far more often than it is written. Optimise for reading.

---

## 2. Why does it matter?

- **Maintenance cost** — 80% of software cost is maintenance. Clean code is cheaper to maintain.
- **Bug rate** — unclear code hides bugs. Clean code makes assumptions explicit and errors obvious.
- **Onboarding** — a new developer can contribute faster when code reads like prose.
- **Refactoring safety** — clean code has small, well-named functions that can be safely changed without side effects.
- **Your future self** — you will read your own code again. Make it a gift, not a burden.

---

## 3. Naming

Naming is the most impactful clean code practice. A well-named variable, function, or class is worth more than any comment.

### Variables — describe what the data is

```js
// ❌ Cryptic abbreviations
const d   = new Date();
const u   = getUser();
const arr = [1, 2, 3];
const fn  = (x) => x * 2;

// ✅ Names that tell you what the value IS
const today          = new Date();
const currentUser    = getUser();
const itemPrices     = [1, 2, 3];
const double         = (n) => n * 2;
```

### Booleans — use is/has/can/should prefix

```js
// ❌ Ambiguous — is this a number? a state? a count?
const active  = true;
const visible = false;
const loaded  = true;
const admin   = user.role === "admin";

// ✅ Reads like a sentence
const isActive   = true;
const isVisible  = false;
const hasLoaded  = true;
const isAdmin    = user.role === "admin";
const canEdit    = isAdmin || isOwner;
const shouldRetry = attempts < maxRetries;
```

### Functions — describe what they DO (verb phrase)

```js
// ❌ Noun or vague
const userData = () => fetchUser();
const order    = () => processPayment();
const check    = () => validateEmail(email);

// ✅ Verb phrase — reads like an instruction
const fetchUserById    = (id) => getUser(id);
const processPayment   = (amount, card) => chargeCard(amount, card);
const validateEmail    = (email) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
const isEmailValid     = (email) => validateEmail(email); // predicate form
```

### Don't add noise words

```js
// ❌ Noise — "data", "info", "manager", "handler", "object", "util"
const userData     = { name: "Alice" };      // just: user
const accountInfo  = { balance: 100 };       // just: account
const UserManager  = class { ... };          // just: UserService or UserRepository
const handleClick  = () => { ... };          // just: onClick or submitForm
const userUtils    = { formatName: () => {} }; // just: formatName directly

// ✅ Name the thing, not the category of thing
const user    = { name: "Alice" };
const account = { balance: 100 };
class UserService { ... }
const onClick = () => { ... };
```

### Avoid encodings and type prefixes

```js
// ❌ Hungarian notation — type in the name
const strName    = "Alice";
const arrItems   = [];
const objConfig  = {};
const bIsLoading = false;

// ✅ Type is evident from context (and TypeScript/JSDoc if needed)
const name      = "Alice";
const items     = [];
const config    = {};
const isLoading = false;
```

### Use consistent vocabulary

```js
// ❌ Three different words for the same concept
getUserById()
fetchAccount()
retrieveOrder()

// ✅ Pick one verb per concept and use it everywhere
getUser()
getAccount()
getOrder()
```

---

## 4. Functions

### Do one thing

A function should do exactly one thing, do it well, and do it only.

```js
// ❌ Three responsibilities in one function
async function registerUser(data) {
  // 1. Validate
  if (!data.email) throw new Error("Email required");
  if (!data.password || data.password.length < 8) throw new Error("Weak password");

  // 2. Hash password
  const hash = await bcrypt.hash(data.password, 12);

  // 3. Save to DB
  const user = await db.users.create({ ...data, password: hash });

  // 4. Send email
  await sendWelcomeEmail(user.email);

  // 5. Return
  return user;
}

// ✅ Each function has one job
async function registerUser(data) {
  validateRegistrationData(data);
  const user = await createUser(data);
  await sendWelcomeEmail(user.email);
  return user;
}

function validateRegistrationData(data) {
  if (!data.email)    throw new ValidationError({ email: ["Required"] });
  if (!data.password) throw new ValidationError({ password: ["Required"] });
  if (data.password.length < 8) throw new ValidationError({ password: ["Min 8 chars"] });
}

async function createUser(data) {
  const hash = await bcrypt.hash(data.password, 12);
  return db.users.create({ ...data, password: hash });
}
```

### Small functions — aim for 5–15 lines

If a function needs to be scrolled to read, it probably does too much. Extract sub-tasks into well-named helper functions. The extracted name documents what the code does — removing the need for comments.

### Function arguments — fewer is better

```js
// ❌ Positional arguments — easy to mix up, hard to read at call site
createUser("Alice", "alice@example.com", "admin", true, false);
// What does `true, false` mean here?

// ✅ Named options object for 3+ arguments
createUser({
  name:      "Alice",
  email:     "alice@example.com",
  role:      "admin",
  verified:  true,
  suspended: false
});

// ✅ Destructure in the parameter
function createUser({ name, email, role = "user", verified = false, suspended = false }) {
  // ...
}
```

### No side effects in predicates/getters

```js
// ❌ A function named isValid should only CHECK — not modify
function isEmailValid(email) {
  this.lastChecked = email; // ❌ side effect!
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// ✅ Pure predicate — returns boolean, changes nothing
function isEmailValid(email) {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

### Return early — avoid deep nesting

```js
// ❌ Arrow-shaped code — logic buried at the deepest indent level
function processOrder(order) {
  if (order) {
    if (order.items.length > 0) {
      if (order.user.isVerified) {
        if (!order.isPaid) {
          return chargeCard(order);
        } else {
          return "already paid";
        }
      } else {
        return "unverified user";
      }
    } else {
      return "empty order";
    }
  } else {
    return "no order";
  }
}

// ✅ Guard clauses — happy path is at the lowest indent level
function processOrder(order) {
  if (!order)                  return "no order";
  if (order.items.length === 0) return "empty order";
  if (!order.user.isVerified)   return "unverified user";
  if (order.isPaid)             return "already paid";

  return chargeCard(order);
}
```

---

## 5. Comments

### The best comment is a well-named function

```js
// ❌ Comment explains what cryptic code does
// Check if user has active subscription and enough credits
if (user.sub && user.sub.active && user.credits > 0 && !user.sub.paused) {
  // ...
}

// ✅ Name extracts the intent — no comment needed
if (canUserAccess(user)) {
  // ...
}

function canUserAccess(user) {
  return user.sub?.active && user.credits > 0 && !user.sub?.paused;
}
```

### Comments that DO add value

```js
// ✅ Why — explains business rule not obvious from code
// RFC 5322 allows + in local part — this is intentional
const emailRegex = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@/;

// ✅ Warning of known issue or workaround
// TODO: remove when Safari 16 is no longer supported (adds native :has() support)
const hasParent = el.closest("[data-parent]") !== null;

// ✅ Explaining non-obvious algorithm
// Fisher-Yates shuffle — O(n), unbiased
function shuffle(arr) { ... }

// ✅ API contract for public functions
/**
 * Retry an async operation with exponential backoff.
 * @param {() => Promise<T>} fn - The operation to retry
 * @param {number} [retries=3]  - Max attempts
 * @returns {Promise<T>}
 */
async function withRetry(fn, retries = 3) { ... }
```

### Comments that should be deleted

```js
// ❌ Restating what the code obviously does
i++; // increment i
const user = getUser(id); // get the user

// ❌ Commented-out code
// const oldResult = processData(data);
// console.log(oldResult);

// ❌ Journal / history (use git for this)
// 2024-01-15 Alice: fixed the bug
// 2024-02-01 Bob: added new feature

// ❌ Misleading comments (worse than no comment)
// Returns the user's name
function getFullName(user) {
  return `${user.firstName} ${user.lastName}`; // returns full name, not just name
}
```

---

## 6. Code Structure and Organisation

### Keep related things together

```js
// ❌ Feature logic scattered across many files with no clear grouping
/src
  /helpers/userHelper.js
  /utils/userUtils.js
  /services/userService.js
  /controllers/userController.js
  /models/userModel.js
  /validators/userValidator.js

// ✅ Feature-first organisation — everything for "user" lives together
/src
  /users
    user.model.js      // schema / DB layer
    user.service.js    // business logic
    user.controller.js // HTTP layer
    user.validator.js  // validation rules
    user.test.js       // tests
  /orders
    order.model.js
    ...
```

### The newspaper metaphor

Organise a file like a newspaper article — high-level summary at the top, details at the bottom:

```js
// ✅ Top: the main exported function (what callers care about)
export async function registerUser(data) {
  validateRegistrationData(data);
  const user = await createUser(data);
  await sendWelcomeEmail(user.email);
  return user;
}

// ─── Details below ───────────────────────────────────────────────────────────

function validateRegistrationData(data) {
  // ...
}

async function createUser(data) {
  // ...
}

async function sendWelcomeEmail(email) {
  // ...
}
```

### Consistent abstraction levels

A function should work at one level of abstraction. Don't mix high-level business logic with low-level implementation details:

```js
// ❌ Mixes high-level (process order) with low-level (regex, SQL)
async function processOrder(orderId) {
  const order = await db.query(`SELECT * FROM orders WHERE id = ${orderId}`); // SQL!
  if (!/^\d+$/.test(String(orderId))) throw new Error("bad id"); // regex!
  order.status = "processing";
  await db.query(`UPDATE orders SET status='processing' WHERE id=${orderId}`); // SQL!
  await sendEmail(order.user_email, "Your order is processing");
}

// ✅ All at the same level — each line reads as a step in the process
async function processOrder(orderId) {
  const order = await orderRepository.findById(orderId);
  await orderRepository.updateStatus(order.id, "processing");
  await notifyUser(order.userId, "order_processing");
}
```

---

## 7. The DRY Principle

**Don't Repeat Yourself** — every piece of knowledge should have a single, authoritative representation in the system.

```js
// ❌ Same validation logic in three places
// In userController.js:
if (!req.body.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(req.body.email)) {
  return res.status(400).json({ error: "Invalid email" });
}

// In adminController.js — copy-pasted, slightly different
if (!data.email || !data.email.includes("@")) {
  throw new Error("Bad email");
}

// In importService.js — another variation
if (typeof row.email !== "string" || row.email.trim() === "") {
  errors.push("Invalid email");
}

// ✅ One canonical implementation, called everywhere
// validators.js
export function isValidEmail(email) {
  return typeof email === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// Used in all three places
if (!isValidEmail(req.body.email)) { ... }
```

**DRY applies to knowledge, not just syntax.** Two pieces of code that look similar but represent different business rules should NOT be merged — that would be wrong abstraction.

---

## 8. The SOLID Principles (simplified for JS)

### S — Single Responsibility Principle

A module/class/function should have one reason to change.

```js
// ❌ One class does too much — multiple reasons to change
class UserService {
  async getUser(id) { return db.query(/* ... */); }     // DB
  validateEmail(email) { return /@/.test(email); }       // Validation
  formatForApi(user) { return { ...user }; }             // Serialisation
  sendWelcomeEmail(email) { return mailer.send(/* */); } // Email
  logAccess(userId) { logger.info(/* ... */); }          // Logging
}

// ✅ Each class has one job
class UserRepository  { async findById(id)  { ... } } // DB only
class UserValidator   { isEmailValid(email) { ... } } // Validation only
class UserSerializer  { toApiResponse(user) { ... } } // Serialisation only
class UserMailer      { sendWelcome(email)  { ... } } // Email only
```

### O — Open/Closed Principle

Open for extension, closed for modification. Add new behaviour without changing existing code.

```js
// ❌ Adding a new payment method requires editing this function
function processPayment(type, amount) {
  if (type === "card")   return chargeCard(amount);
  if (type === "paypal") return chargePaypal(amount);
  // Every new method = edit this function
}

// ✅ Register new providers without touching existing code
const paymentProviders = new Map();
paymentProviders.set("card",   chargeCard);
paymentProviders.set("paypal", chargePaypal);

function processPayment(type, amount) {
  const charge = paymentProviders.get(type);
  if (!charge) throw new Error(`Unknown payment type: ${type}`);
  return charge(amount);
}

// Adding Stripe: paymentProviders.set("stripe", chargeStripe) — zero existing code changed
```

### L — Liskov Substitution Principle

Subclasses should be substitutable for their base class without breaking the program.

```js
// ❌ Breaks Liskov — subclass changes the contract (throws instead of returning null)
class UserRepository {
  findById(id) { return null; } // returns null if not found
}
class CachedUserRepository extends UserRepository {
  findById(id) { throw new Error("Use findByIdAsync"); } // breaks contract
}

// ✅ Subclass honours the contract
class CachedUserRepository extends UserRepository {
  findById(id) {
    return this.cache.get(id) ?? super.findById(id); // same contract
  }
}
```

### I — Interface Segregation Principle

Don't force modules to depend on methods they don't use. Keep interfaces small and focused.

```js
// ❌ One large "service" object passed everywhere — most consumers use 2 of 10 methods
function renderUserCard(userService) {
  const user = userService.findById(id);     // uses this
  return formatCard(user);                   // uses this
  // never uses: userService.delete, userService.bulkImport, userService.exportCsv...
}

// ✅ Pass only what's needed
function renderUserCard({ findById }) {  // accepts a thin interface
  const user = findById(id);
  return formatCard(user);
}
```

### D — Dependency Inversion Principle

High-level modules should not depend on low-level modules. Both should depend on abstractions.

```js
// ❌ High-level service directly depends on low-level DB implementation
class OrderService {
  constructor() {
    this.db = new PostgresDatabase(); // ← hard-coded dependency
  }
}

// ✅ Inject the dependency — service works with anything that has findOrder/saveOrder
class OrderService {
  constructor(orderRepository) {      // ← accept any compatible object
    this.repo = orderRepository;
  }
  async getOrder(id) { return this.repo.findById(id); }
}

// In production:
new OrderService(new PostgresOrderRepository());
// In tests:
new OrderService(new InMemoryOrderRepository());
```

---

## 9. The YAGNI Principle

**You Aren't Gonna Need It** — don't build features before they're needed.

```js
// ❌ Over-engineered for a feature that doesn't exist yet
class DataProcessor {
  constructor(options = {}) {
    this.strategy     = options.strategy     ?? "default";
    this.pluginHooks  = options.pluginHooks  ?? [];
    this.cacheLayer   = options.cacheLayer   ?? null;
    this.retryPolicy  = options.retryPolicy  ?? { retries: 3 };
    this.transformer  = options.transformer  ?? (x => x);
    this.validator    = options.validator    ?? (() => true);
    // 100 lines of configuration for requirements that don't exist
  }
}

// ✅ Build for what you need today — refactor when requirements actually change
class DataProcessor {
  process(data) {
    return transform(validate(data));
  }
}
```

---

## 10. The Law of Demeter — "Don't talk to strangers"

A function should only call methods on:
- Itself
- Its direct parameters
- Objects it creates
- Its direct component parts

```js
// ❌ Reaching through object chains — tightly coupled to three levels of structure
function applyDiscount(order) {
  const tier = order.user.account.membership.tier; // fragile chain
  if (tier === "gold") return order.total * 0.9;
  return order.total;
}

// ✅ Ask for what you need, or add a method to the right object
function applyDiscount(order) {
  const tier = order.getUserMembershipTier(); // order knows its user
  if (tier === "gold") return order.total * 0.9;
  return order.total;
}

// Or: pass the tier directly — caller resolves the chain
function applyDiscount(total, membershipTier) {
  return membershipTier === "gold" ? total * 0.9 : total;
}
```

---

## 11. Magic Numbers and Magic Strings

```js
// ❌ What is 86400000? What is 3? What is "admin"?
if (Date.now() - user.createdAt > 86400000) expireAccount(user);
if (items.length > 3) showPagination();
if (user.role === "admin") showDashboard();

// ✅ Named constants tell you WHAT the value represents and WHY
const ONE_DAY_MS          = 24 * 60 * 60 * 1000; // 86400000
const PAGINATION_THRESHOLD = 3;
const ROLES               = Object.freeze({ ADMIN: "admin", USER: "user" });

if (Date.now() - user.createdAt > ONE_DAY_MS)   expireAccount(user);
if (items.length > PAGINATION_THRESHOLD)          showPagination();
if (user.role === ROLES.ADMIN)                    showDashboard();
```

---

## 12. Error Handling

```js
// ❌ Silent failures hide bugs
try {
  processData(input);
} catch { /* ignored */ }

// ❌ Generic errors lose context
throw new Error("failed");

// ✅ Handle specifically, log everything, preserve context
try {
  processData(input);
} catch (e) {
  if (e instanceof ValidationError) return res.status(422).json({ error: e.message });
  logger.error({ err: e, input }, "processData failed");
  throw e; // re-throw unknown errors
}

// ✅ Descriptive errors with context
throw new NotFoundError("User"); // "User not found" — from topic 73
throw new Error("Failed to process payment", { cause: originalError });
```

---

## 13. Clean Async Code

```js
// ❌ Callback pyramid of doom
getUser(id, function(err, user) {
  if (err) return cb(err);
  getOrders(user.id, function(err, orders) {
    if (err) return cb(err);
    processOrders(orders, function(err, result) {
      if (err) return cb(err);
      cb(null, result);
    });
  });
});

// ✅ async/await — reads like synchronous code
async function getUserOrders(id) {
  const user   = await getUser(id);
  const orders = await getOrders(user.id);
  return processOrders(orders);
}

// ✅ Parallel when independent
async function getDashboard(userId) {
  const [user, orders, notifications] = await Promise.all([
    getUser(userId),
    getOrders(userId),
    getNotifications(userId)
  ]);
  return { user, orders, notifications };
}
```

---

## 14. Real-World Refactoring Example

Before — a real-world-style messy function:

```js
// ❌ BEFORE
async function p(d) {
  let r = false;
  if (d && d.e && d.p) {
    if (d.p.length >= 8) {
      const u = await db.query("SELECT * FROM users WHERE email = '" + d.e + "'");
      if (!u) {
        const h = await bcrypt.hash(d.p, 10);
        await db.query("INSERT INTO users (email, password) VALUES ('" + d.e + "', '" + h + "')");
        await mailer.send(d.e, "Welcome!");
        r = true;
      }
    }
  }
  return r;
}
```

Problems:
- One-letter names mean nothing
- SQL injection vulnerability
- Three levels of nesting
- Multiple responsibilities
- Returns `true`/`false` instead of throwing on failure
- No specific error messages

After:

```js
// ✅ AFTER
async function registerUser({ email, password }) {
  validateRegistrationInput({ email, password });
  await assertEmailNotTaken(email);

  const user = await createUserRecord({ email, password });
  await sendWelcomeEmail(user.email);

  return user;
}

function validateRegistrationInput({ email, password }) {
  if (!email)    throw new ValidationError({ email: ["Required"] });
  if (!password) throw new ValidationError({ password: ["Required"] });
  if (password.length < 8) throw new ValidationError({ password: ["Minimum 8 characters"] });
}

async function assertEmailNotTaken(email) {
  const existing = await userRepository.findByEmail(email);
  if (existing) throw new ConflictError("Email already registered");
}

async function createUserRecord({ email, password }) {
  const passwordHash = await bcrypt.hash(password, 12);
  return userRepository.create({ email, passwordHash });
}

async function sendWelcomeEmail(email) {
  await mailer.send(email, "welcome");
}
```

Each function: one job, named after what it does, no nesting, SQL injection gone (repository pattern), throws specific errors.

---

## 15. Clean Code Checklist

Use this before every code review or PR:

```
NAMING
  □ Variables named for what they ARE, not type or category
  □ Booleans start with is/has/can/should
  □ Functions start with a verb — say what they DO
  □ No noise words (data, info, manager, util)
  □ Consistent vocabulary across similar operations

FUNCTIONS
  □ Does one thing
  □ Fits on one screen (~15 lines)
  □ ≤ 3 parameters (or options object)
  □ No side effects in predicates
  □ Guard clauses at top, happy path last

COMMENTS
  □ No comments that restate what code does
  □ No commented-out code
  □ No changelog comments (use git)
  □ WHY comments present where needed
  □ JSDoc on public API functions

STRUCTURE
  □ Related code is in the same file/folder
  □ Abstraction is consistent within a function
  □ Magic numbers/strings extracted to named constants
  □ No deep nesting (>3 levels) — extract or guard clause

ERRORS
  □ No silent catches
  □ Errors are specific types with context
  □ Unknown errors are re-thrown

ASYNC
  □ async/await instead of callback nesting
  □ Independent operations use Promise.all
  □ Errors handled or explicitly propagated

PRINCIPLES
  □ DRY — no copy-pasted logic
  □ YAGNI — no speculative features
  □ Single Responsibility — each unit has one job
  □ Dependency injection for testability
```

---

## 16. Practice Exercises

### Easy — rename and clarify

Take this function and rewrite it with clean naming and guard clauses only — no logic changes:

```js
async function proc(x, y, z) {
  let res = null;
  if (x) {
    if (y && y.length > 0) {
      if (z === true) {
        res = await db.q("INSERT INTO t VALUES (?, ?)", [x, y]);
      }
    }
  }
  return res;
}
```

---

### Medium — extract and refactor

Refactor this into small, single-responsibility functions with clean names. Handle errors properly. Do not change the observable behaviour:

```js
app.post("/checkout", async (req, res) => {
  try {
    const { userId, items, cardToken } = req.body;
    if (!userId || !items || !items.length || !cardToken) {
      return res.status(400).json({ error: "Missing fields" });
    }
    const user = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
    if (!user) return res.status(404).json({ error: "User not found" });
    let total = 0;
    for (const item of items) {
      const product = await db.query(`SELECT * FROM products WHERE id = ${item.productId}`);
      total += product.price * item.qty;
    }
    const charge = await stripe.charges.create({ amount: total * 100, currency: "usd", source: cardToken });
    await db.query(`INSERT INTO orders (user_id, total, stripe_id) VALUES (${userId}, ${total}, '${charge.id}')`);
    await mailer.send(user.email, `Order confirmed! Total: $${total}`);
    res.json({ success: true, total });
  } catch (e) {
    res.status(500).json({ error: "Something went wrong" });
  }
});
```

---

### Hard — design review

Review this module and write a list of every clean code violation you can find. Then rewrite it cleanly.

```js
const x = [];
let f = false;

export function a(item) {
  if (item && item.n && item.p > 0) {
    if (!f) {
      x.push(item);
      if (x.length > 100) {
        x.splice(0, x.length - 100);
      }
    } else {
      console.log("locked");
    }
  }
}

export function b() {
  f = true;
  setTimeout(() => f = false, 5000);
}

export function c(id) {
  for (let i = 0; i < x.length; i++) {
    if (x[i].id === id) {
      x.splice(i, 1);
      return true;
    }
  }
  return false;
}

export function d() {
  return x.map(i => ({ n: i.n, p: i.p }));
}
```

---

## 17. Quick Reference

```
NAMING
  Variables   — what the data IS (noun): currentUser, orderTotal, isAdmin
  Functions   — what they DO (verb phrase): fetchUser, validateEmail, calculateTax
  Booleans    — is/has/can/should: isLoading, hasErrors, canEdit, shouldRetry
  Constants   — UPPER_SNAKE_CASE or descriptive camelCase: MAX_RETRIES, ONE_DAY_MS

FUNCTIONS
  Single responsibility — one job per function
  Guard clauses first  — return early, keep happy path unindented
  ≤ 3 params          — use options object for more
  No side effects in predicates

COMMENTS
  Best comment = well-named function
  DO comment:   WHY (business rules), warnings, non-obvious algorithms, public API
  DON'T comment: WHAT (restate code), commented-out code, changelogs

DRY    — one authoritative source of truth for each piece of knowledge
YAGNI  — don't build it until you need it
SRP    — one reason to change per unit
OCP    — add behaviour by extension, not modification
DI     — inject dependencies, don't hard-code them
LoD    — don't reach through object chains

MAGIC VALUES
  const ONE_DAY_MS = 24 * 60 * 60 * 1000;
  const ROLES = Object.freeze({ ADMIN: "admin" });

ERRORS
  Never: catch (e) {}
  Always: log, handle specifically, re-throw unknown

ASYNC
  async/await over callbacks
  Promise.all for independent operations
```

---

## 18. Connected Topics

- **73 — Custom error classes** — specific, typed errors are a clean code requirement
- **75 — Immutability** — pure functions with no side effects are foundational to clean code
- **76 — Module pattern** — intentional public API, hidden internals
- **77 — Observer pattern** — open/closed principle in action (add subscribers, don't modify publisher)
- **59 — ES Modules** — file organisation, single-responsibility per module
- **42 — Classes** — SRP and dependency injection with class constructors
- **Everything in this curriculum** — clean code is the synthesis of every concept covered. Every topic you've learned is a tool. Clean code is knowing when and how to use each one.
