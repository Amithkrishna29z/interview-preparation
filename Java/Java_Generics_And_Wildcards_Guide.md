# Java Generics & Wildcards Study Guide

## Overview

Generics let you write code that works with **any type** while staying **type-safe** at compile time. They power almost every modern Java API — `List<String>`, `Map<K, V>`, `Optional<T>` — and the whole Spring/JPA stack (`JpaRepository<User, Long>`, `ResponseEntity<UserDto>`).

This is a very common backend interview topic because it tests type safety, the compiler, and the famous **PECS** rule.

---

## Table of Contents

1. [Why Generics Exist](#why-generics-exist)
2. [Generic Classes](#generic-classes)
3. [Generic Methods](#generic-methods)
4. [Multiple Type Parameters](#multiple-type-parameters)
5. [Bounded Type Parameters](#bounded-type-parameters)
6. [Wildcards: ?, extends, super](#wildcards--extends-super)
7. [The PECS Rule](#the-pecs-rule)
8. [Type Erasure](#type-erasure)
9. [Generics & Inheritance (Invariance)](#generics--inheritance-invariance)
10. [Common Gotchas](#common-gotchas)
11. [Generics in Spring / JPA](#generics-in-spring--jpa)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Why Generics Exist

Before Java 5 (2004), collections held `Object`. Anything could go in, and you cast everything coming out — so type errors hid until runtime:

```java
List names = new ArrayList();   // raw list — holds Object
names.add("Alice");
names.add(42);                  // OOPS — compiler says nothing
String bad = (String) names.get(1);  // ClassCastException AT RUNTIME
```

With generics, the type is checked at compile time and no casting is needed:

```java
List<String> names = new ArrayList<>();  // holds ONLY Strings
names.add("Alice");
names.add(42);              // COMPILE ERROR — caught instantly
String first = names.get(0); // no cast needed
```

> **Interview soundbite**: "Generics move type errors from runtime to compile time, and eliminate manual casting."

---

## Generic Classes

A **generic class** takes a *type* as a parameter in angle brackets `<T>`. `T` is a placeholder filled in when you create an object.

```java
public class Box<T> {        // T is a type parameter — a placeholder
    private T content;
    public void set(T content) { this.content = content; }
    public T get() { return content; }
}
```

```java
Box<String> stringBox = new Box<>();  // T becomes String
stringBox.set("Hello");
String s = stringBox.get();           // no cast needed
stringBox.set(42);                    // COMPILE ERROR — Strings only
```

**Naming conventions** (just conventions, but expected): `T` = Type, `E` = Element, `K` = Key, `V` = Value, `N` = Number, `R` = Return.

---

## Generic Methods

A **generic method** declares its own type parameter, with `<T>` placed **before the return type**. The compiler usually *infers* `T` from the arguments.

```java
public static <T> T firstElement(List<T> list) {  // <T> before return type
    return list.get(0);
}
```

```java
List<String> words = List.of("apple", "banana");
String w = Utils.firstElement(words);  // compiler infers T = String
```

| | Generic Class | Generic Method |
|---|---|---|
| Where `<T>` is declared | After class name: `class Box<T>` | Before return type: `<T> T m(...)` |
| Scope of `T` | The whole class | Just that method |
| Type chosen when | You create the object | You call the method (usually inferred) |

> A generic method can live inside a non-generic class (e.g. `Collections.sort`).

---

## Multiple Type Parameters

A class or method can take more than one type parameter, separated by commas — like a key-value `Pair`:

```java
public class Pair<K, V> {       // two type parameters
    private final K key;
    private final V value;
    public Pair(K key, V value) { this.key = key; this.value = value; }
    public K getKey()   { return key; }
    public V getValue() { return value; }
}

Pair<String, Integer> age = new Pair<>("Alice", 30);  // K=String, V=Integer
```

This is exactly how `Map<K, V>` works — each entry pairs a key type with a value type.

---

## Bounded Type Parameters

By default `T` can be any type. A **bound** restricts it using `extends`, which unlocks that type's methods.

```java
// WITHOUT a bound — T could be anything, so number methods aren't allowed
public static <T> double sumBad(List<T> list) {
    double total = 0;
    for (T item : list) total += item.doubleValue(); // COMPILE ERROR — T might be String
    return total;
}

// WITH a bound — T must be a Number, so doubleValue() is allowed
public static <T extends Number> double sum(List<T> list) {
    double total = 0;
    for (T item : list) total += item.doubleValue(); // OK
    return total;
}
```

```java
sum(List.of(1, 2, 3));    // T = Integer — OK
sum(List.of("a", "b"));   // COMPILE ERROR — String is not a Number
```

In generics, `extends` means "is a subtype of" — it covers both classes and interfaces. You always write `extends`, never `implements`, inside `<>`.

| Bound | Meaning |
|---|---|
| `<T>` | Any type |
| `<T extends Number>` | Number or a subclass |
| `<T extends Comparable<T>>` | Must implement `Comparable` (can call `compareTo`) |
| `<T extends Animal & Serializable>` | Multiple bounds — **class first**, then interfaces |

---

## Wildcards: ?, extends, super

A **wildcard** `?` represents an **unknown type**. Use wildcards in method parameters to accept many generic types, not just one.

**1. Unbounded `<?>`** — any unknown type. You can only read as `Object` and cannot add (except `null`):

```java
public static void printSize(List<?> list) {
    System.out.println("Size: " + list.size());  // OK — doesn't depend on element type
    // list.add("x");  // COMPILE ERROR — don't know the real type
}
```

**2. Upper bound `<? extends T>`** — T or any subtype. You can **READ** as `T`, but **cannot add**:

```java
public static double sumList(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();  // safe to read as Number
    return total;
    // list.add(5);  // COMPILE ERROR
}
```

**3. Lower bound `<? super T>`** — T or any supertype. You can **ADD** T, but reading only gives `Object`:

```java
public static void addNumbers(List<? super Integer> list) {
    list.add(1);   // safe to add an Integer
    Object o = list.get(0);  // reading only gives Object
}
```

| Wildcard | Meaning | READ? | ADD? | Use when |
|---|---|---|---|---|
| `<?>` | Any unknown type | As `Object` | No (except `null`) | You don't care about the type |
| `<? extends T>` | T or a **sub**type | Yes, as `T` | **No** | You **read** values |
| `<? super T>` | T or a **super**type | As `Object` | **Yes** | You **write** values |

> **Memory trick**: `extends` = "get stuff out" (read). `super` = "put stuff in" (write).

---

## The PECS Rule

**PECS = Producer Extends, Consumer Super.** The most asked generics question — it tells you *which* wildcard to use.

- **Producer Extends**: if a parameter produces values (you read FROM it), use `<? extends T>`.
- **Consumer Super**: if a parameter consumes values (you write INTO it), use `<? super T>`.
- **Both** read and write → use an exact type `T` (no wildcard).

Real JDK example — `Collections.copy`:

```java
// src PRODUCES (read from) → extends ; dest CONSUMES (write to) → super
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++) {
        T item = src.get(i);  // read from producer
        dest.add(item);       // write into consumer
    }
}
```

```java
List<Integer> source = List.of(1, 2, 3);   // producer
List<Number> target = new ArrayList<>();   // consumer (Number is a supertype of Integer)
copy(target, source);  // works
```

> **Interview gold**: `Collections.copy`, `Collections.max(Collection<? extends T>)`, and `Stream.forEach(Consumer<? super T>)` are all real PECS examples.

---

## Type Erasure

**Type erasure** is how Java implements generics. The compiler checks generic types at **compile time**, then **removes** the type info — at **runtime** the generic type is gone, replaced by `Object` (or the bound).

```java
// What you write:
List<String> list = new ArrayList<>();
String s = list.get(0);

// After erasure (roughly):
List list = new ArrayList();
String s = (String) list.get(0);   // compiler inserts the cast for you
```

**Why?** Generics were added in Java 5 but had to stay **backward compatible** — old pre-generics code and new generic code run on the same JVM. So generics are a compile-time-only feature; no JVM changes were needed.

**Runtime consequences (interview favorites):**

| What you CAN'T do | Why | Workaround |
|---|---|---|
| `new T()` | T unknown at runtime | Pass `Supplier<T>` or `Class<T>` |
| `new T[10]` | Array type unknown | `(T[]) new Object[10]` (unchecked) |
| `obj instanceof List<String>` | `<String>` erased | Use `instanceof List<?>` |
| Overload by generic type only | Same erased signature | Rename one method |

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass());  // TRUE — both are just ArrayList.class
```

> **Interview tip**: "Generics are compile-time only. Thanks to type erasure, `List<String>` and `List<Integer>` are both just `List` at runtime."

---

## Generics & Inheritance (Invariance)

The concept that trips up everyone: **`List<String>` is NOT a subtype of `List<Object>`**, even though `String` IS a subtype of `Object`. Generics are **invariant**.

```java
List<String> strings = new ArrayList<>();
List<Object> objects = strings;  // COMPILE ERROR — blocked on purpose
// If it were allowed:
objects.add(42);                 // would put an Integer into a List<String>!
String s = strings.get(1);       // would explode
```

**Contrast with arrays** — arrays are **covariant**, which is a known flaw:

```java
Object[] array = new String[3];  // allowed (covariant)
array[0] = 42;                   // compiles, but throws ArrayStoreException AT RUNTIME

List<Object> list = new ArrayList<String>();  // COMPILE ERROR — caught early, safer
```

| Feature | Relationship | Error caught at |
|---|---|---|
| **Arrays** | Covariant: `String[]` IS an `Object[]` | **Runtime** (`ArrayStoreException`) |
| **Generics** | Invariant: `List<String>` is NOT a `List<Object>` | **Compile time** (safe) |

Wildcards restore flexibility safely:

```java
void printAll(List<?> list) { }     // accepts any element type
printAll(new ArrayList<String>());  // OK
printAll(new ArrayList<Integer>()); // OK
```

> **Interview soundbite**: "Generics are invariant to preserve type safety. Wildcards (`? extends`, `? super`) restore controlled flexibility."

---

## Common Gotchas

```java
// 1. Raw types — drops all type safety
List list = new ArrayList();   // always use List<String> or List<?>

// 2. Can't add to <? extends T>
List<? extends Number> nums = new ArrayList<Integer>();
nums.add(1);   // COMPILE ERROR — extends is for reading (PECS)

// 3. No type info at runtime
T t = new T();                    // ERROR — type erased
if (list instanceof List<String>) // ERROR — use instanceof List<?>

// 4. Forgetting a bound, then can't call methods
public <T> T biggest(List<T> list) {
    list.get(0).compareTo(list.get(1));  // ERROR — T has no compareTo()
}
// FIX: <T extends Comparable<T>> T biggest(List<T> list)
```

| Mistake | Fix |
|---|---|
| Raw type `List` | Use `List<String>` or `List<?>` |
| `add` to `<? extends T>` | Use `<? super T>` to add |
| `new T()` / `new T[]` | Pass `Class<T>` or `Supplier<T>` |
| `instanceof List<String>` | Use `instanceof List<?>` |
| No bound but calling methods | Add `<T extends SomeType>` |

> **`<T>` vs `<?>`**: declare `<T>` when you need to name and reuse the type (e.g. in the return). Use `<?>` when you just accept "any type" and don't name it.

---

## Generics in Spring / JPA

### 1. `JpaRepository<T, ID>` — entity type + primary-key type

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // T = User, ID = Long → all inherited methods are typed for User:
    //   Optional<User> findById(Long id);  ← no casting
    Optional<User> findByEmail(String email);
}
```

Generics let Spring define `JpaRepository<T, ID>` once; every repository is automatically type-safe for its entity.

### 2. `ResponseEntity<T>` — typed HTTP responses

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return ResponseEntity.ok(userService.findById(id));  // body type checked at compile time
}
```

### 3. `Optional<T>` — a typed "maybe" container

```java
Optional<User> maybeUser = userRepository.findByEmail("a@b.com");
User user = maybeUser.orElseThrow(() -> new NotFoundException());
```

| Spring/JPA Type | What the type params mean |
|---|---|
| `JpaRepository<T, ID>` | Entity type, primary-key type |
| `ResponseEntity<T>` | The HTTP response body type |
| `Optional<T>` | A value that may or may not be present |
| `List<T>` / `Page<T>` | The element type in the collection/page |

---

## Common Interview Questions

**Q: Why were generics added?**
Compile-time type safety and no manual casting. Before generics, collections held `Object`, so the wrong type could be added and you got a `ClassCastException` at runtime.

**Q: Generic class vs generic method?**
A generic class declares `<T>` after the class name; the type is chosen when you create an object. A generic method declares `<T>` before the return type; the type is usually inferred when you call it. A generic method can live in a non-generic class.

**Q: What is a bounded type parameter?**
A restriction using `extends`. `<T extends Number>` means `T` must be `Number` or a subclass, so you can call `Number` methods. Multiple bounds use `&` (class first): `<T extends Animal & Serializable>`.

**Q: Explain the three wildcards.**
`<?>` — any type, read as `Object`, can't add. `<? extends T>` — T or subtype, read as `T`, can't add. `<? super T>` — T or supertype, can add T, read only as `Object`.

**Q: What is PECS?**
Producer Extends, Consumer Super. Read from a parameter → `<? extends T>`. Write into it → `<? super T>`. Both → exact `T`. Example: `Collections.copy(List<? super T> dest, List<? extends T> src)`.

**Q: Why can't you add to `List<? extends Number>`?**
It could be a `List<Integer>`, `List<Double>`, etc. — the compiler doesn't know the exact type, so adding could break type safety. You can only read (as `Number`) or add `null`.

**Q: What is type erasure?**
The compiler checks generic types, then removes them so bytecode contains only raw types (`List`, not `List<String>`), with `T` replaced by `Object` or its bound. It exists for backward compatibility with pre-Java-5 code.

**Q: Name things you can't do because of erasure.**
`new T()`, `new T[10]`, `instanceof List<String>`, overloading by generic type only, and reading the actual generic type at runtime.

**Q: Is `List<String>` a subtype of `List<Object>`?**
No — generics are invariant. If it were, you could assign it to a `List<Object>` reference and add an `Integer`, corrupting the list. Java blocks this at compile time.

**Q: How do arrays and generics differ on inheritance?**
Arrays are covariant (`String[]` IS an `Object[]`) — the wrong type throws `ArrayStoreException` at runtime. Generics are invariant — the same mistake is caught at compile time.

**Q: How do generics appear in Spring Data JPA?**
`JpaRepository<T, ID>` — `T` is the entity, `ID` is the primary-key type. `extends JpaRepository<User, Long>` makes all inherited methods type-safe for `User`, so you never cast results.

---

## Quick Reference Cheat Sheet

```
WHY GENERICS:
  → compile-time type safety   → no manual casting (no ClassCastException surprises)

DECLARING:
  Generic class   → class Box<T> { T value; }
  Generic method  → <T> T first(List<T> l)     ← <T> goes BEFORE return type
  Multiple params → class Pair<K, V>           ← e.g. Map<K, V>
  Names: T=Type  E=Element  K=Key  V=Value  N=Number  R=Return

BOUNDS:
  <T>                          → any type
  <T extends Number>           → Number or subclass (can call Number methods)
  <T extends Comparable<T>>    → can call compareTo()
  <T extends Animal & Serializable> → multiple bounds (class FIRST)

WILDCARDS:
  <?>             → unknown type; read as Object; CANNOT add (except null)
  <? extends T>   → T or subtype; READ as T; CANNOT add        ← PRODUCER
  <? super T>     → T or supertype; ADD T; read only as Object ← CONSUMER

PECS (Producer Extends, Consumer Super):
  read FROM it (source)       → ? extends T
  write INTO it (destination) → ? super T
  both read AND write         → exact T (no wildcard)
  JDK: Collections.copy(List<? super T> dest, List<? extends T> src)

TYPE ERASURE (compile-time only):
  At runtime, List<String> and List<Integer> are BOTH just List
  CANNOT: new T() | new T[10] | instanceof List<String> | overload by generic type
  WHY: backward compatibility with pre-Java-5 code

INVARIANCE:
  List<String> is NOT a List<Object>   ← generics INVARIANT (safe, compile-time)
  String[]     IS  an  Object[]        ← arrays COVARIANT (unsafe, runtime error)

REAL SPRING / JPA:
  JpaRepository<T, ID>  → T = entity, ID = primary-key type
  ResponseEntity<T>     → typed HTTP response body
  Optional<T>           → maybe-a-value container
  List<T> / Page<T>     → typed collections

TOP GOTCHAS:
  raw type List            → always use List<String> or List<?>
  add to <? extends T>     → not allowed; use <? super T> to add
  new T() / new T[]        → pass Class<T> or Supplier<T>
  instanceof List<String>  → use instanceof List<?>
  <T> vs <?>               → <T> declares & names a type; <?> = "any unknown type"
```

---

*Last Updated: 2026-06-18*
