# C# Fundamentals for Java Developers

## Overview

If you already know Java, you know roughly 80% of C#. Both are statically-typed, object-oriented, garbage-collected languages with C-style syntax that compile to an intermediate bytecode (Java → JVM bytecode, C# → IL/CIL run on the .NET runtime). This guide maps the **fundamentals** of C# onto the Java concepts you already understand, so you stop memorizing and start translating. We use modern C# (.NET 8+). Wherever C# does something differently — properties instead of getter/setter boilerplate, `decimal` for money, value types via `struct`, real nullable value types — we call it out explicitly. Deeper topics (collections, LINQ, async, value-type internals) have their own guides; this one is your "Core Java basics" equivalent.

## Table of Contents

1. [Java → C# Quick Mapping](#1-java--c-quick-mapping)
2. [Hello World, Namespaces, and using](#2-hello-world-namespaces-and-using)
3. [Built-in Types](#3-built-in-types)
4. [Value Types vs Reference Types (Preview)](#4-value-types-vs-reference-types-preview)
5. [Variables: var, const, and readonly](#5-variables-var-const-and-readonly)
6. [Strings](#6-strings)
7. [Properties](#7-properties)
8. [Nullability](#8-nullability)
9. [Operators and Control Flow](#9-operators-and-control-flow)
10. [Methods](#10-methods)
11. [Boxing and Unboxing](#11-boxing-and-unboxing)
12. [Console I/O and String Formatting](#12-console-io-and-string-formatting)
13. [Equality](#13-equality)
14. [Common Interview Questions](#14-common-interview-questions)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. Java → C# Quick Mapping

**Think of it like...** a phrasebook for a country whose language shares your alphabet. You don't relearn how to read — you just swap a few words.

| Java concept | C# equivalent | Notes |
|---|---|---|
| `package com.app` | `namespace App` | Folder structure not enforced by namespace |
| `import java.util.List` | `using System.Collections.Generic;` | `using` also = try-with-resources |
| `public static void main(String[] args)` | `static void Main(string[] args)` or top-level statements | Top-level = no boilerplate |
| `System.out.println(x)` | `Console.WriteLine(x)` | |
| `String` | `string` (alias for `System.String`) | lowercase alias, same immutability |
| `int` / `Integer` | `int` (`System.Int32`) | One type, no separate wrapper |
| `boolean` | `bool` | |
| `BigDecimal` (money) | `decimal` | Built-in keyword, not a class |
| `final` (variable) | `const` or `readonly` | Two different tools |
| getter/setter methods | Properties `{ get; set; }` | Huge boilerplate reduction |
| `Integer x = null` | `int? x = null` | True nullable value types |
| `String.format("%s", x)` | `$"{x}"` interpolation | |
| `StringBuilder` | `StringBuilder` | Identical idea |
| enhanced for `for (T t : list)` | `foreach (var t in list)` | |
| `==` (refs) vs `.equals()` | `==` (often overloaded) vs `.Equals()` | C# `==` can be value-based |
| `Object` | `object` (`System.Object`) | |
| varargs `int... nums` | `params int[] nums` | |
| method overloading | method overloading | Same |
| autoboxing | boxing/unboxing | Same idea, explicit cost |

---

## 2. Hello World, Namespaces, and using

**Think of it like...** moving to a new apartment in the same city. The address format (`namespace`) and the way you "import" your neighbors' tools (`using`) look slightly different, but the city is the same.

In Java you must wrap everything in a class with a `main` method. Modern C# lets you skip that ceremony with **top-level statements**: the compiler generates the `Main` for you.

```csharp
// File: Program.cs
using System;                       // Like Java's "import java.lang.*" — brings names into scope

Console.WriteLine("Hello, World!"); // Top-level statement: no class, no Main needed (Java: System.out.println)
```

The classic, explicit form still exists and is what you'll see in larger apps:

```csharp
using System;                          // import the System namespace (Console lives here)

namespace MyApp                        // Java: package myapp; (but C# uses PascalCase and braces)
{
    class Program                      // a normal class, like Java's public class
    {
        static void Main(string[] args) // entry point; Java: public static void main(String[] args)
        {
            Console.WriteLine("Hello"); // Java: System.out.println("Hello");
        }
    }
}
```

Key differences from Java:
- **Namespaces do not have to match folders.** Java enforces `package` ↔ directory; C# does not (though teams usually keep them aligned by convention).
- **`using` has a second job.** As a statement it imports a namespace; as a block it disposes resources — exactly like Java's try-with-resources.

```csharp
using (var reader = new StreamReader("data.txt")) // Java: try (var reader = new ...) { }
{
    Console.WriteLine(reader.ReadLine());          // resource auto-disposed at end of block
}

// Modern shorthand — disposed when the enclosing scope ends:
using var file = new StreamReader("data.txt");     // no braces needed (C# 8+)
```

- **`global using`** (C#10+) imports a namespace for the entire project once, so you don't repeat it in every file.

```csharp
global using System;                 // available in every file of the project — no per-file repetition
```

---

## 3. Built-in Types

**Think of it like...** the same set of measuring cups in a different kitchen. A cup is still a cup; only the label font changed.

The biggest mental shift: in C# the lowercase keywords like `int` are **aliases for real .NET types** (`int` ≡ `System.Int32`). There is **no separate boxed wrapper class** like Java's `Integer`. One `int` type does both jobs; boxing happens only when you store it in an `object` (see section 11).

```csharp
int age = 30;            // 32-bit integer. Alias for System.Int32. Java: int (and Integer is gone)
long big = 9_000_000_000L; // 64-bit. Underscores allowed for readability. Java: long
double price = 19.99;    // 64-bit floating point. Java: double — NOT for money!
float ratio = 1.5f;      // 32-bit float, needs 'f' suffix. Java: float
decimal money = 19.99m;  // 128-bit exact decimal, 'm' suffix. Java's closest is BigDecimal
bool isReady = true;     // Java: boolean (note: C# spells it "bool")
char letter = 'A';       // single UTF-16 char, single quotes. Java: char
string name = "Amith";   // alias for System.String. Java: String (capital S)
```

**Use `decimal` for money**, never `double`. `double` is binary floating point, so `0.1 + 0.2 != 0.3`. `decimal` is base-10 and exact — it's the C# answer to Java's `BigDecimal`, but it's a built-in keyword and uses normal `+ - * /` operators (no `.add()` calls).

```csharp
double bad = 0.1 + 0.2;      // 0.30000000000000004  — rounding error, like Java double
decimal good = 0.1m + 0.2m;  // exactly 0.3 — use this for prices, totals, currency
```

Other handy types: `byte`, `short`, `uint`/`ulong` (unsigned — Java has no unsigned types), `sbyte`, and `nint`/`nuint` (native-sized). For 95% of junior work you'll live in `int`, `long`, `double`, `decimal`, `bool`, `string`.

---

## 4. Value Types vs Reference Types (Preview)

**Think of it like...** the difference between handing someone a photocopy (value type — they get their own copy) versus handing them the address of your house (reference type — you both point at the same place). *This is a preview; the deep dive lives in a separate guide.*

In Java, everything except the 8 primitives is a reference type. C# generalizes this: you can define your **own** value types using `struct`, not just `class`.

```csharp
class Pen { public string Color; }    // reference type — variable holds a REFERENCE (Java: class)
struct Point { public int X, Y; }     // value type — variable holds the DATA directly (no Java equivalent for user types)

Pen p1 = new Pen { Color = "Red" };
Pen p2 = p1;                          // copies the REFERENCE — both point to the same Pen
p2.Color = "Blue";                    // p1.Color is now "Blue" too (shared object)

Point a = new Point { X = 1, Y = 2 };
Point b = a;                          // copies the DATA — b is an independent copy
b.X = 99;                             // a.X is STILL 1 (separate value)
```

Rules of thumb (for now):
- `class` = reference type, lives on the heap, copied by reference — like every Java object.
- `struct` = value type, copied by value — like Java's primitives, but you can define your own.
- Built-in numbers (`int`, `double`, `bool`, etc.) are actually structs under the hood.
- `string` is a reference type but behaves like a value because it's immutable.

---

## 5. Variables: var, const, and readonly

**Think of it like...** labeling jars. `var` lets the compiler read the label for you; `const` is a label written in permanent marker at the factory; `readonly` is a label you can write once when you open the jar, then never again.

`var` is **compile-time type inference** — the variable is still strongly typed, just like Java's `var` (Java 10+). It is *not* dynamic.

```csharp
var count = 5;              // compiler infers int. Identical to Java 10+ "var count = 5;"
var name = "Amith";        // inferred as string
var list = new List<int>(); // inferred as List<int> — saves repeating the type
// var x;                  // ERROR: must initialize so the compiler can infer the type
```

Now the two "final-like" tools. Java has one `final`; C# splits it into **`const`** and **`readonly`**:

```csharp
const double Pi = 3.14159;   // compile-time constant. Must be set here. Like Java "static final" literal
                             // Baked into the code at compile time; can ONLY be a literal/constant expression.

class Circle
{
    public readonly int Id;  // runtime constant — set once, in declaration OR constructor
    public Circle(int id)
    {
        Id = id;             // legal: assigned in constructor (Java: blank final field)
    }
}
```

| Need | Java | C# |
|---|---|---|
| Compile-time literal constant | `static final int X = 5;` | `const int X = 5;` |
| Set once at runtime (e.g. in constructor) | `final int id;` | `readonly int id;` |
| Local you can't reassign | `final` local | (no direct keyword — just don't reassign) |

Gotcha: `const` values are **inlined** at compile time. If a `const` in a library changes, code that used it must be **recompiled** to see the new value. `readonly` is read at runtime, so it doesn't have that pitfall — prefer `readonly` for values that might change between versions.

---

## 6. Strings

**Think of it like...** a printed book (string). You can't erase a word on a printed page; to change it you print a new book. A `StringBuilder` is the rough draft on a whiteboard where you *can* edit freely.

Strings in C# are **immutable**, exactly like Java. Every "modification" creates a new string.

```csharp
string s = "hello";
s = s.ToUpper();           // creates a NEW string "HELLO"; original is unchanged (same as Java)
```

**String interpolation** with `$"..."` replaces Java's clunky `String.format` / concatenation:

```csharp
string name = "Amith";
int age = 30;
string msg = $"{name} is {age} years old";        // Java: String.format("%s is %d...", name, age)
string calc = $"Total: {2 + 3}, padded: {age,5}"; // expressions and alignment allowed inside {}
string money = $"Price: {19.99m:C}";              // ":C" = currency format -> "Price: $19.99"
```

**Verbatim strings** with `@"..."` ignore escape sequences — perfect for Windows paths and regex:

```csharp
string path = @"C:\Users\Amith\file.txt";  // no need to escape backslashes. Java has no equivalent (you'd write \\)
string raw  = @"Line1
Line2";                                     // literal newline preserved; multi-line allowed

string both = $@"C:\Users\{name}\file.txt"; // combine: interpolated AND verbatim
```

For heavy concatenation (loops), use **`StringBuilder`** — same reasoning as Java (avoid creating N throwaway strings):

```csharp
var sb = new StringBuilder();              // Java: new StringBuilder()
for (int i = 0; i < 3; i++)
{
    sb.Append(i).Append(",");              // mutable; no new string per append
}
string result = sb.ToString();             // "0,1,2," — Java: sb.toString()
```

---

## 7. Properties

**Think of it like...** a polite receptionist for a field. Outside callers think they're touching `Name` directly, but the receptionist (`get`/`set`) can validate, log, or compute before letting them in. This single feature kills the getter/setter boilerplate that bloats Java classes.

In Java you write `private String name;` plus `getName()` and `setName()`. C# gives you **properties**:

```csharp
class Person
{
    public string Name { get; set; }     // auto-property: replaces a field + getName() + setName()
    public int Age { get; set; }          // C# generates the hidden backing field for you
}

var p = new Person();
p.Name = "Amith";     // calls the SET accessor. Java: p.setName("Amith");
string n = p.Name;    // calls the GET accessor. Java: p.getName();
```

You can add logic when you need it (a "full" property with a backing field):

```csharp
class Account
{
    private decimal _balance;                 // backing field (convention: _camelCase)

    public decimal Balance
    {
        get => _balance;                      // expression-bodied getter
        set                                   // run validation before storing — impossible with a plain field
        {
            if (value < 0)                    // 'value' is the implicit parameter of the setter
                throw new ArgumentException("No negative balance");
            _balance = value;
        }
    }
}
```

**Init-only setters** (`init`) let you set a value only during object creation, then it's read-only — great for immutable objects:

```csharp
class Config
{
    public string ApiKey { get; init; }       // settable ONLY in an object initializer / constructor
}

var c = new Config { ApiKey = "abc123" };     // OK: set during creation (Java: constructor-only final field)
// c.ApiKey = "new";                          // ERROR after construction — effectively read-only
```

**Expression-bodied members** (`=>`) shorten one-line methods and computed properties:

```csharp
class Rectangle
{
    public int Width { get; set; }
    public int Height { get; set; }
    public int Area => Width * Height;         // computed read-only property (Java: int getArea(){return ...;})
    public void Print() => Console.WriteLine(Area); // expression-bodied method
}
```

This is the single biggest "feels nicer than Java" win for a junior dev — point it out in interviews.

---

## 8. Nullability

**Think of it like...** two kinds of empty mailbox. In Java only object mailboxes can be empty (`null`); a primitive `int` mailbox *always* has a number. C# adds a special "this number mailbox is allowed to be empty" sign (`int?`) and also lets you mark object mailboxes as "should never be empty."

**Nullable value types** — append `?` to make a value type hold `null`. This is C#'s answer to Java's `Integer`/`Optional` for primitives:

```csharp
int x = 5;          // can never be null (a value type)
int? y = null;      // nullable int — CAN be null. Java: Integer y = null;
                    // Under the hood this is Nullable<int>.

if (y.HasValue)                 // check before reading
    Console.WriteLine(y.Value); // unwrap the value
int z = y ?? 0;                 // if y is null, use 0 (null-coalescing) — like Java's Optional.orElse(0)
```

**Nullable reference types** (NRT, C# 8+): reference types are non-null by default, and you opt into nullability with `?`. The compiler warns you about possible null dereferences — a compile-time safety net Java lacks.

```csharp
#nullable enable                 // usually on by default in .NET 8 projects
string name = "Amith";           // must NOT be null — compiler warns if you assign null
string? maybe = null;            // explicitly allowed to be null
// Console.WriteLine(maybe.Length); // WARNING: 'maybe' may be null here
```

**Null operators** — your everyday null-handling toolkit:

```csharp
string? input = GetInput();

string a = input ?? "default";   // ?? null-coalescing: use right side if left is null
input ??= "fallback";            // ??= assign only if input is currently null

int? len = input?.Length;        // ?. null-conditional: if input is null, whole expr is null (no NPE)
                                 // Java: input == null ? null : input.length()  (but cleaner)

string upper = input?.ToUpper() ?? "EMPTY"; // chain them: safely call, then default
```

The combination `?.` + `??` replaces the verbose null-check ladders you write in Java, and `int?` finally gives primitives a clean "absent" state.

---

## 9. Operators and Control Flow

**Think of it like...** the same road signs as Java, plus a few new express lanes (`switch` *expressions* and pattern matching) that let you say more with less.

Operators are nearly identical to Java: `+ - * / %`, `== != < > <= >=`, `&& || !`, `& | ^`, `++ --`. Classic `if/else`, `while`, `do/while`, and C-style `for` are the same.

```csharp
for (int i = 0; i < 3; i++)            // identical to Java's classic for loop
    Console.WriteLine(i);

foreach (var item in new[] { "a", "b" }) // Java's enhanced for: for (var item : array)
    Console.WriteLine(item);             // works on anything IEnumerable (Java: Iterable)
```

The **`switch` expression** (C# 8+) is the big upgrade — it *returns a value*, no `break` needed, and supports **pattern matching**:

```csharp
int day = 3;

// Old statement style still exists (like Java's switch statement):
switch (day)
{
    case 1: Console.WriteLine("Mon"); break; // 'break' required, like Java
    default: Console.WriteLine("?");   break;
}

// Modern switch EXPRESSION — assigns a result, arrow syntax, no break:
string name = day switch
{
    1 => "Monday",                  // each arm returns a value
    2 => "Tuesday",
    3 or 4 => "Midweek",            // pattern: 'or' combines cases
    >= 5 => "Weekend-ish",          // relational pattern (compare to value)
    _ => "Unknown"                  // '_' is the default/discard (Java: default)
};
```

Pattern matching also works with `is`:

```csharp
object obj = "hello";

if (obj is string s)               // type pattern: tests type AND assigns to 's' in one step
    Console.WriteLine(s.Length);   // Java: if (obj instanceof String s) (Java 16+ caught up here)

string describe = obj switch
{
    string str when str.Length > 3 => "long string", // 'when' adds a guard condition
    int n => $"number {n}",
    null => "null",
    _ => "something else"
};
```

---

## 10. Methods

**Think of it like...** ordering at a restaurant. Normal arguments are a fixed menu; **named arguments** let you say "hold the onions" by name; **optional parameters** are default sides you can skip; `out`/`ref` are you handing the chef your own plate to fill or modify.

Basic methods look like Java. C# adds several conveniences:

```csharp
int Add(int a, int b) => a + b;          // expression-bodied method (Java: int add(int a,int b){return a+b;})

int Multiply(int a, int b)               // normal block body
{
    return a * b;
}
```

**Optional and named arguments** reduce overload explosion:

```csharp
void Log(string msg, string level = "INFO", bool stamp = true) // defaults make params optional
{
    Console.WriteLine($"[{level}] {msg}");
}

Log("Started");                          // uses defaults: level=INFO, stamp=true
Log("Oops", "ERROR");                    // positional: level overridden
Log("Hi", stamp: false);                 // named argument: skip 'level', set 'stamp' (Java has neither feature)
```

**`ref`, `out`, `in`** control how arguments are passed (Java is always pass-by-value):

```csharp
void Double(ref int n) => n *= 2;        // ref: caller's variable is modified (pass by reference)
int x = 5;
Double(ref x);                           // x is now 10 ('ref' required at call site too)

bool TryParse(string s, out int result)  // out: method MUST assign it; used to "return" extra values
{
    return int.TryParse(s, out result);  // common pattern — returns success + the parsed value
}
if (TryParse("42", out int val))         // declare the out variable inline
    Console.WriteLine(val);              // 42

double Distance(in Point p) => p.X;      // in: pass by reference but READ-ONLY (perf for big structs)
```

**`params` arrays** = Java's varargs:

```csharp
int Sum(params int[] nums)               // Java: int sum(int... nums)
{
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}
Sum(1, 2, 3, 4);                         // pass any number of args, or an int[] directly
```

**Method overloading** works exactly like Java — same name, different parameter lists:

```csharp
void Print(int x) => Console.WriteLine($"int: {x}");
void Print(string x) => Console.WriteLine($"str: {x}"); // compiler picks by argument types, like Java
```

---

## 11. Boxing and Unboxing

**Think of it like...** putting a coin (value type) inside a labeled box so it can sit on the same shelf as the parcels (objects). Taking it back out is unboxing. The boxing costs you a trip to the heap.

Because `int` is a value type but `object` is a reference type, storing a value type in an `object` (or non-generic collection) **boxes** it — wraps it in a heap allocation. This mirrors Java's autoboxing (`int` → `Integer`), but in C# there's no separate wrapper *class* — the same `int` gets boxed into a plain `object`.

```csharp
int n = 42;
object boxed = n;            // BOXING: copies the int onto the heap inside an object
int back = (int)boxed;       // UNBOXING: explicit cast required to get the value back
                             // Java: Integer boxed = n; int back = boxed; (auto)
```

Why care? Boxing allocates memory and hurts performance in hot loops. The fix is the same as Java: **use generics** so the value stays a value.

```csharp
var list = new List<int>();  // generic — stores ints directly, NO boxing (Java: List<Integer> still boxes!)
list.Add(5);                 // stays a value type — actually better than Java here

System.Collections.ArrayList old = new();
old.Add(5);                  // BOXES, because ArrayList holds 'object' — avoid the old non-generic collections
```

Note C# is slightly ahead of Java here: `List<int>` stores raw `int`s without boxing, whereas Java's `List<Integer>` always boxes.

---

## 12. Console I/O and String Formatting

**Think of it like...** the same megaphone (`Console`) as Java's `System.out`, just with shorter command names.

```csharp
Console.WriteLine("Hello");         // print + newline. Java: System.out.println(...)
Console.Write("No newline");        // print, no newline. Java: System.out.print(...)

Console.Write("Enter name: ");
string? name = Console.ReadLine();  // read a line from stdin (returns string?). Java: Scanner.nextLine()
int.TryParse(Console.ReadLine(), out int age); // read + safely parse a number
```

**Formatting** — prefer interpolation, but composite format strings (like Java's `printf`) exist too:

```csharp
decimal price = 1234.5m;
Console.WriteLine($"{price:C}");          // currency -> $1,234.50  (culture-aware)
Console.WriteLine($"{price:N2}");         // 2 decimals -> 1,234.50
Console.WriteLine($"{0.25:P}");           // percent -> 25.00 %
Console.WriteLine($"{42:D5}");            // pad number -> 00042
Console.WriteLine($"{255:X}");            // hex -> FF

Console.WriteLine("Hi {0}, age {1}", name, age); // positional placeholders (Java: System.out.printf)
```

Format specifiers: `C`=currency, `N`=number with separators, `P`=percent, `D`=integer with padding, `X`=hex, `F`=fixed-point.

---

## 13. Equality

**Think of it like...** comparing two ID cards. Are they the *same physical card* (reference equality) or just *the same person's details* (value equality)? Java draws this line with `==` vs `.equals()`. C# draws it too, but `==` can be **overloaded** to mean value equality — so you must know the type.

The core rules:

```csharp
// For STRINGS, == compares VALUE (C# overloads == for string). This differs from Java!
string a = "hi";
string b = "hi";
Console.WriteLine(a == b);             // True — compares characters (Java: a == b is UNRELIABLE; use a.equals(b))
Console.WriteLine(a.Equals(b));        // True — also value comparison

// For your own CLASS, == defaults to REFERENCE equality (same as Java ==)
var p1 = new Person { Name = "X" };
var p2 = new Person { Name = "X" };
Console.WriteLine(p1 == p2);           // False — different objects (Java: same behavior)
Console.WriteLine(p1.Equals(p2));      // False by default — override Equals to make it value-based

// ReferenceEquals ALWAYS checks identity, ignoring any overloaded == or Equals
Console.WriteLine(ReferenceEquals(a, b)); // checks if literally the same object (Java: System.identityHashCode-ish / ==)
```

Summary table:

| Comparison | Class (default) | string | int / struct |
|---|---|---|---|
| `==` | reference identity | **value** (overloaded) | value |
| `.Equals()` | reference (unless overridden) | value | value |
| `ReferenceEquals()` | identity | identity | always boxes → usually false |

The Java trap "`==` on strings compares references" does **not** apply in C# — string `==` is value-based by default. But the general advice still holds: for your own types, override `Equals` (and `GetHashCode`, like Java's `hashCode`) to get value semantics, or use a `record` which does it automatically (covered in another guide).

```csharp
int x = 5, y = 5;
Console.WriteLine(x == y);              // True — value comparison for value types
```

---

## 14. Common Interview Questions

### Q: Is C# `string` the same as Java `String`?
Functionally yes: both are immutable reference types, both intern literals, both should be built with `StringBuilder` in loops. The differences: C# writes it lowercase as an alias for `System.String`, and the `==` operator does a **value** comparison by default (in Java `==` compares references, so you'd use `.equals()`).

### Q: When do I use `decimal` instead of `double`?
Use `decimal` for money and any value where exactness matters (currency, accounting). `double`/`float` are binary floating point and can't represent `0.1` exactly, causing rounding errors. `decimal` is base-10 and exact. It's the C# equivalent of Java's `BigDecimal`, but it's a built-in keyword with normal operators.

### Q: What's the difference between `const` and `readonly`?
`const` is a compile-time constant — must be a literal, is inlined into callers at compile time, and is implicitly static. `readonly` is a runtime constant — it can be assigned in the declaration or the constructor, so it can hold computed/injected values. Java's `final` covers both roles; C# splits them. Prefer `readonly` for anything that might change between library versions, because `const` values get baked into consumers and require recompilation.

### Q: How do properties improve on Java getters/setters?
A property like `public string Name { get; set; }` gives you a field-like syntax (`p.Name = "x"`) while keeping the ability to insert validation, logging, or computation later — without changing the call sites. It collapses the field + `getName()` + `setName()` boilerplate into one line. You can also make them `init`-only (set just at creation) or computed with `=>`.

### Q: What is `var` — is it dynamic like JavaScript?
No. `var` is **compile-time type inference**, identical to Java 10's `var`. The variable has a fixed static type; the compiler just figures it out from the initializer. For actual dynamic typing C# has a separate `dynamic` keyword, which you rarely need.

### Q: Explain nullable value types (`int?`).
`int?` is shorthand for `Nullable<int>`, letting a value type hold `null` — the role Java's `Integer` plays for `int`. Check `.HasValue`, read `.Value`, or use `?? defaultValue` to unwrap safely. Separately, **nullable reference types** (`string?`) make the compiler warn about potential null dereferences, a safety feature Java doesn't have built in.

### Q: What do `?.` and `??` do?
`?.` is the null-conditional operator: `a?.B` returns `null` instead of throwing if `a` is null — avoiding `NullReferenceException` (C#'s NPE). `??` is null-coalescing: `a ?? b` returns `b` when `a` is null. `??=` assigns only if the left side is currently null. Together they replace verbose Java null-check ladders.

### Q: What's boxing and why does it matter?
Boxing wraps a value type (like `int`) into a heap-allocated `object`; unboxing casts it back. It's C#'s version of Java autoboxing. It matters because it allocates memory and hurts performance in hot loops. Avoid it by using generics (`List<int>`), which — unlike Java's `List<Integer>` — actually store the raw value without boxing.

### Q: How does a `switch` expression differ from a `switch` statement?
The classic `switch` statement runs code and needs `break` (like Java). A `switch` **expression** *returns a value* using `case => result` arrows, requires no `break`, supports pattern matching (`>= 5`, `or`, type patterns, `when` guards), and uses `_` for the default. It's more concise and is preferred in modern C#.

### Q: How does equality work for your own classes by default?
By default `==` and `.Equals()` on a custom `class` use **reference identity** — two objects with identical data are not equal (same as Java). To get value equality, override `Equals` and `GetHashCode` (Java's `equals`/`hashCode`), or use a `record`, which generates value-based equality for you automatically.

### Q: What's the difference between `ref` and `out` parameters?
Both pass by reference so the method can modify the caller's variable. `ref` requires the variable to be **initialized before** the call. `out` does **not** — the method is required to assign it before returning, so it's used to "return" extra values (the classic `int.TryParse(s, out int result)` pattern). `in` is read-only pass-by-reference, used for performance with large structs.

### Q: Are namespaces tied to folders like Java packages?
No. Java enforces that a `package` matches its directory path; C# `namespace` is independent of the file system, though teams usually keep them aligned by convention. Also, `using` imports a namespace (like `import`) but additionally powers resource disposal (`using var x = ...`), C#'s try-with-resources.

### Q: What are top-level statements?
Modern C# (.NET 6+) lets `Program.cs` contain executable statements directly, without an enclosing class or explicit `Main` — the compiler generates them. It removes the `public static void main` ceremony for small programs and entry points, while larger apps can still use the explicit class + `Main` form.

---

## 15. Quick Reference Cheat Sheet

```
ENTRY POINT
  Console.WriteLine("Hi");           // top-level statement, no class/Main needed
  static void Main(string[] args)    // explicit entry point (Java: public static void main)

NAMESPACES / IMPORTS
  namespace App { }                  // ~ Java package (NOT tied to folders)
  using System;                      // ~ Java import
  global using System;               // import for the WHOLE project (C#10+)
  using var f = new StreamReader();  // try-with-resources (auto-dispose)

BUILT-IN TYPES (lowercase aliases)
  int long short byte               // integers (uint/ulong = unsigned, no Java equiv)
  double(d) float(f) decimal(m)     // decimal = money/exact (Java BigDecimal)
  bool char string object           // bool not boolean; string/object lowercase

MONEY
  decimal total = 19.99m;            // NEVER use double for money

VALUE vs REFERENCE (preview)
  class  -> reference type (heap, copied by reference)  ~ all Java objects
  struct -> value type   (copied by value)              ~ define-your-own primitive

VARIABLES
  var x = 5;                         // compile-time inference (Java 10 var)
  const double Pi = 3.14;            // compile-time literal, inlined
  readonly int id;                   // set once (decl or constructor)

STRINGS (immutable)
  $"{name} is {age}"                 // interpolation (Java String.format)
  $"{price:C}" $"{n:N2}" $"{n:D5}"   // currency / 2-dp / pad
  @"C:\path\no\escapes"              // verbatim (raw)
  new StringBuilder().Append(x)      // for loops/concatenation

PROPERTIES (replace getters/setters)
  public string Name { get; set; }   // auto-property
  public int Area => W * H;           // computed / expression-bodied
  public string Key { get; init; }    // set only at creation (immutable)
  // inside set: 'value' is the incoming value; add validation here

NULLABILITY
  int? n = null;  n.HasValue  n.Value // nullable value type (Java Integer)
  string? maybe = null;               // nullable reference type (compiler warns)
  a ?? b      // if a null -> b        (Optional.orElse)
  a ??= b     // assign only if a null
  a?.Prop     // null-safe access (no NullReferenceException)

CONTROL FLOW
  foreach (var x in list) { }         // Java enhanced for
  result = day switch { 1=>"Mon", >=5=>"wknd", _=>"?" }; // switch expression
  if (obj is string s) { }            // type pattern (instanceof + bind)
  case int n when n > 0 => ...        // guarded pattern

METHODS
  int Add(int a,int b) => a+b;        // expression body
  void Log(string m, string lvl="INFO")  // optional param
  Log("hi", lvl:"WARN");              // named argument
  void D(ref int n)   D(ref x);       // pass by reference (init required)
  bool Try(out int r) Try(out int r); // method assigns r (TryParse pattern)
  double F(in Point p)                // read-only by-ref (perf)
  int Sum(params int[] n)             // varargs

BOXING
  object o = 5;       // BOX (heap alloc)
  int i = (int)o;     // UNBOX (explicit cast)
  List<int>           // generic: NO boxing (better than Java List<Integer>)

EQUALITY
  string ==           // VALUE compare (UNLIKE Java! Java needs .equals)
  class ==            // reference identity by default (like Java)
  .Equals()           // value for string/struct; override for your class
  ReferenceEquals(a,b)// always identity

CONSOLE
  Console.WriteLine(x)  Console.Write(x)
  string? s = Console.ReadLine();
  int.TryParse(s, out int n);
```

*Last Updated: 2026-06-16*
