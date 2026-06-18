# Java Enums Interview Questions & Study Guide

## Overview

An **enum** (enumeration) is a special Java type representing a **fixed, compile-time set of constants** — like days of the week, order statuses, or payment methods. Enums are a backend interview favorite because they touch type safety, the singleton pattern, JPA persistence (`@Enumerated`), and JSON serialization.

---

## Table of Contents

1. [What Is an Enum & Why Use One](#what-is-an-enum--why-use-one)
2. [Basic Declaration & Usage](#basic-declaration--usage)
3. [Switch Over Enums](#switch-over-enums)
4. [Enums with Fields, Constructors & Methods](#enums-with-fields-constructors--methods)
5. [Constant-Specific Method Bodies](#constant-specific-method-bodies)
6. [Built-in Methods: values, valueOf, name, ordinal](#built-in-methods-values-valueof-name-ordinal)
7. [EnumSet & EnumMap](#enumset--enummap)
8. [Enum Implementing an Interface](#enum-implementing-an-interface)
9. [Enum as the Best Singleton](#enum-as-the-best-singleton)
10. [Persisting Enums — JPA @Enumerated](#persisting-enums--jpa-enumerated)
11. [Enums and JSON (Jackson)](#enums-and-json-jackson)
12. [Common Mistakes & Pitfalls](#common-mistakes--pitfalls)
13. [Common Interview Questions](#common-interview-questions)
14. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What Is an Enum & Why Use One

An **enum** is a type whose value can only be one of a **fixed set of named constants**. Once you declare `enum Day { MONDAY, TUESDAY, ... }`, a variable of type `Day` can only hold one of those values — nothing else compiles.

### The problem enums solve

```java
// OLD WAY — int constants: no type safety
public static final int STATUS_ACTIVE = 0;
void setStatus(int status) { ... }
setStatus(99);  // COMPILES — but 99 isn't valid!

// ENUM WAY — compiler enforces valid values
public enum Status { ACTIVE, INACTIVE, BANNED }
void setStatus(Status status) { ... }
setStatus(Status.ACTIVE);  // safe and readable
setStatus(99);             // COMPILE ERROR
```

| Aspect | `int`/`String` constants | `enum` |
|---|---|---|
| **Type safety** | None | Compiler rejects invalid values |
| **Readability** | `0` means nothing | `Status.ACTIVE` is self-documenting |
| **Invalid values** | Possible | Impossible — won't compile |
| **Can add behavior** | No | Yes — fields, methods, logic |
| **`EnumSet`/`EnumMap`** | No | Yes — very fast |

> **Interview soundbite:** "Enums give you **type safety** and **readability** that `int` or `String` constants can't. The compiler guarantees only valid values, and you can attach data and behavior to each constant."

---

## Basic Declaration & Usage

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}
```

```java
Day today = Day.WEDNESDAY;
System.out.println(today);          // "WEDNESDAY"
if (today == Day.WEDNESDAY) { ... } // use == to compare enums (each constant is a singleton)

String label = today.name();        // "WEDNESDAY"
int position  = today.ordinal();    // 2 (zero-based)
```

**Key facts:**
- Each constant is a **single shared instance** — that's why `==` works for comparison.
- Enums implicitly extend `java.lang.Enum` — they **cannot extend another class**, but *can* implement interfaces.
- Enums are implicitly `final` — you cannot subclass them.

> **Why `==` instead of `.equals()`?** `==` is null-safe (`null == Day.MONDAY` is just `false`, never an NPE) and reads more clearly.

---

## Switch Over Enums

```java
// Classic switch — no enum prefix inside case
public String plan(Day day) {
    switch (day) {
        case MONDAY:          return "Team standup";
        case FRIDAY:          return "Demo day";
        case SATURDAY:
        case SUNDAY:          return "Rest";
        default:              return "Regular work";
    }
}

// Modern switch expression (Java 14+) — no fall-through, returns a value
public String plan(Day day) {
    return switch (day) {
        case MONDAY           -> "Team standup";
        case FRIDAY           -> "Demo day";
        case SATURDAY, SUNDAY -> "Rest";
        default               -> "Regular work";
    };
}
```

> **Interview Tip:** With a switch expression covering **every** constant you can drop `default`, and the compiler forces you to handle any new constants you add later — catching the classic "I added an enum value and forgot to update the switch" bug at compile time.

---

## Enums with Fields, Constructors & Methods

Enums can carry **data** (fields) and **behavior** (methods). Each constant is created by calling a constructor with arguments.

```java
public enum Currency {
    USD("$", "US Dollar"),
    EUR("€", "Euro"),
    INR("₹", "Indian Rupee");   // semicolon required when fields/methods follow

    private final String symbol;
    private final String displayName;

    Currency(String symbol, String displayName) {  // implicitly private
        this.symbol = symbol;
        this.displayName = displayName;
    }

    public String getSymbol()      { return symbol; }
    public String getDisplayName() { return displayName; }
}

// Usage
String s = Currency.INR.getSymbol();   // "₹"
```

> **Key rule:** The semicolon after the last constant is **required** when you add any fields, constructors, or methods. The constructor is always effectively `private` — you can never write `new Currency(...)`.

---

## Constant-Specific Method Bodies

Each constant can provide its own implementation of an **abstract method** — this is the strategy pattern built into the language.

```java
public enum Operation {
    PLUS  { public double apply(double x, double y) { return x + y; } },
    MINUS { public double apply(double x, double y) { return x - y; } },
    TIMES { public double apply(double x, double y) { return x * y; } },
    DIVIDE{ public double apply(double x, double y) { return x / y; } };

    public abstract double apply(double x, double y);
}

double result = Operation.PLUS.apply(3, 4);  // 7.0
```

**Why better than a `switch`?** Adding a new constant forces you to write its `apply()` — code won't compile otherwise. The compiler keeps you honest.

> **Interview Tip:** This is the **strategy pattern** built into the language. Mentioning "Effective Java Item 34 — use constant-specific methods instead of switching on enum values" scores points.

---

## Built-in Methods: values, valueOf, name, ordinal

| Method | Returns | Notes |
|---|---|---|
| `values()` | Array of all constants, declaration order | Static. Great for looping. |
| `valueOf(String)` | Constant matching the name | Static. **Throws** `IllegalArgumentException` if no match — case-sensitive! |
| `name()` | Exact declared name as `String` | `final` — cannot be overridden. |
| `ordinal()` | Zero-based position in declaration | **Avoid for logic/storage** (see warning). |

```java
Day d = Day.valueOf("FRIDAY");       // Day.FRIDAY
// Day bad = Day.valueOf("friday");  // throws IllegalArgumentException

// Safe parsing pattern
public static Optional<Day> parse(String input) {
    try {
        return Optional.of(Day.valueOf(input));
    } catch (IllegalArgumentException e) {
        return Optional.empty();
    }
}
```

### The `ordinal()` warning

`ordinal()` changes if you **reorder or insert constants** — any stored integers now map to wrong constants. Use an explicit `int` field instead:

```java
// GOOD — explicit, stable, won't break on reordering
public enum Priority {
    LOW(1), MEDIUM(5), HIGH(10);
    private final int weight;
    Priority(int weight) { this.weight = weight; }
    public int getWeight() { return weight; }
}
```

> **Rule:** Never use `ordinal()` for business logic or storage. Effective Java Item 35: *"Use instance fields instead of ordinals."*

---

## EnumSet & EnumMap

When your `Set`/`Map` keys are enum constants, use these **specialized, ultra-fast** collections over `HashSet`/`HashMap`.

```java
// EnumSet — backed by a bit vector
EnumSet<Day> weekend  = EnumSet.of(Day.SATURDAY, Day.SUNDAY);
EnumSet<Day> workdays = EnumSet.range(Day.MONDAY, Day.FRIDAY);
EnumSet<Day> all      = EnumSet.allOf(Day.class);
boolean isRest = weekend.contains(Day.SUNDAY);  // very fast bit check

// EnumMap — array indexed by ordinal
EnumMap<Day, String> tasks = new EnumMap<>(Day.class);
tasks.put(Day.MONDAY, "Standup");
tasks.put(Day.FRIDAY, "Demo");
// Iteration order = declaration order (predictable)
```

| | `EnumSet`/`EnumMap` | `HashSet`/`HashMap` |
|---|---|---|
| **Internal storage** | Bit vector / array by `ordinal()` | Hash buckets |
| **Speed** | Extremely fast, no hashing | Slower |
| **Memory** | Compact | Heavier |
| **Iteration order** | Declaration order | Unspecified |

> **Interview Tip:** "For enum keys, use `EnumMap`; for sets of enums, use `EnumSet`. They're far faster and lighter than hash-based collections." (Effective Java Items 36 & 37.)

---

## Enum Implementing an Interface

Enums can't extend a class but **can implement interfaces**, enabling enum-based strategies behind a common contract.

```java
public interface Describable {
    String describe();
}

public enum LogLevel implements Describable {
    INFO  { public String describe() { return "Informational message"; } },
    WARN  { public String describe() { return "Something looks off"; } },
    ERROR { public String describe() { return "A failure occurred"; } };
}

Describable d = LogLevel.WARN;
System.out.println(d.describe());  // "Something looks off"
```

> **Why it matters:** Lets you pass enum constants where any `Describable` is expected — useful for plugging enum-based strategies into generic code.

---

## Enum as the Best Singleton

A **singleton** has exactly one instance for the whole application. Effective Java Item 3 says: **a single-element enum is the best way to implement a singleton.**

```java
public enum Registry {
    INSTANCE;

    private final Map<String, String> data = new ConcurrentHashMap<>();

    public void put(String key, String value) { data.put(key, value); }
    public String get(String key)             { return data.get(key); }
}

// Usage
Registry.INSTANCE.put("env", "prod");
String env = Registry.INSTANCE.get("env");
```

### Why the enum singleton wins

| Concern | Classic singleton | Enum singleton |
|---|---|---|
| **Thread safety** | Must add `synchronized` | Guaranteed by the JVM |
| **Reflection attack** | Can be broken (`setAccessible`) | Impossible — JVM forbids it |
| **Serialization** | Needs `readResolve()` | Automatic — returns same `INSTANCE` |
| **Boilerplate** | Lots | One word: `INSTANCE` |

> **Interview soundbite:** "The single-element enum is the best singleton (Effective Java Item 3): **thread-safe**, **serialization-safe**, and **reflection-proof** — all for free."

---

## Persisting Enums — JPA @Enumerated

When storing an enum field in a database via JPA/Hibernate, you choose storage mode with `@Enumerated`. Always use `STRING`.

```java
@Entity
public class Order {
    @Enumerated(EnumType.STRING)   // stores "PENDING", "SHIPPED" — readable and stable
    private OrderStatus status;
}

public enum OrderStatus { PENDING, PAID, SHIPPED, DELIVERED, CANCELLED }
```

| | `EnumType.ORDINAL` | `EnumType.STRING` |
|---|---|---|
| **What's stored** | `0`, `1`, `2`... | `"PENDING"`, `"SHIPPED"`... |
| **DB column** | `INT` | `VARCHAR` |
| **Readable in DB?** | No | Yes |
| **Safe to reorder/insert?** | **NO — silent data corruption** | **Yes** |
| **Default (no annotation)** | **Yes (dangerous!)** | — |

```java
// ORDINAL danger: insert CONFIRMED in the middle
public enum OrderStatus { PENDING, CONFIRMED, PAID, SHIPPED }
// Now PAID=2, SHIPPED=3 — every DB row that stored "1" (PAID) now reads as CONFIRMED!
```

> **Interview Tip:** "Always use `@Enumerated(EnumType.STRING)`. The default is `ORDINAL` — reordering or inserting a constant silently corrupts existing rows. The only cost of `STRING` is that renaming a constant requires a DB migration."

---

## Enums and JSON (Jackson)

By default, Jackson uses `name()` for serialization (case-sensitive):

```java
// Role.ADMIN serializes to "ADMIN"; "USER" deserializes to Role.USER
```

### Customizing

```java
public enum Role {
    ADMIN("administrator"), USER("standard-user"), GUEST("guest");

    private final String json;
    Role(String json) { this.json = json; }

    @JsonValue                               // serialize with this value instead of name()
    public String getJson() { return json; }

    @JsonCreator                             // custom deserialization
    public static Role from(String value) {
        return Arrays.stream(values())
                     .filter(r -> r.json.equalsIgnoreCase(value))
                     .findFirst()
                     .orElse(GUEST);         // unknown input → default instead of 400
    }
}
```

> **Interview Tip:** "Use `@JsonValue` to control the serialized form and `@JsonCreator` for custom, case-insensitive, or fault-tolerant deserialization."

---

## Common Mistakes & Pitfalls

```java
// PITFALL 1: ordinal() for logic/storage — breaks on reorder. Use an explicit field.
int code = status.ordinal();

// PITFALL 2: Forgetting @Enumerated → silently defaults to ORDINAL
@Enumerated                       // WRONG — defaults to ORDINAL
private OrderStatus status;
@Enumerated(EnumType.STRING)      // CORRECT
private OrderStatus status;

// PITFALL 3: Inserting a constant in the middle with ORDINAL storage → data corruption
// Always APPEND new constants at the END.

// PITFALL 4: valueOf() with wrong case → IllegalArgumentException
Role r = Role.valueOf("admin");   // throws! Correct: "ADMIN". Parse defensively.

// PITFALL 5: HashMap/HashSet for enum keys
Map<Day, Task> m = new HashMap<>();  // prefer new EnumMap<>(Day.class)

// PITFALL 6: .equals() instead of == (NPE risk)
if (status.equals(OrderStatus.PAID)) {}  // NPE if status is null
if (status == OrderStatus.PAID) {}       // null-safe — prefer this
```

**Golden rules:**
1. **Append new constants at the end** (especially with ordinal-based storage).
2. **Never use `ordinal()`** for anything stable — add an explicit field.
3. **Persist with `EnumType.STRING`**, never the default `ORDINAL`.
4. **Use `EnumSet`/`EnumMap`** for enum collections.
5. **Compare with `==`**, not `.equals()`.

---

## Common Interview Questions

### Q: What is an enum and why use it over `int`/`String` constants?

An enum is a type with a fixed set of named constant values. It gives **type safety** (the compiler rejects invalid values), **readability** (`Status.ACTIVE` vs `0`), and the ability to attach fields and behavior. `int`/`String` constants accept any value, so typos and invalid numbers compile fine and fail at runtime.

---

### Q: Can an enum have a constructor? Can you call `new` on it?

Yes, enums can have constructors to initialize per-constant fields. No, you can never call `new` — the constructor is implicitly `private` and the JVM calls it once per constant at class load time.

---

### Q: Can an enum extend a class? Implement an interface?

It **cannot extend** a class — every enum already implicitly extends `java.lang.Enum` (single inheritance). It **can implement** any number of interfaces, and each constant can provide its own method body.

---

### Q: What's the difference between `name()` and `ordinal()`? Why avoid `ordinal()`?

`name()` returns the declared constant name as a stable `String`. `ordinal()` returns the zero-based declaration position. Avoid `ordinal()` because reordering or inserting constants shifts positions, silently corrupting stored or transmitted values. Use an explicit instance field for any stable number.

---

### Q: What are constant-specific method bodies?

You declare an `abstract` method on the enum and each constant supplies its own implementation — the strategy pattern built into the language. It replaces a `switch` on the enum, and the compiler forces every new constant to implement the method (Effective Java Item 34).

---

### Q: Why are `EnumSet` and `EnumMap` preferred over `HashSet`/`HashMap`?

Because the set of enum constants is fixed, `EnumSet` is a **bit vector** and `EnumMap` is an **array indexed by `ordinal()`** — no hashing, compact memory, predictable iteration order. Significantly faster and lighter than hash-based collections for enum keys.

---

### Q: Why is a single-element enum the best way to write a singleton?

It's **thread-safe** (JVM creates the constant once, safely), **serialization-safe** (deserialization returns the same instance automatically), and **immune to reflection attacks** (JVM forbids reflectively constructing enums). A classic singleton needs extra code for all three (Effective Java Item 3).

---

### Q: How should you persist an enum in a database with JPA?

Use `@Enumerated(EnumType.STRING)` — stores the constant's name, readable in the DB and stable across reordering/insertion. The default `EnumType.ORDINAL` stores positional integers; reordering or inserting a constant silently corrupts existing rows.

---

### Q: Can you add fields and methods to an enum?

Yes. Each constant calls a constructor with arguments to initialize `final` fields, and the enum can have regular methods. Example: a `Currency` enum with `symbol`/`displayName` fields and getters, or a `Planet` enum with `mass`/`radius` and a `surfaceGravity()` method.

---

### Q: How does Jackson serialize enums, and how do you customize it?

By default Jackson uses `name()` for serialization and matches by name (case-sensitive) for deserialization. Use `@JsonValue` on a method to control the serialized form, and `@JsonCreator` for custom or fault-tolerant deserialization.

---

### Q: Should you compare enums with `==` or `.equals()`?

Use `==`. Each enum constant is a singleton, so reference equality and `.equals()` give the same result, but `==` is null-safe (`null == X` is `false`) and reads more clearly.

---

### Q: Are enum constants thread-safe?

Yes. Constants are created once at class-load time by the JVM in a thread-safe manner and are effectively immutable when their fields are `final`. This is exactly why the enum singleton is thread-safe for free.

---

## Quick Reference Cheat Sheet

```
WHAT IS AN ENUM
  Fixed set of named constants → type-safe, readable
  Beats int/String constants: compiler rejects invalid values
  Each constant = one shared singleton instance → compare with ==

DECLARATION
  enum Day { MONDAY, TUESDAY, ... }          // simple
  enum Currency { USD("$"), EUR("€"); ... }  // with fields → SEMICOLON required
  Constructor is implicitly private; runs once per constant
  Cannot extend a class; CAN implement interfaces; implicitly final

BUILT-IN METHODS
  values()       → array of all constants (declaration order)
  valueOf("X")   → constant by name (throws IllegalArgumentException if no match)
  name()         → exact declared name as String (stable)
  ordinal()      → zero-based position (UNSTABLE — avoid for logic/storage)

CONSTANT-SPECIFIC BODIES (strategy pattern)
  enum Op { PLUS { double apply(...){...} }, ...; abstract double apply(...); }
  Compiler forces every new constant to implement → safer than switch

COLLECTIONS (always prefer for enum keys)
  EnumSet  → bit-vector backed, super fast (of/range/allOf/noneOf/complementOf)
  EnumMap  → array indexed by ordinal; new EnumMap<>(Day.class)
  Faster + lighter + ordered vs HashSet/HashMap

SINGLETON (Effective Java Item 3 — the best singleton)
  enum Registry { INSTANCE; ... }
  Thread-safe ✓  Serialization-safe ✓  Reflection-proof ✓  (all FOR FREE)

JPA PERSISTENCE (backend must-know)
  @Enumerated(EnumType.STRING)   → stores "PENDING"  → SAFE, readable, stable  ✓
  @Enumerated(EnumType.ORDINAL)  → stores 0,1,2      → breaks on reorder       ✗
  Default (no annotation) = ORDINAL → ALWAYS specify STRING

JSON (Jackson)
  Default      → serializes via name()
  @JsonValue   → custom serialized form
  @JsonCreator → custom / case-insensitive / fault-tolerant deserialization

PITFALLS
  ✗ ordinal() for logic or storage      ✓ explicit int field
  ✗ insert constant in the middle       ✓ append at the end
  ✗ forget @Enumerated (→ ORDINAL)      ✓ @Enumerated(EnumType.STRING)
  ✗ HashMap for enum keys               ✓ EnumMap
  ✗ status.equals(X) (NPE risk)         ✓ status == X (null-safe)

EFFECTIVE JAVA ITEMS TO NAME-DROP
  Item 3  → enum singleton
  Item 34 → enums instead of int constants
  Item 35 → instance fields instead of ordinals
  Item 36 → EnumSet instead of bit fields
  Item 37 → EnumMap instead of ordinal indexing
```

---

*Last Updated: 2026-06-18*
