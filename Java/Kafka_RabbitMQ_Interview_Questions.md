# Kafka & RabbitMQ Interview Questions & Study Guide

## Overview

Apache Kafka and RabbitMQ are the two most commonly asked message broker topics in backend interviews. Kafka is a distributed event streaming platform; RabbitMQ is a traditional message broker. Knowing both — and when to use which — is the core of these questions.

---

## 1. Messaging Fundamentals

### Why Message Brokers?

Direct service-to-service calls create tight coupling. A message broker acts as a middleman, decoupling producers from consumers.

```
Without broker (tight coupling):
OrderService ──HTTP──► InventoryService (must be up)
                  └──► NotificationService (must be up)
                  └──► BillingService (must be up)

With broker (loose coupling):
OrderService ──► [Broker] ──► InventoryService   (can be down, catches up later)
                         ──► NotificationService
                         ──► BillingService
```

### Key Messaging Patterns

| Pattern | Description | Use Case |
|---|---|---|
| **Point-to-Point (Queue)** | One producer, one consumer per message | Task processing, job queues |
| **Publish-Subscribe (Topic)** | One producer, many consumers each get a copy | Event broadcasting, notifications |
| **Request-Reply** | Producer waits for consumer's response | RPC over messaging |
| **Dead Letter Queue (DLQ)** | Failed messages are moved to a special queue | Error handling, debugging |

### Delivery Semantics

| Semantic | Meaning | Risk |
|---|---|---|
| **At-most-once** | Message delivered 0 or 1 times | Message loss possible |
| **At-least-once** | Message delivered 1 or more times | Duplicate processing possible |
| **Exactly-once** | Message delivered exactly once | Most expensive to achieve |

---

## 2. Apache Kafka — Core Concepts

### What is Kafka?

Kafka is a **distributed, fault-tolerant, high-throughput event streaming platform**. It is not just a message queue — it is a persistent, ordered, replayable log.

*Analogy:* a newspaper press — events are printed and stored on shelves (partitions) for a set time; many readers (consumers) read at their own pace, and reading doesn't destroy the copy.

### Core Terminology

| Term | Description |
|---|---|
| **Event / Record / Message** | The data unit: key, value, timestamp, headers |
| **Topic** | A named, ordered, persistent log of events |
| **Partition** | A topic is split into partitions for parallelism; each partition is an ordered, immutable sequence |
| **Offset** | A unique, monotonically increasing ID for each record within a partition |
| **Broker** | A Kafka server. A cluster has multiple brokers |
| **Producer** | Writes events to topics |
| **Consumer** | Reads events from topics |
| **Consumer Group** | A set of consumers that cooperate to consume a topic; each partition is consumed by exactly one member |
| **Zookeeper / KRaft** | Manages cluster metadata; KRaft (Kafka 3.x) replaces ZooKeeper |

---

## 3. Kafka Architecture Deep Dive

### Topic & Partition Structure

```
Topic "orders" (partitions=4): each partition is an ordered, append-only sequence
Partition 0: [offset 0] [offset 1] [offset 2] ...  ← newest appended at the head
```

- Messages within a **partition** are **strictly ordered**.
- Ordering is **not guaranteed across partitions**.
- A partition is assigned to exactly **one broker as leader**; other brokers hold replicas.

### Replication & Leader/Follower

Each partition has one **leader** broker (handles reads/writes) and follower replicas. If the leader dies, a follower is automatically elected.

**ISR (In-Sync Replicas)** *(awareness):* the replicas fully caught up with the leader. Only ISR replicas can become leaders. Tuning `min.insync.replicas` and replication factor is a senior/ops concern.

### Retention Policy

Kafka retains messages by **time** or **size**, not by consumption (e.g. `log.retention.hours=168` keeps 7 days by default).

**Log Compaction** *(awareness):* `cleanup.policy=compact` keeps only the most recent record per key, so the topic holds the current "state" per key — used for event-sourcing / CDC.

---

## 4. Kafka Producers

### Producer Configuration

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "broker1:9092,broker2:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class.getName());

// Reliability settings
props.put(ProducerConfig.ACKS_CONFIG, "all");         // wait for all ISR replicas to ack
props.put(ProducerConfig.RETRIES_CONFIG, 3);
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // avoid duplicate on retry
```

*Performance knobs (awareness):* `batch.size`, `linger.ms`, and `compression.type` batch and compress records for higher throughput — tune these only when needed.

### Producer `acks` Setting — Critical Interview Topic

| acks | Meaning | Durability | Throughput |
|---|---|---|---|
| `0` | Fire and forget; no ack waited | Lowest | Highest |
| `1` | Leader acks; replicas may not have it | Medium | Medium |
| `all` (-1) | All ISR replicas must ack | Highest | Lowest |

### Partitioning Strategy

```java
ProducerRecord<String, Order> record = new ProducerRecord<>(
    "orders",     // topic
    "user-123",   // key  → same key always goes to same partition (ordering guarantee)
    order         // value
);

// No key → round-robin across partitions (no ordering guarantee, higher throughput)
ProducerRecord<String, Order> record = new ProducerRecord<>("orders", null, order);
```

**Rule**: Use a key when ordering within a business entity (e.g., all events for `user-123`) matters.

---

## 5. Kafka Consumers & Consumer Groups

### Consumer Group — How It Works

```
Topic "orders" with 4 partitions:

Consumer Group A (3 consumers):
  Consumer A1 → Partition 0
  Consumer A2 → Partition 1
  Consumer A3 → Partition 2 + 3  (one consumer can handle multiple partitions)

Consumer Group B (2 consumers):
  Consumer B1 → Partition 0 + 1
  Consumer B2 → Partition 2 + 3

Key rules:
- One partition → at most one consumer per group (no two consumers in the same group share a partition)
- One consumer can handle multiple partitions
- More consumers than partitions → some consumers are idle
- Multiple groups can all independently consume the same topic
```

### Offset Management

```java
// Auto commit (simple but can lose messages on crash)
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000);

// Manual commit (recommended for reliability)
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

consumer.poll(Duration.ofMillis(100)).forEach(record -> {
    processRecord(record);
    consumer.commitSync(); // commit after processing
});
```

### Rebalancing

A **rebalance** (partitions reassigned across the group) happens when a consumer joins/leaves or a partition is added; consumers pause during it.

*Awareness:* tune `session.timeout.ms`/`heartbeat.interval.ms` and use `CooperativeStickyAssignor` (incremental rebalancing) to reduce the impact — a senior/ops tuning concern.

### Consumer Configuration

```java
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing-group");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // or "latest"
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
```

| `auto.offset.reset` | Behavior |
|---|---|
| `earliest` | Start from beginning of topic (replay all history) |
| `latest` | Start from newest messages only |
| `none` | Throw exception if no offset found |

---

## 6. Kafka Guarantees & Delivery Semantics

### At-Most-Once (Fire and Forget)

```java
// acks=0, no retries
// Risk: message lost if broker crashes before writing
props.put(ProducerConfig.ACKS_CONFIG, "0");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
```

### At-Least-Once (Most Common)

```java
// acks=all, retries enabled, manual commit after processing
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

// Consumer: commit AFTER processing
consumer.poll(...).forEach(record -> {
    process(record);
    consumer.commitSync();
});
```

Risk: If the process crashes after processing but before committing, the message is reprocessed → **idempotent consumers required**.

### Exactly-Once (awareness)

Kafka can achieve exactly-once via three pieces: an **idempotent producer** (`enable.idempotence=true`), **transactions** (atomic write across topics, via `transactional.id` + `beginTransaction`/`commitTransaction`), and consumers reading with `isolation.level=read_committed`. The internals are a senior topic — a junior just needs to know it exists and requires all three.

---

## 7. Kafka Streams & ksqlDB

*This whole area is senior depth — a junior only needs to know what these are.*

### Kafka Streams (awareness)

A Java library for stream processing apps that read from and write to Kafka (filter, map, join, aggregate). A small example:

```java
KStream<String, Order> orders = builder.stream("orders");
orders.filter((k, o) -> o.getAmount() > 1000).to("high-value-orders");
```

**KStream vs KTable**: a KStream is an unbounded stream of independent events (append-only log); a KTable is a changelog holding the latest value per key (like a DB table).

### ksqlDB (awareness)

Lets you run SQL-like streaming queries (CREATE STREAM / CREATE TABLE ... SELECT) over Kafka topics instead of writing Java.

---

## 8. Kafka in Spring Boot

### Dependency

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
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
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
@Service
@RequiredArgsConstructor
public class OrderProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void sendOrder(Order order) {
        OrderEvent event = new OrderEvent(order.getId(), order.getUserId(), order.getAmount());

        kafkaTemplate.send("orders", order.getUserId(), event) // key = userId for ordering
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to send order {}", order.getId(), ex);
                } else {
                    log.info("Sent order {} to partition {} offset {}",
                        order.getId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### Consumer

```java
@Component
@RequiredArgsConstructor
public class OrderConsumer {

    private final InventoryService inventoryService;

    @KafkaListener(
        topics = "orders",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consume(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        try {
            inventoryService.reserve(event);
            ack.acknowledge(); // manual commit only after successful processing
        } catch (RetryableException e) {
            // don't ack → message will be redelivered
            throw e;
        } catch (NonRetryableException e) {
            ack.acknowledge(); // ack to skip poison pill, send to DLQ manually
            sendToDlq(event);
        }
    }
}
```

### Dead Letter Topic (DLT) (awareness)

Spring Kafka's `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` retries a failing message with backoff and, after N attempts, publishes it to a `{topic}.DLT` topic so it can be inspected instead of blocking the partition.

---

## 9. RabbitMQ — Core Concepts

### What is RabbitMQ?

RabbitMQ is a traditional **message broker** implementing the **AMQP** protocol. Unlike Kafka's log-based storage, RabbitMQ routes messages through exchanges to queues, and messages are deleted after consumption.

*Analogy:* a postal sorting office — producers drop letters (messages) at the office (exchange), which routes them to the right mailbox (queue); the recipient (consumer) picks up the letter and the mailbox is cleared.

### Core Components

```
Producer → Exchange → (Binding with routing key) → Queue → Consumer
```

The exchange type (direct / fanout / topic / headers) plus bindings decide which queues a message reaches.

| Component | Description |
|---|---|
| **Producer** | Publishes messages to an exchange |
| **Exchange** | Routes messages to queues based on type and routing key |
| **Binding** | Rule linking an exchange to a queue, optionally with a routing key |
| **Queue** | Buffer that stores messages until consumed |
| **Consumer** | Subscribes to a queue and processes messages |
| **Virtual Host (vhost)** | Logical grouping of exchanges, queues, and bindings (like namespaces) |

---

## 10. RabbitMQ Exchange Types

### 1. Direct Exchange

Routes to queues whose binding key **exactly matches** the routing key (e.g. `order.created` → order-created-queue only).

```java
rabbitTemplate.convertAndSend("order-exchange", "order.created", orderEvent);
```

### 2. Fanout Exchange

Ignores the routing key and broadcasts a copy to **all bound queues** — useful for broadcasting events (e.g. cache invalidation).

```java
rabbitTemplate.convertAndSend("notifications-exchange", "", event); // routing key ignored
```

### 3. Topic Exchange

Routes using **wildcard patterns** on the routing key (`*` = exactly one word, `#` = zero or more words):

```
binding: "order.*"    → matches order.created, order.updated
binding: "order.#"    → matches order, order.created, order.created.us
binding: "#.critical" → matches anything ending in .critical
```

### 4. Headers Exchange (awareness)

Routes based on **message header attributes** instead of the routing key, using `x-match=all` (AND) or `any` (OR) over the headers. Rarely needed.

**Comparison table:**

| Exchange Type | Routing Logic | Use Case |
|---|---|---|
| Direct | Exact key match | Task distribution, specific event routing |
| Fanout | Broadcast to all | Cache invalidation, pub/sub broadcast |
| Topic | Wildcard pattern | Flexible event routing, log aggregation |
| Headers | Header attributes | Complex routing rules without key conventions |

---

## 11. RabbitMQ Delivery Guarantees

### Consumer Acknowledgements

```java
// Manual ACK (recommended)
@RabbitListener(queues = "order-queue")
public void process(OrderEvent event, Channel channel,
                    @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {
    try {
        orderService.process(event);
        channel.basicAck(deliveryTag, false);  // false = single message, not batch
    } catch (RetryableException e) {
        channel.basicNack(deliveryTag, false, true);  // requeue=true → back to queue
    } catch (PoisonPillException e) {
        channel.basicNack(deliveryTag, false, false); // requeue=false → goes to DLX
    }
}
```

| Method | Description |
|---|---|
| `basicAck` | Successfully processed; remove from queue |
| `basicNack(requeue=true)` | Failed; put back in queue (be careful of infinite loops) |
| `basicNack(requeue=false)` | Failed; discard or route to Dead Letter Exchange |
| `basicReject` | Same as nack but for single message |

### Publisher Confirms (awareness)

Enabling publisher confirms (`channel.confirmSelect()` + a confirm listener) makes the broker ack that it received/persisted a message — the RabbitMQ equivalent of Kafka's `acks`.

### Dead Letter Exchange (DLX)

A message is routed to a configured DLX when the consumer nacks with `requeue=false`, the message TTL expires, or the queue length limit is exceeded. You wire it up with queue arguments:

```java
QueueBuilder.durable("order-queue")
    .withArgument("x-dead-letter-exchange", "dlx-exchange")
    .withArgument("x-message-ttl", 60000)   // message expires after 60s
    .build();
```

### Message Persistence

To survive a broker restart, **both** the queue must be `durable` AND the message sent as `PERSISTENT` (`deliveryMode=2`). If either is non-durable, messages can be lost on restart.

---

## 12. RabbitMQ in Spring Boot

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### Configuration

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10          # unacked messages a consumer holds (fair dispatch)
        concurrency: 3        # consumer threads
    publisher-confirm-type: correlated
```

### Configuration Class

Declare the exchange, queue, and binding as beans (Spring auto-creates them on startup):

```java
@Configuration
public class RabbitMQConfig {

    public static final String ORDER_EXCHANGE = "order-exchange";
    public static final String ORDER_QUEUE = "order-queue";

    @Bean
    TopicExchange orderExchange() {
        return new TopicExchange(ORDER_EXCHANGE, true, false);
    }

    @Bean
    Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", "dlx-exchange") // DLX wiring
            .build();
    }

    @Bean
    Binding orderBinding() {
        return BindingBuilder.bind(orderQueue()).to(orderExchange()).with("order.#");
    }

    @Bean
    MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

### Producer

```java
@Service
@RequiredArgsConstructor
public class OrderPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrder(Order order) {
        OrderEvent event = OrderEvent.from(order);
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            "order.created",   // routing key
            event
        );
    }
}
```

### Consumer

```java
@Component
@RequiredArgsConstructor
public class OrderConsumer {

    @RabbitListener(queues = RabbitMQConfig.ORDER_QUEUE, ackMode = "MANUAL")
    public void consume(OrderEvent event,
                        Channel channel,
                        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        try {
            inventoryService.reserve(event);
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            log.error("Processing failed for order {}", event.getOrderId(), e);
            channel.basicNack(deliveryTag, false, false); // send to DLX
        }
    }
}
```

---

## 13. Kafka vs RabbitMQ — Full Comparison

### Architecture Philosophy

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| **Model** | Distributed log / event stream | Message broker (AMQP) |
| **Storage** | Messages persisted for configurable retention period | Messages deleted after consumption (by default) |
| **Consumer model** | Pull-based (consumers poll) | Push-based (broker pushes to consumers) |
| **Message ordering** | Guaranteed within a partition | Guaranteed within a queue |
| **Routing** | Topic + partition (simple) | Exchange types (powerful, flexible) |
| **Replay** | Yes — consumers can reset offset and re-read | No — once consumed, gone |
| **Protocol** | Custom binary protocol | AMQP, MQTT, STOMP, HTTP |

### Performance & Scale

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| **Throughput** | Millions of messages/sec | Hundreds of thousands/sec |
| **Latency** | Higher (ms range, batched) | Lower (sub-ms for small loads) |
| **Scalability** | Horizontal (add partitions/brokers) | Vertical + clustering |
| **Best for** | High-throughput event streaming | Low-latency task queuing |

### Use Case Decision Guide

```
Use Kafka when:
  ✓ High throughput (millions of events/sec)
  ✓ Event replay / audit log / event sourcing
  ✓ Multiple independent consumers of the same event stream
  ✓ Stream processing, long-term retention

Use RabbitMQ when:
  ✓ Complex routing logic (topic wildcards, header routing)
  ✓ Task queues / work distribution across workers
  ✓ Low-latency delivery, request-reply (RPC)
  ✓ Per-message TTL / priority queues, simpler setup
```

---

## 14. Common Interview Questions & Answers

### Kafka Questions

**Q: How does Kafka achieve fault tolerance?**

A: Kafka replicates each partition across multiple brokers (`replication.factor`). One broker acts as the leader; others are followers. If the leader fails, one of the in-sync replicas (ISR) is automatically elected as the new leader. Producers/consumers redirect to the new leader transparently.

---

**Q: How do you ensure message ordering in Kafka?**

A: Ordering is guaranteed within a partition. To order all events for a business entity (e.g., user), use the entity ID as the message key — Kafka routes all messages with the same key to the same partition. For global ordering, use a single partition (sacrifices throughput).

---

**Q: What happens when a consumer is slower than the producer?**

A: Messages accumulate and **consumer lag** grows (up to the retention limit). Fix it by adding consumers (up to the partition count), adding partitions, or optimizing the consumer's processing.

---

**Q: What is the difference between a Kafka consumer group and multiple independent consumers?**

A: A consumer group cooperatively consumes a topic — each partition is assigned to exactly one consumer, achieving parallel processing. Independent consumers (different group IDs) each get a full copy of all messages — suitable for fan-out (multiple services reacting to the same event).

---

**Q: Explain Kafka's exactly-once semantics (EOS).** *(senior — awareness)*

A: EOS needs three things together: an idempotent producer (`enable.idempotence=true`, dedupes retries), transactions (atomic write across topics), and consumers with `isolation.level=read_committed`.

---

**Q: What is log compaction and when would you use it?**

A: It keeps only the latest record per key, so the topic holds the current state per key. Used for CDC, event sourcing, and config storage.

---

**Q: How does Kafka differ from a traditional database?**

A: Kafka is an append-only, distributed log. It doesn't support random access reads, updates, or deletes (except compaction). It's optimized for sequential reads/writes and streaming. A database supports CRUD, indexing, and complex queries. They complement each other — Kafka is often used to stream data into databases.

---

### RabbitMQ Questions

**Q: What happens to a message if no queue is bound to an exchange?**

A: The message is silently dropped (unroutable messages are discarded unless the producer set `mandatory=true`, in which case the message is returned to the producer via a return callback).

---

**Q: Explain the difference between basicNack with requeue=true vs requeue=false.**

A: `requeue=true` puts the message back on the queue to be redelivered (risking an infinite loop for non-transient errors); `requeue=false` discards it or routes it to a DLX. Use `requeue=true` for transient errors, `requeue=false` for non-retryable ones.

---

**Q: How does RabbitMQ guarantee message durability?**

A: Both the **queue** must be `durable=true` and the **message** sent with `delivery_mode=2` (persistent). If either is non-durable, messages can be lost on broker restart.

---

**Q: What is prefetch count and why does it matter?**

A: `prefetch` limits how many unacked messages a consumer holds at once. Without it, RabbitMQ floods the first available consumer, causing uneven load. `prefetch=1` gives fair dispatch (one message at a time); higher values pipeline for throughput.

---

**Q: How would you implement a retry mechanism in RabbitMQ?** *(senior — awareness)*

A: Combine DLX + TTL: nack to a DLX, route to a retry queue with `x-message-ttl`, whose DLX points back to the original exchange so the message re-enters after the delay; after N retries send it to a final dead letter queue.

---

**Q: What is a virtual host in RabbitMQ?**

A: A vhost is a logical partition within a RabbitMQ broker — like a namespace. Each vhost has its own exchanges, queues, bindings, and user permissions. Different applications or environments (dev/staging/prod) can share one RabbitMQ broker using separate vhosts for isolation.

---

**Q: Can RabbitMQ guarantee exactly-once delivery?**

A: No. RabbitMQ provides at-most-once (auto-ack) or at-least-once (manual ack) delivery. Exactly-once requires idempotent consumers — design your consumer to handle duplicate messages safely (e.g., check if order is already processed before processing it again).

---

### Comparison Questions

**Q: When would you choose Kafka over RabbitMQ?**

A: Kafka for event replay, very high throughput, many independent consumers of the same event, stream processing, and long-term retention. RabbitMQ for complex routing, low latency, work queues with fair dispatch, and per-message TTL/priority. (See the decision guide in section 13.)

---

**Q: Is Kafka a replacement for a database?**

A: No. Kafka is a streaming platform — no random-access reads, complex queries, or update/delete (only append + compaction). It's used *alongside* databases (e.g., CDC streaming changes into Kafka).

---

## 15. System Design Patterns Using Messaging

### Pattern 1: Event-Driven Microservices (Choreography)

```
OrderService ──"order.created"──► Kafka ──► InventoryService + NotificationService
InventoryService ──"inventory.reserved"──► Kafka ──► PaymentService
```

No central orchestrator — services react to events. Highly decoupled but harder to trace flow.

### Pattern 2: Saga Pattern (Distributed Transactions)

A multi-step business transaction across services, driven by events: each step (reserve stock → charge card → confirm order) publishes an event that triggers the next. If any step fails, **compensating transactions** (e.g., `inventory.release`, `payment.refund`) are published to undo the earlier steps.

### Pattern 3: CQRS + Event Sourcing

```
Command side:                          Query side:
User ──► OrderCommand ──► Kafka ──► Consumer rebuilds read model (e.g., Elasticsearch)
                                         └──► User queries read model for fast reads
```

Events are the source of truth. Read models are projections that can be rebuilt by replaying events.

### Pattern 4: Outbox Pattern (Reliable Event Publishing)

Solves the dual-write problem: how to atomically save to DB and publish to Kafka.

```java
// In a single DB transaction:
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);                          // save to orders table
    outboxRepository.save(new OutboxEvent("order.created", order)); // save to outbox table
}
// A separate Debezium CDC connector reads outbox table changes and publishes to Kafka
// This guarantees the event is published if and only if the order is saved
```

### Pattern 5: Work Queue (RabbitMQ)

```
[Producer] ──► [task-queue] ──► [Worker 1]
                            ──► [Worker 2]  ← fair dispatch with prefetch=1
                            ──► [Worker 3]
```

Distribute CPU-intensive tasks across worker instances. RabbitMQ is ideal here.

---

## 16. Quick Reference Cheat Sheet

### Kafka Key Configs Summary

| Config | Producer | Consumer | Recommended Value |
|---|---|---|---|
| `acks` | ✓ | | `all` (reliability) / `1` (balanced) |
| `enable.idempotence` | ✓ | | `true` for exactly-once |
| `retries` | ✓ | | `Integer.MAX_VALUE` with idempotence |
| `batch.size` | ✓ | | `16384` (16KB) |
| `linger.ms` | ✓ | | `5–100ms` (batching wait) |
| `enable.auto.commit` | | ✓ | `false` (manual commit recommended) |
| `auto.offset.reset` | | ✓ | `earliest` or `latest` |
| `max.poll.records` | | ✓ | `500` (tune per processing time) |
| `isolation.level` | | ✓ | `read_committed` for transactions |

### RabbitMQ Queue Arguments Summary

| Argument | Description |
|---|---|
| `x-dead-letter-exchange` | Exchange to route rejected/expired messages |
| `x-dead-letter-routing-key` | Routing key for dead-lettered messages |
| `x-message-ttl` | Max time (ms) a message lives in the queue |
| `x-max-length` | Max number of messages in queue |
| `x-max-priority` | Enable priority queue (0–255) |
| `x-queue-type: quorum` | Quorum queue for better durability (recommended for production) |

### Kafka vs RabbitMQ Decision Matrix

```
┌─────────────────────────────────────────────────────────────┐
│ Need event replay?                → Kafka                   │
│ Need complex routing?             → RabbitMQ                │
│ Need high throughput (>1M/sec)?   → Kafka                   │
│ Need low latency (<5ms)?          → RabbitMQ                │
│ Need multiple consumers, same msg → Kafka (different groups)│
│ Need work queue (1 msg, 1 worker) → RabbitMQ                │
│ Need stream processing?           → Kafka Streams           │
│ Need priority queues?             → RabbitMQ                │
│ Need long-term retention?         → Kafka                   │
│ Need request-reply / RPC?         → RabbitMQ                │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Tips

1. **Mention trade-offs** (durability vs throughput, complexity vs flexibility).

2. **Know the delivery semantics cold** — at-most-once, at-least-once, exactly-once, and the config changes each needs.

3. **Kafka offset vs RabbitMQ ack** — knowing this difference shows you understand the fundamentals of each.

4. **For system design**, know when to use each: Kafka for event sourcing / streaming / fan-out; RabbitMQ for task queues / complex routing / RPC.
