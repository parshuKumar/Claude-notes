# 01 — What Is System Design?
## Category: HLD Fundamentals

---

## What is this?

System design is the process of defining the **architecture, components, data flow, and trade-offs** of a software system to satisfy a given set of requirements. Think of it like being the architect of a skyscraper — before a single brick is laid, someone must decide how many floors it will have, where the elevators go, how the plumbing works, and how it stays standing in an earthquake. System design is that same discipline, applied to software.

It is not about writing code. It is about making the **big decisions** — what kind of database to use, how to handle 10 million users, what happens when one server crashes, how data flows from a user's phone to a data center and back in under 100 milliseconds.

---

## Why does it matter?

**Without system design, code falls apart at scale.**

Imagine you built a fantastic to-do app. It works perfectly for 10 users. Then it gets featured on Product Hunt and 100,000 people sign up in one day. Your single server catches fire. Your database grinds to a halt. Users see errors. The app dies. This happens to real companies — and it happens because the **system** was never designed, only the code.

System design matters because:

- **Scale** — code that works for 100 users breaks at 1,000,000
- **Reliability** — real systems must keep working even when hardware fails
- **Cost** — bad design wastes millions in infrastructure
- **Speed** — wrong architecture makes features impossible to add later
- **Interviews** — every senior/staff/principal engineering interview tests this

Engineers who understand system design get promoted faster, earn more, and are trusted with the most important projects. It is the single biggest gap between a junior and senior engineer.

---

## The core idea — explained simply

Let's use a **real-world analogy**. Imagine you are opening a restaurant.

A **junior cook** thinks about recipes — how to make each dish perfectly. This is like a programmer who thinks about individual functions and algorithms.

An **executive chef / restaurant owner** thinks about the entire operation:
- How many tables do we need? (Capacity)
- What happens when the kitchen printer breaks? (Fault tolerance)
- How do we serve 200 people at once without long waits? (Throughput)
- Where do we store ingredients so the kitchen doesn't run out? (Caching)
- What if we want to open 10 more locations? (Scaling)
- How do we take reservations and payments? (APIs and integrations)

**System design is thinking like the restaurant owner, not the cook.**

When you design a software system, you are making decisions at the restaurant level — not the recipe level. You decide:

- How many servers handle requests
- Where data is stored and how it is accessed
- How the system survives failures
- How it grows from 1,000 to 1,000,000,000 users
- How different parts of the system talk to each other

Here is a concrete before/after to make this click:

**Without system design thinking:**
> "I'll use Node.js and store everything in a MySQL database. Done."

**With system design thinking:**
> "I need to handle 50,000 requests/second. A single MySQL instance handles ~5,000 writes/sec — so I need read replicas for the read-heavy queries. I'll put a cache in front of the DB for frequently read data. I'll put a load balancer in front of multiple Node.js instances so no single server is a bottleneck. I'll use a CDN for static assets. For the write path, I'll use a message queue so spikes don't crash the database."

The second engineer is doing **system design**. The first engineer will be on call at 3 AM when the server explodes.

---

## Key concepts inside this topic

### 1. What system design is NOT
- It is not writing code (though it informs what code gets written)
- It is not just drawing boxes (the reasoning behind every box matters)
- It is not memorizing architectures (it is learning trade-offs so you can build any architecture)

### 2. The two branches of system design

**High Level Design (HLD)**
The macro view — how the system is broken into major components and how they communicate.
- What databases to use
- How many servers, what kind
- How data flows end to end
- Where caches, queues, and CDNs live
- Example question: "Design Twitter's feed"

**Low Level Design (LLD)**
The micro view — how individual components are structured internally, class by class.
- What classes exist and how they relate
- What design patterns to apply
- What interfaces and contracts components expose
- Example question: "Design the classes for a parking lot system"

We will cover both in this curriculum. Think of HLD as the map of a city and LLD as the floor plan of one building.

### 3. The core concerns of every system design
No matter what system you design, you will always reason about these:

| Concern | Question it answers |
|---|---|
| **Scalability** | Can it handle 100x more users? |
| **Availability** | Does it stay up even when parts fail? |
| **Reliability** | Does it produce correct results consistently? |
| **Performance** | Is it fast enough for users? |
| **Maintainability** | Can engineers change it without breaking it? |
| **Cost** | Is it affordable to run? |
| **Security** | Does it protect user data? |
| **Consistency** | Does every user see the same data? |

### 4. The building blocks (you will learn all of these)
Every large system is assembled from a set of standard building blocks:
- **Servers** — compute
- **Databases** — storage
- **Caches** — speed up reads
- **Load balancers** — distribute traffic
- **Message queues** — decouple components
- **CDNs** — serve content fast globally
- **APIs** — define communication contracts
- **DNS** — map names to servers

Understanding what each block does and when to use it is what this curriculum builds.

### 5. Trade-offs are the heart of system design
There is no "perfect" design. Every decision trades one thing for another:

- More availability → potentially inconsistent data
- More speed → more money spent on caching
- More simplicity → less flexibility later
- More flexibility → more complexity now

A principal engineer does not find the "right" answer — they find the **best trade-off** for the given constraints. This curriculum will train that muscle.

---

## Visual / Diagram description

**Draw this on a whiteboard:**

```
Draw a simple before/after:

LEFT SIDE — "Junior Engineer's view":
[ User ] ──→ [ Node.js Server ] ──→ [ MySQL Database ]
Label it: "Works fine for 100 users. Dies at 100,000."

RIGHT SIDE — "System Design view":
[ Users (millions) ]
       │
       ↓
[ DNS ] → resolves to nearest data center
       │
       ↓
[ CDN ] → serves static files (images, JS, CSS) without hitting server
       │
       ↓
[ Load Balancer ] → distributes requests across multiple servers
     /    \
[Server 1] [Server 2] ... [Server N]   ← can add more as load grows
     \    /
       │
       ↓
[ Cache (Redis) ] → answers common reads instantly, skips DB
       │
       ↓ (cache miss only)
[ Primary Database ] ──replicates──→ [ Read Replica 1 ] [ Read Replica 2 ]
       │
       ↓
[ Message Queue ] → offloads heavy async work (emails, notifications)
       │
       ↓
[ Worker Servers ] → process queue jobs in the background
```

Label every arrow with what flows through it (HTTP request, SQL query, event message). This is the standard mental model for a web system — memorize this shape.

---

## Real world examples

### 1. WhatsApp
WhatsApp serves 2 billion users. A single server cannot store the message history of 2 billion people. System design decisions made WhatsApp possible: how messages are routed, how presence (online/offline) is tracked at scale, how messages are stored and delivered reliably even when a phone is offline, how end-to-end encryption works across billions of devices. None of this is a coding problem — it is a system design problem.

### 2. Netflix
Netflix streams video to 230 million subscribers simultaneously. Their system design decisions include: CDNs deployed inside ISPs (so video data never leaves the ISP's network), a microservices architecture with 700+ services, chaos engineering (deliberately killing servers to test resilience), and a sophisticated recommendation pipeline. The engineering that makes this possible is system design, not individual algorithms.

### 3. Amazon's shopping cart
Amazon's shopping cart famously uses "eventual consistency" — meaning if two people add an item to the same cart at exactly the same moment from different devices, both additions eventually show up, even if one is briefly invisible. This is a deliberate system design choice trading perfect consistency for higher availability. A system designer made this call. It was the right trade-off for Amazon's scale.

---

## Trade-offs

### Pros of doing system design well
- Systems that survive failures instead of crashing
- Systems that scale cheaply as users grow
- Systems that are easy to maintain and extend
- Engineers who can work on any problem at any scale
- Career advancement — system design is how senior/staff/principal engineers are evaluated

### Cons / what you give up
- **Time** — designing a system upfront takes time. Sometimes a startup should just ship and fix later
- **Complexity** — a well-designed scalable system has more moving parts than a simple one
- **Over-engineering risk** — designing for 1 billion users on day 1 is wasteful if you have 100 users
- **Learning curve** — system design is a separate skill from coding; it must be learned intentionally

### The key tension
> "Simple is fast to build but hard to scale. Complex is hard to build but easy to scale."

The art of system design is knowing **when to add complexity** — not too early, not too late.

---

## Common interview questions on this topic

**Q1: What is the difference between system design and software design?**
Hint: System design is about the architecture of multiple components (servers, DBs, queues) and how they interact. Software design (LLD) is about the internal structure of a single component — classes, methods, patterns.

**Q2: Why does system design matter more at scale?**
Hint: At small scale, any approach works. At large scale, wrong decisions compound — the wrong DB choice at 1000 users becomes an impossible migration at 100 million users.

**Q3: How do you approach a system design problem you've never seen before?**
Hint: Use a framework — clarify requirements, estimate scale, design high-level components, deep-dive into critical parts, discuss trade-offs. Never start drawing boxes immediately. We'll cover this in depth in topic 03.

**Q4: What are non-functional requirements and why do engineers often miss them?**
Hint: NFRs are requirements like "must handle 10,000 requests/second" or "must have 99.9% uptime" — they describe HOW the system must behave, not WHAT it must do. They are missed because users describe features (functional) but not constraints (non-functional). Missing NFRs cause systems to fail in production.

**Q5: What does "trade-off" mean in system design, and can you give an example?**
Hint: Every design decision has a cost. Example: using a cache improves read speed (pro) but introduces the risk of serving stale data (con). The trade-off is speed vs. data freshness. Good engineers name both sides explicitly.

---

## Practice exercise

**Task — Do this yourself, don't skip it:**

You are hired to design the backend for a new social media app. The CEO tells you:
> "We expect 500 users on day 1. But if it goes viral, maybe 5 million users in a month."

Answer these 3 questions in writing:

1. What is the **single biggest system design risk** in this scenario, and why?
2. Name **two system design decisions** you would make differently for 500 users vs 5 million users.
3. What does "scale" actually mean here — what specifically breaks when you go from 500 to 5 million users?

Write your answers in plain English — no code needed. This is a thinking exercise.

---

## Quick reference cheat sheet

- **System design** = deciding architecture, components, and trade-offs before coding
- **HLD** = macro view — how major components connect (servers, DBs, queues, CDNs)
- **LLD** = micro view — how one component is structured internally (classes, patterns)
- **Scale** = the system's ability to handle growth in users, data, or requests
- **Trade-off** = every design decision costs something — name both sides
- **Core concerns** = scalability, availability, reliability, performance, maintainability, cost, security, consistency
- **Building blocks** = servers, databases, caches, load balancers, queues, CDNs, APIs, DNS
- **No perfect design** — only the best trade-off for the given constraints
- **Simple first, scale when needed** — don't over-engineer before you have real scale problems
- Junior engineers think about code. Senior engineers think about systems. Principal engineers think about trade-offs.

---

## Connected topics

**What to understand before this topic:**
- Basic programming and REST APIs (you already have this ✅)
- What a database is and how a server works (you already have this ✅)

**What comes directly after:**
- **Topic 02 — HLD vs LLD**: Now that you know what system design is, understand its two branches in depth
- **Topic 03 — How to Approach System Design Problems**: The repeatable framework for tackling any design question
- **Topic 04 — Functional vs Non-Functional Requirements**: How to gather what you actually need to design before drawing anything
