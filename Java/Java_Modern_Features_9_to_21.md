# Java Modern Features: Java 9 to Java 21 — Complete Interview Guide

## Overview

Java releases since Java 8 have transformed the language significantly. Java 17 and Java 21 are both Long-Term Support (LTS) releases — the current baseline for modern Java projects. This guide covers every major feature from Java 9 through Java 21 with deep explanations, code examples, and interview Q&A.

---

## Table of Contents

1. [Java 9 Features](#java-9-features)
2. [Java 10 Features](#java-10-features)
3. [Java 11 Features (LTS)](#java-11-features-lts)
4. [Java 14 Features](#java-14-features)
5. [Java 15 Features](#java-15-features)
6. [Java 16 Features](#java-16-features)
7. [Java 17 Features (LTS)](#java-17-features-lts--major-release)
8. [Java 21 Features (LTS)](#java-21-features-lts--the-most-important-modern-java-release)
9. [Impact on Spring Boot Development](#impact-on-spring-boot-development)
10. [Java Versions Quick Reference Table](#java-versions-quick-reference-table)
11. [Interview Questions & Answers](#interview-questions--answers)
12. [Common Pitfalls](#common-pitfalls)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

> **New to modern Java? Start here.** There are a LOT of features below, but for interviews you don't need to master all of them equally. The highest-value features to truly understand are: **Records**, **Sealed classes**, **Text blocks**, **`var`**, **Pattern matching for `switch`/`instanceof`**, and **Virtual Threads (Java 21)**. If you nail those six, you'll handle 90% of "what's new in Java" questions confidently. Skim the rest for awareness, but spend your real study time here.

---

## Java 9 Features

### Module System (JPMS — Java Platform Module System)

The module system (Project Jigsaw) introduces strong encapsulation at the package level.

> **Think of it like:** an office building with locked rooms. Before modules, every room was wide open — anyone could walk in. With modules, the building (your module) keeps most rooms private, and you must explicitly post a sign saying which rooms are open to visitors. If a package isn't `exports`-ed, nobody outside can get in.

```java
// module-info.java (in src/main/java/) — the "front desk rules" for this module
module com.example.orderservice {
    requires java.base;           // implicit, always present (every module needs the core library)
    requires spring.context;       // we depend on Spring — needed to compile AND run
    requires transitive java.sql;  // anyone who uses US also automatically gets java.sql

    exports com.example.orderservice.api;           // this package is public to ALL other modules
    exports com.example.orderservice.internal to com.example.admin; // public ONLY to the admin module

    opens com.example.orderservice.model;           // let frameworks peek inside via reflection
    opens com.example.orderservice.dto to com.fasterxml.jackson.databind; // open only to Jackson (for JSON)

    uses com.example.spi.PaymentGateway;            // "I want to consume this service interface"
    provides com.example.spi.PaymentGateway
        with com.example.orderservice.StripeGateway; // "and here's my implementation of it"
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

> **Think of it like:** ordering a pre-sealed gift box. Once it's packed, you can't add or remove items — it's locked shut. `List.of(...)` hands you a finished, read-only collection in one line.

```java
// Java 9+ — unmodifiable, null-disallowed, no guaranteed order (Set/Map)
List<String> list = List.of("a", "b", "c");           // one-line read-only list
Set<String> set = Set.of("x", "y", "z");              // read-only set (no duplicates allowed)
Map<String, Integer> map = Map.of("one", 1, "two", 2); // key-value pairs, read-only
Map<String, Integer> map2 = Map.ofEntries(            // use ofEntries for many pairs (cleaner)
    Map.entry("one", 1),
    Map.entry("two", 2),
    Map.entry("three", 3)
);

// throws UnsupportedOperationException
list.add("d"); // ERROR — the box is sealed, you cannot add to it
```

**vs Arrays.asList():** Arrays.asList returns a fixed-size but mutable list; List.of is fully immutable.

### Stream API Enhancements

> **Think of it like:** `takeWhile` is reading a list from the top and stopping the moment a condition fails ("keep taking while items are small, stop at the first big one"). `dropWhile` is the opposite — skip items while the condition holds, then keep everything from there on. Both stop checking once the condition first flips.

```java
// takeWhile — KEEP elements while predicate is true, then STOP at the first false
List<Integer> result = Stream.of(1, 2, 3, 4, 5, 1, 2)
    .takeWhile(n -> n < 4)                  // 1,2,3 pass; 4 fails -> stop here (ignores the later 1,2)
    .collect(Collectors.toList()); // [1, 2, 3]

// dropWhile — SKIP elements while predicate is true, then keep ALL the rest
List<Integer> result2 = Stream.of(1, 2, 3, 4, 5)
    .dropWhile(n -> n < 3)                  // drops 1,2; 3 fails the test -> keep 3,4,5
    .collect(Collectors.toList()); // [3, 4, 5]

// Stream.ofNullable — if the value is null, you get an empty stream instead of a crash
Stream<String> s = Stream.ofNullable(null); // empty stream, not NPE

// Stream.iterate with predicate (acts like a for-loop: start, condition, step)
Stream.iterate(0, n -> n < 10, n -> n + 2)  // start at 0, while < 10, add 2 each time
    .forEach(System.out::println); // 0, 2, 4, 6, 8
```

### Optional Enhancements

> **Think of it like:** an `Optional` is a box that may or may not contain a value. These methods let you handle both cases cleanly instead of writing repetitive `if (value != null)` checks.

```java
Optional<String> opt = Optional.of("hello");   // a box that definitely holds "hello"

// ifPresentOrElse — run the first action if present, otherwise run the second
opt.ifPresentOrElse(
    s -> System.out.println("Found: " + s),    // runs when the box has a value
    () -> System.out.println("Not found")       // runs when the box is empty
);

// or() — if empty, supply a fallback Optional (the lambda only runs if needed)
Optional<String> result = opt.or(() -> Optional.of("default"));

// stream() — turn the Optional into a Stream of 0 or 1 element (handy inside stream pipelines)
long count = opt.stream().count(); // 1
```

### Private Methods in Interfaces

> **Think of it like:** giving an interface its own private "helper drawer." Default methods can now share common code through a private method, instead of copy-pasting the same logic into each one.

```java
interface Validator<T> {
    boolean validate(T t);                          // the method implementers must provide

    default boolean validateAll(List<T> items) {    // a ready-made method implementers inherit for free
        return items.stream().allMatch(this::validate); // true only if EVERY item is valid
    }

    private boolean logAndValidate(T t) { // Java 9+ — only callable inside this interface
        System.out.println("Validating: " + t);     // shared helper logic, hidden from the outside
        return validate(t);
    }
}
```

---

## Java 10 Features

### Local Variable Type Inference (`var`)

> **Think of it like:** letting the compiler read the label off the box for you. The right-hand side already says exactly what the type is, so instead of writing the type twice, you write `var` and the compiler figures it out. It's still strongly typed — `var` is NOT a flexible "any" type like in JavaScript.

```java
// var infers the type from the right-hand side (no guessing — it reads what you assigned)
var list = new ArrayList<String>();  // compiler knows this is ArrayList<String>
var map = new HashMap<String, List<Integer>>(); // saves repeating that long type name
var entry = Map.entry("key", 42);

// CANNOT use var for:
var x;              // ERROR: no initializer — nothing to read the type from
var x = null;       // ERROR: null has no type, so nothing can be inferred
var x = (String) null; // OK: the cast tells the compiler the type is String

// In for-loops
for (var item : list) {                       // 'item' is inferred as String from the list
    System.out.println(item.toUpperCase()); // compiler knows item is String
}

// In try-with-resources
try (var stream = Files.lines(Path.of("file.txt"))) { // 'stream' inferred as Stream<String>
    stream.forEach(System.out::println);
}
```

**Limitations:** `var` is only for local variables — not for fields, method parameters, or return types.

### Unmodifiable Collection Copies

> **Think of it like:** taking a photocopy of a document and laminating it. The copy captures the current contents, but nobody can write on it afterward — and editing the original doesn't change the laminated copy.

```java
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> copy = List.copyOf(original);  // a frozen snapshot — can't be modified

Set<String> setCopy = Set.copyOf(original);       // same idea for a Set
Map<String, Integer> mapCopy = Map.copyOf(Map.of("a", 1)); // and for a Map
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

> **Think of it like:** a set of handy text cleanup tools — checking if a string is "empty-ish," trimming invisible spaces off the edges, repeating text, and splitting a block of text into individual lines.

```java
String s = "  hello world  ";
s.isBlank();           // false — has real content besides spaces
"   ".isBlank();       // true — only whitespace counts as blank

s.strip();             // "hello world" — removes leading/trailing whitespace (Unicode-aware trim)
s.stripLeading();      // "hello world  " — only trims the front
s.stripTrailing();     // "  hello world" — only trims the end
// strip() is Unicode-aware; trim() only handles ASCII whitespace

"ha".repeat(3);        // "hahaha" — repeats the string N times

"line1\nline2\nline3".lines()        // splits a multi-line string into a stream of lines
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

> **Think of it like:** a built-in web browser for your code — a modern, official way to call other web services without pulling in an external library. It supports HTTP/2 and can wait for the reply (synchronous) or carry on and be notified later (asynchronous).

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)      // prefer the faster HTTP/2 protocol
    .connectTimeout(Duration.ofSeconds(10))  // give up if connecting takes over 10s
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users")) // the URL to call
    .header("Accept", "application/json")              // tell the server we want JSON back
    .GET()                                             // this is a GET request
    .build();

// Synchronous — your code WAITS here until the response arrives
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
System.out.println(response.statusCode()); // 200 = success
System.out.println(response.body());        // the response text

// Asynchronous — fire the request and keep going; handle the reply when it lands
CompletableFuture<HttpResponse<String>> asyncResponse =
    client.sendAsync(request, BodyHandlers.ofString());
asyncResponse.thenApply(HttpResponse::body).thenAccept(System.out::println); // print body when done
```

### Predicate.not()

> **Think of it like:** a "flip the answer" wrapper. Instead of writing the clunky `s -> !s.isBlank()`, you wrap the existing check and read it almost like English: "filter, not blank."

```java
List<String> nonEmpty = strings.stream()
    .filter(Predicate.not(String::isBlank))  // keep strings that are NOT blank — cleaner than s -> !s.isBlank()
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

> **Think of it like:** a switch that *hands you back an answer* instead of just doing chores. The old switch was a list of instructions ("do this, then break"); the new switch expression is a question that returns a single value you can store in a variable — like a vending machine that gives you exactly one item for your input.

```java
// Old switch (statement, fall-through, verbose)
int numLetters;
switch (day) {
    case MONDAY: case FRIDAY: case SUNDAY: numLetters = 6; break; // must remember 'break' or it falls through
    case TUESDAY: numLetters = 7; break;
    default: numLetters = 8;
}

// New switch expression (Java 14+) — no fall-through, returns value
int numLetters = switch (day) {          // the whole switch produces one value assigned to numLetters
    case MONDAY, FRIDAY, SUNDAY -> 6;    // arrow form: multiple labels, no break needed
    case TUESDAY -> 7;
    case THURSDAY, SATURDAY -> 8;
    case WEDNESDAY -> 9;
};

// With blocks (yield keyword to return value)
String description = switch (status) {
    case ACTIVE -> "User is active";     // simple one-liner case
    case INACTIVE -> {                   // need multiple statements? use a block
        log.info("Inactive user");
        yield "User is inactive";  // 'yield' is how a block hands back the value (NOT 'return')
    }
    default -> "Unknown status";
};
```

**Interview Tip:** Switch expressions are exhaustive — the compiler enforces all cases are handled (with enums, no default needed if all values are covered).

### Pattern Matching for instanceof (Finalized in Java 16)

> **Think of it like:** checking-and-unwrapping in one move. Before, you'd check "is this a String?" and then *separately* cast it to a String (two steps, repeated). Pattern matching does the check AND gives you the ready-to-use typed variable at the same time.

```java
// Old way
if (obj instanceof String) {
    String s = (String) obj;  // redundant cast — you already knew it was a String!
    System.out.println(s.length());
}

// Java 16+ — pattern variable: check + cast + name it, all at once
if (obj instanceof String s) {     // if obj is a String, bind it to 's' automatically
    System.out.println(s.length());  // s is already a String here — no cast needed
}

// Combined with conditions
if (obj instanceof String s && s.length() > 5) { // only true if it's a String AND longer than 5
    System.out.println("Long string: " + s);
}

// In switch (Java 21)
String result = switch (obj) {
    case Integer i -> "Integer: " + i;   // matches AND binds 'i' as an Integer
    case String s -> "String: " + s;     // matches AND binds 's' as a String
    case null -> "null value";           // switch can now handle null explicitly
    default -> "Other: " + obj;          // anything else
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

> **Think of it like:** a triple-quoted note pad. You paste multi-line text in exactly as it looks — no more gluing strings together with `\n` and escaped quotes. What you see between the `"""` marks is (almost) what you get.

```java
// Old multi-line strings — ugly: escaped quotes, manual \n, lots of +
String json = "{\n" +
    "  \"name\": \"John\",\n" +
    "  \"age\": 30\n" +
    "}";

// Text block (triple-quote) — write it like it reads
String json = """
        {
          "name": "John",
          "age": 30
        }
        """;  // trailing newline included; the closing """ position sets where indentation is stripped

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

> **Think of it like:** a data class that writes its own boilerplate. You just declare the fields, and the compiler auto-generates the constructor, the getters, plus `equals()`, `hashCode()`, and `toString()` for you. One line replaces ~50 lines of a traditional POJO.

```java
// Define an immutable data carrier — just list the fields, that's it
public record Point(int x, int y) { }

// Generated automatically (you write NONE of this):
// - final fields: private final int x; private final int y;
// - canonical constructor: Point(int x, int y)
// - accessors: x(), y() (note: x() not getX())
// - equals(), hashCode(), toString()

Point p = new Point(3, 4);
System.out.println(p.x());    // 3 — accessor is named after the field
System.out.println(p);        // Point[x=3, y=4] — toString() is auto-generated

// Custom compact constructor (validates/normalizes the inputs)
public record Range(int min, int max) {
    public Range {  // compact constructor — notice: no parameter list and no field assignments
        if (min > max) throw new IllegalArgumentException("min > max"); // validate before storing
        // the fields (min, max) are assigned automatically after this block runs
    }
}

// Additional constructors
public record Person(String name, int age) {
    public Person(String name) {
        this(name, 0);  // must call the canonical constructor (the full one) with this(...)
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

> **Think of it like:** a shortcut button. The old `.collect(Collectors.toList())` was a mouthful; `.toList()` does almost the same thing in far fewer keystrokes (with one catch: the result is read-only).

```java
// Java 16+ — shorthand for collect(Collectors.toList())
List<String> names = users.stream()
    .map(User::getName)     // pull just the name out of each user
    .toList();  // collect into a list — NOTE: unmodifiable (Collectors.toList() returns a mutable one)
```

---

## Java 17 Features (LTS) — Major Release

### Sealed Classes (Finalized)

> **Think of it like:** a parent saying "only these specific children are allowed in this family — no one else can join." A sealed type lists exactly which classes may extend it. Because the list is fixed and known, the compiler can guarantee a `switch` has covered every possible case.

```java
// Sealed class restricts which classes can extend it
public sealed class Shape
    permits Circle, Rectangle, Triangle { }  // ONLY these three are allowed to extend Shape

public final class Circle extends Shape {     // 'final' = no further subclassing of Circle
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double radius() { return radius; }
}

public final class Rectangle extends Shape {
    private final double width, height;
    // ...
}

public non-sealed class Triangle extends Shape {
    // non-sealed: reopens the door — Triangle CAN be extended by anyone again
}

// sealed + record (very common pattern — fixed set of typed outcomes)
public sealed interface Result<T> permits Result.Success, Result.Failure {
    record Success<T>(T value) implements Result<T> {}   // the "it worked" case, holds the value
    record Failure<T>(String error) implements Result<T> {} // the "it failed" case, holds the error
}

// Usage with exhaustive switch (Java 21 pattern matching)
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();  // c is auto-cast to Circle
    case Rectangle r -> r.width() * r.height();
    case Triangle t -> t.base() * t.height() / 2;
    // no default needed — compiler KNOWS these are the only 3 possible shapes
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

> **Think of it like:** the difference between hiring one expensive full-time worker for every single task versus having one supervisor who juggles a million lightweight to-do notes. Old "platform" threads each tie up a real OS worker (costly, limited). Virtual threads are cheap paper notes: when one is "waiting" (e.g., for a database reply), the supervisor sets it aside and picks up another — so a handful of real workers can handle a million tasks.

**Virtual Threads Solution:**
```java
// Platform thread (old) — backed by a real, heavy OS thread
Thread platformThread = new Thread(() -> doWork());
platformThread.start();

// Virtual thread (Java 21) — lightweight, managed by the JVM
Thread virtualThread = Thread.ofVirtual().start(() -> doWork());

// Virtual thread factory — produces virtual threads on demand
ThreadFactory factory = Thread.ofVirtual().factory();
ExecutorService executor = Executors.newThreadPerTaskExecutor(factory);

// The key executor for virtual threads — gives EACH task its own virtual thread
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();

// Submit millions of tasks — each gets its own virtual thread (this is fine!)
for (int i = 0; i < 1_000_000; i++) {
    virtualExecutor.submit(() -> {
        Thread.sleep(1000);  // while "sleeping", the virtual thread steps aside and frees the OS thread
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

> **Think of it like:** gluing a paper note to a worker's hand so they can't put it down. "Pinning" means a virtual thread gets stuck to its real OS thread and can't step aside while waiting — defeating the whole point. Certain old constructs (`synchronized`) cause this.

```java
// Pinning: virtual thread is stuck to carrier thread, blocking it
// Happens with: synchronized blocks, native methods

// BAD — synchronized PINS the virtual thread to its OS thread
synchronized (lock) {
    Thread.sleep(1000);  // can't step aside — the real OS thread is wasted during this sleep!
}

// GOOD — use ReentrantLock instead (no pinning)
ReentrantLock lock = new ReentrantLock();
lock.lock();           // acquire the lock without pinning
try {
    Thread.sleep(1000);  // virtual thread can correctly step aside (unmount) here
} finally {
    lock.unlock();      // always release in finally so the lock is never left held
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

> **Think of it like:** a group project with one deadline. All sub-tasks are started together and must finish together; if one teammate fails, the whole group stops instead of leaving others running forever. The `scope` block guarantees no "orphan" background tasks leak out.

```java
// Traditional: error-prone parallel task management
Future<User> userFuture = executor.submit(() -> fetchUser(id));
Future<Order> orderFuture = executor.submit(() -> fetchOrder(id));
User user = userFuture.get();   // if orderFuture later fails, userFuture's work may leak/hang
Order order = orderFuture.get();

// Structured Concurrency: the scope manages the lifecycle of all sub-tasks
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) { // if ANY task fails, cancel the rest
    StructuredTaskScope.Subtask<User> user = scope.fork(() -> fetchUser(id));   // start task 1
    StructuredTaskScope.Subtask<Order> order = scope.fork(() -> fetchOrder(id)); // start task 2 (in parallel)

    scope.join();           // wait until ALL forked tasks finish
    scope.throwIfFailed();  // if any task threw, re-throw it here

    return new UserWithOrders(user.get(), order.get()); // both succeeded — safe to read results
} // leaving the block closes the scope: any leftover tasks are cancelled automatically
```

### Scoped Values (Java 21)

> **Think of it like:** a visitor badge that's valid only inside one building and only for one visit. The value (e.g., the current user) is available everywhere within a scope, automatically expires when you leave, and can't be tampered with — unlike `ThreadLocal`, which you must remember to clean up manually.

```java
// ThreadLocal alternative — read-only, inheritable by child threads, no memory leak
public static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance(); // declare the "badge slot"

// Bind value for a scope — CURRENT_USER holds 'user' only inside this run() block
ScopedValue.where(CURRENT_USER, user).run(() -> {
    processOrder();  // anything called here can read CURRENT_USER.get()
    // virtual threads forked within this scope inherit the value automatically
});

// In method
void processOrder() {
    User user = CURRENT_USER.get();  // reads the value bound for the current scope
}
```

**vs ThreadLocal:** ScopedValues are immutable, inherited by child threads, automatically cleaned up when scope exits (no memory leaks).

### Sequenced Collections (Java 21)

> **Think of it like:** finally giving every ordered collection the same "first / last / reverse" buttons. Before, grabbing the last element of a List vs. a LinkedHashSet looked totally different and awkward. Now they all share one tidy, consistent set of methods.

```java
// New interfaces: SequencedCollection, SequencedSet, SequencedMap
// All ordered collections now have consistent first/last/reversed API

List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();   // "a" — cleaner than get(0)
list.getLast();    // "c" — cleaner than get(list.size()-1)
list.removeFirst(); // removes "a" (the front element)
list.removeLast();  // removes "c" (the back element)
list.reversed();    // a reversed VIEW (no copy made — changes reflect back)
list.addFirst("z"); // insert at the very front

// LinkedHashSet now implements SequencedSet
LinkedHashSet<String> set = new LinkedHashSet<>(Set.of("x", "y", "z"));
set.getFirst();   // the first element in insertion order

// LinkedHashMap now implements SequencedMap
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("a", 1); map.put("b", 2);
map.firstEntry();  // Map.Entry("a", 1) — first inserted pair
map.lastEntry();   // Map.Entry("b", 2) — last inserted pair
```

### Record Patterns (Java 21)

> **Think of it like:** unpacking a box and labeling each item in one move. Instead of checking "is this a Point?" and then calling `.x()` and `.y()` separately, you pull the fields straight out into named variables right in the pattern.

```java
record Point(int x, int y) { }
record Segment(Point start, Point end) { }

// Deconstruct record in pattern matching
Object obj = new Point(1, 2);
if (obj instanceof Point(int x, int y)) {   // if it's a Point, unpack it into x and y directly
    System.out.println(x + ", " + y);  // x and y are ready to use
}

// Nested record patterns — unpack a record inside a record, all at once
if (obj instanceof Segment(Point(int x1, int y1), Point(int x2, int y2))) {
    System.out.println("from " + x1 + "," + y1 + " to " + x2 + "," + y2); // all 4 fields extracted
}
```

### Pattern Matching for switch (Java 21, Finalized)

> **Think of it like:** a smart sorting machine that routes an item based on *what type it is* (and optional extra conditions), instead of just matching a single fixed value. It replaces long `if-else instanceof` chains with one clean, readable block.

```java
// Exhaustive switch with type patterns
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "Positive integer: " + i;  // 'when' adds an extra condition (guard)
        case Integer i            -> "Non-positive integer: " + i; // any other Integer
        case String s             -> "String of length " + s.length(); // matched + cast to String
        case null                 -> "null";                    // explicitly handle null
        default                   -> "Unknown: " + obj.getClass().getName(); // everything else
    };
}

// With sealed types — no default needed (compiler knows all the cases)
double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius(); // c is a Circle here
        case Rectangle r -> r.width() * r.height();            // r is a Rectangle here
    };
}
```

### Unnamed Patterns and Variables (Java 21)

> **Think of it like:** writing "N/A" in a form field you don't care about. The underscore `_` says "something goes here, but I'm not going to use it" — so you don't have to invent a throwaway variable name.

```java
// _ as unnamed variable — ignore values you don't need
if (obj instanceof Point(int x, _)) {  // grab x, ignore the y value entirely
    System.out.println("x = " + x);
}

// In catch blocks
try {
    riskyOperation();
} catch (IOException _) {  // we don't need the exception object itself, just the fact it happened
    System.out.println("IO error occurred");
}

// In for-each
for (var _ : list) {  // we don't use the element — we're only counting iterations
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

---

## Quick Reference Cheat Sheet

### LTS versions (the ones companies actually run in production)

```
LTS (Long-Term Support) = supported for years, safe for production
  Java 8   → LTS (still everywhere — the old baseline)
  Java 11  → LTS
  Java 17  → LTS
  Java 21  → LTS (current modern baseline)
Non-LTS (9, 10, 12-16, 18-20) = short-lived "feature preview" releases
Rule of thumb: production teams jump LTS → LTS (8 → 11 → 17 → 21)
```

### One-line "what it's for" per major feature

```
var                → skip repeating the type; compiler infers it (local vars only)
Records            → one-line immutable data class (auto getters/equals/hashCode/toString)
Sealed classes     → lock down WHO can extend a type (fixed set of subtypes)
Text blocks        → multi-line strings with """ — paste SQL/JSON as-is
Switch expression  → a switch that RETURNS a value (uses -> and yield)
Pattern matching   → check-the-type and cast in a single step (instanceof / switch)
Record patterns    → unpack a record's fields directly in the pattern
takeWhile/dropWhile→ keep / skip stream elements while a condition holds
Optional methods   → handle "value or nothing" without null checks
HTTP Client        → built-in modern HTTP/2 client (no external library)
Virtual threads    → millions of cheap threads; great for I/O-bound work
Structured concur. → start parallel tasks together, fail/cancel together
Scoped values      → safer, auto-cleaned ThreadLocal replacement
Sequenced colls.   → consistent getFirst()/getLast()/reversed() everywhere
```

### Copy-paste: Virtual Threads

```java
// 1) One-off virtual thread
Thread.ofVirtual().start(() -> doWork());

// 2) Executor that gives EACH task its own virtual thread (the common one)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // do blocking I/O here (DB call, HTTP call) — scales to huge concurrency
        return fetchSomething();
    });
} // executor auto-closes and waits for tasks (try-with-resources)

// 3) Spring Boot 3.2+ — enable for every web request with ONE line:
// application.yml ->  spring.threads.virtual.enabled: true
```

### Top gotchas (memorize before the interview)

```
Records ≠ JPA entities  → JPA needs a mutable, no-arg-constructor class
synchronized + virtual  → causes "pinning"; use ReentrantLock instead
var x = null            → won't compile (no type to infer)
List.of(null)           → throws NPE (no nulls allowed in factory collections)
Stream.toList()         → returns UNMODIFIABLE list (can't add/remove)
Switch expression       → must be exhaustive (cover all cases / add default)
Sealed permits          → subclasses must be same package or module
Virtual threads         → for I/O-bound work, NOT CPU-bound (use parallel streams there)
```

### 5-second decision helper

| If you need to... | Reach for... |
|---|---|
| A simple DTO / value object | **Record** |
| A fixed, closed set of types | **Sealed interface + records** |
| Branch on an object's type | **Pattern matching `switch`** |
| Embed SQL or JSON in code | **Text block** |
| Handle tons of I/O-bound requests | **Virtual threads** |
| Avoid repeating a long type name | **`var`** |
| Return a value from a switch | **Switch expression** |

---

*Last Updated: 2026-06-06*
