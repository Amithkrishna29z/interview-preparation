# Java Modern Features: Java 9 to Java 21 — Complete Interview Guide

## Overview

Java releases since Java 8 have transformed the language significantly. Java 17 and Java 21 are both Long-Term Support (LTS) releases — the current baseline for modern Java projects. This guide covers every major feature from Java 9 through Java 21 with code examples and interview Q&A.

> **Focus here first:** **Records**, **Sealed classes**, **Text blocks**, **`var`**, **Pattern matching for `switch`/`instanceof`**, and **Virtual Threads (Java 21)**. These six cover 90% of "what's new in Java" interview questions.

---

## Table of Contents

- [Java 9 Features](#java-9-features)
- [Java 10 Features](#java-10-features)
- [Java 11 Features (LTS)](#java-11-features-lts)
- [Java 14 Features](#java-14-features)
- [Java 15 Features](#java-15-features)
- [Java 16 Features](#java-16-features)
- [Java 17 Features (LTS)](#java-17-features-lts--major-release)
- [Java 21 Features (LTS)](#java-21-features-lts--the-most-important-modern-java-release)
- [Impact on Spring Boot Development](#impact-on-spring-boot-development)
- [Java Versions Quick Reference Table](#java-versions-quick-reference-table)
- [Interview Questions & Answers](#interview-questions--answers)
- [Common Pitfalls](#common-pitfalls)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Java 9 Features

### Module System (JPMS)

A module declares what it `requires` and which packages it `exports`. Unexported packages are inaccessible outside the module.

```java
module com.example.orderservice {
    requires spring.context;
    exports com.example.orderservice.api;
}
```

**Interview Tip:** Spring Boot apps typically run on the classpath (not module path) — just understand the basic concept.

### JShell (REPL)
A command-line REPL (`$ jshell`) for trying out Java snippets without a full class.

### Collection Factory Methods (Immutable)

> Think of it as a pre-sealed box — once packed you can't add or remove items.

```java
List<String> list = List.of("a", "b", "c");            // read-only
Set<String> set = Set.of("x", "y", "z");
Map<String, Integer> map = Map.of("one", 1, "two", 2);

list.add("d"); // UnsupportedOperationException
```

**vs Arrays.asList():** `Arrays.asList` is fixed-size but mutable; `List.of` is fully immutable and disallows nulls.

### Stream API Enhancements

```java
// takeWhile — keep while predicate is true, stop at first false
Stream.of(1, 2, 3, 4, 5).takeWhile(n -> n < 4).toList(); // [1, 2, 3]

// dropWhile — skip while true, keep the rest
Stream.of(1, 2, 3, 4, 5).dropWhile(n -> n < 3).toList(); // [3, 4, 5]

// Stream.ofNullable — empty stream instead of NPE
Stream<String> s = Stream.ofNullable(null);

// iterate with predicate (like a for-loop)
Stream.iterate(0, n -> n < 10, n -> n + 2).forEach(System.out::println); // 0,2,4,6,8
```

### Optional Enhancements

```java
Optional<String> opt = Optional.of("hello");

opt.ifPresentOrElse(
    s -> System.out.println("Found: " + s),
    () -> System.out.println("Not found")
);

Optional<String> result = opt.or(() -> Optional.of("default"));
long count = opt.stream().count(); // 1
```

### Private Methods in Interfaces

Java 9+ allows `private` methods in interfaces so `default` methods can share helper logic.

```java
interface Validator<T> {
    boolean validate(T t);
    default boolean validateAll(List<T> items) {
        return items.stream().allMatch(this::validate);
    }
    private boolean logAndValidate(T t) { // only callable inside this interface
        System.out.println("Validating: " + t);
        return validate(t);
    }
}
```

---

## Java 10 Features

### Local Variable Type Inference (`var`)

> The compiler reads the type off the right-hand side so you don't have to write it twice. It's still statically typed — not dynamic like JavaScript.

```java
var list = new ArrayList<String>();
var map = new HashMap<String, List<Integer>>();

// Cannot use var:
var x;           // ERROR: no initializer
var x = null;    // ERROR: null has no type

for (var item : list) { System.out.println(item.toUpperCase()); }
```

**Limitation:** Only for local variables — not fields, parameters, or return types.

### Unmodifiable Collection Copies

```java
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> copy = List.copyOf(original); // frozen snapshot
```

### Additional Stream/Collectors Enhancements
```java
List<String> unmod = stream.collect(Collectors.toUnmodifiableList());
String value = Optional.of("hello").orElseThrow(); // cleaner than get()
```

---

## Java 11 Features (LTS)

### String Methods

```java
"   ".isBlank();        // true — only whitespace
"  hi  ".strip();       // "hi" — Unicode-aware (preferred over trim())
"  hi  ".stripLeading(); // "hi  "
"ha".repeat(3);         // "hahaha"
"a\nb\nc".lines().toList(); // ["a", "b", "c"]
```

### Files Utility Methods
```java
Files.writeString(Path.of("file.txt"), "Hello, Java 11!");
String content = Files.readString(Path.of("file.txt"));
```

### HTTP Client API

A built-in HTTP/2 client — no external library needed.

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_2)
    .connectTimeout(Duration.ofSeconds(10))
    .build();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create("https://api.example.com/users"))
    .header("Accept", "application/json")
    .GET().build();

// Synchronous
HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
System.out.println(response.statusCode()); // 200
// Async variant: client.sendAsync(...) returns CompletableFuture
```

### Predicate.not()

```java
List<String> nonEmpty = strings.stream()
    .filter(Predicate.not(String::isBlank)) // cleaner than s -> !s.isBlank()
    .collect(Collectors.toList());
```

### Running Single-File Programs
```bash
java HelloWorld.java  # no compilation step needed
```

---

## Java 14 Features

### Switch Expressions (Finalized)

> The old switch was a list of instructions; the new switch expression returns a single value — like a vending machine that gives you exactly one item.

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;   // multiple labels, no break needed
    case TUESDAY -> 7;
    case THURSDAY, SATURDAY -> 8;
    case WEDNESDAY -> 9;
};

String description = switch (status) {
    case ACTIVE -> "User is active";
    case INACTIVE -> {
        log.info("Inactive user");
        yield "User is inactive"; // yield returns value from a block
    }
    default -> "Unknown status";
};
```

**Interview Tip:** Switch expressions are exhaustive — compiler enforces all cases are handled.

### Pattern Matching for instanceof (Finalized in Java 16)

> Check and cast in one move — no redundant cast needed.

```java
// Old way
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Java 16+ — check + cast + name, all at once
if (obj instanceof String s) {
    System.out.println(s.length());
}

if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}
```

### Helpful NullPointerExceptions
```java
// Before: NullPointerException (no detail)
// After:  Cannot invoke "String.length()" because "user.address.city" is null
user.getAddress().getCity().length();
```

---

## Java 15 Features

### Text Blocks (Finalized)

> Triple-quoted strings — paste multi-line text exactly as it looks, no `\n` or escaped quotes.

```java
String json = """
        {
          "name": "John",
          "age": 30
        }
        """;

String sql = """
        SELECT u.id, u.name
        FROM users u
        WHERE u.active = true
        """;

// \ prevents a newline; \s preserves a trailing space
String oneLine = """
        Hello, \
        World!
        """; // "Hello, World!\n"
```

---

## Java 16 Features

### Records (Finalized) — Major Feature

> Declare the fields, compiler auto-generates constructor, accessors, `equals()`, `hashCode()`, and `toString()`. One line replaces ~50 lines of a POJO.

```java
public record Point(int x, int y) { }

Point p = new Point(3, 4);
p.x();   // 3  — accessor named after field (not getX())
p;       // Point[x=3, y=4]  — auto toString()

// Compact constructor — validation before fields are assigned
public record Range(int min, int max) {
    public Range {
        if (min > max) throw new IllegalArgumentException("min > max");
    }
}

// Additional constructor must delegate to canonical constructor
public record Person(String name, int age) {
    public Person(String name) { this(name, 0); }
}

// Records CAN: implement interfaces, have static fields/methods, instance methods
// Records CANNOT: extend classes, be abstract, have mutable state

// DTO in Spring
public record CreateUserRequest(@NotBlank String name, @Email String email) { }
```

**Interview Tip:** Records cannot be JPA entities (immutable, no no-arg constructor). Use them as DTOs and value objects.

### Stream.toList()

```java
List<String> names = users.stream()
    .map(User::getName)
    .toList(); // unmodifiable — unlike Collectors.toList() which is mutable
```

---

## Java 17 Features (LTS) — Major Release

### Sealed Classes (Finalized)

> A parent saying "only these specific children are allowed." A sealed type lists exactly which classes may extend it, so the compiler can guarantee a `switch` covers every case.

```java
public sealed class Shape permits Circle, Rectangle, Triangle { }

public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double radius() { return radius; }
}

public non-sealed class Triangle extends Shape { } // Triangle can be freely extended

// Sealed interface + records (very common pattern)
public sealed interface Result<T> permits Result.Success, Result.Failure {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error) implements Result<T> {}
}

// Exhaustive switch — no default needed
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> t.base() * t.height() / 2;
};
```

**Use cases:** payment methods, domain events, result types.

---

## Java 21 Features (LTS) — The Most Important Modern Java Release

### Virtual Threads — Project Loom (The Biggest Feature)

**Why they exist:** Platform threads map 1:1 to OS threads (~1MB stack each), capping throughput at a few thousand concurrent requests. Virtual threads are JVM-managed and lightweight — create millions. When one blocks on I/O it unmounts from its carrier OS thread (freeing it) and remounts when I/O completes.

> Think of one supervisor juggling millions of to-do notes: when a task waits for a reply, the supervisor sets it aside and picks up another.

```java
// Platform thread (old)
Thread.ofPlatform().start(() -> doWork());

// Virtual thread (Java 21)
Thread.ofVirtual().start(() -> doWork());

// Common pattern: one virtual thread per task
ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor();
exec.submit(() -> { Thread.sleep(1000); return "done"; });
```

**Pinning (key gotcha):** Inside a `synchronized` block a virtual thread can get pinned to its carrier and can't step aside. Fix: use `ReentrantLock` instead of `synchronized` in hot paths.

**Spring Boot 3.2+:**
```yaml
spring.threads.virtual.enabled: true  # enables virtual threads for Tomcat + @Async
```

**When NOT to use:** CPU-intensive work — use parallel streams or ForkJoinPool instead.

### Structured Concurrency (Preview)

**Awareness only.** `StructuredTaskScope` forks parallel subtasks that live and die together — if any subtask fails the rest are cancelled. Cleaner than raw `CompletableFuture` chains.

### Scoped Values (Preview)

**Awareness only.** `ScopedValue` is an immutable, auto-cleaned replacement for `ThreadLocal`. Bound for the duration of a scope and inherited by child threads, avoiding `ThreadLocal` memory-leak pitfalls.

### Sequenced Collections (Java 21)

> Finally gives every ordered collection consistent first/last/reverse methods.

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();    // "a"
list.getLast();     // "c"
list.removeFirst();
list.removeLast();
list.reversed();    // reversed view (not a copy)
list.addFirst("z");
```

`LinkedHashSet` and `LinkedHashMap` implement these interfaces too.

### Record Patterns (Java 21)

> Unpack a record's fields directly in the pattern — check and destructure in one move.

```java
record Point(int x, int y) { }
record Segment(Point start, Point end) { }

if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);
}

// Nested destructuring
if (obj instanceof Segment(Point(int x1, int y1), Point(int x2, int y2))) {
    System.out.println("from " + x1 + "," + y1 + " to " + x2 + "," + y2);
}
```

### Pattern Matching for switch (Java 21, Finalized)

> Routes an object by type (and optional guards) instead of matching a fixed value — replaces long `if-else instanceof` chains.

```java
String describe(Object obj) {
    return switch (obj) {
        case Integer i when i > 0 -> "Positive integer: " + i;
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

> `_` means "something goes here but I don't need it."

```java
if (obj instanceof Point(int x, _)) { System.out.println("x = " + x); }

try { riskyOperation(); } catch (IOException _) { System.out.println("IO error"); }

for (var _ : list) { count++; }
```

---

## Impact on Spring Boot Development

| Feature | Spring Boot Use |
|---|---|
| Virtual threads | `spring.threads.virtual.enabled: true` — scales I/O-bound request handling |
| Records | Clean DTOs / request-response bodies with built-in validation annotations |
| Text blocks | Readable inline `@Query` JPQL/SQL |
| Sealed classes + pattern switch | Type-safe domain event handling |

```java
// Domain events with sealed interface + records
public sealed interface OrderEvent permits OrderCreated, OrderCancelled {}
record OrderCreated(Long orderId, LocalDateTime at) implements OrderEvent {}
record OrderCancelled(Long orderId, String reason) implements OrderEvent {}

void handle(OrderEvent event) {
    switch (event) {
        case OrderCreated e   -> notifyCustomer(e.orderId());
        case OrderCancelled e -> refundPayment(e.orderId(), e.reason());
    }
}
```

---

## Java Versions Quick Reference Table

| Version | LTS | Key Features |
|---------|-----|--------------|
| Java 9  | No  | JPMS modules, Collection.of(), Stream enhancements |
| Java 10 | No  | `var` keyword |
| Java 11 | Yes | String methods, HTTP Client, Files.readString() |
| Java 14 | No  | Switch expressions (final), Records (preview), Pattern instanceof (preview) |
| Java 15 | No  | Text blocks (final), Sealed classes (preview) |
| Java 16 | No  | Records (final), Pattern instanceof (final), Stream.toList() |
| Java 17 | Yes | Sealed classes (final), Strong JDK encapsulation |
| Java 21 | Yes | Virtual threads, Sequenced collections, Record patterns, Pattern switch |

---

## Interview Questions & Answers

**Q1: What are virtual threads? How do they differ from platform threads?**

Virtual threads are lightweight JVM-managed threads that can be created by the millions. Platform threads map 1:1 to OS threads (~1MB stack each). Virtual threads are multiplexed over a small pool of carrier threads — when one blocks on I/O it unmounts and frees its carrier, remounting when I/O completes. This lets the thread-per-request model scale to hundreds of thousands of concurrent requests.

**Q2: What is "pinning" in virtual threads?**

Pinning occurs when a virtual thread is stuck to its carrier thread while blocking, preventing other work from using that carrier. It happens inside `synchronized` blocks. Fix: replace `synchronized` with `ReentrantLock` in hot paths.

**Q3: What is a Record? How is it different from a regular class?**

A record is a transparent, immutable data carrier. It auto-generates: canonical constructor, field-named accessors (not `getX()`), `equals()`, `hashCode()`, and `toString()`. Records are implicitly `final`, cannot extend classes, but CAN implement interfaces. Not suitable for JPA entities — use them for DTOs and value objects.

**Q4: Can a Record have custom constructors?**

Yes. The compact constructor (no parameter list, no explicit field assignment) runs before fields are set — useful for validation. Additional constructors must delegate to the canonical constructor with `this(...)`.

**Q5: What are Sealed Classes? What problem do they solve?**

Sealed classes restrict which classes can extend/implement them via `permits`. They make inheritance hierarchies closed and finite, so the compiler can require switch expressions to cover all cases without a `default` branch. Ideal for domain events, result types, and ADTs.

**Q6: What is `var`? How is it different from dynamic typing?**

`var` is local-variable type inference (Java 10). The type is fixed at compile time — bytecode is identical to writing the explicit type. Java remains statically typed. Restrictions: local variables only, requires an initializer, cannot be `null` alone.

**Q7: How do Text Blocks handle indentation?**

The closing `"""` position sets the baseline — common leading whitespace is stripped from all lines. Use `\s` to preserve trailing spaces; use `\` at end of line to suppress a newline.

**Q8: What is Structured Concurrency?**

Structured Concurrency (Java 21 preview) ensures child tasks don't outlive their parent scope. `StructuredTaskScope` guarantees all forked tasks finish when the scope closes; if any subtask fails the rest are cancelled. Prevents thread leaks and simplifies error propagation vs. raw `CompletableFuture` chains.

**Q9: What are Sequenced Collections?**

Java 21 added `SequencedCollection`, `SequencedSet`, and `SequencedMap` interfaces, giving all ordered collections a consistent API: `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, and `reversed()`.

**Q10: What are the key Java 21 features that impact Spring Boot?**

Virtual threads for high I/O concurrency; records for cleaner DTOs; text blocks for inline SQL/JSON; sealed classes + pattern switch for type-safe domain event handling.

**Q11: When should you NOT use virtual threads?**

CPU-intensive work (no I/O to block on) — use parallel streams or ForkJoinPool. Also avoid when code uses `synchronized` heavily (causes pinning — refactor to `ReentrantLock` first).

**Q12: What is the difference between `switch` statement and `switch` expression?**

Switch statements are side-effecting constructs. Switch expressions return a value, are exhaustive (compiler enforces all cases), use `->` syntax (no fall-through), and use `yield` to return from blocks. Finalized in Java 14.

**Q13: How do `List.of()` and `Arrays.asList()` differ?**

`List.of()` is fully immutable — no set/add/remove, and disallows nulls. `Arrays.asList()` is fixed-size but mutable (set allowed, add/remove throw), and allows nulls.

**Q14: What is the difference between `strip()` and `trim()`?**

`trim()` removes ASCII whitespace (chars ≤ `' '`). `strip()` is Unicode-aware and removes all whitespace per `Character.isWhitespace()`. `strip()` is the modern replacement.

---

## Common Pitfalls

| Pitfall | Explanation |
|---------|-------------|
| Records as JPA entities | Records are immutable; JPA needs mutable entities with no-arg constructors |
| Virtual thread + `synchronized` = pinning | Use `ReentrantLock` in performance-critical virtual thread code |
| `var` with null | `var x = null` doesn't compile — type cannot be inferred |
| `List.of()` null elements | Throws NullPointerException — use `new ArrayList<>()` if nulls are needed |
| Sealed class `permits` | All permitted subclasses must be in the same package or module |
| Switch expression non-exhaustive | Compiler error if not all cases are covered (good!) |
| `Stream.toList()` vs `Collectors.toList()` | `toList()` (Java 16) returns unmodifiable; `Collectors.toList()` returns mutable |

---

## Quick Reference Cheat Sheet

### LTS Versions

**8 → 11 → 17 → 21** — production teams run LTS releases. Everything else (9, 10, 12–16, 18–20) is short-lived.

### One-line summary per feature

```
var                → skip repeating the type; compiler infers it (local vars only)
Records            → one-line immutable data class (auto getters/equals/hashCode/toString)
Sealed classes     → lock down WHO can extend a type (fixed set of subtypes)
Text blocks        → multi-line strings with """ — paste SQL/JSON as-is
Switch expression  → a switch that RETURNS a value (uses -> and yield)
Pattern matching   → check-the-type and cast in a single step (instanceof / switch)
Record patterns    → unpack a record's fields directly in the pattern
takeWhile/dropWhile→ keep / skip stream elements while a condition holds
HTTP Client        → built-in modern HTTP/2 client (no external library)
Virtual threads    → millions of cheap threads; great for I/O-bound work
Structured concur. → start parallel tasks together, fail/cancel together
Scoped values      → safer, auto-cleaned ThreadLocal replacement
Sequenced colls.   → consistent getFirst()/getLast()/reversed() everywhere
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

*Last Updated: 2026-06-18*
