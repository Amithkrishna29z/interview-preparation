# PostgreSQL Interview Questions & Answers

---

## PostgreSQL Basics

### Q1: What is PostgreSQL and why is it used?

**Answer:** PostgreSQL is an advanced, open-source relational database management system (RDBMS) known for its reliability, feature robustness, and performance. It's used because:

- **Advanced Features**: Complex data types, indexes, and query optimization
- **Extensibility**: User-defined types, functions, and extensions
- **Standards Compliance**: ACID compliant, SQL standard compliant
- **Data Integrity**: Strong constraints, foreign keys, and triggers
- **Open Source**: Free with permissive license

**Real-world use cases:**
- Complex enterprise applications
- Financial applications requiring strong data integrity
- Web applications with complex data models
- Data warehousing and analytics

### Q2: What are the key differences between PostgreSQL and MySQL?

**Answer:**

| Feature | PostgreSQL | MySQL |
|---------|------------|-------|
| **License** | PostgreSQL License (permissive) | GPL (more restrictive) |
| **SQL Compliance** | Very high | Good, but less strict |
| **Window Functions** | Full support | Limited support |
| **JSON Support** | JSONB (binary, fast) | JSON (text-based) |
| **Stored Procedures** | PL/pgSQL (powerful) | Basic SQL |
| **Extensions** | Rich ecosystem | Limited |
| **Replication** | Logical & physical | Master-slave, group replication |
| **Write Performance** | Excellent | Good |

**When to choose PostgreSQL:**
- Complex data relationships and queries
- Need for advanced data types (JSON, arrays, etc.)
- Complex transactions and concurrency

**When to choose MySQL:**
- Simple CRUD applications
- Read-heavy web workloads
- Easier setup and LAMP stack compatibility

### Q3: What is MVCC in PostgreSQL?

**Answer:** MVCC (Multi-Version Concurrency Control) is how PostgreSQL handles concurrent access without read locks. Each transaction sees a snapshot of the database as of its start time, multiple row versions can exist at once, and readers don't block writers (or vice versa).

**Key points:**
- No dirty reads, snapshot isolation for consistent reads
- Old row versions ("dead tuples") are cleaned up by the VACUUM process
- Internally, hidden system columns (`xmin`/`xmax`) track which transaction created/expired each row version

**VACUUM (awareness only):** `VACUUM` reclaims space from dead tuples so tables don't bloat. `AUTOVACUUM` runs this automatically in the background — junior devs rarely tune it, just know it exists and why it matters.

```sql
-- Reclaim space from dead tuples
VACUUM users;
```

---

## Advanced Data Types

### Q4: What are the advanced data types in PostgreSQL?

**Answer:** PostgreSQL supports many advanced data types beyond standard SQL types.

**Array Types:**

```sql
-- Create table with array column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[],              -- Array of text
    categories INTEGER[]      -- Array of integers
);

-- Insert data with arrays
INSERT INTO products (name, tags, categories)
VALUES ('Laptop', ARRAY['electronics', 'computers'], ARRAY[1, 5, 10]);

-- Query array data
SELECT name, tags FROM products WHERE 'electronics' = ANY(tags);

-- Common array functions
SELECT
    name,
    array_length(tags, 1) as tag_count,      -- Get array length
    unnest(tags) as individual_tag           -- Expand array to rows
FROM products;

-- Add to / remove from array
UPDATE products SET tags = array_append(tags, 'new-tag') WHERE id = 1;
UPDATE products SET tags = array_remove(tags, 'old-tag') WHERE id = 1;
```

**Range Types:**

```sql
-- Create table with range columns
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    date_range DATERANGE,    -- Date range
    price_range NUMRANGE     -- Numeric range
);

-- Insert range data ('[' inclusive, ')' exclusive)
INSERT INTO events (name, date_range, price_range)
VALUES ('Conference', '[2024-01-01, 2024-01-31]', '[100, 500)');

-- Query ranges
SELECT * FROM events WHERE date_range @> '2024-01-15'::date;     -- Contains
SELECT * FROM events WHERE price_range && '[200, 300]'::numrange; -- Overlaps
```

**Custom Types (ENUM):**

```sql
-- Create custom enum type
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

-- Create table using enum
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status DEFAULT 'pending'
);

INSERT INTO orders (status) VALUES ('pending');
SELECT * FROM orders WHERE status = 'pending';

-- Note: enum values can be added but not easily removed/renamed
```

---

## JSON & JSONB Support

### Q5: How do you work with JSON and JSONB in PostgreSQL?

**Answer:** PostgreSQL offers both JSON (text-based) and JSONB (binary, optimized for performance) types. **Use JSONB by default** — it's faster to query and supports indexing.

**JSON vs JSONB:**

| Feature | JSON | JSONB |
|---------|------|-------|
| Storage | Text (exact copy) | Binary (parsed) |
| Querying | Slower (re-parse each time) | Faster (already parsed) |
| Indexing | Limited | Excellent (GIN indexes) |
| Whitespace / key order | Preserved | Not preserved |

**JSON/JSONB Operations:**

```sql
-- Create table with JSONB column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    metadata JSONB
);

-- Insert JSON data
INSERT INTO products (name, metadata)
VALUES (
    'Smartphone',
    '{
        "brand": "TechCorp",
        "specs": {"screen": "6.5 inch", "storage": "128GB"},
        "colors": ["black", "white", "blue"],
        "price": 699.99
    }'::jsonb
);

-- Simple key access (->> returns text, -> returns JSON)
SELECT name, metadata->>'brand' as brand FROM products;

-- Nested key access
SELECT name, metadata->'specs'->>'screen' as screen_size FROM products;

-- JSON path access
SELECT name, metadata#>>'{specs,storage}' as storage FROM products;

-- WHERE clauses with JSON
SELECT * FROM products WHERE metadata->>'brand' = 'TechCorp';
SELECT * FROM products WHERE (metadata->'price')::numeric > 500;

-- Array contains
SELECT * FROM products WHERE metadata->'colors' @> '"black"'::jsonb;

-- Common operators:
-- ->  field as JSON       ->> field as text
-- #>  path as JSON        #>> path as text
-- @>  contains            ?  key exists
```

**JSON Modification:**

```sql
-- Update a field
UPDATE products SET metadata = jsonb_set(metadata, '{price}', '599.99'::jsonb) WHERE id = 1;

-- Add a new field (merge)
UPDATE products SET metadata = metadata || '{"discount": 10}'::jsonb WHERE id = 1;

-- Delete a field
UPDATE products SET metadata = metadata - 'discount' WHERE id = 1;
```

**JSON Indexing:**

```sql
-- GIN index for JSONB (recommended for containment queries)
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Index on a specific JSON field
CREATE INDEX idx_products_brand ON products ((metadata->>'brand'));

-- Used by queries like:
SELECT * FROM products WHERE metadata @> '{"brand": "TechCorp"}'::jsonb;
SELECT * FROM products WHERE metadata->>'brand' = 'TechCorp';
```

**JSON Aggregate Functions:**

```sql
-- Build a JSON object per row
SELECT json_build_object(
    'product', name,
    'brand', metadata->>'brand',
    'specs', metadata->'specs'
) as product_info
FROM products;

-- Aggregate rows into a JSON array
SELECT json_agg(name) as product_names FROM products;
```

---

## Advanced Indexing

### Q6: What are the different types of indexes in PostgreSQL?

**Answer:** PostgreSQL supports several index types. As a junior dev you'll mostly use **B-tree** (the default); know the others by name.

**B-Tree Index (Default — the one you use most):**

```sql
-- Standard B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Unique index
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Composite index (multiple columns)
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Partial index (only rows matching a condition)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Expression index (index on a function result)
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- Enables: SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

**Other index types (awareness — one line each):**
- **Hash**: equality (`=`) only; rarely needed since B-tree handles equality well.
- **GIN** (Generalized Inverted Index): for arrays, JSONB, and full-text search — the one to use for JSONB.
- **GiST** (Generalized Search Tree): for geometric/spatial data and ranges.
- **BRIN** (Block Range Index): tiny index for very large, naturally-ordered tables (e.g. time-series).
- **SP-GiST**: for non-balanced structures like quadtrees and prefix trees.

```sql
-- Example: GIN index on a JSONB or array column
CREATE INDEX idx_products_tags ON products USING GIN (tags);
```

**Concurrent Index Creation:**

```sql
-- Create an index without locking the table (can't run inside a transaction)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

### Q7: How do you optimize indexes in PostgreSQL?

**Answer:** Find unused/inefficient indexes via the stats views, keep statistics fresh, and follow a few best practices.

**Index Usage Analysis:**

```sql
-- Check index usage (idx_scan = how often it's used)
SELECT tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- Find unused indexes (candidates to drop)
SELECT tablename, indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE '%_pkey';
```

**Index Maintenance:**

```sql
-- Rebuild an index
REINDEX INDEX idx_users_email;

-- Update planner statistics (and reclaim space)
ANALYZE users;
VACUUM ANALYZE users;
```

**Index Best Practices:**

```sql
-- 1. Index foreign keys
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 2. Partial indexes for filtered data
CREATE INDEX idx_active_users_email ON users(email) WHERE is_active = true;

-- 3. Covering indexes with INCLUDE (PostgreSQL 11+)
CREATE INDEX idx_orders_covering ON orders(user_id, order_date) INCLUDE (total, status);

-- 4. Pick the right index type (B-tree default; GIN for JSONB/arrays)
-- 5. Monitor and drop unused indexes (query above)
-- 6. Use CONCURRENTLY in production to avoid locking
```

---

## Query Optimization

### Q8: How do you optimize queries in PostgreSQL?

**Answer:** Start with `EXPLAIN ANALYZE` to see the plan, then add indexes and rewrite problem queries.

**EXPLAIN and EXPLAIN ANALYZE:**

```sql
-- Show the plan without running the query
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Run the query and show actual timing
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- Include I/O information
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM large_table WHERE condition;

-- Key things to read:
-- Scan type: Seq Scan (bad), Index Scan (good), Index Only Scan (best)
-- Cost / Actual time / Rows processed
```

**Common Query Patterns:**

```sql
-- 1. Add an index to avoid a full table scan
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'user@example.com';

-- 2. JOIN types
-- INNER JOIN: only matching rows
SELECT u.username, o.order_date FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all from left, matching from right
SELECT u.username, o.order_date FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 3. Prefer JOIN over a correlated subquery
SELECT DISTINCT u.*
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.total > 100;

-- 4. Use CTEs for complex queries
WITH user_orders AS (
    SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
    FROM orders
    GROUP BY user_id
)
SELECT u.username, uo.order_count, uo.total_spent
FROM users u
INNER JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5;

-- 5. Pagination
SELECT * FROM large_table ORDER BY created_at DESC LIMIT 20 OFFSET 0;
-- Keyset pagination is faster for large offsets:
SELECT * FROM large_table WHERE id > last_seen_id ORDER BY id LIMIT 20;
```

**Query Rewriting Examples:**

```sql
-- ❌ Function on a column prevents index use
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- ✅ Add an expression index (or use ILIKE)
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- ❌ Leading wildcard can't use a normal index
SELECT * FROM products WHERE name LIKE '%widget%';
-- ✅ Use full-text search for partial matches
CREATE INDEX idx_products_name ON products USING GIN (to_tsvector('english', name));
SELECT * FROM products
WHERE to_tsvector('english', name) @@ to_tsquery('english', 'widget');
```

---

## Stored Procedures & Functions

### Q9: How do you create and use stored procedures in PostgreSQL?

**Answer:** PostgreSQL supports functions and procedures using PL/pgSQL (Procedural Language/PostgreSQL).

**Basic Function:**

```sql
-- Function returning a table
CREATE OR REPLACE FUNCTION get_user_orders(user_id INTEGER)
RETURNS TABLE (
    order_id INTEGER,
    order_date TIMESTAMP,
    total NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    SELECT o.id, o.order_date, o.total
    FROM orders o
    WHERE o.user_id = get_user_orders.user_id
    ORDER BY o.order_date DESC;
END;
$$ LANGUAGE plpgsql;

-- Use the function
SELECT * FROM get_user_orders(1);
```

**Function returning a scalar value:**

```sql
CREATE OR REPLACE FUNCTION calculate_order_total(order_id INTEGER)
RETURNS NUMERIC AS $$
DECLARE
    order_total NUMERIC;
BEGIN
    SELECT COALESCE(SUM(price * quantity), 0)
    INTO order_total
    FROM order_items
    WHERE order_id = calculate_order_total.order_id;

    RETURN order_total;
END;
$$ LANGUAGE plpgsql;

SELECT o.id, calculate_order_total(o.id) as total FROM orders o;
```

**Stored Procedure (PostgreSQL 11+, can manage transactions):**

```sql
CREATE OR REPLACE PROCEDURE archive_old_orders(p_before DATE) AS $$
BEGIN
    INSERT INTO orders_archive SELECT * FROM orders WHERE order_date < p_before;
    DELETE FROM orders WHERE order_date < p_before;
    COMMIT;
END;
$$ LANGUAGE plpgsql;

CALL archive_old_orders('2023-01-01');
```

**Trigger Functions:**

```sql
-- Trigger function: keep a per-user order count in sync
CREATE OR REPLACE FUNCTION update_user_order_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users SET order_count = order_count + 1 WHERE id = NEW.user_id;
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users SET order_count = order_count - 1 WHERE id = OLD.user_id;
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Attach the trigger
CREATE TRIGGER trigger_update_user_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW
EXECUTE FUNCTION update_user_order_count();
```

---

## PostgreSQL Extensions

### Q10: What are PostgreSQL extensions and how do you use them?

**Answer:** Extensions add functionality and are enabled per database with `CREATE EXTENSION`.

```sql
-- List and install
SELECT * FROM pg_available_extensions;
CREATE EXTENSION IF NOT EXISTS extension_name;
```

**Common extensions juniors should recognize:**
- **pg_stat_statements**: tracks query execution stats to find slow queries.
- **PostGIS**: geographic/spatial data and queries.
- **pg_trgm**: trigram-based fuzzy string matching and similarity.
- **uuid-ossp** (or built-in `gen_random_uuid()`): generate UUID primary keys.
- **hstore**: simple key-value column type (JSONB is usually preferred today).

```sql
-- Example: UUID primary keys
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50)
);

-- Example: pg_trgm fuzzy match
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);
SELECT name, similarity(name, 'laptop') FROM products WHERE name % 'laptop';
```

---

## Replication & High Availability

### Q11: What are the different replication methods in PostgreSQL?

**Answer (awareness only — this is DBA territory):** PostgreSQL replicates data from a primary to one or more standby servers for high availability and read scaling. The two main types:

- **Streaming (physical) replication**: ships the write-ahead log (WAL) to byte-for-byte copies of the whole cluster; standbys can serve read-only queries (`hot_standby`). Used for failover/HA.
- **Logical replication**: replicates selected tables via `PUBLICATION`/`SUBSCRIPTION`; allows replicating a subset of data and across different major versions.

```sql
-- Logical replication: the core commands
-- On the publisher:
CREATE PUBLICATION my_publication FOR TABLE users, orders;

-- On the subscriber:
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host dbname=mydb user=postgres password=password'
PUBLICATION my_publication;
```

```sql
-- Check replication status / lag (handy to know)
SELECT client_addr, state, sync_state FROM pg_stat_replication;   -- on primary
SELECT now() - pg_last_xact_replay_timestamp() AS lag;            -- on standby
```

---

## Performance Tuning

### Q12: How do you tune PostgreSQL performance?

**Answer (awareness only — mostly a DBA task):** Tuning means adjusting a handful of `postgresql.conf` settings to match the hardware, keeping statistics fresh, and finding slow queries. As a junior dev, focus on writing good queries and indexes; know these knobs exist.

**Key configuration parameters:**

```sql
-- Memory
shared_buffers = 4GB           -- ~25% of RAM on a dedicated server
effective_cache_size = 12GB    -- ~50-75% of RAM (planner hint)
work_mem = 64MB                -- per sort/hash operation

-- Planner (SSD-friendly defaults)
random_page_cost = 1.1

-- Connections
max_connections = 200
```

**Find slow queries (the practical part):**

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query, calls, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;
```

`AUTOVACUUM` (on by default) keeps tables from bloating; per-table thresholds can be tuned, but defaults are fine for most apps.

---

## PostgreSQL vs MySQL

### Q13: Detailed comparison of PostgreSQL and MySQL?

**Answer:**

| Area | PostgreSQL | MySQL |
|------|------------|-------|
| Architecture | Process per connection | Thread per connection |
| SQL compliance | Very high | Good, some deviations |
| Data types | JSONB, arrays, ranges, custom types | Standard types, basic JSON |
| Indexes | B-tree, Hash, GiST, GIN, BRIN, SP-GiST | B-tree, Hash, Full-text, Spatial |
| Transactions | Full ACID, savepoints, two-phase commit | Full ACID (InnoDB), savepoints |
| Replication | Streaming (physical), logical | Statement/row-based, group replication |
| Strengths | Complex queries, writes, concurrency | Read-heavy, simple queries |

**Choose PostgreSQL when:** complex data models, advanced data types, analytical queries, geographic data, strong data integrity, or extensibility matter.

**Choose MySQL when:** simple CRUD, read-heavy web workloads, easy setup/maintenance, or LAMP-stack compatibility matter.

---

## Advanced Features

### Q14: What are window functions in PostgreSQL?

**Answer:** Window functions perform calculations across a set of rows related to the current row, without collapsing them like `GROUP BY`.

**Basic Window Functions:**

```sql
-- ROW_NUMBER / RANK / DENSE_RANK
SELECT
    username,
    total,
    ROW_NUMBER() OVER (ORDER BY total DESC) as row_num,
    RANK()       OVER (ORDER BY total DESC) as rank,
    DENSE_RANK() OVER (ORDER BY total DESC) as dense_rank
FROM orders;

-- LAG / LEAD: access other rows' values
SELECT
    username,
    order_date,
    total,
    LAG(total)  OVER (ORDER BY order_date) as previous_total,
    LEAD(total) OVER (ORDER BY order_date) as next_total
FROM orders;

-- Aggregates as window functions (running total)
SELECT
    username,
    order_date,
    total,
    SUM(total) OVER (ORDER BY order_date) as running_total
FROM orders;
```

**PARTITION BY:**

```sql
-- Restart the calculation per user
SELECT
    username,
    order_date,
    total,
    SUM(total)   OVER (PARTITION BY user_id ORDER BY order_date) as user_running_total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) as user_order_number
FROM orders
JOIN users ON orders.user_id = users.id;
```

### Q15: What are Common Table Expressions (CTEs) in PostgreSQL?

**Answer:** CTEs define temporary, named result sets (with `WITH`) that improve readability of complex queries and can be referenced like a table.

**Basic CTE:**

```sql
WITH user_orders AS (
    SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
    FROM orders
    GROUP BY user_id
)
SELECT u.username, uo.order_count, uo.total_spent
FROM users u
JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5;
```

**Recursive CTE (hierarchical data):**

```sql
-- Organization chart
WITH RECURSIVE org_chart AS (
    -- Base case: top-level managers
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: employees under managers
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

---

## Common Mistakes

### Mistake 1: Not using appropriate data types

```sql
-- ❌ BAD: Using VARCHAR for IDs
CREATE TABLE orders (id VARCHAR(50) PRIMARY KEY, user_id VARCHAR(50));

-- ✅ GOOD: Using appropriate types
CREATE TABLE orders (id SERIAL PRIMARY KEY, user_id INTEGER REFERENCES users(id));
```

### Mistake 2: Not using connection pooling

Opening a new database connection per query is expensive. Use a connection pool (e.g. HikariCP in Spring Boot) and reuse connections.

### Mistake 3: Not analyzing tables after bulk operations

```sql
-- After large inserts, update statistics so the planner picks good plans
INSERT INTO large_table SELECT * FROM staging_table;
VACUUM ANALYZE large_table;
```

### Mistake 4: Not using transactions for multi-step operations

```sql
-- ✅ GOOD: All-or-nothing with a transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- or ROLLBACK on error
```

### Mistake 5: Not using prepared statements

```sql
-- ❌ BAD: string concatenation = SQL injection risk
$sql = "SELECT * FROM users WHERE email = '" . $email . "'";

-- ✅ GOOD: parameterized / prepared statement
PREPARE get_user AS SELECT * FROM users WHERE email = $1;
EXECUTE get_user('user@example.com');
```

---

## Short Revision Summary

**Indexes:** B-tree (default), GIN (JSONB/arrays/full-text), GiST (geo/ranges), BRIN (large ordered), Hash (equality). Partial: `WHERE condition`. Expression: `function(column)`.

**Query optimization:** Use `EXPLAIN ANALYZE`; add appropriate indexes; avoid full scans and functions on indexed columns; prefer JOINs over correlated subqueries; use CTEs and window functions.

**MVCC:** Multi-Version Concurrency Control — readers don't block writers; `VACUUM`/autovacuum cleans up dead row versions.

**JSON:** Prefer JSONB. `->>` (text), `->` (JSON), `@>` (contains); index with GIN.

**Transactions:** Full ACID, savepoints, multiple isolation levels.

**Quick reference:**

```sql
-- Index
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_name ON table USING GIN (jsonb_column);

-- JSON
SELECT data->'nested'->>'key' FROM table;
UPDATE table SET data = data || '{"new": "value"}'::jsonb;

-- Window function
SELECT col, ROW_NUMBER() OVER (PARTITION BY g ORDER BY col) FROM table;

-- CTE
WITH cte AS (SELECT ... FROM ...) SELECT * FROM cte;
```

---

**Next Topics to Study:**
- MongoDB Document Modeling and Aggregation
- General Database Concepts (ACID, Normalization, SQL Fundamentals)
- NoSQL vs SQL Decision Making
- Cloud Database Services (AWS RDS, Google Cloud SQL, Azure Database)
