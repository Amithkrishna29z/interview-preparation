# World-Class Software Engineer — Complete Mastery Roadmap

> **Goal**: Become one of the best software engineers in the world.
> This is not a sprint — it is a multi-year, deliberate journey. Follow this roadmap with discipline.

---

## 1. Mindset

### The Two Things That Separate World-Class Engineers

**1. They obsess over fundamentals.** The best engineers can explain *why* a HashMap is O(1) average, what a TCP handshake does, and why SOLID matters — because they understand it, not memorized it.

**2. They build things and reflect.** Reading alone builds zero skills. You must code, break things, debug, optimize, and repeat.

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

Core (know these well as a junior):
- [ ] 8 primitive types + wrappers; autoboxing pitfalls; String pool/interning
- [ ] All Collection implementations and their Big-O; HashMap internals (hash, collisions, resize)
- [ ] ConcurrentHashMap vs synchronized HashMap
- [ ] Streams (lazy evaluation, pitfalls); functional interfaces (Function/Predicate/Consumer/Supplier)
- [ ] Optional — correct use vs anti-patterns; generics (wildcards, bounded types, type erasure)
- [ ] Future / CompletableFuture / ExecutorService basics
- [ ] Records, Sealed classes, Pattern matching (modern Java)

Awareness (grow into these): JVM internals (bytecode, class loading, JIT); GC choices (Serial/Parallel/G1/ZGC); the Java Memory Model (visibility, atomicity, happens-before); Reflection cost; custom annotations; Java modules.

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

**Garbage Collection — G1GC (Default in Java 9+)** *(awareness — deep tuning is a senior concern)*
```
Minor GC clears Eden + Survivors and promotes survivors to Old Gen; concurrent
marking and mixed GC keep Stop-the-World pauses small. Common flags:
-Xms / -Xmx (initial/max heap), -XX:+UseG1GC, -XX:MaxGCPauseMillis=200.
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

#### Hash Map — Implementation From Scratch (core idea)

The key mechanics: an array of buckets, `index = hash & (table.length - 1)`, a linked-list chain per bucket for collisions, and a **resize** (double the table) once `size > capacity * 0.75`.

```java
static class Node<K, V> { K key; V value; int hash; Node<K, V> next; /* ctor */ }

public void put(K key, V value) {
    int hash = key.hashCode() ^ (key.hashCode() >>> 16); // spread high bits
    int idx = hash & (table.length - 1);
    for (Node<K, V> n = table[idx]; n != null; n = n.next)        // update if present
        if (n.hash == hash && n.key.equals(key)) { n.value = value; return; }
    Node<K, V> node = new Node<>(key, value, hash);              // else prepend
    node.next = table[idx]; table[idx] = node;
    if (++size > table.length * 0.75f) resize();                 // double & rehash
}

public V get(K key) {
    int hash = key.hashCode() ^ (key.hashCode() >>> 16);
    for (Node<K, V> n = table[hash & (table.length - 1)]; n != null; n = n.next)
        if (n.hash == hash && n.key.equals(key)) return n.value;
    return null;
}
```

---

### 2.3 Object-Oriented Design — Beyond Syntax

#### SOLID Principles — With Real Examples

**S — Single Responsibility Principle** — each class has one reason to change. Don't put validation, persistence, and email in one `UserService`; split them out.
```java
class UserService { void createUser(User u) { validator.validate(u); repo.save(u); emailer.sendWelcome(u); } }
class UserValidator { void validate(User u) { /* rules */ } }
class UserRepository { void save(User u) { /* persistence */ } }
class UserEmailer { void sendWelcome(User u) { /* email */ } }
```

**O — Open/Closed Principle**
```java
// Open for extension, closed for modification: add new shapes without editing
// AreaCalculator (no growing instanceof chain).
interface Shape { double area(); }
class Circle implements Shape { double area() { return Math.PI * r * r; } }
class Rectangle implements Shape { double area() { return w * h; } }
class Triangle implements Shape { double area() { return 0.5 * b * h; } }
class AreaCalculator { double area(Shape s) { return s.area(); } } // never changes
```

**L — Liskov Substitution Principle** — a subtype must be usable wherever its base type is expected. The classic violation: `Square extends Rectangle` overriding `setWidth` to also change height breaks code that expects Rectangle. Fix with a separate hierarchy/interface.
```java
interface Shape { int area(); }
class Rectangle implements Shape { /* ... */ }
class Square implements Shape { /* ... */ }
```

**I — Interface Segregation Principle** — many small interfaces beat one fat interface that forces classes to implement methods they don't need (a `Robot` shouldn't be made to implement `eat()`).
```java
interface Workable { void work(); }
interface Feedable { void eat(); }
class Human implements Workable, Feedable { /* ... */ }
class Robot implements Workable { public void work() { /* ... */ } }
```

**D — Dependency Inversion Principle** — depend on abstractions, not concretes. Inject an `OrderRepository` interface instead of `new`-ing a `MySQLOrderRepository` — this is exactly what Spring's dependency injection does, and it makes swapping implementations and mocking in tests trivial.
```java
interface OrderRepository { void save(Order o); Order findById(long id); }
class OrderService {
    private final OrderRepository repo;
    OrderService(OrderRepository repo) { this.repo = repo; } // inject dependency
}
```

---

## 3. Phase 2 — Core Engineering (Months 6–18)

### 3.1 Design Patterns — Know When NOT to Use Them Too

#### Creational Patterns

**Singleton** — one instance. The enum form is the simplest thread-safe approach. (The classic alternative is double-checked locking with a `volatile` field.)
```java
public enum DatabaseConnection {
    INSTANCE;
    private final Connection conn = createConnection();
    public Connection getConnection() { return conn; }
}
```

**Builder Pattern** — avoid telescoping constructors; chain fluent setters that each `return this`, then `build()` an immutable object.
```java
public static class Builder {
    private final String url; private String method = "GET"; private int timeout = 30;
    public Builder(String url) { this.url = url; }
    public Builder method(String m) { this.method = m; return this; }
    public Builder timeout(int t) { this.timeout = t; return this; }
    public HttpRequest build() { return new HttpRequest(this); }
}
// Usage: new HttpRequest.Builder(url).method("POST").timeout(60).build();
```

**Factory Method** — centralize object creation behind one method, returning the interface type.
```java
interface Notification { void send(String msg); }
class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SMSNotification();
            default      -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}
```

#### Structural Patterns

**Decorator Pattern** — wrap an object to add behavior without changing it; decorators compose.
```java
interface DataSource { void write(String data); String read(); }
class EncryptionDecorator implements DataSource {
    private final DataSource wrapped;
    EncryptionDecorator(DataSource ds) { wrapped = ds; }
    public void write(String data) { wrapped.write(encrypt(data)); }
    public String read() { return decrypt(wrapped.read()); }
}
// Compose: new CompressionDecorator(new EncryptionDecorator(new FileDataSource(...)))
```

**Proxy Pattern** — a stand-in that implements the same interface and adds behavior (caching, lazy loading, access control) around the real object, transparently to the caller.
```java
class CachingUserServiceProxy implements UserService {
    private final UserService real;
    private final Map<Long, User> cache = new HashMap<>();
    CachingUserServiceProxy(UserService real) { this.real = real; }
    public User getUser(long id) { return cache.computeIfAbsent(id, real::getUser); }
}
```

#### Behavioral Patterns

**Strategy Pattern** — swap interchangeable algorithms behind one interface at runtime.
```java
interface SortStrategy { void sort(int[] arr); }
class DataSorter {
    private SortStrategy strategy;
    void setStrategy(SortStrategy s) { this.strategy = s; }
    void sort(int[] arr) { strategy.sort(arr); }   // QuickSort, MergeSort, ...
}
```

**Observer Pattern** — subject keeps a list of listeners and notifies them on change.
```java
interface EventListener { void onEvent(Object event); }
class EventBus {
    private final List<EventListener> listeners = new ArrayList<>();
    void subscribe(EventListener l) { listeners.add(l); }
    void publish(Object event) { listeners.forEach(l -> l.onEvent(event)); }
}
// Spring's ApplicationEventPublisher / @EventListener is this pattern built in.
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

*Awareness:* the consumer declares the response shape it expects from a provider's endpoint, and the provider runs a test verifying its endpoint matches that contract — so services can deploy independently with confidence.

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

// Awareness (advanced): CyclicBarrier syncs threads at a rendezvous point;
// StampedLock adds optimistic reads for higher read throughput than
// ReentrantReadWriteLock. Reach for these only when profiling demands it.
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

*Awareness:* instead of blocking a thread on each DB/HTTP call, reactive (`Mono`/`Flux`) returns immediately and frees the thread while waiting for I/O, so one thread serves many requests. Powerful but harder to debug — most junior Spring Boot work stays on the blocking MVC stack, and Java 21 virtual threads now give similar scalability with simpler code.

---

### 4.2 Spring Boot — Production Grade

#### Spring Boot Production Checklist

```yaml
# application.yml — the production knobs worth knowing as a junior
server:
  shutdown: graceful           # finish in-flight requests on shutdown
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    hikari:
      maximum-pool-size: 10    # keep at/under the DB's connection limit
      minimum-idle: 5
management:
  endpoints:
    web: { exposure: { include: health,metrics,prometheus } }  # Actuator
logging:
  level: { root: INFO, com.myapp: DEBUG }
# (advanced: Tomcat thread pool sizing, Hikari idle/max-lifetime, trace-id log
#  patterns — tune these once you have real traffic to measure.)
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
// Prevent cascading failures: wrap a flaky downstream call with a breaker +
// fallback. Resilience4j gives annotations for this:
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult processPayment(PaymentRequest req) {
    return paymentClient.process(req);
}
private PaymentResult paymentFallback(PaymentRequest req, Exception ex) {
    paymentQueue.enqueue(req);                 // queue for retry
    return PaymentResult.queued(req.getId());  // graceful degradation
}
// States: CLOSED (normal) → OPEN (fail fast, skip downstream) →
//         HALF_OPEN (let a few through to test recovery) → CLOSED.
```

---

## 5. Phase 4 — Mastery & Leadership (Year 3+)

### 5.1 From Engineer to Architect Thinking

*Awareness (long-term path):* At senior/staff level you stop asking "which option is better?" and start asking "which trade-off fits this context?" — e.g. strong vs eventual consistency, sync REST vs async messaging, blue-green vs canary deploys. You don't need this yet as a junior; just know every architectural choice is a trade-off, not a right answer.

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

*Awareness (you'll meet this at scale, not on day one):*
```
Horizontal sharding splits rows across machines by a shard key. A good shard
key is high-cardinality, evenly distributed, and immutable (e.g. user_id,
tenant_id); avoid created_date or country (hot/uneven shards).

Strategies: range-based (range queries work, but hot shards for sequential IDs),
hash-based (even spread, but resharding is hard), directory-based (flexible,
lookup service is a SPOF). Consistent hashing maps servers + keys onto a ring so
adding a server only moves a small slice of keys; virtual nodes balance it.
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

*Awareness summary:*
```
Choreography: services react to each other's events (loose coupling, but hard
  to trace a workflow). Orchestration / Saga: one orchestrator issues commands
  step by step (clear workflow, but the orchestrator gets complex).
Outbox pattern: write the domain row AND an "outbox" row in the SAME DB
  transaction, then a worker publishes the outbox to Kafka — guarantees no
  event is lost even if the service crashes after commit.
```

### 8.3 Domain-Driven Design (DDD)

*Awareness summary (you'll go deep here as you grow):* DDD splits a system by **bounded context** — "Customer" means different things in Sales vs Billing, each with its own model. Key tactical building blocks worth recognising: **Entity** (has identity, mutable), **Value Object** (no identity, immutable, e.g. Money/Address), **Aggregate** (a cluster with one root that all access goes through), **Domain Event**, and **Repository** (collection-like abstraction per aggregate root).

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
CREATE INDEX idx_users_email ON users(email);               -- B-tree (default: equality + range)
CREATE INDEX idx_orders_created ON orders(created_at DESC); -- B-tree (sorted)
-- Advanced (awareness): GIN indexes for arrays/JSONB & full-text; covering
-- indexes (INCLUDE extra cols for index-only scans); partial indexes
-- (WHERE active = true) to index just a subset of rows.

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

*Awareness (senior-depth topic):* Consensus algorithms like **Raft** make a cluster agree on one ordered log. One Leader is elected; it replicates each command to Followers and commits once a **majority** acknowledge. Used inside etcd (Kubernetes), CockroachDB, and Consul — as a junior you *use* these systems rather than implement the algorithm.

### 10.3 Distributed Patterns

*Awareness summary:*
```
Saga: keep consistency across services without 2PC by chaining local
  transactions, each with a compensating (rollback) action on failure.
Leader election: use Zookeeper/etcd ephemeral nodes — first to grab the lock
  leads; on crash the lock releases and a new election runs.
Distributed lock: Redis RedLock acquires a lock on a majority of nodes with a
  TTL. It has clock-drift edge cases — prefer Zookeeper for critical locks.
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
// Tools (awareness): profilers (JProfiler, async-profiler, Java Flight Recorder),
// jmap/jcmd for heap dumps, and JMH for microbenchmarks.

// Common memory leaks a junior WILL hit:
// 1. Static collections that grow unbounded
private static List<String> cache = new ArrayList<>();  // never cleared!
// 2. Listeners not removed — caller stays referenced (removeListener on cleanup)
// 3. ThreadLocal not cleared — MUST call threadLocal.remove() when done,
//    especially in thread pools where threads are reused.

// CPU insight: hot methods = where time goes; lock contention = threads waiting
// (shrink synchronized scope); high allocation rate of short-lived objects = GC pressure.
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

*Awareness (platform/ops depth — you'll mostly read these manifests, not author them early on):*
```yaml
# A Deployment manages N replica Pods and does rolling updates. Things to know:
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  template:
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:1.2.3
        resources:                       # requests = guaranteed, limits = cap
          requests: { memory: "256Mi", cpu: "250m" }
          limits:   { memory: "512Mi", cpu: "500m" }
        readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
        livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 } }
        env:                             # secrets come from Secret objects, not YAML
        - name: DB_PASSWORD
          valueFrom: { secretKeyRef: { name: db-secret, key: password } }
```

### 13.3 CI/CD Pipeline (GitHub Actions)

*Awareness:* a typical pipeline runs jobs in stages — **test** (`mvn test`) → **security-scan** (OWASP dependency check) → **build-and-push** a Docker image tagged with the commit SHA → **deploy** (e.g. `kubectl set image` + `kubectl rollout status`). Later stages `needs:` the earlier ones, and deploys are usually gated to the `main` branch.

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

Junior essentials: a **README** that runs the app locally in < 5 commands, and **OpenAPI/Swagger** API docs auto-generated from annotations. *Awareness:* teams also keep **ADRs** (Architecture Decision Records: context → decision → consequences), C4 architecture diagrams, and incident runbooks.

### 15.3 Incident Management

*Awareness summary (you'll be guided through your first incidents):* when production is down — acknowledge fast, **mitigate before diagnosing** (rollback / scale up / redirect traffic), communicate every 15–30 min, then run a **blameless post-mortem** using **5 Whys** to find the real root cause and assign action items with owners.

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

### Tier 2 / Tier 3 — Read As You Grow (awareness)

Advanced: *System Design Interview Vol 1 & 2* (Xu), *Domain-Driven Design* (Evans), *Building Microservices* (Newman), *Release It!* (Nygard), *The Staff Engineer's Path* (Reilly).

Mastery: *Computer Systems: A Programmer's Perspective* (Bryant & O'Hallaron), *Introduction to Algorithms / CLRS*, *Fundamentals of Software Architecture* (Richards & Ford), *Accelerate* (Forsgren, Humble, Kim).

---

## 18. Daily Habits of Elite Engineers

```
Daily:   1 LeetCode problem (medium); read 1 engineering blog post; re-read
         yesterday's code with fresh eyes.
Weekly:  pick one deep topic; reflect + write a short summary Friday; spend a
         couple of weekend hours on a side project or open source.
Annual:  Y1 language + DSA + one framework · Y2 system design + cloud cert ·
         Y3 architecture + a specialty · Y4+ blog/talks/OSS.
```

**Platforms:** LeetCode/NeetCode, Martin Fowler & High Scalability blogs, MIT OCW, MIT 6.824 (distributed systems on YouTube), GitHub/Stack Overflow.

**How to learn a new tech fast:** build the simplest thing that works → read the official docs (not tutorials) → read the source → break it and watch how it recovers → teach it (notes or a blog post).

---

## Summary — Your Roadmap (Cheat Sheet)

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
