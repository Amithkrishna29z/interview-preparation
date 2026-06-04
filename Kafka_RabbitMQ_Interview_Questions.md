# Kafka & RabbitMQ Interview Questions & Study Guide

## Overview

Apache Kafka and RabbitMQ are the two most commonly asked message broker topics in backend and system design interviews. Kafka is a distributed event streaming platform; RabbitMQ is a traditional message broker. Understanding both — and when to use which — is critical for senior/mid-level backend roles.

---

## Table of Contents

1. [Messaging Fundamentals](#1-messaging-fundamentals)
2. [Apache Kafka — Core Concepts](#2-apache-kafka--core-concepts)
3. [Kafka Architecture Deep Dive](#3-kafka-architecture-deep-dive)
4. [Kafka Producers](#4-kafka-producers)
5. [Kafka Consumers & Consumer Groups](#5-kafka-consumers--consumer-groups)
6. [Kafka Guarantees & Delivery Semantics](#6-kafka-guarantees--delivery-semantics)
7. [Kafka Streams & ksqlDB](#7-kafka-streams--ksqldb)
8. [Kafka in Spring Boot](#8-kafka-in-spring-boot)
9. [RabbitMQ — Core Concepts](#9-rabbitmq--core-concepts)
10. [RabbitMQ Exchange Types](#10-rabbitmq-exchange-types)
11. [RabbitMQ Delivery Guarantees](#11-rabbitmq-delivery-guarantees)
12. [RabbitMQ in Spring Boot](#12-rabbitmq-in-spring-boot)
13. [Kafka vs RabbitMQ — Full Comparison](#13-kafka-vs-rabbitmq--full-comparison)
14. [Common Interview Questions & Answers](#14-common-interview-questions--answers)
15. [System Design Patterns Using Messaging](#15-system-design-patterns-using-messaging)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

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

```
Real-world analogy:
Kafka = A newspaper printing press
- The press (producer) prints newspapers (events)
- Newspapers are stored on shelves (partitions/topics) for a set time
- Multiple readers (consumers) can read at their own pace
- Reading doesn't destroy the newspaper
```

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
Topic: "orders"  (replication-factor=3, partitions=4)

Partition 0: [offset 0] [offset 1] [offset 2] [offset 3] ...
Partition 1: [offset 0] [offset 1] [offset 2] ...
Partition 2: [offset 0] [offset 1] ...
Partition 3: [offset 0] [offset 1] [offset 2] [offset 3] [offset 4] ...
                                                              ▲
                                                         (newest, HEAD)
```

- Messages within a **partition** are **strictly ordered**.
- Ordering is **not guaranteed across partitions**.
- A partition is assigned to exactly **one broker as leader**; other brokers hold replicas.

### Replication & Leader/Follower

```
Topic "orders" Partition 0:
  Broker 1 ← LEADER   (handles all reads and writes)
  Broker 2 ← Follower (replicates from leader)
  Broker 3 ← Follower (replicates from leader)

If Broker 1 dies → Broker 2 or 3 is elected as new leader (automatic)
```

**ISR (In-Sync Replicas)**: Set of replicas that are fully caught up with the leader. Only ISR replicas can become leaders.

### Retention Policy

Kafka retains messages by **time** or **size**, not by consumption:

```
# server.properties
log.retention.hours=168        # keep messages for 7 days (default)
log.retention.bytes=1073741824 # or until partition reaches 1 GB
log.cleanup.policy=delete      # delete old segments (default)
log.cleanup.policy=compact     # keep only latest value per key (log compaction)
```

**Log Compaction**: Keeps the most recent record per key. Useful for event-sourcing / maintaining current state.

```
Before compaction:  [user:1, "Alice"] [user:2, "Bob"] [user:1, "Alicia"]
After compaction:   [user:2, "Bob"] [user:1, "Alicia"]
```

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
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // exactly-once at producer level

// Performance settings
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);   // 16 KB batch
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);        // wait up to 5ms to fill batch
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
```

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

A **rebalance** happens when:
- A consumer joins or leaves the group
- A partition is added to the topic

During rebalance, all consumers stop consuming (**stop-the-world** pause). Minimize rebalances by:
- Setting `session.timeout.ms` and `heartbeat.interval.ms` appropriately
- Using `CooperativeStickyAssignor` (incremental rebalancing — only moves partitions that need to move)

### Consumer Configuration

```java
props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing-group");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // or "latest"
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 10000);
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

### Exactly-Once (Kafka Transactions)

```java
// Producer side
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1");

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", key, value));
    producer.send(new ProducerRecord<>("audit-log", key, auditValue));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}

// Consumer side: only read committed messages
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

---

## 7. Kafka Streams & ksqlDB

### Kafka Streams

A Java library for building stream processing applications that read from and write to Kafka.

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, Order> orders = builder.stream("orders");

KStream<String, Order> highValueOrders = orders
    .filter((key, order) -> order.getAmount() > 1000)
    .mapValues(order -> enrichOrder(order));

highValueOrders.to("high-value-orders");

KTable<String, Long> orderCountByUser = orders
    .groupByKey()
    .count(Materialized.as("order-counts"));

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

**KStream vs KTable**:
| | KStream | KTable |
|---|---|---|
| Represents | Unbounded stream of events | Changelog (latest value per key) |
| Each record | An independent event | An update to state |
| Analogy | Append-only log | Database table |

### ksqlDB

SQL-like streaming queries over Kafka topics:

```sql
-- Create a stream from a topic
CREATE STREAM orders_stream (
  order_id VARCHAR,
  user_id VARCHAR,
  amount DOUBLE
) WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='JSON');

-- Filter and push to new topic
CREATE STREAM high_value_orders AS
  SELECT * FROM orders_stream WHERE amount > 1000;

-- Aggregate: count orders per user (materialized view)
CREATE TABLE order_counts AS
  SELECT user_id, COUNT(*) as total
  FROM orders_stream
  GROUP BY user_id;
```

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

### Dead Letter Topic (DLT)

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    // By default, sends to topic "{original-topic}.DLT"

    ExponentialBackOffWithMaxRetries backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000);
    backoff.setMultiplier(2.0);

    return new DefaultErrorHandler(recoverer, backoff);
}
```

---

## 9. RabbitMQ — Core Concepts

### What is RabbitMQ?

RabbitMQ is a traditional **message broker** implementing the **AMQP** (Advanced Message Queuing Protocol). Unlike Kafka's log-based storage, RabbitMQ routes messages through exchanges to queues, and messages are deleted after consumption.

```
Real-world analogy:
RabbitMQ = A postal sorting office
- Producers drop letters (messages) at the office (exchange)
- The sorting office routes letters to the right mailbox (queue) based on rules
- The recipient (consumer) picks up the letter and the mailbox is cleared
```

### Core Components

```
Producer → Exchange → (Binding with routing key) → Queue → Consumer

[Producer]
    │
    ▼
[Exchange]  ← has type: direct / fanout / topic / headers
    │
    ├── binding: routingKey="order.created" ──► [Queue: inventory-queue]
    ├── binding: routingKey="order.*"       ──► [Queue: audit-queue]
    └── binding: (all)                      ──► [Queue: notification-queue]
                                                        │
                                                        ▼
                                                    [Consumer]
```

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

Routes to queues whose binding key **exactly matches** the routing key.

```
Exchange: order-exchange (direct)
  binding: "order.created" → Queue: order-created-queue
  binding: "order.cancelled" → Queue: order-cancelled-queue

Producer sends with routingKey="order.created"
  → goes to: order-created-queue only
```

```java
rabbitTemplate.convertAndSend("order-exchange", "order.created", orderEvent);
```

### 2. Fanout Exchange

Ignores the routing key. Broadcasts to **all bound queues**.

```
Exchange: notifications-exchange (fanout)
  bound queues: email-queue, sms-queue, push-queue

Producer sends 1 message → all 3 queues receive a copy
```

```java
rabbitTemplate.convertAndSend("notifications-exchange", "", event); // routing key ignored
```

**Use case**: Broadcasting events (e.g., cache invalidation across multiple instances).

### 3. Topic Exchange

Routes using **wildcard patterns** on the routing key.

```
Pattern rules:
  * = exactly one word
  # = zero or more words

Exchange: events-exchange (topic)
  binding: "order.*"    → Queue: order-events-queue     (matches order.created, order.updated)
  binding: "order.#"    → Queue: order-all-queue        (matches order, order.created, order.created.us)
  binding: "#.critical" → Queue: critical-alerts-queue  (matches anything ending in .critical)

Producer sends with routingKey="order.created"
  → goes to: order-events-queue AND order-all-queue
```

### 4. Headers Exchange

Routes based on **message header attributes**, not the routing key.

```java
Map<String, Object> headers = new HashMap<>();
headers.put("x-match", "all"); // "all" = AND, "any" = OR
headers.put("region", "us-east");
headers.put("priority", "high");
// Message is routed to this queue only if ALL headers match
```

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

### Publisher Confirms

Ensures the broker received the message (equivalent to Kafka's `acks`):

```java
// In RabbitMQ, enable publisher confirms on the channel
channel.confirmSelect();

channel.addConfirmListener(
    (deliveryTag, multiple) -> log.info("Message confirmed"),
    (deliveryTag, multiple) -> log.warn("Message nacked by broker")
);
```

### Dead Letter Exchange (DLX)

Messages are sent to a DLX when:
- Consumer nacks with `requeue=false`
- Message TTL expires
- Queue length limit exceeded

```java
@Bean
Queue orderQueue() {
    return QueueBuilder.durable("order-queue")
        .withArgument("x-dead-letter-exchange", "dlx-exchange")
        .withArgument("x-dead-letter-routing-key", "order.dead")
        .withArgument("x-message-ttl", 60000)   // message expires after 60s
        .withArgument("x-max-length", 10000)    // max 10k messages
        .build();
}

@Bean
Queue deadLetterQueue() {
    return QueueBuilder.durable("order-dlq").build();
}

@Bean
Binding dlqBinding() {
    return BindingBuilder.bind(deadLetterQueue())
        .to(new DirectExchange("dlx-exchange"))
        .with("order.dead");
}
```

### Message Persistence

```java
// Persistent message (survives broker restart)
MessageProperties props = new MessageProperties();
props.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
Message message = new Message(body, props);

// Queue must also be durable
Queue queue = QueueBuilder.durable("order-queue").build();
// NOT: QueueBuilder.nonDurable(...)
```

Both the queue AND message must be durable/persistent to survive a broker restart.

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
        prefetch: 10          # how many unacked messages consumer holds
        concurrency: 3        # 3 consumer threads
        max-concurrency: 10
    publisher-confirm-type: correlated
    publisher-returns: true
```

### Configuration Class

```java
@Configuration
public class RabbitMQConfig {

    public static final String ORDER_EXCHANGE = "order-exchange";
    public static final String ORDER_QUEUE = "order-queue";
    public static final String ORDER_DLQ = "order-dlq";
    public static final String DLX_EXCHANGE = "dlx-exchange";

    @Bean
    TopicExchange orderExchange() {
        return new TopicExchange(ORDER_EXCHANGE, true, false);
    }

    @Bean
    Queue orderQueue() {
        return QueueBuilder.durable(ORDER_QUEUE)
            .withArgument("x-dead-letter-exchange", DLX_EXCHANGE)
            .withArgument("x-message-ttl", 300000)
            .build();
    }

    @Bean
    Binding orderBinding() {
        return BindingBuilder.bind(orderQueue())
            .to(orderExchange())
            .with("order.#");
    }

    @Bean
    DirectExchange dlxExchange() {
        return new DirectExchange(DLX_EXCHANGE);
    }

    @Bean
    Queue deadLetterQueue() {
        return QueueBuilder.durable(ORDER_DLQ).build();
    }

    @Bean
    Binding dlqBinding() {
        return BindingBuilder.bind(deadLetterQueue())
            .to(dlxExchange())
            .with("order.dead");
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
  ✓ Stream processing (Kafka Streams, ksqlDB)
  ✓ Long-term event storage
  ✓ Microservices event-driven architecture at scale
  ✓ Real-time analytics pipelines

Use RabbitMQ when:
  ✓ Complex routing logic (topic exchange wildcards, header routing)
  ✓ Task queues / work distribution across workers
  ✓ Low-latency message delivery
  ✓ Request-reply patterns (RPC over messaging)
  ✓ Per-message TTL and priority queues
  ✓ When you need the message deleted after processing (not retained)
  ✓ Smaller scale, simpler setup
```

### Feature Comparison Table

| Feature | Kafka | RabbitMQ |
|---|---|---|
| Message replay | Yes (offset reset) | No |
| Message TTL | Topic-level retention only | Per-message and per-queue TTL |
| Priority queues | No | Yes (`x-max-priority`) |
| Message routing | Topic/key based | Exchange types (very flexible) |
| Dead letter handling | Dead letter topic (via Spring) | Dead Letter Exchange (native) |
| Exactly-once delivery | Yes (transactions + idempotent producer) | No (at-least-once max) |
| Consumer push/pull | Pull (consumer polls) | Push (broker delivers) |
| Horizontal scaling | Easy (add partitions) | Moderate (clustering, mirroring) |
| Monitoring UI | Kafka UI, AKHQ, Confluent Control Center | Built-in management UI (port 15672) |
| Cloud managed | Confluent Cloud, AWS MSK | AWS Amazon MQ, CloudAMQP |

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

A: Messages accumulate in the partition (up to the retention limit). The consumer lag grows. You can:
1. Add more consumers (up to the number of partitions)
2. Add more partitions (requires rebalance)
3. Optimize consumer processing logic
4. Use Kafka Streams for parallel processing

---

**Q: What is the difference between a Kafka consumer group and multiple independent consumers?**

A: A consumer group cooperatively consumes a topic — each partition is assigned to exactly one consumer, achieving parallel processing. Independent consumers (different group IDs) each get a full copy of all messages — suitable for fan-out (multiple services reacting to the same event).

---

**Q: Explain Kafka's exactly-once semantics (EOS).**

A: EOS in Kafka requires three things:
1. **Idempotent producer** (`enable.idempotence=true`): Each message gets a sequence number; duplicates caused by retries are deduplicated by the broker.
2. **Transactions**: Atomic write across multiple partitions/topics.
3. **Consumer isolation**: `isolation.level=read_committed` — only reads committed transaction data.

---

**Q: What is log compaction and when would you use it?**

A: Log compaction keeps only the latest record per key, removing older updates. The topic retains the current "state" of each key. Use it for:
- Change Data Capture (CDC) — replicate database table state
- Event sourcing — rebuild current state by replaying compacted log
- Configuration storage — keep latest config per key

---

**Q: How does Kafka differ from a traditional database?**

A: Kafka is an append-only, distributed log. It doesn't support random access reads, updates, or deletes (except compaction). It's optimized for sequential reads/writes and streaming. A database supports CRUD, indexing, and complex queries. They complement each other — Kafka is often used to stream data into databases.

---

### RabbitMQ Questions

**Q: What happens to a message if no queue is bound to an exchange?**

A: The message is silently dropped (unroutable messages are discarded unless the producer set `mandatory=true`, in which case the message is returned to the producer via a return callback).

---

**Q: Explain the difference between basicNack with requeue=true vs requeue=false.**

A: `requeue=true` puts the message back at the **head** of the queue — it will be delivered again immediately, risking an infinite loop if the error is non-transient. `requeue=false` discards the message (or routes it to the Dead Letter Exchange if configured). Use requeue=true for transient errors (e.g., temporary network issues) and requeue=false for non-retryable errors.

---

**Q: How does RabbitMQ guarantee message durability?**

A: Two things must both be true:
1. The **queue** must be declared as `durable=true`
2. The **message** must be sent with `delivery_mode=2` (persistent)

If either is non-durable, messages can be lost on broker restart.

---

**Q: What is prefetch count and why does it matter?**

A: `prefetch` (or `basicQos`) limits how many unacknowledged messages a consumer can hold at once. Without it, RabbitMQ pushes all messages to the first available consumer, causing uneven load distribution. Setting `prefetch=1` means each consumer processes one message at a time — fair dispatch. A higher value allows pipelining for better throughput.

---

**Q: How would you implement a retry mechanism in RabbitMQ?**

A: Use a combination of DLX and TTL:
1. Consumer nacks with `requeue=false` → message goes to a DLX
2. DLX routes to a retry queue with `x-message-ttl` set (e.g., 30 seconds)
3. Retry queue's DLX points back to the original exchange
4. Message re-enters the original queue after TTL expires
5. After N retries (tracked via a header counter), send to a final dead letter queue

---

**Q: What is a virtual host in RabbitMQ?**

A: A vhost is a logical partition within a RabbitMQ broker — like a namespace. Each vhost has its own exchanges, queues, bindings, and user permissions. Different applications or environments (dev/staging/prod) can share one RabbitMQ broker using separate vhosts for isolation.

---

**Q: Can RabbitMQ guarantee exactly-once delivery?**

A: No. RabbitMQ provides at-most-once (auto-ack) or at-least-once (manual ack) delivery. Exactly-once requires idempotent consumers — design your consumer to handle duplicate messages safely (e.g., check if order is already processed before processing it again).

---

### Comparison Questions

**Q: When would you choose Kafka over RabbitMQ?**

A: Choose Kafka when:
- You need event replay (e.g., rebuild a read model, replay after a bug fix)
- Throughput is in the millions of events/sec
- Multiple independent services need to consume the same event
- You need stream processing (Kafka Streams/ksqlDB)
- Long-term event retention is a requirement (audit logs, compliance)

Choose RabbitMQ when:
- Complex routing rules are needed (topic exchange wildcards, headers)
- Low latency is critical
- Work queues with fair dispatch among workers
- Per-message TTL or priority queues are needed
- Simpler setup and management is preferred

---

**Q: Is Kafka a replacement for a database?**

A: No. Kafka is a streaming platform, not a database. It lacks:
- Random access reads
- Complex query support
- Update/delete semantics (only append and compaction)

Kafka is often used alongside databases — e.g., CDC to stream database changes into Kafka, then into downstream systems.

---

## 15. System Design Patterns Using Messaging

### Pattern 1: Event-Driven Microservices (Choreography)

```
OrderService ──publish "order.created"──► Kafka
                                             │
                        ┌────────────────────┤
                        │                    │
                        ▼                    ▼
              InventoryService       NotificationService
              (reserves stock)       (sends confirmation email)
                        │
              publish "inventory.reserved" ──► Kafka
                                                  │
                                                  ▼
                                          PaymentService
```

No central orchestrator — services react to events. Highly decoupled but harder to trace flow.

### Pattern 2: Saga Pattern (Distributed Transactions)

```
OrderService creates order (PENDING)
  │
  ├──► Kafka: "order.created"
  │         └──► InventoryService: reserve stock
  │                   └──► Kafka: "inventory.reserved"
  │                             └──► PaymentService: charge card
  │                                       └──► Kafka: "payment.completed"
  │                                                 └──► OrderService: mark CONFIRMED
  │
  └── If any step fails:
        └──► Compensating transactions published to undo previous steps
             (e.g., "inventory.release", "payment.refund")
```

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

### Kafka Cheat Sheet

```
Topic creation:
  kafka-topics.sh --create --topic orders --partitions 4 --replication-factor 3

List topics:
  kafka-topics.sh --list --bootstrap-server localhost:9092

Describe topic:
  kafka-topics.sh --describe --topic orders

Produce messages (CLI):
  kafka-console-producer.sh --topic orders --bootstrap-server localhost:9092

Consume from beginning:
  kafka-console-consumer.sh --topic orders --from-beginning --bootstrap-server localhost:9092

Consumer group lag:
  kafka-consumer-groups.sh --describe --group my-group --bootstrap-server localhost:9092

Reset offset to beginning:
  kafka-consumer-groups.sh --reset-offsets --group my-group --topic orders --to-earliest --execute
```

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

### RabbitMQ Cheat Sheet

```
Management UI: http://localhost:15672 (guest/guest)

CLI:
  rabbitmqctl list_queues name messages consumers
  rabbitmqctl list_exchanges
  rabbitmqctl list_bindings
  rabbitmqctl purge_queue my-queue
  rabbitmqctl list_connections
```

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

1. **Always mention trade-offs.** Interviewers love hearing you think about durability vs throughput, complexity vs flexibility.

2. **Know the delivery semantics cold.** At-most-once, at-least-once, exactly-once — be able to explain each and the code changes needed.

3. **Kafka offset vs RabbitMQ ack.** Understanding this difference shows you know the fundamentals of each system.

4. **Mention the Outbox Pattern** when asked about reliable event publishing — it demonstrates you've thought about real production problems.

5. **Bring up consumer lag** when talking about Kafka scaling — it shows operational awareness.

6. **For system design**, know when to use each: Kafka for event sourcing / streaming / fan-out; RabbitMQ for task queues / complex routing / RPC.

7. **Quorum queues** (RabbitMQ 3.8+) are the production recommendation for durability — mentioning this signals real-world knowledge.
