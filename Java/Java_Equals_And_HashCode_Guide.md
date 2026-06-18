# equals() and hashCode() Contract — Interview Study Guide

## Overview

`equals()` and `hashCode()` are inherited by every class from `Object`. Getting them wrong is a classic Java bug — objects vanishing from a `HashSet`, duplicate keys in a `HashMap`, broken caches. It's a **guaranteed interview topic** because it tests whether you understand how hash collections work under the hood.

---

## Table of Contents

1. [== vs equals()](#-vs-equals)
2. [Default Object Behavior](#default-object-behavior)
3. [The equals() Contract](#the-equals-contract)
4. [The hashCode() Contract](#the-hashcode-contract)
5. [The Golden Rule + The Classic Bug](#the-golden-rule--the-classic-bug)
6. [How HashMap Uses Both](#how-hashmap-uses-both)
7. [Writing equals() and hashCode()](#writing-equals-and-hashcode)
8. [Generating Them: IDE, Lombok, Records](#generating-them-ide-lombok-records)
9. [The Mutable Field Pitfall](#the-mutable-field-pitfall)
10. [JPA / Hibernate Entity Caveat](#jpa--hibernate-entity-caveat)
11. [Common Mistakes](#common-mistakes)
12. [Common Interview Questions](#common-interview-questions)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## == vs equals()

They answer different questions:

- `==` → "Are these the same object in memory?" (reference). For **primitives**, it compares raw values.
- `.equals()` → "Do these objects mean the same thing?" (logical equality) — *if* the class overrides it.

```java
String a = new String("hello");
String b = new String("hello");
System.out.println(a == b);       // false — two different objects
System.out.println(a.equals(b));  // true  — String overrides equals() to compare characters

int x = 5, y = 5;
System.out.println(x == y);       // true — primitives compare values
```

> **Rule of thumb**: `==` for primitives and "same object"/`null` checks. `.equals()` for comparing object contents.

---

## Default Object Behavior

If you don't override these, you get `Object`'s defaults:

- **Default `equals()`** = reference equality (`this == obj`). Two different objects are never equal, even with identical fields.
- **Default `hashCode()`** = a number based on the object's identity. Two different objects almost always get different hash codes.

```java
Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);
System.out.println(p1.equals(p2)); // false! same data, different objects
```

For "value" classes (a `Point`, `Money`, `UserId`) where same data *should* mean equal, you must override both.

---

## The equals() Contract

When you override `equals()`, it must obey 5 rules:

| Rule | Meaning |
|---|---|
| **Reflexive** | `x.equals(x)` is always `true` |
| **Symmetric** | if `x.equals(y)` then `y.equals(x)` |
| **Transitive** | if `x.equals(y)` and `y.equals(z)` then `x.equals(z)` |
| **Consistent** | repeated calls return the same result (if fields unchanged) |
| **Non-null** | `x.equals(null)` is always `false` (never throw NPE) |

> **Interview tip**: The rule that breaks most often is **symmetry**, usually when `instanceof` lets a class be "equal" to a subclass (see later).

---

## The hashCode() Contract

`hashCode()` returns an `int` and is tightly linked to `equals()`:

1. **Consistent** — same object returns the same `int` (if equality-fields don't change).
2. **Equal objects → equal hash codes** — if `a.equals(b)`, then `a.hashCode() == b.hashCode()`. **The big one.**
3. **Unequal objects → may collide** — different objects *can* share a hash code. Allowed, just not ideal.

**Think of `hashCode()` like a ZIP code and `equals()` like the full address:** same address → same ZIP (rule 2 is mandatory), but two different addresses can share a ZIP (collisions are normal).

> **One-way street**: Equal objects MUST have equal hash codes. But equal hash codes do NOT mean the objects are equal — that's just a collision. `hashCode()` is a fast first filter; `equals()` is the final word.

---

## The Golden Rule + The Classic Bug

> ### If you override `equals()`, you MUST override `hashCode()`. Always.

Here's a class that overrides `equals()` but forgets `hashCode()` — the classic mistake:

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    // BUG: no hashCode() — still uses Object's identity-based default
}
```

```java
Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));
System.out.println(set.contains(new Point(1, 2)));  // FALSE (!!) — expected true
```

**Why it fails:**

```
Stored point (1,2) → hashCode 111 → bucket [111]   ← lives here
Search point (1,2) → hashCode 222 → bucket [222]   ← looks here (empty!)
Different buckets → equals() never runs → NOT FOUND
```

**The fix** — add a matching `hashCode()` using the same fields:

```java
@Override public int hashCode() {
    return Objects.hash(x, y);   // equal points now get the SAME hash → same bucket
}
```

> **Interview gold**: Be ready to walk through the *wrong-bucket lookup* — it shows you actually understand it, not just "override both."

---

## How HashMap Uses Both

A `HashSet` is backed by a `HashMap`, so understanding `HashMap` covers both. Entries live in an array of **buckets**:

```
1. Compute key.hashCode()        → an int
2. Convert to a bucket index     → which slot
3. Go to that bucket
4. Walk entries, comparing with equals()
5. Match → return value / confirm contains;  No match → not present
```

**Why two methods?** `equals()` is slow (compares every field); checking it against a million keys would be terrible. `hashCode()` is fast and narrows a million keys to the few in one bucket. So: `hashCode()` for fast bucket selection, `equals()` for the exact match within it.

| Method | Role | Speed |
|---|---|---|
| `hashCode()` | Pick the bucket ("which shelf?") | Fast |
| `equals()` | Confirm exact match in the bucket | Slower |

> Collisions (two keys in one bucket) form a small list — or a balanced tree once a bucket gets large. Good `hashCode()` spreads keys evenly to keep buckets small.

---

## Writing equals() and hashCode()

The standard, safe templates:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;                          // same reference — quick win
    if (o == null || getClass() != o.getClass()) return false;  // null + type check
    Person person = (Person) o;                          // safe cast
    return age == person.age                             // == for primitives
        && Objects.equals(name, person.name)            // Objects.equals = null-safe
        && Objects.equals(email, person.email);
}

@Override
public int hashCode() {
    return Objects.hash(name, email, age);   // SAME fields as equals()
}
```

**Why `Objects.equals()`?** It's null-safe — `name.equals(...)` throws NPE if `name` is null; `Objects.equals(name, other)` handles nulls.

**The cardinal rule**: `equals()` and `hashCode()` must use the **same fields**.

### `getClass()` vs `instanceof`

| Approach | Pros | Cons |
|---|---|---|
| **`getClass()`** | Strict; a `Person` is never equal to a subclass. Keeps **symmetry** safe. | Hibernate proxies (subclasses) won't equal the base type. |
| **`instanceof`** | Allows subclasses/proxies to be equal. | Can **break symmetry** if a subclass adds fields to its `equals()`. |

Use `getClass()` for plain value objects; use `instanceof` for JPA entities (proxies). In an interview, mention both and the symmetry tradeoff.

### Why 31 in hand-written hashCode?

The classic formula multiplies by 31 at each step (`result = 31 * result + field`):
- **Makes order matter** so `(name, email)` differs from `(email, name)`.
- **31 is an odd prime** → good bit distribution, fewer collision patterns.
- **Fast** — JVM optimizes `31 * x` into `(x << 5) - x`.

You normally just use `Objects.hash(...)`, but know the "why 31?" answer.

---

## Generating Them: IDE, Lombok, Records

You rarely hand-write these. Three tools:

**1. IDE generation** — Right-click → *Generate* → *equals() and hashCode()*, pick the fields. Safe default.

**2. Lombok `@EqualsAndHashCode`:**

```java
@EqualsAndHashCode                 // generates both at compile time
public class Person {
    private String name;
    private String email;
}
// Uses all non-static fields by default; both stay in sync automatically.
```

> **Gotcha**: with inheritance, add `@EqualsAndHashCode(callSuper = true)` — by default it ignores the parent's fields.

**3. Java `record` (16+)** — auto-generates `equals()`, `hashCode()`, `toString()` from all components. Cleanest for immutable value objects:

```java
public record Point(int x, int y) { }
// equals compares x,y; hashCode combines x,y — contract honored, impossible to forget one
```

| Tool | Effort | Stays in sync? | Best for |
|---|---|---|---|
| Hand-written | High | Only if careful | Learning / special logic |
| IDE generate | Low | Yes (regenerate) | Most classes |
| Lombok | Very low | Yes | Reducing boilerplate |
| `record` | None | Compiler-enforced | Immutable value objects |

---

## The Mutable Field Pitfall

**If `hashCode()` depends on a field you later change, hash collections break.**

```java
Set<User> users = new HashSet<>();
User u = new User("Alice");
users.add(u);                          // hashCode("Alice") → bucket A

u.name = "Bob";                        // MUTATE the field hashCode() depends on

System.out.println(users.contains(u)); // FALSE — even though u IS in the set!
// contains() computes hashCode("Bob") → looks in bucket B (empty);
// the object is still physically in bucket A.
```

**How to avoid it:**
1. **Prefer immutable (`final`) fields** for anything in `equals`/`hashCode`. (Why `record`/value objects work so well.)
2. **Never mutate** an equality field while the object is a key in a `HashSet`/`HashMap`. If you must — remove, mutate, re-add.

> **Interview phrasing**: "Objects used as `HashMap` keys should be effectively immutable on the fields that drive `equals`/`hashCode`."

---

## JPA / Hibernate Entity Caveat

A backend-specific topic that comes up often. The trap is the auto-generated `@Id`:

```java
Customer c = new Customer();   // id is NULL (not saved yet)
Set<Customer> set = new HashSet<>();
set.add(c);                    // hashCode() uses id=null → bucket "null"
customerRepository.save(c);    // Hibernate assigns id = 42 → hash now maps elsewhere
System.out.println(set.contains(c)); // FALSE — id changed from null to 42 after insert
```

This is the mutable-field pitfall in disguise — the `@Id` mutates once (null → 42), enough to lose the object.

**Recommended approach** — use a stable **business key** (a unique, never-changing field like `email` or a UUID set at creation):

```java
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @Column(unique = true, nullable = false, updatable = false)
    private String email;        // stable business key

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Customer)) return false;   // instanceof — Hibernate uses proxy subclasses
        Customer other = (Customer) o;
        return email != null && email.equals(other.email);
    }

    @Override
    public int hashCode() {
        return Customer.class.hashCode();   // CONSTANT — never changes across the entity lifecycle
    }
}
```

> **Why a constant `hashCode()`?** It guarantees the hash never changes when the entity goes from transient (no id) to persistent. All entities land in one bucket and `equals()` (on the business key) sorts them out — you trade hash-spread for correctness, which matters more for entities.

| Do | Don't |
|---|---|
| Use a stable business key (email, UUID at creation) | Use the auto-generated `@Id` (null before save) |
| Use `instanceof` (handles Hibernate proxies) | Use `getClass()` (proxy ≠ entity class) |
| Make `hashCode()` a constant / class hash | Let it depend on a value that changes after save |

---

## Common Mistakes

- **Overriding `equals()` but not `hashCode()`** — the #1 mistake; breaks `HashMap`/`HashSet`.
- **Wrong parameter type** — `equals(Person p)` instead of `equals(Object o)` is an *overload*, not an override. Always add `@Override` so the compiler catches it.
- **Missing null/type check** — causes `ClassCastException`/`NullPointerException`. Use the template.
- **Different fields in `equals()` vs `hashCode()`** — must be based on the same data.
- **Mutable fields in `hashCode()`** — object lost in a set after mutation.
- **Forgetting `@Override`** — a typo like `hashcode()` silently creates a new method, keeping the broken default.

---

## Common Interview Questions

**Q: Difference between `==` and `.equals()`?**
`==` compares references for objects (and values for primitives). `.equals()` compares logical content if overridden — otherwise it falls back to `Object.equals()` (just `==`).

**Q: If you override `equals()`, why must you override `hashCode()`?**
Hash collections use `hashCode()` to pick a bucket, then `equals()` within it. If equal objects produce different hash codes (the default), they land in different buckets, `equals()` never runs, and the object can't be found.

**Q: State the `equals()` contract.**
Reflexive, Symmetric, Transitive, Consistent, Non-null.

**Q: State the `hashCode()` contract.**
Consistent; equal objects must have equal hash codes (mandatory); unequal objects may collide.

**Q: Can two unequal objects have the same hash code?**
Yes — a collision, perfectly legal. `hashCode()` is an `int` (~4 billion values) but objects are infinite, so collisions are unavoidable. `equals()` resolves them. The reverse (equal objects, different hashes) is forbidden.

**Q: Walk through `HashMap.get(key)`.**
Compute `key.hashCode()` → bucket index → go to that bucket → walk entries calling `equals()` → return value on match, else not present.

**Q: `getClass()` vs `instanceof`?**
`getClass()` is strict and symmetry-safe → value objects. `instanceof` allows subclasses/proxies but can break symmetry → use for JPA entities (Hibernate proxies).

**Q: Why 31 in `hashCode()`?**
Odd prime → good bit distribution; multiplying makes field order matter; JVM optimizes `31 * x` to `(x << 5) - x`.

**Q: Simplest correct `equals`/`hashCode` for an immutable value class?**
A `record` (Java 16+) — the compiler generates both, so you can't forget one.

**Q: What goes wrong with a mutable field in `hashCode()`?**
Mutating it while the object is a key changes the hash, so the collection looks in the wrong bucket — the object becomes unreachable via `get`/`contains`/`remove`. Use immutable fields.

**Q: How to implement `equals`/`hashCode` for a JPA entity?**
Don't use the generated `@Id` (null before save, changes on insert). Use a stable business key, `instanceof` for proxy-safety, and a constant `hashCode()`.

**Q: `equals(Person p)` instead of `equals(Object o)`?**
That's an overload, not an override — collections use the default reference equality and ignore your logic. `@Override` prevents this.

**Q: Does `String` override both? Wrapper types?**
Yes — `String` and all wrappers (`Integer`, `Long`, etc.) override both: `equals()` compares values, `hashCode()` is value-derived. That's why they work as `HashMap` keys.

**Q: Can `hashCode()` return a constant?**
Legal (satisfies the contract) but terrible for performance — every object lands in one bucket, degrading lookups to O(n). The exception is JPA entities, where it's the accepted tradeoff for correctness.

---

## Quick Reference Cheat Sheet

```
== vs equals():
  ==        → same object (reference); raw value for primitives
  .equals() → content equality IF overridden (default equals = ==)

THE GOLDEN RULE:
  Override equals() ⇒ MUST override hashCode()  (use the SAME fields in both)

equals() contract (5): Reflexive, Symmetric, Transitive, Consistent, Non-null
hashCode() contract (3):
  consistent | equal ⇒ equal hash (MANDATORY) | unequal ⇒ may collide

One-way street:
  equal objects     → MUST have equal hash codes
  equal hash codes  → does NOT mean equal (collision)

How HashMap/HashSet find things:
  1. hashCode() → pick the bucket (fast)
  2. equals()   → exact match within the bucket (precise)

Classic bug (override equals, forget hashCode):
  add (1,2)     → default hash → bucket A
  contains(1,2) → different default hash → bucket B (empty) → NOT FOUND

equals() template:
  if (this == o) return true;
  if (o == null || getClass() != o.getClass()) return false;
  Type t = (Type) o;
  return Objects.equals(field, t.field) && ...;
hashCode() template:
  return Objects.hash(field1, field2, ...);   // same fields as equals

getClass() vs instanceof:
  getClass() → strict, symmetry-safe → value objects
  instanceof → proxy-friendly        → JPA entities

Why 31: odd prime + good distribution + JVM optimizes 31*x to (x<<5)-x; order matters

Generation: IDE generate (safe) | @EqualsAndHashCode (watch callSuper) | record (auto, Java 16+)

Mutable field pitfall:
  changing a hashCode field while in a Set/Map key → object lost → use final fields

JPA entity rules:
  DON'T use generated @Id (null before save → changes → lost)
  DO   use a stable business key (email / UUID at creation)
  DO   use instanceof (Hibernate proxies)
  DO   make hashCode() a constant (class hash)

Common mistakes:
  - equals without hashCode
  - equals(Person) instead of equals(Object) → use @Override
  - different fields in equals vs hashCode
  - mutable fields in hashCode
```

---

*Last Updated: 2026-06-18*
