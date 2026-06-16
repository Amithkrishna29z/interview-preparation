# Backend Engineering Mastery Roadmap – Complete Study Guide

> **Target Audience:** Java/Spring Boot developers ready to level up from junior/mid to senior/architect-level roles.  
> **How to Use:** Study each phase in order. The phases build on each other. For each topic, understand the *why* before the *how*.

---

## Table of Contents

- [Phase 1 – SQL Deep Dive + PostgreSQL Internals + Redis](#phase-1)
- [Phase 2 – Kafka + Event-Driven Systems + Messaging Patterns](#phase-2)
- [Phase 3 – System Design Fundamentals + HLD + LLD](#phase-3)
- [Phase 4 – Docker + Kubernetes](#phase-4)
- [Phase 5 – AWS](#phase-5)
- [Phase 6 – Distributed Systems + Consistency + Consensus](#phase-6)
- [Phase 7 – AI Agents + Spring AI + Modern Architectures](#phase-7)

---

# Phase 1 – SQL Deep Dive + PostgreSQL Internals + Redis {#phase-1}

---

## 1.1 SQL Deep Dive

### Core Concepts

**Real-world analogy:** SQL is like asking precise questions to an extremely organized librarian. The better your question, the faster and more accurate the answer.

### ACID Properties

| Property | Meaning | Real-World Example |
|----------|---------|-------------------|
| Atomicity | All or nothing | Bank transfer: debit + credit both succeed or both fail |
| Consistency | Data always valid | Account balance never goes negative if a constraint exists |
| Isolation | Transactions don't interfere | Two users booking the last seat see a clean state |
| Durability | Committed data survives crash | Order confirmed = order exists even after server restart |

### Indexing Deep Dive

**B-Tree Index (default)** – Used for equality and range queries.
```sql
-- Creates a B-Tree index by default
CREATE INDEX idx_users_email ON users(email);

-- Efficient: uses the index
SELECT * FROM users WHERE email = 'john@example.com';

-- Efficient: range scan on B-Tree
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

**Hash Index** – Only for exact equality, faster than B-Tree for =
```sql
CREATE INDEX idx_session_token ON sessions USING HASH (token);
```

**Composite Index – Column Order Matters**
```sql
-- Index on (last_name, first_name)
-- Works for: WHERE last_name = 'Smith'
-- Works for: WHERE last_name = 'Smith' AND first_name = 'John'
-- DOES NOT work for: WHERE first_name = 'John' alone (leading column rule)
CREATE INDEX idx_name ON users(last_name, first_name);
```

**Partial Index** – Index only a subset of rows
```sql
-- Only indexes active users – smaller, faster
CREATE INDEX idx_active_users ON users(email) WHERE status = 'ACTIVE';
```

**Covering Index** – Index contains all columns the query needs (avoids heap fetch)
```sql
-- Query only touches the index, never touches the actual table rows
CREATE INDEX idx_covering ON orders(user_id, status, created_at);
SELECT user_id, status, created_at FROM orders WHERE user_id = 123;
```

### Window Functions (Senior-Level Must-Know)

```sql
-- ROW_NUMBER: Assign a unique rank per partition
SELECT
    employee_id,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_in_dept
FROM employees;

-- LAG/LEAD: Compare with previous/next row
SELECT
    order_date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY order_date) AS prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_revenue;

-- Running total
SELECT
    order_id,
    amount,
    SUM(amount) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;

-- NTILE: Divide into buckets (e.g., quartiles)
SELECT
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) AS spending_quartile
FROM customer_summary;
```

### CTEs vs Subqueries

```sql
-- CTE: Readable, can be referenced multiple times
WITH high_value_customers AS (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
    HAVING SUM(amount) > 10000
),
customer_details AS (
    SELECT c.name, c.email, h.total
    FROM customers c
    JOIN high_value_customers h ON c.id = h.customer_id
)
SELECT * FROM customer_details ORDER BY total DESC;

-- Recursive CTE: For hierarchical data (org charts, trees)
WITH RECURSIVE org_tree AS (
    -- Base case: top-level managers
    SELECT id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case
    SELECT e.id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT * FROM org_tree ORDER BY level, name;
```

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Use When |
|-------|-----------|--------------------|-----------|----|
| READ UNCOMMITTED | Possible | Possible | Possible | Almost never |
| READ COMMITTED | Prevented | Possible | Possible | Most apps (PostgreSQL default) |
| REPEATABLE READ | Prevented | Prevented | Possible | Financial reports |
| SERIALIZABLE | Prevented | Prevented | Prevented | High-integrity banking |

```sql
-- Setting isolation level for a transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;
-- ... other reads ...
COMMIT;
```

### Query Optimization with EXPLAIN ANALYZE

```sql
-- Always use EXPLAIN ANALYZE in development to understand query plans
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name
ORDER BY order_count DESC;

-- Look for:
-- Seq Scan = full table scan (bad for large tables) → add an index
-- Index Scan = good, uses an index
-- Hash Join vs Nested Loop = planner's join strategy
-- cost=X..Y: estimated cost (X=startup, Y=total)
-- actual time=X..Y rows=Z: actual timing and rows
```

---

## 1.2 PostgreSQL Internals

### MVCC – Multi-Version Concurrency Control

**Real-world analogy:** Like Google Docs version history. When you edit, PostgreSQL keeps the old version visible to other transactions still reading it — no one blocks anyone.

```
Every row in PostgreSQL has hidden system columns:
- xmin: transaction ID that inserted this row
- xmax: transaction ID that deleted/updated this row (0 if still live)

When you UPDATE a row, PostgreSQL:
1. Marks old row's xmax = current transaction ID (logically deleted)
2. Inserts a NEW row with xmin = current transaction ID
3. Old row stays until VACUUM cleans it up
```

### WAL – Write-Ahead Log

**How it works:**
1. Before any data change is written to disk, it's first written to the WAL
2. WAL is append-only — extremely fast sequential writes
3. On crash, PostgreSQL replays WAL to recover to a consistent state

**Why it matters:** WAL is also the mechanism for **streaming replication** (primary → replica sync).

```
WAL file location: $PGDATA/pg_wal/
WAL files are 16MB by default
Archive WAL for point-in-time recovery (PITR)
```

### VACUUM and autovacuum

```sql
-- VACUUM reclaims space from dead tuples left by MVCC
VACUUM orders;           -- reclaim dead tuple space
VACUUM FULL orders;      -- compact the table (acquires exclusive lock!)
VACUUM ANALYZE orders;   -- reclaim + update statistics for query planner

-- Check bloat (dead tuples)
SELECT
    schemaname, tablename,
    n_live_tup, n_dead_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Connection Pooling (PgBouncer)

**Problem:** PostgreSQL creates a full OS process per connection — 500 connections = 500 processes = memory exhaustion.

**Solution:** PgBouncer sits between your app and Postgres, maintaining a small pool of real DB connections and multiplexing thousands of app connections through them.

```
App (1000 connections) → PgBouncer → PostgreSQL (20 connections)

Modes:
- Session pooling: one real conn per app session (least efficient)
- Transaction pooling: real conn per transaction (recommended for most apps)
- Statement pooling: real conn per statement (most restrictive)
```

### Partitioning

```sql
-- Range partitioning: split a huge orders table by year
CREATE TABLE orders (
    id BIGINT,
    user_id BIGINT,
    amount DECIMAL,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- List partitioning: split by region
CREATE TABLE sales PARTITION BY LIST (region);
CREATE TABLE sales_us PARTITION OF sales FOR VALUES IN ('US', 'CA', 'MX');
CREATE TABLE sales_eu PARTITION OF sales FOR VALUES IN ('UK', 'DE', 'FR');
```

### PostgreSQL-Specific Features

```sql
-- JSONB: Binary JSON with indexing
CREATE TABLE events (
    id BIGINT PRIMARY KEY,
    metadata JSONB
);
CREATE INDEX idx_events_metadata ON events USING GIN (metadata);

-- Query JSONB
SELECT * FROM events WHERE metadata @> '{"type": "click"}';
SELECT metadata->>'user_id' FROM events WHERE id = 1;

-- Full-Text Search
SELECT title, ts_rank(to_tsvector('english', body), query) AS rank
FROM articles,
     to_tsquery('english', 'postgresql & indexing') query
WHERE to_tsvector('english', body) @@ query
ORDER BY rank DESC;

-- UPSERT (INSERT ON CONFLICT)
INSERT INTO user_preferences (user_id, theme, language)
VALUES (123, 'dark', 'en')
ON CONFLICT (user_id) DO UPDATE
SET theme = EXCLUDED.theme,
    language = EXCLUDED.language,
    updated_at = NOW();
```

---

## 1.3 Redis

### Core Data Structures and When to Use Them

| Structure | Use Case | Example |
|-----------|----------|---------|
| String | Cache, counters, sessions | User session tokens |
| List | Message queues, activity feeds | Recent 100 notifications |
| Set | Unique collections, tags | Online users, user tags |
| Sorted Set | Leaderboards, rate limiting | Top 10 scores |
| Hash | Object storage | User profile fields |
| Bitmap | Feature flags, user tracking | Daily active user tracking |
| HyperLogLog | Approximate distinct counts | Unique page views |
| Stream | Event log | Real-time event processing |

```java
// Spring Boot Redis examples
@Service
public class CacheService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    // String: cache user session
    public void cacheSession(String token, String userId, long ttlMinutes) {
        redisTemplate.opsForValue().set("session:" + token, userId,
            Duration.ofMinutes(ttlMinutes));
    }

    // Sorted Set: leaderboard
    public void updateScore(String gameId, String playerId, double score) {
        redisTemplate.opsForZSet().add("leaderboard:" + gameId, playerId, score);
    }

    public Set<String> getTopPlayers(String gameId, int count) {
        return redisTemplate.opsForZSet()
            .reverseRange("leaderboard:" + gameId, 0, count - 1);
    }

    // Hash: user profile
    public void updateUserProfile(String userId, Map<String, String> fields) {
        redisTemplate.opsForHash().putAll("user:" + userId, fields);
    }

    // Atomic increment: rate limiting
    public boolean isRateLimited(String userId, int maxRequests, long windowSeconds) {
        String key = "rate:" + userId + ":" + (System.currentTimeMillis() / (windowSeconds * 1000));
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, Duration.ofSeconds(windowSeconds));
        }
        return count > maxRequests;
    }
}
```

### Caching Patterns

**Cache-Aside (Lazy Loading)** – Most common
```java
public User getUser(Long id) {
    String key = "user:" + id;
    User cached = (User) redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;          // Cache hit

    User user = userRepository.findById(id);    // Cache miss → DB
    redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
    return user;
}
```

**Write-Through** – Write to cache and DB simultaneously
```java
public User saveUser(User user) {
    User saved = userRepository.save(user);     // Write to DB
    redisTemplate.opsForValue().set("user:" + saved.getId(), saved, Duration.ofMinutes(30));
    return saved;                               // Write to cache
}
```

**Write-Behind (Write-Back)** – Write to cache, async flush to DB (risky, risk of data loss)

### Redis Persistence

| Mode | How | Durability | Performance |
|------|-----|-----------|-------------|
| RDB | Snapshot at intervals | May lose last few minutes | Fast restart |
| AOF | Append every write to log | Near zero data loss | Slower restart |
| RDB + AOF | Both | Best | Combined |
| No persistence | Memory only | Full loss on crash | Fastest |

### Distributed Locking with Redisson

```java
// Redisson handles the Redlock algorithm properly
@Service
public class InventoryService {

    @Autowired
    private RedissonClient redissonClient;

    public boolean purchaseItem(String itemId, int quantity) {
        RLock lock = redissonClient.getLock("lock:inventory:" + itemId);
        try {
            // Try to acquire lock, wait up to 5s, hold for max 30s
            boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
            if (!acquired) throw new RuntimeException("Could not acquire lock");

            int stock = getStock(itemId);
            if (stock < quantity) return false;

            updateStock(itemId, stock - quantity);
            return true;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        } finally {
            if (lock.isHeldByCurrentThread()) lock.unlock();
        }
    }
}
```

### Pub/Sub vs Redis Streams

```java
// Pub/Sub: fire-and-forget, no persistence, no consumer groups
redisTemplate.convertAndSend("notifications", "User 123 placed an order");

// Redis Streams: persistent, consumer groups, acknowledgements (prefer this)
// Producer
redisTemplate.opsForStream().add("orders-stream", Map.of(
    "orderId", "456",
    "userId", "123",
    "amount", "99.99"
));

// Consumer with consumer group
redisTemplate.opsForStream().createGroup("orders-stream", "order-processors");
List<MapRecord<String, Object, Object>> messages =
    redisTemplate.opsForStream().read(
        Consumer.from("order-processors", "consumer-1"),
        StreamReadOptions.empty().count(10),
        StreamOffset.create("orders-stream", ReadOffset.lastConsumed())
    );
```

---

## Phase 1 – Interview Questions & Answers

**Q1: What is the difference between a clustered and non-clustered index?**

A: In PostgreSQL (unlike SQL Server), all tables are heap-organized — there is no true clustered index. However, you can use `CLUSTER` to physically reorder a table once. In SQL Server/MySQL InnoDB, the clustered index IS the table — rows are stored sorted by the primary key. A non-clustered index is a separate B-Tree structure with a pointer (row ID or primary key) back to the actual row.

**Q2: Explain MVCC and why PostgreSQL uses it.**

A: MVCC (Multi-Version Concurrency Control) allows reads and writes to happen concurrently without blocking each other. Instead of locking rows for reads, PostgreSQL keeps multiple versions of each row. A reading transaction sees a snapshot of data as of its start time, ignoring newer versions. This avoids read-write lock contention. The tradeoff is dead tuple accumulation, which VACUUM must periodically clean up.

**Q3: When would you use a Sorted Set in Redis over a regular Set?**

A: When you need ordering. A Set stores unique members with no order. A Sorted Set (ZSet) stores unique members each with a floating-point score. Use Sorted Sets for: leaderboards (rank by score), rate limiting with sliding windows (timestamp as score), priority queues, time-series expiry (TTL as score). The tradeoff is O(log N) operations vs O(1) for regular Set membership checks.

**Q4: What is the N+1 query problem and how do you fix it?**

A: The N+1 problem occurs when fetching N entities and then making 1 additional query per entity to load a related collection. Example: loading 100 users, then 100 separate queries to load each user's orders. Fix with: SQL JOIN + GROUP BY, `JOIN FETCH` in JPA/Hibernate, batch loading (`@BatchSize`), or using projections/DTOs with a single query.

**Q5: Explain PostgreSQL's WAL and how it enables replication.**

A: WAL (Write-Ahead Log) is a sequential append-only log written before any actual data modification. On crash, PostgreSQL replays unfinished WAL entries to recover. For replication, the primary streams WAL segments to standby replicas. The replica applies WAL changes to stay in sync. This is called streaming replication. Physical replication replays raw WAL bytes; logical replication decodes WAL into row-level changes that can be applied selectively.

**Q6: What Redis eviction policy would you choose for a session store?**

A: For session stores, use `volatile-lru` (evict least recently used keys that have a TTL set) or `volatile-ttl` (evict keys with the soonest expiry). This ensures Redis only evicts session keys (which have TTLs) and never evicts permanent data. Avoid `allkeys-lru` for session stores if you have mixed permanent and ephemeral data.

**Q7: How do you design a rate limiter using Redis?**

A: Two common approaches:
- **Fixed window**: `INCR rate:{userId}:{minute}` → if > limit, reject. Cheap but has edge-case burst at window boundary.
- **Sliding window with Sorted Set**: Store each request timestamp as a score. On each request: remove old entries (score < now - window), count remaining, add current timestamp. If count > limit, reject. More accurate, slightly more expensive.

**Q8: What is connection pooling and why is it critical for PostgreSQL?**

A: PostgreSQL spawns a full OS process per connection. A thousand app connections = a thousand processes consuming ~5-10MB RAM each = 5-10GB of RAM just for connections, plus context-switching overhead. Connection pooling (PgBouncer, HikariCP) maintains a small pool of real DB connections and multiplexes many app threads through them. HikariCP pools at the application level (per JVM). PgBouncer pools at the infrastructure level (shared across multiple app instances).

---

# Phase 2 – Kafka + Event-Driven Systems + Messaging Patterns {#phase-2}

---

## 2.1 Kafka Architecture

### Core Concepts

**Real-world analogy:** Kafka is like a newspaper printing press. The press (producer) prints papers once. Each subscriber (consumer group) maintains their own bookmark of which edition they've read. New subscribers can read from the very beginning. Old editions are kept for a configurable period.

```
Kafka Architecture:
┌─────────────┐     ┌──────────────────────────────────────┐     ┌─────────────┐
│  Producers  │────▶│  Kafka Cluster (Brokers + ZooKeeper) │────▶│  Consumers  │
└─────────────┘     │                                      │     └─────────────┘
                    │  Topic: "orders"                     │
                    │  ┌──────────┐ ┌──────────┐ ┌──────────┐
                    │  │Partition0│ │Partition1│ │Partition2│
                    │  │[msg0]    │ │[msg3]    │ │[msg6]    │
                    │  │[msg1]    │ │[msg4]    │ │[msg7]    │
                    │  │[msg2]    │ │[msg5]    │ │[msg8]    │
                    │  └──────────┘ └──────────┘ └──────────┘
                    └──────────────────────────────────────┘
```

### Key Configuration Properties

```java
// Producer configuration
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Reliability settings
        config.put(ProducerConfig.ACKS_CONFIG, "all");         // all ISR must confirm
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // exactly-once

        // Performance settings
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);        // batch for 5ms
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);   // 16KB batch
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        return new DefaultKafkaProducerFactory<>(config);
    }
}

// Consumer configuration
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // Manual offset control for at-least-once processing
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

### Producer with Error Handling

```java
@Service
public class OrderEventProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(order.getId(), "ORDER_CREATED", order);

        CompletableFuture<SendResult<String, OrderEvent>> future =
            kafkaTemplate.send("orders", order.getId().toString(), event);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("Failed to send order event for orderId={}", order.getId(), ex);
                // Save to outbox table for retry
                outboxRepository.save(new OutboxEvent(event));
            } else {
                log.info("Order event sent: partition={}, offset={}",
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            }
        });
    }
}
```

### Consumer with Manual Offset Commit

```java
@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "orders", groupId = "inventory-service",
                   containerFactory = "kafkaListenerContainerFactory")
    public void handleOrderEvent(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment acknowledgment) {

        try {
            OrderEvent event = record.value();
            log.info("Processing event: type={}, orderId={}", event.getType(), event.getOrderId());

            switch (event.getType()) {
                case "ORDER_CREATED" -> inventoryService.reserveStock(event);
                case "ORDER_CANCELLED" -> inventoryService.releaseStock(event);
            }

            acknowledgment.acknowledge(); // Commit offset AFTER successful processing

        } catch (RetryableException e) {
            log.warn("Retryable error, will retry: {}", e.getMessage());
            // Don't acknowledge — Kafka will redeliver
            throw e;
        } catch (Exception e) {
            log.error("Non-retryable error, sending to DLQ", e);
            deadLetterPublisher.send("orders.DLT", record);
            acknowledgment.acknowledge(); // Acknowledge to avoid infinite loop
        }
    }
}
```

### Delivery Guarantees

| Guarantee | How | Risk |
|-----------|-----|------|
| At-most-once | Commit offset before processing | Data loss on crash |
| At-least-once | Commit offset after processing | Duplicate processing |
| Exactly-once | Idempotent producer + transactional API | Highest overhead |

```java
// Exactly-once with Kafka Transactions
@Bean
public KafkaTransactionManager<String, Object> kafkaTransactionManager(
        ProducerFactory<String, Object> pf) {
    return new KafkaTransactionManager<>(pf);
}

@Transactional("kafkaTransactionManager")
public void processAndForward(ConsumerRecord<String, OrderEvent> record) {
    // Read from topic A and write to topic B atomically
    kafkaTemplate.send("processed-orders", transformEvent(record.value()));
}
```

---

## 2.2 Event-Driven Systems

### Event Sourcing

**Real-world analogy:** Your bank statement. The bank doesn't store "current balance = $500". It stores every deposit and withdrawal. The balance is derived by replaying all events.

```java
// Instead of storing current state...
public class Order {
    private String status;  // WRONG: mutable state
}

// Store events
public abstract class OrderEvent {
    private String orderId;
    private Instant occurredAt;
    private String eventType;
}

public class OrderCreatedEvent extends OrderEvent { ... }
public class OrderShippedEvent extends OrderEvent { ... }
public class OrderCancelledEvent extends OrderEvent { ... }

// Rebuild state by replaying events
public Order buildFromEvents(List<OrderEvent> events) {
    Order order = new Order();
    events.forEach(event -> {
        if (event instanceof OrderCreatedEvent e) order.apply(e);
        else if (event instanceof OrderShippedEvent e) order.apply(e);
        else if (event instanceof OrderCancelledEvent e) order.apply(e);
    });
    return order;
}
```

### CQRS – Command Query Responsibility Segregation

```
Command Side (Write)           Query Side (Read)
┌─────────────────┐            ┌───────────────────┐
│  Command        │            │  Query Handler    │
│  Handler        │────sync───▶│  (reads from      │
│  (writes to     │  or async  │  read-optimized   │
│  write DB)      │            │  view/cache)      │
└─────────────────┘            └───────────────────┘
        │                              ▲
        │                              │
        └──── Event ──▶ Projector ─────┘
                         (builds read models)
```

```java
// Command (write)
@RestController
public class OrderCommandController {
    @PostMapping("/orders")
    public ResponseEntity<String> createOrder(@RequestBody CreateOrderCommand cmd) {
        String orderId = orderCommandService.handle(cmd);
        return ResponseEntity.accepted().body(orderId);
    }
}

// Query (read from denormalized read model)
@RestController
public class OrderQueryController {
    @GetMapping("/orders/{id}")
    public OrderSummaryDto getOrder(@PathVariable String id) {
        return orderQueryService.findById(id); // reads from Redis or read DB
    }
}

// Projector: updates read model when events occur
@KafkaListener(topics = "order-events")
public void project(OrderCreatedEvent event) {
    OrderSummaryDto dto = new OrderSummaryDto(event.getOrderId(), event.getUserId(), ...);
    redisTemplate.opsForValue().set("order:" + event.getOrderId(), dto);
    orderReadRepository.save(dto); // also persist for durability
}
```

---

## 2.3 Messaging Patterns

### Saga Pattern (Distributed Transaction Coordination)

```
Choreography-based Saga (event-driven):

OrderService ──ORDER_CREATED──▶ InventoryService
                                    │
                             STOCK_RESERVED ──▶ PaymentService
                                                     │
                                              PAYMENT_CHARGED ──▶ ShippingService

If any step fails:
PaymentService ──PAYMENT_FAILED──▶ InventoryService (compensate: release stock)
                                 ──▶ OrderService (compensate: cancel order)
```

```java
// Orchestration-based Saga (centralized coordinator)
@Service
public class OrderSagaOrchestrator {

    public void executeSaga(CreateOrderCommand cmd) {
        String sagaId = UUID.randomUUID().toString();

        try {
            // Step 1: Reserve inventory
            InventoryReservation reservation = inventoryClient.reserve(cmd.getItems());
            sagaLog.save(sagaId, "INVENTORY_RESERVED", reservation);

            // Step 2: Charge payment
            PaymentResult payment = paymentClient.charge(cmd.getUserId(), cmd.getAmount());
            sagaLog.save(sagaId, "PAYMENT_CHARGED", payment);

            // Step 3: Create shipment
            shipmentClient.create(cmd.getOrderId(), cmd.getAddress());
            sagaLog.save(sagaId, "SHIPMENT_CREATED", null);

        } catch (PaymentFailedException e) {
            // Compensate: release inventory
            inventoryClient.release(sagaLog.get(sagaId, "INVENTORY_RESERVED"));
            orderRepository.updateStatus(cmd.getOrderId(), "PAYMENT_FAILED");
        }
    }
}
```

### Outbox Pattern (Guaranteed Event Delivery)

**Problem:** You need to save to DB and publish to Kafka atomically. If the DB commit succeeds but Kafka publish fails, you have inconsistent state.

```java
// Solution: Save the event IN the same DB transaction
@Service
@Transactional
public class OrderService {

    public Order createOrder(CreateOrderCommand cmd) {
        Order order = new Order(cmd);
        orderRepository.save(order);                    // Save entity

        // Save event to outbox IN THE SAME TRANSACTION
        OutboxEvent outboxEvent = new OutboxEvent(
            "orders",                                   // topic
            order.getId().toString(),                   // key
            new OrderCreatedEvent(order)                // payload
        );
        outboxRepository.save(outboxEvent);

        return order;
        // Both saves happen atomically.
        // A separate "relay" process polls outbox and publishes to Kafka.
    }
}

// Relay process (runs on a schedule or CDC-triggered)
@Scheduled(fixedDelay = 500)
public void relayEvents() {
    List<OutboxEvent> pending = outboxRepository.findUnpublished();
    pending.forEach(event -> {
        kafkaTemplate.send(event.getTopic(), event.getKey(), event.getPayload());
        event.markPublished();
        outboxRepository.save(event);
    });
}
```

### Dead Letter Queue (DLQ)

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, Object> kafkaListenerContainerFactory() {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, Object>();
    factory.setConsumerFactory(consumerFactory());

    // Configure dead-letter publishing after 3 retries
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer,
        new FixedBackOff(1000L, 3L)); // retry 3x with 1s delay

    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```

---

## Phase 2 – Interview Questions & Answers

**Q1: How does Kafka guarantee message ordering?**

A: Kafka guarantees ordering within a single partition. Messages with the same key always go to the same partition (via consistent hashing on the key). So all events for order #123 go to the same partition, in order. But there is NO global ordering across partitions. Design tip: use the entity ID (orderId, userId) as the key to ensure related events are ordered.

**Q2: What is the difference between Kafka and RabbitMQ?**

A: Kafka is a distributed log/event streaming platform — messages are retained for a configurable period and can be replayed. Multiple consumer groups can independently read the same topic. RabbitMQ is a traditional message broker — messages are deleted after consumption, routing is flexible (exchanges, routing keys), and it's better for task queues and RPC patterns. Kafka shines for high-throughput event streaming and audit logs. RabbitMQ shines for complex routing and when you need message TTL per message.

**Q3: Explain the Outbox Pattern and why it's needed.**

A: When you must atomically write to a database AND publish an event to Kafka, you face a dual-write problem: the DB write might succeed but the Kafka publish might fail (or vice versa). The Outbox Pattern solves this by storing the event in an outbox table in the same database transaction as the business data. A separate relay process (or CDC tool like Debezium) then reads from the outbox and publishes to Kafka. This way, you only need the database transaction to be atomic — the relay handles eventual Kafka delivery with retry.

**Q4: What is a consumer group and how does it enable parallel processing?**

A: A consumer group is a set of consumers that jointly consume a topic. Each partition is assigned to exactly one consumer in the group at a time. So if a topic has 6 partitions and a consumer group has 3 consumers, each consumer handles 2 partitions in parallel. Adding more consumers than partitions provides no additional parallelism (excess consumers are idle). Multiple separate consumer groups each receive all messages independently.

**Q5: What is the Saga pattern and when would you use choreography vs orchestration?**

A: The Saga pattern manages distributed transactions by breaking them into local transactions with compensating actions on failure. Choreography: each service listens for events and emits new events — no central coordinator, more decoupled, but harder to trace the overall flow. Orchestration: a central saga orchestrator tells each service what to do and handles failures — easier to reason about, single point of failure risk. Use choreography for simple flows with few steps; use orchestration for complex flows where you need a clear audit trail and explicit compensation logic.

---

# Phase 3 – System Design Fundamentals + HLD + LLD {#phase-3}

---

## 3.1 System Design Fundamentals

### Scalability Patterns

| Problem | Solution | Trade-off |
|---------|---------|----------|
| Single server overloaded | Horizontal scaling + Load Balancer | Complexity, session management |
| Database read bottleneck | Read replicas | Replication lag |
| Database write bottleneck | Sharding | Cross-shard queries become hard |
| Repeated expensive computations | Caching layer (Redis) | Cache invalidation complexity |
| Large files / static assets | CDN | Cost, cache invalidation |
| Chatty microservices | API Gateway | Single point of failure |
| Tight coupling | Message queue (async) | Eventual consistency |

### The CAP Theorem (For System Design Discussions)

```
CAP: You can only guarantee 2 of 3:
- Consistency: Every read gets the most recent write
- Availability: Every request gets a response (not necessarily latest)
- Partition Tolerance: System works despite network partitions

In a distributed system, P is non-negotiable (networks do fail).
So the real choice is: CP vs AP

CP systems (consistency + partition tolerance): ZooKeeper, HBase, etcd
AP systems (availability + partition tolerance): Cassandra, DynamoDB, CouchDB
CA systems (consistency + availability): Traditional RDBMS (single node)
```

### Load Balancing Algorithms

| Algorithm | How | Best For |
|-----------|-----|----------|
| Round Robin | Cycle through servers | Equal-capacity, stateless services |
| Least Connections | Route to server with fewest active connections | Long-lived connections (WebSocket) |
| IP Hash | Hash client IP to server | Session stickiness |
| Weighted Round Robin | More weight = more traffic | Heterogeneous server capacities |
| Random | Random server | Simple stateless workloads |

### Caching Strategies

```
Cache levels (fastest to slowest):
L1: In-process cache (Caffeine, HashMap) — microseconds
L2: Distributed cache (Redis, Memcached) — sub-millisecond
L3: Database query cache — milliseconds
L4: Full response cache (Varnish, CDN) — varies

Cache invalidation strategies:
1. TTL-based: expire after N seconds (simple, may serve stale data)
2. Event-based: invalidate when underlying data changes (accurate, complex)
3. Write-through: update cache on every write (consistent, write overhead)
4. Cache-aside: populate on miss (simple, risk of cache stampede)

Cache stampede prevention:
- Mutex lock: only one thread populates, others wait
- Probabilistic early expiration: refresh before TTL expires
- Background refresh: serve stale while refreshing asynchronously
```

### Database Sharding

```
Sharding strategies:

1. Range-based:
   user_id 1-1M → Shard A
   user_id 1M-2M → Shard B
   Pro: Range queries easy. Con: Hotspots (new users all go to latest shard)

2. Hash-based:
   shard = hash(user_id) % num_shards
   Pro: Even distribution. Con: Range queries require scatter-gather, resharding is hard

3. Directory-based (lookup table):
   A lookup service maps entity → shard
   Pro: Maximum flexibility. Con: Lookup service is a bottleneck/SPOF

Cross-shard issues:
- Joins: must be done at application layer
- Aggregates: scatter-gather then merge
- Transactions: no ACID across shards (use Saga)
```

---

## 3.2 High-Level Design (HLD)

### Designing a URL Shortener (Classic Interview Question)

```
Requirements:
- Shorten URLs: POST /shorten → returns short code
- Redirect: GET /{code} → 301/302 redirect to long URL
- 100M URLs, 10B redirects/day

HLD:
┌────────┐    ┌──────────┐    ┌───────────┐    ┌──────────┐
│Client  │───▶│Load Bal. │───▶│API Service│───▶│PostgreSQL│
└────────┘    └──────────┘    └───────────┘    └──────────┘
                                    │
                              ┌─────▼──────┐
                              │   Redis    │  (cache hot URLs)
                              └────────────┘
                                    │
                              ┌─────▼──────┐
                              │   CDN      │  (cache redirects at edge)
                              └────────────┘

ID Generation:
Option 1: MD5/SHA256 of long URL → take first 7 chars (collision risk)
Option 2: Base62 encode an auto-increment ID (predictable, sequential)
Option 3: Snowflake ID → Base62 encode (distributed, non-sequential)

Cache strategy:
- 80/20 rule: 20% of URLs get 80% of traffic
- Cache top 20% in Redis with TTL
- Redirect cache: 301 (permanent, client caches) vs 302 (temporary, always hits server)
```

### Designing a Notification System

```
Requirements:
- Send email, SMS, push notifications
- 1M notifications/day
- Support templates, user preferences, do-not-disturb

HLD:
API ──▶ Kafka (notifications topic) ──▶ Notification Service
                                              │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                               Email Svc  SMS Svc  Push Svc
                                 (SES)   (Twilio)  (FCM/APNS)

Key decisions:
- Use Kafka for decoupling and buffering burst traffic
- Per-channel worker pools for independent scaling
- User preference check before sending (opt-outs, quiet hours)
- Idempotency: store notification_id, skip if already sent
- Rate limiting per user/channel to avoid spam
- Dead letter queue for failed deliveries + retry with backoff
```

---

## 3.3 Low-Level Design (LLD)

### Designing a Rate Limiter Class

```java
// Token Bucket Rate Limiter
public class TokenBucketRateLimiter {
    private final long capacity;
    private final double refillRatePerMs;
    private double tokens;
    private long lastRefillTime;

    public TokenBucketRateLimiter(long capacity, long refillRatePerSecond) {
        this.capacity = capacity;
        this.refillRatePerMs = refillRatePerSecond / 1000.0;
        this.tokens = capacity;
        this.lastRefillTime = System.currentTimeMillis();
    }

    public synchronized boolean tryConsume() {
        refill();
        if (tokens >= 1) {
            tokens--;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        double tokensToAdd = (now - lastRefillTime) * refillRatePerMs;
        tokens = Math.min(capacity, tokens + tokensToAdd);
        lastRefillTime = now;
    }
}

// Distributed: same logic but in Redis Lua script for atomicity
String luaScript = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    ...
    """;
```

### Designing a Parking Lot System (OOP Design)

```java
// Key classes
public enum VehicleType { MOTORCYCLE, CAR, TRUCK }
public enum SpotType { SMALL, MEDIUM, LARGE }

public abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;
    public abstract SpotType requiredSpotType();
}

public class Car extends Vehicle {
    public Car(String plate) { this.licensePlate = plate; this.type = VehicleType.CAR; }
    public SpotType requiredSpotType() { return SpotType.MEDIUM; }
}

public class ParkingSpot {
    private String id;
    private SpotType type;
    private boolean occupied;
    private Vehicle parkedVehicle;

    public boolean canFit(Vehicle v) {
        return !occupied && this.type == v.requiredSpotType();
    }
}

public class ParkingLot {
    private List<ParkingLevel> levels;

    public Optional<ParkingSpot> park(Vehicle vehicle) {
        return levels.stream()
            .flatMap(l -> l.getSpots().stream())
            .filter(spot -> spot.canFit(vehicle))
            .findFirst()
            .map(spot -> { spot.park(vehicle); return spot; });
    }

    public void leave(ParkingSpot spot) {
        spot.free();
        billingService.charge(spot.getParkedVehicle(), spot.getDuration());
    }
}
```

---

## Phase 3 – Interview Questions & Answers

**Q1: How would you design Twitter's trending topics feature?**

A: A sliding window approach: stream all tweets to Kafka, a consumer counts hashtag occurrences in 1-minute tumbling windows using a distributed stream processor (Kafka Streams or Flink). Store counts in Redis Sorted Sets (hashtag as member, count as score). A periodic job reads top-N from Redis and publishes to a "trending" cache. Serve from cache via CDN for reads. The key challenges are: handling hashtag spam/manipulation (velocity checks), regional trends (partition by geography), and decay functions so trending reflects recent activity not all-time counts.

**Q2: How do you handle database schema migrations in a zero-downtime deployment?**

A: Use the expand-contract pattern:
1. **Expand**: Add the new column/table (nullable, non-breaking). Deploy new code that writes to both old and new.
2. **Migrate**: Background job backfills data into the new structure.
3. **Contract**: Once all data is migrated and reads switch to new structure, remove the old column in a separate deployment.
Never do a rename or destructive migration in a single deployment. Tools: Flyway, Liquibase for versioned migrations.

**Q3: When would you choose microservices over a monolith?**

A: Start with a monolith. Adopt microservices when: (1) specific parts of the system need to scale independently with different resource profiles, (2) different teams need to deploy independently without coordination, (3) a domain is truly autonomous with its own data store. Don't microservice too early — the operational complexity (networking, observability, distributed transactions) is real cost. Martin Fowler's rule: "Don't start with microservices." Extract services when the monolith's boundaries become clear.

**Q4: Explain consistent hashing and why it's used.**

A: In standard hashing (`node = hash(key) % N`), adding/removing a node reshuffles almost all keys. Consistent hashing places nodes on a virtual ring and assigns each key to the nearest clockwise node. When a node is added, only the keys between the new node and its predecessor are reassigned — roughly `1/N` of keys instead of nearly all. Used in: distributed caches (Memcached, Redis Cluster), load balancers with session stickiness, Cassandra's token ring.

---

# Phase 4 – Docker + Kubernetes {#phase-4}

---

## 4.1 Docker

### How Docker Works Internally

**Real-world analogy:** Docker containers are like shipping containers. The container standardizes the interface (dimensions, locking mechanism) so it works the same on any ship, truck, or crane. The app inside doesn't care what the ship looks like.

```
Linux primitives that Docker uses:
- Namespaces: isolate PID, network, mount, UTS, IPC, user spaces
- cgroups (control groups): limit CPU, memory, I/O per container
- Union filesystems (OverlayFS): layer-based image system
- seccomp: restrict system calls a container can make
```

### Dockerfile Best Practices

```dockerfile
# Multi-stage build: build in one stage, ship only runtime artifacts
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
# Cache dependency layer separately (only re-runs if pom.xml changes)
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# Production image: only JRE, no build tools
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
# Run as non-root user for security
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
# Use exec form (not shell form) so signals reach the JVM
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-jar", "app.jar"]
```

```yaml
# docker-compose for local development
version: '3.9'
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_REDIS_HOST: redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes

volumes:
  postgres_data:
```

---

## 4.2 Kubernetes

### Architecture Overview

```
Kubernetes Cluster:
┌────────────────────────────────────┐
│            Control Plane           │
│  ┌──────────┐  ┌────────────────┐  │
│  │API Server│  │etcd (key-value)│  │
│  └──────────┘  └────────────────┘  │
│  ┌──────────┐  ┌────────────────┐  │
│  │Scheduler │  │Controller Mgr  │  │
│  └──────────┘  └────────────────┘  │
└────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Worker 1   │  │   Worker 2   │  │   Worker 3   │
│  ┌─────────┐ │  │  ┌─────────┐ │  │  ┌─────────┐ │
│  │kubelet  │ │  │  │kubelet  │ │  │  │kubelet  │ │
│  │kube-    │ │  │  │kube-    │ │  │  │kube-    │ │
│  │  proxy  │ │  │  │  proxy  │ │  │  │  proxy  │ │
│  │[Pod][Po]│ │  │  │[Pod]    │ │  │  │[Pod][Po]│ │
│  └─────────┘ │  │  └─────────┘ │  │  └─────────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Core Kubernetes Objects

```yaml
# Deployment: manages a ReplicaSet, handles rolling updates
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired count during update
      maxUnavailable: 0  # Zero-downtime: never kill old pod before new one is ready
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:            # Guaranteed resources (used for scheduling)
            memory: "256Mi"
            cpu: "250m"
          limits:              # Hard ceiling (OOM kill if exceeded)
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:        # When to add to load balancer
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:         # When to restart the container
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: environment

---
# Service: stable network endpoint for Pods
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # Internal only; use LoadBalancer for external, NodePort for dev

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up when avg CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# Ingress: HTTP routing to services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

### ConfigMaps and Secrets

```yaml
# ConfigMap: non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  environment: "production"
  log-level: "INFO"
  max-connections: "100"

---
# Secret: sensitive data (base64-encoded, not encrypted by default)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: c2VjcmV0cGFzc3dvcmQ=  # base64("secretpassword")
  # For production: use Vault, AWS Secrets Manager, or Sealed Secrets
```

---

## Phase 4 – Interview Questions & Answers

**Q1: What is the difference between a Docker image and a Docker container?**

A: An image is a read-only blueprint — a layered filesystem with the application code, dependencies, and configuration. A container is a running instance of an image. Multiple containers can run from the same image simultaneously. Think of an image as a class definition and a container as an object instance.

**Q2: What happens when a Pod is OOMKilled in Kubernetes?**

A: The container's memory usage exceeded its `limits.memory`. The kernel's OOM killer terminates the process. Kubernetes restarts the container according to the pod's `restartPolicy` (usually `Always`). To fix: increase memory limits, find and fix memory leaks, or use JVM flags like `-XX:MaxRAMPercentage=75.0` to respect container limits (required for JVM apps).

**Q3: Explain the difference between readinessProbe and livenessProbe.**

A: readinessProbe: "Is this pod ready to receive traffic?" If it fails, the pod is removed from the Service endpoints (no traffic sent) but NOT restarted. Use it to handle warm-up time or temporary unavailability (e.g., DB connection pool initializing). livenessProbe: "Is this pod alive?" If it fails, Kubernetes restarts the container. Use it to detect deadlocks or unrecoverable states. Start with readiness probes; add liveness only if your app can truly get stuck in a state that requires a restart.

**Q4: What is the difference between a Deployment and a StatefulSet?**

A: Deployment is for stateless apps — pods are interchangeable, get random names (pod-xyz123), can be scaled freely, volumes are not persistent by default. StatefulSet is for stateful apps (databases, Kafka, ZooKeeper) — pods get stable ordinal names (pod-0, pod-1), stable network identities, ordered startup/shutdown, and persistent volumes that survive pod restarts. Each pod in a StatefulSet has its own PersistentVolumeClaim.

**Q5: How does Kubernetes handle zero-downtime deployments?**

A: Rolling update strategy: Kubernetes gradually replaces old pods with new ones. With `maxUnavailable: 0`, it ensures at least N replicas are always running. With `maxSurge: 1`, it creates one extra pod before terminating an old one. The key requirement is proper readiness probes — Kubernetes only sends traffic to the new pod once it passes the readiness check. Combine with pod disruption budgets (PDB) to protect against simultaneous node drains.

---

# Phase 5 – AWS {#phase-5}

---

## 5.1 Core AWS Services

### Compute

| Service | Use Case | When to Choose |
|---------|---------|---------------|
| EC2 | VMs, full control | Legacy apps, GPU workloads |
| Lambda | Serverless functions | Event-driven, short-lived tasks |
| ECS (Fargate) | Containers without managing nodes | Small/medium container workloads |
| EKS | Managed Kubernetes | Complex container orchestration |
| Elastic Beanstalk | PaaS, deploy apps without infra | Quick deployments, small teams |

### Storage

| Service | Type | Use Case |
|---------|------|---------|
| S3 | Object storage | Files, backups, static assets, data lake |
| EBS | Block storage | EC2 root volumes, databases |
| EFS | Network filesystem | Shared storage across EC2 instances |
| RDS | Managed SQL | PostgreSQL, MySQL, MariaDB, Oracle |
| Aurora | AWS-native SQL | High performance, auto-scaling storage |
| DynamoDB | NoSQL key-value | Massive scale, single-digit ms latency |
| ElastiCache | Managed Redis/Memcached | Caching, sessions |

### Networking

```
VPC (Virtual Private Cloud) architecture:

┌─────────────────────────────────────────┐
│                  VPC                    │
│    ┌──────────────┐  ┌──────────────┐   │
│    │ Public Subnet│  │Private Subnet│   │
│    │   (AZ-a)     │  │   (AZ-a)     │   │
│    │  ┌─────────┐ │  │  ┌─────────┐ │   │
│    │  │   ALB   │ │  │  │  EC2/   │ │   │
│    │  │(Load Bl)│─┼──┼─▶│  ECS    │ │   │
│    │  └─────────┘ │  │  └─────────┘ │   │
│    └──────────────┘  │  ┌─────────┐ │   │
│    ┌──────────────┐  │  │   RDS   │ │   │
│    │ Public Subnet│  │  └─────────┘ │   │
│    │   (AZ-b)     │  └──────────────┘   │
│    └──────────────┘  ┌──────────────┐   │
│                      │Private Subnet│   │
│  Internet Gateway    │   (AZ-b)     │   │
│  NAT Gateway         └──────────────┘   │
└─────────────────────────────────────────┘
```

### IAM Best Practices

```json
// Principle of least privilege: only grant what's needed
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-app-bucket"
    }
  ]
}

// Use IAM Roles for EC2/ECS/Lambda — NEVER hardcode credentials
// Roles: attached to AWS resources, rotated automatically
// Users: for humans, with MFA
// Policies: attached to roles/users/groups
```

### Lambda + API Gateway (Serverless Pattern)

```java
// Spring Cloud Function for Lambda
@Bean
public Function<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> processOrder() {
    return request -> {
        String body = request.getBody();
        CreateOrderRequest orderRequest = objectMapper.readValue(body, CreateOrderRequest.class);

        Order order = orderService.createOrder(orderRequest);

        APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent();
        response.setStatusCode(201);
        response.setBody(objectMapper.writeValueAsString(order));
        response.setHeaders(Map.of("Content-Type", "application/json"));
        return response;
    };
}
```

### SQS vs SNS

```
SQS (Simple Queue Service) – Point-to-point:
Producer ──▶ Queue ──▶ One Consumer
- Pull-based
- Message stored until consumed (up to 14 days)
- Dead-letter queue for failed messages
- Standard (at-least-once) or FIFO (exactly-once, ordered)

SNS (Simple Notification Service) – Pub/Sub fan-out:
                    ──▶ SQS Queue A (email service)
Publisher ──▶ Topic ──▶ SQS Queue B (SMS service)
                    ──▶ Lambda (analytics)
                    ──▶ HTTP endpoint

Fan-out pattern: SNS → multiple SQS queues
- Decouple publishers from subscribers
- Each subscriber independently processes at its own pace
```

### CloudWatch and Observability

```java
// Custom metrics from Spring Boot to CloudWatch
@Component
public class BusinessMetricsPublisher {

    private final CloudWatchAsyncClient cloudWatch;

    public void recordOrderPlaced(String region, double amount) {
        cloudWatch.putMetricData(PutMetricDataRequest.builder()
            .namespace("MyApp/Orders")
            .metricData(
                MetricDatum.builder()
                    .metricName("OrdersPlaced")
                    .value(1.0)
                    .unit(StandardUnit.COUNT)
                    .dimensions(
                        Dimension.builder().name("Region").value(region).build()
                    )
                    .build(),
                MetricDatum.builder()
                    .metricName("OrderValue")
                    .value(amount)
                    .unit(StandardUnit.NONE)
                    .build()
            )
            .build());
    }
}
```

---

## Phase 5 – Interview Questions & Answers

**Q1: When would you choose Aurora over RDS PostgreSQL?**

A: Aurora PostgreSQL is compatible with standard PostgreSQL but has a different storage engine. Choose Aurora when: you need read replicas with <10ms replication lag (vs RDS's async replication), you want Aurora Serverless for variable/bursty workloads, you need automated storage scaling up to 128TB, or you need Global Database for multi-region active-passive. RDS is cheaper and simpler for smaller workloads and when Aurora's features aren't needed.

**Q2: Explain the difference between ALB and NLB.**

A: ALB (Application Load Balancer) operates at Layer 7 (HTTP/HTTPS). It can route based on URL path, headers, host, query string. Supports WebSocket, gRPC, SSL termination, authentication (Cognito). Use for web apps and microservices. NLB (Network Load Balancer) operates at Layer 4 (TCP/UDP). Handles millions of requests per second with ultra-low latency. Static IP per AZ. Use for gaming, IoT, financial trading apps, or when you need to preserve client IP.

**Q3: What is the difference between Security Groups and NACLs?**

A: Security Groups are stateful, instance-level firewalls. If you allow inbound traffic on port 443, the response is automatically allowed outbound (stateful). They only have allow rules. NACLs (Network Access Control Lists) are stateless, subnet-level firewalls. You must explicitly allow both inbound and outbound traffic. They support both allow and deny rules. NACLs are evaluated before Security Groups.

**Q4: How would you design a serverless image processing pipeline on AWS?**

A: S3 (upload trigger) → Lambda (validate + resize → save thumbnail to S3) → SNS notification → SQS → Lambda (send email/push notification). Key considerations: Lambda timeout (15 min max), payload size limit (6MB sync, 256MB async), use S3 presigned URLs for direct browser uploads to avoid routing through Lambda, use Lambda Layers for shared dependencies (ImageMagick), set appropriate concurrency limits to avoid overwhelming downstream services.

---

# Phase 6 – Distributed Systems + Consistency Models + Consensus {#phase-6}

---

## 6.1 Consistency Models

### The Spectrum of Consistency

```
Strongest ────────────────────────────────────────▶ Weakest
Linearizability → Sequential → Causal → Eventual

Linearizability (Strong Consistency):
- Operations appear instantaneous and in real-time order
- Every read sees the most recent write
- Cost: high latency (all replicas must agree before responding)
- Examples: etcd, ZooKeeper, Google Spanner

Sequential Consistency:
- All operations in program order, but no real-time guarantee
- Writes from one process appear in order to all others
- Cost: lower latency than linearizability

Causal Consistency:
- Causally related operations are seen in the same order by all nodes
- Concurrent operations may be seen in different orders
- "If A caused B, everyone sees A before B"
- Examples: MongoDB causal sessions

Eventual Consistency:
- Given no new writes, all replicas eventually converge
- Reads may return stale data
- Cost: lowest latency
- Examples: DNS, DynamoDB default, Cassandra default
```

### Conflict Resolution in Eventual Consistency

```
Strategies:
1. Last-Write-Wins (LWW): highest timestamp wins
   - Risk: clock skew can lose writes
   - Use: when data loss is acceptable

2. Vector Clocks: track causal history per replica
   - Detect concurrent writes
   - Application resolves conflicts

3. CRDTs (Conflict-free Replicated Data Types):
   - Data structures designed to merge automatically
   - Counter CRDT: increment only, merge = max per replica
   - Set CRDT: grow-only set, merge = union
   - Examples: Redis CRDT (enterprise), Riak

4. Application-level merge: let the application define merge logic
   - Shopping cart: union of items
   - Counter: sum all replica counts
```

---

## 6.2 Consensus Algorithms

### Raft – Understandable Consensus

**Real-world analogy:** Raft is like a company with a CEO (leader) and employees (followers). All decisions go through the CEO. If the CEO goes silent, employees elect a new one.

```
Raft States:
- Follower: default state, waits for heartbeats from leader
- Candidate: when follower times out, becomes candidate and requests votes
- Leader: receives all writes, replicates to followers

Leader Election:
1. Follower timeout expires → becomes Candidate
2. Increments term, votes for itself, requests votes from others
3. Receives majority → becomes Leader for this term
4. Sends heartbeats to prevent new elections

Log Replication:
1. Client sends write to Leader
2. Leader appends to its log
3. Leader sends AppendEntries RPC to all Followers
4. Followers append to their log and acknowledge
5. Leader commits when majority acknowledge (N/2 + 1)
6. Leader tells followers to commit
7. Leader responds to client

Safety guarantees:
- Election Safety: at most one leader per term
- Leader Completeness: leader has all committed entries from previous terms
- State Machine Safety: all nodes apply same entries in same order
```

### Distributed Clocks

**Problem:** Distributed systems have no global clock. `System.currentTimeMillis()` on different machines can drift.

```
Solutions:

1. Logical Clocks (Lamport Timestamps):
   - Each process has a counter
   - Increment on local event
   - On send: include counter value
   - On receive: max(local, received) + 1
   - Establishes causal ordering (A→B means A.time < B.time)
   - But: concurrent events have arbitrary order

2. Vector Clocks:
   - Each process has a vector of counters [p1, p2, p3]
   - Increment own counter on local event
   - On send: include full vector
   - On receive: element-wise max, then increment own counter
   - Can detect concurrent events: neither A→B nor B→A

3. Hybrid Logical Clocks (HLC):
   - Combines physical time + logical counter
   - Always moves forward, close to physical time
   - Used by: CockroachDB, YugabyteDB

4. Google TrueTime:
   - Atomic clocks + GPS receivers in datacenters
   - Returns [earliest, latest] time interval
   - Spanner waits out uncertainty before commit
   - Enables external consistency (strict serializability)
```

### Distributed Transactions

```java
// Two-Phase Commit (2PC) - the classic approach
// Phase 1 (Prepare): Coordinator asks all participants if they can commit
// Phase 2 (Commit/Abort): All said yes → Coordinator sends commit; any said no → abort

// Problems with 2PC:
// - Blocking: if coordinator crashes after prepare, participants are stuck
// - Single point of failure at coordinator
// - High latency (2 round trips)

// Preferred: Saga pattern (see Phase 2)
// - No locking
// - Compensating transactions instead of rollback
// - Better availability at the cost of intermediate inconsistency

// Three-Phase Commit (3PC): adds CanCommit phase to eliminate blocking
// but still has issues with network partitions — rarely used in practice
```

---

## 6.3 Common Distributed System Problems

### Leader Election in Practice

```java
// Using ZooKeeper (via Apache Curator) for leader election
public class LeaderElectionService {

    private final LeaderSelector leaderSelector;

    public LeaderElectionService(CuratorFramework client, String leaderPath) {
        this.leaderSelector = new LeaderSelector(client, leaderPath, new LeaderSelectorListenerAdapter() {
            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                log.info("I am the leader now");
                try {
                    scheduledTask.run();
                    Thread.currentThread().join(); // Hold leadership until interrupted
                } catch (InterruptedException e) {
                    log.info("Lost leadership or interrupted");
                }
            }
        });
        leaderSelector.autoRequeue(); // Re-enter queue after losing leadership
    }
}
```

### Idempotency

```java
// Every write operation should be idempotent in distributed systems
// Use idempotency keys for payment and order processing

@PostMapping("/payments")
public ResponseEntity<Payment> processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    // Check if this request was already processed
    Optional<Payment> existing = idempotencyStore.get(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get()); // Return same result
    }

    Payment payment = paymentService.process(request);

    // Store result mapped to idempotency key (with TTL)
    idempotencyStore.store(idempotencyKey, payment, Duration.ofDays(7));

    return ResponseEntity.status(HttpStatus.CREATED).body(payment);
}
```

---

## Phase 6 – Interview Questions & Answers

**Q1: Explain the CAP theorem and how it applies to real systems.**

A: CAP states that a distributed system can provide at most two of: Consistency (all nodes see same data), Availability (system always responds), Partition Tolerance (works despite network splits). Since network partitions WILL happen, the real choice is CP or AP. During a partition: CP systems (ZooKeeper, etcd, HBase) reject requests on minority partition to avoid serving stale data. AP systems (Cassandra, DynamoDB, CouchDB) continue serving potentially stale data. After the partition heals, AP systems eventually converge. Most real systems offer tunable consistency — Cassandra's QUORUM vs ONE read levels.

**Q2: What is the split-brain problem and how do you prevent it?**

A: Split-brain occurs when a network partition separates a cluster into two groups, each believing they are the active leader. Both accept writes, leading to divergent state that's hard to merge. Prevention: quorum-based writes (only the partition with N/2 + 1 nodes can accept writes), STONITH (Shoot The Other Node In The Head — one partition is forcibly terminated), fencing tokens (each leader gets a monotonically increasing token; writes with lower tokens are rejected).

**Q3: What is the difference between Paxos and Raft?**

A: Both achieve consensus in a distributed system. Paxos was the original algorithm — provably correct but notoriously difficult to understand and implement in practice. It has a two-phase protocol (prepare + accept) and has many variants (Multi-Paxos, Fast Paxos). Raft was designed specifically for understandability. It decomposes consensus into leader election, log replication, and safety. It has stronger leadership invariants that make it easier to reason about. Real-world usage: etcd uses Raft, ZooKeeper uses ZAB (similar to Paxos). Raft is the practical choice for new systems.

**Q4: What is vector clock and when would you use it?**

A: Vector clocks track causal relationships between events in a distributed system. Each node maintains a counter for every other node. When a node sends a message, it includes its vector. The receiver takes the element-wise maximum. By comparing vectors, you can determine if event A happened before B, B before A, or they are concurrent. Use when you need to detect write conflicts in a multi-master database or detect causality in an event log. The cost is O(N) metadata per message (N = number of nodes).

**Q5: Explain idempotency and why it's critical in distributed systems.**

A: An operation is idempotent if calling it multiple times produces the same result as calling it once. In distributed systems, failures are partial — a request might be processed but the response lost, causing the client to retry. Without idempotency, retries cause duplicate effects (double charges, duplicate orders). Strategies: use idempotency keys (client-generated UUID) mapped to results in a store; make operations naturally idempotent (SET is idempotent, INCREMENT is not); use conditional writes (`UPDATE SET x WHERE version = N`).

---

# Phase 7 – AI Agents + Spring AI + Modern Architectures {#phase-7}

---

## 7.1 AI Agents Overview

### What is an AI Agent?

**Real-world analogy:** An AI agent is like a smart intern with access to tools (browser, calculator, database). You give them a goal, and they plan and execute steps using available tools to achieve it, adapting when things don't work as expected.

```
Agent Components:
┌─────────────────────────────────────────────────┐
│                   AI Agent                      │
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌────────────┐  │
│  │ Planner  │──▶│  Tool    │──▶│  Memory    │  │
│  │ (LLM)   │   │ Executor │   │ (Context)  │  │
│  └──────────┘   └──────────┘   └────────────┘  │
│         ▲                              │        │
│         └──────── Observation ─────────┘        │
└─────────────────────────────────────────────────┘

Tools: Web search, code interpreter, database query, API calls
Memory: Short-term (context window), Long-term (vector DB), Episodic
Planning: ReAct (Reason + Act), Chain-of-Thought, Tree of Thought
```

### ReAct Pattern (Reasoning + Acting)

```
Loop:
1. Thought: "I need to find the user's order history to answer this question"
2. Action: query_database(user_id=123, table="orders")
3. Observation: [order records returned]
4. Thought: "I have the data, now I can summarize it"
5. Action: respond_to_user(summary)
6. → Done

This loop continues until the agent reaches a final answer or max iterations.
```

---

## 7.2 Spring AI

### Setup and Configuration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
```

### Basic Chat Completion

```java
@Service
public class CustomerSupportService {

    private final ChatClient chatClient;

    public CustomerSupportService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("""
                You are a helpful customer support agent for an e-commerce platform.
                Be concise, friendly, and accurate. If you cannot answer, say so.
                """)
            .build();
    }

    public String handleQuery(String userId, String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }

    // Streaming response
    public Flux<String> handleQueryStreaming(String question) {
        return chatClient.prompt()
            .user(question)
            .stream()
            .content();
    }
}
```

### Tool Calling (Function Calling)

```java
// Define tools the AI can call
@Service
public class OrderTools {

    @Autowired
    private OrderRepository orderRepository;

    @Tool(description = "Get order status by order ID")
    public OrderStatus getOrderStatus(String orderId) {
        return orderRepository.findById(orderId)
            .map(order -> new OrderStatus(order.getStatus(), order.getTrackingNumber()))
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    @Tool(description = "Get list of recent orders for a customer")
    public List<OrderSummary> getRecentOrders(
            @ToolParam(description = "Customer ID") String customerId,
            @ToolParam(description = "Number of recent orders to return") int limit) {
        return orderRepository.findRecentByCustomerId(customerId, limit);
    }

    @Tool(description = "Cancel an order if it hasn't shipped yet")
    public CancelResult cancelOrder(String orderId, String reason) {
        return orderService.cancel(orderId, reason);
    }
}

// Register tools with the agent
@Service
public class SupportAgent {

    private final ChatClient chatClient;
    private final OrderTools orderTools;

    public SupportAgent(ChatClient.Builder builder, OrderTools orderTools) {
        this.orderTools = orderTools;
        this.chatClient = builder
            .defaultSystem("You are a customer support agent. Use tools to look up real data.")
            .defaultTools(orderTools)  // register tools
            .build();
    }

    public String handleRequest(String customerId, String question) {
        return chatClient.prompt()
            .system(s -> s.param("customerId", customerId))
            .user(question)
            .call()
            .content();
    }
}
```

### RAG – Retrieval-Augmented Generation

**Real-world analogy:** Instead of asking the LLM to recall everything it learned in training, you give it a stack of relevant documents first and say "answer using only these". It's like giving an open-book exam rather than a closed-book one.

```java
// Step 1: Ingest documents into vector store
@Service
public class DocumentIngestionService {

    @Autowired
    private VectorStore vectorStore;

    @Autowired
    private DocumentReader pdfReader;

    public void ingestProductManual(String filePath) {
        List<Document> documents = new TokenTextSplitter()
            .apply(new PagePdfDocumentReader(filePath).get());

        // Add metadata for filtering
        documents.forEach(doc ->
            doc.getMetadata().put("source", "product-manual"));

        vectorStore.add(documents); // Embeds and stores in pgvector
    }
}

// Step 2: Build RAG query flow
@Service
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String answerQuestion(String question) {
        // Retrieve relevant documents based on semantic similarity
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withSimilarityThreshold(0.7)
                .withFilterExpression("source == 'product-manual'")
        );

        // Build context from retrieved documents
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n---\n\n"));

        // Ask LLM with context
        return chatClient.prompt()
            .system("""
                Answer the user's question ONLY using the provided context.
                If the answer is not in the context, say "I don't have that information."
                Context:
                {context}
                """)
            .system(s -> s.param("context", context))
            .user(question)
            .call()
            .content();
    }
}
```

### Conversation Memory

```java
@Service
public class ConversationService {

    private final ChatClient chatClient;
    private final ChatMemoryRepository chatMemoryRepository;

    public ConversationService(ChatClient.Builder builder, ChatMemoryRepository memRepo) {
        this.chatMemoryRepository = memRepo;
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant with memory of past conversations.")
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(
                    new CassandraChatMemory(memRepo),  // persistent memory
                    "default",
                    10  // last 10 messages
                )
            )
            .build();
    }

    public String chat(String conversationId, String message) {
        return chatClient.prompt()
            .user(message)
            .advisors(a -> a.param(CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
            .call()
            .content();
    }
}
```

### Structured Output Extraction

```java
// Extract structured data from unstructured text
public record ProductReview(
    String productName,
    int rating,          // 1-5
    String sentiment,    // POSITIVE, NEGATIVE, NEUTRAL
    List<String> pros,
    List<String> cons
) {}

@Service
public class ReviewAnalysisService {

    private final ChatClient chatClient;

    public ProductReview analyzeReview(String reviewText) {
        return chatClient.prompt()
            .user("Analyze this product review and extract structured data:\n\n" + reviewText)
            .call()
            .entity(ProductReview.class);  // Spring AI handles JSON schema generation
    }
}
```

---

## 7.3 Modern Architectures

### Agentic Architecture Patterns

```
Multi-Agent System:

User Query
    │
    ▼
Orchestrator Agent
    │
    ├──▶ Research Agent (web search, knowledge base)
    │
    ├──▶ Code Agent (writes, runs, tests code)
    │
    ├──▶ Data Agent (SQL queries, data analysis)
    │
    └──▶ Summary Agent (synthesizes results)

Patterns:
- Sequential: agents run one after another
- Parallel: agents run concurrently, results merged
- Hierarchical: manager agents delegate to worker agents
- Competitive: multiple agents solve same problem, best answer wins
```

### Vector Databases

| Database | Type | Strengths |
|---------|------|----------|
| pgvector | PostgreSQL extension | Familiar SQL, ACID, no extra infra |
| Pinecone | Managed cloud | Fully managed, massive scale |
| Weaviate | Open source | Rich filtering, GraphQL API |
| Chroma | Open source | Simple, great for prototyping |
| Qdrant | Open source | High performance, Rust-based |

```java
// Choosing embedding dimensions:
// text-embedding-3-small: 1536 dimensions (fast, cheap)
// text-embedding-3-large: 3072 dimensions (more accurate)

// Index types in pgvector:
// IVFFLAT: faster to build, slightly less accurate (good for large datasets)
// HNSW: slower to build, better recall (good for smaller datasets, production)

// pgvector index creation
// CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops)
// WITH (m = 16, ef_construction = 64);
```

### Modern Spring Boot + AI Architecture

```
┌──────────────────────────────────────────────────────┐
│                   API Layer                          │
│  REST / GraphQL / gRPC / WebSocket                   │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│               Application Layer                      │
│  Spring Boot Services + Spring AI                    │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │
│  │  RAG     │  │  Agents  │  │  Traditional CRUD  │  │
│  │ Service  │  │  Service │  │     Services       │  │
│  └──────────┘  └──────────┘  └────────────────────┘  │
└──────────────────────┬───────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│               Data Layer                             │
│  PostgreSQL + pgvector │ Redis │ Kafka │ S3          │
└──────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────┐
│               AI Provider Layer                      │
│  OpenAI │ Anthropic Claude │ Ollama (local models)   │
└──────────────────────────────────────────────────────┘
```

---

## Phase 7 – Interview Questions & Answers

**Q1: What is RAG and why is it preferred over fine-tuning?**

A: RAG (Retrieval-Augmented Generation) retrieves relevant documents at query time and includes them in the LLM prompt. Fine-tuning bakes knowledge into model weights through training. RAG is preferred when: data changes frequently (you'd have to retrain to update fine-tuned knowledge), you need the model to cite sources, the dataset is proprietary and you don't want it embedded in model weights, or fine-tuning is too expensive. Fine-tuning is better when you need to change model behavior/style, work with a specific domain's terminology, or when inference latency of RAG (extra retrieval step) is a concern.

**Q2: What is semantic search and how does it differ from keyword search?**

A: Keyword search (Elasticsearch BM25) finds documents containing the exact terms. It fails for synonyms, paraphrasing, or conceptually related ideas. Semantic search converts text to embedding vectors (floating-point representations) and finds documents with similar vectors using distance metrics (cosine similarity, dot product). "Good shoes for running" matches "athletic footwear for joggers" even with no shared keywords. The tradeoff: embedding models add latency and cost; hybrid search (keyword + semantic) often outperforms either alone.

**Q3: What are the risks of using AI in production systems?**

A: (1) Hallucination: LLMs confidently produce wrong information — mitigate with RAG, citation requirements, and human review for high-stakes outputs. (2) Prompt injection: malicious user input that overrides system instructions — sanitize inputs, use structured tools instead of free-form execution. (3) Data privacy: user queries sent to third-party LLM providers — use self-hosted models (Ollama) for sensitive data. (4) Cost: LLM calls are expensive at scale — cache common queries, use smaller models for simple tasks. (5) Latency: LLM inference is slow — stream responses, pre-warm caches.

**Q4: Explain the difference between an AI tool call and function calling.**

A: They're the same concept with different terminology. "Function calling" (OpenAI) and "tool use" (Anthropic) let the LLM generate structured JSON describing which function to call and what arguments to pass, rather than producing free-form text. Your application executes the actual function and returns the result to the LLM, which then incorporates it into the response. This is how AI agents interact with external systems — the LLM decides WHAT to call; your code handles the actual execution. Spring AI's `@Tool` annotation automates schema generation from Java method signatures.

**Q5: How would you implement guardrails for a production AI chatbot?**

A: (1) Input guardrails: content moderation API (OpenAI Moderation, AWS Comprehend) to block harmful inputs before they reach the LLM. (2) System prompt hardening: explicit instructions to refuse off-topic requests, and instructions that are resistant to prompt injection. (3) Output guardrails: validate structured outputs against schemas, scan responses for PII before returning. (4) Rate limiting and cost controls: per-user request limits, token budget limits. (5) Logging and monitoring: log all inputs/outputs for auditing (with appropriate PII masking), monitor for unusual patterns. (6) Human-in-the-loop: for high-stakes decisions (financial, medical), require human approval before acting on AI recommendations.

---

# Master Revision Cheat Sheet

## Phase-by-Phase Quick Reference

### Phase 1 – SQL & Redis
- **B-Tree index**: equality + range queries (default)
- **Composite index**: leftmost prefix rule
- **MVCC**: readers don't block writers, dead tuples need VACUUM
- **WAL**: crash recovery + replication mechanism
- **Redis LRU**: `volatile-lru` for session stores
- **Outbox in Redis**: Streams > Pub/Sub (durable, consumer groups)

### Phase 2 – Kafka & Events
- **Ordering**: guaranteed within a partition, not across
- **Consumer group**: 1 partition → 1 consumer per group (scale = partition count)
- **Delivery**: `acks=all` + idempotent producer → exactly-once per-producer
- **Outbox pattern**: solve dual-write with same-transaction outbox + relay
- **Saga**: choreography (event-driven) vs orchestration (centralized coordinator)

### Phase 3 – System Design
- **CAP**: Pick CP or AP (P is always required in distributed systems)
- **Consistent hashing**: minimize reshuffling on node add/remove
- **Sharding**: hash (even distribution) vs range (range query friendly)
- **Cache stampede**: mutex lock, probabilistic early expiration, background refresh
- **Zero-downtime migration**: expand-contract pattern

### Phase 4 – Docker & Kubernetes
- **Layers**: order Dockerfile for max cache reuse (COPY pom.xml before COPY src)
- **Multi-stage**: only ship runtime artifacts, never build tools
- **readiness vs liveness**: readiness = traffic gating; liveness = restart trigger
- **HPA**: scales on CPU/memory/custom metrics — requires resource requests set
- **StatefulSet**: for stateful apps needing stable identity + persistent volumes

### Phase 5 – AWS
- **SG**: stateful, instance level; NACL: stateless, subnet level
- **ALB**: Layer 7, path routing; NLB: Layer 4, ultra-low latency
- **IAM Roles**: always over hardcoded credentials
- **S3 presigned URL**: direct browser → S3 upload (bypass Lambda payload limit)
- **SQS FIFO**: exactly-once, ordered per message group ID

### Phase 6 – Distributed Systems
- **Linearizability**: strongest — every read sees latest write
- **Eventual consistency**: weakest — replicas converge after writes stop
- **Raft**: leader election + log replication — understandable consensus
- **Split-brain**: prevent with quorum writes (N/2 + 1)
- **Idempotency**: idempotency-key → stored result mapping

### Phase 7 – AI Agents & Spring AI
- **RAG vs Fine-tuning**: RAG for changing data + citations; fine-tuning for behavior/style
- **Tool calling**: LLM generates JSON → your code executes → result returned to LLM
- **Vector similarity**: cosine (normalized), dot product (raw magnitude matters)
- **Hallucination**: mitigate with RAG, citations, structured output validation
- **Guardrails**: input moderation → system prompt hardening → output validation → logging

---

## Common Interview Patterns

### "How would you design X?" Framework
1. **Clarify requirements** (functional + non-functional: scale, latency, availability)
2. **Estimate scale** (users, requests/sec, data size, read/write ratio)
3. **High-level design** (identify major components, data flow)
4. **Data model** (schema, indexing strategy)
5. **Deep dive** (bottlenecks, caching, scaling, failure modes)
6. **Trade-offs** (what you'd do differently with more time)

### "Explain X" Framework
1. **What it is** (one sentence definition)
2. **Real-world analogy** (makes it memorable)
3. **How it works** (internals, key steps)
4. **When to use it** (trade-offs vs alternatives)
5. **Gotchas** (common mistakes, edge cases)

### System Design Numbers to Remember
| Metric | Value |
|--------|-------|
| L1 cache | ~1 ns |
| L2 cache | ~10 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 µs |
| Network round-trip (same DC) | ~0.5 ms |
| Network round-trip (cross-region) | ~150 ms |
| PostgreSQL query (indexed) | ~1 ms |
| Redis GET | ~0.1 ms |
| Kafka producer (async) | ~5 ms |
| LLM API call (GPT-4o) | ~500ms–3s |
