# Database Concepts Interview Questions & Answers

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
14. [Short Revision Summary](#short-revision-summary-3)

---

## Database Fundamentals

### Q1: What is a database and what are the different types of databases?

**Answer:** A database is an organized collection of structured information, typically stored electronically in a computer system. It allows for efficient data storage, retrieval, and management.

**Types of Databases:**

**1. Relational Databases (RDBMS):**
- Store data in tables with rows and columns
- Use SQL for querying
- Enforce schema and relationships
- Examples: MySQL, PostgreSQL, Oracle, SQL Server

```sql
-- Example: Relational database structure
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    total DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**2. NoSQL Databases:**
- Document databases: MongoDB, CouchDB
- Key-value stores: Redis, DynamoDB
- Column-family stores: Cassandra, HBase
- Graph databases: Neo4j, Amazon Neptune

```javascript
// Example: Document database (MongoDB)
{
  "_id": "user1",
  "name": "John Doe",
  "email": "john@example.com",
  "orders": [
    {"orderId": "order1", "total": 99.99},
    {"orderId": "order2", "total": 149.99}
  ]
}
```

**3. NewSQL Databases:**
- Combine SQL interface with NoSQL scalability
- Examples: Google Spanner, CockroachDB, NuoDB

**4. Distributed Databases:**
- Data distributed across multiple nodes
- Examples: Cassandra, MongoDB clusters, Google Spanner

**5. In-Memory Databases:**
- Data stored in RAM for fast access
- Examples: Redis, Memcached, SAP HANA

### Q2: What is DBMS and what are its components?

**Answer:** DBMS (Database Management System) is software that allows users to define, create, maintain, and control access to the database.

**DBMS Components:**

```
┌─────────────────────────────────────────┐
│         Users / Applications            │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Query Processor                 │
│  ┌──────────────────────────────────┐  │
│  │  - DML Parser                   │  │
│  │  - DDL Parser                   │  │
│  │  - Query Optimizer              │  │
│  │  - Query Executor               │  │
│  └──────────────────────────────────┘  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Storage Manager                 │
│  ┌──────────────────────────────────┐  │
│  │  - Buffer Manager               │  │
│  │  - Transaction Manager          │  │
│  │  - Log Manager                  │  │
│  │  - Recovery Manager             │  │
│  └──────────────────────────────────┘  │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│         Data Files                      │
│  ┌──────────┐  ┌──────────┐            │
│  │  Data    │  │  Index   │            │
│  │  Files   │  │  Files   │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
```

**Key Components:**

**1. Query Processor:**
- **Parser**: Validates SQL syntax and converts to parse tree
- **Optimizer**: Chooses most efficient execution plan
- **Executor**: Executes the query plan

**2. Storage Manager:**
- **Buffer Manager**: Manages data transfer between disk and memory
- **Transaction Manager**: Ensures ACID properties
- **Log Manager**: Maintains transaction logs for recovery
- **Recovery Manager**: Restores database after failures

**3. Data Files:**
- Store actual data and indexes
- Organized in pages/blocks
- Managed by file system

### Q3: What are the different levels of data independence?

**Answer:** Data independence refers to the capacity to change the schema at one level without affecting the schema at the next higher level.

**Three Levels of Data Independence:**

**1. Physical Level (Internal Schema):**
- Describes how data is stored in storage
- File organization, indexing, storage allocation
- Example: B-tree index, hash table, partitioning

**2. Logical Level (Conceptual Schema):**
- Describes what data is stored and relationships
- Entity types, attributes, relationships, constraints
- Example: Customer has orders, products have categories

**3. View Level (External Schema):**
- Describes how users see the data
- User views, security, access control
- Example: Sales view sees only order data, HR view sees employee data

```
┌─────────────────────────────────────────┐
│         View Level                      │
│    (User Views, External Schema)        │
└──────────────────┬──────────────────────┘
                   │ Logical Independence
┌──────────────────▼──────────────────────┐
│         Logical Level                   │
│    (Conceptual Schema, Tables)          │
└──────────────────┬──────────────────────┘
                   │ Physical Independence
┌──────────────────▼──────────────────────┐
│         Physical Level                  │
│    (Internal Schema, Storage)           │
└─────────────────────────────────────────┘
```

**Examples:**

```sql
-- Physical Independence: Change storage without affecting logical schema
-- Original: Table stored as heap file
-- Modified: Table stored with B-tree index
-- Logical schema (tables, columns) remains unchanged

-- Logical Independence: Change logical schema without affecting views
-- Original: Single customer table
-- Modified: Split into customer and address tables
-- Views can be created to maintain same interface
CREATE VIEW customer_view AS
SELECT c.*, a.street, a.city
FROM customers c
LEFT JOIN addresses a ON c.id = a.customer_id;
```

---

## ACID Properties

### Q4: What are ACID properties in database transactions?

**Answer:** ACID is a set of properties that guarantee database transactions are processed reliably.

**A - Atomicity:**
- All operations in a transaction succeed or none do
- No partial updates
- Implemented using transaction logs and rollback

```sql
-- Example: Bank transfer (atomic operation)
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Deduct from sender
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Add to receiver
-- If either UPDATE fails, both are rolled back
COMMIT;
```

**C - Consistency:**
- Database moves from one valid state to another
- All constraints are maintained (foreign keys, unique, check constraints)
- Data integrity rules are enforced

```sql
-- Example: Maintaining consistency through constraints
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT NOT NULL,
    total DECIMAL(10,2) CHECK (total >= 0),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Database ensures:
-- - Order ID is unique
-- - User must exist
-- - Total cannot be negative
```

**I - Isolation:**
- Concurrent transactions don't interfere with each other
- Each transaction sees a consistent view of data
- Implemented using locking and MVCC

```sql
-- Example: Isolation prevents concurrent issues
-- Transaction 1:
BEGIN TRANSACTION;
SELECT balance FROM accounts WHERE id = 1;  -- Reads 1000

-- Transaction 2 (concurrent):
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance + 500 WHERE id = 1;
COMMIT;  -- Balance now 1500

-- Transaction 1 (continues):
UPDATE accounts SET balance = balance + 200 WHERE id = 1;
-- Balance becomes 1200 or 1700 depending on isolation level
COMMIT;
```

**D - Durability:**
- Once committed, changes are permanent
- Survives system failures, power loss, crashes
- Implemented using write-ahead logging and replication

```sql
-- Example: Durability ensures data persistence
BEGIN TRANSACTION;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
COMMIT;  -- Even if system crashes immediately, order is saved
```

**ACID Violation Examples:**

```sql
-- Atomicity Violation (partial update):
-- Money deducted but not credited if system fails between two UPDATEs

-- Consistency Violation:
-- Referential integrity broken if foreign key constraint violated

-- Isolation Violation:
-- Dirty reads: Reading uncommitted data
-- Non-repeatable reads: Same query returns different results
-- Phantom reads: New rows appear in subsequent queries

-- Durability Violation:
-- Committed data lost if system crashes before write to disk
```

---

## Normalization

### Q5: What is database normalization and what are its normal forms?

**Answer:** Normalization is the process of organizing data to minimize redundancy and dependency. It involves dividing large tables into smaller, related tables and defining relationships.

**Benefits of Normalization:**
- Eliminates data redundancy
- Prevents data anomalies (insertion, update, deletion)
- Improves data integrity
- Simplifies database design

**Normal Forms:**

**First Normal Form (1NF):**
- Each table cell should contain a single value
- Each record needs to be unique

```sql
-- ❌ NOT 1NF: Multiple values in single cell
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    products VARCHAR(500)  -- "Laptop, Mouse, Keyboard"
);

-- ✅ 1NF: Single value per cell
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_name VARCHAR(100),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

**Second Normal Form (2NF):**
- Must be in 1NF
- All non-key attributes must depend on the entire primary key
- No partial dependencies

```sql
-- ❌ NOT 2NF: Partial dependency
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,           -- Depends on (order_id, product_id)
    product_name VARCHAR(100),  -- Depends only on product_id
    PRIMARY KEY (order_id, product_id)
);

-- ✅ 2NF: Remove partial dependencies
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100)
);
```

**Third Normal Form (3NF):**
- Must be in 2NF
- No transitive dependencies (non-key attributes depending on other non-key attributes)

```sql
-- ❌ NOT 3NF: Transitive dependency
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- Depends on customer_id, not order_id
    order_date DATE,
    total DECIMAL(10,2)
);

-- ✅ 3NF: Remove transitive dependencies
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100)
);
```

**Boyce-Codd Normal Form (BCNF):**
- Stronger version of 3NF
- Every determinant must be a candidate key

**Fourth Normal Form (4NF):**
- Must be in BCNF
- No multi-valued dependencies

**Fifth Normal Form (5NF):**
- Must be in 4NF
- No join dependencies

**Denormalization:**
- Sometimes beneficial for performance
- Trade-off between redundancy and query speed
- Common in data warehouses and read-heavy applications

```sql
-- Example: Denormalized for read performance
CREATE TABLE order_summary (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    order_date DATE,
    product_names TEXT,  -- Denormalized product names
    total DECIMAL(10,2)
);

-- Benefits: Faster queries, fewer joins
-- Drawbacks: Data redundancy, update anomalies
```

---

## SQL Fundamentals

### Q6: What are the different types of SQL commands?

**Answer:** SQL commands are categorized based on their functionality:

**1. DDL (Data Definition Language):**
- Define database structure
- Commands: CREATE, ALTER, DROP, TRUNCATE, RENAME

```sql
-- CREATE: Create database objects
CREATE DATABASE mydb;
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
CREATE INDEX idx_user_name ON users(name);

-- ALTER: Modify database objects
ALTER TABLE users ADD COLUMN email VARCHAR(100);
ALTER TABLE users MODIFY COLUMN name VARCHAR(200);
ALTER TABLE users DROP COLUMN email;

-- DROP: Remove database objects
DROP TABLE users;
DROP DATABASE mydb;
DROP INDEX idx_user_name;

-- TRUNCATE: Remove all rows from table
TRUNCATE TABLE users;  -- Faster than DELETE, resets auto-increment

-- RENAME: Rename database objects
RENAME TABLE old_name TO new_name;
```

**2. DML (Data Manipulation Language):**
- Manipulate data within database objects
- Commands: SELECT, INSERT, UPDATE, DELETE

```sql
-- SELECT: Retrieve data
SELECT * FROM users;
SELECT name, email FROM users WHERE id = 1;

-- INSERT: Insert new data
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');
INSERT INTO users (name, email) VALUES ('Jane', 'jane@example.com'), ('Bob', 'bob@example.com');

-- UPDATE: Modify existing data
UPDATE users SET email = 'newemail@example.com' WHERE id = 1;

-- DELETE: Remove data
DELETE FROM users WHERE id = 1;
```

**3. DCL (Data Control Language):**
- Control access to data
- Commands: GRANT, REVOKE

```sql
-- GRANT: Give permissions
GRANT SELECT, INSERT ON users TO 'app_user'@'localhost';
GRANT ALL PRIVILEGES ON mydb.* TO 'admin'@'localhost';

-- REVOKE: Remove permissions
REVOKE INSERT ON users FROM 'app_user'@'localhost';
REVOKE ALL PRIVILEGES ON mydb.* FROM 'admin'@'localhost';
```

**4. TCL (Transaction Control Language):**
- Manage transactions
- Commands: COMMIT, ROLLBACK, SAVEPOINT

```sql
-- COMMIT: Save changes
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Changes are permanent

-- ROLLBACK: Undo changes
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Something went wrong
ROLLBACK;  -- Changes are undone

-- SAVEPOINT: Create rollback point within transaction
SAVEPOINT before_delete;
DELETE FROM temp_table;
-- Something went wrong
ROLLBACK TO before_delete;  -- Rollback only to savepoint
```

### Q7: What are the different types of joins in SQL?

**Answer:** Joins combine rows from two or more tables based on related columns.

**Types of Joins:**

**1. INNER JOIN:**
- Returns rows when there's a match in both tables
- Most common type of join

```sql
SELECT users.name, orders.order_date, orders.total
FROM users
INNER JOIN orders ON users.id = orders.user_id;

-- Only returns users who have placed orders
```

**2. LEFT JOIN (LEFT OUTER JOIN):**
- Returns all rows from left table, matching rows from right table
- NULL for non-matching rows from right table

```sql
SELECT users.name, orders.order_date
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- Returns all users, even those without orders (order_date will be NULL)
```

**3. RIGHT JOIN (RIGHT OUTER JOIN):**
- Returns all rows from right table, matching rows from left table
- NULL for non-matching rows from left table

```sql
SELECT users.name, orders.order_date
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;

-- Returns all orders, even without matching users (unlikely with good FK)
```

**4. FULL JOIN (FULL OUTER JOIN):**
- Returns rows when there's a match in either table
- NULL where there's no match

```sql
SELECT users.name, orders.order_date
FROM users
FULL OUTER JOIN orders ON users.id = orders.user_id;

-- Returns all users and all orders
```

**5. CROSS JOIN:**
- Cartesian product of all rows
- Returns every possible combination

```sql
SELECT users.name, products.name as product_name
FROM users
CROSS JOIN products;

-- Returns every combination of users and products
```

**6. SELF JOIN:**
- Joins a table to itself
- Useful for hierarchical data

```sql
SELECT
    e1.name as employee,
    e2.name as manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;

-- Returns employees with their managers
```

**Join Examples with Multiple Tables:**

```sql
-- Complex join with multiple tables
SELECT
    u.username,
    o.order_date,
    o.total as order_total,
    p.name as product_name,
    oi.quantity,
    oi.price as unit_price
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
WHERE o.order_date >= '2024-01-01'
ORDER BY o.order_date DESC, u.username;
```

---

## Advanced SQL Concepts

### Q8: What are aggregate functions and GROUP BY?

**Answer:** Aggregate functions perform calculations on a set of values and return a single value.

**Common Aggregate Functions:**

```sql
-- COUNT: Count number of rows
SELECT COUNT(*) FROM users;
SELECT COUNT(email) FROM users;  -- Counts non-null emails
SELECT COUNT(DISTINCT country) FROM users;

-- SUM: Sum of values
SELECT SUM(total) FROM orders;
SELECT SUM(total) FROM orders WHERE status = 'completed';

-- AVG: Average of values
SELECT AVG(price) FROM products;
SELECT AVG(total) FROM orders GROUP BY user_id;

-- MIN/MAX: Minimum/Maximum values
SELECT MIN(price), MAX(price) FROM products;
SELECT MIN(order_date), MAX(order_date) FROM orders;

-- GROUP BY: Group rows that have the same values
SELECT
    user_id,
    COUNT(*) as order_count,
    SUM(total) as total_spent,
    AVG(total) as avg_order_value
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 5;  -- Filter groups

-- Multiple grouping
SELECT
    YEAR(order_date) as year,
    MONTH(order_date) as month,
    status,
    COUNT(*) as order_count,
    SUM(total) as total_revenue
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date), status
ORDER BY year, month, status;
```

**Advanced Aggregation:**

```sql
-- ROLLUP: Create subtotals and grand totals
SELECT
    COALESCE(region, 'All Regions') as region,
    COALESCE(category, 'All Categories') as category,
    SUM(sales) as total_sales
FROM sales_data
GROUP BY ROLLUP(region, category);

-- CUBE: Create all possible combinations
SELECT
    COALESCE(region, 'All') as region,
    COALESCE(category, 'All') as category,
    COALESCE(product, 'All') as product,
    SUM(sales) as total_sales
FROM sales_data
GROUP BY CUBE(region, category, product);

-- GROUPING SETS: Specify custom groupings
SELECT
    region,
    category,
    SUM(sales) as total_sales
FROM sales_data
GROUP BY GROUPING SETS (
    (region, category),  -- Group by region and category
    (region),            -- Group by region only
    (category),          -- Group by category only
    ()                   -- Grand total
);
```

### Q9: What are subqueries and correlated subqueries?

**Answer:** Subqueries are queries nested within another query.

**Types of Subqueries:**

**1. Scalar Subquery (returns single value):**

```sql
-- Find products with price higher than average
SELECT name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Find users with order count above average
SELECT username, order_count
FROM (
    SELECT
        u.username,
        COUNT(o.id) as order_count
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.username
) user_orders
WHERE order_count > (SELECT AVG(order_count) FROM (
    SELECT COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
) order_counts);
```

**2. Row Subquery (returns single row):**

```sql
-- Find products matching specific criteria
SELECT * FROM products
WHERE (price, category) = (
    SELECT price, category
    FROM products
    WHERE name = 'Reference Product'
);
```

**3. Column Subquery (returns single column):**

```sql
-- Find users who have placed orders
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- Find products in specific categories
SELECT * FROM products
WHERE category_id IN (SELECT id FROM categories WHERE type = 'electronics');
```

**4. Table Subquery (returns multiple rows/columns):**

```sql
-- Derived table (subquery in FROM clause)
SELECT
    user_orders.username,
    user_orders.order_count,
    user_orders.total_spent
FROM (
    SELECT
        u.username,
        COUNT(o.id) as order_count,
        COALESCE(SUM(o.total), 0) as total_spent
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    GROUP BY u.id, u.username
) user_orders
WHERE user_orders.order_count > 0;
```

**Correlated Subqueries:**

```sql
-- Correlated subquery references outer query
-- Find users with above-average order totals
SELECT u.username, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.total > (
    SELECT AVG(o2.total)
    FROM orders o2
    WHERE o2.user_id = u.id  -- Correlated with outer query
);

-- Find products more expensive than all products in their category
SELECT p1.name, p1.price, p1.category
FROM products p1
WHERE p1.price > ALL (
    SELECT p2.price
    FROM products p2
    WHERE p2.category = p1.category  -- Correlated
    AND p2.id != p1.id
);

-- EXISTS with correlated subquery
SELECT u.username
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id  -- Correlated
    AND o.total > 1000
);
```

**Performance Considerations:**

```sql
-- ❌ BAD: Correlated subquery can be slow
SELECT u.username
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
);

-- ✅ GOOD: JOIN is usually faster
SELECT DISTINCT u.username
FROM users u
JOIN orders o ON u.id = o.user_id;

-- ✅ GOOD: Use IN with subquery for single column
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- ✅ GOOD: Use EXISTS for existence check
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

---

## Database Design

### Q10: What are the different types of relationships in database design?

**Answer:** Relationships define how tables relate to each other in a relational database.

**Types of Relationships:**

**1. One-to-One (1:1):**
- Each record in Table A relates to one record in Table B
- Uncommon, often combined into single table

```sql
-- Example: User and Profile
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100)
);

CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    bio TEXT,
    avatar_url VARCHAR(255),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Each user has exactly one profile
```

**2. One-to-Many (1:N):**
- Each record in Table A can relate to multiple records in Table B
- Most common relationship type
- Implemented with foreign key in "many" table

```sql
-- Example: User and Orders
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    order_date DATE,
    total DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Each user can have multiple orders
-- Each order belongs to one user
```

**3. Many-to-Many (N:M):**
- Records in Table A can relate to multiple records in Table B, and vice versa
- Implemented with junction/link table

```sql
-- Example: Students and Courses
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

-- Junction table
CREATE TABLE student_courses (
    student_id INT,
    course_id INT,
    enrollment_date DATE,
    PRIMARY KEY (student_id, course_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);

-- Each student can take multiple courses
-- Each course can have multiple students
```

**Self-Referencing Relationships:**

```sql
-- Example: Employee hierarchy (Employee and Manager)
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    manager_id INT,
    FOREIGN KEY (manager_id) REFERENCES employees(id)
);

-- Each employee can have one manager
-- Each manager can have multiple employees
```

**Relationship Cardinality and Participation:**

```sql
-- Cardinality: Maximum number of relationships
-- - One-to-One: 1:1
-- - One-to-Many: 1:N or N:1
-- - Many-to-Many: N:M

-- Participation: Minimum number of relationships
-- - Total (Mandatory): Every entity must participate
-- - Partial (Optional): Entity may or may not participate

-- Example: Total participation
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT NOT NULL,  -- Every order must have a user
    total DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Example: Partial participation
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    phone VARCHAR(20)  -- Phone is optional
);
```

---

## Transaction Management

### Q11: What are transaction isolation levels?

**Answer:** Transaction isolation levels define the degree to which transactions must be isolated from each other.

**Isolation Levels:**

**1. Read Uncommitted:**
- Lowest isolation level
- Can read uncommitted changes from other transactions
- Possible anomalies: Dirty reads, non-repeatable reads, phantom reads

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Transaction 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Uncommitted

-- Transaction 2 can read uncommitted change:
SELECT balance FROM accounts WHERE id = 1;  -- Sees -100
```

**2. Read Committed:**
- Only reads committed changes
- Prevents dirty reads
- Possible anomalies: Non-repeatable reads, phantom reads

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Transaction 1:
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Reads 1000

-- Transaction 2:
BEGIN;
UPDATE accounts SET balance = balance + 500 WHERE id = 1;
COMMIT;  -- Balance now 1500

-- Transaction 1 (continuing):
SELECT balance FROM accounts WHERE id = 1;  -- Now reads 1500 (non-repeatable read)
```

**3. Repeatable Read:**
- Sees consistent snapshot throughout transaction
- Prevents dirty reads, non-repeatable reads
- Possible anomalies: Phantom reads

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Transaction 1:
BEGIN;
SELECT * FROM orders WHERE user_id = 1;  -- Returns 5 orders

-- Transaction 2:
BEGIN;
INSERT INTO orders (user_id, total) VALUES (1, 100);
COMMIT;  -- New order added

-- Transaction 1 (continuing):
SELECT * FROM orders WHERE user_id = 1;  -- Still returns 5 orders (no phantom)
```

**4. Serializable:**
- Highest isolation level
- Transactions are completely isolated
- Prevents all anomalies (dirty reads, non-repeatable reads, phantom reads)
- Lowest performance due to locking

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Transactions appear to execute sequentially
-- Prevents all concurrency issues
-- Highest performance cost
```

**Comparison Table:**

| Isolation Level | Dirty Reads | Non-repeatable Reads | Phantom Reads | Performance |
|-----------------|-------------|---------------------|---------------|-------------|
| Read Uncommitted | Possible | Possible | Possible | Best |
| Read Committed | Prevented | Possible | Possible | Good |
| Repeatable Read | Prevented | Prevented | Possible | Fair |
| Serializable | Prevented | Prevented | Prevented | Worst |

---

## Concurrency Control

### Q12: What are the different types of locks in databases?

**Answer:** Locks are used to manage concurrent access to data and prevent conflicts.

**Types of Locks:**

**1. Shared Lock (S-Lock):**
- Allows multiple transactions to read data
- Prevents writes during read
- Used for SELECT operations

```sql
-- Explicit shared lock
SELECT * FROM products WHERE id = 1 FOR SHARE;  -- PostgreSQL
SELECT * FROM products WHERE id = 1 LOCK IN SHARE MODE;  -- MySQL

-- Multiple transactions can hold shared locks on same data
-- But no transaction can hold exclusive lock
```

**2. Exclusive Lock (X-Lock):**
- Allows only one transaction to write data
- Prevents both reads and writes
- Used for UPDATE, DELETE, INSERT operations

```sql
-- Explicit exclusive lock
SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- PostgreSQL/MySQL

-- Only one transaction can hold exclusive lock
-- No other transaction can hold any lock on same data
```

**3. Intention Locks:**
- Indicate intention to acquire shared/exclusive locks
- IS (Intention Shared): Intends to read some rows
- IX (Intention Exclusive): Intends to write some rows

```sql
-- When transaction locks row with X-lock:
-- - Acquires IX lock on table
-- - Acquires X lock on row

-- When transaction locks row with S-lock:
-- - Acquires IS lock on table
-- - Acquires S lock on row
```

**4. Other Lock Types:**

```sql
-- Update Lock (U-Lock): Prevents deadlocks in concurrent updates
SELECT * FROM products WITH (UPDLOCK) WHERE id = 1;  -- SQL Server

-- Gap Locks: Locks gap between index records
-- Prevents phantom reads
-- Used in Repeatable Read isolation level

-- Next-Key Locks: Combination of record lock and gap lock
-- Prevents phantom reads and insertion of new records
```

**Lock Compatibility:**

| | IS | IX | S | X |
|---|----|----|---|---|
| **IS** | ✓ | ✓ | ✓ | ✗ |
| **IX** | ✓ | ✓ | ✗ | ✗ |
| **S** | ✓ | ✗ | ✓ | ✗ |
| **X** | ✗ | ✗ | ✗ | ✗ |

**Deadlock:**

```sql
-- Deadlock scenario:
-- Transaction 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks account 1
-- Waiting for account 2...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Transaction 2:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- Locks account 2
-- Waiting for account 1...
UPDATE accounts SET balance = balance + 100 WHERE id = 1;

-- Both transactions waiting for each other → DEADLOCK

-- Database detects deadlock and rolls back one transaction
```

**Deadlock Prevention:**

```sql
-- 1. Always access tables in same order
-- ❌ BAD: Different order
-- T1: UPDATE accounts SET ... WHERE id = 1; UPDATE accounts SET ... WHERE id = 2;
-- T2: UPDATE accounts SET ... WHERE id = 2; UPDATE accounts SET ... WHERE id = 1;

-- ✅ GOOD: Same order
-- T1: UPDATE accounts SET ... WHERE id = 1; UPDATE accounts SET ... WHERE id = 2;
-- T2: UPDATE accounts SET ... WHERE id = 1; UPDATE accounts SET ... WHERE id = 2;

-- 2. Keep transactions short
-- 3. Use appropriate isolation level
-- 4. Add indexes to reduce lock time
-- 5. Use lower isolation level when possible
```

---

## Database Indexing

### Q13: What are the different types of database indexes?

**Answer:** Indexes are data structures that improve the speed of data retrieval operations on database tables.

**Types of Indexes:**

**1. B-Tree Index (Default):**
- Balanced tree structure
- Good for equality and range queries
- Most common index type

```sql
-- Create B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Used for:
SELECT * FROM users WHERE email = 'user@example.com';  -- Equality
SELECT * FROM users WHERE email > 'a@b.c';              -- Range
SELECT * FROM users WHERE email LIKE 'user%';          -- Prefix

-- Not efficient for:
SELECT * FROM users WHERE email LIKE '%user%';         -- Leading wildcard
SELECT * FROM users WHERE LOWER(email) = 'USER@EXAMPLE.COM';  -- Function on column
```

**2. Hash Index:**
- Hash table structure
- Only for equality operations
- Faster than B-tree for equality

```sql
-- Create hash index
CREATE INDEX idx_products_hash ON products USING HASH (name);

-- Used for:
SELECT * FROM products WHERE name = 'Laptop';  -- Equality only

-- Not for:
SELECT * FROM products WHERE name > 'L';       -- Range
SELECT * FROM products WHERE name LIKE 'Lap%';  -- Pattern
```

**3. Bitmap Index:**
- Uses bitmaps for each key value
- Good for low-cardinality columns
- Common in data warehouses

```sql
-- Create bitmap index
CREATE BITMAP INDEX idx_users_gender ON users(gender);

-- Good for:
SELECT * FROM users WHERE gender = 'M';
-- WHERE gender IN ('M', 'F')

-- Not good for:
-- High-cardinality columns (like email, ID)
-- Frequent updates
```

**4. Full-Text Index:**
- Specialized for text search
- Supports linguistic search

```sql
-- Create full-text index
CREATE FULLTEXT INDEX idx_articles_content ON articles(content);

-- Used for:
SELECT * FROM articles
WHERE MATCH(content) AGAINST('database search' IN NATURAL LANGUAGE MODE);

SELECT * FROM articles
WHERE MATCH(content) AGAINST('+"database" +"search"' IN BOOLEAN MODE);
```

**5. Composite Index:**
- Index on multiple columns
- Order of columns matters

```sql
-- Create composite index
CREATE INDEX idx_orders_user_date_status ON orders(user_id, order_date, status);

-- Used for:
WHERE user_id = 1 AND order_date = '2024-01-01' AND status = 'completed';  -- All columns
WHERE user_id = 1 AND order_date = '2024-01-01';                          -- First two columns
WHERE user_id = 1;                                                         -- First column only

-- Not efficient for:
WHERE order_date = '2024-01-01';         -- Missing leading column
WHERE status = 'completed';              -- Missing leading columns
WHERE status = 'completed' AND user_id = 1;  -- Wrong column order
```

**6. Clustered Index:**
- Data rows stored in index order
- Only one per table
- Usually on primary key

```sql
-- Clustered index is usually created automatically on primary key
CREATE TABLE users (
    id INT PRIMARY KEY,  -- Creates clustered index
    name VARCHAR(100)
);

-- Benefits: Fast range queries on primary key
-- Drawbacks: Slower inserts (must maintain order)
```

**7. Non-Clustered Index:**
- Separate structure from data
- Multiple per table allowed
- Contains pointer to actual data

```sql
-- Create non-clustered index
CREATE INDEX idx_users_email ON users(email);

-- Benefits: Multiple indexes, flexible
-- Drawbacks: Additional lookup to get data
```

---

## Query Optimization

### Q14: How do you optimize SQL queries?

**Answer:** Query optimization improves the performance of SQL queries.

**Optimization Techniques:**

**1. Use EXPLAIN to analyze queries:**

```sql
-- Analyze query execution plan
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Look for:
-- - Scan type: Index scan (good) vs Full table scan (bad)
-- - Rows examined: Fewer is better
-- - Index usage: Which index is being used

-- MySQL: EXPLAIN, EXPLAIN ANALYZE
-- PostgreSQL: EXPLAIN, EXPLAIN ANALYZE, EXPLAIN (BUFFERS)
-- SQL Server: Execution plan
```

**2. Create appropriate indexes:**

```sql
-- Index columns used in WHERE, JOIN, ORDER BY, GROUP BY
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Use covering indexes
CREATE INDEX idx_orders_covering ON orders(user_id, status, total);

-- Use partial indexes (PostgreSQL)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

**3. Optimize SELECT statements:**

```sql
-- ❌ BAD: SELECT * retrieves all columns
SELECT * FROM users;

-- ✅ GOOD: Select only needed columns
SELECT id, name, email FROM users;

-- ❌ BAD: No WHERE clause
SELECT * FROM orders;

-- ✅ GOOD: Use WHERE to filter
SELECT * FROM orders WHERE order_date >= '2024-01-01';

-- ❌ BAD: Leading wildcard in LIKE
SELECT * FROM products WHERE name LIKE '%laptop%';

-- ✅ GOOD: Use full-text search
SELECT * FROM products
WHERE MATCH(name) AGAINST('laptop' IN NATURAL LANGUAGE MODE);
```

**4. Optimize JOINs:**

```sql
-- Ensure JOIN columns are indexed
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);

-- Use INNER JOIN when possible (faster)
SELECT u.name, o.order_date
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Avoid unnecessary joins
-- ❌ BAD: Join when subquery would suffice
SELECT u.*
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.total > 100;

-- ✅ GOOD: Use EXISTS
SELECT u.* FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 100);
```

**5. Optimize subqueries:**

```sql
-- ❌ BAD: Correlated subquery (executed for each row)
SELECT u.*
FROM users u
WHERE (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) > 5;

-- ✅ GOOD: JOIN with GROUP BY
SELECT u.*, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.id
HAVING COUNT(o.id) > 5;
```

**6. Use LIMIT for pagination:**

```sql
-- Basic pagination
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;  -- Page 1

-- Better: Keyset pagination for large offsets
SELECT * FROM products
WHERE id > last_seen_id
ORDER BY id
LIMIT 20;
```

---

## Database Security

### Q15: What are the database security best practices?

**Answer:** Database security protects data from unauthorized access and attacks.

**Security Best Practices:**

**1. User Authentication:**

```sql
-- Create users with strong passwords
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';

-- Use least privilege principle
GRANT SELECT, INSERT ON app_db.users TO 'app_user'@'localhost';

-- Regularly review and remove unused accounts
DROP USER IF EXISTS 'old_user'@'localhost';

-- Enable password expiration
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE INTERVAL 90 DAY;
```

**2. Access Control:**

```sql
-- Grant specific permissions only
GRANT SELECT ON app_db.* TO 'read_only_user'@'%';
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user'@'%';
GRANT ALL PRIVILEGES ON app_db.* TO 'admin'@'localhost';

-- Use views to restrict data access
CREATE VIEW customer_orders AS
SELECT o.id, o.order_date, o.total
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.id = CURRENT_USER_ID();

GRANT SELECT ON customer_orders TO 'customer_user'@'%';
```

**3. Data Encryption:**

```sql
-- Encrypt sensitive data
-- MySQL: Use AES_ENCRYPT
INSERT INTO users (username, password_hash, ssn)
VALUES ('user1', SHA2('password', 256), AES_ENCRYPT('123-45-6789', 'encryption_key'));

-- PostgreSQL: Use pgcrypto extension
CREATE EXTENSION pgcrypto;
INSERT INTO users (username, password_hash, ssn)
VALUES ('user1', crypt('password', gen_salt('bf')), encrypt('123-45-6789', 'encryption_key'));

-- Enable TLS/SSL for connections
-- Require SSL for specific users
CREATE USER 'secure_user'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
```

**4. SQL Injection Prevention:**

```sql
-- ❌ BAD: String concatenation (vulnerable to SQL injection)
$sql = "SELECT * FROM users WHERE email = '" . $email . "'";

-- ✅ GOOD: Use prepared statements
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email");
$stmt->execute(['email' => $email]);

-- ✅ GOOD: Use parameterized queries
$stmt = $mysqli->prepare("SELECT * FROM users WHERE email = ?");
$stmt->bind_param("s", $email);
$stmt->execute();
```

**5. Regular Backups:**

```sql
-- Logical backup
mysqldump -u root -p --single-transaction --quick --lock-tables=false app_db > backup.sql

-- Physical backup
pg_dump -U postgres app_db > backup.sql

-- Automated backup script
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
mysqldump -u root -p'password' app_db > $BACKUP_DIR/app_db_$DATE.sql
```

---

## NoSQL vs SQL

### Q16: What are the differences between SQL and NoSQL databases?

**Answer:**

| Aspect | SQL Databases | NoSQL Databases |
|--------|---------------|-----------------|
| **Data Model** | Relational (tables) | Various (document, key-value, etc.) |
| **Schema** | Fixed, predefined | Flexible, dynamic |
| **Query Language** | SQL | Database-specific or API |
| **Scalability** | Vertical (scale up) | Horizontal (scale out) |
| **Consistency** | Strong (ACID) | Varies (BASE, eventual consistency) |
| **Relationships** | Foreign keys, joins | Embedded documents, references |
| **Transactions** | Full ACID support | Varies by database |
| **Use Cases** | Structured data, complex queries | Unstructured data, rapid scaling |

**When to choose SQL:**
- Structured data with clear relationships
- Complex transactions required
- Data integrity is critical
- Complex queries and reporting
- Regulatory compliance requirements
- Existing SQL expertise

**When to choose NoSQL:**
- Unstructured or semi-structured data
- Rapidly evolving schema
- Need for horizontal scaling
- Real-time big data applications
- Simple query patterns
- Rapid prototyping

---

## Common Database Mistakes

### Mistake 1: Not using indexes appropriately

```sql
-- ❌ BAD: No index on frequently queried column
SELECT * FROM users WHERE email = 'user@example.com';

-- ✅ GOOD: Create index
CREATE INDEX idx_users_email ON users(email);
```

### Mistake 2: Using SELECT *

```sql
-- ❌ BAD: Retrieves all columns
SELECT * FROM users;

-- ✅ GOOD: Select only needed columns
SELECT id, name, email FROM users;
```

### Mistake 3: Not using transactions for multi-step operations

```sql
-- ❌ BAD: No transaction
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- ✅ GOOD: With transaction
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### Mistake 4: Over-normalization

```sql
-- ❌ BAD: Too many joins affect performance
-- Separate tables for first name, last name, etc.

-- ✅ GOOD: Balance normalization and performance
-- Combine related fields, denormalize when appropriate
```

### Mistake 5: Not handling NULL values properly

```sql
-- ❌ BAD: Doesn't account for NULL
SELECT * FROM users WHERE email = 'user@example.com';

-- ✅ GOOD: Handles NULL
SELECT * FROM users
WHERE email = 'user@example.com' OR email IS NULL;
```

---

## Short Revision Summary

### Key Database Concepts

**ACID Properties:**
- A: Atomicity - All or nothing
- C: Consistency - Valid state transitions
- I: Isolation - Concurrent transaction independence
- D: Durability - Permanent committed changes

**Normalization:**
- 1NF: Single values per cell, unique records
- 2NF: No partial dependencies
- 3NF: No transitive dependencies
- BCNF: Stronger 3NF
- Denormalization for performance

**SQL Commands:**
- DDL: CREATE, ALTER, DROP, TRUNCATE
- DML: SELECT, INSERT, UPDATE, DELETE
- DCL: GRANT, REVOKE
- TCL: COMMIT, ROLLBACK, SAVEPOINT

**Joins:**
- INNER JOIN: Matching rows only
- LEFT JOIN: All from left, matching from right
- RIGHT JOIN: All from right, matching from left
- FULL JOIN: All rows from both tables
- CROSS JOIN: Cartesian product
- SELF JOIN: Join table to itself

**Transaction Isolation Levels:**
- Read Uncommitted: Lowest isolation
- Read Committed: No dirty reads
- Repeatable Read: No dirty/non-repeatable reads
- Serializable: Highest isolation

**Index Types:**
- B-Tree: Default, good for most cases
- Hash: Equality only
- Bitmap: Low cardinality
- Full-Text: Text search
- Composite: Multiple columns

### Quick Reference

**Create Table:**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE
);
```

**Basic Query:**
```sql
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at >= '2024-01-01'
GROUP BY u.id, u.name
HAVING COUNT(o.id) > 0
ORDER BY order_count DESC;
```

**Transaction:**
```sql
BEGIN TRANSACTION;
-- SQL statements
COMMIT;  -- or ROLLBACK;
```

**Create Index:**
```sql
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_composite ON table(col1, col2);
```

### Critical Points for Interviews:

1. **ACID**: Atomicity, Consistency, Isolation, Durability - fundamental for transactions
2. **Normalization**: 1NF, 2NF, 3NF - reduce redundancy, prevent anomalies
3. **SQL Commands**: DDL, DML, DCL, TCL - different purposes
4. **Joins**: Understand each type and when to use them
5. **Indexing**: B-tree is default, choose based on query patterns
6. **Transaction Isolation**: Read Uncommitted → Read Committed → Repeatable Read → Serializable
7. **Query Optimization**: Use EXPLAIN, appropriate indexes, avoid SELECT *
8. **Database Design**: Relationships (1:1, 1:N, N:M), foreign keys
9. **Concurrency Control**: Locks (shared, exclusive), deadlocks
10. **Security**: Least privilege, prepared statements, encryption

---

**Next Topics to Study:**
- Database-Specific Interview Questions (MySQL, PostgreSQL, MongoDB)
- Database Administration and Maintenance
- Cloud Database Services
- Data Warehousing and Business Intelligence
- Big Data and Distributed Databases