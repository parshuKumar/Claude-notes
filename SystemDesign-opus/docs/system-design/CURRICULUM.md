# System Design — Complete Curriculum Plan
## From Zero to Principal Engineer Level

> **Total Topics: 120**
> **Parts: 8** (Foundations → LLD Foundations → LLD Patterns → HLD Fundamentals → HLD Advanced → HLD Case Studies → LLD Case Studies → Capstone)
> **Each topic = one markdown file in /docs/system-design/**

---

## Legend
- 📄 = One markdown file will be created
- **[F]** = Foundation — must understand before moving forward
- **[LLD]** = Low Level Design topic
- **[HLD]** = High Level Design topic
- **[CS]** = Case Study (design a real system)

---

# PART 1 — FOUNDATIONS (Topics 01–12)
*What system design IS, how to think about it, and the universal vocabulary every engineer must know.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 01 | `01-what-is-system-design.md` | What Is System Design and Why It Matters | The big picture — what system design actually is, why companies ask it in interviews, and why every senior engineer needs it. |
| 02 | `02-hld-vs-lld.md` | HLD vs LLD — The Two Halves of System Design | The difference between high-level architecture and low-level class design, when each is used, and how they connect. |
| 03 | `03-how-to-approach-system-design.md` | How to Approach a System Design Problem | The universal framework — a step-by-step method to tackle ANY system design question in interviews or real work. |
| 04 | `04-functional-vs-non-functional-requirements.md` | Functional vs Non-Functional Requirements | How to gather what a system must DO vs how well it must PERFORM — the foundation of every design decision. |
| 05 | `05-capacity-estimation.md` | Capacity Estimation and Back-of-the-Envelope Math | How to estimate users, storage, bandwidth, and QPS — the math every interviewer expects you to do on a whiteboard. |
| 06 | `06-latency-and-throughput.md` | Latency vs Throughput — The Two Performance Kings | What response time and request rate mean, real numbers to memorize, and how to reason about performance. |
| 07 | `07-availability-and-reliability.md` | Availability and Reliability — Uptime Guarantees | What 99.9% vs 99.99% uptime means in real minutes, SLAs, SLOs, SLIs, and how companies guarantee reliability. |
| 08 | `08-consistency-models.md` | Consistency Models — Strong, Eventual, and Everything Between | What it means for data to be "consistent," why distributed systems make this hard, and the spectrum of consistency levels. |
| 09 | `09-cap-theorem.md` | CAP Theorem — The Impossible Triangle | Why you cannot have Consistency + Availability + Partition Tolerance all at once, with real database examples. |
| 10 | `10-networking-fundamentals.md` | Networking Fundamentals for System Design | TCP vs UDP, HTTP/HTTPS, WebSockets, TCP handshake, ports, IP addressing — the network knowledge you need before designing systems. |
| 11 | `11-how-the-web-works.md` | How the Web Works End to End | What happens when you type a URL — DNS lookup, TCP connection, TLS handshake, HTTP request, server processing, response rendering. |
| 12 | `12-numbers-every-engineer-should-know.md` | Numbers Every Engineer Should Know | Latency of L1 cache vs RAM vs SSD vs network vs disk, how much data fits in memory, bandwidth limits — the cheat sheet for estimation. |

---

# PART 2 — LLD FOUNDATIONS (Topics 13–24)
*The principles, relationships, and modeling techniques that make your class-level designs clean and maintainable.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 13 | `13-oop-four-pillars.md` | Object Oriented Programming — The 4 Pillars for Design | Encapsulation, Abstraction, Inheritance, Polymorphism — not just code syntax, but WHY each pillar matters for designing systems. |
| 14 | `14-solid-single-responsibility.md` | SOLID — Single Responsibility Principle (SRP) | A class should have one reason to change — what this means in practice with good vs bad code examples. |
| 15 | `15-solid-open-closed.md` | SOLID — Open/Closed Principle (OCP) | Open for extension, closed for modification — how to write code that grows without breaking existing functionality. |
| 16 | `16-solid-liskov-substitution.md` | SOLID — Liskov Substitution Principle (LSP) | Subtypes must be substitutable for their base types — the most misunderstood SOLID principle, explained with real examples. |
| 17 | `17-solid-interface-segregation.md` | SOLID — Interface Segregation Principle (ISP) | No client should be forced to depend on methods it doesn't use — how to design lean, focused interfaces. |
| 18 | `18-solid-dependency-inversion.md` | SOLID — Dependency Inversion Principle (DIP) | High-level modules should not depend on low-level modules — how to invert dependencies for flexible, testable code. |
| 19 | `19-dry-kiss-yagni.md` | DRY, KISS, YAGNI — The Clean Design Trinity | Don't Repeat Yourself, Keep It Simple, You Aren't Gonna Need It — three rules that prevent over-engineering. |
| 20 | `20-uml-class-diagrams.md` | UML Class Diagrams — How to Model Your Design | How to draw class diagrams with attributes, methods, visibility, and relationships — the visual language of LLD. |
| 21 | `21-uml-sequence-diagrams.md` | UML Sequence Diagrams — How to Model Interactions | How to draw the flow of method calls between objects over time — essential for explaining your design in interviews. |
| 22 | `22-uml-use-case-diagrams.md` | UML Use Case Diagrams — How to Model Requirements | How to capture what actors can do in a system — the bridge between requirements and design. |
| 23 | `23-class-relationships.md` | Class Relationships — Association, Aggregation, Composition, Inheritance | The 4 types of class relationships, when to use each, and the "has-a" vs "is-a" decision that shapes every design. |
| 24 | `24-coupling-and-cohesion.md` | Coupling and Cohesion — The Two Forces of Good Design | Why loose coupling + high cohesion = maintainable code, how to measure it, and refactoring toward better design. |
| 25 | `25-design-by-contract.md` | Design by Contract — Preconditions, Postconditions, Invariants | How to define clear agreements between classes — what each method promises and what it requires. |
| 26 | `26-composition-over-inheritance.md` | Composition Over Inheritance — Why and When | Why the Gang of Four said "favor composition" — concrete examples where inheritance fails and composition wins. |
| 27 | `27-abstraction-and-interfaces.md` | Abstraction and Interfaces in Design | How to identify the right abstractions, when to create interfaces, and how abstraction enables flexibility at scale. |

---

# PART 3 — LLD DESIGN PATTERNS (Topics 28–52)
*The battle-tested solutions to recurring design problems. Every principal engineer knows these cold.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 28 | `28-what-are-design-patterns.md` | What Are Design Patterns and Why They Exist | The history, categories (creational, structural, behavioral), and how to pick the right pattern for the right problem. |
| **Creational Patterns** | | | |
| 29 | `29-pattern-singleton.md` | Singleton Pattern | Ensure a class has only one instance — when it's necessary, when it's an anti-pattern, and thread-safe implementations. |
| 30 | `30-pattern-factory-method.md` | Factory Method Pattern | Delegate object creation to subclasses — how to create objects without specifying exact classes. |
| 31 | `31-pattern-abstract-factory.md` | Abstract Factory Pattern | Create families of related objects — when factory method isn't enough and you need a factory of factories. |
| 32 | `32-pattern-builder.md` | Builder Pattern | Construct complex objects step by step — how to avoid constructors with 15 parameters. |
| 33 | `33-pattern-prototype.md` | Prototype Pattern | Create new objects by cloning existing ones — when construction is expensive and copying is cheap. |
| **Structural Patterns** | | | |
| 34 | `34-pattern-adapter.md` | Adapter Pattern | Make incompatible interfaces work together — the "translator" between two systems that don't speak the same language. |
| 35 | `35-pattern-decorator.md` | Decorator Pattern | Add behavior to objects dynamically — how to extend functionality without modifying original classes. |
| 36 | `36-pattern-facade.md` | Facade Pattern | Provide a simplified interface to a complex subsystem — how to hide complexity behind a clean API. |
| 37 | `37-pattern-proxy.md` | Proxy Pattern | Control access to an object through a surrogate — lazy loading, access control, logging, and caching proxies. |
| 38 | `38-pattern-composite.md` | Composite Pattern | Treat individual objects and compositions uniformly — how to build tree structures (files/folders, UI components). |
| 39 | `39-pattern-bridge.md` | Bridge Pattern | Separate abstraction from implementation — how to avoid the "explosion of subclasses" problem. |
| 40 | `40-pattern-flyweight.md` | Flyweight Pattern | Share common state between many objects — how to handle millions of objects without running out of memory. |
| **Behavioral Patterns** | | | |
| 41 | `41-pattern-observer.md` | Observer Pattern | One-to-many dependency — when one object changes, all its dependents are notified (event systems, pub-sub). |
| 42 | `42-pattern-strategy.md` | Strategy Pattern | Define a family of algorithms and make them interchangeable — how to swap behavior at runtime. |
| 43 | `43-pattern-command.md` | Command Pattern | Encapsulate a request as an object — how to support undo, redo, queuing, and logging of operations. |
| 44 | `44-pattern-iterator.md` | Iterator Pattern | Access elements of a collection sequentially without exposing its underlying representation. |
| 45 | `45-pattern-template-method.md` | Template Method Pattern | Define the skeleton of an algorithm, letting subclasses fill in the steps — the "recipe" pattern. |
| 46 | `46-pattern-state.md` | State Pattern | Allow an object to change its behavior when its internal state changes — the "state machine" pattern. |
| 47 | `47-pattern-chain-of-responsibility.md` | Chain of Responsibility Pattern | Pass a request along a chain of handlers — middleware, validation pipelines, and approval workflows. |
| 48 | `48-pattern-mediator.md` | Mediator Pattern | Reduce chaotic dependencies between objects by centralizing communication — chat rooms, air traffic control. |
| 49 | `49-pattern-memento.md` | Memento Pattern | Capture and restore an object's state — how undo/redo and snapshots work internally. |
| 50 | `50-pattern-visitor.md` | Visitor Pattern | Add new operations to existing classes without modifying them — the double-dispatch trick. |
| **Concurrency Patterns** | | | |
| 51 | `51-pattern-producer-consumer.md` | Producer-Consumer Pattern | Decouple data production from consumption using a shared buffer — the foundation of message queues. |
| 52 | `52-concurrency-fundamentals.md` | Concurrency Fundamentals — Thread Pool, Mutex, Locks, Event Loop | Thread pools, mutexes, locks, semaphores, deadlocks, and how Node.js handles concurrency with its event loop. |

---

# PART 4 — HLD FUNDAMENTALS AND COMPONENTS (Topics 53–89)
*The building blocks of large-scale distributed systems. Every component a principal engineer reaches for when designing architecture.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 53 | `53-client-server-architecture.md` | Client-Server Architecture | How the client-server model works, thin vs thick clients, request-response lifecycle, and why everything is built on this. |
| 54 | `54-dns-deep-dive.md` | DNS — Domain Name System Deep Dive | How domain name resolution works end to end — recursive resolvers, root servers, TTL, DNS-based load balancing. |
| 55 | `55-load-balancing.md` | Load Balancing — Algorithms, L4 vs L7, Health Checks | Round robin, least connections, IP hash, weighted algorithms, Layer 4 vs Layer 7, active/passive health checks. |
| 56 | `56-horizontal-vs-vertical-scaling.md` | Horizontal vs Vertical Scaling | Scale out vs scale up — when to add more machines vs bigger machines, limits, costs, and real-world decisions. |
| 57 | `57-reverse-proxy.md` | Reverse Proxy — What It Is and How It Works | Forward proxy vs reverse proxy, Nginx as reverse proxy, SSL termination, compression, and how it differs from a load balancer. |
| 58 | `58-api-gateway.md` | API Gateway — The Front Door of Microservices | What an API gateway does, routing, authentication, rate limiting, request transformation, and how it differs from a reverse proxy. |
| 59 | `59-caching-in-depth.md` | Caching — Strategies, Layers, and Pitfalls | What to cache, where (client, CDN, server, DB), cache-aside, write-through, write-behind, read-through, eviction (LRU, LFU, FIFO), cache stampede, cache invalidation. |
| 60 | `60-cdn.md` | CDN — Content Delivery Networks | How CDNs work, push vs pull CDN, edge servers, cache invalidation, when to use, and how Netflix/YouTube use CDNs. |
| 61 | `61-databases-sql-vs-nosql.md` | Databases — SQL vs NoSQL Deep Dive | Relational vs document vs key-value vs column-family vs graph databases, ACID vs BASE, when to choose each. |
| 62 | `62-database-indexing.md` | Database Indexing — How Indexes Work Under the Hood | B-tree, B+ tree, hash indexes, composite indexes, covering indexes, when indexing helps, when it hurts. |
| 63 | `63-database-replication.md` | Database Replication — Master-Slave, Master-Master | How data is copied across servers, replication lag, read replicas, synchronous vs asynchronous replication, failover. |
| 64 | `64-database-sharding.md` | Database Sharding — Horizontal Partitioning | How to split data across multiple databases, sharding strategies (range, hash, directory), hotspot problem, resharding. |
| 65 | `65-database-connection-pooling.md` | Database Connection Pooling | Why connections are expensive, how pooling works, pool sizing, connection lifecycle, and common libraries. |
| 66 | `66-database-transactions-and-isolation.md` | Database Transactions and Isolation Levels | Read uncommitted, read committed, repeatable read, serializable — what anomalies each prevents and performance costs. |
| 67 | `67-message-queues.md` | Message Queues and Async Processing | Why queues exist, producers/consumers, Kafka vs RabbitMQ concepts, at-least-once vs at-most-once vs exactly-once delivery. |
| 68 | `68-event-driven-architecture.md` | Event-Driven Architecture | Events vs commands, event sourcing, CQRS (Command Query Responsibility Segregation), and when event-driven beats request-response. |
| 69 | `69-api-design-rest-graphql-grpc.md` | API Design — REST vs GraphQL vs gRPC | When to use each, REST maturity levels, GraphQL over-fetching solution, gRPC for inter-service, versioning, pagination strategies. |
| 70 | `70-rate-limiting.md` | Rate Limiting — Algorithms and Implementation | Token bucket, leaky bucket, fixed window, sliding window log, sliding window counter — how each works with diagrams. |
| 71 | `71-microservices-vs-monolith.md` | Microservices vs Monolith Architecture | When each makes sense, migration strategies, service boundaries, data ownership, the "distributed monolith" trap. |
| 72 | `72-service-discovery.md` | Service Discovery — How Services Find Each Other | Client-side vs server-side discovery, service registry, DNS-based discovery, Consul, etcd, Kubernetes service discovery. |
| 73 | `73-circuit-breaker.md` | Circuit Breaker Pattern | Closed, open, half-open states — how to prevent cascading failures in distributed systems, with real examples. |
| 74 | `74-consistent-hashing.md` | Consistent Hashing — The Key to Distributed Systems | The problem with naive hashing, hash rings, virtual nodes, real use in distributed caches and databases. |
| 75 | `75-data-partitioning.md` | Data Partitioning Strategies | Range-based, hash-based, directory-based, geographic partitioning — how to split data and handle cross-partition queries. |
| 76 | `76-eventual-consistency.md` | Eventual Consistency in Practice | What "eventually consistent" actually means, conflict resolution, vector clocks, last-writer-wins, CRDTs. |
| 77 | `77-distributed-transactions.md` | Distributed Transactions — 2PC and SAGA | Two-phase commit protocol, its limitations, SAGA pattern (choreography vs orchestration), compensating transactions. |
| 78 | `78-blob-storage.md` | Blob Storage — Object Storage at Scale | What blob storage is, how S3 works conceptually, multipart upload, pre-signed URLs, storage classes, lifecycle policies. |
| 79 | `79-search-systems.md` | Search Systems — How Search Engines Work | Inverted index, tokenization, relevance scoring (TF-IDF, BM25), Elasticsearch architecture, sharding search indexes. |
| 80 | `80-monitoring-and-observability.md` | Monitoring and Observability — Logs, Metrics, Traces | The three pillars, structured logging, Prometheus/Grafana, distributed tracing (Jaeger/Zipkin), alerting strategies. |
| 81 | `81-security-fundamentals.md` | Security Fundamentals in System Design | Authentication at scale, OAuth 2.0, JWT, API keys, encryption in transit (TLS) vs at rest, secrets management, OWASP top threats. |
| 82 | `82-content-negotiation-and-protocols.md` | Protocols — HTTP/2, HTTP/3, WebSockets, SSE, Long Polling | When to use each communication protocol, multiplexing, server push, bidirectional communication, real-time data patterns. |
| 83 | `83-leader-election.md` | Leader Election in Distributed Systems | Why distributed systems need leaders, Bully algorithm, Raft consensus basics, ZooKeeper leader election. |
| 84 | `84-distributed-locking.md` | Distributed Locking — Coordination Across Services | Why local locks don't work in distributed systems, Redis-based locks (Redlock), ZooKeeper locks, fencing tokens. |
| 85 | `85-idempotency.md` | Idempotency — Designing Safe Retries | What idempotency means, idempotency keys, why it matters for payments, how to make APIs idempotent. |
| 86 | `86-data-replication-strategies.md` | Data Replication Strategies — Quorum, Gossip, Anti-Entropy | Quorum reads/writes (W+R>N), gossip protocol, Merkle trees for anti-entropy, tunable consistency. |
| 87 | `87-batch-vs-stream-processing.md` | Batch Processing vs Stream Processing | MapReduce, Hadoop concepts, stream processing (Kafka Streams, Flink), Lambda architecture, Kappa architecture. |
| 88 | `88-containers-and-orchestration.md` | Containers and Orchestration — Docker and Kubernetes Concepts | What containers are, why they matter for system design, Kubernetes pods/services/deployments at a conceptual level. |
| 89 | `89-serverless-architecture.md` | Serverless Architecture — Functions as a Service | What serverless means, AWS Lambda concepts, cold starts, when serverless makes sense, limitations at scale. |
| 90 | `90-disaster-recovery.md` | Disaster Recovery and Backup Strategies | RPO vs RTO, hot/warm/cold standby, multi-region architecture, chaos engineering principles. |
| 91 | `91-data-lakes-and-warehouses.md` | Data Lakes and Data Warehouses | OLTP vs OLAP, data lake vs data warehouse vs data lakehouse, ETL vs ELT, columnar storage concepts. |
| 92 | `92-graph-databases-and-relationships.md` | Graph Databases and Relationship-Heavy Systems | When relational isn't enough, Neo4j concepts, social graphs, recommendation engines, shortest path queries. |

---

# PART 5 — HLD CASE STUDIES (Topics 93–108)
*Design real systems end to end — the way a principal engineer would in an interview or architecture review.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 93 | `93-hld-approach-framework.md` | How to Approach Any HLD Problem — The 5-Step Framework | Requirements → Estimation → High-level design → Deep dive → Bottlenecks — the proven method for every HLD interview. |
| 94 | `94-hld-url-shortener.md` | Design a URL Shortener (TinyURL) | Base62 encoding, key generation service, redirection, analytics, read-heavy system design. |
| 95 | `95-hld-rate-limiter.md` | Design a Rate Limiter | Distributed rate limiting, algorithm choices, Redis-based implementation, rules engine. |
| 96 | `96-hld-key-value-store.md` | Design a Key-Value Store (Distributed) | Partitioning, replication, consistency, failure detection, storage engine — build a mini DynamoDB. |
| 97 | `97-hld-pastebin.md` | Design a Pastebin | Content storage, unique URLs, expiration, read-heavy optimization, abuse prevention. |
| 98 | `98-hld-notification-system.md` | Design a Notification System | Push, SMS, email channels — prioritization, rate limiting, templates, delivery guarantees, fanout. |
| 99 | `99-hld-chat-system.md` | Design WhatsApp / Real-Time Chat | WebSocket connections, message delivery, group chat, online presence, message storage, end-to-end encryption concepts. |
| 100 | `100-hld-instagram.md` | Design Instagram / Photo Sharing | News feed generation, image storage/CDN, fanout on write vs fanout on read, celebrity problem. |
| 101 | `101-hld-youtube.md` | Design YouTube / Video Streaming | Video upload pipeline, transcoding, adaptive bitrate streaming, CDN, recommendation engine concepts. |
| 102 | `102-hld-uber.md` | Design Uber / Ride Sharing | Location tracking, driver matching, geospatial indexing (QuadTree, Geohash), surge pricing, ETA calculation. |
| 103 | `103-hld-twitter.md` | Design Twitter / Social Feed | Tweet fanout, timeline generation, trending topics, search, celebrity optimization, cache strategies. |
| 104 | `104-hld-amazon.md` | Design Amazon / E-Commerce Platform | Product catalog, cart, order pipeline, inventory, payment, search, recommendation — full e-commerce architecture. |
| 105 | `105-hld-distributed-cache.md` | Design a Distributed Cache (Redis) | Cache partitioning, replication, eviction, persistence, pub-sub, cluster mode — build Redis from concepts. |
| 106 | `106-hld-search-autocomplete.md` | Design a Search Autocomplete System | Trie data structure, ranking, real-time updates, distributed trie, typeahead service. |
| 107 | `107-hld-payment-system.md` | Design a Payment System | Payment flow, idempotency, double-entry ledger, reconciliation, fraud detection, PCI compliance concepts. |
| 108 | `108-hld-google-docs.md` | Design Google Docs / Collaborative Editing | Real-time collaboration, OT (Operational Transformation), CRDT, conflict resolution, WebSocket architecture. |
| 109 | `109-hld-web-crawler.md` | Design a Web Crawler | URL frontier, politeness, deduplication, distributed crawling, robots.txt, content parsing pipeline. |
| 110 | `110-hld-ticket-master.md` | Design Ticketmaster / Event Booking | Seat reservation, distributed locking, handling flash sales, virtual queue, payment timeout, eventual consistency. |

---

# PART 6 — LLD CASE STUDIES (Topics 111–132)
*Design real systems class by class — every class, method, and relationship, with JavaScript/Node.js code.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 111 | `111-lld-approach-framework.md` | How to Approach Any LLD Problem — Step by Step | Requirements → Identify objects → Define relationships → Apply patterns → Write code — the LLD interview method. |
| 112 | `112-lld-parking-lot.md` | Design a Parking Lot System | Vehicles, spots, floors, ticketing, payment — a classic starter LLD problem with full class hierarchy. |
| 113 | `113-lld-library-management.md` | Design a Library Management System | Books, members, borrowing, reservations, fines — modeling entities and their interactions. |
| 114 | `114-lld-atm-machine.md` | Design an ATM Machine | States (idle, card inserted, PIN entered, transaction), state pattern, transaction handling. |
| 115 | `115-lld-elevator-system.md` | Design an Elevator System | Scheduling algorithms (SCAN, LOOK), multiple elevators, request queuing, state management. |
| 116 | `116-lld-vending-machine.md` | Design a Vending Machine | State pattern, inventory management, payment processing, product dispensing. |
| 117 | `117-lld-hotel-booking.md` | Design a Hotel Booking System | Rooms, reservations, availability, pricing, cancellation policies, search and filter. |
| 118 | `118-lld-movie-ticket-booking.md` | Design BookMyShow / Movie Ticket Booking | Theaters, shows, seat selection, concurrent booking, payment, locking strategies. |
| 119 | `119-lld-chess-game.md` | Design a Chess Game | Board, pieces, move validation, check/checkmate detection, game state management. |
| 120 | `120-lld-snake-and-ladder.md` | Design a Snake and Ladder Game | Board generation, dice, player turns, snakes/ladders, win detection — modeling game logic. |
| 121 | `121-lld-splitwise.md` | Design Splitwise / Expense Sharing | Users, groups, expenses, split strategies, debt simplification algorithm, balance tracking. |
| 122 | `122-lld-logging-framework.md` | Design a Logging Framework | Log levels, formatters, handlers/appenders, sinks, singleton logger, strategy pattern for output. |
| 123 | `123-lld-rate-limiter.md` | Design a Rate Limiter (Class Level) | Token bucket and sliding window as classes, interface design, pluggable algorithms. |
| 124 | `124-lld-lru-cache.md` | Design an LRU Cache | Doubly linked list + hash map, O(1) get/put, eviction, capacity management — a top interview question. |
| 125 | `125-lld-notification-service.md` | Design a Notification Service (Class Level) | Notification types, channels, templates, observer pattern, priority queue, delivery tracking. |
| 126 | `126-lld-food-delivery.md` | Design a Food Delivery App (Zomato/Swiggy) | Restaurants, menus, orders, delivery agents, order tracking, pricing, assignment algorithm. |
| 127 | `127-lld-uber-matching.md` | Design Uber Driver-Rider Matching (Class Level) | Location, matching algorithm, ride states, fare calculation, trip management. |
| 128 | `128-lld-social-media-feed.md` | Design a Social Media Feed (Class Level) | Posts, users, feed generation, ranking, pagination, like/comment modeling. |
| 129 | `129-lld-api-gateway.md` | Design an API Gateway (Class Level) | Routing, middleware chain, authentication, rate limiting, request/response transformation. |
| 130 | `130-lld-task-scheduler.md` | Design a Task Scheduler | Cron-like scheduling, priority queue, delayed execution, recurring tasks, retry logic. |
| 131 | `131-lld-online-auction.md` | Design an Online Auction System (eBay) | Bidding, auction lifecycle, bid validation, auto-bidding, winner determination, concurrent bid handling. |
| 132 | `132-lld-file-system.md` | Design an In-Memory File System | Files, directories, composite pattern, path resolution, permissions, CRUD operations. |

---

# PART 7 — ADVANCED TOPICS (Topics 133–140)
*Topics that separate senior engineers from principal engineers.*

| # | File | Topic | What You'll Learn |
|---|------|-------|-------------------|
| 133 | `133-system-design-trade-off-matrix.md` | The System Design Trade-Off Matrix | A comprehensive matrix of all major trade-offs — consistency vs availability, latency vs throughput, etc. — how to reason about trade-offs like a principal engineer. |
| 134 | `134-multi-region-architecture.md` | Multi-Region Architecture and Global Systems | Active-active vs active-passive, data sovereignty, global load balancing, cross-region replication. |
| 135 | `135-zero-downtime-deployments.md` | Zero Downtime Deployments | Blue-green deployments, canary releases, rolling updates, feature flags, database migrations without downtime. |
| 136 | `136-cost-optimization.md` | Cost Optimization in System Design | How to estimate infrastructure costs, reserved vs on-demand, caching ROI, right-sizing, the cost dimension of every design. |
| 137 | `137-performance-optimization.md` | Performance Optimization Patterns | Connection pooling, batching, compression, lazy loading, pre-computation, denormalization for read performance. |
| 138 | `138-testing-distributed-systems.md` | Testing Distributed Systems | Integration testing, contract testing, chaos engineering, load testing, fault injection — how to verify your design works. |
| 139 | `139-evolutionary-architecture.md` | Evolutionary Architecture — Designing for Change | How to build systems that can evolve, fitness functions, avoiding big-bang rewrites, strangler fig pattern. |
| 140 | `140-system-design-interview-playbook.md` | System Design Interview Playbook — The Complete Guide | Time management, communication framework, common mistakes, scoring rubric, how to practice, and a final checklist. |

---

# SUMMARY

| Part | Topics | Range | Focus |
|------|--------|-------|-------|
| Part 1 — Foundations | 12 | 01–12 | Core vocabulary and mental models |
| Part 2 — LLD Foundations | 15 | 13–27 | OOP, SOLID, UML, relationships |
| Part 3 — LLD Design Patterns | 25 | 28–52 | All major design patterns with JS code |
| Part 4 — HLD Fundamentals | 40 | 53–92 | Every building block of distributed systems |
| Part 5 — HLD Case Studies | 18 | 93–110 | Design real systems end to end |
| Part 6 — LLD Case Studies | 22 | 111–132 | Design real systems class by class |
| Part 7 — Advanced Topics | 8 | 133–140 | Principal engineer level mastery |
| **TOTAL** | **140** | | |

---

# COMPLETE FILE LIST

```
/docs/system-design/CURRICULUM.md                          ← This file (the master plan)
/docs/system-design/01-what-is-system-design.md
/docs/system-design/02-hld-vs-lld.md
/docs/system-design/03-how-to-approach-system-design.md
/docs/system-design/04-functional-vs-non-functional-requirements.md
/docs/system-design/05-capacity-estimation.md
/docs/system-design/06-latency-and-throughput.md
/docs/system-design/07-availability-and-reliability.md
/docs/system-design/08-consistency-models.md
/docs/system-design/09-cap-theorem.md
/docs/system-design/10-networking-fundamentals.md
/docs/system-design/11-how-the-web-works.md
/docs/system-design/12-numbers-every-engineer-should-know.md
/docs/system-design/13-oop-four-pillars.md
/docs/system-design/14-solid-single-responsibility.md
/docs/system-design/15-solid-open-closed.md
/docs/system-design/16-solid-liskov-substitution.md
/docs/system-design/17-solid-interface-segregation.md
/docs/system-design/18-solid-dependency-inversion.md
/docs/system-design/19-dry-kiss-yagni.md
/docs/system-design/20-uml-class-diagrams.md
/docs/system-design/21-uml-sequence-diagrams.md
/docs/system-design/22-uml-use-case-diagrams.md
/docs/system-design/23-class-relationships.md
/docs/system-design/24-coupling-and-cohesion.md
/docs/system-design/25-design-by-contract.md
/docs/system-design/26-composition-over-inheritance.md
/docs/system-design/27-abstraction-and-interfaces.md
/docs/system-design/28-what-are-design-patterns.md
/docs/system-design/29-pattern-singleton.md
/docs/system-design/30-pattern-factory-method.md
/docs/system-design/31-pattern-abstract-factory.md
/docs/system-design/32-pattern-builder.md
/docs/system-design/33-pattern-prototype.md
/docs/system-design/34-pattern-adapter.md
/docs/system-design/35-pattern-decorator.md
/docs/system-design/36-pattern-facade.md
/docs/system-design/37-pattern-proxy.md
/docs/system-design/38-pattern-composite.md
/docs/system-design/39-pattern-bridge.md
/docs/system-design/40-pattern-flyweight.md
/docs/system-design/41-pattern-observer.md
/docs/system-design/42-pattern-strategy.md
/docs/system-design/43-pattern-command.md
/docs/system-design/44-pattern-iterator.md
/docs/system-design/45-pattern-template-method.md
/docs/system-design/46-pattern-state.md
/docs/system-design/47-pattern-chain-of-responsibility.md
/docs/system-design/48-pattern-mediator.md
/docs/system-design/49-pattern-memento.md
/docs/system-design/50-pattern-visitor.md
/docs/system-design/51-pattern-producer-consumer.md
/docs/system-design/52-concurrency-fundamentals.md
/docs/system-design/53-client-server-architecture.md
/docs/system-design/54-dns-deep-dive.md
/docs/system-design/55-load-balancing.md
/docs/system-design/56-horizontal-vs-vertical-scaling.md
/docs/system-design/57-reverse-proxy.md
/docs/system-design/58-api-gateway.md
/docs/system-design/59-caching-in-depth.md
/docs/system-design/60-cdn.md
/docs/system-design/61-databases-sql-vs-nosql.md
/docs/system-design/62-database-indexing.md
/docs/system-design/63-database-replication.md
/docs/system-design/64-database-sharding.md
/docs/system-design/65-database-connection-pooling.md
/docs/system-design/66-database-transactions-and-isolation.md
/docs/system-design/67-message-queues.md
/docs/system-design/68-event-driven-architecture.md
/docs/system-design/69-api-design-rest-graphql-grpc.md
/docs/system-design/70-rate-limiting.md
/docs/system-design/71-microservices-vs-monolith.md
/docs/system-design/72-service-discovery.md
/docs/system-design/73-circuit-breaker.md
/docs/system-design/74-consistent-hashing.md
/docs/system-design/75-data-partitioning.md
/docs/system-design/76-eventual-consistency.md
/docs/system-design/77-distributed-transactions.md
/docs/system-design/78-blob-storage.md
/docs/system-design/79-search-systems.md
/docs/system-design/80-monitoring-and-observability.md
/docs/system-design/81-security-fundamentals.md
/docs/system-design/82-content-negotiation-and-protocols.md
/docs/system-design/83-leader-election.md
/docs/system-design/84-distributed-locking.md
/docs/system-design/85-idempotency.md
/docs/system-design/86-data-replication-strategies.md
/docs/system-design/87-batch-vs-stream-processing.md
/docs/system-design/88-containers-and-orchestration.md
/docs/system-design/89-serverless-architecture.md
/docs/system-design/90-disaster-recovery.md
/docs/system-design/91-data-lakes-and-warehouses.md
/docs/system-design/92-graph-databases-and-relationships.md
/docs/system-design/93-hld-approach-framework.md
/docs/system-design/94-hld-url-shortener.md
/docs/system-design/95-hld-rate-limiter.md
/docs/system-design/96-hld-key-value-store.md
/docs/system-design/97-hld-pastebin.md
/docs/system-design/98-hld-notification-system.md
/docs/system-design/99-hld-chat-system.md
/docs/system-design/100-hld-instagram.md
/docs/system-design/101-hld-youtube.md
/docs/system-design/102-hld-uber.md
/docs/system-design/103-hld-twitter.md
/docs/system-design/104-hld-amazon.md
/docs/system-design/105-hld-distributed-cache.md
/docs/system-design/106-hld-search-autocomplete.md
/docs/system-design/107-hld-payment-system.md
/docs/system-design/108-hld-google-docs.md
/docs/system-design/109-hld-web-crawler.md
/docs/system-design/110-hld-ticket-master.md
/docs/system-design/111-lld-approach-framework.md
/docs/system-design/112-lld-parking-lot.md
/docs/system-design/113-lld-library-management.md
/docs/system-design/114-lld-atm-machine.md
/docs/system-design/115-lld-elevator-system.md
/docs/system-design/116-lld-vending-machine.md
/docs/system-design/117-lld-hotel-booking.md
/docs/system-design/118-lld-movie-ticket-booking.md
/docs/system-design/119-lld-chess-game.md
/docs/system-design/120-lld-snake-and-ladder.md
/docs/system-design/121-lld-splitwise.md
/docs/system-design/122-lld-logging-framework.md
/docs/system-design/123-lld-rate-limiter.md
/docs/system-design/124-lld-lru-cache.md
/docs/system-design/125-lld-notification-service.md
/docs/system-design/126-lld-food-delivery.md
/docs/system-design/127-lld-uber-matching.md
/docs/system-design/128-lld-social-media-feed.md
/docs/system-design/129-lld-api-gateway.md
/docs/system-design/130-lld-task-scheduler.md
/docs/system-design/131-lld-online-auction.md
/docs/system-design/132-lld-file-system.md
/docs/system-design/133-system-design-trade-off-matrix.md
/docs/system-design/134-multi-region-architecture.md
/docs/system-design/135-zero-downtime-deployments.md
/docs/system-design/136-cost-optimization.md
/docs/system-design/137-performance-optimization.md
/docs/system-design/138-testing-distributed-systems.md
/docs/system-design/139-evolutionary-architecture.md
/docs/system-design/140-system-design-interview-playbook.md
```

---

# TOPICS I ADDED BEYOND YOUR ORIGINAL LIST

These were missing from your curriculum but are critical for principal-engineer-level understanding:

1. **Topic 10 — Networking Fundamentals** — You can't design systems without understanding TCP/UDP/HTTP/WebSockets
2. **Topic 11 — How the Web Works** — The full request lifecycle is assumed knowledge in every interview
3. **Topic 12 — Numbers Every Engineer Should Know** — Jeff Dean's famous latency numbers — required for estimation
4. **Topic 26 — Composition Over Inheritance** — The most important OOP design principle after SOLID
5. **Topic 27 — Abstraction and Interfaces** — Critical for designing extensible systems
6. **Topic 66 — Database Transactions and Isolation** — Can't design databases without understanding isolation levels
7. **Topic 68 — Event-Driven Architecture** — Crucial for modern distributed systems, not just message queues
8. **Topic 82 — Protocols (HTTP/2, WebSocket, SSE)** — Must know which protocol to pick for real-time vs batch
9. **Topic 83 — Leader Election** — Fundamental distributed systems concept for consensus
10. **Topic 84 — Distributed Locking** — Required for any system with concurrent resource access
11. **Topic 85 — Idempotency** — Critical for payment systems and retry-safe APIs
12. **Topic 86 — Data Replication Strategies** — Quorum, gossip protocol — advanced but essential
13. **Topic 87 — Batch vs Stream Processing** — MapReduce, Kafka Streams — data pipeline design
14. **Topic 88 — Containers and Kubernetes** — Can't discuss modern architecture without containers
15. **Topic 89 — Serverless Architecture** — Important trade-off discussion for modern systems
16. **Topic 90 — Disaster Recovery** — RPO/RTO is asked in every senior design interview
17. **Topic 91 — Data Lakes and Warehouses** — OLTP vs OLAP, the analytics side of system design
18. **Topic 92 — Graph Databases** — When relational doesn't fit — social networks, recommendations
19. **Topic 108 — Design Google Docs** — Collaborative editing is a top-tier HLD question
20. **Topic 109 — Design a Web Crawler** — Classic distributed systems problem
21. **Topic 110 — Design Ticketmaster** — Flash sale / booking problem with real concurrency challenges
22. **Topic 130 — Design a Task Scheduler** — Important LLD problem for cron/background jobs
23. **Topic 131 — Design an Online Auction** — Bidding systems have unique concurrency patterns
24. **Topic 132 — Design an In-Memory File System** — Composite pattern + real interview favorite
25. **Topics 133–140 — Advanced Topics** — Trade-off matrix, multi-region, zero-downtime deployments, cost, performance, testing, evolutionary architecture, and the final interview playbook

---

> **Does this plan look good? Type START to begin topic 01, or tell me if you want to add or change anything.**
