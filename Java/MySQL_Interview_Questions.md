# MySQL Interview Questions & Answers

## MySQL Basics

### Q1: What is MySQL and why is it used?

**Answer:** MySQL is an open-source relational database management system (RDBMS) that uses SQL to manage structured data. It's trusted by companies like Facebook and Netflix because it's ACID-compliant, performant on read-heavy workloads, easy to set up, and has a massive community. InnoDB (the default engine) provides full transaction support and row-level locking.

### Q2: What are the different MySQL storage engines?

**Answer:**

| Engine | Transactions | Locking | Best For |
|--------|-------------|---------|----------|
| InnoDB (default) | Yes (ACID) | Row-level | Most apps needing data integrity |
| MyISAM | No | Table-level | Read-only/legacy data |
| Memory | No | Table-level | Temporary data, caching |

```sql
CREATE TABLE users (id INT PRIMARY KEY AUTO_INCREMENT, username VARCHAR(50)) ENGINE=InnoDB;
```

### Q3: What is the difference between CHAR and VARCHAR?

**Answer:** `CHAR` is fixed-length (always pads to declared size); `VARCHAR` is variable-length (stores only actual data + 1–2 length bytes). Use `CHAR` for truly fixed-length data like country codes (`CHAR(2)`); use `VARCHAR` for names, emails, and other variable content.

---

## Data Types & Storage

### Q4: What are the different numeric data types in MySQL?

**Answer:**

| Type | Size | Range (Signed) | Use Case |
|------|------|----------------|----------|
| TINYINT | 1 byte | -128 to 127 | Booleans, small flags |
| INT | 4 bytes | -2B to 2B | Standard IDs |
| BIGINT | 8 bytes | ±9 quintillion | Large IDs, timestamps |
| DECIMAL | Variable | Exact | Money (never use FLOAT) |

```sql
-- ✅ GOOD: right types for the job
CREATE TABLE transactions (
    id     INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    flag   TINYINT(1),           -- boolean
    amount DECIMAL(10,2)         -- money, exact precision
);
```

### Q5: What are the temporal data types in MySQL?

**Answer:**

| Type | Size | Range | Use Case |
|------|------|-------|----------|
| DATE | 3 bytes | 1000–9999 | Birthdays, events |
| DATETIME | 8 bytes | 1000–9999 | Specific moments (no TZ conversion) |
| TIMESTAMP | 4 bytes | 1970–2038 | Auto-updating, TZ-aware |

Key distinction: `TIMESTAMP` converts to/from UTC automatically; `DATETIME` stores as-is. `TIMESTAMP` has the 2038 limit.

```sql
CREATE TABLE events (
    id         INT PRIMARY KEY AUTO_INCREMENT,
    event_date DATE,
    start_time DATETIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Q6: What are the string data types in MySQL?

**Answer:**

| Type | Max Length | Use Case |
|------|------------|----------|
| VARCHAR | 65,535 | Short to medium strings |
| TEXT | 65,535 | Descriptions, short articles |
| MEDIUMTEXT | 16 MB | Blog posts |
| LONGTEXT | 4 GB | Books, large content |
| BLOB variants | Same sizes | Binary data |

```sql
CREATE TABLE content (
    title        VARCHAR(200),
    summary      TEXT,
    article_body LONGTEXT,
    metadata     JSON           -- MySQL 5.7+
);
```

---

## Indexes & Performance

### Q7: What are indexes and why are they important?

**Answer:** Indexes are data structures that speed up data retrieval at the cost of extra storage and slower writes. Think of them like a book's index — instead of scanning every page, you jump straight to the right one (O(log n) vs O(n)).

```sql
-- Primary Key (auto-created)
CREATE TABLE users (id INT PRIMARY KEY AUTO_INCREMENT, email VARCHAR(100));

-- Unique index
ALTER TABLE users ADD UNIQUE INDEX idx_email (email);

-- Composite index — follow left-prefix rule
CREATE INDEX idx_user_date ON orders(user_id, order_date);

-- Full-text index
CREATE FULLTEXT INDEX idx_content ON articles(title, body);
```

**Left-prefix rule for composite indexes:**
```sql
-- Index: (user_id, order_date, status)
WHERE user_id = 1 AND order_date = '2024-01-01'  -- uses index ✅
WHERE order_date = '2024-01-01'                  -- skips leading col, full scan ❌
```

### Q8: What is the difference between clustered and non-clustered indexes?

**Answer:** A **clustered index** stores the actual row data in index order — only one per table (InnoDB uses the primary key). A **non-clustered index** is a separate structure whose leaf nodes hold the index key plus a pointer (the primary key) back to the row, requiring a second lookup.

```sql
CREATE TABLE users (
    id       INT PRIMARY KEY,    -- clustered
    username VARCHAR(50),
    email    VARCHAR(100),
    INDEX idx_email (email)      -- non-clustered: lookup -> PK -> row
);
```

### Q9: Index best practices

**Answer:**

**DO:** index WHERE/JOIN/ORDER BY/GROUP BY columns; index foreign keys; use composite indexes for common multi-column queries; consider covering indexes.

**DON'T:** over-index (slows writes); index low-cardinality columns (boolean, gender); apply functions to indexed columns in WHERE clauses.

```sql
-- ✅ Covering index — query answered entirely from index
CREATE INDEX idx_order_covering ON orders(user_id, status, total);
SELECT user_id, status, total FROM orders WHERE user_id = 1;  -- no row lookup needed

-- Check index usage
SHOW INDEX FROM orders;
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

---

## Transactions & Concurrency

### Q10: What are ACID properties in MySQL?

**Answer:** ACID guarantees reliable transaction processing:

- **Atomicity** — all steps succeed or none do (no partial updates)
- **Consistency** — DB moves between valid states; constraints always satisfied
- **Isolation** — concurrent transactions don't interfere with each other
- **Durability** — committed data survives crashes

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;    -- both succeed
-- ROLLBACK;  -- or undo everything on error
```

### Q11: What are the transaction isolation levels in MySQL?

**Answer:**

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Notes |
|-------|-----------|---------------------|--------------|-------|
| Read Uncommitted | Yes | Yes | Yes | Lowest isolation |
| Read Committed | No | Yes | Yes | |
| Repeatable Read | No | No | Yes | **InnoDB default** |
| Serializable | No | No | No | Slowest |

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT @@transaction_isolation;
```

**Phantom read:** Transaction 1 runs the same SELECT twice; between the two reads, Transaction 2 inserts a matching row, so Transaction 1 sees a "phantom" extra row.

### Q12: What are the types of locks in MySQL?

**Answer:**

- **Shared lock (S)** — multiple readers allowed, blocks writers. `SELECT ... LOCK IN SHARE MODE`
- **Exclusive lock (X)** — one writer, blocks all others. `SELECT ... FOR UPDATE`, or any `UPDATE`/`DELETE`/`INSERT`
- Regular `SELECT` uses MVCC (no lock, reads a consistent snapshot)

```sql
START TRANSACTION;
SELECT * FROM products WHERE id = 1 FOR UPDATE;   -- X-lock
UPDATE products SET price = 100 WHERE id = 1;
COMMIT;
```

---

## SQL Queries & Joins

### Q13: What are the different types of JOINs in MySQL?

**Answer:**

```sql
-- INNER JOIN: rows matching in both tables
SELECT u.username, o.order_date
FROM users u INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all left rows + matched right (NULL if no match)
SELECT u.username, o.order_date
FROM users u LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN: all right rows + matched left
SELECT u.username, o.order_date
FROM users u RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL JOIN (MySQL workaround)
SELECT u.username, o.order_date FROM users u LEFT JOIN orders o ON u.id = o.user_id
UNION
SELECT u.username, o.order_date FROM users u RIGHT JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN: cartesian product (every combination)
SELECT u.username, p.name FROM users u CROSS JOIN products p;
```

### Q14: How do you optimize SQL queries?

**Answer:**

```sql
-- 1. EXPLAIN first — look for type:ALL (bad), key:NULL (no index used)
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- 2. Select only needed columns
SELECT id, username, email FROM users;  -- not SELECT *

-- 3. Avoid functions on indexed columns
-- ❌ BAD: index not used
SELECT * FROM users WHERE YEAR(created_at) = 2024;
-- ✅ GOOD: range uses index
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 4. Prefer JOIN over correlated subquery
-- ✅ GOOD
SELECT DISTINCT u.* FROM users u INNER JOIN orders o ON u.id = o.user_id WHERE o.total > 100;

-- 5. Use EXISTS instead of IN for large subqueries
SELECT * FROM users u WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 6. Anchor LIKE patterns at the start (trailing wildcard uses index)
SELECT * FROM products WHERE name LIKE 'widget%';  -- ✅ index works

-- 7. Batch INSERTs
INSERT INTO users (username) VALUES ('user1'), ('user2'), ('user3');

-- 8. UNION ALL vs UNION (skip dedup if not needed)
SELECT name FROM customers UNION ALL SELECT name FROM suppliers;
```

---

## MySQL Architecture

### Q15: What is the MySQL architecture?

**Answer:** MySQL is layered top to bottom:

1. **Connection Management** — client connections, authentication, thread handling
2. **SQL Interface** — parser (syntax check), optimizer (execution plan), query cache
3. **Pluggable Storage Engines** — InnoDB (default, transactional), MyISAM, Memory (swappable per table)
4. **File System** — `.ibd` files for InnoDB data, `.MYD`/`.MYI` for MyISAM

### Q16: How does MySQL handle query execution?

**Answer:** Parse → Optimize → Execute → Return results.

```sql
-- Inspect the execution plan
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
-- type: const/ref = good | ALL = full table scan (bad)
-- key: which index was chosen
-- rows: estimated rows examined
```

`type` values best to worst: `const > eq_ref > ref > range > index > ALL`.

---

## MySQL Security

### Q17: What are the MySQL security best practices?

**Answer:**

```sql
-- Least privilege: grant only what's needed
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user'@'localhost';

-- Review and revoke
SHOW GRANTS FOR 'app_user'@'localhost';
REVOKE DELETE ON app_db.* FROM 'app_user'@'localhost';

-- Restrict host, require SSL
CREATE USER 'secure_user'@'192.168.1.100' IDENTIFIED BY 'password' REQUIRE SSL;

-- Backup
mysqldump --single-transaction --quick --lock-tables=false app_db > backup.sql
```

Never use `GRANT ALL PRIVILEGES ON *.* TO ...` for application accounts. Use prepared statements in application code to prevent SQL injection.

---

## MySQL vs Other Databases

### Q18: MySQL vs PostgreSQL

| Feature | MySQL | PostgreSQL |
|---------|-------|------------|
| ACID | Full (InnoDB) | Full |
| JSON | Basic | Advanced (JSONB) |
| Complex Queries / Window Functions | Good | Excellent |
| Stored Procedures | Basic | Advanced (PL/pgSQL) |
| Performance | Read-optimized | Write-optimized |
| Extensions | Limited | Rich ecosystem |

Use MySQL for LAMP-stack web apps with read-heavy workloads. Use PostgreSQL for complex data models, advanced features (arrays, JSONB), or analytical queries.

### Q19: MySQL vs MongoDB

| Feature | MySQL | MongoDB |
|---------|-------|---------|
| Data Model | Tables, fixed schema | Documents, flexible schema |
| Transactions | ACID | ACID (since 4.0) |
| Scalability | Vertical + read replicas | Horizontal sharding |
| Schema Changes | `ALTER TABLE` (expensive) | None needed |
| Joins | Native | `$lookup` (less natural) |

Use MySQL when data is structured and relational with strict integrity requirements. Use MongoDB when the schema evolves rapidly or data is naturally document-shaped.

---

## Common Mistakes

### Mistake 1: Wrong data types

```sql
-- ❌ VARCHAR for IDs, FLOAT for money
-- ✅ INT for IDs, DECIMAL for money
CREATE TABLE orders (
    id     INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    amount DECIMAL(10,2)
);
```

### Mistake 2: No index on foreign keys

```sql
CREATE TABLE orders (
    id      INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id)   -- ✅ always index FK columns
);
```

### Mistake 3: SELECT *

```sql
SELECT id, username, email FROM users;  -- ✅ only fetch what you need
```

### Mistake 4: Multi-step updates without a transaction

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### Mistake 5: Not handling NULLs

```sql
-- NULL != 'user@example.com' — use IS NULL check or COALESCE
SELECT * FROM users WHERE COALESCE(email, '') = 'user@example.com';
```

### Mistake 6: No prepared statements (SQL injection)

```sql
-- ✅ GOOD (PHP example)
$stmt = $mysqli->prepare("SELECT * FROM users WHERE email = ?");
$stmt->bind_param("s", $email);
$stmt->execute();
```

---

## Quick Reference Cheat Sheet

```sql
-- Indexes
CREATE INDEX idx_name ON table(column);
CREATE UNIQUE INDEX idx_name ON table(column);
SHOW INDEX FROM table;
EXPLAIN SELECT * FROM table WHERE condition;

-- Transactions
START TRANSACTION;
-- statements
COMMIT;   -- or ROLLBACK;

-- Isolation level
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT @@transaction_isolation;

-- Slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;

-- Backup
mysqldump -u root -p database > backup.sql
```

**Interview must-knows:**

1. **InnoDB vs MyISAM** — InnoDB: transactions + row-level locking. MyISAM: neither.
2. **Clustered index** — one per table (the PK), stores rows in index order.
3. **ACID** — Atomicity, Consistency, Isolation, Durability.
4. **Default isolation** — Repeatable Read (InnoDB).
5. **Optimization** — `EXPLAIN` first, index WHERE/JOIN/FK columns, avoid `SELECT *`, no functions on indexed columns.
6. **Data types** — smallest sufficient type; `DECIMAL` (not `FLOAT`) for money.

*Last Updated: 2026-06-18*
