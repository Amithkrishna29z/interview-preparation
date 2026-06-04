# World-Class Software Engineer — Complete Mastery Roadmap

> **Goal**: Become one of the best software engineers in the world.
> This is not a sprint — it is a multi-year, deliberate journey. Follow this roadmap with discipline.

---

## Table of Contents

1. [The Mindset of a World-Class Engineer](#1-mindset)
2. [Phase 1 — Foundation (Months 1–6)](#2-phase-1-foundation)
3. [Phase 2 — Core Engineering (Months 6–18)](#3-phase-2-core-engineering)
4. [Phase 3 — Advanced Systems (Months 18–36)](#4-phase-3-advanced-systems)
5. [Phase 4 — Mastery & Leadership (Year 3+)](#5-phase-4-mastery)
6. [Data Structures & Algorithms — Deep Mastery](#6-dsa-mastery)
7. [System Design — At Scale](#7-system-design)
8. [Software Architecture Patterns](#8-architecture)
9. [Database Mastery](#9-database-mastery)
10. [Distributed Systems](#10-distributed-systems)
11. [Security Engineering](#11-security)
12. [Performance Engineering](#12-performance)
13. [DevOps & Platform Engineering](#13-devops)
14. [Code Craft — Writing World-Class Code](#14-code-craft)
15. [Engineering Leadership](#15-leadership)
16. [Projects That Prove Mastery](#16-projects)
17. [Books Every World-Class Engineer Must Read](#17-books)
18. [Daily Habits of Elite Engineers](#18-habits)

---

## 1. Mindset

### The Two Things That Separate World-Class Engineers

**1. They obsess over fundamentals.**
The best engineers at Google, Amazon, and top-tier companies can explain why a HashMap is O(1) average, what happens during a TCP handshake, why SOLID principles matter, and how a CPU cache works — not because they memorized it, but because they *understand* it at a deep level.

**2. They build things and reflect.**
Reading alone builds zero skills. You must code, break things, debug, optimize, and repeat.

### Core Mindset Principles

| Principle | What It Means |
|-----------|--------------|
| **First-principles thinking** | Don't just accept "it works." Ask *why* it works. |
| **Embrace discomfort** | Easy problems don't grow you. Seek hard problems daily. |
| **Own your mistakes** | Debug your own code. Never blame the framework. |
| **Systems thinking** | Always ask how a change affects the whole system. |
| **Write to think** | Writing documentation forces clarity. If you can't explain it, you don't know it. |
| **Deliberate practice** | 10,000 hours of *deliberate, focused* practice — not passive repetition. |

---

## 2. Phase 1 — Foundation (Months 1–6)

### 2.1 Programming Language Mastery (Java / Your Primary Language)

You must know your primary language at a **compiler level** — not just syntax.

#### Java Deep Mastery Checklist
- [ ] JVM internals: bytecode, class loading, JIT compilation
- [ ] Memory model: heap, stack, metaspace, GC roots
- [ ] Garbage Collectors: Serial, Parallel, G1, ZGC — when to use each
- [ ] Java Memory Model (JMM): visibility, atomicity, ordering, happens-before
- [ ] All 8 primitive types and their wrapper classes
- [ ] String pool, string interning
- [ ] Autoboxing pitfalls and performance cost
- [ ] All Collection implementations and their Big-O guarantees
- [ ] HashMap internal: hash function, collision resolution, tree-ification, resize
- [ ] ConcurrentHashMap vs synchronized HashMap
- [ ] Java Streams: lazy evaluation, parallel streams, pitfalls
- [ ] CompletableFuture, Future, ExecutorService
- [ ] All functional interfaces: Function, Predicate, Consumer, Supplier, BiFunction
- [ ] Optional — correct use vs anti-patterns
- [ ] Java generics: wildcards, bounded types, type erasure
- [ ] Reflection API and its cost
- [ ] Annotations: built-in + creating custom annotations
- [ ] Java modules (Java 9+)
- [ ] Records (Java 16+), Sealed classes (Java 17+), Pattern matching

#### Key Java Concepts Deep Dive

**JVM Memory Layout**
```
JVM Memory
├── Heap
│   ├── Young Generation
│   │   ├── Eden Space         ← New objects born here
│   │   ├── Survivor S0
│   │   └── Survivor S1
│   └── Old Generation         ← Long-lived objects promoted here
├── Metaspace                  ← Class metadata (Java 8+ replaced PermGen)
├── Stack (per thread)         ← Stack frames, local variables
├── PC Register (per thread)   ← Current instruction pointer
└── Native Method Stack        ← C/C++ native method calls
```

**Garbage Collection — G1GC (Default in Java 9+)**
```
1. Minor GC: clears Eden + Survivors → promotes surviving objects to Old Gen
2. Concurrent Marking: marks live objects without stopping all threads
3. Mixed GC: collects both Young and selected Old Gen regions
4. Stop-the-World events are minimized but not eliminated

Tuning flags:
-Xms512m          → initial heap size
-Xmx4g            → max heap size
-XX:+UseG1GC      → enable G1
-XX:MaxGCPauseMillis=200 → target pause time
```

**Java Concurrency Model**
```java
// Volatile — guarantees visibility, not atomicity
private volatile boolean running = true;

// Atomic — both visible AND atomic
private AtomicInteger counter = new AtomicInteger(0);

// ReentrantLock — more flexible than synchronized
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock(); // always unlock in finally
}

// CompletableFuture — async non-blocking
CompletableFuture
    .supplyAsync(() -> fetchData())       // runs on ForkJoinPool
    .thenApplyAsync(data -> process(data))
    .thenAccept(result -> save(result))
    .exceptionally(ex -> handleError(ex));
```

---

### 2.2 Data Structures — Not Just Usage, But Implementation

You must be able to **implement every data structure from scratch**.

#### Must-Implement Data Structures

| Structure | Underlying Mechanism | Key Operations | Use Case |
|-----------|---------------------|----------------|----------|
| Dynamic Array | Contiguous memory + resize | O(1) get, O(n) insert mid | ArrayList |
| Singly Linked List | Node + next pointer | O(1) head insert, O(n) search | Queue base |
| Doubly Linked List | Node + prev + next | O(1) both ends | LRU Cache |
| Stack | Array or LinkedList | O(1) push/pop | Call stack, undo |
| Queue | LinkedList or circular array | O(1) enqueue/dequeue | BFS, task queue |
| Deque | Doubly-ended queue | O(1) both ends | Sliding window |
| Hash Map | Array + linked list + hash fn | O(1) avg get/put | Key-value store |
| Binary Search Tree | Node + left + right | O(log n) avg, O(n) worst | Ordered data |
| AVL Tree | Self-balancing BST | O(log n) guaranteed | DB indexes |
| Red-Black Tree | Self-balancing BST | O(log n) guaranteed | Java TreeMap |
| Min/Max Heap | Complete binary tree as array | O(log n) insert/delete, O(1) peek | Priority Queue |
| Trie | Tree of characters | O(m) where m=word length | Autocomplete, IP routing |
| Graph | Adjacency list / matrix | Varies | Networks, dependencies |
| Segment Tree | Binary tree over ranges | O(log n) range query + update | Range queries |
| Union-Find | Forest of trees | ~O(1) amortized | Connectivity |
| Bloom Filter | Bit array + hash functions | O(1) insert/lookup | Membership test |

#### Hash Map — Implementation From Scratch

```java
public class MyHashMap<K, V> {
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;
    
    private Node<K, V>[] table;
    private int size;
    
    static class Node<K, V> {
        K key; V value; int hash; Node<K, V> next;
        Node(K key, V value, int hash) { this.key = key; this.value = value; this.hash = hash; }
    }
    
    @SuppressWarnings("unchecked")
    public MyHashMap() { table = new Node[DEFAULT_CAPACITY]; }
    
    private int hash(K key) {
        int h = key.hashCode();
        return h ^ (h >>> 16); // spread higher bits into lower bits
    }
    
    private int index(int hash) { return hash & (table.length - 1); }
    
    public void put(K key, V value) {
        int hash = hash(key);
        int idx = index(hash);
        for (Node<K, V> n = table[idx]; n != null; n = n.next) {
            if (n.hash == hash && n.key.equals(key)) { n.value = value; return; }
        }
        Node<K, V> newNode = new Node<>(key, value, hash);
        newNode.next = table[idx];
        table[idx] = newNode;
        if (++size > table.length * LOAD_FACTOR) resize();
    }
    
    public V get(K key) {
        int hash = hash(key); int idx = index(hash);
        for (Node<K, V> n = table[idx]; n != null; n = n.next)
            if (n.hash == hash && n.key.equals(key)) return n.value;
        return null;
    }
    
    private void resize() {
        Node<K, V>[] oldTable = table;
        table = new Node[oldTable.length * 2]; size = 0;
        for (Node<K, V> head : oldTable)
            for (Node<K, V> n = head; n != null; n = n.next) put(n.key, n.value);
    }
}
```

---

### 2.3 Object-Oriented Design — Beyond Syntax

#### SOLID Principles — With Real Examples

**S — Single Responsibility Principle**
```java
// BAD: One class doing too much
class UserService {
    void createUser(User u) { /* ... */ }
    void sendWelcomeEmail(User u) { /* ... */ }   // email concern
    void saveToDatabase(User u) { /* ... */ }     // persistence concern
    void validateUser(User u) { /* ... */ }       // validation concern
}

// GOOD: Each class has one reason to change
class UserService { void createUser(User u) { validator.validate(u); repo.save(u); emailer.sendWelcome(u); } }
class UserValidator { void validate(User u) { /* validation rules */ } }
class UserRepository { void save(User u) { /* persistence */ } }
class UserEmailer { void sendWelcome(User u) { /* email */ } }
```

**O — Open/Closed Principle**
```java
// BAD: Must modify existing code to add new shape
class AreaCalculator {
    double area(Object shape) {
        if (shape instanceof Circle c) return Math.PI * c.radius * c.radius;
        if (shape instanceof Rectangle r) return r.width * r.height;
        // must keep modifying this method for new shapes
    }
}

// GOOD: Open for extension, closed for modification
interface Shape { double area(); }
class Circle implements Shape { double area() { return Math.PI * r * r; } }
class Rectangle implements Shape { double area() { return w * h; } }
class Triangle implements Shape { double area() { return 0.5 * b * h; } }
class AreaCalculator { double area(Shape s) { return s.area(); } } // never changes
```

**L — Liskov Substitution Principle**
```java
// BAD: Square breaks the contract of Rectangle
class Rectangle { int width, height; void setWidth(int w) { width = w; } void setHeight(int h) { height = h; } }
class Square extends Rectangle {
    void setWidth(int w) { width = w; height = w; } // VIOLATION: changes height unexpectedly
}
// Code expecting Rectangle will break if Square is used

// GOOD: Use composition or separate hierarchy
interface Shape { int area(); }
class Rectangle implements Shape { /* ... */ }
class Square implements Shape { /* ... */ }
```

**I — Interface Segregation Principle**
```java
// BAD: Fat interface forces unnecessary implementation
interface Worker { void work(); void eat(); void sleep(); }
class Robot implements Worker {
    void work() { /* ok */ }
    void eat() { throw new UnsupportedOperationException(); } // robots don't eat!
    void sleep() { throw new UnsupportedOperationException(); }
}

// GOOD: Segregated interfaces
interface Workable { void work(); }
interface Feedable { void eat(); }
interface Restable { void sleep(); }
class Human implements Workable, Feedable, Restable { /* ... */ }
class Robot implements Workable { void work() { /* ... */ } }
```

**D — Dependency Inversion Principle**
```java
// BAD: High-level depends on low-level concrete class
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository(); // hard dependency
}

// GOOD: Both depend on abstraction
interface OrderRepository { void save(Order o); Order findById(long id); }
class OrderService {
    private final OrderRepository repo; // depends on interface
    OrderService(OrderRepository repo) { this.repo = repo; } // inject dependency
}
class MySQLOrderRepository implements OrderRepository { /* ... */ }
class MongoOrderRepository implements OrderRepository { /* ... */ }
// Easy to swap, test with mocks, add new DB without changing OrderService
```

---

## 3. Phase 2 — Core Engineering (Months 6–18)

### 3.1 Design Patterns — Know When NOT to Use Them Too

#### Creational Patterns

**Singleton**
```java
// Thread-safe Singleton using enum (best approach)
public enum DatabaseConnection {
    INSTANCE;
    private final Connection conn;
    DatabaseConnection() { conn = createConnection(); }
    public Connection getConnection() { return conn; }
}

// Or: Double-checked locking
public class Config {
    private static volatile Config instance;
    private Config() {}
    public static Config getInstance() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) instance = new Config(); // double check
            }
        }
        return instance;
    }
}
```

**Builder Pattern**
```java
// Avoid telescoping constructors — use Builder
public class HttpRequest {
    private final String url; private final String method;
    private final Map<String, String> headers; private final String body;
    private final int timeout;
    
    private HttpRequest(Builder b) {
        this.url = b.url; this.method = b.method;
        this.headers = b.headers; this.body = b.body; this.timeout = b.timeout;
    }
    
    public static class Builder {
        private final String url; private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body; private int timeout = 30;
        
        public Builder(String url) { this.url = url; }
        public Builder method(String m) { this.method = m; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder body(String b) { this.body = b; return this; }
        public Builder timeout(int t) { this.timeout = t; return this; }
        public HttpRequest build() { return new HttpRequest(this); }
    }
}

// Usage — readable and fluent
HttpRequest req = new HttpRequest.Builder("https://api.example.com/users")
    .method("POST").header("Content-Type", "application/json")
    .body("{\"name\":\"John\"}").timeout(60).build();
```

**Factory Method**
```java
interface Notification { void send(String msg); }
class EmailNotification implements Notification { public void send(String msg) { /* send email */ } }
class SMSNotification implements Notification { public void send(String msg) { /* send sms */ } }
class PushNotification implements Notification { public void send(String msg) { /* push */ } }

class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SMSNotification();
            case "PUSH"  -> new PushNotification();
            default      -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}
```

#### Structural Patterns

**Decorator Pattern**
```java
interface DataSource { void write(String data); String read(); }
class FileDataSource implements DataSource { /* basic file ops */ }

// Each decorator adds behavior without changing others
class EncryptionDecorator implements DataSource {
    private final DataSource wrapped;
    EncryptionDecorator(DataSource ds) { wrapped = ds; }
    public void write(String data) { wrapped.write(encrypt(data)); }
    public String read() { return decrypt(wrapped.read()); }
}

class CompressionDecorator implements DataSource {
    private final DataSource wrapped;
    CompressionDecorator(DataSource ds) { wrapped = ds; }
    public void write(String data) { wrapped.write(compress(data)); }
    public String read() { return decompress(wrapped.read()); }
}

// Compose decorators
DataSource source = new CompressionDecorator(new EncryptionDecorator(new FileDataSource("data.bin")));
```

**Proxy Pattern**
```java
interface UserService { User getUser(long id); }

// Caching proxy — transparent to client
class CachingUserServiceProxy implements UserService {
    private final UserService real;
    private final Map<Long, User> cache = new HashMap<>();
    
    CachingUserServiceProxy(UserService real) { this.real = real; }
    
    public User getUser(long id) {
        return cache.computeIfAbsent(id, real::getUser);
    }
}
```

#### Behavioral Patterns

**Strategy Pattern**
```java
interface SortStrategy { void sort(int[] arr); }
class QuickSort implements SortStrategy { public void sort(int[] arr) { /* quicksort */ } }
class MergeSort implements SortStrategy { public void sort(int[] arr) { /* mergesort */ } }

class DataSorter {
    private SortStrategy strategy;
    DataSorter(SortStrategy s) { this.strategy = s; }
    void setStrategy(SortStrategy s) { this.strategy = s; }
    void sort(int[] arr) { strategy.sort(arr); }
}
// Switch algorithms at runtime without changing DataSorter
```

**Observer Pattern**
```java
interface EventListener<T> { void onEvent(T event); }

class EventBus {
    private final Map<Class<?>, List<EventListener<?>>> listeners = new HashMap<>();
    
    public <T> void subscribe(Class<T> eventType, EventListener<T> listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }
    
    @SuppressWarnings("unchecked")
    public <T> void publish(T event) {
        List<EventListener<?>> subs = listeners.getOrDefault(event.getClass(), List.of());
        for (EventListener<?> l : subs) ((EventListener<T>) l).onEvent(event);
    }
}
```

---

### 3.2 API Design Excellence

#### REST API Design Rules

```
1. Use nouns for resources, not verbs
   WRONG: GET /getUser, POST /createOrder
   RIGHT: GET /users/{id}, POST /orders

2. Use plural nouns
   WRONG: GET /user/1
   RIGHT: GET /users/1

3. Use sub-resources for relationships
   GET  /users/{id}/orders          → all orders of a user
   GET  /users/{id}/orders/{ordId}  → specific order of user

4. HTTP methods have semantics
   GET    → safe + idempotent (no side effects, repeatable)
   POST   → not safe, not idempotent (creates new resource)
   PUT    → not safe, idempotent (full update, same result if repeated)
   PATCH  → not safe, idempotent (partial update)
   DELETE → not safe, idempotent (delete, same result if repeated)

5. Status codes
   200 OK              → successful GET, PUT, PATCH
   201 Created         → successful POST (include Location header)
   204 No Content      → successful DELETE
   400 Bad Request     → client sent invalid data (validation errors)
   401 Unauthorized    → not authenticated
   403 Forbidden       → authenticated but not authorized
   404 Not Found       → resource doesn't exist
   409 Conflict        → business logic violation (e.g., duplicate email)
   422 Unprocessable   → valid JSON but semantically wrong
   429 Too Many Req    → rate limited
   500 Internal Error  → server bug (never expose stack traces)

6. Version your API
   /api/v1/users  → stable
   /api/v2/users  → breaking changes

7. Use consistent error response body
   {
     "error": {
       "code": "VALIDATION_ERROR",
       "message": "Email is required",
       "details": [{ "field": "email", "message": "must not be blank" }],
       "timestamp": "2026-06-04T10:30:00Z",
       "requestId": "req-abc-123"
     }
   }
```

#### Idempotency Keys (Critical for Production APIs)
```
Problem: Client sends POST /orders, times out, retries — creates duplicate order.
Solution: Client generates UUID, sends as header: Idempotency-Key: uuid-xyz
Server: stores (idempotency_key, response) for 24 hours.
If key seen again → return cached response, don't process again.
```

---

### 3.3 Testing Mastery

#### Test Pyramid

```
         /\
        /  \
       / E2E \       Few: slow, flaky, expensive
      /--------\
     / Integration\  Some: hit real DB, check contracts
    /--------------\
   /   Unit Tests   \ Many: fast, isolated, deterministic
  /------------------\
```

#### Unit Testing Best Practices

```java
// Follow AAA: Arrange, Act, Assert
@Test
void shouldThrowExceptionWhenEmailAlreadyExists() {
    // Arrange
    User existingUser = new User("john@example.com", "John");
    when(userRepository.findByEmail("john@example.com")).thenReturn(Optional.of(existingUser));
    
    // Act + Assert
    assertThatThrownBy(() -> userService.register(new RegisterRequest("john@example.com", "John2")))
        .isInstanceOf(EmailAlreadyExistsException.class)
        .hasMessage("Email john@example.com is already registered");
    
    // Verify no user was saved
    verify(userRepository, never()).save(any());
}

// Test one thing per test
// Name tests: should[ExpectedBehavior]When[Condition]
// Test edge cases: null, empty, boundary values, overflow
```

#### Contract Testing with Pact (Microservices)
```
Consumer (Order Service) defines:
  "I expect /products/{id} to return { id, name, price }"

Provider (Product Service) verifies:
  "My /products/{id} endpoint matches that contract"

Result: Services can be deployed independently with confidence.
```

---

## 4. Phase 3 — Advanced Systems (Months 18–36)

### 4.1 Concurrency & Parallelism — Deep Understanding

#### Java Concurrency Primitives

```java
// ThreadLocal — per-thread isolation (used in Spring for request context)
private static final ThreadLocal<User> currentUser = new ThreadLocal<>();
currentUser.set(authenticatedUser);          // set in auth filter
User user = currentUser.get();               // access anywhere in thread
currentUser.remove();                        // MUST clean up to avoid memory leak

// Semaphore — limit concurrent access (e.g., connection pool)
Semaphore semaphore = new Semaphore(10);     // max 10 concurrent
semaphore.acquire();                         // blocks if limit reached
try { doWork(); } finally { semaphore.release(); }

// CountDownLatch — wait for multiple operations to complete
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    executor.submit(() -> { doWork(); latch.countDown(); });
}
latch.await(); // blocks until count reaches 0

// CyclicBarrier — synchronize threads at a point
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All ready!"));
// Each thread calls barrier.await() and waits until all 3 arrive

// StampedLock — optimistic reads (higher throughput than ReentrantReadWriteLock)
StampedLock lock = new StampedLock();
long stamp = lock.tryOptimisticRead();
int x = point.x; int y = point.y;          // read without locking
if (!lock.validate(stamp)) {               // check if write happened
    stamp = lock.readLock();               // fallback to real read lock
    try { x = point.x; y = point.y; } finally { lock.unlockRead(stamp); }
}
```

#### Virtual Threads (Java 21 — Project Loom)
```java
// Old: Platform threads — expensive, limited to ~10k-50k
ExecutorService exec = Executors.newFixedThreadPool(200);

// New: Virtual threads — cheap, can have millions
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
// Each task gets its own virtual thread, mapped to platform threads by JVM
// Blocking I/O automatically parks the virtual thread, frees the platform thread
// Result: Handle 1M concurrent requests without increasing platform threads
```

#### Reactive Programming (WebFlux)
```java
// Traditional blocking:
User user = userRepository.findById(id);          // thread blocked waiting for DB
Order order = orderService.getOrders(user.getId()); // thread blocked again

// Reactive non-blocking:
Mono<User> userMono = userRepository.findById(id); // returns immediately
Flux<Order> orders = userMono
    .flatMap(user -> orderService.getOrders(user.getId()))
    .filter(order -> order.isActive())
    .take(10);
// Thread is freed while waiting for I/O — handles far more concurrent requests
```

---

### 4.2 Spring Boot — Production Grade

#### Spring Boot Production Checklist

```yaml
# application.yml — Production configuration
server:
  port: 8080
  tomcat:
    threads:
      max: 200           # platform threads
      min-spare: 10
    connection-timeout: 30000
  shutdown: graceful     # finish in-flight requests on shutdown

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    hikari:
      maximum-pool-size: 10      # match DB connection limit
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized

logging:
  level:
    root: INFO
    com.myapp: DEBUG
  pattern:
    console: "%d{ISO8601} [%thread] [%X{traceId},%X{spanId}] %-5level %logger{36} - %msg%n"
```

#### Exception Handling — Global Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_ERROR", errors));
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception on {}", req.getRequestURI(), ex);
        return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
        // NEVER expose the exception message to client in production
    }
}
```

#### Circuit Breaker (Resilience4j)
```java
// Prevent cascading failures in microservices
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
@Retry(name = "paymentService", fallbackMethod = "paymentFallback")
@TimeLimiter(name = "paymentService")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() -> paymentClient.process(req));
}

private CompletableFuture<PaymentResult> paymentFallback(PaymentRequest req, Exception ex) {
    log.warn("Payment service unavailable, queuing payment", ex);
    paymentQueue.enqueue(req); // queue for retry
    return CompletableFuture.completedFuture(PaymentResult.queued(req.getId()));
}

// States: CLOSED → OPEN → HALF_OPEN → CLOSED
// CLOSED: normal, requests pass through
// OPEN: all requests fail fast (no calls to downstream)
// HALF_OPEN: let a few requests through to test if service recovered
```

---

## 5. Phase 4 — Mastery & Leadership (Year 3+)

### 5.1 From Engineer to Architect Thinking

At this level, you think in **trade-offs**, not solutions.

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Consistency model | Strong (SQL) | Eventual (NoSQL) | Correctness vs Availability |
| Service coupling | Synchronous REST | Async messaging | Latency vs Decoupling |
| Data replication | Single leader | Multi-leader | Simplicity vs Write scalability |
| Cache invalidation | TTL-based | Event-driven | Simplicity vs Consistency |
| Deployment | Blue-green | Canary | Speed vs Risk |

**The question is never "which is better?" It's "which trade-off fits this context?"**

---

## 6. DSA Mastery

### 6.1 Algorithms — Time & Space Complexity

| Algorithm | Best | Average | Worst | Space |
|-----------|------|---------|-------|-------|
| QuickSort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| MergeSort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| HeapSort | O(n log n) | O(n log n) | O(n log n) | O(1) |
| TimSort | O(n) | O(n log n) | O(n log n) | O(n) |
| BFS | — | O(V+E) | O(V+E) | O(V) |
| DFS | — | O(V+E) | O(V+E) | O(V) |
| Dijkstra | — | O((V+E) log V) | O((V+E) log V) | O(V) |
| Bellman-Ford | — | O(VE) | O(VE) | O(V) |
| Binary Search | O(1) | O(log n) | O(log n) | O(1) |

### 6.2 Algorithm Patterns (Solve 95% of Problems)

#### Pattern 1: Two Pointers
```java
// Find pair with target sum in sorted array — O(n) time, O(1) space
int[] twoSum(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left < right) {
        int sum = arr[left] + arr[right];
        if (sum == target) return new int[]{left, right};
        else if (sum < target) left++;
        else right--;
    }
    return new int[]{-1, -1};
}
```

#### Pattern 2: Sliding Window
```java
// Longest substring without repeating characters — O(n)
int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> seen = new HashMap<>();
    int max = 0, left = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (seen.containsKey(c)) left = Math.max(left, seen.get(c) + 1);
        seen.put(c, right);
        max = Math.max(max, right - left + 1);
    }
    return max;
}
```

#### Pattern 3: Fast & Slow Pointers (Floyd's Cycle)
```java
// Detect cycle in linked list — O(n) time, O(1) space
boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true; // cycle detected
    }
    return false;
}
```

#### Pattern 4: Binary Search on Answer
```java
// Find minimum capacity to ship packages in D days
int shipWithinDays(int[] weights, int days) {
    int left = Arrays.stream(weights).max().getAsInt(); // min = heaviest package
    int right = Arrays.stream(weights).sum();           // max = ship all at once
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canShip(weights, days, mid)) right = mid;
        else left = mid + 1;
    }
    return left;
}
boolean canShip(int[] w, int days, int cap) {
    int daysNeeded = 1, current = 0;
    for (int weight : w) {
        if (current + weight > cap) { daysNeeded++; current = 0; }
        current += weight;
    }
    return daysNeeded <= days;
}
```

#### Pattern 5: Dynamic Programming
```java
// Classic DP: Longest Common Subsequence — O(mn)
int lcs(String s1, String s2) {
    int m = s1.length(), n = s2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            if (s1.charAt(i-1) == s2.charAt(j-1)) dp[i][j] = dp[i-1][j-1] + 1;
            else dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
    return dp[m][n];
}

// DP Template:
// 1. Define subproblem: dp[i] = "answer for first i elements"
// 2. Identify recurrence: dp[i] = f(dp[i-1], dp[i-2], ...)
// 3. Base case: dp[0] or dp[1]
// 4. Build bottom-up or use memoization top-down
```

#### Pattern 6: Backtracking
```java
// Generate all permutations
List<List<Integer>> permutations = new ArrayList<>();
void permute(int[] nums, List<Integer> current, boolean[] used) {
    if (current.size() == nums.length) { permutations.add(new ArrayList<>(current)); return; }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        current.add(nums[i]);
        permute(nums, current, used);  // recurse
        current.remove(current.size() - 1); // backtrack
        used[i] = false;
    }
}
// Template: choose → explore → unchoose
```

#### Pattern 7: Graph BFS/DFS
```java
// BFS — shortest path (unweighted)
int shortestPath(int[][] grid, int[] start, int[] end) {
    int rows = grid.length, cols = grid[0].length;
    boolean[][] visited = new boolean[rows][cols];
    Queue<int[]> queue = new ArrayDeque<>();
    queue.offer(new int[]{start[0], start[1], 0}); // row, col, distance
    visited[start[0]][start[1]] = true;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};
    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        if (curr[0] == end[0] && curr[1] == end[1]) return curr[2];
        for (int[] d : dirs) {
            int nr = curr[0] + d[0], nc = curr[1] + d[1];
            if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && !visited[nr][nc] && grid[nr][nc] == 0) {
                visited[nr][nc] = true;
                queue.offer(new int[]{nr, nc, curr[2] + 1});
            }
        }
    }
    return -1;
}
```

---

## 7. System Design — At Scale

### 7.1 Fundamental Numbers Every Engineer Must Know

```
Latency hierarchy (approximate):
  L1 cache: 1 ns
  L2 cache: 4 ns
  L3 cache: 40 ns
  RAM read: 100 ns
  SSD sequential read: 1 μs (1,000 ns)
  SSD random read: 100 μs
  HDD random read: 10 ms (10,000,000 ns)
  Network in same DC: 500 μs
  Network cross-continent: 150 ms

Throughput:
  SSD: ~500 MB/s sequential, ~50k IOPS random
  Network: 1-25 Gbps in datacenter
  RAM bandwidth: ~50 GB/s

Storage:
  1 char = 1 byte
  1 UUID = 36 bytes
  1 tweet = ~280 bytes
  1 photo = ~300 KB
  1 video minute = ~50 MB

Availability:
  99%       = 3.65 days/year downtime
  99.9%     = 8.76 hours/year
  99.99%    = 52.6 minutes/year
  99.999%   = 5.26 minutes/year (5 nines = telecom grade)
```

### 7.2 System Design Framework (Use in Interviews)

```
1. CLARIFY requirements (5 min)
   - Functional: what must the system DO?
   - Non-functional: scale, latency, availability, consistency?
   - Ask about: DAU, peak QPS, data size, geography

2. ESTIMATE scale (3 min)
   - Traffic: reads/sec, writes/sec
   - Storage: data per record × records × retention
   - Bandwidth: request size × QPS

3. HIGH-LEVEL design (10 min)
   - Draw: client → LB → servers → DB
   - Identify core components
   - Define API contracts

4. DEEP DIVE (15 min)
   - Database schema design
   - Bottlenecks and how to solve them
   - Failure modes and resilience
   - Specific component trade-offs

5. SCALE and improve (5 min)
   - Caching strategy
   - Sharding strategy
   - CDN for static assets
   - Async processing for heavy work
```

### 7.3 Caching Strategies

```
Cache-aside (Lazy loading):
  1. App checks cache for data
  2. Cache MISS: app fetches from DB, writes to cache, returns data
  3. Cache HIT: return from cache
  
  Pros: Only cache what's read, resilient to cache failure
  Cons: First request always slow, cache can have stale data

Write-through:
  1. App writes to cache AND DB synchronously
  
  Pros: Cache always up to date
  Cons: Write penalty, cache may store data that's never read

Write-behind (Write-back):
  1. App writes to cache only
  2. Cache asynchronously writes to DB in batches
  
  Pros: Very fast writes
  Cons: Data loss if cache crashes, complexity

Read-through:
  1. App always reads through cache
  2. Cache manages DB fetching (transparent to app)

Cache Eviction Policies:
  LRU (Least Recently Used)  → evict least recently accessed
  LFU (Least Frequently Used) → evict least often accessed
  TTL (Time to Live)          → evict after expiry
  FIFO                        → evict oldest inserted

Cache invalidation strategies:
  1. TTL-based (simplest): set expiry time
  2. Event-driven: publish cache invalidation events on write
  3. Version tags: include version in cache key, bump on change
```

### 7.4 Database Sharding

```
Horizontal sharding — split rows across machines by shard key

Shard key selection criteria:
  - High cardinality (many distinct values)
  - Even distribution (avoid hot shards)
  - Query locality (related data on same shard)
  - Immutable (shard migration is expensive)

Good shard keys: user_id, tenant_id
Bad shard keys: created_date (hot shard for recent data), country (uneven)

Sharding strategies:
  1. Range-based: user_id 1-1M → shard1, 1M-2M → shard2
     Pros: range queries work
     Cons: hot shards for sequential IDs

  2. Hash-based: hash(user_id) % num_shards
     Pros: even distribution
     Cons: range queries cross all shards, resharding is hard

  3. Directory-based: lookup service maps key → shard
     Pros: flexible, easy to rebalance
     Cons: lookup service is single point of failure

Consistent hashing — for resharding without full redistribution:
  - Map servers and keys onto a ring
  - Key goes to nearest server clockwise
  - Add server: only affects keys between new server and predecessor
  - Virtual nodes: each server has multiple points on ring for balance
```

---

## 8. Architecture Patterns

### 8.1 Microservices — When and How

```
Microservices make sense when:
  - Teams are large (10+ engineers) and need to deploy independently
  - Different services have different scaling needs
  - You want polyglot persistence/technology
  - Services have clear bounded contexts

Microservices are wrong when:
  - Small team (< 5 engineers) — operational overhead kills productivity
  - Tight coupling between "services" — you've built a distributed monolith
  - No domain clarity — premature decomposition creates integration hell

Decomposition strategies:
  - By business capability: OrderService, InventoryService, PaymentService
  - By subdomain (DDD): align with bounded contexts
  - By team ownership: Conway's Law — system structure mirrors org structure
```

### 8.2 Event-Driven Architecture

```
Choreography (events) vs Orchestration (commands)

Choreography:
  OrderService publishes → OrderCreated event
  PaymentService listens → PaymentService processes payment
  InventoryService listens → InventoryService reserves stock
  
  Pros: loose coupling, services are independent
  Cons: hard to trace workflow, hard to handle failures

Orchestration (Saga pattern):
  OrderSaga orchestrates:
    → Command: ProcessPayment to PaymentService
    ← Event: PaymentProcessed
    → Command: ReserveInventory to InventoryService
    ← Event: InventoryReserved
    → Command: ConfirmOrder to OrderService
  
  Pros: clear workflow, easier failure handling
  Cons: orchestrator becomes complex, coupling to orchestrator

Outbox Pattern (guaranteed event delivery):
  BEGIN TRANSACTION
    INSERT INTO orders (...)
    INSERT INTO outbox (event_type, payload)  ← in same transaction
  COMMIT
  
  Background worker: polls outbox, publishes to Kafka, marks published
  Guarantees: no event loss even if service crashes after DB commit
```

### 8.3 Domain-Driven Design (DDD)

```
Strategic DDD (big picture):
  Bounded Context: explicit boundary within which a domain model applies
    → "Customer" means different things in Sales vs Billing vs Shipping
    → Each context has its own model, its own "Customer" definition
  
  Context Map: shows how bounded contexts relate
    → Shared Kernel: teams share a subset of the domain model
    → Customer-Supplier: one team depends on another's API
    → Anti-Corruption Layer: translation layer to isolate from legacy

Tactical DDD (building blocks):
  Entity: has identity, mutable (Order, Customer, Product)
  Value Object: no identity, immutable (Money, Address, Email)
    → Money(100, USD) == Money(100, USD) by value, not reference
  Aggregate: cluster of entities/value objects with one root
    → Order (root) contains OrderItems (entities)
    → All access goes through root — enforce invariants
    → Only aggregate root has a repository
  Domain Event: something that happened (OrderPlaced, PaymentFailed)
  Repository: collection-like abstraction for aggregates
  Domain Service: business logic that doesn't belong to one entity
```

---

## 9. Database Mastery

### 9.1 SQL Optimization

```sql
-- EXPLAIN ANALYZE — understand query execution
EXPLAIN ANALYZE SELECT u.name, COUNT(o.id) 
FROM users u 
LEFT JOIN orders o ON o.user_id = u.id 
WHERE u.created_at > '2026-01-01'
GROUP BY u.id;

-- Read the output: look for Seq Scan (bad) vs Index Scan (good)
-- Seq Scan on large tables = missing index

-- Index types
CREATE INDEX idx_users_email ON users(email);                   -- B-tree (default, equality + range)
CREATE INDEX idx_orders_created ON orders(created_at DESC);     -- B-tree (sorted)
CREATE INDEX idx_users_gin ON users USING gin(tags);            -- GIN (arrays, JSONB)
CREATE INDEX idx_users_fulltext ON users USING gin(to_tsvector('english', bio)); -- full text

-- Covering index — avoids table lookup (index-only scan)
CREATE INDEX idx_orders_covering ON orders(user_id, status) INCLUDE (total, created_at);
-- Query: SELECT total, created_at FROM orders WHERE user_id = ? AND status = ?
-- All columns in index — no table heap access needed

-- Partial index — index only a subset of rows
CREATE INDEX idx_active_users ON users(email) WHERE active = true;
-- Much smaller index, faster for queries that filter active = true

-- Query optimization rules
1. SELECT only needed columns (avoid SELECT *)
2. Filter early — most restrictive WHERE conditions first
3. Avoid functions on indexed columns in WHERE: 
   BAD:  WHERE YEAR(created_at) = 2026
   GOOD: WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
4. Use EXISTS instead of IN for subqueries
5. Avoid N+1 queries — use JOINs or batch loading
```

### 9.2 Transactions & Isolation Levels

```
Transaction isolation levels (from weakest to strongest):

1. READ UNCOMMITTED
   Can read: dirty reads (uncommitted data from other transactions)
   Use case: almost never — metrics approximations maybe

2. READ COMMITTED (default in PostgreSQL)
   Cannot read dirty data
   Can read: non-repeatable reads (same query returns different data in one transaction)
   Use case: most OLTP applications

3. REPEATABLE READ (default in MySQL InnoDB)
   Same row reads same data throughout transaction
   Can see: phantom reads (new rows added by other transactions)
   Use case: bank account balance checks, reports

4. SERIALIZABLE
   Full isolation — transactions appear sequential
   Cannot see: anything inconsistent
   Use case: financial transactions, inventory management
   Cost: lowest throughput, most lock contention

Implementation:
  Pessimistic locking: take locks upfront (2PL — two-phase locking)
    SELECT * FROM accounts WHERE id = 1 FOR UPDATE; -- lock the row
  
  Optimistic locking: check version on commit, retry on conflict
    UPDATE accounts SET balance = ?, version = version + 1
    WHERE id = 1 AND version = ?; -- fails if someone changed it
```

---

## 10. Distributed Systems

### 10.1 CAP Theorem — The Most Misunderstood Concept

```
CAP Theorem: In a distributed system with network partition (P),
you CANNOT have both Consistency (C) and Availability (A).
You must choose: CP or AP.

Network partitions WILL happen. This is not optional.
So the real choice: during a partition, do you:
  CP: refuse to serve (return error) to avoid serving stale data
  AP: serve potentially stale data to remain available

Examples:
  CP: HBase, Zookeeper — bank transactions (can't show wrong balance)
  AP: Cassandra, DynamoDB — shopping cart, social feeds (stale data OK)

PACELC extends CAP:
  Even without partitions:
    EL: trade-off between latency (L) and consistency (C)
    Higher consistency → more coordination → higher latency
```

### 10.2 Consensus — How Distributed Systems Agree

```
Raft Consensus Algorithm:
  Goal: All nodes agree on the same ordered log of commands
  
  Roles: Leader (one), Followers (many), Candidate (during election)
  
  Leader election:
    1. Followers wait for heartbeat from leader
    2. If timeout → become Candidate, request votes
    3. Candidate wins majority → becomes Leader
    4. Leader sends AppendEntries RPCs as heartbeats and log replication
  
  Log replication:
    1. Client sends command to Leader
    2. Leader appends to local log, sends to all Followers
    3. Once majority acknowledge → Leader commits entry
    4. Leader notifies Followers to commit
  
  Used in: etcd (Kubernetes), CockroachDB, Consul
```

### 10.3 Distributed Patterns

```
Saga Pattern (distributed transactions):
  Problem: How to maintain consistency across multiple services without 2PC?
  Solution: Chain of local transactions, with compensating transactions for rollback
  
  Example — Order placement:
    1. OrderService: create order (PENDING)
    2. PaymentService: charge payment
       FAILURE → compensate: OrderService: cancel order
    3. InventoryService: reserve items
       FAILURE → compensate: PaymentService: refund, OrderService: cancel
    4. ShippingService: create shipment
    5. OrderService: update order to CONFIRMED

Leader Election:
  Use Zookeeper or etcd ephemeral nodes
  First to acquire lock is leader
  On leader crash: lock released, election repeats

Distributed Lock:
  RedLock algorithm (Redis):
    1. Get timestamp T1
    2. Try to acquire lock on N/2+1 Redis nodes with TTL
    3. If acquired on majority and (T2-T1) < TTL: lock is valid
    4. If failed: release all acquired locks
  
  Warning: RedLock has subtle issues under clock drift — use ZK for critical locks
```

---

## 11. Security Engineering

### 11.1 OWASP Top 10 — Know and Prevent All

| # | Vulnerability | Prevention |
|---|---------------|------------|
| A01 | Broken Access Control | Check authorization on EVERY request, server-side |
| A02 | Cryptographic Failures | Use TLS 1.3, AES-256-GCM, bcrypt for passwords |
| A03 | Injection (SQL, LDAP, OS) | Parameterized queries, input validation, sanitization |
| A04 | Insecure Design | Threat modeling, security in design phase |
| A05 | Security Misconfiguration | Disable defaults, scan configs, least privilege |
| A06 | Vulnerable Components | Dependency scanning (Snyk, OWASP Dependency Check) |
| A07 | Auth & Session Failures | MFA, secure session management, account lockout |
| A08 | Software & Data Integrity | Verify signatures, secure CI/CD pipeline |
| A09 | Security Logging Failures | Log all auth events, never log sensitive data |
| A10 | SSRF | Allowlist external URLs, block internal IP ranges |

### 11.2 Secure Coding Practices

```java
// SQL Injection — NEVER build queries with string concatenation
// WRONG:
String query = "SELECT * FROM users WHERE email = '" + email + "'";
// Attacker: email = "' OR '1'='1" → returns all users

// CORRECT: PreparedStatement (parameterized)
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
stmt.setString(1, email);

// Password hashing — NEVER store plain text, NEVER use MD5/SHA1
// CORRECT: BCrypt (adaptive — can increase work factor over time)
String hash = BCrypt.hashpw(password, BCrypt.gensalt(12)); // cost factor 12
boolean valid = BCrypt.checkpw(inputPassword, hash);

// XSS Prevention — sanitize output
// WRONG: write user content directly to HTML
// CORRECT: use template engine that auto-escapes, or:
import org.owasp.encoder.Encode;
String safeOutput = Encode.forHtml(userInput);

// CORS — explicit whitelist, never wildcard in production
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://myapp.com")  // explicit, not "*"
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowCredentials(true)
            .maxAge(3600);
    }
}

// Rate limiting — prevent brute force and DoS
@RateLimiter(name = "loginEndpoint", fallbackMethod = "loginRateLimitFallback")
public ResponseEntity<?> login(@RequestBody LoginRequest req) { /* ... */ }
```

---

## 12. Performance Engineering

### 12.1 Performance Testing Types

```
Load Testing: simulate expected user load — verify performance meets SLA
Stress Testing: push beyond limits — find breaking point
Spike Testing: sudden traffic spike — simulate viral event
Soak Testing: run at normal load for hours/days — find memory leaks
Chaos Testing: inject failures — verify resilience (Netflix Chaos Monkey)
```

### 12.2 Java Performance Tuning

```java
// Profiling tools: JProfiler, YourKit, async-profiler, Java Flight Recorder

// Memory leak detection:
// jmap -heap <pid>          → heap summary
// jmap -histo <pid>         → object histogram
// jcmd <pid> GC.run         → trigger GC
// jcmd <pid> VM.native_memory → native memory usage

// Common memory leaks:
// 1. Static collections that grow unbounded
private static List<String> cache = new ArrayList<>();  // never cleared!

// 2. Listeners not removed
button.addActionListener(this);  // holds reference to this
// Fix: removeActionListener on cleanup

// 3. ThreadLocal not cleared
threadLocal.set(obj);  // set in thread
// MUST call: threadLocal.remove() when done (especially in thread pools)

// CPU profiling insights:
// Hot methods: where the program spends most time
// Lock contention: threads waiting for locks → reduce synchronized scope
// Memory allocation rate: too many short-lived objects → GC pressure

// Benchmarking — use JMH (Java Microbenchmark Harness)
@Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
public void benchmarkStringConcat(Blackhole bh) {
    String result = "Hello" + " " + "World";  // vs StringBuilder
    bh.consume(result);  // prevent JIT dead-code elimination
}
```

---

## 13. DevOps & Platform Engineering

### 13.1 Docker Mastery

```dockerfile
# Optimized multi-stage Dockerfile for Java
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S appgroup && adduser -S appuser -G appgroup  # non-root user
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
RUN chown appuser:appgroup app.jar
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]

# Key principles:
# - Multi-stage: build stage (JDK) vs runtime stage (JRE) → smaller image
# - Non-root user: security best practice
# - HEALTHCHECK: orchestrator knows when service is ready
# - UseContainerSupport: JVM respects cgroup memory limits
```

### 13.2 Kubernetes Core Concepts

```yaml
# Deployment — manages Pod replicas
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
      maxSurge: 1        # can have 4 pods during update
      maxUnavailable: 0  # never have less than 3
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
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
        livenessProbe:
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
```

### 13.3 CI/CD Pipeline (GitHub Actions)

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with: { java-version: '21', distribution: 'temurin' }
    - run: mvn test

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: mvn org.owasp:dependency-check-maven:check

  build-and-push:
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        push: true
        tags: myregistry/order-service:${{ github.sha }}

  deploy-staging:
    needs: build-and-push
    environment: staging
    runs-on: ubuntu-latest
    steps:
    - run: |
        kubectl set image deployment/order-service \
          order-service=myregistry/order-service:${{ github.sha }}
        kubectl rollout status deployment/order-service
```

---

## 14. Code Craft — Writing World-Class Code

### 14.1 The Rules

```
1. Make it work → Make it right → Make it fast
   In that order. Never optimize prematurely.

2. Code is read 10x more than it is written.
   Write for the reader, not the computer.

3. The best code is code that doesn't need comments to be understood.
   If you need a comment to explain WHAT the code does → rename variables.
   Only write comments to explain WHY — intent, constraints, trade-offs.

4. Names are the most important thing you write.
   A method named processData() tells you nothing.
   A method named calculateOrderTotalWithTax() tells you everything.

5. Functions should do ONE thing.
   If you can't describe it without "and" → split it.

6. Don't make me think.
   The reader should be able to trace the flow without reversing logic.

7. Fail fast and explicitly.
   Validate inputs at boundaries. Throw descriptive exceptions.
   Don't let bad state silently propagate.

8. Immutability by default.
   Mutable state is the root of most bugs in concurrent code.
   Prefer final fields, immutable value objects.
```

### 14.2 Code Smell Reference

| Smell | Problem | Refactoring |
|-------|---------|-------------|
| Long method (>20 lines) | Does too many things | Extract Method |
| Long parameter list (>3 params) | High coupling | Introduce Parameter Object |
| God class (>500 lines) | Too many responsibilities | Extract Class |
| Feature envy (method uses other class more than its own) | Wrong responsibility placement | Move Method |
| Data clump (same group of data always together) | Should be an object | Extract Class |
| Primitive obsession (String for email, int for money) | Missing domain objects | Replace Primitive with Object |
| Switch statements on type | Hard to extend | Replace Conditional with Polymorphism |
| Duplicate code | Maintenance burden | Extract Method/Superclass |
| Dead code | Confusion | Delete it |
| Speculative generality | "We might need this someday" | Delete it — YAGNI |

---

## 15. Engineering Leadership

### 15.1 Code Review Excellence

**As Author:**
- Keep PRs small — < 400 lines (reviewable in 30 min)
- Write a clear description: WHY this change, not WHAT
- Self-review before requesting review
- Link to ticket/issue
- Add test evidence (screenshots, test output)

**As Reviewer:**
- Review the PR, not the person
- Distinguish: blocking (must fix) vs non-blocking (suggestion)
- Explain WHY, not just what is wrong
- Ask questions instead of statements: "Have you considered X?" not "You should do X"
- Approve with comments (non-blocking) vs request changes (blocking)
- Review within 24 hours — blocked PRs kill team velocity

**Nitpick culture**: mark minor style issues as `nit:` — author can choose to address

### 15.2 Technical Documentation

```
Architecture Decision Records (ADR):
  Title: Short present-tense description
  Status: Proposed / Accepted / Deprecated / Superseded
  Context: The situation we were in
  Decision: What we chose
  Consequences: Trade-offs, follow-ups required

Good documentation:
  README: how to run locally in < 5 commands
  Architecture diagrams: C4 model (System, Container, Component, Code)
  API docs: OpenAPI/Swagger — auto-generated from annotations
  Runbooks: how to handle incidents, step by step
```

### 15.3 Incident Management

```
When production is down:
1. Acknowledge (< 5 min): "I'm investigating, incident channel open"
2. Mitigate first, diagnose later
   → Rollback the deployment? Scale up? Redirect traffic?
   → Fastest path to restore service > finding root cause
3. Communicate: update stakeholders every 15-30 min
4. Resolve and monitor
5. Post-mortem (blameless):
   - Timeline of events
   - Root cause (5 Whys analysis)
   - Contributing factors
   - Action items with owners and dates
   
The 5 Whys — Example:
  Problem: Payments failing
  Why? → Database connection pool exhausted
  Why? → New feature has N+1 query (100 queries per request)
  Why? → Code review didn't catch it — no query count assertion
  Why? → No standard for detecting N+1 in our review process
  Why? → We've never had this issue before at this scale
  Action: Add hibernate.show_sql + query count assertion in integration tests
```

---

## 16. Projects That Prove Mastery

Build these projects in order of complexity. Each one should be deployed to production (not just localhost).

### Tier 1 — Foundation Projects (Months 1–6)
1. **URL Shortener** — REST API + PostgreSQL + Redis caching + deployment
2. **Task Management API** — Full CRUD + JWT auth + role-based access
3. **Real-time Chat** — WebSocket + Spring Boot + message persistence

### Tier 2 — Intermediate Projects (Months 6–18)
4. **E-commerce System** — Orders, inventory, payments, email notifications
5. **Social Media Feed** — Newsfeed generation, follow system, caching strategy
6. **File Storage Service** — S3-like API, chunked uploads, download streaming

### Tier 3 — Advanced Projects (Year 2+)
7. **Distributed Job Queue** — Kafka + workers + retry + dead-letter queue
8. **Search Engine** — Inverted index, relevance ranking, distributed indexing
9. **Database Engine** — Implement a simple B-tree, write-ahead log, query parser

### What Each Project Must Have
```
✅ Unit tests + integration tests (>80% coverage)
✅ Docker + docker-compose
✅ CI/CD pipeline (GitHub Actions)
✅ Deployed on cloud (AWS/GCP/Azure)
✅ Monitoring (Prometheus + Grafana)
✅ Structured logging (ELK or similar)
✅ README with architecture diagram
✅ Performance benchmarks
```

---

## 17. Books Every World-Class Engineer Must Read

### Tier 1 — Read These First (Essential)
| Book | Why Read It |
|------|-------------|
| **Clean Code** — Robert C. Martin | The Bible of writing readable, maintainable code |
| **The Pragmatic Programmer** — Hunt & Thomas | Mindset, habits, and tools of elite engineers |
| **Designing Data-Intensive Applications** — Martin Kleppmann | Best book on distributed systems and databases |
| **Refactoring** — Martin Fowler | How to continuously improve existing code |
| **A Philosophy of Software Design** — John Ousterhout | Deep complexity management principles |

### Tier 2 — Read These Next (Advanced)
| Book | Why Read It |
|------|-------------|
| **System Design Interview (Vol 1 & 2)** — Alex Xu | Practical system design patterns |
| **Domain-Driven Design** — Eric Evans (Blue Book) | The reference for modeling complex domains |
| **Building Microservices** — Sam Newman | Definitive microservices guide |
| **Release It!** — Michael Nygard | How to build systems that survive production |
| **The Staff Engineer's Path** — Tanya Reilly | Becoming a senior/staff engineer |

### Tier 3 — Mastery Level
| Book | Why Read It |
|------|-------------|
| **Computer Systems: A Programmer's Perspective** — Bryant & O'Hallaron | CPU, memory, OS fundamentals |
| **Introduction to Algorithms (CLRS)** — Cormen et al. | The authoritative DSA textbook |
| **Fundamentals of Software Architecture** — Richards & Ford | Architecture patterns and decision-making |
| **Accelerate** — Forsgren, Humble, Kim | Data-backed engineering productivity metrics |

---

## 18. Daily Habits of Elite Engineers

### Morning Routine (30–60 min)
```
1. LeetCode — 1 problem per day, minimum (medium difficulty)
   Track: time taken, approaches tried, optimal solution
2. Read 1 engineering blog post (Netflix Tech Blog, Uber Eng, Martin Fowler)
3. Review your code from yesterday with fresh eyes
```

### Weekly Habits
```
Monday:    Set weekly learning goal (one deep topic)
Tue-Thu:   Implementation + code review + reading
Friday:    Reflect — what did you learn? Write a short summary
Weekend:   Side project or open source contribution (2-4 hours)
```

### Annual Goals
```
Year 1: Master one language + DSA + one framework
Year 2: System design + distributed systems + cloud (AWS cert)
Year 3: Architecture + leadership + deep specialty (ML, security, etc.)
Year 4+: Thought leadership — blog, talks, open source maintainer
```

### Platforms for Continuous Learning
```
Practice:   LeetCode, NeetCode (curated list), HackerRank
Reading:    Martin Fowler's blog, High Scalability blog, ACM Queue
Courses:    MIT OpenCourseWare (free CS fundamentals), Coursera, Pluralsight
Community:  GitHub open source, Stack Overflow, local tech meetups
Videos:     MIT 6.824 Distributed Systems (free on YouTube), GOTO Conf talks
```

### How to Learn New Technology (Efficiently)
```
1. Build the simplest thing that shows it works (hello world → tiny feature)
2. Read the official docs — not tutorials (tutorials often skip trade-offs)
3. Read the source code — understand HOW it works, not just WHAT it does
4. Break it — what happens when things fail? How does it recover?
5. Read post-mortems and case studies of real production use
6. Teach it — write a blog post, explain it to someone, make notes
```

---

## Summary — Your Roadmap

```
Month 1-3:   CS fundamentals, data structures, implement everything from scratch
Month 3-6:   Algorithms (solve 150+ LeetCode problems), Java/OOP mastery
Month 6-12:  Spring Boot production-grade, REST API design, SQL optimization
Month 12-18: System design basics, microservices, messaging (Kafka), Docker/K8s
Month 18-24: Distributed systems, advanced caching, performance engineering
Month 24-36: Architecture patterns, DDD, security engineering, cloud certifications
Year 3+:     Technical leadership, open source, engineering blog, conference talks

The difference between a good engineer and a GREAT engineer:
  - Good: knows the tools
  - Great: understands the trade-offs behind every tool and can reason 
           from first principles when no tool fits
```

---

*World-class is not a destination. It is a direction. Keep moving.*
