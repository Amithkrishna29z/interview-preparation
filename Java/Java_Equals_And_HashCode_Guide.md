# equals() and hashCode() Contract — Interview Study Guide

## Overview

`equals()` and `hashCode()` are two methods that **every Java class inherits** from `Object`. They look harmless, but getting them wrong is one of the most common (and most subtle) bugs in Java — objects silently disappearing from a `HashSet`, duplicate keys in a `HashMap`, broken caches. This is a **guaranteed interview topic** for Java backend roles because it tests whether you really understand how collections work under the hood.

This guide takes you from "what does `==` even do?" all the way to "how should a JPA entity implement these methods?" — at a junior level, step by step.

---

## Table of Contents

1. [== vs equals() — The Basics](#-vs-equals--the-basics)
2. [Default Object.equals() and Object.hashCode()](#default-objectequals-and-objecthashcode)
3. [The equals() Contract](#the-equals-contract)
4. [The hashCode() Contract](#the-hashcode-contract)
5. [The Golden Rule: Override Both Together](#the-golden-rule-override-both-together)
6. [The HashMap/HashSet Bug — Step by Step](#the-hashmaphashset-bug--step-by-step)
7. [How HashMap & HashSet Use hashCode() Then equals()](#how-hashmap--hashset-use-hashcode-then-equals)
8. [Writing a Correct equals()](#writing-a-correct-equals)
9. [Writing a Correct hashCode()](#writing-a-correct-hashcode)
10. [Generating Them: IDE, Lombok, Records](#generating-them-ide-lombok-records)
11. [The Mutable Field Pitfall](#the-mutable-field-pitfall)
12. [JPA / Hibernate Entity Caveat](#jpa--hibernate-entity-caveat)
13. [Common Mistakes & Gotchas](#common-mistakes--gotchas)
14. [Common Interview Questions](#common-interview-questions)
15. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## == vs equals() — The Basics

These two are constantly confused. They answer **different questions**.

- `==` asks: **"Are these two variables pointing to the exact same object in memory?"** (identity / reference)
- `.equals()` asks: **"Do these two objects mean the same thing?"** (logical equality — *if* the class defines it that way)

**Think of it like two people named "John Smith":**
- `==` is like asking *"Are these the same single person?"* (same physical human)
- `.equals()` is like asking *"Do these two driver's licenses describe the same person?"* (same name, same date of birth, same address)

```java
String a = new String("hello");   // creates a brand-new String object in memory
String b = new String("hello");   // creates ANOTHER brand-new String object

System.out.println(a == b);        // false — two different objects in memory
System.out.println(a.equals(b));   // true  — String overrides equals() to compare characters
```

For **primitives** (`int`, `double`, `char`, `boolean`), `==` compares actual values — there are no objects involved:

```java
int x = 5;
int y = 5;
System.out.println(x == y);  // true — comparing the raw values 5 and 5
```

> **Rule of thumb**: Use `==` only for primitives and for the literal "is this the same object" check (and for `null` checks). Use `.equals()` for comparing the *contents* of objects (Strings, your own classes, etc.).

---

## Default Object.equals() and Object.hashCode()

Every class in Java extends `Object`. If you do **not** override these methods, you get `Object`'s default behavior.

### Default equals() = reference equality (just `==`)

The `Object` class implements `equals()` like this (simplified):

```java
// This is what Object.equals() does by default
public boolean equals(Object obj) {
    return (this == obj);   // only true if it's literally the SAME object in memory
}
```

So by default, two different objects are **never** equal, even if every field is identical:

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
}

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

System.out.println(p1.equals(p2)); // false! Same data, but different objects in memory
// Because Point did NOT override equals(), it falls back to Object's "==" check
```

### Default hashCode() = a number derived from the object's memory identity

`hashCode()` returns an `int`. By default, it is based on the object's internal identity (historically related to the memory address). Two different objects almost always get **different** default hash codes:

```java
Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

System.out.println(p1.hashCode()); // e.g., 1846274136
System.out.println(p2.hashCode()); // e.g., 1058025095  — different number!
```

**Think of hashCode() like a coat-check number** at a restaurant. When you hand over your coat, you get a ticket number. That number lets the staff quickly find roughly *where* your coat is hanging, without searching every hook. It's a fast "which shelf?" hint — not a guarantee of which exact coat.

**Key takeaway**: The default implementations treat every `new` object as unique. That is fine for some classes, but for "value" classes (a `Point`, a `Money`, a `UserId`) where two objects with the same data *should* be considered equal, you must override both methods.

---

## The equals() Contract

When you override `equals()`, Java's documentation says it **must** obey 5 rules. Interviewers love asking you to list these. Memorize them.

| Rule | Meaning | Plain English |
|---|---|---|
| **Reflexive** | `x.equals(x)` is always `true` | An object must equal itself |
| **Symmetric** | if `x.equals(y)` then `y.equals(x)` | If A equals B, then B must equal A |
| **Transitive** | if `x.equals(y)` and `y.equals(z)` then `x.equals(z)` | If A=B and B=C, then A=C |
| **Consistent** | repeated calls return the same result (if no fields change) | Same inputs → same answer every time |
| **Non-null** | `x.equals(null)` is always `false` | Nothing equals `null` |

**Think of it like the rules of "being friends":**
- *Reflexive* — you're always "friends" with yourself.
- *Symmetric* — if you're my friend, I'm your friend (it goes both ways).
- *Transitive* — if my friend is friends with someone, the friendship chain stays consistent.
- *Consistent* — we don't randomly stop being friends every time someone asks.
- *Non-null* — you can't be friends with "nobody."

```java
Point a = new Point(1, 2);
Point b = new Point(1, 2);
Point c = new Point(1, 2);

// Reflexive
a.equals(a);                       // must be true

// Symmetric
a.equals(b) == b.equals(a);        // must be the same result both ways

// Transitive
if (a.equals(b) && b.equals(c)) {
    a.equals(c);                   // must be true
}

// Consistent
a.equals(b);                       // calling it 100 times gives the same answer

// Non-null
a.equals(null);                    // must be false (never throw NullPointerException!)
```

> **Interview tip**: The contract that breaks most often in real code is **symmetry**, usually when one class tries to be "equal" to a subclass or to a different type. We'll see why `instanceof` can break symmetry later.

---

## The hashCode() Contract

`hashCode()` has its own contract, and it is **tightly linked** to `equals()`. There are 3 rules:

1. **Consistency** — calling `hashCode()` on the same object multiple times must return the same `int` (as long as the fields used in `equals()` don't change).
2. **Equal objects → equal hash codes** — **if `a.equals(b)` is `true`, then `a.hashCode()` MUST equal `b.hashCode()`.** This is the big one.
3. **Unequal objects → hash codes *may* collide** — if `a.equals(b)` is `false`, their hash codes *can* be the same (a "collision"). It's allowed, just not ideal for performance.

| Situation | Are hash codes required to be equal? |
|---|---|
| `a.equals(b)` is `true` | **YES — mandatory** |
| `a.equals(b)` is `false` | No — they may be equal or different (collisions are OK) |

**Think of hashCode() like a postal ZIP code, and equals() like the full street address:**
- If two letters go to the **same full address**, they MUST share the same ZIP code (rule 2). It would be insane for the same house to have two different ZIP codes.
- But two **different addresses** can share the same ZIP code (rule 3) — a ZIP code covers many houses. That's a collision, and it's perfectly normal.

```java
// If two objects are equal...
a.equals(b);          // true

// ...then this MUST hold:
a.hashCode() == b.hashCode();   // MUST be true, or collections break
```

> **The one-way street**: Equal objects must have equal hash codes. But equal hash codes do **NOT** mean the objects are equal (that's just a collision). Hash code is a fast first filter; `equals()` is the final word.

---

## The Golden Rule: Override Both Together

> ### If you override `equals()`, you **MUST** override `hashCode()`. Always. No exceptions.

**Why?** Because hash-based collections (`HashMap`, `HashSet`, `HashMap`-backed caches) rely on the contract *"equal objects have equal hash codes."* If you override `equals()` so that two objects are "equal," but leave the default `hashCode()` (which gives them *different* numbers), you have **broken the contract** — and the collections will misbehave in confusing ways.

Let's prove it with a concrete bug.

---

## The HashMap/HashSet Bug — Step by Step

Here is a class that overrides `equals()` but **forgets** `hashCode()` — the classic mistake.

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;   // equal if x and y match
    }
    // BUG: no hashCode() override! Still using Object's default (memory-identity based)
}
```

Now watch what happens with a `HashSet`:

```java
Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));                 // store a point

boolean found = set.contains(new Point(1, 2));  // look for an "equal" point
System.out.println(found);                // prints: FALSE  (!!!)  — we expected true
```

### Walking through WHY it fails

**Step 1 — Adding `new Point(1, 2)`:**
- `HashSet` calls `hashCode()` on the point to decide which internal "bucket" to store it in.
- Default `hashCode()` returns, say, `111`. So the point goes into **bucket 111**.

**Step 2 — Searching with `new Point(1, 2)`:**
- This is a *different* object in memory.
- `HashSet` calls `hashCode()` on it. Default `hashCode()` returns, say, `222` (different object → different default hash).
- `HashSet` looks in **bucket 222**.

**Step 3 — The miss:**
- Bucket 222 is empty. The stored point is over in bucket 111.
- `HashSet` never even *gets* to call `equals()`, because it looked in the wrong bucket entirely.
- Result: `contains()` returns `false`. The object is "lost."

```
Stored point  (1,2)  → hashCode 111 → bucket [111]   ← lives here
Search point  (1,2)  → hashCode 222 → bucket [222]   ← looks here (empty!)
Different buckets → never compared with equals() → NOT FOUND
```

### The fix — add a matching hashCode()

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);   // equal points (same x,y) now get the SAME hash code
    }
}
```

Now both `new Point(1, 2)` objects produce the **same** hash code → same bucket → `equals()` runs → match found. `contains()` correctly returns `true`.

> **Interview gold**: Be ready to walk through this exact scenario. Saying "you must override both" is fine, but explaining *the wrong-bucket lookup* shows you actually understand it.

---

## How HashMap & HashSet Use hashCode() Then equals()

A `HashSet` is actually backed by a `HashMap` internally, so understanding `HashMap` covers both. Here's the mental model.

**Think of a HashMap like a library with numbered shelves:**
- The `hashCode()` tells you **which shelf** to walk to (fast — you skip every other shelf).
- Once at the shelf, you read the **book titles** (`equals()`) to find the exact one you want.

A `HashMap` stores entries in an array of **buckets**. The process for both storing and looking up:

```
1. Compute key.hashCode()                  → an int
2. Spread/compress it into a bucket index  → which slot in the array
3. Go to that bucket
4. Walk the entries in that bucket, comparing with equals()
5. Match found → return value / confirm contains
   No match    → key not present
```

```java
Map<Point, String> map = new HashMap<>();
map.put(new Point(1, 2), "home");          // store under bucket = hash(1,2)

String value = map.get(new Point(1, 2));   // look in bucket = hash(1,2)
// Step 1: hashCode of (1,2) → say bucket 5
// Step 2: go to bucket 5
// Step 3: for each entry in bucket 5, call equals() against our (1,2) key
// Step 4: equals() matches → returns "home"
System.out.println(value); // "home"  (only works if BOTH methods are correctly overridden)
```

**Why two methods instead of one?**
- `equals()` is potentially **slow** (compares every field). Comparing your key against all 1,000,000 stored keys would be terrible.
- `hashCode()` is **fast** and narrows 1,000,000 keys down to just the few in one bucket.
- So: `hashCode()` for *fast bucket selection*, then `equals()` for the *exact match* within that small bucket. Best of both worlds.

| Method | Role | Speed | Question it answers |
|---|---|---|---|
| `hashCode()` | Pick the bucket | Fast | "Which shelf?" |
| `equals()` | Confirm exact match in bucket | Slower | "Is this the exact one?" |

> **Note on collisions**: When two different keys land in the same bucket, the entries form a small list (or, in modern Java, a balanced tree once a bucket gets large). `equals()` then distinguishes them. Good `hashCode()` implementations spread keys evenly to keep buckets small.

---

## Writing a Correct equals()

Here is the standard, safe template. Read every comment.

```java
@Override
public boolean equals(Object o) {
    // 1. Same object reference? Quick win — an object always equals itself.
    if (this == o) return true;

    // 2. Null check + type check in one shot.
    //    If o is null, instanceof returns false (no NullPointerException).
    //    If o is a different type, it's not equal.
    if (o == null || getClass() != o.getClass()) return false;

    // 3. Safe cast — we know o is the right type now.
    Person person = (Person) o;

    // 4. Compare the fields that define "equality" for this class.
    //    Use Objects.equals(...) for object fields — it handles nulls safely.
    //    Use == for primitives.
    return age == person.age
        && Objects.equals(name, person.name)
        && Objects.equals(email, person.email);
}
```

### `getClass()` vs `instanceof` — the classic debate

Step 2 above can be written two ways, and interviewers love this:

```java
// Option A — getClass()
if (o == null || getClass() != o.getClass()) return false;

// Option B — instanceof
if (!(o instanceof Person)) return false;   // note: instanceof is false when o is null, so no separate null check needed
```

| Approach | Pros | Cons |
|---|---|---|
| **`getClass()`** | Strict — a `Person` is never equal to a subclass `Employee`. Keeps **symmetry** safe. | A proxy/subclass (e.g., Hibernate proxies) won't be equal to the base type. |
| **`instanceof`** | Allows subclasses to be considered equal; works with proxies. | Can **break symmetry** if a subclass adds fields to its own `equals()`. |

**The symmetry trap with `instanceof`:** Suppose `Person` uses `instanceof`, and `Employee extends Person` adds a `salary` field to its `equals()`. Then:
- `person.equals(employee)` → `Person`'s equals sees an `instanceof Person` → compares only name/age → **true**
- `employee.equals(person)` → `Employee`'s equals checks salary, but `person` has no salary → **false**

That's asymmetric — a contract violation! `getClass()` avoids this because the two objects have different classes, so they're never equal.

> **Practical advice for juniors**: Use `getClass()` for most plain value objects (it's safe and strict). The main exception is JPA entities, where Hibernate creates *proxy subclasses* — there you'll see `instanceof` recommended (more on this later). When in doubt for an interview, mention both and explain the symmetry tradeoff.

### Why `Objects.equals()` for fields?

```java
// WITHOUT Objects.equals — crashes if name is null
return name.equals(person.name);   // NullPointerException if this.name == null

// WITH Objects.equals — null-safe
return Objects.equals(name, person.name);
// Internally: (name == person.name) || (name != null && name.equals(person.name))
// Handles: both null → true, one null → false, neither null → name.equals(...)
```

---

## Writing a Correct hashCode()

The modern, recommended way is `Objects.hash(...)`. Pass it **the same fields** you used in `equals()`.

```java
@Override
public int hashCode() {
    // Use the SAME fields as equals(): name, email, age.
    // Objects.hash combines them into a single int for you.
    return Objects.hash(name, email, age);
}
```

> **The cardinal rule**: `equals()` and `hashCode()` must use the **same set of fields**. If `equals()` compares `name + email` but `hashCode()` uses only `name`, you can get objects that are equal but... actually that's fine for the contract. The *dangerous* mistake is the reverse: never let `hashCode()` depend on a field that `equals()` ignores. Keep them in sync — same fields, both directions, to be safe.

### What does `Objects.hash()` do, and why the magic number 31?

Under the hood, the classic hash-combining formula looks like this:

```java
@Override
public int hashCode() {
    int result = 17;                          // arbitrary non-zero starting value
    result = 31 * result + (name == null ? 0 : name.hashCode());   // mix in name
    result = 31 * result + (email == null ? 0 : email.hashCode()); // mix in email
    result = 31 * result + age;               // mix in age
    return result;
}
```

**Why multiply by 31 each step?**
- **It makes order matter.** Without multiplying, `hash("AB")` and `hash("BA")` would collide. Multiplying spreads fields apart so `(name, email)` hashes differently from `(email, name)`.
- **31 is an odd prime.** Odd primes reduce collision patterns and distribute bits well.
- **It's fast.** The JVM optimizes `31 * x` into `(x << 5) - x` (a bit-shift and a subtract) — cheaper than a real multiplication.

You normally don't write this by hand — just use `Objects.hash(...)`. But interviewers ask "why 31?", so know the answer: **odd prime, good bit distribution, JVM-optimizable to a shift.**

**Think of building a hash like making a smoothie:** you blend each ingredient (field) in, and the `* 31` is the blending action that mixes them so the final color (the int) depends on *all* ingredients *and* the order you added them.

### One small caveat about `Objects.hash()`

`Objects.hash(...)` accepts varargs, which means it creates a small array on every call. For 99% of code this is irrelevant. In an extreme hot path you might inline the `31 * result + ...` formula to avoid the array — but don't prematurely optimize. Reach for `Objects.hash()` first.

---

## Generating Them: IDE, Lombok, Records

You rarely hand-write these in real projects. Three common tools:

### 1. IDE generation (IntelliJ / Eclipse)

Right-click → *Generate* → *equals() and hashCode()*, then pick the fields. The IDE writes the standard template for you. Great default — clean, correct, and you can read exactly what it generated.

### 2. Lombok `@EqualsAndHashCode`

```java
import lombok.EqualsAndHashCode;

@EqualsAndHashCode                 // Lombok generates equals() AND hashCode() at compile time
public class Person {
    private String name;
    private String email;
    private int age;
}
// By default Lombok uses ALL non-static fields. Both methods stay in sync automatically.
```

You can fine-tune which fields are used:

```java
@EqualsAndHashCode
public class Person {
    @EqualsAndHashCode.Include private String email;   // only email is used...
    @EqualsAndHashCode.Exclude private String name;    // ...name is ignored
}
// (requires @EqualsAndHashCode(onlyExplicitlyIncluded = true) for the Include-only style)
```

> **Lombok gotcha**: `@EqualsAndHashCode` on a class that extends another does **not** call the parent's fields by default. Add `@EqualsAndHashCode(callSuper = true)` when inheritance matters. This is a common code-review catch.

### 3. Java `record` — auto-implemented (Java 16+)

A `record` automatically generates `equals()`, `hashCode()`, `toString()`, and accessors based on **all** its components. This is the cleanest option for immutable value classes.

```java
public record Point(int x, int y) { }
// The compiler auto-generates:
//   equals()   → compares x and y
//   hashCode() → combines x and y (contract honored)
//   toString() → "Point[x=1, y=2]"

Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
System.out.println(p1.equals(p2));   // true  — generated equals compares components
System.out.println(p1.hashCode() == p2.hashCode()); // true — generated hashCode is consistent
```

> **Interview tip**: If asked "the easiest way to get a correct, immutable value object with proper equals/hashCode?" — the answer is a `record`. It's impossible to forget one of the two, because the compiler does both.

| Tool | Effort | Stays in sync? | Best for |
|---|---|---|---|
| Hand-written | High | Only if you're careful | Learning / special logic |
| IDE generate | Low | Yes (regenerate if fields change) | Most classes |
| Lombok `@EqualsAndHashCode` | Very low | Yes (automatic) | Reducing boilerplate |
| `record` | None | Yes (compiler-enforced) | Immutable value objects |

---

## The Mutable Field Pitfall

Here's a trap even experienced developers hit: **if `hashCode()` depends on a field that you later change, hash-based collections break.**

**Think of it like a coat-check again:** you check your coat and get ticket #5 (based on the coat's color). Then, while it's hanging, someone repaints the coat a different color. Now the *new* color says "ticket #9." When you go to look it up by the new color, the staff search shelf #9 — but your coat is still hanging on shelf #5. It's lost, even though it's right there in the building.

```java
class User {
    String name;   // hashCode() depends on this MUTABLE field
    User(String name) { this.name = name; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        return Objects.equals(name, ((User) o).name);
    }
    @Override public int hashCode() { return Objects.hash(name); }
}
```

```java
Set<User> users = new HashSet<>();
User u = new User("Alice");
users.add(u);                          // hashCode("Alice") → stored in bucket A

u.name = "Bob";                        // MUTATE the field hashCode() depends on!

System.out.println(users.contains(u)); // FALSE — even though u IS in the set!
// Why: contains() now computes hashCode("Bob") → looks in bucket B (empty),
//      but the object is still physically sitting in bucket A.

for (User x : users) {
    System.out.println(x == u);        // true — it's literally still in there!
}
// The set still "contains" the object when you iterate, but contains()/remove() can't find it.
```

### How to avoid this

1. **Prefer immutable fields** for anything used in `hashCode()`/`equals()`. Make them `final` and don't provide setters. (This is why `record` and value objects work so well.)
2. **Never mutate** a field used in `hashCode()` while the object is inside a `HashSet`/`HashMap` key. If you must, *remove* it from the collection first, mutate, then *re-add*.

> **Interview phrasing**: "Objects used as `HashMap` keys or `HashSet` elements should be effectively immutable on the fields that drive `equals`/`hashCode`."

---

## JPA / Hibernate Entity Caveat

This is a **backend-specific** topic that frequently comes up. JPA entities are tricky because of their auto-generated IDs.

### The problem with using the generated `@Id`

```java
@Entity
public class Customer {
    @Id @GeneratedValue        // the ID is NULL until the entity is saved to the DB!
    private Long id;
    private String email;

    // TEMPTING but DANGEROUS:
    @Override public boolean equals(Object o) { /* compare by id */ }
    @Override public int hashCode() { return Objects.hash(id); }
}
```

Why this breaks:

```java
Customer c = new Customer();         // id is NULL (not saved yet)
Set<Customer> set = new HashSet<>();
set.add(c);                          // hashCode() uses id=null → goes in bucket "null"

customerRepository.save(c);          // NOW Hibernate assigns id = 42
                                     // → hashCode() now uses id=42 → would map to a DIFFERENT bucket

System.out.println(set.contains(c)); // FALSE — the id changed from null to 42 after insertion!
```

This is the **mutable field pitfall in disguise**: the database-generated `id` is `null` before save and non-null after, so it mutates exactly once — and that's enough to lose the object in a set.

### The recommended approaches

**Option 1 — Use a business key (natural key):** a field that is unique and stable from creation, like `email`, `isbn`, `orderNumber`, or a UUID you assign in the constructor.

```java
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @Column(unique = true, nullable = false, updatable = false)
    private String email;        // stable, unique business key — set at creation, never changes

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        // instanceof (not getClass) because Hibernate creates proxy SUBCLASSES at runtime
        if (!(o instanceof Customer)) return false;
        Customer other = (Customer) o;
        return email != null && email.equals(other.email);  // compare by stable business key
    }

    @Override
    public int hashCode() {
        // Return a CONSTANT or a class-based hash so it never changes across the entity lifecycle.
        return Customer.class.hashCode();
    }
}
```

> **Why a constant `hashCode()`?** It guarantees the hash never changes when the entity transitions from transient (no id) to persistent (has id). All entities of this type land in the same bucket, and `equals()` (on the business key) sorts them out. You lose some hash-spreading performance, but you gain correctness — which matters far more for entities. This is the widely recommended pattern (see Vlad Mihalcea's writings on the topic).

**Option 2 — Assign a UUID in the constructor:** then you have a stable, non-null key from the very start and can use it in both methods normally.

### Quick rules for JPA entities

| Do | Don't |
|---|---|
| Use a stable **business key** (email, UUID set at creation) | Use the auto-generated `@Id` (it's null before save) |
| Use `instanceof` (handles Hibernate proxies) | Use `getClass()` (proxy ≠ entity class → breaks equality) |
| Make `hashCode()` return a constant or class hash | Let `hashCode()` depend on a value that changes after save |
| Keep the key fields immutable | Mutate equality fields after putting the entity in a collection |

> **Interview soundbite**: "For JPA entities, never base `equals`/`hashCode` on the generated `@Id`, because it's null before persistence and changes on save — which loses the entity in hash collections. Use a stable business key, `instanceof` for proxy-safety, and a constant `hashCode`."

---

## Common Mistakes & Gotchas

- **Overriding `equals()` but not `hashCode()`** — the #1 mistake. Breaks `HashMap`/`HashSet`. (See the step-by-step bug above.)
- **Wrong parameter type** — writing `public boolean equals(Person p)` instead of `equals(Object o)`. This *overloads* rather than *overrides*; collections still call the inherited `Object.equals(Object)`. Always add `@Override` so the compiler catches this.
- **Forgetting the null/type check** — calling `((Person) o).name` without checking type throws `ClassCastException`; dereferencing a null field throws `NullPointerException`. Use the standard template.
- **Using different fields in `equals()` vs `hashCode()`** — they must be based on the same data, or you can get equal objects with different hash codes (contract violation).
- **Using mutable fields in `hashCode()`** — object gets lost in a set after mutation. Prefer immutable/`final` fields.
- **Breaking symmetry with `instanceof` + subclassing** — see the `getClass()` vs `instanceof` section.
- **Including derived/computed fields** — don't include a field that's calculated from other fields; it adds nothing and risks inconsistency.
- **Forgetting `@Override`** — without it, a typo (like `hashcode()` lowercase) silently creates a new method instead of overriding, and you keep the broken default.
- **Comparing floating-point with `==`** — for `double`/`float` fields, prefer `Double.compare(a, b) == 0` to handle `NaN` and `-0.0` correctly (this is what `Objects.equals` for boxed `Double` and IDE-generated code does).

---

## Common Interview Questions

### Q: What is the difference between `==` and `.equals()`?

`==` compares **references** for objects (are these the same object in memory?) and **values** for primitives. `.equals()` compares **logical content**, but only if the class overrides it — otherwise it falls back to `Object.equals()`, which is just `==`. Example: two `new String("hi")` are `==` false but `.equals()` true, because `String` overrides `equals()`.

---

### Q: If you override `equals()`, why must you override `hashCode()`?

Because hash-based collections (`HashMap`, `HashSet`) rely on the contract that **equal objects have equal hash codes**. They use `hashCode()` to pick a bucket, then `equals()` within that bucket. If two "equal" objects produce different hash codes (the default behavior), they land in different buckets, `equals()` never runs, and the collection can't find the object — leading to "lost" entries and apparent duplicates.

---

### Q: State the `equals()` contract.

Reflexive (`x.equals(x)` true), Symmetric (`x.equals(y)` ⇒ `y.equals(x)`), Transitive (`x=y` and `y=z` ⇒ `x=z`), Consistent (same result on repeated calls), and Non-null (`x.equals(null)` is false).

---

### Q: State the `hashCode()` contract.

(1) Consistent — same object returns the same hash if equality-fields don't change. (2) Equal objects **must** have equal hash codes. (3) Unequal objects *may* share a hash code (collisions are allowed). The critical rule is #2.

---

### Q: Can two unequal objects have the same hash code?

Yes — that's a **collision**, and it's perfectly legal. `hashCode()` returns an `int` (about 4 billion values), but there can be infinitely many distinct objects, so collisions are unavoidable. The collection resolves collisions by calling `equals()` on the entries in the same bucket. What's *not* allowed is the reverse: equal objects with different hash codes.

---

### Q: Walk me through what happens when `HashMap.get(key)` is called.

(1) Compute `key.hashCode()`. (2) Convert it to a bucket index. (3) Go to that bucket. (4) Walk the entries there, calling `equals()` against your key. (5) On a match, return the value; otherwise the key isn't present. `hashCode()` provides fast bucket selection; `equals()` confirms the exact match.

---

### Q: `getClass()` vs `instanceof` in `equals()` — which and why?

`getClass()` is strict: an object is only equal to another of the *exact same class*, which keeps **symmetry** safe even with subclasses. `instanceof` allows subclass instances to be equal but can **break symmetry** if a subclass adds fields to its `equals()`. For plain value objects, `getClass()` is the safe default. For **JPA entities**, use `instanceof` because Hibernate creates proxy subclasses that must still be considered equal to the base entity.

---

### Q: Why is the number 31 used in `hashCode()` implementations?

31 is an **odd prime**, which gives good bit distribution and reduces collisions. Multiplying the running result by 31 at each step makes field **order** matter (so `"AB"` and `"BA"` differ). And the JVM optimizes `31 * x` into `(x << 5) - x`, a fast shift-and-subtract, so it's cheap.

---

### Q: What's the simplest way to get a correct `equals`/`hashCode` for an immutable value class?

Use a **`record`** (Java 16+). The compiler auto-generates `equals()`, `hashCode()`, and `toString()` from the record's components, all honoring the contract — you can't forget one. Alternatively, use IDE generation or Lombok's `@EqualsAndHashCode`.

---

### Q: What goes wrong if a field used in `hashCode()` is mutable?

If you mutate that field while the object is a key in a `HashMap` (or element in a `HashSet`), its hash code changes, so the collection looks in a different bucket than where the object is stored. The object becomes unreachable via `get`/`contains`/`remove`, even though it's still physically inside the collection. Fix: use immutable (`final`) fields for equality, or remove-mutate-re-add.

---

### Q: How should you implement `equals`/`hashCode` for a JPA/Hibernate entity?

Do **not** use the auto-generated `@Id` — it's `null` before save and gets assigned on insert, so the hash changes and the entity is lost in hash collections. Instead use a **stable business key** (e.g., `email`, or a UUID assigned in the constructor). Use `instanceof` (not `getClass()`) so Hibernate proxy subclasses compare correctly, and make `hashCode()` return a **constant** (e.g., the class's hash) so it never changes across the entity's lifecycle.

---

### Q: What happens if you override `equals(Person p)` instead of `equals(Object o)`?

You've created an **overload**, not an **override**. Collections call `Object.equals(Object)`, which you didn't override, so they use the default reference equality — your custom logic is silently ignored. The `@Override` annotation prevents this by failing to compile when the signature doesn't actually override anything.

---

### Q: Does `String` override `equals()` and `hashCode()`? What about wrapper types?

Yes. `String`, and all the wrapper classes (`Integer`, `Long`, `Double`, `Boolean`, etc.), override both — `equals()` compares values, and `hashCode()` is derived from the value. That's why `Integer.valueOf(5).equals(Integer.valueOf(5))` is true and why they work correctly as `HashMap` keys.

---

### Q: Can `hashCode()` return a constant value for all objects? Is it legal?

Legal — yes, it satisfies the contract (equal objects still have equal hash codes). But it's **terrible for performance**: every object lands in the same bucket, turning the `HashMap` into effectively a linked list (or tree), so lookups degrade from O(1) to O(n) or O(log n). The exception is JPA entities, where a constant hash is the accepted tradeoff for correctness across the entity lifecycle.

---

## Quick Reference Cheat Sheet

```
== vs equals():
  ==        → same object in memory (reference); raw value for primitives
  .equals() → logical/content equality IF the class overrides it
              (default equals = ==)

Default (Object) behavior:
  equals()   → reference equality (this == o)
  hashCode() → identity-based number; unique per object

THE GOLDEN RULE:
  Override equals() ⇒ you MUST override hashCode()
  (and use the SAME fields in both)

equals() contract (5):
  Reflexive   → x.equals(x) is true
  Symmetric   → x.equals(y) ⇒ y.equals(x)
  Transitive  → x=y and y=z ⇒ x=z
  Consistent  → same result every call
  Non-null    → x.equals(null) is false

hashCode() contract (3):
  Consistent              → same object, same hash (if fields unchanged)
  equal ⇒ equal hash      → MANDATORY
  unequal ⇒ may collide   → collisions are OK

One-way street:
  equal objects        → MUST have equal hash codes
  equal hash codes     → does NOT mean objects are equal (collision)

How HashMap/HashSet find things:
  1. hashCode()  → pick the bucket (fast)
  2. equals()    → exact match within the bucket (precise)

The classic bug (override equals, forget hashCode):
  add (1,2)    → default hash → bucket A
  contains(1,2)→ different default hash → bucket B (empty) → NOT FOUND

equals() template:
  if (this == o) return true;
  if (o == null || getClass() != o.getClass()) return false;
  Type t = (Type) o;
  return Objects.equals(field, t.field) && ...;

hashCode() template:
  return Objects.hash(field1, field2, ...);   // same fields as equals

getClass() vs instanceof:
  getClass()  → strict, symmetry-safe → value objects
  instanceof  → proxy-friendly        → JPA entities

Why 31 in hashCode:
  odd prime + good distribution + JVM optimizes 31*x to (x<<5)-x
  makes field ORDER matter

Generation:
  IDE generate          → safe default
  @EqualsAndHashCode    → Lombok (watch callSuper for inheritance)
  record                → auto equals/hashCode/toString (Java 16+)

Mutable field pitfall:
  changing a hashCode field while in a Set/Map key → object lost
  → use final/immutable fields for equality

JPA entity rules:
  DON'T use generated @Id (null before save → changes → lost)
  DO   use a stable business key (email / UUID set at creation)
  DO   use instanceof (Hibernate proxies)
  DO   make hashCode() a constant (class hash)

Common mistakes:
  - equals without hashCode
  - equals(Person) instead of equals(Object)  → use @Override
  - different fields in equals vs hashCode
  - mutable fields in hashCode
  - forgetting @Override (typo creates new method)
```

---

*Last Updated: 2026-06-11*
