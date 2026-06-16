# Java Generics & Wildcards Study Guide

## Overview

Generics let you write code that works with **any type** while still being **type-safe** at compile time. They power almost every modern Java API — `List<String>`, `Map<K, V>`, `Optional<T>`, and the entire Spring/JPA stack (`JpaRepository<User, Long>`, `ResponseEntity<UserDto>`). This is a very common Java backend interview topic because it tests whether you truly understand type safety, the compiler, and the famous **PECS** rule.

This guide is written for junior developers. Every concept comes with a real-world analogy, runnable code with line-by-line comments, and the tricky "gotchas" interviewers love to ask about.

---

## Table of Contents

1. [Why Generics Exist (Life Before Generics)](#why-generics-exist-life-before-generics)
2. [Generic Classes](#generic-classes)
3. [Generic Methods](#generic-methods)
4. [Multiple Type Parameters](#multiple-type-parameters)
5. [Bounded Type Parameters](#bounded-type-parameters)
6. [Wildcards: ?, extends, super](#wildcards--extends-super)
7. [The PECS Rule (Producer Extends, Consumer Super)](#the-pecs-rule-producer-extends-consumer-super)
8. [Type Erasure](#type-erasure)
9. [Generics & Inheritance (Invariance)](#generics--inheritance-invariance)
10. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
11. [Generics in Real Spring / JPA Code](#generics-in-real-spring--jpa-code)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Why Generics Exist (Life Before Generics)

**Think of it like labeled storage boxes:**

- **Before generics** = a cardboard box with no label. You can throw anything inside (shoes, books, a cat). When you pull something out, you have *no idea* what it is, so you must guess and "cast" it. If you guess wrong, it breaks.
- **With generics** = a box with a clear label that says **"BOOKS ONLY"**. The label is checked the moment you try to put something in. You can never accidentally put a cat in a book box, and when you pull something out you *know* it's a book — no guessing, no casting.

### Life before generics (Java 1.4 and earlier)

Before Java 5 (2004), collections held `Object`. That meant **anything** could go in, and you had to cast everything coming out.

```java
// PRE-GENERICS CODE (raw types) — this still compiles today, but it's dangerous
List names = new ArrayList();   // a "raw" list — holds Object, no type info
names.add("Alice");             // adding a String — fine
names.add("Bob");               // another String — fine
names.add(42);                  // OOPS! an Integer slipped in — compiler says NOTHING

// Later, somewhere else in the code:
String first = (String) names.get(0);  // cast needed because list returns Object — "Alice", OK
String bad   = (String) names.get(2);  // tries to cast Integer 42 to String
// → throws ClassCastException AT RUNTIME — crashes in production, not at compile time!
```

**The two big problems before generics:**

1. **No type safety** — the compiler couldn't stop you from adding the wrong type. Bugs hid until runtime.
2. **Manual casting everywhere** — every `.get()` returned `Object`, so you cast constantly. Verbose and error-prone.

### The same code WITH generics

```java
// MODERN CODE with generics
List<String> names = new ArrayList<>();  // <String> = "this list holds ONLY Strings"
names.add("Alice");                      // fine — it's a String
names.add("Bob");                        // fine — it's a String
names.add(42);                           // COMPILE ERROR — caught instantly by the compiler!

String first = names.get(0);  // NO cast needed — compiler already knows it's a String
```

**What generics gave us:**

| Benefit | Before Generics | With Generics |
|---|---|---|
| Type safety | Errors found at **runtime** (crash) | Errors found at **compile time** (won't build) |
| Casting | Manual cast on every `.get()` | No casting — compiler infers the type |
| Readability | `List` — holds *what*? | `List<String>` — clearly holds Strings |
| Bugs | `ClassCastException` surprises | Caught before the app even runs |

> **Interview soundbite**: "Generics move type errors from runtime to compile time, and eliminate manual casting." That one sentence answers "why do generics exist?"

---

## Generic Classes

A **generic class** is a class that takes a *type* as a parameter, written in angle brackets `<T>`. The `T` is a placeholder that gets filled in when you create an object.

**Think of it like a recipe template:** A "smoothie recipe" says "blend `<FRUIT>` with ice." `<FRUIT>` is a blank you fill in later — banana smoothie, mango smoothie, etc. The *steps* stay identical; only the fruit changes. A generic class is the recipe; `T` is the blank.

```java
// Box<T> — a container that can hold ONE value of ANY type T
public class Box<T> {           // T is a "type parameter" — a placeholder for a real type
    private T content;          // the field's type is T (decided when you create the Box)

    public void set(T content) {  // accepts a value of type T
        this.content = content;
    }

    public T get() {              // returns a value of type T
        return content;
    }
}
```

```java
// Using the generic class — you "fill in the blank" T with a real type
Box<String> stringBox = new Box<>();  // here T becomes String
stringBox.set("Hello");               // can ONLY set a String now
String s = stringBox.get();           // returns String — no cast needed

Box<Integer> intBox = new Box<>();    // here T becomes Integer
intBox.set(100);                      // can ONLY set an Integer
int n = intBox.get();                 // returns Integer (auto-unboxed to int)

stringBox.set(42);  // COMPILE ERROR — this box only holds Strings
```

### Type parameter naming conventions

These are just conventions (single uppercase letters), but interviewers expect you to know them:

| Letter | Stands For | Typical Use |
|---|---|---|
| `T` | **T**ype | A generic type (most common) |
| `E` | **E**lement | Elements in a collection (`List<E>`) |
| `K` | **K**ey | A map key (`Map<K, V>`) |
| `V` | **V**alue | A map value (`Map<K, V>`) |
| `N` | **N**umber | A numeric type |
| `R` | **R**eturn | A return type (common in functional interfaces) |

> **Note**: `T` is not magic — you could name it `Banana` and it would still compile. The single-letter convention just makes generic code readable.

---

## Generic Methods

A **generic method** declares its *own* type parameter, independent of any class. The `<T>` goes **before the return type**. This lets a single method work with many types.

**Think of it like a universal adapter plug:** A travel adapter doesn't care if you plug in a phone, laptop, or razor — it adapts to whatever device you bring. A generic method adapts to whatever type you pass in.

```java
public class Utils {

    // <T> before the return type = "this method has its own type parameter T"
    public static <T> T firstElement(List<T> list) {  // takes a List<T>, returns a T
        return list.get(0);                           // returns the first element as type T
    }

    // A generic method that prints any array, regardless of element type
    public static <E> void printAll(E[] array) {  // E is inferred from the array you pass
        for (E element : array) {                 // loop variable is type E
            System.out.println(element);
        }
    }

    // Returns the larger of two values — works for any Comparable type
    public static <T extends Comparable<T>> T max(T a, T b) {  // bounded (explained later)
        return (a.compareTo(b) > 0) ? a : b;                  // compare and return the bigger one
    }
}
```

```java
// Calling generic methods — the compiler INFERS T from the arguments
List<String> words = List.of("apple", "banana");
String w = Utils.firstElement(words);  // compiler infers T = String automatically

Integer[] nums = {3, 1, 4, 1, 5};
Utils.printAll(nums);                  // compiler infers E = Integer

String bigger = Utils.max("cat", "dog");  // infers T = String → returns "dog"
Integer biggest = Utils.max(10, 20);      // infers T = Integer → returns 20
```

### Generic method vs generic class — what's the difference?

| | Generic Class | Generic Method |
|---|---|---|
| Where `<T>` is declared | After the class name: `class Box<T>` | Before the return type: `<T> T method(...)` |
| Scope of `T` | The whole class | Just that one method |
| Type chosen when | You create the object (`new Box<String>()`) | You call the method (usually inferred) |
| Example | `class Box<T>` | `static <T> T first(List<T> l)` |

> **Interview tip**: You can have a generic method **inside a non-generic class**, and even a generic method with a *different* type parameter inside a generic class. The method's `<T>` is completely separate from the class's `<T>`.

---

## Multiple Type Parameters

A generic class or method can take **more than one** type parameter, separated by commas. The classic example is a key-value `Pair`.

**Think of it like a labeled luggage tag:** A tag has two slots — a "key" (your name) and a "value" (a phone number). They're different kinds of information, so they get different type slots.

```java
// Pair<K, V> — holds two values of possibly different types
public class Pair<K, V> {       // TWO type parameters: K and V
    private final K key;        // first slot, type K
    private final V value;      // second slot, type V

    public Pair(K key, V value) {  // constructor takes one of each
        this.key = key;
        this.value = value;
    }

    public K getKey()   { return key; }    // returns type K
    public V getValue() { return value; }  // returns type V
}
```

```java
// K and V can be DIFFERENT types — or the same
Pair<String, Integer> age = new Pair<>("Alice", 30);  // K=String, V=Integer
String name = age.getKey();      // "Alice"
Integer years = age.getValue();  // 30

Pair<String, String> capital = new Pair<>("France", "Paris");  // K and V both String
```

This is exactly how `Map<K, V>` works — every entry is essentially a `Pair` of a key type and a value type:

```java
Map<String, List<Integer>> scores = new HashMap<>();  // key=String, value=List<Integer>
scores.put("Alice", List.of(90, 85, 100));            // a String mapped to a list of ints
```

---

## Bounded Type Parameters

By default, a type parameter `T` can be **any** type. A **bound** restricts `T` to be a subtype of something, using the keyword `extends`. This unlocks methods of that bound.

**Think of it like a "must be 18+" sign at a venue:** Without the sign, *anyone* can enter. With "must be 18+", you restrict entry to a category of people — and now you're *allowed* to serve them things only adults can have. The bound restricts the type but lets you safely use that type's abilities.

### Why bounds are needed

```java
// WITHOUT a bound — T could be ANYTHING, so we CAN'T call number methods
public static <T> double sumBad(List<T> list) {
    double total = 0;
    for (T item : list) {
        total += item.doubleValue();  // COMPILE ERROR — T might be String! No doubleValue()
    }
    return total;
}

// WITH a bound — T must be a Number (or subclass), so number methods are allowed
public static <T extends Number> double sum(List<T> list) {  // T extends Number
    double total = 0;
    for (T item : list) {            // item is guaranteed to be a Number
        total += item.doubleValue(); // OK — every Number has doubleValue()
    }
    return total;
}
```

```java
sum(List.of(1, 2, 3));          // T = Integer (Integer extends Number) — OK
sum(List.of(1.5, 2.5));         // T = Double  (Double extends Number)  — OK
sum(List.of("a", "b"));         // COMPILE ERROR — String is NOT a Number
```

### `extends` works for both classes AND interfaces

In generics, `extends` means "is a subtype of" — it covers both `extends` (classes) and `implements` (interfaces). You always write `extends`, never `implements`, inside `<>`.

```java
// T must implement Comparable so we can call compareTo()
public static <T extends Comparable<T>> T max(List<T> list) {
    T biggest = list.get(0);            // start with the first element
    for (T item : list) {
        if (item.compareTo(biggest) > 0) {  // compareTo comes from Comparable
            biggest = item;
        }
    }
    return biggest;
}
```

### Multiple bounds

A type parameter can have **several** bounds joined with `&`. Rule: if one bound is a **class**, it must come **first**; interfaces follow.

```java
// T must BE a subclass of Animal AND implement BOTH Comparable and Serializable
public static <T extends Animal & Comparable<T> & Serializable> void process(T t) {
    // Inside here you can use Animal methods, compareTo(), and treat t as Serializable
}
// Order rule: class (Animal) FIRST, then interfaces. This compiles.
// <T extends Comparable<T> & Animal> would NOT compile — class must be first.
```

| Bound Syntax | Meaning |
|---|---|
| `<T>` | T can be any type |
| `<T extends Number>` | T must be Number or a subclass |
| `<T extends Comparable<T>>` | T must implement Comparable |
| `<T extends Animal & Serializable>` | T must extend Animal AND implement Serializable |

---

## Wildcards: ?, extends, super

A **wildcard** is the question mark `?`. It represents an **unknown type**. You use wildcards in *method parameters* when you want to accept many different generic types, not just one specific one.

**Think of it like a job posting:**
- `List<?>` (unbounded) = "We accept applicants of **any** background." You can read their resume, but you can't assume any specific skill.
- `List<? extends Number>` (upper bound) = "Applicants must have a **finance** background (Number or a subtype)." You know they can do finance things, so you can *read* finance skills from them.
- `List<? super Integer>` (lower bound) = "We're hiring **into** the Integer role; anyone Integer-or-more-general qualifies." You can safely *put* an Integer into this slot.

### 1. Unbounded wildcard: `<?>`

`List<?>` means "a list of some unknown type." Use it when your method doesn't care about the element type — it only uses methods that don't depend on `T` (like `.size()` or printing).

```java
// Accepts a List of ANY type — String, Integer, anything
public static void printSize(List<?> list) {   // ? = unknown element type
    System.out.println("Size: " + list.size()); // size() doesn't care about element type — OK
    for (Object o : list) {                      // we can only read elements as Object
        System.out.println(o);                   // printing as Object — fine
    }
    // list.add("x");  // COMPILE ERROR — we don't know the type, so we can't safely add anything
}
```

```java
printSize(List.of("a", "b", "c"));  // works with List<String>
printSize(List.of(1, 2, 3));        // works with List<Integer>
```

> **Why can't you `add` to a `List<?>`?** Because the list could be a `List<String>` OR a `List<Integer>` — the compiler doesn't know which. Adding *anything* (except `null`) might violate the real type, so the compiler forbids it.

### 2. Upper-bounded wildcard: `<? extends T>`

`List<? extends Number>` means "a list of Number **or any subtype** of Number" (Integer, Double, etc.). You can **READ** elements as `Number`, but you **cannot ADD** (except `null`).

```java
// Accepts List<Number>, List<Integer>, List<Double>, etc.
public static double sumList(List<? extends Number> list) {  // upper bound: Number
    double total = 0;
    for (Number n : list) {       // safe to READ as Number — every element IS a Number
        total += n.doubleValue();
    }
    return total;
    // list.add(5);  // COMPILE ERROR — could be List<Double>, so adding an Integer is unsafe
}
```

```java
sumList(List.of(1, 2, 3));        // List<Integer> — accepted (Integer extends Number)
sumList(List.of(1.5, 2.5));       // List<Double>  — accepted (Double extends Number)
```

### 3. Lower-bounded wildcard: `<? super T>`

`List<? super Integer>` means "a list of Integer **or any supertype** of Integer" (Integer, Number, Object). You can **ADD** Integers, but reading only gives you `Object`.

```java
// Accepts List<Integer>, List<Number>, List<Object>
public static void addNumbers(List<? super Integer> list) {  // lower bound: Integer
    list.add(1);   // SAFE to ADD an Integer — the list holds Integer or a supertype
    list.add(2);   // also safe
    // Integer x = list.get(0);  // COMPILE ERROR — could be List<Object>, so we only get Object
    Object o = list.get(0);      // reading only gives Object (the one type we're sure of)
}
```

```java
List<Integer> ints = new ArrayList<>();
addNumbers(ints);                 // List<Integer> — accepted

List<Number> nums = new ArrayList<>();
addNumbers(nums);                 // List<Number> — accepted (Number is a supertype of Integer)

List<Object> objs = new ArrayList<>();
addNumbers(objs);                 // List<Object> — accepted (Object is a supertype of Integer)
```

### Wildcard comparison table

| Wildcard | Meaning | Can READ? | Can ADD? | Use When |
|---|---|---|---|---|
| `<?>` | Any unknown type | As `Object` only | No (except `null`) | You don't care about the type |
| `<? extends T>` | T or a **sub**type | Yes, as `T` | **No** (except `null`) | You **read/produce** values |
| `<? super T>` | T or a **super**type | As `Object` only | **Yes**, T or subtype | You **write/consume** values |

> **The memory trick**: `extends` = "I want to **get stuff out**" (read). `super` = "I want to **put stuff in**" (write). This leads directly to PECS below.

---

## The PECS Rule (Producer Extends, Consumer Super)

**PECS = Producer Extends, Consumer Super.** This is the single most asked generics interview question. It tells you *which* wildcard to use.

**Think of it like a vending machine vs a donation bin:**
- A **Producer** *gives* you things (a vending machine produces snacks → you take them OUT). For something you read from → use **`extends`**.
- A **Consumer** *takes* things in (a donation bin consumes your old clothes → you put them IN). For something you write to → use **`super`**.

### The definition

- **Producer Extends**: If a parameter **produces** (you read FROM it), use `<? extends T>`.
- **Consumer Super**: If a parameter **consumes** (you write INTO it), use `<? super T>`.

### A real example: a `copy` method (this is in `java.util.Collections`)

```java
// Copy all elements FROM 'src' INTO 'dest'
// src PRODUCES elements (we read from it)   → ? extends T
// dest CONSUMES elements (we write into it) → ? super T
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++) {
        T item = src.get(i);   // READ from src (the producer) — extends lets us read as T
        dest.add(item);        // WRITE into dest (the consumer) — super lets us add a T
    }
}
```

```java
List<Integer> source = List.of(1, 2, 3);   // produces Integers
List<Number> target = new ArrayList<>();   // consumes Integers (Number is a supertype)

copy(target, source);  // works! target gets [1, 2, 3]
// src is List<Integer> (a producer → extends)
// dest is List<Number> (a consumer → super, because Number is a supertype of Integer)
```

### Why PECS matters — what breaks without it

```java
// If 'src' were just List<T> (no extends), you couldn't pass a List<Integer> to a copy<Number>.
// If 'dest' were just List<T> (no super), you couldn't copy Integers into a List<Number>.
// PECS makes the method flexible enough to accept the natural subtype/supertype relationships.
```

### PECS quick decision guide

| What is the parameter doing? | Wildcard to use | Example |
|---|---|---|
| You **read** values out of it (it's a source) | `? extends T` | `List<? extends Number> source` |
| You **write** values into it (it's a destination) | `? super T` | `List<? super Integer> dest` |
| You do **both** read and write | No wildcard — use exact `T` | `List<T> both` |

> **Interview gold**: `Collections.copy(dest, src)`, `Collections.max(Collection<? extends T>)`, and `Stream.forEach(Consumer<? super T>)` are all real JDK examples of PECS. Mention one and you sound like you've read the source code.

---

## Type Erasure

**Type erasure** is the mechanism Java uses to implement generics. At **compile time**, the compiler checks all your generic types. Then it **erases** (removes) the type info, and at **runtime** the generic type is gone — replaced by `Object` (or the bound).

**Think of it like training wheels on a bike:** The training wheels (generics) keep you safe *while you learn to ride* (compile time). Once you're actually riding on the road (runtime), the training wheels are removed — the bike runs without them. The safety happened during practice, not on the road.

### What erasure actually does

```java
// What YOU write:
List<String> list = new ArrayList<>();
list.add("hello");
String s = list.get(0);

// What the COMPILER produces after erasure (roughly):
List list = new ArrayList();        // <String> is ERASED
list.add("hello");
String s = (String) list.get(0);    // compiler INSERTS the cast for you automatically
```

```java
// Erasure replaces unbounded T with Object, and bounded T with its bound:
class Box<T> { T value; }                  // T becomes Object at runtime
class NumBox<T extends Number> { T value; } // T becomes Number at runtime
```

### Why does erasure exist?

Java added generics in version 5 (2004) but needed **backward compatibility** — old pre-generics code and new generic code had to run together on the same JVM. Erasure means generics are a **compile-time-only** feature; the bytecode looks just like old Java 1.4 code. No JVM changes were required.

### Runtime consequences of erasure (interview favorites!)

Because the type is gone at runtime, several things are **impossible**:

```java
// 1. CANNOT use instanceof with a generic type
if (list instanceof List<String>) { }  // COMPILE ERROR — <String> doesn't exist at runtime
if (list instanceof List<?>)      { }  // OK — only the raw/unbounded form is allowed

// 2. CANNOT create a generic array directly
T[] array = new T[10];  // COMPILE ERROR — JVM doesn't know what T is at runtime
// Workaround: create an Object[] and cast, or pass a Class<T> / array constructor
@SuppressWarnings("unchecked")
T[] array2 = (T[]) new Object[10];  // works but generates an "unchecked" warning

// 3. CANNOT call new T() — no way to instantiate an erased type
T obj = new T();  // COMPILE ERROR — T is unknown at runtime
// Workaround: pass a Supplier<T> or a Class<T> and use clazz.getDeclaredConstructor().newInstance()

// 4. Two generic types share ONE class object — the <T> is erased
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass());  // prints TRUE — both are just ArrayList.class

// 5. CANNOT overload methods that differ only by generic type
void print(List<String> l) { }
void print(List<Integer> l) { }  // COMPILE ERROR — after erasure both are print(List), a clash
```

| What you CAN'T do (because of erasure) | Why | Workaround |
|---|---|---|
| `new T()` | T unknown at runtime | Pass `Supplier<T>` or `Class<T>` |
| `new T[10]` | Array type unknown | `(T[]) new Object[10]` (unchecked) |
| `obj instanceof List<String>` | `<String>` erased | Use `instanceof List<?>` |
| Overload by generic type only | Same erased signature | Rename one method |
| Read the generic type at runtime | It's gone | Pass `Class<T>` explicitly (type token) |

> **Interview tip**: "Generics are a compile-time feature. Thanks to type erasure, the type information does not exist at runtime — `List<String>` and `List<Integer>` are both just `List` in the bytecode." Memorize this.

---

## Generics & Inheritance (Invariance)

Here's the concept that trips up almost everyone: **`List<String>` is NOT a subtype of `List<Object>`**, even though `String` IS a subtype of `Object`. Generics are **invariant**.

**Think of it like boxes of fruit:** A `Banana` is a kind of `Fruit`. But a *crate labeled "BANANAS ONLY"* is **not** the same as a *crate labeled "ANY FRUIT"*. If you treated the banana crate as a generic fruit crate, someone could drop an apple into it — violating the "bananas only" label. So Java forbids the substitution to keep you safe.

### Why invariance exists — the unsafe scenario it prevents

```java
List<String> strings = new ArrayList<>();
strings.add("hello");

// IF this were allowed (it is NOT — it's a compile error):
List<Object> objects = strings;  // COMPILE ERROR — List<String> is not a List<Object>

// ...because if it WERE allowed, this could happen:
objects.add(42);                 // adding an Integer into what is really a List<String>!
String s = strings.get(1);       // would explode — Integer is not a String
// Java blocks the assignment at line 7 to make this disaster impossible.
```

### The contrast with arrays (arrays ARE covariant — and that's a flaw)

Arrays behave differently from generics, and it's a known wart in Java:

```java
// Arrays ARE covariant — this compiles but is DANGEROUS
Object[] array = new String[3];  // allowed: String[] IS treated as Object[]
array[0] = 42;                   // compiles fine... but throws ArrayStoreException AT RUNTIME!

// Generics are INVARIANT — the same mistake is caught at COMPILE time instead
List<Object> list = new ArrayList<String>();  // COMPILE ERROR — caught early, much safer
```

| Feature | Relationship | Error caught at |
|---|---|---|
| **Arrays** | Covariant: `String[]` IS an `Object[]` | **Runtime** (`ArrayStoreException`) — unsafe |
| **Generics** | Invariant: `List<String>` is NOT a `List<Object>` | **Compile time** — safe |

### How wildcards bring back flexibility

Invariance is safe but rigid. Wildcards re-introduce flexibility *safely*:

```java
// This does NOT compile — invariance:
void printAll(List<Object> list) { }
printAll(new ArrayList<String>());  // ERROR — List<String> is not List<Object>

// This DOES compile — wildcard accepts any element type:
void printAll(List<?> list) { }     // <?> = "list of some unknown type"
printAll(new ArrayList<String>());  // OK
printAll(new ArrayList<Integer>()); // OK
```

> **Interview soundbite**: "Generics are invariant to preserve type safety. `List<String>` is not a `List<Object>` because that would let you sneak the wrong type into the list. Wildcards (`? extends`, `? super`) restore controlled flexibility."

---

## Common Mistakes & Gotchas

### Gotcha 1: Using raw types (dropping the `<>`)

```java
List list = new ArrayList();   // RAW type — you lost ALL type safety, back to pre-2004
list.add("a");
list.add(1);                   // compiler allows it — bug waiting to happen
// ALWAYS write List<String> or at least List<?> — never a bare List
```

### Gotcha 2: Thinking you can `add` to `<? extends T>`

```java
List<? extends Number> nums = new ArrayList<Integer>();
nums.add(1);   // COMPILE ERROR — extends is for READING, not adding (PECS!)
Number n = nums.get(0);  // reading is fine
```

### Gotcha 3: Expecting type info at runtime

```java
public <T> void create(List<T> list) {
    // T t = new T();                  // ERROR — can't instantiate, type erased
    // if (list instanceof List<String>) // ERROR — can't check generic type at runtime
}
```

### Gotcha 4: Confusing `<T>` (a definition) with `<?>` (a usage)

```java
// <T> here DECLARES a new type parameter for the method:
public <T> T pick(List<T> list) { return list.get(0); }

// <?> here is a WILDCARD — it does NOT declare anything, it means "unknown type":
public void show(List<?> list) { System.out.println(list.size()); }
// Rule of thumb: declare <T> when you need to NAME and REUSE the type (e.g., in the return).
// Use <?> when you just accept "any type" and don't need to name it.
```

### Gotcha 5: Forgetting bounds, then being unable to call methods

```java
public <T> T biggest(List<T> list) {
    // list.get(0).compareTo(list.get(1));  // ERROR — T has no compareTo()
    return null;
}
// FIX: add the bound so the method becomes available:
public <T extends Comparable<T>> T biggest2(List<T> list) {
    return list.get(0).compareTo(list.get(1)) > 0 ? list.get(0) : list.get(1);  // OK now
}
```

### Gotcha 6: Mixing arrays and generics

```java
List<String>[] arr = new List<String>[10];  // COMPILE ERROR — can't create generic arrays
List<String>[] arr2 = new List[10];          // raw-array workaround (generates a warning)
// Prefer: List<List<String>> instead of an array of generics
```

| Mistake | Fix |
|---|---|
| Raw type `List` | Use `List<String>` or `List<?>` |
| `add` to `<? extends T>` | Use `<? super T>` if you need to add |
| `new T()` / `new T[]` | Pass `Class<T>` or `Supplier<T>` |
| `instanceof List<String>` | Use `instanceof List<?>` |
| No bound but calling methods | Add `<T extends SomeType>` |

---

## Generics in Real Spring / JPA Code

Generics are *everywhere* in Spring and JPA. Recognizing them in real code is what interviewers actually want from a backend junior.

### 1. Spring Data JPA repositories — `JpaRepository<T, ID>`

```java
// JpaRepository<EntityType, PrimaryKeyType> — two type parameters
public interface UserRepository extends JpaRepository<User, Long> {
    // T = User (the entity), ID = Long (the type of the @Id field)
    // Because of generics, ALL inherited methods are already typed for User:
    //   Optional<User> findById(Long id);   ← returns User, takes Long — no casting!
    //   List<User> findAll();
    //   User save(User entity);
    Optional<User> findByEmail(String email);  // returns Optional<User> automatically
}
```

Generics let Spring define `JpaRepository<T, ID>` **once**, and every repository you create is automatically type-safe for *its* entity. Without generics, `findById` would return `Object` and you'd cast everywhere.

### 2. `ResponseEntity<T>` — typed HTTP responses

```java
@RestController
public class UserController {

    // ResponseEntity<User> = an HTTP response whose body is a User
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);  // body type is checked to be User at compile time
    }

    // ResponseEntity<List<UserDto>> = response body is a list of DTOs
    @GetMapping("/users")
    public ResponseEntity<List<UserDto>> getUsers() {
        List<UserDto> dtos = userService.findAllDtos();
        return ResponseEntity.ok(dtos);  // compiler guarantees the body is List<UserDto>
    }
}
```

### 3. `Optional<T>` — a typed "maybe" container

```java
// Optional<User> = "maybe a User, maybe nothing" — no more null surprises
Optional<User> maybeUser = userRepository.findByEmail("a@b.com");
User user = maybeUser.orElseThrow(() -> new NotFoundException());  // returns a User, type-safe
```

### 4. DTO mapping with `List<T>` and streams

```java
// Converting List<User> (entities) to List<UserDto> (response objects)
public List<UserDto> toDtos(List<User> users) {       // typed input and output
    return users.stream()                             // Stream<User>
                .map(user -> new UserDto(user))       // Stream<UserDto> — type changes safely
                .collect(Collectors.toList());        // List<UserDto>
}
```

### 5. A generic base service (real-world pattern)

```java
// A reusable base service for ANY entity — generics make it work for all of them
public abstract class BaseService<T, ID> {       // T = entity type, ID = id type
    protected final JpaRepository<T, ID> repository;  // typed repository

    protected BaseService(JpaRepository<T, ID> repository) {
        this.repository = repository;
    }

    public T findById(ID id) {                   // returns the entity type T
        return repository.findById(id)
                         .orElseThrow(() -> new NotFoundException());
    }

    public List<T> findAll() {                   // returns a typed list
        return repository.findAll();
    }
}

// Concrete service just fills in the blanks — no duplicated CRUD code
@Service
public class UserService extends BaseService<User, Long> {  // T=User, ID=Long
    public UserService(UserRepository repo) { super(repo); }
}
```

| Spring/JPA Type | Generic Shape | What the Type Params Mean |
|---|---|---|
| `JpaRepository<T, ID>` | 2 params | Entity type, primary-key type |
| `ResponseEntity<T>` | 1 param | The HTTP response body type |
| `Optional<T>` | 1 param | The value that may or may not be present |
| `List<T>` / `Page<T>` | 1 param | The element type in the collection/page |
| `ResponseEntity<List<UserDto>>` | nested | A response body that is a list of DTOs |

---

## Common Interview Questions

### Q: Why were generics added to Java? What problem do they solve?

Generics provide **compile-time type safety** and **eliminate manual casting**. Before generics (Java 1.4), collections held `Object`, so the wrong type could be added without error and you got a `ClassCastException` at runtime. Generics catch those errors at compile time and remove the need to cast every `.get()`.

---

### Q: What is the difference between a generic class and a generic method?

A **generic class** declares its type parameter after the class name (`class Box<T>`), and the type is chosen when you create an object (`new Box<String>()`). A **generic method** declares its *own* type parameter before the return type (`<T> T first(List<T> l)`), and the type is usually inferred when you call it. A generic method can live inside a non-generic class.

---

### Q: What is a bounded type parameter? Give an example.

A bound restricts what `T` can be, using `extends`. For example, `<T extends Number>` means `T` must be `Number` or a subclass, which lets you call `Number` methods like `doubleValue()` inside the method. `<T extends Comparable<T>>` lets you call `compareTo()`. You can have multiple bounds with `&`: `<T extends Animal & Serializable>` (the class, if any, must come first).

---

### Q: Explain the three kinds of wildcards.

- **`<?>` (unbounded)** — any unknown type; you can only read elements as `Object` and cannot add (except `null`).
- **`<? extends T>` (upper bound)** — T or a subtype; you can **read** as `T` but cannot add.
- **`<? super T>` (lower bound)** — T or a supertype; you can **add** T (or a subtype) but reading only gives `Object`.

---

### Q: What is the PECS rule?

**Producer Extends, Consumer Super.** If a parameter is a **producer** (you read values out of it), use `<? extends T>`. If it's a **consumer** (you write values into it), use `<? super T>`. If you both read and write, use an exact type `T` (no wildcard). Real JDK example: `Collections.copy(List<? super T> dest, List<? extends T> src)` — `dest` consumes (super), `src` produces (extends).

---

### Q: Why can't you add elements to a `List<? extends Number>`?

Because `<? extends Number>` could be a `List<Integer>`, a `List<Double>`, or any subtype list — the compiler doesn't know the exact type. If it let you add a `Double` into what is really a `List<Integer>`, type safety would break. So adding is forbidden (you may only add `null`). You can still **read** elements as `Number`.

---

### Q: What is type erasure?

Type erasure is how Java implements generics. The compiler uses the generic types to check your code, then **removes** the type information so the bytecode contains only raw types (`List` instead of `List<String>`), with `T` replaced by `Object` or its bound. This exists for **backward compatibility** — generic code runs on the same JVM as old pre-generics code with no JVM changes.

---

### Q: Name some things you cannot do because of type erasure.

- Can't call `new T()` (type unknown at runtime).
- Can't create `new T[10]` (generic arrays).
- Can't use `instanceof List<String>` (only `instanceof List<?>`).
- Can't overload two methods that differ only by generic type (same erased signature).
- Can't read the actual generic type at runtime (`List<String>.class` doesn't exist — both are `List.class`).

---

### Q: Is `List<String>` a subtype of `List<Object>`? Why or why not?

**No.** Generics are **invariant**. Even though `String` is a subtype of `Object`, `List<String>` is **not** a subtype of `List<Object>`. If it were, you could assign a `List<String>` to a `List<Object>` reference and then add an `Integer`, corrupting the original list. Java forbids this assignment at compile time to keep it type-safe.

---

### Q: How are arrays and generics different regarding inheritance?

Arrays are **covariant**: `String[]` IS treated as an `Object[]`, so you can assign one to the other — but writing the wrong type throws `ArrayStoreException` at **runtime**. Generics are **invariant**: `List<String>` is NOT a `List<Object>`, so the same mistake is caught at **compile time**. Generics are safer because the error surfaces earlier.

---

### Q: What does the `<T>` before a method's return type mean?

It **declares** a new type parameter scoped to that method. For example, in `public static <T> T first(List<T> list)`, the leading `<T>` says "this method introduces its own type variable `T`," which is then used in the parameter and return type. Without it, the compiler would think `T` is an actual class name and fail.

---

### Q: How do generics appear in Spring Data JPA?

`JpaRepository<T, ID>` is generic: `T` is the entity type and `ID` is the primary-key type. When you write `interface UserRepository extends JpaRepository<User, Long>`, all inherited methods become type-safe for `User` (e.g., `findById(Long)` returns `Optional<User>`). This is why you never cast results from a Spring Data repository.

---

### Q: When would you use `<?>` instead of `<T>`?

Use `<?>` (a wildcard) when a method only needs to accept "a collection of *some* type" and doesn't need to name or reuse that type — for example, a `printSize(List<?> list)` that only calls `.size()`. Use `<T>` (a declared type parameter) when you need to refer to the type elsewhere, such as returning an element of that type or relating two parameters.

---

### Q: Can you have a generic method in a non-generic class?

Yes. A generic method declares its own type parameter independently of the class. For example, `java.util.Collections` is not a generic class, but it has many generic methods like `<T> void sort(List<T> list)`. The method's `<T>` is completely separate from any class-level type parameter.

---

## Quick Reference Cheat Sheet

```
WHY GENERICS:
  → compile-time type safety (catch errors before runtime)
  → no manual casting (no more ClassCastException surprises)
  Before: List list; (String) list.get(0);   ← raw, casts, runtime errors
  After:  List<String> list; list.get(0);     ← typed, no cast, compile-time safe

DECLARING:
  Generic class   → class Box<T> { T value; }
  Generic method  → <T> T first(List<T> l)        ← <T> goes BEFORE return type
  Multiple params → class Pair<K, V>              ← e.g. Map<K, V>

TYPE PARAMETER NAMES (convention):
  T = Type   E = Element   K = Key   V = Value   N = Number   R = Return

BOUNDS:
  <T>                          → any type
  <T extends Number>           → T is Number or subclass (can call Number methods)
  <T extends Comparable<T>>    → can call compareTo()
  <T extends Animal & Serializable> → multiple bounds (class FIRST, then interfaces)

WILDCARDS:
  <?>             → unknown type; read as Object; CANNOT add (except null)
  <? extends T>   → T or subtype; can READ as T; CANNOT add        ← PRODUCER
  <? super T>     → T or supertype; can ADD T; read only as Object ← CONSUMER

PECS (Producer Extends, Consumer Super):
  read FROM it (source)      → ? extends T
  write INTO it (destination)→ ? super T
  both read AND write        → exact T (no wildcard)
  JDK example: Collections.copy(List<? super T> dest, List<? extends T> src)

TYPE ERASURE (generics are compile-time only):
  At runtime, List<String> and List<Integer> are BOTH just List
  CANNOT: new T()  |  new T[10]  |  instanceof List<String>
  CANNOT: overload by generic type only (same erased signature)
  WHY: backward compatibility with pre-Java-5 code

INVARIANCE (inheritance):
  List<String> is NOT a List<Object>   ← generics are INVARIANT (safe, compile-time)
  String[]     IS  an  Object[]        ← arrays are COVARIANT (unsafe, runtime error)
  Use wildcards (<?>, <? extends>, <? super>) to add flexibility safely

REAL SPRING / JPA:
  JpaRepository<T, ID>          → T = entity, ID = primary-key type
  ResponseEntity<T>             → typed HTTP response body
  Optional<T>                   → maybe-a-value container
  List<T> / Page<T>             → typed collections
  ResponseEntity<List<UserDto>> → nested generics

TOP GOTCHAS:
  raw type List              → always use List<String> or List<?>
  add to <? extends T>       → not allowed; use <? super T> to add
  new T() / new T[]          → pass Class<T> or Supplier<T> instead
  instanceof List<String>    → use instanceof List<?>
  <T> vs <?>                 → <T> declares & names a type; <?> = "any unknown type"
```

---

*Last Updated: 2026-06-11*
