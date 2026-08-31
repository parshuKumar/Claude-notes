# 113 — Kafka I: Consumer Groups, Partitions, Rebalancing, Offsets

## Phase: 11 — Distributed Systems & Production
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow`'s `orderflow.order-placed` topic — 12 partitions, keyed by `orderId` — feeding two independent consumer groups: `inventory` (decrements stock, writes to Postgres) and `notifications` (calls an external email/SMS provider). The two groups have completely different processing-time profiles, which is what makes them the right pair to learn `max.poll.interval.ms` on.

---

## Mechanical statement

Read this twice. Everything else is an elaboration of it.

> **A partition is the unit of parallelism AND the unit of ordering. One partition
> maps to at most one consumer in a group.**
>
> Therefore: the maximum useful parallelism of a consumer group is the partition
> count. Consumer number 13 on a 12-partition topic is assigned nothing and does
> nothing, forever. It is not a warning; it is silence.
>
> And therefore: ordering is guaranteed **within a partition only**. Two events for
> the same `orderId` are ordered because they hash to the same partition. Two events
> for different orders have no ordering relationship at all, no matter what their
> timestamps say.
>
> **A consumer that does not call `poll()` again within `max.poll.interval.ms` is
> presumed dead.** The group coordinator removes it and triggers a **rebalance**,
> during which — with the classic eager protocol — **every consumer in the group
> revokes every partition and stops consuming.** Nobody processes anything until
> the assignment completes.
>
> That pause makes the *next* consumer more likely to exceed its own interval,
> because lag accumulated during the pause and each consumer now has a full batch
> waiting. One slow consumer becomes a rebalance, which makes several consumers
> slow, which becomes more rebalances. **This is the rebalance storm, and it is
> self-inflicted and self-sustaining.**

The corollary that decides your design:

> **`max.poll.interval.ms` is a budget for `max.poll.records` messages, not for one
> message.** The default 500 records at 700ms each is 350 seconds — past the
> 5-minute default. Most rebalance storms are this arithmetic, not a hung consumer.

---

## The bridge from what you know

Consumer groups, partitions, offsets and rebalancing are concepts you already have.
This section is only what is different in the Java client and in Spring Kafka.

### Difference 1 — the two timeouts, and why there are two

In most client libraries you have seen, "is the consumer alive" is one question. In
the Java client it is **two independent questions with two independent timeouts**,
and confusing them is the most common configuration error in Kafka.

| Setting | Default | Answers | Measured by |
|---|---|---|---|
| `session.timeout.ms` | 45s (older versions: 10s) | "Is the consumer **process** alive?" | A **background heartbeat thread** |
| `max.poll.interval.ms` | 300000 (5 min) | "Is the consumer **making progress**?" | The gap between successive `poll()` calls on the application thread |
| `heartbeat.interval.ms` | 3s | How often the background thread heartbeats | — |

Since KIP-62 (Kafka 0.10.1), heartbeats moved to a **background thread**. That
means a consumer stuck for four minutes inside your message handler is still
heartbeating happily — the coordinator thinks it is alive — right up until
`max.poll.interval.ms` expires and it is evicted for lack of progress.

The practical consequence: **raising `session.timeout.ms` does nothing for a slow
consumer.** People do this constantly. It is the wrong dial. The processing budget
is `max.poll.interval.ms`.

### Difference 2 — the poll loop is a state machine, not a message stream

The Node/`kafkajs` mental model is "an async iterator delivers messages". The Java
model is a loop you drive, and `poll()` does far more than fetch records:

```java
while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
    // poll() ALSO:
    //   - joins the group on first call, or rejoins after a rebalance
    //   - sends and receives coordinator requests
    //   - runs ConsumerRebalanceListener callbacks
    //   - performs auto-commits, if enabled
    //   - fetches from partition leaders
    for (var record : records) { process(record); }
}
```

**Everything the consumer does happens inside `poll()`.** Not calling it is not
"idling" — it is "not participating in the group". This is why a long
`process(record)` loop is a liveness failure rather than merely a slow consumer, and
it is the mechanical reason `max.poll.interval.ms` exists at all.

Spring Kafka hides the loop behind `@KafkaListener`, and the container calls
`poll()` for you — but only after your listener method **returns**. The budget is
still yours to blow.

### Difference 3 — Spring Kafka's `concurrency` is threads, not consumers-per-partition

```java
@KafkaListener(topics = "orderflow.order-placed", groupId = "inventory", concurrency = "4")
```

`concurrency = 4` creates **4 `KafkaConsumer` instances, each on its own thread**,
all in the group `inventory`, within one JVM. With 12 partitions and 2 pods, that is
8 consumers sharing 12 partitions.

Two things follow that surprise people:

- **`concurrency` above the partition count wastes threads.** Four consumers in one
  pod with only two partitions assigned to that pod means two idle threads.
- **Total group concurrency is `pods x concurrency`**, and *that* is the number that
  must not exceed 12. Scaling pods and raising `concurrency` at the same time is how
  teams end up with 24 consumers on 12 partitions and no idea why throughput did not
  change.

### Difference 4 — Spring Kafka's commit behaviour is not the Kafka default

The raw Java client defaults to `enable.auto.commit=true` with
`auto.commit.interval.ms=5000`. Spring Kafka **sets `enable.auto.commit=false`** and
commits offsets itself, according to an `AckMode` (default `BATCH`: commit after the
whole `poll()` batch has been processed by the listener).

This is a better default and it means the "auto-commit lost my messages" problem is
mostly a raw-client problem. But you must know which one you are in, because a
`@KafkaListener` and a hand-rolled loop in the same codebase have opposite
behaviour.

### Difference 5 — `orderflow`'s two consumer groups are the same topic and nothing else

`inventory` and `notifications` both subscribe to `orderflow.order-placed`. They are
independent: separate offsets, separate rebalances, separate lag. A rebalance storm
in `notifications` does not touch `inventory`.

They also have wildly different processing profiles, which is the whole point:

| Group | Per-message work | Typical duration | Failure mode |
|---|---|---|---|
| `inventory` | One Postgres transaction, one conditional UPDATE (Topic 52) | Milliseconds | Database contention |
| `notifications` | An HTTP call to an external email/SMS provider | Hundreds of ms, up to seconds when degraded | **Exactly the shape that blows `max.poll.interval.ms`** |

The same `max.poll.records` value is fine for one and lethal for the other. Consumer
configuration is per-group, not per-topic, and that is a design affordance you should
use.

---

## What is this?

### The group coordinator

For each consumer group, exactly one broker is the **group coordinator**. It is
chosen deterministically: hash `group.id`, mod the partition count of the internal
`__consumer_offsets` topic (50 partitions by default); the **leader of that
partition** is the coordinator for that group.

The coordinator:

- accepts `JoinGroup` and `SyncGroup` requests,
- tracks heartbeats and evicts members,
- stores committed offsets (as messages in `__consumer_offsets`),
- decides when a rebalance is needed.

Two consequences worth knowing: a broker failure moves the coordinator for every
group whose `__consumer_offsets` partition it led, causing rebalances that have
nothing to do with your consumers; and offsets are themselves a compacted Kafka
topic, so offset commits are ordinary Kafka writes with ordinary Kafka durability
semantics.

### Assignment: who decides which consumer gets which partition

Kafka's classic protocol puts the assignment logic **in the clients**, not the
broker:

1. Every member sends `JoinGroup` to the coordinator.
2. The coordinator picks one member as the **group leader** and sends it the full
   member list and their subscriptions.
3. **The leader runs the assignor** — plain Java code, in a client process — and
   computes the assignment.
4. The leader sends the assignment to the coordinator in `SyncGroup`.
5. The coordinator distributes each member's assignment back to it.

The broker never decides who gets what. That is why the assignor is a client
configuration (`partition.assignment.strategy`) and why all members must agree on
it.

### The assignors

| Assignor | Protocol | Behaviour |
|---|---|---|
| `RangeAssignor` (historical default) | Eager | Per topic, splits partitions into contiguous ranges. **Skews badly** with several topics: consumer 0 gets partition 0 of every topic. |
| `RoundRobinAssignor` | Eager | Spreads across all subscribed partitions. Balanced, but reshuffles everything on any change. |
| `StickyAssignor` | Eager | Balanced **and** minimises movement between rebalances — but still revokes everything first. |
| `CooperativeStickyAssignor` | **Cooperative (incremental)** | Sticky, and only the partitions that actually move are revoked. **This is the one you want.** |

**Eager vs cooperative is the difference between a stop-the-world pause and a
partial one**, and it is a one-line configuration change:

- **Eager rebalance:** every member revokes **all** its partitions, then everyone
  rejoins, then a new assignment is computed and distributed. Between revoke and
  assign, **the entire group consumes nothing**. With 12 partitions and a
  multi-second assignment, that is a 12-partition consumption gap for one consumer
  joining.
- **Cooperative rebalance:** two rounds. Round one, members revoke only the
  partitions the assignor decided to move; everything else keeps consuming
  throughout. Round two assigns the freed partitions. **Consumers that keep their
  partitions never stop.**

```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

**Migration warning, because this one bites:** you cannot flip every consumer from
eager to cooperative in one deploy — the members would disagree about the protocol.
The documented path is a **two-step rolling upgrade**: first deploy with the list
`[CooperativeStickyAssignor, <your current assignor>]` so members negotiate the old
protocol while the fleet is mixed, then deploy again with only
`CooperativeStickyAssignor`. Check the Kafka upgrade notes for your version before
doing it; the ordering matters and getting it wrong wedges the group.

### Static membership

`group.instance.id` (KIP-345) gives a consumer a stable identity. When a consumer
with a static id disconnects, the coordinator **does not immediately rebalance** —
it waits up to `session.timeout.ms` for the same id to come back and hands it the
same partitions.

This is exactly what a rolling restart needs. Without it, restarting 4 pods causes
up to 8 rebalances (one per departure, one per arrival). With it, and with
`session.timeout.ms` comfortably longer than a pod restart, it causes **zero**.

In Kubernetes, derive the id from the StatefulSet ordinal or the pod name — but see
Topic 114's warning: it must be stable **across restarts of the same logical
instance**, so a random UUID or a Deployment's random pod suffix defeats it.

### Offsets

An offset is a per-partition position. **Committing an offset means "the next record
I want is at offset N"** — it is the position to resume from, not the last record
processed. Off-by-one errors here re-deliver or skip exactly one record per
partition and are miserable to debug.

| Mechanism | Where | Semantics |
|---|---|---|
| `enable.auto.commit=true` | Raw client default | Commits the *last polled* offsets periodically, **regardless of whether you processed them**. At-most-once by accident. |
| Spring `AckMode.BATCH` | Spring default | Commit after the whole batch returns from the listener |
| Spring `AckMode.RECORD` | | Commit after each record. Safer, more commit traffic. |
| `AckMode.MANUAL` / `MANUAL_IMMEDIATE` | You call `Acknowledgment.acknowledge()` | Full control. `MANUAL_IMMEDIATE` commits at once; `MANUAL` queues until the batch ends. |
| `auto.offset.reset` | `latest` (default) / `earliest` / `none` | What to do when there is **no committed offset**, or the committed offset no longer exists |

`auto.offset.reset` deserves its own sentence because it is the source of the worst
Kafka incident shape: a new consumer group with `latest` **silently skips
everything already in the topic**, and a group whose retention expired past its
committed offset with `earliest` **silently reprocesses from the beginning**. Both
are silent. `none` throws instead, which for a critical consumer is often the honest
choice.

---

## Why does it matter?

**1. A rebalance storm is a total consumer outage that looks like a slow consumer.**
Lag climbs, throughput drops to near zero, and the consumers all look "up" — they
are running, logging, heartbeating. Nothing is crashing. Without knowing the
mechanism, teams respond by adding consumers, which either does nothing (beyond the
partition count) or makes it worse (more members, more rebalances).

**2. "Add consumers to increase throughput" is wrong beyond the partition count, and
everyone tries it first.** It is the single most predictable wrong first move, and
being able to say why in one sentence — with the `kafka-consumer-groups.sh`
output that proves it — is a visible seniority marker.

**3. `orderflow`'s inventory consumer decrements stock.** Duplicate or reordered
processing is oversell (Topic 52) or lost stock. The ordering guarantee you rely on
is *per-partition*, which means it depends on the key and on the partition count
never changing. Changing the partition count of a keyed topic breaks ordering for
in-flight keys, permanently and silently.

**4. It interacts with everything else in Phase 11.** The outbox relay (Topic 115)
produces to this topic. Delivery is at-least-once (Topic 114), so the consumer must
be idempotent (Topic 116). Graceful shutdown must drain consumers or a deploy causes
duplicates (Topic 123). A rebalance while a consumer holds a database transaction is
a Topic 109 problem.

---

## Machine-level reality

### The poll loop, in detail

`KafkaConsumer` is **not thread-safe** and enforces it: a `ConcurrentModificationException`
is thrown if two threads enter it. One consumer, one thread. That constraint is why
`concurrency` in Spring Kafka creates *n* consumers rather than *n* threads sharing
one.

Inside one `poll(timeout)`:

1. **Coordinator maintenance.** Ensure the coordinator is known; join the group if
   needed; process any pending rebalance, which is where
   `ConsumerRebalanceListener.onPartitionsRevoked` / `onPartitionsAssigned` run —
   **on your application thread, inside `poll()`**.
2. **Auto-commit**, if enabled and the interval elapsed.
3. **Return already-fetched records** from the internal buffer if any are ready,
   after which prefetch requests for the next batch are sent asynchronously.
4. Otherwise send fetch requests to the partition leaders and wait up to the
   timeout.

The consumer **prefetches**: while you process batch N, records for batch N+1 are
already arriving into an in-memory buffer sized by `fetch.max.bytes` and
`max.partition.fetch.bytes`. This is why a consumer's memory footprint is not
`max.poll.records x record size` — it is the fetch buffer, which can be much larger.
A consumer OOM under lag is usually this (Topic 79).

**`max.poll.records` does not change what is fetched.** It changes how many buffered
records `poll()` hands you at a time. Lowering it from 500 to 50 shortens your
processing budget per poll without reducing network traffic — which is exactly what
you want when fixing a `max.poll.interval.ms` problem.

### The heartbeat thread

A separate `HeartbeatThread` inside the consumer sends heartbeats every
`heartbeat.interval.ms`. It runs regardless of what your application thread is doing.

But it is not fully independent: when `max.poll.interval.ms` is exceeded, the
consumer **proactively sends a `LeaveGroup`** on the next poll — it removes itself
rather than waiting to be evicted. This is why the log line you see is about the
consumer leaving the group rather than being kicked, and why it is phrased as
advice about `max.poll.interval.ms` and `max.poll.records`.

*Illustration of the log shape you are looking for, not captured output:*

```
WARN  o.a.k.c.c.i.ConsumerCoordinator - [Consumer clientId=<id>, groupId=notifications]
      consumer poll timeout has expired. This means the time between subsequent calls to poll()
      was longer than the configured max.poll.interval.ms, which typically implies that the poll
      loop is spending too much time processing messages. You can address this either by
      increasing max.poll.interval.ms or by reducing the maximum size of batches returned in
      poll() with max.poll.records.
INFO  o.a.k.c.c.i.ConsumerCoordinator - ... Member <member-id> sending LeaveGroup request ...
INFO  o.a.k.c.c.i.ConsumerCoordinator - ... (Re-)joining group
```

**Those three lines in sequence are the diagnosis.** If you see them repeating, you
have the storm.

### Why the storm sustains itself

Walk the mechanism step by step, because "storm" as a word explains nothing:

1. Consumer A processes a 500-record batch. At 700ms each that is 350s > 300s.
2. At 300s, A's poll timeout expires. A sends `LeaveGroup`. Coordinator triggers a
   rebalance.
3. **Eager protocol:** B, C and D all revoke their partitions and rejoin. The whole
   group stops consuming for the duration of the rebalance.
4. During the pause, producers keep producing. **Lag grows on every partition.**
5. Assignment completes. B, C, D and (rejoined) A resume — each now facing a *full*
   `max.poll.records` batch, because there is more backlog than before.
6. A full batch is the worst case for the processing budget. Now B is at risk of
   exceeding its interval too.
7. B times out. Go to step 2.

Two amplifiers make it worse in practice: A is likely to time out again immediately
(nothing about its per-message cost changed), and after a rebalance every consumer
starts from a committed offset that may be behind where it had processed to, so some
work is repeated — increasing the per-batch cost.

The exit is not automatic. Left alone, this state persists until the backlog is
drained, which it cannot be, because nobody is draining it.

### Cooperative rebalancing changes step 3 only — and that is enough

With `CooperativeStickyAssignor`, when A leaves, B/C/D keep their partitions and
keep consuming. Only A's partitions move. Steps 4, 5 and 6 lose their force: there is
no group-wide pause, so lag does not spike for everyone, so nobody else gets a full
batch, so the cascade does not start.

The storm becomes a single slow consumer, which is a much better problem.

### Where the state lives

| State | Where | Lifetime |
|---|---|---|
| Group membership, generation id | Coordinator broker, in memory + `__consumer_offsets` | Until members leave or time out |
| Committed offsets | `__consumer_offsets`, a compacted topic | `offsets.retention.minutes` after the group becomes empty |
| The assignment | Computed by the **group leader client**, distributed by the coordinator | One generation |
| Fetched-but-unprocessed records | Consumer JVM heap | Until polled and processed |
| The consumer's position | Consumer JVM memory (`position`), separate from the committed offset | Reset on rebalance to the committed offset |

The last row is the one to internalise: **`position` (where you have read to) and
`committed` (where you told the coordinator you had read to) are different numbers.**
Everything between them is what gets reprocessed after a rebalance. That gap is your
duplicate window, and it is why Topic 116 exists.

### KIP-848 — a version-honesty note

Kafka has been moving to a **new consumer group protocol** (KIP-848) in which
assignment is computed **on the broker** rather than by a client leader, which
removes several of the failure modes above. It reached general availability in the
Kafka 4.x line.

> **I am not going to state whether it is the default on the broker and client
> versions you are running, or the exact `group.protocol` values.** Check your
> broker version's documentation. Everything in this document is written for the
> classic protocol, which remains supported and is what you will meet in existing
> systems; the `max.poll.interval.ms` budget arithmetic is unchanged under either
> protocol, because it is about your processing time, not about who computes the
> assignment.

---

## Example 1 — minimal

A single consumer you can starve on purpose. Use the raw client first, because the
loop is the thing you need to see; Spring Kafka hides it.

```xml
<dependency>
  <groupId>org.apache.kafka</groupId>
  <artifactId>kafka-clients</artifactId>
  <!-- version from the Boot BOM -->
</dependency>
```

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orderflow.lab.order-placed --partitions 3 --replication-factor 1
```

```java
package com.orderflow.lab;

public class LabConsumer {

    public static void main(String[] args) {
        long processingMillis = Long.parseLong(args[0]);   // the knob

        var props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "lab-inventory");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // The two settings this whole topic is about. Deliberately small so the
        // failure happens in seconds rather than minutes.
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "50");
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, "10000");   // 10s

        props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
                  CooperativeStickyAssignor.class.getName());

        try (var consumer = new KafkaConsumer<String, String>(props)) {

            consumer.subscribe(List.of("orderflow.lab.order-placed"),
                new ConsumerRebalanceListener() {
                    @Override public void onPartitionsRevoked(Collection<TopicPartition> ps) {
                        // Runs INSIDE poll(), on this thread. Commit here.
                        System.out.printf("[%s] REVOKED %s%n", Instant.now(), ps);
                        consumer.commitSync();
                    }
                    @Override public void onPartitionsAssigned(Collection<TopicPartition> ps) {
                        System.out.printf("[%s] ASSIGNED %s%n", Instant.now(), ps);
                    }
                    @Override public void onPartitionsLost(Collection<TopicPartition> ps) {
                        // Called instead of Revoked when the partitions are ALREADY gone
                        // (we were evicted). Do NOT commit here -- someone else owns them.
                        System.out.printf("[%s] LOST %s%n", Instant.now(), ps);
                    }
                });

            while (true) {
                var records = consumer.poll(Duration.ofMillis(500));
                if (records.isEmpty()) continue;

                System.out.printf("[%s] polled %d records%n", Instant.now(), records.count());
                for (var record : records) {
                    Thread.sleep(processingMillis);          // the slowness knob
                }
                consumer.commitSync();
                System.out.printf("[%s] committed%n", Instant.now());
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Three details in that listener are load-bearing and are usually written wrong:

- **`onPartitionsRevoked` is where you commit.** It runs inside `poll()` before the
  partitions are taken away, so it is your last chance to record progress.
- **`onPartitionsLost` exists and is different.** It fires when the partitions were
  already reassigned (you were evicted, or the cooperative protocol lost them).
  Committing there writes offsets for partitions someone else now owns.
- The listener runs **on the poll thread**. Slow work in it is charged to your
  `max.poll.interval.ms` budget too.

### Produce and observe

```bash
# 300 records, keyed, so they distribute across the 3 partitions.
for i in $(seq 1 300); do echo "ORD-$i:{\"orderId\":\"ORD-$i\"}"; done \
  | kafka-console-producer.sh --bootstrap-server localhost:9092 \
      --topic orderflow.lab.order-placed --property "parse.key=true" --property "key.separator=:"
```

```bash
# Fast: 50 records x 10ms = 0.5s per poll. Well inside a 10s budget.
java -cp ... com.orderflow.lab.LabConsumer 10

# Slow: 50 records x 300ms = 15s per poll. OUTSIDE a 10s budget.
java -cp ... com.orderflow.lab.LabConsumer 300
```

**WHAT TO LOOK FOR** — the timestamps in your own output, and the WARN line about
the poll timeout.

| What you see | What it means |
|---|---|
| `polled` then `committed` repeatedly with a gap under 10s | Healthy. Processing fits the budget. |
| A WARN about `max.poll.interval.ms` having expired, then `(Re-)joining group` | **The eviction.** Your processing exceeded the budget. |
| `LOST` rather than `REVOKED` after that WARN | You were evicted; the partitions were reassigned before you noticed. Confirms eviction rather than a clean shutdown. |
| The same 50 records polled again after rejoining | The commit never happened, so the offsets rolled back. **This is your duplicate window, demonstrated.** |
| The cycle repeating forever with no progress | The single-consumer version of the storm. |

### The one-line fix, and what it costs

```java
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "10");   // 10 x 300ms = 3s < 10s
```

Now it fits. Note that you did **not** make anything faster; you made the unit of
work smaller so it fits the budget. Throughput is unchanged. That distinction is the
core of Trap 2's fix.

---

## Example 2 — production scenario (on the project spine)

### The topology and the constraints

```
orderflow (8 pods)
   |
   +-- outbox relay (Topic 115) --> orderflow.order-placed
                                    12 partitions, key = orderId,
                                    replication.factor = 3, min.insync.replicas = 2
                                            |
                    +-----------------------+------------------------+
                    |                                                |
          group: inventory                                 group: notifications
          2 pods x concurrency 3 = 6 consumers             2 pods x concurrency 2 = 4 consumers
          work: 1 Postgres tx, ~5ms                        work: 1 external HTTP call, 200ms-3s
```

Constraints from the rest of the curriculum:

- Topic 65 baseline: 70/20/10 mix; order placement is 10% of arrival rate, so the
  produce rate to `orderflow.order-placed` equals the order-placement rate.
- Topic 109: `maximum-pool-size: 16` per pod. The inventory consumer's 3 threads each
  take a connection while processing.
- Topic 115: the relay is at-least-once. Duplicates are expected, not exceptional.
- Topic 111: the notification provider is external and can be slow.

### Group `inventory` — fast, database-bound

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP}
    consumer:
      group-id: inventory
      auto-offset-reset: earliest        # a new inventory consumer MUST NOT skip history
      enable-auto-commit: false
      max-poll-records: 100
      properties:
        max.poll.interval.ms: 60000      # 100 x 5ms = 0.5s. 60s is 100x headroom.
        session.timeout.ms: 45000
        heartbeat.interval.ms: 3000
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        group.instance.id: ${POD_NAME}   # static membership; see the note below
    listener:
      ack-mode: BATCH
      concurrency: 3
      observation-enabled: true          # Micrometer/OTel instrumentation (Topics 118-119)
```

```java
@Component
class InventoryConsumer {

    private final InventoryService inventory;

    @KafkaListener(topics = "orderflow.order-placed", groupId = "inventory")
    public void onOrderPlaced(@Payload OrderPlacedEvent event,
                              @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                              @Header(KafkaHeaders.OFFSET) long offset) {
        // Idempotent by construction -- Topic 116. The relay is at-least-once.
        inventory.applyReservation(event.orderId(), event.lines());
    }
}
```

**`concurrency: 3` on 2 pods is 6 consumers for 12 partitions**, so each consumer
owns 2 partitions. Deliberately below 12: it leaves headroom to scale to 4 pods
without exceeding the partition count, and it keeps each pod's Kafka work to 3 of
its 16 database connections.

**`auto-offset-reset: earliest` is a decision, not a default.** If the `inventory`
group's offsets are ever lost — a wiped `__consumer_offsets`, or retention expiring
while the group is down — `latest` would silently skip every unprocessed order.
Reprocessing is survivable because the consumer is idempotent; skipping is not.

**On `group.instance.id` and Kubernetes:** static membership requires an identity
that is stable across restarts of the *same logical instance*. A Deployment gives
pods random names, so `${POD_NAME}` from a Deployment changes on every restart and
buys you nothing. Either run consumers as a **StatefulSet** (stable ordinals) or
derive the id from something stable. If you cannot, omit it — a wrong static id is
worse than none, because a rejoining "new" instance holds the old one's partitions
hostage until `session.timeout.ms`.

### Group `notifications` — slow, externally bound

Same topic, entirely different configuration, because the per-message cost is three
orders of magnitude larger:

```yaml
spring:
  kafka:
    consumer:
      group-id: notifications
      auto-offset-reset: latest          # a missed notification is not worth a backfill storm
      enable-auto-commit: false
      max-poll-records: 10               # 10 x 3s worst case = 30s
      properties:
        max.poll.interval.ms: 120000     # 4x the worst-case batch. Deliberate headroom.
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
    listener:
      ack-mode: RECORD                   # commit per record: a slow batch loses less on eviction
      concurrency: 2
```

```java
@Component
class NotificationConsumer {

    private final GuardedNotificationGateway gateway;   // Topic 111: breaker + bulkhead

    @KafkaListener(topics = "orderflow.order-placed", groupId = "notifications")
    public void onOrderPlaced(OrderPlacedEvent event) {
        gateway.sendOrderConfirmation(event.orderId(), event.customerEmail());
    }
}
```

The budget arithmetic, written out because this is the calculation to internalise:

```
worst-case batch time = max.poll.records x worst-case per-message time
                      = 10 x 3s
                      = 30s

max.poll.interval.ms must exceed that, with headroom for GC pauses (Topic 71),
safepoint pauses (Topic 73), and a slow rebalance callback.

120000 ms / 30000 ms = 4x headroom.  Defensible.
```

Do that arithmetic for every consumer you write, with the **worst case**, not the
p50. The p50 is 200ms and would suggest `max.poll.records: 500` is fine. It is not,
because the provider's bad minutes are exactly when the queue is deepest.

**Note what the circuit breaker (Topic 111) does for the poll budget.** When the
notification provider is down and the breaker is open, calls fail in microseconds.
So the *worst* case for `max.poll.interval.ms` is not "provider down" — it is
"provider slow", the state where the breaker has not opened yet. That is the
interaction to reason about: resilience patterns change your poll-budget arithmetic,
usually favourably, but only for the states they actually cover.

### Error handling — because a poison message must not become an infinite loop

```java
@Bean
DefaultErrorHandler kafkaErrorHandler(KafkaTemplate<String, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    // 3 attempts, 1s apart. NOT an unbounded retry: every retry is charged to the
    // max.poll.interval.ms budget of the current poll.
    var handler = new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 2L));

    // Never retry a deserialization failure -- the bytes will not improve.
    handler.addNotRetryableExceptions(
        org.springframework.kafka.support.serializer.DeserializationException.class,
        org.springframework.messaging.converter.MessageConversionException.class);
    return handler;
}
```

**The container's in-listener retries consume your poll budget.** Three attempts at
one second apart adds up to two extra seconds per failing record, and if a whole
batch is failing, that multiplies by `max.poll.records`. A `FixedBackOff` of 10
attempts at 5 seconds on a 100-record batch is 5000 seconds of retrying inside one
poll — a guaranteed eviction. Either keep container retries tiny and let the DLT
handle the rest, or use the non-blocking retry-topic mechanism
(`@RetryableTopic`), which republishes to delay topics instead of blocking the poll
thread.

### Topic-level decisions that constrain everything above

```bash
kafka-topics.sh --bootstrap-server $KAFKA --create \
  --topic orderflow.order-placed \
  --partitions 12 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000
```

**Why 12.** It is the ceiling on consumer parallelism for every group, forever
(short of the disruptive change in Trap 5). Choose it from the *most parallel
consumer you will ever need*, not from today's. 12 divides evenly by 1, 2, 3, 4, 6
and 12, which makes assignments balanced at many consumer counts — a real
consideration, because 10 partitions across 3 consumers is a 4/3/3 split and one
consumer does 33% more work.

**Why keyed by `orderId`.** Ordering is per-partition, so all events for one order
land on one partition and are processed in order. That is the only ordering
guarantee `orderflow` has, and every consumer's correctness depends on it.

### `[BOOT 3.x DELTA]`

| Concern | Boot 3.x | Boot 4.1 | Note |
|---|---|---|---|
| Client version | From the Boot BOM | From the Boot BOM | Never pin `kafka-clients` yourself |
| Observation | `observation-enabled` on the listener container | Same, plus the OpenTelemetry starter | Topics 118–119 |
| Error handler | `DefaultErrorHandler` (replaced `SeekToCurrentErrorHandler` in 2.8) | Same | `SeekToCurrentErrorHandler` in a snippet is a date stamp |
| Serialization | Jackson 2 by default | **Jackson 3 standard** | A custom `JsonDeserializer` with Jackson 2 types needs updating |
| Test support | `@EmbeddedKafka` or Testcontainers | Same; Testcontainers preferred | Topic 61 |
| Consumer protocol | Classic | Classic, unless your broker and client enable KIP-848 | **Verify — do not assume** |

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — adding consumers beyond the partition count

**Wrong approach**

Lag is growing on `orderflow.order-placed`. The team scales the inventory consumer
deployment from 2 pods to 6, keeping `concurrency: 3`. That is 18 consumers on 12
partitions.

**Exact symptom**

Throughput does not change. Lag keeps growing at the same rate. CPU on the new pods
is near zero. Nothing errors. Nothing logs a warning.

The command that shows it in one screen:

```bash
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group inventory
```

*Illustration of the output columns, not captured output — the values are yours:*

```
GROUP      TOPIC                    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG    CONSUMER-ID           HOST          CLIENT-ID
inventory  orderflow.order-placed   0          <n>             <n>             <n>    consumer-inventory-1  /10.x.x.x     <id>
inventory  orderflow.order-placed   1          <n>             <n>             <n>    consumer-inventory-2  /10.x.x.x     <id>
...
```

**WHAT TO LOOK FOR:** count the **distinct `CONSUMER-ID` values**. It cannot exceed
the partition count.

| What you see | What it means |
|---|---|
| 12 distinct `CONSUMER-ID`s across 12 partitions | Every partition has an owner. **You are at maximum parallelism.** Adding consumers cannot help. |
| Fewer distinct `CONSUMER-ID`s than partitions | Some consumers own several partitions. There is room to scale out. |
| `CONSUMER-ID` shows `-` for some partitions | Nobody owns them — a rebalance is in progress, or the group has fewer members than partitions and has not settled. |
| Lag concentrated on a few partitions while others are at zero | **Key skew**, not a consumer-count problem. A few `orderId` values dominate, or your key is not what you think. |

Confirm the idle members directly:

```bash
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group inventory --members --verbose
```

Members with an empty partition list are doing nothing. That list is the evidence.

**Root cause**

The assignment invariant: **at most one consumer per partition per group.** With 12
partitions and 18 members, 6 members get nothing. They still join, still heartbeat,
still participate in every rebalance — so they add rebalance cost while adding zero
throughput.

**Fix**

First **diagnose why lag is growing**, because "not enough consumers" is one of four
possible causes and the least likely once you are at the partition count:

| Cause | How to tell | Fix |
|---|---|---|
| Genuinely under-parallel | Fewer consumers than partitions | Scale out — up to the partition count |
| At the partition ceiling | 12 distinct consumer ids | Make each message cheaper, or add partitions (Trap 5 — carefully) |
| Per-message processing too slow | Container `spring.kafka.listener` timer p99 | Fix the processing: batch the database writes, remove the N+1 (Topic 50), cache the lookup |
| Rebalance storm | High rebalance rate, lag sawtoothing | Trap 2 |
| Key skew | Lag on a few partitions only | Change the key, or accept the hot partition |

If you are at the ceiling and processing is already efficient, the honest answer is
**more partitions**, and that is a topic-level change with the ordering consequences
in Trap 5. There is no way to exceed the partition count with consumers.

Guardrail: assert it in code so nobody has to remember.

```java
@Component
class ConsumerConcurrencySanityCheck implements ApplicationRunner {

    @Value("${spring.kafka.listener.concurrency}") int concurrency;
    @Value("${orderflow.kafka.order-placed.partitions}") int partitions;
    @Value("${orderflow.expected-replica-count}") int replicas;

    @Override public void run(ApplicationArguments args) {
        int total = concurrency * replicas;
        if (total > partitions) {
            log.warn("kafka_over_provisioned total_consumers={} partitions={} idle={}",
                     total, partitions, total - partitions);
        }
    }
}
```

A warning at startup is worth more than a paragraph in a wiki.

---

### Trap 2 — exceeding `max.poll.interval.ms`, and the rebalance storm

**Wrong approach**

The notification consumer ships with defaults: `max.poll.records: 500`,
`max.poll.interval.ms: 300000`. Per-message work is one HTTP call to an external
provider, normally 200ms.

500 × 200ms = 100s. Comfortably inside 300s. It works fine for months.

Then the provider degrades to 900ms per call. 500 × 900ms = 450s.

**Exact symptom**

Lag on `orderflow.order-placed` for group `notifications` climbs steadily. Then it
climbs faster. Throughput approaches zero. Every consumer pod is running, logging,
and using CPU — nothing has crashed.

```bash
watch -n 5 "kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group notifications"
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group notifications --state
```

| What you see | What it means |
|---|---|
| `--state` shows `PreparingRebalance` or `CompletingRebalance` most times you look | **The group spends most of its life rebalancing.** This is the diagnosis. |
| `CONSUMER-ID` values change every time you run `--describe` | Members are being evicted and rejoining continuously. |
| `LAG` sawtooths — rising, small drop, rising higher | Each rebalance pauses the group; the brief consumption between rebalances cannot keep up. |
| Application logs repeat the `max.poll.interval.ms` WARN then `(Re-)joining group` | The three-line signature from Machine-level reality. |
| The same order confirmations sent multiple times | Offsets never committed before eviction — the duplicate window in action. |

Also check the metrics, which show it without log grepping:

```bash
curl -s localhost:8080/actuator/metrics/kafka.consumer.coordinator.rebalance.rate.per.hour | jq .
curl -s localhost:8080/actuator/metrics/kafka.consumer.coordinator.failed.rebalance.rate.per.hour | jq .
curl -s localhost:8080/actuator/metrics/kafka.consumer.fetch.manager.records.lag.max | jq .
```

**Root cause**

The batch time exceeded the budget, and — this is the part that turns a slow
consumer into an outage — **the eager rebalance pauses the entire group**, so lag
accumulates for everyone, so every consumer's next batch is a full one, so more
consumers exceed the budget. The mechanism is spelled out step by step in
Machine-level reality; re-read it if the cascade is not obvious.

The reason it appeared suddenly after months is that the budget was
`max.poll.records x per-message time`, and only one of those two factors was under
your control. A dependency's latency change silently rewrote your consumer's
liveness configuration.

**Fix — four changes, in order of how much they help**

**(a) Shrink the unit of work.** This is the primary fix.

```yaml
spring:
  kafka:
    consumer:
      max-poll-records: 10          # was 500
      properties:
        max.poll.interval.ms: 120000
```

10 × 900ms = 9s, against a 120s budget. Even at a 3-second worst case per message
that is 30s, still 4× inside. **Throughput is unchanged** — you poll more often for
fewer records. The only cost is slightly more coordinator traffic.

**(b) Switch to cooperative rebalancing.** This removes the cascade.

```yaml
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

Now one consumer timing out does not stop the other three. The storm becomes a
single slow consumer. Remember the two-step rolling upgrade if you are migrating a
live group.

**(c) Bound the per-message time so the budget cannot be rewritten by a dependency.**
Topic 111:

```java
@KafkaListener(topics = "orderflow.order-placed", groupId = "notifications")
public void onOrderPlaced(OrderPlacedEvent event) {
    // Breaker + bulkhead + a hard HTTP read timeout. The worst case is now BOUNDED,
    // which is what makes the max.poll.interval.ms arithmetic valid at all.
    gateway.sendOrderConfirmation(event.orderId(), event.customerEmail());
}
```

Without a hard timeout on the outbound call, "worst case per message" is unbounded
and **no** `max.poll.interval.ms` is large enough. This is the change that makes the
fix durable rather than a bigger number.

**(d) Pause instead of dying, when you know you will be slow.** Spring Kafka lets
you pause the container so it keeps polling (staying alive in the group) without
delivering records:

```java
@Component
class NotificationBackpressure {

    private final KafkaListenerEndpointRegistry registry;

    void onProviderDegraded() {
        registry.getListenerContainer("notifications").pause();
    }
    void onProviderRecovered() {
        registry.getListenerContainer("notifications").resume();
    }
}
```

A paused container still calls `poll()` — it just receives nothing — so the member
stays alive and no rebalance occurs. This is the correct response to an open circuit
breaker on a consumer: stop consuming *without leaving the group*.

**What NOT to do:**

- **Do not just raise `max.poll.interval.ms` to an hour.** It "fixes" the symptom
  and destroys your ability to detect a genuinely hung consumer: a deadlocked
  consumer (Topic 94) now holds its partitions for an hour before anyone notices.
- **Do not raise `session.timeout.ms`.** Wrong dial entirely — heartbeats are on a
  background thread and were never the problem.
- **Do not add consumers.** Trap 1.

---

### Trap 3 — moving work off the poll thread, and committing offsets for work that has not happened

**Wrong approach**

Having read Trap 2, someone concludes: get the slow work off the poll thread.

```java
@KafkaListener(topics = "orderflow.order-placed", groupId = "notifications")
public void onOrderPlaced(OrderPlacedEvent event) {
    executor.submit(() -> gateway.sendOrderConfirmation(event.orderId(), event.customerEmail()));
    // returns IMMEDIATELY -- the container commits the offset
}
```

```java
@Bean
ExecutorService executor() {
    return Executors.newCachedThreadPool();     // unbounded
}
```

The listener returns in microseconds. `max.poll.interval.ms` is never exceeded. Lag
goes to zero. It looks like a triumph.

**Exact symptom**

Three failures, appearing in this order over hours or days:

1. **Silently lost notifications.** The container committed the offset the moment the
   listener returned. If the pod is killed — a deploy, an OOMKill, a node drain —
   every task still in the executor's queue is gone, and Kafka believes those records
   were processed. **At-most-once, by accident.** You will not find this in logs;
   you find it when a customer says they never got a confirmation.
2. **Unbounded memory growth, then OOM.** `newCachedThreadPool` has an unbounded
   `SynchronousQueue` handoff and creates a thread per task when none is free. Kafka
   delivers faster than the provider responds, so thread count grows without limit
   until `OutOfMemoryError: unable to create native thread` (Topic 98).
3. **Ordering is destroyed.** Two events for the same `orderId` — carefully placed
   on the same partition to guarantee ordering — are now handed to two different
   threads and race. The partition's ordering guarantee, which was the entire reason
   for the key, is discarded in one line.

The observations:

```bash
# Lag looks GREAT. That is the trap.
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group notifications

# Thread count -- the real story.
jcmd <pid> Thread.print | grep -c 'pool-'
curl -s localhost:8080/actuator/metrics/jvm.threads.live | jq '.measurements'
```

| What you see | What it means |
|---|---|
| Lag near zero while thread count climbs steadily | The queue moved from Kafka (where it is durable and observable) into your heap (where it is neither). |
| `jvm.threads.live` growing without bound | `newCachedThreadPool` under sustained overload. Topic 98. |
| Confirmations missing after a pod restart, with no error anywhere | **Committed offsets for unprocessed work.** The defining symptom. |
| Duplicate or out-of-order confirmations for one order | The per-partition ordering guarantee is gone. |

**Root cause**

Kafka's offset commit means "I have processed up to here". By returning from the
listener before the work is done, you told Kafka a lie. Kafka's durable, replicated,
observable queue was replaced by an unbounded in-memory queue with none of those
properties.

This is Topic 90's lesson — an unbounded queue is not backpressure, it is a delayed
OOM — arriving through Kafka.

**Fix**

*Fix 1 — do not move the work off the thread; make the batch smaller.* Trap 2's fix.
This is the right answer in the large majority of cases and it keeps every guarantee
intact.

*Fix 2 — if you must parallelise, keep the commit tied to completion and preserve
per-key ordering.*

```java
@KafkaListener(topics = "orderflow.order-placed", groupId = "notifications")
public void onBatch(List<OrderPlacedEvent> events, Acknowledgment ack) {

    // BOUNDED. And group by key so events for one order stay on one thread,
    // preserving the ordering the partition key was chosen to give us.
    var byKey = events.stream().collect(Collectors.groupingBy(OrderPlacedEvent::orderId));

    var futures = byKey.values().stream()
        .map(group -> CompletableFuture.runAsync(
                () -> group.forEach(this::send), boundedExecutor))
        .toList();

    // BLOCK until all are done. This is what keeps the offset honest --
    // and it means the batch time is still charged to max.poll.interval.ms,
    // so the budget arithmetic still applies (with a smaller wall-clock number).
    CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new))
                     .orTimeout(30, TimeUnit.SECONDS)
                     .join();

    ack.acknowledge();
}
```

```java
@Bean
ExecutorService boundedExecutor() {
    return new ThreadPoolExecutor(
        8, 8, 0L, TimeUnit.MILLISECONDS,
        new ArrayBlockingQueue<>(64),                       // BOUNDED
        new ThreadPoolExecutor.CallerRunsPolicy());          // backpressure, Topic 90
}
```

Read what that does: it parallelises **within** a batch and still blocks until the
batch is complete before acknowledging. The offset stays honest. The budget still
applies — you have reduced wall-clock time per batch, not removed the constraint.

*Fix 3 — non-blocking retries via `@RetryableTopic`* for the failure path, so a slow
retry does not sit on the poll thread at all:

```java
@RetryableTopic(attempts = "4",
                backoff = @Backoff(delay = 2000, multiplier = 2.0),
                dltTopicSuffix = ".DLT")
@KafkaListener(topics = "orderflow.order-placed", groupId = "notifications")
public void onOrderPlaced(OrderPlacedEvent event) { ... }
```

This republishes failures to delay topics and consumes them later, rather than
sleeping inside the poll loop.

*Also, always:* graceful shutdown must drain the container. Topic 123.

```yaml
spring:
  kafka:
    listener:
      immediate-stop: false      # finish the current batch before stopping
```

---

### Trap 4 — a rolling deploy that causes eight rebalances

**Wrong approach**

Four consumer pods, a Deployment, `RangeAssignor` (the historical default), no
static membership. A routine rolling deploy.

**Exact symptom**

Every deploy produces a several-minute window of near-zero consumption and a lag
spike that recovers afterwards. It happens on every deploy, so it is normalised as
"deploy noise" and never investigated.

```bash
# Run during a deploy.
watch -n 2 "kafka-consumer-groups.sh --bootstrap-server \$KAFKA --describe --group inventory --state"
```

| What you see | What it means |
|---|---|
| The state alternates `Stable` / `PreparingRebalance` / `CompletingRebalance` repeatedly | Multiple sequential rebalances, one per membership change. |
| `LAG` rises for the whole deploy, then drains | The group consumed nothing during each rebalance. With eager assignment that is **all** partitions each time. |
| Rebalance count roughly `2 x pods` | One departure and one arrival per pod. |
| `kafka.consumer.coordinator.rebalance.rate.per.hour` spikes | The metric to alert on. |

**Root cause**

Two compounding causes.

**(a) Eager assignment.** Every membership change revokes every partition from every
member. Four pods restarting is eight membership changes and therefore eight
group-wide pauses.

**(b) No static membership.** Without `group.instance.id`, a departing pod is a
permanent departure as far as the coordinator is concerned, so it rebalances
immediately — and again when the replacement joins.

**Fix**

```yaml
spring:
  kafka:
    consumer:
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        group.instance.id: ${POD_NAME}        # requires a STABLE name -- StatefulSet
        session.timeout.ms: 45000             # must exceed a pod restart
```

With both:

- A pod going away with a static id does **not** trigger a rebalance. The
  coordinator holds its partitions for up to `session.timeout.ms`.
- The replacement rejoins with the same id and gets the same partitions back. **Zero
  rebalances for a rolling restart**, provided each pod restarts within the session
  timeout.
- If a restart *does* exceed the timeout, cooperative assignment limits the damage
  to that pod's partitions.

Plus Topic 123's graceful shutdown, so the departing consumer commits its offsets
before exiting:

```yaml
spring:
  kafka:
    listener:
      immediate-stop: false
  lifecycle:
    timeout-per-shutdown-phase: 45s
```

And the `terminationGracePeriodSeconds` in your Pod spec must exceed that, or the
kubelet `SIGKILL`s mid-drain and you get duplicates instead.

**The sizing constraint on `session.timeout.ms`:** it must be longer than a pod
restart (so static membership helps) and short enough that a genuinely dead pod is
noticed promptly. Measure your actual pod restart time — image pull, JVM start,
context refresh, first poll (Topic 122) — and set it above the p99 of that, not
above the p50.

---

### Trap 5 — adding partitions to a keyed topic

**Wrong approach**

The inventory group is at the 12-consumer ceiling and lag is still growing. The
obvious move:

```bash
kafka-topics.sh --bootstrap-server $KAFKA --alter \
  --topic orderflow.order-placed --partitions 24
```

**Exact symptom**

Immediately after the change, some orders are processed **out of order**. A
`stock-released` event is processed before the `stock-reserved` it reverses, so
inventory ends up permanently wrong for those orders. Optimistic-lock failures spike
(Topic 52). No error is logged anywhere, because nothing failed — the events were
simply consumed in the wrong sequence by two different consumers.

The damage is limited to orders with events **spanning the change**, which makes it
a small number of corrupted records buried in a large volume: the hardest kind of
bug to find and the hardest to explain afterwards.

```bash
# Show that a key's partition changed.
kafka-console-consumer.sh --bootstrap-server $KAFKA --topic orderflow.order-placed \
  --from-beginning --property print.key=true --property print.partition=true \
  | grep 'ORD-12345'
```

| What you see | What it means |
|---|---|
| The same key appearing on **two different partitions** | The default partitioner is `hash(key) % partitionCount`. Changing the count changed the mapping. **This is the bug.** |
| Events for that key on partition A before the change and partition B after | Two consumers now own that key's history, with no ordering relationship between them. |
| Old partitions keep their existing data | Adding partitions does not redistribute anything. Only new records use the new mapping. |

**Root cause**

The default partitioner is `murmur2(key) % numPartitions`. `numPartitions` is in the
formula. Change it and **every key's partition changes** for records produced after
the change.

Kafka never moves existing records. So for a window equal to your retention period,
a key's events exist on two partitions with no ordering between them.

And it is irreversible: **you cannot reduce partition count.** The only way back is
a new topic and a migration.

**Fix**

*If you have not done it yet:* choose the partition count from the most parallel
consumer you will ever need, and over-provision. Partitions are cheap in small
numbers; the cost is broker file handles, replication overhead and end-to-end
latency, and it is measured in thousands of partitions per broker, not tens. 12 for
a topic that may need 12-way parallelism is fine; 24 would also have been fine.

*If you must increase it on a live keyed topic*, the safe procedure is a topic
migration, not an `--alter`:

1. Create `orderflow.order-placed.v2` with the new partition count.
2. Start the consumer groups on **both** topics.
3. Switch the producer (the outbox relay, Topic 115) to v2 at a known instant.
4. Wait until the v1 consumer's lag reaches zero **and** stays there — the point at
   which every key's history on v1 is fully processed.
5. Stop consuming v1. Delete it after retention.

Step 4 is the one that makes it safe: the ordering hazard exists only while a key
has unprocessed events on both topics, and draining v1 to zero closes it.

*If you truly cannot migrate*, a custom partitioner that preserves the old mapping
for existing keys is possible — and is a bespoke, testable, permanent piece of
complexity you will regret. Prefer the migration.

**Before adding partitions at all, check that you actually need them:**

| Check | Command | If |
|---|---|---|
| Are all 12 consumers busy? | `--describe --members --verbose` | Any member idle → you are not at the ceiling |
| Is lag even across partitions? | `--describe` | Skewed → it is a key problem, not a partition-count problem |
| Is per-message processing efficient? | Container listener timer p99 | A 50ms handler that should be 2ms is 25× of free capacity |
| Are you rebalancing constantly? | `--describe --state` | Fix Trap 2 first; partitions will not help |

More partitions is the last resort, not the first.

---

## Hands-on proof

### Proof 1 — `--describe` is the primary instrument; learn to read it cold

```bash
export KAFKA=localhost:9092

kafka-consumer-groups.sh --bootstrap-server $KAFKA --list
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group inventory
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group inventory --state
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group inventory --members --verbose
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group inventory --offsets
```

Read `--describe` in this order every time:

1. **Number of distinct `CONSUMER-ID`s** → your actual parallelism. Compare to the
   partition count.
2. **`LAG` per partition** → even, or skewed to a few? Even means a throughput
   problem; skewed means a key problem.
3. **`LOG-END-OFFSET` minus `CURRENT-OFFSET`** → the same as `LAG`, but seeing both
   tells you whether the producer or the consumer is moving.
4. **Any `-` in `CONSUMER-ID`** → unowned partitions; a rebalance is in flight.

| What you see | What it means |
|---|---|
| Distinct consumer ids == partition count | Maximum parallelism reached |
| Lag rising uniformly across all partitions | Consumers are collectively too slow |
| Lag rising on 1–2 partitions only | Key skew, or one slow consumer instance |
| Lag flat and non-zero | Keeping up but permanently behind — a fixed backlog you never drained |
| Consumer ids change between two consecutive runs | Rebalancing continuously — Trap 2 |
| `--state` reports `Empty` | No members. The consumers are down, or the group id is misspelled. |
| `LOG-END-OFFSET` not moving | The **producer** stopped. Look upstream — the outbox relay (Topic 115). |

### Proof 2 — measure the rebalance pause directly

Instrument the callbacks. This is the cheapest way to see a group-wide pause.

```java
@Component
class RebalanceObserver implements ConsumerAwareRebalanceListener {

    private final ConcurrentHashMap<String, Long> revokedAtNanos = new ConcurrentHashMap<>();

    @Override
    public void onPartitionsRevokedBeforeCommit(Consumer<?, ?> c, Collection<TopicPartition> ps) {
        revokedAtNanos.put(key(c), System.nanoTime());
        log.warn("rebalance_revoke partitions={} client={}", ps.size(), key(c));
    }

    @Override
    public void onPartitionsAssigned(Consumer<?, ?> c, Collection<TopicPartition> ps) {
        Long start = revokedAtNanos.remove(key(c));
        if (start != null) {
            long pausedMs = Duration.ofNanos(System.nanoTime() - start).toMillis();
            log.warn("rebalance_complete paused_ms={} partitions={} client={}",
                     pausedMs, ps.size(), key(c));
        }
    }
}
```

**WHAT TO LOOK FOR:** the `paused_ms` value, and how many consumers log it for the
same event.

| What you see | What it means |
|---|---|
| Every consumer logs a `revoke` and a `paused_ms` for one membership change | **Eager assignment.** The whole group stopped. |
| Only the affected consumer logs it; others log nothing | **Cooperative assignment.** The others never stopped. This is the change you are trying to make. |
| `paused_ms` in the hundreds of milliseconds | Normal for a small group |
| `paused_ms` in the seconds | Large group, slow assignor, or a slow `onPartitionsRevoked` (are you committing synchronously to a slow broker?) |

Note: `System.nanoTime()` is correct **here** — this is an elapsed-time measurement
of a single multi-second event, not a microbenchmark. See the Measurement section
for where it is wrong.

### Proof 3 — prove the ordering guarantee, and prove its boundary

```bash
# All events for one order must land on ONE partition.
kafka-console-consumer.sh --bootstrap-server $KAFKA --topic orderflow.order-placed \
  --from-beginning \
  --property print.key=true --property print.partition=true \
  | awk '{print $1, $2}' | sort | uniq -c | sort -rn | head -20
```

**WHAT TO LOOK FOR:** each key should appear with exactly one partition number.

| What you see | What it means |
|---|---|
| Every key maps to exactly one partition | The ordering guarantee holds |
| A key on two partitions | The partition count changed (Trap 5), or the producer sent some records with a null key |
| One partition holding a large share of records | Key skew — one `orderId`, or a key that is not as unique as you think |

### Proof 4 — read the offset topic to see what "committed" means

```bash
kafka-console-consumer.sh --bootstrap-server $KAFKA \
  --topic __consumer_offsets --from-beginning \
  --formatter 'kafka.coordinator.group.GroupMetadataManager$OffsetsMessageFormatter' \
  | grep inventory | tail -20
```

The formatter class name has moved between Kafka versions; if it is rejected, check
your distribution's tools documentation rather than guessing. The point of the
exercise stands either way: **offsets are ordinary messages in an ordinary compacted
topic**, and seeing that once removes a lot of mystery about how a group resumes.

---

## Failure drill

**The assignment:** make the consumer slower than `max.poll.interval.ms`. Observe the
rebalance loop and lag growth in consumer metrics.

Every number below is yours. None is printed here.

### Step 0 — a controllable environment

```bash
kafka-topics.sh --bootstrap-server $KAFKA --create \
  --topic orderflow.drill.order-placed --partitions 6 --replication-factor 1
```

Deliberately small timeouts so the failure happens in tens of seconds rather than
tens of minutes:

```yaml
spring:
  kafka:
    consumer:
      group-id: drill-notifications
      auto-offset-reset: earliest
      max-poll-records: 20
      properties:
        max.poll.interval.ms: 15000       # 15s
        session.timeout.ms: 45000
        heartbeat.interval.ms: 3000
        # DELIBERATELY EAGER for step 3. Switched in step 5.
        partition.assignment.strategy: org.apache.kafka.clients.consumer.RangeAssignor
    listener:
      concurrency: 3
      ack-mode: BATCH
```

```java
@Component
class DrillConsumer {

    /** The knob. Change at runtime via the endpoint below. */
    private final AtomicLong perMessageMillis = new AtomicLong(50);

    @KafkaListener(topics = "orderflow.drill.order-placed", groupId = "drill-notifications")
    void consume(String payload) throws InterruptedException {
        Thread.sleep(perMessageMillis.get());
    }

    @PostMapping("/drill/speed") void speed(@RequestParam long ms) { perMessageMillis.set(ms); }
}
```

### Step 1 — establish the healthy baseline

```bash
curl -X POST 'localhost:8080/drill/speed?ms=50'     # 20 x 50ms = 1s, well inside 15s

# Produce a steady stream.
for i in $(seq 1 20000); do echo "ORD-$i:{\"orderId\":\"ORD-$i\"}"; done \
  | kafka-console-producer.sh --bootstrap-server $KAFKA \
      --topic orderflow.drill.order-placed \
      --property parse.key=true --property key.separator=:
```

Record, over five minutes: lag (should hover near zero), rebalance rate (should be
zero after the initial join), and processed records per second.

```bash
watch -n 5 "kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group drill-notifications"
curl -s localhost:8080/actuator/metrics/kafka.consumer.coordinator.rebalance.rate.per.hour | jq '.measurements'
```

**Write these three numbers down.** Everything after is a comparison to them.

### Step 2 — cross the threshold

```bash
curl -X POST 'localhost:8080/drill/speed?ms=900'    # 20 x 900ms = 18s > 15s
```

Do nothing else. Watch for five minutes.

**WHAT TO LOOK FOR:**

| What you see | What it means |
|---|---|
| The `max.poll.interval.ms` WARN, then `LeaveGroup`, then `(Re-)joining group` | The eviction. Note the wall-clock interval between successive occurrences. |
| `--describe --state` mostly showing `PreparingRebalance` / `CompletingRebalance` | The group spends most of its life rebalancing. |
| Consumer ids changing between consecutive `--describe` runs | Continuous eviction and rejoin. |
| Lag rising, with a small drop after each rebalance, then rising higher | The sawtooth. Each cycle loses ground. |
| `rebalance.rate.per.hour` climbing from zero to a large number | The metric that would page you. |
| **All three** consumers logging revoke/assign for one eviction | **Eager assignment: the whole group stopped.** This is the observation the drill exists for. |
| Records reprocessed after each rebalance | The uncommitted window. Count them if you can — it is your duplicate rate. |

Let it run long enough to confirm it does **not** recover on its own. That is the
defining property of a storm and the reason it is an outage rather than a blip.

### Step 3 — quantify the amplification

Two numbers make the case:

1. **Effective throughput** during the storm versus step 1. Records processed per
   minute, from `CURRENT-OFFSET` advancement over a fixed interval.
2. **Wasted work**: records processed but never committed, i.e. reprocessed after
   the next rebalance. Instrument with a counter keyed on `(partition, offset)` and
   count repeats, or simply count total listener invocations against committed
   offset advancement.

The ratio of (2) to (1) is how much of your consumer's CPU is being spent on work
that is thrown away.

### Step 4 — the wrong fix, so you can see it is wrong

```yaml
        max.poll.interval.ms: 600000     # 10 minutes. "Fixed."
```

| What you see | What it means |
|---|---|
| Rebalances stop; lag begins to drain | It "works" |
| Lag still grows if the produce rate exceeds the consume rate | You never fixed throughput, only liveness |
| Now kill -9 a consumer pod | **Its partitions are unowned for up to 10 minutes.** Measure this. It is the price you just paid. |

That last row is the cost. Record the detection time for a genuinely dead consumer
before and after. A large `max.poll.interval.ms` trades a real failure-detection
capability for a symptom suppression.

### Step 5 — the right fixes, one at a time, measuring each

**(a) Cooperative assignment only** (leave `max.poll.records` at 20 and the interval
at 15s):

```yaml
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

| What you see | What it means |
|---|---|
| Only the timing-out consumer logs revoke/assign; the other two keep processing | **The cascade is broken.** Throughput is degraded but non-zero. |
| Lag rises more slowly than in step 2 | Two thirds of the group is still working |
| The slow consumer still cycles | Cooperative assignment does not fix a slow consumer; it stops it from taking everyone down |

Record throughput. Compare to step 2 and to step 1.

**(b) Shrink the batch:**

```yaml
      max-poll-records: 5        # 5 x 900ms = 4.5s, inside 15s
```

| What you see | What it means |
|---|---|
| Rebalances stop entirely | The budget now fits |
| Throughput roughly equal to step 1's, adjusted for the 900ms per message | **Throughput did not drop.** Smaller batches, more polls, same records per second. This is the observation people find surprising and it is the point. |
| Lag drains at the rate the consumers can actually process | Honest backpressure |

**(c) Add the bound on per-message time.** Set the drill speed to something extreme
(`ms=10000`) and confirm that without a timeout **no** `max.poll.interval.ms` is
sufficient — then add a `TimeLimiter` or a hard HTTP read timeout (Topic 111) and
confirm the worst case is bounded again.

### Step 6 — the deliverable

A table with a row per configuration (baseline, storm, raised-interval,
+cooperative, +small-batch, +bounded-processing) and columns: records/second,
peak lag, rebalances/hour, reprocessed-record count, and detection time for a
`kill -9`'d consumer. Six rows, five columns, all your numbers.

Then one paragraph: which single change would you make first, in production, at 3am,
and why. (The answer is `max-poll-records`, because it is a config change with no
protocol implications, and cooperative assignment requires a two-step rolling
upgrade you do not want to attempt during an incident.)

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` measurement of consumer throughput is WRONG.** JIT
> warmup, deserialization costs that only appear at steady state, GC pauses, and
> the fetch-buffer prefetch all mean a hand-rolled timing loop measures something
> other than what you think. For per-message *code* performance, use JMH — **Topic
> 77**. `nanoTime` is legitimate for a single multi-second interval, such as the
> rebalance pause in Proof 2; it is not legitimate for "how fast is my handler".

The consumer-level questions are answered by Kafka's own metrics and by
`kafka-consumer-groups.sh`, not by a benchmark.

### The metrics that matter

```bash
curl -s localhost:8080/actuator/prometheus | grep -E '^kafka_consumer' | cut -d'{' -f1 | sort -u
```

*Illustration of the metric-name shape, not captured output:*

```
kafka_consumer_fetch_manager_records_lag
kafka_consumer_fetch_manager_records_lag_max
kafka_consumer_fetch_manager_records_consumed_rate
kafka_consumer_coordinator_rebalance_rate_per_hour
kafka_consumer_coordinator_failed_rebalance_rate_per_hour
kafka_consumer_coordinator_rebalance_latency_avg
kafka_consumer_coordinator_last_rebalance_seconds_ago
kafka_consumer_coordinator_commit_rate
spring_kafka_listener_seconds_count
```

| Metric | Answers | Alert on |
|---|---|---|
| `records_lag_max` | How far behind, worst partition | **Max, never sum.** One stuck partition among twelve is invisible in a sum. |
| `records_consumed_rate` | Actual throughput | A drop with lag rising |
| `rebalance_rate_per_hour` | How often the group reshuffles | **Any sustained non-zero value outside a deploy.** This is the storm alarm. |
| `failed_rebalance_rate_per_hour` | Rebalances that did not complete | Non-zero is always a problem |
| `last_rebalance_seconds_ago` | Time since the last one | Resetting to near zero repeatedly = storm |
| `spring_kafka_listener_seconds` (timer) | Per-message processing time | **p99, and compare it to `max.poll.interval.ms / max.poll.records`.** This is the budget check, as a live metric. |
| `commit_rate` | How often offsets are committed | Zero while consuming = offsets are not being committed |

### The one derived metric worth building

```
poll_budget_utilisation = (listener_p99_seconds x max.poll.records) / (max.poll.interval.ms / 1000)
```

Dashboard it per consumer group. It is a **leading indicator**: it crosses 1.0
*before* the first eviction, so you get an alert while the group is still healthy.
Alert at 0.5 (half the budget consumed) and treat 0.8 as an incident.

This single derived metric is the difference between "we were paged when lag hit a
million" and "we were paged when the provider got slow".

### Lag alerting, done correctly

Absolute lag is a poor alert. 10,000 records is a crisis at 10 records/second and
nothing at 10,000 records/second. Alert on **time**, not count:

```
estimated_lag_seconds = records_lag_max / records_consumed_rate
```

Alert when the estimate exceeds your freshness SLO (Topic 130) — for `orderflow`'s
inventory consumer, perhaps 60 seconds, because stock levels older than that produce
oversell.

Beware the division: when `records_consumed_rate` is zero — which is exactly what
happens during a storm — the estimate is infinite or undefined. Handle that case
explicitly with a separate "consumption stalled" alert on `records_consumed_rate ==
0 while records_lag_max > 0`.

### Duplicate-processing measurement

At-least-once means duplicates. Measure the rate rather than assuming it is zero.

```sql
-- With the idempotency table from Topic 116 in place, unique-violation rejections
-- ARE your duplicate count.
SELECT date_trunc('hour', created_at) AS hour, count(*) AS duplicates_rejected
FROM processed_event_rejections
GROUP BY 1 ORDER BY 1 DESC LIMIT 24;
```

| What you see | What it means |
|---|---|
| A small steady rate | Normal at-least-once behaviour. The relay occasionally republishes. |
| A spike aligned with a deploy | Rebalance duplicates. Expected — check the magnitude against the uncommitted window. |
| A spike with no deploy | A rebalance storm, or the relay is republishing. Correlate with `rebalance_rate_per_hour`. |
| Zero, always | Suspicious. Verify the counter is wired; at-least-once delivery producing literally zero duplicates over weeks usually means you are not counting. |

### Comparing against the Topic 65 baseline

The consumer side is not in the k6 numbers, but it is coupled to them: order
placement is 10% of the arrival rate and each placement produces one event.

| Measurement | Where from | Fill in |
|---|---|---|
| Produce rate to `orderflow.order-placed` at baseline | Relay metrics / `LOG-END-OFFSET` advancement | |
| `inventory` consume rate at baseline | `records_consumed_rate` | |
| `inventory` lag at steady state | `records_lag_max` | |
| `notifications` listener p99 | `spring_kafka_listener_seconds` | |
| `notifications` poll-budget utilisation | Derived, above | |
| Rebalances during a rolling deploy, eager vs cooperative | `rebalance_rate_per_hour` | |
| Duplicate rate at baseline | Topic 116's rejection counter | |
| Hikari `connections_pending` on the inventory consumer pods | Topic 109's metric | |

That last row matters and is easy to forget: `concurrency: 3` means three consumer
threads each holding a database connection during processing. Consumer concurrency
is a claim on the connection pool, and Topic 109's arithmetic applies to it exactly
as it does to request threads.

---

## Practice exercises

### Easy — read a group's state cold

Against the lab topic with 6 partitions, produce 10,000 keyed records and start
consumers in this sequence, running `--describe`, `--describe --state` and
`--describe --members --verbose` after each step and recording what changed:

1. One consumer.
2. Three consumers.
3. Six consumers.
4. Nine consumers.
5. Kill three consumers.

For each step, before running the commands, **predict** the number of distinct
consumer ids, the partitions per consumer, and whether throughput will change.
Success criterion: your predictions match for all five steps, and you can state in
one sentence why step 4 changed nothing.

### Medium — the two-group topology (combines Topics 50, 52, 61, 109, 111)

Implement both `orderflow` consumer groups against `orderflow.order-placed` and
demonstrate:

1. `inventory` applying reservations idempotently, with a Testcontainers Kafka +
   Postgres integration test (Topic 61) that delivers the same event twice and
   asserts the stock is decremented once.
2. `notifications` with a Resilience4j-guarded outbound call (Topic 111), and a test
   proving that when the breaker is open the listener returns in microseconds and the
   poll budget is unaffected.
3. The `poll_budget_utilisation` metric from the Measurement section, exposed for
   both groups, with a test that fails if either exceeds 0.5 under the drill load.
4. A `DefaultErrorHandler` with a DLT, and a test that a deserialization failure goes
   to the DLT **immediately** rather than being retried three times.
5. A measurement of how many of the inventory pod's 16 Hikari connections are held by
   consumer threads at `concurrency: 3` under load (Topic 109), and a written
   statement of the maximum `concurrency` that pod can support.

### Hard — production simulation on the spine

1. **Execute the Failure drill** in full and deliver its six-row table.
2. **Rolling-deploy rebalance count.** With `RangeAssignor` and no static membership,
   perform a rolling restart of 4 consumer pods and count rebalances and total pause
   time. Then add `CooperativeStickyAssignor`, re-measure. Then add
   `group.instance.id` from a StatefulSet ordinal, re-measure. Report all three, and
   state the residual pause you could not remove and why.
3. **Duplicate quantification.** During each of the three deploys above, count
   duplicate deliveries using Topic 116's rejection counter. Express the result as
   "duplicates per deploy" and relate it to the uncommitted window.
4. **The partition-count trap, safely.** On a scratch topic, produce keyed events,
   `--alter` the partition count, produce more, and demonstrate with
   `print.partition=true` that a key moved. Then execute the migration procedure from
   Trap 5's fix on a second scratch topic and show that no key moves. Time both.
5. **Consumer concurrency vs the connection pool.** Sweep `concurrency` over
   {1, 2, 3, 4, 6} on the inventory consumer at Topic 65's baseline produce rate.
   For each: consume rate, listener p99, `hikaricp_connections_pending`, and the p99
   of `GET /api/products` on the same pod. Find where consumer concurrency starts
   damaging HTTP latency. That crossover is the number you ship.
6. **Graceful shutdown.** With `immediate-stop: false` and a
   `terminationGracePeriodSeconds` deliberately set **below** the container's drain
   timeout, kill a pod under load and count duplicates. Then set it above and
   re-count. Report both, and state the correct relationship between the two
   settings as a formula (Topic 123).
7. **Write the runbook entry.** Twelve lines: how to check lag, how to tell a storm
   from a slow consumer from key skew, the exact `--describe` output to capture, what
   to change first, and what never to change during an incident.

---

## Interview questions

### Q1 — "We added consumers to increase throughput. Did it help?"

**MID-LEVEL:** "It should — more consumers means more parallelism."

**SENIOR:** "Only up to the partition count. A partition maps to at most one
consumer in a group, so on a 12-partition topic, consumer 13 is assigned nothing and
sits idle. It still joins the group, still heartbeats, still participates in every
rebalance — so it adds rebalance cost and zero throughput.

I'd check with `kafka-consumer-groups.sh --describe` and count distinct
`CONSUMER-ID`s. If that equals the partition count, adding consumers is definitively
not the fix, and `--members --verbose` will show the idle ones with empty partition
lists.

Then I'd work out which of four things is actually happening. Lag rising uniformly
across partitions means the consumers are collectively too slow — so make each
message cheaper. Lag on one or two partitions means key skew. Consumer ids changing
between two `--describe` runs means a rebalance storm, and adding consumers makes
that strictly worse. Only if all twelve are busy and the processing is already
efficient would I consider more partitions — and that's a topic-level change that
breaks per-key ordering for events spanning the change, so it needs a topic
migration rather than an `--alter`."

**What separates them:** the mid-level answer states a general truth about
parallelism. The senior answer knows the hard ceiling, names the command that proves
it in ten seconds, and has a differential diagnosis rather than one hypothesis.

**Follow-up:** *"We have 12 partitions and 12 consumers and lag is still growing.
Now what?"* — Make the per-message work cheaper first: batch the database writes,
kill the N+1 (Topic 50), cache the lookup. A handler doing 50ms of work that should
do 2ms is 25× of free capacity, which is more than doubling the partition count
would give and costs nothing structural.

### Q2 — "What is `max.poll.interval.ms` and how is it different from `session.timeout.ms`?"

**MID-LEVEL:** "They're both timeouts for the consumer. `session.timeout.ms` is the
heartbeat timeout."

**SENIOR:** "They answer two different questions and that's the whole point. Since
KIP-62, heartbeats run on a **background thread**, so `session.timeout.ms` asks 'is
the process alive' and a consumer stuck for four minutes inside a message handler is
still heartbeating happily. `max.poll.interval.ms` asks 'is it making progress' — the
gap between successive `poll()` calls on the application thread. Exceed it and the
consumer proactively sends a `LeaveGroup`.

Which means raising `session.timeout.ms` to fix a slow consumer does nothing, and
people do it constantly. It's the wrong dial.

The arithmetic that matters is that `max.poll.interval.ms` is a budget for
`max.poll.records` messages, not for one. Five hundred records at 900ms each is 450
seconds against a 300-second default. So a dependency getting slower silently
rewrites your consumer's liveness config, which is why my first fix is always
`max.poll.records`, not the interval — smaller batches, same throughput, no protocol
implications, safe to change during an incident."

**What separates them:** the mid-level answer knows there are two timeouts. The
senior answer knows *why* there are two (the background heartbeat thread), knows the
common wrong fix, and has the budget arithmetic ready.

**Follow-up:** *"Why not just set `max.poll.interval.ms` to an hour?"* — Because you
have then given up detecting a genuinely hung consumer. A deadlocked consumer (Topic
94) holds its partitions for an hour with no one noticing. You have swapped a real
failure-detection capability for symptom suppression.

### Q3 — "Explain a rebalance storm from first principles."

**MID-LEVEL:** "Consumers keep leaving and rejoining the group, so it never
stabilises."

**SENIOR:** "It's a positive feedback loop, and the loop is what makes it an outage
rather than a blip.

A consumer exceeds `max.poll.interval.ms` and leaves. With the classic eager
protocol, that triggers a rebalance in which **every** member revokes **every**
partition — so the whole group stops consuming until the assignment completes.
During that pause, producers keep producing, so lag grows on every partition. When
consumers resume, each faces a full `max.poll.records` batch, which is the worst case
for the processing budget. So the next consumer times out, and round we go. It does
not self-recover, because recovering requires draining the backlog and nobody is
draining it.

Cooperative sticky assignment breaks the loop by changing exactly one step: only the
partitions that actually move are revoked, so the other consumers never stop, lag
doesn't spike group-wide, and nobody else gets a full batch. The storm degrades into
one slow consumer, which is a much better problem.

I'd diagnose it from `--describe --state` mostly showing `PreparingRebalance`,
consumer ids changing between consecutive `--describe` runs, and
`rebalance_rate_per_hour` climbing. The fixes in order: shrink `max.poll.records`,
switch to cooperative sticky — remembering that's a two-step rolling upgrade, not
something to do mid-incident — and bound the per-message time with a hard timeout,
because without that the worst case is unbounded and no interval value is large
enough."

**What separates them:** the mid-level answer describes the observation. The senior
answer names the feedback loop, identifies which single step cooperative assignment
changes, and knows the cooperative migration has a rolling-upgrade constraint — the
detail that separates having read about it from having done it.

**Follow-up:** *"Would virtual threads help?"* — No. The constraint is a wall-clock
budget between `poll()` calls, not thread scarcity. `KafkaConsumer` is not
thread-safe and must be driven by one thread regardless. Virtual threads change the
cost of blocking, not the poll contract.

### Q4 — "We moved the slow work to a thread pool so the listener returns fast. Good idea?"

**MID-LEVEL:** "Yes — that stops the poll timeout and lag goes to zero."

**SENIOR:** "It stops the timeout and it breaks three other things.

The offset is committed when the listener returns, so you have told Kafka you
processed records you have only queued. Kill the pod — a deploy, an OOMKill, a node
drain — and everything in that queue is gone with the offset already advanced. That's
at-most-once by accident, and it's silent: no error, no log, just a customer who
never got their confirmation.

Second, if the executor is unbounded — `newCachedThreadPool` is the usual choice —
you have moved a durable, replicated, observable queue out of Kafka and into your
heap, where it has none of those properties. Thread count grows until
`OutOfMemoryError: unable to create native thread`.

Third, you destroyed ordering. Two events for the same `orderId` were on the same
partition specifically so they'd be processed in order. Handing them to two threads
races them, and for inventory that means a release processed before its reservation.

If I genuinely need in-batch parallelism, I group by key so one key stays on one
thread, use a bounded pool with `CallerRunsPolicy` for backpressure, and **block
until the batch completes** before acknowledging. That keeps the offset honest, keeps
ordering per key, and the batch time is still charged to the poll budget — I've
reduced wall-clock time, not removed the constraint.

But usually the right answer is just `max.poll.records: 10`."

**What separates them:** the mid-level answer optimises the visible metric. The
senior answer knows the metric got better *because a guarantee was discarded*, and
names all three lost guarantees.

**Follow-up:** *"How would you detect this in an existing codebase?"* — Look for a
listener that returns without awaiting its work, then check whether lag is
suspiciously low relative to a slow downstream. Lag near zero with a downstream you
know is slow is the signature.

### Q5 — "Someone increased partitions from 12 to 24 to get more parallelism. What happened?"

**MID-LEVEL:** "You get more parallelism. There might be some rebalancing."

**SENIOR:** "You get more parallelism and you break per-key ordering for every event
in flight.

The default partitioner is `murmur2(key) % numPartitions`. The partition count is in
the formula, so changing it changes the partition for essentially every key. Kafka
never moves existing records, so for a window equal to your retention period, one
`orderId`'s events exist on two partitions with no ordering relationship between
them — owned by two different consumers.

For `orderflow` that means a `stock-released` processed before the `stock-reserved`
it reverses, so inventory is permanently wrong for those orders. Nothing errors.
Nothing logs. It's a small number of corrupted records in a large volume, which is
the worst shape a bug can have.

It's also irreversible — you cannot reduce partition count.

The safe procedure is a topic migration, not an `--alter`: create a v2 topic with the
new count, run consumers on both, switch the producer at a known instant, wait until
v1's lag is zero and stays there, then stop consuming v1. Draining v1 to zero is what
closes the ordering hazard.

And before any of that I'd check whether the partition count is genuinely the
constraint — all consumers busy, lag even across partitions, per-message work already
efficient, no rebalance storm. More partitions is the last resort, not the first."

**What separates them:** the mid-level answer treats it as a scaling operation. The
senior answer knows the partitioner formula, derives the ordering break from it,
knows it's irreversible, and has the migration procedure — including the specific
step (drain to zero) that makes it safe.

**Follow-up:** *"How do you choose the partition count initially?"* — From the most
parallel consumer you will ever need, over-provisioned, and with a number that
divides evenly by plausible consumer counts. 12 divides by 1, 2, 3, 4, 6 and 12; 10
across 3 consumers is a 4/3/3 split where one consumer does 33% more work.

---

## Mental model checkpoint

1. A topic has 12 partitions. A group has 20 consumers. How many are doing work?
   What does `--describe --members --verbose` show for the rest?
2. State the two timeouts, what each measures, which thread measures it, and which
   one you change when a handler is slow.
3. Write the poll-budget formula. For `max.poll.records: 500` and a p99 handler time
   of 400ms, what interval do you need, and with how much headroom?
4. Walk the rebalance storm loop in six steps. Which single step does cooperative
   assignment change, and why is that sufficient?
5. A listener submits to an executor and returns. Name the three guarantees that
   breaks.
6. `--describe` shows lag rising on 2 of 12 partitions and zero on the rest. What is
   the diagnosis, and what is *not* the fix?
7. You change a keyed topic from 12 to 24 partitions. What breaks, for how long, and
   what is the safe alternative?

---

## Quick reference card

**The invariants**

- One partition → at most one consumer **per group**. Extra consumers idle.
- Ordering is guaranteed **within a partition only**.
- `poll()` is where everything happens: join, rebalance callbacks, commits, fetch.
- Budget: `max.poll.records x per-message-worst-case < max.poll.interval.ms`.

**The settings**

| Setting | Default | Change it when |
|---|---|---|
| `max.poll.records` | 500 | **First** fix for a poll timeout |
| `max.poll.interval.ms` | 300000 | Only after computing the budget; never as a blanket fix |
| `session.timeout.ms` | 45s | Static membership, or a slow-restarting pod |
| `heartbeat.interval.ms` | 3s | Rarely; keep ≈ 1/3 of session timeout |
| `partition.assignment.strategy` | Range (historical) | **Always set `CooperativeStickyAssignor`** |
| `group.instance.id` | unset | Static membership — needs a genuinely stable id |
| `auto.offset.reset` | `latest` | `earliest` when skipping is worse than reprocessing |
| `enable.auto.commit` | true (raw client) / false (Spring) | Keep it false |

**Diagnostics**

```bash
kafka-consumer-groups.sh --bootstrap-server $KAFKA --list
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group <g>
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group <g> --state
kafka-consumer-groups.sh --bootstrap-server $KAFKA --describe --group <g> --members --verbose
kafka-topics.sh --bootstrap-server $KAFKA --describe --topic <t>
```

**Reading `--describe`, in order**

1. Distinct `CONSUMER-ID` count vs partition count → parallelism ceiling
2. `LAG` distribution → uniform (throughput) or skewed (key)
3. `CONSUMER-ID` changing between runs → rebalance storm
4. `-` in `CONSUMER-ID` → rebalance in flight

**Gotchas checklist**

- [ ] Total consumers (`pods x concurrency`) does not exceed the partition count.
- [ ] `CooperativeStickyAssignor` is set — and migrated in two steps if live.
- [ ] The poll budget is computed from the **worst case**, with 3–4× headroom.
- [ ] The per-message worst case is **bounded** by a hard timeout.
- [ ] Container-level retries are small; long retries use `@RetryableTopic`.
- [ ] `auto.offset.reset` was chosen, not defaulted.
- [ ] No work is submitted to an executor without awaiting it before acknowledging.
- [ ] `onPartitionsRevoked` commits; `onPartitionsLost` does not.
- [ ] Consumers are idempotent (Topic 116) — delivery is at-least-once.
- [ ] Alerts on `rebalance_rate_per_hour` and on `poll_budget_utilisation`, not only
      on absolute lag.
- [ ] Consumer concurrency is counted against the Hikari pool (Topic 109).
- [ ] Graceful shutdown drains the container, and the pod grace period exceeds it.

---

## When would I use this at work?

**1. The first time consumer lag pages you.** The differential diagnosis in Trap 1's
fix table — under-parallel, at the ceiling, slow processing, storm, key skew — is the
first five minutes of that incident, and `--describe` answers all five. Without it,
the default action is "add consumers", which is right one time in five.

**2. Sizing a new topic.** Partition count is close to irreversible for a keyed
topic, so the ten minutes spent choosing it is the highest-leverage ten minutes in
the design. "How parallel will the most parallel consumer ever need to be, and does
this number divide evenly?" is the whole conversation.

**3. When a deploy has a reliable lag spike everyone has normalised.** Cooperative
assignment plus static membership usually takes it to zero, and the before/after
measurement is a satisfying, cheap, visible win that also removes a duplicate-delivery
source you were silently paying for.

---

## Connected topics

**Backwards**

- **50 — N+1.** A consumer doing a query per message is the most common reason
  per-message time blows the poll budget.
- **52 — locking.** The inventory consumer's conditional UPDATE; duplicates and
  reordering are oversell.
- **55 / 109 — transactions and the pool.** Consumer concurrency is a claim on the
  connection pool; a transaction spanning a slow call inside a listener is both a
  pool problem and a poll-budget problem.
- **61 — Testcontainers.** The only honest way to test rebalancing and offsets.
- **65 — the baseline.** Produce rate is 10% of the arrival rate; consumer lag is the
  downstream half of the same load test.
- **77 — JMH.** Where per-message code performance is measured. Not with `nanoTime`.
- **90 / 98 — pools and thread leaks.** Trap 3's unbounded executor.
- **94 — deadlock.** What a large `max.poll.interval.ms` stops you from detecting.
- **111 — Resilience4j.** Bounding per-message time is what makes the poll budget
  valid; pausing the container is the right response to an open breaker.

**Forwards**

- **114 — delivery semantics.** At-least-once versus transactional read-process-write;
  why the Postgres write is outside any Kafka guarantee.
- **115 — the outbox.** The producer side of this topic; the relay is at-least-once by
  construction.
- **116 — idempotency.** Mandatory for every consumer here. The duplicate window is
  the gap between `position` and `committed`.
- **117 — sagas.** Order-placement steps arriving as events; compensations are
  consumers too.
- **118 — metrics.** The lag, rebalance and poll-budget series; alert on max lag, not
  sum.
- **119 / 120 — tracing and MDC.** There is no HTTP request on a consumer thread. The
  trace context and correlation id must travel in **message headers**, which makes
  them a schema decision.
- **121 — probes.** Should a consumer with a stalled group fail readiness? Usually no
  — it is not serving HTTP — but it must alert.
- **123 — graceful shutdown.** Draining the container, committing offsets, and the
  relationship between the container timeout and `terminationGracePeriodSeconds`.
- **130 — SLOs.** Freshness as an SLO: lag in seconds, not records.
- **133 — postmortems.** "We added consumers" and "we raised the timeout" are the two
  most common contributing factors in Kafka incidents.
