# Architecture Topics Every Java Developer Should Learn

A senior engineer understands not just how to write code, but how systems behave at scale. This guide covers the core architecture areas with practical depth for Java developers.

---

## Table of Contents

1. [Databases](#1-databases)
2. [Caching](#2-caching)
3. [Messaging Systems](#3-messaging-systems)
4. [Cloud Computing](#4-cloud-computing)
5. [Containers](#5-containers)
6. [Reliability](#6-reliability)
7. [Observability](#7-observability)

---

## 1. Databases

### 1.1 Indexing

**Concept:**
An index is a separate data structure that the database maintains to speed up data retrieval. Think of it like a book's index — instead of reading every page to find a topic, you look it up in the index and jump directly to the right page.

**Real-world analogy:**
Without an index, a query `SELECT * FROM users WHERE email = 'x@y.com'` scans every row (Full Table Scan). With an index on `email`, the DB jumps directly to the matching row via a B-Tree or Hash structure.

**Types of Indexes:**

| Index Type | Best For | How It Works |
|---|---|---|
| B-Tree (default) | Range queries, ORDER BY, equality | Balanced tree, O(log n) lookup |
| Hash Index | Exact equality lookups | Hash map, O(1) avg lookup |
| Composite Index | Multi-column WHERE clauses | Leftmost prefix rule applies |
| Covering Index | Queries only using indexed columns | All data in index, no table hit |
| Partial Index | Filtering a subset of rows | Only indexes rows matching a condition |
| Full-Text Index | LIKE '%keyword%', text search | Inverted index on words |

**Java / Spring Boot context:**
```java
// JPA/Hibernate index definition
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_email", columnList = "email"),
    @Index(name = "idx_name_dob", columnList = "last_name, date_of_birth")
})
public class User {
    @Id
    private Long id;
    private String email;
    private String lastName;
    private LocalDate dateOfBirth;
}
```

**Key Rules:**
- Index columns used in `WHERE`, `JOIN ON`, `ORDER BY`, `GROUP BY`
- Too many indexes slow down `INSERT`/`UPDATE`/`DELETE` (indexes must be updated)
- The **leftmost prefix rule** applies to composite indexes: index on `(a, b, c)` helps queries filtering on `a`, `a+b`, or `a+b+c` — but NOT `b` alone
- **Cardinality matters**: high-cardinality columns (email, UUID) benefit most from indexes

**Common Interview Questions:**
- Why does `SELECT *` with a composite index sometimes not use the index? (Leftmost prefix not matched)
- What is a covering index? (Index contains all columns the query needs — no table lookup required)
- When would you NOT add an index? (Small tables, write-heavy tables, low-cardinality columns like boolean)

---

### 1.2 Query Optimization

**Concept:**
Query optimization is the process of writing and structuring SQL so the database executes it as efficiently as possible. The database engine has a query planner/optimizer that chooses the execution plan.

**EXPLAIN / EXPLAIN ANALYZE:**
```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 100;

-- Look for:
-- Seq Scan (bad for large tables) → means no index used
-- Index Scan (good)
-- Index Only Scan (best — covering index)
-- Nested Loop / Hash Join / Merge Join
```

**Common Optimization Techniques:**

| Problem | Solution |
|---|---|
| Full table scan | Add appropriate index |
| N+1 query problem | Use JOIN or batch fetch (`@EntityGraph`, `JOIN FETCH`) |
| SELECT * fetching too much data | Select only needed columns |
| Too many JOINs on large tables | Denormalize or use caching |
| Slow COUNT(*) | Use approximate counts or cache the result |
| Unindexed ORDER BY | Add index on sort column |

**N+1 Query Problem in Java (Critical Interview Topic):**
```java
// BAD: N+1 — 1 query for orders + N queries for each customer
List<Order> orders = orderRepository.findAll(); // 1 query
for (Order o : orders) {
    System.out.println(o.getCustomer().getName()); // N queries (lazy load)
}

// GOOD: Single JOIN FETCH query
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();

// GOOD: Using @EntityGraph
@EntityGraph(attributePaths = {"customer"})
List<Order> findAll();
```

**Query Optimization Checklist:**
1. Run `EXPLAIN ANALYZE` and look for Seq Scans on large tables
2. Check for N+1 queries in application logs (enable `spring.jpa.show-sql=true`)
3. Use pagination (`LIMIT/OFFSET` or keyset pagination) for large result sets
4. Avoid functions on indexed columns in WHERE clause: `WHERE YEAR(created_at) = 2024` prevents index use — use `WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'`

---

### 1.3 Replication

**Concept:**
Replication copies data from one database server (primary/master) to one or more servers (replicas/slaves) in real-time or near-real-time.

**Real-world analogy:**
A primary database is the source of truth. Replicas are read-only copies that handle read traffic, reducing load on the primary.

**Types of Replication:**

| Type | Description | Trade-off |
|---|---|---|
| Synchronous | Primary waits for replica to confirm write | Strong consistency, higher write latency |
| Asynchronous | Primary writes without waiting for replica | Lower latency, risk of data loss on failure |
| Semi-synchronous | Primary waits for at least one replica | Balance between the two |

**Replication Architecture:**
```
Write Traffic          Read Traffic
     |                    |
  [Primary] ──────► [Replica 1]
      │              [Replica 2]
      └──────────►   [Replica 3]
```

**Use Cases:**
- **Read scaling**: Route SELECT queries to replicas, writes to primary
- **High availability**: Promote replica to primary if primary fails (failover)
- **Disaster recovery**: Geographic replicas in different regions
- **Reporting**: Run heavy analytics queries on replicas without impacting production

**Spring Boot with read/write routing:**
```java
// Route reads to replica, writes to primary
@Transactional(readOnly = true)
public List<Product> getAllProducts() {
    return productRepository.findAll(); // goes to replica
}

@Transactional
public Product createProduct(Product product) {
    return productRepository.save(product); // goes to primary
}
```

**Replication Lag:**
The delay between a write on primary and it appearing on the replica. Critical to understand — a user who writes data and immediately reads it from a replica might see stale data. Solutions: read-your-own-writes consistency, stick the user session to the primary after writes.

---

### 1.4 Partitioning

**Concept:**
Partitioning splits a single large table into smaller, more manageable pieces stored within the same database. The database engine transparently routes queries to the right partition.

**Types of Partitioning:**

| Type | How it Works | Example |
|---|---|---|
| Range Partitioning | Rows split by value ranges | Orders by year: 2022, 2023, 2024 partitions |
| List Partitioning | Rows split by discrete values | Orders by region: US, EU, ASIA partitions |
| Hash Partitioning | Rows split by hash of a column | Distributes evenly by user_id hash |
| Composite | Combination of above | Range by year, then hash within year |

**PostgreSQL Example:**
```sql
-- Create partitioned table
CREATE TABLE orders (
    id BIGINT,
    created_at DATE,
    amount DECIMAL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

**Benefits:**
- **Partition pruning**: Queries with a WHERE on the partition key only scan relevant partitions
- **Maintenance**: Archive/drop old partitions cheaply (instead of slow DELETE on millions of rows)
- **Performance**: Index sizes are smaller per partition

**Partitioning vs Sharding:**
- Partitioning = multiple partitions on the **same database server**
- Sharding = data split across **multiple database servers**

---

### 1.5 Sharding

**Concept:**
Sharding is horizontal scaling of a database — splitting data across multiple independent database servers (shards), where each shard holds a subset of the total data.

**Real-world analogy:**
Instead of one giant warehouse storing all inventory (which becomes too large for one building), you have 4 warehouses. Products A-G go to warehouse 1, H-P to warehouse 2, etc.

**Sharding Strategies:**

| Strategy | How | Pros | Cons |
|---|---|---|---|
| Range-based | Shard by value range (user_id 1–1M → shard1) | Simple, range queries easy | Hotspots if distribution uneven |
| Hash-based | `shard = hash(user_id) % num_shards` | Even distribution | Range queries hit all shards |
| Directory-based | Lookup table maps key to shard | Flexible | Lookup table is a bottleneck |
| Geographic | Region-based assignment | Latency optimization | Uneven data if users clustered |

**Architecture:**
```
Application Layer
      │
  [Shard Router / Middleware]
  /       |       |       \
Shard1  Shard2  Shard3  Shard4
(1-25%) (25-50%) (50-75%) (75-100%)
```

**Challenges (Critical for Interviews):**
- **Cross-shard queries**: JOINs across shards are expensive or impossible — must be handled at application level
- **Rebalancing**: Adding a new shard requires moving data (consistent hashing minimizes this)
- **Distributed transactions**: Maintaining ACID across shards is very hard (use eventual consistency or sagas)
- **Hot shards**: One shard gets disproportionate traffic (use hash-based sharding or virtual shards)

**Java / Application-level sharding example:**
```java
public String getShardKey(Long userId) {
    int shardIndex = (int) (userId % numberOfShards);
    return "shard_" + shardIndex;
}

public DataSource getDataSource(Long userId) {
    String shardKey = getShardKey(userId);
    return dataSourceMap.get(shardKey);
}
```

---

## 2. Caching

### 2.1 Redis

**Concept:**
Redis (Remote Dictionary Server) is an in-memory key-value data store, used as a cache, message broker, and session store. Data lives in RAM, making reads/writes orders of magnitude faster than disk-based databases.

**Data Structures:**

| Structure | Use Case | Example Command |
|---|---|---|
| String | Simple key-value, counters | `SET user:1:name "John"` |
| Hash | Object with fields | `HSET user:1 name "John" age 30` |
| List | Queues, activity feeds | `LPUSH notifications msg1` |
| Set | Unique collections, tags | `SADD user:1:tags "java" "spring"` |
| Sorted Set | Leaderboards, rate limiting | `ZADD leaderboard 1500 "user1"` |
| TTL | Expiring keys | `EXPIRE session:abc 3600` |

**Spring Boot Redis Integration:**
```java
// application.properties
spring.data.redis.host=localhost
spring.data.redis.port=6379

// Enable caching
@SpringBootApplication
@EnableCaching
public class Application { }

// Use @Cacheable
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
        // First call: hits DB, stores result in Redis
        // Subsequent calls: returns from Redis
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
        // Evicts cache entry after update
    }

    @CachePut(value = "products", key = "#result.id")
    public Product createProduct(Product product) {
        return productRepository.save(product);
        // Updates cache with new value
    }
}
```

**Redis Eviction Policies (when memory is full):**
- `allkeys-lru`: Evict least recently used keys from all keys
- `volatile-lru`: Evict LRU from keys with TTL set
- `allkeys-random`: Random eviction
- `noeviction`: Return error on write (default)

**Redis Persistence:**
- **RDB (Snapshot)**: Periodic disk snapshots. Fast restarts, risk of data loss between snapshots
- **AOF (Append Only File)**: Logs every write operation. Durable, but larger files, slower restart
- **Hybrid**: Use both for production

---

### 2.2 Cache Aside Pattern (Lazy Loading)

**Concept:**
The application is responsible for loading data into the cache. On a cache miss, the application reads from the DB, then populates the cache. Cache and DB are managed separately.

**Flow:**
```
Read:
  1. App checks cache → HIT? return data
  2. MISS → App reads from DB
  3. App writes result to cache
  4. Returns data to client

Write:
  1. App writes to DB
  2. App invalidates (deletes) cache entry
  3. Next read will repopulate cache (lazy)
```

**Java Implementation:**
```java
@Service
public class UserService {

    @Autowired private RedisTemplate<String, User> redisTemplate;
    @Autowired private UserRepository userRepository;

    public User getUser(Long userId) {
        String cacheKey = "user:" + userId;

        // Step 1: Check cache
        User cached = (User) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached; // Cache HIT
        }

        // Step 2: Cache MISS — read from DB
        User user = userRepository.findById(userId).orElseThrow();

        // Step 3: Populate cache with TTL
        redisTemplate.opsForValue().set(cacheKey, user, Duration.ofMinutes(30));

        return user;
    }

    public void updateUser(User user) {
        userRepository.save(user);
        // Invalidate cache — next read repopulates it
        redisTemplate.delete("user:" + user.getId());
    }
}
```

**Pros:**
- Cache only contains data that is actually requested
- Resilient: if cache fails, app still works (reads from DB)
- Flexible: cache and DB can use different data models

**Cons:**
- Cache miss on first access (cold start penalty)
- Potential for stale data if invalidation fails
- Three round-trips on cache miss (check cache → read DB → write cache)

---

### 2.3 Write Through Cache

**Concept:**
Every write goes to both the cache and the database synchronously. The cache is always up-to-date with the DB.

**Flow:**
```
Write:
  1. App writes to cache
  2. Cache synchronously writes to DB
  3. Both updated before returning success

Read:
  1. App checks cache → always a HIT (if key exists)
  2. Returns data from cache
```

**Comparison:**

| Aspect | Cache Aside | Write Through |
|---|---|---|
| Cache population | Lazy (on read miss) | Eager (on every write) |
| Read performance | Slow on first read (miss) | Always fast (cache always fresh) |
| Write performance | Fast (fire and forget to cache) | Slower (synchronous to cache + DB) |
| Consistency | Can be stale | Always consistent |
| Cache size | Smaller (only hot data) | Larger (all written data cached) |
| Resilience | Works if cache is down | Writes fail if cache is down |

**When to use Write Through:**
- Read-heavy workloads where data is written once and read many times
- When stale reads are unacceptable
- Session stores, user preferences

---

### 2.4 Cache Invalidation

**Concept:**
Cache invalidation is the process of removing or updating cache entries when the underlying data changes. Phil Karlton famously said: *"There are only two hard things in Computer Science: cache invalidation and naming things."*

**Strategies:**

| Strategy | How | When to Use |
|---|---|---|
| TTL-based (Time-to-Live) | Cache entries expire after N seconds | When occasional staleness is acceptable |
| Event-based | Write triggers explicit invalidation | When consistency is critical |
| Write Through | Cache updated on every write | When cache must always be fresh |
| Cache Versioning | Key includes version number | When partial invalidation is needed |

**TTL Strategy:**
```java
// Set TTL when writing to cache
redisTemplate.opsForValue().set(
    "product:" + id,
    product,
    Duration.ofMinutes(10) // Expires after 10 minutes
);
```

**Event-based Invalidation:**
```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onProductUpdated(ProductUpdatedEvent event) {
    redisTemplate.delete("product:" + event.getProductId());
    redisTemplate.delete("product:list"); // Invalidate list cache too
}
```

**Cache Stampede / Thundering Herd Problem:**
When a popular cache entry expires and thousands of requests simultaneously hit the DB:
```java
// Solution: Probabilistic early expiration or mutex lock
public Product getProduct(Long id) {
    String key = "product:" + id;
    Product cached = redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;

    // Use distributed lock to prevent stampede
    String lockKey = "lock:product:" + id;
    Boolean locked = redisTemplate.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

    if (Boolean.TRUE.equals(locked)) {
        try {
            Product product = productRepository.findById(id).orElseThrow();
            redisTemplate.opsForValue().set(key, product, Duration.ofMinutes(30));
            return product;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // Wait and retry — another thread is populating the cache
        Thread.sleep(100);
        return getProduct(id);
    }
}
```

---

## 3. Messaging Systems

### 3.1 Kafka

**Concept:**
Apache Kafka is a distributed event streaming platform. It acts as a highly scalable, fault-tolerant, ordered log of events that producers write to and consumers read from.

**Real-world analogy:**
Kafka is like a newspaper publisher. Producers (journalists) publish articles to topics (sections: sports, business). Consumers (readers) subscribe to sections and read at their own pace. Old editions are retained for a period regardless of who's read them.

**Core Concepts:**

| Concept | Description |
|---|---|
| Topic | Named log of events (like a channel) |
| Partition | A topic is split into N partitions for parallelism |
| Offset | Sequential ID of a message within a partition |
| Producer | Application that writes events to a topic |
| Consumer | Application that reads events from a topic |
| Consumer Group | Multiple consumers sharing the read load of a topic |
| Broker | A Kafka server; a cluster has multiple brokers |
| Replication Factor | How many copies of each partition across brokers |

**Architecture:**
```
Producers                  Kafka Cluster                Consumers
                    Topic: "orders" (3 partitions)
[Order Service] ──► [Partition 0] ──────────────► [Consumer Group A]
[Mobile App]    ──► [Partition 1] ──────────────► Consumer A-1
                    [Partition 2] ──────────────► Consumer A-2
                                                  Consumer A-3
                                    ──────────► [Consumer Group B]
                                                  Consumer B-1 (analytics)
```

**Spring Boot Kafka:**
```java
// Producer
@Service
public class OrderProducer {

    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrder(Order order) {
        OrderEvent event = new OrderEvent(order.getId(), order.getStatus());
        kafkaTemplate.send("orders", order.getId().toString(), event);
    }
}

// Consumer
@Service
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "notification-service")
    public void handleOrder(OrderEvent event) {
        // Process each order event
        notificationService.sendConfirmation(event.getOrderId());
    }
}
```

**Key Guarantees:**
- **At-least-once delivery** (default): Message may be delivered more than once — consumers must be idempotent
- **Exactly-once**: Achieved with Kafka Transactions + idempotent producers
- **Ordering**: Guaranteed within a partition, not across partitions

**When to use Kafka:**
- High-throughput event streaming (millions of events/sec)
- Event sourcing / audit logs
- Microservices decoupling
- Real-time data pipelines (e.g., feed into data warehouse)

---

### 3.2 RabbitMQ

**Concept:**
RabbitMQ is a traditional message broker based on AMQP (Advanced Message Queuing Protocol). It routes messages through exchanges to queues, where consumers receive and process them.

**Core Concepts:**

| Concept | Description |
|---|---|
| Exchange | Receives messages from producers, routes to queues |
| Queue | Stores messages until consumed |
| Binding | Rule connecting an exchange to a queue |
| Routing Key | Label on a message used by exchange for routing |
| Virtual Host | Logical isolation within RabbitMQ |

**Exchange Types:**

| Type | Routing | Use Case |
|---|---|---|
| Direct | Routes to queue with matching routing key | Task queues, point-to-point |
| Fanout | Routes to ALL bound queues | Broadcast, pub/sub |
| Topic | Routes based on pattern matching on routing key | Selective subscriptions |
| Headers | Routes based on message headers | Complex routing rules |

**Spring Boot RabbitMQ:**
```java
// Configuration
@Configuration
public class RabbitConfig {

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("order.queue")
            .withArgument("x-dead-letter-exchange", "dlx")
            .build();
    }

    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange("order.exchange");
    }

    @Bean
    public Binding binding(Queue orderQueue, DirectExchange orderExchange) {
        return BindingBuilder.bind(orderQueue)
            .to(orderExchange)
            .with("order.created");
    }
}

// Producer
@Service
public class OrderPublisher {
    @Autowired private RabbitTemplate rabbitTemplate;

    public void publish(OrderEvent event) {
        rabbitTemplate.convertAndSend("order.exchange", "order.created", event);
    }
}

// Consumer
@Service
public class OrderListener {
    @RabbitListener(queues = "order.queue")
    public void handleOrder(OrderEvent event) {
        processOrder(event);
    }
}
```

**Kafka vs RabbitMQ:**

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| Model | Pull-based (consumers poll) | Push-based (broker pushes) |
| Message Retention | Retained for a period (replayable) | Deleted after consumption |
| Throughput | Very high (millions/sec) | High (tens of thousands/sec) |
| Ordering | Per partition | Per queue |
| Use Case | Event streaming, log aggregation | Task queues, RPC, complex routing |
| Consumer Groups | Multiple independent groups | Competing consumers on one queue |

---

### 3.3 Event-Driven Architecture

**Concept:**
Services communicate by producing and consuming events rather than making direct synchronous API calls. Services are decoupled — the producer doesn't know who consumes its events.

**Real-world analogy:**
A fire alarm (producer) emits an event. Sprinklers, emergency lighting, and the fire department all react independently. The alarm doesn't call each system individually.

**Patterns:**

**Event Notification:**
```
[Order Service] ──publishes──► "order.placed" ──► [Email Service]
                                                 ──► [Inventory Service]
                                                 ──► [Analytics Service]
```

**Event-Carried State Transfer:**
```java
// Event carries all data consumers need — no follow-up queries required
public class OrderPlacedEvent {
    private Long orderId;
    private String customerId;
    private String customerEmail; // included so email service doesn't need to call user service
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private Instant occurredAt;
}
```

**Saga Pattern (Distributed Transactions):**
```
[Order Saga Orchestrator]
  ├─► Reserve Inventory ──► Success ──► Charge Payment ──► Success ──► Confirm Order
  │                          Fail ──► Compensate: Cancel Order
  └─► Payment Fails ──► Compensate: Release Inventory → Cancel Order
```

**Choreography vs Orchestration:**
- **Choreography**: Each service reacts to events and emits new events. No central coordinator. Decoupled but hard to trace.
- **Orchestration**: A central saga orchestrator tells each service what to do. Easier to reason about, single point of failure.

**Benefits:**
- Loose coupling between services
- Independent scaling of producers and consumers
- Resilience — if a consumer is down, events queue up
- Natural audit log of everything that happened

**Challenges:**
- Eventual consistency — data is consistent across services eventually, not immediately
- Debugging is harder — no single call stack
- Duplicate events — consumers must be idempotent

---

## 4. Cloud Computing

### 4.1 AWS EC2 (Elastic Compute Cloud)

**Concept:**
EC2 provides virtual machines (instances) in the cloud. You choose CPU, memory, storage, and network. Java applications are typically deployed on EC2 instances running inside Docker containers or directly as JVM processes.

**Instance Types (for Java workloads):**

| Family | Optimized For | Example Use |
|---|---|---|
| t3/t4g | Burstable, general | Development, low-traffic APIs |
| m6i/m7i | Balanced CPU+Memory | Application servers |
| c6i/c7i | CPU-intensive | High-throughput processing |
| r6i/r7i | Memory-optimized | JVM with large heaps, in-memory caches |

**Key Concepts:**
- **AMI (Amazon Machine Image)**: Template for your instance (OS + pre-installed software)
- **Security Groups**: Virtual firewalls — control inbound/outbound traffic
- **Elastic IP**: Static IP address for your instance
- **User Data**: Script that runs on first boot (install Java, start your app)
- **Instance Store vs EBS**: Instance store is ephemeral (lost on stop); EBS (Elastic Block Store) persists

**Pricing Models:**
- **On-Demand**: Pay per second, no commitment — use for variable workloads
- **Reserved**: 1-3 year commitment, up to 72% discount — use for stable baseline
- **Spot**: Use spare AWS capacity, up to 90% discount — interruptible, use for batch jobs

---

### 4.2 AWS S3 (Simple Storage Service)

**Concept:**
S3 is object storage — store files (objects) in buckets. Highly durable (11 nines — 99.999999999%), infinitely scalable. Used for static files, backups, data lakes, and serving assets.

**Core Concepts:**
- **Bucket**: Container for objects (like a folder). Must have globally unique name.
- **Object**: A file + its metadata. Max size 5TB. Identified by key (path-like string).
- **Presigned URL**: Temporary URL to give users direct access to private objects

**Java AWS SDK v2:**
```java
@Service
public class S3Service {

    private final S3Client s3Client;
    private final String bucketName = "my-app-bucket";

    // Upload file
    public String uploadFile(String key, byte[] content, String contentType) {
        PutObjectRequest request = PutObjectRequest.builder()
            .bucket(bucketName)
            .key(key)
            .contentType(contentType)
            .build();

        s3Client.putObject(request, RequestBody.fromBytes(content));
        return "s3://" + bucketName + "/" + key;
    }

    // Download file
    public byte[] downloadFile(String key) {
        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(bucketName)
            .key(key)
            .build();

        return s3Client.getObjectAsBytes(request).asByteArray();
    }

    // Generate presigned URL (valid for 1 hour)
    public String generatePresignedUrl(String key) {
        S3Presigner presigner = S3Presigner.create();
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(Duration.ofHours(1))
            .getObjectRequest(r -> r.bucket(bucketName).key(key))
            .build();
        return presigner.presignGetObject(presignRequest).url().toString();
    }
}
```

**S3 Storage Classes:**

| Class | Use Case | Cost |
|---|---|---|
| Standard | Frequently accessed data | Highest |
| Intelligent-Tiering | Unknown or changing access patterns | Auto-moves between tiers |
| Standard-IA | Infrequent access but rapid retrieval needed | Lower |
| Glacier | Archival, retrieval in minutes | Very low |
| Glacier Deep Archive | Long-term archival, retrieval in hours | Lowest |

---

### 4.3 Load Balancers

**Concept:**
A load balancer distributes incoming traffic across multiple backend servers, ensuring no single server is overwhelmed. It also removes unhealthy instances from the pool.

**AWS Load Balancer Types:**

| Type | Layer | Best For |
|---|---|---|
| ALB (Application Load Balancer) | Layer 7 (HTTP/HTTPS) | Microservices, path-based routing, WebSockets |
| NLB (Network Load Balancer) | Layer 4 (TCP/UDP) | Ultra-low latency, static IP requirements |
| CLB (Classic Load Balancer) | Layer 4 + 7 | Legacy, avoid for new workloads |

**ALB Features:**
- **Path-based routing**: `/api/*` → API servers, `/static/*` → S3
- **Host-based routing**: `api.example.com` → API cluster, `app.example.com` → frontend
- **Health checks**: Removes unhealthy instances automatically
- **SSL Termination**: ALB handles HTTPS, backends communicate over HTTP
- **Sticky Sessions**: Route same user to same instance (session affinity)

**Architecture with ALB:**
```
Internet
    │
  [ALB]
  /   \
[EC2]  [EC2]  [EC2]     ← Auto Scaling Group
```

**Load Balancing Algorithms:**
- **Round Robin**: Requests distributed evenly in order
- **Least Connections**: Route to instance with fewest active connections
- **IP Hash**: Same client IP always goes to same server

---

### 4.4 Auto Scaling

**Concept:**
Auto Scaling automatically adjusts the number of EC2 instances based on demand. Scale out (add instances) when load increases, scale in (remove instances) when load drops.

**Scaling Types:**

| Type | How | When to Use |
|---|---|---|
| Target Tracking | Maintain a metric at a target (e.g., 70% CPU) | Most common, simple to configure |
| Step Scaling | Scale by N instances when metric exceeds threshold | Fine-grained control |
| Scheduled Scaling | Scale at specific times | Predictable traffic patterns |
| Predictive Scaling | ML-based, scales before demand arrives | Variable but predictable workloads |

**Auto Scaling Group Configuration:**
- **Minimum capacity**: Never go below this (e.g., 2 for HA)
- **Desired capacity**: Current target
- **Maximum capacity**: Cost protection ceiling

**Key Metrics to Scale On:**
- CPU Utilization
- Memory Utilization (requires CloudWatch agent)
- ALB Request Count per Target
- Custom application metrics (queue depth, active connections)

**Java application considerations for Auto Scaling:**
- Instances must be **stateless** — session data in Redis, not local memory
- Use **graceful shutdown** so in-flight requests complete before instance terminates
- Implement health check endpoint that ALB polls

```java
// Spring Boot graceful shutdown
// application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s

// Health check endpoint (auto-provided by Spring Actuator)
// ALB health check: GET /actuator/health → {"status": "UP"}
```

---

## 5. Containers

### 5.1 Docker

**Concept:**
Docker packages an application and all its dependencies (JDK, libraries, config) into a container — a lightweight, portable, isolated unit that runs identically anywhere.

**Real-world analogy:**
Shipping containers standardized cargo transport — the same container works on any ship, truck, or crane. Docker does the same for software — the same container runs on any machine with Docker installed.

**Key Concepts:**

| Concept | Description |
|---|---|
| Image | Blueprint for a container (read-only layers) |
| Container | Running instance of an image |
| Dockerfile | Instructions to build an image |
| Registry | Stores images (Docker Hub, ECR, GCR) |
| Volume | Persistent storage attached to a container |
| Network | Virtual network connecting containers |

**Optimized Dockerfile for Java/Spring Boot:**
```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q     # Cache dependencies layer
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: Runtime (minimal image)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Create non-root user (security best practice)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy only the JAR from build stage
COPY --from=builder /app/target/app.jar app.jar

# JVM tuning for containers
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+UseContainerSupport", \
  "-jar", "app.jar"]

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
```

**Docker Compose for local development:**
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mydb
      - SPRING_REDIS_HOST=redis
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

**Critical JVM + Docker settings:**
- `-XX:+UseContainerSupport`: JVM respects container memory limits (on by default in Java 10+)
- `-XX:MaxRAMPercentage=75.0`: Use 75% of container memory for heap (leave room for non-heap)
- Without these, JVM may see host RAM and set too-large heap, causing OOM kills

---

### 5.2 Kubernetes

**Concept:**
Kubernetes (K8s) is a container orchestration platform. It automates deploying, scaling, and managing containerized applications across a cluster of machines.

**Real-world analogy:**
Docker is like having a fleet of ships (containers). Kubernetes is the port authority that decides which ship goes where, ensures the right number of ships are running, and reroutes traffic when a ship goes down.

**Core Objects:**

| Object | Description |
|---|---|
| Pod | Smallest deployable unit; one or more containers sharing network/storage |
| Deployment | Manages a set of identical Pods; handles rolling updates |
| Service | Stable network endpoint for a set of Pods (load balances) |
| Ingress | HTTP routing rules to Services (path/host-based) |
| ConfigMap | Non-sensitive config data (externalized configuration) |
| Secret | Sensitive data (passwords, API keys) stored encrypted |
| HPA | Horizontal Pod Autoscaler — auto-scales Pod count |
| PersistentVolume | Storage that outlives a Pod |

**Spring Boot Kubernetes Deployment:**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:1.2.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 20
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
---
# service.yaml
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
  type: ClusterIP
---
# hpa.yaml
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
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Kubernetes vs Docker Compose:**

| Feature | Docker Compose | Kubernetes |
|---|---|---|
| Target | Local development | Production clusters |
| Scaling | Manual | Auto (HPA) |
| Self-healing | No | Yes (restarts failed Pods) |
| Rolling updates | No | Yes (zero-downtime) |
| Service discovery | Container names | DNS via Services |
| Complexity | Simple | Complex |

**Key K8s interview concepts:**
- **Rolling Update**: K8s replaces Pods gradually — no downtime during deployment
- **Readiness vs Liveness Probe**: Readiness = Pod ready to receive traffic; Liveness = Pod is alive (restart if fails)
- **Resource Requests vs Limits**: Requests = guaranteed resources (for scheduling); Limits = max allowed (throttled if exceeded)

---

## 6. Reliability

### 6.1 Circuit Breakers

**Concept:**
A circuit breaker wraps calls to external services. When failures exceed a threshold, the circuit "opens" and calls fail immediately without hitting the downstream service. After a timeout, it allows a test call — if it succeeds, the circuit closes.

**Real-world analogy:**
A home electrical circuit breaker: when a short circuit occurs, the breaker trips to prevent damage. After you fix the issue, you reset it.

**States:**
```
                 Failure threshold exceeded
CLOSED ─────────────────────────────────► OPEN
  ▲                                         │
  │ Test call succeeds                      │ After timeout
  │                                         ▼
HALF-OPEN ◄──────────────────────────────────
          Allows one test call through
```

**Resilience4j in Spring Boot:**
```java
// application.properties
resilience4j.circuitbreaker.instances.paymentService.sliding-window-size=10
resilience4j.circuitbreaker.instances.paymentService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.paymentService.wait-duration-in-open-state=30s
resilience4j.circuitbreaker.instances.paymentService.permitted-number-of-calls-in-half-open-state=3

// Service
@Service
public class OrderService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentServiceClient.charge(request); // external HTTP call
    }

    // Fallback: called when circuit is OPEN or call fails
    public PaymentResult fallbackPayment(PaymentRequest request, Exception ex) {
        // Queue payment for later processing, return pending status
        paymentQueue.add(request);
        return PaymentResult.pending("Payment queued for retry");
    }
}
```

**Circuit Breaker prevents:**
- Cascading failures: one slow service making all its callers slow
- Resource exhaustion: threads piling up waiting for a failing service
- Allows time for failing service to recover

---

### 6.2 Retry Mechanisms

**Concept:**
Automatically retry failed operations, especially for transient failures (network blips, temporary service unavailability). Must be used carefully — retrying non-idempotent operations causes problems.

**Retry with Exponential Backoff:**
```
Attempt 1 fails → wait 1s → Attempt 2 fails → wait 2s → Attempt 3 fails → wait 4s → Give up
```

**Resilience4j Retry:**
```java
// application.properties
resilience4j.retry.instances.inventoryService.max-attempts=3
resilience4j.retry.instances.inventoryService.wait-duration=500ms
resilience4j.retry.instances.inventoryService.exponential-backoff-multiplier=2
resilience4j.retry.instances.inventoryService.retry-exceptions=java.io.IOException,feign.FeignException

// Service
@Retry(name = "inventoryService", fallbackMethod = "fallbackInventoryCheck")
public boolean checkInventory(Long productId, int quantity) {
    return inventoryClient.isAvailable(productId, quantity);
}

public boolean fallbackInventoryCheck(Long productId, int quantity, Exception ex) {
    log.warn("Inventory check failed after retries, assuming available: {}", ex.getMessage());
    return true; // Optimistic fallback
}
```

**Retry + Jitter (prevent thundering herd):**
Adding random delay to retry intervals prevents all instances from retrying simultaneously:
```
Wait = min(cap, base * 2^attempt) + random(0, base)
```

**What NOT to retry:**
- `4xx` HTTP errors (client errors — retrying won't help)
- Non-idempotent operations without deduplication (e.g., charging a payment card twice)
- Operations that are already timing out due to slow response (retry just adds load)

---

### 6.3 Rate Limiting

**Concept:**
Rate limiting controls how many requests a client or service can make in a given time window. Protects services from abuse, overload, and ensures fair usage.

**Algorithms:**

| Algorithm | How it Works | Pros | Cons |
|---|---|---|---|
| Fixed Window | Count requests per time window (1000 req/min) | Simple | Allows burst at window boundary |
| Sliding Window Log | Maintain log of request timestamps | Accurate | High memory |
| Sliding Window Counter | Weighted combination of current + previous windows | Accurate, memory efficient | Slightly approximate |
| Token Bucket | Tokens added at rate R, each request consumes 1 | Allows bursts up to bucket size | Bursty traffic |
| Leaky Bucket | Queue requests, process at fixed rate | Smooth output | Adds latency |

**Implementation with Resilience4j + Redis:**
```java
// application.properties
resilience4j.ratelimiter.instances.apiGateway.limit-for-period=100
resilience4j.ratelimiter.instances.apiGateway.limit-refresh-period=1s
resilience4j.ratelimiter.instances.apiGateway.timeout-duration=0

// Controller
@RestController
public class ApiController {

    @RateLimiter(name = "apiGateway", fallbackMethod = "rateLimitFallback")
    @GetMapping("/api/products")
    public List<Product> getProducts() {
        return productService.findAll();
    }

    public List<Product> rateLimitFallback(RequestNotPermitted ex) {
        throw new ResponseStatusException(HttpStatus.TOO_MANY_REQUESTS, "Rate limit exceeded");
    }
}
```

**Distributed Rate Limiting with Redis:**
```java
// Using Redis to rate limit across multiple instances
public boolean isAllowed(String clientId) {
    String key = "rate_limit:" + clientId;
    Long count = redisTemplate.opsForValue().increment(key);
    if (count == 1) {
        redisTemplate.expire(key, Duration.ofSeconds(60));
    }
    return count <= 100; // 100 requests per minute
}
```

---

### 6.4 Fault Tolerance

**Concept:**
Fault tolerance is the ability of a system to continue operating (possibly in a degraded mode) when components fail. It combines circuit breakers, retries, timeouts, bulkheads, and fallbacks.

**Bulkhead Pattern:**
Isolate resources for different services so one failing dependency doesn't exhaust all threads.

```java
// Separate thread pool for external calls (bulkhead)
resilience4j.bulkhead.instances.paymentService.max-concurrent-calls=10
resilience4j.bulkhead.instances.paymentService.max-wait-duration=100ms

@Bulkhead(name = "paymentService", type = Bulkhead.Type.THREADPOOL)
@CircuitBreaker(name = "paymentService")
@Retry(name = "paymentService")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest request) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(request));
}
```

**Timeout:**
```java
// Always set timeouts — never let a call block indefinitely
resilience4j.timelimiter.instances.paymentService.timeout-duration=3s
resilience4j.timelimiter.instances.paymentService.cancel-running-future=true
```

**Fault Tolerance Hierarchy (apply in order):**
1. Timeout — don't wait forever
2. Retry — handle transient failures
3. Circuit Breaker — stop calling a failing service
4. Bulkhead — isolate resources
5. Fallback — degrade gracefully

---

## 7. Observability

**Concept:**
Observability is the ability to understand the internal state of a system from its external outputs. The three pillars are: Logs, Metrics, and Distributed Traces.

### 7.1 Logging

**Concept:**
Logs are immutable, timestamped records of discrete events that occurred in your application.

**Structured Logging (JSON format):**
```java
// logback-spring.xml or application.properties
// Use JSON format for log aggregation tools
logging.pattern.console={"time":"%d","level":"%p","logger":"%c","msg":"%m"}%n

// Use SLF4J + Logback (Spring Boot default)
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(OrderRequest request) {
        log.info("Creating order for customer={} items={}", request.getCustomerId(), request.getItems().size());

        try {
            Order order = processOrder(request);
            log.info("Order created successfully orderId={} customerId={}", order.getId(), request.getCustomerId());
            return order;
        } catch (PaymentException e) {
            log.error("Payment failed for customerId={} reason={}", request.getCustomerId(), e.getMessage(), e);
            throw e;
        }
    }
}
```

**Log Levels:**
| Level | When to Use |
|---|---|
| ERROR | Unexpected failures that need immediate attention |
| WARN | Degraded operation, unexpected but handled |
| INFO | Key business events (order created, user registered) |
| DEBUG | Detailed diagnostic info (request/response bodies, SQL) |
| TRACE | Highly detailed, for profiling specific issues |

**Correlation IDs (trace requests across services):**
```java
// MDC (Mapped Diagnostic Context) — adds fields to all log lines in a request
@Component
public class CorrelationIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String correlationId = ((HttpServletRequest) request).getHeader("X-Correlation-Id");
        if (correlationId == null) correlationId = UUID.randomUUID().toString();

        MDC.put("correlationId", correlationId);
        ((HttpServletResponse) response).setHeader("X-Correlation-Id", correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
// Log output: {"time":"...","level":"INFO","correlationId":"abc-123","msg":"Order created"}
```

**Log Aggregation Stack:**
```
App Instances → [Log Shipper: Filebeat/Fluentd] → [Elasticsearch] → [Kibana]
             OR → [CloudWatch Logs / Loki]
```

---

### 7.2 Metrics

**Concept:**
Metrics are numerical measurements collected over time — counters, gauges, histograms. They answer "how many" and "how long" questions at an aggregate level.

**Types:**

| Type | Description | Example |
|---|---|---|
| Counter | Monotonically increasing value | Total requests, total errors |
| Gauge | Value that goes up and down | Current active connections, queue depth |
| Histogram | Distribution of values in buckets | Request latency distribution |
| Timer | Special histogram for durations | DB query time, HTTP response time |

**Spring Boot Actuator + Micrometer:**
```java
// application.properties
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.metrics.export.prometheus.enabled=true

// Custom metrics
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderProcessingTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
            .tag("environment", "production")
            .description("Total orders created")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrderInternal(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

**Prometheus + Grafana Stack:**
```
App (exposes /actuator/prometheus) → [Prometheus scrapes metrics] → [Grafana visualizes]
```

**Key Metrics to Monitor for Java Apps:**
- JVM: heap used, GC pause time, GC frequency, thread count
- HTTP: request rate, error rate (5xx), p50/p95/p99 latency
- DB: connection pool active/idle, query latency, slow queries
- Business: orders/min, payment success rate, user signups

---

### 7.3 Monitoring

**Concept:**
Monitoring is the process of collecting, visualizing, and alerting on metrics to ensure the system is healthy and performing within expectations.

**SLO / SLA / SLI (critical terms):**

| Term | Meaning | Example |
|---|---|---|
| SLI (Indicator) | The metric you measure | 99th percentile latency = 200ms |
| SLO (Objective) | Target for the SLI | 99th percentile latency < 500ms, 99.9% of time |
| SLA (Agreement) | Legal contract based on SLOs | If SLO breached, customer gets credit |
| Error Budget | How much you can fail and still meet SLO | 0.1% of requests can fail per month |

**USE Method (for infrastructure):**
- **Utilization**: % time resource is busy (CPU at 80%)
- **Saturation**: How much work is queued (CPU queue depth)
- **Errors**: Error count/rate

**RED Method (for services):**
- **Rate**: Requests per second
- **Errors**: Errors per second
- **Duration**: Latency distribution (p50, p95, p99)

**Alerting Best Practices:**
- Alert on symptoms (high error rate, high latency) not causes (high CPU — which may be fine)
- Set alert thresholds above normal variance to avoid false positives
- Every alert should be actionable — if there's no action, it's not worth waking someone up

---

### 7.4 Distributed Tracing

**Concept:**
A single user request in a microservices system touches many services. Distributed tracing follows a request across all services, creating a visual "trace" — a timeline of every operation performed. Each operation is a "span."

**Real-world analogy:**
If you order pizza online, tracing follows: website → order service → inventory service → payment service → delivery service — and shows exactly how long each step took and where it failed.

**Trace Structure:**
```
TraceId: abc-123
│
├─ Span: POST /orders [order-service] 250ms
│    ├─ Span: SELECT * FROM products [DB] 15ms
│    ├─ Span: POST /payments [payment-service] 180ms
│    │    ├─ Span: Charge credit card [stripe API] 150ms
│    │    └─ Span: INSERT INTO payments [DB] 10ms
│    └─ Span: PUBLISH order.created [Kafka] 5ms
```

**Spring Boot with Micrometer Tracing (+ Zipkin/Jaeger):**
```java
// pom.xml dependencies
// micrometer-tracing-bridge-brave + zipkin-reporter-brave

// application.properties
management.tracing.sampling.probability=1.0  // 100% in dev, 0.1 (10%) in prod
spring.zipkin.base-url=http://zipkin:9411

// Traces propagate automatically through:
// - Spring MVC (HTTP requests)
// - RestTemplate / WebClient (outbound HTTP)
// - Spring Kafka (messages)
// - Spring Data JPA (DB calls)

// Manual span creation
@Service
public class PaymentService {

    @Autowired private Tracer tracer;

    public void processPayment(PaymentRequest request) {
        Span span = tracer.nextSpan().name("process-payment").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("payment.method", request.getMethod());
            externalPaymentGateway.charge(request);
        } catch (Exception e) {
            span.error(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

**Observability Stack (Common Choices):**

| Pillar | Open Source | Cloud (AWS) |
|---|---|---|
| Logs | ELK Stack (Elasticsearch + Logstash + Kibana) or Loki + Grafana | CloudWatch Logs |
| Metrics | Prometheus + Grafana | CloudWatch Metrics |
| Traces | Jaeger or Zipkin | AWS X-Ray |
| All-in-one | Grafana Stack (Loki + Prometheus + Tempo) | AWS CloudWatch + X-Ray |

---

## Summary — Quick Revision

| Topic | Key Takeaway |
|---|---|
| Indexing | B-Tree for range queries; composite index respects leftmost prefix rule |
| Query Optimization | Use EXPLAIN ANALYZE; solve N+1 with JOIN FETCH; paginate large results |
| Replication | Primary for writes, replicas for reads; async = eventual consistency |
| Partitioning | Same DB server, splits table; enables partition pruning |
| Sharding | Multiple DB servers; cross-shard queries are hard; stateless at app layer |
| Redis | In-memory, microsecond latency; supports rich data structures |
| Cache Aside | Lazy loading; resilient if cache down; risk of stale data |
| Write Through | Always consistent; higher write cost; cache grows with all writes |
| Cache Invalidation | TTL for tolerant cases; event-based for strict consistency; prevent stampede with locks |
| Kafka | Event streaming; replayable; high throughput; consumer groups for parallelism |
| RabbitMQ | Task queues; rich routing via exchanges; messages deleted after consumption |
| Event-Driven | Decoupled services; eventual consistency; consumers must be idempotent |
| EC2 | VMs in cloud; right-size instance type for JVM workloads |
| S3 | Object storage; presigned URLs for secure access; choose storage class by access pattern |
| Load Balancers | ALB for HTTP microservices; health checks remove failed instances |
| Auto Scaling | Stateless apps only; use target tracking; set readiness probes |
| Docker | Multi-stage builds; use `+UseContainerSupport`; non-root user |
| Kubernetes | Deployment + Service + HPA; readiness vs liveness probes; resource limits |
| Circuit Breaker | CLOSED → OPEN → HALF-OPEN; prevents cascading failures |
| Retry | Exponential backoff + jitter; only retry idempotent operations |
| Rate Limiting | Token bucket for bursts; sliding window for accuracy; Redis for distributed |
| Fault Tolerance | Timeout → Retry → Circuit Breaker → Bulkhead → Fallback |
| Logging | Structured JSON; correlations IDs; right log levels |
| Metrics | RED method for services (Rate, Errors, Duration); Prometheus + Grafana |
| Monitoring | Alert on symptoms; SLO/SLI/Error Budget mindset |
| Distributed Tracing | TraceId links all spans across services; Micrometer + Zipkin/Jaeger |
