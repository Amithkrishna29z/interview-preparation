# PostgreSQL Interview Questions & Answers

---

## Table of Contents
1. [PostgreSQL Basics](#postgresql-basics)
2. [Advanced Data Types](#advanced-data-types)
3. [JSON & JSONB Support](#json--jsonb-support)
4. [Indexing](#indexing)
5. [Query Optimization](#query-optimization)
6. [Stored Procedures & Functions](#stored-procedures--functions)
7. [PostgreSQL Extensions](#postgresql-extensions)
8. [Replication & High Availability](#replication--high-availability)
9. [Performance Tuning](#performance-tuning)
10. [PostgreSQL vs MySQL](#postgresql-vs-mysql)
11. [Window Functions & CTEs](#window-functions--ctes)
12. [Common Mistakes](#common-mistakes)
13. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## PostgreSQL Basics

### Q1: What is PostgreSQL and why is it used?

PostgreSQL is an open-source RDBMS known for reliability, ACID compliance, and advanced features. It supports complex data types (arrays, JSONB, ranges), full SQL standards, and rich extensibility. Common uses: enterprise apps, financial systems, web apps with complex data models.

### Q2: What are the key differences between PostgreSQL and MySQL?

| Feature | PostgreSQL | MySQL |
|---------|------------|-------|
| SQL Compliance | Very high | Good, less strict |
| JSON Support | JSONB (binary, fast, indexable) | JSON (text-based) |
| Data Types | Arrays, ranges, custom types | Standard types |
| Stored Procedures | PL/pgSQL (powerful) | Basic SQL |
| Extensions | Rich ecosystem | Limited |
| Write Performance | Excellent | Good |

**Choose PostgreSQL:** complex queries, advanced data types, strong integrity.  
**Choose MySQL:** simple CRUD, read-heavy workloads, LAMP-stack compatibility.

### Q3: What is MVCC in PostgreSQL?

MVCC (Multi-Version Concurrency Control) lets readers and writers work simultaneously without blocking each other. Each transaction sees a snapshot of the data at its start time — no dirty reads. Old row versions ("dead tuples") are cleaned up by the `VACUUM` process (`AUTOVACUUM` handles this automatically in the background).

```sql
-- Manually reclaim space from dead tuples
VACUUM users;
```

---

## Advanced Data Types

### Q4: What are the advanced data types in PostgreSQL?

**Arrays:**

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[]
);

INSERT INTO products (name, tags) VALUES ('Laptop', ARRAY['electronics', 'computers']);

SELECT name FROM products WHERE 'electronics' = ANY(tags);
UPDATE products SET tags = array_append(tags, 'new-tag') WHERE id = 1;
```

**Ranges:**

```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    date_range DATERANGE
);

INSERT INTO events (name, date_range) VALUES ('Conference', '[2024-01-01, 2024-01-31]');
SELECT * FROM events WHERE date_range @> '2024-01-15'::date;  -- Contains
```

**ENUMs:**

```sql
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status DEFAULT 'pending'
);
-- Note: enum values can be added but not easily removed or renamed
```

---

## JSON & JSONB Support

### Q5: How do you work with JSON and JSONB in PostgreSQL?

**Use JSONB by default** — it's pre-parsed (binary), faster to query, and supports GIN indexing. JSON (text) preserves whitespace/key order but is slower.

| Feature | JSON | JSONB |
|---------|------|-------|
| Storage | Text (exact copy) | Binary (parsed) |
| Query speed | Slower | Faster |
| Indexing | Limited | Excellent (GIN) |

```sql
CREATE TABLE products (id SERIAL PRIMARY KEY, name VARCHAR(100), metadata JSONB);

INSERT INTO products (name, metadata) VALUES (
    'Smartphone',
    '{"brand": "TechCorp", "specs": {"storage": "128GB"}, "price": 699.99}'::jsonb
);

-- Key operators:
-- ->  returns JSON     ->> returns text     #>> path as text     @> contains     ? key exists
SELECT name, metadata->>'brand' AS brand FROM products;
SELECT name, metadata->'specs'->>'storage' AS storage FROM products;
SELECT * FROM products WHERE metadata->>'brand' = 'TechCorp';
SELECT * FROM products WHERE (metadata->'price')::numeric > 500;
```

**Modification:**

```sql
UPDATE products SET metadata = jsonb_set(metadata, '{price}', '599.99'::jsonb) WHERE id = 1;
UPDATE products SET metadata = metadata || '{"discount": 10}'::jsonb WHERE id = 1;
UPDATE products SET metadata = metadata - 'discount' WHERE id = 1;
```

**Indexing JSONB:**

```sql
-- GIN index for containment queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Index on a specific field
CREATE INDEX idx_products_brand ON products ((metadata->>'brand'));
```

---

## Indexing

### Q6: What index types does PostgreSQL support?

As a junior dev you'll mostly use **B-tree** (the default). Know the others by name and purpose.

```sql
-- Standard B-tree
CREATE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Partial index (only matching rows)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- GIN index for arrays/JSONB/full-text search
CREATE INDEX idx_products_tags ON products USING GIN (tags);

-- Create without locking the table (production-safe)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

**Index types summary:**
- **B-tree**: default; equality, range, ORDER BY — covers most cases.
- **GIN**: arrays, JSONB, full-text search.
- **GiST**: geometric/spatial data, ranges.
- **BRIN**: very large naturally-ordered tables (e.g. time-series).
- **Hash**: equality-only; rarely needed.

### Q7: How do you manage and optimize indexes?

```sql
-- Check index usage (idx_scan = how often it's been used)
SELECT tablename, indexname, idx_scan FROM pg_stat_user_indexes ORDER BY idx_scan;

-- Find unused indexes (candidates to drop)
SELECT tablename, indexname FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE '%_pkey';

-- Refresh planner statistics
ANALYZE users;

-- Covering index: include extra columns to avoid heap lookups
CREATE INDEX idx_orders_covering ON orders(user_id, order_date) INCLUDE (total, status);
```

**Best practices:** index foreign keys; use partial indexes for filtered data; use `CONCURRENTLY` in production; drop unused indexes.

---

## Query Optimization

### Q8: How do you optimize queries in PostgreSQL?

Start with `EXPLAIN ANALYZE`, then add indexes and rewrite problem queries.

```sql
-- Show plan without running
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Run and show actual timing
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- Key things to read: Seq Scan (bad) vs Index Scan (good) vs Index Only Scan (best)
```

**Common patterns:**

```sql
-- Prefer JOIN over correlated subquery
SELECT DISTINCT u.* FROM users u
INNER JOIN orders o ON u.id = o.user_id WHERE o.total > 100;

-- CTE for readable complex queries
WITH user_orders AS (
    SELECT user_id, COUNT(*) AS order_count, SUM(total) AS total_spent
    FROM orders GROUP BY user_id
)
SELECT u.username, uo.order_count, uo.total_spent
FROM users u JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5;

-- Keyset pagination (faster than large OFFSET)
SELECT * FROM large_table WHERE id > last_seen_id ORDER BY id LIMIT 20;
```

**Avoid index-breaking patterns:**

```sql
-- ❌ Function on column prevents index use
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- ✅ Use an expression index instead
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- ❌ Leading wildcard can't use a B-tree index
SELECT * FROM products WHERE name LIKE '%widget%';
-- ✅ Use full-text search
SELECT * FROM products WHERE to_tsvector('english', name) @@ to_tsquery('english', 'widget');
```

---

## Stored Procedures & Functions

### Q9: How do you create stored procedures and functions in PostgreSQL?

**Function returning a table:**

```sql
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INTEGER)
RETURNS TABLE (order_id INTEGER, order_date TIMESTAMP, total NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT o.id, o.order_date, o.total FROM orders o
    WHERE o.user_id = p_user_id ORDER BY o.order_date DESC;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM get_user_orders(1);
```

**Stored Procedure (PostgreSQL 11+ — can manage transactions):**

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

**Trigger function:**

```sql
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

CREATE TRIGGER trigger_update_user_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION update_user_order_count();
```

---

## PostgreSQL Extensions

### Q10: What are PostgreSQL extensions and how do you use them?

Extensions add functionality and are enabled per database with `CREATE EXTENSION`.

**Common extensions to recognize:**
- **pg_stat_statements**: tracks slow queries by execution stats.
- **PostGIS**: geographic/spatial data.
- **pg_trgm**: fuzzy string matching and similarity.
- **uuid-ossp** / `gen_random_uuid()`: UUID primary keys.
- **hstore**: simple key-value column (JSONB is usually preferred).

```sql
-- Enable and use
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE users (id UUID PRIMARY KEY DEFAULT uuid_generate_v4(), username VARCHAR(50));

-- Fuzzy search with pg_trgm
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);
SELECT name, similarity(name, 'laptop') FROM products WHERE name % 'laptop';
```

---

## Replication & High Availability

### Q11: What are the replication methods in PostgreSQL?

Awareness-level for juniors — this is primarily a DBA responsibility.

- **Streaming (physical) replication**: ships the WAL byte-for-byte to standby servers; standbys can serve read-only queries. Used for failover/HA.
- **Logical replication**: replicates selected tables via `PUBLICATION`/`SUBSCRIPTION`; works across major versions.

```sql
-- Logical replication: publisher
CREATE PUBLICATION my_publication FOR TABLE users, orders;

-- Subscriber
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host dbname=mydb user=postgres password=password'
PUBLICATION my_publication;

-- Check replication lag
SELECT now() - pg_last_xact_replay_timestamp() AS lag;  -- on standby
```

---

## Performance Tuning

### Q12: How do you tune PostgreSQL performance?

Awareness-level for juniors — focus on writing good queries and indexes; know these settings exist.

```sql
-- Key postgresql.conf settings
shared_buffers = 4GB        -- ~25% of RAM
effective_cache_size = 12GB -- ~50-75% of RAM (planner hint)
work_mem = 64MB             -- per sort/hash operation
max_connections = 200
```

**Find slow queries with pg_stat_statements:**

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
SELECT query, calls, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 20;
```

`AUTOVACUUM` (on by default) prevents table bloat — defaults are fine for most apps.

---

## PostgreSQL vs MySQL

### Q13: Detailed comparison of PostgreSQL and MySQL

| Area | PostgreSQL | MySQL |
|------|------------|-------|
| SQL compliance | Very high | Good, some deviations |
| Data types | JSONB, arrays, ranges, custom | Standard types, basic JSON |
| Indexes | B-tree, Hash, GiST, GIN, BRIN | B-tree, Hash, Full-text |
| Transactions | Full ACID, savepoints | Full ACID (InnoDB) |
| Replication | Streaming + logical | Statement/row-based |
| Strengths | Complex queries, writes, concurrency | Read-heavy, simple queries |

---

## Window Functions & CTEs

### Q14: What are window functions in PostgreSQL?

Window functions calculate across related rows without collapsing them like `GROUP BY`. Think of them as "aggregates that keep every row."

```sql
-- Ranking functions
SELECT username, total,
    ROW_NUMBER() OVER (ORDER BY total DESC) AS row_num,
    RANK()       OVER (ORDER BY total DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY total DESC) AS dense_rank
FROM orders;

-- Running total + LAG (access previous row)
SELECT username, order_date, total,
    SUM(total) OVER (ORDER BY order_date) AS running_total,
    LAG(total) OVER (ORDER BY order_date) AS previous_total
FROM orders;

-- PARTITION BY: restart calculation per group
SELECT username, order_date, total,
    SUM(total) OVER (PARTITION BY user_id ORDER BY order_date) AS user_running_total
FROM orders JOIN users ON orders.user_id = users.id;
```

### Q15: What are CTEs in PostgreSQL?

CTEs (`WITH` clauses) define temporary named result sets to make complex queries readable. Recursive CTEs traverse hierarchical data.

```sql
-- Basic CTE
WITH user_orders AS (
    SELECT user_id, COUNT(*) AS order_count, SUM(total) AS total_spent
    FROM orders GROUP BY user_id
)
SELECT u.username, uo.order_count, uo.total_spent
FROM users u JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5;

-- Recursive CTE (org chart)
WITH RECURSIVE org_chart AS (
    SELECT id, name, manager_id, 1 AS level FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

---

## Common Mistakes

**Mistake 1: Wrong data types for IDs**
```sql
-- ❌ VARCHAR for IDs
CREATE TABLE orders (id VARCHAR(50) PRIMARY KEY, user_id VARCHAR(50));
-- ✅ Use SERIAL / INTEGER
CREATE TABLE orders (id SERIAL PRIMARY KEY, user_id INTEGER REFERENCES users(id));
```

**Mistake 2: No connection pooling** — opening a new DB connection per query is expensive. Use HikariCP (Spring Boot default) to reuse connections.

**Mistake 3: No ANALYZE after bulk inserts**
```sql
INSERT INTO large_table SELECT * FROM staging_table;
VACUUM ANALYZE large_table;  -- update planner statistics
```

**Mistake 4: No transaction for multi-step operations**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Mistake 5: String concatenation instead of parameterized queries**
```sql
-- ❌ SQL injection risk
-- SELECT * FROM users WHERE email = '" + email + "'"

-- ✅ Prepared statement
PREPARE get_user AS SELECT * FROM users WHERE email = $1;
EXECUTE get_user('user@example.com');
```

---

## Quick Reference Cheat Sheet

**Indexes:** B-tree (default), GIN (JSONB/arrays/full-text), GiST (geo/ranges), BRIN (large ordered). Partial: `WHERE condition`. Expression: `function(column)`.

**Query optimization:** `EXPLAIN ANALYZE`; add appropriate indexes; avoid functions on indexed columns; prefer JOINs over correlated subqueries.

**MVCC:** Readers don't block writers; `AUTOVACUUM` cleans dead row versions.

**JSON:** Prefer JSONB. `->>` (text), `->` (JSON), `@>` (contains); GIN index for fast lookup.

**Transactions:** Full ACID; always wrap multi-step operations in `BEGIN`/`COMMIT`.

```sql
-- Index
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_name ON table USING GIN (jsonb_column);

-- JSON access and update
SELECT data->'nested'->>'key' FROM table;
UPDATE table SET data = data || '{"new": "value"}'::jsonb;

-- Window function
SELECT col, ROW_NUMBER() OVER (PARTITION BY grp ORDER BY col) FROM table;

-- CTE
WITH cte AS (SELECT ... FROM ...) SELECT * FROM cte;
```

---

**Next Topics to Study:**
- MongoDB Document Modeling and Aggregation
- General Database Concepts (ACID, Normalization, SQL Fundamentals)
- NoSQL vs SQL Decision Making
- Cloud Database Services (AWS RDS, Google Cloud SQL, Azure Database)

---

*Last Updated: 2026-06-18*
