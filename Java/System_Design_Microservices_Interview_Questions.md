# System Design & Microservices Interview Questions

> Even for junior roles, interviewers ask basic system design and microservices concepts!

---

## Table of Contents
1. [System Design Basics](#system-design-basics)
2. [Scalability Concepts](#scalability-concepts)
3. [Caching](#caching)
4. [Microservices vs Monolith](#microservices-vs-monolith)
5. [Microservices Core Concepts](#microservices-core-concepts)
6. [Spring Cloud Basics](#spring-cloud-basics)
7. [Message Queues (Kafka/RabbitMQ)](#message-queues-kafkarabbitmq)
8. [Quick Revision Summary](#quick-revision-summary)

---

## System Design Basics

### Q1: How do you approach a system design question?

```
Step 1: Clarify Requirements (2-3 min)
  - "How many users do we expect?"
  - "What's more important — read or write performance?"
  - "What is the expected uptime?"

Step 2: Estimate Scale (2 min)
  - Users: 1M DAU → ~100 reads/sec, ~10 writes/sec

Step 3: High Level Design (5-10 min)
  - Draw: Client → Load Balancer → Servers → Database

Step 4: Deep Dive (10-15 min)
  - Database choice, caching strategy, API design, bottlenecks

Step 5: Discuss Trade-offs
  - Availability vs Consistency, Speed vs Cost
```

---

### Q2: What is the difference between SQL and NoSQL? When to use which?

| | SQL (Relational) | NoSQL |
|--|-----------------|-------|
| **Schema** | Fixed | Flexible |
| **Scaling** | Vertical | Horizontal |
| **Consistency** | Strong (ACID) | Eventual (BASE) |
| **Examples** | MySQL, PostgreSQL | MongoDB, Redis, Cassandra |

**Use SQL when:** clear relationships, ACID transactions, complex JOINs.
**Use NoSQL when:** schema changes often, high write throughput, key-value/document data.

---

## Scalability Concepts

### Q3: Vertical vs Horizontal Scaling

```
Vertical (Scale Up): bigger server — simple, but has limits and single point of failure
Horizontal (Scale Out): more servers — unlimited scaling, needs load balancing
```

In Spring Boot: vertical = more JVM heap (`-Xmx4g`); horizontal = multiple instances behind a load balancer.

---

### Q4: What is a Load Balancer?

Receives all incoming requests and distributes them across servers so no single server is overwhelmed.

**Common algorithms:**
- **Round Robin** — servers take turns: 1, 2, 3, 1, 2, 3…
- **Least Connections** — next request goes to the least-busy server
- **IP Hash** — same client always hits the same server (sticky sessions)

---

### Q5: What is the CAP Theorem?

In a distributed system you can only guarantee **2 of 3**:
- **C**onsistency — all nodes see the same data
- **A**vailability — system always responds
- **P**artition Tolerance — system works despite network failures

Since network partitions are unavoidable, you really choose **C or A**.

| Database | Type | Trade-off |
|----------|------|-----------|
| MySQL, PostgreSQL | CP | May be unavailable during partition |
| Cassandra, DynamoDB | AP | Always available but may show stale data |

---

## Caching

### Q6: What is caching and how does Redis work?

Caching stores frequently accessed data in RAM so you skip the slow database call.

```
Without cache: request → DB (100ms)
With cache:    request → Redis (1ms) ✅ — DB only hit on cache miss
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableCaching
public class Application { ... }

@Service
public class UserService {

    @Cacheable(value = "users", key = "#id")  // cache on first call
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "users", key = "#user.id")  // evict on update
    public User updateUser(User user) {
        return userRepository.save(user);
    }
}
```

**Cache strategies:**

| Strategy | Description |
|----------|-------------|
| **Cache-Aside** | App checks cache first, loads from DB on miss (most common) |
| **Write-Through** | Write to cache and DB simultaneously |
| **Write-Behind** | Write to cache, async write to DB (high write throughput) |

---

## Microservices vs Monolith

### Q7: Monolith vs Microservices

```
MONOLITH: one app, one DB — all modules together
MICROSERVICES: each service is its own app with its own DB
```

| | Monolith | Microservices |
|--|----------|---------------|
| **Complexity** | Simple | Complex |
| **Scaling** | Scale everything | Scale individual services |
| **Tech** | Single stack | Different tech per service |
| **Debugging** | Easier | Harder (distributed tracing) |
| **Best for** | Small teams / startups | Large teams / complex systems |

---

## Microservices Core Concepts

### Q8: What is an API Gateway?

Single entry point for all client requests — routes to the right service, handles auth, rate limiting, and SSL.

```
Client → API Gateway (:80)
  ├── /api/users   → User Service (:8081)
  ├── /api/orders  → Order Service (:8082)
  └── /api/payments → Payment Service (:8083)
```

```yaml
# Spring Cloud Gateway — application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
```

---

### Q9: What is Service Discovery?

Services start/stop dynamically so hardcoded IPs break. Service Discovery (Eureka) acts as a registry — services register themselves, others look them up.

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication { ... }

// Any microservice registers as a client
@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication { ... }
```

```yaml
spring:
  application:
    name: user-service
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

---

### Q10: What is the Circuit Breaker pattern?

If a service keeps failing, the circuit "trips" (OPEN) and stops forwarding calls, returning a fallback instead. This prevents cascade failures.

```
CLOSED → normal calls
  ↓ (too many failures)
OPEN → return fallback immediately (no calls)
  ↓ (after timeout, try a few test calls)
HALF-OPEN → if calls succeed → CLOSED; if fail → OPEN
```

```java
@Service
public class UserService {

    @CircuitBreaker(name = "userService", fallbackMethod = "fallbackGetUser")
    public User getUserFromRemoteService(Long id) {
        return restTemplate.getForObject("http://USER-SERVICE/users/" + id, User.class);
    }

    public User fallbackGetUser(Long id, Exception e) {
        return new User(id, "Fallback User", "unavailable@email.com");
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        failureRateThreshold: 50      # open at 50% failure rate
        waitDurationInOpenState: 30s
```

---

## Spring Cloud Basics

### Q11: Spring Cloud key components

| Component | Purpose |
|-----------|---------|
| **Eureka** | Service Discovery — registry for all services |
| **Spring Cloud Gateway** | API Gateway — routing, auth, rate limiting |
| **Config Server** | Centralized config for all services |
| **Feign Client** | Declarative HTTP client for service-to-service calls |
| **Resilience4j** | Circuit breaker, retry, rate limiter |
| **Zipkin** | Distributed tracing |

```java
// Feign Client — call another service like a local method
@FeignClient(name = "order-service")
public interface OrderServiceClient {
    @GetMapping("/api/orders/user/{userId}")
    List<Order> getOrdersByUser(@PathVariable Long userId);
}

@Service
public class UserDashboardService {
    @Autowired
    private OrderServiceClient orderServiceClient;

    public UserDashboard getDashboard(Long userId) {
        User user = userRepository.findById(userId);
        List<Order> orders = orderServiceClient.getOrdersByUser(userId);
        return new UserDashboard(user, orders);
    }
}
```

---

## Message Queues (Kafka/RabbitMQ)

### Q12: What is a Message Queue and why is it used?

A message queue lets services communicate asynchronously — one service drops a message and another picks it up when ready, without waiting.

```
WITHOUT queue: Order Service → Email Service (must wait — slow!)
WITH queue:    Order Service → [Queue] → Email Service (picks up when ready)
               Order Service returns immediately; queue buffers if Email Service is down.
```

**When to use:** sending notifications, background processing, decoupling services, handling traffic spikes.

```java
// Producer (Order Service)
@Service
public class OrderService {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public Order placeOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        rabbitTemplate.convertAndSend("order-exchange", "order.placed",
            new OrderPlacedEvent(order.getId(), order.getUserEmail()));
        return order; // returns immediately
    }
}

// Consumer (Email Service)
@Service
public class EmailService {
    @RabbitListener(queues = "email-queue")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        emailSender.sendOrderConfirmation(event.getUserEmail(), event.getOrderId());
    }
}
```

---

## Quick Revision Summary

### Key System Design Concepts

| Concept | One-Line Explanation |
|---------|---------------------|
| **Vertical Scaling** | Bigger server |
| **Horizontal Scaling** | More servers |
| **Load Balancer** | Distributes requests across servers |
| **Cache (Redis)** | Store frequent data in RAM |
| **CDN** | Serve static files from servers near users |
| **CAP Theorem** | Pick 2 of: Consistency, Availability, Partition Tolerance |
| **Database Replication** | Master writes, replicas read |

### Microservices Patterns

| Pattern | Purpose |
|---------|---------|
| **API Gateway** | Single entry point, routing, auth |
| **Service Discovery** | Find services dynamically (Eureka) |
| **Circuit Breaker** | Stop calls to failing service (Resilience4j) |
| **Event-Driven** | Services communicate via message queue |

### Interview Tips

1. **Draw diagrams** — always sketch the system when explaining
2. **Mention trade-offs** — "X trades off Y for Z"
3. **Start simple** — monolith first, evolve to microservices as needed
4. **Key phrase** — "It depends on the requirements" shows maturity

### Simple System Design Template
```
1. Client (Browser/Mobile)
2. DNS → CDN (static files)
3. Load Balancer
4. Application Servers (Spring Boot)
5. Cache (Redis)
6. Database (MySQL/PostgreSQL)
7. Message Queue (Kafka/RabbitMQ — async tasks)
```
