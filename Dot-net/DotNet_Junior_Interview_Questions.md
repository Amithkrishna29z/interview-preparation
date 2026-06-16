# .NET / C# Junior Developer Interview Questions

## Overview

This guide is a large bank of real interview questions for **junior .NET/C# developer** roles, written for someone who already knows **Java**. Each answer is concise but complete, uses modern **C#/.NET 8+**, and includes **Java parallel** notes wherever a concept maps cleanly onto something you already know. Skim the Table of Contents, drill the categories you're weak on, and use the Cheat Sheet for last-minute review.

---

## Table of Contents

1. [C# Language Basics](#c-language-basics)
2. [OOP in C#](#oop-in-c)
3. [Generics & Type System](#generics--type-system)
4. [Collections](#collections)
5. [LINQ](#linq)
6. [Async/Await & Threading](#asyncawait--threading)
7. [Exception Handling](#exception-handling)
8. [Memory & Garbage Collection](#memory--garbage-collection)
9. [ASP.NET Core](#aspnet-core)
10. [Entity Framework Core](#entity-framework-core)
11. [Dependency Injection](#dependency-injection)
12. [Security](#security)
13. [Testing](#testing)
14. [.NET Ecosystem & Tooling](#net-ecosystem--tooling)
15. [Behavioral / Practical](#behavioral--practical)
16. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## C# Language Basics

### Q: What is the difference between value types and reference types?

A **value type** holds its data directly; a **reference type** holds a reference (pointer) to data on the heap. Value types include `int`, `double`, `bool`, `char`, `struct`, and `enum`. Reference types include `class`, `string`, arrays, delegates, and `object`. Assigning a value type copies the data; assigning a reference type copies the reference (both point to the same object).

```csharp
int a = 5;
int b = a;      // b is a separate copy
b = 10;         // a is still 5

int[] x = { 1, 2 };
int[] y = x;    // y references the SAME array
y[0] = 99;      // x[0] is now 99 too
```

**Java parallel:** Same model — primitives (`int`, `double`) are value types; objects are reference types. Java just doesn't let you define your own value types easily (records/`struct` differ).

### Q: Where are value types and reference types stored?

Local value types usually live on the **stack**; reference type instances live on the **heap** (the variable/reference itself may be on the stack). But this is an implementation detail — a value type that is a *field* of a class lives on the heap inside that object. Don't over-index on "value = stack" in interviews; say "value types are copied by value, reference types by reference."

**Java parallel:** Identical mental model (primitives on stack/inline, objects on heap).

### Q: What is boxing and unboxing?

**Boxing** wraps a value type in an object on the heap so it can be treated as a reference type (`object` or an interface). **Unboxing** extracts the value back out. Boxing allocates memory and costs performance.

```csharp
int n = 42;
object boxed = n;        // boxing: allocates on the heap
int unboxed = (int)boxed; // unboxing: explicit cast required
```

**Java parallel:** Same idea as autoboxing `int` ⇄ `Integer`, but in C# boxing targets `object`, not a wrapper class.

### Q: Why are strings immutable in C#?

A `string` cannot be changed after creation. Methods like `Replace` or `ToUpper` return a **new** string. Immutability enables safe sharing, hashing, string interning, and thread safety. For heavy concatenation in loops, use `StringBuilder` to avoid allocating many intermediate strings.

```csharp
string s = "hello";
s.ToUpper();          // returns "HELLO" but s is unchanged
s = s.ToUpper();      // reassigning is how you "change" it
```

**Java parallel:** Identical — Java `String` is immutable, and `StringBuilder` exists for the same reason.

### Q: What does `var` do? Is C# dynamically typed?

`var` is **implicit static typing** — the compiler infers the type at compile time from the right-hand side. The variable is still strongly typed; you just don't spell out the type. It is **not** `dynamic`.

```csharp
var count = 5;          // compiler infers int
var name = "Alice";     // string
// count = "text";      // compile error — still strongly typed
```

**Java parallel:** Same as Java's `var` (local variable type inference, Java 10+).

### Q: What is the difference between `const` and `readonly`?

`const` is a compile-time constant — its value is baked into callers at compile time and must be set inline (primitives/strings only). `readonly` is a runtime constant — it can be set in the declaration or the constructor, and can differ per instance.

```csharp
public const double Pi = 3.14159;       // compile-time, implicitly static
public readonly DateTime Created;        // set once, in constructor
public Thing() => Created = DateTime.Now;
```

**Java parallel:** `const` ≈ `static final` compile-time constant; `readonly` ≈ a `final` field assigned in the constructor.

### Q: What are properties and why use them over fields?

A **property** is a member that looks like a field but is backed by `get`/`set` accessors (methods). Properties let you add validation, computation, or change implementation without breaking callers. Auto-properties generate the backing field for you.

```csharp
public string Name { get; set; }              // auto-property
public string Email { get; private set; }     // read-only to outside
public int Age
{
    get => _age;
    set => _age = value < 0 ? 0 : value;      // validation in setter
}
```

**Java parallel:** Replaces the Java `getName()/setName()` boilerplate; the language gives you getters/setters as first-class syntax.

### Q: What are nullable value types (`int?`)?

By default value types cannot be `null`. `int?` (shorthand for `Nullable<int>`) lets a value type also represent "no value." Use `.HasValue` / `.Value`, or the null-coalescing operator `??`.

```csharp
int? age = null;
if (age.HasValue) { /* ... */ }
int safe = age ?? 0;     // 0 if null
```

**Java parallel:** Similar to `Integer` being nullable while `int` is not — but `int?` is still a value type with no boxing required.

### Q: What are nullable reference types (NRTs)?

A compile-time feature (enabled by `<Nullable>enable</Nullable>`) where reference types are non-nullable by default; you write `string?` to allow null. The compiler warns about possible null dereferences. It's static analysis only — it doesn't change runtime behavior.

```csharp
string name = "Bob";    // must not be null
string? maybe = null;   // explicitly nullable
int len = maybe.Length; // compiler warning: possible null
```

**Java parallel:** Like using `@Nullable`/`@NonNull` annotations or `Optional`, but built into the type system and enforced by the compiler.

### Q: What is the difference between `==` and `Equals()`?

For reference types, `==` defaults to **reference equality** (same object), while `Equals()` can be overridden for **value equality**. `string` overloads `==` to compare contents. Value types compare by value for both.

```csharp
string a = "hi";
string b = "hi";
bool r1 = a == b;        // true — string overloads == for content
object o1 = a, o2 = b;
bool r2 = o1 == o2;      // still true (string operator), but reference compare via ReferenceEquals can differ
bool r3 = a.Equals(b);   // true — value equality
```

**Java parallel:** `==` ≈ Java `==` (reference for objects), `Equals()` ≈ Java `.equals()`. Note C# `string ==` does content comparison (Java `==` on Strings does NOT).

### Q: What is string interpolation?

`$"..."` syntax embeds expressions directly in a string literal.

```csharp
string name = "Sam";
int age = 30;
string msg = $"{name} is {age} years old"; // "Sam is 30 years old"
```

**Java parallel:** Like `String.format` / Java 21 string templates, but cleaner and compile-checked.

### Q: What is the `nameof` operator?

`nameof(x)` returns the variable/member name as a string at compile time. Great for exceptions and logging because it survives refactors.

```csharp
throw new ArgumentNullException(nameof(input)); // "input"
```

**Java parallel:** No direct equivalent; avoids hardcoded magic strings.

### Q: What are `default` and the `default` literal?

`default(T)` (or just `default`) gives the default value of a type: `0` for numerics, `false` for `bool`, `null` for reference types.

```csharp
int x = default;        // 0
string? s = default;    // null
```

**Java parallel:** Same defaults as Java field initialization, but you can request it explicitly for generics.

---

## OOP in C#

### Q: What is the difference between `virtual`, `override`, and `new`?

`virtual` marks a method as overridable. `override` provides a new implementation in a derived class (polymorphic). `new` **hides** a base member instead of overriding it — which method runs depends on the **compile-time type** of the reference, not the runtime object.

```csharp
class Base { public virtual void Hi() => Console.WriteLine("Base"); }
class Over : Base { public override void Hi() => Console.WriteLine("Over"); }
class Hide : Base { public new void Hi() => Console.WriteLine("Hide"); }

Base b1 = new Over(); b1.Hi(); // "Over"  — polymorphic
Base b2 = new Hide(); b2.Hi(); // "Base"  — hidden, uses ref type
```

**Java parallel:** `virtual`+`override` ≈ normal Java overriding (Java methods are virtual by default). `new` has no clean Java equivalent — it's method hiding.

### Q: Why aren't C# methods virtual by default?

Performance and safety. Non-virtual calls can be inlined and don't need vtable lookups, and the author opts into extensibility explicitly. You must mark a method `virtual` to allow overriding.

**Java parallel:** Opposite of Java, where instance methods are virtual unless marked `final`/`static`/`private`. In C# the default is effectively "final"; `virtual` is opt-in.

### Q: What is the difference between an abstract class and an interface?

An **abstract class** can have state (fields), constructors, and implemented methods; a class can inherit only one. An **interface** is a contract — historically no state/implementation, though C# 8+ allows default interface methods. A class can implement many interfaces. Use an abstract class for shared base behavior, an interface for a capability/contract.

```csharp
abstract class Shape { public abstract double Area(); public string Name => "Shape"; }
interface IDrawable { void Draw(); }
class Circle : Shape, IDrawable { public override double Area() => 3.14; public void Draw() {} }
```

**Java parallel:** Same distinction; default interface methods ≈ Java 8 `default` methods. Single class inheritance, multiple interface implementation — identical.

### Q: What does `sealed` do?

`sealed` prevents a class from being inherited, or prevents an `override`d member from being further overridden. Use it to lock down a type or enable optimizations.

```csharp
public sealed class FinalThing { }       // cannot be subclassed
```

**Java parallel:** `sealed class` ≈ Java `final class` (note: Java 17 `sealed` means something different — a restricted permitted-subclass list).

### Q: What is the difference between a `struct` and a `class`?

`struct` is a **value type** (copied by value, usually stack/inline, no inheritance except interfaces, can't be null unless nullable). `class` is a **reference type** (heap, supports inheritance, can be null). Use `struct` for small, immutable, value-like data (e.g., a `Point`).

```csharp
struct Point { public int X, Y; }   // copied by value
class Person { public string Name; } // reference semantics
```

**Java parallel:** Java has no user-defined value types (until Project Valhalla). A `struct` is roughly "a small immutable data holder copied by value."

### Q: What are records and why use them?

A `record` is a reference type (or `record struct`) designed for **immutable data** with value-based equality, `with` expressions for copies, and auto-generated `ToString`, `Equals`, `GetHashCode`. Great for DTOs and domain values.

```csharp
public record Person(string Name, int Age);   // positional record
var p1 = new Person("Ann", 30);
var p2 = p1 with { Age = 31 };                 // non-destructive copy
bool same = p1 == new Person("Ann", 30);       // true — value equality
```

**Java parallel:** Like Java `record`, but C# records add value-based `Equals`, `with` copy, and can be reference or value types.

### Q: What are the access modifiers in C#?

- `public` — accessible anywhere.
- `private` — only within the same type.
- `protected` — the type and its subclasses.
- `internal` — anywhere in the **same assembly**.
- `protected internal` — same assembly OR subclasses.
- `private protected` — same assembly AND subclasses.

**Java parallel:** `internal` is the big one with no Java equivalent (Java's package-private is closest, but scoped to a package, not an assembly). `public`/`private`/`protected` map closely.

### Q: What is the default access modifier?

Class members default to `private`. Top-level types default to `internal`. Always be explicit in interviews.

**Java parallel:** Java members default to package-private; C# defaults to `private` (more restrictive).

### Q: Does C# support multiple inheritance?

Not for classes (single base class only), but a type can implement multiple interfaces. Default interface methods provide a limited form of mix-in behavior.

**Java parallel:** Identical rule.

### Q: What is the difference between `is` and `as`?

`is` tests type (and can pattern-match into a variable). `as` attempts a cast and returns `null` on failure (no exception) for reference/nullable types.

```csharp
if (obj is string s) { /* s is typed string */ }
string? maybe = obj as string;   // null if not a string
```

**Java parallel:** `is` ≈ `instanceof` (with pattern matching like Java 16+). `as` ≈ a cast that returns null instead of throwing `ClassCastException`.

### Q: What is the difference between overloading and overriding?

**Overloading** = same method name, different parameter lists, resolved at compile time. **Overriding** = a `virtual`/`abstract` base method replaced in a derived class, resolved at runtime.

**Java parallel:** Identical concepts.

---

## Generics & Type System

### Q: What are generics and why use them?

Generics let you write type-safe, reusable code parameterized by type, avoiding casts and boxing. `List<int>` is type-checked at compile time.

```csharp
public T First<T>(IEnumerable<T> items) => items.First();
List<int> nums = new() { 1, 2, 3 };   // no casting, no boxing
```

**Java parallel:** Same purpose as Java generics — but see the erasure difference below.

### Q: How do C# generics differ from Java generics (type erasure)?

C# has **reified generics** — type information is preserved at runtime. You can do `typeof(T)`, `new T()` (with constraint), check `list is List<int>`, and there's no boxing for value-type generics. Java erases generic types at runtime (`List<String>` becomes `List`), so you can't inspect `T` at runtime.

```csharp
public void Show<T>() => Console.WriteLine(typeof(T).Name); // works at runtime
```

**Java parallel:** This is a major difference — Java erases, C# reifies. Java can't write `new T()` or `T[]` directly; C# can with constraints.

### Q: What are generic constraints?

Constraints restrict what `T` can be, unlocking operations on it.

```csharp
// where T : class            -> reference type
// where T : struct           -> value type
// where T : new()            -> has parameterless constructor
// where T : IComparable<T>   -> implements interface
// where T : BaseClass        -> derives from a base
public T Create<T>() where T : new() => new T();
```

**Java parallel:** `where T : IComparable<T>` ≈ Java `<T extends Comparable<T>>`. Java has no `new()` or `struct`/`class` constraints.

### Q: What is covariance and contravariance?

**Covariance** (`out`) lets you use a more-derived type than specified — e.g., `IEnumerable<string>` is assignable to `IEnumerable<object>`. **Contravariance** (`in`) lets you use a less-derived type — e.g., `Action<object>` is assignable to `Action<string>`. They apply to interfaces and delegates.

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;   // covariant (out)

Action<object> printObj = o => Console.WriteLine(o);
Action<string> printStr = printObj;      // contravariant (in)
```

**Java parallel:** Roughly like Java wildcards `? extends T` (covariance) and `? super T` (contravariance), but in C# variance is declared on the type with `in`/`out`, not at use sites.

### Q: What is the `object` type and `dynamic`?

`object` is the root base type of everything. `dynamic` bypasses compile-time type checking — resolution happens at runtime (useful for interop/reflection but loses safety).

```csharp
dynamic d = "hello";
d.Foo();   // compiles, throws at runtime if Foo doesn't exist
```

**Java parallel:** `object` ≈ `java.lang.Object`. `dynamic` has no direct Java equivalent (closest is reflection).

---

## Collections

### Q: What is the difference between an array and a `List<T>`?

An **array** has a fixed size set at creation. A `List<T>` is a resizable, dynamic collection backed by an array internally; it grows as you add items. Prefer `List<T>` unless you need a fixed-size, low-overhead buffer.

```csharp
int[] arr = new int[3];           // fixed length 3
List<int> list = new() { 1, 2 };
list.Add(3);                       // grows automatically
```

**Java parallel:** `List<T>` ≈ `ArrayList<T>`. C# arrays ≈ Java arrays.

### Q: What is a `Dictionary<TKey, TValue>`?

A hash-based key/value collection with average O(1) lookup. Keys must be unique. Use `TryGetValue` to avoid `KeyNotFoundException`.

```csharp
var ages = new Dictionary<string, int> { ["Ann"] = 30 };
if (ages.TryGetValue("Ann", out int age)) { /* age = 30 */ }
```

**Java parallel:** `Dictionary` ≈ `HashMap`. `TryGetValue` is the idiomatic "get or default" pattern (Java's `getOrDefault`).

### Q: What is the difference between `IEnumerable`, `ICollection`, and `IList`?

They form a capability hierarchy:
- `IEnumerable<T>` — can iterate (foreach) only; lazy, forward-only.
- `ICollection<T>` — adds `Count`, `Add`, `Remove`, `Contains`.
- `IList<T>` — adds index access (`this[i]`), `Insert`, `RemoveAt`.

Expose the **least capable** interface that callers need (usually `IEnumerable<T>` for read-only iteration).

**Java parallel:** `IEnumerable` ≈ `Iterable`, `ICollection`/`IList` ≈ `Collection`/`List`.

### Q: When would you use a `HashSet<T>`?

For a collection of **unique** items with fast O(1) `Contains`/`Add`. Use it for membership tests and de-duplication.

**Java parallel:** `HashSet<T>` ≈ `HashSet`.

### Q: What collection would you use for a queue or stack?

`Queue<T>` (FIFO, `Enqueue`/`Dequeue`) and `Stack<T>` (LIFO, `Push`/`Pop`).

**Java parallel:** `Queue<T>` ≈ `ArrayDeque` as a queue; `Stack<T>` ≈ `ArrayDeque` as a stack (avoid the legacy Java `Stack`).

### Q: What is `IReadOnlyList<T>` / `IReadOnlyCollection<T>` for?

Read-only views that prevent callers from modifying the collection through the interface. Good for exposing internal collections safely.

**Java parallel:** Like returning `Collections.unmodifiableList(...)` or an immutable collection.

### Q: Are .NET collections thread-safe?

The standard collections (`List<T>`, `Dictionary<TKey,TValue>`) are **not** thread-safe for writes. Use the `System.Collections.Concurrent` namespace (`ConcurrentDictionary`, `ConcurrentQueue`, etc.) for concurrent access.

**Java parallel:** `ConcurrentDictionary` ≈ `ConcurrentHashMap`; plain `List` ≈ non-synchronized `ArrayList`.

---

## LINQ

### Q: What is LINQ?

**Language Integrated Query** — a unified query syntax over collections, databases, XML, etc. Two styles: method syntax (`.Where(...).Select(...)`) and query syntax (`from x in xs where ... select ...`). Operates on `IEnumerable<T>` / `IQueryable<T>`.

```csharp
var adults = people.Where(p => p.Age >= 18)
                   .Select(p => p.Name)
                   .ToList();
```

**Java parallel:** Method syntax ≈ Java Streams (`stream().filter().map().collect()`).

### Q: What is deferred (lazy) execution?

Most LINQ operators don't run when defined — they execute when **enumerated** (e.g., `foreach`, `ToList`, `Count`, `First`). This means the query reflects the latest data and can be composed efficiently, but re-enumerating re-runs it.

```csharp
var q = numbers.Where(n => n > 2); // not executed yet
numbers.Add(10);
var result = q.ToList();           // executes now, sees the new 10
```

**Java parallel:** Like Java Streams being lazy until a terminal operation — but a LINQ query variable can be re-enumerated, unlike a one-shot Stream.

### Q: What is the difference between `IEnumerable<T>` and `IQueryable<T>`?

`IEnumerable<T>` executes in memory (LINQ-to-Objects) — operators run as delegates. `IQueryable<T>` builds an **expression tree** that a provider (like EF Core) translates to another language (SQL). Filtering on `IQueryable` happens in the database; switching to `IEnumerable` (e.g., `AsEnumerable()`) pulls data into memory first.

```csharp
// IQueryable: WHERE runs in SQL
var active = db.Users.Where(u => u.IsActive).ToList();
// IEnumerable: would load ALL users, then filter in memory
var bad = db.Users.AsEnumerable().Where(u => u.IsActive).ToList();
```

**Java parallel:** No direct equivalent; conceptually like JPA Criteria/JPQL (DB-side) vs. filtering a `List` in memory.

### Q: What is the difference between `First`, `FirstOrDefault`, `Single`, and `SingleOrDefault`?

- `First` — first match; **throws** if none.
- `FirstOrDefault` — first match or `default` (e.g., `null`/`0`) if none.
- `Single` — exactly one match; **throws** if zero OR more than one.
- `SingleOrDefault` — one match or default; **throws** if more than one.

Use `Single` when exactly one is expected (enforces an invariant); `First` when you just want the top of an ordered set.

**Java parallel:** `FirstOrDefault` ≈ `stream().findFirst().orElse(null)`. `Single` has no clean Stream equivalent (it asserts cardinality).

### Q: What are common LINQ operators?

- `Where` — filter. `Select` — map/project. `SelectMany` — flatten.
- `OrderBy`/`ThenBy` — sort. `GroupBy` — group.
- `Any`/`All`/`Count` — quantify/count. `Sum`/`Average`/`Min`/`Max` — aggregate.
- `Take`/`Skip` — paging. `Distinct` — de-dup. `ToList`/`ToArray`/`ToDictionary` — materialize.

```csharp
var byCity = people.GroupBy(p => p.City)
                   .Select(g => new { City = g.Key, Count = g.Count() });
```

**Java parallel:** `Where`≈`filter`, `Select`≈`map`, `SelectMany`≈`flatMap`, `GroupBy`≈`Collectors.groupingBy`, `ToList`≈`collect(toList())`.

### Q: What does `Select` vs `SelectMany` do?

`Select` projects each element to one result (1→1). `SelectMany` projects each element to a sequence and flattens them all into one (1→many→flat).

```csharp
var allTags = posts.SelectMany(p => p.Tags); // flatten lists of tags
```

**Java parallel:** `Select`≈`map`, `SelectMany`≈`flatMap`.

### Q: Why can re-using a LINQ query be a performance trap?

Because of deferred execution, each enumeration re-runs the query. Calling `.Count()` then `foreach` over the same `IEnumerable` executes it twice (and for `IQueryable`, hits the DB twice). Materialize once with `.ToList()` if you'll use it multiple times.

**Java parallel:** A Java Stream can only be consumed once and throws on reuse — C# silently re-runs, which can be a subtle perf bug.

---

## Async/Await & Threading

### Q: What is `async`/`await`?

`async`/`await` is syntax for asynchronous, non-blocking code. An `async` method returns a `Task` (or `Task<T>`); `await` suspends the method until the awaited task completes, freeing the thread to do other work, then resumes. It's primarily about **scalability** (not blocking threads on I/O), not raw speed.

```csharp
public async Task<string> GetDataAsync()
{
    using var client = new HttpClient();
    string body = await client.GetStringAsync("https://api.example.com");
    return body; // thread is freed while waiting on the network
}
```

**Java parallel:** Closest to `CompletableFuture` chains or Java 21 virtual threads, but `async`/`await` is language-level syntactic sugar around state machines.

### Q: What is the difference between a `Task` and a `Thread`?

A `Thread` is an OS-level thread you manage directly. A `Task` is a higher-level abstraction representing a unit of work, scheduled on the thread pool, that may or may not use a dedicated thread (async I/O often uses none while waiting). Prefer `Task`/async for most work.

**Java parallel:** `Task` ≈ `Future`/`CompletableFuture`; `Thread` ≈ `java.lang.Thread`. Use the high-level abstraction.

### Q: When does async actually help?

For **I/O-bound** work (network, disk, DB) — the thread is released while waiting, improving throughput/scalability (huge for web servers handling many requests). For **CPU-bound** work, async alone doesn't help; you'd offload to a background thread via `Task.Run`.

**Java parallel:** Same principle — async/non-blocking I/O scales better; CPU work needs actual parallelism.

### Q: Why is `async void` bad?

`async void` methods can't be awaited, and exceptions thrown inside them can't be caught by the caller — they crash the process. Use `async Task` instead. The only legitimate use is event handlers.

```csharp
async void Bad() { await Task.Delay(100); throw new Exception(); } // unobservable crash
async Task Good() { await Task.Delay(100); }                        // awaitable
```

**Java parallel:** No direct analog; think "fire-and-forget that swallows errors" — avoid it.

### Q: What causes a deadlock with `.Result` or `.Wait()`?

Blocking on an async call (`task.Result`, `task.Wait()`) from a thread with a synchronization context (classic ASP.NET, UI apps) can deadlock: the awaited continuation needs the captured context, but that thread is blocked waiting for the task. **Async all the way** — await instead of blocking.

```csharp
// DANGER in contexts with a sync context:
var data = GetDataAsync().Result;  // can deadlock
// SAFE:
var data = await GetDataAsync();
```

**Java parallel:** Conceptually like blocking on a future from the same single thread that must complete it.

### Q: What is `ConfigureAwait(false)`?

It tells `await` not to capture/resume on the original synchronization context, resuming on any thread pool thread. Used in **library code** to avoid context overhead and reduce deadlock risk. In ASP.NET Core (no sync context) it's usually unnecessary; in libraries it's still good practice.

```csharp
await SomeIoAsync().ConfigureAwait(false);
```

**Java parallel:** No direct equivalent — it's specific to .NET's synchronization context model.

### Q: What is `Task.WhenAll` vs `Task.WhenAny`?

`Task.WhenAll` awaits multiple tasks concurrently and completes when **all** finish (great for parallel I/O). `Task.WhenAny` completes when the **first** one finishes.

```csharp
var results = await Task.WhenAll(FetchAsync(1), FetchAsync(2)); // run in parallel
```

**Java parallel:** `WhenAll` ≈ `CompletableFuture.allOf`; `WhenAny` ≈ `anyOf`.

### Q: What is a `CancellationToken`?

A token passed into async methods to request cooperative cancellation. The method checks `token.ThrowIfCancellationRequested()` or passes it down to APIs that honor it.

```csharp
public async Task RunAsync(CancellationToken ct)
{
    await Task.Delay(1000, ct); // cancels cleanly if requested
}
```

**Java parallel:** Like checking `Thread.interrupted()` or a `Future.cancel`, but standardized and threaded through APIs.

### Q: Is async multithreading?

No. Async is about **not blocking** while waiting (concurrency), which may or may not involve multiple threads. For I/O there's often no thread doing the waiting at all. Parallelism (multiple threads doing CPU work at once) is a separate concept (`Parallel.For`, `Task.Run`).

**Java parallel:** Same distinction as concurrency vs. parallelism in Java.

---

## Exception Handling

### Q: Does C# have checked exceptions?

No. All exceptions in C# are **unchecked** — you're never forced by the compiler to catch or declare them. There's no `throws` clause. Document and handle exceptions by convention.

**Java parallel:** Big difference from Java's checked exceptions. There is no `throws` in method signatures.

### Q: What is the difference between `throw` and `throw ex`?

`throw;` (rethrow) **preserves** the original stack trace. `throw ex;` **resets** the stack trace to the current line, losing the original error location. Always use `throw;` to rethrow.

```csharp
try { Risky(); }
catch (Exception ex)
{
    Log(ex);
    throw;        // preserves original stack trace
    // throw ex;  // BAD: wipes the stack trace
}
```

**Java parallel:** Java's `throw e;` always preserves the trace — C#'s `throw ex;` pitfall has no Java equivalent, so it surprises Java devs.

### Q: What is the `finally` block?

`finally` always runs whether or not an exception occurred — used for cleanup (closing resources, releasing locks). It runs even if the try/catch returns.

**Java parallel:** Identical to Java `finally`.

### Q: What is `using` and `IDisposable`?

`IDisposable` defines a `Dispose()` method for releasing unmanaged resources (files, sockets, DB connections). A `using` statement/declaration guarantees `Dispose()` is called when the scope ends, even on exception.

```csharp
using var conn = new SqlConnection(cs); // disposed at end of scope
// or block form:
using (var f = File.OpenRead(path)) { /* ... */ } // disposed here
```

**Java parallel:** `IDisposable`/`using` ≈ `AutoCloseable`/try-with-resources.

### Q: How do you create a custom exception?

Derive from `Exception` (or a more specific base) and provide the standard constructors.

```csharp
public class OrderNotFoundException : Exception
{
    public OrderNotFoundException(string message) : base(message) { }
}
```

**Java parallel:** Same as extending `RuntimeException` (no checked-exception constructors needed).

### Q: What are exception filters (`when`)?

A `catch ... when (condition)` clause catches only if the condition is true, without unwinding the stack for non-matching cases.

```csharp
try { Call(); }
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{ /* handle 404 only */ }
```

**Java parallel:** No direct equivalent; in Java you'd catch then re-check and rethrow.

### Q: What is the difference between `Exception` and `ApplicationException`?

`Exception` is the base of all exceptions. `ApplicationException` was historically meant for app-specific errors but is no longer recommended as a base — derive custom exceptions directly from `Exception`.

**Java parallel:** Roughly like the old advice around extending `Exception` vs. `RuntimeException`.

---

## Memory & Garbage Collection

### Q: How does memory management work in C#?

The **CLR** uses an automatic, generational, mark-and-sweep **garbage collector**. You allocate objects with `new`; the GC reclaims unreachable objects. You generally don't free memory manually, but you must dispose **unmanaged** resources.

**Java parallel:** Same model as the JVM — automatic GC, no manual `free`.

### Q: What are GC generations (Gen 0, 1, 2) and the LOH?

The GC groups objects by age for efficiency:
- **Gen 0** — short-lived new objects; collected most frequently and cheaply.
- **Gen 1** — survivors of Gen 0; a buffer between short- and long-lived.
- **Gen 2** — long-lived objects; collected least often (expensive full GC).
- **LOH (Large Object Heap)** — objects ≥ ~85 KB; collected with Gen 2 and not compacted by default (can fragment).

The "generational hypothesis": most objects die young, so collecting Gen 0 often is cheap and effective.

**Java parallel:** Directly analogous to JVM young (Eden/Survivor) and old generations; LOH is like humongous-object handling in G1.

### Q: What is the difference between `IDisposable` and a finalizer?

`Dispose()` (via `IDisposable`) is **deterministic** cleanup you trigger (usually with `using`). A **finalizer** (`~ClassName`) runs non-deterministically when the GC collects the object, as a safety net for unmanaged resources. Finalizers delay collection and hurt performance — prefer `IDisposable`; implement a finalizer only when wrapping raw unmanaged handles.

```csharp
class Resource : IDisposable
{
    public void Dispose() { /* release now */ GC.SuppressFinalize(this); }
    ~Resource() { /* fallback cleanup */ }
}
```

**Java parallel:** `Dispose` ≈ `AutoCloseable.close()`; finalizer ≈ the deprecated `finalize()` (avoid it for the same reasons).

### Q: What causes memory leaks in a GC language?

Objects that stay **reachable** when you no longer need them: unremoved event handler subscriptions, static collections that keep growing, long-lived caches, captured variables in long-lived delegates/closures, and undisposed resources. The GC can't collect anything still referenced.

**Java parallel:** Same root cause — lingering references (static maps, unremoved listeners).

### Q: What is the performance cost of boxing?

Each boxing operation **allocates** a heap object and copies the value; unboxing copies back and requires a cast. In hot loops or large collections this creates GC pressure. Use generics (`List<int>`) instead of non-generic collections (`ArrayList`) to avoid it.

**Java parallel:** Same as autoboxing overhead with `Integer` in tight loops.

### Q: What is the difference between managed and unmanaged resources?

**Managed** resources are handled by the CLR/GC (normal .NET objects). **Unmanaged** resources (file handles, sockets, native memory) aren't tracked by the GC and must be released via `Dispose`/finalizers.

**Java parallel:** Similar to JNI/native handles vs. ordinary heap objects.

### Q: Can you force garbage collection?

`GC.Collect()` exists but you almost never should call it — it disrupts the GC's heuristics and usually hurts performance. Let the GC do its job.

**Java parallel:** Like `System.gc()` — generally avoid.

---

## ASP.NET Core

### Q: What is the middleware pipeline?

Incoming HTTP requests flow through an ordered chain of **middleware** components, each able to process the request, short-circuit, or pass it to the next via `next()`. Order matters (e.g., authentication before authorization, exception handling first).

```csharp
app.UseExceptionHandler("/error");
app.UseHttpsRedirection();
app.UseAuthentication();   // must come before authorization
app.UseAuthorization();
app.MapControllers();
```

**Java parallel:** Like the Servlet `Filter` chain or Spring's filter/interceptor chain.

### Q: What are the DI service lifetimes?

- **Singleton** — one instance for the whole app lifetime.
- **Scoped** — one instance per request (per scope).
- **Transient** — a new instance every time it's requested.

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddScoped<IOrderService, OrderService>();   // per request
builder.Services.AddTransient<IEmailSender, EmailSender>();  // per resolve
```

**Java parallel:** Singleton ≈ Spring default singleton scope; Scoped ≈ request scope; Transient ≈ prototype scope.

### Q: What is `Program.cs` in modern ASP.NET Core?

Since .NET 6 it uses the **minimal hosting model**: a single top-level file builds the app, registers services, configures the middleware pipeline, and runs it. There's no separate `Startup.cs` by default.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();              // register services
var app = builder.Build();
app.UseAuthorization();                          // configure pipeline
app.MapControllers();
app.Run();
```

**Java parallel:** Like a Spring Boot `main` + `@Configuration` rolled into one entry point.

### Q: What is model binding?

The framework automatically maps incoming request data (route values, query string, form, JSON body, headers) onto action method parameters/objects. Combined with model validation via data annotations.

```csharp
[HttpPost]
public IActionResult Create([FromBody] CreateOrderDto dto) { /* dto auto-bound */ }
```

**Java parallel:** Like Spring's `@RequestBody`/`@RequestParam`/`@PathVariable` binding.

### Q: What is `IActionResult` and why use it?

`IActionResult` lets an action return different HTTP responses (200, 404, 400, etc.) polymorphically via helpers like `Ok()`, `NotFound()`, `BadRequest()`, `CreatedAtAction()`. `ActionResult<T>` combines a typed result with status flexibility.

```csharp
public ActionResult<Order> Get(int id)
{
    var order = _repo.Find(id);
    return order is null ? NotFound() : Ok(order);
}
```

**Java parallel:** Like Spring's `ResponseEntity<T>`.

### Q: What is the difference between minimal APIs and controller-based APIs?

**Minimal APIs** map endpoints directly with lambdas in `Program.cs` — lightweight, great for small services/microservices. **Controllers** group actions in classes with attributes, filters, and model binding — better for larger apps needing structure and conventions.

```csharp
// Minimal API
app.MapGet("/ping", () => "pong");
```

**Java parallel:** Controllers ≈ Spring `@RestController`; minimal APIs are closer to a lightweight router (no direct Spring MVC analog).

### Q: What is the difference between middleware and filters?

**Middleware** runs in the broad HTTP pipeline for every request, before MVC even resolves an action. **Filters** (authorization, action, exception, result filters) run within the MVC pipeline around action execution and have access to MVC context (model state, action arguments). Use middleware for cross-cutting concerns app-wide; filters for MVC-specific concerns.

**Java parallel:** Middleware ≈ Servlet filters; MVC filters ≈ Spring `HandlerInterceptor`/controller advice.

### Q: How does configuration work (`appsettings.json`)?

Configuration is layered from multiple providers (JSON files, environment variables, command line, user secrets) into `IConfiguration`. `appsettings.{Environment}.json` overrides the base. Bind sections to typed options with the options pattern.

```csharp
var conn = builder.Configuration.GetConnectionString("Default");
builder.Services.Configure<MySettings>(builder.Configuration.GetSection("MySettings"));
```

**Java parallel:** Like `application.properties`/`application-{profile}.yml` and `@ConfigurationProperties`.

### Q: What is the `[ApiController]` attribute?

It opts a controller into API conventions: automatic model validation (returns 400 on invalid model state), binding source inference, and `ProblemDetails` error responses.

**Java parallel:** Similar conveniences to Spring's validation + `@RestControllerAdvice` defaults.

### Q: What is Kestrel?

The cross-platform, high-performance web server built into ASP.NET Core that hosts your app, often behind a reverse proxy (IIS, Nginx) in production.

**Java parallel:** Like the embedded Tomcat/Netty server in Spring Boot.

---

## Entity Framework Core

### Q: What is Entity Framework Core?

EF Core is the modern **ORM** for .NET — it maps C# classes (entities) to database tables and lets you query with LINQ, translating to SQL. It supports migrations, change tracking, and multiple providers (SQL Server, PostgreSQL, SQLite).

**Java parallel:** EF Core ≈ Hibernate/JPA. LINQ-to-Entities ≈ JPQL/Criteria.

### Q: What is a `DbContext`?

The `DbContext` represents a session with the database — it exposes `DbSet<T>` properties (tables), tracks changes, and persists them on `SaveChanges()`. It's a Unit of Work + repository.

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
}
```

**Java parallel:** `DbContext` ≈ JPA `EntityManager`/Hibernate `Session`. `DbSet<T>` ≈ a typed repository/`EntityManager` for an entity.

### Q: Is `DbContext` thread-safe?

No. A `DbContext` is **not thread-safe** and must not be shared across concurrent operations/threads. In ASP.NET Core it's registered as **Scoped** (one per request). For parallel work, create separate contexts or use a `DbContextFactory`.

**Java parallel:** Like the JPA `EntityManager` — not thread-safe, one per unit of work.

### Q: What is change tracking?

By default the context tracks loaded entities and detects modifications, so `SaveChanges()` generates the right INSERT/UPDATE/DELETE. Tracking has overhead — disable it for read-only queries.

**Java parallel:** Like Hibernate's persistence context dirty checking.

### Q: What is `AsNoTracking` and when do you use it?

`AsNoTracking()` returns entities without tracking them, improving performance for read-only queries (no snapshots, less memory). Use it whenever you won't update the results.

```csharp
var report = db.Orders.AsNoTracking().Where(o => o.Total > 100).ToList();
```

**Java parallel:** Like a read-only/detached query in Hibernate.

### Q: What is the N+1 query problem and how do you fix it?

It happens when you load a list (1 query) then lazily load a related entity for each item (N queries). Fix it with eager loading via `Include` to fetch related data in one (or few) queries.

```csharp
// N+1: each order.Customer triggers a query
var orders = db.Orders.ToList();
// Fixed: one query with a JOIN
var orders2 = db.Orders.Include(o => o.Customer).ToList();
```

**Java parallel:** Same problem in JPA/Hibernate; `Include` ≈ a JPQL `JOIN FETCH` or entity graph.

### Q: What is the difference between eager, lazy, and explicit loading?

- **Eager** — load related data up front with `Include`.
- **Lazy** — related data loads automatically on first access (requires proxies; risks N+1).
- **Explicit** — you manually trigger loading later via `context.Entry(e).Reference(...).Load()`.

**Java parallel:** Same trio as JPA `FetchType.EAGER`/`LAZY` and explicit `Hibernate.initialize`.

### Q: Why does `IQueryable` vs `IEnumerable` matter in EF Core?

While a query is `IQueryable`, filters/projections are translated to SQL and run in the **database**. Once it becomes `IEnumerable` (e.g., `ToList()`, `AsEnumerable()`), further operations run in **memory** on already-fetched data. Filtering after materializing pulls too much data.

**Java parallel:** Conceptually like building a Criteria query (DB-side) vs. filtering a fetched `List` (app-side).

### Q: What are migrations?

Migrations version your database schema as code. You add a migration from model changes and apply it to update the database.

```bash
dotnet ef migrations add AddOrders   # generate migration from model changes
dotnet ef database update            # apply to the database
```

**Java parallel:** Like Flyway/Liquibase, but generated from your entity model (Hibernate's `hbm2ddl` is closer but less controlled).

### Q: What is the difference between `SaveChanges` and `SaveChangesAsync`?

Both persist tracked changes in a transaction; the async version doesn't block the thread during the DB round-trip (better for scalability in web apps). Prefer the async version in request handlers.

**Java parallel:** Like committing a JPA transaction, with an async I/O variant.

---

## Dependency Injection

### Q: What is dependency injection?

A design pattern where a class receives its dependencies from the outside (usually via the constructor) instead of creating them. ASP.NET Core has a **built-in IoC container** that resolves and injects registered services.

```csharp
public class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo; // injected
}
```

**Java parallel:** Same concept as Spring DI; built-in container ≈ Spring `ApplicationContext`.

### Q: Why prefer constructor injection?

It makes dependencies explicit and required, enables `readonly` fields (immutability), and makes the class easy to unit test (pass mocks). Property/method injection are situational.

**Java parallel:** Same recommendation as Spring (constructor injection preferred over field injection).

### Q: What is a captive dependency?

When a **longer-lived** service holds a reference to a **shorter-lived** one — e.g., a Singleton injecting a Scoped service. The Scoped instance gets "captured" and lives as long as the Singleton, breaking its lifetime semantics (and often thread-safety, e.g., capturing a `DbContext`). The container can throw at startup for this in dev.

**Java parallel:** Like injecting a request-scoped bean into a singleton without a proxy in Spring — same bug class.

### Q: How do you inject a Scoped service into a Singleton safely?

Inject `IServiceProvider` (or `IServiceScopeFactory`) and create a scope when you need the scoped service, disposing it after use.

```csharp
using var scope = _scopeFactory.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
```

**Java parallel:** Like using a `ObjectProvider`/scoped proxy in Spring.

### Q: What's the difference between `AddTransient`, `AddScoped`, `AddSingleton`?

They register a service's lifetime: Transient = new every resolve, Scoped = one per request scope, Singleton = one for the app. Choosing wrong (e.g., Singleton for a `DbContext`) causes bugs.

**Java parallel:** prototype / request / singleton scopes in Spring.

---

## Security

### Q: What is the difference between authentication and authorization?

**Authentication** = verifying *who* you are (login, validating a token). **Authorization** = determining *what* you're allowed to do (roles, policies, claims). AuthN comes first, then AuthZ.

**Java parallel:** Same distinction as Spring Security's authentication vs. access control.

### Q: What is a JWT?

A **JSON Web Token** is a signed, self-contained token with three parts (header.payload.signature). The payload holds **claims** (user id, roles, expiry). The server validates the signature to trust it without server-side session state — good for stateless APIs.

**Java parallel:** Identical concept; same JWT standard used with Spring Security.

### Q: What are claims?

Key/value statements about the user (name, email, role, permissions) carried in the authenticated identity (`ClaimsPrincipal`). Authorization policies inspect claims.

```csharp
var name = User.FindFirst(ClaimTypes.Name)?.Value;
```

**Java parallel:** Like authorities/granted roles and JWT claims in Spring Security.

### Q: What does the `[Authorize]` attribute do?

It restricts access to a controller/action to authenticated users, optionally by role or policy. `[AllowAnonymous]` opts back out.

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteUser(int id) { /* admins only */ }
```

**Java parallel:** Like `@PreAuthorize("hasRole('ADMIN')")` / method security in Spring.

### Q: How should you store passwords?

Never store plaintext or simple hashes. Use a **slow, salted** password hash (PBKDF2 via ASP.NET Core Identity's `PasswordHasher`, bcrypt, or Argon2). Salting defeats rainbow tables; slowness defeats brute force.

**Java parallel:** Same as using Spring Security's `BCryptPasswordEncoder`.

### Q: What is CORS?

Cross-Origin Resource Sharing controls which other origins (domains) a browser may call your API from. Configure a CORS policy in ASP.NET Core to allow specific origins/methods/headers.

**Java parallel:** Same browser mechanism; configured similarly in Spring (`@CrossOrigin`/CORS config).

### Q: How does ASP.NET Core protect against common web attacks?

Parameterized queries via EF Core prevent SQL injection; Razor auto-encodes output to mitigate XSS; antiforgery tokens guard against CSRF for cookie-based auth; HTTPS redirection and HSTS enforce transport security.

**Java parallel:** Analogous protections to Spring Security + JPA parameter binding.

---

## Testing

### Q: What testing frameworks are common in .NET?

**xUnit** (popular, modern), **NUnit**, and **MSTest**. **Moq** (or NSubstitute) for mocking, **FluentAssertions** for readable asserts, and `WebApplicationFactory` for integration tests.

**Java parallel:** xUnit/NUnit ≈ JUnit; Moq ≈ Mockito; FluentAssertions ≈ AssertJ.

### Q: What is the difference between `[Fact]` and `[Theory]` in xUnit?

`[Fact]` is a single test with no parameters. `[Theory]` is a parameterized/data-driven test that runs once per data set supplied by `[InlineData]`, `[MemberData]`, or `[ClassData]`.

```csharp
[Theory]
[InlineData(2, 3, 5)]
[InlineData(-1, 1, 0)]
public void Adds(int a, int b, int expected)
    => Assert.Equal(expected, a + b);
```

**Java parallel:** `[Fact]` ≈ `@Test`; `[Theory]`+`[InlineData]` ≈ JUnit 5 `@ParameterizedTest`+`@CsvSource`.

### Q: What is the AAA pattern?

**Arrange** (set up data/mocks), **Act** (call the method under test), **Assert** (verify the outcome). Keeps tests readable and focused.

```csharp
// Arrange
var calc = new Calculator();
// Act
var result = calc.Add(2, 3);
// Assert
Assert.Equal(5, result);
```

**Java parallel:** Same Given/When/Then or Arrange-Act-Assert convention.

### Q: How do you mock a dependency with Moq?

Create a mock of an interface, set up its behavior, inject it, and optionally verify interactions.

```csharp
var repo = new Mock<IOrderRepository>();
repo.Setup(r => r.Find(1)).Returns(new Order { Id = 1 }); // stub return
var sut = new OrderService(repo.Object);

var result = sut.Get(1);

repo.Verify(r => r.Find(1), Times.Once); // verify it was called
```

**Java parallel:** `Mock<T>`/`Setup`/`Verify` ≈ Mockito `mock()`/`when().thenReturn()`/`verify()`.

### Q: What is the difference between a unit test and an integration test?

A **unit test** isolates one class with mocked dependencies (fast, no I/O). An **integration test** exercises multiple components together (real DB, HTTP pipeline) to verify they work as a whole.

**Java parallel:** Same distinction; integration tests ≈ Spring Boot `@SpringBootTest`.

### Q: How do you write an integration test for an API?

Use `WebApplicationFactory<TProgram>` to spin up the app in-memory and send real HTTP requests with an `HttpClient`.

```csharp
public class OrdersTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public OrdersTests(WebApplicationFactory<Program> f) => _client = f.CreateClient();

    [Fact]
    public async Task Get_ReturnsOk()
    {
        var res = await _client.GetAsync("/api/orders");
        res.EnsureSuccessStatusCode();
    }
}
```

**Java parallel:** Like Spring Boot's `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate`/`MockMvc`.

### Q: What makes a good unit test?

Fast, isolated, deterministic, and testing one behavior with a clear name. Avoid hitting external systems and shared state; one logical assertion of behavior per test.

**Java parallel:** Same FIRST principles (Fast, Independent, Repeatable, Self-validating, Timely).

---

## .NET Ecosystem & Tooling

### Q: What is the difference between .NET Framework, .NET Core, and modern .NET?

- **.NET Framework** (1.0–4.8): legacy, Windows-only, maintenance mode.
- **.NET Core** (1–3.1): cross-platform rewrite.
- **.NET 5+** (now just **.NET**, currently .NET 8/9): the unified, cross-platform, open-source successor. Use modern .NET for new work.

**Java parallel:** Roughly like moving from an old vendor JDK to OpenJDK with regular LTS releases; modern .NET ≈ current LTS Java.

### Q: What is the CLR?

The **Common Language Runtime** — .NET's virtual machine that executes managed code, providing JIT compilation, garbage collection, type safety, and exception handling.

**Java parallel:** CLR ≈ the JVM.

### Q: What is IL and JIT?

C# compiles to **IL (Intermediate Language)** stored in assemblies. At runtime the **JIT** compiler translates IL to native machine code on demand. (AOT compilation can produce native code ahead of time.)

**Java parallel:** IL ≈ Java bytecode; JIT ≈ the JVM's JIT (HotSpot).

### Q: What is an assembly?

A compiled .NET deployment unit — a `.dll` or `.exe` containing IL, metadata, and a manifest. It's the unit of versioning and the boundary for `internal` access.

**Java parallel:** Roughly a `.jar`, though assemblies also carry rich metadata and define the `internal` visibility boundary.

### Q: What is NuGet?

The package manager for .NET — you add third-party libraries as packages referenced in the `.csproj`, restored from nuget.org.

```bash
dotnet add package Newtonsoft.Json
```

**Java parallel:** NuGet ≈ Maven/Gradle dependencies (nuget.org ≈ Maven Central).

### Q: What is the `dotnet` CLI?

The cross-platform command-line tool for building, running, testing, and managing .NET projects.

```bash
dotnet new webapi      # scaffold a project
dotnet build           # compile
dotnet run             # run
dotnet test            # run tests
```

**Java parallel:** Like the `mvn`/`gradle` and `java` commands combined.

### Q: What is the difference between a `.csproj` and a solution (`.sln`)?

A `.csproj` defines one project (its target framework, dependencies, build settings). A `.sln` groups multiple projects together for building/IDE organization.

**Java parallel:** `.csproj` ≈ a Maven `pom.xml`/Gradle module; `.sln` ≈ a multi-module Maven parent/Gradle multi-project build.

### Q: What is the Common Type System (CTS) and CLS?

The **CTS** defines how types are declared and used across all .NET languages; the **CLS** is a subset of rules for cross-language interoperability. They're why C#, F#, and VB.NET can share libraries.

**Java parallel:** Conceptually like the JVM's common type model that lets Java, Kotlin, Scala interoperate.

### Q: What is `Span<T>`?

A stack-only, allocation-free view over contiguous memory (arrays, stack memory, strings) used for high-performance, low-allocation slicing. Junior-level: know it exists for perf-critical code.

**Java parallel:** Loosely like `ByteBuffer`/array slicing but with no allocation and compiler-enforced safety.

---

## Behavioral / Practical

### Q: How would you structure a Web API project?

Typically by responsibility: **Controllers/Endpoints** (HTTP), **Services** (business logic), **Repositories/EF DbContext** (data access), **DTOs** (request/response models), **Domain/Entities**, and DI wiring in `Program.cs`. Keep controllers thin and push logic into services. Many teams use a layered or Clean Architecture approach.

**Java parallel:** Mirrors the classic Spring layering: Controller → Service → Repository.

### Q: How do you debug an issue in a .NET app?

Reproduce it, read the exception/stack trace, set breakpoints and inspect locals in the debugger, add structured logging (`ILogger`), check configuration/environment, and write a failing test that reproduces the bug, then fix until it passes. Use the debugger's call stack and watch windows.

**Java parallel:** Same workflow as debugging in IntelliJ with the JVM debugger + SLF4J logging.

### Q: How do the SOLID principles apply in C#?

- **S**ingle Responsibility — a class has one reason to change.
- **O**pen/Closed — extend via interfaces/inheritance, don't modify.
- **L**iskov Substitution — subtypes must be usable as their base.
- **I**nterface Segregation — prefer small, focused interfaces.
- **D**ependency Inversion — depend on abstractions; enabled by built-in DI.

C# features like interfaces, `virtual`/`abstract`, and the DI container make these natural to apply.

**Java parallel:** Identical principles; the DI container plays Spring's role for Dependency Inversion.

### Q: How do you log in ASP.NET Core?

Inject `ILogger<T>` and use structured logging with message templates. Configure providers (console, file, Seq, etc.) and log levels via configuration.

```csharp
public class OrderService(ILogger<OrderService> logger)
{
    public void Process(int id) => logger.LogInformation("Processing order {OrderId}", id);
}
```

**Java parallel:** `ILogger<T>` ≈ SLF4J `Logger`; structured templates ≈ parameterized SLF4J logging.

### Q: How do you handle errors globally in an API?

Use exception-handling middleware (`UseExceptionHandler`) or an `IExceptionFilter`/`ProblemDetails` to return consistent error responses, rather than try/catch in every action.

**Java parallel:** Like a `@RestControllerAdvice` / `@ExceptionHandler` in Spring.

### Q: What's a DTO and why use one?

A **Data Transfer Object** is a flat shape for moving data across boundaries (e.g., API request/response), decoupling your API contract from internal entities. It prevents over-posting, hides internal fields, and keeps the API stable as the domain evolves.

**Java parallel:** Same DTO pattern used in Spring; map with AutoMapper or by hand (≈ MapStruct).

---

## Quick Reference Cheat Sheet

```
VALUE vs REFERENCE  : value copied by value (struct,int,enum); reference copies pointer (class,string,array)
BOXING              : value -> object on heap (allocates); avoid in hot loops; use generics
STRING              : immutable; use StringBuilder for loops; == compares content
VAR                 : compile-time inference, still strongly typed (NOT dynamic)
CONST vs READONLY   : const = compile-time, inline only; readonly = runtime, set in ctor
== vs Equals        : == reference by default (string overloads to content); Equals = value
NULLABLE            : int? = Nullable<int>; string? = nullable reference type (compile-time)
VIRTUAL/OVERRIDE    : opt-in polymorphism; methods NOT virtual by default (unlike Java)
NEW (modifier)      : hides base member; resolved by compile-time type
ABSTRACT vs IFACE   : abstract=state+single inherit; interface=contract+multi (default methods C#8+)
SEALED              : no inheritance (≈ Java final class)
STRUCT vs CLASS     : struct=value type; class=reference type
RECORD              : immutable, value equality, with-copy; great for DTOs
ACCESS              : public/private/protected/internal(assembly)/protected internal/private protected
GENERICS            : reified (typeof(T), new T()) — NO erasure (vs Java)
VARIANCE            : out=covariant, in=contravariant (declared on type)
COLLECTIONS         : List≈ArrayList, Dictionary≈HashMap, HashSet≈HashSet; Concurrent* for threads
IENUM/ICOLL/ILIST   : iterate / +count,add / +index; expose least capable
LINQ                : deferred until enumerated; Where=filter Select=map SelectMany=flatMap
IQUERYABLE vs IENUM : IQueryable -> SQL in DB; IEnumerable -> in memory
FIRST vs SINGLE     : First throws if none; Single throws if !=1; *OrDefault returns default
ASYNC/AWAIT         : non-blocking I/O scalability; returns Task; "async all the way"
ASYNC VOID          : avoid (unobservable exceptions) except event handlers
.Result/.Wait()     : can DEADLOCK with sync context; await instead
WhenAll/WhenAny     : all complete / first completes (≈ allOf/anyOf)
EXCEPTIONS          : all unchecked, no throws clause; use "throw;" not "throw ex;"
USING/IDisposable   : deterministic cleanup (≈ try-with-resources/AutoCloseable)
GC GENERATIONS      : Gen0 young/cheap, Gen1 buffer, Gen2 old/expensive, LOH >=85KB
DISPOSE vs FINALIZER: Dispose deterministic (use this); finalizer non-deterministic fallback
MIDDLEWARE          : ordered HTTP pipeline (≈ servlet filters); order matters
DI LIFETIMES        : Singleton=app, Scoped=per request, Transient=per resolve
CAPTIVE DEPENDENCY  : long-lived holding short-lived (Singleton -> Scoped) = bug
PROGRAM.CS          : minimal hosting; build services -> configure pipeline -> run
IACTIONRESULT       : Ok/NotFound/BadRequest/CreatedAtAction (≈ ResponseEntity)
EF DBCONTEXT        : Unit of Work; Scoped; NOT thread-safe (≈ EntityManager)
ASNOTRACKING        : read-only perf; skip change tracking
N+1                 : fix with Include (eager load / JOIN FETCH)
MIGRATIONS          : dotnet ef migrations add / database update (≈ Flyway)
AUTHN vs AUTHZ      : who you are vs what you can do; [Authorize(Roles=...)]
JWT                 : signed header.payload.signature; claims; stateless
PASSWORDS           : salted + slow hash (PBKDF2/bcrypt/Argon2) — never plaintext
XUNIT               : [Fact]=test, [Theory]+[InlineData]=parameterized; AAA pattern
MOQ                 : new Mock<T>(); Setup().Returns(); Verify() (≈ Mockito)
INTEGRATION TEST    : WebApplicationFactory<Program> + HttpClient
.NET vs FRAMEWORK   : modern .NET 8+ cross-platform; Framework = legacy Windows
CLR/IL/JIT          : CLR≈JVM; IL≈bytecode; JIT compiles IL->native
ASSEMBLY/NUGET      : .dll unit (≈ jar) ; NuGet ≈ Maven; dotnet CLI ≈ mvn+java
SOLID               : SRP, OCP, LSP, ISP, DIP — DI container enables DIP
```

*Last Updated: 2026-06-16*
