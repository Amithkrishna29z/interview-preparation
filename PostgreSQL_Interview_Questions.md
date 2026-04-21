# PostgreSQL Interview Questions & Answers

## Table of Contents
1. [PostgreSQL Basics](#postgresql-basics)
2. [Advanced Data Types](#advanced-data-types)
3. [JSON & JSONB Support](#json--jsonb-support)
4. [Advanced Indexing](#advanced-indexing)
5. [Query Optimization](#query-optimization)
6. [Stored Procedures & Functions](#stored-procedures--functions-1)
7. [PostgreSQL Extensions](#postgresql-extensions)
8. [Replication & High Availability](#replication--high-availability)
9. [Performance Tuning](#performance-tuning)
10. [PostgreSQL vs MySQL](#postgresql-vs-mysql)
11. [Advanced Features](#advanced-features)
12. [Common Mistakes](#common-mistakes-1)
13. [Short Revision Summary](#short-revision-summary-1)

---

## PostgreSQL Basics

### Q1: What is PostgreSQL and why is it used?

**Answer:** PostgreSQL is an advanced, open-source relational database management system (RDBMS) known for its reliability, feature robustness, and performance. It's used because:

- **Advanced Features**: Complex data types, indexes, and query optimization
- **Extensibility**: User-defined types, functions, and extensions
- **Standards Compliance**: ACID compliant, SQL standard compliant
- **Performance**: Advanced query planner and optimizer
- **Scalability**: Supports both vertical and horizontal scaling
- **Data Integrity**: Strong constraints, foreign keys, and triggers
- **Open Source**: Free with permissive license
- **Large Ecosystem**: Rich set of extensions and tools

**Real-world use cases:**
- Complex enterprise applications
- Geographic information systems (GIS)
- Financial applications requiring strong data integrity
- Data warehousing and analytics
- Web applications with complex data models

### Q2: What are the key differences between PostgreSQL and MySQL?

**Answer:**

| Feature | PostgreSQL | MySQL |
|---------|------------|-------|
| **License** | PostgreSQL License (permissive) | GPL (more restrictive) |
| **SQL Compliance** | Very high | Good, but less strict |
| **Complex Queries** | Excellent | Good |
| **Window Functions** | Full support | Limited support |
| **JSON Support** | JSONB (binary, fast) | JSON (text-based) |
| **Full-Text Search** | Excellent (tsvector) | Good |
| **Stored Procedures** | PL/pgSQL (powerful) | Basic SQL |
| **Extensions** | Rich ecosystem | Limited |
| **Replication** | Logical & physical | Master-slave, group replication |
| **MVCC** | Multi-version concurrency control | MVCC (implementation differs) |
| **Write Performance** | Excellent | Good |
| **Read Performance** | Excellent | Excellent |
| **Configuration** | Complex but powerful | Simpler |

**When to choose PostgreSQL:**
- Complex data relationships and queries
- Need for advanced data types (JSON, arrays, etc.)
- Geographic data and spatial queries
- Complex transactions and concurrency
- Need for custom extensions

**When to choose MySQL:**
- Simple CRUD applications
- Web applications with read-heavy workloads
- Need for easier setup and maintenance
- LAMP stack compatibility
- Large community support for web development

### Q3: What is MVCC in PostgreSQL?

**Answer:** MVCC (Multi-Version Concurrency Control) is a method PostgreSQL uses to handle concurrent access to data without locking.

**How MVCC Works:**
- Each transaction sees a snapshot of the database as of the start time
- Multiple versions of rows can exist simultaneously
- Readers don't block writers, writers don't block readers
- Old versions are cleaned up by VACUUM process

**Key Concepts:**

```sql
-- PostgreSQL uses system columns to track versions:
-- xmin: Transaction ID that created the row
-- xmax: Transaction ID that expired the row
-- cmin: Command identifier within creating transaction
-- cmax: Command identifier within expiring transaction

-- View these system columns
SELECT xmin, xmax, cmin, cmax, * FROM users;

-- Example of how MVCC works
BEGIN;
-- Transaction 1 starts
SELECT * FROM users WHERE id = 1;  -- Sees version as of transaction start

-- Meanwhile, Transaction 2 updates the row
-- Transaction 2 sees its own changes
-- Transaction 1 still sees old version
COMMIT;
```

**Benefits of MVCC:**
- High concurrency without read locks
- No dirty reads (in default isolation level)
- Snapshot isolation for consistent reads
- Time travel queries (can query past data)

**VACUUM Process:**
```sql
-- Regular VACUUM reclaims space from dead tuples
VACUUM users;

-- VACUUM FULL rewrites table (locks table)
VACUUM FULL users;

-- AUTOVACUUM (automatic maintenance)
-- Configured in postgresql.conf
autovacuum = on
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
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
    prices NUMERIC(10,2)[],   -- Array of prices
    categories INTEGER[]      -- Array of integers
);

-- Insert data with arrays
INSERT INTO products (name, tags, prices, categories)
VALUES (
    'Laptop',
    ARRAY['electronics', 'computers', 'portable'],
    ARRAY[999.99, 899.99, 799.99],
    ARRAY[1, 5, 10]
);

-- Query array data
SELECT name, tags FROM products WHERE 'electronics' = ANY(tags);

-- Array functions
SELECT
    name,
    array_length(tags, 1) as tag_count,      -- Get array length
    array_to_string(tags, ', ') as tag_list, -- Convert to string
    unnest(tags) as individual_tag           -- Expand array to rows
FROM products;

-- Update array
UPDATE products
SET tags = array_append(tags, 'new-tag')
WHERE id = 1;

-- Remove from array
UPDATE products
SET tags = array_remove(tags, 'old-tag')
WHERE id = 1;
```

**Range Types:**

```sql
-- Create table with range columns
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    date_range DATERANGE,    -- Date range
    price_range NUMRANGE,    -- Numeric range
    time_range TSRANGE       -- Timestamp range
);

-- Insert range data
INSERT INTO events (name, date_range, price_range)
VALUES (
    'Conference',
    '[2024-01-01, 2024-01-31]',  -- Inclusive range
    '[100, 500)'                  -- Inclusive start, exclusive end
);

-- Query ranges
SELECT * FROM events WHERE date_range @> '2024-01-15'::date;  -- Contains
SELECT * FROM events WHERE price_range && '[200, 300]'::numrange;  -- Overlaps

-- Range functions
SELECT
    name,
    lower(date_range) as start_date,
    upper(date_range) as end_date,
    isempty(date_range) as is_empty
FROM events;
```

**Custom Types (ENUM):**

```sql
-- Create custom enum type
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

-- Create table using enum
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert and query
INSERT INTO orders (status) VALUES ('pending');
SELECT * FROM orders WHERE status = 'pending';

-- Add value to enum (can only add at end)
ALTER TYPE order_status ADD VALUE 'refunded' BEFORE 'cancelled';

-- Note: Can't remove or rename enum values easily
```

### Q5: How do you work with JSON and JSONB in PostgreSQL?

**Answer:** PostgreSQL offers robust JSON support with both JSON (text-based) and JSONB (binary, optimized for performance) types.

**JSON vs JSONB:**

| Feature | JSON | JSONB |
|---------|------|-------|
| Storage | Text (exact copy) | Binary (parsed) |
| Insertion | Fast (no parsing) | Slower (parsing required) |
| Querying | Slower (re-parse each time) | Faster (already parsed) |
| Indexing | Limited (GIN indexes) | Excellent (GIN indexes) |
| Whitespace | Preserved | Removed |
| Key order | Preserved | Not preserved |
| Duplicate keys | Preserved | Last value wins |

**JSON/JSONB Operations:**

```sql
-- Create table with JSON column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    metadata JSONB,           -- Use JSONB for better performance
    attributes JSON           -- Use JSON if exact formatting needed
);

-- Insert JSON data
INSERT INTO products (name, metadata, attributes)
VALUES (
    'Smartphone',
    '{
        "brand": "TechCorp",
        "model": "X100",
        "specs": {
            "screen": "6.5 inch",
            "storage": "128GB",
            "ram": "8GB"
        },
        "colors": ["black", "white", "blue"],
        "price": 699.99,
        "available": true
    }'::jsonb,
    '{
        "brand": "TechCorp",
        "model": "X100"
    }'::json
);

-- Query JSON data
-- Simple key access
SELECT name, metadata->>'brand' as brand
FROM products;

-- Nested key access
SELECT name, metadata->'specs'->>'screen' as screen_size
FROM products;

-- Array element access
SELECT name, metadata->'colors'->>0 as first_color
FROM products;

-- JSON path queries
SELECT name, metadata#>>'{specs,storage}' as storage
FROM products;

-- WHERE clauses with JSON
SELECT * FROM products WHERE metadata->>'brand' = 'TechCorp';
SELECT * FROM products WHERE (metadata->'price')::numeric > 500;
SELECT * FROM products WHERE metadata->>'available' = 'true';

-- JSON array contains
SELECT * FROM products WHERE metadata->'colors' @> '"black"'::jsonb;

-- JSON operators:
-- ->  Get JSON object field (returns JSON)
-- ->> Get JSON object field as text
-- #>  Get JSON object at specified path
-- #>> Get JSON object at specified path as text
-- @>  Contains (for JSONB)
-- <@  Contained in (for JSONB)
-- ?   Key exists (for JSONB)
-- ?|  Any of these keys exist (for JSONB)
-- ?&  All of these keys exist (for JSONB)
```

**JSON Modification:**

```sql
-- Update JSON field
UPDATE products
SET metadata = jsonb_set(metadata, '{price}', '599.99'::jsonb)
WHERE id = 1;

-- Add new field
UPDATE products
SET metadata = metadata || '{"discount": 10}'::jsonb
WHERE id = 1;

-- Delete field
UPDATE products
SET metadata = metadata - 'discount'
WHERE id = 1;

-- Add to array
UPDATE products
SET metadata = jsonb_set(metadata, '{colors}', metadata->'colors' || '["red"]'::jsonb)
WHERE id = 1;

-- Remove from array
UPDATE products
SET metadata = jsonb_set(metadata, '{colors}', (metadata->'colors') - 'red')
WHERE id = 1;
```

**JSON Indexing:**

```sql
-- Create GIN index for JSONB (recommended)
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Create GIN index with jsonb_path_ops operator class (smaller, faster)
CREATE INDEX idx_products_metadata_path ON products USING GIN (metadata jsonb_path_ops);

-- Create index on specific JSON field
CREATE INDEX idx_products_brand ON products ((metadata->>'brand'));

-- Create index on JSON array
CREATE INDEX idx_products_colors ON products USING GIN ((metadata->'colors'));

-- These indexes will be used for queries like:
SELECT * FROM products WHERE metadata @> '{"brand": "TechCorp"}'::jsonb;
SELECT * FROM products WHERE metadata->>'brand' = 'TechCorp';
SELECT * FROM products WHERE metadata->'colors' @> '"black"'::jsonb;
```

**JSON Aggregate Functions:**

```sql
-- Aggregate rows into JSON array
SELECT json_agg(name) as product_names FROM products;

-- Aggregate rows into JSON object
SELECT json_object_agg(name, metadata->>'brand') as product_brands
FROM products;

-- Build complex JSON structures
SELECT
    json_build_object(
        'product', name,
        'brand', metadata->>'brand',
        'price', (metadata->>'price')::numeric,
        'specs', metadata->'specs'
    ) as product_info
FROM products;

-- Create JSON from query results
SELECT json_agg(
    json_build_object(
        'id', id,
        'name', name,
        'brand', metadata->>'brand'
    )
) as products_json
FROM products;
```

---

## Advanced Indexing

### Q6: What are the different types of indexes in PostgreSQL?

**Answer:** PostgreSQL supports various index types optimized for different query patterns.

**B-Tree Index (Default):**

```sql
-- Standard B-tree index
CREATE INDEX idx_users_email ON users(email);

-- Unique index
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Composite index
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);

-- Partial index (only index rows that match condition)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Expression index (index on function result)
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- This enables: SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
```

**Hash Index:**

```sql
-- Hash index (only for equality operations)
CREATE INDEX idx_products_hash ON products USING HASH (name);

-- Faster than B-tree for equality, but doesn't support range queries
-- Good for: =, <> operations
-- Bad for: <, >, <=, >=, ORDER BY
```

**GiST Index (Generalized Search Tree):**

```sql
-- For geometric data and full-text search
CREATE EXTENSION IF NOT EXISTS postgis;  -- For geographic data

-- Index on geometric data
CREATE INDEX idx_locations_geom ON locations USING GIST (coordinates);

-- For full-text search
CREATE INDEX idx_articles_content ON articles USING GIST (to_tsvector('english', content));
```

**GIN Index (Generalized Inverted Index):**

```sql
-- For array values, JSONB, and full-text search
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);
CREATE INDEX idx_articles_content ON articles USING GIN (to_tsvector('english', content));

-- GIN index with fastupdate (default)
CREATE INDEX idx_products_tags_fast ON products USING GIN (tags) WITH (fastupdate = on);

-- GIN index with specific operator class
CREATE INDEX idx_products_metadata_path ON products USING GIN (metadata jsonb_path_ops);
```

**BRIN Index (Block Range INdex):**

```sql
-- Very small index for very large tables with naturally ordered data
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at);

-- Good for: Time-series data, append-only tables
-- Bad for: Random data distribution
-- Trade-off: Very small size, but slower queries
```

**SP-GiST Index (Space-Partitioned Generalized Search Tree):**

```sql
-- For non-balanced data structures like trees, quadtrees
CREATE INDEX idx_points_quadtree ON points USING SP-GIST (coordinates);

-- Good for: Spatial data, prefix trees (tries)
```

**Concurrent Index Creation:**

```sql
-- Create index without locking the table
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Important: Cannot be used in a transaction
-- Takes longer but doesn't block writes
```

### Q7: How do you optimize indexes in PostgreSQL?

**Answer:**

**Index Usage Analysis:**

```sql
-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- Find unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexname::regclass) DESC;

-- Analyze index efficiency
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    CASE WHEN idx_scan > 0 THEN idx_tup_read::float / idx_scan ELSE 0 END as avg_tuples_per_scan
FROM pg_stat_user_indexes
ORDER BY avg_tuples_per_scan DESC;
```

**Index Maintenance:**

```sql
-- Reindex specific index
REINDEX INDEX idx_users_email;

-- Reindex all indexes on a table
REINDEX TABLE users;

-- Reindex concurrently (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_users_email;

-- Analyze table to update statistics
ANALYZE users;

-- Vacuum analyze (reclaims space + updates stats)
VACUUM ANALYZE users;
```

**Index Best Practices:**

```sql
-- 1. Create indexes on foreign keys
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 2. Use partial indexes for filtered data
CREATE INDEX idx_active_users_email ON users(email) WHERE is_active = true;

-- 3. Use expression indexes for function-based queries
CREATE INDEX idx_users_lower_name ON users(LOWER(name));

-- 4. Use covering indexes (INCLUDE clause - PostgreSQL 11+)
CREATE INDEX idx_orders_covering ON orders(user_id, order_date)
INCLUDE (total, status);

-- 5. Use appropriate index type
-- B-tree: Default, good for most cases
-- GIN: Arrays, JSONB, full-text search
-- GiST: Geometric data, ranges
-- BRIN: Large ordered data
-- Hash: Equality only

-- 6. Monitor and remove unused indexes
-- See query above for finding unused indexes

-- 7. Consider index-only scans
-- PostgreSQL can satisfy queries from index alone
-- Good for: Frequently accessed columns

-- 8. Use CONCURRENTLY for production
CREATE INDEX CONCURRENTLY idx_large_table_column ON large_table(column);
```

---

## Query Optimization

### Q8: How do you optimize queries in PostgreSQL?

**Answer:**

**EXPLAIN and EXPLAIN ANALYZE:**

```sql
-- Basic explain (shows plan without executing)
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Explain analyze (executes and shows actual timing)
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- Explain with buffers (shows I/O information)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM large_table WHERE condition;

-- Explain with format options
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT * FROM users WHERE email = 'user@example.com';

-- Key information from EXPLAIN:
-- - Scan type: Seq Scan (bad), Index Scan (good), Index Only Scan (best)
-- - Cost: Estimated cost (lower is better)
-- - Actual time: Actual execution time
-- - Rows: Number of rows processed
-- - Buffers: I/O operations (shared hit = cache, read = disk)
```

**Common Query Patterns:**

```sql
-- 1. Avoid full table scans
-- ❌ BAD: No index, full table scan
SELECT * FROM users WHERE email = 'user@example.com';

-- ✅ GOOD: Uses index
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'user@example.com';

-- 2. Use appropriate JOIN types
-- INNER JOIN: Only matching rows
SELECT u.username, o.order_date
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: All from left, matching from right
SELECT u.username, o.order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 3. Optimize subqueries
-- ❌ BAD: Correlated subquery (executed for each row)
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 100);

-- ✅ GOOD: JOIN (usually more efficient)
SELECT DISTINCT u.*
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.total > 100;

-- 4. Use CTEs (Common Table Expressions) for complex queries
WITH user_orders AS (
    SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
    FROM orders
    GROUP BY user_id
)
SELECT u.username, uo.order_count, uo.total_spent
FROM users u
INNER JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5;

-- 5. Use window functions instead of self-joins
-- ❌ BAD: Self-join
SELECT u1.username, u2.username as friend
FROM friendships f
INNER JOIN users u1 ON f.user_id = u1.id
INNER JOIN users u2 ON f.friend_id = u2.id;

-- ✅ GOOD: Window functions (if applicable)
-- Or keep the join if it's the most efficient

-- 6. Use LIMIT for pagination
SELECT * FROM large_table
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;  -- Page 1

-- Better: Keyset pagination for large offsets
SELECT * FROM large_table
WHERE id > last_seen_id
ORDER BY id
LIMIT 20;
```

**Query Rewriting Examples:**

```sql
-- ❌ BAD: Function on column prevents index use
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- ✅ GOOD: Use expression index or ILIKE
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- Or:
SELECT * FROM users WHERE email ILIKE 'user@example.com';

-- ❌ BAD: Leading wildcard in LIKE
SELECT * FROM products WHERE name LIKE '%widget%';

-- ✅ GOOD: Use full-text search for partial matches
CREATE INDEX idx_products_name ON products USING GIN (to_tsvector('english', name));
SELECT * FROM products
WHERE to_tsvector('english', name) @@ to_tsquery('english', 'widget');

-- ❌ BAD: OR conditions (can't use single index)
SELECT * FROM products WHERE category = 'electronics' OR price > 500;

-- ✅ GOOD: Use UNION ALL or separate queries
SELECT * FROM products WHERE category = 'electronics'
UNION ALL
SELECT * FROM products WHERE price > 500 AND category != 'electronics';
```

---

## Stored Procedures & Functions

### Q9: How do you create and use stored procedures in PostgreSQL?

**Answer:** PostgreSQL supports powerful stored procedures and functions using PL/pgSQL (Procedural Language/PostgreSQL).

**Basic Function:**

```sql
-- Create a simple function
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

**Function with Parameters:**

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

-- Use in query
SELECT
    o.id,
    o.order_date,
    calculate_order_total(o.id) as total
FROM orders o;
```

**Stored Procedure (PostgreSQL 11+):**

```sql
-- Create stored procedure (can perform transactions)
CREATE OR REPLACE PROCEDURE process_order(
    p_user_id INTEGER,
    p_product_ids INTEGER[],
    p_quantities INTEGER[]
) AS $$
DECLARE
    v_order_id INTEGER;
    v_total NUMERIC := 0;
    v_product_price NUMERIC;
    v_index INTEGER;
BEGIN
    -- Start transaction
    -- Create order
    INSERT INTO orders (user_id, order_date, total)
    VALUES (p_user_id, NOW(), 0)
    RETURNING id INTO v_order_id;

    -- Add order items
    FOR v_index IN 1..array_length(p_product_ids, 1) LOOP
        -- Get product price
        SELECT price INTO v_product_price
        FROM products
        WHERE id = p_product_ids[v_index];

        -- Insert order item
        INSERT INTO order_items (order_id, product_id, quantity, price)
        VALUES (v_order_id, p_product_ids[v_index], p_quantities[v_index], v_product_price);

        -- Add to total
        v_total := v_total + (v_product_price * p_quantities[v_index]);
    END LOOP;

    -- Update order total
    UPDATE orders
    SET total = v_total
    WHERE id = v_order_id;

    -- Commit is automatic at end of procedure unless ROLLBACK is called
    -- For explicit control, use exception handling
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Error processing order: %', SQLERRM;
        ROLLBACK;
END;
$$ LANGUAGE plpgsql;

-- Call the procedure
CALL process_order(1, ARRAY[1, 2, 3], ARRAY[2, 1, 3]);
```

**Trigger Functions:**

```sql
-- Create trigger function
CREATE OR REPLACE FUNCTION update_user_order_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users
        SET order_count = order_count + 1,
            last_order_date = NEW.order_date
        WHERE id = NEW.user_id;
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users
        SET order_count = order_count - 1
        WHERE id = OLD.user_id;
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER trigger_update_user_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW
EXECUTE FUNCTION update_user_order_count();

-- The trigger automatically updates user order counts
```

---

## PostgreSQL Extensions

### Q10: What are PostgreSQL extensions and how do you use them?

**Answer:** PostgreSQL extensions add additional functionality to the database. They can be enabled/disabled per database.

**Popular Extensions:**

```sql
-- List available extensions
SELECT * FROM pg_available_extensions;

-- List installed extensions
SELECT * FROM pg_extension;

-- Install extension
CREATE EXTENSION IF NOT EXISTS extension_name;

-- Remove extension
DROP EXTENSION IF EXISTS extension_name;
```

**Common Extensions:**

**1. pg_stat_statements (Query Statistics):**

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- View query statistics
SELECT
    query,
    calls,
    total_time,
    mean_time,
    max_time,
    rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

-- Find slow queries
SELECT
    query,
    calls,
    total_time / 1000 as total_seconds,
    mean_time / 1000 as avg_seconds,
    max_time / 1000 as max_seconds
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

**2. PostGIS (Geographic Information Systems):**

```sql
-- Enable PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;

-- Create table with geometry column
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    coordinates GEOMETRY(Point, 4326)  -- WGS84 coordinate system
);

-- Insert location
INSERT INTO locations (name, coordinates)
VALUES ('New York', ST_SetSRID(ST_MakePoint(-74.0060, 40.7128), 4326));

-- Query nearby locations
SELECT name, ST_AsText(coordinates)
FROM locations
WHERE ST_DWithin(
    coordinates,
    ST_SetSRID(ST_MakePoint(-74.0060, 40.7128), 4326),
    0.01  -- Degrees
);

-- Calculate distance
SELECT
    l1.name as location1,
    l2.name as location2,
    ST_Distance(l1.coordinates, l2.coordinates) * 111.32 as distance_km
FROM locations l1, locations l2
WHERE l1.id < l2.id;
```

**3. pg_trgm (Trigram Matching):**

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create trigram index
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- Fuzzy string matching
SELECT * FROM products
WHERE name % 'laptop';  -- Similar to 'laptop'

-- Get similarity score
SELECT
    name,
    similarity(name, 'laptop') as similarity_score
FROM products
WHERE name % 'laptop'
ORDER BY similarity DESC;
```

**4. uuid-ossp (UUID Generation):**

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Generate UUID
SELECT uuid_generate_v1();   -- Time-based UUID
SELECT uuid_generate_v4();   -- Random UUID

-- Use in table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50)
);

-- Insert with UUID
INSERT INTO users (username) VALUES ('user1');
```

**5. hstore (Key-Value Store):**

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS hstore;

-- Create table with hstore column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes hstore
);

-- Insert hstore data
INSERT INTO products (name, attributes)
VALUES ('Laptop', 'color=>black, ram=>8GB, storage=>256GB');

-- Query hstore data
SELECT name, attributes->'color' as color FROM products;
SELECT * FROM products WHERE attributes @> 'ram=>8GB';

-- Update hstore
UPDATE products
SET attributes = attributes || 'warranty=>2 years'
WHERE id = 1;

-- Remove key
UPDATE products
SET attributes = delete(attributes, 'warranty')
WHERE id = 1;
```

---

## Replication & High Availability

### Q11: What are the different replication methods in PostgreSQL?

**Answer:** PostgreSQL supports several replication methods for different use cases.

**Streaming Replication (Physical Replication):**

```sql
-- Master (Primary) Configuration:
-- postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB
hot_standby = on

-- pg_hba.conf (allow replication connections)
host    replication     replicator      192.168.1.0/24      md5

-- Create replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'password';

-- Take base backup on standby
pg_basebackup -h master_host -D /var/lib/postgresql/data -U replicator -P -v -R

-- Standby configuration:
-- postgresql.conf
hot_standby = on

-- recovery.conf (or postgresql.conf in PostgreSQL 12+)
standby_mode = 'on'
primary_conninfo = 'host=master_host port=5432 user=replicator password=password'
```

**Logical Replication:**

```sql
-- Publisher configuration:
-- postgresql.conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10

-- Create publication
CREATE PUBLICATION my_publication FOR TABLE users, orders;

-- Subscriber configuration:
-- Create subscription
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=publisher_host dbname=mydb user=postgres password=password'
PUBLICATION my_publication;

-- Add table to existing publication
ALTER PUBLICATION my_publication ADD TABLE products;

-- Remove table from publication
ALTER PUBLICATION my_publication DROP TABLE products;

-- Drop subscription
DROP SUBSCRIPTION my_subscription;
```

**Replication Monitoring:**

```sql
-- Check replication status on master
SELECT
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;

-- Check replication lag on standby
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

## Performance Tuning

### Q12: How do you tune PostgreSQL performance?

**Answer:**

**Configuration Parameters:**

```sql
-- Memory settings (postgresql.conf)
shared_buffers = 4GB              -- 25% of RAM (for dedicated DB server)
effective_cache_size = 12GB       -- 50-75% of RAM
work_mem = 64MB                   -- Per operation
maintenance_work_mem = 1GB        -- For maintenance operations

-- WAL settings
wal_buffers = 16MB
min_wal_size = 1GB
max_wal_size = 4GB
checkpoint_completion_target = 0.9

-- Query planner
random_page_cost = 1.1            -- For SSDs (default 4.0 for HDDs)
effective_io_concurrency = 200    -- For SSDs

-- Connection settings
max_connections = 200
superuser_reserved_connections = 3

-- Logging
log_duration = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

**Vacuum and Autovacuum Tuning:**

```sql
-- Autovacuum settings
autovacuum = on
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
autovacuum_analyze_scale_factor = 0.1
autovacuum_vacuum_cost_delay = 20ms
autovacuum_vacuum_cost_limit = 200

-- Per-table autovacuum settings
ALTER TABLE users SET (
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_threshold = 500
);
```

**Query Performance Monitoring:**

```sql
-- Enable pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slow queries
SELECT
    query,
    calls,
    total_time,
    mean_time,
    max_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- Check table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan;
```

---

## PostgreSQL vs MySQL

### Q13: Detailed comparison of PostgreSQL and MySQL?

**Answer:**

**Architecture:**
- PostgreSQL: Process-based, one connection per process
- MySQL: Thread-based, one thread per connection

**SQL Compliance:**
- PostgreSQL: Very high compliance with SQL standards
- MySQL: Good compliance but with some deviations

**Data Types:**
- PostgreSQL: Rich set including JSONB, arrays, ranges, custom types
- MySQL: Standard types with basic JSON support

**Indexes:**
- PostgreSQL: B-tree, Hash, GiST, GIN, BRIN, SP-GiST
- MySQL: B-tree, Hash (limited), Full-text, Spatial

**Transactions:**
- PostgreSQL: Full ACID compliance, savepoints, two-phase commit
- MySQL: Full ACID compliance (InnoDB), savepoints

**Replication:**
- PostgreSQL: Streaming (physical), Logical, Third-party tools
- MySQL: Statement-based, Row-based, Mixed, Group replication

**Performance:**
- PostgreSQL: Excellent for complex queries, writes, and concurrency
- MySQL: Excellent for read-heavy workloads, simple queries

**Extensions:**
- PostgreSQL: Rich ecosystem, custom extensions
- MySQL: Limited, mostly storage engines

**Use Cases:**

**Choose PostgreSQL when:**
- Complex data models and relationships
- Need for advanced data types
- Complex analytical queries
- Geographic data processing
- Custom business logic in database
- Strong data integrity requirements
- Need for extensibility

**Choose MySQL when:**
- Simple CRUD applications
- Web applications with read-heavy workloads
- Need for simple setup and maintenance
- Large community support for web development
- LAMP stack compatibility
- Budget constraints with cloud hosting

---

## Advanced Features

### Q14: What are window functions in PostgreSQL?

**Answer:** Window functions perform calculations across a set of table rows related to the current row.

**Basic Window Functions:**

```sql
-- ROW_NUMBER: Unique row number
SELECT
    username,
    order_date,
    total,
    ROW_NUMBER() OVER (ORDER BY total DESC) as rank
FROM orders;

-- RANK: Rank with ties
SELECT
    username,
    total,
    RANK() OVER (ORDER BY total DESC) as rank
FROM orders;

-- DENSE_RANK: Rank without gaps
SELECT
    username,
    total,
    DENSE_RANK() OVER (ORDER BY total DESC) as dense_rank
FROM orders;

-- LAG/LEAD: Access values from other rows
SELECT
    username,
    order_date,
    total,
    LAG(total) OVER (ORDER BY order_date) as previous_order_total,
    LEAD(total) OVER (ORDER BY order_date) as next_order_total
FROM orders;

-- Aggregate functions as window functions
SELECT
    username,
    order_date,
    total,
    SUM(total) OVER (ORDER BY order_date) as running_total,
    AVG(total) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as moving_avg
FROM orders;
```

**PARTITION BY:**

```sql
-- Calculate per-user order totals
SELECT
    username,
    order_date,
    total,
    SUM(total) OVER (PARTITION BY user_id ORDER BY order_date) as user_running_total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) as user_order_number
FROM orders
JOIN users ON orders.user_id = users.id;
```

**Window Frames:**

```sql
-- ROWS vs RANGE
SELECT
    username,
    order_date,
    total,
    -- Sum of current row and 2 previous rows
    SUM(total) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as rolling_sum_3_rows,

    -- Sum of rows within 3 days
    SUM(total) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL '3 days' PRECEDING AND CURRENT ROW
    ) as rolling_sum_3_days
FROM orders;
```

### Q15: What are Common Table Expressions (CTEs) in PostgreSQL?

**Answer:** CTEs allow you to define temporary result sets that can be referenced within a SELECT, INSERT, UPDATE, or DELETE statement.

**Basic CTE:**

```sql
WITH user_orders AS (
    SELECT user_id, COUNT(*) as order_count, SUM(total) as total_spent
    FROM orders
    GROUP BY user_id
)
SELECT
    u.username,
    uo.order_count,
    uo.total_spent
FROM users u
JOIN user_orders uo ON u.id = uo.user_id
WHERE uo.order_count > 5;
```

**Multiple CTEs:**

```sql
WITH
monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) as month,
        SUM(total) as monthly_total
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
),
sales_growth AS (
    SELECT
        month,
        monthly_total,
        LAG(monthly_total) OVER (ORDER BY month) as previous_month,
        (monthly_total - LAG(monthly_total) OVER (ORDER BY month)) / LAG(monthly_total) OVER (ORDER BY month) * 100 as growth_percentage
    FROM monthly_sales
)
SELECT * FROM sales_growth;
```

**Recursive CTE:**

```sql
-- Hierarchical data example (organization chart)
WITH RECURSIVE org_chart AS (
    -- Base case: top level managers
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
CREATE TABLE orders (
    id VARCHAR(50) PRIMARY KEY,
    user_id VARCHAR(50)
);

-- ✅ GOOD: Using appropriate types
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id)
);
```

### Mistake 2: Not using connection pooling

```sql
-- ❌ BAD: Opening new connection for each query
-- (In application code)
for each query:
    connection = create_connection()
    execute_query(connection)
    connection.close()

-- ✅ GOOD: Using connection pool
-- (In application code)
pool = create_connection_pool()
for each query:
    connection = pool.get_connection()
    execute_query(connection)
    pool.return_connection(connection)
```

### Mistake 3: Not analyzing tables after bulk operations

```sql
-- ❌ BAD: Not updating statistics
INSERT INTO large_table SELECT * FROM staging_table;
-- Queries may use inefficient plans

-- ✅ GOOD: Analyze after bulk operations
INSERT INTO large_table SELECT * FROM staging_table;
ANALYZE large_table;
-- Or use VACUUM ANALYZE
VACUUM ANALYZE large_table;
```

### Mistake 4: Not using transactions for multi-step operations

```sql
-- ❌ BAD: No transaction
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- If second UPDATE fails, first is committed

-- ✅ GOOD: With transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Or ROLLBACK if error
```

### Mistake 5: Not using prepared statements

```sql
-- ❌ BAD: SQL injection risk, poor performance
$sql = "SELECT * FROM users WHERE email = '" . $email . "'";
$result = pg_query($connection, $sql);

-- ✅ GOOD: Prepared statement
$result = pg_prepare($connection, "get_user", 'SELECT * FROM users WHERE email = $1');
$result = pg_execute($connection, "get_user", array($email));
```

---

## Short Revision Summary

### Key PostgreSQL Concepts

**Data Types:**
- Advanced: Arrays, JSON/JSONB, Ranges, Custom Types
- JSON vs JSONB: JSON (text) vs JSONB (binary, faster)
- Arrays: `INTEGER[]`, `TEXT[]`, array functions
- Ranges: `DATERANGE`, `NUMRANGE`, `TSRANGE`

**Indexes:**
- B-tree: Default, good for most cases
- GIN: Arrays, JSONB, full-text search
- GiST: Geometric data, ranges
- BRIN: Large ordered data
- Hash: Equality only
- Partial: `WHERE condition`
- Expression: `function(column)`

**Query Optimization:**
- Use EXPLAIN ANALYZE for query analysis
- Create appropriate indexes
- Avoid full table scans
- Use CTEs for complex queries
- Use window functions for analytics
- Optimize JOINs and subqueries

**MVCC:**
- Multi-Version Concurrency Control
- Readers don't block writers
- VACUUM cleans up old versions
- Snapshot isolation

**Transactions:**
- ACID compliant
- Savepoints supported
- Two-phase commit
- Multiple isolation levels

**Extensions:**
- pg_stat_statements: Query statistics
- PostGIS: Geographic data
- pg_trgm: Fuzzy string matching
- uuid-ossp: UUID generation
- hstore: Key-value store

**Replication:**
- Streaming: Physical replication
- Logical: Table-level replication
- High availability with failover

### Quick Reference

**Create Index:**
```sql
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_name ON table USING GIN (jsonb_column);
CREATE INDEX idx_name ON table(column) WHERE condition;
```

**JSON Operations:**
```sql
-- Query
SELECT data->>'key' FROM table;
SELECT data->'nested'->>'key' FROM table;

-- Modify
UPDATE table SET data = data || '{"new": "value"}'::jsonb;
UPDATE table SET data = data - 'key';

-- Index
CREATE INDEX idx_data ON table USING GIN (data);
```

**Window Functions:**
```sql
SELECT
    column,
    ROW_NUMBER() OVER (ORDER BY column) as row_num,
    SUM(column) OVER (PARTITION BY group_column) as total
FROM table;
```

**CTE:**
```sql
WITH cte_name AS (
    SELECT ... FROM ...
)
SELECT * FROM cte_name;
```

### Critical Points for Interviews:

1. **MVCC**: Multi-Version Concurrency Control enables high concurrency
2. **JSON vs JSONB**: JSONB is binary, faster, but loses formatting
3. **Index Types**: Choose based on data and query patterns (B-tree, GIN, GiST)
4. **Query Optimization**: Use EXPLAIN ANALYZE, appropriate indexes
5. **Extensions**: Rich ecosystem for specialized functionality
6. **Replication**: Streaming (physical) and logical replication
7. **Window Functions**: Powerful for analytics without self-joins
8. **CTEs**: Improve readability and performance for complex queries
9. **Transactions**: Full ACID compliance with savepoints
10. **Performance Tuning**: Memory settings, vacuum configuration, connection pooling

---

**Next Topics to Study:**
- MongoDB Document Modeling and Aggregation
- General Database Concepts (ACID, Normalization, SQL Fundamentals)
- NoSQL vs SQL Decision Making
- Database Design Patterns
- Cloud Database Services (AWS RDS, Google Cloud SQL, Azure Database)