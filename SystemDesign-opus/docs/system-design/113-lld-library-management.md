# 113 — Design a Library Management System
## Category: LLD Case Study

---

## What is this?

A **Library Management System (LMS)** is the software that runs a library: it stores the catalog of titles, tracks every physical book on the shelves, lets members search and borrow books, handles returns, charges fines for late returns, and lets people reserve a book that is currently checked out.

Think of the self-service kiosk at your local public library. You scan your membership card, scan the barcode on the back of a book, and the screen says "Due back on 1st August." Behind that simple interaction sits a small object model — members, books, copies, loans, reservations, fines — that we are going to design from scratch.

This is a **classic low-level-design (LLD) interview problem**. It looks easy, but it hides one modelling trap that separates strong candidates from weak ones (we will hit it head-on in section 2).

---

## Why does it matter?

**Interview angle.** Library Management is one of the five or six "canonical" object-modelling questions (alongside Parking Lot, Elevator, and Hotel Booking). Interviewers love it because it tests whether you can turn fuzzy English requirements into clean classes with the right relationships. The single most common failure is conflating a *book title* with a *physical book* — if you model both as one `Book` class you cannot answer "which copy did John borrow?" or "how many copies are on loan?". Getting that distinction right, out loud, signals seniority.

**Real-work angle.** Every inventory system on earth is a library system in disguise. Amazon warehouses, car-rental fleets, hospital equipment tracking, hotel rooms — all have an *abstract thing* (a product, a car model, a room type) and *many concrete instances* (the exact unit with a serial number). If you can model a library, you can model any of them. You will also practise avoiding the **god-class trap** (one giant `Library` class that does everything) and using the **Observer pattern** for notifications — both show up constantly in production code.

---

## The core idea — explained simply

### Analogy: a restaurant menu vs. the plates in the kitchen

Imagine a restaurant. The **menu** lists "Margherita Pizza" once. But in the kitchen there might be *five* Margherita pizzas being cooked right now for five different tables. When a waiter says "table 3 got the Margherita," they don't mean the menu entry — they mean one specific plate.

A library works exactly the same way:

| Restaurant | Library | What it represents |
|---|---|---|
| Menu item "Margherita Pizza" | `Book` (title "Clean Code") | The **abstract title** — one entry, shared metadata (ISBN, author) |
| A specific plate of Margherita | `BookItem` / `BookCopy` (barcode `A-0007`) | A **physical object** you can actually hold, borrow, or lose |
| "We're out of Margherita" | All copies LOANED | The title exists but no physical copy is available |
| The waiter carrying the plate | `BookLending` / `Loan` | The record of *who* has *which physical copy* and *until when* |
| A guest asking to be told when a table frees up | `Reservation` | A promise to notify someone when a copy comes back |

The menu never leaves the restaurant; the plates do. In the same way, a `Book` is never borrowed — a `BookItem` is. Hold onto that sentence; it is the whole lesson.

---

## Key concepts inside this topic

We will follow the **7-step LLD method** from [111 — LLD Approach Framework](./111-lld-approach-framework.md): (1) clarify requirements, (2) find the nouns, (3) find the verbs and assign ownership, (4) draw the class diagram, (5) implement, (6) name the patterns, (7) handle extensions.

### 1. Requirements & clarifying questions

Before writing a single class, pin down what the system must do. In an interview you *say these out loud* and *ask the clarifying questions* — never assume.

**Functional requirements**
- Members can **search** the catalog by title, author, or ISBN.
- Members can **borrow (checkout)** a physical copy of a book; it gets a **due date**.
- Members can **return** a borrowed copy.
- If every copy of a title is checked out, a member can **reserve** it and be **notified** when one is returned.
- Members can **renew** a book (extend the due date) if no one else has reserved it.
- The system charges a **fine** for returns after the due date.
- Each member has a **borrowing limit** (max books held at once).
- **Librarians** manage the catalog: add/remove books and copies, register members, collect fines.

**Non-functional (briefly).** Fast search (we will index it), correct concurrency for checkout (two people cannot grab the same copy — out of scope for the object model but note it), and auditability of every loan.

**Clarifying questions to ask the interviewer:**
1. Can a title have **multiple physical copies**? (Yes — this drives the whole model.)
2. What is the **max number of books** a member may hold at once? (Assume 5.)
3. What is the **fine rate**? Flat or per-day? (Assume $1 per day late.)
4. How does the **reservation queue** work — FIFO? Does it expire? (Assume FIFO; first in line gets notified first.)
5. Do we handle **e-books / audiobooks**, or only physical? (Physical first; we design so e-books can be added — see section 7.)
6. Are fines **blocking** (can't borrow with unpaid fines)? (Assume not for v1, but flag it.)

Writing these down turns a vague prompt into a concrete spec. That is half the interview.

### 2. Identify the core objects (nouns) — *the key lesson*

Underline the nouns in the requirements: *member, book, copy, catalog, loan, reservation, fine, librarian, due date, barcode, ISBN*. Group them into classes.

**THE modelling insight — `Book` vs. `BookItem`:**

> A **`Book`** is the *abstract title*. It owns the metadata that is identical across every physical copy: **ISBN, title, authors, subject, publisher**. There is exactly **one** `Book` object for "Clean Code."
>
> A **`BookItem`** (also called `BookCopy`) is a *single physical copy* sitting on a shelf. It owns per-copy data: a unique **barcode**, its current **status** (available / loaned / reserved / lost), its **due date** when on loan, and a **reference back to its `Book`**. There are **many** `BookItem`s for "Clean Code."

Many candidates model a single `Book` with a `quantity` integer. That collapses the instant you ask "which specific copy is overdue?" or "copy `A-0007` is water-damaged, mark *it* lost." You need per-copy identity. **`Book` 1—to—many `BookItem` is the backbone of the design.**

The rest of the nouns:

| Class | What it is | Key fields |
|---|---|---|
| `Book` | Abstract title | isbn, title, authors[] |
| `BookItem` | Physical copy | barcode, status, dueDate, book(ref) |
| `Account` (base) | A person with a login | id, name, cardId |
| `Member` | Borrower (extends Account) | loans[], borrowLimit |
| `Librarian` | Staff (extends Account) | — (has management methods) |
| `Catalog` | Searchable index of all books | title/author/isbn indexes |
| `BookLending` / `Loan` | One borrow event | bookItem, member, issueDate, dueDate, returnDate |
| `Reservation` | A hold on an unavailable title | member, book, status, createdAt |
| `Fine` | Money owed for a late return | amount, loan, paid |

`Member` and `Librarian` share identity fields, so we extract an `Account` base class — a small **inheritance** decision that keeps common data in one place (good cohesion; recall [24 — Coupling & Cohesion](./24-coupling-and-cohesion.md)).

### 3. Behaviours (verbs) and ownership

Now the verbs: *search, checkout, return, reserve, renew, calculate fine, add copy, notify*. The art is assigning each verb to the object that **owns the data it needs** — this is what stops one class from swelling into a god-class.

| Behaviour | Owner | Why that owner |
|---|---|---|
| `search(byTitle/byAuthor/byIsbn)` | `Catalog` | The catalog holds the indexes; searching is its job |
| `updateStatus()` / `isAvailable()` | `BookItem` | A copy knows its own status |
| `isOverdue()`, `calculateFine(rate)` | `BookLending` (Loan) | The loan knows the due date and return date — it can compute lateness itself |
| holding a member's current loans | `Member` | A member owns their borrowing state and enforces their own limit |
| `checkout()`, `returnItem()`, `reserve()` | `Library` (orchestrator) | These span *multiple* objects (member + item + catalog + reservations), so they belong to a coordinator |
| notify next in queue | `Reservation` queue via **Observer** | Decouples "a book came back" from "tell the waiting member" |

Notice the split: **object-local logic lives on the object** (a `Loan` computes its own fine; a `BookItem` reports its own status), while **cross-object workflows live in the `Library` orchestrator**. The orchestrator coordinates but does not hoard data — that is how you dodge the god-class trap. If `Library` were computing fines *and* storing statuses *and* indexing searches, it would be a 1000-line monster. Instead it delegates.

### 4. Class diagram

```
                         ┌────────────────────┐
                         │      Account       │   (base class)
                         │ id, name, cardId   │
                         └─────────┬──────────┘
                        ▲ extends  │  ▲ extends
              ┌─────────┴───┐   ┌──┴──────────────┐
              │   Member    │   │   Librarian     │
              │ loans[]     │   │ (manages catalog│
              │ borrowLimit │   │  & members)     │
              └──────┬──────┘   └─────────────────┘
                     │ 1
                     │ holds
                     │ *
              ┌──────▼───────────────┐        ┌──────────────────┐
              │   BookLending (Loan) │───────▶│    BookItem      │
              │ issueDate, dueDate,  │  refs  │ barcode, status, │
              │ returnDate           │   1    │ dueDate          │
              │ isOverdue()          │        │ book (ref)  *    │
              │ calculateFine(rate)  │        └────────┬─────────┘
              └──────────────────────┘                 │ *
                                                        │ copies of
                                                        │ 1
                                              ┌─────────▼─────────┐
   ┌──────────────────┐                       │       Book        │
   │     Catalog      │ indexes  *            │ isbn, title,      │
   │ byTitle:  Map    │◀──────────────────────│ authors[]         │
   │ byAuthor: Map    │                       └───────────────────┘
   │ byIsbn:   Map    │                                 ▲ 1
   │ search()         │                                 │ on
   └──────────────────┘                                 │ *
                                              ┌──────────┴────────┐
   ┌──────────────────┐  observes            │   Reservation     │
   │  Library         │─── FIFO queue ──────▶│ member, book,     │
   │ (orchestrator)   │  per Book            │ status, createdAt │
   │ checkout()       │                       └───────────────────┘
   │ returnItem()     │
   │ reserve()        │
   └──────────────────┘
```

**Reading the associations:**
- `Account` is the base; `Member` and `Librarian` **inherit** from it (`←` extends).
- `Book` **1—to—many** `BookItem` (one title, many physical copies).
- `Member` **1—to—many** `BookLending` (a member holds several active loans).
- Each `BookLending` **references exactly one** `BookItem` (the copy borrowed).
- `Catalog` holds **many** `Book`s in three index Maps.
- A `Book` has a **FIFO queue of** `Reservation`s; the `Library` orchestrator drives checkout/return and fires notifications.

### 5. Full JavaScript implementation

Complete and runnable. Save as `library.js` and run `node library.js`.

```js
'use strict';

// ─────────────────────────────────────────────────────────────
// Enums — Object.freeze makes them true immutable constants.
// (JS has no native enum; this is the idiomatic stand-in.)
// ─────────────────────────────────────────────────────────────
const BookStatus = Object.freeze({
  AVAILABLE: 'AVAILABLE',
  RESERVED:  'RESERVED',
  LOANED:    'LOANED',
  LOST:      'LOST',
});

const ReservationStatus = Object.freeze({
  WAITING:   'WAITING',   // in the queue, book not yet free
  COMPLETED: 'COMPLETED', // member was notified / picked it up
  CANCELED:  'CANCELED',
});

const AccountType = Object.freeze({
  MEMBER:    'MEMBER',
  LIBRARIAN: 'LIBRARIAN',
});

// ─────────────────────────────────────────────────────────────
// Book  — the ABSTRACT title. Shared metadata only.
// There is ONE Book per ISBN.
// ─────────────────────────────────────────────────────────────
class Book {
  constructor(isbn, title, authors = []) {
    this.isbn = isbn;
    this.title = title;
    this.authors = authors;          // array of strings
    this.items = [];                 // its physical copies (BookItem[])
    this.reservationQueue = [];      // FIFO queue of Reservation objects
  }

  addItem(bookItem) {
    bookItem.book = this;
    this.items.push(bookItem);
  }

  // How many physical copies are free to borrow right now.
  availableCount() {
    return this.items.filter(i => i.status === BookStatus.AVAILABLE).length;
  }
}

// ─────────────────────────────────────────────────────────────
// BookItem — a single PHYSICAL copy. Owns its own status.
// This is the thing that is actually borrowed. Many per Book.
// ─────────────────────────────────────────────────────────────
class BookItem {
  constructor(barcode) {
    this.barcode = barcode;
    this.status = BookStatus.AVAILABLE;
    this.dueDate = null;             // set when loaned
    this.book = null;                // back-reference, set by Book.addItem
  }

  isAvailable() {
    return this.status === BookStatus.AVAILABLE;
  }
}

// ─────────────────────────────────────────────────────────────
// Account (base) + Member + Librarian
// Common identity lives in the base class (good cohesion).
// ─────────────────────────────────────────────────────────────
class Account {
  constructor(id, name, type) {
    this.id = id;
    this.name = name;
    this.type = type;
  }
}

class Member extends Account {
  constructor(id, name, borrowLimit = 5) {
    super(id, name, AccountType.MEMBER);
    this.borrowLimit = borrowLimit;
    this.loans = [];                 // active BookLending objects
  }

  // A Member enforces its OWN borrow limit — the logic that
  // depends on member state lives on the member.
  atLimit() {
    return this.loans.length >= this.borrowLimit;
  }

  addLoan(loan)    { this.loans.push(loan); }
  removeLoan(loan) { this.loans = this.loans.filter(l => l !== loan); }

  // Observer callback: how a Member reacts to a notification.
  // (The notification "channel" — here just console — is swappable; see §7.)
  notify(message) {
    console.log(`   [NOTIFY -> ${this.name}] ${message}`);
  }
}

class Librarian extends Account {
  constructor(id, name) {
    super(id, name, AccountType.LIBRARIAN);
  }

  addBookItem(catalog, book, barcode) {
    const item = new BookItem(barcode);
    book.addItem(item);
    catalog.addBook(book);           // idempotent: indexes the title
    return item;
  }
}

// ─────────────────────────────────────────────────────────────
// Catalog — searchable index of all Books.
// Uses Maps as indexes, one per search key. This mirrors how a
// real database builds an index per queried column: O(1)/O(log n)
// lookups instead of scanning every row.
// ─────────────────────────────────────────────────────────────
class Catalog {
  constructor() {
    this.byIsbn = new Map();         // isbn        -> Book
    this.byTitle = new Map();        // titleLower  -> Book[]
    this.byAuthor = new Map();       // authorLower -> Book[]
  }

  addBook(book) {
    if (this.byIsbn.has(book.isbn)) return; // already indexed
    this.byIsbn.set(book.isbn, book);

    const t = book.title.toLowerCase();
    if (!this.byTitle.has(t)) this.byTitle.set(t, []);
    this.byTitle.get(t).push(book);

    for (const author of book.authors) {
      const a = author.toLowerCase();
      if (!this.byAuthor.has(a)) this.byAuthor.set(a, []);
      this.byAuthor.get(a).push(book);
    }
  }

  searchByIsbn(isbn)     { return this.byIsbn.get(isbn) || null; }
  searchByTitle(title)   { return this.byTitle.get(title.toLowerCase()) || []; }
  searchByAuthor(author) { return this.byAuthor.get(author.toLowerCase()) || []; }
}

// ─────────────────────────────────────────────────────────────
// BookLending (Loan) — one borrow event.
// It OWNS the due-date logic and computes its own lateness/fine.
// ─────────────────────────────────────────────────────────────
const DAY_MS = 24 * 60 * 60 * 1000;
const LOAN_PERIOD_DAYS = 14;

class BookLending {
  constructor(bookItem, member, issueDate = new Date()) {
    this.bookItem = bookItem;
    this.member = member;
    this.issueDate = issueDate;
    this.dueDate = new Date(issueDate.getTime() + LOAN_PERIOD_DAYS * DAY_MS);
    this.returnDate = null;
    bookItem.dueDate = this.dueDate;
  }

  // A loan knows the due date, so it decides if it is overdue.
  isOverdue(asOf = new Date()) {
    const end = this.returnDate || asOf;
    return end.getTime() > this.dueDate.getTime();
  }

  daysLate(asOf = new Date()) {
    const end = this.returnDate || asOf;
    const late = Math.ceil((end.getTime() - this.dueDate.getTime()) / DAY_MS);
    return Math.max(0, late);
  }

  calculateFine(ratePerDay = 1) {
    return this.daysLate() * ratePerDay;
  }
}

// ─────────────────────────────────────────────────────────────
// Reservation — a hold placed when every copy is checked out.
// ─────────────────────────────────────────────────────────────
class Reservation {
  constructor(member, book) {
    this.member = member;
    this.book = book;
    this.status = ReservationStatus.WAITING;
    this.createdAt = new Date();
  }
}

// ─────────────────────────────────────────────────────────────
// OBSERVER PATTERN — ReservationNotifier.
//
// WHY Observer here? When a reserved copy is returned, the book
// (the "subject") must tell whoever is first in the reservation
// queue (the "observers") WITHOUT knowing or caring who they are
// or how they want to be reached. The book just publishes an
// event; observers react. This decouples "a copy came back" from
// "notify the next member" — the natural fit for this problem
// (recall 41 — Observer Pattern).
// ─────────────────────────────────────────────────────────────
class ReservationNotifier {
  // Given a book that just had a copy freed, notify the next
  // WAITING member in FIFO order and complete their reservation.
  static onCopyReturned(book) {
    const next = book.reservationQueue.find(
      r => r.status === ReservationStatus.WAITING
    );
    if (!next) return null;

    next.status = ReservationStatus.COMPLETED;
    // The observer (Member) reacts to the published event:
    next.member.notify(
      `The book you reserved — "${book.title}" — is now available. ` +
      `Please collect it.`
    );
    return next;
  }
}

// ─────────────────────────────────────────────────────────────
// Library — the ORCHESTRATOR.
// It coordinates workflows that span many objects, but delegates
// all object-local logic. It does NOT store statuses or compute
// fines itself — that would be the god-class trap.
// ─────────────────────────────────────────────────────────────
class Library {
  constructor(name, fineRatePerDay = 1) {
    this.name = name;
    this.catalog = new Catalog();
    this.fineRatePerDay = fineRatePerDay;
    this.fines = [];                 // collected Fine records
  }

  // Checkout: give a member a specific physical copy.
  checkout(member, bookItem) {
    if (member.atLimit()) {
      throw new Error(
        `${member.name} is at the borrow limit (${member.borrowLimit}).`
      );
    }
    if (!bookItem.isAvailable()) {
      throw new Error(
        `Copy ${bookItem.barcode} is not available (status ${bookItem.status}).`
      );
    }
    bookItem.status = BookStatus.LOANED;
    const loan = new BookLending(bookItem, member);
    member.addLoan(loan);
    console.log(
      `-> ${member.name} borrowed "${bookItem.book.title}" ` +
      `(copy ${bookItem.barcode}), due ${loan.dueDate.toDateString()}.`
    );
    return loan;
  }

  // Return: take a copy back, compute any fine, then fire the
  // Observer notification if someone was waiting for this title.
  returnItem(bookItem) {
    const member = bookItem.book.items && this._findHolder(bookItem);
    const loan = member ? member.loans.find(l => l.bookItem === bookItem) : null;
    if (!loan) throw new Error(`No active loan for copy ${bookItem.barcode}.`);

    loan.returnDate = new Date();
    const fine = loan.calculateFine(this.fineRatePerDay);
    if (fine > 0) {
      this.fines.push({ member: loan.member, amount: fine, loan });
      console.log(
        `-> ${loan.member.name} returned "${bookItem.book.title}" LATE. ` +
        `Fine: $${fine}.`
      );
    } else {
      console.log(
        `-> ${loan.member.name} returned "${bookItem.book.title}" on time.`
      );
    }
    loan.member.removeLoan(loan);
    bookItem.dueDate = null;

    // Is anyone waiting for this title? If so, hold this copy for
    // them and notify via the Observer. Otherwise mark available.
    const book = bookItem.book;
    const hasWaiter = book.reservationQueue.some(
      r => r.status === ReservationStatus.WAITING
    );
    if (hasWaiter) {
      bookItem.status = BookStatus.RESERVED;
      ReservationNotifier.onCopyReturned(book);   // <-- Observer fires
    } else {
      bookItem.status = BookStatus.AVAILABLE;
    }
    return fine;
  }

  // Reserve a title when no copy is free. Adds to the FIFO queue.
  reserve(member, book) {
    if (book.availableCount() > 0) {
      throw new Error(
        `"${book.title}" has ${book.availableCount()} copy(ies) available — ` +
        `no need to reserve; just check one out.`
      );
    }
    const res = new Reservation(member, book);
    book.reservationQueue.push(res);
    console.log(
      `-> ${member.name} reserved "${book.title}" ` +
      `(position ${book.reservationQueue.length} in queue).`
    );
    return res;
  }

  // Helper: find which member currently holds a given copy.
  _findHolder(bookItem) {
    // In a real system loans live in a DB; here we scan known members.
    return this._members
      ? this._members.find(m => m.loans.some(l => l.bookItem === bookItem))
      : null;
  }

  registerMembers(members) { this._members = members; }
}

// ─────────────────────────────────────────────────────────────
// main() — proves the whole thing runs end to end.
// ─────────────────────────────────────────────────────────────
function main() {
  console.log('=== Library Management System demo ===\n');

  const library = new Library('City Central Library', 1); // $1/day fine
  const librarian = new Librarian('L1', 'Ms. Rao');

  // 1. Librarian adds a title with TWO physical copies.
  const cleanCode = new Book('978-0132350884', 'Clean Code', ['Robert C. Martin']);
  const copyA = librarian.addBookItem(library.catalog, cleanCode, 'CC-0001');
  const copyB = librarian.addBookItem(library.catalog, cleanCode, 'CC-0002');

  // Another title, one copy.
  const dpBook = new Book('978-0201633610', 'Design Patterns',
    ['Erich Gamma', 'Richard Helm']);
  librarian.addBookItem(library.catalog, dpBook, 'DP-0001');

  console.log(`Catalog seeded. "Clean Code" has ${cleanCode.items.length} copies.\n`);

  // 2. Register members. Alice has a tiny limit of 1 to demo the cap.
  const alice = new Member('M1', 'Alice', 1);   // borrowLimit = 1
  const bob   = new Member('M2', 'Bob', 5);
  library.registerMembers([alice, bob]);

  // 3. Search the catalog (index lookups).
  console.log('Search by author "Robert C. Martin":',
    library.catalog.searchByAuthor('Robert C. Martin').map(b => b.title));
  console.log('Search by ISBN 978-0201633610:',
    library.catalog.searchByIsbn('978-0201633610').title, '\n');

  // 4. Alice borrows both copies of Clean Code? No — she is at limit
  //    after ONE book. Prove the borrow limit is enforced.
  library.checkout(alice, copyA);
  try {
    library.checkout(alice, copyB);
  } catch (e) {
    console.log(`   (expected) ${e.message}`);
  }
  console.log('');

  // 5. Bob borrows the second copy. Now BOTH copies are loaned.
  library.checkout(bob, copyB);
  console.log(`   "Clean Code" available copies now: ${cleanCode.availableCount()}\n`);

  // 6. A third member reserves Clean Code — every copy is out.
  const carol = new Member('M3', 'Carol', 5);
  library.registerMembers([alice, bob, carol]);
  library.reserve(carol, cleanCode);
  console.log('');

  // 7. Alice returns her copy. Because Carol is waiting, the
  //    Observer fires and notifies Carol automatically.
  library.returnItem(copyA);
  console.log('');

  // 8. Show a LATE return + fine. Back-date Bob's loan to be overdue.
  const bobLoan = bob.loans.find(l => l.bookItem === copyB);
  bobLoan.dueDate = new Date(Date.now() - 3 * DAY_MS); // 3 days ago
  library.returnItem(copyB);

  console.log('\n=== demo complete ===');
}

main();

module.exports = {
  BookStatus, ReservationStatus, AccountType,
  Book, BookItem, Account, Member, Librarian,
  Catalog, BookLending, Reservation, ReservationNotifier, Library,
};
```

**Expected output (abridged):**

```
=== Library Management System demo ===
Catalog seeded. "Clean Code" has 2 copies.
Search by author "Robert C. Martin": [ 'Clean Code' ]
-> Alice borrowed "Clean Code" (copy CC-0001), due ...
   (expected) Alice is at the borrow limit (1).
-> Bob borrowed "Clean Code" (copy CC-0002), due ...
   "Clean Code" available copies now: 0
-> Carol reserved "Clean Code" (position 1 in queue).
-> Alice returned "Clean Code" on time.
   [NOTIFY -> Carol] The book you reserved — "Clean Code" — is now available. ...
-> Bob returned "Clean Code" LATE. Fine: $3.
=== demo complete ===
```

The borrow limit blocked Alice's second checkout, the reservation notification fired the moment a copy came back, and the late return produced a $3 fine — the design runs.

### 6. Design patterns used and WHY

| Pattern | Where | Why it fits |
|---|---|---|
| **Observer** ([41](./41-pattern-observer.md)) | `ReservationNotifier.onCopyReturned` | The returned book (subject) publishes a "copy freed" event; the waiting member (observer) reacts. The book does not know *who* is waiting or *how* they're reached — pure decoupling. This is the natural, primary pattern for the reservation feature. |
| **Strategy** (optional) | fine calculation | `calculateFine(ratePerDay)` could accept a pluggable `FineStrategy` (flat vs. per-day vs. tiered) so pricing rules change without touching `BookLending`. |
| **Factory** (optional) | creating `BookItem`s / item subtypes | When we add e-books and audiobooks (§7), a `BookItemFactory.create(type, …)` centralizes which subclass to instantiate. |

**Coupling & cohesion** (recall [24 — Coupling & Cohesion](./24-coupling-and-cohesion.md)): the design keeps three concerns cleanly separated — the **`Catalog`** does searching, the **`BookLending`** does fine/lateness math, and the **`ReservationNotifier`** does notification. Each class has high cohesion (one job) and low coupling (talks through small interfaces). The `Library` orchestrator wires them together but owns none of their internal logic, which is exactly why it never becomes a god class.

### 7. Extensions the interviewer asks for

**"Add e-books and audiobooks."** Introduce subtypes of `BookItem`: `PhysicalBook`, `EBook`, `AudioBook`. A `Book` (the title) stays the same; only the *item* type changes. E-books have no scarcity, so `availableCount()` and reservations simply don't apply to them. A **Factory** decides which subclass to build. The `Book`-vs-`BookItem` split we insisted on in §2 is what makes this drop-in — the title layer is untouched.

**"Add a notification channel — email or SMS."** This is where Observer pays off. Give `Member` an `addNotifier(channel)` where each channel implements `send(message)`. The `notify()` method loops over the member's channels. Adding SMS means writing one `SmsNotifier` class — no change to `Library` or `ReservationNotifier`. That is the whole point of the pattern.

**"Add membership tiers with different limits."** `borrowLimit` is already a field. Add a `MembershipTier` (STUDENT: 3, ADULT: 5, PREMIUM: 10) and set `borrowLimit` from the tier. `atLimit()` needs no change because it reads the field, not a constant.

**"Support multiple branches."** Add a `Branch` class; each `BookItem` gains a `branch` reference (a copy physically lives at one branch). `Catalog` can stay global (search all branches) while checkout/return check the branch. Reservations can be per-branch or allow inter-branch transfer. The core `Book`/`BookItem`/`Loan` model is unchanged — you are only adding a location dimension to the *copy*, which is exactly where physical location belongs.

Each extension slots in without rewriting the core, because the responsibilities were separated correctly up front.

---

## Visual / Diagram description

The sequence below shows the **return-triggers-reservation** flow — the most interesting interaction in the system.

```
 Member(Alice)      Library         BookItem        Book         ReservationNotifier   Member(Carol)
     │                 │               │              │                  │                   │
     │ returnItem(copyA)                              │                  │                   │
     │────────────────▶│               │              │                  │                   │
     │                 │ set returnDate│              │                  │                   │
     │                 │──────────────▶│              │                  │                   │
     │                 │ calculateFine()              │                  │                   │
     │                 │◀──────────────│  (on Loan)   │                  │                   │
     │                 │ any WAITING reservation?     │                  │                   │
     │                 │─────────────────────────────▶│                  │                   │
     │                 │              yes             │                  │                   │
     │                 │ status = RESERVED            │                  │                   │
     │                 │──────────────▶│              │                  │                   │
     │                 │ onCopyReturned(book)                            │                   │
     │                 │────────────────────────────────────────────────▶│                   │
     │                 │                              │   notify("available")                │
     │                 │                              │                  │──────────────────▶│
     │                 │                              │                  │   (Carol reacts)  │
```

**How to read it.** Alice's return goes to the `Library` orchestrator. The orchestrator asks the `Loan` to compute the fine (object-local logic), asks the `Book` whether anyone is queued (it is — Carol), flips the copy to `RESERVED`, and calls `ReservationNotifier.onCopyReturned`. The notifier — the Observer subject — publishes to Carol, who reacts to the message. Alice never knew Carol existed; the `Library` never formatted a notification. Each box does exactly one thing.

---

## Real world examples

### Koha (open-source library software)
Koha is a real, widely deployed open-source integrated library system. It models the exact distinction we built: a **bibliographic record** (the title/ISBN — our `Book`) versus **items** (individual physical copies with barcodes — our `BookItem`). Circulation (checkout/return), holds (reservations), and overdue fines are all first-class. If you want to see this design at production scale, Koha is the reference implementation. *(Representative of how real ILS software is structured.)*

### Amazon (catalog vs. inventory unit)
Amazon's own systems draw the same line, generalized: an **ASIN/product listing** is the abstract item (our `Book`), while each physical unit in a fulfillment center has its own tracking identity (our `BookItem`). "Only 2 left in stock" is `availableCount()`; "notify me when back in stock" is a `Reservation` with Observer-style notification. The library model is inventory management in miniature. *(Conceptual.)*

### Public library kiosks (self-checkout)
The self-service machine you scan your card and book at is a thin client over exactly this object model. Scanning the card identifies the `Member`; scanning the barcode identifies the `BookItem`; the machine creates a `BookLending`, prints the due date, and — if you have holds ready — reads them off the reservation queue. The whole UX is our seven classes with a touchscreen on top.

---

## Trade-offs

**`Book` vs. `BookItem` split**

| Pros | Cons |
|---|---|
| Per-copy identity: track exactly which copy is lost/overdue | Two classes instead of one — slightly more code |
| Accurate `availableCount()`, per-copy status | Must keep the back-reference (`item.book`) consistent |
| Extends cleanly to branches, item types | Overkill for a toy "count only" inventory |

**Observer for reservations**

| Pros | Cons |
|---|---|
| Book doesn't know who is waiting or how to reach them | Indirection — harder to trace the flow when debugging |
| New notification channels (SMS/email/push) drop in | Ordering/delivery guarantees need care in real systems |
| Decouples "copy returned" from "notify member" | Overkill if you only ever `console.log` one message |

**Orchestrator (`Library`) vs. fat domain objects**

| Pros | Cons |
|---|---|
| Cross-object workflows live in one clear place | Risk of the orchestrator itself bloating if undisciplined |
| Domain objects stay small and testable | One more layer to understand |

**The sweet spot:** model `Book` and `BookItem` separately (always — it is the whole point of this problem), use Observer for reservation notifications, and keep a thin orchestrator that *coordinates* but *delegates* all object-local logic. Reach for Strategy/Factory only when the interviewer adds variability that justifies them.

---

## Common interview questions on this topic

### Q1: "Why not just put a `quantity` field on `Book`?"
**Hint:** Because you lose per-copy identity. You cannot say "copy `CC-0001` is overdue" or "copy `CC-0002` is water-damaged, mark it LOST," and you cannot track *which specific copy* a member borrowed for the return/fine flow. A counter answers "how many," never "which one." The `Book`→`BookItem` one-to-many is mandatory.

### Q2: "A book is returned and three people reserved it — who gets it?"
**Hint:** The reservation queue is **FIFO**, so the member who reserved earliest (front of `reservationQueue` with status `WAITING`) is notified first and the copy is held `RESERVED` for them. In a real system you'd add a pickup window (e.g., 48 hours) after which the reservation expires and the next person is notified.

### Q3: "How do you keep search fast as the catalog grows to millions of titles?"
**Hint:** Don't scan the list — **index** it. We keep a `Map` per search key (ISBN, title, author) so lookups are O(1)/O(log n) instead of O(n). Say explicitly that this mirrors how a database builds an index per queried column; the in-memory Maps are the same idea a `CREATE INDEX` gives you.

### Q4: "Where does the fine calculation live, and why there?"
**Hint:** On `BookLending` (the loan), because the loan owns the `dueDate` and `returnDate` — everything `calculateFine` needs. Putting it on the `Library` orchestrator would pull loan internals up into a god class. Object-local logic belongs on the object that holds the data (good cohesion; recall 24).

### Q5: "The interviewer says 'now notify by SMS too.' What changes?"
**Hint:** Almost nothing — that is the payoff of Observer. `Member` holds a list of notification channels, each implementing `send(message)`. Adding SMS is one new `SmsNotifier` class; `Library` and `ReservationNotifier` are untouched. If you had hard-coded `console.log`, every notification site would need editing.

---

## Practice exercise

### Build the "renew" feature (about 30 minutes)

Take the code from section 5 and add a `renew(member, bookItem)` method to the `Library` orchestrator. Requirements:

1. A member may renew a book they currently hold, extending its `dueDate` by another `LOAN_PERIOD_DAYS`.
2. **Renewal must fail** if there is a `WAITING` reservation on that title — the person waiting has priority. Throw a clear error.
3. Renewal must fail if the loan is already overdue (make them return and pay the fine first).
4. Add a line to `main()` that: (a) successfully renews a book with no reservations, and (b) attempts to renew a reserved book and prints the expected failure.

**What to produce:** the updated `renew` method plus two demo lines proving both the success and the blocked-by-reservation cases. Bonus: return the new due date and log it. This exercise forces you to reason about *who owns the reservation check* — and you will find it belongs in the orchestrator, because it spans the loan, the book, and the reservation queue.

---

## Quick reference cheat sheet

- **`Book` = the title, `BookItem` = the physical copy.** This one distinction is the entire lesson of the problem; never merge them.
- **One-to-many:** one `Book` has many `BookItem`s; one `Member` holds many `BookLending`s.
- **`Account` base class** shares identity fields between `Member` and `Librarian` (inheritance for common data).
- **Ownership rule:** object-local logic on the object (`Loan.calculateFine`, `BookItem.isAvailable`), cross-object workflows on the `Library` orchestrator.
- **Enums** via `Object.freeze({...})` — JS has no native enum. `BookStatus`, `ReservationStatus`, `AccountType`.
- **`Catalog` uses Map indexes** per search key (ISBN/title/author) — mirrors database column indexing for O(1) lookups.
- **Reservation queue is FIFO** — earliest reserver notified first when a copy returns.
- **Observer pattern** notifies the next waiting member on return; the book doesn't know who or how (recall 41).
- **Fine lives on the `Loan`**, computed from `dueDate` vs. `returnDate` at a `ratePerDay`.
- **Borrow limit** is a `Member` field; `member.atLimit()` enforces it during checkout.
- **Avoid the god-class trap:** the orchestrator coordinates, it does not hoard data or logic.
- **Extensions drop in** (e-books via item subtypes + Factory, SMS via Observer channels, tiers via a field, branches via a location ref) because responsibilities were separated (recall 24).

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → verbs → diagram → code → patterns → extensions) we followed here. |
| **Next** | [117 — HLD/LLD Hotel Booking](./117-lld-hotel-booking.md) | The next canonical entity-modelling problem; reuses the abstract-vs-instance idea (room type vs. room) and reservations. |
| **Related** | [112 — LLD Parking Lot](./112-lld-parking-lot.md) | The sibling classic; same skill of turning nouns into classes with the right multiplicities. |
| **Related** | [41 — Observer Pattern](./41-pattern-observer.md) | The pattern powering reservation notifications — subject publishes, observers react. |
| **Related** | [24 — Coupling & Cohesion](./24-coupling-and-cohesion.md) | Why we split Catalog, Loan, and notification concerns and kept the orchestrator thin. |
