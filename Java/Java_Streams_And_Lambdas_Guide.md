# Java Streams & Lambdas Interview Questions & Study Guide

## Overview

Java 8 introduced **lambdas** and the **Streams API** to bring functional programming to Java:

- **Less boilerplate** — a 6-line anonymous inner class becomes a 1-line lambda.
- **Declarative code** — describe **what** you want, not **how** to loop through it.
- **Composable pipelines** — chain `filter → map → collect` like readable sentences.

This is one of the **most heavily tested topics** in Java backend interviews. Expect questions on `map` vs `flatMap`, lazy evaluation, `reduce` vs `collect`, `Optional`, and parallel streams.

> Accurate for **Java 8 through Java 21**. Features added after Java 8 are clearly marked.

---

## Table of Contents

1. [Lambda Expressions](#lambda-expressions)
2. [Functional Interfaces](#functional-interfaces)
3. [Method References](#method-references)
4. [Streams — The Big Picture](#streams--the-big-picture)
5. [Intermediate Operations](#intermediate-operations)
6. [Terminal Operations](#terminal-operations)
7. [Collectors](#collectors)
8. [Optional](#optional)
9. [Primitive Streams](#primitive-streams)
10. [Parallel Streams](#parallel-streams)
11. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Lambda Expressions

A **lambda** is a short, nameless function you pass around like a value.

```java
// Before Java 8 — anonymous inner class
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) { return a.compareTo(b); }
});

// After — lambda
Collections.sort(names, (a, b) -> a.compareTo(b));
```

### Lambda syntax forms

```java
Runnable task          = () -> System.out.println("Running!");       // no params
Consumer<String> print = name -> System.out.println(name);           // one param
BinaryOperator<Integer> add = (a, b) -> a + b;                       // multiple params
Function<Integer, Integer> square = n -> n * n;                      // expression body (implicit return)

BiFunction<Integer, Integer, Integer> calc = (a, b) -> {             // block body
    int sum = a + b;
    return sum * 2;    // explicit return required with braces
};
```

### Key insight

A lambda is an instance of a **functional interface** — an interface with exactly one abstract method. `Runnable r = () -> System.out.println("hi")` is shorthand for implementing `Runnable.run()`.

---

## Functional Interfaces

A **functional interface** has **exactly one abstract method** (SAM — Single Abstract Method). `@FunctionalInterface` is optional but enforces this at compile time.

```java
@FunctionalInterface
interface Greeter {
    String greet(String name);
    default void hello() { System.out.println("Hello!"); }  // default methods don't count
}

Greeter g = name -> "Hi, " + name + "!";
System.out.println(g.greet("Alice"));   // Hi, Alice!
```

### Built-in functional interfaces — memorize this table

| Interface | Method | Takes | Returns | Purpose |
|---|---|---|---|---|
| `Function<T,R>` | `apply(T)` | 1 | R | Transform T into R |
| `BiFunction<T,U,R>` | `apply(T,U)` | 2 | R | Combine two inputs |
| `Consumer<T>` | `accept(T)` | 1 | void | Side effect on a value |
| `Supplier<T>` | `get()` | 0 | T | Produce a value |
| `Predicate<T>` | `test(T)` | 1 | boolean | Yes/no question |
| `UnaryOperator<T>` | `apply(T)` | 1 (T) | T | Transform T → T |
| `BinaryOperator<T>` | `apply(T,T)` | 2 (T) | T | Combine two T's |

```java
Function<String, Integer> length = s -> s.length();
Predicate<Integer> isEven = n -> n % 2 == 0;
Supplier<Double> random = () -> Math.random();
BinaryOperator<Integer> max = (a, b) -> a > b ? a : b;
```

> `Predicate` has `.and()`, `.or()`, `.negate()`. `Function` has `.andThen()` and `.compose()`.

---

## Method References

A **method reference** (`::`) is shorthand for a lambda that only calls one existing method.

```java
// 1. Static:            ClassName::staticMethod
Function<String, Integer> parse = Integer::parseInt;         // s -> Integer.parseInt(s)

// 2. Particular object: instance::method
Supplier<String> upper = greeting::toUpperCase;              // () -> greeting.toUpperCase()

// 3. Arbitrary object:  ClassName::instanceMethod
Function<String, String> toLower = String::toLowerCase;      // s -> s.toLowerCase()

// 4. Constructor:       ClassName::new
Supplier<ArrayList<String>> listMaker = ArrayList::new;      // () -> new ArrayList<>()
```

| Method Reference | Equivalent Lambda |
|---|---|
| `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| `System.out::println` | `s -> System.out.println(s)` |
| `String::toLowerCase` | `s -> s.toLowerCase()` |
| `ArrayList::new` | `() -> new ArrayList<>()` |

> Use a method reference only when it makes code **clearer**. For anything beyond a single method call, keep a lambda.

---

## Streams — The Big Picture

A **Stream** is a pipeline that pulls elements from a source, passes them through operations, and produces a result. It is **not a data structure** — it stores nothing.

**Assembly line analogy:** The source is the warehouse, intermediate operations are stations on a conveyor belt (lazy — the belt isn't moving), and the terminal operation starts the belt and packs the final result.

### The 3 parts of every stream pipeline

```java
long count = names.stream()                   // 1. SOURCE
                  .filter(n -> n.length() > 3) // 2. INTERMEDIATE (lazy)
                  .map(String::toUpperCase)     // 2. INTERMEDIATE (lazy)
                  .count();                     // 3. TERMINAL — triggers execution
```

1. **Source** — `list.stream()`, `Arrays.stream(arr)`, `Stream.of(...)`, `IntStream.range(...)`
2. **Intermediate** — return a new Stream, are **lazy**, can be chained
3. **Terminal** — produces a result, **triggers the pipeline**, consumes the stream

### Laziness

Intermediate ops do **nothing** until a terminal op runs. This enables fusing (each element flows through all steps once) and short-circuiting (`findFirst()` stops early).

```java
Stream<String> pipeline = names.stream()
    .filter(n -> { System.out.println("filtering: " + n); return n.length() > 3; });
// NOTHING printed yet — no terminal op

pipeline.count();   // NOW the filter runs
```

### A stream cannot be reused

```java
Stream<String> s = names.stream();
s.forEach(System.out::println);
s.count();   // IllegalStateException: stream already consumed
// FIX: create a fresh stream each time
```

### Collection vs Stream

| Aspect | Collection | Stream |
|---|---|---|
| Stores data? | Yes | No |
| Reusable? | Yes | No — single-use |
| Eager/Lazy? | Eager | Lazy |
| Iteration | External (you write the loop) | Internal (library loops) |

---

## Intermediate Operations

All lazy, all return a new Stream.

```java
List<Integer> nums = Arrays.asList(5, 3, 8, 1, 9, 3, 7, 2);

nums.stream().filter(n -> n > 4).forEach(System.out::println);     // [5, 8, 9, 7]
nums.stream().map(n -> n * 10).forEach(System.out::println);        // [50, 30, 80, ...]
nums.stream().distinct().forEach(System.out::println);              // removes second 3
nums.stream().sorted().forEach(System.out::println);                // [1, 2, 3, 3, 5, 7, 8, 9]
nums.stream().sorted(Comparator.reverseOrder()).forEach(System.out::println);
nums.stream().limit(3).forEach(System.out::println);                // [5, 3, 8]
nums.stream().skip(3).forEach(System.out::println);                 // [1, 9, 3, 7, 2]
nums.stream().peek(n -> System.out.println("saw: " + n)).map(n -> n * 2).forEach(System.out::println);
```

### flatMap — flattening a list of lists

- `map`: 1 element → 1 element
- `flatMap`: 1 element → a stream of elements, then **all merged into one flat stream**

```java
List<List<Integer>> listOfLists = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5),
    Arrays.asList(6, 7, 8, 9)
);

List<Integer> flat = listOfLists.stream()
    .flatMap(innerList -> innerList.stream())
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

| Operation | Mapping | Result |
|---|---|---|
| `map` | 1 → 1 | Same element count |
| `flatMap` | 1 → 0..many | Flattened, count can change |

> Classic interview signal: "you have `List<List<X>>` and want `List<X>`" → use `flatMap`.

---

## Terminal Operations

Trigger the pipeline, consume the stream, produce a result.

```java
List<Integer> nums = Arrays.asList(5, 3, 8, 1, 9);

nums.stream().forEach(System.out::println);
List<Integer> result   = nums.stream().filter(n -> n > 4).collect(Collectors.toList());
long howMany           = nums.stream().filter(n -> n > 4).count();           // 3
Optional<Integer> max  = nums.stream().max(Comparator.naturalOrder());       // Optional[9]
boolean anyBig         = nums.stream().anyMatch(n -> n > 8);                 // true
Optional<Integer> first = nums.stream().filter(n -> n > 4).findFirst();
List<Integer> list     = nums.stream().filter(n -> n > 4).toList();          // Java 16+
```

### reduce — combining everything into one value

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);

int sum     = nums.stream().reduce(0, (total, next) -> total + next);  // 10
int product = nums.stream().reduce(1, (a, b) -> a * b);                // 24

// Without identity → returns Optional (empty stream has no result)
Optional<Integer> maybeSum = nums.stream().reduce((a, b) -> a + b);
```

> **reduce vs collect**: Use `reduce` to fold into a **single immutable value** (sum, max). Use `collect` to accumulate into a **mutable container** (List, Map).

### findFirst vs findAny

- `findFirst()` — first element in encounter order. Deterministic.
- `findAny()` — any matching element. Faster in **parallel** streams because it doesn't enforce order.

---

## Collectors

`collect()` with `Collectors` is the most powerful terminal operation and one of the most heavily tested interview topics.

```java
record Employee(String name, String department, int salary) {}

List<Employee> employees = List.of(
    new Employee("Alice",   "Engineering", 90_000),
    new Employee("Bob",     "Engineering", 80_000),
    new Employee("Charlie", "Sales",       70_000),
    new Employee("Dave",    "Sales",       60_000),
    new Employee("Eve",     "HR",          75_000)
);
```

### toList / toSet / toMap

```java
List<String> names = employees.stream().map(Employee::name).collect(Collectors.toList());
Set<String> depts  = employees.stream().map(Employee::department).collect(Collectors.toSet());

Map<String, Integer> nameToSalary = employees.stream()
    .collect(Collectors.toMap(Employee::name, Employee::salary));

// GOTCHA: duplicate keys throw IllegalStateException — fix with a merge function
Map<String, Integer> deptToTotal = employees.stream()
    .collect(Collectors.toMap(
        Employee::department,
        Employee::salary,
        (existing, incoming) -> existing + incoming));  // {Engineering=170000, ...}
```

### groupingBy

```java
// Basic grouping
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));

// With downstream collectors
Map<String, Long>    countByDept  = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.counting()));
Map<String, Integer> salaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.summingInt(Employee::salary)));
Map<String, Double>  avgByDept    = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.averagingInt(Employee::salary)));
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department,
        Collectors.mapping(Employee::name, Collectors.toList())));
```

### partitioningBy

```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() >= 75_000));
// Always exactly 2 keys: true and false
```

### joining

```java
String csv    = employees.stream().map(Employee::name).collect(Collectors.joining(", "));
String pretty = employees.stream().map(Employee::name).collect(Collectors.joining(", ", "[", "]"));
```

### Common Collectors summary

| Collector | Produces | Purpose |
|---|---|---|
| `toList()` / `toSet()` | `List<T>` / `Set<T>` | Collect into a collection |
| `toMap(k, v)` | `Map<K,V>` | Build a map (add merge fn for dup keys) |
| `groupingBy(fn)` | `Map<K, List<T>>` | Group by key (SQL GROUP BY) |
| `groupingBy(fn, downstream)` | `Map<K, R>` | Group then reduce each group |
| `partitioningBy(pred)` | `Map<Boolean, List<T>>` | Split into true/false groups |
| `joining(", ")` | `String` | Concatenate strings |
| `counting()` | `Long` | Count elements (downstream) |
| `summingInt(fn)` | `Integer` | Sum of int property (downstream) |
| `averagingInt(fn)` | `Double` | Average of int property (downstream) |
| `mapping(fn, downstream)` | depends | Transform then collect (downstream) |

---

## Optional

`Optional<T>` is a container that may or may not hold a value — it makes "no value" **explicit** so you don't get surprised `NullPointerException`s.

**Think of it like a labeled gift box:** You're forced to check whether it's empty before reaching in.

### Creating and using Optionals

```java
Optional<String> present = Optional.of("hello");             // non-null (throws if null passed)
Optional<String> maybe   = Optional.ofNullable(getName());   // safe when value might be null
Optional<String> nothing = Optional.empty();

Optional<String> opt = Optional.ofNullable(findUser());

opt.isPresent();                                             // true if value exists
opt.ifPresent(name -> System.out.println("Found: " + name));
opt.ifPresentOrElse(name -> System.out.println(name), () -> System.out.println("Empty")); // Java 9+
opt.map(String::length);     // Optional<Integer> — transforms if present
opt.filter(n -> n.length() > 3); // stays Optional, becomes empty if predicate fails
opt.flatMap(name -> lookupEmail(name)); // for functions that already return Optional
```

### orElse vs orElseGet vs orElseThrow

```java
String name1 = opt.orElse("default");                        // ALWAYS evaluates "default"
String name2 = opt.orElseGet(() -> computeExpensiveDefault()); // LAZY — only runs when empty
String name3 = opt.orElseThrow(() -> new RuntimeException("Not found"));
```

> `orElse` is fine for cheap constants. Use `orElseGet` when the fallback is expensive — it only runs when the Optional is empty.

### The `.get()` anti-pattern

```java
// BAD — throws NoSuchElementException if empty
String name = opt.get();

// GOOD — handle absence explicitly
String safe = opt.orElse("unknown");
```

> Use `Optional` as a **return type** for methods that might not find a result. Do **not** use it for fields, parameters, or inside collections.

---

## Primitive Streams

`Stream<Integer>` boxes every int. **`IntStream`**, **`LongStream`**, and **`DoubleStream`** work directly with primitives — no boxing overhead — and add numeric methods like `sum()` and `average()`.

```java
IntStream.range(1, 5).forEach(System.out::println);        // 1,2,3,4 — exclusive end
IntStream.rangeClosed(1, 5).forEach(System.out::println);  // 1,2,3,4,5 — inclusive end

int total = IntStream.rangeClosed(1, 100).sum();            // 5050
OptionalDouble avg = IntStream.of(10, 20, 30).average();    // OptionalDouble[20.0]
IntSummaryStatistics stats = IntStream.of(3, 7, 2, 9, 5).summaryStatistics();
// count=5, sum=26, min=2, max=9, average=5.2
```

### Converting between object and primitive streams

```java
// Stream<T> → IntStream
int totalChars = words.stream().mapToInt(String::length).sum();

// IntStream → Stream<Integer> (needed to collect)
List<Integer> list = IntStream.rangeClosed(1, 5).boxed().collect(Collectors.toList());

// IntStream → Stream<String>
List<String> labels = IntStream.rangeClosed(1, 3).mapToObj(i -> "Item-" + i).collect(Collectors.toList());
```

> Prefer `mapToInt(...).sum()` over `map(Integer::intValue)` then summing — cleaner and avoids boxing. Use `boxed()` when you need to `collect()` a primitive stream.

---

## Parallel Streams

A parallel stream splits data across **multiple CPU cores** via the common ForkJoinPool.

```java
long count = hugeList.parallelStream()
    .filter(n -> isPrime(n))
    .count();
```

### When parallel streams HELP
- Large dataset (thousands+ of elements)
- CPU-bound, independent work per element
- Source splits well (`ArrayList`, arrays)

### When parallel streams HURT
- **Small datasets** — split/merge overhead exceeds any gain
- **Shared mutable state** — race conditions and corrupted data
- **Inside web-server request threads** — uses the shared JVM-wide ForkJoinPool, can starve other requests
- **I/O-bound work** — parallel streams are for CPU work, not blocking I/O

```java
// DANGEROUS — ArrayList is not thread-safe
List<Integer> results = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(results::add);  // data corruption!
// CORRECT: use collect() instead
```

> Default to sequential. Use parallel only for large, CPU-heavy, independent workloads — and measure before committing.

---

## Common Mistakes & Gotchas

```java
// 1. REUSING A STREAM
Stream<Integer> s = List.of(1, 2, 3).stream();
s.forEach(System.out::println);
s.count();   // IllegalStateException — FIX: create a new stream each time

// 2. FORGETTING THE TERMINAL OP — nothing runs without one
List.of(1, 2, 3).stream()
    .map(n -> { System.out.println("mapping"); return n; });   // prints NOTHING
// FIX: add .forEach(), .collect(), .count(), etc.

// 3. SIDE EFFECTS IN map/filter
List<Integer> bag = new ArrayList<>();
List.of(1, 2, 3).stream().map(n -> { bag.add(n); return n * 2; });  // BAD + no terminal op
// FIX: keep lambdas pure; use collect() to gather results

// 4. MODIFYING THE SOURCE WHILE STREAMING
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.stream().forEach(n -> { if (n == 2) list.remove(n); });  // ConcurrentModificationException
// FIX: collect what to remove first, or use removeIf()

// 5. PEEK MISUSE — for debugging only
List.of(1, 2, 3).stream().peek(n -> save(n)).count();  // BAD
// FIX: use forEach() for real side effects

// 6. toMap WITH DUPLICATE KEYS
Stream.of("apple", "avocado", "banana")
    .collect(Collectors.toMap(s -> s.charAt(0), s -> s));  // BOOM — 'a' appears twice
// FIX: toMap(key, val, (a, b) -> a)

// 7. Optional.get() without checking — throws NoSuchElementException
//    FIX: orElse / orElseGet / orElseThrow / ifPresent

// 8. orElse running expensive code needlessly — orElse(expensiveCall()) always runs
//    FIX: orElseGet(() -> expensiveCall())
```

---

## Common Interview Questions

### Q: What is a functional interface?

An interface with **exactly one abstract method** (SAM). `default` and `static` methods don't count. The compiler can match a lambda to it because there's only one method to implement. `@FunctionalInterface` enforces this at compile time. Examples: `Runnable`, `Comparator`, `Function`, `Predicate`.

---

### Q: What is the difference between `map` and `flatMap`?

`map` turns each element into **one** new element (same count). `flatMap` turns each element into a **stream of elements** then merges them all into one flat stream (count can change). Use `flatMap` when each element expands into multiple values or to flatten a `List<List<X>>` into a `List<X>`.

---

### Q: What is the difference between intermediate and terminal operations?

Intermediate ops (`filter`, `map`, `sorted`) return a **new Stream** and are **lazy** — nothing runs until a terminal op fires. Terminal ops (`collect`, `forEach`, `count`, `reduce`) produce a **result**, **trigger execution**, and consume the stream. A pipeline with no terminal op does nothing.

---

### Q: Why are streams lazy? What's the benefit?

Laziness lets Java fuse all steps so each element flows through the entire pipeline in one pass (no intermediate copies), and enables short-circuiting — `findFirst()` or `limit(3)` stop as soon as the answer is known. Unused pipelines cost nothing.

---

### Q: What is the difference between `reduce` and `collect`?

`reduce` folds elements into a **single immutable result** (sum, max). `collect` accumulates into a **mutable container** (List, Set, Map) via a `Collector`. Use `collect` for building collections; `reduce` for scalar values.

---

### Q: Can a stream be reused?

No. After a terminal operation, the stream is consumed. Calling another operation throws `IllegalStateException`. Create a fresh stream from the source each time.

---

### Q: What is the difference between a Collection and a Stream?

A Collection **stores** elements in memory, is reusable, and iterated externally. A Stream **stores nothing** — it's a lazy, single-use pipeline iterated internally. Collections are about **storing** data; streams are about **computing** over it.

---

### Q: What is the difference between `findFirst` and `findAny`?

`findFirst()` returns the **first** element in encounter order (deterministic). `findAny()` returns **any** match — in a parallel stream this is faster because it doesn't enforce order.

---

### Q: What is the difference between `Optional.orElse` and `Optional.orElseGet`?

`orElse(value)` **always evaluates** its argument even when a value is present. `orElseGet(supplier)` is lazy — it only calls the supplier **when empty**. Use `orElse` for cheap constants, `orElseGet` for expensive fallbacks.

---

### Q: When should you use parallel streams?

Rarely, and only after measuring. They help with **large, CPU-bound, independent** workloads over sources that split well. Avoid for small data, ordered results, shared mutable state, I/O-bound work, and inside web-server request threads (shared ForkJoinPool). Default to sequential.

---

### Q: What does `@FunctionalInterface` do, and is it required?

It's **optional** — lambdas work without it. It instructs the compiler to enforce "exactly one abstract method," so accidentally adding a second method causes a compile error. It documents intent and prevents breakage.

---

### Q: What is a method reference and what are the four kinds?

A method reference (`::`) is shorthand for a lambda that only calls one method. Four kinds: **static** (`Integer::parseInt`), **instance of a particular object** (`System.out::println`), **instance of an arbitrary object** (`String::toLowerCase`), **constructor** (`ArrayList::new`).

---

### Q: How does `groupingBy` differ from `partitioningBy`?

`groupingBy` creates **one key per distinct classifier value** (like SQL `GROUP BY`). `partitioningBy` always produces **exactly two keys — `true` and `false`** — even if one group is empty. Use `partitioningBy` for a strict yes/no split, `groupingBy` for arbitrary categories.

---

### Q: What happens with `toMap` when two elements produce the same key?

It throws `IllegalStateException: Duplicate key`. Fix with the three-argument form `toMap(keyFn, valueFn, mergeFn)` where the merge function handles collisions (e.g., `(a, b) -> a` to keep the first, or `Integer::sum` to add them).

---

## Quick Reference Cheat Sheet

```
LAMBDA SYNTAX
  ()        -> expr
  x         -> expr            // one param, parens optional
  (x, y)    -> expr
  (x, y)    -> { stmts; return v; }   // block body needs explicit return
  A lambda = instance of a functional interface (one abstract method)

METHOD REFERENCES (::)
  ClassName::staticMethod    -> x -> ClassName.staticMethod(x)
  instance::method           -> () -> instance.method()
  ClassName::instanceMethod  -> x -> x.instanceMethod()
  ClassName::new             -> () -> new ClassName()
```

```
STREAM PIPELINE
  source.stream()  ->  intermediate ops (LAZY)  ->  terminal op (RUNS IT)
  - Stream is NOT a data structure; single-use; ops are lazy.
  - No terminal op = nothing runs.

INTERMEDIATE OPS (lazy, return a Stream)
  filter(pred)     keep matching elements
  map(fn)          transform each element (1→1)
  flatMap(fn)      expand+flatten (1→many)
  distinct()       remove duplicates
  sorted()         sort (natural or Comparator)
  peek(fn)         debug-only window
  limit(n)/skip(n) take first n / drop first n
  mapToInt/boxed   to/from primitive streams

TERMINAL OPS (trigger execution, consume the stream)
  forEach(fn)                  side effect
  collect(collector)           gather into List/Set/Map/String
  toList()                     Java 16+ unmodifiable list
  reduce(id, accumulator)      fold into single value
  count()                      long
  min/max(comparator)          Optional
  anyMatch/allMatch/noneMatch  boolean (short-circuit)
  findFirst/findAny            Optional
```

```
TOP COLLECTORS
  toList() / toSet()                            into a collection
  toMap(k, v)                                   into a map (add merge fn for dup keys!)
  groupingBy(fn)                                Map<K, List<T>>  (SQL GROUP BY)
  groupingBy(fn, counting())                    count per group
  groupingBy(fn, summingInt(prop))              sum per group
  groupingBy(fn, mapping(g, toList()))          transform then collect per group
  partitioningBy(pred)                          Map<Boolean, List<T>> (always 2 keys)
  joining(", ", "[", "]")                       concatenate strings
```

```
OPTIONAL
  Create:    of(v) | ofNullable(v) | empty()
  Check:     isPresent() | isEmpty() | ifPresent(fn) | ifPresentOrElse(fn, runnable)
  Transform: map(fn) | filter(pred) | flatMap(fn)
  Unwrap:    orElse(v)        always evaluates default (cheap constants only)
             orElseGet(sup)   lazy — only if empty (use for expensive defaults)
             orElseThrow(sup) throw if empty
  AVOID:     .get() without checking (NoSuchElementException when empty)
```

---

*Last Updated: 2026-06-18*
