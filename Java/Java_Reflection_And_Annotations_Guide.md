# Java Annotations & Reflection — Interview Study Guide

## Overview

Annotations and Reflection are the "magic" behind almost every Java framework. Spring's `@Autowired`, JUnit's `@Test`, Jackson's JSON mapping, and Hibernate's `@Entity` all work because of **annotations** (metadata attached to code) combined with **reflection** (inspecting and manipulating code at runtime).

Interviewers love this topic because understanding it proves you know *how frameworks actually work* instead of treating them as black boxes.

---

## Table of Contents

1. [What Are Annotations?](#what-are-annotations)
2. [Built-in Annotations](#built-in-annotations)
3. [Meta-Annotations](#meta-annotations)
4. [Writing a Custom Annotation](#writing-a-custom-annotation)
5. [What Is Reflection?](#what-is-reflection)
6. [Getting a Class Object](#getting-a-class-object)
7. [Inspecting Fields, Methods & Constructors](#inspecting-fields-methods--constructors)
8. [Creating Instances & Invoking Methods](#creating-instances--invoking-methods)
9. [setAccessible & Accessing Private Members](#setaccessible--accessing-private-members)
10. [Reading Annotations at Runtime (Full Worked Example)](#reading-annotations-at-runtime-full-worked-example)
11. [How Real Frameworks Use This (Spring & JUnit)](#how-real-frameworks-use-this-spring--junit)
12. [Downsides of Reflection](#downsides-of-reflection)
13. [Common Mistakes & Pitfalls](#common-mistakes--pitfalls)
14. [Common Interview Questions](#common-interview-questions)
15. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## What Are Annotations?

An **annotation** is **metadata** — a label attached to your code (classes, methods, fields, parameters). Annotations do **nothing** by themselves. Something else (the compiler, or a framework using reflection) reads those labels and acts on them.

```java
@Override                          // label: "this overrides a parent method"
public String toString() {         // the compiler READS it and verifies it's true
    return "Hello";
}
```

**Key idea: annotation = data, not behavior.** The behavior lives in whoever reads the annotation.

| Annotation | Read by | When |
|---|---|---|
| `@Override` | Compiler | Compile time |
| `@Entity` | Hibernate | Runtime via reflection |
| `@Test` | JUnit | Runtime via reflection |
| `@Autowired` | Spring | Runtime via reflection |

---

## Built-in Annotations

```java
@Override                        // compile error if you're not actually overriding
public String toString() { return "example"; }

@Deprecated                      // compiler warning when callers use this method
public void oldMethod() { }

@SuppressWarnings("unchecked")   // silence a specific compiler warning
public void riskyCast() { }

@FunctionalInterface             // compile error if interface has more than one abstract method
interface Calculator {
    int calculate(int a, int b);
}
```

| Annotation | Purpose |
|---|---|
| `@Override` | Verify you really override a method |
| `@Deprecated` | Warn that an API is outdated |
| `@SuppressWarnings` | Silence specific compiler warnings |
| `@FunctionalInterface` | Enforce single-abstract-method (for lambdas) |

> **Interview Tip**: All four are read by the **compiler**, not at runtime. Compare to framework annotations like `@Entity` or `@Autowired`, which are read at **runtime** via reflection.

---

## Meta-Annotations

A **meta-annotation** is an annotation you put on *another annotation* to configure how it behaves. There are four you must know.

### @Retention — how long the annotation survives

```java
@Retention(RetentionPolicy.SOURCE)   // discarded by the compiler
@Retention(RetentionPolicy.CLASS)    // kept in .class, NOT accessible at runtime (default)
@Retention(RetentionPolicy.RUNTIME)  // kept and readable via reflection at runtime
```

| Policy | In `.class`? | Readable at runtime? | Example |
|---|---|---|---|
| `SOURCE` | No | No | `@Override` |
| `CLASS` | Yes | No | (default; rarely used directly) |
| `RUNTIME` | Yes | **Yes** | `@Entity`, `@Test`, `@Autowired` |

> **Critical**: If you want a framework to read your annotation at runtime, it MUST be `@Retention(RUNTIME)`. Forgetting this is the #1 mistake with custom annotations — `getAnnotation()` returns `null`.

### @Target — where the annotation can be placed

```java
@Target(ElementType.METHOD)
@Target(ElementType.FIELD)
@Target(ElementType.TYPE)  // classes, interfaces, enums
@Target({ElementType.METHOD, ElementType.FIELD})
```

Misuse causes a **compile error** — it's a safety net.

### @Inherited — subclasses inherit the annotation (classes only)

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface Auditable { }

@Auditable
class BaseEntity { }

class User extends BaseEntity { }  // User is also considered @Auditable automatically
```

### @Documented — include the annotation in generated Javadoc

Purely cosmetic — no behavioral effect.

---

## Writing a Custom Annotation

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)   // MUST be RUNTIME for reflection to read it
@Target(ElementType.METHOD)           // only allowed on methods
@Documented
public @interface LogExecution {
    String value() default "INFO";    // element with a default
    boolean logArgs() default false;
    int timeoutMs() default 1000;
}
```

Usage:

```java
@LogExecution(value = "DEBUG", logArgs = true, timeoutMs = 500)
public void processOrder() { }

@LogExecution("WARN")   // shortcut: if only setting "value", you can drop "value ="
public void cancelOrder() { }

@LogExecution           // all defaults
public void refundOrder() { }
```

| Rule | Detail |
|---|---|
| Allowed element types | primitives, `String`, `Class`, enums, other annotations, arrays of these |
| `default` | Makes the element optional for users |
| `value` is special | If it's the only element being set, you can omit `value =` |
| No `null` defaults | Use `""` or a sentinel value instead |

---

## What Is Reflection?

**Reflection** is Java's ability to **inspect and manipulate classes, fields, methods, and annotations at runtime** — even for types unknown at compile time.

Think of it like an X-ray machine: normally you only see an object's public surface. Reflection lets a program look inside any object, see every field and method (even private ones), and operate on them.

### Why do frameworks rely on reflection?

A framework like Spring or Jackson is written before your classes exist. Jackson has no idea your `User` has a `firstName` field — so it uses reflection to ask your object at runtime "what fields do you have?", reads each value, and builds the JSON.

| Framework | What reflection + annotations enable |
|---|---|
| **Spring** | Find `@Component` classes, inject `@Autowired`, map `@GetMapping` routes |
| **JUnit** | Scan for `@Test` methods and invoke each |
| **Jackson** | Read fields to serialize/deserialize objects |
| **Hibernate** | Read `@Entity`/`@Column` to map objects to database tables |

---

## Getting a Class Object

Everything in reflection starts with a `Class` object.

```java
// Known at compile time
Class<User> c1 = User.class;

// From an existing object (returns the ACTUAL runtime type)
User user = new User();
Class<?> c2 = user.getClass();

// From a class name as a String (decided at runtime — used by frameworks)
Class<?> c3 = Class.forName("com.example.User");  // throws ClassNotFoundException
```

| Method | When to use |
|---|---|
| `MyClass.class` | Class is known at compile time |
| `obj.getClass()` | You have an instance; want its actual runtime type |
| `Class.forName("...")` | Class name comes from config/plugin at runtime |

> **Tip**: `getClass()` returns the **actual runtime type**. If `Object o = new ArrayList<>()`, then `o.getClass()` returns `ArrayList.class`.

---

## Inspecting Fields, Methods & Constructors

```java
Class<?> clazz = User.class;

// Fields
Field[] allFields   = clazz.getDeclaredFields();              // ALL fields of this class (incl. private), NOT inherited
Field[] publicFields = clazz.getFields();                     // public fields only, INCLUDING inherited
Field nameField     = clazz.getDeclaredField("name");         // specific field by name

// Methods
Method[] methods = clazz.getDeclaredMethods();
Method m         = clazz.getDeclaredMethod("setName", String.class); // must specify param types

// Constructors
Constructor<?> noArgs   = clazz.getDeclaredConstructor();
Constructor<?> withArgs = clazz.getDeclaredConstructor(String.class, int.class);
```

### getDeclaredXxx() vs getXxx() — a frequent interview trap

| Method family | Includes private/protected? | Includes inherited? |
|---|---|---|
| `getFields()` / `getMethods()` | No (public only) | Yes |
| `getDeclaredFields()` / `getDeclaredMethods()` | Yes (all visibilities) | No |

**Memory hook**: "**Declared** = declared right here in this class, all visibilities."

---

## Creating Instances & Invoking Methods

```java
Class<?> clazz = Class.forName("com.example.User");

// Create an instance
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class, int.class);
Object u = ctor.newInstance("Alice", 30);   // same as: new User("Alice", 30)

// Invoke a method
Method setName = clazz.getDeclaredMethod("setName", String.class);
setName.invoke(u, "Bob");                   // same as: u.setName("Bob")

// Static method: pass null as the object
Method create = clazz.getDeclaredMethod("create");
Object result = create.invoke(null);

// Read / write a field
Field field = clazz.getDeclaredField("name");
Object value = field.get(u);               // read
field.set(u, "Charlie");                   // write
```

> **Tip**: `method.invoke()` returns `Object` (or `null` for `void`). If the invoked method throws, reflection wraps it in `InvocationTargetException` — call `.getCause()` to get the real exception.

---

## setAccessible & Accessing Private Members

By default, reflection respects access rules. Override with `setAccessible(true)`.

```java
BankAccount account = new BankAccount();
Field balanceField = BankAccount.class.getDeclaredField("balance");
balanceField.setAccessible(true);    // bypass the private check
double bal = (double) balanceField.get(account);
balanceField.set(account, 999.0);
```

Same applies to private methods and constructors.

> **Caution**: This breaks encapsulation deliberately. It's how Hibernate populates entities and Spring injects into private `@Autowired` fields. In your own business code, it's a red flag — it makes code fragile. On Java 9+ with the module system, `setAccessible(true)` may throw `InaccessibleObjectException`.

---

## Reading Annotations at Runtime (Full Worked Example)

This ties annotations + reflection together — exactly what Jackson does internally.

**Step 1: Define the annotation**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface JsonField {
    String name() default "";   // custom JSON key; empty = use field name
}
```

**Step 2: Annotate a class**

```java
public class Person {
    @JsonField
    private String firstName = "Ada";

    @JsonField(name = "surname")
    private String lastName = "Lovelace";

    @JsonField
    private int age = 36;

    private String password = "secret";  // no @JsonField → skipped
}
```

**Step 3: The serializer (reflection + annotation engine)**

```java
public class SimpleJsonSerializer {
    public static String serialize(Object obj) throws IllegalAccessException {
        Class<?> clazz = obj.getClass();
        StringBuilder json = new StringBuilder("{");
        boolean first = true;

        for (Field field : clazz.getDeclaredFields()) {
            if (!field.isAnnotationPresent(JsonField.class)) continue;

            JsonField annotation = field.getAnnotation(JsonField.class);
            String key = annotation.name().isEmpty() ? field.getName() : annotation.name();

            field.setAccessible(true);
            Object value = field.get(obj);

            if (!first) json.append(", ");
            json.append("\"").append(key).append("\": \"").append(value).append("\"");
            first = false;
        }

        return json.append("}").toString();
    }
}
```

**Step 4: Output**

```java
// {"firstName": "Ada", "surname": "Lovelace", "age": "36"}
// "password" is absent — no @JsonField. "lastName" became "surname" from the annotation element.
```

**The big picture**: The engine works for any class (`Person`, `Order`, `Product`) — it's driven by annotations + reflection, not hard-coded field names. This is exactly what Jackson does with `@JsonProperty`, Hibernate with `@Column`, and Spring with `@Value`.

---

## How Real Frameworks Use This (Spring & JUnit)

### Spring Dependency Injection (`@Autowired`)

```java
@Service
public class OrderService {
    @Autowired
    private PaymentGateway gateway;  // no setter, no constructor
}
```

Conceptually, Spring does:
1. Scan classpath for `@Component`/`@Service` classes → `Class.forName()` + reflection
2. Create instances → `constructor.newInstance()`
3. Find `@Autowired` fields → `getDeclaredFields()` + `isAnnotationPresent()`
4. Inject: `field.setAccessible(true); field.set(bean, dependency)`

### JUnit Test Discovery (`@Test`)

JUnit does:
1. `clazz.getDeclaredMethods()` — find all methods
2. `method.isAnnotationPresent(Test.class)` — skip non-test methods
3. `method.invoke(testInstance)` — run each test, record pass/fail

> **Interview gold**: "How does Spring inject into a private field?" / "How does JUnit find tests?" — the answer is always: **reflection to discover members + annotations as the markers.**

---

## Downsides of Reflection

| Downside | Explanation |
|---|---|
| **Performance cost** | Slower than direct calls; JVM can't inline/optimize. Fine for startup; avoid in hot loops. |
| **Breaks encapsulation** | `setAccessible(true)` bypasses `private`; you can corrupt invariants the author relied on. |
| **No compile-time safety** | Members referenced by `String` name — typos compile fine and blow up at runtime. |
| **Harder to debug** | Dynamic code is hard to follow; IDE "find usages" won't find reflective calls. |
| **Module restrictions** | Java 9+ module system can block `setAccessible`, throwing `InaccessibleObjectException`. |

> **Rule of thumb**: Let frameworks use reflection. In your own code, prefer normal method calls and interfaces. Use reflection only when types genuinely aren't known at compile time.

---

## Common Mistakes & Pitfalls

```java
// MISTAKE 1: Forgetting @Retention(RUNTIME)
@Target(ElementType.FIELD)
@interface MyAnnotation { }  // defaults to CLASS — getAnnotation() returns NULL at runtime
// FIX: add @Retention(RetentionPolicy.RUNTIME)

// MISTAKE 2: getFields() instead of getDeclaredFields()
clazz.getFields();          // misses private fields!
clazz.getDeclaredFields();  // gets all fields of this class

// MISTAKE 3: Forgetting setAccessible(true) on private members
Field f = clazz.getDeclaredField("secret");
f.get(instance);            // IllegalAccessException
// FIX: f.setAccessible(true) first

// MISTAKE 4: Primitive vs wrapper in getDeclaredMethod
clazz.getDeclaredMethod("setAge", Integer.class); // NoSuchMethodException if method uses int
// FIX: use int.class for primitives

// MISTAKE 5: Not unwrapping InvocationTargetException
method.invoke(obj);  // real exception is wrapped — call .getCause() to see it
```

| Pitfall | Quick fix |
|---|---|
| `getAnnotation()` returns `null` | Add `@Retention(RetentionPolicy.RUNTIME)` |
| Can't see private fields | Use `getDeclaredFields()` |
| `IllegalAccessException` | Call `setAccessible(true)` first |
| `NoSuchMethodException` on primitive param | Use `int.class`, not `Integer.class` |
| Real exception hidden | Unwrap with `InvocationTargetException.getCause()` |

---

## Common Interview Questions

### Q: What is an annotation? Does it change how your code runs?

An annotation is metadata attached to code. It does nothing on its own — behavior comes from whoever reads it: the compiler (e.g., `@Override`) or a framework using reflection at runtime (e.g., `@Test`, `@Entity`). It changes behavior only indirectly, through its reader.

---

### Q: What's the difference between the three RetentionPolicy values?

`SOURCE` is discarded by the compiler. `CLASS` (the default) stays in bytecode but is not accessible to reflection at runtime. `RUNTIME` is kept and readable via reflection — required for any annotation a framework reads while the app runs.

---

### Q: I wrote a custom annotation but `getAnnotation()` returns null. Why?

You forgot `@Retention(RetentionPolicy.RUNTIME)`. The default is `CLASS`, which the JVM does not expose to reflection. Add `RUNTIME` retention.

---

### Q: What is reflection and why do frameworks need it?

Reflection lets Java inspect and manipulate classes, fields, methods, and annotations at runtime for types unknown at compile time. Frameworks are shipped before your classes exist, so they can't hard-code your field names — they use reflection to discover your structure at runtime and act on it.

---

### Q: Three ways to get a Class object?

`MyClass.class` — class known at compile time. `obj.getClass()` — actual runtime type of an instance. `Class.forName("...")` — class name as a String at runtime (throws `ClassNotFoundException`).

---

### Q: Difference between `getDeclaredFields()` and `getFields()`?

`getDeclaredFields()` returns all fields (including private) declared in this class, excluding inherited ones. `getFields()` returns only public fields, including inherited ones. For framework-style work, use `getDeclaredFields()`.

---

### Q: How can reflection access a private field? Should you do it?

Call `field.setAccessible(true)` to bypass the access check, then `field.get()`/`field.set()`. Frameworks do this (Hibernate, Spring), but in business code it breaks encapsulation and can be blocked by Java 9+ modules.

---

### Q: How does Spring inject into a private field with no setter?

At startup Spring finds `@Autowired` fields via `getDeclaredFields()`, then calls `field.setAccessible(true)` and `field.set(bean, dependency)` — the same reflection pattern as reading an annotation marker and setting a private field directly.

---

### Q: How does JUnit know which methods are tests?

It calls `getDeclaredMethods()`, checks each with `method.isAnnotationPresent(Test.class)`, skips ones without `@Test`, and calls `method.invoke(testInstance)` on the rest. Annotations are the markers; reflection does the discovery and invocation.

---

### Q: What are the downsides of reflection?

Slower than direct calls, breaks encapsulation via `setAccessible`, no compile-time safety (String names mean typos only fail at runtime), harder to debug, and Java 9+ module/security restrictions may block `setAccessible`.

---

### Q: What happens if a method invoked via reflection throws an exception?

Reflection wraps it in `InvocationTargetException`. The real exception is available via `.getCause()`. A common mistake is logging the wrapper and missing the underlying error.

---

### Q: What's a meta-annotation? Name the four.

A meta-annotation is placed on another annotation to configure it. The four: `@Retention` (how long it survives), `@Target` (where it can be placed), `@Inherited` (subclasses inherit it — classes only), `@Documented` (appears in Javadoc).

---

## Quick Reference Cheat Sheet

```
ANNOTATIONS
  Annotation = metadata (a label). Does nothing alone — a reader gives it meaning.
  Readers: compiler (@Override) OR framework via reflection (@Entity, @Test).

BUILT-IN (all read by the COMPILER):
  @Override            → verify you really override a method
  @Deprecated          → warn: API is outdated
  @SuppressWarnings    → silence a specific compiler warning
  @FunctionalInterface → enforce exactly one abstract method (for lambdas)

META-ANNOTATIONS:
  @Retention  → SOURCE  : gone after compile
                CLASS   : in .class, NOT visible to reflection (default)
                RUNTIME : visible to reflection  ← needed for frameworks
  @Target     → where allowed: TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR...
  @Inherited  → subclasses inherit it (CLASSES only)
  @Documented → show it in Javadoc

CUSTOM ANNOTATION:
  @Retention(RUNTIME) @Target(FIELD)
  public @interface JsonField { String name() default ""; }
  - elements look like methods, can have defaults
  - "value" element → can be set without "value =" if it's the only one
  - no null defaults; use "" or a sentinel

REFLECTION
  Get a Class:
    MyClass.class            → known at compile time
    obj.getClass()           → actual runtime type of an object
    Class.forName("a.b.C")   → class name as String (throws ClassNotFoundException)

  Inspect:
    getDeclaredFields()  → ALL fields of THIS class (incl. private), NOT inherited
    getFields()          → PUBLIC fields only, INCLUDING inherited
    getDeclaredMethod("name", ParamType.class)
    getDeclaredConstructor(ParamType.class)

  Create & invoke:
    constructor.newInstance(args)   → like new MyClass(args)
    method.invoke(obj, args)        → like obj.method(args); pass null obj for static
    field.get(obj) / field.set(obj, value)

  Private access:
    member.setAccessible(true)  → bypass private check (breaks encapsulation)

  Read annotations at runtime:
    field.isAnnotationPresent(JsonField.class)
    JsonField a = field.getAnnotation(JsonField.class);
    a.name();   // annotation MUST be @Retention(RUNTIME)

FRAMEWORK PATTERN:
  reflection (discover members) + annotations (markers for which members matter)
  Spring @Autowired → getDeclaredFields → setAccessible(true) → field.set(bean, dep)
  JUnit  @Test      → getDeclaredMethods → isAnnotationPresent → method.invoke

DOWNSIDES:
  slower | breaks encapsulation | no compile-time safety (String names)
  harder to debug | Java 9+ module/security may block setAccessible

COMMON BUGS:
  annotation null at runtime  → forgot @Retention(RUNTIME)
  can't see private fields    → use getDeclaredFields(), not getFields()
  IllegalAccessException      → call setAccessible(true) first
  NoSuchMethodException       → int.class != Integer.class for primitive params
  real error hidden           → unwrap InvocationTargetException.getCause()
```

---

*Last Updated: 2026-06-18*
