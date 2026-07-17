# 128 — Design a Social Media Feed (Class Level)
## Category: LLD Case Study

---

## What is this?

A **social media feed** is the scrollable list of posts you see when you open Instagram or Twitter — posts from the people you follow, arranged in some order. This doc is the **class-level (LLD)** companion to the Instagram and Twitter HLDs (docs 100 and 103). There, we asked "how do we serve this to 500 million users?" Here, we ask a smaller, sharper question: **what are the clean classes that generate and rank a feed, and where are the seams that let us swap behaviour without rewriting everything?**

Think of it like designing the engine of a car versus designing the highway system. The HLD is the highway (fanout at scale, caching, sharding). The LLD — this doc — is the engine: `User`, `Post`, `Feed`, and two swappable strategy knobs (how we *gather* posts, and how we *rank* them).

---

## Why does it matter?

In an interview, "design Instagram" can be asked at two altitudes. If the interviewer says *"forget scale, show me the classes,"* they want exactly this: clean objects, clear ownership, and — the star of the show — the two **Strategy** seams. Candidates who hard-code "sort by time" into the feed service fail the follow-up *"now rank by engagement"* because they have to gut the class. Candidates who put ranking behind a `RankingStrategy` interface answer it in 15 lines.

In real work, feeds change constantly. Product ships chronological, then A/B tests an ML ranker, then injects ads, then adds a "seen" filter. If your feed logic is one 400-line method, every change is terrifying. If generation and ranking are strategies, every change is a new small class. This doc teaches you to build the second kind — and to recognise that the **pull-vs-push** decision you met at scale in the HLD is, at the class level, *just another Strategy*.

---

## The core idea — explained simply

### The analogy: a newspaper editor with two adjustable habits

Imagine a personal newspaper editor who prepares your morning paper. Every day the editor does two separable jobs:

1. **Gathering** — collect the stories worth including. The editor has two working styles. Style A ("pull"): when you ask for the paper, the editor runs to every reporter you subscribe to and grabs their latest stories *right then*. Style B ("push"): every time a reporter files a story, a clerk immediately drops a copy into your personal inbox, so your paper is *already assembled* when you ask.
2. **Ordering** — decide what goes on page one. One editor sorts strictly newest-first. Another scores each story by how interesting it'll be to *you* (recency, how many people liked it, how close you are to the reporter) and leads with the highest score.

The key insight: **gathering and ordering are independent knobs.** You can pull-then-chronological, or push-then-engagement, in any combination. A good class design makes each knob a swappable object.

| Newspaper analogy | Technical concept | Class in our design |
|---|---|---|
| The editor | The service coordinating everything | `FeedService` |
| A reporter you subscribe to | Someone you follow | `User` (a `followingIds` edge) |
| A filed story | A post | `Post` |
| Style A: run to reporters on demand | Fanout-on-read | `PullStrategy` |
| Style B: clerk pre-fills your inbox | Fanout-on-write | `PushStrategy` |
| The clerk who copies on filing | Real-time notification | Observer (`FeedObserver`) |
| Newest-first vs score-first | Ordering policy | `RankingStrategy` subclasses |
| Your assembled paper, one page at a time | The paginated feed | `Feed` + pagination |

Map every part back and the whole design falls out of the analogy. The two "working styles" of gathering become `FeedGenerationStrategy`; the two "ordering habits" become `RankingStrategy`. Both are the **Strategy pattern** (recall [42 — Strategy](./42-pattern-strategy.md)). The clerk-on-filing is the **Observer pattern** (recall [41 — Observer](./41-pattern-observer.md)).

---

## Key concepts inside this topic

We follow the standard 7-step LLD method (recall [111 — LLD Approach Framework](./111-lld-approach-framework.md)): requirements → nouns → behaviours → class diagram → code → patterns → extensions.

### 1. Requirements & clarifying questions

**Functional requirements:**
- Users can create **posts** (text content, timestamp).
- Users can **follow** and **unfollow** other users.
- A user's **feed** shows posts authored by people they follow.
- The feed can be ordered **chronologically** OR by a **ranking** score.
- Users can **like** and **comment** on posts.
- The feed is **paginated** (you scroll, you don't get all posts at once).

**Non-functional (for this LLD, kept light):**
- Generating a feed and re-ranking it must be swappable without touching callers.
- Adding a new ranking rule = adding one class, not editing existing ones (Open/Closed).

**Clarifying questions to ask the interviewer** (asking these signals seniority):
- *"Chronological or ranked?"* → Answer: support **both**, behind a strategy. Don't pick one.
- *"Feed refresh model — pull on read, or push on write?"* → Answer: model **both** as strategies; note the trade-off, but the deep scale answer lives in the HLD.
- *"What's the scale?"* → Here we say: **assume the class design; the fanout-at-scale, celebrity problem, caching and sharding are the HLD ([100](./100-hld-instagram.md) / [103](./103-hld-twitter.md)).** This sentence tells the interviewer you know the difference between LLD and HLD — which is the whole point of doc [02](./02-hld-vs-lld.md)'s lesson.

### 2. Identify the core objects (nouns)

Underline the nouns in the requirements and you get your classes:

| Noun | Class | Responsibility |
|---|---|---|
| User | `User` | Identity + the **social graph edges** (who I follow, who follows me) |
| Post | `Post` | Author, content, timestamp, likes, comments |
| Comment | `Comment` | Text + author on a post |
| Like | (a Set of userIds on `Post`) | Cheap enough to be a field, not a class |
| Feed | `Feed` | A ranked, paginated slice of posts for one user |
| Feed generation | `FeedGenerationStrategy` | HOW posts are gathered (pull vs push) |
| Ranking | `RankingStrategy` | HOW gathered posts are ordered |
| The coordinator | `FeedService` | Owns users, posts, and the two chosen strategies |
| Real-time update | `FeedObserver` | Notified when a followed user posts (push model) |

### 3. Behaviours (verbs) + who owns them

Ownership is where juniors and seniors diverge. Put each behaviour on the object that owns the *data* it touches:

| Behaviour | Owner | Why |
|---|---|---|
| `follow(user)` / `unfollow(user)` | `User` | The user owns their own social-graph edges |
| `addLike` / `addComment` | `Post` | The post owns its engagement data |
| `createPost(user, content)` | `FeedService` | Needs to mint the post AND trigger fanout — cross-cutting |
| `generate(user)` | `FeedGenerationStrategy` | The whole point of the strategy is to own gathering |
| `rank(posts, viewer)` | `RankingStrategy` | Ordering policy lives in the ranker, nowhere else |
| `getFeed(user, page, size)` | `FeedService` | Orchestrates generate → rank → paginate |

The `FeedService` is a **coordinator**, not a god-object: it delegates gathering to a `FeedGenerationStrategy` and ordering to a `RankingStrategy`. It only owns what nobody else can: the registries and the wiring.

### 4. The two design axes — both via STRATEGY

This is the heart of the doc. There are **two independent decisions**, and each is a Strategy seam.

**Axis A — Feed generation (pull vs push).** This is the class-level shadow of the biggest lesson in the Instagram/Twitter HLDs.

- **Pull = fanout-on-read.** When the user asks for their feed, `PullStrategy` walks their `followingIds`, gathers each followee's posts *at read time*, and merges them. Cheap writes (posting is instant), expensive reads (a read touches everyone you follow).
- **Push = fanout-on-write.** When a post is created, `PushStrategy` immediately appends it to the precomputed feed list of every follower. Expensive writes (one post → thousands of inserts), cheap reads (your feed is already sitting in memory).

At the class level they are interchangeable: both implement `generate(user) → Post[]`. **This mirrors the HLD lesson exactly** — the HLD ([100](./100-hld-instagram.md)/[103](./103-hld-twitter.md)) goes deep on *when* to use each, the **celebrity/hybrid** trade-off (a celebrity with 100M followers would trigger 100M writes per post, so you *pull* their posts and *push* everyone else's), caching, and sharding. Here we just build the clean seam that makes swapping trivial. **Say this in the interview:** "the pull/push choice is the same one from the fanout HLD; at the class level it's a Strategy, so the seam is one interface."

**Axis B — Ranking (chronological vs engagement).** Swappable via Strategy (recall [42](./42-pattern-strategy.md)).

- **`ChronologicalRanking`** — sort by `timestamp` descending. Newest first. Deterministic, simple.
- **`EngagementRanking`** — compute a score per post and sort by it. A simple, defensible scoring function:

```
score = w_recency * recencyBoost(post.timestamp)
      + w_likes   * post.likes.size
      + w_comments* post.comments.length
      + w_affinity* affinity(viewer, post.author)

recencyBoost(t) = 1 / (1 + hoursSince(t))   // decays as the post ages
affinity(a, b)  = a follows b ? 1 : 0        // stand-in for a real closeness signal
```

The weights (`w_*`) are tunable constants. In production these become an ML model — but the **seam stays the same**: it's still a `RankingStrategy`. That is the payoff of Strategy: swapping `EngagementRanking` for `MLRanking` never touches `FeedService`.

### 5. Observer for real-time updates (the push fanout)

The **push** model needs a trigger: "when someone I follow posts, put it in my feed *now*." That is the **Observer pattern** (recall [41](./41-pattern-observer.md)) — a one-to-many "when X happens, tell all interested parties" relationship.

Concretely: each user's precomputed feed acts as an observer of the people they follow. When `FeedService.createPost` runs under the push strategy, it notifies every follower's feed to append the new post. The same hook naturally powers a **notification** ("Alice posted") — you just attach a second observer. Observer keeps the poster ignorant of *who* is listening: `createPost` fires an event; the fanout and the notifications react. New reaction? New observer, no edit to `createPost`.

### 6. Class diagram

```
                        ┌───────────────────────────────┐
                        │          FeedService          │
                        │  users:   Map<id, User>       │
                        │  posts:   Map<id, Post>       │
                        │  genStrategy: FeedGeneration…  │───────┐ owns
                        │  ranker:      RankingStrategy  │───┐   │
                        │  + createPost(user, content)  │   │   │
                        │  + getFeed(user, page, size)  │   │   │
                        └───────┬───────────────────────┘   │   │
                                │ manages                    │   │
                    ┌───────────┴───────────┐                │   │
                    ▼                       ▼                │   │
             ┌────────────┐          ┌────────────┐          │   │
             │    User    │  follow  │    Post    │          │   │
             │ id, name   │◀────────▶│ id, author │          │   │
             │ following  │  graph   │ content    │          │   │
             │ followers  │          │ timestamp  │          │   │
             │ +follow()  │  1    *  │ likes:Set  │          │   │
             │ +unfollow()│─────────▶│ comments[] │          │   │
             └────────────┘  authors └─────┬──────┘          │   │
                                           │ 1..*            │   │
                                           ▼                 │   │
                                     ┌───────────┐           │   │
                                     │  Comment  │           │   │
                                     └───────────┘           │   │
                                                             │   │
       Ranking axis  ◀───────────────────────────────────────┘   │
       ┌──────────────────────────┐ (abstract)                    │
       │      RankingStrategy     │  rank(posts, viewer): Post[]   │
       └────────────┬─────────────┘                                │
              ┌──────┴────────┐                                    │
              ▼               ▼                                    │
   ┌────────────────────┐ ┌────────────────────┐                  │
   │ ChronologicalRank… │ │  EngagementRanking │                  │
   └────────────────────┘ └────────────────────┘                  │
                                                                   │
       Generation axis ◀───────────────────────────────────────────┘
       ┌──────────────────────────┐ (abstract)
       │  FeedGenerationStrategy  │  generate(user, service): Post[]
       └────────────┬─────────────┘  onNewPost(post, service)  ← Observer hook
              ┌──────┴────────┐
              ▼               ▼
     ┌────────────────┐ ┌────────────────────────────┐
     │   PullStrategy │ │        PushStrategy        │
     │ gather @ read  │ │ precomputed Map<uid,Post[]>│
     └────────────────┘ │ appends on onNewPost       │
                        └────────────────────────────┘
```

Read it as: `FeedService` **owns** one `FeedGenerationStrategy` and one `RankingStrategy` (both abstract, each with two concrete subclasses). `User` holds the follow graph edges; `User` 1—* `Post`; `Post` 1—* `Comment`. The two strategy hierarchies are the two swap points.

### 7. Full JavaScript implementation

Complete and runnable — paste into `feed.js` and run `node feed.js`.

```javascript
// ============================================================
//  Enums / constants
// ============================================================
const RankingType = Object.freeze({
  CHRONOLOGICAL: 'CHRONOLOGICAL',
  ENGAGEMENT: 'ENGAGEMENT',
});

// ============================================================
//  Domain entities
// ============================================================
class User {
  constructor(id, name) {
    this.id = id;
    this.name = name;
    this.followingIds = new Set(); // people I follow  (my outgoing edges)
    this.followerIds = new Set();  // people who follow me (incoming edges)
  }

  // The User owns its own social-graph edges. We keep BOTH sides in sync
  // so push-fanout can find followers in O(1) without scanning all users.
  follow(other) {
    if (other.id === this.id) return; // can't follow yourself
    this.followingIds.add(other.id);
    other.followerIds.add(this.id);
  }

  unfollow(other) {
    this.followingIds.delete(other.id);
    other.followerIds.delete(this.id);
  }
}

class Comment {
  constructor(id, authorId, text) {
    this.id = id;
    this.authorId = authorId;
    this.text = text;
    this.timestamp = Date.now();
  }
}

class Post {
  constructor(id, authorId, content, timestamp = Date.now()) {
    this.id = id;
    this.authorId = authorId;
    this.content = content;
    this.timestamp = timestamp;
    this.likes = new Set();   // Set of userIds — a Like is too cheap to be a class
    this.comments = [];       // Comment[]
  }

  // The Post owns its own engagement data.
  addLike(userId) { this.likes.add(userId); }
  removeLike(userId) { this.likes.delete(userId); }
  addComment(comment) { this.comments.push(comment); }
}

// ============================================================
//  AXIS B — Ranking strategy (Strategy pattern, recall 42)
// ============================================================
class RankingStrategy {
  // Contract: given posts and the viewer, return a NEW ordered array.
  // Never mutate the input — callers may reuse it.
  rank(_posts, _viewer, _service) {
    throw new Error('RankingStrategy.rank() must be implemented');
  }
}

class ChronologicalRanking extends RankingStrategy {
  rank(posts, _viewer, _service) {
    return [...posts].sort((a, b) => b.timestamp - a.timestamp); // newest first
  }
}

class EngagementRanking extends RankingStrategy {
  // Tunable weights — in production these become a learned model.
  constructor(weights = {}) {
    super();
    this.w = { recency: 3, likes: 1, comments: 2, affinity: 4, ...weights };
  }

  _recencyBoost(timestamp) {
    const hours = (Date.now() - timestamp) / 3_600_000;
    return 1 / (1 + hours); // decays as the post ages
  }

  _affinity(viewer, authorId) {
    // Stand-in for a real closeness signal: do I follow the author?
    return viewer.followingIds.has(authorId) ? 1 : 0;
  }

  score(post, viewer) {
    return (
      this.w.recency  * this._recencyBoost(post.timestamp) +
      this.w.likes    * post.likes.size +
      this.w.comments * post.comments.length +
      this.w.affinity * this._affinity(viewer, post.authorId)
    );
  }

  rank(posts, viewer, _service) {
    return [...posts]
      .map((p) => ({ p, s: this.score(p, viewer) }))
      .sort((a, b) => b.s - a.s || b.p.timestamp - a.p.timestamp) // score, tie-break by time
      .map((x) => x.p);
  }
}

// ============================================================
//  AXIS A — Feed generation strategy (Strategy + Observer)
// ============================================================
class FeedGenerationStrategy {
  // Called at read time: return the RAW (unranked) candidate posts for a user.
  generate(_user, _service) {
    throw new Error('FeedGenerationStrategy.generate() must be implemented');
  }
  // Observer hook (recall 41): called when ANY post is created.
  // Pull ignores it; Push uses it to fan the post out to followers.
  onNewPost(_post, _service) { /* default: no-op */ }
}

// PULL = fanout-on-read. Cheap writes, expensive reads.
class PullStrategy extends FeedGenerationStrategy {
  generate(user, service) {
    const candidates = [];
    // Walk everyone I follow and gather their posts AT READ TIME.
    for (const authorId of user.followingIds) {
      const author = service.users.get(authorId);
      if (!author) continue;
      for (const post of service.postsByAuthor.get(authorId) ?? []) {
        candidates.push(post);
      }
    }
    return candidates; // ranking happens later, in FeedService
  }
}

// PUSH = fanout-on-write. Expensive writes, cheap reads.
class PushStrategy extends FeedGenerationStrategy {
  constructor() {
    super();
    this.precomputed = new Map(); // userId -> Post[]  (the "inbox")
  }

  _inbox(userId) {
    if (!this.precomputed.has(userId)) this.precomputed.set(userId, []);
    return this.precomputed.get(userId);
  }

  // Observer callback: when someone posts, drop a copy into every
  // follower's precomputed inbox. THIS is the fanout-on-write.
  onNewPost(post, service) {
    const author = service.users.get(post.authorId);
    if (!author) return;
    for (const followerId of author.followerIds) {
      this._inbox(followerId).push(post);
    }
    // NOTE: at HLD scale, a celebrity's followerIds could be 100M —
    // that is the celebrity problem; the fix is hybrid pull/push (see 100/103).
  }

  generate(user, _service) {
    return [...this._inbox(user.id)]; // already assembled — just hand it back
  }
}

// ============================================================
//  Feed value object (the paginated result)
// ============================================================
class Feed {
  constructor(items, page, size, total) {
    this.items = items;   // Post[] for THIS page
    this.page = page;
    this.size = size;
    this.total = total;   // total candidates before pagination
    this.hasMore = (page + 1) * size < total;
  }
}

// ============================================================
//  FeedService — the coordinator
// ============================================================
class FeedService {
  constructor(genStrategy, ranker) {
    this.users = new Map();          // id -> User
    this.posts = new Map();          // id -> Post
    this.postsByAuthor = new Map();  // authorId -> Post[]  (index for pull)
    this.genStrategy = genStrategy;  // Axis A — swappable
    this.ranker = ranker;            // Axis B — swappable
    this._postSeq = 0;
    this._commentSeq = 0;
  }

  // ---- registry helpers ----
  addUser(user) { this.users.set(user.id, user); return user; }

  // ---- swap the strategies at runtime (the whole point of the seams) ----
  setRanker(ranker) { this.ranker = ranker; }
  setGenerationStrategy(strategy) { this.genStrategy = strategy; }

  // ---- write path ----
  createPost(user, content) {
    const post = new Post(`p${++this._postSeq}`, user.id, content);
    this.posts.set(post.id, post);
    if (!this.postsByAuthor.has(user.id)) this.postsByAuthor.set(user.id, []);
    this.postsByAuthor.get(user.id).push(post);

    // Fire the Observer hook. Pull ignores it; Push fans out to followers.
    this.genStrategy.onNewPost(post, this);
    return post;
  }

  like(user, post) { post.addLike(user.id); }

  comment(user, post, text) {
    const c = new Comment(`c${++this._commentSeq}`, user.id, text);
    post.addComment(c);
    return c;
  }

  // ---- read path: generate -> rank -> paginate ----
  getFeed(user, page = 0, size = 10) {
    const candidates = this.genStrategy.generate(user, this); // Axis A
    const ranked = this.ranker.rank(candidates, user, this);  // Axis B
    // Simple OFFSET pagination. Cursor pagination is better for infinite
    // scroll (stable under inserts) — recall the API-design lesson — but
    // offset is fine for this demo.
    const start = page * size;
    const items = ranked.slice(start, start + size);
    return new Feed(items, page, size, ranked.length);
  }
}

// ============================================================
//  main() — prove generation, ranking, swappability all run
// ============================================================
function line(label) { console.log(`\n===== ${label} =====`); }
function show(feed) {
  feed.items.forEach((p, i) =>
    console.log(
      `  ${i + 1}. [${p.id}] by ${p.authorId} ` +
      `| ${p.likes.size} likes, ${p.comments.length} comments | "${p.content}"`
    )
  );
  console.log(`  (page ${feed.page}, ${feed.items.length}/${feed.total}, hasMore=${feed.hasMore})`);
}

function main() {
  // Start with PULL generation + CHRONOLOGICAL ranking.
  const service = new FeedService(new PullStrategy(), new ChronologicalRanking());

  // --- create users ---
  const alice = service.addUser(new User('u_alice', 'Alice'));
  const bob   = service.addUser(new User('u_bob', 'Bob'));
  const carol = service.addUser(new User('u_carol', 'Carol'));
  const dave  = service.addUser(new User('u_dave', 'Dave'));   // the "viewer"

  // --- follow graph: Dave follows Alice, Bob, Carol ---
  dave.follow(alice);
  dave.follow(bob);
  dave.follow(carol);

  // --- create posts (with a controlled fake clock so ordering is stable) ---
  const t0 = Date.now();
  const mk = (svc, user, content, minutesAgo) => {
    const p = svc.createPost(user, content);
    p.timestamp = t0 - minutesAgo * 60_000; // override for a deterministic demo
    return p;
  };

  const pA = mk(service, alice, 'Alice: sunset photo',        50);
  const pB = mk(service, bob,   'Bob: hot take on tabs',      30);
  const pC = mk(service, carol, 'Carol: cat video',           10);
  const pA2= mk(service, alice, 'Alice: coffee thread',        5);

  // --- engagement: make Bob's older post go viral ---
  service.like(alice, pB); service.like(carol, pB); service.like(dave, pB);
  service.comment(alice, pB, 'so true');
  service.comment(carol, pB, 'tabs > spaces');
  service.like(dave, pC); // Carol's cat gets one like

  // --- 1) CHRONOLOGICAL feed (pull) ---
  line("Dave's feed — PULL + CHRONOLOGICAL (newest first)");
  show(service.getFeed(dave, 0, 10));
  // Expect: coffee(5m) > cat(10m) > tabs(30m) > sunset(50m)

  // --- 2) swap to ENGAGEMENT ranking — order changes, no other code touched ---
  service.setRanker(new EngagementRanking());
  line("Dave's feed — PULL + ENGAGEMENT (viral post climbs)");
  show(service.getFeed(dave, 0, 10));
  // Expect: Bob's viral 'tabs' post jumps up despite being older.

  // --- 3) pagination (size 2) ---
  line("Dave's feed — ENGAGEMENT, paginated size=2");
  const p0 = service.getFeed(dave, 0, 2);
  const p1 = service.getFeed(dave, 1, 2);
  console.log('Page 0:'); show(p0);
  console.log('Page 1:'); show(p1);

  // --- 4) swap generation PULL -> PUSH, prove SAME feed ---
  //     Rebuild under push so the observer fans out existing posts.
  const pushSvc = new FeedService(new PushStrategy(), new ChronologicalRanking());
  const a = pushSvc.addUser(new User('u_alice', 'Alice'));
  const b = pushSvc.addUser(new User('u_bob', 'Bob'));
  const c = pushSvc.addUser(new User('u_carol', 'Carol'));
  const d = pushSvc.addUser(new User('u_dave', 'Dave'));
  d.follow(a); d.follow(b); d.follow(c);
  const q1 = mk(pushSvc, a, 'Alice: sunset photo',   50); // onNewPost -> Dave's inbox
  const q2 = mk(pushSvc, b, 'Bob: hot take on tabs',  30);
  const q3 = mk(pushSvc, c, 'Carol: cat video',       10);
  const q4 = mk(pushSvc, a, 'Alice: coffee thread',    5);

  line("Dave's feed — PUSH + CHRONOLOGICAL (same result as pull)");
  show(pushSvc.getFeed(d, 0, 10));
  // Same four posts, same chronological order — proving the seam is transparent.
}

main();

module.exports = {
  User, Post, Comment, Feed, FeedService,
  RankingStrategy, ChronologicalRanking, EngagementRanking,
  FeedGenerationStrategy, PullStrategy, PushStrategy, RankingType,
};
```

Running it prints Dave's feed four ways: chronological (newest first), engagement (Bob's viral post climbs above newer posts), the same feed paginated two-per-page, and finally the push-generated feed matching the pull-generated one — proving generation, ranking, and swappability all work.

---

## Visual / Diagram description

The sequence below traces one `getFeed` call, showing the two strategy seams as separate collaborators the service delegates to:

```
 Client        FeedService        genStrategy         ranker
   │                │                  │                 │
   │ getFeed(dave)  │                  │                 │
   ├───────────────▶│                  │                 │
   │                │ generate(dave)   │                 │
   │                ├─────────────────▶│  (pull: walk    │
   │                │                  │   followees;    │
   │                │   candidates[]   │   push: read    │
   │                │◀─────────────────┤   inbox)        │
   │                │       rank(candidates, dave)        │
   │                ├────────────────────────────────────▶│
   │                │                  │   ranked[]       │
   │                │◀────────────────────────────────────┤
   │                │ slice(page,size) │                 │
   │   Feed(page)   │ (paginate)       │                 │
   │◀───────────────┤                  │                 │
```

The read path is a clean three-beat rhythm: **generate → rank → paginate**. Each beat is owned by a different object, so each can change independently. Swapping `genStrategy` changes only the first beat; swapping `ranker` changes only the second.

---

## Real world examples

### Instagram (representative)

Instagram's Home feed is famously *ranked*, not chronological — the class-level equivalent of swapping in an `EngagementRanking` whose weights come from an ML model over signals like recency, relationship, and past interaction. Under the hood the generation side is a hybrid of the pull/push strategies we built, tuned per-account. The full scale story — fanout, caching, sharding — is doc [100](./100-hld-instagram.md); this class design is the shape underneath it.

### Twitter / X (representative)

Twitter historically ran a **push (fanout-on-write)** timeline: when you tweet, it's inserted into your followers' precomputed timelines — exactly our `PushStrategy.onNewPost`. But celebrities with tens of millions of followers made that write explode, so they **pull** celebrity tweets at read time and merge them in — the **hybrid** we mentioned. That's the celebrity problem, covered at scale in doc [103](./103-hld-twitter.md). At the class level it's just: use a different `FeedGenerationStrategy` for high-follower accounts.

### Reddit / Hacker News (conceptual)

Ranked feeds without a personalised follow graph: the score is a public formula over votes and age (e.g. HN's `score = (votes - 1) / (age + 2)^gravity`). This is exactly an `EngagementRanking` with no affinity term — the same seam, simpler weights — showing how one interface spans "personalised" and "global" ranking.

---

## Trade-offs

**Axis A — generation strategy:**

| | Pull (fanout-on-read) | Push (fanout-on-write) |
|---|---|---|
| Write cost | Cheap (one insert) | Expensive (one insert per follower) |
| Read cost | Expensive (gather from all followees) | Cheap (read precomputed inbox) |
| Storage | Low | High (a copy per follower) |
| Best for | Users who follow many; low read volume | Read-heavy feeds; normal follower counts |
| Breaks on | Very active readers | Celebrities (millions of followers) → hybrid |

**Axis B — ranking strategy:**

| | Chronological | Engagement |
|---|---|---|
| Predictability | Total (you know why a post is where) | Lower (score is opaque) |
| Relevance | Poor for high-volume follows | High — surfaces what you'll care about |
| Compute | Trivial sort | Score every candidate |
| Risk | Miss important posts in the noise | Filter-bubble / "why am I seeing this?" |

**The sweet spot:** design *both* axes as Strategy interfaces and pick per-context. Ship chronological for simplicity, keep the ranker seam so you can A/B an engagement or ML ranker later without touching `FeedService`; use push for normal accounts and pull for celebrities (hybrid). The one thing you must *not* do is hard-code either decision into the service — that's the mistake the whole design exists to prevent.

---

## Common interview questions on this topic

### Q1: "Chronological vs ranked — how do you support both without an `if` everywhere?"

**Hint:** Strategy pattern. Define `RankingStrategy.rank(posts, viewer)`, subclass `ChronologicalRanking` and `EngagementRanking`. `FeedService` holds a `ranker` reference and calls `this.ranker.rank(...)` — it never branches on ranking type. Swapping is one setter call. Mention Open/Closed: a new ranker is a new class, not an edit.

### Q2: "How does pull-vs-push map to your classes, and when do you switch?"

**Hint:** Both are `FeedGenerationStrategy` subclasses with a shared `generate(user)`. Pull gathers at read time; push maintains a precomputed inbox and appends via the `onNewPost` observer hook. Switch to pull (or hybrid) for celebrities because push would fan one post out to millions of inboxes — the celebrity problem. Explicitly note the deep version is the HLD ([100](./100-hld-instagram.md)/[103](./103-hld-twitter.md)).

### Q3: "Where does the Observer pattern show up?"

**Hint:** In the push model. When a post is created, `FeedService.createPost` fires `genStrategy.onNewPost(post)`, and the push strategy reacts by fanning the post into each follower's inbox. The poster doesn't know who's listening. Attach a second observer for notifications ("Alice posted") with zero changes to `createPost`.

### Q4: "Offset or cursor pagination for the feed?"

**Hint:** Cursor is better for infinite scroll — it's stable when new posts arrive between page fetches (offset pagination duplicates/skips items when the list shifts). A cursor is an opaque pointer (e.g. last-seen post id + score). Offset is fine for a demo. Show you know *why* cursor wins for feeds specifically.

### Q5: "Your `EngagementRanking` — how would you evolve it to ML?"

**Hint:** The scoring function is already isolated behind `RankingStrategy`. Replace `EngagementRanking` with `MLRanking` whose `rank()` calls a model returning a score per post; feed the same signals (recency, likes, comments, affinity) plus more as features. `FeedService` doesn't change — that's the seam paying off.

---

## Practice exercise

**Build an `MLRankingStub` and an ad-injector — prove the seams hold.**

Starting from the code above (~30 min):

1. Write a new `class MLRankingStub extends RankingStrategy` whose `rank()` returns posts sorted by a "model score" — for the stub, just `Math.random()` seeded per post id (deterministic), or `likes*2 + comments*3 - ageHours`. Swap it in with `service.setRanker(new MLRankingStub())` and confirm the order changes and **you edited zero existing classes.**
2. Add a `class AdInjectingRanking extends RankingStrategy` that *wraps* another ranker (Decorator-flavoured): it calls the inner ranker, then splices a fake ad `Post` into every 3rd slot. Verify ads appear in the paginated output.
3. Bonus: add a `seenIds: Set` to `User` and filter seen posts out inside `getFeed` before ranking; re-fetch and confirm previously-seen posts drop out.

Produce: the two new classes and console output showing (a) ML order differs from chronological, and (b) ads at positions 3, 6, 9. Success = `FeedService`, `User`, and `Post` were never edited.

---

## Quick reference cheat sheet

- **Two axes, two Strategies.** Feed *generation* (how posts are gathered) and *ranking* (how they're ordered) are independent, swappable seams. This is the whole lesson.
- **`FeedGenerationStrategy`** — `PullStrategy` (fanout-on-read: gather at read time) vs `PushStrategy` (fanout-on-write: precomputed inbox, appended via observer).
- **`RankingStrategy`** — `ChronologicalRanking` (time desc) vs `EngagementRanking` (score = recency + likes + comments + affinity).
- **`FeedService` is a coordinator**, not a god-object: it owns registries + the two chosen strategies and does generate → rank → paginate.
- **`User` owns the social graph** — `followingIds` / `followerIds` Sets, kept in sync by `follow`/`unfollow`.
- **`Post` owns engagement** — `likes` Set + `comments` array; a Like is a Set entry, too cheap to be a class.
- **Observer** powers the push fanout: `createPost` fires `onNewPost`; followers' inboxes react; notifications can piggyback.
- **Read path rhythm:** generate → rank → paginate, three beats, three owners, each swappable alone.
- **Celebrity problem** = push fans one post to millions of inboxes → use hybrid (pull celebrities, push everyone else). Depth lives in the HLD.
- **Pagination:** offset is fine for demos; **cursor** is better for infinite scroll (stable under inserts).
- **Open/Closed win:** a new ranker or generator is a NEW class; `FeedService` never changes.
- **This is the LLD.** Fanout-at-scale, caching, sharding, celebrity hybrid = the HLD ([100](./100-hld-instagram.md)/[103](./103-hld-twitter.md)).
- **Interview tell:** naming pull/push as "the same choice from the fanout HLD, here as a Strategy" signals you understand both altitudes.

---

## Design patterns used and WHY

| Pattern | Where | Why it's the right tool |
|---|---|---|
| **Strategy** (recall [42](./42-pattern-strategy.md)) | `RankingStrategy` (chronological/engagement) AND `FeedGenerationStrategy` (pull/push) | Two independent behaviours that must be swapped at runtime without touching the caller. Strategy makes each a self-contained object; `FeedService` depends on the interface, so new behaviours are new classes (Open/Closed). These two seams are the stars of this design. |
| **Observer** (recall [41](./41-pattern-observer.md)) | `onNewPost` hook — push fanout + notifications | A one-to-many "when a post is created, tell everyone interested" relationship. The poster stays ignorant of who reacts; fanout and notifications attach independently. |

**Why Strategy twice?** Because there are genuinely two orthogonal decisions. Some designs collapse them ("chronological push", "ranked pull") into an enum with a giant switch — that's the anti-pattern. Keeping them as two interfaces means the 2×2 grid of combinations costs zero extra code: any generator composes with any ranker.

**Explicit tie to the HLDs.** This class design is the *engine*; the Instagram and Twitter HLDs are the *highway*. The HLD ([100](./100-hld-instagram.md)/[103](./103-hld-twitter.md)) takes the exact same pull/push distinction and answers the questions we deliberately skipped: fanout **at scale** (queues, workers), the **celebrity problem** (hybrid fanout), **caching** the precomputed feed in Redis, and **sharding** the timeline store. Same concept, different altitude — the LLD/HLD split of doc [02](./02-hld-vs-lld.md).

---

## Extensions the interviewer asks for

**"Now rank by an ML model."** Add `class MLRanking extends RankingStrategy` whose `rank()` calls a model returning a score per post; feed it the same features (recency, likes, comments, affinity) plus more. `service.setRanker(new MLRanking())`. `FeedService` is untouched — the Strategy seam absorbs it entirely.

**"Switch pull ↔ push, or go hybrid (celebrity problem)."** Swap `genStrategy`, or write a `HybridStrategy` that holds *both*: on read it returns the follower's precomputed push-inbox merged with freshly-pulled posts from any followee flagged `isCelebrity`. One new `FeedGenerationStrategy`; callers unchanged. Deep version → [100](./100-hld-instagram.md)/[103](./103-hld-twitter.md).

**"Add cursor pagination."** Replace the `slice(page*size, ...)` with a cursor: `getFeed(user, cursor, size)` returns items *after* the cursor (last-seen post id + score) plus a `nextCursor`. Only `getFeed` changes; entities and strategies are untouched. Cursor is stable when new posts arrive mid-scroll (recall the API-design lesson).

**"Add a 'seen' state / dedup."** Give `User` a `seenIds: Set`. In `getFeed`, filter candidates (`!user.seenIds.has(p.id)`) before ranking, and mark returned items seen. Dedup naturally falls out because posts are keyed by id. This is a one-method change inside the service; no strategy is affected.

**"Inject ads / promoted content."** Wrap the ranker in an `AdInjectingRanking` that calls the real ranker, then splices ad `Post`s into fixed slots (Decorator over Strategy). Or make ads first-class candidates the generator emits with a special author. Either way the injection point is a single, isolated class — the feed pipeline absorbs it without the entities knowing ads exist.

Every extension lands as **a new class or a one-method tweak**, never a rewrite. That is the proof the two Strategy seams and the Observer hook were the right skeleton.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → behaviours → classes → code → patterns → extensions) this doc follows. |
| **Next** | [100 — HLD: Instagram](./100-hld-instagram.md) | The scale story: same pull/push distinction, plus fanout-at-scale, celebrity hybrid, caching, sharding. |
| **Related** | [103 — HLD: Twitter](./103-hld-twitter.md) | The push-timeline + celebrity-pull hybrid at scale — the HLD depth behind our `FeedGenerationStrategy`. |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The pattern behind BOTH axes (generation + ranking) — the star of this design. |
| **Related** | [41 — Observer Pattern](./41-pattern-observer.md) | Powers the push-model fanout and notifications via the `onNewPost` hook. |
