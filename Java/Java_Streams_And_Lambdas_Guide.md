# Java Streams & Lambdas Interview Questions & Study Guide

## Overview

Before Java 8 (released in 2014), Java was a strictly **object-oriented** language. If you wanted to pass behavior around (like "what to do with each item in a list"), you had to write a whole class or a clunky anonymous inner class. Looping over collections meant writing verbose `for` loops where you described **how** to do everything, step by step.

Java 8 introduced **lambdas** and the **Streams API** to fix this. Together they bring **functional programming** to Java:

- **Less boilerplate** — a 6-line anonymous inner class becomes a 1-line lambda.
- **Declarative code** — you describe **what** you want ("give me the active users sorted by name"), not **how** to compute it step by step. The library handles the looping, the temporary variables, and the iteration mechanics for you.
- **Composable pipelines** — you chain small operations together like `filter → map → collect`, and each step reads like a sentence.

This is one of the **most heavily tested topics** in Java backend interviews. Almost every Java interview asks about `map` vs `flatMap`, lazy evaluation, `reduce` vs `collect`, `Optional`, and when (not) to use parallel streams. This guide teaches all of it from the ground up.

> Everything here is accurate for **Java 8 through Java 21**. Features added after Java 8 (like `Stream.toList()` in Java 16) are clearly marked.

---

## Lambda Expressions

A **lambda expression** is a short block of code that you can pass around like a value. It's essentially a function without a name that you can store in a variable or hand to a method.

**Think of it like a small "to-go" function:** Imagine you're at a restaurant and you hand the waiter a tiny note that says "when my food is ready, add extra salt." You're not cooking it yourself — you're handing over a small instruction (a behavior) for someone else to run later. A lambda is exactly that: a packaged-up instruction you give to another method to execute when it's ready.

### Before lambdas (the old, painful way)

```java
// Sorting a list BEFORE Java 8 — using an anonymous inner class
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");

Collections.sort(names, new Comparator<String>() {   // a whole anonymous class...
    @Override
    public int compare(String a, String b) {          // ...just to wrap one line of logic
        return a.compareTo(b);
    }
});
// 6 lines, most of it ceremony (the class, the @Override, the method signature)
```

### After lambdas (the clean way)

```java
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");

Collections.sort(names, (a, b) -> a.compareTo(b));
// The lambda "(a, b) -> a.compareTo(b)" IS the Comparator. One line. Same result.
```

### Lambda syntax forms

The syntax is: `(parameters) -> { body }`. The `->` arrow separates the inputs from the work.

```java
// 1. No parameters — use empty parentheses
Runnable task = () -> System.out.println("Running!");
// () means "takes no arguments"; the body prints a message

// 2. One parameter — parentheses are OPTIONAL when there's exactly one
Consumer<String> printer = name -> System.out.println(name);
// could also write (name) -> ...  — both are valid

// 3. Multiple parameters — parentheses REQUIRED
BinaryOperator<Integer> add = (a, b) -> a + b;
// two inputs a and b, returns their sum

// 4. Multi-line body — use braces { } and an explicit 'return'
BiFunction<Integer, Integer, Integer> calc = (a, b) -> {
    int sum = a + b;          // you can have multiple statements inside braces
    int doubled = sum * 2;
    return doubled;           // when you use braces, you MUST write 'return' explicitly
};

// 5. Single-expression body — no braces, no 'return' (the value is returned automatically)
Function<Integer, Integer> square = n -> n * n;
// "n * n" is automatically returned — no need for braces or the return keyword
// (You can also write explicit parameter types like (Integer a, Integer b), but Java usually infers them.)
```

### What a lambda REALLY is (the key insight)

A lambda is **not a new kind of object** invented out of thin air. Under the hood, a lambda is simply **an instance of a functional interface** — an interface with exactly one abstract method. When you write `Runnable r = () -> System.out.println("hi")`, Java sees that `Runnable` has ONE abstract method (`run()`) and treats the lambda body as that method's implementation. It's exact shorthand for the anonymous-class version shown above.

> **Interview Tip**: A lambda can only be assigned to a **functional interface** type. When you write `Runnable r = () -> ...`, the compiler knows the lambda implements `Runnable.run()`. This is why "what is a functional interface" and "what is a lambda" are really the same question viewed from two angles.

---

## Functional Interfaces

A **functional interface** is an interface with **exactly one abstract method** (sometimes called a SAM — Single Abstract Method — interface). Because it has only one method, the compiler can match a lambda to it unambiguously.

You can mark it with the `@FunctionalInterface` annotation. This is optional, but it tells the compiler "enforce that this has exactly one abstract method" — if someone accidentally adds a second abstract method, the code won't compile.

```java
@FunctionalInterface                    // optional, but recommended — compiler enforces "exactly 1 abstract method"
interface Greeter {
    String greet(String name);          // the ONE abstract method
    // default methods and static methods DON'T count against the "one abstract method" rule
    default void hello() { System.out.println("Hello!"); }  // allowed — has a body
}

// Now you can implement Greeter with a lambda:
Greeter g = name -> "Hi, " + name + "!";
System.out.println(g.greet("Alice"));   // prints: Hi, Alice!
```

### The built-in functional interfaces (`java.util.function`)

Java ships with a set of ready-made functional interfaces so you almost never need to write your own. **Memorize this table — it comes up constantly in interviews.**

| Interface | Abstract Method | Takes | Returns | Plain-English Purpose |
|---|---|---|---|---|
| `Function<T,R>` | `R apply(T t)` | 1 input | 1 output | Transform a `T` into an `R` |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | 2 inputs | 1 output | Combine two inputs into a result |
| `Consumer<T>` | `void accept(T t)` | 1 input | nothing | Do something with a value (side effect) |
| `BiConsumer<T,U>` | `void accept(T t, U u)` | 2 inputs | nothing | Do something with two values |
| `Supplier<T>` | `T get()` | nothing | 1 output | Produce/supply a value on demand |
| `Predicate<T>` | `boolean test(T t)` | 1 input | `boolean` | Ask a yes/no question about a value |
| `BiPredicate<T,U>` | `boolean test(T t, U u)` | 2 inputs | `boolean` | Yes/no question about two values |
| `UnaryOperator<T>` | `T apply(T t)` | 1 input | same type | Transform a `T` into another `T` |
| `BinaryOperator<T>` | `T apply(T t, T t)` | 2 same-type inputs | same type | Combine two `T`s into one `T` |

> `UnaryOperator<T>` is just a `Function<T,T>` where input and output are the **same type**. `BinaryOperator<T>` is just a `BiFunction<T,T,T>`. They exist as convenient shorthands.

### A tiny example of each

```java
import java.util.function.*;

// Function<T,R> — transform input to output (here: String -> its length, an Integer)
Function<String, Integer> length = s -> s.length();
System.out.println(length.apply("hello"));        // 5

// Predicate<T> — returns true/false
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4));                // true

// Supplier<T> — supplies a value on demand, takes no input
Supplier<Double> random = () -> Math.random();
System.out.println(random.get());                  // e.g. 0.732...

// BinaryOperator<T> — two same-type inputs, same-type output (used heavily in reduce)
BinaryOperator<Integer> max = (a, b) -> a > b ? a : b;
System.out.println(max.apply(3, 9));               // 9
// The rest follow the same pattern: BiFunction (2 in, 1 out via apply), Consumer/BiConsumer
// (accept, no return), BiPredicate (test on 2 inputs), UnaryOperator (Function<T,T>).
```

> **Interview Tip**: `Predicate` has handy combiner methods: `.and()`, `.or()`, `.negate()`. `Function` has `.andThen()` (run this, then the next) and `.compose()` (run the other first). Example: `isEven.and(n -> n > 10)` is a predicate for "even AND greater than 10".

---

## Method References

A **method reference** is an even shorter shortcut for a lambda that does nothing but call one existing method. If your lambda is just `x -> someMethod(x)`, you can replace it with a method reference using the `::` operator.

**Think of it like a nickname:** Instead of saying "the person who delivers our mail every morning," you just say their name. A method reference is a direct name-drop of a method you already have, instead of wrapping it in a lambda.

### The 4 kinds of method references

```java
// 1. STATIC method reference:  ClassName::staticMethod
Function<String, Integer> parse = Integer::parseInt;
// equivalent lambda: s -> Integer.parseInt(s)
System.out.println(parse.apply("42"));             // 42

// 2. INSTANCE method of a PARTICULAR (specific) object:  instance::method
String greeting = "Hello World";
Supplier<String> upper = greeting::toUpperCase;
// equivalent lambda: () -> greeting.toUpperCase()  — always uses THIS specific 'greeting' object
System.out.println(upper.get());                   // HELLO WORLD

// 3. INSTANCE method of an ARBITRARY object of a type:  ClassName::instanceMethod
Function<String, String> toLower = String::toLowerCase;
// equivalent lambda: s -> s.toLowerCase()
// Here the FIRST argument BECOMES the object the method is called on.
// "s" is the arbitrary String, and we call s.toLowerCase()
System.out.println(toLower.apply("HELLO"));        // hello

// 4. CONSTRUCTOR reference:  ClassName::new
Supplier<ArrayList<String>> listMaker = ArrayList::new;
// equivalent lambda: () -> new ArrayList<>()
List<String> list = listMaker.get();               // creates a new empty ArrayList
```

### When does each map to a lambda?

| Method Reference | Equivalent Lambda | Kind |
|---|---|---|
| `Integer::parseInt` | `s -> Integer.parseInt(s)` | Static |
| `System.out::println` | `s -> System.out.println(s)` | Instance of a particular object (`System.out`) |
| `String::toLowerCase` | `s -> s.toLowerCase()` | Instance of an arbitrary object |
| `String::compareTo` | `(a, b) -> a.compareTo(b)` | Instance of an arbitrary object (first arg is the receiver) |
| `ArrayList::new` | `() -> new ArrayList<>()` | Constructor |

The trickiest one is **kind 3 vs kind 2**. The difference:

- **Kind 2 (particular object)**: the object is fixed and known now — `greeting::toUpperCase` always uses that one `greeting` string.
- **Kind 3 (arbitrary object)**: the object is whatever gets passed in at call time — `String::toLowerCase` calls `.toLowerCase()` on whichever string you pass to `apply()`.

In streams, method references make pipelines read like English: `names.stream().map(String::toUpperCase).forEach(System.out::println)` — kind 3 to uppercase each name, kind 2 to print each result.

> **Interview Tip**: Use a method reference only when it makes the code **clearer**. If a lambda needs to do anything beyond calling a single existing method (e.g., `x -> x.trim().toUpperCase()`), keep it as a lambda — you can't chain in a method reference.

---

## Streams — The Big Picture

A **Stream** is a sequence of elements that you process through a **pipeline** of operations. This is the single most important concept to get right.

### A Stream is NOT a data structure

This is the #1 misconception. A `List` **stores** data in memory. A `Stream` does **not store anything** — it's a *view* or a *pipeline* that pulls elements **from a source** (like a List, array, or file), pushes them through a series of operations, and produces a result. Think of the Stream as the conveyor belt, and the List as the warehouse the belt pulls boxes from.

**Think of it like a factory assembly line / conveyor belt:**

```
   SOURCE                INTERMEDIATE OPS                    TERMINAL OP
(the warehouse)        (stations on the belt)            (the loading dock)

  List of               filter out                          collect into
  raw items   ──────►   bad items    ──────►   transform   ──────►   a final box
                        (lazy)                  each item   (triggers the whole line)
```

Boxes come from the **warehouse** (the *source*), pass through **stations** like `filter` and `map` (the **intermediate operations** — lazy, the belt isn't moving yet), and a worker finally **packs them into a crate** with `collect` (the **terminal operation**) — only then does the belt actually run.

### The 3 parts of every stream pipeline

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Dave");

long count = names.stream()                  // 1. SOURCE      — create stream from the list
                  .filter(n -> n.length() > 3)   // 2. INTERMEDIATE — keep names longer than 3 chars (LAZY)
                  .map(String::toUpperCase)      // 2. INTERMEDIATE — uppercase each one (LAZY)
                  .count();                      // 3. TERMINAL    — count them (triggers execution NOW)
// count = 3  (Alice, Charlie, Dave — "Bob" is filtered out)
```

1. **Source** — where elements come from: `list.stream()`, `Arrays.stream(arr)`, `Stream.of(1,2,3)`, `IntStream.range(...)`.
2. **Intermediate operations** — transform/filter the stream and **return a new Stream**, so you can chain them. They are **lazy** — calling them does nothing until a terminal op runs.
3. **Terminal operation** — produces a result (a value, a collection) or a side effect, and **triggers the whole pipeline**. After a terminal op runs, the stream is **consumed** and cannot be reused.

### Laziness — the most important property

Intermediate operations are **lazy**: they don't process anything until a terminal operation kicks off the pipeline. This enables huge optimizations — Java can fuse all the steps and even short-circuit.

```java
Stream<String> pipeline = names.stream()
    .filter(n -> {
        System.out.println("filtering: " + n);    // this print proves WHEN code runs
        return n.length() > 3;
    });
// NOTHING is printed yet! No terminal operation = the filter never runs.

System.out.println("--- before terminal op ---");
long c = pipeline.count();    // NOW the filter actually executes for each element
```

```
--- before terminal op ---       ← printed FIRST, proving filter hadn't run yet
filtering: Alice
filtering: Bob
filtering: Charlie
filtering: Dave
```

> **Why laziness matters**: With `findFirst()`, Java can stop the moment it finds a match instead of processing the whole list. With fused operations, each element flows through the entire pipeline once, rather than the list being copied at every step.

### A stream cannot be reused

Once a terminal operation runs on a stream, that stream is **consumed**. Touching it again throws `IllegalStateException`.

```java
Stream<String> s = names.stream();
s.forEach(System.out::println);    // terminal op — stream is now consumed
s.count();                         // BOOM! IllegalStateException: stream has already been operated upon or closed
// FIX: create a fresh stream each time you need one: names.stream()...
```

### Collection vs Stream — key differences

| Aspect | Collection (List, Set...) | Stream |
|---|---|---|
| Stores data? | **Yes** — holds elements in memory | **No** — pulls from a source on demand |
| Reusable? | Yes — iterate as many times as you want | **No** — single-use, consumed after terminal op |
| Eager or lazy? | Eager — data is there now | Lazy — nothing runs until terminal op |
| Iteration | External (you write the `for` loop) | Internal (the library loops for you) |
| Focus | **How** data is stored | **What** computation to perform |

---

## Intermediate Operations

Intermediate operations transform a stream and return a **new stream**, so they can be chained. Remember: they're all **lazy** — nothing happens until a terminal op runs.

```java
List<Integer> nums = Arrays.asList(5, 3, 8, 1, 9, 3, 7, 2);

// filter — keep only elements that pass a Predicate (return true)
nums.stream()
    .filter(n -> n > 4)                 // keep only numbers greater than 4 → [5, 8, 9, 7]
    .forEach(System.out::println);

// map — transform each element using a Function (1 input → 1 output)
nums.stream()
    .map(n -> n * 10)                   // multiply each by 10 → [50, 30, 80, 10, ...]
    .forEach(System.out::println);

// distinct — remove duplicates (uses equals())
nums.stream()
    .distinct()                         // [5, 3, 8, 1, 9, 7, 2] — the second 3 is dropped
    .forEach(System.out::println);

// sorted — sort elements (natural order, or pass a Comparator)
nums.stream()
    .sorted()                           // ascending → [1, 2, 3, 3, 5, 7, 8, 9]
    .forEach(System.out::println);

nums.stream()
    .sorted(Comparator.reverseOrder())  // descending → [9, 8, 7, 5, 3, 3, 2, 1]
    .forEach(System.out::println);

// limit — keep only the first N elements (short-circuiting)
nums.stream()
    .limit(3)                           // first 3 → [5, 3, 8]
    .forEach(System.out::println);

// skip — discard the first N elements, keep the rest
nums.stream()
    .skip(3)                            // drop first 3 → [1, 9, 3, 7, 2]
    .forEach(System.out::println);

// peek — look at each element as it flows by, WITHOUT changing it (for debugging)
nums.stream()
    .peek(n -> System.out.println("saw: " + n))   // a "window" into the pipeline
    .map(n -> n * 2)
    .forEach(System.out::println);

// mapToInt / boxed — convert to a primitive IntStream and back (see Primitive Streams below)
```

### flatMap — flattening a list of lists

`flatMap` is the operation people struggle with most, so let's be very clear.

- `map` turns each element into **exactly one** new element.
- `flatMap` turns each element into **a whole stream of elements**, then **flattens** all those mini-streams into **one single stream**.

**Think of it like emptying bags of marbles into one big bowl:** You have several small bags, each holding a few marbles (a list of lists). `flatMap` opens every bag and pours all the marbles into one bowl, so you end up with a single flat collection of marbles — no more bags.

```java
// We have a list of lists (a "bag of bags")
List<List<Integer>> listOfLists = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5),
    Arrays.asList(6, 7, 8, 9)
);

// WRONG approach with map — you'd get a Stream<Stream<Integer>>, still nested:
// listOfLists.stream().map(list -> list.stream())  → NOT flattened, useless

// flatMap — open each inner list into a stream, then merge them all into ONE flat stream
List<Integer> flat = listOfLists.stream()
    .flatMap(innerList -> innerList.stream())   // each inner list becomes a stream; all are merged
    .collect(Collectors.toList());
// flat = [1, 2, 3, 4, 5, 6, 7, 8, 9]  — one flat list!
```

The same idiom flattens a list of sentences into words: `sentences.stream().flatMap(s -> Arrays.stream(s.split(" "))).distinct().collect(toList())` — each sentence expands into a stream of words, all merged into one flat stream.

| Operation | Input → Output mapping | Result shape |
|---|---|---|
| `map` | 1 element → 1 element | `Stream<R>` (same number of elements) |
| `flatMap` | 1 element → 0..many elements (a stream) | `Stream<R>` (flattened, count can change) |

> **Interview Tip**: The classic test is "you have a `List<List<X>>` and want a `List<X>`" or "a list of objects each with a list field, and you want all the inner items combined." Both are textbook `flatMap` cases.

---

## Terminal Operations

A **terminal operation** ends the pipeline, triggers all the lazy intermediate operations to run, and produces a result (or a side effect). After it runs, the stream is consumed.

```java
List<Integer> nums = Arrays.asList(5, 3, 8, 1, 9);

// forEach — perform an action on each element (side effect, returns nothing)
nums.stream().forEach(n -> System.out.println(n));

// collect — gather elements into a collection (the workhorse — see Collectors section)
List<Integer> result = nums.stream().filter(n -> n > 4).collect(Collectors.toList());

// count — how many elements (returns long)
long howMany = nums.stream().filter(n -> n > 4).count();   // 3

// min / max — smallest/largest by a Comparator, returns an Optional (stream could be empty)
Optional<Integer> biggest = nums.stream().max(Comparator.naturalOrder());  // Optional[9]
Optional<Integer> smallest = nums.stream().min(Comparator.naturalOrder()); // Optional[1]

// anyMatch / allMatch / noneMatch — boolean checks (short-circuit when answer is decided)
boolean anyBig  = nums.stream().anyMatch(n -> n > 8);   // true  (9 exists)
boolean allBig  = nums.stream().allMatch(n -> n > 0);   // true  (all positive)
boolean noneBig = nums.stream().noneMatch(n -> n > 100);// true  (nothing over 100)

// findFirst / findAny — get one element wrapped in Optional
Optional<Integer> first = nums.stream().filter(n -> n > 4).findFirst();  // Optional[5]
Optional<Integer> any   = nums.stream().filter(n -> n > 4).findAny();    // Optional[5] (any match)

// toArray — dump the stream into an array
Integer[] arr = nums.stream().toArray(Integer[]::new);

// toList — Java 16+ shortcut for an UNMODIFIABLE list (cleaner than collect(toList()))
List<Integer> list = nums.stream().filter(n -> n > 4).toList();   // [5, 8, 9]
```

### reduce — combining everything into one value

`reduce` repeatedly combines elements into a **single result**. This is how `sum`, `product`, `max`, and string concatenation all work under the hood. It's worth understanding carefully because it's a frequent interview question.

The most explicit form takes two parts:
- **identity** — the starting value (and the result if the stream is empty).
- **accumulator** — a `BinaryOperator` that combines the running total with the next element.

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);

// Sum using reduce:
int sum = nums.stream()
    .reduce(0, (runningTotal, next) -> runningTotal + next);
// identity = 0 (start value), then: 0+1=1, 1+2=3, 3+3=6, 6+4=10 → result = 10

// Product:
int product = nums.stream().reduce(1, (a, b) -> a * b);
// identity = 1 (NOT 0 — multiplying by 0 always gives 0!) → 1*1*2*3*4 = 24

// reduce WITHOUT identity returns Optional (because an empty stream has no result):
Optional<Integer> maybeSum = nums.stream().reduce((a, b) -> a + b);  // Optional[10]
Optional<Integer> emptyResult = Stream.<Integer>of().reduce((a, b) -> a + b); // Optional.empty
```

> **Interview Tip — choosing the identity**: The identity must be a value that "doesn't change the result" when combined. For addition that's `0`, for multiplication it's `1`, for string concatenation it's `""`. Picking the wrong identity is a classic bug.

> **reduce vs collect**: Use `reduce` to fold elements into a **single immutable value** (a sum, a max, a concatenated string). Use `collect` to accumulate elements into a **mutable container** (a List, Map, etc.). Using `reduce` to build a list is an anti-pattern — it's inefficient and breaks parallelism.

### findFirst vs findAny

- `findFirst()` — returns the **first** element in encounter order. Deterministic.
- `findAny()` — returns **any** matching element. In a sequential stream it's usually the first too, but in a **parallel** stream it can return whichever element a thread finds first, which is faster because it doesn't have to respect order.

---

## Collectors

`collect()` is the most powerful terminal operation, and `Collectors` is a utility class full of ready-made "recipes" for accumulating stream elements. **This is one of the most heavily asked stream topics in interviews.**

Let's use a small `Employee` model for realistic examples:

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

### toList / toSet — gather into a collection

```java
// toList — collect into a List (allows duplicates, keeps order)
List<String> names = employees.stream()
    .map(Employee::name)
    .collect(Collectors.toList());          // [Alice, Bob, Charlie, Dave, Eve]

// toSet — collect into a Set (removes duplicates, no guaranteed order)
Set<String> departments = employees.stream()
    .map(Employee::department)
    .collect(Collectors.toSet());           // [Engineering, Sales, HR]
```

### toMap — build a map (and the duplicate-key gotcha)

```java
// toMap(keyMapper, valueMapper) — turn each element into a key→value pair
Map<String, Integer> nameToSalary = employees.stream()
    .collect(Collectors.toMap(
        Employee::name,        // key   = employee's name
        Employee::salary));    // value = employee's salary
// {Alice=90000, Bob=80000, Charlie=70000, Dave=60000, Eve=75000}

// GOTCHA: if two elements produce the SAME key, toMap throws IllegalStateException!
Map<String, Integer> deptToSalary = employees.stream()
    .collect(Collectors.toMap(
        Employee::department,  // "Engineering" appears TWICE (Alice & Bob)
        Employee::salary));    // BOOM! IllegalStateException: Duplicate key Engineering

// FIX: provide a merge function — what to do when keys collide
Map<String, Integer> deptToTotalSalary = employees.stream()
    .collect(Collectors.toMap(
        Employee::department,
        Employee::salary,
        (existing, incoming) -> existing + incoming));  // on collision, ADD the salaries
// {Engineering=170000, Sales=130000, HR=75000}
```

### groupingBy — the most-asked Collector

`groupingBy` is the SQL `GROUP BY` of streams: it splits elements into groups keyed by some classifier function.

```java
// Group employees by department → Map<Department, List<Employee>>
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));
// {
//   Engineering=[Alice, Bob],
//   Sales=[Charlie, Dave],
//   HR=[Eve]
// }
```

#### groupingBy with a downstream collector

The real power: pass a **second collector** that processes each group further.

```java
// Count employees per department  (downstream = counting())
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.counting()));            // {Engineering=2, Sales=2, HR=1}

// Total salary per department  (downstream = summingInt())
Map<String, Integer> salaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.summingInt(Employee::salary)));   // {Engineering=170000, Sales=130000, HR=75000}

// Average salary per department  (downstream = averagingInt())
Map<String, Double> avgByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.averagingInt(Employee::salary))); // {Engineering=85000.0, Sales=65000.0, HR=75000.0}

// Just the NAMES per department  (downstream = mapping(): transform, then collect)
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.mapping(Employee::name, Collectors.toList())));
// {Engineering=[Alice, Bob], Sales=[Charlie, Dave], HR=[Eve]}
```

### partitioningBy — split into exactly two groups (true / false)

```java
// Partition into high earners (>= 75k) vs the rest → Map<Boolean, List<Employee>>
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() >= 75_000));
// {
//   false=[Charlie(70k), Dave(60k)],          ← the < 75k group
//   true=[Alice(90k), Bob(80k), Eve(75k)]     ← the >= 75k group
// }
// Difference from groupingBy: partitioningBy ALWAYS produces exactly 2 keys (true & false),
// even if one group is empty. groupingBy creates a key per distinct value.
```

### joining — concatenate strings

```java
// joining() — glue all strings together
String all = employees.stream().map(Employee::name).collect(Collectors.joining());
// "AliceBobCharlieDaveEve"

// joining(delimiter)
String csv = employees.stream().map(Employee::name).collect(Collectors.joining(", "));
// "Alice, Bob, Charlie, Dave, Eve"

// joining(delimiter, prefix, suffix)
String pretty = employees.stream().map(Employee::name)
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie, Dave, Eve]"
```

### Common Collectors summary

| Collector | Produces | Purpose |
|---|---|---|
| `toList()` | `List<T>` | Collect into a list (duplicates allowed) |
| `toSet()` | `Set<T>` | Collect into a set (dedup) |
| `toMap(k, v)` | `Map<K,V>` | Build a map (needs merge fn for duplicate keys) |
| `groupingBy(fn)` | `Map<K, List<T>>` | Group elements by a key |
| `groupingBy(fn, downstream)` | `Map<K, R>` | Group, then reduce each group |
| `partitioningBy(pred)` | `Map<Boolean, List<T>>` | Split into true/false groups |
| `joining(", ")` | `String` | Concatenate strings with a separator |
| `counting()` | `Long` | Count elements (used as downstream) |
| `summingInt(fn)` | `Integer` | Sum of an int property |
| `averagingInt(fn)` | `Double` | Average of an int property |
| `mapping(fn, downstream)` | depends | Transform, then collect (used as downstream) |

---

## Optional

`Optional<T>` is a container that may or may not hold a value. It was added in Java 8 to make the possibility of "no value" **explicit in the type system**, so you stop accidentally hitting `NullPointerException`.

**Think of it like a gift box:** When someone hands you an `Optional`, it's a box that's either clearly **labeled "has a gift inside"** or clearly **labeled "empty."** You're forced to check the label before reaching in, so you never blindly grab into an empty box and hurt yourself (the NPE). A raw `null` is like an unlabeled box you reach into hopefully — and sometimes get bitten.

### Creating Optionals

```java
Optional<String> present  = Optional.of("hello");        // value is definitely non-null (throws if you pass null!)
Optional<String> maybe    = Optional.ofNullable(getName());// safe when the value MIGHT be null
Optional<String> nothing  = Optional.empty();            // explicitly empty box
```

### Common methods

```java
Optional<String> opt = Optional.ofNullable(findUser());

// isPresent() / isEmpty() — check the label
if (opt.isPresent()) { ... }          // true if there's a value
if (opt.isEmpty())   { ... }          // Java 11+: true if empty

// ifPresent — run code ONLY if a value exists (no manual if needed)
opt.ifPresent(name -> System.out.println("Found: " + name));

// ifPresentOrElse — Java 9+: do one thing if present, another if empty
opt.ifPresentOrElse(
    name -> System.out.println("Found: " + name),   // runs if present
    ()   -> System.out.println("Nobody here"));      // runs if empty

// map — transform the value IF present (stays inside the Optional)
Optional<Integer> len = opt.map(String::length);     // Optional[5] or Optional.empty

// filter — keep the value only if it passes the predicate, else become empty
Optional<String> longName = opt.filter(n -> n.length() > 3);

// flatMap — like map, but for functions that ALREADY return an Optional (avoids Optional<Optional<X>>)
Optional<String> result = opt.flatMap(name -> lookupEmail(name)); // lookupEmail returns Optional<String>
```

### orElse vs orElseGet vs orElseThrow — a classic interview question

All three provide a fallback when the Optional is empty, but they behave differently:

```java
String name1 = opt.orElse("default");
// ALWAYS evaluates "default", even when opt HAS a value. Fine for cheap constants.

String name2 = opt.orElseGet(() -> computeExpensiveDefault());
// Calls the supplier ONLY when opt is empty. Use this when the fallback is expensive to build.

String name3 = opt.orElseThrow(() -> new UserNotFoundException("No user"));
// Throws your exception if empty. orElseThrow() with no args throws NoSuchElementException.
```

> **Interview Tip**: `orElse` eagerly evaluates its argument; `orElseGet` is lazy. If the default is a constant, `orElse` is fine. If the default is expensive (DB call, object creation), use `orElseGet` to avoid wasted work.

### The `.get()` anti-pattern

```java
// ANTI-PATTERN — calling get() without checking defeats the whole purpose of Optional:
String name = opt.get();   // throws NoSuchElementException if empty — same problem as a raw null!

// DO THIS INSTEAD — provide a fallback or handle absence explicitly:
String safe = opt.orElse("unknown");
opt.ifPresent(System.out::println);
```

> **Interview Tip**: Use `Optional` as a **return type** for methods that might not find a result (`findById`). Do **not** use it for fields, method parameters, or in collections — that's considered misuse.

---

## Primitive Streams

Regular streams like `Stream<Integer>` store **boxed** objects. Every time you do math, Java has to **unbox** (`Integer` → `int`) and **re-box** the result — this is **autoboxing overhead**, which costs memory and time in tight loops.

To avoid this, Java provides specialized primitive streams: **`IntStream`**, **`LongStream`**, and **`DoubleStream`**. They work directly with primitives — no boxing — and add convenient numeric methods like `sum()` and `average()`.

```java
// range / rangeClosed — generate a stream of numbers (great for replacing for-loops)
IntStream.range(1, 5)         // 1, 2, 3, 4        — upper bound EXCLUSIVE
         .forEach(System.out::println);

IntStream.rangeClosed(1, 5)   // 1, 2, 3, 4, 5     — upper bound INCLUSIVE
         .forEach(System.out::println);

// sum — built-in on primitive streams (Stream<Integer> doesn't have this directly)
int total = IntStream.rangeClosed(1, 100).sum();        // 5050

// average — returns OptionalDouble (the stream might be empty)
OptionalDouble avg = IntStream.of(10, 20, 30).average(); // OptionalDouble[20.0]

// summaryStatistics — get count, sum, min, max, average all at once
IntSummaryStatistics stats = IntStream.of(3, 7, 2, 9, 5).summaryStatistics();
// stats.getCount()=5, getSum()=26, getMin()=2, getMax()=9, getAverage()=5.2
```

### Converting between object and primitive streams

```java
// mapToInt — turn a Stream<T> into an IntStream (extract an int from each object)
List<String> words = List.of("apple", "fig", "banana");
int totalChars = words.stream()
    .mapToInt(String::length)    // Stream<String> → IntStream of lengths [5, 3, 6]
    .sum();                      // 14

// boxed — turn an IntStream back into a Stream<Integer> (so you can collect it)
List<Integer> list = IntStream.rangeClosed(1, 5)
    .boxed()                     // IntStream → Stream<Integer>
    .collect(Collectors.toList());   // [1, 2, 3, 4, 5]  (toList() can't work on a raw IntStream)

// mapToObj — turn each primitive into an object
List<String> labels = IntStream.rangeClosed(1, 3)
    .mapToObj(i -> "Item-" + i)  // IntStream → Stream<String>
    .collect(Collectors.toList());   // [Item-1, Item-2, Item-3]
```

> **Interview Tip**: If you find yourself doing `.map(Integer::intValue)` then summing, switch to `mapToInt(...).sum()` — it's clearer and avoids autoboxing. Use `boxed()` when you need to `collect()` a primitive stream back into a `List<Integer>`.

---

## Parallel Streams

A **parallel stream** splits the data into chunks and processes them on **multiple CPU cores at once** using the common ForkJoinPool. You opt in with `.parallelStream()` (on a collection) or `.parallel()` (on an existing stream).

```java
long count = hugeList.parallelStream()      // split work across CPU cores
    .filter(n -> isPrime(n))                 // each core checks a chunk of numbers
    .count();
```

**Think of it like a kitchen with several cooks:** One cook (sequential) prepares 100 dishes one after another. Multiple cooks (parallel) each take a share and cook simultaneously — much faster for a big banquet. But coordinating cooks has overhead, and if there's only one dish, hiring five cooks just wastes time and creates confusion.

### When parallel streams HELP

- The dataset is **large** (thousands+ of elements).
- The work per element is **CPU-bound and independent** (heavy computation, no shared state).
- The source **splits well** (`ArrayList`, arrays, `IntStream.range` split evenly; `LinkedList` does not).
- The operations are **stateless** and don't depend on encounter order.

### When parallel streams HURT (be honest and cautious)

- **Small datasets** — the cost of splitting, scheduling, and merging exceeds any gain. A sequential stream is faster.
- **Ordering matters** — preserving order (`forEachOrdered`, `findFirst`) forces coordination that erodes the benefit.
- **Shared mutable state** — if your lambda writes to a shared variable or non-thread-safe collection, you get **race conditions and corrupted/lost data**. (This is the most dangerous trap.)
- **Inside a web request thread** — parallel streams use the **shared common ForkJoinPool** across the whole JVM. One slow parallel stream can starve every other request's parallel work. In server applications this is usually a **bad idea** — prefer a dedicated thread pool / `ExecutorService` or `CompletableFuture` instead.
- **I/O-bound work** (DB calls, HTTP) — parallel streams are tuned for CPU work, not for blocking I/O. They tie up ForkJoinPool threads while they wait.

```java
// DANGEROUS — shared mutable state in a parallel stream = race condition!
List<Integer> results = new ArrayList<>();          // ArrayList is NOT thread-safe
IntStream.range(0, 1000).parallel()
    .forEach(results::add);   // multiple threads call add() at once → lost data or exception!
// CORRECT: collect() instead — it handles thread-safe accumulation for you.
```

> **Interview Tip**: The honest answer to "should I use parallel streams?" is **"rarely, and only after measuring."** Default to sequential. Reach for parallel only for large, CPU-heavy, independent workloads — and never inside a request-handling thread of a web server without understanding the shared ForkJoinPool implications.

---

## Common Mistakes & Gotchas

```java
// 1. REUSING A STREAM — a stream is single-use
Stream<Integer> s = List.of(1, 2, 3).stream();
s.forEach(System.out::println);
s.count();   // IllegalStateException: stream has already been operated upon or closed
// FIX: create a new stream from the source each time.

// 2. SIDE EFFECTS IN LAMBDAS — don't mutate external state inside map/filter
List<Integer> collected = new ArrayList<>();
List.of(1, 2, 3).stream()
    .map(n -> { collected.add(n); return n * 2; }); // side effect hidden in map — BAD
// Also note: this map() never runs at all — there's NO terminal operation! (see #4)
// FIX: keep lambdas pure; use collect() to gather results.

// 3. MODIFYING THE SOURCE WHILE STREAMING — ConcurrentModificationException
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.stream().forEach(n -> { if (n == 2) list.remove(n); }); // modifying 'list' during streaming — BAD
// FIX: collect what to remove first, or use removeIf().

// 4. FORGETTING STREAMS ARE LAZY — no terminal op means NOTHING runs
List.of(1, 2, 3).stream()
    .map(n -> { System.out.println("mapping " + n); return n; });   // prints NOTHING
// There's no terminal operation, so the map never executes.
// FIX: add a terminal op like .forEach(), .collect(), .count().

// 5. PEEK MISUSE — peek is for debugging only, NOT for doing real work
List.of(1, 2, 3).stream()
    .peek(n -> save(n))    // BAD: peek may be skipped/optimized away; not guaranteed to run on every element
    .count();
// FIX: use forEach() for actual side effects; reserve peek() for logging during debugging.

// 6. toMap WITH DUPLICATE KEYS — throws IllegalStateException
Stream.of("apple", "avocado", "banana")
    .collect(Collectors.toMap(s -> s.charAt(0), s -> s)); // two words start with 'a' → BOOM
// FIX: supply a merge function: toMap(key, val, (a, b) -> a)

// 7. Optional.get() WITHOUT CHECKING — throws NoSuchElementException when empty.
//    FIX: orElse / orElseGet / orElseThrow / ifPresent.
// 8. orElse RUNNING EXPENSIVE CODE NEEDLESSLY — orElse(expensiveCall()) runs even when
//    a value is present. FIX: orElseGet(() -> expensiveCall()) runs only when empty.
```

---

## Common Interview Questions

### Q: What is a functional interface?

An interface with **exactly one abstract method** (a SAM — Single Abstract Method — interface). It can have any number of `default` and `static` methods (those have bodies, so they don't count). Because there's only one abstract method, the compiler can match a lambda or method reference to it. The `@FunctionalInterface` annotation enforces this at compile time. Examples: `Runnable`, `Comparator`, `Function`, `Predicate`.

---

### Q: What is the difference between `map` and `flatMap`?

- `map` transforms each element into **exactly one** new element → `Stream<R>` with the same number of elements.
- `flatMap` transforms each element into a **stream of elements**, then **flattens** all those streams into one. Use it when each element expands into multiple values, or to flatten a `List<List<X>>` into a `List<X>`.

---

### Q: What is the difference between intermediate and terminal operations?

- **Intermediate** ops (`filter`, `map`, `sorted`, ...) return a **new Stream** and are **lazy** — they don't run until a terminal op triggers the pipeline. They can be chained.
- **Terminal** ops (`collect`, `forEach`, `count`, `reduce`, ...) produce a **result or side effect** and **trigger execution**. After a terminal op, the stream is consumed and can't be reused. A pipeline with no terminal operation does nothing.

---

### Q: Why are streams lazy? What's the benefit?

Laziness lets Java optimize the pipeline. Operations are **fused** so each element flows through all steps in one pass (no intermediate copies), and operations can **short-circuit** — `findFirst()` or `limit(3)` stop processing as soon as the answer is known, instead of computing the whole stream. Nothing runs until a terminal operation, so unused pipelines cost nothing.

---

### Q: What is the difference between `reduce` and `collect`?

`reduce` folds elements into a **single immutable result** (sum, max, concatenation). `collect` accumulates into a **mutable container** (List, Set, Map, String) via a `Collector`. Use `collect` to build collections; use `reduce` to compute a single scalar value.

---

### Q: Can a stream be reused?

No. A stream is **single-use**. Once a terminal operation runs, the stream is consumed; calling another operation on it throws `IllegalStateException`. To process the data again, create a fresh stream from the source.

---

### Q: What is the difference between a Collection and a Stream?

A **Collection** stores elements in memory, is reusable, and you iterate it externally. A **Stream** stores nothing — it's a lazy, single-use pipeline that iterates internally. Collections are about **storing** data; streams are about **computing** over it.

---

### Q: What is the difference between `findFirst` and `findAny`?

`findFirst()` returns the **first** element in encounter order (deterministic). `findAny()` returns **any** matching element — usually the first in a sequential stream, but in a **parallel** stream whatever a thread finds first, which is faster.

---

### Q: What is the difference between `Optional.orElse` and `Optional.orElseGet`?

`orElse(value)` **always evaluates** its argument, even when a value is present — wasteful if the default is expensive. `orElseGet(supplier)` is **lazy**, calling the supplier **only when empty**. Use `orElse` for cheap constants, `orElseGet` for expensive fallbacks.

---

### Q: When should you use parallel streams?

Rarely, and only after measuring. They help only with **large, CPU-bound, independent** workloads over a source that splits well (`ArrayList`, arrays). Avoid them for small data, ordered results, shared mutable state, I/O-bound work, and especially inside web-server request threads (shared JVM-wide ForkJoinPool can starve other work). Default to sequential.

---

### Q: What does `@FunctionalInterface` do, and is it required?

It's an **optional** annotation that tells the compiler to enforce the "exactly one abstract method" rule — if you accidentally add a second abstract method, compilation fails. Lambdas work on any interface with one abstract method even without the annotation, but adding it documents intent and prevents accidental breakage.

---

### Q: What is a method reference and what are the four kinds?

A method reference (`::`) is shorthand for a lambda that just calls one existing method. The four kinds are: **static** (`Integer::parseInt`), **instance of a particular object** (`System.out::println`), **instance of an arbitrary object** (`String::toLowerCase`), and **constructor** (`ArrayList::new`).

---

### Q: How does `groupingBy` differ from `partitioningBy`?

`groupingBy(classifier)` creates a `Map` with **one key per distinct classifier value** (like SQL `GROUP BY`). `partitioningBy(predicate)` always creates a map with **exactly two keys — `true` and `false`** — even if one group is empty. Use `partitioningBy` for a strict yes/no split, `groupingBy` for arbitrary categories.

---

### Q: What happens with `toMap` when two elements produce the same key?

It throws `IllegalStateException: Duplicate key`. To handle collisions, use the three-argument `toMap(keyFn, valueFn, mergeFn)` where the merge function decides what to do when keys clash (e.g., `(existing, incoming) -> existing` to keep the first, or `Integer::sum` to add them).

---

## Quick Reference Cheat Sheet

```
LAMBDA SYNTAX
  ()        -> expr          // no params
  x         -> expr          // one param (parens optional)
  (x, y)    -> expr          // multiple params
  (x, y)    -> { stmts; return v; }   // block body needs explicit return
  A lambda = an instance of a functional interface (one abstract method)

METHOD REFERENCES (::)
  ClassName::staticMethod    -> x -> ClassName.staticMethod(x)   // static
  instance::method           -> () -> instance.method()          // particular object
  ClassName::instanceMethod  -> x -> x.instanceMethod()          // arbitrary object
  ClassName::new             -> () -> new ClassName()            // constructor
```

```
STREAM PIPELINE
  source.stream()  ->  intermediate ops (LAZY)  ->  terminal op (RUNS IT)
  - Stream is NOT a data structure; it's single-use; ops are lazy.
  - No terminal op = nothing runs.

MOST-USED INTERMEDIATE OPS (lazy, return a Stream)
  filter(pred)     keep matching elements
  map(fn)          transform each element (1->1)
  flatMap(fn)      expand+flatten (1->many, e.g. List<List<X>> -> List<X>)
  distinct()       remove duplicates
  sorted()         sort (natural or Comparator)
  peek(fn)         debug-only window (do NOT use for real work)
  limit(n)/skip(n) take first n / drop first n
  mapToInt/boxed   to/from primitive streams

MOST-USED TERMINAL OPS (trigger execution, consume the stream)
  forEach(fn)               side effect on each
  collect(collector)        gather into List/Set/Map/String
  toList()                  Java 16+ unmodifiable list
  reduce(id, accumulator)   fold into a single value
  count()                   how many (long)
  min/max(comparator)       -> Optional
  anyMatch/allMatch/noneMatch  -> boolean (short-circuit)
  findFirst/findAny         -> Optional
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
  Create:  of(v) | ofNullable(v) | empty()
  Check:   isPresent() | isEmpty() | ifPresent(fn) | ifPresentOrElse(fn, runnable)
  Transform: map(fn) | filter(pred) | flatMap(fn)
  Unwrap:  orElse(v)        always evaluates the default (cheap constants)
           orElseGet(sup)   lazy — only if empty (expensive defaults)
           orElseThrow(sup) throw if empty
  AVOID:   .get() without checking (can throw NoSuchElementException)
```

---

*Last Updated: 2026-06-06*
