# Java Developer Interview Questions & Answers

## Core Java Fundamentals

### Q1: What is the difference between JDK, JRE, and JVM?

**Answer:**

**JDK (Java Development Kit):**
- Development environment for building Java applications
- Includes JRE, compiler (javac), and development tools
- Used by developers to create Java applications
- Contains tools like javac, javadoc, jar, etc.

**JRE (Java Runtime Environment):**
- Runtime environment for executing Java applications
- Includes JVM and Java class libraries
- Used by end-users to run Java applications
- Doesn't include development tools

**JVM (Java Virtual Machine):**
- Virtual machine that executes Java bytecode
- Provides platform independence (Write Once, Run Anywhere)
- Converts bytecode to machine code
- Handles memory management, garbage collection

**Relationship:** JDK → JRE → JVM

```
JDK (Development Kit)
  ├─ JRE (Runtime Environment)
  │   ├─ JVM (Virtual Machine)
  │   └─ Class Libraries
  └─ Development Tools (javac, javadoc, jar)
```

### Q2: What is the difference between `==` and `equals()`?

**Answer:**

**== Operator:**
- Compares references (memory addresses) for objects
- Compares values for primitive types
- Checks if two references point to the same object

**equals() Method:**
- Compares the actual content/state of objects
- Default implementation in Object class uses == operator
- Can be overridden to provide custom comparison logic

**Examples:**
```java
String str1 = new String("Hello");
String str2 = new String("Hello");
String str3 = "Hello";

System.out.println(str1 == str2);        // false (different objects)
System.out.println(str1.equals(str2));   // true (same content)
System.out.println(str1 == str3);        // false (different objects)
System.out.println(str1.equals(str3));   // true (same content)

// String interning
String str4 = "Hello";
System.out.println(str3 == str4);        // true (same interned string)
```

**Best Practice:** Always override `equals()` and `hashCode()` together when defining custom classes.

### Q3: What is the difference between primitive types and wrapper classes?

**Answer:**

**Primitive Types:**
- Basic data types built into Java
- Not objects, don't have methods
- Stored on stack
- More memory efficient
- Faster performance
- Examples: int, double, boolean, char, etc.

**Wrapper Classes:**
- Classes that wrap primitive types
- Part of the java.lang package
- Stored on heap (as objects)
- Provide utility methods
- Used in collections (which require objects)
- Examples: Integer, Double, Boolean, Character, etc.

**Comparison:**
```java
int primitive = 10;
Integer wrapper = Integer.valueOf(10);

// Auto-boxing and auto-unboxing
Integer autoBoxed = primitive;      // Auto-boxing
int autoUnboxed = wrapper;         // Auto-unboxing

// Important: Wrapper comparison
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true (caching for -128 to 127)

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false (no caching beyond 127)
```

### Q4: What is the `final` keyword in Java?

**Answer:** The `final` keyword can be applied to variables, methods, and classes to make them immutable or non-modifiable.

**Final Variables:**
- Cannot be reassigned once initialized
- Must be initialized (either at declaration or in constructor)
- For reference variables, the reference cannot be changed, but the object's state can be

```java
final int x = 10;          // Cannot be reassigned
final int y;               // Must be initialized in constructor
final List<String> list = new ArrayList<>();
list.add("item");          // OK - changing object state
// list = new ArrayList<>(); // NOT OK - cannot reassign reference
```

**Final Methods:**
- Cannot be overridden by subclasses
- Used to prevent method overriding

```java
class Parent {
    final void method() {
        // Cannot be overridden
    }
}
```

**Final Classes:**
- Cannot be extended (inherited from)
- Used for immutable classes and security

```java
final class ImmutableClass {
    // Cannot be extended
}
```

### Q5: What is the `static` keyword in Java?

**Answer:** The `static` keyword indicates that a member belongs to the class rather than instances.

**Static Variables:**
- Shared among all instances of the class
- Loaded when the class is loaded
- Accessed using class name

**Static Methods:**
- Can be called without creating an instance
- Cannot access instance variables or methods directly
- Cannot use `this` keyword

**Static Blocks:**
- Executed when the class is loaded
- Used for static initialization

**Static Inner Classes:**
- Associated with the outer class, not instances
- Cannot access non-static members of outer class

```java
public class Counter {
    private static int count = 0;      // Static variable
    private int instanceCount = 0;     // Instance variable

    static {
        // Static block - runs once when class loads
        System.out.println("Counter class loaded");
    }

    public Counter() {
        count++;           // Access static variable
        instanceCount++;  // Access instance variable
    }

    public static int getCount() {    // Static method
        return count;                 // Can access static variables only
    }

    public int getInstanceCount() {   // Instance method
        return instanceCount;
    }

    static class StaticInnerClass {   // Static inner class
        // Cannot access instance members of outer class
    }
}
```

---

## Object-Oriented Programming

### Q6: What are the four pillars of OOP?

**Answer:**

**1. Encapsulation:**
- Bundling data and methods that operate on that data
- Hiding internal details and providing access through methods
- Achieved using access modifiers (private, protected, public)
- Example: JavaBeans with private fields and getter/setter methods

```java
public class BankAccount {
    private double balance;  // Encapsulated data

    public void deposit(double amount) {
        if (amount > 0) {
            balance += amount;
        }
    }

    public double getBalance() {
        return balance;
    }
}
```

**2. Inheritance:**
- Creating new classes based on existing classes
- Reusing code and creating hierarchical relationships
- Achieved using `extends` keyword
- Java supports single inheritance (one class can extend only one class)

```java
public class Animal {
    public void eat() {
        System.out.println("Animal is eating");
    }
}

public class Dog extends Animal {  // Inheritance
    public void bark() {
        System.out.println("Dog is barking");
    }
}
```

**3. Polymorphism:**
- Objects taking many forms
- Same method behaving differently based on the object
- Two types: Compile-time (overloading) and Runtime (overriding)

```java
// Compile-time polymorphism (method overloading)
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
}

// Runtime polymorphism (method overriding)
Animal animal = new Dog();
animal.eat();  // Calls Dog's eat() method
```

**4. Abstraction:**
- Hiding complex implementation details
- Showing only essential features
- Achieved using abstract classes and interfaces

```java
public abstract class Vehicle {
    public abstract void start();  // Abstract method
    public void stop() {           // Concrete method
        System.out.println("Vehicle stopped");
    }
}

public interface Drawable {
    void draw();  // Abstract method by default
}
```

### Q7: What is the difference between abstract class and interface?

**Answer:**

**Abstract Class:**
- Can have both abstract and concrete methods
- Can have instance variables (fields)
- Can have constructors
- Can have static and final methods
- Single inheritance (can extend only one abstract class)
- Used when classes share common state and behavior

**Interface:**
- All methods are abstract by default (prior to Java 8)
- Cannot have instance variables (only constants)
- Cannot have constructors
- Can have default and static methods (Java 8+)
- Multiple inheritance (can implement multiple interfaces)
- Used to define a contract without implementation

**Key Differences:**

| Aspect | Abstract Class | Interface |
|--------|---------------|------------|
| Method Implementation | Can have both abstract and concrete methods | All methods abstract by default |
| Variables | Can have instance variables | Only static final constants |
| Constructors | Can have constructors | Cannot have constructors |
| Inheritance | Single inheritance | Multiple inheritance |
| State | Can maintain state | Cannot maintain state |
| Use Case | Share code and state | Define contract |

**Example:**
```java
// Abstract class example
public abstract class Animal {
    protected String name;
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public abstract void makeSound();  // Abstract method

    public void sleep() {              // Concrete method
        System.out.println(name + " is sleeping");
    }
}

// Interface example
public interface Flyable {
    void fly();                        // Abstract method

    default void takeOff() {           // Default method (Java 8+)
        System.out.println("Taking off");
    }

    static void checkWeather() {       // Static method (Java 8+)
        System.out.println("Checking weather");
    }
}

// A class can extend an abstract class AND implement an interface
public class Bird extends Animal implements Flyable {
    public Bird(String name, int age) { super(name, age); }

    @Override
    public void makeSound() { System.out.println("Chirp chirp"); }

    @Override
    public void fly() { System.out.println("Flying high"); }
}
```

### Q8: What is method overloading vs method overriding?

**Answer:**

**Method Overloading (Compile-time Polymorphism):**
- Same method name, different parameters
- Same class or subclass
- Different parameter lists (number, type, or order)
- Can have different return types
- Does NOT affect polymorphism

```java
public class Calculator {
    // Overloaded methods
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }
}
```

**Method Overriding (Runtime Polymorphism):**
- Same method name, same parameters
- Subclass provides specific implementation
- Same return type or covariant return type
- Cannot reduce access modifier visibility
- Used for runtime polymorphism

```java
class Animal {
    public void makeSound() {
        System.out.println("Some sound");
    }
}

class Dog extends Animal {
    @Override
    public void makeSound() {
        System.out.println("Bark");
    }
}

class Cat extends Animal {
    @Override
    public void makeSound() {
        System.out.println("Meow");
    }
}

// Runtime polymorphism
Animal animal = new Dog();  // Upcasting
animal.makeSound();        // Prints "Bark"

animal = new Cat();
animal.makeSound();        // Prints "Meow"
```

### Q9: What is constructor chaining in Java?

**Answer:** Constructor chaining is the process of calling one constructor from another constructor in the same class (using `this()`) or from the parent class (using `super()`).

**Key Points:**
- `this()` calls another constructor in the same class
- `super()` calls the parent class constructor
- Must be the first statement in the constructor
- Cannot use both `this()` and `super()` in the same constructor

```java
class Animal {
    private String name;
    private int age;

    public Animal(String name) {
        this(name, 0);  // this() - calls the other constructor in this class
    }

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

class Dog extends Animal {
    private String breed;

    public Dog(String name, int age, String breed) {
        super(name, age);  // super() - calls the parent constructor
        this.breed = breed;
    }
}
```

### Q10: What is the difference between `this` and `super` keyword?

**Answer:**

**`this` keyword:**
- Refers to the current instance of the class
- Used to access instance variables and methods
- Used to call constructors of the same class
- Used to pass current object as a parameter

**`super` keyword:**
- Refers to the parent class object
- Used to access parent class variables and methods
- Used to call parent class constructors
- Used to access hidden members

**Comparison:**
```java
class Parent {
    protected String name = "Parent";
    public void display() { System.out.println("Parent display"); }
}

class Child extends Parent {
    private String name = "Child";

    public Child() {
        super();  // Calls parent constructor
    }

    public void showDetails() {
        System.out.println(this.name);    // "Child"
        System.out.println(super.name);   // "Parent"
        this.display();                   // Calls Child's display
        super.display();                  // Calls Parent's display
    }

    @Override
    public void display() { System.out.println("Child display"); }
}
```

---

## Collections Framework

### Q11: What is the difference between ArrayList and LinkedList?

**Answer:**

**ArrayList:**
- Implemented using dynamic array
- Random access is fast (O(1))
- Insertion/deletion in middle is slow (O(n))
- Less memory overhead
- Default capacity is 10, grows by 1.5x
- Better for frequent access, less frequent modifications

**LinkedList:**
- Implemented using doubly-linked list
- Random access is slow (O(n))
- Insertion/deletion at ends is fast (O(1))
- More memory overhead (each node has 2 references)
- Better for frequent insertions/deletions
- Implements Queue and Deque interfaces

**Performance Comparison:**

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| get(index) | O(1) | O(n) |
| add(element) | O(1)* | O(1) |
| add(index, element) | O(n) | O(n) |
| remove(index) | O(n) | O(n) |
| remove(element) | O(n) | O(n) |

*O(1) amortized, O(n) during resize

**Example:**
```java
// ArrayList - Good for random access
List<Integer> arrayList = new ArrayList<>();
arrayList.add(1);
arrayList.add(2);
int value = arrayList.get(0);  // Fast - O(1)

// LinkedList - Good for frequent modifications
List<Integer> linkedList = new LinkedList<>();
linkedList.add(1);
linkedList.addFirst(0);  // Fast - O(1)
linkedList.removeLast(); // Fast - O(1)

// Wrong usage - bad performance
List<Integer> badLinkedList = new LinkedList<>();
for (int i = 0; i < 10000; i++) {
    badLinkedList.get(i);  // Very slow - O(n) for each access
}
```

### Q12: What is the difference between HashMap and TreeMap?

**Answer:**

**HashMap:**
- Uses hash table implementation
- Keys are not sorted
- O(1) average time for get/put operations
- Allows one null key and multiple null values
- Not thread-safe
- Faster for general operations

**TreeMap:**
- Uses Red-Black tree implementation
- Keys are sorted in natural order or by Comparator
- O(log n) time for get/put operations
- Does not allow null keys
- Not thread-safe
- Provides additional methods like firstKey(), lastKey()

**Example:**
```java
// HashMap - Unsorted
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("C", 3);
hashMap.put("A", 1);
hashMap.put("B", 2);
System.out.println(hashMap);  // {A=1, C=3, B=2} - order not guaranteed

// TreeMap - Sorted by keys
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("C", 3);
treeMap.put("A", 1);
treeMap.put("B", 2);
System.out.println(treeMap);  // {A=1, B=2, C=3} - sorted order

// TreeMap also offers firstKey(), lastKey(), headMap(), tailMap()
```

### Q13: What is the difference between HashSet and TreeSet?

**Answer:**

**HashSet:**
- Internally uses HashMap
- Does not maintain any order
- Allows one null element
- O(1) average time for add/remove/contains
- Faster for most operations

**TreeSet:**
- Internally uses TreeMap
- Maintains elements in sorted order
- Does not allow null elements
- O(log n) time for add/remove/contains
- Implements SortedSet interface

**Example:**
```java
// HashSet - No ordering
Set<Integer> hashSet = new HashSet<>();
hashSet.add(3);
hashSet.add(1);
hashSet.add(2);
System.out.println(hashSet);  // [1, 2, 3] - order not guaranteed

// TreeSet - Sorted order
Set<Integer> treeSet = new TreeSet<>();
treeSet.add(3);
treeSet.add(1);
treeSet.add(2);
System.out.println(treeSet);  // [1, 2, 3] - sorted order

// TreeSet also offers first(), last(), headSet(), tailSet()
```

### Q14: What is the difference between List, Set, and Map?

**Answer:**

**List:**
- Ordered collection of elements
- Allows duplicate elements
- Indexed access (get(index))
- Common implementations: ArrayList, LinkedList, Vector

**Set:**
- Unordered collection of unique elements
- Does not allow duplicate elements
- No indexed access
- Common implementations: HashSet, TreeSet, LinkedHashSet

**Map:**
- Collection of key-value pairs
- Each key must be unique
- Values can be duplicated
- Common implementations: HashMap, TreeMap, LinkedHashMap

**Example:**
```java
// List - Ordered, allows duplicates
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("A");  // Duplicate allowed
System.out.println(list.get(0));  // A - indexed access

// Set - Unique elements
Set<String> set = new HashSet<>();
set.add("A");
set.add("B");
set.add("A");  // Duplicate ignored
// set.get(0)  // No indexed access

// Map - Key-value pairs
Map<String, Integer> map = new HashMap<>();
map.put("A", 1);
map.put("B", 2);
map.put("A", 3);  // Updates existing key
System.out.println(map.get("A"));  // 3
```

### Q15: What is the difference between fail-fast and fail-safe iterators?

**Answer:**

**Fail-fast Iterators:**
- Throw ConcurrentModificationException if collection is modified during iteration
- Work directly on the original collection
- Default behavior for most collections
- Used in ArrayList, HashMap, HashSet

**Fail-safe Iterators:**
- Don't throw exception if collection is modified
- Work on a copy of the collection
- Additional memory overhead
- Used in CopyOnWriteArrayList, ConcurrentHashMap

**Example:**
```java
// Fail-fast iterator
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("C");

Iterator<String> failFast = list.iterator();
while (failFast.hasNext()) {
    String element = failFast.next();
    if (element.equals("B")) {
        list.remove("A");  // ConcurrentModificationException
    }
}

// Fail-safe iterator - works on a copy, so no exception
List<String> copyOnWriteList = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
Iterator<String> failSafe = copyOnWriteList.iterator();
while (failSafe.hasNext()) {
    if (failSafe.next().equals("B")) {
        copyOnWriteList.remove("A");  // No ConcurrentModificationException
    }
}
```

---

## Exception Handling

### Q16: What is the difference between checked and unchecked exceptions?

**Answer:**

**Checked Exceptions:**
- Must be handled or declared in method signature
- Checked at compile time
- Represent conditions from which recovery might be possible
- Subclasses of Exception (except RuntimeException)
- Examples: IOException, SQLException, FileNotFoundException

**Unchecked Exceptions:**
- Don't need to be handled or declared
- Not checked at compile time
- Represent programming errors
- Subclasses of RuntimeException and Error
- Examples: NullPointerException, ArithmeticException, ArrayIndexOutOfBoundsException

**Example:**
```java
// Checked exception
public void readFile(String filename) throws IOException {
    FileReader reader = new FileReader(filename);
    // Must handle IOException or declare it
}

// Unchecked exception
public void divide(int a, int b) {
    int result = a / b;  // ArithmeticException - unchecked
    // No need to handle or declare
}

// Handling checked exception
public void readFileSafely(String filename) {
    try {
        FileReader reader = new FileReader(filename);
        // Process file
    } catch (IOException e) {
        // Handle the exception
        System.out.println("Error reading file: " + e.getMessage());
    }
}
```

### Q17: What are the keywords used in exception handling?

**Answer:**

**try:**
- Contains code that might throw an exception
- Must be followed by catch or finally block

**catch:**
- Handles specific exceptions thrown in try block
- Can have multiple catch blocks
- Handles exceptions from most specific to most general

**finally:**
- Always executes regardless of exception occurrence
- Used for cleanup operations
- Optional but highly recommended for resource cleanup

**throw:**
- Used to explicitly throw an exception
- Can throw checked or unchecked exceptions

**throws:**
- Declares exceptions that a method might throw
- Used in method signature
- Required for checked exceptions

**Example:**
```java
public void example() throws IOException {
    try {
        // Code that might throw exceptions
        throw new IOException("Custom error");  // 'throw' - explicit throw
    } catch (FileNotFoundException e) {
        System.out.println("File not found");
        throw e;  // Re-throw exception
    } catch (IOException e) {
        System.out.println("IO error");
    } finally {
        System.out.println("Always executes - cleanup goes here");
    }
}
```

### Q18: What is the difference between `throw` and `throws`?

**Answer:**

**throw:**
- Used to explicitly throw an exception
- Used within a method
- Can throw only one exception at a time
- Instance of exception is created and thrown

**throws:**
- Used to declare exceptions that a method might throw
- Used in method signature
- Can declare multiple exceptions
- Indicates which exceptions the caller must handle

**Example:**
```java
public void method1() throws IOException, SQLException {
    // Declares multiple exceptions
    if (someCondition) {
        throw new IOException("File error");  // Throws specific exception
    }
    if (anotherCondition) {
        throw new SQLException("Database error");
    }
}

public void method2() {
    try {
        method1();  // Must handle or declare checked exceptions
    } catch (IOException | SQLException e) {
        // Handle exceptions
        e.printStackTrace();
    }
}
```

### Q19: What is a custom exception?

**Answer:** A custom exception is a user-defined exception that extends an existing exception class to handle application-specific scenarios.

**Creating Custom Exceptions:**

```java
// Custom checked exception
public class InsufficientFundsException extends Exception {
    private double amount;
    private double available;

    public InsufficientFundsException(String message, double amount, double available) {
        super(message);
        this.amount = amount;
        this.available = available;
    }

    public double getAmount() {
        return amount;
    }

    public double getAvailable() {
        return available;
    }
}

// Custom unchecked exception
public class InvalidAgeException extends RuntimeException {
    public InvalidAgeException(String message) {
        super(message);
    }
}

// Using a custom exception
public void withdraw(double amount) throws InsufficientFundsException {
    if (amount > balance) {
        throw new InsufficientFundsException("Insufficient funds", amount, balance);
    }
    balance -= amount;
}
```

### Q20: What is the difference between `Error` and `Exception`?

**Answer:**

**Error:**
- Represents serious problems that applications should not try to catch
- Usually related to JVM or system-level issues
- Subclass of Throwable
- Generally unrecoverable
- Examples: OutOfMemoryError, StackOverflowError, VirtualMachineError

**Exception:**
- Represents conditions that applications might want to catch
- Application-level issues that can be handled
- Subclass of Throwable
- Generally recoverable
- Examples: IOException, RuntimeException, NullPointerException

**Hierarchy:**
```
Throwable
├── Error (Unrecoverable)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
└── Exception (Recoverable)
    ├── IOException (Checked)
    ├── SQLException (Checked)
    └── RuntimeException (Unchecked)
        ├── NullPointerException
        ├── ArithmeticException
        └── ArrayIndexOutOfBoundsException
```

---

## Multithreading & Concurrency

### Q21: What is the difference between process and thread?

**Answer:**

**Process:**
- Independent program in execution
- Has its own memory space
- Heavyweight, requires more resources
- Communication between processes is complex
- Isolated from other processes

**Thread:**
- Lightweight unit within a process
- Shares memory with other threads in the same process
- Fast context switching
- Easy communication via shared memory
- Part of a process

**Comparison:**

| Aspect | Process | Thread |
|--------|---------|---------|
| Memory | Separate memory space | Shared memory |
| Resources | Heavyweight | Lightweight |
| Communication | Complex (IPC) | Simple (shared memory) |
| Context Switch | Slow | Fast |
| Isolation | Complete isolation | Shared resources |

**Example:**
```java
// Creating and starting a thread with a Runnable lambda
public class ThreadExample {
    public static void main(String[] args) {
        Thread worker = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Worker: " + i);
            }
        });

        worker.start();
        System.out.println("Main thread continues...");
    }
}
```

### Q22: What are the different states of a thread in Java?

**Answer:**

**Thread States:**
1. **NEW**: Thread created but not started
2. **RUNNABLE**: Thread is running or ready to run
3. **BLOCKED**: Thread waiting to acquire a monitor lock
4. **WAITING**: Thread waiting indefinitely for another thread
5. **TIMED_WAITING**: Thread waiting for a specified time
6. **TERMINATED**: Thread has completed execution

**State Transitions:**
```
NEW --start()--> RUNNABLE
RUNNABLE --sleep()/wait()--> WAITING/TIMED_WAITING
WAITING --notify()/notifyAll()--> RUNNABLE
BLOCKED --acquires lock--> RUNNABLE
RUNNABLE --completes--> TERMINATED
```

**Example:**
```java
Thread thread = new Thread(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) {}
});

System.out.println(thread.getState());  // NEW
thread.start();
System.out.println(thread.getState());  // RUNNABLE
Thread.sleep(100);
System.out.println(thread.getState());  // TIMED_WAITING (during sleep)
```

### Q23: What is the difference between `wait()` and `sleep()`?

**Answer:**

**wait():**
- Releases the lock on the object
- Must be called from synchronized block/context
- Can be woken up by notify() or notifyAll()
- Instance method of Object class
- Used for inter-thread communication

**sleep():**
- Does not release any locks
- Can be called from any context
- Wakes up after specified time elapses
- Static method of Thread class
- Used to pause thread execution

**Example:**
```java
final Object lock = new Object();

// Waiting thread - releases the lock while waiting
new Thread(() -> {
    synchronized (lock) {
        try {
            lock.wait();              // Releases lock, waits for notify()
            System.out.println("Resumed");
        } catch (InterruptedException e) {}
    }
}).start();

// Notifying thread
new Thread(() -> {
    synchronized (lock) {
        try {
            Thread.sleep(2000);      // Does NOT release the lock
            lock.notify();           // Wakes up the waiting thread
        } catch (InterruptedException e) {}
    }
}).start();
```

### Q24: What is the difference between `synchronized` method and block?

**Answer:**

**Synchronized Method:**
- Locks the entire object
- Locks on `this` for instance methods
- Locks on the Class object for static methods
- Less flexible and potentially slower
- Coarse-grained locking

**Synchronized Block:**
- Locks only the specified object
- More precise control over locked scope
- More flexible and potentially faster
- Fine-grained locking
- Can reduce contention

**Example:**
```java
public class SynchronizationExample {

    // Synchronized method - locks entire object
    public synchronized void method1() {
        // Entire method is synchronized
        System.out.println("Method 1 start");
        // Critical section
        System.out.println("Method 1 end");
    }

    // Synchronized block - more precise locking
    public void method2() {
        System.out.println("Method 2 start");
        // Only this block is synchronized (here, on 'this';
        // a dedicated private lock object can be used for finer control)
        synchronized (this) {
            // Critical section
        }
        System.out.println("Method 2 end");
    }
}
```

### Q25: What is `volatile` keyword in Java?

**Answer:** The `volatile` keyword ensures that a variable is always read from and written to main memory, not from CPU cache.

**Key Points:**
- Ensures visibility of changes to variables across threads
- Prevents compiler optimizations and CPU cache
- Provides happens-before guarantee
- Does not provide atomicity for compound operations
- Lighter weight than synchronized

**Example:**
```java
public class VolatileExample {
    private volatile boolean running = true;

    public void start() {
        new Thread(() -> {
            while (running) {
                // Do some work
            }
            System.out.println("Thread stopped");
        }).start();
    }

    public void stop() {
        running = false;  // Change is immediately visible to the other thread
    }
}
```

**When to use volatile:** flags/status indicators and simple read-write operations.

**When NOT to use volatile:** compound operations (like increment) or anything needing atomicity — use `synchronized` or atomic classes instead.

---

## String Handling

### Q26: What is the difference between String, StringBuilder, and StringBuffer?

**Answer:**

**String:**
- Immutable (cannot be modified)
- Thread-safe
- String pool for optimization
- Slower for frequent modifications
- Used when string doesn't change

**StringBuilder:**
- Mutable (can be modified)
- Not thread-safe
- Faster than StringBuffer
- Introduced in Java 5
- Used in single-threaded environments

**StringBuffer:**
- Mutable (can be modified)
- Thread-safe (synchronized methods)
- Slower than StringBuilder
- Used in multi-threaded environments

**Performance Comparison:**

| Aspect | String | StringBuilder | StringBuffer |
|--------|---------|---------------|---------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes | No | Yes |
| Performance | Slow (modifications) | Fast | Medium |
| Memory | Creates new objects | Modifies existing | Modifies existing |

**Example:**
```java
// String - Creates new objects for each modification
String str = "Hello";
str = str + " World";  // Creates new String object
str = str + "!";       // Creates another new String object

// StringBuilder - Modifies existing object
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");  // Modifies existing StringBuilder
sb.append("!");        // Modifies existing StringBuilder
String result = sb.toString();

// StringBuffer - Thread-safe version
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World");  // Thread-safe append
sbf.append("!");
```

### Q27: What is String interning in Java?

**Answer:** String interning is a process of storing only one copy of each distinct String value in the String pool, which is a special memory area.

**Key Points:**
- String literals are automatically interned
- `intern()` method can explicitly intern strings
- Interned strings are shared across the application
- Helps reduce memory usage
- `==` operator works for interned strings

**Example:**
```java
public class StringInterning {
    public static void main(String[] args) {
        String str1 = "Hello";              // Interned (literal)
        String str2 = "Hello";              // Same interned string
        String str3 = new String("Hello");   // New string object
        String str4 = str3.intern();        // Explicitly interned

        System.out.println(str1 == str2);  // true - same interned string
        System.out.println(str1 == str3);  // false - different objects
        System.out.println(str1 == str4);  // true - same interned string
        System.out.println(str1.equals(str3)); // true - same content

        // String pool benefits
        String greeting = "Hello, World!";
        String another = "Hello, World!";
        System.out.println(greeting == another);  // true - shared in pool
    }
}
```

### Q28: What are the important String methods?

**Answer:**

**Length and Character Operations:**
```java
String str = "Hello World";
str.length();           // 11
str.charAt(0);          // 'H'
str.indexOf('o');        // 4
str.lastIndexOf('o');   // 7
str.contains("World");  // true
str.startsWith("Hello"); // true
str.endsWith("World");   // true
```

**Substring and Manipulation:**
```java
String str = "Hello World";
str.substring(6);            // "World"
str.substring(0, 5);         // "Hello"
str.replace('o', 'a');      // "Hella Warld"
str.replace("World", "Java"); // "Hello Java"
str.toLowerCase();          // "hello world"
str.toUpperCase();          // "HELLO WORLD"
str.trim();                 // Removes whitespace
```

**Splitting and Joining:**
```java
String str = "apple,banana,orange";
String[] fruits = str.split(",");  // ["apple", "banana", "orange"]
String joined = String.join("-", fruits); // "apple-banana-orange"
```

**Comparison:**
```java
String str1 = "Hello";
String str2 = "hello";
str1.equals(str2);        // false (case-sensitive)
str1.equalsIgnoreCase(str2); // true (case-insensitive)
str1.compareTo(str2);      // negative number (str1 < str2)
```

**Format:**
```java
String name = "John";
int age = 25;
String formatted = String.format("Name: %s, Age: %d", name, age);
// "Name: John, Age: 25"
```

---

## Java 8+ Features

### Q29: What are the new features introduced in Java 8?

**Answer:**

**Lambda Expressions:**
- Anonymous functions
- Enable functional programming
- Concise syntax

```java
// Before Java 8
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello");
    }
};

// Java 8+
Runnable r = () -> System.out.println("Hello");
```

**Stream API:**
- Functional-style operations on collections
- Declarative programming
- Lazy evaluation

```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

// Filter and map
names.stream()
     .filter(name -> name.length() > 3)
     .map(String::toUpperCase)
     .forEach(System.out::println);
```

**Method References:**
- Shortcut for lambda expressions
- Refer to existing methods

```java
// Method reference
names.forEach(System.out::println);

// Equivalent lambda
names.forEach(name -> System.out.println(name));
```

**Functional Interfaces:**
- Interfaces with a single abstract method, marked `@FunctionalInterface`
- Examples: Predicate, Function, Consumer, Supplier (covered in detail in Q30)

**Optional:**
- Container for optional values
- Better alternative to null checks
- Prevents NullPointerException

```java
Optional<String> optional = Optional.of("Hello");

optional.ifPresent(value -> System.out.println(value));
String result = optional.orElse("Default");
```

**Default Methods in Interfaces:**
- Interface methods with default implementation
- Enables interface evolution

```java
interface MyInterface {
    default void defaultMethod() {
        System.out.println("Default implementation");
    }
}
```

**Date and Time API (java.time):**
- Modern date and time handling
- Immutable and thread-safe
- Better than old Date/Calendar classes

```java
LocalDate date = LocalDate.now();
LocalTime time = LocalTime.now();
LocalDateTime dateTime = LocalDateTime.now();
ZonedDateTime zonedDateTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
```

**CompletableFuture:** asynchronous programming with chainable operations (an improved alternative to `Future`). Senior-level topic — be aware it exists for async pipelines.

### Q30: What are functional interfaces in Java 8?

**Answer:** Functional interfaces are interfaces that contain exactly one abstract method (SAM - Single Abstract Method). They can have any number of default and static methods.

**Built-in Functional Interfaces:**

**Predicate<T>:**
```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}

// Usage
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4));  // true
System.out.println(isEven.test(3));  // false

Predicate<String> isEmpty = String::isEmpty;
System.out.println(isEmpty.test(""));   // true
System.out.println(isEmpty.test("hi")); // false
```

**Function<T, R>:**
```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}

// Usage
Function<String, Integer> stringLength = String::length;
System.out.println(stringLength.apply("Hello"));  // 5

Function<Integer, Integer> square = n -> n * n;
System.out.println(square.apply(5));  // 25

// Function chaining
Function<Integer, Integer> addThenSquare = square.compose(n -> n + 1);
System.out.println(addThenSquare.apply(3));  // 16 ((3+1)^2)
```

**Consumer<T>:**
```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}

// Usage
Consumer<String> printer = System.out::println;
printer.accept("Hello");  // Hello

Consumer<List<Integer>> listPrinter = list ->
    list.forEach(System.out::println);

listPrinter.accept(Arrays.asList(1, 2, 3));
```

**Supplier<T>:**
```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}

// Usage
Supplier<String> stringSupplier = () -> "Hello World";
System.out.println(stringSupplier.get());  // Hello World

Supplier<Double> randomSupplier = Math::random;
System.out.println(randomSupplier.get());  // Random number
```

**Custom Functional Interface:**
```java
@FunctionalInterface
interface MathOperation {
    int operate(int a, int b);
}

public class FunctionalInterfaceExample {
    public static void main(String[] args) {
        MathOperation add = (a, b) -> a + b;
        MathOperation subtract = (a, b) -> a - b;
        MathOperation multiply = (a, b) -> a * b;

        System.out.println(add.operate(5, 3));       // 8
        System.out.println(subtract.operate(5, 3));   // 2
        System.out.println(multiply.operate(5, 3));  // 15
    }
}
```

---

## JVM Internals

### Q31: How does garbage collection work in Java?

**Answer:** Garbage collection is the process of automatically reclaiming memory by destroying objects that are no longer reachable.

**Key Concepts:**

**Reachability:**
- An object is reachable if it can be accessed from any live thread
- Objects that are not reachable are eligible for garbage collection

**Generational Hypothesis:**
- Most objects die young
- Objects that survive multiple GC cycles tend to live longer

**Generations:**
- **Young Generation**: New objects
  - **Eden Space**: Where new objects are allocated
  - **Survivor Spaces (S0, S1)**: Objects that survive Eden GC

- **Old Generation**: Long-lived objects
  - Objects that survive multiple young generation GC cycles

- **Metaspace**: Class metadata (replaced PermGen in Java 8+)

**GC Algorithms (awareness level):**
- **Serial GC**: single-threaded, good for small apps, causes pauses.
- **Parallel GC**: multi-threaded young generation collection (higher throughput).
- **G1 GC**: modern default, divides heap into regions and collects the ones with most garbage first; good for large heaps.
- **ZGC**: low-latency collector for very large heaps with sub-millisecond pauses (specialist use).

**Example:**
```java
// Explicitly suggest garbage collection (not recommended)
System.gc();

// Memory analysis
Runtime runtime = Runtime.getRuntime();
System.out.println("Max Memory: " + runtime.maxMemory() / (1024 * 1024) + " MB");
System.out.println("Used Memory: " +
    (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024) + " MB");
```

### Q32: What is the Java Memory Model?

**Answer:** The Java Memory Model (JMM) defines how threads interact through memory and what behaviors are guaranteed.

**Key Concepts:**

**Heap Memory:**
- Shared by all threads
- Stores objects and arrays
- Managed by garbage collector
- Divided into Young, Old, and Metaspace

**Stack Memory:**
- Private to each thread
- Stores method calls, local variables, and references
- Grows and shrinks with method calls
- Automatically managed

**Other runtime areas:** Method Area (class data, static variables, method code), PC Registers (program counter per thread), and the Native Method Stack (for native method calls).

**Example:**
```java
public class MemoryModelExample {
    // Static variables - stored in Method Area
    private static int staticVar = 10;

    // Instance variables - stored in Heap
    private int instanceVar;

    public void method() {
        // Local variables - stored in Stack
        int localVar = 5;

        // Object reference in Stack, object in Heap
        String str = new String("Hello");
    }
}
```

### Q33: What is class loading in Java?

**Answer:** Class loading is the process of loading class files into JVM memory.

**Class Loading Process:**

1. **Loading**: Read class file and create Class object
2. **Linking**: Verify, prepare, and resolve the class
   - **Verification**: Ensure class file is valid
   - **Preparation**: Allocate memory for static fields
   - **Resolution**: Replace symbolic references with direct references
3. **Initialization**: Execute static initializers and assignments

**Class Loaders:**

**Bootstrap ClassLoader:**
- Loads core Java classes (java.lang, java.util)
- Implemented in native code
- Highest priority

**Extension/Platform ClassLoader:**
- Loads extension classes
- Located in jre/lib/ext directory

**Application/System ClassLoader:**
- Loads application classes
- Loads classes from CLASSPATH
- Default class loader

**Class Loading Delegation:**
- Child class loader delegates to parent before loading
- Ensures unique class instances
- Prevents class loading conflicts

**Example:**
```java
// Inspect the class loader hierarchy and load a class dynamically
ClassLoader cl = MyClass.class.getClassLoader();
System.out.println(cl);              // Application/System ClassLoader
System.out.println(cl.getParent());  // Platform ClassLoader

Class<?> clazz = Class.forName("java.lang.String");
```

---

## Design Patterns

### Q34: What are common design patterns in Java?

**Answer:**

**1. Singleton Pattern:**
- Ensures only one instance of a class
- Private constructor, static instance
- Thread-safe implementation

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {
        // Private constructor
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**2. Factory Pattern:**
- Creates objects without exposing creation logic
- Uses a common interface for object creation

```java
interface Shape {
    void draw();
}

class Circle implements Shape {
    public void draw() {
        System.out.println("Drawing Circle");
    }
}

class Rectangle implements Shape {
    public void draw() {
        System.out.println("Drawing Rectangle");
    }
}

class ShapeFactory {
    public static Shape getShape(String shapeType) {
        if (shapeType.equalsIgnoreCase("CIRCLE")) {
            return new Circle();
        } else if (shapeType.equalsIgnoreCase("RECTANGLE")) {
            return new Rectangle();
        }
        return null;
    }
}

// Usage
Shape shape = ShapeFactory.getShape("CIRCLE");
shape.draw();
```

**3. Observer Pattern:**
- One-to-many dependency
- When one object changes state, all dependents are notified

```java
import java.util.ArrayList;
import java.util.List;

interface Observer {
    void update(String message);
}

class Subject {
    private List<Observer> observers = new ArrayList<>();

    public void attach(Observer observer) {
        observers.add(observer);
    }

    public void notifyObservers(String message) {
        for (Observer observer : observers) {
            observer.update(message);
        }
    }
}

class ConcreteObserver implements Observer {
    private String name;
    public ConcreteObserver(String name) { this.name = name; }

    @Override
    public void update(String message) {
        System.out.println(name + " received: " + message);
    }
}

// Usage
Subject subject = new Subject();
subject.attach(new ConcreteObserver("Observer 1"));
subject.notifyObservers("Hello Observers!");
```

**4. Strategy Pattern:**
- Defines a family of algorithms
- Makes algorithms interchangeable

```java
interface PaymentStrategy {
    void pay(int amount);
}

class CreditCardPayment implements PaymentStrategy {
    public void pay(int amount) { System.out.println("Paid " + amount + " by Credit Card"); }
}

class PayPalPayment implements PaymentStrategy {
    public void pay(int amount) { System.out.println("Paid " + amount + " by PayPal"); }
}

class ShoppingCart {
    private PaymentStrategy paymentStrategy;

    public void setPaymentStrategy(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    public void checkout(int amount) {
        paymentStrategy.pay(amount);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.setPaymentStrategy(new CreditCardPayment());
cart.checkout(100);
```

### Q35: What is the Singleton pattern and how do you implement it?

**Answer:** The Singleton pattern ensures a class has only one instance and provides a global point of access to it.

**Thread-safe Singleton Implementations:**

Besides the eager (`private static final` field) and the double-checked locking approach shown in Q34, the two preferred forms are:

**1. Static Inner Class (Bill Pugh Solution):**
```java
public class StaticInnerClassSingleton {
    private StaticInnerClassSingleton() {}

    private static class SingletonHolder {
        private static final StaticInnerClassSingleton INSTANCE =
            new StaticInnerClassSingleton();
    }

    public static StaticInnerClassSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

**2. Enum Singleton (simplest, serialization-safe):**
```java
public enum EnumSingleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("Singleton instance is working");
    }
}

// Usage
EnumSingleton.INSTANCE.doSomething();
```

**Comparison:**

| Implementation | Thread Safety | Lazy Loading | Serialization Safety |
|---------------|---------------|--------------|---------------------|
| Eager Initialization | Yes | No | Needs work |
| Double-Checked Locking | Yes | Yes | Needs work |
| Static Inner Class | Yes | Yes | Needs work |
| Enum | Yes | No | Yes (built-in) |

---

## Common Coding Questions

### Q36: How do you find the second largest number in an array?

**Answer:**

```java
public class SecondLargest {
    public static int findSecondLargest(int[] arr) {
        if (arr.length < 2) {
            throw new IllegalArgumentException("Array must have at least 2 elements");
        }

        int largest = Integer.MIN_VALUE;
        int secondLargest = Integer.MIN_VALUE;

        for (int num : arr) {
            if (num > largest) {
                secondLargest = largest;
                largest = num;
            } else if (num > secondLargest && num != largest) {
                secondLargest = num;
            }
        }

        if (secondLargest == Integer.MIN_VALUE) {
            throw new IllegalArgumentException("No second largest element found");
        }

        return secondLargest;
    }

    public static void main(String[] args) {
        int[] arr = {12, 35, 1, 10, 34, 1};
        System.out.println("Second largest: " + findSecondLargest(arr));  // 34
    }
}
```

### Q37: How do you check if a string is a palindrome?

**Answer:**

```java
public class PalindromeChecker {
    // Method 1: Using StringBuilder
    public static boolean isPalindromeUsingBuilder(String str) {
        String reversed = new StringBuilder(str).reverse().toString();
        return str.equals(reversed);
    }

    // Method 2: Two-pointer approach (more efficient)
    public static boolean isPalindrome(String str) {
        int left = 0;
        int right = str.length() - 1;

        while (left < right) {
            if (str.charAt(left) != str.charAt(right)) {
                return false;
            }
            left++;
            right--;
        }
        return true;
    }

    // Method 3: Ignoring case and non-alphanumeric characters
    public static boolean isPalindromeIgnoreNonAlphanumeric(String str) {
        String cleaned = str.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
        return isPalindrome(cleaned);
    }

    public static void main(String[] args) {
        System.out.println(isPalindrome("radar"));  // true
        System.out.println(isPalindrome("hello"));  // false
        System.out.println(isPalindromeIgnoreNonAlphanumeric("A man, a plan, a canal: Panama"));  // true
    }
}
```

### Q38: How do you reverse a string?

**Answer:**

```java
public class StringReverser {
    // Method 1: Using StringBuilder
    public static String reverseUsingBuilder(String str) {
        return new StringBuilder(str).reverse().toString();
    }

    // Method 2: Using character array
    public static String reverseUsingCharArray(String str) {
        char[] chars = str.toCharArray();
        int left = 0;
        int right = chars.length - 1;

        while (left < right) {
            char temp = chars[left];
            chars[left] = chars[right];
            chars[right] = temp;
            left++;
            right--;
        }

        return new String(chars);
    }

    public static void main(String[] args) {
        String str = "Hello World";

        System.out.println(reverseUsingBuilder(str));
        System.out.println(reverseUsingCharArray(str));
    }
}
```

### Q39: How do you find duplicates in an array?

**Answer:**

```java
import java.util.*;

public class DuplicateFinder {
    // Method 1: Using HashSet
    public static <T> Set<T> findDuplicatesUsingHashSet(T[] array) {
        Set<T> duplicates = new HashSet<>();
        Set<T> seen = new HashSet<>();

        for (T element : array) {
            if (!seen.add(element)) {
                duplicates.add(element);
            }
        }

        return duplicates;
    }

    // Method 2: Using Map for counting
    public static <T> Map<T, Integer> countOccurrences(T[] array) {
        Map<T, Integer> occurrences = new HashMap<>();

        for (T element : array) {
            occurrences.put(element, occurrences.getOrDefault(element, 0) + 1);
        }

        return occurrences;
    }

    // Method 3: Using Streams (Java 8+)
    public static <T> Set<T> findDuplicatesUsingStreams(T[] array) {
        Set<T> seen = new HashSet<>();
        return Arrays.stream(array)
                     .filter(e -> !seen.add(e))
                     .collect(Collectors.toSet());
    }

    public static void main(String[] args) {
        Integer[] numbers = {1, 2, 3, 4, 5, 2, 3, 6, 7, 3};

        System.out.println("Duplicates: " + findDuplicatesUsingHashSet(numbers));
        System.out.println("Occurrences: " + countOccurrences(numbers));
        System.out.println("Via streams: " + findDuplicatesUsingStreams(numbers));
    }
}
```

### Q40: How do you sort an array?

**Answer:**

```java
import java.util.*;

public class ArraySorter {
    // Method 1: Using Arrays.sort (primitive types)
    public static void sortPrimitiveArray(int[] arr) {
        Arrays.sort(arr);
    }

    // Method 2: Using Arrays.sort (objects)
    public static void sortObjectArray(String[] arr) {
        Arrays.sort(arr);  // Uses natural ordering
    }

    // Method 3: Using custom comparator
    public static void sortWithComparator(String[] arr) {
        Arrays.sort(arr, Comparator.reverseOrder());  // Descending order
    }

    // Method 4: Using Collections.sort (for Lists)
    public static void sortList(List<Integer> list) {
        Collections.sort(list);  // Ascending order
    }

    public static void main(String[] args) {
        // Primitive array
        int[] numbers = {5, 2, 8, 1, 9, 3};
        sortPrimitiveArray(numbers);
        System.out.println("Sorted numbers: " + Arrays.toString(numbers));

        // Object array
        String[] names = {"John", "Alice", "Bob", "David"};
        sortObjectArray(names);
        System.out.println("Sorted names: " + Arrays.toString(names));

        // List sorting
        List<Integer> list = new ArrayList<>(Arrays.asList(5, 2, 8, 1, 9));
        sortList(list);
        System.out.println("Sorted list: " + list);
    }
}
```

---

## Streams API

### Q41: What is the difference between Collection and Stream?

**Answer:**

**Collection:**
- Represents a group of objects stored in memory
- Can be modified (add, remove, clear)
- Can be traversed multiple times
- Eager evaluation
- Supports iteration

**Stream:**
- Represents a sequence of elements for computation
- Cannot be modified
- Can be traversed only once
- Lazy evaluation
- Supports functional operations

**Example:**
```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

// Collection - can be modified
names.add("David");
names.remove("John");
for (String name : names) {
    System.out.println(name);
}

// Stream - functional operations
names.stream()
     .filter(name -> name.startsWith("J"))
     .map(String::toUpperCase)
     .forEach(System.out::println);

// Stream cannot be reused
Stream<String> nameStream = names.stream();
nameStream.forEach(System.out::println);
// nameStream.forEach(System.out::println);  // IllegalStateException!
```

### Q42: What are intermediate and terminal operations in Streams?

**Answer:**

**Intermediate Operations:**
- Return a new Stream
- Lazy evaluation (executed only when terminal operation is called)
- Can be chained
- Examples: filter, map, sorted, distinct, limit, skip

**Terminal Operations:**
- Return a result or side-effect
- Trigger execution of intermediate operations
- Cannot be chained after terminal operation
- Examples: forEach, collect, reduce, count, anyMatch, allMatch

**Example:**
```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice", "Bob");

// Intermediate operations (lazy)
long count = names.stream()
                  .filter(name -> name.length() > 3)  // Intermediate
                  .distinct()                         // Intermediate
                  .sorted()                           // Intermediate
                  .count();                           // Terminal

System.out.println("Count: " + count);  // 4

// Different terminal operations
List<String> filteredList = names.stream()
                                 .filter(name -> name.startsWith("J"))
                                 .collect(Collectors.toList());  // Terminal

System.out.println("Filtered: " + filteredList);

String namesString = names.stream()
                          .collect(Collectors.joining(", "));  // Terminal
System.out.println("Names: " + namesString);
```

### Q43: What are common Stream operations?

**Answer:**

**Filter:**
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

List<Integer> evenNumbers = numbers.stream()
                                   .filter(n -> n % 2 == 0)
                                   .collect(Collectors.toList());

// [2, 4, 6, 8, 10]
```

**Map:**
```java
List<String> names = Arrays.asList("john", "jane", "bob");

List<String> upperCaseNames = names.stream()
                                   .map(String::toUpperCase)
                                   .collect(Collectors.toList());

// [JOHN, JANE, BOB]
```

**FlatMap:**
```java
List<List<Integer>> nestedList = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5, 6),
    Arrays.asList(7, 8, 9)
);

List<Integer> flattened = nestedList.stream()
                                   .flatMap(List::stream)
                                   .collect(Collectors.toList());

// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

**Reduce:**
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

int sum = numbers.stream()
                 .reduce(0, (a, b) -> a + b);

// 15

Optional<Integer> max = numbers.stream()
                               .max(Integer::compare);

// Optional[5]
```

**Collect:**
```java
List<String> names = Arrays.asList("John", "Jane", "Bob", "Alice");

// To List
List<String> nameList = names.stream()
                             .collect(Collectors.toList());

// To Set
Set<String> nameSet = names.stream()
                          .collect(Collectors.toSet());

// To Map
Map<String, Integer> nameMap = names.stream()
                                   .collect(Collectors.toMap(
                                       name -> name,
                                       String::length
                                   ));

// Joining
String joined = names.stream()
                    .collect(Collectors.joining(", "));

// Grouping
Map<Integer, List<String>> groupedByLength = names.stream()
                                                   .collect(Collectors.groupingBy(String::length));
```

**Sorted:**
```java
List<Integer> numbers = Arrays.asList(5, 2, 8, 1, 9, 3);

List<Integer> sorted = numbers.stream()
                               .sorted()                          // natural order
                               .collect(Collectors.toList());     // [1, 2, 3, 5, 8, 9]
// Use .sorted(Comparator.reverseOrder()) for descending order.
```

---

## Java Best Practices

### Q44: What are Java coding best practices?

**Answer:**

**1. Naming Conventions:**
```java
// Classes: PascalCase
public class UserService { }

// Interfaces: PascalCase (often start with 'I')
public interface UserRepository { }

// Methods: camelCase
public void getUserById() { }
public boolean isValid() { }

// Variables: camelCase
private String userName;
private int maxRetryCount;

// Constants: UPPER_SNAKE_CASE
public static final int MAX_RETRIES = 3;
public static final String DEFAULT_ENCODING = "UTF-8";

// Packages: lowercase
package com.example.service;
```

**2. Proper Exception Handling:**
```java
// Good - Specific exception handling
try {
    FileReader reader = new FileReader("file.txt");
    // Process file
} catch (FileNotFoundException e) {
    logger.error("File not found: " + e.getMessage());
    throw new BusinessException("File not found", e);
} catch (IOException e) {
    logger.error("IO error: " + e.getMessage());
    throw new BusinessException("Error reading file", e);
}

// Bad - Catching generic exception
try {
    // Some code
} catch (Exception e) {
    // Don't catch Exception directly
}
```

**3. Use constants instead of magic numbers:**
```java
// Bad
if (age < 18) {
    throw new IllegalArgumentException("Under age");
}

// Good
private static final int MINIMUM_AGE = 18;
if (age < MINIMUM_AGE) {
    throw new IllegalArgumentException("Under age");
}
```

**4. Immutability:**
```java
// Good - Immutable class
public final class ImmutablePerson {
    private final String name;
    private final int age;

    public ImmutablePerson(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public int getAge() {
        return age;
    }
}
```

**5. Use StringBuilder for string concatenation in loops:**
```java
// Bad
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // Creates new String objects
}

// Good
StringBuilder result = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    result.append(i);
}
String finalResult = result.toString();
```

**6. Use try-with-resources:**
```java
// Good - Automatic resource management
try (FileReader reader = new FileReader("file.txt");
     BufferedReader br = new BufferedReader(reader)) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
} catch (IOException e) {
    e.printStackTrace();
}
// Resources are closed automatically — no manual finally/close() needed.
```

### Q45: What are common Java performance optimization techniques?

**Answer:**

**1-3. Basics (covered earlier):** use `StringBuilder` for string concatenation in loops, prefer primitives over wrappers to avoid boxing, and pick the right collection for the access pattern (ArrayList for random access, LinkedList for end insertions, HashSet/HashMap for lookups).

**4. Lazy initialization:**
```java
public class LazyInitialization {
    private ExpensiveObject expensiveObject;

    public ExpensiveObject getExpensiveObject() {
        if (expensiveObject == null) {
            expensiveObject = new ExpensiveObject();
        }
        return expensiveObject;
    }
}
```

**5. Stream vs Loop:**
```java
// For simple operations, loops may be faster
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Loop (faster for simple cases)
int sum1 = 0;
for (int num : numbers) {
    sum1 += num;
}

// Stream (more readable, slightly slower)
int sum2 = numbers.stream()
                 .mapToInt(Integer::intValue)
                 .sum();
```

**6. Avoid unnecessary object creation:**
```java
// Bad - Creates objects in loop
for (int i = 0; i < 1000; i++) {
    String date = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
}

// Good - Reuse objects
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
for (int i = 0; i < 1000; i++) {
    String date = sdf.format(new Date());
}
```

---

## Quick Reference Summary

### Important Keywords
- `final`: Cannot be changed
- `static`: Belongs to class, not instance
- `abstract`: Cannot be instantiated
- `synchronized`: Thread safety
- `volatile`: Visibility across threads

### Collections
- **List**: ordered, allows duplicates (ArrayList, LinkedList)
- **Set**: unique elements (HashSet, TreeSet)
- **Map**: key-value pairs (HashMap, TreeMap)

### Exception Hierarchy
```
Throwable
├── Error (Unrecoverable)
└── Exception (Recoverable)
    ├── Checked (Must handle)
    └── Unchecked (Runtime)
```

**Interview tips:** focus on fundamentals over memorization, back answers with code examples, and explain trade-offs (when to use which approach).
