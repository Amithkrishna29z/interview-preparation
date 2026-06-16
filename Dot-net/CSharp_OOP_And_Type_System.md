# C# OOP and Type System (for Java Developers)

## Overview

This guide teaches C# object-oriented programming and its type system by **mapping every concept to the Java you already know**. If you can write a Java class, you can write a C# class — but C# has sharper rules in a few critical places that interviewers love to probe: methods are **not virtual by default** (the opposite of Java), C# has true **value types** (`struct`), C# generics are **reified** (no type erasure), and C# `record`/`enum`/interface features go beyond Java's.

Read top to bottom the first time. After that, use the [Cheat Sheet](#16-quick-reference-cheat-sheet) and [Interview Questions](#15-common-interview-questions) for revision. All code targets **modern C# (.NET 8+)**.

---

## Table of Contents

1. [Java → C# Quick Mapping](#1-java--c-quick-mapping)
2. [Classes, Objects, Constructors, `this`, Object Initializers](#2-classes-objects-constructors-this-object-initializers)
3. [Access Modifiers](#3-access-modifiers)
4. [Inheritance, `base`, `sealed`](#4-inheritance-base-sealed)
5. [Polymorphism: `virtual` / `override` / `new`, `abstract`](#5-polymorphism-virtual--override--new-abstract)
6. [Interfaces](#6-interfaces)
7. [`struct` vs `class`: Value vs Reference Types](#7-struct-vs-class-value-vs-reference-types)
8. [`record` Types](#8-record-types)
9. [Enums](#9-enums)
10. [Generics (and No Type Erasure)](#10-generics-and-no-type-erasure)
11. [Static Classes, Members, Static Constructors](#11-static-classes-members-static-constructors)
12. [Partial and Nested Classes](#12-partial-and-nested-classes)
13. [`object` Members: `ToString`, `Equals`, `GetHashCode`](#13-object-members-tostring-equals-gethashcode)
14. [`is` / `as`, Pattern Matching, Boxing Pitfalls](#14-is--as-pattern-matching-boxing-pitfalls)
15. [Common Interview Questions](#15-common-interview-questions)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Java → C# Quick Mapping

**Think of it like...** moving to a new city where they speak almost your language — same grammar, a few words swapped, and a couple of traffic rules reversed (look the *other* way before crossing on the `virtual` street).

| Java | C# | Notes |
|------|-----|-------|
| `extends` | `:` | Same `:` is also used for interfaces |
| `implements` | `:` | C# uses one colon for both; base class comes first |
| `super` | `base` | Call parent constructor/method |
| `final` class | `sealed` class | Cannot be inherited |
| `final` method | (methods are non-virtual by default) | Just don't mark it `virtual` |
| `@Override` | `override` (a **keyword**, required) | Not optional, not an annotation |
| methods virtual **by default** | methods **non-virtual** by default | **Biggest gotcha** — use `virtual`/`override` |
| `Object` | `object` (alias for `System.Object`) | |
| `toString()` | `ToString()` | PascalCase methods |
| `equals()` / `hashCode()` | `Equals()` / `GetHashCode()` | |
| `instanceof` | `is` | Plus rich pattern matching |
| cast `(Foo) x` | `(Foo)x` or `x as Foo` | `as` returns `null` instead of throwing |
| `package-private` (default) | `internal` | Visible within the **assembly** (DLL), not the namespace |
| `interface` default methods | default interface methods (C# 8+) | Similar idea |
| `record` (Java 16+) | `record` / `record struct` | C# records are richer (`with`, mutability options) |
| `enum` (full classes) | `enum` (named integer constants) | C# enums are simpler; add behavior via extension methods |
| generics (type-erased) | generics (**reified**, no erasure) | C# knows `T` at runtime |
| `T extends X` | `where T : X` | Constraint syntax differs |
| `? extends X` / `? super X` | `out T` / `in T` (variance on interfaces) | Declaration-site, not use-site |
| no value types (except primitives) | `struct` = real user-defined value types | Big deal for performance |
| `static { }` init block | `static` constructor | Runs once before first use |
| `String` | `string` (alias for `System.String`) | Reference type, immutable, value equality on `==` |

---

## 2. Classes, Objects, Constructors, `this`, Object Initializers

**Think of it like...** a Java class with the keywords lightly rearranged — properties replace getter/setter boilerplate, and object initializers let you set fields at construction without writing a constructor for every combo.

```csharp
public class Person                        // 'public class' — same as Java
{
    // Auto-property: compiler generates a hidden backing field + get/set.
    // In Java you'd write a private field + getX()/setX(). C# folds it into one line.
    public string Name { get; set; }       // read/write property
    public int Age { get; private set; }   // public read, private write (set only inside class)

    // A field (like a Java field). Prefer properties for public data.
    private readonly DateTime _created;     // 'readonly' ≈ Java 'final' field (set once, in ctor)

    // Constructor — same name as the class, like Java.
    public Person(string name, int age)
    {
        Name = name;                        // assign via property
        Age = age;
        _created = DateTime.UtcNow;          // readonly field must be set here (or at declaration)
    }

    // Constructor chaining with ': this(...)' — like Java 'this(...)' but BEFORE the body.
    public Person(string name) : this(name, 0) { }  // calls the 2-arg ctor

    public void HaveBirthday() => Age++;     // expression-bodied method (=>), concise one-liner
}
```

Creating objects and using **object initializers**:

```csharp
var p1 = new Person("Ada", 36);             // 'var' = inferred type (like Java 'var')

// Object initializer: set properties right after construction, no special ctor needed.
// Java has no direct equivalent — you'd need a builder or many constructors.
var p2 = new Person("Linus")                 // calls 1-arg ctor...
{
    // ...then these run AFTER the constructor:
    Name = "Linus Torvalds"                  // overrides what the ctor set
    // Note: can only set properties that are accessible/settable here
};
```

`required` members (C# 11+) force the caller to set them, replacing some constructor boilerplate:

```csharp
public class Config
{
    public required string Host { get; init; }  // MUST be set in an initializer; 'init' = set only during construction
    public int Port { get; init; } = 8080;       // default value
}

var c = new Config { Host = "localhost" };        // compiler error if Host is omitted
// c.Host = "other";  // ERROR: 'init' properties are immutable after construction
```

---

## 3. Access Modifiers

**Think of it like...** Java's modifiers plus one extra concept: `internal` controls visibility across **assemblies** (compiled DLLs), which is C#'s answer to "package-private" but at a coarser, project-level scope.

| C# Modifier | Visible to... | Closest Java |
|-------------|---------------|--------------|
| `public` | Everyone | `public` |
| `private` | Same class only | `private` |
| `protected` | Same class + subclasses | `protected` (roughly) |
| `internal` | Same **assembly** (DLL) only | package-private (but assembly-wide, not folder-wide) |
| `protected internal` | Same assembly **OR** any subclass (union) | no direct equivalent |
| `private protected` | Same assembly **AND** subclass (intersection) | no direct equivalent |
| `file` (C# 11+) | Same source **file** only | no equivalent |

```csharp
public class Account
{
    private decimal _balance;                 // only this class
    protected string AccountId;               // this class + subclasses
    internal int InternalCode;                // anywhere in THIS DLL
    public string Owner { get; set; }         // everyone

    // protected internal: subclasses (even in other DLLs) OR same-DLL code can see it
    protected internal void Audit() { }
}
```

> **Default access** in C#: class members default to `private`; top-level types default to `internal`. (In Java, the default is package-private.)

**Think of it like...** `internal` = "friends within my project." If you split your code into multiple DLLs, `internal` keeps helpers hidden from consumers of your library — exactly what library authors want.

---

## 4. Inheritance, `base`, `sealed`

**Think of it like...** `extends` with a colon. `base` is just `super` renamed. `sealed` is `final` for classes.

```csharp
public class Animal                            // base class
{
    public string Name { get; }
    public Animal(string name) => Name = name; // ctor

    public virtual string Speak() => "...";    // 'virtual' = overridable (see next section)
}

public class Dog : Animal                      // ': Animal' means 'extends Animal'
{
    public Dog(string name) : base(name) { }   // 'base(name)' = Java 'super(name)'

    public override string Speak() => "Woof";  // 'override' required to replace virtual method

    public string Describe()
        => $"{base.Speak()} but really {Speak()}"; // base.Speak() calls the parent version
}

public sealed class Poodle : Dog               // 'sealed' = cannot be subclassed (Java 'final class')
{
    public Poodle(string name) : base(name) { }
}
// public class X : Poodle { }  // COMPILE ERROR: cannot derive from sealed 'Poodle'
```

Key rules vs Java:
- C# is **single inheritance** for classes (like Java), multiple for interfaces (like Java).
- If a base class has no parameterless constructor, derived constructors **must** call `: base(...)`.
- You can also `sealed` an **override** to stop further overriding while still inheriting:
  ```csharp
  public override sealed string Speak() => "Final woof"; // can't be overridden by subclasses of THIS class
  ```

---

## 5. Polymorphism: `virtual` / `override` / `new`, `abstract`

**Think of it like...** Java where *every* method is `virtual` automatically. In C#, you must **opt in** with `virtual`, and the subclass must **opt in** with `override`. Forget, and you get *method hiding* — a subtle bug interviewers love.

### The #1 difference from Java

```csharp
public class Base
{
    public void Foo() => Console.WriteLine("Base.Foo");      // NOT virtual (default)
    public virtual void Bar() => Console.WriteLine("Base.Bar"); // virtual — overridable
}

public class Derived : Base
{
    // 'new' HIDES the non-virtual Foo. This is NOT polymorphism!
    public new void Foo() => Console.WriteLine("Derived.Foo");

    // 'override' truly replaces Bar — real polymorphism.
    public override void Bar() => Console.WriteLine("Derived.Bar");
}

Base b = new Derived();
b.Foo();   // prints "Base.Foo"    <-- chosen by the VARIABLE type (hiding). Surprises Java devs!
b.Bar();   // prints "Derived.Bar" <-- chosen by the OBJECT type (override). Like Java always does.
```

In Java, `b.Foo()` would print `Derived.Foo` because Java methods are virtual by default. **In C#, only `virtual`+`override` gives you that.**

- `new` keyword = "I know I'm hiding the parent's member; do it on purpose." Without it the compiler warns.
- Choosing `virtual`-by-default-off is a deliberate C# design: it makes the API author decide what's safe to override.

### `abstract` classes and methods

**Think of it like...** Java abstract classes — identical concept.

```csharp
public abstract class Shape                    // cannot be instantiated
{
    public abstract double Area();             // no body — subclasses MUST implement (implicitly virtual)

    public virtual string Describe()           // has a body but can be overridden
        => $"A shape with area {Area():0.00}";
}

public class Circle : Shape
{
    public double Radius { get; init; }
    public override double Area()              // 'override' required for abstract members too
        => Math.PI * Radius * Radius;
}

// Shape s = new Shape();  // ERROR: cannot instantiate abstract class
Shape s = new Circle { Radius = 2 };           // OK — polymorphism through the abstract base
Console.WriteLine(s.Describe());               // calls Circle.Area() via abstract dispatch
```

---

## 6. Interfaces

**Think of it like...** Java interfaces, including default methods (C# 8+). C# adds **explicit interface implementation** for resolving name clashes.

```csharp
public interface IRepository<T>                // interfaces are conventionally prefixed with 'I'
{
    T? GetById(int id);                        // method signature (no body)
    void Save(T entity);

    int Count => 0;                            // DEFAULT interface method (C# 8+), like Java's 'default'
}

public class UserRepository : IRepository<User> // ': IRepository<User>' = 'implements'
{
    public User? GetById(int id) => /* ... */ null; // must implement non-default members
    public void Save(User entity) { /* ... */ }
    // Count is inherited from the default implementation; override only if needed.
}
```

### Multiple interface inheritance

```csharp
public interface IReadable { string Read(); }
public interface IWritable { void Write(string s); }

// A class can implement many interfaces (like Java). Base class, if any, comes FIRST:
public class File : object, IReadable, IWritable
{
    public string Read() => "data";
    public void Write(string s) { }
}
```

### Explicit interface implementation

**Think of it like...** a way to implement two interfaces that declare the same method differently, or to "hide" an interface method from the public surface.

```csharp
public interface IEnglish { string Greet(); }
public interface IFrench  { string Greet(); }

public class Bilingual : IEnglish, IFrench
{
    // Explicit implementations — note: NO access modifier, qualified by interface name.
    string IEnglish.Greet() => "Hello";        // only callable via IEnglish reference
    string IFrench.Greet()  => "Bonjour";      // only callable via IFrench reference
}

var x = new Bilingual();
// x.Greet();                       // ERROR: not on the public surface
Console.WriteLine(((IEnglish)x).Greet()); // "Hello"  — cast to pick the implementation
Console.WriteLine(((IFrench)x).Greet());  // "Bonjour"
```

---

## 7. `struct` vs `class`: Value vs Reference Types

**Think of it like...** the difference between Java `int` (copied by value, lives on the stack/inline) and a Java object (a reference to the heap). C# lets *you* create your own value types with `struct`. This is one of C#'s biggest advantages and a guaranteed interview topic.

| Aspect | `class` (reference type) | `struct` (value type) |
|--------|--------------------------|------------------------|
| Stored | Heap; variable holds a reference | Inline / stack (or inside its container) |
| Copy semantics | Copies the **reference** (alias) | Copies the **whole value** |
| `null` | Allowed | Not allowed (unless `Nullable<T>` / `T?`) |
| Default value | `null` | All fields zeroed |
| Inheritance | Yes (single base) | No base class inheritance (only interfaces) |
| Equality (default) | Reference identity | Field-by-field value equality |
| Use for | Most things, entities, large objects | Small, immutable, short-lived data (points, money) |

```csharp
public struct Point                            // value type
{
    public int X { get; init; }
    public int Y { get; init; }
    public Point(int x, int y) { X = x; Y = y; }
}

Point a = new Point(1, 2);
Point b = a;                                   // COPIES all fields — b is independent
b = b with { X = 99 };                          // (record struct only) — see below; struct shown for contrast
// With a plain struct you'd reassign: b = new Point(99, 2);
// 'a' is unchanged because the assignment copied the value, not a reference.

class Box { public int V; }
Box c = new Box { V = 1 };
Box d = c;                                     // copies the REFERENCE — c and d are the same object
d.V = 99;                                      // c.V is now also 99 (aliasing)
```

### `readonly struct` and `record struct`

```csharp
// readonly struct: ALL fields immutable; compiler enforces no mutation. Great for value semantics + safety.
public readonly struct Money
{
    public decimal Amount { get; }             // get-only
    public string Currency { get; }
    public Money(decimal amount, string currency) => (Amount, Currency) = (amount, currency);
}

// record struct: value type WITH auto value-equality, ToString, and 'with' expressions (C# 10+).
public readonly record struct Temperature(double Celsius)
{
    public double Fahrenheit => Celsius * 9 / 5 + 32; // computed property
}

var t1 = new Temperature(20);
var t2 = t1 with { Celsius = 25 };             // non-destructive copy with one change
Console.WriteLine(t1 == new Temperature(20));  // True — value equality
```

### When to use `struct`
- Small (≤ ~16 bytes is a common guideline), logically a single value (coordinate, color, money).
- Immutable.
- You don't need inheritance.
- Otherwise prefer `class`. Large structs copied often hurt performance (the opposite of the usual intent).

> **Boxing warning:** treating a struct as `object` or an interface copies it onto the heap (boxing) — see §14.

---

## 8. `record` Types

**Think of it like...** Java 16 `record`s on steroids. C# records are reference types by default (`record class`) with compiler-generated **value equality**, `ToString`, and the `with` expression for non-destructive copies.

```csharp
// Positional record: parameters become public init-only properties automatically.
public record Person(string FirstName, string LastName); // like Java record, but...

var p1 = new Person("Ada", "Lovelace");
var p2 = new Person("Ada", "Lovelace");
Console.WriteLine(p1 == p2);                   // True — VALUE equality (Java records: equals() yes, '==' no)
Console.WriteLine(p1);                          // Person { FirstName = Ada, LastName = Lovelace } — auto ToString

// 'with' expression: copy p1 but change LastName. p1 stays unchanged (immutability-friendly).
var p3 = p1 with { LastName = "Byron" };       // Java has no built-in equivalent
Console.WriteLine(p3);                          // Person { FirstName = Ada, LastName = Byron }

// Deconstruction (like a Java record's component accessors, but positional):
var (first, last) = p3;                         // first = "Ada", last = "Byron"
```

Records can have bodies, methods, inheritance, and mutable members if you want:

```csharp
public record Employee(string Name, int Id)
{
    public string Department { get; init; } = "General"; // extra init-only property
    public string Badge => $"{Id}:{Name}";               // computed property
}
```

| Feature | Java `record` | C# `record class` |
|---------|---------------|-------------------|
| Value equality | Yes (`equals`) | Yes (`==` and `Equals`) |
| `==` value equality | No (reference) | **Yes** |
| Non-destructive copy | Manual | `with` expression |
| Mutable members | No | Optional (`set`) |
| Inheritance | No | **Yes** |
| Value-type variant | No | `record struct` |

---

## 9. Enums

**Think of it like...** C named integer constants — simpler than Java's full-blown enum classes. To add *behavior*, you attach **extension methods** instead of writing methods inside the enum (Java lets you put methods directly in the enum).

```csharp
public enum Status                             // backed by 'int' by default
{
    Active,                                     // = 0
    Suspended,                                  // = 1
    Closed                                      // = 2
}

// Custom backing type + explicit values:
public enum HttpCode : short                   // ': short' sets the underlying integer type
{
    Ok = 200,
    NotFound = 404,
    ServerError = 500
}

Status s = Status.Active;
int n = (int)s;                                // explicit cast to underlying value -> 0
Status back = (Status)1;                       // cast int back -> Status.Suspended
string name = s.ToString();                    // "Active"
Status parsed = Enum.Parse<Status>("Closed");  // string -> enum
```

### `[Flags]` enums (bitwise combinations)

**Think of it like...** Java's `EnumSet`, but using bit flags on a single integer.

```csharp
[Flags]                                        // marks this enum as bit flags (affects ToString + intent)
public enum Permissions
{
    None    = 0,
    Read    = 1,                                // 0b0001
    Write   = 2,                                // 0b0010
    Execute = 4,                                // 0b0100
    All     = Read | Write | Execute            // 0b0111 — combine with bitwise OR
}

var p = Permissions.Read | Permissions.Write;  // combine flags
bool canWrite = p.HasFlag(Permissions.Write);  // True
Console.WriteLine(p);                          // "Read, Write" (Flags makes ToString list them)
```

### Adding behavior via extension methods

```csharp
public static class StatusExtensions
{
    // 'this Status' makes this an EXTENSION method — call it like an instance method.
    public static bool IsTerminal(this Status s) => s == Status.Closed;
}

bool done = Status.Closed.IsTerminal();        // reads like a method ON the enum
```

> Unlike Java, C# enums **cannot** hold extra fields or have constructors. For rich behavior, use a `class` (or the "smart enum"/`record` pattern) instead.

---

## 10. Generics (and No Type Erasure)

**Think of it like...** Java generics, but the type information **survives to runtime** (reified). No `List<String>` vs `List<Integer>` collapsing to raw `List`. You can write `typeof(T)`, `new T()`, and `is List<int>` — all impossible in Java.

```csharp
public class Box<T>                            // generic class, like Java's class Box<T>
{
    private T _value;
    public Box(T value) => _value = value;
    public T Get() => _value;
    public Type GetItemType() => typeof(T);    // RUNTIME type — Java can't do this (erasure)!
}

var b = new Box<int>(5);
Console.WriteLine(b.GetItemType());            // System.Int32 — real type at runtime
```

### Generic methods

```csharp
public static T Max<T>(T a, T b) where T : IComparable<T> // constraint
    => a.CompareTo(b) >= 0 ? a : b;

int m = Max(3, 9);                             // T inferred as int — prints 9
```

### Constraints (`where`)

**Think of it like...** Java's `<T extends X>`, with a richer vocabulary:

```csharp
public class Service<T>
    where T : class                            // T must be a reference type (Java: no equivalent)
    where T : new()                            // ...also can't combine 'class' and 'new()' on same line; shown separately below
{ }

// Common constraints:
//   where T : class           -> reference type
//   where T : struct          -> non-nullable value type
//   where T : new()           -> has a public parameterless constructor
//   where T : SomeBaseClass   -> derives from a class
//   where T : ISomeInterface  -> implements an interface (like Java 'extends')
//   where T : notnull         -> non-nullable
//   where T : U               -> T derives from another type parameter U

public class Factory<T> where T : new()        // requires a no-arg constructor
{
    public T Create() => new T();              // 'new T()' — IMPOSSIBLE in Java due to erasure!
}
```

### Covariance (`out`) and Contravariance (`in`)

**Think of it like...** Java's `? extends T` (covariance) and `? super T` (contravariance), but declared **once on the interface** (declaration-site) instead of at every use.

```csharp
// 'out T' = COVARIANT: IEnumerable<Dog> is usable as IEnumerable<Animal> (T only comes OUT).
public interface IProducer<out T> { T Produce(); }

IProducer<Dog> dogs = /* ... */ null!;
IProducer<Animal> animals = dogs;              // OK — covariance (Java: IEnumerable<? extends Animal>)

// 'in T' = CONTRAVARIANT: IConsumer<Animal> is usable as IConsumer<Dog> (T only goes IN).
public interface IConsumer<in T> { void Consume(T item); }

IConsumer<Animal> anyAnimal = /* ... */ null!;
IConsumer<Dog> dogConsumer = anyAnimal;        // OK — contravariance (Java: Consumer<? super Dog>)
```

### No type erasure — why interviewers ask

| Capability | Java (erased) | C# (reified) |
|------------|---------------|--------------|
| `obj is List<int>` at runtime | No | **Yes** |
| `typeof(T)` / `T.class` of the param | No | **Yes** (`typeof(T)`) |
| `new T()` | No | **Yes** (with `new()` constraint) |
| `List<int>` vs `List<string>` distinct at runtime | No (same raw class) | **Yes** (distinct types) |
| Primitives in generics | No (boxing only, `List<Integer>`) | **Yes** (`List<int>` stores ints directly) |

That last row matters for performance: `List<int>` in C# stores raw `int`s; Java must box every element into `Integer`.

---

## 11. Static Classes, Members, Static Constructors

**Think of it like...** Java statics, plus a `static class` (a class that *only* holds statics — Java approximates this with a `final` class and a private constructor).

```csharp
public static class MathUtils                  // 'static class': cannot be instantiated or inherited
{
    public static double Pi = 3.14159;         // static field — one copy for the whole app
    public static double Square(double x) => x * x; // static method

    static MathUtils()                          // STATIC CONSTRUCTOR (like Java's 'static { }' block)
    {
        // Runs ONCE, automatically, before the first access to any member. No access modifier, no params.
        Console.WriteLine("MathUtils initialized");
    }
}

double area = MathUtils.Square(3);             // call without an instance
```

- A `static class` can contain only static members (compiler-enforced).
- A **static constructor** runs once, lazily, on first use — same role as Java's static initializer block.
- Non-static classes can also have a static constructor for one-time setup of static state.

---

## 12. Partial and Nested Classes

**Think of it like...** Nested classes are Java's `static` nested classes. Partial classes are *new to Java devs*: one class split across multiple files — heavily used by code generators (e.g., UI designers, source generators).

```csharp
// File A: Order.Core.cs
public partial class Order                     // 'partial' — definition continues elsewhere
{
    public int Id { get; set; }
}

// File B: Order.Generated.cs
public partial class Order                     // SAME class, merged by the compiler
{
    public decimal Total { get; set; }
}
// At compile time these become a single 'Order' class with both members.
```

Nested classes:

```csharp
public class Outer
{
    private int _secret = 42;

    public class Nested                        // nested type — like a Java 'static nested class'
    {
        // Note: in C#, a nested class does NOT get an implicit reference to an Outer instance.
        // (Unlike a Java INNER (non-static) class. C# has no non-static inner classes.)
        public int Read(Outer o) => o._secret; // can access Outer's private members if given an instance
    }
}

var n = new Outer.Nested();                    // qualify with the outer type name
```

> C# has **no** non-static inner classes (Java's `inner` that captures the enclosing instance). All C# nested classes behave like Java `static` nested classes.

---

## 13. `object` Members: `ToString`, `Equals`, `GetHashCode`

**Think of it like...** `java.lang.Object`'s `toString()`, `equals()`, `hashCode()` — same trio, PascalCase names, same contract (override `Equals` and `GetHashCode` together).

```csharp
public class Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) { X = x; Y = y; }

    // Override ToString — like Java's toString().
    public override string ToString() => $"({X}, {Y})";

    // Override Equals — value equality. Note the 'object?' parameter (nullable).
    public override bool Equals(object? obj)
        => obj is Point p && p.X == X && p.Y == Y;   // pattern match + compare (see §14)

    // MUST override GetHashCode whenever you override Equals (same rule as Java).
    public override int GetHashCode()
        => HashCode.Combine(X, Y);                   // helper — like Objects.hash(x, y) in Java
}
```

Extras vs Java:
- `object.ReferenceEquals(a, b)` = explicit identity check (Java's `==` on references).
- `==` is **overloadable** in C#. For `string` and records it already does value equality. For your own classes, `==` is reference equality unless you overload it.
- Tip: if you just want value equality, use a `record` and get `Equals`/`GetHashCode`/`ToString`/`==` for free.

---

## 14. `is` / `as`, Pattern Matching, Boxing Pitfalls

**Think of it like...** Java's `instanceof` and casts, but with modern pattern matching that also **binds a variable** in one step.

```csharp
object o = "hello";

// 'is' with a type pattern — checks AND binds 'str' in one go (like Java 16+ pattern instanceof).
if (o is string str)                           // true; 'str' is now a string
    Console.WriteLine(str.Length);             // 5

// 'as' — cast that returns null instead of throwing on failure (Java has no direct equivalent).
string? maybe = o as string;                   // non-null here
int? notInt = o as int?;                       // 'as' only works with reference/nullable types

// Direct cast — throws InvalidCastException on failure (like Java's ClassCastException).
string forced = (string)o;
```

### Switch / pattern matching

```csharp
string Describe(object x) => x switch          // switch EXPRESSION (returns a value)
{
    null            => "nothing",              // null pattern
    int n when n < 0 => "negative int",         // 'when' guard (like Java's switch guards)
    int             => "some int",             // type pattern
    string s        => $"string of length {s.Length}",
    Point(var px, _) => $"point at x={px}",     // positional/deconstruction pattern (records/Deconstruct)
    _               => "unknown"               // '_' = default (discard)
};
```

### Boxing pitfalls (value types ↔ `object`)

**Think of it like...** Java auto-boxing `int` → `Integer`, but here it can hit any `struct` and silently allocate on the heap.

```csharp
int i = 42;
object boxed = i;                              // BOXING: copies the int onto the heap, wrapped as object
int unboxed = (int)boxed;                       // UNBOXING: copies it back; wrong cast type throws

// Pitfall 1: boxing in loops/collections hurts performance.
//   Prefer List<int> (no boxing) over ArrayList-style 'List<object>'.

// Pitfall 2: a mutable struct boxed becomes a COPY — mutating the original doesn't affect the box.
struct Counter { public int N; }
Counter c = new Counter { N = 1 };
object box = c;                                // boxes a COPY
c.N = 99;                                       // changes the original only
Console.WriteLine(((Counter)box).N);           // 1 — the boxed copy is untouched

// Pitfall 3: calling an interface method on a struct may box it.
//   Use generic constraints (where T : IComparable<T>) to avoid boxing in hot paths.
```

---

## 15. Common Interview Questions

### Q: In C#, are methods virtual by default? How does this differ from Java?
**No** — C# methods are **non-virtual by default**; you must mark them `virtual` to allow overriding and use `override` in the subclass. Java methods are virtual by default. If you redeclare a non-virtual method in a subclass with `new`, you get *method hiding*: the call resolves by the *variable's* compile-time type, not the object's runtime type. This is the single most common Java→C# gotcha.

### Q: What's the difference between `override` and `new` (method hiding)?
`override` provides true polymorphism — the runtime object's type decides which method runs, even through a base-class reference. `new` *hides* the base method — the variable's declared type decides, so `Base b = new Derived(); b.Foo();` runs `Base.Foo`. `override` requires the base method to be `virtual`/`abstract`; `new` is for non-virtual methods (and only suppresses a compiler warning).

### Q: `struct` vs `class` — what are the key differences and when do you choose a struct?
`struct` is a **value type** (copied by value, lives inline/stack, can't be `null`, no inheritance, default value-equality). `class` is a **reference type** (copied by reference, heap-allocated, nullable, supports inheritance, default reference-equality). Use a `struct` for small (~16 bytes), immutable, single-value data like coordinates or money. Use a `class` for everything else, especially large or mutable objects, since copying big structs is expensive.

### Q: What is boxing/unboxing and why is it a pitfall?
Boxing wraps a value type in a heap `object`; unboxing extracts it back. It silently allocates memory (GC pressure) and, because boxing copies the value, mutating the original doesn't change the box (and vice versa) — a source of subtle bugs with mutable structs. Avoid it by using generics (`List<int>` not `List<object>`) and generic constraints.

### Q: Do C# generics use type erasure like Java?
No. C# generics are **reified** — type arguments are preserved at runtime. You can do `typeof(T)`, `new T()` (with a `new()` constraint), `x is List<int>`, and `List<int>` stores raw ints without boxing. Java erases generics to raw types, so none of that is possible and `List<Integer>` boxes every element.

### Q: What is `internal` and what's its Java equivalent?
`internal` limits visibility to the current **assembly** (compiled DLL). It's the closest thing to Java's package-private default, but the scope is the whole assembly, not a folder/package. Library authors use it to hide helpers from consumers. `protected internal` = assembly **OR** subclass; `private protected` = assembly **AND** subclass.

### Q: How do C# records differ from Java records?
Both generate value equality. But C# records also: support `==` value equality (Java `==` stays reference equality), provide the `with` expression for non-destructive copies, allow inheritance, allow mutable members, and offer a value-type variant (`record struct`). C# records are reference types by default (`record class`).

### Q: What is explicit interface implementation and why use it?
Implementing a member as `string IFoo.Bar()` (qualified, no access modifier) ties it to a specific interface. It's used to (a) resolve clashes when two interfaces declare the same member differently, or (b) keep an interface method off the class's public surface so it's only callable through the interface reference.

### Q: Explain covariance (`out`) and contravariance (`in`).
`out T` (covariant) means `T` only flows *out*, so `IEnumerable<Dog>` can be used as `IEnumerable<Animal>` (like Java `? extends`). `in T` (contravariant) means `T` only flows *in*, so `IComparer<Animal>` can be used as `IComparer<Dog>` (like Java `? super`). C# declares variance once on the interface/delegate (declaration-site), unlike Java's use-site wildcards.

### Q: How do C# enums compare to Java enums?
C# enums are basically named integer constants with an optional underlying type (`: byte`, `: long`, etc.) and `[Flags]` support for bitwise combinations. They **cannot** hold fields, constructors, or instance methods like Java enums can. To add behavior, use **extension methods**. For truly rich "enum" types, use a class or the smart-enum pattern.

### Q: What's the difference between `is` and `as`?
`is` tests type (and can bind a variable via pattern matching), returning a `bool`. `as` performs a cast that returns `null` on failure instead of throwing (only for reference/nullable types). A direct `(T)x` cast throws `InvalidCastException` on failure. Use `is`/pattern matching for branching, `as` when a failed cast is acceptable.

### Q: When must you override `GetHashCode`, and how?
Whenever you override `Equals` (same contract as Java: equal objects must have equal hash codes). Use `HashCode.Combine(field1, field2, ...)` — the C# analog of `Objects.hash(...)`. Failing to do so breaks `Dictionary`/`HashSet` lookups. Records do this automatically.

### Q: What is a static constructor and when does it run?
A parameterless, modifier-less constructor (`static MyClass() { }`) that the runtime invokes **once, automatically, before the first access** to the type's members. It's the equivalent of Java's `static { }` initializer block and is used to initialize static state safely.

### Q: Can a `struct` implement an interface or inherit?
A `struct` can implement interfaces but **cannot** inherit from a class or be inherited (it implicitly derives from `System.ValueType`). Calling an interface method on a struct value can box it, so prefer generic constraints (`where T : IMyInterface`) in performance-sensitive code.

### Q: What's the difference between `readonly struct` and a regular `struct`?
A `readonly struct` guarantees immutability — all instance fields must be read-only and the compiler forbids mutation. This avoids defensive copies the compiler would otherwise make and clearly signals value semantics. A regular `struct` can have mutable fields, which is a common source of bugs.

---

## 16. Quick Reference Cheat Sheet

```
INHERITANCE / POLYMORPHISM
  class Dog : Animal            // extends
  class Dog : Animal, IBark     // extends + implements (base class first)
  base(args) / base.Method()    // super(...) / super.method()
  sealed class                  // final class (no subclassing)
  virtual / override            // OPT-IN polymorphism (Java is automatic!)
  new                           // method HIDING (resolves by variable type)
  abstract class / method       // same as Java

ACCESS MODIFIERS
  public            everyone
  private           same class (default for members)
  protected         class + subclasses
  internal          same assembly/DLL  (~package-private, default for top-level types)
  protected internal  assembly OR subclass
  private protected   assembly AND subclass
  file              same source file (C# 11+)

VALUE vs REFERENCE
  class   -> reference type, heap, copy reference, nullable, inheritance
  struct  -> value type, inline, copy value, non-null, NO inheritance
  readonly struct  -> immutable value type
  record struct    -> value type + value equality + 'with'
  Nullable: T?  (e.g., int?, Point?)
  Boxing: object o = 42;  // value -> heap (avoid in hot paths)

RECORDS
  record Person(string Name, int Age);   // value equality + ToString + ==
  var p2 = p with { Age = 30 };           // non-destructive copy
  var (n, a) = p;                          // deconstruction

GENERICS (REIFIED — no erasure)
  class Box<T> { T Get(); }
  T Max<T>(T a, T b) where T : IComparable<T>
  Constraints: where T : class | struct | new() | BaseType | IFace | notnull
  Variance: interface IOut<out T> {}  // covariant (? extends)
            interface IIn<in T> {}     // contravariant (? super)
  typeof(T), new T(), x is List<int>   // all legal (impossible in Java)

ENUMS
  enum Status { Active, Closed }            // int constants
  enum Code : short { Ok = 200 }            // custom backing type
  [Flags] enum Perm { Read=1, Write=2 }     // bit flags; HasFlag(...)
  Add behavior via extension methods (this Status s)

INTERFACES
  interface IRepo<T> { T Get(int id); int Count => 0; }  // default method
  string IFoo.Bar() => ...;                  // explicit implementation

OBJECT MEMBERS
  override ToString()        // toString()
  override Equals(object?)   // equals()
  override GetHashCode()     // hashCode() -> HashCode.Combine(...)
  object.ReferenceEquals(a,b)  // identity check

CASTING / PATTERNS
  if (o is string s) ...           // instanceof + bind
  var x = o as string;             // null on failure
  var y = (string)o;               // throws on failure
  o switch { int n when n>0 => .., string s => .., _ => .. }

STATIC / PARTIAL / NESTED
  static class Utils { }           // only static members
  static MyType() { }              // static constructor (Java static {} block)
  partial class X { }              // split across files
  Outer.Nested                     // nested = Java static nested class (no inner classes)

REMEMBER (Java vs C#)
  * Methods NON-virtual by default in C#  (Java: virtual by default)
  * Generics REIFIED in C#               (Java: erased)
  * struct = real value type in C#       (Java: only primitives)
  * '==' does value equality for string & records (Java: reference)
```

---

*Last Updated: 2026-06-16*
