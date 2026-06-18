# Database Concepts Interview Questions & Answers

## Overview

Covers the core database concepts a junior developer needs: database types, ACID, normalization, SQL commands, joins, indexing, transactions, and security.

---

## Table of Contents

1. [Database Fundamentals](#database-fundamentals)
2. [ACID Properties](#acid-properties)
3. [Normalization](#normalization)
4. [SQL Fundamentals](#sql-fundamentals)
5. [Advanced SQL Concepts](#advanced-sql-concepts)
6. [Database Design](#database-design)
7. [Transaction Management](#transaction-management)
8. [Concurrency Control](#concurrency-control)
9. [Database Indexing](#database-indexing)
10. [Query Optimization](#query-optimization)
11. [Database Security](#database-security)
12. [NoSQL vs SQL](#nosql-vs-sql)
13. [Common Database Mistakes](#common-database-mistakes)
14. [Short Revision Summary](#short-revision-summary)

---

## Database Fundamentals

### Q1: What is a database and what are the different types?

**Answer:** A database is an organized collection of structured information stored electronically. The main types are:

**1. Relational (RDBMS):** Tables with rows/columns, SQL querying, enforced schema. Examples: MySQL, PostgreSQL, Oracle.

```sql
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(100), email VARCHAR(100));
CREATE TABLE orders (id INT PRIMARY KEY, user_id INT, total DECIMAL(10,2), FOREIGN KEY (user_id) REFERENCES users(id));
```

**2. NoSQL:** Flexible schema, various data models.
- Document: MongoDB, CouchDB
- Key-value: Redis, DynamoDB
- Column-family: Cassandra
- Graph: Neo4j

**3. Others:** NewSQL (SQL + horizontal scale, e.g. CockroachDB), In-Memory (Redis, Memcached).

---

### Q2: What is a DBMS and what are its key components?

**Answer:** A DBMS (Database Management System) is software that lets you define, create, and manage a database. Key components: Query Processor (parses, optimizes, executes SQL), Storage Manager (buffers data between disk and memory, handles transactions and recovery), and Data Files (stores pages/blocks of actual data and indexes).

---

### Q3: What are the levels of data independence?

**Answer:** The ability to change one schema layer without affecting the layer above it.

- **Physical independence:** Change storage (e.g., add an index) without altering the logical schema.
- **Logical independence:** Change the logical schema (e.g., split a table) without altering user views.

The three schema levels are Physical (how data is stored), Logical (what data and relationships exist), and View (what each user sees).

---

## ACID Properties

### Q4: What are ACID properties?

**Answer:** ACID guarantees reliable transaction processing.

**A - Atomicity:** All operations succeed or none do — no partial updates.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- If either fails, both are rolled back
```

**C - Consistency:** The database moves from one valid state to another; all constraints (foreign keys, checks, unique) are enforced.

**I - Isolation:** Concurrent transactions don't interfere. Implemented via locking and MVCC.

**D - Durability:** Committed changes survive crashes. Implemented via write-ahead logging.

---

## Normalization

### Q5: What is normalization and what are the normal forms?

**Answer:** Normalization organizes data to eliminate redundancy and prevent update/insert/delete anomalies.

**1NF:** Each cell holds one value; every record is unique.

```sql
-- ❌ products VARCHAR(500) stores "Laptop, Mouse" in one cell
-- ✅ Use a separate order_items table with one product per row
CREATE TABLE order_items (item_id INT PRIMARY KEY, order_id INT, product_name VARCHAR(100));
```

**2NF:** In 1NF + no partial dependencies (all non-key columns depend on the whole primary key).

```sql
-- ❌ product_name depends only on product_id in a (order_id, product_id) key table
-- ✅ Move product_name to a separate products table
```

**3NF:** In 2NF + no transitive dependencies (non-key columns must not depend on other non-key columns).

```sql
-- ❌ customer_name stored in orders table (depends on customer_id, not order_id)
-- ✅ Move customer_name to a customers table; store only customer_id in orders
```

**BCNF:** Stronger 3NF — every determinant must be a candidate key. **4NF/5NF:** Handle multi-valued and join dependencies (rarely needed for junior interviews).

**Denormalization:** Intentionally add redundancy for read performance (common in reporting/data warehouses). Trade-off: faster reads, risk of update anomalies.

---

## SQL Fundamentals

### Q6: What are the types of SQL commands?

**Answer:**

| Category | Commands | Purpose |
|----------|----------|---------|
| **DDL** | CREATE, ALTER, DROP, TRUNCATE | Define schema |
| **DML** | SELECT, INSERT, UPDATE, DELETE | Manipulate data |
| **DCL** | GRANT, REVOKE | Control access |
| **TCL** | COMMIT, ROLLBACK, SAVEPOINT | Manage transactions |

```sql
-- DDL
ALTER TABLE users ADD COLUMN email VARCHAR(100);
TRUNCATE TABLE users;  -- Removes all rows, resets auto-increment

-- DML
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');
UPDATE users SET email = 'new@example.com' WHERE id = 1;
DELETE FROM users WHERE id = 1;

-- DCL
GRANT SELECT, INSERT ON users TO 'app_user'@'localhost';
REVOKE INSERT ON users FROM 'app_user'@'localhost';

-- TCL
BEGIN TRANSACTION;
SAVEPOINT before_delete;
DELETE FROM temp_table;
ROLLBACK TO before_delete;  -- Undo only to savepoint
COMMIT;
```

---

### Q7: What are the types of SQL joins?

**Answer:** Joins combine rows from two or more tables on a related column.

| Join Type | Returns |
|-----------|---------|
| INNER JOIN | Rows matching in both tables |
| LEFT JOIN | All from left + matching from right (NULL if no match) |
| RIGHT JOIN | All from right + matching from left (NULL if no match) |
| FULL OUTER JOIN | All rows from both tables |
| CROSS JOIN | Every combination (Cartesian product) |
| SELF JOIN | Table joined to itself (e.g., employee → manager) |

```sql
-- INNER JOIN: users who have placed orders
SELECT u.name, o.order_date FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all users, even those without orders
SELECT u.name, o.order_date FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- SELF JOIN: employee hierarchy
SELECT e1.name AS employee, e2.name AS manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;
```

---

## Advanced SQL Concepts

### Q8: What are aggregate functions and GROUP BY?

**Answer:** Aggregate functions compute a result over a set of rows. Use `GROUP BY` to aggregate per group and `HAVING` to filter groups.

```sql
SELECT
    user_id,
    COUNT(*)        AS order_count,
    SUM(total)      AS total_spent,
    AVG(total)      AS avg_order_value
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;
```

Common functions: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. `COUNT(DISTINCT col)` counts unique values. `COUNT(email)` ignores NULLs.

---

### Q9: What are subqueries and correlated subqueries?

**Answer:** A subquery is a query nested inside another query.

```sql
-- Scalar: find products priced above average
SELECT name, price FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- IN: users who placed an order
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- Derived table (subquery in FROM)
SELECT * FROM (
    SELECT u.username, COUNT(o.id) AS order_count
    FROM users u LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.username
) AS summary
WHERE order_count > 0;

-- Correlated (references outer query — runs per outer row, can be slow)
SELECT u.username FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 1000
);
```

Prefer a `JOIN` over a correlated subquery when possible for better performance.

---

## Database Design

### Q10: What are the types of relationships in database design?

**Answer:**

**One-to-One (1:1):** Each row in Table A relates to one row in Table B. Use when separating optional or large data (e.g., users ↔ user_profiles).

**One-to-Many (1:N):** One row in A relates to many in B. The most common type — foreign key lives in the "many" table (e.g., users ↔ orders).

**Many-to-Many (N:M):** Use a junction table.

```sql
CREATE TABLE student_courses (
    student_id INT,
    course_id  INT,
    enrollment_date DATE,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id)  REFERENCES courses(id)
);
```

**Self-Referencing:** A table references itself (e.g., employees with manager_id pointing back to employees).

---

## Transaction Management

### Q11: What are transaction isolation levels?

**Answer:** Isolation levels control what a transaction can see from other concurrent transactions.

| Level | Dirty Reads | Non-repeatable Reads | Phantom Reads | Performance |
|-------|-------------|---------------------|---------------|-------------|
| Read Uncommitted | Possible | Possible | Possible | Best |
| Read Committed | Prevented | Possible | Possible | Good |
| Repeatable Read | Prevented | Prevented | Possible | Fair |
| Serializable | Prevented | Prevented | Prevented | Worst |

- **Dirty read:** reading uncommitted data from another transaction.
- **Non-repeatable read:** same row returns different values within one transaction.
- **Phantom read:** a range query returns different rows within one transaction.

Most databases default to **Read Committed** or **Repeatable Read**.

---

## Concurrency Control

### Q12: What are database locks and what is a deadlock?

**Answer:** Locks prevent conflicting concurrent access to data.

- **Shared lock (S):** Multiple transactions can read; no writes allowed. Acquired by `SELECT ... FOR SHARE`.
- **Exclusive lock (X):** Only one transaction can write; blocks all others. Acquired automatically by `UPDATE`/`DELETE`, or explicitly with `SELECT ... FOR UPDATE`.

**Deadlock:** Two transactions each hold a lock the other needs, so both wait forever.

```sql
-- T1 locks account 1, waits for account 2
-- T2 locks account 2, waits for account 1 → DEADLOCK
```

The database detects and rolls back one transaction automatically. **Prevention:** always lock rows in the same order, keep transactions short, use proper indexes.

---

## Database Indexing

### Q13: What are the types of database indexes?

**Answer:** Indexes speed up data retrieval at the cost of extra storage and slower writes.

**B-Tree (default):** Balanced tree; good for equality, range, and prefix queries. Inefficient for leading wildcards (`LIKE '%x%'`) or functions on the column.

```sql
CREATE INDEX idx_users_email ON users(email);
```

**Composite index:** Covers multiple columns — the leftmost-prefix rule applies.

```sql
CREATE INDEX idx_orders ON orders(user_id, order_date, status);
-- Efficient for queries filtering by user_id, or user_id + order_date, or all three
-- NOT efficient if leading column (user_id) is skipped
```

**Clustered index:** Data rows are physically stored in index order. Only one per table (usually the primary key). Fast for range queries on the PK.

**Non-clustered index:** Separate structure with a pointer to the row. Multiple allowed per table.

**Other types (awareness):**
- **Hash:** equality only, no range queries.
- **Full-text:** for text search (`MATCH ... AGAINST`).
- **Bitmap:** low-cardinality columns in data warehouses.

---

## Query Optimization

### Q14: How do you optimize SQL queries?

**Answer:**

**1. Use EXPLAIN** to inspect the execution plan — look for full table scans and high row counts.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

**2. Index WHERE, JOIN, ORDER BY, GROUP BY columns.**

**3. Avoid `SELECT *`** — select only needed columns.

**4. Avoid leading-wildcard LIKE** (`'%laptop%'`) — use full-text search instead.

**5. Ensure JOIN columns are indexed.**

**6. Use LIMIT / keyset pagination** for large result sets.

```sql
-- Keyset pagination (faster than OFFSET for large pages)
SELECT * FROM products WHERE id > :last_seen_id ORDER BY id LIMIT 20;
```

**7. Replace correlated subqueries** with JOINs where possible.

---

## Database Security

### Q15: What are database security best practices?

**Answer:**

**1. Least privilege:** Grant only the permissions a user needs.

```sql
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT SELECT, INSERT ON app_db.users TO 'app_user'@'localhost';
```

**2. Prevent SQL injection — always use prepared statements.**

```sql
-- ❌ BAD: string concatenation
$sql = "SELECT * FROM users WHERE email = '" . $email . "'";

-- ✅ GOOD: prepared statement
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => $email]);
```

**3. Encrypt sensitive data:** hash passwords (bcrypt), encrypt columns at rest, require TLS for connections.

**4. Regular backups:** schedule `mysqldump`/`pg_dump` and test restores periodically.

---

## NoSQL vs SQL

### Q16: What are the key differences between SQL and NoSQL?

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| Data model | Relational tables | Document, key-value, column, graph |
| Schema | Fixed | Flexible |
| Query language | SQL | DB-specific API |
| Scalability | Vertical | Horizontal |
| Consistency | Strong (ACID) | Eventual (BASE) |
| Transactions | Full ACID | Varies |

**Choose SQL when:** data is structured, transactions are critical, complex joins/reporting are needed, data integrity is paramount.

**Choose NoSQL when:** schema evolves rapidly, horizontal scaling is needed, data is unstructured, simple access patterns suffice.

---

## Common Database Mistakes

```sql
-- Mistake 1: No index on frequently queried column
CREATE INDEX idx_users_email ON users(email);  -- ✅ add the index

-- Mistake 2: SELECT * fetches unnecessary columns
SELECT id, name, email FROM users;  -- ✅ select only what you need

-- Mistake 3: Multi-step operations without a transaction
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- ✅ atomic

-- Mistake 4: Over-normalization → too many joins hurt performance
-- Balance 3NF with practical denormalization where needed

-- Mistake 5: Forgetting NULL handling
SELECT * FROM users WHERE email = 'x@x.com' OR email IS NULL;  -- ✅
```

---

## Short Revision Summary

**ACID:** Atomicity (all or nothing) · Consistency (valid state) · Isolation (no interference) · Durability (survives crash)

**Normal forms:** 1NF (atomic values) → 2NF (no partial deps) → 3NF (no transitive deps) → BCNF

**SQL categories:** DDL (CREATE/ALTER/DROP) · DML (SELECT/INSERT/UPDATE/DELETE) · DCL (GRANT/REVOKE) · TCL (COMMIT/ROLLBACK)

**Joins:** INNER (match both) · LEFT (all left) · RIGHT (all right) · FULL (all both) · CROSS (cartesian) · SELF (self-reference)

**Isolation levels:** Read Uncommitted → Read Committed → Repeatable Read → Serializable (increasing isolation, decreasing performance)

**Index types:** B-Tree (default) · Hash (equality only) · Full-Text (search) · Composite (multi-column, leftmost-prefix rule)

### Quick Reference

```sql
-- Create table
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(100), email VARCHAR(100) UNIQUE);

-- Aggregate query
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at >= '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 0
ORDER BY order_count DESC;

-- Transaction
BEGIN TRANSACTION;
-- statements
COMMIT;  -- or ROLLBACK

-- Indexes
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_composite ON table(col1, col2);
```

---

*Last Updated: 2026-06-18*
