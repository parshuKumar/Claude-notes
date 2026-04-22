# System Design Complete Curriculum Plan
### From Zero to Principal Engineer Level

> This is our contract. Every topic below becomes exactly one file in `/docs/system-design/`.
> We follow this order. One file at a time.

---

## PHASE 1 — FOUNDATIONS
*You must understand these before touching any real design. These are the vocabulary and mental models every engineer uses.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 01 | `01-what-is-system-design.md` | What Is System Design? | What the discipline is, what problems it solves, why every senior engineer must know it, how it differs from coding |
| 02 | `02-hld-vs-lld.md` | HLD vs LLD — The Big Picture | The difference between designing a city (HLD) and designing a building's floor plan (LLD), when each is used, how they connect |
| 03 | `03-how-to-approach-system-design.md` | How to Approach Any System Design Problem | The universal framework: clarify → estimate → design → deep-dive → trade-offs. How to structure your thinking in interviews and at work |
| 04 | `04-functional-vs-nonfunctional-requirements.md` | Functional vs Non-Functional Requirements | How to gather requirements before drawing a single box, what FRs vs NFRs mean, why missing NFRs is what kills real systems |
| 05 | `05-capacity-estimation.md` | Capacity Estimation & Back-of-Envelope Math | How to estimate storage, bandwidth, and request rates from scratch, the numbers every engineer memorizes, worked examples |
| 06 | `06-latency-vs-throughput.md` | Latency vs Throughput | What latency and throughput mean, why they are often in conflict, real numbers to remember (L1 cache, disk, network), how to optimize each |
| 07 | `07-availability-and-reliability.md` | Availability, Reliability, and Fault Tolerance | What 99.9% vs 99.999% uptime actually means in minutes of downtime per year, SLA vs SLO vs SLI, how systems stay alive when things fail |
| 08 | `08-consistency-vs-availability.md` | Consistency vs Availability — The Core Tension | Why you cannot always have both, what it means when data is "stale", the intuition behind the CAP theorem before we go deep |
| 09 | `09-networking-basics-for-engineers.md` | Networking Basics Every Engineer Must Know | TCP vs UDP, HTTP vs HTTPS, how a request travels from your browser to a server and back, status codes, what a socket is, keep-alive connections |
| 10 | `10-how-the-internet-works.md` | How the Internet Actually Works | IP addresses, ports, routers, what happens when you type google.com, packets, bandwidth — the physical and logical layers simplified |

---

## PHASE 2 — LLD FOUNDATIONS
*Before you can design systems with classes and patterns, you need deep OOP and design principles. Most people think they know OOP — but principal-level OOP is different.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 11 | `11-oop-four-pillars-deep.md` | OOP — The Four Pillars with Design Focus | Encapsulation, Abstraction, Inheritance, Polymorphism — not just definitions but WHY each exists, what breaks without it, design-level examples |
| 12 | `12-solid-principles.md` | SOLID Principles — Each One Deeply Explained | SRP, OCP, LSP, ISP, DIP — every principle with a bad code example (what it prevents) and a good code example (how to fix it), in JavaScript |
| 13 | `13-dry-kiss-yagni.md` | DRY, KISS, YAGNI — Clean Design Rules | Don't Repeat Yourself, Keep It Simple, You Aren't Gonna Need It — when to apply each, when NOT to, common beginner mistakes |
| 14 | `14-uml-diagrams.md` | UML Diagrams — How to Draw System Blueprints | Class diagrams, sequence diagrams, use-case diagrams — what each shows, the notation, how to read and draw them, when each is used |
| 15 | `15-class-relationships.md` | Class Relationships — Association, Aggregation, Composition, Inheritance | The four ways classes relate to each other, the difference between "has-a" and "is-a", when composition beats inheritance, real examples |
| 16 | `16-coupling-and-cohesion.md` | Coupling and Cohesion | Why loose coupling and high cohesion are the goal of every LLD, how to measure them, what tightly coupled code looks like and why it fails |
| 17 | `17-design-by-contract.md` | Design by Contract — Preconditions, Postconditions, Invariants | How to think about a function's promises and guarantees, how this thinking prevents bugs at the design stage, how it connects to interfaces |
| 18 | `18-interfaces-vs-abstract-classes.md` | Interfaces vs Abstract Classes | When to use each, how they enforce contracts, the "program to an interface" principle, practical JavaScript/Node.js examples |
| 19 | `19-composition-over-inheritance.md` | Composition Over Inheritance | Why deep inheritance hierarchies fail, how to replace them with composition, the decorator and strategy pattern intuition |
| 20 | `20-error-handling-design.md` | Error Handling as a Design Decision | How to design error handling at the class and module level, custom error types, error propagation, fail-fast vs graceful degradation |

---

## PHASE 3 — LLD DESIGN PATTERNS
*Patterns are proven solutions to common design problems. They are the vocabulary of LLD interviews. Every senior engineer knows these cold.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 21 | `21-what-are-design-patterns.md` | What Are Design Patterns and Why They Exist | The history (Gang of Four), why patterns exist, how to identify when to use one, the three categories — with JavaScript context |
| 22 | `22-pattern-singleton.md` | Creational — Singleton Pattern | One instance only, global access, thread safety concerns, when it is appropriate vs when it is an anti-pattern, JS implementation |
| 23 | `23-pattern-factory-method.md` | Creational — Factory Method Pattern | Delegate object creation to subclasses, decoupling creation from usage, when to prefer this over `new`, JS implementation |
| 24 | `24-pattern-abstract-factory.md` | Creational — Abstract Factory Pattern | Factory of factories, creating families of related objects, real-world example (UI themes, DB drivers), JS implementation |
| 25 | `25-pattern-builder.md` | Creational — Builder Pattern | Building complex objects step-by-step, fluent interface, separating construction from representation, JS implementation |
| 26 | `26-pattern-prototype.md` | Creational — Prototype Pattern | Clone existing objects instead of creating from scratch, deep vs shallow copy, when this is the right tool, JS implementation |
| 27 | `27-pattern-adapter.md` | Structural — Adapter Pattern | Make incompatible interfaces work together, wrapping old code for new systems, real example: payment gateway adapters, JS implementation |
| 28 | `28-pattern-decorator.md` | Structural — Decorator Pattern | Add behavior to objects dynamically without changing their class, the middleware pattern in Node.js is a decorator, JS implementation |
| 29 | `29-pattern-facade.md` | Structural — Facade Pattern | Provide a simple interface to a complex subsystem, reduce client complexity, the "one entry point" idea, JS implementation |
| 30 | `30-pattern-proxy.md` | Structural — Proxy Pattern | Control access to an object, lazy loading, caching, logging, access control — how a proxy intercepts operations, JS implementation |
| 31 | `31-pattern-composite.md` | Structural — Composite Pattern | Treat individual objects and groups of objects uniformly, tree structures, file system and UI component examples, JS implementation |
| 32 | `32-pattern-bridge.md` | Structural — Bridge Pattern | Decouple abstraction from implementation so both can vary independently, when to use over inheritance, JS implementation |
| 33 | `33-pattern-flyweight.md` | Structural — Flyweight Pattern | Share common state among many objects to save memory, the intrinsic vs extrinsic state idea, game objects example, JS implementation |
| 34 | `34-pattern-observer.md` | Behavioral — Observer Pattern | Notify many objects when one object changes, event-driven architecture foundation, Node.js EventEmitter is an observer, JS implementation |
| 35 | `35-pattern-strategy.md` | Behavioral — Strategy Pattern | Define a family of algorithms, encapsulate each, make them interchangeable — swap behavior at runtime, JS implementation |
| 36 | `36-pattern-command.md` | Behavioral — Command Pattern | Encapsulate a request as an object — undo/redo, queuing, logging operations, JS implementation |
| 37 | `37-pattern-iterator.md` | Behavioral — Iterator Pattern | Access elements of a collection without knowing its internal structure, how JavaScript's for-of and generators implement this natively |
| 38 | `38-pattern-template-method.md` | Behavioral — Template Method Pattern | Define the skeleton of an algorithm in a base class, let subclasses fill in the steps, the "hook" concept, JS implementation |
| 39 | `39-pattern-state.md` | Behavioral — State Pattern | Allow an object to change its behavior when its internal state changes, state machines, vending machine and order state examples, JS implementation |
| 40 | `40-pattern-chain-of-responsibility.md` | Behavioral — Chain of Responsibility | Pass a request along a chain of handlers until one handles it — middleware pipelines in Express.js are this pattern, JS implementation |
| 41 | `41-pattern-mediator.md` | Behavioral — Mediator Pattern | Define an object that encapsulates how objects interact, reduce direct dependencies between many objects, chat room example, JS implementation |
| 42 | `42-pattern-memento.md` | Behavioral — Memento Pattern | Capture and restore an object's internal state without violating encapsulation — undo history, save/restore, JS implementation |
| 43 | `43-pattern-visitor.md` | Behavioral — Visitor Pattern | Add operations to objects without modifying their classes, the double dispatch trick, when to use over polymorphism, JS implementation |
| 44 | `44-concurrency-patterns.md` | Concurrency Patterns — Producer-Consumer, Thread Pool, Mutex | How concurrent systems coordinate, the producer-consumer queue, thread pool concept, race conditions, how Node.js event loop relates |
| 45 | `45-pattern-event-sourcing-cqrs-intro.md` | Event Sourcing & CQRS — Introduction | Store events not state, separate reads from writes, how this becomes critical in distributed systems — a bridge from LLD patterns to HLD |

---

## PHASE 4 — HLD FUNDAMENTALS & COMPONENTS
*The building blocks of every large-scale system. Every box on a system design whiteboard comes from this phase.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 46 | `46-client-server-architecture.md` | Client-Server Architecture | How clients and servers communicate, thin vs thick clients, request-response cycle, stateless vs stateful servers, real examples |
| 47 | `47-dns-explained.md` | DNS — Domain Name Resolution End to End | What happens from typing a URL to a server responding, recursive vs iterative resolution, TTL, DNS caching, DNS as a failure point |
| 48 | `48-load-balancing.md` | Load Balancing — Algorithms, L4 vs L7, Health Checks | What a load balancer does, round-robin, least-connections, IP-hash, weighted, consistent hashing, L4 vs L7, sticky sessions, health checks |
| 49 | `49-horizontal-vs-vertical-scaling.md` | Horizontal vs Vertical Scaling | Scale up vs scale out, when each works, limits of each, stateless design as a prerequisite for horizontal scaling, real cost trade-offs |
| 50 | `50-caching-complete.md` | Caching — The Complete Guide | What to cache and where (client, CDN, server, DB), cache-aside, read-through, write-through, write-behind, eviction policies (LRU, LFU, FIFO, TTL), cache stampede, thundering herd, hot key problem |
| 51 | `51-cdn.md` | CDN — Content Delivery Networks | What a CDN is, how it works, PoPs, push vs pull CDN, cache invalidation, when to use, when not to, real examples (Cloudflare, Akamai) |
| 52 | `52-sql-vs-nosql.md` | SQL vs NoSQL — Deep Dive | Relational vs non-relational, ACID vs BASE, when to use each, the 4 types of NoSQL (document, key-value, column-family, graph), trade-offs |
| 53 | `53-database-indexing.md` | Database Indexing — How Indexes Actually Work | What an index is, B-tree index internals, clustered vs non-clustered, composite indexes, when indexes help vs hurt, covering index, index bloat |
| 54 | `54-database-replication.md` | Database Replication | Primary-replica (master-slave), multi-primary, synchronous vs asynchronous replication, replication lag, read replicas, failover |
| 55 | `55-database-sharding.md` | Database Sharding | Horizontal partitioning, range-based, hash-based, directory-based sharding, hotspot problem, cross-shard queries, resharding challenges |
| 56 | `56-database-connection-pooling.md` | Database Connection Pooling | Why connections are expensive, what a pool is, pool sizing, connection leak, pgBouncer concept, Node.js pool libraries |
| 57 | `57-acid-vs-base.md` | ACID vs BASE — Transactions and Consistency Models | Atomicity, Consistency, Isolation, Durability in depth, isolation levels (read uncommitted → serializable), BASE as the NoSQL trade-off |
| 58 | `58-message-queues.md` | Message Queues and Async Processing | Why queues exist, decoupling producers and consumers, Kafka vs RabbitMQ conceptual comparison, at-least-once vs exactly-once vs at-most-once delivery, dead letter queues |
| 59 | `59-api-design.md` | API Design — REST vs GraphQL vs gRPC | REST principles, GraphQL query model, gRPC and protobuf, versioning strategies, pagination (cursor vs offset), idempotency, API contracts |
| 60 | `60-rate-limiting.md` | Rate Limiting — Algorithms and Implementation | Token bucket, leaky bucket, fixed window counter, sliding window log, sliding window counter — how each works, trade-offs, where to implement |
| 61 | `61-reverse-proxy.md` | Reverse Proxy — What It Is and Why It Matters | Forward vs reverse proxy, nginx as reverse proxy, SSL termination, request routing, how it differs from a load balancer |
| 62 | `62-api-gateway.md` | API Gateway — The Front Door to Microservices | What an API gateway does, authentication, rate limiting, routing, request transformation, how it differs from a reverse proxy, Kong / AWS API Gateway examples |
| 63 | `63-microservices-vs-monolith.md` | Microservices vs Monolith | When a monolith is correct, when to split, the strangler fig pattern, inter-service communication (sync vs async), the failure modes of microservices |
| 64 | `64-service-discovery.md` | Service Discovery — How Services Find Each Other | The problem of dynamic IPs in containers, client-side vs server-side discovery, service registry (Consul, Eureka), DNS-based discovery |
| 65 | `65-circuit-breaker.md` | Circuit Breaker Pattern | Cascading failure problem, the three states (closed, open, half-open), how it prevents system-wide collapse, Hystrix / Resilience4j concepts |
| 66 | `66-consistent-hashing.md` | Consistent Hashing | The problem with naive hash-based sharding, the ring concept, virtual nodes, what happens when a node joins or leaves, used in Cassandra, DynamoDB, CDNs |
| 67 | `67-cap-theorem-deep.md` | CAP Theorem — Deep Dive | You can only pick 2 of 3, partition tolerance is not optional, CP systems vs AP systems, real database examples of each, how to answer CAP questions in interviews |
| 68 | `68-eventual-consistency.md` | Eventual Consistency — What It Actually Means | Stale reads, eventual convergence, conflict resolution (last-write-wins, vector clocks), real examples (DNS, shopping cart, social feed likes) |
| 69 | `69-distributed-transactions.md` | Distributed Transactions — 2PC and SAGA | The problem of transactions across services, Two-Phase Commit (coordinator, participants, what happens on failure), SAGA pattern (choreography vs orchestration) |
| 70 | `70-data-partitioning.md` | Data Partitioning Strategies | Range partitioning, hash partitioning, list partitioning, directory-based, composite partitioning, how to choose, hotspot problem in each |
| 71 | `71-blob-storage.md` | Blob Storage — Object Storage and S3 | What object storage is, how S3 works conceptually (buckets, objects, keys), pre-signed URLs, multipart upload, use cases, vs block storage vs file storage |
| 72 | `72-search-systems.md` | Search Systems — Elasticsearch and Inverted Index | How full-text search works, what an inverted index is, tokenization, relevance scoring, when to use a dedicated search engine vs DB search |
| 73 | `73-monitoring-and-observability.md` | Monitoring and Observability — Logs, Metrics, Traces | The three pillars of observability, structured logging, time-series metrics, distributed tracing, alerting strategy, SLI/SLO/SLA, on-call practices |
| 74 | `74-security-in-system-design.md` | Security in System Design | Authentication at scale (session vs JWT vs OAuth2), API key management, mTLS, encryption in transit vs at rest, secrets management, OWASP Top 10 for system designers |
| 75 | `75-websockets-and-real-time.md` | WebSockets and Real-Time Communication | HTTP polling vs long-polling vs WebSockets vs SSE, when to use each, connection management at scale, how WhatsApp and Slack handle real-time |
| 76 | `76-event-driven-architecture.md` | Event-Driven Architecture | Events vs commands vs queries, pub-sub model, event bus, event streaming (Kafka), choreography, how this scales vs request-response |
| 77 | `77-data-pipelines-and-stream-processing.md` | Data Pipelines and Stream Processing | Batch vs stream processing, ETL pipelines, Apache Kafka Streams concept, Lambda architecture, use cases (analytics, fraud detection, recommendations) |
| 78 | `78-distributed-file-systems.md` | Distributed File Systems | HDFS concepts, how files are split into blocks and replicated, the namenode/datanode model, GFS paper concepts, use cases |
| 79 | `79-consensus-algorithms.md` | Consensus Algorithms — Raft and Paxos | The problem of agreement in distributed systems, what a leader election is, how Raft achieves consensus, how etcd and ZooKeeper use this |
| 80 | `80-service-mesh.md` | Service Mesh — Istio and Envoy Concepts | What a service mesh is, sidecar proxy pattern, how it solves observability and security between microservices, when you need it |
| 81 | `81-containers-and-orchestration.md` | Containers and Orchestration — Docker and Kubernetes Concepts | What containers are, why they changed deployment, Kubernetes pod/service/deployment concepts, how k8s affects system design decisions |
| 82 | `82-geo-distributed-systems.md` | Geo-Distributed Systems and Multi-Region Design | Why you go multi-region, active-active vs active-passive, data residency, global load balancing, cross-region replication challenges |
| 83 | `83-disaster-recovery.md` | Disaster Recovery and Business Continuity | RTO vs RPO, backup strategies, hot/warm/cold standby, chaos engineering, runbooks, what "recovery" actually looks like in production |
| 84 | `84-serverless-architecture.md` | Serverless Architecture | What FaaS is (AWS Lambda), cold start problem, when serverless wins vs loses, stateless function design, event-driven serverless patterns |

---

## PHASE 5 — HLD CASE STUDIES
*Apply everything you've learned by designing real systems from scratch. This is what interviews test.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 85 | `85-hld-approach-framework.md` | How to Approach Any HLD Problem — The 5-Step Framework | The repeatable process: clarify → estimate → high-level design → deep dive → bottleneck analysis. How to lead a design conversation |
| 86 | `86-hld-url-shortener.md` | HLD Case Study — URL Shortener (TinyURL) | Hashing strategies, redirect flow, analytics, scaling reads, storage estimation — the classic beginner HLD |
| 87 | `87-hld-rate-limiter.md` | HLD Case Study — Rate Limiter Service | Distributed rate limiting, where to store counters (Redis), sliding window in a distributed system, per-user vs per-IP limiting |
| 88 | `88-hld-key-value-store.md` | HLD Case Study — Distributed Key-Value Store | Consistent hashing, replication factor, quorum reads/writes, conflict resolution, the DynamoDB/Cassandra design |
| 89 | `89-hld-pastebin.md` | HLD Case Study — Pastebin | Unique key generation, content storage (blob), read-heavy optimization, expiration, analytics |
| 90 | `90-hld-notification-system.md` | HLD Case Study — Notification System | Push vs pull, FCM/APNS/email/SMS channels, fan-out problem, priority queues, delivery guarantees, notification preferences |
| 91 | `91-hld-whatsapp.md` | HLD Case Study — WhatsApp / Chat System | Message storage, real-time delivery (WebSockets), message ordering, group chats, online presence, end-to-end encryption concepts, read receipts |
| 92 | `92-hld-instagram.md` | HLD Case Study — Instagram / Photo Sharing | Upload flow, CDN for images/video, feed generation (fan-out on write vs read), follower graph, search, explore page |
| 93 | `93-hld-youtube.md` | HLD Case Study — YouTube / Video Streaming | Video upload and transcoding pipeline, chunked streaming, CDN, recommendation system overview, view count at scale |
| 94 | `94-hld-uber.md` | HLD Case Study — Uber / Ride Sharing | Geo-spatial indexing (QuadTree, S2 cells), driver location updates, matching algorithm, surge pricing, trip state machine |
| 95 | `95-hld-twitter.md` | HLD Case Study — Twitter / Social Feed | Tweet storage, follower graph, feed generation at scale (fan-out problem), trending topics, search, celebrities vs normal users |
| 96 | `96-hld-amazon.md` | HLD Case Study — Amazon / E-Commerce | Product catalog, inventory service, cart, order management, payment, search, recommendation, flash sale problem |
| 97 | `97-hld-distributed-cache.md` | HLD Case Study — Distributed Cache (Redis) | Cache cluster design, consistent hashing for cache nodes, replication, persistence in cache, eviction at scale, Redis Sentinel vs Cluster |
| 98 | `98-hld-search-autocomplete.md` | HLD Case Study — Search Autocomplete / Typeahead | Trie data structure, prefix matching, ranking suggestions, real-time vs precomputed, distributed trie, scaling for Google-scale |
| 99 | `99-hld-payment-system.md` | HLD Case Study — Payment System | Idempotency, distributed transactions, SAGA for payment flow, double-spend prevention, PCI compliance concepts, reconciliation |
| 100 | `100-hld-google-drive.md` | HLD Case Study — Google Drive / File Storage | Chunked upload, deduplication (block-level dedup), sync protocol, conflict resolution, sharing and permissions, metadata vs content separation |
| 101 | `101-hld-web-crawler.md` | HLD Case Study — Web Crawler | URL frontier, politeness policy, distributed crawling, duplicate detection (bloom filter), storage, robots.txt |
| 102 | `102-hld-ticketmaster.md` | HLD Case Study — Ticketmaster / Seat Booking | Flash sale traffic, seat locking (optimistic vs pessimistic locking), preventing double booking, queue-based access, payment integration |

---

## PHASE 6 — LLD CASE STUDIES
*Design real systems class by class, method by method. This is what LLD interviews test.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 103 | `103-lld-approach-method.md` | How to Approach Any LLD Problem — The Step-by-Step Method | Gather requirements → identify entities → define relationships → identify design patterns → write classes/interfaces → validate with scenarios |
| 104 | `104-lld-parking-lot.md` | LLD Case Study — Parking Lot System | Entities (lot, floor, spot, vehicle, ticket), spot types, entry/exit flow, pricing strategy pattern, full JavaScript class implementation |
| 105 | `105-lld-library-management.md` | LLD Case Study — Library Management System | Books, members, loans, reservations, fine calculation, search — complete class diagram and JS implementation |
| 106 | `106-lld-atm.md` | LLD Case Study — ATM Machine | State pattern for ATM states, card, account, transaction, cash dispenser, receipt printer — complete JS implementation |
| 107 | `107-lld-elevator.md` | LLD Case Study — Elevator System | Multiple elevators, request scheduling (SCAN algorithm), direction state, floor requests, dispatcher — complete JS implementation |
| 108 | `108-lld-vending-machine.md` | LLD Case Study — Vending Machine | State machine, inventory, coin handling, change return, product dispensing — complete JS implementation |
| 109 | `109-lld-hotel-booking.md` | LLD Case Study — Hotel Booking System | Room types, availability calendar, booking flow, pricing, cancellation, guest management — complete JS implementation |
| 110 | `110-lld-bookmyshow.md` | LLD Case Study — Movie Ticket Booking (BookMyShow) | Shows, theaters, seats, booking with locking, payment, concurrency handling — complete JS implementation |
| 111 | `111-lld-chess.md` | LLD Case Study — Chess Game | Pieces with polymorphism, board, move validation, check/checkmate detection, game state — complete JS implementation |
| 112 | `112-lld-snake-and-ladder.md` | LLD Case Study — Snake and Ladder Game | Board, players, dice, snakes, ladders, game loop, extensibility — complete JS implementation |
| 113 | `113-lld-splitwise.md` | LLD Case Study — Splitwise / Expense Sharing | Groups, expenses, split strategies (equal, percentage, exact, shares), balance calculation, settlement algorithm — complete JS implementation |
| 114 | `114-lld-logging-framework.md` | LLD Case Study — Logging Framework | Log levels, appenders (file, console, remote), formatters, async logging, log rotation, chain of responsibility pattern — complete JS implementation |
| 115 | `115-lld-rate-limiter.md` | LLD Case Study — Rate Limiter (Class Level) | Token bucket and sliding window at the class level, per-user and per-route limiting, in-memory vs Redis-backed — complete JS implementation |
| 116 | `116-lld-lru-cache.md` | LLD Case Study — LRU Cache | Doubly linked list + hash map design, O(1) get and put, eviction logic, capacity management, thread-safety concept — complete JS implementation |
| 117 | `117-lld-notification-service.md` | LLD Case Study — Notification Service | Channel abstraction (email, SMS, push), template engine, retry logic, observer pattern, notification preferences — complete JS implementation |
| 118 | `118-lld-food-delivery.md` | LLD Case Study — Food Delivery App (Zomato/DoorDash) | Restaurants, menus, orders, delivery agents, real-time tracking, pricing, state machine for order lifecycle — complete JS implementation |
| 119 | `119-lld-uber-matching.md` | LLD Case Study — Uber Driver-Rider Matching | Geo-based matching, driver state machine, ride request, pricing engine, surge logic, trip tracking — complete JS implementation |
| 120 | `120-lld-social-media-feed.md` | LLD Case Study — Social Media Feed | Posts, users, follows, feed generation algorithm, pagination, like/comment subsystems — complete JS implementation |
| 121 | `121-lld-api-gateway.md` | LLD Case Study — API Gateway | Plugin architecture, auth middleware, rate limiter integration, request routing, response transformation — complete JS implementation |
| 122 | `122-lld-inventory-management.md` | LLD Case Study — Inventory Management System | Products, warehouses, stock levels, reservations, restock triggers, supplier management — complete JS implementation |
| 123 | `123-lld-payment-gateway.md` | LLD Case Study — Payment Gateway | Payment methods, transaction lifecycle, idempotency keys, retry with backoff, refund flow, audit log — complete JS implementation |

---

## PHASE 7 — ADVANCED TOPICS & INTERVIEW MASTERY
*What separates a senior engineer from a principal engineer. The deep technical topics and the soft skills of system design communication.*

| # | File | Topic | What You'll Learn |
|---|------|--------|-------------------|
| 124 | `124-database-internals.md` | Database Internals — How Databases Actually Work | WAL (write-ahead log), B+ tree vs LSM tree storage engines, MVCC for concurrency, vacuum/compaction, why this matters for choosing a DB |
| 125 | `125-distributed-locking.md` | Distributed Locking | Why you can't use a local mutex in a distributed system, Redis SETNX and Redlock, ZooKeeper-based locking, fencing tokens, lease-based locks |
| 126 | `126-vector-clocks-and-crdt.md` | Vector Clocks and CRDTs | How distributed systems track causality, what a vector clock is, Conflict-free Replicated Data Types — how collaborative editing works |
| 127 | `127-bloom-filters.md` | Bloom Filters and Probabilistic Data Structures | What a bloom filter is, false positives but no false negatives, use cases (cache miss optimization, duplicate URL detection), HyperLogLog |
| 128 | `128-time-in-distributed-systems.md` | Time in Distributed Systems | Wall clock vs logical clock, why NTP is not enough, Lamport timestamps, Google Spanner's TrueTime, why ordering events is hard |
| 129 | `129-zero-downtime-deployments.md` | Zero Downtime Deployments | Blue-green deployment, canary releases, feature flags, rolling deployment, database migration during deployment, backward compatibility |
| 130 | `130-performance-optimization-patterns.md` | Performance Optimization Patterns | Lazy loading, eager loading, denormalization, materialized views, pre-computation, write amplification vs read amplification trade-offs |
| 131 | `131-multi-tenancy.md` | Multi-Tenancy Design | Silo vs pool vs bridge model, tenant isolation, noisy neighbor problem, per-tenant customization, billing and usage tracking |
| 132 | `132-gdpr-and-data-privacy.md` | GDPR and Data Privacy in System Design | Right to be forgotten, data residency requirements, PII handling, audit logging, anonymization vs pseudonymization, how to design for compliance |
| 133 | `133-cost-optimization.md` | Cost Optimization in System Design | Spot instances vs on-demand, tiered storage (hot/warm/cold), data compression, request batching, right-sizing, FinOps mindset |
| 134 | `134-system-design-interview-mastery.md` | System Design Interview Mastery | How to think out loud, how to handle ambiguity, what interviewers actually look for, common mistakes, how to structure 45 minutes, trade-off communication |
| 135 | `135-real-world-post-mortems.md` | Real-World Post-Mortems — What Failed and Why | Famous outages (Amazon S3, Cloudflare, Facebook), what went wrong, what was fixed, lessons every engineer should internalize |

---

## Summary

| Phase | Topics | Files |
|-------|--------|-------|
| Phase 1 — Foundations | 10 | 01–10 |
| Phase 2 — LLD Foundations | 10 | 11–20 |
| Phase 3 — LLD Design Patterns | 25 | 21–45 |
| Phase 4 — HLD Fundamentals & Components | 39 | 46–84 |
| Phase 5 — HLD Case Studies | 18 | 85–102 |
| Phase 6 — LLD Case Studies | 21 | 103–123 |
| Phase 7 — Advanced Topics & Interview Mastery | 12 | 124–135 |
| **TOTAL** | **135 topics** | **135 files** |

---

## How we will work together

1. I will generate **ONE complete doc at a time** — fully filled in, no shortcuts.
2. After every doc I will say exactly:
   `Doc saved: /docs/system-design/[XX]-[topic-name].md — Complete the practice exercise. Paste your answer when done.`
3. When you paste your answer I will review it: what you got right, what you missed, the ideal answer.
4. Then I will say: `Type NEXT when ready for the next topic.`

**Commands you can use at any time:**
- `START` → begin with topic 01
- `NEXT` → move to the next topic
- `HINT` → get a nudge on the current exercise without the answer
- `PROGRESS` → see the full curriculum with ✅ on completed topics
- `REDO [topic name]` → regenerate a topic completely

---

> **Does this plan look good?**
> It covers **135 topics** across 7 phases — from absolute zero to principal engineer level.
> Every topic becomes one file in `/docs/system-design/`.
>
> **Type `START` to begin with topic 01, or tell me if you want to add or change anything.**
