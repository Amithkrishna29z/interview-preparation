# Java Exception Handling Interview Questions & Study Guide

## Overview

Exception handling is how Java deals with errors that happen while a program runs — like a missing file, a bad network call, or a `null` value. It is one of the most commonly tested fundamentals in Java backend interviews. Expect questions on checked vs unchecked exceptions, the `try/catch/finally` flow, `throw` vs `throws`, custom exceptions, and tricky output-prediction puzzles involving `finally` and `return`.

This guide is written for junior developers — every concept comes with a plain-English analogy, code with line-by-line comments, and the kind of follow-up questions interviewers actually ask.

---

## Table of Contents

1. [What Is an Exception?](#what-is-an-exception)
2. [The Throwable Hierarchy](#the-throwable-hierarchy)
3. [Checked vs Unchecked Exceptions](#checked-vs-unchecked-exceptions)
4. [try / catch / finally Mechanics](#try--catch--finally-mechanics)
5. [Multi-Catch and Catch Order](#multi-catch-and-catch-order)
6. [finally and return Interactions](#finally-and-return-interactions)
7. [try-with-resources and AutoCloseable](#try-with-resources-and-autocloseable)
8. [throw vs throws and Exception Propagation](#throw-vs-throws-and-exception-propagation)
9. [Creating Custom Exceptions](#creating-custom-exceptions)
10. [Exception Chaining and Wrapping](#exception-chaining-and-wrapping)
11. [Best Practices](#best-practices)
12. [Common Pitfalls](#common-pitfalls)
13. [Connecting to Spring Backend](#connecting-to-spring-backend)
14. [Common Interview Questions](#common-interview-questions)
15. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What Is an Exception?

An **exception** is an object that Java creates when something goes wrong during execution. Instead of letting the program crash silently or continue with garbage data, Java "throws" this object and looks for code that can "catch" and handle it.

**Think of it like a fire alarm in a building:**

- Something unexpected happens (a fire / a divide-by-zero).
- An alarm goes off (an exception object is **thrown**).
- The alarm travels up through the floors looking for someone who can respond (the exception **propagates** up the call stack).
- A trained responder handles it (a `catch` block), or if nobody does, the whole building is evacuated (the program **crashes** with a stack trace).

```java
public class Example {
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3};   // an array with 3 elements (indexes 0, 1, 2)
        System.out.println(numbers[5]); // index 5 does NOT exist
        // Java throws ArrayIndexOutOfBoundsException here
        // Since nothing catches it, the program crashes and prints a stack trace
        System.out.println("This line never runs"); // unreachable after the crash
    }
}
```

When an exception is thrown and not handled, the JVM prints a **stack trace** — a report showing exactly which method threw the error and the chain of method calls that led there. Reading stack traces is a core debugging skill.

```
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3
    at Example.main(Example.java:4)   ← the exact file and line number that failed
```

---

## The Throwable Hierarchy

Every error in Java is an object, and all of them descend from a single root class called `Throwable`. Understanding this family tree is essential — interviewers love to ask "where does `RuntimeException` sit in the hierarchy?"

**Think of it like a family tree of problems:**

- `Throwable` = the great-grandparent of all problems.
- `Error` = catastrophic problems you should NOT try to recover from (the building is collapsing).
- `Exception` = problems your program can reasonably handle (a door is locked — find another way).
- `RuntimeException` = a special branch of `Exception` for programming bugs (you forgot your keys).

```
                    ┌─────────────┐
                    │  Throwable  │   ← root of ALL errors/exceptions (can be thrown & caught)
                    └─────────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
      ┌──────────┐                ┌────────────┐
      │  Error   │                │ Exception  │
      │ (serious,│                │ (recoverable)
      │  do NOT  │                └────────────┘
      │  catch)  │                      │
      └──────────┘            ┌─────────┴──────────┐
            │                 │                    │
   OutOfMemoryError     RuntimeException     (everything else =
   StackOverflowError   (UNCHECKED)           CHECKED exceptions)
   VirtualMachineError        │                    │
                      ┌───────┴────────┐     IOException
                NullPointerException   │     SQLException
                ArithmeticException    │     FileNotFoundException
                IllegalArgumentException     ClassNotFoundException
                ArrayIndexOutOfBounds
```

| Branch | Examples | Should you catch it? |
|---|---|---|
| `Error` | `OutOfMemoryError`, `StackOverflowError` | **No** — the JVM is in serious trouble; recovery is usually impossible |
| `Exception` (checked) | `IOException`, `SQLException` | **Yes** — these are expected failures you plan for |
| `RuntimeException` (unchecked) | `NullPointerException`, `IllegalArgumentException` | Usually **fix the bug** rather than catch it |

> **Interview Tip**: `RuntimeException` is a subclass of `Exception`, which is a subclass of `Throwable`. `Error` is a separate sibling of `Exception` (both extend `Throwable`). Memorize: **Throwable → Error / Exception → RuntimeException**.

---

## Checked vs Unchecked Exceptions

This is the single most asked exception-handling question. The distinction controls whether the **compiler forces you** to deal with the exception.

**Think of it like two kinds of warnings on a road trip:**

- **Checked exception** = a posted road sign ("Bridge out ahead — plan a detour"). The law (compiler) requires you to acknowledge it before you drive.
- **Unchecked exception** = a sudden pothole. Nobody warned you in advance; it usually means you should have driven more carefully (fixed your code).

### Checked Exceptions

A **checked exception** is checked by the compiler at compile time. If a method can throw one, you MUST either catch it or declare it with `throws`. If you don't, your code won't compile.

```java
import java.io.FileReader;
import java.io.IOException;

public class FileExample {
    public void readFile() throws IOException {  // we DECLARE that this method may throw IOException
        FileReader reader = new FileReader("data.txt"); // may throw FileNotFoundException (a checked exception)
        reader.read();                                   // may throw IOException (checked)
        reader.close();
    }
    // If we removed "throws IOException", the code would NOT compile —
    // the compiler forces us to handle these expected failures.
}
```

### Unchecked Exceptions

An **unchecked exception** (any `RuntimeException` or its subclasses, plus `Error`) is NOT checked by the compiler. You are not forced to catch or declare it. These usually signal **programming bugs**.

```java
public class MathExample {
    public int divide(int a, int b) {  // notice: NO "throws" needed
        return a / b;                  // if b == 0, throws ArithmeticException (unchecked)
        // The compiler does NOT force us to handle this.
        // The real fix is to validate b != 0 before dividing.
    }
}
```

### Side-by-Side Comparison

| Feature | Checked Exception | Unchecked Exception |
|---|---|---|
| Parent class | `Exception` (but NOT `RuntimeException`) | `RuntimeException` or `Error` |
| Checked at compile time? | **Yes** | No |
| Must catch or declare? | **Yes** (or won't compile) | No |
| Typical cause | External/expected failure (file, network, DB) | Programming bug (null, bad index, bad argument) |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |
| Recommended action | Handle it (retry, fallback, wrap) | Fix the underlying code |

> **Interview Tip**: A common trick question — "Is `NullPointerException` checked or unchecked?" Answer: **unchecked**, because it extends `RuntimeException`. The compiler never forces you to catch an NPE.

---

## try / catch / finally Mechanics

The `try/catch/finally` block is the core tool for handling exceptions.

**Think of it like a science experiment:**

- `try` = perform the risky experiment.
- `catch` = what to do if the experiment explodes (handle a specific failure).
- `finally` = clean the lab afterward — this happens whether or not the experiment exploded.

```java
public class TryCatchExample {
    public void process() {
        try {
            // TRY: the risky code that might throw an exception
            int result = 10 / 0;          // throws ArithmeticException immediately
            System.out.println(result);   // SKIPPED — we never reach here after the throw
        } catch (ArithmeticException e) {
            // CATCH: runs ONLY if a matching exception was thrown in the try block
            System.out.println("Cannot divide by zero: " + e.getMessage());
        } finally {
            // FINALLY: runs ALWAYS — whether the try succeeded, threw, or was caught
            System.out.println("Cleanup done."); // perfect for closing resources
        }
        System.out.println("Program continues."); // runs because the exception was caught
    }
}
```

**Output:**
```
Cannot divide by zero: / by zero
Cleanup done.
Program continues.
```

### Rules to Remember

| Block | Required? | Runs when? |
|---|---|---|
| `try` | Yes | Always entered |
| `catch` | Optional* | Only when a matching exception is thrown |
| `finally` | Optional* | Almost always (see exceptions below) |

\* You must have **at least one** `catch` OR `finally` after a `try`. A bare `try` alone won't compile.

You can have a `try` with only `finally` (no `catch`) — useful when you want cleanup but want the exception to propagate:

```java
public void readData() throws IOException {
    FileReader reader = new FileReader("data.txt");
    try {
        reader.read();         // if this throws, the exception propagates UP...
    } finally {
        reader.close();        // ...but finally still closes the file first
    }
}
```

---

## Multi-Catch and Catch Order

### Multiple catch blocks

A single `try` can have several `catch` blocks, one per exception type. Java checks them **top to bottom** and runs the **first** one that matches.

```java
public void parseAndDivide(String input) {
    try {
        int number = Integer.parseInt(input); // may throw NumberFormatException
        int result = 100 / number;            // may throw ArithmeticException
        System.out.println(result);
    } catch (NumberFormatException e) {
        // handles bad input like "abc"
        System.out.println("Not a valid number: " + input);
    } catch (ArithmeticException e) {
        // handles division by zero
        System.out.println("Cannot divide by zero.");
    }
}
```

### Multi-catch (one block, multiple types)

Since Java 7, you can catch several exception types in **one** block using the pipe `|`. Use this when the handling logic is identical.

```java
public void load(String input) {
    try {
        riskyOperation(input);
    } catch (NumberFormatException | NullPointerException e) {
        // ONE block handles BOTH types — avoids duplicate code
        // Note: 'e' is implicitly final here; you cannot reassign it
        System.out.println("Bad input: " + e.getMessage());
    }
}
```

> **Rule**: In a multi-catch, the types must NOT be related by inheritance. `catch (IOException | FileNotFoundException e)` won't compile because `FileNotFoundException` is already an `IOException` — it would be redundant.

### Catch Order: Specific Before General

This is a classic compile-error trap. You must order `catch` blocks from **most specific to most general**. If a parent type comes first, it would catch everything and make the child block unreachable — the compiler rejects this.

```java
// WRONG — does NOT compile
try {
    doWork();
} catch (Exception e) {           // Exception is the PARENT — catches everything first
    System.out.println("general");
} catch (IOException e) {          // UNREACHABLE — IOException is a child of Exception
    System.out.println("specific"); // compiler error: "exception IOException has already been caught"
}

// CORRECT — specific child first, general parent last
try {
    doWork();
} catch (IOException e) {           // specific: handle file/network errors first
    System.out.println("specific");
} catch (Exception e) {            // general: catch-all fallback for anything else
    System.out.println("general");
}
```

> **Think of it like sorting mail**: You sort the most specific category first (bills), then the general bucket (everything else) last. If you dump everything into "general" first, nothing ever reaches the specific bins.

---

## finally and return Interactions

This is the **#1 output-prediction puzzle** in interviews. Two key facts:

1. `finally` runs even when `try` or `catch` has a `return` statement.
2. If `finally` itself has a `return`, it **overrides** any return/exception from `try` or `catch` (this is bad practice but commonly tested).

### finally runs even after return

```java
public int test() {
    try {
        return 1;        // Java evaluates this return value (1)...
    } finally {
        System.out.println("finally runs"); // ...but finally executes BEFORE the method actually returns
    }
    // Output: "finally runs", then the method returns 1
}
```

### finally with return overrides everything (avoid this!)

```java
public int test() {
    try {
        return 1;        // this return value is PREPARED...
    } finally {
        return 2;        // ...but finally's return WINS — method returns 2, not 1!
    }
    // Returns 2. The 'return 1' is completely discarded. This silently swallows exceptions too.
}
```

### finally can swallow exceptions (a dangerous gotcha)

```java
public int dangerous() {
    try {
        throw new RuntimeException("boom"); // an exception is thrown...
    } finally {
        return 42;        // ...but finally's return SWALLOWS the exception entirely!
    }
    // The RuntimeException vanishes; the method just returns 42. NEVER return from finally.
}
```

### When finally does NOT run

`finally` is "almost always" — here are the rare cases it is skipped:

| Scenario | Does finally run? |
|---|---|
| Normal completion of `try` | Yes |
| Exception thrown and caught | Yes |
| Exception thrown and NOT caught (propagates) | Yes (runs before propagating) |
| `return` inside `try` or `catch` | Yes |
| **`System.exit(0)` called in `try`** | **No** — JVM shuts down immediately |
| **JVM crashes / power loss / `kill -9`** | **No** |
| **Infinite loop or deadlock in `try`** | **No** — `try` never finishes |
| **Daemon thread, JVM exits** | **No** |

```java
public void willNotRunFinally() {
    try {
        System.out.println("in try");
        System.exit(0);   // JVM terminates RIGHT HERE — no cleanup, no finally
    } finally {
        System.out.println("this NEVER prints"); // skipped because the JVM is already gone
    }
}
```

> **Interview Tip**: The cleanest answer to "when does `finally` not run?" is: *"When the JVM stops running before `finally` is reached — `System.exit()`, a JVM crash, or the thread never completing the try block (infinite loop)."*

---

## try-with-resources and AutoCloseable

Before Java 7, closing resources (files, DB connections, streams) meant verbose `finally` blocks that were easy to get wrong. **try-with-resources** automates this.

**Think of it like a self-closing door:** You walk through, and no matter what happens inside the room, the door closes itself behind you automatically. You never forget to shut it.

Any class that implements the `AutoCloseable` interface (which has a single `close()` method) can be used in try-with-resources. Java calls `close()` for you automatically.

### The old way (verbose and error-prone)

```java
public void oldWay() throws IOException {
    FileReader reader = null;
    try {
        reader = new FileReader("data.txt");
        reader.read();
    } finally {
        if (reader != null) {   // must null-check manually
            reader.close();     // must remember to close; close() itself can throw
        }
    }
}
```

### The new way (try-with-resources)

```java
public void newWay() throws IOException {
    // Resource declared inside the try(...) — Java auto-closes it when the block ends
    try (FileReader reader = new FileReader("data.txt")) {
        reader.read();
        // No finally needed! Java calls reader.close() automatically,
        // even if an exception is thrown inside the block.
    }
}
```

### Multiple resources (closed in reverse order)

```java
public void copy() throws IOException {
    // Resources are closed in REVERSE order of declaration: writer first, then reader
    try (FileReader reader = new FileReader("in.txt");
         FileWriter writer = new FileWriter("out.txt")) {
        int c;
        while ((c = reader.read()) != -1) {
            writer.write(c);
        }
    } // writer.close() runs first, then reader.close() — both guaranteed
}
```

### Implementing AutoCloseable yourself

```java
public class DatabaseConnection implements AutoCloseable {
    public DatabaseConnection() {
        System.out.println("Connection opened");
    }

    public void query() {
        System.out.println("Running query");
    }

    @Override
    public void close() {  // Java calls this automatically at the end of try-with-resources
        System.out.println("Connection closed");
    }
}

// Usage:
try (DatabaseConnection conn = new DatabaseConnection()) {
    conn.query();
} // conn.close() runs here automatically
// Output: Connection opened → Running query → Connection closed
```

### Why try-with-resources beats finally

| Aspect | finally | try-with-resources |
|---|---|---|
| Boilerplate | High (null checks, nested try) | Minimal |
| Forgetting to close | Easy to forget | Impossible — automatic |
| Closing multiple resources | Manual, error-prone order | Automatic, reverse order |
| **Suppressed exceptions** | Lost — if `close()` throws, it hides the original | Preserved via `getSuppressed()` |

**The suppressed-exception problem**: With a manual `finally`, if your `try` throws exception A and then `close()` in `finally` throws exception B, exception A is **lost** — only B propagates. try-with-resources solves this: the original exception A propagates, and B is attached as a "suppressed" exception you can retrieve via `e.getSuppressed()`.

> **Interview Tip**: Always prefer try-with-resources for anything `Closeable`/`AutoCloseable` (files, streams, JDBC connections, sockets). It is shorter, safer, and preserves the original exception.

---

## throw vs throws and Exception Propagation

These two keywords look almost identical but do completely different things. Interviewers love this confusion.

| Keyword | Where it goes | What it does |
|---|---|---|
| `throw` | Inside a method body | **Actually throws** one exception object right now |
| `throws` | In the method signature | **Declares** that this method might throw certain exceptions (a warning to callers) |

```java
public class ValidationExample {

    // "throws" in the signature = a DECLARATION: "callers, beware, I may throw these"
    public void validateAge(int age) throws IllegalArgumentException {
        if (age < 0) {
            // "throw" = the ACTION: create and throw an exception object right now
            throw new IllegalArgumentException("Age cannot be negative: " + age);
        }
        System.out.println("Valid age: " + age);
    }
}
```

> Memory trick: **`throws` has an "s"** because it can list **s**everal exception types. **`throw`** throws exactly **one** object.

### Exception Propagation

When an exception is thrown and the current method does not catch it, Java **propagates** it up the call stack — it pops out of the current method and checks the caller, then the caller's caller, and so on, until either a `catch` handles it or it reaches `main` and crashes the program.

**Think of it like passing a hot potato up a chain of people:** Whoever is holding it either deals with it (catches it) or passes it to the person who handed it to them. If it reaches the top and nobody catches it, it falls on the floor (program crashes).

```java
public class PropagationExample {

    public static void main(String[] args) {
        try {
            methodA();           // calls A, which calls B, which throws
        } catch (RuntimeException e) {
            // The exception thrown deep in methodC bubbles ALL the way up to here
            System.out.println("Caught in main: " + e.getMessage());
        }
    }

    static void methodA() {
        methodB();               // A does not catch — exception passes through it
    }

    static void methodB() {
        methodC();               // B does not catch — exception passes through it
    }

    static void methodC() {
        throw new RuntimeException("Error in C"); // thrown here, propagates A ← B ← C up to main
    }
}
// Output: Caught in main: Error in C
// The stack trace shows the full path: main → methodA → methodB → methodC
```

---

## Creating Custom Exceptions

Sometimes the built-in exceptions don't describe your problem well. A `UserNotFoundException` is far clearer than a generic `RuntimeException`. Custom exceptions make your code self-documenting and let callers catch *your specific* error type.

**Think of it like custom labels on warning lights in a car:** A generic "Check Engine" light tells you little. A specific "Low Oil Pressure" light tells you exactly what's wrong and how to react.

### The key design decision: extend Exception or RuntimeException?

| Extend... | Makes it... | Use when... |
|---|---|---|
| `Exception` | **Checked** (caller forced to handle) | The caller can reasonably recover and you want to force them to deal with it |
| `RuntimeException` | **Unchecked** (caller not forced) | The error is a programming/business condition the caller usually can't recover from at that spot |

Modern style (and Spring) leans toward **unchecked** custom exceptions to avoid cluttering every method signature with `throws`.

### A custom unchecked exception (common in Spring apps)

```java
// Extends RuntimeException → UNCHECKED (no forced try/catch on callers)
public class UserNotFoundException extends RuntimeException {

    // Always provide a constructor that takes a message
    public UserNotFoundException(String message) {
        super(message); // pass the message up to RuntimeException so getMessage() works
    }

    // Also provide a constructor that takes a CAUSE — critical for exception chaining
    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause); // keeps the original exception attached (see chaining section)
    }
}
```

```java
// Usage
public User findUser(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException("No user with id: " + id));
    // Clear, specific, and easy for callers to catch by type
}
```

### A custom checked exception

```java
// Extends Exception → CHECKED (callers MUST catch or declare it)
public class InsufficientFundsException extends Exception {
    private final double shortfall; // you can add extra fields with useful context

    public InsufficientFundsException(String message, double shortfall) {
        super(message);
        this.shortfall = shortfall;
    }

    public double getShortfall() { // expose the extra data to handlers
        return shortfall;
    }
}
```

### Good custom-exception design rules

- Always include a constructor that accepts a **message** and one that accepts a **cause** (`Throwable`).
- Name it clearly and end with `Exception` (e.g., `OrderNotFoundException`).
- Add fields only when they give the handler useful context (an ID, an error code).
- Don't create a custom exception for every tiny case — reuse built-ins like `IllegalArgumentException` or `IllegalStateException` when they fit.

---

## Exception Chaining and Wrapping

When you catch a low-level exception and throw a different, higher-level one, you must **preserve the original cause**. Otherwise you lose the real reason for the failure — one of the most frustrating debugging experiences.

**Think of it like a doctor's referral note:** A specialist (high-level exception) should attach the original test results (the cause). If they throw away the lab report and just say "something's wrong," nobody can diagnose the real problem.

```java
public void loadConfig() {
    try {
        readFileFromDisk();          // throws a low-level IOException
    } catch (IOException e) {
        // WRAP the low-level IOException in a meaningful domain exception,
        // BUT pass 'e' as the cause so the original stack trace is preserved
        throw new ConfigLoadException("Failed to load app config", e);
        //                                                          ↑ the cause
    }
}
```

### getCause() — retrieving the original

```java
try {
    loadConfig();
} catch (ConfigLoadException e) {
    System.out.println("High-level: " + e.getMessage()); // "Failed to load app config"
    Throwable original = e.getCause();                   // gets back the original IOException
    System.out.println("Root cause: " + original.getMessage());
}
```

### The dangerous mistake: losing the cause

```java
// WRONG — the original IOException is thrown away. Stack trace points nowhere useful.
catch (IOException e) {
    throw new ConfigLoadException("Failed to load config"); // no 'e' passed → cause is LOST
}

// CORRECT — the cause is preserved
catch (IOException e) {
    throw new ConfigLoadException("Failed to load config", e); // 'e' chained as the cause
}
```

When you print a chained exception, Java shows the full chain with a helpful **"Caused by:"** section:

```
ConfigLoadException: Failed to load app config
    at MyApp.loadConfig(MyApp.java:10)
Caused by: java.io.IOException: data.txt (No such file or directory)   ← the real root cause
    at MyApp.readFileFromDisk(MyApp.java:20)
```

> **Interview Tip**: Always use the two-argument constructor (`message, cause`) or `initCause()` when re-throwing. "Wrapping" without the cause is a classic way to lose the stack trace and make production bugs un-debuggable.

---

## Best Practices

| Practice | Why |
|---|---|
| **Never swallow exceptions** (empty `catch {}`) | Errors vanish silently; bugs become invisible |
| **Catch specific exceptions**, not `Exception` broadly | Broad catches hide bugs you didn't intend to handle |
| **Never catch `Throwable` or `Error`** | `Error` means the JVM is dying (OOM, stack overflow) — catching it can hide fatal problems |
| **Log with the exception object**, not just the message | `log.error("msg", e)` keeps the full stack trace; `log.error(e.getMessage())` loses it |
| **Fail fast** — validate inputs early and throw immediately | Catch bad data at the boundary, not deep inside business logic |
| **Preserve the cause** when wrapping | Keeps the root cause for debugging |
| **Prefer try-with-resources** for cleanup | Auto-closes, avoids leaks, preserves suppressed exceptions |
| **Don't use exceptions for normal control flow** | Exceptions are expensive and obscure intent — use `if/else` for expected conditions |
| **Throw early, catch late** | Throw where the error is detected; catch where you can actually handle it |

### Logging an exception correctly

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void process(Order order) {
        try {
            paymentGateway.charge(order);
        } catch (PaymentException e) {
            // CORRECT: pass the exception object 'e' as the LAST argument → full stack trace logged
            log.error("Payment failed for order {}", order.getId(), e);

            // WRONG (do NOT do this): loses the stack trace entirely
            // log.error("Payment failed: " + e.getMessage());

            throw new OrderProcessingException("Could not process order", e); // re-wrap with cause
        }
    }
}
```

### Fail fast with input validation

```java
public void transfer(Account from, Account to, double amount) {
    // Validate at the TOP — throw immediately on bad input (fail fast)
    if (from == null || to == null) {
        throw new IllegalArgumentException("Accounts must not be null");
    }
    if (amount <= 0) {
        throw new IllegalArgumentException("Amount must be positive, got: " + amount);
    }
    // ...by here we KNOW the inputs are valid, so business logic stays clean
}
```

---

## Common Pitfalls

### 1. The empty catch block (swallowing)

```java
try {
    riskyCall();
} catch (Exception e) {
    // EMPTY — the worst anti-pattern. The error disappears with zero trace.
    // When this fails in production, you'll have NO idea why.
}
```

Even if you truly must ignore it, at least log it and explain why in a comment.

### 2. Catching Exception too broadly

```java
try {
    int x = Integer.parseInt(input);  // you EXPECT NumberFormatException here
    doDatabaseWork();                 // but this could throw something totally unrelated
} catch (Exception e) {               // catches BOTH — hides the unexpected DB error
    System.out.println("Invalid number"); // misleading message for a DB failure!
}
```

Catch only what you can actually handle. Let unexpected exceptions propagate.

### 3. Losing the stack trace when re-throwing

```java
catch (SQLException e) {
    throw new DataAccessException("DB error"); // 'e' not passed → root cause GONE
}
// Fix: throw new DataAccessException("DB error", e);
```

### 4. Resource leaks (forgetting to close)

```java
public void leak() throws IOException {
    FileReader reader = new FileReader("data.txt");
    reader.read();   // if this throws, reader.close() below is NEVER reached → file handle leaks!
    reader.close();
}
// Fix: use try-with-resources, which guarantees close()
```

### 5. Catching `NullPointerException` to "handle" nulls

```java
// WRONG — catching NPE to mask a bug
try {
    return user.getName().toUpperCase();
} catch (NullPointerException e) {
    return "UNKNOWN"; // hides the real problem: why was user (or name) null?
}
// Fix: validate or use Optional / null checks. NPE means a BUG, not a condition to catch.
```

### 6. Returning from finally (swallows exceptions and return values)

Covered earlier — `return` inside `finally` overrides everything, including thrown exceptions. Never do it.

---

## Connecting to Spring Backend

Exception handling in a real Spring Boot backend builds directly on these fundamentals. Two areas come up constantly in interviews.

### 1. Global exception handling with @ControllerAdvice / @ExceptionHandler

Instead of `try/catch` in every controller, Spring lets you handle exceptions **centrally**. A `@ControllerAdvice` class catches exceptions thrown anywhere in your controllers and converts them into clean HTTP responses.

**Think of it like a building's central security desk:** Instead of every room handling its own incidents, all alarms route to one desk that responds consistently.

```java
@RestControllerAdvice   // applies to ALL controllers globally
public class GlobalExceptionHandler {

    // When ANY controller throws UserNotFoundException, this method handles it
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException e) {
        // Convert the exception into a 404 response with a clean JSON body
        ErrorResponse body = new ErrorResponse(404, e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }

    // A catch-all for anything unexpected → returns 500
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        ErrorResponse body = new ErrorResponse(500, "Internal server error");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }
}
```

This keeps controllers clean (no `try/catch` clutter) and gives consistent error responses across the whole API. (A separate validation guide covers `@Valid` and field-level error handling in depth.)

### 2. @Transactional rolls back on unchecked exceptions only (by default)

This ties exception **type** directly to database behavior — a very common interview gotcha.

```java
@Transactional
public void saveOrder(Order order) throws IOException {
    orderRepository.save(order);   // step 1: data written

    if (somethingWrong) {
        throw new IllegalStateException("bad state"); // UNCHECKED → transaction ROLLS BACK ✓
    }

    if (otherProblem) {
        throw new IOException("file error");          // CHECKED → transaction COMMITS anyway! ✗
        // The save() above is NOT rolled back — a silent data bug.
    }
}
```

| Exception thrown | Default rollback? |
|---|---|
| `RuntimeException` / unchecked | **Yes** — rolls back |
| `Error` | **Yes** — rolls back |
| Checked `Exception` (e.g., `IOException`) | **No** — commits anyway |

**Fix**: To roll back on a checked exception, tell Spring explicitly:

```java
@Transactional(rollbackFor = IOException.class) // now a checked IOException also triggers rollback
public void saveOrder(Order order) throws IOException { ... }
```

> **Interview Tip**: "By default, `@Transactional` rolls back only on unchecked exceptions (`RuntimeException`) and `Error`. Checked exceptions commit unless you add `rollbackFor`." This single sentence impresses interviewers.

---

## Common Interview Questions

### Q: What is the difference between checked and unchecked exceptions?

**Checked** exceptions extend `Exception` (but not `RuntimeException`) and are verified by the compiler — you must catch or declare them with `throws`, or the code won't compile. They represent expected, recoverable failures (file not found, network down). **Unchecked** exceptions extend `RuntimeException` and are not checked by the compiler — you're not forced to handle them. They usually indicate programming bugs (null access, bad array index).

---

### Q: Where does RuntimeException sit in the Throwable hierarchy?

`Throwable` is the root. It has two direct children: `Error` and `Exception`. `RuntimeException` is a subclass of `Exception`. So the chain is **Throwable → Exception → RuntimeException**. `Error` is a sibling of `Exception`, not a parent of `RuntimeException`.

---

### Q: What is the difference between `throw` and `throws`?

`throw` is a statement used **inside** a method to actually throw an exception object (`throw new IllegalArgumentException(...)`). `throws` is used in a method **signature** to declare that the method might throw certain checked exceptions, warning callers they must handle them. One performs the action; the other is a declaration.

---

### Q: Does `finally` always run? When does it NOT run?

`finally` runs in almost all cases — including when `try`/`catch` has a `return`, and even when an exception propagates uncaught. It does **not** run if the JVM stops before reaching it: `System.exit()` is called, the JVM crashes or is killed, or the `try` block never completes (infinite loop / deadlock).

---

### Q: What will this code print?

```java
public int test() {
    try {
        return 1;
    } finally {
        return 2;
    }
}
```

It returns **2**. The `finally` block's `return` overrides the `try` block's `return 1`. (This also silently swallows any exception — which is exactly why you should never `return` from `finally`.)

---

### Q: Why is try-with-resources better than a finally block for closing resources?

It automatically calls `close()` on any `AutoCloseable` resource — no manual null checks, no risk of forgetting. It closes multiple resources in reverse order automatically. Critically, it **preserves the original exception**: if both the `try` body and `close()` throw, the original is propagated and the secondary one is attached as a suppressed exception (retrievable via `getSuppressed()`). A manual `finally` would lose the original.

---

### Q: Why must catch blocks go from specific to general?

Java matches `catch` blocks top to bottom and uses the first match. If a general parent (like `Exception`) comes before a specific child (like `IOException`), the parent catches everything and the child block becomes unreachable. The compiler rejects this with "exception has already been caught." So order most-specific first, most-general last.

---

### Q: How do you create a custom exception, and should it be checked or unchecked?

Extend `RuntimeException` for an unchecked exception or `Exception` for a checked one. Provide constructors taking a `message` and a `(message, cause)` pair to support chaining. Use **unchecked** (extends `RuntimeException`) when the caller usually can't recover at that point — this is the common modern/Spring style and avoids cluttering signatures with `throws`. Use **checked** when you want to force the caller to handle a recoverable condition.

---

### Q: What is exception chaining and why does it matter?

Chaining means passing the original exception as the **cause** when you throw a new one: `throw new MyException("msg", originalException)`. This preserves the root cause and its stack trace. Without it, you lose the real reason for the failure, making production bugs nearly impossible to debug. You retrieve the cause with `getCause()`, and printed traces show it under "Caused by:".

---

### Q: Why should you never catch `Throwable` or `Error`?

`Error` represents serious JVM-level problems like `OutOfMemoryError` and `StackOverflowError` — the JVM is in a state you cannot reliably recover from. Catching `Throwable` (which includes `Error`) can mask these fatal conditions and leave the application in a corrupt, unpredictable state. Catch `Exception` or, better, specific exception types instead.

---

### Q: What is the problem with an empty catch block?

It **swallows** the exception — the error disappears with no log, no trace, no signal. When the code fails in production, there's no information to diagnose it. At minimum, log the exception with its stack trace. Empty catches are one of the most damaging anti-patterns in Java.

---

### Q: By default, which exceptions cause a Spring `@Transactional` method to roll back?

By default, only **unchecked** exceptions (`RuntimeException` and its subclasses) and `Error` trigger a rollback. **Checked** exceptions (like `IOException`) do **not** roll back — the transaction commits anyway. To roll back on a checked exception, use `@Transactional(rollbackFor = SomeCheckedException.class)`.

---

### Q: What is the difference between `final`, `finally`, and `finalize`?

A classic trick question on three unrelated keywords:
- **`final`** — a modifier: a final variable can't be reassigned, a final method can't be overridden, a final class can't be extended.
- **`finally`** — a block after `try/catch` that almost always runs, used for cleanup.
- **`finalize()`** — a deprecated `Object` method the garbage collector *might* call before reclaiming an object. Don't rely on it; it's removed in modern Java.

---

### Q: What happens if an exception is thrown and never caught?

It propagates up the call stack from method to method until it either gets caught or reaches the top (`main` or the thread's run method). If nothing catches it, the JVM prints the stack trace to `System.err` and terminates that thread (the whole program if it's the main thread).

---

## Quick Reference Cheat Sheet

```
THROWABLE HIERARCHY:
  Throwable
    ├── Error              → serious, do NOT catch (OutOfMemoryError, StackOverflowError)
    └── Exception          → recoverable
          ├── RuntimeException → UNCHECKED (NullPointerException, IllegalArgumentException)
          └── (others)         → CHECKED   (IOException, SQLException)

CHECKED vs UNCHECKED:
  Checked   → extends Exception (not RuntimeException); compiler forces catch/throws
  Unchecked → extends RuntimeException; not enforced; usually a code bug

try / catch / finally:
  try     → risky code
  catch   → handle a matching exception (specific BEFORE general!)
  finally → cleanup; runs almost always
  multi-catch → catch (A | B e) { }   (types must be unrelated)

finally does NOT run when:
  → System.exit() called
  → JVM crash / killed
  → infinite loop / deadlock in try (try never completes)

finally + return GOTCHA:
  return in finally OVERRIDES try/catch return AND swallows exceptions → never do it

try-with-resources:
  try (Resource r = new Resource()) { ... }   // auto-calls r.close()
  needs AutoCloseable; closes in REVERSE order; preserves suppressed exceptions
  BEATS finally: less code, no leaks, keeps original exception

throw vs throws:
  throw  → statement, throws ONE object now:  throw new X("msg");
  throws → signature declaration of possible exceptions:  void m() throws IOException

PROPAGATION:
  uncaught exception bubbles UP the call stack until caught or program crashes

CUSTOM EXCEPTIONS:
  extends RuntimeException → unchecked (common, Spring style)
  extends Exception        → checked (forces handling)
  ALWAYS add (String message) and (String message, Throwable cause) constructors

CHAINING / WRAPPING:
  throw new MyException("msg", cause);   // preserve root cause!
  e.getCause()                           // retrieve original
  printed trace shows "Caused by:"

BEST PRACTICES:
  ✓ catch specific, not Exception/Throwable
  ✓ log WITH the exception: log.error("msg", e)
  ✓ fail fast — validate inputs early
  ✓ preserve the cause when wrapping
  ✓ use try-with-resources for cleanup
  ✗ never swallow (empty catch)
  ✗ never return from finally
  ✗ never catch Throwable/Error
  ✗ don't use exceptions for normal control flow

SPRING BACKEND:
  @RestControllerAdvice + @ExceptionHandler → central exception → HTTP response
  @Transactional rollback (default):
    RuntimeException / Error → ROLLS BACK
    checked Exception        → COMMITS (use rollbackFor = X.class to roll back)

final / finally / finalize:
  final    → can't reassign/override/extend
  finally  → cleanup block after try
  finalize → deprecated GC hook; don't use
```

---

*Last Updated: 2026-06-11*
