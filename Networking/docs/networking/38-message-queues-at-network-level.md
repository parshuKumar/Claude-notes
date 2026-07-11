# 38 — Message Queues at the Network Level

> **Phase 6 — Topic 4 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a busy restaurant kitchen.

- The **waiters** take orders from customers and shout them at the kitchen.
- If they shout directly at a specific cook and *wait* until that cook finishes the dish before taking the next order, the whole restaurant grinds to a halt the moment one cook is slow.
- Instead, the waiters clip each order onto a **spinning ticket rail** (the queue) and immediately go back to serving customers.
- The **cooks** pull tickets off the rail whenever they're free. If ten orders arrive at once, the rail holds them. Nobody is blocked. If a cook drops a ticket, it can be re-hung.

A **message queue** is that ticket rail. The waiters are your **producers**, the cooks are your **consumers**, and the rail is the **broker** (Kafka or RabbitMQ) sitting in the middle. The waiter doesn't need to know *which* cook will handle the order, or even that any cook is awake right now — it just hangs the ticket and moves on.

That "hang it and move on" is the whole point: it decouples the person creating work from the person doing it, **in both time and space**.

---

## Where This Lives in the Network Stack

A message broker is an **application-layer (Layer 7)** server that speaks its own binary protocol over a **long-lived TCP (Layer 4)** connection.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   AMQP 0-9-1 (RabbitMQ) / Kafka protocol  │  ← the broker's wire protocol
│  Layer 6 — Presentation  TLS (5671 AMQPS / 9093 Kafka+TLS)       │  ← optional encryption
│  Layer 5 — Session       long-lived connection, channels/streams │
│  Layer 4 — Transport     TCP  (5672 RabbitMQ / 9092 Kafka)       │  ← ONE persistent connection
│  Layer 3 — Network       IP   (broker IP, partition-leader IPs)  │
│  Layer 2 — Data Link     Ethernet inside the data center         │
│  Layer 1 — Physical      NIC / fiber                             │
└─────────────────────────────────────────────────────────────────┘
```

The key network fact: **you do not open a new TCP connection per message.** You open *one* persistent TCP connection to the broker at startup and keep it alive for the life of the process, pushing thousands of messages through it. RabbitMQ multiplexes many logical **channels** over that one connection (like HTTP/2 streams — see topic 11); Kafka multiplexes many in-flight requests over its one connection per broker.

---

## What Is This?

A **message queue** (more broadly, a **message broker** or **event streaming platform**) is a server that sits between the code that *produces* work and the code that *consumes* it, so the two never have to talk directly.

Instead of the synchronous request/response you learned in `15-rest-over-http.md`:

```
SYNCHRONOUS (REST):   Service A ──HTTP request──► Service B
                      Service A ◄──HTTP response── Service B
                      A is BLOCKED the whole time. If B is down, A fails.
```

...a broker gives you **asynchronous** messaging:

```
ASYNCHRONOUS (queue):
                    ┌──────────┐         ┌───────────────┐         ┌──────────┐
   produce("job")  │ Producer │────────►│    Broker     │────────►│ Consumer │
   returns instantly│  (App A) │  TCP    │ queue / topic │  TCP    │  (App B) │
                    └──────────┘         │ (buffered,    │         └──────────┘
                                         │  durable)     │         consumes when ready
                                         └───────────────┘
```

The producer's `publish()` call returns as soon as the broker acknowledges receipt. The consumer processes the message **later**, at its own pace, possibly on a different machine, possibly after a restart. That gap in time and location is called **decoupling**.

The two dominant open-source brokers, and the ones this topic dissects:

| | **RabbitMQ** | **Apache Kafka** |
|---|---|---|
| Model | Message queue (smart broker, dumb consumer) | Distributed commit log (dumb broker, smart consumer) |
| Protocol | AMQP 0-9-1 (open standard) | Custom binary Kafka protocol |
| Default port | 5672 (TLS: 5671) | 9092 (TLS: 9093) |
| Message after read | Deleted (acked and gone) | Retained on disk (replayable by offset) |
| Best at | Task/work queues, routing, RPC | High-throughput event streams, replay, log |

---

## Why Does It Matter for a Backend Developer?

A queue is not a "nice to have." It solves problems that synchronous HTTP physically cannot. This ties directly to `35-network-in-system-design.md` — the queue is one of the three or four building blocks you reach for in every design discussion.

**1. Buffering / backpressure absorption.** Traffic is spiky. A flash sale sends 50,000 orders in one second; your fraud-check service can process 500/second. Synchronously, 49,500 requests time out. With a queue, the broker holds all 50,000 and the consumer drains them over ~100 seconds. The spike is *smoothed* instead of *dropped*.

```
Incoming:  ▁▁▁█████████▁▁▁   (spike: 50k/s)
           │
           ▼
Broker:    [========== buffer ==========]   ← absorbs the burst
           │
           ▼
Consumer:  ▁▂▃▃▃▃▃▃▃▃▃▃▃▂▁   (steady drain: 500/s)
```

**2. Time decoupling.** The consumer can be down for maintenance. Messages pile up safely and are processed when it returns. With HTTP, a down consumer means immediate failures.

**3. Retries & durability.** If a consumer crashes mid-processing, the message wasn't acknowledged, so the broker redelivers it. No work is lost. You get *at-least-once* delivery for free.

**4. Fan-out.** One event ("user signed up") needs to trigger five things: send email, provision account, update analytics, warm cache, notify Slack. With HTTP that's five synchronous calls the signup handler must make and wait on. With pub/sub, you publish *once* and five independent consumers react.

**5. Spike smoothing across a slow downstream.** Your service is fast but calls a slow third-party API. The queue lets you accept work instantly and drip it to the third party at whatever rate it tolerates.

If you can't explain **consumer lag** and **why we don't open a connection per message**, you'll write systems that fall over under load. That's why this is a full topic.

---

## The Packet/Protocol Anatomy

### RabbitMQ — AMQP 0-9-1 frames over one TCP connection

AMQP is a **framed** protocol. After a one-time handshake (`AMQP\x00\x00\x09\x01` protocol header), everything is a **frame**. Every frame carries a **channel number**, which is how one TCP connection is multiplexed.

```
AMQP frame on the wire:
┌────────┬──────────┬─────────────┬───────────────────────┬────────────┐
│ type   │ channel  │ size        │ payload               │ frame-end  │
│ 1 byte │ 2 bytes  │ 4 bytes     │ (size bytes)          │ 0xCE       │
└────────┴──────────┴─────────────┴───────────────────────┴────────────┘
   │          │
   │          └── which CHANNEL this frame belongs to (0 = connection-global)
   └── 1=METHOD  2=HEADER  3=BODY  8=HEARTBEAT

Publishing one message = 3 frames on the same channel:
  METHOD frame  → basic.publish (exchange, routing key)
  HEADER frame  → properties (content-type, delivery-mode=2 for persistent)
  BODY frame    → the actual message bytes (split if > frame-max)
```

Channels let a single connection carry many independent conversations:

```
                        ONE TCP connection (port 5672)
   ┌──────────────────────────────────────────────────────────────┐
   │  Channel 1:  publisher thread  ──► basic.publish ...          │
   │  Channel 2:  consumer thread   ──► basic.consume / basic.ack  │
   │  Channel 3:  another consumer  ──► basic.consume / basic.ack  │
   └──────────────────────────────────────────────────────────────┘
        Like HTTP/2 streams (topic 11): multiplex, don't reconnect.
```

RabbitMQ's routing lives *inside* the broker — producers publish to an **exchange**, not a queue:

```
producer ──►[ EXCHANGE ]──binding(routing key)──►[ QUEUE ]──► consumer
             direct / fanout / topic / headers
```

### Kafka — binary request/response over persistent TCP

Kafka's protocol is a sequence of length-prefixed **request/response** pairs. Every request shares a header (API key = which request type, correlation ID to match responses, client ID).

```
Kafka request on the wire:
┌────────────┬──────────┬──────────────┬───────────────┬───────────┬─────────┐
│ length     │ api_key  │ api_version  │ correlation_id│ client_id │ payload │
│ 4 bytes    │ 2 bytes  │ 2 bytes      │ 4 bytes       │ string    │  ...    │
└────────────┴──────────┴──────────────┴───────────────┴───────────┴─────────┘
   api_key examples: 0=Produce  1=Fetch  3=Metadata  11=JoinGroup  14=SyncGroup
```

The unit that everything revolves around is the **partition** — an append-only, disk-backed log:

```
Topic "orders", 3 partitions, each an ordered append-only log with OFFSETS:

 Partition 0:  [0][1][2][3][4][5][6]───► new writes append here
 Partition 1:  [0][1][2][3][4]────────►
 Partition 2:  [0][1][2][3][4][5][6][7]►
                │                    │
              oldest              newest (log-end-offset)
              (retained until retention.ms / retention.bytes expires)
```

A message is never "removed when read." A consumer just remembers **which offset it has processed**. Read = advance a pointer, not delete data.

---

## How It Works — Step by Step

### Path A — RabbitMQ: publish and consume a work queue

```
Step 1 — CONNECT (once, at process startup)
   App opens TCP to broker:5672
   TCP 3-way handshake (SYN / SYN-ACK / ACK)  ← topic 08
   AMQP protocol header exchanged, then connection.start / .tune / .open
   Result: one ESTABLISHED, long-lived TCP connection.

Step 2 — OPEN A CHANNEL
   channel.open on channel 1. All work happens on channels, not the raw connection.

Step 3 — DECLARE TOPOLOGY (idempotent)
   queue.declare  name="emails" durable=true
   (exchange.declare / queue.bind if using a non-default exchange)

Step 4 — PRODUCER PUBLISHES
   basic.publish → exchange="" routing_key="emails"
   delivery-mode=2 (persistent → written to disk)
   If PUBLISHER CONFIRMS are on, broker replies basic.ack once it's safe.

Step 5 — CONSUMER SUBSCRIBES with QoS
   basic.qos prefetch_count=20   ← at most 20 UNACKED messages in flight
   basic.consume queue="emails"
   Broker pushes messages down the channel as they arrive.

Step 6 — CONSUMER ACKNOWLEDGES
   Process message → basic.ack (delivery_tag)  ← now broker deletes it
   On failure → basic.nack requeue=true        ← redeliver later
   If the TCP connection drops before ack → broker redelivers to someone else.
```

RabbitMQ is a **push** model: the broker sends messages to consumers, bounded by `prefetch`. Prefetch is your flow-control knob — it stops one greedy consumer from grabbing 10,000 messages while others sit idle.

### Path B — Kafka: the bootstrap → metadata → leader flow

This is the part most engineers get wrong. You **do not** connect to a single "the Kafka server." You bootstrap.

```
Step 1 — BOOTSTRAP CONNECT
   Client connects to ANY broker in bootstrap.servers (e.g. b1:9092).
   This is just an entry point, not necessarily where your data lives.

Step 2 — METADATA REQUEST (api_key 3)
   Client: "Tell me about topic 'orders'."
   Broker replies with the FULL cluster map:
     - every broker's host:port  (b1:9092, b2:9092, b3:9092)
     - for each partition: which broker is the LEADER
       orders-0 → leader b2      orders-1 → leader b3      orders-2 → leader b1

Step 3 — CONNECT DIRECTLY TO LEADERS
   The client now opens a SEPARATE persistent TCP connection to each broker
   that leads a partition it cares about. Produce/fetch go to the LEADER only.

     Client ──TCP──► b2:9092   (writes to orders-0)
     Client ──TCP──► b3:9092   (writes to orders-1)
     Client ──TCP──► b1:9092   (writes to orders-2)

Step 4 — PRODUCE (api_key 0)
   Producer batches records per partition (linger.ms, batch.size),
   optionally compresses the batch (gzip/snappy/lz4/zstd),
   sends to the partition leader. Leader appends to its log, replicates to
   followers, then acks based on acks=0/1/all.

Step 5 — CONSUME (api_key 1, FETCH) — PULL model
   Consumer sends Fetch(partition, offset). Broker returns records from that
   offset. Consumer PULLS; the broker never pushes. Consumer commits its
   processed offset (to the internal __consumer_offsets topic).
```

### Replication and `acks`

```
Producer writes orders-0 (leader = b2, replicas = b2,b3,b1, ISR = {b2,b3,b1})

 acks=0    fire-and-forget. Producer doesn't wait. Fastest, can lose data.
 acks=1    leader b2 wrote it to its log, then acks. Lost if b2 dies before
           followers copy it.
 acks=all  leader waits until all IN-SYNC REPLICAS (ISR) have it, then acks.
           Combined with min.insync.replicas=2, survives a broker failure.

           b2(leader) ──replicate──► b3 ✓
                       ──replicate──► b1 ✓   ← now all ISR have it → ack
```

**ISR** = In-Sync Replicas: the followers currently caught up with the leader. If a follower lags too far it's dropped from ISR; if the leader dies, a new leader is elected *only* from the ISR, which is why `acks=all` + `min.insync.replicas` gives durability.

### ZooKeeper vs KRaft (the cluster's brain)

Kafka needs somewhere to store cluster metadata (who's the controller, which broker leads which partition, topic configs).

- **ZooKeeper** (old): a separate 3–5 node ZooKeeper ensemble held that metadata. Extra system to run, and the controller-to-ZK path became a scaling bottleneck.
- **KRaft** (KIP-500, production-ready in Kafka 3.3, ZooKeeper removed in Kafka 4.0): Kafka stores its own metadata in an internal Raft-replicated log run by dedicated **controller** brokers. One system to operate, faster failover, millions of partitions. New clusters should use KRaft.

Either way, this is invisible to your client protocol — clients always just talk the Kafka wire protocol to brokers.

---

## Exact Syntax Breakdown

### Kafka: describe a consumer group (the single most useful command)

```
kafka-consumer-groups.sh  --bootstrap-server localhost:9092  --describe  --group order-service
│                          │                                  │           │
│                          │                                  │           └── which consumer group
│                          │                                  └── show per-partition offsets + LAG + assignment
│                          └── entry point broker(s); tool discovers the rest via Metadata
└── admin CLI shipped with Kafka
```

### RabbitMQ: list queues and connections

```
rabbitmqctl  list_queues  name  messages  consumers  messages_unacknowledged
│            │            │     │         │          │
│            │            │     │         │          └── delivered but not yet acked (in-flight)
│            │            │     │         └── how many consumers are attached
│            │            │     └── READY messages waiting (this IS the backlog)
│            │            └── column: queue name
│            └── subcommand
└── RabbitMQ admin CLI (run on a broker node)

rabbitmqctl list_connections name peer_host channels state
                                             │
                                             └── how many channels multiplexed on that ONE TCP conn
```

### ss: prove the connections are persistent (topic 24)

```
ss  -tnp  state established  '( dport = :9092 )'
│   ││││                        │
│   ││││                        └── filter: destination port 9092 (Kafka)
│   │││└── p = show owning process/PID
│   ││└── n = numeric (don't resolve DNS/port names)
│   │└── t = TCP only
│   └── socket statistics
└── one line per LIVE connection to a broker — expect a small, STABLE count
```

---

## Example 1 — Basic

Produce and consume from a Kafka topic, then read the lag. This runs against a single local broker.

```bash
# 1. Create a topic with 3 partitions (parallelism) and 1 replica (local dev)
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 3 --replication-factor 1

# 2. Produce three messages (type each, Enter, then Ctrl-D)
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
>order-1001
>order-1002
>order-1003

# 3. Consume from the beginning, as a NAMED group so offsets are tracked
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --from-beginning --group order-service
```

Output:
```
order-1001
order-1003
order-1002        ← NOTE: not in produce order! Different partitions = no global order.
```

That out-of-order output is the single most important lesson in Kafka: **ordering is guaranteed only within a partition**, never across the topic. `order-1001` might be partition 0 offset 0, `order-1002` partition 2 offset 0 — the consumer interleaves them.

Now check the group's position and lag:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group order-service
```

```
GROUP          TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID        HOST         CLIENT-ID
order-service  orders  0          1               1               0    consumer-1-a1b2..  /127.0.0.1   consumer-1
order-service  orders  1          1               1               0    consumer-1-a1b2..  /127.0.0.1   consumer-1
order-service  orders  2          1               1               0    consumer-1-a1b2..  /127.0.0.1   consumer-1
```

Read it like this:
- **LOG-END-OFFSET** = newest message written to that partition.
- **CURRENT-OFFSET** = last offset this group committed (processed).
- **LAG = LOG-END-OFFSET − CURRENT-OFFSET** = how many messages are waiting. `0` means fully caught up.
- One `CONSUMER-ID` owns all three partitions because there's only one consumer in the group.

---

## Example 2 — Production Scenario

**The situation:** It's Monday 09:00. Your `order-service` consumer group processes an `orders` topic (12 partitions, 4 consumer instances). Alerts fire: **customers see order confirmations 8 minutes late.** The queue is "backing up." You need to know if consumers are *too slow* or if a *rebalance storm* is thrashing them.

### Step 1 — Look at the lag

```bash
kafka-consumer-groups.sh --bootstrap-server kafka-1:9092 --describe --group order-service
```

```
GROUP          TOPIC   PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG      CONSUMER-ID   HOST          CLIENT-ID
order-service  orders  0          8421032         8443011         21979    -             -             -
order-service  orders  1          8402150         8443980         41830    -             -             -
order-service  orders  2          8410090         8444120         34030    -             -             -
...
order-service  orders  11         8399201         8442877         43676    -             -             -
```

Two red flags at once:
1. **LAG is large and roughly equal across all partitions** (~20k–43k each) → the group as a whole can't keep up.
2. **CONSUMER-ID / HOST are `-`** → right now *no consumer owns these partitions.* That only happens mid-rebalance. Catch it in that state repeatedly and you have a **rebalance storm**: the group spends its time reassigning partitions instead of processing.

### Step 2 — Confirm the rebalance storm in the logs

```bash
# On a consumer host, count rebalances in the last 10 minutes
grep -c "Revoke previously assigned partitions\|Attempt to heartbeat failed" \
  /var/log/order-service/consumer.log
```

```
147          ← 147 rebalances in 10 minutes. Healthy is ~0. This is a storm.
```

```
# The smoking gun in the log:
Member consumer-3 ... has left the group during rebalance
  reason: consumer poll timeout has expired.
  This means the time between subsequent calls to poll() was longer than
  the configured max.poll.interval.ms (300000).
```

**Root cause:** each `poll()` returns a batch of 500 records; processing each order calls a slow downstream payment API. Processing 500 records takes **longer than `max.poll.interval.ms`**, so the broker's group coordinator decides the consumer is dead, kicks it out, and triggers a rebalance. The kicked consumer finishes, rejoins → another rebalance. Repeat forever. During each rebalance every consumer pauses (eager rebalancing = stop-the-world), so lag grows even though the consumers aren't actually broken.

### Step 3 — Verify the connections are healthy (rule out a network flap)

```bash
ss -tnp state established '( dport = :9092 )'
```

```
Recv-Q  Send-Q   Local Address:Port     Peer Address:Port    Process
0       0        10.0.3.11:51830        10.0.1.5:9092        users:(("java",pid=8123,fd=88))
0       0        10.0.3.11:51844        10.0.1.6:9092        users:(("java",pid=8123,fd=91))
0       0        10.0.3.11:51902        10.0.1.7:9092        users:(("java",pid=8123,fd=94))
```

Stable, small connection count (one per broker) — the TCP layer is fine. This is an application-level rebalance problem, not a network flap.

### Step 4 — The fixes

```properties
# 1. Process fewer records per poll so a batch finishes well within the timeout
max.poll.records=50

# 2. Give processing more headroom (5 min → 10 min) if batches are genuinely slow
max.poll.interval.ms=600000

# 3. Heartbeats run on a background thread; keep session timeout sane
session.timeout.ms=45000
heartbeat.interval.ms=15000        # rule of thumb: ~1/3 of session.timeout.ms

# 4. Use cooperative (incremental) rebalancing so consumers DON'T all stop the world
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

And structurally:
- **Scale consumers to the partition count.** 12 partitions can feed at most **12 active consumers** in one group — a 13th sits idle. If 4 consumers can't keep up, add up to 8 more (and add partitions if you need to go beyond 12).
- **Make processing idempotent** (see topic 15) because these tuning changes keep at-least-once delivery — the same order *will* be redelivered after a rebalance. Dedupe on `order_id`.

### The RabbitMQ version of this same fire

The equivalent RabbitMQ outage looks different on the wire. You get paged that the broker is at 100% memory and refusing publishes:

```bash
rabbitmqctl list_connections name peer_host channels state | wc -l
```

```
9841        ← ~10,000 connections. For a service with 40 pods. That's the bug.
```

```bash
rabbitmqctl list_connections name channels
```

```
name                    channels
10.0.3.11:52001 -> ...   1        ← one connection, ONE channel — opened per message
10.0.3.11:52002 -> ...   1
10.0.3.11:52003 -> ...   1
...(thousands more)...
```

**Root cause:** the app opens a **new TCP connection for every message published**, then closes it. Each connection costs RabbitMQ ~100 KB+ of memory and a full TCP + AMQP handshake. Ten thousand of them exhausts the broker's memory watermark, which triggers **flow control** and blocks all publishers.

**Fix:** open **one long-lived connection per process** at startup, and open a **channel per thread** on it. Connections are expensive; channels are cheap.

```
BEFORE (wrong):  every publish() → connect → publish → close   (10,000 conns)
AFTER  (right):  startup → connect ONCE → per-thread channel → reuse forever  (40 conns)
```

---

## Common Mistakes

### Mistake 1: Opening a connection per message (or per thread)

```
WRONG (RabbitMQ):
   for each message:
       conn = amqp.connect("amqp://broker:5672")   # TCP + AMQP handshake EVERY time
       ch = conn.channel()
       ch.basic_publish(...)
       conn.close()
   → Thousands of TCP handshakes/sec, broker memory exhausted, flow control kicks in.

RIGHT:
   startup:  conn = amqp.connect(...)          # ONE connection, kept alive
   per thread/task:  ch = conn.channel()       # cheap logical stream (like HTTP/2 stream)
   hot path:  ch.basic_publish(...)            # no handshake, just frames
```

Same rule for Kafka: create **one producer/consumer client per process** and reuse it. Each Kafka client already maintains one persistent connection per broker internally. Creating a client per request is the Kafka version of this bug and will melt your broker with metadata churn.

### Mistake 2: Assuming global ordering across partitions

```
WRONG: "Kafka preserves order, so messages come out in the order I sent them."
REALITY: Order is guaranteed ONLY within a single partition.
         Topic with 12 partitions = 12 independent ordered logs, interleaved.

FIX: If two messages MUST stay ordered (e.g. all events for one account),
     give them the SAME partition KEY:
         producer.send("orders", key=account_id, value=event)
     Same key → same partition → same consumer → guaranteed order for that account.
```

RabbitMQ has the mirror trap: a single queue is FIFO, but the moment you attach **multiple consumers** (or use `basic.nack requeue=true`), delivery order is no longer the order you'd expect.

### Mistake 3: Not handling redelivery idempotently

```
WRONG: Consumer charges a credit card, then crashes BEFORE sending the ack.
       Broker sees no ack → redelivers → card charged TWICE.

At-least-once delivery means DUPLICATES WILL HAPPEN. This is not a bug in the
broker; it's the contract.

FIX (topic 15 idempotency): make the handler idempotent.
   INSERT INTO processed(msg_id) VALUES ($id) ON CONFLICT DO NOTHING;
   if row already existed → skip the side effect, just ack.
```

### Mistake 4: Ignoring consumer lag until it's a fire

```
WRONG: No alerting on lag. You find out consumers stalled when CUSTOMERS complain.
REALITY: Lag is your #1 health metric. Growing lag = consumers falling behind =
         a queue that will eventually overflow retention and LOSE data.

FIX: Alert on lag trend, not absolute value:
     - lag steadily RISING for N minutes  → consumers can't keep up (scale/tune)
     - lag flat & high                     → add consumers/partitions
     Export it: kafka-consumer-groups --describe, Burrow, or JMX records-lag-max.
```

### Mistake 5: Session/poll timeouts too aggressive → rebalance storm

```
WRONG: session.timeout.ms=6000 with slow per-message processing.
       One slow batch → missed heartbeat → coordinator evicts the consumer →
       rebalance → repeat → the group thrashes and NEVER makes progress.

FIX: raise max.poll.interval.ms to comfortably exceed worst-case batch time,
     shrink max.poll.records, keep heartbeat.interval ≈ session.timeout / 3,
     use the CooperativeStickyAssignor so rebalances aren't stop-the-world.
```

### Mistake 6: Treating the queue as a database

```
WRONG: "I'll just query the queue to find message X" / "store everything forever."
REALITY: A broker is a PIPE, not a store of record.
   - RabbitMQ: message is GONE the instant it's acked.
   - Kafka: retained only until retention.ms / retention.bytes — then deleted.
FIX: If you need durable query-able state, write it to a database (topic 39).
     Use the queue to MOVE work, the DB to STORE truth.
```

### Mistake 7: Using `acks=1` (or `0`) when you needed durability

```
WRONG: Payment events produced with acks=1. Leader acks, then the leader broker
       crashes before followers replicate → those payments are GONE.

FIX for data you cannot lose:
   producer: acks=all, enable.idempotence=true
   topic:    replication.factor=3, min.insync.replicas=2
   Now a single broker failure loses nothing.
   (Use acks=1 only for high-volume, loss-tolerant data like metrics/clicks.)
```

---

## Hands-On Proof

```bash
# ---------- KAFKA ----------
# 1. See your client's ACTUAL persistent connections to brokers (topic 24)
ss -tnp state established '( dport = :9092 )'
#    → one line per broker. Small, stable count. NOT one-per-message.

# 2. Watch lag update live as a consumer drains a topic
watch -n 2 "kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-service"
#    → watch the LAG column shrink toward 0 as the consumer catches up.

# 3. List every consumer group on the cluster
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 4. Inspect a topic's partition/replica/leader layout
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
#    → Leader:, Replicas:, Isr: per partition. This is what Metadata returns.

# 5. Force a rebalance and watch it: start a second consumer in the same group
#    in another terminal, then re-run --describe. Partitions get REASSIGNED.

# ---------- RABBITMQ ----------
# 6. See the backlog (READY = waiting, UNACKED = in-flight)
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged consumers

# 7. Prove channel multiplexing: few connections, many channels
rabbitmqctl list_connections name channels
rabbitmqctl list_channels connection number prefetch_count messages_unacknowledged

# 8. Watch the raw TCP: expect a HANDFUL of long-lived connections on 5672
ss -tnp state established '( dport = :5672 )'

# 9. The management UI (port 15672) shows the same data graphically
#    open http://localhost:15672  (guest/guest by default, local only)
```

---

## Practice Exercises

### Exercise 1 — Easy: Read the lag and connections

```bash
# Spin up a topic, produce 10 messages, consume 6 of them (Ctrl-C after 6),
# then run:
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group myg

# Answer from the output:
# 1. What is the LAG on each partition, and what is the total lag?
# 2. What does CURRENT-OFFSET vs LOG-END-OFFSET tell you?
# 3. Run `ss -tn state established '( dport = :9092 )'`. How many TCP
#    connections does your single consumer hold, and why that number?
```

### Exercise 2 — Medium: Prove per-partition ordering

```bash
# Create a topic with 4 partitions.
kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic events --partitions 4 --replication-factor 1

# Produce 12 messages WITHOUT a key (round-robins across partitions):
seq 1 12 | kafka-console-producer.sh --bootstrap-server localhost:9092 --topic events

# Now produce 12 messages WITH the SAME key so they all land on one partition:
seq 1 12 | kafka-console-producer.sh --bootstrap-server localhost:9092 --topic events \
  --property "parse.key=true" --property "key.separator=:" \
  # (feed lines like "acct-A:1", "acct-A:2", ... )

# Consume with --from-beginning and observe.
# Answer:
# 1. Were the keyless messages delivered in the order you sent them? Why not?
# 2. Were the same-key messages in order? Why?
# 3. In a real system, what would you use as the partition key for "all
#    events for one user must stay ordered"?
```

### Exercise 3 — Hard: Diagnose and fix a rebalance storm

```bash
# Given: a group with 6 partitions, 3 consumers, processing 200ms/message,
#        max.poll.records=500, max.poll.interval.ms=60000.

# 1. Do the math: how long does one poll() of 500 messages take to process?
#    Compare it to max.poll.interval.ms. What WILL the coordinator do, and why?

# 2. Reproduce: run `--describe` in a loop and catch the state where
#    CONSUMER-ID shows "-" (mid-rebalance). Count rebalances in the logs:
grep -c "Revoke previously assigned partitions" consumer.log

# 3. Propose concrete config changes (max.poll.records, max.poll.interval.ms,
#    heartbeat/session timeouts, assignment strategy) with numbers, and justify
#    each. Then explain why you ALSO need idempotent processing after this fix.

# 4. Bonus: your fix lets the group keep up at 6 consumers, but load doubles.
#    You have 6 partitions. What's the ceiling on consumers, and what do you
#    change to go higher?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A producer calls `publish()` and it returns in 2ms, but the consumer processes the message 40 seconds later on a different machine.** Name the two dimensions along which the broker just decoupled these two services.

2. **How many TCP connections should a RabbitMQ producer process hold to publish 5,000 messages/second across 10 threads, and how does it isolate those 10 threads on the wire?**

3. **A Kafka client is configured with `bootstrap.servers=b1:9092`, but you observe it holding live connections to b1, b2, AND b3.** Walk through the sequence of requests that made that happen.

4. **You produced messages `A, B, C` in order to a 3-partition topic and the consumer printed `A, C, B`.** Is this a bug? If you needed A, B, C to always arrive in order, what one change fixes it — and what does it cost you?

5. **Consumer lag on every partition is large AND the `--describe` output shows `-` for CONSUMER-ID/HOST.** What single condition explains both symptoms, and what's the usual root cause?

6. **What does `acks=all` actually wait for, and why is it useless for durability without also setting `min.insync.replicas`?**

7. **Your consumer charges a card, then crashes before acknowledging. What does the broker do next, what's the risk, and what property must the handler have to be safe?**

---

## Quick Reference Card

| Concept | RabbitMQ | Kafka |
|---|---|---|
| Model | Queue / point-to-point (each msg → 1 consumer) | Log / pub-sub (each msg replayable by many groups) |
| Protocol | AMQP 0-9-1 (framed, standard) | Custom binary request/response |
| Data port / TLS | 5672 / 5671 | 9092 / 9093 |
| Mgmt / admin | 15672 (HTTP UI) | JMX + `kafka-*.sh` CLIs |
| Multiplexing | Channels over one TCP connection | Requests over one TCP connection per broker |
| Delivery push/pull | Push (bounded by prefetch/QoS) | Pull (consumer Fetches by offset) |
| After consume | Deleted on ack | Retained until retention expires |
| Ordering unit | Per queue | Per partition |
| Parallelism unit | Consumers on a queue (prefetch) | Partitions (≤ N consumers per group) |
| Flow control | `basic.qos` prefetch_count | `max.poll.records`, backpressure via poll |
| Health metric | `messages_ready` (backlog) | Consumer **lag** (LEO − committed offset) |

**Delivery semantics:**
```
at-most-once   ack/commit BEFORE processing → may LOSE on crash, never duplicates
at-least-once  ack/commit AFTER processing  → never loses, may DUPLICATE (the default)
exactly-once   at-least-once + idempotent consumer, OR Kafka transactions
               (enable.idempotence + transactional producer)
```

**Durability recipe (Kafka, don't-lose-data):**
```
producer:  acks=all   enable.idempotence=true
topic:     replication.factor=3   min.insync.replicas=2
```

**Rebalance-safe consumer tuning:**
```
max.poll.records         = small enough that a batch processes < max.poll.interval.ms
max.poll.interval.ms     = comfortably > worst-case batch processing time
session.timeout.ms       = ~45000
heartbeat.interval.ms    = ~ session.timeout.ms / 3
partition.assignment.strategy = CooperativeStickyAssignor   (avoid stop-the-world)
consumers per group      ≤ partition count   (extra consumers sit idle)
```

**Key commands:**
```bash
kafka-consumer-groups.sh --bootstrap-server B:9092 --describe --group G   # LAG + assignment
kafka-topics.sh --bootstrap-server B:9092 --describe --topic T            # leaders/ISR
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
rabbitmqctl list_connections name channels state                          # reuse check
ss -tnp state established '( dport = :9092 )'                             # persistent conns
```

---

## When Would I Use This at Work?

### Scenario 1: "The signup endpoint is slow because it does too much"
Signup synchronously sends a welcome email, provisions storage, updates analytics, and pings Slack — 900ms total, and it fails if any downstream is down. Publish one `user.signed_up` event and let four independent consumers fan out from it. The endpoint returns in 20ms and survives any one consumer being down. This is the fan-out benefit from the "Why it matters" section, applied.

### Scenario 2: "We got a traffic spike and dropped orders"
A marketing push sent 20× normal order volume for 90 seconds; the payment service (rate-limited by the provider) couldn't keep up and requests timed out. Put the orders on a queue: the broker buffers the burst, and the consumer drains it at the provider's tolerated rate. Nothing is dropped — it's just processed a bit later. You'd watch lag climb during the spike and fall back to zero after, which is exactly healthy behavior.

### Scenario 3: "Confirmations are 8 minutes late" (the Example 2 fire)
You now diagnose it in the right order: check **lag** (`kafka-consumer-groups --describe`) → is it a *throughput* problem (lag high, consumers assigned) or a *rebalance storm* (CONSUMER-ID shows `-`, logs full of "Revoke previously assigned partitions")? Then fix with the right lever — scale consumers/partitions for throughput, or tune `max.poll.*` and switch to cooperative rebalancing for a storm — and make processing idempotent so the redeliveries don't double-charge anyone.

### Scenario 4: "The broker is out of memory / refusing connections"
Run `rabbitmqctl list_connections | wc -l`. Thousands of connections for a few dozen pods means the app opens a connection per message. Fix it to one long-lived connection per process with a channel per thread. This is the same class of resource-exhaustion bug as opening a raw DB connection per request (topic 39) — the answer is always *pool and reuse*, never *reconnect per unit of work*.

---

## Connected Topics

**Study before this:**
- `08-tcp-in-depth.md` — brokers ride on one long-lived TCP connection; handshakes, keep-alive, and teardown are the foundation for why reconnecting per message is so costly.
- `15-rest-over-http.md` — the synchronous request/response model that queues are the asynchronous alternative to; also the source of the idempotency rules your consumers must obey.
- `35-network-in-system-design.md` — where the queue fits as a core building block, and the latency/throughput trade-offs of async messaging.

**Study alongside / after:**
- `11-http2.md` — RabbitMQ channels are conceptually HTTP/2 streams: many logical conversations multiplexed over one connection.
- `24-netstat-and-ss.md` — the tooling to prove your connections are persistent and few, not per-message.
- `39-database-connections-over-network.md` — the same "pool and reuse persistent connections" lesson, applied to Postgres and PgBouncer; the DB is where durable truth lives, the queue only moves work to it.

---

*Doc saved: `/docs/networking/38-message-queues-at-network-level.md`*
