# MySQL Interview Questions & Answers

## MySQL Basics

### Q1: What is MySQL and why is it used?

**Answer:** MySQL is an open-source relational database management system (RDBMS) that uses SQL (Structured Query Language) to manage data. It's used because:

- **Open Source**: Free and community-supported with enterprise options
- **Reliability**: Proven track record with major companies (Facebook, Google, Netflix)
- **Performance**: Optimized for read-heavy workloads
- **Scalability**: Supports replication and clustering for horizontal scaling
- **Easy to Use**: Simple installation and management
- **Large Community**: Extensive documentation and third-party tools
- **ACID Compliant**: Ensures data integrity with transactions

### Q2: What are the different MySQL storage engines?

**Answer:**

**InnoDB (Default since MySQL 5.5):**
- ACID compliant with full transaction support
- Row-level locking for high concurrency
- Foreign key constraints
- Crash recovery capabilities
- Best for: Most applications requiring data integrity

**MyISAM (Legacy):**
- Table-level locking (lower concurrency)
- No transaction support
- Faster for read-heavy, write-light workloads
- Full-text search (before InnoDB supported it)
- Best for: Read-only data, historical data

**Memory:**
- Stores data in RAM
- Very fast but data is lost on restart
- Table-level locking
- Best for: Temporary data, caching, lookup tables

**Others (awareness):** CSV (CSV files, import/export), Archive (compressed, INSERT/SELECT only for logs).

**Example: Creating tables with different engines**

```sql
-- InnoDB (default, most common)
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50),
    email VARCHAR(100)
) ENGINE=InnoDB;

-- MyISAM (read-heavy, no transactions needed)
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
) ENGINE=MyISAM;

-- Memory (temporary data)
CREATE TABLE session_cache (
    session_id VARCHAR(50) PRIMARY KEY,
    user_data TEXT,
    created_at TIMESTAMP
) ENGINE=MEMORY;
```

### Q3: What is the difference between CHAR and VARCHAR?

**Answer:**

**CHAR:**
- Fixed-length string
- Always uses specified length (pads with spaces)
- Faster for frequent updates (no row movement)
- Best for: Fixed-length data like country codes, phone numbers

**VARCHAR:**
- Variable-length string
- Uses only actual length + 1-2 bytes for length
- More space-efficient for variable data
- Best for: Names, descriptions, emails

**Example:**

```sql
-- CHAR always uses 10 bytes
CREATE TABLE fixed (
    country_code CHAR(2)  -- Always 2 bytes
);

-- VARCHAR uses actual length
CREATE TABLE variable (
    email VARCHAR(100)    -- Uses 1-101 bytes depending on content
);

-- Storage comparison
INSERT INTO fixed VALUES ('US');     -- Uses 2 bytes, padded to 2
INSERT INTO variable VALUES ('a@b.c');  -- Uses 5 bytes (3 + 2 length bytes)
```

**Performance Impact:**
- CHAR can be faster for fixed data (no length calculation)
- VARCHAR saves space for variable-length data
- Choose based on data patterns, not just space

---

## Data Types & Storage

### Q4: What are the different numeric data types in MySQL?

**Answer:**

**Integer Types:**

| Type | Storage | Range (Signed) | Range (Unsigned) | Use Case |
|------|---------|----------------|------------------|----------|
| TINYINT | 1 byte | -128 to 127 | 0 to 255 | Boolean flags, small counters |
| SMALLINT | 2 bytes | -32,768 to 32,767 | 0 to 65,535 | Small IDs, years |
| MEDIUMINT | 3 bytes | -8M to 8M | 0 to 16M | Medium IDs |
| INT | 4 bytes | -2B to 2B | 0 to 4B | Standard IDs |
| BIGINT | 8 bytes | -9 quintillion to 9 quintillion | 0 to 18 quintillion | Large IDs, timestamps |

**Floating Point Types:**
```sql
-- FLOAT: Single precision, 4 bytes
CREATE TABLE products (
    price FLOAT(7,2)  -- 7 total digits, 2 after decimal
);

-- DOUBLE: Double precision, 8 bytes
CREATE TABLE scientific_data (
    measurement DOUBLE(15,8)  -- More precise
);

-- DECIMAL: Exact decimal, user-defined precision
CREATE TABLE financial (
    amount DECIMAL(15,2)  -- Exact for money
    -- Uses storage: (precision/2) + 1 bytes
);
```

**Example: Choosing the right type**

```sql
-- ❌ BAD: Using INT for boolean
CREATE TABLE users (
    is_active INT  -- Wastes 4 bytes
);

-- ✅ GOOD: Using TINYINT for boolean
CREATE TABLE users (
    is_active TINYINT(1)  -- Uses 1 byte
);

-- ❌ BAD: Using FLOAT for money
CREATE TABLE transactions (
    amount FLOAT  -- Floating point precision issues!
);

-- ✅ GOOD: Using DECIMAL for money
CREATE TABLE transactions (
    amount DECIMAL(10,2)  -- Exact precision
);
```

### Q5: What are the temporal data types in MySQL?

**Answer:**

**Date/Time Types:**

| Type | Storage | Range | Format | Use Case |
|------|---------|-------|--------|----------|
| DATE | 3 bytes | 1000-01-01 to 9999-12-31 | YYYY-MM-DD | Birthdays, events |
| TIME | 3 bytes | -838:59:59 to 838:59:59 | HH:MM:SS | Duration, time of day |
| DATETIME | 8 bytes | 1000-01-01 00:00:00 to 9999-12-31 23:59:59 | YYYY-MM-DD HH:MM:SS | Specific moments |
| TIMESTAMP | 4 bytes | 1970-01-01 00:00:01 UTC to 2038-01-19 03:14:07 UTC | YYYY-MM-DD HH:MM:SS | Auto-updating timestamps |
| YEAR | 1 byte | 1901 to 2155 | YYYY | Years only |

**Example Usage:**

```sql
CREATE TABLE events (
    id INT PRIMARY KEY AUTO_INCREMENT,
    event_name VARCHAR(100),
    event_date DATE,              -- For calendar dates
    start_time DATETIME,          -- Specific moment (timezone-aware)
    duration TIME,                -- Duration of event
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- Auto-set
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    year_column YEAR              -- Just the year
);

-- Inserting dates
INSERT INTO events (event_name, event_date, start_time, duration)
VALUES ('Conference', '2024-12-25', '2024-12-25 09:00:00', '08:00:00');

-- Date functions
SELECT
    event_name,
    event_date,
    YEAR(event_date) as year,
    MONTH(event_date) as month,
    DAY(event_date) as day,
    DATEDIFF(NOW(), event_date) as days_since
FROM events;
```

**Important: TIMESTAMP vs DATETIME**
- TIMESTAMP: Converted to UTC for storage, converted back for retrieval (timezone-aware)
- DATETIME: Stored as-is, no timezone conversion
- TIMESTAMP: Limited range (2038 problem)
- DATETIME: Wider range but uses more storage

### Q6: What are the string data types in MySQL?

**Answer:**

**Text Types:**

| Type | Max Length | Storage | Use Case |
|------|------------|---------|----------|
| CHAR | 255 | Fixed | Fixed-length data |
| VARCHAR | 65,535 | Variable | Variable-length strings |
| TINYTEXT | 255 | Variable + 1 byte | Very short text |
| TEXT | 65,535 | Variable + 2 bytes | Short articles, descriptions |
| MEDIUMTEXT | 16M | Variable + 3 bytes | Articles, blog posts |
| LONGTEXT | 4GB | Variable + 4 bytes | Books, large content |

**Binary Types:**
- BINARY, VARBINARY: Binary strings
- BLOB types: TINYBLOB, BLOB, MEDIUMBLOB, LONGBLOB

**Example:**

```sql
CREATE TABLE content (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(200),           -- Short strings
    summary TEXT,                 -- Medium text
    article_body LONGTEXT,        -- Long articles
    thumbnail BLOB,               -- Binary data
    metadata JSON                 -- JSON data (MySQL 5.7+)
);

-- String functions
SELECT
    title,
    LENGTH(title) as length,
    UPPER(title) as uppercase,
    SUBSTRING(title, 1, 10) as first_10_chars,
    CONCAT(title, ' - Summary') as combined
FROM content;
```

---

## Indexes & Performance

### Q7: What are indexes and why are they important?

**Answer:** Indexes are data structures that improve the speed of data retrieval operations on database tables at the cost of additional writes and storage space.

**Real-world analogy:**
- **Without index**: Like searching through an unsorted phone book page by page
- **With index**: Like searching through a phone book with tabs organized alphabetically

**Types of Indexes:**

```sql
-- Primary Key Index (automatically created)
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50)
);

-- Unique Index
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    UNIQUE INDEX idx_email (email)
);

-- Regular Index
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    category_id INT,
    INDEX idx_category (category_id)
);

-- Composite Index (multiple columns)
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    order_date DATE,
    status VARCHAR(20),
    INDEX idx_user_date_status (user_id, order_date, status)
);

-- Full-text Index (for text search)
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    body TEXT,
    FULLTEXT INDEX idx_content (title, body)
);
```

**Index Performance Impact:**

```sql
-- Without index (slow - full table scan)
SELECT * FROM users WHERE email = 'user@example.com';  -- O(n)

-- With index (fast - index lookup)
SELECT * FROM users WHERE email = 'user@example.com';  -- O(log n)

-- Composite index usage
-- Index: (user_id, order_date, status)
SELECT * FROM orders
WHERE user_id = 1 AND order_date = '2024-01-01' AND status = 'completed';  -- Uses all columns

SELECT * FROM orders
WHERE user_id = 1 AND order_date = '2024-01-01';  -- Uses first two columns

SELECT * FROM orders
WHERE user_id = 1;  -- Uses first column only

-- ❌ Won't use index efficiently (missing leading column)
SELECT * FROM orders
WHERE order_date = '2024-01-01';  -- Full scan or index scan
```

### Q8: What is the difference between clustered and non-clustered indexes?

**Answer:**

**Clustered Index:**
- Data rows are stored in the order of the index
- Only one per table (usually the primary key)
- Faster for range queries
- InnoDB: Primary key is clustered
- MyISAM: No clustered index (heap storage)

**Non-Clustered Index:**
- Separate structure from data
- Multiple per table allowed
- Contains pointer to actual data
- Slower for range queries but more flexible

In short: a clustered index leaf node holds the actual row data, while a non-clustered index leaf node holds the index key plus a pointer (the primary key) back to the row.

**Example:**

```sql
-- InnoDB: Primary key is clustered
CREATE TABLE users (
    id INT PRIMARY KEY,           -- Clustered index
    username VARCHAR(50),
    email VARCHAR(100),
    INDEX idx_username (username),  -- Non-clustered index
    INDEX idx_email (email)         -- Non-clustered index
);

-- Query using clustered index (fastest)
SELECT * FROM users WHERE id = 1;

-- Query using non-clustered index
-- 1. Find email in idx_email
-- 2. Get primary key from leaf node
-- 3. Look up row in clustered index using primary key
SELECT * FROM users WHERE email = 'user@example.com';
```

### Q9: What are the best practices for creating indexes?

**Answer:**

**DO:**
- Index columns used in WHERE, JOIN, ORDER BY, GROUP BY clauses
- Create composite indexes for frequently queried column combinations
- Index foreign key columns (improves JOIN performance)
- Consider covering indexes for frequently accessed columns

**DON'T:**
- Over-index (slows down INSERT/UPDATE/DELETE)
- Index low-cardinality columns (like boolean, gender)
- Index columns that are frequently updated
- Create indexes that won't be used

**Examples:**

```sql
-- ✅ GOOD: Index columns used in WHERE clause
CREATE INDEX idx_user_email ON users(email);

-- ✅ GOOD: Composite index for multiple conditions
CREATE INDEX idx_user_status_date ON users(status, created_at);

-- ✅ GOOD: Index foreign keys
CREATE INDEX idx_order_user ON orders(user_id);

-- ❌ BAD: Indexing low-cardinality column
CREATE INDEX idx_user_active ON users(is_active);  -- Only 2 values (true/false)

-- ❌ BAD: Indexing frequently updated column
CREATE INDEX idx_user_balance ON users(balance);  -- Balance changes often

-- ✅ GOOD: Covering index (includes all queried columns)
CREATE INDEX idx_order_covering ON orders(user_id, status, total);
-- This query can be satisfied entirely from the index:
SELECT user_id, status, total FROM orders WHERE user_id = 1;
```

**Index Analysis:**

```sql
-- Check which indexes exist
SHOW INDEX FROM users;

-- Analyze index usage (type: const, key: idx_email, rows: 1 = good)
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

---

## Transactions & Concurrency

### Q10: What are ACID properties in MySQL?

**Answer:** ACID is a set of properties that guarantee database transactions are processed reliably:

**Atomicity:**
- All operations in a transaction succeed or none do
- No partial updates

**Consistency:**
- Database moves from one valid state to another
- All constraints are maintained

**Isolation:**
- Concurrent transactions don't interfere with each other
- Each transaction sees a consistent view of data

**Durability:**
- Once committed, changes are permanent
- Survives system failures

**Example:**

```sql
-- Transaction example (bank transfer)
START TRANSACTION;

-- Step 1: Deduct from sender
UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

-- Step 2: Add to receiver
UPDATE accounts
SET balance = balance + 100
WHERE id = 2;

-- Step 3: Record transaction
INSERT INTO transactions (sender_id, receiver_id, amount)
VALUES (1, 2, 100);

-- If any step fails, rollback all changes
-- If all succeed, commit
COMMIT;

-- Or if error occurs:
ROLLBACK;
```

### Q11: What are the different transaction isolation levels in MySQL?

**Answer:**

**Read Uncommitted:**
- Lowest isolation
- Can read uncommitted changes from other transactions
- Dirty reads, non-repeatable reads, phantom reads possible

**Read Committed (Default in some DBs):**
- Only reads committed changes
- No dirty reads
- Non-repeatable reads, phantom reads possible

**Repeatable Read (Default in InnoDB):**
- Sees consistent snapshot throughout transaction
- No dirty reads, no non-repeatable reads
- Phantom reads possible

**Serializable:**
- Highest isolation
- Transactions are completely isolated
- No dirty reads, non-repeatable reads, or phantom reads
- Lowest performance

**Example:**

```sql
-- Set isolation level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check current isolation level
SELECT @@transaction_isolation;

-- Example of phantom read issue
-- Transaction 1:
START TRANSACTION;
SELECT * FROM products WHERE price > 100;  -- Returns 10 rows
-- Transaction 2 inserts a product with price 200 and commits
-- Transaction 1:
SELECT * FROM products WHERE price > 100;  -- Now returns 11 rows (phantom)
COMMIT;
```

### Q12: What are the different types of locks in MySQL?

**Answer:**

**Lock Types:**

**Shared Lock (S-lock):**
- Allows multiple transactions to read data
- Prevents writes during read
- Used in SELECT ... LOCK IN SHARE MODE

**Exclusive Lock (X-lock):**
- Allows only one transaction to write data
- Prevents both reads and writes
- Used in UPDATE, DELETE, INSERT

**Intention & Gap locks (awareness):** Intention locks (IS/IX) signal intent to lock rows at the table level; gap locks lock the gap between index records to prevent phantom reads in Repeatable Read.

**Example:**

```sql
-- Explicit locking
START TRANSACTION;

-- Shared lock (other transactions can also read)
SELECT * FROM products WHERE id = 1 LOCK IN SHARE MODE;

-- Exclusive lock (other transactions blocked)
SELECT * FROM products WHERE id = 1 FOR UPDATE;

-- Regular SELECT (no lock, uses MVCC)
SELECT * FROM products WHERE id = 1;

-- UPDATE automatically acquires exclusive lock
UPDATE products SET price = 100 WHERE id = 1;

COMMIT;

-- Check locks
SHOW ENGINE INNODB STATUS;
```

---

## SQL Queries & Joins

### Q13: What are the different types of JOINs in MySQL?

**Answer:**

**INNER JOIN:**
- Returns rows when there's a match in both tables
- Most common type

```sql
SELECT users.username, orders.order_date, orders.total
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

**LEFT JOIN (LEFT OUTER JOIN):**
- Returns all rows from left table, matching rows from right
- NULL for non-matching right table rows

```sql
SELECT users.username, orders.order_date
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
-- Returns all users, even those without orders
```

**RIGHT JOIN (RIGHT OUTER JOIN):**
- Returns all rows from right table, matching rows from left
- NULL for non-matching left table rows

```sql
SELECT users.username, orders.order_date
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
-- Returns all orders, even without matching users (unlikely with good FK)
```

**FULL JOIN (not directly supported in MySQL):**
- Use UNION of LEFT JOIN and RIGHT JOIN

```sql
SELECT users.username, orders.order_date
FROM users
LEFT JOIN orders ON users.id = orders.user_id
UNION
SELECT users.username, orders.order_date
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```

**CROSS JOIN:**
- Cartesian product of all rows
- Rarely used intentionally

```sql
SELECT users.username, products.name
FROM users
CROSS JOIN products;
-- Returns every combination of users and products
```

**Example with multiple joins:**

```sql
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

### Q14: How do you optimize SQL queries?

**Answer:**

**Query Optimization Techniques:**

```sql
-- 1. Use EXPLAIN to analyze query plan
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Look for:
-- - type: 'const' or 'ref' is good, 'ALL' is bad (full scan)
-- - key: which index is used
-- - rows: estimated rows to examine
-- - Extra: 'Using index' is good (covering index)

-- 2. Avoid SELECT *
-- ❌ BAD
SELECT * FROM users;

-- ✅ GOOD
SELECT id, username, email FROM users;

-- 3. Use LIMIT for large result sets
SELECT * FROM large_table LIMIT 1000;

-- 4. Use appropriate WHERE clauses
-- ❌ BAD: Function on column prevents index use
SELECT * FROM users WHERE YEAR(created_at) = 2024;

-- ✅ GOOD: Range comparison uses index
SELECT * FROM users
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 5. Use JOIN instead of subqueries when possible
-- ❌ BAD: Subquery executed for each row
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 100);

-- ✅ GOOD: JOIN is more efficient
SELECT DISTINCT users.*
FROM users
INNER JOIN orders ON users.id = orders.user_id
WHERE orders.total > 100;

-- 6. Use EXISTS instead of IN for subqueries
-- ❌ BAD: IN can be slow with large lists
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- ✅ GOOD: EXISTS can be faster (stops at first match)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 7. Avoid wildcard at start of LIKE pattern
-- ❌ BAD: Can't use index
SELECT * FROM products WHERE name LIKE '%widget%';

-- ✅ GOOD: Can use index prefix
SELECT * FROM products WHERE name LIKE 'widget%';

-- 8. Use UNION ALL instead of UNION if you don't need deduplication
-- ❌ BAD: UNION performs deduplication (slower)
SELECT name FROM customers
UNION
SELECT name FROM suppliers;

-- ✅ GOOD: UNION ALL is faster
SELECT name FROM customers
UNION ALL
SELECT name FROM suppliers;

-- 9. Batch INSERT operations
-- ❌ BAD: Multiple individual INSERTs
INSERT INTO users (username) VALUES ('user1');
INSERT INTO users (username) VALUES ('user2');
INSERT INTO users (username) VALUES ('user3');

-- ✅ GOOD: Single INSERT with multiple values
INSERT INTO users (username) VALUES ('user1'), ('user2'), ('user3');

-- 10. Use appropriate data types
-- ❌ BAD: Using VARCHAR for IDs
CREATE TABLE orders (
    id VARCHAR(50) PRIMARY KEY
);

-- ✅ GOOD: Using INT for IDs
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT
);
```

---

## Stored Procedures & Functions

### Q15: What are stored procedures and when should you use them?

**Answer:** Stored procedures are precompiled SQL statements stored in the database that can be executed with a single call.

**Benefits:**
- Reduced network traffic
- Improved performance (precompiled)
- Enhanced security (parameterized queries)
- Code reuse and modularity
- Transaction management

**When to use:**
- Complex business logic
- Repeated operations
- Security requirements (hide table structure)
- Performance-critical operations

**Example:**

```sql
-- Create stored procedure
DELIMITER //

CREATE PROCEDURE GetUserOrders(IN user_id INT, IN start_date DATE, IN end_date DATE)
BEGIN
    SELECT
        o.id as order_id,
        o.order_date,
        o.total,
        COUNT(oi.id) as item_count
    FROM orders o
    LEFT JOIN order_items oi ON o.id = oi.order_id
    WHERE o.user_id = user_id
    AND o.order_date BETWEEN start_date AND end_date
    GROUP BY o.id
    ORDER BY o.order_date DESC;
END //

DELIMITER ;

-- Call stored procedure
CALL GetUserOrders(1, '2024-01-01', '2024-12-31');

-- With parameters
CALL GetUserOrders(5, '2024-06-01', '2024-06-30');

-- Drop procedure
DROP PROCEDURE IF EXISTS GetUserOrders;
```

**Stored Procedure with Output Parameters:**

```sql
DELIMITER //

CREATE PROCEDURE GetUserSummary(
    IN user_id INT,
    OUT total_orders INT,
    OUT total_spent DECIMAL(10,2),
    OUT last_order_date DATE
)
BEGIN
    SELECT
        COUNT(*),
        COALESCE(SUM(total), 0),
        MAX(order_date)
    INTO total_orders, total_spent, last_order_date
    FROM orders
    WHERE user_id = user_id;
END //

DELIMITER ;

-- Call with output parameters
CALL GetUserSummary(1, @orders, @spent, @last_order);

-- Use output variables
SELECT @orders as total_orders, @spent as total_spent, @last_order as last_order_date;
```

### Q16: What are user-defined functions in MySQL?

**Answer:** User-defined functions (UDFs) are custom functions that return a single value and can be used in SQL statements.

**Types:**
- Scalar functions: Return single value
- Aggregate functions: Operate on sets of values (more complex)

**Example:**

```sql
-- Create scalar function
DELIMITER //

CREATE FUNCTION CalculateAge(birth_date DATE)
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE age INT;
    SET age = TIMESTAMPDIFF(YEAR, birth_date, CURDATE());
    RETURN age;
END //

DELIMITER ;

-- Use function in query
SELECT
    username,
    birth_date,
    CalculateAge(birth_date) as age
FROM users
WHERE CalculateAge(birth_date) >= 18;

-- Create function with business logic
DELIMITER //

CREATE FUNCTION CalculateDiscount(total_amount DECIMAL(10,2), customer_level VARCHAR(20))
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE discount DECIMAL(10,2);

    SET discount = 0;

    IF customer_level = 'gold' THEN
        SET discount = total_amount * 0.15;
    ELSEIF customer_level = 'silver' THEN
        SET discount = total_amount * 0.10;
    ELSEIF customer_level = 'bronze' THEN
        SET discount = total_amount * 0.05;
    END IF;

    RETURN discount;
END //

DELIMITER ;

-- Use in query
SELECT
    o.id,
    o.total,
    u.customer_level,
    CalculateDiscount(o.total, u.customer_level) as discount_amount,
    o.total - CalculateDiscount(o.total, u.customer_level) as final_amount
FROM orders o
JOIN users u ON o.user_id = u.id;
```

---

## MySQL Optimization

### Q17: How do you optimize MySQL database performance?

**Answer:**

**Schema Optimization:**

```sql
-- 1. Choose appropriate data types
-- Use smallest sufficient type
CREATE TABLE users (
    id INT UNSIGNED,           -- Use UNSIGNED if no negative values
    status TINYINT(1),         -- Use TINYINT for boolean
    country_code CHAR(2),      -- Use CHAR for fixed-length
    zip_code VARCHAR(10)       -- Use VARCHAR for variable-length
);

-- 2. Normalize to reduce redundancy
-- First Normal Form (1NF): No repeating groups
-- Second Normal Form (2NF): No partial dependencies
-- Third Normal Form (3NF): No transitive dependencies

-- 3. Denormalize for read-heavy workloads
CREATE TABLE order_summary (
    user_id INT PRIMARY KEY,
    total_orders INT,
    total_spent DECIMAL(10,2),
    last_order_date DATE,
    INDEX idx_user_id (user_id)
);

-- 4. Partitioning (awareness): split very large tables into partitions
--    (e.g. PARTITION BY RANGE on year) so queries scan fewer rows.
--    This is mostly a DBA-level concern.
```

**Query Optimization:**

```sql
-- 1. Use EXPLAIN to analyze queries
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

-- 2. Create appropriate indexes
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- 3. Use covering indexes
CREATE INDEX idx_orders_covering ON orders(user_id, status, total);

-- 4. Optimize JOINs
-- Ensure JOIN columns are indexed
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);

-- 5. Use query cache (MySQL 5.7 and earlier)
SELECT SQL_CACHE * FROM products WHERE id = 1;

-- 6. Use LIMIT for pagination
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;  -- Page 1

-- Better pagination for large offsets (keyset pagination)
SELECT * FROM products
WHERE id > last_seen_id
ORDER BY id
LIMIT 20;
```

**Configuration Optimization (awareness):**

Server tuning lives in `my.cnf`/`my.ini` and is mostly a DBA concern. The key knob is `innodb_buffer_pool_size` (set to ~70-80% of RAM so data/indexes stay cached); others include `max_connections` and `innodb_log_file_size`. Inspect current values with `SHOW VARIABLES LIKE 'innodb_buffer_pool_size';`.

### Q18: What are the common MySQL performance issues and solutions?

**Answer:** (awareness — mostly DBA-level diagnosis)

| Issue | How to spot it | Fix |
|-------|----------------|-----|
| Slow queries | Slow query log (`long_query_time`), `mysqldumpslow` | Add indexes, rewrite queries |
| Lock contention | `SHOW ENGINE INNODB STATUS` | Keep transactions short, right isolation level |
| Poor index usage | `EXPLAIN` shows `type: ALL` | Add/composite indexes, no functions on indexed cols |
| Table fragmentation | `data_free` in `information_schema.tables` | `OPTIMIZE TABLE` |
| Connection exhaustion | `Threads_connected` status | App-side connection pooling, raise `max_connections` |

```sql
-- Enable the slow query log to find the worst offenders
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- Log queries taking > 2 seconds
```

---

## MySQL Architecture

### Q19: What is the MySQL architecture?

**Answer:** MySQL is layered, top to bottom:

1. **Connection Management** — client connections, authentication, thread handling.
2. **SQL Interface** — parser (validates syntax), optimizer (builds the execution plan), and query cache.
3. **Pluggable Storage Engines** — InnoDB (default, transactional), MyISAM (non-transactional, faster reads), Memory, etc. The engine is swappable per table.
4. **File System** — actual data and log files (e.g. `.ibd` for InnoDB data, `.MYD`/`.MYI` for MyISAM data/indexes).

### Q20: How does MySQL handle query execution?

**Answer:**

**Query Execution Steps:**

```sql
-- When you execute: SELECT * FROM users WHERE id = 1;

-- Step 1: Connection
-- - Client connects to MySQL
-- - Authentication
-- - Thread assigned

-- Step 2: Parsing
-- - SQL syntax validation
-- - Parse tree created

-- Step 3: Optimization
-- - Query optimizer analyzes
-- - Creates execution plan
-- - Chooses best access method (index vs full scan)

-- Step 4: Execution
-- - Storage engine executes
-- - Retrieves data
-- - Applies filters

-- Step 5: Result Return
-- - Results formatted
-- - Returned to client
```

**View Execution Plan:**

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Key columns to read: key (index used), rows (estimated rows), type (access method).
-- type, best to worst: const > eq_ref > ref > range > index > ALL (full scan, worst).
```

---

## MySQL Security

### Q21: What are the MySQL security best practices?

**Answer:**

**User Management:**

```sql
-- 1. Create users with minimal privileges
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'strong_password';

-- 2. Grant specific privileges only
GRANT SELECT, INSERT, UPDATE ON app_db.* TO 'app_user'@'localhost';

-- 3. Avoid using GRANT ALL PRIVILEGES
-- ❌ BAD
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'localhost';

-- 4. Use principle of least privilege
-- Read-only user
CREATE USER 'read_only'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT ON app_db.* TO 'read_only'@'localhost';

-- 5. Regularly review privileges
SHOW GRANTS FOR 'app_user'@'localhost';

-- 6. Revoke unnecessary privileges
REVOKE DELETE ON app_db.* FROM 'app_user'@'localhost';
```

**Connection Security:**

```sql
-- 1. Restrict host access
-- Only allow connections from specific hosts
CREATE USER 'app_user'@'192.168.1.100' IDENTIFIED BY 'password';

-- 2. Use SSL/TLS for connections
-- Require SSL for specific user
CREATE USER 'secure_user'@'%' IDENTIFIED BY 'password' REQUIRE SSL;

-- 3. Disable remote root access
-- Remove root remote access
DROP USER 'root'@'%';
-- Keep only local root
```

**Data Security:**

```sql
-- 1. Encrypt sensitive data
-- Use AES_ENCRYPT for sensitive columns
INSERT INTO users (username, password_hash, ssn)
VALUES ('user1', SHA2('password', 256), AES_ENCRYPT('123-45-6789', 'encryption_key'));

-- 2. Use stored procedures to hide table structure
CREATE PROCEDURE GetUserProfile(IN user_id INT)
BEGIN
    SELECT username, email, created_at FROM users WHERE id = user_id;
END;

-- 3. Regular backups
-- Logical backup
mysqldump -u root -p app_db > backup.sql

-- Physical backup (for InnoDB)
mysqldump --single-transaction --quick --lock-tables=false app_db > backup.sql
```

---

## MySQL vs Other Databases

### Q22: What are the differences between MySQL and PostgreSQL?

**Answer:**

| Feature | MySQL | PostgreSQL |
|---------|-------|------------|
| License | GPL (dual license) | PostgreSQL License (more permissive) |
| Default Storage Engine | InnoDB | Single engine (advanced) |
| ACID Compliance | Full (InnoDB) | Full |
| JSON Support | Basic | Advanced (JSONB) |
| Full-Text Search | Good | Excellent |
| Complex Queries | Good | Excellent |
| Window Functions | Basic (improving) | Excellent |
| Stored Procedures | Basic | Advanced (PL/pgSQL) |
| Extensions | Limited | Rich ecosystem |
| Replication | Master-slave, group replication | Master-slave, logical replication |
| Performance | Read-optimized | Write-optimized |
| Community | Very large | Large and active |

**When to choose MySQL:**
- Web applications with read-heavy workloads
- Need for simple setup and management
- Large community and resources
- LAMP stack compatibility

**When to choose PostgreSQL:**
- Complex data models and relationships
- Need for advanced features (JSON, arrays, etc.)
- Complex analytical queries
- Need for extensibility

### Q23: What are the differences between MySQL and MongoDB?

**Answer:**

| Feature | MySQL (Relational) | MongoDB (NoSQL) |
|---------|-------------------|-----------------|
| Data Model | Structured, tables | Document-based, collections |
| Schema | Fixed, defined upfront | Flexible, dynamic |
| Query Language | SQL | MongoDB Query Language |
| Transactions | ACID compliant | ACID compliant (since 4.0) |
| Scalability | Vertical scaling, read replicas | Horizontal sharding |
| Joins | Supported | Supported (lookup, $lookup) |
| Relationships | Foreign keys, normalized | Embedded documents, references |
| Indexing | B-tree indexes | B-tree, geospatial, text indexes |
| Aggregation | GROUP BY, HAVING | Aggregation pipeline |
| Schema Changes | ALTER TABLE (expensive) | No schema changes needed |

**When to choose MySQL:**
- Structured data with clear relationships
- Complex transactions required
- Data integrity is critical
- ACID compliance essential
- Standardized reporting needed

**When to choose MongoDB:**
- Rapidly evolving schema
- Document-oriented data
- Horizontal scaling required
- Flexible data structures
- Real-time analytics

---

## Common Mistakes

### Mistake 1: Not using appropriate data types

```sql
-- ❌ BAD: Using VARCHAR for IDs
CREATE TABLE orders (
    id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50)
);

-- ✅ GOOD: Using appropriate integer types
CREATE TABLE orders (
    INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    user_id INT UNSIGNED NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Mistake 2: Not creating indexes on foreign keys

```sql
-- ❌ BAD: Foreign key without index
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- ✅ GOOD: Indexed foreign key
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id)
);
```

### Mistake 3: Using SELECT *

```sql
-- ❌ BAD: Retrieves all columns
SELECT * FROM users;

-- ✅ GOOD: Retrieves only needed columns
SELECT id, username, email FROM users;
```

### Mistake 4: Not using transactions for multi-step operations

```sql
-- ❌ BAD: No transaction
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- If second UPDATE fails, first is committed

-- ✅ GOOD: With transaction
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### Mistake 5: Not handling NULL values properly

```sql
-- ❌ BAD: Assumes no NULL values
SELECT * FROM users WHERE email = 'user@example.com';
-- Won't find users with NULL email

-- ✅ GOOD: Handles NULL values
SELECT * FROM users
WHERE email = 'user@example.com' OR email IS NULL;

-- Or use COALESCE
SELECT * FROM users
WHERE COALESCE(email, '') = 'user@example.com';
```

### Mistake 6: Not using prepared statements

```sql
-- ❌ BAD: SQL injection risk
$sql = "SELECT * FROM users WHERE email = '" . $email . "'";
$result = $mysqli->query($sql);

-- ✅ GOOD: Prepared statement
$stmt = $mysqli->prepare("SELECT * FROM users WHERE email = ?");
$stmt->bind_param("s", $email);
$stmt->execute();
$result = $stmt->get_result();
```

---

## Short Revision Summary

### Quick Reference Cheat Sheet

```sql
-- Create index
CREATE INDEX idx_name ON table(column);
CREATE UNIQUE INDEX idx_name ON table(column);

-- Transaction
START TRANSACTION;
-- SQL statements
COMMIT; -- or ROLLBACK;

-- Explain a query
EXPLAIN SELECT * FROM table WHERE condition;

-- Backup
mysqldump -u root -p database > backup.sql
```

**Interview must-knows:**

1. **InnoDB vs MyISAM**: InnoDB supports transactions and row-level locking; MyISAM doesn't.
2. **Clustered index**: One per table (the primary key), stores rows in index order.
3. **ACID**: Atomicity, Consistency, Isolation, Durability.
4. **Isolation**: Repeatable Read is the InnoDB default.
5. **Optimization**: Use EXPLAIN, add the right indexes, avoid `SELECT *`, always index foreign keys.
6. **Data types**: Choose the smallest sufficient type; use DECIMAL (not FLOAT) for money.