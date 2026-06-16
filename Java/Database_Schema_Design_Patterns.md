# Database Schema Design Patterns — Full Stack Java Developer Interview Preparation

## Table of Contents
1. [Entity-Relationship Design Fundamentals](#entity-relationship-design-fundamentals)
2. [Normalization](#normalization)
3. [Primary Key Strategies](#primary-key-strategies)
4. [Relationship Patterns](#relationship-patterns)
5. [Soft Delete Pattern](#soft-delete-pattern)
6. [Audit Tables Pattern](#audit-tables-pattern)
7. [Multi-Tenancy Patterns](#multi-tenancy-patterns)
8. [Hierarchical Data](#hierarchical-data)
9. [Pagination Patterns](#pagination-patterns)
10. [Common Domain Schemas](#common-domain-schemas)
11. [Polymorphic Associations and JPA Inheritance](#polymorphic-associations-and-jpa-inheritance)
12. [Schema Migrations with Flyway and Liquibase](#schema-migrations-with-flyway-and-liquibase)
13. [Indexing Strategies](#indexing-strategies)
14. [Interview Questions and Answers](#interview-questions-and-answers)

---

## Entity-Relationship Design Fundamentals

### What is an Entity-Relationship Diagram (ERD)?

An ERD is a visual blueprint of a database. Before writing a single line of SQL, you model:
- **Entities**: things in the real world you need to track (User, Order, Product)
- **Attributes**: properties of those things (User has email, name, created_at)
- **Relationships**: how entities are connected (User places Order)

This upfront design prevents costly schema changes later.

---

### Entities and Attributes

**Entity**: a distinct object or concept tracked in the database. Represented as a rectangle in ERD notation.

**Attribute types:**

| Type | Description | Example |
|------|-------------|---------|
| Simple | Atomic, not divisible | `email`, `price` |
| Composite | Made of sub-parts | `full_name` → `first_name` + `last_name` |
| Derived | Computed from other data | `age` derived from `date_of_birth` |
| Multi-valued | Can hold multiple values | `phone_numbers` (a person can have many) |

**Design rule**: Multi-valued attributes signal a missing table. If a user has multiple phone numbers, create a `user_phone_numbers` table rather than storing `phone1`, `phone2`, `phone3` columns.

```sql
-- Wrong: multi-valued in columns
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phone1 VARCHAR(20),
    phone2 VARCHAR(20),  -- bad: what if they have 3?
    phone3 VARCHAR(20)
);

-- Right: separate table
CREATE TABLE user_phone_numbers (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    phone VARCHAR(20) NOT NULL,
    type VARCHAR(20) DEFAULT 'mobile'  -- mobile, home, work
);
```

**Derived attributes** should generally NOT be stored. Store `date_of_birth`, compute `age` in a query or application layer. Exception: if the computation is expensive and the source data rarely changes, store the derived value and update it via trigger.

---

### Cardinality

Cardinality defines how many instances of entity A can be associated with entity B.

**One-to-One (1:1)**: A user has exactly one profile; a profile belongs to exactly one user.

**One-to-Many (1:N)**: A department has many employees; each employee belongs to one department.

**Many-to-Many (M:N)**: A student can enroll in many courses; a course can have many students. Always resolved with a junction (associative) table.

---

### Participation Constraints

**Total participation (mandatory)**: Every instance of entity A MUST participate in the relationship. Drawn as a double line in ER diagrams. Implemented as `NOT NULL` in SQL.

**Partial participation (optional)**: An instance of entity A MAY or MAY NOT participate. Drawn as a single line. Implemented as a nullable FK in SQL.

```sql
-- Department is mandatory for employee (total participation)
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    department_id BIGINT NOT NULL REFERENCES departments(id)  -- NOT NULL = mandatory
);

-- Manager is optional for employee (partial participation)
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    manager_id BIGINT REFERENCES employees(id)  -- nullable = optional
);
```

---

### Crow's Foot Notation

The most widely used ERD notation in industry tools (Lucidchart, draw.io, dbdiagram.io).

```
One:         ──|
Many:        ──<
One and only one:   ──||
Zero or one:        ──|O
One or many:        ──|<
Zero or many:       ──O<
```

Reading a relationship: start from one entity, read toward the other.
- `User ──||—O< Orders`: a user must have zero or many orders; each order belongs to exactly one user.

---

### Identifying vs Non-Identifying Relationships

**Identifying relationship**: The child entity cannot be uniquely identified without the parent. The parent's PK becomes part of the child's PK.

```sql
-- OrderItem cannot exist without Order — identifying
CREATE TABLE order_items (
    order_id BIGINT NOT NULL REFERENCES orders(id),
    line_number INT NOT NULL,
    product_id BIGINT NOT NULL REFERENCES products(id),
    PRIMARY KEY (order_id, line_number)  -- PK includes parent's PK
);
```

**Non-identifying relationship**: The child has its own independent identity. The FK to the parent is just a regular attribute (can be nullable or mandatory, but not part of PK).

```sql
-- Employee has its own identity regardless of department
CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    department_id BIGINT REFERENCES departments(id)  -- FK, not part of PK
);
```

---

### Weak Entities

A weak entity has no primary key of its own — it depends on a strong (parent) entity for identification. It uses a **partial key** (discriminator) combined with the parent's PK.

Classic example: `OrderItem` depends on `Order`. `Room` depends on `Building`.

```sql
-- Building is the strong entity
CREATE TABLE buildings (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Room is a weak entity: "Room 101 in Building A" is unique, but "Room 101" alone is not
CREATE TABLE rooms (
    building_id BIGINT NOT NULL REFERENCES buildings(id) ON DELETE CASCADE,
    room_number VARCHAR(10) NOT NULL,  -- partial key (discriminator)
    capacity INT,
    PRIMARY KEY (building_id, room_number)  -- composite PK = strong PK + partial key
);
```

---

## Normalization

Normalization is the process of structuring a relational database to reduce data redundancy and improve data integrity. Each normal form builds on the previous.

### First Normal Form (1NF)

**Rules:**
1. Every column must contain atomic (indivisible) values.
2. No repeating groups (no multiple columns for the same type of data).
3. Each row must be uniquely identifiable (has a primary key).

**Violation example:**

```
| order_id | customer | products              |
|----------|----------|-----------------------|
| 1        | Alice    | Laptop, Mouse, Keyboard|  ← not atomic
| 2        | Bob      | Monitor               |
```

**Fix: move products to a separate table.**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_products (
    order_id BIGINT REFERENCES orders(id),
    product_name VARCHAR(100),
    PRIMARY KEY (order_id, product_name)
);
```

**Repeating group violation:**

```
| emp_id | skill1 | skill2 | skill3 |
```

**Fix:**

```sql
CREATE TABLE employee_skills (
    emp_id BIGINT REFERENCES employees(id),
    skill VARCHAR(100),
    PRIMARY KEY (emp_id, skill)
);
```

---

### Second Normal Form (2NF)

**Applies only to tables with a composite primary key.**

**Rule**: Every non-key attribute must depend on the WHOLE composite primary key, not just part of it.

**Violation example (composite PK: student_id + course_id):**

```
| student_id | course_id | student_name | course_title  | grade |
```

- `student_name` depends only on `student_id` — partial dependency.
- `course_title` depends only on `course_id` — partial dependency.
- `grade` depends on both `student_id` AND `course_id` — full dependency. This is correct.

**Fix:**

```sql
CREATE TABLE students (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL
);

CREATE TABLE enrollments (
    student_id BIGINT REFERENCES students(id),
    course_id BIGINT REFERENCES courses(id),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

**Note**: If a table has a single-column PK, it is automatically in 2NF (partial dependency is impossible with one key column).

---

### Third Normal Form (3NF)

**Rule**: No non-key attribute should depend on another non-key attribute (no transitive dependencies).

A transitive dependency is: `non-key column A → non-key column B`, where A is not the PK.

**Violation example:**

```
| emp_id | emp_name | dept_id | dept_name |
```

- `dept_name` depends on `dept_id`, not on `emp_id`. That is a transitive dependency through `dept_id`.

**Fix:**

```sql
CREATE TABLE departments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    dept_id BIGINT REFERENCES departments(id)
);
```

**Memory aid**: "Every non-key attribute must depend on the key, the whole key, and nothing but the key."

---

### Boyce-Codd Normal Form (BCNF / 3.5NF)

**Rule**: Every determinant (the left side of a functional dependency) must be a candidate key.

BCNF is stricter than 3NF. A table in 3NF can violate BCNF when there are multiple overlapping candidate keys.

**Violation example:**

```
Table: Course_Teacher_Room
Candidate keys: (course, room) and (course, teacher)
Dependencies:
  teacher → room (a teacher always uses the same room)
  course, teacher → room (derived)
```

Here `teacher` is a determinant but not a candidate key — BCNF violation.

**Fix**: Decompose into:
- `Teacher_Room (teacher, room)` — teacher determines room
- `Course_Teacher (course, teacher)`

---

### Fourth Normal Form (4NF)

**Rule**: A table should not contain two or more independent multi-valued facts about an entity.

**Violation example:**

```
| employee | skill    | language |
|----------|----------|----------|
| Alice    | Java     | English  |
| Alice    | Java     | French   |
| Alice    | Python   | English  |
| Alice    | Python   | French   |
```

`skills` and `languages` are independent — no relationship between them. Storing together causes combinatorial explosion.

**Fix:**

```sql
CREATE TABLE employee_skills (emp_id BIGINT, skill VARCHAR(100));
CREATE TABLE employee_languages (emp_id BIGINT, language VARCHAR(100));
```

---

### When to Denormalize

Full normalization maximizes data integrity but can hurt read performance due to multiple joins. Denormalization intentionally introduces redundancy for performance.

**Common denormalization strategies:**

| Strategy | Example | Use Case |
|----------|---------|----------|
| Stored derived values | `total_price` on order | Avoid summing order_items on every read |
| Redundant columns | `user_email` on orders table | Avoid join to users table for reports |
| Duplicate tables | Read replicas / materialized views | Reporting vs transactional queries |
| Pre-joined tables | Denormalized analytics table | OLAP / data warehouse |

**Decision framework:**
- If read-heavy and joins are becoming a bottleneck, measure first (EXPLAIN ANALYZE), then denormalize selectively.
- Document every denormalization — future developers won't understand why data is duplicated without context.
- Create update triggers or application-level logic to keep denormalized data in sync.

```sql
-- Denormalized: store user_email on orders to avoid join in reporting queries
ALTER TABLE orders ADD COLUMN user_email VARCHAR(255);

-- Keep in sync with a trigger
CREATE OR REPLACE FUNCTION sync_user_email()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders SET user_email = NEW.email WHERE user_id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_email
AFTER UPDATE OF email ON users
FOR EACH ROW EXECUTE FUNCTION sync_user_email();
```

---

## Primary Key Strategies

### Surrogate vs Natural Keys

**Natural key**: A real-world attribute that uniquely identifies a record and is stable over time.

```sql
-- Natural key examples
CREATE TABLE countries (
    code CHAR(2) PRIMARY KEY,     -- ISO 3166-1 alpha-2: 'US', 'GB'
    name VARCHAR(100) NOT NULL
);

CREATE TABLE currencies (
    code CHAR(3) PRIMARY KEY,     -- ISO 4217: 'USD', 'EUR'
    name VARCHAR(100)
);
```

**When natural keys work well:**
- Truly unique (no two records share it)
- Stable (never changes — SSNs get reassigned, emails change)
- Short (char(2) or char(3) is fine as a FK target)
- Meaningful (improves query readability, avoids joins just to see the country name)

**Surrogate key**: A system-generated identifier with no business meaning.

```sql
-- Surrogate key: auto-increment integer
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- PostgreSQL
    -- id BIGINT AUTO_INCREMENT PRIMARY KEY  -- MySQL
    -- id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY  -- SQL standard
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL
);
```

**When surrogate keys are preferred (most cases):**
- Natural candidate is long (URL, email) — costly to store as FK in child tables
- Natural candidate might change (email, username)
- No clear natural unique identifier exists
- Merging data from multiple systems (each system has its own ID)

**Rule of thumb**: Always add a surrogate PK. Use natural keys as UNIQUE constraints, not PKs.

```sql
-- Best of both worlds
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,         -- surrogate: used as FK everywhere
    sku VARCHAR(50) UNIQUE NOT NULL,  -- natural: enforced, but not the PK
    name VARCHAR(255) NOT NULL
);
```

---

### UUID as Primary Key

UUID (Universally Unique Identifier) is a 128-bit number. Useful when:
- Distributed systems generate IDs independently (no central sequence)
- IDs must be unguessable (security: prevents enumeration attacks)
- Data merging across databases without collision

```sql
-- PostgreSQL: UUID primary key
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,  -- pg 13+, no extension needed
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- With uuid-ossp extension (older versions)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE orders (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

```java
// JPA with UUID — Java 17+ / Jakarta EE 3.1
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)  // JPA 3.1 standard
    private UUID id;

    // Older approach with Hibernate-specific annotation
    @Id
    @GeneratedValue(generator = "uuid2")
    @GenericGenerator(name = "uuid2", strategy = "uuid2")
    @Column(columnDefinition = "VARCHAR(36)")
    private String id;  // stored as string in MySQL

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

---

### UUID Variants: v4 vs v7 vs ULID vs Snowflake

**The B-tree fragmentation problem with UUID v4:**

UUID v4 is random. When you insert a new row, its UUID is random across the entire 128-bit space. In a clustered index (InnoDB in MySQL, any B-tree index), new rows must be inserted at a random position in the index, causing:
- Page splits (B-tree nodes must be split and rebalanced)
- Write amplification
- Poor cache locality (random disk seeks)
- Index fragmentation over time

**Comparison:**

| ID Type | Ordered? | URL Safe? | DB Dependency? | Size | Notes |
|---------|----------|-----------|----------------|------|-------|
| BIGSERIAL (auto-increment) | Yes | N/A | Yes (DB sequence) | 8 bytes | Best performance, guessable |
| UUID v4 | No (random) | No (with dashes) | No | 16 bytes | Fragmentation problem |
| UUID v7 | Yes (time-ordered) | No (with dashes) | No | 16 bytes | Fixes fragmentation, RFC 9562 |
| ULID | Yes (lexicographic) | Yes | No | 16 bytes | 48-bit time + 80-bit random |
| Snowflake ID | Yes (time-ordered) | N/A | Worker config needed | 8 bytes | Twitter/Discord approach |

```java
// UUID v7 in Java (Java 24+ has built-in; earlier requires library)
// Using java-uuid-generator library
import com.fasterxml.uuid.Generators;

UUID uuidV7 = Generators.timeBasedEpochGenerator().generate();
// Time-ordered: sequential inserts, no fragmentation

// ULID in Java
import de.huxhorn.sulky.ulid.ULID;
ULID ulid = new ULID();
String ulidStr = ulid.nextULID();  // "01ARZ3NDEKTSV4RRFFQ69G5FAV"
```

**Snowflake ID structure (64-bit):**
```
| 1 bit (sign) | 41 bits (timestamp ms) | 10 bits (worker/machine ID) | 12 bits (sequence) |
```
Gives ~69 years of timestamps, 1024 workers, 4096 IDs/ms per worker. Used by Twitter, Discord, Instagram.

```java
// Simple Snowflake ID generator
public class SnowflakeIdGenerator {
    private final long workerId;
    private long lastTimestamp = -1L;
    private long sequence = 0L;

    private static final long SEQUENCE_BITS = 12L;
    private static final long WORKER_BITS = 10L;
    private static final long MAX_SEQUENCE = ~(-1L << SEQUENCE_BITS);  // 4095

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {
                // Wait for next millisecond
                while ((timestamp = System.currentTimeMillis()) <= lastTimestamp) {}
            }
        } else {
            sequence = 0L;
        }
        lastTimestamp = timestamp;
        return (timestamp << (WORKER_BITS + SEQUENCE_BITS))
             | (workerId << SEQUENCE_BITS)
             | sequence;
    }
}
```

---

## Relationship Patterns

### One-to-One

Two entities share a strict 1:1 correspondence. Common use: extending a table without modifying it (open-closed principle for schemas).

```sql
-- Option 1: shared primary key (strongest coupling, child row shares PK with parent)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    -- user_id IS the PK and the FK — ensures exactly one profile per user
    bio TEXT,
    avatar_url VARCHAR(500),
    date_of_birth DATE,
    website_url VARCHAR(500)
);
```

```sql
-- Option 2: FK with UNIQUE constraint (more common, slightly looser coupling)
CREATE TABLE user_profiles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500)
);
-- UNIQUE on user_id ensures only one profile per user
```

**When to use each:**
- Shared PK: when profile cannot exist without user, and you want a strict guarantee. Zero chance of orphaned profiles.
- Separate PK + UNIQUE FK: when profile may be lazily created, when you need to distinguish between "no profile" and "user does not exist."

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true)
    private User user;

    private String bio;
}
```

**Important**: `@OneToOne` is `EAGER` by default in JPA. Always override to `LAZY` to prevent N+1 style problems when loading a list of users.

---

### One-to-Many

The most common relationship. A parent entity has multiple child entities. The FK lives on the child (many) side.

```sql
-- Department has many employees
CREATE TABLE departments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    location VARCHAR(100)
);

CREATE TABLE employees (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    salary DECIMAL(10,2),
    department_id BIGINT NOT NULL REFERENCES departments(id),  -- FK on many side
    manager_id BIGINT REFERENCES employees(id),  -- self-referential FK (nullable)
    hired_at DATE DEFAULT CURRENT_DATE
);

-- CRITICAL: Always index foreign keys used in JOINs and WHERE clauses
CREATE INDEX idx_employees_department_id ON employees(department_id);
CREATE INDEX idx_employees_manager_id ON employees(manager_id);
```

**Why index FKs?** Without an index, `SELECT * FROM employees WHERE department_id = 5` performs a full table scan. With the index, it's an O(log n) lookup. For cascading deletes/updates, the DB engine also uses FK indexes to find child rows.

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // mappedBy = the field name in Employee that owns the relationship
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();
}

@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // The owning side: has the @JoinColumn
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id", nullable = false)
    private Department department;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "manager_id")
    private Employee manager;

    @OneToMany(mappedBy = "manager", fetch = FetchType.LAZY)
    private List<Employee> reports = new ArrayList<>();
}
```

**Hibernate tip**: Always initialize collection fields (`= new ArrayList<>()`) to avoid `NullPointerException` when the collection is empty.

---

### Many-to-Many with Junction Table

A pure M:N relationship resolves to two 1:N relationships through a junction (associative/bridge) table.

```sql
-- Students enroll in Courses
CREATE TABLE students (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    credits INT NOT NULL DEFAULT 3
);

-- Junction table with composite PK and additional attributes
CREATE TABLE enrollments (
    student_id BIGINT NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id BIGINT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    grade CHAR(2),                     -- A, B+, C, etc.
    status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, WITHDRAWN, COMPLETED
    PRIMARY KEY (student_id, course_id) -- composite PK prevents duplicate enrollments
);

-- Index the reverse direction (course → students) for queries like "who's in this course?"
CREATE INDEX idx_enrollments_course_id ON enrollments(course_id);
```

**When the junction table becomes a full entity**: If the junction table has its own meaningful attributes (like `grade`, `enrolled_at`, `status`), model it as a proper entity.

```java
// Simple M:N (no extra attributes) — let Hibernate manage the junction table
@Entity
public class Student {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

// M:N with extra attributes — model junction table as entity
@Entity
@Table(name = "enrollments")
public class Enrollment {

    @EmbeddedId
    private EnrollmentId id;  // composite key

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
    private String status;
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

Hard delete (`DELETE FROM table WHERE id = ?`) is permanent and irreversible. Soft delete marks a record as deleted without physically removing it.

**Reasons to use soft delete:**
- **Audit trail**: regulatory compliance (GDPR right to erasure is different — see below), financial records, legal requirements
- **Recovery**: accidentally deleted records can be restored without restoring a backup
- **Referential integrity**: child records that reference a deleted parent still make sense (e.g., order history for a deleted user account)
- **Soft dependencies**: other systems may cache or reference the ID

**Trade-offs:**
- All queries must filter out soft-deleted rows (`WHERE deleted_at IS NULL`) — easy to forget
- Unique constraints become complicated (see below)
- Table grows indefinitely; need archival strategy
- Index size increases (must include deleted rows)

---

### Implementing Soft Delete in SQL

```sql
-- Add soft delete column to existing table
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Soft delete a user
UPDATE users SET deleted_at = NOW() WHERE id = 42;

-- Query active users
SELECT * FROM users WHERE deleted_at IS NULL;

-- Restore a deleted user
UPDATE users SET deleted_at = NULL WHERE id = 42;

-- Query only deleted users (admin use)
SELECT * FROM users WHERE deleted_at IS NOT NULL;

-- Partial index: only index non-deleted rows (faster, smaller index)
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;

-- Partial unique index: unique email among active users only
CREATE UNIQUE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;
-- This allows multiple deleted users with the same email (they were re-registered)
```

**Why partial unique index beats `UNIQUE(email, deleted_at)`:**

```sql
-- WRONG approach: unique on (email, deleted_at)
CREATE UNIQUE INDEX idx ON users(email, deleted_at);
-- Problem: deleted_at IS NULL for active users
-- In SQL, NULL != NULL, so (email, NULL) satisfies uniqueness...
-- But only in some DBs. PostgreSQL: multiple (email, NULL) allowed!
-- This does NOT enforce "one active user per email."

-- RIGHT approach: partial unique index
CREATE UNIQUE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;
-- Only active (non-deleted) users are indexed — uniqueness enforced only for them
```

---

### Implementing Soft Delete with JPA/Hibernate

```java
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.SQLRestriction;

@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
// Hibernate 6.3+: @SQLRestriction replaces deprecated @Where
@SQLRestriction("deleted_at IS NULL")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String name;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    // Hibernate intercepts repository.delete(user) and runs the SQLDelete instead
    // All findAll(), findById() automatically add WHERE deleted_at IS NULL
}
```

```java
// To query deleted users — bypass @SQLRestriction with native query
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Uses @SQLRestriction automatically — active users only
    List<User> findByEmail(String email);

    // Native query bypasses @SQLRestriction
    @Query(value = "SELECT * FROM users WHERE id = ?1", nativeQuery = true)
    Optional<User> findByIdIncludingDeleted(Long id);

    // JPQL with filter override — Hibernate 6 approach
    @Query("SELECT u FROM User u WHERE u.deletedAt IS NOT NULL")
    // @SQLRestriction is applied at the entity level; JPQL WHERE adds additional filter
    // To truly bypass, use native query or a separate @Entity without @SQLRestriction
    List<User> findDeletedUsers();
}
```

**Important Hibernate behavior**: `entityManager.remove(entity)` is intercepted by `@SQLDelete` and converted to an UPDATE. `entityManager.find()` and all JPQL queries automatically include the `@SQLRestriction` filter. Native queries bypass it.

---

### GDPR and Soft Delete

GDPR "right to erasure" (Article 17) requires truly removing personal data, which conflicts with keeping soft-deleted records. Solutions:

1. **Anonymize on soft delete**: Replace PII with nulls or hashed values.
2. **Separate PII into a personal_data table**: Soft-delete the entity, hard-delete the PII.
3. **Scheduled hard delete**: Soft delete immediately, hard delete after retention period.

```sql
-- Anonymize approach: replace PII with nulls on soft delete
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

The minimum audit information: who created, when created, who last modified, when last modified.

```sql
-- Base structure for auditable tables
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    total DECIMAL(10,2),
    -- Audit columns
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by BIGINT REFERENCES users(id),
    updated_by BIGINT REFERENCES users(id)
);

-- Auto-update trigger for updated_at
CREATE OR REPLACE FUNCTION fn_update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION fn_update_updated_at();
```

**Reusable trigger function**: Because the function returns a generic TRIGGER type, the same function `fn_update_updated_at()` can be attached to multiple tables.

```sql
-- Same function, different table
CREATE TRIGGER trg_products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION fn_update_updated_at();
```

---

### Spring Data JPA Auditing

Spring Data JPA has built-in auditing support that eliminates manual `created_at`/`updated_at` management.

```java
// 1. Enable JPA auditing in configuration
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    public AuditorAware<Long> auditorProvider() {
        return () -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null || !auth.isAuthenticated()) {
                return Optional.empty();
            }
            // Assuming your UserDetails implementation exposes getId()
            return Optional.ofNullable(((AppUserDetails) auth.getPrincipal()).getId());
        };
    }
}

// 2. Base entity (MappedSuperclass — no table of its own)
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private Long createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private Long updatedBy;
}

// 3. Extend in your entities
@Entity
@Table(name = "orders")
public class Order extends Auditable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private BigDecimal total;
}
```

---

### Full Audit Trail (History Table / Change Log)

For compliance-heavy domains (banking, healthcare), you need a full history of every change, including who changed what and what the previous value was.

```sql
-- Audit table for the orders table
CREATE TABLE orders_audit (
    audit_id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    changed_by BIGINT,
    client_ip INET,
    old_data JSONB,    -- previous state (NULL for INSERT)
    new_data JSONB,    -- new state (NULL for DELETE)
    changed_columns TEXT[]  -- which columns actually changed (UPDATE only)
);

CREATE INDEX idx_orders_audit_order_id ON orders_audit(order_id);
CREATE INDEX idx_orders_audit_changed_at ON orders_audit(changed_at);

-- Trigger function
CREATE OR REPLACE FUNCTION fn_audit_orders()
RETURNS TRIGGER AS $$
DECLARE
    changed_cols TEXT[] := '{}';
    col_name TEXT;
BEGIN
    -- For UPDATE: detect which columns changed
    IF TG_OP = 'UPDATE' THEN
        FOREACH col_name IN ARRAY (
            SELECT column_name FROM information_schema.columns
            WHERE table_name = 'orders'
        )
        LOOP
            IF row_to_json(OLD)->col_name IS DISTINCT FROM row_to_json(NEW)->col_name THEN
                changed_cols := array_append(changed_cols, col_name);
            END IF;
        END LOOP;
    END IF;

    INSERT INTO orders_audit (
        order_id, operation, changed_by,
        old_data, new_data, changed_columns
    ) VALUES (
        COALESCE(NEW.id, OLD.id),
        TG_OP,
        current_setting('app.current_user_id', true)::BIGINT,
        CASE WHEN TG_OP != 'INSERT' THEN to_jsonb(OLD) END,
        CASE WHEN TG_OP != 'DELETE' THEN to_jsonb(NEW) END,
        CASE WHEN TG_OP = 'UPDATE' THEN changed_cols END
    );

    RETURN NULL;  -- AFTER trigger; return value is ignored for AFTER row triggers
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION fn_audit_orders();
```

---

### Temporal Tables (Bitemporal Design)

Bitemporal modeling tracks two dimensions of time:
- **Valid time**: when the data was true in the real world
- **Transaction time**: when the data was recorded in the database

```sql
-- Product prices with valid time (effective dating)
CREATE TABLE product_prices (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    price DECIMAL(10,2) NOT NULL,
    valid_from DATE NOT NULL,
    valid_to DATE NOT NULL DEFAULT '9999-12-31',  -- open-ended current records

    -- Ensure no overlapping price periods for the same product
    CONSTRAINT no_overlap EXCLUDE USING gist (
        product_id WITH =,
        daterange(valid_from, valid_to, '[)') WITH &&
    )
    -- Requires btree_gist extension: CREATE EXTENSION btree_gist;
);

-- What is the current price?
SELECT price FROM product_prices
WHERE product_id = 1
  AND valid_from <= CURRENT_DATE
  AND valid_to > CURRENT_DATE;

-- What was the price on 2023-01-01?
SELECT price FROM product_prices
WHERE product_id = 1
  AND valid_from <= '2023-01-01'
  AND valid_to > '2023-01-01';

-- Update price (close old record, open new record)
BEGIN;
UPDATE product_prices
SET valid_to = CURRENT_DATE
WHERE product_id = 1 AND valid_to = '9999-12-31';

INSERT INTO product_prices (product_id, price, valid_from)
VALUES (1, 29.99, CURRENT_DATE);
COMMIT;
```

**SQL:2011 standard temporal tables** are supported natively in some DBs (MySQL 8.0+, MariaDB, MS SQL Server). PostgreSQL requires manual implementation as above.

---

## Multi-Tenancy Patterns

Multi-tenancy means one software instance serves multiple customers (tenants), each with their own isolated data.

---

### Pattern 1: Database-per-Tenant

Each tenant gets a completely separate database.

```
Tenant A → database_a
Tenant B → database_b
Tenant C → database_c
```

**Pros:**
- Maximum isolation: a bug or migration error in one tenant cannot affect others
- Easy per-tenant backup, restore, and deletion
- Can run on different hardware if needed
- Compliance friendly (data sovereignty — tenant data can be stored in specific regions)

**Cons:**
- Very high cost at scale (hundreds of databases = hundreds of connection pools)
- Complex connection management (need routing layer)
- Schema migrations must be applied to every database

**Use when:** healthcare, finance, government — high regulatory isolation requirements; small number of large enterprise tenants.

---

### Pattern 2: Schema-per-Tenant

One database, one schema per tenant.

```
database: saas_db
  ├── schema: tenant_a
  │     ├── orders
  │     ├── products
  │     └── users
  ├── schema: tenant_b
  │     ├── orders
  │     └── ...
```

```sql
-- Create a new tenant schema
CREATE SCHEMA tenant_abc;

-- Create tables in that schema
CREATE TABLE tenant_abc.users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- Set search_path to query tenant's tables without prefix
SET search_path TO tenant_abc;
SELECT * FROM users;  -- queries tenant_abc.users
```

```java
// Spring Boot + Hibernate: schema-per-tenant
// 1. MultiTenantConnectionProvider: switches schema per request
public class SchemaMultiTenantConnectionProvider
        extends AbstractMultiTenantConnectionProvider<String> {

    private final DataSource dataSource;

    @Override
    protected DataSource selectAnyDataSource() { return dataSource; }

    @Override
    protected DataSource selectDataSource(String tenantIdentifier) { return dataSource; }

    @Override
    public Connection getConnection(String tenantIdentifier) throws SQLException {
        Connection connection = dataSource.getConnection();
        connection.createStatement().execute("SET search_path TO " + tenantIdentifier);
        return connection;
    }
}

// 2. CurrentTenantIdentifierResolver: gets tenant from request context
@Component
public class TenantIdentifierResolver implements CurrentTenantIdentifierResolver<String> {

    @Override
    public String resolveCurrentTenantIdentifier() {
        return TenantContext.getCurrentTenant();
    }

    @Override
    public boolean validateExistingCurrentSessions() { return true; }
}

// 3. Request filter: set tenant from request header
@Component
public class TenantFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String tenantId = request.getHeader("X-Tenant-ID");
        TenantContext.setCurrentTenant(tenantId);
        try {
            filterChain.doFilter(request, response);
        } finally {
            TenantContext.clear();  // CRITICAL: clear after request to avoid thread leakage
        }
    }
}
```

---

### Pattern 3: Shared Schema (Row-Level Multi-Tenancy)

All tenants in the same tables, distinguished by a `tenant_id` column.

```sql
-- Every tenant-scoped table has tenant_id
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    total DECIMAL(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Composite index: tenant_id first (most selective filter)
CREATE INDEX idx_orders_tenant_status ON orders(tenant_id, status);
CREATE INDEX idx_orders_tenant_user ON orders(tenant_id, user_id);
```

**PostgreSQL Row Level Security (RLS):**

RLS is a database-level enforcement mechanism that prevents cross-tenant data access even if application code has a bug.

```sql
-- Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- also applies to table owner

-- Create policy: users can only see their tenant's data
CREATE POLICY tenant_isolation_policy ON orders
    AS PERMISSIVE
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);

-- Application must set this before every query
SET app.current_tenant_id = '42';
-- or in a transaction:
BEGIN;
SET LOCAL app.current_tenant_id = '42';
SELECT * FROM orders;  -- automatically filtered to tenant 42
COMMIT;
```

```java
// Spring: set RLS variable in a Hibernate event listener or interceptor
@Component
public class TenantRlsInterceptor extends EmptyInterceptor {

    @Override
    public void beforeTransactionCompletion(Transaction tx) { }

    // Better: use AOP or connection event listener
    // Set on connection acquisition:
    @Bean
    public DataSourceConnectionProvider dataSourceConnectionProvider(DataSource dataSource) {
        return new TenantAwareConnectionProvider(dataSource);
    }
}

// Or use Hibernate's StatementInspector to prepend SET LOCAL
@Component
public class TenantStatementInspector implements StatementInspector {
    // Intercept before each statement — set tenant context
}
```

**Comparison Table:**

| Pattern | Isolation | DB Cost | Migration Complexity | Tenant Count |
|---------|-----------|---------|---------------------|--------------|
| DB-per-tenant | Highest | Highest | Highest (one DB per) | Tens |
| Schema-per-tenant | High | Medium | Medium (one schema per) | Hundreds |
| Shared schema | Low (RLS helps) | Lowest | Low (one migration) | Thousands+ |

---

## Hierarchical Data

Hierarchical data has parent-child relationships of arbitrary depth: categories, org charts, file systems, threaded comments.

### Adjacency List (Simple Self-Referential FK)

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id BIGINT REFERENCES categories(id) ON DELETE CASCADE
    -- parent_id IS NULL means root node
);

INSERT INTO categories (name, parent_id) VALUES
    ('Electronics', NULL),     -- id=1, root
    ('Computers', 1),          -- id=2, child of Electronics
    ('Laptops', 2),            -- id=3, child of Computers
    ('Gaming Laptops', 3);     -- id=4, child of Laptops

-- Find all descendants of "Electronics" (id=1) using recursive CTE
WITH RECURSIVE category_tree AS (
    -- Anchor: start node
    SELECT id, name, parent_id, 0 AS depth, ARRAY[id] AS path
    FROM categories
    WHERE id = 1

    UNION ALL

    -- Recursive: find children of current level
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || c.id
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT id, name, depth,
       REPEAT('  ', depth) || name AS indented_name
FROM category_tree
ORDER BY path;

-- Find all ancestors of "Gaming Laptops" (id=4) — traverse upward
WITH RECURSIVE ancestors AS (
    SELECT id, name, parent_id FROM categories WHERE id = 4
    UNION ALL
    SELECT c.id, c.name, c.parent_id
    FROM categories c
    JOIN ancestors a ON c.id = a.parent_id
)
SELECT * FROM ancestors;
```

**Pros**: Simple, easy to insert/move nodes, natural representation.
**Cons**: Retrieving a full subtree requires recursive CTE (or application-level recursion); depth is not directly stored.

---

### Nested Set Model

Each node has `lft` and `rgt` values. All descendants have `lft`/`rgt` values between the parent's `lft` and `rgt`.

```sql
CREATE TABLE categories_ns (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    lft INT NOT NULL,
    rgt INT NOT NULL
);

-- Example tree (Electronics → Computers → Laptops → Gaming Laptops):
-- Electronics:     lft=1, rgt=8
-- Computers:       lft=2, rgt=7
-- Laptops:         lft=3, rgt=6
-- Gaming Laptops:  lft=4, rgt=5

-- Get all descendants of Electronics (id=1, lft=1, rgt=8)
SELECT c.* FROM categories_ns AS c, categories_ns AS parent
WHERE c.lft BETWEEN parent.lft + 1 AND parent.rgt - 1
  AND parent.id = 1
ORDER BY c.lft;

-- Get all ancestors of Gaming Laptops (id=4, lft=4, rgt=5)
SELECT parent.* FROM categories_ns AS node, categories_ns AS parent
WHERE node.lft BETWEEN parent.lft AND parent.rgt
  AND node.id = 4
ORDER BY parent.lft;

-- Leaf nodes: rgt = lft + 1 (no children)
SELECT * FROM categories_ns WHERE rgt = lft + 1;

-- Immediate children count: (rgt - lft - 1) / 2
```

**Pros**: Very fast reads for getting entire subtrees (range query, no recursion).
**Cons**: Inserts and moves require updating `lft`/`rgt` values for many rows — O(n) writes.

---

### Closure Table (Best Balance)

Stores every ancestor-descendant pair explicitly in a separate table.

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE category_closure (
    ancestor_id BIGINT NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    descendant_id BIGINT NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    depth INT NOT NULL,  -- 0 = self-reference, 1 = parent-child, 2 = grandparent, etc.
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- Create index on descendant for "find all ancestors" queries
CREATE INDEX idx_closure_descendant ON category_closure(descendant_id);

-- Insert node "Electronics" (id=1): insert self-reference
INSERT INTO category_closure (ancestor_id, descendant_id, depth) VALUES (1, 1, 0);

-- Insert child "Computers" (id=2) under "Electronics" (id=1):
-- 1. Self-reference for new node
INSERT INTO category_closure VALUES (2, 2, 0);
-- 2. Copy ancestor's paths, increment depth
INSERT INTO category_closure (ancestor_id, descendant_id, depth)
SELECT ancestor_id, 2, depth + 1
FROM category_closure
WHERE descendant_id = 1;  -- all ancestors of parent (1), now become ancestors of new node (2)

-- All descendants of Electronics (id=1)
SELECT c.* FROM categories c
JOIN category_closure cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 1 AND cc.depth > 0
ORDER BY cc.depth;

-- All ancestors of Gaming Laptops (id=4)
SELECT c.* FROM categories c
JOIN category_closure cc ON c.id = cc.ancestor_id
WHERE cc.descendant_id = 4 AND cc.depth > 0
ORDER BY cc.depth DESC;  -- root first

-- Immediate children only (depth = 1)
SELECT c.* FROM categories c
JOIN category_closure cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 1 AND cc.depth = 1;
```

**Pros**: Fast reads in all directions (descendants, ancestors, subtrees). Depth is directly available.
**Cons**: More storage (O(n) rows per path). Inserts are O(depth) rows. Moves require deleting and reinserting paths — O(subtree size * average depth).

**Comparison Summary:**

| Model | Read (subtree) | Read (ancestors) | Insert | Move | Storage |
|-------|---------------|-----------------|--------|------|---------|
| Adjacency list | Recursive CTE | Recursive CTE | O(1) | O(1) | Minimal |
| Nested set | O(log n) range | O(log n) range | O(n) | O(n) | Minimal |
| Closure table | O(1) join | O(1) join | O(depth) | O(subtree × depth) | O(n × depth) |

**Recommendation**: Use adjacency list with recursive CTE for most use cases. Use closure table when you query ancestors/descendants frequently in a large tree. Avoid nested set unless you have a mostly static tree with heavy read requirements.

---

## Pagination Patterns

### Offset Pagination

```sql
-- Get page 3 (0-indexed), 20 items per page
SELECT * FROM orders
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;  -- skip first 40 rows
```

**Problems:**

1. **Performance degrades at scale**: `OFFSET 10000` means the DB engine must read and discard 10,000 rows even though you only need 20. O(offset) cost.

2. **Page drift**: If a new order is inserted between page 1 and page 2 requests, items shift — you see duplicates or miss items.

3. **Count queries**: Total count (`COUNT(*)`) for pagination metadata is expensive on large tables.

```java
// Spring Data JPA offset pagination
Page<Order> findByUserId(Long userId, Pageable pageable);

// Service layer
Pageable pageable = PageRequest.of(pageNumber, pageSize,
    Sort.by(Sort.Direction.DESC, "createdAt"));
Page<Order> page = orderRepository.findByUserId(userId, pageable);

// Page metadata
page.getTotalElements()  // total count (expensive!)
page.getTotalPages()
page.getNumber()         // current page
page.getContent()        // List<Order>
```

---

### Keyset Pagination (Cursor-Based)

Uses the last item's values as the starting point for the next page instead of a row offset.

```sql
-- First page
SELECT id, title, created_at FROM articles
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Suppose last item was: created_at='2024-03-15 09:00:00', id=987

-- Next page: fetch items "before" the cursor
SELECT id, title, created_at FROM articles
WHERE (created_at < '2024-03-15 09:00:00')
   OR (created_at = '2024-03-15 09:00:00' AND id < 987)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Simplified with row value comparison (PostgreSQL supports this)
SELECT id, title, created_at FROM articles
WHERE (created_at, id) < ('2024-03-15 09:00:00', 987)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Index to support this query efficiently
CREATE INDEX idx_articles_created_id ON articles(created_at DESC, id DESC);
```

```java
// Keyset pagination in Spring Data JPA
public interface ArticleRepository extends JpaRepository<Article, Long> {

    @Query("SELECT a FROM Article a " +
           "WHERE a.createdAt < :lastCreatedAt " +
           "OR (a.createdAt = :lastCreatedAt AND a.id < :lastId) " +
           "ORDER BY a.createdAt DESC, a.id DESC")
    List<Article> findNextPage(
        @Param("lastCreatedAt") LocalDateTime lastCreatedAt,
        @Param("lastId") Long lastId,
        Pageable pageable);
}

// Cursor is typically encoded as Base64 JSON for API clients
public record CursorPage<T>(
    List<T> content,
    String nextCursor,   // Base64-encoded last item's sort key
    boolean hasNext
) {}

// Encode cursor
String cursor = Base64.getEncoder().encodeToString(
    String.format("%s,%d", lastItem.getCreatedAt(), lastItem.getId()).getBytes()
);
```

**Offset vs Keyset comparison:**

| Aspect | Offset | Keyset |
|--------|--------|--------|
| Performance | O(offset) — degrades | O(log n) — consistent |
| Page drift | Yes (unstable pages) | No (cursor is stable) |
| Random access | Yes (`page=5` directly) | No (must iterate) |
| Total count | Easy (but expensive) | Hard |
| Implementation | Simple | Requires stable unique sort key |
| API design | `?page=2&size=20` | `?cursor=abc123&size=20` |

**When to use keyset**: Any API that returns a feed or timeline. When table has millions of rows. When real-time data changes are frequent.

**When to use offset**: Admin panels with random page access. Small tables (< 100k rows). When total count display is required.

---

## Common Domain Schemas

### E-Commerce Schema

A well-designed e-commerce schema handles product variants, order line items (with price snapshots), and inventory.

```sql
-- Categories (hierarchical using adjacency list)
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);

-- Products
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id BIGINT REFERENCES categories(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Product variants: a product can have multiple SKUs (size=S/M/L, color=red/blue)
CREATE TABLE product_variants (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(100) UNIQUE NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    is_active BOOLEAN NOT NULL DEFAULT true
);

-- Variant attributes: key-value pairs (color=red, size=XL)
CREATE TABLE variant_attributes (
    variant_id BIGINT NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    attribute_name VARCHAR(50) NOT NULL,
    attribute_value VARCHAR(100) NOT NULL,
    PRIMARY KEY (variant_id, attribute_name)
);

-- Users
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Addresses (a user can have multiple)
CREATE TABLE addresses (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(20) DEFAULT 'SHIPPING',  -- SHIPPING, BILLING
    street VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country CHAR(2) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    is_default BOOLEAN DEFAULT false
);

-- Orders
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING'
        CHECK (status IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED')),
    shipping_address_id BIGINT REFERENCES addresses(id),
    subtotal DECIMAL(10,2) NOT NULL,
    tax DECIMAL(10,2) NOT NULL DEFAULT 0,
    shipping_fee DECIMAL(10,2) NOT NULL DEFAULT 0,
    total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

-- Order line items: snapshot the price at purchase time
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id),
    -- SNAPSHOT: copy price/name at purchase time — never reference live product data for past orders
    product_name VARCHAR(255) NOT NULL,   -- snapshot
    variant_sku VARCHAR(100) NOT NULL,    -- snapshot
    unit_price DECIMAL(10,2) NOT NULL,    -- snapshot: price at time of purchase
    quantity INT NOT NULL CHECK (quantity > 0),
    subtotal DECIMAL(10,2) GENERATED ALWAYS AS (unit_price * quantity) STORED
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

**Critical design decision — price snapshot**: Store `unit_price` as a snapshot in `order_items`. Never JOIN to `product_variants.price` when displaying historical orders — if the price changes, historical orders must show the original price paid.

---

### RBAC Schema (Role-Based Access Control)

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT true
);

CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,  -- 'ADMIN', 'MANAGER', 'VIEWER'
    description TEXT
);

CREATE TABLE permissions (
    id BIGSERIAL PRIMARY KEY,
    resource VARCHAR(100) NOT NULL,  -- 'orders', 'products', 'users'
    action VARCHAR(50) NOT NULL,     -- 'read', 'write', 'delete', 'approve'
    UNIQUE (resource, action)
);

-- Many-to-many: users have roles
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    granted_by BIGINT REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);

-- Many-to-many: roles have permissions
CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

-- Check if user has a specific permission (one query)
SELECT COUNT(*) > 0 AS has_permission
FROM users u
JOIN user_roles ur ON u.id = ur.user_id
JOIN role_permissions rp ON ur.role_id = rp.role_id
JOIN permissions p ON rp.permission_id = p.id
WHERE u.id = :userId
  AND p.resource = :resource
  AND p.action = :action;
```

```java
// Spring Security integration
@Service
public class PermissionService {

    public boolean hasPermission(Long userId, String resource, String action) {
        return userRoleRepository.existsPermission(userId, resource, action);
    }
}

// Custom Security Expression
@PreAuthorize("@permissionService.hasPermission(authentication.principal.id, 'orders', 'write')")
public Order createOrder(CreateOrderRequest request) { ... }
```

---

## Polymorphic Associations and JPA Inheritance

Polymorphism in schema design handles scenarios where one entity type can be one of several subtypes. Example: `Payment` can be `CreditCardPayment`, `PayPalPayment`, or `BankTransfer`.

### Single Table Inheritance (STI)

All subtypes in one table. A discriminator column identifies the subtype.

```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(30) NOT NULL,        -- discriminator: 'CREDIT_CARD', 'PAYPAL', 'BANK'
    order_id BIGINT REFERENCES orders(id),
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMPTZ DEFAULT NOW(),

    -- Credit card specific (NULL for other types)
    card_last_four CHAR(4),
    card_brand VARCHAR(20),
    card_expiry_month INT,
    card_expiry_year INT,
    card_token VARCHAR(255),  -- tokenized, never store actual card numbers

    -- PayPal specific
    paypal_transaction_id VARCHAR(100),
    paypal_payer_email VARCHAR(255),

    -- Bank transfer specific
    bank_name VARCHAR(100),
    account_number_last_four CHAR(4),
    routing_number VARCHAR(20),
    transfer_reference VARCHAR(100)
);
```

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type", discriminatorType = DiscriminatorType.STRING)
public abstract class Payment {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;

    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    private PaymentStatus status;

    private LocalDateTime createdAt;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    private String cardLastFour;
    private String cardBrand;
    private String cardToken;
}

@Entity
@DiscriminatorValue("PAYPAL")
public class PayPalPayment extends Payment {
    private String paypalTransactionId;
    private String paypalPayerEmail;
}
```

**Pros**: No JOINs needed — fast polymorphic queries. Simple schema.
**Cons**: Many NULL columns. Subtype-specific constraints are impossible (can't add NOT NULL to a subtype column). Table grows wide as subtypes grow.

---

### Joined Table Inheritance (Normalized)

Base table for shared columns, one table per subtype for specific columns. JPA JOINs them.

```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(30) NOT NULL,
    order_id BIGINT REFERENCES orders(id),
    amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE credit_card_payments (
    id BIGINT PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
    card_last_four CHAR(4) NOT NULL,
    card_brand VARCHAR(20) NOT NULL,
    card_token VARCHAR(255)
);

CREATE TABLE paypal_payments (
    id BIGINT PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
    transaction_id VARCHAR(100) NOT NULL,
    payer_email VARCHAR(255) NOT NULL
);
```

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "type")
public abstract class Payment { ... }

@Entity
@Table(name = "credit_card_payments")
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    // id is shared with payments.id via @PrimaryKeyJoinColumn (implicit)
    @Column(nullable = false)
    private String cardLastFour;
}
```

**Pros**: Fully normalized. Subtype-specific NOT NULL constraints work correctly. No null columns.
**Cons**: Every query JOINs base + subtype tables. Polymorphic query (all payment types) uses LEFT JOINs or UNION.

---

### Table Per Class Inheritance

Each concrete class has a complete table with all columns (inherited + specific). No shared base table.

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.TABLE)
    // Cannot use IDENTITY with TABLE_PER_CLASS — requires TABLE or SEQUENCE strategy
    private Long id;
    private BigDecimal amount;
}

@Entity
@Table(name = "credit_card_payments")
public class CreditCardPayment extends Payment {
    // All Payment columns duplicated here
    private String cardLastFour;
}
```

**Pros**: Clean subtype tables. No NULL columns. Fast queries per concrete type.
**Cons**: Polymorphic queries (`SELECT all payments`) require UNION ALL across all tables — expensive. Cannot use `IDENTITY` strategy (no single sequence). Rarely recommended.

---

### Comparison Table

| Strategy | Join Needed? | Null Columns | Polymorphic Query | Subtype Constraints |
|----------|-------------|--------------|-------------------|---------------------|
| SINGLE_TABLE | No | Many | Fast (single table) | Not possible |
| JOINED | Yes (JOIN) | None | Moderate (outer join) | Yes (per subtype) |
| TABLE_PER_CLASS | No | None | Expensive (UNION ALL) | Yes (per table) |

**Recommendation**: Use `SINGLE_TABLE` for simple polymorphism with few subtypes and shared attributes. Use `JOINED` for normalized domains where subtype constraints matter.

---

## Schema Migrations with Flyway and Liquibase

### Why Migrations?

Without a migration tool, schema changes are manually applied to each environment — error-prone and untrackable. Migration tools:
- Version-control schema changes alongside code
- Automate applying migrations in CI/CD pipelines
- Track which migrations have been applied (via a history table)
- Support rollback (Liquibase) or forward-only (Flyway default)

---

### Flyway

Flyway uses versioned SQL scripts. Scripts are named `V{version}__{description}.sql`.

```
src/main/resources/db/migration/
  V1__create_users.sql
  V2__create_orders.sql
  V3__add_user_roles.sql
  V4__add_deleted_at_to_users.sql
```

```sql
-- V1__create_users.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- V4__add_deleted_at_to_users.sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
CREATE UNIQUE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;
```

```properties
# application.properties
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true  # for existing databases not yet managed
spring.flyway.validate-on-migrate=true   # checksums must match
```

```java
// Spring Boot auto-configures Flyway. Manual configuration:
@Bean
public Flyway flyway(DataSource dataSource) {
    Flyway flyway = Flyway.configure()
        .dataSource(dataSource)
        .locations("classpath:db/migration")
        .baselineOnMigrate(true)
        .load();
    flyway.migrate();
    return flyway;
}
```

**Flyway rules**:
- Never modify an applied migration file — checksums are stored and validated
- Always create a new migration file for changes
- Use repeatable migrations (`R__`) for views, functions, stored procedures (re-run when checksum changes)

---

### Liquibase

Liquibase supports XML, YAML, JSON, and SQL changesets. Each change has a unique `id` + `author`.

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-users.yaml
  - include:
      file: db/changelog/changes/002-create-orders.yaml
```

```yaml
# db/changelog/changes/001-create-users.yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users
      author: dev-team
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: created_at
                  type: TIMESTAMPTZ
                  defaultValueComputed: NOW()
      rollback:
        - dropTable:
            tableName: users
```

**Liquibase vs Flyway:**

| Feature | Flyway | Liquibase |
|---------|--------|-----------|
| Format | SQL (primary) | XML/YAML/JSON/SQL |
| Rollback | Manual (Undo scripts) | Built-in rollback |
| Diff tool | Teams plan | Built-in |
| Learning curve | Low | Medium |
| Multi-DB support | Good | Excellent |
| Spring Boot integration | Auto-config | Auto-config |

---

### Best Practices for Schema Migrations in CI/CD

```
Local dev:
  1. Write migration file
  2. Start app — Flyway auto-applies
  3. Test

CI pipeline:
  1. Spin up test DB
  2. Run Flyway migrate
  3. Run tests
  4. Flyway validate (checksum check)

Deployment (zero-downtime):
  1. Apply migration (must be backward-compatible with current code)
  2. Deploy new code version
  3. Run cleanup migration if needed

Zero-downtime migration rules:
  - Never DROP a column in the same release as removing code that uses it
  - Never RENAME a column (add new, copy data, remove old — 3 releases)
  - Never add NOT NULL without a DEFAULT (existing rows fail constraint)
```

---

## Indexing Strategies

### Types of Indexes

```sql
-- B-tree index (default, for =, <, >, BETWEEN, LIKE 'abc%')
CREATE INDEX idx_orders_status ON orders(status);

-- Composite index (order matters: most selective or most filtered column first)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Supports: WHERE user_id=? AND status=?
-- Supports: WHERE user_id=? (leftmost prefix rule)
-- Does NOT support: WHERE status=? alone (not covered by leftmost prefix)

-- Partial index (index only a subset of rows)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'PENDING';
-- Smaller, faster for queries filtering on status='PENDING'

-- Covering index (includes all columns needed by query — avoids table lookup)
CREATE INDEX idx_orders_covering ON orders(user_id, status)
INCLUDE (total, created_at);
-- Query: SELECT total, created_at FROM orders WHERE user_id=1 AND status='DELIVERED'
-- Answered entirely from index — no heap access

-- Hash index (only =, faster than B-tree for equality)
CREATE INDEX idx_users_email_hash ON users USING HASH (email);

-- GIN index (for JSONB, arrays, full-text search)
CREATE INDEX idx_product_attributes ON products USING GIN (attributes);
-- Supports: WHERE attributes @> '{"color": "red"}'

-- Expression index (index on a function result)
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
-- Supports: WHERE LOWER(email) = 'alice@example.com'
```

### When NOT to Add an Index

- Small tables (full scan is faster than index lookup + heap fetch)
- Columns with very low cardinality (e.g., `is_active BOOLEAN` — index on boolean rarely helps)
- Tables with very frequent writes (every insert/update/delete must update all indexes)
- Columns rarely used in WHERE, JOIN, or ORDER BY

---

## Interview Questions and Answers

### Normalization

**Q1: What is normalization and why do we do it?**

Normalization is the process of organizing a relational database schema to eliminate redundancy and ensure data integrity. We normalize to:
- Prevent update anomalies (changing one fact requires updating multiple rows)
- Prevent insertion anomalies (cannot insert data without unrelated data)
- Prevent deletion anomalies (deleting a record unintentionally deletes other data)
- Reduce storage waste

The trade-off: fully normalized schemas require more JOINs, which can hurt read performance. Most production schemas are normalized to 3NF with selective, documented denormalization for performance.

---

**Q2: What is the difference between 2NF and 3NF?**

Both address dependency problems, but they target different types:

- **2NF** removes **partial dependencies**: non-key columns must depend on the ENTIRE composite primary key, not just part of it. Only applies to tables with composite PKs.

- **3NF** removes **transitive dependencies**: non-key columns must not depend on other non-key columns. A column should depend directly on the PK, not via another non-key column.

Example: In a table `(employee_id, dept_id, dept_name)` — `dept_name` depends on `dept_id` (a non-key), not on `employee_id`. That is a transitive dependency, violating 3NF.

---

**Q3: When would you denormalize a database schema?**

Denormalize when:
1. You have profiled queries with EXPLAIN ANALYZE and confirmed that JOINs are the bottleneck
2. The data is read-heavy and write-infrequent
3. Reporting or analytics queries need to aggregate across many tables
4. You accept the overhead of keeping denormalized copies in sync (triggers, application code, or ETL)

Common scenarios: storing `order_total` on the orders table instead of summing `order_items`; storing `user_email` on an events table to avoid joining users; creating a materialized view for a dashboard.

Never denormalize preemptively. Measure first.

---

### Primary Keys

**Q4: What is the difference between a natural key and a surrogate key?**

A **natural key** is a real-world attribute that uniquely identifies a record: email address, ISBN, country ISO code. It has business meaning.

A **surrogate key** is a system-generated identifier with no business meaning: auto-increment integer, UUID.

Natural keys seem appealing but have pitfalls: people change emails, SSNs can be reassigned, names are not unique. Surrogate keys are stable, simple, and compact. Best practice: use a surrogate PK, then add a UNIQUE constraint on the natural key.

---

**Q5: Why is UUID v4 bad for clustered index inserts? What are the alternatives?**

UUID v4 is randomly generated across the 128-bit space. In a clustered index (like InnoDB's primary key index), new rows must be physically inserted in sorted order. A random UUID inserts at a random position in the B-tree:
- The target index page may not be in memory → disk read
- If the page is full → page split (the page is divided, half the data moves, parent pointers update)
- Over time → fragmentation → wasted space and poor sequential scan performance

**Alternatives:**
- **UUID v7**: time-ordered UUID. First 48 bits are millisecond timestamp, making it lexicographically sortable. Fixes the fragmentation problem while keeping UUID properties.
- **ULID**: 48-bit timestamp + 80-bit randomness. Lexicographically sortable, URL-safe.
- **Snowflake ID**: 64-bit integer, time-ordered. Twitter's solution. Sortable, 8 bytes instead of 16.
- **Auto-increment BIGINT**: still the best for single-node systems — sequential, 8 bytes, no fragmentation.

---

### Soft Delete

**Q6: What is the soft delete pattern? What are its challenges?**

Soft delete marks records as "deleted" without physically removing them, typically by adding a `deleted_at TIMESTAMPTZ` column. A record with `deleted_at IS NOT NULL` is considered deleted.

**Benefits**: data recovery, audit trail, preserving referential integrity for historical records.

**Challenges:**

1. **Every query must filter**: `WHERE deleted_at IS NULL` must appear on every active-data query. Easy to forget.

2. **Unique constraint conflicts**: If `email` must be unique among active users, a standard `UNIQUE(email)` will prevent a new user from registering an email that a deleted user once had. Fix: partial unique index `WHERE deleted_at IS NULL`.

3. **Performance**: Without a partial index, queries scanning active rows must skip deleted rows. Use `CREATE INDEX ... WHERE deleted_at IS NULL`.

4. **Storage growth**: Deleted records accumulate forever. Need periodic archival.

5. **GDPR conflicts**: "Right to erasure" requires actual deletion of PII. Soft delete must be paired with anonymization or a scheduled hard delete.

---

**Q7: How do you implement soft delete with JPA/Hibernate?**

Use `@SQLDelete` to override the DELETE SQL and `@SQLRestriction` to automatically filter queries:

```java
@Entity
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
@SQLRestriction("deleted_at IS NULL")
public class User {
    @Id Long id;
    String email;
    LocalDateTime deletedAt;
}
```

`@SQLDelete` intercepts `entityManager.remove(user)` and executes the UPDATE instead. `@SQLRestriction` appends `AND deleted_at IS NULL` to all JPQL and Criteria API queries for this entity. To bypass the restriction (e.g., admin view), use a native query.

---

### Multi-Tenancy

**Q8: What are the three multi-tenancy patterns and when do you use each?**

1. **Database-per-tenant**: Complete isolation, one database per customer. Use for regulated industries (healthcare, finance) with few large enterprise customers. High cost.

2. **Schema-per-tenant**: One database, separate schema per customer. Good isolation, shared infrastructure. Use for mid-scale SaaS with hundreds of tenants. PostgreSQL schemas are cheap; Flyway can migrate all schemas.

3. **Shared schema (row-level)**: One table, `tenant_id` column on every row. Lowest cost, simplest migration. Use for SaaS with thousands of tenants. Risk: application bugs can leak cross-tenant data. Mitigate with PostgreSQL Row Level Security (RLS).

---

**Q9: What is PostgreSQL Row Level Security (RLS) and how does it help with multi-tenancy?**

RLS is a database-level feature that automatically filters rows based on the current database session's context. Even if application code forgets to add `WHERE tenant_id = ?`, RLS enforces the filter at the storage engine level.

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant_id')::BIGINT);
```

The application sets `SET LOCAL app.current_tenant_id = '42'` at the start of each request. All subsequent queries on `orders` are silently filtered to `tenant_id = 42`. This is a defense-in-depth strategy — the application still adds `WHERE tenant_id = ?` for clarity and index usage, but RLS is the safety net.

---

### Pagination

**Q10: What is keyset pagination and why is it better than offset pagination at scale?**

**Offset pagination** skips a fixed number of rows. `OFFSET 10000` forces the database to read and discard 10,000 rows, even though you only return 20. Cost is O(offset). Also suffers from page drift: new inserts between requests cause items to shift.

**Keyset pagination** uses the last item's sort key values as a cursor:
```sql
WHERE (created_at, id) < (:lastCreatedAt, :lastId)
ORDER BY created_at DESC, id DESC LIMIT 20;
```
With a composite index on `(created_at DESC, id DESC)`, this is an O(log n) B-tree seek directly to the cursor position — consistent performance regardless of how deep the page is.

The downside: no random access (cannot jump to page 500 directly), and you need a stable, unique sort key (usually `(timestamp, id)`).

---

### Hierarchical Data

**Q11: What are the different ways to store hierarchical data in SQL?**

**Adjacency list**: Parent ID stored on each node. Simple, supports fast inserts/moves. Retrieving a full subtree requires a recursive CTE. Best for write-heavy trees or moderate depth.

**Nested set**: Each node has `lft`/`rgt` values. Subtrees are range queries. Extremely fast reads, but inserts and moves update O(n) rows. Best for mostly-static trees.

**Closure table**: Separate table of all ancestor-descendant pairs with depth. Fast in all query directions. Insert requires O(depth) new rows. Move requires rebuilding paths. Best for read-heavy trees with complex traversal needs.

---

**Q12: What is the difference between an adjacency list and a closure table for hierarchical data?**

**Adjacency list**: Each row stores only its direct parent (`parent_id`). Simple to understand. To find all descendants, you need a recursive CTE or application-level loop. Can become slow for deep trees with many descendants.

**Closure table**: A separate table stores every ancestor-descendant pair (not just parent-child). Row `(1, 4, 3)` means "node 1 is an ancestor of node 4 at depth 3." Getting all descendants is a simple `WHERE ancestor_id = X` query. Getting all ancestors is a simple `WHERE descendant_id = X` query. No recursion needed.

The closure table trades storage (more rows) for query simplicity and speed.

---

### JPA Inheritance

**Q13: What is the difference between Single Table, Joined, and Table Per Class JPA inheritance strategies?**

- **SINGLE_TABLE**: All subtypes in one table, discriminator column identifies type. Fastest reads (no JOIN), but allows NULL columns and cannot enforce NOT NULL on subtype-specific columns.

- **JOINED**: Base columns in base table, subtype columns in subtype tables. JPA JOINs them on query. Fully normalized, no NULL columns, subtype constraints work. Slower for polymorphic queries.

- **TABLE_PER_CLASS**: Each concrete class has its own complete table (all columns duplicated). No JOINs per subtype query. Polymorphic queries require UNION ALL across all subtype tables. Cannot use IDENTITY auto-increment. Rarely recommended.

**Rule of thumb**: Use SINGLE_TABLE for simple hierarchies. Use JOINED when subtype integrity constraints are important.

---

**Q14: How do you design a schema for a many-to-many relationship with additional attributes?**

A pure M:N requires a junction table. When that junction has its own attributes (like `enrolled_at`, `grade`), the junction becomes a full entity:

```sql
CREATE TABLE enrollments (
    student_id BIGINT NOT NULL REFERENCES students(id),
    course_id BIGINT NOT NULL REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ DEFAULT NOW(),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
```

In JPA, model the junction as an `@Entity` with an `@EmbeddedId` (composite key). Avoid `@ManyToMany` with `@JoinTable` when the junction has extra columns — JPA cannot map those attributes with a plain `@ManyToMany`.

---

**Q15: What is a partial unique index and why is it important for soft delete?**

A partial index is a regular B-tree index with a `WHERE` clause that limits which rows are indexed. A partial unique index enforces uniqueness only among the rows matching the `WHERE` condition.

For soft delete, a standard `UNIQUE(email)` fails because it enforces uniqueness across all rows, including soft-deleted ones. If Alice deletes her account and re-registers with the same email, the insert fails.

The fix:
```sql
CREATE UNIQUE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;
```
This enforces "no two active users share an email" while allowing multiple deleted records to have the same email. It is also smaller and faster than a full index because deleted rows are excluded.

---

**Q16: How do you handle schema migrations in CI/CD with Flyway or Liquibase?**

The migration file is committed alongside the code change in the same PR. The CI/CD pipeline spins up a test database, runs `flyway migrate` (or `liquibase update`) before running tests, verifying the migration works.

**Key rules:**

1. Never modify an applied migration file — Flyway will detect the checksum mismatch and refuse to start.
2. For zero-downtime deployments, migrations must be backward-compatible with the current running code version. This means additive-first: add a column, deploy new code, then remove the old column in a later migration.
3. Avoid adding NOT NULL columns without DEFAULT values to large tables — the ALTER locks the table in some databases.
4. Use repeatable migrations (`R__`) in Flyway for views and stored procedures that are re-run when they change.

---

**Q17: What are temporal tables and why would you use them?**

Temporal tables track the history of data changes over time, allowing queries like "what was the price of this product on January 1st?"

**Valid time**: when the fact was true in the real world.
**Transaction time**: when the record was stored in the database.

A bitemporal table tracks both. For most use cases, valid time alone suffices — a "from/to" date range on each record, with the open-ended current record using `'9999-12-31'` as the `valid_to` sentinel.

```sql
SELECT price FROM product_prices
WHERE product_id = 1
  AND valid_from <= '2023-01-01'
  AND valid_to > '2023-01-01';
```

Use temporal tables for: product pricing history, employee salary history, insurance policy terms, any domain where "what was the state at time T?" is a business requirement.

---

**Q18: What is the N+1 query problem in JPA and how does schema design affect it?**

The N+1 problem occurs when loading N entities triggers N additional queries to load their associations.

```java
// Fetches 100 orders (1 query)
List<Order> orders = orderRepository.findAll();
// Then for each order, fetches user (100 queries!) = N+1
orders.forEach(o -> System.out.println(o.getUser().getName()));
```

**Schema design impact**: The relationship between `Order` and `User` is a FK (`user_id` on `orders`). The schema is fine — the problem is in how JPA fetches it.

**Solutions:**
1. `fetch = FetchType.LAZY` (default for `@ManyToOne` should be set explicitly) — defers loading, but still triggers N queries when accessed
2. `JOIN FETCH` in JPQL: `SELECT o FROM Order o JOIN FETCH o.user`
3. `@EntityGraph` to specify which associations to eagerly fetch per query
4. Batch fetching: `@BatchSize(size = 50)` — Hibernate fetches 50 users per query instead of 1

```java
// JOIN FETCH solution
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") String status);
```

---

**Q19: How would you design a schema for a notification system that supports multiple notification types (email, SMS, push)?**

```sql
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    type VARCHAR(20) NOT NULL CHECK (type IN ('EMAIL', 'SMS', 'PUSH')),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    subject VARCHAR(255),        -- email only
    body TEXT NOT NULL,
    recipient VARCHAR(255) NOT NULL,  -- email address, phone, or device token
    sent_at TIMESTAMPTZ,
    failed_at TIMESTAMPTZ,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    metadata JSONB              -- type-specific data (push payload, email headers, etc.)
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_status ON notifications(status) WHERE status = 'PENDING';
```

The `metadata JSONB` column handles type-specific fields without requiring multiple nullable columns or a separate subtype table — appropriate when subtypes are simple and not heavily queried by their specific fields.

---

**Q20: What is the difference between ON DELETE CASCADE and ON DELETE SET NULL in foreign key constraints?**

`ON DELETE CASCADE`: When the parent row is deleted, all child rows that reference it are automatically deleted. Use when child rows have no meaning without the parent (e.g., `order_items` when an `order` is deleted).

`ON DELETE SET NULL`: When the parent row is deleted, the FK column in child rows is set to NULL. Use when child rows can exist independently but the relationship becomes optional (e.g., when an employee's manager leaves, set `manager_id = NULL` rather than deleting the employee).

`ON DELETE RESTRICT` (default): Prevents deleting the parent if any child rows reference it. Forces explicit cleanup before deletion. Use when you want to prevent accidental data loss.

```sql
-- Cascade: deleting order deletes its items
CREATE TABLE order_items (
    order_id BIGINT REFERENCES orders(id) ON DELETE CASCADE
);

-- Set null: deleting manager doesn't delete reports
CREATE TABLE employees (
    manager_id BIGINT REFERENCES employees(id) ON DELETE SET NULL
);

-- Restrict (default): cannot delete department while it has employees
CREATE TABLE employees (
    department_id BIGINT REFERENCES departments(id) ON DELETE RESTRICT
);
```

---

**Q21: How do you ensure uniqueness for soft-deleted records with a composite unique constraint?**

A standard unique index `UNIQUE(email)` or `UNIQUE(email, deleted_at)` does not work reliably for soft delete because NULL values have special behavior in unique indexes (most databases treat each NULL as distinct).

The correct approach: a **partial unique index** that only covers active (non-deleted) rows.

```sql
-- Enforces uniqueness only among non-deleted records
CREATE UNIQUE INDEX idx_users_email_active ON users(email)
WHERE deleted_at IS NULL;

-- For composite uniqueness (e.g., unique username per tenant, among active records)
CREATE UNIQUE INDEX idx_users_tenant_username_active
ON users(tenant_id, username)
WHERE deleted_at IS NULL;
```

This allows:
- Two active users: `alice@example.com` → index blocks it (UNIQUE violation)
- One active, one deleted: `alice@example.com` → allowed (deleted row not in partial index)
- Two deleted users with same email → allowed (neither is in the partial index)

---

**Q22: What is an EXCLUDE constraint in PostgreSQL and when would you use it?**

`EXCLUDE` is a generalization of UNIQUE. While UNIQUE says "no two rows may have the same value," EXCLUDE says "no two rows may satisfy this comparison using this operator."

The most common use is preventing overlapping date/time ranges using GiST index operators.

```sql
-- Ensure no two bookings for the same room overlap in time
CREATE TABLE room_bookings (
    id BIGSERIAL PRIMARY KEY,
    room_id BIGINT NOT NULL,
    booked_during TSTZRANGE NOT NULL,  -- time range type

    EXCLUDE USING gist (
        room_id WITH =,                         -- same room
        booked_during WITH &&                   -- overlapping ranges (&&)
    )
);

-- Insert succeeds: different time ranges
INSERT INTO room_bookings VALUES (DEFAULT, 1, '[2024-01-01 09:00, 2024-01-01 10:00)');
INSERT INTO room_bookings VALUES (DEFAULT, 1, '[2024-01-01 11:00, 2024-01-01 12:00)');

-- Insert fails: overlapping range for same room
INSERT INTO room_bookings VALUES (DEFAULT, 1, '[2024-01-01 09:30, 2024-01-01 10:30)');
-- ERROR: conflicting key value violates exclusion constraint
```

This technique is also used in temporal tables to prevent overlapping price validity periods.

---

**Q23: What are the advantages and disadvantages of using JSONB columns in PostgreSQL for schema flexibility?**

**Advantages:**
- Flexible schema for varying or unknown attributes (product attributes, event metadata, configuration)
- Native JSON operators: `->`, `->>`, `@>`, `?`
- Can index with GIN for fast containment queries
- Stored as binary (faster to query than JSON text)

**Disadvantages:**
- No referential integrity within JSONB (cannot add FKs to JSON fields)
- Cannot enforce data types or NOT NULL constraints on JSON fields
- Schema validation must be done at the application layer
- Harder to query and join compared to normalized columns
- JSONB updates require rewriting the entire document

**Good use of JSONB:**
```sql
-- Event payload (schema varies per event type)
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(50) NOT NULL,
    user_id BIGINT REFERENCES users(id),
    occurred_at TIMESTAMPTZ DEFAULT NOW(),
    payload JSONB NOT NULL
);

CREATE INDEX idx_events_payload_gin ON events USING GIN(payload);

-- Query events where payload contains a specific field
SELECT * FROM events
WHERE type = 'ORDER_PLACED'
  AND payload @> '{"currency": "USD"}';
```

**Rule**: Use JSONB for genuinely schemaless data (metadata, event payloads, third-party API responses). Use typed columns for anything you JOIN on, filter frequently, or need constraints on.

---

**Q24: How would you model a product catalog where products can have different attributes depending on their category (e.g., books have ISBN, electronics have wattage)?**

This is the "Entity-Attribute-Value" (EAV) problem. Several approaches:

**Option 1: JSONB attributes column (recommended for moderate flexibility)**
```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    category_id BIGINT REFERENCES categories(id),
    attributes JSONB NOT NULL DEFAULT '{}'
);
-- Electronics: attributes = {"wattage": 65, "voltage": "220V", "warranty_years": 2}
-- Books: attributes = {"isbn": "978-3-16-148410-0", "author": "...", "pages": 350}
```

**Option 2: EAV table (maximum flexibility, poor query performance)**
```sql
CREATE TABLE product_attributes (
    product_id BIGINT REFERENCES products(id),
    attribute_name VARCHAR(100) NOT NULL,
    attribute_value TEXT,
    PRIMARY KEY (product_id, attribute_name)
);
```
EAV is an anti-pattern for most cases — queries are complex, type safety is lost, aggregations are painful.

**Option 3: Joined table inheritance per category**
```sql
CREATE TABLE products (id BIGSERIAL PRIMARY KEY, name VARCHAR(255));
CREATE TABLE books (
    id BIGINT PRIMARY KEY REFERENCES products(id),
    isbn CHAR(13) UNIQUE NOT NULL,
    author VARCHAR(255),
    pages INT
);
CREATE TABLE electronics (
    id BIGINT PRIMARY KEY REFERENCES products(id),
    wattage INT,
    voltage VARCHAR(20)
);
```
Best type safety and constraints. Requires schema change to add a new category type.

**Recommendation**: JSONB for catalogs with many dynamic category types; Joined inheritance for a small number of well-defined category types.

---

**Q25: What is the difference between TRUNCATE and DELETE in SQL? When would you use each?**

`DELETE FROM table WHERE condition` removes specific rows. It fires row-level triggers, is transactional (can be rolled back), and updates sequence counters slowly row by row. Logged fully in the WAL (write-ahead log) for each row.

`TRUNCATE TABLE table` removes all rows instantaneously by deallocating the table's data pages. Does NOT fire row-level triggers. Very fast (O(1) regardless of row count). In PostgreSQL, TRUNCATE is transactional (can be rolled back). Resets auto-increment sequences.

```sql
-- Delete specific rows (transactional, triggers fire)
DELETE FROM sessions WHERE expires_at < NOW();

-- Remove all rows instantly (full table wipe, no triggers)
TRUNCATE TABLE audit_logs;

-- TRUNCATE with restart sequence
TRUNCATE TABLE users RESTART IDENTITY;  -- resets the auto-increment sequence
```

**Use cases:**
- `DELETE`: targeted row removal, when you need WHERE conditions or triggers
- `TRUNCATE`: clearing test data, resetting a staging environment, emptying a staging table before a bulk load

---

**Q26: How do you handle database connection pooling in a Spring Boot application?**

Spring Boot auto-configures HikariCP (the fastest JDBC connection pool) when it is on the classpath (default with `spring-boot-starter-data-jpa`).

```properties
# HikariCP configuration
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000    # ms: max wait for connection
spring.datasource.hikari.idle-timeout=600000          # ms: close idle connections after 10 min
spring.datasource.hikari.max-lifetime=1800000          # ms: close connections after 30 min
spring.datasource.hikari.keepalive-time=60000          # ms: send keepalive queries
```

**Pool size formula (Hikari recommendation):**
```
pool_size = (core_count * 2) + effective_spindle_count
```
For a 4-core server with SSD: `(4 * 2) + 1 = 9` connections. Counter-intuitively, large pools often perform worse due to lock contention. Start with 10-20, tune based on monitoring.

**Key considerations:**
- Each connection to PostgreSQL spawns a backend process — too many connections kills DB server
- Use PgBouncer (connection pooler at DB level) for very high connection counts
- For multi-tenant schema-switching, be careful: the schema setting is connection-level state, requiring careful pool management

---

**Q27: What is a covering index and how does it improve query performance?**

A covering index includes all columns referenced in a query (in SELECT, WHERE, ORDER BY) so the query can be answered entirely from the index without accessing the actual table rows (heap).

```sql
-- Query: find orders for a user and return status + total
SELECT status, total FROM orders WHERE user_id = 42;

-- Regular index: finds row IDs (user_id=42), then fetches each row from heap
CREATE INDEX idx_orders_user ON orders(user_id);

-- Covering index: entire query answered from index, no heap access
CREATE INDEX idx_orders_user_covering ON orders(user_id)
INCLUDE (status, total);  -- PostgreSQL INCLUDE syntax

-- MySQL equivalent:
CREATE INDEX idx_orders_user_covering ON orders(user_id, status, total);
```

PostgreSQL's `INCLUDE` clause adds non-key columns to the index leaf pages without making them part of the sort key. This avoids index bloat from non-selective columns while still covering the query.

Use EXPLAIN to verify index-only scan (look for "Index Only Scan" in PostgreSQL EXPLAIN output).

---

**Q28: How would you design the schema for a messaging system (like a chat application)?**

```sql
CREATE TABLE conversations (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(20) NOT NULL DEFAULT 'DIRECT',  -- DIRECT, GROUP
    name VARCHAR(100),  -- null for direct messages
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE conversation_participants (
    conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    last_read_at TIMESTAMPTZ,  -- for unread count calculation
    PRIMARY KEY (conversation_id, user_id)
);

CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    sender_id BIGINT NOT NULL REFERENCES users(id),
    content TEXT NOT NULL,
    sent_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    edited_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ  -- soft delete
);

-- Efficient queries: latest message per conversation, messages for a conversation
CREATE INDEX idx_messages_conversation_sent ON messages(conversation_id, sent_at DESC);

-- Unread count: messages sent after user's last_read_at
-- SELECT COUNT(*) FROM messages m
-- JOIN conversation_participants cp ON m.conversation_id = cp.conversation_id
-- WHERE cp.user_id = :userId
--   AND m.sent_at > cp.last_read_at
--   AND m.sender_id != :userId
```

---

**Q29: What is optimistic locking and how do you implement it in JPA?**

Optimistic locking handles concurrent updates by assuming conflicts are rare. Instead of locking the row on read, it checks that the row hasn't changed before applying the update. If the row changed (another transaction updated it first), the update fails with an optimistic lock exception.

```sql
-- Add version column to table
ALTER TABLE orders ADD COLUMN version INT NOT NULL DEFAULT 0;

-- Optimistic update: only succeeds if version matches
UPDATE orders
SET status = 'SHIPPED', version = version + 1
WHERE id = 42 AND version = 3;
-- If 0 rows updated, another transaction already changed the row
```

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version  // Hibernate manages version automatically
    private Integer version;

    private String status;
}

// When save() is called, Hibernate generates:
// UPDATE orders SET status=?, version=? WHERE id=? AND version=?
// If 0 rows updated → throws OptimisticLockException

// Handle in service layer
try {
    orderRepository.save(order);
} catch (OptimisticLockException e) {
    // Retry or return conflict response (HTTP 409)
}
```

**Pessimistic locking** (alternative): Locks the row on read with `SELECT ... FOR UPDATE`. Prevents concurrent reads, simpler to reason about but reduces throughput. Use for critical financial operations where conflicts are expected.

```java
// JPA pessimistic lock
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Order> findById(Long id);
```

---

**Q30: How do you design a schema for a feature flag / A/B testing system?**

```sql
CREATE TABLE feature_flags (
    id BIGSERIAL PRIMARY KEY,
    key VARCHAR(100) UNIQUE NOT NULL,  -- 'new_checkout_flow', 'dark_mode'
    description TEXT,
    is_enabled BOOLEAN NOT NULL DEFAULT false,
    rollout_percentage INT NOT NULL DEFAULT 0 CHECK (rollout_percentage BETWEEN 0 AND 100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User-level overrides (override global flag for specific users)
CREATE TABLE feature_flag_overrides (
    flag_id BIGINT NOT NULL REFERENCES feature_flags(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    is_enabled BOOLEAN NOT NULL,
    PRIMARY KEY (flag_id, user_id)
);

-- Segment targeting (enable for users in a segment)
CREATE TABLE user_segments (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL  -- 'beta_testers', 'premium_users'
);

CREATE TABLE feature_flag_segments (
    flag_id BIGINT REFERENCES feature_flags(id),
    segment_id BIGINT REFERENCES user_segments(id),
    PRIMARY KEY (flag_id, segment_id)
);
```

**Evaluation logic (in application)**:
1. Check user-level override first (highest priority)
2. Check if user is in a targeted segment
3. Apply rollout percentage (hash user_id % 100 < rollout_percentage)
4. Fall back to global `is_enabled`

---

**Q31: What is connection-level state in PostgreSQL and why does it matter for multi-tenancy?**

PostgreSQL sessions maintain state at the connection level: `search_path`, `SET` variables (`app.current_tenant_id`), `SET LOCAL` variables (transaction-scoped), and other configuration parameters.

In schema-per-tenant multi-tenancy, you `SET search_path TO tenant_abc` to route queries to the correct schema. In shared schema with RLS, you `SET LOCAL app.current_tenant_id = '42'` before queries.

**The danger with connection pooling**: HikariCP reuses connections. If connection A was used for tenant 42 (with `search_path = tenant_42` set) and is returned to the pool, the next request using that connection might accidentally query the wrong schema.

**Solutions:**
1. Use `SET LOCAL` (transaction-scoped) instead of `SET` — automatically reverts after the transaction
2. Use a connection pool wrapper that resets connection state after returning to pool
3. Use Hibernate's `MultiTenantConnectionProvider` which sets the schema on every connection acquisition
4. Use `DISCARD ALL` or `RESET ALL` when returning connections to pool (expensive, but safe)

---

**Q32: How do you handle schema evolution when you need to rename a column without downtime?**

Renaming a column directly (`ALTER TABLE orders RENAME COLUMN old_name TO new_name`) causes an outage because deployed application code still references `old_name`.

**Zero-downtime rename pattern (3-phase deployment):**

```
Phase 1 — Add new column, copy data, dual-write:
  - Migration: ALTER TABLE orders ADD COLUMN new_col_name VARCHAR(255);
  - Migration: UPDATE orders SET new_col_name = old_col_name;
  - Migration: CREATE TRIGGER ... (sync writes to both columns)
  - Deploy app version that writes to BOTH columns, reads from old

Phase 2 — Switch reads to new column:
  - Deploy app version that reads from new_col_name, writes to both

Phase 3 — Remove old column:
  - Migration: ALTER TABLE orders DROP COLUMN old_col_name;
  - Deploy app version that only writes to new_col_name
```

This pattern ensures no version of the application ever reads a column that doesn't exist.

---

**Q33: What is a database view and when should you use one?**

A view is a named, stored query. Querying a view is equivalent to running its underlying query. Views do not store data (unlike materialized views).

```sql
-- View: encapsulate complex join for reporting
CREATE VIEW order_summaries AS
SELECT
    o.id,
    o.created_at,
    u.email AS user_email,
    COUNT(oi.id) AS item_count,
    SUM(oi.unit_price * oi.quantity) AS subtotal
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, o.created_at, u.email;

-- Query the view
SELECT * FROM order_summaries WHERE created_at > '2024-01-01';
```

**Materialized view**: Caches the query result. Must be manually refreshed. Use for expensive aggregations that power dashboards or reports.

```sql
-- Materialized view for daily sales report
CREATE MATERIALIZED VIEW daily_sales AS
SELECT
    DATE(created_at) AS sale_date,
    COUNT(*) AS order_count,
    SUM(total) AS revenue
FROM orders
WHERE status = 'DELIVERED'
GROUP BY DATE(created_at);

-- Refresh manually or on a schedule
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
-- CONCURRENTLY: allows reads during refresh (requires unique index on the view)
```

**Use views for**: encapsulating complex joins, providing read-only access to specific columns (security), presenting consistent API despite schema changes.

---

**Q34: Explain the CAP theorem and its relevance to distributed database design.**

The CAP theorem states that in a distributed system, you can guarantee at most 2 of 3 properties simultaneously:

- **Consistency (C)**: Every read receives the most recent write (or an error). All nodes see the same data at the same time.
- **Availability (A)**: Every request receives a response (not an error), even if the data might be stale.
- **Partition tolerance (P)**: The system continues operating even if network partitions (node communication failures) occur.

Since network partitions are unavoidable in distributed systems, real-world systems choose between **CP** (consistent, may be unavailable during partition) and **AP** (always available, may return stale data during partition).

**Practical relevance for Java full-stack developers:**

- **PostgreSQL (single node)**: CP — consistent, but not distributed natively
- **PostgreSQL (read replicas)**: AP for reads — replicas may lag, but are always available
- **Redis**: CP (single node) or AP (Redis Cluster)
- **Cassandra**: AP — highly available, eventually consistent
- **DynamoDB**: configurable (strongly consistent reads = CP, eventually consistent = AP)

For schema design: eventual consistency requires careful handling of distributed writes — consider idempotency keys, distributed transactions (Saga pattern), or conflict-free replicated data types (CRDTs) depending on the consistency requirements.

---

**Q35: How do you choose between a relational database and a NoSQL database for a new project?**

**Choose a relational database (PostgreSQL, MySQL) when:**
- Data has complex relationships and JOINs are necessary
- ACID transactions are required (financial, inventory)
- Schema is well-defined and relatively stable
- Complex queries with aggregations and reporting
- Consistency is more important than write throughput

**Choose NoSQL when:**
- Document model: data is hierarchical and self-contained (MongoDB for product catalogs with varying attributes)
- Key-value: ultra-high-speed simple lookups (Redis for sessions, caching, rate limiting)
- Time-series: append-only write-heavy with time-based queries (InfluxDB, TimescaleDB for metrics)
- Wide-column: massive scale with flexible columns (Cassandra for write-heavy event storage)
- Schema is evolving rapidly or varies significantly per record

**The pragmatic approach for most full-stack Java applications**: Start with PostgreSQL. It supports JSONB for flexible schemas, has excellent full-text search, handles time-series with TimescaleDB extension, and scales well. Add Redis for caching/sessions alongside it. Only introduce a specialized NoSQL database when you have a specific, measured bottleneck that PostgreSQL cannot address.

---

## Quick Reference Summary

### Normalization Levels

| Form | Violation | Fix |
|------|-----------|-----|
| 1NF | Non-atomic values, repeating groups | Separate table for multi-valued attributes |
| 2NF | Partial dependency on composite PK | Move partially-dependent columns to own table |
| 3NF | Transitive dependency (A→B→C) | Extract the chain into a separate table |
| BCNF | Determinant is not a candidate key | Decompose the table |

### PK Strategy Decision Tree

```
Single-node system, sequential inserts needed? → BIGSERIAL (auto-increment)
Distributed system, no DB dependency needed?  → UUID v7 or ULID (time-ordered)
Security: IDs must not be guessable?          → UUID v4 (random) or UUID v7
High write throughput, sortable?              → Snowflake ID
```

### Soft Delete Checklist

- [ ] Add `deleted_at TIMESTAMPTZ` column (nullable)
- [ ] Create partial index: `WHERE deleted_at IS NULL`
- [ ] Create partial unique indexes for all UNIQUE constraints: `WHERE deleted_at IS NULL`
- [ ] Add `@SQLDelete` + `@SQLRestriction` to JPA entities
- [ ] Plan for GDPR: anonymize PII on soft delete or schedule hard delete
- [ ] Implement archival for old soft-deleted records

### Hierarchy Model Selection

| Need | Use |
|------|-----|
| Simple, write-heavy, moderate depth | Adjacency list + recursive CTE |
| Static tree, read-heavy, fast subtree reads | Nested set |
| Complex traversal, both directions | Closure table |
| PostgreSQL only, ordered children | `ltree` extension |

### Pagination Selection

| Use Case | Choose |
|----------|--------|
| Small table (< 100k), admin panel, random page access | Offset |
| Large table, feed/timeline, real-time data | Keyset (cursor) |
| Infinite scroll | Keyset |
| "Show page 5 of 50" UI | Offset |
