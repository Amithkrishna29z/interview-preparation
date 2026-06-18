# SQL Advanced Window Functions & Query Optimization
## Complete Interview Preparation Guide for Full Stack Java Developers

---

## Table of Contents
1. [Window Functions — Overview](#1-window-functions--overview)
2. [Ranking Functions](#2-ranking-functions)
3. [Offset Functions](#3-offset-functions)
4. [Aggregate Window Functions](#4-aggregate-window-functions)
5. [PARTITION BY and ORDER BY in Windows](#5-partition-by-and-order-by-in-windows)
6. [Frame Specification: ROWS vs RANGE](#6-frame-specification-rows-vs-range)
7. [Common Table Expressions (CTEs)](#7-common-table-expressions-ctes)
8. [Recursive CTEs](#8-recursive-ctes)
9. [Query Techniques: Self Joins, Anti-Joins, LATERAL](#9-query-techniques-self-joins-anti-joins-lateral)
10. [EXPLAIN and Query Optimization](#10-explain-and-query-optimization)
11. [Advanced Aggregation](#11-advanced-aggregation)
12. [JSON in SQL](#12-json-in-sql)
13. [Transactions and Locking](#13-transactions-and-locking)
14. [Classic SQL Interview Problems](#14-classic-sql-interview-problems)
15. [Interview Questions & Answers](#15-interview-questions--answers)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Window Functions — Overview

A **window function** calculates across a set of rows related to the current row without collapsing them into one output row.

**Key difference from GROUP BY:** GROUP BY turns 10 rows into 3. Window functions keep all 10 rows and add a calculated column to each.

```sql
-- GROUP BY: one row per department
SELECT department_id, AVG(salary) FROM employees GROUP BY department_id;

-- Window function: one row per employee + department average alongside it
SELECT employee_id, name, department_id, salary,
       AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary
FROM employees;
```

### Syntax

```sql
function_name([args]) OVER (
    [PARTITION BY col, ...]
    [ORDER BY col [ASC|DESC], ...]
    [ROWS|RANGE BETWEEN frame_start AND frame_end]
)
```

### Processing Order (Critical for Interviews)

```
FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT (window functions here) → ORDER BY → LIMIT
```

**Window functions run in SELECT, AFTER WHERE/HAVING.** You cannot filter on a window function result in WHERE — wrap in a subquery or CTE:

```sql
-- WRONG
WHERE RANK() OVER (...) = 1   -- ERROR

-- CORRECT
SELECT * FROM (
    SELECT *, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
) t WHERE rnk = 1;
```

### Window Functions vs Aggregate Functions

| Feature | GROUP BY Aggregate | Window Function |
|---|---|---|
| Collapses rows | Yes | No |
| Access individual row columns | No | Yes |
| Filter results with WHERE | Use HAVING | Use subquery/CTE |
| Access other rows (LAG, LEAD) | No | Yes |

---

## 2. Ranking Functions

### Sample Data

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY, name VARCHAR(100), department VARCHAR(50),
    salary DECIMAL(10,2), hire_date DATE, manager_id INT
);
INSERT INTO employees VALUES
(1,'Alice','Engineering',95000,'2019-01-15',NULL),
(2,'Bob','Engineering',85000,'2020-03-10',1),
(3,'Charlie','Engineering',85000,'2020-06-01',1),
(4,'Diana','Engineering',75000,'2021-08-20',1),
(5,'Eve','Marketing',70000,'2019-05-01',NULL),
(9,'Iris','Finance',90000,'2018-07-01',NULL),
(10,'Jack','Finance',80000,'2019-11-30',9),
(11,'Karen','Finance',80000,'2020-04-15',9),
(12,'Leo','Finance',72000,'2021-12-01',9);
```

### ROW_NUMBER()

Assigns a **unique sequential integer** — no ties, even for identical values.

```sql
SELECT name, department, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
FROM employees;
```

**Common pattern — get top row per group:**
```sql
SELECT department, name, salary FROM (
    SELECT department, name, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
) t WHERE rn = 1;
```

Also used to **deduplicate**: `DELETE ... WHERE rn > 1`.

### RANK()

Tied rows get the **same rank**; numbers after a tie **skip** (1, 1, 3).

```sql
SELECT name, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
-- Engineering: Alice=1, Bob=2, Charlie=2, Diana=4
```

### DENSE_RANK()

Tied rows get the **same rank**; numbers **never skip** (1, 1, 2).

```sql
SELECT name, salary,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dr
FROM employees;
-- Engineering: Alice=1, Bob=2, Charlie=2, Diana=3
```

**Common pattern — top N salary tiers:**
```sql
SELECT department, name, salary FROM (
    SELECT department, name, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dr
    FROM employees
) t WHERE dr <= 3;
```

### Comparison

| name    | salary | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|---|
| Alice   | 95000  | 1 | 1 | 1 |
| Bob     | 85000  | 2 | 2 | 2 |
| Charlie | 85000  | 3 | 2 | 2 |
| Diana   | 75000  | 4 | 4 | 3 |

### NTILE(n)

Divides rows into **n equal buckets** (quartiles, deciles, etc.).

```sql
SELECT name, salary,
       NTILE(4) OVER (PARTITION BY department ORDER BY salary DESC) AS quartile
FROM employees WHERE department = 'Engineering';
```

---

## 3. Offset Functions

Access values from **other rows** relative to the current row — no self-join needed.

### LAG(column, offset, default)

Returns the value from `offset` rows **before** the current row.

```sql
SELECT sale_date, revenue,
       LAG(revenue, 1, 0) OVER (ORDER BY sale_date) AS prev_day_revenue,
       revenue - LAG(revenue, 1, 0) OVER (ORDER BY sale_date) AS daily_change
FROM daily_sales;
```

**Common use — find gaps in sequences:**
```sql
SELECT user_id, prev_login, curr_login, gap_days FROM (
    SELECT user_id, login_date AS curr_login,
           LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS prev_login,
           login_date - LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS gap_days
    FROM user_logins
) t WHERE gap_days > 1;
```

### LEAD(column, offset, default)

Returns the value from `offset` rows **after** the current row (mirror of LAG).

```sql
SELECT session_id, page_name, view_time,
       LEAD(page_name) OVER (PARTITION BY session_id ORDER BY view_time) AS next_page
FROM page_views;
```

### FIRST_VALUE / LAST_VALUE

`FIRST_VALUE` is straightforward. `LAST_VALUE` requires an explicit frame — by default the frame ends at the current row, so it returns the current row's value, not the partition's last value.

```sql
-- CORRECT LAST_VALUE — extend frame to end of partition
SELECT name, salary,
       LAST_VALUE(name) OVER (
           PARTITION BY department ORDER BY salary DESC
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS lowest_earner
FROM employees;
```

---

## 4. Aggregate Window Functions

Standard aggregates (SUM, AVG, COUNT, MIN, MAX) become window functions with an OVER clause.

### Running Total

```sql
SELECT sale_date, revenue,
       SUM(revenue) OVER (ORDER BY sale_date) AS running_total
FROM daily_sales;
```

### 7-Day Moving Average

```sql
SELECT sale_date, revenue,
       ROUND(AVG(revenue) OVER (
           ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ), 2) AS moving_avg_7d
FROM daily_sales;
```

### Percent of Total

```sql
SELECT sale_date, revenue,
       SUM(revenue) OVER () AS grand_total,   -- empty OVER = entire result set
       ROUND(revenue / SUM(revenue) OVER () * 100, 2) AS pct_of_total
FROM daily_sales;
```

---

## 5. PARTITION BY and ORDER BY in Windows

**PARTITION BY** splits rows into independent groups (like GROUP BY, but never reduces rows).

```sql
-- Each employee row keeps all columns + the department average
SELECT employee_id, name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg,
       salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

No PARTITION BY = the entire result set is one window.

**ORDER BY inside OVER** controls the row order for the calculation. Without it, each row sees the full partition (good for AVG/SUM of the whole group). With it, the default frame becomes `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (good for running totals).

```sql
-- No ORDER BY: every row gets the same dept total
SUM(salary) OVER (PARTITION BY department)

-- With ORDER BY: running total that grows row by row
SUM(salary) OVER (PARTITION BY department ORDER BY hire_date)
```

---

## 6. Frame Specification: ROWS vs RANGE

### ROWS — Physical Row Count (Prefer This)

ROWS counts actual rows — exact and predictable.

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- running total
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW           -- trailing 7-row window
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING           -- 3-row centered window
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- entire partition
```

### RANGE — Logical Value Range

RANGE groups rows with **equal ORDER BY values** as peers. For two rows with score=10, both see the same cumulative sum (both are included in each other's frame). Use ROWS for predictability; RANGE only when equal values should be treated as a group.

---

## 7. Common Table Expressions (CTEs)

### Syntax

```sql
WITH cte_name AS (
    SELECT ...
),
second_cte AS (
    SELECT ... FROM cte_name
)
SELECT * FROM second_cte;
```

### CTE vs Subquery vs Temp Table

| Aspect | Subquery | CTE | Temp Table |
|---|---|---|---|
| Readability | Poor for complex | Excellent | Good |
| Reusable in same query | No | Yes | Yes |
| Indexes | No | No | Yes |
| Recursive | No | Yes | No |
| Best for | Simple, one-time | Multi-step, readable, recursive | Large datasets, cross-query reuse |

```sql
-- CTE (readable) vs subquery (nested):
WITH dept_avg AS (
    SELECT name, salary, AVG(salary) OVER (PARTITION BY department) AS avg_sal
    FROM employees
)
SELECT name, salary FROM dept_avg WHERE salary > avg_sal;
```

### Multiple CTEs (step-by-step analysis)

```sql
WITH
monthly_revenue AS (
    SELECT DATE_TRUNC('month', order_date) AS month, customer_id, SUM(amount) AS monthly_spend
    FROM orders GROUP BY 1, 2
),
ranked_months AS (
    SELECT *, RANK() OVER (PARTITION BY customer_id ORDER BY monthly_spend DESC) AS spend_rank
    FROM monthly_revenue
),
best_months AS (
    SELECT customer_id, month AS best_month, monthly_spend AS peak_spend
    FROM ranked_months WHERE spend_rank = 1
)
SELECT c.customer_id, c.name, bm.best_month, bm.peak_spend
FROM customers c JOIN best_months bm USING (customer_id)
ORDER BY peak_spend DESC;
```

---

## 8. Recursive CTEs

Recursive CTEs solve hierarchical problems (org charts, category trees, sequence generation).

### Syntax

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor: base case
    SELECT ...

    UNION ALL

    -- Recursive member: references cte_name
    SELECT ... FROM ... JOIN cte_name ON ...
    -- Must eventually return no rows (termination condition)
)
SELECT * FROM cte_name;
```

### Employee Org Chart

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, name, manager_id, 0 AS depth, name::TEXT AS path
    FROM employees WHERE manager_id IS NULL   -- top-level employees

    UNION ALL

    SELECT e.employee_id, e.name, e.manager_id, oc.depth + 1, oc.path || ' -> ' || e.name
    FROM employees e JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT employee_id, name, depth, path FROM org_chart ORDER BY path;
```

**Infinite loop prevention:** Ensure the recursive member's WHERE eventually returns no rows. PostgreSQL caps depth at 100 by default (`max_recursion_depth`).

Other uses: category trees, date range generation (`d + INTERVAL '1 day'`), bill of materials.

---

## 9. Query Techniques: Self Joins, Anti-Joins, LATERAL

### Self Join

Join a table to itself — for hierarchies or comparing rows within the same table.

```sql
-- Employees who earn more than their manager
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS mgr_salary
FROM employees e JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

### Anti-Join

Find rows in A with no match in B. Prefer NOT EXISTS (handles NULLs correctly):

```sql
-- NOT EXISTS (preferred)
SELECT c.customer_id, c.name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- LEFT JOIN ... IS NULL (equivalent)
SELECT c.customer_id, c.name FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```

Avoid `NOT IN` — if the subquery returns any NULL, the entire query returns no rows.

### LATERAL Join

A LATERAL join lets the right-side subquery reference columns from the left side. Useful for top-N per group:

```sql
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_id, amount FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY amount DESC LIMIT 2
) o;
```

---

## 10. EXPLAIN and Query Optimization

### Reading EXPLAIN ANALYZE

Run `EXPLAIN ANALYZE <query>` to see the real execution plan. Watch for:
- **Seq Scan on a large table** → likely a missing index
- **Estimated rows far from actual rows** → stale statistics (run `ANALYZE`)
- **High-cost nodes** → focus optimization there first

### Indexing

**B-tree** (default) covers `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'prefix%'`.

```sql
CREATE INDEX idx_salary ON employees(salary);
CREATE INDEX idx_dept_salary ON employees(department, salary);  -- composite
```

**Left-prefix rule for composite indexes:** columns must be filtered left-to-right.
```sql
-- Uses index: WHERE department = 'Eng'
-- Uses index: WHERE department = 'Eng' AND salary > 80000
-- Does NOT use: WHERE salary > 80000  (skips left column)
```

**Other index types:**
- **Partial** — `CREATE INDEX ... WHERE status = 'ACTIVE'` (smaller, faster for filtered queries)
- **Expression** — `CREATE INDEX ... ON employees(LOWER(email))` (query must use same expression)
- **Covering** — `INCLUDE (name, salary)` enables index-only scans

### Key Optimization Techniques

**1. Avoid functions on indexed columns in WHERE:**
```sql
-- BAD: index unused
WHERE YEAR(hire_date) = 2020
-- GOOD: transform the constant
WHERE hire_date BETWEEN '2020-01-01' AND '2020-12-31'
```

**2. Keyset pagination instead of OFFSET:**
```sql
-- BAD: scans and discards N rows
SELECT * FROM orders ORDER BY order_id LIMIT 20 OFFSET 10000;
-- GOOD: O(log n) with index
SELECT * FROM orders WHERE order_id > :last_seen_id ORDER BY order_id LIMIT 20;
```

**3. Avoid SELECT * :** Select only needed columns; prevents covering index scans.

**4. Replace correlated subqueries with window functions:**
```sql
-- BAD: runs subquery once per row
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department = e.department)

-- GOOD: calculates averages once
SELECT * FROM (
    SELECT *, AVG(salary) OVER (PARTITION BY department) AS dept_avg FROM employees
) t WHERE salary > dept_avg;
```

**5. N+1 query problem:** Never query inside a loop. JOIN or use `IN`/`EXISTS` to fetch all data in one query.

---

## 11. Advanced Aggregation

### GROUPING SETS, ROLLUP, CUBE

```sql
-- ROLLUP: hierarchical subtotals (region → department → grand total)
SELECT region, department, SUM(revenue)
FROM regional_sales
GROUP BY ROLLUP(region, department);

-- GROUPING SETS: specify exactly which combinations you need
GROUP BY GROUPING SETS ((department, year), (department), ());

-- CUBE: all possible combinations (2^n groupings)
GROUP BY CUBE(region, product, year);
```

### FILTER Clause

Apply a condition to a specific aggregate — cleaner than `CASE WHEN`:

```sql
SELECT department,
       COUNT(*) AS total,
       COUNT(*) FILTER (WHERE salary > 80000) AS high_earners,
       AVG(salary) FILTER (WHERE hire_date >= '2020-01-01') AS avg_new_hire_salary
FROM employees GROUP BY department;
```

### PERCENTILE_CONT / PERCENTILE_DISC

```sql
SELECT department,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary,
       PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY salary) AS p90_salary
FROM employees GROUP BY department;
```

---

## 12. JSON in SQL

### PostgreSQL JSONB

```sql
-- -> returns JSONB, ->> returns text
SELECT name,
       metadata ->> 'brand'          AS brand_text,
       metadata -> 'specs' ->> 'ram' AS ram
FROM products;

-- Containment filter
SELECT * FROM products WHERE metadata @> '{"brand": "Dell"}';

-- GIN index for fast JSONB queries
CREATE INDEX idx_metadata ON products USING GIN(metadata);
```

### MySQL JSON

Uses functions: `JSON_EXTRACT(metadata, '$.brand')` or shorthand `->>>'$.brand'`. To index a JSON value, create a generated column:

```sql
ALTER TABLE products ADD COLUMN brand VARCHAR(100)
    GENERATED ALWAYS AS (metadata->>'$.brand') STORED;
CREATE INDEX idx_brand ON products(brand);
```

---

## 13. Transactions and Locking

### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | Yes | Yes | Yes |
| READ COMMITTED | No | Yes | Yes |
| REPEATABLE READ | No | No | Yes |
| SERIALIZABLE | No | No | No |

- **Dirty Read:** reading uncommitted data that may roll back.
- **Non-Repeatable Read:** same row reads differently within one transaction.
- **Phantom Read:** same query returns different rows within one transaction.

### Pessimistic Locking

```sql
-- Lock rows before modifying
BEGIN;
SELECT * FROM inventory WHERE product_id = 101 FOR UPDATE;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 101;
COMMIT;

-- SKIP LOCKED: job queue — skip rows locked by other workers
SELECT * FROM job_queue WHERE status = 'PENDING'
ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED;
```

### Optimistic Locking

No DB lock — use a version column to detect conflicts:

```sql
-- Read
SELECT account_id, balance, version FROM accounts WHERE account_id = 1;
-- Update only if version unchanged
UPDATE accounts SET balance = balance - 100, version = version + 1
WHERE account_id = 1 AND version = 5;
-- If rowsAffected == 0: conflict — another transaction modified the row
```

Pessimistic = high-contention (financial). Optimistic = low-contention (profile updates).

### Deadlocks

Two transactions each holding a lock the other needs. Prevention: **always acquire locks in the same order**. Databases auto-detect deadlocks and kill one transaction — add retry logic in your app.

---

## 14. Classic SQL Interview Problems

### Problem 1: Second Highest Salary Per Department

```sql
SELECT dept, name, salary FROM (
    SELECT dept, name, salary,
           DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS dr
    FROM emp_salaries
) t WHERE dr = 2;
```

### Problem 2: Employees Who Earn More Than Their Manager

```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS mgr_salary
FROM employees e JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

### Problem 3: Find and Delete Duplicates (Keep One)

```sql
-- Find duplicates
SELECT email, COUNT(*) FROM contacts GROUP BY email HAVING COUNT(*) > 1;

-- Delete duplicates via CTE (PostgreSQL)
WITH dupes AS (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn FROM contacts
)
DELETE FROM contacts WHERE id IN (SELECT id FROM dupes WHERE rn > 1);
```

### Problem 4: Running Total and Moving Average

```sql
SELECT sale_date, revenue,
       SUM(revenue) OVER (ORDER BY sale_date ROWS UNBOUNDED PRECEDING) AS running_total,
       ROUND(AVG(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS moving_avg_7d
FROM daily_sales;
```

### Problem 5: Find Gaps in a Sequence

```sql
SELECT curr_id - prev_id - 1 AS gap_size, prev_id + 1 AS gap_start, curr_id - 1 AS gap_end
FROM (
    SELECT id AS curr_id, LAG(id) OVER (ORDER BY id) AS prev_id FROM sequence_data
) t WHERE curr_id - prev_id > 1;
```

### Problem 6: Pivot Table (Conditional Aggregation)

```sql
SELECT department,
       SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS "2022",
       SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS "2023",
       SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS "2024"
FROM dept_revenue GROUP BY department;
```

### Problem 7: Customers Who Bought A But Not B

```sql
SELECT DISTINCT o.customer_id FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE oi.product_id = 'A'
  AND NOT EXISTS (
      SELECT 1 FROM orders o2
      JOIN order_items oi2 ON o2.order_id = oi2.order_id
      WHERE o2.customer_id = o.customer_id AND oi2.product_id = 'B'
  );
```

### Problem 8: Top 3 Products Per Category

```sql
SELECT category, product_name, total_sales FROM (
    SELECT p.category, p.product_name, SUM(oi.quantity * oi.unit_price) AS total_sales,
           RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.quantity * oi.unit_price) DESC) AS rnk
    FROM products p JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category, p.product_name
) t WHERE rnk <= 3 ORDER BY category, rnk;
```

### Problem 9: Month-over-Month Growth Rate

```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date)::DATE AS month, SUM(amount) AS revenue
    FROM orders GROUP BY 1
)
SELECT month, revenue,
       ROUND(
           (revenue - LAG(revenue) OVER (ORDER BY month))
           / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100, 2
       ) AS mom_growth_pct
FROM monthly ORDER BY month;
```

---

## 15. Interview Questions & Answers

**Q1: What is the difference between ROW_NUMBER(), RANK(), and DENSE_RANK()?**

All three rank rows within a partition, but differ on ties. ROW_NUMBER gives unique sequential numbers (no ties). RANK gives equal numbers for ties but skips afterward (1,1,3). DENSE_RANK gives equal numbers for ties with no skipping (1,1,2). Use ROW_NUMBER for deduplication or "exactly one row per group"; use DENSE_RANK for "top N salary tiers."

---

**Q2: What is the difference between ROWS and RANGE in frame specifications?**

ROWS counts physical rows — `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` always includes exactly 3 rows. RANGE uses logical values — rows with equal ORDER BY values are treated as peers and all included together. ROWS is more predictable; prefer it unless equal values should be grouped.

---

**Q3: Why doesn't LAST_VALUE() work as expected by default?**

The default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which ends at the current row — so LAST_VALUE returns the current row's value. Fix it by extending the frame: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

---

**Q4: Can you use a window function in a WHERE clause?**

No — window functions execute in SELECT, after WHERE and HAVING. Wrap the query in a subquery or CTE and filter there.

---

**Q5: What does PARTITION BY do? How is it different from GROUP BY?**

PARTITION BY divides rows into groups for the window calculation but keeps all rows in the output. GROUP BY collapses rows — each group becomes one row and you can only reference grouped/aggregated columns.

---

**Q6: When would you use a CTE instead of a subquery?**

CTEs are better when: the same result is referenced multiple times (avoids repetition); the query has multiple logical steps (CTEs make each step named and readable); you need recursion; or you want to filter on window function results. Subqueries are fine for simple, one-off filtering.

---

**Q7: What is a recursive CTE? Give a real-world example.**

A recursive CTE references itself. It has an anchor member (base case) and a recursive member, joined by UNION ALL. The recursion continues until the recursive member returns no rows. Classic example: traversing an employee org chart from the CEO down to all reports at any depth.

---

**Q8: How do you prevent infinite recursion in a recursive CTE?**

Ensure the recursive member has a WHERE clause that eventually returns no rows (e.g., depth limit). PostgreSQL enforces a default max of 100 iterations.

---

**Q9: How do you use EXPLAIN ANALYZE to optimize a query?**

Run `EXPLAIN ANALYZE <query>` and look for: Seq Scan on large tables (missing index), large gap between estimated and actual rows (stale stats — run ANALYZE), and high-cost nodes (optimize these first).

---

**Q10: When should you NOT add an index?**

Write-heavy tables (indexes slow every INSERT/UPDATE/DELETE), small tables (full scan is faster), low-cardinality columns (boolean/status with few values), and columns never used in WHERE/JOIN/ORDER BY.

---

**Q11: What is a covering index?**

An index that includes all columns a query needs, enabling an index-only scan without hitting the table. Example: `CREATE INDEX idx ON employees(department) INCLUDE (name, salary)` lets `SELECT name, salary WHERE department = 'Eng'` run entirely from the index.

---

**Q12: How do you handle pagination efficiently at scale?**

Use keyset pagination instead of OFFSET. OFFSET forces the DB to scan and discard N rows each time. Keyset pagination filters by the last seen value — O(log n) with an index: `WHERE order_id > :last_seen_id ORDER BY order_id LIMIT 20`.

---

**Q13: Explain the N+1 query problem and how SQL solves it.**

N+1 occurs when code runs 1 query to fetch N items, then N separate queries for related data. Fix it by JOINing everything in one query, or using `IN`/`EXISTS` to batch the lookups.

---

**Q14: What is the difference between WHERE and HAVING?**

WHERE filters rows before grouping; HAVING filters groups after GROUP BY. Aggregate functions cannot appear in WHERE — use HAVING. Window functions cannot appear in either.

---

**Q15: What are the four transaction isolation levels?**

READ UNCOMMITTED (no protection), READ COMMITTED (no dirty reads — PostgreSQL default), REPEATABLE READ (no dirty or non-repeatable reads — MySQL InnoDB default), SERIALIZABLE (no anomalies, lowest concurrency).

---

**Q16: What is SKIP LOCKED and when do you use it?**

SKIP LOCKED (used with SELECT FOR UPDATE) skips rows already locked by another transaction. Classic use: job queues where multiple workers process jobs concurrently without blocking each other.

---

**Q17: Explain optimistic vs pessimistic locking.**

Pessimistic locks the row on read (`SELECT FOR UPDATE`) — safe but reduces concurrency. Optimistic uses a version column — no lock on read, but the UPDATE checks the version hasn't changed; if it has, retry. Pessimistic suits high-contention (financial); optimistic suits low-contention (profile edits).

---

**Q18: What is a deadlock and how do you prevent it?**

A deadlock is two transactions each holding a lock the other needs. Prevention: always acquire locks in the same order across transactions, keep transactions short, and add retry logic in your app for deadlock errors.

---

**Q19: What is the difference between -> and ->> in PostgreSQL JSONB?**

`->` returns the value as JSONB (preserves type, can chain). `->>` returns the value as text. Use `->>` when comparing or displaying; use `->` when chaining or passing to JSON functions.

---

**Q20: What is the difference between UNION and UNION ALL?**

UNION removes duplicate rows (slower — does a DISTINCT pass). UNION ALL keeps all rows including duplicates (faster). Use UNION ALL by default unless deduplication is required.

---

**Q21: What is the difference between NOT EXISTS and NOT IN?**

NOT EXISTS is safer and generally preferred. NOT IN returns no rows if the subquery contains any NULL (because `x NOT IN (1, NULL)` evaluates as UNKNOWN). NOT EXISTS handles NULLs correctly.

---

**Q22: What is ROLLUP vs CUBE vs GROUPING SETS?**

ROLLUP generates hierarchical subtotals left-to-right plus a grand total — good for Year > Quarter > Month reports. CUBE generates all 2^n combinations — for OLAP-style multi-dimensional analysis. GROUPING SETS gives full control: specify exactly which combinations you need.

---

**Q23: What does NULLIF do?**

`NULLIF(a, b)` returns NULL if `a = b`, otherwise returns `a`. Mainly used to prevent division-by-zero: `revenue / NULLIF(cost, 0)`.

---

**Q24: What is a partial index?**

An index that only includes rows matching a WHERE condition. Use it when only a subset of rows is frequently queried (e.g., `WHERE status = 'ACTIVE'`) — smaller and faster than a full index.

---

**Q25: How do window functions affect performance?**

Window functions require sorting within partitions, but they eliminate far costlier self-joins and correlated subqueries. Best practices: index the PARTITION BY and ORDER BY columns; filter rows before window functions execute (in WHERE or an earlier CTE); prefer ROWS over RANGE.

---

## 16. Quick Reference Cheat Sheet

### Window Function Syntax
```sql
fn() OVER (PARTITION BY col ORDER BY col ROWS BETWEEN x AND y)
```

### Ranking Functions
| Function | Ties | Gaps | Best For |
|---|---|---|---|
| ROW_NUMBER() | Unique | N/A | Deduplication, latest-per-group |
| RANK() | Same rank | Yes (1,1,3) | Competition rankings |
| DENSE_RANK() | Same rank | No (1,1,2) | Top-N salary tiers |
| NTILE(n) | N/A | N/A | Quartiles, percentiles |

### Common Frame Clauses
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW          -- running total
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW                  -- 7-day trailing window
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING                  -- 3-row centered window
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- entire partition
```

### Anti-Patterns to Avoid
1. Functions on indexed columns in WHERE (prevents index use)
2. SELECT * (column bloat, prevents covering index scans)
3. OFFSET pagination at scale (O(n) scan cost)
4. NOT IN with subquery that may return NULLs
5. Correlated subqueries in SELECT (runs once per row)
6. Missing ANALYZE after large data loads (bad estimates)
7. LAST_VALUE() without explicit UNBOUNDED FOLLOWING frame

---

*Last Updated: 2026-06-18*
