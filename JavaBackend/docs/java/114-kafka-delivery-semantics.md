# 114 — Kafka II: Delivery Semantics — At-Least-Once vs Exactly-Once

## Phase: 11 — Distributed Systems & Production
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21
## Project spine: `orderflow`'s order-placed producer and its inventory consumer get an explicit, written-down delivery contract. The producer is made idempotent. One consumer path is converted to a transactional read-process-write to prove what that does and does not cover. The Postgres write stays outside the guarantee, deliberately, and Topics 115–116 close that gap.

---

## Mechanical statement

Read this twice. The rest of the document is an elaboration of it.

> **Kafka's "exactly-once" is two mechanisms bolted together:**
>
> 1. An **idempotent producer** — the broker assigns your producer a numeric
>    **producer ID (PID)**, an **epoch**, and tracks a **monotonic sequence number per
>    partition**. A retried duplicate carries a sequence number the broker has already
>    seen, and the broker drops it and answers "fine".
>
> 2. A **transactional read-process-write** — your producer writes output records *and*
>    the input topic's consumer offsets under one transaction ID, and a **transaction
>    coordinator** commits or aborts them together by writing **control records** into
>    every partition involved. Consumers configured `isolation.level=read_committed`
>    then skip the aborted ones.
>
> **This is a guarantee about records that live inside Kafka.**
>
> It says nothing about a row you wrote in Postgres. It says nothing about an HTTP
> call you made to a payment gateway. It says nothing about an email you sent. Those
> side effects are not in the transaction, cannot be rolled back by the transaction
> coordinator, and will happen again when the record is redelivered.
>
> **Exactly-once is a property of the EFFECT, not of the transport.**

There is no configuration flag that makes a Postgres INSERT or a `POST /charges`
participate in a Kafka transaction. There never will be. The only thing that makes an
effect exactly-once is that the effect itself is idempotent — which is Topic 116, and
which is why this document exists before it.

---

## The bridge from what you know

You already know the delivery-semantics taxonomy. You have drawn the ack diagrams. So
this section will not re-teach at-most-once / at-least-once / exactly-once. It teaches
the four places where the *Kafka-specific mechanics* diverge from the mental model you
built on SQS, RabbitMQ, or BullMQ.

### Divergence 1 — there is no per-message ack

In SQS you delete a message. In RabbitMQ you `basicAck` a delivery tag. Both are
**per-message** and both are **random-access**: acking message 7 while message 5 is
still in flight is normal and supported.

Kafka has no such thing. A Kafka consumer commits **one integer per partition** — the
offset it wants to resume from. That single integer is a claim about *everything
before it*. There is no way to say "I processed 5 and 7 but not 6".

Consequences you must internalise:

- **Re-delivery is a range, not a message.** A crash re-delivers every record from the
  last committed offset forward, not just the one you were working on. Your consumer
  must be idempotent for a *batch*, not for a message.
- **Out-of-order completion is your problem.** If you fan a poll batch out to a thread
  pool (Topic 90), you cannot commit until the *lowest* incomplete offset finishes, or
  you will skip records on the next crash. Most hand-rolled "parallel consumers" get
  this wrong.
- **There is no dead-letter mechanism in the broker.** A DLQ is something your client
  code produces to. Spring Kafka's `DeadLetterPublishingRecoverer` is a client-side
  producer, not a broker feature.

### Divergence 2 — "acks" is about replication, not about processing

`acks=all` means "the leader and all in-sync replicas have this record in their log".
It does not mean a consumer saw it. It does not mean anyone processed it. Engineers
coming from HTTP-shaped systems read `acks=all` as an end-to-end acknowledgement. It
is a **durability** setting only.

The durability triangle is `acks` × `min.insync.replicas` × `replication.factor`:

- `acks=all` with `min.insync.replicas=1` and `replication.factor=3` is **not** durable
  against one broker loss, because "all in-sync replicas" can legitimately be one.
- `acks=all` with `min.insync.replicas=2` and `replication.factor=3` is the standard
  safe configuration, and it means a produce fails with
  `NotEnoughReplicasException` when only one replica is alive. That failure is the
  point: it is the system refusing to accept data it cannot keep.

### Divergence 3 — the idempotent producer solves a smaller problem than you think

The producer retry duplicate is real: producer sends, broker writes, the ack is lost
on the network, producer retries, broker writes again. Two identical records. The
idempotent producer fixes exactly that.

It does **not** fix:

- **Application-level retries.** If your code catches an exception and calls
  `send()` again with a new `ProducerRecord`, that is a new sequence number. It is a
  genuinely new record as far as the broker is concerned. The dedup window is inside
  the producer client's retry loop, not around your business logic.
- **Producer restarts.** A new producer instance gets a *new* PID (unless you set
  `transactional.id`). Everything it sends is new. A `kill -9` between "wrote the row"
  and "published the event" is not helped at all — that is Topic 115.

### Divergence 4 — `read_committed` changes what "lag" means

A `read_committed` consumer cannot read past the **Last Stable Offset (LSO)** — the
offset before the oldest still-open transaction. So an open transaction on a partition
**stalls consumption of that partition entirely**, even for records that were committed
after it. A hung transactional producer with a long `transaction.timeout.ms` looks
exactly like a dead consumer: lag climbing, no errors.

Also: transaction **control records occupy offsets**. A `read_committed` consumer
never delivers them to you, so its position can advance past records you never saw,
and computed lag can sit at a small non-zero value on an idle partition. Do not build
a "lag must be zero" alert on a transactional topic.

### The honest verdict table

| What you know | Kafka mechanic | Verdict |
|---|---|---|
| At-least-once + idempotent consumer | Exactly the right default here too | **TRANSFERS** |
| Per-message ack / nack / requeue | Does not exist; one offset per partition | **NO ANALOGUE** — this is the one to relearn |
| Visibility timeout (SQS) | `max.poll.interval.ms` + rebalance (Topic 113) | **PARTIAL** — the failure mode is a group-wide stall, not one redelivered message |
| Broker-managed DLQ | Client-side only | **PARTIAL** |
| "Exactly-once mode" in a queue product | Only within Kafka; never across a sink | **PARTIAL, and the delta is the whole lesson** |

---

## What is this?

Three separate features, usually discussed as one, which is the source of most of the
confusion.

### Feature 1 — the idempotent producer

```properties
enable.idempotence=true
```

Default `true` since Kafka 3.0. Turning it on forces, and will fail startup if you
contradict:

- `acks=all`
- `max.in.flight.requests.per.connection <= 5`
- `retries > 0`

On first send the producer calls `InitProducerId` and receives a **PID**. Every record
batch it sends to a partition carries `(PID, epoch, baseSequence)`. The broker keeps
the last five sequence numbers per `(PID, partition)` in memory and in the log's
producer-state snapshot.

- Sequence equals one already seen → broker replies `DUPLICATE_SEQUENCE_NUMBER`, which
  the client treats as **success**. No duplicate is written.
- Sequence higher than expected → `OUT_OF_ORDER_SEQUENCE_NUMBER`. The batch is
  rejected. This is why the in-flight limit is five: the broker's dedup window is five
  deep, so more in-flight requests could reorder past it.

Scope of the guarantee: **one producer session, per partition.** Not across restarts.
Not across partitions. Not across your application's own retry logic.

### Feature 2 — transactions

```properties
transactional.id=orderflow-inventory-projector-0
enable.idempotence=true
```

Setting `transactional.id` gives you a **stable** PID across restarts and adds
**fencing**: when a new instance calls `initTransactions()` with the same
transactional ID, the coordinator bumps the epoch. Any older instance still holding the
previous epoch gets `ProducerFencedException` on its next operation and must die. That
is the zombie-fencing property, and it is the reason a transactional ID must be stable
*and* unique per logical producer instance.

The Java shape:

```java
producer.initTransactions();                 // once, at startup
try {
    producer.beginTransaction();
    for (var record : outputs) producer.send(record);
    producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
    producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
    producer.close();                        // fatal: this instance must not continue
} catch (KafkaException e) {
    producer.abortTransaction();
}
```

`sendOffsetsToTransaction` is the load-bearing line. It writes the consumer offsets to
the internal `__consumer_offsets` topic **as part of the same transaction as your
output records**. That is what makes "read, process, write" atomic: either the outputs
and the new offsets both land, or neither does and you reprocess.

### Feature 3 — `read_committed` consumers

```properties
isolation.level=read_committed
```

Default is `read_uncommitted`. This is the single most commonly missed line. A team
enables transactions on the producer, does not change the consumer, and gets aborted
records delivered downstream. Everything looks fine until the first abort.

### What "exactly-once" therefore is

Exactly-once semantics in Kafka = idempotent producer + transactional read-process-write
+ `read_committed` consumers, forming a **closed loop inside Kafka**. Kafka Streams
packages the whole loop behind `processing.guarantee=exactly_once_v2` because Streams
owns both ends. When you write a plain consumer that reads Kafka and writes Postgres,
you own one end and Kafka owns the other, and the loop is not closed.

---

## Why does it matter?

**1. The phrase is used as a load-bearing assumption in design reviews, and it is
usually false.**

"We have exactly-once, so we don't need idempotency at the sink" is a sentence that
ships double charges. The wallet debit in `orderflow` is a Postgres UPDATE. It is not
in any Kafka transaction. Under redelivery it runs twice.

**2. The failure is silent and financial.**

A duplicate inventory decrement shows up as phantom stock-outs. A duplicate wallet
debit shows up as a customer support ticket three days later. Neither raises an
exception, neither increments an error counter, and neither appears in a trace as
anything other than two successful requests.

**3. Enabling it has real costs that nobody budgets for.**

Transactions add a coordinator round trip per transaction, control records in every
partition, and `read_committed` consumers blocked behind the LSO. Committing per record
is catastrophic for throughput; committing per poll batch is the only sane granularity.
Teams enable EOS "for safety", lose throughput, and still have the duplicate charge.

**4. It is the cleanest senior/mid filter in the distributed-systems section of an
interview.**

Because the mid answer ("we turned on exactly-once") and the senior answer are not
different levels of detail. They are different claims about what is true.

---

## Machine-level reality

Everything in this section is checkable with a command. Where I am unsure of a default
that changed across Kafka versions, I say so and give you the command.

### The producer ID, epoch and sequence number

Every record batch header in the Kafka log format (v2, since Kafka 0.11) contains:

```
baseOffset, batchLength, partitionLeaderEpoch, magic, crc,
attributes  (bit 4 = isTransactional, bit 5 = isControlBatch),
lastOffsetDelta, baseTimestamp, maxTimestamp,
producerId, producerEpoch, baseSequence,
recordsCount
```

That is not documentation prose — it is the on-disk layout, and you can print it:

```bash
kafka-dump-log.sh \
  --files /var/lib/kafka/data/orderflow.order-placed-0/00000000000000000000.log \
  --print-data-log | head -40
```

**WHAT TO LOOK FOR:** the `producerId`, `producerEpoch`, `baseSequence`,
`isTransactional` and `isControl` fields on each batch line.

| What you see | What it means |
|---|---|
| `producerId: -1`, `baseSequence: -1` | Non-idempotent producer. `enable.idempotence` is off, or an old client. No dedup is possible for this batch. |
| `producerId: <n>`, `producerEpoch: 0`, `baseSequence` increasing per batch | Idempotent producer, first session. The broker is deduplicating retries for this PID. |
| `producerEpoch` higher than you expect | The transactional ID was re-initialised. A previous instance was fenced. If you did not deploy, something restarted — investigate before assuming it is benign. |
| `isTransactional: true` | The batch belongs to a transaction. A `read_committed` consumer will not deliver it until a matching control batch commits it. |
| `isControl: true` | This is a transaction marker (COMMIT or ABORT). It occupies an offset and is never delivered to your application. |
| Two batches with the **same** `producerId` and the **same** `baseSequence` | Should not happen. If it does, you are looking at two different producer *sessions* that were assigned the same PID after a coordinator issue, or at a log from before the batch was deduplicated. Capture it and escalate. |

The broker stores producer state in `*.snapshot` files alongside the log segments, and
expires it after `transactional.id.expiration.ms` (broker-side) and
`producer.id.expiration.ms`. **If a producer is idle longer than the expiry, its state
is discarded and dedup no longer applies to it.** That is a real hole for low-volume
topics and almost nobody knows it. Confirm the current defaults on your build with
`kafka-configs.sh --describe --entity-type brokers --entity-default` rather than
trusting a blog post; these values have moved between versions.

### The transaction coordinator

The coordinator is a **broker**, selected as the leader of the `__transaction_state`
partition that `hash(transactional.id) % numPartitions` lands in. That has three
consequences:

1. **The transactional ID determines which broker coordinates you.** Two instances with
   the same transactional ID contact the same coordinator — that is how fencing can
   work at all.
2. **Coordinator failover is a real pause.** If that broker dies, the coordinator moves
   with the partition leadership, and in-flight transactions wait.
3. **`transactional.id` must be stable across restarts, and distinct per instance.**
   If two live instances share one ID, they fence each other in a loop. If an instance
   gets a random ID at startup, you lose fencing entirely and you leak transactional-ID
   state on the coordinator until expiry. On Kubernetes, derive it from the StatefulSet
   ordinal or the assigned partitions — not from a UUID, and not from the pod name of a
   Deployment.

The commit protocol, in order:

```
AddPartitionsToTxn      (coordinator learns which partitions are involved)
Produce                  (records written to partition leaders, marked transactional)
AddOffsetsToTxn +
  TxnOffsetCommit        (offsets written to __consumer_offsets, also transactional)
EndTxn                   (coordinator writes PREPARE_COMMIT to __transaction_state)
WriteTxnMarkers          (coordinator writes a COMMIT control record to EVERY
                          involved partition, including __consumer_offsets)
                         (coordinator writes COMPLETE_COMMIT)
```

The commit is a **two-phase commit with the coordinator's log as the decision record**.
`commitTransaction()` returns once `PREPARE_COMMIT` is durable; the markers are written
asynchronously. So a commit is decided before it is visible everywhere. That gap is why
`read_committed` consumers can briefly stall at the LSO after a commit.

Inspect live transactions:

```bash
kafka-transactions.sh --bootstrap-server localhost:9092 list
kafka-transactions.sh --bootstrap-server localhost:9092 \
  describe --transactional-id orderflow-inventory-projector-0
kafka-transactions.sh --bootstrap-server localhost:9092 \
  find-hanging --topic orderflow.order-placed
```

**WHAT TO LOOK FOR:** the transaction state and its start timestamp.

| What you see | What it means |
|---|---|
| State `Ongoing` with an old start timestamp | A hung transaction. Every `read_committed` consumer on the involved partitions is blocked at the LSO. This is your lag incident. |
| State `CompleteCommit` / `CompleteAbort` | Terminal. Nothing is blocked. |
| `find-hanging` returns a producer ID | You have a partition with an open transaction from a producer that is gone. It will clear at `transaction.timeout.ms`, or you abort it with `kafka-transactions.sh abort`. Do not abort one you have not identified. |
| `list` shows dozens of transactional IDs you do not recognise | Randomly generated transactional IDs. You have no fencing and you are leaking coordinator state. Fix the ID derivation. |

### `transaction.timeout.ms` and the two-sided cap

The producer's `transaction.timeout.ms` (client-side, default one minute at time of
writing — verify with `kafka-configs.sh`) must not exceed the broker's
`transaction.max.timeout.ms` (default 15 minutes), or the producer fails to initialise.

Set it deliberately. It is the maximum time a `read_committed` consumer can be blocked
by one stuck producer. A 15-minute timeout on a hot topic is a 15-minute outage
waiting for a bad deploy.

### Where the guarantee ends, precisely

```
+-------------------------------------------------------+
|                  KAFKA TRANSACTION                    |
|                                                       |
|   input offsets  ->  output records                   |
|   (__consumer_offsets)   (orderflow.inventory-changed)|
|                                                       |
+-------------------------------------------------------+
             |                            |
             |  NOT IN THE TRANSACTION    |
             v                            v
   INSERT INTO payments(...)     POST https://gateway/charges
   UPDATE wallets SET ...        (already sent; cannot be unsent)
   (already committed or not;
    the coordinator has no
    say either way)
```

The coordinator can write an ABORT marker. It cannot issue a Postgres `ROLLBACK`, and
it cannot un-send an HTTP request. There is no distributed transaction manager here and
adding XA does not help — Postgres supports prepared transactions, Kafka's protocol is
not XA, and a payment gateway has no two-phase commit interface at all.

### `[BOOT 3.x DELTA]` — Spring's transaction plumbing

Spring for Apache Kafka gives you `KafkaTransactionManager` and
`KafkaTemplate.executeInTransaction(...)`. On Boot 2.x you may still find
`ChainedKafkaTransactionManager`, which wrapped a JPA transaction manager and a Kafka
transaction manager and was **removed in Spring for Apache Kafka 3.0**. It was removed
because it was routinely misread as a distributed transaction. It never was one: it
synchronised the two commits so that one begins after the other, leaving a window in
between. If you inherit a codebase using it, that codebase has the dual-write bug of
Topic 115 with a reassuring class name on top.

On Boot 4.1 the mechanism is the same as 3.x: `KafkaTransactionManager` for
Kafka-to-Kafka, and an outbox for anything involving the database. Do not invent a
version number for `spring-kafka` — take whatever the Boot BOM resolves, and check it
with `./mvnw dependency:tree -Dincludes=org.springframework.kafka`.

---

## Example 1 — minimal

Two producers, one topic, one difference. The point is to see the sequence numbers.

```java
package com.orderflow.lab.kafka;

import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import java.util.Properties;

/**
 * Sends the same three records twice, forcing a client retry by using a very short
 * request timeout against a deliberately slow broker (or by network-partitioning the
 * container mid-run). Run it once with idempotence on and once with it off.
 */
public class IdempotenceProbe {

    public static void main(String[] args) {
        boolean idempotent = Boolean.parseBoolean(args[0]);

        Properties p = new Properties();
        p.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        p.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        p.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        p.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, idempotent);
        p.put(ProducerConfig.ACKS_CONFIG, idempotent ? "all" : "1");
        p.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        p.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120_000);
        p.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 250);   // provoke retries
        p.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        try (var producer = new KafkaProducer<String, String>(p)) {
            for (int i = 0; i < 3; i++) {
                var record = new ProducerRecord<>(
                        "orderflow.lab.delivery",
                        "SKU-4471",
                        "{\"orderId\":\"ORD-" + i + "\",\"sku\":\"SKU-4471\",\"units\":1}");
                producer.send(record, (md, ex) -> {
                    if (ex != null) System.out.println("FAILED " + ex);
                    else System.out.println("acked partition=" + md.partition()
                            + " offset=" + md.offset());
                });
            }
            producer.flush();
        }
    }
}
```

Run it, then read the log:

```bash
java IdempotenceProbe false
kafka-dump-log.sh --files <segment>.log --print-data-log | grep -E "producerId|payload"

java IdempotenceProbe true
kafka-dump-log.sh --files <segment>.log --print-data-log | grep -E "producerId|payload"
```

**WHAT TO LOOK FOR:** the count of records with the same `orderId` payload, and the
`producerId` / `baseSequence` fields.

| What you see | What it means |
|---|---|
| Run 1 writes more records than you sent, `producerId: -1` | A retry duplicated a batch. Nothing could deduplicate it: there was no PID and no sequence number. This is the raw at-least-once shape. |
| Run 2 writes exactly what you sent, `producerId` set, `baseSequence` strictly increasing | The broker deduplicated the retried batch. You have just observed the idempotent producer working. |
| Run 1 writes exactly what you sent | Your retry never fired. Lower `request.timeout.ms` further, or add packet delay to the broker container (`tc qdisc add dev eth0 root netem delay 400ms`). Absence of a duplicate is not proof of dedup. |
| Run 2 fails at startup with a config exception | You contradicted a forced setting — `acks` not `all`, or in-flight above five, or `retries=0`. The client refuses rather than silently weakening the guarantee. Good client design; read the message. |

Now the second half, which is the actual lesson:

```java
    // Add this to the send callback and re-run BOTH configurations.
    producer.send(record, (md, ex) -> {
        if (ex == null) {
            jdbc.update("insert into inventory_ledger(sku, delta) values (?, ?)",
                        "SKU-4471", -1);      // <-- NOT in any Kafka guarantee
        }
    });
```

The `inventory_ledger` row count is identical in both runs and is wrong in both runs
if the JVM is killed between the ack and the insert. **The idempotent producer changed
what is in Kafka. It changed nothing about what is in Postgres.** That sentence is the
whole topic in one experiment.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline:

- 100k products, 1M orders, 5M order lines, containerised, k6 open-model load.
- Scenario mix 70% catalogue read / 20% order read / 10% order placement.
- `orderflow.order-placed` has 12 partitions, keyed by `orderId`, `replication.factor=3`,
  `min.insync.replicas=2`.
- Two consumer groups from Topic 113: `inventory-projector` and `notification-sender`.
- Placement is transactional across four writes (Topic 54): reserve inventory, debit
  wallet, insert payment, insert order.

Two consumers, two different correct answers. That contrast is the point.

### Consumer A — `inventory-projector`: Kafka in, Kafka out. EOS applies.

This consumer reads `orderflow.order-placed`, computes a stock delta, and produces to
`orderflow.inventory-changed`. Both ends are Kafka. This is the *only* shape where
Kafka's exactly-once is the complete answer.

```java
package com.orderflow.inventory.projector;

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.TopicPartition;
import java.time.Duration;
import java.util.*;

public final class InventoryProjector implements Runnable {

    private final KafkaConsumer<String, OrderPlaced> consumer;
    private final KafkaProducer<String, InventoryChanged> producer;

    public InventoryProjector(String instanceOrdinal) {
        var c = new Properties();
        c.put(ConsumerConfig.GROUP_ID_CONFIG, "inventory-projector");
        c.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);   // mandatory for EOS
        c.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        c.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 200);
        c.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
              "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
        this.consumer = new KafkaConsumer<>(c);

        var p = new Properties();
        // Stable across restarts, unique per instance. StatefulSet ordinal, NOT a UUID.
        p.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,
              "orderflow-inventory-projector-" + instanceOrdinal);
        p.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        p.put(ProducerConfig.ACKS_CONFIG, "all");
        p.put(ProducerConfig.TRANSACTION_TIMEOUT_MS_CONFIG, 30_000);
        this.producer = new KafkaProducer<>(p);
    }

    @Override
    public void run() {
        producer.initTransactions();                       // fences any older epoch
        consumer.subscribe(List.of("orderflow.order-placed"));

        while (!Thread.currentThread().isInterrupted()) {
            ConsumerRecords<String, OrderPlaced> records = consumer.poll(Duration.ofMillis(500));
            if (records.isEmpty()) continue;

            try {
                producer.beginTransaction();

                for (var record : records) {
                    var change = project(record.value());
                    producer.send(new ProducerRecord<>(
                            "orderflow.inventory-changed", change.sku(), change));
                }

                // The load-bearing line: offsets join the SAME transaction.
                Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
                for (var partition : records.partitions()) {
                    var forPartition = records.records(partition);
                    long last = forPartition.get(forPartition.size() - 1).offset();
                    offsets.put(partition, new OffsetAndMetadata(last + 1));
                }
                producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());

                producer.commitTransaction();

            } catch (ProducerFencedException | OutOfOrderSequenceException e) {
                // FATAL. Another instance owns this transactional.id. Die; do not retry.
                producer.close();
                throw e;
            } catch (KafkaException e) {
                producer.abortTransaction();
                // Rewind to the committed offsets so the batch is reprocessed cleanly.
                for (var partition : records.partitions()) {
                    var committed = consumer.committed(Set.of(partition)).get(partition);
                    consumer.seek(partition, committed == null ? 0L : committed.offset());
                }
            }
        }
    }

    private InventoryChanged project(OrderPlaced e) { /* pure function */ }
}
```

Four details a reviewer should challenge, and the answers:

1. **Why one transaction per poll batch and not per record?** A transaction costs a
   coordinator round trip plus one control record per involved partition. Per record at
   the baseline's placement rate, that is a multiplier on broker write volume and a
   collapse in throughput. Per batch is the correct granularity. The cost of a larger
   batch is that a failure reprocesses the whole batch — which is fine, because the
   output is keyed and the downstream is idempotent.
2. **Why `read_committed` on a consumer that is reading from a non-transactional
   producer?** Because the upstream producer might become transactional later, and
   because the setting is free when there is nothing to filter. Setting it
   defensively costs you nothing and closes a future foot-gun.
3. **Why does `ProducerFencedException` kill the process?** Because it means another
   process has claimed this transactional ID. Continuing means two writers with the
   same identity. The only correct response is to exit and let the orchestrator decide.
4. **Why `seek` back after an abort?** Because the abort rolled back the *offset*
   commit too, but the consumer's in-memory position has already advanced past the
   batch. Without the seek, the next poll continues from the in-memory position and you
   silently skip the batch you just aborted. This is the single most common bug in
   hand-written EOS consumers.

### Consumer B — `payment-settler`: Kafka in, Postgres and HTTP out. EOS does NOT apply.

This consumer reads `orderflow.payment-authorized`, writes a `payments` row and a
wallet ledger entry in Postgres, and calls the gateway to capture. Enabling Kafka
transactions here would be theatre.

```java
package com.orderflow.payments;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

@Component
public class PaymentSettler {

    private final PaymentRepository payments;
    private final WalletLedgerRepository ledger;
    private final ProcessedEventRepository processed;   // Topic 116's dedup table
    private final PaymentGateway gateway;

    public PaymentSettler(PaymentRepository payments,
                          WalletLedgerRepository ledger,
                          ProcessedEventRepository processed,
                          PaymentGateway gateway) {
        this.payments = payments;
        this.ledger = ledger;
        this.processed = processed;
        this.gateway = gateway;
    }

    @KafkaListener(topics = "orderflow.payment-authorized",
                   groupId = "payment-settler",
                   containerFactory = "manualAckContainerFactory")
    public void onAuthorized(PaymentAuthorized event, Acknowledgment ack) {

        // The gateway call happens OUTSIDE the database transaction (Topic 55).
        // It is made idempotent by the gateway's own idempotency key, which we
        // derive deterministically from the event - NOT from a UUID generated here.
        var captureRef = gateway.capture(event.paymentId(), event.amountMinor(),
                                         /* idempotencyKey */ "cap-" + event.paymentId());

        boolean firstTime = recordSettlement(event, captureRef);
        if (!firstTime) {
            // Duplicate delivery. Nothing was written twice. Ack and move on.
        }
        ack.acknowledge();
    }

    /**
     * The dedup insert and the effect are in ONE database transaction.
     * That is what makes redelivery safe. Topic 116 explains the constraint.
     */
    @Transactional
    boolean recordSettlement(PaymentAuthorized event, String captureRef) {
        int inserted = processed.insertIfAbsent("payment-settler", event.eventId());
        if (inserted == 0) return false;                 // already processed
        payments.markCaptured(event.paymentId(), captureRef);
        ledger.appendDebit(event.walletId(), event.amountMinor(), event.eventId());
        return true;
    }
}
```

Note what is *not* here: no `transactional.id`, no `sendOffsetsToTransaction`, no
claim of exactly-once. The design is **at-least-once delivery with an idempotent
effect**, and it is correct under every failure this topic describes.

The offset ack happens after the database commit, so a crash between them redelivers
the event and the dedup insert absorbs it. The gateway call happens before the database
transaction opens, so it does not pin a Hikari connection across a network call
(Topic 55, Topic 109); it is made safe by the gateway's own idempotency key, derived
deterministically from `paymentId` so a retry produces the same key.

### The written contract — put this in the repository

For every topic in `orderflow`, one table row. This is the artefact, and Topic 124's
production-readiness review will ask for it.

| Topic | Producer guarantee | Consumer guarantee | Sink | Idempotency mechanism |
|---|---|---|---|---|
| `orderflow.order-placed` | idempotent producer, `acks=all`, `min.isr=2` | at-least-once | Kafka (`inventory-changed`) | Kafka transaction (`inventory-projector`) |
| `orderflow.order-placed` | as above | at-least-once | Postgres (`notifications_sent`) | unique key on `(consumer_group, event_id)` |
| `orderflow.payment-authorized` | idempotent producer | at-least-once | Postgres + gateway HTTP | dedup table + gateway idempotency key |
| `orderflow.inventory-changed` | **transactional** | at-least-once | Redis projection | last-write-wins by version; naturally idempotent |

The column that matters is the last one. If a row has "none" in it, that row is a
production incident with a date attached.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — "We enabled exactly-once, so the Postgres write and the payment call are covered"

**Wrong:**

```java
@Bean
public ProducerFactory<String, Object> producerFactory() {
    var props = new HashMap<String, Object>();
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "orderflow-tx-");
    return new DefaultKafkaProducerFactory<>(props);
}

@KafkaListener(topics = "orderflow.payment-authorized")
@Transactional                                    // JPA transaction manager
public void settle(PaymentAuthorized e) {
    gateway.capture(e.paymentId(), e.amountMinor());     // HTTP. Not transactional.
    paymentRepo.markCaptured(e.paymentId());             // Postgres. Not in Kafka's tx.
    kafkaTemplate.send("orderflow.payment-settled", e.paymentId());
}
```

**Exact symptom:** a consumer restart, a rebalance, or a `max.poll.interval.ms`
breach produces **two `capture` calls to the gateway for one payment**. The gateway
either double-charges or (if it has its own dedup) returns the same capture reference
twice — and you never notice, so the bug survives until you integrate a gateway that
does not dedup. Observable as: gateway settlement report total exceeding
`SELECT sum(amount_minor) FROM payments WHERE status='CAPTURED'`. That reconciliation
gap is the symptom. There is no exception, no error log, and no failed metric.

A second, sharper symptom for confirmation: set the consumer group's
`max.poll.interval.ms` low enough to force a rebalance mid-batch (Topic 113's drill),
then count rows in a `gateway_calls` audit table. The count exceeds the record count.

**Root cause:** three independent facts.

1. `gateway.capture(...)` is an HTTP request. It left the process. There is no
   mechanism in Kafka, Spring, or the JVM that can un-send it.
2. `paymentRepo.markCaptured(...)` is in a JPA transaction managed by
   `JpaTransactionManager`. Kafka's transaction coordinator does not know it exists.
3. The `@Transactional` annotation on a `@KafkaListener` method covers the database
   only. The Kafka offset commit is performed by the listener container **after** the
   method returns, in a separate operation. There is a window between them either way.

**Fix:** stop claiming exactly-once for this path. Redesign as at-least-once with an
idempotent sink:

```java
@KafkaListener(topics = "orderflow.payment-authorized",
               containerFactory = "manualAckContainerFactory")
public void settle(PaymentAuthorized e, Acknowledgment ack) {
    // Deterministic key derived from the event, so a retry reuses it.
    var ref = gateway.capture(e.paymentId(), e.amountMinor(), "cap-" + e.paymentId());
    applySettlement(e, ref);        // dedup insert + effect in ONE db transaction
    ack.acknowledge();
}
```

and delete the `transactional.id` from this producer factory. Keeping it costs you
throughput and buys you nothing on this path.

---

### Trap 2 — Transactions enabled on the producer, `read_uncommitted` left on the consumer

**Wrong:**

```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: orderflow-tx-
    consumer:
      # isolation-level not set -> read_uncommitted (the default)
      group-id: inventory-projector
```

**Exact symptom:** the inventory projection contains changes from orders that were
never placed. Concretely, `SELECT count(*) FROM inventory_projection` exceeds the count
derivable from `orders`, and the extra rows all correspond to placements that threw and
aborted. During normal operation with no aborts, the system looks perfect. The first
incident that causes aborts causes a *second*, data-corruption incident hours later.

**Root cause:** aborted records are physically present in the partition log. The
broker does not delete them; it writes an ABORT control record. Filtering aborted
records is a **consumer-side** behaviour, and it is off by default. Your producer's
abort was honoured by the coordinator and ignored by the consumer.

**Fix:**

```yaml
spring:
  kafka:
    consumer:
      isolation-level: read_committed
```

**How to prove you fixed it, before deploying:**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.order-placed --from-beginning \
  --isolation-level read_uncommitted --max-messages 200 | wc -l

kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.order-placed --from-beginning \
  --isolation-level read_committed --max-messages 200 | wc -l
```

| What you see | What it means |
|---|---|
| The two counts differ | Aborted records exist in the log and `read_uncommitted` consumers are seeing them. Every consumer group on this topic needs auditing. |
| The two counts are identical | No aborts have occurred in the range read — **not** proof the setting is unnecessary. Induce an abort first, then re-run. |
| `read_committed` hangs and returns nothing | An open transaction is holding the LSO. Run `kafka-transactions.sh find-hanging`. |

---

### Trap 3 — Committing the offset before the work is durable

**Wrong:**

```java
@KafkaListener(topics = "orderflow.order-placed",
               containerFactory = "manualAckContainerFactory")
public void onPlaced(OrderPlaced e, Acknowledgment ack) {
    ack.acknowledge();               // "get the offset out of the way first"
    inventoryService.reserve(e);     // may throw, may crash the pod
}
```

Or the equivalent and far more common form:

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: true       # commits on a timer, independent of your code
```

**Exact symptom:** **silent event loss**. Inventory levels drift upward relative to
orders, with no error, no lag, and no DLQ entry. `SELECT count(*) FROM orders` exceeds
the count of inventory reservations, and the gap grows only during pod restarts and
deployments — which makes it look like a deployment bug rather than a consumer bug.

**Root cause:** an offset commit is a promise that everything before it is done.
Committing first converts the consumer to **at-most-once**. `enable.auto.commit=true`
does the same thing on a timer: the background commit fires between polls regardless of
whether your handler succeeded, so a crash after the auto-commit and before the handler
finishes loses every record in that window.

**Fix:** commit after the effect is durable. With Spring, that means an explicit ack
mode and never acking before the work:

```java
@Bean
ConcurrentKafkaListenerContainerFactory<String, Object> manualAckContainerFactory(
        ConsumerFactory<String, Object> cf) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
    factory.setConsumerFactory(cf);
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    return factory;
}
```

`AckMode.RECORD` and `AckMode.BATCH` (Spring's default) also commit *after* the
listener returns successfully, which is correct. The trap is `enable.auto.commit=true`
and hand-written `ack.acknowledge()` at the top of a method.

**Note the honest trade you just made:** at-least-once means duplicates. You have
moved the problem from "silent loss" to "duplicates you must absorb". That is the right
direction — a duplicate you can dedup, a lost event you cannot reconstruct — but it is
only correct if the sink is idempotent. Topic 116.

---

### Trap 4 — A random or shared `transactional.id`

**Wrong:**

```java
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,
          "orderflow-projector-" + UUID.randomUUID());
```

or, in a Kubernetes Deployment with three replicas:

```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: orderflow-projector   # same value in all three pods
```

**Exact symptom, random ID variant:** nothing visibly wrong for weeks. Then a pod is
killed mid-transaction, and the transaction stays `Ongoing` until
`transaction.timeout.ms`. Every `read_committed` consumer on those partitions stalls
behind the LSO for that duration. `kafka-consumer-groups.sh --describe` shows lag
climbing on a healthy-looking consumer with a live member and a recent heartbeat.
Separately, `kafka-transactions.sh list` accumulates thousands of dead transactional
IDs.

**Exact symptom, shared ID variant:** a `ProducerFencedException` loop. Pod A
initialises, pod B initialises and bumps the epoch, pod A is fenced and restarts, pod A
initialises and bumps the epoch, pod B is fenced. Throughput collapses to near zero and
the pods CrashLoopBackOff in alternation. This one is loud, at least.

**Root cause:** the transactional ID is an **identity**, and the coordinator uses it
for exactly one purpose: deciding which producer instance is the legitimate current
one. A random ID gives every restart a new identity, so fencing never happens and dead
state accumulates. A shared ID gives two live instances the same identity, so they
fence each other forever.

**Fix:** derive it from a stable per-instance ordinal.

```java
// Kubernetes StatefulSet: HOSTNAME is "inventory-projector-0", "-1", "-2".
String ordinal = System.getenv("HOSTNAME").replaceAll(".*-", "");
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "orderflow-projector-" + ordinal);
```

Spring's `transaction-id-prefix` appends a suffix derived from the consumer group,
topic and partition when the producer is used inside a listener container, which makes
it safe *in that context*. Outside it — a standalone `KafkaTemplate` — the prefix alone
is the whole ID and is shared across replicas. **Read Spring's current documentation
for the exact suffixing rule on the `spring-kafka` version your BOM resolves; this
behaviour has changed across major versions and I will not assert a specific version's
rule from memory.**

**Deployment consequence people miss:** because fencing is per transactional ID, a
StatefulSet with N replicas needs N stable IDs, and scaling down leaves IDs with no
owner until `transactional.id.expiration.ms`. Plan the scale-down, or use Kafka Streams
EOS v2, which manages this for you.

---

### Trap 5 — Retrying `send()` in application code and calling it "idempotent"

**Wrong:**

```java
@Retryable(maxAttempts = 3, backoff = @Backoff(delay = 200))   // Topic 111
public void publishOrderPlaced(OrderPlaced e) {
    kafkaTemplate.send("orderflow.order-placed", e.orderId(), e).get(5, SECONDS);
}
```

with `enable.idempotence=true`, and the team's belief that the retries are therefore
safe.

**Exact symptom:** duplicates on the topic that survive `read_committed`, with valid,
increasing sequence numbers and the same `producerId`. A downstream consumer that was
written assuming Kafka's dedup produces two inventory reservations for one order.
Visible as `kafka-console-consumer.sh` showing the same `orderId` twice with different
offsets, and `kafka-dump-log.sh` showing two batches with *different* `baseSequence`
values — that difference is the tell.

**Root cause:** the idempotent producer deduplicates **the client's own internal
retries** of a batch it has already assigned a sequence number to. Your `@Retryable`
sits *above* the client. Each attempt calls `send()` again, which creates a new record,
which gets a new sequence number, which the broker correctly accepts as new data. Worse:
the first attempt may have succeeded and only its ack was lost — `Future.get()` timing
out does not mean the record was not written.

**Fix, and there are three defensible ones:**

1. **Delete the application-level retry.** The producer already retries internally up
   to `delivery.timeout.ms`, and that retry *is* deduplicated. Application retries on
   top of a correctly configured producer are almost always wrong.
2. **If you must retry across producer sessions, make the record idempotent at the
   consumer** with a deterministic event ID — Topic 116.
3. **Move the publish into the outbox** so the relay owns delivery and republication —
   Topic 115. This is the answer for `orderflow`.

> **The general rule to carry forward:** every layer that retries must be either
> deduplicated by the layer below it or idempotent at the sink. Stacking retries
> without checking which layer owns dedup is how a partial outage becomes a data
> corruption incident (Topic 111's amplification, with a correctness twist).

---

## Hands-on proof

Every command here is one **you** run against your Topic 65 docker-compose stack. I
have no Kafka broker and will not print output and call it captured.

### Setup

```bash
cd ~/orderflow
docker compose up -d kafka postgres redis
docker compose exec kafka kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orderflow.lab.delivery --partitions 3 --replication-factor 1 \
  --if-not-exists

# A shell alias so the commands below stay short.
alias kcli='docker compose exec -T kafka'
```

### Proof 1 — the consumer group's committed offsets and lag

```bash
kcli kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group inventory-projector
```

**WHAT TO LOOK FOR:** the `CURRENT-OFFSET`, `LOG-END-OFFSET`, `LAG`, `CONSUMER-ID` and
`HOST` columns, per partition.

| What you see | What it means |
|---|---|
| `LAG` small and stable, `CONSUMER-ID` populated on every partition | Healthy. The group has a live member per partition and is keeping up. |
| `LAG` climbing, `CONSUMER-ID` populated, `CURRENT-OFFSET` not advancing | The consumer is alive but stuck. Either your handler is blocked, or a `read_committed` consumer is blocked at the LSO by an open transaction. Check `kafka-transactions.sh find-hanging` before you look at your code. |
| `CONSUMER-ID` shows `-` on some partitions | No member owns them. A rebalance is in progress, or you have fewer consumers than partitions and the assignor has not yet settled. Topic 113. |
| `LAG` shows `-` | The group has never committed for that partition. Expected for a brand-new group; alarming for an old one. |
| `LAG` sits at a small constant like 1 or 2 on an idle transactional topic | Very likely transaction control records, which occupy offsets and are never delivered. **Do not alert on lag equals zero for a transactional topic.** |
| `LAG` negative | Offsets were reset externally, or the topic was recreated under the group. Investigate before restarting anything. |

### Proof 2 — see aborted records exist, and see them filtered

Produce a transaction and abort it. The simplest reliable way is a tiny Java program;
the console producer cannot begin and abort a transaction.

```java
var p = new Properties();
p.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
p.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "abort-probe-0");
p.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
p.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
p.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

try (var producer = new KafkaProducer<String, String>(p)) {
    producer.initTransactions();

    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orderflow.lab.delivery", "k", "COMMITTED-1"));
    producer.commitTransaction();

    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orderflow.lab.delivery", "k", "ABORTED-1"));
    producer.abortTransaction();

    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orderflow.lab.delivery", "k", "COMMITTED-2"));
    producer.commitTransaction();
}
```

```bash
kcli kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.lab.delivery --from-beginning --timeout-ms 5000 \
  --isolation-level read_uncommitted

kcli kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.lab.delivery --from-beginning --timeout-ms 5000 \
  --isolation-level read_committed
```

| What you see | What it means |
|---|---|
| `read_uncommitted` prints `ABORTED-1`; `read_committed` does not | **The core demonstration.** The record is in the log. Filtering is a consumer decision, and the default is not to filter. |
| Both print `ABORTED-1` | Your abort did not happen, or you are on a partition whose ABORT marker has not been written yet. Re-run with a `Thread.sleep(2000)` before consuming. |
| `read_committed` prints nothing at all and the command times out | An open transaction is holding the LSO at or before your first record. `kafka-transactions.sh find-hanging --topic orderflow.lab.delivery`. |
| Offsets are non-contiguous in `--property print.offset=true` output | Control records occupy the gaps. This is normal and is why offset arithmetic is not record arithmetic. |

### Proof 3 — read the batch headers

```bash
docker compose exec kafka bash -c \
 'kafka-dump-log.sh --files /var/lib/kafka/data/orderflow.lab.delivery-0/*.log \
   --print-data-log' | head -60
```

**WHAT TO LOOK FOR:** `producerId`, `producerEpoch`, `baseSequence`, `isTransactional`,
`isControl`, and the batch `baseOffset`/`lastOffset` boundaries.

Use the reading table from **Machine-level reality** above. The single most instructive
line is a batch with `isControl: true` — that is a transaction marker, and seeing one
makes the two-phase commit concrete in a way no diagram does.

> The data directory path inside the container depends on the image you used in Topic
> 65. Find it with `docker compose exec kafka bash -c 'ls -d /var/lib/kafka/data*
> /kafka* 2>/dev/null'` or by reading `log.dirs` from the broker config. I am not
> asserting a path for your image.

### Proof 4 — a hung transaction stalls a `read_committed` consumer

Start the abort-probe program, call `beginTransaction()` and `send()`, then **do not
commit** — sleep for two minutes. In another terminal:

```bash
kcli kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group inventory-projector
kcli kafka-transactions.sh --bootstrap-server localhost:9092 list
```

| What you see | What it means |
|---|---|
| Lag climbing on the affected partition while the consumer is live and heartbeating | The consumer is blocked at the LSO. This is the failure mode that looks like a consumer bug and is a producer bug. |
| `kafka-transactions.sh list` shows state `Ongoing` for `abort-probe-0` | Confirmed. The open transaction is the cause. |
| Lag clears by itself after roughly `transaction.timeout.ms` | The coordinator aborted it on timeout. This is why that timeout is a latency budget, not a safety net. |
| A consumer with `read_uncommitted` is unaffected | Correct, and it is also reading data that may never be committed. This is not a workaround. |

### Proof 5 — confirm the effective client configuration, not the intended one

```properties
logging.level.org.apache.kafka.clients.producer.ProducerConfig=INFO
logging.level.org.apache.kafka.clients.consumer.ConsumerConfig=INFO
```

The Kafka clients log their **resolved** configuration at startup, including values you
did not set and warnings for keys you supplied that they do not recognise.

| What you see | What it means |
|---|---|
| `The configuration 'isolation.level' was supplied but isn't a known config` | You put a consumer property in a producer's map, or misspelled a key. Your setting is doing nothing. This warning is the single highest-value line in Kafka client logging. |
| `enable.idempotence = true` and `acks = all` in the producer's resolved config | Confirmed on. Do not trust the YAML; trust this line. |
| `isolation.level = read_uncommitted` when you thought you set it | Spring's property is `spring.kafka.consumer.isolation-level`. A raw `properties:` map entry under the wrong prefix is silently ignored. |

---

## Failure drill

**Mandatory.** Produce the failure yourself and write down what you saw before reading
the fix. The point is not the knowledge; it is the memory of the second charge.

### The scenario

A consumer with exactly-once fully enabled on the Kafka side performs a Postgres write
and an HTTP call. You will kill it mid-batch and show that the Kafka side is perfectly
consistent while the money is wrong.

### Setup

Add a fake gateway that records every call:

```sql
create table gateway_calls (
  id           bigserial primary key,
  payment_id   text        not null,
  amount_minor bigint      not null,
  called_at    timestamptz not null default now()
);
create table settlements (
  payment_id   text primary key,
  amount_minor bigint not null
);
```

```java
package com.orderflow.lab.eos;

@Component
public class EosDrillConsumer {

    private final JdbcTemplate jdbc;
    private final KafkaTemplate<String, String> template;   // transactional producer

    @KafkaListener(topics = "orderflow.lab.authorized",
                   groupId = "eos-drill",
                   containerFactory = "eosContainerFactory")   // transaction-id-prefix set,
                                                               // isolation.level=read_committed
    public void handle(ConsumerRecord<String, String> record) {

        String paymentId = record.key();
        long amount = Long.parseLong(record.value());

        // 1. The "HTTP call". Not in any transaction.
        jdbc.update("insert into gateway_calls(payment_id, amount_minor) values (?,?)",
                    paymentId, amount);

        // 2. Slow enough that you can land the kill in the window.
        sleepQuietly(3000);

        // 3. The Postgres effect. Also not in any Kafka transaction.
        jdbc.update("insert into settlements(payment_id, amount_minor) values (?,?) "
                  + "on conflict (payment_id) do nothing", paymentId, amount);

        // 4. The Kafka output. THIS one is in the Kafka transaction.
        template.send("orderflow.lab.settled", paymentId, String.valueOf(amount));
    }
}
```

Configure the container factory for full EOS:

```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: eos-drill-tx-
      properties:
        enable.idempotence: true
    consumer:
      isolation-level: read_committed
      enable-auto-commit: false
      group-id: eos-drill
```

### Commands

```bash
# Seed exactly five authorizations.
for i in 1 2 3 4 5; do
  echo "PAY-$i:1999" | kcli kafka-console-producer.sh \
    --bootstrap-server localhost:9092 --topic orderflow.lab.authorized \
    --property parse.key=true --property key.separator=:
done

# Start the service, then kill it hard while record 3 is in its 3-second sleep.
./mvnw spring-boot:run &
SERVICE_PID=$!
sleep 9            # roughly mid-record-3; adjust from your own log timestamps
kill -9 $SERVICE_PID

# Look at the damage BEFORE restarting.
psql -h localhost -U orderflow -d orderflow -c \
  "select payment_id, count(*) from gateway_calls group by 1 order by 1;"
psql -h localhost -U orderflow -d orderflow -c "select * from settlements order by 1;"

kcli kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group eos-drill

# Now restart and let it finish.
./mvnw spring-boot:run &
sleep 30
psql -h localhost -U orderflow -d orderflow -c \
  "select payment_id, count(*) from gateway_calls group by 1 order by 1;"
kcli kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orderflow.lab.settled --from-beginning --timeout-ms 5000 \
  --isolation-level read_committed --property print.key=true
```

### What to capture

Write these down before reading on:

1. `gateway_calls` grouped counts **after the kill, before the restart**.
2. `gateway_calls` grouped counts **after the restart completes**.
3. The `settlements` rows after the restart.
4. The records on `orderflow.lab.settled` under `read_committed`.
5. The consumer group's committed offset after the kill.

### How to read it

| What you see | What it means |
|---|---|
| After restart, one `payment_id` has **two** rows in `gateway_calls` | **The drill has fired.** The record was redelivered because its offset was never committed, and the "HTTP call" ran a second time. Kafka's exactly-once did not and cannot prevent this. |
| `settlements` has exactly one row per payment | The `on conflict do nothing` made *that* effect idempotent. Note that this is your code, not Kafka. This contrast is the entire lesson: same delivery, one effect duplicated and one not. |
| `orderflow.lab.settled` under `read_committed` has exactly one record per payment | The Kafka side is perfect. The transaction that was in flight at the kill was aborted by the coordinator on timeout, and its output was filtered out. **Kafka kept its promise exactly. The promise was just smaller than you thought.** |
| The committed offset after the kill points at record 3, not record 4 | Correct and expected. The transaction that would have advanced it was aborted. |
| No duplicate in `gateway_calls` at all | Your kill landed outside the window. Increase the sleep to 8 seconds and retry. **Absence of the bug in one run is not absence of the bug.** |
| `orderflow.lab.settled` under `read_uncommitted` has an extra record | The aborted output. Run the same consumer both ways and put the two outputs side by side — this is the clearest single artefact from the whole drill. |

### Now fix it, and notice what the fix is not

The fix is **not** a Kafka setting. There is no setting. The fix is at the sink:

```java
// Deterministic idempotency key, derived from the event - not generated per attempt.
jdbc.update("insert into gateway_calls(payment_id, amount_minor, idem_key) "
          + "values (?,?,?) on conflict (idem_key) do nothing",
            paymentId, amount, "cap-" + paymentId);
```

Re-run the drill. The duplicate disappears — and it disappears because **you** made the
effect idempotent, with a database constraint, in Postgres. Write that sentence down.
Topic 116 turns it into a general mechanism.

---

## Measurement

### The standing rule first

> **A naive `System.nanoTime()` measurement of anything in this topic is wrong.** JIT
> warm-up, dead-code elimination, on-stack replacement and coordinated omission all
> apply, and on top of them Kafka adds batching (`linger.ms`), broker-side page-cache
> effects, and a first-call cost that includes metadata fetch and connection setup.
> Timing "one send" tells you nothing. Use JMH for in-JVM cost (Topic 77) and the k6
> open-model generator from Topic 65 for end-to-end latency. Any number you get from a
> loop and a stopwatch belongs in the bin.

### The four things to measure, and why each one

**1. Consumer lag — per partition, not aggregated.**

```bash
kcli kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group inventory-projector
```

Aggregate lag hides the failure that matters: one partition stuck at the LSO while
eleven others are fine. Alert on **max lag across partitions**, not sum, and alert on
**lag derivative** (is it growing?) rather than an absolute threshold, because absolute
thresholds are meaningless across the traffic profile of a day.

For a scraped version, `kafka_consumergroup_lag` from a lag exporter, or Micrometer's
`kafka.consumer.fetch.manager.records.lag.max` from the client itself (Topic 118 wires
this; it comes free from `MicrometerConsumerListener` when a `MeterRegistry` is
present). Client-side lag is cheaper but blind while the consumer is down — which is
exactly when you need it. Run both.

**2. Duplicate-processing count.**

This is the number that makes the topic real, and almost nobody instruments it.

```java
int inserted = processed.insertIfAbsent(group, eventId);
if (inserted == 0) {
    duplicatesCounter.increment();     // Micrometer counter, Topic 118
    return;
}
```

Tag it with the consumer group and the topic — **never** with the event ID (Topic 118's
cardinality trap). A steady low rate is healthy and expected under at-least-once. A
spike correlates with rebalances and restarts and is your evidence that the dedup layer
is load-bearing rather than theoretical.

**3. Transaction health, if you use transactions.**

Client metrics exposed via Micrometer from the Kafka producer:
`kafka.producer.txn.commit.time.avg`, `kafka.producer.txn.abort.time.avg`, and the
record-error rate. Plus `kafka-transactions.sh list | grep -c Ongoing` on a schedule.

**The single most valuable alert in this topic:** the age of the oldest `Ongoing`
transaction. If it exceeds a fraction of `transaction.timeout.ms`, your
`read_committed` consumers are about to appear broken. Nothing else surfaces this.

**4. End-to-end latency against the Topic 65 baseline.**

Enabling transactions is not free, and the honest way to state the cost is a delta
against a recorded baseline, per endpoint:

```bash
k6 run --out json=results-eos-off.json load/orderflow-mix.js
# enable transactions on the projector, redeploy
k6 run --out json=results-eos-on.json load/orderflow-mix.js
```

Compare p50 / p95 / p99 / p999 per endpoint against `/docs/java/baselines/`. The
Topic 65 gate rule applies: if the un-changed baseline no longer reproduces within
±10%, your measurement environment has drifted and the comparison is worthless — fix
that before drawing a conclusion.

**What to expect qualitatively** (and then verify, rather than trusting me): the
producer path gains a coordinator round trip per transaction, amortised across the poll
batch, so throughput cost falls as batch size rises. The consumer path gains LSO
blocking, which shows up as **latency variance**, not mean latency. So look at p999 and
at the standard deviation, not the average. If your dashboard only has an average, you
cannot see this effect at all — which is Topic 118's argument, arriving early.

### What "proof" looks like at the end of this topic

A committed table in the repository, one row per topic, with the guarantee and the
idempotency mechanism named — plus a `duplicates_absorbed` counter on every consumer
and a max-lag-per-partition alert. That artefact is what Topic 124 will review.

---

## Practice exercises

### 1 — Easy: the guarantee inventory

Without writing code, produce a table for the five `orderflow` topics you have from
Topic 113 with these columns: topic, producer config (`acks`, `enable.idempotence`,
`transactional.id` present?), every consumer group, each group's `isolation.level` and
ack mode, the sink type (Kafka / Postgres / HTTP / Redis), and the idempotency
mechanism at that sink.

Then answer three questions in one sentence each:

1. Which rows have "none" in the idempotency column, and what is the specific business
   damage a duplicate causes there?
2. Which rows would be *made worse* by enabling Kafka transactions, and why?
3. For which single row would Kafka transactions be the complete and correct answer?

Verify the config columns with the resolved-config log lines from Proof 5, not from
your YAML. The gap between what you believed and what the log says is the useful part.

### 2 — Medium: the audit (combines Topics 40, 54, 55, 90, 92, 109, 111, 113)

This consumer has **seven** defects. Four are from this topic. Three are from earlier
topics. For each: name the topic, state the **observable** symptom in production, and
write the fix.

```java
package com.orderflow.notifications;

@Service
public final class NotificationConsumer {

    @Autowired private NotificationRepository repo;
    @Autowired private EmailGateway email;
    @Autowired private KafkaTemplate<String, String> template;

    private final ExecutorService pool = Executors.newCachedThreadPool();
    private final Set<String> seen = new HashSet<>();

    @KafkaListener(topics = "orderflow.order-placed", groupId = "notifications")
    @Transactional
    public void onPlaced(ConsumerRecord<String, OrderPlaced> record,
                         Acknowledgment ack) throws MessagingException {
        ack.acknowledge();

        pool.submit(() -> {
            String id = record.value().eventId();
            if (seen.contains(id)) return;
            seen.add(id);

            email.send(record.value().customerEmail(), renderBody(record.value()));
            repo.markNotified(id);
            template.send("orderflow.notification-sent", id);
        });
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    private void markFailed(String id) { repo.markFailed(id); }
}
```

Hints, in the order to think about them. One defect makes the whole `@Transactional`
annotation inert regardless of everything else — find that one first because it hides
two others. One converts the consumer to at-most-once. One is a Topic 92 check-then-act
race that also happens to be a Topic 79 unbounded-retention leak. One is a Topic 90
unbounded queue. One is a Topic 54 rollback-rule bug involving a checked exception. One
is a Topic 55 connection-lifetime bug. One is specific to this topic and concerns what
`ack.acknowledge()` promises.

### 3 — Hard: production simulation on `orderflow` under load

**Part A — establish the baseline.** Run the Topic 65 k6 mix for ten minutes with the
current configuration. Record per-endpoint p50/p95/p99/p999, throughput, error rate,
and per-partition consumer lag for both groups. Confirm you are within ±10% of the
committed baseline before continuing. If you are not, stop and fix the environment.

**Part B — measure the duplicate rate you already have.** Add a `duplicates_absorbed`
counter to the `payment-settler` dedup insert. Run the load again while performing a
rolling restart of the consumer deployment (three pods, one at a time, 30 seconds
apart). Record the duplicate count and correlate it with rebalance events from Topic
113's instrumentation. Write down the number of duplicates per rebalance.

**Part C — the false fix.** Enable full Kafka EOS on the `payment-settler`:
`transactional.id` per instance, `read_committed`, `sendOffsetsToTransaction`. Re-run
Part B exactly. Record the new duplicate count and the new latency percentiles.

State, before you look: what do you predict happens to (a) the duplicate count, (b)
p99, (c) p999, (d) throughput? Then compare with what you measured, and explain every
prediction you got wrong.

**Part D — the real fix, and its cost.** Revert EOS. Instead, make the gateway call
idempotent with a deterministic key and move the dedup insert into the same transaction
as the effect. Re-run Part B. Record duplicate count, latency, and the write amplification
on the `processed_events` table (`pg_stat_user_tables` `n_tup_ins` and `n_dead_tup`).

**Part E — the cost nobody budgets.** The `processed_events` table now grows by one row
per event forever. At the baseline's placement rate, compute the row count after 90
days and the index size. Design and implement a retention policy. State what breaks if
the retention window is shorter than the maximum possible redelivery delay, and how you
would determine that maximum from the Kafka configuration (`retention.ms`, and the
worst-case time a consumer group can be down and still resume).

**Part F — argue against yourself.** You recommended at-least-once plus an idempotent
sink. Make the strongest possible case that the `inventory-projector` should use full
Kafka EOS instead. Then say what would have to be true about the downstream consumers
for that case to win, and how you would verify it.

---

## Interview questions

### Q1 — "We enabled exactly-once semantics on our Kafka pipeline. Walk me through what that actually guarantees."

**Mid-level answer:** "It means each message is processed exactly once — no duplicates
and no loss. You set `enable.idempotence=true` and use transactions, and Kafka handles
deduplication for you."

**Senior answer:** "It guarantees exactly-once for records *inside* Kafka, and I would
want to be precise about the boundary because that is where the design lives.

Concretely it is three things. One, the idempotent producer: the broker assigns a
producer ID and tracks a monotonic sequence number per partition, so a *client retry*
of a batch it has already sequenced is dropped as a duplicate. Two, a transactional
read-process-write: the producer writes output records and the input consumer offsets
under one transactional ID, and a transaction coordinator commits both by writing
control records into every involved partition. Three — and this is the one teams
forget — consumers must set `isolation.level=read_committed`, because the default is
`read_uncommitted` and aborted records are physically in the log.

What it does not cover is anything outside Kafka. If my consumer writes a Postgres row
or calls a payment gateway, those effects are not in the transaction. The coordinator
can write an ABORT marker; it cannot issue a Postgres ROLLBACK and it cannot un-send an
HTTP request. So on a redelivery — a rebalance, a restart, a `max.poll.interval.ms`
breach — those effects happen again.

So my default design is at-least-once delivery with an idempotent sink: a unique
constraint on an event ID inserted in the same database transaction as the effect, and
a deterministic idempotency key on any outbound HTTP call. I reserve Kafka transactions
for genuinely Kafka-to-Kafka stages, where they are the complete answer and where
Kafka Streams already packages them.

The way I would say it in a design review: exactly-once is a property of the effect,
not of the transport."

**What separates them:** the mid answer states a marketing claim. The senior answer
names the three mechanisms separately, knows `read_committed` is not the default,
locates the boundary precisely, and — the real signal — states the *design consequence*:
at-least-once plus idempotent sinks as the default, with transactions as a special case.
Someone who has actually operated this also mentions the transactional-ID identity
problem unprompted.

**Follow-up:** "You said `read_committed` filters aborted records. What does it cost
you?" — Consumers cannot read past the Last Stable Offset, so an open transaction
blocks that partition entirely. A hung producer looks exactly like a dead consumer: lag
climbing with a live, heartbeating member. The first thing I check is
`kafka-transactions.sh find-hanging`, not my consumer code.

---

### Q2 — "Your consumer processes a message, writes to Postgres, and then commits the offset. The pod is killed between the write and the commit. What happens, and is that acceptable?"

**Mid-level answer:** "The message gets redelivered because the offset wasn't
committed, so it's processed twice. You'd add a check to see if you've already
processed that message ID."

**Senior answer:** "It is redelivered, which is correct and by design — that ordering
is what makes the consumer at-least-once rather than at-most-once. If I had committed
first I would have *lost* the message on that crash, which is strictly worse: a
duplicate I can absorb, a lost order I have to reconstruct from somewhere.

Whether it is acceptable depends entirely on the effect. If the write is
`INSERT ... ON CONFLICT DO NOTHING` on a natural key, redelivery is a no-op and I am
done. If it is `UPDATE wallets SET balance = balance - 1999`, redelivery double-debits
a customer and it is a money bug.

The general fix is to make the dedup and the effect one atomic unit. I insert a row
into a `processed_events` table keyed on `(consumer_group, event_id)` with a unique
constraint, in the *same* transaction as the effect. If the insert violates the
constraint, this is a redelivery and the transaction rolls back harmlessly. What I
would not do is `SELECT` first and then `INSERT` — that is check-then-act, it races
under concurrency, and two consumers processing the same redelivered record both pass
the check. The unique constraint makes the database the arbiter, which is the only
component that can arbitrate.

Two Java-specific details I would raise in review. First, a constraint violation
surfaces as `DataIntegrityViolationException`, and once Hibernate has hit a constraint
violation at flush the persistence context is in an unrecoverable state — I cannot
catch it and continue in the same transaction, so the dedup insert has to be structured
so the violation ends the transaction. Second, the redelivery is a *range*, not a
message: Kafka commits one offset per partition, so a crash re-delivers everything from
the last committed offset. My handler has to be idempotent for the whole poll batch."

**What separates them:** the mid answer names dedup. The senior answer explains why
this ordering is the *right* one, distinguishes idempotent from non-idempotent effects,
rejects check-then-act by name, knows the constraint is what arbitrates, and knows the
Java-level consequence of the violation. The "redelivery is a range" point is what
signals real operational experience.

**Follow-up:** "How long do you keep rows in `processed_events`?" — Longer than the
maximum possible redelivery delay, which is bounded by the topic's `retention.ms` and
by how long a consumer group can be down and still resume from a valid offset. Shorter
than that and an old redelivery slips through after the dedup row is gone. I would also
partition or time-bucket the table, because at the baseline's rate it grows without
bound and a full-table unique index eventually dominates the write cost.

---

### Q3 — "What is a `transactional.id` and what happens if two instances share one?"

**Mid-level answer:** "It identifies the producer for transactions. Each instance
should have its own, otherwise there could be conflicts."

**Senior answer:** "It is an *identity* the transaction coordinator uses to decide
which producer instance is the legitimate current one. Two things hang off it.

First, it gives a stable producer ID across restarts. A plain idempotent producer gets
a fresh PID on every restart, so its dedup state is per session; a transactional ID
makes the PID recoverable.

Second, it enables zombie fencing. When an instance calls `initTransactions()`, the
coordinator bumps the epoch for that ID. Any older instance still holding the previous
epoch gets `ProducerFencedException` on its next operation. That is a fatal error — the
correct response is to close the producer and exit, not to retry.

If two live instances share one ID, they fence each other in a loop: A initialises, B
initialises and bumps the epoch, A is fenced and restarts, A bumps the epoch, B is
fenced. Throughput goes to zero and you get alternating crash loops. It is loud, at
least.

The subtler failure is the opposite mistake — deriving the ID from a UUID at startup.
Then fencing never happens, and every killed pod leaves an `Ongoing` transaction that
blocks `read_committed` consumers at the LSO until `transaction.timeout.ms` expires.
Lag climbs on a consumer that is alive and heartbeating, which sends everyone
debugging the wrong component. And coordinator state accumulates until
`transactional.id.expiration.ms`.

On Kubernetes I derive it from a StatefulSet ordinal so it is stable per instance and
unique across instances. And I plan the scale-down, because a removed replica leaves an
orphaned ID."

**What separates them:** the mid answer knows the words. The senior answer knows it is
an identity used for fencing, knows the exception is fatal rather than retryable, and —
crucially — names the *random ID* failure, which is the one that actually happens and
the one that presents as somebody else's bug.

**Follow-up:** "You said `ProducerFencedException` is fatal. Why not just call
`initTransactions()` again and continue?" — Because being fenced means another process
has legitimately claimed this identity, probably a replacement started by the
orchestrator. Re-claiming it starts a fight. The only safe response is to exit and let
the orchestration layer decide who should be alive.

---

### Q4 — "Consumer lag on one partition is climbing. The consumer is alive and heartbeating and its CPU is idle. Where do you look?"

**Mid-level answer:** "Check if the consumer is stuck on a slow message, look at
`max.poll.interval.ms`, maybe increase the number of consumers or partitions."

**Senior answer:** "Idle CPU with climbing lag on *one* partition rules out most of the
obvious causes, so I would work down a short list.

First — and this is specific to `read_committed` consumers — an open transaction
holding the Last Stable Offset. The consumer physically cannot read past it. I check
`kafka-transactions.sh find-hanging --topic <t>` and `list`, looking for a state of
`Ongoing` with an old start timestamp. That is a *producer* problem presenting as a
consumer symptom, and it is the one people never check.

Second, partition skew: the key distribution. If the key is `productId` and there is a
flash-sale product, one partition gets a disproportionate share. Idle CPU argues
against it here, but I would confirm with per-partition record rates.

Third, a blocked handler — a slow downstream, a lock, a connection-pool wait. Idle CPU
fits a blocking wait perfectly, so I would take a thread dump with `jcmd <pid>
Thread.print` and look at where the listener threads are parked. If they are in
`HikariPool.getConnection`, that is Topic 109 and the Kafka layer is innocent.

Fourth, the partition has no owner. `kafka-consumer-groups.sh --describe` showing `-`
in `CONSUMER-ID` means a rebalance is in progress or stuck — Topic 113's rebalance
storm.

What I would *not* do first is add consumers. Beyond the partition count that does
nothing, and if the cause is an LSO block or a stuck rebalance, adding a member makes
it worse by triggering another rebalance."

**What separates them:** the mid answer reaches for scaling. The senior answer has a
ranked differential diagnosis, puts the transaction-coordinator cause first because it
is the one that matches the specific symptom, knows which tool answers each hypothesis,
and explicitly rejects the intervention that makes it worse.

**Follow-up:** "How would you alert on the LSO cause specifically?" — Age of the oldest
`Ongoing` transaction, alerting well below `transaction.timeout.ms`. Consumer lag alone
cannot distinguish it from a slow consumer, and by the time lag alerts you are already
in the incident.

---

### Q5 — "Design the delivery guarantees for order placement in a system like this. Talk me through the trade-offs."

**Mid-level answer:** "Use Kafka with `acks=all` and replication factor 3, enable
idempotence on the producer, and use consumer groups so we can scale. Add retries for
reliability."

**Senior answer:** "I would design it per *sink*, because there is no single answer for
the pipeline.

Durability first: `acks=all`, `replication.factor=3`, `min.insync.replicas=2`. Note
that `acks=all` with `min.insync.replicas=1` is not actually durable against one broker
loss, which is a configuration people get wrong because the two settings are read
independently.

Publishing: the order write and the event publish are a dual write. Publishing after
the commit leaves a window where a crash loses the event silently, and
`@TransactionalEventListener(AFTER_COMMIT)` has exactly the same window — it runs after
the commit, so a crash between them loses it identically. So the event goes into an
outbox table in the same transaction as the order, and a relay publishes it. The relay
is at-least-once by construction, because it can crash between publishing and marking
the row sent.

Consumption: at-least-once everywhere, with the idempotency mechanism named per sink.
Kafka-to-Kafka stages can use a Kafka transaction and I would consider it — it is the
one place where the guarantee is complete. Postgres sinks get a unique constraint on
`(consumer_group, event_id)` inserted in the same transaction as the effect. HTTP sinks
get a deterministic idempotency key derived from the event, so a retry reuses it.

Ordering: per-order ordering comes from keying by order ID, which pins it to a
partition. That is the only ordering I get, and I would write down that cross-order
ordering does not exist so nobody builds on it later.

What I would explicitly *not* do is turn on exactly-once and stop thinking. It covers
the Kafka boundary, it costs throughput and adds LSO blocking, and it leaves the
payment API call and the wallet debit exactly as exposed as before.

The artefact I would produce is a table: one row per topic, with producer guarantee,
consumer guarantee, sink type, and idempotency mechanism. Any row with 'none' in the
last column is an incident with a date on it."

**What separates them:** the mid answer configures Kafka. The senior answer designs per
sink, catches the `min.insync.replicas` interaction, raises the dual-write problem
unprompted, names the `AFTER_COMMIT` equivalence, states the ordering guarantee's exact
scope, and produces an artefact. The explicit "what I would not do" is the strongest
signal in the answer.

**Follow-up:** "Retries — where would you put them and where would you not?" — Inside
the producer client only, because those are the ones the idempotent producer
deduplicates. An application-level `@Retryable` around `send()` creates genuinely new
records with new sequence numbers, so it is not deduplicated at all, and it is also
Topic 111's amplification: three retries is a 4x load multiplier on a dependency that
is already failing.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The idempotent producer's dedup window is five sequence numbers deep, and
   `max.in.flight.requests.per.connection` is forced to at most five. Derive the second
   fact from the first, without recalling it from the text.

2. A transaction commit writes a control record into *every* involved partition. From
   that one fact, derive why offsets on a transactional topic are non-contiguous, and
   derive one consequence for anything that computes lag by subtracting offsets.

3. `sendOffsetsToTransaction` writes to `__consumer_offsets`, which is an ordinary Kafka
   topic. Explain why that single design choice is what makes read-process-write atomic
   at all — and name what it would take to include a Postgres write the same way.

4. Suppose Kafka allowed you to register a callback that the transaction coordinator
   invoked on commit, so you could do your Postgres write there. Explain precisely why
   that still would not give you exactly-once, and name the failure that remains.

5. A `read_committed` consumer cannot read past the LSO. Argue that this is a design
   flaw. Then argue it is the only possible design. Which argument is stronger, and does
   your answer change if the transaction timeout is one second rather than fifteen
   minutes?

6. You are told a system achieves end-to-end exactly-once between Kafka and Postgres.
   Name the two things that must be true for that claim to be honest, and design the
   single query you would run to falsify it.

7. Your consumer is at-least-once with an idempotent sink. A colleague proposes removing
   the dedup table because "we have never actually seen a duplicate in production". What
   is the strongest version of their argument, what is the counter-argument, and what
   measurement would settle it?

---

## Quick reference card

### The three mechanisms

| Mechanism | Config | Scope of the guarantee |
|---|---|---|
| Idempotent producer | `enable.idempotence=true` (default since 3.0) | Client retries of one batch, one producer session, per partition |
| Transactions | `transactional.id=<stable, per-instance>` | Output records + consumer offsets, atomically, inside Kafka |
| `read_committed` | `isolation.level=read_committed` (**not** the default) | Filters aborted records at the consumer |

### Forced settings when idempotence is on

```
acks                                    = all
max.in.flight.requests.per.connection  <= 5
retries                                 > 0
```

The client fails at startup if you contradict these. That is a feature.

### The transactional producer loop

```java
producer.initTransactions();                       // once, at startup; fences older epochs
producer.beginTransaction();
producer.send(...);
producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
producer.commitTransaction();
// ProducerFencedException / OutOfOrderSequenceException -> FATAL, close and exit
// other KafkaException -> abortTransaction(), then seek() back to committed offsets
```

### Diagnostic commands

```bash
kafka-consumer-groups.sh --bootstrap-server <b> --describe --group <g>
kafka-consumer-groups.sh --bootstrap-server <b> --describe --group <g> --members --verbose
kafka-console-consumer.sh --bootstrap-server <b> --topic <t> --isolation-level read_committed
kafka-transactions.sh --bootstrap-server <b> list
kafka-transactions.sh --bootstrap-server <b> describe --transactional-id <id>
kafka-transactions.sh --bootstrap-server <b> find-hanging --topic <t>
kafka-dump-log.sh --files <segment>.log --print-data-log     # producerId, epoch, sequence
kafka-configs.sh --bootstrap-server <b> --describe --entity-type brokers --entity-default
```

```properties
logging.level.org.apache.kafka.clients.producer.ProducerConfig=INFO
logging.level.org.apache.kafka.clients.consumer.ConsumerConfig=INFO
```

### Symptom to cause

| Symptom | First thing to check |
|---|---|
| Lag climbing, consumer alive and idle | Open transaction blocking the LSO — `find-hanging` |
| Aborted records reaching downstream | Consumer `isolation.level` left at `read_uncommitted` |
| Duplicates with the same PID, different sequence | Application-level retry above the producer client |
| Duplicates with no PID at all | `enable.idempotence` off, or an old client |
| Alternating pod crash loop with `ProducerFencedException` | Two instances sharing a `transactional.id` |
| Events silently missing after restarts only | Offset committed before the effect (`enable.auto.commit=true`) |
| Lag stuck at a small constant on an idle topic | Transaction control records occupying offsets — expected |

### Gotchas checklist

- [ ] `isolation.level=read_committed` is **not** the default. Set it explicitly.
- [ ] `acks=all` with `min.insync.replicas=1` is not durable. Set both.
- [ ] `enable.auto.commit=true` is at-most-once for your business logic.
- [ ] Never `ack.acknowledge()` before the effect is durable.
- [ ] `transactional.id` must be stable per instance and unique across instances.
- [ ] `ProducerFencedException` is fatal. Close and exit.
- [ ] After `abortTransaction()`, `seek()` back or you silently skip the batch.
- [ ] Application-level retries around `send()` are not deduplicated.
- [ ] Redelivery is a range, not a message. Be idempotent for the whole poll batch.
- [ ] The Postgres write and the HTTP call are never in the Kafka transaction. Ever.

---

## When would I use this at work?

**1. Killing an "exactly-once" claim in a design review, with a specific failure.**

An architecture document says the pipeline is exactly-once end to end. You ask one
question: "the consumer writes to Postgres — which transaction is that write in?" There
is no answer, because there is no mechanism. You then propose the concrete alternative:
at-least-once with a unique constraint on `(consumer_group, event_id)` inserted in the
same transaction as the effect, plus a deterministic idempotency key on the gateway
call. That reframes the meeting from a belief to a design in ninety seconds, and it is
the highest-value thing in this document.

**2. Diagnosing lag that is not a consumer problem.**

Pager fires: consumer lag climbing on `orderflow.payment-authorized`. The consumer is
healthy, CPU idle, heartbeating. Everyone starts reading consumer code. You run
`kafka-transactions.sh find-hanging`, find an `Ongoing` transaction from a pod that was
killed eleven minutes ago, and now the question is why the transactional ID is not
stable rather than why the consumer is slow. Ten minutes instead of two hours, and the
fix is a deployment change rather than a code change.

**3. Costing a proposal honestly before agreeing to it.**

A team proposes enabling EOS across all consumers "for correctness". You can state
exactly what it buys (nothing for the three consumers whose sinks are Postgres and
HTTP), exactly what it costs (a coordinator round trip per transaction, control records
in every partition, LSO blocking that shows up as p999 variance, and a
transactional-ID lifecycle to manage across scale events), and exactly which one
consumer it genuinely helps. Then you propose spending the same effort on dedup tables
instead. Being able to argue that with specifics rather than preference is the
difference between a senior engineer and a strong one.

---

## Connected topics

**Prerequisites:**

- **54 — `@Transactional` semantics**: the database transaction is the *other* half of
  every trap here, and the checked-exception rollback default (a checked exception
  commits) is a live hazard inside a Kafka listener.
- **55 — Isolation and the connection pool**: a transaction pins a Hikari connection
  for its lifetime, which is why the gateway call in Example 2 sits *outside* the
  database transaction.
- **65 — The load baseline**: every latency claim about enabling transactions is a
  delta against `/docs/java/baselines/`, or it is a guess.
- **90–92 — Executors and check-then-act**: fanning a poll batch out to a pool without
  tracking the lowest incomplete offset is how "parallel consumers" lose records, and
  the in-memory `seen` set is Topic 92's race with a leak attached.
- **109 — HikariCP**: consumer threads blocked in `getConnection` present as Kafka lag.
  Take a thread dump before blaming the broker.
- **111 — Resilience4j**: retries that are not deduplicated by the layer below are
  amplification, not reliability.
- **113 — Kafka consumer groups**: partitions, rebalancing, `max.poll.interval.ms`. Every
  redelivery in this document arrives through one of those mechanisms.

**This unlocks:**

- **115 — The transactional outbox**: the producer-side half of the problem this topic
  leaves open. The publish-after-commit window, and the `AFTER_COMMIT` listener that
  has the same window.
- **116 — Idempotency**: the sink-side half. The unique constraint that actually
  delivers exactly-once *effects*, and the concurrent-duplicate race that a naive
  dedup check loses.
- **117 — Sagas**: what to do when the effect cannot be made idempotent and must instead
  be compensated.
- **118 — Micrometer**: `duplicates_absorbed`, per-partition max lag, transaction commit
  timers — and the cardinality rule that says never to tag any of them with an event ID.
- **119 — Tracing**: propagating trace context through Kafka record headers, so a
  redelivery is visible as a second span on the same trace rather than an unrelated
  request.
- **120 — MDC and structured logging**: putting the event ID and the offset in every log
  line so a duplicate is greppable.
- **123 — Graceful shutdown**: draining a consumer so a rolling deploy does not
  manufacture the redeliveries this document is about.
- **129 — Capacity**: the throughput cost of transactions, and the growth curve of a
  `processed_events` table, both belong in the capacity model.
- **130 — SLOs**: "duplicate effects per million events" is a real SLI and almost nobody
  defines it.
- **133 — Postmortems**: the double-charge incident writes itself once you have the
  drill in this document.

---

*Java baseline 21, running on JDK 25. Spring Boot 4.1 / Framework 7.0. Three things in
this document are deliberately hedged rather than asserted: the current default values
of `transaction.timeout.ms`, `transactional.id.expiration.ms` and
`producer.id.expiration.ms`, which have moved across Kafka releases — check them with
`kafka-configs.sh --describe --entity-type brokers --entity-default`; the exact suffix
Spring for Apache Kafka appends to `transaction-id-prefix` inside a listener container,
which has changed across major versions — check the reference documentation for the
version your BOM resolves; and the on-disk log directory path in your broker image,
which depends on the image you chose in Topic 65. None of the three affects the
mechanical statement. The mechanical statement has been true since Kafka 0.11 and will
still be true the next time someone tells you their pipeline is exactly-once.*
