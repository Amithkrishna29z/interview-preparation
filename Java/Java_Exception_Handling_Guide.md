# Java Exception Handling Interview Questions & Study Guide

## Overview

An **exception** is how Java signals that something went wrong while a program runs — a missing file, a bad network call, a `null` value, a divide-by-zero. Instead of crashing silently, Java "throws" an exception object and looks for code that can "catch" and handle it.

This is one of the most commonly tested fundamentals in Java backend interviews. Expect questions on checked vs unchecked exceptions, `try/catch/finally`, `throw` vs `throws`, custom exceptions, and the `finally`/`return` puzzle.

---

## Table of Contents

1. [The Throwable Hierarchy](#the-throwable-hierarchy)
2. [Checked vs Unchecked Exceptions](#checked-vs-unchecked-exceptions)
3. [try / catch / finally](#try--catch--finally)
4. [Multi-Catch and Catch Order](#multi-catch-and-catch-order)
5. [finally and return](#finally-and-return)
6. [try-with-resources](#try-with-resources)
7. [throw vs throws and Propagation](#throw-vs-throws-and-propagation)
8. [Custom Exceptions](#custom-exceptions)
9. [Exception Chaining](#exception-chaining)
10. [Best Practices & Common Pitfalls](#best-practices--common-pitfalls)
11. [Connecting to Spring Backend](#connecting-to-spring-backend)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## The Throwable Hierarchy

Every error in Java descends from one root class, `Throwable`. Interviewers love asking "where does `RuntimeException` sit?"

```
            Throwable                ← root of ALL errors/exceptions
            /        \
        Error      Exception
   (serious, do    /        \
    NOT catch)  Runtime    (everything else
               Exception    = CHECKED)
              (UNCHECKED)
```

- **`Error`** — catastrophic JVM problems you should NOT catch (`OutOfMemoryError`, `StackOverflowError`).
- **`Exception`** — problems your program can reasonably handle.
- **`RuntimeException`** — a branch of `Exception` for programming bugs (`NullPointerException`, `ArithmeticException`).

> **Interview Tip**: Memorize the chain — **Throwable → Exception → RuntimeException**. `Error` is a *sibling* of `Exception`, not a parent of `RuntimeException`.

---

## Checked vs Unchecked Exceptions

The single most asked question. The distinction is whether the **compiler forces you** to deal with the exception.

| Feature | Checked | Unchecked |
|---|---|---|
| Parent class | `Exception` (not `RuntimeException`) | `RuntimeException` or `Error` |
| Checked at compile time? | **Yes** | No |
| Must catch or declare? | **Yes** (or won't compile) | No |
| Typical cause | External failure (file, network, DB) | Programming bug (null, bad index) |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

**Checked** — if a method can throw one, you MUST catch it or declare it with `throws`:

```java
public void readFile() throws IOException {   // declared — or it won't compile
    FileReader reader = new FileReader("data.txt"); // may throw IOException (checked)
    reader.read();
    reader.close();
}
```

**Unchecked** — not enforced by the compiler; usually signals a bug to fix:

```java
public int divide(int a, int b) {  // no "throws" needed
    return a / b;                  // if b == 0, throws ArithmeticException (unchecked)
}
```

> **Interview Tip**: "Is `NullPointerException` checked or unchecked?" → **unchecked**, because it extends `RuntimeException`.

---

## try / catch / finally

- `try` = the risky code.
- `catch` = what to do if a matching exception is thrown.
- `finally` = cleanup that runs whether or not an exception happened.

```java
try {
    int result = 10 / 0;          // throws ArithmeticException
    System.out.println(result);   // skipped
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero: " + e.getMessage());
} finally {
    System.out.println("Cleanup done.");  // runs always
}
System.out.println("Program continues."); // runs because the exception was caught
```

**Output:**
```
Cannot divide by zero: / by zero
Cleanup done.
Program continues.
```

**Rule**: A `try` needs at least one `catch` OR `finally`. You can have `try`/`finally` with no `catch` (cleanup while letting the exception propagate).

---

## Multi-Catch and Catch Order

**Multiple catch blocks** — Java checks them top to bottom and runs the first match:

```java
try {
    int number = Integer.parseInt(input); // may throw NumberFormatException
    int result = 100 / number;            // may throw ArithmeticException
} catch (NumberFormatException e) {
    System.out.println("Not a valid number");
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero.");
}
```

**Multi-catch** (Java 7+) — one block for several unrelated types, using `|`:

```java
catch (NumberFormatException | NullPointerException e) {
    System.out.println("Bad input: " + e.getMessage());
}
```

**Catch order: specific before general.** A parent type first makes the child unreachable — a compile error:

```java
// WRONG — won't compile: Exception catches everything first
catch (Exception e) { ... }
catch (IOException e) { ... }   // unreachable

// CORRECT — specific child first, general parent last
catch (IOException e) { ... }
catch (Exception e) { ... }
```

---

## finally and return

The **#1 output-prediction puzzle**. Two facts:

1. `finally` runs even when `try`/`catch` has a `return`.
2. A `return` inside `finally` **overrides** any return or exception from `try`/`catch` — bad practice, but commonly tested.

```java
public int test() {
    try {
        return 1;        // value prepared...
    } finally {
        return 2;        // ...but finally's return WINS → returns 2
    }
}
```

A `return` in `finally` also **swallows exceptions** (an exception thrown in `try` just vanishes). Never return from `finally`.

**When `finally` does NOT run:** the JVM stops before reaching it — `System.exit()` is called, the JVM crashes/is killed, or `try` never completes (infinite loop/deadlock).

---

## try-with-resources

Automatically closes resources (files, streams, DB connections) — no manual `finally`. Any class implementing `AutoCloseable` works.

```java
// Old way — verbose, easy to forget close()
FileReader reader = null;
try {
    reader = new FileReader("data.txt");
    reader.read();
} finally {
    if (reader != null) reader.close();
}

// New way — Java auto-closes, even if an exception is thrown
try (FileReader reader = new FileReader("data.txt")) {
    reader.read();
}
```

Multiple resources are closed in **reverse order** of declaration. If both the `try` body and `close()` throw, try-with-resources **preserves the original exception** and attaches the other as a *suppressed* exception (`getSuppressed()`) — a manual `finally` would lose the original.

> **Interview Tip**: Always prefer try-with-resources for anything `Closeable`/`AutoCloseable` — shorter, safer, no leaks.

---

## throw vs throws and Propagation

| Keyword | Where | What it does |
|---|---|---|
| `throw` | Inside a method body | **Actually throws** one exception object now |
| `throws` | In the method signature | **Declares** the exceptions a method might throw |

```java
public void validateAge(int age) throws IllegalArgumentException {  // declaration
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative"); // action
    }
}
```

> Memory trick: **`throws` has an "s"** — it can list **s**everal types. **`throw`** throws exactly **one** object.

**Propagation** — if a method doesn't catch an exception, Java passes it up the call stack (caller, then caller's caller…) until something catches it or it reaches `main` and crashes:

```java
public static void main(String[] args) {
    try {
        methodA();   // A → B → C; the exception bubbles all the way up to here
    } catch (RuntimeException e) {
        System.out.println("Caught in main: " + e.getMessage());
    }
}
static void methodA() { methodB(); }
static void methodB() { methodC(); }
static void methodC() { throw new RuntimeException("Error in C"); }
// Output: Caught in main: Error in C
```

---

## Custom Exceptions

A `UserNotFoundException` is clearer than a generic `RuntimeException` and lets callers catch *your specific* error.

**Key decision — extend `Exception` or `RuntimeException`?**

| Extend... | Makes it... | Use when... |
|---|---|---|
| `Exception` | **Checked** (caller forced to handle) | The caller can recover and you want to force handling |
| `RuntimeException` | **Unchecked** (not forced) | A bug/business condition the caller usually can't recover from |

Modern style (and Spring) leans **unchecked** to avoid cluttering signatures with `throws`.

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
    // Constructor with a cause — needed for chaining
    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Usage
return userRepository.findById(id)
    .orElseThrow(() -> new UserNotFoundException("No user with id: " + id));
```

**Design rules**: always provide `(String message)` and `(String message, Throwable cause)` constructors; name it clearly ending in `Exception`; reuse built-ins like `IllegalArgumentException` when they already fit.

---

## Exception Chaining

When you catch a low-level exception and throw a higher-level one, **pass the original as the cause** — or you lose the real reason for the failure.

```java
try {
    readFileFromDisk();          // throws IOException
} catch (IOException e) {
    throw new ConfigLoadException("Failed to load config", e); // 'e' = the cause
}
```

```java
catch (IOException e) {
    throw new ConfigLoadException("Failed to load config");  // WRONG — cause lost
    throw new ConfigLoadException("Failed to load config", e); // CORRECT
}
```

Retrieve the original with `e.getCause()`. Printed stack traces show it under **"Caused by:"**.

> **Interview Tip**: Always use the `(message, cause)` constructor when re-throwing. Wrapping without the cause makes production bugs un-debuggable.

---

## Best Practices & Common Pitfalls

| Do | Why |
|---|---|
| **Catch specific exceptions**, not `Exception` broadly | Broad catches hide bugs you didn't mean to handle |
| **Never catch `Throwable` or `Error`** | `Error` means the JVM is dying — catching hides fatal problems |
| **Log with the exception object**: `log.error("msg", e)` | Keeps the full stack trace; `e.getMessage()` alone loses it |
| **Preserve the cause** when wrapping | Keeps the root cause for debugging |
| **Prefer try-with-resources** for cleanup | Auto-closes, no leaks |
| **Fail fast** — validate inputs early and throw | Catch bad data at the boundary |

**Avoid:**

- **Empty catch block** — `catch (Exception e) {}` swallows the error with zero trace. The worst anti-pattern.
- **Catching too broadly** — `catch (Exception e)` over code that throws several unrelated exceptions hides the unexpected one.
- **Catching `NullPointerException` to "handle" nulls** — an NPE is a bug; validate or use `Optional` instead.
- **Returning from `finally`** — overrides returns and swallows exceptions.

```java
// Correct logging + re-wrap with cause
catch (PaymentException e) {
    log.error("Payment failed for order {}", order.getId(), e); // full stack trace
    throw new OrderProcessingException("Could not process order", e);
}
```

---

## Connecting to Spring Backend

### 1. Global handling with `@RestControllerAdvice` / `@ExceptionHandler`

Instead of `try/catch` in every controller, Spring handles exceptions **centrally** and converts them into clean HTTP responses:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, e.getMessage()));
    }

    @ExceptionHandler(Exception.class)  // catch-all → 500
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "Internal server error"));
    }
}
```

### 2. `@Transactional` rolls back on unchecked exceptions only (by default)

A very common gotcha — exception **type** controls database behavior:

| Exception thrown | Default rollback? |
|---|---|
| `RuntimeException` / unchecked | **Yes** |
| `Error` | **Yes** |
| Checked `Exception` (e.g. `IOException`) | **No** — commits anyway |

**Fix** — roll back on a checked exception explicitly:

```java
@Transactional(rollbackFor = IOException.class)
public void saveOrder(Order order) throws IOException { ... }
```

> **Interview Tip**: "By default `@Transactional` rolls back only on unchecked exceptions and `Error`; checked exceptions commit unless you add `rollbackFor`."

---

## Common Interview Questions

**Q: Difference between checked and unchecked exceptions?**
Checked extend `Exception` (not `RuntimeException`) and are verified by the compiler — you must catch or declare them. They represent expected, recoverable failures. Unchecked extend `RuntimeException`, aren't checked by the compiler, and usually mean a programming bug.

**Q: Where does `RuntimeException` sit in the hierarchy?**
`Throwable → Exception → RuntimeException`. `Error` is a sibling of `Exception`, not a parent of `RuntimeException`.

**Q: Difference between `throw` and `throws`?**
`throw` is a statement inside a method that actually throws an object. `throws` is a signature declaration that a method might throw certain exceptions.

**Q: Does `finally` always run?**
Almost always — including when `try`/`catch` returns or an exception propagates. It does NOT run if the JVM stops first: `System.exit()`, a JVM crash/kill, or an infinite loop in `try`.

**Q: What does this print?**
```java
try { return 1; } finally { return 2; }
```
Returns **2** — `finally`'s return overrides the `try` return (and would swallow any exception).

**Q: Why is try-with-resources better than `finally`?**
It auto-calls `close()` (no null checks, no forgetting), closes multiple resources in reverse order, and preserves the original exception (the secondary one becomes a suppressed exception).

**Q: Why must catch blocks go specific to general?**
Java uses the first matching `catch`. A general parent first makes the specific child unreachable — a compile error.

**Q: What is exception chaining and why does it matter?**
Passing the original exception as the cause: `throw new MyException("msg", original)`. It preserves the root cause and stack trace, retrieved via `getCause()` and shown under "Caused by:".

**Q: Why never catch `Throwable` or `Error`?**
`Error` represents fatal JVM problems (`OutOfMemoryError`, `StackOverflowError`) you can't reliably recover from; catching them masks fatal conditions.

**Q: Difference between `final`, `finally`, and `finalize`?**
`final` — a modifier (can't reassign/override/extend). `finally` — a cleanup block after `try/catch`. `finalize()` — a deprecated `Object` method the GC might call; don't use it.

**Q: By default, which exceptions roll back a `@Transactional` method?**
Only unchecked (`RuntimeException`) and `Error`. Checked exceptions commit unless you add `rollbackFor`.

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
  try → risky code   catch → handle a match (specific BEFORE general!)   finally → cleanup
  multi-catch → catch (A | B e) { }   (types must be unrelated)

finally does NOT run: System.exit(), JVM crash/kill, infinite loop in try
finally + return: overrides try/catch return AND swallows exceptions → never do it

try-with-resources:
  try (Resource r = new Resource()) { ... }   // auto-calls r.close()
  needs AutoCloseable; closes in REVERSE order; preserves suppressed exceptions

throw vs throws:
  throw  → statement, throws ONE object now:  throw new X("msg");
  throws → signature declaration:  void m() throws IOException

CUSTOM EXCEPTIONS:
  extends RuntimeException → unchecked (common, Spring style)
  extends Exception        → checked
  ALWAYS add (String message) and (String message, Throwable cause) constructors

CHAINING:
  throw new MyException("msg", cause);   // preserve root cause!
  e.getCause()   → retrieve original     printed trace shows "Caused by:"

SPRING:
  @RestControllerAdvice + @ExceptionHandler → central exception → HTTP response
  @Transactional rollback (default): RuntimeException/Error → ROLLS BACK;
                                     checked → COMMITS (use rollbackFor = X.class)

DO: catch specific, log WITH e, fail fast, preserve cause, use try-with-resources
DON'T: swallow (empty catch), return from finally, catch Throwable/Error
```

---

*Last Updated: 2026-06-18*
