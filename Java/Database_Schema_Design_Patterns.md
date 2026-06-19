# Database Schema Design Patterns — Full Stack Java Developer Interview Preparation

## Overview

Covers the core database design concepts a junior full-stack Java developer needs for interviews: ERD fundamentals, normalization, primary key strategies, relationship patterns, soft delete, auditing, pagination, common domain schemas, JPA inheritance, and indexing.

---

## Table of Contents

1. [Entity-Relationship Design Fundamentals](#entity-relationship-design-fundamentals)
2. [Normalization](#normalization)
3. [Primary Key Strategies](#primary-key-strategies)
4. [Relationship Patterns](#relationship-patterns)
5. [Soft Delete Pattern](#soft-delete-pattern)
6. [Audit Tables Pattern](#audit-tables-pattern)
7. [Pagination Patterns](#pagination-patterns)
8. [Common Domain Schemas](#common-domain-schemas)
9. [Polymorphic Associations and JPA Inheritance](#polymorphic-associations-and-jpa-inheritance)
10. [Indexing Strategies](#indexing-strategies)
11. [Interview Questions and Answers](#interview-questions-and-answers)
12. [Quick Reference Summary](#quick-reference-summary)

---

## Entity-Relationship Design Fundamentals

### What is an Entity-Relationship Diagram (ERD)?

An ERD is a visual blueprint of a database. Before writing SQL, you model:
- **Entities**: things to track (User, Order, Product)
- **Attributes**: properties of those things (User has email, name, created_at)
- **Relationships**: how entities connect (User places Order)

### Entities and Attributes

| Type | Description | Example |
|------|-------------|---------|
| Simple | Atomic, not divisible | `email`, `price` |
| Composite | Made of sub-parts | `full_name` → `first_name` + `last_name` |
| Derived | Computed from other data | `age` from `date_of_birth` |
| Multi-valued | Can hold multiple values | `phone_numbers` |

**Design rule**: Multi-valued attributes signal a missing table. Create `user_phone_numbers` instead of `phone1`, `phone2`, `phone3` columns.

```sql
-- Right: separate table for multi-valued attribute
CREATE TABLE user_phone_numbers (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    phone VARCHAR(20) NOT NULL,
    type VARCHAR(20) DEFAULT 'mobile'
);
```

Store `date_of_birth`, compute `age` at query time. Only store derived values if the computation is expensive and source data rarely changes.

### Cardinality

- **One-to-One (1:1)**: A user has one profile.
- **One-to-Many (1:N)**: A department has many employees.
- **Many-to-Many (M:N)**: Students enroll in many courses; always resolved with a junction table.

### Participation Constraints

- **Total (mandatory)**: Every entity instance MUST participate → implement as `NOT NULL`.
- **Partial (optional)**: An instance MAY or MAY NOT participate → implement as nullable FK.

### Crow's Foot Notation

```
One and only one:  ──||
Zero or one:       ──|O
One or many:       ──|<
Zero or many:      ──O<
```

`User ──||—O< Orders`: each order belongs to exactly one user; a user may have zero or many orders.

### Identifying vs Non-Identifying Relationships

**Identifying**: Child cannot exist without parent; parent's PK is part of child's PK.

```sql
CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id),
    line_number INT NOT NULL,
    product_id BIGINT NOT NULL REFERENCES products(id),
    PRIMARY KEY (order_id, line_number)
);
```

**Non-identifying**: Child has its own identity; FK is just a regular column.

### Weak Entities

A weak entity has no PK of its own — it depends on a parent. Uses the parent's PK plus a partial key (discriminator) as a composite PK.

```sql
CREATE TABLE rooms (
    building_id BIGINT NOT NULL REFERENCES buildings(id) ON DELETE CASCADE,
    room_number VARCHAR(10) NOT NULL,
    capacity INT,
    PRIMARY KEY (building_id, room_number)
);
```

---

## Normalization

Structuring a schema to eliminate redundancy and improve integrity. Each form builds on the previous.

### First Normal Form (1NF)

- Every column must be atomic (no comma-separated lists).
- No repeating groups (no `skill1`, `skill2`, `skill3` columns).
- Every row uniquely identifiable (has a PK).

```sql
-- Fix multi-valued: separate table
CREATE TABLE employee_skills (
    emp_id BIGINT REFERENCES employees(id),
    skill VARCHAR(100),
    PRIMARY KEY (emp_id, skill)
);
```

### Second Normal Form (2NF)

Applies to tables with **composite PKs** only. Every non-key column must depend on the **whole** PK, not just part of it.

```sql
-- Violation: student_name depends only on student_id (partial)
-- Fix: separate tables
CREATE TABLE students (id BIGSERIAL PRIMARY KEY, name VARCHAR(100) NOT NULL);
CREATE TABLE courses (id BIGSERIAL PRIMARY KEY, title VARCHAR(200) NOT NULL);
CREATE TABLE enrollments (
    student_id BIGINT REFERENCES students(id),
    course_id BIGINT REFERENCES courses(id),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

### Third Normal Form (3NF)

No non-key column should depend on another non-key column (no transitive dependencies).

```sql
-- Violation: dept_name depends on dept_id (non-key), not emp_id
-- Fix:
CREATE TABLE departments (id BIGSERIAL PRIMARY KEY, name VARCHAR(100) NOT NULL);
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    dept_id BIGINT REFERENCES departments(id)
);
```

**Memory aid**: "Every non-key attribute must depend on the key, the whole key, and nothing but the key."

### Boyce-Codd Normal Form (BCNF)

Stricter than 3NF. Every determinant must be a candidate key. Violations occur with multiple overlapping candidate keys. Decompose so each table has one candidate key driving its non-key attributes.

### Fourth Normal Form (4NF)

No two independent multi-valued facts about an entity in the same table.

```sql
-- Fix: separate skills and languages into their own tables
CREATE TABLE employee_skills (emp_id BIGINT, skill VARCHAR(100));
CREATE TABLE employee_languages (emp_id BIGINT, language VARCHAR(100));
```

### When to Denormalize

| Strategy | Example | Use Case |
|----------|---------|----------|
| Stored derived value | `total_price` on order | Avoid summing order_items on every read |
| Redundant column | `user_email` on orders | Avoid join for reports |
| Materialized view | Denormalized analytics table | OLAP / dashboards |

Measure with `EXPLAIN ANALYZE` first. Document every denormalization and keep copies in sync via triggers or application logic.

---

## Primary Key Strategies

### Surrogate vs Natural Keys

**Natural key**: a real-world attribute that's uniquely stable (ISO country codes, currency codes).

**Surrogate key**: system-generated, no business meaning.

**Best of both worlds**:
```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,        -- surrogate: used as FK everywhere
    sku VARCHAR(50) UNIQUE NOT NULL, -- natural: enforced, but not the PK
    name VARCHAR(255) NOT NULL
);
```

Always add a surrogate PK. Enforce natural keys as UNIQUE constraints.

### UUID as Primary Key

Use UUID when: IDs are generated across distributed systems, IDs must be unguessable, or merging data from multiple databases.

```sql
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)  // JPA 3.1+
    private UUID id;
}
```

### UUID Variants

| ID Type | Ordered? | Size | Notes |
|---------|----------|------|-------|
| BIGSERIAL | Yes | 8 bytes | Best performance, guessable |
| UUID v4 | No | 16 bytes | Random — causes index fragmentation |
| UUID v7 | Yes | 16 bytes | Time-ordered, fixes fragmentation |
| ULID | Yes | 16 bytes | URL-safe, 48-bit time + 80-bit random |
| Snowflake | Yes | 8 bytes | Time-ordered, distributed |

**Rule**: Use BIGSERIAL for single-node systems. Use UUID v7 or ULID when you need distributed, unguessable, sortable IDs.

---

## Relationship Patterns

### One-to-One

Use a shared PK (strongest coupling) or a UNIQUE FK (more common).

```sql
-- UNIQUE FK approach
CREATE TABLE user_profiles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500)
);
```

```java
@Entity
public class User {
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;
}
```

**Important**: `@OneToOne` is EAGER by default — always override to LAZY.

### One-to-Many

The FK lives on the child (many) side. Always index foreign keys.

```sql
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    department_id BIGINT NOT NULL REFERENCES departments(id),
    manager_id BIGINT REFERENCES employees(id)
);
CREATE INDEX idx_employees_department_id ON employees(department_id);
CREATE INDEX idx_employees_manager_id ON employees(manager_id);
```

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();
}

@Entity
public class Employee {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id", nullable = false)
    private Department department;
}
```

**Tip**: Always initialize collections (`= new ArrayList<>()`) to avoid NullPointerException on empty collections.

### Many-to-Many with Junction Table

```sql
CREATE TABLE enrollments (
    student_id BIGINT NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id BIGINT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    grade CHAR(2),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    PRIMARY KEY (student_id, course_id)
);
CREATE INDEX idx_enrollments_course_id ON enrollments(course_id);
```

When the junction table has its own attributes, model it as a full `@Entity` with `@EmbeddedId`. Avoid plain `@ManyToMany` when the junction has extra columns.

```java
@Entity
@Table(name = "enrollments")
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    @JoinColumn(name = "student_id")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    @JoinColumn(name = "course_id")
    private Course course;

    private LocalDateTime enrolledAt;
    private String grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals() and hashCode() required
}
```

---

## Soft Delete Pattern

### Why Soft Delete?

Hard delete is permanent. Soft delete marks records as deleted without removing them, using a `deleted_at TIMESTAMPTZ` column.

**Benefits**: data recovery, audit trail, preserves referential integrity for historical records.

**Trade-offs**: all queries must filter `WHERE deleted_at IS NULL`; unique constraints need special handling; table grows indefinitely.

### SQL Implementation

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = 42;

-- Query active users
SELECT * FROM users WHERE deleted_at IS NULL;

-- Partial unique index: uniqueness only among active users
CREATE UNIQUE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;
```

**Why partial unique index?** `UNIQUE(email, deleted_at)` does NOT work because PostgreSQL allows multiple `(email, NULL)` rows — NULL != NULL in unique constraints. The partial index covers only active rows.

### JPA/Hibernate Implementation

```java
@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
@SQLRestriction("deleted_at IS NULL")  // Hibernate 6.3+
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;
    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;
}
```

`@SQLDelete` intercepts `repository.delete(user)` and runs the UPDATE. `@SQLRestriction` automatically adds `WHERE deleted_at IS NULL` to all JPQL queries. Bypass with a native query.

### GDPR and Soft Delete

GDPR "right to erasure" requires truly removing PII. Solutions:
1. **Anonymize on soft delete**: replace PII with nulls/hashed values.
2. **Scheduled hard delete**: soft delete now, hard delete after retention period.

```sql
UPDATE users
SET email = 'deleted_' || id || '@anonymized.invalid',
    name = 'Deleted User',
    phone = NULL,
    deleted_at = NOW()
WHERE id = 42;
```

---

## Audit Tables Pattern

### Created/Updated Timestamps

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by BIGINT REFERENCES users(id),
    updated_by BIGINT REFERENCES users(id)
);

CREATE OR REPLACE FUNCTION fn_update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
```

The same trigger function can be reused across multiple tables.

### Spring Data JPA Auditing

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {
    @Bean
    public AuditorAware<Long> auditorProvider() {
        return () -> Optional.ofNullable(
            ((AppUserDetails) SecurityContextHolder.getContext()
                .getAuthentication().getPrincipal()).getId());
    }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private Long createdBy;

    @LastModifiedBy
    private Long updatedBy;
}

@Entity
public class Order extends Auditable {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal total;
}
```

### Full Audit Trail and Temporal Tables

**Full audit trail**: A separate `*_audit` table stores `operation`, `changed_at`, `changed_by`, `old_data JSONB`, `new_data JSONB`, populated by an `AFTER INSERT OR UPDATE OR DELETE` trigger. Needed for compliance-heavy domains.

**Temporal tables**: Track data across valid time (when true in the real world) and transaction time (when recorded). Common pattern: `valid_from`/`valid_to` columns, current record uses `'9999-12-31'` as `valid_to`.

```sql
SELECT price FROM product_prices
WHERE product_id = 1
  AND valid_from <= '2023-01-01'
  AND valid_to > '2023-01-01';
```

---

## Pagination Patterns

### Offset Pagination

```sql
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 40;
```

**Problems**: O(offset) cost degrades at scale; page drift when new rows are inserted between requests.

```java
Pageable pageable = PageRequest.of(pageNumber, pageSize, Sort.by(Direction.DESC, "createdAt"));
Page<Order> page = orderRepository.findByUserId(userId, pageable);
```

### Keyset Pagination (Cursor-Based)

Uses the last item's sort key as a starting point instead of a row offset.

```sql
-- Next page after (created_at='2024-03-15 09:00:00', id=987)
SELECT id, title, created_at FROM articles
WHERE (created_at, id) < ('2024-03-15 09:00:00', 987)
ORDER BY created_at DESC, id DESC
LIMIT 20;

CREATE INDEX idx_articles_created_id ON articles(created_at DESC, id DESC);
```

```java
@Query("SELECT a FROM Article a WHERE a.createdAt < :lastCreatedAt " +
       "OR (a.createdAt = :lastCreatedAt AND a.id < :lastId) " +
       "ORDER BY a.createdAt DESC, a.id DESC")
List<Article> findNextPage(@Param("lastCreatedAt") LocalDateTime lastCreatedAt,
                            @Param("lastId") Long lastId, Pageable pageable);
```

| Aspect | Offset | Keyset |
|--------|--------|--------|
| Performance | O(offset) — degrades | O(log n) — consistent |
| Page drift | Yes | No |
| Random access | Yes | No |
| Total count | Easy (expensive) | Hard |
| API design | `?page=2&size=20` | `?cursor=abc&size=20` |

**Use keyset** for feeds, timelines, large tables. **Use offset** for admin panels with random page access, small tables.

---

## Common Domain Schemas

### E-Commerce Schema

Key tables: `categories`, `products`, `product_variants`, `users`, `addresses`, `orders`, `order_items`.

```sql
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id),
    product_name VARCHAR(255) NOT NULL,  -- snapshot at purchase time
    variant_sku VARCHAR(100) NOT NULL,   -- snapshot
    unit_price DECIMAL(10,2) NOT NULL,  -- snapshot: NEVER join live price for history
    quantity INT NOT NULL CHECK (quantity > 0),
    subtotal DECIMAL(10,2) GENERATED ALWAYS AS (unit_price * quantity) STORED
);
```

**Critical**: Snapshot `unit_price` in `order_items`. Never JOIN to `product_variants.price` for historical orders — price changes must not affect past records.

### RBAC Schema (Role-Based Access Control)

```sql
CREATE TABLE users (id BIGSERIAL PRIMARY KEY, email VARCHAR(255) UNIQUE NOT NULL, password_hash VARCHAR(255) NOT NULL);
CREATE TABLE roles (id BIGSERIAL PRIMARY KEY, name VARCHAR(50) UNIQUE NOT NULL);
CREATE TABLE permissions (id BIGSERIAL PRIMARY KEY, resource VARCHAR(100) NOT NULL, action VARCHAR(50) NOT NULL, UNIQUE (resource, action));

CREATE TABLE user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Check permission in one query
SELECT COUNT(*) > 0 AS has_permission
FROM users u
JOIN user_roles ur ON u.id = ur.user_id
JOIN role_permissions rp ON ur.role_id = rp.role_id
JOIN permissions p ON rp.permission_id = p.id
WHERE u.id = :userId AND p.resource = :resource AND p.action = :action;
```

```java
@PreAuthorize("@permissionService.hasPermission(authentication.principal.id, 'orders', 'write')")
public Order createOrder(CreateOrderRequest request) { ... }
```

---

## Polymorphic Associations and JPA Inheritance

Handles entities with subtypes: `Payment` → `CreditCardPayment`, `PayPalPayment`, `BankTransfer`.

### Single Table Inheritance (STI)

All subtypes in one table with a discriminator column.

```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(30) NOT NULL,
    order_id BIGINT REFERENCES orders(id),
    amount DECIMAL(10,2) NOT NULL,
    -- credit card fields (NULL for other types)
    card_last_four CHAR(4),
    card_token VARCHAR(255),
    -- paypal fields
    paypal_transaction_id VARCHAR(100),
    paypal_payer_email VARCHAR(255)
);
```

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal amount;
    @Enumerated(EnumType.STRING)
    private PaymentStatus status;
}

@Entity @DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    private String cardLastFour;
    private String cardToken;
}
```

**Pros**: No JOINs, fast polymorphic queries.
**Cons**: Many NULL columns; cannot enforce NOT NULL on subtype-specific columns.

### Joined Table Inheritance

Base table for shared columns, separate table per subtype.

```sql
CREATE TABLE payments (id BIGSERIAL PRIMARY KEY, type VARCHAR(30) NOT NULL, amount DECIMAL(10,2) NOT NULL);
CREATE TABLE credit_card_payments (
    id BIGINT PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
    card_last_four CHAR(4) NOT NULL,
    card_token VARCHAR(255)
);
```

```java
@Entity @Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment { ... }

@Entity @Table(name = "credit_card_payments") @DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    @Column(nullable = false)
    private String cardLastFour;
}
```

**Pros**: Fully normalized, subtype NOT NULL constraints work.
**Cons**: Every query JOINs base + subtype tables.

### Comparison Table

| Strategy | Join Needed? | Null Columns | Polymorphic Query | Subtype Constraints |
|----------|-------------|--------------|-------------------|---------------------|
| SINGLE_TABLE | No | Many | Fast | Not possible |
| JOINED | Yes | None | Moderate | Yes |
| TABLE_PER_CLASS | No | None | Expensive (UNION ALL) | Yes |

**Rule of thumb**: Use SINGLE_TABLE for simple hierarchies. Use JOINED when subtype integrity constraints matter. Avoid TABLE_PER_CLASS.

---

## Indexing Strategies

```sql
-- B-tree (default): =, <, >, BETWEEN, LIKE 'abc%'
CREATE INDEX idx_orders_status ON orders(status);

-- Composite (leftmost prefix rule)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Supports: WHERE user_id=? AND status=?  and  WHERE user_id=?
-- Does NOT support: WHERE status=? alone

-- Partial: index only a subset of rows
CREATE INDEX idx_orders_pending ON orders(created_at) WHERE status = 'PENDING';

-- Covering: includes extra columns to avoid heap access
CREATE INDEX idx_orders_covering ON orders(user_id, status) INCLUDE (total, created_at);

-- Expression index
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
-- Supports: WHERE LOWER(email) = 'alice@example.com'

-- GIN: for JSONB, arrays, full-text search
CREATE INDEX idx_product_attributes ON products USING GIN (attributes);
```

### When NOT to Add an Index

- Small tables (full scan is faster)
- Very low cardinality columns (`is_active BOOLEAN`)
- Write-heavy tables (every insert/update/delete maintains all indexes)
- Columns rarely used in WHERE, JOIN, or ORDER BY

---

## Interview Questions and Answers

**Q1: What is normalization and why do we do it?**

Normalization organizes a schema to eliminate redundancy and prevent anomalies: update anomalies (one fact in multiple rows), insertion anomalies (can't insert without unrelated data), and deletion anomalies (deleting a record loses other data). The trade-off: fully normalized schemas need more JOINs. Most production schemas are at 3NF with selective, documented denormalization for performance.

---

**Q2: What is the difference between 2NF and 3NF?**

2NF eliminates **partial dependencies** — non-key columns must depend on the entire composite PK. 3NF eliminates **transitive dependencies** — non-key columns must not depend on other non-key columns. Example: `(emp_id, dept_id, dept_name)` — `dept_name` depends on `dept_id` (a non-key), not `emp_id`. That's a transitive dependency, violating 3NF.

---

**Q3: When would you denormalize a database schema?**

When you have profiled queries with `EXPLAIN ANALYZE` and confirmed JOIN cost is the bottleneck, the data is read-heavy and write-infrequent, and you have a plan to keep denormalized copies in sync. Never denormalize preemptively. Common examples: storing `order_total` to avoid summing `order_items`, storing `user_email` on events to avoid a join.

---

**Q4: What is the difference between a natural key and a surrogate key?**

A natural key is a real-world attribute that uniquely identifies a record (email, ISBN). A surrogate key is system-generated with no business meaning (auto-increment, UUID). Natural keys can change (emails change) or be reassigned (SSNs). Best practice: surrogate PK for all FKs, UNIQUE constraint on the natural key.

---

**Q5: Why is UUID v4 bad for clustered index inserts? What are the alternatives?**

UUID v4 is random, so inserts scatter across the B-tree, causing page splits, write amplification, and fragmentation. Alternatives: **UUID v7** (time-ordered, fixes fragmentation), **ULID** (sortable, URL-safe), **Snowflake ID** (64-bit time-ordered integer). For single-node systems, BIGSERIAL is still the best choice.

---

**Q6: What is the soft delete pattern? What are its challenges?**

Soft delete marks records deleted with `deleted_at TIMESTAMPTZ` rather than physically removing them. Benefits: data recovery, audit trail, preserving referential integrity. Challenges: all queries need `WHERE deleted_at IS NULL` (easy to forget); standard UNIQUE constraints break (use partial unique index); table grows indefinitely; GDPR requires actual PII removal (pair with anonymization).

---

**Q7: How do you implement soft delete with JPA/Hibernate?**

Use `@SQLDelete` to override the DELETE SQL and `@SQLRestriction` to automatically filter queries. `@SQLDelete` intercepts `repository.delete()` and executes an UPDATE instead. `@SQLRestriction` appends `AND deleted_at IS NULL` to all JPQL queries. Use a native query to bypass the restriction for admin views.

---

**Q8: What are the three multi-tenancy patterns and when do you use each?**

1. **Database-per-tenant**: Complete isolation, highest cost. Use for regulated industries with few large customers.
2. **Schema-per-tenant**: Shared DB, separate schema per tenant. Good isolation at medium cost. Use for hundreds of tenants.
3. **Shared schema**: `tenant_id` column on every table. Lowest cost, scales to thousands of tenants. Use PostgreSQL RLS as a safety net against cross-tenant data leaks.

---

**Q9: What is PostgreSQL Row Level Security (RLS)?**

RLS automatically filters rows based on the session context at the database level. Even if application code forgets `WHERE tenant_id = ?`, RLS enforces it. The app sets `SET LOCAL app.current_tenant_id = '42'` per request; all queries on enabled tables are silently filtered. It's a defense-in-depth safety net — the app still adds the WHERE clause for clarity and index use.

---

**Q10: What is keyset pagination and why is it better than offset at scale?**

Offset pagination reads and discards all skipped rows — O(offset) cost, plus page drift when rows are inserted between requests. Keyset pagination uses the last item's sort key values as a cursor, so the DB does an O(log n) B-tree seek directly to the cursor position. Downside: no random page access; needs a stable unique sort key (usually `(timestamp, id)`).

---

**Q11: What are the different ways to store hierarchical data in SQL?**

**Adjacency list**: parent_id on each node. Simple inserts/moves; subtree needs recursive CTE. **Nested set**: `lft`/`rgt` values; subtrees are range queries (fast reads, but O(n) updates). **Closure table**: separate table of all ancestor-descendant pairs; fast in all directions, more storage. Use adjacency list for most cases; closure table for complex read-heavy traversal.

---

**Q12: What is the difference between SINGLE_TABLE, JOINED, and TABLE_PER_CLASS JPA inheritance?**

**SINGLE_TABLE**: one table, discriminator column. Fast queries, no JOINs, but many NULL columns and no subtype NOT NULL constraints. **JOINED**: base table + subtype tables JOINed together. Normalized, subtype constraints work, slower polymorphic queries. **TABLE_PER_CLASS**: one complete table per subtype, polymorphic queries need UNION ALL — rarely recommended. Use SINGLE_TABLE for simple cases, JOINED when constraints on subtype columns matter.

---

**Q13: How do you design a schema for a many-to-many relationship with additional attributes?**

A pure M:N uses a junction table. When the junction has its own attributes (grade, enrolled_at), model it as a full entity with `@EmbeddedId`. Don't use `@ManyToMany` + `@JoinTable` when the junction table has extra columns — JPA cannot map those attributes.

---

**Q14: What is a partial unique index and why is it important for soft delete?**

A partial index only covers rows matching a WHERE clause. For soft delete, `UNIQUE(email)` enforces uniqueness across all rows including deleted ones, blocking a re-registration with the same email. `CREATE UNIQUE INDEX ... ON users(email) WHERE deleted_at IS NULL` enforces uniqueness only among active users, allowing multiple deleted records with the same email.

---

**Q15: How do you handle schema migrations in CI/CD with Flyway?**

Commit the migration file in the same PR as the code change. CI spins up a test DB, runs `flyway migrate`, then runs tests. Never modify an applied migration — Flyway validates checksums. For zero-downtime: migrations must be backward-compatible with the current running code (additive first — add the column, deploy code, then drop old columns in a later migration).

---

**Q16: What is the N+1 query problem and how does it relate to schema design?**

Loading N entities triggers N additional queries to load each association. The schema is typically fine (FK is there); the problem is JPA fetching lazily in a loop. Fix with `JOIN FETCH` in JPQL, `@EntityGraph` to eagerly fetch specific associations per query, or `@BatchSize` to batch-fetch associations.

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") String status);
```

---

**Q17: What is the difference between ON DELETE CASCADE, SET NULL, and RESTRICT?**

`CASCADE`: delete child rows automatically when the parent is deleted. Use when children have no meaning without the parent (order_items). `SET NULL`: set the FK to NULL instead of deleting. Use when the child can exist independently (employee.manager_id). `RESTRICT` (default): prevents deleting a parent if children reference it — forces explicit cleanup.

---

**Q18: What is optimistic locking and how do you implement it in JPA?**

Optimistic locking checks that a row hasn't changed before updating, without locking it on read. Add a `version` column; the UPDATE includes `WHERE id=? AND version=?`. If 0 rows updated, a concurrent transaction already changed it. In JPA, annotate with `@Version` — Hibernate manages version checks automatically and throws `OptimisticLockException` on conflict.

```java
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Version
    private Integer version;
}
```

Handle with retry or HTTP 409. Use pessimistic locking (`@Lock(LockModeType.PESSIMISTIC_WRITE)`) for critical financial operations where conflicts are expected.

---

**Q19: What is a database view vs a materialized view?**

A regular view is a stored query — no data stored, executes the underlying SQL on every access. A materialized view caches the query result and must be refreshed manually or on a schedule. Use regular views to encapsulate complex joins or provide a security-filtered subset. Use materialized views for expensive aggregations powering dashboards.

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
-- CONCURRENTLY allows reads during refresh (requires unique index on the view)
```

---

**Q20: When should you use JSONB columns in PostgreSQL?**

Use JSONB for genuinely schemaless data: event payloads, product attributes that vary by category, third-party API responses. Supports GIN indexing for containment queries (`@>`). Avoid JSONB for columns you JOIN on, filter frequently, or need FK/NOT NULL constraints on — use typed columns for those.

---

## Quick Reference Summary

### Normalization Levels

| Form | Violation | Fix |
|------|-----------|-----|
| 1NF | Non-atomic values, repeating groups | Separate table for multi-valued attributes |
| 2NF | Partial dependency on composite PK | Move partially-dependent columns to own table |
| 3NF | Transitive dependency (non-key → non-key) | Extract into separate table |
| BCNF | Determinant is not a candidate key | Decompose the table |

### PK Strategy Decision Tree

```
Single-node, sequential inserts?         → BIGSERIAL
Distributed, unguessable, sortable?      → UUID v7 or ULID
High write throughput, sortable?         → Snowflake ID
Legacy/simple random UUID needed?        → UUID v4 (avoid for clustered PK)
```

### Hierarchy Model Selection

| Need | Use |
|------|-----|
| Simple, write-heavy, moderate depth | Adjacency list + recursive CTE |
| Static tree, read-heavy | Nested set |
| Complex traversal, both directions | Closure table |

### Pagination Selection

| Use Case | Choose |
|----------|--------|
| Small table, admin panel, random page access | Offset |
| Large table, feed/timeline, real-time data | Keyset (cursor) |
| Infinite scroll | Keyset |
| "Show page 5 of 50" UI | Offset |

---

*Last Updated: 2026-06-18*
