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

---

## Entity Basics

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // AUTO_INCREMENT
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING) // store as "ACTIVE" not 0/1
    private Status status;

    @CreationTimestamp    // Hibernate-specific
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;

    @Version              // optimistic locking
    private Integer version;
}
```

### Generation Strategies

| Strategy | Description | Best For |
|---|---|---|
| `IDENTITY` | DB auto-increment (`AUTO_INCREMENT`) | MySQL, PostgreSQL |
| `SEQUENCE` | DB sequence object | PostgreSQL, Oracle |
| `TABLE` | Special table for ID generation | Portability (slow) |
| `AUTO` | JPA chooses based on DB | Development only |
| `UUID` | Random UUID | Distributed systems |

---

## Entity Lifecycle States

Every entity goes through four states managed by the **EntityManager** (Hibernate Session).

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
| **Persistent** | Yes | Yes (or pending) | Yes |
| **Detached** | Yes | Yes | No |
| **Removed** | Yes | Yes (pending DELETE) | Yes |

```java
// Transient
User user = new User();
user.setName("Alice");

// Persistent (after persist)
entityManager.persist(user);
user.setEmail("alice@example.com"); // automatically synced on flush — no explicit save needed

// Detached (after context closes)
entityManager.close();
user.setName("Updated"); // changes NOT tracked

// Merge detached entity
User managed = entityManager.merge(user); // returns new managed instance

// Removed
entityManager.remove(managed); // queued for DELETE
```

> **Interview Tip**: A persistent entity is "dirty tracked" — any field changes are automatically persisted on `flush()` or transaction commit. You do NOT need to call `save()` again.

---

## Relationships & Mappings

### @OneToOne

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id") // FK in users table
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "profile") // owned by User
    private User user;
}
```

### @OneToMany / @ManyToOne

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id") // FK in orders table
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
}
```

### @ManyToMany

```java
@Entity
public class Student {
    @Id @GeneratedValue
    private Long id;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",             // junction table name
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id @GeneratedValue
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### Owning Side vs Inverse Side

- The **owning side** has the FK column (`@JoinColumn`) — changes here are persisted
- The **inverse side** has `mappedBy` — changes here are ignored by Hibernate
- In `@ManyToMany`, the owning side has `@JoinTable`

---

## Fetch Types: LAZY vs EAGER

```java
// LAZY (default for @OneToMany, @ManyToMany)
// Association is loaded only when accessed
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders; // SQL fired only when orders is accessed

// EAGER (default for @ManyToOne, @OneToOne)
// Association is always loaded with the parent
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer; // always loaded with order
```

### Default Fetch Types

| Annotation | Default Fetch |
|---|---|
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |
| `@ManyToOne` | EAGER |
| `@OneToOne` | EAGER |

> **Best Practice**: Always use `LAZY` for collections. Use `JOIN FETCH` in JPQL when you need associations. `EAGER` on collections causes serious performance problems.

### LazyInitializationException

Occurs when accessing a LAZY association after the Hibernate Session is closed.

```java
// WRONG — session closed after repository call in @Transactional method
User user = userRepository.findById(1L).get();
// session closes here (outside @Transactional)
user.getOrders().size(); // LazyInitializationException!

// FIX 1: Use JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);

// FIX 2: Use @Transactional to keep session open
@Transactional
public void processUser(Long id) {
    User user = userRepository.findById(id).get();
    user.getOrders().size(); // session still open — works
}

// FIX 3: Use DTOs with specific projections
// FIX 4: Use @EntityGraph
@EntityGraph(attributePaths = {"orders"})
User findWithOrdersById(Long id);
```

---

## Cascade Types

Cascading propagates operations from parent to child entity.

| Cascade Type | Effect |
|---|---|
| `PERSIST` | persist() on parent persists children |
| `MERGE` | merge() on parent merges children |
| `REMOVE` | remove() on parent removes children |
| `REFRESH` | refresh() propagated to children |
| `DETACH` | detach() propagated to children |
| `ALL` | All of the above |

```java
// orphanRemoval vs CascadeType.REMOVE
@OneToMany(cascade = CascadeType.REMOVE)   // DELETE children when parent deleted
@OneToMany(orphanRemoval = true)            // DELETE children when removed from collection

// Example: removing an item from order automatically deletes it from DB
order.getItems().remove(item);  // with orphanRemoval=true → DELETE fired
```

> **Interview Tip**: `orphanRemoval = true` is stronger than `CascadeType.REMOVE`. It also fires DELETE when an item is simply removed from the parent's collection, not just when the parent is deleted.

---

## N+1 Problem & Solutions

The N+1 problem is the #1 most-asked Hibernate interview topic.

### The Problem

```java
// 1 query to fetch all orders
List<Order> orders = orderRepository.findAll(); // SELECT * FROM orders — 1 query

// N queries to fetch each order's customer (EAGER or lazy access in loop)
for (Order order : orders) {
    System.out.println(order.getCustomer().getName()); // SELECT * FROM customers WHERE id=? — N queries
}
// Total: 1 + N queries!
```

### Solution 1: JOIN FETCH in JPQL

```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer")
List<Order> findAllWithCustomer();
// Generates: SELECT o.*, c.* FROM orders o JOIN customers c ON o.customer_id = c.id
// 1 query total
```

### Solution 2: @EntityGraph

```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findAll(); // Spring Data JPA — generates JOIN FETCH automatically
```

### Solution 3: @BatchSize (Hibernate-specific)

```java
@OneToMany
@BatchSize(size = 20) // loads 20 collections in one IN() query instead of N queries
private List<OrderItem> items;
// SELECT * FROM order_items WHERE order_id IN (1, 2, 3, ... 20)
```

### Solution 4: DTO Projection

```java
@Query("SELECT new com.example.OrderSummary(o.id, c.name) FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();
// Only fetches needed fields — no entity graph overhead
```

### Detecting N+1

```yaml
# application.properties — log SQL to detect N+1
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

---

## JPQL & Criteria API

### JPQL (Java Persistence Query Language)

Like SQL but operates on **entities and fields**, not tables and columns.

```java
// Basic JPQL
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId")
List<Order> findOrdersWithItems(@Param("customerId") Long customerId);

// Named parameters
@Query("SELECT u FROM User u WHERE u.status = :status AND u.age > :minAge")
List<User> findActiveAdults(@Param("status") Status status, @Param("minAge") int minAge);

// Pagination
@Query("SELECT p FROM Product p WHERE p.price < :maxPrice")
Page<Product> findCheapProducts(@Param("maxPrice") BigDecimal price, Pageable pageable);

// Native SQL query
@Query(value = "SELECT * FROM users WHERE created_at > NOW() - INTERVAL 7 DAY", nativeQuery = true)
List<User> findRecentUsers();

// Update/Delete
@Modifying
@Transactional
@Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
int updateStatus(@Param("id") Long id, @Param("status") Status status);
```

### Query Derivation (Spring Data JPA)

```java
// Method name → auto-generated query
List<User> findByEmail(String email);
// → SELECT * FROM users WHERE email = ?

List<User> findByNameAndStatus(String name, Status status);
// → SELECT * FROM users WHERE name = ? AND status = ?

List<User> findByAgeBetween(int min, int max);
// → SELECT * FROM users WHERE age BETWEEN ? AND ?

List<User> findByNameContainingIgnoreCase(String keyword);
// → SELECT * FROM users WHERE UPPER(name) LIKE UPPER('%?%')

List<User> findByOrderByCreatedAtDesc();
// → SELECT * FROM users ORDER BY created_at DESC

Optional<User> findFirstByOrderByCreatedAtDesc();
// → SELECT * FROM users ORDER BY created_at DESC LIMIT 1

boolean existsByEmail(String email);
long countByStatus(Status status);
```

### Criteria API (Type-safe dynamic queries)

```java
public List<User> searchUsers(String name, Status status, Integer minAge) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<User> query = cb.createQuery(User.class);
    Root<User> root = query.from(User.class);

    List<Predicate> predicates = new ArrayList<>();

    if (name != null) {
        predicates.add(cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%"));
    }
    if (status != null) {
        predicates.add(cb.equal(root.get("status"), status));
    }
    if (minAge != null) {
        predicates.add(cb.greaterThanOrEqualTo(root.get("age"), minAge));
    }

    query.where(predicates.toArray(new Predicate[0]));
    query.orderBy(cb.desc(root.get("createdAt")));

    return entityManager.createQuery(query).getResultList();
}
```

---

## @Transactional — Propagation & Isolation

### @Transactional Basics

```java
@Service
public class OrderService {

    @Transactional  // start transaction before method, commit/rollback after
    public Order createOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        paymentService.charge(order);  // if this throws, order is also rolled back
        return order;
    }
}
```

### Transaction Propagation

Controls what happens when a `@Transactional` method is called from another `@Transactional` method.

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing transaction; create new if none |
| `REQUIRES_NEW` | Always create new transaction; suspend existing |
| `NESTED` | Create nested transaction (savepoint); roll back to savepoint on failure |
| `SUPPORTS` | Join if exists; run non-transactionally if none |
| `NOT_SUPPORTED` | Suspend existing transaction; run non-transactionally |
| `MANDATORY` | Must join existing transaction; throw if none |
| `NEVER` | Must not be in transaction; throw if there is one |

```java
@Transactional(propagation = Propagation.REQUIRED)   // default
public void outer() {
    innerRequired();     // joins outer transaction
    innerRequiresNew();  // starts NEW transaction, outer is suspended
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void innerRequiresNew() {
    // if this fails, outer transaction is NOT rolled back
}
```

### Transaction Isolation Levels

Controls how concurrent transactions see each other's changes.

```java
@Transactional(isolation = Isolation.READ_COMMITTED) // default in most DBs
```

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible |
| `READ_COMMITTED` | Prevented | Possible | Possible |
| `REPEATABLE_READ` | Prevented | Prevented | Possible |
| `SERIALIZABLE` | Prevented | Prevented | Prevented |

- **Dirty Read**: Reading uncommitted changes from another transaction
- **Non-Repeatable Read**: Same row returns different values in same transaction
- **Phantom Read**: Same query returns different rows in same transaction

### @Transactional Gotchas

```java
// GOTCHA 1: self-invocation bypasses proxy — @Transactional has NO effect
@Service
public class MyService {
    public void outer() {
        inner(); // calls this.inner(), not the proxy — @Transactional IGNORED
    }

    @Transactional
    public void inner() { ... }
}
// Fix: inject MyService into itself, or use ApplicationContext.getBean()

// GOTCHA 2: only RuntimeException triggers rollback by default
@Transactional
public void process() throws IOException {
    // IOException is checked — does NOT cause rollback by default!
}
// Fix:
@Transactional(rollbackFor = IOException.class)

// GOTCHA 3: @Transactional on private methods is ignored (Spring AOP)
@Transactional
private void doWork() { } // NO EFFECT — must be public
```

---

## Hibernate Caching

### First-Level Cache (Session Cache)

- Built into every Hibernate Session (EntityManager)
- Enabled by default, cannot be disabled
- Scoped to a single transaction/session
- Within one session, same entity is loaded from cache on second access

```java
User u1 = userRepo.findById(1L).get(); // SQL fired
User u2 = userRepo.findById(1L).get(); // NO SQL — served from L1 cache
assert u1 == u2; // same object reference
```

### Second-Level Cache (Shared Cache)

- Shared across sessions/transactions
- NOT enabled by default — needs explicit configuration
- Requires a cache provider: EhCache, Caffeine, Hazelcast

```java
// Entity opt-in
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }

// application.properties
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=
    org.hibernate.cache.jcache.JCacheCacheRegionFactory
```

### Query Cache

Caches query result sets (not entities). Depends on second-level cache.

```java
@Query("SELECT p FROM Product p WHERE p.category = :cat")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(@Param("cat") String category);
```

---

## Connection Pooling (HikariCP)

HikariCP is the default connection pool in Spring Boot. It maintains a pool of DB connections to avoid the overhead of creating new connections per request.

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
    hikari:
      pool-name: MyAppPool
      minimum-idle: 5          # min connections in pool
      maximum-pool-size: 20    # max connections in pool
      idle-timeout: 600000     # ms before idle connection removed (10 min)
      connection-timeout: 30000 # ms to wait for connection (30s)
      max-lifetime: 1800000    # ms max connection lifetime (30 min)
      leak-detection-threshold: 2000 # log warning if connection held > 2s
```

> **Interview Tip**: Pool size rule of thumb = `(CPU cores × 2) + effective spindle count`. A pool that is too large causes context-switching overhead; too small causes request queuing.

---

## Spring Data JPA Repositories

```java
// JpaRepository provides: CRUD, findAll(Pageable), flush, saveAndFlush, deleteInBatch
public interface UserRepository extends JpaRepository<User, Long> {

    // Query derivation
    Optional<User> findByEmail(String email);
    List<User> findByStatusOrderByCreatedAtDesc(Status status);
    boolean existsByEmail(String email);
    long countByStatus(Status status);

    // Custom JPQL
    @Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.id = :id")
    Optional<User> findByIdWithRoles(@Param("id") Long id);

    // Projection — only specific fields
    List<UserSummary> findByStatus(Status status); // UserSummary is an interface

    // Pagination
    Page<User> findByStatus(Status status, Pageable pageable);

    // Modifying
    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.status = :status")
    void deleteByStatus(@Param("status") Status status);
}

// Projection interface
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

// Usage with pagination
Page<User> page = userRepository.findByStatus(
    Status.ACTIVE,
    PageRequest.of(0, 20, Sort.by("createdAt").descending())
);
page.getContent();    // list of users on this page
page.getTotalPages(); // total number of pages
page.getTotalElements(); // total count of matching records
```

### Custom Repository Implementation

```java
public interface UserRepositoryCustom {
    List<User> searchUsers(String keyword, Status status);
}

public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<User> searchUsers(String keyword, Status status) {
        // Criteria API or JPQL implementation
    }
}

public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom { }
```

---

## Common Interview Questions

### Q: What is the N+1 problem and how do you fix it?

The N+1 problem occurs when fetching a list of N entities triggers N additional queries to load their associations — 1 query for the list + N queries for each associated entity.

**Fix**: Use `JOIN FETCH` in JPQL, `@EntityGraph`, or `@BatchSize`.

---

### Q: What is the difference between `save()` and `saveAndFlush()`?

- `save()`: Persists entity but may hold it in the persistence context until the transaction commits (or flush is triggered). Write to DB is deferred.
- `saveAndFlush()`: Immediately flushes (writes SQL to DB) within the current transaction. Use when you need the DB state visible to the next query in the same transaction.

---

### Q: Explain `@Transactional` propagation REQUIRED vs REQUIRES_NEW.

- `REQUIRED`: If a transaction exists, the method joins it. If not, a new one is created. Both methods share the same transaction — if either fails, both roll back.
- `REQUIRES_NEW`: Always creates a new transaction. The outer transaction is suspended. If the inner fails, only the inner rolls back — the outer continues independently.

---

### Q: What is the difference between `CascadeType.REMOVE` and `orphanRemoval = true`?

- `CascadeType.REMOVE`: When the parent is deleted, cascades the DELETE to children.
- `orphanRemoval = true`: Deletes children that are removed from the parent's collection, even without deleting the parent. Implies `CascadeType.REMOVE`.

---

### Q: What causes `LazyInitializationException`?

Accessing a `LAZY` association after the Hibernate session (EntityManager) has been closed. 

**Fixes**: `JOIN FETCH` in query, `@EntityGraph`, `@Transactional` around the caller, DTO projections, or (anti-pattern) `open-session-in-view`.

---

### Q: What is the difference between JPQL and native SQL?

- **JPQL**: Object-oriented query language. Queries against entity classes and fields. Database-independent. Auto-maps to entities.
- **Native SQL**: Actual SQL for the specific database. Required for DB-specific features (window functions, stored procedures, JSON operations). Use `nativeQuery = true`.

---

### Q: How does Hibernate dirty checking work?

When you load an entity into a `PERSISTENT` state, Hibernate takes a snapshot of its fields. At `flush()` time (before commit), Hibernate compares the current state to the snapshot. If any field changed, it automatically generates an `UPDATE` SQL. You don't need to call `save()` — it's automatic.

---

## Quick Reference Cheat Sheet

```
JPA     → specification (interfaces)
Hibernate → JPA implementation + extra features
Spring Data JPA → abstraction over JPA (JpaRepository)

Entity States:
  Transient  → new, no id, not tracked
  Persistent → id, tracked, auto-synced
  Detached   → id, not tracked (after close/clear)
  Removed    → queued for DELETE

Fetch Types:
  LAZY  → load on access (default for @OneToMany, @ManyToMany)
  EAGER → always load (default for @ManyToOne, @OneToOne)
  Best practice: use LAZY for all collections

N+1 Fix:
  → JOIN FETCH in JPQL
  → @EntityGraph
  → @BatchSize

@Transactional:
  rollback  → RuntimeException (default), add rollbackFor for checked exceptions
  self-call → bypasses proxy, @Transactional has no effect
  private   → @Transactional ignored on private methods

Propagation:
  REQUIRED     → join or create
  REQUIRES_NEW → always new, suspend outer

Cascade:
  REMOVE       → delete children when parent deleted
  orphanRemoval→ delete children removed from collection

Caching:
  L1 → session scope, always on (same session = same object)
  L2 → shared across sessions, opt-in (EhCache, Caffeine)

HikariCP → default connection pool in Spring Boot
  maximum-pool-size → cap concurrent DB connections (default 10)
```

---

*Last Updated: 2026-06-04*
