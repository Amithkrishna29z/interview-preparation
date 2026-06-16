# System Design & Microservices Interview Questions

> 🎯 Even for junior roles, interviewers ask basic system design and microservices concepts!

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

**Easy Explanation:** Think of designing a system like planning a city. You need roads (communication), buildings (services), warehouses (storage), and traffic management (load balancing).

**Step-by-step approach for interviews:**

```
Step 1: Clarify Requirements (2-3 min)
  - "How many users do we expect?"
  - "What's more important — read or write performance?"
  - "Do we need real-time updates?"
  - "What is the expected uptime? 99.9%?"

Step 2: Estimate Scale (2 min)
  - Users: 1 million DAU (daily active users)
  - Reads: 10 million/day → 100 reads/second
  - Writes: 1 million/day → 10 writes/second
  - Data: 1KB per record × 1M records = 1GB

Step 3: High Level Design (5-10 min)
  - Draw components: Client → Load Balancer → Servers → Database
  - Identify main services

Step 4: Deep Dive (10-15 min)
  - Database choice (SQL vs NoSQL)
  - Caching strategy
  - API design
  - Handle bottlenecks

Step 5: Discuss Trade-offs
  - Availability vs Consistency
  - Speed vs Cost
  - Simplicity vs Scalability
```

---

### Q2: What is the difference between SQL and NoSQL? When to use which?

| | SQL (Relational) | NoSQL |
|--|-----------------|-------|
| **Data** | Structured (tables) | Flexible (documents, key-value) |
| **Schema** | Fixed, rigid | Dynamic, flexible |
| **Relationships** | Strong (foreign keys) | Weak (manual) |
| **Scaling** | Vertical (bigger server) | Horizontal (more servers) |
| **Consistency** | Strong ACID | Eventually consistent (BASE) |
| **Examples** | MySQL, PostgreSQL | MongoDB, Redis, Cassandra |

**Use SQL when:**
- Data has clear relationships (users → orders → products)
- Need ACID transactions (banking, payments)
- Complex queries with JOINs
- Data structure won't change much

**Use NoSQL when:**
- Schema changes frequently (startup phase)
- Need to store huge volumes of data at scale
- Need high write throughput (logs, events)
- Data is hierarchical/document-like (user profiles, product catalogs)
- Need low-latency key-value storage (caching, sessions)

---

## Scalability Concepts

### Q3: What is the difference between vertical and horizontal scaling?

**Easy Explanation:**

```
Vertical Scaling (Scale Up) = Buy a bigger/better server
  [Small Server] → [Big Server]
  Pros: Simple, no code changes
  Cons: Has limits, expensive, single point of failure

Horizontal Scaling (Scale Out) = Add more servers
  [Server] → [Server 1] [Server 2] [Server 3]
  Pros: Unlimited scaling, fault tolerant
  Cons: More complex, need load balancing
```

**In Java/Spring Boot context:**
- **Vertical:** Give the JVM more memory (`-Xmx4g`), more CPU cores
- **Horizontal:** Run multiple Spring Boot instances behind a load balancer

---

### Q4: What is a Load Balancer?

**Easy Explanation:** A load balancer is like a traffic cop — it receives all incoming requests and distributes them evenly across multiple servers so no single server gets overwhelmed.

```
                    ┌──────────────┐
Client Requests → → │ Load Balancer│ → Server 1 (handles some requests)
                    │              │ → Server 2 (handles some requests)
                    └──────────────┘ → Server 3 (handles some requests)
```

**Load Balancing Algorithms:**
- **Round Robin** = Requests go to servers in order: 1, 2, 3, 1, 2, 3...
- **Least Connections** = Next request goes to server with fewest active connections
- **Weighted Round Robin** = Powerful servers get more requests
- **IP Hash** = Same client always goes to same server (sticky sessions)

---

### Q5: What is the CAP Theorem?

**Easy Explanation:** In a distributed system, you can only guarantee 2 out of 3:
- **C**onsistency — All nodes see the same data at the same time
- **A**vailability — System always responds to requests
- **P**artition Tolerance — System works even if network fails between nodes

```
        Consistency
           /\
          /  \
         / CP \
        /------\
       / CA   AP \
      /____________\
  Availability   Partition Tolerance
```

| Database | Type | Trade-off |
|----------|------|-----------|
| MySQL, PostgreSQL | CP | Consistent but may be unavailable during partition |
| Cassandra, DynamoDB | AP | Always available but may show stale data |
| Redis (single node) | CA | Consistent & available but no partition tolerance |

> 💡 **Partition tolerance is unavoidable** in distributed systems (networks DO fail), so you really choose between C and A.

---

## Caching

### Q6: What is caching and how does Redis work?

**Easy Explanation:** Caching is storing frequently accessed data in fast memory (RAM) so you don't hit the slow database every time.

**Problem without cache:**
```
User requests → App → Database (takes 100ms) → App → User
(Every request hits database = slow and expensive)
```

**With Redis cache:**
```
User requests → App → Redis Cache (takes 1ms) → User ✅
                         ↓ (cache miss - not in cache)
                      Database (100ms) → Store in Redis → User
                         (next request for same data = 1ms)
```

**Spring Boot + Redis:**
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
// application.properties
spring.redis.host=localhost
spring.redis.port=6379
spring.cache.type=redis

// Enable caching
@SpringBootApplication
@EnableCaching
public class Application { ... }

// Use @Cacheable to cache results
@Service
public class UserService {

    // First call → hit database, store in cache
    // Subsequent calls → return from cache (fast!)
    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    // When user is updated, invalidate the cache
    @CacheEvict(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }

    // Clear entire users cache
    @CacheEvict(value = "users", allEntries = true)
    public void clearAllUsersCache() { }
}
```

**Cache Strategies:**

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Cache-Aside** | App checks cache first, loads from DB on miss | Most common pattern |
| **Write-Through** | Write to cache AND database simultaneously | When you need consistency |
| **Write-Behind** | Write to cache, async write to DB | High write throughput |
| **Read-Through** | Cache fetches from DB automatically | Transparent to app |

---

## Microservices vs Monolith

### Q7: What is the difference between Monolith and Microservices?

**Easy Explanation:**

**Monolith** = One big app (like a shopping mall — everything in one building)
**Microservices** = Many small apps (like separate shops in a street)

```
MONOLITH:
┌──────────────────────────────────────────┐
│          Single Application              │
│  ┌────────┐ ┌────────┐ ┌────────┐       │
│  │ Users  │ │ Orders │ │Payment │       │
│  │ Module │ │ Module │ │ Module │       │
│  └────────┘ └────────┘ └────────┘       │
│           Single Database                │
└──────────────────────────────────────────┘

MICROSERVICES:
┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │  Order   │  │ Payment  │
│ Service  │  │ Service  │  │ Service  │
│  :8081   │  │  :8082   │  │  :8083   │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
 ┌───▼──┐      ┌───▼──┐      ┌───▼──┐
 │User  │      │Order │      │Pay   │
 │  DB  │      │  DB  │      │  DB  │
 └──────┘      └──────┘      └──────┘
```

**Comparison:**

| | Monolith | Microservices |
|--|----------|---------------|
| **Complexity** | Simple to build | Complex |
| **Deployment** | Deploy one thing | Deploy many things |
| **Scaling** | Scale everything | Scale individual services |
| **Tech** | Same language/tech | Different tech per service |
| **Teams** | One team | Multiple small teams |
| **Debugging** | Easier | Harder (distributed tracing) |
| **Startup** | Slow (whole app) | Fast (individual services) |
| **Best for** | Small teams, startups | Large teams, complex systems |

---

## Microservices Core Concepts

### Q8: What is an API Gateway?

**Easy Explanation:** API Gateway is the single entry point for all client requests. It routes to the right microservice, handles auth, rate limiting, etc.

```
Client (React/Mobile)
         ↓
    API Gateway (:80)
    ├── /api/users → User Service (:8081)
    ├── /api/orders → Order Service (:8082)
    ├── /api/payments → Payment Service (:8083)
    │
    Also handles:
    ├── Authentication (JWT verification)
    ├── Rate limiting
    ├── Load balancing
    └── SSL termination
```

```java
// Spring Cloud Gateway (application.yml)
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE    # lb = load balancer
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1

        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
```

---

### Q9: What is Service Discovery?

**Easy Explanation:** In microservices, services start/stop dynamically. Service Discovery is like a phone directory — services register themselves, and others look them up to communicate.

```
Order Service wants to call User Service:

Without Discovery: Order Service must know IP:PORT of User Service
                  What if User Service moves? ❌ Hardcoded IPs break!

With Eureka Discovery:
1. User Service starts → registers with Eureka ("I'm User Service at 192.168.1.5:8081")
2. Order Service asks Eureka: "Where is User Service?"
3. Eureka responds: "192.168.1.5:8081"
4. Order Service calls User Service
```

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication { ... }

// Eureka Client (any microservice)
@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication { ... }

// application.yml for client
spring:
  application:
    name: user-service   # Name registered in Eureka

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

---

### Q10: What is Circuit Breaker pattern?

**Easy Explanation:** Like a real circuit breaker in your house — if too many failures happen, it "trips" and stops calling the failing service to prevent cascade failures.

**States:**
```
CLOSED → Normal operation, calls go through
  ↓ (too many failures)
OPEN → Stop all calls, return fallback immediately
  ↓ (after timeout)
HALF-OPEN → Try a few calls to see if service recovered
  ↓ (success)      ↓ (failure)
CLOSED            OPEN
```

```java
// Using Resilience4j (Spring Boot)
@Service
public class UserService {

    @CircuitBreaker(name = "userService", fallbackMethod = "fallbackGetUser")
    public User getUserFromRemoteService(Long id) {
        // Call remote service that might fail
        return restTemplate.getForObject("http://USER-SERVICE/users/" + id, User.class);
    }

    // Fallback method called when circuit is OPEN
    public User fallbackGetUser(Long id, Exception e) {
        return new User(id, "Fallback User", "unavailable@email.com");
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        failureRateThreshold: 50        # Open circuit at 50% failure rate
        waitDurationInOpenState: 30s    # Wait 30s before trying HALF-OPEN
        permittedCallsInHalfOpenState: 3  # Allow 3 test calls in HALF-OPEN
```

---

## Spring Cloud Basics

### Q11: What is Spring Cloud and its key components?

| Component | Purpose | Simple Analogy |
|-----------|---------|----------------|
| **Eureka** | Service Discovery | Phone directory for services |
| **Spring Cloud Gateway** | API Gateway | Security guard & traffic director |
| **Config Server** | Centralized configuration | Shared settings file for all services |
| **Feign Client** | Easier HTTP calls between services | Simplified REST client |
| **Zipkin** | Distributed tracing | GPS tracker for requests |
| **Resilience4j** | Circuit breaker, retry, rate limit | Safety net for failures |

```java
// Feign Client — call another microservice easily
@FeignClient(name = "order-service")  // "order-service" is Eureka name
public interface OrderServiceClient {

    @GetMapping("/api/orders/user/{userId}")
    List<Order> getOrdersByUser(@PathVariable Long userId);
}

// Use in UserService
@Service
public class UserDashboardService {

    @Autowired
    private OrderServiceClient orderServiceClient;

    public UserDashboard getDashboard(Long userId) {
        User user = userRepository.findById(userId);
        List<Order> orders = orderServiceClient.getOrdersByUser(userId); // Calls Order Service!
        return new UserDashboard(user, orders);
    }
}
```

---

## Message Queues (Kafka/RabbitMQ)

### Q12: What is a Message Queue and why is it used?

**Easy Explanation:** Message queue = mailbox between services. One service drops a message, another picks it up when ready. They don't need to wait for each other!

```
WITHOUT Message Queue (Direct/Synchronous):
Order Service → calls → Email Service (must wait for response)
If Email Service is slow → Order Service is slow too!

WITH Message Queue (Asynchronous):
Order Service → sends message → [Queue] → Email Service picks it up
Order Service continues immediately, doesn't wait!
Even if Email Service is down → messages wait in queue!
```

**When to use:**
- Send notifications (email, SMS) after order placed
- Process images/videos in background
- Decouple services (order from notification, payment from shipping)
- Handle traffic spikes (buffer requests)

```java
// Spring Boot + RabbitMQ example

// Producer (Order Service) - sends message
@Service
public class OrderService {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public Order placeOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));

        // Send async message — don't wait!
        rabbitTemplate.convertAndSend(
            "order-exchange",
            "order.placed",
            new OrderPlacedEvent(order.getId(), order.getUserEmail())
        );

        return order; // Return immediately!
    }
}

// Consumer (Email Service) - receives message
@Service
public class EmailService {

    @RabbitListener(queues = "email-queue")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Process when ready
        emailSender.sendOrderConfirmation(event.getUserEmail(), event.getOrderId());
    }
}
```

---

## Quick Revision Summary

### 🔑 Key System Design Concepts

| Concept | One-Line Explanation |
|---------|---------------------|
| **Vertical Scaling** | Bigger server |
| **Horizontal Scaling** | More servers |
| **Load Balancer** | Distributes requests across servers |
| **Cache (Redis)** | Store frequent data in RAM |
| **CDN** | Serve static files from servers near users |
| **Database Sharding** | Split database across multiple servers |
| **Database Replication** | Master writes, slaves read |
| **CAP Theorem** | Pick 2 of: Consistency, Availability, Partition Tolerance |

### 🔑 Microservices Patterns

| Pattern | Purpose |
|---------|---------|
| **API Gateway** | Single entry point, routing, auth |
| **Service Discovery** | Find services dynamically (Eureka) |
| **Circuit Breaker** | Stop calls to failing service (Resilience4j) |
| **Saga Pattern** | Distributed transactions across services |
| **Event-Driven** | Services communicate via message queue |

### 📝 Interview Tips

1. **For junior roles**: Focus on concepts, not implementation details
2. **Draw diagrams**: Always draw the system when explaining
3. **Mention trade-offs**: "We could use X but it trades off Y for Z"
4. **Start simple**: Monolith first, then evolve to microservices as needed
5. **Key phrase**: "It depends on the requirements" — shows maturity

### 🏗️ Simple System Design Template
```
1. Client (Browser/Mobile)
2. DNS (resolve domain to IP)
3. CDN (static files — images, JS, CSS)
4. Load Balancer (distribute traffic)
5. Application Servers (Spring Boot)
6. Cache (Redis — hot data)
7. Database (MySQL/PostgreSQL)
8. Object Storage (S3 — files, images)
9. Message Queue (Kafka — async tasks)
10. Monitoring (logs, metrics, alerts)
```
