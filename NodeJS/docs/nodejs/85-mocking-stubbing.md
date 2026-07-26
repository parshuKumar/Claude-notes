# 85 — Mocking and stubbing

## What is this?

Mocking and stubbing means replacing a real dependency — a database call, an email service, a payment API, a file write — with a fake version you fully control, so your test never actually touches the real thing. Think of it like a flight simulator for pilots: the cockpit controls are real, but the "engine," "weather," and "runway" are fake, letting the pilot practice every scenario (including engine failure) without risking a real plane. Jest gives you `jest.fn()` for fake functions, `jest.spyOn()` to watch or replace a method on a real object, and `jest.mock()` to swap out an entire module — while `nock` does the same job specifically for outgoing HTTP requests.

---

## Why does it matter for backend development?

Backend code is full of side effects: sending emails, charging cards, calling third-party APIs, writing to disk, querying a database. If your tests actually did these things, every test run would send real emails, cost real money, and depend on external services being online — tests would be slow, flaky, and dangerous to run repeatedly. Mocking lets you test your own logic (validation, error handling, response shaping) in isolation, at millisecond speed, with zero real-world side effects. `nock` specifically solves the "my Node service calls a third-party REST API" problem — extremely common in backend work (payment gateways, weather APIs, SMS providers) — by intercepting `http`/`https` requests before they leave the process.

---

## Syntax / API

```js
// jest.fn() — creates a standalone fake function you can inspect
const sendEmailMock = jest.fn();               // a fake function, does nothing by default
sendEmailMock.mockReturnValue('sent');          // makes every call return 'sent'
sendEmailMock('user@test.com');                 // call it like a normal function
expect(sendEmailMock).toHaveBeenCalledWith('user@test.com'); // assert on how it was called

// jest.spyOn() — wraps an EXISTING method on a real object, can restore it later
const emailService = require('./emailService'); // the real module
const spy = jest.spyOn(emailService, 'send');   // watch (or replace) emailService.send
spy.mockResolvedValue({ status: 'queued' });     // make it resolve with a fake value
// ... run code that calls emailService.send() ...
spy.mockRestore();                              // put the ORIGINAL method back

// jest.mock() — auto-mocks an entire module, every export becomes a jest.fn()
jest.mock('./db');                              // all exports of db.js are now fakes
const db = require('./db');
db.findUserById.mockResolvedValue({ id: 1, name: 'Asha' }); // control what it returns

// nock — intercepts outgoing HTTP calls made with fetch/axios/http.request
const nock = require('nock');
nock('https://api.paymentgateway.com')          // target base URL
  .post('/charge')                              // method + path to intercept
  .reply(200, { status: 'success', chargeId: 'ch_123' }); // fake response to return
```

---

## How it works — line by line

`jest.fn()` builds a brand-new fake function from nothing. It records every call made to it (arguments, return value, how many times) so you can later assert `expect(fn).toHaveBeenCalledTimes(1)`. By default it returns `undefined`, but `mockReturnValue()` or `mockResolvedValue()` (for promises) tells it what to hand back.

`jest.spyOn(object, 'methodName')` is different — it takes a method that already exists on a real object and wraps it. Unless you call `.mockImplementation()` or `.mockResolvedValue()` on the spy, the original method still runs underneath; the spy just watches. This is useful when you want to verify a real method was called correctly without fully replacing its behavior — or replace it temporarily and restore it afterward with `mockRestore()`.

`jest.mock('./path')` works at the module level. Jest intercepts every `require('./path')` call anywhere in the test file and swaps in an automatic mock where every exported function becomes a `jest.fn()`. This is the standard way to fake an entire database or API-client module so your unit test only exercises your own logic, never the real dependency.

`nock` works one level lower — at the network layer. It patches Node's built-in HTTP client so that any outgoing request matching the URL/method/path you declared never reaches the real internet; instead it returns the canned response you specified. Anything that doesn't match a declared interceptor either throws (in strict mode) or falls through to the real network, which is why you always call `nock.disableNetConnect()` in test setup to catch accidental real calls.

---

## Example 1 — basic

```js
// File: src/utils/notifier.js
// A tiny module that depends on an external "mailer" to send notifications.

function notifyUser(mailer, userId, message) {
  // Delegates the actual sending to whatever mailer object is passed in
  return mailer.send(userId, message);
}

module.exports = { notifyUser };
```

```js
// File: src/utils/notifier.test.js
// Demonstrates jest.fn() for a plain fake dependency.

const { notifyUser } = require('./notifier');

test('notifyUser calls mailer.send with the right arguments', () => {
  // Create a fake mailer object with a fake `send` function
  const fakeMailer = {
    send: jest.fn().mockResolvedValue({ delivered: true }), // fake resolves like a real async call
  };

  const userId = 'user_42';                      // realistic id, not "foo"
  const message = 'Your order has shipped';       // the notification text

  // Call our real function, but pass the FAKE mailer instead of a real one
  return notifyUser(fakeMailer, userId, message).then((result) => {
    // Assert the fake was called exactly once
    expect(fakeMailer.send).toHaveBeenCalledTimes(1);
    // Assert it was called with the exact arguments we expect
    expect(fakeMailer.send).toHaveBeenCalledWith(userId, message);
    // Assert our function correctly returned the mailer's response
    expect(result).toEqual({ delivered: true });
  });
});
```

---

## Example 2 — real world backend use case

```js
// File: src/services/userService.js
// Real service: looks up a user in the DB, then charges them via a payment API.

const db = require('../db');                    // module we'll mock in the test
const paymentClient = require('../paymentClient'); // module we'll mock too

async function chargeUserForSubscription(userId, amountCents) {
  const user = await db.findUserById(userId);     // hits the real DB in production
  if (!user) {
    throw new Error(`User not found: ${userId}`);  // guard clause — no user, no charge
  }

  // Calls the real payment gateway in production
  const charge = await paymentClient.createCharge({
    customerId: user.paymentCustomerId,           // stored payment provider id
    amount: amountCents,                          // amount in the smallest currency unit
  });

  return { userId, chargeId: charge.id, status: charge.status }; // shape the response
}

module.exports = { chargeUserForSubscription };
```

```js
// File: src/services/userService.test.js
// Mocks BOTH the db module and the paymentClient module — full isolation.

jest.mock('../db');                              // auto-mock: every export becomes jest.fn()
jest.mock('../paymentClient');                    // same for the payment gateway module

const db = require('../db');
const paymentClient = require('../paymentClient');
const { chargeUserForSubscription } = require('./userService');

test('charges an existing user successfully', async () => {
  const userId = 'user_42';
  const amountCents = 1999;                       // $19.99 in cents

  // Tell the mocked db what to return when findUserById is called
  db.findUserById.mockResolvedValue({
    id: userId,
    paymentCustomerId: 'cus_abc123',
  });

  // Tell the mocked payment client what to return when createCharge is called
  paymentClient.createCharge.mockResolvedValue({
    id: 'ch_789',
    status: 'succeeded',
  });

  const result = await chargeUserForSubscription(userId, amountCents);

  // Verify the payment client received the correctly mapped arguments
  expect(paymentClient.createCharge).toHaveBeenCalledWith({
    customerId: 'cus_abc123',
    amount: amountCents,
  });

  // Verify our function returned the expected shape
  expect(result).toEqual({ userId, chargeId: 'ch_789', status: 'succeeded' });
});

test('throws when the user does not exist', async () => {
  db.findUserById.mockResolvedValue(null);        // simulate "no such user"

  // Assert the promise rejects with the expected error message
  await expect(chargeUserForSubscription('user_999', 500))
    .rejects.toThrow('User not found: user_999');

  // paymentClient must NEVER be called if the user lookup failed
  expect(paymentClient.createCharge).not.toHaveBeenCalled();
});
```

```js
// File: src/services/weatherService.test.js
// Real HTTP mocking with nock — for code that calls an external REST API directly.

const nock = require('nock');
const { getWeatherForCity } = require('./weatherService'); // uses fetch/axios internally

beforeAll(() => {
  nock.disableNetConnect();                       // block ANY real network call in tests
});

afterEach(() => {
  nock.cleanAll();                                // remove leftover interceptors between tests
});

test('returns parsed weather data for a valid city', async () => {
  const city = 'Mumbai';

  // Intercept the exact outgoing request our service will make
  nock('https://api.weatherapi.com')
    .get('/v1/current.json')
    .query({ q: city, key: 'test-api-key' })       // must match query params exactly
    .reply(200, { location: { name: city }, current: { temp_c: 31 } });

  const result = await getWeatherForCity(city, 'test-api-key');

  expect(result.temp_c).toBe(31);                 // service correctly parsed the fake response
});

test('throws when the weather API is down', async () => {
  nock('https://api.weatherapi.com')
    .get('/v1/current.json')
    .query(true)                                  // match any query string
    .reply(500, { error: 'internal server error' }); // simulate the API failing

  await expect(getWeatherForCity('Mumbai', 'test-api-key'))
    .rejects.toThrow();                           // our service should surface the failure
});
```

---

## Common mistakes

### Mistake 1 — Forgetting to restore a spy, leaking state into other tests

```js
// ❌ WRONG — spy replaces the real method and is never restored,
// so every LATER test in the file also gets the fake behavior
test('sends a welcome email', () => {
  jest.spyOn(emailService, 'send').mockResolvedValue({ status: 'queued' });
  // ... test runs, but the spy is still active for the next test ...
});

// ✅ CORRECT — always restore in afterEach, or use mockRestore per test
afterEach(() => {
  jest.restoreAllMocks();                         // puts every spied method back to the original
});

test('sends a welcome email', () => {
  jest.spyOn(emailService, 'send').mockResolvedValue({ status: 'queued' });
  // ... test runs; afterEach cleans up automatically ...
});
```

### Mistake 2 — Mocking a module but forgetting to reset mock call history between tests

```js
// ❌ WRONG — mock call counts accumulate ACROSS tests in the same file,
// so "toHaveBeenCalledTimes(1)" can fail even when the logic is correct
jest.mock('../db');
const db = require('../db');

test('first test calls findUserById once', async () => {
  db.findUserById.mockResolvedValue({ id: 'user_1' });
  await chargeUserForSubscription('user_1', 500);
  expect(db.findUserById).toHaveBeenCalledTimes(1); // passes alone
});

test('second test also calls findUserById once', async () => {
  db.findUserById.mockResolvedValue({ id: 'user_2' });
  await chargeUserForSubscription('user_2', 500);
  expect(db.findUserById).toHaveBeenCalledTimes(1); // FAILS — actually called 2 times total
});

// ✅ CORRECT — clear mock history before every test
beforeEach(() => {
  jest.clearAllMocks();                            // resets call counts and mock.calls arrays
});
```

### Mistake 3 — Leaving nock active and accidentally hitting the real network

```js
// ❌ WRONG — no cleanup between tests means leftover interceptors,
// or worse, a typo in the mocked path silently falls through to the REAL API
test('fetches user data', async () => {
  nock('https://api.example.com').get('/users/42').reply(200, { id: 42 });
  // ... if the code actually requests '/user/42' (typo), nock does not match
  // and the request escapes to the real internet unless net connect is disabled
});

// ✅ CORRECT — disable real network globally and clean interceptors every test
beforeAll(() => {
  nock.disableNetConnect();                        // any unmatched request throws instead of going out
});

afterEach(() => {
  expect(nock.isDone()).toBe(true);                // fails loudly if an expected call never happened
  nock.cleanAll();                                 // remove all interceptors before the next test
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a module `src/utils/logger.js` that exports a function `logActivity(loggerClient, userId, action)` which simply calls `loggerClient.log(userId, action)` and returns whatever the client returns. Then write a test using `jest.fn()` that:
1. Creates a fake `loggerClient` object with a fake `log` method returning `{ logged: true }`.
2. Calls `logActivity` with a realistic `userId` (e.g. `'user_17'`) and action string (e.g. `'login'`).
3. Asserts the fake `log` method was called exactly once with the correct arguments.
4. Asserts the function's return value matches what the fake returned.

```js
// Write your code here
```

---

### Exercise 2 — medium

You have a module `src/services/orderService.js` with a function `cancelOrder(orderId)` that internally calls `db.findOrderById(orderId)` and, if found and not already shipped, calls `db.updateOrderStatus(orderId, 'cancelled')`. Using `jest.mock('../db')`:
1. Write a test where the order exists and is `status: 'pending'` — assert `updateOrderStatus` is called with `(orderId, 'cancelled')`.
2. Write a second test where the order's `status` is already `'shipped'` — assert the function throws an error like `Cannot cancel a shipped order` and that `updateOrderStatus` is **never** called.
3. Write a third test where `db.findOrderById` resolves `null` (order not found) — assert an appropriate error is thrown.
4. Use `beforeEach` with `jest.clearAllMocks()` so call counts don't leak between your three tests.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build and test a `src/services/smsService.js` module with a function `sendVerificationCode(phoneNumber)` that:
1. Generates a 6-digit code (you can stub `Math.random` via `jest.spyOn(Math, 'random')` to make it deterministic in tests).
2. Sends the code by making an HTTP POST request to `https://api.smsprovider.com/v1/send` with a JSON body `{ to: phoneNumber, message: '...' }` (use `fetch` or `axios` — your choice).
3. Returns `{ phoneNumber, code, status: 'sent' }` on success, and throws a descriptive error if the provider responds with a non-2xx status.

Then write a test file using `nock` that:
1. Calls `nock.disableNetConnect()` in `beforeAll` and `nock.cleanAll()` in `afterEach`.
2. Stubs `Math.random` so the generated code is predictable, then restores it afterward.
3. Intercepts the POST request and asserts the request body sent to the provider matches exactly what your service should have sent.
4. Tests one success case (provider replies 200) and one failure case (provider replies 503), asserting your function's behavior in both.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
JEST.FN() — standalone fake function
  jest.fn()                          → fake fn, returns undefined by default
  .mockReturnValue(x)                → always returns x (sync)
  .mockResolvedValue(x)              → always resolves with x (async)
  .mockRejectedValue(err)            → always rejects with err (async)
  .mockImplementation(fn)            → custom fake logic
  toHaveBeenCalled()                 → was it called at all
  toHaveBeenCalledWith(...args)      → was it called with these exact args
  toHaveBeenCalledTimes(n)           → exact call count

JEST.SPYON() — wrap a method on a REAL object
  jest.spyOn(obj, 'method')          → watches the real method (still runs by default)
  jest.spyOn(obj, 'method').mockResolvedValue(x)  → replaces behavior too
  spy.mockRestore()                  → puts the ORIGINAL method back
  jest.restoreAllMocks()             → restore every spy at once (put in afterEach)

JEST.MOCK() — auto-mock an entire module
  jest.mock('./path')                → every export of that module becomes jest.fn()
  Must be called at the TOP of the test file (Jest hoists it above imports)
  jest.clearAllMocks()               → reset call history, keep mock implementations
  jest.resetAllMocks()               → reset call history AND remove mock implementations

NOCK — mock outgoing HTTP requests
  nock('https://host.com').get('/path').reply(200, body)   → intercept + fake response
  nock(...).post('/path', requestBodyMatcher).reply(...)   → match on request body too
  nock.disableNetConnect()           → block ALL real network calls (always set in test setup)
  nock.cleanAll()                    → clear interceptors between tests (afterEach)
  nock.isDone()                      → true only if every declared interceptor was actually hit

GOTCHAS
  - jest.mock() calls are hoisted above require/import — order in the file doesn't matter
  - Forgetting jest.clearAllMocks()/restoreAllMocks() leaks state between tests
  - nock interceptors must match method + path + query/body EXACTLY or they silently miss
  - Prefer mocking at your OWN module boundary (db, paymentClient) over mocking node internals
```

---

## Connected topics

- **84 — Testing async code** — mocked functions almost always return promises (`mockResolvedValue`/`mockRejectedValue`), so writing correct async assertions is a prerequisite here.
- **86 — Testing Express routes** — supertest tests for routes typically mock the same service/db layer covered here so route tests don't hit a real database or third-party API.
- **82 — Testing fundamentals** — unit tests rely on mocking to isolate the unit under test from its dependencies, which is the whole reason mocking exists in the test pyramid.
