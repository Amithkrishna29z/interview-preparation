# Architecture Topics Every Java Developer Should Learn

A junior developer should *recognize* these architecture concepts and explain what each is and why it matters. This guide keeps the core ideas with one clear example each; deeper architect-level detail is summarized to awareness level.

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
- What is a covering index? (Index contains all columns the query needs — no table lookup required)
- When would you NOT add an index? (Small tables, write-heavy tables, low-cardinality columns like boolean)

---

### 1.2 Query Optimization

**Concept:**
Query optimization is the process of writing and structuring SQL so the database executes it as efficiently as possible. The database engine has a query planner/optimizer that chooses the execution plan.

**EXPLAIN / EXPLAIN ANALYZE:**
```sql
-- PostgreSQL: see the execution plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 100;
-- Seq Scan = no index used (bad on large tables); Index Scan = good.
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

**Quick checklist:** run `EXPLAIN ANALYZE` to spot Seq Scans, watch for N+1 (`spring.jpa.show-sql=true`), paginate large results, and avoid wrapping indexed columns in functions (`WHERE YEAR(created_at) = 2024` blocks the index — use a `BETWEEN` range instead).

---

### 1.3 Replication

*Awareness level — recognize the term and why it exists.*

Replication keeps read-only copies (replicas) of a primary database in sync. **Why:** route reads to replicas to scale read traffic, and promote a replica if the primary fails (high availability). In Spring Boot you mark read paths with `@Transactional(readOnly = true)` so they can be routed to a replica.

- **Async** replication = lower latency but **replication lag** (a replica can briefly serve stale data); **sync** = strong consistency but slower writes.

---

### 1.4 Partitioning

*Awareness level.*

Partitioning splits one large table into smaller pieces **on the same database server** (e.g. `orders` split by year via `PARTITION BY RANGE (created_at)`). **Why:** queries with a WHERE on the partition key scan only the relevant partition (partition pruning), and old partitions can be dropped cheaply instead of slow bulk DELETEs.

---

### 1.5 Sharding

*Awareness level.*

Sharding splits data across **multiple independent database servers** (shards), each holding a subset (e.g. `shard = hash(user_id) % num_shards`). **Why:** horizontal scaling when one server can't hold all the data. **Key gotcha for interviews:** cross-shard JOINs and distributed transactions are hard, so this is an architect-level decision, not an everyday junior task.

- Partitioning = same server; Sharding = many servers.

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

**Good to know:** when memory fills, Redis evicts keys per an eviction policy (e.g. `allkeys-lru` evicts least-recently-used). It can persist to disk via RDB snapshots and/or the AOF write log.

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

**Pros:** only caches data that is actually requested; resilient (app still works if cache is down).
**Cons:** cold-start miss on first access; risk of stale data if invalidation fails.

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

**Cache Stampede / Thundering Herd (awareness):** when a popular key expires, many requests hit the DB at once. Mitigate with a short distributed lock so only one request repopulates the cache, or randomized/early TTL expiry.

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
- **At-least-once delivery** (default): a message may arrive more than once — consumers must be idempotent.
- **Ordering**: guaranteed within a partition, not across partitions.

**Use it for:** high-throughput event streaming, decoupling microservices, audit logs / data pipelines.

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
// Producer: send to an exchange with a routing key
rabbitTemplate.convertAndSend("order.exchange", "order.created", event);

// Consumer
@RabbitListener(queues = "order.queue")
public void handleOrder(OrderEvent event) {
    processOrder(event);
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

**Patterns to recognize (awareness):**
- **Event-Carried State Transfer**: the event carries all data consumers need (e.g. `OrderPlacedEvent` includes `customerEmail`), so they don't call back to other services.
- **Saga**: model a distributed transaction as a sequence of steps with compensating actions on failure. *Choreography* = services react to each other's events (no coordinator); *Orchestration* = a central coordinator drives the steps.

**Benefits:** loose coupling, independent scaling, resilience (events queue if a consumer is down).
**Challenges:** eventual consistency, harder debugging (no single call stack), duplicate events — so consumers must be idempotent.

---

## 4. Cloud Computing

### 4.1 AWS EC2 (Elastic Compute Cloud)

**Concept:**
EC2 provides virtual machines (instances) in the cloud — you choose CPU, memory, storage, and network. Java apps run on EC2 either in Docker containers or directly as JVM processes. Instance *families* are sized for the workload (e.g. `t3` burstable for dev, `r6i` memory-optimized for large JVM heaps).

**Terms to recognize:** **AMI** (instance template), **Security Group** (virtual firewall), **Elastic IP** (static IP), **EBS** (persistent disk; instance store is ephemeral).

**Pricing:** On-Demand (pay-as-you-go), Reserved (1–3 yr commitment, big discount), Spot (cheap spare capacity, interruptible).

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

    // Download: s3Client.getObjectAsBytes(GetObjectRequest...).asByteArray()
}
```

**Good to know:** a **presigned URL** (via `S3Presigner`) gives a user temporary direct access to a private object. **Storage classes** trade cost for access speed — Standard (hot data) down to Glacier / Deep Archive (cheap, slow archival).

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

**ALB features to recognize:** path-/host-based routing (`/api/*` → API servers), automatic **health checks** (removes unhealthy instances), **SSL termination**, and **sticky sessions** (session affinity).

**Common algorithms:** Round Robin (even distribution), Least Connections (fewest active), IP Hash (same client → same server).

---

### 4.4 Auto Scaling

**Concept:**
Auto Scaling automatically adjusts the number of EC2 instances based on demand. Scale out (add instances) when load increases, scale in (remove instances) when load drops.

**How it scales:** the common approach is **target tracking** — keep a metric (e.g. CPU at 70%, or ALB request count) at a target. An Auto Scaling Group has min / desired / max capacity (min for HA, max as a cost ceiling).

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

**Critical JVM + Docker setting:** `-XX:+UseContainerSupport` (default in Java 10+) makes the JVM respect the container's memory limit; pair with `-XX:MaxRAMPercentage=75.0`. Without these the JVM may size the heap to host RAM and get OOM-killed.

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

**Spring Boot Kubernetes Deployment (core shape):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:1.2.0
        ports:
        - containerPort: 8080
        resources:                 # requests = guaranteed, limits = max
          requests: { memory: "512Mi", cpu: "250m" }
          limits:   { memory: "1Gi",   cpu: "500m" }
        readinessProbe:            # ready to receive traffic
          httpGet: { path: /actuator/health/readiness, port: 8080 }
        livenessProbe:             # alive — restart if it fails
          httpGet: { path: /actuator/health/liveness, port: 8080 }
```
A **Service** then gives the Pods a stable endpoint, and an **HPA** auto-scales replica count on a metric (e.g. CPU 70%).

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
// Config in application.properties sets failure-rate-threshold, wait-duration-in-open-state, etc.
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentServiceClient.charge(request); // external HTTP call
}

// Fallback runs when the circuit is OPEN or the call fails
public PaymentResult fallbackPayment(PaymentRequest request, Exception ex) {
    paymentQueue.add(request);
    return PaymentResult.pending("Payment queued for retry");
}
```

**Why:** prevents cascading failures and thread exhaustion when a downstream service is slow/down, and gives it time to recover.

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
// Config sets max-attempts, wait-duration, exponential-backoff-multiplier, retry-exceptions
@Retry(name = "inventoryService", fallbackMethod = "fallbackInventoryCheck")
public boolean checkInventory(Long productId, int quantity) {
    return inventoryClient.isAvailable(productId, quantity);
}
```

**Add jitter** (random delay) to backoff so all instances don't retry at the same instant.

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

**Resilience4j Rate Limiter:**
```java
@RateLimiter(name = "apiGateway", fallbackMethod = "rateLimitFallback")
@GetMapping("/api/products")
public List<Product> getProducts() {
    return productService.findAll();
}
// fallback throws HttpStatus.TOO_MANY_REQUESTS (429)
```

**Distributed (across instances):** use a Redis counter per client — `INCR` the key, set its TTL on first hit, and reject once the count exceeds the limit.

---

### 6.4 Fault Tolerance

**Concept:**
Fault tolerance is the ability of a system to continue operating (possibly in a degraded mode) when components fail. It combines circuit breakers, retries, timeouts, bulkheads, and fallbacks.

**Bulkhead Pattern:** isolate resources (e.g. a separate thread pool per dependency) so one failing service can't exhaust all threads. Resilience4j annotations stack — `@Bulkhead`, `@CircuitBreaker`, `@Retry` on the same method.

**Timeout:** always set one — never let a call block indefinitely (`resilience4j.timelimiter` or per-client timeouts).

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

**Correlation IDs:** put a per-request ID into SLF4J's **MDC** so every log line in that request carries it (and propagate it via an `X-Correlation-Id` header) — this lets you trace one request across services. Logs are then shipped to an aggregator (ELK / Loki / CloudWatch) for searching.

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
// Expose metrics: management.endpoints.web.exposure.include=health,metrics,prometheus
// Register custom meters from the injected MeterRegistry:
Counter orderCounter = Counter.builder("orders.created").register(registry);
orderCounter.increment();
```
Prometheus scrapes `/actuator/prometheus` and Grafana visualizes it.

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

**Two monitoring frameworks to know:** **USE** (infrastructure — Utilization, Saturation, Errors) and **RED** (services — Rate, Errors, Duration/latency p50/p95/p99).

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

**Spring Boot with Micrometer Tracing (+ Zipkin/Jaeger):** add `micrometer-tracing` and set a sampling probability (`management.tracing.sampling.probability` — 1.0 in dev, ~0.1 in prod). Spring auto-propagates traces through MVC, RestTemplate/WebClient, Kafka, and JPA; you can also create manual spans via the injected `Tracer` for custom operations.

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
| Databases | Index WHERE/JOIN columns; use EXPLAIN ANALYZE; fix N+1 with JOIN FETCH. Replication = read replicas; partitioning = one server; sharding = many servers |
| Caching | Redis = in-memory; cache-aside is the default pattern; invalidate via TTL or events; watch for stampedes |
| Messaging | Kafka = replayable event streaming; RabbitMQ = task queues + routing; event-driven = decoupled + idempotent consumers |
| Cloud | EC2 = VMs; S3 = object storage (presigned URLs); ALB load-balances + health checks; auto scaling needs stateless apps |
| Containers | Docker multi-stage builds + `+UseContainerSupport`; K8s = Deployment + Service + HPA, readiness vs liveness probes |
| Reliability | Timeout → Retry (backoff + jitter) → Circuit Breaker → Bulkhead → Fallback |
| Observability | Logs (structured + correlation IDs), Metrics (RED method, Prometheus/Grafana), Traces (TraceId across services) |
