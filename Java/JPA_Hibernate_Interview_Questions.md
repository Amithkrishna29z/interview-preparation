# JPA & Hibernate Interview Questions & Study Guide

## Overview

JPA (Java Persistence API) and Hibernate are the backbone of data access in Spring Boot applications. This is one of the most heavily tested topics in Java backend interviews — expect questions on entity relationships, fetch strategies, transactions, and the N+1 problem.

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

- **JPA** = specification (interfaces & rules). Defines `@Entity`, `@Id`, `EntityManager`. Does nothing on its own.
- **Hibernate** = the real engine that implements JPA — converts Java objects to SQL. Adds HQL, caching, and extra annotations.
- **Spring Data JPA** = abstraction on top of Hibernate. You write an interface like `UserRepository`; Spring generates the implementation.

```
Your Code → Spring Data JPA → JPA API → Hibernate → JDBC → Database
```

```
JPA (Java Persistence API)
  └── Specification — javax.persistence / jakarta.persistence
  └── Defines: @Entity, @Table, @Id, EntityManager, JPQL

Hibernate
  └── Most popular JPA implementation
  └── Adds: HQL, Session, SessionFactory, caching

Spring Data JPA
  └── Abstraction ON TOP of JPA/Hibernate
  └── Adds: JpaRepository, query derivation, @Query, pagination
```

---

## Entity Basics

An **Entity** is a Java class that maps to a database table. Each field maps to a column; each object instance maps to a row.

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // DB auto-increments
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)  // stores "ACTIVE" not 0/1 — use this always
    private Status status;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;

    @Version  // optimistic locking
    private Integer version;
}
```

### Key Annotations

- `@Entity` / `@Table` — marks class as JPA entity; maps to a DB table
- `@Id` + `@GeneratedValue` — primary key and auto-generation strategy
- `@Column` — customizes column name, nullability, length, uniqueness
- `@Enumerated(EnumType.STRING)` — stores enum as text; without this, index (0,1,2) is stored and breaks if enum order changes
- `@Version` — adds a version number per row for **optimistic locking**: if two users edit the same record concurrently, the second save throws `OptimisticLockException` instead of silently overwriting

### Generation Strategies

| Strategy | Use For |
|---|---|
| `IDENTITY` | MySQL, PostgreSQL (most common) |
| `SEQUENCE` | PostgreSQL, Oracle |
| `AUTO` | Development only |
| `UUID` | Microservices, distributed systems |

---

## Entity Lifecycle States

Every JPA entity goes through **4 states** managed by the EntityManager.

```
TRANSIENT  ──persist()──► PERSISTENT ──close()/clear()──► DETACHED
              ◄──remove()──                                  │
                                                        merge() → back to PERSISTENT
                                REMOVED ◄──remove(managed)──
```

| State | Has ID | In DB | Tracked |
|---|---|---|---|
| **Transient** | No | No | No |
| **Persistent** | Yes | Yes (or pending) | Yes |
| **Detached** | Yes | Yes | No |
| **Removed** | Yes | Yes (pending DELETE) | Yes |

```java
User user = new User();           // TRANSIENT
entityManager.persist(user);      // PERSISTENT — JPA now watches all changes
user.setEmail("a@b.com");         // auto-synced on flush, no save() needed!

entityManager.close();            // DETACHED — changes after this are ignored
User managed = entityManager.merge(user); // back to PERSISTENT (returns new object)
entityManager.remove(managed);    // REMOVED — DELETE on next flush
```

> **Interview Tip**: In `PERSISTENT` state you do NOT need to call `.save()` again after changing a field. JPA detects changes via **dirty checking** and syncs automatically on flush.

---

## Relationships & Mappings

### @OneToOne

```java
@Entity
public class User {
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")  // FK lives in users table — owning side
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @OneToOne(mappedBy = "profile")  // inverse side — no FK here
    private User user;
}
```

### @OneToMany / @ManyToOne

```java
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")  // FK lives in orders table
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

@Entity
public class Customer {
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
}
```

### @ManyToMany

```java
@Entity
public class Student {
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();  // owning side
}

@Entity
public class Course {
    @ManyToMany(mappedBy = "courses")  // inverse side
    private Set<Student> students = new HashSet<>();
}
```

### Owning Side vs Inverse Side

- **Owning side**: Has `@JoinColumn` or `@JoinTable`. JPA writes FK changes **from this side only**.
- **Inverse side**: Has `mappedBy`. JPA ignores changes made here — it's a read-only navigation reference.

> **Common Mistake**: Adding to the inverse-side collection without also adding to the owning side means Hibernate will NOT save the relationship. Always update the owning side.

---

## Fetch Types: LAZY vs EAGER

- **LAZY** = load related data only when accessed
- **EAGER** = always load related data with the parent

```java
@OneToMany(fetch = FetchType.LAZY)   // load orders only when getOrders() is called
private List<Order> orders;

@ManyToOne(fetch = FetchType.EAGER)  // customer always loaded with order
private Customer customer;
```

### Default Fetch Types

| Annotation | Default |
|---|---|
| `@OneToMany` | **LAZY** |
| `@ManyToMany` | **LAZY** |
| `@ManyToOne` | **EAGER** |
| `@OneToOne` | **EAGER** |

> **Best Practice**: Always use `LAZY` for collections. Use `JOIN FETCH` when you need associations.

### LazyInitializationException

Occurs when you access a LAZY collection after the Hibernate session has closed.

```java
// WRONG
User user = userRepository.findById(1L).get();
user.getOrders().size(); // BOOM — session already closed

// FIX 1: JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);

// FIX 2: @Transactional keeps session open
@Transactional
public void processUser(Long id) {
    User user = userRepository.findById(id).get();
    user.getOrders().size(); // works — session still open
}

// FIX 3: @EntityGraph
@EntityGraph(attributePaths = {"orders"})
User findWithOrdersById(Long id);
```

---

## Cascade Types

Cascading means: "When I act on the parent, automatically do the same to the children."

| Cascade Type | What it does |
|---|---|
| `PERSIST` | Saving parent also saves children |
| `MERGE` | Merging parent also merges children |
| `REMOVE` | Deleting parent also deletes children |
| `REFRESH` | Refreshing parent also refreshes children |
| `DETACH` | Detaching parent also detaches children |
| `ALL` | All of the above |

### `CascadeType.REMOVE` vs `orphanRemoval = true`

```java
// CascadeType.REMOVE: children deleted only when parent is deleted
@OneToMany(cascade = CascadeType.REMOVE)
private List<OrderItem> items;
// order.getItems().remove(item) → item NOT deleted from DB

// orphanRemoval: children deleted when removed from collection OR when parent deleted
@OneToMany(orphanRemoval = true)
private List<OrderItem> items;
// order.getItems().remove(item) → item IS deleted from DB
```

> **Rule**: `orphanRemoval` is stronger and implies `CascadeType.REMOVE`. Use it when a child cannot exist without its parent (e.g., `OrderItem` without `Order`).

---

## N+1 Problem & Solutions

The N+1 problem is the **#1 most asked Hibernate interview topic**.

```java
List<Order> orders = orderRepository.findAll(); // 1 query → 100 orders
for (Order order : orders) {
    order.getCustomer().getName(); // 1 query per order → 100 more queries
}
// Total: 101 queries — classic N+1
```

### Solution 1: JOIN FETCH

```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
// 1 query fetches orders AND customers together
```

### Solution 2: @EntityGraph

```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findAll();
// Spring auto-generates JOIN FETCH — cleaner syntax
```

### Solution 3: @BatchSize

```java
@OneToMany
@BatchSize(size = 20)
private List<OrderItem> items;
// Instead of N queries: SELECT ... WHERE order_id IN (1,2,...,20)
// 100 orders → 5 batches instead of 100 queries
```

### Solution 4: DTO Projection

```java
@Query("SELECT new com.example.OrderSummary(o.id, c.name) FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();
```

### How to detect N+1

```yaml
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
```

If you see the same query repeating with different IDs → you have N+1.

---

## JPQL & Criteria API

### JPQL

JPQL queries **Java entity classes and fields**, not DB tables. Database-independent.

```java
// SQL uses table/column names; JPQL uses class/field names
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId")
List<Order> findOrdersWithItems(@Param("customerId") Long customerId);

// UPDATE/DELETE require both @Modifying and @Transactional
@Modifying
@Transactional
@Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
int updateStatus(@Param("id") Long id, @Param("status") Status status);

// Native SQL for DB-specific features
@Query(value = "SELECT * FROM users WHERE created_at > NOW() - INTERVAL 7 DAY", nativeQuery = true)
List<User> findRecentUsers();
```

### Query Derivation

Spring generates SQL from method names automatically:

```java
List<User> findByEmail(String email);
// → SELECT * FROM users WHERE email = ?

List<User> findByNameAndStatus(String name, Status status);
List<User> findByAgeBetween(int min, int max);
List<User> findByNameContainingIgnoreCase(String keyword);
List<User> findByOrderByCreatedAtDesc();
boolean existsByEmail(String email);
long countByStatus(Status status);
```

### Criteria API

Use for **dynamic queries** where filters are optional (avoids messy string concatenation).

```java
public List<User> searchUsers(String name, Status status, Integer minAge) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    List<Predicate> predicates = new ArrayList<>();
    if (name != null)   predicates.add(cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%"));
    if (status != null) predicates.add(cb.equal(root.get("status"), status));
    if (minAge != null) predicates.add(cb.greaterThanOrEqualTo(root.get("age"), minAge));

    query.where(predicates.toArray(new Predicate[0]));
    query.orderBy(cb.desc(root.get("createdAt")));
    return entityManager.createQuery(query).getResultList();
}
```

---

## @Transactional — Propagation & Isolation

### Basics

```java
@Transactional  // opens transaction before method; commits or rolls back after
public Order createOrder(OrderRequest request) {
    Order order = new Order(request);
    orderRepository.save(order);   // step 1
    paymentService.charge(order);  // if this throws, step 1 is rolled back too
    return order;
}
```

### Propagation

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing transaction or create new. Both share the same fate. |
| `REQUIRES_NEW` | Always create a fresh transaction; suspend the existing one. They're independent. |
| `NESTED` | Save point inside the existing transaction; inner failure rolls back to save point only. |
| `SUPPORTS` | Join if one exists; run without if none. |
| `MANDATORY` | Must be called inside an existing transaction; throws if not. |
| `NEVER` | Must NOT be in a transaction; throws if one exists. |

**Use `REQUIRES_NEW`** when you want something to persist (e.g., audit log) even if the main transaction rolls back.

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE_READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

- **Dirty Read**: Reading uncommitted data that may roll back
- **Non-Repeatable Read**: Same row gives different values in the same transaction
- **Phantom Read**: Same query returns different row counts in the same transaction

`READ_COMMITTED` is the sweet spot for most applications.

### @Transactional Gotchas

```java
// GOTCHA 1: Self-invocation bypasses Spring proxy — @Transactional has NO effect
@Service
public class MyService {
    public void outer() {
        inner(); // calls this.inner() directly — skips proxy!
    }
    @Transactional
    public void inner() { ... } // @Transactional ignored here
}
// FIX: Inject MyService into itself and call through the injected reference

// GOTCHA 2: Checked exceptions do NOT trigger rollback by default
@Transactional(rollbackFor = IOException.class) // explicit rollback for checked exceptions

// GOTCHA 3: @Transactional on private methods is silently ignored
// FIX: Make the method public
```

---

## Hibernate Caching

### First-Level Cache (L1) — always on

Scoped to one Session/transaction. Loading the same entity twice in the same transaction hits the DB only once.

```java
User u1 = userRepo.findById(1L).get(); // SQL fired
User u2 = userRepo.findById(1L).get(); // NO SQL — from L1 cache
assert u1 == u2; // same Java object
```

### Second-Level Cache (L2) — opt-in

Shared across all sessions in the application. Must be explicitly configured.

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }
```

```yaml
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheCacheRegionFactory
```

Best for: frequently read, rarely changed data (e.g., product catalog).

### Cache Summary

| Cache | Scope | Default | Use For |
|---|---|---|---|
| L1 (Session) | One transaction | Always on | Repeated access within same request |
| L2 (Shared) | Whole application | Opt-in | Frequently read, rarely changed data |
| Query Cache | Whole application | Opt-in | Repeated identical queries |

---

## Connection Pooling (HikariCP)

Creating a DB connection is expensive (TCP handshake, auth, SSL). HikariCP maintains a **pool** of pre-established connections — requests borrow one and return it instead of creating/closing each time.

```yaml
spring:
  datasource:
    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 600000       # remove idle connection after 10 min
      connection-timeout: 30000  # throw if no connection free within 30s
      max-lifetime: 1800000      # recycle connections older than 30 min
      leak-detection-threshold: 2000 # warn if connection held > 2s
```

> **Interview Tip**: Pool size rule of thumb = `(CPU cores × 2) + number of disk spindles`. Too large causes context-switching overhead; too small causes queuing and timeouts.

---

## Spring Data JPA Repositories

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);
    List<User> findByStatusOrderByCreatedAtDesc(Status status);
    boolean existsByEmail(String email);

    @Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.id = :id")
    Optional<User> findByIdWithRoles(@Param("id") Long id);

    List<UserSummary> findByStatus(Status status);  // projection

    Page<User> findByStatus(Status status, Pageable pageable);

    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.status = :status")
    void deleteByStatus(@Param("status") Status status);
}
```

### Projection Interface

```java
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
    // Only these 3 columns fetched — not all 30+ columns in User
}
```

### Pagination

```java
Page<User> page = userRepository.findByStatus(
    Status.ACTIVE,
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
);
page.getContent();       // List<User> for this page
page.getTotalPages();    // total page count
page.getTotalElements(); // total matching records
```

### Custom Repository Implementation

```java
public interface UserRepositoryCustom {
    List<User> searchUsers(String keyword, Status status);
}

// Class name MUST be "<RepositoryName>Impl"
public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<User> searchUsers(String keyword, Status status) {
        // Criteria API or custom JPQL
    }
}

public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom { }
```

---

## Common Interview Questions

### Q: What is the N+1 problem and how do you fix it?

Fetching N entities triggers N additional queries for their associations (e.g., 1 query for 100 orders + 100 queries to load each customer = 101 total). Fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize` to collapse those into 1 (or a few) queries.

---

### Q: What is the difference between `save()` and `saveAndFlush()`?

`save()` stages the entity — the SQL may be deferred to commit time. `saveAndFlush()` forces the SQL to fire immediately. Use `saveAndFlush()` when the next operation in the same transaction must see the just-saved data.

---

### Q: Explain `@Transactional` propagation REQUIRED vs REQUIRES_NEW.

`REQUIRED` joins an existing transaction — both share the same fate (either both commit or both roll back). `REQUIRES_NEW` suspends the existing transaction and starts a completely independent one — inner can commit while outer rolls back. Use `REQUIRES_NEW` for audit logs or actions that must persist regardless of the main transaction outcome.

---

### Q: What is the difference between `CascadeType.REMOVE` and `orphanRemoval = true`?

`CascadeType.REMOVE` deletes children only when the parent is deleted. `orphanRemoval = true` also deletes children when they are removed from the parent's collection (even without deleting the parent) and implies `CascadeType.REMOVE`. Use `orphanRemoval` when a child cannot exist without its parent.

---

### Q: What causes `LazyInitializationException`?

Accessing a LAZY association after the Hibernate Session has closed. Fix with `JOIN FETCH`, `@EntityGraph`, `@Transactional` around the caller, or DTO projections. Avoid `open-session-in-view` — it's an anti-pattern that keeps DB connections open for the entire HTTP request.

---

### Q: What is the difference between JPQL and native SQL?

JPQL queries Java entity classes and fields — database-independent and auto-maps to entities. Native SQL is actual database SQL — required for DB-specific features (window functions, JSON operations) and needs `nativeQuery = true`.

---

### Q: How does Hibernate dirty checking work?

When an entity enters `PERSISTENT` state, Hibernate snapshots all its field values. At flush time, it compares current values against the snapshot. Any changed field triggers an automatic `UPDATE` — no need to call `save()` again.

---

### Q: What is the difference between L1 and L2 cache?

L1 is scoped to one Session (one transaction), always active — same entity loaded twice in the same transaction returns the same Java object with no second DB hit. L2 is shared across all sessions and transactions in the entire application, is opt-in, and survives across requests.

---

## Quick Reference Cheat Sheet

```
JPA          → specification (interfaces & rules)
Hibernate    → JPA implementation + extras (HQL, caching)
Spring Data  → abstraction over JPA (JpaRepository, query derivation)

Entity States:
  Transient  → new object, no id, not in DB, not tracked
  Persistent → tracked by JPA, auto-synced on flush
  Detached   → has id, not tracked (after session close)
  Removed    → queued for DELETE on next flush

Fetch Types:
  LAZY  → load on access (default: @OneToMany, @ManyToMany)
  EAGER → always load with parent (default: @ManyToOne, @OneToOne)
  Best practice: LAZY everywhere — use JOIN FETCH when needed

N+1 Fixes:
  → JOIN FETCH in JPQL
  → @EntityGraph
  → @BatchSize

@Transactional Gotchas:
  rollback  → only on RuntimeException by default; use rollbackFor for checked
  self-call → bypasses proxy; @Transactional has no effect
  private   → @Transactional ignored on private methods

Propagation:
  REQUIRED     → join existing or create new (shared fate)
  REQUIRES_NEW → always create new, independent of caller

Cascade:
  REMOVE        → delete children when parent is deleted
  orphanRemoval → also delete children removed from collection (stronger)

Isolation Problems:
  Dirty Read           → reading uncommitted data
  Non-Repeatable Read  → same row, different values in same transaction
  Phantom Read         → same query, different rows in same transaction

Caching:
  L1 → session scope, always on
  L2 → application scope, opt-in (EhCache, Caffeine)

HikariCP → default connection pool in Spring Boot
  maximum-pool-size → cap concurrent DB connections
  Rule of thumb: (CPU cores × 2) + disk spindles
```

---

*Last Updated: 2026-06-18*
