# Apache Kafka Interview Questions & Study Guide

## Overview

Kafka is the industry-standard distributed event streaming platform, central to microservices architectures. If you're applying for a Java backend or full-stack role that involves microservices, expect Kafka questions — especially around topics, partitions, consumer groups, delivery semantics, and Spring Kafka.

---

## Table of Contents

1. [What is Kafka?](#what-is-kafka)
2. [Core Architecture](#core-architecture)
3. [Topics, Partitions & Offsets](#topics-partitions--offsets)
4. [Producers](#producers)
5. [Consumers & Consumer Groups](#consumers--consumer-groups)
6. [Brokers & Clusters](#brokers--clusters)
7. [Delivery Semantics](#delivery-semantics)
8. [Kafka vs RabbitMQ](#kafka-vs-rabbitmq)
9. [Spring Kafka](#spring-kafka)
10. [Kafka Streams](#kafka-streams)
11. [Performance & Configuration](#performance--configuration)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What is Kafka?

Apache Kafka is a **distributed event streaming platform** designed for:
- **High-throughput** publish-subscribe messaging (millions of messages/sec)
- **Durable** storage of event streams (retained on disk for days/weeks)
- **Fault-tolerant** distributed architecture (replication across brokers)
- **Replay** — consumers can re-read past events

```
Traditional Message Queue            Kafka
──────────────────────              ──────────────────────────────
Producer → Queue → Consumer         Producer → Topic (log) → Consumer Group
Message deleted after consumption   Message retained on disk (configurable)
One consumer per message            Multiple consumer groups read independently
Point-to-point or pub-sub           Always pub-sub style
```

---

## Core Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                             │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │  ← Kafka Brokers      │
│  │ (Leader) │  │(Follower)│  │(Follower)│                       │
│  └──────────┘  └──────────┘  └──────────┘                       │
│       │              │              │                            │
│       └──────────────┴──────────────┘                           │
│                     ZooKeeper / KRaft (metadata)                 │
└──────────────────────────────────────────────────────────────────┘
          ▲                               │
          │ publish                       │ subscribe
    ┌──────────────┐               ┌──────────────────┐
    │   Producers  │               │  Consumer Groups  │
    └──────────────┘               └──────────────────┘
```

### Key Components

| Component | Role |
|---|---|
| **Broker** | A Kafka server that stores and serves messages |
| **Topic** | Named channel/category for messages (like a table in DB) |
| **Partition** | Ordered, immutable log — a topic is split into partitions |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Group of consumers sharing topic partitions |
| **ZooKeeper/KRaft** | Manages cluster metadata, leader election |
| **Offset** | Unique sequential ID for each message in a partition |

---

## Topics, Partitions & Offsets

### Topic

A topic is a logical feed name. Messages are published to a topic and consumers subscribe to it.

```
Topic: "orders"
  ├── Partition 0: [msg0, msg1, msg2, msg3, ...]
  ├── Partition 1: [msg0, msg1, msg2, ...]
  └── Partition 2: [msg0, msg1, msg2, msg3, msg4, ...]
```

### Partition

A partition is an **ordered, immutable, append-only log**:
- Messages within a partition are ordered by offset
- Ordering is only guaranteed **within** a partition, NOT across partitions
- Each partition is replicated across brokers for fault tolerance
- The number of partitions = max parallel consumers in a group

```
Partition 0:  [offset 0] [offset 1] [offset 2] [offset 3] →
Partition 1:  [offset 0] [offset 1] [offset 2] →
Partition 2:  [offset 0] [offset 1] [offset 3] [offset 4] [offset 5] →
```

### Offset

- The offset is a unique, monotonically increasing integer per partition
- Consumers track which offset they've processed
- Consumers can **reset offsets** to replay past messages
- Offsets are stored in a special Kafka topic: `__consumer_offsets`

### Partition Key

```java
// No key → round-robin across partitions (even load, no ordering)
producer.send(new ProducerRecord<>("orders", value));

// With key → same key ALWAYS goes to same partition (ordered per key)
producer.send(new ProducerRecord<>("orders", "customer-123", value));
// All orders for customer-123 go to same partition → guaranteed ordering
```

> **Interview Tip**: Use a meaningful partition key (e.g., customer ID, order ID) when you need ordered processing for a specific entity. All messages with the same key always land in the same partition.

### Replication

```
Topic "orders" with replication-factor=3, 3 brokers:

Partition 0: Leader=Broker1, Replicas=Broker2, Broker3
Partition 1: Leader=Broker2, Replicas=Broker1, Broker3
Partition 2: Leader=Broker3, Replicas=Broker1, Broker2

Producers & consumers only talk to the LEADER of each partition.
Followers replicate from the leader (ISR = In-Sync Replicas).
If leader fails, ZooKeeper/KRaft elects a new leader from ISR.
```

---

## Producers

### Producer Configuration

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

// Reliability settings
props.put(ProducerConfig.ACKS_CONFIG, "all");   // wait for all ISR replicas
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // exactly-once

// Performance settings
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);          // batch messages
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);               // wait 5ms to batch more
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy"); // compress batches

KafkaProducer<String, Order> producer = new KafkaProducer<>(props);
```

### Sending Messages

```java
// Async send (fire and forget)
producer.send(new ProducerRecord<>("orders", order.getId(), order));

// Async with callback
producer.send(
    new ProducerRecord<>("orders", order.getId(), order),
    (metadata, exception) -> {
        if (exception != null) {
            log.error("Failed to send: {}", exception.getMessage());
        } else {
            log.info("Sent to partition {} offset {}", metadata.partition(), metadata.offset());
        }
    }
);

// Synchronous send (blocks until ack)
RecordMetadata meta = producer.send(record).get();
```

### acks Configuration

| `acks` | Meaning | Durability | Speed |
|---|---|---|---|
| `0` | No acknowledgement | Lowest | Fastest |
| `1` | Leader acknowledges | Medium | Medium |
| `all` (`-1`) | All ISR replicas acknowledge | Highest | Slowest |

---

## Consumers & Consumer Groups

### Consumer Group

- All consumers in a group share the partitions of a topic
- Each partition is consumed by **exactly one consumer** per group
- Multiple groups can consume the same topic independently (each gets all messages)

```
Topic "orders" with 3 partitions

Consumer Group A (3 consumers):        Consumer Group B (1 consumer):
  Consumer A1 → Partition 0              Consumer B1 → Partition 0
  Consumer A2 → Partition 1              Consumer B1 → Partition 1
  Consumer A3 → Partition 2              Consumer B1 → Partition 2

Group A processes each partition once.
Group B gets all messages independently.
```

### Parallelism Rule

```
Number of partitions = max useful consumers per group

3 partitions → max 3 consumers in a group can work in parallel
              (4th consumer would be idle — no partition assigned)
```

### Consumer Configuration

```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service-group");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

// When to commit offsets
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");        // manual commit
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");      // replay from start
// "latest" → only new messages from now (default)
// "earliest" → replay all messages from beginning

KafkaConsumer<String, Order> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));
```

### Consuming Messages

```java
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Order> record : records) {
        log.info("Received: topic={} partition={} offset={} key={} value={}",
            record.topic(), record.partition(), record.offset(),
            record.key(), record.value());
        processOrder(record.value());
    }

    consumer.commitSync();  // commit after processing batch (at-least-once)
}
```

### Rebalancing

When a consumer joins or leaves a group, Kafka **rebalances** — reassigns partitions to consumers. During rebalance, no consumption happens (brief pause).

---

## Brokers & Clusters

```bash
# Create a topic
kafka-topics.sh --create --topic orders \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 2

# List topics
kafka-topics.sh --list --bootstrap-server localhost:9092

# Describe topic (partitions, leaders, replicas)
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092

# Produce messages (CLI)
kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092

# Consume messages (CLI)
kafka-console-consumer.sh --topic orders --from-beginning --bootstrap-server localhost:9092

# Consumer group info
kafka-consumer-groups.sh --describe --group order-service-group --bootstrap-server localhost:9092
```

---

## Delivery Semantics

| Semantic | Description | Risk | Config |
|---|---|---|---|
| **At-most-once** | Messages may be lost; never duplicated | Data loss | `acks=0`, no retry |
| **At-least-once** | No data loss; duplicates possible | Duplicates | `acks=all`, retry, idempotent consumer |
| **Exactly-once** | No loss, no duplicates | Complexity | Idempotent producer + Kafka transactions |

### At-Least-Once (most common)

```java
// Producer: acks=all + retries
// Consumer: manually commit AFTER processing
for (ConsumerRecord<String, Order> record : records) {
    processOrder(record.value()); // process first
}
consumer.commitSync();           // commit only after success
// If processing fails → don't commit → message re-delivered → at-least-once
```

**Problem**: If app crashes after processing but before commit → message re-delivered → duplicate!

**Solution**: Make consumers **idempotent** — check if already processed before acting.

### Exactly-Once (Kafka Transactions)

```java
// Producer config
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1");

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", order));
    producer.send(new ProducerRecord<>("audit", auditEvent));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

---

## Kafka vs RabbitMQ

| Feature | Kafka | RabbitMQ |
|---|---|---|
| **Architecture** | Log-based (partitioned, replicated log) | Queue-based (messages deleted after consumption) |
| **Message retention** | Yes (configurable, days/weeks) | No (deleted after ack) |
| **Throughput** | Very high (millions/sec) | Medium (thousands/sec) |
| **Replay** | Yes — consumers can reset offsets | No |
| **Consumer model** | Pull (consumer polls) | Push (broker pushes to consumer) |
| **Ordering** | Per partition | Per queue |
| **Use case** | Event streaming, audit log, analytics | Task queues, RPC, complex routing |
| **Protocol** | Custom Kafka protocol | AMQP |
| **Routing** | By topic + partition key | Exchanges + routing keys (flexible) |

**Choose Kafka when**: High throughput, event replay needed, multiple independent consumers, event sourcing, stream processing.  
**Choose RabbitMQ when**: Complex routing logic, task queues with per-message acknowledgement, lower throughput requirements.

---

## Spring Kafka

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### Configuration

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
    consumer:
      group-id: order-service-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "com.example.dto"
    listener:
      ack-mode: MANUAL_IMMEDIATE
```

### Producer — KafkaTemplate

```java
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, Order> kafkaTemplate;

    public void sendOrder(Order order) {
        // Simple send
        kafkaTemplate.send("orders", order.getId(), order);

        // Send with callback
        CompletableFuture<SendResult<String, Order>> future =
            kafkaTemplate.send("orders", order.getId(), order);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to send order {}: {}", order.getId(), ex.getMessage());
            } else {
                log.info("Order {} sent to partition {} offset {}",
                    order.getId(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            }
        });
    }
}
```

### Consumer — @KafkaListener

```java
@Service
public class OrderConsumer {

    @KafkaListener(
        topics = "orders",
        groupId = "order-service-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(Order order, Acknowledgment ack,
                        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                        @Header(KafkaHeaders.OFFSET) long offset) {
        try {
            log.info("Received order {} from partition {} offset {}", order.getId(), partition, offset);
            processOrder(order);
            ack.acknowledge();  // manual commit after successful processing
        } catch (Exception e) {
            log.error("Error processing order {}: {}", order.getId(), e.getMessage());
            // Don't ack — message will be retried
        }
    }

    // Multiple topics
    @KafkaListener(topics = {"orders", "returns"})
    public void consumeMultiple(ConsumerRecord<String, String> record) {
        log.info("Topic: {}, Key: {}, Value: {}", record.topic(), record.key(), record.value());
    }

    // Batch listener
    @KafkaListener(topics = "orders")
    public void consumeBatch(List<Order> orders, Acknowledgment ack) {
        orders.forEach(this::processOrder);
        ack.acknowledge();
    }
}
```

### Error Handling & Retry

```java
@Bean
public DefaultErrorHandler errorHandler() {
    // Retry 3 times with 1s delay, then send to DLT
    FixedBackOff backOff = new FixedBackOff(1000L, 3L);
    DefaultErrorHandler handler = new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(kafkaTemplate), backOff
    );
    // Don't retry these exceptions
    handler.addNotRetryableExceptions(IllegalArgumentException.class);
    return handler;
}
```

### Dead Letter Topic (DLT)

Messages that fail after all retries are sent to `<original-topic>.DLT`:

```java
// Consumer for DLT
@KafkaListener(topics = "orders.DLT")
public void consumeDeadLetter(Order order,
    @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage) {
    log.error("Dead letter: order {} failed with: {}", order.getId(), errorMessage);
    // alert, store for manual review, etc.
}
```

### Testing Kafka with EmbeddedKafka

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 3,
    topics = {"orders"},
    brokerProperties = {"listeners=PLAINTEXT://localhost:9092"}
)
class OrderProducerTest {

    @Autowired KafkaTemplate<String, Order> kafkaTemplate;
    @Autowired KafkaConsumer<String, Order> consumer;

    @Test
    void sendOrder_messageReceivedByConsumer() throws Exception {
        Order order = new Order("order-1", 99.99);
        kafkaTemplate.send("orders", order.getId(), order).get();

        ConsumerRecord<String, Order> received =
            KafkaTestUtils.getSingleRecord(consumer, "orders");
        assertThat(received.value().getId()).isEqualTo("order-1");
    }
}
```

---

## Kafka Streams

Kafka Streams is a **client library** for building stream processing applications on Kafka.

```java
StreamsBuilder builder = new StreamsBuilder();

// Read from topic
KStream<String, Order> orders = builder.stream("orders");

// Filter
KStream<String, Order> highValue = orders
    .filter((key, order) -> order.getAmount() > 1000);

// Transform
KStream<String, String> summaries = orders
    .mapValues(order -> "Order " + order.getId() + ": $" + order.getAmount());

// Group and aggregate
KTable<String, Long> countPerCustomer = orders
    .groupBy((key, order) -> order.getCustomerId())
    .count(Materialized.as("order-counts-store"));

// Branch
Map<String, KStream<String, Order>> branches = orders
    .split(Named.as("branch-"))
    .branch((k, v) -> v.getAmount() > 1000, Branched.as("high-value"))
    .branch((k, v) -> v.getAmount() <= 1000, Branched.as("regular"))
    .defaultBranch();

// Write to output topic
highValue.to("high-value-orders");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

---

## Performance & Configuration

### Producer Tuning

```properties
# Throughput optimization
batch.size=65536            # 64KB batch (default 16KB)
linger.ms=20                # wait 20ms for more messages to batch
compression.type=lz4        # compress batches (lz4 is fastest)
buffer.memory=67108864      # 64MB producer buffer

# Reliability
acks=all
retries=2147483647          # max retries
max.in.flight.requests.per.connection=5  # with idempotence enabled
enable.idempotence=true
```

### Consumer Tuning

```properties
fetch.min.bytes=1048576     # 1MB min fetch (reduces round trips)
fetch.max.wait.ms=500       # wait up to 500ms for min bytes
max.poll.records=500        # max records per poll
session.timeout.ms=30000    # consumer heartbeat timeout
max.poll.interval.ms=300000 # max processing time per poll
```

### Topic Configuration

```properties
retention.ms=604800000      # 7 days (default)
retention.bytes=-1          # no size limit (default)
cleanup.policy=delete       # delete old segments (vs compact)
min.insync.replicas=2       # with acks=all, require 2 ISR copies
```

---

## Common Interview Questions

### Q: What is the difference between a topic and a partition?

- **Topic**: Logical category/channel for messages. Producers publish to topics; consumers subscribe to topics.
- **Partition**: Physical unit of storage. A topic is split into N partitions. Each partition is an ordered, append-only log stored on a broker. Partitions enable parallelism and horizontal scaling.

---

### Q: How does Kafka guarantee message ordering?

Kafka guarantees ordering **within a partition** only. If you need all messages for a specific entity (e.g., one customer's orders) in order, use that entity's ID as the partition key — all messages with the same key always go to the same partition.

---

### Q: What happens if a consumer is slower than the producer?

Messages accumulate on the Kafka broker (within retention limits). Since Kafka is a pull-based system, the consumer catches up at its own pace. You can increase parallelism by adding more consumers to the group (up to the partition count) or by processing messages in parallel within a consumer.

---

### Q: What is consumer group rebalancing?

When consumers join or leave a group (or a new topic matches a subscribed pattern), Kafka reassigns partitions among the active consumers. During rebalancing, all consumers stop consuming briefly. Use `incremental cooperative rebalancing` (default in recent versions) to minimize pauses.

---

### Q: What is the difference between `auto.offset.reset=earliest` and `latest`?

- `earliest`: Start consuming from the **beginning** of the topic (offset 0). Used for replaying all historical messages.
- `latest` (default): Start from the **latest offset** — only consume new messages published after the consumer started.

---

### Q: How do you implement exactly-once semantics?

1. **Idempotent producer**: Enable `enable.idempotence=true` — Kafka deduplicates retried messages.
2. **Kafka Transactions**: Use `initTransactions()` + `beginTransaction()` + `commitTransaction()` to atomically write to multiple topics.
3. **Idempotent consumer**: Track processed message IDs (store in DB) — check before processing.

---

## Quick Reference Cheat Sheet

```
Core concepts:
  Broker     → Kafka server (stores partitions)
  Topic      → named message channel
  Partition  → ordered log within a topic (parallelism unit)
  Offset     → sequential ID for each message in a partition
  Consumer Group → consumers sharing topic partitions (1 partition → 1 consumer per group)
  ISR        → In-Sync Replicas (must match leader to be eligible for election)

Ordering:
  → guaranteed within a partition only
  → use partition key to group related messages to same partition

Producer acks:
  0 = no ack (fastest, no guarantee)
  1 = leader ack
  all/-1 = all ISR ack (safest)

Delivery:
  at-most-once  → acks=0, commit before process
  at-least-once → acks=all, commit after process, idempotent consumer
  exactly-once  → idempotent producer + Kafka transactions

Consumer auto.offset.reset:
  earliest = replay from start
  latest   = only new messages

Parallelism:
  max useful consumers per group = number of partitions
  add partitions to increase parallelism (can't remove)

Spring Kafka:
  @KafkaListener  → consume messages
  KafkaTemplate   → produce messages
  Acknowledgment  → manual commit (ack-mode: MANUAL)
  @EmbeddedKafka  → in-memory Kafka for tests

Kafka vs RabbitMQ:
  Kafka → high throughput, retention, replay, event streaming
  RabbitMQ → complex routing, task queues, lower throughput

DLT (Dead Letter Topic) → messages that fail after all retries
```

---

*Last Updated: 2026-06-04*
