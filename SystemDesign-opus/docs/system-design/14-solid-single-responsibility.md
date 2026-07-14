# 14 — SOLID — Single Responsibility Principle (SRP)

## Category: LLD Fundamentals

---

## What is this?

The Single Responsibility Principle says: **a class should have one, and only one, reason to change.** Notice it does not say "a class should do one thing" — that is the popular misquote, and it is the reason most people apply SRP badly.

The real meaning is about **people**. A "reason to change" is a **person or group of people** who can walk up to your team and demand that the code behave differently. Uncle Bob (Robert C. Martin, who named SOLID) later restated it as: *"A module should be responsible to one, and only one, actor."*

Think of a restaurant menu that is also the receipt, also the health inspection certificate, and also the employee schedule — printed on one sheet of paper. The chef wants to change the dishes. Accounting wants to change the tax line. The health inspector wants to change the certificate wording. The manager wants to change the shifts. **Four different people, one sheet of paper.** Every time any one of them edits it, they risk destroying somebody else's section. That sheet of paper violates SRP.

---

## Why does it matter?

Because **shared code is a shared blast radius.**

When one class serves four different stakeholders, a change requested by one of them forces a redeploy, a re-review, and a re-test of code that the other three depend on. The classic failure looks like this:

> The finance team asks for a tweak to how overtime pay is rounded. An engineer edits `Employee.calculatePay()`. Two weeks later, the HR reporting dashboard is silently wrong — because `Employee.reportHours()` was quietly reusing the same private `regularHours()` helper that got "fixed" for finance. Nobody tested HR's report, because nobody knew HR's report depended on finance's math.

That is not a hypothetical. It is Uncle Bob's original motivating example, and it happens in real codebases constantly.

**What breaks without SRP:**
- **Merge conflicts.** Four teams editing the same 900-line file, every sprint.
- **Untestable code.** You cannot unit-test email validation if the same class also opens a Postgres connection and calls SendGrid.
- **Fear-driven development.** Nobody wants to touch `UserManager.js` because nobody knows everything it does.
- **Accidental coupling.** Two features share a private helper; changing it for one silently breaks the other.

**In interviews:** SRP is the very first thing an interviewer checks in an LLD round. If you draw one god class named `BookingManager` with 14 methods, you have already lost points, no matter how correct the logic is. Interviewers explicitly probe with: *"What happens when we switch from MySQL to Mongo? Which classes change?"* — that is an SRP question wearing a disguise.

**At real work:** SRP is the principle that makes code reviewable. Small, focused classes mean a pull request touches 2 files instead of 1 file that everybody owns.

---

## The core idea — explained simply

### The Swiss Army Knife vs. the Toolbox Analogy

A Swiss Army knife is a marvel. Knife, scissors, corkscrew, screwdriver, tweezers, toothpick — all in one object. It is genuinely useful when you are camping and can carry only one thing.

Now imagine you run a professional workshop. Would you hand your carpenter a Swiss Army knife?

- The **carpenter** wants a bigger, sharper blade.
- The **sommelier** wants a smoother corkscrew.
- The **electrician** wants a different screwdriver head.

To sharpen the blade for the carpenter, the manufacturer must **open up and re-machine the whole knife**. Every other tool in the body is disturbed. The sommelier's corkscrew now has a tiny wobble. And you cannot even test the blade change without re-assembling the entire knife.

A **toolbox** solves this. Separate blade, separate corkscrew, separate screwdriver — each in its own slot. Sharpen the blade; the corkscrew never moves. Test the blade alone. Replace the screwdriver with a better one without touching anything else.

**SRP says: build toolboxes, not Swiss Army knives.**

| Analogy piece | Technical meaning |
|---|---|
| The Swiss Army knife | A god class — `UserManager`, `OrderService`, `Utils` |
| Each tool on the knife | A responsibility (validation, persistence, email, formatting) |
| Carpenter / sommelier / electrician | The **actors** — Security, DBA, Marketing, Finance |
| "Re-machining the whole knife" | Editing and redeploying a class four teams depend on |
| Nicking the corkscrew while sharpening the blade | A regression in feature B caused by a change to feature A |
| The toolbox | A set of small collaborating classes |
| Picking up just the blade to test it | Unit-testing one class with no database, no network |

### The critical reframe: "one reason to change" = "one actor"

Here is the test that actually works. For a given class, ask:

> **"Who signs off on a change to this code? Whose job description does this behaviour belong to?"**

If you get more than one answer, you have more than one responsibility.

```
Class: User

  Who could force a change to it?
  ┌─────────────────────────────────────────────────────────────┐
  │ validate()        → the Product/Compliance team             │
  │                     ("passwords must now be 12 chars")      │
  │                                                             │
  │ save()            → the DBA / Platform team                 │
  │                     ("we're migrating Postgres → DynamoDB") │
  │                                                             │
  │ hashPassword()    → the Security team                       │
  │                     ("bcrypt is out, use argon2id")         │
  │                                                             │
  │ sendWelcomeEmail()→ the Growth/Marketing team               │
  │                     ("new onboarding copy + a CTA button")  │
  │                                                             │
  │ toCSVReport()     → the Finance/BI team                     │
  │                     ("add a signup_channel column")         │
  └─────────────────────────────────────────────────────────────┘

  FIVE actors. FIVE reasons to change. ONE class. → SRP violated.
```

Note something important: `hashPassword` and `validate` both "do one thing" in the naive sense. The naive reading of SRP ("a class does one thing") cannot tell you where to cut. The **actor** reading can.

---

## Key concepts inside this topic

### 1. The BAD example — a fat `User` class

Here is the class almost every codebase grows by month six. Read it and count the actors.

```javascript
// ❌ BAD — five actors, five reasons to change, one file.
import bcrypt from 'bcrypt';
import { pool } from './db.js';          // a Postgres connection pool
import sgMail from '@sendgrid/mail';

class User {
  constructor({ id, email, name, password, role = 'member' }) {
    this.id = id;
    this.email = email;
    this.name = name;
    this.password = password;
    this.role = role;
    this.createdAt = new Date();
  }

  // ── Actor 1: Product / Compliance ──────────────────────────
  validate() {
    const errors = [];
    if (!this.email || !this.email.includes('@')) errors.push('invalid email');
    if (!this.name || this.name.length < 2) errors.push('name too short');
    if (!this.password || this.password.length < 8) errors.push('password too short');
    if (errors.length) throw new Error(errors.join(', '));
    return true;
  }

  // ── Actor 2: Security ──────────────────────────────────────
  async hashPassword() {
    this.password = await bcrypt.hash(this.password, 10);
  }

  // ── Actor 3: DBA / Platform ────────────────────────────────
  async save() {
    const sql = `INSERT INTO users (email, name, password, role, created_at)
                 VALUES ($1, $2, $3, $4, $5) RETURNING id`;
    const res = await pool.query(sql, [
      this.email, this.name, this.password, this.role, this.createdAt,
    ]);
    this.id = res.rows[0].id;
    return this;
  }

  // ── Actor 4: Growth / Marketing ────────────────────────────
  async sendWelcomeEmail() {
    await sgMail.send({
      to: this.email,
      from: 'hello@acme.io',
      subject: `Welcome aboard, ${this.name}!`,
      html: `<h1>Hi ${this.name}</h1><p>Thanks for joining Acme.</p>`,
    });
  }

  // ── Actor 5: Finance / BI ──────────────────────────────────
  toCSVReport() {
    return `${this.id},${this.name},${this.email},${this.role},${this.createdAt.toISOString()}`;
  }
}
```

**What is actually wrong here?** It is not "the class is long." It is:

1. **Five teams share one file.** A Security change to `hashPassword` and a Marketing change to `sendWelcomeEmail` collide in the same pull request.
2. **`new User(...)` drags in the world.** Importing `User` imports `bcrypt`, a live Postgres pool, and the SendGrid SDK. Your test suite now needs a database.
3. **You cannot test validation in isolation.** Try writing "rejects an email with no @" as a test — you'll be booting Postgres to do it.
4. **You cannot reuse anything.** Need to hash an admin's password? It lives inside `User`. Need to CSV-export orders? The formatting logic is trapped in `User`.
5. **Swapping Postgres for DynamoDB means editing `User`** — a class that has nothing to do with storage conceptually.

### 2. The refactor — one actor per class

Cut along the **actor lines**, not arbitrary lines.

```javascript
// ✅ GOOD — user.js
// Actor: nobody external. Pure domain data + invariants the business owns.
export class User {
  constructor({ id = null, email, name, passwordHash, role = 'member', createdAt = new Date() }) {
    this.id = id;
    this.email = email;
    this.name = name;
    this.passwordHash = passwordHash; // NOTE: never a raw password. The entity holds a hash.
    this.role = role;
    this.createdAt = createdAt;
  }

  // Real domain behaviour belongs on the entity — this is business truth,
  // not infrastructure. An anemic entity would be a different mistake.
  isAdmin() {
    return this.role === 'admin';
  }

  promoteTo(role) {
    const allowed = ['member', 'moderator', 'admin'];
    if (!allowed.includes(role)) throw new Error(`Unknown role: ${role}`);
    this.role = role;
  }
}
```

```javascript
// ✅ GOOD — user-validator.js
// Actor: Product / Compliance. Zero I/O — pure functions in, errors out.
export class ValidationError extends Error {
  constructor(errors) {
    super(errors.join('; '));
    this.name = 'ValidationError';
    this.errors = errors;
  }
}

export class UserValidator {
  // Rules live here as data, so a compliance change is a one-line edit.
  static MIN_PASSWORD_LENGTH = 12;

  validate({ email, name, password }) {
    const errors = [];
    if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) errors.push('invalid email');
    if (!name || name.trim().length < 2) errors.push('name must be at least 2 characters');
    if (!password || password.length < UserValidator.MIN_PASSWORD_LENGTH) {
      errors.push(`password must be at least ${UserValidator.MIN_PASSWORD_LENGTH} characters`);
    }
    if (errors.length) throw new ValidationError(errors);
    return true;
  }
}
```

```javascript
// ✅ GOOD — password-hasher.js
// Actor: Security. When argon2id replaces bcrypt, exactly ONE file changes.
import bcrypt from 'bcrypt';

export class PasswordHasher {
  constructor(rounds = 12) {
    this.rounds = rounds;
  }

  async hash(plainText) {
    return bcrypt.hash(plainText, this.rounds);
  }

  async verify(plainText, hash) {
    return bcrypt.compare(plainText, hash);
  }
}
```

```javascript
// ✅ GOOD — user-repository.js
// Actor: DBA / Platform. All SQL is trapped inside this boundary.
export class UserRepository {
  constructor(db) {
    this.db = db; // injected — see topic 18, Dependency Inversion
  }

  async save(user) {
    const sql = `INSERT INTO users (email, name, password_hash, role, created_at)
                 VALUES ($1, $2, $3, $4, $5) RETURNING id`;
    const res = await this.db.query(sql, [
      user.email, user.name, user.passwordHash, user.role, user.createdAt,
    ]);
    user.id = res.rows[0].id;
    return user;
  }

  async findByEmail(email) {
    const res = await this.db.query('SELECT * FROM users WHERE email = $1', [email]);
    if (!res.rows.length) return null;
    const r = res.rows[0];
    // The repository is also the translator between DB rows and domain objects.
    return new User({
      id: r.id, email: r.email, name: r.name,
      passwordHash: r.password_hash, role: r.role, createdAt: r.created_at,
    });
  }
}
```

```javascript
// ✅ GOOD — email-service.js
// Actor: Growth / Marketing. Copy changes never touch domain code again.
export class EmailService {
  constructor(mailClient) {
    this.mail = mailClient;
  }

  async sendWelcome(user) {
    await this.mail.send({
      to: user.email,
      from: 'hello@acme.io',
      subject: `Welcome aboard, ${user.name}!`,
      html: `<h1>Hi ${user.name}</h1><p>Thanks for joining Acme.</p>`,
    });
  }
}
```

```javascript
// ✅ GOOD — user-report-formatter.js
// Actor: Finance / BI. Adding a CSV column is a 1-file, 0-risk change.
export class UserReportFormatter {
  toCSVRow(user) {
    return [user.id, user.name, user.email, user.role, user.createdAt.toISOString()].join(',');
  }

  header() {
    return 'id,name,email,role,created_at';
  }
}
```

### 3. Who orchestrates? The use-case / service class

Splitting classes does not mean the *workflow* disappears. Someone still has to run the steps in order. That someone is a thin **use-case class** whose single responsibility is *sequencing*.

```javascript
// ✅ GOOD — register-user.js
// Actor: the application itself. Its only job is ORDER OF OPERATIONS.
import { User } from './user.js';

export class RegisterUser {
  constructor({ validator, hasher, repository, emailService }) {
    this.validator = validator;
    this.hasher = hasher;
    this.repository = repository;
    this.emailService = emailService;
  }

  async execute({ email, name, password }) {
    this.validator.validate({ email, name, password });

    if (await this.repository.findByEmail(email)) {
      throw new Error('Email already registered');
    }

    const passwordHash = await this.hasher.hash(password);
    const user = await this.repository.save(new User({ email, name, passwordHash }));

    // Email failure must NOT fail the registration — a deliberate design choice.
    try {
      await this.emailService.sendWelcome(user);
    } catch (err) {
      console.error('welcome email failed', err);
    }

    return user;
  }
}
```

Read `execute()` out loud: *validate, check duplicate, hash, save, email.* That is a **story**, and stories are easy to review. Compare that to reading 200 lines of interleaved SQL, regex, and HTML.

### 4. Why SRP makes unit testing trivial — concretely

This is the payoff, and it is not abstract. Watch what testing looks like on each side.

**Before (fat `User`):** to test that a bad email is rejected, you must construct a `User`, which imports `bcrypt`, a Postgres pool, and SendGrid. Your "unit" test needs a database container, ~2–5 seconds of setup, and it fails in CI when Postgres is slow.

**After:**

```javascript
// user-validator.test.js — no DB, no network, no mocks. Runs in ~1 millisecond.
import { test } from 'node:test';
import assert from 'node:assert';
import { UserValidator, ValidationError } from './user-validator.js';

test('rejects an email with no @ sign', () => {
  const validator = new UserValidator();
  assert.throws(
    () => validator.validate({ email: 'nope', name: 'Ada', password: 'correct-horse-battery' }),
    ValidationError
  );
});

test('rejects a password under 12 characters', () => {
  const validator = new UserValidator();
  assert.throws(
    () => validator.validate({ email: 'a@b.io', name: 'Ada', password: 'short' }),
    /at least 12/
  );
});
```

And testing the *workflow* is now a matter of passing in fakes — because `RegisterUser` depends on collaborators it receives, not on modules it imports:

```javascript
// register-user.test.js — tests the ORDER OF OPERATIONS with zero infrastructure.
import { test } from 'node:test';
import assert from 'node:assert';
import { RegisterUser } from './register-user.js';

test('hashes the password before saving, and never stores plaintext', async () => {
  const saved = [];
  const sent = [];

  const registerUser = new RegisterUser({
    validator:    { validate: () => true },
    hasher:       { hash: async (p) => `hashed(${p})` },
    repository:   { findByEmail: async () => null, save: async (u) => { saved.push(u); return u; } },
    emailService: { sendWelcome: async (u) => sent.push(u.email) },
  });

  await registerUser.execute({ email: 'ada@acme.io', name: 'Ada', password: 'a-long-password' });

  assert.strictEqual(saved[0].passwordHash, 'hashed(a-long-password)');
  assert.strictEqual(saved[0].password, undefined);   // plaintext never reaches the entity
  assert.deepStrictEqual(sent, ['ada@acme.io']);
});
```

**The whole test suite runs in milliseconds with no Docker.** That is what SRP buys you. Speed of tests is not a vanity metric — a suite that runs in 2 seconds gets run on every save; a suite that takes 4 minutes gets run once a day, and bugs live longer.

### 5. The practical test: describe the class in one sentence, with no "and"

This is the fastest SRP smell test that exists. Say what the class does, out loud, in one sentence. **If you need the word "and", or a comma-list, or the word "manager"/"handler"/"util", you probably have more than one responsibility.**

| Sentence | Verdict |
|---|---|
| "`User` holds a user's identity and role." | Fine — one clause. |
| "`UserValidator` checks that user input satisfies the business rules." | Fine. |
| "`UserRepository` reads and writes users to the database." | Fine — "reads and writes" is one responsibility (persistence), one actor (the DBA). |
| "`User` validates itself **and** saves itself **and** hashes its password **and** emails people." | Four actors. Split it. |
| "`UserManager` manages users." | Says nothing. A `Manager` suffix is usually a confession, not a name. |

Beware the trap: **the word "and" is a hint, not a law.** "Reads and writes" is fine because both are the persistence actor. What matters is: *do these two things change for different reasons, requested by different people?*

### 6. The honest warning — you can absolutely over-apply this

SRP taken to its extreme produces codebases that are miserable in a *new* way. Be aware of the three classic overcorrections.

**a) Anemic domain models.** If you strip *all* behaviour out of `User` and leave a bag of public fields, you have not achieved SRP — you have achieved a struct with a bunch of procedural services operating on it. Martin Fowler calls this the **anemic domain model anti-pattern**. Real business rules like `promoteTo(role)` or `canAccess(resource)` **belong on the entity**. The rules of thumb:

> **Domain logic stays on the entity. Infrastructure (DB, network, crypto, formatting) leaves.**

`user.isAdmin()` belongs on `User`. `user.save()` does not.

**b) The 200-one-method-file codebase.** There is a real cost to a class per function. To follow one request, a new engineer opens `RegisterUser` → `UserValidator` → `ValidationRules` → `EmailRuleSet` → `RegexProvider`. Five files, five tabs, five levels of indirection, and the actual rule is a single regex buried at the bottom. This is **indirection tax**, and it is a genuine cost — not a stylistic quibble. It slows onboarding, slows debugging, and hides logic.

**c) Splitting on speculation.** Do not create `UserRepository`, `UserCacheRepository`, `UserRepositoryFactory`, and `IUserRepository` for an app with 30 users and one database you will never change. Splitting **that the business has not asked for** is YAGNI (see [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md)).

**The calibration:** SRP is about *actors*, and actors are a **real-world fact about your organisation**, not a code-shape preference. If validation and persistence genuinely never change independently in your world — a 300-line internal tool with one maintainer — a single class is fine. Split when you can name the second person who will request a change.

---

## Visual / Diagram description

### Diagram 1: BEFORE — one class, five actors, five reasons to change

```
        ┌────────────┐  ┌────────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐
        │  Product   │  │  Security  │  │    DBA    │  │ Marketing │  │  Finance │
        │ (Compliance)│  │            │  │(Platform) │  │  (Growth) │  │   (BI)   │
        └─────┬──────┘  └─────┬──────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘
              │               │               │              │             │
              │  "12-char     │  "argon2id    │ "move to     │ "new copy"  │ "add a
              │   passwords"  │   now"        │  DynamoDB"   │             │  column"
              │               │               │              │             │
              └───────────────┴───────┬───────┴──────────────┴─────────────┘
                                      │
                                      ▼   ALL FIVE EDIT THE SAME FILE
                        ┌───────────────────────────────┐
                        │            User               │
                        ├───────────────────────────────┤
                        │ - id, email, name, password   │
                        ├───────────────────────────────┤
                        │ + validate()          ◀─ Product   │
                        │ + hashPassword()      ◀─ Security  │
                        │ + save()              ◀─ DBA       │
                        │ + sendWelcomeEmail()  ◀─ Marketing │
                        │ + toCSVReport()       ◀─ Finance   │
                        └───────────────────────────────┘
                          imports: bcrypt, pg, @sendgrid/mail
                          ⚠ Cannot be instantiated without a live database.
```

Every arrow into that box is a merge conflict, a redeploy, and a full regression test of features nobody intended to touch.

### Diagram 2: AFTER — one actor per class

```
                            ┌──────────────────────────┐
                            │      RegisterUser        │   (use case: sequencing only)
                            ├──────────────────────────┤
                            │ + execute(dto)           │
                            └───┬───┬────┬────┬────┬────┘
                    ┌───────────┘   │    │    │    └───────────┐
                    │           ┌───┘    │    └────┐           │
                    ▼           ▼        ▼         ▼           ▼
        ┌───────────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
        │ UserValidator │ │Password  │ │UserRepository│ │ EmailService │
        │               │ │Hasher    │ │              │ │              │
        ├───────────────┤ ├──────────┤ ├──────────────┤ ├──────────────┤
        │ + validate()  │ │ + hash() │ │ + save()     │ │+sendWelcome()│
        │               │ │ + verify()│ │ + findByEmail│ │              │
        └───────┬───────┘ └────┬─────┘ └──────┬───────┘ └──────┬───────┘
                │              │              │                │
             Product        Security        DBA            Marketing
             owns this      owns this      owns this        owns this

                                  ┌──────────────┐   ┌──────────────────────┐
                                  │     User     │   │ UserReportFormatter  │
                                  │   (entity)   │   │                      │
                                  ├──────────────┤   ├──────────────────────┤
                                  │ + isAdmin()  │   │ + toCSVRow(user)     │
                                  │ + promoteTo()│   │ + header()           │
                                  └──────────────┘   └──────────┬───────────┘
                                   Business owns              Finance owns
                                   (pure domain rules)         this
```

**How to read this diagram:** `RegisterUser` **depends on** (arrows point to) four collaborators, each owned by exactly one actor. A Security change to `PasswordHasher` cannot possibly break Finance's CSV export — the two files never touch, never import each other, and never share a helper. Notice `User` kept `isAdmin()` and `promoteTo()`: those are business rules, not infrastructure, so they stay. That is the line between SRP and an anemic model.

---

## Real world examples

### 1. Express.js middleware

Express is SRP as an architecture. A request passes through a chain of tiny functions, each with exactly one responsibility and one owner:

```javascript
app.use(helmet());                  // Security team owns headers
app.use(express.json());            // Platform owns body parsing
app.use(morgan('combined'));        // SRE owns request logging
app.use(rateLimit({ max: 100 }));   // Platform owns abuse protection
app.use('/api/users', userRouter);  // The product team owns routes
```

The security team can swap `helmet` for a custom header middleware and the logging line is untouched. Compare with a monolithic `handleRequest()` that parses, logs, rate-limits, authenticates, and routes — every team would edit the same function.

### 2. The Repository pattern in real Node backends (Prisma, TypeORM, Mongoose)

Every serious Node ORM pushes you toward one class per persistence concern. In a typical layered service you will find `UserRepository`, `OrderRepository`, `InvoiceRepository` — all wrapping the DB client. This exists precisely so that a database migration (Postgres → Aurora → DynamoDB) touches only the repository layer. Teams that skipped this and put `prisma.user.findMany()` calls directly inside route handlers and React server components discover, during migration, that the DB call sites are scattered across 400 files.

### 3. Unix command-line tools

`grep` searches. `sort` sorts. `wc` counts. `cut` slices columns. None of them does two of those things. They compose through pipes:

```bash
cat access.log | grep " 500 " | cut -d' ' -f1 | sort | uniq -c | sort -rn | head
```

This is the Unix philosophy — "do one thing and do it well" — and it is the oldest, most battle-tested SRP argument in computing. It is also why you can swap `grep` for `ripgrep` in that pipeline without changing anything else.

---

## Trade-offs

| Aspect | Fat class (SRP violated) | Focused classes (SRP applied) |
|---|---|---|
| **Files to open to follow a flow** | 1 | 3–6 |
| **Blast radius of a change** | Whole class, all its consumers | One file, one actor |
| **Unit testability** | Poor — needs DB, network, mocks everywhere | Excellent — pure functions, injectable fakes |
| **Test suite speed** | Seconds to minutes (Docker, DB) | Milliseconds |
| **Merge conflicts** | Frequent, across teams | Rare |
| **Onboarding a new engineer** | "Just read this 900-line file" | More files, but each is small and named |
| **Swapping an implementation** | Surgery inside a shared class | Change one constructor argument |
| **Risk of over-engineering** | None | Real — anemic models, indirection tax |

| Symptom you observe | Likely cause | What to do |
|---|---|---|
| A test needs a database to check a regex | Validation is fused with persistence | Extract the validator |
| Two teams keep conflicting in one file | Two actors, one class | Split along the actor line |
| The class name contains `Manager`, `Util`, `Helper`, `Processor` | No clear responsibility | Rename it after what it actually does; if you can't, it does too much |
| You open 6 files to find one `if` statement | Over-applied SRP | Inline the trivial ones back |

**The sweet spot:** Split a class when you can **name a second real human or team** who would request a change to it independently. Do not split because a file crossed some arbitrary line count, and do not split for a stakeholder who does not exist yet. **SRP is a response to organisational reality, not a code-shape aesthetic.**

**Rule of thumb:** Domain logic stays on the entity. Infrastructure — the database, the network, cryptography, presentation formatting — moves out. If you get only that right, you have captured 80% of SRP's value.

---

## Common interview questions on this topic

### Q1: "What does 'one reason to change' actually mean?"
**Hint:** It means **one actor** — one person or team who can demand a change. Not "one method" and not "does one thing." Give the payroll example: `Employee.calculatePay()` (CFO), `Employee.reportHours()` (COO), `Employee.save()` (DBA). Three actors sharing a private helper method means a fix for the CFO silently breaks the COO's report. Mentioning "actor" instead of "does one thing" instantly signals you have read past the tweet-length version of SRP.

### Q2: "Isn't `UserRepository.save()` and `findByEmail()` two responsibilities? There's an 'and'."
**Hint:** No. Both serve the **same actor** — the DBA/platform team — and both change for the same reason (a database migration). The "no 'and' in the sentence" test is a heuristic for spotting mixed *actors*, not a ban on classes with multiple methods. A class with one method is not automatically SRP-compliant, and a class with six is not automatically a violation.

### Q3: "How do you avoid SRP turning into 200 tiny files nobody can navigate?"
**Hint:** Anchor every split in a real stakeholder. If you cannot name the human who would request that change independently, do not create the class — that is speculative generality (YAGNI). Also keep true domain behaviour on the entity so you don't drift into an anemic domain model. Be explicit in the interview that you *know* the cost of indirection; interviewers are wary of candidates who apply principles with no awareness of trade-offs.

### Q4: "Show me how SRP improves testability."
**Hint:** Walk through it concretely. In the fat class, testing "reject an email without an @" requires constructing a `User`, which imports `pg` and `@sendgrid/mail` — so a pure-logic test needs Docker. After extraction, `new UserValidator().validate(...)` is a pure function: no I/O, no mocks, runs in under a millisecond. Then show that `RegisterUser` is testable with plain object fakes because its collaborators are injected, not imported. Fast tests get run; slow tests get skipped.

### Q5: "Doesn't SRP conflict with cohesion? Aren't you scattering related code?"
**Hint:** The opposite — SRP *maximises* cohesion. Cohesion means everything in a class changes together for the same reason. A fat `User` has *low* cohesion (five unrelated concerns coexisting). `UserValidator` has very high cohesion. The thing SRP increases is **the number of classes**, and therefore coupling *between* files — which is why you inject dependencies (see [18 — Dependency Inversion](./18-solid-dependency-inversion.md)) to keep that coupling loose. See [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md).

---

## Practice exercise

### Break up the `Order` God Class

You inherit this class. Your job: refactor it, and **justify every cut by naming the actor.**

```javascript
// ❌ Refactor me.
class Order {
  constructor(items, customer, couponCode) {
    this.items = items;         // [{ sku, name, price, qty }]
    this.customer = customer;   // { id, email, name, tier }
    this.couponCode = couponCode;
    this.status = 'PENDING';
  }

  calculateSubtotal() { /* sums item.price * item.qty */ }
  applyDiscount() { /* SUMMER20 = 20% off; GOLD tier = extra 5% */ }
  calculateTax() { /* 18% GST if India, 0% otherwise */ }
  async chargeCard(cardToken) { /* calls the Stripe SDK */ }
  async saveToDatabase() { /* raw SQL INSERT into orders + order_items */ }
  async sendConfirmationEmail() { /* calls SendGrid with an HTML template */ }
  generateInvoicePDF() { /* builds a PDF buffer */ }
  toJSON() { /* shapes the object for the REST API response */ }
}
```

**What to produce (aim for ~30 minutes):**

1. **The actor table.** A markdown table with two columns: `Method` and `Which team/person would request a change to it?`. Be specific — "Finance", "Tax/Compliance", "Payments", "DBA", "Marketing", "API consumers".
2. **The refactored class list.** Name each new class and write its one-sentence description **with no "and"** in it. Decide honestly which behaviour stays on `Order` itself — `calculateSubtotal()` is arguably domain logic that belongs on the entity. Defend your choice.
3. **A `PlaceOrder` use-case class** in JavaScript that wires the collaborators together and runs them in the right order (validate → price → tax → charge → save → email). Constructor-inject every dependency.
4. **One unit test, using `node:test`, that verifies the discount math — with zero database, zero network, and zero mocking libraries.** If your test needs Stripe, your refactor is not done.
5. **The honesty check.** Name one class you *chose not to create* because no real actor justified it yet.

---

## Quick reference cheat sheet

- **SRP** = "A class should have one, and only one, reason to change." Uncle Bob's own restatement: **"responsible to one, and only one, actor."**
- **"Reason to change" = a PERSON** — a team or stakeholder who can demand different behaviour. Not "does one thing."
- **The one-sentence test:** describe the class without using "and". If you can't, it likely serves two actors.
- **`Manager`, `Util`, `Helper`, `Processor`, `Handler`** in a class name is usually a confession that you couldn't name its responsibility.
- **The killer symptom:** a unit test that needs a database to check a regex.
- **Extract infrastructure, keep domain logic.** `user.isAdmin()` stays on `User`; `user.save()` does not.
- **Repository = persistence actor.** Multiple methods (`save`, `findById`, `delete`) are still ONE responsibility.
- **A use-case class** (`RegisterUser`, `PlaceOrder`) has a legitimate single responsibility: **sequencing** the collaborators.
- **SRP's biggest payoff is testing** — pure classes need no Docker, so the suite runs in milliseconds and actually gets run.
- **Over-application is real:** anemic domain models, 200 one-method files, and indirection tax are genuine costs.
- **Split on evidence, not speculation.** Name the second stakeholder before you create the second class (see YAGNI).
- **SRP raises cohesion, and raises the number of dependencies** — which is exactly why the next principles (OCP, DIP) exist.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [13 — OOP: The Four Pillars](./13-oop-four-pillars.md) — encapsulation and abstraction are the tools SRP uses to cut classes apart |
| **Next** | [15 — SOLID: Open/Closed Principle](./15-solid-open-closed.md) — once each class has one job, how do you extend it without editing it? |
| **Related** | [24 — Coupling and Cohesion](./24-coupling-and-cohesion.md) — SRP is the practical recipe for high cohesion |
| **Related** | [18 — SOLID: Dependency Inversion](./18-solid-dependency-inversion.md) — the injection technique that makes the split classes wire together cleanly |
| **Related** | [19 — DRY, KISS, YAGNI](./19-dry-kiss-yagni.md) — the guardrail that stops SRP from becoming 200 one-method files |
