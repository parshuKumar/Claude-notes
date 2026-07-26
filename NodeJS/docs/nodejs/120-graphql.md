# 120 — GraphQL with Node.js

## What is this?

GraphQL is a query language for APIs where the **client asks for exactly the fields it needs**, and the server returns exactly that — nothing more, nothing less. Apollo Server is the most popular Node.js library for building a GraphQL API: you define a **schema** (the shape of your data), write **resolvers** (functions that fetch the actual data), and Apollo wires the two together behind a single `/graphql` endpoint. Think of a REST API as a restaurant with fixed combo meals (`/users/5`, `/users/5/posts`) — you get what's on the menu. GraphQL is more like an à la carte order form: one request, you list precisely the dishes (fields) you want, and one plate comes back.

---

## Why does it matter for backend development?

REST APIs suffer from two classic problems: **over-fetching** (client wants a username but the endpoint returns the entire user object with 20 fields) and **under-fetching** (client needs a user plus their last 5 orders, so it fires two or three separate requests). GraphQL solves both with a single endpoint and client-specified field selection. Backend developers reach for GraphQL when building APIs consumed by many different clients (web, mobile, third-party) that each need different slices of the same data, or when a frontend team is iterating fast and doesn't want to wait on new REST endpoints for every UI change. The trade-off backend devs must manage is the **N+1 query problem** — GraphQL's flexibility makes it easy to accidentally fire one database query per item in a list — which is why DataLoader is a required tool, not an optional one, in any serious GraphQL backend.

---

## Syntax / API

```js
// npm install @apollo/server graphql

const { ApolloServer } = require('@apollo/server');       // the GraphQL server engine
const { startStandaloneServer } = require('@apollo/server/standalone'); // quick HTTP wrapper

// ── 1. Type definitions (the schema) — describes the SHAPE of your data ────
const typeDefs = `#graphql
  type User {
    id: ID!               # ! means this field can never be null
    name: String!
    email: String!
  }

  type Query {
    user(id: ID!): User    # a query that returns a single User by id
    users: [User!]!        # a query that returns a list of Users
  }
`;

// ── 2. Resolvers — the FUNCTIONS that actually fetch the data ──────────────
const resolvers = {
  Query: {
    // parent = unused here, args = { id }, contextValue = shared per-request data
    user: (parent, args, contextValue) => {
      return contextValue.dbConnection.findUserById(args.id); // fetch one user
    },
    users: (parent, args, contextValue) => {
      return contextValue.dbConnection.findAllUsers();        // fetch all users
    },
  },
};

// ── 3. Create and start the server ──────────────────────────────────────
const apolloServer = new ApolloServer({ typeDefs, resolvers }); // wire schema + resolvers

async function startApolloServer() {
  const { url } = await startStandaloneServer(apolloServer, {
    listen: { port: 4000 }, // GraphQL endpoint available at this URL
  });
  console.log(`GraphQL server ready at ${url}`); // e.g. http://localhost:4000/
}

startApolloServer(); // kick it off
```

---

## How it works — line by line

- `typeDefs` is a string written in **GraphQL Schema Definition Language (SDL)** — it declares what types of data exist (`User`) and what questions the client can ask (`Query`). `!` after a type means "this can never be null"; `[User!]!` means "a non-null list of non-null Users."
- `resolvers` is a plain object that mirrors the shape of `typeDefs`. For every field the client can ask for, there must be a function that knows how to produce that value. Every resolver function receives four arguments: `parent` (the result of the resolver one level up, used for nested fields), `args` (the arguments the client passed, like `id`), `contextValue` (an object shared across all resolvers in a single request — usually holds the database connection, the logged-in user, auth token), and `info` (metadata about the query, rarely used directly).
- `new ApolloServer({ typeDefs, resolvers })` combines the schema and the resolver functions into one executable engine that knows how to answer any valid query against that schema.
- `startStandaloneServer` spins up an actual HTTP server (built on Express internally) and mounts Apollo at a single route — unlike REST, GraphQL almost always exposes just **one** endpoint (`/` or `/graphql`), and the client tells it what data to return via the query body, not the URL path.
- When a client sends a query, Apollo walks the query field by field, calls the matching resolver for each field, and assembles the results into a JSON response shaped exactly like the query — this is the core mental model: **schema defines what's possible, resolvers define how to get it, the client's query defines what's actually returned**.

---

## Example 1 — basic

```js
// File: src/graphql/server.js
// A minimal Apollo Server with an in-memory "database" — no real DB yet.

const { ApolloServer } = require('@apollo/server');
const { startStandaloneServer } = require('@apollo/server/standalone');

// Fake in-memory data — stands in for a real database table
const bookRecords = [
  { id: '1', title: 'Clean Code', authorName: 'Robert Martin' },
  { id: '2', title: 'The Pragmatic Programmer', authorName: 'Dave Thomas' },
];

// Schema: describes a Book type and two queries
const typeDefs = `#graphql
  type Book {
    id: ID!
    title: String!
    authorName: String!
  }

  type Query {
    books: [Book!]!           # return every book
    book(id: ID!): Book       # return one book by id
  }
`;

// Resolvers: map each Query field to a function that returns matching data
const resolvers = {
  Query: {
    books: () => bookRecords, // no args needed — just return the whole array
    book: (parent, args) => {
      // args.id comes from the client's query, e.g. book(id: "1")
      return bookRecords.find((bookRecord) => bookRecord.id === args.id);
    },
  },
};

const apolloServer = new ApolloServer({ typeDefs, resolvers }); // build the server

async function startApolloServer() {
  const { url } = await startStandaloneServer(apolloServer, {
    listen: { port: 4000 },
  });
  console.log(`Book API ready at ${url}`);
}

startApolloServer();

// Example client query (sent in the request body, not the URL):
// query {
//   book(id: "1") {
//     title
//     authorName
//   }
// }
// Response: { "data": { "book": { "title": "Clean Code", "authorName": "Robert Martin" } } }
```

---

## Example 2 — real world backend use case

```js
// File: src/graphql/server.js
// Realistic setup: users with nested posts, DataLoader to avoid N+1 queries,
// contextValue carrying the authenticated user, and a subscription for live events.

const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4'); // mount inside Express
const { ApolloServerPluginDrainHttpServer } = require('@apollo/server/plugin/drainHttpServer');
const { makeExecutableSchema } = require('@graphql-tools/schema');
const { PubSub } = require('graphql-subscriptions');            // simple in-memory pub/sub
const DataLoader = require('dataloader');                       // batches + caches per request
const express = require('express');
const http = require('http');
const jwt = require('jsonwebtoken');
const dbConnection = require('../db/connection');                // your real DB layer

const pubSub = new PubSub(); // used to publish/subscribe to real-time events
const POST_ADDED = 'POST_ADDED'; // event name constant, avoids typos across the file

// ── Schema ───────────────────────────────────────────────────────────────
const typeDefs = `#graphql
  type Post {
    id: ID!
    title: String!
    authorId: ID!
  }

  type User {
    id: ID!
    email: String!
    posts: [Post!]!          # nested field — resolved via DataLoader below
  }

  type Query {
    me: User                 # the currently authenticated user
    users: [User!]!
  }

  type Mutation {
    createPost(title: String!): Post!   # requires auth — checked in resolver
  }

  type Subscription {
    postAdded: Post!          # clients subscribe to be notified in real time
  }
`;

// ── DataLoader — batches many "get posts for userId X" calls into ONE query ─
function createPostsByAuthorLoader() {
  return new DataLoader(async (userIds) => {
    // userIds = ['1', '2', '3'] — DataLoader collects all requested ids in one tick
    const allPosts = await dbConnection.query(
      'SELECT * FROM posts WHERE author_id = ANY($1)',
      [userIds]
    );
    // Must return results in the SAME ORDER as the input userIds array
    return userIds.map((userId) =>
      allPosts.rows.filter((postRow) => postRow.author_id === userId)
    );
  });
}

// ── Resolvers ────────────────────────────────────────────────────────────
const resolvers = {
  Query: {
    me: (parent, args, contextValue) => {
      if (!contextValue.currentUser) return null; // not logged in
      return dbConnection.findUserById(contextValue.currentUser.userId);
    },
    users: (parent, args, contextValue) => dbConnection.findAllUsers(),
  },
  User: {
    // Called once PER USER in a list — DataLoader collapses all of these
    // into a single batched SQL query instead of one query per user (N+1 fix)
    posts: (parentUser, args, contextValue) => {
      return contextValue.postsByAuthorLoader.load(parentUser.id);
    },
  },
  Mutation: {
    createPost: async (parent, args, contextValue) => {
      if (!contextValue.currentUser) {
        throw new Error('Not authenticated'); // reject unauthenticated mutations
      }
      const newPost = await dbConnection.insertPost({
        title: args.title,
        authorId: contextValue.currentUser.userId,
      });
      pubSub.publish(POST_ADDED, { postAdded: newPost }); // notify subscribers
      return newPost;
    },
  },
  Subscription: {
    postAdded: {
      // asyncIterator streams events to any client that subscribed
      subscribe: () => pubSub.asyncIterator([POST_ADDED]),
    },
  },
};

const schema = makeExecutableSchema({ typeDefs, resolvers });

async function startServer() {
  const app = express();
  const httpServer = http.createServer(app); // needed so subscriptions can share the port

  const apolloServer = new ApolloServer({
    schema,
    plugins: [ApolloServerPluginDrainHttpServer({ httpServer })], // clean shutdown
  });
  await apolloServer.start();

  app.use(
    '/graphql',
    express.json(),
    expressMiddleware(apolloServer, {
      // contextValue is rebuilt on EVERY request — this is what makes
      // DataLoader "per request" caching correct (no stale cache across users)
      context: async ({ req }) => {
        const authToken = req.headers.authorization?.replace('Bearer ', '');
        let currentUser = null;
        try {
          currentUser = authToken ? jwt.verify(authToken, process.env.JWT_SECRET) : null;
        } catch {
          currentUser = null; // invalid/expired token — treat as logged out
        }
        return {
          currentUser,
          dbConnection,
          postsByAuthorLoader: createPostsByAuthorLoader(), // fresh loader per request
        };
      },
    })
  );

  httpServer.listen(4000, () => console.log('GraphQL API ready at :4000/graphql'));
}

startServer();
```

---

## Common mistakes

### Mistake 1 — Fetching relations without DataLoader (the N+1 problem)

```js
// ❌ WRONG — resolver runs once PER USER in the list, firing one SQL query each
// Requesting 100 users' posts = 1 query for users + 100 queries for posts = N+1
const resolvers = {
  User: {
    posts: (parentUser) => {
      return dbConnection.query('SELECT * FROM posts WHERE author_id = $1', [parentUser.id]);
    },
  },
};

// ✅ CORRECT — DataLoader batches all pending calls in the same tick into ONE query
const DataLoader = require('dataloader');
const postsByAuthorLoader = new DataLoader(async (userIds) => {
  const result = await dbConnection.query('SELECT * FROM posts WHERE author_id = ANY($1)', [userIds]);
  return userIds.map((userId) => result.rows.filter((row) => row.author_id === userId));
});

const resolvers = {
  User: {
    posts: (parentUser, args, contextValue) => contextValue.postsByAuthorLoader.load(parentUser.id),
  },
};
```

### Mistake 2 — Reusing one DataLoader instance across all requests

```js
// ❌ WRONG — a module-level loader caches results FOREVER and LEAKS between users
// User A's request populates the cache; User B's request sees stale/wrong data
const sharedLoader = new DataLoader(batchLoadPosts); // created once at startup

app.use('/graphql', expressMiddleware(apolloServer, {
  context: async () => ({ postsByAuthorLoader: sharedLoader }), // BUG: same instance every time
}));

// ✅ CORRECT — create a brand new DataLoader inside context, on every request
app.use('/graphql', expressMiddleware(apolloServer, {
  context: async ({ req }) => ({
    postsByAuthorLoader: new DataLoader(batchLoadPosts), // fresh cache per request, no leakage
  }),
}));
```

### Mistake 3 — Trusting `args` without validation, or leaking internal errors

```js
// ❌ WRONG — no validation on args, and raw DB error details leak to the client
const resolvers = {
  Mutation: {
    createPost: async (parent, args) => {
      // args.title could be empty, 10MB long, or contain malicious content
      return dbConnection.insertPost({ title: args.title });
      // if insertPost throws, Apollo sends the FULL stack trace to the client by default
    },
  },
};

// ✅ CORRECT — validate input and wrap errors in safe, generic messages
const { GraphQLError } = require('graphql');

const resolvers = {
  Mutation: {
    createPost: async (parent, args, contextValue) => {
      if (!args.title || args.title.trim().length === 0) {
        throw new GraphQLError('Title is required', {
          extensions: { code: 'BAD_USER_INPUT' }, // structured error code for clients
        });
      }
      try {
        return await dbConnection.insertPost({ title: args.title.trim() });
      } catch (dbError) {
        throw new GraphQLError('Could not create post', {
          extensions: { code: 'INTERNAL_SERVER_ERROR' }, // no raw dbError.stack exposed
        });
      }
    },
  },
};
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Apollo Server with a `Product` type having `id`, `name`, and `price` fields, backed by an in-memory array of at least 4 products. Add two queries: `products` (returns all) and `product(id: ID!)` (returns one matching product, or `null` if not found). Start the server on port 4000 and verify both queries work.

```js
// Write your code here
```

---

### Exercise 2 — medium

Extend the `Product` type to include a nested `reviews: [Review!]!` field, where each `Review` has `id`, `comment`, and `rating`. Store reviews in a separate in-memory array keyed by `productId`. Write the resolver for `Product.reviews` naively first (one lookup per product), then add a `Mutation.addReview(productId: ID!, comment: String!, rating: Int!)` that validates `rating` is between 1 and 5 (throw a `GraphQLError` with code `BAD_USER_INPUT` otherwise) and pushes a new review into the array.

```js
// Write your code here
```

---

### Exercise 3 — hard

Take the Exercise 2 schema and fix the N+1 problem: implement a `DataLoader` for batching `reviews` lookups by `productId`, wire it into `contextValue` freshly per request using `expressMiddleware`. Then add a `Subscription.reviewAdded(productId: ID!)` using `graphql-subscriptions`' `PubSub`, so any client subscribed to a specific product's reviews gets notified in real time whenever `addReview` is called for that product. Test it by running two client queries side by side — one subscription listener, one mutation call — and confirm the listener receives the new review instantly.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PIECES
  typeDefs    → SDL string describing types, queries, mutations, subscriptions
  resolvers   → functions that fetch the actual data for each field
  contextValue → per-request shared object (db connection, auth user, loaders)

SCHEMA SYNTAX
  String, Int, Float, Boolean, ID   → built-in scalar types
  Type!                              → non-null (cannot be null)
  [Type!]!                           → non-null list of non-null items
  type Query { ... }                 → entry points for READING data
  type Mutation { ... }              → entry points for WRITING data
  type Subscription { ... }          → entry points for REAL-TIME events

RESOLVER SIGNATURE
  (parent, args, contextValue, info) => { ... }
  parent       → result of the parent field (for nested resolvers)
  args         → arguments passed in the query/mutation
  contextValue → shared per-request data (auth, db, loaders)

DATALOADER (fixes N+1)
  new DataLoader(async (keys) => { ... })   → batches all .load() calls in one tick
  MUST return results in SAME ORDER as input keys array
  MUST be created FRESH per request (inside context()), never at module level

SUBSCRIPTIONS
  PubSub from 'graphql-subscriptions'   → in-memory pub/sub (single instance only)
  pubSub.publish(EVENT_NAME, payload)   → fire an event
  pubSub.asyncIterator([EVENT_NAME])    → resolver subscribes to it
  Production: use graphql-redis-subscriptions for multi-instance servers

ERROR HANDLING
  throw new GraphQLError('message', { extensions: { code: 'BAD_USER_INPUT' } })
  Never let raw DB/stack trace errors reach the client

COMMON GOTCHAS
  Forgetting DataLoader           → N+1 queries, slow at scale
  Sharing DataLoader across users → stale/leaked cached data
  No input validation on args     → bad data reaches your database
  One giant resolver doing everything → split into services/repositories
```

---

## Connected topics

- **53 — Express.js fundamentals** — Apollo Server mounts as middleware on an Express app (`expressMiddleware`), so knowing `app.use()` and middleware ordering is required to wire it up correctly.
- **119 — gRPC with Node.js** — the other major alternative to REST; compare GraphQL's flexible client-driven queries against gRPC's strict, contract-first, binary-protocol RPC style to know when to pick each.
- **92 — Connecting to Redis** — production subscriptions and DataLoader-style caching often move from in-memory `PubSub`/per-request caches to Redis-backed pub/sub and shared caches once you scale past one server instance.
