# Modern C# Language Features, C# 8–12 (for Java Developers)

## Overview

Your other guides cover the *big* concepts (OOP, generics, async, LINQ). This guide fills the **"small syntax" gap** — the modern C# features (versions 8 through 12) that a Java dev has never seen and that show up in *every* real .NET 8 codebase and code review. None of these are hard individually, but stringing them together is what makes code look like **idiomatic modern C#** instead of "Java with different keywords."

Interviewers use these to gauge whether you've written *recent* C# or only old tutorials. Skim the [Mapping](#1-java--c-quick-mapping), then work through each feature. All code targets **C# 12 / .NET 8**. Drill the [Interview Questions](#13-common-interview-questions) and [Cheat Sheet](#14-quick-reference-cheat-sheet) before applying.

---

## Table of Contents

1. [Java → C# Quick Mapping](#1-java--c-quick-mapping)
2. [String Interpolation & Raw String Literals](#2-string-interpolation--raw-string-literals)
3. [Tuples & Deconstruction](#3-tuples--deconstruction)
4. [Switch Expressions & Pattern Matching](#4-switch-expressions--pattern-matching)
5. [Nullable Reference Types (NRTs)](#5-nullable-reference-types-nrts)
6. [Null Operators: `?.` `??` `??=` `!`](#6-null-operators------)
7. [`init`, `required`, and Object Initializers](#7-init-required-and-object-initializers)
8. [Primary Constructors](#8-primary-constructors)
9. [Target-Typed `new` & Collection Expressions](#9-target-typed-new--collection-expressions)
10. [Ranges & Indices (`^` and `..`)](#10-ranges--indices--and-)
11. [`using` Declarations & Top-Level Statements](#11-using-declarations--top-level-statements)
12. [Local Functions, `yield`, and Small Wins](#12-local-functions-yield-and-small-wins)
13. [Common Interview Questions](#13-common-interview-questions)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)

---

## 1. Java → C# Quick Mapping

**Think of it like...** upgrading from Java 8 to Java 21 all at once — records, switch expressions, pattern matching, text blocks. C# added the *same modern conveniences*, often earlier, plus a few Java still lacks (properties, `init`, ranges).

| Java | C# (8–12) | Notes |
|------|-----------|-------|
| `String.format("Hi %s", n)` | `$"Hi {n}"` (interpolation) | Inline, compile-checked |
| Text blocks `"""..."""` (Java 15+) | Raw string literals `"""..."""` | Same idea, richer |
| `record Point(int x, int y)` | `record Point(int X, int Y)` | Both have positional records |
| Switch expression (Java 14+) | Switch expression + patterns | C# patterns go further |
| `instanceof Foo f` (Java 16+) | `is Foo f` (type pattern) | C# had it first |
| `Optional<T>` | Nullable reference types `T?` | Compile-time null tracking |
| `Objects.requireNonNullElse(a, b)` | `a ?? b` | Null-coalescing |
| `map.getOrDefault(k, d)` | `dict.GetValueOrDefault(k, d)` | — |
| No value tuples | `(int, string)` value tuples | Lightweight multi-return |
| Destructuring (none in Java) | Deconstruction `var (a, b) = p;` | — |
| `list.get(list.size()-1)` | `list[^1]` (index from end) | — |
| `list.subList(1, 4)` | `list[1..4]` (range) | — |
| `final` local | (locals are mutable; use `readonly`/`const` on fields) | — |
| try-with-resources | `using` declaration | — |
| No first-class properties | `{ get; init; }` properties | C# exclusive |

---

## 2. String Interpolation & Raw String Literals

```csharp
string name = "Amith";
int score = 92;

// STRING INTERPOLATION — prefix with $, embed expressions in { }:
string msg = $"{name} scored {score} ({score / 100.0:P0})";  // "Amith scored 92 (92 %)"
//                                          ^^^^ format specifier after ':'  (P0 = percent, 0 decimals)

// Common format specifiers inside {expr:FORMAT}:
Console.WriteLine($"{score:D5}");        // 00092   (integer, 5 digits, zero-padded)
Console.WriteLine($"{3.14159:F2}");      // 3.14    (fixed, 2 decimals)
Console.WriteLine($"{1234.5:C}");        // $1,234.50 (currency, culture-aware)
Console.WriteLine($"{DateTime.Now:yyyy-MM-dd}");  // 2026-07-09

// RAW STRING LITERALS (C# 11) — triple quotes; no escaping needed. Great for JSON/SQL/paths:
string json = """
    {
        "name": "Amith",
        "active": true
    }
    """;   // leading whitespace up to the closing """ is stripped automatically

// Interpolation + raw combined (use $$ so single { } are literal, {{ }} inject):
string sql = $$"""
    SELECT * FROM Users WHERE Name = '{{name}}';
    """;
```

> **Java parallel:** `$"..."` replaces `String.format`/`+` concatenation; `"""..."""` is Java's text block. Prefer interpolation everywhere for readability.

---

## 3. Tuples & Deconstruction

**Value tuples** let a method return several values without a wrapper class — perfect for small internal returns.

```csharp
// Return multiple values with NAMES (no class needed):
(string Name, int Age) GetUser() => ("Amith", 30);

var user = GetUser();
Console.WriteLine(user.Name);   // Amith  — access by the tuple element name
Console.WriteLine(user.Age);    // 30

// DECONSTRUCTION — unpack straight into variables:
var (name, age) = GetUser();
Console.WriteLine($"{name} is {age}");   // Amith is 30

// Discard values you don't need with _:
var (_, onlyAge) = GetUser();

// Tuples as lightweight keys / swaps:
(int a, int b) = (1, 2);
(a, b) = (b, a);        // swap without a temp variable -> a=2, b=1

// Deconstruct your OWN types by adding a Deconstruct method:
public record Point(int X, int Y);           // records auto-generate Deconstruct
var p = new Point(3, 4);
var (x, y) = p;                               // works because Point deconstructs
```

> **When NOT to use tuples:** For public APIs or anything long-lived, prefer a `record` or class with real names. Tuples are for *local, short-lived* multi-returns. Interviewers like hearing that boundary.

---

## 4. Switch Expressions & Pattern Matching

The modern `switch` is an **expression that returns a value**, combined with rich **pattern matching**. This replaces long `if/else` ladders.

```csharp
// SWITCH EXPRESSION — note '=>' arms, ',' separators, '_' default, and it RETURNS a value:
string Describe(int n) => n switch
{
    < 0        => "negative",     // relational pattern
    0          => "zero",         // constant pattern
    > 0 and < 10 => "small",      // combined pattern with 'and'
    _          => "large"         // '_' = default (discard)
};

// TYPE PATTERNS — test type AND bind a variable in one step (Java's instanceof Foo f):
decimal Area(object shape) => shape switch
{
    Circle c    => Math.PI * c.Radius * c.Radius,   // 'c' is typed as Circle here
    Rectangle r => r.Width * r.Height,
    null        => 0,
    _           => throw new ArgumentException("Unknown shape")
};

// PROPERTY PATTERNS — match on an object's properties:
string Shipping(Order o) => o switch
{
    { Total: > 100 }              => "free",
    { Country: "US", Total: <= 100 } => "flat $5",
    _                             => "calculated"
};

// 'is' pattern in an if — test + cast + null-check together:
if (shape is Circle circle && circle.Radius > 0)
    Console.WriteLine(circle.Radius);   // 'circle' is in scope and non-null

// List patterns (C# 11) — match array/list shapes:
int[] arr = { 1, 2, 3 };
string s = arr switch
{
    []          => "empty",
    [var only]  => $"one: {only}",
    [var first, .., var last] => $"first {first}, last {last}",  // '..' = the middle
};
```

> **Interview soundbite:** "A switch *statement* performs actions; a switch *expression* produces a value. Pattern matching lets each arm test type, constants, relational conditions, and properties, binding variables inline."

---

## 5. Nullable Reference Types (NRTs)

This is the **single most important modern C# feature to understand**, and Java has no exact equivalent. Enabled by default in .NET 6+ templates (`<Nullable>enable</Nullable>` in the `.csproj`), NRTs make the compiler track whether a reference *can* be null.

```csharp
// With NRTs ENABLED:
string name = "Amith";     // NON-nullable: the compiler assumes it's never null
string? maybe = null;      // NULLABLE: the '?' says "this can be null" — allowed

name = null;               // ⚠️ COMPILER WARNING: assigning null to non-nullable
int len = maybe.Length;    // ⚠️ WARNING: possible null dereference of 'maybe'

// The compiler forces you to handle the null case:
if (maybe != null)
    Console.WriteLine(maybe.Length);   // OK — inside the null check it's treated as non-null

Console.WriteLine(maybe?.Length ?? 0); // OK — null-safe access + default
```

**Key points interviewers probe:**
- NRTs are **compile-time only** — they produce *warnings*, not runtime enforcement. At runtime any reference can still technically be null.
- They're the C# answer to `NullReferenceException`, analogous in *goal* to Java's `Optional<T>` but implemented as **type annotations** rather than a wrapper type — zero runtime cost.
- `string?` and `string` are the *same* type at runtime; the `?` is metadata for the compiler's flow analysis.

> **Golden rule:** Treat NRT warnings as errors. A clean-compiling NRT codebase has drastically fewer null bugs. Don't silence warnings with `!` unless you're certain.

---

## 6. Null Operators: `?.` `??` `??=` `!`

```csharp
string? input = GetInput();

// ?.  NULL-CONDITIONAL: call member only if non-null; whole expression is null otherwise.
int? length = input?.Length;          // null if input is null (no exception)
input?.Trim();                         // no-op if null

// ??  NULL-COALESCING: use the right side when the left is null.
string safe = input ?? "default";      // "default" if input is null

// ??= NULL-COALESCING ASSIGNMENT: assign only if currently null.
input ??= "fallback";                  // input = input ?? "fallback"

// Chaining ?. and ?? — the idiomatic null-safe pattern:
int count = order?.Items?.Count ?? 0;  // 0 if order or Items is null

// !   NULL-FORGIVING: "trust me, this isn't null" — suppresses the warning, NO runtime check.
string definitelyThere = FindUser(id)!.Name;   // use sparingly; you're overriding the compiler
```

| Operator | Meaning | Java analog |
|----------|---------|-------------|
| `a?.B` | member access if `a` non-null | `a == null ? null : a.B` |
| `a ?? b` | `a` if non-null, else `b` | `Objects.requireNonNullElse(a, b)` |
| `a ??= b` | assign `b` only if `a` is null | `if (a == null) a = b;` |
| `a!` | suppress null warning (compile-time only) | (no equivalent) |

---

## 7. `init`, `required`, and Object Initializers

C# properties can be set at construction but frozen afterward — **immutability without a giant constructor**.

```csharp
public class Person
{
    public required string Name { get; init; }   // MUST be set at creation; can't change after
    public int Age { get; init; }                // settable only in an initializer, then read-only
    public string Country { get; set; } = "US";  // fully mutable, with a default
}

// OBJECT INITIALIZER syntax — set properties by name after 'new':
var p = new Person
{
    Name = "Amith",    // 'required' -> compiler ERRORS if you omit this
    Age = 30
};

// p.Age = 31;   // ❌ compile error: 'init' properties are read-only after construction
p.Country = "UK";  // ✅ ok — Country has a normal 'set'
```

- **`init`** — like a `set` that only works during object initialization; gives you immutable-after-construction properties without constructor boilerplate.
- **`required`** — the caller *must* set this property (enforced at compile time), even without a constructor parameter for it.

> **Java parallel:** This achieves what a Java `@Builder` + `final` fields do, but as a built-in language feature — no Lombok, no builder class.

---

## 8. Primary Constructors

C# 12 lets you declare constructor parameters **right on the class/struct header** — they're in scope throughout the type. Great for DI-heavy classes.

```csharp
// BEFORE (classic) — the boilerplate every DI service used to have:
public class OrderService_Old
{
    private readonly IRepository _repo;
    private readonly ILogger _logger;
    public OrderService_Old(IRepository repo, ILogger logger)
    {
        _repo = repo;
        _logger = logger;
    }
    public void Process(int id) => _logger.Log($"Processing {id}");
}

// AFTER (C# 12 primary constructor) — parameters declared on the class line:
public class OrderService(IRepository repo, ILogger logger)
{
    // 'repo' and 'logger' are available in every member — no field declarations needed:
    public void Process(int id) => logger.Log($"Processing {id} via {repo}");
}
```

**Watch out:** Unlike a `record`, a primary constructor on a **class** does *not* auto-create public properties — the parameters are just captured values usable in members. (On a `record`, positional parameters *do* become public properties.)

> **Interview note:** Primary constructors are C# 12 (.NET 8). Older codebases won't have them — recognize both styles.

---

## 9. Target-Typed `new` & Collection Expressions

```csharp
// TARGET-TYPED new (C# 9) — omit the type on the right when it's obvious from the left:
List<string> names = new();                 // instead of new List<string>()
Dictionary<int, string> map = new();        // type inferred from the declaration
private readonly StringBuilder _sb = new(); // common in fields

// COLLECTION EXPRESSIONS (C# 12) — one bracket syntax for arrays, lists, spans:
int[] arr        = [1, 2, 3];               // array
List<int> list   = [1, 2, 3];               // list
int[] combined   = [.. arr, 4, 5, .. list]; // '..' spreads another collection in
Span<int> span   = [1, 2, 3];               // even spans

// Compare to the old, verbose forms:
int[] oldArr     = new int[] { 1, 2, 3 };
List<int> oldList = new List<int> { 1, 2, 3 };
```

> **Why it matters:** Modern C# reviewers expect `new()` and `[...]`. Using the old verbose forms everywhere signals dated code.

---

## 10. Ranges & Indices (`^` and `..`)

C# has built-in syntax for slicing arrays, strings, and lists — something Java makes you do with `substring`/`subList` and index math.

```csharp
int[] nums = { 10, 20, 30, 40, 50 };

// INDEX FROM END with ^  (^1 = last, ^2 = second-to-last):
int last       = nums[^1];    // 50
int secondLast = nums[^2];    // 40

// RANGE with ..  (start inclusive, end EXCLUSIVE — like Python/substring):
int[] first3   = nums[0..3];  // { 10, 20, 30 }
int[] fromTwo  = nums[2..];   // { 30, 40, 50 }   (open end)
int[] lastTwo  = nums[^2..];  // { 40, 50 }
int[] middle   = nums[1..^1]; // { 20, 30, 40 }   (drop first and last)

// Works on strings too:
string s = "Hello, World";
string hello = s[..5];        // "Hello"
string world = s[7..];        // "World"
```

> **Gotcha:** The end of a range is **exclusive**: `nums[0..3]` gives indices 0, 1, 2. `^1` is the last element (not `^0`, which is one *past* the end).

---

## 11. `using` Declarations & Top-Level Statements

```csharp
// USING DECLARATION (C# 8) — auto-disposes at the end of the enclosing scope.
// No extra braces needed (compare to Java's try-with-resources):
void ReadFile(string path)
{
    using var reader = new StreamReader(path);   // disposed when the method returns
    string content = reader.ReadToEnd();
    Console.WriteLine(content);
}   // reader.Dispose() called here automatically

// vs the classic 'using statement' block (still valid, scopes disposal tightly):
using (var conn = new SqlConnection(cs))
{
    conn.Open();
}   // disposed at the closing brace

// TOP-LEVEL STATEMENTS (C# 9) — no Main/class boilerplate for entry points.
// An entire Program.cs can be just:
Console.WriteLine("Hello");           // the compiler wraps this in Main() for you
var name = args.Length > 0 ? args[0] : "world";
Console.WriteLine($"Hi {name}");

// GLOBAL USINGS (C# 10) — declare once, available in every file of the project:
// global using System.Text.Json;    // put in GlobalUsings.cs; no per-file 'using' needed
```

> **Java parallel:** `using var` = try-with-resources without the nesting. Top-level statements are like a scripting mode — you'll see them in every `Program.cs` in .NET 6+.

---

## 12. Local Functions, `yield`, and Small Wins

```csharp
// LOCAL FUNCTIONS — a named method inside another method (cleaner than a lambda for recursion):
int Factorial(int n)
{
    return Compute(n);
    int Compute(int x) => x <= 1 ? 1 : x * Compute(x - 1);   // local, can recurse by name
}

// yield return — lazy iterators without building a whole list (like a generator):
IEnumerable<int> EvensUpTo(int max)
{
    for (int i = 0; i <= max; i += 2)
        yield return i;      // produces one value at a time, on demand
}
foreach (var e in EvensUpTo(6)) Console.Write($"{e} ");   // 0 2 4 6 — nothing materialized upfront

// EXPRESSION-BODIED members — '=>' for one-liners (methods, properties, ctors):
public class Temperature(double celsius)
{
    public double Fahrenheit => celsius * 9 / 5 + 32;       // computed property
    public override string ToString() => $"{celsius}°C";    // one-line method
}

// PATTERN: 'switch' + tuple for clean multi-condition logic:
string Rps(string a, string b) => (a, b) switch
{
    ("rock", "scissors") or ("paper", "rock") or ("scissors", "paper") => "A wins",
    _ when a == b => "tie",
    _ => "B wins"
};
```

> **`yield` is heavily interviewed.** It creates a **lazy** sequence — values are produced only as the consumer iterates, so `EvensUpTo(1_000_000).Take(3)` computes just three numbers. This is how many LINQ operators work internally.

---

## 13. Common Interview Questions

### Q: What are nullable reference types and how do they help?
Since C# 8 (default in .NET 6+ templates), the compiler tracks nullability via annotations: `string` is assumed non-null, `string?` may be null. It emits **warnings** on possible null dereferences and on assigning null to non-nullable references, catching `NullReferenceException` bugs at *compile time*. It's compile-time only (no runtime cost or enforcement) — the C# analog in *purpose* to Java's `Optional`, but implemented as type annotations.

### Q: Difference between `?.`, `??`, and `??=`?
`?.` is null-conditional access (returns null instead of throwing if the target is null). `??` is null-coalescing (returns the right operand if the left is null). `??=` assigns the right operand only if the left is currently null. They chain: `order?.Items?.Count ?? 0`.

### Q: What's the difference between a switch statement and a switch expression?
A switch *statement* executes code blocks (`case`/`break`). A switch *expression* (`x switch { ... => ... }`) **evaluates to a value**, uses `=>` arms and `,` separators, `_` for default, and requires all cases be handled. It pairs with pattern matching (type, relational, property, list patterns).

### Q: When would you use a tuple vs a record?
Use a **value tuple** `(int, string)` for *local, short-lived* multiple return values where naming a type is overkill. Use a **record** for anything public, long-lived, or that needs a meaningful name, value equality, and `with` support. Tuples leak implementation detail across API boundaries; records document intent.

### Q: What do `init` and `required` do?
`init` makes a property settable only during object initialization, then read-only — immutability without a full constructor. `required` forces the caller to set a property at creation (compile-time enforced). Together they enable safe, immutable object-initializer construction without builder boilerplate.

### Q: What is `yield return`?
It builds a **lazy iterator**: each `yield return` produces one element on demand as the consumer iterates, without materializing the whole collection. It enables infinite/large sequences and deferred execution, and is how many LINQ operators are implemented internally.

### Q: What are primary constructors (C# 12)?
Constructor parameters declared on the class/struct header, in scope throughout the type — removing field-assignment boilerplate, especially for DI. On a `class` they don't auto-create properties (just captured values); on a `record` positional parameters become public properties.

### Q: What is the null-forgiving operator `!`?
`expr!` tells the compiler "I know this isn't null" and **suppresses the nullability warning**. It's compile-time only — no runtime null check is added. Overusing it defeats the purpose of NRTs; reserve it for cases the compiler can't prove but you can.

---

## 14. Quick Reference Cheat Sheet

```
STRINGS:
  $"Hi {name}, {score:P0}"     interpolation + format specifier
  """ ... """                  raw string literal (no escaping)
  formats: :D5 :F2 :C :P0 :yyyy-MM-dd

TUPLES / DECONSTRUCTION:
  (string Name, int Age) t = ("Amy", 30);   named value tuple
  var (a, b) = point;                        deconstruct
  (a, b) = (b, a);                           swap
  var (_, age) = t;                          discard with _

SWITCH EXPRESSION + PATTERNS:
  x switch { <0 => "neg", 0 => "zero", _ => "pos" }
  obj switch { Circle c => ..., null => ..., _ => ... }   type pattern
  o switch { { Total: > 100 } => "free" }                 property pattern
  arr switch { [] => .., [var x] => .., [var f, .., var l] => .. }  list pattern
  if (obj is Foo f) ...                                    is-pattern

NULLABILITY (enable in .csproj: <Nullable>enable</Nullable>):
  string  = non-nullable         string? = nullable
  a?.B    null-conditional       a ?? b  null-coalescing
  a ??= b assign-if-null         a!      null-forgiving (compile-time only)
  order?.Items?.Count ?? 0       idiomatic null-safe chain

IMMUTABILITY:
  { get; init; }     set only at creation
  required string X  caller must set it
  new Person { Name = "A", Age = 30 }   object initializer

MODERN SYNTAX:
  List<int> x = new();           target-typed new
  int[] a = [1, 2, 3];           collection expression
  int[] c = [.. a, 4, .. b];     spread
  nums[^1]   last element        nums[1..^1]  range (end exclusive)
  using var r = ...;             auto-dispose at scope end
  Console.WriteLine("hi");       top-level statement (no Main)
  yield return x;                lazy iterator

JAVA -> C#:
  String.format      -> $"..."
  text block """     -> raw string """
  Optional<T>        -> T? (NRT)
  list.get(n-1)      -> list[^1]
  subList(1,4)       -> list[1..4]
  try-with-resources -> using var

GOLDEN RULES:
  1. Enable NRTs and treat their warnings as errors.
  2. Prefer switch expressions + patterns over if/else ladders.
  3. Use new() and [..] — not the verbose old forms.
  4. Tuples for local multi-return; records for public shapes.
  5. init/required for immutable objects; primary ctors for DI services.
```

---

*Related guides: `CSharp_Fundamentals_For_Java_Devs.md`, `CSharp_OOP_And_Type_System.md` (records/structs), `CSharp_LINQ_Guide.md` (yield & lazy eval).*

*Last Updated: 2026-07-09*
