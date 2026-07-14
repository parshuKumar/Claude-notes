# 19 — DRY, KISS, YAGNI — The Clean Design Trinity
## Category: LLD Fundamentals

---

## What is this?

DRY, KISS, and YAGNI are three short rules that keep code from rotting. **DRY** (Don't Repeat Yourself) says every piece of *knowledge* should live in exactly one place. **KISS** (Keep It Simple, Stupid) says pick the simplest thing that works. **YAGNI** (You Aren't Gonna Need It) says don't build features you only *imagine* you'll need.

Think of them as three people arguing in your head while you code. DRY says "you wrote that twice!" KISS says "nobody can read this." YAGNI says "who asked for that?" Good design is the compromise the three of them settle on — and the most interesting part of this doc is what happens when one of them wins too hard.

---

## Why does it matter?

Without DRY, a single business rule (say, "orders over ₹5000 get free shipping") is copy-pasted into 6 files. Someone changes the threshold to ₹3000 and updates 5 of them. The 6th silently keeps charging customers. That's a bug you can't test your way out of — you have to *find* it.

Without KISS, you get code that only its author can modify, and its author left the company. Without YAGNI, you get a 400-line "pluggable notification framework" that has exactly one plugin: email.

But here's the thing almost nobody tells you: **each of these principles is dangerous when over-applied.** Over-applied DRY is a leading cause of tangled, coupled codebases — you merged two things that merely *looked* alike, and now a change to billing breaks reporting. Over-applied YAGNI is how you end up with a system that can't accept a second payment provider without a rewrite.

**Interview angle:** These come up in every LLD round, usually as a follow-up: "This looks duplicated — would you extract it?" The junior answer is "yes, DRY." The senior answer is "depends — do these two change for the same reason?" That one sentence separates candidates.

**Real-work angle:** Every code review you ever do involves these three. Knowing *when not to* apply them is what makes your reviews trusted.

---

## The core idea — explained simply

### The Recipe Book Analogy

You run a bakery. You keep a recipe book.

**DRY, done right.** The oven temperature for all your breads is 220°C. You write `BREAD_TEMP = 220°C` once at the front of the book, and every bread recipe says "bake at BREAD_TEMP." One day you get a new oven that runs hot, so you change one number and every bread recipe is correct. That's DRY: *one fact, one home.*

**DRY, done wrong.** Your sourdough recipe says "bake 40 minutes." Your banana bread recipe also says "bake 40 minutes." A clever assistant notices the duplication, writes `BAKE_TIME = 40` at the front, and makes both recipes reference it. Six months later you tune the sourdough to 45 minutes. Now you must either change `BAKE_TIME` (breaking banana bread), or add a flag: `BAKE_TIME(isSourdough)`. The two 40s **looked** the same but were never the same fact. They were a **coincidence**, not a shared truth.

That is the whole lesson, and it's the one most people miss:

> **DRY is about duplicated KNOWLEDGE, not duplicated TEXT.**

The original wording from *The Pragmatic Programmer* is: "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system." Not "every piece of *code*." **Knowledge.**

**KISS, in the bakery.** A recipe that says "combine flour, water, salt, yeast; knead 10 min; rest 1 hr" is usable by anyone. A recipe that says "apply hydration protocol H-3 with gluten-development phase per Appendix B" is technically the same instructions and useless at 5am.

**YAGNI, in the bakery.** You buy a ₹2,00,000 industrial dough sheeter because "we might do croissants someday." Two years later it is a very expensive table. You never did croissants.

**Mapping the analogy back:**

| Bakery thing | Software thing |
|---|---|
| `BREAD_TEMP = 220°C` used by all breads | A shared constant/function representing one real rule |
| Two recipes that both say "40 min" by coincidence | Two functions with identical bodies that change for different reasons |
| Adding `BAKE_TIME(isSourdough)` | The **parameter-flag explosion** — the smell of a wrong abstraction |
| "Apply hydration protocol H-3" | Clever, compressed, unreadable code |
| The unused dough sheeter | The plugin system with one plugin |
| Waiting until three recipes agree before writing a rule | The **Rule of Three** |

---

## Key concepts inside this topic

### 1. DRY — one piece of knowledge, one home

A *piece of knowledge* is a rule the business would recognise: a tax formula, a validation rule, a state machine, a threshold. When that rule changes, it should change in exactly one file.

**Bad — the rule lives in three places:**

```javascript
// routes/checkout.js
if (order.total > 5000) order.shipping = 0;

// services/cart-preview.js
const shipping = cart.total > 5000 ? 0 : 99;    // the same rule, duplicated

// jobs/nightly-invoice.js
const ship = invoice.amount > 5000 ? 0 : 99;    // duplicated AGAIN
```

Marketing drops the free-shipping bar to ₹3000. You update checkout and cart preview. Nightly invoicing keeps charging ₹99. Support tickets follow.

**Good — the rule has one home:**

```javascript
// domain/shipping-policy.js
// The single authoritative source of the free-shipping rule.
// If marketing changes the threshold, THIS is the only file that moves.
export const FREE_SHIPPING_THRESHOLD = 3000;
export const FLAT_SHIPPING_FEE = 99;

export function shippingFeeFor(subtotal) {
  return subtotal >= FREE_SHIPPING_THRESHOLD ? 0 : FLAT_SHIPPING_FEE;
}
```

Every caller now imports `shippingFeeFor`. This is real DRY: the *knowledge* moved, not just the characters.

DRY also applies beyond code — a rule duplicated between your DB `CHECK` constraint, your API validator, and your frontend form is triplicated knowledge. Sometimes you accept that duplication deliberately (the frontend copy exists for fast UX feedback); the point is to *know* you're doing it and to name the one that's authoritative.

### 2. The trap — duplicated text is not duplicated knowledge

Here are two functions from a real e-commerce codebase. Look how identical they are.

```javascript
// billing/format.js
function formatMoneyForInvoice(amountInPaise) {
  const rupees = amountInPaise / 100;
  return `₹${rupees.toFixed(2)}`;
}

// analytics/format.js
function formatMoneyForDashboard(amountInPaise) {
  const rupees = amountInPaise / 100;
  return `₹${rupees.toFixed(2)}`;
}
```

Byte-for-byte the same body. A reviewer says "DRY violation — extract it." So you do:

```javascript
// shared/format-money.js
export function formatMoney(amountInPaise) {
  const rupees = amountInPaise / 100;
  return `₹${rupees.toFixed(2)}`;
}
```

**Now watch what actually happens over the next year.**

Legal requires invoices to be locale-aware and to show the currency code (`INR 1,234.50`):

```javascript
export function formatMoney(amountInPaise, { forInvoice = false } = {}) {
  const rupees = amountInPaise / 100;
  if (forInvoice) {
    return `INR ${rupees.toLocaleString('en-IN', { minimumFractionDigits: 2 })}`;
  }
  return `₹${rupees.toFixed(2)}`;
}
```

Then the dashboard team wants compact numbers (`₹1.2L`) for large values. Flag #2. Then invoices for export orders must render USD. Flag #3. Then the dashboard wants the symbol hidden inside chart axes. Flag #4. You end up here:

```javascript
// The wrong abstraction, fully grown. Nobody dares touch it.
export function formatMoney(amountInPaise, {
  forInvoice = false,     // named after a CALLER, not a behaviour
  compact = false,        // named after a CALLER, not a behaviour
  currency = 'INR',
  hideSymbol = false,
  locale = 'en-IN',
} = {}) {
  const value = amountInPaise / 100;
  if (forInvoice && compact) throw new Error('unsupported combination'); // <-- the tell
  // ...30 lines of branching that serve two callers who share nothing
}
```

This is the **parameter-flag explosion**. The signs are unmistakable:

- Boolean flags whose names describe the *caller*, not the behaviour (`forInvoice`).
- Combinations that are invalid, so you throw — the function is really two functions in a trench coat.
- Every change made for the invoice team requires a regression run for the dashboard team. **You coupled two teams who have nothing to do with each other.**

**The un-merge.** The fix is to reverse the DRY-ification:

```javascript
// billing/format.js — owned by Billing. Changes when tax/legal changes.
export function formatInvoiceAmount(amountInPaise, currency = 'INR') {
  const value = amountInPaise / 100;
  return `${currency} ${value.toLocaleString('en-IN', { minimumFractionDigits: 2 })}`;
}

// analytics/format.js — owned by Data. Changes when the dashboard changes.
export function formatDashboardAmount(amountInPaise, { compact = false } = {}) {
  const value = amountInPaise / 100;
  if (compact && value >= 100000) return `₹${(value / 100000).toFixed(1)}L`;
  return `₹${value.toFixed(2)}`;
}
```

Yes, `amountInPaise / 100` now appears twice. That is **fine.** The duplication is three characters of arithmetic; the coupling you removed was worth weeks of pain.

> **The test that decides it:** *"If requirement A changes, must requirement B change too?"* If **no** — they are different knowledge. Leave them apart, no matter how identical they look.

This is really the Single Responsibility Principle in disguise: a module should have **one reason to change**. Two callers with different reasons to change should not share a function. (Recall from [14 — SOLID: Single Responsibility](./14-solid-single-responsibility.md) that "reason to change" means an *actor* — a person or team — who requests the change.)

### 3. The counterweights — Rule of Three, WET, and AHA

Because premature DRY is so costly, the industry evolved three antibodies.

**The Rule of Three (Martin Fowler).** Duplicate once: fine. Duplicate twice: note it. On the **third** occurrence, extract. Why three? Two data points cannot tell you the shape of the abstraction — *every* pair of things looks similar. The third instance reveals what is genuinely common and what is incidental.

**WET — "Write Everything Twice."** A joking backronym with a serious point: writing it twice before abstracting isn't a sin, it's *evidence gathering*.

**AHA — "Avoid Hasty Abstractions" (Kent C. Dodds).** The rule: *"Prefer duplication over the wrong abstraction, and optimize for change first."* You should feel **pain** from the duplication before you remove it. If no pain has arrived yet, you don't know enough to abstract well.

**Sandi Metz's line, which is this whole doc in one sentence:**

> **"Duplication is far cheaper than the wrong abstraction."**

Her reasoning: when you find a wrong abstraction, the natural instinct is to add a parameter (see §2). Each parameter makes the next person *less* likely to unpick it — the code gets harder to remove precisely as it gets worse. Duplication, by contrast, is trivially removable the moment you understand it. **The cost of duplication is linear. The cost of the wrong abstraction compounds.**

| Cost dimension | Duplication | Wrong abstraction |
|---|---|---|
| How you find bugs | grep, then fix N copies | you often don't — flag combos hide them |
| Cost to reverse | low, mechanical | high, and grows every sprint |
| Blast radius | one module at a time | every caller, forever |
| Failure mode | you miss a copy | you break an unrelated team |

### 4. KISS — the simplest thing that works

Simple ≠ short. Simple = **the code a competent colleague understands on the first read, at 2am, during an incident.**

**Bad — clever, compressed, unreadable:**

```javascript
// "Group orders by customer and sum their totals." Technically correct. Do not do this.
const report = orders.reduce((a, o) => ((a[o.customerId] = (a[o.customerId] || 0) + o.total), a), {});
```

Problems: a comma operator hidden inside parentheses, an assignment used as an expression, mutation of the accumulator, and no name for anything. Reading it costs 30 seconds. It will cost 30 seconds *every time*, for *every* reader, forever.

**Good — boring, obvious, one screen:**

```javascript
function totalsByCustomer(orders) {
  const totals = new Map();
  for (const order of orders) {
    const current = totals.get(order.customerId) ?? 0;
    totals.set(order.customerId, current + order.total);
  }
  return totals;
}
```

Same complexity — O(n) — and roughly the same number of lines. But it names the operation, reads top to bottom, and you can set a breakpoint on any line. **Readable beats clever every time**, because reading happens ten times more often than writing.

**KISS applies at the design level too:**

```javascript
// Over-engineered: an "event-driven" abstraction for something with ONE listener.
class EventBus { /* 60 lines of subscribe/unsubscribe/wildcards/priorities */ }
bus.subscribe('user.created', 'sendWelcomeEmail', { priority: 5 });

// Simple: call the function.
await sendWelcomeEmail(user);
```

The EventBus becomes correct when you have *many* listeners that must not know about each other. With one listener, it is pure ceremony — plus an indirection that hides the call graph from every future reader.

**When "simple" is the wrong word:** KISS does not mean "avoid all structure." A single 800-line `index.js` is *simple to write* and horrifying to change. The correct reading is **"minimise total complexity,"** not "minimise the number of files."

### 5. YAGNI — build for today, design so tomorrow is possible

YAGNI comes from Extreme Programming (Ron Jeffries): *"Always implement things when you actually need them, never when you just foresee that you need them."*

The cost of a speculative feature is not just the day you spent writing it. It is:

1. **Build cost** — the day itself.
2. **Carry cost** — every future reader must understand it, every refactor must preserve it, every CI run must test it.
3. **Repair cost** — when the real requirement arrives shaped differently, you must *remove* your guess before you can build the real thing.
4. **Opportunity cost** — the thing you didn't build instead.

Only cost #1 is visible on the day you make the decision. That is why speculation feels cheap and isn't.

**Bad — a config-driven plugin system for a feature with exactly one implementation:**

```javascript
// notifications/registry.js
// Built "so we can add SMS and Slack later." We never did.
import fs from 'node:fs';
import path from 'node:path';

class NotificationChannelRegistry {
  #channels = new Map();

  registerFromConfig(configPath) {
    const config = JSON.parse(fs.readFileSync(configPath, 'utf8'));
    for (const entry of config.channels) {
      if (!entry.enabled) continue;
      // Dynamic module loading: no static analysis, no IDE jump-to-definition,
      // and a typo in JSON becomes a runtime crash at boot.
      const Ctor = require(path.resolve(entry.module)).default;
      this.#channels.set(entry.name, new Ctor(entry.options));
    }
  }

  get(name) {
    const channel = this.#channels.get(name);
    if (!channel) throw new Error(`No channel registered: ${name}`); // a failure mode we invented
    return channel;
  }
}

class NotificationChannel {                       // the "interface"
  async send(_recipient, _payload) { throw new Error('not implemented'); }
}

class EmailChannel extends NotificationChannel {  // the ONLY implementation, ever
  constructor(options) { super(); this.from = options.from; }
  async send(recipient, payload) { /* smtp call */ }
}

// notifications/config.json
// { "channels": [ { "name": "email", "module": "./email-channel.js",
//                   "enabled": true, "options": { "from": "no-reply@shop.com" } } ] }

const registry = new NotificationChannelRegistry();
registry.registerFromConfig('./notifications/config.json');
await registry.get('email').send(user.email, { subject: 'Welcome', body: '...' });
```

Count what you now own: a JSON schema, a dynamic module loader, a registry, a base class, a brand-new "channel not found" runtime failure, and a stack trace that walks three indirections when SMTP times out at 3am. All to send one email.

**Good — the 20-line version that beats it:**

```javascript
// notifications/email.js
import nodemailer from 'nodemailer';

const transport = nodemailer.createTransport({ host: process.env.SMTP_HOST });

/**
 * Sends a transactional email. One function, one job.
 * The seam is here: every SMTP detail lives in THIS file. Swapping providers
 * is a one-file change — which is all the "flexibility" we actually needed.
 */
export async function sendEmail(to, { subject, body }) {
  if (!to) throw new Error('recipient required');
  await transport.sendMail({
    from: process.env.MAIL_FROM,
    to,
    subject,
    text: body,
  });
}
```

That's it. And crucially, **the 20-line version is not a dead end.** The day SMS genuinely arrives, you write `sendSms()`, then a 10-line dispatcher — and you now have *two real examples* to design the abstraction from. The abstraction you write then will be *right*, because reality informed it instead of imagination.

**YAGNI's own failure mode.** Taken too far, YAGNI becomes an excuse for a system that cannot evolve: hardcoded credentials, no seam between the domain and the database, business logic welded into HTTP handlers. The distinction that saves you:

- **Speculative features** — YAGNI says no. (A plugin registry with one plugin. A multi-tenant schema for one tenant. Configurable retry backoff nobody configures.)
- **Reversibility / seams** — YAGNI says nothing; keep them. (Putting the SMTP call behind `sendEmail()`. Keeping SQL inside one repository module instead of scattered across 40 route handlers.)

You are not required to *build* the future. You are required not to **weld the door shut** on it. A cheap seam — a function boundary, a repository module — costs almost nothing and preserves options. A framework costs a lot and destroys them.

### 6. Decision rules — WHEN to abstract

Run these gates in order. Extract only if you pass all four.

```
┌────────────────────────────────────────────────────────────┐
│  You spotted duplication. Should you extract it?           │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
        ┌───────────────────────────────────────┐
        │ 1. Is this the THIRD occurrence?      │──No──▶ WAIT. Duplicate it.
        │    (Rule of Three)                    │        Leave a // TODO note.
        └──────────────┬────────────────────────┘
                       │ Yes
                       ▼
        ┌───────────────────────────────────────┐
        │ 2. Do all copies change for the SAME  │──No──▶ STOP. Different knowledge.
        │    business reason? (the real test)   │        Keep them duplicated.
        └──────────────┬────────────────────────┘
                       │ Yes
                       ▼
        ┌───────────────────────────────────────┐
        │ 3. Can you name it in a DOMAIN word?  │──No──▶ You don't understand it yet.
        │    ("shipping policy", not "utils")   │        Not "helper". Not "manager".
        └──────────────┬────────────────────────┘
                       │ Yes
                       ▼
        ┌───────────────────────────────────────┐
        │ 4. Does it stay FLAG-FREE? (no bool   │──No──▶ It's two things wearing
        │    param named after a caller)        │        one name. Keep two.
        └──────────────┬────────────────────────┘
                       │ Yes
                       ▼
                  ✔  EXTRACT IT
```

Two heuristics worth memorising:

- **The flag test.** If your extracted function needs a boolean parameter to tell it *which caller* is calling, you extracted the wrong thing. Split it back.
- **The deletion test.** If the abstraction would be *easier to delete* than to understand, it isn't earning its keep.

### 7. How the three principles fight each other

They are not friends. They pull in different directions, and design is where you referee.

| Tension | What it looks like | How to settle it |
|---|---|---|
| **DRY vs KISS** | Extracting a shared function forces callers through 3 layers of indirection to save 4 lines | KISS wins under ~5 lines of trivial logic. Indirection has a cost — pay it only for real rules. |
| **DRY vs YAGNI** | "Let's make it generic *now* so it's reusable later" | YAGNI wins. Generic-for-one-caller is speculation, not reuse. |
| **KISS vs correctness** | The simple version ignores timezones / retries / partial failure | Correctness wins. KISS means "no *unnecessary* complexity," not "no necessary complexity." |
| **YAGNI vs evolvability** | "Don't add a repository module, we only use Postgres" | Split it: skip the *feature* (multi-DB support), keep the *seam* (one module owns SQL). |

**The one-line arbitration rule:** *Optimise for the change you can see; keep the seams for the change you can't.*

---

## Visual / Diagram description

### Diagram 1: The cost curves — why duplication is cheaper than the wrong abstraction

```
  Cost of
  change
    ▲
    │                                       ╱   WRONG ABSTRACTION
    │                                    ╱      (each new flag makes the next
    │                                 ╱          change harder — it COMPOUNDS)
    │                              ╱
    │                          ╱
    │                     ╱
    │                ╱                        ──────────────  DUPLICATION
    │           ╱                   ─────────                 (linear: N copies,
    │      ╱          ─────────                                fix N places)
    │  ╱   ─────────
    │ ─────
    └──────────────────────────────────────────────────────────▶ requirements over time
      ▲                    ▲
      │                    │
   You extracted        The 4th flag lands.
   too early.           Now nobody dares refactor it.
```

**What this shows.** Duplication's cost grows *linearly* with the number of copies — annoying, but always fixable with a mechanical edit. The wrong abstraction's cost grows *super-linearly*, because each parameter you bolt on both worsens the design **and** raises the cost of undoing it. That asymmetry — cheap-to-reverse versus expensive-to-reverse — is exactly why "wait for the third case" is the safe default.

### Diagram 2: The wrong DRY-ification, and the un-merge

```
STEP 1 — Two functions. Identical text, different owners.

   ┌────────────────────────┐      ┌────────────────────────┐
   │ billing/format.js      │      │ analytics/format.js    │
   │ formatMoneyForInvoice  │      │ formatMoneyForDashboard│
   │   amt/100 → "₹x.xx"    │      │   amt/100 → "₹x.xx"    │
   └────────────────────────┘      └────────────────────────┘
      owner: Billing team              owner: Data team
      changes when: tax law changes     changes when: UX changes

STEP 2 — "It's a DRY violation!" Both now point at one function.

                ┌──────────────────────────────┐
                │  shared/format-money.js      │
                │  formatMoney(amt)            │
                └───────▲──────────────▲───────┘
                        │              │
              ┌─────────┴────┐   ┌─────┴────────┐
              │   Billing    │   │  Analytics   │
              └──────────────┘   └──────────────┘
                  COUPLED. Neither team knows it yet.

STEP 3 — Requirements diverge. Flags accumulate.

    ┌─────────────────────────────────────────────────────┐
    │  formatMoney(amt, {                                 │
    │      forInvoice,   ◀── flag named after a CALLER    │
    │      compact,      ◀── flag named after a CALLER    │
    │      currency,                                      │
    │      hideSymbol,                                    │
    │  })                                                 │
    │  if (forInvoice && compact) throw ...  ◀── THE TELL │
    └─────────────────────────────────────────────────────┘
       A change for Billing now breaks Analytics' tests.

STEP 4 — The UN-MERGE. Duplicate 3 chars of math. Free both teams.

   ┌──────────────────────────┐   ┌────────────────────────────┐
   │ billing/format.js        │   │ analytics/format.js        │
   │ formatInvoiceAmount(     │   │ formatDashboardAmount(      │
   │        amt, currency)    │   │        amt, {compact})      │
   └──────────────────────────┘   └────────────────────────────┘
        ONE reason to change            ONE reason to change
```

**What this shows.** The arrows in STEP 2 *are* the damage. The moment two independent modules point at one function, a change to that function is a change to **both** — and neither team can see the other's tests. STEP 4 restores independence at the price of one duplicated division. That is a bargain.

### Diagram 3: YAGNI — what the plugin system actually bought you

```
   SPECULATIVE (built for imagined SMS/Slack)     ACTUAL (what shipped)

        notifications/config.json                    sendEmail(to, msg)
                  │ parsed at boot                          │
                  ▼                                         ▼
        Registry.registerFromConfig()                   nodemailer
                  │ dynamic require()
                  ▼                                   1 file. 20 lines.
        NotificationChannel  (base class)             0 config files.
                  │ extended by...                    1 stack frame.
                  ▼                                   0 invented failure modes.
        EmailChannel   ◀── the only one, ever
                  │
                  ▼
             nodemailer

   5 files. ~140 lines. A runtime "channel not found".
   3 extra stack frames when SMTP times out at 3am.
   Zero second implementations, two years later.
```

**What this shows.** Every box on the left is a thing a future engineer must read, test, and keep alive — built to serve a second implementation that never arrived. On the right, when SMS *does* arrive, you write `sendSms()` plus a small dispatcher, and by then you have two real cases to shape the abstraction with. The left path pays up front for a guess; the right path pays later for a fact.

---

## Real world examples

### 1. Sandi Metz — "All the Little Things" (RailsConf 2014)

This talk is the origin of the industry's most-quoted line on the topic: **"duplication is far cheaper than the wrong abstraction."** Her argument, delivered while live-refactoring code on stage, is that programmers get *stuck* with bad abstractions because the sunk-cost instinct pushes them to add a parameter rather than inline the code back and start over. Her prescribed remedy is exactly the un-merge in Diagram 2: re-inline the shared function into its callers, delete the shared code, let the callers diverge, and only then look for the *real* commonality. If you quote one thing in an interview, quote this.

### 2. Kent C. Dodds and AHA — the React component lesson

Dodds coined **AHA (Avoid Hasty Abstractions)** after watching React codebases accumulate "flexible" components that took 15 props and served three callers badly — the parameter-flag explosion, in component form. His guidance, *"prefer duplication over the wrong abstraction, and optimize for change first,"* became standard advice in component design: copy the component, let the two copies diverge for a while, and extract only once you can see which parts are genuinely the same. It shows in how modern component libraries are shaped — small composable primitives rather than one mega-component with a prop for every situation.

### 3. Go's standard library and "a little copying"

The Go community codified this into an official proverb from Rob Pike's *Go Proverbs* talk: **"A little copying is better than a little dependency."** Go's standard library deliberately duplicates small helper routines across packages rather than creating a shared `utils` package that would couple unrelated packages together. It's the same trade seen from the dependency-management side: a copied five-line function has a maintenance cost of roughly zero, while a shared dependency creates a permanent coupling plus a version to keep in sync. The Node ecosystem learned the same lesson the hard way — the 2016 `left-pad` incident, where the un-publishing of an 11-line package broke builds across the internet, is the reductio ad absurdum of "never duplicate anything, always depend."

---

## Trade-offs

**DRY**

| Pros | Cons |
|---|---|
| One place to change a business rule | Creates coupling between every caller |
| Bugs fixed once, everywhere | Premature extraction produces the flag explosion |
| Rules become explicit and named | Adds indirection — readers must jump between files |
| Domain knowledge lives in one artifact | "Looks the same" ≠ "is the same"; easy to misjudge |

**KISS**

| Pros | Cons |
|---|---|
| Anyone can read it and change it | Can be used to justify having no structure at all |
| Fast to debug during an incident | May under-serve a genuinely complex domain |
| Fewer moving parts, fewer bugs | Naive-simple can hide performance cliffs (O(n²) loops) |

**YAGNI**

| Pros | Cons |
|---|---|
| Ships faster; less code to carry | Taken literally, produces unevolvable systems |
| Design informed by real requirements, not guesses | Retrofitting a truly-needed capability (multi-tenancy, i18n) is expensive |
| Fewer abstractions to explain to new hires | Some decisions really are one-way doors |

**Where each principle turns toxic:**

| Principle | Over-applied, it becomes... | The symptom you'll see |
|---|---|---|
| **DRY** | Coupling. A "shared" module everything depends on. | Boolean flags; a `utils.js` with 40 unrelated exports; one change breaks six teams |
| **KISS** | Naivety. No layering, no seams. | 800-line route handlers; raw SQL inside HTTP controllers |
| **YAGNI** | Paralysis-by-rewrite. | "We can't add a second payment provider without rewriting checkout" |

**The sweet spot:**

- Duplicate **twice** without guilt. Extract on the **third** — and only if all copies change for the same reason.
- Write the **boring** version. Optimise for the reader at 2am, not for the author showing off.
- Build **only what's asked**, but keep the **seams** cheap: one module owns SMTP, one owns SQL, one owns the pricing rule. Seams aren't speculation — they're how you stay able to say yes later.

**Rule of thumb:** *Prefer duplication over the wrong abstraction. Prefer boring over clever. Prefer today's requirement over tomorrow's guess — but never weld the door shut.*

---

## Common interview questions on this topic

### Q1: "You see two functions with identical bodies. Would you extract a shared one?"
**Hint:** Say "it depends — on whether they represent the same *knowledge*." Ask: do they change for the same business reason? If invoice formatting and dashboard formatting merely coincide today, merging them couples two teams and leads to the parameter-flag explosion. State the rule explicitly — DRY is about duplicated knowledge, not duplicated text — and offer the Rule of Three as the practical default.

### Q2: "What does 'prefer duplication over the wrong abstraction' actually mean?"
**Hint:** Sandi Metz. Duplication's cost is linear and reversible — grep, fix N copies. The wrong abstraction's cost compounds: each new caller adds a flag, each flag makes the code harder to unpick, and eventually a change made for one caller breaks another. Give the escape route too: when you find a wrong abstraction, **inline it back** into its callers, let them diverge, then re-extract from a position of knowledge.

### Q3: "Isn't YAGNI just an excuse for bad design? What if we need it later?"
**Hint:** Draw the distinction: skip the *speculative feature*, keep the *cheap seam*. Don't build a plugin registry for one plugin — but do put the SMTP call behind a `sendEmail()` function so a provider swap is a one-file change. YAGNI forbids building the future; it does not forbid making the future *possible*. Name the four costs of speculation: build, carry, repair, opportunity — and note that only the first is visible up front.

### Q4: "When is DRY actively harmful?"
**Hint:** When it couples modules that have different reasons to change — DRY violating SRP. Give the tells: boolean parameters named after callers (`forInvoice`), invalid flag combinations that throw, a `utils.js` half the codebase imports, and a shared function whose test file mentions three unrelated features. The fix is the un-merge, and you should be willing to name it as a legitimate refactor rather than a regression.

### Q5: "How do you decide when to abstract?"
**Hint:** Walk the four gates: (1) third occurrence, (2) do all copies change for the same reason, (3) can you name it in domain language — not "helper", not "manager", (4) does it stay flag-free. Pass all four → extract. Fail any → duplicate and wait. Add the deletion test: if the abstraction would be easier to delete than to understand, it isn't earning its keep.

---

## Practice exercise

### The Un-Merge Kata

You've inherited this function. It is the wrong abstraction, fully grown. Your job is to kill it correctly.

```javascript
// shared/notify.js — used by 3 unrelated features
export async function notify(user, type, data, options = {}) {
  const {
    urgent = false,
    skipIfOptedOut = true,
    includeUnsubscribeLink = true,
    retries = 3,
  } = options;

  if (type === 'PASSWORD_RESET' && includeUnsubscribeLink) {
    throw new Error('security emails cannot have unsubscribe links');
  }
  if (skipIfOptedOut && user.optedOutOfMarketing
      && type !== 'PASSWORD_RESET' && type !== 'ORDER_SHIPPED') {
    return;
  }

  const subject =
      type === 'PASSWORD_RESET' ? 'Reset your password'
    : type === 'ORDER_SHIPPED'  ? `Order ${data.orderId} shipped`
    : type === 'WEEKLY_DIGEST'  ? 'Your weekly digest'
    : 'Notification';

  const body = renderTemplate(type, data);
  const attempts = urgent ? retries + 2 : retries;

  for (let i = 0; i < attempts; i++) {
    try {
      await sendMail({ to: user.email, subject, body, unsubscribe: includeUnsubscribeLink });
      return;
    } catch (err) {
      if (i === attempts - 1) throw err;
      await new Promise((r) => setTimeout(r, 2 ** i * 100)); // exponential backoff
    }
  }
}
```

**What to produce (real JavaScript, ~20-40 minutes):**

1. **Identify the reasons to change.** List, one line each, the distinct business rules tangled inside this function, and name the *actor* who owns each. (Security owns password resets. Marketing owns the digest. Fulfilment owns shipping notices.) Mark every flag that exists purely to serve one caller.

2. **Un-merge it.** Write three separate functions — `sendPasswordResetEmail(user, token)`, `sendOrderShippedEmail(user, order)`, `sendWeeklyDigest(user, digest)` — each doing only what *its* actor needs. Password reset must ignore opt-out and must never carry an unsubscribe link. The digest must respect opt-out. Accept the duplication that results.

3. **Find the ONE thing that is genuinely shared.** There is exactly one piece of real, non-coincidental knowledge in the original: **retry with exponential backoff**. That isn't a business rule about notifications at all — it's infrastructure. Extract it as `withRetry(fn, { attempts })` and have all three callers use it. Then write one sentence explaining *why* this extraction is safe when the others weren't. (Hint: it changes for a reason none of the three actors own.)

4. **Justify with the four gates.** For every piece of duplication you deliberately kept, write one sentence naming which gate it failed.

**Success criteria:** no boolean parameter in any of your three functions is named after a caller, and adding a *fourth* notification type must require modifying **zero** existing functions.

---

## Quick reference cheat sheet

- **DRY = duplicated KNOWLEDGE, not duplicated TEXT.** "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."
- **The decisive test:** *do these copies change for the same business reason?* If no, they are not duplicates. Leave them alone.
- **Rule of Three:** duplicate once freely, twice with a note, extract on the third. Two examples can never reveal an abstraction's true shape.
- **AHA — Avoid Hasty Abstractions:** feel the *pain* of duplication before you remove it. Optimise for change, not for cleverness.
- **Sandi Metz:** "Duplication is far cheaper than the wrong abstraction." Duplication costs linearly; the wrong abstraction compounds.
- **The parameter-flag explosion** is the smell of a wrong abstraction: boolean flags named after *callers*, plus combinations that throw.
- **The un-merge is a legitimate refactor.** Inline the shared function back into its callers, let them diverge, re-extract later from knowledge.
- **KISS = the code a tired colleague understands at 2am.** Simple ≠ short. A clever one-liner taxes every reader, forever.
- **KISS ≠ no structure.** Minimise *total* complexity, not the file count. An 800-line handler is "simple" and unmaintainable.
- **YAGNI = don't build for imagined futures.** A plugin system with one plugin is a liability, not a foundation.
- **The four costs of speculation:** build, carry, repair, opportunity. Only the first one is visible when you decide.
- **YAGNI's line:** skip the *feature*, keep the *seam*. Don't build multi-provider support; do keep SMTP behind one function.
- **Over-applied DRY = coupling. Over-applied KISS = naivety. Over-applied YAGNI = a system that can't evolve.**
- **Arbitration rule:** optimise for the change you can see; keep cheap seams for the change you can't.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [18 — SOLID: Dependency Inversion](./18-solid-dependency-inversion.md) — the "seams" YAGNI lets you keep are exactly DIP's abstractions |
| **Next** | [20 — UML Class Diagrams](./20-uml-class-diagrams.md) — how to draw the structure you just learned not to over-engineer |
| **Related** | [14 — SOLID: Single Responsibility](./14-solid-single-responsibility.md) — "one reason to change" is the test that settles every DRY argument |
| **Related** | [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md) — the wrong abstraction is, precisely, accidental coupling |
