# Kafka & RabbitMQ Interview Questions & Study Guide

## Overview

Apache Kafka and RabbitMQ are the two most commonly asked message broker topics in backend interviews. Kafka is a distributed event streaming platform; RabbitMQ is a traditional message broker. Knowing both — and when to use which — is the core of these questions.

---

## Table of Contents

1. [Messaging Fundamentals](#1-messaging-fundamentals)
2. [Apache Kafka — Core Concepts](#2-apache-kafka--core-concepts)
3. [Kafka Producers & Consumers](#3-kafka-producers--consumers)
4. [Kafka Delivery Semantics](#4-kafka-delivery-semantics)
5. [Kafka in Spring Boot](#5-kafka-in-spring-boot)
6. [RabbitMQ — Core Concepts](#6-rabbitmq--core-concepts)
7. [RabbitMQ Exchange Types & Delivery](#7-rabbitmq-exchange-types--delivery)
8. [RabbitMQ in Spring Boot](#8-rabbitmq-in-spring-boot)
9. [Kafka vs RabbitMQ — Comparison](#9-kafka-vs-rabbitmq--comparison)
10. [Common Interview Questions & Answers](#10-common-interview-questions--answers)
11. [System Design Patterns](#11-system-design-patterns)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. Messaging Fundamentals

A broker decouples producers from consumers — downstream services can be down and catch up later.

### Key Patterns & Delivery Semantics

| Pattern | Description | Use Case |
|---|---|---|
| **Point-to-Point** | One consumer per message | Task queues |
| **Pub-Subscribe** | Many consumers each get a copy | Event broadcasting |
| **Dead Letter Queue** | Failed messages moved to special queue | Error handling |

| Semantic | Meaning | Risk |
|---|---|---|
| **At-most-once** | Delivered 0 or 1 times | Message loss |
| **At-least-once** | Delivered 1+ times | Duplicate processing |
| **Exactly-once** | Delivered exactly once | Most expensive |

---

## 2. Apache Kafka — Core Concepts

Kafka is a **distributed, fault-tolerant, high-throughput event streaming platform** — a persistent, ordered, replayable log.

### Core Terminology

| Term | Description |
|---|---|
| **Topic** | Named, ordered, persistent log of events |
| **Partition** | Topic split for parallelism; each is an ordered, immutable sequence |
| **Offset** | Unique monotonically increasing ID per record within a partition |
| **Broker** | A Kafka server; a cluster has multiple brokers |
| **Consumer Group** | Consumers cooperating to consume a topic; each partition assigned to exactly one member |
| **KRaft** | Replaces ZooKeeper in Kafka 3.x |

### Architecture Key Points

- Messages within a **partition** are **strictly ordered**; ordering is **not guaranteed across partitions**.
- Each partition has one **leader** broker; followers replicate it. If the leader dies, an ISR follower is elected automatically.
- Kafka retains messages by **time/size**, not by consumption (`log.retention.hours=168` = 7 days default).
- **Log Compaction** *(awareness):* `cleanup.policy=compact` keeps only the latest record per key — used for CDC / event sourcing.

---

## 3. Kafka Producers & Consumers

### Producer `acks` Setting — Critical Interview Topic

| acks | Durability | Throughput |
|---|---|---|
| `0` (fire and forget) | Lowest | Highest |
| `1` (leader acks only) | Medium | Medium |
| `all` / `-1` (all ISR ack) | Highest | Lowest |

```java
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // avoid duplicates on retry

// Key → same key always goes to same partition (ordering guarantee)
new ProducerRecord<>("orders", "user-123", order);
// No key → round-robin (higher throughput, no ordering)
new ProducerRecord<>("orders", null, order);
```

### Consumer Groups & Offset Management

```
Topic "orders" (4 partitions), Consumer Group A (3 consumers):
  Consumer A1 → Partition 0
  Consumer A2 → Partition 1
  Consumer A3 → Partition 2 + 3   ← one consumer can hold multiple partitions

Rules:
- One partition → at most one consumer per group
- More consumers than partitions → some consumers idle
- Multiple groups independently consume the same topic (fan-out)
```

```java
// Manual commit (recommended)
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // "latest" = new msgs only

consumer.poll(Duration.ofMillis(100)).forEach(record -> {
    processRecord(record);
    consumer.commitSync(); // commit only after processing
});
```

**Rebalancing:** partitions reassign when a consumer joins/leaves; consumers pause during it.

---

## 4. Kafka Delivery Semantics

### At-Least-Once (Most Common)

```java
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
// commit AFTER processing — crash before commit → reprocessed → need idempotent consumer
```

### Exactly-Once *(awareness)*

Requires all three: **idempotent producer** (`enable.idempotence=true`), **transactions** (`transactional.id` + `beginTransaction`/`commitTransaction`), and `isolation.level=read_committed` on consumers.

### Kafka Streams & ksqlDB *(awareness)*

**Kafka Streams**: Java library to filter/map/join/aggregate directly on Kafka topics. KStream = event stream; KTable = latest value per key.
**ksqlDB**: SQL-like streaming queries over Kafka without writing Java.

---

## 5. Kafka in Spring Boot

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
    consumer:
      group-id: order-service
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "com.example.dto"
    listener:
      ack-mode: MANUAL_IMMEDIATE
```

### Producer

```java
@Service @RequiredArgsConstructor
public class OrderProducer {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void sendOrder(Order order) {
        kafkaTemplate.send("orders", order.getUserId(), OrderEvent.from(order))
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("Failed to send order {}", order.getId(), ex);
                else log.info("Sent to partition {} offset {}",
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            });
    }
}
```

### Consumer

```java
@Component @RequiredArgsConstructor
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "inventory-service")
    public void consume(@Payload OrderEvent event, Acknowledgment ack) {
        try {
            inventoryService.reserve(event);
            ack.acknowledge();           // manual commit after success
        } catch (RetryableException e) {
            throw e;                     // don't ack → redelivered
        } catch (NonRetryableException e) {
            ack.acknowledge();           // skip poison pill
            sendToDlq(event);
        }
    }
}
```

**Dead Letter Topic (DLT)** *(awareness):* `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` retries with backoff and routes to `{topic}.DLT` after N failures.

---

## 6. RabbitMQ — Core Concepts

RabbitMQ is a **message broker** implementing **AMQP**. Messages are deleted after consumption (unlike Kafka's retention).

*Analogy:* a postal sorting office — producers drop messages at the exchange, which routes them to the right queue; the consumer picks up and the queue is cleared.

```
Producer → Exchange → (Binding with routing key) → Queue → Consumer
```

| Component | Description |
|---|---|
| **Exchange** | Routes messages based on type and routing key |
| **Binding** | Rule linking an exchange to a queue |
| **Queue** | Buffer storing messages until consumed |
| **Virtual Host** | Logical namespace — own exchanges, queues, permissions |

---

## 7. RabbitMQ Exchange Types & Delivery

### Exchange Types

| Type | Routing Logic | Use Case |
|---|---|---|
| **Direct** | Exact key match | Task distribution, specific event routing |
| **Fanout** | Broadcast to all queues | Cache invalidation, pub/sub |
| **Topic** | Wildcard pattern (`*`=1 word, `#`=0+ words) | Flexible routing, log aggregation |
| **Headers** | Header attributes (rarely used) | Complex routing without key conventions |

```
Topic patterns:  "order.*"    → order.created, order.updated
                 "order.#"    → order, order.created, order.created.us
```

### Consumer Acknowledgements

```java
channel.basicAck(deliveryTag, false);              // success → remove from queue
channel.basicNack(deliveryTag, false, true);       // requeue=true → redeliver (risk: infinite loop)
channel.basicNack(deliveryTag, false, false);      // requeue=false → route to DLX
```

### Dead Letter Exchange (DLX) & Durability

```java
// Wire up DLX and TTL on queue declaration:
QueueBuilder.durable("order-queue")
    .withArgument("x-dead-letter-exchange", "dlx-exchange")
    .withArgument("x-message-ttl", 60000)
    .build();
```

**Durability:** both the queue must be `durable=true` AND message sent as `PERSISTENT` (`deliveryMode=2`). Either non-durable → messages lost on restart.

**Publisher Confirms** *(awareness):* the RabbitMQ equivalent of Kafka's `acks` — broker acks that it received/persisted a message.

---

## 8. RabbitMQ in Spring Boot

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10      # unacked messages per consumer (fair dispatch)
        concurrency: 3
```

```java
@Configuration
public class RabbitMQConfig {
    public static final String ORDER_EXCHANGE = "order-exchange";
    public static final String ORDER_QUEUE    = "order-queue";

    @Bean TopicExchange orderExchange() { return new TopicExchange(ORDER_EXCHANGE, true, false); }

    @Bean Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", "dlx-exchange").build();
    }

    @Bean Binding orderBinding() {
        return BindingBuilder.bind(orderQueue()).to(orderExchange()).with("order.#");
    }

    @Bean MessageConverter jsonMessageConverter() { return new Jackson2JsonMessageConverter(); }
}
```

```java
// Producer
public void publishOrder(Order order) {
    rabbitTemplate.convertAndSend(ORDER_EXCHANGE, "order.created", OrderEvent.from(order));
}

// Consumer
@RabbitListener(queues = ORDER_QUEUE, ackMode = "MANUAL")
public void consume(OrderEvent event, Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
    try {
        inventoryService.reserve(event);
        channel.basicAck(deliveryTag, false);
    } catch (Exception e) {
        channel.basicNack(deliveryTag, false, false); // send to DLX
    }
}
```

---

## 9. Kafka vs RabbitMQ — Comparison

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| **Model** | Distributed log / event stream | Message broker (AMQP) |
| **Storage** | Retained for configurable period | Deleted after consumption |
| **Consumer model** | Pull-based (consumers poll) | Push-based (broker pushes) |
| **Ordering** | Within a partition | Within a queue |
| **Routing** | Topic + partition (simple) | Exchange types (powerful) |
| **Replay** | Yes — reset offset and re-read | No |
| **Throughput** | Millions of messages/sec | Hundreds of thousands/sec |
| **Latency** | Higher (ms, batched) | Lower (sub-ms) |
| **Best for** | High-throughput event streaming | Low-latency task queuing |

```
Use Kafka when:           Use RabbitMQ when:
  ✓ Event replay          ✓ Complex routing (wildcards, headers)
  ✓ High throughput       ✓ Task queues / work distribution
  ✓ Fan-out (many         ✓ Low-latency / request-reply (RPC)
    independent groups)   ✓ Per-message TTL / priority queues
  ✓ Stream processing
  ✓ Long-term retention
```

---

## 10. Common Interview Questions & Answers

**Q: How does Kafka achieve fault tolerance?**
A: Each partition is replicated across brokers. If the leader fails, an ISR follower is automatically elected and producers/consumers redirect transparently.

---

**Q: How do you ensure message ordering in Kafka?**
A: Ordering is guaranteed within a partition. Use the entity ID as the message key — same key always routes to the same partition. For global ordering, use a single partition (sacrifices throughput).

---

**Q: What happens when a consumer is slower than the producer?**
A: Consumer lag grows. Fix by adding consumers (up to partition count), adding partitions, or optimizing processing.

---

**Q: What is the difference between a consumer group and multiple independent consumers?**
A: A consumer group divides partitions among members for parallel processing. Independent consumers (different group IDs) each receive all messages — enabling fan-out.

---

**Q: Explain Kafka's exactly-once semantics.** *(awareness)*
A: Requires idempotent producer (`enable.idempotence=true`), transactions (`transactional.id`), and `isolation.level=read_committed` on consumers — all three together.

---

**Q: What is log compaction and when would you use it?**
A: Retains only the latest record per key so the topic holds current state. Used for CDC, event sourcing, and config/state storage.

---

**Q: What happens to a RabbitMQ message if no queue is bound to an exchange?**
A: Silently dropped unless the producer set `mandatory=true`, in which case it's returned via a return callback.

---

**Q: Explain basicNack requeue=true vs requeue=false.**
A: `requeue=true` puts the message back for redelivery (risks infinite loop). `requeue=false` discards it or routes to a DLX. Use `true` for transient errors, `false` for non-retryable ones.

---

**Q: How does RabbitMQ guarantee message durability?**
A: Both queue `durable=true` AND message `delivery_mode=2` (persistent). Either non-durable risks message loss on restart.

---

**Q: What is prefetch count?**
A: Limits unacked messages a consumer holds at once. Without it, RabbitMQ floods the first available consumer. `prefetch=1` gives fair dispatch; higher values pipeline for throughput.

---

**Q: Can RabbitMQ guarantee exactly-once delivery?**
A: No — at-most-once (auto-ack) or at-least-once (manual ack) only. Exactly-once requires idempotent consumers that handle duplicates safely.

---

**Q: What is a virtual host in RabbitMQ?**
A: A logical partition within a broker with its own exchanges, queues, and permissions. Use separate vhosts to isolate apps or environments on a shared broker.

---

## 11. System Design Patterns

### Event-Driven Microservices (Choreography)
Services react to events — no central orchestrator. Highly decoupled but harder to trace.
```
OrderService ──"order.created"──► Kafka ──► InventoryService + NotificationService
```

### Saga Pattern (Distributed Transactions)
Multi-step transaction across services via events. If any step fails, **compensating events** (e.g., `inventory.release`) undo earlier steps.

### Outbox Pattern (Reliable Event Publishing)
Solves dual-write (atomically save to DB and publish to Kafka):
```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
    outboxRepository.save(new OutboxEvent("order.created", order));
    // Debezium CDC connector reads outbox table → publishes to Kafka
}
```

### CQRS + Event Sourcing
Events are the source of truth. Read models (e.g., Elasticsearch) are projections rebuilt by replaying events.

### Work Queue (RabbitMQ)
Distribute CPU-intensive tasks across workers with `prefetch=1` for fair dispatch.

---

## 12. Quick Reference Cheat Sheet

### Kafka Key Configs

| Config | Side | Recommended Value |
|---|---|---|
| `acks` | Producer | `all` (reliability) / `1` (balanced) |
| `enable.idempotence` | Producer | `true` |
| `retries` | Producer | `Integer.MAX_VALUE` with idempotence |
| `enable.auto.commit` | Consumer | `false` (manual commit) |
| `auto.offset.reset` | Consumer | `earliest` or `latest` |
| `max.poll.records` | Consumer | `500` |
| `isolation.level` | Consumer | `read_committed` for transactions |

### RabbitMQ Queue Arguments

| Argument | Description |
|---|---|
| `x-dead-letter-exchange` | Exchange for rejected/expired messages |
| `x-message-ttl` | Max time (ms) a message lives in the queue |
| `x-max-length` | Max messages in queue |
| `x-max-priority` | Priority queue (0–255) |
| `x-queue-type: quorum` | Better durability (recommended for production) |

### Decision Matrix

```
Need event replay?                → Kafka
Need complex routing?             → RabbitMQ
Need high throughput (>1M/sec)?   → Kafka
Need low latency (<5ms)?          → RabbitMQ
Need multiple consumers, same msg → Kafka (different groups)
Need work queue (1 msg, 1 worker) → RabbitMQ
Need stream processing?           → Kafka Streams
Need priority queues?             → RabbitMQ
Need long-term retention?         → Kafka
Need request-reply / RPC?         → RabbitMQ
```

---

## Interview Tips

1. **Mention trade-offs** — durability vs throughput, complexity vs flexibility.
2. **Know delivery semantics cold** — at-most-once, at-least-once, exactly-once, and the configs each needs.
3. **Kafka offset vs RabbitMQ ack** — this difference shows you understand the fundamentals of each.
4. **System design**: Kafka for event sourcing / streaming / fan-out; RabbitMQ for task queues / complex routing / RPC.

---

*Last Updated: 2026-06-18*
