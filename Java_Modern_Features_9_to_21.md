# Java Modern Features: Java 9 to Java 21 — Complete Interview Guide

## Overview

Java releases since Java 8 have transformed the language significantly. Java 17 and Java 21 are both Long-Term Support (LTS) releases — the current baseline for modern Java projects. This guide covers every major feature from Java 9 through Java 21 with deep explanations, code examples, and interview Q&A.

---

## Java 9 Features

### Module System (JPMS — Java Platform Module System)

The module system (Project Jigsaw) introduces strong encapsulation at the package level.

```java
// module-info.java (in src/main/java/)
module com.example.orderservice {
    requires java.base;           // implicit, always present
    requires spring.context;       // compile and runtime dependency
    requires transitive java.sql;  // transitive: consumers also get java.sql

    exports com.example.orderservice.api;           // accessible to all modules
    exports com.example.orderservice.internal to com.example.admin; // restricted export

    opens com.example.orderservice.model;           // allows deep reflection (for frameworks)
    opens com.example.orderservice.dto to com.fasterxml.jackson.databind; // restricted open

    uses com.example.spi.PaymentGateway;            // service consumer
    provides com.example.spi.PaymentGateway
        with com.example.orderservice.StripeGateway; // service provider
}
```

**Key keywords:**
- `requires`: declares a dependency on another module
- `requires transitive`: re-exports the dependency to consumers
- `exports`: makes a package accessible to other modules
- `opens`: allows reflective access (needed by Spring, Hibernate, Jackson)
- `uses`/`provides`: ServiceLoader SPI mechanism

**Interview Tip:** Spring Boot applications typically run on the classpath (not module path), so JPMS is not mandatory for Spring apps but you should understand the concepts.

### JShell (REPL)
```bash
$ jshell
jshell> int x = 10
x ==> 10
jshell> x * 3
$2 ==> 30
jshell> /list
jshell> /exit
```

### Collection Factory Methods (Immutable)
```java
// Java 9+ — unmodifiable, null-disallowed, no guaranteed order (Set/Map)
List<String> list = List.of("a", "b", "c");
Set<String> set = Set.of("x", "y", "z");
Map<String, Integer> map = Map.of("one", 1, "two", 2);
Map<String, Integer> map2 = Map.ofEntries(
    Map.entry("one", 1),
    Map.entry("two", 2),
    Map.entry("three", 3)
);

// throws UnsupportedOperationException
list.add("d"); // ERROR
```

**vs Arrays.asList():** Arrays.asList returns a fixed-size but mutable list; List.of is fully immutable.

### Stream API Enhancements
```java
// takeWhile — take elements while predicate is true (stops at first false)
List<Integer> result = Stream.of(1, 2, 3, 4, 5, 1, 2)
    .takeWhile(n -> n < 4)
    .collect(Collectors.toList()); // [1, 2, 3]

// dropWhile — drop elements while predicate is true
List<Integer> result2 = Stream.of(1, 2, 3, 4, 5)
    .dropWhile(n -> n < 3)
    .collect(Collectors.toList()); // [3, 4, 5]

// Stream.ofNullable — avoids NullPointerException
Stream<String> s = Stream.ofNullable(null); // empty stream, not NPE

// Stream.iterate with predicate (like a for loop)
Stream.iterate(0, n -> n < 10, n -> n + 2)
    .forEach(System.out::println); // 0, 2, 4, 6, 8
```

### Optional Enhancements
```java
Optional<String> opt = Optional.of("hello");

// ifPresentOrElse — do something or else
opt.ifPresentOrElse(
    s -> System.out.println("Found: " + s),
    () -> System.out.println("Not found")
);

// or() — provide alternative Optional (lazy)
Optional<String> result = opt.or(() -> Optional.of("default"));

// stream() — converts Optional to Stream (0 or 1 element)
long count = opt.stream().count(); // 1
```

### Private Methods in Interfaces
```java
interface Validator<T> {
    boolean validate(T t);

    default boolean validateAll(List<T> items) {
        return items.stream().allMatch(this::validate); // uses private helper
    }

    private boolean logAndValidate(T t) { // Java 9+
        System.out.println("Validating: " + t);
        return validate(t);
    }
}
```

---

## Java 10 Features

### Local Variable Type Inference (`var`)
```java
// var infers the type from the right-hand side
var list = new ArrayList<String>();  // ArrayList<String>
var map = new HashMap<String, List<Integer>>();
var entry = Map.entry("key", 42);

// CANNOT use var for:
var x;              // ERROR: no initializer
var x = null;       // ERROR: type cannot be inferred from null
var x = (String) null; // OK: explicit cast gives type

// In for-loops
for (var item : list) {
    System.out.println(item.toUpperCase()); // compiler knows item is String
}

// In try-with-resources
try (var stream = Files.lines(Path.of("file.txt"))) {
    stream.forEach(System.out::println);
}
```

**Limitations:** `var` is only for local variables — not for fields, method parameters, or return types.

### Unmodifiable Collection Copies
```java
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> copy = List.copyOf(original);  // unmodifiable copy

Set<String> setCopy = Set.copyOf(original);
Map<String, Integer> mapCopy = Map.copyOf(Map.of("a", 1));
```

### Additional Stream/Collectors Enhancements
```java
// Collectors.toUnmodifiableList/Set/Map
List<String> unmod = stream.collect(Collectors.toUnmodifiableList());

// Optional.orElseThrow() — cleaner than get()
String value = Optional.of("hello").orElseThrow(); // NoSuchElementException if empty
```

---

## Java 11 Features (LTS)

### String Methods
```java
String s = "  hello world  ";
s.isBlank();           // false (only true for empty or whitespace-only)
"   ".isBlank();       // true

s.strip();             // "hello world" — Unicode-aware trim
s.stripLeading();      // "hello world  "
s.stripTrailing();     // "  hello world"
// strip() is Unicode-aware; trim() only handles ASCII whitespace

"ha".repeat(3);        // "hahaha"

"line1\nline2\nline3".lines()
    .collect(Collectors.toList()); // ["line1", "line2", "line3"]
```

### Files Utility Methods
```java
Path path = Path.of("myfile.txt");

// Write and read strings directly
Files.writeString(path, "Hello, Java 11!");
String content = Files.readString(path);
```

### HTTP Client API (Standard, was incubator in Java 9/10)
```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Accept", "application/json")
    .GET()
    .build();

// Synchronous
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
System.out.println(response.statusCode()); // 200
System.out.println(response.body());

// Asynchronous
CompletableFuture<HttpResponse<String>> asyncResponse =
    client.sendAsync(request, BodyHandlers.ofString());
asyncResponse.thenApply(HttpResponse::body).thenAccept(System.out::println);
```

### Predicate.not()
```java
List<String> nonEmpty = strings.stream()
    .filter(Predicate.not(String::isBlank))  // cleaner than s -> !s.isBlank()
    .collect(Collectors.toList());
```

### Running Single-File Programs
```bash
# No compilation step needed
java HelloWorld.java
```

---

## Java 14 Features

### Switch Expressions (Finalized)
```java
// Old switch (statement, fall-through, verbose)
int numLetters;
switch (day) {
    case MONDAY: case FRIDAY: case SUNDAY: numLetters = 6; break;
    case TUESDAY: numLetters = 7; break;
    default: numLetters = 8;
}

// New switch expression (Java 14+) — no fall-through, returns value
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    case THURSDAY, SATURDAY -> 8;
    case WEDNESDAY -> 9;
};

// With blocks (yield keyword to return value)
String description = switch (status) {
    case ACTIVE -> "User is active";
    case INACTIVE -> {
        log.info("Inactive user");
        yield "User is inactive";  // yield, not return!
    }
    default -> "Unknown status";
};
```

**Interview Tip:** Switch expressions are exhaustive — the compiler enforces all cases are handled (with enums, no default needed if all values are covered).

### Pattern Matching for instanceof (Finalized in Java 16)
```java
// Old way
if (obj instanceof String) {
    String s = (String) obj;  // redundant cast
    System.out.println(s.length());
}

// Java 16+ — pattern variable
if (obj instanceof String s) {
    System.out.println(s.length());  // s is in scope here
}

// Combined with conditions
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}

// In switch (Java 21)
String result = switch (obj) {
    case Integer i -> "Integer: " + i;
    case String s -> "String: " + s;
    case null -> "null value";
    default -> "Other: " + obj;
};
```

### Helpful NullPointerExceptions
```java
// Java 14+ NullPointerException includes exact location
// Before: NullPointerException (no detail)
// After:  Cannot invoke "String.length()" because "user.address.city" is null
user.getAddress().getCity().length();
```

---

## Java 15 Features

### Text Blocks (Finalized)
```java
// Old multi-line strings
String json = "{\n" +
    "  \"name\": \"John\",\n" +
    "  \"age\": 30\n" +
    "}";

// Text block (triple-quote)
String json = """
        {
          "name": "John",
          "age": 30
        }
        """;  // trailing newline included; closing """ position matters

// In Spring Boot — SQL queries, JSON templates
String sql = """
        SELECT u.id, u.name, o.total
        FROM users u
        JOIN orders o ON o.user_id = u.id
        WHERE u.active = true
        ORDER BY o.total DESC
        """;

// \s — explicit space (prevents trailing whitespace stripping)
// \ — line continuation (no newline in output)
String oneLine = """
        Hello, \
        World!
        """; // "Hello, World!\n"
```

---

## Java 16 Features

### Records (Finalized) — Major Feature
```java
// Define an immutable data carrier
public record Point(int x, int y) { }

// Generated automatically:
// - final fields: private final int x; private final int y;
// - canonical constructor: Point(int x, int y)
// - accessors: x(), y() (not getX/getY)
// - equals(), hashCode(), toString()

Point p = new Point(3, 4);
System.out.println(p.x());    // 3
System.out.println(p);        // Point[x=3, y=4]

// Custom compact constructor (validates/normalizes)
public record Range(int min, int max) {
    public Range {  // compact constructor — no parameter list
        if (min > max) throw new IllegalArgumentException("min > max");
        // fields are assigned after this block automatically
    }
}

// Additional constructors
public record Person(String name, int age) {
    public Person(String name) {
        this(name, 0);  // delegates to canonical
    }
}

// Records CAN: implement interfaces, have static fields/methods, have instance methods
// Records CANNOT: extend classes (implicitly extend Record), be abstract, have mutable state

// Use as DTO in Spring
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email,
    @Min(18) int age
) { }

@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest req) {
    // req.name(), req.email(), req.age()
}

// Use with JPA (as projection interface alternative)
interface UserSummary {
    String getName();
    String getEmail();
}
// Records as projections require @Query with constructor expression
```

**Interview Tip:** Records cannot be used as JPA entities (they're immutable, JPA needs mutable no-arg constructor). Use them as DTOs, request/response bodies, and value objects.

### Stream.toList()
```java
// Java 16+ — shorthand for collect(Collectors.toList())
List<String> names = users.stream()
    .map(User::getName)
    .toList();  // unmodifiable list (unlike Collectors.toList() which is mutable)
```

---

## Java 17 Features (LTS) — Major Release

### Sealed Classes (Finalized)
```java
// Sealed class restricts which classes can extend it
public sealed class Shape
    permits Circle, Rectangle, Triangle { }

public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double radius() { return radius; }
}

public final class Rectangle extends Shape {
    private final double width, height;
    // ...
}

public non-sealed class Triangle extends Shape {
    // non-sealed: Triangle can be extended by anyone
}

// sealed + record (very common pattern)
public sealed interface Result<T> permits Result.Success, Result.Failure {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error) implements Result<T> {}
}

// Usage with exhaustive switch (Java 21 pattern matching)
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t -> t.base() * t.height() / 2;
    // no default needed — compiler knows all permitted subclasses
};
```

**Use cases in Domain Modeling:**
- Payment methods: `sealed interface Payment permits CreditCard, PayPal, BankTransfer`
- Event types: `sealed interface DomainEvent permits OrderCreated, OrderCancelled`
- Result types: `sealed interface Result<T> permits Success, Failure`

---

## Java 21 Features (LTS) — The Most Important Modern Java Release

### Virtual Threads — Project Loom (The Biggest Feature)

**The Problem with Platform Threads:**
- Traditional Java threads map 1:1 to OS threads
- Creating/blocking on OS threads is expensive (1MB stack, context switch overhead)
- Thread-per-request model limits throughput (e.g., 200 concurrent requests = 200 OS threads)
- When a thread blocks on I/O (DB query, HTTP call), the OS thread is wasted

**Virtual Threads Solution:**
```java
// Platform thread (old)
Thread platformThread = new Thread(() -> doWork());
platformThread.start();

// Virtual thread (Java 21)
Thread virtualThread = Thread.ofVirtual().start(() -> doWork());

// Virtual thread factory
ThreadFactory factory = Thread.ofVirtual().factory();
ExecutorService executor = Executors.newThreadPerTaskExecutor(factory);

// The key executor for virtual threads
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();

// Submit millions of tasks — each gets its own virtual thread
for (int i = 0; i < 1_000_000; i++) {
    virtualExecutor.submit(() -> {
        Thread.sleep(1000);  // blocking: virtual thread is unmounted, not OS thread
        return "done";
    });
}
```

**How Virtual Threads Work:**
- Virtual threads are JVM-managed, not OS-managed
- Many virtual threads share few OS threads ("carrier threads")
- When a virtual thread blocks (I/O, sleep, lock), it is **unmounted** from carrier thread
- Carrier thread picks up another virtual thread to run
- When I/O completes, virtual thread is **remounted** to a carrier thread

**Pinning (What to Avoid):**
```java
// Pinning: virtual thread is stuck to carrier thread, blocking it
// Happens with: synchronized blocks, native methods

// BAD — synchronized pins the carrier thread
synchronized (lock) {
    Thread.sleep(1000);  // virtual thread is pinned during sleep!
}

// GOOD — use ReentrantLock instead
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    Thread.sleep(1000);  // virtual thread unmounts correctly
} finally {
    lock.unlock();
}
```

**Spring Boot with Virtual Threads (Spring Boot 3.2+):**
```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true  # enables virtual threads for Tomcat/Jetty/Undertow and @Async
```
```java
// Or programmatically
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}
```

**When NOT to Use Virtual Threads:**
- CPU-intensive tasks (no I/O blocking): use ForkJoinPool / parallel streams instead
- When you use synchronized heavily (pinning): migrate to ReentrantLock
- Not a replacement for reactive programming in all cases

### Structured Concurrency (Java 21)
```java
// Traditional: error-prone parallel task management
Future<User> userFuture = executor.submit(() -> fetchUser(id));
Future<Order> orderFuture = executor.submit(() -> fetchOrder(id));
User user = userFuture.get();   // what if orderFuture failed? userFuture leaks
Order order = orderFuture.get();

// Structured Concurrency: scope manages lifecycle
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    StructuredTaskScope.Subtask<User> user = scope.fork(() -> fetchUser(id));
    StructuredTaskScope.Subtask<Order> order = scope.fork(() -> fetchOrder(id));

    scope.join();           // wait for all
    scope.throwIfFailed();  // propagate any exception

    return new UserWithOrders(user.get(), order.get());
} // scope closes: cancels remaining tasks if any failed
```

### Scoped Values (Java 21)
```java
// ThreadLocal alternative — read-only, inheritable by child threads, no memory leak
public static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// Bind value for a scope
ScopedValue.where(CURRENT_USER, user).run(() -> {
    processOrder();  // can access CURRENT_USER.get() here
    // virtual threads forked within this scope inherit the value
});

// In method
void processOrder() {
    User user = CURRENT_USER.get();  // always present within the scope
}
```

**vs ThreadLocal:** ScopedValues are immutable, inherited by child threads, automatically cleaned up when scope exits (no memory leaks).

### Sequenced Collections (Java 21)
```java
// New interfaces: SequencedCollection, SequencedSet, SequencedMap
// All ordered collections now have consistent first/last/reversed API

List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();   // "a" — replaces get(0)
list.getLast();    // "c" — replaces get(list.size()-1)
list.removeFirst(); // removes "a"
list.removeLast();  // removes "c"
list.reversed();    // reversed view (not a copy)
list.addFirst("z"); // insert at front

// LinkedHashSet now implements SequencedSet
LinkedHashSet<String> set = new LinkedHashSet<>(Set.of("x", "y", "z"));
set.getFirst();   // first insertion-order element

// LinkedHashMap now implements SequencedMap
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("a", 1); map.put("b", 2);
map.firstEntry();  // Map.Entry("a", 1)
map.lastEntry();   // Map.Entry("b", 2)
```

### Record Patterns (Java 21)
```java
record Point(int x, int y) { }
record Segment(Point start, Point end) { }

// Deconstruct record in pattern matching
Object obj = new Point(1, 2);
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);  // x and y are bound
}

// Nested record patterns
if (obj instanceof Segment(Point(int x1, int y1), Point(int x2, int y2))) {
    System.out.println("from " + x1 + "," + y1 + " to " + x2 + "," + y2);
}
```

### Pattern Matching for switch (Java 21, Finalized)
```java
// Exhaustive switch with type patterns
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "Positive integer: " + i;  // guarded pattern
        case Integer i            -> "Non-positive integer: " + i;
        case String s             -> "String of length " + s.length();
        case null                 -> "null";
        default                   -> "Unknown: " + obj.getClass().getName();
    };
}

// With sealed types — no default needed
double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
    };
}
```

### Unnamed Patterns and Variables (Java 21)
```java
// _ as unnamed variable — ignore values you don't need
if (obj instanceof Point(int x, _)) {  // ignore y
    System.out.println("x = " + x);
}

// In catch blocks
try {
    riskyOperation();
} catch (IOException _) {  // exception object not needed
    System.out.println("IO error occurred");
}

// In for-each
for (var _ : list) {  // just counting iterations
    count++;
}
```

---

## Impact on Spring Boot Development

### Virtual Threads Change Thread-per-Request Model
```yaml
# Spring Boot 3.2+ — one line to enable virtual threads
spring.threads.virtual.enabled: true
```
- Tomcat uses virtual threads per request — supports 100k+ concurrent requests
- @Async methods use virtual threads automatically
- JDBC, RestTemplate, WebClient all benefit from virtual thread unmounting on I/O

### Records as Spring DTOs
```java
// Clean request/response objects
public record CreateProductRequest(
    @NotBlank String name,
    @Positive BigDecimal price,
    @NotNull Category category
) {}

public record ProductResponse(Long id, String name, BigDecimal price) {
    public static ProductResponse from(Product p) {
        return new ProductResponse(p.getId(), p.getName(), p.getPrice());
    }
}
```

### Text Blocks for SQL in Spring Data
```java
@Query("""
    SELECT new com.example.UserSummary(u.name, COUNT(o.id))
    FROM User u LEFT JOIN u.orders o
    WHERE u.active = true
    GROUP BY u.name
    HAVING COUNT(o.id) > :minOrders
    """)
List<UserSummary> findActiveUsersWithMinOrders(@Param("minOrders") int minOrders);
```

### Sealed Classes for Domain Events
```java
public sealed interface OrderEvent permits
    OrderCreated, OrderConfirmed, OrderShipped, OrderCancelled {}

record OrderCreated(Long orderId, LocalDateTime at) implements OrderEvent {}
record OrderCancelled(Long orderId, String reason) implements OrderEvent {}

// Handler with exhaustive switch
void handle(OrderEvent event) {
    switch (event) {
        case OrderCreated e    -> notifyCustomer(e.orderId());
        case OrderConfirmed e  -> startFulfillment(e.orderId());
        case OrderShipped e    -> sendTrackingEmail(e.orderId());
        case OrderCancelled e  -> refundPayment(e.orderId(), e.reason());
    }
}
```

---

## Java Versions Quick Reference Table

| Version | LTS | Key Features |
|---------|-----|--------------|
| Java 9  | No  | JPMS modules, Collection.of(), Stream enhancements |
| Java 10 | No  | var keyword |
| Java 11 | Yes | String methods, HTTP Client, Files.readString() |
| Java 12 | No  | Switch expression (preview) |
| Java 14 | No  | Switch expressions (final), Records (preview), Pattern instanceof (preview) |
| Java 15 | No  | Text blocks (final), Sealed classes (preview) |
| Java 16 | No  | Records (final), Pattern instanceof (final), Stream.toList() |
| Java 17 | Yes | Sealed classes (final), Strong JDK encapsulation |
| Java 21 | Yes | Virtual threads, Structured concurrency, Sequenced collections, Record patterns, Pattern switch |

---

## Interview Questions & Answers

**Q1: What are virtual threads? How do they differ from platform threads?**

Virtual threads are lightweight JVM-managed threads that can be created by the millions without the overhead of OS threads. Platform threads have a 1:1 mapping to OS threads (expensive, ~1MB stack each). Virtual threads are multiplexed over a small pool of carrier threads; when a virtual thread blocks on I/O, it is unmounted from its carrier thread (freeing it for others) and remounted when I/O completes. This enables the thread-per-request model to scale to hundreds of thousands of concurrent requests.

**Q2: What is "pinning" in virtual threads?**

Pinning occurs when a virtual thread is stuck to its carrier thread even while blocking — the carrier thread cannot pick up other work. Pinning happens inside `synchronized` blocks and native method frames. The fix is to replace `synchronized` with `ReentrantLock`. Spring Boot's built-in libraries were updated to avoid pinning.

**Q3: What is a Record in Java? How is it different from a regular class?**

A record is a transparent data carrier. It automatically generates: canonical constructor, accessor methods (field-name based, not getX), `equals()`, `hashCode()`, and `toString()`. Records are implicitly final and cannot extend classes (they implicitly extend `java.lang.Record`). They CAN implement interfaces. Fields are implicitly `private final`. Records are ideal for DTOs, value objects, and request/response bodies — NOT for JPA entities (which need mutable state and a no-arg constructor).

**Q4: Can a Record have custom constructors?**

Yes. The compact constructor (no parameter list) runs before field assignment — useful for validation/normalization. Additional constructors must delegate to the canonical constructor using `this(...)`.

**Q5: What are Sealed Classes? What problem do they solve?**

Sealed classes restrict which classes can extend/implement them using `permits`. They solve the problem of open inheritance hierarchies that make exhaustive pattern matching impossible. With sealed types, the compiler knows all subtypes at compile time and can require switch expressions to handle all cases without a default branch. Perfect for ADTs (algebraic data types), domain events, and result types.

**Q6: What is the difference between `var` and dynamic typing?**

`var` is static type inference at compile time — the type is fixed once inferred. It's purely a compile-time feature; the bytecode is identical to writing the explicit type. Java remains statically typed. `var` cannot be used for fields, method parameters, or return types.

**Q7: How do Text Blocks handle indentation?**

The closing `"""` position determines the baseline indentation to strip. All lines have the common leading whitespace removed. Lines can have trailing whitespace preserved with `\s`. The `\` at end of a line prevents a newline from being inserted.

**Q8: What is Structured Concurrency?**

Structured Concurrency (Java 21 preview) ensures that child tasks (subtasks) do not outlive their parent scope. `StructuredTaskScope` creates a scope; tasks forked within it are guaranteed to finish when the scope closes. If any subtask fails, remaining ones are cancelled. This prevents thread leaks and makes error propagation clean — unlike raw `CompletableFuture` chains.

**Q9: What are Sequenced Collections?**

Java 21 added `SequencedCollection`, `SequencedSet`, and `SequencedMap` interfaces that provide a consistent API for first/last element access (`getFirst()`, `getLast()`) and reversed view (`reversed()`) for all ordered collections (List, LinkedHashSet, LinkedHashMap, etc.).

**Q10: What are the key Java 21 features that impact Spring Boot applications?**

1. **Virtual threads** (`spring.threads.virtual.enabled=true`) — Tomcat handles each request on a virtual thread, enabling massive concurrency for I/O-bound workloads without reactive programming
2. **Records** — Cleaner DTOs and request/response bodies with less boilerplate
3. **Text blocks** — Readable inline SQL queries and JSON templates
4. **Sealed classes + Pattern matching switch** — Type-safe domain event handling without instanceof chains
5. **Structured concurrency** — Safer parallel operations replacing some CompletableFuture patterns

**Q11: When should you NOT use virtual threads?**

- CPU-intensive operations (no blocking I/O) — use parallel streams or ForkJoinPool
- Code that uses `synchronized` heavily (causes pinning) — refactor to `ReentrantLock` first
- When your framework/driver has synchronization issues (check for pinning warnings in JVM logs: `-Djdk.tracePinnedThreads=full`)

**Q12: What is the difference between `switch` statement and `switch` expression?**

Switch statements are side-effecting constructs that execute code. Switch expressions return a value and are exhaustive (compiler enforces all cases). Switch expressions use `->` (no fall-through) or `yield` for multi-line cases. Switch expressions were finalized in Java 14.

**Q13: How do Collection factory methods differ from `Arrays.asList()`?**

`List.of()` returns a truly immutable list — no set/add/remove. `Arrays.asList()` returns a fixed-size but mutable list (set is allowed, add/remove throw). `List.of()` also disallows null elements; `Arrays.asList()` allows nulls.

**Q14: What is the `var` keyword? What are its limitations?**

`var` is local variable type inference introduced in Java 10. Limitations: only for local variables (not fields, parameters, return types); requires an initializer; cannot initialize to `null` alone; cannot be used with array initializer shorthand (`var arr = {1, 2, 3}` is illegal).

**Q15: What is the difference between `strip()` and `trim()`?**

`trim()` removes ASCII whitespace (chars ≤ ' '). `strip()` is Unicode-aware and removes all whitespace as defined by `Character.isWhitespace()`, including Unicode spaces. `strip()` is the modern replacement.

---

## Common Pitfalls

| Pitfall | Explanation |
|---------|-------------|
| Records as JPA entities | Records are immutable; JPA needs mutable entities with no-arg constructors |
| Virtual thread + synchronized = pinning | Use ReentrantLock in performance-critical virtual thread code |
| `var` for null | `var x = null` doesn't compile — type cannot be inferred |
| `List.of()` null elements | Throws NullPointerException — use `new ArrayList<>()` if null is needed |
| Sealed class `permits` | All permitted subclasses must be in same package or module |
| Switch expression non-exhaustive | Compiler error if not all cases covered (good!) |
| `Stream.toList()` vs `Collectors.toList()` | `toList()` (Java 16) returns unmodifiable; `Collectors.toList()` returns mutable |
