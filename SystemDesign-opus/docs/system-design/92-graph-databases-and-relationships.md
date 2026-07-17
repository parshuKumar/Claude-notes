# 92 — Graph Databases and Relationship-Heavy Systems
## Category: HLD Components

---

## What is this?

A **graph database** is a database where the *connections between things* are stored as first-class data, not derived at query time. Instead of tables and foreign keys, you store **nodes** (the things: people, products, devices) and **edges** (the relationships: `FRIEND_OF`, `BOUGHT`, `LOGGED_IN_FROM`) — and each node keeps a direct pointer to its neighbours.

Think of the difference between a **phone book** and a **social gathering**. A phone book (relational table) is great at "give me Priya's number" — one lookup, done. But ask it "who does Priya know, who do *they* know, and who do *they* know?" and you're flipping pages thousands of times. At an actual gathering, Priya just turns and points: "that's my friend Raj." One hop, zero searching. A graph database is the gathering.

---

## Why does it matter?

Because there's a class of question that relational databases answer *catastrophically* badly, and it's a class of question that modern products ask constantly:

- "Who are my friends' friends' friends?" (LinkedIn's 3rd-degree connections)
- "Which accounts share a device fingerprint with a known fraudster?" (fraud rings)
- "People who bought this also bought…" (recommendations)
- "If this switch dies, which services go down?" (impact analysis)

Every one of these is a **traversal** — a walk across relationships of variable depth. SQL was designed around *set operations on rows*, and a traversal in SQL becomes a chain of JOINs whose cost explodes exponentially with depth. We'll do the arithmetic below and it is genuinely ugly.

**In interviews:** Graph databases come up in "design LinkedIn," "design a fraud detection system," "design a recommendation engine." The mark of a strong candidate is *not* shouting "use Neo4j!" It's (a) explaining **index-free adjacency** precisely, (b) knowing exactly when a graph DB is the wrong tool, and (c) knowing that Facebook and Twitter did *not* use one — they built custom caching layers over MySQL. That last point wins interviews.

**At work:** You will rarely start with a graph DB. You'll start with Postgres, and one day a product manager asks for "mutual connections within 3 degrees," your query takes 40 seconds, and now you have a real decision to make.

---

## The core idea — explained simply

### The Rolodex vs. The Party Analogy

**The Rolodex (a relational database).**
You have a box of index cards. Each card has a person's name and, on the back, a list of ID numbers of their friends. The cards are sorted alphabetically.

To find Priya's friends: find Priya's card (fast, it's sorted), read 200 friend IDs off the back, then go find 200 more cards. That's 200 lookups.

To find friends-of-friends: for each of those 200 cards, read *their* 200 friend IDs, then look up 40,000 cards. To go one level deeper: 8,000,000 lookups.

Notice what's happening. **Every hop requires you to search the whole box again**, because the card doesn't tell you *where* the friend's card is — it only tells you *who* the friend is. You must re-search by name every single time. That "re-search" is a B-tree index lookup, and it costs O(log n) where n is the *total number of people in the database*.

**The Party (a native graph database).**
Now the same 200 friends are standing in a room, and Priya is physically holding a piece of string tied to each of her friends' wrists. To find her friends, she doesn't search anything — she pulls the strings. Each friend is holding *their* strings. Follow those. Follow those again.

**Nobody ever searched.** You only ever *followed a pointer*. The cost of a hop is O(1), and the total cost of the query is proportional to **how much of the graph you actually touched** — not to how big the database is.

### Mapping the analogy back

| Analogy | Technical concept |
|---|---|
| Index card | A row in a table |
| Name on the card | Primary key |
| Friend ID written on the back | Foreign key |
| Searching the box by name | B-tree index lookup, O(log n) |
| Person at the party | Node (vertex) |
| String tied between wrists | Edge (relationship), stored with a physical pointer |
| Pulling a string | **Index-free adjacency** — pointer dereference, O(1) |
| "Who's holding this string, and what colour is it?" | Edge type + direction + properties |
| Searching the whole box on *every* hop | Why SQL traversals blow up exponentially |

The single sentence to remember:

> **In a relational database, a relationship is computed at read time by searching an index. In a native graph database, a relationship is a stored physical pointer, so it is simply followed.**

---

## Key concepts inside this topic

### 1. The data model — nodes, edges, properties

A **property graph** has exactly three ingredients:

- **Node** — an entity. Has one or more **labels** (`:User`, `:Product`) and a bag of **properties** (`{name: "Priya", age: 29}`).
- **Edge** (relationship) — connects two nodes. Critically, an edge has:
  - a **type** (`FRIEND`, `BOUGHT`, `REPORTS_TO`) — its own semantic identity,
  - a **direction** (Priya → Raj is not the same as Raj → Priya),
  - and its **own properties** (`{since: "2021-03-04", weight: 0.8}`).
- **Properties** live on both nodes and edges.

**This is the key difference from a foreign key.** A foreign key is a dumb integer sitting in a column. It has no type of its own, it can't carry attributes, and if you want a *second* kind of relationship between the same two tables you need a whole new column or a whole new join table. In a graph, an edge is a *thing*. You can query edges, filter on their properties, and have twenty different edge types between the same two nodes.

A small social graph:

```
                  {since: 2019}
      ┌────────┐  ─────FRIEND────▶  ┌────────┐
      │ Priya  │                    │  Raj   │
      │ :User  │  ◀────FRIEND─────  │ :User  │
      │ age:29 │      {since:2019}  │ age:31 │
      └───┬────┘                    └───┬────┘
          │                             │
       FRIEND                        WORKS_AT
     {since:2022}                  {role:"SRE"}
          │                             │
          ▼                             ▼
      ┌────────┐   BOUGHT           ┌──────────┐
      │  Meera │ ──{qty:2}────────▶ │  Acme    │
      │ :User  │                    │ :Company │
      └────┬───┘                    └──────────┘
           │
        BOUGHT {qty:1}
           │
           ▼
      ┌──────────┐
      │  Laptop  │
      │ :Product │
      └──────────┘
```

Read it out loud: *Priya is friends with Raj since 2019 (mutual, so two directed edges). Priya is friends with Meera since 2022. Raj works at Acme as an SRE. Meera bought a Laptop.* The graph is the sentence.

---

### 2. Why relational databases fall apart on deep relationships — with the math

Here is the schema everyone starts with:

```sql
CREATE TABLE users (
  id    BIGINT PRIMARY KEY,
  name  TEXT
);

CREATE TABLE friendships (
  user_id   BIGINT REFERENCES users(id),
  friend_id BIGINT REFERENCES users(id),
  since     DATE,
  PRIMARY KEY (user_id, friend_id)
);
CREATE INDEX idx_friend_lookup ON friendships(user_id);
```

Perfectly reasonable. Now the product asks for **friends-of-friends-of-friends** — 3 hops — for user 42. In SQL that is a self-join of `friendships` against itself, three times:

```sql
-- "Friends of friends of friends of user 42"
-- Read this and feel the pain. This is the whole argument.
SELECT DISTINCT u4.id, u4.name
FROM friendships f1
JOIN friendships f2 ON f2.user_id = f1.friend_id     -- hop 2
JOIN friendships f3 ON f3.user_id = f2.friend_id     -- hop 3
JOIN users u4      ON u4.id       = f3.friend_id
WHERE f1.user_id = 42
  AND f3.friend_id <> 42                -- don't return myself
  AND f3.friend_id NOT IN (             -- exclude direct friends
        SELECT friend_id FROM friendships WHERE user_id = 42)
  AND f3.friend_id NOT IN (             -- exclude 2nd degree
        SELECT f2b.friend_id
        FROM friendships f1b
        JOIN friendships f2b ON f2b.user_id = f1b.friend_id
        WHERE f1b.user_id = 42);
```

Now the arithmetic. Take the well-known average of **200 friends per user**:

| Hop | Intermediate rows the engine must materialise | Index lookups performed |
|---|---|---|
| 1 | 200 | 1 |
| 2 | 200 × 200 = **40,000** | 200 |
| 3 | 200 × 200 × 200 = **8,000,000** | 40,000 |
| 4 | **1,600,000,000** | 8,000,000 |

Each of those index lookups is a B-tree descent costing roughly `O(log n)` where **n is the size of the whole `friendships` table**. If you have 100M users × 200 friends = 20 **billion** rows, `log₂(2×10¹⁰) ≈ 34` — so ~34 node visits per lookup, most of them cache misses on disk.

So hop 3 costs roughly **8,000,000 rows × 34 comparisons ≈ 270 million operations**, and then the engine still has to `DISTINCT` (i.e. sort or hash) 8 million rows. On real hardware that's tens of seconds to minutes. Hop 4 is simply not going to finish.

Two things are lethal here:
1. **The branching factor compounds.** Cost is `O(k^d)` where k = average degree, d = depth. That's exponential in depth. It doesn't matter how good your index is.
2. **The cost depends on the total dataset size.** Every hop re-enters the index, so growing from 10M to 100M users makes *every single lookup* slower.

### 3. Index-free adjacency — the actual value proposition

A native graph database stores, physically alongside each node, a **linked list of its adjacent edges**, each edge holding the direct **on-disk (or in-memory) address** of the node at the other end.

```
  Node record (fixed size, so node N lives at offset N × size — O(1) to find)
  ┌──────────────────────────────────────────────┐
  │ Node #42 "Priya" │ firstEdgePtr ──────────┐  │
  └──────────────────────────────────────────────┘
                                              │
                                              ▼
  Edge chain (a doubly linked list of edge records)
  ┌────────────────────────────────┐  ┌────────────────────────────────┐
  │ type: FRIEND                   │  │ type: FRIEND                   │
  │ from: 42   to: 77  ────────────┼─▶│ from: 42   to: 91  ────────────┼─▶ …
  │ props: {since: 2019}           │  │ props: {since: 2022}           │
  │ nextEdgeForNode42 ─────────────┘  │ nextEdgeForNode42 ─────────────┘
  └────────────────────────────────┘  └────────────────────────────────┘
             │ "to: 77" is a direct address, not a key to look up
             ▼
  ┌──────────────────────────────────────────────┐
  │ Node #77 "Raj"   │ firstEdgePtr ─────────▶…  │
  └──────────────────────────────────────────────┘
```

Traversing one edge = **dereference a pointer**. That's it. No index. No search. **O(1) per edge.**

The consequence is the whole reason graph databases exist:

> **Query cost is proportional to the size of the RESULT — the portion of the graph you actually walk — not to the size of the DATASET.**

Say that plainly in an interview and you're most of the way there. Concretely: a 5-hop traversal touching ~50,000 nodes costs about the same on a 10-million-user graph as on a 100-million-user graph, because you're touching the same 50,000 nodes either way. In SQL, going from 10M to 100M users makes every one of your millions of index lookups measurably slower.

Only the **starting node** requires an index lookup (to turn "Priya" into node #42). After that, you're free.

Here's the mental model as runnable JS — an in-memory adjacency structure is exactly what a graph DB does on disk:

```js
// A node holds DIRECT REFERENCES to its edges. No lookup table involved.
class GraphNode {
  constructor(id, labels = [], props = {}) {
    this.id = id;
    this.labels = labels;
    this.props = props;
    this.outEdges = [];  // <-- index-free adjacency lives here
    this.inEdges = [];
  }
}

class Edge {
  // An edge is a first-class object: it has a TYPE, a DIRECTION, and PROPERTIES.
  constructor(type, from, to, props = {}) {
    this.type = type;
    this.from = from;   // reference to a GraphNode, not an id we must look up
    this.to = to;
    this.props = props;
  }
}

class Graph {
  constructor() { this.nodes = new Map(); }

  addNode(id, labels, props) {
    const n = new GraphNode(id, labels, props);
    this.nodes.set(id, n);
    return n;
  }

  addEdge(type, fromId, toId, props) {
    const from = this.nodes.get(fromId);
    const to = this.nodes.get(toId);
    const e = new Edge(type, from, to, props);
    from.outEdges.push(e);  // the pointer is written ONCE, at write time
    to.inEdges.push(e);     // read time never pays for it again
    return e;
  }

  // Breadth-first traversal to exactly `depth` hops along a given edge type.
  // Cost is O(nodes touched + edges touched) — NOT O(total graph size).
  traverse(startId, edgeType, depth) {
    let frontier = [this.nodes.get(startId)];
    const seen = new Set([startId]);      // dedupe as we go — no giant DISTINCT at the end
    for (let hop = 0; hop < depth; hop++) {
      const next = [];
      for (const node of frontier) {
        for (const e of node.outEdges) {  // pointer walk: O(1) per edge
          if (e.type !== edgeType) continue;
          if (seen.has(e.to.id)) continue;
          seen.add(e.to.id);
          next.push(e.to);
        }
      }
      frontier = next;
    }
    return frontier;
  }
}
```

Notice `traverse` never consults an index after the first `this.nodes.get(startId)`. That's the entire trick.

---

### 4. The same query in Cypher

**Cypher** is Neo4j's query language (also standardised as openCypher, and the basis of the new ISO **GQL** standard). It is ASCII-art: `()` is a node, `-[]->` is an edge.

```cypher
// Friends-of-friends-of-friends of Priya. Compare to the 20-line SQL above.
MATCH (me:User {name: 'Priya'})-[:FRIEND*3]-(fof:User)
WHERE fof <> me
  AND NOT (me)-[:FRIEND]-(fof)       // exclude direct friends
RETURN DISTINCT fof.name;
```

`[:FRIEND*3]` means "follow exactly 3 FRIEND edges." Want variable depth? `[:FRIEND*1..6]`. Want *unbounded*? `[:FRIEND*]`. Try expressing "between 1 and 6 hops, I don't know how many" in SQL — you can't, without a recursive CTE or generating N different queries.

The readability contrast is not a cosmetic point. It's evidence that **the query language matches the shape of the problem**. When your tool fights the shape of your problem, you're using the wrong tool.

Driving it from Node:

```js
import neo4j from 'neo4j-driver';

const driver = neo4j.driver(
  'neo4j://localhost:7687',
  neo4j.auth.basic('neo4j', process.env.NEO4J_PASSWORD)
);

async function friendsOfFriendsOfFriends(name) {
  // Sessions are cheap; the driver pools the underlying connections.
  const session = driver.session({ defaultAccessMode: neo4j.session.READ });
  try {
    const result = await session.run(
      `MATCH (me:User {name: $name})-[:FRIEND*3]-(fof:User)
       WHERE fof <> me AND NOT (me)-[:FRIEND]-(fof)
       RETURN DISTINCT fof.name AS name
       LIMIT 100`,
      { name }                     // always parameterise — enables plan caching too
    );
    return result.records.map(r => r.get('name'));
  } finally {
    await session.close();
  }
}
```

---

### 5. When to reach for a graph database — and the queries that justify it

The honest rule, three conditions. Ideally you want **all three**:

1. **Relationships are the primary thing you query** — not an incidental join, but the actual subject of the question.
2. **Traversal depth is variable or unbounded** — "however many hops it takes."
3. **Relationship types are heterogeneous** — many different kinds of edge between the same entities.

**Social networks** — degrees of separation, mutual connections, "people you may know."

```cypher
// Shortest path between two people — LinkedIn's "3rd degree connection" badge
MATCH p = shortestPath((a:User {id: 42})-[:FRIEND*..6]-(b:User {id: 9001}))
RETURN length(p) AS degreesOfSeparation;

// "People you may know" = friends-of-friends ranked by mutual-friend count
MATCH (me:User {id: 42})-[:FRIEND]-(mutual)-[:FRIEND]-(candidate)
WHERE candidate <> me AND NOT (me)-[:FRIEND]-(candidate)
RETURN candidate.name, count(DISTINCT mutual) AS mutuals
ORDER BY mutuals DESC LIMIT 20;
```

**Recommendations** — "users who bought X also bought Y" is literally a 2-hop traversal:

```cypher
MATCH (p:Product {sku: 'X'})<-[:BOUGHT]-(:User)-[:BOUGHT]->(other:Product)
WHERE other <> p
RETURN other.name, count(*) AS coPurchases
ORDER BY coPurchases DESC LIMIT 10;
```

**Fraud detection — the killer app.** This is worth dwelling on. Fraud rings work by sharing infrastructure: five "different" people who all logged in from one device, or share a shipping address, or funnel money through one card. **The fraud IS the shape of the subgraph.** A dense cluster of accounts sharing identifiers is not something you can express as a WHERE clause — it's a *topology*.

```cypher
// Find rings: accounts that transitively share a device, card, or address
MATCH (a:Account)-[:USED_DEVICE|USED_CARD|SHIPS_TO]->(shared)
        <-[:USED_DEVICE|USED_CARD|SHIPS_TO]-(b:Account)
WHERE a.id < b.id                      // dedupe the symmetric pair
WITH shared, collect(DISTINCT a) + collect(DISTINCT b) AS ring
WHERE size(ring) >= 4                  // 4+ accounts on one identifier = suspicious
RETURN labels(shared)[0] AS sharedType, shared.value, size(ring) AS ringSize
ORDER BY ringSize DESC;
```

Try that relationally and you'll be writing self-joins across three link tables with an unknown number of levels. It's nearly impossible to spot, and that asymmetry is precisely why banks and payment processors run graph databases next to their transactional store.

**Other genuine fits:**
- **Knowledge graphs** — Google's entity graph, Wikidata: heterogeneous entities, heterogeneous edges.
- **Network topology & impact analysis** — "if this router fails, which services lose connectivity?" is a reachability query.
- **Identity / permission hierarchies** — "does user U have read access to resource R through *any* chain of group memberships and role inheritances?" Depth unknown. (This is essentially Google's Zanzibar problem.)
- **Supply chains** — "which of our finished products contain a component from this recalled supplier, at any tier?"

---

### 6. When NOT to use one — be firm

This is where candidates over-reach, and interviewers pounce. Graph databases are **bad** at:

| Workload | Why the graph DB loses | Use instead |
|---|---|---|
| Whole-dataset aggregation ("total revenue last quarter") | Must walk millions of nodes; no columnar storage, no vectorised scans | Data warehouse (BigQuery, Snowflake, Redshift) |
| Simple CRUD on independent entities | You're paying for a relationship engine you never use | Postgres / MySQL |
| High-volume transactional writes | Maintaining adjacency pointers + ACID on a connected structure is expensive; write throughput is far below a tuned RDBMS or Cassandra | Postgres, Cassandra, DynamoDB |
| Full-text search | Graph engines have weak text indexing | Elasticsearch / OpenSearch (see `79-search-systems.md`) |
| Massive fan-out reads at web scale | General-purpose traversal machinery is overkill and slow for shallow, hot queries | A purpose-built cached index (see §8) |

**And the big one: graphs are hard to shard.**

Sharding works when your data partitions cleanly — user 1–1M on shard A, 1M–2M on shard B. But **a graph resists partitioning precisely BECAUSE everything is connected.** Every edge you cut by placing its two endpoints on different machines becomes a **network hop** during traversal. A 3-hop query that crosses shards twice per hop turns your O(1) pointer dereference into a 0.5ms RPC — and you've thrown away the entire value proposition.

Finding a partition that minimises cut edges is **graph partitioning**, which is NP-hard, and — worse — social graphs are *specifically* the hardest case because they're small-world networks with no natural cut. This is a genuinely open problem. Distributed graph databases exist, but they are the least mature part of the ecosystem, and most successful graph deployments run on a **single large machine with the graph in RAM** (which is fine — a 10B-edge graph fits in a few hundred GB).

Add to that: smaller operator community, fewer managed offerings, fewer engineers who know Cypher, weaker tooling.

**The default answer is still Postgres.**

Most "graph" problems at modest scale are perfectly fine with a well-indexed relational table. And Postgres can do bounded traversal natively with a **recursive CTE** (`WITH RECURSIVE` — a query that references itself, repeatedly feeding its own output back in until nothing new comes out):

```sql
-- 3-degree network of user 42, in plain Postgres. No new database required.
WITH RECURSIVE network AS (
  -- Base case: me, at depth 0
  SELECT 42::BIGINT AS id, 0 AS depth

  UNION       -- UNION (not UNION ALL) dedupes, which prevents cycling forever

  -- Recursive case: everyone one hop from someone already in `network`
  SELECT f.friend_id, n.depth + 1
  FROM network n
  JOIN friendships f ON f.user_id = n.id
  WHERE n.depth < 3                          -- ALWAYS bound the depth
)
SELECT u.id, u.name, MIN(n.depth) AS degree
FROM network n
JOIN users u ON u.id = n.id
WHERE n.depth > 0
GROUP BY u.id, u.name
ORDER BY degree;
```

This is genuinely good. It dedupes as it goes (so you don't materialise 8M duplicate rows), it's ACID, it runs on the database you already operate, and for a graph of a few million edges it returns in tens of milliseconds. **Reach for this before you reach for Neo4j.** You move to a graph DB when this stops being fast enough *and* traversal is central to your product — not before.

---

### 7. Graph algorithms you should be able to name

| Algorithm | One line | Used for |
|---|---|---|
| **Shortest path (BFS / Dijkstra)** | Fewest hops (BFS) or lowest total edge weight (Dijkstra) between two nodes | Degrees of separation, routing, "how do I know this person?" |
| **PageRank** | A node is important if important nodes point at it — computed by iterating until it converges | Influence ranking, search ranking, spotting hub accounts |
| **Community detection (Louvain, label propagation)** | Find clusters that are densely connected inside and sparsely connected outside | Friend groups, market segments, fraud rings |
| **Centrality (degree, betweenness, closeness)** | Which nodes are hubs (degree) or sit on the most paths between others (betweenness) | Key influencers, single points of failure in a network |
| **Triangle counting / clustering coefficient** | How often do my neighbours also know each other? | Measuring community tightness, link prediction |

### 8. Query languages and graph flavours

- **Cypher / openCypher** — declarative, ASCII-art patterns, by far the easiest to read. Neo4j, Memgraph, AWS Neptune. Being folded into the ISO **GQL** standard.
- **Gremlin** — imperative traversal steps you chain: `g.V().has('name','Priya').out('FRIEND').out('FRIEND').dedup()`. Apache TinkerPop; runs on JanusGraph, Neptune, Cosmos DB. More powerful, less readable.
- **SQL/PGQ** — "Property Graph Queries," added in **SQL:2023**. Lets you write graph pattern matching *inside* SQL over your existing tables. If it lands well in Postgres, it removes much of the reason to adopt a separate graph DB for mid-scale problems. Watch it.

**Property graph vs RDF/triple store.** A *property graph* (Neo4j) puts rich properties on nodes and edges. An *RDF triple store* (Apache Jena, Blazegraph) stores everything as `(subject, predicate, object)` triples — `(Priya, knows, Raj)` — queried with **SPARQL**. RDF is built for open, federated, standards-based data sharing (linked open data, life sciences, government ontologies) and supports formal reasoning. Property graphs are built for application engineering. **For interviews: property graph unless someone says "ontology," "semantic web," or "linked data."**

---

## Visual / Diagram description

**Diagram 1 — Relational join blow-up vs. graph pointer walk.**

```
  RELATIONAL: 3-hop friends-of-friends-of-friends (avg 200 friends)

   user 42 ──▶ [B-tree index lookup] ──▶ 200 rows
                                           │
                 for EACH of 200 rows: another index lookup (O(log n))
                                           ▼
                                       40,000 rows
                                           │
              for EACH of 40,000 rows: another index lookup (O(log n))
                                           ▼
                                    8,000,000 rows  ──▶ DISTINCT (sort 8M) ──▶ 💀
                                     
   Cost = O(k^d · log n)      k = degree, d = depth, n = TOTAL TABLE SIZE
                              ▲                              ▲
                              exponential in depth     grows with dataset


  GRAPH: same query

   [index lookup ONCE] ──▶ ●42
                            ├─▶ ●77 ─┬─▶ ●301 ──▶ ●950
                            ├─▶ ●91  ├─▶ ●188 ──▶ ●412
                            └─▶ ●55  └─▶ ●207 ──▶ ●776
                            └─ every arrow is a POINTER DEREFERENCE, O(1) ─┘

   Cost = O(nodes touched + edges touched)     ← proportional to the RESULT
                                               ← INDEPENDENT of dataset size
```

The top half shows the fan-out: each level multiplies by the branching factor *and* each row costs a fresh index descent. The bottom half shows that a graph pays an index cost exactly once — to find the entry point — and then everything is pointer-following. **If you draw one thing on a whiteboard, draw this.**

**Diagram 2 — A fraud ring: the fraud is the shape.**

```
     ┌──────────┐                                    ┌──────────┐
     │ Account  │───USED_DEVICE──┐   ┌───USED_DEVICE─│ Account  │
     │  A-1001  │                │   │               │  A-1002  │
     └────┬─────┘                ▼   ▼               └────┬─────┘
          │                  ┌───────────────┐            │
       SHIPS_TO              │ Device        │         USED_CARD
          │                  │ fp:9f2a...    │            │
          ▼                  └───────▲───────┘            ▼
     ┌───────────────┐               │             ┌──────────────┐
     │ Address       │◀──SHIPS_TO────┤             │ Card         │
     │ 12 Oak St     │               │             │ ****4417     │
     └───────▲───────┘          USED_DEVICE        └──────▲───────┘
             │                       │                    │
             │                  ┌────┴─────┐              │
             └────SHIPS_TO──────│ Account  │──USED_CARD───┘
                                │  A-1003  │
                                └──────────┘

   4 "unrelated" accounts. Individually: clean. As a subgraph: a ring.
```

Each account, examined in isolation with a row-oriented query, looks fine. The signal is *only* visible in the topology — a dense cluster collapsing onto three shared identifiers. That's the argument for graphs in fraud, in one picture.

---

## Real world examples

### Facebook — TAO (over MySQL), not a graph database

Facebook's social graph is served by **TAO** ("The Associations and Objects"), described in their 2013 USENIX paper. Crucially, TAO is **not an off-the-shelf graph database**. It is a custom, geographically distributed, read-optimised **graph-serving layer** built *on top of sharded MySQL*, with an enormous cache tier in front.

The model is deliberately tiny: **objects** (typed nodes with a key-value property bag) and **associations** (typed, directed edges with a timestamp). The API is correspondingly tiny — fetch an object, fetch an association list, count associations, range-scan an association list by time.

The point: TAO serves *billions* of reads per second, and it does so by only supporting **very shallow queries** — usually one or two hops ("get this post's comments," "is A friends with B?"). It gives up general traversal entirely in exchange for a read path that is essentially a cache hit. Eventual consistency across regions is accepted as the price.

### Twitter — FlockDB

Twitter open-sourced **FlockDB**, its store for the follow graph. Again: **not a general graph database.** It is a sharded, replicated **adjacency-list store** over MySQL, tuned for exactly one thing — enormous adjacency lists (an account with 50M followers), very high write rates, and *shallow* set operations (intersect two follower lists, page through followers).

FlockDB explicitly does **not** do multi-hop traversal. Twitter didn't need it: the follow graph is queried at depth 1 (who do I follow?) and occasionally depth 2. Building a traversal engine would have cost them the read throughput they actually needed.

### Neo4j in fraud detection and identity

Financial institutions and identity providers are the clearest commercial graph-DB fit: moderate data volumes (millions to low billions of edges — fits on one big machine), but **deep, exploratory, variable-depth** queries where the analyst genuinely doesn't know how many hops the ring extends. This is exactly the workload TAO and FlockDB *can't* do and don't try to.

**The lesson — say this in the interview:**

> At **extreme scale with shallow queries**, a purpose-built, heavily cached, sharded index over a boring database beats a general-purpose graph DB. Graph databases shine for **deep, exploratory, variable-depth traversals at moderate scale.** Match the tool to the *depth* of the query, not to the word "graph" in the problem statement.

---

## Trade-offs

**Graph database vs. relational**

| | Native graph DB | Relational (Postgres) |
|---|---|---|
| Deep traversal (3+ hops) | O(result size) — excellent | O(k^d · log n) — exponential blow-up |
| Variable/unbounded depth | Native (`[:FRIEND*1..6]`) | Recursive CTE, awkward, must bound |
| Relationship as a typed, propertied object | Yes, first-class | No — join table, extra columns |
| Whole-dataset aggregation | Poor | Good (great in a warehouse) |
| Write throughput | Lower | High |
| Sharding | Genuinely hard (NP-hard partitioning, cut edges = RPCs) | Well-understood |
| Full-text search | Weak | Weak — use Elasticsearch either way |
| Ecosystem maturity, hiring pool, managed options | Thinner | Enormous |
| Schema flexibility | High (add an edge type anytime) | Migrations |

**Cost of adopting a graph DB**

| You gain | You give up |
|---|---|
| Traversals that are fast *and* readable | A second database to operate, back up, monitor |
| Queries whose cost tracks the result, not the dataset | Your team's SQL fluency |
| Natural modelling of heterogeneous relationships | Easy horizontal scaling |
| Built-in algorithms (PageRank, Louvain, shortest path) | Strong analytics and reporting |
| Fraud/topology patterns you literally cannot express in SQL | Simplicity — now you're syncing two stores |

**Rule of thumb:** Start in Postgres. Add indexes. Try a `WITH RECURSIVE` CTE and *measure it*. Only introduce a graph database when (a) traversal depth is genuinely variable, (b) it's central to the product rather than one report, and (c) you've hit a measured wall. And if your traversals are **shallow but the scale is colossal**, don't reach for a graph DB at all — build a cached adjacency index (the TAO/FlockDB answer).

**The sweet spot:** Deep, exploratory, variable-depth queries over a graph of millions-to-billions of edges that fits on one large machine — fraud rings, recommendations, permission chains, knowledge graphs.

---

## Common interview questions on this topic

### Q1: "Why is a graph database faster than SQL for a friends-of-friends query?"

**Hint:** Name **index-free adjacency** explicitly. In SQL, each hop is a JOIN requiring a fresh B-tree lookup per intermediate row — cost `O(k^d · log n)`, exponential in depth and dependent on total table size. In a native graph DB, each node stores physical pointers to its neighbours, so a hop is a pointer dereference — O(1). Total cost is proportional to the **portion of the graph you touch**, not the dataset size. Then do the arithmetic out loud: 200 → 40,000 → 8,000,000 rows at 3 hops.

### Q2: "You're designing LinkedIn. Would you use Neo4j for the connection graph?"

**Hint:** This is a trap, and the trap is enthusiasm. Say: LinkedIn's dominant query is **shallow** — "am I connected to X, and at what degree?" — at colossal read volume. That's the TAO/FlockDB profile: a sharded, cached adjacency index over MySQL, with degree-of-separation precomputed or bounded-BFS'd from both ends. A general graph DB would struggle to shard at that scale (cut edges become network hops). Where a graph DB *could* earn its place is offline/analytical work — "people you may know," community detection — run on a snapshot. **Right answer: not as the primary store.**

### Q3: "Why are graph databases hard to shard?"

**Hint:** Because a graph resists partitioning *precisely because everything is connected*. Any partition cuts edges, and every cut edge turns an O(1) pointer dereference into a network RPC (~0.5ms) during traversal — which destroys the one thing you bought a graph DB for. Minimising cut edges is graph partitioning, which is NP-hard, and social graphs are the worst case (small-world, no natural cut). Hence most real deployments scale **up** (big machine, graph in RAM) with read replicas, not out.

### Q4: "How would you detect a fraud ring?"

**Hint:** Model accounts, devices, cards, addresses, and IPs as nodes; `USED_DEVICE` / `USED_CARD` / `SHIPS_TO` as edges. A ring is a **dense subgraph** — several accounts collapsing onto shared identifiers. Query for shared-identifier clusters above a size threshold, then run **community detection (Louvain)** and **centrality** to score them. The line that lands: *"individually each account looks clean; the fraud is the SHAPE of the subgraph, and shape is exactly what a row-oriented query cannot see."*

### Q5: "When would you NOT use a graph database?"

**Hint:** Whole-dataset aggregations (use a warehouse), simple CRUD on independent entities (Postgres), high-volume transactional writes, full-text search (Elasticsearch), and anything needing easy horizontal sharding. Then show pragmatism: **most "graph" problems at modest scale are fine in Postgres** — mention `WITH RECURSIVE` and that you'd measure it before adopting a whole new datastore.

---

## Practice exercise

### Build a fraud-ring detector, in memory and in SQL

**Part 1 — Model it (10 min).** On paper, design a property graph for a payments company: node labels `Account`, `Device`, `Card`, `Address`, `IP`; edge types `USED_DEVICE`, `USED_CARD`, `SHIPS_TO`, `LOGGED_IN_FROM`, each with at least one property (e.g. `firstSeen`, `count`). Draw a ring of 4 accounts sharing 2 identifiers.

**Part 2 — Implement it (20 min).** Take the `Graph` / `GraphNode` / `Edge` classes from §3 above and extend `Graph` with:

```js
// Return clusters of >= minRing accounts that transitively share identifiers.
findFraudRings(minRing = 4) { /* your code */ }
```

Do it with BFS over the *undirected* view of the graph, starting from each `Account` node, traversing only identifier edge types, and collecting the connected component. Report any component containing `minRing` or more `Account` nodes. Seed it with ~12 accounts, 3 of which form a genuine ring.

**Part 3 — Contrast it (10 min).** Write the *same* detection as a Postgres `WITH RECURSIVE` query over tables `accounts`, `identifiers`, `account_identifiers(account_id, identifier_id)`. Then write one paragraph answering: at what data volume and query depth would you actually migrate to Neo4j — and what would you measure to prove it?

**Produce:** the graph drawing, the working `findFraudRings` (with a `main()` demo printing the ring it found), the recursive SQL, and the paragraph.

---

## Quick reference cheat sheet

- **Node = entity, Edge = relationship.** An edge is a **first-class object** with its own type, direction, and properties — a foreign key is none of those things.
- **Index-free adjacency** is the whole idea: each node stores **direct pointers** to its neighbours, so one hop is a pointer dereference, **O(1)**.
- **Graph query cost ∝ size of the RESULT**, not the size of the dataset. A 5-hop query costs the same on 10M users as on 100M.
- **SQL traversal cost = O(k^d · log n)** — exponential in depth *and* growing with total table size. At 200 friends: 200 → 40,000 → **8,000,000** rows in 3 hops.
- **Cypher:** `MATCH (me:User)-[:FRIEND*3]-(fof) RETURN fof`. Variable depth: `[:FRIEND*1..6]`.
- **Use a graph DB when:** relationships are the query subject **AND** depth is variable/unbounded **AND** edge types are heterogeneous. All three, ideally.
- **Killer app: fraud detection** — the fraud IS the shape of the subgraph, and shape is invisible to row-oriented queries.
- **Don't use for:** whole-dataset aggregations, plain CRUD, high write volume, full-text search.
- **Graphs resist sharding** because everything is connected — every cut edge becomes a network hop. Partitioning is NP-hard and this is a genuinely open problem. Most deployments **scale up**, not out.
- **The default answer is still Postgres.** Try a `WITH RECURSIVE` CTE first and *measure it*.
- **Algorithms:** shortest path (degrees of separation), PageRank (influence), Louvain (communities), centrality (hubs), triangle counting (tightness).
- **Languages:** Cypher (readable, → ISO GQL), Gremlin (imperative, TinkerPop), SQL/PGQ (SQL:2023). Property graph ≠ RDF/SPARQL triple store.
- **Facebook TAO and Twitter FlockDB are NOT graph databases** — they're custom cached, sharded graph-*serving* layers over MySQL, built for **shallow** queries at colossal scale.
- **The lesson:** extreme scale + shallow queries → purpose-built cached index. Moderate scale + deep, exploratory traversals → graph database.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — graph DBs are a NoSQL family; understand the landscape before picking this branch of it |
| **Next** | [103 — HLD: Twitter](./103-hld-twitter.md) — the follow graph in practice, and why Twitter built FlockDB over MySQL instead of adopting a graph database |
| **Related** | [62 — Database Indexing](./62-database-indexing.md) — you cannot appreciate *index-free* adjacency until you know exactly what a B-tree index lookup costs per hop |
| **Related** | [128 — LLD: Social Media Feed](./128-lld-social-media-feed.md) — the feed is built on top of the follow graph; fan-out strategy depends entirely on its shape |
| **Related** | [79 — Search Systems](./79-search-systems.md) — the thing graph databases are worst at; pair a graph store with an inverted index rather than making either do both |
