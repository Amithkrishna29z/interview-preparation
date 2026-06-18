# World-Class Software Engineer — Complete Mastery Roadmap

> **Goal**: Become a strong software engineer through deliberate, phased learning.
> Follow this roadmap with discipline — fundamentals first, then systems, then scale.

---

## 1. Mindset

| Principle | What It Means |
|-----------|--------------|
| **First-principles thinking** | Don't just accept "it works." Ask *why* it works. |
| **Embrace discomfort** | Easy problems don't grow you. Seek hard problems daily. |
| **Own your mistakes** | Debug your own code. Never blame the framework. |
| **Systems thinking** | Always ask how a change affects the whole system. |
| **Write to think** | If you can't explain it, you don't know it. |

---

## 2. Phase 1 — Foundation (Months 1–6)

### 2.1 Programming Language Mastery (Java)

#### Junior Core — Know These Well
- [ ] 8 primitive types + wrappers; autoboxing pitfalls; String pool/interning
- [ ] All Collection implementations and their Big-O; HashMap internals (hash, collisions, resize)
- [ ] ConcurrentHashMap vs synchronized HashMap
- [ ] Streams (lazy evaluation, pitfalls); functional interfaces (Function/Predicate/Consumer/Supplier)
- [ ] Optional — correct use vs anti-patterns; generics (wildcards, bounded types, type erasure)
- [ ] Future / CompletableFuture / ExecutorService basics
- [ ] Records, Sealed classes, Pattern matching (modern Java)

#### Awareness (grow into these)
JVM internals (bytecode, class loading, JIT); GC choices (G1/ZGC); Java Memory Model (visibility, happens-before); Reflection; custom annotations; Java modules.

#### JVM Memory Layout
```
JVM Memory
├── Heap
│   ├── Young Generation (Eden + S0 + S1)  ← New objects born here
│   └── Old Generation                     ← Long-lived objects promoted here
├── Metaspace                              ← Class metadata (Java 8+)
├── Stack (per thread)                     ← Stack frames, local variables
└── PC Register (per thread)
```

#### Java Concurrency Essentials
```java
private volatile boolean running = true;          // visibility, not atomicity
private AtomicInteger counter = new AtomicInteger(0);  // atomic + visible

CompletableFuture
    .supplyAsync(() -> fetchData())
    .thenApplyAsync(data -> process(data))
    .thenAccept(result -> save(result))
    .exceptionally(ex -> handleError(ex));
```

---

### 2.2 Data Structures — Know and Be Able to Implement

| Structure | Mechanism | Key Operations | Use Case |
|-----------|-----------|----------------|----------|
| Dynamic Array | Contiguous memory + resize | O(1) get, O(n) insert mid | ArrayList |
| Linked List | Node + next/prev | O(1) head insert, O(n) search | Queue base, LRU |
| Stack / Queue | Array or LinkedList | O(1) push/pop/enqueue | BFS, undo |
| Hash Map | Array + linked list + hash fn | O(1) avg get/put | Key-value store |
| Binary Search Tree | Node + left + right | O(log n) avg | Ordered data |
| Min/Max Heap | Complete binary tree as array | O(log n) insert, O(1) peek | Priority Queue |
| Trie | Tree of characters | O(m) where m = word length | Autocomplete |
| Graph | Adjacency list / matrix | Varies | Networks, dependencies |
| Union-Find | Forest of trees | ~O(1) amortized | Connectivity |

**HashMap core mechanics:** `index = hash & (table.length - 1)`, linked-list per bucket for collisions, resize (double) when `size > capacity * 0.75`.

---

### 2.3 Object-Oriented Design — SOLID

**S — Single Responsibility**: each class has one reason to change. Split `UserService` into `UserValidator`, `UserRepository`, `UserEmailer`.

**O — Open/Closed**: extend via new implementations, not by modifying existing code. Use interfaces/polymorphism instead of growing `instanceof` chains.

**L — Liskov Substitution**: a subtype must be usable wherever its base type is expected. Classic violation: `Square extends Rectangle` that overrides `setWidth` to also change height.

**I — Interface Segregation**: many small interfaces beat one fat interface. A `Robot` shouldn't be forced to implement `eat()`.

**D — Dependency Inversion**: depend on abstractions, not concretes. Inject an `OrderRepository` interface — exactly what Spring DI does.

---

## 3. Phase 2 — Core Engineering (Months 6–18)

### 3.1 Design Patterns

**Singleton** — one instance; enum form is simplest thread-safe approach.

**Builder** — avoid telescoping constructors; chain fluent setters returning `this`, then `build()` an immutable object.

**Factory Method** — centralize object creation behind one method returning the interface type.

**Decorator** — wrap an object to add behavior without changing it; decorators compose.

**Proxy** — stand-in implementing the same interface that adds caching, lazy loading, or access control transparently.

**Strategy** — swap interchangeable algorithms behind one interface at runtime.

**Observer** — subject keeps a list of listeners and notifies them on change. Spring's `ApplicationEventPublisher`/`@EventListener` is this pattern built in.

---

### 3.2 REST API Design Rules

```
1. Nouns for resources, not verbs
   RIGHT: GET /users/{id}, POST /orders

2. HTTP method semantics
   GET    → safe + idempotent
   POST   → creates new resource
   PUT    → full update (idempotent)
   PATCH  → partial update (idempotent)
   DELETE → delete (idempotent)

3. Status codes
   200 OK, 201 Created, 204 No Content
   400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
   409 Conflict, 422 Unprocessable, 429 Too Many Requests, 500 Internal Error

4. Version your API: /api/v1/users

5. Consistent error body:
   { "error": { "code": "VALIDATION_ERROR", "message": "...", "field": "email" } }
```

**Idempotency keys**: client sends `Idempotency-Key: uuid` on POST; server stores (key → response) for 24 h to prevent duplicate processing on retry.

---

### 3.3 Testing

**Test Pyramid**: many fast unit tests → some integration tests (real DB) → few slow E2E tests.

```java
// AAA pattern: Arrange, Act, Assert
@Test
void shouldThrowExceptionWhenEmailAlreadyExists() {
    when(userRepository.findByEmail("john@example.com")).thenReturn(Optional.of(existingUser));

    assertThatThrownBy(() -> userService.register(new RegisterRequest("john@example.com", "John2")))
        .isInstanceOf(EmailAlreadyExistsException.class);

    verify(userRepository, never()).save(any());
}
// Name tests: should[ExpectedBehavior]When[Condition]
// Test edge cases: null, empty, boundary values
```

*Contract testing (Pact)*: consumer declares expected response shape; provider verifies its endpoint matches — lets services deploy independently.

---

## 4. Phase 3 — Advanced Systems (Months 18–36)

### 4.1 Concurrency — Key Primitives

```java
// ThreadLocal — per-thread isolation (Spring request context)
private static final ThreadLocal<User> currentUser = new ThreadLocal<>();
currentUser.set(user);       // MUST call remove() to avoid memory leak in thread pools

// Semaphore — limit concurrent access
Semaphore semaphore = new Semaphore(10);
semaphore.acquire();
try { doWork(); } finally { semaphore.release(); }

// CountDownLatch — wait for N tasks to complete
CountDownLatch latch = new CountDownLatch(3);
executor.submit(() -> { doWork(); latch.countDown(); });
latch.await();
```

**Virtual Threads (Java 21)**: `Executors.newVirtualThreadPerTaskExecutor()` — cheap, can have millions; blocking I/O parks the virtual thread and frees the platform thread.

*Reactive (WebFlux)*: awareness — `Mono`/`Flux` free threads during I/O. Most junior Spring Boot work stays on blocking MVC; Java 21 virtual threads now give similar scalability with simpler code.

---

### 4.2 Spring Boot — Production Essentials

```yaml
server:
  shutdown: graceful
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
management:
  endpoints:
    web: { exposure: { include: health,metrics,prometheus } }
```

**Global exception handler** with `@RestControllerAdvice`: map `MethodArgumentNotValidException` → 400, `ResourceNotFoundException` → 404, `Exception` → 500 (never expose stack traces to clients).

**Circuit Breaker (Resilience4j)**: wrap flaky downstream calls with `@CircuitBreaker` + fallback. States: CLOSED → OPEN (fail fast) → HALF_OPEN (test recovery) → CLOSED.

---

## 5. Phase 4 — Mastery & Leadership (Year 3+)

*Awareness:* at senior/staff level every architectural choice is a trade-off (consistency vs availability, sync vs async, blue-green vs canary). You don't need this yet; just know trade-offs exist.

---

## 6. DSA Mastery

### Algorithm Complexity Reference

| Algorithm | Average | Worst | Space |
|-----------|---------|-------|-------|
| QuickSort | O(n log n) | O(n²) | O(log n) |
| MergeSort | O(n log n) | O(n log n) | O(n) |
| BFS/DFS | O(V+E) | O(V+E) | O(V) |
| Dijkstra | O((V+E) log V) | same | O(V) |
| Binary Search | O(log n) | O(log n) | O(1) |

### Core Algorithm Patterns

**Two Pointers** — sorted array pair sum: move left/right inward. O(n) time, O(1) space.

**Sliding Window** — longest substring without repeating chars: expand right, shrink left on duplicate. O(n).

**Fast & Slow Pointers (Floyd's Cycle)** — detect cycle in linked list: fast moves 2 steps, slow 1; cycle if they meet. O(n), O(1).

**Binary Search on Answer** — search the answer space (not the array) when you can check `canShip(capacity)`. Template: `left = min possible, right = max possible, find smallest valid mid`.

**Dynamic Programming** — template: (1) define `dp[i]` = answer for first i elements; (2) write recurrence; (3) set base case; (4) build bottom-up or memoize top-down.

**Backtracking** — template: choose → explore → unchoose (undo). Use for permutations, combinations, subsets.

**Graph BFS** — shortest path (unweighted): queue + visited array + `{row, col, distance}`. O(V+E).

---

## 7. System Design

### Numbers Every Engineer Should Know

```
L1 cache: 1 ns        RAM: 100 ns          SSD random: 100 μs
Network same DC: 500 μs                    Network cross-continent: 150 ms

Availability:  99.9% = 8.76 h/year downtime   99.99% = 52.6 min/year
```

### System Design Interview Framework

```
1. CLARIFY (5 min) — functional requirements, scale (DAU, QPS), latency, consistency
2. ESTIMATE (3 min) — reads/s, writes/s, storage size, bandwidth
3. HIGH-LEVEL (10 min) — client → LB → servers → DB; core components; API contracts
4. DEEP DIVE (15 min) — schema, bottlenecks, failure modes, component trade-offs
5. SCALE (5 min) — caching, sharding, CDN, async processing
```

### Caching Strategies

| Strategy | How | Pros | Cons |
|----------|-----|------|------|
| Cache-aside | App checks cache; miss → fetch DB + fill cache | Only caches reads; resilient to cache failure | First request slow; stale data risk |
| Write-through | Write to cache AND DB synchronously | Always up to date | Write penalty |
| Write-behind | Write to cache; async flush to DB | Very fast writes | Data loss if cache crashes |

Eviction policies: LRU, LFU, TTL, FIFO.

### Database Sharding *(awareness)*
Horizontal sharding splits rows by shard key (user_id, tenant_id). Strategies: range-based, hash-based, consistent hashing (ring — adding a server moves only a small slice of keys).

---

## 8. Architecture Patterns

### Microservices — When and When Not

```
Use microservices when: large teams (10+), independent scaling needs, clear bounded contexts.
Avoid when: small team (<5), no domain clarity, tight coupling (= distributed monolith).
Decompose by: business capability (OrderService, PaymentService) or DDD bounded context.
```

### Event-Driven Architecture *(awareness)*
- **Choreography**: services react to events (loose coupling, hard to trace).
- **Orchestration/Saga**: orchestrator issues commands step by step (clear flow, orchestrator complexity).
- **Outbox pattern**: write domain row + outbox row in the same DB transaction; worker publishes to Kafka — no events lost on crash.

### Domain-Driven Design *(awareness)*
Key building blocks: **Entity** (has identity, mutable), **Value Object** (no identity, immutable — e.g. Money), **Aggregate** (cluster with one root), **Domain Event**, **Repository** (collection abstraction per aggregate root).

---

## 9. Database Mastery

### SQL Optimization Rules

```sql
-- Use EXPLAIN ANALYZE to find Seq Scans (missing index) vs Index Scans
CREATE INDEX idx_users_email ON users(email);  -- B-tree default

-- Rules:
1. SELECT only needed columns
2. Avoid functions on indexed columns in WHERE:
   BAD:  WHERE YEAR(created_at) = 2026
   GOOD: WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
3. Use EXISTS instead of IN for subqueries
4. Avoid N+1 queries — use JOINs or batch loading
```

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Default In |
|-------|-----------|---------------------|--------------|------------|
| READ UNCOMMITTED | Yes | Yes | Yes | — |
| READ COMMITTED | No | Yes | Yes | PostgreSQL |
| REPEATABLE READ | No | No | Yes | MySQL InnoDB |
| SERIALIZABLE | No | No | No | — |

Pessimistic locking: `SELECT ... FOR UPDATE`. Optimistic locking: check `version` column on update, retry on conflict.

---

## 10. Distributed Systems

### CAP Theorem

```
Network partitions WILL happen. During a partition, choose:
  CP: refuse to serve (return error) to avoid stale data  → HBase, Zookeeper
  AP: serve potentially stale data to stay available       → Cassandra, DynamoDB
```

*PACELC extends CAP*: even without partitions, higher consistency → more coordination → higher latency.

### Distributed Patterns *(awareness)*
- **Saga**: chain local transactions with compensating rollbacks for cross-service consistency.
- **Distributed lock**: Redis RedLock (acquire on majority of nodes with TTL); prefer Zookeeper for critical locks.
- **Consensus (Raft)**: leader elected, replicates to majority before commit — used in etcd, CockroachDB. As a junior you *use* these systems, not implement the algorithm.

---

## 11. Security Engineering

### OWASP Top 10

| # | Vulnerability | Prevention |
|---|---------------|------------|
| A01 | Broken Access Control | Check authorization server-side on every request |
| A02 | Cryptographic Failures | TLS 1.3, AES-256-GCM, bcrypt for passwords |
| A03 | Injection (SQL, LDAP, OS) | Parameterized queries, input validation |
| A04 | Insecure Design | Threat modeling in design phase |
| A05 | Security Misconfiguration | Disable defaults, least privilege |
| A06 | Vulnerable Components | Snyk / OWASP Dependency Check |
| A07 | Auth & Session Failures | MFA, secure sessions, account lockout |
| A08 | Software & Data Integrity | Verify signatures, secure CI/CD |
| A09 | Security Logging Failures | Log auth events; never log passwords/tokens |
| A10 | SSRF | Allowlist external URLs, block internal IP ranges |

### Secure Coding Essentials

```java
// Parameterized queries — NEVER concatenate user input into SQL
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
stmt.setString(1, email);

// Password hashing — bcrypt with cost factor 12
String hash = BCrypt.hashpw(password, BCrypt.gensalt(12));

// CORS — explicit whitelist, never "*" in production
registry.addMapping("/api/**").allowedOrigins("https://myapp.com");

// Rate limiting — @RateLimiter on login endpoints to prevent brute force
```

---

## 12. Performance Engineering

### Test Types to Know
- **Load**: simulate expected load — verify SLA.
- **Stress**: push beyond limits — find breaking point.
- **Soak**: run at normal load for hours — find memory leaks.
- **Spike**: sudden traffic burst.

### Common Java Memory Leaks
```java
// 1. Static collections that grow unbounded — never cleared
// 2. Listeners not removed — caller stays referenced
// 3. ThreadLocal not removed in thread pools — MUST call threadLocal.remove()
```

*Tools (awareness)*: async-profiler / Java Flight Recorder for CPU/memory profiling; `jmap`/`jcmd` for heap dumps; JMH for microbenchmarks.

---

## 13. DevOps & Platform Engineering

### Docker — Optimized Java Image
```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml . && COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```
Key: multi-stage (JDK build → JRE runtime = smaller image), non-root user, HEALTHCHECK, `UseContainerSupport` (JVM respects cgroup limits).

### Kubernetes Core Concepts *(awareness)*
Know: **Pod** (one container group), **Deployment** (manages N replicas + rolling updates), **Service** (stable DNS/IP), **ConfigMap/Secret** (config injection). Resources: `requests` = guaranteed, `limits` = cap. Probes: `readinessProbe` (traffic gate), `livenessProbe` (restart trigger).

### CI/CD Pipeline *(awareness)*
Typical pipeline: **test** → **security-scan** (OWASP) → **build-and-push** Docker image tagged with commit SHA → **deploy** (`kubectl set image` + `kubectl rollout status`). Later stages `needs:` earlier ones; deploys gated to `main`.

---

## 14. Code Craft

### The Rules

```
1. Make it work → Make it right → Make it fast (in that order).
2. Code is read 10x more than written — write for the reader.
3. Comments explain WHY, not WHAT. If you need WHAT → rename things.
4. Functions do ONE thing. If you use "and" to describe it → split it.
5. Fail fast: validate inputs at boundaries, throw descriptive exceptions.
6. Immutability by default: prefer final fields, immutable value objects.
```

### Code Smell Reference

| Smell | Refactoring |
|-------|-------------|
| Long method (>20 lines) | Extract Method |
| Long parameter list (>3) | Introduce Parameter Object |
| God class (>500 lines) | Extract Class |
| Primitive obsession (String for email) | Replace Primitive with Object |
| Switch on type | Replace with Polymorphism |
| Duplicate code | Extract Method/Superclass |
| Speculative generality | Delete it — YAGNI |

---

## 15. Engineering Leadership

### Code Review

**As Author**: PRs < 400 lines; describe WHY not WHAT; self-review first; link to ticket; add test evidence.

**As Reviewer**: review within 24 h; distinguish blocking vs `nit:`; explain WHY; ask questions ("Have you considered X?").

### Technical Documentation
Junior essentials: a **README** that runs the app in < 5 commands; **OpenAPI/Swagger** auto-generated from annotations.
*Awareness*: ADRs (Architecture Decision Records), C4 diagrams, incident runbooks.

### Incident Management *(awareness)*
Acknowledge fast → **mitigate before diagnosing** (rollback/scale/redirect) → communicate every 15–30 min → blameless post-mortem with **5 Whys** → action items with owners.

---

## 16. Projects That Prove Mastery

### Tier 1 — Foundation (Months 1–6)
1. **URL Shortener** — REST API + PostgreSQL + Redis caching + deployment
2. **Task Management API** — Full CRUD + JWT auth + role-based access
3. **Real-time Chat** — WebSocket + Spring Boot + message persistence

### Tier 2 — Intermediate (Months 6–18)
4. **E-commerce System** — Orders, inventory, payments, email notifications
5. **Social Media Feed** — Newsfeed, follow system, caching strategy
6. **File Storage Service** — S3-like API, chunked uploads, streaming

### Tier 3 — Advanced (Year 2+)
7. **Distributed Job Queue** — Kafka + workers + retry + dead-letter queue
8. **Search Engine** — Inverted index, relevance ranking
9. **Database Engine** — Simple B-tree, write-ahead log, query parser

### Every Project Must Have
```
✅ Unit + integration tests (>80% coverage)
✅ Docker + docker-compose
✅ CI/CD pipeline (GitHub Actions)
✅ Deployed to cloud (AWS/GCP/Azure)
✅ Monitoring (Prometheus + Grafana)
✅ Structured logging
✅ README with architecture diagram
```

---

## 17. Essential Reading

### Tier 1 — Read These First
| Book | Why |
|------|-----|
| **Clean Code** — Martin | Writing readable, maintainable code |
| **The Pragmatic Programmer** — Hunt & Thomas | Mindset and habits of elite engineers |
| **Designing Data-Intensive Applications** — Kleppmann | Best book on distributed systems and databases |
| **Refactoring** — Fowler | How to continuously improve existing code |
| **A Philosophy of Software Design** — Ousterhout | Deep complexity management |

### Tier 2 / 3 — As You Grow *(awareness)*
*System Design Interview Vol 1 & 2* (Xu), *Domain-Driven Design* (Evans), *Building Microservices* (Newman), *Release It!* (Nygard), *CLRS* (algorithms), *The Staff Engineer's Path* (Reilly).

---

## 18. Daily Habits

```
Daily:   1 LeetCode medium; read 1 engineering blog post
Weekly:  pick one deep topic; write a short Friday summary; side project time
Annual:  Y1 language + DSA + framework · Y2 system design + cloud cert ·
         Y3 architecture + specialty · Y4+ blog/talks/OSS
```

**Platforms:** LeetCode/NeetCode, Martin Fowler blog, High Scalability, MIT OCW, MIT 6.824 (distributed systems).

---

## Summary — Roadmap Cheat Sheet

```
Month 1-3:   CS fundamentals, data structures, implement everything from scratch
Month 3-6:   Algorithms (150+ LeetCode), Java/OOP mastery
Month 6-12:  Spring Boot production-grade, REST API design, SQL optimization
Month 12-18: System design basics, microservices, messaging (Kafka), Docker/K8s
Month 18-24: Distributed systems, advanced caching, performance engineering
Month 24-36: Architecture, DDD, security engineering, cloud certifications
Year 3+:     Technical leadership, open source, engineering blog, talks
```

*World-class is a direction, not a destination. Keep moving.*

---

*Last Updated: 2026-06-18*
