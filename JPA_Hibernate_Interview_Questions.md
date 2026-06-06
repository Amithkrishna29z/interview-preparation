# JPA & Hibernate Interview Questions & Study Guide

## Overview

JPA (Java Persistence API) and Hibernate are the backbone of data access in Spring Boot applications. This is one of the most heavily tested topics in Java backend interviews — expect deep questions on entity relationships, fetch strategies, transactions, and the N+1 problem.

---

## Table of Contents

1. [JPA vs Hibernate vs Spring Data JPA](#jpa-vs-hibernate-vs-spring-data-jpa)
2. [Entity Basics](#entity-basics)
3. [Entity Lifecycle States](#entity-lifecycle-states)
4. [Relationships & Mappings](#relationships--mappings)
5. [Fetch Types: LAZY vs EAGER](#fetch-types-lazy-vs-eager)
6. [Cascade Types](#cascade-types)
7. [N+1 Problem & Solutions](#n1-problem--solutions)
8. [JPQL & Criteria API](#jpql--criteria-api)
9. [@Transactional — Propagation & Isolation](#transactional--propagation--isolation)
10. [Hibernate Caching](#hibernate-caching)
11. [Connection Pooling (HikariCP)](#connection-pooling-hikaricp)
12. [Spring Data JPA Repositories](#spring-data-jpa-repositories)
13. [Common Interview Questions](#common-interview-questions)
14. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## JPA vs Hibernate vs Spring Data JPA

**Think of it like building a house:**

- **JPA** = the **blueprint rules** (a standard specification). It says "you MUST have a door, a kitchen, etc." — it defines the rules and interfaces like `@Entity`, `@Id`, `EntityManager`. JPA itself does **nothing** on its own — it's just a contract.
- **Hibernate** = the **actual builder** who follows those blueprints. Hibernate is the real engine that converts your Java objects into SQL and talks to the database. It also adds extra bonus features (caching, HQL, extra annotations, etc.).
- **Spring Data JPA** = a **smart assistant** on top of Hibernate. It hides even more boilerplate. You just write an interface like `UserRepository` and Spring auto-generates all the SQL logic for you.

```
JPA (Java Persistence API)
  └── Specification (interfaces & annotations) — javax.persistence / jakarta.persistence
  └── Defines: @Entity, @Table, @Id, EntityManager, JPQL

Hibernate
  └── Most popular JPA implementation (ORM framework)
  └── Adds: HQL, Session, SessionFactory, caching, extra annotations

Spring Data JPA
  └── Abstraction layer ON TOP of JPA/Hibernate
  └── Adds: JpaRepository, query derivation, @Query, pagination
  └── You write the interface; Spring generates the implementation
```

```
Your Code → Spring Data JPA → JPA API → Hibernate → JDBC → Database
```

**Think of this like**: You tell a waiter (Spring Data JPA) what you want → the waiter tells the kitchen (JPA/Hibernate) → the kitchen prepares the food (SQL) → the food comes from the fridge (database).

---

## Entity Basics

An **Entity** is a Java class that maps to a database table. Each field maps to a column, and each object instance maps to a row.

```java
@Entity                           // tells JPA: "this class maps to a DB table"
@Table(name = "users")            // maps to the "users" table in DB (default = class name lowercase)
public class User {

    @Id                           // this field is the Primary Key
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // DB auto-increments the ID (like AUTO_INCREMENT in MySQL)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    // maps to column "full_name", cannot be null, max 100 characters
    private String name;

    @Column(unique = true, nullable = false)  // must be unique and not null
    private String email;

    @Enumerated(EnumType.STRING)   // stores enum as "ACTIVE" (text) not 0/1 (number) — more readable
    private Status status;

    @CreationTimestamp             // Hibernate auto-sets this when entity is first saved to DB
    private LocalDateTime createdAt;

    @UpdateTimestamp               // Hibernate auto-updates this every time entity is modified
    private LocalDateTime updatedAt;

    @Version                       // used for optimistic locking (prevents two people overwriting each other)
    private Integer version;
}
```

### What each annotation does

- `@Entity` — marks the class as a JPA entity (required)
- `@Table(name = "users")` — specifies the DB table name. Without it, JPA uses the class name
- `@Id` — marks this field as the primary key
- `@GeneratedValue` — tells JPA how to generate the ID automatically
- `@Column` — customizes the column: name, nullability, length, uniqueness
- `@Enumerated(EnumType.STRING)` — stores the enum label as text. Without this, it stores 0, 1, 2 (index-based), which is unreadable and breaks if enum order changes
- `@CreationTimestamp` / `@UpdateTimestamp` — Hibernate-specific; auto-manages timestamps for you
- `@Version` — adds an integer version number to every row. Used for **optimistic locking**

### What is Optimistic Locking (@Version)?

Imagine two users edit the same record at the same time:
- User A loads the record (version = 1)
- User B loads the record (version = 1)
- User A saves changes → DB version becomes 2
- User B tries to save → Hibernate checks "is DB version still 1?" → No, it's 2 → throws `OptimisticLockException`

This prevents silent data overwrites without locking the database row.

### Generation Strategies — how the Primary Key ID gets created

| Strategy | Simple Explanation | Best For |
|---|---|---|
| `IDENTITY` | DB itself auto-increments the ID | MySQL, PostgreSQL (most common) |
| `SEQUENCE` | DB uses a special "counter" sequence object | PostgreSQL, Oracle |
| `TABLE` | JPA uses a separate table to track IDs | Portability (slow, avoid in production) |
| `AUTO` | JPA picks strategy based on your DB | Development only |
| `UUID` | Generates a random unique string like `abc123-...` | Microservices, distributed systems |

---

## Entity Lifecycle States

Every JPA entity (Java object mapped to DB) goes through **4 states** managed by the EntityManager (Hibernate Session).

**Think of it like a job application:**
- `TRANSIENT` = You wrote your name on paper (object exists, but company doesn't know you yet)
- `PERSISTENT` = You're hired (company tracks everything you do)
- `DETACHED` = You're on unpaid leave (company has your record, but isn't tracking you)
- `REMOVED` = You're being let go (company will delete your record soon)

```
┌─────────────┐    persist()    ┌─────────────┐
│  TRANSIENT  │ ──────────────► │  PERSISTENT │
│  (new obj,  │                 │  (managed,  │
│  no id,     │ ◄────────────── │  tracked by │
│  not in DB) │    remove()     │  context)   │
└─────────────┘                 └─────────────┘
                                      │  ▲
                              close() │  │ merge()
                              clear() ▼  │
                                ┌─────────────┐
                                │  DETACHED   │
                                │  (has id,   │
                                │  not tracked│
                                │  by context)│
                                └─────────────┘
                                      │
                               merge() then remove()
                                      │
                                ┌─────────────┐
                                │   REMOVED   │
                                │  (scheduled │
                                │  for DELETE)│
                                └─────────────┘
```

| State | Has ID | In DB | Tracked by EntityManager |
|---|---|---|---|
| **Transient** | No | No | No |
| **Persistent** | Yes | Yes (or pending) | **Yes** |
| **Detached** | Yes | Yes | No |
| **Removed** | Yes | Yes (pending DELETE) | Yes |

```java
// TRANSIENT — just a plain Java object, no ID, DB doesn't know about it yet
User user = new User();
user.setName("Alice");
// Like writing a name on paper — DB has no idea this exists

// PERSISTENT — you told JPA to manage it
entityManager.persist(user);
// Now JPA watches every change you make to 'user'
user.setEmail("alice@example.com"); // JPA automatically syncs this on flush — no need to call save() again!

// DETACHED — session/transaction is closed, JPA stops tracking
entityManager.close();
user.setName("Updated"); // This change is IGNORED — JPA no longer watching
// Like an employee on unpaid leave — changes happen but HR doesn't record them

// MERGE — bring a detached entity back into managed state
User managed = entityManager.merge(user); // returns a NEW managed copy with your changes applied
// Note: merge() returns a new object — don't use the old 'user' variable after this

// REMOVED — scheduled for deletion from DB
entityManager.remove(managed); // queued for DELETE SQL on next flush/commit
```

> **Interview Tip**: In `PERSISTENT` state, you do NOT need to call `.save()` again after changing a field. JPA watches every persistent entity and syncs changes automatically on `flush()` or transaction commit. This is called **dirty checking**.

---

## Relationships & Mappings

### @OneToOne — One entity has exactly one of another

**Real world**: One person has one passport. One passport belongs to one person.

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")  // the "profile_id" FK column lives in the "users" table
    private UserProfile profile;
    // User is the OWNING SIDE — it holds the Foreign Key
}

@Entity
public class UserProfile {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "profile")  // "mappedBy" means: I'm not the FK holder, User is
    private User user;
    // UserProfile is the INVERSE SIDE — just a navigation reference
}
```

---

### @OneToMany / @ManyToOne — One entity has many of another

**Real world**: One customer places many orders. Each order belongs to one customer.

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)   // many Orders belong to ONE customer
    @JoinColumn(name = "customer_id")    // "customer_id" FK column lives in the "orders" table
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    // one Order has MANY items
    // mappedBy = "order" means OrderItem holds the FK (it's the owning side)
    private List<OrderItem> items = new ArrayList<>();
}

@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    // one customer has MANY orders
    // mappedBy = the "customer" field in the Order class
    private List<Order> orders = new ArrayList<>();
}
```

The FK `customer_id` lives in the **orders** table (the "many" side always holds the FK in a `@ManyToOne` relationship).

---

### @ManyToMany — Many entities can relate to many of another

**Real world**: One student can enroll in many courses. One course can have many students.

```java
@Entity
public class Student {
    @Id @GeneratedValue
    private Long id;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",                              // creates a junction/bridge table
        joinColumns = @JoinColumn(name = "student_id"),       // FK pointing back to Student
        inverseJoinColumns = @JoinColumn(name = "course_id")  // FK pointing to Course
    )
    private Set<Course> courses = new HashSet<>();
    // Student is the OWNING SIDE — it has @JoinTable
}

@Entity
public class Course {
    @Id @GeneratedValue
    private Long id;

    @ManyToMany(mappedBy = "courses")   // Student owns the relationship
    private Set<Student> students = new HashSet<>();
    // Course is the INVERSE SIDE
}
```

**Why a junction table?** SQL cannot store a list inside a column. So JPA creates a separate bridge table `student_course` with two FK columns: `student_id` and `course_id`. Each row in this table represents one enrollment.

---

### Owning Side vs Inverse Side — Key Concept

This is one of the most confusing parts of JPA. Here's the simple rule:

- **Owning side**: Has `@JoinColumn` or `@JoinTable`. JPA writes FK changes **from this side only**. If you want the relationship saved, update this side.
- **Inverse side**: Has `mappedBy`. JPA **ignores** changes made only on this side. It's just a read-only navigation pointer back to the owning side.

> **Common Mistake**: If you add a course only to `course.getStudents()` (inverse side) without also adding to `student.getCourses()` (owning side), Hibernate will **not** save the relationship to the DB. Always update the owning side.

---

## Fetch Types: LAZY vs EAGER

**LAZY** = Don't load related data until I actually ask for it.
**EAGER** = Always load related data immediately, even if I don't need it.

```java
// LAZY — SQL fires only when you actually call user.getOrders()
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;

// EAGER — customer is always loaded with the order, no matter what
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

**Real world analogy**:
- **EAGER** = When you order a combo meal, fries always come with it (even if you don't want them today).
- **LAZY** = Fries only arrive when you specifically ask for them.

### Default Fetch Types (memorize this for interviews!)

| Annotation | Default Fetch | Why |
|---|---|---|
| `@OneToMany` | **LAZY** | Could be thousands of records — don't load by default |
| `@ManyToMany` | **LAZY** | Same reason |
| `@ManyToOne` | **EAGER** | Single related entity — usually needed |
| `@OneToOne` | **EAGER** | Single related entity — usually needed |

> **Best Practice**: Always use `LAZY` for collections. Use `JOIN FETCH` in JPQL when you need associations. `EAGER` on collections causes serious performance problems (loads everything even when you don't need it).

---

### LazyInitializationException — the most common JPA error

This happens when you try to access a LAZY collection **after the Hibernate session has already been closed**.

```java
// WRONG — this breaks at runtime
User user = userRepository.findById(1L).get();
// Session closes automatically here (the method returned, transaction ended)
user.getOrders().size(); // BOOM! LazyInitializationException — session is gone, can't load orders
```

**Why?** Hibernate needs an open session to go back to the DB and load the lazy collection. Once the session is closed, it can't do that.

```java
// FIX 1: JOIN FETCH — load orders together with user in one SQL query
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
// One SQL: SELECT u.*, o.* FROM users u JOIN orders o ON u.id = o.user_id WHERE u.id = ?

// FIX 2: Keep session open with @Transactional
@Transactional
public void processUser(Long id) {
    User user = userRepository.findById(id).get();
    user.getOrders().size(); // Works! Session is still open inside @Transactional
}

// FIX 3: @EntityGraph — Spring auto-generates JOIN FETCH for you
@EntityGraph(attributePaths = {"orders"})
User findWithOrdersById(Long id);

// FIX 4: DTO projection — only fetch the fields you need (no lazy loading issue)
```

---

## Cascade Types

**Cascading** means: "When I do something to the parent, automatically do the same to the children."

**Real world**: When a company closes (parent), all employees (children) are also let go. You don't have to fire each one individually.

```java
@OneToMany(cascade = CascadeType.ALL)
// ALL = persist/merge/remove/refresh/detach all propagate to children automatically
```

| Cascade Type | What it does |
|---|---|
| `PERSIST` | Saving parent also saves children (no need to save children separately) |
| `MERGE` | Merging parent also merges children |
| `REMOVE` | Deleting parent also deletes children |
| `REFRESH` | Refreshing parent also refreshes children from DB |
| `DETACH` | Detaching parent also detaches children |
| `ALL` | All of the above |

---

### `CascadeType.REMOVE` vs `orphanRemoval = true`

This is a common interview question. They are similar but NOT the same.

```java
// CascadeType.REMOVE: Children are deleted ONLY when the parent itself is deleted
@OneToMany(cascade = CascadeType.REMOVE)
private List<OrderItem> items;
// If you do: orderRepo.delete(order) → all items are also deleted
// But if you do: order.getItems().remove(item) → item is NOT deleted from DB!

// orphanRemoval = true: Children are deleted when removed from collection OR when parent is deleted
@OneToMany(orphanRemoval = true)
private List<OrderItem> items;
// If you do: order.getItems().remove(item) → item IS deleted from DB (it became an "orphan")
// Also: if you delete the parent → children are deleted too (implies CascadeType.REMOVE)
```

**Simple rule**: `orphanRemoval` is stronger. Use it when a child **cannot logically exist without its parent** (e.g., `OrderItem` makes no sense without an `Order`).

> **Interview Tip**: `orphanRemoval = true` implies `CascadeType.REMOVE`. It also fires DELETE when an item is simply removed from the parent's collection, not just when the parent is deleted.

---

## N+1 Problem & Solutions

The N+1 problem is the **#1 most asked Hibernate interview topic**.

### The Problem

```java
List<Order> orders = orderRepository.findAll();
// SQL fired: SELECT * FROM orders  →  1 query, returns 100 orders

for (Order order : orders) {
    System.out.println(order.getCustomer().getName());
    // SQL fired for EACH order: SELECT * FROM customers WHERE id = ?
    // → runs 100 times, once per order!
}
// Total: 1 + 100 = 101 queries  ← this is the N+1 problem
```

**Analogy**: You need to deliver 100 packages to different addresses. Instead of loading all packages into the truck at once and making one optimized trip, you make 100 separate trips to the warehouse — one per package. Incredibly inefficient.

---

### Solution 1: JOIN FETCH in JPQL

```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
// SQL generated: SELECT o.*, c.* FROM orders o JOIN customers c ON o.customer_id = c.id
// 1 query fetches orders AND customers together — problem solved!
```

---

### Solution 2: @EntityGraph

```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findAll();
// Spring Data JPA auto-generates JOIN FETCH for "customer" and "items"
// Still just 1 query — simpler syntax than writing @Query manually
```

---

### Solution 3: @BatchSize (Hibernate-specific)

```java
@OneToMany
@BatchSize(size = 20)  // load up to 20 collections in a single IN() query
private List<OrderItem> items;
// Instead of: SELECT * FROM order_items WHERE order_id = 1
//             SELECT * FROM order_items WHERE order_id = 2
//             ... (N queries)
// Becomes: SELECT * FROM order_items WHERE order_id IN (1, 2, 3, ..., 20)
// 100 orders → 5 batches → only 5 queries instead of 100!
```

---

### Solution 4: DTO Projection

```java
@Query("SELECT new com.example.OrderSummary(o.id, c.name) FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();
// Only fetches o.id and c.name — not the full entity graph
// Most efficient when you only need a subset of fields
```

### How to detect N+1 in your logs

```yaml
# application.properties — add these to see all SQL
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

If you see the same query repeating 100 times with different IDs in the console → you have N+1.

---

## JPQL & Criteria API

### JPQL (Java Persistence Query Language)

JPQL looks like SQL but operates on **Java entity classes and their fields**, not DB tables and columns. It's database-independent — works on MySQL, PostgreSQL, Oracle without changes.

```java
// SQL:   SELECT * FROM users WHERE email = ?
// JPQL:  SELECT u FROM User u WHERE u.email = :email
// "User" = Java class name, "u.email" = Java field name (not column name)

@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// JOIN FETCH — load items together with orders (prevents N+1)
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId")
List<Order> findOrdersWithItems(@Param("customerId") Long customerId);

// Multiple named parameters
@Query("SELECT u FROM User u WHERE u.status = :status AND u.age > :minAge")
List<User> findActiveAdults(@Param("status") Status status, @Param("minAge") int minAge);

// Pagination — pass Pageable parameter, Spring handles the rest
@Query("SELECT p FROM Product p WHERE p.price < :maxPrice")
Page<Product> findCheapProducts(@Param("maxPrice") BigDecimal price, Pageable pageable);

// Native SQL — for DB-specific features JPQL can't handle (window functions, JSON, etc.)
@Query(value = "SELECT * FROM users WHERE created_at > NOW() - INTERVAL 7 DAY", nativeQuery = true)
List<User> findRecentUsers();

// UPDATE or DELETE — requires both @Modifying and @Transactional
@Modifying
@Transactional
@Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
int updateStatus(@Param("id") Long id, @Param("status") Status status);
```

---

### Query Derivation (Spring Data JPA) — Spring magic from method names

Spring Data JPA reads your **method name** and auto-generates the SQL. No `@Query` annotation needed at all.

```java
List<User> findByEmail(String email);
// → SELECT * FROM users WHERE email = ?

List<User> findByNameAndStatus(String name, Status status);
// → SELECT * FROM users WHERE name = ? AND status = ?

List<User> findByAgeBetween(int min, int max);
// → SELECT * FROM users WHERE age BETWEEN ? AND ?

List<User> findByNameContainingIgnoreCase(String keyword);
// → SELECT * FROM users WHERE UPPER(name) LIKE UPPER('%keyword%')

List<User> findByOrderByCreatedAtDesc();
// → SELECT * FROM users ORDER BY created_at DESC

Optional<User> findFirstByOrderByCreatedAtDesc();
// → SELECT * FROM users ORDER BY created_at DESC LIMIT 1

boolean existsByEmail(String email);
// → SELECT COUNT(*) > 0 FROM users WHERE email = ?

long countByStatus(Status status);
// → SELECT COUNT(*) FROM users WHERE status = ?
```

**How it works**: Spring parses your method name word by word:
- `findBy` → WHERE clause starts
- `And` / `Or` → AND / OR
- `Between` → BETWEEN
- `Containing` → LIKE '%value%'
- `IgnoreCase` → case-insensitive comparison
- `OrderBy` + `Desc` / `Asc` → ORDER BY

---

### Criteria API — type-safe dynamic queries

Use Criteria API when the query structure itself is **dynamic** (e.g., search filters where some are optional). Writing dynamic JPQL as string concatenation is messy and error-prone.

```java
public List<User> searchUsers(String name, Status status, Integer minAge) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder(); // the query builder tool
    CriteriaQuery<User> query = cb.createQuery(User.class);  // SELECT ... FROM User
    Root<User> root = query.from(User.class);                // FROM User u  (root = the "u" alias)

    List<Predicate> predicates = new ArrayList<>();          // builds the WHERE clause dynamically

    if (name != null) {
        // WHERE LOWER(u.name) LIKE '%name%'
        predicates.add(cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%"));
    }
    if (status != null) {
        // WHERE u.status = ?
        predicates.add(cb.equal(root.get("status"), status));
    }
    if (minAge != null) {
        // WHERE u.age >= ?
        predicates.add(cb.greaterThanOrEqualTo(root.get("age"), minAge));
    }

    query.where(predicates.toArray(new Predicate[0]));  // combine all WHERE conditions with AND
    query.orderBy(cb.desc(root.get("createdAt")));      // ORDER BY createdAt DESC

    return entityManager.createQuery(query).getResultList();
}
```

**Why Criteria API?** If all three filters are optional, you'd have to dynamically build a SQL/JPQL string like `"SELECT ... WHERE " + (name != null ? "name LIKE ? AND" : "") + ...`. That's ugly and bug-prone. Criteria API lets you add/skip conditions in pure Java code — type-safe, refactor-friendly, and clean.

---

## @Transactional — Propagation & Isolation

### @Transactional Basics

A **transaction** means: all operations inside either ALL succeed together or ALL fail together (atomicity).

```java
@Service
public class OrderService {

    @Transactional  // opens a transaction before the method, commits (or rolls back) after it returns
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);         // step 1
        paymentService.charge(order);        // step 2 — if this throws, step 1 is ROLLED BACK too
        return order;
    }
}
```

**Analogy**: Like buying a concert ticket — you pay AND get the ticket. If payment fails, you don't get the ticket. If the ticket can't be issued, you don't pay. Either both succeed or neither does.

---

### Transaction Propagation — what happens when one `@Transactional` calls another

```java
@Transactional(propagation = Propagation.REQUIRED)  // default
public void outer() {
    innerRequired();     // joins the outer transaction — they share the same fate
    innerRequiresNew();  // starts its OWN fresh transaction — completely independent
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void innerRequiresNew() {
    // if this fails → only THIS rolls back. The outer transaction is NOT affected.
    // if outer fails → this has already committed independently.
}
```

| Propagation | Simple Explanation |
|---|---|
| `REQUIRED` (default) | Join the existing transaction. If none exists, create one. Both share the same fate — if either fails, both roll back. |
| `REQUIRES_NEW` | Always start a fresh transaction. Suspend the existing one. They're completely independent — one can commit while the other rolls back. |
| `NESTED` | Create a "save point" inside the existing transaction. If this inner part fails, roll back only to the save point (outer continues). |
| `SUPPORTS` | Join if one exists. Run without a transaction if none. |
| `NOT_SUPPORTED` | Always suspend any existing transaction. Run without one. |
| `MANDATORY` | Must be called inside an existing transaction. Throws if there isn't one. |
| `NEVER` | Must NOT be in a transaction. Throws if there is one. |

---

### Transaction Isolation Levels — how much one transaction can "see" other concurrent transactions

Multiple users can query/update the DB at the same time. Isolation controls what each transaction sees.

```java
@Transactional(isolation = Isolation.READ_COMMITTED) // most common default
```

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | **Prevented** | Possible | Possible |
| `REPEATABLE_READ` | **Prevented** | **Prevented** | Possible |
| `SERIALIZABLE` | **Prevented** | **Prevented** | **Prevented** |

**Three concurrency problems explained simply:**

- **Dirty Read**: You read data that another transaction wrote but hasn't committed yet. That transaction could still roll back — so you read "garbage" data that may never exist.
- **Non-Repeatable Read**: You read the same row twice in the same transaction, but another transaction modified it between your two reads — you get different values each time.
- **Phantom Read**: You run the same query twice in the same transaction, but another transaction inserted or deleted rows between your reads — you get different row counts.

Higher isolation = safer data, but slower performance. `READ_COMMITTED` is the sweet spot for most applications.

---

### @Transactional Gotchas (these appear in almost every interview!)

```java
// GOTCHA 1: Self-invocation bypasses the Spring proxy — @Transactional has NO effect
@Service
public class MyService {
    public void outer() {
        inner(); // calls this.inner() directly — skips Spring's proxy wrapper!
        // @Transactional on inner() is COMPLETELY IGNORED
    }

    @Transactional
    public void inner() { ... }
}
// WHY: Spring makes @Transactional work by wrapping your bean in a proxy class.
// @Transactional only activates when called THROUGH the proxy (i.e., from outside the class).
// Calling inner() from within the same class bypasses the proxy entirely.
// FIX: Inject MyService into itself and call through the injected reference, or use ApplicationContext.getBean()

// GOTCHA 2: Checked exceptions do NOT trigger rollback by default
@Transactional
public void process() throws IOException {
    // Even if IOException is thrown, the transaction COMMITS — silent data bug!
}
// WHY: @Transactional only rolls back on RuntimeException (unchecked) by default.
// FIX:
@Transactional(rollbackFor = IOException.class)

// GOTCHA 3: @Transactional on private methods is silently ignored
@Transactional
private void doWork() { }  // Spring AOP cannot intercept private methods — no transaction
// FIX: Make the method public
```

---

## Hibernate Caching

### First-Level Cache (Session Cache) — always on

The L1 cache is built into every Hibernate Session (EntityManager). Within one transaction, loading the same entity twice only hits the DB once.

```java
User u1 = userRepo.findById(1L).get(); // SQL fired: SELECT * FROM users WHERE id = 1
User u2 = userRepo.findById(1L).get(); // NO SQL — served from L1 cache!
assert u1 == u2; // they are literally the SAME Java object in memory
```

- **Scope**: One Session / one transaction only. When the transaction ends, the L1 cache is cleared.
- **Cannot be disabled** — it's always active.
- **Analogy**: Within one request, if you ask for the same user twice, Hibernate remembers the first lookup and returns it instantly without hitting the DB again.

---

### Second-Level Cache (Shared Cache) — opt-in

The L2 cache is shared across all sessions and transactions in the application. A `Product` loaded by User A is still cached when User B requests it seconds later.

```java
// Step 1: Annotate entity to opt-in to L2 cache
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // allows both reads and writes with cache sync
public class Product { ... }
```

```yaml
# application.properties — enable L2 cache and configure provider
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheCacheRegionFactory
```

- **NOT enabled by default** — requires explicit configuration + a cache provider (EhCache, Caffeine, Hazelcast)
- **Scope**: Entire application lifetime
- **Best for**: Frequently read, rarely modified data (e.g., product catalog, config data)

---

### Query Cache

Caches the **result set of a query** (list of IDs), not the entities themselves. Works together with L2 cache.

```java
@Query("SELECT p FROM Product p WHERE p.category = :cat")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(@Param("cat") String category);
// Same category query → returns cached result without hitting DB
```

**Cache Summary:**

| Cache | Scope | Default | Use For |
|---|---|---|---|
| L1 (Session) | One transaction | Always on | Repeated access within same request |
| L2 (Shared) | Whole application | Opt-in | Frequently read, rarely changed data |
| Query Cache | Whole application | Opt-in | Repeated identical queries with same params |

---

## Connection Pooling (HikariCP)

**The problem without pooling**: Creating a new DB connection is expensive (TCP handshake, authentication, SSL negotiation). If your app creates a new connection per request and closes it after — you waste time on every single request.

**The solution**: HikariCP maintains a **pool** of pre-established connections. When a request needs one, it borrows a connection from the pool. When the request is done, it returns the connection (NOT closes it) so the next request can reuse it.

**Analogy**: A taxi fleet vs. ordering a new car for every trip. The pool is the fleet — cars are ready and waiting. You "borrow" one, use it, and return it.

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
    hikari:
      pool-name: MyAppPool
      minimum-idle: 5           # always keep at least 5 connections ready (warm standby)
      maximum-pool-size: 20     # never open more than 20 connections at once
      idle-timeout: 600000      # remove an idle connection after 10 minutes (600,000 ms)
      connection-timeout: 30000 # if no connection is free within 30 seconds → throw exception
      max-lifetime: 1800000     # force-recycle connections older than 30 minutes (prevents stale connections)
      leak-detection-threshold: 2000 # log a warning if a connection is held for more than 2 seconds (indicates a leak)
```

> **Interview Tip**: Pool size rule of thumb = `(CPU cores × 2) + number of disk spindles`. Don't go too high — too many connections cause context-switching overhead. Too small causes request queuing and timeouts.

---

## Spring Data JPA Repositories

Spring Data JPA lets you define a repository interface and it auto-generates the implementation. You get full CRUD for free.

```java
// JpaRepository<EntityType, PrimaryKeyType>
public interface UserRepository extends JpaRepository<User, Long> {

    // Query derivation — Spring auto-generates SQL from method name
    Optional<User> findByEmail(String email);
    List<User> findByStatusOrderByCreatedAtDesc(Status status);
    boolean existsByEmail(String email);
    long countByStatus(Status status);

    // Custom JPQL — load roles together in one query (prevents N+1)
    @Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.id = :id")
    Optional<User> findByIdWithRoles(@Param("id") Long id);

    // Projection — return only specific fields, not the full entity (more efficient)
    List<UserSummary> findByStatus(Status status);

    // Pagination — returns a Page object with content + metadata
    Page<User> findByStatus(Status status, Pageable pageable);

    // Bulk delete — requires @Modifying + @Transactional
    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.status = :status")
    void deleteByStatus(@Param("status") Status status);
}
```

### Projection Interface — fetch only the fields you need

```java
// Define an interface with getters for the fields you want
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
    // Only these 3 columns are fetched from DB — not all 30+ columns in the User entity
}
```

**Why projections?** If your `User` entity has 30 columns but you only need 3 fields for a dropdown list, loading all 30 is wasteful. Projections tell Hibernate to only `SELECT` those specific columns — faster query, less memory.

---

### Pagination

```java
Page<User> page = userRepository.findByStatus(
    Status.ACTIVE,
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
    // page 0 = first page, 20 items per page, sorted newest first
);

page.getContent();        // List<User> — the 20 users on this page
page.getTotalPages();     // total number of pages (e.g., 5 if 100 total users, 20 per page)
page.getTotalElements();  // total count of ALL matching users across all pages (e.g., 100)
page.getNumber();         // current page number (0-indexed)
page.isFirst();           // true if this is page 0
page.isLast();            // true if this is the last page
```

---

### Custom Repository Implementation — when Spring's magic isn't enough

```java
// Step 1: Define your custom methods in an interface
public interface UserRepositoryCustom {
    List<User> searchUsers(String keyword, Status status);
}

// Step 2: Implement it — class name MUST be exactly "<RepositoryName>Impl"
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    @PersistenceContext
    private EntityManager em;  // Spring injects the EntityManager

    @Override
    public List<User> searchUsers(String keyword, Status status) {
        // Write Criteria API or custom JPQL here
    }
}

// Step 3: Extend BOTH interfaces in your repository
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {
    // Now you get: built-in CRUD + findBy methods + your custom searchUsers()
}
```

---

## Common Interview Questions

### Q: What is the N+1 problem and how do you fix it?

The N+1 problem occurs when fetching a list of N entities triggers N additional queries to load their associations. Example: 1 query for 100 orders + 100 separate queries to load each order's customer = 101 queries total.

**Fix**: Use `JOIN FETCH` in JPQL, `@EntityGraph`, or `@BatchSize`. All three reduce N+1 queries to 1 (or a few batched) queries.

---

### Q: What is the difference between `save()` and `saveAndFlush()`?

- `save()` — Stages the entity for persistence. The actual SQL `INSERT`/`UPDATE` may be deferred until the transaction commits or Hibernate decides to flush. The write is batched.
- `saveAndFlush()` — Forces the SQL to fire immediately within the current transaction. Use when the very next query in the same transaction must see the just-saved data (e.g., you save a record and then query for it by a field).

---

### Q: Explain `@Transactional` propagation REQUIRED vs REQUIRES_NEW.

- `REQUIRED`: If a transaction already exists, the method joins it — they share the same transaction. If either method fails, both roll back together.
- `REQUIRES_NEW`: Always creates a brand new transaction, suspending any existing one. They are fully independent — inner can commit while outer rolls back, or vice versa.

**Use `REQUIRES_NEW`** when you want to persist something (like an audit log) even if the main transaction rolls back.

---

### Q: What is the difference between `CascadeType.REMOVE` and `orphanRemoval = true`?

- `CascadeType.REMOVE`: Children are deleted **only** when the parent entity itself is deleted.
- `orphanRemoval = true`: Children are also deleted when they are removed from the parent's collection (even without deleting the parent). It also implies `CascadeType.REMOVE`.

**Simple rule**: If a child cannot logically exist without its parent (like `OrderItem` without `Order`), use `orphanRemoval = true`.

---

### Q: What causes `LazyInitializationException`?

Accessing a `LAZY` association after the Hibernate Session (EntityManager) has been closed.

**Fixes**: `JOIN FETCH` in query, `@EntityGraph`, `@Transactional` around the caller, DTO projections. Avoid `open-session-in-view` (anti-pattern — keeps DB connections open for the entire HTTP request).

---

### Q: What is the difference between JPQL and native SQL?

- **JPQL**: Object-oriented query language. Queries against Java entity classes and fields. Database-independent — works on any DB. Results are automatically mapped to entities.
- **Native SQL**: Actual SQL for your specific database. Required for DB-specific features like window functions, stored procedures, JSON operations. Use `nativeQuery = true`. Results need manual mapping unless you use projections.

---

### Q: How does Hibernate dirty checking work?

When you load an entity into `PERSISTENT` state, Hibernate takes a **snapshot** of all its field values. At `flush()` time (before commit), Hibernate compares the current field values against that snapshot. If any field changed, it automatically generates an `UPDATE` SQL statement.

You **never need to call `save()` again** after modifying a loaded entity — Hibernate detects and syncs changes automatically.

---

### Q: What is the difference between L1 and L2 cache?

- **L1 cache**: Scoped to one Session (one transaction). Always active. Within the same transaction, loading the same entity twice returns the same Java object (no DB hit on second access).
- **L2 cache**: Shared across all sessions and transactions in the entire application. Opt-in — must be explicitly enabled and configured with a provider like EhCache. Survives across requests.

---

## Quick Reference Cheat Sheet

```
JPA          → specification (interfaces & rules)
Hibernate    → JPA implementation + extra features (HQL, caching, etc.)
Spring Data  → abstraction over JPA (JpaRepository, query derivation)

Entity States:
  Transient  → new Java object, no id, not in DB, not tracked
  Persistent → has id, tracked by JPA, auto-synced on flush
  Detached   → has id, not tracked (after session close/clear)
  Removed    → queued for DELETE on next flush

Fetch Types:
  LAZY  → load on access (default for @OneToMany, @ManyToMany)
  EAGER → always load with parent (default for @ManyToOne, @OneToOne)
  Best practice: LAZY for everything — use JOIN FETCH when needed

N+1 Fix:
  → JOIN FETCH in JPQL
  → @EntityGraph
  → @BatchSize

@Transactional Gotchas:
  rollback  → only on RuntimeException by default; use rollbackFor for checked
  self-call → bypasses proxy; @Transactional has no effect
  private   → @Transactional ignored on private methods

Propagation:
  REQUIRED     → join existing or create new (both share fate)
  REQUIRES_NEW → always create new, suspend existing (independent)

Cascade:
  REMOVE        → delete children when parent is deleted
  orphanRemoval → delete children removed from parent's collection (stronger)

Isolation Problems:
  Dirty Read           → reading uncommitted data
  Non-Repeatable Read  → same row, different values in same transaction
  Phantom Read         → same query, different rows in same transaction

Caching:
  L1 → session scope, always on (same session = same object reference)
  L2 → application scope, opt-in (EhCache, Caffeine — survives across requests)

HikariCP → default connection pool in Spring Boot
  maximum-pool-size → cap concurrent DB connections (default: 10)
  Rule of thumb: (CPU cores × 2) + disk spindles
```

---

*Last Updated: 2026-06-06*
