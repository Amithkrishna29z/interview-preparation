# C# Exception Handling — A Guide for Java Developers

## Overview

If you already know Java exception handling, you know 80% of C# exception handling. The mechanics (`try`/`catch`/`finally`, throwing, catching by type, a class hierarchy rooted at a base `Exception` type) are almost identical. This guide focuses on the **differences that trip up Java developers** and the C#-specific features you'll be asked about in a junior .NET interview.

The single biggest difference: **C# has no checked exceptions.** There is no `throws` clause, the compiler never forces you to catch or declare anything, and every C# exception behaves like a Java `RuntimeException`. Everything else is a refinement of ideas you already understand.

This guide assumes **.NET 8+** and modern C# syntax.

---

## Table of Contents

- [Java → C# Exceptions Mapping](#java--c-exceptions-mapping)
- [The Exception Hierarchy](#the-exception-hierarchy)
- [No Checked Exceptions (The Big One)](#no-checked-exceptions-the-big-one)
- [try / catch / finally](#try--catch--finally)
- [Multiple Catch Blocks and Ordering](#multiple-catch-blocks-and-ordering)
- [Exception Filters (`when`)](#exception-filters-when)
- [`throw` vs `throw;` vs `throw ex;` (Interview Gotcha)](#throw-vs-throw-vs-throw-ex-interview-gotcha)
- [Custom Exceptions](#custom-exceptions)
- [`using`, `IDisposable`, and Resource Cleanup](#using-idisposable-and-resource-cleanup)
- [Common Built-in Exceptions](#common-built-in-exceptions)
- [Guard Clauses and `ThrowIfNull`](#guard-clauses-and-throwifnull)
- [`nameof` for Parameter Names](#nameof-for-parameter-names)
- [Exception Handling in async/await and AggregateException](#exception-handling-in-asyncawait-and-aggregateexception)
- [Best Practices](#best-practices)
- [Global Exception Handling in ASP.NET Core](#global-exception-handling-in-aspnet-core)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Java → C# Exceptions Mapping

| Java | C# | Notes |
|------|-----|-------|
| `Throwable` / `Exception` | `System.Exception` | Base of (almost) everything you'd catch. |
| `RuntimeException` (unchecked) | *(all C# exceptions are unchecked)* | No checked/unchecked split in C#. |
| Checked exceptions | *(do not exist)* | No compiler enforcement at all. |
| `throws IOException` clause | *(none in C#)* | You never declare what a method throws. |
| `NullPointerException` | `NullReferenceException` | Thrown on dereferencing `null`. |
| `IllegalArgumentException` | `ArgumentException` | Bad argument value. |
| `NullPointerException` (for a null arg) | `ArgumentNullException` | C# distinguishes "arg was null" from "deref of null". |
| `IndexOutOfBoundsException` | `IndexOutOfRangeException` (arrays) / `ArgumentOutOfRangeException` | Two different ones in C#. |
| `IllegalStateException` | `InvalidOperationException` | Object in wrong state for the call. |
| `NumberFormatException` | `FormatException` | Bad parse input. |
| `ArithmeticException` (`/ by zero`) | `DivideByZeroException` | Integer divide by zero. |
| `ClassCastException` | `InvalidCastException` | Bad cast. |
| `UnsupportedOperationException` | `NotSupportedException` | Operation not supported. |
| *(no direct equivalent)* | `NotImplementedException` | "Not done yet" stub marker. |
| `OutOfMemoryError` | `OutOfMemoryError` → `OutOfMemoryException` | Fatal-ish. |
| `StackOverflowError` | `StackOverflowException` | **Uncatchable** in C# — process dies. |
| `finally` block | `finally` | Identical behavior. |
| try-with-resources `try (var r = ...)` | `using` statement / `using` declaration | Backed by `IDisposable`. |
| `AutoCloseable` / `Closeable` | `IDisposable` / `IAsyncDisposable` | `Dispose()` ≈ `close()`. |
| `e.getCause()` | `e.InnerException` | The wrapped exception. |
| `e.getMessage()` | `e.Message` | The message text. |
| `e.printStackTrace()` | `e.StackTrace` (string) / logging | C# exposes it as a string. |
| `e.getStackTrace()` | `e.StackTrace` / `e.ToString()` | |
| `catch (A \| B e)` (multi-catch) | `catch (Exception e) when (e is A or B)` | Use a filter to emulate. |
| `Objects.requireNonNull(x)` | `ArgumentNullException.ThrowIfNull(x)` | Guard helper. |

---

## The Exception Hierarchy

**Think of it like...** Java's `Throwable → Exception → RuntimeException` tree. C# has a similar tree, just rooted at `System.Exception`, but without the "checked vs unchecked" branch.

```
System.Object
  └── System.Exception              // root of everything you catch (≈ Java Throwable/Exception)
        ├── SystemException         // thrown by the CLR/runtime & BCL (e.g. NullReferenceException)
        │     ├── NullReferenceException
        │     ├── ArgumentException
        │     │     ├── ArgumentNullException
        │     │     └── ArgumentOutOfRangeException
        │     ├── InvalidOperationException
        │     ├── IndexOutOfRangeException
        │     └── ... (FormatException, InvalidCastException, etc.)
        └── ApplicationException    // historically "for your app's exceptions" — now discouraged
```

```csharp
// Catching the root catches (almost) everything.
try
{
    DoSomething();
}
catch (Exception ex) // ≈ Java: catch (Exception ex) — but in C# this also catches
{                    //   things Java would call Errors, e.g. OutOfMemoryException.
    Console.WriteLine(ex.Message);
}
```

**`SystemException` vs `ApplicationException`:**
- `SystemException` is the base for exceptions thrown by the .NET runtime and base class library.
- `ApplicationException` was *originally* intended as the base for your custom exceptions (mirroring an old Java idea). **Microsoft now recommends deriving custom exceptions directly from `System.Exception`, not `ApplicationException`.** The distinction added no value, so just ignore `ApplicationException`.

---

## No Checked Exceptions (The Big One)

**Think of it like...** *every* C# exception is a Java `RuntimeException`. The compiler never makes you handle or declare anything.

This is the difference Java developers must internalize:

```csharp
// C#: no 'throws' clause exists. This compiles fine even though it can throw.
public string ReadConfig(string path)
{
    return File.ReadAllText(path); // can throw FileNotFoundException, IOException...
}                                  // ...but the signature says nothing about it.

// Java equivalent would FORCE you to write:
//   public String readConfig(String path) throws IOException { ... }
// or wrap it in try/catch. C# does neither.
```

**Implications you should be able to articulate in an interview:**

1. **No `throws` keyword.** Method signatures never list exceptions. (`throws` in C# does not exist; `throw` is the statement.)
2. **The compiler won't help you.** You discover what a method can throw from documentation (XML docs / IntelliSense), not the signature.
3. **No "catch or specify" rule.** You catch only where you can meaningfully handle. Otherwise let it propagate.
4. **Cleaner signatures, but less safety.** No `throws IOException, SQLException` noise — but also no compiler reminder that something can fail.
5. **No checked-exception wrapping ceremony.** In Java you often wrap checked exceptions in `RuntimeException` to cross interfaces (e.g. lambdas, streams). In C# that problem simply doesn't exist.

```csharp
// Because there are no checked exceptions, lambdas and LINQ "just work" with throwing code:
var lengths = files.Select(f => File.ReadAllText(f).Length); // no checked-exception headaches
// In Java, File.readAllText-style calls inside a stream force ugly try/catch-in-lambda.
```

---

## try / catch / finally

**Think of it like...** Java's `try/catch/finally`. Same keywords, same semantics.

```csharp
try
{
    int x = int.Parse("not a number"); // throws FormatException
}
catch (FormatException ex)
{
    // Handle a specific type, just like Java.
    Console.WriteLine($"Bad input: {ex.Message}");
}
finally
{
    // Always runs — for cleanup. Same as Java's finally.
    Console.WriteLine("Cleanup happens here.");
}
```

Notes for Java devs:
- `finally` runs whether or not an exception was thrown — identical to Java.
- You can have `try`/`finally` with **no** `catch` (cleanup without handling) — also identical to Java.
- A `catch` block with no type, just `catch { }`, catches everything (like `catch (Exception)`). Prefer the typed form for clarity.

---

## Multiple Catch Blocks and Ordering

**Think of it like...** Java multi-`catch` blocks, including the rule that **a more specific exception must come before a more general one.**

```csharp
try
{
    Process();
}
catch (ArgumentNullException ex) // most specific FIRST
{
    Console.WriteLine("A required argument was null.");
}
catch (ArgumentException ex)      // ArgumentNullException's base — comes AFTER
{
    Console.WriteLine("Some argument was invalid.");
}
catch (Exception ex)              // most general LAST
{
    Console.WriteLine("Something else went wrong.");
}
```

If you put `catch (Exception)` first, the compiler errors: the later, more-specific blocks become unreachable — **C# enforces this at compile time**, just like Java.

C# has **no single-block multi-type catch** like Java's `catch (IOException | SQLException e)`. To catch several unrelated types in one block, use an exception filter (next section):

```csharp
catch (Exception ex) when (ex is FormatException or OverflowException)
{
    // Handles both, similar to Java's catch (FormatException | OverflowException e)
}
```

---

## Exception Filters (`when`)

**Think of it like...** *there is no Java equivalent.* This is a genuinely C#-only feature worth highlighting.

An exception filter lets you add a boolean condition to a `catch`. The block runs **only if the condition is true**; otherwise the exception keeps propagating as if this `catch` weren't there.

```csharp
try
{
    CallApi();
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // Only catches 404s. Other HttpRequestExceptions propagate untouched.
    Console.WriteLine("Resource not found.");
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.ServiceUnavailable)
{
    Retry();
}
```

**Why filters matter (interview-worthy points):**
- The exception is **not unwound** when the filter is evaluated — the stack stays intact while deciding. This makes debugging better than a `catch`-then-`if`-then-`throw;`.
- Great for conditional handling without rethrowing.
- A common idiom is a logging filter that *always returns false* (logs as a side effect, never catches):

```csharp
catch (Exception ex) when (Log(ex)) { } // Log returns false, so this never actually catches.
// The exception is logged at the point of throw, then continues propagating.
```

---

## `throw` vs `throw;` vs `throw ex;` (Interview Gotcha)

**Think of it like...** Java's `throw new X()` to raise, and a bare re-`throw`. But C# has a subtle, frequently-tested distinction between `throw;` and `throw ex;`.

```csharp
try
{
    DeepCall();
}
catch (Exception ex)
{
    // 1) Raise a brand-new exception (optionally wrapping the original):
    throw new DataException("Failed to load data.", ex); // 'ex' becomes InnerException

    // 2) Re-throw the CURRENT exception, PRESERVING the original stack trace:
    throw;          // ✅ correct rethrow — keeps where it originally came from

    // 3) Re-throw the SAME object but RESET the stack trace to HERE:
    throw ex;       // ❌ BUG: loses the original throw location, hides the real source
}
```

**The gotcha:** `throw ex;` overwrites the stack trace so it looks like the exception originated at this line, destroying the trail to the real cause. `throw;` (no operand) preserves the original stack trace. **Always use `throw;` to rethrow.**

Java comparison: in Java, `throw e;` re-throws *without* mangling the stack trace, so Java devs habitually write `throw e;`. In C# that habit is a bug — you must write the bare `throw;`.

| Statement | Effect | Use when |
|-----------|--------|----------|
| `throw new X(...)` | New exception (wrap original as inner) | Adding context / translating exception type |
| `throw;` | Rethrow current, **keep** stack trace | You inspected/logged and want it to propagate |
| `throw ex;` | Rethrow current, **reset** stack trace | **Almost never** — this is the anti-pattern |

---

## Custom Exceptions

**Think of it like...** extending `RuntimeException` in Java — except you derive from `System.Exception` (not `ApplicationException`).

A well-behaved custom exception provides the standard constructors:

```csharp
public class OrderNotFoundException : Exception   // derive from Exception, not ApplicationException
{
    // 1) Parameterless
    public OrderNotFoundException() { }

    // 2) Message only
    public OrderNotFoundException(string message)
        : base(message) { }

    // 3) Message + inner exception (so you can wrap a cause — like Java's Throwable cause)
    public OrderNotFoundException(string message, Exception inner)
        : base(message, inner) { }

    // 4) Optional: custom data carried on the exception
    public int OrderId { get; }

    public OrderNotFoundException(int orderId)
        : base($"Order {orderId} was not found.")
    {
        OrderId = orderId;
    }
}
```

Usage:

```csharp
try
{
    LoadOrder(id);
}
catch (SqlException ex)
{
    // Wrap a low-level cause in a domain-specific exception.
    throw new OrderNotFoundException($"Could not load order {id}.", ex);
    // ex is now accessible via the new exception's .InnerException (≈ Java getCause()).
}
```

Conventions:
- Name ends in `Exception` (like Java).
- Provide the three standard constructors so callers can use it idiomatically.
- Derive from `Exception` directly (modern guidance).
- *(Legacy note: old code implemented `ISerializable` / a serialization constructor. Binary serialization of exceptions is obsolete in .NET 8+, so you can skip it for a junior role.)*

---

## `using`, `IDisposable`, and Resource Cleanup

**Think of it like...** Java's try-with-resources. `IDisposable.Dispose()` is C#'s `AutoCloseable.close()`.

Any type implementing `IDisposable` can be wrapped in a `using` so its `Dispose()` is called automatically — even if an exception is thrown.

**`using` statement (classic, scoped block):**

```csharp
using (var file = new StreamReader("data.txt")) // ≈ Java: try (var file = new ...) {
{
    var text = file.ReadToEnd();
} // file.Dispose() called here automatically, even on exception
```

**`using` declaration (C# 8+, no braces — disposes at end of enclosing scope):**

```csharp
void Read()
{
    using var file = new StreamReader("data.txt"); // disposed when method scope ends
    var text = file.ReadToEnd();
    // ... no extra indentation, no braces; Dispose() runs when Read() returns
}
```

**`await using` for async cleanup (`IAsyncDisposable`):**

```csharp
await using var conn = new SqlConnection(cs); // calls DisposeAsync() at scope end
await conn.OpenAsync();
// Java has no direct async-close equivalent; this awaits the cleanup.
```

Writing your own disposable:

```csharp
public sealed class TempFile : IDisposable
{
    private readonly string _path = Path.GetTempFileName();

    public void Dispose() // ≈ Java AutoCloseable.close()
    {
        if (File.Exists(_path)) File.Delete(_path);
    }
}

// Used the same way:
using var tmp = new TempFile(); // cleaned up automatically
```

`using` is just syntactic sugar for `try/finally` calling `Dispose()` — exactly like Java's try-with-resources is sugar for `try/finally` calling `close()`.

---

## Common Built-in Exceptions

**Think of it like...** the standard `RuntimeException` family in Java — just with .NET names.

```csharp
// NullReferenceException — dereferencing null (≈ Java NullPointerException)
string s = null;
int len = s.Length; // 💥 NullReferenceException

// ArgumentException — invalid argument value (≈ Java IllegalArgumentException)
throw new ArgumentException("Name cannot be blank.", nameof(name));

// ArgumentNullException — a required argument was null
throw new ArgumentNullException(nameof(customer));

// ArgumentOutOfRangeException — argument outside the valid range
throw new ArgumentOutOfRangeException(nameof(age), age, "Age must be 0–150.");

// IndexOutOfRangeException — bad ARRAY index (≈ Java IndexOutOfBoundsException)
int[] arr = new int[3];
int x = arr[5]; // 💥 IndexOutOfRangeException

// InvalidOperationException — object is in the wrong state for this call (≈ IllegalStateException)
var e = list.GetEnumerator();
var cur = e.Current; // 💥 InvalidOperationException (haven't called MoveNext)

// FormatException — bad parse input (≈ Java NumberFormatException)
int n = int.Parse("abc"); // 💥 FormatException

// InvalidCastException — bad cast (≈ Java ClassCastException)
object o = "hello";
int i = (int)o; // 💥 InvalidCastException

// DivideByZeroException — integer divide by zero (≈ Java ArithmeticException)
int q = 10 / 0; // 💥 DivideByZeroException

// NotImplementedException — stub marker, "I haven't written this yet"
public void Future() => throw new NotImplementedException();

// NotSupportedException — operation legitimately unsupported (≈ UnsupportedOperationException)
```

Quick distinction junior devs get asked about:
- **`IndexOutOfRangeException`** = thrown by the CLR when you index an **array** out of bounds.
- **`ArgumentOutOfRangeException`** = thrown by **your code / library methods** when a passed-in index/value argument is out of the allowed range (e.g. `List<T>` indexer, `Substring`).

---

## Guard Clauses and `ThrowIfNull`

**Think of it like...** Java's `Objects.requireNonNull(x)`, but with cleaner built-in helpers.

Validate inputs at the top of a method and fail fast:

```csharp
public void Register(Customer customer, string email)
{
    // Modern, concise null check (.NET 6+). Uses the caller's argument name automatically.
    ArgumentNullException.ThrowIfNull(customer); // ≈ Java Objects.requireNonNull(customer)

    // Throws ArgumentException if null OR empty/whitespace (.NET 8 has this too)
    ArgumentException.ThrowIfNullOrWhiteSpace(email);

    // Range guards
    ArgumentOutOfRangeException.ThrowIfNegative(customer.Age);

    // ... method body can now assume valid inputs
}
```

The old-style equivalent (still common, good to recognize):

```csharp
if (customer is null)
    throw new ArgumentNullException(nameof(customer)); // explicit version of ThrowIfNull
```

`ThrowIfNull` is preferred: it's shorter, captures the parameter name automatically (via `[CallerArgumentExpression]`), and reads clearly.

---

## `nameof` for Parameter Names

**Think of it like...** a compile-time way to refer to a symbol's name so refactors don't break your error messages. Java has no built-in equivalent (you'd hardcode the string).

```csharp
public void SetSpeed(int speed)
{
    if (speed < 0)
        throw new ArgumentOutOfRangeException(nameof(speed), "Speed cannot be negative.");
        //                                    ^^^^^^^^^^^^^ becomes the literal "speed"
}
```

Why `nameof("speed")` instead of the literal string `"speed"`:
- If you **rename** the parameter, `nameof` updates automatically and the build fails if it can't resolve — a hardcoded `"speed"` would silently go stale.
- Tooling (analyzers, IDEs) can track the reference.

`ArgumentException` and friends take the parameter name as their last argument; always pass it with `nameof`.

---

## Exception Handling in async/await and AggregateException

**Think of it like...** exceptions from a `Future`/`CompletableFuture` in Java, but `await` "unwraps" them so they feel synchronous.

```csharp
public async Task LoadAsync()
{
    try
    {
        await FetchDataAsync(); // if this throws, the exception surfaces right here
    }
    catch (HttpRequestException ex)
    {
        // Caught just like a synchronous exception — await re-raises the original.
        Console.WriteLine(ex.Message);
    }
}
```

Key points:
- `await` re-throws the **original** exception (not wrapped), so normal `try/catch` works as expected.
- A faulted `Task` stores the exception; `await` extracts and rethrows it.

**`AggregateException`** appears when **multiple** exceptions happen together — e.g. `Task.WhenAll` or blocking with `.Result` / `.Wait()`:

```csharp
var tasks = new[] { Work1Async(), Work2Async(), Work3Async() };
try
{
    await Task.WhenAll(tasks); // await surfaces only the FIRST exception...
}
catch (Exception ex)
{
    // ...but ALL failures are still available via the tasks:
    foreach (var t in tasks.Where(t => t.IsFaulted))
        Console.WriteLine(t.Exception?.InnerException?.Message);
}

// If you BLOCK instead of await, you get the AggregateException directly:
try
{
    Task.WhenAll(tasks).Wait();
}
catch (AggregateException agg)
{
    foreach (var inner in agg.Flatten().InnerExceptions) // Flatten() un-nests them
        Console.WriteLine(inner.Message);
}
```

Takeaway: prefer `await` (clean single-exception experience). `AggregateException` is mostly seen with `WhenAll` or legacy blocking code.

---

## Best Practices

**Think of it like...** the same disciplines Java teaches, with one extra C#-specific rule (`throw;`).

- **Don't swallow exceptions.** Empty `catch { }` hides bugs. At minimum, log.
  ```csharp
  catch (Exception ex) { /* nothing */ } // ❌ silent failure
  ```
- **Don't use exceptions for control flow.** Don't throw to signal "not found" in a hot path; return a value or use `TryParse`/`Try...` patterns.
  ```csharp
  if (int.TryParse(input, out var n)) { /* use n */ } // ✅ no exception for expected bad input
  ```
- **Catch only what you can handle.** No checked exceptions means *don't* wrap everything in `try/catch` reflexively — let exceptions propagate to a layer that can act.
- **Fail fast.** Validate arguments at method entry with guard clauses (`ThrowIfNull`).
- **Rethrow with `throw;`,** never `throw ex;` (preserves the stack trace).
- **Wrap with context using `InnerException`.** When translating to a domain exception, pass the original as the inner exception so the root cause isn't lost.
  ```csharp
  catch (SqlException ex)
  {
      throw new RepositoryException("Failed to save order.", ex); // ex preserved as InnerException
  }
  ```
- **Catch specific types, not bare `Exception`,** except at top-level boundaries.
- **Don't catch what you can't recover from** (e.g. `OutOfMemoryException`). And note `StackOverflowException` is uncatchable.
- **Throw the most specific built-in exception** that fits (`ArgumentNullException` over `Exception`).

---

## Global Exception Handling in ASP.NET Core

**Think of it like...** a Spring `@ControllerAdvice` / `@ExceptionHandler` — one place to turn uncaught exceptions into proper HTTP responses.

You don't try/catch in every controller. Instead, register centralized handling. Modern (.NET 8) approach uses `IExceptionHandler` plus `ProblemDetails`:

```csharp
// 1) Implement a handler (one place for all unhandled exceptions)
public sealed class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var problem = new ProblemDetails
        {
            Status = StatusCodes.Status500InternalServerError,
            Title = "An unexpected error occurred."
        };
        ctx.Response.StatusCode = problem.Status.Value;
        await ctx.Response.WriteAsJsonAsync(problem, ct);
        return true; // handled
    }
}

// 2) Wire it up in Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
// ...
app.UseExceptionHandler(); // middleware that routes unhandled exceptions to the handler
```

For a junior interview, it's enough to know: **ASP.NET Core has centralized exception-handling middleware (`UseExceptionHandler` / `IExceptionHandler`), so you handle cross-cutting errors in one place instead of per-controller — analogous to Spring's `@ControllerAdvice`.**

---

## Common Interview Questions

**1. What's the biggest difference between Java and C# exceptions?**
C# has **no checked exceptions**. There's no `throws` clause, the compiler never forces you to catch or declare exceptions, and every C# exception behaves like a Java `RuntimeException`.

**2. What is the difference between `throw;` and `throw ex;`?**
`throw;` (bare) rethrows the current exception while **preserving the original stack trace**. `throw ex;` rethrows the same object but **resets the stack trace** to the current line, hiding where it really originated. Always use `throw;` to rethrow.

**3. What's the base class of all exceptions in C#?**
`System.Exception`. Runtime/BCL exceptions derive from `SystemException`; custom exceptions should derive directly from `Exception`.

**4. `SystemException` vs `ApplicationException` — which should custom exceptions use?**
Neither, really — derive custom exceptions directly from `System.Exception`. `ApplicationException` was meant for app exceptions but added no value and is now discouraged.

**5. What is an exception filter? Does Java have one?**
A `catch (Ex e) when (condition)` clause that runs the block only if `condition` is true; otherwise the exception keeps propagating. Java has no equivalent. Bonus: the stack isn't unwound while the filter is evaluated, which helps debugging.

**6. How do you catch multiple exception types in one block?**
C# has no `catch (A | B)` like Java. Use a filter: `catch (Exception e) when (e is A or B)`.

**7. What's the C# equivalent of try-with-resources?**
The `using` statement / `using` declaration, backed by `IDisposable` (`Dispose()` ≈ `close()`). For async cleanup, `await using` with `IAsyncDisposable`.

**8. Difference between `IndexOutOfRangeException` and `ArgumentOutOfRangeException`?**
`IndexOutOfRangeException` is thrown by the CLR for **array** indexing out of bounds. `ArgumentOutOfRangeException` is thrown by **library/your code** when a passed argument (e.g. a `List<T>` index or `Substring` start) is out of the allowed range.

**9. Difference between `NullReferenceException` and `ArgumentNullException`?**
`NullReferenceException` happens when you **dereference** a null (like Java's NPE). `ArgumentNullException` is something **you throw deliberately** when a required argument is null — a guard clause for clearer failures.

**10. How does exception handling work with async/await?**
`await` rethrows the **original** exception from a faulted `Task`, so normal `try/catch` works. When multiple tasks fail (e.g. `Task.WhenAll`), `await` surfaces the first one, but all are available via the tasks; blocking calls (`.Wait()`/`.Result`) throw an `AggregateException` you can `Flatten()`.

**11. Why prefer `ArgumentNullException.ThrowIfNull` and `nameof`?**
`ThrowIfNull` is concise and auto-captures the argument name; `nameof` produces the parameter name at compile time so renames don't leave stale strings in error messages — and the build fails if the name can't be resolved.

**12. Should you use exceptions for control flow? How do you avoid it?**
No — exceptions are for exceptional conditions and are costly. For expected failures use the `Try...` pattern (`int.TryParse`, `Dictionary.TryGetValue`) which returns a bool instead of throwing.

**13. Can you catch a `StackOverflowException`?**
No. Since .NET 2.0 it's uncatchable and terminates the process. (`OutOfMemoryException` is technically catchable but usually unrecoverable.)

**14. How do you preserve the original cause when translating exceptions?**
Pass the caught exception as the `innerException` constructor argument: `throw new MyException("context", ex);`. It's then available via `.InnerException` (≈ Java's `getCause()`).

---

## Quick Reference Cheat Sheet

```text
JAVA → C# QUICK MAP
  Exception ................... System.Exception
  RuntimeException ............ (all C# exceptions are unchecked)
  checked exceptions .......... (do not exist in C#)
  throws clause ............... (none — signatures never list exceptions)
  NullPointerException ........ NullReferenceException
  IllegalArgumentException .... ArgumentException
  IllegalStateException ....... InvalidOperationException
  IndexOutOfBoundsException ... IndexOutOfRangeException (arrays) / ArgumentOutOfRangeException
  NumberFormatException ....... FormatException
  ClassCastException .......... InvalidCastException
  try-with-resources .......... using statement / using declaration
  AutoCloseable.close() ....... IDisposable.Dispose()
  getCause() .................. InnerException
  Objects.requireNonNull ...... ArgumentNullException.ThrowIfNull

THE BIG RULE
  No checked exceptions. No throws clause. Compiler never forces catch/declare.
  Every C# exception behaves like a Java RuntimeException.

RETHROW (interview gotcha)
  throw new X("msg", ex);  // new exception, wrap original as InnerException
  throw;                   // ✅ rethrow, PRESERVES original stack trace
  throw ex;                // ❌ rethrow, RESETS stack trace — anti-pattern

TRY / CATCH / FINALLY
  try { ... }
  catch (SpecificEx e) when (cond) { ... }   // most specific first; 'when' = filter
  catch (Exception e) { ... }                // most general last
  finally { ... }                            // always runs (cleanup)

CATCH MULTIPLE TYPES
  catch (Exception e) when (e is A or B) { } // no Java-style catch(A|B)

RESOURCE CLEANUP
  using (var r = new Resource()) { ... }     // scoped, Dispose() at block end
  using var r = new Resource();              // declaration, Dispose() at scope end
  await using var r = new AsyncResource();   // IAsyncDisposable, DisposeAsync()

GUARD CLAUSES (fail fast)
  ArgumentNullException.ThrowIfNull(arg);
  ArgumentException.ThrowIfNullOrWhiteSpace(text);
  ArgumentOutOfRangeException.ThrowIfNegative(n);
  throw new ArgumentException("msg", nameof(arg));   // use nameof()

COMMON BUILT-INS
  NullReferenceException, ArgumentException, ArgumentNullException,
  ArgumentOutOfRangeException, InvalidOperationException, FormatException,
  InvalidCastException, IndexOutOfRangeException, DivideByZeroException,
  NotImplementedException, NotSupportedException

ASYNC
  await rethrows the ORIGINAL exception → normal try/catch works.
  Task.WhenAll → first surfaces via await; blocking (.Wait/.Result) → AggregateException.
  agg.Flatten().InnerExceptions → all failures.

BEST PRACTICES
  Don't swallow. Don't use for control flow (use TryParse).
  Catch specific types. Fail fast with guards. Rethrow with throw;.
  Wrap with InnerException. Throw the most specific built-in.
  Uncatchable: StackOverflowException.

ASP.NET CORE GLOBAL HANDLING (≈ Spring @ControllerAdvice)
  AddExceptionHandler<GlobalExceptionHandler>(); + app.UseExceptionHandler();
  IExceptionHandler + ProblemDetails for centralized HTTP error responses.
```

---

*Last Updated: 2026-06-16*
