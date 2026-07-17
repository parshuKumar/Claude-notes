# 108 — Design Google Docs / Collaborative Editing
## Category: HLD Case Study

---

## What is this?

Design a system where **many people edit the same document at the same time** and every one of them sees a live, converging view — you type a sentence, your colleague deletes a word two lines up, and within a heartbeat both of you are looking at the *identical* document.

Think of a shared whiteboard where twenty markers move at once. The magic isn't the drawing — it's that no matter who drew what in what order, everyone ends up seeing the same picture. That guarantee, called **convergence**, is the whole game. This is the hardest consistency problem in the course.

---

## Why does it matter?

Most systems you have designed so far had a *single source of truth* — one database row, one authoritative value. Collaborative editing shatters that assumption: several people mutate the same object concurrently, offline, from different continents, and you must merge those mutations so that **nobody's intent is lost and everybody converges**.

**Interview angle:** This question separates people who memorized architectures from people who understand *distributed correctness*. The interviewer wants to hear "Operational Transformation vs CRDTs," why last-write-wins is catastrophic here, and how you keep replicas convergent. If you only talk about WebSockets and load balancers, you've missed the point.

**Real-work angle:** The exact same problem shows up in multiplayer design tools (Figma), shared spreadsheets, collaborative code editors (VS Code Live Share), whiteboards, and offline-first mobile apps that sync. The moment two clients can edit the same state independently, you own this problem. Getting it wrong means silent data loss — the worst kind of bug, because users only notice their paragraph vanished an hour later.

---

## The core idea — explained simply

### Analogy: two editors marking up the same paper, in different rooms

Imagine a printed sheet that says **`cat`**. You photocopy it and hand one copy to Alice in Room A and one to Bob in Room B. They can't see each other.

- **Alice** inserts an `s` at the end → her paper now reads `cats`.
- **Bob**, at the very same moment, deletes the first letter → his paper now reads `at`.

Both edits were made against the **same original** `cat`. Now you must reconcile the two rooms into one final document that *both* Alice and Bob will accept as correct. What should it say?

Intuitively: Bob removed the `c`, Alice added an `s` at the end. The merged intent is `ats`. Both edits survived. Nobody's work was thrown away. **That reconciliation — done automatically, on every keystroke, across the network — is collaborative editing.**

Map the analogy back to the technical concepts:

| Analogy | Technical concept |
|---|---|
| The printed sheet `cat` | The shared document state (a replicated data structure) |
| Alice and Bob in separate rooms | Two replicas / clients editing concurrently |
| "insert s at end", "delete first letter" | **Operations** (the unit we send over the wire, not the whole doc) |
| Both edits made against `cat` | Concurrent operations against the same base version |
| Merging to `ats` with nobody's work lost | **Convergence** + intention preservation |
| The photocopier and messengers | The network + sync server |

The naive approach — "whoever's edit reaches the server last wins" (**last-write-wins**, LWW) — would take one paper and throw the other in the trash. Alice's `s` or Bob's deletion silently disappears. That is unacceptable for a document. So we need something smarter than LWW: a merge that is **commutative** (order-independent) and **intention-preserving**.

---

## Key concepts inside this topic

We'll execute the standard **5-step HLD framework** — (1) Requirements, (2) Capacity, (3) API, (4) Data model, (5) Architecture — and then go deep on the two solutions (OT and CRDT), sync, persistence, and failure modes.

### 1. Requirements (Step 1 of the framework)

**Clarifying questions to ask the interviewer first:**
- How many concurrent editors per document? (Usually small — 1 to ~20. This matters: it means per-doc scale is tiny, but correctness is everything.)
- Rich text (bold, images, tables) or plain text? (Start with plain text; rich text is layered on top.)
- Must it work offline and sync later? (Yes — this is a defining feature and it heavily influences OT-vs-CRDT.)
- Do we need full edit history / version restore? (Yes.)

**Functional requirements:**
- Multiple users edit the **same document simultaneously** and see each other's changes in **near-real-time**.
- All changes **converge** — everyone ends with the *identical* document, byte for byte.
- Show **cursors and presence** of other editors (who is here, where their caret is).
- **Edit history** — view and restore previous versions.
- **Offline editing** that reconciles automatically on reconnect.

**Non-functional requirements:**
- **Low latency** — a local edit must appear instantly (< ~100 ms round-trip for remote edits to show).
- **Strong eventual consistency** — this is the hard one. All replicas **MUST** converge to the same state. *A document where two users see different text is broken.* This is a correctness requirement, not a nice-to-have.
- **High availability** — the editor should keep working even during partitions; queue locally, sync later.
- **Graceful handling of concurrent conflicting edits** — never lose intent, never require a human to resolve a merge.

### 2. Capacity estimation (Step 2)

The headline: **the challenge is correctness of concurrent merge, not raw scale.** Still, do the math.

Assume the product has **50M daily active users**, each with ~5 documents open per day, and at peak **2M documents are actively being edited** at once.

- **Concurrent editors per doc:** small — model an average of 3, peak 20. So one doc's session is trivial to hold in a single server's memory.
- **Edit event rate:** *every keystroke is an operation.* An active typist emits ~5 ops/sec while typing. If 2M docs are active and, say, 10% have someone actively typing at any instant → 200,000 typers × 5 ops/s = **1,000,000 operations/second** flowing through the system at peak. That's the real load — millions of tiny messages, not gigabytes.
- **Operation size:** an op is tiny — `{type, position, char, siteId, version}` ≈ 50–100 bytes. 1M ops/s × 100 bytes = **100 MB/s** of op traffic ≈ 0.8 Gbps. Very manageable when sharded per-doc.
- **Storage (append the op log):** 1M ops/s × 100 bytes × 86,400 s/day = **~8.6 TB/day** of raw ops before compaction. Over 5 years that's petabytes — which is exactly why we **snapshot and compact** (see Persistence). We don't keep every keystroke forever.
- **Materialized doc storage:** 500M total docs × avg 50 KB = **25 TB** for current document state. Cheap.

Takeaway: per-document load is tiny (a few editors, a few ops/sec), so a single server easily hosts one doc's session. The system-wide challenge is (a) routing all editors of one doc to the same session, and (b) merging concurrent ops correctly. **Correctness dominates scale here.**

### 3. The core problem — concurrent edits conflict (state it vividly)

Let's nail the problem down concretely before proposing solutions.

The document is **`cat`** (positions: `c`=0, `a`=1, `t`=2).

- **User A** inserts `s` at position 3 → intends `cats`. Op: `insert("s", 3)`.
- **User B**, at the same instant, deletes position 0 (`c`) → intends `at`. Op: `delete(0)`.

Both ops were generated against the base `cat`. Now watch what happens if we apply them **naively in different orders on different machines:**

**On A's machine** (A already applied its own op, so it has `cats`, then B's `delete(0)` arrives):
```
cats  --delete(0)-->  ats        ✓ (deletes the 'c')
```

**On B's machine** (B already has `at`, then A's `insert("s", 3)` arrives):
```
at    --insert("s",3)-->  a t _ s   position 3 doesn't even exist!
```
`at` only has positions 0 and 1. Applying `insert at 3` is out of bounds, or if we clamp it, produces `ats` by luck — but change the example slightly and it produces garbage or crashes. The positions **shifted** the moment B deleted a character, and A's op still refers to the *old* coordinate system.

This is the entire problem in one picture: **an operation carries positions that were valid against the version it was created on, but by the time it arrives elsewhere, the document has moved underneath it.** Apply it blindly and replicas diverge — A sees `ats`, B sees something else. Divergence = broken document.

And **last-write-wins won't save you:** if the server just kept "the last op it received" and discarded the other, one user's entire edit evaporates. You need to **merge both** so the result is `ats` on *every* replica and *both* intents survive. Two families of solutions exist: **Operational Transformation** and **CRDTs**.

### 4. Operational Transformation (OT) — the first solution

**Core idea:** when you receive a remote operation that was generated against an *older* version of the document than the one you currently hold, you don't apply it as-is. You **transform** it against the operations that happened in between — adjusting its parameters so it makes sense in your current coordinate system. An insert that happened before your position pushes your position over by one; a delete before your position pulls it back by one.

Walk our example through OT. B's `delete(0)` removed a character at position 0, which is *before* A's insert position 3. So when A's op arrives at B (or is transformed by the server after B's op), we transform it:

```
Original op A:              insert("s", position = 3)
Concurrent op B:            delete(position = 0)   // removed a char before pos 3
Transform A against B:      a char before pos 3 was deleted, so shift left by 1
Transformed op A':          insert("s", position = 2)
```

Now apply the transformed op to B's document `at`:
```
at  --insert("s", 2)-->  ats     ✓
```

And on A's side, B's `delete(0)` doesn't need transformation (position 0 is before A's insert, deleting the `c` still targets the right character in `cats`):
```
cats  --delete(0)-->  ats        ✓
```

**Both replicas converge to `ats`. Both intents preserved.** That is OT working.

**Key properties OT must satisfy** (state these at a high level in an interview):
- **TP1 (convergence):** for two concurrent ops `a` and `b`, applying `a` then `transform(b, a)` must yield the same document as applying `b` then `transform(a, b)`. Both paths reach the identical state.
- **TP2:** transformation functions must be consistent across *longer* chains of concurrent operations, so that transforming through different intermediate orderings still converges. TP2 is the famously brutal one — the Google **Wave** team publicly documented how many subtle edge cases (especially with rich text and complex ops) break naive transform functions. Getting all pairs of op-types (insert/insert, insert/delete, delete/delete, with position ties) correct is genuinely hard.

**OT typically needs a central server** to impose a single global order on operations. Each client sends its op, the server assigns it a sequence number, transforms it against anything the client hadn't seen yet, applies it, and broadcasts the transformed op with its version. The central authority makes reasoning about "what happened in between" tractable. **Google Docs uses OT.** The upside: small metadata (ops carry plain integer positions), well-understood with a server. The downside: the transformation functions are complex and easy to get subtly wrong, and it's awkward for pure peer-to-peer.

### 5. CRDTs — the modern alternative

**Conflict-free Replicated Data Types** attack the problem from the other end. Instead of *fixing up* operations after the fact with transforms, you **design the data structure so that concurrent operations always commute** — they can be applied in any order on any replica and *always* produce the same result, with **no transformation and no central coordinator**.

The trick for text is to **stop using integer positions**, because integers shift the instant anyone inserts or deletes. Instead, give **every character a unique, immutable, densely-orderable identifier.** Once assigned, a character's ID never changes. An insert doesn't say "at position 3" (a coordinate that moves) — it says "between the character with ID X and the character with ID Y" (stable anchors that never move).

The identifier is often a **fractional index** or a **path**: to insert between two characters whose IDs are `0.4` and `0.5`, you mint `0.45`. Between `0.45` and `0.5` you mint `0.475`. There's always room to squeeze another character in — the space is dense. (Real systems like **Logoot / LSEQ** use variable-length paths, **RGA** uses per-char unique IDs with a linked structure, and **Yjs** uses an optimized variant.)

**Concurrent inserts "at the same place"** — two people inserting between the same two anchors — would mint colliding IDs. Resolve the tie **deterministically** by appending a unique **site ID** (client ID) and a counter. Every replica breaks the tie the same way, so ordering is identical everywhere. **Deletes become tombstones:** you don't physically remove the character (that would break other ops referencing it); you mark it deleted and hide it from the rendered text.

Because operations commute, **replicas converge simply by exchanging ops in any order, with no server needed to sequence them.** This is why CRDTs are the natural fit for **peer-to-peer and offline-first** apps — a client that was offline for an hour just replays its queued ops and receives everyone else's; order doesn't matter; everything merges.

**Honest OT vs CRDT comparison:**

| Dimension | Operational Transformation (OT) | CRDT |
|---|---|---|
| Coordination | Needs a **central server** to order ops | **Decentralized** — no coordinator required |
| Positions | Integer positions, transformed on receipt | Immutable unique IDs, no transform |
| Merge logic | Complex transform functions (TP1/TP2) | Simpler — ops commute by design |
| Metadata size | **Small** (ops are tiny) | **Larger** — per-char IDs + tombstones add overhead |
| Offline / P2P | Awkward — must transform against everything missed | **Natural** — replay in any order |
| Correctness risk | Easy to get transforms subtly wrong | Merge is easy; memory/GC of tombstones is the pain |
| Used by | **Google Docs** (and historically Wave) | **Figma**, **Yjs**, **Automerge**, many P2P apps |

**Rule of thumb:** if you have a reliable central server and want minimal per-character overhead, OT. If you need offline-first, peer-to-peer, or want simpler merge logic and can pay the metadata/tombstone cost, CRDT.

### 6. The real-time sync architecture (Step 5)

Now the plumbing that carries ops between editors. **Recall from [82 — Content Negotiation and Protocols]** that HTTP request/response is wrong for live bidirectional updates — you need a persistent, full-duplex channel: **WebSockets**.

Each editor opens a WebSocket to a **collaboration server** that holds the **authoritative in-memory session** for that document (the current doc state + version counter). The flow for a single edit:

1. User types. The client **applies the op locally immediately** (optimistic update) so typing feels instant — never wait for the server round-trip to render your own keystroke.
2. The client sends the op to its collab server, tagged with the client's last-seen version.
3. The server **orders** the op (assigns the next sequence/version number), transforms it (OT) or integrates it (CRDT), applies it to the authoritative doc, **persists it** (append to the op log — see §7), and only then **broadcasts** the ordered op to all *other* editors.
4. Other clients receive the ordered op and apply it; the originating client receives an **ack** confirming the server's assigned version, and reconciles (fixing up any local ops it made in the meantime).
5. **Presence and cursors** ride a **separate, lightweight channel** — cursor moves are frequent but disposable, so they don't need persistence or the same ordering guarantees; if one is dropped, the next overwrites it.

**The scaling problem (mirror of the chat system):** every editor of *one* document must reach the *same* server that holds that doc's session, so we route by **session affinity on `doc_id`** — a consistent-hash / router layer maps a doc to exactly one collab server. When editors of the same doc land on different servers (during rebalancing or failover), a **pub/sub backplane** (Redis Pub/Sub or Kafka, keyed by `doc_id`) fans ordered ops out between servers. **Recall from [99 — HLD Chat System]** — this is the identical "get everyone in the same conversation onto the same routing path" problem you solved for chat rooms.

### 7. Persistence — log + snapshot (event sourcing)

**You do NOT save the whole document on every keystroke** — that's 1M writes/sec of full documents. Instead:

- **Append the operation log.** Every applied op is appended to an immutable log per document. This is **event sourcing** (recall the pattern): the document is not the source of truth — the *sequence of operations* is. Replay the log from the start and you reconstruct any version. History, undo, and "restore to yesterday" all fall out for free, because you can replay to any point.
- **Periodically snapshot** the materialized document (e.g. every N ops or every few minutes) so that replay isn't unbounded — you load the latest snapshot and replay only the ops *after* it, not from the beginning of time.
- **Compact** old ops: once a snapshot subsumes a range of ops, the fine-grained keystroke ops before it can be collapsed/archived to cold storage. You keep coarse version checkpoints for history, not every character.

```
Op log (append-only):   [op1][op2][op3]...[op500] [op501]...[op1000] [op1001]...
                                         │                  │
                        Snapshot@500 ────┘   Snapshot@1000 ─┘
                                                             ▲
To load current doc:  take Snapshot@1000, replay op1001.. onward.
To view version 700:  take Snapshot@500, replay op501..op700.
```

Snapshots go in blob/object storage (durable, cheap); the recent op log lives in a fast append-optimized store (a log like Kafka, or an append-friendly DB) so writes are cheap and sequential.

### 8. Deep dives (the hardest sub-problems)

**(a) Offline editing and sync-on-reconnect.** While offline, the client **queues its local ops** and keeps editing against its local replica. On reconnect it must merge its queue with everything that happened server-side while it was gone. **CRDTs make this natural** — the queued ops commute with the remote ops, so the client just exchanges op sets and everything converges regardless of order. **OT is harder** — each queued op must be transformed against the entire sequence of ops that occurred during the offline window (and vice versa), a long transformation chain where TP2 correctness really bites.

**(b) Undo/redo in a collaborative setting.** In a single-user editor, undo = "revert the last change." In a shared doc, **undo must only revert YOUR edit, not your collaborator's.** If Alice hits Ctrl-Z, she expects *her* last insert gone — not Bob's paragraph that arrived after it. This means undo can't be "pop the global stack"; it must generate a *new inverse operation* targeting your specific past op, then transform/integrate that inverse against everything that happened since. It's surprisingly hard and a classic follow-up question.

**(c) Cursor and selection presence.** Broadcast each user's caret position and selection range on the lightweight presence channel. Because positions shift as others edit, cursors are also expressed relative to stable anchors (in CRDTs, an anchor ID; in OT, transformed like any position). Presence is ephemeral — no persistence, last-write-wins is fine here (the opposite of document ops).

**(d) Access control and sharing.** Per-document ACLs: owner / editor / commenter / viewer. The collab server checks permission on WebSocket connect and rejects or downgrades ops (a viewer's ops are refused). Sharing links map a token to a role.

**(e) Large documents and performance.** A 200-page doc with millions of tombstones (CRDT) gets heavy — mitigate with tombstone garbage collection once all replicas have acknowledged a delete, and by structuring the document as a tree of blocks so edits touch only a subtree rather than re-scanning the whole doc.

### 9. Bottlenecks and failure modes

- **A collab server dies mid-session.** Its in-memory authoritative doc is lost — but the **op log was persisted before broadcast**, so nothing is truly gone. Clients detect the dropped WebSocket, reconnect (the router assigns a new server for that `doc_id`), the new server **rebuilds the session from the latest snapshot + replayed ops**, and clients **replay their un-acked local ops** from their last acknowledged version. Everyone re-converges.
- **Why persist-and-ack ordering *before* broadcasting matters:** if the server broadcast an op to peers *before* durably logging it and then crashed, peers would have an op the recovered server has never heard of — permanent divergence. So the invariant is strict: **assign version → persist → then broadcast.** Ordering and durability come before fan-out. This is the single most important correctness rule in the whole design.
- **Hot document / thundering herd on reconnect:** if a popular doc's server dies, all its editors reconnect at once. Mitigate with jittered reconnect backoff and by keeping snapshots warm.

---

## Visual / Diagram description

The real-time sync architecture:

```
   Alice (browser)        Bob (browser)         Carol (offline→reconnects)
        │                     │                        │
   optimistic                optimistic          queues ops locally
   local apply               local apply               │
        │  WebSocket          │  WebSocket              │  WebSocket (on reconnect)
        ▼                     ▼                         ▼
   ┌──────────────────────────────────────────────────────────────┐
   │                     ROUTER / LOAD BALANCER                     │
   │        session affinity: hash(doc_id) → one collab server     │
   └───────────────┬───────────────────────────┬──────────────────┘
                   │  (all editors of doc_42)   │
                   ▼                            ▼
        ┌─────────────────────┐      ┌─────────────────────┐
        │  Collab Server  N1  │      │  Collab Server  N2  │
        │  ┌───────────────┐  │      │   (holds other docs)│
        │  │ doc_42 SESSION│  │◀────▶│                     │
        │  │ state + ver=57│  │ pub/ │  Redis / Kafka      │
        │  └───────┬───────┘  │ sub  │  backplane keyed    │
        │  1.order 2.transform│ back-│  by doc_id          │
        │  3.PERSIST 4.broadcast│plane│                     │
        └─────────┬───────────┘      └─────────────────────┘
                  │ append op (before broadcast!)
                  ▼
        ┌──────────────────────┐     ┌──────────────────────┐
        │   OP LOG (append-only)│────▶│  SNAPSHOT STORE       │
        │   Kafka / log store   │ every│ (object storage)     │
        │   [op1..op57]         │  N ops replay = current doc │
        └──────────────────────┘     └──────────────────────┘

   (Cursors/presence travel on a separate lightweight channel — not shown,
    not persisted, last-write-wins.)
```

**How to read it:** each editor keeps its own local replica and applies edits optimistically for instant feedback. Ops flow over WebSockets to a router that pins every editor of `doc_42` to the *same* collab server via session affinity. That server is the authority: it **orders → transforms → persists → broadcasts** (in that exact order). The op log + periodic snapshots give durable history and crash recovery. The pub/sub backplane bridges servers when editors of one doc are split across nodes. Presence rides a separate throwaway channel.

---

## Real world examples

### Google Docs (OT)

Google Docs uses **Operational Transformation** with a central server that sequences and transforms every operation. Its predecessor, **Google Wave**, was an ambitious OT system whose engineering team publicly documented (representative accounts widely cited in the OT literature) just how many transform-function edge cases exist for rich content — a big reason the industry later gravitated toward CRDTs for new projects. Docs keeps a fine-grained revision log so you get version history and "see edit history" for free from the op log.

### Figma (CRDT-style)

Figma's multiplayer design canvas uses a **CRDT-inspired** model: each object/property has a last-writer-wins register keyed by a unique ID, and a central server relays changes. Because design objects have stable IDs, concurrent moves and edits to different properties merge cleanly without transformation. (Representative — Figma has described a custom CRDT-like scheme rather than a textbook RGA/Logoot.)

### Yjs / Automerge (open-source CRDT libraries)

**Yjs** and **Automerge** are widely used open-source **CRDT** libraries for building collaborative apps (editors, whiteboards, note apps). They implement optimized sequence CRDTs with tombstones and support offline-first, peer-to-peer sync out of the box — you can wire them to WebSockets or even WebRTC with no central authority. They're the go-to when you want CRDT semantics without writing the merge logic yourself.

---

## Trade-offs

**OT vs CRDT (the central decision):**

| | OT | CRDT |
|---|---|---|
| Needs central server | Yes (to order ops) | No |
| Metadata overhead | Low | Higher (IDs + tombstones) |
| Offline / P2P | Hard | Natural |
| Merge complexity | High (transform fns) | Lower (commute by design) |
| Memory over time | Stable | Grows with tombstones (needs GC) |

**Optimistic local apply vs server-authoritative:**

| Choice | Pro | Con |
|---|---|---|
| Apply locally first (optimistic) | Instant, responsive typing | Must reconcile if server orders differently |
| Wait for server confirmation | Simpler, always consistent | Every keystroke feels laggy — unusable |

**Persistence: op log vs materialized doc only:**

| Choice | Pro | Con |
|---|---|---|
| Append op log (event sourcing) | Free history, undo, crash recovery, cheap writes | Replay cost → needs snapshots + compaction |
| Save whole doc each time | Simple to read current state | Huge write volume, no history, LWW-flavored |

**The sweet spot:** WebSockets + session affinity by `doc_id`, **optimistic local apply** for responsiveness, a **central server that orders → persists → broadcasts**, an **append-only op log with periodic snapshots** for durability and history, and either OT (if you're server-centric like Docs) or a CRDT (if you need offline-first / P2P like Figma and Yjs). The non-negotiable invariant: **never broadcast an op you haven't durably ordered and persisted.**

---

## Common interview questions on this topic

### Q1: "Two users edit `cat` at the same time — A appends `s`, B deletes the first letter. Walk me through how both converge."
**Hint:** State the divergence risk first (positions shift; naive apply goes out of bounds; LWW loses an edit). Then show OT: transform A's `insert("s", 3)` against B's `delete(0)` → a char before position 3 was removed → shift to `insert("s", 2)`, giving `ats` on both replicas. Mention TP1 guarantees this converges regardless of apply order. Alternatively, describe the CRDT path: characters have stable IDs, so A's insert anchors relative to `t`'s ID and B's delete tombstones `c`'s ID — they commute, both get `ats`.

### Q2: "OT vs CRDT — which would you pick and why?"
**Hint:** No dogmatic answer. If there's a reliable central server and you want minimal metadata → OT (what Docs uses). If you need offline-first/P2P or want simpler merge logic and can pay tombstone/metadata overhead → CRDT (Figma, Yjs, Automerge). Name the trade: OT = small ops but brutal transform functions (TP2); CRDT = simple merge but memory grows with tombstones (needs GC).

### Q3: "Why can't you just use last-write-wins?"
**Hint:** LWW keeps one op and discards the other, silently destroying a user's edit. Documents require **intention preservation** — both edits must survive and everyone must converge. LWW is acceptable only for *ephemeral, single-owner* fields like cursor position, never for shared document content.

### Q4: "How do you scale so all editors of one doc see each other?"
**Hint:** Session affinity — route by `hash(doc_id)` so every editor of a doc lands on the same collab server holding that session. When they're split across servers (rebalancing/failover), use a pub/sub backplane (Redis/Kafka) keyed by `doc_id` to fan ops between servers. It's the same routing problem as the chat system (99).

### Q5: "A collab server crashes mid-edit. What happens?"
**Hint:** Nothing is lost *if* the invariant held: ops were persisted to the log *before* broadcast. Clients detect the dropped socket, reconnect (router assigns a new server), the new server rebuilds the session from latest snapshot + replayed ops, and clients replay un-acked local ops from their last acknowledged version. Everyone re-converges. Stress: persist-then-broadcast is what makes this safe.

---

## Practice exercise

### Build a two-client OT convergence demo (Node.js, ~30-40 min)

Write a small Node.js program (no network needed — simulate two clients as objects in one process) that proves convergence:

1. Represent a document as a plain string, starting at `"cat"`.
2. Define two op types: `{ type: 'insert', ch, pos }` and `{ type: 'delete', pos }`, and an `apply(doc, op)` function.
3. Write a `transform(op1, op2)` function that adjusts `op1`'s position to account for `op2` having already been applied (insert before shifts right; delete before shifts left; handle a position tie with a site-id rule).
4. Create client A with op `insert('s', 3)` and client B with op `delete(0)`, both against `"cat"`.
5. Apply them in **both orders** on two separate copies of the doc, transforming the second op against the first each time.
6. **Assert that both copies end up as the identical string** (`"ats"`). Print both results.

**What to produce:** the running script plus a one-paragraph note explaining *why* the transform was needed and which TP property your assertion is demonstrating (TP1 — convergence under both orderings). Bonus: add a third concurrent op and see whether your transforms still converge (that's TP2 territory, and where it gets hard).

---

## Quick reference cheat sheet

- **Convergence:** the core requirement — every replica MUST end at the identical document. Divergence = broken.
- **Operation:** the unit sent over the wire — `insert`/`delete` with parameters — not the whole document.
- **Last-write-wins (LWW):** unacceptable for document content (loses an edit); fine only for ephemeral fields like cursors.
- **Operational Transformation (OT):** transform an incoming op against ops it hadn't seen, adjusting positions; needs a central server; small metadata; hard transform functions. **Google Docs.**
- **TP1 / TP2:** OT's convergence properties — TP1 = two concurrent ops converge under both orderings; TP2 = consistency across longer concurrent chains (the brutal one).
- **CRDT:** design ops to commute — stable unique IDs instead of shifting integer positions; no central coordinator; deletes = tombstones; natural for offline/P2P. **Figma, Yjs, Automerge.**
- **Tombstone:** a deleted character kept (hidden) so other ops referencing it still resolve; needs garbage collection later.
- **Optimistic local apply:** render your own keystroke instantly, reconcile with server ordering afterward — essential for < 100 ms feel.
- **Session affinity:** route all editors of one `doc_id` to the same collab server; pub/sub backplane bridges servers.
- **The invariant:** assign version → **persist** → then **broadcast.** Never broadcast an un-logged op.
- **Event sourcing:** the op log is the source of truth; snapshots + compaction bound replay cost; history/undo are free.
- **Presence channel:** cursors/selections travel separately, unpersisted, LWW — cheap and disposable.
- **WebSockets:** the transport (recall 82) — persistent full-duplex, because request/response can't push live edits.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [./82-content-negotiation-and-protocols.md](./82-content-negotiation-and-protocols.md) | WebSockets — the persistent full-duplex channel that carries live ops between editors. |
| **Next** | [./99-hld-chat-system.md](./99-hld-chat-system.md) | Same routing problem: get everyone in one room/doc onto the same server via session affinity + pub/sub. |
| **Related** | [./76-eventual-consistency.md](./76-eventual-consistency.md) | Collab editing demands *strong eventual consistency* — replicas diverge briefly but MUST converge. |
| **Related** | [./67-message-queues.md](./67-message-queues.md) | The pub/sub backplane and the append-only op log lean on queue/log infrastructure (Kafka/Redis). |
| **Related** | [./08-consistency-models.md](./08-consistency-models.md) | Where this sits on the consistency spectrum — and why LWW is the wrong model for shared documents. |

---

## Appendix — reference Node.js code

### A minimal OT transform function

```javascript
// An operation is one of:
//   { type: 'insert', ch: 's', pos: 3, site: 1 }
//   { type: 'delete', pos: 0, site: 2 }
// `site` is a unique client id used ONLY to break position ties deterministically.

function apply(doc, op) {
  if (op.type === 'insert') {
    return doc.slice(0, op.pos) + op.ch + doc.slice(op.pos);
  }
  // delete
  return doc.slice(0, op.pos) + doc.slice(op.pos + 1);
}

// Transform `op` so it can be applied AFTER `against` has already been applied.
// Returns a new op with an adjusted position. This is the heart of OT.
function transform(op, against) {
  const shifted = { ...op };

  if (against.type === 'insert') {
    // A character was inserted at `against.pos`.
    // If it landed before us, everything to our right shifted one step right.
    if (against.pos < op.pos) {
      shifted.pos = op.pos + 1;
    } else if (against.pos === op.pos && op.type === 'insert') {
      // Concurrent inserts at the SAME spot: break the tie by site id so
      // every replica orders them identically (deterministic convergence).
      if (against.site < op.site) shifted.pos = op.pos + 1;
    }
  } else {
    // `against` deleted a character at `against.pos`.
    if (against.pos < op.pos) {
      // A char before us was removed -> shift left by one.
      shifted.pos = op.pos - 1;
    }
    // (If both delete the SAME index, a real system marks it already-deleted;
    //  omitted here for brevity.)
  }
  return shifted;
}

// --- Prove convergence (TP1): apply A,B and B,A -> identical result ---
const base = 'cat';
const opA = { type: 'insert', ch: 's', pos: 3, site: 1 }; // cat -> cats
const opB = { type: 'delete', pos: 0, site: 2 };          // cat -> at

// Order 1: apply A, then B transformed against A
let doc1 = apply(base, opA);              // 'cats'
doc1 = apply(doc1, transform(opB, opA));  // delete(0) unaffected -> 'ats'

// Order 2: apply B, then A transformed against B
let doc2 = apply(base, opB);              // 'at'
doc2 = apply(doc2, transform(opA, opB));  // insert('s',3)->insert('s',2) -> 'ats'

console.log(doc1, doc2, doc1 === doc2);   // 'ats' 'ats' true  <-- converged
```

### The server-side WebSocket op-broadcast loop

```javascript
const { WebSocketServer } = require('ws');

// One authoritative in-memory session per document.
class DocSession {
  constructor(docId, initialText = '') {
    this.docId = docId;
    this.doc = initialText;
    this.version = 0;          // monotonically increasing op sequence number
    this.log = [];             // append-only op log (persisted in real systems)
    this.clients = new Set();  // connected WebSockets editing this doc
  }

  // Rebuild history-based transform: transform an incoming op against every
  // op the client had NOT yet seen (those after its clientVersion).
  integrate(incomingOp, clientVersion) {
    let op = incomingOp;
    for (let i = clientVersion; i < this.version; i++) {
      op = transform(op, this.log[i]); // reuse the transform() above
    }
    // 1. order  -> 2. apply
    this.doc = apply(this.doc, op);
    this.version += 1;
    this.log.push(op);
    // 3. PERSIST BEFORE BROADCAST (the invariant). Pseudo:
    //    await opLog.append(this.docId, op, this.version);
    return { op, version: this.version };
  }
}

const sessions = new Map(); // docId -> DocSession
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  const docId = new URL(req.url, 'http://x').searchParams.get('doc');
  const session = sessions.get(docId) ?? sessions.set(docId, new DocSession(docId, 'cat')).get(docId);
  session.clients.add(ws);

  // Send current state so the newcomer starts converged.
  ws.send(JSON.stringify({ kind: 'sync', doc: session.doc, version: session.version }));

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw);
    if (msg.kind === 'op') {
      // order -> transform -> apply -> persist, then broadcast the ORDERED op.
      const { op, version } = session.integrate(msg.op, msg.clientVersion);

      // Ack the sender its assigned version so it can reconcile local ops.
      ws.send(JSON.stringify({ kind: 'ack', version }));

      // Broadcast to every OTHER editor of this doc (fan-out).
      for (const peer of session.clients) {
        if (peer !== ws && peer.readyState === peer.OPEN) {
          peer.send(JSON.stringify({ kind: 'op', op, version }));
        }
      }
      // In a multi-server deployment, also publish to the pub/sub backplane
      // keyed by docId so peers on OTHER collab servers receive it.
    }
    // msg.kind === 'presence' -> relay on the lightweight channel, no persist.
  });

  ws.on('close', () => session.clients.delete(ws));
});
```
