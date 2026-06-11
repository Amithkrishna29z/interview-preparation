# Java Enums Interview Questions & Study Guide

## Overview

An **enum** (short for "enumeration") is a special Java type that represents a **fixed, known-at-compile-time set of constants** — like days of the week, order statuses, or payment methods. Enums are a favorite backend interview topic because they touch type safety, the singleton pattern, JPA persistence (`@Enumerated`), and JSON serialization. This guide takes you from "what is an enum" all the way to "why is an enum the best singleton in Java" — at a junior-friendly pace.

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

An **enum** is a type whose value can only be one of a **fixed set of named constants**. Once you declare `enum Day { MONDAY, TUESDAY, ... }`, a variable of type `Day` can ONLY hold one of those values — nothing else compiles.

**Think of it like a multiple-choice question:** With a free-text answer box (a `String`), someone could type "Mondey", "monday", "MON", or "🍕". With multiple-choice radio buttons (an enum), they can only pick from the options you defined. The compiler enforces the options for you.

### The problem enums solve

Before enums, developers used `int` or `String` constants:

```java
// THE OLD, BAD WAY — "int constants" anti-pattern
public static final int STATUS_ACTIVE   = 0;   // just a number, no type safety
public static final int STATUS_INACTIVE = 1;
public static final int STATUS_BANNED   = 2;

void setStatus(int status) { ... }   // accepts ANY int

setStatus(0);       // ok... but what does 0 mean? Unreadable.
setStatus(99);      // COMPILES FINE — but 99 is not a valid status! Bug waiting to happen.
setStatus(STATUS_ACTIVE + STATUS_BANNED);  // nonsense math compiles — 0 + 2 = 2 = BANNED?!
```

```java
// SLIGHTLY-LESS-BAD WAY — "String constants"
public static final String ACTIVE = "ACTIVE";

void setStatus(String status) { ... }

setStatus("ACTIVE");   // ok
setStatus("active");   // wrong case — silently broken
setStatus("Actve");    // typo — compiles fine, breaks at runtime
```

### The enum way

```java
public enum Status { ACTIVE, INACTIVE, BANNED }   // a real type with exactly 3 valid values

void setStatus(Status status) { ... }   // accepts ONLY a Status

setStatus(Status.ACTIVE);   // readable and safe
setStatus(99);              // COMPILE ERROR — not a Status
setStatus("active");        // COMPILE ERROR — not a Status
```

| Aspect | `int` / `String` constants | `enum` |
|---|---|---|
| **Type safety** | None — any int/string accepted | Compiler rejects invalid values |
| **Readability** | `0` means nothing | `Status.ACTIVE` is self-documenting |
| **Invalid values** | Possible (`99`, `"actve"`) | Impossible — won't compile |
| **Namespace** | Can clash with other constants | Scoped under the enum name |
| **Can add behavior** | No | Yes — fields, methods, logic |
| **Use in `switch`** | Yes | Yes (and compiler can warn on missing cases) |
| **`EnumSet`/`EnumMap`** | No | Yes — very fast |

> **Interview soundbite:** "Enums give you **type safety** and **readability** that `int` or `String` constants can't. The compiler guarantees a variable holds only a valid value, and you can attach data and behavior to each constant."

---

## Basic Declaration & Usage

```java
// The simplest enum — just a list of constants
public enum Day {
    MONDAY,      // each name is a constant; by convention written in UPPER_CASE
    TUESDAY,
    WEDNESDAY,
    THURSDAY,
    FRIDAY,
    SATURDAY,
    SUNDAY       // no semicolon needed when the enum has ONLY constants
}
```

Using it:

```java
Day today = Day.WEDNESDAY;          // declare a variable of the enum type

System.out.println(today);          // prints "WEDNESDAY" (enum's default toString = its name)

if (today == Day.WEDNESDAY) {       // use == to compare enums (safe — each constant is a singleton)
    System.out.println("Midweek!");
}

// Enums are objects, so you can call methods on them
String label = today.name();        // "WEDNESDAY"
int position  = today.ordinal();    // 2 (zero-based position in the declaration)
```

**Key facts about every enum:**

- Each constant (`MONDAY`, etc.) is a **single, shared instance** — there is exactly one `Day.MONDAY` object in the whole JVM. That's why `==` works for comparison.
- Enums implicitly extend `java.lang.Enum`, so they **cannot extend another class** (Java has single inheritance). They *can* implement interfaces (covered later).
- Enums are implicitly `final` — you cannot subclass them.

> **Why `==` instead of `.equals()`?** Because each enum constant is a singleton, `==` (reference equality) and `.equals()` give the same result. `==` is preferred: it's null-safe (`null == Day.MONDAY` is just `false`, never an NPE) and slightly faster to read.

---

## Switch Over Enums

Enums work beautifully with `switch` — and you don't prefix the constant with the enum name inside the `case`.

```java
public String plan(Day day) {
    switch (day) {                       // switch on the enum value
        case MONDAY:                     // note: just MONDAY, NOT Day.MONDAY
            return "Team standup";
        case FRIDAY:
            return "Demo day";
        case SATURDAY:
        case SUNDAY:                     // fall-through: both weekend days return the same thing
            return "Rest";
        default:
            return "Regular work";       // covers the remaining weekdays
    }
}
```

Modern (Java 14+) **switch expression** — more concise, no fall-through bugs, returns a value:

```java
public String plan(Day day) {
    return switch (day) {                // switch EXPRESSION (note the arrows)
        case MONDAY            -> "Team standup";
        case FRIDAY            -> "Demo day";
        case SATURDAY, SUNDAY  -> "Rest";          // multiple labels, comma-separated
        default                -> "Regular work";
    };                                   // semicolon — it's an expression assigned to return
}
```

> **Interview Tip:** With a switch *expression* over an enum, if you cover **every** constant you can drop `default`, and the compiler will force you to handle new constants you add later. This catches the classic "I added a new enum value and forgot to update the switch" bug at compile time.

---

## Enums with Fields, Constructors & Methods

Enums can carry **data** (fields) and **behavior** (methods). Each constant is created by calling a constructor with arguments.

**Think of it like a vending machine row:** Each slot (constant) holds a specific product with its own price and name baked in. The slot *is* the enum constant; the price/name are its fields.

```java
public enum Planet {
    // Each constant calls the constructor with (mass, radius) arguments
    MERCURY (3.303e+23, 2.4397e6),     // constant + constructor args
    EARTH   (5.976e+24, 6.37814e6),
    JUPITER (1.9e+27,   7.1492e7);     // SEMICOLON required here — fields/methods follow

    private final double mass;          // in kilograms — final because constants never change
    private final double radius;        // in meters

    // The constructor is implicitly private (you can't call `new Planet(...)`)
    Planet(double mass, double radius) {
        this.mass = mass;               // store the value passed for this constant
        this.radius = radius;
    }

    // A regular method — every constant inherits it and uses ITS OWN fields
    public double surfaceGravity() {
        final double G = 6.67300E-11;   // gravitational constant
        return G * mass / (radius * radius);
    }

    // Compute the weight of an object on this planet
    public double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}
```

Using it:

```java
double earthWeight = 75.0;
double mass = earthWeight / Planet.EARTH.surfaceGravity();   // back out the object's mass

for (Planet p : Planet.values()) {                           // loop every constant
    System.out.printf("Weight on %s: %.2f%n", p, p.surfaceWeight(mass));
}
// Each planet uses its OWN mass/radius to compute a different result
```

A very common backend example — a `Currency` enum carrying a symbol and code:

```java
public enum Currency {
    USD("$", "US Dollar"),              // symbol, display name
    EUR("€", "Euro"),
    INR("₹", "Indian Rupee"),
    GBP("£", "British Pound");          // semicolon ends the constant list

    private final String symbol;
    private final String displayName;

    Currency(String symbol, String displayName) {  // constructor runs once per constant
        this.symbol = symbol;
        this.displayName = displayName;
    }

    public String getSymbol()      { return symbol; }       // getter for the symbol
    public String getDisplayName() { return displayName; }
}

// Usage
String s = Currency.INR.getSymbol();   // "₹"
```

> **Key rule:** The semicolon after the last constant is **required** the moment you add any fields, constructors, or methods. The constructor is always effectively `private` — you can never write `new Currency(...)`.

---

## Constant-Specific Method Bodies

Sometimes each constant needs to behave **differently**. You can give the enum an **abstract method** and let each constant provide its own implementation. This is cleaner than a giant `switch` inside one method.

**Think of it like a TV remote's buttons:** Every button "does something when pressed," but Power, Volume-Up, and Mute each do a completely different thing. The "do something" is the abstract method; each button supplies its own body.

```java
public enum Operation {
    PLUS  { public double apply(double x, double y) { return x + y; } },  // PLUS's own body
    MINUS { public double apply(double x, double y) { return x - y; } },  // MINUS's own body
    TIMES { public double apply(double x, double y) { return x * y; } },
    DIVIDE{ public double apply(double x, double y) { return x / y; } };  // semicolon ends list

    // Abstract method — EVERY constant MUST provide its own implementation above
    public abstract double apply(double x, double y);
}
```

Using it:

```java
double result = Operation.PLUS.apply(3, 4);    // 7.0 — calls PLUS's body
double diff   = Operation.MINUS.apply(10, 6);  // 4.0 — calls MINUS's body

for (Operation op : Operation.values()) {
    System.out.printf("6 %s 2 = %.1f%n", op, op.apply(6, 2));
}
// Each constant runs its OWN apply() — no switch statement anywhere
```

**Why is this better than a switch?**

```java
// THE WORSE ALTERNATIVE — switch inside one method
public double apply(double x, double y) {
    switch (this) {
        case PLUS:  return x + y;
        case MINUS: return x - y;
        // If you add a new constant (e.g., MODULO) and FORGET to add a case here,
        // this falls through to a runtime error instead of a compile error.
        default: throw new AssertionError("Unknown op: " + this);
    }
}
```

With constant-specific bodies, **adding a new constant forces you to write its `apply()`** — the code won't compile otherwise. The compiler keeps you honest.

> **Interview Tip:** This is the **strategy pattern** built into the language. Mentioning "Effective Java Item 34 — use constant-specific methods instead of switching on enum values" scores points.

---

## Built-in Methods: values, valueOf, name, ordinal

Every enum automatically gets these methods (you don't write them):

| Method | Returns | Example | Notes |
|---|---|---|---|
| `values()` | array of all constants, in declaration order | `Day.values()` → `[MONDAY, ..., SUNDAY]` | Static. Great for looping. |
| `valueOf(String)` | the constant matching the name | `Day.valueOf("MONDAY")` → `Day.MONDAY` | Static. **Throws** `IllegalArgumentException` if no match — case-sensitive! |
| `name()` | the exact declared name as a `String` | `Day.MONDAY.name()` → `"MONDAY"` | `final` — cannot be overridden. |
| `ordinal()` | zero-based position in declaration | `Day.WEDNESDAY.ordinal()` → `2` | **Avoid using this for logic/storage** (see warning). |

```java
// values() — iterate every constant
for (Day d : Day.values()) {
    System.out.println(d.ordinal() + " = " + d.name());
}
// 0 = MONDAY, 1 = TUESDAY, ...

// valueOf() — convert a String (e.g., from a request or DB) back into an enum
Day d = Day.valueOf("FRIDAY");        // Day.FRIDAY
// Day bad = Day.valueOf("friday");   // throws IllegalArgumentException (wrong case!)
// Day bad = Day.valueOf("Funday");   // throws IllegalArgumentException (no such constant)

// Safe parsing pattern — handle bad input instead of crashing
public static Optional<Day> parse(String input) {
    try {
        return Optional.of(Day.valueOf(input));   // try the conversion
    } catch (IllegalArgumentException e) {
        return Optional.empty();                  // unknown value → empty, no crash
    }
}
```

### The `ordinal()` warning (very important)

`ordinal()` returns the **position** of a constant. The danger: this position **changes if you reorder or insert constants**.

```java
public enum Priority { LOW, MEDIUM, HIGH }
// LOW=0, MEDIUM=1, HIGH=2

// Someone later inserts URGENT in the middle:
public enum Priority { LOW, URGENT, MEDIUM, HIGH }
// LOW=0, URGENT=1, MEDIUM=2 (!!), HIGH=3 (!!)
// Every MEDIUM and HIGH ordinal you ever saved to a DB or sent over the wire is now WRONG.
```

> **Rule:** **Never** use `ordinal()` for business logic, storage, or any value that must stay stable. If you need a stable number, add your own explicit `int` field. Use `ordinal()` only for internal stuff like `EnumMap`/`EnumSet` (which manage it safely for you). Effective Java Item 35: *"Use instance fields instead of ordinals."*

```java
// GOOD — explicit, stable code that won't break on reordering
public enum Priority {
    LOW(1), MEDIUM(5), HIGH(10);
    private final int weight;
    Priority(int weight) { this.weight = weight; }
    public int getWeight() { return weight; }   // use THIS, not ordinal()
}
```

---

## EnumSet & EnumMap

When your `Set` keys or `Map` keys are enum constants, the JDK gives you two **specialized, ultra-fast** collections: `EnumSet` and `EnumMap`. Always prefer them over `HashSet`/`HashMap` for enums.

**Think of it like a row of light switches:** Because there's a fixed, known number of enum constants, the JVM can represent "which ones are present" as **bits in a single number** (a bit vector) instead of hashing objects into buckets. Flipping bits is about as fast as computers get.

### EnumSet

```java
// Build a set of enum constants
EnumSet<Day> weekend  = EnumSet.of(Day.SATURDAY, Day.SUNDAY);   // just these two
EnumSet<Day> workdays = EnumSet.range(Day.MONDAY, Day.FRIDAY);  // a contiguous range
EnumSet<Day> all      = EnumSet.allOf(Day.class);               // every constant
EnumSet<Day> none     = EnumSet.noneOf(Day.class);              // empty, typed set
EnumSet<Day> notWeekend = EnumSet.complementOf(weekend);        // everything EXCEPT weekend

boolean isRest = weekend.contains(Day.SUNDAY);   // true — backed by a bit check, very fast
```

### EnumMap

```java
// A Map whose KEYS are enum constants
EnumMap<Day, String> tasks = new EnumMap<>(Day.class);   // must pass the key's Class
tasks.put(Day.MONDAY, "Standup");                        // store value per enum key
tasks.put(Day.FRIDAY, "Demo");

String monday = tasks.get(Day.MONDAY);   // "Standup"
// Iteration order is the enum's DECLARATION order — predictable, unlike HashMap
```

### Why prefer them over HashSet/HashMap?

| | `EnumSet` / `EnumMap` | `HashSet` / `HashMap` |
|---|---|---|
| **Internal storage** | Bit vector / plain array indexed by `ordinal()` | Hash buckets + linked nodes |
| **Speed** | Extremely fast, no hashing | Slower — hash + equals on every access |
| **Memory** | Compact (often a single `long`) | Heavier (node objects, table) |
| **Iteration order** | Enum declaration order (predictable) | Unspecified |
| **Null keys** | Not allowed | `HashMap` allows one null key |

```java
// Prefer this:
EnumMap<Day, String> good = new EnumMap<>(Day.class);
// Over this:
HashMap<Day, String> meh  = new HashMap<>();   // works, but slower & unordered for enum keys
```

> **Interview Tip:** "For enum keys, use `EnumMap`; for sets of enums, use `EnumSet`. They use the enum's `ordinal()` internally to index an array/bit-vector, so they're far faster and lighter than hash-based collections." (Effective Java Items 36 & 37.)

---

## Enum Implementing an Interface

Enums can't extend a class, but they **can implement interfaces**. This lets different enums share a common contract, or lets you swap enum-based strategies behind an interface type.

```java
// A shared contract
public interface Describable {
    String describe();      // every implementer must provide this
}

// An enum implementing it
public enum LogLevel implements Describable {
    INFO  { public String describe() { return "Informational message"; } },
    WARN  { public String describe() { return "Something looks off"; } },
    ERROR { public String describe() { return "A failure occurred"; } };
    // each constant supplies its own describe() body
}

// Now you can treat enum constants as the interface type
Describable d = LogLevel.WARN;
System.out.println(d.describe());   // "Something looks off"
```

You can also implement the method **once** for all constants (no per-constant body needed):

```java
public enum Operation implements DoubleBinaryOperator {  // a built-in functional interface
    PLUS("+")  { public double applyAsDouble(double x, double y) { return x + y; } },
    TIMES("*") { public double applyAsDouble(double x, double y) { return x * y; } };

    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }

    @Override public String toString() { return symbol; }   // override toString for nice printing
}
```

> **Why it matters:** Implementing an interface lets you pass enum constants where any `Describable`/`Comparator`/`DoubleBinaryOperator` is expected — useful for plugging enum-based strategies into generic code.

---

## Enum as the Best Singleton

A **singleton** is a class with exactly one instance for the whole application (e.g., a config holder, a connection registry). Joshua Bloch's *Effective Java* (Item 3) says: **a single-element enum is the best way to implement a singleton.**

**Think of it like the moon:** There is exactly one. No matter how many times you "look it up," you get the same one. The JVM guarantees this for enum constants automatically.

### The classic (flawed) singleton approaches

```java
// Approach 1: classic lazy singleton — NOT thread-safe without extra work,
// and can be BROKEN by reflection and broken by serialization.
public class Registry {
    private static Registry instance;
    private Registry() { }                       // private constructor...
    public static Registry getInstance() {
        if (instance == null) instance = new Registry();  // race condition under threads
        return instance;
    }
}
// Problems:
//  - Threads: two threads can both see null and create TWO instances.
//  - Reflection: setAccessible(true) on the private ctor can create a 2nd instance.
//  - Serialization: deserializing creates a NEW instance unless you add readResolve().
```

### The enum singleton — solves all of that for free

```java
public enum Registry {                       // single-element enum
    INSTANCE;                                 // the ONE and only instance

    private final Map<String, String> data = new ConcurrentHashMap<>();  // its state

    public void put(String key, String value) { data.put(key, value); }  // behavior
    public String get(String key)             { return data.get(key); }
}

// Usage anywhere in the app:
Registry.INSTANCE.put("env", "prod");
String env = Registry.INSTANCE.get("env");
```

### Why the enum singleton wins

| Concern | Classic singleton | Enum singleton |
|---|---|---|
| **Thread safety** | You must add `synchronized` / double-checked locking | Guaranteed by the JVM — constants are created once, safely |
| **Reflection attack** | Can be broken (`setAccessible`) | Impossible — JVM forbids reflectively constructing enums |
| **Serialization** | Creates a new instance unless you add `readResolve()` | Handled automatically — deserialization returns the same `INSTANCE` |
| **Boilerplate** | Lots | One word: `INSTANCE` |

> **Interview soundbite:** "The single-element enum is the best singleton (Effective Java Item 3): it's **thread-safe** and **serialization-safe** by construction, and it's **immune to reflection attacks** — none of which the classic private-constructor singleton gives you for free."
>
> **One caveat to mention:** an enum singleton can't *lazily* initialize based on constructor arguments and can't extend a class. For 99% of singleton needs, that's irrelevant.

---

## Persisting Enums — JPA @Enumerated

This is a **must-know backend topic**. When you store an enum field in a database via JPA/Hibernate, you choose **how** it's stored with `@Enumerated`. There are two modes: `ORDINAL` and `STRING`.

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @Enumerated(EnumType.STRING)    // stores the NAME as text → "PENDING", "SHIPPED"
    private OrderStatus status;
}

public enum OrderStatus { PENDING, PAID, SHIPPED, DELIVERED, CANCELLED }
```

### ORDINAL vs STRING

| | `EnumType.ORDINAL` | `EnumType.STRING` |
|---|---|---|
| **What's stored** | The `ordinal()` number: `0`, `1`, `2`... | The `name()` text: `"PENDING"`, `"SHIPPED"` |
| **DB column** | `INT` / `SMALLINT` | `VARCHAR` |
| **Readable in DB?** | No — `2` means nothing to a human | Yes — self-documenting |
| **Safe to reorder constants?** | **NO** — breaks every stored row | **Yes** — names are stable |
| **Safe to insert a constant in the middle?** | **NO** — shifts all later ordinals | **Yes** |
| **Storage size** | Tiny | Slightly larger (text) |
| **Default if you forget `@Enumerated`** | **ORDINAL** (the dangerous one!) | — |

### Why STRING is the safe default for backend

```java
// Suppose status was stored as ORDINAL:
public enum OrderStatus { PENDING, PAID, SHIPPED }       // PENDING=0, PAID=1, SHIPPED=2
// DB now has rows with status = 0, 1, 2

// A developer adds a new status in the middle (very common!):
public enum OrderStatus { PENDING, CONFIRMED, PAID, SHIPPED }
// Now PAID=2 and SHIPPED=3
// Every row that stored "1" (meaning PAID) now reads back as CONFIRMED. SILENT DATA CORRUPTION.
```

With `EnumType.STRING`, the DB stores `"PAID"`, and `"PAID"` always maps back to `OrderStatus.PAID` no matter where it sits in the declaration. Reordering and inserting are safe.

> **Interview Tip — say this:** "Always use `@Enumerated(EnumType.STRING)` for persisted enums. The default is `ORDINAL`, which stores positional integers — reordering or inserting a constant silently corrupts existing data. `STRING` is self-documenting in the DB and stable across code changes. The only costs are slightly more storage and renaming a constant requires a data migration."

A note on the trade-off: with `STRING`, **renaming** a constant (e.g., `PAID` → `SETTLED`) breaks old rows, so renames need a DB migration. But renames are rarer and more visible than reorders, so `STRING` is still the safer choice.

---

## Enums and JSON (Jackson)

In Spring Boot REST APIs, Jackson serializes enums to/from JSON. By **default**, Jackson uses the enum's `name()`:

```java
public enum Role { ADMIN, USER, GUEST }

// Serializing  Role.ADMIN  →  "ADMIN"   (uses name() by default)
// Deserializing "USER"     →  Role.USER (case-sensitive by default)
```

### Customizing the JSON value

```java
public enum Role {
    ADMIN("administrator"),     // internal name vs. the JSON value can differ
    USER("standard-user"),
    GUEST("guest");

    private final String json;
    Role(String json) { this.json = json; }

    @JsonValue                              // tells Jackson: serialize THIS value, not name()
    public String getJson() { return json; }
}
// Role.ADMIN now serializes to "administrator" instead of "ADMIN"
```

```java
// Accept unknown enum values gracefully instead of throwing on a bad request body
@JsonCreator                                // custom deserialization entry point
public static Role from(String value) {
    return Arrays.stream(values())
                 .filter(r -> r.json.equalsIgnoreCase(value))   // case-insensitive match
                 .findFirst()
                 .orElse(GUEST);            // unknown input → default, instead of 400 error
}
```

Useful Jackson knobs (set globally in config):

- `READ_UNKNOWN_ENUM_VALUES_AS_NULL` — unknown enum strings deserialize to `null` instead of failing.
- `@JsonProperty("...")` on a constant — map a specific JSON string to that constant.

> **Interview Tip:** "By default Jackson maps enums by `name()`. Use `@JsonValue` to control the serialized form and `@JsonCreator` for custom, case-insensitive, or fault-tolerant deserialization."

---

## Common Mistakes & Pitfalls

```java
// PITFALL 1: Using ordinal() for logic or storage
int code = status.ordinal();   // breaks the instant someone reorders the enum. Use a field instead.

// PITFALL 2: Forgetting @Enumerated → JPA silently defaults to ORDINAL
@Enumerated                      // <-- defaults to ORDINAL! Always specify STRING.
private OrderStatus status;
// FIX:
@Enumerated(EnumType.STRING)
private OrderStatus status;

// PITFALL 3: Inserting a constant in the MIDDLE when using ORDINAL persistence / serialized ordinals
enum Status { A, B, C }          // A=0, B=1, C=2  ... DB has these numbers
enum Status { A, X, B, C }       // now B=2, C=3 — all old data misaligned. Append at the END instead.

// PITFALL 4: valueOf() with wrong case or unknown value → IllegalArgumentException
Role r = Role.valueOf("admin"); // throws! name() is "ADMIN". Match case, or parse defensively.

// PITFALL 5: Using HashMap/HashSet for enum keys
Map<Day, Task> m = new HashMap<>();   // works but slower; prefer new EnumMap<>(Day.class)

// PITFALL 6: Comparing enums with .equals() when == is clearer and null-safe
if (status.equals(OrderStatus.PAID)) {}   // NPE if status is null
if (status == OrderStatus.PAID) {}        // null-safe, idiomatic — prefer this
```

**Golden rules:**

1. **Always append new constants at the end** (especially with ordinal-based storage/serialization).
2. **Never use `ordinal()`** for anything that must stay stable — add an explicit field.
3. **Persist with `EnumType.STRING`**, never the default `ORDINAL`.
4. **Use `EnumSet`/`EnumMap`** for enum collections.
5. **Compare with `==`**, not `.equals()`.

---

## Common Interview Questions

### Q: What is an enum and why use it over `int`/`String` constants?

An enum is a type with a fixed set of named constant values. It gives **type safety** (the compiler rejects invalid values), **readability** (`Status.ACTIVE` vs `0`), a proper namespace, and the ability to attach fields and behavior. `int`/`String` constants accept any value, so typos and invalid numbers compile fine and fail at runtime.

---

### Q: Can an enum have a constructor? Can you call `new` on it?

Yes, an enum can have a constructor used to initialize per-constant fields. No, you can never call `new` — the constructor is implicitly `private` and the JVM calls it once per constant when the enum class loads.

---

### Q: Can an enum extend a class? Implement an interface?

It **cannot extend** a class — every enum already implicitly extends `java.lang.Enum`, and Java has single inheritance. It **can implement** any number of interfaces, and each constant can provide its own method body.

---

### Q: What's the difference between `name()` and `ordinal()`? Why avoid `ordinal()`?

`name()` returns the declared constant name as a stable `String`. `ordinal()` returns the zero-based position in the declaration. Avoid `ordinal()` for logic or storage because reordering or inserting constants changes positions, silently corrupting any stored or transmitted values. Use an explicit instance field if you need a stable number.

---

### Q: What are constant-specific method bodies?

You declare an `abstract` method on the enum, and each constant supplies its own implementation. This is the strategy pattern built into the language — it replaces a `switch` on the enum value, and the compiler forces every new constant to implement the method (Effective Java Item 34).

---

### Q: Why are `EnumSet` and `EnumMap` preferred over `HashSet`/`HashMap`?

Because the set of enum constants is fixed and known, `EnumSet` is implemented as a **bit vector** and `EnumMap` as an **array indexed by `ordinal()`**. No hashing, compact memory, and iteration follows declaration order. They're significantly faster and lighter than hash-based collections for enum keys.

---

### Q: Why is a single-element enum the best way to write a singleton?

It's **thread-safe** (the JVM creates the constant once, safely), **serialization-safe** (deserialization returns the same instance, no `readResolve()` needed), and **immune to reflection attacks** (the JVM forbids reflectively constructing enums). A classic private-constructor singleton needs extra code for all three (Effective Java Item 3).

---

### Q: How should you persist an enum in a database with JPA?

Use `@Enumerated(EnumType.STRING)`. It stores the constant's name, which is readable in the DB and stable across reordering/insertion of constants. The default is `EnumType.ORDINAL`, which stores positional integers — reordering or inserting a constant silently corrupts existing rows.

---

### Q: What's the danger of `@Enumerated(EnumType.ORDINAL)` (or forgetting `@Enumerated`)?

It stores `ordinal()` integers. If anyone reorders constants or inserts one in the middle, every previously stored number now maps to a different constant — silent data corruption. Forgetting `@Enumerated` defaults to ORDINAL, so always specify `STRING` explicitly.

---

### Q: Can you add fields and methods to an enum? Give an example.

Yes. Each constant calls a constructor with arguments to initialize `final` fields, and the enum can have regular methods. Example: a `Currency` enum with `symbol` and `displayName` fields plus getters, or a `Planet` enum with `mass`/`radius` and a `surfaceGravity()` method.

---

### Q: How does Jackson serialize enums by default, and how do you customize it?

By default Jackson uses `name()` for serialization and matches by name (case-sensitive) for deserialization. Use `@JsonValue` on a method to control the serialized form, and `@JsonCreator` for custom or fault-tolerant deserialization (e.g., case-insensitive, default-on-unknown).

---

### Q: Should you compare enums with `==` or `.equals()`?

Use `==`. Each enum constant is a singleton, so reference equality and `.equals()` give the same result, but `==` is null-safe (`null == X` is just `false`) and reads more clearly.

---

### Q: What happens if `valueOf()` receives a name that doesn't exist?

It throws `IllegalArgumentException` (and `NullPointerException` if the argument is `null`). For untrusted input, wrap it in a try/catch or iterate `values()` to match defensively and return a default.

---

### Q: Are enum constants thread-safe?

Yes. Enum constants are created once at class-load time by the JVM in a thread-safe manner, and they're effectively immutable if you keep their fields `final`. This is exactly why the enum singleton is thread-safe for free.

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

*Last Updated: 2026-06-11*
