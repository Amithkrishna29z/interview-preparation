# Apache Kafka Interview Questions & Study Guide

## Overview

Kafka is the industry-standard distributed event streaming platform, central to microservices architectures. If you're applying for a Java backend or full-stack role, expect Kafka questions — especially around topics, partitions, consumer groups, delivery semantics, and Spring Kafka.

---

## Table of Contents

1. [What is Kafka?](#what-is-kafka)
2. [Core Architecture](#core-architecture)
3. [Topics, Partitions & Offsets](#topics-partitions--offsets)
4. [Producers](#producers)
5. [Consumers & Consumer Groups](#consumers--consumer-groups)
6. [Delivery Semantics](#delivery-semantics)
7. [Kafka vs RabbitMQ](#kafka-vs-rabbitmq)
8. [Spring Kafka](#spring-kafka)
9. [Common Interview Questions](#common-interview-questions)
10. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

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
```

---

## Core Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │  ← Kafka Brokers      │
│  │ (Leader) │  │(Follower)│  │(Follower)│                       │
│  └──────────┘  └──────────┘  └──────────┘                       │
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
| **Topic** | Named channel/category for messages |
| **Partition** | Ordered, immutable log — a topic is split into partitions |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Group of consumers sharing topic partitions |
| **ZooKeeper/KRaft** | Manages cluster metadata, leader election |
| **Offset** | Unique sequential ID for each message in a partition |

---

## Topics, Partitions & Offsets

### Topic & Partition

A topic is a logical feed name split into **ordered, immutable, append-only** partitions:
- Ordering is guaranteed **within** a partition only, NOT across partitions
- Each partition is replicated across brokers for fault tolerance
- Number of partitions = max parallel consumers in a group

```
Topic: "orders"
  ├── Partition 0: [offset 0] [offset 1] [offset 2] →
  ├── Partition 1: [offset 0] [offset 1] →
  └── Partition 2: [offset 0] [offset 1] [offset 2] [offset 3] →
```

### Offset

- Unique, monotonically increasing integer per partition
- Consumers track which offset they've processed and can reset to replay messages
- Stored in the internal topic `__consumer_offsets`

### Partition Key

```java
// No key → round-robin across partitions (no ordering guarantee)
producer.send(new ProducerRecord<>("orders", value));

// With key → same key ALWAYS goes to same partition (ordered per key)
producer.send(new ProducerRecord<>("orders", "customer-123", value));
```

> **Interview Tip**: Use a meaningful partition key (e.g., customer ID) when you need ordered processing for a specific entity.

### Replication

```
Partition 0: Leader=Broker1, Replicas=Broker2, Broker3
Partition 1: Leader=Broker2, Replicas=Broker1, Broker3

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
props.put(ProducerConfig.ACKS_CONFIG, "all");             // wait for all ISR replicas
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // exactly-once
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);             // wait 5ms to batch more

KafkaProducer<String, Order> producer = new KafkaProducer<>(props);
```

### Sending Messages

```java
// Async with callback (preferred)
producer.send(
    new ProducerRecord<>("orders", order.getId(), order),
    (metadata, exception) -> {
        if (exception != null) log.error("Failed: {}", exception.getMessage());
        else log.info("Sent to partition {} offset {}", metadata.partition(), metadata.offset());
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
- Multiple groups can consume the same topic independently

```
Topic "orders" — 3 partitions

Consumer Group A (3 consumers):        Consumer Group B (1 consumer):
  Consumer A1 → Partition 0              Consumer B1 → all 3 partitions
  Consumer A2 → Partition 1
  Consumer A3 → Partition 2
```

**Parallelism rule**: Adding a 4th consumer to Group A above leaves it idle — max useful consumers = number of partitions.

### Consumer Configuration

```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service-group");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");   // manual commit
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // replay from start

KafkaConsumer<String, Order> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));
```

### Consuming Messages

```java
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Order> record : records) {
        processOrder(record.value());
    }
    consumer.commitSync(); // commit after processing batch (at-least-once)
}
```

### Rebalancing

When a consumer joins or leaves a group, Kafka **rebalances** — reassigns partitions among active consumers. All consumption pauses briefly during rebalance.

---

## Delivery Semantics

| Semantic | Description | Risk | Config |
|---|---|---|---|
| **At-most-once** | Messages may be lost; never duplicated | Data loss | `acks=0`, no retry |
| **At-least-once** | No data loss; duplicates possible | Duplicates | `acks=all`, retry, idempotent consumer |
| **Exactly-once** | No loss, no duplicates | Complexity | Idempotent producer + Kafka transactions |

### At-Least-Once (most common)

```java
// Commit AFTER processing — if crash before commit, message is re-delivered
for (ConsumerRecord<String, Order> record : records) {
    processOrder(record.value());
}
consumer.commitSync();
```

Make consumers **idempotent** — check if a message was already processed before acting to handle duplicates.

### Exactly-Once (Kafka Transactions)

```java
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
| **Consumer model** | Pull (consumer polls) | Push (broker pushes) |
| **Ordering** | Per partition | Per queue |
| **Use case** | Event streaming, audit log, analytics | Task queues, RPC, complex routing |

**Choose Kafka when**: High throughput, event replay, multiple independent consumers, event sourcing.  
**Choose RabbitMQ when**: Complex routing logic, task queues, lower throughput requirements.

---

## Spring Kafka

### Configuration (application.yml)

```yaml
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
        CompletableFuture<SendResult<String, Order>> future =
            kafkaTemplate.send("orders", order.getId(), order);

        future.whenComplete((result, ex) -> {
            if (ex != null) log.error("Failed to send order {}: {}", order.getId(), ex.getMessage());
            else log.info("Order {} sent to partition {} offset {}",
                order.getId(),
                result.getRecordMetadata().partition(),
                result.getRecordMetadata().offset());
        });
    }
}
```

### Consumer — @KafkaListener

```java
@Service
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "order-service-group")
    public void consume(Order order, Acknowledgment ack,
                        @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                        @Header(KafkaHeaders.OFFSET) long offset) {
        try {
            processOrder(order);
            ack.acknowledge(); // manual commit after successful processing
        } catch (Exception e) {
            log.error("Error processing order {}: {}", order.getId(), e.getMessage());
            // Don't ack — message will be retried
        }
    }
}
```

### Error Handling & Dead Letter Topic (DLT)

```java
@Bean
public DefaultErrorHandler errorHandler() {
    // Retry 3 times with 1s delay, then send to DLT
    return new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(kafkaTemplate),
        new FixedBackOff(1000L, 3L)
    );
}

// Consume failed messages from DLT (named <original-topic>.DLT)
@KafkaListener(topics = "orders.DLT")
public void consumeDeadLetter(Order order,
    @Header(KafkaHeaders.EXCEPTION_MESSAGE) String errorMessage) {
    log.error("Dead letter: order {} failed with: {}", order.getId(), errorMessage);
}
```

### Testing with EmbeddedKafka

```java
@SpringBootTest
@EmbeddedKafka(partitions = 3, topics = {"orders"})
class OrderProducerTest {

    @Autowired KafkaTemplate<String, Order> kafkaTemplate;

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

## Common Interview Questions

### Q: What is the difference between a topic and a partition?

A **topic** is the logical category producers publish to and consumers subscribe to. A **partition** is the physical unit — a topic is split into N ordered, append-only logs stored on brokers. Partitions enable parallelism and horizontal scaling.

---

### Q: How does Kafka guarantee message ordering?

Only **within a partition**. Use a meaningful partition key (e.g., customer ID) so all messages for the same entity always land in the same partition.

---

### Q: What happens if a consumer is slower than the producer?

Messages accumulate on the broker within retention limits. Since Kafka is pull-based, the consumer catches up at its own pace. Add more consumers to the group (up to the partition count) to increase parallelism.

---

### Q: What is consumer group rebalancing?

When consumers join or leave a group, Kafka reassigns partitions among active consumers. All consumers pause briefly during rebalance. Recent Kafka versions use incremental cooperative rebalancing to minimize this pause.

---

### Q: What is the difference between `auto.offset.reset=earliest` and `latest`?

- `earliest`: Start from **offset 0** — replays all historical messages.
- `latest` (default): Start from the **latest offset** — only new messages published after the consumer started.

---

### Q: How do you implement exactly-once semantics?

1. **Idempotent producer**: `enable.idempotence=true` — Kafka deduplicates retried messages.
2. **Kafka Transactions**: `initTransactions()` + `beginTransaction()` + `commitTransaction()` to atomically write to multiple topics.
3. **Idempotent consumer**: Track processed message IDs in a DB — check before processing.

---

## Quick Reference Cheat Sheet

```
Core concepts:
  Broker        → Kafka server (stores partitions)
  Topic         → named message channel
  Partition     → ordered log within a topic (parallelism unit)
  Offset        → sequential ID per message in a partition
  Consumer Group → consumers sharing partitions (1 partition → 1 consumer per group)
  ISR           → In-Sync Replicas (eligible for leader election)

Ordering:
  → guaranteed within a partition only
  → use partition key to route related messages to the same partition

Producer acks:
  0    = no ack (fastest, no guarantee)
  1    = leader ack
  all  = all ISR ack (safest)

Delivery:
  at-most-once  → acks=0, commit before process
  at-least-once → acks=all, commit after process, idempotent consumer
  exactly-once  → idempotent producer + Kafka transactions

Consumer auto.offset.reset:
  earliest = replay from start
  latest   = only new messages

Parallelism:
  max useful consumers per group = number of partitions

Spring Kafka:
  @KafkaListener  → consume messages
  KafkaTemplate   → produce messages
  Acknowledgment  → manual commit (ack-mode: MANUAL)
  @EmbeddedKafka  → in-memory Kafka for tests

Kafka vs RabbitMQ:
  Kafka     → high throughput, retention, replay, event streaming
  RabbitMQ  → complex routing, task queues, lower throughput

DLT (Dead Letter Topic) → messages that fail after all retries
```

---

*Last Updated: 2026-06-18*
