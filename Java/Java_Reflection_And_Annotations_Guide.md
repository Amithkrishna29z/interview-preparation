# Java Annotations & Reflection — Interview Study Guide

## Overview

Annotations and Reflection are the "magic" behind almost every Java framework you'll use as a backend developer. Spring's `@Autowired`, JUnit's `@Test`, Jackson's JSON mapping, and Hibernate's `@Entity` all work because of **annotations** (metadata you attach to code) combined with **reflection** (the ability to inspect and manipulate code at runtime).

You don't have to write reflection code often in day-to-day work, but interviewers love this topic because understanding it proves you know *how the frameworks actually work* instead of treating them as black boxes.

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

An **annotation** is **metadata** — extra information you attach to your code (classes, methods, fields, parameters). Annotations do **nothing** by themselves. They are just labels. Something else (the compiler, or a framework using reflection) reads those labels and decides to do something because of them.

**Think of it like sticky notes on documents:**
A sticky note that says "URGENT" on a folder doesn't *make* anything happen on its own. But a person (or a program) who reads the folder sees the note and reacts to it — maybe they process it first. The note is metadata; the *reader* gives it meaning.

```java
@Override                          // a sticky note that says "this overrides a parent method"
public String toString() {         // the compiler READS that note and verifies it's true
    return "Hello";
}
```

Key idea: **annotation = data, not behavior.** The behavior lives in whoever reads the annotation.

| Annotation looks like... | It really is... |
|---|---|
| `@Override` | A flag the **compiler** reads at compile time |
| `@Entity` | A flag **Hibernate** reads at runtime via reflection |
| `@Test` | A flag **JUnit** reads at runtime to find test methods |
| `@Autowired` | A flag **Spring** reads at runtime to inject dependencies |

---

## Built-in Annotations

Java ships with several annotations in `java.lang`. These are the ones interviewers expect you to recognize instantly.

```java
public class Examples {

    @Override
    // Tells the COMPILER: "I'm overriding a method from my parent/interface."
    // If you misspell the method name or the signature doesn't match,
    // compilation FAILS. This catches typo bugs early.
    public String toString() {
        return "example";
    }

    @Deprecated
    // Marks this method as outdated. Anyone who calls it gets a compiler WARNING.
    // It still works — it's just a polite "please stop using this, use the new one."
    public void oldMethod() { }

    @SuppressWarnings("unchecked")
    // Tells the compiler: "I know there's a warning here, hide it — I accept the risk."
    // "unchecked" silences unchecked-generics warnings. Use sparingly and intentionally.
    public void riskyCast() {
        List rawList = new ArrayList();
        List<String> typed = rawList; // would normally warn; suppressed here
    }
}

@FunctionalInterface
// Tells the COMPILER: "this interface must have EXACTLY ONE abstract method."
// If someone adds a second abstract method, compilation FAILS.
// This protects lambdas — a lambda can only target a single-method interface.
interface Calculator {
    int calculate(int a, int b);   // exactly one abstract method — OK
    // int extra();                // adding this would break compilation
}
```

| Annotation | Read by | When | Purpose |
|---|---|---|---|
| `@Override` | Compiler | Compile time | Verify you really are overriding a method |
| `@Deprecated` | Compiler | Compile time | Warn that an API is outdated |
| `@SuppressWarnings` | Compiler | Compile time | Silence specific compiler warnings |
| `@FunctionalInterface` | Compiler | Compile time | Enforce a single-abstract-method interface (for lambdas) |

> **Interview Tip**: All four built-in annotations above are read by the **compiler**, not at runtime. They exist to catch mistakes early. Compare this to framework annotations like `@Entity` or `@Autowired`, which are read at **runtime** using reflection.

---

## Meta-Annotations

A **meta-annotation** is an annotation that you put on *another annotation* to describe how it behaves. When you write your own custom annotation, you configure it using meta-annotations.

**Think of it like the settings on a sticky note:**
Before you hand out sticky notes, you decide their rules: "These notes are visible only while writing (then thrown away)" or "These stay attached forever and visible to anyone reading the document." Meta-annotations are those rules.

There are four you must know: `@Retention`, `@Target`, `@Inherited`, `@Documented`.

### @Retention — how long the annotation survives

```java
@Retention(RetentionPolicy.SOURCE)   // discarded by the compiler; never in the .class file
@Retention(RetentionPolicy.CLASS)    // kept in the .class file, but NOT loaded at runtime (default)
@Retention(RetentionPolicy.RUNTIME)  // kept and readable at runtime via reflection
```

| Retention Policy | Survives in `.class` file? | Readable by reflection at runtime? | Example use |
|---|---|---|---|
| `SOURCE` | No | No | `@Override`, `@SuppressWarnings` (compiler-only) |
| `CLASS` | Yes | No | Bytecode tools, rarely used directly (this is the default) |
| `RUNTIME` | Yes | **Yes** | `@Entity`, `@Test`, `@Autowired` — anything a framework reads at runtime |

> **Critical interview point**: If you want a framework (or your own code) to read an annotation **at runtime via reflection**, it MUST be `@Retention(RetentionPolicy.RUNTIME)`. This is the single most common mistake when writing custom annotations — people forget it, then wonder why reflection returns `null`.

### @Target — where the annotation is allowed to go

```java
@Target(ElementType.METHOD)   // can only be placed on methods
@Target(ElementType.FIELD)    // can only be placed on fields
@Target(ElementType.TYPE)     // can only be placed on classes/interfaces/enums
@Target({ElementType.METHOD, ElementType.FIELD}) // allowed on both methods and fields
```

| ElementType | Annotation can be placed on |
|---|---|
| `TYPE` | Class, interface, enum |
| `FIELD` | Fields (instance variables) |
| `METHOD` | Methods |
| `PARAMETER` | Method parameters |
| `CONSTRUCTOR` | Constructors |
| `LOCAL_VARIABLE` | Local variables |
| `ANNOTATION_TYPE` | Other annotations (meta-annotations) |

If you try to put an annotation somewhere its `@Target` doesn't allow, compilation fails. This is a safety net.

### @Inherited — child classes inherit the annotation

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@interface Auditable { }

@Auditable
class BaseEntity { }

class User extends BaseEntity { }
// Because @Auditable is @Inherited, User is ALSO considered @Auditable —
// even though we never wrote @Auditable directly on User.
// WITHOUT @Inherited, User would NOT be treated as annotated.
```

Note: `@Inherited` only works for annotations on **classes**, not on methods or fields.

### @Documented — include the annotation in generated Javadoc

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@interface ApiVersion { }
// When you run javadoc, @ApiVersion will appear in the generated HTML docs.
// Purely cosmetic — affects documentation only, not behavior.
```

---

## Writing a Custom Annotation

Now let's combine the meta-annotations to build our own annotation from scratch.

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)   // MUST be RUNTIME so reflection can read it later
@Target(ElementType.METHOD)           // this annotation can only be placed on methods
@Documented                           // show it in Javadoc
public @interface LogExecution {      // note: "@interface" — that's how you DECLARE an annotation

    // These are called "elements" — they look like methods but act like configurable attributes.

    String value() default "INFO";    // an element named "value" with a default of "INFO"

    boolean logArgs() default false;  // another element, defaults to false

    int timeoutMs() default 1000;     // numeric element with a default
}
```

How you'd use it:

```java
// Using all elements explicitly:
@LogExecution(value = "DEBUG", logArgs = true, timeoutMs = 500)
public void processOrder() { }

// Using defaults (logArgs=false, timeoutMs=1000):
@LogExecution("WARN")    // shortcut: if you only set the element named "value",
                         // you can drop "value =" entirely
public void cancelOrder() { }

// Using ALL defaults:
@LogExecution            // value="INFO", logArgs=false, timeoutMs=1000
public void refundOrder() { }
```

### Rules for annotation elements

| Rule | Explanation |
|---|---|
| Allowed element types | primitives, `String`, `Class`, enums, other annotations, and arrays of these |
| `default` | Optional. If provided, the user can omit that element |
| `value` is special | If you name an element `value` and it's the only one being set, you can write `@LogExecution("DEBUG")` without `value =` |
| No `null` defaults | You cannot default an element to `null`. Use empty string `""` or a sentinel value instead |
| No method body | Elements are declarations only — no logic inside |

> **Think of it like a form template**: Each element is a blank on the form. `default` is the pre-filled value. The person filling out the form (the developer using your annotation) can accept the default or override it.

---

## What Is Reflection?

**Reflection** is Java's ability to **inspect and manipulate code at runtime** — examine a class's fields, methods, and annotations, create objects, and call methods, all *without knowing the class names when you wrote the code*.

Normally you write `new User()` because you know about `User` at compile time. Reflection lets you do the equivalent of `new <whatever class name is in this String>()` — decided while the program is running.

**Think of it like an X-ray machine for objects:**
Normally you only see the outside of an object — the public buttons you're allowed to press. Reflection is an X-ray: it lets a program look *inside* any object, see every field and method (even private ones), and operate on them, even if it had never heard of that class until runtime.

### Why do frameworks rely on reflection?

A framework like Spring or Jackson is written and shipped **before your classes exist**. The Jackson library has no idea your `User` class will have a `firstName` field. So how does it convert your `User` to JSON?

It uses reflection: at runtime, it asks your object "what fields do you have?", reads each value, and builds the JSON. This is why frameworks can work with classes the framework authors never saw.

| Framework | What it uses reflection + annotations for |
|---|---|
| **Spring** | Find `@Component`/`@Service` classes, inject `@Autowired` dependencies, map `@GetMapping` routes |
| **JUnit** | Scan test classes for `@Test` methods and invoke each one |
| **Jackson** | Read object fields to serialize to JSON / write fields when deserializing |
| **Hibernate** | Read `@Entity`/`@Column` to map objects to database tables and set field values from query results |

---

## Getting a Class Object

Everything in reflection starts with a `Class` object — a runtime description of a class. There are three ways to get one.

```java
// WAY 1: .class literal — when you know the class at compile time
Class<User> c1 = User.class;
// Use this when you literally type the class name in your code.

// WAY 2: getClass() — when you have an OBJECT and want its actual runtime class
User user = new User();
Class<?> c2 = user.getClass();
// Returns the REAL class of the object. If 'user' were actually an AdminUser
// subclass, this returns AdminUser.class, not User.class.

// WAY 3: Class.forName(String) — when you only have the class NAME as text
Class<?> c3 = Class.forName("com.example.User");
// The class name is a String — decided at runtime (e.g., read from a config file).
// Throws ClassNotFoundException if no such class exists on the classpath.
// This is how frameworks load classes named in config files / plugins.
```

| Method | You start with | Common use |
|---|---|---|
| `MyClass.class` | A known class name in code | Compile-time known types |
| `obj.getClass()` | An existing object | Finding an object's real runtime type |
| `Class.forName("...")` | A class name as a `String` | Loading classes named in config/plugins (frameworks) |

> **Interview Tip**: `getClass()` returns the **actual runtime type**, which matters with inheritance. If `Object o = new ArrayList<>();` then `o.getClass()` returns `ArrayList.class`, not `Object.class`.

---

## Inspecting Fields, Methods & Constructors

Once you have a `Class` object, you can examine its members.

```java
Class<?> clazz = User.class;

// ---- FIELDS ----
Field[] publicFields = clazz.getFields();
// getFields() → only PUBLIC fields, INCLUDING inherited public fields.

Field[] allDeclared = clazz.getDeclaredFields();
// getDeclaredFields() → ALL fields declared in THIS class (private, protected, public)
// but NOT inherited ones. This is the one you usually want.

Field nameField = clazz.getDeclaredField("name");
// Get one specific field by name. Throws NoSuchFieldException if missing.

// ---- METHODS ----
Method[] methods = clazz.getDeclaredMethods();
// All methods declared in this class (any visibility), not inherited ones.

Method m = clazz.getDeclaredMethod("setName", String.class);
// Get a specific method. You MUST pass the parameter types to identify the
// exact overload — here, the setName(String) version.

// ---- CONSTRUCTORS ----
Constructor<?> noArgs = clazz.getDeclaredConstructor();
// The no-argument constructor.

Constructor<?> withArgs = clazz.getDeclaredConstructor(String.class, int.class);
// The constructor taking (String, int). Again, parameter types identify the overload.
```

### `getDeclaredXxx()` vs `getXxx()` — a frequent interview trap

| Method family | Includes private/protected? | Includes inherited members? |
|---|---|---|
| `getFields()` / `getMethods()` | **No** (public only) | **Yes** |
| `getDeclaredFields()` / `getDeclaredMethods()` | **Yes** (all visibilities) | **No** (this class only) |

**Memory hook**: "**Declared** = declared *right here* (this class), all visibilities." "Plain `getFields` = public, but **inherited too**."

---

## Creating Instances & Invoking Methods

Reflection can build objects and call methods dynamically.

```java
Class<?> clazz = Class.forName("com.example.User");

// ---- CREATE AN INSTANCE ----
Constructor<?> constructor = clazz.getDeclaredConstructor(); // get the no-arg constructor
Object instance = constructor.newInstance();                 // same as calling: new User()
// Note: the old shortcut clazz.newInstance() is DEPRECATED — prefer the constructor approach.

// With arguments:
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class, int.class);
Object u = ctor.newInstance("Alice", 30);  // same as: new User("Alice", 30)

// ---- INVOKE A METHOD ----
Method setName = clazz.getDeclaredMethod("setName", String.class);
setName.invoke(instance, "Bob");
// Translates to: instance.setName("Bob")
// invoke's 1st arg = the object to call it on; remaining args = the method arguments.

// For a STATIC method, pass null as the object (no instance needed):
Method staticMethod = clazz.getDeclaredMethod("create");
Object result = staticMethod.invoke(null);  // static — no object required

// ---- READ A FIELD VALUE ----
Field field = clazz.getDeclaredField("name");
Object value = field.get(instance);  // reads instance.name — returns it as Object

// ---- WRITE A FIELD VALUE ----
field.set(instance, "Charlie");      // sets instance.name = "Charlie"
```

> **Interview Tip**: `method.invoke(obj, args...)` returns `Object` (the method's return value, or `null` for `void`). If the invoked method throws an exception, reflection wraps it in an `InvocationTargetException` — call `.getCause()` to get the real underlying exception.

---

## setAccessible & Accessing Private Members

By default, reflection respects Java's access rules — you can't read or call a `private` member. But you can override that with `setAccessible(true)`.

```java
class BankAccount {
    private double balance = 100.0;  // private — normally untouchable from outside
}

BankAccount account = new BankAccount();

Field balanceField = BankAccount.class.getDeclaredField("balance");
balanceField.setAccessible(true);   // turn OFF the access check for this field
// Without this line, the next line throws IllegalAccessException.

double bal = (double) balanceField.get(account); // reads private balance → 100.0
balanceField.set(account, 999.0);                // overwrites private balance → 999.0
```

The same works for private methods and constructors:

```java
Method secret = SomeClass.class.getDeclaredMethod("secretMethod");
secret.setAccessible(true);   // bypass the private check
secret.invoke(instance);      // now we can call the private method
```

**Think of it like a master key:**
`private` is a locked door. `setAccessible(true)` is a master key that opens it anyway. Powerful — but if you start unlocking doors the class owner deliberately locked, you might break things the owner relied on being hidden.

> **Caution**: This **breaks encapsulation** on purpose. It's how frameworks set fields without needing public setters (Hibernate populating your entity, Spring injecting into a private `@Autowired` field). But in *your own* business code, reaching into another class's privates with reflection is a red flag — it makes code fragile and bypasses the safety the class author intended. On modern Java (9+) with the module system, `setAccessible(true)` can also be blocked and throw `InaccessibleObjectException` if the module isn't "opened".

---

## Reading Annotations at Runtime (Full Worked Example)

This is the section that ties **annotations + reflection** together — and it's exactly what frameworks like Jackson do internally. We'll build a tiny "object to JSON-ish" serializer driven by a custom annotation.

### Step 1: Define the custom annotation

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)   // RUNTIME is mandatory — reflection must read it later
@Target(ElementType.FIELD)            // this annotation goes on fields
public @interface JsonField {
    String name() default "";         // optional: a custom JSON key. Empty = use the field's name.
}
```

### Step 2: Annotate a normal class

```java
public class Person {

    @JsonField                        // include in output, key = "firstName" (field name)
    private String firstName = "Ada";

    @JsonField(name = "surname")      // include in output, but use custom key "surname"
    private String lastName = "Lovelace";

    @JsonField
    private int age = 36;

    private String password = "secret"; // NO @JsonField → this field is SKIPPED (stays private/hidden)
}
```

### Step 3: Write the serializer (the reflection + annotation engine)

```java
import java.lang.reflect.Field;

public class SimpleJsonSerializer {

    public static String serialize(Object obj) throws IllegalAccessException {
        Class<?> clazz = obj.getClass();          // get the runtime class of whatever we were given
        StringBuilder json = new StringBuilder("{");

        Field[] fields = clazz.getDeclaredFields(); // grab every field declared in the class
        boolean first = true;

        for (Field field : fields) {
            // Only process fields marked with @JsonField — skip everything else (like password)
            if (!field.isAnnotationPresent(JsonField.class)) {
                continue;                            // no annotation → ignore this field
            }

            // Read the annotation instance so we can access its elements
            JsonField annotation = field.getAnnotation(JsonField.class);

            // Decide the JSON key: use annotation's name() if provided, else the field name
            String key = annotation.name().isEmpty()
                    ? field.getName()                // e.g. "firstName"
                    : annotation.name();             // e.g. "surname"

            field.setAccessible(true);               // fields are private — unlock them to read
            Object value = field.get(obj);           // read the actual value from the object

            if (!first) {
                json.append(", ");                   // comma-separate entries after the first
            }
            json.append("\"").append(key).append("\": ")
                .append("\"").append(value).append("\"");
            first = false;
        }

        json.append("}");
        return json.toString();
    }
}
```

### Step 4: Run it

```java
public class Demo {
    public static void main(String[] args) throws Exception {
        Person p = new Person();
        String result = SimpleJsonSerializer.serialize(p);
        System.out.println(result);
    }
}

// OUTPUT:
// {"firstName": "Ada", "surname": "Lovelace", "age": "36"}
//
// Notice: "password" is absent (no @JsonField), and lastName became "surname"
// because the annotation overrode the key. We never wrote ANY Person-specific
// code in the serializer — it works for ANY class with @JsonField fields.
```

**What just happened (the big picture):**

1. The annotation `@JsonField` is pure **metadata** — it does nothing on its own.
2. The serializer uses **reflection** to (a) discover the fields, (b) check which carry the annotation, (c) read the annotation's `name()` element, and (d) read each field's value.
3. This decoupling is the whole point: the engine works for `Person`, `Order`, `Product` — any class — because it's driven by annotations + reflection, not hard-coded field names.

> This is a simplified version of exactly what Jackson does with `@JsonProperty`, Hibernate does with `@Column`, and Spring does with `@Value`. Understanding this one example explains all of them.

---

## How Real Frameworks Use This (Spring & JUnit)

You won't typically write the reflection yourself, but interviewers want to know you understand *conceptually* how the magic works.

### Spring Dependency Injection (`@Autowired`)

```java
@Service
public class OrderService {

    @Autowired                       // "Spring, please put the right object here"
    private PaymentGateway gateway;  // notice: NO setter, NO constructor — it's a private field
}
```

**Conceptually, at startup Spring does roughly this:**

```
1. Scan the classpath for classes annotated @Component/@Service/@Repository (reflection + annotations).
2. Create an instance of each such class (reflection: constructor.newInstance()).
3. For each instance, scan its fields for @Autowired (reflection: getDeclaredFields()).
4. For each @Autowired field, find a matching bean by type from the container.
5. field.setAccessible(true); field.set(serviceInstance, matchingBean);  ← injection happens here
```

That last step is *exactly* the `setAccessible(true)` + `field.set(...)` pattern from our serializer example. That's why Spring can inject into a `private` field with no setter — it uses reflection to bypass access control.

### JUnit Test Discovery (`@Test`)

```java
public class CalculatorTest {

    @Test                            // marks this as a test method
    void addsTwoNumbers() {
        assertEquals(4, 2 + 2);
    }

    void helperMethod() { }          // NO @Test → JUnit ignores it
}
```

**Conceptually, JUnit does roughly this:**

```
1. Take the test class → clazz.getDeclaredMethods()  (reflection finds all methods).
2. For each method, check method.isAnnotationPresent(Test.class).
3. Skip methods WITHOUT @Test (like helperMethod).
4. For each @Test method: create a test instance, then method.invoke(instance).
5. Catch exceptions/assertion failures to report pass/fail.
```

> **Interview gold**: When asked "how does Spring inject a private field?" or "how does JUnit find test methods?", the answer is always the same shape: **reflection to discover members + annotations as the markers that tell reflection which members matter.**

---

## Downsides of Reflection

Reflection is powerful but comes with real costs. Interviewers expect you to name these trade-offs.

| Downside | Explanation |
|---|---|
| **Performance cost** | Reflective calls are slower than direct calls. The JVM can't optimize/inline them as aggressively, and there's overhead in looking up methods/fields by name. Fine for startup (one-time scanning); avoid in tight hot loops. |
| **Breaks encapsulation** | `setAccessible(true)` reaches past `private`. You can corrupt invariants the class author relied on, and your code becomes coupled to internal details that may change. |
| **No compile-time safety** | You reference members by `String` names (`"setName"`). A typo or rename compiles fine and only blows up at **runtime** with `NoSuchMethodException`/`NoSuchFieldException`. The compiler and IDE refactoring can't protect you. |
| **Harder to read & debug** | Indirect, dynamic code is harder to follow. "Find usages" in your IDE won't find a method only ever called by reflection. |
| **Security / module restrictions** | On Java 9+, the module system can block `setAccessible`, throwing `InaccessibleObjectException`. Security managers can also restrict reflection. |

**Think of it like surgery:**
Reflection is like operating on the inside of a machine while it's running. Surgeons (frameworks) do it carefully for good reasons. But you don't open up the engine for an oil change you could do from outside. Use reflection only when there's no ordinary, type-safe way to do the job.

> **Rule of thumb**: Let frameworks use reflection. In your own application code, prefer normal method calls, interfaces, and polymorphism. Reach for reflection only when you genuinely don't know the types at compile time (plugins, generic tooling).

---

## Common Mistakes & Pitfalls

```java
// MISTAKE 1: Forgetting @Retention(RUNTIME) on a custom annotation
@Target(ElementType.FIELD)
@interface MyAnnotation { }        // defaults to RetentionPolicy.CLASS!
// field.getAnnotation(MyAnnotation.class) returns NULL at runtime — annotation was discarded.
// FIX: add @Retention(RetentionPolicy.RUNTIME)

// MISTAKE 2: Using getFields() when you meant getDeclaredFields()
clazz.getFields();          // returns ONLY public fields (misses your private ones!)
clazz.getDeclaredFields();  // returns ALL fields of this class — usually what you want

// MISTAKE 3: Forgetting setAccessible(true) before touching private members
Field f = clazz.getDeclaredField("secret");
f.get(instance);            // IllegalAccessException — it's private!
// FIX: f.setAccessible(true); before get/set

// MISTAKE 4: Wrong parameter types when looking up a method/constructor
clazz.getDeclaredMethod("setAge", Integer.class); // looks for setAge(Integer)
// but the real method is setAge(int) → NoSuchMethodException
// FIX: use int.class for primitives, Integer.class for the wrapper — they differ!

// MISTAKE 5: Not unwrapping InvocationTargetException
method.invoke(obj);
// If the target method throws, you get InvocationTargetException (a WRAPPER),
// not the real exception. Call .getCause() to see what actually went wrong.

// MISTAKE 6: Catching the broad ReflectiveOperationException and ignoring it
// Reflection throws checked exceptions for a reason — swallowing them hides real bugs.
```

| Pitfall | Quick fix |
|---|---|
| Annotation returns `null` at runtime | Add `@Retention(RetentionPolicy.RUNTIME)` |
| Can't see private fields | Use `getDeclaredFields()`, not `getFields()` |
| `IllegalAccessException` | Call `setAccessible(true)` first |
| `NoSuchMethodException` on a primitive param | Use `int.class`, not `Integer.class` (they're different) |
| Real exception hidden | Unwrap with `InvocationTargetException.getCause()` |

---

## Common Interview Questions

### Q: What is an annotation? Does it change how your code runs?

An annotation is **metadata** attached to code (classes, methods, fields). By itself it does **nothing** — it's just a label. Behavior comes from whoever *reads* the annotation: the compiler (e.g., `@Override`) or a framework using reflection at runtime (e.g., `@Entity`, `@Test`). So an annotation changes behavior only indirectly, through its reader.

---

### Q: What's the difference between the three `RetentionPolicy` values?

- `SOURCE` — discarded by the compiler; never reaches the `.class` file (e.g., `@Override`).
- `CLASS` — kept in the bytecode but **not** available to reflection at runtime (this is the default).
- `RUNTIME` — kept and **readable via reflection at runtime**. Required for any annotation a framework reads while the app runs (e.g., `@Autowired`, `@Test`).

---

### Q: I wrote a custom annotation but `getAnnotation()` returns null. Why?

You almost certainly forgot `@Retention(RetentionPolicy.RUNTIME)`. The default retention is `CLASS`, which the JVM does **not** expose to reflection at runtime, so the annotation effectively disappears. Add the `RUNTIME` retention and it'll be readable.

---

### Q: What is reflection and why do frameworks need it?

Reflection is Java's ability to **inspect and manipulate classes, fields, methods, and annotations at runtime**, even for types unknown at compile time. Frameworks (Spring, Jackson, Hibernate, JUnit) are shipped *before your classes exist*, so they can't hard-code your class/field names. Reflection lets them discover your structure at runtime and act on it — that's how Jackson serializes *your* object to JSON without ever having seen your class.

---

### Q: What are the three ways to get a `Class` object, and when do you use each?

- `MyClass.class` — when the class is known at compile time.
- `obj.getClass()` — when you have an instance and want its **actual runtime type** (matters with inheritance/polymorphism).
- `Class.forName("com.example.MyClass")` — when you only have the class **name as a String**, decided at runtime (e.g., from config). It can throw `ClassNotFoundException`.

---

### Q: Difference between `getDeclaredFields()` and `getFields()`?

- `getDeclaredFields()` — **all** fields declared in *this* class (private/protected/public), but **not inherited** ones.
- `getFields()` — only **public** fields, **including inherited** public ones.

For framework-style work where you need private fields of the class itself, use `getDeclaredFields()`.

---

### Q: How can reflection access a `private` field? Should you do it?

Call `field.setAccessible(true)` to disable the access check, then `field.get()`/`field.set()`. This is how Hibernate populates entities and Spring injects into private `@Autowired` fields. You *can* do it, but in your own business code it's a red flag — it **breaks encapsulation**, makes code fragile, and on Java 9+ may be blocked by the module system. Let frameworks do it; avoid it in application logic.

---

### Q: How does Spring inject a dependency into a private field with no setter?

At startup Spring scans for beans, instantiates them via reflection, finds `@Autowired` fields with `getDeclaredFields()`, then does `field.setAccessible(true)` followed by `field.set(bean, dependency)`. It's the exact reflection pattern of reading an annotation marker and setting a private field directly.

---

### Q: How does JUnit know which methods are tests?

It uses reflection: it gets all methods via `getDeclaredMethods()`, checks each with `method.isAnnotationPresent(Test.class)`, skips the ones without `@Test`, and calls `method.invoke(testInstance)` on the rest, recording pass/fail. Annotations are the markers; reflection does the discovery and invocation.

---

### Q: How do you read a custom annotation's element value at runtime?

Get the annotated member (e.g., a `Field`), confirm it's present with `field.isAnnotationPresent(MyAnn.class)`, then `MyAnn ann = field.getAnnotation(MyAnn.class)` and call the element like a method: `ann.name()`. The annotation must be `@Retention(RUNTIME)` for this to work.

---

### Q: What are the downsides of reflection?

Slower than direct calls (harder for the JVM to optimize), breaks encapsulation via `setAccessible`, no compile-time safety (you use `String` names, so typos fail only at runtime), harder to read/debug, and possible module/security restrictions on Java 9+. Use it only when types truly aren't known at compile time.

---

### Q: What happens if a method invoked via reflection throws an exception?

Reflection wraps it in an `InvocationTargetException`. The actual exception thrown by the target method is available through `.getCause()`. A common mistake is logging the wrapper and missing the real error.

---

### Q: What's a meta-annotation? Name the four you know.

A meta-annotation is an annotation placed on *another annotation* to configure it. The four: `@Retention` (how long it survives), `@Target` (where it can be applied), `@Inherited` (subclasses inherit it — classes only), and `@Documented` (appears in Javadoc).

---

### Q: Why must an annotation read by reflection be `RUNTIME`, and what's the role of `@Target`?

`RUNTIME` is required because only `RUNTIME`-retained annotations are kept in a form the JVM exposes to reflection; `SOURCE` and `CLASS` aren't readable at runtime. `@Target` restricts *where* the annotation may legally be placed (method, field, type, etc.); misuse causes a compile error — it's a safety net, not a runtime concern.

---

## Quick Reference Cheat Sheet

```
ANNOTATIONS
  Annotation = metadata (a label). Does nothing alone — a reader gives it meaning.
  Readers: compiler (@Override) OR framework via reflection (@Entity, @Test).

Built-in (all read by the COMPILER):
  @Override            → verify you really override a method
  @Deprecated          → warn: API is outdated
  @SuppressWarnings    → silence a specific compiler warning
  @FunctionalInterface → enforce exactly one abstract method (for lambdas)

META-ANNOTATIONS (put on your own annotations):
  @Retention  → SOURCE  : gone after compile (not in .class)
                CLASS   : in .class, NOT visible to reflection (default)
                RUNTIME : visible to reflection  ← needed for frameworks
  @Target     → where allowed: TYPE, FIELD, METHOD, PARAMETER, CONSTRUCTOR...
  @Inherited  → subclasses inherit it (CLASSES only)
  @Documented → show it in Javadoc

CUSTOM ANNOTATION:
  @Retention(RUNTIME) @Target(FIELD)
  public @interface JsonField { String name() default ""; }
  - elements look like methods, can have default
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
    (same Declared-vs-plain rule applies to methods & constructors)

  Create & invoke:
    constructor.newInstance(args)   → like new MyClass(args)
    method.invoke(obj, args)        → like obj.method(args); pass null obj for static
    field.get(obj) / field.set(obj, value)

  Private access:
    member.setAccessible(true)  → bypass private check (breaks encapsulation)
    needed before get/set/invoke on private members

  Read annotations at runtime:
    field.isAnnotationPresent(JsonField.class)
    JsonField a = field.getAnnotation(JsonField.class);
    a.name();   // read an element  (annotation must be @Retention(RUNTIME))

HOW FRAMEWORKS WORK (the recurring pattern):
  reflection (discover members) + annotations (markers for which members matter)
  Spring @Autowired → getDeclaredFields → setAccessible(true) → field.set(bean, dep)
  JUnit  @Test      → getDeclaredMethods → isAnnotationPresent → method.invoke

DOWNSIDES:
  slower (no inlining) | breaks encapsulation | no compile-time safety (String names)
  harder to debug | Java 9+ module/security may block setAccessible

COMMON BUGS:
  annotation null at runtime  → forgot @Retention(RUNTIME)
  can't see private fields     → use getDeclaredFields(), not getFields()
  IllegalAccessException       → call setAccessible(true) first
  NoSuchMethodException        → int.class != Integer.class for primitive params
  real error hidden            → unwrap InvocationTargetException.getCause()
```

---

*Last Updated: 2026-06-11*
