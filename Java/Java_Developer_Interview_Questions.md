# Java Developer Interview Questions & Answers

## Overview

45 questions covering Core Java, OOP, Collections, Exceptions, Threads, Strings, Java 8+, JVM internals, Design Patterns, coding problems, and best practices — scoped to what a junior developer needs to know.

---

## Table of Contents

1. [Core Java Fundamentals](#core-java-fundamentals) — Q1–Q5
2. [Object-Oriented Programming](#object-oriented-programming) — Q6–Q10
3. [Collections Framework](#collections-framework) — Q11–Q15
4. [Exception Handling](#exception-handling) — Q16–Q20
5. [Multithreading & Concurrency](#multithreading--concurrency) — Q21–Q25
6. [String Handling](#string-handling) — Q26–Q28
7. [Java 8+ Features](#java-8-features) — Q29–Q30
8. [JVM Internals](#jvm-internals) — Q31–Q33
9. [Design Patterns](#design-patterns) — Q34–Q35
10. [Common Coding Questions](#common-coding-questions) — Q36–Q40
11. [Streams API](#streams-api) — Q41–Q43
12. [Java Best Practices](#java-best-practices) — Q44–Q45
13. [Quick Reference Summary](#quick-reference-summary)

---

## Core Java Fundamentals

### Q1: What is the difference between JDK, JRE, and JVM?

**Answer:** JDK is the full development kit (compiler + tools + JRE). JRE is the runtime environment (JVM + class libraries) used to run apps. JVM is the virtual machine that executes bytecode and provides platform independence.

```
JDK
  ├─ JRE
  │   ├─ JVM
  │   └─ Class Libraries
  └─ Development Tools (javac, javadoc, jar)
```

**Relationship:** JDK ⊃ JRE ⊃ JVM

---

### Q2: What is the difference between `==` and `equals()`?

**Answer:** `==` compares references (memory addresses) for objects and values for primitives. `equals()` compares the actual content; its default Object implementation uses `==`, but String overrides it to compare characters.

```java
String str1 = new String("Hello");
String str2 = new String("Hello");
System.out.println(str1 == str2);       // false (different objects)
System.out.println(str1.equals(str2));  // true (same content)

String str3 = "Hello";
String str4 = "Hello";
System.out.println(str3 == str4);       // true (same interned literal)
```

**Best Practice:** Always override `equals()` and `hashCode()` together in custom classes.

---

### Q3: What is the difference between primitive types and wrapper classes?

**Answer:** Primitives (`int`, `double`, `boolean`, etc.) are stored on the stack and have no methods. Wrapper classes (`Integer`, `Double`, `Boolean`, etc.) wrap primitives as objects, enabling use in collections and providing utility methods.

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true  (cached range -128 to 127)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (no cache beyond 127)
```

Auto-boxing converts `int → Integer`; auto-unboxing converts `Integer → int`.

---

### Q4: What is the `final` keyword in Java?

**Answer:** `final` makes something non-modifiable. On a variable: cannot be reassigned (but an object's internals can still change). On a method: cannot be overridden. On a class: cannot be subclassed.

```java
final int x = 10;                      // Cannot reassign x
final List<String> list = new ArrayList<>();
list.add("item");                      // OK — modifying the object
// list = new ArrayList<>();           // Compile error — reassigning reference

class Parent { final void method() {} }  // Subclass cannot override method()
final class Immutable {}               // Cannot be extended
```

---

### Q5: What is the `static` keyword in Java?

**Answer:** `static` means the member belongs to the class, not to any instance. Static variables are shared across all instances. Static methods can be called without creating an object but cannot access instance members or `this`.

```java
public class Counter {
    private static int count = 0;

    static { System.out.println("Class loaded"); }  // runs once at class load

    public Counter() { count++; }

    public static int getCount() { return count; }  // no instance needed
}
```

---

## Object-Oriented Programming

### Q6: What are the four pillars of OOP?

**Answer:**

**1. Encapsulation** — bundle data with methods; hide internals behind access modifiers.
```java
public class BankAccount {
    private double balance;
    public void deposit(double amount) { if (amount > 0) balance += amount; }
    public double getBalance() { return balance; }
}
```

**2. Inheritance** — a subclass reuses and extends a parent class via `extends`.
```java
public class Dog extends Animal {
    public void bark() { System.out.println("Bark"); }
}
```

**3. Polymorphism** — same method, different behavior. Overloading = compile-time; overriding = runtime.
```java
Animal animal = new Dog();
animal.eat();  // calls Dog's eat() at runtime
```

**4. Abstraction** — hide implementation details; expose only essentials via abstract classes or interfaces.
```java
public abstract class Vehicle {
    public abstract void start();
    public void stop() { System.out.println("Stopped"); }
}
```

---

### Q7: What is the difference between abstract class and interface?

**Answer:** An abstract class can have state (fields, constructors) and both abstract and concrete methods; a class can extend only one. An interface defines a contract with no state; a class can implement many. Use an abstract class when sharing common state/behavior; use an interface to define a capability contract.

| Aspect | Abstract Class | Interface |
|--------|---------------|-----------|
| Methods | Abstract + concrete | Abstract by default (default/static since Java 8) |
| Fields | Instance variables allowed | Only `static final` constants |
| Constructors | Yes | No |
| Inheritance | Single | Multiple |

```java
public abstract class Animal {
    protected String name;
    public Animal(String name) { this.name = name; }
    public abstract void makeSound();
    public void sleep() { System.out.println(name + " sleeping"); }
}

public interface Flyable {
    void fly();
    default void takeOff() { System.out.println("Taking off"); }
}

public class Bird extends Animal implements Flyable {
    public Bird(String name) { super(name); }
    public void makeSound() { System.out.println("Chirp"); }
    public void fly() { System.out.println("Flying"); }
}
```

---

### Q8: What is method overloading vs method overriding?

**Answer:** Overloading = same name, different parameter list, same class — resolved at compile time. Overriding = same name and parameters, subclass replaces parent behavior — resolved at runtime.

```java
// Overloading (compile-time)
class Calculator {
    int add(int a, int b)       { return a + b; }
    double add(double a, double b) { return a + b; }
}

// Overriding (runtime)
class Animal { public void makeSound() { System.out.println("..."); } }
class Dog extends Animal {
    @Override public void makeSound() { System.out.println("Bark"); }
}

Animal a = new Dog();
a.makeSound();  // "Bark"
```

---

### Q9: What is constructor chaining in Java?

**Answer:** Calling one constructor from another. `this()` chains to another constructor in the same class; `super()` calls the parent constructor. Both must be the first statement and cannot be combined.

```java
class Animal {
    String name; int age;
    public Animal(String name) { this(name, 0); }          // chains internally
    public Animal(String name, int age) { this.name = name; this.age = age; }
}

class Dog extends Animal {
    String breed;
    public Dog(String name, int age, String breed) {
        super(name, age);    // calls Animal(String, int)
        this.breed = breed;
    }
}
```

---

### Q10: What is the difference between `this` and `super`?

**Answer:** `this` refers to the current instance; used to access instance members or call same-class constructors. `super` refers to the parent class; used to access parent members or call parent constructors.

```java
class Child extends Parent {
    private String name = "Child";
    public void showDetails() {
        System.out.println(this.name);   // "Child"
        System.out.println(super.name);  // "Parent"
        super.display();                 // Parent's display()
    }
}
```

---

## Collections Framework

### Q11: What is the difference between ArrayList and LinkedList?

**Answer:** ArrayList uses a dynamic array — fast random access O(1), slow middle insert/delete O(n). LinkedList uses a doubly-linked list — slow random access O(n), fast end insert/delete O(1). Prefer ArrayList for most use cases; LinkedList when you frequently add/remove at the ends.

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| get(index) | O(1) | O(n) |
| add(end) | O(1) amortized | O(1) |
| add(index, e) | O(n) | O(n) |
| remove(index) | O(n) | O(n) |

---

### Q12: What is the difference between HashMap and TreeMap?

**Answer:** HashMap uses a hash table — O(1) average for get/put, unordered, allows one null key. TreeMap uses a Red-Black tree — O(log n), keys sorted naturally or by Comparator, no null keys.

```java
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("C", 3); hashMap.put("A", 1); hashMap.put("B", 2);
System.out.println(hashMap);  // order not guaranteed

Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("C", 3); treeMap.put("A", 1); treeMap.put("B", 2);
System.out.println(treeMap);  // {A=1, B=2, C=3}
```

---

### Q13: What is the difference between HashSet and TreeSet?

**Answer:** HashSet (backed by HashMap) is unordered, allows one null, O(1) operations. TreeSet (backed by TreeMap) keeps elements sorted, no null, O(log n) operations.

---

### Q14: What is the difference between List, Set, and Map?

**Answer:**

| | List | Set | Map |
|---|---|---|---|
| Order | Ordered | Unordered | Unordered (TreeMap sorted) |
| Duplicates | Yes | No | Keys unique, values can repeat |
| Access | By index | No index | By key |
| Implementations | ArrayList, LinkedList | HashSet, TreeSet | HashMap, TreeMap |

---

### Q15: What is the difference between fail-fast and fail-safe iterators?

**Answer:** Fail-fast iterators (ArrayList, HashMap) throw `ConcurrentModificationException` if the collection is modified during iteration — they operate on the original. Fail-safe iterators (CopyOnWriteArrayList, ConcurrentHashMap) work on a snapshot copy so no exception is thrown.

```java
// Fail-fast
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) list.remove("A");  // throws ConcurrentModificationException
}

// Fail-safe
List<String> cowList = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
for (String s : cowList) {
    cowList.remove("A");  // no exception
}
```

---

## Exception Handling

### Q16: What is the difference between checked and unchecked exceptions?

**Answer:** Checked exceptions must be handled or declared (`throws`) and are verified at compile time — they represent recoverable conditions (e.g., `IOException`, `SQLException`). Unchecked exceptions extend `RuntimeException` and are not required to be declared — they represent programming errors (e.g., `NullPointerException`, `ArrayIndexOutOfBoundsException`).

```java
// Checked — must declare or catch
public void readFile(String f) throws IOException { new FileReader(f); }

// Unchecked — no declaration needed
public void divide(int a, int b) { int r = a / b; }  // ArithmeticException
```

---

### Q17: What are the exception-handling keywords?

**Answer:**

- `try` — code that might throw
- `catch` — handles a specific exception; multiple allowed, most specific first
- `finally` — always runs; used for cleanup
- `throw` — explicitly throws an exception instance
- `throws` — declares checked exceptions in a method signature

```java
public void example() throws IOException {
    try {
        throw new IOException("error");
    } catch (FileNotFoundException e) {
        System.out.println("Not found");
    } catch (IOException e) {
        System.out.println("IO error");
    } finally {
        System.out.println("Always runs");
    }
}
```

---

### Q18: What is the difference between `throw` and `throws`?

**Answer:** `throw` (inside a method) creates and throws one exception instance. `throws` (on the method signature) declares which checked exceptions callers must handle; multiple can be listed.

```java
public void method() throws IOException, SQLException {
    if (someCondition) throw new IOException("File error");
}
```

---

### Q19: What is a custom exception?

**Answer:** A user-defined exception class that extends `Exception` (checked) or `RuntimeException` (unchecked) to represent an application-specific error.

```java
public class InsufficientFundsException extends Exception {
    private final double amount;
    public InsufficientFundsException(String message, double amount) {
        super(message);
        this.amount = amount;
    }
    public double getAmount() { return amount; }
}

public void withdraw(double amount) throws InsufficientFundsException {
    if (amount > balance) throw new InsufficientFundsException("Insufficient funds", amount);
    balance -= amount;
}
```

---

### Q20: What is the difference between `Error` and `Exception`?

**Answer:** Both extend `Throwable`. `Error` signals serious JVM/system problems that apps generally should not catch (e.g., `OutOfMemoryError`, `StackOverflowError`). `Exception` represents application-level conditions that can often be caught and recovered from.

```
Throwable
├── Error            (Unrecoverable — OutOfMemoryError, StackOverflowError)
└── Exception        (Recoverable)
    ├── IOException  (Checked)
    └── RuntimeException (Unchecked — NullPointerException, etc.)
```

---

## Multithreading & Concurrency

### Q21: What is the difference between process and thread?

**Answer:** A process is an independent program with its own memory space (heavyweight). A thread is a lightweight unit of execution inside a process that shares memory with other threads in the same process.

| Aspect | Process | Thread |
|--------|---------|--------|
| Memory | Separate | Shared |
| Weight | Heavy | Light |
| Communication | Complex (IPC) | Simple (shared memory) |
| Context switch | Slow | Fast |

```java
Thread worker = new Thread(() -> System.out.println("Worker running"));
worker.start();
```

---

### Q22: What are the thread states in Java?

**Answer:** NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED.

- **NEW**: created, not started
- **RUNNABLE**: running or ready to run
- **BLOCKED**: waiting to acquire a monitor lock
- **WAITING**: waiting indefinitely (e.g., `wait()`)
- **TIMED_WAITING**: waiting for a set duration (e.g., `sleep()`)
- **TERMINATED**: finished

---

### Q23: What is the difference between `wait()` and `sleep()`?

**Answer:** `wait()` (Object method) releases the lock and waits until `notify()`/`notifyAll()` is called — must be inside a `synchronized` block. `sleep()` (Thread static method) pauses execution for a fixed time without releasing any lock.

```java
// wait() — releases lock
synchronized (lock) { lock.wait(); }

// sleep() — keeps lock
synchronized (lock) { Thread.sleep(2000); }
```

---

### Q24: What is the difference between a synchronized method and a synchronized block?

**Answer:** A synchronized method locks the entire object (`this`) for the duration of the method. A synchronized block locks only a specified object for a smaller section of code, giving finer control and reducing contention.

```java
public synchronized void method1() { /* entire method locked on 'this' */ }

public void method2() {
    // non-critical work
    synchronized (this) { /* only this block locked */ }
    // more non-critical work
}
```

---

### Q25: What is the `volatile` keyword?

**Answer:** `volatile` ensures every read/write of a variable goes to main memory, not a CPU cache, so changes are immediately visible to all threads. It does not provide atomicity — for compound operations like increment, use `synchronized` or atomic classes (`AtomicInteger`).

```java
private volatile boolean running = true;

public void stop() { running = false; }  // immediately visible to all threads
```

---

## String Handling

### Q26: What is the difference between String, StringBuilder, and StringBuffer?

**Answer:** `String` is immutable — every modification creates a new object. `StringBuilder` is mutable and fast but not thread-safe. `StringBuffer` is mutable and thread-safe but slower. Use `StringBuilder` for single-threaded string building; `String` for constants.

| | String | StringBuilder | StringBuffer |
|---|---|---|---|
| Mutable | No | Yes | Yes |
| Thread-safe | Yes | No | Yes |
| Performance | Slow (modifications) | Fast | Medium |

```java
// Inefficient — creates new String each iteration
String s = "";
for (int i = 0; i < 100; i++) s += i;

// Efficient
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100; i++) sb.append(i);
String result = sb.toString();
```

---

### Q27: What is String interning?

**Answer:** The JVM maintains a String pool. String literals are automatically interned (pooled), so two literals with the same value share the same object. `new String("Hello")` bypasses the pool; call `.intern()` to add it. Interned strings can be compared with `==`.

```java
String s1 = "Hello";
String s2 = "Hello";
String s3 = new String("Hello");
String s4 = s3.intern();

System.out.println(s1 == s2);  // true  (same pool object)
System.out.println(s1 == s3);  // false (heap object)
System.out.println(s1 == s4);  // true  (interned)
```

---

### Q28: What are important String methods?

**Answer:**

```java
String s = "Hello World";
s.length();              // 11
s.charAt(0);             // 'H'
s.indexOf('o');          // 4
s.contains("World");     // true
s.startsWith("Hello");   // true
s.substring(6);          // "World"
s.substring(0, 5);       // "Hello"
s.replace("World", "Java"); // "Hello Java"
s.toLowerCase();         // "hello world"
s.trim();                // strips leading/trailing whitespace
s.split(",");            // splits into String[]
String.join("-", "a","b"); // "a-b"
s.equals("Hello World");   // true
s.equalsIgnoreCase("hello world"); // true
String.format("Hi %s, age %d", "John", 25); // "Hi John, age 25"
```

---

## Java 8+ Features

### Q29: What are the key features introduced in Java 8?

**Answer:**

**Lambda Expressions** — concise anonymous functions:
```java
Runnable r = () -> System.out.println("Hello");
```

**Stream API** — declarative, functional-style operations on collections (lazy evaluation):
```java
List<String> names = Arrays.asList("John", "Jane", "Bob");
names.stream()
     .filter(n -> n.length() > 3)
     .map(String::toUpperCase)
     .forEach(System.out::println);
```

**Method References** — shorthand for lambdas:
```java
names.forEach(System.out::println);  // same as n -> System.out.println(n)
```

**Optional** — avoids null checks and NullPointerException:
```java
Optional<String> opt = Optional.of("Hello");
opt.ifPresent(System.out::println);
String val = opt.orElse("Default");
```

**Default Methods in Interfaces** — interface methods with a body:
```java
interface MyInterface {
    default void hello() { System.out.println("Default"); }
}
```

**Date/Time API (`java.time`)** — immutable, thread-safe replacement for `Date`/`Calendar`:
```java
LocalDate date = LocalDate.now();
LocalDateTime dt = LocalDateTime.now();
```

**Functional Interfaces** — interfaces with one abstract method; covered in Q30.

---

### Q30: What are functional interfaces in Java 8?

**Answer:** An interface with exactly one abstract method (SAM). Annotate with `@FunctionalInterface`. The four most common built-in ones:

| Interface | Signature | Use |
|-----------|-----------|-----|
| `Predicate<T>` | `boolean test(T t)` | filter / test |
| `Function<T,R>` | `R apply(T t)` | transform |
| `Consumer<T>` | `void accept(T t)` | consume/print |
| `Supplier<T>` | `T get()` | produce a value |

```java
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4));   // true

Function<String, Integer> len = String::length;
System.out.println(len.apply("Hello")); // 5

Consumer<String> print = System.out::println;
print.accept("Hi");

Supplier<Double> rand = Math::random;
System.out.println(rand.get());
```

Custom:
```java
@FunctionalInterface
interface MathOperation { int operate(int a, int b); }

MathOperation add = (a, b) -> a + b;
System.out.println(add.operate(5, 3));  // 8
```

---

## JVM Internals

### Q31: How does garbage collection work in Java?

**Answer:** The GC automatically reclaims memory occupied by objects no longer reachable from any live thread. It uses a generational model: most objects die young, so the heap is split into Young Generation (Eden + two Survivor spaces) and Old Generation. Objects start in Eden; survivors get promoted to Old Gen over time. Metaspace (Java 8+) stores class metadata.

Common GC algorithms to be aware of: **Serial** (single-threaded, small apps), **Parallel** (multi-threaded, high throughput), **G1** (modern default, large heaps), **ZGC** (ultra-low latency).

**Do not** call `System.gc()` in production code — it is only a hint to the JVM.

---

### Q32: What is the Java Memory Model?

**Answer:**

- **Heap** — shared by all threads; stores objects and arrays; managed by GC.
- **Stack** — private per thread; stores method frames, local variables, and object references.
- **Method Area** — stores class metadata, static variables, and method bytecode.
- **PC Register** — each thread has its own program counter.

```java
public class Example {
    private static int staticVar = 10;  // Method Area
    private int instanceVar;            // Heap

    public void method() {
        int localVar = 5;               // Stack
        String s = new String("Hi");    // reference on Stack, object on Heap
    }
}
```

---

### Q33: What is class loading in Java?

**Answer:** Class loading brings a `.class` file into JVM memory in three phases: **Loading** (reads the file, creates a `Class` object), **Linking** (verifies bytecode, allocates static fields, resolves references), **Initialization** (runs static blocks and assignments).

Three class loaders, following the parent-delegation model:
1. **Bootstrap** — loads `java.lang`, `java.util` (native code, no parent)
2. **Platform/Extension** — loads extension classes
3. **Application** — loads app classes from CLASSPATH

```java
ClassLoader cl = MyClass.class.getClassLoader();  // Application ClassLoader
Class<?> c = Class.forName("java.lang.String");   // dynamic load
```

---

## Design Patterns

### Q34: What are common design patterns in Java?

**Answer:**

**Singleton** — one instance only (see Q35 for full treatment):
```java
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```

**Factory** — create objects through a factory method, hiding concrete types:
```java
class ShapeFactory {
    public static Shape getShape(String type) {
        if ("CIRCLE".equalsIgnoreCase(type)) return new Circle();
        if ("RECTANGLE".equalsIgnoreCase(type)) return new Rectangle();
        return null;
    }
}
Shape s = ShapeFactory.getShape("CIRCLE");
s.draw();
```

**Observer** — one-to-many notification when state changes:
```java
class Subject {
    private List<Observer> observers = new ArrayList<>();
    public void attach(Observer o) { observers.add(o); }
    public void notifyAll(String msg) { observers.forEach(o -> o.update(msg)); }
}
```

**Strategy** — swap algorithms at runtime:
```java
interface PaymentStrategy { void pay(int amount); }
class CreditCard implements PaymentStrategy {
    public void pay(int amount) { System.out.println("Card: " + amount); }
}

ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCard());
cart.checkout(100);
```

---

### Q35: What is the Singleton pattern and how do you implement it?

**Answer:** Ensures a class has exactly one instance and provides global access to it. Three common thread-safe approaches:

**Double-Checked Locking** (shown in Q34) — lazy, works in most cases.

**Static Inner Class** (preferred — lazy + thread-safe without synchronization overhead):
```java
public class Singleton {
    private Singleton() {}
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

**Enum** (simplest, also serialization-safe):
```java
public enum Singleton {
    INSTANCE;
    public void doWork() { System.out.println("Working"); }
}
Singleton.INSTANCE.doWork();
```

| Implementation | Lazy | Thread-safe | Serialization-safe |
|----------------|------|-------------|-------------------|
| Eager field | No | Yes | Needs work |
| Double-checked locking | Yes | Yes | Needs work |
| Static inner class | Yes | Yes | Needs work |
| Enum | No | Yes | Yes |

---

## Common Coding Questions

### Q36: Find the second largest number in an array

```java
public static int findSecondLargest(int[] arr) {
    if (arr.length < 2) throw new IllegalArgumentException("Need at least 2 elements");
    int largest = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int n : arr) {
        if (n > largest) { second = largest; largest = n; }
        else if (n > second && n != largest) { second = n; }
    }
    if (second == Integer.MIN_VALUE) throw new IllegalArgumentException("No second largest");
    return second;
}
// findSecondLargest(new int[]{12, 35, 1, 10, 34}) → 34
```

---

### Q37: Check if a string is a palindrome

```java
// Two-pointer approach (O(n) time, O(1) space)
public static boolean isPalindrome(String s) {
    int l = 0, r = s.length() - 1;
    while (l < r) {
        if (s.charAt(l++) != s.charAt(r--)) return false;
    }
    return true;
}

// Ignoring case and non-alphanumeric
public static boolean isPalindromeClean(String s) {
    String clean = s.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
    return isPalindrome(clean);
}
// isPalindromeClean("A man, a plan, a canal: Panama") → true
```

---

### Q38: Reverse a string

```java
// Option 1: StringBuilder (concise)
public static String reverse(String s) {
    return new StringBuilder(s).reverse().toString();
}

// Option 2: Two-pointer char array (manual)
public static String reverseManual(String s) {
    char[] c = s.toCharArray();
    for (int l = 0, r = c.length - 1; l < r; l++, r--) {
        char tmp = c[l]; c[l] = c[r]; c[r] = tmp;
    }
    return new String(c);
}
```

---

### Q39: Find duplicates in an array

```java
// Using HashSet (O(n) time)
public static <T> Set<T> findDuplicates(T[] array) {
    Set<T> seen = new HashSet<>(), duplicates = new HashSet<>();
    for (T e : array) if (!seen.add(e)) duplicates.add(e);
    return duplicates;
}

// Using Streams (Java 8+)
public static <T> Set<T> findDuplicatesStream(T[] array) {
    Set<T> seen = new HashSet<>();
    return Arrays.stream(array).filter(e -> !seen.add(e)).collect(Collectors.toSet());
}
// findDuplicates(new Integer[]{1,2,3,2,3,4}) → [2, 3]
```

---

### Q40: Sort an array

```java
int[] nums = {5, 2, 8, 1, 9};
Arrays.sort(nums);                              // ascending
System.out.println(Arrays.toString(nums));      // [1, 2, 5, 8, 9]

String[] names = {"John", "Alice", "Bob"};
Arrays.sort(names);                             // natural (alphabetical)
Arrays.sort(names, Comparator.reverseOrder());  // descending

List<Integer> list = new ArrayList<>(Arrays.asList(5, 2, 8));
Collections.sort(list);                         // ascending
list.sort(Comparator.reverseOrder());           // descending
```

---

## Streams API

### Q41: What is the difference between Collection and Stream?

**Answer:** A Collection stores data in memory and can be iterated multiple times and modified. A Stream is a one-time pipeline for processing data with lazy evaluation — once consumed, it cannot be reused.

```java
// Stream — lazy, one-time
Stream<String> s = names.stream();
s.forEach(System.out::println);
// s.forEach(System.out::println);  // IllegalStateException — already consumed
```

---

### Q42: What are intermediate and terminal operations?

**Answer:** Intermediate operations (filter, map, sorted, distinct, limit, skip) return a new Stream and are **lazy** — they don't execute until a terminal operation is called. Terminal operations (forEach, collect, count, reduce, anyMatch) trigger execution and produce a result.

```java
long count = names.stream()
    .filter(n -> n.length() > 3)  // intermediate
    .distinct()                    // intermediate
    .sorted()                      // intermediate
    .count();                      // terminal — triggers all above
```

---

### Q43: What are the common Stream operations?

**Answer:**

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6);

// filter
nums.stream().filter(n -> n % 2 == 0).collect(Collectors.toList()); // [2,4,6]

// map
List.of("a","b").stream().map(String::toUpperCase).collect(Collectors.toList()); // [A,B]

// flatMap — flatten nested lists
List.of(List.of(1,2), List.of(3,4)).stream()
    .flatMap(List::stream).collect(Collectors.toList()); // [1,2,3,4]

// reduce
int sum = nums.stream().reduce(0, Integer::sum);  // 21

// collect to various types
Collectors.toList()
Collectors.toSet()
Collectors.joining(", ")
Collectors.toMap(name -> name, String::length)
Collectors.groupingBy(String::length)

// sorted
nums.stream().sorted().collect(Collectors.toList());                    // ascending
nums.stream().sorted(Comparator.reverseOrder()).collect(Collectors.toList()); // descending
```

---

## Java Best Practices

### Q44: What are Java coding best practices?

**Answer:**

**Naming conventions:**
```java
class UserService {}           // PascalCase
void getUserById() {}          // camelCase
private String userName;       // camelCase
static final int MAX = 3;      // UPPER_SNAKE_CASE
package com.example.service;   // lowercase
```

**Catch specific exceptions:**
```java
// Good
try { ... } catch (FileNotFoundException e) { ... } catch (IOException e) { ... }

// Bad — never swallow generic Exception
try { ... } catch (Exception e) { /* don't do this */ }
```

**Use try-with-resources for I/O:**
```java
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line;
    while ((line = br.readLine()) != null) System.out.println(line);
}  // auto-closed
```

**Replace magic numbers with named constants:**
```java
private static final int MINIMUM_AGE = 18;
if (age < MINIMUM_AGE) throw new IllegalArgumentException("Under age");
```

**Prefer immutable objects when possible:**
```java
public final class Person {
    private final String name;
    private final int age;
    public Person(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
}
```

---

### Q45: What are common Java performance tips?

**Answer:**

1. **String concatenation in loops** — use `StringBuilder`, not `+`.
2. **Prefer primitives over wrappers** where possible to avoid boxing overhead.
3. **Choose the right collection** — ArrayList for random access, HashSet/HashMap for O(1) lookups.
4. **Avoid creating objects inside loops** — create once, reuse (e.g., `SimpleDateFormat`).
5. **Lazy initialization** — create expensive objects only when needed.
6. **Streams vs loops** — streams are more readable; plain loops are slightly faster for simple operations. Use streams for clarity, loops for tight hot paths.

```java
// Bad — new formatter per iteration
for (int i = 0; i < 1000; i++) {
    String d = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
}
// Good — reuse
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 1000; i++) { String d = sdf.format(new Date()); }
```

---

## Quick Reference Summary

### Keywords
| Keyword | Meaning |
|---------|---------|
| `final` | Variable can't be reassigned; method can't be overridden; class can't be extended |
| `static` | Belongs to class, not instance |
| `abstract` | Class can't be instantiated; method has no body |
| `synchronized` | Only one thread at a time |
| `volatile` | Always read/write from main memory |

### Collections Cheat Sheet
| Type | Interface | Ordered | Duplicates | Null | Complexity |
|------|-----------|---------|------------|------|------------|
| ArrayList | List | Yes (insertion) | Yes | Yes | get O(1) |
| LinkedList | List/Deque | Yes (insertion) | Yes | Yes | add-ends O(1) |
| HashSet | Set | No | No | One | O(1) |
| TreeSet | SortedSet | Sorted | No | No | O(log n) |
| HashMap | Map | No | Keys: No | One null key | O(1) |
| TreeMap | SortedMap | Sorted by key | Keys: No | No | O(log n) |

### Exception Hierarchy
```
Throwable
├── Error            (don't catch — OutOfMemoryError, StackOverflowError)
└── Exception
    ├── Checked      (must handle — IOException, SQLException)
    └── RuntimeException (unchecked — NullPointerException, IllegalArgumentException)
```

### Java 8 Functional Interfaces
| Interface | Method | Use |
|-----------|--------|-----|
| `Predicate<T>` | `boolean test(T)` | filter |
| `Function<T,R>` | `R apply(T)` | transform |
| `Consumer<T>` | `void accept(T)` | consume |
| `Supplier<T>` | `T get()` | produce |

**Interview tip:** Explain trade-offs, not just definitions. Show with code when you can. For every "what" question, be ready to answer "when would you use it?"

---

*Last Updated: 2026-06-18*
