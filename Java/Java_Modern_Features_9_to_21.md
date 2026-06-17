# Java Modern Features: Java 9 to Java 21 — Complete Interview Guide

## Overview

Java releases since Java 8 have transformed the language significantly. Java 17 and Java 21 are both Long-Term Support (LTS) releases — the current baseline for modern Java projects. This guide covers every major feature from Java 9 through Java 21 with deep explanations, code examples, and interview Q&A.

---

> **New to modern Java? Start here.** There are a LOT of features below, but for interviews you don't need to master all of them equally. The highest-value features to truly understand are: **Records**, **Sealed classes**, **Text blocks**, **`var`**, **Pattern matching for `switch`/`instanceof`**, and **Virtual Threads (Java 21)**. If you nail those six, you'll handle 90% of "what's new in Java" questions confidently. Skim the rest for awareness, but spend your real study time here.

---

## Java 9 Features

### Module System (JPMS — Java Platform Module System)

**Awareness summary (juniors rarely write `module-info.java`).** The module system (Project Jigsaw) adds strong encapsulation at the package level: a module declares what it `requires` and which packages it `exports`. If a package isn't `exports`-ed, code outside the module can't access it.

```java
// module-info.java — basic shape
module com.example.orderservice {
    requires spring.context;              // depend on another module
    exports com.example.orderservice.api; // make this package visible to others
}
```

Other keywords: `requires transitive` (re-export a dependency), `opens` (allow reflective access for Spring/Hibernate/Jackson), `uses`/`provides` (ServiceLoader SPI).

**Interview Tip:** Spring Boot apps typically run on the classpath (not module path), so JPMS is not mandatory — just understand the basic concept.

### JShell (REPL)
A command-line REPL (`$ jshell`) for trying out Java snippets interactively without a full class — handy for quick experiments.

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

`takeWhile` keeps elements while a condition holds and stops at the first failure; `dropWhile` is the opposite (skip while true, then keep the rest). Both stop checking once the condition first flips.

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

Java 9+ lets an interface have `private` methods so its `default` methods can share common logic instead of duplicating it.

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

`List.copyOf` (and `Set.copyOf`/`Map.copyOf`) takes a frozen snapshot — the copy can't be modified, and later edits to the original don't affect it.

```java
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> copy = List.copyOf(original);  // a frozen snapshot — can't be modified
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

A built-in, modern HTTP/2 client for calling other web services without an external library. Supports synchronous (wait for the reply) and asynchronous (be notified later) calls.

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

// There is also an async variant: client.sendAsync(request, ...) returns a CompletableFuture.
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
// Old switch: a statement with fall-through — must remember 'break', assigns via a variable.

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
// (Pattern matching also works in switch — see "Pattern Matching for switch" under Java 21.)
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
// Old way: ugly concatenation with escaped quotes and manual \n
//   String json = "{\n  \"name\": \"John\"\n}";

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

**Why they exist:** Traditional platform threads map 1:1 to OS threads and are expensive (~1MB stack each), so the thread-per-request model caps out at a few thousand concurrent requests. Virtual threads are lightweight, JVM-managed threads — you can create millions. When one blocks on I/O it is *unmounted* from its underlying OS "carrier" thread (freeing it for other work) and *remounted* when the I/O completes.

> **Think of it like:** one supervisor juggling a million lightweight to-do notes. When a task is "waiting" (e.g., for a database reply), the supervisor sets it aside and picks up another — so a handful of real OS threads can handle huge numbers of I/O-bound tasks.

```java
// Platform thread (old) — backed by a real, heavy OS thread
Thread platformThread = new Thread(() -> doWork());
platformThread.start();

// Virtual thread (Java 21) — lightweight, managed by the JVM
Thread.ofVirtual().start(() -> doWork());

// The key executor — gives EACH task its own virtual thread (the common pattern)
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();
virtualExecutor.submit(() -> {
    Thread.sleep(1000);  // while "waiting", the virtual thread frees its OS thread
    return "done";
});
```

**Pinning (the one gotcha to know):** inside a `synchronized` block a virtual thread can get *pinned* — stuck to its OS thread and unable to step aside while blocking, defeating the purpose. Fix: use `ReentrantLock` instead of `synchronized` in hot paths.

**Spring Boot 3.2+ — enable with one line:**
```yaml
# application.yml
spring.threads.virtual.enabled: true  # virtual threads for Tomcat requests and @Async
```

**When NOT to use:** CPU-intensive work (no I/O to block on — use parallel streams / ForkJoinPool instead).

### Structured Concurrency (Java 21, preview)

**Awareness only — rarely written by juniors.** *What/why:* `StructuredTaskScope` lets you fork several parallel subtasks that are treated as one unit — they start together and finish together, and if any one fails the rest are cancelled (no leaked/orphan tasks). A cleaner, safer alternative to juggling raw `CompletableFuture` chains.

### Scoped Values (Java 21, preview)

**Awareness only.** *What/why:* `ScopedValue` is an immutable, auto-cleaned replacement for `ThreadLocal`. A value (e.g., the current user) is bound for the duration of a scope, inherited by child/virtual threads, and cleared automatically when the scope exits — avoiding the memory-leak pitfalls of `ThreadLocal`.

### Sequenced Collections (Java 21)

> **Think of it like:** finally giving every ordered collection the same "first / last / reverse" buttons. Before, grabbing the last element of a List vs. a LinkedHashSet looked totally different and awkward. Now they all share one tidy, consistent set of methods.

```java
// New interfaces: SequencedCollection, SequencedSet, SequencedMap
// All ordered collections now share a consistent first/last/reversed API

List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();    // "a" — cleaner than get(0)
list.getLast();     // "c" — cleaner than get(list.size()-1)
list.removeFirst(); // removes "a"
list.removeLast();  // removes "c"
list.reversed();    // a reversed VIEW (no copy — changes reflect back)
list.addFirst("z"); // insert at the very front
```

`LinkedHashSet` (`getFirst()`) and `LinkedHashMap` (`firstEntry()`/`lastEntry()`) implement these interfaces too.

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
With `spring.threads.virtual.enabled: true` (Spring Boot 3.2+), Tomcat runs each request on a virtual thread (100k+ concurrent requests), `@Async` methods use them automatically, and JDBC/RestTemplate/WebClient all benefit from unmounting on I/O.

### Records as Spring DTOs
Use records for request/response bodies and projections — `record CreateProductRequest(@NotBlank String name, @Positive BigDecimal price)` plus a static `from(entity)` factory for responses (see the Records section above).

### Text Blocks for SQL in Spring Data
Use text blocks for readable inline `@Query` JPQL/SQL instead of `+`-concatenated strings (see the Text Blocks section above).

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

**Q6: What is `var`, and how is it different from dynamic typing?**

`var` (Java 10) is local variable type inference — static, fixed once inferred at compile time (bytecode is identical to writing the explicit type), so Java stays statically typed. Limitations: only for local variables (not fields, parameters, or return types); requires an initializer; can't be `null` alone; no array-initializer shorthand (`var arr = {1, 2, 3}` is illegal).

**Q7: How do Text Blocks handle indentation?**

The closing `"""` position determines the baseline indentation to strip. All lines have the common leading whitespace removed. Lines can have trailing whitespace preserved with `\s`. The `\` at end of a line prevents a newline from being inserted.

**Q8: What is Structured Concurrency?**

Structured Concurrency (Java 21 preview) ensures that child tasks (subtasks) do not outlive their parent scope. `StructuredTaskScope` creates a scope; tasks forked within it are guaranteed to finish when the scope closes. If any subtask fails, remaining ones are cancelled. This prevents thread leaks and makes error propagation clean — unlike raw `CompletableFuture` chains.

**Q9: What are Sequenced Collections?**

Java 21 added `SequencedCollection`, `SequencedSet`, and `SequencedMap` interfaces that provide a consistent API for first/last element access (`getFirst()`, `getLast()`) and reversed view (`reversed()`) for all ordered collections (List, LinkedHashSet, LinkedHashMap, etc.).

**Q10: What are the key Java 21 features that impact Spring Boot applications?**

Virtual threads (`spring.threads.virtual.enabled=true`) for high I/O concurrency; records for cleaner DTOs; text blocks for inline SQL/JSON; sealed classes + pattern-matching switch for type-safe domain event handling.

**Q11: When should you NOT use virtual threads?**

- CPU-intensive operations (no blocking I/O) — use parallel streams or ForkJoinPool
- Code that uses `synchronized` heavily (causes pinning) — refactor to `ReentrantLock` first
- When your framework/driver has synchronization issues (check for pinning warnings in JVM logs: `-Djdk.tracePinnedThreads=full`)

**Q12: What is the difference between `switch` statement and `switch` expression?**

Switch statements are side-effecting constructs that execute code. Switch expressions return a value and are exhaustive (compiler enforces all cases). Switch expressions use `->` (no fall-through) or `yield` for multi-line cases. Switch expressions were finalized in Java 14.

**Q13: How do Collection factory methods differ from `Arrays.asList()`?**

`List.of()` returns a truly immutable list — no set/add/remove. `Arrays.asList()` returns a fixed-size but mutable list (set is allowed, add/remove throw). `List.of()` also disallows null elements; `Arrays.asList()` allows nulls.

**Q14: What is the difference between `strip()` and `trim()`?**

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

LTS = supported for years, safe for production. The LTS line is **8 → 11 → 17 → 21**; production teams jump LTS to LTS. Everything else (9, 10, 12-16, 18-20) is a short-lived feature-preview release.

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
