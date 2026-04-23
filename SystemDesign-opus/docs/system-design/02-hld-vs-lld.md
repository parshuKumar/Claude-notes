# 02 — HLD vs LLD — The Two Halves of System Design
## Category: HLD Fundamentals

---

## What is this?

High-Level Design (HLD) and Low-Level Design (LLD) are the **two zoom levels** of system design. HLD is the **satellite view** — you see the entire city, the roads connecting neighborhoods, where the airport and train station are. LLD is the **street view** — you see the inside of one building, the layout of rooms, where the furniture goes, how the plumbing is connected.

When someone says "design WhatsApp," HLD answers **"what components exist and how they talk to each other"** while LLD answers **"what classes, methods, and data structures live inside each component."**

---

## Why does it matter?

If you only know HLD, you can draw beautiful architecture diagrams but you can't actually implement them — your code will be a mess of tightly coupled classes, duplicated logic, and unmaintainable spaghetti.

If you only know LLD, you can write clean, well-structured code inside a single service but you'll have no idea how to make that service work with 50 other services, handle millions of users, or survive server crashes.

**In interviews:**
- **Senior roles (L5/SDE3):** You'll get BOTH HLD and LLD rounds. Some companies do them as separate interviews.
- **Staff+ roles (L6+):** HLD is weighted more heavily, but you're still expected to zoom into LLD when the interviewer probes.
- **Mid-level roles (L4/SDE2):** LLD (machine coding) is often the primary design round.

**At work:**
- You start a project with HLD (architecture review with the team)
- Then each engineer takes a component and does LLD (class design, API contracts)
- Both happen in sequence — HLD first, then LLD

---

## The core idea — explained simply

### The House-Building Analogy

You want to build a house. There are two phases of design:

**Phase 1 — High-Level Design (the architect's blueprint):**
- The house will have 3 floors
- Ground floor: living room, kitchen, garage
- First floor: 3 bedrooms, 2 bathrooms
- Second floor: home office, terrace
- The electrical system connects to the city grid
- Water comes from municipal supply → water tank → rooms
- There will be a main entrance and a back entrance

This tells you WHAT exists and WHERE everything goes. But you can't build from this alone.

**Phase 2 — Low-Level Design (the engineer's detail drawings):**
- The kitchen will have a granite countertop, 120cm × 60cm
- The sink is a double-basin stainless steel, positioned under the window
- Electrical outlets: 4 on the east wall at 110cm height, 2 on the north wall
- The water pipe from the tank to the kitchen is 3/4 inch PVC, running through the west wall cavity
- Cabinet doors use soft-close hinges, opening left-to-right

This tells you HOW to actually build each room — the exact materials, measurements, and connections.

**In software, it's the same:**

| Aspect | HLD (Architect's Blueprint) | LLD (Engineer's Detail Drawing) |
|--------|---------------------------|-------------------------------|
| **Scope** | The entire system | One component/service |
| **Question answered** | "What are the pieces and how do they connect?" | "How does this one piece work internally?" |
| **Output** | Architecture diagram, component list, data flow | Class diagrams, sequence diagrams, code structure |
| **Audience** | Tech leads, architects, product managers | Engineers who will write the code |
| **Tools** | Boxes, arrows, service names, databases, queues | Classes, interfaces, methods, design patterns |

---

## Key concepts inside this topic

### 1. What High-Level Design (HLD) Covers

HLD is about the **big boxes and the arrows between them.** When you do HLD, you're deciding:

**a) Components / Services**
What are the major building blocks?
```
Example for a chat app:
- User Service (handles registration, profiles, authentication)
- Message Service (handles sending, receiving, storing messages)
- Notification Service (handles push notifications)
- Presence Service (tracks who is online/offline)
- Media Service (handles image/video uploads)
```

**b) Communication Between Services**
How do they talk to each other?
```
- Synchronous: REST APIs, gRPC (one service calls another and WAITS for response)
- Asynchronous: Message queues like Kafka/RabbitMQ (one service sends a message and DOESN'T wait)
```

**c) Data Storage**
What databases, where, and what type?
```
- User data → PostgreSQL (relational, structured data)
- Messages → Cassandra (write-heavy, time-series-like data)
- Media files → S3 / Blob storage (large binary files)
- User sessions → Redis (fast, temporary data)
```

**d) Scalability Strategy**
How does the system handle growth?
```
- Load balancer in front of API servers
- Database sharding by user_id
- CDN for media files
- Message queue to decouple services
```

**e) Reliability & Fault Tolerance**
What happens when things break?
```
- Database replication (if master dies, replica takes over)
- Multiple instances of each service (if one crashes, others handle traffic)
- Circuit breakers (if a downstream service is down, fail gracefully)
```

### 2. What Low-Level Design (LLD) Covers

LLD is about the **internals of one component.** When you do LLD, you're deciding:

**a) Classes and Objects**
What entities exist in this component?
```javascript
// For the Message Service:
class User {
  constructor(id, name, email) { ... }
}

class Message {
  constructor(id, senderId, receiverId, content, timestamp) { ... }
}

class Conversation {
  constructor(id, participants, messages) { ... }
}

class MessageService {
  sendMessage(senderId, receiverId, content) { ... }
  getMessages(conversationId, page, limit) { ... }
  deleteMessage(messageId, userId) { ... }
}
```

**b) Relationships Between Classes**
How do classes connect?
```
- A User HAS MANY Messages (one-to-many)
- A Conversation HAS MANY Users (many-to-many)
- A Message BELONGS TO a Conversation (many-to-one)
- MessageService USES a MessageRepository (dependency)
```

**c) Design Patterns**
Which patterns solve which problems?
```
- Observer pattern → notify users when a new message arrives
- Strategy pattern → different message delivery strategies (push, SMS, email)
- Factory pattern → create different types of messages (text, image, video)
- Singleton pattern → single database connection pool instance
```

**d) Method Signatures and Logic**
What does each method actually do?
```javascript
class MessageService {
  /**
   * Sends a message from one user to another.
   * 1. Validate sender and receiver exist
   * 2. Create the message object
   * 3. Store in database
   * 4. Add to conversation
   * 5. Notify receiver (async via event)
   */
  async sendMessage(senderId, receiverId, content) {
    const sender = await this.userRepo.findById(senderId);
    if (!sender) throw new Error('Sender not found');

    const receiver = await this.userRepo.findById(receiverId);
    if (!receiver) throw new Error('Receiver not found');

    const message = new Message({
      senderId,
      receiverId,
      content,
      timestamp: Date.now(),
      status: 'SENT'
    });

    await this.messageRepo.save(message);
    this.eventBus.emit('message:sent', message);

    return message;
  }
}
```

**e) Interface Definitions**
What contracts do classes follow?
```javascript
// Any storage mechanism must implement this interface
class MessageRepository {
  async save(message) { throw new Error('Not implemented'); }
  async findById(id) { throw new Error('Not implemented'); }
  async findByConversation(conversationId, page, limit) { throw new Error('Not implemented'); }
  async delete(id) { throw new Error('Not implemented'); }
}

// Concrete implementations
class PostgresMessageRepository extends MessageRepository {
  async save(message) { /* SQL INSERT */ }
}

class MongoMessageRepository extends MessageRepository {
  async save(message) { /* MongoDB insertOne */ }
}
```

### 3. How HLD and LLD Connect

They are NOT separate processes. They're **zoom levels of the same design:**

```
ZOOM LEVEL 1 (HLD):  System → Services → Databases → Queues → External APIs
                          │
                     Pick one service, zoom in
                          │
                          ▼
ZOOM LEVEL 2 (LLD):  Service → Classes → Methods → Data Structures → Patterns
```

**In an interview, the flow often is:**
1. Interviewer: "Design a chat system" → You do HLD
2. Interviewer: "Zoom into the message service. How would you design the classes?" → You switch to LLD
3. Interviewer: "How does your message delivery handle failures?" → Mix of HLD (retry queues) and LLD (retry logic in code)

**At work, the flow is:**
1. Team meeting: Agree on HLD architecture → produces an architecture document
2. Sprint planning: Each engineer takes a service → does LLD for that service
3. Code review: LLD is verified through actual code

### 4. The Key Differences — Side by Side

| Dimension | HLD | LLD |
|-----------|-----|-----|
| **Scope** | Entire system | Single component/module |
| **Abstraction** | Very high — boxes and arrows | Very low — classes and methods |
| **Output** | Architecture diagram, tech stack decisions, data flow | Class diagram, sequence diagram, actual code |
| **Key decisions** | Which database? How many services? Sync or async? | Which design pattern? What class hierarchy? What methods? |
| **Scale concerns** | Millions of users, terabytes of data | Memory efficiency, time complexity, clean interfaces |
| **Failure handling** | Replication, failover, circuit breakers, retries | Exception handling, validation, error states |
| **Who does it** | Tech leads, architects, senior engineers | All engineers (senior engineers design, juniors implement) |
| **Interview format** | 45-min whiteboard: draw the architecture | 45-60 min: design classes + sometimes write code |
| **Example question** | "Design Instagram" | "Design the classes for a parking lot system" |

### 5. When to Use HLD vs LLD

```
"We need to build a new microservice" 
  → Start with HLD: Where does it fit in the architecture? What does it talk to?
  → Then LLD: What classes go inside it?

"We need to add a feature to an existing service"
  → Mostly LLD: What new classes/methods are needed?
  → Maybe small HLD: Does this feature need a new database table? A new queue?

"We need to redesign the entire backend"
  → Heavy HLD: New architecture, new service boundaries, new data flow
  → Then LLD for each service

"We need to optimize a slow function"
  → Pure LLD: Algorithm choice, data structure choice, code refactoring
```

---

## Visual / Diagram description

### Diagram 1: HLD View of a Chat System

```
┌──────────┐       ┌──────────────────┐       ┌────────────────────────────────┐
│  Mobile   │       │                  │       │         API Gateway            │
│  Client   │──────▶│  Load Balancer   │──────▶│  (auth, rate limit, routing)   │
│           │◀──────│                  │◀──────│                                │
└──────────┘       └──────────────────┘       └────────┬───────┬───────┬───────┘
                                                       │       │       │
                                               ┌───────▼──┐ ┌──▼─────┐ ┌▼──────────┐
                                               │  User    │ │Message │ │Presence   │
                                               │  Service │ │Service │ │Service    │
                                               └────┬─────┘ └──┬─────┘ └─────┬─────┘
                                                    │           │             │
                                               ┌────▼─────┐ ┌──▼─────┐  ┌───▼──────┐
                                               │ User DB  │ │Msg DB  │  │  Redis    │
                                               │(Postgres)│ │(Cassan)│  │ (online   │
                                               └──────────┘ └────────┘  │  status)  │
                                                                        └──────────┘
                                                       │
                                               ┌───────▼──────────┐
                                               │  Notification    │
                                               │  Queue (Kafka)   │
                                               └───────┬──────────┘
                                                       │
                                               ┌───────▼──────────┐
                                               │  Push Service    │
                                               │  (FCM / APNs)    │
                                               └──────────────────┘
```

**This is HLD.** You see services, databases, queues, and external systems. You do NOT see what's inside any service.

### Diagram 2: LLD View — Zoomed Into the Message Service

```
┌─────────────────────────────────────────────────────────────┐
│                     MESSAGE SERVICE                          │
│                                                             │
│  ┌─────────────────┐    uses    ┌──────────────────────┐   │
│  │ MessageController│──────────▶│   MessageService      │   │
│  │ (REST endpoints)│           │                        │   │
│  │                 │           │ + sendMessage()        │   │
│  │ POST /messages  │           │ + getMessages()        │   │
│  │ GET /messages   │           │ + deleteMessage()      │   │
│  │ DELETE /messages│           │ + markAsRead()         │   │
│  └─────────────────┘           └──────────┬─────────────┘   │
│                                           │                  │
│                              uses         │      emits       │
│                         ┌─────────────────┼──────────┐       │
│                         │                 │          │       │
│                         ▼                 ▼          ▼       │
│  ┌──────────────────┐  ┌──────────┐  ┌───────────┐          │
│  │MessageRepository │  │  Message  │  │ EventBus  │          │
│  │  (interface)     │  │  (model)  │  │(observer) │          │
│  │                  │  │          │  │           │          │
│  │ + save()         │  │ + id     │  │ + emit()  │          │
│  │ + findById()     │  │ + sender │  │ + on()    │          │
│  │ + findByConvo()  │  │ + text   │  │ + off()   │          │
│  │ + delete()       │  │ + time   │  └───────────┘          │
│  └───────┬──────────┘  │ + status │                          │
│          │             └──────────┘                          │
│     implements                                               │
│          │                                                   │
│  ┌───────▼──────────┐                                       │
│  │CassandraMessage  │                                       │
│  │Repository        │                                       │
│  │                  │                                       │
│  │ + save()  {CQL}  │                                       │
│  │ + findById(){CQL}│                                       │
│  └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

**This is LLD.** You see classes, methods, relationships, and patterns INSIDE one service. You do NOT see other services or the broader system.

---

## Real world examples

### 1. Amazon — HLD vs LLD Split

**HLD decisions Amazon made:**
- Broke the monolith into hundreds of microservices (each team owns their service)
- Used DynamoDB for cart/session data (fast key-value lookups)
- Used SQS queues between order service and fulfillment service
- CDN (CloudFront) for product images

**LLD decisions inside Amazon's Order Service (conceptual):**
- `Order` class with states: CREATED → PAID → SHIPPED → DELIVERED → CANCELLED
- State pattern to manage order transitions
- Strategy pattern for different shipping calculators (standard, express, prime)
- Repository pattern to abstract DynamoDB access

### 2. Uber — Both Levels in Action

**HLD:**
- Separate services for rider, driver, matching, pricing, payment, notifications
- Real-time location updates via WebSockets
- Kafka for event streaming between services
- Geospatial database for driver locations

**LLD inside the Matching Service:**
- `Ride` class (rider, driver, pickup, dropoff, status)
- `MatchingAlgorithm` interface with different implementations (nearest driver, best rated, etc.)
- Strategy pattern to swap matching algorithms
- Observer pattern to notify rider when driver is assigned

### 3. GitHub — Where Design Levels Meet

**HLD:**
- Web service, Git service, file storage service, CI/CD service, notification service
- PostgreSQL for metadata, Git objects stored in custom storage
- Redis for caching, Kafka for events (push notifications, webhooks)

**LLD inside the Pull Request module:**
- `PullRequest` class (title, description, source branch, target branch, status, reviewers)
- `Review` class (reviewer, comments, approval status)
- `MergeStrategy` interface (merge commit, squash, rebase) — Strategy pattern
- State pattern for PR lifecycle: OPEN → REVIEWING → APPROVED → MERGED / CLOSED

---

## Trade-offs

| Decision | Choosing HLD-First | Choosing LLD-First |
|----------|-------------------|-------------------|
| **When it works** | Greenfield projects, large teams, complex systems | Small features, single-service changes, prototypes |
| **Benefit** | Everyone agrees on architecture before coding starts | Ship faster, iterate quickly |
| **Risk** | Analysis paralysis — spend too long designing, never ship | Local optimization — clean code that doesn't fit the bigger picture |
| **At what scale** | Systems with 3+ services or 5+ engineers | Single service, 1-2 engineers |

| Approach | Benefit | Cost |
|----------|---------|------|
| Very detailed HLD | Clear contracts between teams, fewer integration surprises | Slower to start, may over-design for current needs |
| Minimal HLD | Start building faster | May need expensive rewrites when services don't fit together |
| Very detailed LLD | Clean, maintainable code, easy to test | Takes longer to deliver, might be overkill for a prototype |
| Minimal LLD | Ship fast, iterate | Technical debt accumulates, harder to modify later |

**The sweet spot:** Do enough HLD that all engineers agree on the architecture. Do enough LLD that the critical/complex components are well-designed. Don't over-design simple CRUD services.

---

## Common interview questions on this topic

### Q1: "What's the difference between HLD and LLD?"
**Hint:** HLD = system architecture (services, databases, communication). LLD = internal design of a component (classes, methods, patterns). HLD is the map; LLD is the building blueprints.

### Q2: "If you had 45 minutes to design a system, how would you split time between HLD and LLD?"
**Hint:** ~5 min requirements, ~5 min estimation, ~15 min HLD (draw the architecture), ~15 min deep-dive into 1-2 components (mix of HLD detail and LLD), ~5 min trade-offs/bottlenecks. Don't try to do full LLD in an HLD interview — the interviewer will tell you when to zoom in.

### Q3: "Can you skip HLD and jump straight to LLD?"
**Hint:** For a small feature in an existing system, yes — the HLD already exists. For a new system, absolutely not — you'll build components that don't fit together. Always understand the big picture first.

### Q4: "In an interview, how do you know whether they want HLD or LLD?"
**Hint:** Listen to the question. "Design Instagram" = HLD. "Design the classes for a parking lot" = LLD. "Design Instagram's news feed service" = starts HLD, will zoom into LLD. If unsure, ASK: "Would you like me to focus on the high-level architecture or the class-level design?"

### Q5: "What comes first — HLD or LLD? Always?"
**Hint:** HLD almost always comes first. You need to know what the system looks like before you design the internals. Exception: if you're adding to an existing system where HLD is already decided, you go straight to LLD.

---

## Practice exercise

### The Two-Zoom Exercise

Imagine you're building **a simple URL shortener** (like bit.ly). Users enter a long URL and get a short one back. When someone visits the short URL, they get redirected to the original.

**Part A — HLD (draw the big picture):**
1. List the **services/components** you think this system needs (at least 3).
2. What **database(s)** would you use and what data goes in each?
3. Describe the **data flow** for TWO operations:
   - User creates a short URL (what components are involved, in what order?)
   - User visits a short URL and gets redirected (what components are involved, in what order?)

**Part B — LLD (zoom into one component):**
1. For the "URL Service" (the core service), list the **classes** you would create (at least 3 classes).
2. For each class, list **2-3 methods** it would have.
3. What is **one design pattern** you might use here and why?

This exercise tests whether you can think at BOTH zoom levels. Keep it simple — don't over-engineer.

---

## Quick reference cheat sheet

- **HLD** = High-Level Design = architecture = "what components exist and how they connect"
- **LLD** = Low-Level Design = class design = "what classes/methods exist inside a component"
- **HLD output:** architecture diagram, service list, database choices, data flow, scaling strategy
- **LLD output:** class diagram, sequence diagram, method signatures, design patterns, code
- **Interview split:** ~60% HLD + ~40% deep-dive for senior roles; mostly LLD for mid-level roles
- **HLD comes first** — you need the big picture before designing the internals
- **Both are required** — HLD without LLD = unbuildable; LLD without HLD = unscalable
- **The zoom trick:** In interviews, start at HLD and zoom into LLD when asked — shows range
- **Don't over-design:** Simple CRUD services need minimal LLD; complex core services need thorough LLD

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [01 — What Is System Design](./01-what-is-system-design.md) — the foundation we just built |
| **Next** | [03 — How to Approach a System Design Problem](./03-how-to-approach-system-design.md) — the step-by-step framework for tackling any design question |
| **Related** | [20 — UML Class Diagrams](./20-uml-class-diagrams.md) — the visual language for LLD |
| **Related** | [93 — HLD Approach Framework](./93-hld-approach-framework.md) — the detailed 5-step HLD method |
