# C# Delegates, Events & Functional C# (for Java Developers)

## Overview

This guide covers the **one big C# topic Java devs consistently underestimate**: delegates and events. In Java you pass behavior around with **functional interfaces** (`Runnable`, `Function<T,R>`, `Consumer<T>`) and lambdas. C# bakes the *same idea* into the language as a first-class type called a **delegate** — a type-safe "method pointer." On top of delegates, C# adds **events** (the built-in observer pattern) and the ready-made generic delegates **`Func<>`, `Action<>`, and `Predicate<>`**.

Interviewers love this area because it reveals whether you *actually* understand C# or just write Java with `;` swapped. Read top to bottom once; then drill the [Interview Questions](#10-common-interview-questions) and [Cheat Sheet](#11-quick-reference-cheat-sheet). All code targets **modern C# (.NET 8+)**.

---

## Table of Contents

1. [Java → C# Quick Mapping](#1-java--c-quick-mapping)
2. [What Is a Delegate?](#2-what-is-a-delegate)
3. [`Func`, `Action`, `Predicate` — the Built-In Delegates](#3-func-action-predicate--the-built-in-delegates)
4. [Lambdas & Anonymous Methods](#4-lambdas--anonymous-methods)
5. [Multicast Delegates](#5-multicast-delegates)
6. [Events: The Observer Pattern, Built In](#6-events-the-observer-pattern-built-in)
7. [The Standard `EventHandler` Pattern](#7-the-standard-eventhandler-pattern)
8. [Closures & Captured Variables (Gotchas)](#8-closures--captured-variables-gotchas)
9. [Where You'll Actually Use This on the Job](#9-where-youll-actually-use-this-on-the-job)
10. [Common Interview Questions](#10-common-interview-questions)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. Java → C# Quick Mapping

**Think of it like...** a universal remote. In Java each button needs its own branded remote (`Runnable`, `Supplier`, `Consumer`...). C# ships **one programmable remote type** — the delegate — and pre-labels the common buttons as `Func`, `Action`, and `Predicate`.

| Java | C# | Notes |
|------|-----|-------|
| Functional interface (`@FunctionalInterface`) | **delegate** type | A named type describing a method signature |
| `Runnable` (no args, no return) | `Action` | Fire-and-forget |
| `Consumer<T>` (arg, no return) | `Action<T>` | Takes input, returns nothing |
| `Supplier<T>` (no arg, returns T) | `Func<T>` | Returns a value |
| `Function<T,R>` | `Func<T, R>` | Last type param is the **return** type |
| `BiFunction<T,U,R>` | `Func<T, U, R>` | Up to 16 inputs |
| `Predicate<T>` | `Predicate<T>` **or** `Func<T, bool>` | Returns `bool` |
| `x -> x + 1` | `x => x + 1` | `=>` instead of `->` |
| Listener interface + `addListener()` | **`event`** keyword | Observer pattern built into the language |
| `PropertyChangeListener` | `EventHandler` / `EventHandler<T>` | Standard event delegate |
| Manually invoke listeners in a loop | **multicast delegate** invocation | `+=` chains handlers automatically |
| Method reference `obj::method` | **method group** `obj.Method` | Assign a method to a delegate by name |

**The single biggest idea:** In Java a lambda *is an object implementing an interface*. In C# a lambda *is a value of a delegate type*. Same concept, one extra vocabulary word (`delegate`).

---

## 2. What Is a Delegate?

A **delegate** is a **type** that represents a method signature. A variable of that type can hold (point to) any method whose parameters and return type match — including lambdas, static methods, and instance methods.

```csharp
// 1) DECLARE a delegate TYPE. Read it like a method signature with a name.
//    "A MathOp is any method that takes two ints and returns an int."
public delegate int MathOp(int a, int b);   // like a Java @FunctionalInterface

public class Calculator
{
    // Methods that MATCH the MathOp signature (two ints -> int):
    public static int Add(int a, int b) => a + b;      // expression-bodied method
    public static int Multiply(int a, int b) => a * b;
}

class Program
{
    static void Main()
    {
        // 2) CREATE a delegate instance pointing at a method (a "method group").
        MathOp op = Calculator.Add;   // no parentheses — we point AT the method, not call it

        // 3) INVOKE it — calling the delegate calls the method it points to.
        int sum = op(3, 4);           // => 7   (shorthand for op.Invoke(3, 4))
        Console.WriteLine(sum);

        // 4) REPOINT it at a different matching method at runtime.
        op = Calculator.Multiply;
        Console.WriteLine(op(3, 4));  // => 12

        // 5) A lambda also matches the signature, so it can be assigned directly:
        op = (a, b) => a - b;
        Console.WriteLine(op(10, 3)); // => 7
    }
}
```

**Why does this exist?** Delegates let you **pass behavior as data** — the foundation of callbacks, event handlers, LINQ, and strategy patterns. It's exactly what Java's functional interfaces do; C# just gives the concept its own keyword.

> **Interview soundbite:** "A delegate is a type-safe function pointer. It's C#'s equivalent of a Java functional interface — a named type describing a method signature that lambdas and method references can satisfy."

---

## 3. `Func`, `Action`, `Predicate` — the Built-In Delegates

You rarely declare your own `delegate` types anymore. .NET ships generic delegates that cover almost every case. **Learn these three cold — they appear constantly in LINQ and APIs.**

```csharp
// Action<...>  = takes 0–16 inputs, returns VOID.  (Java: Runnable / Consumer)
Action greet          = () => Console.WriteLine("Hi");        // no args, no return
Action<string> log    = msg => Console.WriteLine($"LOG: {msg}"); // 1 arg,  no return
greet();              // Hi
log("started");       // LOG: started

// Func<...>    = takes 0–16 inputs, returns a VALUE. LAST type param = return type.
Func<int> roll        = () => 4;                    // no args -> int   (Java: Supplier)
Func<int, int> square = n => n * n;                 // int -> int       (Java: Function)
Func<int, int, int> add = (a, b) => a + b;          // (int,int) -> int (Java: BiFunction)
Console.WriteLine(square(5));   // 25
Console.WriteLine(add(2, 3));   // 5

// Predicate<T> = takes one T, returns BOOL. (Java: Predicate<T>)
Predicate<int> isEven = n => n % 2 == 0;
Console.WriteLine(isEven(4));   // True
// Note: Func<int, bool> means the SAME thing; LINQ uses Func<T,bool>, older APIs use Predicate<T>.
```

**Reading the generic type parameters** (the #1 source of confusion):

| Signature | Means |
|-----------|-------|
| `Action` | `() -> void` |
| `Action<string>` | `(string) -> void` |
| `Action<int, string>` | `(int, string) -> void` |
| `Func<int>` | `() -> int` |
| `Func<string, int>` | `(string) -> int` — **last param is the return type** |
| `Func<int, string, bool>` | `(int, string) -> bool` |
| `Predicate<T>` | `(T) -> bool` |

> **Golden rule:** In `Func<...>`, **the last generic argument is always the return type**; everything before it is inputs. In `Action<...>`, they are *all* inputs (there is no return).

**This is why LINQ works.** `Where` takes a `Func<T, bool>`; `Select` takes a `Func<T, TResult>`:

```csharp
var nums = new[] { 1, 2, 3, 4, 5 };
var evens = nums.Where(n => n % 2 == 0);   // n => ... IS a Func<int, bool>
var squares = nums.Select(n => n * n);     // n => ... IS a Func<int, int>
```

---

## 4. Lambdas & Anonymous Methods

A **lambda** is an inline, unnamed method. It's the everyday way to create a delegate value.

```csharp
// Expression lambda — single expression, value is returned implicitly:
Func<int, int> dbl = x => x * 2;

// Statement lambda — a block with { }; use 'return' explicitly:
Func<int, string> classify = x =>
{
    if (x < 0) return "negative";
    if (x == 0) return "zero";
    return "positive";
};

// Multiple / zero parameters need parentheses:
Action hello       = () => Console.WriteLine("hello");
Func<int,int,int> a = (x, y) => x + y;

// Method group conversion — pass a method by NAME (Java's obj::method):
Func<int, int, int> add = Calculator.Add;      // instead of (a,b) => Calculator.Add(a,b)
var names = new[] { "bob", "amy" };
names.Select(char.ToUpper);                     // BUT: char.ToUpper matches Func<char,char>...
List<string> upper = names.Select(s => s.ToUpper()).ToList();
```

`static` lambdas (C# 9+) prevent accidentally capturing variables (a perf + correctness win):

```csharp
Func<int, int> pure = static x => x * x;   // 'static' => cannot capture outer variables
```

> **Java parallel:** `x => x * 2` is exactly Java's `x -> x * 2`. The only visible difference is `=>` vs `->`.

---

## 5. Multicast Delegates

Here's something Java's functional interfaces **cannot** do: a single C# delegate can point at **multiple methods at once** and invoke them all in order. This is what makes `event` work.

```csharp
Action pipeline = () => Console.WriteLine("Step 1");
pipeline += () => Console.WriteLine("Step 2");   // += ADDS a handler
pipeline += () => Console.WriteLine("Step 3");

pipeline();   // invokes ALL three, in order:  Step 1 / Step 2 / Step 3

pipeline -= () => Console.WriteLine("Step 2");    // -= REMOVES (must be the same delegate ref!)
```

**Two gotchas interviewers probe:**

1. **Return values are lost.** For a multicast `Func<int>`, invoking it returns only the **last** handler's result — everything else is discarded. So multicast really only makes sense for `void`-returning delegates (`Action`/events).
2. **An exception in one handler stops the rest.** If handler #2 throws, handler #3 never runs. (To be robust, iterate `GetInvocationList()` and try/catch each.)

```csharp
Func<int> f = () => 1;
f += () => 2;
Console.WriteLine(f());   // 2  — only the LAST result survives
```

---

## 6. Events: The Observer Pattern, Built In

An **event** is a delegate field with a safety wrapper. It lets a class **broadcast notifications** to any number of subscribers without knowing who they are — the Observer pattern, with language support.

**Think of it like...** a newsletter. The publisher (`Button`) doesn't know its readers. Readers **subscribe** (`+=`), the publisher **sends an issue** (raises the event), and every subscriber's handler runs. Readers can **unsubscribe** (`-=`) anytime.

```csharp
public class Button
{
    // 1) DECLARE the event. It's an Action-typed broadcast slot.
    //    'event' restricts outsiders to only += and -= (they can't invoke or overwrite it).
    public event Action? Clicked;

    // 2) The publisher RAISES the event when something happens.
    public void SimulateClick()
    {
        Console.WriteLine("Button was clicked");
        Clicked?.Invoke();   // ?. => only fire if there's at least one subscriber (avoids NRE)
    }
}

class Program
{
    static void Main()
    {
        var button = new Button();

        // 3) SUBSCRIBERS attach handlers with +=  (like addActionListener in Swing)
        button.Clicked += () => Console.WriteLine("Handler A: play sound");
        void logHandler() => Console.WriteLine("Handler B: write to log");
        button.Clicked += logHandler;

        button.SimulateClick();
        // Button was clicked / Handler A: play sound / Handler B: write to log

        // 4) UNSUBSCRIBE — important for long-lived publishers to avoid memory leaks
        button.Clicked -= logHandler;
        button.SimulateClick();   // Handler A only now
    }
}
```

**Why `event` and not just a public delegate field?** The `event` keyword enforces encapsulation: outside code may only `+=` / `-=`. It **cannot** overwrite all subscribers with `=`, and it **cannot** raise the event itself — only the declaring class can. That protection is the whole point.

> **Memory-leak gotcha (commonly asked):** A subscriber stays alive as long as the publisher holds a reference to its handler. If a long-lived publisher (e.g., a singleton) subscribes a short-lived object and never unsubscribes, the subscriber can't be garbage-collected. **Always `-=` when done.**

---

## 7. The Standard `EventHandler` Pattern

.NET has a **convention** for events used across the whole framework (UI, ASP.NET, etc.). Interviewers expect you to recognize it. Instead of `Action`, use **`EventHandler<TEventArgs>`**, which passes `(object sender, TEventArgs e)`.

```csharp
// 1) A data class carrying event details, by convention named ...EventArgs.
public class OrderPlacedEventArgs : EventArgs
{
    public int OrderId { get; }
    public decimal Total { get; }
    public OrderPlacedEventArgs(int orderId, decimal total)
    {
        OrderId = orderId;
        Total = total;
    }
}

public class OrderService
{
    // 2) EventHandler<T> is a built-in delegate: (object? sender, T e) -> void
    public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

    public void PlaceOrder(int id, decimal total)
    {
        // ... save order ...

        // 3) Raise it. 'this' is the sender; the args carry the payload.
        //    The temp-variable copy is the classic thread-safe raise pattern.
        OrderPlaced?.Invoke(this, new OrderPlacedEventArgs(id, total));
    }
}

class Program
{
    static void Main()
    {
        var svc = new OrderService();

        // 4) Handler signature MUST match: (object? sender, OrderPlacedEventArgs e)
        svc.OrderPlaced += (sender, e) =>
            Console.WriteLine($"Email sent for order {e.OrderId}, total {e.Total:C}");
        svc.OrderPlaced += (sender, e) =>
            Console.WriteLine($"Inventory updated for order {e.OrderId}");

        svc.PlaceOrder(101, 59.99m);
    }
}
```

**Why the `(sender, args)` shape?** It's the framework-wide standard so any handler can identify *who* raised the event and receive strongly-typed *details*. When you wire up a button in a real app, its `Click` event is exactly this pattern.

---

## 8. Closures & Captured Variables (Gotchas)

A lambda can **capture** variables from its surrounding scope — a **closure**. This behaves like Java, with one crucial difference around loops.

```csharp
int factor = 10;
Func<int, int> scale = x => x * factor;   // captures 'factor' by REFERENCE, not value
factor = 20;
Console.WriteLine(scale(5));   // 100? NO -> 100 is wrong; it's 100... actually 5*20 = 100
// It prints 100 because 'factor' is captured by reference and was updated to 20 before the call.
```

Wait — read that carefully: **C# captures variables by reference**, so the lambda sees the *latest* value (`20`), printing `100`. This differs from Java, where captured locals must be *effectively final*. C# has no such restriction, which creates the classic **loop-variable trap**:

```csharp
// PRE-C# 5 / other languages had this bug; modern C# 'foreach' is SAFE, but a 'for' loop is NOT:
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
    actions.Add(() => Console.WriteLine(i));   // all capture the SAME 'i'

foreach (var a in actions) a();   // prints 3, 3, 3  (i is 3 after the loop ends)

// FIX: copy into a fresh variable inside the loop so each lambda captures its own:
var fixedActions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    int copy = i;                                  // new variable each iteration
    fixedActions.Add(() => Console.WriteLine(copy));
}
foreach (var a in fixedActions) a();   // prints 0, 1, 2

// NOTE: 'foreach' loop variables ARE captured correctly per-iteration since C# 5 — no copy needed.
```

> **Interview trap:** "What does this loop print?" If it's a `for` loop capturing `i`, the answer is the final value repeated. If it's a `foreach`, it prints each element correctly. Know the difference.

---

## 9. Where You'll Actually Use This on the Job

You won't declare custom `delegate` types often, but you'll use the *concept* constantly:

- **LINQ** — every `.Where()`, `.Select()`, `.OrderBy()` takes a `Func<>` delegate.
- **ASP.NET Core middleware** — the pipeline is literally `Func<RequestDelegate, RequestDelegate>`; `app.Use(async (context, next) => ...)` passes a delegate.
- **Dependency injection factories** — `services.AddScoped(sp => new Thing(...))` registers a `Func<IServiceProvider, T>`.
- **Callbacks & retries** — Polly, `Timer`, `Task.Run(() => ...)` all take delegates.
- **Events** — `INotifyPropertyChanged`, UI button clicks, `AppDomain.CurrentDomain.ProcessExit`, custom domain events.
- **Testing** — Moq's `Setup(x => x.Method())` and `It.Is<T>(predicate)` are delegate/expression based.

Being fluent here is what makes idiomatic C# code *click* instead of reading like translated Java.

---

## 10. Common Interview Questions

### Q: What is a delegate?
A **type-safe function pointer** — a named type describing a method signature. A delegate variable can hold any method (lambda, static, or instance) with a matching signature, and invoking the delegate calls that method. It's C#'s equivalent of a Java functional interface.

### Q: What's the difference between `Func`, `Action`, and `Predicate`?
- **`Action<...>`** returns `void`; all type params are inputs.
- **`Func<...>`** returns a value; the **last** type param is the return type, the rest are inputs.
- **`Predicate<T>`** takes one `T` and returns `bool` (equivalent to `Func<T, bool>`).

### Q: What is a multicast delegate?
A delegate instance pointing at **multiple methods**. Adding handlers with `+=` chains them; invoking runs them all in order. For non-void delegates only the last return value is kept, and an exception in one handler stops the remaining ones. Events are built on multicast delegates.

### Q: What's the difference between a delegate and an event?
An `event` is a **restricted wrapper** around a delegate. Outside the declaring class, subscribers can only `+=` / `-=`; they can't overwrite the invocation list with `=` or raise the event. This enforces the publisher/subscriber (Observer) contract. A plain public delegate field gives outsiders full control — which is unsafe.

### Q: How do you avoid a `NullReferenceException` when raising an event?
Use the null-conditional invoke: `MyEvent?.Invoke(this, args);`. If there are no subscribers the event field is `null`, and `?.` skips the call. Capturing to a local first (`var handler = MyEvent; handler?.Invoke(...)`) additionally guards against a subscriber unsubscribing on another thread mid-raise.

### Q: How can events cause memory leaks?
As long as a publisher holds a handler referencing a subscriber, that subscriber can't be garbage-collected. A long-lived publisher (e.g., a static/singleton) subscribing a short-lived object without unsubscribing keeps it alive forever. **Always `-=` when the subscriber is done** (or use weak event patterns).

### Q: What is a closure? Does C# capture by value or by reference?
A closure is a lambda that captures variables from its enclosing scope. **C# captures by reference** (the variable itself, not its current value), so the lambda sees later changes. This causes the classic `for`-loop trap where all lambdas share one `i`; fix it by copying into a per-iteration local. (`foreach` variables are captured correctly since C# 5.)

### Q: What is a method group?
Referring to a method by name **without** parentheses (`Console.WriteLine`, `Calculator.Add`) so it can be assigned to a compatible delegate — C#'s equivalent of Java's `obj::method` reference. The compiler converts the method group into a delegate.

### Q: Why does LINQ rely on delegates?
LINQ operators take delegates as parameters: `Where` takes `Func<T, bool>`, `Select` takes `Func<T, TResult>`. The lambda you pass *is* a delegate value. That's how you inject custom behavior (filtering, projection) into generic operators.

---

## 11. Quick Reference Cheat Sheet

```
DELEGATE = a type describing a method signature (type-safe function pointer)

BUILT-IN DELEGATES:
  Action                 ()            -> void
  Action<T>              (T)           -> void
  Action<T1,T2>          (T1,T2)       -> void        (up to 16 inputs)
  Func<TResult>          ()            -> TResult
  Func<T,TResult>        (T)           -> TResult      LAST param = return type
  Func<T1,T2,TResult>    (T1,T2)       -> TResult
  Predicate<T>           (T)           -> bool         (== Func<T,bool>)

LAMBDAS:
  x => x + 1                   expression lambda (implicit return)
  (a, b) => a + b             multiple params
  () => DoThing()             no params
  x => { ...; return y; }     statement lambda (explicit return)
  static x => x * x           cannot capture (C# 9+)
  Console.WriteLine           method group (like Java obj::method)

MULTICAST:
  d += handler;   // add
  d -= handler;   // remove
  d?.Invoke(a);   // safe invoke (null if no subscribers)
  - only LAST return value kept; exception stops remaining handlers

EVENT (Observer pattern):
  public event EventHandler<TArgs>? Something;   // declare
  Something?.Invoke(this, args);                 // raise (inside the class only)
  obj.Something += Handler;                      // subscribe
  obj.Something -= Handler;                      // unsubscribe (prevents leaks!)
  Handler signature: void H(object? sender, TArgs e)

JAVA -> C#:
  Runnable          -> Action
  Consumer<T>       -> Action<T>
  Supplier<T>       -> Func<T>
  Function<T,R>     -> Func<T,R>
  Predicate<T>      -> Predicate<T> / Func<T,bool>
  x -> x+1          -> x => x+1
  obj::method       -> obj.Method (method group)
  Listener interface-> event keyword

GOLDEN RULES:
  1. Func's LAST generic arg is the return type; Action has no return.
  2. Prefer Func/Action/Predicate over declaring custom delegates.
  3. Use 'event' (not a public delegate) to expose notifications safely.
  4. Always raise events with ?.Invoke; always -= to avoid leaks.
  5. Beware for-loop closures: copy the loop var into a local.
```

---

*Related guides: `CSharp_LINQ_Guide.md` (delegates in action), `DependencyInjection_And_Middleware_Guide.md` (delegate-based pipeline), `CSharp_Async_Await_Concurrency.md` (delegates + Task).*

*Last Updated: 2026-07-09*
