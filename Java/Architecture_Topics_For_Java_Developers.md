# Architecture Topics Every Java Developer Should Learn

A junior developer should *recognize* these architecture concepts and explain what each is and why it matters. This guide keeps the core ideas with one clear example each.

---

## Table of Contents
1. [Databases](#1-databases)
   - [Indexing](#11-indexing)
   - [Query Optimization](#12-query-optimization)
   - [Replication](#13-replication)
   - [Partitioning](#14-partitioning)
   - [Sharding](#15-sharding)
2. [Caching](#2-caching)
   - [Redis](#21-redis)
   - [Cache Aside Pattern](#22-cache-aside-pattern-lazy-loading)
   - [Write Through Cache](#23-write-through-cache)
   - [Cache Invalidation](#24-cache-invalidation)
3. [Messaging Systems](#3-messaging-systems)
   - [Kafka](#31-kafka)
   - [RabbitMQ](#32-rabbitmq)
   - [Event-Driven Architecture](#33-event-driven-architecture)
4. [Cloud Computing](#4-cloud-computing)
   - [AWS EC2](#41-aws-ec2-elastic-compute-cloud)
   - [AWS S3](#42-aws-s3-simple-storage-service)
   - [Load Balancers](#43-load-balancers)
   - [Auto Scaling](#44-auto-scaling)
5. [Containers](#5-containers)
   - [Docker](#51-docker)
   - [Kubernetes](#52-kubernetes)
6. [Reliability](#6-reliability)
   - [Circuit Breakers](#61-circuit-breakers)
   - [Retry Mechanisms](#62-retry-mechanisms)
   - [Rate Limiting](#63-rate-limiting)
   - [Fault Tolerance](#64-fault-tolerance)
7. [Observability](#7-observability)
   - [Logging](#71-logging)
   - [Metrics](#72-metrics)
   - [Monitoring](#73-monitoring)
   - [Distributed Tracing](#74-distributed-tracing)
8. [Summary — Quick Revision](#summary--quick-revision)

---

## 1. Databases

### 1.1 Indexing

An index is a separate data structure the database maintains to speed up lookups — like a book's index letting you jump to the right page instead of scanning every one.

**Types of Indexes:**

| Index Type | Best For |
|---|---|
| B-Tree (default) | Range queries, ORDER BY, equality — O(log n) |
| Hash Index | Exact equality only — O(1) avg |
| Composite Index | Multi-column WHERE (leftmost prefix rule) |
| Covering Index | Query only needs indexed columns — no table hit |
| Full-Text Index | Text search (`LIKE '%keyword%'`) |

**Java / Spring Boot:**
```java
@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_email", columnList = "email"),
    @Index(name = "idx_name_dob", columnList = "last_name, date_of_birth")
})
public class User { ... }
```

**Key Rules:**
- Index columns used in `WHERE`, `JOIN ON`, `ORDER BY`, `GROUP BY`
- Too many indexes slow `INSERT`/`UPDATE`/`DELETE` — indexes must be kept in sync
- **Leftmost prefix rule**: index on `(a, b, c)` helps queries on `a`, `a+b`, `a+b+c` — but NOT `b` alone
- High-cardinality columns (email, UUID) benefit most

**Common Interview Questions:**
- *What is a covering index?* — It contains all columns the query needs, so no table lookup is required.
- *When would you NOT add an index?* — Small tables, write-heavy tables, low-cardinality columns (e.g. a boolean).

---

### 1.2 Query Optimization

Writing SQL so the database executes it as efficiently as possible.

**EXPLAIN ANALYZE:**
```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 100;
-- Seq Scan = no index used (bad on large tables); Index Scan = good
```

**Common Fixes:**

| Problem | Solution |
|---|---|
| Full table scan | Add an index |
| N+1 query problem | Use `JOIN FETCH` or `@EntityGraph` |
| SELECT * fetching too much | Select only needed columns |
| Slow ORDER BY | Add index on sort column |

**N+1 Problem (Critical Interview Topic):**
```java
// BAD: 1 query for orders + N queries for each customer
List<Order> orders = orderRepository.findAll();
for (Order o : orders) {
    System.out.println(o.getCustomer().getName()); // lazy load = N queries
}

// GOOD: single JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
```

**Quick checklist:** run `EXPLAIN ANALYZE` to spot Seq Scans, enable `spring.jpa.show-sql=true` to catch N+1, paginate large results, avoid wrapping indexed columns in functions (`WHERE YEAR(created_at) = 2024` blocks the index — use `BETWEEN` instead).

---

### 1.3 Replication

*Awareness level.*

Replication keeps read-only copies (replicas) of a primary database in sync. Route reads to replicas to scale read traffic; promote a replica if the primary fails. In Spring Boot, mark read paths with `@Transactional(readOnly = true)`.

- **Async** replication = lower latency but possible **replication lag** (stale reads); **sync** = strong consistency but slower writes.

---

### 1.4 Partitioning

*Awareness level.*

Partitioning splits one large table into smaller pieces **on the same server** (e.g. `PARTITION BY RANGE (created_at)` by year). Queries with a WHERE on the partition key scan only the relevant partition (partition pruning), and old partitions can be dropped cheaply.

---

### 1.5 Sharding

*Awareness level.*

Sharding splits data across **multiple independent database servers**, each holding a subset (e.g. `shard = hash(user_id) % num_shards`). Used for horizontal scaling when one server can't hold all data. Cross-shard JOINs and distributed transactions are hard — this is an architect-level decision.

- Partitioning = same server; Sharding = many servers.

---

## 2. Caching

### 2.1 Redis

Redis (Remote Dictionary Server) is an in-memory key-value store used as a cache, message broker, and session store. Data lives in RAM, making reads/writes far faster than a disk-based database.

**Data Structures:**

| Structure | Use Case | Example |
|---|---|---|
| String | Key-value, counters | `SET user:1:name "John"` |
| Hash | Object with fields | `HSET user:1 name "John" age 30` |
| List | Queues, activity feeds | `LPUSH notifications msg1` |
| Set | Unique collections | `SADD user:1:tags "java"` |
| Sorted Set | Leaderboards | `ZADD leaderboard 1500 "user1"` |
| TTL | Expiring keys | `EXPIRE session:abc 3600` |

**Spring Boot:**
```java
@SpringBootApplication
@EnableCaching
public class Application { }

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
        // First call hits DB; subsequent calls return from Redis
    }

    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product); // evicts cache entry
    }
}
```

When memory fills, Redis evicts keys per an eviction policy (e.g. `allkeys-lru`).

---

### 2.2 Cache Aside Pattern (Lazy Loading)

The application loads data into the cache on demand. On a miss, it reads from the DB and populates the cache.

```
Read:  check cache → HIT: return | MISS: read DB → write to cache → return
Write: write to DB → invalidate cache entry (next read repopulates)
```

```java
public User getUser(Long userId) {
    String key = "user:" + userId;
    User cached = (User) redisTemplate.opsForValue().get(key);
    if (cached != null) return cached;

    User user = userRepository.findById(userId).orElseThrow();
    redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
    return user;
}

public void updateUser(User user) {
    userRepository.save(user);
    redisTemplate.delete("user:" + user.getId()); // invalidate
}
```

**Pros:** only caches hot data; works even if cache is down.
**Cons:** cold-start miss on first access; risk of stale data if invalidation fails.

---

### 2.3 Write Through Cache

Every write goes to **both** the cache and the DB synchronously — the cache is always up to date.

**Comparison:**

| Aspect | Cache Aside | Write Through |
|---|---|---|
| Cache population | Lazy (on read miss) | Eager (on every write) |
| Read performance | Slow on first miss | Always fast |
| Write performance | Fast | Slower (sync to cache + DB) |
| Consistency | Can be stale | Always consistent |
| Resilience | Works if cache is down | Writes fail if cache is down |

Use Write Through for read-heavy workloads where stale reads are unacceptable (e.g. session stores).

---

### 2.4 Cache Invalidation

Removing or updating cache entries when underlying data changes.

| Strategy | How | When to Use |
|---|---|---|
| TTL | Entries expire after N seconds | Occasional staleness acceptable |
| Event-based | Write triggers explicit delete | Consistency is critical |
| Write Through | Cache updated on every write | Cache must always be fresh |

```java
// Event-based invalidation
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onProductUpdated(ProductUpdatedEvent event) {
    redisTemplate.delete("product:" + event.getProductId());
}
```

**Cache Stampede (awareness):** when a popular key expires, many requests hit the DB simultaneously. Mitigate with a short distributed lock so only one request repopulates, or use randomized TTL expiry.

---

## 3. Messaging Systems

### 3.1 Kafka

Apache Kafka is a distributed event streaming platform — a highly scalable, fault-tolerant, ordered log of events that producers write to and consumers read from.

**Core Concepts:**

| Concept | Description |
|---|---|
| Topic | Named log of events |
| Partition | Topic split into N parts for parallelism |
| Offset | Sequential ID of a message within a partition |
| Consumer Group | Multiple consumers sharing the read load |
| Broker | A Kafka server; cluster has multiple brokers |

**Spring Boot:**
```java
// Producer
@Service
public class OrderProducer {
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrder(Order order) {
        kafkaTemplate.send("orders", order.getId().toString(), new OrderEvent(order.getId(), order.getStatus()));
    }
}

// Consumer
@Service
public class OrderConsumer {
    @KafkaListener(topics = "orders", groupId = "notification-service")
    public void handleOrder(OrderEvent event) {
        notificationService.sendConfirmation(event.getOrderId());
    }
}
```

**Key Guarantees:**
- **At-least-once delivery** (default): a message may arrive more than once — consumers must be idempotent.
- **Ordering**: guaranteed within a partition, not across partitions.

Use for: high-throughput event streaming, decoupling microservices, audit logs.

---

### 3.2 RabbitMQ

RabbitMQ is a traditional message broker (AMQP). Producers send messages to an **Exchange**, which routes them to **Queues** via binding rules.

**Exchange Types:**

| Type | Routing | Use Case |
|---|---|---|
| Direct | Exact routing key match | Task queues, point-to-point |
| Fanout | All bound queues | Broadcast |
| Topic | Pattern match on routing key | Selective subscriptions |

**Spring Boot:**
```java
rabbitTemplate.convertAndSend("order.exchange", "order.created", event);

@RabbitListener(queues = "order.queue")
public void handleOrder(OrderEvent event) { processOrder(event); }
```

**Kafka vs RabbitMQ:**

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| Model | Pull-based (consumers poll) | Push-based (broker pushes) |
| Message Retention | Retained, replayable | Deleted after consumption |
| Throughput | Very high (millions/sec) | High (tens of thousands/sec) |
| Use Case | Event streaming, log aggregation | Task queues, complex routing |

---

### 3.3 Event-Driven Architecture

Services communicate by producing and consuming events rather than direct synchronous API calls. The producer doesn't know who consumes its events.

```
[Order Service] ──publishes──► "order.placed" ──► [Email Service]
                                                 ──► [Inventory Service]
                                                 ──► [Analytics Service]
```

**Patterns to recognize:**
- **Event-Carried State Transfer**: the event contains all data consumers need, so they don't need to call back.
- **Saga**: a distributed transaction as a sequence of steps with compensating actions on failure. *Choreography* = services react to each other's events; *Orchestration* = a central coordinator drives the steps.

**Benefits:** loose coupling, independent scaling, resilience.
**Challenges:** eventual consistency, harder debugging, duplicate events — consumers must be idempotent.

---

## 4. Cloud Computing

### 4.1 AWS EC2 (Elastic Compute Cloud)

EC2 provides virtual machines in the cloud. Java apps run on EC2 in Docker containers or directly as JVM processes.

**Terms to know:** **AMI** (instance template), **Security Group** (virtual firewall), **Elastic IP** (static IP), **EBS** (persistent disk; instance store is ephemeral).

**Pricing:** On-Demand (pay-as-you-go), Reserved (1–3 yr commitment, big discount), Spot (cheap spare capacity, interruptible).

---

### 4.2 AWS S3 (Simple Storage Service)

S3 is object storage — store files in buckets. Highly durable (99.999999999%), infinitely scalable. Used for static files, backups, and assets.

- **Bucket**: container for objects (globally unique name)
- **Object**: a file + metadata, identified by a key
- **Presigned URL**: temporary URL for direct access to a private object

**Java AWS SDK v2:**
```java
@Service
public class S3Service {
    private final S3Client s3Client;
    private final String bucketName = "my-app-bucket";

    public String uploadFile(String key, byte[] content, String contentType) {
        PutObjectRequest request = PutObjectRequest.builder()
            .bucket(bucketName).key(key).contentType(contentType).build();
        s3Client.putObject(request, RequestBody.fromBytes(content));
        return "s3://" + bucketName + "/" + key;
    }
}
```

**Storage classes** trade cost for access speed — Standard (hot) down to Glacier/Deep Archive (cheap, slow archival).

---

### 4.3 Load Balancers

Distributes incoming traffic across multiple backend servers and removes unhealthy instances from the pool.

**AWS Types:**

| Type | Layer | Best For |
|---|---|---|
| ALB (Application) | Layer 7 (HTTP/HTTPS) | Microservices, path-based routing |
| NLB (Network) | Layer 4 (TCP/UDP) | Ultra-low latency, static IP |

**ALB features:** path-/host-based routing, health checks, SSL termination, sticky sessions.

**Algorithms:** Round Robin, Least Connections, IP Hash.

---

### 4.4 Auto Scaling

Automatically adjusts the number of EC2 instances based on demand (scale out on load, scale in when idle). **Target tracking** is the common approach — keep a metric (e.g. CPU at 70%) at a target.

**Java app requirements for Auto Scaling:**
- Instances must be **stateless** — session data in Redis, not local memory
- Use **graceful shutdown** so in-flight requests complete before termination
- Expose a health check endpoint for the ALB to poll

```properties
# application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
# ALB polls: GET /actuator/health → {"status":"UP"}
```

---

## 5. Containers

### 5.1 Docker

Docker packages an application and all its dependencies into a **container** — a lightweight, portable, isolated unit that runs identically anywhere.

**Key Concepts:**

| Concept | Description |
|---|---|
| Image | Blueprint for a container (read-only layers) |
| Container | Running instance of an image |
| Dockerfile | Instructions to build an image |
| Registry | Stores images (Docker Hub, ECR) |
| Volume | Persistent storage for a container |

**Optimized Dockerfile for Spring Boot:**
```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: Runtime (minimal image)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
COPY --from=builder /app/target/app.jar app.jar
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-XX:+UseContainerSupport", "-jar", "app.jar"]
EXPOSE 8080
```

**Docker Compose for local dev:**
```yaml
version: '3.8'
services:
  app:
    build: .
    ports: ["8080:8080"]
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mydb
      - SPRING_REDIS_HOST=redis
    depends_on: [db, redis]
  db:
    image: postgres:15-alpine
    environment: { POSTGRES_DB: mydb, POSTGRES_PASSWORD: secret }
    volumes: [postgres_data:/var/lib/postgresql/data]
  redis:
    image: redis:7-alpine
volumes:
  postgres_data:
```

**Critical:** `-XX:+UseContainerSupport` (default in Java 10+) makes the JVM respect container memory limits. Pair with `-XX:MaxRAMPercentage=75.0` to avoid OOM kills.

---

### 5.2 Kubernetes

Kubernetes (K8s) automates deploying, scaling, and managing containerized applications across a cluster. Docker runs containers; Kubernetes decides where they run, keeps the right number running, and reroutes traffic when one fails.

**Core Objects:**

| Object | Description |
|---|---|
| Pod | Smallest deployable unit; one or more containers |
| Deployment | Manages a set of identical Pods; handles rolling updates |
| Service | Stable network endpoint for a set of Pods |
| Ingress | HTTP routing rules to Services |
| ConfigMap | Non-sensitive config (externalized configuration) |
| Secret | Sensitive data (passwords, API keys) |
| HPA | Horizontal Pod Autoscaler |

**Spring Boot Deployment:**
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
        resources:
          requests: { memory: "512Mi", cpu: "250m" }
          limits:   { memory: "1Gi",   cpu: "500m" }
        readinessProbe:
          httpGet: { path: /actuator/health/readiness, port: 8080 }
        livenessProbe:
          httpGet: { path: /actuator/health/liveness, port: 8080 }
```

**Key concepts:**
- **Rolling Update**: Pods replaced gradually — no downtime during deployment
- **Readiness vs Liveness**: Readiness = ready for traffic; Liveness = still alive (restart if fails)
- **Requests vs Limits**: Requests = guaranteed (for scheduling); Limits = max allowed

**Docker Compose vs Kubernetes:**

| Feature | Docker Compose | Kubernetes |
|---|---|---|
| Target | Local development | Production clusters |
| Scaling | Manual | Auto (HPA) |
| Self-healing | No | Yes (restarts failed Pods) |
| Rolling updates | No | Yes (zero-downtime) |

---

## 6. Reliability

### 6.1 Circuit Breakers

Wraps calls to external services. When failures exceed a threshold, the circuit "opens" and calls fail immediately without hitting the downstream service. After a timeout it allows a test call — if it succeeds, the circuit closes.

```
CLOSED ──(failures exceed threshold)──► OPEN
  ▲                                       │
  │ test call succeeds          after timeout
  └────────── HALF-OPEN ◄─────────────────┘
```

**Resilience4j:**
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
public PaymentResult processPayment(PaymentRequest request) {
    return paymentServiceClient.charge(request);
}

public PaymentResult fallbackPayment(PaymentRequest request, Exception ex) {
    paymentQueue.add(request);
    return PaymentResult.pending("Payment queued for retry");
}
```

**Why:** prevents cascading failures and thread exhaustion, and gives downstream services time to recover.

---

### 6.2 Retry Mechanisms

Automatically retry failed operations, especially for transient failures. Use **exponential backoff** so retries don't hammer a struggling service.

```
Attempt 1 fails → wait 1s → Attempt 2 fails → wait 2s → Attempt 3 fails → give up
```

**Resilience4j:**
```java
@Retry(name = "inventoryService", fallbackMethod = "fallbackInventoryCheck")
public boolean checkInventory(Long productId, int quantity) {
    return inventoryClient.isAvailable(productId, quantity);
}
```

Add **jitter** (random delay) so all instances don't retry simultaneously.

**Do NOT retry:**
- `4xx` errors (client errors — retrying won't help)
- Non-idempotent operations without deduplication (e.g. charging a card twice)
- Operations already timing out due to slow response

---

### 6.3 Rate Limiting

Controls how many requests a client can make in a time window. Protects services from abuse and ensures fair usage.

**Common Algorithms:**

| Algorithm | How | Note |
|---|---|---|
| Fixed Window | Count per window | Allows burst at boundary |
| Token Bucket | Tokens added at rate R; each request consumes 1 | Allows bursts up to bucket size |
| Leaky Bucket | Queue requests, process at fixed rate | Smooth output, adds latency |

**Resilience4j:**
```java
@RateLimiter(name = "apiGateway", fallbackMethod = "rateLimitFallback")
@GetMapping("/api/products")
public List<Product> getProducts() { return productService.findAll(); }
// fallback returns HTTP 429 Too Many Requests
```

**Distributed rate limiting:** use a Redis counter per client (`INCR` + TTL).

---

### 6.4 Fault Tolerance

The ability of a system to continue operating (possibly in degraded mode) when components fail.

**Fault Tolerance Hierarchy — apply in order:**
1. **Timeout** — don't wait forever
2. **Retry** — handle transient failures
3. **Circuit Breaker** — stop calling a failing service
4. **Bulkhead** — isolate resources (separate thread pool per dependency)
5. **Fallback** — degrade gracefully

Resilience4j annotations (`@Bulkhead`, `@CircuitBreaker`, `@Retry`) can be stacked on the same method.

---

## 7. Observability

Observability is the ability to understand the internal state of a system from its external outputs. The three pillars are **Logs**, **Metrics**, and **Distributed Traces**.

### 7.1 Logging

Immutable, timestamped records of discrete events.

```java
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(OrderRequest request) {
        log.info("Creating order customerId={} items={}", request.getCustomerId(), request.getItems().size());
        try {
            Order order = processOrder(request);
            log.info("Order created orderId={}", order.getId());
            return order;
        } catch (PaymentException e) {
            log.error("Payment failed customerId={} reason={}", request.getCustomerId(), e.getMessage(), e);
            throw e;
        }
    }
}
```

**Log Levels:**

| Level | When to Use |
|---|---|
| ERROR | Unexpected failures needing immediate attention |
| WARN | Degraded but handled |
| INFO | Key business events |
| DEBUG | Detailed diagnostics (SQL, request bodies) |

**Correlation IDs:** put a per-request ID into SLF4J's MDC so every log line carries it. Propagate via `X-Correlation-Id` header to trace one request across services. Ship logs to an aggregator (ELK / Loki / CloudWatch).

---

### 7.2 Metrics

Numerical measurements over time that answer "how many" and "how long" questions.

| Type | Description | Example |
|---|---|---|
| Counter | Monotonically increasing | Total requests, total errors |
| Gauge | Goes up and down | Active connections, queue depth |
| Histogram | Distribution of values | Request latency buckets |
| Timer | Duration histogram | DB query time |

**Spring Boot Actuator + Micrometer:**
```java
// application.properties: management.endpoints.web.exposure.include=health,metrics,prometheus
Counter orderCounter = Counter.builder("orders.created").register(registry);
orderCounter.increment();
// Prometheus scrapes /actuator/prometheus; Grafana visualizes it
```

**Key metrics for Java apps:** JVM heap/GC, HTTP request rate/error rate/p99 latency, DB connection pool, business metrics (orders/min).

---

### 7.3 Monitoring

Collecting, visualizing, and alerting on metrics to ensure the system is healthy.

**SLO / SLA / SLI:**

| Term | Meaning | Example |
|---|---|---|
| SLI | The metric you measure | p99 latency = 200ms |
| SLO | Target for the SLI | p99 < 500ms, 99.9% of the time |
| SLA | Legal contract based on SLOs | Breach = customer credit |
| Error Budget | How much failure is allowed within the SLO | 0.1% of requests/month |

**Two frameworks:** **USE** (infrastructure — Utilization, Saturation, Errors) and **RED** (services — Rate, Errors, Duration).

**Alerting:** alert on symptoms (high error rate, high latency), not causes (high CPU). Every alert should be actionable.

---

### 7.4 Distributed Tracing

Follows a single request across all services, creating a trace — a timeline of every operation (each called a **span**).

```
TraceId: abc-123
├─ Span: POST /orders [order-service] 250ms
│    ├─ Span: SELECT * FROM products [DB] 15ms
│    ├─ Span: POST /payments [payment-service] 180ms
│    │    └─ Span: Charge credit card [Stripe] 150ms
│    └─ Span: PUBLISH order.created [Kafka] 5ms
```

**Spring Boot:** add `micrometer-tracing` and set `management.tracing.sampling.probability` (1.0 in dev, ~0.1 in prod). Spring auto-propagates traces through MVC, RestTemplate/WebClient, Kafka, and JPA.

**Observability Stack:**

| Pillar | Open Source | AWS |
|---|---|---|
| Logs | ELK or Loki + Grafana | CloudWatch Logs |
| Metrics | Prometheus + Grafana | CloudWatch Metrics |
| Traces | Jaeger or Zipkin | AWS X-Ray |

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

---

*Last Updated: 2026-06-18*
